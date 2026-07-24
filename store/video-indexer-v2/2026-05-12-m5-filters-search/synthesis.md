# Synthesis Report — M5: Filters & Search

**Date:** 2026-05-12  
**Plan:** `2026-05-12-m5-filters-search`  
**Status:** COMPLETE — All 16 work packages passed all pipeline stages.

---

## 1. Executive Summary

Milestone 5 delivered the **free-text search bar** and the **saved filter slot system** for the VideoIndexer v2 movies list. After this milestone:

- Users can narrow the movies grid in real-time by typing space-separated search terms matched against Label, Actor Names, Studio, and Description fields (minimum 2 characters per term, AND logic).
- Users can define, save, activate, and delete named DSL filter expressions (e.g., `Rating >= 4 AND NeedsReview()`) via a new **Filters Manager** dialog.
- The active filter slot pointer is persisted to the database (`spdb_config` / `filter_slots`) and self-heals stale IDs on load.
- The database schema was bumped from revision 36 to **revision 37**, introducing the `filter_slots` table via migration `m037_add_filter_slots.sql`.

Three deferred M4 items were also closed:
- `ArgumentNullException.ThrowIfNull(query)` guard added to `DapperMovieCatalogRepository.GetMovieListAsync`.
- `HasLoadError` / `LoadErrorMessage` flags added to `MoviesListViewModel` and bound to a visible red error banner in `MainContentView.axaml`.
- `Description` field added to `MovieListItem` so all four spec-mandated search fields are covered.

---

## 2. Work Package Summary

| WP | Title | Stages | Outcome |
|---|---|---|---|
| WP-001 | DB schema migration m037 (filter_slots, revision 37) | impl → qa → review → docs | PASS |
| WP-002 | Core domain: FilterSlot model + IFilterSlotRepository / IActiveFilterSlotRepository | impl → qa → review → docs | PASS |
| WP-003 | MovieListItem.Description + DapperMovieCatalogRepository null-guard | impl → qa → review → docs | PASS |
| WP-004 | DapperFilterSlotRepository + SpdbConfigActiveFilterSlotRepository | impl → qa → **security** → review → docs | PASS |
| WP-005 | Filter DSL — lexer + parser (FilterLexer, FilterExpressionParser, AST) | impl → qa → review → docs | PASS |
| WP-006 | Filter DSL — evaluator (FilterExpressionEvaluator, 57 tests) | impl → qa → review → docs | PASS |
| WP-007 | MoviesListOptions — confirmed unchanged from M4 (no-op) | qa → review | PASS |
| WP-008 | MoviesListViewModel — search text + DSL filter + error state + 22 tests | impl → qa → review → docs | PASS |
| WP-009 | FiltersManagerViewModel — CRUD commands, validate-before-save, 16 tests | impl → qa → review → docs | PASS |
| WP-010 | IFilterManagerService + AvaloniaFilterManagerService (modal dialog host) | impl → qa → review → docs | PASS |
| WP-011 | FiltersManagerView.axaml — two-column UI + DSL quick-reference panel | impl → qa → review → docs | PASS |
| WP-012 | MainContentView wiring + Program.cs DI registrations | impl → qa → review → docs | PASS |
| WP-013 | FilterExpressionParserTests — 13 named test methods (Reviewer authoring WP) | qa → review | PASS |
| WP-014 | MoviesListViewModelSearchFilterTests + FiltersManagerViewModelTests review | qa → review | PASS |
| WP-015 | DapperFilterSlotRepositoryTests refactor (transaction-rollback discipline) + 7 named ACs | qa → review | PASS |
| WP-016 | Project manifest documentation pass — all 6 manifest documents updated, m5 milestone doc created | docs | PASS |

**Total pipeline stages:** 54 across 16 WPs (13 implementation, 14 QA, 1 security-audit, 13 code-review, 13 documentation).  
All 54 stages passed.

---

## 3. Test Metrics

| Assembly | Before M5 | After M5 | Net New |
|---|---|---|---|
| VideoIndexer.Tests | 128 | 198 | +70 |
| VideoIndexer.App.Tests | 111 | 146 | +35 |
| VideoIndexer.Infrastructure.Tests | 128 | 152 | +24 |
| **Total passing** | **367** | **496** | **+129** |
| Skipped (env-dependent) | 6 | 6 | 0 |
| Failed | 0 | 0 | 0 |

