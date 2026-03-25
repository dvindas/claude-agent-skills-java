# Domain-Centric Architecture — Full Worked Example

Read this file when the user needs a complete, end-to-end example of the domain-centric (Option B) structure. This shows the same "Rider Assignment" feature from `layered-example.md` rebuilt with a clean domain, use cases, ports, and infrastructure adapters. Compare the two side by side to understand the tradeoffs.

---

## Feature: Assign a Rider to a Pending Order

Same flow: `POST /orders/{id}/assign-rider` → find the best available rider in the order's zone → assign them.

---

## Package Structure

```
com.example.delivery
└── order/
    ├── domain/
    │   ├── Order.java                      ← entity: owns its own invariants
    │   ├── OrderId.java                    ← value object (record)
    │   ├── OrderStatus.java                ← enum
    │   ├── RiderId.java                    ← value object (record)
    │   ├── Zone.java                       ← value object (record)
    │   ├── OrderRepository.java            ← port: owned by domain
    │   └── RiderAssignmentPort.java        ← port: owned by domain
    ├── application/
    │   ├── AssignRiderToOrder.java         ← use case
    │   └── dto/
    │       ├── AssignRiderCommand.java     ← record
    │       └── AssignRiderResult.java      ← record
    └── infrastructure/
        ├── JpaOrderRepository.java         ← adapter: implements OrderRepository
        ├── OrderJpaEntity.java             ← JPA entity (separate from domain Order)
        ├── JpaRiderAssignment.java         ← adapter: implements RiderAssignmentPort
        ├── RiderJpaEntity.java
        ├── OrderMapper.java                ← maps JPA entity ↔ domain object
        └── OrderController.java            ← adapter: HTTP → use case
```

---

## Domain Layer

Pure Java. No framework imports. The entity enforces its own invariants. Value objects make identity explicit and type-safe.

```java
// Value objects — typed identity prevents passing a RiderId where an OrderId is expected
public record OrderId(UUID value) {
    public OrderId {
        Objects.requireNonNull(value, "OrderId value must not be null");
    }

    public static OrderId of(final String raw) {
        return new OrderId(UUID.fromString(raw));
    }
}

public record RiderId(UUID value) {
    public RiderId {
        Objects.requireNonNull(value, "RiderId value must not be null");
    }
}

public record Zone(String code) {
    public Zone {
        Objects.requireNonNull(code, "Zone code must not be null");
        if (code.isBlank()) throw new IllegalArgumentException("Zone code must not be blank");
    }
}

public enum OrderStatus { PENDING, ASSIGNED, IN_TRANSIT, DELIVERED, CANCELLED }
```

```java
// Entity: all mutation goes through methods that enforce the business rules
// No setter is ever exposed publicly
public class Order {

    private final OrderId id;
    private final Zone zone;
    private OrderStatus status;
    private RiderId assignedRiderId;

    public Order(final OrderId id, final Zone zone, final OrderStatus status) {
        this.id     = Objects.requireNonNull(id);
        this.zone   = Objects.requireNonNull(zone);
        this.status = Objects.requireNonNull(status);
    }

    // Business rule: assignment is only valid from PENDING state
    public void assignTo(final RiderId riderId) {
        Objects.requireNonNull(riderId, "riderId must not be null");
        if (this.status != OrderStatus.PENDING) {
            throw new InvalidOrderStateException(id, status, OrderStatus.PENDING);
        }
        this.assignedRiderId = riderId;
        this.status          = OrderStatus.ASSIGNED;
    }

    public boolean isPending()               { return status == OrderStatus.PENDING; }
    public OrderId getId()                   { return id; }
    public Zone getZone()                    { return zone; }
    public OrderStatus getStatus()           { return status; }
    public Optional<RiderId> getAssignedRiderId() { return Optional.ofNullable(assignedRiderId); }
}
```

```java
// Ports: interfaces owned by the domain — they describe what the domain needs
// The domain has no idea how these are implemented

public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    void save(Order order);
}

// The domain asks for rider assignment capabilities through this port
// It does not know about JPA, Redis, or any concrete data source
public interface RiderAssignmentPort {
    Optional<RiderId> findBestAvailableRider(Zone zone, int maxConcurrentOrders);
    void incrementActiveOrderCount(RiderId riderId);
}
```

---

## Application Layer

Use cases orchestrate domain objects and call ports. No framework types appear here. No Spring annotations.

