# Plan: M9 Images Rework 2 — Post-Synthesis Rework 1

## Plan Audit Cycles
- Audits: 1 — Plan Auditor v1.4.0
- Architectural Reviews: 1 — Plan Architect Reviewer v1.5.0

---

## Summary

This plan addresses all actionable items extracted from the synthesis report for
`2026-05-22-m9-images-rework-2`. Four concrete issues were identified: a priority-order
inversion in `FfprobeRunner.ResolveBinaryPath()` that silently ignores the user's ffprobe
override setting; a missing round-trip test in `SettingsViewModelTests` for the
`FfmpegOverridePath` ↔ `Library.FfmpegPath` wiring; a 22-parameter constructor on
`MovieEditorViewModel` that is approaching the boundary of maintainability; and an
unresolved database timestamp strategy where `IncrementClickAsync` uses `NOW()` (server
local time) while the project should standardise on `UTC_TIMESTAMP()`. M10 milestone
planning and the Release Engineer PR for `manifest.md` are delegated to their respective
agents and are explicitly out of scope here.

---

## Architectural Context

- **`FfprobeRunner`** — `src/VideoIndexer.Infrastructure/Library/FfprobeRunner.cs`; resolves
  the ffprobe binary via a three-step chain but currently evaluates the provisioner path
  (`ExternalTools.Ffmpeg.FfprobePath`) before the user override (`Library.FfprobePath`),
  the reverse of `FfmpegRunner`'s correct order. Documented as an open item in
  `constraints.md` (line ~169).
- **`FfprobeRunnerTests`** — `tests/VideoIndexer.Infrastructure.Tests/Library/FfprobeRunnerTests.cs`;
  contains one live-skip path-priority test (`ProbeAsync_UsesProvisionedPath_WhenBothPathsAreSet`)
  that validates the *wrong* (current, inverted) behaviour and must be rewritten.
- **`SettingsViewModel`** — `src/VideoIndexer.App/ViewModels/SettingsViewModel.cs`; maps
  `Library.FfmpegPath` ↔ `FfmpegOverridePath` in `PopulateFromSettings` and `Save()`, but
  `SettingsViewModelTests.cs` has no assertion covering this field, flagged in `constraints.md`
  (line ~170).
- **`MovieEditorViewModel`** — `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs`; the
  DI-facing constructor currently takes 22 parameters: two required (`Movie`, `IMovieRepository`)
  followed by 20 optional nullable services. The factory in `Program.cs` (lines 275–298) spells
  out every service explicitly.
- **`DapperBookmarkRepository`** — `src/VideoIndexer.Infrastructure/Database/DapperBookmarkRepository.cs`;
  `IncrementClickSql` uses `NOW()` (server local time) for the `last_clicked` column. No other
  SQL in the repository uses an explicit timestamp function; `InsertSql` relies on the DB column
  default for `added`.

---

## Approach / Architecture

**WP-A — FfprobeRunner priority fix:**
Swap the two `if` branches in `FfprobeRunner.ResolveBinaryPath()` so that
`Library.FfprobePath` (user override) is checked first, matching `FfmpegRunner`'s documented
three-step chain: user override → provisioner → PATH. Update the XML doc comment to reflect
the corrected order. Rewrite the one existing priority test (currently live-skip) and add a new
pure-unit test (no ffprobe binary required) that exercises `ResolveBinaryPath()` directly, in
the same style as `FfmpegRunnerTests.cs`.

**WP-B — SettingsViewModel round-trip tests:**
Add two `[Fact]` tests to `SettingsViewModelTests.cs` using the existing `BuildSut` helper:
one asserting `LoadAsync` maps `Library.FfmpegPath` → `FfmpegOverridePath`, the other
asserting `SaveCommand` maps `FfmpegOverridePath` back into `Library.FfmpegPath`.

**WP-C — MovieEditorOptions parameter-object:**
Extract a new `sealed record MovieEditorOptions` in the App layer
(`src/VideoIndexer.App/ViewModels/MovieEditorOptions.cs`) containing all 20 optional
nullable service parameters currently on the full constructor. The full constructor signature
shrinks to `(Movie movie, IMovieRepository movieRepository, MovieEditorOptions? options = null)`.
The parameterless design-time constructor is unchanged. The `Program.cs` factory builds one
`MovieEditorOptions` with all `sp.GetRequiredService<…>()` calls. Test callsites that pass
positional optional args are updated to pass a `new MovieEditorOptions(…)` with named
arguments.

