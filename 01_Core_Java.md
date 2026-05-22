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
- **synchronized**: Simple mutual exclusion + visibility. Good default for small critical sections.
- **ReentrantLock**: Use when you need `tryLock`, timeout, fairness, interruptible lock acquisition, or more control.
- **ReadWriteLock**: Useful for read-heavy workloads where reads can run concurrently but writes need exclusivity.
- **StampedLock**: Optimistic reads for read-mostly paths; always validate the stamp or fall back to a real read lock.
- **ThreadLocal**: Thread-confined state; useful for request context/correlation IDs, but dangerous with pools if not cleared.

### ReentrantLock — timed lock example
```java
private final Lock lock = new ReentrantLock();

public boolean process() throws InterruptedException {
  if (!lock.tryLock(200, TimeUnit.MILLISECONDS)) {
    return false; // fail fast instead of waiting forever
  }
  try {
    // critical section
    return true;
  } finally {
    lock.unlock();
  }
}
```

### StampedLock — optimistic read example
```java
private final StampedLock lock = new StampedLock();
private int x;

public int readX() {
  long stamp = lock.tryOptimisticRead();
  int value = x;

  if (!lock.validate(stamp)) {
    stamp = lock.readLock();
    try {
      value = x;
    } finally {
      lock.unlockRead(stamp);
    }
  }
  return value;
}
```

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
---

## `equals()` and `hashCode()` — Interview Implementation Notes

This is a common senior Java interview topic because it tests object identity, collection behavior, and memory/reference understanding. If someone says “implement `equals` and `hashCode`”, avoid random IDs, timestamps, or `System.identityHashCode()`. The implementation must be **stable** and based on the fields that define logical equality.

### Mental model
- `==` compares object references.
- `equals()` compares logical equality.
- `hashCode()` is used by hash-based collections like `HashMap`, `HashSet`, and `ConcurrentHashMap`.
- If two objects are equal according to `equals()`, they **must** return the same `hashCode()`.
- If two objects have the same hash code, they are **not necessarily equal**; collisions are allowed.
- Never generate a random hash code. Hash code must remain stable while the object is used as a key.

### Correct implementation example
```java
public final class Employee {
  private final Long id;
  private final String email;

  public Employee(Long id, String email) {
    this.id = id;
    this.email = email;
  }

  public Long getId() {
    return id;
  }

  public String getEmail() {
    return email;
  }
 
  @Override
  public boolean equals(Object o) {
    if (this == o) { // Reference Equality
      return true;
    }

    if (o == null || getClass() != o.getClass()) { //Null or type Check. Accepts subsclasses
      return false;
    }

    Person other = (Person) o;

    return Objects.equals(id, other.id);
  }
  
  @Override
  public int hashCode() {
    return Objects.hash(id, email);
  }
}
 
//HashCode Implementation
@Override
public int hashCode() {
  int result = 1;

  result = 31 * result + Objects.hashCode(name);
  result = 31 * result + Objects.hashCode(surname);
  result = 31 * result + Objects.hashCode(phone);

  return result;
}


```

### Interview explanation
“I first check reference equality because the same object is always equal to itself. Then I check type compatibility with `instanceof`. After that I compare the fields that define business equality using `Objects.equals` because it is null-safe. For `hashCode`, I use the same fields as `equals`. This keeps the equals/hashCode contract valid, especially when the object is used in `HashMap` or `HashSet`. I would not use random numbers because the hash code must be stable.”

### Common pitfalls
- Overriding `equals()` but not `hashCode()`.
- Using mutable fields in `hashCode()` while the object is stored in a `HashSet` or used as a `HashMap` key.
- Using database IDs before they are assigned, especially with JPA entities.
- Using `getClass()` vs `instanceof` without understanding inheritance/proxy implications. With Hibernate proxies, `getClass()` can be problematic.
- Thinking hash code must be unique. It does not; it only helps distribute objects into buckets.

### JPA entity note
For JPA entities, be careful with generated IDs. Before persistence, the ID may be `null`. A safer approach is often to use a stable natural key when one exists, or implement entity equality carefully based on your persistence lifecycle. Do not include lazy collections in `equals()`/`hashCode()`.

---

## HashMap, HashSet and `compute*` Methods

### What happens inside `HashMap`?
- `HashMap` calculates the key hash, chooses a bucket, then compares keys with `equals()`.
- Good `hashCode()` distribution improves performance.
- Bad hash distribution causes collisions. In modern Java, heavily-collided buckets can become tree bins, but you should still design keys properly.
- `HashMap` is not thread-safe. Use `ConcurrentHashMap` for concurrent access.

### `computeIfAbsent` — group values by key
```java
Map<String, List<String>> employeesByDepartment = new HashMap<>();

employeesByDepartment
        .computeIfAbsent("engineering", key -> new ArrayList<>())
        .add("Burak");

employeesByDepartment
        .computeIfAbsent("engineering", key -> new ArrayList<>())
        .add("Alice");
```

