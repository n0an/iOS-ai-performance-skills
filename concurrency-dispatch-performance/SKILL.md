---
name: concurrency-dispatch-performance
description: iOS concurrency and dispatch performance expert. Covers GCD queues, OperationQueue, async/await, actors, thread explosion, priority inversion, deadlocks, main thread protection, QoS classes, Instruments profiling (Time Profiler, Thread Performance Checker). Trigger when reviewing concurrent code, DispatchQueue usage, actor isolation, async Task hierarchies, performance bottlenecks on main thread, or threading crashes.
---

# iOS Concurrency & Dispatch Performance Expert

## The Concurrency Model: Two Eras

### GCD / OperationQueue (pre-Swift 5.5)
Thread-based. GCD manages a thread pool; you submit work to queues. Risk: thread explosion.

### Swift Structured Concurrency (Swift 5.5+)
Cooperative thread pool (≈ CPU core count threads). Tasks are suspended and resumed, not blocked. Risk: blocking cooperative threads.

**Both coexist.** Mixing them requires care — blocking a cooperative thread with GCD's `sync` or `Semaphore.wait` starves the pool.

---

## GCD: Queue Types and QoS

### Serial vs Concurrent

```swift
// Serial: tasks execute one at a time, in submission order
let serial = DispatchQueue(label: "com.app.serial")

// Concurrent: tasks execute in parallel (up to thread pool capacity)
let concurrent = DispatchQueue(label: "com.app.concurrent", attributes: .concurrent)

// Global queues — system-provided concurrent queues by QoS
DispatchQueue.global(qos: .userInteractive)  // highest — animations, input
DispatchQueue.global(qos: .userInitiated)    // immediate result needed
DispatchQueue.global(qos: .utility)          // long-running, user-visible progress
DispatchQueue.global(qos: .background)       // not user-visible, lowest priority
```

### QoS classes — pick the right one

| QoS | Use for | Thread priority |
|---|---|---|
| `.userInteractive` | Main thread work, animations | Highest |
| `.userInitiated` | Immediate response to user tap | High |
| `.utility` | Progress bar operations, I/O | Medium |
| `.background` | Sync, backups, prefetch | Lowest |
| `.default` | When you don't specify | Medium-low |

**Under-QoS pitfall**: submitting UI work to `.background` makes it run at lowest OS priority — janky UI. Over-QoS pitfall: submitting all work at `.userInteractive` starves background tasks and drains battery.

---

## Thread Explosion — The Core GCD Danger

GCD creates new threads when all existing threads are blocked (waiting on locks, semaphores, sync I/O). If your code creates many concurrent operations that each block, GCD can spawn hundreds of threads.

Symptoms: high memory usage (each thread stack = 512 KB default), CPU thrashing, increased latency for all tasks.

**Bad pattern:**
```swift
// Creates a thread per request, all potentially blocking on network/disk
for item in items {
    DispatchQueue.global().async {
        let result = blockingFetch(item)  // blocks thread
        // ...
    }
}
```

**Fix with OperationQueue:**
```swift
let queue = OperationQueue()
queue.maxConcurrentOperationCount = 4  // caps threads
for item in items {
    queue.addOperation { process(item) }
}
```

**Fix with Swift concurrency:**
```swift
await withTaskGroup(of: Result.self) { group in
    for item in items {
        group.addTask { await fetch(item) }  // suspends, never blocks a thread
    }
    for await result in group { collect(result) }
}
```

---

## Deadlocks: Patterns and Prevention

### Sync on the current queue

```swift
// DEADLOCK: serial queue dispatches sync to itself
let queue = DispatchQueue(label: "com.app.serial")
queue.async {
    queue.sync {  // blocks waiting for queue to empty — which it can't, it's blocked
        doWork()
    }
}
```

Never call `queue.sync { }` from a closure already running on the same serial queue.

### Main thread deadlock

```swift
// DEADLOCK: any thread syncing to main while main waits for it
DispatchQueue.global().async {
    DispatchQueue.main.sync {  // waits for main
        update()
    }
}
// Meanwhile main thread is blocking on the global queue result
```

Never call `DispatchQueue.main.sync` from a non-main thread. Use `DispatchQueue.main.async` or `await MainActor.run { }`.

---

## Main Thread Protection

The main thread runs the run loop, handles UI events, and drives CADisplayLink (60/120 fps). Any work exceeding ~8 ms blocks a frame.

