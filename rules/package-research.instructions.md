---
name: .NET Package Research Ladder (constraint)
description: "Hard constraint — when encountering an unfamiliar NuGet package, library, or third-party API, always follow the research ladder and never decompile first. Full procedure lives in the dotnet-package-research skill."
applyTo: "**/*.cs, **/*.csproj"
---

# .NET Package Research (hard constraint)

When you encounter an unfamiliar NuGet package, library, or third-party API in a .NET project:

- **NEVER decompile first.** The full research procedure lives in the `dotnet-package-research` skill (`skills/dotnet-package-research/SKILL.md`) and is loaded automatically on task match.
- Follow that skill's 4-rung ladder: `com.microsoft/nuget/*` metadata → official docs → source repository → `@WebResearcher`.
- Decompilation is pre-approved by the user but only as a last resort — after the ladder is walked or the package is closed-source with no usable docs/source. Never silently; treat output as a hint, not truth.
...