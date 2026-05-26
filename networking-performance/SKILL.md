---
name: networking-performance
description: iOS networking performance expert. Covers URLSession configuration, HTTP/2, connection reuse, caching (NSURLCache, ETags, Cache-Control), background transfers, request batching, Combine/async-await integration, timeout strategy, retry logic, certificate pinning overhead, and network Instruments profiling. Trigger when reviewing URLSession code, API client architecture, image downloading pipelines, offline support, or diagnosing slow/failing network requests.
---

# iOS Networking Performance Expert

## URLSession Configuration — Choose the Right One

Three session types; using the wrong one wastes resources or breaks correctness:

| Configuration | Survives app termination | Use case |
|---|---|---|
| `.default` | No | Standard foreground requests |
| `.ephemeral` | No | Private/incognito, no disk cache or cookie persistence |
| `.background(withIdentifier:)` | **Yes** | Large uploads/downloads that must complete even if app dies |

```swift
// One shared session per configuration — do NOT create a session per request
class APIClient {
    static let shared = APIClient()

    private let session: URLSession = {
        let config = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = 30    // per-request timeout
        config.timeoutIntervalForResource = 300  // total resource timeout
        config.httpMaximumConnectionsPerHost = 6 // HTTP/1.1 limit; HTTP/2 ignores this
        config.requestCachePolicy = .useProtocolCachePolicy
        return URLSession(configuration: config)
    }()
}
```

**Never** create `URLSession(configuration: .default)` inside a function — each call creates a new session with its own connection pool, DNS cache, and TLS session cache. Connection warm-up cost repeats on every request.

---

## HTTP/2 and Connection Reuse

URLSession automatically uses HTTP/2 when the server supports it. HTTP/2 multiplexes multiple requests over a single TCP+TLS connection — there is no effective per-host connection limit.

Implications:
- Under HTTP/2, setting `httpMaximumConnectionsPerHost` to >1 doesn't help; the multiplexer handles concurrency
- Fewer connections = lower TLS handshake overhead on first use
- Server push (rare in mobile) can pre-deliver resources

**Diagnose connection reuse** in Instruments → Network template → look for new TCP connections per request sequence. Repeated connections from a single session indicate the server is closing connections prematurely or TLS session resumption is failing.

---

## Caching

### NSURLCache

URLSession uses `NSURLCache` for HTTP caching. Cache is shared by default.

```swift
// Set a larger cache at app startup (AppDelegate/App init)
URLCache.shared = URLCache(
    memoryCapacity: 20 * 1024 * 1024,   // 20 MB in-memory
    diskCapacity: 100 * 1024 * 1024,    // 100 MB on disk
    directory: nil                       // default location
)
```

Cache works automatically when the server sends proper headers:
- `Cache-Control: max-age=3600` — cache for 1 hour
- `ETag: "abc123"` — client sends `If-None-Match: "abc123"` on revalidation; server replies 304 Not Modified (no body)
- `Last-Modified` — similar, uses `If-Modified-Since`

### Cache policies

| Policy | Behavior | Use when |
|---|---|---|
| `.useProtocolCachePolicy` | Respects server headers | Default; correct for most cases |
| `.reloadIgnoringLocalCacheData` | Always network | Forced refresh |
| `.returnCacheDataElseLoad` | Cache first, network if miss | Offline-tolerant read |
| `.returnCacheDataDontLoad` | Cache only, fail if miss | Strict offline mode |

```swift
var request = URLRequest(url: url)
request.cachePolicy = .returnCacheDataElseLoad  // great for images/static assets
```

### Manual cache for non-HTTP sources

NSURLCache only caches GET requests with HTTP/HTTPS. For anything else (POST responses, custom protocols), build your own:

```swift
actor ResponseCache {
    private var store: [String: (data: Data, expiry: Date)] = [:]

    func get(key: String) -> Data? {
        guard let entry = store[key], entry.expiry > Date() else {
            store.removeValue(forKey: key)
            return nil
        }
        return entry.data
    }

    func set(_ data: Data, for key: String, ttl: TimeInterval) {
        store[key] = (data, Date().addingTimeInterval(ttl))
    }
}
```

---

## Request Design: Batching and Pagination

### Batch small requests

N separate requests = N round trips. Prefer one request that returns N results when latency dominates:

```swift
// BAD: waterfall of N requests, each waits for previous
for id in ids { await fetch(id: id) }

// GOOD: one batch request
let results = await fetchBatch(ids: ids)

// GOOD: concurrent if batching isn't possible and requests are independent
await withTaskGroup(of: Result.self) { group in
    for id in ids { group.addTask { await fetch(id: id) } }
    for await result in group { process(result) }
}
```

### Pagination

Always paginate large lists. Use cursor-based pagination over offset-based when possible — offset queries become slow as the dataset grows on the server side.

```swift
struct Page<T: Decodable>: Decodable {
    let items: [T]
    let nextCursor: String?
}

func fetchAll<T: Decodable>(endpoint: URL) async throws -> [T] {
    var all: [T] = []
    var cursor: String? = nil
    repeat {
        var url = endpoint
        if let c = cursor { url.append(queryItems: [URLQueryItem(name: "cursor", value: c)]) }
        let page: Page<T> = try await fetch(url)
        all += page.items
        cursor = page.nextCursor
    } while cursor != nil
    return all
}
```

---

## Async/Await Integration

