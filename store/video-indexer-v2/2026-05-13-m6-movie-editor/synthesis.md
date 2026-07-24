# Synthesis Report — M6: Movie Editor

**Project:** `2026-05-13-m6-movie-editor`
**Date:** 2026-05-14
**Status:** COMPLETE
**Work Packages:** 11 / 11 COMPLETE — 0 BLOCKED, 0 FAILED

---

## Executive Summary

Milestone 6 delivered the **Movie Editor** — the central workspace for viewing and editing a single movie's metadata, review status, and utilities. The editor opens as an inline navigation target (replacing the movies list in `MainContentView`), following the established `ViewLocator` pattern used by `LibraryFoldersView`. Users can open the editor via double-click or the new context menu on the movies grid.

The full M6 feature surface is live:

- **Three-column layout** (`MovieEditorView`) with an editable left sidebar (Label, ActorNames, Studio, Description, Rating, Year, Season, Episode, ViewCount), stub center tabs (Cover Image, Thumbnails, Video — M9/M10), and a stub right sidebar (Tagging — M7).
- **Full change-tracking VM** (`MovieEditorViewModel`) with Apply Changes, Save & Exit, Discard Changes, and Close guards.
- **Review-status management** — Flag for Review with collapsible message panel, Mark Reviewed.
- **View-counter reset.**
- **Label Cleaner dialog** (`LabelCleanerView`) — filename → metadata parser using the new `NameParser`, with live-updating toggles (PreserveDates, UCWords, StripKeywords).
- **Movie Properties dialog** (`MoviePropertiesView`) — read-only technical metadata with Open Data Folder / Show in Folder file-shell integration; ffprobe-dependent fields deferred to M9.
- **Database schema revision 38** — `year`, `season`, `episode` columns added via idempotent `ADD COLUMN IF NOT EXISTS` migration.
- **Carry-forward items resolved:** MoviesListView CancellationToken lifecycle (M5 debt), FiltersManagerView AXAML cosmetic fix, and the M5 FilterExpressionEvaluator test-count annotation.

---

## Work Package Summary

| WP | Title | Stages | Tests (Pass/Fail) |
|---|---|---|---|
| WP-001 | MoviesListView — CTS lifecycle & error handling | impl → qa → review → docs | 517 / 0 |
| WP-002 | Schema migration m038 (year/season/episode) | impl → qa → review → docs | 517 / 0 |
| WP-003 | INameParser / NameParser / LabelCleaner Core | impl → qa → sec → review → docs | 537 / 0 |
| WP-004 | Cosmetic / manifest cleanup | impl → qa → review → docs | 517 / 0 |
| WP-005 | Movie model, IMovieRepository, DapperMovieRepository | impl → qa → sec → review → docs | 537 / 0 |
| WP-006 | MovieEditorViewModel (32 tests) | impl → qa → review → docs | 575 / 0 |
| WP-007 | MovieEditorView AXAML + IntDecimalConverter | impl → qa → review → docs | 573 / 0 |
| WP-008 | Movie Properties dialog (27 tests) | impl → qa → sec → review → docs | 600 / 0 |
| WP-009 | Navigation wiring (double-click, ContextMenu, reload) | impl → qa → review → docs | 573 / 0 |
| WP-010 | Label Cleaner dialog (12 tests) | impl → qa → review → docs | 613 / 0 |
| WP-011 | Final manifest sweep (data-flows § 9, constraints) | docs | — |

All 11 WPs achieved full pipeline PASS with zero test failures or build warnings.

---

## Metrics

| Metric | Value |
|---|---|
| **Final test count** | 613 passed, 0 failed, 6 skipped (env-conditional) |
| **Tests added this milestone** | ~113 net new tests |
| **Build warnings** | 0 (TreatWarningsAsErrors=true enforced throughout) |
| **Security audits completed** | 3 (WP-003, WP-005, WP-008) |
| **Security issues (Critical / High / Medium)** | 0 / 0 / 0 |
| **Security issues (Low)** | 4 Low across all audits |
| **Reviewer Fix-Forward applied** | 6 (see detail below) |
| **Schema revision** | 37 → **38** |
| **New source files** | ~28 new files across Core / Infrastructure / App / Tests layers |

### New Test Suites

