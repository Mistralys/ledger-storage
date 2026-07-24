# Project Synthesis — Post-Rebuild Housekeeping & Bookmark Frame Capture

**Plan:** `2026-06-01-post-rebuild-housekeeping`
**Synthesised:** 2026-06-03
**Status:** COMPLETE — 12 / 12 work packages

---

## Executive Summary

The post-rebuild housekeeping plan delivered two concurrent work streams against the Video Indexer v2 codebase (version 1.3.0, .NET 10 / Avalonia UI).

**Track A — Bookmark frame capture (WP-001 – WP-006):** The bookmark dialog previously hardcoded `videoPath: null` in every call path, making frame-capture previews impossible. This plan threaded the current movie file path end-to-end from `MovieEditorViewModel._currentFilePath` → `BookmarksPanelViewModel._videoPath` → `IBookmarkSettingsService.ShowAsync` → `BookmarkSettingsViewModel`. With the path now available, `InitializeAsync` fires an automatic frame capture at the bookmark's timestamp when `CanNudge` is true, matching the existing nudge pattern exactly. The view bindings in `BookmarkSettingsView.axaml` were corrected from `IsCreateMode` to `CanNudge` so nudge controls are correctly hidden when ffmpeg is unavailable. Comprehensive unit tests were added, including a `FakeFfmpegRunner` that exercises the capture guard without disk I/O.

**Track B — Housekeeping (WP-007 – WP-012):** Closed the sole remaining M5 open work package, relocated three misplaced `FfprobeRunner.ResolveBinaryPath_*` pure-unit tests from the integration test project to the unit test project, removed stale M10 milestone annotations from source code, cleared resolved deferred items from the M9 milestone document, corrected the README platform requirements to include Linux (x64), and formalised the 1.3.0 release entry in `CHANGELOG.md` with full M9 feature coverage.

The project manifest (`api-surface.md`, `constraints.md`, `data-flows.md`, `file-tree.md`) was brought fully in sync with the delivered implementation.

---

## Metrics

| Metric | Value |
|---|---|
| Work packages | 12 / 12 COMPLETE |
| Total pipeline stages passed | 42 / 42 |
| Stages failed | 0 |
| Final unit test count | 329 |
| Final app test count | 543 |
| **Total tests (all green)** | **872** |
| Tests failed | 0 |
| Release build warnings | 0 |
| Release build errors | 0 |
| Version released | **1.3.0** |
| Source / test files modified | ~15 |
| Manifest / documentation files modified | ~8 |

---

## Work Package Summary

### Track A — Bookmark Frame Capture

| WP | Description | Key Deliverable |
|---|---|---|
| WP-001 | Thread `videoPath` through `IBookmarkSettingsService.ShowAsync` | Interface signature updated; `AvaloniaBookmarkSettingsService` forwards the argument; `FakeBookmarkSettingsService.LastVideoPath` added; stale `<remarks>` limitation block removed |
| WP-002 | Wire `videoPath` through `BookmarksPanelViewModel` and `MovieEditorViewModel` | `_videoPath` stored in panel VM; `AddBookmarkCommand` forwards it; `MovieEditorViewModel` passes `_currentFilePath` at construction; new `AddBookmarkCommand_PassesVideoPath_ToSettingsService` test |
| WP-003 | Initial frame capture in `BookmarkSettingsViewModel.InitializeAsync` | Automatic preview on dialog open when `CanNudge` is true; non-fatal failure handling (warnings only, dialog still opens) |
| WP-004 | Fix `BookmarkSettingsView.axaml` `IsVisible` bindings | Nudge StackPanel and "Add frame as thumbnail" CheckBox now bound to `CanNudge` (was `IsCreateMode`); Apply Preset StackPanel correctly retains `IsCreateMode` |
| WP-005 | Unit test coverage for `CanNudge`, `InitializeAsync` capture, `videoPath` wiring | 6 new `BookmarkSettingsViewModelTests` tests; `FakeFfmpegRunner` inner class (tracks call count + `positionMs`, no disk I/O); full `CanNudge` combinatorial coverage; 1 additional `BookmarksPanelViewModelTests` test |
| WP-006 | Relocate `FfprobeRunner.ResolveBinaryPath_*` tests to correct layer | New `tests/VideoIndexer.Tests/FfprobeRunnerTests.cs` using `InMemorySettingsService` (no Moq); Infrastructure test file retains only `[SkippableFact]` live-binary tests |

### Track B — Housekeeping

