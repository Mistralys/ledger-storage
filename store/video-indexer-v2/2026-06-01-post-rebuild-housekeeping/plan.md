# Plan: Post-Rebuild Housekeeping & Bookmark Frame Capture

## Plan Audit Cycles
- Audits: 1 — Plan Auditor v1.4.0
- Architectural Reviews: none — Plan Architect Reviewer v1.5.0


## Summary

The rebuild roadmap (M1–M9, with M10 merged into M9) is complete and the codebase is at version 1.3.0. This plan addresses all remaining open items from that cycle in two tracks:

**Track A — Bookmark frame capture (deferred from M9):** The `BookmarkSettingsDialog` opens with nudge controls and a preview image slot already present in the AXAML, but the video path is never threaded through to the service call — `AvaloniaBookmarkSettingsService.ShowAsync` always passes `videoPath: null`. This plan wires the current file path from `MovieEditorViewModel` down through `BookmarksPanelViewModel` and `IBookmarkSettingsService.ShowAsync` into `BookmarkSettingsViewModel`, enabling the initial frame capture at the bookmark's timestamp on dialog open and full nudge preview.

**Track B — Housekeeping:** Closes the sole remaining open work package (M5 WP-016 documentation, now verifiably complete), relocates three misplaced pure-unit tests from the integration test project into the unit test project, updates the README for Linux support, and formalises the CHANGELOG 1.3.0 release entry.


## Architectural Context

The relevant call chain for bookmark creation is:

```
MovieEditorViewModel                       (App/ViewModels)
  └─ creates BookmarksPanelViewModel       (App/ViewModels)
       └─ calls IBookmarkSettingsService.ShowAsync
                └─ AvaloniaBookmarkSettingsService   (App/Services)
                     └─ constructs BookmarkSettingsViewModel (App/ViewModels)
                          └─ calls IFfmpegRunner.ExtractFrameAsync  (Core/Abstractions)
                                   └─ FfmpegRunner                  (Infrastructure/Library)
```

Key files:

| File | Role |
|---|---|
| `src/VideoIndexer.App/Services/IBookmarkSettingsService.cs` | Service interface — `ShowAsync` signature |
| `src/VideoIndexer.App/Services/AvaloniaBookmarkSettingsService.cs` | Production implementation — always passes `videoPath: null` today |
| `src/VideoIndexer.App/ViewModels/BookmarksPanelViewModel.cs` | Panel VM — `AddBookmarkCommand` calls `ShowAsync` |
| `src/VideoIndexer.App/ViewModels/BookmarkSettingsViewModel.cs` | Dialog VM — `InitializeAsync`, `NudgeAsync`; `CanNudge` checks `_videoPath` |
| `src/VideoIndexer.App/Views/BookmarkSettingsView.axaml` | Nudge controls visible when `IsCreateMode`; should be `CanNudge` |
| `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs` | Has `_currentFilePath`; constructs `BookmarksPanelViewModel` |
| `tests/VideoIndexer.App.Tests/BookmarkSettingsViewModelTests.cs` | Unit tests for dialog VM |
| `tests/VideoIndexer.App.Tests/BookmarksPanelViewModelTests.cs` | Unit tests for panel VM |
| `tests/VideoIndexer.App.Tests/TestHelpers/FakeBookmarkSettingsService.cs` | In-memory fake; needs `LastVideoPath` tracking |
| `tests/VideoIndexer.Infrastructure.Tests/Library/FfprobeRunnerTests.cs` | Contains 3 misplaced pure-unit tests |
| `tests/VideoIndexer.Tests/FfmpegRunnerTests.cs` | Canonical location for `ResolveBinaryPath` unit tests |


## Approach / Architecture

### Track A — Bookmark frame capture

The `BookmarkSettingsViewModel` already has full nudge and preview logic implemented; the only missing piece is the video path reaching the constructor. The fix is a narrow, additive thread from `MovieEditorViewModel` down to the dialog:

1. Add `string? videoPath = null` as a defaulted parameter to `IBookmarkSettingsService.ShowAsync` — backwards-compatible; existing callers and the fake compile unchanged.
2. Thread the parameter through `AvaloniaBookmarkSettingsService` and `BookmarksPanelViewModel`.
3. In `MovieEditorViewModel`, pass `_currentFilePath` when constructing `BookmarksPanelViewModel`.
4. Add an initial frame capture in `BookmarkSettingsViewModel.InitializeAsync` so a preview appears immediately when the dialog opens in Create mode (same logic as `NudgeAsync`).
5. Fix AXAML visibility bindings in `BookmarkSettingsView.axaml`: the nudge `StackPanel` (line 30) and the "Add frame as thumbnail" `CheckBox` (line 89) are both bound to `IsCreateMode` but should be `CanNudge` — both are inert without a capturable frame, so showing them when no video path is available is misleading. The "Apply Preset" `StackPanel` (line 63) is correctly bound to `IsCreateMode` and must not be changed.

No new interfaces, no new services, no schema change. The `BookmarkThumbnailPath` convention and `_videoPath`/`_ffmpegRunner` extension points are already in place.

### Track B — Housekeeping

- **M5 WP-016:** Verification pass confirms the manifest is current (file-tree, api-surface, constraints, tech-stack, data-flows all reflect M5 content). Mark WP-016 Complete and set M5 Status → Complete in the milestone document.
- **FfprobeRunner test placement:** The 3 `ResolveBinaryPath_*` tests at the bottom of `FfprobeRunnerTests.cs` (Infrastructure.Tests) are pure-unit tests with no database dependency. They belong in `VideoIndexer.Tests` alongside `FfmpegRunnerTests.cs`. Create `tests/VideoIndexer.Tests/FfprobeRunnerTests.cs` carrying these tests, then delete them from the Infrastructure file (leaving the two live `[SkippableFact]` tests in place). Update `file-tree.md`.
- **README platform update:** Requirements section still lists "Windows (x64)". The Linux support change (CHANGELOG [Unreleased]) makes that inaccurate. Update to list Windows x64 and Linux x64.
- **CHANGELOG 1.3.0 release:** Rename `[Unreleased]` → `[1.3.0] - 2026-06-01` and add the full M9 milestone feature list that is currently missing from the CHANGELOG (the [Unreleased] section covers only security hardening and the m042 migration; the M9 player, bookmarks, images, and cover management features shipped as part of 1.3.0 and must be recorded).


## Rationale

- **Adding `videoPath` to the interface rather than a new overload** — one interface method is simpler; the parameter is defaulted to `null` so all existing call sites continue to compile without modification. A second overload would fragment the interface and add noise to the fake.
- **Initial capture in `InitializeAsync` rather than a new command** — the spec mandates a live preview immediately on dialog open. Wiring a separate `CaptureInitialFrameCommand` called from the view's `Loaded` handler would introduce a second code path identical to `NudgeAsync`. Sharing the same helper avoids duplication.
- **`CanNudge` instead of `IsCreateMode` for nudge visibility** — the dialog can be opened without a video path (e.g. when the movie has no current file). Showing non-functional nudge buttons in that case is misleading. `CanNudge` already encodes the correct three-way check.
- **`CanNudge` for the "Add frame as thumbnail" checkbox too** — the checkbox is only meaningful when a frame capture is possible. Displaying it when `CanNudge` is false implies a thumbnail will be saved when none can ever be captured; binding it to `CanNudge` is consistent with the nudge section above it.
- **Test relocation without deletion of live tests** — the two `[SkippableFact]` tests (`ProbeAsync_WhenFfprobeExitsNonZero_Throws…` and `ProbeAsync_UsesLibraryOverridePath_WhenBothPathsAreSet`) legitimately belong in Infrastructure.Tests because they require a live binary. Only the three pure-unit `ResolveBinaryPath_*` tests move.


## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|----------|--------------|-------------------------|-------------------|
| Threading video path | Add `videoPath` param to `IBookmarkSettingsService.ShowAsync` | Inject `IMovieRepository` into `BookmarksPanelViewModel` and resolve path on demand | Resolving on demand adds an async call inside `AddBookmarkCommand` and couples the panel to the repository; passing the path from the editor is cleaner and more explicit |
| Initial frame capture trigger | Inside `InitializeAsync` | Separate `LoadPreviewCommand` called from view `Loaded` | `InitializeAsync` is already called before the dialog shows; a second entry point duplicates async machinery and creates a race window |
| Nudge section visibility | `CanNudge` | Keep `IsCreateMode` | `IsCreateMode` shows non-functional controls when no video path is available; `CanNudge` is the correct semantic guard |
| "Add frame as thumbnail" checkbox visibility | `CanNudge` | Keep `IsCreateMode` | The checkbox is inert without a captured frame; showing it when `CanNudge` is false implies a thumbnail will be saved when none can be captured |
| Test relocation strategy | New `FfprobeRunnerTests.cs` in `VideoIndexer.Tests` | Move entire `FfprobeRunnerTests.cs` | Live tests must stay in Infrastructure.Tests (they use `[SkippableFact]`); moving only the pure-unit tests preserves the correct test-suite boundary |


## Pattern Alignment

| Pattern | File | Alignment |
|---------|------|-----------|
| Defaulted optional service parameter | `IBookmarkSettingsService.ShowAsync` | Follows the established `IFfmpegRunner? ffmpegRunner = null` optional-injection pattern in `BookmarkSettingsViewModel` |
| Initial async load in `InitializeAsync` | `BookmarkSettingsViewModel` | Matches `ThumbnailsViewModel.LoadCommand`, `CoverImageViewModel.LoadCommand` — all perform their first async I/O in a named async initialisation method |
| `FakeXxx` service with `LastXxx` call-tracking | `FakeBookmarkSettingsService` | Follows `FakeBookmarkRepository.GetByMovieIdCallCount`, `FakeLabelCleanerService.WasShowCalled`, etc. |
| Pure-unit tests in `VideoIndexer.Tests` | `FfprobeRunnerTests.cs` | Follows `FfmpegRunnerTests.cs` which sits in the same project and tests the same `ResolveBinaryPath` internal method |
| CHANGELOG Keep-a-Changelog format | `CHANGELOG.md` | No departure — existing format maintained |


## Detailed Steps

### Track A — Bookmark frame capture

1. **`IBookmarkSettingsService.ShowAsync`** — add `string? videoPath = null` as the sixth parameter (before `CancellationToken ct = default`).

2. **`FakeBookmarkSettingsService`** — add `public string? LastVideoPath { get; private set; }` and capture it in `ShowAsync`.

3. **`AvaloniaBookmarkSettingsService.ShowAsync`** — accept `string? videoPath = null` and pass it to the `BookmarkSettingsViewModel` constructor (replacing the hardcoded `videoPath: null`). Remove the now-outdated `<remarks>` block documenting the limitation.

4. **`BookmarksPanelViewModel`** — add `string? videoPath = null` constructor parameter; store in `_videoPath` field; pass to `_settingsService.ShowAsync` in `AddBookmarkCommand`.

5. **`MovieEditorViewModel`** — pass `videoPath: _currentFilePath` when constructing `BookmarksPanelViewModel` (the field is already assigned at line 653 before `BookmarksPanelViewModel` is constructed at line 727).

6. **`BookmarkSettingsViewModel.InitializeAsync`** — after loading suggestions and presets, if `CanNudge` is true (i.e. `IsCreateMode && _ffmpegRunner is not null && !string.IsNullOrEmpty(_videoPath)`), fire an initial frame capture using the same temp-file-and-load pattern as `NudgeAsync`. Wrap in `try/catch` and log a warning on failure — a missing initial frame is not fatal.

