---
name: memory-management-performance
description: iOS memory management and ARC performance expert. Covers retain cycles, weak/unowned, closure capture lists, memory warnings, NSCache, image memory, large allocations, heap profiling, Instruments (Allocations, Leaks, Memory Graph Debugger), and memory footprint optimization. Trigger when reviewing closure captures, delegate patterns, cache implementations, memory warnings handling, or diagnosing crashes from memory pressure.
---

# iOS Memory Management & ARC Performance Expert

## ARC Fundamentals

Swift uses Automatic Reference Counting. Every class instance has a reference count. When it drops to zero, `deinit` runs and memory is freed.

**Three reference strengths:**

| Keyword | Count | Nil on dealloc | Use when |
|---|---|---|---|
| `strong` (default) | +1 | No | Default ownership |
| `weak` | 0 | **Yes** → becomes `nil` | Back-references, delegates, may be nil |
| `unowned` | 0 | No → crash if accessed after dealloc | Back-references, guaranteed outlives owner |

---

## Retain Cycles: Patterns and Fixes

A retain cycle = A → B → A (strong), neither can reach zero count.

### Closures capturing self

```swift
class ViewModel {
    var onUpdate: (() -> Void)?

    func setup() {
        // CYCLE: onUpdate strong → closure → self strong
        onUpdate = { self.reload() }

        // FIX: weak capture
        onUpdate = { [weak self] in self?.reload() }

        // FIX: unowned (only if closure can't outlive self — e.g., UIView.animate)
        onUpdate = { [unowned self] in self.reload() }
    }
}
```

**When to use `unowned` vs `weak`:**
- `weak` — nullable reference; safe even if `self` is deallocated before the closure runs
- `unowned` — non-nullable; use only when the closure is guaranteed not to outlive `self` (e.g., completion handlers owned by `self` itself, UIView animation blocks in `deinit`-safe positions)

### Delegate pattern (always weak)

```swift
// Protocol must be class-bound for weak to work
protocol DataDelegate: AnyObject {
    func didUpdate()
}

class DataSource {
    weak var delegate: DataDelegate?  // weak — delegate owns DataSource, not reverse
}
```

Never store a delegate as a `strong` reference — it creates a cycle.

### Timer and CADisplayLink

`Timer` and `CADisplayLink` hold a strong reference to their `target`. If `self` is the target, `self` is kept alive by the timer even after the view controller is dismissed.

```swift
// FIX: use a proxy weak target
class TimerProxy: NSObject {
    weak var target: AnyObject?
    var selector: Selector

    init(target: AnyObject, selector: Selector) {
        self.target = target
        self.selector = selector
    }

    @objc func fire() {
        _ = target?.perform(selector)
    }
}

// Or: invalidate timer in deinit / viewDidDisappear
deinit {
    timer?.invalidate()
    timer = nil
}
```

### NotificationCenter observers (Swift)

Swift block-based observers:
```swift
// WRONG: block captures self strongly, observer token not stored → leak
NotificationCenter.default.addObserver(forName: .myNotification, object: nil, queue: .main) {
    _ in self.update()
}

// CORRECT: capture weakly + store and remove token
var token: NSObjectProtocol?

token = NotificationCenter.default.addObserver(forName: .myNotification, object: nil, queue: .main) {
    [weak self] _ in self?.update()
}

deinit {
    if let token { NotificationCenter.default.removeObserver(token) }
}
```

Selector-based observers (`addObserver(_:selector:name:object:)`) don't retain — but you still need to `removeObserver` in `deinit`.

---

## Memory Warnings: Handle or Crash

iOS terminates apps that use too much memory — no swap space on iPhone. The OS sends memory warnings progressively before terminating.

```swift
// UIViewController
override func didReceiveMemoryWarning() {
    super.didReceiveMemoryWarning()
    imageCache.removeAllObjects()  // free non-essential caches
    // Do NOT release data needed for current UI — only caches and off-screen content
}

// AppDelegate (global notification)
NotificationCenter.default.addObserver(
    self,
    selector: #selector(handleMemoryWarning),
    name: UIApplication.didReceiveMemoryWarningNotification,
    object: nil
)
```

**What to free:**
- Image caches (NSCache handles this automatically — use it)
- Off-screen pre-rendered content
- Prefetched data not yet displayed
- Decoded audio/video frames not currently playing

**What NOT to free:**
- Data backing the currently visible UI — freeing it causes blank cells/views

---

## NSCache: The Right Tool for In-Memory Caches

NSCache is thread-safe and auto-evicts under memory pressure. Never use `Dictionary` as a cache.

```swift
final class ImageCache {
    private let cache = NSCache<NSString, UIImage>()

    init() {
        cache.countLimit = 150          // max 150 items
        cache.totalCostLimit = 60 * 1024 * 1024  // 60 MB
    }

    func image(for key: String) -> UIImage? {
        cache.object(forKey: key as NSString)
    }

    func store(_ image: UIImage, for key: String) {
        let cost = Int(image.size.width * image.size.height * image.scale * image.scale * 4)
        cache.setObject(image, forKey: key as NSString, cost: cost)
    }

    func remove(for key: String) {
        cache.removeObject(forKey: key as NSString)
    }
}
```

