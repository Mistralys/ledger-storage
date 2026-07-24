# M9 Images Rework 2 — Synthesis Report

**Plan:** `2026-05-22-m9-images-rework-2`
**Version:** 1.2.0 → 1.3.0
**Completed:** 2026-05-30
**Work Packages:** 31 / 31 COMPLETE

---

## Executive Summary

Milestone M9 (Images Rework 2) delivered a comprehensive bookmark infrastructure, an embedded
video player, and a bookmarks browser for the `video-indexer-v2` desktop application. The
milestone spanned 31 work packages covering:

- **IDisposable lifecycle hardening** across all primary ViewModels (WP-001, WP-029, WP-030)
- **Bookmark domain layer** — models, repository interfaces, DI-registered Dapper
  implementations (WP-009 – WP-012, WP-016 – WP-018)
- **LibVLCSharp integration** — embedded video player with screenshot capture, seek controls,
  and external player launch (WP-013, WP-021, WP-022, WP-026, WP-029)
- **Bookmark CRUD UI** — `BookmarkSettingsDialog` (create/edit), `BookmarksPanelView` (live
  panel inside movie editor), `BookmarksBrowserView` (standalone primary-nav destination)
  (WP-023, WP-027, WP-028, WP-030)
- **Filter DSL extensions** — `HasRatedBookmarks()`, `AmountBookmarks`, `BookmarkContains()`
  promoted to active identifiers (WP-025)
- **Database migration m042** — `thumbnail_path` column added to `movies_bookmarks` (WP-007)
- **Milestone documentation** — `manifest.md`, project-manifest files (`api-surface`,
  `file-tree`, `data-flows`, `constraints`) all updated (WP-031)

The build ships clean: **0 warnings, 0 errors**, **1,068 tests passing** (531 App, 326 Core,
211 Infrastructure), 6 skipped by design (live-DB infrastructure tests).

---

## Metrics

| Metric | Value |
|---|---|
| Work Packages | 31 / 31 COMPLETE |
| Tests Passing | 1,068 |
| Tests Failed | 0 |
| Tests Skipped | 6 (expected — live-DB integration) |
| Build Warnings | 0 |
| Build Errors | 0 |
| DB Schema Revision | 41 → 42 (m042) |
| Version Bump | 1.2.0 → 1.3.0 |
| Rework Pipelines | 1 (WP-027 code-review: FAIL → PASS) |

---

## Deliverables by Theme

### 1. IDisposable Lifecycle Hardening

- **WP-001** — `MoviesListViewModel` and `MovieEditorViewModel` implement `IDisposable`;
  `OnUnloaded` wiring added; constraints.md rule added.
- **WP-029** — `PlayerViewModel` wired into `MovieEditorViewModel` via `Func<PlayerViewModel>`
  factory; `DisposeImageViewModels` renamed to `DisposeSubViewModels` to reflect expanded scope.
- **WP-030** — `BookmarksPanelViewModel` wired with event unsubscribe-before-dispose ordering
  constraint enforced.

### 2. Infrastructure & DI Fixes

- **WP-002** — `DapperThumbnailRepository` registration corrected: `AddTransient` →
  `AddSingleton` (captive dependency fix).
- **WP-003** — `ThumbnailGenerationRequest.IntervalSeconds` model-level guard
  (`ArgumentOutOfRangeException`); three-layer defense: model + `Math.Max` clamp +
  `Debug.Assert`.
- **WP-004** — `SyncProgress<T>` promoted from private inner class to shared Fixtures helper.
- **WP-005** — `FfmpegRunner.ResolveBinaryPath()` 3-step resolution chain: user override →
  provisioner → PATH; Settings UI `TextBox` added.
- **WP-007** — DB migration m042: `thumbnail_path VARCHAR(512) NULL` added to
  `movies_bookmarks`; version bumped 1.2.0 → 1.3.0.
- **WP-013** — LibVLCSharp 3.9.7.1 + VideoLAN.LibVLC.Windows 3.0.23.1 added; `LibVLC`
  registered as `AddSingleton`.

### 3. Bookmark Domain Layer

- **WP-009** — `MovieListItem` extended: `BookmarkCount`, `HasRatedBookmarks`,
  `BookmarkDescriptions`.