```java
// Input and output are plain records — no HTTP types, no JSON annotations
public record AssignRiderCommand(OrderId orderId) {
    public AssignRiderCommand {
        Objects.requireNonNull(orderId, "orderId must not be null");
    }
}

public record AssignRiderResult(OrderId orderId, RiderId riderId, OrderStatus status) {}
```

```java
public class AssignRiderToOrder {

    private static final int MAX_CONCURRENT_ORDERS = 3;

    private final OrderRepository orders;
    private final RiderAssignmentPort riderAssignment;

    public AssignRiderToOrder(final OrderRepository orders,
                              final RiderAssignmentPort riderAssignment) {
        this.orders          = Objects.requireNonNull(orders);
        this.riderAssignment = Objects.requireNonNull(riderAssignment);
    }

    public AssignRiderResult execute(final AssignRiderCommand command) {
        var order = orders.findById(command.orderId())
            .orElseThrow(() -> new OrderNotFoundException(command.orderId()));

        var riderId = riderAssignment
            .findBestAvailableRider(order.getZone(), MAX_CONCURRENT_ORDERS)
            .orElseThrow(() -> new NoRidersAvailableException(order.getZone()));

        order.assignTo(riderId);                        // domain enforces the state transition
        riderAssignment.incrementActiveOrderCount(riderId);
        orders.save(order);

        return new AssignRiderResult(order.getId(), riderId, order.getStatus());
    }
}
```

---

## Infrastructure Layer

Adapters implement the ports. They handle JPA, mapping, and Spring concerns. They know about the domain — but the domain knows nothing about them.

**JPA entities — separate from domain classes:**

```java
// The JPA entity lives here, not in the domain
// Changing a column name or adding an index does not touch Order.java
@Entity
@Table(name = "orders")
public class OrderJpaEntity {

    @Id
    private UUID id;

    @Column(nullable = false)
    private String zone;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private String status;

    @Column
    private UUID assignedRiderId;

    // Getters/setters for JPA
}

@Entity
@Table(name = "riders")
public class RiderJpaEntity {

    @Id
    private UUID id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String zone;

    @Column(nullable = false)
    private String status;

    @Column(nullable = false)
    private int activeOrderCount;
}
```

**Mapper — translates between JPA entity and domain object:**

```java
@Component
public class OrderMapper {

    public Order toDomain(final OrderJpaEntity entity) {
        return new Order(
            new OrderId(entity.getId()),
            new Zone(entity.getZone()),
            OrderStatus.valueOf(entity.getStatus())
        );
    }

    public OrderJpaEntity toEntity(final Order order) {
        var entity = new OrderJpaEntity();
        entity.setId(order.getId().value());
        entity.setZone(order.getZone().code());
        entity.setStatus(order.getStatus().name());
        order.getAssignedRiderId().ifPresent(r -> entity.setAssignedRiderId(r.value()));
        return entity;
    }
}
```

**Repository adapter — implements the domain port using Spring Data:**

```java
@Repository
public class JpaOrderRepository implements OrderRepository {

    private final OrderJpaEntityRepository jpa;
    private final OrderMapper mapper;

    public JpaOrderRepository(final OrderJpaEntityRepository jpa,
                               final OrderMapper mapper) {
        this.jpa    = Objects.requireNonNull(jpa);
        this.mapper = Objects.requireNonNull(mapper);
    }

    @Override
    public Optional<Order> findById(final OrderId id) {
        return jpa.findById(id.value()).map(mapper::toDomain);
    }

    @Override
    public void save(final Order order) {
        jpa.save(mapper.toEntity(order));
    }
}

// Spring Data interface — lives here, not in the domain
interface OrderJpaEntityRepository extends JpaRepository<OrderJpaEntity, UUID> {}
```

**Rider assignment adapter:**

```java
@Repository
public class JpaRiderAssignment implements RiderAssignmentPort {

    private final RiderJpaEntityRepository jpa;

    public JpaRiderAssignment(final RiderJpaEntityRepository jpa) {
        this.jpa = Objects.requireNonNull(jpa);
    }

    @Override
    public Optional<RiderId> findBestAvailableRider(final Zone zone,
                                                     final int maxConcurrentOrders) {
        return jpa.findByZoneAndStatus(zone.code(), "AVAILABLE")
            .stream()
            .filter(r -> r.getActiveOrderCount() < maxConcurrentOrders)
            .min(Comparator.comparingInt(RiderJpaEntity::getActiveOrderCount))
            .map(r -> new RiderId(r.getId()));
    }

    @Override
    public void incrementActiveOrderCount(final RiderId riderId) {
        jpa.findById(riderId.value()).ifPresent(rider -> {
            rider.setActiveOrderCount(rider.getActiveOrderCount() + 1);
            jpa.save(rider);
        });
    }
}

interface RiderJpaEntityRepository extends JpaRepository<RiderJpaEntity, UUID> {
    List<RiderJpaEntity> findByZoneAndStatus(String zone, String status);
}
```

