# Project Synthesis Report
**Plan:** `2026-04-30-net10-upgrade-rework-1`
**Date:** 2026-05-05
**Status:** COMPLETE

---

## Executive Summary

This sprint addressed all four actionable items carried forward from the `.NET 10 Upgrade` synthesis. The work resolved a HIGH-severity transitive vulnerability in Avalonia, removed a debt shield in the build configuration, added unit test coverage for `ViewLocator`, corrected misleading constructor comments across five view code-behinds, and audited and bumped five framework-agnostic infrastructure packages.

All four work packages completed with no failures. The codebase exits this sprint with a clean build (0 errors, 0 warnings), a fully green test suite (156 passed, 0 failed), and a verified zero-vulnerable-packages posture.

---

## Work Package Summary

| WP | Title | Stages | All AC Met |
|---|---|---|---|
| WP-001 | Avalonia 11.3.14 Upgrade + Transitive Audit Restore | impl → qa → security-audit → code-review → documentation | 6/6 ✓ |
| WP-002 | View Constructor Comment Cleanup | impl → qa → code-review | 4/4 ✓ |
| WP-003 | Framework-Agnostic Package Audit | impl → qa → security-audit → code-review → documentation | 6/6 ✓ |
| WP-004 | ViewLocator Unit Tests | impl → qa → code-review → documentation | 6/6 ✓ |

**Total pipeline stages completed:** 17 / 17 — all PASS.

---

## Metrics

### Test Suite

| Metric | Value |
|---|---|
| Tests passed | **156** |
| Tests failed | **0** |
| Tests skipped | **2** (environment-gated: live DB / network fixture) |
| Tests total | 158 |
| Net new tests (WP-004) | +5 `[AvaloniaFact]` ViewLocator tests |

The baseline entering this sprint was approximately 153 total tests (151 passed, 2 skipped per environment). WP-004 added 5 new tests, bringing the solution total to 156 passed.

### Build Health

| Metric | Value |
|---|---|
| Build errors | 0 |
| Build warnings | 0 |
| Projects in solution | 6 |
| `TreatWarningsAsErrors` | Active |

### Security

| Metric | Value |
|---|---|
| Vulnerable packages (post-sprint) | **0** |
| CVEs resolved | **1** — GHSA-xrw6-gwf8-vvr9 (HIGH) |
| OWASP findings | 0 Critical / 0 High / 0 Medium |

---

## Artifacts Modified

| File | WP(s) |
|---|---|
| `src/VideoIndexer.App/VideoIndexer.App.csproj` | WP-001 |
| `Directory.Build.props` | WP-001 |
| `src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj` | WP-003 |
| `src/VideoIndexer.App/Properties/AssemblyInfo.cs` *(created)* | WP-004 |
| `tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj` | WP-004 |
| `tests/VideoIndexer.App.Tests/ViewLocatorTests.cs` *(created)* | WP-004 |
| `src/VideoIndexer.App/Views/ProvisioningToolsView.axaml.cs` | WP-002 |
| `src/VideoIndexer.App/Views/DatabaseConnectorView.axaml.cs` | WP-002 |
| `src/VideoIndexer.App/Views/LogonView.axaml.cs` | WP-002 |
| `src/VideoIndexer.App/Views/PasswordSetupView.axaml.cs` | WP-002 |
| `src/VideoIndexer.App/Views/MainContentView.axaml.cs` | WP-002 |
| `README.md` | WP-001 (doc), WP-004 (doc) |
| `CHANGELOG.md` | WP-001 (impl + doc), WP-003 (impl + doc) |

---

## Security Highlights (WP-001 + WP-003)

**GHSA-xrw6-gwf8-vvr9 — RESOLVED.**
Avalonia 11.2.5 pulled in `Tmds.DBus.Protocol ≤ 0.20.0` (HIGH severity, remote code execution vector on D-Bus systems). Upgrading to Avalonia 11.3.14 updates the transitive to `Tmds.DBus.Protocol 0.21.3`, eliminating the advisory. A live `dotnet list package --vulnerable --include-transitive` scan post-sprint confirms 0 vulnerable packages across all 6 projects.

**NuGetAuditMode=direct — REMOVED.**
The `<NuGetAuditMode>direct</NuGetAuditMode>` debt shield in `Directory.Build.props` was suppressing transitive dependency audit warnings at build time. Its removal restores full transitive NuGet audit coverage. No new audit warnings surfaced after removal, confirming the environment is clean.

---

## Package Version Changes

### WP-001 — Avalonia

| Package | Before | After |
|---|---|---|
| Avalonia | 11.2.5 | **11.3.14** |
| Avalonia.Desktop | 11.2.5 | **11.3.14** |
| Avalonia.Themes.Fluent | 11.2.5 | **11.3.14** |
| Avalonia.Fonts.Inter | 11.2.5 | **11.3.14** |

### WP-003 — Infrastructure packages

