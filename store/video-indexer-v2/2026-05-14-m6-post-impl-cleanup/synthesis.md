# Synthesis Report — M6 Post-Implementation Cleanup

**Date:** 2026-05-14  
**Plan:** `2026-05-14-m6-post-impl-cleanup`  
**Status:** COMPLETE  
**Work Packages:** 6 / 6 COMPLETE  
**Version Released:** 1.0.0 → **1.1.0**

---

## Executive Summary

This session delivered all ten carry-forward items from the M6 Synthesis report, organised into six work packages. No new user-visible features were introduced; every change resolved a code-quality, correctness, security, or test-coverage gap identified in the previous cycle. Work spanned three distinct concerns:

1. **Structural cleanup** — test-infrastructure deduplication (WP-001) and missing test coverage (WP-005).
2. **Correctness and robustness** — defensive exception handling in `LoadFilterSlotsAsync`, event-handler guard ordering in `OnEditorCloseRequested`, and deterministic `GetOriginalFilenameAsync` behaviour backed by a new schema column (WP-002, WP-004).
3. **Security and persistence** — path-injection remediation in `WindowsFileLauncherService` (WP-003) and full `ISettingsService` wiring for `LabelCleanerOptions` persistence (WP-006).

The session concluded with a minor-version release (1.1.0). All pipelines passed with zero blocking findings.

---

## Metrics

| Metric | Value |
|---|---|
| Work packages completed | 6 / 6 |
| Total pipeline stages passed | 20 / 20 |
| Reviewer Fix-Forwards applied | 3 |
| Security audit findings (Critical/High/Medium) | 0 / 0 / 0 |
| Security audit findings (Low) | 1 (silent Process.Start discard — not exploitable) |
| Final test count (non-skipped) | **626 passed, 0 failed** |
| Skipped tests (live-DB, unconfigured) | 6 |
| Build warnings | 0 |
| Build errors | 0 |
| Schema revision | 38 → **39** |
| Version | 1.0.0 → **1.1.0** |

### Test Count Progression

| After WP | Tests Passing |
|---|---|
| WP-001 QA | 613 |
| WP-002 QA | 613 |
| WP-003 QA | 613 |
| WP-004 QA | 614 (+1 new integration test) |
| WP-005 QA | 614 + 9 new App tests = implied ~623 |
| WP-006 QA | **626** (+3 new App tests) |

---

## Work Package Outcomes

### WP-001 — Test Infrastructure Consolidation
**Goal:** Eliminate four duplicate `NonDisposingConnection` / `SharedConnectionFactory` private inner classes from test files.

**Delivered:**
- Created `tests/VideoIndexer.Infrastructure.Tests/TestHelpers/SharedTransactionalConnectionFactory.cs` containing both shared types using `IDbConnection`/`IDbTransaction` interfaces (maximally general, not tied to `MySqlConnector` concrete types).
- Removed all four per-file duplicates from `DapperMovieRepositoryTests`, `DapperMovieCatalogRepositoryTests`, `DapperFilterSlotRepositoryTests`, and `LibraryScannerIntegrationTests`.
- Updated `SpdbConfigRepositoryTests` to use `new NonDisposingConnection(_connection, null)` via the shared class.
- All 4 acceptance criteria met; build clean; 613 tests pass.

**Residual note:** `SpdbConfigRepositoryTests` retains a local `SingleConnectionFactory` class — this is intentional because `SharedConnectionFactory` requires a non-null `IDbTransaction`. If `SharedConnectionFactory` is ever made nullable, the local class can be eliminated.

---

### WP-002 — Defensive Code Fixes
**Goal:** Prevent unhandled exceptions from `LoadFilterSlotsAsync` propagating to the UI; guard `OnEditorCloseRequested` against calling `CloseSubView()` when the page has already changed.

**Delivered:**
- `MoviesListViewModel.LoadFilterSlotsAsync`: wrapped body in `try/catch (Exception ex) when (ex is not OperationCanceledException)` — logs error, clears `FilterSlots`, sets `ActiveFilterSlot = null`. `OperationCanceledException` correctly re-propagates.
- `MainContentViewModel.OnEditorCloseRequested`: unsubscription is now unconditional and ordered *before* the `CurrentPage != MainContentPage.MovieEditor` guard. Correct ordering satisfies AC-2a.
- Added `FakeFilterSlotRepository.SetException()` and two new unit tests; 613 tests pass.

