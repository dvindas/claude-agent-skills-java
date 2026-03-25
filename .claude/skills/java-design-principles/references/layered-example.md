# Layered Architecture — Full Worked Example

Read this file when the user needs a complete, end-to-end example of the layered (Option A) structure. This shows every class in a single feature: "Rider Assignment" for a food delivery platform.

---

## Feature: Assign a Rider to a Pending Order

The flow: an operator calls `POST /orders/{id}/assign-rider`. The system finds the best available rider in the order's zone and assigns them.

---

## Package Structure

```
com.example.delivery
├── controller/
│   └── OrderController.java
├── service/
│   └── OrderService.java
├── repository/
│   ├── OrderRepository.java
│   └── RiderRepository.java
├── model/
│   ├── Order.java
│   ├── OrderStatus.java
│   ├── Rider.java
│   └── RiderStatus.java
└── dto/
    ├── AssignRiderRequest.java
    └── AssignRiderResponse.java
```

---

## Model Layer

Plain JPA entities. Annotations and domain data live together in the same class — acceptable in the layered style, where framework independence is not a goal.

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status;

    @Column(nullable = false)
    private String zone;

    @Column
    private UUID assignedRiderId;

    // Getters and setters (or use Lombok @Data in a real project)
    public UUID getId()                        { return id; }
    public OrderStatus getStatus()             { return status; }
    public String getZone()                    { return zone; }
    public UUID getAssignedRiderId()           { return assignedRiderId; }
    public void setAssignedRiderId(UUID riderId) { this.assignedRiderId = riderId; }
    public void setStatus(OrderStatus status)  { this.status = status; }
}

public enum OrderStatus { PENDING, ASSIGNED, IN_TRANSIT, DELIVERED, CANCELLED }

@Entity
@Table(name = "riders")
public class Rider {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String zone;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private RiderStatus status;

    @Column(nullable = false)
    private int activeOrderCount;

    public UUID getId()              { return id; }
    public String getName()          { return name; }
    public String getZone()          { return zone; }
    public RiderStatus getStatus()   { return status; }
    public int getActiveOrderCount() { return activeOrderCount; }
    public void setActiveOrderCount(int count) { this.activeOrderCount = count; }
}

public enum RiderStatus { AVAILABLE, BUSY, OFFLINE }
```

---

## Repository Layer

Spring Data JPA repositories. The interface is defined here and lives in the same package as the other repository interfaces.

```java
public interface OrderRepository extends JpaRepository<Order, UUID> {
    List<Order> findByZoneAndStatus(String zone, OrderStatus status);
}

public interface RiderRepository extends JpaRepository<Rider, UUID> {
    List<Rider> findByZoneAndStatus(String zone, RiderStatus status);
}
```

---

## DTO Layer

Plain records for request/response shapes. These decouple the API contract from the model.

```java
public record AssignRiderRequest(UUID orderId) {}

public record AssignRiderResponse(
    UUID orderId,
    UUID riderId,
    String riderName,
    String status
) {}
```

---

## Service Layer

Business logic lives here. The service orchestrates the assignment: validates state, finds the best rider, updates both entities, and returns the result.

```java
@Service
@Transactional
public class OrderService {

    private static final int MAX_CONCURRENT_ORDERS = 3;

    private final OrderRepository orderRepository;
    private final RiderRepository riderRepository;

    public OrderService(final OrderRepository orderRepository,
                        final RiderRepository riderRepository) {
        this.orderRepository = orderRepository;
        this.riderRepository = riderRepository;
    }

    public AssignRiderResponse assignRider(final UUID orderId) {
        var order = orderRepository.findById(orderId)
            .orElseThrow(() -> new EntityNotFoundException("Order not found: " + orderId));

        if (order.getStatus() != OrderStatus.PENDING) {
            throw new IllegalStateException(
                "Order %s cannot be assigned — current status: %s".formatted(orderId, order.getStatus())
            );
        }

        var rider = findBestAvailableRider(order.getZone());

        order.setAssignedRiderId(rider.getId());
        order.setStatus(OrderStatus.ASSIGNED);
        rider.setActiveOrderCount(rider.getActiveOrderCount() + 1);

        orderRepository.save(order);
        riderRepository.save(rider);

        return new AssignRiderResponse(order.getId(), rider.getId(), rider.getName(), "ASSIGNED");
    }

    private Rider findBestAvailableRider(final String zone) {
        return riderRepository.findByZoneAndStatus(zone, RiderStatus.AVAILABLE)
            .stream()
            .filter(r -> r.getActiveOrderCount() < MAX_CONCURRENT_ORDERS)
            .min(Comparator.comparingInt(Rider::getActiveOrderCount))
            .orElseThrow(() -> new IllegalStateException("No available riders in zone: " + zone));
    }
}
```

---

## Controller Layer

HTTP entry point. Translates the request, delegates to the service, returns the response.

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(final OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping("/{id}/assign-rider")
    public ResponseEntity<AssignRiderResponse> assignRider(@PathVariable final UUID id) {
        var response = orderService.assignRider(id);
        return ResponseEntity.ok(response);
    }
}
```

---

## What Works Well Here

- **Simple to navigate.** Any Java developer knows where to look: controller calls service, service calls repository.
- **Fast to scaffold.** A new CRUD feature adds one class per layer; Spring handles the wiring.
- **Spring Data saves time.** `findByZoneAndStatus` is derived from the method name — no SQL needed.

## Where This Starts to Break Down

- **Business logic lives in the service, not the model.** `order.setStatus(ASSIGNED)` is a mutation with no guard. Any caller can set any status without the entity objecting. As the domain grows, these unchecked mutations multiply.
- **Feature coupling across packages.** Adding "automatic reassignment on rider dropout" touches `OrderService`, `RiderRepository`, and potentially `OrderController` — three packages for one conceptual change.
- **Testing requires Spring.** `OrderService` depends on `OrderRepository` (the JPA interface). Testing it without a database requires either an embedded H2 or a Mockito mock of a JPA interface — both are fragile.
- **The model carries two concerns.** `Order` is both a JPA entity and a domain object. Changing the column name changes the serialization contract. Adding behavior is awkward next to persistence annotations.

These are the signals to start migrating toward the domain-centric structure (see `domain-centric-example.md`).