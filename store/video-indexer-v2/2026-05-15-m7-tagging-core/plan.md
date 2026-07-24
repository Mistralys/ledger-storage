# Plan: M7 — Tagging Core

## Summary

M7 delivers the full tagging subsystem for Video Indexer MKII. This includes defining the
in-memory tag graph (`ITagsManager`), persisting it via `ITagsRepository` /
`DapperTagsRepository`, and shipping the `TaggerView` user control wired into the Movie
Editor's right sidebar stub. Supporting management surfaces include a Tag Editor dialog,
a Category Editor, a Grants Management view, and a Tag Merge dialog. The Filter DSL is
upgraded: `HasTag` / `TagHasSubTags` are activated, `TagsWeight` / `AmountTags` are
upgraded to effective-set semantics, and `StoredTagsWeight` / `AmountStoredTags` are
added as new stored-only identifiers. Schema migration m040 enforces uniqueness constraints
on the tag tables. `LabelCleanerOptions.DetectTags` is activated in `NameParser`. Three
M6 carry-forwards also land here: actor reorder list selection wiring,
`ShowPropertiesCommand` injection into `MovieEditorViewModel`, and a Cover Image tab label
correction.


## Architectural Context

The codebase follows a strict three-layer architecture: **Core** (pure domain — no
external NuGet deps), **Infrastructure** (Dapper/MySqlConnector implementations), **App**
(Avalonia UI, ViewModels, DI host). All cross-layer calls go through Core interfaces;
`Program.cs` is the sole wiring point.

### Relevant existing modules

| Module / File | Role |
|---|---|
| `VideoIndexer.Core/Models/MovieListItem.cs` | List projection served to the grid; currently carries `TagWeight` (stored sum) and `TagCount` (stored count). Both will be renamed and new effective-set fields will be added. |
| `VideoIndexer.Core/Filtering/FilterExpressionParser.cs` | DSL parser; `HasTag` and `TagHasSubTags` are recognised but gated behind `M7Identifiers` and throw a deferred exception. |
| `VideoIndexer.Core/Filtering/FilterExpressionEvaluator.cs` | Static evaluator against `MovieListItem`; must be extended for the four new tag-aware identifiers. |
| `VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs` | `GetMovieListAsync` returns `IReadOnlyList<MovieListItem>`; SQL currently JOINs stored tag weight sum. Must be extended to load stored tag IDs and run the effective-set enrichment pass. |
| `VideoIndexer.App/ViewModels/MovieEditorViewModel.cs` | Has `MoveActorUpCommand`/`MoveActorDownCommand` (logic implemented, no selection wiring) and `ShowPropertiesCommand` (no-op stub; `IMoviePropertiesService` not injected). Right sidebar is a stub `TextBlock`. |
| `VideoIndexer.App/Views/MovieEditorView.axaml` | Three-column layout; right sidebar currently shows stub placeholder text. |
| `VideoIndexer.App/ViewModels/ShellViewModel.cs` | Drives the auth state machine; transitions to `ShellState.Ready` on successful logon or password setup. Tag manager warm-up hooks in here. |
| `spdb_config`, `tags`, `tags_categories`, `tags_grants`, `tags_movies`, `tags_bookmarks` | Existing MariaDB tables; schema revision **39**. Tag tables exist but contain no application data in the rebuild. |

### Spec conflict: filter-management-specification vs tagging-management-specification

`filter-management-specification.md §3.4` describes `TagsWeight` as "Sum of explicitly
assigned tag weights. Granted tags do not contribute." This reflects legacy behaviour.
`tagging-management-specification.md §6.4` explicitly overrides this: `TagsWeight` and
`AmountTags` operate on the **effective** set (stored ∪ grants ∪ parent inheritance) in
the rebuild. The tagging spec is authoritative for M7; the filter spec must be updated to
reflect the new semantics (see Documentation Updates).


## Approach / Architecture

### 1. Core domain layer

Define two new interfaces and supporting models in `VideoIndexer.Core`:

- **`ITagsRepository`** — raw DB CRUD for all five tag tables (`tags`, `tags_categories`,
  `tags_grants`, `tags_movies`, `tags_bookmarks`). Batch-load methods used by
  `TagsManager.EnsureLoadedAsync` also live here. Only `TagsManager` calls this interface
  directly.
- **`ITagsManager`** — the in-memory facade. Owns the loaded tag graph (categories + tags
  + grant edges + parent edges). Exposes read access, effective-set computation,
  mutation (add/edit/delete tag or category, connect/disconnect movie–tag and
  bookmark–tag), merge, cycle detection (`WouldCreateParentCycle`,
  `WouldCreateGrantCycle`), and lazy initialisation via `EnsureLoadedAsync`. `EnsureLoadedAsync`
  must be idempotent and thread-safe (multiple callers during initial list load).

New Core models: `Tag` (sealed record), `TagCategory` (sealed record),
`TagGrant` (sealed record: `long SourceTagId`, `long GrantedTagId`),
`TagDeleteImpact` (counts record for delete preview), `TagMergeImpact` (counts record for
merge preview). New Core enum: `TagImportance` (values 1–5). New Core constant class:
`TagConstants` (`MinWeight = -10`, `MaxWeight = +10`).

### 2. MovieListItem — rename and extension

Rename the existing fields to make semantics explicit, then add the effective-set fields:

| Old field | New field | Semantics |
|---|---|---|
| `TagWeight` | `StoredTagWeight` | SQL `SUM` of stored tag weights only |
| `TagCount` | `StoredTagCount` | SQL `COUNT` of stored tags only |

New fields (non-required init, default to empty/zero so existing call sites and tests
compile without change):

| New field | Type | Source |
|---|---|---|
| `StoredTagIds` | `IReadOnlyList<long>` | Raw IDs from JOIN batch query |
| `StoredTagParentIds` | `IReadOnlyList<long>` | `parent_tag_id` of each stored tag |
| `EffectiveTagIds` | `IReadOnlySet<long>` | Transitive closure via `ITagsManager`; `HashSet`-backed for O(1) `Contains` in the filter evaluator |
| `EffectiveTagWeight` | `int` | Computed via `ITagsManager` |
| `EffectiveTagCount` | `int` | Computed via `ITagsManager` |

`TagsWeight` / `AmountTags` in the Filter DSL now map to `EffectiveTagWeight` /
`EffectiveTagCount` (semantic upgrade, per spec §6.4). The movies-list grid binding for
the Weight column is updated to `EffectiveTagWeight`; the TagCount column binding is updated
to `StoredTagCount`. The `ColumnKeys.TagWeight` string
constant value is kept unchanged for settings backward-compatibility; only the ViewModel
binding target changes.

All callers of the renamed fields are updated in the same work package as the rename:
`DapperMovieCatalogRepository`, `FilterExpressionEvaluator` (including the `HasTags()`
lambda which becomes `StoredTagCount > 0`), `MoviesListView.axaml` (`{Binding TagWeight}`
→ `{Binding EffectiveTagWeight}`, `{Binding TagCount}` → `{Binding StoredTagCount}`), and
all affected test files (see Modified — Tests).

**`Movie` model extension**: `Movie.cs` gains a `StoredTagIds` property
(`IReadOnlyList<long>`, non-required init, defaults to `[]`). `DapperMovieRepository.GetByIdAsync`
is updated to execute a second query (`SELECT tag_id FROM tags_movies WHERE movie_id = @MovieId`)
and populate `StoredTagIds`. This gives `MovieEditorViewModel` direct access to the movie's
stored tag IDs at load time — the value is passed verbatim as `initialStoredTagIds` when
constructing `TaggerViewModel` (see step 22). The query lives in `IMovieRepository` /
`DapperMovieRepository`, not in `ITagsRepository`, so the constraint that only `TagsManager`
calls `ITagsRepository` is preserved.

### 3. Filter DSL activation

Changes are confined to `FilterExpressionParser` and `FilterExpressionEvaluator`:

- **Parser**: remove `HasTag` / `TagHasSubTags` from `M7Identifiers`; add them to the
  single-parameter function set. Add `StoredTagsWeight` and `AmountStoredTags` to
  `NumericIdentifiers`.
- **Evaluator** — updated identifier-to-field mapping:

| DSL identifier | `MovieListItem` field | Change |
|---|---|---|
| `TagsWeight` | `EffectiveTagWeight` | Semantic upgrade (was `TagWeight`) |
| `AmountTags` | `EffectiveTagCount` | Semantic upgrade (was `TagCount`) |
| `StoredTagsWeight` | `StoredTagWeight` | New |
| `AmountStoredTags` | `StoredTagCount` | New |
| `HasTag(id)` | `EffectiveTagIds.Contains((long)id)` | New activation |
| `TagHasSubTags(id)` | `StoredTagParentIds.Contains((long)id)` | New activation |
| `HasTags()` | `StoredTagCount > 0` | Field rename (was `TagCount`; no semantic change) |

The evaluator remains a **purely functional static method** — no `ITagsManager` reference
at evaluation time; all tag data is pre-materialised in `MovieListItem`. This preserves
testability and keeps the filter pipeline side-effect-free.

### 4. Infrastructure — DapperTagsRepository

`VideoIndexer.Infrastructure/Library/DapperTagsRepository.cs` — implements `ITagsRepository`.
Key implementation notes:

- All SQL uses Dapper named parameters; no string-interpolated user data.
- Category CRUD, Tag CRUD (with application-level root-uniqueness pre-check; see §6 Known
  MySQL NULL Limitation), Grant CRUD, Movie–tag and Bookmark–tag connect/disconnect.
