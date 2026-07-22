---
name: ABP DDD & Architectural Constraints
description: Hard rules for DDD encapsulation, entity modeling, and forbidden code patterns in ABP Framework
applyTo: "**/*.cs"
---

# ABP DDD & Architectural Constraints

Hard rules — enforced every time you write or review C# code.

1. **MUST use `protected set` or `private set` on AggregateRoot/Entity properties**. NEVER use `public set`.
2. **NEVER use AutoMapper/Mapster inside Domain Layer**. Manual mapping only.
3. **NEVER execute async DB/network calls inside `for`/`foreach` loops**. MUST use `Task.WhenAll` or batch query APIs.
4. **NEVER put business logic in API Controllers or DbContext**. MUST delegate to Application Services.
5. **Sub-entities MUST use `internal` constructors**. No direct instantiation from outside the aggregate.

> For complete DDD patterns, code examples, and best practices, see the `abp-ddd` skill (`skills/abp-ddd/SKILL.md`).
