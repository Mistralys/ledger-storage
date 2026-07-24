# Synthesis Report — M8 System Tools

**Plan:** `2026-05-19-m8-system-tools`  
**Date:** 2026-05-20  
**Status:** COMPLETE — All 15 work packages delivered  
**Version shipped:** 1.1.1 → **1.2.0** (minor bump; new public API + schema migration)

---

## Executive Summary

M8 delivered the full **System Tools** milestone for Video Indexer MKII. The release ships three independent but interconnected subsystems:

1. **Six-tab Preferences page** — `SettingsViewModel` + `SettingsView` replace the previous `StubViewModel`/`StubView` stub at the "settings" primary-navigation destination. Tabs: General, Database (read-only), Thumbnails, Appearance, Logging, Obfuscation. Save/Cancel operate on an immutable `AppOptions` snapshot via `with { }` + `ISettingsService.SaveAsync`.

2. **Database Backup workflow** — `MysqlDumpBackupService` invokes `mysqldump` via `ProcessStartInfo.ArgumentList` (shell-injection-safe by construction). A modal `DatabaseBackupView` dialog is surfaced from a new **File** menu bar added to `MainContentView`.

3. **File Obfuscation toggle** — `ObfuscationService.ToggleAsync` renames movie files to/from their hash values, with cooperative cancellation (returns normally, never throws `OperationCanceledException`), per-file progress reporting, and the `spdb_config["obfuscation_enabled"]` flag written on both normal completion and cancellation. The Obfuscation tab in Preferences exposes this as `ObfuscationSettingsViewModel`.

Supporting infrastructure: **schema migration m041** adds `movies.original_filename VARCHAR(255) NULL`, bumping `DatabaseBootstrapper.ExpectedRevision` 40 → 41, with full rollback procedure documented.

---

## Metrics

| Metric | Value |
|---|---|
| Work packages completed | 15 / 15 |
| Pipeline stages executed | 64 (sum of all passed stages) |
| WPs requiring rework | 4 |
| Final test count (full suite) | **814 passed, 0 failed, 6 skipped** |
| Build warnings at close | **0** |
| Security issues (critical / high) | **0 / 0** |
| Security issues (medium) | 1 |
| New files created | ~27 |
| Version | 1.1.1 → **1.2.0** |
| Schema revision | 40 → **41** |

### Test Count Progression

| Milestone | Tests Passed |
|---|---|
| WP-001/002/003 baseline | 777 |
| After m041 applied to test DB (WP-004) | 786 |
| After WP-004 rework + full suite | 786 |
| After WP-005–009 | 800 |
| After WP-010–011 | 810 |
| After WP-012–014 | **814** |

---

## Rework Events

Four work packages required rework before reaching a final PASS. These are recorded for process insight.

| WP | Stage | Root Cause | Resolution |
|---|---|---|---|
| **WP-004** | QA FAIL | Dapper materialisation bug: `MySqlConnector` maps `INT UNSIGNED` to `System.UInt32`, but `MovieForObfuscation` expects `long`. Direct `QueryAsync<MovieForObfuscation>` threw `InvalidOperationException` at runtime. | Added `private sealed class MovieObfuscationRow` (plain class with property setters, `long MovieId`), following the existing `MovieRow` pattern. |
| **WP-008** | QA FAIL | The implementation pipeline was mis-targeted — it verified the already-complete `MysqlDumpBackupService` instead of creating `ObfuscationService.cs`. The file was entirely absent. | Developer created `ObfuscationService.cs` in full; QA re-ran to PASS. |
| **WP-009** | Code-Review FAIL | `SaveCommand` did not update `_loadedTheme` after `SaveAsync` succeeded. A second save after a theme change would skip `IThemeService.SetAsync`, leaving the UI theme stale while the settings file was correct. | Developer added `_loadedTheme = SelectedTheme;` after `SaveAsync`. |
| **WP-011** | Code-Review FAIL (2 blockers) | (1) `async void OnLoaded` caught only `OperationCanceledException` — any non-cancellation exception would crash the process. (2) `OnUnloaded` did not unsubscribe `ObfuscationVm.PropertyChanged`, creating an event-handler memory leak. | Developer added general `catch (Exception)` handler (consistent with `MoviesListView` template) and added unsubscription to `OnUnloaded`. |

---

## Security Findings

### Medium (1)

| Finding | File | Detail | Status |
|---|---|---|---|
| Password visible in process listing | `MysqlDumpBackupService.cs` | `-p{password}` passed as an `ArgumentList` entry prevents shell injection but is transiently visible in OS process listings (Task Manager, `ps aux`). Any local user with process-inspection rights can read it for the mysqldump process lifetime. | **Open** — acceptable for current desktop-only threat model. Remediation: write a `--defaults-extra-file` temp file with owner-read-only permissions. |

### Low (4)