| WP | Description | Key Deliverable |
|---|---|---|
| WP-007 | Mark M5 milestone Complete | `m5-filters-search.md` status → Complete; WP-016 marked Complete; deferred identifier section retitled "all resolved" |
| WP-008 | Remove stale M10 annotations from `FilterExpressionEvaluator.cs` | Comment-only cleanup — 3 `<item>` description suffixes and 2 inline `// M10` comments removed; class-level `<remarks>` intentionally preserved |
| WP-009 | Remove resolved deferred items from M9 milestone document | "Bookmark thumbnail frame capture" and "FfprobeRunner.ResolveBinaryPath test placement" rows removed; "BinaryRunner shared abstraction" retained as still open |
| WP-010 | Update README platform requirements | Requirements section: "Windows (x64) or Linux (x64)" |
| WP-011 | Finalise v1.3.0 release | `CHANGELOG.md`: `[Unreleased]` → `[1.3.0] - 2026-06-01` with full M9 feature list; `README.md`: 4 new M9 feature bullets, security doc corrections, test count corrections |
| WP-012 | Final manifest consistency pass | `constraints.md`: new "Bookmarks (M10)" section documenting `CanNudge` dual-purpose convention and `_videoPath` immutability contract; `data-flows.md` Section 19 expanded with full videoPath threading and `InitializeAsync` capture branch |

---

## Open Items — Recommended Follow-up

The following items were explicitly deferred during this cycle. Recommended for the next planning session in priority order:

| Priority | Item | Source |
|---|---|---|
| Medium | **EditBookmarkCommand frame capture** — `EditBookmarkCommand` in `BookmarksPanelViewModel` calls `ShowAsync` without `videoPath` (null default). Edit mode does not offer frame-capture previews. Wiring `_videoPath` here would complete the Add/Edit symmetry. | WP-002 (Developer, QA, Reviewer) |
| Medium | **CaptureFrameAsync helper extraction** — The ~20-line frame-capture block (IsBusy toggle, temp file, `ExtractFrameAsync`, bitmap swap with disposal, `LogWarning` on failure, finally cleanup) is duplicated between `InitializeAsync` and `NudgeAsync`. Extract to a `private CaptureFrameAsync(string videoPath, long positionMs, string context)` helper. | WP-003 (code review) |
| Low | **FakeFfmpegRunner deduplication** — Private inner class duplicated in `BookmarkSettingsViewModelTests.cs` and `PlayerViewModelTests.cs`. Promote to `tests/VideoIndexer.App.Tests/TestHelpers/FakeFfmpegRunner.cs`. | WP-005 (Developer, code review) |
| Low | **Empty-string videoPath test** — No explicit test for `videoPath: string.Empty`. Guard (`string.IsNullOrEmpty`) handles it correctly; a test would add defence-in-depth. | WP-005 (QA) |
| Low | **FakeBookmarkSettingsService.LastHash** — Hash argument is not tracked; future hash-based assertions will need a `LastHash` property added first. | WP-001 (code review) |
| Open | **BinaryRunner shared abstraction** — Remaining open deferred item in M9 milestone (`m9-images-player-bookmarks.md`). | WP-009 |

---

## Pre-existing Issues

- **Transient test flakiness (`ThumbnailItemViewModelTests`):** `LoadBitmapAsync_FirstLoad_SetsBitmapProperty` and `Dispose_WithLoadedBitmap_DoesNotThrow` fail intermittently on first run under the Avalonia headless test runner. Both are pre-existing and unrelated to any change in this plan. Tests pass reliably on subsequent runs or in isolation.

---

## Strategic Notes

**Feature delivery discipline.** The bookmark frame-capture feature was cleanly decomposed across four independent-yet-coordinated WPs (001–004) with a dedicated test WP (005). Interface plumbing, ViewModel wiring, behavioral logic, view bindings, and tests were each scoped separately, keeping pipelines focused and reviewable. This is a useful template for similar multi-layer feature work in this codebase.

**CanNudge dual-purpose convention.** The `CanNudge` property on `BookmarkSettingsViewModel` now serves as both a behavioral gate (frame-capture in `InitializeAsync`/`NudgeAsync`) and a UI visibility gate (`IsVisible` in `BookmarkSettingsView.axaml`). The prior `IsCreateMode` binding was semantically incorrect — it would show nudge controls even when ffmpeg was unavailable. This convention is now documented in `constraints.md` and should not be reverted. Any future refactoring of the nudge UI must preserve the `CanNudge` binding.

**Test layer hygiene (WP-006).** Moving pure binary-path resolution tests out of the integration test project and into the unit test project (using `InMemorySettingsService` instead of Moq) was the correct call. The pattern established by `FfmpegRunnerTests.cs` should be the default for all new `ResolveBinaryPath_*`-style tests.

**1.3.0 release is clean.** Directory.Build.props was already at 1.3.0 before this plan began. The CHANGELOG and README have been brought into full alignment. The build is clean with zero warnings under `TreatWarningsAsErrors=true`.
