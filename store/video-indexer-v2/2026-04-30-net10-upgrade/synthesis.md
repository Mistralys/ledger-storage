# Project Synthesis — .NET 10 Upgrade

**Plan:** `2026-04-30-net10-upgrade`
**Status:** COMPLETE
**Date:** 2026-05-05
**Prepared by:** Head of Operations (Synthesis)

---

## Executive Summary

The `video-indexer-v2` solution has been fully migrated from .NET 8 to .NET 10. The work covered five areas: SDK/TFM centralization (WP-001), the .NET 10 package band upgrade (WP-002), test-project configuration unification (WP-003), end-to-end clean-build and smoke validation (WP-004), and forward-facing documentation cleanup (WP-005).

All five work packages completed with every pipeline stage at PASS. The solution builds with zero warnings and zero errors under `TreatWarningsAsErrors=true` / `WarningLevel=9999`, all 151 automated tests pass, and the Avalonia application launches successfully on .NET 10.0.6. Two latent defects unrelated to the upgrade itself were discovered and fixed as part of this cycle.

---

## Metrics

| Metric | Value |
|---|---|
| Work Packages | 5 of 5 COMPLETE |
| Pipeline stages executed | 16 (15 planned + 1 QA self-rework on WP-004) |
| Tests passed | 151 |
| Tests failed | 0 |
| Tests skipped (intentional live/integration) | 2 |
| Build warnings | 0 |
| Build errors | 0 |
| Smoke run | PASS — app reached initial window state on .NET 10.0.6 |

---

## What Was Built

### WP-001 — SDK / TFM Centralization

- `global.json` pinned to SDK `10.0.202` with `rollForward: latestFeature` (stops at 10.1).
- `TargetFramework=net10.0` centralized into `Directory.Build.props`; individual `<TargetFramework>` entries removed from all six `.csproj` files.
- `<RollForward>` shims removed from `VideoIndexer.Tests` and `VideoIndexer.App.Tests`.
- `NuGetAuditMode=direct` added to `Directory.Build.props` as a temporary debt shield for the transitive `Tmds.DBus.Protocol 0.20.0` vulnerability (GHSA-xrw6-gwf8-vvr9) via Avalonia 11.2.5 — documented in README with an explicit call-out.
- Reviewer Fix-Forward: `<IsTestProject>true</IsTestProject>` added to `VideoIndexer.Infrastructure.Tests.csproj` for tooling consistency.

**Files modified:** `global.json`, `Directory.Build.props`, all six `.csproj` files.

---

### WP-002 — .NET 10 Package Band Upgrade

- All `Microsoft.Extensions.*` packages in `VideoIndexer.Infrastructure` and `VideoIndexer.App` upgraded to `10.0.7`.
- `Serilog.Extensions.Hosting` upgraded to `10.0.0` in both projects (consistent, byte-identical).
- Framework-agnostic packages left untouched: Dapper, MySqlConnector, SharpCompress, Serilog.Sinks.*, CommunityToolkit.Mvvm, Avalonia* (all unchanged).
- Pre-existing version skew between the two csprojs (e.g., `Configuration.Json 8.0.0` vs `8.0.1`) resolved by alignment.
- `CHANGELOG.md` created at the repository root (none existed previously).

**Files modified:** `VideoIndexer.Infrastructure.csproj`, `VideoIndexer.App.csproj`, `CHANGELOG.md`.

---

### WP-003 — Test Project Configuration Unification

- Unified test framework packages across all three test projects: `Microsoft.NET.Test.Sdk 17.12.0`, `xunit 2.9.3`, `xunit.runner.visualstudio 2.8.2`, `FluentAssertions 6.12.2`.
- `TreatWarningsAsErrors=false` applied consistently to all three test projects.
- Serilog direct reference in `VideoIndexer.Infrastructure.Tests` upgraded to `4.3.0` to match the transitive version brought in by `Serilog.Extensions.Hosting 10.0.0`.
- Pre-existing CS8767 nullability mismatch in `SpdbConfigRepositoryTests.cs` fixed with `[AllowNull]` on the `ConnectionString` property.
- `VideoIndexer.Infrastructure.Tests` added to `VideoIndexer.sln` (it was silently excluded).
- Reviewer Fix-Forward: `<RootNamespace>VideoIndexer.Infrastructure.Tests</RootNamespace>` added for consistency.

