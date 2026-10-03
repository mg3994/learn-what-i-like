---
name: flutter-testing-and-tooling
description: Zero-build_runner testing strategies, pure Dart fakes, deterministic BlocSignal unit testing, and automated package split migration workflows.
rules:
  - "Eliminate Mockito and build_runner for unit testing; write pure Dart hand-crafted fakes."
  - "Run unit tests synchronously in milliseconds without pumpWidget or async pumps where possible."
  - "Test CubitSignal and BlocSignal state transitions deterministically using value comparison."
  - "Automate package split migrations using AST/regex transformation scripts."
---

# Flutter Testing, Zero-`build_runner` Mocking, and Package Migration

This skill outlines testing practices and codebase refactoring tools based on Randal L. Schwartz's engineering principles.

---

## 1. Stop Paying the `build_runner` Tax: Pure Dart Fakes

### The Cost of `build_runner`
Using `Mockito` with `@GenerateMocks` forces developer workflows to pause for minutes while code generation runs. It introduces fragile build artifacts, pollutes git history, and slows TDD cycles.

### The Pure Dart Fake Solution
Dart's sound null safety and strong static typing make hand-crafted pure Dart fakes fast, explicit, and maintenance-free.

```dart
// Contract
abstract class UserRepository {
  Future<User> getUser(String id);
  Future<void> saveUser(User user);
}

// Zero-dependency Pure Dart Fake
class FakeUserRepository implements UserRepository {
  final Map<String, User> _db = {};
  bool shouldThrowError = false;

  @override
  Future<User> getUser(String id) async {
    if (shouldThrowError) throw Exception('Database failure');
    return _db[id] ?? User(id: id, name: 'Default User');
  }

  @override
  Future<void> saveUser(User user) async {
    if (shouldThrowError) throw Exception('Database failure');
    _db[user.id] = user;
  }
}
```

---

## 2. Deterministic Unit Testing in BlocSignal

BlocSignal state transitions occur synchronously via Signals. Unit tests do not require `async`/`await` pumps or `testWidgets`.

```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('CounterCubit Unit Tests', () {
    late CounterCubit cubit;

    setUp(() {
      cubit = CounterCubit();
    });

    test('initial state is 0', () {
      expect(cubit.state, equals(0));
    });

    test('increment updates state synchronously', () {
      cubit.increment();
      expect(cubit.state, equals(1));

      cubit.increment();
      expect(cubit.state, equals(2));
    });
  });
}
```

---

## 3. Automating Package Split Migrations

When splitting a monolithic package into modular packages, automate import transformations using Dart scripts:

```dart
import 'io.dart';

void migratePackageImports(Directory dir, String oldPkg, String newPkg) {
  for (final entity in dir.listSync(recursive: true)) {
    if (entity is File && entity.path.endsWith('.dart')) {
      final content = entity.readAsStringSync();
      if (content.contains(oldPkg)) {
        final updated = content.replaceAll(
          "package:$oldPkg/",
          "package:$newPkg/",
        );
        entity.writeAsStringSync(updated);
        print('Updated imports in: ${entity.path}');
      }
    }
  }
}
```
