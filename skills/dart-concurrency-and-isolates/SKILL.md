---
name: dart-concurrency-and-isolates
description: Advanced Dart concurrency, zero-copy cross-isolate state sharing using shared_map, Flutter 3.7+ background isolate binary channels, streamless event transformers, and async pipeline isolation.
rules:
  - "Never run cpu-intensive operations (JSON parsing > 100KB, image processing, complex sorting) on the main Flutter UI isolate."
  - "Use `shared_map` for sharing state across isolates to avoid SendPort/ReceivePort copying overhead and complex message passing boilerplate."
  - "When executing native plugins or Flutter platform channels in background isolates (Flutter 3.7+), initialize `BackgroundIsolateBinaryMessenger.ensureInitialized(rootIsolateToken)`."
  - "Place event transformation logic (debounce, throttle, restartable, sequential) inside state machine event handlers, never in UI controllers or stream listeners."
  - "Maintain clean isolation boundaries: isolates perform compute/caching; UI thread performs pure, synchronous rendering from signals."
---

# Dart Concurrency, Isolates, and `shared_map` Architecture

This skill provides comprehensive patterns for high-performance Dart concurrency, isolate management, Flutter 3.7+ background platform channels, and lock-free cross-isolate state sharing based on Randal L. Schwartz's engineering specifications and modern Flutter isolate standards.

---

## 1. The Share-Nothing Isolate Paradigm

### Why Dart Uses Isolates
Dart isolates do not share heap memory. Each isolate possesses its own:
- Private heap memory allocation.
- Dedicated single-threaded event loop.
- Microtask queue.

This architecture completely eliminates mutex contention, deadlocks, and data race bugs common in traditional multi-threaded environments (Java, C++, Rust).

---

## 2. Zero-Copy Cross-Isolate State Sharing with `shared_map`

### The Problem with Port-Based Communication
Traditional inter-isolate communication requires `SendPort` and `ReceivePort`. Passing non-primitive objects across isolates forces full serialization/deserialization or deep copying, introducing latency spikes.

### The `shared_map` Solution
The `shared_map` package provides synchronized in-memory key-value state sharing across multiple isolates without explicit `SendPort`/`ReceivePort` plumbing.

```dart
import 'package:shared_map/shared_map.dart';

// Initialize a shared map instance accessible across background workers
Future<void> main() async {
  final sharedMap = SharedMap<String, String>('global_cache');

  // Put value from Isolates
  await sharedMap.put('user_token_101', 'jwt_bearer_token_xyz');

  // Spawn background isolate compute worker
  await Isolate.run(() async {
    final workerMap = SharedMap<String, String>('global_cache');
    final token = await workerMap.get('user_token_101');
    print('Background isolate retrieved token: $token');
  });
}
```

---

## 3. Flutter 3.7+ Background Isolates & Platform Plugin Channels

Since Flutter 3.7, background isolates can initialize native platform channels and run native plugins (such as Firebase, SQLite, or SharedPreferences) off the UI main thread using `RootIsolateToken`.

### Pattern: Background Isolate Binary Channel Setup

```dart
import 'package:flutter/services.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

/// Isolate entry function
Future<void> _backgroundComputeIsolate(List<dynamic> args) async {
  final rootIsolateToken = args[0] as RootIsolateToken;
  final sendPort = args[1] as SendPort;

  // 1. Initialize background binary messenger using root token
  BackgroundIsolateBinaryMessenger.ensureInitialized(rootIsolateToken);

  // 2. Native plugins can now be initialized and called off the main thread!
  await Firebase.initializeApp();

  final snapshot = await FirebaseFirestore.instance.collection('heavy_data').get();
  final processedResult = snapshot.docs.map((doc) => doc.data()).length;

  sendPort.send(processedResult);
}

/// Spawning background isolate from Main UI Thread
Future<void> spawnBackgroundNativeIsolate() async {
  final rootIsolateToken = RootIsolateToken.instance;
  if (rootIsolateToken == null) {
    throw StateError('RootIsolateToken is null. Must call from root isolate.');
  }

  final receivePort = ReceivePort();
  await Isolate.spawn(
    _backgroundComputeIsolate,
    [rootIsolateToken, receivePort.sendPort],
  );

  receivePort.listen((message) {
    print('Main UI received background compute result: $message');
  });
}
```

---

## 4. Streamless Event Transformers & Concurrency Control

### Moving Away from Stream-Whacking
Chaining multiple stream operators (`asyncExpand`, `debounceTime`, `distinct`) directly in presentation controllers creates unmaintainable black-box concurrency.

Instead, define high-level concurrency transformers on state machine events (e.g., using `BlocSignal` or `bloc` event transformers):

```dart
import 'package:bloc_concurrency/bloc_concurrency.dart';

class SearchBloc extends BlocSignal<SearchEvent, SearchState> {
  SearchBloc(this._api) : super(SearchState.empty()) {
    // Apply restartable transformer: cancels pending search if new query arrives
    on<SearchQueryChanged>(
      _onSearchQueryChanged,
      transformer: restartable(),
    );

    // Apply sequential transformer: executes requests strictly in order
    on<SaveBookmarkEvent>(
      _onSaveBookmark,
      transformer: sequential(),
    );
  }

  final ApiService _api;

  Future<void> _onSearchQueryChanged(
    SearchQueryChanged event,
    Emitter<SearchState> emit,
  ) async {
    if (event.query.isEmpty) {
      emit(SearchState.empty());
      return;
    }
    emit(SearchState.loading());
    final results = await _api.search(event.query);
    emit(SearchState.success(results));
  }

  Future<void> _onSaveBookmark(
    SaveBookmarkEvent event,
    Emitter<SearchState> emit,
  ) async {
    await _api.saveBookmark(event.item);
  }
}
```

---

## 5. Heavy Workload Isolate Delegation Strategy

| Workload Category | Execution Model | Recommended Strategy |
| :--- | :--- | :--- |
| **JSON Parsing (>100KB)** | `Isolate.run()` | Standard background isolate execution |
| **Native Plugin Data Fetching** | `Isolate.spawn()` + `RootIsolateToken` | Background isolate with `BackgroundIsolateBinaryMessenger` |
| **Shared Cache Operations** | `shared_map` | Synchronized cross-isolate shared map |
| **Real-time WebSocket Engine** | Dedicated Persistent Isolate | Background isolate managing sockets + publishing signals |