| File | Tests |
|---|---|
| `LabelCleanerTests.cs` | 17 (Core layer, NameParser algorithm) |
| `DapperMovieRepositoryTokenisationTests.cs` | 9 (Infrastructure, TokeniseActorNames) |
| `MovieEditorViewModelTests.cs` | 32 (App VM, change-tracking + commands) |
| `MoviePropertiesViewModelTests.cs` | 27 (App VM, file-size formatting + init) |
| `MainContentViewModelTests.cs` (4 new) | 4 (navigation: open, close, null-repo guard, reload) |
| `LabelCleanerViewModelTests.cs` | 12 (App VM, option toggles + Accept/Discard) |

---

## Reviewer Fix-Forwards Applied

| WP | Fix Applied |
|---|---|
| WP-002 | `DatabaseBootstrapperTests.cs` — hardcoded revision string literals replaced with `DatabaseBootstrapper.ExpectedRevision` constant arithmetic; sentinel test intentionally left hardcoded with explanatory comment. |
| WP-003 | `NameParser.cs` — three date-pattern regexes hoisted to `static readonly (Regex CompiledRegex, string Format)[]` with `RegexOptions.Compiled`, eliminating per-call allocation (flagged by Security Auditor). |
| WP-006 | `MovieEditorViewModel.cs` — misleading XML doc on the parameterless constructor corrected to enumerate only the persistence-requiring no-op commands. |
| WP-007 | `MovieEditorView.axaml` — `IsEnabled='False'` added to MoveActorUp/Down stub buttons (no `ListBox` wired yet; buttons were always no-ops but appeared enabled). |
| WP-008 | `Program.cs` — triplicated `ownerFactory` lambda for `ShowDialog` hoisted to a single shared local, eliminating 3× duplication across service registrations. |
| WP-010 | `LabelCleanerViewModel.cs` — explanatory comment added above `_uCWords` backing field clarifying the non-standard casing required by CommunityToolkit.Mvvm's source generator to produce the `UCWords` property name. |

---

## Security Audit Summary

Three WPs with new Core / Infrastructure / Service logic received dedicated security-audit pipelines.

- **WP-003 (NameParser):** All caller-supplied terms pass through `Regex.Escape` before pattern construction — regex injection fully mitigated. Date patterns are static constants. No external network calls. 0 Critical/High/Medium.
- **WP-005 (DapperMovieRepository):** All 7 query methods use Dapper parameterized syntax — no string-concatenated SQL. `SaveAsync` intentionally excludes `review`, `review_message`, and `watch_count` columns, preventing accidental state overwrite. 0 Critical/High/Medium.
- **WP-008 (WindowsFileLauncherService):** `ShowInExplorer` uses direct path interpolation into `explorer.exe /select,"{path}"` arguments. `UseShellExecute=true` targets explorer directly (no shell interpreter), so there is no code-execution path; worst case is explorer navigating to the wrong location. Exploitability requires prior DB write access (higher-severity breach). Flagged as Low — remediation via `Path.GetFullPath()` or `ProcessStartInfo.ArgumentList` recommended.

---

## Strategic Recommendations (Gold Nuggets)

### 1. `LoadFilterSlotsAsync` — Missing internal error handling (Medium)
`MoviesListViewModel.LoadFilterSlotsAsync` has no internal `try/catch`. The view's `Loaded` handler now guards this call site, but any future caller would receive unguarded exceptions. Mirror the try/catch pattern used by `LoadAsync` internally to make `LoadFilterSlotsAsync` robust regardless of its call site.

### 2. `DatabaseBootstrapperTests` — Sentinel test maintenance contract (Medium)
`Assert.Equal(38, DatabaseBootstrapper.ExpectedRevision)` at line 167 is an intentional migration tripwire that **must be manually updated** on each schema revision bump. The explanatory comment (added by Documentation in WP-002) is in place. Ensure this pattern is captured in onboarding notes for new contributors.

### 3. `SharedConnectionFactory` / `NonDisposingConnection` duplication (Low)
This test-infrastructure helper is duplicated verbatim in four files: `DapperMovieCatalogRepositoryTests`, `DapperFilterSlotRepositoryTests`, `LibraryScannerIntegrationTests`, and `DapperMovieRepositoryTests`. Consolidating into a shared base class or static helper in `VideoIndexer.Infrastructure.Tests` would reduce future maintenance cost.

