# Synthesis Report — M9: Images

**Plan:** `2026-05-22-m9-images`
**Date:** 2026-05-23
**Status:** COMPLETE
**Work Packages:** 34 / 34 COMPLETE

---

## Executive Summary

M9 delivers the full Images milestone for Video Indexer v2: cover image management with crop, rotate, and colour-adjust; thumbnail management with a grid view and context actions; FFmpeg-backed thumbnail generation (interval-based extraction for single and multi-movie workflows); a zoomed thumbnail viewer dialog; live cover-image preview in the movies-list panel; and carry-forward fixes for `HasCoverImage`, the deferred `GenerateThumbnailsAsync` stub, and four ffprobe-deferred `MoviePropertiesViewModel` fields (`FileType`, `Resolution`, `Duration`, `Bitrate`).

The milestone spans all three layers: Core domain (new models and interfaces), Infrastructure (FFmpeg runner, ImageSharp-backed cover service, Dapper repository, thumbnail generator), and App (six new ViewModels, four new Views, editor tab wiring, DI composition root). The centre `TabControl` in `MovieEditorView` is now fully functional for the Cover Image, Thumbnails, and Generate tabs; the Video tab remains a stub for M10.

---

## Metrics

| Metric | Value |
|---|---|
| **Work Packages** | 34 / 34 COMPLETE |
| **Pipeline stages total** | 138 (mix of 4-stage and 5-stage pipelines) |
| **Security-audited WPs** | 5 (WP-010, WP-014, WP-015, WP-016, WP-020) |
| **Security findings** | 1 Medium (A04 infinite loop — closed via Reviewer fix-forward) |
| **Unit tests at milestone start** | 282 |
| **Unit tests at milestone end** | 309 (+27) |
| **App tests at milestone start** | 361 |
| **App tests at milestone end** | 435 (+74) |
| **Total tests** | 744 (+101 net new tests) |
| **Failed tests** | 0 |
| **Release build warnings** | 0 |
| **WP rework cycles** | 2 (WP-020 impl once; WP-033 QA bounce once) |

---

## What Was Built

### Phase 1 — Core Domain (WP-001–WP-008)
- **WP-001:** `IAppPaths` extended with `CoverImagePath(hash)` and `ThumbnailFilePath(hash, id)`; both `FakeAppPaths` implementations updated for test compatibility.
- **WP-002:** New Core models — `Thumbnail`, `CropRect`, `CoverImageTransform`, `ThumbnailGenerationRequest` (with `IntervalSeconds` default 30), `ThumbnailGenerationResult`, `ThumbnailGenerationProgress`.
- **WP-003:** `IFfmpegRunner` interface — `ExtractFrameAsync(videoPath, outputPath, positionMs, ct)` in Core.
- **WP-004:** `IThumbnailRepository` interface — CRUD and bulk delete operations.
- **WP-005:** `ICoverImageService` interface — `LoadAsync`, `ApplyAndSaveAsync`, `RotateForPreviewAsync`.
- **WP-006:** `IThumbnailGenerationService` interface — `GenerateAsync` with progress reporting.
- **WP-007:** `IMovieRepository` extended — `GetCurrentFilePathsAsync` batch path query; `Movie.CurrentFilePath` property via JOIN.
- **WP-008:** `ThumbnailItemViewModel` — per-thumbnail row VM for the grid view.

### Phase 2 — Infrastructure (WP-009–WP-020)
- **WP-009:** `AppPaths` — implemented `CoverImagePath` and `ThumbnailFilePath` helpers.
- **WP-010 (DapperMovieRepository):** `GetByIdAsync` LEFT JOIN on `movies_filenames`; `GetCurrentFilePathsAsync` batch correlated subquery with null-fill guarantee.
- **WP-011 (DapperMovieCatalogRepository):** `HasCoverImage` post-query `File.Exists` enrichment loop; `IAppPaths` injected as optional parameter for test backward-compatibility.
- **WP-012:** `FfmpegRunner` — two-step binary resolution chain (provisioner path → PATH fallback); throws `InvalidOperationException` on non-zero exit, mirroring `IFfprobeRunner` contract.
- **WP-013 (DapperThumbnailRepository):** Implemented all six CRUD methods (created in WP-033 scope as remediation — see Process Findings).
- **WP-014 (ICoverImageService / CoverImageService):** ImageSharp-backed service; `ApplyAndSaveAsync` uses single `Mutate` pass with atomic `.tmp` + `File.Move(overwrite: true)` write pattern; `RotateForPreviewAsync` returns in-memory bytes without writing to disk.
- **WP-020 (FfmpegThumbnailGeneratorService):** Full implementation with per-movie ffprobe duration probe, pre-generation DB cleanup, per-frame extraction via `IFfmpegRunner`, cooperative cancellation (check-before-progress ordering), partial-frame rollback on failure, and normal return on cancellation.

