# Agent Guide: Creating a Distributed .NET Application Solution

> **Purpose**: Step-by-step instructions for an AI agent to scaffold a distributed .NET application using **Aspire** as the orchestrator and either **Orleans** (virtual-actor model) or **Akka.NET** (classic-actor model) as the distributed compute framework — with multi-process hosts, grain or actor contracts, optional gateway, and full CI/CD from commit one.
> **Inputs required from user before starting**: application name, actor framework (Orleans or Akka.NET), domain grain or actor topology, storage backend, whether a TCP/gRPC gateway is needed.
> **Output**: a complete, CI-ready, runnable distributed system repository scaffold with layered architecture, Aspire orchestration, and zero-tolerance code quality from commit one.
> **Standalone guarantee**: every shared repository-root template and every distributed-system-specific scaffold step needed to complete the repository appears in this file.

---

## Scope

This guide targets a **multi-process distributed system** where multiple service hosts run as separate processes, business logic lives in grains or actors rather than plain classes, Aspire provides orchestration across all hosts, and tests use an in-process cluster or actor harness. It includes the shared repository-root baseline directly in Phase 1, then layers distributed-system structure, hosting, CI, and validation guidance on top.

**Replaced entirely in this guide:**

- Solution structure — `src/host/`, `src/grain/` (or `src/actors/`), `src/common/`, `src/protocol/`, `test/`
- CI pipeline — builds and integration-tests multiple service hosts; no single-file publish
- `Build.cs` — adds `run-local` (starts Aspire AppHost), `run-silo`, no single-exe publish
- `Directory.Build.props` hierarchy — host layer, grain layer, test layer each with different defaults
- Test strategy — Orleans `TestCluster` or Akka.NET `TestKit` + Aspire integration tests

**Not applicable** (single-process and library concerns removed):

- NuGet packaging, MinVer, PublicApiGenerator, ReadMeTest README-sync
- Single-file publish, SelfContained, PublishSingleFile
- Solution filters by domain layer (filters are by host role instead)

---

## Phase 0: Gather Inputs

Ask the user for (or infer from context):

| Variable              | Example                                                      | Notes                                                                                           |
| --------------------- | ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| `APPNAME`             | `MySystem`                                                   | PascalCase; becomes solution name, namespace prefix, and Docker image base name                 |
| `ROOTNS`              | `MySystem`                                                   | Root namespace prefix; all projects use `<ROOTNS>.<Layer>.<Module>`                             |
| `DESCRIPTION`         | `A distributed order-processing system.`                     | One sentence for README                                                                         |
| `AUTHOR`              | `yourname`                                                   | GitHub username                                                                                 |
| `DOTNET_VERSION`      | `net10.0`                                                    | Target framework for all projects                                                               |
| `CSHARP_VERSION`      | `14.0`                                                       | Match to chosen .NET version                                                                    |
| `SDK_VERSION`         | `10.0.103`                                                   | Exact SDK version from `dotnet --version`                                                       |
| `GITHUB_URL`          | `https://github.com/you/APPNAME`                             | Full repo URL                                                                                   |
| `LICENSE`             | `MIT`                                                        | License type                                                                                    |
| `YEAR`                | `2026`                                                       | Copyright year                                                                                  |
| `COMPANY`             | `yourname`                                                   | Used in copyright                                                                               |
| `ACTOR_FRAMEWORK`     | `Orleans`                                                    | `Orleans` or `Akka.NET`                                                                         |
| `ORLEANS_VERSION`     | `10.0.1`                                                     | Required if `ACTOR_FRAMEWORK=Orleans`                                                           |
| `ASPIRE_VERSION`      | `13.2`                                                       | `Aspire.Hosting` meta-package version                                                           |
| `USE_GATEWAY`         | `true`                                                       | true if the system needs a TCP/gRPC/HTTP gateway for external client connections                |
| `GATEWAY_PROTOCOL`    | `TCP`                                                        | `TCP` (DotNetty), `gRPC`, `HTTP`, or `WebSocket`; only if `USE_GATEWAY=true`                   |
| `STORAGE_BACKEND`     | `InMemory`                                                   | `InMemory` / `PostgreSQL` / `MongoDB` / `AzureBlob` — drives which storage NuGet packages appear |
| `NEEDS_UNSAFE`        | `false`                                                      | true for P/Invoke, SIMD, crypto, or gateway packet manipulation                                 |
| `GRAIN_TOPOLOGY`      | (see Phase 4 interview)                                      | Ordered list of grains/actors and their key types                                               |
| `COMMON_LAYERS`       | `Common, Common.Persistence`                                 | Shared library layers under `src/common/` — drives project stubs                               |

### Grain / Actor Topology Interview

The most important input is the **grain (Orleans) or actor (Akka.NET) topology**. Ask the user to enumerate the major entities:

**Orleans example:**

```
IGrainWithStringKey grains  → ISessionGrain   "session:{id}"
                             IChannelGrain    "world:{worldId}:channel:{channelId}"
IGrainWithIntegerKey grains → IPlayerGrain    characterId (long)
                             IAccountGrain   accountId (int)
IGrainWithGuidKey grains   → IPartyGrain     partyId (Guid)
                             IGuildGrain     guildId (Guid)
```

**Akka.NET example:**

```
Stateful actors  → OrderActor     orderId (string)
                   InventoryActor productId (string)
Stateless actors → PricingActor   (singleton)
Routed pools    → NotificationActor (pool of 4)
```

Record the final list as `GRAIN_TOPOLOGY` — it drives `GrainInterfaces/` (Orleans) or `Messages/` + `Actors/` (Akka.NET) project content.

### Persistence Backend Interview

Grain / actor persistence drives which storage packages are needed:

| Backend     | Orleans package                          | Akka.NET package              |
| ----------- | ---------------------------------------- | ----------------------------- |
| In-memory   | `Microsoft.Orleans.Core` (built-in)      | `Akka.Persistence` (built-in) |
| PostgreSQL  | `Orleans.Persistence.AdoNet` + Npgsql    | `Akka.Persistence.PostgreSql` |
| MongoDB     | `Orleans.Persistence.MongoDB`            | `Akka.Persistence.MongoDb`    |
| Azure Blob  | `Orleans.Persistence.AzureStorage`       | —                             |
| SQL Server  | `Orleans.Persistence.AdoNet` + SqlClient | `Akka.Persistence.SqlServer`  |

> **Agent note**: The `STORAGE_BACKEND` variable drives which `<PackageVersion>` entries appear in `Directory.Packages.props`. Always add the packages; delete the ones not needed after user confirms the choice.

---

## Phase 0b: Initialize Git Repository

Before creating any files, initialize the repository:

```shell
mkdir <APPNAME> && cd <APPNAME>
git init -b main
```

**Verify git authorship** before the first commit:

```shell
git config user.name   # must print the real name, not a placeholder
git config user.email  # must print the real email / GitHub-linked address
```

If the output is wrong (for example it shows a machine hostname, `bia10@github.com`, or is empty), fix it before committing:

```shell
# Fix locally (this repo only):
git config user.name  "Your Name"
git config user.email "you@example.com"

# Or remove a local override and fall back to global:
git config --unset user.name
git config --unset user.email
```

> **Agent note**: Do not set or override `user.name`/`user.email` yourself. Flag any mismatch to the user.

**Run CSharpier before the first commit.** After all source files are created but before any `git add`:

```shell
dotnet tool restore
dotnet csharpier format .
dotnet format style
dotnet format analyzers
```

### Branch and PR workflow

Create a scoped branch before adding source changes:

```shell
git checkout -b feature/<scope>-<summary>
# or: fix/<scope>-<summary>, docs/<scope>-<summary>, refactor/<scope>-<summary>,
#     test/<scope>-<summary>, chore/<scope>-<summary>, perf/<scope>-<summary>,
#     research/<scope>-<summary>
```

Use lowercase prefixes. When the work is tied to a GitHub issue, include the issue number in the branch name, for example `feature/issue-123-add-aspire-gateway-host`.

After the first green push, open a **draft** pull request, link the related issue in the PR body when one exists (`Fixes #123`), and keep review, verification, and remediation commits on that same branch until the PR is ready for human merge review.

### Recommended commit strategy

```shell
# Commit 1: Root config files (Phase 1)
git add .editorconfig .csharpierrc.json .gitattributes .gitignore .jscpd.json .markdownlint.json \
    .pre-commit-config.yaml .config/ global.json nuget.config \
    LICENSE SECURITY.md CONTRIBUTING.md AGENTS.md CHANGELOG.md \
    .github/ Directory.Build.props Directory.Packages.props README.md <APPNAME>.slnx
git commit -m "chore(repo): add root config and repository health files"

# Commit 2: Project structure
git add src/ test/ Build.cs
git commit -m "chore(scaffold): add distributed project structure and Build.cs task runner"

# Commit 3: CI pipeline
git add .github/workflows/ .github/dependabot.yml
git commit -m "ci(github): add Actions workflow and Dependabot"
```

---

## Phase 1: Repository Root Files

Create the shared repository-root files below first. This guide is standalone, so do not rely on any external guide or companion document for the baseline root files. After the shared baseline is in place, apply the distributed-system-specific `a` variants that follow.

### 1.1 `global.json`

```json
{
  "sdk": {
    "version": "<SDK_VERSION>",
    "rollForward": "latestPatch",
    "allowPrerelease": false
  }
}
```

### 1.2 `nuget.config`

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
  </packageSources>
</configuration>
```

### 1.3 `.gitignore`

Generate the base file:

```shell
dotnet new gitignore
```

Then append these shared entries:

```gitignore
# Centralized build output (UseArtifactsOutput)
artifacts/

# BenchmarkDotNet local output
BenchmarkDotNet.Artifacts/

# Benchmark results (committed selectively by the author, not auto-generated)
# benchmarks/ is NOT ignored — results are committed intentionally
```

### 1.4 `.gitattributes`

```gitattributes
* text=auto eol=lf
*.cs text eol=lf
*.csproj text eol=lf
*.props text eol=lf
*.targets text eol=lf
*.md text eol=lf
*.yml text eol=lf
*.json text eol=lf
```

### 1.5 `.editorconfig`

```editorconfig
root = true

[*]
indent_style = space

[*.{csproj,props,targets,config,nuspec}]
indent_size = 2

[*.cs]
indent_size = 4
insert_final_newline = true
end_of_line = lf
charset = utf-8-bom

[Build.cs]
charset = utf-8

[*.{yml,yaml}]
charset = utf-8
insert_final_newline = true
indent_size = 2
end_of_line = lf

[*.json]
indent_size = 2
insert_final_newline = true

dotnet_sort_system_directives_first = true
dotnet_style_require_accessibility_modifiers = omit_if_default
csharp_style_namespace_declarations = file_scoped:warning
csharp_style_throw_expression = true:error
dotnet_diagnostic.IDE0055.severity = none
dotnet_diagnostic.IDE0005.severity = error
csharp_new_line_before_open_brace = accessors, lambdas, types, methods, properties, control_blocks

dotnet_naming_rule.private_fields.symbols = private_fields
dotnet_naming_rule.private_fields.style = _camelcase
dotnet_naming_rule.private_fields.severity = suggestion
dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private, protected
dotnet_naming_style._camelcase.required_prefix = _
dotnet_naming_style._camelcase.capitalization = camel_case

dotnet_naming_rule.private_static_readonly.symbols = private_static_readonly
dotnet_naming_rule.private_static_readonly.style = s_camelcase
dotnet_naming_rule.private_static_readonly.severity = suggestion
dotnet_naming_symbols.private_static_readonly.applicable_kinds = field
dotnet_naming_symbols.private_static_readonly.applicable_accessibilities = private, protected
dotnet_naming_symbols.private_static_readonly.required_modifiers = static, readonly
dotnet_naming_style.s_camelcase.required_prefix = s_
dotnet_naming_style.s_camelcase.capitalization = camel_case

dotnet_diagnostic.CA1707.severity = none
dotnet_diagnostic.CS8981.severity = none
dotnet_diagnostic.CA1805.severity = none
dotnet_diagnostic.MA0008.severity = none
dotnet_diagnostic.MA0051.severity = none
```

### 1.6 `.jscpd.json`

```json
{
  "threshold": 5,
  "reporters": ["console"],
  "ignore": ["**/__mocks__/**", "**/node_modules/**", "**/artifacts/**"],
  "absolute": true
}
```

### 1.7 `.markdownlint.json`

```json
{
  "default": true,
  "MD013": false,
  "MD033": false,
  "MD041": false
}
```

### 1.8 `.pre-commit-config.yaml`

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: <LATEST_GITLEAKS_TAG>
    hooks:
      - id: gitleaks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: <LATEST_PRECOMMIT_HOOKS_TAG>
    hooks:
      - id: end-of-file-fixer
      - id: trailing-whitespace
  - repo: local
    hooks:
      - id: csharpier
        name: CSharpier
        language: system
        entry: dotnet csharpier check
        types: [c#]
```

Install `pre-commit` via `pip install pre-commit` or `pipx install pre-commit`. If Python is not available, skip this file and rely on CI plus the format checks.

### 1.9 `codecov.yml` _(Optional)_

```yaml
coverage:
  status:
    project:
      default:
        target: auto
        threshold: 0%
    patch:
      default:
        target: auto
        threshold: 0%
```

### 1.11 `LICENSE`

Use the full MIT license text with `Copyright (c) <YEAR> <COMPANY>`.

### 1.12 `SECURITY.md`

```markdown
# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability, please report it via
[GitHub's private vulnerability reporting](https://github.com/<AUTHOR>/<APPNAME>/security/advisories/new).

Do **not** open a public issue for security vulnerabilities.
```

### 1.13 `CONTRIBUTING.md`

````markdown
# Contributing to <APPNAME>

Thank you for considering contributing.

## How to Contribute

1. Create a scoped branch from `main`, for example `feature/<scope>-<summary>` or `fix/<scope>-<summary>`.
2. Run `dotnet run Build.cs format` to restore local tools (CSharpier + dotnet-coverage) and normalize formatting.
3. Ensure the repository passes:

   ```shell
   dotnet run Build.cs build Release
   dotnet run Build.cs -- test -c Release
   dotnet run Build.cs format check
   ```

5. Push the branch and open a draft pull request against `main`.

## Reporting Issues

Use GitHub Issues for bugs and feature requests.
````

### 1.13a `AGENTS.md`

## Optional parameters and nullability

1. **Optional ≠ nullable.** `param = default` means callers may omit it; `T?` means the value may be null. Do not conflate the two contracts.

2. **Do not add `= null` to avoid updating call sites.** If the dependency/value is actually required, make callers pass it and fix every compile error.

3. **Nullable annotations are API contracts.** Use `T` for required non-null values, `T?` only when null is explicitly supported and tested.

4. **Every nullable parameter creates a handling obligation.** If `ILogger? logger = null`, every implementation path must safely handle `null`; otherwise the API is lying.

5. **Prefer overloads for convenience.** Use `DoThing()` delegating to `DoThing(requiredLogger)` or a well-defined default, instead of spreading null checks through the code.

6. **Prefer Null Object/default implementations over null.** Example: use `NullLogger<T>.Instance` internally when “no logging” is a valid behavior.

7. **Do not make services/dependencies optional by default.** Constructor-injected dependencies, loggers, clients, repositories, clocks, etc. should usually be required.

8. **Optional parameter defaults are versioning hazards.** In C#, default values are baked into call sites; changing the default may not affect already-compiled callers.

