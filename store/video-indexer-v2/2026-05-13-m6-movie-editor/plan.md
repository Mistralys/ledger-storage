# Plan — M6: Movie Editor

## Summary

Milestone 6 delivers the **Movie Editor** — the central workspace for viewing and editing a single movie's metadata, actors, review status, and utilities. Opening the editor replaces the movies list in the main content area with a three-column layout: a left sidebar with all editable metadata fields, a center tab strip (stubs for Cover Image, Thumbnails, and Video — to be wired by M9 and M10), and a right sidebar stub reserved for the Tagging control (M7). The editor supports field editing with change tracking, Apply Changes / Save & Exit / unsaved-changes confirmation, review-status management, view-counter reset, the Label Cleaner dialog (filename→metadata parser), and a Movie Properties dialog (read-only technical metadata from the DB and filesystem; ffprobe-based fields deferred to M9).

M6 also opens the movies list's context menu for the Edit action and wires double-click on a grid row.

Carry-forward items from the M5 Rework synthesis are resolved as WP-001 (lifecycle hardening) and WP-008 (manifest spec-count correction and cosmetic debt fix).

---

## Architectural Context

**Layer overview relevant to M6:**

| Layer | Key existing modules |
|---|---|
| Core/Models | `MovieListItem`, `Movie` *(new)*, `LabelCleanerResult` *(new)* |
| Core/Abstractions | `IMovieCatalogRepository`, `IMovieRepository` *(new)*, `INameParser` *(new)* |
| Core/Options | `AppOptions`, `MoviesListOptions`, `LabelCleanerOptions` *(new)* |
| Infrastructure/Database/migrations | `m036_add_file_created.sql`, `m037_add_filter_slots.sql`, `m038_add_year_season_episode.sql` *(new)* |
| Infrastructure/Database | `DatabaseBootstrapper` (revision bump: 37 → 38) |
| Infrastructure/Library | `DapperMovieCatalogRepository`, `DapperMovieRepository` *(new)*, `NameParser` *(new)* |
| App/Services | `IFilterManagerService`, `AvaloniaFilterManagerService`, `ILabelCleanerService` *(new)*, `AvaloniaLabelCleanerService` *(new)*, `IMoviePropertiesService` *(new)*, `AvaloniaMoviePropertiesService` *(new)* |
| App/ViewModels | `MainContentViewModel`, `MoviesListViewModel`, `MovieEditorViewModel` *(new)*, `LabelCleanerViewModel` *(new)*, `MoviePropertiesViewModel` *(new)* |
| App/Views | `MainContentPage` (enum), `MoviesListView.axaml/.cs`, `MovieEditorView.axaml/.cs` *(new)*, `LabelCleanerView.axaml/.cs` *(new)*, `MoviePropertiesView.axaml/.cs` *(new)* |
| App/Program.cs | DI registrations |

**Existing integration points relied upon:**

- `MainContentViewModel.CurrentChildViewModel` + `ViewLocator` — already handles inline navigation (LibraryFolders, RefreshOverlay); M6 adds `MovieEditor` to `MainContentPage`.
- `IConnectionEditorService` / `AvaloniaConnectionEditorService` pattern in `src/VideoIndexer.App/Services/` — M6 follows this for `ILabelCleanerService` and `IMoviePropertiesService`.
- `MoviesListViewModel.SelectedMovie` / `SelectedMovies` — already populated via `MoviesListView.axaml.cs` code-behind.
- `DapperMovieCatalogRepository.InsertMovieAsync` — must be updated to include the three new columns once m038 runs.

---

## Approach / Architecture

### Editor as inline navigation (not a modal dialog)

The editor is a `UserControl` (`MovieEditorView`) resolved by `ViewLocator` when `MainContentViewModel.CurrentChildViewModel` is a `MovieEditorViewModel`. This mirrors the `LibraryFoldersView` pattern: the `ContentControl` in `MainContentView.axaml` renders whatever VM is currently active. No separate Window is opened for the editor.

**Navigation flow:**
1. User double-clicks a row or right-clicks → Edit in `MoviesListView`.
2. `MoviesListViewModel.OpenEditorForSelectedMovieCommand` fires `EditMovieRequested(MovieListItem)`.
3. `MainContentViewModel` (subscribed at construction) calls `OpenMovieEditorAsync(item)`.
4. `MainContentViewModel` loads the full `Movie` from `IMovieRepository.GetByIdAsync(item.MovieId)`, creates a `MovieEditorViewModel` via factory, subscribes to `editor.CloseRequested`, sets `CurrentPage = MainContentPage.MovieEditor`, sets `CurrentChildViewModel = editorVm`.
5. When the editor requests close, `MainContentViewModel` unsubscribes, calls `CloseSubView()` (reverts to `Default` + `MoviesListVm`), then calls `MoviesListVm.LoadAsync()` to refresh the list.

**Factory registration:** `Func<Movie, MovieEditorViewModel>` is registered in `Program.cs` and injected into `MainContentViewModel`. This avoids DI-resolved transient construction with a specific `Movie` parameter.

### Change tracking and close guard

`MovieEditorViewModel` tracks `HasChanges` via a private `bool` updated whenever any editable property changes from its loaded value. The close button is disabled when `HasChanges` is true. The editor exposes:

- `ApplyChangesCommand` — saves without closing.
- `SaveAndExitCommand` — saves, then fires `CloseRequested`.
- `DiscardChangesCommand` (enabled only when `HasChanges`) — resets all fields to loaded values.
- `CloseCommand` (enabled only when `!HasChanges`) — fires `CloseRequested` directly.

This avoids a confirmation dialog service for M6 while still honoring the spec's intent (user cannot exit with unsaved changes without deliberate action). A future lifecycle WP can add a confirmation dialog once a general `IDialogService` is introduced.

### Schema migration m038

`year`, `season`, and `episode` exist as dedicated columns in the legacy DB at revision 33+ (confirmed via legacy-app reference; `structure.sql` is not present in this repository). The rebuild's `movies` table currently lacks them. Migration m038 uses `ALTER TABLE … ADD COLUMN IF NOT EXISTS` (supported since MariaDB 10.0.2) to safely add them to both fresh installs and existing legacy databases. `DatabaseBootstrapper.ExpectedRevision` bumps to 38.

`DapperMovieCatalogRepository.InsertMovieAsync` is updated to explicitly set `year = NULL, season = NULL, episode = NULL` for new records to avoid ambiguity, though the columns default to NULL anyway.

### Movie model

