---
description: UITableView and UICollectionView performance expert for iOS. Covers cell reuse, dequeue patterns, data source best practices, dynamic height, batch updates, editing/reordering, and avoiding common scroll jank causes. Trigger when reviewing table/collection view code, cell configuration, data source implementations, insert/delete animations, or scroll performance issues.
---

# UITableView & UICollectionView Performance Expert

Based on Apple's "Table View Programming Guide for iOS." Use when reviewing table/collection view implementations for correctness and performance.

---

## Cell Reuse — The Foundation of Performance

UITableView recycles cells as they scroll off screen. **Never allocate a new cell if a reused one is available.**

### Correct dequeue pattern (modern)

```swift
// Register once in viewDidLoad
tableView.register(MyCell.self, forCellReuseIdentifier: "MyCell")

// Dequeue in cellForRowAt
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "MyCell", for: indexPath) as! MyCell
    let item = data[indexPath.row]
    cell.configure(with: item)  // always reconfigure — cell may carry state from previous use
    return cell
}
```

`dequeueReusableCell(withIdentifier:for:)` (with indexPath) always returns a cell — it creates one if the queue is empty. Never returns nil.

### Always reconfigure dequeued cells

A reused cell retains the configuration of the row it previously displayed. Reset every visual property you set — do not assume default state:

```swift
func configure(with item: Item) {
    titleLabel.text = item.title
    subtitleLabel.text = item.subtitle
    iconImageView.image = item.icon ?? UIImage(named: "placeholder")  // explicit fallback
    accessoryType = item.isSelected ? .checkmark : .none
    backgroundColor = .clear  // reset if you ever change it
}
```

### Multiple cell types — use distinct reuse identifiers

```swift
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    switch data[indexPath.row].type {
    case .header:
        let cell = tableView.dequeueReusableCell(withIdentifier: "HeaderCell", for: indexPath) as! HeaderCell
        cell.configure(with: data[indexPath.row])
        return cell
    case .item:
        let cell = tableView.dequeueReusableCell(withIdentifier: "ItemCell", for: indexPath) as! ItemCell
        cell.configure(with: data[indexPath.row])
        return cell
    }
}
```

---

## Data Source Design

### Keep data model flat and indexed

`cellForRowAt` is called 60 times per second during fast scrolling. The data lookup must be O(1):

```swift
// GOOD: direct array index
let item = items[indexPath.row]

// BAD: filtering, sorting, or searching inside cellForRowAt
let item = allItems.filter { $0.section == indexPath.section }[indexPath.row]
```

Pre-process and cache the data structure your table needs. Sort and filter before storing in `items`, not inside data source methods.

### Section index data — prepare upfront

```swift
var sections: [String] = []           // section titles
var sectionedItems: [[Item]] = []     // items grouped by section

func loadData() {
    let sorted = allItems.sorted { $0.name < $1.name }
    let grouped = Dictionary(grouping: sorted) { String($0.name.prefix(1)) }
    sections = grouped.keys.sorted()
    sectionedItems = sections.map { grouped[$0] ?? [] }
    tableView.reloadData()
}

func numberOfSections(in tableView: UITableView) -> Int { sections.count }
func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    sectionedItems[section].count
}
func sectionIndexTitles(for tableView: UITableView) -> [String]? { sections }
```

---

## Row Height

### Fixed height — fastest

```swift
tableView.rowHeight = 44  // set once, no delegate call needed
tableView.estimatedRowHeight = 0  // disable estimation when height is fixed
```

### Dynamic height (self-sizing cells)

```swift
tableView.rowHeight = UITableView.automaticDimension
tableView.estimatedRowHeight = 80  // provide a good estimate to avoid layout thrashing

// Cell must use Auto Layout with constraints from top to bottom of contentView
```

A bad `estimatedRowHeight` causes visible jumps in scroll position as cells are measured. Profile with the scroll position indicator — jumps indicate poor estimates.

### Variable height from data — cache it

```swift
var heightCache: [IndexPath: CGFloat] = [:]

func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
    if let cached = heightCache[indexPath] { return cached }
    let height = computeHeight(for: data[indexPath.row])
    heightCache[indexPath] = height
    return height
}
```

Invalidate the cache when data changes.

---

## Batch Updates — Animate Changes Correctly

Never call `reloadData()` when you can use batch updates — `reloadData()` is jarring and loses scroll position context.

### iOS 13+ diffable data source (preferred)

```swift
var dataSource: UITableViewDiffableDataSource<Section, Item>!

func applySnapshot(items: [Item], animatingDifferences: Bool = true) {
    var snapshot = NSDiffableDataSourceSnapshot<Section, Item>()
    snapshot.appendSections([.main])
    snapshot.appendItems(items)
    dataSource.apply(snapshot, animatingDifferences: animatingDifferences)
}
```

Diffable data source computes the diff automatically. `Item` must be `Hashable`.

### Manual batch updates (UIKit)

```swift
tableView.performBatchUpdates({
    tableView.insertRows(at: insertedPaths, with: .automatic)
    tableView.deleteRows(at: deletedPaths, with: .automatic)
    tableView.reloadRows(at: reloadedPaths, with: .automatic)
    // Index paths must be consistent with the data model AFTER the updates
}, completion: nil)
```

**Critical**: index paths for insertions are in the post-update coordinate space; index paths for deletions are in the pre-update coordinate space. Update your data model inside the `performBatchUpdates` block before calling insert/delete.

### Ordering of operations

