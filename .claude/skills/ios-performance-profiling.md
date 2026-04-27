---
description: iOS performance profiling and instrumentation expert. Covers the full Instruments toolset (Time Profiler, Allocations, Leaks, Core Animation, Network, Hangs, MetricKit), os_signpost for custom timing, XCTest performance metrics, on-device vs simulator differences, and a systematic workflow for diagnosing unknown bottlenecks. Trigger when profiling any iOS performance issue, setting up benchmarks, measuring regressions, or choosing which Instruments tool to use.
---

# iOS Performance Profiling Expert

## The Golden Rule: Measure First

Never guess. Never optimize without data. The bottleneck is almost always somewhere unexpected.

**Profiling order:**
1. Reproduce the issue consistently
2. Identify the category (CPU, GPU, memory, I/O, network)
3. Use the right tool to pinpoint the cause
4. Fix one thing at a time
5. Re-measure to verify improvement

**Always profile on a real device with a release build** (`-O` optimization). The simulator uses your Mac's CPU and GPU — results are not representative of device behavior.

---

## Instruments Tool Selection Guide

| Problem | Primary Tool | Secondary |
|---|---|---|
| Slow/janky UI | Time Profiler | Core Animation |
| Main thread hang | Hangs | Time Profiler |
| Dropped frames | Core Animation | GPU Driver |
| Memory leak | Leaks | Memory Graph |
| Growing memory | Allocations | VM Tracker |
| Slow launch | App Launch | Time Profiler |
| Slow network | Network | HTTP Traffic |
| Battery drain | Energy Log | CPU / Network |
| Threading bugs | Thread Sanitizer | Thread Perf Checker |
| Concurrency issues | Swift Concurrency | Time Profiler |

---

## Time Profiler: CPU Bottlenecks

### Setup
Product → Profile → Time Profiler. Use **Release** scheme.

### Reading the output
- Timeline shows CPU usage per thread
- Call Tree shows where time was spent (sampled every 1ms)
- **Always check "Hide System Libraries"** to see your code
- **"Separate by Thread"** — identify which thread is doing what
- **"Invert Call Tree"** — shows leaf functions (actual work, not callers)

### Typical findings
```
Main Thread → UITableView → cellForRowAt → ImageDecoder.decode
              → NSJSONSerialization.JSONObjectWithData  ← decode on main
              → NSAttributedString.init(html:)          ← slow HTML parsing on main
```

Any of these > 8ms per frame = jank.

### Mark sections with os_signpost (see below) to zoom into specific operations.

---

## Core Animation Instrument: Rendering Performance

Attach: Instruments → Core Animation template.

**FPS counter** — below 60 fps (or 120 fps on ProMotion) = dropped frames.

### Debug display options (enable in Simulator or device via Instruments)

| Option | Shows | Action needed when red |
|---|---|---|
| Color Blended Layers | Non-opaque layers (GPU blending cost) | Mark layers `isOpaque = true`, add `backgroundColor` |
| Color Offscreen-Rendered | Layers needing offscreen passes | Remove `mask`, add `shadowPath`, reconsider `shouldRasterize` |
| Color Hits Green / Misses Red | `shouldRasterize` cache hits | Red = content changing too often, remove `shouldRasterize` |
| Color Copied Images | Images being copied to GPU (format mismatch) | Use `kCVPixelFormatType_32BGRA` for CVPixelBuffer |
| Color Immediately | Draws each frame immediately (no buffering) | Diagnosing timing issues |

---

## Allocations Instrument: Memory Growth

Attach: Instruments → Allocations.

### Finding leaks with generations
1. Navigate to screen A (baseline)
2. Press "Mark Generation" in Instruments
3. Navigate to screen B and back to A
4. Press "Mark Generation" again
5. Look at generation 2 — everything still allocated that wasn't in generation 1 is a candidate leak

### Reading the heap
- **#Persistent**: objects still alive at this moment
- **#Transient**: objects created and released (healthy — high count ok if #Persistent is stable)
- **Bytes**: total memory, includes stack + heap

Look for **growing #Persistent** over time with repeated actions = memory not being freed.

---

## Leaks Instrument

Automatically detects objects with zero incoming strong references from any root → classic retain cycle.

**Limitation**: doesn't detect "logical leaks" — objects still referenced but should have been released (e.g., view controller held in a static array).

For those, use the **Memory Graph Debugger**.

---

## Memory Graph Debugger (Xcode)

The best tool for retain cycles:

1. Run app on device or simulator
2. Navigate to the screen with the suspected leak
3. Navigate away (the leaked object should have been released)
4. In Xcode: Debug → Memory Graph (`⌘⌥ F9`)
5. In the left panel, search for the class you expect to be gone
6. If it's still there, click it — the graph shows all references keeping it alive
7. Follow the reference chain to find the cycle

```bash
# Export and analyze from command line
leaks --referenceTree MyApp.memgraph 2>/dev/null | head -100
```

---

## os_signpost: Custom Timing Markers

Add semantic markers to your code so Instruments can show your operations on the timeline:

```swift
import os.signpost

let log = OSLog(subsystem: "com.myapp", category: "DataLoading")

// Point signpost (instant event)
os_signpost(.event, log: log, name: "Cache Hit")

// Interval signpost (duration)
func loadData(id: String) async throws -> Data {
    let signpostID = OSSignpostID(log: log)
    os_signpost(.begin, log: log, name: "LoadData", signpostID: signpostID, "%{public}s", id)
    defer {
        os_signpost(.end, log: log, name: "LoadData", signpostID: signpostID)
    }
    return try await fetch(id: id)
}
```