- `DeleteTagAsync` cascades in order: remove `tags_movies` rows, remove `tags_grants` rows
  (source or destination), re-parent children (`parent_tag_id` set to the deleted tag's
  parent), then delete the tag row.
- `DeleteCategoryAsync` rejects non-empty categories.
- `GetTagDeleteImpactAsync(long tagId)` returns `TagDeleteImpact` with counts of affected
  movies, bookmarks, grant edges, and children to re-parent — used for the impact-preview
  dialog (§6.12).
- Batch-load methods (`GetAllTagsAsync`, `GetAllCategoriesAsync`, `GetAllGrantsAsync`) are
  used exclusively by `TagsManager.EnsureLoadedAsync`.

### 5. Infrastructure — TagsManager

`VideoIndexer.Infrastructure/Library/TagsManager.cs` — implements `ITagsManager`.

- **Lazy initialisation**: `EnsureLoadedAsync()` is idempotent; on first call it loads all
  rows from `tags`, `tags_categories`, `tags_grants` in three queries. Concurrent callers
  are protected by a `SemaphoreSlim(1,1)` so initialisation runs exactly once.
- **Transitive closure**: computed via iterative BFS/DFS (no recursion, stack-safe) over
  both parent edges and grant edges after each load or mutation. Stored as
  `Dictionary<long, IReadOnlySet<long>> _closure` (key = stored tag ID, value = complete
  reachable set including the tag itself).
- **Cycle detection**: before adding a parent edge or grant edge, walks the **unified
  closure** (combining both parent and grant edge types) to confirm no cycle would form;
  throws `InvalidOperationException` if detected. Mixed-path cycles — where a proposed
  edge creates a cycle only when traversing both parent and grant edges together — are
  therefore covered by both `WouldCreateParentCycle` and `WouldCreateGrantCycle`.
- **Most Used virtual category**: constructed with `CategoryId = 0` sentinel (not
  persisted); `IsMostUsed = true`. Tags included are those with
  `Importance ≤ AppOptions.Tagging.MostUsedThreshold`. `Delete()` on this category is guarded.
- **Quantile-based `TagImportance`** (§6.6): after each load or tag-count change, all tags
  are re-ranked into five `TagImportance` buckets (top 10% → 1, next 25% → 2, next 40%
  → 3, remaining non-zero → 4, zero-use → 5).
- **Weight clamping**: all tag weights are clamped to `[TagConstants.MinWeight,
  TagConstants.MaxWeight]` on read.
- **`DataChanged` event**: fired after any mutation so `MoviesListViewModel` can trigger a
  grid reload.
- **`ISettingsService`** injected for `MostUsedThreshold` access.

### 6. Schema migration m040

```sql
ALTER TABLE tags_categories
    ADD UNIQUE KEY uq_category_name (name);

ALTER TABLE tags
    ADD UNIQUE KEY uq_tag_name (category_id, parent_tag_id, name);
```

> **Known MySQL/MariaDB NULL limitation.** A `UNIQUE` index treats `NULL` values as
> distinct, so `uq_tag_name` does not prevent two root-level tags (both
> `parent_tag_id = NULL`) in the same category from sharing a name. Application-level
> validation in `DapperTagsRepository.CreateTagAsync` and `UpdateTagAsync` must explicitly
> check for name collisions at the root level before issuing the INSERT/UPDATE. The DB
> constraint still protects non-root duplicates and all bulk-edit paths that target
> non-null parents.

`ExpectedRevision` bumped from **39 → 40**.

### 7. DapperMovieCatalogRepository enrichment

`ITagsManager` is injected as a constructor parameter. `GetMovieListAsync` is updated:

1. Execute a single batch query after the primary movies query:
   `SELECT tm.movie_id, tm.tag_id, t.parent_tag_id FROM tags_movies tm JOIN tags t ON t.tag_id = tm.tag_id WHERE tm.movie_id IN (...)`.
   When mapping Dapper rows to `StoredTagParentIds`, filter out any rows where
   `parent_tag_id` is `NULL` (root-level tags); use `long?` for the DB row property and
   assign only non-null values to the `IReadOnlyList<long>` field.
2. Call `await _tagsManager.EnsureLoadedAsync(ct).ConfigureAwait(false)`.
3. For each movie, call `_tagsManager.GetEffectiveTagIds(storedTagIds)` and populate the
   new `MovieListItem` fields.
4. Keep existing `StoredTagWeight` and `StoredTagCount` from the SQL `SUM`/`COUNT`
   in the main query.
5. **Defensive guard**: if the number of tag rows returned for a movie does not match the SQL
   `StoredTagCount`, log a warning and fall back to stored-only values (protects against
   data inconsistency between the batch query and the main query aggregate counts).
6. If `!_tagsManager.IsInitialized` (defensive fallback), enrichment is skipped and stored
   values are used.

### 8. Shell initialisation

In `ShellViewModel.LogOnAsync` (after `VerifyAsync` succeeds) and
`ShellViewModel.SetPasswordAsync` (after `SetAsync`), call
`await _tagsManager.EnsureLoadedAsync(ct).ConfigureAwait(false)` before
`Transition(ShellState.Ready)`. This ensures `TagsManager` is warm when
`MoviesListViewModel.LoadAsync` runs. Add `ITagsManager` as a constructor parameter of
`ShellViewModel`.

### 9. DataChanged reload

In `MoviesListViewModel`, subscribe to `ITagsManager.DataChanged`. When the event fires,
call `LoadAsync(CancellationToken.None)` (fire-and-forget with `.ContinueWith` fault
logging) to refresh the grid with updated effective weights.

To avoid multiple overlapping `GetMovieListAsync` round-trips when a user rapidly checks
several tags in succession, the handler must use a short debounce window (100–200 ms)
before issuing the reload. Implement by cancelling and replacing a `CancellationTokenSource`
on each `DataChanged` firing and calling `Task.Delay` before `LoadAsync`. This requires
no new dependencies.

### 10. Tagger UI (reusable UserControl)

`TaggerView` / `TaggerViewModel` live in `VideoIndexer.App/Views/` and
`VideoIndexer.App/ViewModels/` respectively.

**Reusability contract**: `TaggerViewModel` accepts service dependencies, the initial
stored tag ID list, and two async callbacks at construction:
```csharp
TaggerViewModel(
    ITagsManager tagsManager,
    ITagEditorService tagEditorService,
    ITagMergeService tagMergeService,
    IGrantsManagementService grantsManagementService,
    IReadOnlyList<long> initialStoredTagIds,
    Func<long, CancellationToken, Task> connectTag,
    Func<long, CancellationToken, Task> disconnectTag)
```
The Movie Editor passes movie-specific callbacks (delegating to
`ITagsManager.ConnectMovieTagAsync` / `DisconnectMovieTagAsync`). The M10 Video Player
will pass bookmark-specific callbacks. No Core model changes are needed for M10
reusability.

**Features**:
- `ObservableCollection<TaggerCategoryViewModel> Categories` — one per `TagCategory` plus
  the Most Used virtual category (always first).
- Each `TaggerCategoryViewModel` owns an `ObservableCollection<TaggerTagViewModel>`.
- `TaggerTagViewModel` carries: `Tag Tag`, `bool IsCheckedStored`, `bool IsCheckedImplied`,
  `string? ImpliedViaLabel`, `int Depth` (for flat-list `Margin.Left = Depth * 16`
  indentation), `bool IsVisible`, `bool IsDimmed`, `string ImportanceColor` (computed
  from `Tag.Importance` via a switch expression; rendering concern excluded from Core).
- `string FilterText` — always-visible; on change sets `IsVisible`/`IsDimmed` inline
  without removing items from the collection (§6.8).
- `[RelayCommand] AddNewTagAsync` — opens `ITagEditorService.ShowAsync(new TagEditorViewModel(…))`.
- Right-click context menu commands: `EditTagCommand`, `DeleteTagCommand`,
  `MergeTagCommand` ("Merge into…"), `AddCategoryCommand`, `EditCategoryCommand`,
  `DeleteCategoryCommand`.
- Implied tags (effective-but-not-stored) are displayed with a subdued "via X" label;
  clicking them does not create a redundant `tags_movies` row (§6.9).

**`TaggerView.axaml`** replaces the right-sidebar stub. Uses a `TabControl` for categories
and a `ListBox` with item template. Header toolbar: Add Tag, Add Category, Manage Grants.

### 11. Tag Editor dialog

`TagEditorView` / `TagEditorViewModel` / `ITagEditorService` / `AvaloniaTagEditorService`
— mirrors the existing `LabelCleanerView` dialog pattern. Fields per §3.3/§3.4: Name,
Abbreviation, Category, Parent Tag (picker), Weight (clamped to
`[TagConstants.MinWeight, TagConstants.MaxWeight]`), SortingWeight, Description, Highlight
toggle. Grants section (§3.8): `ObservableCollection<Tag>` with Add (opens a tag-picker
`ContentDialog`) and Remove. Cycle-detection errors shown as inline `TextBlock` beneath
the offending field. `TagEditorViewModel` emits `CloseRequested<Tag?>`. Delete button
shows an impact-count `ContentDialog` (counts from `ITagsManager.GetDeleteImpactAsync`)
before executing.

### 12. Category Editor

`CategoryEditorViewModel` / `CategoryEditorView` — lightweight `ContentDialog` for
create/rename. Rejects empty names and duplicates inline. Default Category and Most Used
virtual category cannot be renamed or deleted (silent guard per spec §3.7). Delete is
blocked if the category contains any tags.

