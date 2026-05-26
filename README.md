# iOS AI Performance Skills

Claude Code skills для iOS-разработки, сфокусированные на производительности. Каждый скилл — это экспертный контекст, который Claude использует при анализе и ревью кода.

## Установка

Установить весь набор одной командой:

```bash
npx skills add levabond/iOS-ai-performance-skills --all
```

Или установить только нужные скиллы:

```bash
npx skills add levabond/iOS-ai-performance-skills --skill swiftui-performance --skill memory-management-performance
```

CLI спросит, в каких агентов (Claude Code, Codex, Cursor, Gemini, ...) добавить скиллы и куда установить — в текущий проект или глобально.

Если нет `npx`, поставьте Node (`brew install node`); если нет `brew`, сначала [поставьте Homebrew](https://brew.sh).

### Альтернатива

**Claude Code** (ставит весь набор сразу):

```bash
/plugin install levabond/iOS-ai-performance-skills
```

Или клонируйте репозиторий и разложите папки скиллов туда, где их ждёт ваш агент.

## Скиллы

### Swift

| Скилл | Триггер |
|---|---|
| `swift-collections-performance` | Кастомные коллекции, SortedSet, B-Tree, COW, binary search |
| `swift-internals-performance` | Stack/heap, ARC, dispatch (static/vtable/PWT/message), existentials |
| `swift-generics-internals` | Generics, protocol conformances, existentials vs generics, Requirement Machine |
| `swift-concurrency-advanced` | async/await, акторы, reentrancy, TaskGroup, AsyncStream, task-locals |

### UIKit / SwiftUI

| Скилл | Триггер |
|---|---|
| `swiftui-performance` | Body recomputation, @Observable, EquatableView, LazyVStack, Canvas |
| `tableview-performance` | Cell reuse, batch updates, diffable data source, dynamic height |
| `core-animation-performance` | Offscreen rendering, shadowPath, shouldRasterize, GPU/CPU bottlenecks |

### Системный уровень

| Скилл | Триггер |
|---|---|
| `networking-performance` | URLSession, HTTP/2, NSURLCache, retry, background transfers |
| `concurrency-dispatch-performance` | GCD, OperationQueue, thread explosion, deadlocks, @MainActor |
| `memory-management-performance` | Retain cycles, weak/unowned, NSCache, image memory, autoreleasepool |
| `app-launch-performance` | Pre-main, +load, static linking, didFinishLaunching, MetricKit |
| `ios-performance-profiling` | Instruments, os_signpost, XCTest metrics, bottleneck workflow |

## Источники

- *Optimizing Collections in Swift* — Karoly Lorentey
- *iOS Core Animation: Advanced Techniques* — Nick Lockwood
- *Compiling Swift Generics* — Slava Pestov (Apple)
- *Swift Internals* — Aaqib Hussain (Kodeco)
- *Table View Programming Guide for iOS* — Apple
