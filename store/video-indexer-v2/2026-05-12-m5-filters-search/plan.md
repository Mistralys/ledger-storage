# Plan — M5: Filters & Search

## Summary

Milestone 5 completes the interactive movie list by adding a **free-text search bar** and a **saved-filter-slot system** backed by a small domain-specific language (DSL). After M5, the user can narrow the movies grid in real-time by typing search terms against label, actor names, studio, and description fields, and can define, save, and activate named filter expressions (e.g. `Rating >= 4 AND NeedsReview()`). Filter slots are persisted in the database alongside the active-slot pointer (both in `spdb_config` / `filter_slots`), so the active selection is database-scoped and cannot become stale when the user switches connections. Filtering is applied in-memory against the already-loaded movie collection, keeping keystroke response instantaneous without a DB round-trip per change.

M5 also closes three small items deferred from M4: adding `ArgumentNullException.ThrowIfNull(query)` to `DapperMovieCatalogRepository.GetMovieListAsync`, surfacing a `HasLoadError` flag in `MoviesListViewModel` **and binding it to a visible error banner in `MainContentView.axaml`**, and adding a `Description` field to `MovieListItem` so the text search covers all four spec-mandated fields.

---

## Architectural Context

**Key modules touched or introduced:**

| Layer | Module | Role |
|---|---|---|
| Core | `VideoIndexer.Core/Models/MovieListItem.cs` | Add `Description` property |
| Core | `VideoIndexer.Core/Models/MovieListQuery.cs` | Remains empty; null-guard clarified in contract comment |
| Core | `VideoIndexer.Core/Models/FilterSlot.cs` *(new)* | Domain model for a saved filter slot |
| Core | `VideoIndexer.Core/Abstractions/IFilterSlotRepository.cs` *(new)* | CRUD interface for filter slots |
| Core | `VideoIndexer.Core/Filtering/` *(new directory)* | DSL lexer, parser, AST, and evaluator |
| Core | `VideoIndexer.Core/Abstractions/IActiveFilterSlotRepository.cs` *(new)* | Read/write the active filter slot pointer in `spdb_config` |
| Infrastructure | `VideoIndexer.Infrastructure/Library/SpdbConfigActiveFilterSlotRepository.cs` *(new)* | `spdb_config` key/value impl for the active slot pointer |
| Infrastructure | `VideoIndexer.Infrastructure/Database/migrations/m037_add_filter_slots.sql` *(new)* | New `filter_slots` table; bump schema to revision 37 |
| Infrastructure | `VideoIndexer.Infrastructure/Database/DatabaseBootstrapper.cs` | `ExpectedRevision = 37` |
| Infrastructure | `VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs` | Add `Description` to SQL; add null-guard |
| Infrastructure | `VideoIndexer.Infrastructure/Library/DapperFilterSlotRepository.cs` *(new)* | Dapper CRUD for `filter_slots` |
| App | `VideoIndexer.App/Services/IFilterManagerService.cs` *(new)* | Dialog-host service interface |
| App | `VideoIndexer.App/Services/AvaloniaFilterManagerService.cs` *(new)* | Modal window impl |
| App | `VideoIndexer.App/ViewModels/MoviesListViewModel.cs` | Search + filter logic, new deps |
| App | `VideoIndexer.App/ViewModels/FiltersManagerViewModel.cs` *(new)* | Filter slot CRUD + validation VM |
| App | `VideoIndexer.App/Views/FiltersManagerView.axaml` *(new)* | Filter slot editor dialog |
| App | `VideoIndexer.App/Views/MainContentView.axaml` | Enable search stub; add filter slot dropdown + indicator |
| App | `VideoIndexer.App/Program.cs` | Register new services and VMs |

**Existing integration points:**
- `MainContentView.axaml` already has a disabled `TextBox` stub labeled `"Search (M5)"`.
- `MoviesListViewModel.LoadAsync` currently passes `new MovieListQuery()` (empty) to the repository.
- `IConnectionEditorService` / `AvaloniaConnectionEditorService` in `VideoIndexer.App/Services/` is the pattern to follow for the Filter Manager dialog service.
- `DapperMovieCatalogRepository.MovieListSql` is a `const` string — M5 extends it with `m.description`.

---

## Approach / Architecture

### Filtering model: in-memory with cached source

`MoviesListViewModel.LoadAsync` will populate both a private `_allMovies` cache (`IReadOnlyList<MovieListItem>`) **and** the observable `Movies` collection (unchanged from M4 for the baseline case). A new private method `ApplyFilter()` rebuilds `Movies` from `_allMovies` by applying the text search and the active DSL expression in sequence (logical AND per spec §5).

**Why in-memory?** The spec mandates keystroke-level responsiveness for text search. For a personal collection (hundreds to low-thousands of items), an in-memory LINQ pass completes in under a millisecond. No debounce is needed. A DB round-trip per keystroke would require debouncing, an async pipeline, and cancellation handling — much higher complexity for no benefit at this scale.

**`MovieListQuery` stays empty.** The null-guard (`ArgumentNullException.ThrowIfNull(query)`) is added as the M4 tech-debt item, and the XML documentation comment is updated to clarify that M5 filtering is performed by the ViewModel, not the repository. The interface signature is unchanged. (Removing the parameter entirely was considered — see Considered Alternatives — but the test-fixture churn outweighs the cleanup benefit at this stage; the parameter is retained as an extension point for any future server-side query needs.)

### DSL parser/evaluator: placed in Core

A small recursive-descent parser (`FilterExpressionParser`) lives in `VideoIndexer.Core/Filtering/`. The output is an immutable AST. A separate `FilterExpressionEvaluator` walks the AST against a `MovieListItem`. Both are pure C# with no external NuGet dependencies, consistent with the Core constraint.

**Numeric literal type:** `NumberLiteralNode.Value` is `decimal`. The lexer accepts both integer (`5`) and decimal (`5.0`) forms per spec §3.1; both are stored as `decimal`. At evaluation time, integer `MovieListItem` fields (`Rating`, `TagWeight`, `TagCount`, `ThumbnailCount`, `ViewCount`) are promoted to `decimal` before comparison. This avoids `double`-equality fragility for `=`/`!=`, gives exact round-trip semantics for any literal a user can type, and keeps the comparison rules uniform across the supported numeric grammar.

**Supported DSL scope in M5** (all properties/functions evaluable from `MovieListItem` fields):

| Category | Identifiers |
|---|---|
| Numeric properties | `Rating`, `TagsWeight`, `AmountTags`, `AmountThumbnails`, `ViewCount` |
| Boolean functions (no args) | `HasBookmarks()`, `HasCoverImage()`, `HasThumbnails()`, `HasRating()`, `NeedsReview()`, `HasTags()` |
| String-predicate functions | `ActorsContain(text)`, `StudioContains(text)` |
| Operators | `AND`, `OR`, `NOT`, `=`, `!=`, `<`, `>`, `<=`, `>=`, `(` `)` |

**Deferred DSL identifiers** (M7 / M10 prerequisites; parser recognises them but rejects at validation time with a clear message):

| Identifier | Deferred to |
|---|---|
| `HasTag(id)`, `TagHasSubTags(id)` | M7 (Tagging Core) |
| `HasRatedBookmarks()`, `BookmarkContains(text)`, `AmountBookmarks` | M10 (Player & Bookmarks) |

### Filter slot storage: new `filter_slots` table

A new migration (`m037_add_filter_slots.sql`) creates a `filter_slots` table:

```sql
CREATE TABLE IF NOT EXISTS filter_slots (
    slot_id    INT          NOT NULL AUTO_INCREMENT,
    name       VARCHAR(200) NOT NULL,
    expression TEXT         NOT NULL,
    PRIMARY KEY (slot_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Schema revision bumped from 36 → **37**. No `sort_order` column in M5 — slots are returned `ORDER BY slot_id`. A reorder column will be added in a later migration when (and if) a reorder UI is scoped.

`IFilterSlotRepository` / `DapperFilterSlotRepository` follows the `ILibraryFolderRepository` / `SpdbConfigLibraryFolderRepository` pattern: async, Dapper, parameterised queries only.

### Active filter slot persistence: `spdb_config` (database-scoped)

The active slot pointer is database-scoped (filter slots themselves are database-scoped), so it lives in `spdb_config` next to the slots, not in user-scoped `appsettings.json`. A new `IActiveFilterSlotRepository` (Core) exposes `GetActiveSlotIdAsync()` / `SetActiveSlotIdAsync(int? slotId)`; the Dapper-backed `SpdbConfigActiveFilterSlotRepository` (Infrastructure) reads and writes a single key (`active_filter_slot_id`) in `spdb_config`, mirroring the `SpdbConfigLibraryFolderRepository` pattern. A `null` value (or a missing key) means "no active filter". By co-locating the pointer with the entity it points to, we eliminate by construction the cross-database staleness scenario that arises from per-user-file storage. (`MoviesListOptions` is **not** modified for this milestone; `ShowCoverPanel`/`ColumnVisibility` remain user-scoped because column layout is genuinely a per-user preference, but the active filter slot is not.)

### Filter Manager dialog: follows `IConnectionEditorService` pattern

`IFilterManagerService.ShowAsync(FiltersManagerViewModel viewModel)` opens a modal Avalonia `Window` hosting `FiltersManagerView`. The caller (`MoviesListViewModel.OpenFilterManagerCommand`) creates the VM directly via `new FiltersManagerViewModel(_filterSlotRepository)` (no DI resolution), calls `await vm.LoadAsync()` to populate slots before the dialog opens, calls `ShowAsync`, then reloads the filter slot list. This follows the same modal-window hosting technique as `AvaloniaConnectionEditorService`; the signatures differ (`ShowAsync` has a void return and accepts the caller-supplied VM, whereas `ShowEditorAsync` returns `Task<DatabaseConnectionOptions?>` and creates its own VM internally).

---

## Rationale

- **In-memory filtering** — simplest implementation consistent with spec, no risk of SQL-injection vectors in dynamic query building, and no observable performance penalty at expected collection sizes.
- **DSL in Core** — the filter expression language is domain logic, not UI logic. Placing it in Core keeps the evaluator testable without Avalonia, follows the interface-first constraint, and lets Infrastructure or future headless consumers reuse it.
- **`filter_slots` dedicated table (no `sort_order`)** — named slots with multiple typed fields exceed what `spdb_config` key/value pairs can cleanly represent (the JSON-encoding workaround used by `SpdbConfigLibraryFolderRepository` is a smell the new feature should not inherit). The `sort_order` column is deferred until a reorder UI is actually scoped — adding speculative columns now would have no writer.
- **Active slot pointer in `spdb_config`** — the slot pointer is database-scoped because filter slots themselves are database-scoped. Co-locating the pointer with the entity eliminates the cross-DB staleness scenario by construction. Storing it in `MoviesListOptions` (originally considered) would have required silent-clearing fallback logic on every DB switch.
- **Numeric literals are `decimal`** — the spec admits decimal literal forms (`5.0`); pinning the literal type now (rather than letting it default to `double` or be silently rejected) prevents float-equality fragility for `=`/`!=` and locks comparison semantics before downstream consumers depend on them.
- **Single `FilterExpressionException` with a `Phase` enum** — by contract, the parser rejects every malformed input, so an evaluator-thrown exception only ever signals a bug. A single exception type with a `Phase` (`Parse`/`Evaluate`) discriminator carries the same diagnostic information with one fewer file and one fewer catch site.
- **Deferred DSL identifiers cause a parse-time error** — rejecting unknowns at save time (per spec §4) is safer than silently evaluating to `false`, which could mislead users into thinking their filter is correct when it is silently doing nothing.

---

## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|---|---|---|---|
| Filtering location | In-memory ViewModel filter | Server-side SQL WHERE clauses | SQL approach requires dynamic query building (injection risk, `TreatWarningsAsErrors` complexity), requires DB round-trip per keystroke, and yields no performance benefit at personal-collection scale |
| DSL location | `VideoIndexer.Core/Filtering/` | App layer; Infrastructure layer | App layer would pollute domain logic with UI concerns; Infrastructure would add a UI-only feature to the data layer. Core is the natural home for pure domain evaluation logic |
| Filter slot storage | Dedicated `filter_slots` table | `spdb_config` key/value rows | `spdb_config` cannot cleanly represent multi-field named entities; a dedicated table is minimal and correct |
| `sort_order` column | Deferred until a reorder UI exists | Include now, leave unused | Including now adds a column with no current writer; the reorder UI is not in M5 scope, so the column ships dead. A future migration can add it when the writer arrives. |
| Active slot persistence | `spdb_config` (database-scoped) | `MoviesListOptions` (per-user `appsettings.json`) | The slot pointer is database-scoped (it points at rows in this DB); co-locating it with the slots eliminates the cross-DB staleness scenario by construction. The user-scoped option would have required silent-clearing fallback on every DB switch. |
| Numeric literal type | `decimal` (with implicit promotion of `int` fields) | `int`/`long` (reject decimal literals); `double` (silent widening) | `int`/`long` would reject the spec-permitted `5.0` form; `double` introduces float-equality fragility for `=`/`!=`. `decimal` covers every literal a user can type and gives exact comparison semantics. |
| DSL exception types | Single `FilterExpressionException` with a `Phase` enum | Two distinct types (`FilterParseException` + `FilterEvaluationException`) | The evaluator throws only on parser bugs (deferred identifiers are rejected at parse time), so a second type carries no extra information. One type means one catch site in the VM. |
| `MovieListItem.Description` cardinality | `string Description { get; init; } = ""` | `required string Description { get; init; }` | The `movies.description` column is `TEXT NOT NULL DEFAULT ''`, so empty-string is the natural absent value. Defaulting eliminates the test-fixture ripple (`MakeItem`, `InMemoryMovieCatalogRepository`, future call sites) without any semantic loss. |
| `MovieListQuery` parameter | Retained, null-guarded | Remove from the interface | Removal is the honest shape but ripples through every test fixture mocking `IMovieCatalogRepository` and through any not-yet-merged WIP. The friction outweighs the cleanup benefit at this stage; the parameter is retained as an extension point. |
| DSL parser | Hand-rolled recursive descent | Third-party parsing library (e.g. Superpower, Sprache) | Adding a NuGet library would violate the Core "no external dependencies" constraint and is unjustified for a small, well-scoped grammar |
| Deferred identifiers | Parse-time validation error | Silent `false` evaluation | Silent `false` is dishonest; the spec explicitly states that unknown identifiers are rejected at save time. A clear message (e.g. "HasTag is not available until M7") is better UX |

---

## Pattern Alignment

| Pattern | Alignment |
|---|---|
| `IMovieCatalogRepository` / `DapperMovieCatalogRepository` in `Library/` | `IFilterSlotRepository` / `DapperFilterSlotRepository` follows the same file location and naming convention |
| `ILibraryFolderRepository.GetAllAsync / AddAsync / RemoveAsync` | `IFilterSlotRepository.GetAllAsync / SaveAsync / DeleteAsync` mirrors the same method shape |
| `SpdbConfigLibraryFolderRepository` (key/value access to `spdb_config`) | `SpdbConfigActiveFilterSlotRepository` follows the same key-based read/write shape (single key `active_filter_slot_id`) |
| `IConnectionEditorService` / `AvaloniaConnectionEditorService` | `IFilterManagerService` / `AvaloniaFilterManagerService` follows the same modal-window hosting technique; signatures differ (void return + caller-supplied VM vs. `Task<DTO?>` + internally-created VM) |
| `MoviesListOptions` sealed record with default values | Unchanged in M5 — active filter slot is database-scoped (`spdb_config`), not user-scoped (`appsettings.json`) |
| `Interlocked.CompareExchange` single-flight in `LoadAsync` | Retained unchanged; `ApplyFilter()` is synchronous and does not require its own guard |
| `[ObservableProperty]` / `[RelayCommand]` source generators | All new ViewModel properties and commands follow the same pattern |
| `ColumnKeys` nested static class in `MoviesListOptions` | No analogous constant class is needed for filter DSL identifiers; they are defined as `const string` fields in the evaluator mapping |
| `ReflectionBinding` for DataGrid header commands | No new DataGrid columns added in M5; existing pattern unchanged |
| `ConfigureAwait(false)` on all infrastructure `await` calls | Maintained throughout `DapperFilterSlotRepository` |
| `INSERT IGNORE` / parameterised Dapper queries | Maintained in `DapperFilterSlotRepository.SaveAsync` (upsert pattern) |
| Core has no NuGet deps | DSL parser/evaluator uses only BCL types — no new packages needed |

---

## Detailed Steps

1. **WP-001 — DB migration m037_add_filter_slots.sql + schema revision 37**
   - Create `src/VideoIndexer.Infrastructure/Database/migrations/m037_add_filter_slots.sql`.
   - Script creates `filter_slots` table (`slot_id`, `name`, `expression` — no `sort_order`) and updates `db_revision` to `37`.
   - Update `DatabaseBootstrapper.ExpectedRevision = 37`.

2. **WP-002 — Core: `FilterSlot` model + `IFilterSlotRepository` + `IActiveFilterSlotRepository`**
   - Create `src/VideoIndexer.Core/Models/FilterSlot.cs` — sealed record with `SlotId`, `Name`, `Expression` (no `SortOrder`).
   - Create `src/VideoIndexer.Core/Abstractions/IFilterSlotRepository.cs` — `GetAllAsync`, `Task<FilterSlot> SaveAsync(FilterSlot slot)` (returns the entity with the database-assigned `SlotId` for new inserts; follows the `ILibraryFolderRepository.AddAsync` precedent), `DeleteAsync(int slotId)`.
   - Create `src/VideoIndexer.Core/Abstractions/IActiveFilterSlotRepository.cs` — `Task<int?> GetActiveSlotIdAsync(CancellationToken)`, `Task SetActiveSlotIdAsync(int? slotId, CancellationToken)`.

3. **WP-003 — `MovieListItem` `Description` field + SQL + null-guard**
   - Add `string Description { get; init; } = "";` to `MovieListItem` (default-empty, **not** `required`, so existing test fixtures need no modification).
   - Add `m.description AS Description` to `MovieListSql` in `DapperMovieCatalogRepository`.
   - Add `ArgumentNullException.ThrowIfNull(query)` at the top of `GetMovieListAsync`.
   - Update `MovieListQuery` XML doc comment to clarify that M5 filtering is in-memory (not in SQL).

4. **WP-004 — `DapperFilterSlotRepository` + `SpdbConfigActiveFilterSlotRepository`**
   - Create `src/VideoIndexer.Infrastructure/Library/DapperFilterSlotRepository.cs`.
     - `GetAllAsync` — `SELECT slot_id, name, expression FROM filter_slots ORDER BY slot_id`.
     - `SaveAsync` — `INSERT INTO filter_slots … ON DUPLICATE KEY UPDATE …` (upsert keyed on `slot_id = 0` for new, existing `slot_id` for update). For new slots (`slot.SlotId == 0`), retrieve the assigned ID with `await connection.ExecuteScalarAsync<int>("SELECT LAST_INSERT_ID()")` and return `slot with { SlotId = newId }`. For updates (`slot.SlotId > 0`), return the input slot unchanged.
     - `DeleteAsync` — parameterised `DELETE FROM filter_slots WHERE slot_id = @SlotId`.
   - Create `src/VideoIndexer.Infrastructure/Library/SpdbConfigActiveFilterSlotRepository.cs`.
     - Constructor: `public SpdbConfigActiveFilterSlotRepository(ISpdbConfigRepository configRepository)` — injects `ISpdbConfigRepository` rather than `IDbConnectionFactory`. This class manages a single key with no cross-key transaction scope, so the higher-level `ISpdbConfigRepository` abstraction is the simpler and more appropriate dependency (avoids the raw-connection boilerplate used by `SpdbConfigLibraryFolderRepository`, which needs transaction scope for its two-key upsert).
     - `GetActiveSlotIdAsync` — delegates to `ISpdbConfigRepository.GetAsync("active_filter_slot_id")`; returns `null` if the key is missing or the value is empty/non-numeric.
     - `SetActiveSlotIdAsync(null)` — calls `ISpdbConfigRepository.SetAsync("active_filter_slot_id", "")` (clears the pointer via empty-string convention).
     - `SetActiveSlotIdAsync(int)` — calls `ISpdbConfigRepository.SetAsync("active_filter_slot_id", slotId.ToString())`.
   - All queries fully parameterised; no string interpolation.

5. **WP-005 — Filter DSL: lexer, parser, AST**
   - Create `src/VideoIndexer.Core/Filtering/` directory with:
     - `FilterToken.cs` — token enum (`And`, `Or`, `Not`, `LParen`, `RParen`, `Eq`, `Neq`, `Lt`, `Gt`, `Lte`, `Gte`, `Identifier`, `NumberLiteral`, `StringLiteral`, `Eof`)
     - `FilterLexer.cs` — tokenises an expression string; reports `FilterExpressionException` (Phase = Parse) on unrecognised characters. Numeric tokens accept both integer (`42`) and decimal (`5.0`) forms; both produce a `decimal` value on the resulting `NumberLiteralNode`.
     - `FilterExpressionNode.cs` — sealed record hierarchy: `BinaryNode` (left, op, right), `UnaryNotNode` (operand), `ComparisonNode` (left, op, right), `FunctionCallNode` (name, args), `IdentifierNode` (name), `NumberLiteralNode(decimal Value)`, `StringLiteralNode` (value).
     - `FilterExpressionParser.cs` — recursive descent; entry point `FilterExpressionNode Parse(string expression)`. Throws `FilterExpressionException` (Phase = Parse) on syntax errors or deferred-identifier usage.
     - `FilterExpressionException.cs` — single `Exception` subclass carrying `Phase` (`Parse` / `Evaluate`), the offending token position, and the message. Replaces the earlier two-exception design.

6. **WP-006 — Filter DSL: evaluator**
   - Create `src/VideoIndexer.Core/Filtering/FilterExpressionEvaluator.cs`.
   - Entry point: `bool Evaluate(FilterExpressionNode expression, MovieListItem item)`.
   - Maps M5-supported identifiers to `MovieListItem` fields (see table above).
   - Numeric property mapping: `Rating→(decimal)item.Rating`, `TagsWeight→(decimal)item.TagWeight`, `AmountTags→(decimal)item.TagCount`, `AmountThumbnails→(decimal)item.ThumbnailCount`, `ViewCount→(decimal)item.ViewCount`. All comparisons happen in `decimal` space.
   - Boolean function mapping: `HasBookmarks()→item.HasBookmarks`, `HasCoverImage()→item.HasCoverImage`, `HasThumbnails()→item.ThumbnailCount>0`, `HasRating()→item.Rating>0`, `NeedsReview()→item.NeedsReview`, `HasTags()→item.TagCount>0`.
   - String-predicate functions: `ActorsContain(t)→item.ActorNames.Contains(t, OrdinalIgnoreCase)`, `StudioContains(t)→item.Studio.Contains(t, OrdinalIgnoreCase)`.
   - Throws `FilterExpressionException` (Phase = Evaluate) only as a defensive guard — the parser is expected to have already rejected any input that would reach this branch.

7. **WP-007 — `MoviesListOptions` (no change in M5)**
   - No changes to `MoviesListOptions` in M5. The active filter slot pointer is database-scoped and lives in `spdb_config` via `IActiveFilterSlotRepository` (covered by WP-002/WP-004), not in `appsettings.json`.
   - This work package is retained as a checkpoint — confirm `MoviesListOptions` is left alone and no `ActiveFilterSlotId` field is added.

8. **WP-008 — `MoviesListViewModel` updates: search + filter integration**
   - Add `IFilterSlotRepository`, `IActiveFilterSlotRepository`, and `IFilterManagerService` constructor parameters (new full constructor; parameterless test constructor remains).
   - Add `private IReadOnlyList<MovieListItem> _allMovies = Array.Empty<MovieListItem>();` backing cache.
   - Update `LoadAsync` to set `_allMovies` and then call `ApplyFilter()` instead of directly populating `Movies`.
   - Add `[ObservableProperty] private string _searchText = string.Empty;` — `partial void OnSearchTextChanged(string value)` calls `ApplyFilter()`.
   - Add `[ObservableProperty] private IReadOnlyList<FilterSlot> _filterSlots = Array.Empty<FilterSlot>();`.
   - Add `[ObservableProperty] private FilterSlot? _activeFilterSlot;` — `partial void OnActiveFilterSlotChanged(FilterSlot? value)` persists the new ID via `IActiveFilterSlotRepository.SetActiveSlotIdAsync(value?.SlotId)` (fire-and-forget with logged error), then calls `ApplyFilter()`.
   - Add `[ObservableProperty] private bool _isFilterActive;` — `true` when `ActiveFilterSlot` is non-null.
   - Add `[ObservableProperty] private bool _hasLoadError;` — set in `LoadAsync` catch block. **Bound to a visible banner in `MainContentView.axaml` — see WP-012.**
   - Add `[ObservableProperty] private string? _loadErrorMessage;` — populated alongside `HasLoadError` so the banner can show a meaningful message.
   - Add `[RelayCommand] private async Task LoadFilterSlotsAsync()` — guard: `if (_filterSlotRepository is null || _activeSlotRepository is null) return;` (parameterless-constructor path, consistent with the `LoadAsync` null-guard pattern). Populates `FilterSlots`, then resolves `ActiveFilterSlot` from `IActiveFilterSlotRepository.GetActiveSlotIdAsync()`. If the stored ID is present in `FilterSlots`, set `ActiveFilterSlot` to the matching entry. If the stored ID is not present in the loaded slot list, silently call `SetActiveSlotIdAsync(null)` to clear the stale pointer (defensive guard against manual DB edits — should not normally happen with the database-scoped storage).
   - Add `[RelayCommand] private async Task OpenFilterManagerAsync()` — constructs `new FiltersManagerViewModel(_filterSlotRepository)` directly (no DI lookup; mirrors how `ConnectionEditorViewModel` is constructed in `AvaloniaConnectionEditorService`), calls `await vm.LoadAsync()` to populate slots before the dialog opens, calls `await _filterManagerService.ShowAsync(vm)`, then calls `await LoadFilterSlotsAsync()`.
   - Add `private FilterExpressionNode? _activeFilterExpression;` — cached AST produced by `FilterExpressionParser.Parse(ActiveFilterSlot.Expression)`. Set in `partial void OnActiveFilterSlotChanged(FilterSlot? value)` before calling `ApplyFilter()` (parse once on slot change, not on every keystroke). Set to `null` when the slot is cleared.
   - Add `private void ApplyFilter()` — LINQ pass over `_allMovies`: text search (label + actor names + studio + description, split on whitespace, min 2 chars, all terms must match), then DSL filter by calling `FilterExpressionEvaluator.Evaluate(_activeFilterExpression, item)` when `_activeFilterExpression` is non-null (reads the cached AST — no re-parse on each invocation), populates `Movies`, updates `IsEmpty` and `IsFilterActive`.
   - Update `LoadAsync` to call `LoadFilterSlotsAsync` before the first `ApplyFilter()`.

9. **WP-010 — `FiltersManagerViewModel`**
    - Create `src/VideoIndexer.App/ViewModels/FiltersManagerViewModel.cs`.
    - Properties: `ObservableCollection<FilterSlot> Slots`, `[ObservableProperty] FilterSlot? SelectedSlot`, `[ObservableProperty] string EditName`, `[ObservableProperty] string EditExpression`, `[ObservableProperty] string? ValidationMessage`, `[ObservableProperty] bool ConfirmDelete`.
    - `event EventHandler? CloseRequested;` — raised by `CloseCommand`. `AvaloniaFilterManagerService.ShowAsync` subscribes `(_, _) => dialog.Close()` before calling `dialog.ShowDialog(owner)`, mirroring the `ConnectionEditorViewModel.CloseRequested` pattern in `AvaloniaConnectionEditorService`.
    - Commands: `AddSlotCommand` (append new blank slot), `SaveSlotCommand` (validate expression via parser, upsert via repository; on new slot, updates `SelectedSlot` with the returned entity to reflect the assigned `SlotId`), `DeleteSlotCommand` (CanExecute: `SelectedSlot != null && ConfirmDelete == true`; delete via repository; resets `ConfirmDelete` to `false` after execution), `CloseCommand` (raises `CloseRequested`).
    - The ViewModel resets `ConfirmDelete` to `false` in `partial void OnSelectedSlotChanged(FilterSlot? value)`, preventing accidental deletes after slot selection changes. This keeps the reset in the VM and makes delete confirmation fully testable in unit tests without a live dialog.
    - `SaveSlotCommand` uses `FilterExpressionParser.Parse(EditExpression)` for validation; sets `ValidationMessage` on `FilterExpressionException` (Phase = Parse).
    - Constructor accepts `IFilterSlotRepository`.
    - `LoadAsync()` — populates `Slots` from repository.

10. **WP-009 — Filter Manager dialog service** *(must be implemented after step 9 — WP-010's `FiltersManagerViewModel` type must exist before this interface can compile)*
    - Create `src/VideoIndexer.App/Services/IFilterManagerService.cs`:
      ```csharp
      Task ShowAsync(FiltersManagerViewModel viewModel, CancellationToken cancellationToken = default);
      ```
    - Create `src/VideoIndexer.App/Services/AvaloniaFilterManagerService.cs`: opens a modal Avalonia `Window` hosting `FiltersManagerView`. Before calling `dialog.ShowDialog(owner)`, subscribes `vm.CloseRequested += (_, _) => dialog.Close()` to close the window when the VM signals close (mirrors `AvaloniaConnectionEditorService`).
    - Constructor: `public AvaloniaFilterManagerService(Func<Window?> ownerFactory)` — the owner window is resolved lazily at call time, not at construction time. In WP-012 (`Program.cs`), register with the same factory lambda used for `IConnectionEditorService`: `builder.Services.AddSingleton<IFilterManagerService>(_ => new AvaloniaFilterManagerService(() => (App.Current?.ApplicationLifetime as IClassicDesktopStyleApplicationLifetime)?.MainWindow));`.

11. **WP-011 — `FiltersManagerView.axaml` + code-behind**
    - Create `src/VideoIndexer.App/Views/FiltersManagerView.axaml` (Window, not UserControl).
    - Layout: two-column split — left: `ListBox` bound to `Slots` with Add/Delete buttons; right: `TextBox` for Name, multi-line `TextBox` for Expression, `TextBlock` for `ValidationMessage`, Save button.
    - Include a reference `Expander` panel listing all M5-supported functions and properties with brief descriptions (static text matching spec §3).
    - Code-behind: parameterless constructor. The window is closed via the `CloseRequested` event on `FiltersManagerViewModel`, subscribed by `AvaloniaFilterManagerService.ShowAsync` — the code-behind does not call `Close()` directly.

12. **WP-012 — `MainContentView.axaml` update + DI wiring**
    - In `MainContentView.axaml`: replace the `IsEnabled="False"` search `TextBox` stub with a live `TextBox` bound to `MoviesListVm.SearchText`; add a `ComboBox` bound to `MoviesListVm.FilterSlots` / `MoviesListVm.ActiveFilterSlot`; add an orange visual indicator (`Ellipse` or `Border`) visible when `MoviesListVm.IsFilterActive`; add a "Filters…" `Button` bound to `MoviesListVm.OpenFilterManagerCommand`.
    - **Add a load-error banner** above the movies grid: a `Border` with a red/orange background and a `TextBlock` bound to `MoviesListVm.LoadErrorMessage`, with `IsVisible="{Binding MoviesListVm.HasLoadError}"`. This closes the M4 deferred item end-to-end (flag → user-visible surface), preventing the silent-empty-grid failure mode.
    - In `Program.cs`: register `IFilterSlotRepository` → `DapperFilterSlotRepository` (**singleton**, consistent with `IMovieCatalogRepository`/`DapperMovieCatalogRepository`; both are stateless and open connections per call via `IDbConnectionFactory`); register `IActiveFilterSlotRepository` → `SpdbConfigActiveFilterSlotRepository` (**singleton**, same rationale); register `IFilterManagerService` → `AvaloniaFilterManagerService` (singleton, using the lazy-owner-factory lambda pattern — see step 10 WP-009). *(Note: `FiltersManagerViewModel` is **not** registered in DI — it is constructed directly in `OpenFilterManagerAsync` via `new FiltersManagerViewModel(_filterSlotRepository)`, mirroring how `ConnectionEditorViewModel` is handled in `AvaloniaConnectionEditorService`.)*
    - Extend `MoviesListViewModel` DI registration to inject the new constructor parameters.

13. **WP-013 — DSL unit tests**
    - Create `tests/VideoIndexer.Tests/Filtering/FilterExpressionParserTests.cs` — test file.
      - Valid expressions: single property comparison, boolean function, AND/OR chain, NOT, grouped expression, string function, numeric literals, string literals.
      - Invalid: syntax errors (missing paren, unrecognised operator, bare string), deferred identifiers (`HasTag(1)` → parse exception with message containing "M7"), type mismatches at parse time where detectable.
    - Create `tests/VideoIndexer.Tests/Filtering/FilterExpressionEvaluatorTests.cs` — test file.
      - Evaluate `Rating >= 4` against items with various ratings.
      - Evaluate `NeedsReview()` true/false.
      - Evaluate `ActorsContain("Smith")` with matching/non-matching actor names.
      - Evaluate `HasThumbnails() AND HasBookmarks()` — all combinations.
      - Evaluate `NOT NeedsReview()`.
      - Evaluate complex expression with AND/OR precedence.
      - Evaluate `AmountTags = 0` against items with zero/nonzero tag counts.

14. **WP-014 — `MoviesListViewModel` extended unit tests**
    - Extend `tests/VideoIndexer.App.Tests/MoviesListViewModelTests.cs` with new test cases:
      - `SearchText_SingleTerm_FiltersMovies` — search narrows collection.
      - `SearchText_MultiTerm_AllTermsMustMatch` — both terms required.
      - `SearchText_ShortTerm_OneChar_IsIgnored` — terms < 2 chars are dropped.
      - `SearchText_ClearText_RestoresFullList`.
      - `SearchText_MatchesDescription` — term found in description field.
      - `ActiveFilterSlot_Set_FilterIsApplied` — DSL filter narrows results.
      - `ActiveFilterSlot_Cleared_RestoresFullList`.
      - `IsFilterActive_TrueWhenSlotActive_FalseOtherwise`.
      - `LoadAsync_RepositoryThrows_SetsHasLoadError`.
    - Create `tests/VideoIndexer.App.Tests/TestHelpers/FakeFilterSlotRepository.cs` — implements `IFilterSlotRepository`; returns configurable list; tracks call counts.
    - Create `tests/VideoIndexer.App.Tests/TestHelpers/FakeActiveFilterSlotRepository.cs` — implements `IActiveFilterSlotRepository`; stores the active slot ID in memory; returns `null` by default.
    - Create `tests/VideoIndexer.App.Tests/TestHelpers/FakeFilterManagerService.cs` — implements `IFilterManagerService`; no-op `ShowAsync`; satisfies the new `MoviesListViewModel` full constructor.
    - Update `BuildSut` helper to accept three new optional parameters: `FakeFilterSlotRepository? filterSlotRepo = null`, `FakeActiveFilterSlotRepository? activeSlotRepo = null`, `FakeFilterManagerService? filterManagerService = null`; wire them into the new full constructor alongside the existing `repo` and `settings` parameters.
    - Create `tests/VideoIndexer.App.Tests/FiltersManagerViewModelTests.cs`:
      - `SaveSlot_ValidExpression_SlotSavedViaRepository` — AC: valid expressions are accepted and persisted
      - `SaveSlot_InvalidExpression_SetsValidationMessage` — AC: syntax errors display inline validation message
      - `SaveSlot_InvalidExpression_DoesNotCallRepository` — AC: syntax errors prevent saving
      - `SaveSlot_NewSlot_SelectedSlotUpdatedWithAssignedId` — AC: `SelectedSlot` reflects DB-assigned `SlotId` after insert
      - `DeleteSlot_ConfirmDeleteFalse_CommandIsDisabled` — AC: delete requires confirmation
      - `DeleteSlot_NoSelectedSlot_CommandIsDisabled` — AC: delete requires a selected slot
      - `DeleteSlot_Executes_ResetsConfirmDeleteToFalse` — AC: confirmation flag cleared after delete
      - `CloseCommand_RaisesCloseRequested` — AC: dialog close signalled via VM event
      - `LoadAsync_PopulatesSlots` — AC: slot list populated on load

15. **WP-015 — Infrastructure integration tests: `DapperFilterSlotRepository` + `SpdbConfigActiveFilterSlotRepository`**
    - Create `tests/VideoIndexer.Infrastructure.Tests/Library/DapperFilterSlotRepositoryTests.cs`.
      - `GetAllAsync_ReturnsEmpty_WhenNoSlots`.
      - `SaveAsync_NewSlot_InsertsRow` — verify `slot_id > 0` after insert.
      - `SaveAsync_ExistingSlot_UpdatesRow` — verify name/expression changed.
      - `DeleteAsync_RemovesRow` — verify row gone after delete.
    - Create `tests/VideoIndexer.Infrastructure.Tests/Library/SpdbConfigActiveFilterSlotRepositoryTests.cs`.
      - `GetActiveSlotIdAsync_MissingKey_ReturnsNull`.
      - `SetActiveSlotIdAsync_Int_PersistsAndReadsBack`.
      - `SetActiveSlotIdAsync_Null_ClearsKey`.
    - All tests use transaction rollback discipline (follow `DapperMovieCatalogRepositoryTests` fixture pattern).

16. **WP-016 — Manifest updates**
    - Update all affected project manifest documents (see Documentation Updates section below).

---

## Dependencies

- WP-001 must complete before WP-004 (migration defines the table shape).
- WP-002 must complete before WP-004 (interface and model needed by repository).
- WP-003 must complete before WP-008 (`MovieListItem.Description` needed by `ApplyFilter()`).
- WP-005 must complete before WP-006 (AST nodes needed by evaluator).
- WP-005 and WP-006 must complete before WP-008 (`FilterExpressionEvaluator` used by `ApplyFilter()`).
- WP-004 must complete before WP-008 (repository injected into VM).
- WP-008 must complete before WP-014 (VM under test must exist).
- WP-004 must complete before WP-015 (repository under test must exist).
- WP-009 must complete before WP-012 (service must be registerable).
- WP-010 must complete before WP-009, WP-011, WP-012 (VM type must exist for service + view + DI).
- WP-011 must complete before WP-012 (view must exist for DI factory).

**Parallel tracks (after WP-001/WP-002/WP-003):**
- Track A: WP-004 → WP-008 → WP-014
- Track B: WP-005 → WP-006 → WP-013
- Track C: WP-007 (small; can be done alongside any other WP)
- Track D: WP-010 → WP-009 → WP-011 → WP-012

---

## Required Components

**New source files:**
- `src/VideoIndexer.Infrastructure/Database/migrations/m037_add_filter_slots.sql`
- `src/VideoIndexer.Core/Models/FilterSlot.cs`
- `src/VideoIndexer.Core/Abstractions/IFilterSlotRepository.cs`
- `src/VideoIndexer.Core/Abstractions/IActiveFilterSlotRepository.cs`
- `src/VideoIndexer.Core/Filtering/FilterToken.cs`
- `src/VideoIndexer.Core/Filtering/FilterLexer.cs`
- `src/VideoIndexer.Core/Filtering/FilterExpressionNode.cs`
- `src/VideoIndexer.Core/Filtering/FilterExpressionParser.cs`
- `src/VideoIndexer.Core/Filtering/FilterExpressionException.cs`
- `src/VideoIndexer.Core/Filtering/FilterExpressionEvaluator.cs`
- `src/VideoIndexer.Infrastructure/Library/DapperFilterSlotRepository.cs`
- `src/VideoIndexer.Infrastructure/Library/SpdbConfigActiveFilterSlotRepository.cs`
- `src/VideoIndexer.App/Services/IFilterManagerService.cs`
- `src/VideoIndexer.App/Services/AvaloniaFilterManagerService.cs`
- `src/VideoIndexer.App/ViewModels/FiltersManagerViewModel.cs`
- `src/VideoIndexer.App/Views/FiltersManagerView.axaml`
- `src/VideoIndexer.App/Views/FiltersManagerView.axaml.cs`
- `tests/VideoIndexer.Tests/Filtering/FilterExpressionParserTests.cs`
- `tests/VideoIndexer.Tests/Filtering/FilterExpressionEvaluatorTests.cs`
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeFilterSlotRepository.cs`
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeActiveFilterSlotRepository.cs`
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeFilterManagerService.cs`
- `tests/VideoIndexer.App.Tests/FiltersManagerViewModelTests.cs`
- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperFilterSlotRepositoryTests.cs`
- `tests/VideoIndexer.Infrastructure.Tests/Library/SpdbConfigActiveFilterSlotRepositoryTests.cs`

**Modified source files:**
- `src/VideoIndexer.Core/Models/MovieListItem.cs` — add `Description` (default-empty `init` property; no `required`)
- `src/VideoIndexer.Core/Models/MovieListQuery.cs` — update XML doc comment
- `src/VideoIndexer.Infrastructure/Database/DatabaseBootstrapper.cs` — `ExpectedRevision = 37`
- `src/VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs` — add `Description` to SQL; null-guard
- `src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs` — search + filter + new deps + `LoadErrorMessage`
- `src/VideoIndexer.App/Views/MainContentView.axaml` — enable search, add filter controls, **add load-error banner**
- `src/VideoIndexer.App/Program.cs` — DI registrations
- `tests/VideoIndexer.App.Tests/MoviesListViewModelTests.cs` — new test cases + `BuildSut` extended with three optional fake parameters

*(Note: `MoviesListOptions.cs`, `appsettings.json`, `InMemoryMovieCatalogRepository.cs`, and the existing `MakeItem()` helpers in test fixtures are **not** modified — `MoviesListOptions` is unchanged because the active slot pointer lives in `spdb_config`, and `Description` defaults to empty string so existing `MovieListItem` constructions compile without change.)*

---

## Assumptions

- The `movies` table `description` column is `TEXT NOT NULL DEFAULT ''` (consistent with `InsertMovieAsync` which already writes an empty string); the SQL `AS Description` mapping will never produce `NULL`. WP-003's `Description = ""` default depends on this invariant.
- The `spdb_config` table is created by an earlier migration and is available before M5; storing the active filter slot pointer there does not require schema changes beyond m037.
- Filter slot expressions are short enough that loading all slots in memory on startup is not a concern (personal collection: tens of slots, not thousands).
- The `filter_slots` table will be created manually by the user running `m037_add_filter_slots.sql` — no auto-migration exists (consistent with how m036 was handled).
- The Avalonia modal window host for `FiltersManagerView` can be resolved from `App.Services` using the same pattern as `AvaloniaConnectionEditorService`.
- No new NuGet packages are required. The DSL parser is a hand-rolled recursive descent implementation using BCL types only.

---

## Constraints

- All warnings are errors (`TreatWarningsAsErrors=true`) — zero new warnings are acceptable.
- Core must not reference any external NuGet packages — the DSL parser/evaluator uses only BCL.
- All Dapper queries in `DapperFilterSlotRepository` must use `@Param` parameterised variables — no string concatenation or interpolation into SQL.
- The active filter slot pointer is persisted exclusively via `IActiveFilterSlotRepository.SetActiveSlotIdAsync` (writes `spdb_config`). It must **not** be stored in `MoviesListOptions` or written via `ISettingsService.SaveAsync`. (The global `AppOptions` immutability rule — always produce a new record via `with { }` + `ISettingsService.SaveAsync` — still applies to all other user-scoped settings.)
- `FilterExpressionParser.Parse()` must be synchronous (no async) — it is called from the ViewModel on the UI thread during expression validation.
- `ArgumentNullException.ThrowIfNull(query)` must be the first line in `DapperMovieCatalogRepository.GetMovieListAsync` before any other code.
- `NumberLiteralNode.Value` is `decimal`. `MovieListItem` numeric fields are promoted to `decimal` at evaluation time. Do not introduce `double` in the comparison path.
- The search TextBox minimum-term length is **2 characters** (spec §1) — terms with fewer characters are silently skipped.
- `FiltersManagerView` is a `Window` (modal), not a `UserControl`, following the `ConnectionEditorView` non-pattern exception — see existing `AvaloniaConnectionEditorService` for the correct modal hosting technique.

---

## Out of Scope

- **Tag-based DSL functions** (`HasTag(id)`, `TagHasSubTags(id)`) — deferred to M7.
- **Bookmark-depth DSL properties/functions** (`HasRatedBookmarks()`, `BookmarkContains(text)`, `AmountBookmarks`) — deferred to M10.
- **Context menu command implementations** (Play, PlayFullscreen, Edit, GenerateThumbnails, CopyTo, DeleteOnDisk) — stubs remain until M6/M9/M10.
- **Column width persistence** (`MoviesListOptions.ColumnWidths`) — deferred to a future milestone.
- **Server-side (SQL) filtering** — all filtering is in-memory in M5.
- **Filter expression auto-completion** in the editor dialog.
- **Pagination** — the full catalog is loaded into `_allMovies` on demand.
- **`HasCoverImage` real data** — remains hardcoded `0` in SQL until M9.

---

## Acceptance Criteria

- [ ] Typing in the search bar narrows the movies grid in real time; clearing the field restores the full list.
- [ ] Search terms under 2 characters are ignored; terms ≥ 2 characters must match (AND) in at least one of: label, actor names, studio, description (case-insensitive substring).
- [ ] Multiple space-separated terms are all required (logical AND) per spec §1.
- [ ] The filter slot dropdown lists all saved slots and a "none" entry; selecting a slot filters the list; selecting none removes the filter.
- [ ] The orange filter-active indicator is visible when a filter slot is selected and hidden when none is active.
- [ ] The active slot selection persists across app restarts.
- [ ] Clicking "Filters…" opens the Filter Manager dialog.
- [ ] In the Filter Manager, the user can add a slot (name + expression), edit an existing slot, and delete a slot (with confirmation).
- [ ] Valid expressions are accepted; syntax errors and deferred-identifier usage display an inline validation message and prevent saving.
- [ ] After closing the Filter Manager, the filter slot dropdown reflects changes without restarting the app.
- [ ] A `HasLoadError` flag is set on the movies list VM when `GetMovieListAsync` throws **and a visible error banner appears in `MainContentView.axaml`** displaying `LoadErrorMessage`. (Closes the M4-deferred item end-to-end.)
- [ ] `GetMovieListAsync(null)` throws `ArgumentNullException` (null-guard in place).
- [ ] `Description` is visible in text search results (i.e., a movie whose label/actor/studio does not match a term but whose description does will appear in results).
- [ ] Schema revision 37 check passes; `DatabaseBootstrapper` rejects revision 36.
- [ ] The active filter slot persists across app restarts **per-database** (switching to a different database connection does not surface a stale slot from another database).
- [ ] DSL expressions accept both integer (`Rating >= 4`) and decimal (`Rating >= 4.5`) numeric literals; `Rating = 4` matches `item.Rating == 4` exactly with no float-equality fragility.
- [ ] All 16 work packages compile with zero warnings; all tests pass.

---

## Testing Strategy

**Unit tests** cover the DSL subsystem exhaustively (parser and evaluator are pure functions — no mocks needed) and the ViewModel search/filter logic (using `FakeMovieCatalogRepository`, `FakeSettingsService`, `FakeFilterSlotRepository`). The `FiltersManagerViewModel` is tested for validation logic and CRUD command behavior.

**Integration tests** cover `DapperFilterSlotRepository` CRUD against a live MariaDB database using the existing `LiveDbFixture` + transaction-rollback discipline.

**Manual acceptance** verifies the end-to-end UI flow: search bar behavior, filter slot dropdown, filter-active indicator, and the Filter Manager dialog (add/edit/delete/validate).

---

## Test Plan

- `tests/VideoIndexer.Tests/Filtering/FilterExpressionParserTests.cs`
  - `Parse_DecimalLiteral_ProducesDecimalValuedNode` — AC: decimal literals (`5.0`) accepted and round-tripped as `decimal`
  - `Parse_IntegerLiteral_ProducesDecimalValuedNode` — AC: integer literals stored as `decimal` (uniform numeric type)
  - `Parse_SinglePropertyComparison_ReturnsComparisonNode` — AC: valid expressions accepted
  - `Parse_BooleanFunction_ReturnsFunctionCallNode` — AC: valid expressions accepted
  - `Parse_AndChain_RespectsPrecedence` — AC: AND/OR operators work correctly
  - `Parse_NotOperator_ReturnsUnaryNode` — AC: NOT operator works
  - `Parse_GroupedExpression_ParsesCorrectly` — AC: parentheses group correctly
  - `Parse_StringFunctionWithArg_ReturnsCorrectNode` — AC: string functions parsed
  - `Parse_MissingCloseParen_ThrowsFilterParseException` — AC: syntax errors detected
  - `Parse_UnrecognisedToken_ThrowsFilterParseException` — AC: syntax errors detected
  - `Parse_DeferredHasTag_ThrowsWithM7Message` — AC: deferred identifiers rejected with clear message
  - `Parse_DeferredAmountBookmarks_ThrowsWithM10Message` — AC: deferred identifiers rejected
  - `Parse_EmptyExpression_ThrowsFilterParseException` — AC: empty input rejected

- `tests/VideoIndexer.Tests/Filtering/FilterExpressionEvaluatorTests.cs`
  - `Evaluate_RatingGte4_TrueForRating5` — AC: numeric comparison works
  - `Evaluate_RatingGte4_FalseForRating3` — AC: numeric comparison works
  - `Evaluate_NeedsReview_TrueForReviewedItem` — AC: boolean function mapping
  - `Evaluate_NeedsReview_FalseForNonReviewedItem` — AC: boolean function mapping
  - `Evaluate_ActorsContain_CaseInsensitiveMatch` — AC: string-predicate function
  - `Evaluate_HasThumbnails_TrueForCountGtZero` — AC: derived boolean from numeric
  - `Evaluate_HasThumbnails_FalseForCountZero` — AC: derived boolean from numeric
  - `Evaluate_AndOperator_RequiresBothTrue` — AC: AND logic
  - `Evaluate_OrOperator_TrueIfEitherTrue` — AC: OR logic
  - `Evaluate_NotOperator_InvertsBooleanResult` — AC: NOT logic
  - `Evaluate_ComplexExpression_CorrectResult` — AC: compound expressions evaluate correctly
  - `Evaluate_AmountTags_EqZero_TrueForNoTags` — AC: AmountTags property mapping

- `tests/VideoIndexer.App.Tests/MoviesListViewModelTests.cs` (new cases)
  - `SearchText_SingleTerm_FiltersToMatchingMovies` — AC: search bar narrows list
  - `SearchText_MultiTerm_RequiresAllTermsToMatch` — AC: multi-term AND logic
  - `SearchText_TermUnderTwoChars_IsIgnored` — AC: min-2-char rule
  - `SearchText_Cleared_RestoresFullCollection` — AC: clear restores full list
  - `SearchText_MatchesDescription_IncludesMovie` — AC: description field searched
  - `ActiveFilterSlot_Set_DSLFilterApplied` — AC: active slot filters list
  - `ActiveFilterSlot_Cleared_RestoresFullCollection` — AC: deactivating slot removes filter
  - `IsFilterActive_TrueWhenSlotSet_FalseWhenNull` — AC: filter-active indicator
  - `LoadAsync_RepositoryThrows_SetsHasLoadError` — AC: error state surfaced
  - `LoadFilterSlotsAsync_StoredIdNotInSlotList_ClearsPointerAndSetsActiveFilterSlotNull` — AC: stale `spdb_config` pointer cleared when the referenced slot no longer exists (configure `FakeActiveFilterSlotRepository` to return an ID absent from `FakeFilterSlotRepository`'s list; assert `SetActiveSlotIdAsync(null)` was called and `ActiveFilterSlot` is null)

- `tests/VideoIndexer.App.Tests/FiltersManagerViewModelTests.cs`
  - `SaveSlot_ValidExpression_SlotSavedViaRepository` — AC: valid expressions are accepted
  - `SaveSlot_InvalidExpression_SetsValidationMessage` — AC: syntax errors display inline validation message and prevent saving
  - `SaveSlot_InvalidExpression_DoesNotCallRepository` — AC: syntax errors prevent saving
  - `SaveSlot_NewSlot_SelectedSlotUpdatedWithAssignedId` — AC: `SelectedSlot` reflects DB-assigned `SlotId` after insert
  - `DeleteSlot_ConfirmDeleteFalse_CommandIsDisabled` — AC: delete requires confirmation
  - `DeleteSlot_NoSelectedSlot_CommandIsDisabled` — AC: delete requires a selected slot
  - `DeleteSlot_Executes_ResetsConfirmDeleteToFalse` — AC: confirmation flag cleared after delete
  - `CloseCommand_RaisesCloseRequested` — AC: dialog closed via VM event
  - `LoadAsync_PopulatesSlots` — AC: slot list populated on load

- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperFilterSlotRepositoryTests.cs`
  - `GetAllAsync_EmptyTable_ReturnsEmptyList` — AC: empty state works
  - `SaveAsync_NewSlot_InsertsAndAssignsSlotId` — AC: new slot persisted
  - `SaveAsync_ExistingSlot_UpdatesNameAndExpression` — AC: edit persisted
  - `DeleteAsync_ExistingSlot_RemovesRow` — AC: delete persisted