**WP-D — UTC timestamp convention:**
Change the `last_clicked = NOW()` expression in `DapperBookmarkRepository.IncrementClickSql`
to `last_clicked = UTC_TIMESTAMP()`. Add a project-wide convention to `constraints.md`:
"All explicit timestamp writes in application SQL must use `UTC_TIMESTAMP()`, not `NOW()`".
No schema migration is required — only the in-process SQL constant changes.

---

## Rationale

- **WP-A:** The inverted priority silently ignores the user's explicit setting when a
  provisioner path exists. This is a behaviour defect, not a style issue. The fix is minimal
  (two-line swap) and self-evidently correct by symmetry with `FfmpegRunner`.
- **WP-B:** The missing test creates a blind spot around a user-facing General tab field.
  Adding the two round-trip tests directly follows the existing `SettingsViewModelTests.cs`
  pattern and closes a known constraint item with minimal effort.
- **WP-C:** The 22-parameter constructor makes the DI factory in `Program.cs` a wall of
  `GetRequiredService` calls that grows every milestone. A parameter-object (`MovieEditorOptions`)
  is the smallest shape that resolves this without changing runtime behaviour. An alternative —
  splitting `MovieEditorViewModel` into sub-ViewModels wired via a coordinator — was considered
  but rejected as a larger scope that should follow, not precede, a structural stabilisation step.
- **WP-D:** `UTC_TIMESTAMP()` is the standard for distributed systems and avoids bugs when the
  DB server's timezone differs from the application server's. The change is a one-line SQL
  constant update; no migration is needed.

---

## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|----------|--------------|-------------------------|-------------------|
| FfprobeRunner fix scope | Two-line branch swap; update doc + tests | Refactor both runners into a shared `BinaryResolver` helper | Shared helper is a good long-term investment but out of proportion for a two-line fix; defer to a future tools-layer refactor. |
| MovieEditorViewModel refactor shape | `MovieEditorOptions` record (all optionals) | (a) Keep constructor as-is. (b) Extract sub-ViewModel factories. (c) `MovieEditorOptions` with only a subset of optionals. | (a) Status-quo doesn't prevent further growth. (b) Sub-VM extraction is a milestone-scale change. (c) Partial grouping still needs re-work when the next parameter lands. Full grouping is the stable end-state. |
| UTC alignment scope | Fix explicit `NOW()` in IncrementClickSql only | (a) Full schema migration to change column defaults. (b) Leave as-is and track. | Column default change requires a m043 migration and a forced DB update for all users; disproportionate for a rarely-queried `last_clicked` column. The in-process `UTC_TIMESTAMP()` fix aligns new writes correctly without a deployment risk. |

---

## Pattern Alignment

| Pattern | Status | Source |
|---------|--------|--------|
| Three-step binary resolution (user override → provisioner → PATH) | Follows `FfmpegRunner.ResolveBinaryPath()` | `src/VideoIndexer.Infrastructure/Library/FfmpegRunner.cs` lines 101–114 |
| Pure-unit `ResolveBinaryPath` tests via `internal` accessor | Follows `FfmpegRunnerTests.cs` pattern | `tests/VideoIndexer.Tests/FfmpegRunnerTests.cs` |
| `SettingsViewModel` round-trip tests using `FakeSettingsService` | Follows existing `SaveCommand_ProducesUpdatedAppOptions` test | `tests/VideoIndexer.App.Tests/SettingsViewModelTests.cs` |
| Parameter-object record in App-layer `ViewModels/` folder | New pattern — justified by constructor size; analogous to `BookmarkBrowserQuery` in Core | `src/VideoIndexer.Core/Models/` |
| `sealed record` for options/query objects | Follows `BookmarkBrowserQuery`, `CoverImageTransform`, etc. | `src/VideoIndexer.Core/Models/` |
| `UTC_TIMESTAMP()` SQL convention | Formalises a new constraint; consistent with standard distributed-system practice | n/a (new constraint) |

---

## Detailed Steps

### WP-A: FfprobeRunner Priority Fix

1. **Fix `ResolveBinaryPath()`** in `src/VideoIndexer.Infrastructure/Library/FfprobeRunner.cs`:
   - Move the `Library.FfprobePath` check to be the *first* branch (user override).
   - Move the `ExternalTools.Ffmpeg.FfprobePath` check to be the *second* branch (provisioner).
   - Update the XML doc comment list items to reflect the corrected order: (1) user override,
     (2) provisioner, (3) PATH.