### 13. Grants Management view (§6.10)

`GrantsManagementViewModel` / `GrantsManagementView` / `IGrantsManagementService` /
`AvaloniaGrantsManagementService` — accessible from the Tagger toolbar "Manage Grants"
button. `TaggerViewModel` holds an injected `IGrantsManagementService`; the toolbar button
calls `IGrantsManagementService.ShowAsync()`. Two-column `DataGrid` (Source Tag → Granted
Tag). Add row opens a tag-picker pair; cycle detection runs on Add — cycle-forming grants
are rejected (not warned). Remove row triggers the standard confirmation dialog.

### 14. Tag Merge dialog (§6.11)

`TagMergeViewModel` / `TagMergeView` / `ITagMergeService` / `AvaloniaTagMergeService` —
dialog with Source Tag picker and Target Tag picker. Impact preview (`TagMergeImpact`:
movies affected, bookmarks affected, grant edges rewritten, children to re-parent) loaded
from `ITagsManager.GetMergeImpactAsync` before execute. "Merge into…" context menu entry
on `TaggerTagViewModel` opens this dialog.

### 15. Movie Editor carry-forwards from M6

- **Cover Image tab label**: change `"Cover image panel (M8)"` → `"Cover image panel (M9)"`
  in `MovieEditorView.axaml`.
- **Actor reorder**: replace the actor `TextBox` with a `ListBox` bound to a new
  `ObservableCollection<string> ActorList` property. Rename the existing
  `_selectedActorName` / `SelectedActorName` observable property to `_selectedActor` /
  `SelectedActor` (same semantics — the currently-selected actor string; no logic change
  to `MoveActorUpCommand` / `MoveActorDownCommand` internals beyond using the new name).
  Add `int SelectedActorIndex` observable property (default `-1`; bound to
  `ListBox.SelectedIndex`). `MoveActorUpCommand` `CanExecute` = `SelectedActorIndex > 0`;
  `MoveActorDownCommand` `CanExecute` = `SelectedActorIndex >= 0 && SelectedActorIndex < ActorList.Count - 1`.
  On `LoadAsync`, split `Movie.ActorNames` into `ActorList`. Synchronise `ActorNames` ↔
  `ActorList` on mutations. Remove `IsEnabled="False"` from `MoveActorUp` / `MoveActorDown`
  buttons; bind `CanExecute` to `SelectedActorIndex`.
- **ShowPropertiesCommand**: inject `IMoviePropertiesService` into `MovieEditorViewModel`
  and replace the `Task.CompletedTask` stub with `_moviePropertiesService.ShowAsync(...)`.

### 16. Label Cleaner tag detection

- `INameParser.Parse` gains a **fifth** parameter (appended after the existing `knownStudios`
  parameter): `IReadOnlyList<string> knownTagNames`.
  Callers that do not supply tags pass `[]` (consistent with the existing
  `knownActorNames` / `knownStudios` parameter pattern; keeps `NameParser` stateless).
- `NameParser` populates `LabelCleanerResult.DetectedTags` when
  `options.DetectTags == true`, using the same whole-word boundary match algorithm
  already used for actor detection, applied to `knownTagNames`.
- `LabelCleanerViewModel` injects `ITagsManager` and passes
  `ITagsManager.Tags.Select(t => t.Name).ToArray()` (after `EnsureLoadedAsync`) as
  `knownTagNames` to `INameParser.Parse`.
- `LabelCleanerView.axaml` and `LabelCleanerViewModel` expose the `DetectTags` toggle.
- All existing callers of `INameParser.Parse` (`NameParser.cs`, `LabelCleanerViewModel.cs`,
  `FakeNameParser.cs`, all test call sites) are updated atomically with the interface change.

### 17. AppOptions extension

Add `TaggingOptions` sub-record to `AppOptions`:
```csharp
sealed record TaggingOptions {
    public int MostUsedThreshold { get; init; } = 2;  // Importance ≤ N qualifies for Most Used tab
}
```
Bundled `VideoIndexer.App/Assets/appsettings.json` gains `"Tagging": { "MostUsedThreshold": 2 }`.


## Rationale

The single `ITagsRepository` interface (rather than four split interfaces) is chosen
because all five tag tables form one tightly coupled domain aggregate; splitting them adds
friction in DI registration with no decoupling benefit. `TagsManager` remains the sole
consumer of the repository.

Pre-computing effective-set fields on `MovieListItem` (rather than passing `ITagsManager`
into the filter evaluator at runtime) preserves the evaluator's purely functional contract
and avoids threading concerns when the filter runs on a background thread.

The callback-based `TaggerViewModel` constructor is chosen over a `LoadForMovieAsync(Movie)`
method because it avoids coupling the ViewModel to a concrete entity type, making the
component reusable for bookmark-level tagging in M10 by simply passing different closures.

The mandatory `knownTagNames` parameter on `INameParser.Parse` is chosen over an optional
`ITagsManager?` parameter because it is consistent with the existing `knownActorNames` /
`knownStudios` pattern and keeps `NameParser` stateless — no ordering risk between parser
construction and `TagsManager` initialisation.

Grants Management and Tag Merge are included in M7 scope (rather than deferred to M8)
because both operate directly on the tag graph owned by `ITagsManager`, which is the
defining feature of this milestone.


## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|---|---|---|---|
| Repository interface shape | Single `ITagsRepository` | Four split interfaces (`ITagRepository`, `ITagCategoryRepository`, `ITagGrantRepository`, `ITagMovieRepository`) | The five tag tables are a single domain aggregate consumed only by `TagsManager`; four interfaces add DI noise with no cross-cutting benefit. |
| Effective-set computation | In-memory `TagsManager` + SQL JOIN batch query | SQL recursive CTE | CTE produces complex, fragile SQL with cycle-depth risk; in-memory closure is simpler, already required for the Tagger UI, and recomputes in O(tags) on invalidation. |
| Enrichment integration point | `DapperMovieCatalogRepository` constructor-injected with `ITagsManager` | Separate `ITagEnricher` service in App layer | Repository self-containment means the ViewModel API is unchanged and enrichment is always applied; a separate service would require a parallel VM call and risks being skipped. |
| `TaggerViewModel` reusability | Callback delegates at construction | `LoadForMovieAsync(Movie)` method; `ITaggerTarget` interface on domain models | Callbacks avoid coupling the VM to `Movie`/`Bookmark` concrete types; no Core model changes needed for M10. |
| Tag detection API shape | Mandatory `IReadOnlyList<string> knownTagNames` param on `INameParser.Parse` | Optional `ITagsManager? tagsManager = null` param | Mandatory parameter is consistent with `knownActorNames` / `knownStudios`; keeps `NameParser` stateless; no ordering risk. |
| Cycle detection location | Application layer in `TagsManager` + pre-write guard in repository | DB-level CHECK constraint or trigger | MariaDB has limited support for recursive CHECK constraints; triggers are opaque and hard to test. Application-level check is transparent and unit-testable. |
| Root-level tag uniqueness | App-level guard + partial DB constraint | `COALESCE` virtual column for functional unique index | Virtual columns add schema complexity; the MariaDB NULL-distinctness behaviour means the `UNIQUE` constraint still protects non-root duplicates, and an app-level guard closes the gap. |
| Most Used ownership | `TagsManager` | `TaggerViewModel` (legacy pattern) | Owning the virtual category in `TagsManager` makes it testable, removes the legacy `OrderBy`-discard bug, and ensures consistent guard behaviour regardless of which view opens the tagger. |
| Grants Management + Tag Merge scope | Included in M7 | Deferred to M8 | Both features operate on the tag graph owned by `TagsManager`; deferring them would leave the tag vocabulary management surface incomplete at the end of M7. |


## Pattern Alignment

| Pattern | Alignment |
|---|---|
| Core has no external NuGet dependencies | Followed — all new Core types use only BCL types. |
| Interface-first cross-layer calls | Followed — `ITagsRepository` and `ITagsManager` are Core interfaces; `DapperTagsRepository` and `TagsManager` (Infrastructure) are not referenced directly from App. |
| DI wiring in `Program.cs` only | Followed — all new service registrations go in `Program.cs`. |
| Sealed records for domain models | Followed — `Tag`, `TagCategory`, `TagGrant`, `TagDeleteImpact`, `TagMergeImpact` follow `LibraryFolder`, `Movie`, `FilterSlot`. |
| Dialog service pattern (interface + Avalonia impl) | Followed — `ITagEditorService` / `AvaloniaTagEditorService`, `ITagMergeService` / `AvaloniaTagMergeService`, `IGrantsManagementService` / `AvaloniaGrantsManagementService` mirror `ILabelCleanerService`, `IMoviePropertiesService`. |
| `CloseRequested` event on dialog ViewModels | Followed — `TagEditorViewModel`, `CategoryEditorViewModel`, `TagMergeViewModel` all fire `CloseRequested`. |
| `PageHeaderView` code-behind wiring for toolbar actions | Followed — Tagger header actions (Add Tag, Add Category, Manage Grants) follow the `MoviesListView.axaml.cs` pattern. |
| Compiled bindings (`x:DataType`) | Followed — all new AXAML files use compiled bindings. |
| `.ConfigureAwait(false)` on all async calls in Infrastructure and Core | Followed. |
| `TreatWarningsAsErrors` | Followed — all new code must compile with zero warnings. |
| `AppOptions` is immutable — always use `with { }` + `ISettingsService.SaveAsync` | Followed — no in-place mutation of `AppOptions`. |
| ViewLocator naming convention (`XxxViewModel` → `XxxView`) | Followed — `TaggerViewModel` → `TaggerView`, `TagEditorViewModel` → `TagEditorView`, etc. |
| **Departure — `INameParser.Parse` signature** | Adding a **fifth** `knownTagNames` parameter (appended after `knownStudios`) is a breaking change. All call sites (`NameParser.cs`, `LabelCleanerViewModel.cs`, `FakeNameParser.cs`, all test files) must be updated atomically. This is the smallest shape that activates tag detection without making `NameParser` stateful. |
| **Departure — `TagsManager` is a stateful singleton in Infrastructure** | Infrastructure previously contained only thin Dapper repositories. `TagsManager` is an in-memory aggregate that implements business logic (closure computation, importance bucketing, cycle detection). Justified by the precedent of `RefreshOrchestrator` (state machine in Infrastructure) and the fact that the logic does not belong in a thin repository. |


