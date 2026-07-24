# Plan — M10: Player & Bookmarks

## Plan Audit Cycles
- Audits: 1 — Plan Auditor v1.3.1
- Architectural Reviews: 1 — Plan Architect Reviewer v1.4.0


## Summary

This plan addresses all actionable items from the M9 synthesis report
(`docs/agents/plans/2026-05-22-m9-images/synthesis.md`) and constitutes the full **M10 (Player &
Bookmarks)** milestone — the final planned milestone for the Video Indexer rebuild. It is structured
in two major thrusts: (1) **Phase 0** closes the six technical-debt items carried forward from M9
(`IDisposable` hygiene, a DI-lifetime fix, a model-level validation guard, a promoted test helper, a
constraint-documentation entry, and integration tests for `DapperThumbnailRepository`); and (2)
**Phases 1–9** deliver the M10 feature set: an embedded LibVLCSharp video player wired into the
Movie Editor Video tab; per-movie bookmark CRUD with ffmpeg-backed frame capture in a sidebar panel;
a paginated global Bookmarks Browser as a new primary-navigation destination; external VLC process
launch from the movies-list context menu; a `LibraryOptions.FfmpegPath` user override making the
ffmpeg binary resolution chain three-step; activation of four deferred Filter DSL bookmark
identifiers; a database migration (m042) that adds `thumbnail_path` to the already-existing
`movies_bookmarks` table; and manifest documentation updates across all four manifest documents.


## Architectural Context

### Foundation
- .NET 10 / Avalonia 11.3.14 / CommunityToolkit.Mvvm 8.3.2; strict layering: Core → Infrastructure → App.
- DI composition root is `src/VideoIndexer.App/Program.cs`; `App.Services` is the non-DI bridge for
  Avalonia paths.
- `MovieEditorViewModel` orchestrates all editor tab sub-VMs. From M9 it already holds
  `CoverImageVm`, `ThumbnailsVm`, and `GeneratorVm`. The Video tab is an empty stub.
- `MainContentViewModel` registers five primary navigation destinations in
  `IPrimaryNavigationService`; the sixth ("bookmarks") is added by this plan.

### Database
- MariaDB/MySQL, schema revision 41 post-M9.
- `movies_bookmarks`, `tags_bookmarks`, and `bookmarks_presets` **already exist** in the production
  schema (inherited from the legacy SPDB Indexer). Migration m042 only adds `thumbnail_path VARCHAR(512) NULL`
  to `movies_bookmarks` and bumps `ExpectedRevision` to 42.
- `DapperMovieCatalogRepository.MovieListSql` already JOINs `movies_bookmarks` and populates
  `MovieListItem.HasBookmarks`; this plan extends the query for `BookmarkCount` and
  `HasRatedBookmarks`.

### Bookmark-adjacent code (current state)
- `MovieListItem.HasBookmarks` (`required bool`) is populated. `BookmarkCount`, `HasRatedBookmarks`,
  and `BookmarkDescriptions` do not yet exist.
- DSL identifiers `HasRatedBookmarks()`, `BookmarkContains(text)`, and `AmountBookmarks` are
  recognised by `FilterExpressionParser` but immediately throw `FilterExpressionException` with
  "will be added in M10."
- `ITagsRepository.ConnectBookmarkTagAsync` / `DisconnectBookmarkTagAsync` are defined but unused.
  Bookmark tagging UI is **out of scope for M10** (table and interface methods are preserved as a
  future-milestone foundation).

### IAppPaths
- `IAppPaths` has four implementations that must all be updated when the interface grows:
  `AppPaths` (`src/VideoIndexer.Infrastructure/AppPaths.cs`), `FakeAppPaths`
  (`tests/VideoIndexer.Tests/Fixtures/FakeAppPaths.cs`), `FakeAppPaths`
  (`tests/VideoIndexer.App.Tests/TestHelpers/FakeAppPaths.cs`), and `LiveAppPaths` (private nested
  class in `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/FfmpegProvisionerLiveTests.cs`).

### IDisposable precedent
- `ThumbnailsViewModel : ObservableObject, IDisposable` is the reference implementation for VMs
  that hold Avalonia `Bitmap` handles. `MoviesListViewModel` and `MovieEditorViewModel` must follow
  the same pattern (M9 synthesis carry-forward).

### Dialog services precedent
- All dialog services (`IThumbnailViewerService`, `IThumbnailGenerationDialogService`, etc.) are
  defined as interfaces in `src/VideoIndexer.App/Services/`. This plan follows the same location
  for `IBookmarkSettingsService` and `IExternalPlayerService`.


## Approach / Architecture

### Phase 0 — M9 Carry-Forward Fixes
Six self-contained, non-feature commits that must land before any M10 feature code. They clean up
known technical debt without changing external behaviour.

### Phase 1 — Schema & Settings Foundation
Two groundwork items: m042 migration adds `thumbnail_path` to `movies_bookmarks` and bumps
`ExpectedRevision`; `LibraryOptions.FfmpegPath` adds the user-override field, updates
`FfmpegRunner.ResolveBinaryPath()` to a three-step chain, and exposes the value in the Settings UI.

### Phase 2 — Core Domain
New Core models (`Bookmark`, `BookmarkPreset`, `BookmarkListItem`, `BookmarkBrowserQuery`) and
interfaces (`IBookmarkRepository`, `IBookmarkBrowserRepository`, `IBookmarkPresetRepository`); an
`IAppPaths.BookmarkThumbnailPath` extension; `IMovieRepository.IncrementViewCountAsync`; and
`MovieListItem` extensions for DSL bookmark identifiers.

### Phase 3 — Infrastructure
Dapper repositories (`DapperBookmarkRepository`, `DapperBookmarkPresetRepository`), the
`DapperMovieCatalogRepository` SQL extension for `BookmarkCount`/`HasRatedBookmarks`/
`BookmarkDescriptions`, `DapperMovieRepository.IncrementViewCountAsync`, `FfmpegRunner` three-step
chain, and the LibVLCSharp NuGet package registration.

### Phase 4 — App Layer: Player
`PlayerViewModel` (owns `LibVLC` + `MediaPlayer`, implements `IDisposable`) and `PlayerView.axaml`
(LibVLCSharp `VideoView` + toolbar controls) wired into `MovieEditorView`'s Video tab. A
`ScreenshotPreviewViewModel` + `ScreenshotPreviewView` handles the frame-capture confirmation
dialog. View count is incremented via `IMovieRepository.IncrementViewCountAsync` on playback start.

### Phase 5 — App Layer: Per-Movie Bookmarks
`BookmarkItemViewModel`, `BookmarkSettingsViewModel` (create/edit dialog with frame-nudge via
`IFfmpegRunner.ExtractFrameAsync`, description autocomplete, presets, rating), and
`BookmarksPanelViewModel` (sidebar list with CRUD, click-to-seek). Panel is wired into
`MovieEditorView`'s left sidebar.

### Phase 6 — App Layer: Bookmarks Browser
`BookmarksBrowserViewModel` (paginated 40-per-page, search + rating filter, zoom, click-to-select
mode) and `BookmarksBrowserView` registered as a new primary-navigation destination ("bookmarks")
in `MainContentViewModel`.

### Phase 7 — External VLC Launch
`IExternalPlayerService` / `DesktopExternalPlayerService` using `ProcessStartInfo.ArgumentList`,
guarded by `UseVlc && !string.IsNullOrWhiteSpace(VlcExecutablePath)`. **Play** and **Play
Fullscreen** context-menu items added to `MoviesListView`.

### Phase 8 — Filter DSL Activation
Remove the "will be added in M10" guards from `FilterExpressionParser` for `HasRatedBookmarks()`,
`AmountBookmarks`, and `BookmarkContains(text)`. Implement the three evaluations in
`FilterExpressionEvaluator` using the new `MovieListItem` fields.

### Phase 9 — Manifest Documentation
Update all four manifest documents (`file-tree.md`, `api-surface.md`, `constraints.md`,
`data-flows.md`) to reflect the final M10 state.


## Rationale

- **Re-using `IFfmpegRunner.ExtractFrameAsync` for bookmark frames** — structurally identical to
  thumbnail extraction; avoids a new abstraction.
- **`BookmarkThumbnailPath` on `IAppPaths`** — consistent with `ThumbnailFilePath` and
  `CoverImagePath`; all path logic stays in one place.
