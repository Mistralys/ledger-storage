# Plan: M9 — Images

## Plan Audit Cycles
- Audits: 3 — Plan Auditor v1.3.1 (Cycle 1: FAIL — 1 Critical, 4 Major, 6 Minor; integrated. Cycle 2: PASS WITH FINDINGS — 0 Critical, 1 Major, 3 Minor; integrated. Cycle 3: PASS WITH FINDINGS — 0 Critical, 0 Major, 1 Minor; deferred)
- Architectural Reviews: 1 — Plan Architect Reviewer v1.4.0


## Summary

M9 delivers the full Images milestone: cover image management (crop, rotate, colour-adjust via `cv.cim` in the Movie Editor's Cover Image tab), thumbnail management (a grid view with context actions in the Thumbnails tab), thumbnail generation (FFmpeg-backed interval extraction in a new Generate tab and a multi-movie dialog from the main list), and the zoomed thumbnail viewer dialog. Supplementary carry-forward work includes: a live cover-image preview in the movies-list cover panel; wiring the previously-stubbed `GenerateThumbnailsAsync` command in `MoviesListViewModel`; fixing `HasCoverImage` in `DapperMovieCatalogRepository` from a hardcoded SQL `0` to an actual post-query filesystem check; and wiring the four ffprobe-deferred fields (`FileType`, `Resolution`, `Duration`, `Bitrate`) in `MoviePropertiesViewModel`. On completion, the centre `TabControl` in `MovieEditorView` is fully functional for Cover Image, Thumbnails, and Generate tabs; the Video tab remains stubbed for M10.


## Architectural Context

| Area | Current State | What M9 Changes |
|---|---|---|
| `IAppPaths` / `AppPaths` | `MovieDataDirectory(hash)` only | Add `CoverImagePath(hash)` and `ThumbnailFilePath(hash, id)` helpers |
| `MovieListItem.HasCoverImage` | Always `false` — hardcoded `0 AS HasCoverImage` in SQL | Post-load `File.Exists` enrichment pass in `DapperMovieCatalogRepository` |
| `MovieEditorView` centre tabs | Cover Image / Thumbnails / Video stubs (placeholder text) | Replace Cover Image + Thumbnails stubs; add Generate tab (Video stays M10 stub) |
| `MovieEditorViewModel` | No child VMs for image tabs | Add `CoverImageVm`, `ThumbnailsVm`, `GeneratorVm` properties; load on `LoadAsync` |
| `MoviesListViewModel.CoverImageBitmap` | No such property; cover panel shows a placeholder `TextBlock` | Load `Avalonia.Media.Imaging.Bitmap` from `cv.cim` on selection change |
| `MoviesListViewModel.GenerateThumbnailsAsync` | Static no-op stub | Open multi-movie generator dialog |
| `MoviePropertiesViewModel` | `FileType`, `Resolution`, `Duration`, `Bitrate` return `"—"` (TODO M9 comments) | Wired via `IFfprobeRunner` + `_movie.CurrentFilePath` (no extra DB round-trip); fire-and-forget async load |
| `movies_thumbnails` DB table | Exists at schema revision 41; no repository or service wraps it | Add `IThumbnailRepository` / `DapperThumbnailRepository`, generator service, cover service |
| `IMovieRepository` | No method to resolve the current video file path | Add `GetCurrentFilePathsAsync(IReadOnlyList<long>)` (batch); also expose `Movie.CurrentFilePath` via JOIN in `GetByIdAsync` |
| FFmpeg invocation for frames | `FfprobeRunner` covers JSON probing; no runner for frame extraction | Add `IFfmpegRunner` / `FfmpegRunner` — two-step binary resolution chain (provisioner path → PATH fallback); no `LibraryOptions.FfmpegPath` user-override in M9 scope |

**Existing entry points and stubs:**
- `MovieEditorView.axaml` (centre TabControl) — three stub `TabItem` elements: "Cover Image" / "Thumbnails" / "Video" (each has a placeholder `TextBlock "… (M9)"`). Only "Video" remains after this milestone; a "Generate" tab is added.
- `MoviePropertiesViewModel` — `FileType`, `Resolution`, `Duration`, `Bitrate` currently return `"—"` with `// TODO M9:` comments.
- `DapperMovieCatalogRepository.GetMovieListAsync` — `0 AS HasCoverImage` in SQL.
- `MovieEditorViewModel` — no thumbnail or cover-image sub-VMs yet.
- `FilterExpressionEvaluator` — `HasCoverImage`, `HasThumbnails`, `AmountThumbnails` already wired; reads `MovieListItem.HasCoverImage` and `ThumbnailCount`. No DSL changes needed.

**File and data conventions (from legacy SPDB Indexer):**
- Cover image path: `{MovieDataDirectory(hash)}/cv.cim` — JPEG content with `.cim` extension.
- Thumbnail path: `{MovieDataDirectory(hash)}/{thumbnail_id}.thb2` — JPEG content with `.thb2` extension.
- Both conventions match the original spdb-indexer; M9 reads and writes against existing on-disk data.
- Database table `movies_thumbnails`: `thumbnail_id UINT AI PK`, `milliseconds UINT NOT NULL`, `movie_id UINT NOT NULL FK→movies(movie_id) ON DELETE CASCADE`, `custom ENUM('yes','no') DEFAULT 'no'`, `mosaic ENUM('yes','no') DEFAULT 'no'`, `favorite ENUM('yes','no') DEFAULT 'no'`.
- `movies_filenames.filename` stores the **full absolute path** to the current file on disk. After obfuscation, the path has an obfuscated extension; the full path is always the canonical current location.
- FFmpeg frame extraction command: `ffmpeg -y -loglevel panic -nostdin -ss HH:MM:SS.fff -i "src" -vframes 1 -f image2 "dest"`.

**Relevant existing infrastructure:**
- `src/VideoIndexer.Core/Abstractions/IFfprobeRunner.cs` — `ProbeAsync` returns raw ffprobe JSON; re-used by `FfmpegThumbnailGeneratorService` for duration and by `MoviePropertiesViewModel` for media info.
- `src/VideoIndexer.Infrastructure/Library/FfprobeRunner.cs` — resolves the binary via `ISettingsService.Current.ExternalTools.Ffmpeg.FfprobePath` → `LibraryOptions.FfprobePath` → `"ffprobe"` on PATH. `FfmpegRunner` uses a two-step chain (provisioner path → `"ffmpeg"` on PATH); no `LibraryOptions.FfmpegPath` field exists in M9.
- `src/VideoIndexer.Infrastructure/AppPaths.cs` — `MovieDataDirectory(hash)` creates the data folder on demand; cover image and thumbnails live inside it.
- `src/VideoIndexer.App/Program.cs` — all DI registrations; M9 additions follow the existing factory-lambda pattern.
- `Directory.Packages.props` — NuGet CPM; a new `SixLabors.ImageSharp` entry must be added.
- `src/VideoIndexer.App/Views/MoviesListView.axaml` — cover panel (column 1) holds a placeholder `TextBlock`; context menu "Generate Thumbnails" binds to `GenerateThumbnailsCommand`.


## Approach / Architecture

### Layer Overview

```
VideoIndexer.Core
  ├── IAppPaths            (extend: CoverImagePath, ThumbnailFilePath)
  ├── Models               (new: Thumbnail, CropRect, CoverImageTransform,
  │                               ThumbnailGenerationRequest, ThumbnailGenerationResult,
  │                               ThumbnailGenerationProgress)
  ├── Models/Movie         (extend: CurrentFilePath string?)
  └── Abstractions
        IFfmpegRunner
        IThumbnailRepository
        ICoverImageService
        IThumbnailGenerationService
        IMovieRepository   (extend: GetCurrentFilePathsAsync)

VideoIndexer.Infrastructure
  ├── NuGet: SixLabors.ImageSharp 3.x  (image processing, Infrastructure only)
  ├── AppPaths.cs                       (implement new IAppPaths methods)
  ├── Library/FfmpegRunner              (implements IFfmpegRunner)
  ├── Library/DapperThumbnailRepository (implements IThumbnailRepository)
  ├── Library/CoverImageService         (implements ICoverImageService via ImageSharp)
  ├── Library/FfmpegThumbnailGeneratorService (implements IThumbnailGenerationService)
  ├── Library/DapperMovieRepository     (extend GetByIdAsync + new batch path query)
  └── Library/DapperMovieCatalogRepository (HasCoverImage post-query enrichment)

VideoIndexer.App
  ├── Services
  │     IThumbnailViewerService / AvaloniaThumbnailViewerService
  │     IThumbnailGenerationDialogService / AvaloniaThumbnailGenerationDialogService
  ├── ViewModels
  │     ThumbnailItemViewModel (new — row VM for thumbnails grid)
  │     ThumbnailsViewModel (new)
  │     CoverImageViewModel (new)
  │     ThumbnailGeneratorMovieViewModel (new — per-movie progress row)
  │     ThumbnailGeneratorViewModel (new)
  │     ThumbnailViewerViewModel (new)
  │     MovieEditorViewModel (extend: sub-VMs, file path, new service deps)
  │     MoviePropertiesViewModel (extend: IFfprobeRunner, LoadAsync)
  │     MoviesListViewModel (extend: CoverImageBitmap, GenerateThumbnailsAsync)
  └── Views
        CoverImageCropperControl.axaml / .cs  (new custom UserControl)
        CoverImageView.axaml / .cs
        ThumbnailsView.axaml / .cs
        ThumbnailGeneratorView.axaml / .cs
        ThumbnailViewerView.axaml / .cs
        MovieEditorView.axaml (replace stub tabs)
        MoviesListView.axaml (cover panel + Generate Thumbnails wiring)
```

### Key Subsystem Details

**`IFfmpegRunner` / `FfmpegRunner`:** A new `IFfmpegRunner` interface with a single method `ExtractFrameAsync(videoPath, outputPath, positionMs, ct)` is added to `VideoIndexer.Core/Abstractions/` and implemented by `FfmpegRunner` in `VideoIndexer.Infrastructure/Library/`. The binary resolution chain uses a two-step chain (provisioner path → `"ffmpeg"` on PATH) — the user-override step is intentionally omitted in M9 as no `LibraryOptions.FfmpegPath` field exists. Throws `InvalidOperationException` on non-zero exit code — same contract as `IFfprobeRunner`. `FfmpegThumbnailGeneratorService` receives `IFfmpegRunner` by constructor injection, keeping frame extraction testable in isolation.

**`IAppPaths` path helpers:** Two new methods centralise path construction:
- `CoverImagePath(string hash)` → `{MovieDataDirectory(hash)}/cv.cim`
- `ThumbnailFilePath(string hash, long thumbnailId)` → `{MovieDataDirectory(hash)}/{thumbnailId}.thb2`

No caller constructs these paths manually.

**Thumbnail file I/O pattern:** The repository inserts a DB row first to obtain `thumbnail_id`, then returns the `Thumbnail` record. The caller derives the file path via `IAppPaths.ThumbnailFilePath(hash, thumbnailId)` and invokes `IFfmpegRunner.ExtractFrameAsync` to write the JPEG. If extraction throws, the caller deletes the DB row (rollback). File deletion is always the caller's (service's) responsibility — the repository is a pure DB wrapper.

**`HasCoverImage` enrichment:** `DapperMovieCatalogRepository.GetMovieListAsync` performs a post-query loop over returned `MovieListItem` rows to set `HasCoverImage = File.Exists(_appPaths.CoverImagePath(item.Hash))`. This mirrors the existing effective-tag enrichment pass. `IAppPaths` is injected as an **optional** parameter (nullable, defaulting to `null`) so existing unit tests that construct the repository without it continue to pass; in production the parameter is always provided.

**`Movie.CurrentFilePath` resolution:** `DapperMovieRepository.GetByIdAsync` performs a LEFT JOIN with `movies_filenames` to retrieve the most-recently-added filename (`ORDER BY mf.created_at DESC LIMIT 1`). The value is exposed as `string? CurrentFilePath` on the `Movie` record. `MovieEditorViewModel.LoadAsync` stores it as `_currentFilePath` for sub-VM construction and the Properties command.

**Batch file-path query for multi-movie generation:** `IMovieRepository.GetCurrentFilePathsAsync(IReadOnlyList<long> movieIds, CancellationToken)` issues a single batch query against `movies_filenames` returning the latest path per `movie_id`. Used by `MoviesListViewModel.GenerateThumbnailsAsync` when building requests for multi-movie generation.

**`ThumbnailGenerationService` cancellation contract:** When the `CancellationToken` is signalled, the service returns normally (no `OperationCanceledException`). Partial frames for the in-progress movie are discarded (DB rows deleted, `.thb2` files removed). Completed movies retain their thumbnails. Mirrors the `ILibraryScanner` contract.

**Re-generation:** Before generating for a movie that already has thumbnails, `ThumbnailGeneratorViewModel` checks thumbnail counts, shows an in-view confirmation panel, and only calls the service after the user confirms. The service itself always performs a `DeleteAllByMovieIdAsync` pass before writing new thumbnails, making re-generation safe.

