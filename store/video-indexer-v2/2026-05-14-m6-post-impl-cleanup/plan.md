# Plan

# M6 Post-Implementation Cleanup

## Summary

Ten actionable carry-forward items were identified by the M6 Synthesis report and its Strategic Recommendations. None of the items introduce new user-visible features; all resolve code-quality, correctness, security, or test-coverage gaps surfaced during M6 implementation and review. The items span four categories: (1) test-infrastructure deduplication, (2) defensive-code and bug fixes, (3) security tightening, and (4) missing test coverage. One item (`GetOriginalFilenameAsync` determinism, Recommendation #4) requires a database schema migration and is the only dependency-bearing piece of work.


## Architectural Context

The codebase follows a strict Core → Infrastructure → App layering enforced at the project-reference level. All cross-layer calls go through Core interfaces (`IMovieRepository`, `ISettingsService`, `INameParser`, etc.); DI wiring lives exclusively in `Program.cs`.

Relevant modules and files:

- **`src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs`** — `LoadFilterSlotsAsync` (Rec #1)
- **`src/VideoIndexer.App/ViewModels/MainContentViewModel.cs`** — `OnEditorCloseRequested` (Rec #5)
- **`src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs`** — `CleanLabel` command, `LabelCleanerOptions` construction (Rec #7)
- **`src/VideoIndexer.App/Services/WindowsFileLauncherService.cs`** — `OpenFolder` / `ShowInExplorer` (Rec #6)
- **`src/VideoIndexer.App/Converters/IntDecimalConverter.cs`** — converter under test (Rec #9)
- **`src/VideoIndexer.Infrastructure/Library/DapperMovieRepository.cs`** — `GetOriginalFilenameAsync` (Rec #4)
- **`src/VideoIndexer.Core/Abstractions/IMovieRepository.cs`** — interface contract for #4
- **`src/VideoIndexer.Core/Abstractions/ISettingsService.cs`** — used for Rec #7
- **`src/VideoIndexer.Core/Options/AppOptions.cs`** — `LabelCleaner` property (Rec #7)
- **`tests/VideoIndexer.App.Tests/TestHelpers/FakeMovieRepository.cs`** — `GetOriginalFilenameAsync` always returns null (Rec #8)
- **`tests/VideoIndexer.App.Tests/MovieEditorViewModelTests.cs`** — missing `MoveActorDown_IsNoOp_WhenNoActorSelected` (Rec #10)
- **`tests/VideoIndexer.Infrastructure.Tests/Library/DapperMovieRepositoryTests.cs`** — `NonDisposingConnection` duplicate (Rec #3)
- **`tests/VideoIndexer.Infrastructure.Tests/Library/DapperMovieCatalogRepositoryTests.cs`** — `NonDisposingConnection` duplicate (Rec #3)
- **`tests/VideoIndexer.Infrastructure.Tests/Library/DapperFilterSlotRepositoryTests.cs`** — `NonDisposingConnection` duplicate (Rec #3)
- **`tests/VideoIndexer.Infrastructure.Tests/Library/LibraryScannerIntegrationTests.cs`** — `NonDisposingConnection` duplicate (Rec #3)
- **`docs/agents/project-manifest/constraints.md`** — sentinel test maintenance (Rec #2)


## Approach / Architecture

All work is confined to existing abstractions. No new Core interfaces, new NuGet packages, or new architectural layers are introduced. The plan groups the ten recommendations into six work packages ordered by dependency:

1. **WP-001 — Test infrastructure consolidation** (Rec #3): Extract `NonDisposingConnection` and `SharedConnectionFactory` into a single shared file in `VideoIndexer.Infrastructure.Tests`; update four test files to use the shared type. No production code changes.
2. **WP-002 — Defensive code fixes** (Rec #1, #5): `LoadFilterSlotsAsync` gets internal `try/catch`; `OnEditorCloseRequested` gets a `CurrentPage` guard. Both fixes are accompanied by tests.
3. **WP-003 — Security: `WindowsFileLauncherService` path quoting** (Rec #6): Fix `OpenFolder` to quote the path argument; migrate `ShowInExplorer` to `ProcessStartInfo.ArgumentList` collection syntax to eliminate manual quoting risk entirely.
4. **WP-004 — Schema migration m039 + query fix** (Rec #4): Add a `created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP` column to `movies_filenames`, bump schema revision to 39, and add `ORDER BY mf.created_at ASC` to `GetOriginalFilenameAsync`. Update the integration test and the sentinel test constant.
5. **WP-005 — Test coverage gaps** (Rec #8, #9, #10): Add `SetOriginalFilename()` to `FakeMovieRepository` and a corresponding `CleanLabel` primary-path test; add `IntDecimalConverterTests.cs`; add `MoveActorDown_IsNoOp_WhenNoActorSelected`.
6. **WP-006 — `LabelCleanerOptions` persistence** (Rec #7): Inject `ISettingsService` into `MovieEditorViewModel`; read `_settingsService.Current.LabelCleaner` as the initial `LabelCleanerOptions` when opening the Label Cleaner dialog; save the accepted options back via `SaveAsync` so the user's toggles persist across invocations. Add `FakeSettingsService` test helper and tests.

Recommendation #2 (sentinel test contract) is addressed as a documentation-only step inside WP-004 (the natural home for schema-revision changes) by adding a bullet to `constraints.md`.


## Rationale

- All ten items were explicitly surfaced by the M6 reviewer and synthesiser; none are speculative.
- The `NonDisposingConnection` consolidation (WP-001) is sequenced first because it has zero production-code risk and makes subsequent test files cleaner when WP-005 adds new integration tests.
- `GetOriginalFilenameAsync` determinism (WP-004) requires a schema migration and must therefore be done as a standalone WP with its own sentinel-test update; sequencing it before WP-005 means the new test for the primary `CleanLabel` path will run against a deterministic query.
- `LabelCleanerOptions` persistence (WP-006) is sequenced last because it requires a new `FakeSettingsService` test helper; placing it after WP-005 avoids having two WPs add helpers to the same test project simultaneously.
- `WindowsFileLauncherService` changes (WP-003) are isolated to a single file with no DB or DI side-effects, making them safe to deliver as a small standalone WP.


## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|----------|--------------|-------------------------|-------------------|
| `NonDisposingConnection` extraction target | New shared file `TestHelpers/SharedTransactionalConnectionFactory.cs` in `VideoIndexer.Infrastructure.Tests` | Shared base class for all suites | A static helper / inner classes file avoids forcing all test classes to share a common base class hierarchy, which would complicate the IClassFixture pattern already in use. |
| `LabelCleanerOptions` save-back timing | Save on Accept (inside `CleanLabelCommand`, after the dialog returns) | Save lazily on VM disposal or Save only on explicit user preference action | Accept is the natural affordance for "I want these settings"; saves immediately when the user acts intentionally. Disposal save would silently persist Discard. |
| `ProcessStartInfo.ArgumentList` vs manual quoting | `ArgumentList` for both `OpenFolder` and `ShowInExplorer` | Quote-wrap the raw path string | `ArgumentList` delegates escaping to the runtime and is idiomatic .NET; manual quoting introduces brittleness if paths contain nested quotes. |
| Schema migration approach for `created_at` | New migration file `m039_add_filenames_created_at.sql` | Amend m038 in-place | Amending deployed migrations is prohibited; a new migration file is the only safe approach. |


## Pattern Alignment

| Pattern | Status |
|---------|--------|
| `try/catch` inside async VM methods mirroring `LoadAsync` — `src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs` | Followed (WP-002) |
| Guard-clause early returns at top of event handlers — e.g. `ShowLibraryFolders()` checks `CurrentPage == LibraryFolders` before proceeding | Followed (WP-002, OnEditorCloseRequested) |
| Idempotent schema migrations using `ADD COLUMN IF NOT EXISTS` — established in m036–m038 | Followed (WP-004) |
| `DatabaseBootstrapper.ExpectedRevision` arithmetic in tests — established by M6 WP-002 fix-forward | Followed (WP-004, sentinel test update) |
| `AppOptions` immutability — `with { }` + `ISettingsService.SaveAsync` — `src/VideoIndexer.Core/Options/AppOptions.cs` | Followed (WP-006) |
| Test helper `FakeXxx` pattern — e.g. `FakeMovieRepository`, `FakeRefreshOrchestrator` in `tests/VideoIndexer.App.Tests/TestHelpers/` | Followed (WP-005, WP-006) |
| `ProcessStartInfo.ArgumentList` for shell invocations — no prior use in this codebase | New pattern, justified by security recommendation |


## Detailed Steps

### WP-001 — Test Infrastructure: `NonDisposingConnection` Consolidation

1. Create `tests/VideoIndexer.Infrastructure.Tests/TestHelpers/SharedTransactionalConnectionFactory.cs` containing:
   - `internal sealed class NonDisposingConnection : IDbConnection` — wraps `MySqlConnection` + `MySqlTransaction`; proxies all members; `Dispose()` is a no-op.
   - `internal sealed class SharedConnectionFactory : IDbConnectionFactory` — returns the shared `NonDisposingConnection` from `CreateOpenConnectionAsync`.
2. Remove the private `NonDisposingConnection` inner class from `DapperMovieRepositoryTests.cs`, `DapperMovieCatalogRepositoryTests.cs`, `DapperFilterSlotRepositoryTests.cs`, and `LibraryScannerIntegrationTests.cs`. Update each file to reference the shared type.
3. Remove the distinct `NonDisposingConnectionWrapper` inner class from `SpdbConfigRepositoryTests.cs`. Unify it with the transaction-aware variant by widening the shared class constructor to `(IDbConnection inner, IDbTransaction? transaction = null)`. The four transaction-aware callers pass a non-null `MySqlTransaction`; `SpdbConfigRepositoryTests.cs` passes `null`. This eliminates the need for two distinct types in the shared file. **Also update `SpdbConfigRepositoryTests.cs`'s `SingleConnectionFactory.CreateOpenConnectionAsync` to return `new NonDisposingConnection(_connection, null)` instead of `new NonDisposingConnectionWrapper(_connection)` — failing to do this produces a compile error because `NonDisposingConnectionWrapper` no longer exists after this step.**
4. Verify the build is clean and all tests still pass.

### WP-002 — Defensive Code Fixes

**`LoadFilterSlotsAsync` internal error handling:**

5. In `MoviesListViewModel.LoadFilterSlotsAsync`, wrap the repository calls in a `try/catch (Exception ex) when (ex is not OperationCanceledException)` block. On catch, log the exception via the existing `_logger` field using `LogError`, clear `FilterSlots`, and set `ActiveFilterSlot = null` (same safe-state outcome as a repository returning empty). Mirror the pattern used by `LoadAsync` in the same class, which uses the same cancellation-safe `when` guard.
6. Before writing the test, perform the following prerequisite fixes:
   - **Extend `FakeFilterSlotRepository`** with a `SetException(Exception? ex)` method and conditional-throw logic in `GetAllAsync`, mirroring the exception-injection pattern in `FakeMovieRepository` (`tests/VideoIndexer.App.Tests/TestHelpers/FakeMovieRepository.cs`).
   - **Extend `MoviesListViewModelTests.cs::BuildSut`** to accept optional `FakeFilterSlotRepository?` and `FakeActiveFilterSlotRepository?` parameters and pass them to the `MoviesListViewModel` constructor, following the pattern in `MoviesListViewModelSearchFilterTests.cs::BuildSut`. Without this change `LoadFilterSlotsAsync` returns at its null-guard before any repository call, making the test impossible.

   Then add the unit test: `LoadFilterSlotsAsync_WhenRepositoryThrows_DoesNotPropagateException` — configure `FakeFilterSlotRepository.SetException` to throw, assert no exception escapes the method and `FilterSlots` is empty.

**`OnEditorCloseRequested` `CurrentPage` guard:**

7. In `MainContentViewModel.OnEditorCloseRequested`, perform the event unsubscription first (unconditionally), then add the early-return guard: `if (CurrentPage != MainContentPage.MovieEditor) return;` before the `CloseSubView()` call. Placing the guard *before* the unsubscription would leave the handler registered on the `MovieEditorViewModel` whenever the guard fires, preventing GC of the editor VM and risking a second spurious invocation. The correct order is: (1) `if (sender is MovieEditorViewModel editorVm) editorVm.CloseRequested -= OnEditorCloseRequested;` — then (2) `if (CurrentPage != MainContentPage.MovieEditor) return;` — then (3) `CloseSubView()`.
8. Add a unit test to `MainContentViewModelTests.cs`: `OnEditorCloseRequested_WhenCurrentPageIsNotMovieEditor_DoesNotCloseSubView` — construct the SUT in a non-MovieEditor state (e.g. `MainContentPage.Default`), fire `CloseRequested` manually, and assert `CurrentPage` remains `Default` and `CurrentChildViewModel` is unchanged.

### WP-003 — Security: `WindowsFileLauncherService` Path Quoting

9. Rewrite `OpenFolder` to use `ProcessStartInfo.ArgumentList` with a single argument equal to `path`. Remove the `Arguments` property assignment.
10. Rewrite `ShowInExplorer` to use `ProcessStartInfo.ArgumentList` with a single entry: `ArgumentList.Add($"/select,{filePath}")`. Remove the `Arguments` property assignment. **Do not use two separate `ArgumentList` entries** (`/select,` then `filePath`) — `explorer.exe` treats the argument list as space-separated tokens, so splitting produces `"/select," "C:\path"` which explorer cannot parse, silently opening the folder without selecting the file.
11. Update `docs/agents/project-manifest/constraints.md`: add a bullet under the existing **UI & MVVM** section (not a new top-level section) documenting that `WindowsFileLauncherService` uses `ProcessStartInfo.ArgumentList` to avoid path-injection risk.

### WP-004 — Schema Migration m039 + `GetOriginalFilenameAsync` Determinism

12. Create `src/VideoIndexer.Infrastructure/Database/Migrations/m039_add_filenames_created_at.sql`:
    ```sql
    ALTER TABLE movies_filenames
        ADD COLUMN IF NOT EXISTS created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP;
    UPDATE spdb_config SET config_value = '39' WHERE config_name = 'db_revision';
    ```
    Include rollback comment:
    ```sql
    -- Rollback: ALTER TABLE movies_filenames DROP COLUMN IF EXISTS created_at;
    --           UPDATE spdb_config SET config_value = '38' WHERE config_name = 'db_revision';
    ```
13. Increment `DatabaseBootstrapper.ExpectedRevision` from `38` to `39`. (No migration-registration step is required — `DatabaseBootstrapper` checks the revision constant only; it has no migration runner. The SQL file from step 12 and the constant bump in this step are the complete production-code changes for WP-004.)
14. Update `DapperMovieRepository.GetOriginalFilenameAsync` to add `ORDER BY mf.created_at ASC` before `LIMIT 1`.
15. Update the sentinel test in `DatabaseBootstrapper` tests: change `Assert.Equal(38, ...)` to `Assert.Equal(39, ...)` (leave the intentional-hardcoding comment in place).
16. Add an integration test to `DapperMovieRepositoryTests.cs`: `GetOriginalFilenameAsync_WithMultipleFilenames_ReturnsEarliest` — insert two filenames for the same movie with different `created_at` values (using an explicit INSERT with a fixed timestamp for the older one), assert the method returns the earlier filename.
17. Update `constraints.md`: bump the "Expected schema revision" entry to `39`; add rollback procedure for m039; add the sentinel test maintenance bullet explicitly stating "this literal must be updated on every schema revision bump; it is intentional and must not be replaced with an expression".

### WP-005 — Test Coverage Gaps

18. Confirm that `tests/VideoIndexer.Infrastructure.Tests/TestHelpers/SharedTransactionalConnectionFactory.cs` (introduced in WP-001) is in place and that WP-001 Step 3's `SingleConnectionFactory` update in `SpdbConfigRepositoryTests.cs` is complete before proceeding; a clean build is required before new test code is added.

**`FakeMovieRepository.SetOriginalFilename` + `CleanLabel` primary path:**

19. Add to `FakeMovieRepository`:
    ```csharp
    private string? _originalFilename;
    public void SetOriginalFilename(string? filename) => _originalFilename = filename;
    ```
    Update `GetOriginalFilenameAsync` to return `_originalFilename` instead of always `null`.
20. Add a test to `MovieEditorViewModelTests.cs` (or a dedicated `CleanLabelTests` region): `CleanLabelCommand_UsesOriginalFilenameFromRepository_AsRawInput` — set a filename on the fake repo, execute `CleanLabelCommand`, assert `LabelCleanerViewModel` received the `Path.GetFileNameWithoutExtension` of that filename as its initial text via a `FakeLabelCleanerService` or the existing test infrastructure.

**`IntDecimalConverter` unit tests:**

21. Create `tests/VideoIndexer.App.Tests/Converters/IntDecimalConverterTests.cs` with:
    - `Convert_WithInt_ReturnsDecimal` — assert `(decimal?)5` for input `5`.
    - `Convert_WithNull_ReturnsNull` — assert `null` for `null` input.
    - `Convert_WithNonInt_ReturnsNull` — assert `null` for a `string` input.
    - `ConvertBack_WithDecimal_ReturnsInt` — assert `5` for input `5.0m` with `targetType = typeof(int)`.
    - `ConvertBack_WithDecimal_ReturnsNullableInt` — assert `(int?)5` for input `5.0m` with `targetType = typeof(int?)`.
    - `ConvertBack_WithNull_ReturnsZero_ForNonNullableInt` — assert `0` for `null` input with `targetType = typeof(int)`.
    - `ConvertBack_WithNull_ReturnsNull_ForNullableInt` — assert `null` for `null` input with `targetType = typeof(int?)`.

**`MoveActorDown` null-guard test:**

22. Add to `MovieEditorViewModelTests.cs`:
    `MoveActorDown_IsNoOp_WhenNoActorSelected` — construct a VM with actors loaded, ensure `SelectedActor` is `null`, call `MoveActorDownCommand.Execute(null)`, assert actor list order is unchanged and no exception is thrown.

### WP-006 — `LabelCleanerOptions` Persistence

23. **Extend the existing** `tests/VideoIndexer.App.Tests/TestHelpers/FakeSettingsService.cs` — **do not create a new file** (the file already exists and fully implements `ISettingsService`). Add the two missing tracking members needed by WP-006 tests:
    - `public int SaveCallCount { get; private set; }` — incremented in `SaveAsync`.
    - `public AppOptions? LastSaved { get; private set; }` — set to `updated` in `SaveAsync`.
24. Add `ISettingsService? _settingsService` field to `MovieEditorViewModel`. Add it as a constructor parameter in the full DI constructor (alongside the existing required `IMovieRepository movieRepository` and optional `INameParser?`, `ILabelCleanerService?`). The parameterless testing constructor leaves it `null`.
25. In `CleanLabelCommand`, replace `new LabelCleanerOptions()` with:
    ```csharp
    var initialOptions = _settingsService?.Current.LabelCleaner ?? new LabelCleanerOptions();
    ```
    Pass `initialOptions` to the `LabelCleanerViewModel` constructor.
26. After the dialog returns an accepted result, extract the accepted options from the `LabelCleanerViewModel` and persist them:
    ```csharp
    if (_settingsService is not null)
    {
        var updated = _settingsService.Current with { LabelCleaner = vm.ToOptions() };
        await _settingsService.SaveAsync(updated, ct).ConfigureAwait(false);
    }
    ```
    Verify that `LabelCleanerViewModel` exposes a `ToOptions()` method (or equivalent property) returning the current `LabelCleanerOptions`; add it if absent.
27. Register `ISettingsService` in `Program.cs`'s `MovieEditorViewModel` factory lambda.
28. Add tests to `MovieEditorViewModelTests.cs` (or a new `CleanLabelSettingsTests` region):
    - `CleanLabelCommand_ReadsInitialOptionsFromSettingsService` — pre-load `FakeSettingsService` with a `LabelCleanerOptions` with `PreserveDates = false`, open dialog, assert the `LabelCleanerViewModel` was constructed with `PreserveDates = false`.
    - `CleanLabelCommand_SavesAcceptedOptionsToSettingsService` — accept the dialog, assert `FakeSettingsService.SaveCallCount == 1` and `LastSaved.LabelCleaner` matches the accepted toggles.
    - `CleanLabelCommand_WhenSettingsServiceIsNull_UsesDefaultOptions` — ensure the parameterless constructor path (no settings service) still works.


## Dependencies

- WP-001 has no dependencies; can be done first.
- WP-002 and WP-003 have no dependencies on each other or on WP-001; all three can proceed in parallel.
- WP-004 must precede WP-005 because the new integration test in step 16 exercises the `ORDER BY` fix; and step 19/20 (FakeMovieRepository) does not depend on the DB migration.
- WP-005 (steps 19–22) depends on nothing except the test infrastructure established in WP-001.
- WP-006 depends on WP-005 having introduced `FakeSettingsService` infrastructure awareness; the `FakeSettingsService` helper is a WP-006 deliverable (step 23), so no hard dependency on WP-005 for that file. Independent.


## Required Components

### New files

- `tests/VideoIndexer.Infrastructure.Tests/TestHelpers/SharedTransactionalConnectionFactory.cs` (WP-001)
- `tests/VideoIndexer.App.Tests/Converters/IntDecimalConverterTests.cs` (WP-005)
- ~~`tests/VideoIndexer.App.Tests/TestHelpers/FakeSettingsService.cs`~~ (WP-006 — file already exists; extend with `SaveCallCount` and `LastSaved` instead)
- `src/VideoIndexer.Infrastructure/Database/Migrations/m039_add_filenames_created_at.sql` (WP-004)

### Modified files

- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperMovieRepositoryTests.cs` (WP-001, WP-004)
- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperMovieCatalogRepositoryTests.cs` (WP-001)
- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperFilterSlotRepositoryTests.cs` (WP-001)
- `tests/VideoIndexer.Infrastructure.Tests/Library/LibraryScannerIntegrationTests.cs` (WP-001)
- `tests/VideoIndexer.Infrastructure.Tests/Database/SpdbConfigRepositoryTests.cs` (WP-001)
- `src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs` (WP-002)
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeFilterSlotRepository.cs` (WP-002)
- `tests/VideoIndexer.App.Tests/MoviesListViewModelTests.cs` (WP-002)
- `src/VideoIndexer.App/ViewModels/MainContentViewModel.cs` (WP-002)
- `tests/VideoIndexer.App.Tests/MainContentViewModelTests.cs` (WP-002, WP-005)
- `src/VideoIndexer.App/Services/WindowsFileLauncherService.cs` (WP-003)
- `src/VideoIndexer.Infrastructure/Database/DatabaseBootstrapper.cs` (WP-004)
- `src/VideoIndexer.Infrastructure/Library/DapperMovieRepository.cs` (WP-004)
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeMovieRepository.cs` (WP-005)
- `tests/VideoIndexer.App.Tests/MovieEditorViewModelTests.cs` (WP-005, WP-006)
- `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs` (WP-006)
- `src/VideoIndexer.App/ViewModels/LabelCleanerViewModel.cs` (WP-006 — `ToOptions()` if absent)
- `src/VideoIndexer.App/Program.cs` (WP-006 — factory lambda update)
- `docs/agents/project-manifest/constraints.md` (WP-003, WP-004)


## Assumptions

- `MoviesListViewModelTests.cs` exists and contains a `FakeFilterSlotRepository` or equivalent fake; if not, the WP-002 test step must also add that fake.
- `LabelCleanerViewModel` can expose the currently-selected options as a method or property without architectural conflict (it is an App-layer class).
- `explorer.exe` correctly handles `ProcessStartInfo.ArgumentList` for both `OpenFolder` and `ShowInExplorer` use cases on Windows 10/11.
- `movies_filenames.created_at DEFAULT CURRENT_TIMESTAMP` will be populated for rows inserted after the migration; rows inserted before the migration will receive the migration timestamp (acceptable for determinism purposes, as the LIMIT 1 was already non-deterministic).


## Constraints

- No new NuGet packages; all new code uses existing dependencies.
- All warnings must remain zero (`TreatWarningsAsErrors=true`).
- `VideoIndexer.Core` must not gain any new external NuGet dependency.
- `AppOptions` mutation must always produce a new record via `with { }` and persist through `ISettingsService.SaveAsync`.
- Schema revision must be bumped atomically alongside the migration file and the `DatabaseBootstrapper.ExpectedRevision` constant. All three must change in the same WP-004 commit.
- `ILibraryScanner.RefreshAsync` cancellation contract must not be touched (not in scope).
- `test-config.json` must not be committed.


## Out of Scope

- M7 Tagging right sidebar.
- M8 Cover Image tab.
- Actor `ListBox` replacement (MoveActorUp/Down wiring) — deferred to a future milestone per synthesis Next Steps #3.
- `GetOriginalFilenameAsync` full-word boundary limitation fix in `INameParser` — flagged in constraints.md as a known limitation; not addressed here.
- Any changes to the DSL filter language or `FilterExpressionEvaluator`.


## Acceptance Criteria

- AC-1: `LoadFilterSlotsAsync` does not propagate exceptions to its caller when the filter-slot repository throws; `FilterSlots` is empty and `ActiveFilterSlot` is `null` after the failure.
- AC-2: `OnEditorCloseRequested` does not call `CloseSubView()` when `CurrentPage` is not `MainContentPage.MovieEditor`.
- AC-3: `WindowsFileLauncherService.OpenFolder` passes the path through `ProcessStartInfo.ArgumentList` (no manual quoting or string concatenation in `Arguments`).
- AC-4: `WindowsFileLauncherService.ShowInExplorer` uses `ProcessStartInfo.ArgumentList` to compose the `/select,` argument.
- AC-5: Schema revision is 39. `DatabaseBootstrapper.ExpectedRevision == 39`. The `m039` migration file is present and idempotent.
- AC-6: `DapperMovieRepository.GetOriginalFilenameAsync` returns the filename with the earliest `created_at` when multiple filenames exist for the same movie.
- AC-7: All four `NonDisposingConnection` duplicates in `VideoIndexer.Infrastructure.Tests` are removed; all integration tests pass using the shared type.
- AC-8: `IntDecimalConverterTests.cs` exists with the seven enumerated tests all passing.
- AC-9: `MoveActorDown_IsNoOp_WhenNoActorSelected` test exists and passes.
- AC-10: `FakeMovieRepository.SetOriginalFilename()` exists; the `CleanLabel` primary-path test passes using a non-null filename.
- AC-11: `MovieEditorViewModel.CleanLabelCommand` reads initial `LabelCleanerOptions` from `ISettingsService.Current.LabelCleaner` when the service is injected; saves accepted options back on dialog accept.
- AC-12: Build produces zero warnings. All 613+ tests pass (or more with new tests added).


## Testing Strategy

All changes are covered by unit or integration tests at the layer where the logic lives:

- **App-layer VM changes** (WP-002, WP-005, WP-006): tested in `VideoIndexer.App.Tests` using existing fake infrastructure.
- **Infrastructure changes** (WP-001, WP-004): tested in `VideoIndexer.Infrastructure.Tests` (integration; self-skip without a live DB).
- **Converter tests** (WP-005): pure unit tests in `VideoIndexer.App.Tests`; no DI or DB needed.
- **`WindowsFileLauncherService`** (WP-003): the service invokes `Process.Start` which cannot be unit-tested without a shell. AC-3 and AC-4 are verified by code inspection and the absence of `Arguments =` string-concatenation in the implementation. A compile-time structural check (reading the source) is sufficient.


## Test Plan

- `MoviesListViewModelTests.cs` — `LoadFilterSlotsAsync_WhenRepositoryThrows_DoesNotPropagateException` — asserts no exception escapes; `FilterSlots` empty after fault — **AC-1**
- `MainContentViewModelTests.cs` — `OnEditorCloseRequested_WhenCurrentPageIsNotMovieEditor_DoesNotCloseSubView` — asserts guard fires correctly — **AC-2**
- `DapperMovieRepositoryTests.cs` — `GetOriginalFilenameAsync_WithMultipleFilenames_ReturnsEarliest` — asserts ORDER BY created_at semantics — **AC-6**
- `VideoIndexer.App.Tests/Converters/IntDecimalConverterTests.cs` — 7 tests covering `Convert` and `ConvertBack` for all null/type combinations — **AC-8**
- `MovieEditorViewModelTests.cs` — `MoveActorDown_IsNoOp_WhenNoActorSelected` — asserts null-guard symmetry with Up variant — **AC-9**
- `MovieEditorViewModelTests.cs` — `CleanLabelCommand_UsesOriginalFilenameFromRepository_AsRawInput` — asserts primary filename path — **AC-10**
- `MovieEditorViewModelTests.cs` — `CleanLabelCommand_ReadsInitialOptionsFromSettingsService` — asserts options read from settings — **AC-11**
- `MovieEditorViewModelTests.cs` — `CleanLabelCommand_SavesAcceptedOptionsToSettingsService` — asserts save-back on accept — **AC-11**
- `MovieEditorViewModelTests.cs` — `CleanLabelCommand_WhenSettingsServiceIsNull_UsesDefaultOptions` — asserts null-safe fallback — **AC-11**


## Documentation Updates

- `docs/agents/project-manifest/constraints.md` — WP-003: add `WindowsFileLauncherService` uses `ProcessStartInfo.ArgumentList` note under Shell Integration.
- `docs/agents/project-manifest/constraints.md` — WP-004: bump "Expected schema revision" to `39`; add m039 rollback procedure; add sentinel test maintenance bullet.
- `docs/agents/project-manifest/api-surface.md` — WP-006: replace the `MovieEditorViewModel` full constructor entry with the complete five-parameter signature: `MovieEditorViewModel(Movie movie, IMovieRepository movieRepository, INameParser? nameParser = null, ILabelCleanerService? labelCleanerService = null, ISettingsService? settingsService = null)` (the manifest currently lists only the two required parameters, omitting `INameParser?` and `ILabelCleanerService?` which were added in M6); also add `ToOptions()` to `LabelCleanerViewModel` if added.
- `docs/agents/project-manifest/data-flows.md` — WP-006: update the `CleanLabelCommand` section to show initial options being read from `ISettingsService.Current.LabelCleaner` instead of `new LabelCleanerOptions()`, and accepted options being persisted via `ISettingsService.SaveAsync` after dialog accept.
- `docs/agents/project-manifest/file-tree.md` — WP-001: add `TestHelpers/SharedTransactionalConnectionFactory.cs` entry; WP-004: add `m039_add_filenames_created_at.sql` entry; WP-005: add `Converters/IntDecimalConverterTests.cs` entry; WP-006: `TestHelpers/FakeSettingsService.cs` is already listed — no new entry needed; optionally amend its description to reflect the new `SaveCallCount` and `LastSaved` tracking members.
- `docs/agents/project-manifest/tech-stack.md` — WP-004: bump schema revision row from `38` to `39`.


## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **`MoviesListViewModelTests.cs` lacks a `FakeFilterSlotRepository`** | The Engineer must check for its existence before implementing WP-002 step 6; add the fake if absent before writing the test. |
| **`m039` migration back-fills `created_at` with migration timestamp** | This is acceptable: all existing filenames get the same timestamp, so LIMIT 1 remains deterministic (arbitrary but stable). Document in the migration file comment. |
| **`ProcessStartInfo.ArgumentList` + `explorer.exe` edge cases** | `explorer.exe` is the documented target; `ArgumentList` is a .NET runtime API that handles quoting correctly for Windows process creation. Manual verification on a path with spaces is sufficient smoke-test. |
| **`LabelCleanerViewModel.ToOptions()` absent** | The Engineer confirms existence before WP-006 step 26; adds the method if absent. The implementation must include **all four `LabelCleanerOptions` properties** — `PreserveDates`, `UCWords`, `StripKeywords`, and `Keywords = _keywords` — to avoid silently resetting the user's keyword list to `[]` on every dialog accept. `_keywords` is a `private readonly` field set at construction, so `ToOptions()` must be an instance method: `public LabelCleanerOptions ToOptions() => new LabelCleanerOptions { PreserveDates = PreserveDates, UCWords = UCWords, StripKeywords = StripKeywords, Keywords = _keywords };`. The test `CleanLabelCommand_SavesAcceptedOptionsToSettingsService` must also assert that `LastSaved.LabelCleaner.Keywords` matches the original keyword list. |
| **`FakeFilterSlotRepository` throws check** | If `LoadFilterSlotsAsync` currently catches `OperationCanceledException` separately, the new catch block must not swallow cancellation. Use `catch (Exception ex) when (ex is not OperationCanceledException)` to be safe. |
