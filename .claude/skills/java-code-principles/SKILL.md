---
name: java-code-principles
description: >
  Write and review Java code following established code quality principles, largely inspired by Robert C. Martin's
  (Uncle Bob) Clean Code, adapted to modern Java and current practices. Covers naming, functions, comments,
  classes, error handling, formatting, objects vs data structures, and DRY, with original Java examples using
  records, sealed types, Optional, and pattern matching. Use this skill whenever writing new Java classes or
  methods, reviewing existing code for readability, or when the user mentions any of these: bad naming, unclear
  intent, long methods, too many arguments, magic numbers, flag arguments, hidden side effects, train wrecks,
  null returns, nested try/catch, duplicated logic, a class doing too much, anemic model, objects vs data
  structures, or code that is hard to understand or maintain. Also trigger when the user says things like
  "clean this up", "this is messy", "review my code", "refactor this", or "how should I structure this class".
  Also trigger on: "code smell", "this smells", "is this good code", "how do I improve this", "what's wrong
  with this code", "this feels wrong", "is this readable", or any request for a code quality opinion.
license: MIT
metadata:
  author: dvindas
  version: '1.0.0'
---

# Java Code Principles

Apply these principles when writing new Java code or reviewing existing code.

## Quick Reference

| Rule | Short form |
|---|---|
| Name for intent | Every name answers: why it exists, what it does, how it is used |
| No magic numbers | Extract to named `static final` constants |
| No abbreviations / noise words | `calculateTotal`, not `calcTtl`; `Customer`, not `CustomerData` |
| No encodings | No Hungarian notation, `m_` prefixes, or `IFoo` markers |
| One purpose per method | If you need "and" to describe it, split it |
| Cohesive methods | Cohesive and focused; extract chunks into well-named helpers |
| Stepdown rule | Each method delegates the *how* to helpers one level below |
| 0–2 arguments ideal | 3 acceptable; 4+ → introduce a parameter object (or `record`) |
| No output arguments | Don't mutate a parameter as a return channel; return a value instead |
| No flag arguments | Boolean that changes behavior → split into two methods |
| No hidden side effects | A method named `calculate` must not silently mutate state |
| Command Query Separation | A method either acts or answers - never both |
| Prefer code over comments | Comments explain *why*; rename when the comment explains *what* |
| Single Responsibility | A class or method has one reason to change |
| Law of Demeter | Talk to friends, not strangers; avoid train wrecks |
| No null returns | `Optional<T>` for optional results; empty collection for collections |
| No null arguments | Use overloads or the Null Object pattern |
| Specific exception types | Specific over generic; `Exception`/`RuntimeException` only when no suitable type exists |
| No return codes | Use exceptions to separate error handling from the happy path |
| No nested try/catch | Extract each failure mode into its own method |
| DRY | Extract duplicated logic; every concept lives in exactly one place — unless duplication is deliberate and justified |

---

## Guiding Philosophy

Code is read far more often than it is written. Every rule in this skill is a consequence of that.

When writing or reviewing code, two questions drive every decision:

**Does this communicate intent?** Names, method size, and abstraction level exist so the reader understands what code does without tracing its execution. If a name needs a comment, rename it. If a method needs a mental map to follow, split it.

**Does this separate concerns cleanly?** High cohesion means a unit does one thing - everything inside serves that purpose and nothing else. Low coupling means a unit hides its internals - callers depend on what it does, not how it does it.

The two reinforce each other: when something is hard to name, it usually does too much. Fixing structure improves readability; fixing names clarifies structure.

Use these as your decision heuristic when the rules don't give an obvious answer.

---

## Part 1 - Naming

Every name (variable, method, class) should answer three questions: why it exists, what it does, and how it is used. If a name requires a comment, it does not reveal its intent.

