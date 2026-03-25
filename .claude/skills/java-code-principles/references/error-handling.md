# Error Handling - Complex Real-World Patterns

Read this file when the user needs depth on error handling: nested exception flows, custom exception hierarchies, Optional chaining for complex domain logic, or the Null Object pattern.

---

## Refactoring Nested try/catch - Full Example

Nested try/catch is the most common error handling smell. Each catch block belongs to a different operation, yet they all share the same scope. The result is that it is impossible to tell which exception belongs to which statement.

**Before - three levels of nesting, unclear ownership:**

```java
public void processShipment(ShipmentId shipmentId, PaymentToken paymentToken) {
    try {
        var shipment = shipmentRepository.findById(shipmentId);
        try {
            var payment = paymentGateway.charge(paymentToken, shipment.getCost());
            try {
                inventoryService.reserveItems(shipment.getItems());
            } catch (InsufficientStockException e) {
                log.error("Stock reservation failed after charging shipment {}", shipmentId, e);
                paymentGateway.refund(payment.getTransactionId());
                throw e;
            }
        } catch (PaymentDeclinedException e) {
            log.warn("Payment declined for shipment {}", shipmentId, e);
            shipment.markAsPaymentFailed();
            shipmentRepository.save(shipment);
            throw e;
        }
    } catch (ShipmentNotFoundException e) {
        log.error("Shipment not found: {}", shipmentId, e);
        throw e;
    }
}
```

Problems: each catch block is visually mixed with unrelated code, the refund logic sits inside a stock exception handler, and the reader must trace all three levels to understand a single flow.

**After - each method owns one failure mode:**

```java
// Top-level: reads as a business narrative - no error handling visible here
public void processShipment(final ShipmentId shipmentId, final PaymentToken paymentToken) {
    var shipment = loadShipment(shipmentId);
    var payment  = chargeForShipment(shipment, paymentToken);
    reserveStock(shipment, payment);
}

private Shipment loadShipment(final ShipmentId shipmentId) {
    try {
        return shipmentRepository.findById(shipmentId);
    } catch (ShipmentNotFoundException e) {
        log.error("Shipment not found: {}", shipmentId, e);
        throw e;
    }
}

private PaymentConfirmation chargeForShipment(final Shipment shipment,
                                               final PaymentToken token) {
    try {
        return paymentGateway.charge(token, shipment.getCost());
    } catch (PaymentDeclinedException e) {
        log.warn("Payment declined for shipment {}", shipment.getId(), e);
        shipment.markAsPaymentFailed();
        shipmentRepository.save(shipment);
        throw e;
    }
}

private void reserveStock(final Shipment shipment, final PaymentConfirmation payment) {
    try {
        inventoryService.reserveItems(shipment.getItems());
    } catch (InsufficientStockException e) {
        log.error("Stock reservation failed after charging shipment {}", shipment.getId(), e);
        paymentGateway.refund(payment.getTransactionId());
        throw e;
    }
}
```

Each method is independently readable and testable. The top-level reads as a narrative. Each failure mode is isolated to the method that can actually handle it.

---

## Custom Exception Hierarchy

Raw `RuntimeException` and `Exception` carry no domain meaning. Define a hierarchy that mirrors the problem space and lets callers catch at the right level of granularity.

```java
// Root of the application exception tree
// Callers who want to catch everything from this domain catch this
public abstract class DeliveryException extends RuntimeException {
    protected DeliveryException(final String message) {
        super(message);
    }
    protected DeliveryException(final String message, final Throwable cause) {
        super(message, cause);
    }
}

// Domain exceptions - represent business rule violations
public class OrderNotFoundException extends DeliveryException {
    public OrderNotFoundException(final OrderId id) {
        super("Order not found: " + id.value());
    }
}

public class InvalidOrderStateException extends DeliveryException {
    public InvalidOrderStateException(final OrderId id,
                                      final OrderStatus current,
                                      final OrderStatus required) {
        super("Order %s is in state %s, expected %s".formatted(id.value(), current, required));
    }
}

public class NoRidersAvailableException extends DeliveryException {
    public NoRidersAvailableException(final Zone zone) {
        super("No riders available in zone: " + zone.name());
    }
}

// Infrastructure exceptions - wrap external failures with domain context
public class PaymentGatewayException extends DeliveryException {
    public PaymentGatewayException(final String reason, final Throwable cause) {
        super("Payment gateway error: " + reason, cause);
    }
}
```


---

## Optional for Complex Domain Chains

`Optional` is most valuable when a chain of lookups may produce no result at any step. The goal is to eliminate intermediate null checks while keeping the logic readable.

**Flat chain - one lookup that may be absent:**

```java
public Optional<RiderSummary> findAssignedRider(final OrderId orderId) {
    return orderRepository.findById(orderId)
        .flatMap(order -> riderRepository.findById(order.getAssignedRiderId()));
}
```

**Deep chain - multiple lookups, each may be absent:**

```java
// What zone is the restaurant of a given order located in?
public Optional<Zone> findRestaurantZone(final OrderId orderId) {
    return orderRepository.findById(orderId)
        .map(Order::getRestaurantId)
        .flatMap(restaurantRepository::findById)
        .map(Restaurant::getAddress)
        .map(Address::getZone);
}
```

Each step is a separate concern. If any returns empty, the entire chain short-circuits and returns `Optional.empty()` - no null checks, no intermediate variables.

**Branching on the result:**

```java
findAssignedRider(orderId)
    .ifPresentOrElse(
        rider -> notifications.riderDetailsUpdated(rider),
        ()    -> log.warn("No rider assigned to order {}", orderId)
    );
```

**Never use Optional as a field or parameter.** It is a return type only. Storing an `Optional` in a field means the field itself can be null, which defeats the purpose.

---

## Null Object Pattern

When a method is always expected to return something - but there is a valid "nothing" state - `Optional` is not always the right tool. The Null Object pattern returns a real object that represents absence, avoiding null checks throughout the call chain.

```java
// Without Null Object: every caller must check for null
Promotion promotion = promotionService.findActiveFor(customer);
if (promotion != null) {
    price = promotion.apply(price);
}

// With Null Object: absence is represented as a real object
public interface Promotion {
    Money apply(Money price);

    // The null object: applies no discount, returns price unchanged
    Promotion NONE = price -> price;
}

// Caller: no null check required - NONE.apply() is safe
Promotion promotion = promotionService.findActiveFor(customer);
Money finalPrice = promotion.apply(basePrice);
```

The `NONE` constant on the interface is a clean way to expose the null object alongside the contract. The service returns `Promotion.NONE` when no promotion is active instead of `null`.

**When to use Null Object vs Optional:**

| Situation | Use |
|---|---|
| Caller always needs to call a method on the result | Null Object |
| Caller needs to decide whether to act at all | `Optional` |
| Result is a collection that might be empty | Empty collection |
| Result is used in a stream chain | `Optional` |