**ImageSharp usage:** `SixLabors.ImageSharp` is referenced only from `VideoIndexer.Infrastructure`. `CoverImageService` loads `.cim` files via `Image.Load(path)` (magic-byte detection), applies all transforms in a single `Mutate` pass (rotate → crop → brightness/contrast/gamma), and saves atomically by encoding to a `MemoryStream`, writing those bytes to a `.tmp` file, then `File.Move(overwrite: true)` — consistent with `JsonSettingsService`'s atomic write pattern. `ApplyAndSaveAsync` returns the in-memory bytes directly so the ViewModel can update the display without a second disk read. `RotateForPreviewAsync` applies a rotation-only `Mutate` pass, encodes to a `MemoryStream`, and returns the bytes without writing to disk — keeping ImageSharp entirely within Infrastructure.

**Cover image editor behaviour:** Brightness/Contrast/Gamma sliders update their values but do **not** change the live preview image — adjustments are applied only at save time ("Apply Now"). Rotation (CW/CCW) immediately updates the display by calling `ICoverImageService.RotateForPreviewAsync` (reads the `cv.cim` file, applies the rotation in-memory via ImageSharp, returns JPEG bytes) and creating a new `Bitmap` from the result to set `PreviewBitmap`. This matches the spec §1.2 constraint, keeps ImageSharp out of the App layer, and avoids per-slider processing on the UI thread.

**`ThumbnailGeneratorViewModel` dual mode:** Supports single-movie (editor Generate tab) and multi-movie (standalone dialog) via a constructor parameter `IReadOnlyList<ThumbnailGenerationRequest>`. The editor builds a single-element list; the standalone dialog builds a multi-element list. Both modes share the same View. `IsMultiMovieMode` (computed property) drives minor UI differences (overall progress row visibility).

**Cover image preview in movies list:** `MoviesListViewModel` gains an optional `IAppPaths` constructor parameter. A new `Bitmap? CoverImageBitmap` property is populated on selection change via a fire-and-forget async helper: cancel any prior load, read `File.ReadAllBytesAsync(coverPath)`, create `new Bitmap(new MemoryStream(bytes))`, dispose previous bitmap, set `CoverImageBitmap`. The cover panel in `MoviesListView.axaml` replaces its placeholder `TextBlock` with an `Image` control.

**`MoviePropertiesViewModel` media metadata:** One optional constructor parameter — `IFfprobeRunner?` — is added. A new `LoadMediaInfoAsync(CancellationToken)` method reads `_movie.CurrentFilePath` directly (populated via `GetByIdAsync` LEFT JOIN — no extra DB round-trip), then calls `IFfprobeRunner.ProbeAsync(filePath)`, and parses the JSON to populate `Duration`, `Resolution`, `FileType` (codec name), and `Bitrate` (Mbps). `AvaloniaMoviePropertiesService.ShowAsync` calls `LoadMediaInfoAsync` fire-and-forget after the window is shown so the dialog appears immediately. The four properties are backed by `[ObservableProperty]` to allow async update. If ffprobeRunner is null, or `CurrentFilePath` is null, fields remain `"—"`.


## Rationale

- **`IFfmpegRunner` over inline process spawning:** A separate interface keeps ffmpeg invocation unit-testable in isolation and mirrors the established `IFfprobeRunner` / `FfprobeRunner` pattern. The generator service receives it by constructor injection.
- **ImageSharp over SkiaSharp:** ImageSharp is pure managed .NET — no native `.so`/`.dylib`/`.dll` per platform. SkiaSharp would add native binary distribution complexity with no benefit for crop/rotate/colour operations.
- **`Movie.CurrentFilePath` via JOIN** rather than a separate `IMovieFileResolver` interface: one field on an existing model avoids a new abstraction for a DB-join. The JOIN is structurally adjacent to the existing `GetByIdAsync`.
- **Post-query `HasCoverImage` enrichment** rather than a SQL subquery or DB column: cover image state lives on disk, not in the DB. A SQL-only solution would require denormalising to a boolean DB column (write on every cover create/delete) or impossible cross-table filesystem subqueries. The O(N) `File.Exists` pass is the established precedent (effective-tag enrichment, M7) and is the only cross-platform option.
- **Optional `IAppPaths?` in `DapperMovieCatalogRepository`:** Existing unit tests construct the repository without `IAppPaths`. A required parameter change would break them. Optional injection with null-guard preserves backward compatibility.
- **Single `ThumbnailGeneratorViewModel` for both modes** rather than two VMs: the workflow is identical (set interval → generate → show progress → summary). An `IsMultiMovieMode` flag adds minimal branching while eliminating duplication.
- **`ThumbnailGenerationService` cancellation returns normally (no `OperationCanceledException`):** Consistent with `ILibraryScanner.RefreshAsync` — both are background workers launched by commands. An unexpected `OperationCanceledException` propagating up would fault `AsyncRelayCommand` machinery.
- **`ApplyAndSaveAsync` returns `byte[]`:** The ViewModel can reload the displayed image from the return value without a second disk read, avoiding a reload race.
- **Rotation updates preview immediately; colour sliders do not:** Per-change ImageSharp processing for colour adjustments would cause noticeable stutter. Rotation is a lossless in-memory transform on the already-loaded bytes and is fast enough for immediate feedback.


## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|---|---|---|---|
| Image processing library | `SixLabors.ImageSharp` (Infrastructure, pure managed) | `SkiaSharp` (native bindings), `System.Drawing` (Windows-only) | ImageSharp is cross-platform pure-managed, fits the rebuild's OS targets, and has a complete API for the required transforms. |
| FFmpeg abstraction | `IFfmpegRunner` / `FfmpegRunner` (throws on failure, mirrors `IFfprobeRunner`) | `IFrameCaptureService` (bool return, no throw); inline process spawning in generator service | Explicit interface keeps ffmpeg invocation testable; mirrors the established runner pattern. Throw-on-failure keeps error propagation explicit at the service boundary. |
| `HasCoverImage` source of truth | Post-query `File.Exists` enrichment (optional `IAppPaths?`) | Store `has_cover_image BOOL` in DB; SQL `EXISTS` subquery | DB column requires a write on every cover-image create/delete. Post-query loop is the only viable cross-platform option; optional injection preserves test compatibility. |
| File path in `Movie` model | `CurrentFilePath string?` via JOIN in `GetByIdAsync` + batch `GetCurrentFilePathsAsync` | Separate `IMovieFileResolver` service; runtime folder scanning; additional `GetCurrentFilenameAsync` round-trip in `MoviePropertiesViewModel` | JOIN on `movies_filenames` populates `Movie.CurrentFilePath` directly — `MoviePropertiesViewModel` reads it without a DB round-trip. Only the batch query remains in `IMovieRepository` for multi-movie generation; `GetCurrentFilenameAsync` is eliminated as redundant. |
| `CoverImageTransform` shape | `CoverImageTransform(Brightness, Contrast, Gamma, RotationDegrees, CropRect?)` with separate `CropRect` record | Flat `CoverImageAdjustments` with CropX/Y/SizeRatio floats | Separate `CropRect` encapsulates the three crop parameters coherently; the nullable `CropRect?` makes "no crop" unambiguous. |
| Thumbnail generation cancellation | Returns normally; discards in-progress movie's partial output | Throw `OperationCanceledException`; swallow silently | Consistent with `ILibraryScanner` contract. Avoids faulting `AsyncRelayCommand` infrastructure. Discarding partial frames is the safest behaviour (no zombie DB rows). |
| Dual-mode generator VM | Single `ThumbnailGeneratorViewModel` with `IReadOnlyList<ThumbnailGenerationRequest>` | Separate single- and multi-movie VMs | Workflow is identical across both modes. A single `IsMultiMovieMode` flag drives minor UI branching. |
| Cover colour live preview | No real-time preview — apply-only; rotation preview only | Live preview per slider change using ImageSharp | Per-change processing on the UI thread would cause stutter. Spec §1.2 explicitly specifies apply-at-save-time. |
| `IMoviePropertiesService` signature | Keep `ShowAsync(MoviePropertiesViewModel vm)` unchanged; caller constructs VM | Change to `ShowAsync(Movie movie)` so service constructs VM | Keeping the existing signature is the minimal change; `IFfprobeRunner` becomes an optional VM constructor param forwarded by `MovieEditorViewModel`; `Movie.CurrentFilePath` carries the file path without any `IMovieRepository` involvement. |


## Pattern Alignment

| Pattern | Status |
|---|---|
| Core interface + Infrastructure impl (`src/VideoIndexer.Core/Abstractions/` + `src/VideoIndexer.Infrastructure/`) | Followed for all five new interfaces (`IFfmpegRunner`, `IThumbnailRepository`, `ICoverImageService`, `IThumbnailGenerationService`, + extensions to `IAppPaths` / `IMovieRepository`). |
| `IFfprobeRunner` / `FfprobeRunner` binary resolution (three-step chain: provisioner path → user override → PATH) | Followed by `IFfmpegRunner` / `FfmpegRunner` with a **two-step** chain only (provisioner path → PATH fallback). The third step (user override via `LibraryOptions.FfmpegPath`) is deferred — `LibraryOptions` has `FfprobePath` but not `FfmpegPath`; adding a settings field is out of M9 scope. |
| CancellationToken contract: scanner returns normally (`src/VideoIndexer.Infrastructure/Library/LibraryScanner.cs`, `constraints.md`) | `FfmpegThumbnailGeneratorService` follows the same contract (no `OperationCanceledException` throw). |
| Post-query enrichment pass (`src/VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs` — effective-tag enrichment) | `HasCoverImage` enrichment follows the same post-query loop pattern. |
| Atomic file writes via `.tmp` + `File.Move(overwrite: true)` (`src/VideoIndexer.Infrastructure/Settings/JsonSettingsService.cs`) | `CoverImageService.ApplyAndSaveAsync` follows the same pattern. |
| `[ObservableProperty]` with `partial` classes (all existing ViewModels) | All new ViewModels follow this convention. |
| Compiled Avalonia bindings (`x:DataType`) (`constraints.md`) | All new AXAML files use compiled bindings. No reflective bindings introduced. |
| Sub-VM dialog services (`src/VideoIndexer.App/Services/AvaloniaDatabaseBackupDialogService.cs`) | `AvaloniaThumbnailViewerService` and `AvaloniaThumbnailGenerationDialogService` follow the same pattern: construct VM directly, show modal Window, unsubscribe in `try/finally`. |
| Sub-VMs not registered in DI; constructed by parent VM with contextual data (`constraints.md` — ObfuscationSettingsViewModel AddSingleton rule) | `ThumbnailsViewModel`, `CoverImageViewModel`, `ThumbnailGeneratorViewModel`, `ThumbnailViewerViewModel` are constructed by `MovieEditorViewModel` or the dialog service, not via DI. Dialog services (`AvaloniaThumbnailViewerService`, `AvaloniaThumbnailGenerationDialogService`) are `AddSingleton`. |
| DI factory (`Func<T, VM>`) for contextual VMs (`src/VideoIndexer.App/Program.cs`) | `ThumbnailGeneratorViewModel` does **not** use this factory pattern — it is constructed directly with a pre-resolved request list. |
| `PageHeaderView` code-behind wiring (`src/VideoIndexer.App/Views/MoviesListView.axaml.cs`) | Cover Image, Thumbnails, and Generate views are **embedded** in `MovieEditorView`'s TabControl and do not have their own PageHeader — the editor's existing header is used. |
| NuGet CPM — version declared in `Directory.Packages.props` only (`constraints.md`) | Followed for `SixLabors.ImageSharp`. |
| `File.Exists` sync accepted for local filesystem metadata (`src/VideoIndexer.Infrastructure/Library/ObfuscationService.cs`) | `HasCoverImage` enrichment loop uses `File.Exists` synchronously. Same justification: local filesystem metadata syscall is effectively instant on a desktop app. |
| Optional constructor parameters for test compatibility | `DapperMovieCatalogRepository` `IAppPaths?`; new services in `MovieEditorViewModel`, `MoviesListViewModel`, `MoviePropertiesViewModel` are optional. |


## Detailed Steps

### Phase 1 — Core Domain

**WP-001: `IAppPaths` extension + `AppPaths` implementation**

1. Add to `src/VideoIndexer.Core/Abstractions/IAppPaths.cs`:
   ```csharp
   string CoverImagePath(string hash);
   // Returns: {MovieDataDirectory(hash)}/cv.cim

   string ThumbnailFilePath(string hash, long thumbnailId);
   // Returns: {MovieDataDirectory(hash)}/{thumbnailId}.thb2
   ```
2. Implement both in `src/VideoIndexer.Infrastructure/AppPaths.cs`:
   ```csharp
   public string CoverImagePath(string hash) =>
       Path.Combine(MovieDataDirectory(hash), "cv.cim");

   public string ThumbnailFilePath(string hash, long thumbnailId) =>
       Path.Combine(MovieDataDirectory(hash), $"{thumbnailId}.thb2");
   ```
   No additional directory-creation logic in `AppPaths` — the data folder is created on demand by callers before first use (see WP-009 and WP-010).