## Detailed Steps

1. **Core: Tag and TagCategory models** — Create `Tag` sealed record and `TagCategory`
   sealed record in `VideoIndexer.Core/Models/`. `Tag` carries all columns from `tags`
   plus computed properties `Importance` (`TagImportance` enum), `NameWithPath` (full
   display name including parent chain). `ImportanceColor` is a rendering concern and is
   excluded from the Core model; it is computed as a property on `TaggerTagViewModel` via
   a switch expression over `Tag.Importance`. `TagCategory` carries `long CategoryId`,
   `string Name`, `bool IsDefault`, `bool IsMostUsed`.

2. **Core: TagGrant, TagDeleteImpact, TagMergeImpact models** — Create in
   `VideoIndexer.Core/Models/`. `TagGrant`: `long SourceTagId`, `long GrantedTagId`.
   `TagDeleteImpact`: `int AffectedMovies`, `int AffectedBookmarks`, `int AffectedGrants`,
   `int ChildrenToReparent`. `TagMergeImpact`: same shape.

3. **Core: TagImportance enum + TagConstants** — Add `TagImportance` enum (values 1–5)
   to `VideoIndexer.Core/Enums/`. Add `TagConstants` static class (`MinWeight = -10`,
   `MaxWeight = 10`) to `VideoIndexer.Core/`.

4. **Core: AppOptions.TaggingOptions** — Add `TaggingOptions` sealed record with
   `int MostUsedThreshold { get; init; } = 2`. Add `TaggingOptions Tagging` property
   to `AppOptions`. Update `VideoIndexer.App/Assets/appsettings.json` with
   `"Tagging": { "MostUsedThreshold": 2 }`.

5. **Core: ITagsRepository** — Define `ITagsRepository` in
   `VideoIndexer.Core/Abstractions/` with full CRUD and batch-load methods for all five
   tag tables. Include `GetTagDeleteImpactAsync(long tagId)` and
   `GetMergeImpactAsync(long sourceTagId, long targetTagId)`.

6. **Core: ITagsManager** — Define `ITagsManager` in `VideoIndexer.Core/Abstractions/`
   with: read properties (`Categories`, `Tags`, `bool IsInitialized`); `EnsureLoadedAsync`;
   mutation methods (`CreateTagAsync`, `UpdateTagAsync`, `DeleteTagAsync`, `MergeTagAsync`,
   `CreateCategoryAsync`, `UpdateCategoryAsync`, `DeleteCategoryAsync`, `AddGrantAsync`,
   `RemoveGrantAsync`); association methods (`ConnectMovieTagAsync`,
   `DisconnectMovieTagAsync`, `ConnectBookmarkTagAsync`, `DisconnectBookmarkTagAsync`);
   effective-set helpers (`GetEffectiveTagIds(IEnumerable<long>)`,
   `GetDeleteImpactAsync(long tagId) → TagDeleteImpact`,
   `GetMergeImpactAsync(long, long) → TagMergeImpact`); cycle guards
   (`WouldCreateParentCycle`, `WouldCreateGrantCycle`); `DataChanged` event.

7. **Core: MovieListItem rename and extension; Movie StoredTagIds** — Rename
   `TagWeight → StoredTagWeight`, `TagCount → StoredTagCount`. Add `StoredTagIds`,
   `StoredTagParentIds`, `EffectiveTagIds`, `EffectiveTagWeight`, `EffectiveTagCount`
   (all non-required init, with default empty/zero values). `StoredTagParentIds` must be
   populated from **non-null** `parent_tag_id` values only — root-level tags have
   `parent_tag_id = NULL` in the DB; filter these out before constructing the list.
   Update `ColumnKeys.TagWeight`
   XML comment to indicate the string key is unchanged for settings compat but the ViewModel
   binding target changes. Update all existing callers of the renamed fields in the same step,
   including: `MoviesListView.axaml` (`{Binding TagWeight}` → `{Binding EffectiveTagWeight}`,
   `{Binding TagCount}` → `{Binding StoredTagCount}`); `FilterExpressionEvaluator`
   `s_numericIdentifiers` entries and the `HasTags()` entry in `s_functions`
   (`i.TagCount > 0` → `i.StoredTagCount > 0`); and all test files that construct or
   mutate `MovieListItem` with these fields (see Modified — Tests).
   Also add `StoredTagIds` (`IReadOnlyList<long>`, non-required init, defaults to `[]`) to
   `Movie.cs`. Update `DapperMovieRepository.GetByIdAsync` to execute a second query
   (`SELECT tag_id FROM tags_movies WHERE movie_id = @MovieId`) after the main SELECT and
   populate `Movie.StoredTagIds` from the results.

8. **Core: Filter DSL** — Update `FilterExpressionParser`: remove `HasTag` /
   `TagHasSubTags` from `M7Identifiers`, add to `FunctionIdentifiers` (single-arg
   numeric). Add `StoredTagsWeight` and `AmountStoredTags` to `NumericIdentifiers`.
   Update `FilterExpressionEvaluator`: rewire `TagsWeight → EffectiveTagWeight`,
   `AmountTags → EffectiveTagCount`; update `HasTags()` → `StoredTagCount > 0` (field
   rename, no semantic change); add `StoredTagsWeight → StoredTagWeight`,
   `AmountStoredTags → StoredTagCount`, `HasTag(id) → EffectiveTagIds.Contains((long)id)`,
   `TagHasSubTags(id) → StoredTagParentIds.Contains((long)id)`.

9. **Core: INameParser.Parse signature update** — Add `IReadOnlyList<string> knownTagNames`
   as the **fifth parameter, appended after the existing `knownStudios` parameter**. Update
   `NameParser`, `FakeNameParser`, `LabelCleanerViewModel`,
   and all test call sites atomically. `NameParser` tag-detection step activates when
   `options.DetectTags == true`, using whole-word boundary matching against `knownTagNames`.
   `LabelCleanerResult.DetectedTags` is populated.

10. **Infrastructure: DapperTagsRepository** — Create
    `VideoIndexer.Infrastructure/Library/DapperTagsRepository.cs` implementing
    `ITagsRepository`. Key notes: application-level root-uniqueness check in
    `CreateTagAsync` / `UpdateTagAsync` (see §6 NULL limitation); cascade delete order per
    Approach §4; `GetTagDeleteImpactAsync` returns counts for impact preview.

11. **Infrastructure: TagsManager** — Create
    `VideoIndexer.Infrastructure/Library/TagsManager.cs` implementing `ITagsManager`.
    Features per Approach §5: lazy `EnsureLoadedAsync` with `SemaphoreSlim(1,1)`, iterative
    BFS/DFS closure, cycle detection, Most Used virtual category (sentinel `CategoryId = 0`),
    quantile importance, weight clamping, `DataChanged` event, `ISettingsService` injection.

12. **Infrastructure: DapperMovieCatalogRepository enrichment** — Add `ITagsManager`
    constructor parameter. Update `GetMovieListAsync` per Approach §7 (single JOIN batch
    query, `EnsureLoadedAsync`, effective-set enrichment, defensive guard on tag-count
    mismatch, fallback to stored values when `!IsInitialized`).

13. **Infrastructure: Schema migration m040** — Create
    `VideoIndexer.Infrastructure/Database/migrations/m040_add_tag_uniqueness.sql` with the
    two `ADD UNIQUE KEY` statements from Approach §6. Bump
    `DatabaseBootstrapper.ExpectedRevision` from `39` to `40` (literal, not expression).
    Add `{ 40, "VideoIndexer.Infrastructure.Database.migrations.m040_add_tag_uniqueness.sql" }`
    to the `Migrations` dictionary in `DatabaseBootstrapper.cs` (alongside the existing entries
    for revisions 36–39). Without this entry `ApplyMigrationsAsync` throws
    `InvalidOperationException` at startup for any database at revision 39.
    Update the `ExpectedRevision_MatchesCurrentSchemaRevision` sentinel test literal.

14. **App: ShellViewModel update** — Add `ITagsManager` constructor parameter. In
    `LogOnAsync` and `SetPasswordAsync`, call
    `await _tagsManager.EnsureLoadedAsync(ct).ConfigureAwait(false)` before
    `Transition(ShellState.Ready)`.

15. **App: MoviesListViewModel update** — Subscribe to `ITagsManager.DataChanged`;
    fire-and-forget grid reload on the event. Both `MoviesListViewModel` and `TagsManager`
    are session-lifetime singletons, so no unsubscription is required (they share the
    same DI scope). Implement a 100–200 ms debounce in the `DataChanged` handler: cancel
    and replace a `CancellationTokenSource` on each event firing; call `Task.Delay` before
    issuing `LoadAsync` so that only the final event in a rapid sequence triggers a reload.