- **Bookmarks Browser as a primary-nav destination** — the spec says "Main Menu → Bookmarks
  Favorites," which maps to the navigation rail exactly as existing destinations do.
- **`LibraryOptions.FfmpegPath` rather than `ExternalToolsOptions.*`** — follows the established
  `LibraryOptions.FfprobePath` user-override pattern (`src/VideoIndexer.Core/Options/LibraryOptions.cs`).
- **`IExternalPlayerService` in App layer** — consistent with `IFileLauncherService`
  (`src/VideoIndexer.App/Services/IFileLauncherService.cs`); no domain logic involved.
- **Bookmark tagging UI deferred** — `tags_bookmarks` table already exists; `ITagsRepository`
  bookmark methods are already defined; adding UI now would extend M10 beyond a reasonable scope
  without losing any data. A future plan can deliver the UI.
- **`movies_bookmarks` NOT recreated via migration** — the table already exists in production
  databases inherited from SPDB Indexer. The migration only adds the new `thumbnail_path` column.


## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|---|---|---|---|
| Bookmark thumbnail storage | `BookmarkThumbnailPath(hash, bookmarkId)` on `IAppPaths` | Inline path construction at call sites; reuse `ThumbnailFilePath` | Consistent with M9 pattern; `ThumbnailFilePath` has wrong extension/semantics |
| Bookmarks Browser surface | New primary-nav destination (rail) | Modal dialog from context menu; sub-page inside movies list | Spec says "Main Menu → Bookmarks Favorites" — maps naturally to a rail destination |
| FFmpeg path resolution order | User override (`Library.FfmpegPath`) → provisioner → PATH | Only provisioner + PATH (status quo) | User-override wins highest priority; matches existing `FfprobePath` pattern |
| LibVLCSharp vs. process-launched VLC | LibVLCSharp embedded (Video tab) + process-launched VLC (context menu) | Only process-launched; only LibVLCSharp | Spec explicitly requires both surfaces |
| `IntervalSeconds` guard location | Constructor `ArgumentOutOfRangeException` on `ThumbnailGenerationRequest` | Keep service-level clamp only | Model guard catches bad values before reaching the service layer |
| Bookmark tagging in M10 | Out-of-scope: table + interface methods only, no UI | Full UI in M10; drop the table | Deferred UI avoids feature creep; foundation preserved for a follow-up plan |
| `BookmarkDescriptions` DSL field | Batch-loaded alongside movies list (separate SQL round-trip) | `GROUP_CONCAT` in `MovieListSql`; defer `BookmarkContains` to server-side filtering | Separate round-trip matches the `BatchTagSql` pattern; `GROUP_CONCAT` has a length cap risk |


## Pattern Alignment

- `BookmarkItemViewModel` follows `ThumbnailItemViewModel` — lightweight per-item VM with immutable
  identity + observable state.
- `BookmarkSettingsViewModel` follows `CategoryEditorViewModel` — `CloseRequested<TResult>` event,
  modal dialog service.
- `BookmarksPanelViewModel` follows `ThumbnailsViewModel` (`src/VideoIndexer.App/ViewModels/ThumbnailsViewModel.cs`)
  — `IDisposable`, `CancellationTokenSource`, load-on-demand, owns `ObservableCollection<T>`.
- `BookmarksBrowserViewModel` follows `MoviesListViewModel` — paginated load, debounced search,
  filter model.
- `DapperBookmarkRepository` follows `DapperThumbnailRepository` — per-call connection open;
  stateless; registered as `AddSingleton`.
- `DesktopExternalPlayerService` follows `DesktopFileLauncherService`
  (`src/VideoIndexer.App/Services/DesktopFileLauncherService.cs`) — `ProcessStartInfo.ArgumentList`;
  `ILogger<T>`.
- **Departure**: `PlayerViewModel` receives `LibVLC` as a DI singleton (constructor-injected) and
  owns `MediaPlayer` (per-session) — no existing VM uses a native media handle. The VM must
  implement `IDisposable`; `Dispose()` stops and disposes `MediaPlayer` only — `LibVLC` lifetime is
  managed by the DI container and must not be disposed from `PlayerViewModel`. The VM must not hold
  Avalonia rendering objects; it exposes a `MediaPlayer` reference that `PlayerView` binds to a
  `VideoView` surface. This is the canonical LibVLCSharp + Avalonia pattern.


## Detailed Steps

### Phase 0 — M9 Technical Debt Cleanup

**Step 1 — `IDisposable` for `MoviesListViewModel` and `MovieEditorViewModel`**
- `MoviesListViewModel` (`src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs`):
  implement `IDisposable`; `Dispose()` cancels and disposes `_coverLoadCts` and
  `_dataChangedDebounce`; calls `DisposeCoverBitmap()`.
- `MovieEditorViewModel` (`src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs`):
  implement `IDisposable`; `Dispose()` calls the existing `DisposeImageViewModels()` and disposes
  `TaggerVm`. `DisposeImageViewModels()` must unsubscribe all event handlers from child VMs
  **before** disposing them (e.g., `BookmarksPanelVm.BookmarkSeekRequested -= OnBookmarkSeekRequested;`
  must precede `BookmarksPanelVm?.Dispose()`) to prevent memory leaks and double-seek on
  `LoadAsync` re-entry.
- `MoviesListView.axaml.cs` — `OnUnloaded` calls `(DataContext as MoviesListViewModel)?.Dispose()`.
- `MovieEditorView.axaml.cs` — `OnUnloaded` calls `(DataContext as MovieEditorViewModel)?.Dispose()`.
- Add a constraint rule to `constraints.md`: any VM that holds Avalonia `Bitmap` handles or native
  resources must implement `IDisposable`; the matching view's `OnUnloaded` handler must call
  `Dispose()`.

**Step 2 — `DapperThumbnailRepository` → `AddSingleton`**
- `src/VideoIndexer.App/Program.cs` line ~187: change
  `AddTransient<IThumbnailRepository, DapperThumbnailRepository>()` to
  `AddSingleton<IThumbnailRepository, DapperThumbnailRepository>()`.
- Add note to `constraints.md`: `DapperThumbnailRepository` opens a connection per call (stateless);
  registered `AddSingleton`.

**Step 3 — `IntervalSeconds` model-level guard**
- `src/VideoIndexer.Core/Models/ThumbnailGenerationRequest.cs`: override the `IntervalSeconds` positional property inside the record body to enforce the guard:
  ```csharp
  public int IntervalSeconds { get; init; } = IntervalSeconds > 0
      ? IntervalSeconds
      : throw new ArgumentOutOfRangeException(nameof(IntervalSeconds), IntervalSeconds, "IntervalSeconds must be > 0.");
  ```
- `src/VideoIndexer.App/ViewModels/ThumbnailGeneratorViewModel.cs`: ensure the `IntervalSeconds`
  setter clamps to `>= 1` before constructing a request (client-side defence; the model guard is
  the authoritative boundary).
- Keep the existing `if (intervalMs <= 0) { intervalMs = 30_000L; }` clamp in `FfmpegThumbnailGeneratorService` as a defensive fallback; add `Debug.Assert(false, "IntervalSeconds must already be > 0")` immediately before the `if` block to surface violations in Debug builds.
- Update `api-surface.md` `ThumbnailGenerationRequest` entry to document the guard.

**Step 4 — `SyncProgress<T>` promoted to shared test fixture**
- Create `tests/VideoIndexer.Tests/Fixtures/SyncProgress.cs`:
  `public sealed class SyncProgress<T>(Action<T> callback) : IProgress<T>` with
  `Report(T value) => callback(value)`.
- Remove the private inner class from
  `tests/VideoIndexer.Tests/FfmpegThumbnailGeneratorServiceTests.cs`; update all three usages to
  reference `Fixtures.SyncProgress<T>`.

**Step 5 — `FormattableString.Invariant` convention in `constraints.md`**
- Add rule under the **Testing** section of `docs/agents/project-manifest/constraints.md`:
  *Any test helper or assertion that formats `double` or `float` values into strings subsequently
  parsed back must use `FormattableString.Invariant(…)` or `.ToString("Fx", CultureInfo.InvariantCulture)`.
  Never use implicit format strings for floating-point values in tests — on non-English locales,
  the decimal separator becomes a comma, causing `InvariantCulture` parsing to misread the value
  by a factor of 1 000.*

