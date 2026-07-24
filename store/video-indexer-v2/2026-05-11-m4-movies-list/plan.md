# Plan: M4 — Movies List

## Summary

M4 introduces the primary browsing surface of the application: the main movies list grid. Building on the M3 library/indexing shell, M4 replaces the empty Default page in `MainContentViewModel` with a fully populated `DataGrid` backed by a new `IMovieCatalogRepository.GetMovieListAsync` query. The grid delivers all twelve specification-required columns (Label, Actor Names, Studio, Rating, Weight, Thumbnails, Tags, Modified, Cover, Bookmarks, Views, Review), multi-select, per-column sorting, column visibility persistence, a toggleable cover preview panel (placeholder only — M9 populates it), and a right-click context menu. Filter bar integration is scaffolded but left empty for M5. All context menu actions are present but non-functional stubs.


## Architectural Context

### Existing Ready-state shell

`MainContentViewModel` (`src/VideoIndexer.App/ViewModels/MainContentViewModel.cs`) is the root VM when the shell reaches `ShellState.Ready`. It currently composes `LibraryFoldersViewModel` and `RefreshIndexViewModel`, exposes an `IndexedMovieCount`, and swaps between three `MainContentPage` values:

- `Default` — `CurrentChildViewModel = null`, renders empty content
- `LibraryFolders` — shows folder management panel
- `RefreshOverlay` — shows refresh progress panel

The `MainContentView.axaml` (`src/VideoIndexer.App/Views/MainContentView.axaml`) renders a toolbar row and a `ContentControl` bound to `CurrentChildViewModel`. The `Default` page currently produces no visible content.

### Data layer

`IMovieCatalogRepository` (`src/VideoIndexer.Core/Abstractions/IMovieCatalogRepository.cs`) currently exposes five methods: `GetCountAsync`, `MovieExistsAsync`, `InsertMovieAsync`, `FilenameExistsAsync`, `AddFilenameAsync`. None returns a rich movie model suitable for the list view. The concrete implementation is `DapperMovieCatalogRepository` (`src/VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs`).

### Database schema (verified against structure.sql, revision 33; live is revision 35)

Relevant tables for M4:

| Table | Key columns used by the list query |
|---|---|
| `movies` | `movie_id`, `hash`, `label`, `name` (actors), `studio`, `rating`, `watch_count`, `review` |
| `tags_movies` | `movie_id`, `tag_id` (explicit tag assignments — no grants) |
| `tags` | `tag_id`, `weight` |
| `movies_bookmarks` | `movie_id` (presence check) |

**Schema gap — Modified column:** The `movies` table (revision 33) has no file-creation-date column. The spec states the catalog stores `File.GetCreationTime` for the Modified column. Revisions 34–35 (delta not available for inspection) may have added a column. This must be verified in the first work package.

### Settings system

`AppOptions` (`src/VideoIndexer.Core/Options/AppOptions.cs`) is a sealed record with six top-level properties bound from `appsettings.json`. New settings sub-sections require a new `sealed record` in `VideoIndexer.Core/Options/`, a new property on `AppOptions`, and a default entry in the bundled `Assets/appsettings.json`.

### Packages

`Avalonia.Controls.DataGrid` is not present in `Directory.Packages.props`. It must be added before any DataGrid AXAML is compiled.

### Test fakes

Two fakes implement `IMovieCatalogRepository` and must be updated for every interface change:

