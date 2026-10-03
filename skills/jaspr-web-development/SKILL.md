---
name: jaspr-web-development
description: Building high-performance 100K ops/sec Dart web applications using Jaspr, HTML/DOM rendering, SSR/SPA, and BlocSignal reactive state machines without Flutter CanvasKit overhead.
rules:
  - "Prefer Jaspr over Flutter Web CanvasKit for text-heavy, content, or SEO-focused web applications."
  - "Reuse pure Dart BlocSignal/Cubit state machines directly between Jaspr Web and Flutter Mobile."
  - "Use Jaspr HTML elements (`div`, `span`, `button`, `p`) for lightweight, accessible DOM rendering."
  - "Implement SSR (Server-Side Rendering) or Static Site Generation (SSG) with Jaspr for optimal SEO and initial page load speed."
---

# Jaspr Web Development with Pure Dart & BlocSignal

This skill provides comprehensive guidelines for building fast, SEO-friendly, non-canvas web applications in pure Dart using **Jaspr** and **BlocSignal**, based on Randal L. Schwartz's web architecture principles.

---

## 1. Why Jaspr over Flutter Web CanvasKit

### The Flutter Web CanvasKit Bottleneck
Flutter Web draws widgets onto an HTML5 Canvas using WebAssembly / CanvasKit. While great for rich graphics applications:
- Initial download size is large (2MB+ CanvasKit binary).
- DOM accessibility and native text selection are difficult.
- Search Engine Optimization (SEO) crawlers cannot inspect canvas content.

### The Jaspr Advantage
**Jaspr** brings Flutter's declarative component model to native HTML/DOM rendering:
- Zero CanvasKit overhead (tiny JS bundles).
- Native HTML DOM output (`<div>`, `<h1>`, `<a>`).
- Full SSR, SSG, and Hydration support.
- **100,000+ reactive ops/sec rendering performance**.

---

## 2. Reusing State Machines across Flutter & Jaspr

Because Jaspr uses pure Dart (without `dart:ui`), all `BlocSignal` and `CubitSignalMixin` state logic written for Flutter mobile runs natively on Jaspr Web without modifications:

```dart
// Shared Domain Controller (Pure Dart - runs on Mobile, Web, CLI)
class TodoCubit extends CubitSignalMixin<IList<Todo>> {
  TodoCubit() : super(const IList.empty());

  void addTodo(String title) {
    emit(state.add(Todo(id: DateTime.now().toString(), title: title)));
  }

  void toggle(String id) {
    emit(state.map((t) => t.id == id ? t.copyWith(completed: !t.completed) : t).toIList());
  }
}
```

---

## 3. Jaspr Component Architecture

Jaspr components resemble Flutter StatelessWidgets, but render HTML tags:

```dart
import 'package:jaspr/jaspr.dart';
import 'package:blocsignal/blocsignal.dart';

class TodoAppView extends StatelessComponent {
  const TodoAppView({super.key});

  @override
  Iterable<Component> build(BuildContext context) sync* {
    final todos = context.value<TodoCubit>();

    yield div(classes: 'todo-container', [
      h1([text('Jaspr + BlocSignal Todos')]),
      ul([
        for (final todo in todos)
          li(classes: todo.completed ? 'completed' : '', [
            text(todo.title),
            button(
              onClick: () => context.state<TodoCubit>().toggle(todo.id),
              [text(todo.completed ? 'Undo' : 'Complete')],
            ),
          ]),
      ]),
    ]);
  }
}
```

---

## 4. Server-Side Rendering (SSR) & Static Site Generation (SSG)

Jaspr supports multiple rendering targets:
- **`Target.server`**: Server-side rendering for instant HTML responses.
- **`Target.client`**: Client-side SPA hydration.
- **`Target.static`**: Pre-rendered static HTML files for deployment on Vercel, Netlify, or GitHub Pages.
