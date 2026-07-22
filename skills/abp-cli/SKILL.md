---
name: abp-cli
description: ABP CLI commands - generate-proxy, install-libs, add-package-ref, new-module, install-module, abp update, abp clean, abp suite generate. Use when the user asks how to run ABP CLI commands, generate proxies, install libraries, or use ABP Suite.
user-invocable: false
---

# ABP CLI Commands

ABP CLI is the command-line tool for ABP Framework. Use `abp help [command]` for detailed options.

## Generate Client Proxies

Generate typed HTTP client proxies from running backend:

```bash
# Angular proxies (host must be running)
abp generate-proxy -t ng

# C# client proxies (for HttpApi.Client projects)
abp generate-proxy -t csharp -u https://localhost:44300

# Integration services only (microservice inter-service communication)
abp generate-proxy -t csharp -u https://localhost:44300 -st integration

# JavaScript proxies
abp generate-proxy -t js -u https://localhost:44300

# With specific module filter
abp generate-proxy -t csharp -u https://localhost:44300 -m identity

# Without generating contracts (for microservices)
abp generate-proxy -t csharp -u https://localhost:44300 --without-contracts
```

| Flag | Short | Description |
|------|-------|-------------|
| `--type` | `-t` | Proxy type: `ng`, `csharp`, `js` |
| `--url` | `-u` | Backend API URL (must be running) |
| `--service-type` | `-st` | Service type filter: `application`, `integration` |
| `--module` | `-m` | Module name filter |
| `--without-contracts` | | Skip generating contracts (microservices) |

## Install Client-Side Libraries

```bash
# Install NPM packages for MVC/Blazor Server (Bootstrap, jQuery, etc.)
abp install-libs
```

This downloads and installs client-side libraries defined in `abp.resourcemapping.js`.

## Add Package Reference

Add project/module reference with automatic module dependency configuration:

```bash
# Add reference to a domain project
abp add-package-ref Acme.BookStore.Domain

# Add reference with target project specified
abp add-package-ref Acme.BookStore.Domain -t Acme.BookStore.Application
```

## Module Operations

```bash
# Create a new module in the current solution
abp new-module Acme.OrderManagement -t module:ddd

# Install a published ABP module (e.g., Blogging, CMS Kit)
abp install-module Volo.Blogging

# Add an ABP NuGet package to the solution
abp add-package Volo.Abp.Caching.StackExchangeRedis
```

## Update & Clean

```bash
# Update all ABP packages to latest version
abp update

# Update to a specific version
abp update --version 8.0.0

# Preview update without applying
abp update --dry-run

# Clean all bin/obj folders in the solution
abp clean
```

## ABP Suite (CRUD Generation)

Generate full CRUD pages from entity definitions:

```bash
# Generate CRUD from entity JSON file
abp suite generate --entity .suite/entities/Book.json --solution ./Acme.BookStore.sln
```

Entity JSON files are created via ABP Suite UI and stored in `.suite/entities/`. The generate command creates:
- Entity class
- DTOs
- Application service interface + implementation
- EF Core configuration + migration
- UI pages (Angular/Blazor/MVC depending on template)
- Localization entries
- Permission definitions

## Quick Reference

| Task | Command |
|------|---------|
| Angular proxies | `abp generate-proxy -t ng` |
| C# proxies | `abp generate-proxy -t csharp -u https://localhost:44300` |
| Integration proxies | `abp generate-proxy -t csharp -u https://localhost:44300 -st integration` |
| JS proxies | `abp generate-proxy -t js -u https://localhost:44300` |
| Install libs | `abp install-libs` |
| Add project ref | `abp add-package-ref <project>` |
| New module | `abp new-module <name> -t module:ddd` |
| Install module | `abp install-module <module-name>` |
| Add NuGet package | `abp add-package <package-name>` |
| Update packages | `abp update` |
| Update to version | `abp update --version 8.0.0` |
| Clean solution | `abp clean` |
| Suite generate | `abp suite generate --entity <json> --solution <sln>` |
| Help | `abp help [command]` |

## Common Workflows

### After Adding a New Application Service

```bash
# 1. Start the backend
cd src/MyProject.HttpApi.Host
dotnet run

# 2. In another terminal, generate Angular proxies
cd angular/
abp generate-proxy -t ng
```

### After Adding a New Microservice Integration

```bash
# Generate C# proxies for integration services
abp generate-proxy -t csharp -u https://localhost:44361 -m catalog -st integration --without-contracts
```

### Updating ABP Version

```bash
# Check current version in Directory.Packages.props first
# Then update:
abp update --version 8.1.0
# Review changes, then:
abp install-libs  # Update client-side libs if needed
```
| C# proxies | `abp generate-proxy -t csharp -u URL` |
| Install JS libs | `abp install-libs` |
| Add reference | `abp add-package-ref PackageName` |
| Create module | `abp new-module ModuleName` |
| Install module | `abp install-module ModuleName` |
| Update packages | `abp update` |
| Clean solution | `abp clean` |
| Suite CRUD | `abp suite generate -e entity.json -s solution.sln` |
| Get help | `abp help [command]` |
