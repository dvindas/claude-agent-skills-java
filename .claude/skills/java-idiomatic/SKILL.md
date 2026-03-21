---
name: java-idiomatic
description: >
  Write idiomatic modern Java using the right language feature for each problem.
  Use this skill whenever writing new Java code, reviewing or modernizing existing
  code, or choosing between old and modern constructs. Trigger on mentions of:
  records, streams, Optional, var, text blocks, sealed classes, pattern matching,
  virtual threads, null safety, immutability, resource handling, HttpClient, or
  any collection/data-handling code. Also use when the user has boilerplate-heavy
  Java that could be simplified.
license: MIT
metadata:
  author: dvindas
  version: '1.0.0'
---

# Java Idiomatic

Apply these patterns by default when writing or reviewing Java code.

## Default Rules

Apply these without being asked:

- Reject null at public method boundaries with `Objects.requireNonNull` — makes the contract explicit and surfaces bugs at the entry point rather than deep inside the call stack.
- Return `Optional<T>` instead of `null` from any method that may produce no result — forces callers to handle the absent case and eliminates silent NullPointerExceptions.
- Return empty collections instead of `null` — callers can iterate or stream safely without a null check.
- Use `final` on local variables and parameters that are never reassigned — communicates intent and prevents accidental mutation.
- Use `var` when the type is obvious from the right-hand side — reduces noise without sacrificing clarity.
- Declare variables, parameters, and return types using the interface type, not the concrete class — decouples the contract from the implementation so the concrete type can be swapped without touching callers.
- Prefer `record` over a plain class for any simple data holder — eliminates boilerplate (getters, equals, hashCode, toString) and makes immutability the default.
- Use `List.of`, `Map.of`, `Set.of` for fixed collections; never `Arrays.asList` — factory methods return truly immutable collections and reject null entries eagerly.
- Use modern API replacements: `java.time` over `Date`/`Calendar`, `ArrayDeque` over `Stack`, `HashMap` over `Hashtable` — legacy types are poorly designed, often synchronized when you don't need it, or carry broken semantics.
- Close all resources with try-with-resources — guarantees cleanup even when exceptions are thrown, without requiring a finally block.

---

## Feature Selection Guide

| When you need to...                                              | Use                    | Since |
| ---------------------------------------------------------------- | ---------------------- |-------|
| Prevent a variable or param from being reassigned                | `final`                | N/A   |
| Reject null inputs immediately                                   | `Objects.requireNonNull` | N/A |
| Fall back to a default when a value may be null                  | `Objects.requireNonNullElse` | 9 |
| Compare two values where either may be null                      | `Objects.equals`       | N/A   |
| Hash multiple fields without null checks                         | `Objects.hash`         | N/A   |
| Strip null entries from a stream                                 | `.filter(Objects::nonNull)` | N/A |
| Expose a collection without allowing mutation                    | Unmodifiable collections | N/A |
| Close resources safely                                           | try-with-resources     | N/A   |
| Model a fixed set of named values                                | `enum`                 | N/A   |
| Avoid scattering repeated literals across the codebase           | Named `static final` constant | N/A |
| Communicate the exact failure reason when throwing               | Specific exceptions    | N/A   |
| Build a string piece by piece in a single thread                 | `StringBuilder`        | N/A   |
| Build a string piece by piece across multiple threads            | `StringBuffer`         | N/A   |
| Declare variables, params, and return types by contract          | Interface type (`List`, `Map`, `Set`, ...) | N/A |
| Return no result without using null                              | `Optional<T>`          | N/A   |
| Return a collection that may be empty                            | Empty collection (`List.of()`, `Collections.emptyList()`) | 9 |
| Filter, transform, sort, or aggregate elements in a collection   | Stream API             | N/A   |
| Represent a date, time, duration, or timezone                    | `java.time` (`LocalDate`, `ZonedDateTime`, `Instant`, `Duration`) | N/A |
| Replace a legacy `Stack`                                         | `ArrayDeque`           | N/A   |
| Replace a legacy `Hashtable`                                     | `HashMap` / `ConcurrentHashMap` | N/A |
| Create a small immutable collection inline                       | `List.of`, `Map.of`, `Set.of` | 9  |
| Declare a local variable whose type is obvious from context      | `var`                  | 10    |
| Send an HTTP request without third-party dependencies            | `HttpClient`           | 11    |
| Group related fields into a simple, immutable data class         | Record                 | 16    |
| Embed a multiline string like SQL, JSON, or HTML cleanly in code | Text block             | 15    |
| Restrict a type to a known, closed set of subtypes               | Sealed class/interface | 17    |
| Match and destructure types with switch expressions or instanceof | Pattern matching       | 21    |
| Avoid tying up OS threads while waiting on blocking I/O tasks    | Virtual threads        | 21    |

