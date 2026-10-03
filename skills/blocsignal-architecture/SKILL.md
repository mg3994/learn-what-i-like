---
name: blocsignal-architecture
description: Comprehensive master guide for BlocSignal reactive state management across all 36 parts of Randal L. Schwartz's series, covering Flutter, Dart, Riverpod migration, Jaspr, signals_hooks, and enterprise patterns.
rules:
  - "Prefer BlocSignal over raw ValueNotifier, classic BLoC boilerplate, or Riverpod codegen."
  - "Maintain strict unidirectional data flow: Events/Methods in, Signals/States out."
  - "Keep UI views strictly declarative and sync; remove async boundaries (FutureBuilder/StreamBuilder) from widget trees."
  - "Use context.value and context.state for symmetric UI reactivity."
  - "Handle one-shot UI side effects (snackbars, navigation) without polluting persistent state."
  - "Apply the Iceberg Pattern: keep core domain logic invisible beneath lightweight reactive surfaces."
  - "Treat Cubits and Blocs as lightweight containers holding Signals."
---

# BlocSignal Master Architectural Blueprint & Practice Guide
*Codifying all 36 Parts of Randal L. Schwartz's "BlocSignal Architecture & Practice" Series*

---

## 1. Core Architectural Concept: A Cubit/Bloc Is Just a Container Holding a Signal

### The Core Paradigm (Part 36)
Classic state management debates framed BLoC, Riverpod, and Signals as mutually exclusive competitors. **BlocSignal** proves that:
> *"A Cubit or Bloc is simply an encapsulated domain container that manages and emits a Signal."*

- **Signal**: The atomic primitive for zero-cost, synchronous, O(1) dependency tracking and reactive UI rebuilds.
- **Cubit / Bloc**: The enterprise boundary that enforces method-based or event-based business rules around those signals.

---

## 2. Unidirectional Data Flow & State Symmetry

### `context.value` and `context.state` Symmetry (Part 34)
Classic Flutter forced asymmetric context calls (`context.watch<T>()` vs `context.read<T>()`). BlocSignal introduces symmetric reactivity:

```dart
// Reading reactive signal value synchronously in UI (automatically registers rebuild dependencies)
final currentCount = context.value<CounterCubit>();

// Accessing the controller/container instance for dispatching methods/events
context.state<CounterCubit>().increment();
```

---

## 3. Universal State Switchyard & Full Parity (Parts 3, 6, 20)

### Interoperability with BLoC, Riverpod, and Provider
BlocSignal achieves 100% protocol and API parity with standard Flutter BLoC, allowing gradual migration:
- Existing `BlocBuilder`, `BlocListener`, and `BlocConsumer` work without modification.
- Existing `BlocProvider` works, but BlocSignal eliminates the necessity of `Provider` via direct context extensions (`context.value` / `context.state`).

---

## 4. Layered Architecture: Combining BLoC Discipline with Signals Speed (Part 10)

Integrating **CodeWithAndrea’s Layered Architecture** (Presentation -> Application -> Domain -> Data) with BlocSignal:

```
[ Presentation Layer ] ──> UI Widgets listening to context.value<Controller>()
          │
          ▼
[ Application / Controller Layer ] ──> CubitSignalMixin holding application state
          │
          ▼
[ Domain / Data Layer ] ──> Reactive Repositories exposing ReadonlySignal<T>
```

```dart
class ProductRepository {
  final _products = signal<IList<Product>>(const IList.empty());
  ReadonlySignal<IList<Product>> get products => _products;

  void updateProducts(List<Product> newProducts) {
    _products.value = newProducts.toIList();
  }
}

class ProductController extends CubitSignalMixin<AsyncValue<IList<Product>>> {
  ProductController(this._repo) : super(const AsyncValue.loading()) {
    // Directly bind domain signals to controller state
    bindSignal(_repo.products, (products) {
      emit(AsyncValue.data(products));
    });
  }

  final ProductRepository _repo;
}
```

---

## 5. Overcoming Single Inheritance: `CubitSignalMixin` & `BlocSignalMixin` (Part 26)