9. **Only use optional parameters for stable, obvious defaults.** Good examples: flags, timeouts, `CancellationToken cancellationToken = default`; bad examples: hidden dependencies or required business data.

10. **When adding a parameter, decide deliberately:** required → update all call sites; truly optional → define exact default behavior; nullable → document and test null behavior.

Create this as a standalone repository-root file using the full template below, or reuse the user's existing stronger house-standard `AGENTS.md` when one already exists (for example an ArcNET-style contract). It is a first-class repo contract, not part of `CONTRIBUTING.md`, issue reporting, or `.github/ISSUE_TEMPLATE` content. Do **not** replace it with a short placeholder note about repo layout or common commands:

````markdown
# **🛠 Universal AI Agent Standards & Repository Health**

This document serves as the mandatory operational framework for all AI Agents (GitHub Copilot, Cursor, Codex, etc.) interacting with this repository. Adherence to these standards is required for all generated code, tests, scripts, and commits.

## **1\. Git Commit Standards: Conventional Commits**

To maintain a clean, machine-readable, and professional history, you must strictly adhere to the [Conventional Commits](https://www.conventionalcommits.org/) specification.

* **Format:** \<type\>(\<scope\>): \<description\>
* **The Imperative Mood:** Always use the imperative, present tense. Use "Add feature" instead of "Added feature" or "Adds feature."
* **Case Sensitivity:** The type and scope must be strictly **lowercase**.
* **Granularity:** If a task involves both a refactor and a new feature, you are required to split them into two distinct commits.
* **Scope:** This is mandatory and must represent the specific module or component affected (e.g., repo, agents, routing, github).
* **Authors:** Ensure that commits are authored by the authenticated user only; do not add co-authors.
* **Branch Names:** Agent-created branches must use lowercase prefixes such as `feature/`, `fix/`, `chore/`, `docs/`, `refactor/`, `test/`, `perf/`, or `research/`.
* **Issue Branches:** When a GitHub issue number exists, include it in the branch name, for example `feature/issue-123-add-oauth-provider`.
* **PR Re-iteration:** Review, verification, and remediation passes must stay on the current PR head branch. Do not create a second branch for the same PR.

| Type | Use Case | Example |
| :---- | :---- | :---- |
| **feat** | A new feature for the user. | feat(auth): add OAuth2 provider |
| **fix** | A bug fix for the user. | fix(api): resolve null pointer in user-lookup |
| **docs** | Documentation-only changes. | docs(automation): explain assigned issue prerequisites |
| **refactor** | Code change that neither fixes a bug nor adds a feature. | refactor(db): flatten repository hierarchy |
| **test** | Adding missing tests or correcting existing tests. | test(vault): add boundary checks for encryption |
| **chore** | Updating build tasks, package manager configs, etc. | chore(deps): bump Newtonsoft.Json to 13.0.3 |
| **ci** | CI workflow or automation pipeline changes. | ci(github): add release workflow validation |
| **perf** | Performance improvements. | perf(parser): reduce tokenization allocations |
| **tool** | Automation scripts or internal dev-tooling. | tool(automation): add C\# script for log rotation |

## **2\. Testing: The "Anti-Pollution" Mandate**

We prioritize **quality and logical failure paths** over coverage metrics. You are forbidden from generating "shallow" or "ritualistic" tests.

* **Ban on Mock-Only Tests:** Do not write tests that only verify if a mock was called (e.g., \_mock.Verify(x \=\> x.Save(), Times.Once)). This tests implementation details (how the code is written), not business behavior (what the code does).
* **The "Mutation" Requirement:** Every test must be designed so that if the underlying logic is changed or deleted, the test **fails**. If a test passes after the logic it is supposedly testing is removed because everything is mocked, the test is pollution and must be deleted.
* **Behavioral Focus:** Focus on state changes, return values, and edge cases. If a method is a simple "Pass-Through" (calling another service with no internal logic), **do not unit test it.**
* **Dependency Limit:** If a unit test requires more than 3 Mock\<T\> objects, the code is too highly coupled. Stop and suggest a refactor or write an **Integration Test** instead.

## **3\. TUnit: Microsoft Testing Platform Rules**

This repository uses **TUnit on Microsoft.Testing.Platform (MTP)**, not VSTest. Agents must use TUnit-compatible filters, timeouts, and hang-debugging options.

* **Never use VSTest filters:** Do not run `dotnet test --filter ...` with TUnit. Use `--treenode-filter` instead. If platform args are rejected, pass them after `--`.
* **Filter shape:** TUnit filters use `/Assembly/Namespace/Class/TestName[Property=Value]`. Common patterns: class `/*/*/ClassName/*`, method `/*/*/*/MethodName`, category `/*/*/*/*[Category=Integration]`, exclude category `/*/*/*/*[Category!=Slow]`, and combined properties `/*/*/*/*[(Category=Smoke)&(Priority=High)]`.
* **Tag tests explicitly:** Prefer `[Property("Category", "Integration")]`, `[Property("Priority", "High")]`, and other explicit metadata so filters stay stable and self-documenting.
* **Timeout defaults are mandatory:** Add an assembly-level `[assembly: Timeout(...)]` and override slower classes or methods with more specific `[Timeout(...)]` attributes.
* **Cancellation is mandatory for async and IO tests:** Every async, integration, or polling-heavy test must accept `CancellationToken cancellationToken` and pass it to HTTP calls, DB calls, delays, process waits, and application startup/shutdown.
* **No sync-over-async or uncancellable waits:** Do not use `.Wait()`, `.Result`, `.GetAwaiter().GetResult()`, uncancellable `Task.Delay(...)`, or polling loops that ignore the token.
* **Parallelism can look like hangs:** TUnit runs tests in parallel by default. If a full-suite run hangs but a filtered run passes, suspect shared ports, shared databases, static locks/state, or non-thread-safe fixtures. Use `[NotInParallel]`, `[ParallelGroup]`, `[ParallelLimiter<T>]`, or lower global parallelism.
* **Use MTP run-level watchdogs for CI:** Prefer `dotnet test -c Release --timeout 10m` for suite timeouts, or `dotnet test -c Release -- --timeout 10m` when the SDK requires explicit forwarding.
* **Use hang dumps for true deadlocks:** When cooperative cancellation is not enough, add `Microsoft.Testing.Extensions.HangDump` and use `--hangdump --hangdump-timeout <duration>` instead of VSTest-only `--blame-hang`.

```text
TUnit quick rules:
- Preferred filter run: dotnet test -c Release --treenode-filter "<filter>"
- Forwarded filter run: dotnet test -c Release -- --treenode-filter "<filter>"
- Preferred timeout run: dotnet test -c Release --timeout 10m
- Filter shape: /Assembly/Namespace/Class/TestName[Property=Value]
```

## **4\. Automation: C\# 10+ File-Based Apps**

**Bash and PowerShell are deprecated in this repository.** All automation, maintenance, and tooling must be written as **C\# 10 File-Based Apps**.

* **Standalone Execution:** Use the single-file format that runs via dotnet run \<filename\>.cs.
* **NuGet Integration:** Use the \#:package directive at the top of the file to manage dependencies.
* **No Boilerplate:** Do not use namespace, class Program, or static void Main. Write logic directly using Top-Level Statements.
* **Portability:** Use Path.Combine or forward slashes. Scripts must be execution-ready on Windows, macOS, and Linux without modification.
* **Option Forwarding:** When invoking a file-based app and forwarding option-style args to the script, insert `--` after the script file. *Correct:* `dotnet run Build.cs -- test -c Release`. *Incorrect:* `dotnet run Build.cs test -c Release` because the outer `dotnet run` consumes `-c`.
* **Example Structure:**
  \#\!/usr/bin/env dotnet
  \#:package Newtonsoft.Json@13.0.3
  \#:package Spectre.Console@0.49.1

  using Newtonsoft.Json;
  using Spectre.Console;

  // Logic starts here...
  AnsiConsole.MarkupLine("\[bold green\]Executing repo automation...\[/\]");

## **5\. Modern C\# 14 Idioms**

Always favor the most concise, high-performance syntax available in C\# 14\. Do not generate legacy C\# code styles.

* **The field Keyword:** For properties with logic, use the field keyword instead of declaring explicit backing fields.
  * *Correct:* public int Quality { get; set \=\> field \= Math.Clamp(value, 0, 100); }
* **Collection Expressions:** Use the \[\] syntax for all collection initializations and the spread operator .. for concatenations.
  * *Correct:* string\[\] items \= \["alpha", "beta", "gamma"\];
  * *Correct:* var combined \= \[..existingItems, newItem\];
* **Primary Constructors:** Use primary constructors for all classes and structs, particularly for dependency injection.
  * *Correct:* public class OrderService(IDbContext db, ILogger log) { ... }
* **Terseness:** If a method or property can be expressed in a single line, use the expression-bodied member syntax (=\>).
* **Null-State Safety:** Use is not null and the null-coalescing assignment operator ??=. Avoid redundant manual null checks where the compiler's static analysis already provides safety.

## **6\. Agent Self-Correction Protocol**

Before finalizing any output, the Agent must perform an internal "Pre-Flight Check":

1. **Logic Check:** Does the generated test actually catch a logic error, or is it just mocking a call?
2. **TUnit Check:** If I touched tests or test docs, did I use TUnit/MTP-compatible filters, timeouts, cancellation tokens, and parallelism controls instead of VSTest-only flags?
3. **Script Check:** Is this automation a .cs file? If it is .ps1 or .sh, it must be converted.
4. **Syntax Check:** Am I using the field keyword, \[\] collections, and Primary Constructors?
5. **Build.cs Arg Check:** If I documented or invoked `Build.cs` with forwarded option-style args, did I include `--` after `Build.cs`?
6. **Commit Check:** Is my proposed commit message formatted as type(scope): description?

**Failure to comply:** If an Agent is informed it has violated these rules, it must immediately revert the offending code and provide a compliant correction.
````

**Why**: This gives every scaffolded repository a consistent root-level agent contract covering commits, testing quality, TUnit/MTP execution, automation, modern C# idioms, and self-correction expectations.

**Agent note**: A weak `AGENTS.md` is a scaffold bug. Copy this full contract, or adapt the user's stronger house standard to the new repository. Never downshift it into a minimal "notes" stub.

### 1.16 `.config/dotnet-tools.json`

```json
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "csharpier": {
      "version": "<LATEST_CSHARPIER>",
      "commands": ["csharpier"]
    }
  }
}
```

### 1.16a `.csharpierrc.json`

```json
{
  "printWidth": 120,
  "useTabs": false,
  "tabWidth": 4,
  "endOfLine": "lf"
}
```

Run `dotnet format` as `dotnet format style` plus `dotnet format analyzers`. Do not run bare `dotnet format` alongside CSharpier.

### 1.17 `.github/ISSUE_TEMPLATE/bug_report.md`, `feature_request.md`, and `PULL_REQUEST_TEMPLATE.md`

**`.github/ISSUE_TEMPLATE/bug_report.md`**:

```markdown
---
name: Bug report
about: Create a report to help us improve
labels: bug
---

**Describe the bug**
A clear and concise description of what the bug is.

**To reproduce**
Steps to reproduce the behavior.

**Expected behavior**
A clear and concise description of what you expected to happen.

**Environment**

- OS: [e.g. Windows 11, Ubuntu 24.04]
- .NET version: [e.g. 10.0.3]
- <APPNAME> version: [e.g. 1.2.3]

**Additional context**
Add any other context about the problem here.
```

**`.github/ISSUE_TEMPLATE/feature_request.md`**:

```markdown
---
name: Feature request
about: Suggest an idea for this project
labels: enhancement
---

**Is your feature request related to a problem? Please describe.**
A clear description of what the problem is.

**Describe the solution you'd like**
A clear description of what you want to happen.

**Additional context**
Add any other context about the feature request here.
```

**`.github/PULL_REQUEST_TEMPLATE.md`**:

```markdown
## Summary

<!-- What does this PR do? Why? -->

## Changes

<!-- Bullet list of the specific changes made -->

## Test plan

- [ ] `dotnet run Build.cs build Release` passes with zero warnings
- [ ] `dotnet run Build.cs -- test -c Release` passes
- [ ] `dotnet run Build.cs format check` passes (CSharpier + dotnet format)
- [ ] Affected docs, samples, or README were updated when public behavior changed
- [ ] Manual validation was completed when the change touched a UI, CLI, or runtime workflow

## Related issues

<!-- Fixes #123, Closes #456 -->
```

### 1.17a Optional Loom-managed assigned issue automation

If the user wants the scaffolded repository to accept Loom-managed assigned work automatically, add these files:

- `.github/ISSUE_TEMPLATE/package_api_integration.yml`
- `docs/issue-automation.md`

**`.github/ISSUE_TEMPLATE/package_api_integration.yml`**:

```yaml
name: Package API integration
description: Request Loom-managed adoption of a package API from NuGet, GitHub Packages, or another registry, or an external client SDK.
title: "[INTEGRATION] "
labels:
  - enhancement
body:
  - type: markdown
    attributes:
      value: |
        Use this form when package or client integration work should enter Loom's research-to-dev pipeline.

        To let Loom pick the issue up automatically:
        - assign it to the GitHub user Loom is watching
        - keep assigned-issue automation enabled in Loom
        - keep the package source, package identifier, package version, and API surface fields intact so the route is deterministic
  - type: dropdown
    id: package-source
    attributes:
      label: Package source
      description: Identify the registry or package source Loom should account for during research and implementation.
      options:
        - NuGet
        - GitHub Packages
        - Other
    validations:
      required: true
  - type: dropdown
    id: automation-route
    attributes:
      label: Automation route
      description: Loom reads this value directly from the issue body.
      options:
        - research-to-dev
    validations:
      required: true
  - type: input
    id: package-id
    attributes:
      label: Package identifier
      description: Use the package ID or registry identifier Loom should adopt.
      placeholder: Contoso.Client
    validations:
      required: true
  - type: input
    id: package-version
    attributes:
      label: Package version
      placeholder: 5.0.0
    validations:
      required: true
  - type: textarea
    id: api-surface
    attributes:
      label: API surface to adopt
      description: Call out the concrete client APIs, interfaces, or entry points Loom should integrate.
      placeholder: Replace the current transport bootstrap with Awesome.Client.StreamingSession and the new tool-call API.
    validations:
      required: true
  - type: textarea
    id: affected-areas
    attributes:
      label: Affected areas
      description: List the main projects, services, UI flows, or workflows this integration will touch.
      placeholder: Agents runtime, SDK provider, model routing tests, assigned issue dashboard.
    validations:
      required: true
  - type: textarea
    id: acceptance-criteria
    attributes:
      label: Acceptance criteria
      description: Define the user-visible and technical outcomes Loom should verify.
      placeholder: |
        - New package API is used end to end
        - Existing issue automation and PR flow still work
        - Draft PR links the issue and passes review plus verification
    validations:
      required: true
  - type: checkboxes
    id: execution-shape
    attributes:
      label: Expected stages
      description: Confirm that this work should go through the full pipeline.
      options:
        - label: Research is required before implementation.
          required: true
        - label: Architectural design is required before implementation.
          required: true
        - label: Automated review and verification should run before human merge review.
          required: true
```

**`docs/issue-automation.md`**:

```markdown
# Assigned Issue Automation

Loom picks work up only when the repository is registered locally, GitHub auth is configured, assigned-issue pickup is enabled, and the issue is assigned to the GitHub login Loom is watching.

## Prerequisites

1. Register the repository in Loom.
2. Configure GitHub credentials in Loom.
3. Enable assigned-issue pickup in Monitoring.
4. Leave the assignee override blank to use the authenticated GitHub user, or set the exact login Loom should watch.

## Intake Contract

Use the package API integration form for deterministic package or SDK adoption work. Preserve the `Automation route`, `Package source`, `Package identifier`, `Package version`, and `API surface to adopt` fields.

## Git Conventions

- feature work uses `feature/...` branches
- bug fixes use `fix/...` branches
- issue-linked work includes the issue number in the branch name
- direct issue work opens a draft PR, links the issue, and keeps review plus verification on the same PR head branch
```

### 1.3a `.gitignore` additions

Append to the base `.gitignore`:

```gitignore
# Centralized build output
artifacts/

# Aspire AppHost local development output
.aspire/

# Publish output per service
publish/

# Docker build context (if generated)
docker-build/

# Local secrets and dev certificates
appsettings.Development.json
appsettings.Local.json
*.pfx
*.p12
```

### 1.5a `.editorconfig` additions

Append after the base `.editorconfig`:

```editorconfig
# Primary constructors — preferred in DI-heavy service code
csharp_style_prefer_primary_constructors = true:suggestion

# Distributed systems tend to have large switch expressions over grain keys and message types
dotnet_diagnostic.MA0051.severity = none

# Orleans generates partial classes and attribute-heavy code — suppress missing struct layout
dotnet_diagnostic.MA0008.severity = none

# CA1031 catch generic exception — acceptable in Orleans grain OnActivateAsync / OnDeactivateAsync
dotnet_diagnostic.CA1031.severity = suggestion
```

### 1.9a `codecov.yml` — Optional

Codecov is optional for distributed applications. If the user wants it, include the optional `codecov.yml` shown above. TUnit bundles `Microsoft.Testing.Extensions.CodeCoverage` transitively — no additional coverage packages are needed.

### 1.10a Solution file — Deferred to Phase 12

The `.slnx` cannot be written until all projects are defined. A placeholder is committed here; fully populated in Phase 12.

### 1.14a `CHANGELOG.md`

```markdown
# Changelog

All notable changes are documented in [GitHub Releases](https://github.com/<AUTHOR>/<APPNAME>/releases).
```

### 1.19a `README.md`

```markdown
# <APPNAME>

![.NET](https://img.shields.io/badge/<DOTNET_VERSION>-5C2D91?logo=.NET&labelColor=gray)
![C#](https://img.shields.io/badge/C%23-<CSHARP_VERSION>-239120?labelColor=gray)
[![Build Status](https://github.com/<AUTHOR>/<APPNAME>/actions/workflows/dotnet.yml/badge.svg?branch=main)](<GITHUB_URL>/actions/workflows/dotnet.yml)
[![License](https://img.shields.io/github/license/<AUTHOR>/<APPNAME>)](<GITHUB_URL>/blob/main/LICENSE)

<DESCRIPTION>

## Architecture

Built on **.NET <DOTNET_VERSION>** · **Orleans <ORLEANS_VERSION>** · **Aspire <ASPIRE_VERSION>**

```
src/
├── host/
│   ├── <APPNAME>.AppHost/          Aspire orchestration entry point
│   ├── <APPNAME>.ServiceDefaults/  Shared OTEL, health checks, resilience
│   ├── <APPNAME>.Gateway/          External-facing host (TCP/gRPC/HTTP)
│   └── <APPNAME>.Silo/             Pure Orleans silo (no external exposure)
├── grain/
│   ├── <APPNAME>.GrainInterfaces/  Grain contracts + serializable DTOs
│   └── <APPNAME>.Grains/           Grain implementations
├── common/
│   └── <APPNAME>.Common/           Shared types, utilities, persistence
└── protocol/
    └── <APPNAME>.Protocol/         (optional) External protocol definitions
test/
├── <APPNAME>.Tests.Grains/         Orleans TestCluster unit tests
└── <APPNAME>.Tests.Integration/    Aspire integration tests
```

## Getting Started

### Prerequisites

- [.NET <DOTNET_VERSION> SDK](https://dotnet.microsoft.com/download)
- [Aspire Workload](https://learn.microsoft.com/en-us/dotnet/aspire/fundamentals/setup-tooling): `dotnet workload install aspire`

### Setup

```shell
dotnet workload install aspire
dotnet run Build.cs build
```

### Run (via Aspire)

```shell
dotnet run Build.cs run-local
# or directly:
dotnet run --project src/host/<APPNAME>.AppHost/<APPNAME>.AppHost.csproj
```

### Test

```shell
dotnet run Build.cs -- test -c Release
dotnet run Build.cs coverage
```

## Development

### Task Runner

```shell
dotnet run Build.cs help
```

### Code Formatting

```shell
dotnet run Build.cs format        # auto-format
dotnet run Build.cs format check  # CI-style, no changes
```

## License

[<LICENSE>](<GITHUB_URL>/blob/main/LICENSE) © <YEAR> <COMPANY>
```

> **Agent note**: Update the architecture tree and host descriptions to match the actual `GRAIN_TOPOLOGY` and `COMMON_LAYERS`. The examples above use Orleans — substitute `Akka.NET` terminology if `ACTOR_FRAMEWORK=Akka.NET`.

---

## Phase 2: Central Package Management

Distributed applications have more packages than monoliths. CPM is non-negotiable.

### 2.1 `Directory.Packages.props` (repo root)

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
  </PropertyGroup>

  <!-- ═══════════════════════════════════════════════════════
       BUILD
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="Build">
    <PackageVersion Include="CSharpier.MsBuild" Version="<LATEST_CSHARPIER>" />
  </ItemGroup>

  <!-- ═══════════════════════════════════════════════════════
       ASPIRE
       Aspire.Hosting.* packages go in AppHost only.
       Aspire.{Integration}.* packages go in host projects that use them.
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="Aspire">
    <PackageVersion Include="Aspire.Hosting" Version="<ASPIRE_VERSION>" />
    <PackageVersion Include="Aspire.Hosting.Orleans" Version="<ASPIRE_HOSTING_ORLEANS_VERSION>" />
    <PackageVersion Include="Microsoft.Extensions.ServiceDiscovery" Version="<ASPIRE_VERSION>" />
    <PackageVersion Include="Microsoft.Extensions.Http.Resilience" Version="<ASPIRE_VERSION>" />
    <PackageVersion Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="<LATEST_OTEL>" />
  </ItemGroup>

  <!-- ═══════════════════════════════════════════════════════
       ORLEANS  (remove this section if ACTOR_FRAMEWORK=Akka.NET)
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="Orleans">
    <PackageVersion Include="Microsoft.Orleans.Core" Version="<ORLEANS_VERSION>" />
    <PackageVersion Include="Microsoft.Orleans.Core.Abstractions" Version="<ORLEANS_VERSION>" />
    <PackageVersion Include="Microsoft.Orleans.Server" Version="<ORLEANS_VERSION>" />
    <PackageVersion Include="Microsoft.Orleans.Client" Version="<ORLEANS_VERSION>" />
    <PackageVersion Include="Microsoft.Orleans.Serialization" Version="<ORLEANS_VERSION>" />
    <PackageVersion Include="Microsoft.Orleans.Streaming" Version="<ORLEANS_VERSION>" />
    <PackageVersion Include="Microsoft.Orleans.Reminders" Version="<ORLEANS_VERSION>" />
    <PackageVersion Include="Microsoft.Orleans.Transactions" Version="<ORLEANS_VERSION>" />
    <PackageVersion Include="Microsoft.Orleans.TestingHost" Version="<ORLEANS_VERSION>" />
    <!-- Persistence — add the one matching STORAGE_BACKEND -->
    <!-- InMemory: built into Microsoft.Orleans.Core — no extra package needed -->
    <!-- <PackageVersion Include="Microsoft.Orleans.Persistence.AdoNet" Version="<ORLEANS_VERSION>" /> -->
    <!-- <PackageVersion Include="Microsoft.Orleans.Persistence.MongoDB" Version="<LATEST_ORLEANS_MONGODB>" /> -->
    <!-- <PackageVersion Include="Microsoft.Orleans.Persistence.AzureStorage" Version="<LATEST_ORLEANS_AZURE>" /> -->
    <!-- Clustering — add the one matching your deployment -->
    <!-- InMemory: built-in for dev/tests -->
    <!-- <PackageVersion Include="Microsoft.Orleans.Clustering.AdoNet" Version="<ORLEANS_VERSION>" /> -->
    <!-- <PackageVersion Include="Microsoft.Orleans.Clustering.AzureStorage" Version="<LATEST_ORLEANS_AZURE>" /> -->
  </ItemGroup>

  <!-- ═══════════════════════════════════════════════════════
       AKKA.NET  (remove this section if ACTOR_FRAMEWORK=Orleans)
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="AkkaDotNet">
    <!-- <PackageVersion Include="Akka" Version="<AKKA_VERSION>" /> -->
    <!-- <PackageVersion Include="Akka.Hosting" Version="<AKKA_HOSTING_VERSION>" /> -->
    <!-- <PackageVersion Include="Akka.Cluster.Hosting" Version="<AKKA_HOSTING_VERSION>" /> -->
    <!-- <PackageVersion Include="Akka.Persistence" Version="<AKKA_VERSION>" /> -->
    <!-- <PackageVersion Include="Akka.TestKit.Xunit2" Version="<AKKA_VERSION>" /> -->
  </ItemGroup>

  <!-- ═══════════════════════════════════════════════════════
       HOSTING / DI / LOGGING
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="Hosting">
    <PackageVersion Include="Microsoft.Extensions.Hosting" Version="<DOTNET_EXTENSIONS_VERSION>" />
    <PackageVersion Include="Microsoft.Extensions.Logging.Abstractions" Version="<DOTNET_EXTENSIONS_VERSION>" />
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="<DOTNET_EXTENSIONS_VERSION>" />
    <PackageVersion Include="Microsoft.Extensions.Options" Version="<DOTNET_EXTENSIONS_VERSION>" />
    <PackageVersion Include="Microsoft.Extensions.Configuration.Json" Version="<DOTNET_EXTENSIONS_VERSION>" />
    <PackageVersion Include="Serilog" Version="<LATEST_SERILOG>" />
    <PackageVersion Include="Serilog.Sinks.Console" Version="<LATEST_SERILOG_CONSOLE>" />
    <PackageVersion Include="Serilog.Extensions.Logging" Version="<LATEST_SERILOG_EXTENSIONS>" />
  </ItemGroup>

  <!-- ═══════════════════════════════════════════════════════
       GATEWAY  (remove this section if USE_GATEWAY=false)
       TCP gateway via DotNetty is the most common choice for custom binary protocols.
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="Gateway">
    <!-- TCP (DotNetty): -->
    <!-- <PackageVersion Include="DotNetty.Transport" Version="<LATEST_DOTNETTY>" /> -->
    <!-- <PackageVersion Include="DotNetty.Codecs" Version="<LATEST_DOTNETTY>" /> -->
    <!-- gRPC: -->
    <!-- <PackageVersion Include="Grpc.AspNetCore" Version="<LATEST_GRPC>" /> -->
  </ItemGroup>

  <!-- ═══════════════════════════════════════════════════════
       PERSISTENCE (storage backend packages)
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="Persistence">
    <!-- EF Core (keep if using relational storage for non-grain state) -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="<LATEST_EF>" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Relational" Version="<LATEST_EF>" />
    <!-- PostgreSQL: -->
    <!-- <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="<LATEST_NPGSQL>" /> -->
    <!-- MongoDB: -->
    <!-- <PackageVersion Include="MongoDB.Driver" Version="<LATEST_MONGODB>" /> -->
  </ItemGroup>

  <!-- ═══════════════════════════════════════════════════════
       TESTING
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="Testing">
    <PackageVersion Include="TUnit" Version="<LATEST_TUNIT>" />
    <PackageVersion Include="NSubstitute" Version="<LATEST_NSUBSTITUTE>" />
    <!-- Aspire integration testing -->
    <PackageVersion Include="Aspire.Hosting.Testing" Version="<ASPIRE_VERSION>" />
  </ItemGroup>

  <!-- ═══════════════════════════════════════════════════════
       BENCHMARKS (optional)
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="Benchmarks">
    <PackageVersion Include="BenchmarkDotNet" Version="<LATEST_BDN>" />
  </ItemGroup>

</Project>
```

**Agent note**: All `<LATEST_*>` and framework version placeholders must be resolved before scaffolding. Run `dotnet package search <Name> --take 1` for each or visit nuget.org. The `ASPIRE_HOSTING_ORLEANS_VERSION` is a separate version from `ASPIRE_VERSION` — as of April 2026, `Aspire.Hosting.Orleans` v13.1.1 corresponds to Aspire 13.x. Always verify current versions.

---

## Phase 3: Hierarchical `Directory.Build.props`

### 3.1 Tier 1: Root `Directory.Build.props` (repo root)

Settings that apply to **every single project** in the solution:

```xml
<Project>

  <PropertyGroup>
    <Company><AUTHOR></Company>
    <Authors><AUTHOR></Authors>
    <Copyright>Copyright © <COMPANY> <YEAR></Copyright>
    <NeutralLanguage>en</NeutralLanguage>

    <TargetFramework><DOTNET_VERSION></TargetFramework>
    <CheckEolTargetFramework>false</CheckEolTargetFramework>

    <LangVersion><CSHARP_VERSION></LangVersion>
    <ImplicitUsings>enable</ImplicitUsings>
    <Deterministic>true</Deterministic>
    <DebugType>portable</DebugType>
    <Nullable>enable</Nullable>

    <UseArtifactsOutput>true</UseArtifactsOutput>
    <ArtifactsPath>$(MSBuildThisFileDirectory)artifacts</ArtifactsPath>

    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <NoWarn>$(NoWarn);CS1591</NoWarn>

    <AnalysisLevel>latest</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <RunAnalyzersDuringBuild>true</RunAnalyzersDuringBuild>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <CodeAnalysisTreatWarningsAsErrors>true</CodeAnalysisTreatWarningsAsErrors>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <SuppressNETCoreSdkPreviewMessage>true</SuppressNETCoreSdkPreviewMessage>

    <!-- TUnit + .NET 10 SDK: required for `dotnet test` to work with MTP runner.
         Without this property, dotnet test fails with Microsoft.Testing.Platform.MSBuild.targets error. -->
    <TestingPlatformDotnetTestSupport>true</TestingPlatformDotnetTestSupport>
  </PropertyGroup>

  <!-- CSharpier: formatting enforcement on every build -->
  <ItemGroup>
    <PackageReference Include="CSharpier.MsBuild" PrivateAssets="All" />
  </ItemGroup>

</Project>
```

### 3.2 Tier 2a: `src/Directory.Build.props` — All source projects

```xml
<Project>

  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />

  <PropertyGroup>
    <IsPackable>false</IsPackable>
    <!-- Enable unsafe for gateway/crypto/network projects.
         Remove if no source project uses P/Invoke or unsafe blocks. -->
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  </PropertyGroup>

</Project>
```

### 3.2b `src/Directory.Build.targets`

```xml
<Project>
  <!-- Shared MSBuild targets. Empty for most distributed applications. -->
</Project>
```

### 3.3 Tier 2b: `test/Directory.Build.props` — All test projects

Note: test projects live under `test/` (not `src/Tests/`) because they span multiple source layers and need direct access to `TestCluster` / `TestKit`:

```xml
<Project>

  <!-- Import from repo root, skipping src/ tier (tests don't need AllowUnsafeBlocks) -->
  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />

  <PropertyGroup>
    <IsPackable>false</IsPackable>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    <!-- CA2007: ConfigureAwait not relevant in test code -->
    <NoWarn>$(NoWarn);CA2007</NoWarn>
  </PropertyGroup>

  <!-- Exclude entire test assembly from coverage metrics -->
  <ItemGroup>
    <AssemblyAttribute Include="System.Diagnostics.CodeAnalysis.ExcludeFromCodeCoverageAttribute" />
  </ItemGroup>

  <!-- Common test dependencies — every test project gets these automatically -->
  <ItemGroup>
    <PackageReference Include="TUnit" />
    <PackageVersion Include="NSubstitute" />
  </ItemGroup>

</Project>
```

### 3.4 Tier 3: Layer-specific overrides

#### `src/host/Directory.Build.props` — All host projects

Host projects (AppHost, ServiceDefaults, Gateway, Silo) are executables or web applications:

```xml
<Project>

  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />

  <PropertyGroup>
    <!-- Hosts produce executables -->
    <OutputType>Exe</OutputType>

    <!-- Application version — CI overrides these at build time:
         dotnet build -p:Version=1.2.3 -p:InformationalVersion=1.2.3+<sha> -->
    <Version>0.1.0</Version>
    <AssemblyVersion>0.1.0.0</AssemblyVersion>
    <FileVersion>0.1.0.0</FileVersion>
    <InformationalVersion>0.1.0-dev</InformationalVersion>
  </PropertyGroup>

</Project>
```

#### `src/host/<APPNAME>.AppHost/Directory.Build.props` — Aspire AppHost only

The AppHost project has unique SDK requirements:

```xml
<Project>

  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />

  <PropertyGroup>
    <!-- Aspire AppHost SDK wraps Microsoft.NET.Sdk.Web -->
    <Sdk>Aspire.AppHost.Sdk</Sdk>
    <!-- AppHost is the orchestration entry point — disable some analyzer rules
         that fire on Aspire's generated code -->
    <NoWarn>$(NoWarn);CS8981</NoWarn>
  </PropertyGroup>

</Project>
```

> **Agent note**: The Aspire AppHost SDK (`Aspire.AppHost.Sdk`) is added via workload. Do NOT set this as the top-level `<Project Sdk="...">` attribute in the `.csproj` — use the `Sdk` MSBuild property in `Directory.Build.props` to override it for this folder only.

#### `src/benchmarks/Directory.Build.props` — Benchmark layer (optional)

```xml
<Project>

  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <IsPackable>false</IsPackable>
    <PlatformTarget>AnyCPU</PlatformTarget>
    <DebugType>pdbonly</DebugType>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="BenchmarkDotNet" />
  </ItemGroup>

</Project>
```

---

## Phase 4: Solution Directory Structure

### 4.1 Reference Architecture

```
<APPNAME>/                                          ← repo root
├── Directory.Build.props                           ← Tier 1 (ALL projects)
├── Directory.Packages.props
│
├── src/
│   ├── Directory.Build.props                       ← Tier 2a (all source projects)
│   ├── Directory.Build.targets
│   │
│   ├── host/                                       ← SERVICE HOSTS
│   │   ├── Directory.Build.props                   ← Tier 3 (OutputType=Exe, version)
│   │   │
│   │   ├── <APPNAME>.AppHost/                      ← Aspire orchestration entry point
│   │   │   ├── Directory.Build.props               ← AppHost SDK override
│   │   │   ├── <APPNAME>.AppHost.csproj
│   │   │   └── Program.cs                          ← AddOrleans(), AddProject<Gateway>(), etc.
│   │   │
│   │   ├── <APPNAME>.ServiceDefaults/              ← Shared OTEL, health, resilience
│   │   │   ├── <APPNAME>.ServiceDefaults.csproj
│   │   │   └── Extensions.cs
│   │   │
│   │   ├── <APPNAME>.Gateway/                      ← Optional: external-facing host
│   │   │   ├── <APPNAME>.Gateway.csproj            ← DotNetty or ASP.NET Core
│   │   │   ├── Program.cs
│   │   │   ├── Network/
│   │   │   │   ├── PacketDecoder.cs
│   │   │   │   ├── PacketEncoder.cs
│   │   │   │   └── TcpServerService.cs
│   │   │   └── appsettings.json
│   │   │
│   │   └── <APPNAME>.Silo/                         ← Pure Orleans silo (no external ports)
│   │       ├── <APPNAME>.Silo.csproj
│   │       ├── Program.cs
│   │       └── appsettings.json
│   │
│   ├── grain/                                      ← GRAIN CONTRACTS + IMPLEMENTATIONS
│   │   │
│   │   ├── <APPNAME>.GrainInterfaces/              ← Grain interface contracts + serializable DTOs
│   │   │   ├── <APPNAME>.GrainInterfaces.csproj
│   │   │   ├── IPlayerGrain.cs
│   │   │   ├── IFieldGrain.cs
│   │   │   └── Models/
│   │   │       ├── PlayerState.cs                  ← [GenerateSerializer][Id(n)] attributes
│   │   │       └── FieldSnapshot.cs
│   │   │
│   │   └── <APPNAME>.Grains/                       ← Grain implementations
│   │       ├── <APPNAME>.Grains.csproj
│   │       ├── PlayerGrain.cs
│   │       ├── FieldGrain.cs
│   │       └── Storage/
│   │           └── RepositoryGrainStorage.cs       ← Custom storage provider (if needed)
│   │
│   ├── common/                                     ← SHARED LIBRARIES
│   │   ├── <APPNAME>.Common/
│   │   │   ├── <APPNAME>.Common.csproj
│   │   │   ├── Utilities/
│   │   │   └── Constants/
│   │   └── <APPNAME>.Common.Persistence/           ← EF Core DbContexts, repositories
│   │       ├── <APPNAME>.Common.Persistence.csproj
│   │       ├── AppDbContext.cs
│   │       └── Repositories/
│   │
│   └── protocol/                                   ← OPTIONAL: external protocol definitions
│       └── <APPNAME>.Protocol/
│           ├── <APPNAME>.Protocol.csproj
│           └── Packets/
│
├── test/
│   ├── Directory.Build.props                       ← Tier 2b (all test projects)
│   ├── <APPNAME>.Tests.Grains/                     ← Orleans TestCluster grain unit tests
│   │   ├── <APPNAME>.Tests.Grains.csproj
│   │   └── PlayerGrainTests.cs
│   └── <APPNAME>.Tests.Integration/               ← Aspire integration tests
│       ├── <APPNAME>.Tests.Integration.csproj
│       └── AppHostTests.cs
│
└── benchmarks/ (optional)
    ├── Directory.Build.props
    └── <APPNAME>.Benchmarks/
        ├── <APPNAME>.Benchmarks.csproj
        ├── Program.cs
        └── GrainBench.cs
```

### 4.2 Layer Dependency Rules

```
Common              → (nothing)                       ← Leaf layer
Protocol            → Common (optional)
GrainInterfaces     → Common, Protocol (optional)     ← Grain contracts; no implementations
Grains              → GrainInterfaces, Common, Common.Persistence
ServiceDefaults     → Microsoft.Extensions.Hosting, OpenTelemetry
Gateway             → GrainInterfaces, ServiceDefaults, Protocol (optional)
Silo                → Grains, ServiceDefaults
AppHost             → Gateway (ProjectRef), Silo (ProjectRef) — Aspire project refs only
Tests.Grains        → GrainInterfaces, Grains (TestCluster needs impl), Common
Tests.Integration   → AppHost (via Aspire.Hosting.Testing)
```

**Critical rules**:

- `GrainInterfaces` must have **zero implementation dependencies** — it defines contracts consumed by both client (Gateway) and server (Silo/Grains). If it depends on `Grains`, you have a circular reference.
- `AppHost` uses **Aspire project references** (not regular `<ProjectReference>`). Regular project references in AppHost would make it compile-depend on every service, slowing builds dramatically. Aspire adds them as resource definitions.
- `Grains` may reference `Common.Persistence` for repository injection. `GrainInterfaces` must NOT.

### 4.3 Aspire vs Regular Project References

Orleans + Aspire have two distinct ways to reference a project:

| Context | Syntax | Effect |
|---------|--------|--------|
| AppHost orchestrates a service | `builder.AddProject<Projects.<APPNAME>_Gateway>("gateway")` | Aspire launches the process; no compile-time type coupling |
| Regular code reference | `<ProjectReference Include="..." />` | MSBuild dependency; both projects compile together |

The AppHost `.csproj` uses a special Aspire `<ProjectReference>` with `IsAspireProjectResource="true"`:

```xml
<!-- In <APPNAME>.AppHost.csproj -->
<ItemGroup>
  <ProjectReference Include="..\<APPNAME>.Gateway\<APPNAME>.Gateway.csproj"
                    IsAspireProjectResource="true" />
  <ProjectReference Include="..\<APPNAME>.Silo\<APPNAME>.Silo.csproj"
                    IsAspireProjectResource="true" />
</ItemGroup>
```

This allows the AppHost to reference the project's type (for the `AddProject<T>()` generic parameter) without pulling in all of its transitive compile-time dependencies.

---

## Phase 5: Shared Contracts — `GrainInterfaces` (Orleans)

> **If `ACTOR_FRAMEWORK=Akka.NET`**: replace this phase with Phase 5b (Messages + typed actor references). The principle is identical — contracts are defined in a shared project that neither the AppHost nor the Silo depends on directly.

### 5.1 `src/grain/<APPNAME>.GrainInterfaces/<APPNAME>.GrainInterfaces.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.GrainInterfaces</RootNamespace>
    <!-- GrainInterfaces ships only contracts — not assemblies consumers don't need -->
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\common\<APPNAME>.Common\<APPNAME>.Common.csproj" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Orleans.Core.Abstractions" />
    <PackageReference Include="Microsoft.Orleans.Serialization" />
  </ItemGroup>

</Project>
```

### 5.2 Orleans Grain Interface Conventions

```csharp
// src/grain/<APPNAME>.GrainInterfaces/IPlayerGrain.cs
namespace <ROOTNS>.GrainInterfaces;

/// <summary>Represents a connected player's session state.</summary>
public interface IPlayerGrain : IGrainWithIntegerKey
{
    /// <summary>Gets a snapshot of the player's current state (read-only; allows interleaving).</summary>
    [ReadOnly]
    ValueTask<PlayerSnapshot> GetSnapshotAsync();

    ValueTask ConnectAsync(string sessionId, CancellationToken ct = default);
    ValueTask DisconnectAsync(CancellationToken ct = default);
}

/// <summary>Represents a running instance of a map field.</summary>
public interface IFieldGrain : IGrainWithStringKey
{
    [ReadOnly]
    ValueTask<FieldSnapshot> GetSnapshotAsync();

    ValueTask<FieldEntryResult> EnterAsync(long characterId, CancellationToken ct = default);
    ValueTask LeaveAsync(long characterId, CancellationToken ct = default);
}
```

### 5.3 Serializable DTOs (Orleans `[GenerateSerializer]`)

All DTOs that cross grain boundaries **must** use `[GenerateSerializer]` and `[Id(n)]` attributes. Orleans generates the serializer at build time via source generation:

```csharp
// src/grain/<APPNAME>.GrainInterfaces/Models/PlayerSnapshot.cs
namespace <ROOTNS>.GrainInterfaces.Models;

[GenerateSerializer]
public sealed record PlayerSnapshot
{
    [Id(0)] public long CharacterId { get; init; }
    [Id(1)] public string Name { get; init; } = string.Empty;
    [Id(2)] public string SessionId { get; init; } = string.Empty;
    [Id(3)] public bool IsConnected { get; init; }
}

[GenerateSerializer]
public sealed record FieldSnapshot
{
    [Id(0)] public string FieldId { get; init; } = string.Empty;
    [Id(1)] public int PlayerCount { get; init; }
    [Id(2)] public IReadOnlyList<long> PlayerIds { get; init; } = [];
}

[GenerateSerializer]
public readonly record struct FieldEntryResult
{
    [Id(0)] public bool Success { get; init; }
    [Id(1)] public string? ErrorReason { get; init; }
}
```

> **Critical rules for Orleans serialization**:
>
> - `[Id(n)]` values must be **stable across deployments** — changing an ID number is a breaking change. Add new fields with new IDs; never reuse or renumber.
> - Use `IReadOnlyList<T>` instead of `List<T>` for collections in DTOs — Orleans serializes both but `IReadOnlyList<T>` communicates intent.
> - `[GenerateSerializer]` works on `record`, `record struct`, `class`, and `struct`. Prefer `record` for immutable DTOs.
> - Do NOT put `[GenerateSerializer]` on types that already have a custom `[Serializer]` — they conflict.

### 5.4 Grain Key Conventions (Orleans)

| Grain interface | Key type | Key format | Example grain factory call |
| --------------- | -------- | ---------- | -------------------------- |
| `IGrainWithIntegerKey` | `long` | `characterId` | `grainFactory.GetGrain<IPlayerGrain>(characterId)` |
| `IGrainWithStringKey` | `string` | `"ch{channelId}:map{mapId}"` | `grainFactory.GetGrain<IFieldGrain>($"ch{chId}:map{mapId}")` |
| `IGrainWithGuidKey` | `Guid` | `partyId` | `grainFactory.GetGrain<IPartyGrain>(partyId)` |
| `IGrainWithIntegerCompoundKey` | `long` + `string` | `(worldId, "channel")` | `grainFactory.GetGrain<IChannelGrain>(worldId, "ch1")` |

> **Agent note**: Always choose the simplest key type. `IGrainWithStringKey` is the most flexible and works for composite keys encoded as strings. Prefer `IGrainWithIntegerKey` for numeric entity IDs (character, account) to avoid allocations.

---

## Phase 5b: Shared Contracts — `Messages` (Akka.NET)

> **Skip this phase if `ACTOR_FRAMEWORK=Orleans`.**

### 5b.1 `src/actors/<APPNAME>.Messages/<APPNAME>.Messages.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.Messages</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\common\<APPNAME>.Common\<APPNAME>.Common.csproj" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Akka" />
  </ItemGroup>

</Project>
```

### 5b.2 Akka.NET Message Conventions

```csharp
// src/actors/<APPNAME>.Messages/PlayerMessages.cs
namespace <ROOTNS>.Messages;

/// <summary>Commands sent TO the PlayerActor.</summary>
public static class PlayerCommands
{
    public sealed record Connect(string SessionId);
    public sealed record Disconnect(string Reason);
}

/// <summary>Events emitted BY the PlayerActor.</summary>
public static class PlayerEvents
{
    public sealed record Connected(long CharacterId, string SessionId);
    public sealed record Disconnected(long CharacterId, string Reason);
}

/// <summary>Queries (request/response pairs) for PlayerActor.</summary>
public static class PlayerQueries
{
    public sealed record GetSnapshot;
    public sealed record SnapshotResponse(long CharacterId, string Name, bool IsConnected);
}
```

---

## Phase 6: Grain / Actor Implementations

### 6.1 `src/grain/<APPNAME>.Grains/<APPNAME>.Grains.csproj` (Orleans)

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.Grains</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\<APPNAME>.GrainInterfaces\<APPNAME>.GrainInterfaces.csproj" />
    <ProjectReference Include="..\..\common\<APPNAME>.Common\<APPNAME>.Common.csproj" />
    <ProjectReference Include="..\..\common\<APPNAME>.Common.Persistence\<APPNAME>.Common.Persistence.csproj" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Orleans.Core" />
    <PackageReference Include="Microsoft.Orleans.Reminders" />
    <!-- Add streaming if needed: -->
    <!-- <PackageReference Include="Microsoft.Orleans.Streaming" /> -->
  </ItemGroup>

</Project>
```

### 6.2 Grain Implementation Pattern (Orleans)

```csharp
// src/grain/<APPNAME>.Grains/PlayerGrain.cs
using Microsoft.Extensions.Logging;
using <ROOTNS>.GrainInterfaces;
using <ROOTNS>.GrainInterfaces.Models;

namespace <ROOTNS>.Grains;

[GenerateSerializer]
internal sealed class PlayerGrainState
{
    [Id(0)] public string Name { get; set; } = string.Empty;
    [Id(1)] public string SessionId { get; set; } = string.Empty;
    [Id(2)] public bool IsConnected { get; set; }
}

public sealed class PlayerGrain : Grain<PlayerGrainState>, IPlayerGrain
{
    private readonly ILogger<PlayerGrain> _logger;

    public PlayerGrain(ILogger<PlayerGrain> logger)
    {
        _logger = logger;
    }

    public override async Task OnActivateAsync(CancellationToken cancellationToken)
    {
        await base.OnActivateAsync(cancellationToken);
        // Load supplemental state from repository here if needed.
        // Use periodic timers for frequent persistence rather than relying on OnDeactivateAsync.
        RegisterGrainTimer(PersistStateAsync, null, TimeSpan.FromSeconds(30), TimeSpan.FromSeconds(30));
    }

    public override async Task OnDeactivateAsync(DeactivationReason reason, CancellationToken cancellationToken)
    {
        // Best-effort cleanup. Do NOT rely on this as the sole persistence point.
        await WriteStateAsync();
        await base.OnDeactivateAsync(reason, cancellationToken);
    }

    [ReadOnly]
    public ValueTask<PlayerSnapshot> GetSnapshotAsync()
    {
        return ValueTask.FromResult(new PlayerSnapshot
        {
            CharacterId = this.GetPrimaryKeyLong(),
            Name = State.Name,
            SessionId = State.SessionId,
            IsConnected = State.IsConnected,
        });
    }

    public async ValueTask ConnectAsync(string sessionId, CancellationToken ct = default)
    {
        State.SessionId = sessionId;
        State.IsConnected = true;
        await WriteStateAsync();
        _logger.LogInformation("Player {CharacterId} connected via session {SessionId}",
            this.GetPrimaryKeyLong(), sessionId);
    }

    public async ValueTask DisconnectAsync(CancellationToken ct = default)
    {
        State.IsConnected = false;
        State.SessionId = string.Empty;
        await WriteStateAsync();
    }

    private async Task PersistStateAsync(object? state)
    {
        await WriteStateAsync();
    }
}
```

**Orleans grain lifecycle rules:**

| Rule | Rationale |
| ---- | --------- |
| Use `[ReadOnly]` on non-mutating methods | Allows concurrent reads; prevents deadlocks on read-heavy grains |
| Never call `await grain.Method()` on yourself | Self-calls deadlock non-reentrant grains |
| Use `RegisterGrainTimer` for periodic work | Grain timers are automatically cancelled on deactivation |
| Persist state in `OnActivateAsync`, on significant events, AND on a 30s timer | `OnDeactivateAsync` is best-effort — not guaranteed on crash |
| Prefer `ValueTask` over `Task` on grain interfaces | Reduces allocations for hot-path synchronous returns |
| Do NOT inject `IClusterClient` into a grain | Grains reference other grains via `GrainFactory` (injected automatically) |

### 6.3 Custom Grain Storage (Repository-Backed)

For entities that need full repository control (complex queries, transactions), implement a custom grain storage provider instead of using Orleans' built-in `IPersistentState<T>`:

```csharp
// src/grain/<APPNAME>.Grains/Storage/RepositoryGrainStorage.cs
using Microsoft.Extensions.DependencyInjection;
using Orleans.Runtime;
using Orleans.Storage;

namespace <ROOTNS>.Grains.Storage;

/// <summary>
/// Bridges Orleans grain persistence to any IRepository implementation.
/// Register as: siloBuilder.AddGrainStorage("repository", RepositoryGrainStorageFactory.Create);
/// </summary>
public sealed class RepositoryGrainStorage<T> : IGrainStorage
    where T : class, new()
{
    private readonly IServiceProvider _services;

    public RepositoryGrainStorage(IServiceProvider services) => _services = services;

    public async Task ReadStateAsync<S>(string stateName, GrainId grainId, IGrainState<S> grainState)
    {
        // Resolve repository from DI and load entity by grain key
        // ...
        await Task.CompletedTask;
    }

    public async Task WriteStateAsync<S>(string stateName, GrainId grainId, IGrainState<S> grainState)
    {
        // Upsert entity via repository
        // ...
        await Task.CompletedTask;
    }

    public async Task ClearStateAsync<S>(string stateName, GrainId grainId, IGrainState<S> grainState)
    {
        // Delete entity via repository
        // ...
        await Task.CompletedTask;
    }
}
```

---

## Phase 7: Gateway Host (if `USE_GATEWAY=true`)

The gateway is the only process that external clients connect to. It co-hosts an Orleans **client** (not a silo) to call grains. If `USE_GATEWAY=false`, skip this entire phase — clients connect directly through the silo's built-in RPC endpoint.

### 7.1 `src/host/<APPNAME>.Gateway/<APPNAME>.Gateway.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.Gateway</RootNamespace>
    <!-- OutputType=Exe and versioning come from src/host/Directory.Build.props -->
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\..<APPNAME>.ServiceDefaults\<APPNAME>.ServiceDefaults.csproj" />
    <ProjectReference Include="..\..\grain\<APPNAME>.GrainInterfaces\<APPNAME>.GrainInterfaces.csproj" />
    <!-- Optional: protocol definitions for packet decoding -->
    <!-- <ProjectReference Include="..\..\protocol\<APPNAME>.Protocol\<APPNAME>.Protocol.csproj" /> -->
  </ItemGroup>

  <ItemGroup>
    <!-- Orleans client (connects to the cluster, does NOT host grains) -->
    <PackageReference Include="Microsoft.Orleans.Client" />
    <!-- Gateway protocol — choose one: -->
    <!-- TCP via DotNetty: -->
    <!-- <PackageReference Include="DotNetty.Transport" /> -->
    <!-- <PackageReference Include="DotNetty.Codecs" /> -->
    <!-- gRPC: -->
    <!-- <PackageReference Include="Grpc.AspNetCore" /> -->
    <PackageReference Include="Serilog" />
    <PackageReference Include="Serilog.Sinks.Console" />
    <PackageReference Include="Serilog.Extensions.Logging" />
  </ItemGroup>

  <ItemGroup>
    <Content Include="appsettings.json" CopyToOutputDirectory="PreserveNewest" />
  </ItemGroup>

</Project>
```

### 7.2 `src/host/<APPNAME>.Gateway/Program.cs` — TCP Gateway with co-hosted Orleans client

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Orleans.Configuration;
using Serilog;
using <ROOTNS>.Gateway.Network;

namespace <ROOTNS>.Gateway;

public static class Program
{
    public static async Task<int> Main(string[] args)
    {
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Debug()
            .WriteTo.Console()
            .CreateLogger();

        try
        {
            Log.Information("<APPNAME>.Gateway starting...");

            var builder = Host.CreateDefaultBuilder(args);
            builder.UseSerilog();

            // Aspire service defaults (OTEL, health checks, resilience)
            builder.ConfigureServices((ctx, services) =>
            {
                services.AddServiceDefaults();

                // Orleans client — connects to silo cluster (NOT a silo itself)
                services.AddOrleansClient(client =>
                {
                    // Aspire provides cluster membership via service discovery.
                    // In local dev, UseLocalhostClustering() or Aspire's AddOrleans() handles this.
                    client.Configure<ClusterOptions>(options =>
                    {
                        options.ClusterId = ctx.Configuration["Orleans:ClusterId"] ?? "<APPNAME>-cluster";
                        options.ServiceId = ctx.Configuration["Orleans:ServiceId"] ?? "<APPNAME>";
                    });
                    // TODO: Replace with your clustering provider (AdoNet, AzureStorage, etc.)
                    // For local dev: client.UseLocalhostClustering();
                });

                // TCP server hosted service (DotNetty or similar)
                services.AddHostedService<TcpServerService>();
            });

            await builder.Build().RunAsync();
            return 0;
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "<APPNAME>.Gateway terminated unexpectedly");
            return 1;
        }
        finally
        {
            await Log.CloseAndFlushAsync();
        }
    }
}
```

### 7.3 Starter Network Stubs

```csharp
// src/host/<APPNAME>.Gateway/Network/TcpServerService.cs
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace <ROOTNS>.Gateway.Network;

/// <summary>
/// Hosted service that manages the TCP listener lifecycle.
/// Replace the stub body with your DotNetty / System.Net.Sockets bootstrap.
/// </summary>
public sealed class TcpServerService : IHostedService
{
    private readonly ILogger<TcpServerService> _logger;

    public TcpServerService(ILogger<TcpServerService> logger)
    {
        _logger = logger;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("TCP server starting...");
        // TODO: Initialize DotNetty ServerBootstrap or System.Net.Sockets listener here
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("TCP server stopping...");
        // TODO: Gracefully close all active connections
        return Task.CompletedTask;
    }
}
```

---

## Phase 8: Silo Host

### 8.1 `src/host/<APPNAME>.Silo/<APPNAME>.Silo.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.Silo</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\<APPNAME>.ServiceDefaults\<APPNAME>.ServiceDefaults.csproj" />
    <ProjectReference Include="..\..\grain\<APPNAME>.Grains\<APPNAME>.Grains.csproj" />
  </ItemGroup>

  <ItemGroup>
    <!-- Orleans silo: hosts grains -->
    <PackageReference Include="Microsoft.Orleans.Server" />
    <!-- Storage backend — choose one based on STORAGE_BACKEND: -->
    <!-- InMemory: built into Microsoft.Orleans.Server — no extra package needed -->
    <!-- <PackageReference Include="Microsoft.Orleans.Persistence.AdoNet" /> -->
    <!-- Clustering: -->
    <!-- <PackageReference Include="Microsoft.Orleans.Clustering.AdoNet" /> -->
    <PackageReference Include="Serilog" />
    <PackageReference Include="Serilog.Sinks.Console" />
    <PackageReference Include="Serilog.Extensions.Logging" />
  </ItemGroup>

  <ItemGroup>
    <Content Include="appsettings.json" CopyToOutputDirectory="PreserveNewest" />
  </ItemGroup>

</Project>
```

### 8.2 `src/host/<APPNAME>.Silo/Program.cs`

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Orleans.Configuration;
using Serilog;

namespace <ROOTNS>.Silo;

public static class Program
{
    public static async Task<int> Main(string[] args)
    {
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Debug()
            .WriteTo.Console()
            .CreateLogger();

        try
        {
            Log.Information("<APPNAME>.Silo starting...");

            var builder = Host.CreateDefaultBuilder(args);
            builder.UseSerilog();

            builder.UseOrleans((ctx, silo) =>
            {
                silo.Configure<ClusterOptions>(options =>
                {
                    options.ClusterId = ctx.Configuration["Orleans:ClusterId"] ?? "<APPNAME>-cluster";
                    options.ServiceId = ctx.Configuration["Orleans:ServiceId"] ?? "<APPNAME>";
                });

                // Development: in-memory clustering and storage (replace for production)
                silo.UseLocalhostClustering();
                silo.AddMemoryGrainStorage("Default");
                silo.AddMemoryGrainStorage("PubSubStore"); // required for streams

                // Production examples (uncomment as needed):
                // silo.UseAdoNetClustering(options => options.ConnectionString = ctx.Configuration.GetConnectionString("Orleans"));
                // silo.AddAdoNetGrainStorage("Default", options => { ... });

                // Enable streams (if using Orleans Streams)
                // silo.AddMemoryStreams("default");

                // Enable reminders
                silo.UseInMemoryReminderService();

                // OTEL dashboard integration via Aspire
                silo.UseDashboard(options => options.Port = 8888);
            });

            builder.ConfigureServices((ctx, services) =>
            {
                services.AddServiceDefaults();
            });

            await builder.Build().RunAsync();
            return 0;
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "<APPNAME>.Silo terminated unexpectedly");
            return 1;
        }
        finally
        {
            await Log.CloseAndFlushAsync();
        }
    }
}
```

---

## Phase 9: Aspire AppHost

### 9.1 `src/host/<APPNAME>.AppHost/<APPNAME>.AppHost.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <!-- Note: Sdk is overridden to Aspire.AppHost.Sdk in Directory.Build.props -->

  <PropertyGroup>
    <RootNamespace><ROOTNS>.AppHost</RootNamespace>
    <!-- Aspire workload sets IsAspireHost=true automatically via Aspire.AppHost.Sdk -->
  </PropertyGroup>

  <!-- Aspire Hosting packages -->
  <ItemGroup>
    <PackageReference Include="Aspire.Hosting" />
    <PackageReference Include="Aspire.Hosting.Orleans" />
  </ItemGroup>

  <!-- Reference all hosted services as Aspire project resources -->
  <ItemGroup>
    <ProjectReference Include="..\<APPNAME>.Gateway\<APPNAME>.Gateway.csproj"
                      IsAspireProjectResource="true" />
    <ProjectReference Include="..\<APPNAME>.Silo\<APPNAME>.Silo.csproj"
                      IsAspireProjectResource="true" />
  </ItemGroup>

</Project>
```

### 9.2 `src/host/<APPNAME>.AppHost/Program.cs`

```csharp
using Aspire.Hosting.Orleans;

var builder = DistributedApplication.CreateBuilder(args);

// Backing services (add as needed per STORAGE_BACKEND)
// var postgres = builder.AddPostgres("postgres").AddDatabase("orleans");

// Orleans cluster — Aspire manages cluster membership discovery
var orleans = builder.AddOrleans("<APPNAME>-cluster")
    .WithClustering()                   // Uses Aspire service discovery for silo membership
    .WithGrainStorage("Default")        // In-memory (replace with AddStorageProvider for prod)
    .WithGrainStorage("PubSubStore");   // Required if using Orleans Streams

// Silo: pure grain host
var silo = builder.AddProject<Projects.<APPNAME>_Silo>("<APPNAME>-silo")
    .WithReference(orleans)             // Injects cluster configuration into the silo's environment
    .WithReplicas(1);                   // Increase for multi-silo clusters in prod

// Gateway: client-facing host (omit if USE_GATEWAY=false)
var gateway = builder.AddProject<Projects.<APPNAME>_Gateway>("<APPNAME>-gateway")
    .WithReference(orleans)             // Allows gateway to locate silos via service discovery
    .WaitFor(silo);                     // Gateway starts after silo is healthy

builder.Build().Run();
```

> **Agent note**: `Projects.<APPNAME>_Silo` and `Projects.<APPNAME>_Gateway` are generated by Aspire's source generator from the `IsAspireProjectResource="true"` project references in the `.csproj`. The generated type name uses underscores in place of dots (e.g., `<APPNAME>.Silo` → `<APPNAME>_Silo`).

### 9.3 `src/host/<APPNAME>.ServiceDefaults/Extensions.cs`

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Trace;

namespace <ROOTNS>.ServiceDefaults;

/// <summary>
/// Extension methods for configuring Aspire service defaults across all host projects.
/// Every host calls services.AddServiceDefaults() in its DI setup.
/// </summary>
public static class Extensions
{
    public static IHostApplicationBuilder AddServiceDefaults(this IHostApplicationBuilder builder)
    {
        builder.ConfigureOpenTelemetry();
        builder.AddDefaultHealthChecks();
        builder.Services.AddServiceDiscovery();
        builder.Services.ConfigureHttpClientDefaults(http =>
        {
            http.AddStandardResilienceHandler();
            http.AddServiceDiscovery();
        });
        return builder;
    }

    private static IHostApplicationBuilder ConfigureOpenTelemetry(this IHostApplicationBuilder builder)
    {
        builder.Logging.AddOpenTelemetry(logging =>
        {
            logging.IncludeFormattedMessage = true;
            logging.IncludeScopes = true;
        });

        builder.Services.AddOpenTelemetry()
            .WithMetrics(metrics => metrics
                .AddAspNetCoreInstrumentation()
                .AddRuntimeInstrumentation())
            .WithTracing(tracing => tracing
                .AddAspNetCoreInstrumentation()
                .AddGrpcClientInstrumentation()
                .AddHttpClientInstrumentation());

        builder.AddOpenTelemetryExporters();
        return builder;
    }

    private static IHostApplicationBuilder AddOpenTelemetryExporters(this IHostApplicationBuilder builder)
    {
        var useOtlpExporter = !string.IsNullOrWhiteSpace(builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]);
        if (useOtlpExporter)
        {
            builder.Services.AddOpenTelemetry().UseOtlpExporter();
        }
        return builder;
    }

    private static IHostApplicationBuilder AddDefaultHealthChecks(this IHostApplicationBuilder builder)
    {
        builder.Services.AddHealthChecks()
            .AddCheck("self", () => HealthCheckResult.Healthy(), ["live"]);
        return builder;
    }

    public static WebApplication MapDefaultEndpoints(this WebApplication app)
    {
        app.MapHealthChecks("/health");
        app.MapHealthChecks("/alive", new HealthCheckOptions { Predicate = r => r.Tags.Contains("live") });
        return app;
    }
}
```

### 9.4 `src/host/<APPNAME>.ServiceDefaults/<APPNAME>.ServiceDefaults.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.ServiceDefaults</RootNamespace>
    <!-- ServiceDefaults is a library, not an exe -->
    <OutputType>Library</OutputType>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Hosting" />
    <PackageReference Include="Microsoft.Extensions.Http.Resilience" />
    <PackageReference Include="Microsoft.Extensions.ServiceDiscovery" />
    <PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" />
  </ItemGroup>

</Project>
```

---

## Phase 10: Test Projects

### 10.1 Orleans TestCluster (Grain Unit Tests)

TestCluster spins up a full in-process Orleans cluster — no mocking required. Tests call real grains.

#### `test/<APPNAME>.Tests.Grains/<APPNAME>.Tests.Grains.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.Tests.Grains</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\grain\<APPNAME>.GrainInterfaces\<APPNAME>.GrainInterfaces.csproj" />
    <ProjectReference Include="..\..\src\grain\<APPNAME>.Grains\<APPNAME>.Grains.csproj" />
  </ItemGroup>

  <ItemGroup>
    <!-- TestingHost provides TestCluster + TestClusterBuilder -->
    <PackageReference Include="Microsoft.Orleans.TestingHost" />
  </ItemGroup>

</Project>
```

#### Starter TestCluster Test

```csharp
// test/<APPNAME>.Tests.Grains/PlayerGrainTests.cs
using Orleans.TestingHost;
using <ROOTNS>.GrainInterfaces;

namespace <ROOTNS>.Tests.Grains;

public sealed class PlayerGrainTests : IDisposable
{
    // TestCluster is expensive to create — share across tests in this class.
    private readonly TestCluster _cluster;

    public PlayerGrainTests()
    {
        var builder = new TestClusterBuilder();
        builder.AddSiloBuilderConfigurator<TestSiloConfigurator>();
        _cluster = builder.Build();
        _cluster.Deploy();
    }

    [Test]
    public async Task GetSnapshot_NewPlayer_ReturnsDisconnectedState()
    {
        var grain = _cluster.GrainFactory.GetGrain<IPlayerGrain>(characterId: 42L);

        var snapshot = await grain.GetSnapshotAsync();

        await Assert.That(snapshot.CharacterId).IsEqualTo(42L);
        await Assert.That(snapshot.IsConnected).IsFalse();
    }

    [Test]
    public async Task ConnectAsync_SetsConnectedState()
    {
        var grain = _cluster.GrainFactory.GetGrain<IPlayerGrain>(characterId: 99L);

        await grain.ConnectAsync("session-abc");
        var snapshot = await grain.GetSnapshotAsync();

        await Assert.That(snapshot.IsConnected).IsTrue();
        await Assert.That(snapshot.SessionId).IsEqualTo("session-abc");
    }

    public void Dispose() => _cluster.StopAllSilos();

    private sealed class TestSiloConfigurator : ISiloConfigurator
    {
        public void Configure(ISiloBuilder silo)
        {
            silo.AddMemoryGrainStorage("Default");
            silo.AddMemoryGrainStorage("PubSubStore");
            silo.UseInMemoryReminderService();
        }
    }
}
```

> **TestCluster lifecycle**: `TestCluster.Deploy()` is synchronous and expensive — it boots a real Orleans silo in process. Create one cluster per test class using constructor/`IDisposable`. Never create a cluster per test method.

### 10.2 Aspire Integration Tests

Aspire 8.0+ provides `DistributedApplicationTestingBuilder` for full-stack integration tests:

#### `test/<APPNAME>.Tests.Integration/<APPNAME>.Tests.Integration.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.Tests.Integration</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <!-- Reference AppHost for DistributedApplicationTestingBuilder -->
    <ProjectReference Include="..\..\src\host\<APPNAME>.AppHost\<APPNAME>.AppHost.csproj" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Aspire.Hosting.Testing" />
  </ItemGroup>

</Project>
```

#### Starter Aspire Integration Test

```csharp
// test/<APPNAME>.Tests.Integration/AppHostTests.cs
using Aspire.Hosting.Testing;
using Microsoft.Extensions.DependencyInjection;

namespace <ROOTNS>.Tests.Integration;

public sealed class AppHostTests
{
    [Test]
    public async Task AppHost_StartsAndReportsHealthy()
    {
        await using var app = await DistributedApplicationTestingBuilder
            .CreateAsync<Projects.<APPNAME>_AppHost>();

        await using var client = app.CreateHttpClient("<APPNAME>-gateway");

        await app.StartAsync();

        var response = await client.GetAsync("/health");
        await Assert.That((int)response.StatusCode).IsEqualTo(200);
    }
}
```

> **Agent note**: Aspire integration tests launch real child processes (the silo and gateway). They require the .NET 10 SDK and the Aspire workload. On CI, ensure the workload is installed in the workflow (`dotnet workload install aspire`).

---

## Phase 11: `Build.cs` — Task Runner

**Critical pitfall**: If any forwarded command arguments begin with `-` or `--` (for example `-c`, `--no-build`, `-o`, or `--verbosity`), keep the `--` separator after `Build.cs`. `dotnet run Build.cs test -c Release` is wrong because the outer `dotnet run` consumes `-c`.

```csharp
#!/usr/bin/env dotnet
// Task runner for <APPNAME>
// Usage: dotnet run Build.cs <command> [args]

#:property PublishAot=false

using System.Diagnostics;
using System.Runtime.CompilerServices;
using System.Runtime.InteropServices;

var repoRoot = RepoRoot();
var solutionPath = Path.Combine(repoRoot, "<APPNAME>.slnx");
var coverageOutputDirectory = Path.Combine(repoRoot, "artifacts", "TestResults");
var appHostProject = Path.Combine("src", "host", "<APPNAME>.AppHost", "<APPNAME>.AppHost.csproj");
var siloProject = Path.Combine("src", "host", "<APPNAME>.Silo", "<APPNAME>.Silo.csproj");
var gatewayProject = Path.Combine("src", "host", "<APPNAME>.Gateway", "<APPNAME>.Gateway.csproj");
var grainsTestProject = Path.Combine("test", "<APPNAME>.Tests.Grains", "<APPNAME>.Tests.Grains.csproj");
var integrationTestProject = Path.Combine("test", "<APPNAME>.Tests.Integration", "<APPNAME>.Tests.Integration.csproj");
var command = args.FirstOrDefault()?.ToLowerInvariant() ?? "help";
var commandArgs = args.Skip(1).ToArray();

switch (command)
{
    case "build":
        var buildConfig = commandArgs.FirstOrDefault(a => !a.StartsWith('-')) ?? "Debug";
        Run("dotnet", ["build", solutionPath, "-c", buildConfig], repoRoot);
        return 0;

    case "test":
        Run("dotnet", ["test", "--solution", solutionPath, .. commandArgs], repoRoot);
        return 0;

    case "test-grains":
        RunTestProject(grainsTestProject, commandArgs);
        return 0;

    case "test-integration":
        RunTestProject(integrationTestProject, commandArgs);
        return 0;

    case "coverage":
        Run("dotnet", ["tool", "restore"], repoRoot);
        Directory.CreateDirectory(coverageOutputDirectory);
        Run("dotnet", ["build", solutionPath, "-c", "Release"], repoRoot);
        foreach (var testProject in FindTestProjects(repoRoot, "test"))
        {
            CollectCoverage(repoRoot, testProject, coverageOutputDirectory);
        }
        return 0;

    case "run-local":
        Run("dotnet", ["run", "--project", appHostProject], repoRoot);
        return 0;

    case "run-silo":
        Run("dotnet", ["run", "--project", siloProject], repoRoot);
        return 0;

    case "run-gateway":
        Run("dotnet", ["run", "--project", gatewayProject], repoRoot);
        return 0;

    case "publish":
        PublishHosts(commandArgs);
        return 0;

    case "format":
        Format(commandArgs);
        return 0;

    case "clean":
        DeleteIfPresent(Path.Combine(repoRoot, "artifacts"));
        DeleteIfPresent(Path.Combine(repoRoot, "publish"));
        Run("dotnet", ["clean", solutionPath], repoRoot);
        return 0;

    case "rename":
        if (commandArgs.Length < 2)
        {
            Console.Error.WriteLine("Usage: dotnet run Build.cs rename <OldName> <NewName>");
            return 1;
        }

        RenameAll(repoRoot, commandArgs[0], commandArgs[1]);
        return 0;

    default:
        Help();
        return 0;
}

void RunTestProject(string projectPath, string[] arguments)
{
    string[] forwardedArgs = arguments.Length == 0 ? ["-c", "Release", "--verbosity", "normal"] : arguments;
    Run("dotnet", ["test", projectPath, .. forwardedArgs], repoRoot);
}

void PublishHosts(string[] arguments)
{
    var nextArgumentIndex = 0;
    var rid =
        arguments.ElementAtOrDefault(nextArgumentIndex) is { } ridArg && !ridArg.StartsWith('-')
            ? arguments[nextArgumentIndex++]
            : RuntimeInformation.RuntimeIdentifier;
    var publishConfig =
        arguments.ElementAtOrDefault(nextArgumentIndex) is { } configArg && !configArg.StartsWith('-')
            ? arguments[nextArgumentIndex++]
            : "Release";
    var extraPublishArgs = arguments.Skip(nextArgumentIndex).ToArray();
    var outputBase = "publish";
    var filteredPublishArgs = new List<string>();

    for (var index = 0; index < extraPublishArgs.Length; index++)
    {
        var argument = extraPublishArgs[index];
        if ((string.Equals(argument, "-o", StringComparison.OrdinalIgnoreCase)
            || string.Equals(argument, "--output", StringComparison.OrdinalIgnoreCase))
            && index + 1 < extraPublishArgs.Length)
        {
            outputBase = extraPublishArgs[++index];
            continue;
        }

        filteredPublishArgs.Add(argument);
    }

    PublishHost(siloProject, rid, publishConfig, Path.Combine(outputBase, "silo"), filteredPublishArgs);
    PublishHost(gatewayProject, rid, publishConfig, Path.Combine(outputBase, "gateway"), filteredPublishArgs);
}

void PublishHost(string projectPath, string rid, string publishConfig, string outputDirectory, IEnumerable<string> extraArgs)
{
    List<string> publishArgs =
    [
        "publish",
        projectPath,
        "-c",
        publishConfig,
        "-r",
        rid,
        "-o",
        outputDirectory,
    ];

    publishArgs.AddRange(extraArgs);
    Run("dotnet", publishArgs, repoRoot);
}

void Format(string[] arguments)
{
    var verify = arguments.Length > 0 && string.Equals(arguments[0], "check", StringComparison.OrdinalIgnoreCase);
    Run("dotnet", ["tool", "restore"], repoRoot);
    Run(
        "dotnet",
        verify ? ["tool", "run", "csharpier", "--", "check", "."] : ["tool", "run", "csharpier", "--", "format", "."],
        repoRoot
    );

    foreach (var project in ProjectFiles(repoRoot))
    {
        var projectRelativePath = NormalizeRelativePath(Path.GetRelativePath(repoRoot, project));
        string[] extraFormatArgs = verify ? ["--no-restore", "--verify-no-changes"] : ["--no-restore"];
        Run("dotnet", ["format", "style", projectRelativePath, .. extraFormatArgs], repoRoot);
        Run("dotnet", ["format", "analyzers", projectRelativePath, .. extraFormatArgs], repoRoot);
    }
}

void Help()
{
    Console.WriteLine(
        @"Usage: dotnet run Build.cs <command> [args]
       dotnet run Build.cs -- <command> [args]   (use -- when forwarding option-style args)

Commands:
    build [config]                         Build the solution (default: Debug)
    test [args]                            Run solution tests with forwarded dotnet test args
    test-grains [args]                     Run grain tests with forwarded dotnet test args
    test-integration [args]                Run integration tests with forwarded dotnet test args
    coverage                               Collect Cobertura coverage into artifacts/TestResults/
    run-local                              Start the Aspire AppHost
    run-silo                               Run the silo standalone
    run-gateway                            Run the gateway standalone
    publish [rid] [config] [args]          Publish silo and gateway artifacts
    format [check]                         Run CSharpier plus dotnet format style/analyzers
    clean                                  Delete artifacts and publish output, then clean the solution
    rename <OldName> <NewName>             Rename template throughout repository
    help                                   Show this command list"
    );
}

static void DeleteIfPresent(string path)
{
    if (!Directory.Exists(path))
        return;

    Directory.Delete(path, recursive: true);
    Console.WriteLine($"Deleted {path}");
}

static IEnumerable<string> ProjectFiles(string repoRoot) =>
    Directory
        .EnumerateFiles(repoRoot, "*.csproj", SearchOption.AllDirectories)
        .Where(path => !path.Contains($"{Path.DirectorySeparatorChar}bin{Path.DirectorySeparatorChar}", StringComparison.OrdinalIgnoreCase))
        .Where(path => !path.Contains($"{Path.DirectorySeparatorChar}obj{Path.DirectorySeparatorChar}", StringComparison.OrdinalIgnoreCase));

static IEnumerable<string> FindTestProjects(string repoRoot, string relativeDirectory)
{
    var testRoot = Path.Combine(repoRoot, relativeDirectory);
    if (!Directory.Exists(testRoot))
    {
        yield break;
    }

    foreach (var projectPath in Directory
        .GetFiles(testRoot, "*.csproj", SearchOption.AllDirectories)
        .OrderBy(path => path, StringComparer.OrdinalIgnoreCase))
    {
        yield return projectPath;
    }
}

static void CollectCoverage(string repoRoot, string projectPath, string coverageOutputDirectory)
{
    var projectName = Path.GetFileNameWithoutExtension(projectPath);
    var projectRelativePath = NormalizeRelativePath(Path.GetRelativePath(repoRoot, projectPath));
    var outputPath = Path.Combine(coverageOutputDirectory, $"{projectName}.coverage.cobertura.xml");

    Run("dotnet",
        [
            "dotnet-coverage",
            "collect",
            $"dotnet test {QuoteCommandValue(projectRelativePath)} -c Release --no-build --no-restore --verbosity normal",
            "--output",
            outputPath,
            "--output-format",
            "cobertura",
        ],
        repoRoot);
}

static void Run(string executable, IEnumerable<string> arguments, string workingDirectory)
{
    var argumentList = arguments.ToArray();
    Console.WriteLine();
    Console.WriteLine($"> {executable} {string.Join(' ', argumentList.Select(EscapeArgument))}");

    var processStartInfo = new ProcessStartInfo(executable)
    {
        WorkingDirectory = workingDirectory,
        UseShellExecute = false,
    };

    foreach (var argument in argumentList)
    {
        processStartInfo.ArgumentList.Add(argument);
    }

    using var process = Process.Start(processStartInfo)
        ?? throw new InvalidOperationException($"Failed to start '{executable}'.");
    process.WaitForExit();
    if (process.ExitCode != 0)
    {
        throw new InvalidOperationException(
            $"Command failed with exit code {process.ExitCode}: {executable}");
    }
}

static string EscapeArgument(string argument) =>
    argument.Contains(' ', StringComparison.Ordinal) || argument.Contains('"', StringComparison.Ordinal)
        ? $"\"{argument.Replace("\"", "\\\"", StringComparison.Ordinal)}\""
        : argument;