Without `computeIfAbsent`, you usually write verbose `containsKey/get/put` logic. In interviews, this is a clean way to show practical Java knowledge.

### `compute` — update a counter
```java
Map<String, Integer> frequency = new HashMap<>();

for (String word : List.of("java", "spring", "java")) {
        frequency.compute(word, (key, oldValue) -> oldValue == null ? 1 : oldValue + 1);
        }
```

### `merge` — simpler frequency counter
```java
Map<String, Integer> frequency = new HashMap<>();

for (String word : List.of("java", "spring", "java")) {
        frequency.merge(word, 1, Integer::sum);
}
```

### `ConcurrentHashMap` note
```java
ConcurrentHashMap<String, LongAdder> counters = new ConcurrentHashMap<>();

counters
        .computeIfAbsent("orders.created", key -> new LongAdder())
        .increment();
```

This avoids coarse locking and is a common production-style pattern for high-update counters.

---

## Immutability

Immutable objects improve safety, simplify reasoning, and are naturally thread-safe after proper construction. This topic often appears in senior interviews because it connects Java memory model, defensive copying, collections, DTO design, and object references.

### Rules for an immutable class
- Make the class `final` so it cannot be subclassed and mutated through inheritance.
- Make fields `private final`.
- Do not provide setters.
- Validate constructor inputs.
- Do not expose mutable internal objects directly.
- Use defensive copies for mutable constructor arguments.
- Return defensive copies or unmodifiable views from getters.
- Do not let `this` escape from the constructor.

### Basic immutable class
```java
public final class User {
  private final String name;
  private final LocalDate createdAt; // LocalDate is already immutable

  public User(String name, LocalDate createdAt) {
    this.name = Objects.requireNonNull(name);
    this.createdAt = Objects.requireNonNull(createdAt);
  }

  public String getName() {
    return name;
  }

  public LocalDate getCreatedAt() {
    return createdAt;
  }
}
```

### Defensive copy with collections
```java
public final class Team {
    private final String name;
    private final List<String> members;

    public Team(String name, List<String> members) {
        this.name = Objects.requireNonNull(name);
        this.members = List.copyOf(members); // defensive copy + unmodifiable
    }

    public String getName() {
        return name;
    }

    public List<String> getMembers() {
        return members; // safe because List.copyOf created an unmodifiable list
    }
}
```

For immutable element types like `String`, `Integer`, `LocalDate`, returning the unmodifiable copied list is enough. You do not need to copy each element because the elements themselves cannot be mutated.

### Deep copy with mutable nested objects
If the list contains mutable objects, copying only the list is not enough. The caller can still mutate the object reference inside the list.

```java
public final class Grade {
  private final String course;
  private final int score;

  public Grade(String course, int score) {
    this.course = Objects.requireNonNull(course);
    this.score = score;
  }

  public Grade(Grade other) {
    this(other.course, other.score);
  }

  public String getCourse() { return course; }
  public int getScore() { return score; }
}

public final class Student {
  private final String name;
  private final List<Grade> grades;

  public Student(String name, List<Grade> grades) {
    this.name = Objects.requireNonNull(name);
    this.grades = grades.stream()
            .map(Grade::new)      // deep copy each element
            .toList();            // unmodifiable in modern Java
  }

  public String getName() {
    return name;
  }

  public List<Grade> getGrades() {
    return grades.stream()
            .map(Grade::new)      // protect internal state
            .toList();
  }
}
```

### Mutable object example: `Date`
```java
public final class Token {
    private final Date expiresAt;

    public Token(Date expiresAt) {
        this.expiresAt = new Date(expiresAt.getTime()); // copy in constructor
    }

    public Date getExpiresAt() {
        return new Date(expiresAt.getTime()); // copy in getter
    }
}
```

Prefer modern immutable date/time classes like `Instant`, `LocalDate`, and `ZonedDateTime` where possible.

### Records are shallowly immutable
```java
public record Product(String id, String name, BigDecimal price) {}
```

Records make fields final and remove setters, but they do **not** automatically deep-copy mutable fields.

```java
public record OrderView(String orderId, List<String> items) {
  public OrderView {
    items = List.copyOf(items);
  }
}
```

### Interview answer
“An immutable class is final, has private final fields, no setters, validates inputs, and protects mutable state with defensive copies. If the field is immutable, like `String` or `LocalDate`, returning it is fine. If the field is mutable, like `Date`, `List`, or a custom mutable object, I copy it in the constructor and also protect it in the getter. For nested mutable objects, I need deep copy, not only `List.copyOf`.”

### Common mistakes
- `final List<T>` does not mean the list contents are immutable. It only means the reference cannot point to another list.
- `Collections.unmodifiableList(original)` is only a view. If the original list changes, the view reflects it. Prefer `List.copyOf(original)`.
- Defensive copy of the list is not enough when elements are mutable.
- Do not expose arrays directly; use `array.clone()` or convert to an immutable list.

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