| Finding | WP | Summary |
|---|---|---|
| Path traversal in output filename | WP-005 | `conn.Database` used directly in output filename; if it contains path separators, `Path.Combine` resolves outside the destination folder. Remediation: sanitize with `Path.GetInvalidFileNameChars()`. |
| Partial backup left on cancellation | WP-005 | Truncated `.sql` file remains on disk on `OperationCanceledException`. Remediation: delete file in finally on cancellation. |
| `BackupCommand` no `CanExecute` guard | WP-006 | Backup button active with empty `DestinationFolder` — dump could be written to process CWD. Remediation: `CanExecute = () => !string.IsNullOrWhiteSpace(DestinationFolder)`. |
| No cancel path while IsBusy | WP-006 | Dialog cannot be dismissed while mysqldump runs; no CancelBackupCommand exists. |

---

## Strategic Recommendations (Gold Nuggets)

### 1. Dapper UInt32→long Convention Must Be in `constraints.md`

**Source:** WP-004 rework, WP-004 Reviewer  
`MySqlConnector` maps `INT UNSIGNED` to `System.UInt32`. Direct `QueryAsync<T>` where `T` has `long` ID properties will throw at runtime. The project now has two private row-class workarounds (`MovieRow`, `MovieObfuscationRow`) but no documented convention. **Action:** Add a `constraints.md` rule documenting the private row-class pattern for any query materialising a positional record with a `long` ID. Alternatively, register a `SqlMapper.AddTypeHandler` for `UInt32 → long` coercion to eliminate per-query boilerplate.

### 2. `SettingsViewModel` Needs Unit Tests for Non-Trivial Behaviour

**Source:** WP-009 rework (missed `_loadedTheme` update), WP-009/012 QA coverage-gap notes  
The `_loadedTheme` guard (only call `SetAsync` when theme changed) and `CancelCommand` not reloading `ObfuscationVm` are both observable contracts that were caught in code review, not automated tests. `SettingsViewModelTests` (added in WP-012) covers the happy path but not these edge cases. **Action:** Add tests for: (a) `SaveCommand` updates `_loadedTheme` so a second save skips `SetAsync`; (b) `CancelCommand` does not invoke `ObfuscationVm.LoadAsync`.

### 3. `async void` Load Handlers Must Always Follow the `MoviesListView` Template

**Source:** WP-011 rework (crash-risk blocker)  
Two deviations from the established template (`MoviesListView.axaml.cs`) caused a code-review failure: missing general exception catch and missing event unsubscription in `OnUnloaded`. The fix required a rework cycle. **Action:** Document the `MoviesListView.axaml.cs` code-behind pattern formally in `constraints.md` as the canonical template for view lifecycle management, listing: `OnLoaded` must catch `(Exception)`, `OnUnloaded` must unsubscribe all event handlers, `CancellationTokenSource` must be cancelled and disposed on unload.

### 4. Consolidate Stub/Fake Infrastructure to Avoid Duplication

**Source:** WP-012, WP-013 comments  
`StubObfuscationMap` appears as a private nested class in both `SettingsViewModelTests.cs` and `ObfuscationSettingsViewModelTests.cs`. The `NullDbConnection`/`NullDbCommand` family in `ObfuscationServiceTests.cs` partially duplicates `FakeDbConnectionFactory.cs`. **Action:** Promote `StubObfuscationMap` to `tests/VideoIndexer.App.Tests/TestHelpers/FakeObfuscationMap.cs`. Evaluate whether `NullDbConnection` should be promoted to shared infrastructure in `VideoIndexer.Tests/Fixtures/`.

### 5. `DatabaseBackupViewModel` Needs a `CanExecute` Guard

**Source:** WP-006 security review, WP-006 QA  
Both QA and Security Audit independently flagged that clicking Backup with an empty `DestinationFolder` silently writes the dump to the process working directory — a security and UX problem. **Action:** Add `CanExecute = () => !string.IsNullOrWhiteSpace(DestinationFolder)` to the `[RelayCommand]` attribute on `Backup()` in `DatabaseBackupViewModel`.

### 6. Obfuscation Toggle Needs a Cancel Button

**Source:** WP-006 security audit, WP-006 code review  
The `DatabaseBackupView` (and by extension, the Obfuscation toggle) has no user-facing cancel mechanism. If the underlying operation hangs, the dialog is undismissable without OS-level intervention. **Action:** Add a `CancelBackupCommand` to `DatabaseBackupViewModel` that cancels the in-flight task, and re-enable the Close button while `IsBusy = true`.

### 7. `GetAllMoviesForObfuscationAsync` Correlated-Subquery Replaced — Document the Pattern