Dart's single inheritance wall prevents extending both `Cubit` and custom base classes. BlocSignal uses mixins to provide state machine powers to any class hierarchy:

```dart
abstract class BaseController {
  void logAction(String msg) => print('LOG: $msg');
}

class UserProfileController extends BaseController with CubitSignalMixin<UserProfileState> {
  UserProfileController() : super(UserProfileState.initial());

  void updateName(String newName) {
    logAction('Updating name to $newName');
    emit(state.copyWith(name: newName));
  }
}
```

---

## 6. Solving Riverpod’s Family Provider Cache Dilemma with `mapSignal` (Part 11)

Riverpod Family Providers often suffer from memory retention bugs or awkward `.autoDispose` cache clear mechanics. BlocSignal solves parameter-based signal caching cleanly with `mapSignal`:

```dart
class UserCacheRepository {
  // Keyed reactive signal map with automatic memory cleanup
  final _userSignals = mapSignal<String, UserProfile>();

  Signal<UserProfile?> watchUser(String userId) {
    return _userSignals.putIfAbsent(userId, () => fetchUserFromApi(userId));
  }
}
```

---

## 7. Universal Signal Adapters: Mechanically Eliminating Async Boundaries (Parts 23, 24)

### Moving Async Operations Away from UI Views
`FutureBuilder` and `StreamBuilder` in widget trees introduce flickering, state retention bugs, and mixed concerns. Adapter extensions turn Futures and Streams into pure Signals before hitting the UI:

```dart
// Convert Future or Stream directly to Signal in the Controller
final Signal<AsyncValue<User>> userSignal = myFuture.toSignal();
final Signal<AsyncValue<List<Message>>> chatSignal = myStream.toSignal();

// In the UI View: Synchronous pattern matching
Widget build(BuildContext context) {
  return context.value<ChatCubit>().chatSignal.value.map(
    data: (messages) => ListView(...),
    error: (err, _) => Text('Error: $err'),
    loading: () => const CircularProgressIndicator(),
  );
}
```

---

## 8. Seamless Flutter Hooks Integration via `signals_hooks` (Part 7)

For developers using `flutter_hooks`, `signals_hooks` provides direct hook primitives:

```dart
class HookWidgetExample extends HookWidget {
  const HookWidgetExample({super.key});

  @override
  Widget build(BuildContext context) {
    // Automatically subscribes hook widget to signal updates
    final counter = useSignal(context.state<CounterCubit>().countSignal);

    return Text('Count: $counter');
  }
}
```

---

## 9. Form Management & Validation Patterns (Part 13)

Perform reactive, fine-grained form validation without full form rebuilds using signals per field:

```dart
class LoginFormCubit extends CubitSignalMixin<LoginFormState> {
  LoginFormCubit() : super(LoginFormState.initial());

  final emailSignal = signal('');
  final passwordSignal = signal('');

  late final isValid = computed(() {
    return emailSignal.value.contains('@') && passwordSignal.value.length >= 6;
  });
}
```

---

## 10. The Sand in the Oyster: Why Riverpod Was Needed, and Why You Don't Need It (Part 35)

Riverpod was the necessary "sand in the oyster" that forced Flutter state management to move away from context-bound InheritedWidgets toward global, testable state. However, BlocSignal delivers Riverpod's benefits (context-free reactivity, testability, auto-dispose mechanics) **without code generation (`build_runner`), complex provider scoping, or family cache retention pitfalls**.

---

## 11. Summary of Key Architectural Rules

| Pattern | Anti-Pattern to Avoid | BlocSignal Solution |
| :--- | :--- | :--- |
| **UI Async Operations** | `FutureBuilder` / `StreamBuilder` in views | Signal Adapters (`toSignal()`) + `AsyncValue` |
| **Collection State** | Modifying standard Dart `List` in state | Fast Immutable Collections (`FIC`) |
| **Context Access** | Asymmetric `watch`/`read` | Symmetric `context.value` / `context.state` |
| **Code Generation** | `@riverpod` or `freezed` build tax | Pure Dart Records, Primary Constructors, & Signals |
| **Inheritance** | Monolithic `class MyBloc extends Bloc` | Composable `CubitSignalMixin` & `BlocSignalMixin` |