**Files modified:** All three test `.csproj` files, `VideoIndexer.sln`, `SpdbConfigRepositoryTests.cs`, `CHANGELOG.md`.

---

### WP-004 — End-to-End Validation and Smoke Testing

- Clean-from-scratch build (after full `bin/`+`obj/` wipe): 0 warnings, 0 errors, all artifacts under `net10.0/` only.
- All three test suites validated independently (151 passed, 0 failed, 2 intentional skips).
- Discovered and fixed a **pre-existing DI registration defect** in `Program.cs` (see Critical Findings below).
- Smoke run: `VideoIndexer.App` launched on .NET 10.0.6, reached the `ProvisioningToolsView` initial window state, ran stably for 8+ seconds without exceptions.

**Files modified:** `src/VideoIndexer.App/Program.cs`.

---

### WP-005 — Forward-Facing Documentation Cleanup

- Repository-wide search for `.NET 8` / `net8.0` outside historical plan folders: zero matches remaining.
- `README.md` and `docs/projects/rebuild/rebuild.md` were already updated by a prior Release Engineer pipeline.
- `docs/projects/rebuild/milestones/m1-foundation.md`: updated three entries (`net8.0` → `net10.0`, removed now-moot .NET 8 runtime risk row).
- No source code touched.

**Files modified:** `docs/projects/rebuild/milestones/m1-foundation.md`.

---

## Critical Findings

### 1. Pre-existing DI Registration Defect — FIXED (HIGH)

**WP-004 QA discovered a latent crash on app startup.** `ViewLocator.Build()` calls `App.Services.GetRequiredService(viewType)` for all content views, but `Program.cs` only registered `MainWindow`. All five content views (`ProvisioningToolsView`, `DatabaseConnectorView`, `LogonView`, `PasswordSetupView`, `MainContentView`) were unregistered, causing a deterministic `InvalidOperationException` on every launch.

**Fix applied:** Five `AddTransient<>` factory registrations added to `Program.cs` using explicit `new XView()` factories (necessary because content view ViewModels are not DI services — they are ShellViewModel-owned). The application now starts cleanly.

**This defect was not introduced by this upgrade cycle.** It predated WP-001 and would have caused the app to crash on any .NET version.

---

### 2. VideoIndexer.Infrastructure.Tests Missing from Solution — FIXED (MEDIUM)

`VideoIndexer.Infrastructure.Tests.csproj` was never added to `VideoIndexer.sln`. It was silently excluded from all solution-level builds (`dotnet build` on the `.sln`) and from IDE test discovery. Fixed in WP-003. Root cause: likely an oversight at project creation time.

---

## Strategic Recommendations

### Avalonia Upgrade — Tmds.DBus.Protocol Vulnerability (HIGH)

`NuGetAuditMode=direct` in `Directory.Build.props` is a deliberate debt shield for `Tmds.DBus.Protocol 0.20.0` (GHSA-xrw6-gwf8-vvr9), a high-severity vulnerability brought in transitively via Avalonia 11.2.5. This suppression was intentionally deferred from this upgrade cycle.

**Recommendation:** Plan an Avalonia version upgrade. Once an Avalonia release resolves the transitive `Tmds.DBus.Protocol` dependency, remove the `NuGetAuditMode=direct` property from `Directory.Build.props` to re-enable full transitive audit mode. This is the correct long-term state for a security-conscious build.

### View Constructor Comments (LOW)