Signpost intervals appear as colored lanes in Instruments → Points of Interest instrument. You can see exactly how long each `loadData` call took and align it with CPU, GPU, and network timelines.

---

## XCTest Performance Metrics

Add to your test suite to catch regressions in CI:

```swift
func testCellConfiguration() {
    let cell = MyTableViewCell()
    let item = testItem()
    measure {
        cell.configure(with: item)  // measured 10 times, reports mean + stddev
    }
}

// With specific metrics
func testImageDecoding() {
    let data = imageData()
    measure(metrics: [XCTClockMetric(), XCTMemoryMetric(), XCTCPUMetric()]) {
        _ = UIImage(data: data)
    }
}

// Launch time
func testLaunchTime() {
    measure(metrics: [XCTApplicationLaunchMetric()]) {
        XCUIApplication().launch()
    }
}
```

Set a `performanceBaseline` after the first run — subsequent runs fail CI if they regress beyond the threshold.

---

## Hangs Instrument (Xcode 14+)

Detects main thread hangs > 250ms. Available in Instruments → Hangs.

Shows:
- Duration of each hang
- Full main thread call stack at the moment of the hang
- Whether the hang was in your code or system code

Also available in MetricKit (production data):
```swift
func didReceive(_ payloads: [MXMetricPayload]) {
    payloads.forEach { payload in
        payload.applicationHangMetrics?.histogrammedHangDuration.bucketEnumerator.forEach { bucket in
            guard let bucket = bucket as? MXHistogramBucket<UnitDuration> else { return }
            print("Hang \(bucket.bucketStart) – \(bucket.bucketEnd): \(bucket.bucketCount) times")
        }
    }
}
```

---

## Network Instrument

Instruments → Network template shows:
- All TCP connections
- DNS lookup duration (repeated = session not reused)
- TLS handshake duration (repeated = connection not reused or server not supporting session resumption)
- Time to First Byte (TTFB) = server-side latency
- Data transfer time = bandwidth

**HTTP Traffic** instrument (Xcode 14+): shows actual request/response headers and bodies — useful for verifying caching headers are being sent and honored.

---

## Energy Log

Instruments → Energy Log shows:
- CPU wakeups (each wakeup costs ~30ms CPU time equivalent in energy)
- Network radio state (WiFi/cellular warmup time)
- GPU usage
- Location, Bluetooth, audio usage

**Common energy drain patterns:**
- Timer firing too frequently (use `CADisplayLink` only when animating, not always-on)
- Background location updates at full accuracy
- Network requests with no batching (many small requests = many radio warmups)
- Retained background tasks that keep CPU from sleeping

---

## Swift Concurrency Instrument (Xcode 13.2+)

Instruments → Swift Concurrency template shows:
- Task creation and cancellation
- Actor isolation (which actor is running)
- Task states: running, suspended, enqueued
- Cooperative thread pool utilization

Use to identify:
- Tasks spending long time enqueued (thread pool saturated)
- Actors with long queues (actor contention)
- Tasks never being cancelled when they should be

---

## Systematic Bottleneck Diagnosis Workflow

```
1. User complaint: "The app feels slow when scrolling the feed"

2. Reproduce: scroll the feed on a real device in Instruments

3. Category check (Time Profiler):
   - Is main thread > 80% during the slow period? → CPU-bound
   - Is main thread idle but GPU spike? → GPU-bound (use Core Animation)
   - Network calls in scroll path? → I/O-bound

4a. If CPU-bound (Time Profiler):
    - Find top frames in the call tree under main thread
    - Is it cellForRowAt? viewDidLoad? layoutSubviews?
    - Move work to background, cache results, reduce work per frame

4b. If GPU-bound (Core Animation):
    - Enable debug color options
    - Find red/yellow layers
    - Fix: shadowPath, remove offscreen passes, mark opaque

4c. If I/O-bound:
    - Move disk/network reads off main thread
    - Add caching layer
    - Prefetch next page before it's needed

5. Fix one root cause at a time, measure again
```

---

## On-Device vs Simulator Differences

| Aspect | Simulator | Real device |
|---|---|---|
| CPU | Mac CPU (much faster) | A-series chip |
| GPU | Not used (Metal emulated) | A-series GPU |
| Memory pressure | Not representative | Real pressure at 2–6 GB |
| Thermal throttling | None | Occurs after sustained load |
| Network | Mac network | LTE/WiFi with real latency |
| dyld | Not representative | Real framework loading |

**Never report launch time, animation performance, or memory footprint from simulator results.**

---

## Code Review Checklist

- [ ] Are `os_signpost` markers added around non-trivial operations for Instruments visibility?
- [ ] Are XCTest performance metrics added for critical paths (cell config, image decode, data parse)?
- [ ] Is profiling done on a real device with a Release build?
- [ ] Is MetricKit subscribed to in production for real-world hang and launch data?
- [ ] Are repeated runs (not just one) used to establish a stable baseline?
- [ ] After fixing a bottleneck, is it re-profiled to confirm improvement?
- [ ] Are Instruments sessions saved (`.trace` files) for regression comparison?
