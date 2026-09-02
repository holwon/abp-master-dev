---
name: ABP Framework Architectural Standards & Skill Routing
description: "Mandatory skill-routing and coding standard rules for ABP Framework development. Automatically applies to all C# and project files."
applyTo: "**"
---

# ABP Architectural Standards & Skill Navigation

## 1. Mandatory Skill Loading Gate
Before writing, refactoring, or reviewing ANY C# code, you MUST use the file-reading tool to inspect the corresponding Skill document based on the file path or current task:

| Target File Path / Task Context | Mandatory Skill Path | Core Focus & Patterns |
| :--- | :--- | :--- |
| **New Feature / Complete CRUD Flow** | `skills/abp-development-flow/SKILL.md` | 11-step standard workflow, full lifecycle (Entity, DbContext, DTO, Service, Permissions, Tests) |
| `*.Domain/**/*.cs` (Entities, Aggregates, Value Objects, Domain Services) | `skills/abp-ddd/SKILL.md` | Aggregate boundaries, encapsulation, protected/internal constructors, BusinessException, domain events |
| `*.Domain.Shared/**/*.cs` or Core Config/DI/Time/Exceptions | `skills/abp-core/SKILL.md` | `IClock.Now`, DI marker interfaces (`ITransientDependency`), constants, localization |
| `*.Application.Contracts/**/*.cs` (DTOs, Service Interfaces) | `skills/abp-application-layer/SKILL.md` | DTO naming conventions, input isolation per method, data annotations |
| `*.Application/**/*.cs` (App Services, Object Mapping) | `skills/abp-application-layer/SKILL.md` | Explicit `UpdateAsync` calls, Mapperly/AutoMapper, prevent business logic leakage |
| `*.EntityFrameworkCore/**/*.cs` (DbContext, Repositories) | `skills/abp-ef-core/SKILL.md` | `b.ConfigureByConvention()`, repositories for aggregate roots only, `IncludeDetails` |
| Permissions / `*Permissions.cs` / `[Authorize]` | `skills/abp-authorization/SKILL.md` | Permission trees, definition providers, policy naming & checks |
| Module Classes `*Module.cs` / Dependencies `[DependsOn]` | `skills/abp-module/SKILL.md` | Module dependency graph, service configuration lifecycle |
| Unit / Integration Tests `*Tests/**/*.cs` | `skills/abp-testing/SKILL.md` | `AbpIntegratedTest` base classes, test data seeding, assertions |

## 2. Top-Level ABP Architectural Red Lines
1. **Aggregate & Entity Encapsulation**: MUST use `protected set` or `private set` on entity properties — NEVER `public set`. NEVER add EF navigation properties between different aggregate roots (reference by Guid ID only).
2. **Repository & Data Access Boundaries**: Create repositories ONLY for aggregate roots — NEVER create repositories for child/sub-entities. NEVER inject `DbContext` directly into Application or Domain layers; use `IRepository<T>` instead.
3. **Time & Infrastructure Abstractions**: MUST use `IClock.Now` / `Clock.Now` — NEVER use `DateTime.Now` or `DateTime.UtcNow`.
4. **EF Core Convention Mapping**: Entity configurations MUST call `b.ConfigureByConvention()` as the very first statement inside `builder.Entity<T>()`.
5. **Exceptions & Error Handling**: Business rule violations MUST throw `BusinessException` with namespaced error codes (e.g., `ModuleName:ErrorCode`) — NEVER throw generic `Exception`.
