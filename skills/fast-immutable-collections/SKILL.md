---
name: fast-immutable-collections
description: Structural immutability in Dart and Flutter using fast_immutable_collections (FIC) to eliminate defensive shallow copies, improve state comparison performance, and prevent state pollution.
rules:
  - "Use IList, IMap, and ISet instead of standard Dart List, Map, and Set in state objects."
  - "Leverage structural sharing for O(1) mutations (add, remove, replace) without shallow copying."
  - "Do not use List.unmodifiable; it throws runtime errors on mutation and still requires shallow copies for state updates."
  - "Enable compile-time constant empty collections: `const IList.empty()`."
---

# Fast Immutable Collections (FIC) for Flutter & Dart State Management

This skill codifies structural immutability patterns using `fast_immutable_collections` (FIC) as specified by Randal L. Schwartz for building bulletproof Flutter state objects.

---

## 1. Why Standard Dart Collections Fail in State Management

In state management patterns (BLoC, Cubit, Redux, Signals), state immutability is mandatory. However, standard Dart `List` / `Map` collections present two major problems:

1. **Defensive Shallow Copy Overhead ($O(N)$)**: Adding an item forces `List.from(oldList)..add(newItem)`, allocating a brand-new array copy. For 10,000 items, this degrades UI frame rates.
2. **Accidental In-Place Mutation**: Calling `.add()` or `[0] = item` on a standard list mutates the existing state in-place, breaking state change listeners and UI rebuild detection.

---

## 2. Fast Immutable Collections (FIC) Structural Sharing

FIC provides `IList`, `IMap`, and `ISet` which use **structural sharing** (tree-based nodes):
- **Adding / Removing items**: $O(1)$ time complexity without copying unchanged elements.
- **Equality Comparison**: $O(1)$ identity check, followed by structural hash comparison.
- **Immutability**: Compile-time safe; modification methods return a new `IList` instance without mutating the original.

```dart
import 'package:fast_immutable_collections/fast_immutable_collections.dart';

class CartState {
  CartState({required this.items});

  final IList<CartItem> items; // Structurally immutable

  CartState addItem(CartItem item) {
    // O(1) operation! Reuses existing node tree.
    return CartState(items: items.add(item));
  }

  CartState removeItem(String id) {
    return CartState(items: items.removeWhere((i) => i.id == id));
  }
}
```

---

## 3. FIC Helper Extensions

FIC provides fluent extension methods on standard Dart collections:

```dart
// Convert standard List to IList
final IList<String> immutableNames = ['Alice', 'Bob'].toIList();

// Convert IMap back to standard Map when required by external APIs
final Map<String, dynamic> jsonMap = immutableMap.unlock;

// Configurable global behavior (e.g. strict equality checks)
void setupFic() {
  ImmutableCollection.defaultConfig = ConfigList(
    isDeepEquals: true,
    autoFlush: true,
  );
}
```

---

## 4. Comparing Immutable Collection Options

| Feature | Standard `List` | `List.unmodifiable` | `freezed` list | FIC (`IList`) |
| :--- | :--- | :--- | :--- | :--- |
| **Mutation Prevention** | None (Mutable) | Runtime Exception | Shallow copy | Compile-time Immutable |
| **Add / Remove Cost** | In-place / $O(N)$ copy | N/A (Throws) | $O(N)$ shallow copy | **$O(1)$ Structural Sharing** |
| **Equality Check** | Identity only ($O(1)$) | Identity only | Deep check ($O(N)$) | **Fast Structural ($O(1)$)** |
| **Code Generation** | None | None | Required (`build_runner`) | **Zero Codegen** |