Set `totalCostLimit` based on image byte size — `countLimit` alone doesn't account for image size variance.

---

## Large Allocations and Image Memory

### Image memory formula

`width × height × 4 bytes (RGBA) × scale²`

A 1000×1000 image at @3x = 1000 × 1000 × 4 × 9 = **36 MB** in memory, even if the file is 200 KB on disk.

### Downscale before displaying

Never load a 4000×3000 photo into a 100×100 thumbnail cell:

```swift
func downsampledImage(url: URL, targetSize: CGSize, scale: CGFloat) -> UIImage? {
    let options: [CFString: Any] = [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceShouldCacheImmediately: true,
        kCGImageSourceCreateThumbnailWithTransform: true,
        kCGImageSourceThumbnailMaxPixelSize: max(targetSize.width, targetSize.height) * scale
    ]
    guard let source = CGImageSourceCreateWithURL(url as CFURL, nil),
          let cgImage = CGImageSourceCreateThumbnailAtIndex(source, 0, options as CFDictionary)
    else { return nil }
    return UIImage(cgImage: cgImage, scale: scale, orientation: .up)
}
```

`CGImageSourceCreateThumbnailAtIndex` decodes only the pixels needed for the target size — far less memory than decoding the full image and scaling in UIKit.

### UIImage vs CGImage memory

- `UIImage(named:)` — cached by system, decoded lazily on first draw
- `UIImage(data:)` / `UIImage(contentsOfFile:)` — not cached, decoded lazily
- `CGImage` — when held, keeps the full decoded pixel buffer in memory

Explicitly release large images by setting the variable to `nil` when done — ARC won't release until all strong references drop.

---

## Autorelease Pool: For Tight Loops

Objective-C objects (and some Swift types bridged to ObjC) go into the autorelease pool and are freed only at the end of the run loop iteration. In tight loops, this accumulates:

```swift
// Without autoreleasepool: temporary UIImages accumulate until loop ends
for path in imagePaths {
    autoreleasepool {
        let image = UIImage(contentsOfFile: path)  // Obj-C autorelease
        process(image)
        // image released HERE, not at loop end
    }
}
```

Use `autoreleasepool {}` in loops that create many ObjC objects (images, strings, Data from file).

---

## Instruments: Memory Profiling

### Allocations instrument
- Shows all heap allocations over time
- Look for: growing allocation count without corresponding release ("memory leak sawtooth")
- "Mark Generation" to compare heap state before and after an action — everything not freed is a candidate for a leak

### Leaks instrument
- Automatically detects objects with no path to a root → classic retain cycles
- Less useful for weak-reference cycles (where both sides are alive but logically disconnected)

### Memory Graph Debugger (Xcode)
- **Best tool for retain cycles**: pause app → Debug → Memory Graph (`⌘⌥ F9`)
- Click on any object to see all references TO it (incoming edges) and FROM it
- Look for cycles: A → B → A in the reference graph
- Export `.memgraph` file and open in command line: `leaks --referenceTree App.memgraph`

### VM Tracker
- Shows virtual memory regions: heap, stack, mapped files, GPU allocations
- "Dirty" memory = actually used RAM; "Resident" = in RAM but may be paged; "Swapped" = absent on iOS (no swap)
- Target: minimize dirty memory footprint

---

## Structural Patterns to Reduce Allocations

### Value types over classes for data models

```swift
// Every Model instance is a separate heap allocation + ARC
class Model { var id: Int; var name: String }

// Struct: stored inline wherever Model lives — no heap allocation
struct Model { var id: Int; var name: String }
```

For data-only types (no shared identity needed), struct eliminates the heap allocation and ARC overhead.

### Lazy initialization for rarely-used properties

```swift
class ViewController: UIViewController {
    lazy var expensiveComponent: ExpensiveView = {
        ExpensiveView(frame: .zero)
    }()  // allocated only on first access
}
```

### Preallocate with known capacity

```swift
var items = [Item]()
items.reserveCapacity(expectedCount)  // one allocation instead of O(log n) reallocations

var dict = [String: Item](minimumCapacity: expectedCount)
```

---

## Code Review Checklist

- [ ] Does every closure that captures `self` use `[weak self]` or `[unowned self]` where a cycle could form?
- [ ] Are delegate properties `weak var`?
- [ ] Is `Timer` or `CADisplayLink` invalidated in `deinit` / `viewDidDisappear`?
- [ ] Are NotificationCenter block-based observers stored and removed in `deinit`?
- [ ] Is `NSCache` used instead of `Dictionary` for memory caches?
- [ ] Does `NSCache` have `totalCostLimit` set based on actual byte cost of cached objects?
- [ ] Does `didReceiveMemoryWarning` free caches without removing data needed for current UI?
- [ ] Are large images downsampled before display (`CGImageSourceCreateThumbnailAtIndex`)?
- [ ] Do tight loops creating ObjC objects use `autoreleasepool`?
- [ ] Are data models that don't need identity represented as structs (not classes)?
- [ ] Are arrays/dicts preallocated with `reserveCapacity` when the count is known upfront?