16. **App: TaggerViewModel + TaggerCategoryViewModel + TaggerTagViewModel** — Implement
    per Approach §10. Constructor accepts callbacks. `FilterText` dims/undims inline. Right-
    click commands delegate to service interfaces.

17. **App: TaggerView** — Create `VideoIndexer.App/Views/TaggerView.axaml` and `.cs`.
    `TabControl` for categories; `ListBox` with flat-indent item template for tags. Header
    toolbar with Add Tag, Add Category, Manage Grants buttons.

18. **App: TagEditorViewModel + TagEditorView + ITagEditorService** — Dialog per Approach
    §11. Implements `CloseRequested<Tag?>`, weight clamping, cycle detection feedback,
    grants `ObservableCollection<TagGrantRowViewModel>` (each row is a
    `TagGrantRowViewModel(Tag grantedTag, IRelayCommand removeCommand)` wrapper that exposes
    the granted tag for display and a bound Remove command), delete impact preview.

19. **App: CategoryEditorViewModel + CategoryEditorView** — Lightweight `ContentDialog`
    per Approach §12. Guards Default Category and Most Used from rename/delete.

20. **App: GrantsManagementViewModel + GrantsManagementView + IGrantsManagementService** — Two-column `DataGrid` view
    per Approach §13. `IGrantsManagementService` / `AvaloniaGrantsManagementService` follow the existing
    dialog service pattern. Cycle detection on Add; silent rejection of duplicates.

21. **App: TagMergeViewModel + TagMergeView + ITagMergeService** — Dialog per Approach
    §14. Impact preview before execute. "Merge into…" context menu wired in `TaggerTagViewModel`.

22. **App: MovieEditorViewModel carry-forwards** — (a) Inject `IMoviePropertiesService`;
    wire `ShowPropertiesCommand`. (b) Rename `SelectedActorName` → `SelectedActor`; add
    `ActorList` and `SelectedActorIndex`; synchronise with `ActorNames`; enable actor
    reorder commands (see step 15 for full detail). (c) Inject `ITagsManager`;
    expose `TaggerViewModel TaggerVm` property initialised on movie load with movie-specific
    callbacks. Pass `movie.StoredTagIds` (populated by `DapperMovieRepository.GetByIdAsync`
    per step 7) as `initialStoredTagIds`; pass closures that capture `_originalMovie.MovieId`
    as the connect/disconnect callbacks — e.g.
    `connectTag: (tagId, ct) => _tagsManager.ConnectMovieTagAsync(_originalMovie.MovieId, tagId, ct)`,
    `disconnectTag: (tagId, ct) => _tagsManager.DisconnectMovieTagAsync(_originalMovie.MovieId, tagId, ct)`.
    (`ConnectMovieTagAsync` / `DisconnectMovieTagAsync` take three arguments: `movieId`,
    `tagId`, `CancellationToken`; the callback shape is two-argument `Func<long, CancellationToken, Task>`.)

23. **App: MovieEditorView.axaml carry-forwards** — (a) Replace right-sidebar stub with
    `<views:TaggerView DataContext="{Binding TaggerVm}" />`. (b) Replace actor `TextBox`
    with `ListBox`; bind `SelectedIndex` to `SelectedActorIndex`; remove
    `IsEnabled="False"` from `MoveActorUp` / `MoveActorDown`. (c) Fix Cover Image tab
    label `"Cover image panel (M8)"` → `"Cover image panel (M9)"`.

24. **App: LabelCleanerViewModel update** — Add `ITagsManager? tagsManager = null` as a
    **nullable optional** constructor parameter (appended after the existing `knownStudios`
    parameter; the `= null` default ensures all existing 5-argument call sites compile
    without change). **Do not call `EnsureLoadedAsync` at parse time** — `RunParse()` and
    its property-change callers are synchronous; calling an async method there risks a
    UI-thread deadlock. Instead, cache the tag names once during construction:
    `_knownTagNames = tagsManager?.IsInitialized == true ? tagsManager.Tags.Select(t => t.Name).ToArray() : []`.
    Use `_knownTagNames` in every `RunParse()` call. Wire the Detected Tags display row in
    `LabelCleanerView`. Also update the `MovieEditorViewModel.CleanLabelCommand` call site
    (`VideoIndexer.App/ViewModels/MovieEditorViewModel.cs` line ~383) to pass `_tagsManager`
    as the new nullable `ITagsManager?` argument when constructing `LabelCleanerViewModel`.

25. **App: Program.cs DI registrations** — Register `ITagsRepository → DapperTagsRepository`
    (singleton); `ITagsManager → TagsManager` (singleton); `ITagEditorService →
    AvaloniaTagEditorService`; `ITagMergeService → AvaloniaTagMergeService`;
    `IGrantsManagementService → AvaloniaGrantsManagementService`;
    `TaggerView` (transient); `TagEditorView` (transient); `CategoryEditorView` (transient);
    `GrantsManagementView` (transient); `TagMergeView` (transient). Update factory
    lambdas for `DapperMovieCatalogRepository`, `ShellViewModel`,
    `MoviesListViewModel`, `MovieEditorViewModel`, and `LabelCleanerViewModel` to inject
    the new dependencies.

26. **Tests and Documentation** — See Test Plan and Documentation Updates sections.


## Dependencies

- M6 must be complete (Movie Editor shell with right-sidebar stub, `MovieEditorViewModel`
  with `MoveActorUpCommand` logic, `IMoviePropertiesService` + `AvaloniaMoviePropertiesService`
  already implemented, `INameParser` and `NameParser` present, schema revision 39).
- No new NuGet packages required — Dapper, MySqlConnector, CommunityToolkit.Mvvm, and
  Avalonia are already declared in `Directory.Packages.props`.


## Required Components

### New — Core
- `VideoIndexer.Core/Models/Tag.cs`
- `VideoIndexer.Core/Models/TagCategory.cs`
- `VideoIndexer.Core/Models/TagGrant.cs`
- `VideoIndexer.Core/Models/TagDeleteImpact.cs`
- `VideoIndexer.Core/Models/TagMergeImpact.cs`
- `VideoIndexer.Core/Enums/TagImportance.cs`
- `VideoIndexer.Core/TagConstants.cs`
- `VideoIndexer.Core/Options/TaggingOptions.cs`
- `VideoIndexer.Core/Abstractions/ITagsRepository.cs`
- `VideoIndexer.Core/Abstractions/ITagsManager.cs`

### Modified — Core
- `VideoIndexer.Core/Models/Movie.cs` — add `StoredTagIds` field
- `VideoIndexer.Core/Models/MovieListItem.cs` — rename + new fields
- `VideoIndexer.Core/Options/AppOptions.cs` — add `TaggingOptions Tagging`
- `VideoIndexer.Core/Abstractions/INameParser.cs` — add `knownTagNames` parameter
- `VideoIndexer.Core/Filtering/FilterExpressionParser.cs` — remove M7 guards
- `VideoIndexer.Core/Filtering/FilterExpressionEvaluator.cs` — rewire identifier mappings

### New — Infrastructure
- `VideoIndexer.Infrastructure/Library/DapperTagsRepository.cs`
- `VideoIndexer.Infrastructure/Library/TagsManager.cs`
- `VideoIndexer.Infrastructure/Database/migrations/m040_add_tag_uniqueness.sql`

### Modified — Infrastructure
- `VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs` — enrichment extension
- `VideoIndexer.Infrastructure/Library/DapperMovieRepository.cs` — `GetByIdAsync` extended to populate `StoredTagIds`
- `VideoIndexer.Infrastructure/Library/NameParser.cs` — `DetectTags` activation
- `VideoIndexer.Infrastructure/Database/DatabaseBootstrapper.cs` — `ExpectedRevision = 40`

### New — App (ViewModels)
- `VideoIndexer.App/ViewModels/TaggerViewModel.cs`
- `VideoIndexer.App/ViewModels/TaggerCategoryViewModel.cs`
- `VideoIndexer.App/ViewModels/TaggerTagViewModel.cs`
- `VideoIndexer.App/ViewModels/TagEditorViewModel.cs`
- `VideoIndexer.App/ViewModels/TagGrantRowViewModel.cs`
- `VideoIndexer.App/ViewModels/CategoryEditorViewModel.cs`
- `VideoIndexer.App/ViewModels/GrantsManagementViewModel.cs`
- `VideoIndexer.App/ViewModels/TagMergeViewModel.cs`

### New — App (Views)
- `VideoIndexer.App/Views/TaggerView.axaml` / `.cs`
- `VideoIndexer.App/Views/TagEditorView.axaml` / `.cs`
- `VideoIndexer.App/Views/CategoryEditorView.axaml` / `.cs`
- `VideoIndexer.App/Views/GrantsManagementView.axaml` / `.cs`
- `VideoIndexer.App/Views/TagMergeView.axaml` / `.cs`

### New — App (Services)
- `VideoIndexer.App/Services/ITagEditorService.cs`
- `VideoIndexer.App/Services/AvaloniaTagEditorService.cs`
- `VideoIndexer.App/Services/ITagMergeService.cs`
- `VideoIndexer.App/Services/AvaloniaTagMergeService.cs`
- `VideoIndexer.App/Services/IGrantsManagementService.cs`
- `VideoIndexer.App/Services/AvaloniaGrantsManagementService.cs`