`Movie` is a new sealed record in `VideoIndexer.Core/Models/` carrying all editable fields: `MovieId`, `Hash`, `Label`, `ActorNames`, `Studio`, `Description`, `Rating`, `Year?`, `Season?`, `Episode?`, `NeedsReview`, `ReviewMessage?`, `ViewCount`, `FileCreated?`. It is the domain read/write model complementing `MovieListItem` (which is the read-optimised list projection).

### IMovieRepository

New interface in `VideoIndexer.Core/Abstractions/`:
- `GetByIdAsync(long movieId, CancellationToken)` → `Movie?`
- `SaveAsync(Movie movie, CancellationToken)` — updates all editable columns by `movie_id`.
- `ResetViewCountAsync(long movieId, CancellationToken)` — sets `watch_count = 0`.
- `SetReviewAsync(long movieId, bool needsReview, string? message, CancellationToken)` — sets `review`/`review_message`.
- `GetOriginalFilenameAsync(long movieId, CancellationToken)` → `string?` (from `movies_filenames`).
- `GetKnownActorNamesAsync(CancellationToken)` → `IReadOnlyList<string>` (distinct, parsed from all `name` values; tokenised by comma).
- `GetKnownStudiosAsync(CancellationToken)` → `IReadOnlyList<string>` (distinct non-empty `studio` values).

`ApplyChangesCommand` in the ViewModel calls `IMovieRepository.SaveAsync`. Review commands call `IMovieRepository.SetReviewAsync`. View-counter reset calls `IMovieRepository.ResetViewCountAsync`. These are all separate so callers do not need to load and re-save the entire `Movie` for single-field operations.

### IAppPaths extension

