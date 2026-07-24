# Project Synthesis — 2026-04-30-net10-upgrade-rework-2

**Date:** 2026-05-06  
**Status:** COMPLETE  
**Plan:** [plan.md](plan.md)

---

## Executive Summary

This sprint delivered the six actionable items carried forward from the `2026-04-30-net10-upgrade-rework-1` synthesis: latent Avalonia theme-resolution risk in the test project was eliminated, headless test bootstrapping was decoupled into a dedicated file, `ViewLocator.Build()` boundary conditions were formally tested, NuGet package versions were centrally managed via CPM, shared test fakes were extracted into a `TestHelpers/` directory, and all six consuming test files were migrated to the shared fakes — with dead code removed in the process.

The test suite grew by two tests (156 → 158 passing) and is left in a clean, low-duplication state. Every pipeline stage — implementation, QA, code review, and documentation — passed on the first attempt for all six work packages.

---

## Work Packages

| WP | Title | Status | Tests | Build |
|---|---|---|---|---|
| WP-001 | Add Avalonia.Themes.Fluent to test project | COMPLETE | 156 ✓ | 0 errors |
| WP-002 | Extract TestBootstrap.cs | COMPLETE | 156 ✓ | 0 errors |
| WP-003 | Add ViewLocator boundary-condition tests | COMPLETE | 158 ✓ | 0 errors |
| WP-004 | NuGet Central Package Management (CPM) | COMPLETE | 158 ✓ | 0 errors |
| WP-005 | Create shared TestHelpers/ fakes | COMPLETE | 158 ✓ | 0 errors |
| WP-006 | Migrate callers to shared fakes | COMPLETE | 158 ✓ | 0 errors |

---

## Metrics

| Metric | Value |
|---|---|
| Work packages completed | 6 / 6 |
| Pipeline stages total | 24 |
| Pipeline stages passed | 24 (100%) |
| Pipeline stages failed | 0 |
| Tests at start | 156 passed, 2 skipped, 0 failed |
| Tests at end | **158 passed, 2 skipped, 0 failed** |
| Build errors throughout | 0 |
| Build warnings throughout | 0 |
| Files modified (net) | 22 |

---

## What Was Built

### WP-001 — Avalonia.Themes.Fluent in VideoIndexer.App.Tests
Added `Avalonia.Themes.Fluent 11.3.14` as a direct `PackageReference` in `VideoIndexer.App.Tests.csproj`, eliminating the latent risk of theme-resolution failures in headless Avalonia tests. All Avalonia packages across the solution are now pinned consistently at `11.3.14`.

### WP-002 — TestBootstrap.cs extraction
Extracted the inline `TestApp` class and `[assembly: AvaloniaTestApplication]` attribute from `ViewLocatorTests.cs` into a dedicated `TestBootstrap.cs` file. `ViewLocatorTests.cs` is now correctly focused on test logic only. An inline comment was added to `TestBootstrap.cs` explaining why `AppBuilder.Configure<Application>()` (base type) is used instead of `AppBuilder.Configure<App>()` — this is intentional test isolation to avoid triggering `OnFrameworkInitializationCompleted` and DI host setup in headless tests.

### WP-003 — ViewLocator boundary-condition tests
Two new `[AvaloniaFact]` tests were added to `ViewLocatorTests.cs`:
- `Build_NullData_ReturnsNull` — covers the null-input early-return guard
- `Build_ViewTypeNotRegisteredInServices_ThrowsInvalidOperationException` — covers the `GetRequiredService` throw path when a view type is not registered in DI

Both tests follow the established `try/finally App.Services` cleanup pattern for proper test isolation. Total `ViewLocatorTests` count is now 7.

### WP-004 — NuGet Central Package Management
Enabled CPM by:
- Adding `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>` to `Directory.Build.props`
- Creating `Directory.Packages.props` at the repo root with 26 `PackageVersion` entries across 6 vendor-scoped `ItemGroup` sections
- Stripping `Version` attributes from all five target `.csproj` files (`VideoIndexer.Core.csproj` was correctly excluded — it has no `PackageReference` elements)

`Directory.Packages.props` includes a file-level XML comment identifying it as the canonical version authority, plus a file-level `README.md` section documenting the two-step process for adding new packages.

### WP-005 — Shared TestHelpers/ fakes
Created `tests/VideoIndexer.App.Tests/TestHelpers/` with four shared fake implementations:

| File | Interface | Notes |
|---|---|---|
| `FakeBootstrapper.cs` | `IDatabaseBootstrapper` | Constructor params unify fixed-result and throwing variants (`throws: true`) |
| `FakeExternalToolProvisioner.cs` | `IExternalToolProvisioner` | Caller-supplied `Exception?` for flexible error injection |
| `FakePasswordService.cs` | `IPasswordService` | Mutable properties for per-test configuration |
| `FakeDatabaseConnectionStore.cs` | `IDatabaseConnectionStore` | Full CRUD semantics; auto-Guid assignment; clears `ActiveId` on `DeleteAsync` |

