# Agent Guide: Creating a "nietras-quality" Initial .NET Library Solution

> **Purpose**: Step-by-step instructions for an AI agent to scaffold a new .NET library solution (one or more NuGet-publishable libraries) matching the engineering standards demonstrated in the `nietras/CudaSharp` and `nietras/Sep` repositories.
> **Inputs required from user before starting**: library name, one-sentence description, primary namespace, target audience, whether AOT compatibility is required, whether the library wraps a native library.
> **Output**: a complete, CI-ready, NuGet-publishable repository scaffold with zero-tolerance code quality from commit one.

---

## Phase 0: Gather Inputs

Ask the user for (or infer from context):

| Variable          | Example                          | Notes                                                                                                                        |
| ----------------- | -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `LIBNAME`         | `CudaSharp`                      | PascalCase; becomes project name and NuGet package ID                                                                        |
| `ROOTNS`          | `CudaSharp`                      | Usually same as LIBNAME                                                                                                      |
| `DESCRIPTION`     | `Low-level CUDA interop in C#`   | One sentence, goes in .csproj and README                                                                                     |
| `AUTHOR`          | `yourname`                       | GitHub username                                                                                                              |
| `DOTNET_VERSION`  | `net10.0`                        | Usually latest; use `net8.0` for LTS requirement                                                                             |
| `CSHARP_VERSION`  | `14.0`                           | Match to chosen .NET version                                                                                                 |
| `SDK_VERSION`     | `10.0.103`                       | Exact SDK version from `dotnet --version`                                                                                    |
| `NEEDS_UNSAFE`    | `true`                           | true for P/Invoke, bit manipulation, SIMD                                                                                    |
| `NEEDS_AOT`       | `true`                           | true for any library intended for NativeAOT use                                                                              |
| `WRAPS_NATIVE`    | `true`                           | true if P/Invoking a native .dll/.so                                                                                         |
| `GITHUB_URL`      | `https://github.com/you/LIBNAME` | Full repo URL                                                                                                                |
| `LICENSE`         | `MIT`                            | MIT for open source; set appropriately                                                                                       |
| `YEAR`            | `2026`                           | Copyright year                                                                                                               |
| `COMPANY`         | `yourname`                       | Used in copyright                                                                                                            |
| `RootClass`       | `CudaSharp`                      | Primary entry-point class name; PascalCase placeholder used as `<RootClass>` throughout templates; usually same as `LIBNAME` |
| `DOTNET_CONSTANT` | `NET10_0`                        | MSBuild preprocessor symbol matching `DOTNET_VERSION` (`net10.0`→`NET10_0`, `net9.0`→`NET9_0`, `net8.0`→`NET8_0`)            |

---

## Phase 0b: Initialize Git Repository

Before creating any files, initialize the repository. MinVer, `.gitattributes`, and the validation checklist all require a git repo:

```shell
mkdir <LIBNAME> && cd <LIBNAME>
git init -b main
```

**Verify git authorship** before the first commit — local git config can override the global setting and produce commits attributed to a placeholder account:

```shell
git config user.name   # must print the real name, not a placeholder
git config user.email  # must print the real email / GitHub-linked address
```

If the output is wrong (e.g., shows a machine hostname, "bia10@github.com", or is empty), fix it before committing:

```shell
# Fix locally (this repo only):
git config user.name  "Your Name"
git config user.email "you@example.com"

# Or remove a local override and fall back to global:
git config --unset user.name
git config --unset user.email
```

> **Agent note**: Do not set or override `user.name`/`user.email` yourself. Verify they're correct and flag any mismatch to the user before proceeding.

**Run CSharpier before the first commit.** After all source files are created but before any `git add`, install local tools and format:

```shell
dotnet tool restore          # installs CSharpier from .config/dotnet-tools.json
dotnet csharpier format .    # format all .cs files in place
dotnet format style          # fix naming / using-directive style
dotnet format analyzers      # fix Roslyn analyzer violations
```

Skipping this step is the single most common cause of post-scaffold "fix formatting" commits. CSharpier's Allman brace style and `dotnet format` naming rules will flag every generated file — format once up front and the CI `format` job passes green on the very first push.

The first commit should happen **after Phase 1 is complete** (all root files created) **and after formatting**:

```shell
git add -A
git commit -m "Initial scaffold"
```

**Why**: MinVer computes versions from git tag distance — without a repo and at least one commit, `dotnet pack` fails. `.gitattributes` line-ending rules only apply to tracked files. The `Build.cs rename` validation step in Phase 13 also requires a working tree.

### Recommended commit strategy

Rather than a single monolithic "Initial scaffold" commit, use three logical commits to keep the history readable:

```shell
# Commit 1: Root config files (Phase 1)
git add .editorconfig .csharpierrc.json .gitattributes .gitignore .jscpd.json .markdownlint.json \
    .pre-commit-config.yaml .config/ codecov.yml global.json nuget.config \
    LICENSE SECURITY.md CONTRIBUTING.md CHANGELOG.md \
    .github/FUNDING.yml .github/ISSUE_TEMPLATE/ .github/PULL_REQUEST_TEMPLATE.md \
    Icon.png README.md <LIBNAME>.slnx
git commit -m "chore: root config and repository health files"

# Commit 2: Source projects and task runner (Phases 2–8, 10–11)
git add src/ Build.cs benchmarks/
git commit -m "chore: initial project structure and Build.cs task runner"

# Commit 3: CI pipeline (Phase 9)
git add .github/workflows/ .github/dependabot.yml
git commit -m "ci: GitHub Actions workflow and Dependabot"
```

> **Note**: Phase 11b (seeding benchmark results) adds a fourth commit after benchmarks have been run locally.

---

## Phase 1: Repository Root Files

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

**Why**: Pins the exact SDK so every developer, every CI runner, and every contributor uses identical tooling. `latestPatch` allows security patches without re-pinning while preventing accidental major SDK upgrades.

---

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

**Why**: Explicit NuGet source with `<clear />` prevents machine-level and user-level NuGet config sources from being inherited — ensuring no accidental corporate feed fallback in open source CI.

---

### 1.3 `.gitignore`

Generate the base file then append framework-specific entries:

```shell
dotnet new gitignore
```

Then **append** these lines at the end (they are not in the default template):

```gitignore
# Centralized build output (UseArtifactsOutput)
artifacts/

# BenchmarkDotNet local output
BenchmarkDotNet.Artifacts/

# Benchmark results (committed selectively by the author, not auto-generated)
# benchmarks/ is NOT ignored — results are committed intentionally
```

**Why**: `artifacts/` is the designated build output dir (via `UseArtifactsOutput`); must never be committed. The `dotnet new gitignore` template already covers `*.user`, `.vs/`, `bin/`, `obj/`, etc.

---

### 1.4 `.gitattributes`

```
* text=auto eol=lf
*.cs text eol=lf
*.csproj text eol=lf
*.props text eol=lf
*.targets text eol=lf
*.md text eol=lf
*.yml text eol=lf
*.json text eol=lf
```

**Why**: LF everywhere. `Build.cs` (repo task runner) is a regular `.cs` file already covered by the `*.cs` rule — its shebang requires LF, which is the default. Prevents `git diff` noise across OS contributors.

---

### 1.5 `.editorconfig`

Create at root. Key settings — do not skip any of these sections:

```editorconfig
root = true

# All files
[*]
indent_style = space

# XML project files
[*.{csproj,props,targets,config,nuspec}]
indent_size = 2

# C# files
[*.cs]
indent_size = 4
insert_final_newline = true
end_of_line = lf
charset = utf-8-bom

# Build.cs task runner — BOM would break the Unix shebang (#!/usr/bin/env dotnet)
[Build.cs]
charset = utf-8

# YAML files
[*.{yml,yaml}]
charset = utf-8
insert_final_newline = true
indent_size = 2
end_of_line = lf

# JSON files
[*.json]
indent_size = 2
insert_final_newline = true

# Sorting
dotnet_sort_system_directives_first = true

# Accessibility modifiers: omit private (cleaner code)
dotnet_style_require_accessibility_modifiers = omit_if_default

# File-scoped namespaces (required, not optional)
csharp_style_namespace_declarations = file_scoped:warning

# Throw expressions required
csharp_style_throw_expression = true:error

# Formatting: IDE0055 disabled — CSharpier is the authoritative formatter.
# dotnet format runs style + analyzers only; the whitespace sub-check is skipped via
# split CI steps to avoid conflict with CSharpier's Allman-style object-initializer
# and switch-expression brace formatting.
dotnet_diagnostic.IDE0055.severity = none

# Remove unused usings
dotnet_diagnostic.IDE0005.severity = error

# Allman braces for types, methods, properties, control blocks
csharp_new_line_before_open_brace = accessors, lambdas, types, methods, properties, control_blocks

# Naming: private instance fields → _camelCase
dotnet_naming_rule.private_fields.symbols = private_fields
dotnet_naming_rule.private_fields.style = _camelcase
dotnet_naming_rule.private_fields.severity = suggestion
dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private, protected
dotnet_naming_style._camelcase.required_prefix = _
dotnet_naming_style._camelcase.capitalization = camel_case

# Naming: private static readonly fields → s_camelCase
dotnet_naming_rule.private_static_readonly.symbols = private_static_readonly
dotnet_naming_rule.private_static_readonly.style = s_camelcase
dotnet_naming_rule.private_static_readonly.severity = suggestion
dotnet_naming_symbols.private_static_readonly.applicable_kinds = field
dotnet_naming_symbols.private_static_readonly.applicable_accessibilities = private, protected
dotnet_naming_symbols.private_static_readonly.required_modifiers = static, readonly
dotnet_naming_style.s_camelcase.required_prefix = s_
dotnet_naming_style.s_camelcase.capitalization = camel_case

# Selective diagnostic suppressions — add as needed
# CA1707 underscores in identifiers (test methods, benchmarks use them)
dotnet_diagnostic.CA1707.severity = none
# CS8981 lowercase type names (if you build a class intentionally named lowercase)
dotnet_diagnostic.CS8981.severity = none
# CA1805 do not initialize unnecessarily (prefer being explicit)
dotnet_diagnostic.CA1805.severity = none
# MA0008 (Meziantou) StructLayout missing on unmanaged structs — fires on P/Invoke structs by design
dotnet_diagnostic.MA0008.severity = none
# MA0051 (Meziantou) Method too long — library code with large switch/table methods is intentionally long
dotnet_diagnostic.MA0051.severity = none
```

**Why**: Formatting-as-error (`IDE0055`) means CI fails on style drift, not just function. The naming rules catch the common confusion between `_camelCase`, `s_camelCase`, and raw `camelCase` on first PR.

---

### 1.6 `.jscpd.json`

