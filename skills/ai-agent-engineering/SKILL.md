---
name: ai-agent-engineering
description: Autonomous AI coding agent workflows, Socratic prompting physics, NLP meta-model pruning, forced continuity defect remedies, and synthetic trauma transfer.
rules:
  - "Prefer Dart over Python for enterprise agent implementation and typed safety."
  - "Apply Socratic Prompting: guide agents with targeted constraint questions rather than direct scalar fixes."
  - "Eliminate the Forced Continuity Defect: force agents to explicitly validate failure states, edge cases, and network timeouts before happy-path code."
  - "Bridge AI knowledge cutoffs by training agents with structured, explicit Dart 1.0 -> 3.x feature specs and project skill files."
---

# AI Agent Engineering & Socratic Prompting Physics

This skill encapsulates Randal L. Schwartz's methodology for engineering autonomous AI coding agents, implementing Socratic Prompting physics, preventing 3 AM agent crashes, and eliminating happy-path illusions.

---

## 1. Socratic Prompting Physics & The NLP Meta-Model

### The Failure of Scalar Direct Fixes
Barking direct code fixes at an LLM turns the developer into an exhausted "human compiler" and creates high cognitive strain. Direct scalar patches cause **Somatic Recoil** (the LLM over-correcting in one direction and introducing regression in another).

### Socratic Prompting Principles
Instead of dictating solutions, prune the agent's search tree at depth $d=1$ using Socratic questioning based on the **NLP Meta-Model**:

```
[ Developer Input ]
       │
       ▼
"What happens to the state machine if the network drops on packet 3 of 5?"
       │
       ▼
[ AI Agent Internal Tree Search Pruning ] ──> Evaluates Failure Branch
       │
       ▼
[ Robust Code Architecture Generation ]
```

#### Key Socratic Question Frameworks:
1. **De-deletion**: *"What hidden assumptions exist regarding the lifetime of this subscription?"*
2. **De-distortion**: *"How does this state change behave when two events are dispatched concurrently?"*
3. **De-generalization**: *"Is it true that this signal always emits synchronously, or can it throw under isolate execution?"*

---

## 2. Preventing 3 AM Crashes: The Forced Continuity Defect

### The Forced Continuity Defect
LLMs are trained on happy-path code completions. When encountering errors, their default autoregressive bias tries to "continue the happy sequence" (forced continuity), resulting in shallow error handling (e.g. empty `catch (e) {}`).

### Teaching Agents Pain (Synthetic Scars)
Force agents to write **Failure-First Architecture**:

```dart
// AGENT INSTRUCTION PATTERN: Failure-First Execution Boundary
Future<Result<Data, NetworkException>> fetchData() async {
  try {
    final response = await _client.get('/endpoint').timeout(const Duration(seconds: 5));
    if (response.statusCode != 200) {
      return Result.failure(ServerException(response.statusCode));
    }
    return Result.success(Data.fromJson(response.data));
  } on TimeoutException catch (e, st) {
    // Explicit pain handling
    _telemetry.logError('Network timeout in fetchData', e, st);
    return Result.failure(NetworkTimeoutException());
  } catch (e, st) {
    _telemetry.logCritical('Unexpected failure', e, st);
    return Result.failure(UnknownException(e.toString()));
  }
}
```

---

## 3. Why "Prefer Dart Over Python" for Coding Agents

1. **Strong Static Typing**: Catch 90% of LLM halluciations at compile time rather than runtime.
2. **Sound Null Safety**: Prevents null pointer errors (`NullPointerException` / `AttributeError: NoneType`).
3. **Isolate Isolation**: Prevents background crashes from polluting the primary execution loop.
4. **Single Binary Executables**: `dart compile exe` produces fast CLI agent tools with instant startup.

---

## 4. Bridging AI Knowledge Cutoffs (Dart 1.0 to 3.14)

Provide explicit context maps in project repository guidelines (e.g. `AGENTS.md` / `SKILL.md`) so the LLM uses modern syntax:
- Primary Constructors (`class C(this.x);`)
- Records & Pattern Matching (`var (a, b) = pair;`)
- Class Modifiers (`sealed`, `final`, `interface`, `base`)
- Enhanced Enums with constructor tearoffs
