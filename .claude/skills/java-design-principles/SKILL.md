---
name: java-design-principles
description: >
  Design and review Java applications following structural design principles: Dependency Rule, Open/Closed,
  Single Responsibility, Inversion of Control, Dependency Injection, transaction boundaries, and project structure.
  Use this skill whenever structuring a new Java application or module, reviewing how layers depend on each other,
  deciding between a layered or domain-centric project structure, or when the user mentions any of these:
  project structure, package organization, layered architecture, hexagonal architecture, dependency inversion,
  IoC container, dependency injection, transaction management, business logic inside controllers or repositories,
  or coupling between infrastructure and domain. Also trigger when the user says things like "how should I
  structure my project", "where should this logic go", "how do I separate concerns", "is this the right layer
  for this", "should I use constructor injection", "where does @Transactional go", or any question about what
  belongs in which layer. Also trigger on: "SRP", "OCP", "IoC", "DI", "dependency rule", "inversion of control",
  "should my service depend on my repository", "anemic domain model".
license: MIT
metadata:
  author: dvindas
  version: '1.0.0'
---

# Java Clean Architecture

Apply these principles when designing or reviewing the structure of a Java application or module.

## Quick Reference

| Principle | Short form |
|---|---|
| Dependency Rule | Dependencies point inward: outer layers depend on inner layers, never the reverse |
| Single Responsibility | Each class has one reason to change |
| Open/Closed | Open for extension, closed for modification — add behavior through new code, not edits |
| Inversion of Control | High-level modules do not control low-level modules; a container or framework manages object lifecycles |
| Dependency Injection | Collaborators are supplied from outside, not created inside the class that uses them |
| Layered structure | Suitable for small/medium apps: Controller → Service → Repository → Model |
| Domain-centric structure | Suitable for complex domains: Infrastructure wraps Application, which wraps Domain |

---

## Guiding Philosophy

The central idea behind Clean Architecture is **control over change.** Business rules should be able to evolve without touching the HTTP layer. The database should be swappable without touching the domain. Frameworks should serve the application — not the other way around.

Every principle in this skill is a consequence of that idea. SRP keeps each layer focused so it changes for only one reason. The Dependency Rule ensures inner layers never know what wraps them. OCP lets you extend behavior without editing stable code. IoC and DI together make those dependencies replaceable.

Two questions guide every structural decision:

**Who depends on whom?** Dependencies should flow toward stability. The domain is the most stable part — it should not depend on the controller, the database driver, or the HTTP framework. If a core business class imports something from a web or persistence framework, that dependency is pointing the wrong direction.

**Could I replace this with a different implementation without touching anything else?** If switching from PostgreSQL to MongoDB requires editing service classes, the repository is not properly abstracted. If adding a new delivery zone fee requires modifying the pricing class, the OCP is being violated.

---

## Part 1 — The Dependency Rule

**Source code dependencies must point inward.** Outer layers may depend on inner layers. Inner layers must never depend on outer layers. The domain — entities and business rules — is the center. HTTP, persistence, and messaging wrap around it.

```
  Controller → Service → Repository
       ↓           ↓          ↓
               Domain Model
```

The practical consequence: a domain class has **zero imports** from Spring, JPA, Jackson, or any framework. It does not know it is being called over HTTP or that its state is stored in a relational table.

