# Synthesis Report — M4: Movies List
**Plan:** `2026-05-11-m4-movies-list`
**Date:** 2026-05-12
**Status:** COMPLETE — 12/12 work packages delivered

---

## Executive Summary

Milestone 4 delivered a fully functional, read-only Movies List screen for Video Indexer v2. The work spans every layer of the architecture: a database schema migration, Core domain models, a repository query, a ViewModel with settings persistence, an Avalonia DataGrid view, DI wiring, and a full test suite. The application now launches directly into the movies grid after the library refresh completes, with column visibility toggling, a cover-panel toggle, and a multi-select context menu scaffold ready for M5 implementation.

All 12 work packages passed all pipeline stages. One rework cycle was triggered (WP-008: missing `IsReadOnly="True"` on DataGrid root). Security audit on the repository layer (WP-005) returned zero Critical/High/Medium findings. The final test suite stands at **364 passing, 0 failing, 6 skipped** (5 pre-existing environment skips + 1 environment-conditional integration test).

---

## Deliverables by Work Package

| WP | Title | Pipeline Stages | Tests |
|---|---|---|---|
| WP-001 | DB migration: `file_created` column + schema revision 36 | impl → qa → review → docs | 352 ✓ |
| WP-002 | NuGet: `Avalonia.Controls.DataGrid` 11.3.13 | impl → qa → review → docs | 352 ✓ |
| WP-003 | Core domain: `MovieListItem`, `MovieListQuery`, `MoviesListOptions` | impl → qa → review → docs | 352 ✓ |
| WP-004 | Interface: `IMovieCatalogRepository.GetMovieListAsync` | impl → qa → review → docs | — (compile-break check) |
| WP-005 | Repository: `DapperMovieCatalogRepository.GetMovieListAsync` | impl → qa → **security** → review → docs | 352 ✓ |
| WP-006 | Test fakes: `FakeMovieCatalogRepository`, `InMemoryMovieCatalogRepository` | impl → qa → review → docs | 352 ✓ |
| WP-007 | ViewModel: `MoviesListViewModel` | impl → qa → review → docs | 352 ✓ |
| WP-008 | View: `MoviesListView.axaml` / code-behind | impl → qa → **rework** → review → docs | 364 ✓ |
| WP-009 | DI wiring: `MainContentViewModel` default page + `Program.cs` | impl → qa → review → docs | 364 ✓ |
| WP-010 | VM unit tests: `MoviesListViewModelTests` (9 tests) | qa → review | 9 new ✓ |
| WP-011 | Integration tests: `GetMovieListAsync` (4 DB tests) | qa → review | 3 pass / 1 env-skip |
| WP-012 | Final manifest sweep | docs | — |

---

## Metrics

| Metric | Value |
|---|---|
| Work packages | 12 / 12 COMPLETE |
| Pipeline stages passed | 49 (all active stages across all WPs) |
| Rework cycles | 1 (WP-008: DataGrid `IsReadOnly` missing) |
| Final test count | 364 passed, 0 failed, 6 skipped |
| New unit tests | 9 (`MoviesListViewModelTests`) |
| New integration tests | 4 (`GetMovieListAsync` DB roundtrip) |
| Security findings | 0 Critical / 0 High / 0 Medium / 1 Low advisory |
| Build warnings | 0 (enforced: `TreatWarningsAsErrors=true`) |
| Schema revision | 35 → **36** (`file_created DATETIME NULL`) |
| Reviewer Fix-Forwards applied | 7 (across WP-002, WP-005, WP-007, WP-008, WP-009) |

---

## Key Artifacts

**New source files:**
- `src/VideoIndexer.Infrastructure/Database/migrations/m036_add_file_created.sql`
- `src/VideoIndexer.Core/Models/MovieListItem.cs` — 14-property sealed record
- `src/VideoIndexer.Core/Models/MovieListQuery.cs` — empty sealed record (M5 extension hook)
- `src/VideoIndexer.Core/Options/MoviesListOptions.cs` — with nested `ColumnKeys` static class
- `src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs`
- `src/VideoIndexer.App/Views/MoviesListView.axaml` + `.axaml.cs`

**Modified source files:**
- `src/VideoIndexer.Core/Options/AppOptions.cs` — added `MoviesList` property
- `src/VideoIndexer.Core/Abstractions/IMovieCatalogRepository.cs` — added `GetMovieListAsync`
- `src/VideoIndexer.Infrastructure/Database/DatabaseBootstrapper.cs` — `ExpectedRevision = 36`
- `src/VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs` — SQL query impl
- `src/VideoIndexer.App/ViewModels/MainContentViewModel.cs` — `MoviesListVm` default page
- `src/VideoIndexer.App/Views/MainContentView.axaml` — search stub (M5)
- `src/VideoIndexer.App/Program.cs` — DI registrations

**New test infrastructure:**
- `tests/VideoIndexer.App.Tests/MoviesListViewModelTests.cs` (9 tests)
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeSettingsService.cs`
- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperMovieCatalogRepositoryTests.cs` (4 added)

---

## Strategic Recommendations ("Gold Nuggets")

1. **`MovieListQuery` empty-record pattern** (WP-004, Reviewer) — Declaring `MovieListQuery` as an intentionally empty sealed record, with XML documentation explaining the M4/M5 boundary, is excellent forward-compatible API design. Adding filter predicates in M5 requires only adding properties to `MovieListQuery`; the `IMovieCatalogRepository` interface signature remains unchanged.

2. **`ColumnKeys` nested static class** (WP-003, Reviewer) — Single source of truth for all 12 column key strings. Prevents magic-string drift between C# code, AXAML bindings, and `appsettings.json`. Adopt this pattern for any future settings dictionary that uses string keys.