7. **`BookmarkSettingsView.axaml`** — two `IsVisible` binding changes:
   - Nudge `StackPanel` (line 30): `{Binding IsCreateMode}` → `{Binding CanNudge}`.
   - "Add frame as thumbnail" `CheckBox` (line 89): `{Binding IsCreateMode}` → `{Binding CanNudge}`.
   - "Apply Preset" `StackPanel` (line 63): leave as `{Binding IsCreateMode}` — presets are useful regardless of frame-capture availability.

8. **`BookmarkSettingsViewModelTests`** — add tests:
   - `CanNudge_IsFalse_WhenCreateMode_FfmpegRunnerSet_VideoPathNull` (ffmpegRunner provided, videoPath null)
   - `CanNudge_IsFalse_WhenEditMode_AndVideoPathProvided` (correct mode guard)
   - `CanNudge_IsTrue_WhenCreateMode_FfmpegRunnerSet_VideoPathSet`
   - `NudgeForwardCommand_NoOps_WhenCanNudgeIsFalse` (no IFfmpegRunner)

9. **`BookmarksPanelViewModelTests`** — add tests:
   - `AddBookmarkCommand_PassesVideoPath_ToSettingsService` (new `MakeSut` overload accepting `videoPath`; assert `FakeBookmarkSettingsService.LastVideoPath`)
   - `AddBookmarkCommand_PassesNullVideoPath_WhenNotProvided`

### Track B — Housekeeping

10. **`m5-filters-search.md`** — set WP-016 status to `Complete`; change milestone `Status` from `Active` to `Complete`. Update the DSL Quick Reference "Deferred identifiers" section: remove `HasTag` and `TagHasSubTags` from the M7 deferred list (activated in M7 WP-007) and remove `HasRatedBookmarks`, `BookmarkContains`, and `AmountBookmarks` from the M10 deferred list (activated in M9 rework — all five identifiers are now fully active).

11. **`tests/VideoIndexer.Tests/FfprobeRunnerTests.cs`** (new file) — move the three `ResolveBinaryPath_*` pure-unit tests from `VideoIndexer.Infrastructure.Tests/Library/FfprobeRunnerTests.cs` to this new file. Follow the namespace and fixture pattern of the neighbouring `FfmpegRunnerTests.cs`.

12. **`tests/VideoIndexer.Infrastructure.Tests/Library/FfprobeRunnerTests.cs`** — delete the three `ResolveBinaryPath_*` methods (the two `[SkippableFact]` live-binary tests remain).

13. **`README.md`** — update the Requirements section: replace `Windows (x64)` with `Windows (x64) or Linux (x64)`.

14. **`CHANGELOG.md`** — rename `[Unreleased]` → `[1.3.0] - 2026-06-01`; add a comprehensive M9 feature summary section covering images, player, bookmarks, and the bookmark filter DSL extensions, which were not captured in prior CHANGELOG versions. Remove the `### Technical Debt` subsection — the `MovieEditorViewModel` 22-parameter constructor it described was resolved by the `MovieEditorOptions` record delivered in M9 rework; add a `### Changed` bullet noting the constructor consolidation instead.

16. **`src/VideoIndexer.Core/Filtering/FilterExpressionEvaluator.cs`** — remove the stale `M10 (Player & Bookmarks)` milestone annotations from the `<item>` XML doc-comment entries for `AmountBookmarks`, `HasRatedBookmarks()`, and `BookmarkContains()` (lines 28, 41, 52), and also remove the two inline `// M10 (Player & Bookmarks):` implementation comments at lines 149 and 180. These identifiers are now fully active; all milestone markers are misleading and will cause confusion in future maintenance.

17. **`docs/projects/rebuild/milestones/m9-images-player-bookmarks.md`** — remove the "Bookmark thumbnail frame capture" and "FfprobeRunner.ResolveBinaryPath test placement" rows from the Deferred Items table (both are resolved by this plan).

