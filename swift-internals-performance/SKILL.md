---
name: swift-internals-performance
description: Swift runtime internals and performance expert. Covers stack vs heap allocation, ARC, method dispatch (static/vtable/witness table/message), existentials vs generics, COW, structured concurrency, SIL optimizations, unsafe Swift, and modularization. Trigger when reviewing Swift code for dispatch overhead, ARC traffic, existential boxing costs, protocol design decisions, concurrency correctness, or build time issues.
---

# Swift Internals Performance Expert

Based on "Swift Internals" by Aaqib Hussain (Kodeco). Use when analyzing Swift code for runtime performance, type system design decisions, or architectural trade-offs.

---

## Memory: Stack vs Heap

### Where things live

| Type | Default location | Notes |
|---|---|---|
| `struct`, `enum` | Stack | O(1) allocation/deallocation via stack pointer |
| `class`, `actor` | Heap | Dynamic allocation + ARC overhead |
| Non-escaping closure | Stack | Captured variables inlined |
| Escaping closure | Heap | Captures moved to heap-allocated context |
| Large struct | Heap (sometimes) | Compiler may promote to heap if too large for stack |
| Struct captured by escaping closure | Heap | Lifetime extends beyond declaring scope |

### Stack is free; heap is not

Stack allocation = one integer adjustment to the stack pointer. No locks, no searching for free blocks.

Heap allocation = thread-safe search for a free memory block + bookkeeping. Multiple threads allocating simultaneously requires synchronization.

### Class instance memory layout

A class instance on the heap contains (minimum):
1. **isa pointer** (word 1) — points to type metadata: vtable, method pointers, conformance tables, superclass
2. **Reference count** (word 2) — ARC counter (strong), plus separate weak/unowned counts
3. **Stored properties** — inline after the header

On 64-bit, each word = 8 bytes. An empty class costs ≥16 bytes on the heap before any stored properties.

### Struct containing a class property

```swift
struct Owner {
    var name: String       // String contains a heap-allocated buffer internally
    var vehicle: Vehicle   // Vehicle is a class — this field is a pointer (8 bytes on stack)
}
```

The `Owner` struct itself is stack-allocated, but `vehicle` is a heap pointer. Copying `Owner` copies the pointer — both copies share the same `Vehicle`. Changes through either copy affect the same object.

### Class containing a struct property

```swift
class Player {
    var position: Position  // Position (struct) stored inline in the heap block
}
```

`position` is not on the stack; it's inside the `Player`'s heap allocation. It retains value semantics (copying `position` makes an independent copy), but lives on the heap.

---

## ARC: What it costs and how to minimize it

### What ARC does

Every `retain` and `release` on a class reference is an atomic increment/decrement. Atomics require memory barriers — expensive on multi-core hardware.

### ARC overhead patterns

```swift
// Expensive: each array element is a class — N retain/release pairs on copy
var items: [MyClass] = ...

// Cheaper: struct elements have no ARC unless they contain class properties
var items: [MyStruct] = ...
```

### Minimizing ARC traffic

1. **Prefer structs** for data that doesn't need shared identity
2. **Use `unowned` for back-references** where the reference can't outlive the owner (no overhead for unowned vs strong — but crashes on dangling access)
3. **Use `weak` only when the reference can become nil** — adds indirection through a side table
4. **Avoid unnecessary captures** in closures — each captured class reference adds a retain/release on closure creation and destruction
5. **`withExtendedLifetime(_:)`** when you need to guarantee an object lives through a scope without retaining in a stored property

```swift
// Avoid retaining self unnecessarily in tight loops
func processItems() {
    let items = self.items  // one retain for items array
    for item in items { item.process() }
    // items released here, not self retained per-item
}
```

---

## Method Dispatch: Performance Hierarchy

From fastest to slowest:

### 1. Static dispatch — zero runtime cost

The compiler resolves the call at compile time, can inline the body.

Triggered by:
- Methods on `struct` or `enum`
- `final class` or `final` method
- `private` / `fileprivate` methods (compiler infers final)
- Protocol extension default implementations (not in the requirement list)

```swift
struct Size {
    func area() -> Double { width * height }  // static dispatch
}
final class Engine {
    func start() { ... }  // static dispatch
}
```

### 2. V-Table dispatch — one pointer lookup

Each class has a vtable (array of function pointers). Non-final class methods use vtable dispatch. Cost: one memory load + indirect call.

```swift
class Animal {
    func sound() { }  // vtable dispatch — subclass may override
}
```

Add `final` to eliminate vtable dispatch when you know a method won't be overridden.

### 3. Protocol Witness Table dispatch — vtable + existential overhead

When calling a method via a protocol-typed variable (`any Protocol`), Swift:
1. Looks up the Protocol Witness Table (PWT) stored in the existential container
2. Calls through the PWT

The existential container adds overhead:
- For values ≤ 3 words: stored inline in the container (no heap allocation)
- For values > 3 words: heap allocation for the value + container stores a pointer

```swift
// Creates existential container — potential heap allocation for large conformers
let logger: any Logger = FileLogger()
logger.log("msg")  // PWT dispatch

// Concrete type — vtable or static dispatch
let logger: FileLogger = FileLogger()
logger.log("msg")  // vtable dispatch (or static if final)
```

### 4. Message dispatch — slowest, most dynamic

Used when bridging to Objective-C: `@objc`, `dynamic`, `#selector`, KVO, method swizzling.

Cost: ObjC runtime method lookup in a hash table per call.

Avoid in hot paths. Use only when dynamic behavior (swizzling, KVO) is explicitly required.

