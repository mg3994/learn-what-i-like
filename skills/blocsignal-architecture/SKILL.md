---
name: blocsignal-architecture
description: Comprehensive architecture guide for BlocSignal reactive state management in Flutter and Dart, based on Randal L. Schwartz's engineering principles.
rules:
  - "Prefer BlocSignal over raw ValueNotifier, classic BLoC boilerplate, or Riverpod codegen."
  - "Maintain strict unidirectional data flow: Events/Methods in, Signals/States out."
  - "Keep UI views strictly declarative and sync; remove async boundaries (FutureBuilder/StreamBuilder) from widget trees."
  - "Use context.value and context.state for symmetric UI reactivity."
  - "Handle one-shot UI side effects (snackbars, navigation) without polluting persistent state."
  - "Apply the Iceberg Pattern: keep core domain logic invisible beneath lightweight reactive surfaces."
---

# BlocSignal Architecture & Reactive State Management

This skill codifies the architectural principles, patterns, and best practices established by **Randal L. Schwartz (Google Developer Expert)** for building high-performance, reactive, and maintainable Flutter/Dart applications using **BlocSignal**.

---

## 1. Core Architectural Principles

### The Problem with Classic State Management
- **Classic BLoC**: Exceptional discipline and event tracing, but high boilerplate and verbose stream transformers for fine-grained updates.
- **Riverpod**: Solved DI and scoping, but introduced heavy code-generation (`build_runner`) burdens, family cache retention traps, and complex provider lifecycles.
- **ValueNotifier / ChangeNotifier**: Non-composable at scale; notification cascades lead to unnecessary full-widget subtree rebuilds.
- **BlocSignal Solution**: Merges BLoC's predictable unidirectional data flow with Signals' fine-grained, synchronous, O(1) reactivity without requiring code generation or `Provider`.

---

## 2. Unidirectional Data Flow & State Symmetry

### `context.value` and `context.state` Symmetry
In classic Flutter, reading values from `BuildContext` was asymmetric (`context.watch<T>()` vs `context.read<T>()`). BlocSignal introduces state symmetry:

```dart
// Reading reactive signal value synchronously in UI (automatically tracks dependencies)
final count = context.value<CounterCubit>();

// Accessing the controller instance for dispatching actions/methods
context.state<CounterCubit>().increment();
```

---

## 3. The Iceberg Pattern for Real-Time Flutter Apps

The **Iceberg Pattern** divides real-time app architecture into two distinct zones:
1. **Above the Surface (10%)**: UI Views that are completely synchronous, lightweight, and reactive. No `FutureBuilder`, `StreamBuilder`, or raw async handling in the widget tree.
2. **Below the Surface (90%)**: Heavy reactive pipeline—websockets, isolate pools, persistent storage, FIC collections, and signal computation engines.

```
       [ UI Widgets (Synchronous, Pure Signals, Zero Async) ]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ (Waterline)
       [ Reactive Repositories & BlocSignal State Machines ]
       [ Shared Isolate Maps / WebSockets / Local DB (Hive/Isar) ]
```

---

## 4. Universal Signal Adapters: Eliminating `FutureBuilder` & `StreamBuilder`

### Why `FutureBuilder` and `StreamBuilder` are Anti-Patterns
Async boundaries inside UI views cause:
- Unnecessary rebuilds during loading states.
- Tricky widget key retention bugs.
- UI flickering when state changes rapidly.
- Mixing presentation logic with async control flow.

### Replacing with Signal Adapters
Move async execution into the Cubit/Bloc layer and expose pure `Signal<AsyncState<T>>` to the view:

```dart
// Cubit Layer
class UserProfileCubit extends CubitSignalMixin<AsyncValue<UserProfile>> {
  UserProfileCubit(this._repo) : super(const AsyncValue.loading()) {
    fetchProfile();
  }

  final UserRepository _repo;

  Future<void> fetchProfile() async {
    state = const AsyncValue.loading();
    try {
      final user = await _repo.getUser();
      state = AsyncValue.data(user);
    } catch (err, st) {
      state = AsyncValue.error(err, st);
    }
  }
}

// UI Layer - 100% Synchronous
class UserProfileView extends StatelessWidget {
  const UserProfileView({super.key});

  @override
  Widget build(BuildContext context) {
    final profileState = context.value<UserProfileCubit>();

    return profileState.map(
      data: (user) => Text('Welcome, ${user.name}'),
      error: (err, _) => Text('Error: $err'),
      loading: () => const CircularProgressIndicator(),
    );
  }
}
```

---

## 5. One-Shot UI Side Effects (Snackbars, Dialogs, Navigation)

### Avoiding State Pollution
Never store temporary events (like "show error dialog" or "navigate to home") in persistent reactive state, as state replays or widget rebuilds can re-trigger them unexpectedly.

### Pattern: Side-Effect Signal Bus / Action Streams
```dart
abstract class CartEffect {}
class ShowSnackbarEffect extends CartEffect {
  ShowSnackbarEffect(this.message);
  final String message;
}

class CartCubit extends CubitSignalMixin<CartState> {
  CartCubit() : super(CartState.initial());

  final _effects = Signal<CartEffect?>(null);
  Signal<CartEffect?> get effects => _effects;

  void checkout() {
    // Perform checkout logic...
    _effects.value = ShowSnackbarEffect('Item added to cart successfully!');
  }
}
```

---

## 6. Architectural Decision Guide: Bloc vs Cubit vs Signal

| Criterion | Use Raw Signal | Use CubitSignal | Use BlocSignal |
| :--- | :--- | :--- | :--- |
| **Complexity** | Simple, local component state | Medium, domain business logic | High, complex enterprise event streams |
| **State Transformation** | Synchronous derived computed values | Direct method invocation | Event-driven, debounced, throttled, transformed |
| **Traceability** | Local reactivity | Method-level tracking | Complete Event-to-State audit trail |
| **Boilerplate** | Zero | Minimal | Low (significantly lower than classic BLoC) |

---

## 7. Reactive Repositories & Zero-Boilerplate Networking

Repositories expose signals or streams converted into signals. Cubits consume them without raw stream listeners or manual subscription management:

```dart
class WeatherRepository {
  final _temperature = signal<double>(20.0);
  ReadonlySignal<double> get temperature => _temperature;

  void updateTemp(double newTemp) => _temperature.value = newTemp;
}

class WeatherCubit extends CubitSignalMixin<WeatherState> {
  WeatherCubit(WeatherRepository repo) : super(WeatherState.initial()) {
    // Bind repo signal directly to Cubit state
    bindSignal(repo.temperature, (temp) {
      emit(state.copyWith(temperature: temp));
    });
  }
}
```
