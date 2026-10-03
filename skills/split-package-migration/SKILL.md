---
name: split-package-migration
description: Automating Flutter and Dart split package migrations, monorepo refactoring, and package extraction using Melos, Dart AST transformations, and AI agent scripts.
rules:
  - "Use Melos for Dart/Flutter monorepo package workspace management."
  - "Automate package import refactoring via AST/regex transformation scripts."
  - "Keep domain interfaces in lightweight core packages; keep implementation details in feature packages."
  - "Enforce strict dependency boundaries: feature packages must never depend on each other directly."
---

# Split Package Migration & Monorepo Architecture

This skill provides patterns and automation scripts for splitting monolithic Flutter/Dart codebases into modular packages using **Melos** and automated refactoring scripts, based on Randal L. Schwartz's package engineering principles.

---

## 1. Why Monolithic Flutter Apps Fail at Scale

As Flutter codebases grow beyond 50,000 lines:
- Incremental compile times degrade.
- Circular dependencies creep into feature modules.
- Merge conflicts increase across large engineering teams.

Splitting monolithic apps into **independent internal packages** restores fast build times and enforces strict architectural boundaries.

---

## 2. Monorepo Structure with Melos

Organize packages under a `packages/` directory managed by **Melos**:

```
my_app_monorepo/
├── melos.yaml
├── pubspec.yaml
└── packages/
    ├── app_core/           (Shared models, interfaces, BlocSignal primitives)
    ├── feature_auth/       (Authentication domain & UI)
    ├── feature_payment/    (Payment integration & state machines)
    └── network_client/     (HTTP & Isolate networking)
```

### `melos.yaml` Configuration
```yaml
name: my_app_monorepo
packages:
  - packages/**

scripts:
  analyze:
    run: melos exec -- dart analyze .
  test:
    run: melos exec -- flutter test
```

---

## 3. Automated Import Refactoring Script

When extracting code from `lib/src/` into a new package `package:feature_auth/`, automate the import replacements across hundreds of files using a pure Dart script:

```dart
import 'dart:io';

void main() {
  final targetDir = Directory('lib');
  const oldImportPrefix = 'package:monolith/src/auth/';
  const newImportPrefix = 'package:feature_auth/';

  int modifiedFiles = 0;

  for (final entity in targetDir.listSync(recursive: true)) {
    if (entity is File && entity.path.endsWith('.dart')) {
      final content = entity.readAsStringSync();
      if (content.contains(oldImportPrefix)) {
        final updatedContent = content.replaceAll(oldImportPrefix, newImportPrefix);
        entity.writeAsStringSync(updatedContent);
        modifiedFiles++;
        print('Updated imports in: ${entity.path}');
      }
    }
  }

  print('Migration complete! Updated $modifiedFiles files.');
}
```
