---
description: Swift generics compiler internals expert. Covers generic signatures, requirements, substitution maps, conformance resolution, existential types vs generics, opaque types, conditional conformances, and how the Requirement Machine works. Trigger when reviewing complex generic code, protocol design with associated types, conditional conformances, opaque return types, protocol witness tables, or compiler errors about generic constraints.
---

# Swift Generics Internals Expert

Based on "Compiling Swift Generics" by Slava Pestov (Apple Swift compiler team). Use when debugging complex generic type errors, designing protocol hierarchies, or understanding why the compiler accepts/rejects certain generic code.

---

## The Four Goals of Swift Generics

1. **Independent type checking** — generic definitions are checked without instantiation. `func f<T: Equatable>(_ x: T)` is checked once; concrete call sites don't re-verify the body.
2. **Full abstraction** — a generic function can use only what is guaranteed by its constraints; no peeking at the concrete type.
3. **Efficient code generation** — specialization (concrete copies per type) or shared thunks (one copy with runtime type info).
4. **Expressive constraints** — same-type requirements, superclass constraints, protocol compositions.

---

## Generic Signatures

A **generic signature** is the complete set of generic parameters and their requirements attached to a declaration. It is normalized into a canonical form by the compiler.

```swift
func merge<T, U>(_ a: T, _ b: U) -> [T] where T: Collection, T.Element == U
// Generic signature: <T, U where T: Collection, T.Element == U>
```

### Requirements

| Requirement | Syntax | Meaning |
|---|---|---|
| Conformance | `T: Protocol` | T must conform to Protocol |
| Superclass | `T: SomeClass` | T must be SomeClass or a subclass |
| Same-type | `T.Element == U` | Two types must be identical |
| Layout | `T: AnyObject` | T must be a class (reference type) |

### Reduced types

Inside a generic signature, the compiler normalizes types using the requirements. `T.Element` where `T: Collection` reduces to the Element associated type. Two types that are constrained to be equal become interchangeable.

### Requirement minimization

The compiler removes redundant requirements. If you write:
```swift
func f<T>(_ x: T) where T: Hashable, T: Equatable
```
`Hashable: Equatable`, so `T: Equatable` is implied and minimized away. The compiler stores only the minimal set.

---

## Generic Declarations and Constraints

### The `where` clause

Use `where` for constraints that can't be expressed in the angle brackets:

```swift
// Angle bracket form (simple conformances)
func zip<A: Sequence, B: Sequence>(_ a: A, _ b: B) -> [(A.Element, B.Element)]

// Where clause (associated type constraints)
func equal<T: Sequence>(_ lhs: T, _ rhs: T) -> Bool where T.Element: Equatable

// Same-type requirement
func convert<T: StringProtocol>(_ value: T) -> String where T == String.SubSequence
```

### Opaque parameters (SE-0341)

```swift
// Equivalent to a generic function with an implicit type parameter
func process(_ value: some Equatable) -> Bool
// Same as: func process<T: Equatable>(_ value: T) -> Bool
```

`some` in parameter position is syntactic sugar for a fresh generic parameter. The compiler specializes for each concrete type at call sites.

---

## Conformances and Protocol Witness Tables

### Conformance lookup

When the compiler sees `x.method()` where `x` has a protocol type, it looks up the conformance to find which implementation to call. The conformance maps the concrete type's methods to the protocol's requirements.

### Type witnesses

For associated types, the conformance provides a **type witness** — the concrete type that satisfies the associated type:

```swift
protocol Container {
    associatedtype Element
    func element(at index: Int) -> Element
}

struct IntStack: Container {
    typealias Element = Int  // type witness (can be inferred)
    func element(at index: Int) -> Int { storage[index] }
}
```

### Conditional conformances

A type can conform to a protocol only when its type parameters satisfy additional constraints:

```swift
extension Array: Equatable where Element: Equatable {
    static func == (lhs: [Element], rhs: [Element]) -> Bool { ... }
}
```

The conformance is conditional — `[Int]` is `Equatable`, but `[UIView]` is not. The compiler checks this at each use site.

### Protocol inheritance conformances

```swift
protocol Animal {}
protocol Pet: Animal {}

struct Dog: Pet {}
```

`Dog` implicitly conforms to `Animal` through `Pet`. The compiler generates an inherited conformance that delegates to the `Pet` conformance.

---

## Substitution Maps

A **substitution map** resolves generic parameters to concrete types at a specific call site. When calling `Array<Int>.append(_:)`, the substitution map is `{Element → Int}`.

### Composing substitutions

Substitution maps compose: if `f<T>` calls `g<U>` where `U = T.Element`, and `f` is called with `T = [Int]`, then `g` sees `U = Int`.

### Context substitution maps

Each declaration has a context substitution map that maps its own type parameters to themselves (identity substitution). This is used when type-checking the declaration body independently of any call site.

---

## Generic Environments and Archetypes

During type checking of a generic function body, type parameters are represented as **archetypes** — placeholder types that carry their constraints. The archetype for `T: Equatable` knows it responds to `==`.