```java
// Violation: domain entity knows about JPA and JSON serialization
// Any change to the column name or JSON key forces touching this class
@Entity
@Table(name = "delivery_orders")
public class DeliveryOrder {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String status;

    @JsonProperty("customer_id")
    private Long customerId;
}

// Clean: pure domain class — no framework annotations
// Value objects make identity and money explicit and type-safe
public class DeliveryOrder {

    private final DeliveryOrderId id;
    private final CustomerId customerId;
    private DeliveryStatus status;
    private Money deliveryFee;

    public void accept() {
        if (this.status != DeliveryStatus.PENDING) {
            throw new InvalidOrderStateException(id, status, DeliveryStatus.PENDING);
        }
        this.status = DeliveryStatus.ACCEPTED;
    }

    public boolean isPending() {
        return this.status == DeliveryStatus.PENDING;
    }
}

// Value objects: typed identity prevents passing a CustomerId where a DeliveryOrderId is expected
public record DeliveryOrderId(UUID value) {
    public DeliveryOrderId {
        Objects.requireNonNull(value, "id must not be null");
    }
}

public record CustomerId(UUID value) {
    public CustomerId {
        Objects.requireNonNull(value, "customerId must not be null");
    }
}

public record Money(BigDecimal amount, Currency currency) {
    public Money {
        Objects.requireNonNull(amount, "amount must not be null");
        Objects.requireNonNull(currency, "currency must not be null");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative: " + amount);
        }
    }
}
```

The JPA mapping belongs in a separate infrastructure class — a `DeliveryOrderJpaEntity` that maps to the table and converts to/from the domain `DeliveryOrder`. That class lives in the outer layer and its dependency arrow points inward toward the domain, never the reverse.

---

## Part 2 — Single Responsibility Principle (SRP)

**A class should have one reason to change.** When a class handles HTTP parsing, business logic, and database access together, it changes whenever any of those three concerns change. Each concern should live in a class dedicated to it.

```java
// Violation: controller owns validation, business rule, persistence, and notification
@PostMapping("/orders/{id}/accept")
public ResponseEntity<Void> accept(@PathVariable String id) {
    var order = entityManager.find(DeliveryOrderJpaEntity.class, UUID.fromString(id));
    if (!order.getStatus().equals("PENDING")) return ResponseEntity.badRequest().build();
    order.setStatus("ACCEPTED");
    entityManager.merge(order);
    notificationClient.push(order.getRiderId(), "New order available");
    return ResponseEntity.ok().build();
}

// Clean: each class has one reason to change
// Controller: translates HTTP → command only
@PostMapping("/orders/{id}/accept")
public ResponseEntity<Void> accept(@PathVariable final String id) {
    acceptDeliveryOrder.execute(new AcceptOrderCommand(DeliveryOrderId.of(id)));
    return ResponseEntity.ok().build();
}

// Use case: orchestrates — calls domain and ports, no HTTP or JPA types
public void execute(final AcceptOrderCommand command) {
    var order = orders.findById(command.orderId())
        .orElseThrow(() -> new DeliveryOrderNotFoundException(command.orderId()));
    order.accept();           // business rule lives in the entity
    orders.save(order);
    notifications.orderAvailable(order);
}

// Entity: enforces its own invariants — no service needed to guard the transition
public void accept() {
    if (this.status != DeliveryStatus.PENDING)
        throw new InvalidOrderStateException(id, status, DeliveryStatus.PENDING);
    this.status = DeliveryStatus.ACCEPTED;
}
```

> **Caveat:** SRP is a signal, not a mechanical splitting rule. A class with many methods that all serve one unified purpose should stay together. The question is not "does this class have many methods?" but "does it change for more than one reason in practice?"

---

## Part 3 — Open/Closed Principle (OCP)

**A class should be open for extension and closed for modification.** When adding new behavior requires editing a stable, working class, every existing test and caller is at risk. The goal is to add new code — not change old code.

The typical mechanism in Java is an interface that defines a contract. New behavior arrives as a new implementation, not as a new `if/else` branch inside the existing one.

