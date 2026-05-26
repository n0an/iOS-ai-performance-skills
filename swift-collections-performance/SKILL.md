---
name: swift-collections-performance
description: Expert Swift collection performance optimization skill. Covers sorted collections, Copy-On-Write semantics, Red-Black Trees, B-Trees, unsafe buffers, and benchmarking for iOS/macOS. Trigger when user asks to optimize collection code, review data structure choices, implement custom collections, or profile collection performance.
---

# Swift Collections Performance Expert

You are an expert in optimizing Swift collection types, based on deep knowledge of how Swift's memory model, value semantics, and data structures interact. When asked to review, optimize, or implement collections for iOS/macOS, apply these principles.

## Core Mental Model

Always evaluate a collection across three axes:
1. **Algorithmic complexity** — big-O for `contains`, `insert`, `remove`, iteration
2. **Constant factors** — cache locality, allocation overhead, ARC traffic
3. **Value semantics cost** — how expensive is COW when storage is shared vs. unique

Big-O rarely tells the full story. A B-Tree and a Red-Black Tree both have O(log n) insert, but the B-Tree is often 5–10× faster in practice because of cache locality.

---

## Data Structure Selection Guide

### SortedArray — best for small or read-heavy sets (≤ ~1000 elements)

- `contains`: O(log n) via binary search — extremely cache-friendly, fits in L1
- `insert`: O(n) — linear shift, but fast for small n due to contiguous memory
- Ideal when: insertions are rare, reads dominate, element count is bounded

Binary search implementation pattern:
```swift
func index(for element: Element) -> Int {
    var start = 0
    var end = elements.count
    while start < end {
        let mid = start + (end - start) / 2  // avoids overflow vs (start+end)/2
        if elements[mid] < element { start = mid + 1 }
        else { end = mid }
    }
    return start
}
```
Always compute midpoint as `start + (end - start) / 2`, not `(start + end) / 2`.

### Red-Black Tree — balanced O(log n) for all operations, poor cache behavior

- All operations: O(log n) guaranteed
- Each node is a separate heap allocation → pointer chasing → L1/L2 misses
- Pure value-semantics variant (enum-based): elegant but copies entire path on every mutation
- COW variant (class-based nodes): better mutation performance, requires careful `isKnownUniquelyReferenced` usage

Use when: you need guaranteed log n worst case and element count is large, but access patterns are random.

### B-Tree — best overall for large sorted sets

- All operations: O(log n) with a very large base (order = 100–2000 children per node)
- Nodes store elements in contiguous sorted arrays → cache-friendly
- In practice, depth ≤ 5 for any realistic dataset
- ~99.8% of elements live in leaf nodes → optimize leaf access

**Optimal order**: tune so node element buffer ≈ L1 cache / 4:
```swift
public init() {
    let cacheSize = Self.l1CacheSize() ?? 32768
    let order = cacheSize / (4 * MemoryLayout<Element>.stride)
    self.init(order: Swift.max(16, order))
}

private static func l1CacheSize() -> Int? {
    var result: Int = 0
    var size = MemoryLayout<Int>.size
    let status = sysctlbyname("hw.l1dcachesize", &result, &size, nil, 0)
    return status == -1 ? nil : result
}
```

Use when: large sorted sets (thousands+), mixed read/write workloads.

---

## Copy-On-Write (COW) Implementation

### The fundamental rule
Before **any** mutation of a reference-type node, call `isKnownUniquelyReferenced`. If it returns `false`, clone first.

```swift
// CORRECT
mutating func makeRootUnique() -> Node? {
    if root != nil, !isKnownUniquelyReferenced(&root) {
        root = root!.clone()
    }
    return root
}

// WRONG — the `let` binding creates a temporary reference, making
// isKnownUniquelyReferenced always return false
mutating func makeRootUnique() -> Node? {
    if let root = self.root, !isKnownUniquelyReferenced(&root) { // BUG
        self.root = root.clone()
    }
    return self.root
}
```

Always pass the stored property directly to `isKnownUniquelyReferenced` — never a local copy.

### COW is silent on failure
If you copy less than necessary → value semantics break, mutations affect other variables.
If you copy more than necessary → performance degrades, no crash.
There is no runtime trap. Test with shared-storage benchmarks, not just correctness tests.

### Shared-storage benchmark pattern
```swift
func sharedInsertBenchmark(_ input: [Element]) {
    var set = Self()
    var copy = set
    for value in input {
        set.insert(value)
        copy = set  // force shared storage on every iteration
    }
    _ = copy
}
```
Run this alongside the normal benchmark. If sharedInsert is dramatically slower, COW is copying too much data per mutation.

---

## Index Design for Custom Collections

### Mutation count pattern
Embed a `mutationCount: Int64` in each node. Capture it when creating an index. Validate in subscript and `index(after:)`:

```swift
struct Index: Comparable {
    weak var root: Node?
    var mutationCount: Int64
    var path: [Int]  // slot indices from root to element

    func validate(for tree: MyTree) {
        precondition(mutationCount == tree.root?.mutationCount,
                     "Index used after collection mutation")
    }
}
```

Use `Int64` for `mutationCount` to avoid overflow even on 32-bit systems. Only the root node's count needs tracking for invalidation purposes — storing it in every node wastes memory.