### Modified — App
- `VideoIndexer.App/Assets/appsettings.json` — add Tagging section
- `VideoIndexer.App/Program.cs` — new DI registrations
- `VideoIndexer.App/ViewModels/MovieEditorViewModel.cs` — carry-forwards, `TaggerVm`, actor list, `ITagsManager` injection
- `VideoIndexer.App/ViewModels/ShellViewModel.cs` — `ITagsManager` injection, `EnsureLoadedAsync` call
- `VideoIndexer.App/ViewModels/MoviesListViewModel.cs` — `DataChanged` subscription
- `VideoIndexer.App/ViewModels/LabelCleanerViewModel.cs` — `ITagsManager` injection, `knownTagNames`
- `VideoIndexer.App/Views/MovieEditorView.axaml` — Tagger wiring, actor `ListBox`, tab label fix
- `VideoIndexer.App/Views/LabelCleanerView.axaml` — `DetectTags` checkbox
- `VideoIndexer.App/Views/MoviesListView.axaml` — update `{Binding TagWeight}` → `{Binding EffectiveTagWeight}`; update `{Binding TagCount}` → `{Binding StoredTagCount}`

### New — Tests
- `tests/VideoIndexer.Tests/TagsManagerTests.cs`
- `tests/VideoIndexer.Tests/FilterExpressionEvaluatorM7Tests.cs`
- `tests/VideoIndexer.Tests/NameParserDetectTagsTests.cs`
- `tests/VideoIndexer.App.Tests/TaggerViewModelTests.cs`
- `tests/VideoIndexer.App.Tests/TagEditorViewModelTests.cs`
- `tests/VideoIndexer.App.Tests/TagMergeViewModelTests.cs`
- `tests/VideoIndexer.App.Tests/GrantsManagementViewModelTests.cs`
- `tests/VideoIndexer.App.Tests/CategoryEditorViewModelTests.cs`
- `tests/VideoIndexer.Infrastructure.Tests/DapperTagsRepositoryTests.cs`
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeTagsManager.cs`
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeMoviePropertiesService.cs`
  — implements `IMoviePropertiesService`; exposes `bool WasShowCalled` for assertion;
  follows the `FakeLabelCleanerService` call-tracking pattern.
  > Note: `FakeTagsManager` and `FakeMoviePropertiesService` live in `VideoIndexer.App.Tests`
  > only. Infrastructure tests (`DapperMovieCatalogRepositoryTests.cs`,
  > `LibraryScannerIntegrationTests.cs`) use `new Mock<ITagsManager>()` via Moq (already
  > available in that project) — they do not reference these fakes.

### Modified — Tests
- `tests/VideoIndexer.App.Tests/MovieEditorViewModelTests.cs` — new carry-forward tests; rename `vm.SelectedActorName` → `vm.SelectedActor` in all existing actor-reorder test call sites
- `tests/VideoIndexer.App.Tests/LabelCleanerViewModelTests.cs` — add two new `DetectTags` test cases (see Test Plan)
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeNameParser.cs` — updated signature
- `tests/VideoIndexer.Infrastructure.Tests/DatabaseBootstrapperTests.cs` — sentinel literal `40`
- `tests/VideoIndexer.Tests/Filtering/FilterExpressionEvaluatorTests.cs` — rename `TagWeight → StoredTagWeight`, `TagCount → StoredTagCount` in `DefaultItem()` and all `with`-expressions
- `tests/VideoIndexer.App.Tests/MoviesListViewModelTests.cs` — rename `TagWeight → StoredTagWeight`, `TagCount → StoredTagCount` in `DefaultItem()` (note: `ColumnKeys.TagWeight` / `ColumnKeys.TagCount` string constants are unchanged)
- `tests/VideoIndexer.App.Tests/MainContentViewModelTests.cs` — rename `TagWeight → StoredTagWeight`, `TagCount → StoredTagCount` in `MovieListItem` initialisers
- `tests/VideoIndexer.App.Tests/MoviesListViewModelSearchFilterTests.cs` — rename `TagWeight → StoredTagWeight`, `TagCount → StoredTagCount` in `DefaultItem()`
- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperMovieCatalogRepositoryTests.cs` —
  update `row.TagWeight` assertion to `row.StoredTagWeight`; update `CreateSut()` and
  `CreateGuardSut()` factory methods to pass `new Mock<ITagsManager>().Object` as the new
  second constructor argument added in Step 12
- `tests/VideoIndexer.Infrastructure.Tests/Library/LibraryScannerIntegrationTests.cs` —
  update the two `DapperMovieCatalogRepository` construction sites in `CreateCatalog()` (lines ~98
  and ~107) to pass `new Mock<ITagsManager>().Object` as the second constructor argument
- `tests/VideoIndexer.Tests/Filtering/FilterExpressionParserTests.cs` — remove or invert `Parse_DeferredHasTag_ThrowsWithM7Message`; replace with an assertion that `FilterExpressionParser.Parse("HasTag(1)")` no longer throws
- `tests/VideoIndexer.Tests/LabelCleanerTests.cs` — remove `Parse_DetectedTags_AlwaysEmpty_UntilM7`; append `knownTagNames: []` to all existing `Sut.Parse(...)` call sites in this file


## Assumptions

- The five legacy tagging tables (`tags_categories`, `tags`, `tags_grants`, `tags_movies`,
  `tags_bookmarks`) exist at every schema revision ≥ 35 (they predate revision 35, the
  lowest migratable revision). Migration m040 therefore only adds indexes and cannot fail
  due to missing tables.
- `tags_bookmarks` exists in the schema but has no application data — DB-layer read/write
  operations are safe to implement without data-migration concerns.
- The tag graph (categories, tags, grants) is small enough (hundreds of rows) to hold
  entirely in memory without memory pressure.
- `ITagsManager` is registered as a **singleton** in DI (one shared instance per
  application session, lazily initialised, invalidated on mutation).
- `tags` table uses `long` / `bigint` primary keys (confirmed by DB schema); IDs are
  typed as `long` throughout.


## Constraints

- `VideoIndexer.Core` must not gain any external NuGet dependency.
- New `<PackageReference>` entries must not include a `Version` attribute; versions live
  exclusively in `Directory.Packages.props`.
- All warnings are errors (`TreatWarningsAsErrors = true`). Zero new warnings permitted.
- Nullable reference types are enabled — all new nullable flows must be explicitly handled.
- `FilterExpressionEvaluator.Evaluate` must remain a static method.
- `M10Identifiers` deferred guard (`HasRatedBookmarks`, `BookmarkContains`,
  `AmountBookmarks`) must NOT be removed — these remain deferred until M10.
- `DatabaseBootstrapper.ExpectedRevision` sentinel literal in `DatabaseBootstrapperTests.cs`
  must be updated from `39` to `40`; it must not be replaced with an expression.
- `ITagsManager.EnsureLoadedAsync` must be idempotent and thread-safe.
- `AppOptions` must never be mutated in-place; use `with { }` + `ISettingsService.SaveAsync`.
- `ILibraryScanner.RefreshAsync` cooperative-cancellation contract is unchanged.
- `visualtags_*` tables must not be referenced by any new code (per tagging spec §1).
- `tags_bookmarks` bookmark-tag association methods are implemented at the DB layer in M7
  but the UI wiring from the Video Player is M10. No `ITagBookmarkRepository` is exposed
  directly to App.


## Out of Scope

- Bookmark model (`Bookmark`), `IBookmarkRepository`, and the Video Player Tagger context
  — all deferred to M10. Bookmark-tag DB methods are implemented in `ITagsManager` /
  `DapperTagsRepository` but no UI is built.
- Preferences dialog UI for `MostUsedThreshold` — the option is read from
  `appsettings.json` but no settings panel is built until M8.
- Cover Image, Thumbnails, and Video tabs in the Movie Editor — deferred to M9/M10.
- `ffprobe`-dependent fields in `MoviePropertiesViewModel` — deferred to M9.
- Multi-Edit — dropped per rebuild.md feature pruning.
- Graphical grant-graph visualisation — the Grants Management view is a simple two-column
  list; a graph visualisation is a future polish item.


## Acceptance Criteria

- [ ] The Movie Editor right sidebar shows the Tagger control with category tabs (Most Used
  first), tag checkboxes with hierarchical indentation, and the inline filter field.
- [ ] Checking a tag inserts a row into `tags_movies`; unchecking removes it. The
  `EffectiveTagWeight` column in the movies grid updates after the grid reload.
- [ ] Tags reached via grant or parent inheritance appear as "checked-by-implication" with
  a subdued "via X" indicator; clicking them does not insert a duplicate `tags_movies` row.
- [ ] The Tag Editor dialog creates, edits, and deletes tags. Weight is clamped to
  `[-10, +10]` on save. Parent-cycle assignments are rejected with an inline error.
- [ ] Deleting a tag shows an impact-count confirmation dialog before executing.
- [ ] Category CRUD: add, rename (blocked for Default and Most Used), delete (blocked if
  non-empty).
- [ ] The Grants Management view shows all grant rules; adding a cycle-forming grant is
  rejected.
- [ ] The Tag Merge dialog previews impact counts and executes the merge; the source tag
  is removed from the vocabulary afterward.
- [ ] Filter DSL: `HasTag(N)`, `TagHasSubTags(N)`, `StoredTagsWeight > X`,
  `AmountStoredTags < Y` parse and evaluate without error.
- [ ] `TagsWeight` and `AmountTags` in the DSL reflect the effective set (stored + grants
  + parent inheritance); `StoredTagsWeight` and `AmountStoredTags` reflect stored only.
