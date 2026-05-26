---
name: core-animation-performance
description: Core Animation and rendering performance expert for iOS. Covers GPU vs CPU bottlenecks, offscreen rendering, layer compositing, image I/O, drawing optimizations, and Instruments profiling. Trigger when reviewing UIView/CALayer rendering code, animation performance, scroll jank, offscreen passes, shadow/mask/blur usage, or image loading patterns.
---

# Core Animation Performance Expert

Based on "iOS Core Animation: Advanced Techniques" by Nick Lockwood. Use this skill when diagnosing rendering bottlenecks, reviewing layer hierarchy, or optimizing animation smoothness.

## Mental Model: The Rendering Pipeline

Every frame goes through three stages — identify which is your bottleneck before optimizing:

1. **CPU stage** — layout, display (Core Graphics drawing), preparation (image decoding, geometry)
2. **GPU stage** — compositing (blending layers), rasterization (converting vectors/text to pixels)
3. **I/O stage** — loading image data from disk

Target: 60 fps = 16.67 ms per frame budget, shared across CPU + GPU.

**Rule**: Measure with Instruments on a real device before changing anything. The simulator does not use the GPU the same way.

---

## GPU-Bound Bottlenecks

### Offscreen Rendering — the biggest GPU killer

Offscreen rendering forces the GPU to switch render targets mid-frame, which is expensive. It is triggered by:

| Property / Operation | Triggers Offscreen? |
|---|---|
| `layer.masksToBounds = true` + `cornerRadius` | Yes |
| `layer.mask` (custom mask layer) | Yes |
| `layer.shadowOpacity > 0` without `shadowPath` | Yes |
| `layer.shouldRasterize = true` | Yes (but caches result) |
| `layer.allowsGroupOpacity` + alpha on group | Yes |
| `CAShapeLayer` with complex paths | Sometimes |
| `drawsAsynchronously = true` on CATiledLayer | No (async) |

**Detect with Instruments**: Core Animation instrument → "Color Offscreen-Rendered Yellow" debug option highlights offscreen-rendered layers in yellow.

### Fix: shadowPath eliminates offscreen pass for shadows

```swift
// BAD — GPU must calculate shadow shape every frame
layer.shadowOpacity = 0.5
layer.shadowRadius = 4

// GOOD — provide explicit path, no offscreen pass needed
layer.shadowOpacity = 0.5
layer.shadowRadius = 4
layer.shadowPath = UIBezierPath(roundedRect: layer.bounds,
                                cornerRadius: layer.cornerRadius).cgPath
```

Update `shadowPath` in `layoutSubviews` when bounds change.

### Fix: shouldRasterize for static complex layers

```swift
// Good when layer content doesn't change between frames
layer.shouldRasterize = true
layer.rasterizationScale = UIScreen.main.scale  // always set this
```

Rasterization caches the offscreen result. Use only when:
- Content is static between frames
- Layer will be composited many times (e.g., cells in a list)
- Set `rasterizationScale` to match screen — otherwise blurry on Retina

Do NOT use on layers that change content frequently — cache invalidation cost exceeds benefit.

### Fix: Opaque layers eliminate blending

Blending (non-opaque layers) requires the GPU to read both the layer and what's behind it.

```swift
// Mark layers opaque when background is solid
layer.isOpaque = true
layer.backgroundColor = UIColor.white.cgColor

// For UIView
view.isOpaque = true
view.backgroundColor = .white
```

**Detect**: Core Animation instrument → "Color Blended Layers" — red = blending (slow), green = opaque (fast).

### Reduce layer count

Fewer layers = fewer compositing operations. Flatten layers where possible using `renderInContext` to produce a single image instead of a complex hierarchy.

---

## CPU-Bound Bottlenecks

### drawRect / Core Graphics drawing

`drawRect:` / `draw(_:)` forces Core Graphics software rendering on the CPU. The result is cached in a backing store (extra memory allocation proportional to view size × scale²).

- Never implement an empty `draw(_:)` — it forces a backing store allocation even if unused
- `setNeedsDisplay()` triggers full redraw of the entire rect (use `setNeedsDisplay(_:)` with a dirty rect)
- Prefer layer properties (backgroundColor, borderColor, cornerRadius) over drawing equivalent effects in Core Graphics

### Asynchronous drawing

For complex custom drawing, offload to a background thread:

```swift
// Using CATiledLayer for large scrolling content
// CATiledLayer draws tiles on background threads automatically

// For UICollectionView / UITableView cells: draw into image on bg queue, then set as layer.contents
DispatchQueue.global(qos: .userInitiated).async {
    let image = self.renderCellImage()
    DispatchQueue.main.async {
        self.layer.contents = image.cgImage
    }
}
```

Or use `layer.drawsAsynchronously = true` — signals that drawing commands can be deferred, but does not guarantee background execution.

### Layout performance

- `layoutSubviews` is called on every bounds change — keep it fast
- Avoid trigonometric functions and expensive calculations in layout
- Cache computed values; don't recalculate the same geometry on every layout pass