```java
// Implicit: what does p[0] mean? what is 0.43?
public boolean elig(int[] p, double amt) {
    return p[0] >= 700 && p[1] >= 24 && amt <= p[2] * 0.43;
}

// Explicit: reads like a business requirement
public boolean isEligibleForLoan(CreditProfile profile, double requestedAmount) {
    return profile.getCreditScore() >= MIN_CREDIT_SCORE
        && profile.getEmploymentMonths() >= MIN_EMPLOYMENT_MONTHS
        && requestedAmount <= profile.getMonthlyIncome() * MAX_DEBT_TO_INCOME_RATIO;
}
```

Core rules:
- **Classes:** noun or noun phrase - `OrderService`, `InvoiceCalculator`, `CustomerDTO`; not `DataProcessor` or `OrderHandler`
- **Methods:** verb or verb phrase - `isEligibleForDiscount`, not `check`
- **Avoid encodings:** no `strName`, `bIsActive`, `m_field`, `IFoo` - the type system handles types
- **Avoid abbreviations:** `calculateTotal`, not `calcTtl`
- **Avoid noise words:** `Customer`, not `CustomerData` or `CustomerManager`
- **Avoid magic numbers:** extract to named `static final` constants that explain significance, not just the value

```java
// Avoid
if (account.getBalance() < 100.0) notifyLowBalance(account);

// Prefer: the name explains what the threshold means
private static final double LOW_BALANCE_THRESHOLD = 100.0;
if (account.getBalance() < LOW_BALANCE_THRESHOLD) notifyLowBalance(account);
```

> For deeper examples - boolean naming patterns, naming by layer, collection naming, diagnosing a bad name - read `references/naming.md`.

---

## Part 2 - Functions

The golden rule: **a method should have a single, clear purpose.** If you can describe what it does using the word "and", it is doing too much.

**Keep methods cohesive and focused.** Extract chunks into well-named helpers - but measure by cohesion, not line count. A focused 40-line method beats four artificial 10-line splits.

**Stepdown Rule.** Each method reads at one level of abstraction and delegates the *how* to helpers below it.

```java
// Mixed levels: validation, arithmetic, and notification all in one method
public void processOrder(Order order) {
    Objects.requireNonNull(order);
    double total = 0;
    for (var item : order.getItems()) total += item.getPrice() * item.getQuantity();
    if (total > 1000) total *= 0.9;
    emailService.send(order.getCustomer().getEmail(), "Your total: " + total);
}

// Stepdown: top level reads as a narrative; each helper owns one concern
public void processOrder(final Order order) {
    Objects.requireNonNull(order, "order must not be null");
    var total = calculateTotal(order);
    notifyCustomer(order.getCustomer(), total);
}
```

**Arguments:** 0–2 ideal; 3 acceptable; 4+ → introduce a parameter `record`.

**Avoid flag arguments** - a boolean that changes behavior means the method does two things; split it into two methods.

**Avoid output arguments** - don't mutate a parameter as a return channel; return a value instead.

**Command Query Separation** - a method either changes state (command, returns `void`) or answers a question (query, returns a value). Never both.

**Wrap complex conditions** in intention-revealing methods so the `if` reads like a sentence.

> For deeper examples - the stepdown rule across three levels, CQS in real services, argument objects with validation, hidden side effects - read `references/functions.md`.

---

## Part 3 - Comments

Prefer code that explains itself. A comment that describes what the code does is a failure of naming; a comment that explains why is often valuable.

**When comments are useful:**
- Legal headers
- Explaining non-obvious business rules or constraints ("penalty is capped by regulation X")
- Warning about known side effects or performance implications
- TODO notes (keep them short-lived)

**When to remove comments:**
- Comments that restate the code: delete them and improve the name instead
- Commented-out code: delete it; version control preserves history
- Misleading or outdated comments: they are worse than no comment

```java
// Avoid: restates the code
// Increment counter
counter++;

// Avoid: commented-out code
// order.validate();
// order.submit();

// Useful: explains the why, not the what
// Regulatory cap: penalty cannot exceed 10% of the original amount (EU Directive 2019/882)
double penalty = Math.min(calculatedPenalty, amount * MAX_PENALTY_RATE);
```