```java
// Violation: every new zone or time rule requires editing this class
// Adding "airport surcharge" means touching code that already works
public class DeliveryFeeCalculator {

    public Money calculate(DeliveryOrder order) {
        if (order.zone() == Zone.CITY_CENTER) {
            return Money.of(2.50);
        } else if (order.zone() == Zone.SUBURBS) {
            return Money.of(4.00);
        } else if (order.isPeakHour()) {      // added later — required editing this class
            return Money.of(6.00);
        }
        return Money.of(5.00);
    }
}

// Clean: new fee rules arrive as new classes — DeliveryFeeCalculator never changes
public interface DeliveryFeePolicy {
    boolean appliesTo(DeliveryOrder order);
    Money calculate(DeliveryOrder order);
}

// Each variant is its own class: CityZonePolicy, PeakHoursPolicy, AirportSurchargePolicy...
// Adding a new zone means adding a new class — zero edits to existing code
public class CityZonePolicy implements DeliveryFeePolicy {
    @Override public boolean appliesTo(final DeliveryOrder order) { return order.zone() == Zone.CITY_CENTER; }
    @Override public Money calculate(final DeliveryOrder order)   { return Money.of(new BigDecimal("2.50"), USD); }
}

public class DeliveryFeeCalculator {

    private final List<DeliveryFeePolicy> policies;

    public DeliveryFeeCalculator(final List<DeliveryFeePolicy> policies) {
        this.policies = List.copyOf(Objects.requireNonNull(policies));
    }

    public Money calculate(final DeliveryOrder order) {
        return policies.stream()
            .filter(p -> p.appliesTo(order))
            .findFirst()
            .map(p -> p.calculate(order))
            .orElse(Money.of(new BigDecimal("5.00"), Currency.getInstance("USD")));
    }
}
```

`DeliveryFeeCalculator` did not change when `AirportSurchargePolicy` was added. It was closed for modification and open for extension through the `DeliveryFeePolicy` interface. This also pairs naturally with Dependency Injection: which policies are active is decided externally, not hardcoded inside the calculator.

Sealed interfaces are a natural companion to OCP for closed hierarchies — when the set of variants is fixed and you want the compiler to enforce exhaustive handling:

```java
// All payment outcomes are known — sealed prevents accidental open extension
public sealed interface PaymentResult
    permits PaymentResult.Approved, PaymentResult.Declined, PaymentResult.Pending {}

public record Approved(String transactionId, Instant processedAt) implements PaymentResult {}
public record Declined(String reason)                              implements PaymentResult {}
public record Pending(String referenceCode)                        implements PaymentResult {}

// Switch is exhaustive — the compiler rejects incomplete handling
String message = switch (result) {
    case Approved a -> "Payment confirmed: " + a.transactionId();
    case Declined d -> "Payment failed: " + d.reason();
    case Pending p  -> "Awaiting confirmation: " + p.referenceCode();
};
```

---

## Part 4 — Inversion of Control (IoC)

### The Big Idea: a System of Contracts

At a high level, a well-designed system is a graph of **contracts talking to each other** — not a graph of concrete classes. Each component declares what it needs through an interface. No component knows or cares which concrete class will fulfil that contract at runtime. The wiring — deciding which implementation goes where — happens once, externally, and can change at any time without touching the components themselves.

```
┌─────────────────────┐        ┌──────────────────────────┐
│  AcceptDeliveryOrder │───────▶│  DeliveryOrderRepository  │  ← contract (interface)
│     (use case)       │        └──────────────────────────┘
└─────────────────────┘                    ▲
                                           │ implements
                              ┌────────────┴──────────────┐
                              │  JpaDeliveryOrderRepository│  ← runtime implementation
                              └───────────────────────────┘
```

`AcceptDeliveryOrder` knows about `DeliveryOrderRepository` — the contract. It has no idea that `JpaDeliveryOrderRepository` exists. At runtime, the IoC container (Spring) wires the concrete class in. In a test, the test wires in an in-memory double instead. The use case never changes — only the wiring does.

This is the inversion: instead of a class pulling in its own dependencies with `new`, control over which implementation is used is handed to something outside the class.

### Without IoC — classes in control of their own dependencies

```java
// AcceptDeliveryOrder decides what it uses and how it creates it
// Swapping JpaDeliveryOrderRepository for anything else means editing this class
public class AcceptDeliveryOrder {

    private final DeliveryOrderRepository orders;
    private final RiderNotificationPort notifications;

    public AcceptDeliveryOrder() {
        this.orders        = new JpaDeliveryOrderRepository();  // hardcoded, untestable
        this.notifications = new FirebasePushNotification();    // hardcoded, untestable
    }
}
```