**Source:** WP-004 implementation (replaced WP-003's subquery with INNER JOIN)  
WP-003 originally implemented a correlated-subquery form that would run once per row at O(n) cost. WP-004 replaced it with an INNER JOIN. This happened because WP-003 delivered the "contract only" WP and made an incorrect implementation choice. **Action:** Document in `constraints.md` that DapperMovieRepository queries using per-movie sub-aggregation must use INNER JOIN + derived tables, not correlated subqueries.

---

## Per-WP Summary

| WP | Scope | Stages | Result |
|---|---|---|---|
| WP-001 | `ThumbnailsOptions` record; `AppOptions.Thumbnails`; `ExternalToolsOptions.UseVlc` | Impl → QA → Review → Docs | PASS |
| WP-002 | Migration m041; `ExpectedRevision` = 41; `GetOriginalFilenameAsync` COALESCE; version 1.2.0 | Impl → QA → Review → RE → Docs | PASS |
| WP-003 | Core contracts: `IObfuscationService`, `IDatabaseBackupService`, 3 model records, `IMovieRepository` extension | Impl → QA → Review → Docs | PASS |
| WP-004 | `GetAllMoviesForObfuscationAsync` INNER JOIN impl; **1 rework** (Dapper UInt32 bug) | Impl → QA (FAIL) → Impl → QA → Review → Docs | PASS |
| WP-005 | `MysqlDumpBackupService` (`mysqldump` process invocation, `ArgumentList`) | Impl → QA → Security → Review → Docs | PASS |
| WP-006 | `DatabaseBackupView` + `DatabaseBackupViewModel` + `AvaloniaDatabaseBackupDialogService` | Impl → QA → Security → Review → Docs | PASS |
| WP-007 | `ObfuscationSettingsViewModel` (`ToggleCommand`, `ExtensionMap`, `IsBusy`) | Impl → QA → Review → Docs | PASS |
| WP-008 | `ObfuscationService` (enable/disable, file rename, cooperative cancellation); **1 rework** (missing impl) | Impl → QA (FAIL) → Impl → QA → Security → Review → Docs | PASS |
| WP-009 | `SettingsViewModel` (6-tab, Save/Cancel, `_loadedTheme` guard); **1 rework** | Impl → QA → Review (FAIL) → Impl → QA → Review → Docs | PASS |
| WP-010 | Integration tests: m041 migration validation + `GetAllMoviesForObfuscationAsync` live-DB test | Impl → QA → Review → Docs | PASS |
| WP-011 | `SettingsView.axaml` (6 tabs, compiled bindings, CTS lifecycle); **1 rework** (2 blockers) | Impl → QA → Review (FAIL) → Impl → QA → Review → Docs | PASS |
| WP-012 | Unit tests: `SettingsViewModelTests`, `ObfuscationSettingsViewModelTests`, `DatabaseBackupViewModelTests` + 3 fake helpers | Impl → QA → Review → Docs | PASS |
| WP-013 | Unit tests: `ObfuscationServiceTests` (12 methods), `MysqlDumpBackupServiceTests` (3 new methods) | Impl → QA → Review → Docs | PASS |
| WP-014 | `MainContentView` File menu bar; DI wiring of all 6 M8 services/VMs in `Program.cs` | Impl → QA → Review → Docs | PASS |
| WP-015 | Project manifest update (`api-surface.md`, `constraints.md`, `data-flows.md`, `file-tree.md`, `tech-stack.md`, milestone doc) | Docs | PASS |

---

## Next Steps for the Planner

The following items are deferred work identified during this cycle:

1. **Security (medium):** `MysqlDumpBackupService` password exposure — implement `--defaults-extra-file` pattern.
2. **UX/Security (low):** `DatabaseBackupViewModel` — add `CanExecute` guard for empty `DestinationFolder`.
3. **UX:** `DatabaseBackupView` — add cancel-while-busy capability.
4. **Tests:** `SettingsViewModel` — add unit tests for `_loadedTheme` guard and `CancelCommand` contract (not regressed, but unverified by automation).
5. **Constraints doc:** Document the `MoviesListView` code-behind template as the canonical view lifecycle pattern.
6. **Constraints doc:** Document the Dapper `UInt32 → long` private-row-class convention (or add a global `TypeHandler`).
7. **Test infrastructure:** Promote `StubObfuscationMap` to a shared test helper.
8. **Error UX:** `SettingsView.axaml` — bind `HasLoadError`/`LoadErrorMessage` (added in rework) to a visible error banner in the view.
9. **Obfuscation Errors UX:** Obfuscation tab "Errors" label is bound to `IsBusy` — consider binding to `Errors.Count > 0` instead, so errors remain visible after the operation completes.
10. **M9 scope:** `GetOriginalFilenameAsync` on `IMovieRepository` returns the earliest-recorded filename via `COALESCE(m.original_filename, mf.filename ORDER BY created_at ASC)`. No M8 UI surfaces this — consider surfacing "original filename" in the movie editor or details panel.

---

*Generated by Synthesis Agent — 2026-05-20*
