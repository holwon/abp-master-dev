---
name: ABP DDD & Architectural Constraints
description: DDD encapsulation, entity modeling, and forbidden code patterns in ABP Framework
applyTo: "**/*.cs"
---

# ABP DDD & Architectural Constraints

1. **Avoid Property Duplication**: DO NOT declare properties (like `Id`, `IsDeleted`, `CreationTime`, or `ConcurrencyStamp`) that are already provided by inherited ABP base classes (`AggregateRoot<TKey>`, `FullAuditedAggregateRoot<TKey>`, `Entity<TKey>`, etc.).
2. **Encapsulation**: Custom `AggregateRoot` and `Entity` properties MUST ALWAYS use `protected set` or `private set`. NEVER use `public set`.
3. **Aggregate Boundaries**: Sub-entities (`Entity`) MUST restrict direct instantiation from outside the aggregate (use `internal` constructors).
4. **State Mutation**: Entity state must only be modified via constructors or dedicated domain methods.
5. **Forbidden Patterns**:
   - NEVER use AutoMapper or Mapster inside Domain Layers; use manual object mapping or dedicated domain mappers.
   - NEVER execute async database/network calls inside `for`/`foreach` loops. MUST use `Task.WhenAll` or batch query APIs.
   - NEVER put business logic in API Controllers or DbContext. Controllers strictly delegate to Application Services or Mediator handlers.