```json
{
  "threshold": 5,
  "reporters": ["console"],
  "ignore": ["**/__mocks__/**", "**/node_modules/**", "**/artifacts/**"],
  "absolute": true
}
```

**Why**: Copy-paste detection enforces DRY discipline from day one. `threshold: 5` requires at least 5 duplicated tokens before flagging, which avoids noise from short boilerplate patterns (like benchmark `[Params]` attributes) while catching real copy-paste bugs. When you have many structurally similar methods (like 450 P/Invoke bindings), duplication detection distinguishes intentional pattern repetition from accidental copy-paste bugs.

---

### 1.7 `.markdownlint.json`

```json
{
  "default": true,
  "MD013": false,
  "MD033": false,
  "MD041": false
}
```

**Why**: Lint Markdown for broken lists, heading hierarchy, etc. Disable `MD013` (line length) and `MD033` (inline HTML) which are too strict for technical docs.

---

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

**Why**: `gitleaks` prevents API keys, tokens, and connection strings from ever entering git history. End-of-file and trailing whitespace hooks enforce clean diffs. The **CSharpier hook** catches formatting violations before they reach CI — a local `system` hook that requires `dotnet tool restore` to have been run (see Phase 1.16). Run `pre-commit install` after cloning to activate.

**Prerequisite**: `pre-commit` is a Python tool. Install via `pip install pre-commit` (or `pipx install pre-commit`). If Python is not available on the system, skip this file — the CI format job and `.editorconfig` still enforce style. Add a note in the README's "Development" section about this optional dependency.

> **Hook versions**: Like package versions, the `rev:` values are placeholder tags (`<LATEST_GITLEAKS_TAG>`, `<LATEST_PRECOMMIT_HOOKS_TAG>`). Resolve them via `git ls-remote --tags https://github.com/gitleaks/gitleaks | tail -1` or visit the respective GitHub release pages.

---

### 1.9 `codecov.yml`

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

**Why**: Codecov integration gives per-PR coverage delta visibility. `target: auto` means "don't regress from current". `threshold: 0%` enforces zero regression — any coverage drop fails the check. Change to `1%` during early development if you expect high churn before stabilizing.

---

### 1.10 `<LIBNAME>.slnx`

```xml
<Solution>
  <Project Path="src/<LIBNAME>/<LIBNAME>.csproj" />
  <Project Path="src/<LIBNAME>.Test/<LIBNAME>.Test.csproj" />
  <Project Path="src/<LIBNAME>.XyzTest/<LIBNAME>.XyzTest.csproj" />
  <Project Path="src/<LIBNAME>.Benchmarks/<LIBNAME>.Benchmarks.csproj" />
  <Project Path="src/<LIBNAME>.ComparisonBenchmarks/<LIBNAME>.ComparisonBenchmarks.csproj" />
  <Project Path="src/<LIBNAME>.Tester/<LIBNAME>.Tester.csproj" />
</Solution>
```

**Why**: `.slnx` is the new SDK-style solution format (VS 2022 17.9+). No GUIDs, no XML cruft.

> **IDE compatibility note**: JetBrains Rider's `.slnx` support is partial (as of 2025.1). If your team uses Rider, generate a traditional `.sln` instead: `dotnet new sln` then `dotnet sln add src/**/*.csproj`. Add this to the Decision Table if needed.

---

### 1.11 `LICENSE`

MIT license body with `Copyright (c) <YEAR> <COMPANY>`.

---

### 1.12 `SECURITY.md`

GitHub displays a prominent "Security policy: Not enabled" badge on repos without this file. Even a minimal policy establishes a vulnerability disclosure channel:

```markdown
# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability, please report it via
[GitHub's private vulnerability reporting](https://github.com/<AUTHOR>/<LIBNAME>/security/advisories/new).

Do **not** open a public issue for security vulnerabilities.
```

**Why**: Required for any public repository aspiring to professional-grade trust signals. GitHub Security Advisories are built-in and free.

---

### 1.13 `CONTRIBUTING.md`

````markdown
# Contributing to <LIBNAME>

Thank you for considering contributing!

## How to Contribute

1. Fork the repository and create a feature branch from `main`.
2. Run `dotnet tool restore` to install local tools (CSharpier).
3. Ensure your code passes all checks:
   ```shell
   dotnet build -c Release
   dotnet test
   dotnet csharpier check .
   dotnet format style --verify-no-changes
   dotnet format analyzers --verify-no-changes
   ```
````

4. Open a Pull Request against `main` with a clear description of the change.

## Code Style

