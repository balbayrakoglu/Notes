# Core Java — Senior Developer Edition

> **Senior-level TL;DR — Core Java (Java 17+)**
> - Prefer **immutability** (records for DTOs, final fields) and avoid shared mutable state.
> - Understand **JMM happens-before**; `volatile` = visibility (not atomicity). Use `Atomic*`/locks for atomic ops.
> - Choose the right **lock**: `synchronized` (simple), `ReentrantLock` (tryLock/timed), `ReadWriteLock` (read-heavy), `StampedLock` (optimistic reads).
> - Use **virtual threads** (JDK 21) for blocking I/O concurrency; audit `ThreadLocal` usage before migrating.
> - Compose async with **CompletableFuture**; always handle exceptions (`exceptionally`, `handle`), prefer custom executors.
> - Keep **exceptions** meaningful: catch to recover/translate; never swallow; log context (no PII).
> - Favor **streams** for declarative ops; avoid side effects; `parallelStream()` only for CPU-bound work with enough data.
> - Use **Optional** only as return type; not in fields/params; don’t nest.
> - Watch **autoboxing** in hot paths; prefer primitives; benchmark with **JMH**.
> - Know your **GCs** (G1 default; ZGC/Shenandoah for low-pause); size heap sanely; observe with JFR/VisualVM.
> - Use **try-with-resources**; `StringBuilder` in loops; `equals()` vs `==` for Strings.
> - Implement consistent `equals()/hashCode()`; prefer **enum** to magic constants.
> - Diagnose with **thread dumps**, **jstat**, **JFR**; treat deadlocks & contention as first-class failures.
> - Keep **code locality & clarity**: small methods, single responsibility, package-by-feature.
> - Treat **performance** as a science: measure, profile, change one variable at a time.

## Overview
This section covers the fundamentals and advanced concepts of Java 17, focusing on performance, concurrency, memory management, and clean design principles expected from a senior developer.

---

## JVM Architecture
The Java Virtual Machine (JVM) provides an abstraction between compiled Java code and the operating system. It consists of the following major components:

- **Class Loader Subsystem**: Loads `.class` files into memory and performs linking (verification, preparation, resolution).
- **Runtime Data Areas**
    - **Heap**: Stores objects and class instances (Young + Old Generation).
    - **Stack**: Holds local variables and method call frames per thread.
    - **Metaspace**: Stores class metadata (replaced PermGen).
    - **PC Register**: Holds the current instruction address per thread.
    - **Native Method Stack**: Used for JNI calls.
- **Execution Engine**: Includes the interpreter and JIT compiler for optimized native code.
- **Garbage Collector (GC)**: Automatically manages object lifecycle. Modern collectors include **G1GC**, **ZGC**, and **Shenandoah**.

```bash
java -Xms512m -Xmx1024m -XX:+UseG1GC -jar app.jar
```

---

## Java Memory Model (JMM)
Defines how threads interact through memory and ensures visibility and ordering of operations using the **happens-before** relationship.

```java
volatile boolean flag = false;

public void writer() {
    flag = true; // write
}

public void reader() {
    if (flag) { // guaranteed to see updated value
        System.out.println("Visible to reader thread");
    }
}
```

- `volatile` ensures visibility but **not atomicity**.
- Use `AtomicInteger`, `synchronized`, or `Lock` for atomic operations.
- Synchronization creates a happens-before relationship between threads.

**Also know**
- *Final field semantics*: writes to `final` fields in a constructor become visible to other threads after the constructor completes (unless `this` escapes during construction).
- Avoid *publication before construction* (don’t leak `this` from constructors or field initializers).

---