- `tests/VideoIndexer.Tests/Fixtures/InMemoryMovieCatalogRepository.cs`
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeMovieCatalogRepository.cs`


## Approach / Architecture

M4 is structured as a vertical slice across all three layers:

1. **Core** — New `MovieListItem` model (all grid columns), new `MovieListQuery` record (empty for M4; M5 adds filter fields), new `MoviesListOptions` settings record; `IMovieCatalogRepository` extended with `GetMovieListAsync`.
2. **Infrastructure** — `DapperMovieCatalogRepository.GetMovieListAsync` runs a single SQL query with LEFT JOINs to aggregate tag weight, tag count, and bookmark presence per movie.
3. **App** — New `MoviesListViewModel` exposed as the `Default` page content inside `MainContentViewModel`; new `MoviesListView.axaml` with `DataGrid`, cover panel, and toolbar additions; DI registration in `Program.cs`.

The `Default` page of `MainContentViewModel` is redefined: instead of null/empty, it always points to the `MoviesListViewModel`. The Library Folders and Refresh overlays continue to use the existing `ContentControl` overlay mechanism — they will occlude the movies list when active.

A `MovieListQuery` record is introduced now, even though it carries no fields in M4. This future-proofs `IMovieCatalogRepository.GetMovieListAsync` so M5 can add filter/search fields to the query without changing the interface signature.


## Rationale

- **Native Avalonia DataGrid** is the idiomatic Avalonia component for tabular data. It supports virtual scrolling, column sorting, multi-select, and column header customization without additional libraries.
- **Single SQL query with LEFT JOINs** for aggregated columns (tag weight, tag count, bookmark flag) avoids N+1 queries per row. The join is efficient on indexed `movie_id` foreign keys.
- **`MovieListQuery` introduced early** so M5's filter integration is a data-layer addition only — no interface signature change required at that point.
- **`MoviesListOptions` in Core** keeps column visibility and panel state portable and persisted across sessions via the existing `ISettingsService` mechanism.
- **Cover image deferred to M9** — `HasCoverImage` requires knowledge of the data-folder path convention for per-movie assets; introducing that filesystem I/O in M4 would couple the list query to the file system. M4 returns `false` for this field; M9 populates it. `ThumbnailCount` is NOT deferred — it is aggregated from `movies_thumbnails` via LEFT JOIN in the same query, reading only database state.


## Detailed Steps

1. **Verify live `movies` schema (Modified column)** — run `DESCRIBE movies;` against the configured MariaDB instance (revision 35). Look for a `file_created`, `date_added`, or equivalent datetime column. Depending on result, choose one of two paths:
   - **Column exists:** add it to `MovieListItem.Modified` and include it in the `GetMovieListAsync` query.
   - **Column absent:** add `file_created DATETIME NULL` to the `movies` table via a migration SQL script committed at `src/VideoIndexer.Infrastructure/Database/migrations/m036_add_file_created.sql`, update `DatabaseBootstrapper.ExpectedRevision` to `36`, and add the column to the `InsertMovieAsync` INSERT statement (passing `NULL` default so no scanner changes are required for M4). Document the migration in `constraints.md`.

     > **Note (audit):** The `migrations/` subdirectory does not currently exist under `src/VideoIndexer.Infrastructure/Database/`. It must be created as part of this step. Include the new directory in the `file-tree.md` manifest update (Step 21).

2. **Add `Avalonia.Controls.DataGrid 11.3.13`** to `Directory.Packages.props` (new `ItemGroup` under Avalonia section). Add a version-free `<PackageReference Include="Avalonia.Controls.DataGrid" />` to `src/VideoIndexer.App/VideoIndexer.App.csproj`. Also add `<StyleInclude Source="avares://Avalonia.Controls.DataGrid/Themes/Fluent.xaml"/>` inside `<Application.Styles>` in `src/VideoIndexer.App/App.axaml` — without this the DataGrid renders without column headers, borders, or row styling despite compiling correctly.

   > **Note (audit):** Version `11.3.14` does not exist on NuGet.org — the DataGrid package is independently versioned and the 11.x series ends at `11.3.13`. Use `11.3.13`, which declares a dependency on `Avalonia >= 11.3.13` and is fully compatible with the project's `11.3.14` Avalonia suite. Do not use `12.0.0` — it targets Avalonia 12.x and is incompatible with this project's pinned Avalonia `11.3.14`.

3. **Create `MoviesListOptions`** at `src/VideoIndexer.Core/Options/MoviesListOptions.cs` — a `sealed record` with:
   - `bool ShowCoverPanel { get; init; } = true`
   - `IReadOnlyDictionary<string, bool> ColumnVisibility { get; init; }` — default: all twelve columns visible. Keys must use the following 12 string constants, defined as a nested `public static class ColumnKeys` inside `MoviesListOptions.cs` to prevent typos in AXAML bindings and `appsettings.json`:
     `"Label"`, `"ActorNames"`, `"Studio"`, `"Rating"`, `"TagWeight"`, `"TagCount"`, `"ThumbnailCount"`, `"HasCoverImage"`, `"HasBookmarks"`, `"ViewCount"`, `"NeedsReview"`, `"Modified"`

     The property initializer must enumerate all 12 keys explicitly:
     ```csharp
     public IReadOnlyDictionary<string, bool> ColumnVisibility { get; init; } =
         new Dictionary<string, bool>
         {
             [ColumnKeys.Label]          = true,
             [ColumnKeys.ActorNames]     = true,
             [ColumnKeys.Studio]         = true,
             [ColumnKeys.Rating]         = true,
             [ColumnKeys.TagWeight]      = true,
             [ColumnKeys.TagCount]       = true,
             [ColumnKeys.ThumbnailCount] = true,
             [ColumnKeys.HasCoverImage]  = true,
             [ColumnKeys.HasBookmarks]   = true,
             [ColumnKeys.ViewCount]      = true,
             [ColumnKeys.NeedsReview]    = true,
             [ColumnKeys.Modified]       = true,
         };
     ```

   _Sort column and direction fields are deferred to M5 (see Out of Scope). The DataGrid always opens sorted by Label ascending in M4._

4. **Add `MoviesList` property to `AppOptions`** — `public MoviesListOptions MoviesList { get; init; } = new();`

5. **Update bundled `Assets/appsettings.json`** — add `"MoviesList"` section with all twelve columns set to `true`, `ShowCoverPanel: true`.

6. **Create `MovieListItem`** at `src/VideoIndexer.Core/Models/MovieListItem.cs` — a `sealed record` with:
   - `required long MovieId { get; init; }`
   - `required string Hash { get; init; }`
   - `required string Label { get; init; }`
   - `required string ActorNames { get; init; }`
   - `required string Studio { get; init; }`
   - `required int Rating { get; init; }`
   - `required int TagWeight { get; init; }` — sum of `tags.weight` for assigned tags
   - `required int TagCount { get; init; }` — count of rows in `tags_movies`
   - `required int ThumbnailCount { get; init; }` — count of rows in `movies_thumbnails` (aggregated via LEFT JOIN; no filesystem access required)
   - `required bool HasCoverImage { get; init; }` — `false` until M9
   - `required bool HasBookmarks { get; init; }`
   - `required int ViewCount { get; init; }` — maps to `movies.watch_count`
   - `required bool NeedsReview { get; init; }` — maps to `movies.review = 'yes'`
   - `required DateTime? Modified { get; init; }` — null if schema gap remains unresolved

7. **Create `MovieListQuery`** at `src/VideoIndexer.Core/Models/MovieListQuery.cs` — a `sealed record` with no properties for M4. It is the future extension point for filter predicates (M5).

8. **Extend `IMovieCatalogRepository`** — add:
   ```csharp
   Task<IReadOnlyList<MovieListItem>> GetMovieListAsync(
       MovieListQuery query,
       CancellationToken cancellationToken = default);
   ```

9. **Implement `DapperMovieCatalogRepository.GetMovieListAsync`** — single SQL query:
   ```sql
   SELECT
       m.movie_id                        AS MovieId,
       m.hash,
       m.label,
       m.name                            AS ActorNames,
       m.studio,
       m.rating,
       m.watch_count                     AS ViewCount,  -- watch_count = embedded-player plays; open_count = file-manager opens (excluded per spec)
       CASE WHEN m.review = 'yes' THEN 1 ELSE 0 END AS NeedsReview,
       COALESCE(SUM(t.weight), 0)        AS TagWeight,
       COUNT(DISTINCT tm.tag_id)         AS TagCount,
       COUNT(DISTINCT mt.thumbnail_id)   AS ThumbnailCount,
       0                                 AS HasCoverImage,
       CASE WHEN bk.movie_id IS NOT NULL THEN 1 ELSE 0 END AS HasBookmarks,
       m.file_created                    AS Modified    -- or NULL if column absent
   FROM movies m
   LEFT JOIN tags_movies tm       ON tm.movie_id = m.movie_id
   LEFT JOIN tags t               ON t.tag_id    = tm.tag_id
   LEFT JOIN movies_thumbnails mt ON mt.movie_id = m.movie_id
   LEFT JOIN (
       SELECT DISTINCT movie_id FROM movies_bookmarks
   ) bk ON bk.movie_id = m.movie_id
   GROUP BY m.movie_id
   ORDER BY m.label
   ```
   Map to `MovieListItem` via Dapper's `QueryAsync<T>`. Use `.ConfigureAwait(false)` on all awaits.

10. **Update `InMemoryMovieCatalogRepository`** (`tests/VideoIndexer.Tests/Fixtures/`) — add a stub `GetMovieListAsync` that returns an empty list.

11. **Update `FakeMovieCatalogRepository`** (`tests/VideoIndexer.App.Tests/TestHelpers/`) — add the following members to support the Step 19 unit tests:
    - `SetMovies(IReadOnlyList<MovieListItem> movies)` — stores the list returned by `GetMovieListAsync`
    - `SetException(Exception? ex)` — when non-null, `GetMovieListAsync` throws that exception instead of returning; enables `LoadAsync_SetsIsLoadingFalse_InFinallyBlock`
    - `public int GetMovieListCallCount { get; private set; }` — incremented on each `GetMovieListAsync` invocation; enables `LoadAsync_ConcurrentCall_SecondCallIsNoOp` to assert the fake was called exactly once
    - A stub `GetMovieListAsync` implementation that increments `GetMovieListCallCount`, throws if `SetException` was called with a non-null value, and otherwise returns the configured list

12. **Create `MoviesListViewModel`** at `src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs`:
    - Extends `ObservableObject` (no `IDisposable` — M4 introduces no event subscriptions or unmanaged resources that require cleanup; add `IDisposable` when a `SettingsChanged` or similar subscription is introduced in a later milestone)
    - Constructor injects `IMovieCatalogRepository`, `ISettingsService`, `ILogger<MoviesListViewModel>`
    - `ObservableCollection<MovieListItem> Movies` (observable backing field)
    - `MovieListItem? SelectedMovie` (`[ObservableProperty]`)
    - `IList<MovieListItem> SelectedMovies` (multi-select backing field; populated via a `SelectionChanged` event handler in `MoviesListView.axaml.cs` — Avalonia 11 DataGrid does not support two-way collection binding for `SelectedItems`)
    - `bool ShowCoverPanel` (`[ObservableProperty]`)
    - `IReadOnlyDictionary<string, bool> ColumnVisibility` (`[ObservableProperty]`)
    - `bool IsLoading` (`[ObservableProperty]`) — `true` while `GetMovieListAsync` is in-flight; drives the loading indicator in the view.
    - `bool IsEmpty` (`[ObservableProperty]`) — `true` when `!IsLoading && Movies.Count == 0`; drives the empty-state message in the view. Set at the end of `LoadAsync` after populating `Movies`.
    - `Task LoadAsync(CancellationToken ct)` — uses a **separate** `private int _loadingGuard = 0` field (not `[ObservableProperty]`) as an atomic single-flight gate via `Interlocked.CompareExchange`. The guard is distinct from the `_isLoading` backing field that drives the UI. Pattern:
      ```csharp
      if (Interlocked.CompareExchange(ref _loadingGuard, 1, 0) != 0) return;
      try { IsLoading = true; Movies.Clear(); var items = await _catalogRepository.GetMovieListAsync(...); foreach (var item in items) Movies.Add(item); }
      finally { IsLoading = false; IsEmpty = Movies.Count == 0; Interlocked.Exchange(ref _loadingGuard, 0); }
      ```
      Always begin the try block with `Movies.Clear()` before issuing the database query, then repopulate via `foreach (var item in result) Movies.Add(item)`. Never replace the `Movies` collection reference — `Movies` is not `[ObservableProperty]`, so assigning a new collection silently breaks the DataGrid binding with no compiler error or runtime exception. This matches the `Interlocked.CompareExchange` single-flight pattern already used by `RefreshOrchestrator` in this codebase and makes the `LoadAsync_ConcurrentCall_SecondCallIsNoOp` unit test deterministic.
    - `[RelayCommand] ToggleCoverPanel()` — produces new `MoviesListOptions` via `with {}`, calls `ISettingsService.SaveAsync`
    - `[RelayCommand] SetColumnVisibility(string columnName)` — toggles the named column, persists settings
    - Context menu stubs (all `[RelayCommand]`, all no-ops for M4): `PlayMovie`, `PlayMovieFullscreen`, `EditMovie`, `GenerateThumbnails`, `CopyMovieTo`, `DeleteMovieOnDisk`
    - Parameterless constructor for tests (all services null; no-op behaviour)

13. **Create `MoviesListView.axaml`** at `src/VideoIndexer.App/Views/MoviesListView.axaml`:
    - Root `Grid` with two columns: `*` (list area) and `Auto` (cover panel)
    - Column 0 (list area) is a nested `Grid` with two rows (`Auto` toolbar + `*` content):
      - **Local toolbar row** (`Auto`): `StackPanel` containing a `CheckBox` bound to `ShowCoverPanel` / `ToggleCoverPanelCommand` — co-located here rather than in `MainContentView.axaml` to avoid a cross-VM binding from `MainContentViewModel` into `MoviesListViewModel`
      - **Content area row** (`*`): a `Panel` (overlapping layers) containing three elements in z-order:
        1. An indeterminate `ProgressBar` docked at the top, `IsVisible="{Binding IsLoading}"` — shown while the DB query is in-flight
        2. A centred `TextBlock` with text `"No movies indexed yet — add folders in Library Folders and run Refresh Index"`, `IsVisible="{Binding IsEmpty}"` — shown when the query completes with zero results
        3. The `DataGrid` bound to `Movies` with `x:DataType="vm:MoviesListViewModel"`:
           - Twelve `DataGridTextColumn` / `DataGridCheckBoxColumn` entries. Because `AvaloniaUseCompiledBindingsByDefault=true`, column `Binding` expressions compile against the `x:DataType` of the enclosing element — i.e., `MoviesListViewModel`, which does not expose row properties. Each column binding **must carry its own `x:DataType`** pointing at the item type to resolve correctly:
             ```xml
             <DataGridTextColumn Header="Label"
                 Binding="{Binding Label, x:DataType=models:MovieListItem}" />
             <DataGridTextColumn Header="Actors"
                 Binding="{Binding ActorNames, x:DataType=models:MovieListItem}" />
             <!-- … repeat for all 12 columns … -->
             <DataGridCheckBoxColumn Header="Review"
                 Binding="{Binding NeedsReview, x:DataType=models:MovieListItem}" />
             ```
             Add `xmlns:models="using:VideoIndexer.Core.Models"` to the file's root element namespace declarations.
           - `CanUserSortColumns="True"`, `SelectionMode="Extended"` (multi-select)
           - `SelectionChanged` event attribute wired to a code-behind handler (see Step 14) that pushes the updated selection into `MoviesListViewModel.SelectedMovies`
           - Column header right-click context menu for visibility toggle (one `MenuItem` per column bound to `SetColumnVisibilityCommand`). Use `x:Static` to pass the `ColumnKeys` constant as `CommandParameter` rather than hardcoding a string literal:
             ```xml
             <MenuItem Header="Label"
                 Command="{Binding SetColumnVisibilityCommand}"
                 CommandParameter="{x:Static coreOpts:MoviesListOptions+ColumnKeys.Label}" />
             ```
             `MoviesListOptions` is defined in `VideoIndexer.Core.Options`, **not** in `VideoIndexer.App.ViewModels`. The existing `xmlns:vm` alias cannot resolve it. Add a separate namespace declaration to the root element: `xmlns:coreOpts="using:VideoIndexer.Core.Options"`. Use `coreOpts:MoviesListOptions+ColumnKeys.Label` (and the same prefix for all twelve column-key constants). The `vm:` alias continues to serve `x:DataType` and other ViewModel bindings unchanged. Using the wrong prefix causes an Avalonia XAML compile error, which is treated as a build failure under `TreatWarningsAsErrors=true`.
    - Cover panel: `Border` in column 1, `IsVisible="{Binding ShowCoverPanel}"`, contains a placeholder `TextBlock` ("No cover — M9") shown **unconditionally** regardless of selection state — the same placeholder appears whether zero or one row is selected; M9 will replace it with a real image

14. **Create `MoviesListView.axaml.cs`** — standard code-behind; no constructor injection (follows existing view conventions). Must include two event handlers wired in the parameterless constructor:
    - **`Loaded` handler** — when `DataContext` is `MoviesListViewModel`, calls `_ = vm.LoadAsync(CancellationToken.None)` as a bare fire-and-forget (matching the `DatabaseConnectorView.axaml.cs` pattern: `_ = vm.LoadAsync()`). Do **not** add a `.ContinueWith` exception logger here — the view code-behind has no `ILogger` field (parameterless constructor constraint) and the existing convention does not use one. Exception logging for `LoadAsync` failures must be handled **inside `MoviesListViewModel.LoadAsync`** using the injected `ILogger<MoviesListViewModel>`. This is the sole trigger for the initial DataGrid population on first launch; without it the grid is permanently empty.
    - **`SelectionChanged` handler** (on the DataGrid) — reads `e.AddedItems` / `e.RemovedItems`, rebuilds the selection list, and assigns it to `vm.SelectedMovies`, enabling multi-select context menu actions.

15. **Update `MainContentViewModel`**:
    - Add `MoviesListViewModel MoviesListVm { get; }` property (named `MoviesListVm` to distinguish it clearly from a View type)
    - In full DI constructor: accept `MoviesListViewModel moviesListVm`, assign to `MoviesListVm`. Also in the DI constructor body (immediately after `MoviesListVm` is assigned): set `CurrentChildViewModel = MoviesListVm` so the Default page shows the movies grid as soon as `MainContentViewModel` is constructed (which happens on Shell transition to `Ready`). Do _not_ place this assignment inside `ActivateAsync` — `ActivateAsync` is not called from production code and would leave the grid permanently blank.
    - In the **existing** `OnRefreshStateChanged` handler: inside the `RefreshState.Completed or RefreshState.Faulted` branch (not `Idle` — `Idle` signals a cancelled run per `RefreshOrchestrator` source), add a call to `_ = MoviesListVm.LoadAsync(CancellationToken.None)` (fire-and-forget with exception logging) alongside the existing `RefreshCountAsync` call. Do not add a second `StateChanged` subscription — one already exists.
    - Update `CloseSubView()` — replace `CurrentChildViewModel = null` with `CurrentChildViewModel = MoviesListVm` so returning from the Library Folders panel shows the grid
    - Update the `else` branch of `OnRefreshStateChanged` — replace `CurrentChildViewModel = null` with `CurrentChildViewModel = MoviesListVm` so the grid reappears after a Refresh overlay closes

16. **Update `MainContentView.axaml`** — add a disabled `TextBox` with `Watermark="Search (M5)"` to the toolbar row as a placeholder for the M5 filter bar. _Do not_ add the cover panel toggle here — it lives inside `MoviesListView.axaml` (Step 13) to keep its binding co-located with the VM that owns `ShowCoverPanel`. The `ContentControl` body (Row 1) remains as-is; `MoviesListView` is the resolved view for the Default page.

17. **Register `MoviesListViewModel` in DI** (`src/VideoIndexer.App/Program.cs`) — **Transient** registration (`builder.Services.AddTransient<MoviesListViewModel>()`), matching the existing `LibraryFoldersViewModel`, `RefreshIndexViewModel`, and `MainContentViewModel` registrations. Selection state and scroll position reset on each Ready-state re-entry; this is acceptable for M4. Registering as Singleton while `MainContentViewModel` is Transient would silently couple multiple `MainContentViewModel` instances to the same `MoviesListViewModel` on disconnect/reconnect cycles.

18. **Register `MoviesListView` in DI** (`src/VideoIndexer.App/Program.cs`) — add `builder.Services.AddTransient<MoviesListView>(_ => new MoviesListView());` in the view-registrations block after the existing `RefreshIndexView` entry. This is required even though `ViewLocator.cs` uses a convention-based name lookup: `ViewLocator.Build()` always resolves the view through `App.Services.GetRequiredService(viewType)`, so the type must be registered. Every other content view in the codebase has an equivalent explicit registration.

19. **Write unit tests** for `MoviesListViewModel` (`tests/VideoIndexer.App.Tests/`):
    - `LoadAsync_WithMovies_PopulatesCollection`
    - `LoadAsync_WithMovies_SetsIsEmptyFalse`
    - `LoadAsync_EmptyCatalog_LeavesCollectionEmpty`
    - `LoadAsync_EmptyCatalog_SetsIsEmptyTrue`
    - `LoadAsync_SetsIsLoadingFalse_InFinallyBlock` (fake repo throws; verify `IsLoading` is `false` after exception)
    - `LoadAsync_ConcurrentCall_SecondCallIsNoOp` (verify `_loadingGuard` prevents overlapping loads — configure `FakeMovieCatalogRepository.GetMovieListAsync` with a `TaskCompletionSource` gate that pauses until explicitly released; fire-and-forget the first `LoadAsync` call, then immediately call `LoadAsync` again synchronously; release the gate; assert `FakeMovieCatalogRepository.GetMovieListCallCount == 1`)
    - `ToggleCoverPanel_PersistsSettingsViaSettingsService`
    - `SetColumnVisibility_TogglesColumn_PersistsSettings`
    - `MoviesListOptions_Serialization_RoundTrips` — serialize a `MoviesListOptions` instance with at least two columns toggled to `false` via `System.Text.Json.JsonSerializer.Serialize`, then deserialize back, and assert all twelve column-visibility values match the original. This guards against the `IReadOnlyDictionary<string, bool>` property failing to survive a JSON round-trip, which would silently reset column-visibility preferences on every app restart.

20. **Write integration tests** for `GetMovieListAsync` (`tests/VideoIndexer.Infrastructure.Tests/Library/DapperMovieCatalogRepositoryTests.cs`):
    - `GetMovieListAsync_ReturnsAllInsertedMovies`
    - `GetMovieListAsync_AggregatesTagWeightCorrectly` (insert movie + tag + tags_movies row, verify TagWeight)
    - `GetMovieListAsync_ReviewYes_SetsNeedsReviewTrue` (insert movie with `review = 'yes'`; assert `NeedsReview == true` — validates the `CASE WHEN` mapping and catches any alias regression silently)
    - `GetMovieListAsync_EmptyDatabase_ReturnsEmptyList`
    - All tests self-skip when no live DB is configured.

21. **Update manifest documents**:
    - `docs/agents/project-manifest/api-surface.md` — add `MovieListItem`, `MovieListQuery`, `MoviesListOptions`, updated `IMovieCatalogRepository`, `MoviesListViewModel` signatures
    - `docs/agents/project-manifest/file-tree.md` — add new files
    - `docs/agents/project-manifest/tech-stack.md` — add `Avalonia.Controls.DataGrid 11.3.13`
    - `docs/agents/project-manifest/constraints.md` — update `ExpectedRevision` if schema migration was required; add `MoviesListOptions` mutation rule
    - `docs/projects/rebuild/milestones/m4-movies-list.md` — update Status to `Complete`


## Dependencies

- M3 complete (library folders, hashing, refresh worker) ✅
- `movies`, `tags_movies`, `tags`, `movies_bookmarks` tables present in target database
- Schema revision check: if Modified column is absent, a `db_revision` bump and migration script are required


## Required Components

**New files (Core):**
- `src/VideoIndexer.Core/Options/MoviesListOptions.cs`
- `src/VideoIndexer.Core/Models/MovieListItem.cs`
- `src/VideoIndexer.Core/Models/MovieListQuery.cs`

**New files (App):**
- `src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs`
- `src/VideoIndexer.App/Views/MoviesListView.axaml`
- `src/VideoIndexer.App/Views/MoviesListView.axaml.cs`

**New files (Infrastructure, conditional):**
- `src/VideoIndexer.Infrastructure/Database/migrations/m036_add_file_created.sql` — only if the Modified column is absent from the live schema

**Modified files:**
- `Directory.Packages.props` — add `Avalonia.Controls.DataGrid 11.3.13`
- `src/VideoIndexer.App/VideoIndexer.App.csproj` — add `PackageReference`
- `src/VideoIndexer.Core/Options/AppOptions.cs` — add `MoviesList` property
- `src/VideoIndexer.App/Assets/appsettings.json` — add `MoviesList` defaults
- `src/VideoIndexer.Core/Abstractions/IMovieCatalogRepository.cs` — add `GetMovieListAsync`
- `src/VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs` — implement `GetMovieListAsync`; conditionally extend `InsertMovieAsync` for the new column
- `src/VideoIndexer.App/ViewModels/MainContentViewModel.cs` — wire `MoviesListViewModel`, reload on refresh
- `src/VideoIndexer.App/App.axaml` — add DataGrid Fluent theme style include
- `src/VideoIndexer.App/Views/MainContentView.axaml` — toolbar additions
- `src/VideoIndexer.App/Program.cs` — DI registration
- `tests/VideoIndexer.Tests/Fixtures/InMemoryMovieCatalogRepository.cs` — stub `GetMovieListAsync`
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeMovieCatalogRepository.cs` — stub + configurable list
- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperMovieCatalogRepositoryTests.cs` — new test cases
- `docs/agents/project-manifest/api-surface.md`
- `docs/agents/project-manifest/file-tree.md`
- `docs/agents/project-manifest/tech-stack.md`
- `docs/agents/project-manifest/constraints.md` (conditional)
- `docs/projects/rebuild/milestones/m4-movies-list.md`

**Conditional (schema migration path only):**
- `src/VideoIndexer.Infrastructure/Database/DatabaseBootstrapper.cs` — `ExpectedRevision = 36`


## Assumptions

- The live `movies` table schema (revision 35) either already has a `file_created` column, or M4 adds it via a migration. The plan handles both paths (Step 1 + conditional Step 9 / Step 15).
- Column drag-to-reorder and persistence of column order are deferred. M4 delivers column visibility persistence only. Sort-state persistence (which column, which direction) is also deferred to M5; `MoviesListOptions` contains no sort fields for M4.
- Pagination is deferred. M4 loads all movies in a single query; the Avalonia DataGrid provides virtual scrolling natively. For the expected dataset size (production: hundreds to low-thousands of rows), this is acceptable.
- `HasCoverImage` returns `false` for all rows in M4; filesystem-dependent cover-image detection is deferred to M9 when the data-folder path convention is established. `ThumbnailCount` is NOT deferred — it is aggregated from `movies_thumbnails` in the same SQL query.
- All context menu items are present but are no-ops (log a warning, do nothing). No destructive or side-effecting operations are implemented.
- `MoviesListView` follows the standard parameterless-constructor convention. `MoviesListViewModel` is injected via DI through the factory registered in `Program.cs`.
- M4 resolves the M3 deferred debt: `MainContentViewModel.ActivateAsync` (currently defined but never called from production code) continues to serve its existing role of loading `IndexedMovieCount`. The initial `CurrentChildViewModel = MoviesListVm` assignment is placed directly in the DI constructor body (Step 15) so no external activation trigger is required — the assignment fires automatically when the Shell transitions to the `Ready` state.


## Constraints

- **Compiled bindings required** — `MoviesListView.axaml` must declare `x:DataType="vm:MoviesListViewModel"` on the root element (or on the DataGrid). All `{Binding …}` expressions inside must resolve at compile time.
- **ViewLocator convention** — `MoviesListViewModel` must resolve to `MoviesListView` via the existing naming convention. Verify that `ViewLocator.cs` picks it up before running the app.
- **Core has no external NuGet dependencies** — `MovieListItem`, `MovieListQuery`, `MoviesListOptions` must not reference any third-party package.
- **`ConfigureAwait(false)` on all Infrastructure awaits** — every `await` in `DapperMovieCatalogRepository.GetMovieListAsync` must use `.ConfigureAwait(false)`.
- **`AppOptions` is immutable** — all `MoviesListOptions` mutations (cover panel, column toggle) must produce a new record via `with { }` and be committed through `ISettingsService.SaveAsync`.
- **No `Version=` in `.csproj`** — `Avalonia.Controls.DataGrid` version must be declared in `Directory.Packages.props` only.
- **`TreatWarningsAsErrors=true`** — the DataGrid package may emit Avalonia XAML-compiler warnings for unresolved binding paths. All bindings must be verifiably correct before the PR is merged.
- **Single-flight refresh guard** — `MoviesListViewModel.LoadAsync` may be called concurrently (e.g., rapid double-refresh). Add a `private int _loadingGuard = 0` field and use `Interlocked.CompareExchange` as the single-flight gate (see Step 12). Do **not** use the `IsLoading` observable property as the guard — it is not atomic and is reserved for driving the UI loading indicator.
- **`MainContentPage.Default` no longer means empty** — after M4, the `Default` page maps `CurrentChildViewModel` to `MoviesListVm`, not `null`. Both `CloseSubView()` and the `OnRefreshStateChanged` else-branch must assign `CurrentChildViewModel = MoviesListVm` when returning to `Default`. Any future code or documentation that equates `Default` with empty/null content is stale.


## Out of Scope

- Filter bar and filter expressions — M5
- Filter slots and saved filters — M5
- Movie editor — M6
- Cover image display (real image) — M9
- Thumbnail count (real count) — M9 (note: `ThumbnailCount` in the list view IS implemented in M4 via `movies_thumbnails` JOIN; this Out of Scope entry refers to the thumbnail *generator* workflow and mosaic counts only)
- Thumbnail generator — M9
- Video player / VLC launch — M10
- Bookmark browser — M10
- Delete-on-disk implementation — deferred (M6 or later)
- Column drag-to-reorder persistence
- Pagination / virtual loading from DB
- Label Cleaner, Properties dialog, Review dialog — M6
- Column sort-state persistence (persisting active sort column and direction across restarts) — M5


## Acceptance Criteria

1. After the M3 `Connecting → LoggingOn → Ready → Refresh Index` flow, the main content area shows a populated DataGrid with all twelve columns.
2. Clicking any column header sorts the list by that column ascending; clicking again reverses to descending.
3. Multi-select (Ctrl+Click, Shift+Click, Ctrl+A) selects multiple rows.
4. The cover preview panel, when visible, always displays a placeholder ("No cover — M9") regardless of selection state — the same placeholder is shown whether zero or one row is selected.
5. The cover panel is shown/hidden via a toolbar toggle; toggle state survives an app restart.
6. Column visibility can be toggled from a right-click header context menu; state survives an app restart.
7. Right-clicking a row shows a context menu with all six actions (Play, Play Fullscreen, Edit, Generate Thumbnails, Copy To…, Delete on Disk); clicking any action produces no exception.
8. After a library refresh completes, the grid reloads automatically to reflect newly indexed movies.
9. `dotnet build -c Release` produces zero warnings and zero errors across all four projects.
10. `dotnet test` is fully green (all unit tests pass; integration tests self-skip if no DB configured).
11. While the initial movie list is loading (or reloading after a refresh completes), a visual loading indicator is visible and the grid area does not show stale or empty content without explanation.
12. When the library has zero indexed movies, the content area shows the "No movies indexed yet" message rather than a blank grid.


## Testing Strategy

- **Unit tests** (`VideoIndexer.App.Tests`) — `MoviesListViewModelTests`: `LoadAsync` populates the collection and clears `IsEmpty`; empty catalog sets `IsEmpty = true`; `IsLoading` is always `false` after `LoadAsync` completes (including the exception path); the `_loadingGuard` prevents concurrent loads (asserted via `FakeMovieCatalogRepository.GetMovieListCallCount`); `ToggleCoverPanel` invokes `ISettingsService.SaveAsync` with the toggled value; `SetColumnVisibility` produces the correct updated dictionary; `MoviesListOptions` serializes and deserializes with all twelve column-visibility values intact.
- **Integration tests** (`VideoIndexer.Infrastructure.Tests`) — `DapperMovieCatalogRepositoryTests.GetMovieListAsync_*`: verify the query returns inserted movies, aggregates tag weight correctly, and returns an empty list on an empty schema. Self-skip when no live DB is configured.
- **Manual smoke test** — run the M3 smoke test to populate the database; then verify each acceptance criterion by hand.


## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Modified column absent from live schema** | Step 1 verifies before any code is written. If absent, the conditional migration path (Step 1 / Step 15) adds the column. Both paths are fully planned. |
| **Avalonia DataGrid binding warnings treated as errors** | Prototype the DataGrid AXAML in isolation with a minimal `x:DataType` binding before writing the full view. Fix any compiler warnings before proceeding. |
| **`GetMovieListAsync` performance on large catalogs** | The query uses indexed `movie_id` JOINs and is expected to be fast for hundreds-to-low-thousands of rows. Pagination is noted as a follow-up if needed. |
| **Column visibility `Dictionary<string, bool>` JSON round-trip** | Test the `MoviesListOptions` serialization (serialize → deserialize → compare) in a unit test to verify the dictionary survives JSON round-trip via `System.Text.Json`. |
| **`MoviesListView` compiled bindings fail on DataGrid column templates** | Avalonia 11 DataGrid `DataGridTextColumn.Binding` uses compiled binding paths; test compile in a DEBUG build early. Use `{Binding Path=…, x:DataType=…}` syntax where required. |
| **M3 deferred debt: `RefreshCountAsync` exception handler** | Verify the M3 Rework 1 fix (`_ = RefreshCountAsync()` now has an exception handler) is present before starting M4. If absent, add it as a prerequisite WP. |
| **DataGrid renders unstyled without theme style include** | Add `<StyleInclude Source="avares://Avalonia.Controls.DataGrid/Themes/Fluent.xaml"/>` to `<Application.Styles>` in `App.axaml` as part of Step 2. Verify visual rendering in a DEBUG build before proceeding with column layout work. |
| **`DataGrid.SelectedItems` not two-way bindable in Avalonia 11** | Synchronize multi-selection via a `SelectionChanged` event handler in `MoviesListView.axaml.cs` (Step 14). Test Ctrl+Click and Shift+Click selection in a DEBUG build to confirm `SelectedMovies` is populated before wiring context menu actions. |
| **DataGrid empty on initial app launch without activation trigger** | Add a `Loaded` event handler to `MoviesListView.axaml.cs` (Step 14) that calls `vm.LoadAsync(CancellationToken.None)` fire-and-forget. Test by launching to the Ready state and confirming the grid is non-empty before any user interaction. |
