# Agent Guide: Creating a "nietras-quality" Large .NET Application Solution

> **Purpose**: Step-by-step instructions for an AI agent to scaffold a large .NET application solution (game client, desktop app, tool suite) with ~100 library projects and an executable entry point — matching the engineering rigor of the [Library Scaffold Guide](NietrasScaffold_LibSolution_AgentGuide.md) but adapted for application-scale architecture.
> **Companion to**: `NietrasScaffold_LibSolution_AgentGuide.md` (library-focused). This guide references that document for shared root-config patterns and avoids duplicating them.
> **Inputs required from user before starting**: application name, domain layers, target platforms, whether native interop is needed.
> **Output**: a complete, CI-ready, publishable application repository scaffold with layered architecture and zero-tolerance code quality from commit one.

---

## How This Guide Relates to the Library Guide

The [Library Scaffold Guide](NietrasScaffold_LibSolution_AgentGuide.md) targets a **single NuGet library** (or a small set of related libraries) with satellite test/benchmark projects (~6 projects, 36 files). This guide targets a **large application** with many internal libraries — the kind of solution where you open Visual Studio and see 100+ projects.

**Reused verbatim** from the library guide (read those sections there — not duplicated here):
- Phase 1.1 `global.json`
- Phase 1.2 `nuget.config`
- Phase 1.3 `.gitignore` (with additions noted below)
- Phase 1.4 `.gitattributes`
- Phase 1.5 `.editorconfig` (with additions noted below)
- Phase 1.6 `.jscpd.json`
- Phase 1.7 `.markdownlint.json`
- Phase 1.8 `.pre-commit-config.yaml`
- Phase 1.11 `LICENSE`
- Phase 1.12 `SECURITY.md`
- Phase 1.13 `CONTRIBUTING.md`
- Phase 1.16 `.config/dotnet-tools.json` (pre-populated with CSharpier)
- Phase 1.16a `.csharpierrc.json`
- Phase 1.17 Issue/PR templates

**Replaced entirely** in this guide:
- Solution file (`.slnx`) — needs solution folders for 100+ projects
- `Directory.Build.props` — needs hierarchical layered configuration
- Project structure — layered domain folders, not flat `src/`
- CI pipeline — builds an app, not a NuGet package
- `Build.cs` task runner — different commands (run, publish, no pack/nuget)
- README — application README, not library API reference
- Test strategy — per-layer unit tests + integration tests, not README-sync
- Benchmarks — optional profiling harness, not comparison-against-competitors

**Not applicable** (library-only concerns removed):
- NuGet packaging (Icon.png, PackageReadmeFile, snupkg, MinVer)
- ReadMeTest pattern (README-sync from test code)
- ComparisonBenchmarks against competing libraries
- PublicApiGenerator snapshot testing
- Codecov integration (optional for apps — add if desired)

---

## Phase 0: Gather Inputs

Ask the user for (or infer from context):

| Variable | Example | Notes |
|---|---|---|
| `APPNAME` | `Edelstein` | PascalCase; becomes solution name and exe name |
| `ROOTNS` | `Edelstein` | Root namespace prefix; all projects use `<ROOTNS>.<Layer>.<Module>` |
| `DESCRIPTION` | `MapleStory v95 game client reimplementation` | One sentence for README |
| `AUTHOR` | `yourname` | GitHub username |
| `DOTNET_VERSION` | `net10.0` | Target framework for all projects |
| `CSHARP_VERSION` | `14.0` | Match to chosen .NET version |
| `SDK_VERSION` | `10.0.103` | Exact SDK version from `dotnet --version` |
| `GITHUB_URL` | `https://github.com/you/APPNAME` | Full repo URL |
| `LICENSE` | `MIT` | License type |
| `YEAR` | `2026` | Copyright year |
| `COMPANY` | `yourname` | Used in copyright |
| `TARGET_PLATFORMS` | `win-x64` | Semicolon-separated RIDs for publish: `win-x64;linux-x64` |
| `NEEDS_UNSAFE` | `true` | true for P/Invoke, bit manipulation, SIMD, packet manipulation |
| `NEEDS_NATIVE` | `true` | true if shipping native `.dll`/`.so` alongside the exe |
| `APP_SDK` | `Microsoft.NET.Sdk` | SDK for the app project. Use `Microsoft.NET.Sdk` for console, or custom for windowed apps |
| `DOMAIN_LAYERS` | (see Phase 2) | Ordered list of domain layer folders and their purpose |

### Domain Layer Interview

The most important input is the **domain layer breakdown**. Ask the user to describe the major subsystems of their application. For a game client, a typical breakdown is:

```
Core        → Primitives, math, collections, memory utilities
Data        → Asset loading, data templates, string tables
Crypto      → CRC, encryption, packet cipher
Network     → Packet I/O, connection management, protocol types
Game        → Game logic: combat, inventory, quests, field management
Rendering   → Graphics backend, sprites, animation, compositing
UI          → UI framework, windows, controls, text rendering
Audio       → Sound effects, background music
App         → The executable entry point, startup orchestration
```

The user may add, remove, or rename layers. Record the final list as `DOMAIN_LAYERS` — it drives the entire directory structure.

---

## Phase 0b: Initialize Git Repository

Identical to the library guide:

```shell
mkdir <APPNAME> && cd <APPNAME>
git init -b main
```

**Verify git authorship** before the first commit — see the library guide's Phase 0b for the full verification and fix commands. In short:

```shell
git config user.name   # must print the real name
git config user.email  # must print the real email
```

> **Agent note**: Do not set or override `user.name`/`user.email` yourself. Flag any mismatch to the user.

### Recommended commit strategy

```shell
# Commit 1: Root config files (Phase 1)
git add .editorconfig .csharpierrc.json .gitattributes .gitignore .jscpd.json .markdownlint.json \
    .pre-commit-config.yaml .config/ global.json nuget.config \
    LICENSE SECURITY.md CONTRIBUTING.md CHANGELOG.md \
    .github/ Directory.Packages.props README.md <APPNAME>.slnx
git commit -m "chore: root config and repository health files"

# Commit 2: Build infrastructure (Directory.Build.props hierarchy, Build.cs)
git add src/ Build.cs
git commit -m "chore: layered project structure and Build.cs task runner"

# Commit 3: CI pipeline
git add .github/workflows/ .github/dependabot.yml
git commit -m "ci: GitHub Actions workflow and Dependabot"
```

---

## Phase 1: Repository Root Files

**Reuse from the Library Guide**: Create all files listed in Library Guide Phases 1.1–1.8, 1.11–1.13, 1.16–1.17 verbatim. Then apply the modifications below.

### 1.3a `.gitignore` additions

Append to the base `.gitignore` from the library guide:

```gitignore
# Centralized build output (UseArtifactsOutput)
artifacts/

# Native build output (if building native dependencies from source)
build/out/

# Publish output
publish/

# Solution filters are user-specific working sets — do NOT ignore .slnf files
# (they are committed so the team shares standard subsets)

# Local launch profiles override
Properties/launchSettings.local.json
```

### 1.5a `.editorconfig` additions

Append after the library guide's `.editorconfig` content:

```editorconfig
# Solution-wide: ban 'var' when the type is not apparent (large codebase readability)
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_style_var_elsewhere = false:suggestion

# Primary constructors — prefer for DI injection patterns in large codebases
csharp_style_prefer_primary_constructors = true:suggestion

# MA0008 (Meziantou) StructLayout missing on unmanaged structs — fires on P/Invoke structs by design
dotnet_diagnostic.MA0008.severity = none
# MA0051 (Meziantou) Method too long — large codebases have legitimate long methods (parsers, handlers)
dotnet_diagnostic.MA0051.severity = none
```

**CSharpier integration note**: The \`.editorconfig\` must set \IDE0055.severity = none\ -- **do not use \rror\**. CSharpier owns whitespace formatting (indentation, line wrapping, brace placement). \dotnet format\ must always be run as two explicit sub-checks -- \dotnet format style\ and \dotnet format analyzers\ -- never as bare \dotnet format\. Running bare \dotnet format\ triggers the \whitespace\ sub-check which rewrites CSharpier's Allman-style brace formatting, producing \WHITESPACE\ errors in CI even on correctly formatted code. See the library guide's Phase 1.16a for the full explanation.

### 1.9a `codecov.yml` — Optional for Apps

Codecov is valuable for libraries (consumers rely on test coverage). For applications it is optional. If the user wants coverage tracking, include `codecov.yml` from the library guide. No additional coverage package is needed — TUnit already bundles `Microsoft.Testing.Extensions.CodeCoverage` transitively.

> ⚠️ **Do NOT use `coverlet.collector` or `coverlet.msbuild` with TUnit.** TUnit runs on Microsoft Testing Platform (MTP), not VSTest. Coverlet's VSTest data collectors are [not compatible with MTP](https://tunit.dev/docs/extending/code-coverage/) and will produce empty coverage. Coverage is collected via the `--coverage` flag (e.g., `dotnet test --coverage --coverage-output-format cobertura`). See the library guide's Phase 5.1 for the full explanation.
>
> **VS Code users**: Install the [Coverage Gutters](https://marketplace.visualstudio.com/items?itemName=ryanluker.vscode-coverage-gutters) extension to view inline coverage highlighting from the generated `coverage.cobertura.xml`.

### 1.10a Solution file — Deferred to Phase 10

The `.slnx` for a 100-project solution requires solution folders and cannot be written until all projects are defined. A placeholder is created here; it is fully populated in Phase 10.

### 1.14a `CHANGELOG.md`

```markdown
# Changelog

