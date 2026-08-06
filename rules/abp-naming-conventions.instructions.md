---
name: Naming Conventions
description: "Use when creating or renaming any class, interface, or DTO — enforces naming conventions for services, repositories, entities, value objects, and DTOs."
applyTo: "**/*.cs"
---

# Naming Conventions

Hard rules — enforced every time a class, interface, or DTO is created or renamed.

1. **Application Services**: class `{Entity}AppService`, interface `I{Entity}AppService`. E.g. `BookAppService` / `IBookAppService`.
2. **Domain Services**: class `{Entity}Manager`, interface `I{Entity}Manager`. E.g. `BookManager` / `IBookManager`.
3. **Repositories**: interface `I{Entity}Repository`; EF Core implementation `EfCore{Entity}Repository`. E.g. `IBookRepository` / `EfCoreBookRepository`.
4. **Entities / AggregateRoots**: plain domain name, NO suffix. E.g. `Book`, `Order` — never `BookEntity`.
5. **Value Objects**: plain domain concept name. E.g. `Address`, `Money` — no `Vo` suffix unless required to disambiguate.
6. **DTOs** (ABP conventions):
   - Read model: `{Entity}Dto` — e.g. `BookDto`.
   - Create/Update: `CreateUpdate{Entity}Dto` — e.g. `CreateUpdateBookDto` (covers both create and update).
   - List input: `Get{Entity}ListInput` — e.g. `GetBookListInput`.
   - If create and update differ, split into `Create{Entity}Dto` / `Update{Entity}Dto`.
7. **Controllers**: class `{Entity}Controller`.
8. **Enums**: plain concept name. E.g. `BookType`, `OrderStatus` — no `Enum` suffix.
9. **Constants & error codes**: use ABP pattern `{Entity}ErrorCodes` static class with PascalCase members, e.g. `OrderErrorCodes.CannotCancelCompletedOrder`.
10. **FORBIDDEN**:
    - Suffix collisions: one suffix per aggregate per layer. `BookManager` (Domain) and `BookAppService` (Application) coexist by design; the smell is a second suffix for the same aggregate in the same layer (e.g. `BookManager` + `BookDomainService` both in Domain) — pick the one that names the layer's concern.
    - Generic names (`Manager`, `Service`, `Helper`, `Data`) without an entity prefix.
    - Abbreviations or acronyms for entity names (e.g. `Bk`, `Ord`) — always the full domain term.