**Step 6 — Integration tests for `DapperThumbnailRepository`**
- Create `tests/VideoIndexer.Infrastructure.Tests/Database/DapperThumbnailRepositoryTests.cs`.
- Apply `[Collection("LiveDB")]`, `IClassFixture<LiveDbFixture>`, and `[SkippableFact]` on each test method, following the pattern in `DapperTagsRepositoryTests.cs`.
- Test methods (each an `[SkippableFact]`):
  - `InsertAsync_ReturnsPositiveId_AndRoundTripsFields`
  - `GetByMovieIdAsync_ReturnsInsertedItems`
  - `GetCountByMovieIdAsync_MatchesInsertedCount`
  - `GetIdsByMovieIdAsync_MatchesInsertedIds`
  - `DeleteAsync_RemovesSpecificThumbnail`
  - `DeleteAllByMovieIdAsync_RemovesAllForMovie`

### Phase 1 — Schema & Settings Foundation

**Step 7 — Migration m042: add `thumbnail_path` to `movies_bookmarks`**
- Create `src/VideoIndexer.Infrastructure/Database/migrations/m042_add_bookmark_thumbnail_path.sql`:
  ```sql
  ALTER TABLE movies_bookmarks
      ADD COLUMN thumbnail_path VARCHAR(512) NULL DEFAULT NULL AFTER rating;
  UPDATE spdb_config SET config_value = '42' WHERE config_name = 'db_revision';
  ```
- Bump `DatabaseBootstrapper.ExpectedRevision` from `41` to `42` in
  `src/VideoIndexer.Infrastructure/Database/DatabaseBootstrapper.cs`.
- Update the `DatabaseBootstrapperTests.cs` sentinel literal that asserts `ExpectedRevision == 41`
  to `42`.
- Add m042 rollback procedure to `constraints.md`:
  `ALTER TABLE movies_bookmarks DROP COLUMN thumbnail_path; UPDATE spdb_config SET config_value = '41' WHERE config_name = 'db_revision';`
- Update `constraints.md` and `tech-stack.md` schema-revision references.

**Step 8 — `LibraryOptions.FfmpegPath` user override + three-step binary resolution**
- `src/VideoIndexer.Core/Options/LibraryOptions.cs`: add
  `public string? FfmpegPath { get; init; }` (parallels existing `FfprobePath`).
- `src/VideoIndexer.Infrastructure/Library/FfmpegRunner.cs` — update `ResolveBinaryPath()` to
  three-step chain:
  1. `settings.Library.FfmpegPath` (non-null, non-whitespace)
  2. `settings.ExternalTools.Ffmpeg.FfmpegPath` (provisioner path)
  3. `"ffmpeg"` (PATH fallback)
- `src/VideoIndexer.App/ViewModels/SettingsViewModel.cs`: add
  `[ObservableProperty] private string _ffmpegOverridePath = string.Empty;`; populate in
  `LoadFromOptions`; persist in `SaveCommand` via
  `options.Library with { FfmpegPath = string.IsNullOrWhiteSpace(...) ? null : ... }`.
- `src/VideoIndexer.App/Views/SettingsView.axaml`: add a labelled `TextBox` for the ffmpeg
  override path in the External Tools section.
- Add a `"FfmpegPath": null` default entry to the `Library` section of
  `src/VideoIndexer.App/Assets/appsettings.json` (if the file exists; otherwise the record default
  of `null` is sufficient).
- Update `api-surface.md` with the new `LibraryOptions.FfmpegPath` field.

### Phase 2 — Core Domain

**Step 9 — `Bookmark` model**
- Create `src/VideoIndexer.Core/Models/Bookmark.cs`:
  ```csharp
  public sealed record Bookmark(
      long     BookmarkId,
      long     MovieId,
      long     PositionMs,
      string   Description,
      int      Rating,          // 0 = unrated; 1–6 starred
      string?  ThumbnailPath,
      int      ClickCount,
      DateTime? LastClicked,
      DateTime CreatedAt);
  ```
  The legacy `favorite` column is intentionally not mapped.

**Step 10 — `BookmarkPreset` model + `IBookmarkPresetRepository`**
- Create `src/VideoIndexer.Core/Models/BookmarkPreset.cs`:
  `public sealed record BookmarkPreset(long PresetId, string Label);`
- Create `src/VideoIndexer.Core/Abstractions/IBookmarkPresetRepository.cs`:
  ```csharp
  public interface IBookmarkPresetRepository
  {
      Task<IReadOnlyList<BookmarkPreset>> GetAllAsync(CancellationToken ct = default);
      Task<BookmarkPreset>               InsertAsync(string label, CancellationToken ct = default);
      Task                               DeleteAsync(long presetId, CancellationToken ct = default);
  }
  ```

**Step 11 — `IBookmarkRepository`**
- Create `src/VideoIndexer.Core/Abstractions/IBookmarkRepository.cs`:
  ```csharp
  public interface IBookmarkRepository
  {
      Task<IReadOnlyList<Bookmark>> GetByMovieIdAsync(long movieId, CancellationToken ct = default);
      Task<Bookmark>                InsertAsync(Bookmark bookmark, CancellationToken ct = default);
      Task                          UpdateAsync(Bookmark bookmark, CancellationToken ct = default);  // description + rating only
      Task                          DeleteAsync(long bookmarkId, CancellationToken ct = default);
      Task                          IncrementClickAsync(long bookmarkId, CancellationToken ct = default);
      Task<IReadOnlyList<string>>   GetAllDescriptionsAsync(CancellationToken ct = default);
  }
  ```

**Step 12 — `IBookmarkBrowserRepository`, `BookmarkListItem`, `BookmarkBrowserQuery`**
- Create `src/VideoIndexer.Core/Models/BookmarkListItem.cs`:
  `public sealed record BookmarkListItem(long BookmarkId, long MovieId, string MovieLabel, long PositionMs, string Description, int Rating, string? ThumbnailPath, int ClickCount);`
- Create `src/VideoIndexer.Core/Models/BookmarkBrowserQuery.cs`:
  `public sealed record BookmarkBrowserQuery(string? SearchText = null, int? MinRating = null);`
- Create `src/VideoIndexer.Core/Abstractions/IBookmarkBrowserRepository.cs`:
  ```csharp
  public interface IBookmarkBrowserRepository
  {
      Task<int>                          GetCountAsync(BookmarkBrowserQuery query, CancellationToken ct = default);
      Task<IReadOnlyList<BookmarkListItem>> GetPageAsync(BookmarkBrowserQuery query, int page, int pageSize, CancellationToken ct = default);
  }
  ```

**Step 13 — `IAppPaths.BookmarkThumbnailPath`**
- Add to `IAppPaths` (`src/VideoIndexer.Core/Abstractions/IAppPaths.cs`):
  `string BookmarkThumbnailPath(string hash, long bookmarkId);`
- Implement in `AppPaths`:
  `Path.Combine(MovieDataDirectory(hash), $"bk_{bookmarkId}.jpg")`
- Implement the same formula in all three fake/live implementations:
  `tests/VideoIndexer.Tests/Fixtures/FakeAppPaths.cs`,
  `tests/VideoIndexer.App.Tests/TestHelpers/FakeAppPaths.cs`, and
  `LiveAppPaths` (nested class in `FfmpegProvisionerLiveTests.cs`).

**Step 14 — `IMovieRepository.IncrementViewCountAsync`**
- Add to `IMovieRepository` (`src/VideoIndexer.Core/Abstractions/IMovieRepository.cs`):
  `Task IncrementViewCountAsync(long movieId, CancellationToken cancellationToken = default);`
- Add a no-op `IncrementViewCountAsync` implementation to `FakeMovieRepository` (`tests/VideoIndexer.App.Tests/TestHelpers/FakeMovieRepository.cs`). Note: `InMemoryMovieCatalogRepository` implements `IMovieCatalogRepository` (not `IMovieRepository`) and requires no changes for this step.

**Step 15 — `MovieListItem` extensions for bookmark DSL identifiers**
- Add to `src/VideoIndexer.Core/Models/MovieListItem.cs`:
  ```csharp
  public int                    BookmarkCount        { get; init; }
  public bool                   HasRatedBookmarks    { get; init; }
  public IReadOnlyList<string>  BookmarkDescriptions { get; init; } = [];
  ```

### Phase 3 — Infrastructure

**Step 16 — `DapperBookmarkRepository`**
- Create `src/VideoIndexer.Infrastructure/Database/DapperBookmarkRepository.cs`.
- Implements both `IBookmarkRepository` and `IBookmarkBrowserRepository`.
- Maps `movies_bookmarks` columns using a private `BookmarkRow` class (following the
  `DapperThumbnailRepository` pattern).
