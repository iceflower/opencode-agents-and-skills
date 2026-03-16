---
name: code-quality
description: >-
  Code quality and design principles.
  Use when writing, reviewing, or refactoring code.
---
# Code Quality and Design Principles

## 1. Code Quality Pillars

Every piece of code should satisfy these six qualities:

- **Readable**: Other developers must understand the intent immediately
- **Predictable**: Code behaves as callers expect with no hidden surprises
- **Hard to misuse**: The API makes incorrect usage difficult or impossible
- **Modular**: Each unit has a single, well-defined responsibility
  with minimal coupling
- **Reusable and generalizable**: Code avoids unnecessary assumptions
  and can be adapted without modification
- **Testable**: Code can be verified through automated tests
  without excessive setup

---

## 2. Abstraction Layers

### Layer Separation

- Each layer of code should solve one well-defined problem
  and provide a clean API to the layer above
- Implementation details must not leak through the public API
- If a function or class operates at mixed abstraction levels,
  split it

### API Design

- Public APIs define the contract —
  they must be minimal, clear, and stable
- Functions should do one thing at one level of abstraction
- Classes should encapsulate a single coherent concept
- Interfaces should define the minimal set of operations
  a caller needs

### Null Values and Pseudo-Code Contracts

- Prefer explicit types (Optional, sealed class, Result)
  over null to represent absence
- When null is unavoidable, document the contract clearly

---

## 3. Code Contracts with Other Developers

### Your Code vs Others' Code

- Assume your public API will be called by developers
  who haven't read your implementation
- Design APIs so that correct usage is obvious
  and incorrect usage fails fast

### Checks and Assertions

- **Precondition checks**: Validate inputs at public API boundaries
  and fail immediately with clear messages
- **Assertions**: Use for internal invariants that should never
  be violated — these signal bugs, not user errors
- Do NOT silently swallow invalid states —
  make violations visible

### Immutable Objects

- Prefer immutable objects by default —
  they eliminate accidental mutation bugs
- If mutability is needed, limit the scope of mutable state
- Immutable objects are inherently thread-safe
  and easier to reason about

---

## 4. Error Handling

### Recoverability

- Classify errors by whether the caller can reasonably recover:
  - **Recoverable**: invalid user input, network timeout,
    missing optional resource → signal to caller
  - **Unrecoverable**: programming bugs,
    corrupted state → fail fast

### Robustness vs Fail-Fast

- **Fail fast** for programming errors —
  hiding bugs causes worse failures later
- **Be robust** at system boundaries (user input, external APIs)
  — validate and handle gracefully
- Never silently ignore errors —
  either handle them or propagate them explicitly

### Error Signaling Strategy

| Strategy | When to use |
| --- | --- |
| Checked exception | Caller MUST handle this error |
| Unchecked exception | Programming bug, should not occur |
| Result/Either type | Functional error handling |
| Optional/nullable return | Absence is a normal outcome |

- Be consistent within a codebase —
  do not mix strategies arbitrarily
- Make error information specific enough
  to diagnose the problem

### Compiler Warnings

- Never ignore compiler warnings — treat them as errors
- Warnings often indicate latent bugs that will surface later

---

## 5. Readability

### Descriptive Naming

- Names should describe WHAT, not HOW
- Avoid abbreviations that aren't universally understood
- Use names that distinguish the purpose from similar concepts

### Comments

- ❌ Do NOT comment WHAT the code does —
  the code itself should be clear
- ✅ Comment WHY —
  the reasoning, constraints, or non-obvious decisions
- Remove outdated comments immediately —
  misleading comments are worse than none

### Avoid Deep Nesting

- Deep nesting (3+ levels) severely harms readability
- Use early returns, guard clauses, or extract functions
  to flatten structure
- Each nesting level adds cognitive load

### Eliminate Magic Numbers

- Replace literal values with named constants
- The name should explain the meaning, not just the value
- Configuration values should be externalized,
  not embedded in code

### Function Calls as Documentation

- Function and method calls should read like
  a description of the operation
- Avoid boolean parameters — they obscure intent
  at the call site. Prefer enums or separate methods

---

## 6. Predictable Code

### Minimize Side Effects

- Functions should primarily communicate
  through their return values
- Side effects (modifying external state, I/O, logging)
  should be explicit and documented
- Avoid functions that silently modify their arguments

### Null Safety

- Never return null when the caller expects a value —
  use Optional or throw
- Never accept null as a parameter
  unless explicitly documented
- Prefer the type system to enforce null safety
  over runtime checks

### Clear Inputs and Outputs

- Function signatures should make the data flow obvious
- Avoid using mutable shared state
  as an implicit input/output channel
- If a function needs many inputs,
  consider grouping them into a dedicated parameter object

### Beware of Unexpected Side Effects

- A getter should never modify state
- A validation method should never fix the data
- Name functions honestly —
  if it has side effects, the name must reflect them

---

## 7. Hard to Misuse

### Immutable by Default

- Make fields `final`/`val`/`readonly`
  unless mutation is explicitly required
- Return unmodifiable collections from public APIs
- Use copy-on-write or defensive copying
  when exposing internal state

### Use Enums Over Constants

- When a value belongs to a fixed set,
  use an enum — not string/int constants
- Enums enable exhaustive matching
  and prevent invalid values at compile time

### Access Restriction

- Expose the minimum visibility necessary
  (`private` > `internal` > `public`)
- Package-private/internal visibility is preferable
  to public for implementation classes
- Avoid making things public "just in case" —
  expanding visibility is easy, restricting it is hard

### Make Temporal Dependencies Explicit

- If operations must occur in a specific order,
  enforce it through the API design
  (e.g., builder pattern, state machine)
- Do not rely on documentation alone
  to communicate ordering requirements

---

## 8. Modularity

### Single Responsibility

- Each class/module should have exactly one reason to change
- If a class description requires "and",
  it likely has multiple responsibilities — split it

### Dependency Injection

- Do not hard-code dependencies —
  accept them through constructors
- DI improves testability, flexibility,
  and makes dependencies visible
- Combine static factory methods with constructor injection
  for production convenience

### Composition Over Inheritance

- Prefer composition (`has-a`) over inheritance (`is-a`)
- Inheritance creates tight coupling and fragile hierarchies
- Use inheritance only when there is a genuine
  "is-a" relationship AND the superclass
  is designed for extension

### Interface Segregation

- Clients should not depend on methods they don't use
- Split large interfaces into smaller, focused ones
- A class can implement multiple small interfaces

### Minimize Inter-Module Dependencies

- Depend on abstractions, not concrete implementations
- Keep the dependency graph shallow and acyclic
- If two modules are tightly coupled,
  consider merging them or extracting a shared abstraction

---

## 9. Reusability and Generalization

### Minimize Assumptions

- Do not assume the caller's context —
  design for the general case
- Avoid embedding business rules in utility code
- Parameterize behavior instead of hard-coding decisions

### Return Type Flexibility

- Return the most general type
  that provides the needed functionality
- Returning `List` instead of `ArrayList`
  allows implementation changes without breaking callers
- For collections, consider returning immutable views

### Global State

- Avoid global mutable state —
  it creates hidden dependencies and makes testing difficult
- If shared state is necessary,
  encapsulate it behind a clear interface
- Prefer explicit parameter passing over ambient context

### Keep Reuse Pragmatic

- Do not prematurely generalize —
  extract common code only when duplication actually occurs
- Three occurrences is a reasonable threshold
  for extraction (Rule of Three)
- Over-abstraction is as harmful as duplication