static string QuoteCommandValue(string value) =>
    value.Contains(' ', StringComparison.Ordinal) ? $"\"{value}\"" : value;

static string NormalizeRelativePath(string path) => path.Replace('\\', '/');

static void RenameAll(string repoRoot, string oldName, string newName)
{
    var ignore = new[] {
        Path.DirectorySeparatorChar + ".git" + Path.DirectorySeparatorChar,
        Path.DirectorySeparatorChar + "artifacts" + Path.DirectorySeparatorChar,
        Path.DirectorySeparatorChar + "publish" + Path.DirectorySeparatorChar,
    };

    var files = Directory.EnumerateFiles(repoRoot, "*", SearchOption.AllDirectories)
        .Where(f => !ignore.Any(seg => f.Contains(seg)))
        .ToList();

    var textExtensions = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
    {
        ".cs", ".csproj", ".props", ".targets", ".json", ".xml",
        ".yml", ".yaml", ".md", ".txt", ".slnx", ".sln", ".slnf",
        ".editorconfig", ".gitignore", ".gitattributes", ".config",
    };

    foreach (var file in files)
    {
        if (!textExtensions.Contains(Path.GetExtension(file))) { continue; }
        string content;
        try { content = File.ReadAllText(file); }
        catch { continue; }
        if (content.Contains(oldName))
            File.WriteAllText(file, content.Replace(oldName, newName));

        var name = Path.GetFileName(file);
        if (name.Contains(oldName))
        {
            var newPath = Path.Combine(Path.GetDirectoryName(file)!, name.Replace(oldName, newName));
            File.Move(file, newPath);
        }
    }

    var dirs = Directory.EnumerateDirectories(repoRoot, "*", SearchOption.AllDirectories)
        .Where(d => Path.GetFileName(d).Contains(oldName)
                 && !ignore.Any(seg => d.Contains(seg)))
        .OrderByDescending(d => d.Length)
        .ToList();

    foreach (var dir in dirs)
    {
        var parent = Path.GetDirectoryName(dir)!;
        Directory.Move(dir, Path.Combine(parent, Path.GetFileName(dir).Replace(oldName, newName)));
    }

    Console.WriteLine($"Renamed '{oldName}' to '{newName}' throughout repository.");
}