- `InsertAsync` uses `INSERT INTO movies_bookmarks (movie_id, milliseconds, description, rating, thumbnail_path, added) VALUES (...); SELECT LAST_INSERT_ID();`
- `UpdateAsync` updates `description` and `rating` only (timestamp is immutable per spec).
- `IncrementClickAsync`:
  `UPDATE movies_bookmarks SET click_count = click_count + 1, last_clicked = NOW() WHERE bookmark_id = @Id`
- `GetAllDescriptionsAsync`:
  `SELECT DISTINCT description FROM movies_bookmarks ORDER BY description`
- `GetCountAsync` / `GetPageAsync`: parameterised `WHERE` clause from `BookmarkBrowserQuery`
  joined to `movies` for `MovieLabel`; ordered by `m.label ASC, mb.milliseconds ASC`; paginated
  with `LIMIT @PageSize OFFSET @Offset`.
- Register in `Program.cs`:
  ```csharp
  builder.Services.AddSingleton<DapperBookmarkRepository>();
  builder.Services.AddSingleton<IBookmarkRepository>(sp => sp.GetRequiredService<DapperBookmarkRepository>());
  builder.Services.AddSingleton<IBookmarkBrowserRepository>(sp => sp.GetRequiredService<DapperBookmarkRepository>());
  ```

**Step 17 — `DapperBookmarkPresetRepository`**
- Create `src/VideoIndexer.Infrastructure/Database/DapperBookmarkPresetRepository.cs`.
- Implements `IBookmarkPresetRepository`; queries `bookmarks_presets` table.
- Register `AddSingleton<IBookmarkPresetRepository, DapperBookmarkPresetRepository>()` in
  `Program.cs`.

**Step 18 — `DapperMovieCatalogRepository` SQL extension**
- Extend `MovieListSql` to include:
  ```sql
  COUNT(DISTINCT mb.bookmark_id)                               AS BookmarkCount,
  CASE WHEN MAX(CASE WHEN mb.rating > 0 THEN 1 ELSE 0 END) = 1 THEN 1 ELSE 0 END AS HasRatedBookmarks,
  ```
  with `LEFT JOIN movies_bookmarks mb ON mb.movie_id = m.movie_id` (rename the existing `bk`
  subquery join to use `mb` directly, consolidating the two `movies_bookmarks` references into
  one).
- Add a `BatchBookmarkDescriptionSql` round-trip (parallel to `BatchTagSql`):
  ```sql
  SELECT movie_id AS MovieId, description AS Description
  FROM   movies_bookmarks
  ORDER  BY movie_id, milliseconds
  ```
- In `GetMovieListAsync`, after the existing tag-enrichment pass, add a bookmark-descriptions
  enrichment pass that groups the query results by `MovieId` and assigns
  `BookmarkDescriptions = [...descriptions]` on each `MovieListItem`.
- Update the Dapper row class (or anon projection) to include `BookmarkCount` and
  `HasRatedBookmarks`.

**Step 19 — `DapperMovieRepository.IncrementViewCountAsync`**
- Add to `src/VideoIndexer.Infrastructure/Library/DapperMovieRepository.cs`:
  ```csharp
  public async Task IncrementViewCountAsync(long movieId, CancellationToken cancellationToken = default)
  {
      using var conn = await _connectionFactory.CreateOpenConnectionAsync(cancellationToken);
      await conn.ExecuteAsync("UPDATE movies SET watch_count = watch_count + 1 WHERE movie_id = @MovieId", new { MovieId = movieId });
  }
  ```

**Step 20 — LibVLCSharp NuGet package registration**
- Research and select the latest stable `LibVLCSharp` and `LibVLCSharp.Avalonia` versions
  compatible with Avalonia 11.3.x and .NET 10.
- Add version entries to `Directory.Packages.props`.
- Add `<PackageReference Include="LibVLCSharp" />`,
  `<PackageReference Include="LibVLCSharp.Avalonia" />`, and
  `<PackageReference Include="VideoLAN.LibVLC.Windows" />` to
  `src/VideoIndexer.App/VideoIndexer.App.csproj` only (App layer; Core must have zero external
  NuGet deps).
- Register `LibVLC` as a DI singleton in `Program.cs`:
  ```csharp
  builder.Services.AddSingleton<LibVLC>();
  ```
- Add a note to `constraints.md` documenting the LibVLCSharp lifetime contract: `LibVLC` is a DI
  singleton (one per application lifetime); only `MediaPlayer` is per-session. `PlayerViewModel`
  must dispose `MediaPlayer` in `Dispose()` but must **not** dispose `LibVLC` (the DI container
  owns that lifetime). Never let `MediaPlayer` be garbage-collected.

### Phase 4 — App Layer: Player

**Step 21 — `PlayerViewModel`**
- Create `src/VideoIndexer.App/ViewModels/PlayerViewModel.cs`.
- Implements `IDisposable`; receives `LibVLC _libVlc` via constructor injection (DI singleton);
  owns `MediaPlayer _mediaPlayer` (per-session resource, created as `new MediaPlayer(_libVlc)` in
  the constructor); also receives `IThumbnailRepository _thumbnailRepository`.
- `[ObservableProperty]`: `bool IsPlaying`, `long PositionMs`, `long DurationMs`, `bool IsMuted`,
  `float Volume`.
- `MediaPlayer _mediaPlayer` is exposed as a read-only property for `PlayerView` binding.
- `LoadVideoCommand(string videoPath)` — creates a `Media` from the path and calls
  `_mediaPlayer.Play(media)`.
- `PlayPauseCommand` — toggles `_mediaPlayer.Pause()` / `_mediaPlayer.Play()`.
- `SeekCommand(long positionMs)` — sets `_mediaPlayer.Time = positionMs` (clamped to `[0, DurationMs]`).
- `SeekRelativeCommand(long deltaMs)` — adds delta to current position, clamps, calls `SeekCommand`.
- `ScreenshotCommand` — extracts the current frame via `IFfmpegRunner.ExtractFrameAsync` at
  `PositionMs`, then invokes `IScreenshotPreviewService.ShowAsync(...)`. If `ShowAsync` returns
  `true` and `viewModel.AddToThumbnails` is set, calls
  `_thumbnailRepository.InsertAsync(...)` to persist the frame as a thumbnail.
- `MuteCommand` — toggles `_mediaPlayer.Mute`.
- On `_mediaPlayer.Playing` event (first play only): calls
  `IMovieRepository.IncrementViewCountAsync(MovieId)` fire-and-forget with a try/catch + log.
- `Dispose()` calls `_mediaPlayer.Stop(); _mediaPlayer.Dispose();` — does **not** dispose
  `_libVlc` (the DI container owns that singleton lifetime).
- Register as `AddTransient<PlayerViewModel>` in `Program.cs` (short-lived, one per editor
  session; owned and disposed by `MovieEditorViewModel`; `LibVLC` singleton is injected by DI).

**Step 22 — `PlayerView.axaml`**
- Create `src/VideoIndexer.App/Views/PlayerView.axaml` + `PlayerView.axaml.cs`.
- Root: `UserControl`. Contains a `LibVLCSharp.Avalonia.VideoView` bound to
  `{Binding MediaPlayer}`.
- Toolbar (below video): Play/Pause `ToggleButton`, progress `Slider` (`Value` ↔ `PositionMs`,
  `Maximum = DurationMs`), time display `TextBlock`
  (`{Binding PositionMs, Converter=...}` → `hh:mm:ss.fff`), mute `ToggleButton`, volume `Slider`,
  screenshot `Button`.
- Precision seek row: two `Slider` controls (back / forward) labelled with their configured range;
  `Value` changes fire `SeekRelativeCommand` with negative / positive delta.
- Manual seek `TextBox` bound to a `SeekInputText` observable string; `Enter` key binding calls
  `SeekCommand` after parsing.
- NOT registered in `ViewLocator` (embedded in `MovieEditorView`; not a standalone nav target).

**Step 23 — `IScreenshotPreviewService`, `ScreenshotPreviewViewModel`, `ScreenshotPreviewView`**
- Create `src/VideoIndexer.App/Services/IScreenshotPreviewService.cs`:
  ```csharp
  public interface IScreenshotPreviewService
  {
      Task<bool> ShowAsync(ScreenshotPreviewViewModel viewModel, CancellationToken ct = default);
  }
  ```