3. Update both `FakeAppPaths` implementations to add the two new `IAppPaths` methods: (a) `tests/VideoIndexer.Tests/Fixtures/FakeAppPaths.cs` — return deterministic temp paths based on `hash` and `thumbnailId`; (b) `tests/VideoIndexer.App.Tests/TestHelpers/FakeAppPaths.cs` — return hardcoded fake paths (e.g. `Path.Combine(@"C:\FakeAppData", hash, "cv.cim")` and `Path.Combine(@"C:\FakeAppData", hash, $"{thumbnailId}.thb2")`).

---

**WP-002: Core models**

Add to `src/VideoIndexer.Core/Models/`:

**`Thumbnail.cs`**
```csharp
public sealed record Thumbnail
{
    public required long ThumbnailId { get; init; }
    public required long MovieId     { get; init; }
    public required long PositionMs  { get; init; }
    public required bool IsCustom    { get; init; }
    public required bool IsFavorite  { get; init; }
}
```

**`CropRect.cs`**
```csharp
/// <summary>
/// Fractional crop rectangle (all values in [0,1] relative to source image dimensions).
/// The crop region is always square: <see cref="Size"/> is the fraction of the source dimension.
/// </summary>
public sealed record CropRect(float X, float Y, float Size);
```

**`CoverImageTransform.cs`**
```csharp
public sealed record CoverImageTransform(
    float     Brightness,     // 1.0 = no change; range 0.0–2.0
    float     Contrast,       // 1.0 = no change; range 0.0–2.0
    float     Gamma,          // 1.0 = no change; range 0.1–5.0
    int       RotationDegrees, // 0, 90, 180, 270 (clockwise)
    CropRect? Crop            // null = no crop
);
```

**`ThumbnailGenerationRequest.cs`**
```csharp
public sealed record ThumbnailGenerationRequest(
    long   MovieId,
    string Hash,
    string VideoFilePath,
    string Label
);
```

**`ThumbnailGenerationResult.cs`**
```csharp
public sealed record ThumbnailGenerationResult(
    long    MovieId,
    string  Hash,
    int     Count,
    bool    Cancelled,
    bool    Error,
    string? ErrorMessage
);
```

**`ThumbnailGenerationProgress.cs`**
```csharp
public sealed record ThumbnailGenerationProgress(
    long   CurrentMovieId,
    string CurrentMovieLabel,
    int    FramesDone,
    int    FramesTotal,
    int    MoviesCompleted,
    int    MoviesTotal
);
```

---

**WP-003: Extend `Movie` model**

Add to `src/VideoIndexer.Core/Models/Movie.cs`:
```csharp
/// <summary>
/// Current absolute file path on disk (from the most-recently-added
/// <c>movies_filenames</c> row). Null when no filename rows exist.
/// Populated by <c>DapperMovieRepository.GetByIdAsync</c>.
/// </summary>
public string? CurrentFilePath { get; init; }
```

---

**WP-004: New Core interfaces**

Add to `src/VideoIndexer.Core/Abstractions/`:

**`IFfmpegRunner.cs`**
```csharp
/// <summary>
/// Extracts a single frame from a video file at the given millisecond position.
/// Throws <see cref="InvalidOperationException"/> on non-zero FFmpeg exit code.
/// </summary>
public interface IFfmpegRunner
{
    Task ExtractFrameAsync(
        string            videoPath,
        string            outputPath,
        long              positionMs,
        CancellationToken ct = default);
}
```

**`IThumbnailRepository.cs`**
```csharp
public interface IThumbnailRepository
{
    Task<IReadOnlyList<Thumbnail>> GetByMovieIdAsync(long movieId, CancellationToken ct = default);
    Task<IReadOnlyList<long>>      GetIdsByMovieIdAsync(long movieId, CancellationToken ct = default);
    Task<int>                      GetCountByMovieIdAsync(long movieId, CancellationToken ct = default);
    Task<Thumbnail>                InsertAsync(long movieId, long positionMs, bool isCustom = false, CancellationToken ct = default);
    Task                           DeleteAsync(long thumbnailId, CancellationToken ct = default);
    Task                           DeleteAllByMovieIdAsync(long movieId, CancellationToken ct = default);
}
```

**`ICoverImageService.cs`**
```csharp
public interface ICoverImageService
{
    /// <summary>Returns true when cv.cim exists on disk. Synchronous.</summary>
    bool HasCoverImage(string hash);

    /// <summary>Reads the raw JPEG bytes of cv.cim, or null when no cover exists.</summary>
    Task<byte[]?> LoadAsync(string hash, CancellationToken ct = default);

    /// <summary>
    /// Reads cv.cim, applies transform (rotate → crop → colour),
    /// atomically saves back to cv.cim, and returns the saved JPEG bytes.
    /// </summary>
    Task<byte[]> ApplyAndSaveAsync(string hash, CoverImageTransform transform, CancellationToken ct = default);

    /// <summary>
    /// Copies <paramref name="sourcePath"/> to cv.cim, replacing any existing cover.
    /// </summary>
    Task SetFromSourceAsync(string hash, string sourcePath, CancellationToken ct = default);

    /// <summary>
    /// Reads the cv.cim file, applies the given clockwise rotation in memory,
    /// and returns the rotated JPEG bytes without writing to disk.
    /// Returns null when no cover image exists.
    /// </summary>
    Task<byte[]?> RotateForPreviewAsync(string hash, int rotationDegrees, CancellationToken ct = default);
}
```

**`IThumbnailGenerationService.cs`**
```csharp
/// <summary>
/// Generates thumbnails for one or more movies at a given interval.
/// Cancellation returns normally (no OperationCanceledException).
/// Partial frames for the in-progress movie are discarded on cancellation.
/// </summary>
public interface IThumbnailGenerationService
{
    Task<IReadOnlyList<ThumbnailGenerationResult>> GenerateAsync(
        IReadOnlyList<ThumbnailGenerationRequest> requests,
        int                                        intervalSeconds,
        IProgress<ThumbnailGenerationProgress>?   progress          = null,
        CancellationToken                          cancellationToken = default);
}
```

---

**WP-005: Extend `IMovieRepository`**

Add to `src/VideoIndexer.Core/Abstractions/IMovieRepository.cs`:
```csharp
/// <summary>
/// Returns the current absolute file path for each supplied movie ID,
/// or null when no filename row exists for that ID.
/// Uses the most-recently-added movies_filenames row per movie_id.
/// </summary>
Task<IReadOnlyDictionary<long, string?>> GetCurrentFilePathsAsync(
    IReadOnlyList<long> movieIds,
    CancellationToken   ct = default);
```

---

### Phase 2 — Infrastructure

**WP-006: NuGet — `SixLabors.ImageSharp`**

1. Add to `Directory.Packages.props` under a new `<!-- ─── Image processing ──────────────────────────────────────── -->` group:
   ```xml
   <PackageVersion Include="SixLabors.ImageSharp" Version="3.1.9" />
   ```
   (Verify latest stable 3.x on NuGet.org at implementation time.)
2. Add to `src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj`:
   ```xml
   <PackageReference Include="SixLabors.ImageSharp" />
   ```

---

**WP-007: `FfmpegRunner`**

New file: `src/VideoIndexer.Infrastructure/Library/FfmpegRunner.cs`

- Implements `IFfmpegRunner`.
- Constructor: `ISettingsService settingsService, ILogger<FfmpegRunner> logger`.
- Binary resolution (two-step chain):
  1. `ISettingsService.Current.ExternalTools.Ffmpeg.FfmpegPath` (provisioner-populated).
  2. `"ffmpeg"` fallback (PATH lookup).
  Note: a user-override knob via `LibraryOptions.FfmpegPath` is deferred to a future milestone when settings UI is added (`LibraryOptions` has `FfprobePath` but not `FfmpegPath`).
- `ExtractFrameAsync`: build `ProcessStartInfo` using `ArgumentList` only (no `Arguments` string — no shell injection):
  `-y`, `-loglevel`, `panic`, `-nostdin`, `-ss`, `{hh:mm:ss.fff}`, `-i`, `{videoPath}`, `-vframes`, `1`, `-f`, `image2`, `{outputPath}`.
- Await process exit; throw `InvalidOperationException` on non-zero exit code; log warning before throwing.
- `internal static string FormatPosition(long positionMs)` — `TimeSpan.FromMilliseconds(positionMs).ToString(@"hh\:mm\:ss\.fff")`. Marked `internal static` for direct unit-test access.
- Registered as `AddSingleton<IFfmpegRunner, FfmpegRunner>()` in `Program.cs`.

---

**WP-008: `DapperThumbnailRepository`**

New file: `src/VideoIndexer.Infrastructure/Library/DapperThumbnailRepository.cs`

- Implements `IThumbnailRepository`.
- Constructor: `IDbConnectionFactory connectionFactory`.
- `InsertAsync`:
  ```sql
  INSERT INTO movies_thumbnails (milliseconds, movie_id, custom, mosaic, favorite)
  VALUES (@PositionMs, @MovieId, 'no', 'no', 'no');
  SELECT LAST_INSERT_ID();
  ```
  Returns the inserted `Thumbnail` record (select the row back or reconstruct from args + last-insert-id).
- `GetByMovieIdAsync`:
  ```sql
  SELECT thumbnail_id, movie_id, milliseconds, custom, favorite
  FROM movies_thumbnails
  WHERE movie_id = @MovieId
  ORDER BY milliseconds ASC
  ```
  Map `custom = 'yes'` → `IsCustom = true`; `favorite = 'yes'` → `IsFavorite = true`.
- `GetIdsByMovieIdAsync`: `SELECT thumbnail_id FROM movies_thumbnails WHERE movie_id = @MovieId`.
- `GetCountByMovieIdAsync`: `SELECT COUNT(*) FROM movies_thumbnails WHERE movie_id = @MovieId`.
- `DeleteAsync`: `DELETE FROM movies_thumbnails WHERE thumbnail_id = @ThumbnailId`.
- `DeleteAllByMovieIdAsync`: `DELETE FROM movies_thumbnails WHERE movie_id = @MovieId`.
- All methods use `.ConfigureAwait(false)` on every `await`.
- Registered as `AddTransient<IThumbnailRepository, DapperThumbnailRepository>()` in `Program.cs`.

---

**WP-009: `CoverImageService`**

New file: `src/VideoIndexer.Infrastructure/Library/CoverImageService.cs`

- Implements `ICoverImageService`.
- Constructor: `IAppPaths appPaths, ILogger<CoverImageService> logger`.
- `HasCoverImage(hash)`: `File.Exists(_appPaths.CoverImagePath(hash))` (synchronous).
- `LoadAsync(hash, ct)`: Return null if `!HasCoverImage(hash)`. Otherwise `return await File.ReadAllBytesAsync(..., ct).ConfigureAwait(false)`.
- `ApplyAndSaveAsync(hash, transform, ct)`:
  0. `Directory.CreateDirectory(_appPaths.MovieDataDirectory(hash))`.
  1. `using var img = await Image.LoadAsync(coverPath, ct).ConfigureAwait(false)`.
  2. If `transform.RotationDegrees != 0`: `img.Mutate(x => x.Rotate(transform.RotationDegrees))`.
  3. If `transform.Crop is { } crop`: compute square pixel `Rectangle` from fractional values; `img.Mutate(x => x.Crop(rect))`.
  4. `img.Mutate(x => x.Brightness(transform.Brightness).Contrast(transform.Contrast).GammaCorrection(transform.Gamma))`.
  5. `var ms = new MemoryStream(); await img.SaveAsJpegAsync(ms, ct).ConfigureAwait(false); var bytes = ms.ToArray();`
  6. `await File.WriteAllBytesAsync(tmpPath, bytes, ct).ConfigureAwait(false);`
  7. `File.Move(tmpPath, coverPath, overwrite: true);`
  8. `return bytes;`
- `SetFromSourceAsync(hash, sourcePath, ct)`:
  0. `Directory.CreateDirectory(_appPaths.MovieDataDirectory(hash))`.
  `File.Copy(sourcePath, _appPaths.CoverImagePath(hash), overwrite: true)`. Synchronous copy is accepted (same justification as `File.Move` in `ObfuscationService`; single-image file on local disk).
- `RotateForPreviewAsync(hash, rotationDegrees, ct)`:
  1. Return `null` if `!HasCoverImage(hash)`.
  2. `using var img = await Image.LoadAsync(coverPath, ct).ConfigureAwait(false)`.
  3. `img.Mutate(x => x.Rotate(rotationDegrees))`.
  4. `var ms = new MemoryStream(); await img.SaveAsJpegAsync(ms, ct).ConfigureAwait(false);`
  5. `return ms.ToArray();`
- Registered as `AddSingleton<ICoverImageService, CoverImageService>()` in `Program.cs`.

---

**WP-010: `FfmpegThumbnailGeneratorService`**

New file: `src/VideoIndexer.Infrastructure/Library/FfmpegThumbnailGeneratorService.cs`