static string RepoRoot([CallerFilePath] string path = "") =>
    Path.GetDirectoryName(path)!;
```

---

## Phase 12: Solution Organization

### 12.1 `<APPNAME>.slnx` — With Solution Folders

```xml
<Solution>

  <!-- ═══════════ Hosts ═══════════ -->
  <Folder Name="/Hosts/">
    <Project Path="src/host/<APPNAME>.AppHost/<APPNAME>.AppHost.csproj" />
    <Project Path="src/host/<APPNAME>.ServiceDefaults/<APPNAME>.ServiceDefaults.csproj" />
    <Project Path="src/host/<APPNAME>.Gateway/<APPNAME>.Gateway.csproj" />
    <Project Path="src/host/<APPNAME>.Silo/<APPNAME>.Silo.csproj" />
  </Folder>

  <!-- ═══════════ Grains ═══════════ -->
  <Folder Name="/Grain/">
    <Project Path="src/grain/<APPNAME>.GrainInterfaces/<APPNAME>.GrainInterfaces.csproj" />
    <Project Path="src/grain/<APPNAME>.Grains/<APPNAME>.Grains.csproj" />
  </Folder>

  <!-- ═══════════ Common ═══════════ -->
  <Folder Name="/Common/">
    <Project Path="src/common/<APPNAME>.Common/<APPNAME>.Common.csproj" />
    <Project Path="src/common/<APPNAME>.Common.Persistence/<APPNAME>.Common.Persistence.csproj" />
  </Folder>

  <!-- ═══════════ Protocol ═══════════ -->
  <Folder Name="/Protocol/">
    <Project Path="src/protocol/<APPNAME>.Protocol/<APPNAME>.Protocol.csproj" />
  </Folder>

  <!-- ═══════════ Tests ═══════════ -->
  <Folder Name="/Tests/">
    <Project Path="test/<APPNAME>.Tests.Grains/<APPNAME>.Tests.Grains.csproj" />
    <Project Path="test/<APPNAME>.Tests.Integration/<APPNAME>.Tests.Integration.csproj" />
  </Folder>

  <!-- ═══════════ Benchmarks ═══════════ -->
  <Folder Name="/Benchmarks/">
    <Project Path="benchmarks/<APPNAME>.Benchmarks/<APPNAME>.Benchmarks.csproj" />
  </Folder>