All notable changes to this project are documented in [GitHub Releases](https://github.com/<AUTHOR>/<APPNAME>/releases).
```

No MinVer reference — applications typically use build numbers or manual versioning.

### 1.18a `Icon.png` — Optional for Apps

Not required (no NuGet package). If the application has a window icon or tray icon, place it in `src/App/<APPNAME>/Resources/` instead. Skip the repo-root `Icon.png`.

### 1.19a `README.md`

```markdown
# <APPNAME>

![.NET](https://img.shields.io/badge/<DOTNET_VERSION>-5C2D91?logo=.NET&labelColor=gray)
![C#](https://img.shields.io/badge/C%23-<CSHARP_VERSION>-239120?labelColor=gray)
[![Build Status](https://github.com/<AUTHOR>/<APPNAME>/actions/workflows/dotnet.yml/badge.svg?branch=main)](<GITHUB_URL>/actions/workflows/dotnet.yml)
[![License](https://img.shields.io/github/license/<AUTHOR>/<APPNAME>)](<GITHUB_URL>/blob/main/LICENSE)

<DESCRIPTION>

## Architecture

```
src/
├── Core/           Primitives, math, collections, memory
├── Data/           Asset loading, templates, string tables
├── Crypto/         CRC, encryption, packet cipher
├── Network/        Protocol I/O, connection management
├── Game/           Game logic: combat, inventory, quests
├── Rendering/      Graphics, sprites, animation
├── UI/             UI framework, windows, controls
├── Audio/          Sound effects, BGM
└── App/            Executable entry point
```

See [Solution Structure](#solution-structure) for the full project breakdown.

## Getting Started

### Prerequisites

- [.NET <DOTNET_VERSION> SDK](https://dotnet.microsoft.com/download)

### Setup

```shell
dotnet tool restore   # Install local tools (CSharpier formatter)
```

### Build

```shell
dotnet build
```

### Run

```shell
dotnet Build.cs run
# or directly:
dotnet run --project src/App/<APPNAME>/<APPNAME>.csproj
```

### Test

```shell
dotnet test
```

### Publish

```shell
dotnet Build.cs publish win-x64
```

## Solution Structure

| Layer | Projects | Depends On |
|---|---|---|
| `Core` | Primitives, math, collections | (none — leaf layer) |
| `Data` | Asset loading, templates | Core |
| `Crypto` | CRC, cipher | Core |
| `Network` | Packets, connection | Core, Crypto |
| `Game` | Combat, inventory, quest, field | Core, Data, Network |
| `Rendering` | Graphics, sprites, animation | Core, Data |
| `UI` | Windows, controls, text | Core, Rendering |
| `Audio` | SFX, BGM | Core, Data |
| `App` | Entry point | All layers |

## Development

### Prerequisites

- [.NET <DOTNET_VERSION> SDK](https://dotnet.microsoft.com/download)
- Run `dotnet tool restore` after cloning to install local tools (CSharpier formatter)

### Task Runner

All common operations are available via `Build.cs`:

```shell
dotnet Build.cs help
```

### Code Formatting

This project uses **CSharpier** (opinionated C# formatter) and **`dotnet format`** (code style analyzers):

```shell
# Auto-format everything:
dotnet Build.cs format

# Check only (CI-style, no changes):
dotnet Build.cs format check
```

IDE integration: Install the CSharpier extension for [VS Code](https://marketplace.visualstudio.com/items?itemName=csharpier.csharpier-vscode), [Visual Studio](https://marketplace.visualstudio.com/items?itemName=csharpier.CSharpier), or [Rider](https://plugins.jetbrains.com/plugin/18243-csharpier) for format-on-save.

### Solution Filters

To load a subset of the solution in your IDE:

- `Core.slnf` — Core libraries + their tests only
- `Network.slnf` — Network + Crypto + Core + tests
- `Game.slnf` — Game + Data + Network + Core + tests
- `All.slnf` — Everything (same as opening the `.slnx`)

## License

[<LICENSE>](<GITHUB_URL>/blob/main/LICENSE) © <YEAR> <COMPANY>
```

**Agent note**: Update the Architecture tree and Solution Structure table to match the user's actual `DOMAIN_LAYERS`. The examples above use the game client breakdown.

---

## Phase 2: Central Package Management

At application scale, **Central Package Management (CPM)** is mandatory from day one. Without it, 100 projects will inevitably drift to different versions of the same package.

### 2.1 `Directory.Packages.props` (repo root)

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
  </PropertyGroup>

  <!-- ═══════════════════════════════════════════════════════
       FRAMEWORK & BUILD
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="Build">
    <!-- CSharpier: opinionated C# formatting, enforced on every build via CSharpier.MsBuild -->
    <PackageVersion Include="CSharpier.MsBuild" Version="<LATEST_CSHARPIER>" />
  </ItemGroup>

  <!-- ═══════════════════════════════════════════════════════
       CORE / SHARED
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="Core">
    <PackageVersion Include="System.IO.Pipelines" Version="10.0.0" />
    <PackageVersion Include="System.Runtime.CompilerServices.Unsafe" Version="6.1.0" />
    <!-- Add as needed: System.Memory, System.Buffers, etc. -->
  </ItemGroup>

  <!-- ═══════════════════════════════════════════════════════
       LOGGING
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="Logging">
    <PackageVersion Include="Microsoft.Extensions.Logging.Abstractions" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Logging" Version="10.0.0" />
    <PackageVersion Include="Serilog" Version="4.3.0" />
    <PackageVersion Include="Serilog.Sinks.Console" Version="6.0.0" />
    <PackageVersion Include="Serilog.Extensions.Logging" Version="9.0.0" />
  </ItemGroup>

  <!-- ═══════════════════════════════════════════════════════
       DEPENDENCY INJECTION
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="DI">
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Hosting" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Options" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Configuration" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Configuration.Json" Version="10.0.0" />
  </ItemGroup>

  <!-- ═══════════════════════════════════════════════════════
       TESTING
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="Testing">
    <PackageVersion Include="TUnit" Version="3.6.41" />
    <PackageVersion Include="NSubstitute" Version="5.3.0" />
  </ItemGroup>

  <!-- ═══════════════════════════════════════════════════════
       BENCHMARKS
       ═══════════════════════════════════════════════════════ -->
  <ItemGroup Label="Benchmarks">
    <PackageVersion Include="BenchmarkDotNet" Version="0.14.0" />
  </ItemGroup>

</Project>
```

**Critical rules for CPM**:
- **All `<PackageReference>` in `.csproj` files omit `Version`** — the version comes only from `Directory.Packages.props`.
- **`CentralPackageTransitivePinningEnabled`** pins transitive dependencies too, preventing supply-chain version confusion.
- Package versions shown above are **examples** — resolve current stable versions via `dotnet package search <name> --take 1` before scaffolding.
- Organize by labeled `<ItemGroup>` sections for readability. With 50+ packages, flat lists become unmanageable.

**Agent note**: The package list above is a **starter template for a game client**. Remove packages the user doesn't need. Add domain-specific packages (graphics libraries, audio libraries, etc.) as the user specifies them. The point is the *structure*, not the exact package set. CSharpier.MsBuild is the one **non-optional** package — it enforces formatting from build one.

> **Resolving `<LATEST_CSHARPIER>`**: Run `dotnet package search CSharpier --take 1` or visit [nuget.org/packages/CSharpier](https://www.nuget.org/packages/CSharpier/). The version in `Directory.Packages.props` must match the version in `.config/dotnet-tools.json` — they are the same tool, referenced in two ways (MSBuild analyzer vs CLI tool).

---

## Phase 3: Hierarchical `Directory.Build.props`

A large solution needs **layered build configuration**. MSBuild automatically discovers `Directory.Build.props` walking up from each project to the repo root. We exploit this with a three-tier hierarchy:

```
<APPNAME>/                              ← repo root
├── Directory.Build.props               ← Tier 1: global settings (ALL projects)
├── Directory.Packages.props            ← Central Package Management
└── src/
    ├── Directory.Build.props           ← Tier 2: source project defaults
    ├── Directory.Build.targets         ← Shared targets (usually minimal)
    ├── Core/
    │   └── <APPNAME>.Core/
    │       └── <APPNAME>.Core.csproj
    ├── App/
    │   ├── Directory.Build.props       ← Tier 3: app-layer overrides
    │   └── <APPNAME>/
    │       └── <APPNAME>.csproj
    └── Tests/
        ├── Directory.Build.props       ← Tier 3: test-layer overrides
        └── <APPNAME>.Core.Tests/
            └── <APPNAME>.Core.Tests.csproj
```

### 3.1 Tier 1: Root `Directory.Build.props` (repo root)

Settings that apply to **every single project** — source code, tests, benchmarks, the app itself:

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

    <!-- Artifact output layout: all bins go to artifacts/ at repo root -->
    <UseArtifactsOutput>true</UseArtifactsOutput>
    <ArtifactsPath>$(MSBuildThisFileDirectory)artifacts</ArtifactsPath>

    <!-- Generate docs XML; suppress missing-doc warnings globally -->
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

  <!-- CSharpier formatting enforcement: checks formatting on every build.
       Violations appear as build errors when TreatWarningsAsErrors is enabled.
       Install the local tool first: dotnet tool restore
       Version is centrally managed in Directory.Packages.props -->
  <ItemGroup>
    <PackageReference Include="CSharpier.MsBuild" PrivateAssets="All" />
  </ItemGroup>

</Project>
```

**What's NOT here** (compared to the library guide):
- No `MinVer` — applications use assembly version, `Version` property, or build-injected version
- No `<IsPackable>` default — set per-project or per-tier
- No NuGet metadata — we're not shipping packages
- No `PublishRelease`/`PackRelease` — those are library publishing concerns

### 3.2 Tier 2: `src/Directory.Build.props`

Settings shared by all **source projects** (libraries + app) but NOT tests or benchmarks:

```xml
<Project>

  <!-- Import the parent (repo root) Directory.Build.props -->
  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />

  <PropertyGroup>
    <!-- No source project is a NuGet package by default -->
    <IsPackable>false</IsPackable>

    <!-- All source projects allow unsafe by default for a game client.
         Remove this line if your app has no unsafe code. -->
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>

    <!-- Non-CLS-compliant: common for game clients with interop.
         REMOVE for pure managed applications. -->
    <CLSCompliant>false</CLSCompliant>
  </PropertyGroup>

  <ItemGroup>
    <AssemblyAttribute Include="System.CLSCompliantAttribute">
      <_Parameter1>false</_Parameter1>
    </AssemblyAttribute>
  </ItemGroup>

</Project>
```

### 3.2b `src/Directory.Build.targets`

Minimal shared targets file. Exists so MSBuild's auto-import chain isn't broken:

```xml
<Project>
  <!-- Add shared MSBuild targets here if needed.
       For most applications this file stays empty. -->
</Project>
```

### 3.3 Tier 3: Layer-specific overrides

Each layer that needs different settings gets its own `Directory.Build.props`. Most layers don't need one — they inherit Tier 2 directly.

#### `src/App/Directory.Build.props` — Application layer

```xml
<Project>

  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />

  <PropertyGroup>
    <!-- App projects produce executables -->
    <OutputType>Exe</OutputType>

    <!-- Version stamping for the application -->
    <Version>0.1.0</Version>
    <AssemblyVersion>0.1.0.0</AssemblyVersion>
    <FileVersion>0.1.0.0</FileVersion>
    <InformationalVersion>0.1.0-dev</InformationalVersion>
  </PropertyGroup>

</Project>
```

**Why `Version` here instead of MinVer**: Application versioning is fundamentally different from library versioning. Libraries need semver for NuGet compatibility. Applications need human-meaningful version strings that map to releases, and can be injected at build time:

```shell
# CI overrides version at build time:
dotnet build -p:Version=1.2.3 -p:InformationalVersion=1.2.3+abcdef1
```

#### `src/Tests/Directory.Build.props` — Test layer

```xml
<Project>

  <!-- Import from repo root (skip src/ tier — tests don't need AllowUnsafeBlocks etc.) -->
  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../../'))" />

  <PropertyGroup>
    <IsPackable>false</IsPackable>
    <!-- Allow unsafe in tests only if test code directly tests unsafe APIs -->
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    <!-- ConfigureAwait not relevant in test code -->
    <NoWarn>$(NoWarn);CA2007</NoWarn>
  </PropertyGroup>

  <!-- Exclude all test assemblies from code coverage metrics -->
  <ItemGroup>
    <AssemblyAttribute Include="System.Diagnostics.CodeAnalysis.ExcludeFromCodeCoverageAttribute" />
  </ItemGroup>

  <!-- Common test dependencies — every test project gets these automatically -->
  <ItemGroup>
    <PackageReference Include="TUnit" />
    <PackageReference Include="NSubstitute" />
  </ItemGroup>

</Project>
```

**Why test packages go in Tier 3**: Every test project needs `TUnit` and a mocking framework. Declaring them once in the layer `Directory.Build.props` eliminates the need to repeat `<PackageReference Include="TUnit" />` in 30+ test `.csproj` files.

> ⚠️ **TUnit + .NET 10 SDK compatibility**: TUnit uses `Microsoft.Testing.Platform` (MTP), which conflicts with the legacy VSTest runner that `dotnet test` uses by default on .NET 10 SDK. On .NET 10, `dotnet test` will fail with a `Microsoft.Testing.Platform.MSBuild.targets(263)` error unless you add this property to `Directory.Build.props` (the root one, **not** inside any test `.csproj`):
>
> ```xml
> <TestingPlatformDotnetTestSupport>true</TestingPlatformDotnetTestSupport>
> ```
>
> Add it under a `<PropertyGroup>` in the root `Directory.Build.props`. At application scale with 30+ test projects, this single line is much cleaner than configuring each project individually.

#### `src/Benchmarks/Directory.Build.props` — Benchmark layer (optional)

```xml
<Project>

  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../../'))" />

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <IsPackable>false</IsPackable>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    <PlatformTarget>AnyCPU</PlatformTarget>
    <DebugType>pdbonly</DebugType>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="BenchmarkDotNet" />
  </ItemGroup>

</Project>
```

---

## Phase 4: Source Directory Structure

This is the heart of the guide. Each domain layer is a folder under `src/` containing one or more library projects. The naming convention is `<APPNAME>.<Layer>` or `<APPNAME>.<Layer>.<Module>` for larger layers that split into multiple assemblies.

### 4.1 Reference Architecture (Game Client)

```
src/
├── Directory.Build.props                           ← Tier 2 (all source projects)
├── Directory.Build.targets
│
├── Core/                                           ← LAYER: Foundation (no dependencies)
│   ├── <APPNAME>.Core/                             ← Primitives, math types, Result<T>
│   │   ├── <APPNAME>.Core.csproj
│   │   ├── Math/
│   │   │   └── Vector2D.cs
│   │   ├── Memory/
│   │   │   └── PooledBuffer.cs
│   │   └── Primitives/
│   │       ├── FieldId.cs
│   │       └── ItemId.cs
│   └── <APPNAME>.Core.Collections/                 ← Specialized collections (optional split)
│       ├── <APPNAME>.Core.Collections.csproj
│       └── FixedCapacityList.cs
│
├── Data/                                           ← LAYER: Data loading
│   ├── <APPNAME>.Data/                             ← Abstract data access interfaces
│   │   ├── <APPNAME>.Data.csproj
│   │   └── IDataProvider.cs
│   └── <APPNAME>.Data.Wz/                          ← WZ file format implementation
│       ├── <APPNAME>.Data.Wz.csproj
│       └── WzReader.cs
│
├── Crypto/                                         ← LAYER: Cryptography
│   └── <APPNAME>.Crypto/
│       ├── <APPNAME>.Crypto.csproj
│       ├── MapleAes.cs
│       └── MapleCrc.cs
│
├── Network/                                        ← LAYER: Networking
│   ├── <APPNAME>.Network/                          ← Packet I/O, connection management
│   │   ├── <APPNAME>.Network.csproj
│   │   ├── PacketReader.cs
│   │   └── MapleClient.cs
│   └── <APPNAME>.Network.Protocol/                 ← Opcode definitions, packet structures
│       ├── <APPNAME>.Network.Protocol.csproj
│       └── Opcodes/
│           ├── RecvOps.cs
│           └── SendOps.cs
│
├── Game/                                           ← LAYER: Game logic
│   ├── <APPNAME>.Game/                             ← Core game state, shared abstractions
│   │   ├── <APPNAME>.Game.csproj
│   │   └── GameState.cs
│   ├── <APPNAME>.Game.Combat/                      ← Combat mechanics
│   │   ├── <APPNAME>.Game.Combat.csproj
│   │   └── DamageCalculator.cs
│   ├── <APPNAME>.Game.Inventory/                   ← Inventory system
│   │   ├── <APPNAME>.Game.Inventory.csproj
│   │   └── InventoryManager.cs
│   ├── <APPNAME>.Game.Quest/                       ← Quest system
│   │   ├── <APPNAME>.Game.Quest.csproj
│   │   └── QuestManager.cs
│   └── <APPNAME>.Game.Field/                       ← Map/field management
│       ├── <APPNAME>.Game.Field.csproj
│       └── FieldManager.cs
│
├── Rendering/                                      ← LAYER: Graphics
│   ├── <APPNAME>.Rendering/                        ← Abstract rendering interfaces
│   │   ├── <APPNAME>.Rendering.csproj
│   │   └── IRenderer.cs
│   └── <APPNAME>.Rendering.Sprites/                ← Sprite loading and animation
│       ├── <APPNAME>.Rendering.Sprites.csproj
│       └── SpriteSheet.cs
│
├── UI/                                             ← LAYER: User interface
│   └── <APPNAME>.UI/
│       ├── <APPNAME>.UI.csproj
│       └── UIManager.cs
│
├── Audio/                                          ← LAYER: Audio
│   └── <APPNAME>.Audio/
│       ├── <APPNAME>.Audio.csproj
│       └── AudioManager.cs
│
├── App/                                            ← LAYER: Application entry point
│   ├── Directory.Build.props                       ← Tier 3 (OutputType=Exe, version)
│   └── <APPNAME>/
│       ├── <APPNAME>.csproj
│       ├── Program.cs
│       └── appsettings.json
│
├── Tests/                                          ← LAYER: All tests
│   ├── Directory.Build.props                       ← Tier 3 (TUnit, test config)
│   ├── <APPNAME>.Core.Tests/
│   │   ├── <APPNAME>.Core.Tests.csproj
│   │   └── Math/
│   │       └── Vector2DTests.cs
│   ├── <APPNAME>.Crypto.Tests/
│   │   ├── <APPNAME>.Crypto.Tests.csproj
│   │   └── MapleCrcTests.cs
│   ├── <APPNAME>.Network.Tests/
│   │   ├── <APPNAME>.Network.Tests.csproj
│   │   └── PacketReaderTests.cs
│   ├── <APPNAME>.Game.Combat.Tests/
│   │   ├── <APPNAME>.Game.Combat.Tests.csproj
│   │   └── DamageCalculatorTests.cs
│   └── <APPNAME>.Integration.Tests/                ← Cross-layer integration tests
│       ├── <APPNAME>.Integration.Tests.csproj
│       └── StartupTests.cs
│
└── Benchmarks/                                     ← LAYER: Performance benchmarks (optional)
    ├── Directory.Build.props                       ← Tier 3 (BDN, Exe output)
    └── <APPNAME>.Benchmarks/
        ├── <APPNAME>.Benchmarks.csproj
        ├── Program.cs
        └── PacketReaderBench.cs
```

### 4.2 Layer Dependency Rules

These rules **must be enforced** — violations create circular dependencies that cripple build times and testability:

```
Core           → (nothing)                          ← Pure leaf. Zero external references.
Data           → Core
Crypto         → Core
Network        → Core, Crypto
Game           → Core, Data, Network
Rendering      → Core, Data
UI             → Core, Rendering
Audio          → Core, Data
App            → ALL layers (composition root)
Tests          → The layer they test + test infra
Benchmarks     → The layer they benchmark + BDN
```

**Enforcement mechanism**: The dependency rules are enforced by `<ProjectReference>` in each `.csproj`. There is no magic — if `Rendering` has no `<ProjectReference>` to `Network`, it cannot use network types. The agent must verify no `.csproj` violates these rules.

For teams wanting automated enforcement, add an architecture test:

```csharp
// In <APPNAME>.Integration.Tests/ArchitectureTests.cs
[Test]
public void Core_HasNoProjectReferences()
{
    // Read Core .csproj, assert zero <ProjectReference> elements
}

[Test]
public void Rendering_DoesNotReference_Network()
{
    // Read Rendering .csproj, assert no reference to Network projects
}
```

### 4.3 When to Split a Layer into Multiple Projects

A layer starts as **one project**. Split into multiple projects when:

| Signal | Action |
|---|---|
| Two subsystems within a layer have **no shared types** | Split: `Game.Combat`, `Game.Inventory` |
| One part of a layer needs different dependencies | Split: `Rendering` (abstract) vs `Rendering.DirectX` (platform-specific) |
| Build time within one project exceeds ~5 seconds | Consider splitting by subdomain |
| A test project needs to reference only part of a layer | Split the layer so the test references the smaller piece |

Do **not** split preemptively. Start with one project per layer and split as the codebase grows.

---

## Phase 5: Core Library Projects

### 5.1 Template `.csproj` for Internal Libraries

All library projects within the solution follow this minimal template. The heavy lifting is done by the inherited `Directory.Build.props` chain:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.Core</RootNamespace>
  </PropertyGroup>

  <!-- Core has NO project references — it's the leaf layer -->

</Project>
```

**That's it.** Compare this to the library guide's 50-line `.csproj` — everything else is inherited:
- `TargetFramework` → from root `Directory.Build.props`
- `Nullable`, `ImplicitUsings`, `LangVersion` → from root
- `AllowUnsafeBlocks` → from `src/Directory.Build.props`
- `IsPackable=false` → from `src/Directory.Build.props`
- `TreatWarningsAsErrors` → from root
- Package versions → from `Directory.Packages.props`

### 5.2 Template `.csproj` with Dependencies

A project that references other layers:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.Network</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\Core\<APPNAME>.Core\<APPNAME>.Core.csproj" />
    <ProjectReference Include="..\..\Crypto\<APPNAME>.Crypto\<APPNAME>.Crypto.csproj" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="System.IO.Pipelines" />
    <!-- No Version attribute — CPM provides it from Directory.Packages.props -->
  </ItemGroup>

</Project>
```

### 5.3 Template `.csproj` with InternalsVisibleTo

When a library exposes internal APIs to its test project:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.Crypto</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\Core\<APPNAME>.Core\<APPNAME>.Core.csproj" />
  </ItemGroup>

  <!-- Allow the test project to access internal types -->
  <ItemGroup>
    <AssemblyAttribute Include="System.Runtime.CompilerServices.InternalsVisibleTo">
      <_Parameter1><APPNAME>.Crypto.Tests</_Parameter1>
    </AssemblyAttribute>
  </ItemGroup>

</Project>
```

### 5.4 Starter Source Files

Each library project needs at least one compilable file. Create a minimal entry point per project:

**`src/Core/<APPNAME>.Core/Primitives/FieldId.cs`** — Example strongly-typed ID:
```csharp
namespace <ROOTNS>.Core.Primitives;

/// <summary>
/// Strongly-typed field (map) identifier.
/// </summary>
public readonly record struct FieldId(int Value)
{
    public static readonly FieldId None = new(0);

    public override string ToString() => Value.ToString();
}
```

**`src/Crypto/<APPNAME>.Crypto/MapleCrc.cs`** — Example placeholder:
```csharp
namespace <ROOTNS>.Crypto;

public static class MapleCrc
{
    public static uint Compute(ReadOnlySpan<byte> data)
    {
        // TODO: Implement CRC32 computation
        throw new NotImplementedException();
    }
}
```

**Agent note**: Create **one** file per project with enough real structure to compile and be testable. Do not create empty placeholder classes — create the actual type that the project will contain, even if the implementation is `throw new NotImplementedException()`. This proves the project reference chain compiles end-to-end.

---

## Phase 6: The Application Project

### 6.1 `src/App/<APPNAME>/<APPNAME>.csproj`

The composition root — the only project that references all layers:

```xml
<Project Sdk="<APP_SDK>">

  <PropertyGroup>
    <RootNamespace><ROOTNS></RootNamespace>
    <!-- OutputType=Exe comes from src/App/Directory.Build.props -->

    <!-- Self-contained publish for distribution -->
    <PublishSingleFile>true</PublishSingleFile>
    <SelfContained>true</SelfContained>
    <PublishTrimmed>false</PublishTrimmed>
    <!-- Set PublishTrimmed=true only after verifying no reflection-based
         code breaks. Game clients with plugin systems or dynamic loading
         should leave this false. -->
  </PropertyGroup>

  <!-- Reference all domain layers.
       The app project is the ONLY place where all layers converge. -->
  <ItemGroup>
    <ProjectReference Include="..\..\Core\<APPNAME>.Core\<APPNAME>.Core.csproj" />
    <ProjectReference Include="..\..\Data\<APPNAME>.Data\<APPNAME>.Data.csproj" />
    <ProjectReference Include="..\..\Data\<APPNAME>.Data.Wz\<APPNAME>.Data.Wz.csproj" />
    <ProjectReference Include="..\..\Crypto\<APPNAME>.Crypto\<APPNAME>.Crypto.csproj" />
    <ProjectReference Include="..\..\Network\<APPNAME>.Network\<APPNAME>.Network.csproj" />
    <ProjectReference Include="..\..\Network\<APPNAME>.Network.Protocol\<APPNAME>.Network.Protocol.csproj" />
    <ProjectReference Include="..\..\Game\<APPNAME>.Game\<APPNAME>.Game.csproj" />
    <ProjectReference Include="..\..\Game\<APPNAME>.Game.Combat\<APPNAME>.Game.Combat.csproj" />
    <ProjectReference Include="..\..\Game\<APPNAME>.Game.Inventory\<APPNAME>.Game.Inventory.csproj" />
    <ProjectReference Include="..\..\Game\<APPNAME>.Game.Quest\<APPNAME>.Game.Quest.csproj" />
    <ProjectReference Include="..\..\Game\<APPNAME>.Game.Field\<APPNAME>.Game.Field.csproj" />
    <ProjectReference Include="..\..\Rendering\<APPNAME>.Rendering\<APPNAME>.Rendering.csproj" />
    <ProjectReference Include="..\..\Rendering\<APPNAME>.Rendering.Sprites\<APPNAME>.Rendering.Sprites.csproj" />
    <ProjectReference Include="..\..\UI\<APPNAME>.UI\<APPNAME>.UI.csproj" />
    <ProjectReference Include="..\..\Audio\<APPNAME>.Audio\<APPNAME>.Audio.csproj" />
  </ItemGroup>

  <!-- Host builder + configuration -->
  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Hosting" />
    <PackageReference Include="Serilog" />
    <PackageReference Include="Serilog.Sinks.Console" />
    <PackageReference Include="Serilog.Extensions.Logging" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" />
  </ItemGroup>

  <!-- Copy appsettings.json to output -->
  <ItemGroup>
    <Content Include="appsettings.json" CopyToOutputDirectory="PreserveNewest" />
  </ItemGroup>

</Project>
```

### 6.2 `src/App/<APPNAME>/Program.cs`

The entry point wires up DI, logging, and starts the application:

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Serilog;

namespace <ROOTNS>;

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
            Log.Information("<APPNAME> starting...");

            var host = Host.CreateDefaultBuilder(args)
                .UseSerilog()
                .ConfigureServices((context, services) =>
                {
                    // Register domain services here:
                    // services.AddSingleton<IDataProvider, WzDataProvider>();
                    // services.AddSingleton<IRenderer, SomeRenderer>();
                    // services.AddHostedService<GameLoop>();
                })
                .Build();

            await host.RunAsync();
            return 0;
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "<APPNAME> terminated unexpectedly");
            return 1;
        }
        finally
        {
            await Log.CloseAndFlushAsync();
        }
    }
}
```

### 6.3 `src/App/<APPNAME>/appsettings.json`

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

### 6.4 Native Asset Copying (if `NEEDS_NATIVE=true`)

If the application ships native `.dll`/`.so` files alongside the managed binary, add a native asset project or use build targets to copy them:

**Option A: Directly in the app `.csproj`**:

```xml
<!-- Copy native dependencies to output directory -->
<ItemGroup>
  <None Include="runtimes\win-x64\native\*.dll" CopyToOutputDirectory="PreserveNewest" Link="%(Filename)%(Extension)" />
  <None Include="runtimes\linux-x64\native\*.so" CopyToOutputDirectory="PreserveNewest" Link="%(Filename)%(Extension)" />
</ItemGroup>
```

**Option B: A dedicated `<APPNAME>.Native` project** (when native binaries are large or platform-specific):

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <IncludeBuildOutput>false</IncludeBuildOutput>
  </PropertyGroup>
  <ItemGroup>
    <Content Include="runtimes\**\*" CopyToOutputDirectory="PreserveNewest" Link="%(RecursiveDir)%(Filename)%(Extension)" />
  </ItemGroup>
</Project>
```

**Agent note**: Only create the native asset structure if `NEEDS_NATIVE=true`. Most game clients do need this for graphics/audio backends.

---

## Phase 7: Test Projects

### 7.1 Test Project Template

Each layer gets a corresponding test project under `src/Tests/`. The template is minimal because `src/Tests/Directory.Build.props` provides `TUnit`, `NSubstitute`, and test configuration:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.Core.Tests</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\Core\<APPNAME>.Core\<APPNAME>.Core.csproj" />
  </ItemGroup>

</Project>
```

**That's 11 lines.** Every test project follows this exact pattern — only the `RootNamespace` and `ProjectReference` change.

### 7.2 Starter Test File

Every test project gets one real test. Example for `<APPNAME>.Core.Tests`:

```csharp
using <ROOTNS>.Core.Primitives;

namespace <ROOTNS>.Core.Tests;

public class FieldIdTests
{
    [Test]
    public async Task None_HasValueZero()
    {
        await Assert.That(FieldId.None.Value).IsEqualTo(0);
    }

    [Test]
    public async Task FieldId_EqualityByValue()
    {
        var a = new FieldId(100000000);
        var b = new FieldId(100000000);
        await Assert.That(b).IsEqualTo(a);
    }
}
```

### 7.3 Integration Test Project

Cross-layer integration tests live in `<APPNAME>.Integration.Tests`:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.Integration.Tests</RootNamespace>
  </PropertyGroup>

  <!-- Integration tests can reference multiple layers -->
  <ItemGroup>
    <ProjectReference Include="..\..\Core\<APPNAME>.Core\<APPNAME>.Core.csproj" />
    <ProjectReference Include="..\..\Crypto\<APPNAME>.Crypto\<APPNAME>.Crypto.csproj" />
    <ProjectReference Include="..\..\Network\<APPNAME>.Network\<APPNAME>.Network.csproj" />
    <ProjectReference Include="..\..\Network\<APPNAME>.Network.Protocol\<APPNAME>.Network.Protocol.csproj" />
  </ItemGroup>

</Project>
```

Starter test:

```csharp
namespace <ROOTNS>.Integration.Tests;

public class StartupTests
{
    [Test]
    public async Task AllAssemblies_Load()
    {
        // Proves the full dependency chain resolves at runtime.
        // If any ProjectReference is broken, this test fails.
        var coreAssembly = typeof(<ROOTNS>.Core.Primitives.FieldId).Assembly;
        await Assert.That(coreAssembly).IsNotNull();

        var cryptoAssembly = typeof(<ROOTNS>.Crypto.MapleCrc).Assembly;
        await Assert.That(cryptoAssembly).IsNotNull();
    }
}
```

### 7.4 Which Layers Get Test Projects?

| Layer | Test project? | Rationale |
|---|---|---|
| Core | Yes, always | Foundation — must be rock solid |
| Data | Yes | Data parsing correctness is critical |
| Crypto | Yes, always | Crypto bugs are security vulnerabilities |
| Network | Yes | Protocol correctness |
| Game (each sub) | Yes per subsystem | Business logic — highest bug density |
| Rendering | Optional | Hard to unit test graphics; prefer visual/manual testing |
| UI | Optional | Same as Rendering |
| Audio | Optional | Same |
| App | No unit tests | Tested via Integration.Tests |
| Benchmarks | No | Benchmarks ARE the test |

**Rule**: Create a test project for every layer that has **deterministic, assertable behavior**. Skip layers that are primarily side-effect-driven (rendering, audio) unless you have a headless test harness.

---

## Phase 8: Benchmarks (Optional)

If the user wants performance benchmarks, follow this simplified pattern. Unlike the library guide, there are no comparison benchmarks (you're not competing with another game client on NuGet):

### 8.1 `src/Benchmarks/<APPNAME>.Benchmarks/<APPNAME>.Benchmarks.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <RootNamespace><ROOTNS>.Benchmarks</RootNamespace>
    <!-- OutputType, IsPackable, BDN reference come from src/Benchmarks/Directory.Build.props -->
  </PropertyGroup>

  <ItemGroup>
    <!-- Reference the specific layers you want to benchmark -->
    <ProjectReference Include="..\..\Core\<APPNAME>.Core\<APPNAME>.Core.csproj" />
    <ProjectReference Include="..\..\Crypto\<APPNAME>.Crypto\<APPNAME>.Crypto.csproj" />
    <ProjectReference Include="..\..\Network\<APPNAME>.Network\<APPNAME>.Network.csproj" />
  </ItemGroup>

</Project>
```

### 8.2 `src/Benchmarks/<APPNAME>.Benchmarks/Program.cs`

```csharp
using BenchmarkDotNet.Running;

BenchmarkSwitcher.FromAssembly(typeof(Program).Assembly).Run(args);
```

### 8.3 Starter Benchmark

```csharp
using BenchmarkDotNet.Attributes;
using <ROOTNS>.Core.Primitives;

namespace <ROOTNS>.Benchmarks;

[MemoryDiagnoser]
public class FieldIdBench
{
    [Benchmark]
    public FieldId Create() => new(100000000);

    [Benchmark]
    public string ToStringBench() => new FieldId(100000000).ToString();
}
```

---

## Phase 9: `Build.cs` — Task Runner

The same file-based app pattern from the library guide, but with application-specific commands:

### `Build.cs` (repo root)

```csharp
#!/usr/bin/env dotnet
// Task runner for <APPNAME>
// Usage: dotnet Build.cs <command> [args]

#:property PublishAot=false

using System.Diagnostics;
using System.Runtime.CompilerServices;
using System.Runtime.InteropServices;

var repoRoot = RepoRoot();
var cmd = args.Length > 0 ? args[0] : "help";

return cmd switch
{
    "build" => Build(args),
    "test" => Test(args),
    "run" => RunApp(args),
    "publish" => Publish(args),
    "bench" => Bench(args),
    "format" => Format(args),
    "clean" => Clean(),
    "prettier" => Prettier(),
    "rename" => Rename(args),
    "help" or _ when cmd == "help" => Help(),
    _ => Help(),
};

int Build(string[] a)
{
    var config = a.Length > 1 ? a[1] : "Debug";
    Run("dotnet", $"build -c {config}", repoRoot);
    return 0;
}

int Test(string[] a)
{
    var config = a.Length > 1 ? a[1] : "Debug";
    var filter = a.Length > 2 ? $" --filter \"{a[2]}\"" : "";
    Run("dotnet", $"test -c {config} --verbosity normal{filter}", repoRoot);
    return 0;
}

int RunApp(string[] a)
{
    var appProject = Path.Combine("src", "App", "<APPNAME>", "<APPNAME>.csproj");
    var extraArgs = a.Length > 1 ? " -- " + string.Join(' ', a[1..]) : "";
    Run("dotnet", $"run --project {appProject}{extraArgs}", repoRoot);
    return 0;
}

int Publish(string[] a)
{
    var rid = a.Length > 1 ? a[1] : RuntimeInformation.RuntimeIdentifier;
    var config = a.Length > 2 ? a[2] : "Release";
    var appProject = Path.Combine("src", "App", "<APPNAME>", "<APPNAME>.csproj");
    var publishDir = Path.Combine(repoRoot, "publish", rid);

    Run("dotnet", $"publish {appProject} -c {config} -r {rid} -o \"{publishDir}\"", repoRoot);

    Console.WriteLine($"Published to: {publishDir}");
    return 0;
}

int Bench(string[] a)
{
    var filter = a.Length > 1 ? a[1] : "*";
    Run("dotnet",
        $"run --project src/Benchmarks/<APPNAME>.Benchmarks/<APPNAME>.Benchmarks.csproj"
        + $" -c Release -- --filter \"{filter}\" --join",
        repoRoot);
    return 0;
}

int Format(string[] a)
{
    // CSharpier: opinionated whitespace/formatting
    // dotnet format: code style analyzers (naming, usings, etc.)
    var verify = a.Length > 1 && a[1] == "check";
    Run("dotnet", "tool restore", repoRoot);
    Run("dotnet", verify ? "csharpier check ." : "csharpier format .", repoRoot);
    Run("dotnet", verify ? "format --verify-no-changes" : "format", repoRoot);
    return 0;
}

int Clean()
{
    var artifacts = Path.Combine(repoRoot, "artifacts");
    if (Directory.Exists(artifacts))
    {
        Directory.Delete(artifacts, recursive: true);
        Console.WriteLine("Cleaned artifacts/");
    }

    var publish = Path.Combine(repoRoot, "publish");
    if (Directory.Exists(publish))
    {
        Directory.Delete(publish, recursive: true);
        Console.WriteLine("Cleaned publish/");
    }

    Run("dotnet", "clean", repoRoot);
    return 0;
}

int Prettier()
{
    Run("npx", "prettier --write \"**/*.{yml,yaml,json,md}\" --ignore-path .gitignore", repoRoot);
    return 0;
}

int Rename(string[] a)
{
    if (a.Length < 3)
    {
        Console.Error.WriteLine("Usage: dotnet Build.cs rename <OldName> <NewName>");
        return 1;
    }
    RenameAll(repoRoot, a[1], a[2]);
    return 0;
}

int Help()
{
    Console.WriteLine("""
        Usage: dotnet Build.cs <command> [args]

        Commands:
          build [config]              Build solution (default: Debug)
          test [config] [filter]      Run tests (default: Debug, all tests)
          run [-- app-args]           Run the application
          publish [rid] [config]      Publish self-contained app (default: current RID, Release)
          bench [filter]              Run BenchmarkDotNet benchmarks
          format [check]              Format C# (CSharpier) + code style (dotnet format); 'check' verifies only
          clean                       Delete artifacts/ and publish/ directories
          prettier                    Format YAML/Markdown/JSON via Prettier (requires Node)
          rename <Old> <New>          Rename template throughout repository
          help                        Show this help
        """);
    return 0;
}

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

    Console.WriteLine($"Renamed '{oldName}' → '{newName}' throughout repository.");
}

static string RepoRoot([CallerFilePath] string path = "") =>
    Path.GetDirectoryName(path)!;
```

---

## Phase 10: Solution Organization

### 10.1 `<APPNAME>.slnx` — With Solution Folders

A 100-project solution is unnavigable without solution folders. `.slnx` supports them natively:

```xml
<Solution>

  <!-- ═══════════ Core ═══════════ -->
  <Folder Name="/Core/">
    <Project Path="src/Core/<APPNAME>.Core/<APPNAME>.Core.csproj" />
    <!-- <Project Path="src/Core/<APPNAME>.Core.Collections/<APPNAME>.Core.Collections.csproj" /> -->
  </Folder>

  <!-- ═══════════ Data ═══════════ -->
  <Folder Name="/Data/">
    <Project Path="src/Data/<APPNAME>.Data/<APPNAME>.Data.csproj" />
    <Project Path="src/Data/<APPNAME>.Data.Wz/<APPNAME>.Data.Wz.csproj" />
  </Folder>

  <!-- ═══════════ Crypto ═══════════ -->
  <Folder Name="/Crypto/">
    <Project Path="src/Crypto/<APPNAME>.Crypto/<APPNAME>.Crypto.csproj" />
  </Folder>

  <!-- ═══════════ Network ═══════════ -->
  <Folder Name="/Network/">
    <Project Path="src/Network/<APPNAME>.Network/<APPNAME>.Network.csproj" />
    <Project Path="src/Network/<APPNAME>.Network.Protocol/<APPNAME>.Network.Protocol.csproj" />
  </Folder>

  <!-- ═══════════ Game ═══════════ -->
  <Folder Name="/Game/">
    <Project Path="src/Game/<APPNAME>.Game/<APPNAME>.Game.csproj" />
    <Project Path="src/Game/<APPNAME>.Game.Combat/<APPNAME>.Game.Combat.csproj" />
    <Project Path="src/Game/<APPNAME>.Game.Inventory/<APPNAME>.Game.Inventory.csproj" />
    <Project Path="src/Game/<APPNAME>.Game.Quest/<APPNAME>.Game.Quest.csproj" />
    <Project Path="src/Game/<APPNAME>.Game.Field/<APPNAME>.Game.Field.csproj" />
  </Folder>

  <!-- ═══════════ Rendering ═══════════ -->
  <Folder Name="/Rendering/">
    <Project Path="src/Rendering/<APPNAME>.Rendering/<APPNAME>.Rendering.csproj" />
    <Project Path="src/Rendering/<APPNAME>.Rendering.Sprites/<APPNAME>.Rendering.Sprites.csproj" />
  </Folder>

  <!-- ═══════════ UI ═══════════ -->
  <Folder Name="/UI/">
    <Project Path="src/UI/<APPNAME>.UI/<APPNAME>.UI.csproj" />
  </Folder>

  <!-- ═══════════ Audio ═══════════ -->
  <Folder Name="/Audio/">
    <Project Path="src/Audio/<APPNAME>.Audio/<APPNAME>.Audio.csproj" />
  </Folder>

  <!-- ═══════════ App ═══════════ -->
  <Folder Name="/App/">
    <Project Path="src/App/<APPNAME>/<APPNAME>.csproj" />
  </Folder>

  <!-- ═══════════ Tests ═══════════ -->
  <Folder Name="/Tests/">
    <Project Path="src/Tests/<APPNAME>.Core.Tests/<APPNAME>.Core.Tests.csproj" />
    <Project Path="src/Tests/<APPNAME>.Crypto.Tests/<APPNAME>.Crypto.Tests.csproj" />
    <Project Path="src/Tests/<APPNAME>.Network.Tests/<APPNAME>.Network.Tests.csproj" />
    <Project Path="src/Tests/<APPNAME>.Game.Combat.Tests/<APPNAME>.Game.Combat.Tests.csproj" />
    <Project Path="src/Tests/<APPNAME>.Integration.Tests/<APPNAME>.Integration.Tests.csproj" />
  </Folder>

  <!-- ═══════════ Benchmarks ═══════════ -->
  <Folder Name="/Benchmarks/">
    <Project Path="src/Benchmarks/<APPNAME>.Benchmarks/<APPNAME>.Benchmarks.csproj" />
  </Folder>

</Solution>
```

**Agent note**: The `.slnx` must list **every** `.csproj` in the solution. When adding new projects, add them here too. The `.slnx` solution-folder structure should mirror the `src/` folder structure for consistency.

### 10.2 Solution Filters (`.slnf`)

Solution filters let developers load a subset of the solution. This is essential when the full solution takes 30+ seconds to load:

**`Core.slnf`** — Just the core layer and its tests:
```json
{
  "solution": {
    "path": "<APPNAME>.slnx",
    "projects": [
      "src/Core/<APPNAME>.Core/<APPNAME>.Core.csproj",
      "src/Tests/<APPNAME>.Core.Tests/<APPNAME>.Core.Tests.csproj"
    ]
  }
}
```

**`Network.slnf`** — Network stack with transitive dependencies:
```json
{
  "solution": {
    "path": "<APPNAME>.slnx",
    "projects": [
      "src/Core/<APPNAME>.Core/<APPNAME>.Core.csproj",
      "src/Crypto/<APPNAME>.Crypto/<APPNAME>.Crypto.csproj",
      "src/Network/<APPNAME>.Network/<APPNAME>.Network.csproj",
      "src/Network/<APPNAME>.Network.Protocol/<APPNAME>.Network.Protocol.csproj",
      "src/Tests/<APPNAME>.Core.Tests/<APPNAME>.Core.Tests.csproj",
      "src/Tests/<APPNAME>.Crypto.Tests/<APPNAME>.Crypto.Tests.csproj",
      "src/Tests/<APPNAME>.Network.Tests/<APPNAME>.Network.Tests.csproj"
    ]
  }
}
```

**`Game.slnf`** — Game logic with all transitive dependencies:
```json
{
  "solution": {
    "path": "<APPNAME>.slnx",
    "projects": [
      "src/Core/<APPNAME>.Core/<APPNAME>.Core.csproj",
      "src/Data/<APPNAME>.Data/<APPNAME>.Data.csproj",
      "src/Data/<APPNAME>.Data.Wz/<APPNAME>.Data.Wz.csproj",
      "src/Crypto/<APPNAME>.Crypto/<APPNAME>.Crypto.csproj",
      "src/Network/<APPNAME>.Network/<APPNAME>.Network.csproj",
      "src/Network/<APPNAME>.Network.Protocol/<APPNAME>.Network.Protocol.csproj",
      "src/Game/<APPNAME>.Game/<APPNAME>.Game.csproj",
      "src/Game/<APPNAME>.Game.Combat/<APPNAME>.Game.Combat.csproj",
      "src/Game/<APPNAME>.Game.Inventory/<APPNAME>.Game.Inventory.csproj",
      "src/Game/<APPNAME>.Game.Quest/<APPNAME>.Game.Quest.csproj",
      "src/Game/<APPNAME>.Game.Field/<APPNAME>.Game.Field.csproj",
      "src/Tests/<APPNAME>.Game.Combat.Tests/<APPNAME>.Game.Combat.Tests.csproj"
    ]
  }
}
```

**Rule**: Every `.slnf` must include *all transitive `ProjectReference` dependencies* of the projects it lists. If `Network` depends on `Core` and `Crypto`, the Network filter must include all three. Visual Studio will error on load if a referenced project is missing from the filter.

### 10.3 Project Scaling Guidance

| Project count | Strategy |
|---|---|
| 1–15 | Open the `.slnx` directly. No filters needed. |
| 15–40 | Create 2–3 `.slnf` files for the most-edited layers. |
| 40–80 | Create one `.slnf` per major domain layer. Add `All.slnf` as a convenience alias. |
| 80+ | Same as above, plus consider splitting into multiple `.slnx` solutions with shared project references if build times exceed 60 seconds. |

---

## Phase 11: CI/CD Pipeline

### 11.1 `.github/workflows/dotnet.yml`

The CI pipeline builds and tests the application. No NuGet publishing — instead it produces a build artifact (the published app):

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
        description: "Version tag to create (e.g. 1.2.3)"
        required: false

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
        uses: step-security/harden-runner@<PINNED-SHA>
        with:
          egress-policy: audit

      - uses: actions/checkout@<PINNED-SHA>

      - name: Setup .NET (global.json)
        uses: actions/setup-dotnet@<PINNED-SHA>
        with:
          global-json-file: global.json

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build -c ${{ matrix.configuration }} --no-restore

      - name: Test
        run: dotnet test -c ${{ matrix.configuration }} --no-build --verbosity normal

  format:
    runs-on: ubuntu-latest
    steps:
      - name: Harden Runner
        uses: step-security/harden-runner@<PINNED-SHA>
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
      - name: Format verify no changes
        run: dotnet format --verify-no-changes

  publish:
    needs: [build-and-test, format]
    if: github.ref == 'refs/heads/main' || github.event.inputs.version != ''
    strategy:
      matrix:
        include:
          - os: windows-latest
            rid: win-x64
          - os: ubuntu-latest
            rid: linux-x64
          # Add more RID/OS combos as needed:
          # - os: macos-latest
          #   rid: osx-x64

    runs-on: ${{ matrix.os }}

    steps:
      - name: Harden Runner
        uses: step-security/harden-runner@<PINNED-SHA>
        with:
          egress-policy: audit

      - uses: actions/checkout@<PINNED-SHA>

      - name: Setup .NET
        uses: actions/setup-dotnet@<PINNED-SHA>
        with:
          global-json-file: global.json

      - name: Set version
        id: version
        shell: bash
        run: |
          if [ -n "${{ github.event.inputs.version }}" ]; then
            echo "version=${{ github.event.inputs.version }}" >> $GITHUB_OUTPUT
          else
            echo "version=0.0.0-ci.${{ github.run_number }}" >> $GITHUB_OUTPUT
          fi

      - name: Publish
        run: >
          dotnet publish src/App/<APPNAME>/<APPNAME>.csproj
          -c Release
          -r ${{ matrix.rid }}
          -p:Version=${{ steps.version.outputs.version }}
          -o publish/${{ matrix.rid }}

      - name: Upload artifact
        uses: actions/upload-artifact@<PINNED-SHA>
        with:
          name: <APPNAME>-${{ matrix.rid }}
          if-no-files-found: error
          retention-days: 30
          path: publish/${{ matrix.rid }}/

  create-release:
    needs: [publish]
    if: github.event.inputs.version != ''
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Harden Runner
        uses: step-security/harden-runner@<PINNED-SHA>
        with:
          egress-policy: audit

      - uses: actions/checkout@<PINNED-SHA>

      - name: Download all artifacts
        uses: actions/download-artifact@<PINNED-SHA>
        with:
          path: release-artifacts/

      - name: Create GitHub Release
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          cd release-artifacts
          for dir in */; do
            (cd "$dir" && zip -r "../${dir%/}.zip" .)
          done
          cd ..
          gh release create "v${{ github.event.inputs.version }}" \
            release-artifacts/*.zip \
            --title "v${{ github.event.inputs.version }}" \
            --generate-notes
```

**Key differences from the library CI**:
- No NuGet pack/push — replaced by `dotnet publish` + artifact upload
- No Codecov — optional for apps
- OS matrix is smaller (linux + windows, not 5 OSes) — expand if you need macOS/ARM
- Release creates a GitHub Release with zipped binaries, not a NuGet push
- Version is driven by `workflow_dispatch` input or CI run number, not MinVer

### 11.2 `.github/dependabot.yml`

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

## Phase 12: Publish Profiles (Optional)

For complex publish scenarios, create publish profiles:

### `src/App/<APPNAME>/Properties/PublishProfiles/win-x64.pubxml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project>
  <PropertyGroup>
    <Configuration>Release</Configuration>
    <Platform>Any CPU</Platform>
    <PublishDir>..\..\..\..\publish\win-x64</PublishDir>
    <RuntimeIdentifier>win-x64</RuntimeIdentifier>
    <SelfContained>true</SelfContained>
    <PublishSingleFile>true</PublishSingleFile>
    <PublishTrimmed>false</PublishTrimmed>
    <IncludeNativeLibrariesForSelfExtract>true</IncludeNativeLibrariesForSelfExtract>
  </PropertyGroup>
</Project>
```

Create one per target platform. Invoke via:
```shell
dotnet publish src/App/<APPNAME>/<APPNAME>.csproj /p:PublishProfile=win-x64
```

---

## Phase 13: Validation Checklist

Before declaring the scaffold complete, verify:

### Build and Test
- [ ] `dotnet build -c Debug` passes with zero warnings
- [ ] `dotnet build -c Release` passes with zero warnings (CSharpier.MsBuild checks formatting automatically)
- [ ] `dotnet test -c Debug` passes (all test projects)
- [ ] `dotnet test -c Release` passes
- [ ] `dotnet tool restore` succeeds (installs CSharpier)
- [ ] `dotnet csharpier check .` passes (opinionated formatting)
- [ ] `dotnet format --verify-no-changes` passes (code style analyzers)
- [ ] `dotnet Build.cs format check` passes (runs both CSharpier + dotnet format)

### Application
- [ ] `dotnet run --project src/App/<APPNAME>/<APPNAME>.csproj` starts and exits cleanly
- [ ] `dotnet Build.cs run` starts and exits cleanly
- [ ] `dotnet Build.cs publish win-x64` produces output in `publish/win-x64/`
- [ ] The published executable runs standalone

### Solution Structure
- [ ] `.slnx` lists every `.csproj` in the repository
- [ ] `.slnx` solution folders match the `src/` directory layout
- [ ] No layer violates the dependency rules (e.g., Core references nothing, Rendering doesn't reference Network)
- [ ] Every library `.csproj` has exactly the inherited properties — no duplicated `TargetFramework`, `Nullable`, etc.
- [ ] `Directory.Packages.props` has no placeholder versions — all resolved to real values

### CI and Repository Health
- [ ] All action SHAs in `.github/workflows/dotnet.yml` are 40-char hashes (or have `# TODO:` comments)
- [ ] `.github/dependabot.yml` exists
- [ ] `SECURITY.md` exists
- [ ] `CONTRIBUTING.md` exists
- [ ] `CHANGELOG.md` exists
- [ ] `.config/dotnet-tools.json` exists
- [ ] `.github/ISSUE_TEMPLATE/` and `PULL_REQUEST_TEMPLATE.md` exist

### Task Runner
- [ ] `dotnet Build.cs help` prints the command list without errors
- [ ] `dotnet Build.cs rename <APPNAME> TestRename` runs without errors; revert with `dotnet Build.cs rename TestRename <APPNAME>`

### Solution Filters (if created)
- [ ] Each `.slnf` opens in Visual Studio / Rider without errors
- [ ] Each `.slnf` includes all transitive `ProjectReference` dependencies

---

## Complete File Manifest

After all phases, the repository contains these files (substitute `<APPNAME>` and adjust layers to match user input):

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
├── src/
│   ├── Directory.Build.props                           ← Tier 2
│   ├── Directory.Build.targets
│   │
│   ├── Core/
│   │   └── <APPNAME>.Core/
│   │       ├── <APPNAME>.Core.csproj
│   │       └── Primitives/
│   │           └── FieldId.cs
│   ├── Data/
│   │   ├── <APPNAME>.Data/
│   │   │   ├── <APPNAME>.Data.csproj
│   │   │   └── IDataProvider.cs
│   │   └── <APPNAME>.Data.Wz/
│   │       ├── <APPNAME>.Data.Wz.csproj
│   │       └── WzReader.cs
│   ├── Crypto/
│   │   └── <APPNAME>.Crypto/
│   │       ├── <APPNAME>.Crypto.csproj
│   │       └── MapleCrc.cs
│   ├── Network/
│   │   ├── <APPNAME>.Network/
│   │   │   ├── <APPNAME>.Network.csproj
│   │   │   └── PacketReader.cs
│   │   └── <APPNAME>.Network.Protocol/
│   │       ├── <APPNAME>.Network.Protocol.csproj
│   │       └── Opcodes/
│   │           └── RecvOps.cs
│   ├── Game/
│   │   ├── <APPNAME>.Game/
│   │   │   ├── <APPNAME>.Game.csproj
│   │   │   └── GameState.cs
│   │   ├── <APPNAME>.Game.Combat/
│   │   │   ├── <APPNAME>.Game.Combat.csproj
│   │   │   └── DamageCalculator.cs
│   │   ├── <APPNAME>.Game.Inventory/
│   │   │   ├── <APPNAME>.Game.Inventory.csproj
│   │   │   └── InventoryManager.cs
│   │   ├── <APPNAME>.Game.Quest/
│   │   │   ├── <APPNAME>.Game.Quest.csproj
│   │   │   └── QuestManager.cs
│   │   └── <APPNAME>.Game.Field/
│   │       ├── <APPNAME>.Game.Field.csproj
│   │       └── FieldManager.cs
│   ├── Rendering/
│   │   ├── <APPNAME>.Rendering/
│   │   │   ├── <APPNAME>.Rendering.csproj
│   │   │   └── IRenderer.cs
│   │   └── <APPNAME>.Rendering.Sprites/
│   │       ├── <APPNAME>.Rendering.Sprites.csproj
│   │       └── SpriteSheet.cs
│   ├── UI/
│   │   └── <APPNAME>.UI/
│   │       ├── <APPNAME>.UI.csproj
│   │       └── UIManager.cs
│   ├── Audio/
│   │   └── <APPNAME>.Audio/
│   │       ├── <APPNAME>.Audio.csproj
│   │       └── AudioManager.cs
│   │
│   ├── App/
│   │   ├── Directory.Build.props                       ← Tier 3 (app)
│   │   └── <APPNAME>/
│   │       ├── <APPNAME>.csproj
│   │       ├── Program.cs
│   │       └── appsettings.json
│   │
│   ├── Tests/
│   │   ├── Directory.Build.props                       ← Tier 3 (tests)
│   │   ├── <APPNAME>.Core.Tests/
│   │   │   ├── <APPNAME>.Core.Tests.csproj
│   │   │   └── FieldIdTests.cs
│   │   ├── <APPNAME>.Crypto.Tests/
│   │   │   ├── <APPNAME>.Crypto.Tests.csproj
│   │   │   └── MapleCrcTests.cs
│   │   ├── <APPNAME>.Network.Tests/
│   │   │   ├── <APPNAME>.Network.Tests.csproj
│   │   │   └── PacketReaderTests.cs
│   │   ├── <APPNAME>.Game.Combat.Tests/
│   │   │   ├── <APPNAME>.Game.Combat.Tests.csproj
│   │   │   └── DamageCalculatorTests.cs
│   │   └── <APPNAME>.Integration.Tests/
│   │       ├── <APPNAME>.Integration.Tests.csproj
│   │       └── StartupTests.cs
│   │
│   └── Benchmarks/
│       ├── Directory.Build.props                       ← Tier 3 (benchmarks)
│       └── <APPNAME>.Benchmarks/
│           ├── <APPNAME>.Benchmarks.csproj
│           ├── Program.cs
│           └── FieldIdBench.cs
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
├── Core.slnf                                           ← Solution filter (Phase 10.2)
├── Network.slnf                                        ← Solution filter (Phase 10.2)
├── Game.slnf                                           ← Solution filter (Phase 10.2)
├── Directory.Build.props                               ← Tier 1 (root)
├── Directory.Packages.props
├── global.json
├── LICENSE
├── nuget.config
├── README.md
├── SECURITY.md
└── <APPNAME>.slnx
```

**Total**: ~76 files across ~40 directories for the game client reference architecture (9 layers, 16 source projects, 5 test projects, 1 benchmark project, 1 app project). Scales to 100+ projects by adding more modules within existing layers.

---

## Decision Table: Common Customizations

| Scenario | Change |
|---|---|
| No benchmarks needed | Remove `src/Benchmarks/` entirely and its `.slnx` folder entry |
| Need WPF or WinForms | Change `APP_SDK` to `Microsoft.NET.Sdk.WindowsDesktop`; add `<UseWPF>true</UseWPF>` or `<UseWindowsForms>true</UseWindowsForms>` to app `Directory.Build.props` |
| Multiple executables (client + server + tools) | Add more projects under `src/App/`: `<APPNAME>.Server/`, `<APPNAME>.Tools.PacketSniffer/` — each with its own `.csproj` |
| Shared code between multiple apps | Extract into a `src/Shared/<APPNAME>.Shared/` project referenced by both apps |
| Native interop (P/Invoke) | Add `<APPNAME>.Native/` project under the relevant layer with `NativeLibrary.Load` and runtime-specific native binaries |
| Plugin/mod system | Add `<APPNAME>.Sdk/` project defining the plugin contract interfaces; this IS packable (`<IsPackable>true</IsPackable>`) and ships as a NuGet package for mod developers |
| Docker deployment | Add `Dockerfile` and `docker-compose.yml` at repo root; CI builds and pushes container images instead of zip artifacts |
| Team uses Rider | Replace `.slnx` with `.sln` via `dotnet new sln` + `dotnet sln add`. Keep solution folders by using `dotnet sln add --solution-folder`. Rider `.slnx` support is partial. |
| Need code coverage tracking | Re-add `codecov.yml` from library guide. No additional packages are needed — TUnit already bundles `Microsoft.Testing.Extensions.CodeCoverage` transitively. Do NOT use `coverlet.collector` — it is a VSTest data collector and [does not work with TUnit's MTP runner](https://tunit.dev/docs/extending/code-coverage/). In CI, use `dotnet test --coverage --coverage-output-format cobertura` instead of `--collect:"XPlat Code Coverage"`. **VS Code**: install [Coverage Gutters](https://marketplace.visualstudio.com/items?itemName=ryanluker.vscode-coverage-gutters) for inline coverage display. |
| Multi-targeting (net8.0 + net10.0) | Change `<TargetFramework>` to `<TargetFrameworks>` in root `Directory.Build.props`; be aware this doubles build time |
| Release versioning from CI | In CI workflow, pass `-p:Version=X.Y.Z -p:InformationalVersion=X.Y.Z+<commit-sha>` to build and publish steps |
| Need Git commit hash in app | Add `<SourceRevisionId>$(GITHUB_SHA)</SourceRevisionId>` to root `Directory.Build.props` under a `Condition="'$(GITHUB_ACTIONS)' == 'true'"` guard. Access at runtime via `Assembly.GetCustomAttribute<AssemblyInformationalVersionAttribute>()`. |
| CSharpier line width too narrow / too wide | Edit `.csharpierrc.json` `printWidth` value. 100 is CSharpier default, 120 is the scaffold default. Do not go below 80 or above 150. |
| Want to disable CSharpier build-time check temporarily | Set `<CSharpier_Check>false</CSharpier_Check>` in a `PropertyGroup` — do NOT remove the `CSharpier.MsBuild` package. Re-enable before merging. |

---

## Anti-Patterns to Avoid

1. **Do not flatten all projects into `src/`**. A flat directory with 100 `.csproj` files is unnavigable. Use layer folders.
2. **Do not duplicate build settings in every `.csproj`**. If you see `<TargetFramework>` or `<Nullable>` in a `.csproj`, it should be inherited from `Directory.Build.props`. The only exception is when a single project genuinely needs a different value.
3. **Do not skip Central Package Management**. At 10+ projects, version drift is inevitable without CPM. At 50+, it's guaranteed.
4. **Do not create circular layer dependencies**. If `Network` references `Game` and `Game` references `Network`, the architecture is broken — extract the shared types into `Core` or create a new shared layer.
5. **Do not put test projects inside source layer folders**. Keep all tests under `src/Tests/` so the test `Directory.Build.props` applies cleanly without complex Import paths.
6. **Do not create `.slnf` files that are missing transitive dependencies**. Visual Studio will fail to load the filter and developers will stop using filters entirely.
7. **Do not add MinVer to an application project**. MinVer is designed for library semver on NuGet. Applications use explicit version properties injected at build time.
8. **Do not use `Directory.Build.props` Import chains deeper than 3 tiers**. Root → `src/` → layer is the maximum. Deeper chains become impossible to debug ("where is this property coming from?").
9. **Do not pack internal libraries as NuGet packages** unless you are explicitly building a plugin SDK. Internal libraries are referenced via `<ProjectReference>`, not `<PackageReference>`.
10. **Do not skip the validation checklist**. A scaffold that doesn't build is worse than no scaffold — it teaches the team that "it's fine to commit broken code."
11. **Do not skip CSharpier or rely solely on `.editorconfig` for formatting**. `.editorconfig` formatting rules are advisory in most IDEs — only `dotnet format` enforces them, and it is lenient about whitespace. CSharpier is deterministic: two developers formatting the same file will always get identical output. Without it, formatting "wars" in PRs are inevitable.
12. **Do not remove `CSharpier.MsBuild` to "speed up builds"**. The overhead is negligible (<1s per project) and the alternative — discovering formatting violations only in CI after a 10-minute round trip — is far more expensive.
