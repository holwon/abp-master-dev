---
name: C# XML Documentation Comments
description: "Use when writing, editing, or reviewing C# code — enforces correct XML doc comments (///) on all public APIs, entities, services, and DTOs in ABP projects."
applyTo: "**/*.cs"
---

# C# XML Documentation Comments

Hard rules — enforced every time C# code is generated, edited, or reviewed.

1. **Coverage**: ALL public classes, interfaces, methods, properties, enums, and delegates MUST have `///` XML docs. `internal`/`private` members: docs optional, prefer readable names.
2. **Required Tags**:
   - `<summary>` — complete sentence starting with a verb, present tense, ending with a period. NEVER restate the member name.
   - `<param name="...">` — one per parameter; describe meaning, not type.
   - `<returns>` — meaning of the returned value; for async, the awaited result.
   - `<value>` — for properties: the meaning of the value the property represents (per Microsoft's recommended tags).
   - `<exception cref="...">` — for every caller-must-handle exception (e.g. `BusinessException`, `EntityNotFoundException`).
   - `<typeparam name="...">` — one per generic parameter.
   - `<remarks>` — invariants, side effects, thread-safety, or logic the summary can't capture.
3. **Formatting**:
   - Use multi-line block style — opening tag on its own line, content on the next, closing tag on its own line:
     ```csharp
     /// <summary>
     /// The content goes here.
     /// </summary>
     ```
     NEVER inline content on the tag line (`/// <summary>Content</summary>`).
   - `cref` MUST resolve to a real type/member (simple name, e.g. `<exception cref="BusinessException">`).
   - Reference members via `<see cref="..."/>`, parameters via `<paramref name="..."/>`, generic parameters via `<typeparamref name="..."/>`.
4. **Inheritance**: Use `<inheritdoc/>` on overrides/implementations unless base docs are wrong; document behavioral deviations in `<remarks>`.
5. **ABP / DDD Context**:
   - Entities/AggregateRoots: `<summary>` = domain concept; `<remarks>` = invariants; behavior methods document what they do AND when they throw.
   - Application Services: docs are the API contract — cover use-case, DTO meaning, and every `BusinessException` path.
   - DTOs: property docs are the OpenAPI surface — write for API consumers.
   - Repositories/Domain Services: document contracts (what is guaranteed), not implementations (how).
6. **FORBIDDEN**:
   - Empty or placeholder comments (`/// <summary></summary>`, `/// TODO`).
   - Documenting the obvious (e.g. `<summary>Adds two numbers.</summary>` on `int Add(int a, int b) => a + b;`).
   - Mixing English and Chinese within one comment (match surrounding code language).