---

## I/O-Bound Bottlenecks

### Image loading and decompression

Images loaded with `UIImage(named:)` are:
- Cached in memory by the system
- Decompressed lazily on first draw → causes jank on first appearance

`UIImage(contentsOfFile:)`:
- Not cached
- Still lazy decompression

**Force decompression on a background thread** before display:

```swift
func decompressedImage(from url: URL) -> UIImage? {
    guard let image = UIImage(contentsOfFile: url.path) else { return nil }
    UIGraphicsBeginImageContextWithOptions(image.size, true, 0)
    image.draw(at: .zero)
    let decompressed = UIGraphicsGetImageFromCurrentImageContext()
    UIGraphicsEndImageContext()
    return decompressed
}

// Call on background queue, hand result to main queue
```

### Image format matters

- **PNG**: lossless, no decompression cost beyond initial load, good for UI assets with transparency
- **JPEG**: lossy compression, smaller files, higher decode cost
- **PVRTC**: GPU-native format for game-like content, extremely fast GPU upload, no CPU decompression

### CATiledLayer for large images

Load only what's visible. CATiledLayer draws tiles on background threads:
- Set `levelsOfDetail` and `levelsOfDetailBias` for zoom-level support
- Use for content larger than screen size (maps, PDFs, large photos)

### NSCache for custom image caching

```swift
let imageCache = NSCache<NSString, UIImage>()
imageCache.countLimit = 100       // max items
imageCache.totalCostLimit = 50 * 1024 * 1024  // 50 MB

func cachedImage(for key: String) -> UIImage? {
    return imageCache.object(forKey: key as NSString)
}
func cacheImage(_ image: UIImage, for key: String) {
    imageCache.setObject(image, forKey: key as NSString,
                         cost: Int(image.size.width * image.size.height * 4))
}
```

NSCache evicts automatically under memory pressure. Prefer it over Dictionary for image caches.

---

## Layer Properties — Performance Impact Reference

| Property | Performance Cost | Notes |
|---|---|---|
| `backgroundColor` | Negligible | GPU-composited, very cheap |
| `cornerRadius` alone | Low | Only clips background color |
| `cornerRadius` + `masksToBounds` | High | Offscreen pass for all sublayers |
| `shadowOpacity` (no shadowPath) | High | Dynamic shadow shape calculation |
| `shadowOpacity` + `shadowPath` | Low | Pre-computed shape |
| `mask` (layer mask) | High | Always offscreen |
| `shouldRasterize = true` | Low (cached) | Expensive on first render, then cheap |
| `opacity < 1.0` on group | Medium | Blending required |
| `isOpaque = true` | None (saves work) | Eliminates blending pass |
| CAShapeLayer | Medium | Vector rasterization per frame if animated |
| CATextLayer | Medium | CPU renders text to bitmap |

---

## Instruments Workflow

1. **Profile on device** — never the simulator
2. **Core Animation instrument** — shows FPS, frame time
3. Enable debug options:
   - "Color Blended Layers" → red overlays = blending (GPU cost)
   - "Color Offscreen-Rendered Yellow" → yellow = offscreen pass
   - "Color Hits Green and Misses Red" → with `shouldRasterize`, shows cache hit rate
4. **Time Profiler** — find CPU-bound drawing (look for `CA::Render`, `drawRect`, `-[CALayer display]`)

---

## CADisplayLink for Timer-Based Animation

```swift
var displayLink: CADisplayLink?

func startAnimation() {
    displayLink = CADisplayLink(target: self, selector: #selector(tick))
    displayLink?.add(to: .main, forMode: .common)  // .common keeps running during scroll
}

@objc func tick(link: CADisplayLink) {
    let elapsed = link.timestamp - link.targetTimestamp + link.duration
    // update based on elapsed, not a fixed step
}

func stopAnimation() {
    displayLink?.invalidate()
    displayLink = nil
}
```

Use `.common` run loop mode — `.default` pauses during scroll and touch tracking.

Compute animation progress from `timestamp`, not frame count — frame rate can drop below 60 fps.

---

## Code Review Checklist

- [ ] Does any layer use `shadowOpacity > 0` without `shadowPath`? → Fix: set explicit `shadowPath`
- [ ] Does any layer combine `cornerRadius` + `masksToBounds`? → Consider alternatives: pre-rendered image, CAShapeLayer clip
- [ ] Is `shouldRasterize = true` set with `rasterizationScale`? → Must match `UIScreen.main.scale`
- [ ] Are images decompressed before being handed to the main thread?
- [ ] Is `draw(_:)` implemented but effectively empty or trivial? → Remove it (saves backing store allocation)
- [ ] Are large image assets cached with NSCache rather than loaded every time?
- [ ] Is CADisplayLink using `.common` run loop mode?
- [ ] Are opaque views marked `isOpaque = true` with a solid `backgroundColor`?
- [ ] Does any cell or frequently-reused view do synchronous image loading in `cellForRow`?