This project enforces formatting via **CSharpier** (opinionated C# formatter) and **`dotnet format`** (code style analyzers). CI will reject PRs with formatting violations.

- **Auto-format before committing**: `dotnet csharpier format .` first (fixes whitespace/brace style), then `dotnet format style && dotnet format analyzers` (fixes naming/usings). **Order matters**: always run CSharpier first. Never run bare `dotnet format` — the `whitespace` sub-check overwrites CSharpier's Allman-style brace formatting.
- IDE integration: Install the CSharpier extension for [VS Code](https://marketplace.visualstudio.com/items?itemName=csharpier.csharpier-vscode), [Visual Studio](https://marketplace.visualstudio.com/items?itemName=csharpier.CSharpier), or [Rider](https://plugins.jetbrains.com/plugin/18243-csharpier) for format-on-save.

## Reporting Issues

Use [GitHub Issues](https://github.com/<AUTHOR>/<LIBNAME>/issues) for bugs and feature requests.

````

**Why**: Establishes the contribution workflow upfront. GitHub shows a prominent "Contributing" link in the repository sidebar when this file exists.

---

### 1.14 `CHANGELOG.md`

```markdown
# Changelog

All notable changes to this project are documented in [GitHub Releases](https://github.com/<AUTHOR>/<LIBNAME>/releases).

This project uses [MinVer](https://github.com/adamralph/minver) for semantic versioning based on git tags.
````

**Why**: While MinVer + GitHub Releases handle versioning, some package consumers and tooling (e.g., `keepachangelog` linters) expect a `CHANGELOG.md`. This redirect establishes the pattern without duplicating release notes.

---

### 1.15 `.github/FUNDING.yml` _(Optional)_

```yaml
github: [<AUTHOR>]
```

**Why**: Enables the "Sponsor" button on the GitHub repository page. Trivial file, significant professional signal. Skip if sponsorship is not desired.

---

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

**Why**: Establishes the local tool manifest from day one with **CSharpier** pre-installed. `dotnet tool restore` after clone installs CSharpier immediately — no manual setup required. CSharpier is an opinionated C# formatter (like Prettier for C#) that enforces consistent formatting across the entire codebase without bikeshedding over brace placement or line wrapping.

> **Resolving `<LATEST_CSHARPIER>`**: Run `dotnet package search csharpier --take 1` or visit [nuget.org/packages/CSharpier](https://www.nuget.org/packages/CSharpier/).

---

### 1.16a `.csharpierrc.json`

```json
{
  "printWidth": 120,
  "useTabs": false,
  "tabWidth": 4,
  "endOfLine": "lf"
}
```

**Why**: CSharpier uses opinionated defaults (100 char width). A `printWidth` of 120 aligns with modern widescreen development. `endOfLine: lf` matches the `.gitattributes` and `.editorconfig` LF-everywhere policy. This file is intentionally minimal — CSharpier's opinionated nature means fewer config knobs than a traditional formatter.

**CSharpier vs `dotnet format` — split responsibilities, split CI steps**:

- **CSharpier** owns **whitespace formatting**: indentation, line wrapping, brace placement, spacing. It formats object initializers and switch expressions in Allman style (opening brace on its own line).
- **`dotnet format`** owns **code style analyzers**: naming conventions (`_camelCase`), using directive ordering, expression-bodied members, `var` usage, unused code removal (IDE0005).

**The conflict**: `dotnet format --verify-no-changes` (bare, no subcommand) runs three sub-checks: `whitespace`, `style`, and `analyzers`. The `whitespace` sub-check rewrites CSharpier's Allman-style object-initializer and switch-expression braces to K&R, producing `WHITESPACE` errors in CI on correctly CSharpier-formatted code.

**The fix**: always run `dotnet format` as two explicit sub-checks and skip `whitespace` entirely:

```shell
dotnet format style --verify-no-changes
dotnet format analyzers --verify-no-changes
```

Set `IDE0055.severity = none` in `.editorconfig`. CSharpier is authoritative for whitespace; the `whitespace` sub-check of `dotnet format` must never be run alongside it.

---

### 1.17 `.github/ISSUE_TEMPLATE/bug_report.md`, `feature_request.md`, and `PULL_REQUEST_TEMPLATE.md`

Create three community health files that GitHub surfaces automatically to contributors.

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
- <LIBNAME> version: [e.g. 1.2.3]

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

Brief description of the change and why it was made.

## Checklist

- [ ] `dotnet build -c Release` passes with zero warnings
- [ ] `dotnet test` passes
- [ ] `dotnet csharpier check .` passes (formatting)
- [ ] `dotnet format style --verify-no-changes` passes (naming/style)
- [ ] `dotnet format analyzers --verify-no-changes` passes (Roslyn analyzers)
- [ ] Public API changes are reflected in README (run `dotnet test` to auto-update)
- [ ] Breaking changes are noted in the PR description
```

**Why**: GitHub displays "This project doesn't have issue templates" in the new-issue dialog without these files. They lower the barrier for quality bug reports, signal an actively maintained repository, and ensure contributors run the required pre-PR checks.

---

### 1.18 `Icon.png`

Place a **128×128 PNG** at the repo root named `Icon.png`. This file is **required** — the library `.csproj` packs it into the NuGet package (`<None Include="../../Icon.png" Pack="true" PackagePath="\" />`). The CI build will fail with a missing-file error if it is absent.

- For initial scaffold: commit a placeholder 128×128 PNG (even a solid color) and replace it before the first real NuGet publish.
- Design guidance: square, looks good at 32×32 (used on nuget.org listings).

**Agent note — generating a placeholder PNG**: AI agents cannot produce binary files directly. Use one of these approaches:

1. **PowerShell + .NET** (no extra tools):
   ```powershell
   # Generate a minimal 128x128 solid-color PNG at repo root
   Add-Type -AssemblyName System.Drawing
   $bmp = [System.Drawing.Bitmap]::new(128, 128)
   $g = [System.Drawing.Graphics]::FromImage($bmp)
   $g.Clear([System.Drawing.Color]::FromArgb(92, 45, 145))  # .NET purple
   $g.Dispose()
   $bmp.Save("Icon.png", [System.Drawing.Imaging.ImageFormat]::Png)
   $bmp.Dispose()
   ```
2. **ImageMagick** (if available): `magick -size 128x128 xc:#5C2D91 Icon.png`
3. **Manual step**: Mark as `# TODO: Add Icon.png (128×128 placeholder)` and instruct the user to provide one.

---

### 1.19 `README.md` — Initial Template

````markdown
# <LIBNAME>

![.NET](https://img.shields.io/badge/<DOTNET_VERSION>-5C2D91?logo=.NET&labelColor=gray)
![C#](https://img.shields.io/badge/C%23-<CSHARP_VERSION>-239120?labelColor=gray)
[![Build Status](https://github.com/<AUTHOR>/<LIBNAME>/actions/workflows/dotnet.yml/badge.svg?branch=main)](<GITHUB_URL>/actions/workflows/dotnet.yml)
[![codecov](https://codecov.io/gh/<AUTHOR>/<LIBNAME>/branch/main/graph/badge.svg)](https://codecov.io/gh/<AUTHOR>/<LIBNAME>)
[![Nuget](https://img.shields.io/nuget/v/<LIBNAME>?color=purple)](https://www.nuget.org/packages/<LIBNAME>/)
[![License](https://img.shields.io/github/license/<AUTHOR>/<LIBNAME>)](<GITHUB_URL>/blob/main/LICENSE)

<DESCRIPTION>. Cross-platform, trimmable and AOT/NativeAOT compatible.

⭐ Please star this project if you like it. ⭐

[Example](#example) | [Example Catalogue](#example-catalogue) | [Public API Reference](#public-api-reference)

## Example

```csharp
// Example code will be auto-populated from ReadMeTest.cs by tests
```
````

For more examples see [Example Catalogue](#example-catalogue).

## Benchmarks

Benchmarks.

### Detailed Benchmarks

#### Comparison Benchmarks

##### TestBench Benchmark Results

###### Results will be populated here after running `dotnet Build.cs comparison-bench` then `dotnet test`

## Example Catalogue

The following examples are available in [ReadMeTest.cs](src/<LIBNAME>.XyzTest/ReadMeTest.cs).

### Example - Empty

```csharp
// Example code will be auto-populated from ReadMeTest.cs by tests
```

## Public API Reference

```csharp
// Public API will be auto-populated by ReadMeTest_PublicApi test
```

```

**Critical**: The README section headers must be **exactly** the strings referenced in `ReadMeTest.cs`. They form a contract between the test code and the document. Never rename them without updating both files simultaneously.

---

## Phase 2: Source Directory Structure

```

src/
Directory.Build.props ← shared build config for all projects
Directory.Build.targets ← shared build targets (usually minimal)
<LIBNAME>/ ← THE LIBRARY
<LIBNAME>.Test/ ← unit tests
<LIBNAME>.XyzTest/ ← integration + README sync tests
<LIBNAME>.Benchmarks/ ← BDN perf benchmarks
<LIBNAME>.ComparisonBenchmarks/ ← BDN comparison vs. alternatives
<LIBNAME>.Tester/ ← console app for manual testing

````

---

## Phase 3: `src/Directory.Build.props`

This is the **master build configuration**. All projects inherit from it.

```xml
<Project>

  <PropertyGroup>
    <!-- MinVer: derives version from git tags -->
    <MinVerDefaultPreReleaseIdentifiers>preview</MinVerDefaultPreReleaseIdentifiers>
    <MinVerTagPrefix>v</MinVerTagPrefix>
    <MinVerVerbosity>minimal</MinVerVerbosity>

    <Company><AUTHOR></Company>
    <Authors><AUTHOR></Authors>
    <Copyright>Copyright © <COMPANY> <YEAR></Copyright>
    <NeutralLanguage>en</NeutralLanguage>

    <!-- Target framework: ONE value here, no multi-targeting unless explicitly needed -->
    <TargetFramework><DOTNET_VERSION></TargetFramework>
    <!-- Defensive: suppresses build warning if the target framework becomes EOL in future.
         Not needed on a fresh scaffold, but prevents CI breakage when an LTS version sunsets. -->
    <CheckEolTargetFramework>false</CheckEolTargetFramework>

    <LangVersion><CSHARP_VERSION></LangVersion>
    <ImplicitUsings>enable</ImplicitUsings>
    <Deterministic>true</Deterministic>
    <DebugType>portable</DebugType>
    <Nullable>enable</Nullable>

    <!-- Artifact output layout: all bins go to artifacts/ at repo root -->
    <UseArtifactsOutput>true</UseArtifactsOutput>
    <ArtifactsPath>$(MSBuildThisFileDirectory)../artifacts</ArtifactsPath>
    <VSTestResultsDirectory>$(ArtifactsPath)/TestResults</VSTestResultsDirectory>

    <PublishRelease>true</PublishRelease>
    <PackRelease>true</PackRelease>

    <!-- Generate docs XML; suppress missing-doc warnings for tests -->
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <NoWarn>$(NoWarn);CS1591</NoWarn>

    <!-- Analyzers: warnings are errors -->
    <AnalysisLevel>latest</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <RunAnalyzersDuringBuild>true</RunAnalyzersDuringBuild>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <CodeAnalysisTreatWarningsAsErrors>true</CodeAnalysisTreatWarningsAsErrors>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <SuppressNETCoreSdkPreviewMessage>true</SuppressNETCoreSdkPreviewMessage>
  </PropertyGroup>

  <!-- Non-CLS-compliant: required for P/Invoke and low-level types.
       REMOVE this entire ItemGroup for pure managed libraries that expose no non-CLS types (WRAPS_NATIVE=false, NEEDS_UNSAFE=false). -->
  <ItemGroup>
    <AssemblyAttribute Include="System.CLSCompliantAttribute">
      <_Parameter1>false</_Parameter1>
    </AssemblyAttribute>
  </ItemGroup>

  <!-- MinVer package: private, not shipped to consumers -->
  <!-- Verify current stable version at: https://www.nuget.org/packages/MinVer/ -->
  <ItemGroup>
    <PackageReference Include="MinVer" Version="<LATEST_MINVER>" PrivateAssets="All" />
  </ItemGroup>

  <!-- CSharpier formatting enforcement: checks formatting on every build.
       Violations appear as build errors when TreatWarningsAsErrors is enabled.
       Install the local tool first: dotnet tool restore -->
  <ItemGroup>
    <PackageReference Include="CSharpier.MsBuild" Version="<LATEST_CSHARPIER>" PrivateAssets="All" />
  </ItemGroup>

</Project>
````

**Why each decision matters:**

- `<Deterministic>true</Deterministic>` — identical inputs produce byte-identical outputs; required for reproducible builds and binary signing
- `<DebugType>portable</DebugType>` — portable PDB format works on all OSes
- `<Nullable>enable</Nullable>` — null-safety from day one; retrofitting later is painful
- `<ImplicitUsings>enable</ImplicitUsings>` — SDK default since .NET 6; declared explicitly for clarity since every other property in this file is explicit
- `<CodeAnalysisTreatWarningsAsErrors>true</CodeAnalysisTreatWarningsAsErrors>` — analyzer diagnostics (CA/IDE prefixed) block merges, not just warn
- `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` — ALL compiler warnings (CS-prefixed) are also errors; catches unused variables, unreachable code, etc.
- `<UseArtifactsOutput>true</UseArtifactsOutput>` — centralizes all build output to `artifacts/` at repo root instead of per-project `bin/`/`obj/` folders

---

## Phase 3b: `src/Directory.Build.targets`

Create as a minimal file. For most libraries this file is intentionally empty — only add content if you have shared build targets that don’t belong in `Directory.Build.props`:

```xml
<Project>
  <!-- Add shared MSBuild targets here if needed.
       For most libraries this file stays empty. -->
</Project>
```

---

## Phase 4: The Library Project

### 4.1 `src/<LIBNAME>/<LIBNAME>.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <RootNamespace><ROOTNS></RootNamespace>
    <IsPackable>true</IsPackable>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>   <!-- Remove if NEEDS_UNSAFE=false and no unsafe code is anticipated -->

    <!-- AOT/Trimming: ENFORCE from day one.
         If NEEDS_AOT=false, remove these four lines and the description suffix below. -->
    <IsTrimmable>true</IsTrimmable>
    <IsAotCompatible>true</IsAotCompatible>
    <EnableTrimAnalyzer>true</EnableTrimAnalyzer>
    <EnableAOTAnalyzer>true</EnableAOTAnalyzer>

    <Description><DESCRIPTION>. Cross-platform, trimmable and AOT/NativeAOT compatible.</Description>

    <!-- NuGet metadata -->
    <PackageId><LIBNAME></PackageId>
    <PackageTags>REPLACE;WITH;RELEVANT;TAGS;performance;dotnet</PackageTags>
    <PackageIcon>Icon.png</PackageIcon>
    <PackageReadmeFile>README.md</PackageReadmeFile>
    <PackageProjectUrl><GITHUB_URL>/</PackageProjectUrl>
    <PackageReleaseNotes><GITHUB_URL>/releases</PackageReleaseNotes>
    <PackageLicenseExpression><LICENSE></PackageLicenseExpression>
    <PackageRequireLicenseAcceptance>false</PackageRequireLicenseAcceptance>

    <!-- Source Link: allows debugger to step into NuGet source.
         SourceLink is built into the .NET 8+ SDK; no explicit
         Microsoft.SourceLink.GitHub PackageReference is needed. -->
    <PublishRepositoryUrl>true</PublishRepositoryUrl>
    <RepositoryUrl><GITHUB_URL>/</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
    <IncludeSymbols>true</IncludeSymbols>
    <SymbolPackageFormat>snupkg</SymbolPackageFormat>
    <EmbedUntrackedSources>true</EmbedUntrackedSources>

    <!-- Deterministic on CI -->
    <ContinuousIntegrationBuild Condition="'$(GITHUB_ACTIONS)' == 'true'">true</ContinuousIntegrationBuild>
    <ContinuousIntegrationBuild Condition="'$(TF_BUILD)' == 'true'">true</ContinuousIntegrationBuild>
  </PropertyGroup>

  <!-- Pack icon and README into NuGet package -->
  <ItemGroup>
    <None Include="../../Icon.png" Pack="true" PackagePath="\" />
    <None Include="../../README.md" Pack="true" PackagePath="\" />
  </ItemGroup>

  <!-- InternalsVisibleTo: allow test/benchmark projects to access internal APIs -->
  <ItemGroup>
    <AssemblyAttribute Include="System.Runtime.CompilerServices.InternalsVisibleTo">
      <_Parameter1>$(MSBuildProjectName).Test</_Parameter1>
    </AssemblyAttribute>
    <AssemblyAttribute Include="System.Runtime.CompilerServices.InternalsVisibleTo">
      <_Parameter1>$(MSBuildProjectName).XyzTest</_Parameter1>
    </AssemblyAttribute>
    <AssemblyAttribute Include="System.Runtime.CompilerServices.InternalsVisibleTo">
      <_Parameter1>$(MSBuildProjectName).Benchmarks</_Parameter1>
    </AssemblyAttribute>
    <AssemblyAttribute Include="System.Runtime.CompilerServices.InternalsVisibleTo">
      <_Parameter1>$(MSBuildProjectName).ComparisonBenchmarks</_Parameter1>
    </AssemblyAttribute>
  </ItemGroup>

</Project>
```

### 4.2 `src/<LIBNAME>/<RootClass>.cs`

Replace `<RootClass>` with the primary entry-point class name. It must be:

- `partial` (to be split across multiple files as the API grows)
- `static` (for utility/interop libraries) or a normal class (for object-model libraries)
- Named to match the native library or domain, not necessarily PascalCase if domain convention differs

```csharp
namespace <ROOTNS>;

public static partial class <RootClass>
{
    static <RootClass>()
    {
        // Reserved for initialization: NativeLibrary.Load, context setup, etc.
    }

    public static void Empty() { }
}
```

**The `Empty()` method** is not a joke. It serves as:

1. The initial baseline benchmark (0.0004 ns = JIT overhead floor)
2. The first test target (proves the test infrastructure works)
3. A compilation check that the project builds at all

Do not delete it until real API is added and the tests/benchmarks are updated.

---

## Phase 5: Test Projects

### 5.1 `src/<LIBNAME>.Test/<LIBNAME>.Test.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <RootNamespace><ROOTNS>.Test</RootNamespace>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    <IsPackable>false</IsPackable>
    <NoWarn>$(NoWarn);CA2007</NoWarn>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\<LIBNAME>\<LIBNAME>.csproj" />
  </ItemGroup>

  <!-- Exclude entire test assembly from code coverage metrics -->
  <ItemGroup>
    <AssemblyAttribute Include="System.Diagnostics.CodeAnalysis.ExcludeFromCodeCoverageAttribute" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="TUnit" Version="<LATEST_TUNIT>" />
  </ItemGroup>
</Project>
```

> ⚠️ **Code coverage with TUnit**: The `TUnit` meta package **already includes `Microsoft.Testing.Extensions.CodeCoverage` transitively** — no additional coverage package is needed. Do NOT add `coverlet.collector` or `coverlet.msbuild` — these are VSTest data collectors and are [not compatible with TUnit's MTP runner](https://tunit.dev/docs/extending/code-coverage/). Coverage is collected via the `--coverage` flag:
>
> ```shell
> # Collect coverage (cobertura XML for CI tools like Codecov):
> dotnet test --coverage --coverage-output-format cobertura
>
> # Multiple formats in one run:
> dotnet test --coverage --coverage-output-format cobertura --coverage-output-format xml
> ```
>
> **Viewing coverage results**:
>
> - **Visual Studio** — Built-in coverage viewer (Enterprise/Community 2026+)
> - **VS Code** — Install the [Coverage Gutters](https://marketplace.visualstudio.com/items?itemName=ryanluker.vscode-coverage-gutters) extension, then open the generated `coverage.cobertura.xml` to see inline coverage highlighting
> - **ReportGenerator** — Generate HTML reports: `reportgenerator -reports:coverage.cobertura.xml -targetdir:coveragereport`
> - **CI** — Most CI systems (GitHub Actions, Azure Pipelines) parse Cobertura format natively

### 5.1a `src/<LIBNAME>.Test/<RootClass>Test.cs`

Every test project needs at least one test to prove the infrastructure works:

```csharp
namespace <ROOTNS>.Test;

public class <RootClass>Test
{
    [Test]
    public void Empty_DoesNotThrow()
    {
        <RootClass>.Empty();
    }
}
```

**Why**: Mirrors the `Empty()` philosophy — this test proves the test runner, the project reference, and the library all link and execute. Delete it when real tests are added.

### 5.2 `src/<LIBNAME>.XyzTest/<LIBNAME>.XyzTest.csproj`

> **Naming note**: `XyzTest` is the `nietras` convention for “everything that isn’t a pure unit test” — README sync tests, public API snapshot validation, integration tests. If your team prefers a clearer name (`IntegrationTest`, `E2ETest`), rename it consistently across the `.slnx`, the `InternalsVisibleTo` attributes in the library `.csproj`, and all `ProjectReference` entries.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <RootNamespace><ROOTNS>.XyzTest</RootNamespace>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    <IsPackable>false</IsPackable>
    <NoWarn>$(NoWarn);CA2007</NoWarn>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\<LIBNAME>\<LIBNAME>.csproj" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="TUnit" Version="<LATEST_TUNIT>" />
    <PackageReference Include="PublicApiGenerator" Version="<LATEST_PUBLICAPIGEN>" />
  </ItemGroup>
</Project>
```

**Note**: `XyzTest` is NOT excluded from coverage at the assembly level — individual test classes that do infrastructure work (ReadMeTest) should be decorated with `[ExcludeFromCodeCoverage]` directly.

### 5.3 `src/<LIBNAME>.XyzTest/AssemblyInitializeCultureTest.cs`

```csharp
using System.Globalization;

namespace <ROOTNS>.XyzTest;

public static class AssemblyInitializeCultureTest
{
    [Before(Assembly)]
    public static void SetInvariantCulture()
    {
        CultureInfo.DefaultThreadCurrentCulture = CultureInfo.InvariantCulture;
        CultureInfo.DefaultThreadCurrentUICulture = CultureInfo.InvariantCulture;
    }
}
```

> **Parallelism note**: TUnit runs tests in parallel by default. To prevent README-writing tests from corrupting the file, apply `[NotInParallel]` to the `ReadMeTest` class (see Phase 6). TUnit's parallelism is controlled per-class/method via `[NotInParallel]` rather than at the assembly level.

> ⚠️ **TUnit + .NET 10 SDK compatibility**: TUnit uses `Microsoft.Testing.Platform` (MTP), which conflicts with the legacy VSTest runner that `dotnet test` uses by default on .NET 10 SDK. On .NET 10, `dotnet test` will fail with an `Microsoft.Testing.Platform.MSBuild.targets(263)` error unless you add this to `Directory.Build.props` (the root one, **not** the test `.csproj`):
>
> ```xml
> <TestingPlatformDotnetTestSupport>true</TestingPlatformDotnetTestSupport>
> ```
>
> Alternatively, run tests via `dotnet run --project src/<LIBNAME>.Test/<LIBNAME>.Test.csproj` which bypasses VSTest entirely. Both approaches work; the `Directory.Build.props` property is cleaner for CI and multi-project solutions.

**Why**: Culture-invariant tests prevent locale-dependent failures (decimal separators, date formats).

> **Package versions**: All package versions are represented as `<LATEST_*>` placeholder variables. Like the `<PINNED-SHA>` placeholders in Phase 9, these **must be resolved before scaffolding**. Run the helper command below or visit [nuget.org](https://www.nuget.org/) to get current stable versions:
>
> ```shell
> # Example: get latest stable MinVer version
> dotnet package search MinVer --take 1
> ```
>
> | Placeholder             | Package                                                                  | Verify at                                  |
> | ----------------------- | ------------------------------------------------------------------------ | ------------------------------------------ |
> | `<LATEST_MINVER>`       | [MinVer](https://www.nuget.org/packages/MinVer/)                         | `dotnet package search MinVer`             |
> | `<LATEST_CSHARPIER>`    | [CSharpier](https://www.nuget.org/packages/CSharpier/)                   | `dotnet package search CSharpier`          |
> | `<LATEST_TUNIT>`        | [TUnit](https://www.nuget.org/packages/TUnit/)                           | `dotnet package search TUnit`              |
> | `<LATEST_PUBLICAPIGEN>` | [PublicApiGenerator](https://www.nuget.org/packages/PublicApiGenerator/) | `dotnet package search PublicApiGenerator` |
> | `<LATEST_BDN>`          | [BenchmarkDotNet](https://www.nuget.org/packages/BenchmarkDotNet/)       | `dotnet package search BenchmarkDotNet`    |
>
> If the table is empty at scaffold time, the agent **must not invent version numbers**. Instead, run `dotnet package search <PackageName> --take 1` to resolve each version, or leave the placeholder and add a `<!-- TODO: resolve package version -->` comment.

---

## Phase 6: The ReadMeTest Pattern (Core Innovation)

### 6.1 Philosophy

The `ReadMeTest.cs` pattern enforces three guarantees via CI:

1. **Code examples in README always compile and are correct** — they are literally the test source
2. **Benchmark tables are always current** — they are auto-embedded from BDN output files
3. **Public API in README always matches the compiled assembly** — generated by reflection

### 6.2 `src/<LIBNAME>.XyzTest/ReadMeTest.cs`

````csharp
#pragma warning disable CA2007 // ConfigureAwait
#pragma warning disable CA1822 // Mark as static

using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Linq;
using System.Runtime.CompilerServices;
using PublicApiGenerator;

namespace <ROOTNS>.XyzTest;

[NotInParallel]
[System.Diagnostics.CodeAnalysis.ExcludeFromCodeCoverage]
public partial class ReadMeTest
{
    static readonly string s_testSourceFilePath = SourceFile();
    // Navigate from src/<LIBNAME>.XyzTest/ up to repo root (2 levels: XyzTest → src → root)
    static readonly string s_rootDirectory =
        Path.GetFullPath(Path.Combine(Path.GetDirectoryName(s_testSourceFilePath)!, "..", ".."))
        + Path.DirectorySeparatorChar;
    static readonly string s_readmeFilePath = s_rootDirectory + "README.md";

    // ─────────────────────────────────────────────────────────────
    // SECTION 1: Write your README example code here.
    // These methods are directly snipped into README.md by the update test.
    // ─────────────────────────────────────────────────────────────

    [Test]
    public void ReadMeTest_()
    {
        // REPLACE THIS with actual example calls to your library.
        // This exact method body will appear in the README "## Example" section.
        <RootClass>.Empty();
    }

    // ─────────────────────────────────────────────────────────────
    // SECTION 2: README sync tests — run only on latest TFM
    // ─────────────────────────────────────────────────────────────

#if <DOTNET_CONSTANT>
    [Test]
#endif
    public void ReadMeTest_UpdateExampleCodeInMarkdown()
    {
        var readmeLines = File.ReadAllLines(s_readmeFilePath);
        var testSourceLines = File.ReadAllLines(s_testSourceFilePath);

        var testBlocksToUpdate = new (string StartLineContains, string ReadmeLineBeforeCodeBlock)[]
        {
            // Intentional: same example method body is embedded in both the main
            // Example section and the Example Catalogue entry.
            (nameof(ReadMeTest_) + "()", "## Example"),
            (nameof(ReadMeTest_) + "()", "### Example - Empty"),
        };

        readmeLines = UpdateReadme(testSourceLines, readmeLines, testBlocksToUpdate,
            sourceStartLineOffset: 2, "    }", sourceEndLineOffset: 0,
            sourceWhitespaceToRemove: 8);

        var newReadme = string.Join(Environment.NewLine, readmeLines) + Environment.NewLine;
        File.WriteAllText(s_readmeFilePath, newReadme, System.Text.Encoding.UTF8);
    }

#if <DOTNET_CONSTANT>
    [Test]
#endif
    public void ReadMeTest_UpdateBenchmarksInMarkdown()
    {
        var readmeFilePath = s_readmeFilePath;
        var benchmarkFileNameToConfig = new Dictionary<string, (string Description, string ReadmeBefore, string ReadmeEnd, string SectionPrefix)>()
        {
            { "TestBench.md", ("TestBench Benchmark Results", "##### TestBench Benchmark Results", "## Example Catalogue", "######") },
        };

        var benchmarksDirectory = Path.Combine(s_rootDirectory, "benchmarks");
        if (!Directory.Exists(benchmarksDirectory)) { return; } // not run yet

        var processorDirectories = Directory.EnumerateDirectories(benchmarksDirectory).ToArray();
        var readmeLines = File.ReadAllLines(readmeFilePath);

        foreach (var (fileName, config) in benchmarkFileNameToConfig)
        {
            var all = "";
            foreach (var processorDirectory in processorDirectories)
            {
                var contentsFilePath = Path.Combine(processorDirectory, fileName);
                if (!File.Exists(contentsFilePath)) { continue; }

                var versionsFilePath = Path.Combine(processorDirectory, "Versions.txt");
                var versions = File.ReadAllText(versionsFilePath);
                var contents = File.ReadAllText(contentsFilePath);
                var processor = processorDirectory
                    .Split(Path.DirectorySeparatorChar, Path.AltDirectorySeparatorChar).Last();
                var section = $"{config.SectionPrefix}{processor} - {config.Description} ({versions})";
                var benchmarkTable = contents.Substring(contents.IndexOf('|'));
                all += $"{section}{Environment.NewLine}{Environment.NewLine}{benchmarkTable}{Environment.NewLine}";
            }

            readmeLines = ReplaceReadmeLines(readmeLines, [all],
                config.ReadmeBefore, config.SectionPrefix, 0, config.ReadmeEnd, 0);
        }

        var newReadme = string.Join(Environment.NewLine, readmeLines) + Environment.NewLine;
        File.WriteAllText(s_readmeFilePath, newReadme, System.Text.Encoding.UTF8);
    }

#if <DOTNET_CONSTANT>
    [Test]
#endif
    public void ReadMeTest_PublicApi()
    {
        // REPLACE <RootClass> with your actual root type
        var publicApi = typeof(<RootClass>).Assembly.GeneratePublicApi();
        var readmeLines = File.ReadAllLines(s_readmeFilePath);
        readmeLines = ReplaceReadmeLines(readmeLines, [publicApi],
            "## Public API Reference", "```csharp", 1, "```", 0);
        var newReadme = string.Join(Environment.NewLine, readmeLines) + Environment.NewLine;
        File.WriteAllText(s_readmeFilePath, newReadme, System.Text.Encoding.UTF8);
    }

    // ─────────────────────────────────────────────────────────────
    // INFRASTRUCTURE — do not modify
    // ─────────────────────────────────────────────────────────────

    static string[] UpdateReadme(
        string[] sourceLines, string[] readmeLines,
        (string StartLineContains, string ReadmeLineBefore)[] blocksToUpdate,
        int sourceStartLineOffset, string sourceEndLineStartsWith,
        int sourceEndLineOffset, int sourceWhitespaceToRemove,
        string readmeStartLineStartsWith = "```csharp", int readmeStartLineOffset = 1,
        string readmeEndLineStartsWith = "```", int readmeEndLineOffset = 0)
    {
        foreach (var (startLineContains, readmeLineBeforeBlock) in blocksToUpdate)
        {
            var sourceExampleLines = SnipLines(sourceLines,
                startLineContains, sourceStartLineOffset,
                sourceEndLineStartsWith, sourceEndLineOffset,
                sourceWhitespaceToRemove);
            readmeLines = ReplaceReadmeLines(readmeLines, sourceExampleLines,
                readmeLineBeforeBlock, readmeStartLineStartsWith, readmeStartLineOffset,
                readmeEndLineStartsWith, readmeEndLineOffset);
        }
        return readmeLines;
    }

    static string[] ReplaceReadmeLines(string[] readmeLines, string[] newLines,
        string readmeLineBeforeBlock, string readmeStartLineStartsWith, int readmeStartLineOffset,
        string readmeEndLineStartsWith, int readmeEndLineOffset)
    {
        var beforeIndex = Array.FindIndex(readmeLines,
            l => l.StartsWith(readmeLineBeforeBlock, StringComparison.Ordinal));
        if (beforeIndex < 0) { throw new ArgumentException($"README line '{readmeLineBeforeBlock}' not found."); }

        var replaceStart = Array.FindIndex(readmeLines, beforeIndex,
            l => l.StartsWith(readmeStartLineStartsWith, StringComparison.Ordinal)) + readmeStartLineOffset;
        Debug.Assert(replaceStart >= 0);
        var replaceEnd = Array.FindIndex(readmeLines, replaceStart,
            l => l.StartsWith(readmeEndLineStartsWith, StringComparison.Ordinal)) + readmeEndLineOffset;

        return readmeLines[..replaceStart]
            .AsEnumerable()
            .Concat(newLines)
            .Concat(readmeLines[replaceEnd..])
            .ToArray();
    }

    static string[] SnipLines(string[] sourceLines,
        string startLineContains, int startLineOffset,
        string endLineStartsWith, int endLineOffset, int whitespaceToRemove = 8)
    {
        var start = Array.FindIndex(sourceLines,
            l => l.Contains(startLineContains, StringComparison.Ordinal)) + startLineOffset;
        var end = Array.FindIndex(sourceLines, start,
            l => l.StartsWith(endLineStartsWith, StringComparison.Ordinal)) + endLineOffset;
        return sourceLines[start..end]
            .Select(l => l.Length > whitespaceToRemove ? l.Remove(0, whitespaceToRemove) : l.TrimStart())
            .ToArray();
    }

    static string SourceFile([CallerFilePath] string sourceFilePath = "") => sourceFilePath;
}
````

**Replace `<DOTNET_CONSTANT>` with the correct preprocessor symbol**:

- `net10.0` → `NET10_0`
- `net9.0` → `NET9_0`
- `net8.0` → `NET8_0`

**Why the `#if` guard**: When you multi-target (`net8.0;net10.0`), the README update tests will write to the same file from two test runs running in parallel. Guard to only one TFM writes it.

---

## Phase 7: Benchmarks

### 7.1 `src/<LIBNAME>.Benchmarks/<LIBNAME>.Benchmarks.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <RootNamespace><ROOTNS>.Benchmarks</RootNamespace>
    <OutputType>Exe</OutputType>
    <PlatformTarget>AnyCPU</PlatformTarget>
    <DebugType>pdbonly</DebugType>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\<LIBNAME>\<LIBNAME>.csproj" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="BenchmarkDotNet" Version="<LATEST_BDN>" />
    <!-- Windows-only diagnostics (ETW hardware counter support). Gracefully absent on Linux/macOS. -->
    <PackageReference Include="BenchmarkDotNet.Diagnostics.Windows" Version="<LATEST_BDN>" />
  </ItemGroup>
</Project>
```

### 7.2 `src/<LIBNAME>.Benchmarks/Program.cs`

Entry point for `dotnet Build.cs bench`. Without this file the `<OutputType>Exe</OutputType>` project will not compile.

```csharp
using BenchmarkDotNet.Running;

BenchmarkSwitcher.FromAssembly(typeof(Program).Assembly).Run(args);

// The single-file Program type is auto-generated by top-level statements.
// BenchmarkSwitcher discovers all [Benchmark]-attributed classes in the assembly.
// Invoke: dotnet run -c Release -- --filter "*" --join
```

### 7.2a `src/<LIBNAME>.Benchmarks/<RootClass>Bench.cs`

Starter benchmark class — `BenchmarkSwitcher` needs at least one `[Benchmark]` method in the assembly to function. This mirrors the `Empty()` baseline pattern:

```csharp
using BenchmarkDotNet.Attributes;

namespace <ROOTNS>.Benchmarks;

public class <RootClass>Bench
{
    [Benchmark]
    public void Empty() => <RootClass>.Empty();
}
```

Replace with real benchmarks as the library API develops. The `Empty()` baseline confirms BDN infrastructure works (expect ~0.0004 ns).

### 7.3 `src/<LIBNAME>.ComparisonBenchmarks/<LIBNAME>.ComparisonBenchmarks.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <RootNamespace><ROOTNS>.ComparisonBenchmarks</RootNamespace>
    <OutputType>Exe</OutputType>
    <PlatformTarget>AnyCPU</PlatformTarget>
    <DebugType>pdbonly</DebugType>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\<LIBNAME>\<LIBNAME>.csproj" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="BenchmarkDotNet" Version="<LATEST_BDN>" />
    <!-- Windows-only diagnostics (ETW hardware counter support). Gracefully absent on Linux/macOS. -->
    <PackageReference Include="BenchmarkDotNet.Diagnostics.Windows" Version="<LATEST_BDN>" />
  </ItemGroup>
</Project>
```

### 7.4 `src/<LIBNAME>.ComparisonBenchmarks/TestBench.cs`

**Important**: Do NOT copy `ReaderSpec.cs` from Sep. Design this file around your actual domain:

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Configs;

namespace <ROOTNS>.ComparisonBenchmarks;

[GroupBenchmarksBy(BenchmarkLogicalGroupRule.ByCategory, BenchmarkLogicalGroupRule.ByParams)]
[BenchmarkCategory("0")]
public class TestBench
{
    [Params(25_000)]
    public int Count { get; set; }

    [Benchmark(Baseline = true)]
    public void <LIBNAME>______()
    {
        // Baseline: call the core operation you're benchmarking
        <RootClass>.Empty();
    }

    // Add competing benchmarks here as alternatives are identified:
    // [Benchmark]
    // public void AlternativeLibrary______() { ... }
}
```

**Note**: The `______` trailing underscores are intentional — in the Markdown table they right-pad the method name column, making the table visually cleaner. The trailing underscores fill to a fixed width.

**Note**: The `[Params]` attribute makes `Count` available for parameterized benchmark scenarios. With `TreatWarningsAsErrors` active, every field and property must be used — `[Params]` satisfies this by BDN's runtime injection. Replace or remove `Count` when you define your real benchmark parameters.

### 7.5 `src/<LIBNAME>.ComparisonBenchmarks/Program.cs`

Runs benchmarks and saves results to `benchmarks/<CPU.Name>/` so `ReadMeTest_UpdateBenchmarksInMarkdown` can embed them into the README:

```csharp
using <ROOTNS>.ComparisonBenchmarks;
using System.Runtime.CompilerServices;
using BenchmarkDotNet.Columns;
using BenchmarkDotNet.Configs;
using BenchmarkDotNet.Exporters;
using BenchmarkDotNet.Loggers;
using BenchmarkDotNet.Running;

// Usage (via task runner): dotnet Build.cs comparison-bench
// Direct:                  dotnet run -c Release -- run
if (args is not ["run"])
{
    Console.Error.WriteLine("Usage: dotnet run -c Release -- run");
    return 1;
}

var config = ManualConfig.CreateMinimumViable()
    .WithOptions(ConfigOptions.DisableOptimizationsValidator)
    .AddColumnProvider(DefaultColumnProviders.Instance)
    .AddExporter(MarkdownExporter.GitHub)
    .AddLogger(ConsoleLogger.Default);

var summary = BenchmarkRunner.Run<TestBench>(config);

// Write results to benchmarks/<CPU.Name>/ for README embedding
var cpu = summary.HostEnvironmentInfo.CpuInfo.Value?.ProcessorName ?? "Unknown";
var cpuDirName = string.Concat(cpu.Split(Path.GetInvalidFileNameChars()));
var outputDir = Path.Combine(RepoRoot(), "benchmarks", cpuDirName);
Directory.CreateDirectory(outputDir);

// Versions.txt — expand with competing package versions when you add alternatives
File.WriteAllText(Path.Combine(outputDir, "Versions.txt"), $".NET {Environment.Version}");

// Copy the GitHub-flavored Markdown report from BDN output → TestBench.md
var mdSource = Directory
    .GetFiles(summary.ResultsDirectoryPath, "*TestBench-report-github.md")
    .FirstOrDefault();
if (mdSource is not null)
    File.Copy(mdSource, Path.Combine(outputDir, "TestBench.md"), overwrite: true);
else
    Console.Error.WriteLine("WARNING: No benchmark Markdown found in " + summary.ResultsDirectoryPath);

return 0;

// [CallerFilePath] resolves to the absolute path of this file at compile time.
// From src/<LIBNAME>.ComparisonBenchmarks/Program.cs → up 2 dirs → repo root.
static string RepoRoot([CallerFilePath] string path = "") =>
    Path.GetFullPath(Path.Combine(Path.GetDirectoryName(path)!, "..", ".."));
```

**Agent notes**:

- `summary.ResultsDirectoryPath` points to BDN’s artifact output (`BenchmarkDotNet.Artifacts/results/` relative to the benchmark project directory). The glob `*TestBench-report-github.md` matches BDN’s auto-generated filename. If your benchmark class is renamed from `TestBench`, update the glob.
- Expand `Versions.txt` content with all competing package versions once you add competitors, so the README benchmark section shows meaningful context.
- The `RepoRoot()` traversal is exactly 2 levels up from `src/<LIBNAME>.ComparisonBenchmarks/`. Adjust if you restructure the project tree.

> ⚠️ **BenchmarkDotNet `.Value` API removal**: In BDN 0.14+, `summary.HostEnvironmentInfo.CpuInfo.Value` was removed. The `Lazy<CpuInfo>` property became a plain `CpuInfo?`. If the above code fails to compile with `’Lazy<CpuInfo>’ does not contain a definition for ‘Value’`, change line to:
>
> ```csharp
> var cpu = summary.HostEnvironmentInfo.CpuInfo?.ProcessorName ?? "Unknown";
> ```

---

## Phase 8: Tester Console App

### 8.1 `src/<LIBNAME>.Tester/<LIBNAME>.Tester.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <RootNamespace><ROOTNS>.Tester</RootNamespace>
    <OutputType>Exe</OutputType>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\<LIBNAME>\<LIBNAME>.csproj" />
  </ItemGroup>
</Project>
```

### 8.2 `src/<LIBNAME>.Tester/Program.cs`

```csharp
using <ROOTNS>;

Console.WriteLine($"<LIBNAME> version: {typeof(<RootClass>).Assembly.GetName().Version}");
<RootClass>.Empty();
Console.WriteLine("OK");
```

---

## Phase 9: CI/CD Pipeline

### 9.1 `.github/workflows/dotnet.yml`

**Critical decisions**:

- Pin ALL action references to **full SHA-256 commit hashes**, not floating tags. Get the hash by:
  1. Visiting the action's GitHub page
  2. Looking at the "latest release" commit
  3. Using the full 40-character SHA

- Include `step-security/harden-runner` with `egress-policy: audit` on every job

Full template:

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
    inputs:
      version:
        description: "Release version to tag and create"
        required: false

env:
  DOTNET_SKIP_FIRST_TIME_EXPERIENCE: 1
  DOTNET_NOLOGO: true
  NuGetDirectory: ${{ github.workspace }}/nuget

jobs:
  build-and-test:
    strategy:
      matrix:
        os:
          [
            ubuntu-latest,
            windows-latest,
            macos-latest,
            windows-11-arm,
            ubuntu-24.04-arm,
          ]
        configuration: [Debug, Release]

    runs-on: ${{ matrix.os }}

    steps:
      - name: Harden Runner
        uses: step-security/harden-runner@<PINNED-SHA> # vX.Y.Z
        with:
          egress-policy: audit

      - uses: actions/checkout@<PINNED-SHA> # vX.Y.Z

      - name: Setup .NET (global.json)
        uses: actions/setup-dotnet@<PINNED-SHA> # vX.Y.Z
        with:
          global-json-file: global.json

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build -c ${{ matrix.configuration }} --no-restore

      - name: Test
        run: dotnet test -c ${{ matrix.configuration }} --no-build --verbosity normal --coverage --coverage-output-format cobertura

      - name: Upload coverage to Codecov
        if: matrix.configuration == 'Release' # Upload only once per OS to avoid duplicate reports
        uses: codecov/codecov-action@<PINNED-SHA> # vX.Y.Z
        with:
          flags: ${{ matrix.os }}
          token: ${{ secrets.CODECOV_TOKEN }}

  format:
    runs-on: ubuntu-latest
    steps:
      - name: Harden Runner
        uses: step-security/harden-runner@<PINNED-SHA> # vX.Y.Z
        with:
          egress-policy: audit
      - uses: actions/checkout@<PINNED-SHA>
      - name: Setup .NET
        uses: actions/setup-dotnet@<PINNED-SHA>
        with:
          global-json-file: global.json
      - name: Restore tools
        run: dotnet tool restore
      - name: CSharpier verify no changes
        run: dotnet csharpier check .
      - name: Format style verify no changes
        run: dotnet format style --verify-no-changes
      - name: Format analyzers verify no changes
        run: dotnet format analyzers --verify-no-changes

  pack:
    runs-on: windows-latest
    steps:
      - name: Harden Runner
        uses: step-security/harden-runner@<PINNED-SHA>
        with:
          egress-policy: audit
      - uses: actions/checkout@<PINNED-SHA>
        with:
          fetch-depth: 0 # full history for MinVer
      - name: Create tag (to set version)
        if: ${{ github.event.inputs.version != '' && github.actor == '<AUTHOR>' }}
        run: git tag v${{ github.event.inputs.version }}
      - name: Setup .NET
        uses: actions/setup-dotnet@<PINNED-SHA>
        with:
          global-json-file: global.json
      - name: Pack NuGet package
        run: dotnet pack -c Release --output ${{ env.NuGetDirectory }}
      - uses: actions/upload-artifact@<PINNED-SHA>
        with:
          name: nuget
          if-no-files-found: error
          retention-days: 7
          path: ${{ env.NuGetDirectory }}/*nupkg

  create-release-push:
    needs: [build-and-test, format, pack]
    runs-on: windows-latest
    permissions:
      contents: write
      id-token: write # for OIDC
    if: ${{ github.event.inputs.version != '' && github.actor == '<AUTHOR>' }}

    steps:
      - name: Harden Runner
        uses: step-security/harden-runner@<PINNED-SHA>
        with:
          egress-policy: audit
      - uses: actions/checkout@<PINNED-SHA>
      - name: Download nuget packages
        uses: actions/download-artifact@<PINNED-SHA>
        with:
          name: nuget
          path: ${{ env.NuGetDirectory }}
      - name: Setup .NET
        uses: actions/setup-dotnet@<PINNED-SHA>
        with:
          global-json-file: global.json
      - name: Push packages
        shell: bash
        run: |
          dotnet nuget push "${{ env.NuGetDirectory }}"/*.nupkg --api-key ${{ secrets.NUGET_API_KEY }} --source https://api.nuget.org/v3/index.json --skip-duplicate
          dotnet nuget push "${{ env.NuGetDirectory }}"/*.snupkg --api-key ${{ secrets.NUGET_API_KEY }} --source https://api.nuget.org/v3/index.json --skip-duplicate
```

> **Why glob patterns instead of explicit filenames**: Using `*.nupkg` / `*.snupkg` avoids hardcoding `<LIBNAME>.${{ github.event.inputs.version }}` in the push command. On `windows-latest` runners, glob expansion in YAML `run:` steps is PowerShell by default — PowerShell does **not** expand `*.nupkg` globs the same way bash does. Always add `shell: bash` to steps that use globs on Windows runners; without it the push command passes a literal `*.nupkg` string to the NuGet CLI and fails with "file not found".

> **Why direct `NUGET_API_KEY` instead of OIDC `NuGet/login`**: The OIDC-based `NuGet/login` action adds complexity (a separate step, OIDC token exchange, an additional action to SHA-pin) and is unnecessary for straightforward pushes. A repository secret named `NUGET_API_KEY` is simpler, more widely documented, and behaves identically across GitHub-hosted and self-hosted runners.

**How to get pinned SHAs**: Go to each action's GitHub release page and copy the commit SHA for the version you want. Pin to an exact 40-char SHA — not a tag. Update SHAs periodically (Dependabot will do this automatically if configured).

**Agent note — resolving `<PINNED-SHA>` placeholders**: AI agents cannot browse GitHub at scaffold time. Before scaffolding, the user (or a setup script) must populate the SHA lookup table below. Run the helper command for each action to get the current commit hash:

```shell
# Example: resolve actions/checkout latest release SHA
git ls-remote --tags https://github.com/actions/checkout | tail -1
# Or visit: https://github.com/actions/checkout/releases → copy commit SHA
```

| Placeholder in workflow       | Action                        | Minimum version | SHA (fill before scaffolding)      |
| ----------------------------- | ----------------------------- | --------------- | ---------------------------------- |
| `step-security/harden-runner` | `step-security/harden-runner` | v2.13+          | `________________________________` |
| `actions/checkout`            | `actions/checkout`            | v6.0+           | `________________________________` |
| `actions/setup-dotnet`        | `actions/setup-dotnet`        | v5.0+           | `________________________________` |
| `codecov/codecov-action`      | `codecov/codecov-action`      | v5.4+           | `________________________________` |
| `actions/upload-artifact`     | `actions/upload-artifact`     | v7.0+           | `________________________________` |
| `actions/download-artifact`   | `actions/download-artifact`   | v8.0+           | `________________________________` |

If the table above is empty at scaffold time, the agent **must** use version tags as a temporary fallback (e.g., `actions/checkout@v6`) and add a `TODO:` comment on each line: `# TODO: Pin to full SHA before merging to main`. The Phase 13 validation checklist will catch these.

---

### 9.2 `.github/dependabot.yml`

Automatically opens PRs to keep action SHAs and NuGet package versions current:

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
    directory: "/src"
    schedule:
      interval: "monthly"
    commit-message:
      prefix: "deps"
```

---

## Phase 10: `Build.cs` — Task Runner at Repo Root

A single **.NET 10 file-based app** at the repo root replaces any collection of shell scripts. File-based apps require no `.csproj` — they are compiled and run directly via `dotnet Build.cs`, and work on Windows, Linux, and macOS with any .NET 10 SDK installation.

> **Hard requirement**: `Build.cs` file-based apps require the **.NET 10 SDK** (the feature was introduced in .NET 10). If your repository must be buildable with an older SDK (e.g., because it multi-targets `net8.0` only and your CI does not install the .NET 10 SDK), this approach is **not usable** — fall back to PowerShell Core (`.ps1`) or a `Makefile`.

### Why prefer a C# file-based app over shell scripts?

| Concern         | Shell scripts (multiple files)        | `Build.cs` (1 file)                       |
| --------------- | ------------------------------------- | ----------------------------------------- |
| Cross-platform  | Requires `pwsh` or `bash` separately  | Works anywhere with the .NET 10 SDK       |
| Discoverability | Scattered across the repo, no help    | `dotnet Build.cs help` lists all commands |
| IDE support     | Limited                               | Full IntelliSense, compile-time checks    |
| Type safety     | Silent string-concat bugs             | Compiler catches errors                   |
| AOT publish     | N/A                                   | `dotnet publish Build.cs` → native binary |

### AOT note

File-based apps target native AOT **by default** in .NET 10. However, this task runner invokes `dotnet` CLI commands via `Process`, so it inherently requires the SDK at runtime. We disable AOT (`#:property PublishAot=false`) to keep the build-of-the-build-tool fast — it is JIT-compiled on first run and cached in the system temp directory.

To produce a true standalone binary (e.g., for CI runners): remove `#:property PublishAot=false` and run `dotnet publish Build.cs -r win-x64`. The resulting binary is platform-specific, so publish separately per target.

### Invocation

```
# From repo root (standard usage — works on Windows, Linux, macOS):
dotnet Build.cs bench
dotnet Build.cs pack
dotnet Build.cs rename CudaSharp MyNewLib

# Unix — make executable once, then invoke directly:
chmod +x Build.cs
./Build.cs bench
```

> **Explicit form** (when the `.NET 10` file-based shorthand is unavailable or arg passing needs disambiguation):
>
> ```
> dotnet run Build.cs -- bench
> dotnet run Build.cs -- rename CudaSharp MyNewLib
> ```

### Folder layout note

Place `Build.cs` at the **repo root**, one level above all `.csproj` files. Per .NET SDK guidance on [avoiding project file cones](https://learn.microsoft.com/en-us/dotnet/core/sdk/file-based-apps#avoid-project-file-cones), do **not** nest `Build.cs` inside a directory that contains a `.csproj`. The repo root has no `.csproj` at that level, so this layout is safe.

**Important**: use **LF line endings** in `Build.cs`. The `#!/usr/bin/env dotnet` shebang requires LF to work on Unix. The `*.cs text eol=lf` rule in `.gitattributes` handles this automatically.

### `Build.cs` — Complete file

```csharp
#!/usr/bin/env dotnet
// Task runner for the repository.
// Usage: dotnet Build.cs <command> [args]
// Requires: .NET 10 SDK (file-based apps are a .NET 10 feature).
// Invoke from any directory — [CallerFilePath] locates the repo root at compile time.

#:property PublishAot=false

using System.Diagnostics;
using System.Runtime.CompilerServices;

var repoRoot = RepoRoot();
var cmd = args.Length > 0 ? args[0] : "help";

switch (cmd)
{
    case "bench":
        Run("dotnet",
            "run --project src/<LIBNAME>.Benchmarks/<LIBNAME>.Benchmarks.csproj"
            + " -c Release -- --filter \"*\" --join",
            repoRoot);
        break;

    case "comparison-bench":
        Run("dotnet",
            "run --project src/<LIBNAME>.ComparisonBenchmarks/<LIBNAME>.ComparisonBenchmarks.csproj"
            + " -c Release -- run",
            repoRoot);
        break;

    case "disasm":
        Run("dotnet",
            "run --project src/<LIBNAME>.Benchmarks/<LIBNAME>.Benchmarks.csproj"
            + " -c Release -- --filter \"*<LIBNAME>*\" --disasm",
            repoRoot);
        break;

    case "pack":
        Run("dotnet",
            "pack src/<LIBNAME>/<LIBNAME>.csproj -c Release",
            repoRoot);
        break;

    case "publish-tester":
        var rid = args.Length > 1 ? args[1] : System.Runtime.InteropServices.RuntimeInformation.RuntimeIdentifier;
        Run("dotnet",
            $"publish src/<LIBNAME>.Tester/<LIBNAME>.Tester.csproj -c Release -r {rid} --self-contained",
            repoRoot);
        break;

    case "prettier":
        // Requires Node.js / npx on PATH.
        // Prettier respects .editorconfig for indent sizes since v2.0;
        // no separate .prettierrc is needed unless you want to override
        // specific Prettier behaviors beyond what .editorconfig covers.
        Run("npx",
            "prettier --write \"**/*.{yml,yaml,json,md}\" --ignore-path .gitignore",
            repoRoot);
        break;

    case "format":
        // CSharpier: opinionated whitespace/formatting
        // dotnet format style + analyzers: code style (naming, usings, etc.)
        // NOTE: never run bare "dotnet format" — the whitespace sub-check conflicts with CSharpier's Allman brace style.
        var verify = args.Length > 1 && args[1] == "check";
        Run("dotnet", "tool restore", repoRoot);
        Run("dotnet", verify ? "csharpier check ." : "csharpier format .", repoRoot);
        Run("dotnet", verify ? "format style --verify-no-changes" : "format style", repoRoot);
        Run("dotnet", verify ? "format analyzers --verify-no-changes" : "format analyzers", repoRoot);
        break;

    case "rename":
        if (args.Length < 3)
        {
            Console.Error.WriteLine("Usage: dotnet Build.cs rename <OldName> <NewName>");
            return 1;
        }
        RenameAll(repoRoot, args[1], args[2]);
        break;

    default:
        Console.WriteLine("""
            Usage: dotnet Build.cs <command>

            Commands:
              bench                     Run BenchmarkDotNet benchmarks
              comparison-bench          Run comparison benchmarks (writes to benchmarks/)
              disasm                    Inspect JIT disassembly via BDN disassembler
              pack                      Pack NuGet package locally
              publish-tester [RID]      Publish tester app (defaults to current platform RID)
              format [check]            Format C# (CSharpier) + code style (dotnet format); 'check' verifies only
              prettier                  Format YAML/Markdown/JSON via Prettier (requires Node)
              rename <OldName> <New>    Rename template throughout repository
            """);
        break;
}

return 0;

// ─── Helpers ─────────────────────────────────────────────────────────────────

static void Run(string exe, string arguments, string workingDir)
{
    var psi = new ProcessStartInfo(exe, arguments)
    {
        WorkingDirectory = workingDir,
        UseShellExecute = false,
    };
    using var p = Process.Start(psi) ?? throw new Exception($"Failed to start '{exe}'");
    p.WaitForExit();
    if (p.ExitCode != 0)
        throw new Exception($"'{exe}' exited with code {p.ExitCode}");
}

static void RenameAll(string repoRoot, string oldName, string newName)
{
    var ignore = new[] {
        Path.DirectorySeparatorChar + ".git" + Path.DirectorySeparatorChar,
        Path.DirectorySeparatorChar + "artifacts" + Path.DirectorySeparatorChar,
    };

    // Replace content and rename files
    var files = Directory.EnumerateFiles(repoRoot, "*", SearchOption.AllDirectories)
        .Where(f => !ignore.Any(seg => f.Contains(seg)))
        .ToList(); // snapshot before any renames

    var textExtensions = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
    {
        ".cs", ".csproj", ".props", ".targets", ".json", ".xml",
        ".yml", ".yaml", ".md", ".txt", ".slnx", ".sln",
        ".editorconfig", ".gitignore", ".gitattributes", ".config",
    };

    foreach (var file in files)
    {
        if (!textExtensions.Contains(Path.GetExtension(file))) { continue; }
        string content;
        try { content = File.ReadAllText(file); }
        catch { continue; } // skip binary or otherwise unreadable files
        if (content.Contains(oldName))
            File.WriteAllText(file, content.Replace(oldName, newName));

        var name = Path.GetFileName(file);
        if (name.Contains(oldName))
        {
            var newPath = Path.Combine(Path.GetDirectoryName(file)!, name.Replace(oldName, newName));
            File.Move(file, newPath);
        }
    }

    // Rename directories bottom-up (deepest first to avoid parent invalidation)
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

    Console.WriteLine($"Renamed '{oldName}' → '{newName}' throughout repository.");
}

// Returns the directory containing Build.cs (repo root) regardless of cwd.
// [CallerFilePath] is resolved at compile time to the absolute path of this file.
static string RepoRoot([CallerFilePath] string path = "") =>
    Path.GetDirectoryName(path)!;
```

---

## Phase 11: Benchmarks Output Directory

Create the directory structure with a placeholder:

```
benchmarks/
  .gitkeep
```

The CI and `dotnet Build.cs comparison-bench` will populate `benchmarks/<CPU.Name>/TestBench.md` and `benchmarks/<CPU.Name>/Versions.txt` automatically. The author typically pre-commits results from their own machine so the README shows real data from day one.

---

## Phase 11b: Seed Benchmark Results and Embed in README

This phase must be completed before Phase 13's validation checklist — the checklist requires a populated benchmark table before the scaffold is declared complete.

```shell
# 1. Build all projects first (required — comparison-bench runs in Release config)
dotnet build -c Release

# 2. Run comparison benchmarks — writes to benchmarks/<CPU.Name>/TestBench.md
dotnet Build.cs comparison-bench

# 3. Run full test suite — ReadMeTest_UpdateBenchmarksInMarkdown embeds the
#    benchmark tables and ReadMeTest_PublicApi embeds the public API into README.md
dotnet test -c Release
```

After all three commands succeed:

```shell
git add benchmarks/ README.md
git commit -m "chore: seed benchmark results and README tables"
```

**Why**: `ReadMeTest_UpdateBenchmarksInMarkdown` reads from `benchmarks/<CPU.Name>/` which doesn't exist until `comparison-bench` runs. Without this phase, that test crashes with an empty directory guard bypass but the README benchmark section remains empty — and the Phase 13 `0.0004 ns` validation step can never pass. Running benchmarks here also proves BDN, the comparison project, and the `RepoRoot()` path calculation all work correctly end-to-end.

---

## Phase 12: Required GitHub Repository Settings

After pushing the initial commit, configure these in the repository settings:

1. **Branch protection on `main`**: Require CI to pass before merge
2. **Codecov secret**: Add `CODECOV_TOKEN` to repository secrets (get from codecov.io)
3. **NuGet secret**: Add `NUGET_API_KEY` to repository secrets (get from nuget.org → Account → API Keys)
4. **Dependabot**: Enable for GitHub Actions (will keep action SHAs updated automatically)
5. **CodeQL**: _Optional._ Add `.github/workflows/codeql.yml` using GitHub’s standard CodeQL starter workflow template if security scanning is desired. See the Decision Table for guidance.

---

## Phase 13: Validation Checklist

Before declaring the scaffold complete, verify:

- [ ] `dotnet tool restore` succeeds and `dotnet csharpier format .` + `dotnet format style` + `dotnet format analyzers` have been run locally (no pending formatting changes before the first push)
- [ ] `dotnet build -c Debug` passes with zero warnings
- [ ] `dotnet build -c Release` passes with zero warnings (CSharpier.MsBuild checks formatting automatically)
- [ ] `dotnet test -c Debug` passes
- [ ] `dotnet test -c Release` passes
- [ ] `dotnet csharpier check .` passes (opinionated formatting)
- [ ] `dotnet format style --verify-no-changes` passes (naming/style diagnostics)
- [ ] `dotnet format analyzers --verify-no-changes` passes (Roslyn analyzers)
- [ ] `dotnet Build.cs format check` passes (runs CSharpier + dotnet format style + analyzers)
- [ ] `dotnet pack` produces a `.nupkg` and `.snupkg`
- [ ] README contains the `Empty()` method body under `## Example`
- [ ] README contains the public API snippet under `## Public API Reference`
- [ ] The benchmark result row in README shows `0.0004 ns` or similar (confirms BDN baseline was run)
- [ ] Pre-commit hooks are installed and `gitleaks` does not fire _(skip if Python/pre-commit unavailable — see Phase 1.8 prerequisite)_
- [ ] All action SHAs in `.github/workflows/dotnet.yml` are 40-char hashes, not version tags (or have `# TODO: Pin to full SHA` comments if lookup table was empty)
- [ ] `SECURITY.md` exists at repo root
- [ ] `CONTRIBUTING.md` exists at repo root
- [ ] `CHANGELOG.md` exists at repo root
- [ ] `.config/dotnet-tools.json` exists at repo root
- [ ] `.github/ISSUE_TEMPLATE/bug_report.md` and `feature_request.md` exist
- [ ] `.github/PULL_REQUEST_TEMPLATE.md` exists
- [ ] All `<LATEST_*>` package version placeholders have been resolved to real versions
- [ ] `dotnet Build.cs help` prints the command list without errors (smoke test the task runner)
- [ ] `dotnet Build.cs rename <LIBNAME> TestRename` runs without errors; revert with `dotnet Build.cs rename TestRename <LIBNAME>`

---

## Complete File Manifest

After all phases are complete, the repository contains exactly these files (substitute `<LIBNAME>` and `<RootClass>` values):

```
<LIBNAME>/                                      ← repo root
├── .config/
│   └── dotnet-tools.json
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   └── feature_request.md
│   ├── workflows/
│   │   └── dotnet.yml
│   ├── dependabot.yml
│   ├── FUNDING.yml
│   └── PULL_REQUEST_TEMPLATE.md
├── benchmarks/
│   └── .gitkeep
├── src/
│   ├── Directory.Build.props
│   ├── Directory.Build.targets
│   ├── <LIBNAME>/
│   │   ├── <LIBNAME>.csproj
│   │   └── <RootClass>.cs
│   ├── <LIBNAME>.Test/
│   │   ├── <LIBNAME>.Test.csproj
│   │   └── <RootClass>Test.cs
│   ├── <LIBNAME>.XyzTest/
│   │   ├── <LIBNAME>.XyzTest.csproj
│   │   ├── AssemblyInitializeCultureTest.cs
│   │   └── ReadMeTest.cs
│   ├── <LIBNAME>.Benchmarks/
│   │   ├── <LIBNAME>.Benchmarks.csproj
│   │   ├── <RootClass>Bench.cs
│   │   └── Program.cs
│   ├── <LIBNAME>.ComparisonBenchmarks/
│   │   ├── <LIBNAME>.ComparisonBenchmarks.csproj
│   │   ├── Program.cs
│   │   └── TestBench.cs
│   └── <LIBNAME>.Tester/
│       ├── <LIBNAME>.Tester.csproj
│       └── Program.cs
├── .editorconfig
├── .csharpierrc.json
├── .gitattributes
├── .gitignore
├── .jscpd.json
├── .markdownlint.json
├── .pre-commit-config.yaml
├── Build.cs
├── CHANGELOG.md
├── codecov.yml
├── CONTRIBUTING.md
├── global.json
├── Icon.png
├── LICENSE
├── nuget.config
├── README.md
├── SECURITY.md
└── <LIBNAME>.slnx
```

**Total**: 36 files across 13 directories. Phase 11b adds `benchmarks/<CPU.Name>/TestBench.md` and `benchmarks/<CPU.Name>/Versions.txt`.

---

## Decision Table: Common Customizations

| Scenario                                               | Change                                                                                                                                                                                                                                                                                                                                   |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Library wraps a native `.dll`/`.so`                    | Add `NativeLibrary.Load` override in static constructor; add platform-specific `<NativeLibraryPath>` to csproj                                                                                                                                                                                                                           |
| Multi-targeting (`net8.0;net10.0`)                     | Change `<TargetFramework>` to `<TargetFrameworks>` in Library csproj; guard README write tests with `#if NET10_0`                                                                                                                                                                                                                        |
| Library has no comparison alternative                  | Keep `ComparisonBenchmarks` but only add the one baseline benchmark; remove when irrelevant                                                                                                                                                                                                                                              |
| Not publishing to NuGet                                | Remove `create-release-push` job; keep `pack` for local testing                                                                                                                                                                                                                                                                          |
| Library uses `unsafe` code heavily                     | It's already `AllowUnsafeBlocks = true` everywhere; no extra steps                                                                                                                                                                                                                                                                       |
| Adding CodeQL for security scanning                    | Add `.github/workflows/codeql.yml` using the `github/codeql-action` action family                                                                                                                                                                                                                                                        |
| Type-heavy library (hundreds of types)                 | Keep one `partial` entry point; use nested partial classes per domain in separate files                                                                                                                                                                                                                                                  |
| Library has heavy string/span work                     | Add `[SkipLocalsInit]` attribute at module level for performance-critical paths                                                                                                                                                                                                                                                          |
| Many NuGet dependencies across projects                | Add `src/Directory.Packages.props` with `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>` and move all `Version="..."` attributes there. Not used in the default scaffold because there are only 4 distinct packages — but adopt immediately once the dependency count grows beyond ~6 to eliminate version drift. |
| Team uses JetBrains Rider                              | Replace `.slnx` with a traditional `.sln` via `dotnet new sln` + `dotnet sln add src/**/*.csproj`. Rider's `.slnx` support is partial (as of 2025.1).                                                                                                                                                                                    |
| Add OpenSSF Scorecard security checks                  | Add `.github/workflows/scorecard.yml` using `ossf/scorecard-action`. Produces automated supply-chain health checks and a public Scorecard badge. Pairs naturally with `harden-runner` and SHA-pinned actions already in the scaffold.                                                                                                    |
| `AnalysisLevel latest` breaks on SDK update            | Pin temporarily (`<AnalysisLevel>10</AnalysisLevel>`) to stop new analyzer rules blocking CI, fix each new violation, then restore `<AnalysisLevel>latest</AnalysisLevel>`. Do not permanently suppress analyzers.                                                                                                                       |
| CSharpier line width too narrow / too wide             | Edit `.csharpierrc.json` `printWidth` value. 100 is CSharpier default, 120 is the scaffold default. Do not go below 80 or above 150.                                                                                                                                                                                                     |
| Want to disable CSharpier build-time check temporarily | Set `<CSharpier_Check>false</CSharpier_Check>` in a `PropertyGroup` — do NOT remove the `CSharpier.MsBuild` package. Re-enable before merging.                                                                                                                                                                                           |

---

## Anti-Patterns to Avoid

1. **Do not use floating action tags** (`@v2`, `@main`). Always SHA-pin. Tags can be moved by maintainers.
2. **Do not skip the XyzTest/ReadMeTest pattern** just because it's "just a scaffold". The moment you skip it, documentation drift starts, and it never gets added back.
3. **Do not add multi-targeting unless the user explicitly requires it**. It doubles the build matrix and requires `#if` guards everywhere.
4. **Do not copy-paste domain-specific helpers from any reference project** into a new library. Every reference project carries domain-specific abstractions that are irrelevant in a different domain — design for your domain from scratch.
5. **Do not omit `fetch-depth: 0`** in the `pack` CI job. MinVer requires full git history to compute semantic versions from tag distance. Without it, every package is `0.0.0-preview.1`.
6. **Do not omit `<EmbedUntrackedSources>true</EmbedUntrackedSources>`**. Source Link requires this to allow debuggers to step into NuGet packages.
7. **Do not omit the `format` CI job**. Without it, formatting diverges silently across PRs until it becomes a big conflict.
8. **Do not put `InternalsVisibleTo` in `AssemblyInfo.cs`**. Use `<AssemblyAttribute>` MSBuild items in the csproj — it auto-generates the attribute and is refactoring-safe.
9. **Do not skip CSharpier or rely solely on `.editorconfig` for formatting**. `.editorconfig` formatting rules are advisory in most IDEs — only `dotnet format` enforces them, and it is lenient about whitespace. CSharpier is deterministic: two developers formatting the same file will always get identical output. Without it, formatting "wars" in PRs are inevitable.
10. **Do not remove `CSharpier.MsBuild` to "speed up builds"**. The overhead is negligible (<1s per project) and the alternative — discovering formatting violations only in CI after a 10-minute round trip — is far more expensive.