### Phase 3 — App Layer (WP-015–WP-032)
- **WP-015–WP-019:** `ThumbnailsViewModel` (grid, bidirectional selection sync), `CoverImageViewModel` (brightness/contrast/gamma sliders + rotation preview + crop), `ThumbnailGeneratorViewModel` (dual single/multi-movie mode), `ThumbnailViewerViewModel`, `ThumbnailGeneratorMovieViewModel` (per-movie progress row).
- **WP-021–WP-027:** New Views — `ThumbnailsView`, `CoverImageView` (with `CoverImageCropperControl`), `ThumbnailGeneratorView`, `ThumbnailViewerView`; dialog services `AvaloniaThumbnailViewerService`, `AvaloniaThumbnailGenerationDialogService`; `MovieEditorView` stub tabs replaced.
- **WP-028:** `MovieEditorViewModel` extended — sub-VMs wired (`CoverImageVm`, `ThumbnailsVm`, `GeneratorVm`); optional service params for backward test compatibility.
- **WP-029:** `MoviesListViewModel` extended — `CoverImageBitmap` property with fire-and-forget async load (CTS cancel/replace); `GenerateThumbnailsAsync` command wired to multi-movie generator dialog.
- **WP-030–WP-031:** `MoviePropertiesViewModel` — `LoadMediaInfoAsync` added; `FileType`, `Resolution`, `Duration`, `Bitrate` populated via `IFfprobeRunner` from `Movie.CurrentFilePath`; four properties backed by `[ObservableProperty]`.
- **WP-032:** `MoviesListView` — cover panel placeholder replaced with `Image` control bound to `CoverImageBitmap`.
- **WP-033:** DI composition root — all 9 new services registered; both factory lambdas wired (with rework cycle to fix `MoviesListViewModel` missing 4 args); 4 new view types registered.
- **WP-034:** Manifest documentation — `file-tree.md`, `api-surface.md`, `constraints.md`, `data-flows.md` all updated to reflect M9 state.

---

## Security Review Summary

Five WPs underwent a full security-audit pipeline stage. All audits passed.

| WP | Subject | Finding | Resolution |
|---|---|---|---|
| WP-010 | `DapperMovieRepository` | 0 findings | — |
| WP-014 | `CoverImageService` | 0 findings | — |
| WP-015 | `DapperThumbnailRepository` | 0 findings | — |
| WP-016 | `CoverImageViewModel` | 0 findings | — |
| WP-020 | `FfmpegThumbnailGeneratorService` | **1 Medium — A04 (Insecure Design):** `IntervalSeconds = 0` produces `intervalMs = 0`, causing an infinite frame-position loop | Fixed by Reviewer fix-forward in code-review pipeline: `intervalMs` clamped to 30 000 ms with `LogWarning` if `IntervalSeconds <= 0`. |

---

## Process Findings

### WP-015 Misread (High-Priority Process Incident)
The Developer agent assigned to WP-015 read `work/WP-008.md` (ThumbnailItemViewModel spec) instead of the DapperThumbnailRepository spec and implemented the wrong class. WP-015 was marked COMPLETE in the ledger without `DapperThumbnailRepository.cs` ever existing. This was detected during WP-033 when the DI composition root attempted to register the missing type. `DapperThumbnailRepository` was created as a remediation within WP-033 scope with no test gap (integration tests require a live DB and self-skip without one).

**Root cause:** The `work_package_file` metadata field in the ledger mapped WP-015 to the wrong spec file; the implementing agent used that path without cross-checking.

**Recommendation:** Agents should cross-reference the WP title and acceptance criteria against the spec file content before starting implementation.

### WP-033 QA Bounce
`MoviesListViewModel` DI factory lambda was missing 4 named service arguments (`IMovieRepository`, `IThumbnailGenerationService`, `IThumbnailRepository`, `IThumbnailGenerationDialogService`), causing `GenerateThumbnailsCommand.CanExecute` to be permanently `false` in production. Fixed in a single rework cycle (implementation + QA re-verification). The Reviewer then applied a fix-forward to convert all 12 factory arguments to named arguments as a regression-prevention measure.

