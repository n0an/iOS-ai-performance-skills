---
description: iOS app launch performance expert. Covers cold vs warm vs hot launch, dyld loading, +load vs +initialize, static vs dynamic linking at launch, pre-main work, main thread work deferral, MetricKit/XCTest metrics, and Launch Time Instruments. Trigger when reviewing AppDelegate, app initialization code, framework linking strategy, +load method usage, or diagnosing slow app launch (>400ms pre-main or main thread work in didFinishLaunching).
---

# iOS App Launch Performance Expert

## Launch Types

| Type | Cold start | Warm start | Resume (hot) |
|---|---|---|---|
| Definition | App not in memory at all | App killed but binary/data cached | App suspended in background |
| dyld work | Full | Reduced | None |
| +load / initializers | Run | Run | None |
| `didFinishLaunching` | Runs | Runs | None — `applicationWillEnterForeground` |
| Target | <400ms total | <200ms | Near-instant |

**Measure on a cold start** — warm starts are misleading. Reboot the device (or use `xcrun simctl terminate` + `xcrun simctl launch`) to force cold starts.

---

## Pre-Main Phase: dyld

Everything before `main()` runs in the pre-main phase:
1. dyld loads the app binary and all dynamic frameworks
2. ObjC runtime registration (classes, categories, protocols)
3. C++ static initializers
4. Swift type metadata initialization
5. `+load` methods (ObjC)

### Reduce pre-main time

**Static linking over dynamic frameworks:**
Each dynamic framework requires dyld to: find the `.dylib`, load it into memory, fix up relocations, call its initializers. A typical dynamic framework adds 1–5ms at cold launch.

```
# Build settings
MACH_O_TYPE = staticlib   // for internal modules you control
```

Merge small internal frameworks. 10 dynamic frameworks → 1 static library can save 30–50ms cold launch.

**Eliminate `+load` methods:**
```objc
// BAD: runs unconditionally at launch, before main()
@implementation MyClass
+ (void)load {
    // Anything here runs at startup — even if MyClass is never used
    [self swizzleMethod];  // expensive ObjC runtime work
}

// GOOD: runs on first use of the class
+ (void)initialize {
    if (self == [MyClass class]) {
        [self setupOnce];
    }
}
```

`+load` runs for every class and category at startup — every `+load` in every framework you link, even third-party ones. Each adds latency. Use `+initialize` or `dispatch_once`.

In Swift, `static let` with lazy initialization is the equivalent of safe `dispatch_once`:
```swift
class Registry {
    static let shared = Registry()  // initialized on first access, thread-safe
}
```

**Reduce ObjC category count:**
Each ObjC category registers itself at load time. Many categories on widely-used classes (NSString, UIView) add meaningful pre-main time.

---

## Application Startup: didFinishLaunching

`didFinishLaunching` runs on the main thread. Anything slow here delays the first frame.

Apple's time-to-first-frame budget: **400ms** from app launch to first rendered frame. If you exceed this, the OS may terminate the app for being slow to launch.

### Defer everything not needed for first frame

```swift
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    // DO synchronously — needed for first frame
    setupRootViewController()
    setupCoreData()  // only if first screen needs it

    // DEFER — not needed at launch
    DispatchQueue.main.async {
        // Run after first frame renders
        self.setupAnalytics()
        self.checkForUpdates()
        self.prefetchSecondaryData()
    }

    // DEFER further — background, no urgency
    DispatchQueue.global(qos: .utility).asyncAfter(deadline: .now() + 2) {
        self.cleanupOldCaches()
        self.syncOfflineData()
    }

    return true
}
```

The `DispatchQueue.main.async` trick works because the block is queued after the current run loop cycle — after the first frame is presented.

### Priority of initialization tasks

1. **Synchronous in didFinishLaunching**: Window, root view controller, authentication state
2. **Main async (after first frame)**: Analytics, push notification registration, A/B config fetch
3. **Background utility**: Cache cleanup, secondary data sync, prefetching
4. **Background (lazy)**: Everything else on first use

---

## SceneDelegate and Multiple Windows

With SceneDelegate (iOS 13+), `scene(_:willConnectTo:options:)` creates the UI. Keep it as lightweight as `didFinishLaunching`:

```swift
func scene(_ scene: UIScene, willConnectTo session: UISceneSession,
           options: UIScene.ConnectionOptions) {
    guard let windowScene = scene as? UIWindowScene else { return }
    window = UIWindow(windowScene: windowScene)
    window?.rootViewController = makeInitialViewController()
    window?.makeKeyAndVisible()
    // Don't block here — defer all secondary work
}
```

---

## MetricKit: Production Launch Metrics

MetricKit (iOS 13+) collects real-world performance data from your users:

```swift
class MetricsObserver: NSObject, MXMetricManagerSubscriber {
    override init() {
        super.init()
        MXMetricManager.shared.add(self)
    }

    func didReceive(_ payloads: [MXMetricPayload]) {
        for payload in payloads {
            if let launchMetrics = payload.applicationLaunchMetrics {
                // Histogram of launch times
                let median = launchMetrics.histogrammedTimeToFirstDrawKey?.histogram
                log("Launch time median: \(median)")
            }
            if let hangMetrics = payload.applicationHangMetrics {
                log("Hang rate: \(hangMetrics.hangDuration)")
            }
        }
    }
}
```

MetricKit data aggregates over 24h periods and represents real user devices — more representative than Instruments on a development device.

### XCTest launch metrics

```swift
func testLaunchPerformance() {
    measure(metrics: [XCTApplicationLaunchMetric()]) {
        XCUIApplication().launch()
    }
}
```

Runs 5 launch iterations, reports mean and standard deviation. Add to CI to catch regressions.

---

## Instruments: Launch Template

1. Product → Profile → Launch template
2. Start recording, cold-launch the app
3. Key sections in the timeline:
   - **Pre-main**: dyld loading, `+load`, initializers — target < 300ms
   - **UIKit initialization**: window setup
   - **First frame**: `viewDidLoad`, `layoutSubviews`, first `drawRect`/`draw(_:)`
4. Drill into the Call Tree to find the heaviest pre-main call
5. Check "Hidden System Libraries" unchecked — shows framework `+load` contributors

---

## Startup Crashers: Launch Arguments for Debugging

```
# Add to scheme Run arguments to catch over-releasing in pre-main
DYLD_PRINT_LIBRARIES=1          # logs every framework loaded
OBJC_PRINT_LOAD_METHODS=1       # logs every +load method called (crash candidates)
```

---

## Third-Party SDKs at Launch

Every SDK initialized in `didFinishLaunching` contributes to launch time. Common offenders:

- Analytics SDKs: delay initialization until after first frame
- Crash reporters: usually must be first — but check if they offer deferred init
- Ad SDKs: often safe to defer 2–5 seconds

```swift
// Defer analytics — 2 second delay acceptable
DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
    Analytics.initialize(key: "...")
}

// Crash reporter — first, synchronous
CrashReporter.start()

// Ad SDK — after any UI is visible
DispatchQueue.main.async {
    AdSDK.initialize()
}
```

---

## SwiftUI App Launch

```swift
@main
struct MyApp: App {
    // Runs on main thread — keep minimal
    @StateObject private var appState = AppState()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(appState)
                .task {
                    // Runs async after first render — safe for deferred work
                    await appState.loadInitialData()
                }
        }
    }
}
```

Use `.task { }` modifier on the root view for work that needs to happen after first render.

---

## Code Review Checklist

- [ ] Is `didFinishLaunching` free of synchronous network calls or heavy I/O?
- [ ] Are third-party SDK initializations deferred via `DispatchQueue.main.async` or `asyncAfter`?
- [ ] Are `+load` methods absent (or replaced with `+initialize` / `dispatch_once` / `static let`)?
- [ ] Are internal modules linked statically rather than as dynamic frameworks?
- [ ] Is the app's framework count ≤ 6 dynamic frameworks (excluding Apple's)?
- [ ] Are XCTest launch metrics added to CI to catch regressions?
- [ ] Is MetricKit subscribed to in production to monitor real-world launch times?
- [ ] Is the root view controller creation the first and only thing done synchronously?
- [ ] Are heavy data stores (Core Data, Realm) initialized lazily or after first frame?
- [ ] In SwiftUI apps, is `.task { }` used for post-render work instead of `init()`?