### What must run on main thread
- All UIKit / SwiftUI mutations
- `CALayer` property changes (unless you're only preparing content)
- `UIApplication` access

### Detecting main thread violations

```swift
// In debug builds, assert you're on the main thread
extension UIView {
    func safeAddSubview(_ view: UIView) {
        assert(Thread.isMainThread, "UI updates must be on main thread")
        addSubview(view)
    }
}
```

Enable **Thread Sanitizer** (TSan) in scheme settings to catch data races. Enable **Main Thread Checker** (always on in debug) to catch UIKit access from background threads.

### Moving work off main thread

```swift
// Pattern: compute on background, apply on main
Task {
    let processed = await Task.detached(priority: .utility) {
        expensiveComputation(data)  // runs on cooperative pool, not main
    }.value
    await MainActor.run {
        self.result = processed  // UI update on main
    }
}
```

---

## DispatchGroup — Waiting for Multiple Async Operations

```swift
let group = DispatchGroup()
var results: [String: Data] = [:]
let lock = NSLock()

for url in urls {
    group.enter()
    URLSession.shared.dataTask(with: url) { data, _, _ in
        defer { group.leave() }
        if let data = data {
            lock.lock(); results[url.absoluteString] = data; lock.unlock()
        }
    }.resume()
}

group.notify(queue: .main) {
    // All done — safe to use results
    process(results)
}
```

**Prefer task groups in modern Swift** — `withTaskGroup` handles cancellation and error propagation automatically.

---

## Actors: Isolation and Performance

### Actor basics

An actor serializes access to its state — only one caller executes inside the actor at a time. Actor methods are implicitly `async` from outside.

```swift
actor Counter {
    private var value = 0
    func increment() { value += 1 }
    func get() -> Int { value }
}

let counter = Counter()
await counter.increment()  // actor hop: suspends caller, runs on actor
```

### Minimize actor hops

Each `await` on an actor method is a potential context switch. Batch work inside the actor:

```swift
// BAD: 1000 actor hops
for _ in 0..<1000 { await counter.increment() }

// GOOD: 1 actor hop
actor Counter {
    func incrementBy(_ n: Int) { value += n }  // nonisolated: not needed
}
await counter.incrementBy(1000)
```

### @MainActor

`@MainActor` is a global actor that runs work on the main thread:

```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var items: [Item] = []

    func load() async {
        let fetched = await fetchItems()  // leaves MainActor during await
        items = fetched  // back on MainActor when assigning @Published
    }
}
```

Mark view models and view controllers `@MainActor` to eliminate manual `DispatchQueue.main.async` calls.

---

## Blocking the Cooperative Thread Pool

**The most common Swift concurrency bug**: calling blocking APIs on cooperative threads.

```swift
// BAD: blocks a cooperative thread — starves other async tasks
Task {
    Thread.sleep(forTimeInterval: 1)  // blocks the thread
    semaphore.wait()                  // blocks the thread
    let data = try! Data(contentsOf: url)  // synchronous I/O blocks thread
}

// GOOD: suspend without blocking
Task {
    try await Task.sleep(for: .seconds(1))   // suspends, frees the thread
    let (data, _) = try await URLSession.shared.data(from: url)  // async I/O
}
```

If you must call a blocking API, wrap it in `Task.detached` with a dedicated thread or `DispatchQueue`:

```swift
// Escape to a non-cooperative thread for blocking work
Task {
    let result = try await withCheckedThrowingContinuation { continuation in
        DispatchQueue.global(qos: .utility).async {
            do {
                let data = try blockingSyncOperation()
                continuation.resume(returning: data)
            } catch {
                continuation.resume(throwing: error)
            }
        }
    }
}
```

---

## Continuation Bridging: Callback → async/await

```swift
// Bridge completion-handler APIs to async/await
func fetchData(from url: URL) async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        URLSession.shared.dataTask(with: url) { data, _, error in
            if let error = error {
                continuation.resume(throwing: error)
            } else {
                continuation.resume(returning: data ?? Data())
            }
        }.resume()
    }
}
```

Rules:
- `continuation.resume` must be called **exactly once** — more than once = crash, zero = task leaks forever
- Use `withCheckedContinuation` in debug (validates the rule), `withUnsafeContinuation` in release only if profiling shows overhead

---

## Priority Inversion

Priority inversion: a high-priority task is waiting on a resource held by a low-priority task.

GCD handles this automatically for `DispatchQueue` with priority inheritance. Swift's cooperative scheduler handles it for actors.

**Manual lock inversion** (not automatically resolved):
```swift
// High-priority task blocked by low-priority lock holder
lock.lock()        // low-priority thread holds this
// high-priority thread calls lock.lock() — blocked at low priority
```

Prefer actors over manual locks when possible — the runtime handles priority donation.

---

## OperationQueue: Dependencies and Cancellation

```swift
let operationQueue = OperationQueue()
operationQueue.maxConcurrentOperationCount = 3  // controls parallelism

let fetchOp = AsyncBlockOperation { await fetchData() }
let processOp = AsyncBlockOperation { await processData() }
let saveOp = BlockOperation { saveData() }

// Dependency: processOp waits for fetchOp to finish
processOp.addDependency(fetchOp)
saveOp.addDependency(processOp)

operationQueue.addOperations([fetchOp, processOp, saveOp], waitUntilFinished: false)

// Cancellation propagates through dependency graph
operationQueue.cancelAllOperations()
```

Check `isCancelled` inside operation blocks and return early to honor cancellation.

---

## Instruments: Finding Concurrency Issues

### Time Profiler
- Look for the main thread call stack — anything blocking main for >8ms is a candidate
- Collapse system libraries, focus on your code
- "Heavy" view shows the most expensive call paths

### Thread Performance Checker (Xcode 13+)
- Automatically catches: main thread UI updates from background, non-main thread UIKit access
- Enable in scheme → Diagnostics → Thread Performance Checker

### Swift Concurrency template (Instruments, Xcode 13.2+)
- Shows task creation, actor isolation, awaits, task cancellation
- Visualize where tasks spend time: running vs suspended vs enqueued

### Hangs instrument
- Shows main thread hangs > 250ms
- Provides stack trace at the moment of the hang

---

## Code Review Checklist

- [ ] Are serial queues used for shared mutable state (not concurrent + lock)?
- [ ] Is `DispatchQueue.main.sync` absent (it will deadlock if called from non-main)?
- [ ] Is `maxConcurrentOperationCount` set on OperationQueues doing I/O (prevents thread explosion)?
- [ ] Are Swift Tasks never blocking the cooperative thread pool with `Thread.sleep`, semaphores, or sync I/O?
- [ ] Are continuation-bridged callbacks calling `resume` exactly once?
- [ ] Are actor hops minimized by batching work inside actors?
- [ ] Is `@MainActor` used on view models instead of manual `DispatchQueue.main.async`?
- [ ] Are heavy computations (JSON decoding, image processing) explicitly moved off main thread?
- [ ] Does any code capture `self` strongly in an async closure that outlives `self`? → Need `[weak self]`
- [ ] Is Thread Sanitizer enabled for testing to catch data races?