```swift
// URLSession data task as async/throws (iOS 15+)
let (data, response) = try await URLSession.shared.data(for: request)
guard let http = response as? HTTPURLResponse, (200..<300).contains(http.statusCode) else {
    throw APIError.badStatus((response as? HTTPURLResponse)?.statusCode ?? -1)
}
let decoded = try JSONDecoder().decode(MyType.self, from: data)

// Cancel via task
let task = Task {
    let result = try await fetch(url)
    await MainActor.run { update(with: result) }
}
// task.cancel() — URLSession automatically cancels the underlying data task
```

### Cancellation in task groups

```swift
await withThrowingTaskGroup(of: MyType.self) { group in
    for url in urls { group.addTask { try await fetch(url) } }
    var results: [MyType] = []
    for try await result in group { results.append(result) }
    return results
}
// If any child task throws, the group cancels all remaining tasks
```

---

## Retry Logic

```swift
func fetchWithRetry<T>(
    _ request: URLRequest,
    maxAttempts: Int = 3,
    delay: TimeInterval = 1.0
) async throws -> T where T: Decodable {
    var lastError: Error?
    for attempt in 0..<maxAttempts {
        do {
            let (data, _) = try await URLSession.shared.data(for: request)
            return try JSONDecoder().decode(T.self, from: data)
        } catch let error as URLError where error.code == .networkConnectionLost
                                        || error.code == .timedOut {
            lastError = error
            if attempt < maxAttempts - 1 {
                try await Task.sleep(nanoseconds: UInt64(delay * Double(1 + attempt) * 1_000_000_000))
            }
        }
    }
    throw lastError!
}
```

Use **exponential backoff** (multiply delay by attempt number). Do NOT retry on 4xx client errors — they won't succeed on retry.

---

## Background Transfers

For uploads/downloads that must complete after the app is backgrounded or killed:

```swift
// Configure ONCE with a stable identifier
let config = URLSessionConfiguration.background(withIdentifier: "com.app.bg-transfer")
config.isDiscretionary = false     // start immediately, not at OS convenience
config.sessionSendsLaunchEvents = true  // wake app on completion

let session = URLSession(configuration: config, delegate: self, delegateQueue: nil)

// Start a download
let task = session.downloadTask(with: url)
task.resume()

// AppDelegate / Scene delegate must recreate session with same identifier
// when app is relaunched due to background transfer completion
func application(_ application: UIApplication,
                 handleEventsForBackgroundURLSession identifier: String,
                 completionHandler: @escaping () -> Void) {
    backgroundCompletionHandler = completionHandler
    _ = BackgroundSession.shared  // ensure session is created
}
```

---

## Response Parsing Performance

### JSON decoding on a background thread

JSONDecoder is CPU-intensive. Never decode on the main thread for large payloads:

```swift
func fetch<T: Decodable>(_ url: URL) async throws -> T {
    let (data, _) = try await URLSession.shared.data(from: url)
    // data(from:) already returns on a background thread in async context
    return try JSONDecoder().decode(T.self, from: data)
}
```

In async/await context, `URLSession.data(from:)` calls back on a cooperative thread pool thread — safe to decode there. Only switch to `MainActor` when updating UI.

### Reuse JSONDecoder

`JSONDecoder` is relatively cheap to create but if decoding in tight loops, reuse:

```swift
extension JSONDecoder {
    static let shared: JSONDecoder = {
        let d = JSONDecoder()
        d.dateDecodingStrategy = .iso8601
        return d
    }()
}
```

---

## Certificate Pinning — Overhead Aware

Certificate pinning prevents MITM attacks but adds latency on every new TLS connection (pin validation happens after the TLS handshake):

```swift
class PinningDelegate: NSObject, URLSessionDelegate {
    func urlSession(_ session: URLSession,
                    didReceive challenge: URLAuthenticationChallenge,
                    completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        // Validate public key hash, not the certificate itself (survives cert renewal)
        if isPinValid(serverTrust) {
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}
```

Pin the **public key** (SPKI hash), not the leaf certificate — certificates rotate, public keys don't.

---

## Instruments Profiling

1. **Network template** in Instruments — shows requests, latency, DNS, TCP connect, TLS, TTFB
2. **HTTP Traffic** instrument — shows request/response bodies, headers, timing breakdown
3. Look for:
   - Repeated DNS lookups (session not being reused)
   - TLS handshakes on every request (connection not being reused or server closing early)
   - Large time in "Waiting (TTFB)" — server-side latency, not client issue
   - Large response payloads being decoded synchronously on main thread (check Time Profiler)

---

## Code Review Checklist

- [ ] Is a single `URLSession` instance reused (not created per request)?
- [ ] Is `URLSessionConfiguration.background` used for large downloads/uploads?
- [ ] Does the app handle `handleEventsForBackgroundURLSession` for background sessions?
- [ ] Is the cache policy appropriate — no `reloadIgnoringLocalCacheData` on every request?
- [ ] Are independent requests made concurrently (task group) not sequentially?
- [ ] Is JSON decoding happening off the main thread?
- [ ] Does retry logic use exponential backoff and skip retrying 4xx responses?
- [ ] Is certificate pinning using SPKI hash (not leaf cert)?
- [ ] Are `URLSessionConfiguration` timeouts set (`timeoutIntervalForRequest`)?
- [ ] Is pagination implemented for list endpoints (not loading everything at once)?