- [ ] `LabelCleanerOptions.DetectTags = true` populates `DetectedTags` in the Label
  Cleaner dialog.
- [ ] MoveActorUp / MoveActorDown buttons are enabled and functional when an actor is
  selected in the actor list.
- [ ] Movie Properties dialog opens from the Movie Editor toolbar.
- [ ] Schema revision is 40 after migration m040 runs on a revision-39 database.
- [ ] `dotnet build` produces 0 errors, 0 warnings.
- [ ] `dotnet test` shows all existing and new tests passing.


## Testing Strategy

Unit tests (no external deps) cover `TagsManager` closure computation, cycle detection,
importance bucketing, Most Used category behaviour, and weight clamping — all with
in-memory fakes. The filter DSL evaluator is tested with synthetic `MovieListItem`
instances. App-layer ViewModels (`TaggerViewModel`, `TagEditorViewModel`,
`TagMergeViewModel`, `MovieEditorViewModel` carry-forwards) are unit-tested via
`Avalonia.Headless.XUnit` with `FakeTagsManager`. Integration tests (self-skipping without
a DB connection) cover `DapperTagsRepository` CRUD round-trips and
`DapperMovieCatalogRepository.GetMovieListAsync` effective-set enrichment.


## Test Plan

### Core / Logic Unit Tests (`VideoIndexer.Tests/`)

**`TagsManagerTests.cs`**
- `GetEffectiveTagIds_SingleTag_ReturnsSelf` — base case; no grants/parents → `{tagId}`. *(AC: effective-set semantics)*
- `GetEffectiveTagIds_SingleHopGrant_IncludesGrantedTag` — A grants B; stored `{A}` → effective `{A,B}`. *(AC: DSL HasTag)*
- `GetEffectiveTagIds_TransitiveGrant_FollowsChain` — A→B→C; stored `{A}` → effective `{A,B,C}`. *(AC: transitive closure)*
- `GetEffectiveTagIds_ParentInheritance_IncludesAncestors` — child tag stored; parent and grandparent appear in effective set. *(AC: parent inheritance)*
- `GetEffectiveTagIds_MixedGrantAndParent_CombinedClosure` — both edge types followed. *(AC: effective-set semantics)*
- `ComputeEffectiveWeight_StoredWeightOutOfRange_ClampsOnRead` — tag weight stored as 15; clamped to 10. *(AC: weight clamp)*
- `WouldCreateParentCycle_DirectSelfReference_ReturnsTrue` — propose A.parent = A. *(AC: cycle rejection)*
- `WouldCreateParentCycle_TransitiveCycle_ReturnsTrue` — A→B→C; propose C.parent = A. *(AC: cycle rejection)*
- `WouldCreateGrantCycle_TransitiveCycle_ReturnsTrue` — A grants B, B grants C; propose C grants A. *(AC: cycle rejection)*
- `MostUsedCategory_IncludesTagsWithinThreshold_ExcludesAbove` — threshold = 2; tags with Importance 1 and 2 appear. *(AC: Most Used tab)*
- `MostUsedCategory_DeleteGuard_Throws` — `DeleteCategoryAsync` on Most Used throws. *(AC: guard)*
- `Importance_QuantileBucketing_CorrectFiveLevels` — five tags with known `MoviesTotal` values; all five levels assigned correctly. *(AC: importance display)*
- `OrderBy_TagsReturnedSorted_NotDiscarded` — regression test for legacy `OrderBy`-discard bug.

**`FilterExpressionEvaluatorM7Tests.cs`**
- `HasTag_DirectStored_ReturnsTrue` — `EffectiveTagIds={1}`, `HasTag(1)` → true. *(AC: DSL HasTag)*
- `HasTag_ViaEffectiveSet_ReturnsTrue` — `EffectiveTagIds={1,2}`, `HasTag(2)` → true. *(AC: DSL HasTag effective set)*
- `HasTag_NotPresent_ReturnsFalse` — `HasTag(99)` → false. *(AC: DSL HasTag)*
- `TagHasSubTags_StoredChildExists_ReturnsTrue` — `StoredTagParentIds={5}`, `TagHasSubTags(5)` → true. *(AC: DSL TagHasSubTags)*
- `TagHasSubTags_NoMatch_ReturnsFalse` — `StoredTagParentIds=[]`, `TagHasSubTags(5)` → false. *(AC: DSL TagHasSubTags)*
- `TagsWeight_MapsToEffectiveTagWeight` — `EffectiveTagWeight=8`, `TagsWeight > 5` → true. *(AC: semantic upgrade)*
- `AmountTags_MapsToEffectiveTagCount` — `EffectiveTagCount=3`, `AmountTags = 3` → true. *(AC: semantic upgrade)*
- `StoredTagsWeight_MapsToStoredTagWeight` — `StoredTagWeight=3`, `StoredTagsWeight < 5` → true. *(AC: DSL StoredTagsWeight)*
- `AmountStoredTags_MapsToStoredTagCount` — `StoredTagCount=2`, `AmountStoredTags = 2` → true. *(AC: DSL AmountStoredTags)*
- `HasTag_ParseM7GuardRemoved_DoesNotThrow` — `FilterExpressionParser.Parse("HasTag(1)")` no longer throws. *(AC: guard removal)*
- `TagHasSubTags_ParseM7GuardRemoved_DoesNotThrow` — same for `TagHasSubTags(1)`. *(AC: guard removal)*

**`NameParserDetectTagsTests.cs`**
- `Parse_DetectTagsTrue_DetectsKnownTagInFilename` — filename contains tag name; `DetectedTags` populated. *(AC: DetectTags activated)*
- `Parse_DetectTagsFalse_AlwaysEmpty` — opt-in behaviour enforced. *(AC: DetectTags opt-in)*
- `Parse_DetectTagsTrue_EmptyKnownTags_ReturnsEmpty` — graceful empty input. *(AC: DetectTags graceful)*

### App-Layer ViewModel Tests (`VideoIndexer.App.Tests/`)

**`TaggerViewModelTests.cs`**
- `Constructor_LoadsCategoriesFromTagsManager` — mock `ITagsManager`; VM exposes expected categories. *(AC: Tagger categories)*
- `Constructor_MostUsedCategory_IsFirst` — Most Used virtual category appears first. *(AC: Most Used first)*
- `ConnectTag_CallsConnectCallback` — connect callback invoked; `IsCheckedStored` true. *(AC: tag connection)*
- `ConnectTag_ImpliedTag_DoesNotCallConnectCallback` — implied tag checked; callback not invoked. *(AC: implied tag not double-stored)*
- `DisconnectTag_CallsDisconnectCallback` — disconnect callback invoked; `IsCheckedStored` false. *(AC: tag disconnection)*
- `FilterText_DimsNonMatchingTags_KeepsMatchingVisible` — two tags; filter narrows. *(AC: inline filter)*
- `TaggerTagViewModel_ImpliedTag_ShowsViaLabel` — `IsCheckedImplied=true`, `ImpliedViaLabel` populated. *(AC: "via X" indicator)*

**`TagEditorViewModelTests.cs`**
- `SaveTag_ClampsWeightAboveMax` — Weight=11 → clamped to 10. *(AC: weight clamping)*
- `SaveTag_ClampsWeightBelowMin` — Weight=-11 → clamped to -10. *(AC: weight clamping)*
- `SaveTag_ParentCycle_ShowsInlineError` — `WouldCreateParentCycle` true → save blocked. *(AC: cycle rejection)*
- `AddGrant_Valid_AppearsInGrantsList` — grant added to observable collection. *(AC: grants CRUD)*
- `AddGrant_WouldCreateCycle_RejectsWithInlineError` — grant cycle detected; not added. *(AC: grant cycle rejection)*
- `DeleteTag_ShowsImpactCountsBeforeExecute` — `FakeTagsManager` returns non-zero impact. *(AC: delete impact preview)*

**`TagMergeViewModelTests.cs`**
- `Preview_PopulatesImpactCounts` — impact counts loaded from manager. *(AC: merge impact preview)*
- `Execute_CallsMergeTag_WithCorrectIds` — merge executed; source/target IDs correct. *(AC: merge executes)*

**`MovieEditorViewModelTests.cs`** (additions)
- `ShowPropertiesCommand_CallsMoviePropertiesService` — verify `IMoviePropertiesService.ShowAsync` called. *(AC: ShowPropertiesCommand wired)*
- `ActorList_IsPopulatedFromActorNames` — `Movie.ActorNames = "Alice,Bob"` → `ActorList = ["Alice","Bob"]`. *(AC: actor list wiring)*
- `MoveActorUpCommand_CanExecute_WhenSecondActorSelected` — `SelectedActorIndex=1` → `CanExecute=true`. *(AC: actor reorder enabled)*
- `MoveActorUpCommand_IsNoOp_WhenFirstActorSelected` — index 0; command does not throw. *(AC: actor reorder no-op)*
- `TaggerVm_IsInitialized_WhenMovieLoaded` — `TaggerVm` is non-null after `LoadAsync`. *(AC: Tagger in editor)*

**`LabelCleanerViewModelTests.cs`** (additions)
- `DetectTags_WhenTagsManagerInitialized_PopulatesDetectedTags` — `ITagsManager.IsInitialized=true`, tag name present in filename → `DetectedTags` populated. *(AC: DetectTags activated)*
- `DetectTags_WhenTagsManagerNotInitialized_IsEmpty` — `IsInitialized=false`; graceful no-op, `DetectedTags` empty. *(AC: DetectTags fallback)*
  > Note: `ITagsManager?` is nullable in `LabelCleanerViewModel`; existing `BuildSut` helper
  > construction (which omits the parameter) compiles without change. Use `FakeTagsManager`
  > in the two new test cases.

