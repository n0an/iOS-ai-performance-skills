---
name: swiftui-performance
description: SwiftUI rendering performance expert. Covers view identity and lifetime, body recomputation, @State/@ObservableObject/@Observable granularity, equatable views, lazy containers, GeometryReader cost, task/onAppear timing, view modifier order, drawing performance, and Instruments profiling for SwiftUI. Trigger when reviewing SwiftUI views for unnecessary redraws, slow lists, excessive body calls, state architecture decisions, or animation jank.
---

# SwiftUI Performance Expert

## How SwiftUI Rendering Works

SwiftUI diffs the view tree on every state change. Understanding what triggers a redraw is the foundation of SwiftUI performance.

### The rendering cycle
1. State changes → SwiftUI calls `body` on affected views
2. SwiftUI diffs the new view tree against the previous one
3. Only changed subtrees are re-rendered on screen

**`body` is called frequently — keep it cheap.** No synchronous I/O, no heavy computation, no unnecessary allocations inside `body`.

---

## View Identity: Structural vs Explicit

SwiftUI uses identity to decide whether to update or replace a view.

### Structural identity — position in the view tree
```swift
// These are structurally different views — SwiftUI treats them as separate instances
if condition {
    Text("A")  // identity: "first branch of if"
} else {
    Text("B")  // identity: "second branch of if"
}

// AnyView breaks structural identity — avoid it
func makeView() -> AnyView {
    if condition { return AnyView(Text("A")) }
    return AnyView(Text("B"))
}
```

`AnyView` erases type information — SwiftUI cannot optimize the diff. It recreates the wrapped view on every change instead of updating it.

**Never use `AnyView` for performance.** Use `@ViewBuilder` or `some View` instead:

```swift
// GOOD: preserves structural identity, allows SwiftUI to optimize
@ViewBuilder
func makeView() -> some View {
    if condition { Text("A") }
    else { Text("B") }
}
```

### Explicit identity — `.id()` modifier
```swift
// Forces SwiftUI to destroy and recreate the view when id changes
TextField("Name", text: $name)
    .id(userID)  // recreates text field when userID changes (useful for resetting state)
```

Use `.id()` deliberately — it destroys state. Don't use it as a workaround for refresh problems.

---

## @State and Body Recomputation

Every `@State` change triggers a `body` call on the owning view and all its children that depend on it.

### Minimize state scope

```swift
// BAD: one large state object — any field change redraws everything
struct ContentView: View {
    @State private var name = ""
    @State private var age = 0
    @State private var isLoading = false
    @State private var items: [Item] = []
    // body uses all of these — every change redraws everything
}

// GOOD: push state down to the view that needs it
struct ContentView: View {
    var body: some View {
        VStack {
            NameField()      // owns name state
            AgeSlider()      // owns age state
            ItemList()       // owns items state
        }
    }
}
```

A child view only redraws when its own inputs change. Pushing state down = smaller redraw scope.

---

## @Observable vs @ObservableObject: Granularity

### @ObservableObject (pre-iOS 17) — whole-object granularity
```swift
class ViewModel: ObservableObject {
    @Published var name = ""
    @Published var count = 0
    @Published var items: [Item] = []
}

struct MyView: View {
    @ObservedObject var vm: ViewModel
    var body: some View {
        Text(vm.name)  // redraws when name, count, OR items change
    }
}
```

Any `@Published` property change publishes through `objectWillChange` → every view observing `vm` redraws, even views that don't use the changed property.

### @Observable (iOS 17+) — property-level granularity
```swift
@Observable
class ViewModel {
    var name = ""
    var count = 0
    var items: [Item] = []
}

struct MyView: View {
    var vm: ViewModel
    var body: some View {
        Text(vm.name)  // redraws ONLY when name changes
    }
}
```

`@Observable` tracks which properties are actually accessed in `body`. If `body` reads only `name`, changes to `count` don't trigger a redraw.

**Prefer `@Observable` on iOS 17+.** It's both simpler and more performant.

### Split view models for @ObservableObject

If you're on iOS 16 and can't use `@Observable`, split one large model into smaller ones:

```swift
// Each view only observes the model it needs
class HeaderViewModel: ObservableObject { @Published var title = "" }
class ListViewModel: ObservableObject { @Published var items: [Item] = [] }
```

---

## EquatableView: Skip the Diff

When SwiftUI calls `body`, it diffs the result. For complex views with stable, equatable inputs, skip the body call entirely:

```swift
struct ItemView: View, Equatable {
    let item: Item

    static func == (lhs: ItemView, rhs: ItemView) -> Bool {
        lhs.item.id == rhs.item.id && lhs.item.updatedAt == rhs.item.updatedAt
    }

    var body: some View {
        // expensive body — only called when == returns false
        VStack { ... }
    }
}

// Wrap with .equatable() to enable the optimization
ItemView(item: item).equatable()
```

`.equatable()` tells SwiftUI to skip `body` and the diff entirely when `==` returns `true`.

Only use when `body` is expensive. Don't add Equatable conformance everywhere — the `==` call itself has a cost.

---

## Lazy Containers: List and LazyVStack

### List — virtualized by default
`List` recycles cells like UITableView. Use it for long data sets.

```swift
// Efficient for 10,000+ items
List(items, id: \.id) { item in
    ItemRow(item: item)
}
```

### LazyVStack / LazyHStack — render on demand
```swift
ScrollView {
    LazyVStack(spacing: 8) {
        ForEach(items, id: \.id) { item in
            ItemRow(item: item)  // only created when scrolled into view
        }
    }
}
```

### VStack — all children created immediately
```swift
// BAD for long lists: creates all 10,000 views immediately
ScrollView {
    VStack {
        ForEach(items, id: \.id) { ItemRow(item: $0) }
    }
}
```

**Rule**: > ~20 dynamic items → use `List` or `LazyVStack`.

### Pinned section headers in LazyVStack
```swift
ScrollView {
    LazyVStack(pinnedViews: [.sectionHeaders]) {
        ForEach(sections) { section in
            Section(header: SectionHeader(section: section)) {
                ForEach(section.items) { ItemRow(item: $0) }
            }
        }
    }
}
```

---

## ForEach Identity: Stable IDs

SwiftUI uses `id` to track items in `ForEach`. Unstable IDs force recreation.

```swift
// BAD: index is unstable — any insertion changes all subsequent indices
ForEach(0..<items.count, id: \.self) { i in
    ItemView(item: items[i])
}

// GOOD: stable UUID from the data model
ForEach(items, id: \.id) { item in
    ItemView(item: item)
}
```

Items must conform to `Identifiable` or have stable, unique `id` values. If an item's `id` changes, SwiftUI destroys and recreates the view (losing any local state).

---

## Task and onAppear: Async Work Timing

```swift
struct ItemDetailView: View {
    let itemID: String
    @State private var detail: Detail?

    var body: some View {
        Group {
            if let detail { DetailContent(detail: detail) }
            else { ProgressView() }
        }
        .task(id: itemID) {
            // Cancels and restarts when itemID changes
            detail = try? await fetchDetail(id: itemID)
        }
    }
}
```

`.task(id:)` — cancels the previous task and starts a new one when `id` changes. Use it instead of `.onAppear` + manual cancellation.

`.onAppear` fires every time the view appears (including back-navigation). Use only for truly appearance-driven side effects, not data loading.

---

## GeometryReader: Use Sparingly

`GeometryReader` reads its container's size by filling all available space. It causes an extra layout pass.

```swift
// BAD: GeometryReader on every cell in a list = N extra layout passes
List(items) { item in
    GeometryReader { geo in
        ItemView(item: item, width: geo.size.width)
    }
}

// GOOD: read geometry once at the container level
GeometryReader { geo in
    List(items) { item in
        ItemView(item: item, width: geo.size.width)
    }
}

// BETTER: use flexible frames instead
ItemView(item: item)
    .frame(maxWidth: .infinity, alignment: .leading)
```

---

## Modifier Order and View Hierarchy

Modifiers create new views. Order matters for both correctness and performance.