**Controller adapter — translates HTTP to use case call:**

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final AssignRiderToOrder assignRiderToOrder;

    public OrderController(final AssignRiderToOrder assignRiderToOrder) {
        this.assignRiderToOrder = Objects.requireNonNull(assignRiderToOrder);
    }

    @PostMapping("/{id}/assign-rider")
    public ResponseEntity<AssignRiderResponse> assignRider(@PathVariable final String id) {
        var result   = assignRiderToOrder.execute(new AssignRiderCommand(OrderId.of(id)));
        var response = new AssignRiderResponse(
            result.orderId().value().toString(),
            result.riderId().value().toString(),
            result.status().name()
        );
        return ResponseEntity.ok(response);
    }
}

// Response DTO — separate from the domain result; API contract evolves independently
public record AssignRiderResponse(String orderId, String riderId, String status) {}
```

---

## Unit Test — No Spring, No Database

Because `AssignRiderToOrder` depends on interfaces, the test uses in-memory doubles. It runs in milliseconds.

```java
class AssignRiderToOrderTest {

    @Test
    void assigns_the_rider_with_fewest_active_orders() {
        final var orderId  = new OrderId(UUID.randomUUID());
        final var zone     = new Zone("NORTH");
        final var order    = new Order(orderId, zone, OrderStatus.PENDING);
        final var riderId  = new RiderId(UUID.randomUUID());

        final var orders   = new InMemoryOrderRepository(order);
        final var riders   = new StubRiderAssignment(riderId);
        final var useCase  = new AssignRiderToOrder(orders, riders);

        var result = useCase.execute(new AssignRiderCommand(orderId));

        assertThat(result.status()).isEqualTo(OrderStatus.ASSIGNED);
        assertThat(result.riderId()).isEqualTo(riderId);
        assertThat(orders.findById(orderId))
            .flatMap(Order::getAssignedRiderId)
            .hasValue(riderId);
    }

    @Test
    void throws_when_order_is_not_pending() {
        final var orderId = new OrderId(UUID.randomUUID());
        final var order   = new Order(orderId, new Zone("NORTH"), OrderStatus.ASSIGNED);
        final var orders  = new InMemoryOrderRepository(order);
        final var riders  = new StubRiderAssignment(new RiderId(UUID.randomUUID()));
        final var useCase = new AssignRiderToOrder(orders, riders);

        assertThatThrownBy(() -> useCase.execute(new AssignRiderCommand(orderId)))
            .isInstanceOf(InvalidOrderStateException.class);
    }

    // --- Test doubles ---

    static class InMemoryOrderRepository implements OrderRepository {
        private final Map<OrderId, Order> store = new HashMap<>();

        InMemoryOrderRepository(final Order... orders) {
            Arrays.stream(orders).forEach(o -> store.put(o.getId(), o));
        }

        @Override public Optional<Order> findById(final OrderId id) { return Optional.ofNullable(store.get(id)); }
        @Override public void save(final Order order)               { store.put(order.getId(), order); }
    }

    static class StubRiderAssignment implements RiderAssignmentPort {
        private final RiderId riderId;
        StubRiderAssignment(final RiderId riderId) { this.riderId = riderId; }

        @Override public Optional<RiderId> findBestAvailableRider(Zone zone, int max) { return Optional.of(riderId); }
        @Override public void incrementActiveOrderCount(RiderId riderId) { /* no-op in test */ }
    }
}
```

---

## Enforcing Boundaries with ArchUnit

Add this test once per module. It fails the build if any domain class imports from infrastructure.

```java
@AnalyzeClasses(packages = "com.example.delivery")
public class ArchitectureTest {

    @ArchTest
    static final ArchRule domainMustNotDependOnInfrastructure =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure..");

    @ArchTest
    static final ArchRule applicationMustNotDependOnInfrastructure =
        noClasses()
            .that().resideInAPackage("..application..")
            .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure..");
}
```

This makes the Dependency Rule a compile-time guarantee rather than a convention.