## Threading and Locks
- **synchronized**: Ensures mutual exclusion and visibility.
- **ReentrantLock**: Advanced locking (`tryLock`, fairness, interruptible, timed).
- **ReadWriteLock**: Improves concurrency for read-heavy workloads.
- **StampedLock**: Optimistic reads for read-mostly paths; validate stamp or fall back to write lock.
- **ThreadLocal**: Provides thread-confined variables (use with care to avoid leaks).
> **Tip — StampedLock optimistic reads**
```java

Lock lock = new ReentrantLock(true);

public void process() {

var sl = new java.util.concurrent.locks.StampedLock();
long stamp = sl.tryOptimisticRead();
int localX = this.x;
if (!sl.validate(stamp)) {
    stamp = sl.readLock();
    try { localX = this.x; } finally { sl.unlockRead(stamp); }
}
// use localX
```


    if (lock.tryLock()) {
        try {
            // critical section
        } finally {
            lock.unlock();
        }
    }
}


---

## Virtual Threads (Java 21+)
Virtual threads are lightweight and reduce blocking cost in high-concurrency environments. They enable large numbers of concurrent tasks without exhausting platform threads.

> **Note:** Virtual threads require **Java 21+**. If you’re on Java 17 in production, plan migration/compat testing first.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> System.out.println(Thread.currentThread()));
}
```

---

## CompletableFuture & Asynchronous Programming
Used for non-blocking asynchronous operations and composition of tasks.

```java
CompletableFuture.supplyAsync(() -> fetchData())
    .thenApply(this::transform)
    .thenAccept(System.out::println)
    .exceptionally(ex -> {
        log.error("Error occurred", ex);
        return null;
    });
```

- Use `thenCombine()` for merging tasks, `allOf()` for aggregation.
- Prefer custom executors to control thread pool size.

**Patterns**
- Use `handle((res, ex) -> ...)` when you need to deal with both result and exception in the same stage; `exceptionally` handles only failures.
- With `allOf`, collect results safely:
```java
CompletableFuture<List<Result>> all =
    CompletableFuture.allOf(f1, f2, f3)
        .thenApply(v -> List.of(f1, f2, f3).stream()
            .map(CompletableFuture::join)
            .toList());
```
- Use `handle((res, ex) -> ...)` when you need both result and error.

---

## Immutability
Immutable objects improve safety and thread-safety.

Rules:
- Fields are `private final`.
- No setters.
- Defensive copies for mutable fields.
- Class declared `final`.

```java
public final class User {
    private final String name;
    private final LocalDate createdAt;

    public User(String name, LocalDate createdAt) {
        this.name = name;
        this.createdAt = createdAt;
    }

    public String getName() { return name; }
    public LocalDate getCreatedAt() { return createdAt; }
}
```

Using Java 16+ Records:

> **Defensive copies** for mutable fields
```java
public final class Report {
  private final List<String> lines;
  public Report(List<String> lines) { this.lines = List.copyOf(lines); }
  public List<String> lines() { return List.copyOf(lines); }
}
```

```java
public record Product(String id, String name, BigDecimal price) {}
```

---

## Exception Handling

### Hierarchy
```
Throwable
 ├─ Exception (Checked)
 │   └─ IOException, SQLException
 └─ RuntimeException (Unchecked)
     └─ NullPointerException, IllegalArgumentException
```

### Best Practices
- Catch only when recovery is possible.
- Wrap low-level exceptions into domain-specific ones.
- Never swallow exceptions silently.
- Log meaningful context (no PII).

```java
try {
    process();
} catch (IOException ex) {
    throw new BusinessException("Failed to process", ex);
}
```

---

## Functional Programming & Streams
Streams provide declarative data processing.

```java
List<Integer> list = List.of(1, 2, 3, 4, 5);
int sum = list.stream()
              .filter(n -> n % 2 == 0)
              .mapToInt(Integer::intValue)
              .sum();
```

- Avoid modifying shared state in stream operations.
- Use `parallelStream()` only for CPU-bound tasks with sufficient data sizes.

Example – Word frequency map:

```java
Map<String, Long> frequency = words.stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
```

**Merging to a Map with duplicates**
```java
Map<String, Integer> totals = items.stream()
  .collect(Collectors.toMap(Item::key, Item::value, Integer::sum));
```

---

## Optional API
Used to avoid `NullPointerException` and make absence explicit.

```java
Optional.ofNullable(user)
    .map(User::getEmail)
    .filter(email -> email.endsWith("@company.com"))
    .ifPresent(System.out::println);