### With IoC — the container is in control

```java
// AcceptDeliveryOrder declares what contracts it needs
// It never decides which implementation fulfils them — that is not its job
public class AcceptDeliveryOrder {

    private final DeliveryOrderRepository orders;
    private final RiderNotificationPort notifications;

    public AcceptDeliveryOrder(final DeliveryOrderRepository orders,
                               final RiderNotificationPort notifications) {
        this.orders        = Objects.requireNonNull(orders);
        this.notifications = Objects.requireNonNull(notifications);
    }
}
```

Spring reads the constructor, sees the interfaces, and injects whatever implementation is registered for each contract. The use case is identical in production, in a test, and in a future migration — only the wiring differs.

### Swapping implementations without touching business logic

Because every component depends on a contract, not a class, the implementation behind any contract can be replaced freely:

```java
// Production wiring — Spring picks this up automatically via @Repository
@Repository
public class JpaDeliveryOrderRepository implements DeliveryOrderRepository { ... }

// In a test — pass directly through the constructor, no Spring context needed
var orders   = new InMemoryDeliveryOrderRepository();
var notifier = new SpyRiderNotification();
var useCase  = new AcceptDeliveryOrder(orders, notifier);

// Migrating from Firebase to SNS — only the adapter changes, nothing else
@Component
public class SnsRiderNotification implements RiderNotificationPort { ... }
```

All three scenarios — production JPA, in-memory test double, new SNS notification — wire into the exact same `AcceptDeliveryOrder`. No edits to the use case, no edits to the domain. The system is a graph of stable contracts; implementations are pluggable details.

---

## Part 5 — Dependency Injection (DI)

**Collaborators are supplied from outside the class that uses them.** DI is the most common way to implement IoC. Three styles exist in Spring:

| Style | When to use |
|---|---|
| Constructor injection | Preferred — required dependencies, supports `final` fields, makes contract explicit |
| Setter injection | Optional dependencies or when a framework requires a no-arg constructor |
| Field injection (`@Autowired`) | Avoid — hides the contract, prevents `final`, and makes unit testing harder |

**Always prefer constructor injection with `final` fields.**

```java
// Avoid: field injection hides the dependency contract
// You cannot tell from the outside what this class needs
// Fields cannot be final — state may change after construction
@Service
public class OrderAssignmentService {

    @Autowired
    private DeliveryOrderRepository orderRepository;  // hidden

    @Autowired
    private RiderAvailabilityPort riderAvailability;  // hidden
}

// Prefer: constructor injection — contract is visible, fields are immutable
// Spring auto-wires the single constructor; @Autowired annotation is not required
@Service
public class OrderAssignmentService {

    private final DeliveryOrderRepository orderRepository;
    private final RiderAvailabilityPort riderAvailability;

    public OrderAssignmentService(final DeliveryOrderRepository orderRepository,
                                  final RiderAvailabilityPort riderAvailability) {
        this.orderRepository   = Objects.requireNonNull(orderRepository);
        this.riderAvailability = Objects.requireNonNull(riderAvailability);
    }
}
```

Constructor injection also makes unit testing straightforward — pass a test double through the constructor, no Spring context or reflection required:

```java
class AcceptDeliveryOrderTest {

    @Test
    void transitions_order_to_accepted_state() {
        final var order       = pendingDeliveryOrder();
        final var orders      = new InMemoryDeliveryOrderRepository(order);
        final var notifier    = new SpyRiderNotification();
        final var useCase     = new AcceptDeliveryOrder(orders, notifier);

        useCase.execute(new AcceptOrderCommand(order.getId()));

        assertThat(orders.findById(order.getId()))
            .map(DeliveryOrder::getStatus)
            .hasValue(DeliveryStatus.ACCEPTED);
        assertThat(notifier.wasNotifiedFor(order.getId())).isTrue();
    }
}
```