---

## Part 4 - Classes

**Single Responsibility Principle (SRP):** a class should have one reason to change. When a class handles business logic and persistence and formatting, it will be touched for three different reasons, which is a cohesion problem.

```java
// Too many responsibilities
class OrderService {
    void calculateTotal(Order order) { }
    void saveToDatabase(Order order) { }
    void sendConfirmationEmail(Order order) { }
    String formatForInvoice(Order order) { }
}

// Split by responsibility
class OrderCalculator { void calculateTotal(Order order) { } }
class OrderRepository  { void save(Order order) { } }
class OrderNotifier    { void sendConfirmation(Order order) { } }
class OrderFormatter   { String formatForInvoice(Order order) { } }
```

> **Caveat - split when there is a clear reason, not by default.** SRP is a signal to watch for, not a mechanical rule to apply everywhere. Splitting a class that genuinely does two things reduces coupling and makes each part easier to test and change. Splitting a class that just *looks* big adds indirection, spreads related logic across files, and forces the reader to jump around to understand something simple. Before extracting, ask: does this class change for more than one reason in practice? If the answer is no, leave it together. A single cohesive class is better than three artificial ones.
>
> ```java
> // This class has several methods - but they all serve one purpose: calculating an invoice.
> // Every field and method is used together. There is one reason to change it: pricing rules.
> // Splitting it into InvoiceTaxCalculator, InvoiceDiscountCalculator, etc. would only add
> // indirection without any real benefit.
> class InvoiceCalculator {
>     double calculateSubtotal(List<LineItem> items) { ... }
>     double applyDiscount(double subtotal, Customer customer) { ... }
>     double applyTax(double subtotal, TaxRegion region) { ... }
>     double calculateTotal(List<LineItem> items, Customer customer, TaxRegion region) { ... }
> }
> ```

**Keep classes small and cohesive.** If most methods use most fields, cohesion is high. When a subset of methods use a subset of fields, the class is likely two classes in disguise.

**Objects vs Data Structures.** These are opposites, and mixing them creates the worst of both worlds.

- **Objects** hide their data behind private fields and expose *behavior*. Callers tell them what to do; they decide how. `order.cancel()` - the caller doesn't touch status directly.
- **Data structures** expose their data and have no meaningful behavior. `record OrderSummary(UUID id, String status, BigDecimal total) {}` - pure data, no logic.

