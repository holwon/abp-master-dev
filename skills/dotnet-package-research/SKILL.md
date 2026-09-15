---
name: dotnet-package-research
description: .NET package research - step-by-step ladder for researching unfamiliar NuGet packages and third-party libraries. Use when you need to understand a new/replacement NuGet package or Volo.Abp module, check versions, dependencies, documentation, or source, or decide whether decompilation is justified. Trigger words: NuGet, .NET package, Volo.Abp, package research, IL decompile, ilspycmd, dnSpy.
user-invocable: false
---

# .NET Package Research Ladder

Step-by-step procedure for researching an unfamiliar NuGet package, library, or third-party API. Pairs with the always-on constraint rule `../rules/package-research.instructions.md` (never decompile first).

## When to use

You hit an unfamiliar package, library, or API in a .NET project and need to understand what it does, its dependencies, its versions, its docs, or how it is meant to be used.

## Overview: the 4-rung ladder

Exhaust each rung completely before moving to the next. Decompilation is NOT a rung on the ladder — it sits below it as an escape hatch.

1. **Package metadata** — `com.microsoft/nuget/*` tools
2. **Official documentation** — XML docs, README, Microsoft Learn, abp.io/docs, vendor docs
3. **Source code** — the real repository and its tests
4. **`@WebResearcher` delegation** — community usage, GitHub issues

## Detailed steps

### Rung 1: Package metadata (fast, authoritative)
- Use the `com.microsoft/nuget/*` tools to get exact identity, versions, dependency tree, project/repository URL, deprecation and vulnerability data.
- If that answers the question (what it is, what it depends on, whether it is deprecated or vulnerable), stop here.

### Rung 2: Official documentation
- Read the package's XML doc comments and README.
- Look up Microsoft Learn, abp.io/docs, and vendor documentation.

### Rung 3: Source code
- Open the repository URL from Rung 1 (Volo.Abp and most .NET packages are open source).
- Read the real implementation and its tests — never decompiled or IDE "metadata source".

### Rung 4: @WebResearcher
- Delegate to `@WebResearcher` for community articles, GitHub issues, and real-world usage patterns.
- Require links in the returned summary.

## Escape hatch: decompilation (pre-approved, last resort only)

The user has already approved decompilation as a fallback — do NOT ask permission again. Decompile ONLY when:

- The ladder above was walked and could not answer the question, OR
- The package is closed-source with no usable documentation or source.

When you decompile:

- Say so explicitly and note which ladder rungs already failed — never decompile silently.
- Treat decompiled output as a hint, not truth — flag any conclusion that lacks documentation or source confirmation.

## Quality / completion check

- You can name what the package does and its real use case from a trusted source (metadata / docs / source): done.
- You decompiled as a shortcut or as the first move: that is a violation — go back to the ladder.