- Create `src/VideoIndexer.App/ViewModels/ScreenshotPreviewViewModel.cs`:
  - `[ObservableProperty]`: `Bitmap? PreviewBitmap`, `bool IsBusy`, `long PositionMs`, `bool AddToThumbnails`.
  - Commands: `StepForwardCommand(int multiplier)`, `StepBackwardCommand(int multiplier)`,
    `AcceptCommand`, `CancelCommand`.
  - Step commands re-invoke `IFfmpegRunner.ExtractFrameAsync` at ± (multiplier × 1 000 ms),
    updating `PositionMs` and `PreviewBitmap`.
  - `AcceptCommand`: fires `CloseRequested(true)` only — thumbnail persistence is handled by the
    caller (`PlayerViewModel.ScreenshotCommand`); `ScreenshotPreviewViewModel` has no dependency on
    `IThumbnailRepository`.
  - `CancelCommand`: deletes the temp frame file; fires `CloseRequested(false)`.
  - Implements `IDisposable` (disposes `PreviewBitmap`).
- Create `src/VideoIndexer.App/Views/ScreenshotPreviewView.axaml` + `.cs`.
- Create `src/VideoIndexer.App/Services/AvaloniaScreenshotPreviewService.cs` implementing
  `IScreenshotPreviewService` (modal window).
- Register `AddSingleton<IScreenshotPreviewService, AvaloniaScreenshotPreviewService>()` in
  `Program.cs`.

**Step 24 — Wire Video tab in `MovieEditorView`**
- `MovieEditorViewModel` adds optional `Func<PlayerViewModel>? _playerFactory` constructor
  parameter (DI-injected).
- `LoadAsync` creates `PlayerVm = _playerFactory?.Invoke()` and calls `PlayerVm.LoadVideoCommand`
  with the resolved file path (obtained from `IMovieRepository` or the existing `_currentMovie`).
- `DisposeImageViewModels()` extended to also call `PlayerVm?.Dispose(); PlayerVm = null;`.
- `MovieEditorView.axaml` Video tab content: replace the stub with
  `<views:PlayerView DataContext="{Binding PlayerVm}" />` (null-safe via `IsVisible` binding).
- `Program.cs`: register `Func<PlayerViewModel>` factory delegate.

### Phase 5 — App Layer: Per-Movie Bookmarks

**Step 25 — `BookmarkItemViewModel`**
- Create `src/VideoIndexer.App/ViewModels/BookmarkItemViewModel.cs`.
- Immutable: `long BookmarkId`, `long PositionMs`.
- `[ObservableProperty]`: `string Description`, `int Rating`, `Bitmap? ThumbnailBitmap`.
- `string PositionLabel` computed: `TimeSpan.FromMilliseconds(PositionMs).ToString(@"hh\:mm\:ss")`.
- Implements `IDisposable` (disposes `ThumbnailBitmap`).

**Step 26 — `IBookmarkSettingsService`, `BookmarkSettingsViewModel`, `BookmarkSettingsView`**
- Create `src/VideoIndexer.App/Services/IBookmarkSettingsService.cs`:
  ```csharp
  public enum BookmarkEditMode { Create, Edit }
  public interface IBookmarkSettingsService
  {
      Task<Bookmark?> ShowAsync(BookmarkEditMode mode, long movieId, string hash,
                                long positionMs, Bookmark? existing = null,
                                CancellationToken ct = default);
  }
  ```
- Create `src/VideoIndexer.App/ViewModels/BookmarkSettingsViewModel.cs`:
  - Create mode: pre-filled `PositionMs` (read-only display); `Description` (autocomplete +
    presets dropdown via `IBookmarkPresetRepository`); `Rating` (0–6 selector); frame-nudge
    controls (`1×`, `2×`, `4×` step forward/back via `IFfmpegRunner.ExtractFrameAsync`); "Add to
    Thumbnails" checkbox.
  - Edit mode: `PositionMs` display only; `Description` + `Rating` editable; no frame-nudge.
  - `CloseRequested<Bookmark?>` — `null` = cancel.
- Create `src/VideoIndexer.App/Views/BookmarkSettingsView.axaml` + `.cs`.
- Create `src/VideoIndexer.App/Services/AvaloniaBookmarkSettingsService.cs` implementing
  `IBookmarkSettingsService`.
- Register `AddSingleton<IBookmarkSettingsService, AvaloniaBookmarkSettingsService>()` in
  `Program.cs`.

**Step 27 — `BookmarksPanelViewModel` + `BookmarksPanelView`**
- Create `src/VideoIndexer.App/ViewModels/BookmarksPanelViewModel.cs` (implements `IDisposable`).
- Constructor: `IBookmarkRepository bookmarkRepository`, `IBookmarkSettingsService settingsService`,
  `long movieId`, `string hash`.
- `ObservableCollection<BookmarkItemViewModel> Bookmarks`.
- `LoadAsync(CancellationToken)` — loads `GetByMovieIdAsync`; populates `Bookmarks`; loads
  thumbnail bitmaps for items that have a `ThumbnailPath`.
- `AddBookmarkCommand(long positionMs)` — invokes `settingsService.ShowAsync(Create, ...)`; on
  non-null result calls `bookmarkRepository.InsertAsync` and inserts a new
  `BookmarkItemViewModel` into `Bookmarks`.
- `EditBookmarkCommand(BookmarkItemViewModel)` — invokes `settingsService.ShowAsync(Edit, ...)`; on
  non-null result calls `bookmarkRepository.UpdateAsync` and updates the VM properties.
- `DeleteBookmarkCommand(BookmarkItemViewModel)` — calls `bookmarkRepository.DeleteAsync`; removes
  the item; deletes the thumbnail file if `ThumbnailPath != null`.
- `BookmarkClickedCommand(BookmarkItemViewModel)` — calls `bookmarkRepository.IncrementClickAsync`;
  raises a `BookmarkSeekRequested` event carrying `PositionMs` (subscribed by
  `MovieEditorViewModel` to forward to `PlayerVm.SeekCommand`).
- Create `src/VideoIndexer.App/Views/BookmarksPanelView.axaml` + `.cs` — list ordered by
  `PositionMs` ascending; per-item context menu: Edit / Delete; **Add Bookmark** button at top.

**Step 28 — Wire bookmarks panel into `MovieEditorView`**
- `MovieEditorViewModel` adds optional constructor parameters:
  `IBookmarkRepository? bookmarkRepository`, `IBookmarkSettingsService? bookmarkSettingsService`.
- `LoadAsync` creates `BookmarksPanelVm = new BookmarksPanelViewModel(...)` when both services are
  present; subscribes to `BookmarkSeekRequested` and forwards to `PlayerVm?.SeekCommand`.
- `DisposeImageViewModels()` extended to unsubscribe `BookmarkSeekRequested` and then dispose
  `BookmarksPanelVm`: `BookmarksPanelVm.BookmarkSeekRequested -= OnBookmarkSeekRequested;` must
  precede `BookmarksPanelVm?.Dispose(); BookmarksPanelVm = null;` — this prevents memory leaks
  and double-seek on `LoadAsync` re-entry.
- `MovieEditorView.axaml` left sidebar: add `<views:BookmarksPanelView>` below the review section.
- `Program.cs` passes both services into the `MovieEditorViewModel` factory lambda.

### Phase 6 — App Layer: Bookmarks Browser

**Step 29 — `BookmarksBrowserViewModel` + `BookmarksBrowserView`**
- Create `src/VideoIndexer.App/ViewModels/BookmarksBrowserViewModel.cs`.
- `[ObservableProperty]`: `string SearchText`, `int? MinRating`, `int CurrentPage`, `int TotalPages`,
  `int ZoomLevel`, `bool IsClickToSelectMode`.
- `ObservableCollection<BookmarkListItem> Items`.
- Commands: `LoadPageCommand(int page)`, `SearchChangedCommand` (debounced 300 ms),
  `ClearFilterCommand`, `DeleteBookmarkCommand(BookmarkListItem)`,
  `EditBookmarkCommand(BookmarkListItem)`, `PlayBookmarkCommand(BookmarkListItem)`,
  `EditMovieCommand(BookmarkListItem)`.
- Page size fixed at 40.
- `ZoomLevel` persisted via `AppOptions` — add `BookmarkBrowserOptions` nested record to
  `src/VideoIndexer.Core/Options/AppOptions.cs` with `int ZoomLevel { get; init; } = 3`.