- `tests/VideoIndexer.Infrastructure.Tests/Library/SpdbConfigActiveFilterSlotRepositoryTests.cs` *(new)*
  - `GetActiveSlotIdAsync_MissingKey_ReturnsNull` — AC: per-DB pointer defaults to null
  - `SetActiveSlotIdAsync_Int_PersistsAndReadsBack` — AC: pointer round-trips
  - `SetActiveSlotIdAsync_Null_ClearsKey` — AC: clearing the pointer works

---

## Documentation Updates

- `docs/agents/project-manifest/api-surface.md`
  - Add `FilterSlot` model signature
  - Add `IActiveFilterSlotRepository` interface signatures and `SpdbConfigActiveFilterSlotRepository` constructor
  - Add `IFilterSlotRepository` interface signatures
  - Add `FilterExpressionParser` and `FilterExpressionEvaluator` static entry points
  - Update `MovieListItem` to include `Description`
  - Update `MoviesListViewModel` constructor and new observable properties / commands
  - Add `FiltersManagerViewModel` constructor and surface
  - Add `IFilterManagerService.ShowAsync` signature
  - Update `DatabaseBootstrapper.ExpectedRevision = 37`

- `docs/agents/project-manifest/file-tree.md`
  - Add `VideoIndexer.Core/Filtering/` directory and all new files (note: single `FilterExpressionException.cs`, not two separate exception files)
  - Add `FilterSlot.cs`, `IFilterSlotRepository.cs`, `IActiveFilterSlotRepository.cs` under their respective directories
  - Add `DapperFilterSlotRepository.cs` and `SpdbConfigActiveFilterSlotRepository.cs` under `Library/`
  - Add `IFilterManagerService.cs`, `AvaloniaFilterManagerService.cs` under `Services/`
  - Add `FiltersManagerViewModel.cs` under `ViewModels/`
  - Add `FiltersManagerView.axaml` + `.axaml.cs` under `Views/`
  - Add `m037_add_filter_slots.sql` under `migrations/`