- Implements `IThumbnailGenerationService`.
- Constructor: `IFfprobeRunner ffprobeRunner, IFfmpegRunner ffmpegRunner, IThumbnailRepository thumbnailRepo, IAppPaths appPaths, ILogger<FfmpegThumbnailGeneratorService> logger`.
- `GenerateAsync(requests, intervalSeconds, progress, ct)`:
  1. For each `request` in order:
     a. Call `ffprobeRunner.ProbeAsync(request.VideoFilePath, ct)`. Parse `format.duration` (float seconds) → compute frame positions: `for (var ms = intervalMs; ms < durationMs; ms += intervalMs)`. On ffprobe failure: add `Error=true` result; skip to next movie.
     b. Enumerate existing thumbnail IDs via `thumbnailRepo.GetIdsByMovieIdAsync` and delete their `.thb2` files; then call `thumbnailRepo.DeleteAllByMovieIdAsync` (this clears prior thumbnails before re-generating).
     c. `Directory.CreateDirectory(appPaths.MovieDataDirectory(request.Hash))`.
     d. For each frame position `ms`:
        - `var thumb = await thumbnailRepo.InsertAsync(request.MovieId, ms, ct: ct).ConfigureAwait(false)`.
        - Derive `thumbPath = appPaths.ThumbnailFilePath(request.Hash, thumb.ThumbnailId)`.
        - Try `await ffmpegRunner.ExtractFrameAsync(request.VideoFilePath, thumbPath, ms, ct).ConfigureAwait(false)`. On `InvalidOperationException`: `await thumbnailRepo.DeleteAsync(thumb.ThumbnailId, ct).ConfigureAwait(false)`; try `File.Delete(thumbPath)`; increment error count.
        - `progress?.Report(new ThumbnailGenerationProgress(...))`.
        - If `ct.IsCancellationRequested`: delete all DB rows + `.thb2` files inserted for the in-progress movie; pass `CancellationToken.None` (not the cancelled `ct`) to all cleanup calls (`thumbnailRepo.DeleteAsync`, `File.Delete`); add `Cancelled=true` result; **break** (do **not** throw `OperationCanceledException`).
  2. Return `IReadOnlyList<ThumbnailGenerationResult>`.
- `internal static string BuildTimecode(long positionMs)` — wraps `FfmpegRunner.FormatPosition` for log messages; accessible in unit tests.
- Registered as `AddSingleton<IThumbnailGenerationService, FfmpegThumbnailGeneratorService>()` in `Program.cs`.

---

**WP-011: `DapperMovieRepository` — extend `GetByIdAsync` + add batch path query**

**`GetByIdAsync` change:** Add a subquery to select the most-recently-added filename:

```sql
LEFT JOIN (
    SELECT hash, filename
    FROM movies_filenames
    WHERE hash = m.hash
    ORDER BY created_at DESC
    LIMIT 1
) mf ON mf.hash = m.hash
```

Map `mf.filename` → `CurrentFilePath` (null if no rows). Update the Dapper row class accordingly.

**`GetCurrentFilePathsAsync`:** Batch query (use correlated subquery for MariaDB ≤ 10.1 compatibility; prefer window function for ≥ 10.2):

```sql
-- Window function variant (MariaDB ≥ 10.2)
SELECT m.movie_id AS MovieId, mf.filename AS FilePath
FROM movies m
LEFT JOIN (
    SELECT hash, filename,
           ROW_NUMBER() OVER (PARTITION BY hash ORDER BY created_at DESC) AS rn
    FROM movies_filenames
) mf ON mf.hash = m.hash AND mf.rn = 1
WHERE m.movie_id IN @MovieIds

-- Correlated subquery fallback
SELECT m.movie_id AS MovieId,
       (SELECT filename FROM movies_filenames WHERE hash = m.hash
        ORDER BY created_at DESC LIMIT 1) AS FilePath
FROM movies m
WHERE m.movie_id IN @MovieIds
```

Returns `IReadOnlyDictionary<long, string?>`.

---

**WP-012: `DapperMovieCatalogRepository` — `HasCoverImage` enrichment**

1. Add **optional** fourth constructor parameter `IAppPaths? appPaths = null`.
2. In `GetMovieListAsync`, after the main SQL query and all existing enrichment passes, add:
   ```csharp
   if (_appPaths is not null)
       rows = rows
           .Select(r => r with { HasCoverImage = File.Exists(_appPaths.CoverImagePath(r.Hash)) })
           .ToList();
   ```
3. The existing SQL literal `0 AS HasCoverImage` remains (provides the default when `_appPaths` is null in tests).
4. Update the `Program.cs` DI registration to pass `sp.GetRequiredService<IAppPaths>()` as the fourth argument.

---

### Phase 3 — App Layer: ViewModels

**WP-013: `ThumbnailItemViewModel`**

New file: `src/VideoIndexer.App/ViewModels/ThumbnailItemViewModel.cs`

- Properties: `long ThumbnailId`, `long PositionMs`, `string FilePath`.
- `[ObservableProperty] Bitmap? _bitmap`.
- `[ObservableProperty] bool _isSelected`.
- `string PositionLabel` computed: `TimeSpan.FromMilliseconds(PositionMs).ToString(@"hh\:mm\:ss")`.
- `LoadBitmapAsync(CancellationToken ct)` — reads file bytes async, creates `new Bitmap(new MemoryStream(bytes))`; disposes previous `Bitmap`.
- `IDisposable` — disposes `Bitmap`.

---

**WP-014: `ThumbnailsViewModel`**

New file: `src/VideoIndexer.App/ViewModels/ThumbnailsViewModel.cs`

- Constructor: `IThumbnailRepository repo, ICoverImageService coverService, IThumbnailViewerService viewerService, IAppPaths appPaths, ISettingsService settingsService, long movieId, string hash`.
- `ObservableCollection<ThumbnailItemViewModel> Thumbnails`.
- `string TabLabel` — `$"Thumbnails ({Thumbnails.Count})"`. Computed property. Subscribe to `Thumbnails.CollectionChanged` in the constructor and call `OnPropertyChanged(nameof(TabLabel))` from the handler (canonical pattern per `constraints.md`).
- `[ObservableProperty] bool _isBusy`.
- `bool HasSelection` — computed from any item's `IsSelected`.
- `IAsyncRelayCommand LoadCommand` — calls `LoadAsync`.
- `IAsyncRelayCommand DeleteSelectedCommand` — guarded by `HasSelection`; for each selected item: `repo.DeleteAsync` + `File.Delete(item.FilePath)`; remove from `Thumbnails`; dispose item.
- `IRelayCommand SelectAllCommand` — sets `IsSelected = true` on all items.
- `IRelayCommand SelectNoneCommand` — sets `IsSelected = false` on all items.
- `IAsyncRelayCommand<ThumbnailItemViewModel> UseThumbnailAsCoverCommand` — calls `coverService.SetFromSourceAsync(hash, item.FilePath, ct)`; raises `CoverImageUpdated`.
- `IRelayCommand<ThumbnailItemViewModel> ViewZoomedCommand` — constructs `ThumbnailViewerViewModel(Thumbnails, indexOf(item))` + calls `viewerService.ShowAsync(vm)`.
- `IRelayCommand<ThumbnailItemViewModel> JumpToPositionCommand` — fires `JumpToVideoPositionRequested` event (`long PositionMs`). Stubbed in M9 (no handler until M10 wires the embedded player).
- `event EventHandler? CoverImageUpdated` — raised after `UseThumbnailAsCoverCommand` succeeds; `CoverImageViewModel` subscribes.
- `LoadAsync(CancellationToken ct)` — calls `repo.GetByMovieIdAsync`; builds `ThumbnailItemViewModel` list; fires `LoadBitmapAsync` per item as fire-and-forget so the grid appears immediately.
- `IDisposable` — `Dispose()` must: cancel any in-flight bitmap load `CancellationTokenSource`; call `item.Dispose()` on each item in `Thumbnails`; clear the collection.

---

**WP-015: `CoverImageViewModel`**

New file: `src/VideoIndexer.App/ViewModels/CoverImageViewModel.cs`

- Constructor: `ICoverImageService coverImageService, string movieHash`.
- `[ObservableProperty] Bitmap? _previewBitmap` — backing store for the display image. The ViewModel creates the `Bitmap` from service-returned bytes (via `new Bitmap(new MemoryStream(bytes))`), disposes the previous bitmap before replacing, and sets `_previewBitmap`.
- `[ObservableProperty] bool _hasCoverImage`.
- `[ObservableProperty] float _brightness = 1.0f`.
- `[ObservableProperty] float _contrast = 1.0f`.
- `[ObservableProperty] float _gamma = 1.0f`.
- `[ObservableProperty] int _rotationDegrees = 0`.
- `[ObservableProperty] float _cropX = 0f`, `_cropY = 0f`, `_cropSize = 1.0f`.
- `[ObservableProperty] bool _showCropMask`.
- `bool CanApply` — `HasCoverImage`.
- `IAsyncRelayCommand LoadCommand` — calls `LoadAsync`.
- `IAsyncRelayCommand ApplyNowCommand` — guarded by `CanApply`; builds `CoverImageTransform` from current property values; calls `coverImageService.ApplyAndSaveAsync` (returns new bytes); creates a new `Bitmap` from the returned bytes, disposes the previous `_previewBitmap`, and sets `PreviewBitmap`; resets pending state.
- `IAsyncRelayCommand RotateCwCommand` — `RotationDegrees = (RotationDegrees + 90) % 360`; calls `await _coverImageService.RotateForPreviewAsync(_hash, RotationDegrees, ct)` and creates a new `Bitmap` from the returned bytes, disposing the previous `_previewBitmap` and setting `PreviewBitmap`.
- `IAsyncRelayCommand RotateCcwCommand` — `RotationDegrees = (RotationDegrees + 270) % 360`; same.
- `IRelayCommand ToggleCropMaskCommand` — toggles `ShowCropMask`.
- `LoadAsync(CancellationToken ct)` — calls `coverImageService.LoadAsync`; creates a new `Bitmap` from the returned bytes, disposes the previous `_previewBitmap`, sets `PreviewBitmap` and `HasCoverImage`.
- `OnCoverImageUpdated()` — public method; called by `ThumbnailsViewModel.CoverImageUpdated` handler to reload the cover image after "Use as Cover Image".

---

**WP-016: `ThumbnailGeneratorMovieViewModel`**

New file: `src/VideoIndexer.App/ViewModels/ThumbnailGeneratorMovieViewModel.cs`

- Properties: `long MovieId`, `string Label`.
- `[ObservableProperty] string _statusText = string.Empty`.
- `[ObservableProperty] int _framesDone = 0`.
- `[ObservableProperty] int _framesTotal = 0`.
- `int ProgressPercent` computed: `FramesTotal > 0 ? (int)(FramesDone * 100.0 / FramesTotal) : 0`.

---

**WP-017: `ThumbnailGeneratorViewModel`**

New file: `src/VideoIndexer.App/ViewModels/ThumbnailGeneratorViewModel.cs`

- Constructor: `IThumbnailGenerationService generationService, IThumbnailRepository thumbnailRepository, IReadOnlyList<ThumbnailGenerationRequest> requests`.
- `ObservableCollection<ThumbnailGeneratorMovieViewModel> Movies` — initialised from `requests`.
- `[ObservableProperty] int _intervalSeconds = 30`.
- `[ObservableProperty] bool _isBusy`.
- `[ObservableProperty] bool _isRegenerateConfirmRequired`.
- `[ObservableProperty] string _summaryText = string.Empty`.
- `bool HasSummary` — computed from `SummaryText`.
- `bool IsMultiMovieMode` — `requests.Count > 1`.
- `private bool _regenerationConfirmed = false` — tracks whether the user has confirmed re-generation in the current execution; reset to `false` after generation completes or is cancelled.
- `IAsyncRelayCommand GenerateCommand` — CanExecute = `!IsBusy`:
  - Check if any target already has thumbnails via `thumbnailRepository.GetCountByMovieIdAsync`.
  - If any count > 0 **and `!_regenerationConfirmed`**: set `IsRegenerateConfirmRequired = true`; return.
  - Build `IProgress<ThumbnailGenerationProgress>` callback that updates the matching `ThumbnailGeneratorMovieViewModel`.
  - Call `generationService.GenerateAsync(requests, IntervalSeconds, progress, cts.Token)`.
  - Build `SummaryText` from the returned results (total generated, failed, cancelled movies).
  - Reset `_regenerationConfirmed = false`.
- `IRelayCommand ConfirmRegenerateCommand` — sets `_regenerationConfirmed = true`; sets `IsRegenerateConfirmRequired = false`; calls `GenerateCommand.ExecuteAsync(null)`.
- `IRelayCommand CancelCommand` — cancels the internal `CancellationTokenSource`.
- `event EventHandler? CloseRequested`.
- `IRelayCommand CloseCommand` — fires `CloseRequested`.

---

**WP-018: `ThumbnailViewerViewModel`**

New file: `src/VideoIndexer.App/ViewModels/ThumbnailViewerViewModel.cs`