The danger zone is the **anemic model**: a class that looks like an object (it has a name, it's instantiated) but behaves like a data structure (all fields exposed via getters/setters, all logic living in external services). It gives you the verbosity of OOP with none of the encapsulation benefits.

```java
// Anemic: Order is a data bag - any caller can put it in an invalid state
public class Order {
    private OrderStatus status;
    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus status) { this.status = status; } // no guard
}

// In some service far away:
order.setStatus(OrderStatus.CANCELLED); // works even if the order was already delivered

// Rich object: Order enforces its own rules
public class Order {
    private OrderStatus status;

    public void cancel() {
        if (this.status == OrderStatus.DELIVERED) {
            throw new IllegalStateException("Cannot cancel a delivered order");
        }
        this.status = OrderStatus.CANCELLED;
    }
}
```

Use data structures (`record`, DTOs) freely for carrying data across boundaries. Use objects for anything that has invariants to protect.

**Model closed hierarchies with sealed types.** When a concept has a fixed set of variants, use a `sealed` interface with `record` implementations. This makes all valid states visible at a glance and lets the compiler enforce exhaustive handling - no silent `default` branches that hide forgotten cases.

```java
// Modern sealed hierarchy: variants are explicit, compiler rejects incomplete switches
sealed interface Notification permits EmailNotification, SmsNotification {}
record EmailNotification(String address) implements Notification {}
record SmsNotification(String phone)    implements Notification {}

// Exhaustive - the compiler rejects this if a permitted variant is missing
String destination = switch (notification) {
    case EmailNotification e -> e.address();
    case SmsNotification s   -> s.phone();
};
```

**Encapsulate what varies.** Fields should be private. Expose behavior through methods, not raw data.

**Law of Demeter: talk to friends, not strangers.** A method should only call methods on objects it directly owns: itself, its fields, its parameters, or objects it creates. Reaching through an object to call methods on its internals creates tight coupling to internal structure.

```java
// Train wreck: the caller knows the internal structure of Order, Customer, and Account
double fee = order.getCustomer().getAccount().getPlan().getMonthlyFee();

// Better: ask Order directly; let it figure out the path
double fee = order.getMonthlyFee();
```

---

## Part 5 - Error Handling

**Don't use exceptions for flow control** - exceptions are for unexpected conditions, not expected decisions.

**Use specific exception types** - a caller catching `Exception` cannot distinguish a validation failure from a system error.

**Use exceptions over return codes** - return codes clutter the happy path and are easy to silently ignore.

**Avoid nested try/catch** - nesting means the method does too much. Extract each block so each method handles one failure mode at one level of abstraction.

**Don't return null** - return `Optional<T>` for optional results, or an empty collection for collections.

```java
// Avoid: every caller must remember to null-check
public Order findOrder(Long id) {
    if (!exists(id)) return null;
}

// Prefer: the absence is explicit and the caller is forced to handle it
public Optional<Order> findOrder(Long id) {
    if (!exists(id)) return Optional.empty();
}
```

**Don't pass null** as an argument - use overloads or the Null Object pattern instead.

> For a full nested try/catch refactoring, custom exception hierarchy design, Optional chaining for complex domain logic, and the Null Object pattern - read `references/error-handling.md`.

---

## Part 6 - Duplication (DRY)

Every piece of knowledge should have a single, authoritative representation. Duplicate code means duplicate bugs; a fix in one copy is silently not applied to the others.

When you spot duplicated logic, extract it:
- Into a private method if it's within a class
- Into a shared utility/service if it's across classes
- Into a base class or interface default method if it's structural

---

## Part 7 - Formatting

Formatting is about communication. A consistently formatted file lets a reader scan structure without parsing every line.

**The Newspaper Metaphor.** A file should read like a news article: the headline (class name) at the top, the high-level summary (public methods) near the top, and the fine-grained details (private helpers) toward the bottom. A reader should be able to get the gist from the top without reading everything.

```java
// Good vertical order: public contract first, implementation details below
public class InvoiceService {

    public Invoice generate(Order order) { ... }        // ← reader sees the what
    public void send(Invoice invoice) { ... }

    private Money calculateSubtotal(List<LineItem> items) { ... }  // ← details below
    private Money applyTax(Money subtotal, TaxRegion region) { ... }
}
```

**Vertical distance.** Related concepts belong close together; unrelated concepts belong apart. If you have to scroll to understand how two things relate, they are too far apart.

- Declare variables close to their first use, not at the top of the method
- Keep a caller and its called method in the same file, close together
- Group fields that are always used together; separate fields that serve different purposes

**One level of change per line.** Each line should do one thing. Chained method calls that each do something significant belong on separate lines so each step can be read and debugged independently.

```java
// Avoid: multiple concerns on one line - hard to read, impossible to set a breakpoint on one step
return orderRepository.findById(id).map(o -> { o.cancel(); return orderRepository.save(o); }).orElseThrow();

// Prefer: one step per line - each line has a clear purpose
var order = orderRepository.findById(id)
    .orElseThrow(() -> new OrderNotFoundException(id));
order.cancel();
return orderRepository.save(order);
```

**Horizontal length.** Keep lines under ~120 characters. Longer lines force horizontal scrolling and hide structure. When a method call has many arguments or a condition has many clauses, break onto multiple lines and align for readability.

**Consistent indentation and braces.** Pick a style and apply it everywhere. Inconsistency forces readers to mentally re-parse the same constructs each time they appear. In Java projects, the team convention (enforced by a formatter like Checkstyle or Spotless) matters more than which specific style is chosen.

---