```

- Do not use `Optional` as a class field.
- Avoid nested `Optional<Optional<T>>`.
- Use for method return types, not parameters.

> **Gotcha — `orElse` vs `orElseGet`**: `orElse` eagerly evaluates its argument, `orElseGet` is lazy.
```java
var v = find().orElseGet(() -> computeExpensiveDefault());
```

**Anti-patterns:** `Optional` in fields/DTOs, nested optionals, or passing `Optional` as a parameter.

---

## Wrapper Classes & Autoboxing
Autoboxing converts primitives to wrapper types automatically. Excessive boxing can impact performance.

```java
Integer counter = 0;
for (int i = 0; i < 1_000_000; i++) {
    counter += 1; // creates new Integer each iteration
}
```

Prefer primitives in performance-critical sections.

> **FYI**: `Integer` caches values in range [-128,127]. Don’t rely on identity (`==`) outside that range.

---

## Common Pitfalls & Best Practices
- Always use **try-with-resources** for automatic resource management:

```java
try (var reader = Files.newBufferedReader(Path.of("data.txt"))) {
    return reader.readLine();
}
```

- Use `StringBuilder` for concatenation in loops.
- Use `equals()` instead of `==` for string comparison.
- Use `enum` instead of magic constants.
- Ensure `equals()` and `hashCode()` are consistent.
- Protect shared mutable state with synchronization.
- Benchmark critical code with **JMH**.

---

### Deadlocks — Quick Checklist
- Always acquire locks in a **consistent global order**.
- Keep critical sections **tiny**; avoid calling out to other services while holding locks.
- Prefer **timeouts** (`tryLock(timeout)`) and **fail fast**.
- Use `jstack`/JFR **thread dumps** to locate the cycle; add structured logs with thread/lock ids.

---

## Performance & Profiling Tools
- **JMH**: Micro-benchmarking.
- **VisualVM / Java Flight Recorder (JFR)**: Profiling memory and CPU usage.
- **jstat -gc <PID> 1000**: Monitor GC behavior.
- **Thread dumps**: Analyze deadlocks and contention.

```bash
# Thread dump
jcmd <PID> Thread.print > threads.txt

# GC stats every second (20 samples)
jstat -gc <PID> 1000 20

# Start a 60s JFR capture
jcmd <PID> JFR.start name=sample settings=profile duration=60s filename=sample.jfr
```

**See also:**
- [JPA Performance — batching & fetch plans](06_Data_Access_and_Performance_JPA.md)
- [Resilience patterns in microservices](03_Spring_Boot_and_Microservices.md)
- [Concurrency patterns & backpressure](07_System_Design_Essentials.md)

---

## Interview Drill — Core Java (Quick Q&A)

**Q1. What does `volatile` actually guarantee?**  
**A.** Visibility and ordering (a volatile write happens-before subsequent volatile reads). It does **not** ensure atomicity—use `Atomic*` or locks for compound operations.

**Q2. When would you use `ReadWriteLock` vs `StampedLock`?**  
**A.** `ReadWriteLock` is straightforward and strong under many-readers/few-writers. `StampedLock` adds **optimistic reads**—great for read-mostly workloads with low contention; validate the stamp or fall back to write lock.

**Q3. `CompletableFuture`: how do you handle errors cleanly?**  
**A.** Terminate chains with `exceptionally` or `handle`. Prefer a custom executor to bound threads. Example: `supplyAsync(...).thenApply(...).handle((res, ex) -> { ... });`.

**Q4. `Optional` best practices?**  
**A.** Only as return type; never fields/params. Avoid nesting. Prefer `map/flatMap/filter/orElseGet` composition instead of `isPresent()/get()`.

**Q5. How do you diagnose a suspected deadlock in prod?**  
**A.** Capture multiple thread dumps (`jcmd Thread.print`), inspect blocked threads and lock owners, confirm with JFR. Fix by enforcing lock ordering, shrinking critical sections, and using timed `tryLock`.