</Solution>
```

### 12.2 Solution Filters (`.slnf`)

**`Grains.slnf`** — Grain layer + tests only (fastest for grain development):

```json
{
  "solution": {
    "path": "<APPNAME>.slnx",
    "projects": [
      "src/common/<APPNAME>.Common/<APPNAME>.Common.csproj",
      "src/grain/<APPNAME>.GrainInterfaces/<APPNAME>.GrainInterfaces.csproj",
      "src/grain/<APPNAME>.Grains/<APPNAME>.Grains.csproj",
      "test/<APPNAME>.Tests.Grains/<APPNAME>.Tests.Grains.csproj"
    ]
  }
}
```

**`Silo.slnf`** — Silo + grains + dependencies:

```json
{
  "solution": {
    "path": "<APPNAME>.slnx",
    "projects": [
      "src/common/<APPNAME>.Common/<APPNAME>.Common.csproj",
      "src/common/<APPNAME>.Common.Persistence/<APPNAME>.Common.Persistence.csproj",
      "src/grain/<APPNAME>.GrainInterfaces/<APPNAME>.GrainInterfaces.csproj",
      "src/grain/<APPNAME>.Grains/<APPNAME>.Grains.csproj",
      "src/host/<APPNAME>.ServiceDefaults/<APPNAME>.ServiceDefaults.csproj",
      "src/host/<APPNAME>.Silo/<APPNAME>.Silo.csproj"
    ]
  }
}
```

**`Gateway.slnf`** — Gateway + interfaces + dependencies:

```json
{
  "solution": {
    "path": "<APPNAME>.slnx",
    "projects": [
      "src/common/<APPNAME>.Common/<APPNAME>.Common.csproj",
      "src/protocol/<APPNAME>.Protocol/<APPNAME>.Protocol.csproj",
      "src/grain/<APPNAME>.GrainInterfaces/<APPNAME>.GrainInterfaces.csproj",
      "src/host/<APPNAME>.ServiceDefaults/<APPNAME>.ServiceDefaults.csproj",
      "src/host/<APPNAME>.Gateway/<APPNAME>.Gateway.csproj"
    ]
  }
}
```

---

## Phase 13: CI/CD Pipeline

### 13.1 `.github/workflows/dotnet.yml`

```yaml
# yaml-language-server: $schema=https://json.schemastore.org/github-workflow.json