| Package | Before | After | Notes |
|---|---|---|---|
| Dapper | 2.1.35 | **2.1.72** | Non-breaking |
| MySqlConnector | 2.3.7 | **2.5.0** | Non-breaking |
| SharpCompress | 0.38.0 | **0.47.4** | Non-breaking (narrow XZStream usage surface) |
| Serilog.Sinks.Console | 6.0.0 | **6.1.1** | Non-breaking |
| Serilog.Sinks.File | 6.0.0 | **7.0.0** | Major bump — confirmed non-breaking for current `WriteTo.File` call signature |

### WP-004 — Test infrastructure

| Package | Before | After |
|---|---|---|
| Avalonia.Headless.XUnit | *(absent)* | **11.3.14** |

---

## Strategic Recommendations (Gold Nuggets)

### 1. Adopt Central Package Management (`Directory.Packages.props`) — Low Priority
*Raised by: WP-001 Implementation + Code Review*

All four Avalonia packages must move in lockstep; they are always updated together. A `Directory.Packages.props` file (NuGet Central Package Management) would reduce version skew risk if Avalonia references ever appear in multiple projects, and would consolidate all version strings in one location. Not urgent at current single-project scope, but worth adopting before the solution grows.

### 2. Extract `TestApp` to `TestBootstrap.cs` — Medium Priority
*Raised by: WP-004 Code Review (documentation-forward)*

`TestApp` is defined inside `ViewLocatorTests.cs` but the `[assembly: AvaloniaTestApplication]` attribute scopes it to the entire test assembly. Any contributor adding `[AvaloniaFact]` tests to a new file will find no obvious entry point for the headless bootstrap. Extracting `TestApp` to a dedicated `TestBootstrap.cs` at the project root makes the shared infrastructure immediately discoverable. README.md documents its existence, but the code-level discoverability is still low.

### 3. Consolidate Shared Test Fakes — Low Priority
*Raised by: WP-004 Implementation + QA*

`FakeDatabaseConnectionStore`, `FakeBootstrapper`, `FakePasswordService`, and `FakeExternalToolProvisioner` are duplicated between `ViewLocatorTests.cs` and `ShellViewModelTests.cs`. A shared `TestHelpers/` folder or a dedicated `VideoIndexer.TestHelpers` assembly would eliminate drift between the two copies and reduce maintenance overhead as test coverage grows.

### 4. Add `ViewLocator.Build()` Edge-Case Tests — Low Priority
*Raised by: WP-004 QA (coverage-gap)*

The new `ViewLocator` tests cover all five registered content views. Two paths remain untested:
- `ViewLocator.Build(null)` → returns `null` (trivially correct by inspection)
- View type resolved from assembly but not registered in `App.Services` → `GetRequiredService` throws `InvalidOperationException` with no fallback. A future ViewModel pointing to an unregistered view would surface as a runtime crash during navigation, not at startup. A test documenting this contract would prevent silent regressions.

### 5. Monitor `SharpCompress` Upgrade Path — Low Priority
*Raised by: WP-003 Implementation + Code Review*

`SharpCompress` spans 9 minor versions (`0.38.0 → 0.47.4`) in `0.x` space where semver guarantees do not formally apply. The upgrade was clean because the usage surface is limited to `XZStream`. Future maintainers should treat any `SharpCompress` bump as requiring explicit build + test validation — the package's `0.x` convention does not promise backward compatibility across minor versions.

### 6. Add `Avalonia.Themes.Fluent` to Test Project — Low Priority
*Raised by: WP-004 QA (convention)*

The `VideoIndexer.App.Tests` project has no Avalonia theme package. Tests pass because `ViewLocator.Build()` does not trigger full style resolution, but deeper headless tests that render controls could exhibit flakiness. Adding `Avalonia.Themes.Fluent 11.3.14` is a one-line `csproj` change that eliminates this latent risk.

---

## Next Steps (Planner / Program Manager)

1. **Avalonia 12.x migration planning:** The 12.x series (currently 12.0.2) is the next major Avalonia track with documented breaking changes. A separate migration sprint with its own breaking-change analysis is the correct vehicle — this is out of scope for this plan but is the logical follow-on.

2. **Address open debt items (low priority, next maintenance window):**
   - Extract `TestApp` to `TestBootstrap.cs` (WP-004, medium priority)
   - Add `ViewLocator.Build(null)` and unregistered-view edge-case tests (WP-004, low priority)
   - Consolidate shared test fakes into a `TestHelpers/` folder (WP-004, low priority)
   - Add `Avalonia.Themes.Fluent` to `VideoIndexer.App.Tests.csproj` (WP-004, low priority)

3. **Central Package Management:** Consider adopting `Directory.Packages.props` before the next major dependency upgrade cycle.

4. **Scheduled package audit cadence:** All infrastructure packages are now at their latest stable versions. A recurring quarterly or per-sprint audit cadence for the five framework-agnostic packages would prevent the version lag that accumulated before this sprint.

---

*Report generated by Synthesis Agent — Head of Operations*
*Session date: 2026-05-05*
