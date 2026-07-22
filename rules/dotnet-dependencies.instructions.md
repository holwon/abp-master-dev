---
name: .NET & ABP Dependency Rules
description: Rules for NuGet dependency management and official Volo.Abp module usage
applyTo: "**/*.cs, **/*.csproj"
---

# .NET & ABP Dependency Rules

1. **Exhaustive ABP Module Usage**: You MUST exhaustively use official `Volo.Abp.*` and `Microsoft.*` packages.
2. **Infrastructure Fallback**: When integrating infrastructure (e.g., Message Brokers like Kafka, Distributed Caches like Redis), you MUST use ABP's low-level integration modules (e.g., `Volo.Abp.Kafka`) to leverage ABP's built-in Options pattern and configuration management.
3. **Forbidden Direct Drivers**: It is STRICTLY FORBIDDEN to bypass ABP wrappers for raw third-party drivers (e.g., using `Confluent.Kafka` directly instead of `Volo.Abp.Kafka`).
4. **Third-Party Verification**: If proposing any non-standard third-party NuGet package, you MUST verify its exact name on nuget.org and explicitly justify why official ABP/Microsoft modules cannot satisfy the requirement.
