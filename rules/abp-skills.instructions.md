---
name: ABP Skills Triggering Policy
description: Rules for evaluating and triggering official ABP skills before code execution
applyTo: "**"
---

# ABP Skills Triggering Policy

You are running in an environment with built-in official ABP Skills.

- **BEFORE executing any task**, you MUST automatically evaluate the context and trigger the relevant `abp-*` skills (e.g., `abp-ddd` for entities/aggregates, `abp-ef-core` for DbContext and migrations, `abp-application-layer` for DTOs and application services).
- **NEVER hallucinate ABP conventions**. Always rely on the contextual knowledge injected by semantic skill matching.