`InMemoryDeliveryOrderRepository` is a test double that implements `DeliveryOrderRepository` using a `Map<DeliveryOrderId, DeliveryOrder>`. It runs in microseconds, requires no database setup, and is reusable across all use case tests.

**Inject interfaces, not concrete classes.** The injection point should declare the abstraction, not the implementation:

```java
// Avoid: tied to the JPA implementation — cannot swap without changing the constructor
public AcceptDeliveryOrder(final JpaDeliveryOrderRepository orders, ...) { }

// Prefer: tied to the contract — any implementation satisfies it
public AcceptDeliveryOrder(final DeliveryOrderRepository orders, ...) { }
```

The port interface `DeliveryOrderRepository` lives in the domain layer. The JPA implementation lives in the infrastructure layer. The dependency arrow points inward — this is the Dependency Rule and DI working together.

> For the complete port interface + JPA adapter + mapper implementation, see `references/domain-centric-example.md`.

---

## Part 6 — Transaction Boundaries

**`@Transactional` belongs at the use-case layer, not the repository layer.**

A single business operation often spans multiple repository calls that must succeed or fail together. If each repository method manages its own transaction, those calls run in separate transactions — a failure midway leaves the database in a partially-updated state.

```java
// Violation: @Transactional on the repository
// Each save() is its own transaction — no atomicity across operations
@Repository
public class JpaOrderRepository implements OrderRepository {
    @Transactional          // ← too low; only covers this one call
    public void save(Order order) { ... }
}

// Elsewhere, two independent transactions — not atomic:
orderRepository.save(order);   // transaction 1 commits
riderRepository.save(rider);   // transaction 2 — if this fails, order is already saved
```

```java
// Clean: @Transactional at the use-case boundary wraps the entire operation
@Transactional                  // ← one transaction for the whole business operation
public AssignRiderResult execute(final AssignRiderCommand command) {
    var order = orders.findById(command.orderId()).orElseThrow();
    var rider = riderAssignment.findBestAvailable(order.getZone()).orElseThrow();
    order.assignTo(rider);
    orders.save(order);         // same transaction
    riderAssignment.increment(rider.getId()); // same transaction — atomic
    return new AssignRiderResult(order.getId(), rider.getId());
}
```

**Practical rules:**
- Annotate use-case classes (or their public `execute` methods) with `@Transactional`
- Keep repositories free of `@Transactional` — they participate in whatever transaction is already open
- Use `@Transactional(readOnly = true)` on query-only use cases for a performance hint to the JPA provider
- Never call `@Transactional` methods from within the same class — Spring's proxy cannot intercept internal calls

---

## Part 6 — Project Structure

Two layouts are common in Java projects. Choose based on domain complexity.

**Option A — Layered:** organize by technical role (`controller/`, `service/`, `repository/`, `model/`, `dto/`). Dependency direction: `Controller → Service → Repository → Model`. Simple to navigate, fast to scaffold, familiar to every Java developer. The tradeoff: adding a feature touches every package, business logic accumulates in services, and testing usually requires Spring.

**Option B — Domain-Centric:** organize by feature first, then by role within the feature (`domain/`, `application/`, `infrastructure/`). Dependency direction: `Infrastructure → Application → Domain`. The domain owns the port interfaces; infrastructure implements them. Business rules live in entities; use cases orchestrate without knowing about HTTP or JPA. Unit tests need no framework.

| Consideration | Layered (A) | Domain-centric (B) |
|---|---|---|
| Domain complexity | Low to medium | Medium to high |
| Framework independence | Not required | Required |
| Testability of business logic | Spring context often needed | Plain unit tests |
| Feature-level isolation | Low | High |

You can start with Option A and migrate toward Option B as the domain grows: extract repository interfaces first → move business rules into entities → separate JPA entities from domain classes → enforce boundaries with ArchUnit.

> For complete, end-to-end worked examples of both structures using the same feature, read:
> - `references/layered-example.md` — every class in a layered project, with tradeoff annotations
> - `references/domain-centric-example.md` — domain, use cases, adapters, mapper, test doubles, and ArchUnit rules
