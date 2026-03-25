# Naming - Deep Examples and Edge Cases

Read this file when the user needs depth on naming: difficult real-world cases, anti-pattern diagnosis, or naming decisions that aren't covered by the quick rules in SKILL.md.

---

## Naming by Layer

Names carry different expectations depending on where they live. A domain class that is honest about what it is requires less ceremony than a utility buried in infrastructure.

**Domain layer** - names should read like business language. Avoid technical suffixes that add no meaning.

```java
// Avoid: suffix adds nothing to the name
class CustomerEntity { }
class OrderData { }

// Prefer: name the concept, not the role
class Customer { }
class Order { }
```

`DTO` is an acceptable suffix when the class is explicitly a data transfer object crossing a boundary, signaling intent to the reader. `ProductDTO`, `OrderDTO`, and similar are fine. Avoid it on domain classes that are not DTOs.

**Application / service layer** - names describe actions, not mechanisms. The `Service` suffix is acceptable and widely understood for classes that orchestrate business logic.

```java
// Avoid: mechanism-focused, vague
class OrderProcessor { }
class InvoiceHandler { }

// Acceptable: Service suffix signals the role clearly
class OrderService { }
class InvoiceService { }

// Also good: intent-focused use case names
class PlaceOrder { }
class GenerateInvoice { }
```

**Infrastructure layer** - names can and should reveal the technology, because that is relevant here.

```java
// Good: technology name is informative in this layer
class KafkaShipmentEventPublisher implements ShipmentEventPort { }
class SmtpEmailNotifier implements NotificationPort { }
```

---

## Naming Collections

Collections should be named for what they contain, in the plural - never for how they are structured.

```java
// Avoid: structure leaks into the name
List<Order> orderList;
Set<String> emailSet;
Map<Long, Rider> riderMap;

// Prefer: the name says what it holds
List<Order> orders;
Set<String> emails;
Map<Long, Rider> ridersById;   // "ById" is acceptable when the key matters
```

When the key is the meaningful part, append the grouping criterion:

```java
Map<Zone, List<Rider>> ridersByZone;
Map<CustomerId, List<Order>> ordersByCustomer;
```

---

## Naming Booleans

Boolean names should read as a yes/no question. The answer to the question is the value.

```java
// Avoid: doesn't read as a question
boolean active;
boolean flag;
boolean check;
boolean valid;

// Prefer: reads as a question with a clear yes/no answer
boolean isActive;
boolean hasOpenOrders;
boolean canAcceptRiders;
boolean wasDelivered;
```

Consistency matters. Choose one prefix per concept and stick to it across the codebase:

| Prefix | Use for |
|---|---|
| `is` | State: `isActive`, `isPending`, `isCancelled` |
| `has` | Possession: `hasDiscount`, `hasOpenOrders` |
| `can` | Permission or capability: `canBeReassigned`, `canAcceptPayment` |
| `was` | Past events: `wasDelivered`, `wasProcessed`, `wasCancelled` |

---

## When Abbreviations Are Acceptable

Abbreviations are acceptable only when they are **universally understood within the domain** and the full form would be actively worse.

```java
// Universally understood - abbreviation is the canonical name
CustomerId id;
String url;
Map<String, Object> dto;
UUID uuid;

// Domain-specific and well-established within the team
BigDecimal vat;     // Value Added Tax - if the whole codebase uses it
String iban;        // International Bank Account Number - same rule
```

Abbreviations that require decoding are not acceptable, even if the team knows them:

```java
// Avoid: saving 3 characters at the cost of clarity
int qty;           // quantity
String addr;       // address
BigDecimal amt;    // amount
boolean flg;       // flag

// Prefer: spell it out
int quantity;
String address;
BigDecimal amount;
boolean isEligible;
```

---

## Constants: Name the Significance, Not the Value

A constant name should explain why the value matters, not just what the value is.

```java
// Avoid: the name describes the value, not its purpose
private static final int THIRTY = 30;
private static final double POINT_TWO_ONE = 0.21;
private static final String ADMIN = "ADMIN";

// Prefer: the name explains what this value controls
private static final int SESSION_TIMEOUT_MINUTES = 30;
private static final double VAT_RATE = 0.21;
private static final String ROLE_ADMIN = "ADMIN";
```

When multiple constants are related, group them and prefix consistently:

```java
// Fragmented - no signal they belong together
private static final int MIN_CREDIT_SCORE = 650;
private static final int LOAN_REVIEW_THRESHOLD = 700;
private static final int PREMIUM_CREDIT_SCORE = 750;

// Better: the prefix signals they belong to the same policy
private static final int CREDIT_SCORE_MINIMUM         = 650;
private static final int CREDIT_SCORE_REVIEW_THRESHOLD = 700;
private static final int CREDIT_SCORE_PREMIUM          = 750;
```

---

## Diagnosing a Bad Name

When a name feels wrong but you can't articulate why, run these checks:

| Question | If yes → |
|---|---|
| Does it need a comment to explain what it is? | Rename it |
| Could it refer to two different things? | Make it more specific |
| Does it contain `data`, `info`, `manager`, `processor`, or `handler`? | Prefer a more specific name when one exists |
| Is it an abbreviation someone might not know? | Spell it out |
| Is it a verb when it should be a noun (or vice versa)? | Fix the part of speech |
| Does the name promise purity but the implementation has side effects? | Rename or split |