- `PlayBookmarkCommand` — if external VLC is configured, launches via `IExternalPlayerService`;
  otherwise navigates to the Movie Editor for the bookmark's movie (no auto-seek; seek is left to
  the user).
- `EditMovieCommand` — raises a `NavigateToMovieRequested(long movieId)` event subscribed by
  `MainContentViewModel`.
- Create `src/VideoIndexer.App/Views/BookmarksBrowserView.axaml` + `.cs` — `ItemsControl` with
  zoom-responsive item template; pagination controls; filter toolbar at top.

**Step 30 — Register Bookmarks Browser as a primary-nav destination**
- `MainContentViewModel` full constructor adds `BookmarksBrowserViewModel bookmarksBrowserVm`
  parameter.
- Add `new PrimaryDestination("bookmarks", "Bookmarks", bookmarksBrowserVm)` to the six-item
  `primaryNavService.Register([...])` array.
- `MainContentView.axaml` navigation rail adds a Bookmarks icon item (sixth position).
- Register `AddSingleton<BookmarksBrowserViewModel>` in `Program.cs`.

### Phase 7 — External VLC Launch

**Step 31 — `IExternalPlayerService` + `DesktopExternalPlayerService`**
- Create `src/VideoIndexer.App/Services/IExternalPlayerService.cs`:
  ```csharp
  public interface IExternalPlayerService
  {
      bool CanUseExternalPlayer { get; }
      void Play(string filePath);
      void PlayFullscreen(string filePath);
  }
  ```
- Create `src/VideoIndexer.App/Services/DesktopExternalPlayerService.cs` implementing the
  interface; reads `ExternalToolsOptions.VlcExecutablePath` and `UseVlc` from `ISettingsService`.
- `CanUseExternalPlayer` — `UseVlc && !string.IsNullOrWhiteSpace(VlcExecutablePath)`.
- `Play(filePath)` — `ProcessStartInfo { FileName = VlcPath, ArgumentList = { filePath } }`.
- `PlayFullscreen(filePath)` — `ArgumentList = { "--fullscreen", filePath }`.
- Uses `ILogger<DesktopExternalPlayerService>` for error logging.
- Register `AddSingleton<IExternalPlayerService, DesktopExternalPlayerService>()` in `Program.cs`.

**Step 32 — External VLC context menu in `MoviesListView`**
- `MoviesListViewModel` adds optional `IExternalPlayerService? _externalPlayerService` constructor
  parameter.
- `PlayExternalCommand` — `[RelayCommand(CanExecute = nameof(CanPlayExternal))]`; guard:
  `SelectedMovies.Count == 1 && _externalPlayerService?.CanUseExternalPlayer == true`.
- `PlayFullscreenExternalCommand` — same guard.
- `MoviesListView.axaml` context menu: add separator + **Play** / **Play Fullscreen** items;
  `IsVisible` bound to `ExternalPlayerAvailable` observable property.
- `Program.cs` passes `IExternalPlayerService` into `MoviesListViewModel` factory.

### Phase 8 — Filter DSL Activation

**Step 33 — Activate bookmark DSL identifiers**
- `src/VideoIndexer.Core/Filtering/FilterExpressionParser.cs`:
  - Remove the deferred-guard entries for `HasRatedBookmarks`, `AmountBookmarks`, and
    `BookmarkContains`. (`HasBookmarks()` is already active.)
- `src/VideoIndexer.Core/Filtering/FilterExpressionEvaluator.cs`:
  - `HasRatedBookmarks()` → `item.HasRatedBookmarks`
  - `AmountBookmarks` (property identifier) → `item.BookmarkCount`
  - `BookmarkContains(text)` → `item.BookmarkDescriptions.Any(d => d.Contains(args[0], StringComparison.OrdinalIgnoreCase))`
- Update `tests/VideoIndexer.Tests/Filtering/FilterExpressionParserTests.cs` and
  `FilterExpressionEvaluatorTests.cs` with positive and negative test cases for each newly
  activated identifier.

### Phase 9 — Manifest Documentation

**Step 34 — Update all four manifest documents**
- `docs/agents/project-manifest/file-tree.md` — add every new file from Phases 0–8 with accurate
  annotations.
- `docs/agents/project-manifest/api-surface.md` — add/update entries for:
  `IBookmarkRepository`, `IBookmarkBrowserRepository`, `IBookmarkPresetRepository`,
  `IBookmarkSettingsService`, `IScreenshotPreviewService`, `IExternalPlayerService`,
  `Bookmark`, `BookmarkPreset`, `BookmarkListItem`, `BookmarkBrowserQuery`,
  `DapperBookmarkRepository`, `DapperBookmarkPresetRepository`,
  `PlayerViewModel`, `ScreenshotPreviewViewModel`, `BookmarkItemViewModel`,
  `BookmarkSettingsViewModel`, `BookmarksPanelViewModel`, `BookmarksBrowserViewModel`,
  `IAppPaths.BookmarkThumbnailPath`, `IMovieRepository.IncrementViewCountAsync`,
  `LibraryOptions.FfmpegPath`, `MovieListItem.BookmarkCount/HasRatedBookmarks/BookmarkDescriptions`,
  `ThumbnailGenerationRequest` (guard note), `AppOptions.BookmarkBrowserOptions`.
- `docs/agents/project-manifest/constraints.md` — add rules for:
  IDisposable obligation for Bitmap-holding VMs; `DapperThumbnailRepository` singleton
  registration rationale; LibVLCSharp disposal contract; FfmpegPath 3-step chain; BookmarkThumbnailPath
  convention; m042 rollback procedure; FormattableString.Invariant test rule.
- `docs/agents/project-manifest/data-flows.md` — add:
  Player startup and view-count increment flow; Bookmark creation flow (player position →
  `IFfmpegRunner.ExtractFrameAsync` → BookmarkSettings dialog → `InsertAsync`); Bookmarks Browser
  load and paginate flow.
- `docs/agents/project-manifest/tech-stack.md` — update schema revision from 41 to 42.


## Dependencies

- **Phase 0** — no dependencies; can start immediately.
- **Phase 1** (Schema + Settings) — no code dependencies; should land before integration tests run.
- **Phase 2** (Core) — no dependencies; can start after or alongside Phase 0.
- **Phase 3** (Infrastructure) — requires Phase 2 (implements Core interfaces).
- **Phase 4** (Player) — requires Phase 3 (LibVLCSharp packages; `IMovieRepository`).
- **Phase 5** (Bookmarks per-movie) — requires Phase 3 (repositories); can run in parallel with
  Phase 4.
- **Phase 6** (Browser) — requires Phase 5 (uses `BookmarkListItem`; `IBookmarkBrowserRepository`).
- **Phase 7** (External VLC) — no dependency on Phases 4–6; can run in parallel with them.
- **Phase 8** (DSL) — requires Phase 3 (`MovieListItem` extensions; enriched catalog query).
- **Phase 9** (Docs) — must run last.


## Required Components

### New files
```
tests/VideoIndexer.Tests/Fixtures/SyncProgress.cs
tests/VideoIndexer.Infrastructure.Tests/Database/DapperThumbnailRepositoryTests.cs
tests/VideoIndexer.Infrastructure.Tests/Database/DapperBookmarkRepositoryTests.cs
src/VideoIndexer.Infrastructure/Database/migrations/m042_add_bookmark_thumbnail_path.sql
src/VideoIndexer.Core/Models/Bookmark.cs
src/VideoIndexer.Core/Models/BookmarkPreset.cs
src/VideoIndexer.Core/Models/BookmarkListItem.cs
src/VideoIndexer.Core/Models/BookmarkBrowserQuery.cs
src/VideoIndexer.Core/Abstractions/IBookmarkRepository.cs
src/VideoIndexer.Core/Abstractions/IBookmarkPresetRepository.cs
src/VideoIndexer.Core/Abstractions/IBookmarkBrowserRepository.cs
src/VideoIndexer.Infrastructure/Database/DapperBookmarkRepository.cs
src/VideoIndexer.Infrastructure/Database/DapperBookmarkPresetRepository.cs
src/VideoIndexer.App/ViewModels/PlayerViewModel.cs
src/VideoIndexer.App/ViewModels/ScreenshotPreviewViewModel.cs
src/VideoIndexer.App/ViewModels/BookmarkItemViewModel.cs
src/VideoIndexer.App/ViewModels/BookmarkSettingsViewModel.cs
src/VideoIndexer.App/ViewModels/BookmarksPanelViewModel.cs
src/VideoIndexer.App/ViewModels/BookmarksBrowserViewModel.cs
src/VideoIndexer.App/Services/IBookmarkSettingsService.cs
src/VideoIndexer.App/Services/AvaloniaBookmarkSettingsService.cs
src/VideoIndexer.App/Services/IScreenshotPreviewService.cs
src/VideoIndexer.App/Services/AvaloniaScreenshotPreviewService.cs
src/VideoIndexer.App/Services/IExternalPlayerService.cs
src/VideoIndexer.App/Services/DesktopExternalPlayerService.cs
src/VideoIndexer.App/Views/PlayerView.axaml  (+.cs)
src/VideoIndexer.App/Views/ScreenshotPreviewView.axaml  (+.cs)
src/VideoIndexer.App/Views/BookmarkSettingsView.axaml  (+.cs)
src/VideoIndexer.App/Views/BookmarksPanelView.axaml  (+.cs)
src/VideoIndexer.App/Views/BookmarksBrowserView.axaml  (+.cs)
tests/VideoIndexer.App.Tests/TestHelpers/FakeBookmarkRepository.cs
tests/VideoIndexer.App.Tests/TestHelpers/FakeBookmarkBrowserRepository.cs
```