2. **Rewrite the priority test** in
   `tests/VideoIndexer.Infrastructure.Tests/Library/FfprobeRunnerTests.cs`:
   - Rename `ProbeAsync_UsesProvisionedPath_WhenBothPathsAreSet` to
     `ProbeAsync_UsesLibraryOverridePath_WhenBothPathsAreSet`.
   - Adjust the test body so the *library path* gets the real ffprobe and the *provisioned path*
     gets the non-existent binary. The assertion (still `InvalidOperationException` from probing
     a nonexistent file) confirms the library override was used rather than falling back to the
     provisioner.

3. **Add pure-unit path-resolution tests** (no ffprobe binary, no `SkippableFact`):
   - `ResolveBinaryPath_ReturnsLibraryPath_WhenLibraryPathIsSet` — library path set, provisioner
     path also set; asserts the returned string equals the library path.
   - `ResolveBinaryPath_ReturnsProvisionerPath_WhenOnlyProvisionerPathIsSet` — only provisioner
     path set; asserts the returned string equals the provisioner path.
   - `ResolveBinaryPath_ReturnsFallback_WhenNeitherPathIsSet` — both paths null/empty; asserts
     the returned string is `"ffprobe"`.
   - Make `ResolveBinaryPath()` `internal` (change from `private`) in `FfprobeRunner.cs` so the
     tests can call it directly (mirrors `FfmpegRunner.ResolveBinaryPath()` which is already
     `internal`).

4. **Update `constraints.md`**: remove the "open item" paragraph about the priority inversion
   and replace it with a confirmation that `FfprobeRunner` and `FfmpegRunner` now share the
   same three-step resolution order.

### WP-B: SettingsViewModel Round-Trip Tests

5. **Add two tests** to `tests/VideoIndexer.App.Tests/SettingsViewModelTests.cs`:
   - `LoadAsync_MapsLibraryFfmpegPath_To_FfmpegOverridePath` — initialise options with a
     non-null `Library.FfmpegPath`; call `LoadAsync`; assert `Sut.FfmpegOverridePath` equals
     the value.
   - `SaveCommand_Maps_FfmpegOverridePath_Back_To_LibraryFfmpegPath` — call `LoadAsync`; set
     `Sut.FfmpegOverridePath` to a new path; execute `SaveCommand`; assert
     `SettingsService.LastSaved.Library.FfmpegPath` equals the new path.

6. **Update `constraints.md`**: remove the "open item" paragraph about the missing
   `SettingsViewModelTests` coverage for `FfmpegOverridePath` (now resolved).

### WP-C: MovieEditorOptions Parameter-Object

7. **Create `src/VideoIndexer.App/ViewModels/MovieEditorOptions.cs`** — new file containing:
   ```csharp
   public sealed record MovieEditorOptions(
       INameParser?                       NameParser               = null,
       ILabelCleanerService?              LabelCleanerService      = null,
       ISettingsService?                  SettingsService          = null,
       IMoviePropertiesService?           MoviePropertiesService   = null,
       IAppPaths?                         AppPaths                 = null,
       IFileLauncherService?              FileLauncherService      = null,
       ITagsManager?                      TagsManager              = null,
       ITagEditorService?                 TagEditorService         = null,
       ITagMergeService?                  TagMergeService          = null,
       IGrantsManagementService?          GrantsManagementService  = null,
       ICategoryEditorService?            CategoryEditorService    = null,
       IThumbnailRepository?              ThumbnailRepository      = null,
       ICoverImageService?                CoverImageService        = null,
       IThumbnailGenerationService?       ThumbnailGenerationService = null,
       IThumbnailViewerService?           ThumbnailViewerService   = null,
       IFfprobeRunner?                    FfprobeRunner            = null,
       Func<PlayerViewModel>?             PlayerFactory            = null,
       IBookmarkRepository?               BookmarkRepository       = null,
       IBookmarkSettingsService?          BookmarkSettingsService  = null,
       ILogger<BookmarksPanelViewModel>?  BookmarksPanelLogger     = null);
   ```

8. **Refactor `MovieEditorViewModel`'s full constructor**:
   - Replace all 20 optional parameters with `MovieEditorOptions? options = null`.
   - Unpack the record properties into private fields using `options?.PropertyName ?? null`
     (or `?? NullLogger<BookmarksPanelViewModel>.Instance` for the logger field).
   - All private field names and assignment logic remain identical — only the constructor
     signature and unpacking code changes.