- `docs/agents/project-manifest/constraints.md`
  - Update schema revision: **37**
  - Add m037 rollback procedure: `DROP TABLE filter_slots; DELETE FROM spdb_config WHERE config_name = 'active_filter_slot_id'; UPDATE spdb_config SET config_value = '36' WHERE config_name = 'db_revision';`
  - Add note: "Active filter slot pointer is stored in `spdb_config` (key `active_filter_slot_id`), **not** in `MoviesListOptions`/`appsettings.json`. This keeps the pointer database-scoped, matching the scope of the `filter_slots` table itself."
  - Add note: "DSL numeric literals are `decimal`. `MovieListItem` integer fields are promoted to `decimal` at evaluation time. Do not introduce `double` in the comparison path."
  - Add note: "DSL exceptions are a single type (`FilterExpressionException`) discriminated by a `Phase` enum (`Parse` / `Evaluate`). Do not split into separate exception classes."
  - Add note: "When M7 adds tagging: implement `HasTag(id)` and `TagHasSubTags(id)` in `FilterExpressionEvaluator` — the parser already recognises these identifiers; the evaluator is the only change required."
  - Add note: "When M10 adds bookmarks: implement `HasRatedBookmarks()`, `BookmarkContains(text)`, `AmountBookmarks` in `FilterExpressionEvaluator` and add `AmountBookmarks` to `MovieListItem`."
  - Add constraint: "All `DapperFilterSlotRepository` and `SpdbConfigActiveFilterSlotRepository` SQL must use Dapper `@Param` — no string interpolation (SQL injection risk, A03)."