### Significantly modified files
```
src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs          IDisposable; PlayExternalCommand
src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs         IDisposable; PlayerVm; BookmarksPanelVm
src/VideoIndexer.App/ViewModels/SettingsViewModel.cs            FfmpegOverridePath
src/VideoIndexer.App/ViewModels/MainContentViewModel.cs         BookmarksBrowserVm; sixth destination
src/VideoIndexer.App/Views/MoviesListView.axaml(.cs)            Play/PlayFullscreen context menu
src/VideoIndexer.App/Views/MovieEditorView.axaml(.cs)           Video tab; Bookmarks panel; OnUnloaded dispose
src/VideoIndexer.App/Views/SettingsView.axaml                   FfmpegPath field
src/VideoIndexer.App/Views/MainContentView.axaml(.cs)           Bookmarks rail item
src/VideoIndexer.App/Program.cs                                 All new DI registrations
src/VideoIndexer.Core/Abstractions/IAppPaths.cs                 BookmarkThumbnailPath
src/VideoIndexer.Core/Abstractions/IMovieRepository.cs          IncrementViewCountAsync
src/VideoIndexer.Core/Models/MovieListItem.cs                   BookmarkCount; HasRatedBookmarks; BookmarkDescriptions
src/VideoIndexer.Core/Models/ThumbnailGenerationRequest.cs      IntervalSeconds guard
src/VideoIndexer.Core/Options/LibraryOptions.cs                 FfmpegPath
src/VideoIndexer.Core/Options/AppOptions.cs                     BookmarkBrowserOptions
src/VideoIndexer.Core/Filtering/FilterExpressionParser.cs       Activate bookmark identifiers
src/VideoIndexer.Core/Filtering/FilterExpressionEvaluator.cs    Bookmark evaluations
src/VideoIndexer.Infrastructure/AppPaths.cs                     BookmarkThumbnailPath
src/VideoIndexer.Infrastructure/Database/DatabaseBootstrapper.cs  ExpectedRevision 42
src/VideoIndexer.Infrastructure/Library/DapperMovieRepository.cs  IncrementViewCountAsync
src/VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs  BookmarkCount; HasRatedBookmarks; BookmarkDescriptions
src/VideoIndexer.Infrastructure/Library/FfmpegRunner.cs         3-step resolution chain
tests/VideoIndexer.Tests/Fixtures/FakeAppPaths.cs               BookmarkThumbnailPath
tests/VideoIndexer.App.Tests/TestHelpers/FakeAppPaths.cs        BookmarkThumbnailPath
tests/VideoIndexer.Infrastructure.Tests/ExternalTools/FfmpegProvisionerLiveTests.cs  LiveAppPaths.BookmarkThumbnailPath
tests/VideoIndexer.Tests/FfmpegThumbnailGeneratorServiceTests.cs  Use promoted SyncProgress<T>
Directory.Packages.props                                         LibVLCSharp versions
```


## Assumptions

- `movies_bookmarks`, `tags_bookmarks`, and `bookmarks_presets` already exist in all production
  databases (confirmed from `spdb-indexer/SPDB Indexer/sql/structure.sql`). Migration m042 adds
  only the `thumbnail_path` column.
- LibVLCSharp 4.x (or the current stable) is compatible with .NET 10 and Avalonia 11.3.x. If
  compatibility research in Step 20 reveals an incompatibility, fall back to process-launched VLC
  for the Video tab and document the decision in `constraints.md`.
- Bookmark thumbnails are stored as `.jpg` files, consistent with cover images and thumbnail frames.
- `BookmarkDescriptions` batch-load performance is acceptable for typical library sizes (hundreds to
  low thousands of movies); each batch query returns at most `N_movies × avg_bookmarks_per_movie`
  rows.
- `IExternalPlayerService` belongs in the App layer (no domain logic; parallels `IFileLauncherService`).
- The legacy `favorite` column in `movies_bookmarks` is preserved in the database but not surfaced
  in the rebuild domain model.


## Constraints

- Core must have zero external NuGet dependencies — LibVLCSharp packages go in
  `VideoIndexer.App.csproj` only.
- `TreatWarningsAsErrors=true`, `WarningLevel=9999` — all new code must compile clean.
- All four `IAppPaths` implementations must be updated simultaneously in Step 13.
- Bookmark tagging UI is out of scope — `tags_bookmarks` and `ITagsRepository` bookmark methods
  remain as stubs.
- `LibraryOptions.FfmpegPath` user override is step 1 in the resolution chain (highest priority).
- `Bookmark` domain model does not map the legacy `favorite` column.
- Migration m042 only adds a column — no structural changes to existing columns.
- Bookmarks Browser pagination is fixed at 40 items per page.
- `CancellationToken` cancellation in all `ILibraryScanner.RefreshAsync` paths must return normally
  (no `OperationCanceledException` propagation; existing constraint unchanged).
- `AppOptions` mutations must use `with { }` and persist through `ISettingsService.SaveAsync`
  (existing constraint; applies to new `BookmarkBrowserOptions.ZoomLevel`).


## Out of Scope