9. **Update `Program.cs`** factory (lines 275–298):
   - Construct `new MovieEditorOptions(NameParser: sp.GetRequiredService<INameParser>(), …)`
     with all 20 services as named arguments.
   - Pass the options object as the third argument to `new MovieEditorViewModel(movie, repo, options)`.

10. **Update `MovieEditorViewModelTests.cs`** factory helpers and **`LabelCleanerViewModelTests.cs`**
    direct constructions — every callsite that passes positional or named optional arguments to
    `MovieEditorViewModel` must be migrated to `new MovieEditorOptions(…)`:

    *In `tests/VideoIndexer.App.Tests/MovieEditorViewModelTests.cs`:*
    - `BuildSutWithCleanLabel` — replace positional `nameParser, labelCleaner` args with
      `new MovieEditorOptions(NameParser: nameParser, LabelCleanerService: labelCleaner)`.
    - `BuildSutWithCleanLabelAndSettings` — wrap the four positional optional args in
      `new MovieEditorOptions(NameParser: …, LabelCleanerService: …, SettingsService: …)`.
    - `BuildSutWithShowProperties` (approx. line 579) — uses named optional-parameter syntax;
      wrap all supplied services into a `MovieEditorOptions` instance.
    - `BuildSutWithTagger` (approx. line 632) — uses named optional-parameter syntax;
      wrap all supplied services into a `MovieEditorOptions` instance.
    - Inline construction at approx. line 694 — direct `new MovieEditorViewModel(…)` with
      positional arguments; replace the optional positional args with `new MovieEditorOptions(…)`.
    - `BuildSutWithBookmarks` (approx. line 724) — uses named optional-parameter syntax;
      wrap all supplied services into a `MovieEditorOptions` instance.
    - The zero-extra-arg `BuildSut` helper requires no change.

    *In `tests/VideoIndexer.App.Tests/LabelCleanerViewModelTests.cs`:*
    - Direct positional construction at approx. line 225:
      `new MovieEditorViewModel(movie, repo, parser, labelSvc)` — replace the two trailing
      positional optional args with `new MovieEditorOptions(NameParser: parser, LabelCleanerService: labelSvc)`.
    - Direct positional construction at approx. line 255: same pattern as above.

11. **Update manifest docs**:
    - `file-tree.md` — add an annotation entry for `MovieEditorOptions.cs` inside the
      `ViewModels/` subtree.
    - `api-surface.md` — add a `MovieEditorOptions` record signature block in the App-layer
      section; update the `MovieEditorViewModel` constructor signature.

### WP-D: UTC Timestamp Convention

12. **Update `DapperBookmarkRepository.cs`** — change:
    ```csharp
    private const string IncrementClickSql =
        "UPDATE movies_bookmarks SET click_count = click_count + 1, last_clicked = NOW() "
        + "WHERE bookmark_id = @BookmarkId";
    ```
    to:
    ```csharp
    private const string IncrementClickSql =
        "UPDATE movies_bookmarks SET click_count = click_count + 1, last_clicked = UTC_TIMESTAMP() "
        + "WHERE bookmark_id = @BookmarkId";
    ```

13. **Update `api-surface.md`** — in the `IBookmarkRepository.IncrementClickAsync` doc block,
    change "sets LastClicked to current server time (NOW())" to "sets LastClicked to
    `UTC_TIMESTAMP()` (UTC)".

14. **Update `constraints.md`**:
    - Remove the existing `NOW()` vs `UTC_TIMESTAMP()` note from the open items section (if
      present) and replace it with a convention rule in the **Database** section:
      "All explicit timestamp writes in application SQL must use `UTC_TIMESTAMP()`, not
      `NOW()`. Column defaults defined in DDL/migrations are outside application control and
      are not required to match; only `UPDATE`/`INSERT` statements authored in application
      code are subject to this rule."

---

## Dependencies

- WP-A has no dependencies on other WPs in this plan.
- WP-B has no dependencies on other WPs in this plan.
- WP-C must be completed before the build can pass (it changes the primary constructor
  signature); WP-C step 9 (Program.cs) and step 10 (tests) depend on step 7 (record
  definition) and step 8 (constructor refactor).
- WP-D has no dependencies on other WPs in this plan.
- All four WPs are independent of each other and can be implemented in any order.

---

## Required Components

