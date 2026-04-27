---
description: Swift Structured Concurrency advanced performance expert. Covers Task lifecycle and cancellation, actor isolation and reentrancy, task groups vs async let, custom executors, priority propagation, clock APIs, AsyncSequence patterns, cooperative cancellation, actor contention diagnostics, and Swift Concurrency Instruments. Trigger when reviewing async/await code, actor design, task cancellation handling, AsyncSequence pipelines, or diagnosing concurrency-related hangs and races.
---

# Swift Structured Concurrency Advanced Performance Expert

## The Cooperative Thread Pool

Swift concurrency runs on a fixed-size thread pool (≈ number of CPU cores, typically 4–8 on iPhone). Tasks are **suspended** at `await` points, freeing the thread for other work. This is fundamentally different from GCD — you never block threads, you suspend tasks.

**Invariant**: cooperative threads must never be blocked. A blocked cooperative thread = one fewer thread for all other tasks.

```swift
// These BLOCK the cooperative thread — forbidden
Thread.sleep(forTimeInterval: 1)
semaphore.wait()
mutex.lock()       // if contended
DispatchQueue.someQueue.sync { }

// These SUSPEND (correct)
try await Task.sleep(for: .seconds(1))
await someActor.method()
let (data, _) = try await URLSession.shared.data(from: url)
```

---

## Task Hierarchy and Structured Concurrency

### Structured tasks — inherit context, cancel together

```swift
func processAll(items: [Item]) async throws -> [Result] {
    try await withThrowingTaskGroup(of: Result.self) { group in
        for item in items {
            group.addTask {           // inherits priority + task-local values
                try await process(item)
            }
        }
        var results: [Result] = []
        for try await result in group {
            results.append(result)
        }
        return results
    }
}
// When processAll is cancelled → group cancelled → all child tasks cancelled
```

### async let — structured concurrent bindings

```swift
func loadDashboard() async throws -> Dashboard {
    async let user = fetchUser()        // starts immediately, runs concurrently
    async let stats = fetchStats()      // starts immediately, runs concurrently
    async let feed = fetchFeed()        // starts immediately, runs concurrently
    return try await Dashboard(         // awaits all three here
        user: user, stats: stats, feed: feed
    )
}
// All three requests run in parallel, not waterfall
// If any throws, the others are cancelled
```

Use `async let` when you have a fixed, known set of concurrent operations. Use `TaskGroup` when the number of concurrent operations varies.

### Unstructured tasks — opt-out of hierarchy

```swift
// Task {} — inherits actor context and priority, but NOT cancellation
let task = Task {
    await doWork()
}
task.cancel()  // must cancel manually

// Task.detached {} — inherits NOTHING — no actor, no priority, no task-locals
Task.detached(priority: .utility) {
    await doBackgroundWork()
}
```

Use unstructured tasks sparingly — they break the cancellation tree and leak if not stored and cancelled.

---

## Task Cancellation: Cooperative, Not Preemptive

Cancellation is a request, not a command. Your code must check for and respond to cancellation.

### Checking cancellation

```swift
func processLargeDataset(_ items: [Item]) async throws -> [Result] {
    var results: [Result] = []
    for item in items {
        try Task.checkCancellation()    // throws CancellationError if cancelled
        results.append(try await process(item))
    }
    return results
}

// Or: non-throwing check
guard !Task.isCancelled else { return [] }
```

### withTaskCancellationHandler — respond to cancellation immediately

```swift
func fetchWithCancellation(url: URL) async throws -> Data {
    try await withTaskCancellationHandler {
        try await URLSession.shared.data(from: url).0
    } onCancel: {
        // Called immediately when task is cancelled, even if URLSession is in flight
        // Note: runs on arbitrary thread — must be safe to call from any context
        URLSession.shared.invalidateAndCancel()
    }
}
```

### Propagating CancellationError correctly

```swift
// CORRECT: let CancellationError propagate
func load() async throws -> Data {
    try await withTaskCancellationHandler { ... } onCancel: { ... }
}

// WRONG: swallowing cancellation — caller never knows task was cancelled
func load() async -> Data? {
    try? await withTaskCancellationHandler { ... } onCancel: { ... }
    // CancellationError silently becomes nil — caller can't distinguish "no data" from "cancelled"
}
```

---

## Actors: Reentrancy and Isolation

### Actors are reentrant

When an actor `await`s something, it can interleave other work. State may change across an `await`:

```swift
actor BankAccount {
    var balance: Int = 1000

    func withdraw(_ amount: Int) async throws {
        guard balance >= amount else { throw InsufficientFunds() }
        // DANGER: balance can change here while we await
        await logTransaction(amount)   // actor suspends here, other callers can run
        // balance might now be < amount — the guard above is no longer valid
        balance -= amount
    }
}
```

**Fix**: don't assume state is unchanged across `await`. Re-validate after awaiting:

```swift
func withdraw(_ amount: Int) async throws {
    guard balance >= amount else { throw InsufficientFunds() }
    await logTransaction(amount)
    guard balance >= amount else { throw InsufficientFunds() }  // re-check
    balance -= amount
}
```

Or: move the `await` to after the state mutation:

```swift
func withdraw(_ amount: Int) async throws {
    guard balance >= amount else { throw InsufficientFunds() }
    balance -= amount                  // mutate state synchronously
    await logTransaction(amount)       // log after — safe, state already committed
}
```

### Actor isolation rules

```swift
actor DataStore {
    var items: [Item] = []

    // Synchronous — no suspension, no reentrancy window
    func append(_ item: Item) { items.append(item) }

    // nonisolated — runs on caller's context, cannot access actor state
    nonisolated func describe() -> String { "DataStore" }
}

// From outside the actor — must await
await store.append(item)

// From inside the actor — no await needed
func batchAppend(_ newItems: [Item]) {
    for item in newItems { append(item) }  // synchronous call within actor
}
```

### Actor contention — the bottleneck pattern

If many concurrent tasks all await the same actor, they queue up. The actor becomes a serial bottleneck:

```swift
// 100 tasks all awaiting the same actor sequentially = no real concurrency
actor Logger {
    func log(_ message: String) { ... }  // all 100 tasks wait their turn
}
```

**Diagnose**: Swift Concurrency instrument → actor lane shows long queues.

**Fix options**:
1. Make the hot method `nonisolated` if it doesn't need isolated state
2. Batch the work: pass arrays instead of calling the method per item
3. Use a non-actor mechanism (like `OSLog`) for logging — no actor overhead

```swift
// Fix: batch logging
actor Logger {
    func log(messages: [String]) { messages.forEach { write($0) } }  // one hop
}

// Fix: nonisolated computation
actor DataProcessor {
    nonisolated func computeResult(from input: Data) -> Result {
        // Pure computation — no actor state needed
        return Result(input)  // runs on caller's thread, zero actor overhead
    }
}
```

---

## @MainActor: Main Thread Guarantee

`@MainActor` is a global actor. Methods/properties annotated with it always run on the main thread.

```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var items: [Item] = []
    @Published var isLoading = false

    func load() async {
        isLoading = true           // main thread
        let result = await fetch() // suspends — leaves main thread during await
        items = result             // back on main thread after await
        isLoading = false          // main thread
    }
}
```

After an `await` in a `@MainActor` function, Swift resumes on the main thread automatically — no `DispatchQueue.main.async` needed.

### Hopping to main only when needed

```swift
func processAndDisplay(data: Data) async {
    // Heavy work on cooperative pool (background)
    let processed = await Task.detached(priority: .userInitiated) {
        expensiveProcessing(data)
    }.value

    // Update UI — hop to main
    await MainActor.run {
        self.displayResult(processed)
    }
}
```

`MainActor.run { }` is more explicit than making the whole function `@MainActor` when only a small part needs main thread execution.

---

## AsyncSequence: Streaming Data Pipelines

### Consuming AsyncSequence

```swift
func streamResults(from url: URL) async throws {
    let (bytes, _) = try await URLSession.shared.bytes(from: url)
    for try await line in bytes.lines {
        try Task.checkCancellation()
        await process(line)
    }
}
```

### Custom AsyncSequence

```swift
struct Countdown: AsyncSequence {
    typealias Element = Int
    let start: Int

    struct AsyncIterator: AsyncIteratorProtocol {
        var current: Int

        mutating func next() async -> Int? {
            guard current > 0 else { return nil }
            try? await Task.sleep(for: .seconds(1))
            defer { current -= 1 }
            return current
        }
    }

    func makeAsyncIterator() -> AsyncIterator { AsyncIterator(current: start) }
}

// Usage
for await count in Countdown(start: 5) {
    print(count)
}
```

### AsyncStream: Bridge callback APIs

```swift
func locationUpdates() -> AsyncStream<CLLocation> {
    AsyncStream { continuation in
        let delegate = LocationDelegate { location in
            continuation.yield(location)    // push value into stream
        }
        continuation.onTermination = { _ in
            delegate.stopUpdating()         // cleanup when stream is cancelled
        }
        delegate.startUpdating()
    }
}

// Consume:
for await location in locationUpdates() {
    await processLocation(location)
}
```

`AsyncStream.Continuation.yield` is safe to call from any thread. `onTermination` handles cancellation cleanup.

---

## Custom Executors (Swift 5.9+)

Replace the default actor executor with a custom one — e.g., run an actor on a specific serial DispatchQueue:

```swift
actor DatabaseActor {
    nonisolated var unownedExecutor: UnownedSerialExecutor {
        dbQueue.asUnownedSerialExecutor()  // run on a dedicated queue
    }
    private let dbQueue = DispatchSerialQueue(label: "com.app.db")

    func query(_ sql: String) -> [Row] {
        // Executes on dbQueue — can use APIs that require a specific thread/queue
        db.query(sql)
    }
}
```