---

### WP-003 — Security: WindowsFileLauncherService Path Injection Fix
**Goal:** Eliminate `Arguments =` string concatenation in `WindowsFileLauncherService` to prevent OWASP A03 argument injection on paths containing spaces or special characters.

**Delivered:**
- `OpenFolder`: replaced `Arguments = path` with `ArgumentList.Add(path)`.
- `ShowInExplorer`: replaced manual quoting with `ArgumentList.Add($"/select,{filePath}")` as a **single entry** (two-entry split would break `explorer.exe` token parsing — correctly avoided).
- Added `constraints.md` bullet under UI & MVVM to codify the `ArgumentList` requirement with rationale and a Do-Not-Revert directive.
- Security audit: **0 Critical, 0 High, 0 Medium**; 1 Low (Process.Start result silently discarded — acceptable for UI convenience operations where the user directly observes the outcome).
- All 3 acceptance criteria met; 613 tests pass.

---

### WP-004 — Schema Migration m039 + GetOriginalFilenameAsync Determinism
**Goal:** Add `created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP` to `movies_filenames`, bump schema to revision 39, and make `GetOriginalFilenameAsync` return the earliest-inserted filename deterministically.

**Delivered:**
- `m039_add_filenames_created_at.sql`: adds `created_at` column and drops the legacy `hash_2` UNIQUE KEY (which incorrectly prevented multiple filenames per movie — an inherited defect from the legacy app).
- `DatabaseBootstrapper.ExpectedRevision` bumped from 38 to 39; sentinel test updated.
- `GetOriginalFilenameAsync` updated with `ORDER BY mf.created_at ASC` before `LIMIT 1`.
- New integration test `GetOriginalFilenameAsync_WithMultipleFilenames_ReturnsEarliest` confirms correctness against a live DB.
- All three manifest documents (`constraints.md`, `api-surface.md`, `tech-stack.md`) updated with revision 39.
- Reviewer Fix-Forward: corrected stale `'literal 38'` comment in sentinel test to `'literal 39'`.
- **Version bumped to 1.1.0** (minor — backwards-compatible DDL, no breaking API changes).
- 614 tests pass.

**Note:** The `hash_2` UNIQUE KEY removal is a notable schema correctness fix beyond the original WP scope — it was a prerequisite for the integration test to be meaningful (multiple filenames per movie must be storable).

---

### WP-005 — Test Coverage Gaps
**Goal:** Add `IntDecimalConverterTests.cs`, `MoveActorDown_IsNoOp_WhenNoActorSelected`, `FakeMovieRepository.SetOriginalFilename`, and a `CleanLabel` primary-path test.

**Delivered:**
- `FakeMovieRepository.SetOriginalFilename(string?)`: consistent with the established `SetMovie`/`SetException` configuration pattern.
- `IntDecimalConverterTests.cs` (new file in `tests/VideoIndexer.App.Tests/Converters/`): 7 tests covering all 3 `Convert` branches and all 4 `ConvertBack` branches.
- `MoveActorDown_IsNoOp_WhenNoActorSelected`: mirrors the existing `MoveActorUp` symmetry counterpart.
- `CleanLabel_CallsGetOriginalFilenameAsync_AndPassesFilenameStemToLabelCleaner`: primary-path test verifying invocation count, label cleaner show, and `RawFilename` stem.
- Reviewer Fix-Forward: moved `_originalFilename` field and `SetOriginalFilename()` from the `IMovieRepository` implementation region to the `Configuration` section of `FakeMovieRepository.cs`, aligning with the established grouping convention.
- 232/232 App tests pass; 0 failures.

---

### WP-006 — LabelCleanerOptions Persistence
**Goal:** Inject `ISettingsService` into `MovieEditorViewModel` so the Label Cleaner dialog reads initial options from settings and saves accepted options back on accept.