3. **`Interlocked.CompareExchange` single-flight gate** (WP-007) — `LoadAsync` uses `Interlocked.CompareExchange(ref _loadingGuard, 1, 0)` to silently discard concurrent load invocations without locks or `SemaphoreSlim`. The guard is released in `finally`, making it exception-safe. This pattern should be standardized across all ViewModels that expose a `LoadAsync`.

4. **`TaskCompletionSource.RunContinuationsAsynchronously` for concurrency tests** (WP-010, Reviewer) — The concurrent-load test uses a TCS gate with `RunContinuationsAsynchronously` to test the Interlocked guard without `Thread.Sleep` or `Task.Delay`. This is the correct, deterministic pattern for async concurrency tests — carry it forward to any future concurrency scenarios.

5. **`ReflectionBinding` for DataGrid column header ContextMenu** (WP-008) — DataGrid column header `DataTemplate`s sit outside the compiled-binding scope in Avalonia 11. The `{ReflectionBinding ElementName=LayoutRoot, Path=DataContext.SetColumnVisibilityCommand}` pattern is the correct idiomatic workaround. Document this as the standard approach for all future DataGrid column header command bindings.

---

## Blockers and Failures

| WP | Issue | Severity | Resolution |
|---|---|---|---|
| WP-008 | `<DataGrid>` missing `IsReadOnly="True"` — all 9 `DataGridTextColumn` cells were double-click editable despite having no edit/save path | High | Fixed in rework implementation; confirmed by rework QA and code-review |

---

## Security Summary

WP-005 underwent a dedicated security audit across all OWASP Top 10 categories.

- **Result:** PASS — 0 Critical, 0 High, 0 Medium
- **Finding (Low — advisory):** When M5 adds filter/search predicates to `MovieListQuery`, all new fields **must** be incorporated into the SQL exclusively via Dapper `@Param` parameterized variables. Any inline string concatenation or interpolation into `MovieListSql` would introduce a SQL injection vector (A03).

No other security issues were identified. `MovieListSql` is a compile-time `const` with zero runtime interpolation.

---

## Technical Debt and Deferred Items

| Item | Owner | Target |
|---|---|---|
| `GetMovieListAsync` `query` parameter null-guard (`ArgumentNullException.ThrowIfNull`) | M5 Developer | When `MovieListQuery` gains filter fields |
| `GetMovieListAsync` XML summary "paged or filtered" wording | M5 Documentation | When M5 implements filtering semantics |
| Migration `m036_add_file_created.sql` not idempotent (no `IF NOT EXISTS` on `ADD COLUMN`) | Ops | Documented; acceptable for one-shot manual migration |
| No transaction wrapper on `ALTER TABLE + UPDATE` in m036 | Ops | Rollback procedure documented in `constraints.md` and migration file header |
| Column widths hardcoded pixel values in `MoviesListView.axaml` | Future milestone | Persist in new `MoviesListOptions.ColumnWidths` dictionary |
| `HasCoverImage` hardcoded `0` in SQL | M9 Developer | Populate from filesystem cover-image discovery |
| `InMemoryMovieCatalogRepository` / `FakeMovieCatalogRepository` duplication | Future refactor | Consolidate test fakes (low priority, out of M4 scope) |
| `IReadOnlyDictionary<string,bool>` declared type vs `Dictionary<string,bool>` concrete initializer in `MoviesListOptions.ColumnVisibility` | Low | Cosmetic inconsistency; binder replaces it regardless |
| `Constructor_ChildViewModels_AreAssigned` assertion gap for `MoviesListVm` | Already resolved | Fixed by Reviewer Fix-Forward in WP-009 |

---

## Manifest Updates Delivered

All required manifest documents were updated as part of this milestone:

| Document | Updates |
|---|---|
| `constraints.md` | Schema revision 36, migration rollback procedure, `MainContentPage.Default = MoviesListVm` invariant |
| `tech-stack.md` | Schema revision 36, `Avalonia.Controls.DataGrid 11.3.13` |
| `api-surface.md` | `IMovieCatalogRepository.GetMovieListAsync`, `MovieListItem`, `MovieListQuery`, `MoviesListOptions`, `MoviesListViewModel`, `MoviesListView`, `MainContentViewModel` constructor + `MoviesListVm` property |
| `file-tree.md` | `migrations/` directory, `MovieListItem.cs`, `MovieListQuery.cs`, `MoviesListOptions.cs`, `MoviesListViewModel.cs`, `MoviesListView.axaml`, `.axaml.cs` |
| `README.md` | `MoviesList` configuration section (ShowCoverPanel, ColumnVisibility, 12 column keys) |
| `m4-movies-list.md` | Status → Complete |

---

## Next Steps for M5

1. **Filter & search:** Populate `MovieListQuery` with filter/search predicates. Add `ArgumentNullException.ThrowIfNull(query)` to `DapperMovieCatalogRepository.GetMovieListAsync`. Implement the disabled search `TextBox` in `MainContentView.axaml` (stub already placed).
2. **Context menu commands:** Implement the six no-op stubs in `MoviesListViewModel` (Play, PlayFullscreen, Edit, GenerateThumbnails, CopyTo, DeleteOnDisk).
3. **Column width persistence:** Add `ColumnWidths` dictionary to `MoviesListOptions` and bind to DataGrid column widths.
4. **Error state surface:** Consider adding an `[ObservableProperty] bool HasLoadError` to `MoviesListViewModel.LoadAsync` so the view can display an error message instead of an empty grid on DB failure.
5. **Cover image discovery:** `HasCoverImage` is hardcoded `0` — M9 work item; no M5 action required.