Use when integrating with libraries that require a specific queue or thread (Core Data with a private context, SQLite, some audio APIs).

---

## Task-Local Values: Structured Context Propagation

Task-local values propagate automatically through the task hierarchy — no need to thread parameters through every function:

```swift
enum RequestContext {
    @TaskLocal static var traceID: String = ""
}

func handleRequest(id: String) async {
    await RequestContext.$traceID.withValue(id) {
        await fetchData()    // sees traceID = id
        await processData()  // sees traceID = id
        // All child tasks also see traceID = id
    }
}

func logError(_ error: Error) {
    print("[\(RequestContext.traceID)] Error: \(error)")
}
```

Task-local values are read-only inside tasks and propagate to structured child tasks and task groups, but NOT to unstructured `Task { }` or `Task.detached { }`.

---

## Clocks and Time

Swift 5.7+ provides type-safe clock APIs:

```swift
// ContinuousClock — wall time, advances while device sleeps
let clock = ContinuousClock()
let elapsed = await clock.measure {
    await doWork()
}
print("Took: \(elapsed)")

// SuspendingClock — time the task was actually running (excludes sleep time)
try await Task.sleep(until: .now + .seconds(5), clock: .continuous)
try await Task.sleep(until: .now + .seconds(5), clock: .suspending)
```

Prefer `ContinuousClock` for user-facing timeouts. `SuspendingClock` for measuring actual computation time excluding device sleep.

---

## Common Patterns and Anti-Patterns

### Anti-pattern: Task store without cancellation

```swift
// LEAK: task runs indefinitely, no way to cancel
class ViewModel {
    func refresh() {
        Task { await loadData() }  // stored nowhere, can't be cancelled
    }
}

// CORRECT: store and cancel on deinit
class ViewModel {
    private var refreshTask: Task<Void, Never>?

    func refresh() {
        refreshTask?.cancel()
        refreshTask = Task { await loadData() }
    }

    deinit { refreshTask?.cancel() }
}
```

### Anti-pattern: Ignoring cancellation in loops

```swift
// BAD: loop runs to completion even if task is cancelled
for item in items {
    await process(item)
}

// GOOD: check cancellation each iteration
for item in items {
    try Task.checkCancellation()
    try await process(item)
}
```

### Anti-pattern: Task.detached for everything

```swift
// BAD: loses priority, actor context, task-locals
Task.detached { await doWork() }

// GOOD: structured task inherits context
Task { await doWork() }

// Task.detached ONLY when you explicitly need to escape the actor/priority
Task.detached(priority: .background) { await lowPriorityWork() }
```

### Anti-pattern: Semaphore in async context

```swift
let semaphore = DispatchSemaphore(value: 1)

// DEADLOCK/HANG risk: blocks cooperative thread
Task {
    semaphore.wait()   // blocks — can starve the thread pool
    await doWork()
    semaphore.signal()
}

// CORRECT: use actor for mutual exclusion in async context
actor GuardedResource {
    func use() async { await doWork() }  // serialized by actor
}
```

---

## Instruments: Swift Concurrency

### Swift Concurrency template (Instruments, Xcode 13.2+)

Tracks:
- **Task creation** — when tasks start and their priority
- **Task state** — running, suspended, enqueued
- **Actor isolation** — which actor is running, how long tasks wait in the actor queue
- **Cooperative thread pool** — utilization, thread count

### Diagnosing actor contention

1. Open Swift Concurrency instrument
2. Find an actor with many tasks in "enqueued" state → contention
3. Look at what the actor is doing while tasks queue up
4. Fix: make methods `nonisolated`, reduce actor state, batch operations

### Diagnosing thread pool starvation

In Swift Concurrency instrument:
- If all cooperative threads are "running" (not "suspended") but throughput is low → threads are blocked
- Find which tasks are running: if they're in `wait`, `lock`, or system calls → blocking

Use the **Thread Performance Checker** (scheme → Diagnostics) to auto-detect blocking calls on cooperative threads.

---

## Code Review Checklist

- [ ] Does every unstructured `Task {}` get stored and cancelled when no longer needed?
- [ ] Are `async let` or `withTaskGroup` used for concurrent work (not sequential `await`)?
- [ ] Does code in loops call `try Task.checkCancellation()` to support cancellation?
- [ ] After `await` inside an actor, is state re-validated (reentrancy guard)?
- [ ] Are `DispatchSemaphore.wait()`, `Thread.sleep`, sync I/O absent from async contexts?
- [ ] Is `Task.detached` used only when intentionally escaping actor context or priority?
- [ ] Are hot actor methods that don't need isolated state marked `nonisolated`?
- [ ] Is `AsyncStream.onTermination` set to clean up resources when the stream is cancelled?
- [ ] Are `@TaskLocal` values used for cross-cutting context (tracing, request ID) instead of passed parameters?
- [ ] Does error handling let `CancellationError` propagate (not swallowed with `try?`)?
- [ ] Is `withTaskCancellationHandler` used when wrapping callback APIs that need immediate cancellation?