All five content view code-behinds carry the comment `// DI constructor — DataContext is injected at construction time.` on their second constructors. After the WP-004 fix, this comment is misleading: these constructors are not invoked during production startup (DataContext comes from Avalonia's `ContentPresenter` via the `CurrentViewModel` binding). Update the comments to clarify they are available for direct instantiation only (e.g., in test scenarios).

### ShellState Transition Coverage Gap (LOW)

The WP-004 smoke run validated only the initial `ProvisioningTools` shell state. The four remaining states (`Connecting`, `LoggingOn`, `SettingPassword`, `Ready`) were not exercised. Future QA or integration tests should cover all five `ShellState` transitions to confirm `ViewLocator` resolves each view correctly.

### NonDisposingConnectionWrapper — Extraction Candidate (LOW)

The `NonDisposingConnectionWrapper` test helper in `SpdbConfigRepositoryTests.cs` is well-implemented with a good XML doc comment. If the pattern is needed across additional test files, it should be extracted to a shared test utilities class. No action required now — flag for consideration if usage grows.

---

## Files Modified (Full List)

| File | WP | Change |
|---|---|---|
| `global.json` | WP-001 | SDK 10.0.202, rollForward: latestFeature |
| `Directory.Build.props` | WP-001 | TargetFramework=net10.0, NuGetAuditMode=direct |
| `src/VideoIndexer.Core/VideoIndexer.Core.csproj` | WP-001 | Removed TargetFramework |
| `src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj` | WP-001, WP-002 | Removed TFM; upgraded Microsoft.Extensions.* to 10.0.7, Serilog.Extensions.Hosting to 10.0.0 |
| `src/VideoIndexer.App/VideoIndexer.App.csproj` | WP-001, WP-002 | Removed TFM; upgraded Microsoft.Extensions.* to 10.0.7, Serilog.Extensions.Hosting to 10.0.0 |
| `tests/VideoIndexer.Tests/VideoIndexer.Tests.csproj` | WP-001, WP-003 | Removed TFM, RollForward; unified test packages |
| `tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj` | WP-001, WP-003 | Removed TFM, RollForward; unified test packages |
| `tests/VideoIndexer.Infrastructure.Tests/VideoIndexer.Infrastructure.Tests.csproj` | WP-001, WP-003 | IsTestProject, RootNamespace, unified test packages, Serilog 4.3.0 |
| `tests/VideoIndexer.Infrastructure.Tests/Database/SpdbConfigRepositoryTests.cs` | WP-003 | [AllowNull] fix on ConnectionString |
| `src/VideoIndexer.sln` | WP-003 | Added VideoIndexer.Infrastructure.Tests project |
| `src/VideoIndexer.App/Program.cs` | WP-004 | Added 5 content view DI factory registrations |
| `CHANGELOG.md` | WP-002, WP-003 | Created; documented WP-001 through WP-003 changes |
| `README.md` | WP-001 | .NET 10 references, NuGetAudit note |
| `docs/projects/rebuild/rebuild.md` | WP-001 | .NET 10+ in Tech Stack |
| `docs/projects/rebuild/milestones/m1-foundation.md` | WP-005 | net10.0 references, removed .NET 8 risk row |

---

## Next Steps for Planner / Manager

1. **Avalonia upgrade sprint** — Address the GHSA-xrw6-gwf8-vvr9 vulnerability by planning an Avalonia version upgrade. Removing `NuGetAuditMode=direct` should be a mandatory acceptance criterion of that sprint.
2. **ShellState integration tests** — Create tests or a structured smoke procedure that exercises all five `ShellState` transitions to ensure `ViewLocator` resolves views correctly end-to-end.
3. **Content view constructor comment cleanup** — A low-effort housekeeping task: update the misleading `// DI constructor` comments in the five content view code-behinds.
4. **Framework-agnostic package review** — Dapper, MySqlConnector, SharpCompress, and the Serilog sinks were intentionally left unchanged. A dedicated package audit sprint could assess whether newer major versions are warranted.
