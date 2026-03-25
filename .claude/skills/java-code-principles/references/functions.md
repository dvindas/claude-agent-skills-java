# Functions - Deep Examples and Method Design

Read this file when the user needs depth on method design: the stepdown rule across multiple levels, complex conditional extraction, CQS in real services, or argument object patterns.

---

## The Stepdown Rule - Three Levels Deep

The stepdown rule says each method should read at one level of abstraction and delegate everything below it. Here is a realistic three-level example using a rider assignment flow.

```java
// Level 0 - orchestration: reads like a business summary
// You can understand what happens without reading any helper
public void assignRiderToOrder(OrderId orderId) {
    var order       = loadPendingOrder(orderId);
    var rider       = findAvailableRider(order.getZone());
    confirmAssignment(order, rider);
}

// Level 1 - coordination: each method handles one sub-goal
// You can understand the sub-goal without reading level 2
private DeliveryOrder loadPendingOrder(OrderId orderId) {
    var order = orderRepository.findById(orderId)
        .orElseThrow(() -> new OrderNotFoundException(orderId));
    if (!order.isPending()) {
        throw new InvalidOrderStateException(orderId, order.getStatus());
    }
    return order;
}

private Rider findAvailableRider(Zone zone) {
    return riderRepository.findAvailableByZone(zone)
        .stream()
        .min(Comparator.comparing(Rider::getActiveOrderCount))
        .orElseThrow(() -> new NoRidersAvailableException(zone));
}

private void confirmAssignment(DeliveryOrder order, Rider rider) {
    order.assignTo(rider.getId());
    rider.incrementActiveOrderCount();
    orderRepository.save(order);
    riderRepository.save(rider);
    notifications.riderAssigned(order, rider);
}
```

Each level is independently readable. `assignRiderToOrder` tells you the story. `findAvailableRider` tells you how a rider is chosen. The actual SQL query lives one level further down in the repository - not here.

---

## Extracting Complex Conditions

When a boolean expression requires parsing to understand its intent, extract it into a method whose name states the conclusion.

**Single condition - extract when the meaning is not obvious:**

```java
// Unclear: the reader must decode this formula
if (order.getItems().size() > 5 && order.getTotalAmount().compareTo(MIN_BULK_AMOUNT) >= 0) {
    applyBulkDiscount(order);
}

// Clear: the condition has a name
if (qualifiesForBulkDiscount(order)) {
    applyBulkDiscount(order);
}

private boolean qualifiesForBulkDiscount(final Order order) {
    return order.getItems().size() > MIN_BULK_ITEM_COUNT
        && order.getTotalAmount().compareTo(MIN_BULK_AMOUNT) >= 0;
}
```

**Compound condition - extract each clause when they represent distinct rules:**

```java
// Unclear: three independent business rules collapsed into one expression
if (rider.getStatus() == RiderStatus.AVAILABLE
        && rider.getZone() == order.getZone()
        && rider.getActiveOrderCount() < MAX_CONCURRENT_ORDERS
        && !rider.isOnBreak()) {
    assign(order, rider);
}

// Clear: each rule has a name; the compound reads like prose
if (isAvailableForAssignment(rider, order)) {
    assign(order, rider);
}

private boolean isAvailableForAssignment(final Rider rider, final DeliveryOrder order) {
    return isActive(rider)
        && isInCorrectZone(rider, order)
        && hasCapacity(rider);
}

private boolean isActive(final Rider rider) {
    return rider.getStatus() == RiderStatus.AVAILABLE && !rider.isOnBreak();
}

private boolean isInCorrectZone(final Rider rider, final DeliveryOrder order) {
    return rider.getZone() == order.getZone();
}

private boolean hasCapacity(final Rider rider) {
    return rider.getActiveOrderCount() < MAX_CONCURRENT_ORDERS;
}
```

The extracted methods are also individually testable - each business rule can be verified in isolation.

---

## Command Query Separation - Real Service Examples

CQS says a method should either change state (command) or return information (query) - never both. The pattern becomes critical in services where mixing the two creates ambiguous contracts.

**Violation - looks like a query, but creates a cart if one does not exist:**