**Delivered:**
- `ISettingsService?` injected as optional parameter into `MovieEditorViewModel` (parameterless constructor continues to work with default options — AC-11c).
- `CleanLabel` command reads `_settingsService?.Current.LabelCleaner ?? new LabelCleanerOptions()` as initial state; on accept, calls `SaveAsync` using the immutable `with { LabelCleaner = vm.ToOptions() }` pattern.
- `LabelCleanerViewModel.ToOptions()` added as a new public method returning current toggle state.
- `FakeSettingsService` extended with `SaveCallCount` and `LastSaved` tracking (consistent with other fakes).
- DI wiring updated in `Program.cs`.
- Reviewer Fix-Forward: strengthened `CleanLabel_SavesOptionsToSettings_OnDialogAccept` test with property-level assertions on `LastSaved.LabelCleaner` (the original only checked `SaveCallCount==1` and `LastSaved!=null`).
- All 4 acceptance criteria met; 626 tests pass.

---

## Open Documentation-Forward Items

These items were logged during review but deferred to the Documentation agent:

| # | Item | Source |
|---|---|---|
| D-1 | `file-tree.md`: `VideoIndexer.Infrastructure.Tests/` section is a bare stub — expand to list `TestHelpers/SharedTransactionalConnectionFactory.cs`, `TestHelpers/LiveDbFixture.cs`, `Database/`, `Library/` subdirectories. | WP-001 Reviewer |
| D-2 | `constraints.md`: Document migration file naming convention — `m{revision}_{description}.sql`, lowercase snake_case. Consistent across m036–m039 but undocumented. | WP-004 Developer + Reviewer |
| D-3 | `README.md`: `SpdbConfigLibraryFolderRepository` → Architectural invariant section references the `hash_2` UNIQUE KEY which was removed in m039. Update to reflect `SELECT DISTINCT` cardinality handling. | WP-004 Release Engineer |
| D-4 | `api-surface.md`: Add `LabelCleanerViewModel.ToOptions()` signature, return type, and purpose (captures toggle state for settings persistence). | WP-006 Reviewer |
| D-5 | `FakeMovieRepository` class-level XML doc: Update summary to mention `SetOriginalFilename` alongside `SetMovie` and `SetException`. | WP-005 Reviewer |

---

## Strategic Recommendations (Gold Nuggets)

The following items were surfaced across pipelines but are out of scope for this plan. Recommended for a future cleanup cycle:

1. **Make `SharedConnectionFactory` accept `IDbTransaction?`** — would eliminate the residual `SingleConnectionFactory` private class in `SpdbConfigRepositoryTests`, further consolidating test infrastructure. Low effort, zero production-code risk.

2. **Introduce `IProcessLauncherService` abstraction** — `WindowsFileLauncherService` cannot be unit-tested because `Process.Start` calls cannot be intercepted. A thin abstraction would enable test coverage for the launch logic and the silent-failure scenario the Security Auditor flagged.

3. **Add CleanLabel discard-path test** — the `if (result is not null)` guard in `CleanLabel` is correct, but no test asserts that `SaveAsync` is *not* called when the dialog is cancelled. A single test closes this regression gap.

4. **`LabelCleanerViewModel._keywords` mutability** — currently a private `IReadOnlyList<string>`. If a keyword editor is ever added to the Label Cleaner dialog, this field will need to become an `ObservableCollection<string>`-backed property. Worth noting before any dialog extension work begins.

5. **`IntDecimalConverter.ConvertBack` non-decimal non-null input** — minor gap: the `value is not decimal d` guard handles non-null non-decimal values (e.g., a string) but no test explicitly covers that path. Negligible in practice since `NumericUpDown` only produces `decimal?`.

---

## Next Steps for Planner / PM

1. **Assign Documentation agent** to address all five `D-*` items above before the next implementation cycle begins. `D-3` (README stale reference) is the highest-priority because it contains incorrect factual claims about a removed database constraint.

2. **Consider Milestone 7 planning**: With all M6 carry-forwards resolved and version 1.1.0 shipped, the codebase is in a clean state. The Planner should review remaining TODO items in `src/VideoIndexer.Infrastructure/` and `src/VideoIndexer.App/` for the next milestone scope.

3. **Live-DB integration tests**: The 6 skipped tests require a configured `test-config.json`. Consider documenting setup steps more prominently or automating the configuration in CI to prevent them from remaining perpetually skipped.

---

*Generated by Synthesis Agent — 2026-05-14*