- `docs/agents/project-manifest/tech-stack.md`
  - Update schema revision from 36 → **37**

- `docs/agents/project-manifest/data-flows.md`
  - Add section "4. Filter & Text Search Flow": text search → in-memory LINQ pass; active filter slot → `IActiveFilterSlotRepository.GetActiveSlotIdAsync` (read from `spdb_config`) → DSL parse → evaluator walk → `Movies` rebuilt; selecting a different slot writes back via `SetActiveSlotIdAsync`.
  - Update the load-error path: `MoviesListViewModel.LoadAsync` failure now sets `HasLoadError`/`LoadErrorMessage` and renders a banner in `MainContentView`.

- `docs/projects/rebuild/milestones/m5-filters-search.md` *(new)*
  - Create the milestone document using the roadmap template; set Status to Active.

- `README.md`
  - No new `appsettings.json` keys to document for M5 (active filter slot lives in `spdb_config`, not `appsettings.json`). Mention the new search/filter UI affordances under the features section if applicable.

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **DSL parser complexity** — hand-rolled recursive descent parser may harbour edge-case bugs (e.g. operator precedence, nested NOT, mixed types) | Parser unit tests cover all operator precedence combinations and edge cases (WP-013) before the evaluator or ViewModel can depend on it |
| **`Description` field ripple** — adding a property to `MovieListItem` could force every test fixture to be updated | Resolved by declaring `Description` as `string Description { get; init; } = ""` instead of `required`; existing constructions compile unchanged |
| **Active slot pointer staleness across DB switches** — eliminated by construction by storing the pointer in `spdb_config` next to the slots themselves; a defensive guard in `LoadFilterSlotsAsync` clears the pointer if the row is missing (e.g. manual DB edit) without surfacing an error |
| **Avalonia compiled bindings for `MoviesListVm.*` in `MainContentView.axaml`** — accessing nested ViewModel properties across compiled binding contexts can produce warnings | Bind using the full typed path `{Binding MoviesListVm.SearchText}` with `x:DataType="vm:MainContentViewModel"` where `MoviesListVm` is typed as `MoviesListViewModel?`; add null-check branches if the compiler warns |
| **Filter Manager `Window` host** — `AvaloniaFilterManagerService` needs a reference to the main window to open a modal dialog | Follow the `AvaloniaConnectionEditorService` pattern exactly; it already resolves the owner window from `App.Current?.ApplicationLifetime` |
| **Security — SQL injection in DSL expressions** — a user could craft an expression string that gets interpolated into SQL | All filtering is in-memory (no expression text is passed to any SQL query). `DapperFilterSlotRepository` stores expressions as opaque text via `@Param` parameters. Zero SQL interpolation risk. |
| **Migration m037 not idempotent** — `CREATE TABLE IF NOT EXISTS` is used, making the migration re-runnable | Acceptable; `IF NOT EXISTS` on `CREATE TABLE` is idempotent. `UPDATE spdb_config` revision bump is not protected but consistent with m036 precedent |