- Constructor: `IReadOnlyList<ThumbnailItemViewModel> thumbnails, int initialIndex`.
- `[ObservableProperty] int _currentIndex` — initialised from `initialIndex`.
- `ThumbnailItemViewModel? Current` — computed from `_currentIndex`.
- `bool CanNavigatePrev` — `CurrentIndex > 0`.
- `bool CanNavigateNext` — `CurrentIndex < thumbnails.Count - 1`.
- `string PositionLabel` — `$"{CurrentIndex + 1} / {thumbnails.Count}"`.
- `IRelayCommand PrevCommand` — guarded by `CanNavigatePrev`; decrements `CurrentIndex`.
- `IRelayCommand NextCommand` — guarded by `CanNavigateNext`; increments `CurrentIndex`.
- `event EventHandler? CloseRequested`.
- `IRelayCommand CloseCommand` — fires `CloseRequested`.

---

**WP-019: `MoviePropertiesViewModel` ffprobe wiring**

Modify `src/VideoIndexer.App/ViewModels/MoviePropertiesViewModel.cs`:

1. Add optional constructor parameter: `IFfprobeRunner? ffprobeRunner = null`. Note: `IMovieRepository? movieRepo` is removed — the file path is obtained directly from `_movie.CurrentFilePath` (populated by the `GetByIdAsync` LEFT JOIN added in WP-011).
2. Replace the four computed-property `"—"` stubs with `[ObservableProperty]` backing fields:
   ```csharp
   [ObservableProperty] private string _fileType   = "—";
   [ObservableProperty] private string _resolution = "—";
   [ObservableProperty] private string _duration   = "—";
   [ObservableProperty] private string _bitrate    = "—";
   [ObservableProperty] private bool   _isLoadingMediaInfo;
   ```
3. Add `Task LoadMediaInfoAsync(CancellationToken ct = default)`:
   - Guard: `if (_ffprobeRunner is null) return`.
   - `IsLoadingMediaInfo = true`.
   - `var filePath = _movie.CurrentFilePath`. If null, return.
   - `var json = await _ffprobeRunner.ProbeAsync(filePath, ct).ConfigureAwait(false)`. Parse via `System.Text.Json.JsonDocument`:
     - `FileType`: `streams[0].codec_long_name`.
     - `Resolution`: `streams[0].width` × `streams[0].height` → `"1920×1080"`.
     - `Duration`: `format.duration` (float seconds) → `TimeSpan.FromSeconds(d).ToString(@"hh\:mm\:ss")`.
     - `Bitrate`: `format.bit_rate` (bps) → `"N.N Mbps"`.
   - Catch all exceptions; silently leave `"—"` for unparsable fields.
   - `IsLoadingMediaInfo = false`.
4. Modify `src/VideoIndexer.App/Services/AvaloniaMoviePropertiesService.cs`: after showing the dialog window (fire-and-forget), call `_ = vm.LoadMediaInfoAsync(CancellationToken.None)` (wrapped in `try/catch` to prevent unobserved-exception crash; on failure set `vm.Duration = "Error"` silently (the inner `LoadMediaInfoAsync` catch already handles field-level protection)).

---

### Phase 4 — App Layer: Services

**WP-020: `IThumbnailViewerService` + `AvaloniaThumbnailViewerService`**

New file `src/VideoIndexer.App/Services/IThumbnailViewerService.cs`:
```csharp
public interface IThumbnailViewerService
{
    Task ShowAsync(ThumbnailViewerViewModel viewModel, CancellationToken ct = default);
}
```

New file `src/VideoIndexer.App/Services/AvaloniaThumbnailViewerService.cs`:
- Constructs `ThumbnailViewerView` directly (not via ViewLocator).
- Modal `Window`, 800×600, non-resizable.
- Sets `window.DataContext = viewModel`.
- Wires `viewModel.CloseRequested` → `window.Close()` in `try/finally` block (unsubscribe on finally).
- Registered as `AddSingleton<IThumbnailViewerService, AvaloniaThumbnailViewerService>()` in `Program.cs`.

---

**WP-021: `IThumbnailGenerationDialogService` + `AvaloniaThumbnailGenerationDialogService`**

New file `src/VideoIndexer.App/Services/IThumbnailGenerationDialogService.cs`:
```csharp
public interface IThumbnailGenerationDialogService
{
    Task ShowAsync(ThumbnailGeneratorViewModel viewModel, CancellationToken ct = default);
}
```

New file `src/VideoIndexer.App/Services/AvaloniaThumbnailGenerationDialogService.cs`:
- Constructs `ThumbnailGeneratorView` directly.
- Modal `Window`, 640×480, resizable.
- Sets `window.DataContext = viewModel`.
- Wires `viewModel.CloseRequested` → `window.Close()` in `try/finally` block.
- Registered as `AddSingleton<IThumbnailGenerationDialogService, AvaloniaThumbnailGenerationDialogService>()` in `Program.cs`.

---

### Phase 5 — App Layer: Views

**WP-023: `CoverImageCropperControl`**

New files: `src/VideoIndexer.App/Views/CoverImageCropperControl.axaml` / `.axaml.cs`

- `UserControl` with a `Grid` root containing an `Image` and a `Canvas` overlay.
- Styled properties: `Source (Bitmap?)`, `CropX (float)`, `CropY (float)`, `CropSize (float)`, `ShowMask (bool)`.
- `Source` property-changed handler sets `Image.Source = value` directly.
- The `Canvas` overlay renders:
  - When `ShowMask = true`: four semi-transparent dark `Rectangle` elements covering the area outside the crop region.
  - A light-bordered `Rectangle` for the crop region boundary (always visible when a cover image is loaded).
- Pointer event handlers in code-behind: `PointerPressed` → begin drag; `PointerMoved` → update `CropX`/`CropY` (constrained to `[0, 1 - CropSize]`); `PointerReleased` → commit.
- Coordinate math: normalise pointer position relative to the image's rendered `Bounds`; clamp to valid range. Extract as a `internal static` pure method for unit testability.
- `SetCurrentValue` used for `CropX`/`CropY` so compiled bindings back-propagate to the ViewModel.
- Not registered in DI — embedded directly in `CoverImageView`.

---

**WP-024: `CoverImageView`**

New files: `src/VideoIndexer.App/Views/CoverImageView.axaml` / `.axaml.cs`

- `DockPanel` root, `x:DataType="vm:CoverImageViewModel"`.
- **Bottom toolbar** (`Dock=Bottom`): Rotate CW, Rotate CCW, Show/Hide Crop Mask toggle, separator, Apply Now (bound to `ApplyNowCommand`, disabled when `!CanApply`, shows activity indicator when `IsBusy`).
- **Right panel** (`Dock=Right`): `StackPanel` with three labelled sliders: Brightness (0.0–2.0), Contrast (0.0–2.0), Gamma (0.1–5.0). Sliders update VM properties only — no live image processing.
- **Bottom error bar** (`Dock=Bottom`, below toolbar): `Border` visible when `HasError`.
- **Centre**: `Grid` containing:
  - `CoverImageCropperControl` bound to `PreviewBitmap` (as `Source`), `CropX`, `CropY`, `CropSize`, `ShowCropMask` — visible when `HasCoverImage`.
  - `TextBlock "No cover image. Use the Thumbnails tab to set one."` visible when `!HasCoverImage`.
- Not ViewLocator-registered — embedded directly in `MovieEditorView`.

---

**WP-025: `ThumbnailsView`**

New files: `src/VideoIndexer.App/Views/ThumbnailsView.axaml` / `.axaml.cs`

- `DockPanel` root, `x:DataType="vm:ThumbnailsViewModel"`.
- **Top toolbar** (`Dock=Top`): "Delete Selected" (`DeleteSelectedCommand`, disabled when `!HasSelection`), "Select All" (`SelectAllCommand`), "Select None" (`SelectNoneCommand`), separator, `TextBlock` bound to `TabLabel`.
- **Centre**: `ListBox` with `SelectionMode="Multiple"` (gives Ctrl+Click natively), `ItemsSource="{Binding Thumbnails}"`, `WrapPanel` as `ItemsPanel`. Item template: `Border` wrapping `Image Source="{Binding Bitmap}"` sized from `ThumbnailsOptions.ThumbnailSize` (passed into `ThumbnailsViewModel` via `ISettingsService`). `ContextMenu` items: "Use as Cover Image" (`UseThumbnailAsCoverCommand`, `CommandParameter={Binding}`), "View Zoomed" (`ViewZoomedCommand`, `CommandParameter={Binding}`), "Jump to Video Position" (disabled stub until M10), "Delete" (per-item: sets `IsSelected = true` then calls `DeleteSelectedCommand`).
- Not ViewLocator-registered.

---

**WP-026: `ThumbnailGeneratorView`**

New files: `src/VideoIndexer.App/Views/ThumbnailGeneratorView.axaml` / `.axaml.cs`

- `DockPanel` root, `x:DataType="vm:ThumbnailGeneratorViewModel"`.
- **Top** (`Dock=Top`): `NumericUpDown` (interval seconds, bound to `IntervalSeconds`) + label "seconds between frames".
- **Centre**: 
  - Regenerate confirmation panel — `Border` visible when `IsRegenerateConfirmRequired`: warning text + "Confirm & Regenerate" button (`ConfirmRegenerateCommand`).
  - Progress area — visible when `IsBusy`: `ItemsControl` over `Movies` showing `ThumbnailGeneratorMovieViewModel` rows (label, `ProgressBar`, frame counter).
  - Summary panel — visible when `HasSummary`: `TextBlock` bound to `SummaryText`.
- **Bottom toolbar** (`Dock=Bottom`): Generate (`GenerateCommand`, disabled while `IsBusy`), Cancel (`CancelCommand`, disabled when `!IsBusy`), Close (`CloseCommand`).
- Not ViewLocator-registered.

---

**WP-027: `ThumbnailViewerView`**

New files: `src/VideoIndexer.App/Views/ThumbnailViewerView.axaml` / `.axaml.cs`

- `DockPanel` root, `x:DataType="vm:ThumbnailViewerViewModel"`.
- **Top** (`Dock=Top`): `TextBlock` bound to `PositionLabel` (e.g. "3 / 24") + position timestamp (`Current.PositionLabel`).
- **Centre**: `Image Source="{Binding Current.Bitmap}"` with `Stretch="Uniform"`.
- **Bottom toolbar** (`Dock=Bottom`): Previous (`PrevCommand`, `IsEnabled="{Binding CanNavigatePrev}"`), Next (`NextCommand`, `IsEnabled="{Binding CanNavigateNext}"`), Close (`CloseCommand`).
- Not ViewLocator-registered.

---

### Phase 6 — Movie Editor Wiring

**WP-028: `MovieEditorViewModel` update**

Modify `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs`:

1. Add optional constructor parameters: `IThumbnailRepository? thumbnailRepo = null`, `ICoverImageService? coverImageService = null`, `IThumbnailGenerationService? generationService = null`, `IThumbnailViewerService? thumbnailViewerService = null`, `IFfprobeRunner? ffprobeRunner = null`. (`IMovieRepository` and `IAppPaths` are already present.)
2. Add observable properties: `CoverImageViewModel? CoverImageVm`, `ThumbnailsViewModel? ThumbnailsVm`, `ThumbnailGeneratorViewModel? GeneratorVm`.
3. `ThumbnailTabHeader` — computed string `$"Thumbnails ({count})"`, updated when `ThumbnailsVm.Thumbnails.Count` changes.
4. In `LoadAsync` (after `_movie` is loaded):
   - Store `_currentFilePath = _movie.CurrentFilePath`.
   - If all required image services are non-null:
     - `ThumbnailsVm = new ThumbnailsViewModel(thumbnailRepo, coverImageService, thumbnailViewerService, appPaths, _settingsService, _movie.MovieId, _movie.Hash)`.
     - `CoverImageVm = new CoverImageViewModel(coverImageService, _movie.Hash)`.
     - `GeneratorVm` = (if `_currentFilePath != null`) `new ThumbnailGeneratorViewModel(generationService, thumbnailRepo, [new ThumbnailGenerationRequest(...)])`.
     - Subscribe `ThumbnailsVm.CoverImageUpdated` → `CoverImageVm.OnCoverImageUpdated()`.
     - Subscribe `ThumbnailsVm.Thumbnails.CollectionChanged` → update `ThumbnailTabHeader`.
     - `await ThumbnailsVm.LoadCommand.ExecuteAsync(null).ConfigureAwait(false)`.
     - `await CoverImageVm.LoadCommand.ExecuteAsync(null).ConfigureAwait(false)`.
   - Dispose previous sub-VMs before replacing (if they implement `IDisposable`).
5. `ShowPropertiesCommand`: Obtain `originalFilename` by awaiting `_movieRepository?.GetOriginalFilenameAsync(_movie.MovieId, ct)` (as in the existing implementation). Construct `new MoviePropertiesViewModel(_movie, _appPaths, _fileLauncherService, filePath: _currentFilePath, fileSizeBytes: null, originalFilename: originalFilename, ffprobeRunner: _ffprobeRunner)`. Call `_propertiesService.ShowAsync(vm)`. `MovieEditorViewModel` forwards `_ffprobeRunner` as the `ffprobeRunner` optional named parameter when constructing `MoviePropertiesViewModel`. (The `AvaloniaMoviePropertiesService` fires `LoadMediaInfoAsync` fire-and-forget after showing.) `IMovieRepository` is not forwarded to the VM — `MoviePropertiesViewModel` reads `_movie.CurrentFilePath` directly for media info.
6. Update the `Func<Movie, MovieEditorViewModel>` factory lambda in `Program.cs` to forward the five new optional services.