### Index complexity requirements
- `index(after:)` must be O(1) amortized — use path-based indices, not searching
- `startIndex`/`endIndex` must be O(1)
- For BidirectionalCollection, `index(before:)` must also be O(1) amortized

---

## Unsafe Buffers — When and How

Use `UnsafeMutablePointer<Element>` to replace `Array<Element>` inside collection nodes **only** when profiling confirms Array overhead is a bottleneck (typically 10–20% for B-Tree nodes).

### Allocation
```swift
let elements: UnsafeMutablePointer<Element> = .allocate(capacity: order)
```
Allocate `order` slots (one more than max size) so a node can temporarily overflow before splitting.

### Deallocation — always deinitialize before deallocate
```swift
deinit {
    elements.deinitialize(count: elementCount)
    elements.deallocate()
}
```
Skipping `deinitialize` leaks ARC references inside elements.

### Insert in the middle — shift right, then initialize
```swift
func insertElement(_ element: Element, at slot: Int) {
    (elements + slot + 1).moveInitialize(from: elements + slot, count: elementCount - slot)
    (elements + slot).initialize(to: element)
    elementCount += 1
}
```

### Split — use moveInitialize for reference-counted types
```swift
let c = elementCount - middle - 1
newNode.elements.moveInitialize(from: self.elements + middle + 1, count: c)
newNode.elementCount = c
self.elementCount = middle
```
`moveInitialize` is faster than copy+remove for reference types; no difference for plain value types.

### Use ContiguousArray for child arrays
```swift
var children: ContiguousArray<Node> = []
```
`ContiguousArray` avoids the Objective-C bridging overhead that `Array` of reference types can incur.

---

## Internal vs. Leaf Node Optimization

In a B-Tree of order ≥ 100, over 99% of elements are in leaf nodes. Separate optimization budgets accordingly:

- **Leaf nodes**: optimize aggressively — this is where iteration and insertion spend most time
- **Internal nodes**: can use smaller order (e.g., 16) to reduce COW copy cost when storage is shared

```swift
// When growing a new root level, use small order for internal nodes
if let splinter = splinter {
    let newRoot = Node(order: 16)  // small order for internal node
    newRoot.elements.initialize(to: splinter.separator)
    newRoot.elementCount = 1
    newRoot.children = [self.root, splinter.node]
    self.root = newRoot
}
```
This can reduce shared-insertion cost by 2–2.5× for large trees with no meaningful impact on in-place insertion.

---

## Performance Comparison Reference

| Structure | contains | insert (unique) | insert (shared) | iteration | Memory |
|-----------|----------|-----------------|------------------|-----------|--------|
| SortedArray | O(log n) fast | O(n) fast-small | O(n) full copy | O(n) fastest | Contiguous |
| RedBlackTree (enum) | O(log n) | O(log n) always copies path | same | O(n) slow | Fragmented |
| RedBlackTree (COW) | O(log n) | O(log n) in-place | O(log n) copies path | O(n) slow | Fragmented |
| BTree | O(log n) | O(log n) | O(log n)+node copy | O(n) fast | Semi-contiguous |
| BTree+UnsafeBuffer | O(log n) | O(log n) fastest | O(log n) | O(n) 2× faster | Semi-contiguous |

---

## Algebraic Data Types vs. Classes for Trees

### Enum-based (algebraic) — elegant, slow for mutations
```swift
indirect enum Tree<Element> {
    case empty
    case node(Tree, Element, Tree, Color)
}
```
Pros: pattern matching is clean, balancing code is readable.
Cons: every insertion copies the entire path from root to leaf; no way to do in-place mutation.

### Class-based with COW — better performance
```swift
class Node<Element> {
    var color: Color
    var value: Element
    var left: Node?
    var right: Node?
    var mutationCount: Int64 = 0
}
```
Pros: `isKnownUniquelyReferenced` enables in-place mutation when not shared.
Cons: boilerplate, fragile COW helpers, pointer chasing kills cache performance.

Choose class-based + COW whenever mutation performance matters.

---

## Common Pitfalls

1. **Midpoint overflow**: use `start + (end - start) / 2`, never `(start + end) / 2`
2. **COW false negative**: never create a `let` binding before `isKnownUniquelyReferenced` — it increments the refcount
3. **Forgetting deinitialize**: always call before `deallocate` when using unsafe pointers
4. **Wrong index after mutation**: capture `mutationCount` in index, validate on use
5. **Premature node order tuning**: profile first; default to `L1CacheSize / (4 * stride)` before tweaking
6. **Optimizing internal nodes first**: profile confirms leaves dominate; internal nodes are rarely the bottleneck

---

## When Reviewing Code — Checklist

- [ ] Is the data structure appropriate for the access pattern (read-heavy → SortedArray, large mixed → BTree)?
- [ ] Is binary search midpoint computed safely?
- [ ] Does COW check `isKnownUniquelyReferenced` on the stored property directly (not a local copy)?
- [ ] Is `mutationCount` captured in indices and validated before use?
- [ ] Is unsafe memory correctly deinitalized before deallocation?
- [ ] Does the shared-storage benchmark show acceptable performance?
- [ ] For B-Trees: is the order tuned to L1 cache size?
- [ ] For B-Trees: are internal nodes using smaller order when shared-insertion perf matters?