When mixing inserts and deletes in one batch:
1. Deletions are processed first (using original index paths)
2. Insertions are processed second (using final index paths)

```swift
// Update data BEFORE calling batch update methods
data.remove(at: deleteIndex)       // mutate data first
data.insert(newItem, at: insertIndex)
tableView.performBatchUpdates({
    tableView.deleteRows(at: [IndexPath(row: deleteIndex, section: 0)], with: .fade)
    tableView.insertRows(at: [IndexPath(row: insertIndex, section: 0)], with: .automatic)
})
```

---

## Editing and Reordering

### Enabling editing mode

```swift
// Toggle from navigation bar button
navigationItem.leftBarButtonItem = editButtonItem

override func setEditing(_ editing: Bool, animated: Bool) {
    super.setEditing(editing, animated: animated)
    tableView.setEditing(editing, animated: animated)
}
```

### Deletion

```swift
// Control which rows can be deleted
func tableView(_ tableView: UITableView, canEditRowAt indexPath: IndexPath) -> Bool {
    return indexPath.row != 0  // protect first row
}

func tableView(_ tableView: UITableView, editingStyleForRowAt indexPath: IndexPath) -> UITableViewCell.EditingStyle {
    return .delete
}

func tableView(_ tableView: UITableView, commit editingStyle: UITableViewCell.EditingStyle, forRowAt indexPath: IndexPath) {
    if editingStyle == .delete {
        data.remove(at: indexPath.row)  // update model FIRST
        tableView.deleteRows(at: [indexPath], with: .automatic)
    }
}
```

### Reordering

```swift
func tableView(_ tableView: UITableView, canMoveRowAt indexPath: IndexPath) -> Bool { true }

func tableView(_ tableView: UITableView, moveRowAt source: IndexPath, to destination: IndexPath) {
    let item = data.remove(at: source.row)
    data.insert(item, at: destination.row)
}

// Prevent moving into restricted sections
func tableView(_ tableView: UITableView,
               targetIndexPathForMoveFromRowAt source: IndexPath,
               toProposedIndexPath proposed: IndexPath) -> IndexPath {
    if proposed.section == 0 { return source }  // block moves to section 0
    return proposed
}
```

---

## Cell Content Design for Performance

### Flat view hierarchy inside cells

Deep view hierarchies inside cells add layout and compositing overhead that multiplies by the number of visible cells.

- Prefer `UITableViewCell` built-in `textLabel`, `detailTextLabel`, `imageView` for simple content
- Use custom cells only when standard styles don't fit
- In custom cells: keep subview count minimal, use Auto Layout anchors (not nested stack views 4 levels deep)

### Images in cells

Never load images synchronously in `cellForRowAt`:

```swift
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "ImageCell", for: indexPath) as! ImageCell
    cell.imageView?.image = UIImage(named: "placeholder")
    
    let url = data[indexPath.row].imageURL
    ImageLoader.shared.load(url) { [weak cell] image in
        guard let cell = cell, cell.imageView?.image == UIImage(named: "placeholder") else { return }
        DispatchQueue.main.async { cell.imageView?.image = image }
    }
    return cell
}
```

Cancel pending image loads when a cell is reused (implement in `prepareForReuse`).

### prepareForReuse — cancel async work

```swift
override func prepareForReuse() {
    super.prepareForReuse()
    imageView?.image = UIImage(named: "placeholder")
    imageLoadTask?.cancel()   // cancel pending network/disk load
    imageLoadTask = nil
}
```

---

## Selection Handling

```swift
func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
    tableView.deselectRow(at: indexPath, animated: true)  // always deselect with animation
    let item = data[indexPath.row]
    // navigate or act on item
}
```

### Selection list (radio button pattern)

```swift
var selectedIndex: Int?

func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
    let previous = selectedIndex
    selectedIndex = indexPath.row
    
    var toReload = [indexPath]
    if let prev = previous { toReload.append(IndexPath(row: prev, section: 0)) }
    tableView.reloadRows(at: toReload, with: .none)
}

func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
    cell.accessoryType = indexPath.row == selectedIndex ? .checkmark : .none
    return cell
}
```

---

## Programmatic Scroll and Selection

```swift
// Scroll to a specific row
tableView.scrollToRow(at: IndexPath(row: 5, section: 0), at: .middle, animated: true)

// Select a row programmatically (does not call didSelectRowAt)
tableView.selectRow(at: indexPath, animated: true, scrollPosition: .none)

// Trigger the delegate manually if needed
tableView.delegate?.tableView?(tableView, didSelectRowAt: indexPath)
```

---

## Code Review Checklist

- [ ] Does `cellForRowAt` use `dequeueReusableCell(withIdentifier:for:)` (never `init` directly)?
- [ ] Does cell configuration reset all dynamic properties (not just set new ones)?
- [ ] Is the data model pre-processed so `cellForRowAt` does only O(1) lookups?
- [ ] Is `rowHeight` set to a constant when all rows are the same height?
- [ ] Is `estimatedRowHeight` set to a realistic value for auto-dimension cells?
- [ ] Are batch updates used instead of `reloadData()` for incremental changes?
- [ ] In `performBatchUpdates`: is the data model mutated before calling insert/delete methods?
- [ ] Does `prepareForReuse` cancel pending async operations (image loads, tasks)?
- [ ] Is image loading async with a placeholder, not synchronous in `cellForRowAt`?
- [ ] Are `canEditRowAt` / `canMoveRowAt` used to protect rows that shouldn't be edited/moved?