### 4. `GetOriginalFilenameAsync` — Non-deterministic LIMIT 1 (Low)
`DapperMovieRepository.GetOriginalFilenameAsync` uses `LIMIT 1` with no `ORDER BY` because `movies_filenames` has no insertion-timestamp column. If multiple filenames exist for a movie, the returned name is non-deterministic. Adding a `created_at` timestamp column to `movies_filenames` in a future migration would enable deterministic ordering.

### 5. `OnEditorCloseRequested` — Missing CurrentPage guard (Low)
If a library refresh transitions `CurrentPage` away from `MovieEditor` before the editor's `CloseRequested` fires, `OnEditorCloseRequested` will call `CloseSubView()` and interrupt the refresh. Adding a `CurrentPage == MainContentPage.MovieEditor` guard at the top of the handler would make the close path safe under concurrent state transitions.

### 6. `WindowsFileLauncherService` — Path argument quoting (Low, Security)
`OpenFolder` does not quote the path argument passed to `explorer.exe`. Safe today because paths are hash-derived (no metacharacters), but wrapping in double-quotes would guard against any future `IAppPaths` implementation that returns user-facing paths with spaces. Prefer `ProcessStartInfo.ArgumentList` collection syntax in a future cleanup pass.

### 7. `LabelCleanerOptions` persistence not wired (Low)
`MovieEditorViewModel.CleanLabelCommand` always constructs `LabelCleanerOptions` with defaults, ignoring `AppOptions.LabelCleaner`. If persisted toggle preferences are desired (user sets PreserveDates once, it stays), inject `ISettingsService` and read `settings.Current.LabelCleaner` as the initial options in the Label Cleaner dialog.

### 8. `FakeMovieRepository.GetOriginalFilenameAsync` always returns null (Low)
`CleanLabelCommand` tests only exercise the `LabelText`-fallback path. Adding a `SetOriginalFilename(string?)` configuration method to `FakeMovieRepository` would enable tests for the primary code path (filename from DB → `Path.GetFileNameWithoutExtension`).

### 9. `IntDecimalConverter` — No isolated unit tests (Low)
The converter is exercised indirectly by compilation but lacks dedicated `xUnit` tests for `ConvertBack` null and round-trip scenarios. Low risk (NumericUpDown min/max constraints prevent out-of-range values), but recommended before the converter is reused for other numeric fields.

### 10. `MoveActorDown` null-guard test missing (Low)
`MoveActorDown_IsNoOp_WhenNoActorSelected` has no counterpart in `MovieEditorViewModelTests.cs` (the parallel `MoveActorUp` null test exists). Low risk — both methods share the same null guard — but the asymmetric coverage should be resolved when the test file is next touched.

---

## Deferred Stubs (Planned Future Milestones)

| Stub | Target Milestone |
|---|---|
| Cover Image tab (center panel) | M8 |
| Thumbnails tab (center panel) | M9 |
| Video tab + Bookmarks (center panel) | M10 |
| Tagging right sidebar | M7 |
| ffprobe-dependent fields in MoviePropertiesViewModel (FileType, Resolution, Duration, Bitrate) | M9 |
| MoveActorUp/Down (requires ListBox + split-string converter in `MovieEditorView`) | Future |

---

## Next Steps for Planner / Project Manager

1. **M7 — Tagging control** is the next natural dependency (right sidebar stub in MovieEditorView is already reserved).
2. **M8 — Cover Image** can proceed in parallel with M7; the center tab is already stubbed.
3. **Actor ListBox** (replacing the plain `TextBox` for `ActorNamesText`) should be scoped — either as part of M7 (if the tagging UX is closely related) or as a standalone polish WP. The MoveActorUp/Down buttons are `IsEnabled='False'` until this is wired.
4. **`LoadFilterSlotsAsync` internal error handling** (Recommendation #1) is low-effort and high-safety — good candidate for inclusion as a carry-forward item in the next milestone.
5. **`SharedConnectionFactory` consolidation** (Recommendation #3) can be addressed as a test-infrastructure cleanup WP at the start of any milestone that adds new integration tests.