### Modified Files
- `src/VideoIndexer.Infrastructure/Library/FfprobeRunner.cs`
- `src/VideoIndexer.Infrastructure/Database/DapperBookmarkRepository.cs`
- `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs`
- `src/VideoIndexer.App/Program.cs`
- `tests/VideoIndexer.Infrastructure.Tests/Library/FfprobeRunnerTests.cs`
- `tests/VideoIndexer.App.Tests/SettingsViewModelTests.cs`
- `tests/VideoIndexer.App.Tests/MovieEditorViewModelTests.cs`
- `tests/VideoIndexer.App.Tests/LabelCleanerViewModelTests.cs`

### New Files
- `src/VideoIndexer.App/ViewModels/MovieEditorOptions.cs` (WP-C)

### Documentation Files
- `docs/agents/project-manifest/file-tree.md`
- `docs/agents/project-manifest/api-surface.md`
- `docs/agents/project-manifest/constraints.md`

---

## Assumptions

- `FfmpegRunner.ResolveBinaryPath()` is already `internal`; `FfprobeRunner.ResolveBinaryPath()`
  is currently `private` and must be promoted to `internal` as part of WP-A to support
  direct unit tests.
- The `added` column in `movies_bookmarks` is populated by a DB-side column default (`NOW()`)
  that was defined in the original legacy DDL, not by application code. This default is out of
  scope for WP-D.
- `FakeSettingsService.LastSaved` already captures the most-recently saved `AppOptions` record,
  consistent with its usage in the existing `SaveCommand_ProducesUpdatedAppOptions` test.
- `LibraryOptions` already has a `FfprobePath` property (confirmed in
  `FfprobeRunnerTests.cs` `BuildSut` helper — line 31).
- All 20 optional services currently injected into `MovieEditorViewModel` remain required in
  production; the `MovieEditorOptions` record just packages them — no services are dropped.

---

## Constraints

- `TreatWarningsAsErrors=true` — the constructor refactor must not introduce any nullability
  warnings. All optional fields that are currently assigned `?? NullLogger<…>.Instance` must
  retain that fallback.
- **Core has no external NuGet dependencies** — `MovieEditorOptions` lives in the App layer;
  it must not be placed in Core (it references App-layer types like `IThumbnailViewerService`
  and `IBookmarkSettingsService`).
- No new NuGet packages are introduced by any WP in this plan.
- No database schema migration is introduced by WP-D.
- `test.ps1` unit and app suites must remain green (`.\test.ps1` = no failures).

---

## Out of Scope

- M10 milestone planning (Filter DSL operators, bookmark thumbnail generation).
- Release Engineer PR for `manifest.md`.
- Refactoring `FfmpegRunner` and `FfprobeRunner` into a shared `BinaryResolver` helper.
- Splitting `MovieEditorViewModel` into sub-ViewModels or introducing a coordinator.
- Adding a schema migration (m043) to change the DB-side `DEFAULT NOW()` for timestamp columns.
- Any changes to how the `added` column is populated on bookmark insert.
- `FfprobeOverridePath` Settings UI field (adding a UI TextBox analogous to
  `FfmpegOverridePath` is a separate UX concern and should be planned in M10).

---

## Acceptance Criteria

- **AC-A1** `FfprobeRunner.ResolveBinaryPath()` returns `Library.FfprobePath` when that
  value is non-empty, regardless of whether `ExternalTools.Ffmpeg.FfprobePath` is also set.
- **AC-A2** `FfprobeRunner.ResolveBinaryPath()` returns `ExternalTools.Ffmpeg.FfprobePath`
  when `Library.FfprobePath` is null/empty but the provisioner path is set.
- **AC-A3** `FfprobeRunner.ResolveBinaryPath()` returns `"ffprobe"` when both paths are
  null/empty.
- **AC-B1** `SettingsViewModelTests` has a test proving `LoadAsync` maps
  `Library.FfmpegPath` to `FfmpegOverridePath`.
- **AC-B2** `SettingsViewModelTests` has a test proving `SaveCommand` maps
  `FfmpegOverridePath` back to `Library.FfmpegPath`.