- **WP-010** — 7 new Core bookmark domain files: `Bookmark`, `BookmarkPreset`,
  `BookmarkListItem`, `BookmarkBrowserQuery`, `IBookmarkRepository`,
  `IBookmarkPresetRepository`, `IBookmarkBrowserRepository`.
- **WP-011** — `IAppPaths.BookmarkThumbnailPath(hash, bookmarkId)` added to all 5
  implementations.
- **WP-012** — `IMovieRepository.IncrementViewCountAsync` added; `DapperMovieRepository`
  implementation complete.
- **WP-016** — `DapperBookmarkRepository` implementing `IBookmarkRepository` +
  `IBookmarkBrowserRepository`; shared-instance DI pattern; SQL column name corrections
  (`milliseconds`, `added`).
- **WP-017** — `DapperBookmarkPresetRepository`: `GetAllAsync`, `InsertAsync`, `DeleteAsync`;
  singleton registration.
- **WP-018** — `DapperMovieCatalogRepository` extended: bookmark aggregates via `LEFT JOIN`;
  `COUNT(DISTINCT)` prevents join inflation from the join.

### 4. Video Player

- **WP-022** — `PlayerViewModel`: `MediaPlayer` lifecycle, `LoadVideoCommand`,
  `PlayPauseCommand`, `SeekRelativeCommand` (clamped), `ScreenshotCommand`; 15 unit tests.
  Fix: temp file cleanup moved to `finally` block.
- **WP-026** — `PlayerView.axaml`: `VideoView`, progress slider, toolbar, precision seek
  sliders, manual seek `TextBox`; `MsToTimeConverter` as static on `PlayerView`.
  CommunityToolkit.Mvvm Async-strip convention documented in constraints.md.

### 5. Bookmark UI

- **WP-014** — `BookmarkItemViewModel`: `IDisposable`, immutable `BookmarkId`/`PositionMs`,
  observable `Description`/`Rating`/`ThumbnailBitmap`, computed `PositionLabel` (hh:mm:ss).
- **WP-015** — `IExternalPlayerService` + `DesktopExternalPlayerService`: injection-safe VLC
  launch via `ProcessStartInfo.ArgumentList`; 3 unit tests.
- **WP-020** — `IScreenshotPreviewService` + `ScreenshotPreviewViewModel` +
  `AvaloniaScreenshotPreviewService`; temp file cleanup moved to `finally` block (Reviewer
  fix-forward).
- **WP-021** — `MoviesListViewModel`: VLC context-menu (`Play in VLC` / `Play Fullscreen in
  VLC`); `ExternalPlayerAvailable` property; 12 unit tests.
- **WP-023** — `BookmarkSettingsViewModel` + `BookmarkSettingsView` +
  `AvaloniaBookmarkSettingsService`; create/edit modes; 23 unit tests.
- **WP-024** — `BookmarksBrowserViewModel`: full pagination, filtering, and navigation; all
  12 ACs met.
- **WP-027** — `BookmarksPanelViewModel`: `AddBookmarkCommand`, `DeleteBookmarkCommand`,
  `SavePresetCommand`, `RenameBookmarkCommand`; `HasNoBookmarks` / `HasRating` computed
  properties; rework required to fix `ObjectConverters` misuse.
- **WP-028** — `BookmarksBrowserViewModel` registered as 6th primary-nav destination in
  `MainContentViewModel`; `NavigateToMovieEditorAsync` shared helper extracted.

### 6. Filter DSL

- **WP-025** — `HasRatedBookmarks()`, `AmountBookmarks`, `BookmarkContains()` promoted from
  deferred M10 identifiers to active `FunctionIdentifiers`/`NumericIdentifiers`; 11 new
  tests.

### 7. Documentation & Testing

- **WP-006** — constraints.md rule: culture-invariant float format strings in tests.
- **WP-008** — `DapperThumbnailRepositoryTests`: 6 `SkippableFact` live-DB integration tests.
- **WP-019** — No-op: `IncrementViewCountAsync` was pre-implemented by WP-012.
- **WP-031** — `manifest.md` created; `api-surface.md`, `file-tree.md`, `data-flows.md`,
  and `constraints.md` fully updated for M9/M10.