---

**WP-029: `MovieEditorView.axaml` update**

Replace the stub `TabItem` content in `src/VideoIndexer.App/Views/MovieEditorView.axaml`:

```xml
<TabItem Header="Cover Image">
    <views:CoverImageView DataContext="{Binding CoverImageVm}" />
</TabItem>

<TabItem Header="{Binding ThumbnailTabHeader}">
    <views:ThumbnailsView DataContext="{Binding ThumbnailsVm}" />
</TabItem>

<TabItem Header="Generate">
    <views:ThumbnailGeneratorView DataContext="{Binding GeneratorVm}" />
</TabItem>
```

The "Video" `TabItem` remains the M10 stub. The `x:DataType="vm:MovieEditorViewModel"` on `MovieEditorView` ensures compiled bindings resolve `ThumbnailTabHeader` and sub-VM properties.

---

### Phase 7 — Movies List Cover Panel and Multi-Movie Generator

**WP-030: `MoviesListViewModel` updates**

Modify `src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs`:

1. Add optional constructor parameters: `IAppPaths? appPaths = null`, `IMovieRepository? movieRepository = null`, `IThumbnailGenerationService? generationService = null`, `IThumbnailRepository? thumbnailRepo = null`, `IThumbnailGenerationDialogService? generatorDialogService = null`.
2. Add `[ObservableProperty] Bitmap? _coverImageBitmap`.
3. Change `SelectedMovies` from a plain auto-property (`public IList<MovieListItem> SelectedMovies { get; set; }`) to an explicit property with a setter. The setter: (a) assigns `_selectedMovies = value`; (b) cancels any previous cover-load `CancellationTokenSource`; (c) invokes the fire-and-forget cover-image load helper if single selection and `HasCoverImage == true` and `_appPaths is not null`; (d) sets `CoverImageBitmap = null` when selection is empty, multi-select, or `HasCoverImage == false`. No code-behind changes required — `MoviesListView.axaml.cs` already assigns `vm.SelectedMovies` and this setter provides the notification hook.
4. Implement `GenerateThumbnailsAsync(CancellationToken ct)`:
   ```csharp
   var movieIds = SelectedMovies.Select(m => m.MovieId).ToList();
   var paths    = await _movieRepository!.GetCurrentFilePathsAsync(movieIds, ct).ConfigureAwait(false);
   var requests = SelectedMovies
       .Select(m => new ThumbnailGenerationRequest(
           m.MovieId, m.Hash,
           paths.GetValueOrDefault(m.MovieId) ?? string.Empty,
           m.Label))
       .Where(r => !string.IsNullOrEmpty(r.VideoFilePath))
       .ToList();
   if (requests.Count == 0) return;
   var vm = new ThumbnailGeneratorViewModel(_generationService, _thumbnailRepo, requests);
   await _generatorDialogService!.ShowAsync(vm, ct).ConfigureAwait(false);
   await LoadAsync(ct).ConfigureAwait(false); // refresh ThumbnailCount in list
   ```
   `CanGenerateThumbnails()` — `SelectedMovies.Count > 0 && _generatorDialogService != null`.
   Note: `_generationService` and `_thumbnailRepo` must be added as optional constructor parameters (alongside `_generatorDialogService`).
5. Update the `MoviesListViewModel` factory lambda in `Program.cs` to pass `IAppPaths`, `IMovieRepository`, `IThumbnailGenerationService`, `IThumbnailRepository`, and `IThumbnailGenerationDialogService`.

---

**WP-031: `MoviesListView.axaml` cover panel update**

Modify `src/VideoIndexer.App/Views/MoviesListView.axaml`:

Replace the placeholder `TextBlock "Cover image&#x0A;preview"` inside the cover panel `Border` with:
```xml
<Grid>
  <Image Source="{Binding CoverImageBitmap}"
         Stretch="Uniform"
         IsVisible="{Binding CoverImageBitmap, Converter={x:Static converters:ObjectConverters.IsNotNull}}" />
  <TextBlock Text="No cover"
             IsVisible="{Binding CoverImageBitmap, Converter={x:Static converters:ObjectConverters.IsNull}}"
             HorizontalAlignment="Center"
             VerticalAlignment="Center"
             Foreground="Gray" />
</Grid>
```

Ensure `GenerateThumbnailsCommand` binding on the existing context menu item is updated from `ReflectionBinding` to a compiled binding (`x:DataType` on `MoviesListView`).

---

### Phase 8 — DI Wiring

**WP-032: `Program.cs` registrations**

Add the following in a new `// ── M9 Image Management ──────────────────────────────────────────` block:

```csharp
// Infrastructure
services.AddSingleton<IFfmpegRunner,                FfmpegRunner>();
services.AddTransient<IThumbnailRepository,          DapperThumbnailRepository>();
services.AddSingleton<ICoverImageService,            CoverImageService>();
services.AddSingleton<IThumbnailGenerationService,   FfmpegThumbnailGeneratorService>();

// App services
services.AddSingleton<IThumbnailViewerService,       AvaloniaThumbnailViewerService>();
services.AddSingleton<IThumbnailGenerationDialogService, AvaloniaThumbnailGenerationDialogService>();

// Views
services.AddTransient<CoverImageView>(_ => new CoverImageView());
services.AddTransient<ThumbnailsView>(_ => new ThumbnailsView());
services.AddTransient<ThumbnailGeneratorView>(_ => new ThumbnailGeneratorView());
services.AddTransient<ThumbnailViewerView>(_ => new ThumbnailViewerView());
```

Update existing registrations:
- `DapperMovieCatalogRepository` transient registration: pass `sp.GetRequiredService<IAppPaths>()` as the fourth constructor argument.
- `MovieEditorViewModel` factory lambda: forward `IFfmpegRunner`, `IThumbnailRepository`, `ICoverImageService`, `IThumbnailGenerationService`, `IThumbnailViewerService`.
- `MoviesListViewModel` factory lambda: forward `IAppPaths`, `IMovieRepository`, `IThumbnailGenerationService`, `IThumbnailRepository`, `IThumbnailGenerationDialogService`.

---

### Phase 9 — Tests

**WP-033: Infrastructure unit tests (`VideoIndexer.Tests`)**

New test files:

- `FfmpegRunnerTests.cs`:
  - `FormatPosition_ZeroMs_Returns_00_00_00_000` — 0 ms → `"00:00:00.000"`.
  - `FormatPosition_90500Ms_Returns_00_01_30_500` — 90 500 ms → `"00:01:30.500"`.

- `FfmpegThumbnailGeneratorServiceTests.cs`:
  - `BuildTimecode_90500_FormatsCorrectly` — `BuildTimecode(90500)` → `"00:01:30.500"`.

- `CoverImageServiceTests.cs`:
  - `HasCoverImage_ReturnsFalse_WhenFileNotPresent` — temp directory, no file.
  - `HasCoverImage_ReturnsTrue_WhenFilePresent` — write a stub `cv.cim`.
  - `SetFromSourceAsync_CopiesFile_ToDestination` — verify destination file exists with source bytes.

- `DapperMovieCatalogRepositoryHasCoverImageTests.cs`:
  - `GetMovieListAsync_SetsHasCoverImageFalse_WhenFileAbsent` — inject `FakeAppPaths` pointing at empty temp dir.
  - `GetMovieListAsync_SetsHasCoverImageTrue_WhenCvCimPresent` — create `cv.cim` in temp data dir.

Integration tests (`VideoIndexer.Infrastructure.Tests` — self-skip when no DB):
- `DapperThumbnailRepositoryTests.cs`:
  - `InsertAsync_ReturnsAssignedId`.
  - `GetByMovieId_ReturnsThumbnailsOrderedByMilliseconds`.
  - `Delete_RemovesRow`.
  - `DeleteAll_RemovesAllRowsForMovie`.

---

**WP-034: App layer unit tests (`VideoIndexer.App.Tests`)**

New test fakes in `tests/VideoIndexer.App.Tests/TestHelpers/`:
- `FakeThumbnailRepository.cs` — in-memory `IThumbnailRepository` (dictionary-backed).
- `FakeCoverImageService.cs` — configurable `ICoverImageService`; tracks `SetFromSourceAsync` call count and arguments.
- `FakeThumbnailGenerationService.cs` — returns preset `IReadOnlyList<ThumbnailGenerationResult>`; captures call arguments.
- `FakeThumbnailViewerService.cs` — call-tracking `IThumbnailViewerService`.
- `FakeThumbnailGenerationDialogService.cs` — call-tracking `IThumbnailGenerationDialogService`.

New test files:

- `CoverImageViewModelTests.cs`:
  - `LoadAsync_SetsHasCoverImage_WhenFileExists`.
  - `LoadAsync_SetsHasCoverImageFalse_WhenNoCover`.
  - `ApplyNowCommand_CallsServiceWithCorrectTransform` — verifies `CropX`/`CropY`/`CropSize`, `RotationDegrees`, and brightness/contrast/gamma values are forwarded correctly.
  - `RotateCwCommand_IncrementsBy90`.
  - `RotateCcwCommand_DecrementsBy90_EquivalentToCwThreeTimes`.
  - `RotateCw_FourTimes_ReturnsToZero`.
  - `CanApply_False_WhenNoCoverImage`.
  - `OnCoverImageUpdated_TriggersReload`.

- `ThumbnailsViewModelTests.cs`:
  - `LoadAsync_PopulatesThumbnails_FromRepository`.
  - `LoadAsync_EmptyList_WhenNoThumbnails`.
  - `SelectAllCommand_SelectsAll`.
  - `SelectNoneCommand_DeselectsAll`.
  - `DeleteSelectedCommand_CallsDeleteOnRepoAndRemovesFromCollection`.
  - `UseThumbnailAsCoverCommand_CallsCoverService_AndFiresUpdatedEvent`.
  - `ViewZoomedCommand_CallsViewerService`.
  - `TabLabel_ReflectsThumbnailCount`.
  - Note: All `ThumbnailsViewModel` constructions in this test file must pass a `FakeSettingsService` instance (new required `ISettingsService settingsService` constructor parameter added in WP-014) — the existing test fakes (`FakeThumbnailRepository.cs` etc.) do not need updating.

- `ThumbnailGeneratorViewModelTests.cs`:
  - `GenerateCommand_CallsServiceWithSingleRequest`.
  - `GenerateCommand_UpdatesMovieProgress_ViaProgressCallback`.
  - `GenerateCommand_SetsSummaryText_OnCompletion`.
  - `GenerateCommand_ShowsConfirmPanel_WhenMovieAlreadyHasThumbnails`.
  - `ConfirmRegenerateCommand_SetsConfirmedAndRerunsGenerate`.
  - `CancelCommand_CancelsToken`.
  - `IsMultiMovieMode_True_ForMultipleRequests`.

- `ThumbnailViewerViewModelTests.cs`:
  - `CurrentIndex_InitialisedFromConstructor`.
  - `PrevCommand_DecrementsIndex`.
  - `NextCommand_IncrementsIndex`.
  - `CanNavigatePrev_False_AtFirstIndex`.
  - `CanNavigateNext_False_AtLastIndex`.
  - `CloseCommand_FiresCloseRequestedEvent`.

- `MoviePropertiesViewModelTests.cs` (additions):
  - `LoadMediaInfoAsync_SetsAllFourFields_WhenFfprobeSucceeds` — mock runner returning valid JSON; verify `Duration`, `Resolution`, `FileType`, `Bitrate` are populated.
  - `LoadMediaInfoAsync_LeavesDefaultDash_WhenCurrentFilePathIsNull` — tests `_movie.CurrentFilePath == null`.
  - `LoadMediaInfoAsync_LeavesDefaultDash_WhenRunnerIsNull`.
  - `LoadMediaInfoAsync_LeavesDefaultDash_OnFfprobeException`.

- `MoviesListViewModelTests.cs` (additions):
  - `GenerateThumbnailsCommand_CannotExecute_WithNoSelection`.
  - `GenerateThumbnailsCommand_CallsDialogService_WithResolvedRequests`.

---

### Phase 10 — Manifest Updates

**WP-035: Project manifest documents**