- **AC-C1** `MovieEditorViewModel`'s full DI constructor accepts `(Movie, IMovieRepository,
  MovieEditorOptions?)` — maximum 3 parameters.
- **AC-C2** `Program.cs` factory compiles cleanly and no runtime `InvalidOperationException`
  is thrown when the application starts.
- **AC-C3** All existing `MovieEditorViewModelTests` pass without modification to their test
  assertions (only factory-helper callsites are updated).
- **AC-D1** `DapperBookmarkRepository.IncrementClickSql` contains `UTC_TIMESTAMP()` instead
  of `NOW()`.
- **AC-ALL** `.\test.ps1` (unit + app suites) completes with zero failures and the Release
  build (`dotnet build -c Release`) reports zero warnings and zero errors.

---

## Testing Strategy

WP-A adds three pure-unit tests (no external binary needed) for the three resolution branches
and rewrites the one existing live-skip test to validate the corrected priority. WP-B adds two
`[Fact]` tests to the existing `SettingsViewModelTests`. WP-C is a refactor — the test
obligation is that all existing `MovieEditorViewModelTests` continue to pass after callsite
updates. WP-D has no testable unit boundary (the SQL constant change is verified by inspection
and integration test if a live DB is available). Full regression is confirmed via `.\test.ps1`.

---

## Test Plan

- `tests/VideoIndexer.Infrastructure.Tests/Library/FfprobeRunnerTests.cs` —
  `ResolveBinaryPath_ReturnsLibraryPath_WhenLibraryPathIsSet` (pure unit) — AC-A1
- `tests/VideoIndexer.Infrastructure.Tests/Library/FfprobeRunnerTests.cs` —
  `ResolveBinaryPath_ReturnsProvisionerPath_WhenOnlyProvisionerPathIsSet` (pure unit) — AC-A2
- `tests/VideoIndexer.Infrastructure.Tests/Library/FfprobeRunnerTests.cs` —
  `ResolveBinaryPath_ReturnsFallback_WhenNeitherPathIsSet` (pure unit) — AC-A3
- `tests/VideoIndexer.Infrastructure.Tests/Library/FfprobeRunnerTests.cs` —
  `ProbeAsync_UsesLibraryOverridePath_WhenBothPathsAreSet` (live-skip) — AC-A1
- `tests/VideoIndexer.App.Tests/SettingsViewModelTests.cs` —
  `LoadAsync_MapsLibraryFfmpegPath_To_FfmpegOverridePath` — AC-B1
- `tests/VideoIndexer.App.Tests/SettingsViewModelTests.cs` —
  `SaveCommand_Maps_FfmpegOverridePath_Back_To_LibraryFfmpegPath` — AC-B2
- `tests/VideoIndexer.App.Tests/MovieEditorViewModelTests.cs` —
  All existing tests (no new tests; regression validation only) — AC-C3

---

## Documentation Updates

- `docs/agents/project-manifest/constraints.md` —
  (a) Remove "FfprobeRunner priority inversion" open item; add confirmation that both runners
  share the three-step order.
  (b) Remove "missing FfmpegOverridePath test" open item.
  (c) Add UTC timestamp convention rule to the Database section.
- `docs/agents/project-manifest/api-surface.md` —
  (a) Add `MovieEditorOptions` record signature block in the App-layer ViewModels section.
  (b) Update `MovieEditorViewModel` constructor signature to `(Movie, IMovieRepository, MovieEditorOptions?)`.
  (c) Update `IBookmarkRepository.IncrementClickAsync` doc note: `NOW()` → `UTC_TIMESTAMP()`.
- `docs/agents/project-manifest/file-tree.md` —
  Add annotation for `MovieEditorOptions.cs` in the `ViewModels/` subtree entry.

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **`MovieEditorViewModel` refactor breaks DI factory at runtime** | Run `dotnet build -c Release` and `.\test.ps1` after WP-C. The app-startup test in `VideoIndexer.App.Tests` exercises the DI container and will catch missing registrations. |
| **`FfprobeRunner.ResolveBinaryPath()` accessibility change breaks InternalsVisibleTo** | Check `AssemblyInfo.cs` / `Directory.Build.props` for `[InternalsVisibleTo]` attributes before promoting to `internal`. The Infrastructure test project already has access to `FfmpegRunner`'s internal method so the attribute should already be present. |
| **UTC_TIMESTAMP change breaks integration tests** | Integration tests for `IncrementClickAsync` are live-DB gated (`SkippableFact`). No unit test covers the SQL constant. The risk is a regression detectable only on a live DB; accepted given the trivial nature of the change. |
| **`MovieEditorOptions` record with nullable members triggers new compiler warnings** | All 20 members are explicitly nullable reference types; the record primary constructor syntax is fully supported under `Nullable=enable`. No additional null-analysis issues are expected. |