name: dotnet

permissions: read-all

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]
  workflow_dispatch:

env:
  DOTNET_SKIP_FIRST_TIME_EXPERIENCE: 1
  DOTNET_NOLOGO: true

jobs:
  build-and-test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        configuration: [Debug, Release]

    runs-on: ${{ matrix.os }}

    steps:
      - name: Harden Runner
        uses: step-security/harden-runner@9af89fc71515a100421586dfdb3dc9c984fbf411 # v2.19.4
        with:
          egress-policy: audit

      - uses: actions/checkout@df4cb1c069e1874edd31b4311f1884172cec0e10 # v6.0.3

      - name: Setup .NET (global.json)
        uses: actions/setup-dotnet@9a946fdbd5fb07b82b2f5a4466058b876ab72bb2 # v5.3.0
        with:
          global-json-file: global.json

      - name: Install Aspire workload
        run: dotnet workload install aspire

      - name: Build
        run: dotnet run Build.cs build ${{ matrix.configuration }}

      - name: Test (grain unit tests)
        run: >
          dotnet run Build.cs --
          test-grains
          -c ${{ matrix.configuration }}
          --no-build
          --verbosity normal

      # Integration tests (Aspire) run only on ubuntu-latest to reduce cost.
      # They boot real processes and require Docker or the Aspire workload.
      - name: Test (integration)
        if: matrix.os == 'ubuntu-latest' && matrix.configuration == 'Debug'
        run: >
          dotnet run Build.cs --
          test-integration
          -c ${{ matrix.configuration }}
          --no-build
          --verbosity normal

  coverage:
    needs: build-and-test
    runs-on: ubuntu-latest
    steps:
      - name: Harden Runner
        uses: step-security/harden-runner@9af89fc71515a100421586dfdb3dc9c984fbf411 # v2.19.4
        with:
          egress-policy: audit

      - uses: actions/checkout@df4cb1c069e1874edd31b4311f1884172cec0e10 # v6.0.3
        with:
          fetch-depth: 0

      - name: Setup .NET (global.json)
        uses: actions/setup-dotnet@9a946fdbd5fb07b82b2f5a4466058b876ab72bb2 # v5.3.0
        with:
          global-json-file: global.json

      - name: Install Aspire workload
        run: dotnet workload install aspire

      - name: Collect coverage
        run: dotnet run Build.cs coverage

      - name: Upload coverage artifacts
        uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
        with:
          name: coverage-reports
          if-no-files-found: error
          path: artifacts/TestResults/*.coverage.cobertura.xml

      - name: Upload coverage to Codecov
        if: ${{ env.CODECOV_TOKEN != '' }}
        env:
          CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
        uses: codecov/codecov-action@fb8b3582c8e4def4969c97caa2f19720cb33a72f # v7.0.0
        with:
          token: ${{ env.CODECOV_TOKEN }}
          files: artifacts/TestResults/*.coverage.cobertura.xml
          disable_search: true
          fail_ci_if_error: true
          verbose: true

  format:
    runs-on: ubuntu-latest
    steps:
      - name: Harden Runner
        uses: step-security/harden-runner@9af89fc71515a100421586dfdb3dc9c984fbf411 # v2.19.4
        with:
          egress-policy: audit

      - uses: actions/checkout@df4cb1c069e1874edd31b4311f1884172cec0e10 # v6.0.3

      - name: Setup .NET
        uses: actions/setup-dotnet@9a946fdbd5fb07b82b2f5a4466058b876ab72bb2 # v5.3.0
        with:
          global-json-file: global.json

      - name: Verify formatting
        run: dotnet run Build.cs format check

  publish:
    needs: [build-and-test, coverage, format]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    steps:
      - name: Harden Runner
        uses: step-security/harden-runner@9af89fc71515a100421586dfdb3dc9c984fbf411 # v2.19.4
        with:
          egress-policy: audit

      - uses: actions/checkout@df4cb1c069e1874edd31b4311f1884172cec0e10 # v6.0.3

      - name: Setup .NET
        uses: actions/setup-dotnet@9a946fdbd5fb07b82b2f5a4466058b876ab72bb2 # v5.3.0
        with:
          global-json-file: global.json

      - name: Install Aspire workload
        run: dotnet workload install aspire

      - name: Set version
        id: version
        run: echo "version=0.0.0-ci.${{ github.run_number }}" >> $GITHUB_OUTPUT

      # Publish each host as a self-contained linux-x64 artifact
      - name: Publish hosts
        run: >
          dotnet run Build.cs --
          publish
          linux-x64
          Release
          --self-contained
          -p:Version=${{ steps.version.outputs.version }}

      - name: Upload Silo artifact
        uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
        with:
          name: <APPNAME>-silo-linux-x64
          if-no-files-found: error
          retention-days: 14
          path: publish/silo/

      - name: Upload Gateway artifact
        uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
        with:
          name: <APPNAME>-gateway-linux-x64
          if-no-files-found: error
          retention-days: 14
          path: publish/gateway/
```

**Key differences from the monolith CI**:

- `dotnet workload install aspire` step is required on every runner
- Grain unit tests and Aspire integration tests are separated — integration tests run only on ubuntu/Debug to reduce cost
- Publish builds each host separately (`Silo`, `Gateway`) as independent artifacts
- Coverage runs in a dedicated ubuntu job; Codecov upload happens only when `CODECOV_TOKEN` is configured
- No NuGet pack/push — this is a distributed application, not a library

**Agent note — resolving `<PINNED-SHA>` placeholders**: The SHA lookup table is pre-populated:

| Action                        | Pinned to version | SHA                                        | Last verified |
| ----------------------------- | ----------------- | ------------------------------------------ | ------------- |
| `step-security/harden-runner` | v2.19.4           | `9af89fc71515a100421586dfdb3dc9c984fbf411` | 2026-06-10    |
| `actions/checkout`            | v6.0.3            | `df4cb1c069e1874edd31b4311f1884172cec0e10` | 2026-06-10    |
| `actions/setup-dotnet`        | v5.3.0            | `9a946fdbd5fb07b82b2f5a4466058b876ab72bb2` | 2026-06-10    |
| `codecov/codecov-action`      | v7.0.0            | `fb8b3582c8e4def4969c97caa2f19720cb33a72f` | 2026-06-10    |
| `actions/upload-artifact`     | v7.0.1            | `043fb46d1a93c77aae656e7c1c64a875d1fc6a0a` | 2026-06-10    |
| `actions/download-artifact`   | v8.0.1            | `3e5f45b2cfb9172054b4087a40e8e0b5a5461e7c` | 2026-06-10    |

If any of these are stale at scaffold time, resolve the latest SHA via:

```shell
git ls-remote --tags https://github.com/actions/checkout | tail -1
# Or visit: https://github.com/actions/checkout/releases
```

### 13.2 `.github/dependabot.yml`

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    commit-message:
      prefix: "ci"
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "monthly"
    commit-message:
      prefix: "deps"
```

---

## Phase 14: Validation Checklist

Before declaring the scaffold complete, verify:

### Build and Test

- [ ] `dotnet workload install aspire` succeeds on the development machine
- [ ] `dotnet run Build.cs build Debug` passes with zero warnings
- [ ] `dotnet run Build.cs build Release` passes with zero warnings
- [ ] `dotnet run Build.cs -- test-grains -c Release --verbosity normal` passes (all grain tests)
- [ ] `dotnet run Build.cs -- test-integration -c Debug --verbosity normal` passes when Aspire prerequisites are available
- [ ] `dotnet run Build.cs -- test -c Release` passes
- [ ] README, CONTRIBUTING, AGENTS, and workflow commands that forward option-style args to `Build.cs` use the `dotnet run Build.cs -- ...` form
- [ ] `dotnet run Build.cs coverage` writes Cobertura XML to `artifacts/TestResults/`
- [ ] `dotnet run Build.cs format check` passes (runs both CSharpier + dotnet format)

### Local Run

- [ ] `dotnet run Build.cs run-local` starts the Aspire AppHost without errors
- [ ] The Aspire dashboard is accessible at `http://localhost:18888` (or the configured port)
- [ ] Silo appears healthy in the Aspire dashboard
- [ ] Gateway appears healthy (if applicable)
- [ ] Grain telemetry (OTEL traces) appears in the dashboard

### Grain Contract Verification

- [ ] All grain interfaces derive from `IGrainWith*Key` (not plain `interface`)
- [ ] All grain DTOs have `[GenerateSerializer]` and `[Id(n)]` on every field/property
- [ ] No `[Id(n)]` values are reused or renumbered within a single type
- [ ] `GrainInterfaces` has zero `ProjectReference` to `Grains` (no circular dependency)
- [ ] `Grains` has a `ProjectReference` to `GrainInterfaces` (one-way)

### Orleans Serialization

- [ ] `dotnet build` passes with no `ORLEANS_` prefixed build errors (serializer generation)
- [ ] TestCluster tests pass (confirms serializer round-trips work)
- [ ] No `[GenerateSerializer]` on types that already have a `[Serializer]`

### Solution Structure

- [ ] `.slnx` lists every `.csproj` in the repository
- [ ] Solution folders match the `src/` and `test/` directory layout
- [ ] AppHost references hosts via `IsAspireProjectResource="true"` (not regular `ProjectReference`)
- [ ] `Directory.Packages.props` has no placeholder versions — all resolved to real values
- [ ] `GrainInterfaces` does NOT reference `Grains`

### CI and Repository Health

- [ ] All action SHAs in `.github/workflows/dotnet.yml` are 40-char hashes
- [ ] `.github/dependabot.yml` exists with both `github-actions` and `nuget` ecosystems
- [ ] `SECURITY.md` exists
- [ ] `CONTRIBUTING.md` exists
- [ ] `CHANGELOG.md` exists
- [ ] `.config/dotnet-tools.json` exists
- [ ] `.github/ISSUE_TEMPLATE/` and `PULL_REQUEST_TEMPLATE.md` exist

### Task Runner

- [ ] `dotnet run Build.cs help` prints the command list without errors
- [ ] `dotnet run Build.cs rename <APPNAME> TestRename` runs without errors; revert with `dotnet run Build.cs rename TestRename <APPNAME>`

### Solution Filters

- [ ] Each `.slnf` opens in Visual Studio / Rider without errors
- [ ] Each `.slnf` includes all transitive `ProjectReference` dependencies

---

## Complete File Manifest

```
<APPNAME>/                                              ← repo root
├── .config/
│   └── dotnet-tools.json
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   └── feature_request.md
│   ├── workflows/
│   │   └── dotnet.yml
│   ├── dependabot.yml
│   └── PULL_REQUEST_TEMPLATE.md
│
├── src/
│   ├── Directory.Build.props                           ← Tier 2a (all source)
│   ├── Directory.Build.targets
│   │
│   ├── host/
│   │   ├── Directory.Build.props                       ← Tier 3 (OutputType=Exe)
│   │   ├── <APPNAME>.AppHost/
│   │   │   ├── Directory.Build.props                   ← Aspire.AppHost.Sdk override
│   │   │   ├── <APPNAME>.AppHost.csproj
│   │   │   └── Program.cs
│   │   ├── <APPNAME>.ServiceDefaults/
│   │   │   ├── <APPNAME>.ServiceDefaults.csproj
│   │   │   └── Extensions.cs
│   │   ├── <APPNAME>.Gateway/
│   │   │   ├── <APPNAME>.Gateway.csproj
│   │   │   ├── Program.cs
│   │   │   ├── Network/
│   │   │   │   └── TcpServerService.cs
│   │   │   └── appsettings.json
│   │   └── <APPNAME>.Silo/
│   │       ├── <APPNAME>.Silo.csproj
│   │       ├── Program.cs
│   │       └── appsettings.json
│   │
│   ├── grain/
│   │   ├── <APPNAME>.GrainInterfaces/
│   │   │   ├── <APPNAME>.GrainInterfaces.csproj
│   │   │   ├── IPlayerGrain.cs
│   │   │   ├── IFieldGrain.cs
│   │   │   └── Models/
│   │   │       ├── PlayerSnapshot.cs
│   │   │       └── FieldSnapshot.cs
│   │   └── <APPNAME>.Grains/
│   │       ├── <APPNAME>.Grains.csproj
│   │       ├── PlayerGrain.cs
│   │       ├── FieldGrain.cs
│   │       └── Storage/
│   │           └── RepositoryGrainStorage.cs
│   │
│   ├── common/
│   │   ├── <APPNAME>.Common/
│   │   │   └── <APPNAME>.Common.csproj
│   │   └── <APPNAME>.Common.Persistence/
│   │       ├── <APPNAME>.Common.Persistence.csproj
│   │       └── AppDbContext.cs
│   │
│   └── protocol/
│       └── <APPNAME>.Protocol/
│           └── <APPNAME>.Protocol.csproj
│
├── test/
│   ├── Directory.Build.props                           ← Tier 2b (all tests)
│   ├── <APPNAME>.Tests.Grains/
│   │   ├── <APPNAME>.Tests.Grains.csproj
│   │   └── PlayerGrainTests.cs
│   └── <APPNAME>.Tests.Integration/
│       ├── <APPNAME>.Tests.Integration.csproj
│       └── AppHostTests.cs
│
├── benchmarks/ (optional)
│   ├── Directory.Build.props
│   └── <APPNAME>.Benchmarks/
│       ├── <APPNAME>.Benchmarks.csproj
│       ├── Program.cs
│       └── GrainBench.cs
│
├── .editorconfig
├── .csharpierrc.json
├── .gitattributes
├── .gitignore
├── .jscpd.json
├── .markdownlint.json
├── .pre-commit-config.yaml
├── Build.cs
├── CHANGELOG.md
├── CONTRIBUTING.md
├── Grains.slnf
├── Silo.slnf
├── Gateway.slnf
├── Directory.Build.props                               ← Tier 1 (root)
├── Directory.Packages.props
├── global.json
├── LICENSE
├── nuget.config
├── README.md
├── SECURITY.md
└── <APPNAME>.slnx
```

**Total**: ~55 files across ~30 directories for the reference architecture (1 AppHost, 1 ServiceDefaults, 1 Gateway, 1 Silo, 2 grain projects, 2 common projects, 1 protocol project, 2 test projects). Scales to 50+ projects by adding more grains, more common libraries, and more specialized hosts (e.g., a separate worker silo, an admin API host).

---

## Decision Table: Common Customizations

| Scenario | Change |
| -------- | ------ |
| No gateway (internal-only system) | Remove `<APPNAME>.Gateway/`; remove from AppHost `AddProject<>()` call; delete `Gateway.slnf` |
| Multi-silo cluster | Increase `WithReplicas(n)` in AppHost; replace `UseLocalhostClustering()` with a real provider (AdoNet, AzureStorage) |
| PostgreSQL storage backend | Add `Orleans.Persistence.AdoNet` + `Npgsql` to `Directory.Packages.props`; replace `AddMemoryGrainStorage("Default")` with `AddAdoNetGrainStorage`; add `Orleans.Clustering.AdoNet` |
| MongoDB storage backend | Add `Orleans.Persistence.MongoDB`; configure in Silo `Program.cs` |
| Orleans Streams (pub/sub between grains) | Add `Microsoft.Orleans.Streaming` to Grains; `AddMemoryStreams("default")` in Silo; `AddOrleans(...).WithStreaming()` in AppHost |
| Orleans Reminders (durable timers) | Built-in in `Microsoft.Orleans.Reminders`; `UseInMemoryReminderService()` in Silo for dev; `UseAdoNetReminderService()` for prod |
| Orleans Transactions (ACID across grains) | Add `Microsoft.Orleans.Transactions`; annotate methods with `[Transaction(TransactionOption.Create)]`; configure in Silo |
| gRPC gateway instead of TCP | Use `Grpc.AspNetCore` in Gateway; `Microsoft.NET.Sdk.Web` SDK; generate `.proto` contracts from `<APPNAME>.Protocol/` |
| Akka.NET instead of Orleans | Replace `grain/` with `actors/` (`<APPNAME>.Messages/` + `<APPNAME>.Actors/`); use `Akka.Hosting` for DI; `TestKit` for tests; `Akka.Cluster.Hosting` for distributed deployment |
| Aspire with Docker Compose | Add `builder.AddDockerfile(...)` in AppHost for database services; use `AddConnectionString` to inject connection strings |
| Multiple silo types (different grain sets) | Create additional silo projects (`<APPNAME>.Silo.Analytics/`); each references only the grain DLLs it needs; prevents loading unnecessary grains |
| Orleans grain versioning for rolling upgrades | Add `[Version(n)]` attribute to grain interfaces; configure `VersioningOptions` in Silo; test with `TestClusterBuilder` and `ISiloBuilder.ConfigureApplicationParts` |
| Admin / dashboard API | Add `<APPNAME>.Admin/` host under `src/host/`; register as Aspire project; uses `IClusterClient` (Orleans client) to query grain state |
| Plugin extension system | Add `<APPNAME>.Sdk/` library with Orleans grain interface contracts; mark as `<IsPackable>true</IsPackable>`; external plugins reference this via NuGet |
| Docker image builds | Add `Dockerfile` per host at `src/host/<APPNAME>.Silo/Dockerfile`; use `builder.AddDockerfile("silo", "src/host/<APPNAME>.Silo")` in AppHost |
| Windows deployment | Add `win-x64` to publish matrix; ensure `UseLocalhostClustering` is replaced for multi-machine |
| CI runs too slowly | Run grain unit tests on all matrix; run Aspire integration tests only on `ubuntu-latest:Debug` (already in template) |
| Orleans dashboard | `silo.UseDashboard(options => options.Port = 8888)` in Silo `Program.cs`; already included in template |
| Need code coverage | TUnit bundles `Microsoft.Testing.Extensions.CodeCoverage` — no extra packages. Add `--coverage --coverage-output-format cobertura` to `dotnet test` steps in CI. Do NOT use `coverlet.collector` — it is a VSTest data collector and incompatible with TUnit's MTP runner. |

---

## Optional parameters and nullability

1. **Optional ≠ nullable.** `param = default` means callers may omit it; `T?` means the value may be null. Do not conflate the two contracts.

2. **Do not add `= null` to avoid updating call sites.** If the dependency/value is actually required, make callers pass it and fix every compile error.

3. **Nullable annotations are API contracts.** Use `T` for required non-null values, `T?` only when null is explicitly supported and tested.

4. **Every nullable parameter creates a handling obligation.** If `ILogger? logger = null`, every implementation path must safely handle `null`; otherwise the API is lying.

5. **Prefer overloads for convenience.** Use `DoThing()` delegating to `DoThing(requiredLogger)` or a well-defined default, instead of spreading null checks through the code.

6. **Prefer Null Object/default implementations over null.** Example: use `NullLogger<T>.Instance` internally when “no logging” is a valid behavior.

7. **Do not make services/dependencies optional by default.** Constructor-injected dependencies, loggers, clients, repositories, clocks, etc. should usually be required.

8. **Optional parameter defaults are versioning hazards.** In C#, default values are baked into call sites; changing the default may not affect already-compiled callers.

9. **Only use optional parameters for stable, obvious defaults.** Good examples: flags, timeouts, `CancellationToken cancellationToken = default`; bad examples: hidden dependencies or required business data.

10. **When adding a parameter, decide deliberately:** required → update all call sites; truly optional → define exact default behavior; nullable → document and test null behavior.

## Anti-Patterns to Avoid

1. **Do not put grain implementations in `GrainInterfaces`**. Interfaces define contracts; implementations depend on them. Reversing this creates a circular reference and forces the Gateway to compile grain implementation code it should never see.

2. **Do not use regular `<ProjectReference>` in AppHost for hosted services**. Always use `IsAspireProjectResource="true"`. Regular references pull in all transitive dependencies at compile time and destroy incremental build performance.

3. **Do not call another grain from within `OnActivateAsync` unless absolutely necessary**. Grain-to-grain calls during activation can deadlock if both grains activate simultaneously and call each other. Load data from repositories; call other grains from normal methods only.

4. **Do not change `[Id(n)]` values in existing DTOs**. Orleans serializers use the field IDs to round-trip data stored in grain state. Renumbering or reusing IDs silently corrupts deserialization of existing persisted state. Add new fields with new IDs; never reorder.

5. **Do not rely solely on `OnDeactivateAsync` for persistence**. It is best-effort — Orleans does NOT call it on silo crash. Use a periodic persist timer (`RegisterGrainTimer`) + persist-on-significant-event as the primary strategy.

6. **Do not inject `IClusterClient` into a grain**. Use `GrainFactory` (auto-injected into the grain base class) for grain-to-grain calls. `IClusterClient` is for external callers (gateways, clients). Injecting it into a grain creates an implicit external call that bypasses grain activation and can cause reentrancy issues.

7. **Do not share a `TestCluster` across test classes**. `TestCluster.Deploy()` creates a real in-process silo. Sharing across classes causes test isolation failures (grain state leaks between test runs). Create one cluster per class; dispose in `IDisposable.Dispose()`.

8. **Do not put `appsettings.json` in `GrainInterfaces` or `Grains`**. Configuration belongs in host projects (`Silo`, `Gateway`). Grain libraries are configuration-agnostic — they receive configuration via injected `IOptions<T>` from the host.

9. **Do not skip Central Package Management**. Orleans, Aspire, and EF Core each carry multiple packages with tightly coupled version requirements. Version drift across 20+ projects causes cryptic runtime failures. CPM is mandatory from day one.

10. **Do not create more silo projects than necessary**. Start with one `Silo` project. Split only when you need different grain sets, different storage configurations, or different placement strategies for performance isolation.

11. **Do not skip the `dotnet workload install aspire` step**. The Aspire AppHost SDK is not part of the base .NET 10 SDK — without the workload installed, AppHost will fail to build with cryptic SDK resolution errors.

12. **Do not run bare `dotnet format --verify-no-changes`**. The `whitespace` sub-check rewrites CSharpier's Allman-style brace formatting, causing CI failures on correctly formatted code. Always split into `dotnet format style --verify-no-changes` + `dotnet format analyzers --verify-no-changes`.