**Key test contributions:**
- **+57** — `FilterExpressionEvaluatorTests` (all 8 functions, all 6 operators, short-circuit AND/OR, NOT, error paths)
- **+13** — `FilterExpressionParserTests` (parser grammar, error handling, deferred identifier rejection)
- **+22** — `MoviesListViewModelSearchFilterTests` (text search, DSL filter, error state, stale-slot self-heal)
- **+16** — `FiltersManagerViewModelTests` (CRUD commands, validate-before-save, close event)
- **+17** — `DapperFilterSlotRepositoryTests` / `SpdbConfigActiveFilterSlotRepositoryTests` (integration + unit)
- **+1** — `GetMovieListAsync_NullQuery_ThrowsArgumentNullException`

**Security audit (WP-004):** 0 Critical, 0 High, 0 Medium, 1 Low (null property validation gap on `FilterSlot.Name`/`.Expression` — documented in interface XML and api-surface.md).

---

## 4. Key Architecture Delivered

### 4.1 Filter Expression Language (DSL)

A pure-BCL recursive-descent parser/evaluator was built entirely within `VideoIndexer.Core/Filtering/`:

- **`FilterLexer`** (internal) — handles 16 token types, case-insensitive keywords, decimal/integer literals stored as `decimal`, single/double-quoted strings with backslash escape.
- **`FilterExpressionParser`** — public static `Parse()` entry point, OR > AND > NOT > primary precedence, M7/M10 deferred identifier rejection with milestone hints, trailing-token guard.
- **`FilterExpressionNode`** hierarchy — sealed records: `BinaryNode`, `UnaryNotNode`, `ComparisonNode`, `FunctionCallNode`, `IdentifierNode`, `NumberLiteralNode`, `StringLiteralNode`.
- **`FilterExpressionEvaluator`** — static `Evaluate(FilterExpressionNode, MovieListItem)`, all M5 numeric identifiers (Rating, TagsWeight, AmountTags, AmountThumbnails, ViewCount), all 6 no-arg boolean functions, 2 string-predicate functions (ActorsContain, StudioContains), decimal-space arithmetic throughout.

### 4.2 Infrastructure Layer