```swift
func f<T: Equatable>(_ x: T, _ y: T) -> Bool {
    return x == y  // valid: archetype T knows about ==
}
```

Archetypes are an internal compiler concept; they never appear in source code.

---

## Opaque Return Types (`some`)

```swift
func makeLogger() -> some Logger { ConsoleLogger() }
```

The compiler knows the concrete return type of each `return` statement. It verifies all returns produce the same concrete type. The caller sees only the protocol constraint.

**Difference from `any Logger`**:
- `some Logger`: compiler knows concrete type → static dispatch, no boxing
- `any Logger`: runtime flexibility → existential container, PWT dispatch

**Opaque archetypes** are used to represent `some` types inside the compiler. They differ from primary archetypes (from generic type parameters) in that each call site may see a different concrete type — but within a single function, the type is fixed.

---

## Existential Types and Their Limitations

An **existential** (`any Protocol`) erases the concrete type at runtime.

### The existential container

Values stored in an existential container:
- **Value buffer** (3 words inline): if the concrete value fits in 3 words (24 bytes on 64-bit), stored inline; otherwise, a heap allocation is made and the buffer stores a pointer
- **Type metadata pointer**: points to the type's metadata (isa-like)
- **Protocol Witness Table pointer(s)**: one per conformed protocol

### When existentials fail to compile

Protocols with `associatedtype` or `Self` requirements cannot be used as existentials in positions where the associated type must be known:

```swift
protocol Producer {
    associatedtype Output
    func produce() -> Output
}

// ERROR in function bodies that need to USE Output
func consume(_ p: any Producer) {
    let value = p.produce()  // ERROR: type of 'value' is unknown
    process(value)
}

// FIX: use generics
func consume<P: Producer>(_ p: P) {
    let value = p.produce()  // OK: Output is known as P.Output
    process(value)
}
```

**Exception**: if the associated type only appears in non-constrained return positions, the existential can be used:
```swift
func describe(_ p: any Producer) {
    print(p.produce())  // OK: Any absorbs the unknown Output type
}
```

### Self-conforming protocols

`Any` and `AnyObject` are existentials that also conform to themselves. Most protocols do NOT self-conform — `any Equatable` is not itself `Equatable`.

---

## Witness Thunks

When a protocol requirement's signature differs from the conforming method's signature (different generic context, different abstraction level), the compiler generates a **witness thunk** — a small wrapper function that bridges between them.

Thunks add a small overhead at the call site. This is unavoidable for existential dispatch but is eliminated by specialization.

---

## The Requirement Machine (Swift 5.6+)

The Requirement Machine replaced the older constraint solver for generic signature computation. It:
1. Converts requirements into rewrite rules
2. Computes the minimal canonical form via term rewriting (completion algorithm)
3. Answers queries about the signature (are two types equivalent? does type T conform to P?)

This is an internal compiler mechanism. Its correctness directly affects whether conditional conformances, same-type requirements, and complex protocol hierarchies compile correctly.

---

## Practical Design Rules

### Avoid over-constraining

```swift
// Over-constrained: requires Hashable when only Equatable is needed
func findFirst<T: Hashable>(_ items: [T], matching value: T) -> T? { ... }

// Better: minimum necessary constraint
func findFirst<T: Equatable>(_ items: [T], matching value: T) -> T? { ... }
```

### Prefer generic over existential for performance

```swift
// Existential — one function, runtime dispatch, boxing
func log(_ message: any CustomStringConvertible) { print(message.description) }

// Generic — specialized per type, static dispatch, no boxing
func log<T: CustomStringConvertible>(_ message: T) { print(message.description) }
```

For small concrete types (≤ 24 bytes), the existential inline buffer avoids heap allocation. For larger types (String, Array, custom structs) existentials heap-allocate.

### Conditional conformance design

Make conformances as narrow as possible:
```swift
// Too broad: forces Element to be Hashable even when you only need Equatable
extension MyCollection: Hashable where Element: Hashable { ... }
extension MyCollection: Equatable where Element: Equatable { ... }

// Add each separately with the minimum constraint each requires
```

### Same-type requirements for associated type unification

```swift
// Require that two sequences have the same element type
func zip<S1: Sequence, S2: Sequence>(_ s1: S1, _ s2: S2) where S1.Element == S2.Element
```

---

## Code Review Checklist

- [ ] Are generic constraints minimal — no `Hashable` where `Equatable` suffices?
- [ ] Is `any Protocol` used where `some Protocol` or `<T: Protocol>` would enable static dispatch?
- [ ] Are conditional conformances declared with the weakest element constraint that works?
- [ ] Are protocols with `associatedtype` used with generics rather than existentials at call sites?
- [ ] Is `some` used in return position when the concrete type is always the same?
- [ ] Are same-type requirements (`T.Element == U`) used to unify associated types across protocol boundaries?
- [ ] Is the `where` clause used for constraints that can't be expressed in angle brackets?
- [ ] Are protocol hierarchies designed so that `Hashable: Equatable` and similar relationships handle redundancy automatically?