- `docs/agents/project-manifest/api-surface.md` — add all new interface signatures (`IFfmpegRunner`, `IThumbnailRepository`, `ICoverImageService`, `IThumbnailGenerationService`); add all new model record shapes; add `IAppPaths` new methods; update `IMovieRepository` with `GetCurrentFilePathsAsync`; update `Movie.CurrentFilePath`; add all new ViewModel constructors + key property/command signatures; add `IThumbnailViewerService`, `IThumbnailGenerationDialogService`; update `MovieEditorViewModel`, `MoviePropertiesViewModel`, `MoviesListViewModel`.
- `docs/agents/project-manifest/file-tree.md` — annotate all new and modified source files.
- `docs/agents/project-manifest/constraints.md` — add: (1) `File.Exists` sync call in `DapperMovieCatalogRepository` post-load enrichment is accepted (no async BCL overload); (2) `FfmpegThumbnailGeneratorService.GenerateAsync` returns normally on cancellation (no `OperationCanceledException`); (3) `CoverImageService` atomic save pattern (`.tmp` + `File.Move`); (4) thumbnail filename convention `{thumbnail_id}.thb2`; (5) cover image filename convention `cv.cim`; (6) `FfmpegRunner` binary resolution chain mirrors `FfprobeRunner`; (7) `SixLabors.ImageSharp` is Infrastructure-only — Core must not reference it; (8) `HasCoverImage` is always derived by `File.Exists` post-query (never stored in DB).
- `docs/agents/project-manifest/tech-stack.md` — add `SixLabors.ImageSharp 3.1.9` under Infrastructure NuGet table (note: pure managed .NET, Infrastructure-only, cover image crop/rotate/colour).
- `docs/projects/rebuild/milestones/m9-images.md` — create milestone summary document (status, deliverables, work packages list, schema revision note — no new migration for M9, metrics).


## Dependencies

- M6 (Movie Editor shell, `Movie` model, `MovieEditorViewModel`, `MoviePropertiesViewModel` stubs) — complete.
- M7 (Tagging Core) — complete; `ITagsManager` already in `DapperMovieCatalogRepository`.
- M8 (System Tools) — complete; `ThumbnailsOptions.ThumbnailSize` from `AppOptions`; `AppPaths.MovieDataDirectory(hash)` exists and creates the data folder.
- `IFfprobeRunner` / `FfprobeRunner` functional (M2/M3 foundation) — complete; re-used for duration extraction in `FfmpegThumbnailGeneratorService` and media info in `MoviePropertiesViewModel`.
- `movies_thumbnails` DB table — exists at schema revision 41; **no new migration required**.


## Required Components

### New Files

| File | Type |
|---|---|
| `src/VideoIndexer.Core/Abstractions/IFfmpegRunner.cs` | New interface |
| `src/VideoIndexer.Core/Abstractions/IThumbnailRepository.cs` | New interface |
| `src/VideoIndexer.Core/Abstractions/ICoverImageService.cs` | New interface |
| `src/VideoIndexer.Core/Abstractions/IThumbnailGenerationService.cs` | New interface |
| `src/VideoIndexer.Core/Models/Thumbnail.cs` | New sealed record |
| `src/VideoIndexer.Core/Models/CropRect.cs` | New sealed record |
| `src/VideoIndexer.Core/Models/CoverImageTransform.cs` | New sealed record |
| `src/VideoIndexer.Core/Models/ThumbnailGenerationRequest.cs` | New sealed record |
| `src/VideoIndexer.Core/Models/ThumbnailGenerationResult.cs` | New sealed record |
| `src/VideoIndexer.Core/Models/ThumbnailGenerationProgress.cs` | New sealed record |
| `src/VideoIndexer.Infrastructure/Library/FfmpegRunner.cs` | New implementation |
| `src/VideoIndexer.Infrastructure/Library/DapperThumbnailRepository.cs` | New implementation |
| `src/VideoIndexer.Infrastructure/Library/CoverImageService.cs` | New implementation |
| `src/VideoIndexer.Infrastructure/Library/FfmpegThumbnailGeneratorService.cs` | New implementation |
| `src/VideoIndexer.App/Services/IThumbnailViewerService.cs` | New dialog service interface |
| `src/VideoIndexer.App/Services/AvaloniaThumbnailViewerService.cs` | New dialog service impl |
| `src/VideoIndexer.App/Services/IThumbnailGenerationDialogService.cs` | New dialog service interface |
| `src/VideoIndexer.App/Services/AvaloniaThumbnailGenerationDialogService.cs` | New dialog service impl |
| `src/VideoIndexer.App/ViewModels/ThumbnailItemViewModel.cs` | New ViewModel |
| `src/VideoIndexer.App/ViewModels/ThumbnailsViewModel.cs` | New ViewModel (`IDisposable`) |
| `src/VideoIndexer.App/ViewModels/CoverImageViewModel.cs` | New ViewModel |
| `src/VideoIndexer.App/ViewModels/ThumbnailGeneratorMovieViewModel.cs` | New ViewModel |
| `src/VideoIndexer.App/ViewModels/ThumbnailGeneratorViewModel.cs` | New ViewModel |
| `src/VideoIndexer.App/ViewModels/ThumbnailViewerViewModel.cs` | New ViewModel |
| `src/VideoIndexer.App/Views/CoverImageCropperControl.axaml` / `.axaml.cs` | New custom UserControl |
| `src/VideoIndexer.App/Views/CoverImageView.axaml` / `.axaml.cs` | New View |
| `src/VideoIndexer.App/Views/ThumbnailsView.axaml` / `.axaml.cs` | New View |
| `src/VideoIndexer.App/Views/ThumbnailGeneratorView.axaml` / `.axaml.cs` | New View |
| `src/VideoIndexer.App/Views/ThumbnailViewerView.axaml` / `.axaml.cs` | New View |
| `tests/VideoIndexer.Tests/FfmpegRunnerTests.cs` | New unit tests |
| `tests/VideoIndexer.Tests/FfmpegThumbnailGeneratorServiceTests.cs` | New unit tests |
| `tests/VideoIndexer.Tests/CoverImageServiceTests.cs` | New unit tests |
| `tests/VideoIndexer.Tests/DapperMovieCatalogRepositoryHasCoverImageTests.cs` | New unit tests |
| `tests/VideoIndexer.App.Tests/TestHelpers/FakeThumbnailRepository.cs` | New test fake |
| `tests/VideoIndexer.App.Tests/TestHelpers/FakeCoverImageService.cs` | New test fake |
| `tests/VideoIndexer.App.Tests/TestHelpers/FakeThumbnailGenerationService.cs` | New test fake |
| `tests/VideoIndexer.App.Tests/TestHelpers/FakeThumbnailViewerService.cs` | New test fake |
| `tests/VideoIndexer.App.Tests/TestHelpers/FakeThumbnailGenerationDialogService.cs` | New test fake |
| `tests/VideoIndexer.App.Tests/CoverImageViewModelTests.cs` | New unit tests |
| `tests/VideoIndexer.App.Tests/ThumbnailsViewModelTests.cs` | New unit tests |
| `tests/VideoIndexer.App.Tests/ThumbnailGeneratorViewModelTests.cs` | New unit tests |
| `tests/VideoIndexer.App.Tests/ThumbnailViewerViewModelTests.cs` | New unit tests |
| `docs/projects/rebuild/milestones/m9-images.md` | New milestone summary |

### Modified Files

| File | Change |
|---|---|
| `src/VideoIndexer.Core/Abstractions/IAppPaths.cs` | +2 path helper methods |
| `src/VideoIndexer.Core/Abstractions/IMovieRepository.cs` | +`GetCurrentFilePathsAsync` |
| `src/VideoIndexer.Core/Models/Movie.cs` | +`CurrentFilePath string?` |
| `src/VideoIndexer.Infrastructure/AppPaths.cs` | +2 method implementations |
| `src/VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs` | +optional `IAppPaths?` param; +`HasCoverImage` enrichment pass |
| `src/VideoIndexer.Infrastructure/Library/DapperMovieRepository.cs` | +LEFT JOIN for `CurrentFilePath`; +`GetCurrentFilePathsAsync` |
| `src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj` | +`SixLabors.ImageSharp` PackageReference |
| `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs` | +5 optional service params; +3 child VM properties + `ThumbnailTabHeader`; +child-VM init in `LoadAsync`; +`ShowPropertiesCommand` ffprobe forwarding |
| `src/VideoIndexer.App/ViewModels/MoviePropertiesViewModel.cs` | +`IFfprobeRunner?` param; +`LoadMediaInfoAsync` (reads `_movie.CurrentFilePath` directly); convert 4 stub properties to `[ObservableProperty]` |
| `src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs` | +optional `IAppPaths?`, `IMovieRepository?`, `IThumbnailGenerationService?`, `IThumbnailRepository?`, `IThumbnailGenerationDialogService?` params; +`CoverImageBitmap`; +`GenerateThumbnailsAsync` implementation |
| `src/VideoIndexer.App/Services/AvaloniaMoviePropertiesService.cs` | Fire-and-forget `vm.LoadMediaInfoAsync(CancellationToken.None)` after `dialog.ShowDialog(owner)` |
| `src/VideoIndexer.App/Views/MovieEditorView.axaml` | Replace Cover Image + Thumbnails stubs; add Generate tab |
| `src/VideoIndexer.App/Views/MoviesListView.axaml` | Replace cover panel placeholder; update context menu binding |
| `src/VideoIndexer.App/Program.cs` | +9 new DI entries; updated factory lambdas for 4 existing registrations |
| `tests/VideoIndexer.App.Tests/MoviePropertiesViewModelTests.cs` | +4 new test cases for media info loading |
| `tests/VideoIndexer.App.Tests/MoviesListViewModelTests.cs` | +2 new test cases for generator command |
| `Directory.Packages.props` | +`SixLabors.ImageSharp 3.1.9` version entry |
| `docs/agents/project-manifest/api-surface.md` | Add all new types and updated signatures |
| `docs/agents/project-manifest/file-tree.md` | Annotate all new and modified files |
| `docs/agents/project-manifest/constraints.md` | +8 new rules (see WP-035) |
| `docs/agents/project-manifest/tech-stack.md` | +`SixLabors.ImageSharp` NuGet entry |


## Assumptions

- `SixLabors.ImageSharp 3.1.9` (or latest stable 3.x at implementation time) is available on NuGet and compatible with `net10.0`. ImageSharp targets `netstandard2.1` / `net6.0+`, which is fully compatible.
- `FfmpegRunner` uses a two-step binary resolution chain: provisioner-populated `ISettingsService.Current.ExternalTools.Ffmpeg.FfmpegPath` → `"ffmpeg"` PATH fallback. There is no user-override step via `LibraryOptions` because `LibraryOptions` exposes `FfprobePath` but not `FfmpegPath`. Verify the provisioner key name before implementing WP-007.
- `movies_thumbnails.custom`, `mosaic`, `favorite` columns default to `'no'`; all M9-generated thumbnails use these defaults. `mosaic` and `favorite` are unused (always false) in M9.
- `CoverImageTransform.Crop == null` (or `CropRect.Size >= 1.0f`) means no crop is applied.
- The target MariaDB version supports window functions (≥ 10.2) for `GetCurrentFilePathsAsync`. If not, use the correlated subquery fallback provided in WP-011.
- `ThumbnailsOptions.ThumbnailSize` is accessible inside `ThumbnailsViewModel` — the constructor receives `ISettingsService` (add as constructor parameter alongside `IAppPaths`).
- The `AvaloniaMoviePropertiesService.ShowAsync` signature remains `ShowAsync(MoviePropertiesViewModel vm)` — the caller (`MovieEditorViewModel`) constructs the VM with the new optional services.


## Constraints

- `VideoIndexer.Core` must not reference `SixLabors.ImageSharp` — that dependency lives in `VideoIndexer.Infrastructure` only.
- All new `async` methods in Core and Infrastructure must use `.ConfigureAwait(false)` on every `await`.
- `FfmpegThumbnailGeneratorService.GenerateAsync` must return normally on cancellation — no `OperationCanceledException`.
- `TreatWarningsAsErrors=true` — all new code must compile warning-free.
- NuGet CPM — `SixLabors.ImageSharp` version declared only in `Directory.Packages.props`.
- `ThumbnailItemViewModel` and `CoverImageViewModel` reside in the App project; referencing `Avalonia.Media.Imaging.Bitmap` there is permitted.
- No schema migration required; the `movies_thumbnails` table exists at schema revision 41.
- Atomic cover image writes: use `.tmp` + `File.Move(overwrite: true)` (same pattern as `JsonSettingsService`).
- `ICoverImageService.HasCoverImage(hash)` is synchronous (`bool`, not `Task<bool>`) to support use in the post-query enrichment loop without requiring async infrastructure.
- The `.cim` and `.thb2` files are treated as JPEG regardless of extension; format detection is by magic bytes only.


## Out of Scope

- **"Jump to Video Position"** in `ThumbnailsView` context menu — shown as a disabled stub; wired in M10 when the embedded player ships.
- **Bookmark thumbnail capture** — M10.
- **Screenshot button** in the embedded player — M10.
- **`KeepOriginalSize` thumbnail setting** — `ThumbnailsOptions.KeepOriginalSize` exists in `AppOptions`; honouring it in `ThumbnailsView` cell sizing is a follow-up unless trivially implementable.
- **Shift+Click range-select in `ThumbnailsView`** — Avalonia `ListBox` `SelectionMode="Multiple"` handles Ctrl+Click natively; Shift+Click range-select requires additional code-behind; deferred unless Avalonia handles it automatically.
- **Mosaic generator** — dropped per rebuild philosophy.
- **Multi-edit thumbnail operations** — dropped per rebuild philosophy.


## Acceptance Criteria