```swift
// These are different — modifier order affects the view tree
Text("Hello")
    .background(Color.blue)  // background behind text bounds
    .padding()               // padding outside background

Text("Hello")
    .padding()               // padding first
    .background(Color.blue)  // background behind padded bounds
```

**Avoid deeply nested modifier chains** where a single view can be expressed with fewer layers. Each modifier wraps the view in a new type.

```swift
// One conditional instead of two modifiers
Text("Hello")
    .foregroundStyle(isActive ? .blue : .gray)
    .fontWeight(isActive ? .bold : .regular)

// Avoid: .if() extensions — they often force AnyView
```

---

## Canvas: Drawing Without View Overhead

For data-dense visualizations (charts, graphs, custom gauges) where individual subviews would be too expensive:

```swift
Canvas { context, size in
    // Direct drawing — no view tree overhead per element
    for (i, value) in values.enumerated() {
        let x = CGFloat(i) / CGFloat(values.count) * size.width
        let height = value * size.height
        let rect = CGRect(x: x, y: size.height - height, width: 4, height: height)
        context.fill(Path(rect), with: .color(.blue))
    }
}
.frame(height: 200)
```

`Canvas` renders all content in a single draw call. Use when you have 100+ visual elements that change together.

---

## TimelineView: Scheduled Redraws

For animations that update on a schedule (clocks, live data):

```swift
TimelineView(.periodic(from: .now, by: 1.0)) { timeline in
    ClockFace(date: timeline.date)
}
```

`TimelineView` only redraws on the schedule — not on every frame. More efficient than `Timer` + `@State` because it doesn't create extra state change notifications.

For 60fps animations, use `.animation` schedule:
```swift
TimelineView(.animation) { timeline in
    AnimatedView(time: timeline.date.timeIntervalSince1970)
}
```

---

## @StateObject vs @ObservedObject

```swift
struct ParentView: View {
    var body: some View {
        ChildView()  // recreates ChildView's @ObservedObject on every parent redraw
    }
}

struct ChildView: View {
    @ObservedObject var vm = ViewModel()  // BAD: new ViewModel on every redraw
    @StateObject var vm = ViewModel()    // GOOD: created once, owned by this view
}
```

`@StateObject` ties the object's lifetime to the view. `@ObservedObject` assumes the object is owned externally. Never initialize an `@ObservedObject` inside the view — use `@StateObject`.

---

## Instruments: SwiftUI Profiling

### Time Profiler — find expensive `body` calls
- Look for `SwiftUI.View.body.getter` in the call tree
- Drill into which specific view's `body` is expensive
- "Separate by Thread" to confirm main thread execution

### SwiftUI instrument (Xcode 12+)
- Shows view body invocations with duration
- "View Body Invocations" lane — each bar = one `body` call
- Long bars = expensive `body`
- Frequent bars on the same view = unnecessary redraws

### Finding unnecessary redraws
Add print to `body` temporarily:
```swift
var body: some View {
    let _ = Self._printChanges()  // prints what changed to trigger this body call
    return Text("Hello")
}
```

`Self._printChanges()` is a debug-only API that prints the diff reason to the console.

---

## Code Review Checklist

- [ ] Is `AnyView` absent from performance-sensitive view hierarchies? → Replace with `@ViewBuilder`
- [ ] Is state pushed as far down the view hierarchy as possible?
- [ ] Is `@Observable` used instead of `@ObservableObject` on iOS 17+ targets?
- [ ] Are long dynamic lists using `List` or `LazyVStack`, not `VStack`?
- [ ] Does `ForEach` use stable, data-model-based IDs (not array indices)?
- [ ] Are expensive views using `.equatable()` with a meaningful `==` implementation?
- [ ] Is `GeometryReader` avoided inside list cells?
- [ ] Are data loads using `.task(id:)` instead of `.onAppear`?
- [ ] Is `@StateObject` used for view-owned models (not `@ObservedObject` initialized in the view)?
- [ ] Are data-dense visualizations using `Canvas` instead of many individual views?
- [ ] Is `Self._printChanges()` temporarily added to diagnose unexpected redraws?