- Bookmark tagging UI (wiring `tags_bookmarks`, `ConnectBookmarkTagAsync` in the UI).
- macOS / Linux LibVLC native package declarations (platform-specific research deferred).
- `visualtags_*` tables (legacy; no rebuild spec).
- `action_defs` / `movies_actionshots` (legacy-only; no rebuild spec).
- Copy Movies and Delete on Disk operations (`movie-management-specification.md §5`).
- `InMemoryBookmarkRepository` and `InMemoryBookmarkPresetRepository` test fakes. App-layer tests use hand-written fakes following the `FakeMovieRepository` pattern (`tests/VideoIndexer.App.Tests/TestHelpers/`). `InMemoryBookmarkRepository`, `InMemoryBookmarkPresetRepository` are deferred because they are needed only in unit tests; App-layer tests use `FakeBookmarkRepository` and `FakeBookmarkBrowserRepository` per-test files.
- Auto-seek from Bookmarks Browser to the embedded player (no cross-VM coordination mechanism
  exists; `PlayBookmarkCommand` navigates to the Movie Editor for the bookmark's movie only).


## Acceptance Criteria

1. `.\test.ps1` (unit + app suites) passes with 0 failures after every phase.
2. `dotnet build -c Release` produces 0 warnings and 0 errors after every phase.
3. `MoviesListViewModel` and `MovieEditorViewModel` implement `IDisposable`; their views' `OnUnloaded`
   handlers call `Dispose()`.
4. `DapperThumbnailRepository` is registered as `AddSingleton` in `Program.cs`.
5. The `IntervalSeconds` property in `ThumbnailGenerationRequest` throws `ArgumentOutOfRangeException` when initialised with `0` or negative values via the record-body override guard.
6. `tests/VideoIndexer.Tests/Fixtures/SyncProgress.cs` exists; `FfmpegThumbnailGeneratorServiceTests.cs`
   uses it.
7. `constraints.md` contains the `FormattableString.Invariant` test rule.
8. `DapperThumbnailRepositoryTests.cs` contains ≥ 6 integration tests that self-skip without a
   live DB.
9. `m042_add_bookmark_thumbnail_path.sql` exists; `DatabaseBootstrapper.ExpectedRevision == 42`.
10. A non-null `LibraryOptions.FfmpegPath` value is used by `FfmpegRunner` with higher priority than
    the provisioner path.
11. The Video tab in `MovieEditorView` renders a LibVLCSharp player when a movie with a valid file
    path is loaded; Play/Pause, seek, and mute controls are functional.
12. Creating a bookmark at the current player position persists to `movies_bookmarks` and appears in
    the sidebar panel.
13. Clicking a bookmark in the sidebar seeks the player to the bookmark's `position_ms`.
14. The Bookmarks Browser shows bookmarks across the library, paginates at 40, and responds to
    search text and minimum-rating filters.
15. `HasRatedBookmarks()`, `AmountBookmarks`, and `BookmarkContains(text)` DSL expressions evaluate
    correctly against the corresponding `MovieListItem` fields.
16. **Play** / **Play Fullscreen** context menu actions launch VLC when `UseVlc = true` and
    `VlcExecutablePath` is set.
17. All four manifest documents reflect the final M10 state.


## Testing Strategy

Unit tests (xUnit, no external dependencies) cover: `ThumbnailGenerationRequest` guard;
`FilterExpressionEvaluator` new bookmark identifier branches; `DesktopExternalPlayerService`
argument construction and guard; `SyncProgress<T>` fixture usage; `BookmarkItemViewModel.PositionLabel`
formatting; `FfmpegRunner` three-step resolution chain via fake settings.

Integration tests (DB-required, self-skip) cover: `DapperThumbnailRepository` full CRUD;
`DapperBookmarkRepository` full CRUD.

App-layer headless tests (Avalonia `HeadlessApp`) cover: `BookmarksPanelViewModel` load, add, edit,
delete with `InMemoryBookmarkRepository`; `BookmarksBrowserViewModel` pagination and filter
delegation; `PlayerViewModel` command states (`SeekRelativeCommand` clamping,
`PlayPauseCommand` toggle). App-layer headless tests use `FakeBookmarkRepository` and `FakeBookmarkBrowserRepository` (hand-written fakes in `tests/VideoIndexer.App.Tests/TestHelpers/`, following the `FakeMovieRepository` pattern), not a mocking framework.


## Test Plan

| Test File | What It Asserts | Acceptance Criterion |
|---|---|---|
| `tests/VideoIndexer.Tests/ThumbnailGenerationRequestTests.cs` | `IntervalSeconds = 0` and `-1` throw `ArgumentOutOfRangeException`; `IntervalSeconds = 1` does not throw | AC5 |
| `tests/VideoIndexer.Tests/Filtering/FilterExpressionParserTests.cs` | `HasRatedBookmarks()` no longer throws; `AmountBookmarks > 2` no longer throws; `BookmarkContains("x")` no longer throws | AC15 |
| `tests/VideoIndexer.Tests/Filtering/FilterExpressionEvaluatorTests.cs` | `HasRatedBookmarks()` true when `item.HasRatedBookmarks = true`; false when false; `AmountBookmarks > 0` true/false; `BookmarkContains("foo")` matches case-insensitively; no-match returns false | AC15 |
| `tests/VideoIndexer.Tests/Fixtures/SyncProgress.cs` | (Verified by existing `FfmpegThumbnailGeneratorServiceTests.cs` referencing the promoted type) | AC6 |
| `tests/VideoIndexer.Tests/FfmpegRunnerTests.cs` (extended) | When `Library.FfmpegPath` is set, `ResolveBinaryPath()` returns it; when null, falls back to provisioner path; when both null, returns "ffmpeg" | AC10 |
| `tests/VideoIndexer.Tests/Services/DesktopExternalPlayerServiceTests.cs` | `Play(path)` builds correct argument list; `PlayFullscreen(path)` includes `--fullscreen`; `CanUseExternalPlayer = false` when `UseVlc = false` | AC16 |
| `tests/VideoIndexer.Infrastructure.Tests/Database/DapperThumbnailRepositoryTests.cs` | `InsertAsync`, `GetByMovieIdAsync`, `GetCountByMovieIdAsync`, `GetIdsByMovieIdAsync`, `DeleteAsync`, `DeleteAllByMovieIdAsync` (6 self-skipping tests) | AC8 |
| `tests/VideoIndexer.Infrastructure.Tests/Database/DapperBookmarkRepositoryTests.cs` | `InsertAsync`, `GetByMovieIdAsync`, `UpdateAsync`, `DeleteAsync`, `IncrementClickAsync`, `GetAllDescriptionsAsync` (6 self-skipping tests) | AC12 |
| `tests/VideoIndexer.App.Tests/PlayerViewModelTests.cs` | `PlayPauseCommand` toggles `IsPlaying`; `SeekRelativeCommand` clamps to `[0, DurationMs]`; `Dispose()` does not throw | AC11 |
| `tests/VideoIndexer.App.Tests/BookmarksPanelViewModelTests.cs` | `LoadAsync` populates `Bookmarks`; `DeleteBookmarkCommand` removes item; `BookmarkClickedCommand` raises `BookmarkSeekRequested` with correct `PositionMs` | AC12–AC13 |
| `tests/VideoIndexer.App.Tests/BookmarksBrowserViewModelTests.cs` | `LoadPageCommand(1)` populates `Items`; `SearchText` change triggers reload; `ClearFilterCommand` resets filters | AC14 |


## Documentation Updates

| Document | Change |
|---|---|
| `docs/agents/project-manifest/constraints.md` | Add: IDisposable obligation for Bitmap-holding VMs; `DapperThumbnailRepository`/`DapperBookmarkRepository` singleton note; LibVLCSharp disposal contract; FfmpegPath 3-step chain; `BookmarkThumbnailPath` convention; m042 rollback procedure; `FormattableString.Invariant` test rule |
| `docs/agents/project-manifest/api-surface.md` | Add all new types/interfaces from Steps 9–32; update `LibraryOptions`, `MovieListItem`, `ThumbnailGenerationRequest`, `AppOptions`, `IAppPaths`, `IMovieRepository` entries |
| `docs/agents/project-manifest/file-tree.md` | Add every new file from Phases 0–8 with one-line annotations |
| `docs/agents/project-manifest/data-flows.md` | Add: Player startup + view-count flow; Bookmark creation flow; Bookmarks Browser load/paginate flow |
| `docs/agents/project-manifest/tech-stack.md` | Update expected schema revision from 41 to 42 |


## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **LibVLCSharp version incompatibility with .NET 10 / Avalonia 11.3.x** | Step 20 is the first WP of Phase 4; if no compatible version exists, fall back to a process-launched VLC WebView or embed-less preview (doc the decision in `constraints.md`); keep Phases 5–9 unblocked |
| **`BookmarkDescriptions` batch-load performance on large libraries** | Limit `GetAllDescriptionsAsync` to the first 60 characters per description (`SUBSTRING(description, 1, 60)`); add a `LIMIT 10000` safety cap; log query duration at DEBUG level |
| **`movies_bookmarks` table absent on a fresh dev DB that pre-dates legacy schema** | The `DatabaseBootstrapper` will refuse to start when `db_revision < 42`; the m042 `ALTER TABLE` will fail with "Table 'movies_bookmarks' doesn't exist." Add a pre-check in m042: `IF EXISTS (SELECT 1 FROM information_schema.tables WHERE table_name = 'movies_bookmarks' AND table_schema = DATABASE()) THEN ALTER TABLE...; END IF;` and a fallback `CREATE TABLE ... ELSE` branch |
| **`IDisposable` on long-lived singletons not called by the DI container** | `IServiceProvider.Dispose()` calls `Dispose()` on transients/scoped but **not** singletons. Verify `OnUnloaded` fires on navigation away; add a `LogWarning` in `Dispose()` of `MoviesListViewModel` if called a second time (double-dispose guard) |
| **Precision seek bar UX complexity** | If the `Slider`-based seek bar is insufficient, isolate seek-delta computation in a `PrecisionSeekBar` UserControl with a unit-testable `double ComputeSeekDelta(double clickX, double totalWidth, double maxSeekMs)` static method, following the `CoverImageCropperControl` precedent |
| **M10 scope is large; risk of partial delivery** | Phases 0–2 (debt + schema + core) are fully self-contained and should be treated as mandatory. Phases 4–6 (player + bookmarks) can be staged if time is a concern: deliver the bookmark data-layer first, then the player UI, then the Browser |