---

## Generics vs Existentials: The Critical Performance Decision

### Generics — compiler specializes, static dispatch

```swift
func process<T: Logger>(_ logger: T) {
    logger.log("msg")  // static dispatch after specialization
}
```

With `-O` optimization, the compiler generates a **specialized copy** of `process` for each concrete `T`. The concrete type is known at compile time → static dispatch → inlining possible. Zero existential overhead.

### Existentials (`any`) — dynamic dispatch, boxing

```swift
func process(_ logger: any Logger) {
    logger.log("msg")  // PWT dispatch, existential container
}
```

No specialization. The same function body handles all conformers at runtime. Heap allocation possible for large values.

### Decision rule

| Use case | Prefer |
|---|---|
| Single concrete type known at call site | Concrete type or generics |
| Heterogeneous collection (different types in same array) | `any Protocol` |
| Protocol has `associatedtype` or `Self` | Generics (`some` / `<T: P>`) |
| SwiftUI view returns | `some View` (opaque, no boxing) |
| Stored property that can hold different conformers | `any Protocol` |

### `some` vs `any`

```swift
// some: opaque type — compiler knows the concrete type, no boxing
// One specific type must always be returned from a given call site
func makeLogger() -> some Logger { ConsoleLogger() }

// any: existential — runtime flexibility, boxing cost
// Different types can be returned at different times
func makeLogger(_ useFile: Bool) -> any Logger {
    useFile ? FileLogger() : ConsoleLogger()
}
```

`some` is zero-cost at the call site compared to `any`. Use `some` in return positions whenever the concrete type is always the same.

---

## Structured Concurrency: Performance Patterns

### Task hierarchy and cancellation

```swift
// Structured — child tasks cancelled when parent cancels
await withTaskGroup(of: Result.self) { group in
    for item in items {
        group.addTask { await process(item) }
    }
    // collect results
}
```

### Actor isolation — avoid unnecessary hops

Each `await` crossing an actor boundary is a context switch. Minimize actor hops in hot paths:

```swift
actor DataStore {
    // BAD: caller awaits twice, two context switches
    func getValue() -> Int { ... }
    func getOtherValue() -> Int { ... }
    
    // GOOD: batch the work inside the actor, one context switch
    func getBothValues() -> (Int, Int) { (value, otherValue) }
}
```

### `nonisolated` for pure computations

```swift
actor NetworkManager {
    nonisolated func parseResponse(_ data: Data) -> Model {
        // Pure function — no actor state access
        // Caller doesn't pay the actor hop cost
        return Model(data: data)
    }
}
```

### Task priority and cooperative threading

- `.userInteractive` / `.userInitiated`: run immediately
- `.utility` / `.background`: may be deferred
- Don't block cooperative threads with `Thread.sleep` or synchronous I/O — use `Task.sleep` and async I/O

---

## Compiler and SIL: What the Optimizer Does

The Swift compiler emits **Swift Intermediate Language (SIL)** before generating machine code. Key optimizations:

- **Inlining**: replaces function calls with the function body (eliminates call overhead + enables further optimization)
- **Generic specialization**: generates concrete versions of generic functions for each type used
- **Devirtualization**: converts vtable calls to direct calls when the compiler can prove the concrete type
- **ARC optimization**: eliminates redundant retain/release pairs
- **Ownership optimization**: avoids copies when the original value is about to be destroyed

**Help the compiler**:
- Add `@inline(__always)` on tiny, hot functions if the compiler doesn't inline them automatically
- Add `@inline(never)` on large functions that shouldn't be inlined (reduces code size)
- Mark methods `final` to enable devirtualization
- Use `-O` (optimize for speed) vs `-Onone` (debug) — `-O` is required for ARC optimizations to fire

---

## Modularization: Build Time and Binary Size

### Static vs dynamic linking

| | Static library / framework | Dynamic framework |
|---|---|---|
| Launch time | No overhead | Dynamic linker work at startup |
| Binary size | Code embedded in app binary | Separate file, shared if used by multiple targets |
| Build time | Recompiles on change | Independent compilation |
| Symbol visibility | All symbols visible to linker | Explicit exports only |

**For iOS apps**: prefer static frameworks for dependencies. Dynamic frameworks increase app launch time because dyld must load and link them at startup. Apple's own frameworks (UIKit, Foundation) are pre-linked and don't add cost.

### Build time optimizations

- **Whole-module optimization** (`-whole-module-optimization`): cross-file inlining and specialization, slower build but faster runtime
- **Incremental builds**: avoid touching files that don't need to change
- **Module boundaries**: large modules rebuild entirely on any change — split large modules into smaller ones with clear boundaries
- **`@_implementationOnly import`**: hides internal dependencies from the module's public interface, reduces transitive recompilation

---

## Code Review Checklist

- [ ] Is `any Protocol` used where a generic `<T: Protocol>` would enable specialization and static dispatch?
- [ ] Are class methods marked `final` where no override is intended?
- [ ] Are large value types (structs) passed as `inout` or stored as class properties to avoid copies?
- [ ] Are actor boundaries crossed only when necessary — is work batched inside actors?
- [ ] Are `@objc dynamic` and KVO used only where required, not as a default pattern?
- [ ] Are escaping closures capturing `self` creating retain cycles? (Need `[weak self]`)
- [ ] Is `some` used instead of `any` in return positions where the type is always the same?
- [ ] Are `nonisolated` functions used for pure computations on actors?
- [ ] Is `-O` optimization enabled in the release build configuration?