**`MoviesListViewModelTests.cs`** (additions)
- `DataChanged_SingleFiring_TriggersOneReload` — fire `ITagsManager.DataChanged` once; advance time past the debounce window; verify `FakeMovieCatalogRepository.GetMovieListAsync` is called exactly once. *(AC: grid refresh after tag change)*
- `DataChanged_RapidFirings_TriggersOnlyOneReload` — fire `ITagsManager.DataChanged` three times in rapid succession; verify `FakeMovieCatalogRepository.GetMovieListAsync` is called exactly once (after the debounce settles). *(AC: grid refresh after tag change)*

**`GrantsManagementViewModelTests.cs`** (new)
- `AddGrant_Valid_AppearsInList` — add a non-cycle-forming grant; row appears in the DataGrid collection. *(AC: Grants Management view)*
- `AddGrant_WouldCreateCycle_Rejects` — `WouldCreateGrantCycle` returns true; row is not added. *(AC: cycle rejection)*
- `RemoveGrant_RemovesFromList` — remove an existing grant row; collection count decreases. *(AC: Grants Management view)*

**`CategoryEditorViewModelTests.cs`** (new)
- `Delete_NonEmptyCategory_Blocked` — category with tags cannot be deleted; command is no-op or shows guard message. *(AC: category delete guard)*
- `Rename_DefaultCategory_Blocked` — Default Category rename is rejected. *(AC: category rename guard)*
- `Rename_MostUsedCategory_Blocked` — Most Used virtual category rename is rejected. *(AC: category rename guard)*

### Integration Tests (`VideoIndexer.Infrastructure.Tests/`)

**`DapperTagsRepositoryTests.cs`** (self-skipping without DB connection)
- `CreateCategory_And_GetAll_RoundTrip` — insert, read back. *(AC: category CRUD)*
- `DeleteCategory_WithTags_Throws` — non-empty category delete rejected. *(AC: category delete guard)*
- `CreateTag_And_GetById_RoundTrip` — insert, read back with weight clamping. *(AC: tag CRUD)*
- `DeleteTag_CascadesAssociationsAndReparentsChildren` — verify `tags_movies`, `tags_grants` cleaned, children reparented. *(AC: cascading delete)*
- `AddGrant_WouldCreateCycle_ThrowsBeforeInsert` — cycle guard fires before INSERT. *(AC: server-side cycle rejection)*
- `ConnectMovieTag_And_Disconnect_RoundTrip` — row inserted and removed. *(AC: movie-tag association)*
- `ConnectBookmarkTag_InsertsTags_bookmarksRow` — bookmark-tag DB row inserted. *(AC: bookmark-tag DB layer)*

**`DapperMovieCatalogRepositoryTests.cs`** (additions)
- `GetMovieListAsync_WithTags_EnrichesEffectiveWeight` — movie with stored tag (weight 3) granting another (weight 5); `TagWeight` (effective) = 8; `StoredTagWeight` = 3. *(AC: effective weight in list)*
- `GetMovieListAsync_TagsManagerNotInitialized_FallsBackToStoredWeight` — `IsInitialized = false`; `EffectiveTagWeight == StoredTagWeight`. *(AC: graceful fallback)*


## Documentation Updates

Per AGENTS.md maintenance rules:

- `docs/agents/project-manifest/api-surface.md`
  — Add `ITagsRepository`, `ITagsManager`, `Tag`, `TagCategory`, `TagGrant`,
    `TagDeleteImpact`, `TagMergeImpact`, `TagImportance`, `TagConstants`, `TaggingOptions`.
  — Update `MovieListItem` property list (renamed and new fields).
  — Update `Movie` property list (add `StoredTagIds` field).
  — Update DSL identifier table (semantic upgrades, new identifiers, guards removed).
  — Add `TaggerViewModel`, `TaggerCategoryViewModel`, `TaggerTagViewModel`,
    `TagEditorViewModel`, `TagGrantRowViewModel`, `CategoryEditorViewModel`,
    `GrantsManagementViewModel`, `TagMergeViewModel`.
  — Add `ITagEditorService`, `ITagMergeService`, `IGrantsManagementService`.
  — Update `AppOptions` (`Tagging` section).
  — Update `INameParser.Parse` signature.
  — Update `DatabaseBootstrapper.ExpectedRevision` constant.

- `docs/agents/project-manifest/file-tree.md`
  — Add all new Core, Infrastructure, and App files from Required Components.
  — Add new test files and `FakeTagsManager`.
  — Update `DatabaseBootstrapper.cs` annotation (revision 39→40, migration m040).

- `docs/agents/project-manifest/constraints.md`
  — Update `ExpectedRevision` to **40**.
  — Add m040 rollback procedure.
  — Add: "`TagsManager` is a singleton; `EnsureLoadedAsync` is idempotent and protected by `SemaphoreSlim(1,1)`."
  — Add: "Root-level tag uniqueness (MySQL NULL behaviour): application-level pre-insert check required in `DapperTagsRepository.CreateTagAsync` / `UpdateTagAsync`."
  — Update DSL deferred-identifier table: `HasTag` / `TagHasSubTags` are now active; update `TagsWeight` / `AmountTags` semantics note; add `StoredTagsWeight` / `AmountStoredTags`.
  — Add: "`INameParser.Parse` has a **fifth** `knownTagNames` parameter as of M7 (appended after the existing `knownStudios` parameter); callers that do not supply tags pass `[]`."

- `docs/agents/project-manifest/data-flows.md`
  — Update Shell Authentication Flow to include `ITagsManager.EnsureLoadedAsync()` call before `Transition(ShellState.Ready)`.
  — Add "Tagger tag-toggle flow" (user checks tag → `TaggerTagViewModel.ToggleCommand` → `connectTag` callback → `ITagsManager.ConnectMovieTagAsync` → `ITagsRepository` DB write → closure cache invalidated → `DataChanged` event → `MoviesListViewModel` grid reload).
  — Add note to `GetMovieListAsync` step: `EnsureLoadedAsync` + single JOIN batch tag query + effective-set enrichment.

- `docs/projects/rebuild/management-areas/filter-management-specification.md`
  — §3.4: update `TagsWeight` description to "Sum of effective tag weights (stored ∪ granted ∪ inherited)"; update `AmountTags` similarly.
  — §3.4: add `StoredTagsWeight` and `AmountStoredTags` rows.
  — §3.2: update `HasTag` and `TagHasSubTags` notes (remove "deferred until M7" language).

- `docs/projects/rebuild/milestones/m7-tagging-core.md` — create milestone document
  using the template from `roadmap.md`.

- `docs/agents/project-manifest/tech-stack.md`
  — Update the Schema revision row (line 63) from `Expected revision \`39\`` to
    `Expected revision \`40\``.


## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **Batch tag query mismatch** — data inconsistency between the JOIN batch query row count and the main query's `StoredTagCount` aggregate | Defensive guard in the Dapper row mapper: if the number of tag rows returned for a movie does not match the SQL `StoredTagCount`, log a warning and fall back to stored-only values. |
| **TagsManager init ordering** — `GetMovieListAsync` called before `EnsureLoadedAsync` completes | The `IsInitialized` fallback guard in `DapperMovieCatalogRepository` prevents any exception; enrichment is simply skipped and stored values are returned. |
| **Thread safety on concurrent `EnsureLoadedAsync` calls** | `SemaphoreSlim(1,1)` in `TagsManager` ensures initialisation runs exactly once regardless of concurrent callers. Document the threading contract in `constraints.md`. |
| **Effective-weight discrepancy between grid and editor** — grid shows old weight until reload after tag toggle | `ITagsManager.DataChanged` subscription in `MoviesListViewModel` triggers a reload. Transient delay is acceptable; no data integrity issue. |
| **Cycle detection false negatives under concurrent writes** | Tag graph mutations are low-frequency admin operations; concurrent writes are highly unlikely. Document in `constraints.md` that `WouldCreateCycle` is not transaction-safe; a DB constraint cannot enforce acyclicity. |
| **`m040` migration fails on DBs with existing duplicate data** | The legacy app enforces uniqueness at the UI layer so duplicates are highly unlikely. If the migration fails, `DatabaseBootstrapStatus.RevisionTooOld` surfaces a clear error message. Document rollback procedure in `constraints.md`. |
| **`MovieListItem` rename breaks DSL-dependent tests** | `grep` all test files for `TagWeight` / `TagCount` before starting; fix all references in the same work package as the rename step. |
| **`INameParser.Parse` signature change breaks all existing callers** | Update all callers (`NameParser.cs`, `LabelCleanerViewModel.cs`, `FakeNameParser.cs`, all test call sites) atomically with the interface change in a single work package. |
| **Tagger callback pattern adaptation for M10** | The `Func<long, CancellationToken, Task>` callback shape is already sufficient for bookmark-level tagging; M10 only needs to pass different closures. No M7 refactoring risk. |
| **Avalonia flat-list indentation edge cases** — deeply nested or orphaned tags | Start with flat `ListBox` + `Margin.Left = Depth * 16`; validate against real tag data early in the UI work package before committing to a full `TreeView`. |
| **`TaggerViewModel` reload race when `DataChanged` fires while Tagger is open** | `MoviesListViewModel` reload does not affect the open `TaggerViewModel`; the tagger maintains its own loaded state. A "tags updated" snackbar can be added as a polish item in a later milestone. |