---

## Part 1 - Core Practices

### Use `final` to signal immutability

Apply `final` to local variables and parameters whenever the value should not be reassigned. It communicates intent clearly and prevents accidental mutation.

```java
public void process(final Order order) {
    final var total = order.calculateTotal();
    // neither order nor total can be reassigned
}
```

### Fail fast on bad input

Validate inputs at the start of every method. Catching problems at the entry point makes bugs easier to trace.

```java
public Invoice createInvoice(Customer customer, List<Item> items) {
    Objects.requireNonNull(customer, "customer must not be null");
    Objects.requireNonNull(items, "items must not be null");
    // ...
}
```

### Use `Objects` utilities for null safety

```java
// Default fallback
String name = Objects.requireNonNullElse(input, "Anonymous");

// Null-safe comparison (avoids NullPointerException when either side may be null)
if (Objects.equals(a, b)) { ... }

// Safe hashCode over multiple fields
@Override
public int hashCode() {
    return Objects.hash(name, email);
}

// Strip nulls from a stream
list.stream()
    .filter(Objects::nonNull)
    .toList();
```

### Use try-with-resources for closeable resources

Resources like connections, streams, and readers must be closed even when exceptions occur. try-with-resources guarantees this without requiring a `finally` block, keeping the code compact and safe.

```java
try (var connection = dataSource.getConnection();
     var stmt = connection.prepareStatement(sql)) {
    // connection and stmt are closed automatically
}
```

### Throw specific exceptions

Never throw raw `Exception` or `RuntimeException`. Use the most specific type that fits the situation.

```java
throw new IllegalArgumentException("Price must be positive, got: " + price);
throw new IllegalStateException("Order is already closed");
```

### Use enums instead of int or String constants

```java
// Instead of: public static final int STATUS_ACTIVE = 1;
public enum Status { ACTIVE, INACTIVE, PENDING }
```

### Prefer named constants over hardcoded literals

Extract repeated or meaningful literals into named constants. Makes intent clear and changes happen in one place.

```java
// Avoid
if (user.getRole().equals("ADMIN")) { ... }

// Prefer
private static final String ROLE_ADMIN = "ADMIN";
if (user.getRole().equals(ROLE_ADMIN)) { ... }
```

- If the value belongs to a fixed set, prefer an `enum` over a `String` constant
- Group related constants in a dedicated class rather than scattering them

---

## Part 2 - Efficient Data Handling

### Declare types by interface, not by concrete class

Always declare variables, parameters, and return types using the interface. This keeps code flexible and easy to swap implementations without touching callers.

```java
// Declare as the interface
List<String> names = new ArrayList<>();
Map<String, User> userMap = new HashMap<>();

// Return the interface from methods
public List<User> findAll() { return new ArrayList<>(); }

// Accept the interface in parameters
public void process(List<String> items) { }
```

- Use `List` instead of `ArrayList`, `Map` instead of `HashMap`, `Set` instead of `HashSet`
- Switching implementations only requires changing the instantiation line, not every caller

### Stream API (Since Java 8)

Use for any collection transformation. Prefer over imperative loops, unless the loop is more efficient by nature (e.g. index-based access).

```java
List<String> result = users.stream()
        .filter(User::isActive)
        .map(User::getName)
        .map(String::toUpperCase)
        .sorted()
        .toList();

// Grouping
Map<Department, List<Employee>> byDept = employees.stream()
        .collect(Collectors.groupingBy(Employee::department));

// Dual aggregation
var stats = employees.stream().collect(Collectors.teeing(
        Collectors.averagingDouble(Employee::salary),
        Collectors.counting(),
        SalaryStats::new
));
```