---

## Issues & Rework

### Rework Pipelines

| WP | Stage | Outcome | Root Cause |
|---|---|---|---|
| WP-027 | code-review | FAIL → PASS | `ObjectConverters` misuse: `Bookmarks.Count` and `Rating` bound directly through converters instead of computed properties (`HasNoBookmarks`, `HasRating`). Fixed by adding computed properties and `CollectionChanged` wiring. |

### Ledger Issues (Project Manager Interventions)

1. **WP-023 / WP-025** — Stray acceptance criteria from other WPs contaminated ledger
   entries, blocking auto-finalization. PM resolved manually.
2. **WP-027** — Two mis-assigned ACs (`SavePresetCommand`/`UpsertAsync`,
   `RenameBookmarkCommand`) that don't match the WP spec blocked auto-COMPLETE. PM forced
   COMPLETE manually.
3. **WP-030** — ACs `#2` (`IPlayerPositionService`) and `#3` (`ApplyFilterCommand`) were
   stale plan-draft artifacts (neither interface/command exists in the codebase). QA marked
   them as met to unblock.

### Known Open Issue

> **FfprobeRunner priority inversion (flagged in WP-005):** `FfprobeRunner.ResolveBinaryPath()`
> checks the provisioner path *before* the user override (`Library.FfprobePath`) — the reverse
> of `FfmpegRunner`. The user's ffprobe path setting is silently ignored. Documented in
> constraints.md. **Needs a follow-up WP.**

---

## Strategic Recommendations

### Gold Nuggets

1. **Avalonia compiled-binding DataContext/IsVisible pitfall (WP-029)**
   When you need both `DataContext='{Binding SubVm}'` and `IsVisible='{Binding SubVm,
   Converter=IsNotNull}'` on the same AXAML element, Avalonia's compiled-binding compiler
   resolves `IsVisible` against the `DataContext` type (the sub-VM) instead of the parent VM,
   causing a compile-time binding error. The canonical fix ("Panel-wrapper for sub-VM
   visibility") is to place `IsVisible` on an outer `Panel` binding to the parent VM while the
   inner view carries `DataContext='{Binding SubVm}'`. Documented in constraints.md.

2. **C# record `with` expressions bypass property-setter guards (WP-003)**
   C# record `with` expressions reconstruct instances without invoking property setters, so any
   `ArgumentOutOfRangeException` or clamping logic in a setter is silently bypassed. The
   required defence is three-layered: model-level setter guard + `Math.Max`/`Math.Min` clamp at
   the call site + `Debug.Assert` for invariant validation.

3. **CommunityToolkit.Mvvm strips `Async` suffix from command names (WP-026)**
   The CommunityToolkit.Mvvm source generator strips the `Async` suffix from `[RelayCommand]`
   method names: a method named `ScreenshotAsync` generates `ScreenshotCommand`, not
   `ScreenshotAsyncCommand`. AXAML bindings must use the stripped name. This is the library's
   intentional convention but surprises developers expecting the `Async` suffix to be preserved.
   Documented in constraints.md.

---

## Next Steps

1. **FfprobeRunner priority fix (follow-up WP)** — Correct `FfprobeRunner.ResolveBinaryPath()`
   to match `FfmpegRunner`'s three-step order: user override → provisioner → PATH.
2. **MovieEditorViewModel parameter-object refactor** — The constructor now has 22 parameters.
   A `MovieEditorOptions` record or builder pattern should be introduced in the next milestone
   to prevent further growth. Tracked in `CHANGELOG.md` under Technical Debt.
3. **`NOW()` vs `UTC_TIMESTAMP()` alignment (WP-016)** — Bookmark timestamps use `NOW()`
   (server local time) rather than `UTC_TIMESTAMP()`. Decide on a project-wide timezone
   strategy before persisting additional timestamp columns.
4. **M10 planning** — Filter DSL operators and bookmark thumbnail generation are partially
   complete. Review deferred M10 identifiers and plan the next milestone accordingly.
5. **Release Engineer PR** — `manifest.md` is ready at
   `docs/agents/plans/2026-05-22-m9-images-rework-2/manifest.md`; the Release Engineer should
   author the PR description referencing it.
