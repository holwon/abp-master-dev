---
name: Cloud-Native & Kubernetes Concurrency Rules
description: Distributed systems, K8s multi-pod concurrency, statelessness, and locking rules
applyTo: "**/*.cs"
---

# Cloud-Native & Kubernetes Concurrency Rules

1. **Multi-Pod Execution Assumption**: Assume ALL state-mutating operations are executed concurrently across multiple K8s pods.
2. **Concurrency Control**: For state-mutating operations, active measures MUST be taken to prevent race conditions. Use ABP's `IDistributedLockProvider` or EF Core Optimistic Concurrency (`[ConcurrencyCheck]` / `ConcurrencyStamp`).
3. **Stateless Host**: `Http.Host` MUST be strictly stateless. Shared state MUST go to `IDistributedCache` (Redis), NEVER in-memory caches like `IMemoryCache`.
4. **Idempotency**: Handlers in `Event.Host` and jobs in `Background.Host` MUST be strictly idempotent to safely process duplicate messages/triggers.
5. **No Local In-Process Locks**: NEVER use in-process thread locking like `lock (object)` or `SemaphoreSlim` for cross-pod business logic, as they fail in multi-replica Kubernetes deployments.