### WP-006 — Migrate callers to shared fakes
All six `VideoIndexer.App.Tests` test files were migrated to consume the shared `TestHelpers/` fakes. Inline duplicates of `FakeBootstrapper`, `FakeExternalToolProvisioner`, `FakePasswordService`, and `FakeDatabaseConnectionStore` were removed. Two fakes were intentionally retained as private-to-file:
- `FakeStore` in `DatabaseConnectorViewModelTests.cs` (not covered by shared fakes)
- `FakeThemeService` in `MainWindowViewModelTests.cs` (not covered by shared fakes)

The code reviewer applied two Fix-Forward changes: removed 4 orphaned `using` directives from `ViewLocatorTests.cs` and removed the dead `TrackingShell` class from `DatabaseConnectorViewModelTests.cs`.

---

## Strategic Recommendations (Gold Nuggets)

### 1. Normalize exception-injection API across shared fakes (medium priority)
The four shared fakes have inconsistent error-injection APIs: `FakeExternalToolProvisioner` accepts a caller-supplied `Exception?` (most flexible), while `FakeBootstrapper`, `FakePasswordService`, and `FakeDatabaseConnectionStore` use `bool` flags and hardcode `InvalidOperationException` internally. A future refactor should normalize to the `Exception?` pattern for consistency and to allow tests to assert on specific exception messages.

### 2. `FakeBootstrapper` async throw fidelity (low priority)
`FakeBootstrapper.CheckAsync` throws synchronously rather than returning a faulted `Task`. All current call sites use `try/catch` around `await`, so the behavior is functionally correct today. For strict async-interface fidelity, consider: `return Task.FromException<DatabaseBootstrapResult>(new InvalidOperationException(...))`. Safe to defer.

### 3. Consider promoting `TrackingPasswordService` to TestHelpers/ (low priority)
`LogonViewModelTests.cs` and `PasswordSetupViewModelTests.cs` both define private `TrackingPasswordService` variants that add call-tracking lists (`LogOnCalls`, `SetCalls`). If tracking behavior is needed elsewhere, this is a candidate for promotion to a shared `TestHelpers/TrackingPasswordService.cs`. No action required now.

### 4. `FakeThemeService` coverage gap (low priority)
`FakeThemeService` exists only in `MainWindowViewModelTests.cs` and has no shared equivalent. If `IThemeService` behavior needs to be tested in other future test files, it should be added to `TestHelpers/`.

### 5. CPM spec count discrepancy (informational)
The WP-004 acceptance criterion specified 27 packages but the actual unique count is 26. The WP spec count was off by one — all actual `PackageReference` elements are covered. Update the plan notes if this plan is referenced in future sprints.

---

## Deferred / Out-of-scope Items

| Item | Priority | Suggested Next Sprint |
|---|---|---|
| Normalize exception-injection API in shared fakes | Medium | Yes |
| `FakeBootstrapper` async throw fidelity | Low | Optional |
| Promote `TrackingPasswordService` to TestHelpers/ | Low | Optional |
| Add `FakeThemeService` to TestHelpers/ | Low | Only if needed |

---

## Files Modified (Summary)

| File | WPs |
|---|---|
| `CHANGELOG.md` | WP-001, WP-002 |
| `Directory.Build.props` | WP-004 |
| `Directory.Packages.props` | WP-004, WP-004 docs |
| `README.md` | WP-004 docs, WP-005 docs, WP-006 docs |
| `src/VideoIndexer.App/VideoIndexer.App.csproj` | WP-004 |
| `src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj` | WP-004 |
| `tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj` | WP-001, WP-004 |
| `tests/VideoIndexer.Infrastructure.Tests/VideoIndexer.Infrastructure.Tests.csproj` | WP-004 |
| `tests/VideoIndexer.Tests/VideoIndexer.Tests.csproj` | WP-004 |
| `tests/VideoIndexer.App.Tests/TestBootstrap.cs` | WP-002, WP-002 docs |
| `tests/VideoIndexer.App.Tests/ViewLocatorTests.cs` | WP-002, WP-003, WP-003 docs, WP-006 |
| `tests/VideoIndexer.App.Tests/TestHelpers/FakeBootstrapper.cs` | WP-005, WP-005 docs |
| `tests/VideoIndexer.App.Tests/TestHelpers/FakeExternalToolProvisioner.cs` | WP-005 |
| `tests/VideoIndexer.App.Tests/TestHelpers/FakePasswordService.cs` | WP-005 |
| `tests/VideoIndexer.App.Tests/TestHelpers/FakeDatabaseConnectionStore.cs` | WP-005 |
| `tests/VideoIndexer.App.Tests/ShellViewModelTests.cs` | WP-006 |
| `tests/VideoIndexer.App.Tests/LogonViewModelTests.cs` | WP-006 |
| `tests/VideoIndexer.App.Tests/PasswordSetupViewModelTests.cs` | WP-006 |
| `tests/VideoIndexer.App.Tests/DatabaseConnectorViewModelTests.cs` | WP-006 |
| `tests/VideoIndexer.App.Tests/MainWindowViewModelTests.cs` | WP-006 |