- **AC-1:** After loading a movie in the editor, the Thumbnails tab loads and displays existing thumbnails from disk as a grid. The tab header shows the count (e.g. `"Thumbnails (24)"`).
- **AC-2:** Right-clicking a thumbnail and choosing **Use as Cover Image** copies the thumbnail to `cv.cim`; the Cover Image tab reloads and shows the new image. The `HasCoverImage` filter reflects the change immediately.
- **AC-3:** The Cover Image tab for a movie with a `cv.cim` shows the image. Rotate CW/CCW updates the preview. Brightness/Contrast/Gamma sliders update their values only. Clicking **Apply Now** saves the modified image atomically, reloads the preview, and updates `CoverImageBitmap` in the movies list.
- **AC-4:** Selecting a movie in the main list with `HasCoverImage = true` shows the cover image in the cover panel; selecting a movie without a cover shows "No cover" text.
- **AC-5:** The Generate tab allows setting an interval (seconds) and clicking **Generate Thumbnails** produces `.thb2` files on disk and DB rows in `movies_thumbnails` for the movie.
- **AC-6:** Clicking **Cancel** during generation stops after the current frame, discards all in-progress frames for the current movie, and leaves previously-completed movies' thumbnails intact.
- **AC-7:** Re-running generation on a movie that already has thumbnails presents an in-view confirmation panel; confirming deletes old thumbnails and starts a fresh run.
- **AC-8:** Selecting multiple movies in the main list and choosing **Generate Thumbnails** from the context menu opens the multi-movie generator dialog pre-loaded with the selection.
- **AC-9:** Double-clicking a thumbnail (or using "View Zoomed") opens the zoomed viewer dialog. **Next** / **Prev** navigate the thumbnail set.
- **AC-10:** Opening **Movie Properties** for a movie with a valid video file shows populated values for **File Type**, **Resolution**, **Duration**, and **Bitrate** (not `"—"`). Fields show `"—"` when the file path cannot be resolved.
- **AC-11:** The Filter DSL `HasCoverImage()` returns `true` only for movies with a `cv.cim` file on disk.
- **AC-12:** `dotnet build -c Release` reports zero errors and zero warnings.
- **AC-13:** `.\test.ps1` (unit + app suites) completes with all tests green.


## Testing Strategy

Unit tests use fakes for all cross-layer dependencies (no disk I/O, no DB, no process spawning for the ViewModel and service layer tests). Infrastructure unit tests use temporary directories for filesystem logic and fake `IDbConnection` for Dapper mapping tests. The `FfmpegThumbnailGeneratorService` is integration-tested via the existing `VideoIndexer.Infrastructure.Tests` project (self-skip when no DB configured). `CoverImageService.ApplyAndSaveAsync` is verified by hand on a real movie (crop + rotate + colour + Apply Now); its ImageSharp I/O end-to-end is not unit-tested. Service dialog tests (`AvaloniaThumbnailViewerService`, `AvaloniaThumbnailGenerationDialogService`) are not unit-tested in isolation (they wrap Avalonia `Window.ShowDialog` requiring a rendered context); verification is by acceptance testing.


## Test Plan

- `FfmpegRunnerTests.FormatPosition_ZeroMs_Returns_00_00_00_000` — AC-5 (correct frame position).
- `FfmpegRunnerTests.FormatPosition_90500Ms_Returns_00_01_30_500` — AC-5.
- `FfmpegThumbnailGeneratorServiceTests.BuildTimecode_90500_FormatsCorrectly` — AC-5.
- `CoverImageServiceTests.HasCoverImage_ReturnsFalse_WhenFileNotPresent` — AC-11.
- `CoverImageServiceTests.HasCoverImage_ReturnsTrue_WhenFilePresent` — AC-11.
- `CoverImageServiceTests.SetFromSourceAsync_CopiesFile_ToDestination` — AC-2.
- `DapperMovieCatalogRepositoryHasCoverImageTests.GetMovieListAsync_SetsHasCoverImageFalse_WhenFileAbsent` — AC-11.
- `DapperMovieCatalogRepositoryHasCoverImageTests.GetMovieListAsync_SetsHasCoverImageTrue_WhenCvCimPresent` — AC-11.
- `CoverImageViewModelTests.LoadAsync_SetsHasCoverImage_WhenFileExists` — AC-3.
- `CoverImageViewModelTests.LoadAsync_SetsHasCoverImageFalse_WhenMissing` — AC-3.
- `CoverImageViewModelTests.ApplyNowCommand_CallsServiceWithCorrectTransform` — AC-3.
- `CoverImageViewModelTests.RotateCwCommand_IncrementsBy90` — AC-3.
- `CoverImageViewModelTests.RotateCcwCommand_DecrementsBy90_EquivalentToCwThreeTimes` — AC-3.
- `CoverImageViewModelTests.RotateCw_FourTimes_ReturnsToZero` — AC-3.
- `CoverImageViewModelTests.CanApply_False_WhenNoCoverImage` — AC-3.
- `CoverImageViewModelTests.OnCoverImageUpdated_TriggersReload` — AC-2.
- `ThumbnailsViewModelTests.LoadAsync_PopulatesThumbnails_FromRepository` — AC-1.
- `ThumbnailsViewModelTests.LoadAsync_EmptyList_WhenNoThumbnails` — AC-1.
- `ThumbnailsViewModelTests.SelectAllCommand_SelectsAll` — AC-1.
- `ThumbnailsViewModelTests.SelectNoneCommand_DeselectsAll` — AC-1.
- `ThumbnailsViewModelTests.DeleteSelectedCommand_CallsDeleteOnRepoAndRemovesFromCollection` — AC-1.
- `ThumbnailsViewModelTests.UseThumbnailAsCoverCommand_CallsCoverService_AndFiresUpdatedEvent` — AC-2.
- `ThumbnailsViewModelTests.ViewZoomedCommand_CallsViewerService` — AC-9.
- `ThumbnailsViewModelTests.TabLabel_ReflectsThumbnailCount` — AC-1.
- `ThumbnailGeneratorViewModelTests.GenerateCommand_CallsServiceWithSingleRequest` — AC-5.
- `ThumbnailGeneratorViewModelTests.GenerateCommand_UpdatesMovieProgress_ViaProgressCallback` — AC-5.
- `ThumbnailGeneratorViewModelTests.GenerateCommand_SetsSummaryText_OnCompletion` — AC-5.
- `ThumbnailGeneratorViewModelTests.GenerateCommand_ShowsConfirmPanel_WhenMovieAlreadyHasThumbnails` — AC-7.
- `ThumbnailGeneratorViewModelTests.ConfirmRegenerateCommand_SetsConfirmedAndRerunsGenerate` — AC-7.
- `ThumbnailGeneratorViewModelTests.CancelCommand_CancelsToken` — AC-6.
- `ThumbnailGeneratorViewModelTests.IsMultiMovieMode_True_ForMultipleRequests` — AC-8.
- `ThumbnailViewerViewModelTests.CurrentIndex_InitialisedFromConstructor` — AC-9.
- `ThumbnailViewerViewModelTests.PrevCommand_DecrementsIndex` — AC-9.
- `ThumbnailViewerViewModelTests.NextCommand_IncrementsIndex` — AC-9.
- `ThumbnailViewerViewModelTests.CanNavigatePrev_False_AtFirstIndex` — AC-9.
- `ThumbnailViewerViewModelTests.CanNavigateNext_False_AtLastIndex` — AC-9.
- `ThumbnailViewerViewModelTests.CloseCommand_FiresCloseRequestedEvent` — AC-9.
- `MoviePropertiesViewModelTests.LoadMediaInfoAsync_SetsAllFourFields_WhenFfprobeSucceeds` — AC-10.
- `MoviePropertiesViewModelTests.LoadMediaInfoAsync_LeavesDefaultDash_WhenCurrentFilePathIsNull` — AC-10.
- `MoviePropertiesViewModelTests.LoadMediaInfoAsync_LeavesDefaultDash_WhenRunnerIsNull` — AC-10.
- `MoviePropertiesViewModelTests.LoadMediaInfoAsync_LeavesDefaultDash_OnFfprobeException` — AC-10.
- `MoviesListViewModelTests.GenerateThumbnailsCommand_CannotExecute_WithNoSelection` — AC-8.
- `MoviesListViewModelTests.GenerateThumbnailsCommand_CallsDialogService_WithResolvedRequests` — AC-8.
- `VideoIndexer.Infrastructure.Tests/DapperThumbnailRepositoryTests.InsertAsync_ReturnsAssignedId` — AC-5.
- `VideoIndexer.Infrastructure.Tests/DapperThumbnailRepositoryTests.GetByMovieId_ReturnsThumbnailsOrderedByMilliseconds` — AC-1.
- `VideoIndexer.Infrastructure.Tests/DapperThumbnailRepositoryTests.Delete_RemovesRow` — AC-6.
- `VideoIndexer.Infrastructure.Tests/DapperThumbnailRepositoryTests.DeleteAll_RemovesAllRowsForMovie` — AC-7.


## Documentation Updates

Per AGENTS.md manifest maintenance rules:

- `docs/agents/project-manifest/api-surface.md` — add `IAppPaths` new methods; add full signatures for all five new Core interfaces; add all six new model record shapes; add `Movie.CurrentFilePath`; add `IMovieRepository.GetCurrentFilePathsAsync`; add all six new ViewModel constructor + key property signatures; add `IThumbnailViewerService`, `IThumbnailGenerationDialogService` signatures; update `MovieEditorViewModel`, `MoviePropertiesViewModel`, `MoviesListViewModel` entries.
- `docs/agents/project-manifest/file-tree.md` — annotate all 50+ new files; update annotations for `DapperMovieCatalogRepository.cs`, `DapperMovieRepository.cs`, `AppPaths.cs`, `MovieEditorView.axaml`, `MoviesListView.axaml`, `MovieEditorViewModel.cs`, `MoviePropertiesViewModel.cs`, `AvaloniaMoviePropertiesService.cs`, `MoviesListViewModel.cs`, `Program.cs`.
- `docs/agents/project-manifest/constraints.md` — add 8 new rules as described in WP-035.
- `docs/agents/project-manifest/tech-stack.md` — add `SixLabors.ImageSharp 3.1.9` to the NuGet table with note: Infrastructure-only; pure managed .NET; cover image crop/rotate/colour processing.
- `docs/projects/rebuild/milestones/m9-images.md` — create milestone summary document once implementation is complete.


## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **`CoverImageCropperControl` pointer-event coordinate complexity** — mapping fractional crop rect to canvas bounds requires careful handling of `Stretch.Uniform` letter-boxing and DPI scaling. | Design the control to compute crop coordinates relative to the rendered `Image` `Bounds` after layout. Extract the coordinate math as a `internal static` pure function for unit testability. If pointer events prove difficult, fall back to numeric `NumericUpDown` controls for X/Y/Size as a minimal viable implementation; drag-crop becomes a polish follow-up. |
| **ImageSharp `GammaCorrection` API name** — the exact method name may vary between minor 3.x versions. | Pin `SixLabors.ImageSharp` to a specific patch version in `Directory.Packages.props`. Verify API name at implementation time; fall back to `Brightness`/`Contrast` only if the gamma API is absent. |
| **MariaDB window function availability** — `ROW_NUMBER() OVER (...)` requires MariaDB ≥ 10.2; older installs will fail `GetCurrentFilePathsAsync`. | Provide the correlated subquery alternative in WP-011 alongside the window function version; the engineer selects based on the confirmed minimum DB version. |
| **`HasCoverImage` enrichment performance** — O(N) `File.Exists` calls per `GetMovieListAsync` invocation; for a 10 000-movie library this is ~10 000 syscalls on each list refresh. | For M9, accept the cost (desktop app; typical page sizes are 500–2 000 rows). Document in `constraints.md` as a known performance trade-off; flag for caching optimisation if user feedback identifies it as a bottleneck. |
| **`ThumbnailsView` thumbnail image loading performance** — loading all `.thb2` files simultaneously for a large movie could saturate memory. | Load thumbnails asynchronously in `ThumbnailItemViewModel.LoadBitmapAsync` (fire-and-forget on tab activation). If performance is inadequate in testing, switch to lazy loading only for visible virtualised items; interface shape does not need to change. |
| **`ffmpeg` path resolution** — `FfmpegRunner` must follow exactly the same settings key chain as `FfprobeRunner`. | Verify `ISettingsService.Current.ExternalTools.Ffmpeg.FfmpegPath` key name against the provisioner's output before implementing WP-007. |
| **Stale `CurrentFilePath` after file move/delete** — `_movie.CurrentFilePath` reflects the path at load time; if the file was moved or deleted before generation, ffmpeg fails on that movie. | The generator counts it as a failure and includes it in the completion summary error list. This is expected behaviour; no additional mitigation needed. |
| **`SixLabors.ImageSharp` Apache 2.0 licence** | Apache 2.0 is compatible with the application's licensing. No commercial licence required. |
| **`AvaloniaMoviePropertiesService` fire-and-forget `LoadMediaInfoAsync`** — unobserved exceptions could crash the process. | Wrap the call in `try/catch` inside the fire-and-forget lambda; on failure set `vm.Duration = "Error"` (or similar) and log via `ILogger`. |