---

## Strategic Recommendations (Gold Nuggets)

1. **`IntervalSeconds` constraint should be model-level, not service-level.**
   The current fix clamps at the service; the more robust fix is to validate `IntervalSeconds > 0` on `ThumbnailGenerationRequest` itself (e.g. via a constructor guard or `init` validation). A corrupt settings file could still reach the service with a 0-value if the UI allows it. This is a low-risk M10 clean-up item.

2. **`IDisposable` gap in `MovieEditorViewModel` and `MoviesListViewModel`.**
   Both VMs hold `Avalonia.Media.Imaging.Bitmap` instances (sub-VMs and `CoverImageBitmap` respectively) that are not disposed when the VM is torn down. Avalonia Bitmaps hold native image handles. In the current session this is harmless (single long-lived VM), but should be addressed before M10 adds further image-heavy navigation patterns.

3. **`DapperThumbnailRepository` captive dependency.**
   `DapperThumbnailRepository` is registered `AddTransient` but is captured by `FfmpegThumbnailGeneratorService` (registered `AddSingleton`). The repository is stateless (opens a new connection per method call), so no functional bug exists today. However, the registration should be changed to `AddSingleton` to accurately reflect the effective lifetime and prevent future confusion if state is ever added.

4. **`SyncProgress<T>` test helper should be promoted.**
   `FfmpegThumbnailGeneratorServiceTests.cs` contains a `SyncProgress<T>` inner class that dispatches `IProgress<T>` callbacks synchronously — required for any test verifying progress side-effects. This pattern is likely to recur in M10 (scanner progress, etc.). Promoting it to `tests/VideoIndexer.Tests/Fixtures/SyncProgress.cs` eliminates copy-paste.

5. **`FormattableString.Invariant` convention for test helpers.**
   Discovered in WP-020: a test helper that used `:F6` format without `FormattableString.Invariant` produced comma decimal separators on non-English locales, causing `InvariantCulture` double parse to misread values by a factor of 1 000. The convention should be documented: *any test helper formatting doubles or floats into parseable strings must use `FormattableString.Invariant(…)` or `.ToString("F6", CultureInfo.InvariantCulture)`.*

6. **`window function` query upgrade path documented.**
   `DapperMovieRepository` uses correlated scalar subqueries for `GetByIdAsync` and `GetCurrentFilePathsAsync`. A window function variant is documented in code comments for a future upgrade once minimum MariaDB version is confirmed ≥ 10.2. No action needed now; retain the comment as a signal for the eventual DB version policy decision.

---

## Open Work Items Carried into M10

| Item | Source | Priority |
|---|---|---|
| Video tab in `MovieEditorView` remains a stub | Plan scope — explicitly deferred to M10 | Medium |
| `LibraryOptions.FfmpegPath` user-override setting | `FfmpegRunner` uses two-step chain only; third step deferred — no `LibraryOptions.FfmpegPath` field in M9 | Low |
| `IDisposable` for `Bitmap` in `MovieEditorViewModel` + `MoviesListViewModel` | Reviewer observation (WP-033) | Medium |
| `DapperThumbnailRepository` registration → `AddSingleton` | Reviewer observation (WP-033) | Low |
| `IntervalSeconds` model-level guard | Security Auditor + QA (WP-020) | Low |
| `SyncProgress<T>` to shared fixture | Developer observation (WP-020) | Low |
| `FormattableString.Invariant` convention documented in `constraints.md` | Developer observation (WP-020) | Low |
| Integration test coverage for `DapperThumbnailRepository` | QA note (WP-033) — requires live DB | Medium |

---

## Next Steps for Planner / Manager

1. **Start M10 planning.** The Video tab stub is the primary remaining tab in `MovieEditorView`. M10 scope should address it along with `LibraryOptions.FfmpegPath` user override (enabling custom ffmpeg binary paths).
2. **Address `IDisposable` gap** for Bitmap fields — include this as a technical-debt WP at the start of M10, before adding further image-heavy navigation.
3. **Integration test suite for `DapperThumbnailRepository`** — the six repository methods have no automated coverage. Add integration tests (DB-backed) for at minimum `InsertAsync`, `GetByMovieIdAsync`, and `DeleteAllByMovieIdAsync`.
4. **Enforce `FormattableString.Invariant` in `constraints.md`** — add a single-line rule to prevent locale-sensitive decimal format bugs in future test helpers.
