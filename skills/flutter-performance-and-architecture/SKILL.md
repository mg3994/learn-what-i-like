---
name: flutter-performance-and-architecture
description: High-density rendering performance, reactive infinite scroll state machines, 6 DI patterns, and Jaspr web app architecture.
rules:
  - "Transform ScrollController into a reactive state machine using CubitSignalMixin to eliminate double-fetch and scroll stutter."
  - "Decouple state location from BuildContext to prevent ProviderNotFoundException."
  - "Choose the correct Dependency Injection flavor (Service Locator, InheritedWidget, Signal DI, Isolate Container) based on scope."
  - "Target 120 FPS by avoiding widget rebuilds via fine-grained Signal selections."
---

# Flutter Performance, Rendering, and Dependency Injection Architecture

This skill defines performance optimization, UI state machines, Jaspr web rendering, and dependency injection choices specified by Randal L. Schwartz.

---

## 1. Reactive Infinite Scroll State Machine

### Why 3 Lines of `async*` Fail
Using raw `async*` or scroll listeners inside views causes:
- Duplicate network requests when scrolled rapidly.
- Missing bottom-reach triggers on high-refresh-rate displays.
- Messy pagination state tracking inside stateful widgets.

### The `CubitSignalMixin` Solution
Turn the `ScrollController` into a reactive state machine:

```dart
class InfiniteScrollCubit extends CubitSignalMixin<ScrollState> {
  InfiniteScrollCubit(this._api) : super(ScrollState.initial());

  final ApiService _api;

  void onScrollPositionChanged(double currentPixels, double maxPixels) {
    // Reached threshold (80% of scroll height)
    if (currentPixels >= maxPixels * 0.8) {
      fetchNextPage();
    }
  }

  Future<void> fetchNextPage() async {
    if (state.isLoading || state.hasReachedEnd) return;

    emit(state.copyWith(isLoading: true));
    try {
      final newItems = await _api.getItems(page: state.currentPage + 1);
      if (newItems.isEmpty) {
        emit(state.copyWith(isLoading: false, hasReachedEnd: true));
      } else {
        emit(state.copyWith(
          isLoading: false,
          items: [...state.items, ...newItems],
          currentPage: state.currentPage + 1,
        ));
      }
    } catch (e) {
      emit(state.copyWith(isLoading: false, error: e.toString()));
    }
  }
}
```

---

## 2. The Six Flavors of Dependency Injection in Flutter

| Flavor | Mechanism | Best Use Case | Pros / Cons |
| :--- | :--- | :--- | :--- |
| **1. Constructor Injection** | Direct parameter passing | Child components, pure functions | Strict, explicit / Parameter drilling |
| **2. Service Locator (`get_it`)** | Global singleton / factory registry | Services, Repositories | Global access / Hidden dependencies if abused |
| **3. InheritedWidget / Provider** | Tree-scoped lookup via `BuildContext` | UI-scoped controllers | Safe scoping / Fails if context moved above provider |
| **4. Signal-Based DI** | Direct signal references | Modular, context-free reactive state | Zero context dependency / High flexibility |
| **5. Environment Registry** | Compile-time / runtime config injection | Flavors (dev, staging, prod) | Environment isolation |
| **6. Cross-Isolate Scope** | `shared_map` isolate lookup | Background worker tasks | Zero-copy concurrency / Requires isolate coordination |

---

## 3. High-Throughput Web Rendering with Jaspr and Pure Dart

Jaspr allows running Dart web apps with HTML/DOM rendering at 100K ops/sec without Flutter Web canvas overhead.
- Use `BlocSignal` state machines in pure Dart.
- Reuse state logic seamlessly across Flutter Mobile, Jaspr Web, and CLI tools.