```java
// The name suggests a read. The caller has no way to know a cart may be created and saved.
public Cart getActiveCart(CustomerId customerId) {
    return cartRepository.findActiveByCustomer(customerId)
        .orElseGet(() -> {
            var cart = new Cart(customerId);
            cartRepository.save(cart);     // hidden side effect
            return cart;
        });
}
```

The caller expects a read. They may call this method in a read-only context - a GET endpoint, a report, a dry-run - and silently create data they never intended to create.

**Fix - separate command and query:**

```java
// Query: reads state, changes nothing
public Optional<Cart> findActiveCart(CustomerId customerId) {
    return cartRepository.findActiveByCustomer(customerId);
}

// Command: creates and persists a new cart, returns nothing
public void createCart(CustomerId customerId) {
    var cart = new Cart(customerId);
    cartRepository.save(cart);
}

// Call site is explicit - the caller decides what happens on absence
var cart = cartService.findActiveCart(customerId)
    .orElseGet(() -> {
        cartService.createCart(customerId);
        return cartService.findActiveCart(customerId).orElseThrow();
    });
```

---

## Argument Objects - When and How to Use Records

Three or more arguments to a method are a signal that those arguments belong together as a concept. Use a `record` to give the concept a name.

**Identifying the right grouping:**

The arguments should represent one coherent concept, not an arbitrary collection of parameters. If you cannot name the record without using "And", the grouping is wrong.

```java
// Four loose arguments - hard to read at the call site
scheduleDelivery("123 Main St", "456 Oak Ave", LocalDateTime.now().plusHours(2), Priority.EXPRESS);

// What does the second String mean? What is the LocalDateTime for?
// The call site gives you nothing.
```

```java
// Group the related data into a concept
public record ScheduleDeliveryRequest(
    Address pickupAddress,
    Address dropoffAddress,
    LocalDateTime scheduledAt,
    Priority priority
) {
    public ScheduleDeliveryRequest {
        Objects.requireNonNull(pickupAddress,  "pickupAddress must not be null");
        Objects.requireNonNull(dropoffAddress, "dropoffAddress must not be null");
        Objects.requireNonNull(scheduledAt,    "scheduledAt must not be null");
        Objects.requireNonNull(priority,       "priority must not be null");
        if (scheduledAt.isBefore(LocalDateTime.now())) {
            throw new IllegalArgumentException("scheduledAt must be in the future");
        }
    }
}

scheduleDelivery(new ScheduleDeliveryRequest(pickup, dropoff, scheduledAt, Priority.EXPRESS));
// Now the call site names what it is passing
```

The compact constructor is also the right place for validation - arguments are validated once, at the boundary, and the method body can trust what it receives.

---

## Avoiding Hidden Side Effects

A method name is a promise. Breaking that promise - by doing something the name does not suggest - creates a temporal coupling that callers have no way to discover without reading the implementation.

**Classic pattern: "calculate" that mutates:**

```java
// The name implies a pure computation.
// The caller uses the return value - and does not notice the voucher was consumed.
public Money calculateDiscountedTotal(Order order, Voucher voucher) {
    var discount = order.getTotal().multiply(voucher.getDiscountRate());
    voucher.markAsUsed();           // hidden mutation: voucher is gone after this call
    return order.getTotal().subtract(discount);
}

// What happens if the payment fails after this call? The voucher is burned.
```

**Fix: separate the pure computation from the side effect:**

```java
// Pure calculation - safe to call repeatedly, no state changes
public Money calculateDiscountedTotal(final Order order, final Voucher voucher) {
    var discount = order.getTotal().multiply(voucher.getDiscountRate());
    return order.getTotal().subtract(discount);
}

// Explicit mutation - the name makes the side effect visible
public void redeemVoucher(final Voucher voucher) {
    voucher.markAsUsed();
}

// The caller now sees both operations and controls the order
var total = pricingService.calculateDiscountedTotal(order, voucher);
paymentGateway.charge(order.getCustomer(), total);
pricingService.redeemVoucher(voucher);   // redeemed only after successful payment
```

The separation also enables better testing: the calculation can be tested without any mutable state, and the redemption can be tested independently.