`IAppPaths.MovieDataDirectory(string hash)` is added (returns the path to the movie's thumbnails/data folder, following the legacy convention `[Root]/data/[hash]/`). The App layer's `AppPaths` class implements it. This is needed for display in the Properties dialog and will be fully consumed in M9.

### Label Cleaner

`INameParser` (Core/Abstractions) is a stateless interface:
```csharp
LabelCleanerResult Parse(
    string filenameWithoutExtension,
    LabelCleanerOptions options,
    IReadOnlyList<string> knownActorNames,
    IReadOnlyList<string> knownStudios);
```

`NameParser` (Infrastructure/Library) implements the algorithm: date extraction (regex-based), separator normalisation (dots, underscores, brackets → spaces/separators), studio detection (longest-match against `knownStudios`), actor name detection (longest-match against `knownActorNames`), keyword stripping (user-configurable keyword list from `LabelCleanerOptions.Keywords`), UCWords, label assembly from remaining parts. Tag detection is stubbed (returns empty list until M7 provides the tag vocabulary).

`LabelCleanerResult` (Core/Models): `string ActorNames`, `string Label`, `string Studio`, `IReadOnlyList<string> DetectedTags`.

`LabelCleanerOptions` (Core/Options): `bool PreserveDates`, `bool UCWords`, `bool DetectTags` (stub), `bool StripKeywords`, `IReadOnlyList<string> Keywords`.

`LabelCleanerViewModel` + `LabelCleanerView.axaml` follow the `ConnectionEditorViewModel` / dialog service pattern for result-returning dialogs: the VM exposes `event EventHandler<LabelCleanerResult?>? CloseRequested` and the Avalonia service shows a modal Window, returning the accepted result via `dialog.ShowDialog<LabelCleanerResult?>`.

**`LabelCleanerViewModel` constructor (option a — pre-fetched lists):** `MovieEditorViewModel.CleanLabelCommand` fetches `knownActorNames` and `knownStudios` from `IMovieRepository` and the original filename from `IMovieRepository.GetOriginalFilenameAsync` immediately before calling `ILabelCleanerService.ShowAsync`. The service constructs the VM as:

```csharp
new LabelCleanerViewModel(
    INameParser nameParser,
    string originalFilename,
    LabelCleanerOptions options,
    IReadOnlyList<string> knownActorNames,
    IReadOnlyList<string> knownStudios)
```

The VM runs an initial parse on construction and re-parses whenever any `LabelCleanerOptions` toggle changes (via `partial void OnXxxChanged` callbacks calling `Reparse()`). `INameParser` and the name lists are held as `readonly` fields — no DB access happens inside the VM itself.

### Movie Properties dialog

`MoviePropertiesViewModel` (App/ViewModels) receives a `Movie` + supplementary data (`string? OriginalFilename`, `long FileSizeBytes`, `string FilePath`, `string DataFolderPath`) and exposes them as read-only display properties. Technical metadata (Resolution, Duration, Bitrate, File Type) that requires ffprobe is explicitly stubbed out (shown as "—") with a comment referencing M9. The "Open Data Folder" and "Show in Folder" actions use `Process.Start` shell commands via `IFileLauncherService` (new, App/Services).

`IMoviePropertiesService` follows the `IFilterManagerService` dialog pattern (takes a pre-built VM, shows modal Window).

### Cosmetic carry-forward fix

`HorizontalAlignment='Stretch'` on buttons inside a `Horizontal` StackPanel in `FiltersManagerView.axaml` (flagged in M5 Rework WP-003) is corrected in WP-008 alongside the manifest spec-count correction.

---

## Rationale

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|---|---|---|---|
| Editor as inline navigation | `MainContentPage.MovieEditor` + `ContentControl` | Separate top-level `Window` | Inline navigation is consistent with existing `LibraryFolders`/`RefreshOverlay` pattern; avoids a second window chrome and z-order issues. |
| Close guard via disabled button + explicit Discard | Explicit Discard/Close commands; no dialog service | `IConfirmationDialogService` showing a MessageBox | No general confirmation service exists yet; introducing one for a single use case conflicts with the no-speculative-abstraction constraint. Explicit buttons satisfy the spec's "Save / Discard / Cancel" intent without a dialog layer. |
| `IMovieRepository` as a separate interface | Distinct interface from `IMovieCatalogRepository` | Extend `IMovieCatalogRepository` | `IMovieCatalogRepository` is a list-centric read interface; mixing single-record write operations into it violates SRP and would require the list tests to be aware of editor operations. |
| `Func<Movie, MovieEditorViewModel>` factory | DI-registered delegate | `IMovieEditorViewModelFactory` interface | A typed delegate is the minimal shape for parameterised construction from DI; introducing a single-method interface (ISP violation in reverse) adds a file and a registration for no testability benefit. |
| NameParser as stateless `INameParser` | Takes known names as parameters | Constructor injection of `IMovieRepository` | Stateless function is maximally testable — unit tests supply arbitrary name lists without a DB. Known names are fetched once per dialog open by the ViewModel. |
| Properties dialog — ffprobe fields deferred | Show DB + filesystem data; stub ffprobe fields | Run ffprobe on dialog open | M9 already plans the ffprobe pipeline for thumbnail generation. Duplicating the probe call here creates competing ffprobe invocations and a partial implementation that is harder to test. |
| `LabelCleanerViewModel` known-names sourcing | Pre-fetch in `MovieEditorViewModel.CleanLabelCommand`; pass as constructor params | (a) Inject `IMovieRepository` into the VM, (b) inject into the service | Option (a) keeps the VM stateless w.r.t. the DB and maximally testable; the VM holds pre-fetched lists as `readonly` fields with no async DB calls at parse time. Option (b) would move fetch logic into the Avalonia service layer, obscuring it from unit tests. |
| `SetReview` prompt mechanism | Inline collapsible TextBox + Confirm/Cancel buttons in the left sidebar | (a) New `IReviewMessageService` dialog, (b) Avalonia `MessageBox` | A collapsible panel is a single-field input requiring no new service, no new dialog VM, and no new dialog Window. Both alternatives add disproportionate abstraction surface for a single string field. `IsReviewMessagePanelVisible` toggle keeps the state machine entirely within `MovieEditorViewModel`, which is fully testable without a dialog service. |

---

## Pattern Alignment

| Pattern | Status |
|---|---|
| `MainContentPage` enum + `ContentControl` inline navigation — `src/VideoIndexer.App/ViewModels/MainContentViewModel.cs` | **Followed**: M6 adds `MovieEditor` value and reuses the existing navigation mechanism |
| Dialog service, fire-and-forget (`IFilterManagerService` / `AvaloniaFilterManagerService`) — `src/VideoIndexer.App/Services/` | **Followed**: `IMoviePropertiesService` follows this pattern (no return value) |
| Dialog service, result-returning (`IConnectionEditorService` / `AvaloniaConnectionEditorService`) — `src/VideoIndexer.App/Services/` | **Followed**: `ILabelCleanerService` / `AvaloniaLabelCleanerService` follow this pattern (`Task<LabelCleanerResult?>` return value; `event EventHandler<LabelCleanerResult?>?` on the VM; `dialog.Close(result)` + `ShowDialog<LabelCleanerResult?>(owner)` mechanism) |
| `CloseRequested` event on dialog VMs — `FiltersManagerViewModel` | **Followed**: `MovieEditorViewModel` uses the same event for inline close signalling |
| Dapper parameterised SQL — `DapperFilterSlotRepository` | **Followed**: All SQL in `DapperMovieRepository` uses named Dapper parameters |
| `IMovieCatalogRepository.InsertMovieAsync` uses `INSERT IGNORE` — `DapperMovieCatalogRepository` | **Extended**: Updated to include nullable `year`/`season`/`episode` columns |
| `.ConfigureAwait(false)` on all Infrastructure/Core `await`s | **Followed**: All new Infrastructure methods call `.ConfigureAwait(false)` |
| `TreatWarningsAsErrors` / no nullable warnings | **Followed**: All new code must compile without warnings |
| No external NuGet in Core | **Followed**: `INameParser`, `LabelCleanerResult`, `LabelCleanerOptions`, `Movie` are pure C# records/interfaces |
| ViewLocator convention (`ViewModels.FooViewModel` → `Views.FooView`) | **Followed**: `MovieEditorViewModel` → `MovieEditorView` |
| Parameterless view constructors in DI | **Followed**: `MovieEditorView`, `LabelCleanerView`, `MoviePropertiesView` all use `new FooView()` in factory lambdas |

---

## Detailed Steps

1. **WP-001 — Lifecycle hardening (M5 carry-forward)**
   - Wrap both `await vm.LoadAsync()` and `await vm.LoadFilterSlotsAsync()` in `MoviesListView.axaml.cs` Loaded handler in a `try/catch(Exception ex)` block; log via the ViewModel's `HasLoadError`/`LoadErrorMessage` path on failure.
   - Pass a `CancellationToken` sourced from the view's `CancellationTokenSource` when calling `LoadFilterSlotsAsync` (`MoviesListViewModel.LoadFilterSlotsAsync` already has an optional `CancellationToken` parameter; no signature change is needed).
   - Update the Loaded handler to pass a `CancellationToken` sourced from a CancellationTokenSource stored in the view (cancelled in `Unloaded`), so in-flight startup I/O can be cancelled if the view is removed from the visual tree.
   - Run the full test suite; confirm 500+ tests pass, 0 failures.

2. **WP-002 — Schema migration m038 + revision bump**
   - Create `src/VideoIndexer.Infrastructure/Database/migrations/m038_add_year_season_episode.sql`:
     ```sql
     ALTER TABLE movies
       ADD COLUMN IF NOT EXISTS year    INT UNSIGNED NULL DEFAULT NULL,
       ADD COLUMN IF NOT EXISTS season  INT UNSIGNED NULL DEFAULT NULL,
       ADD COLUMN IF NOT EXISTS episode INT UNSIGNED NULL DEFAULT NULL;
     UPDATE spdb_config SET config_value = '38' WHERE config_name = 'db_revision';
     ```
   - Set `DatabaseBootstrapper.ExpectedRevision = 38`.
   - Update `DapperMovieCatalogRepository.InsertMovieAsync` SQL to include `year, season, episode` in the column list with `NULL, NULL, NULL` values.
   - Update integration test `ExpectedRevision_MatchesCurrentSchemaRevision` assertion value to 38.
   - Run the full test suite; confirm the revision test passes; confirm integration tests pass against a migrated DB.

3. **WP-003 — `Movie` model + `IMovieRepository` + `IAppPaths.MovieDataDirectory`**
   - Add `Movie` sealed record to `src/VideoIndexer.Core/Models/Movie.cs`.
   - Add `IMovieRepository` to `src/VideoIndexer.Core/Abstractions/IMovieRepository.cs` with all methods listed in the Approach section.
   - Add `MovieDataDirectory(string hash)` to `IAppPaths` and implement in `src/VideoIndexer.Infrastructure/AppPaths.cs` (path: `Path.Combine(Root, "data", hash)`).
   - Add `DapperMovieRepository` to `src/VideoIndexer.Infrastructure/Library/DapperMovieRepository.cs`.
   - Register `IMovieRepository → DapperMovieRepository` as a singleton in `Program.cs`.
   - Add `FakeMovieRepository.cs` to `tests/VideoIndexer.App.Tests/TestHelpers/`.
   - Write unit tests for `DapperMovieRepository` (integration tests; self-skip pattern) and for `GetKnownActorNamesAsync` tokenisation logic.

4. **WP-004 — Navigation plumbing**
   - Add `MovieEditor` value to `MainContentPage` enum in `src/VideoIndexer.App/ViewModels/MainContentPage.cs`.
   - Add `event EventHandler<MovieListItem>? EditMovieRequested` to `MoviesListViewModel`.
   - Add `[RelayCommand] void OpenEditorForSelectedMovie()` to `MoviesListViewModel` — fires `EditMovieRequested(SelectedMovie)` when `SelectedMovie != null`.
   - Extend `MainContentViewModel`:
     - Constructor receives `IMovieRepository` and `Func<Movie, MovieEditorViewModel>` factory.
     - On construction, subscribe to `MoviesListVm.EditMovieRequested`.
     - Add `async Task OpenMovieEditorAsync(MovieListItem item)` — loads `Movie`, creates `MovieEditorViewModel` via factory, subscribes to `editorVm.CloseRequested`, sets `CurrentPage = MovieEditor` and `CurrentChildViewModel = editorVm`.
     - Private handler for `editorVm.CloseRequested`: unsubscribes, calls `CloseSubView()`, calls `MoviesListVm.LoadAsync(CancellationToken.None)` (fire-and-forget with `.ContinueWith` fault log — same pattern as `OnRefreshStateChanged`).
   - Update `MainContentViewModel.Dispose()` to unsubscribe from `MoviesListVm.EditMovieRequested`.
   - Update `MoviesListView.axaml` — add `DataGrid.ContextMenu` with an "Edit" `MenuItem` bound to `OpenEditorForSelectedMovieCommand`; stub "Generate Thumbnails", "Copy To…", "Delete on Disk" as disabled items.
   - Update `MoviesListView.axaml.cs` — handle `MoviesGrid.DoubleTapped` to call `vm.OpenEditorForSelectedMovieCommand.Execute(null)`.
   - Write unit tests in `tests/VideoIndexer.App.Tests/MainContentViewModelTests.cs` (extending the existing file to share its `BuildSut()` helper) for the navigation:
     - Opening the editor navigates `CurrentPage` to `MovieEditor`.
     - `CloseRequested` from the editor VM returns `CurrentPage` to `Default`.
     - `MoviesListVm.LoadAsync` is called after editor close.

5. **WP-005 — `MovieEditorViewModel` + `MovieEditorView` (metadata + layout)**
   - Create `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs`:
     - Constructor takes `Movie movie`, `IMovieRepository movieRepository`, `INameParser nameParser`, `ILabelCleanerService labelCleanerService`, `IMoviePropertiesService propertiesService`, `ISettingsService settingsService` (used by `CleanLabelCommand` to read the persisted `LabelCleaner` options from `AppOptions`).
     - `[ObservableProperty]` fields for every editable field: `LabelText`, `ActorNamesText`, `StudioText`, `DescriptionText`, `Rating` (int), `Year?` (int), `Season?` (int), `Episode?` (int).
     - `bool HasChanges` — computed from comparison to the original loaded `Movie`; updated in `partial void OnXxxChanged` callbacks.
     - `event EventHandler? CloseRequested`.
     - `NeedsReview` (bool, readonly display) + `ReviewMessage?` — loaded from `Movie`; updated only via `SetReviewCommand` / `MarkReviewedCommand`.
     - `ViewCount` (int, readonly display) — loaded from `Movie`.
     - `[RelayCommand]` `ApplyChanges` — persists via `IMovieRepository.SaveAsync`; resets `HasChanges`.
     - `[RelayCommand]` `SaveAndExit` — calls `ApplyChanges` then fires `CloseRequested`.
     - `[RelayCommand(CanExecute = nameof(HasChangesIsTrue))]` `DiscardChanges` — reloads all fields from the original `Movie`; resets `HasChanges`.
     - `[RelayCommand(CanExecute = nameof(HasChangesIsFalse))]` `Close` — fires `CloseRequested`.
     - `[RelayCommand]` `MoveActorUp` / `MoveActorDown` — reorders comma-separated actor names.
     - `[RelayCommand]` `SetReview` — sets `IsReviewMessagePanelVisible = true` (shows the inline collapsible TextBox in the left sidebar); user types an optional message and clicks "Confirm Review"; `ConfirmSetReviewCommand` then calls `IMovieRepository.SetReviewAsync(id, true, ReviewMessage)`; updates `NeedsReview` and hides the panel.
     - `bool IsReviewMessagePanelVisible` (`[ObservableProperty]`, default `false`) — controls the collapsible area visibility.
     - `[RelayCommand]` `ConfirmSetReview` — executes the actual `SetReviewAsync` call and collapses the panel.
     - `[RelayCommand]` `CancelSetReview` — hides the panel without making any DB call.
     - `[RelayCommand]` `MarkReviewed` — calls `IMovieRepository.SetReviewAsync(id, false, null)`; updates flags.
     - `[RelayCommand]` `ResetViewCounter` — calls `IMovieRepository.ResetViewCountAsync`; updates `ViewCount`.
     - `[RelayCommand]` `CleanLabel` — opens `ILabelCleanerService`; on accept, applies result to actor/label/studio fields.
     - `[RelayCommand]` `ShowProperties` — opens `IMoviePropertiesService`.
     - Two parameterless constructors: design-time (all services null, no-ops) and DI full constructor.
   - Create `src/VideoIndexer.App/Views/MovieEditorView.axaml`:
     - Three-column Grid layout (left sidebar, center tabs, right sidebar stub).
     - **Left sidebar**: fields for all editable properties; Move Actor Up/Down buttons; Apply Changes, Save & Exit, Discard, Close toolbar; Review status section (label showing current status/message, "To Review" and "Reviewed" buttons; collapsible `IsVisible="{Binding IsReviewMessagePanelVisible}"` area containing a TextBox for the optional review message plus "Confirm Review" and "Cancel" buttons); Reset View Counter button; Clean Label button.
     - **Center tabs** (`TabControl`): "Cover Image" tab (stub — `TextBlock "Cover image — M9"`), "Thumbnails" tab (stub), "Video" tab (stub).
     - **Right sidebar**: stub `TextBlock "Tagging — M7"`.
     - **Left sidebar bottom**: stub `TextBlock "Bookmarks — M10"` placed below all metadata fields, consistent with the center-tab stub pattern.
     - All bindings via compiled bindings (`x:DataType`).
   - Register `MovieEditorView` in DI and in `Program.cs` (`builder.Services.AddTransient<MovieEditorView>(_ => new MovieEditorView())`).
   - Register the `Func<Movie, MovieEditorViewModel>` factory in `Program.cs`.
   - Write `tests/VideoIndexer.App.Tests/MovieEditorViewModelTests.cs`:
     - HasChanges is false on construction.
     - HasChanges is true after any field modification.
     - HasChanges is false after DiscardChanges.
     - ApplyChanges calls `IMovieRepository.SaveAsync` with correct data.
     - SaveAndExit fires `CloseRequested` after successful save.
     - Close fires `CloseRequested` only when `HasChanges = false`.
     - MoveActorUp / MoveActorDown reorders correctly.
     - ResetViewCounter calls `IMovieRepository.ResetViewCountAsync`.
     - SetReview / MarkReviewed call `IMovieRepository.SetReviewAsync` with correct args.

6. **WP-006 — Label Cleaner**
   - Create `src/VideoIndexer.Core/Models/LabelCleanerResult.cs` (sealed record).
   - Create `src/VideoIndexer.Core/Options/LabelCleanerOptions.cs` (sealed record); add a `LabelCleaner` property of type `LabelCleanerOptions` to `AppOptions` (`public LabelCleanerOptions LabelCleaner { get; init; } = new()`); add a `"LabelCleaner": { "PreserveDates": true, "UCWords": true, "DetectTags": false, "StripKeywords": false, "Keywords": [] }` section to `src/VideoIndexer.App/Assets/appsettings.json`.
   - Create `src/VideoIndexer.Core/Abstractions/INameParser.cs`.
   - Create `src/VideoIndexer.Infrastructure/Library/NameParser.cs` implementing `INameParser`:
     - Date detection (three regex patterns from legacy app).
     - Separator normalisation (dots, underscores, brackets).
     - Studio detection (longest-match against `knownStudios`, case-insensitive).
     - Actor name detection (longest-match against `knownActorNames`, case-insensitive; sorted by length descending).
     - Keyword stripping (when `StripKeywords` is true, matches against `LabelCleanerOptions.Keywords`).
     - UCWords option.
     - `DetectedTags = []` (stub — activated in M7).
   - Create `src/VideoIndexer.App/Services/ILabelCleanerService.cs` — `Task<LabelCleanerResult?> ShowAsync(LabelCleanerViewModel viewModel, CancellationToken ct)`.
   - Create `src/VideoIndexer.App/Services/AvaloniaLabelCleanerService.cs` — follows `AvaloniaConnectionEditorService` pattern: subscribes to `viewModel.CloseRequested += (_, result) => dialog.Close(result)`, then `return await dialog.ShowDialog<LabelCleanerResult?>(owner)`.
   - Create `src/VideoIndexer.App/ViewModels/LabelCleanerViewModel.cs`:
     - Constructor signature: `LabelCleanerViewModel(INameParser nameParser, string originalFilename, LabelCleanerOptions options, IReadOnlyList<string> knownActorNames, IReadOnlyList<string> knownStudios)`.
     - Holds `INameParser`, `knownActorNames`, and `knownStudios` as `readonly` fields.
     - Runs initial parse on construction; `partial void OnXxxChanged` callbacks for each option toggle call a private `Reparse()` method.
     - `[ObservableProperty]` fields: `OriginalFilename`, `ProposedActorNames`, `ProposedLabel`, `ProposedStudio`, `DetectedTags`.
     - `[ObservableProperty]` option toggles: `PreserveDates`, `UCWords`, `StripKeywords`, `DetectTags`.
     - `event EventHandler<LabelCleanerResult?>? CloseRequested` — carries the parsed result; `Accept` fires it with the constructed `LabelCleanerResult`; `Discard` fires it with `null`.
     - `[RelayCommand]` `Accept` — constructs `LabelCleanerResult` from the current proposed fields and fires `CloseRequested` with the result.
     - `[RelayCommand]` `Discard` — fires `CloseRequested` with `null` (no result applied to the caller).
   - In `MovieEditorViewModel.CleanLabelCommand`: before calling `ILabelCleanerService.ShowAsync`, await `IMovieRepository.GetOriginalFilenameAsync`, `IMovieRepository.GetKnownActorNamesAsync`, and `IMovieRepository.GetKnownStudiosAsync`; pass the results to the `LabelCleanerViewModel` constructor.
   - Create `src/VideoIndexer.App/Views/LabelCleanerView.axaml`.
   - Register `INameParser → NameParser` as singleton in `Program.cs`.
   - Register `ILabelCleanerService` singleton in `Program.cs`.
   - Write `tests/VideoIndexer.Tests/LabelCleanerTests.cs` covering:
     - Date extraction (three formats).
     - Actor name detection (exact match, partial-before-full protection, case-insensitive).
     - Studio detection (longest-match).
     - UCWords transformation.
     - Keyword stripping.
     - Empty known-names list graceful handling.

7. **WP-007 — Movie Properties dialog**
   - Create `src/VideoIndexer.App/Services/IFileLauncherService.cs` — `void OpenFolder(string path)`, `void ShowInExplorer(string filePath)`.
   - Create `src/VideoIndexer.App/Services/WindowsFileLauncherService.cs` — implements via `Process.Start("explorer.exe", ...)`.
   - Create `src/VideoIndexer.App/Services/IMoviePropertiesService.cs` — `Task ShowAsync(MoviePropertiesViewModel vm, CancellationToken ct)`.
   - Create `src/VideoIndexer.App/Services/AvaloniaMoviePropertiesService.cs`.
   - Create `src/VideoIndexer.App/ViewModels/MoviePropertiesViewModel.cs`:
     - Receives `Movie movie`, `string? originalFilename`, `string filePath`, `long fileSizeBytes`, `string dataFolderPath`.
     - Read-only display properties: `MovieId`, `Hash`, `FilePath`, `FileSize` (formatted), `DataFolder`, `ViewCount`, `OriginalFilename`, `FileCreated`.
     - Technical fields (`FileType`, `Resolution`, `Duration`, `Bitrate`) show `"—"` with `//TODO: M9 — populate via ffprobe` comment.
     - `[RelayCommand]` `OpenDataFolder` — calls `IFileLauncherService.OpenFolder(DataFolder)`.
     - `[RelayCommand]` `ShowInFolder` — calls `IFileLauncherService.ShowInExplorer(FilePath)`.
   - Create `src/VideoIndexer.App/Views/MoviePropertiesView.axaml` (read-only form).
   - Register all new services in `Program.cs`.
   - Write `tests/VideoIndexer.App.Tests/MoviePropertiesViewModelTests.cs` covering property initialization and file-size formatting.

8. **WP-008 — Cosmetic and manifest carry-forwards**
   - Fix `HorizontalAlignment='Stretch'` on buttons in `src/VideoIndexer.App/Views/FiltersManagerView.axaml` (remove the attribute from buttons inside the Horizontal StackPanel).
   - Update `docs/agents/plans/2026-05-12-m5-filters-search-rework-1/plan.md` (if it exists as a live document) or annotate `api-surface.md` — correct the `FilterExpressionEvaluatorTests` test count from 57 to 56.
   - Run the full test suite; confirm no regressions.

9. **WP-009 — Manifest updates**
   - `docs/agents/project-manifest/api-surface.md` — add `Movie` model, `IMovieRepository`, `LabelCleanerResult`, `LabelCleanerOptions`, `INameParser`, `MovieEditorViewModel`, `LabelCleanerViewModel`, `MoviePropertiesViewModel`; update `IAppPaths` with `MovieDataDirectory`.
   - `docs/agents/project-manifest/file-tree.md` — add all new source files in their correct locations.
   - `docs/agents/project-manifest/constraints.md` — update expected schema revision to 38; add m038 rollback procedure; add `IAppPaths.MovieDataDirectory` convention; add note that movie editor navigation fires list reload; remove the stale Open Item entry “MoviesListView — `LoadFilterSlotsAsync` not yet wired” (resolved in M5 Rework WP-001).
   - `docs/agents/project-manifest/data-flows.md` — add section 9 "Movie Edit Flow" covering the full navigation sequence.

---

## Dependencies

- M5 Rework (complete) — provides runtime-wired filter slot foundation and test suite at 500+.
- MariaDB 10.0.2+ required for `ADD COLUMN IF NOT EXISTS` syntax (confirmed supported in all target versions).
- The `IFfprobeRunner` is **not** a dependency of M6 — Properties dialog stubs the ffprobe-dependent fields.

---

## Required Components

### New files
| File | Type |
|------|------|
| `src/VideoIndexer.Infrastructure/Database/migrations/m038_add_year_season_episode.sql` | Migration SQL |
| `src/VideoIndexer.Core/Models/Movie.cs` | Core model |
| `src/VideoIndexer.Core/Models/LabelCleanerResult.cs` | Core model |
| `src/VideoIndexer.Core/Options/LabelCleanerOptions.cs` | Core options |
| `src/VideoIndexer.Core/Abstractions/IMovieRepository.cs` | Core interface |
| `src/VideoIndexer.Core/Abstractions/INameParser.cs` | Core interface |
| `src/VideoIndexer.Infrastructure/Library/DapperMovieRepository.cs` | Infrastructure impl |
| `src/VideoIndexer.Infrastructure/Library/NameParser.cs` | Infrastructure impl |
| `src/VideoIndexer.App/Services/ILabelCleanerService.cs` | Service interface |
| `src/VideoIndexer.App/Services/AvaloniaLabelCleanerService.cs` | Service impl |
| `src/VideoIndexer.App/Services/IMoviePropertiesService.cs` | Service interface |
| `src/VideoIndexer.App/Services/AvaloniaMoviePropertiesService.cs` | Service impl |
| `src/VideoIndexer.App/Services/IFileLauncherService.cs` | Service interface |
| `src/VideoIndexer.App/Services/WindowsFileLauncherService.cs` | Service impl |
| `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs` | App VM |
| `src/VideoIndexer.App/ViewModels/LabelCleanerViewModel.cs` | App VM |
| `src/VideoIndexer.App/ViewModels/MoviePropertiesViewModel.cs` | App VM |
| `src/VideoIndexer.App/Views/MovieEditorView.axaml` + `.cs` | App View |
| `src/VideoIndexer.App/Views/LabelCleanerView.axaml` + `.cs` | App View |
| `src/VideoIndexer.App/Views/MoviePropertiesView.axaml` + `.cs` | App View |
| `tests/VideoIndexer.App.Tests/MovieEditorViewModelTests.cs` | Unit tests |
| `tests/VideoIndexer.App.Tests/LabelCleanerViewModelTests.cs` | Unit tests |
| `tests/VideoIndexer.App.Tests/MoviePropertiesViewModelTests.cs` | Unit tests |
| `tests/VideoIndexer.App.Tests/TestHelpers/FakeMovieRepository.cs` | Test helper |
| `tests/VideoIndexer.Tests/LabelCleanerTests.cs` | Unit tests |

### Modified files
| File | Change |
|------|--------|
| `src/VideoIndexer.Infrastructure/Database/DatabaseBootstrapper.cs` | `ExpectedRevision = 38` |
| `src/VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs` | Add year/season/episode to `InsertMovieAsync` SQL |
| `src/VideoIndexer.Core/Options/AppOptions.cs` | Add `LabelCleaner` property of type `LabelCleanerOptions` |
| `src/VideoIndexer.App/Assets/appsettings.json` | Add `"LabelCleaner"` default section |
| `src/VideoIndexer.Core/Abstractions/IAppPaths.cs` | Add `MovieDataDirectory(string hash)` |
| `src/VideoIndexer.Infrastructure/AppPaths.cs` | Implement `MovieDataDirectory` |
| `src/VideoIndexer.App/ViewModels/MainContentPage.cs` | Add `MovieEditor` value |
| `src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs` | Add `EditMovieRequested` event + `OpenEditorForSelectedMovieCommand` |
| `src/VideoIndexer.App/ViewModels/MainContentViewModel.cs` | Add movie editor factory, subscription, open/close logic; update Dispose |
| `src/VideoIndexer.App/Views/MoviesListView.axaml` | Add row context menu with Edit stub items |
| `src/VideoIndexer.App/Views/MoviesListView.axaml.cs` | Wire DoubleTapped → open editor command |
| `src/VideoIndexer.App/Views/FiltersManagerView.axaml` | Remove invalid `HorizontalAlignment='Stretch'` on buttons |
| `src/VideoIndexer.App/Program.cs` | Register new services, factory, `MovieEditorView` |
| `tests/VideoIndexer.App.Tests/MainContentViewModelTests.cs` | Update `BuildSut()` to pass `FakeMovieRepository` and a no-op `Func<Movie, MovieEditorViewModel>` factory for the two new constructor parameters |
| `tests/VideoIndexer.Infrastructure.Tests/Database/DatabaseBootstrapperTests.cs` | Update revision assertion to 38 |
| `docs/agents/project-manifest/api-surface.md` | New signatures |
| `docs/agents/project-manifest/file-tree.md` | New file entries |
| `docs/agents/project-manifest/constraints.md` | Revision 38, m038 rollback, `MovieDataDirectory` convention |
| `docs/agents/project-manifest/data-flows.md` | New "Movie Edit Flow" section |

---

## Assumptions

- MariaDB 10.0.2+ is the minimum target — `ADD COLUMN IF NOT EXISTS` is available.
- The `movies` table in all target databases is at revision 37 (post-M5 migration). The m038 migration is applied by the DB administrator or by a future auto-migration feature.
- `Process.Start("explorer.exe", ...)` is acceptable for `IFileLauncherService` in M6 (Windows-only target; `WinExe` output type confirmed).
- `LabelCleanerOptions.Keywords` is persisted in `appsettings.json` under a new `LabelCleaner` section in `AppOptions`. A sensible default set (empty list) is supplied in the bundled `appsettings.json`.
- The `Func<Movie, MovieEditorViewModel>` factory registration pattern (a delegate rather than a typed interface) is consistent with `Program.cs` conventions already established by the `ShellViewModel` factory.

---

## Constraints

- **All warnings are errors** — no CS warnings permitted; all nullability flows must be annotated.
- **Core has no NuGet dependencies** — `Movie`, `LabelCleanerResult`, `LabelCleanerOptions`, `INameParser`, `IMovieRepository` are pure C#.
- **No `Version=` on `<PackageReference>`** — if any new package is needed, version goes only in `Directory.Packages.props`.
- **Parameterised SQL only in `DapperMovieRepository`** — all columns that accept user-supplied data (Label, ActorNames, Studio, Description, ReviewMessage) must use Dapper named parameters.
- **`AppOptions` mutation via `with { }` + `ISettingsService.SaveAsync` only** — the new `LabelCleaner.Keywords` section follows the same `with { }` pattern.
- **`ConfigureAwait(false)` in all Infrastructure/Core await chains.**
- **`MovieEditorViewModel.CloseRequested` event must be unsubscribed** by `MainContentViewModel` after close to prevent memory leaks (the ViewModel is otherwise orphaned).

---

## Out of Scope

- External VLC launch from context menu (M8 External Tools).
- "Copy To…" dialog (M8 System Tools).
- "Delete on Disk" (M8 System Tools).
- Cover Image and Thumbnails tabs (M9 Images).
- Video Player tab and embedded LibVLCSharp (M10 Player & Bookmarks).
- Tagging sidebar (M7 Tagging Core).
- ffprobe-based technical metadata in Properties dialog (M9 Images).
- Multi-movie bulk edit (not in spec for M6).
- `IConfirmationDialogService` (no current consumer beyond editor close guard; explicit Discard/Close buttons satisfy the spec).
- Auto-migration: the m038 migration is not applied automatically; it must be run manually (same convention as m036/m037).

---

## Acceptance Criteria

- [ ] Double-clicking a movie row in the movies list opens the Movie Editor and the movies list is replaced by the editor view.
- [ ] Right-click → Edit on a row opens the Movie Editor for the selected movie.
- [ ] All metadata fields (Label, Actor Names, Studio, Description, Rating, Year, Season, Episode) are displayed and editable.
- [ ] `HasChanges` is false on open; becomes true after any field modification; is false after DiscardChanges.
- [ ] Apply Changes saves to the database and the grid reflects the updated data after navigating back.
- [ ] Save & Exit saves and returns to the movies list.
- [ ] Close button is disabled when `HasChanges = true`; Discard Changes button is enabled when `HasChanges = true`.
- [ ] Move Actor Up / Move Actor Down correctly reorders entries in the Actor Names field.
- [ ] Clicking "To Review" reveals the inline review-message panel (TextBox + Confirm/Cancel); confirming with an optional message calls `SetReviewAsync` and hides the panel; cancelling leaves the flag unchanged.
- [ ] "Reviewed" clears the flag; review message is cleared in the DB.
- [ ] Review column in the movies list updates after returning from the editor.
- [ ] Reset View Counter sets `ViewCount` to 0 in the DB and updates the displayed count.
- [ ] Clean Label dialog opens, shows proposed actor names / label / studio parsed from the original filename, and applies the result on Accept.
- [ ] Label Cleaner date detection works for all three legacy regex patterns.
- [ ] Label Cleaner actor detection matches known names from the database (longest-match, case-insensitive).
- [ ] Label Cleaner UCWords option capitalises first letter of each word.
- [ ] Movie Properties dialog shows Movie ID, Hash, File Path, File Size, Data Folder, Original Filename, View Count.
- [ ] "Open Data Folder" and "Show in Folder" open Explorer in the correct locations.
- [ ] Cover Image, Thumbnails, Video tabs display stub placeholders.
- [ ] Right Sidebar displays a tagging stub placeholder.
- [ ] Left sidebar bottom displays a Bookmarks stub placeholder (`TextBlock "Bookmarks — M10"`).
- [ ] `DatabaseBootstrapper.ExpectedRevision` is 38; migration m038 adds `year`, `season`, `episode` columns.
- [ ] Schema revision assertion in `DatabaseBootstrapperTests` passes at 38.
- [ ] `FiltersManagerView.axaml` no longer has invalid `HorizontalAlignment='Stretch'` on horizontal StackPanel buttons.
- [ ] Full test suite passes: 500+ tests, 0 failures (excluding expected skips).

---

## Testing Strategy

Unit tests cover: `MovieEditorViewModel` (change tracking, commands, HasChanges state machine), `LabelCleanerViewModel` (option toggles trigger re-parse), `NameParser` (all parsing scenarios), `MoviePropertiesViewModel` (property initialisation, file-size formatting), `MainContentViewModel` navigation extension (open editor, close editor, reload).

Integration tests (self-skip without DB) cover: `DapperMovieRepository.GetByIdAsync`, `SaveAsync`, `ResetViewCountAsync`, `SetReviewAsync`, `GetKnownActorNamesAsync`, `GetKnownStudiosAsync`. These live in `tests/VideoIndexer.Infrastructure.Tests/`.

The `HasChanges` state machine and `CloseRequested` event semantics are tested without DB access using `FakeMovieRepository`.

---

## Test Plan

- `tests/VideoIndexer.App.Tests/MovieEditorViewModelTests.cs`
  - `HasChanges_IsFalse_OnConstruction` — AC: HasChanges is false on open.
  - `HasChanges_BecomesTrue_AfterLabelChange` — AC: HasChanges after field edit.
  - `HasChanges_IsFalse_AfterDiscardChanges` — AC: HasChanges after discard.
  - `ApplyChanges_CallsSaveAsync_WithCorrectData` — AC: Apply Changes saves to DB.
  - `SaveAndExit_FiresCloseRequested_AfterSuccessfulSave` — AC: Save & Exit flow.
  - `Close_FiresCloseRequested_WhenNoChanges` — AC: Close button guard.
  - `Close_DoesNotFireCloseRequested_WhenHasChanges` — AC: Close button guard (negative).
  - `MoveActorUp_ReordersActorNamesCorrectly` — AC: Move Actor reorder.
  - `MoveActorDown_ReordersActorNamesCorrectly` — AC: Move Actor reorder.
  - `ResetViewCounter_CallsRepository_AndUpdatesViewCount` — AC: view counter reset.
  - `SetReview_ShowsReviewMessagePanel_WhenInvoked` — AC: inline review panel appears.
  - `ConfirmSetReview_CallsRepository_AndHidesPanel` — AC: inline review panel confirm.
  - `CancelSetReview_DoesNotCallRepository_AndHidesPanel` — AC: inline review panel cancel.
  - `MarkReviewed_CallsRepository_WithFalseFlag` — AC: review cleared.

- `tests/VideoIndexer.App.Tests/MainContentViewModelTests.cs` (extended — add to existing file to share `BuildSut()` helper)
  - `OpenMovieEditor_SetsCurrentPage_ToMovieEditor` — AC: navigation to editor.
  - `EditorCloseRequested_SetsCurrentPage_ToDefault` — AC: navigation back.
  - `EditorCloseRequested_CallsMoviesListLoadAsync` — AC: grid reloads after close.

- `tests/VideoIndexer.App.Tests/MoviePropertiesViewModelTests.cs`
  - `Properties_AreInitialisedFromMovie` — AC: Properties dialog shows correct data.
  - `FileSize_IsFormattedHumanReadable` — AC: file size display.

- `tests/VideoIndexer.App.Tests/LabelCleanerViewModelTests.cs`
  - `Reparse_IsTriggered_WhenUCWordsOptionChanges` — AC: Label Cleaner UCWords option.
  - `Reparse_IsTriggered_WhenStripKeywordsOptionChanges` — AC: keyword stripping option live update.
  - `ProposedFields_ArePopulated_OnConstruction` — AC: Clean Label dialog shows proposed data on open.

- `tests/VideoIndexer.Tests/LabelCleanerTests.cs`
  - `DetectDates_yyyymmddFormat` — AC: date detection.
  - `DetectDates_ddmmyyyyFormat` — AC: date detection.
  - `DetectDates_yymmddFormat` — AC: date detection.
  - `ActorDetection_MatchesKnownNames_CaseInsensitive` — AC: actor detection.
  - `ActorDetection_LongestMatchFirst` — AC: actor detection (longest-match safety).
  - `ActorDetection_EmptyKnownNames_ReturnsNoActors` — AC: graceful empty list.
  - `StudioDetection_MatchesKnownStudios` — AC: studio detection.
  - `UCWords_CapitalisesFirstLetter` — AC: UCWords option.
  - `KeywordStripping_RemovesListedTokens` — AC: keyword stripping.

- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperMovieRepositoryTests.cs` (integration; self-skip)
  - `GetByIdAsync_ReturnsMovie_ForExistingId`
  - `GetByIdAsync_ReturnsNull_ForMissingId`
  - `SaveAsync_PersistsAllFields`
  - `ResetViewCountAsync_SetsWatchCountToZero`
  - `SetReviewAsync_SetsReviewFlag`
  - `GetKnownActorNamesAsync_ReturnsDistinctNames`
  - `GetKnownStudiosAsync_ReturnsDistinctStudios`

---

## Documentation Updates

- `docs/agents/project-manifest/api-surface.md` — add `Movie` sealed record, `IMovieRepository`, `LabelCleanerResult`, `LabelCleanerOptions`, `INameParser`, `MovieEditorViewModel`, `LabelCleanerViewModel`, `MoviePropertiesViewModel`, `IFileLauncherService`; update `IAppPaths` with `MovieDataDirectory`; mark `FiltersManagerView` cosmetic fix as resolved.
- `docs/agents/project-manifest/file-tree.md` — add all new source files under their respective directories.
- `docs/agents/project-manifest/constraints.md` — update expected DB revision to 38; add m038 rollback procedure; add `IAppPaths.MovieDataDirectory` path convention (`[Root]/data/[hash]/`); add note that `MovieEditorViewModel.CloseRequested` must be unsubscribed to avoid memory leaks; correct `FilterExpressionEvaluatorTests` spec count from 57 to 56; remove the stale Open Item “MoviesListView — `LoadFilterSlotsAsync` not yet wired” (resolved in M5 Rework WP-001).
- `docs/agents/project-manifest/data-flows.md` — add section 9 documenting the Movie Edit Flow (list → event → load Movie → open editor → save/close → reload list).

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **m038 runs against a DB with year/season/episode already present** (legacy migration) | `ADD COLUMN IF NOT EXISTS` is a no-op if the column exists; safe for both fresh and migrated DBs |
| **NameParser complexity / accuracy** | The parser is explicitly scoped to the legacy algorithm (date regex, longest-match). "Good enough" output is the spec goal — the user reviews and accepts or discards; imperfect parses are not errors |
| **`Func<Movie, MovieEditorViewModel>` factory lifetime** | Factory lambda registered in `Program.cs` captures the `IServiceProvider`; transient deps resolved per call. Test by passing a `FakeMovieRepository` directly in the constructor |
| **`MainContentViewModel` memory leak if editor CloseRequested not unsubscribed** | `OpenMovieEditorAsync` stores the handler in a local and unsubscribes it in the close callback; `Dispose()` is updated to also call `CloseSubView()` if `CurrentPage == MovieEditor` |
| **Avalonia `DoubleTapped` routing on DataGrid rows** | Avalonia 11 `DoubleTapped` bubbles from the DataRow; the event handler in code-behind guards against null `SelectedMovie` before invoking the command |
| **`IFileLauncherService` is Windows-only in M6** | `WinExe` output type and Windows-only runtime is an established project constraint; macOS path is documented as out-of-scope in `constraints.md` |