- Prefer method references over lambdas when they express the same intent
- Use `flatMap` to flatten nested collections
- Streams are single-use; never store and reuse them
- Use parallel streams only when processing large collections with expensive operations

### Optional (Since Java 8)

Return `Optional<T>` from any method that may produce no result.

```java
public Optional<User> findUser(Long id) {
    return userRepository.findById(id);
}

// Chain without null checks
findUser(42L)
    .map(User::getEmail)
    .ifPresent(emailService::send);

// Throw with context
User user = findUser(42L)
        .orElseThrow(() -> new UserNotFoundException(42L));

// Deep chain
String city = Optional.ofNullable(order)
        .map(Order::getCustomer)
        .map(Customer::getAddress)
        .map(Address::getCity)
        .orElse("Unknown");
```

- Prefer `orElse`, `orElseThrow`, or `map` over `get()`; calling `get()` without `isPresent()` will throw
- Do not use `Optional` as a field type or method parameter; it is designed only as a return type
- Return empty collections instead of `Optional<List<T>>`

### Prefer modern APIs over legacy equivalents

See the Feature Selection Guide above for the full legacy→modern mapping. The most common substitutions in practice:

**Date and time - `java.time` (Since Java 8)**

```java
// Avoid
Date now = new Date();
Calendar cal = Calendar.getInstance();

// Prefer
LocalDate today = LocalDate.now();
LocalDate nextWeek = today.plusDays(7);
ZonedDateTime appointment = ZonedDateTime.now(ZoneId.of("America/New_York"));
Instant timestamp = Instant.now(); // for machine timestamps / audit fields
Duration timeout = Duration.ofSeconds(30);
```

**Collections - factory methods (Since Java 9)**

```java
// Avoid
List<String> roles = Arrays.asList("ADMIN", "USER"); // mutable, fixed-size
Stack<String> stack = new Stack<>();                  // legacy, synchronized
Hashtable<String, User> table = new Hashtable<>();    // legacy, synchronized

// Prefer
List<String> roles = List.of("ADMIN", "USER");        // immutable
Map<String, Integer> codes = Map.of("OK", 200, "NOT_FOUND", 404);
Set<String> allowed = Set.of("GET", "POST", "PUT");
Deque<String> stack = new ArrayDeque<>();             // faster, not synchronized
```

- Use `List.copyOf`, `Map.copyOf`, `Set.copyOf` for an immutable snapshot of an existing collection
- Use `Map.ofEntries(Map.entry(...), ...)` when you need more than 10 entries

### String Concatenation

Strings are immutable — `+` inside a loop creates a new object on every iteration. Use `StringBuilder` instead; if multiple threads share the builder, use `StringBuffer` (same API, all methods synchronized).

```java
StringBuilder sb = new StringBuilder();
for (String part : parts) {
    sb.append(part).append(", ");
}
String result = sb.toString();
```

---

## Part 3 - Modern Syntax

### Local Variable Inference (`var`) (Since Java 11)

Use when the type is obvious from the right-hand side.

```java
var activityMap = new HashMap<String, List<UserActivity>>();

for (var entry : activityMap.entrySet()) {
    process(entry.getKey(), entry.getValue());
}

try (var connection = dataSource.getConnection();
     var stmt = connection.prepareStatement(sql)) {
    var rs = stmt.executeQuery();
}
```

- Skip `var` when the explicit type aids readability: `UserAccountSummary summary = buildSummary(data)`
- Do not use when the type is not obvious from context: `var result = process();`
- `var` is local only; not valid in method signatures or fields

### Text Blocks (Since Java 17)

Use for any multiline string: SQL, JSON, HTML, XML.

**JSON** - use `.formatted()` for simple structures; prefer Jackson for dynamic serialization.

```java
String body = """
        {
            "name": "%s",
            "email": "%s",
            "active": true
        }
        """.formatted(user.getName(), user.getEmail());
```

**SQL** - always pass user values as bound parameters, never embedded in the string.

```java
String sql = """
        SELECT u.id, u.name, u.email
        FROM users u
        WHERE u.active = true
          AND u.department_id = ?
        ORDER BY u.name
        """;

try (PreparedStatement stmt = connection.prepareStatement(sql)) {
    stmt.setLong(1, departmentId);
    ResultSet rs = stmt.executeQuery();
}
```