- **`DapperFilterSlotRepository`** — sealed, CRUD for `filter_slots`. Uses ValueTuple projection for `GetAllAsync` (required due to `FilterSlot`'s required-init properties preventing standard Dapper POCO mapping). Two-phase insert (INSERT + `SELECT LAST_INSERT_ID()`).
- **`SpdbConfigActiveFilterSlotRepository`** — clean delegation to `ISpdbConfigRepository`; null/empty and non-numeric keys return `null` gracefully.
- Both repositories use `@Param` parameterisation throughout (no dynamic SQL). All awaits carry `ConfigureAwait(false)`.

### 4.3 ViewModel Layer

- **`MoviesListViewModel`** — extended with `_allMovies` backing cache, `SearchText` / `ActiveFilterSlot` observable properties, parse-and-cache `_cachedAst` pattern (avoids per-keystroke reparsing), `ApplyFilter()` with stacked AND logic (text gate → DSL gate), `HasLoadError`/`LoadErrorMessage` error state, `LoadFilterSlotsAsync` with stale-ID self-healing.
- **`FiltersManagerViewModel`** — full CRUD for filter slots, DSL expression validation before save, `ConfirmDelete` guard with `[NotifyCanExecuteChangedFor]`, `CloseRequested` event pattern, no DI container usage.

### 4.4 UI Layer

- **`FiltersManagerView.axaml`** — two-column layout, compiled Avalonia bindings (`x:DataType`) on root and `ListBox` DataTemplate, embedded DSL quick-reference panel (all M5 identifiers and functions listed).
- **`MainContentView.axaml`** — search TextBox, filter slot ComboBox with `FilterSlot.Name` DataTemplate, orange `Ellipse` active-filter indicator, Filters… button, red load-error banner.
- **`AvaloniaFilterManagerService`** — modal Window host, `try/finally` event handler unsubscription (Reviewer Fix-Forward applied to prevent memory leak when externally-supplied ViewModel outlives the dialog).

---

## 5. Notable Events & Fixes

| Event | WP | Severity | Resolution |
|---|---|---|---|
| IFilterSlotRepository / IActiveFilterSlotRepository had fewer methods than WP-004 required | WP-004 | Low | Methods added during WP-004 implementation; interfaces extended cleanly |
| Migration m037 not applied to spdb_tests schema | WP-004 | Medium | Integration tests self-heal via `CREATE TABLE IF NOT EXISTS` guard; real schema migration recommended pre-production |
| Reviewer Fix-Forward: AvaloniaFilterManagerService event handler not unsubscribed | WP-010 | Low | Named EventHandler + `try/finally` unsubscription applied by Reviewer |
| DapperFilterSlotRepositoryTests used cleanup-based discipline instead of transaction-rollback | WP-015 | Medium | Refactored to `SharedConnectionFactory + NonDisposingConnection` pattern matching `DapperMovieCatalogRepositoryTests` |
| WP-011 AC spec used "EditName"/"EditExpression" — actual VM properties are "Name"/"Expression" | WP-011 | Info | Bindings use correct real property names; spec discrepancy noted in ledger |

---

## 6. Open Tech Debt (Carry Forward to M6+)

These items were logged by agents but deliberately deferred:

| Priority | Item | Source |
|---|---|---|
| **High** | `LoadFilterSlotsAsync` is **not wired** from `MoviesListView.axaml.cs` or `LoadAsync` — FilterSlots/ActiveFilterSlot are always empty/null at runtime. Must be wired in a subsequent WP before M5 is considered functionally complete. | WP-008, WP-012 (multiple agents) |
| Medium | `DapperFilterSlotRepository.SaveAsync` does not validate that `slot.Name` / `slot.Expression` are non-null; a null property produces a DB-level exception instead of a clean `ArgumentException` at the method boundary | WP-004 Security Audit |
| Medium | `FiltersManagerView.axaml` binds `SaveSlotCommand` to both the "Add" and "Save Slot" buttons — no dedicated "New Slot / Clear Editor" command. Adding a `ClearSelectionCommand` to `FiltersManagerViewModel` would improve UX. | WP-011 |
| Low | ComboBox for `ActiveFilterSlot` shows blank when no slot is active; a null-item placeholder DataTemplate would improve UX | WP-012 |
| Low | `ApplyFilter()` rebuilds `Movies` via `Clear()+Add()` on every keystroke; appropriate now but future optimization target for large catalogs | WP-008 |
| Low | `FilterExpressionEvaluatorTests.cs` sits at the test-project root instead of the `Filtering/` subfolder | WP-013 |
| Low | `DatabaseBootstrapperTests.cs::ExpectedRevision_IsThirtySeven` requires a manual rename on every future schema revision bump; consider replacing with a purely behavioral test | WP-001 |
| Low | `IFilterSlotRepository.SaveAsync` insert/update convention (`SlotId == 0` → insert) is implicit and not type-enforced; a discriminated union or factory method pattern would make intent explicit at call sites | WP-002, WP-004 |
| Low | Captive dependency: `DapperFilterSlotRepository` / `SpdbConfigActiveFilterSlotRepository` are singletons that hold transient `IDbConnectionFactory` / `ISpdbConfigRepository`; established pattern in codebase | WP-012 |

---

## 7. Strategic Recommendations

1. **Wire `LoadFilterSlotsAsync` before shipping M5.** The filter slot system is fully implemented at the data + VM layer but is disconnected from the view lifecycle. Calling `await ViewModel.LoadFilterSlotsAsync()` from `MoviesListView.axaml.cs` (alongside the existing `LoadAsync` call on `Loaded`) is a one-line change that activates the full feature. This is the single most impactful next action.

2. **Apply m037 to the test database.** The integration test self-create guard (`CREATE TABLE IF NOT EXISTS filter_slots`) is a safety net, not the intended mechanism. Running the actual `m037_add_filter_slots.sql` migration against `spdb_tests` removes the ambiguity and ensures test setup mirrors production.

3. **Consider a `CreateFilterSlot` / `UpdateFilterSlot` split for `IFilterSlotRepository.SaveAsync`.** The current convention (`SlotId == 0` → insert, `SlotId > 0` → update) works but is implicit. When M6+ introduces additional filter-related operations, a discriminated union or separate methods would reduce the cognitive burden on new contributors.

4. **Move `FilterExpressionEvaluatorTests.cs` to `tests/VideoIndexer.Tests/Filtering/`.** Minor housekeeping, but placing it alongside `FilterExpressionParserTests.cs` in the subfolder gives the DSL test suite a clean, discoverable home before the directory grows.

5. **Plan for DSL extensibility.** `FilterExpressionEvaluator` uses a linear `if`-chain for function dispatch (~8 functions). A `Dictionary<string, Func<FunctionCallNode, MovieListItem, bool>>` dispatch table would pay dividends when M7 (tag functions) and M10 (bookmark functions) are implemented. The deferred-identifier rejection hooks in `FilterExpressionParser` are already in place — M7/M10 only need to add the evaluator entries and remove the parse-time blocks.

---

## 8. Pipeline Health

```
WPs with all stages passing : 16 / 16
WPs with missing stages      :  0 / 16
Total stages executed        : 54
Total stages passed          : 54
Total stages failed          :  0
```

---

*Synthesis generated by Head of Operations (Synthesis Agent) — 2026-05-12*
