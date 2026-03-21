# Common Mistakes

Consult this during code review or when modernizing existing Java code.

| Mistake                                                           | Replace with                                          |
| ----------------------------------------------------------------- | ----------------------------------------------------- |
| `ArrayList<X> list = new ArrayList<>()`                           | `List<X> list = new ArrayList<>()`                    |
| `if (x == null) return default; else use(x)`                      | `Objects.requireNonNullElse(x, default)`              |
| `a.equals(b)` when `a` may be null                                | `Objects.equals(a, b)`                                |
| Manual `null` fields compared or hashed without `Objects`         | `Objects.equals(a, b)` / `Objects.hash(f1, f2, ...)`  |
| Not closing a resource (`InputStream`, `Connection`, etc.)        | try-with-resources                                    |
| Manual `null` check before every use                              | `Optional<T>` as return type                          |
| `optional.get()` without `isPresent()`                            | `optional.orElseThrow()` or `optional.orElse(...)`    |
| `Optional<List<T>>` return type                                   | Return empty `List` directly                          |
| `Optional<T>` as a field or parameter                             | Use `@Nullable` or require the value at the boundary  |
| Passing a mutable list to a constructor and storing the reference | `List.copyOf(items)` in the constructor               |
| String `+` in a loop                                              | `StringBuilder`                                       |
| Hardcoded string/numeric literal used more than once              | Named `private static final` constant                 |
| `new Date()`, `Calendar.getInstance()`                            | `LocalDate`, `ZonedDateTime`, or `Instant`            |
| `java.util.Vector`                                                | `ArrayList` / `List.of`                               |
| `java.util.Stack`                                                 | `ArrayDeque`                                          |
| `java.util.Hashtable`                                             | `HashMap` / `ConcurrentHashMap`                       |
| `Arrays.asList(...)` for a fixed list                             | `List.of(...)`                                        |
| User input embedded in a SQL text block via `.formatted()` or `+` | `PreparedStatement` with bound parameters            |
| `instanceof` check followed by explicit cast                      | `instanceof Circle c` (pattern variable)              |
| `switch` statement with fall-through                              | `switch` expression with `->` arms                    |
| Raw `Exception` or `RuntimeException` thrown                      | Specific exception (`IllegalArgumentException`, etc.) |
| Mutable collection returned from a getter                         | `Collections.unmodifiableList(...)` or `List.copyOf(...)` |
| Plain class for a simple data holder                              | `record`                                              |
| `HttpURLConnection` for HTTP calls                                | `HttpClient`                                          |
| `new Thread(runnable).start()` for I/O work                       | Virtual thread via `Thread.ofVirtual().start(...)`    |
| Pooling virtual threads                                           | Use `newVirtualThreadPerTaskExecutor()` - no pooling needed |