- Indentation is relative to the closing `"""`; align it to control whitespace
- Never use `.formatted()` or `+` to insert user input into SQL - this opens SQL injection

---

## Part 4 - Type Modeling & Pattern Matching

### Records (Since Java 17)

Default choice for any data structure, DTO, or value object.

```java
public record Point(int x, int y) {}

// Compact constructor with validation
public record Email(String value) {
    private static final Pattern PATTERN = Pattern.compile("^[\\w.+\\-]+@[\\w.\\-]+\\.[a-z]{2,}$");

    public Email {
        if (value == null || !PATTERN.matcher(value).matches())
            throw new IllegalArgumentException("Invalid email: " + value);
    }
}
```

- Use for DTOs, value objects, and data that does not need to change after creation
- Records can implement interfaces but cannot extend classes
- If you need mutable state or complex behavior, use a regular class instead

### Sealed Classes (Since Java 17)

Use when a type has a known, fixed set of subtypes.

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}

public record Circle(double radius)                  implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public record Triangle(double base, double height)   implements Shape {}

// Exhaustive switch - no default required
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t  -> 0.5 * t.base() * t.height();
};
```

- Prefer `sealed interface` over `sealed class`
- Permitted subtypes must be in the same package or module

### Pattern Matching (Since Java 17-21)

Prefer switch expressions over switch statements — they return a value, enforce exhaustiveness at compile time, and eliminate fall-through bugs. This makes the code both safer and more concise.

**instanceof check (Java 17)** - for a single type check inline.

```java
if (obj instanceof Circle c && c.radius() > 0) {
    draw(c);
}
```

**Switch on type (Java 21)** - pairs with sealed interfaces for exhaustive, default-free matching.

```java
String result = switch (shape) {
    case Circle c    -> "radius " + c.radius();
    case Rectangle r -> r.width() + "x" + r.height();
};
```

**Record destructuring (Java 21)** - extract record components directly in the case.

```java
String description = switch (obj) {
    case Point(int x, int y)            -> "point at " + x + ", " + y;
    case Circle(Point center, double r) -> "circle at " + center + " r=" + r;
    default                             -> "unknown shape";
};
```

- Use `when` for guarded patterns: `case Circle c when c.radius() > 0`

---

## Part 5 - Efficient I/O & Threading

### Modern HTTP Client (Since Java 11)

`HttpClient` is the standard since Java 11. Prefer it over `HttpURLConnection` — it supports async requests, connection pooling, HTTP/2, and has a clean builder API. `HttpURLConnection` requires manual stream handling and lacks async support.

```java
// Reuse a single instance (manages connection pooling)
HttpClient client = HttpClient.newHttpClient();

HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/users"))
        .header("Accept", "application/json")
        .timeout(Duration.ofSeconds(5))
        .GET()
        .build();

// Synchronous
HttpResponse<String> response = client.send(request, BodyHandlers.ofString());

// Async (pairs well with virtual threads)
client.sendAsync(request, BodyHandlers.ofString())
        .thenApply(HttpResponse::body)
        .thenAccept(this::process)
        .exceptionally(ex -> { log.error("Failed", ex); return null; });
```

- Always set `.timeout()`; make the value configurable, never hardcoded
- Use `sendAsync()` for non-blocking I/O

### Virtual Threads (Since Java 21)

Use for any I/O-bound work (HTTP, DB, file). Do not pool them.

**One thread per task**

```java
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
executor.submit(() -> process(fetchData()));

// Or directly
Thread.ofVirtual().start(() -> process(fetchData()));
```

**Fan-out: multiple concurrent requests**

```java
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

CompletableFuture.allOf(
        CompletableFuture.supplyAsync(() -> fetchUser(1), executor),
        CompletableFuture.supplyAsync(() -> fetchUser(2), executor),
        CompletableFuture.supplyAsync(() -> fetchUser(3), executor)
).join();
```

- Do not use for CPU-heavy tasks; for those, use classic platform threads

---

## Common Mistakes

Read `references/common-mistakes.md` when reviewing or modernizing existing Java code — it contains a full lookup table of anti-patterns and their modern replacements.