15. **Manifest updates** — update `docs/agents/project-manifest/`:
    - `api-surface.md` — update `IBookmarkSettingsService.ShowAsync` signature to include `videoPath`; update `BookmarksPanelViewModel` constructor signature.
    - `file-tree.md` — add `FfprobeRunnerTests.cs` entry under `VideoIndexer.Tests`.
    - `constraints.md` — document the bookmark frame-capture convention (initial capture in `InitializeAsync`; nudge controls only shown when `CanNudge`).


## Dependencies

- No new NuGet packages required.
- No schema change required (`thumbnail_path` column on `movies_bookmarks` was added in m042).
- Track A and Track B steps are independent; they may be implemented in parallel or in any order.


## Required Components

### New files
- `tests/VideoIndexer.Tests/FfprobeRunnerTests.cs` — relocated pure-unit tests

### Modified files
- `src/VideoIndexer.App/Services/IBookmarkSettingsService.cs`
- `src/VideoIndexer.App/Services/AvaloniaBookmarkSettingsService.cs`
- `src/VideoIndexer.App/ViewModels/BookmarksPanelViewModel.cs`
- `src/VideoIndexer.App/ViewModels/BookmarkSettingsViewModel.cs`
- `src/VideoIndexer.App/Views/BookmarkSettingsView.axaml`
- `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs`
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeBookmarkSettingsService.cs`
- `tests/VideoIndexer.App.Tests/BookmarkSettingsViewModelTests.cs`
- `tests/VideoIndexer.App.Tests/BookmarksPanelViewModelTests.cs`
- `tests/VideoIndexer.Infrastructure.Tests/Library/FfprobeRunnerTests.cs` (3 tests removed)
- `src/VideoIndexer.Core/Filtering/FilterExpressionEvaluator.cs` (stale doc-comment and inline comment cleanup)
- `docs/projects/rebuild/milestones/m5-filters-search.md`
- `docs/projects/rebuild/milestones/m9-images-player-bookmarks.md` (remove resolved deferred items)
- `README.md`
- `CHANGELOG.md`
- `docs/agents/project-manifest/api-surface.md`
- `docs/agents/project-manifest/file-tree.md`
- `docs/agents/project-manifest/constraints.md`
- `docs/agents/project-manifest/data-flows.md`


## Assumptions

- `_currentFilePath` is populated in `MovieEditorViewModel` before `BookmarksPanelViewModel` is constructed (verified: line 653 assigns it, line 727 constructs the panel).
- A null `videoPath` in `BookmarksPanelViewModel` is a valid, non-error state (movie has no current file on disk). The panel must still work without frame capture in that case.
- The `FfmpegRunner` DI registration in `Program.cs` is already in place (it is — registered for `IThumbnailGenerationService` and `IFfmpegRunner`).


## Constraints

- `IBookmarkSettingsService.ShowAsync` signature change must default `videoPath` to `null` to avoid breaking `FakeBookmarkSettingsService` and any other existing callers.
- The initial frame capture in `InitializeAsync` must use `ConfigureAwait(true)` (UI thread continuation) consistent with the rest of `BookmarkSettingsViewModel` (which already uses `ConfigureAwait(true)` for bitmap assignment).
- `CanNudge` is a computed property with no backing field — it must be re-evaluated after `_videoPath` and `_ffmpegRunner` are set. Since these are set only via the constructor, `CanNudge` is computed at construction time and does not need `OnPropertyChanged`.
- All warnings are errors (`TreatWarningsAsErrors=true`) — no new warnings may be introduced.
- `IBookmarkSettingsService` lives in the App layer (not Core) and may reference App-layer constructs.


## Out of Scope

- `BinaryRunner` shared base class for `FfmpegRunner` / `FfprobeRunner` — deferred from M9; only warranted when a third runner is added.
- Bookmark thumbnail save-to-disk on dialog close (`AddToThumbnails` checkbox) — the checkbox exists in the UI; actual disk write on accept is a separate future feature.
- `VideoIndexer.Infrastructure.Tests` FfprobeRunner priority-inversion tests (the two `[SkippableFact]` live tests) — remain in their current file; no change required.
- Packaging / installer work — out of scope for this plan.


## Acceptance Criteria

- [ ] Opening the Bookmark Settings dialog (Create mode) for a movie with a resolvable current file path displays a preview frame at the bookmark's initial position.
- [ ] Clicking any nudge button updates `PreviewBitmap` and `PositionMs` when a video path is available.
- [ ] The nudge row is hidden when the movie has no current file path (i.e. `CanNudge` is false).
- [ ] The "Add frame as thumbnail" `CheckBox` is hidden when `CanNudge` is false (same condition as the nudge section).
- [ ] `BookmarksPanelViewModelTests` passes with `LastVideoPath` assertions.
- [ ] `BookmarkSettingsViewModelTests` passes with new `CanNudge` tests.
- [ ] The three `ResolveBinaryPath_*` tests run in `VideoIndexer.Tests` (unit suite) and no longer appear in `VideoIndexer.Infrastructure.Tests`.
- [ ] `.\test.ps1` (unit + app suites) passes with zero failures.
- [ ] `dotnet build -c Release` reports zero warnings and zero errors.
- [ ] `README.md` Requirements section mentions both Windows (x64) and Linux (x64).
- [ ] `CHANGELOG.md` has a `[1.3.0] - 2026-06-01` section; `[Unreleased]` is empty or absent.
- [ ] `m5-filters-search.md` shows Status: Complete and WP-016: Complete.
- [ ] Manifest documents (`api-surface.md`, `file-tree.md`, `constraints.md`) reflect all changes.


## Testing Strategy

All new test code is unit-level (no I/O, no UI thread). `BookmarkSettingsViewModel` and `BookmarksPanelViewModel` tests use in-memory fakes throughout. The `FfmpegRunner` (used in nudge) is tested via a mock `IFfmpegRunner` that records calls without spawning a process.

The relocated `FfprobeRunner` tests are pure-unit (no binary, no DB) and slot directly into the `VideoIndexer.Tests` project with no additional fixtures required.


## Test Plan

### `BookmarkSettingsViewModelTests.cs` (App.Tests — existing file, new tests)

- `CanNudge_IsFalse_WhenCreateMode_FfmpegRunnerSet_VideoPathNull` — constructs with runner, no videoPath; asserts `CanNudge == false` — AC: nudge row hidden when path absent.
- `CanNudge_IsFalse_WhenEditMode_AndVideoPathProvided` — constructs in Edit mode with runner and path; asserts `CanNudge == false` — AC: nudge only in Create mode.
- `CanNudge_IsTrue_WhenCreateMode_FfmpegRunnerSet_VideoPathSet` — constructs with both; asserts `CanNudge == true` — AC: nudge row visible.
- `NudgeForwardCommand_NoOps_WhenNoFfmpegRunner` — command executes without throwing; `PositionMs` advances but no frame call made — AC: graceful no-op.
- `InitializeAsync_CapturesInitialFrame_WhenCanNudgeIsTrue` — mock `IFfmpegRunner` records the `ExtractFrameAsync` call; asserts called once with initial `PositionMs` — AC: preview shown on open.
- `InitializeAsync_DoesNotCaptureFrame_WhenCanNudgeIsFalse` — mock runner; no videoPath; asserts `ExtractFrameAsync` not called — AC: no spurious I/O.

### `BookmarksPanelViewModelTests.cs` (App.Tests — existing file, new tests)

- `AddBookmarkCommand_PassesVideoPath_ToSettingsService` — constructs `BookmarksPanelViewModel` with `videoPath = "/media/movie.mkv"`; executes `AddBookmarkCommand`; asserts `FakeBookmarkSettingsService.LastVideoPath == "/media/movie.mkv"` — AC: path threaded correctly.
- `AddBookmarkCommand_PassesNullVideoPath_WhenNotConstructedWithPath` — default construction (no videoPath); asserts `LastVideoPath == null` — AC: null path propagated correctly.

### `FfprobeRunnerTests.cs` (VideoIndexer.Tests — new file)

- `ResolveBinaryPath_ReturnsLibraryPath_WhenLibraryPathIsSet` — (relocated) — AC: path resolution priority.
- `ResolveBinaryPath_ReturnsProvisionerPath_WhenOnlyProvisionerPathIsSet` — (relocated) — AC: fallback to provisioner path.
- `ResolveBinaryPath_ReturnsFallback_WhenNeitherPathIsSet` — (relocated) — AC: ultimate fallback to binary name.


## Documentation Updates

- `docs/projects/rebuild/milestones/m5-filters-search.md` — set WP-016 to `Complete`; set milestone Status to `Complete`; update DSL Quick Reference: remove all previously-deferred identifiers (`HasTag`, `TagHasSubTags`, `HasRatedBookmarks`, `BookmarkContains`, `AmountBookmarks`) from their respective deferred lists — all are now active.
- `README.md` — Requirements: add Linux (x64) support.
- `CHANGELOG.md` — rename `[Unreleased]` to `[1.3.0] - 2026-06-01`; add M9 feature summary; remove resolved `### Technical Debt` subsection; add `### Changed` entry for `MovieEditorOptions` constructor consolidation.
- `docs/agents/project-manifest/api-surface.md` — update `IBookmarkSettingsService.ShowAsync` signature; update `BookmarksPanelViewModel` constructor.
- `docs/agents/project-manifest/file-tree.md` — add `FfprobeRunnerTests.cs` entry under `VideoIndexer.Tests`.
- `docs/agents/project-manifest/constraints.md` — add constraint: "Bookmark frame capture — initial frame is captured in `BookmarkSettingsViewModel.InitializeAsync` when `CanNudge` is true; nudge controls and the 'Add frame as thumbnail' checkbox are bound to `CanNudge`, not `IsCreateMode`."
- `docs/projects/rebuild/milestones/m9-images-player-bookmarks.md` — remove "Bookmark thumbnail frame capture" and "FfprobeRunner.ResolveBinaryPath test placement" from the Deferred Items table (both are resolved by this plan).
- `docs/agents/project-manifest/data-flows.md` — update section 19 (Bookmark Creation Flow): add `videoPath: _currentFilePath` to the `BookmarksPanelViewModel` constructor call; add `videoPath` to the `IBookmarkSettingsService.ShowAsync` call; add `ffmpegRunner` and `videoPath` to the `BookmarkSettingsViewModel` constructor call; add the initial frame capture substep inside `InitializeAsync`; remove the stale "videoPath is null in this impl" note.
- `src/VideoIndexer.Core/Filtering/FilterExpressionEvaluator.cs` — remove stale `M10 (Player & Bookmarks)` milestone annotations from `AmountBookmarks`, `HasRatedBookmarks()`, and `BookmarkContains()` doc comments, and the two inline `// M10 (Player & Bookmarks):` implementation comments.


## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **`_currentFilePath` is null for movies with no indexed file** | `BookmarksPanelViewModel` accepts `null`; `CanNudge` is false; no I/O attempted. Graceful degradation is the designed behaviour. |
| **Initial frame capture extends dialog open time** | Capture runs inside `InitializeAsync`, which is called before `ShowDialog`. The added latency is the single ffmpeg frame-extract (typically <500ms). No timeout needed — the existing `NudgeAsync` carries no timeout either. Logging a warning on failure is sufficient. |
| **`FakeBookmarkSettingsService.ShowAsync` signature break** | The new `videoPath` parameter is defaulted; the existing `ShowAsync` implementation compiles unchanged until the parameter is explicitly added. All callers in tests will continue to compile before the fake is updated. |
| **`CanNudge` must fire `OnPropertyChanged` if it ever becomes dynamic** | Currently it is set at construction time only and does not need change notification. If the video path is ever made mutable, `CanNudge` must call `OnPropertyChanged(nameof(CanNudge))` in the setter — note this as a future gotcha in constraints. |
