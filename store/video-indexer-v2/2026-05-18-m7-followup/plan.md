# Plan — M7 Follow-Up: Ship Gate, Merge Execution, Lifecycle & Debt

## Summary

This plan resolves every actionable item identified in the M7 Tagging Core synthesis report (2026-05-18). The work is grouped into ten work packages in priority order:

1. **WP-A — Pre-ship gate: `TagEditorView` null-sentinel** — critical UI fix blocking M7 production readiness.
2. **WP-B — Pre-ship gate: `tags_bookmarks` cascade in `DeleteTagAsync`** — DB hygiene fix; confirmed schema gap.
3. **WP-C — M7 Merge Execution** — add `ITagsRepository.MergeTagAsync`, implement it transactionally in `DapperTagsRepository`, fix `GetMergeImpactAsync` to surface overlap counts, and delegate from `TagsManager`.
4. **WP-D — `TaggerViewModel` lifecycle** — implement `IDisposable`, wire `Unloaded`, add `LoadAsync` re-entry guard to `MovieEditorViewModel`.
5. **WP-E — Tag toggle error surface** — replace fire-and-forget with an error callback that exposes `LastOperationError` on `TaggerViewModel`.
6. **WP-F — `CategoryEditorViewModel` unit tests** — three remaining acceptance-criteria tests; the file already exists with three of the six tests from WP-020.
7. **WP-G — `TagEditorViewModel` unit tests** — two remaining tests; the file already has six of the eight planned tests from WP-020; also covers the new `ClearParentTagCommand` from WP-A.
8. **WP-H — `DapperTagsRepository` integration test expansion** — extend live-DB test coverage to CRUD round-trips, cascade verification, grant management, bookmark association, and the new merge write path.
9. **WP-I — Dapper `CancellationToken` sweep** — propagate cancellation tokens through all five repository implementations using `CommandDefinition`.
10. **WP-J — Minor debt cleanup** — `ImpactRow` deduplication, `MoviesListViewModel` factory lambda, `TaggingOptions` validation, count-mismatch logging, `CS8765` nullability warnings.

## Plan Audit Cycles
- Audits: 3 — Plan Auditor v1.3.0
- Architectural Reviews: 1 — Plan Architect Reviewer v1.4.0

## Architectural Context

**Layer map:**
- `VideoIndexer.Core` — `ITagsRepository`, `ITagsManager`, `TagMergeImpact`, `TagDeleteImpact`, `TagConstants`, `TaggingOptions` (`src/VideoIndexer.Core/`)
- `VideoIndexer.Infrastructure` — `DapperTagsRepository`, `TagsManager` (`src/VideoIndexer.Infrastructure/Library/`)
- `VideoIndexer.App` — `TagEditorViewModel`, `TagEditorView`, `TaggerViewModel`, `TaggerTagViewModel`, `MovieEditorViewModel` (`src/VideoIndexer.App/ViewModels/`, `src/VideoIndexer.App/Views/`)
- Tests — `VideoIndexer.App.Tests` (unit), `VideoIndexer.Infrastructure.Tests` (integration)

**Key pre-existing patterns:**
- Dapper row mapping via private sealed inner classes (`TagRow`, `MergeImpactRow`); no positional records.
- Repository transactions: `BeginTransaction()` / `Commit()` / `Rollback()` with ambient-transaction fallback via `try/catch(InvalidOperationException)` — established in `DeleteTagAsync` (WP-021).
- `CommandDefinition` usage does not yet exist in any repository; all five use bare `QueryAsync`/`ExecuteAsync` overloads.
- `[ObservableProperty]` + `partial void OnXyzChanged` for ViewModel side effects (CommunityToolkit.Mvvm 8.3.2).
- Compiled Avalonia bindings throughout (`x:DataType`); `null` sentinel items in typed `ItemsSource` lists are incompatible with compiled item templates.
- `IDisposable` on ViewModels: not yet present anywhere in `VideoIndexer.App`; `TaggerViewModel` is the first target.
- Test helpers in `tests/VideoIndexer.App.Tests/TestHelpers/` — `FakeTagsManager`, `FakeMoviePropertiesService` exist; pattern: configurable fakes tracked by call count.


## Approach / Architecture

**WP-A** adds a `ClearParentTagCommand` (`IRelayCommand`) to `TagEditorViewModel` that sets `SelectedParentTag = null`, and adds a "Clear" `Button` in `TagEditorView.axaml` beside the Parent Tag `ComboBox`, visible only when `SelectedParentTag` is not `null`. This avoids changing the `AvailableTags` type (avoiding compiled-binding breakage) and avoids any Core model changes.

**WP-B** adds `DELETE FROM tags_bookmarks WHERE tag_id = @TagId` as step 1.5 in `DapperTagsRepository.DeleteTagAsync`, inside the existing transaction. Requires confirming the FK definition in the database schema SQL; if `ON DELETE CASCADE` is already present, the application-level step is a no-op but does no harm.

**WP-C** introduces `MergeTagAsync(long sourceTagId, long targetTagId, CancellationToken)` to `ITagsRepository` and implements it in `DapperTagsRepository` with all five SQL steps wrapped in a transaction (same ambient-transaction guard pattern used in `DeleteTagAsync`). `TagsManager.MergeTagAsync` is refactored to call `_repository.MergeTagAsync` instead of issuing raw SQL directly. The `TagMergeImpact` record gains an `OverlapMovies` property (count of movies tagged with both source and target); `GetMergeImpactAsync` gains the subquery; `TagMergeView` displays the overlap count.

**WP-D** makes `TaggerViewModel` implement `IDisposable`: `Dispose()` unsubscribes `OnTagsDataChanged` from `ITagsManager.DataChanged`. `MovieEditorViewModel.LoadAsync` disposes any prior `TaggerVm` before re-initialising. `TaggerView.axaml.cs` hooks `OnUnloaded` to call `vm.Dispose()` when its DataContext is a `TaggerViewModel`.

**WP-E** adds a `string LastOperationError` observable property to `TaggerViewModel` and a delegate parameter `Action<string>? onError` to `TaggerTagViewModel`'s constructor (injected from `TaggerViewModel`). The toggle callback is wrapped in a `try/catch` that invokes `onError`, which calls `ReportError`. `ReportError` is a one-liner: `LastOperationError = message`. The error is cleared at the start of the next toggle attempt (top of `OnIsCheckedStoredChanged`, before the callback fires) — the same persistent-until-cleared pattern used by `GrantError` and `ParentError` in `TagEditorViewModel`. No `Task.Delay`, no `Dispatcher.UIThread.Post`, no timer cancellation logic. `TaggerView.axaml` binds a `TextBlock` to `LastOperationError` (shown when non-empty). This avoids a new Core interface while surfacing toggle errors in the UI.

**WP-F, WP-G** follow the existing test pattern: `SutBundle` helper, `FakeTagsManager`, assertion via FluentAssertions.

**WP-H** follows the `DapperTagsRepositoryTests.cs` skip/cleanup discipline: `[Collection("LiveDB")]`, `IClassFixture<LiveDbFixture>`, `VI_TEST_TR_` prefix for test rows.

**WP-I** replaces bare Dapper method overloads with `new CommandDefinition(sql, param, cancellationToken: ct)` in all five repositories. No interface or contract changes required.

**WP-J** is a series of minimal targeted edits; each item is independent.


## Rationale

- **Null-sentinel via button** beats a wrapper type or a mixed-type list because compiled bindings require a single `x:DataType` on the item template; a "None" placeholder `Tag` record would pollute the domain model. A clear button is the canonical Avalonia pattern for optional foreign-key fields.
- **`ITagsRepository.MergeTagAsync`** restores the interface-first constraint violated by the current direct `_connectionFactory` usage in `TagsManager.MergeTagAsync`. Moving the SQL to `DapperTagsRepository` also brings transaction safety consistently with the established `DeleteTagAsync` pattern.
- **`TaggerViewModel.LastOperationError` instead of `IErrorBus`** is proportional: there is currently one consumer (tag toggle). A full `IErrorBus` abstraction is justified only once a second consumer exists; this plan records that rationale in `constraints.md` to prevent the IErrorBus from being reintroduced prematurely.
- **`CancellationToken` via `CommandDefinition`** is the Dapper-idiomatic approach; it propagates cancellation all the way to the MySql connector's read loop, not just to the connection-open phase.


## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|----------|--------------|-------------------------|-------------------|
| Null-sentinel UI | "Clear" button + `ClearParentTagCommand` | Mixed-type wrapper; null item in `IReadOnlyList<Tag>`; PlaceholderText-only (no clear path) | Button requires zero type changes; compiled bindings stay intact; least surprise for users |
| Merge execution location | `ITagsRepository.MergeTagAsync` + transaction | Keep raw SQL in `TagsManager`; move to a dedicated `IMergeService` | Repository is the established DB boundary; a dedicated merge service adds indirection with no benefit at this scale |
| TaggerViewModel event leak | `IDisposable` + `Unloaded` wiring | `WeakEventManager`; scoped event subscription in `MovieEditorViewModel` | `IDisposable` is the standard CLR pattern; `WeakEventManager` hides ownership semantics and complicates unit testing |
| Tag toggle errors | `TaggerViewModel.LastOperationError` + delegate callback | Full `IErrorBus` in Core; toast notification service | Single consumer today; a new Core interface would be speculative; log + in-panel message is enough |
| `CommandDefinition` propagation | Replace every Dapper call | Pass `ct` only to connection-open (status quo); per-call wrappers | Bare-call status quo means cancellation only prevents connection-open; `CommandDefinition` is the only path that cancels in-flight queries |
| WP-E error persistence | `LastOperationError` persistent-until-cleared (cleared at the top of `OnIsCheckedStoredChanged` before next attempt) | Auto-clearing 5-second timer via `Task.Delay` + `Dispatcher.UIThread.Post` | Timer wraps a scalar `[ObservableProperty]` setter in `Dispatcher.UIThread.Post` — unnecessary because Avalonia 11 auto-marshals scalar INPC from any thread, contradicting the documented constraint. The fire-and-forget also captures `this` and would set `LastOperationError` on a logically-disposed ViewModel if `TaggerViewModel.Dispose()` (WP-D) runs within 5 seconds of a toggle error — requiring a `CancellationTokenSource` that the plan did not specify. Persistent-until-cleared is consistent with `GrantError`/`ParentError`, eliminates the dispose-interaction gap, and requires no `Dispatcher.UIThread.RunJobs()` flush in tests. |


## Pattern Alignment

| Pattern | Alignment |
|---------|-----------|
| Repository transactions with ambient-transaction guard | WP-B and WP-C follow the exact `DeleteTagAsync` pattern (`src/VideoIndexer.Infrastructure/Library/DapperTagsRepository.cs`) |
| Dapper private sealed row-mapping classes | WP-C `MergeTagAsync` has no row mapping; WP-J ImpactRow consolidation follows existing `DeleteImpactRow`/`MergeImpactRow` precedent |
| `IRelayCommand` for ViewModel commands | WP-A `ClearParentTagCommand` follows `CloseCommand` / `CancelCommand` patterns throughout `VideoIndexer.App.ViewModels` |
| `[ObservableProperty]` side-effect callbacks | WP-E `LastOperationError` follows `GrantError`, `ParentError` in `TagEditorViewModel` |
| Integration tests: skip/cleanup + `VI_TEST_` prefix | WP-H follows `DapperTagsRepositoryTests.cs` |
| Unit test `SutBundle` helper | WP-F/G follow `TaggerViewModelTests.cs` and `TagMergeViewModelTests.cs` |
| **Departure:** `ITagsManager.ConnectMovieTagAsync` and `DisconnectMovieTagAsync` continue to bypass `ITagsRepository` for the movie-tag association path. This pre-existing deviation is out-of-scope for this plan and is noted in `constraints.md`. | |
| **Departure:** `TagsManager.LoadUsageCountsAsync` continues to use `_connectionFactory` directly (not via `ITagsRepository`). Out-of-scope; flagged as future debt. | |


## Detailed Steps

### WP-A — TagEditorView null-sentinel fix

1. In `src/VideoIndexer.App/ViewModels/TagEditorViewModel.cs`:
   - Add `[RelayCommand]` method `ClearParentTag()` that sets `SelectedParentTag = null`.
   - The generated `ClearParentTagCommand` replaces no existing command.
2. In `src/VideoIndexer.App/Views/TagEditorView.axaml`:
   - Wrap the existing Parent Tag `ComboBox` and a new `Button` (text: "Clear", label: "✕ None") in a `Grid` or `DockPanel`.
   - Bind the button's `Command` to `ClearParentTagCommand`.
   - Bind `IsVisible` of the button to `SelectedParentTag` using `ObjectConverters.IsNotNull`.
   - Remove the now-redundant `PlaceholderText="None (root-level)"` from the ComboBox (optional UX polish).

### WP-B — `tags_bookmarks` cascade in `DeleteTagAsync`

1. Inspect `src/VideoIndexer.Infrastructure/Database/migrations/` for the `CREATE TABLE tags_bookmarks` DDL to confirm whether `tag_id` has `ON DELETE CASCADE`. If it does, document the fact in `constraints.md` and skip the application-level step.
2. If `ON DELETE CASCADE` is absent, in `DapperTagsRepository.DeleteTagAsync` add as **step 1.5** (between the existing `parentTagId` fetch and step 2) inside the transaction block:
   ```sql
   DELETE FROM tags_bookmarks WHERE tag_id = @TagId
   ```
   Use `new CommandDefinition(sql, param, ownedTx, cancellationToken: cancellationToken)` for consistent CancellationToken handling (anticipates WP-I).
3. Update `constraints.md` — remove the "Low / Verify FK" entry and document the resolution.

### WP-C — MergeTagAsync proper implementation

1. **`TagMergeImpact` model** (`src/VideoIndexer.Core/Models/TagMergeImpact.cs`):
   - Add `int OverlapMovies { get; init; }` — movies tagged with both source and target already.
2. **`ITagsRepository`** (`src/VideoIndexer.Core/Abstractions/ITagsRepository.cs`):
   - Add method: `Task MergeTagAsync(long sourceTagId, long targetTagId, CancellationToken cancellationToken = default);`
3. **`DapperTagsRepository`** (`src/VideoIndexer.Infrastructure/Library/DapperTagsRepository.cs`):
   - Implement `MergeTagAsync` with transaction (ambient-transaction guard identical to `DeleteTagAsync`):
     - Step 1: `INSERT IGNORE INTO tags_movies … SELECT … FROM tags_movies WHERE tag_id = @SourceTagId` → rewire movies to target.
     - Step 1b: `DELETE FROM tags_movies WHERE tag_id = @SourceTagId`.
     - Step 2: `INSERT IGNORE INTO tags_bookmarks … SELECT … FROM tags_bookmarks WHERE tag_id = @SourceTagId` → rewire bookmarks.
     - Step 2b: `DELETE FROM tags_bookmarks WHERE tag_id = @SourceTagId`.
     - Step 3: `INSERT IGNORE INTO tags_grants … SELECT @TargetTagId, grants_tag_id … WHERE tag_id = @SourceTagId` → migrate outbound grants.
     - Step 3b: `INSERT IGNORE INTO tags_grants … SELECT tag_id, @TargetTagId … WHERE grants_tag_id = @SourceTagId` → migrate inbound grants.
     - Step 3c: `DELETE FROM tags_grants WHERE tag_id = @SourceTagId OR grants_tag_id = @SourceTagId`.
     - Step 4: `UPDATE tags SET parent_tag_id = @TargetTagId WHERE parent_tag_id = @SourceTagId`.
     - Step 5: `DELETE FROM tags WHERE tag_id = @SourceTagId`.
   - Extend `GetMergeImpactAsync` to compute `OverlapMovies`:
     ```sql
     (SELECT COUNT(*) FROM tags_movies tm1
      INNER JOIN tags_movies tm2 ON tm1.movie_id = tm2.movie_id
      WHERE tm1.tag_id = @SourceTagId AND tm2.tag_id = @TargetTagId) AS OverlapMovies
     ```
   - Update `ImpactRow` inner class (unified by WP-J1) to include `OverlapMovies` in the `GetMergeImpactAsync` query mapping.
   - Update the `TagMergeImpact` construction to populate `OverlapMovies`.
4. **`TagsManager`** (`src/VideoIndexer.Infrastructure/Library/TagsManager.cs`):
   - Rewrite `MergeTagAsync` to call `await _repository.MergeTagAsync(sourceTagId, targetTagId, cancellationToken).ConfigureAwait(false)` then `ReloadAsync` + `RaiseDataChanged`.
   - Remove the raw SQL block and the `using var connection = await _connectionFactory…` that was exclusively in `MergeTagAsync`.
5. **`TagMergeView.axaml`** (`src/VideoIndexer.App/Views/TagMergeView.axaml`):
   - Add a row to the impact preview panel displaying `OverlapMovies` ("Already shared: {N} movie(s)").
6. **`api-surface.md`**:
   - Add `MergeTagAsync` to the `ITagsRepository` section.
   - Add `OverlapMovies` to the `TagMergeImpact` model entry.
   - Remove the "MergeTagAsync bypasses ITagsRepository directly (tracked debt — no transaction)" note from the `TagsManager` entry.

### WP-D — `TaggerViewModel` lifecycle

1. **`TaggerViewModel`** (`src/VideoIndexer.App/ViewModels/TaggerViewModel.cs`):
   - Implement `IDisposable`.
   - `Dispose()`: unsubscribe `OnTagsDataChanged` from `_tagsManager.DataChanged`; set a `bool _disposed` flag.
2. **`TaggerView.axaml.cs`** (`src/VideoIndexer.App/Views/TaggerView.axaml.cs`):
   - In `OnUnloaded` handler: if `DataContext is TaggerViewModel vm`, call `vm.Dispose()`.
3. **`MovieEditorViewModel`** (`src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs`):
   - In `LoadAsync`: if `TaggerVm` is already non-null (re-entry guard), call `TaggerVm.Dispose()` before reassigning.
4. **`constraints.md`**:
   - Replace the existing "`TaggerViewModel` event-subscription lifecycle" gotcha with the updated resolved state, noting that `TaggerView.axaml.cs` now owns the `Dispose()` call and `MovieEditorViewModel.LoadAsync` also guards against double-subscribe.
   - Remove the "MovieEditorViewModel.LoadAsync single-call invariant" gotcha entry and replace with a note that the re-entry guard is now implemented.
5. **`tests/VideoIndexer.App.Tests/TaggerViewModelTests.cs`** (extend):
   - Add `Dispose_UnsubscribesFromDataChanged` — raises `ITagsManager.DataChanged` after calling `vm.Dispose()`; asserts that `RebuildCategories` is **not** invoked (subscription has been removed).

### WP-E — Tag toggle error surface

1. **`TaggerViewModel`** (`src/VideoIndexer.App/ViewModels/TaggerViewModel.cs`):
   - Add `[ObservableProperty] private string _lastOperationError = string.Empty;`.
   - Add internal method `ReportError(string message)`: single line — `LastOperationError = message`. No `Task.Delay`, no `Dispatcher.UIThread.Post`, no timer. `LastOperationError` is a scalar `[ObservableProperty]`; Avalonia 11 auto-marshals `PropertyChanged` for scalar properties from any thread, making `Dispatcher.UIThread.Post` unnecessary and test-hostile here.
2. **`TaggerTagViewModel`** (`src/VideoIndexer.App/ViewModels/TaggerTagViewModel.cs`):
   - Add constructor parameter `Action<string>? onError` (nullable for backward compatibility in tests).
   - After the `if (IsCheckedImplied) return;` guard and before invoking the stored-state callback, add `_onError?.Invoke(string.Empty);` — this clears any prior error at the start of each real (non-implied) toggle attempt (persistent-until-cleared pattern, consistent with `GrantError`/`ParentError`). Do **not** place this before the `IsCheckedImplied` guard; doing so would silently clear an unrelated prior error on every implied-tag toggle.
   - Wrap the callback invocation in `try { _ = callback(…) } catch (Exception ex) { onError?.Invoke(ex.Message); }`. Replace bare `_ = callback(…)` with an async lambda that awaits the callback and catches exceptions.
3. **`TaggerViewModel.RebuildCategories`** (or wherever `TaggerTagViewModel` instances are constructed):
   - Pass `error => ReportError(error)` as the `onError` argument.
4. **`TaggerView.axaml`** (`src/VideoIndexer.App/Views/TaggerView.axaml`):
   - Add a `TextBlock` bound to `LastOperationError`, `IsVisible` when non-empty, styled as a transient warning bar below the filter row.

### WP-F — `CategoryEditorViewModel` unit tests

`tests/VideoIndexer.App.Tests/CategoryEditorViewModelTests.cs` already exists with three tests from WP-020 (`Delete_NonEmptyCategory_Blocked`, `Rename_DefaultCategory_Blocked`, `Rename_MostUsedCategory_Blocked`). Add the three remaining acceptance-criteria tests:

1. `EmptyName_DisablesSaveCommand` — `Name=""` → `SaveCommand.CanExecute` is `false`.
2. `CreateMode_DisablesDeleteCommand` — create mode (no `existingCategory`) → `DeleteCommand.CanExecute` is `false`.
3. `CancelCommand_FiresCloseRequestedWithNull` — `CancelCommand.Execute(null)` → `CloseRequested` raised with `null` argument.

(The following three are already covered and require no new test: `IsDefault` disables `SaveCommand` — `Rename_DefaultCategory_Blocked`; `IsMostUsed` disables `SaveCommand` — `Rename_MostUsedCategory_Blocked`; `tagCount > 0` disables `DeleteCommand` — `Delete_NonEmptyCategory_Blocked`.)

### WP-G — `TagEditorViewModel` unit tests

`tests/VideoIndexer.App.Tests/TagEditorViewModelTests.cs` already exists with six tests from WP-020 (`SaveTag_ClampsWeightAboveMax`, `SaveTag_ClampsWeightBelowMin`, `SaveTag_ParentCycle_ShowsInlineError`, `AddGrant_Valid_AppearsInGrantsList`, `AddGrant_WouldCreateCycle_RejectsWithInlineError`, `DeleteTag_ShowsImpactCountsBeforeExecute`). Add the two remaining tests:

1. `ConfirmDeleteAsync_FiresCloseRequestedWithNull` — `ConfirmDeleteCommand.Execute` → `CloseRequested` raised with `null`.
2. `ClearParentTagCommand_SetsSelectedParentTagToNull` — `SelectedParentTag = someTag`; `ClearParentTagCommand.Execute(null)` → `SelectedParentTag == null` (AC for WP-A).

(The following six are already covered and require no new test: Weight clamp above max — `SaveTag_ClampsWeightAboveMax`; Weight clamp below min — `SaveTag_ClampsWeightBelowMin`; cycle parent blocks save — `SaveTag_ParentCycle_ShowsInlineError`; valid grant adds to list — `AddGrant_Valid_AppearsInGrantsList`; grant cycle sets `GrantError` — `AddGrant_WouldCreateCycle_RejectsWithInlineError`; `DeleteCommand` sets `DeleteImpact` — `DeleteTag_ShowsImpactCountsBeforeExecute`.)

### WP-H — `DapperTagsRepository` integration test expansion

Extend `tests/VideoIndexer.Infrastructure.Tests/Database/DapperTagsRepositoryTests.cs` (following existing skip/cleanup discipline, `vi_test_tr_` prefix):

**CRUD round-trips (new):**
- `UpdateTagAsync_Roundtrip_FieldsUpdated` — create → update name/weight → reload → verify.
- `DeleteTagAsync_Cascade_RemovesTagsBookmarksRows` — create tag, connect bookmark, delete tag → verify `tags_bookmarks` rows gone (AC for WP-B).

*(The following are already covered by existing WP-021 tests and require no new test: `CreateTag_And_GetById_RoundTrip` covers create roundtrip; `DeleteTag_CascadesAssociationsAndReparentsChildren` covers tags_movies removal, grant removal, child reparenting, and tag-row deletion.)*

**Grant management (new):**
- `AddGrantAsync_ThenGetAllGrants_ContainsGrant`.
- `RemoveGrantAsync_ThenGetAllGrants_DoesNotContainGrant`.

*(Grant cycle detection is already covered by `AddGrant_WouldCreateCycle_ThrowsBeforeInsert`.)*

**Bookmark association (new):**
- `ConnectBookmarkTagAsync_ThenDisconnectBookmarkTagAsync_Roundtrip` — tests the full connect/disconnect roundtrip (existing `ConnectBookmarkTag_InsertsTags_bookmarksRow` only tests the insert side).

**Merge path (AC for WP-C — all new):**
- `MergeTagAsync_RewiresTags_MoviesToTarget` — connect movie to source, merge → movie now connected to target.
- `MergeTagAsync_DeletesSourceTag` — merge → source tag absent from `GetAllTagsAsync`.
- `MergeTagAsync_DoesNotDuplicateMovies_WhenOverlapExists` — movie connected to both source and target, merge → movie connected to target once only.

**Impact queries (new):**
- `GetMergeImpactAsync_OverlapMovies_CountsCorrectly` — verify `OverlapMovies` returned by WP-C fix.

### WP-I — Dapper `CancellationToken` sweep

For each of the five repositories, replace bare Dapper calls with `CommandDefinition`:

1. **`DapperMovieRepository`** (`src/VideoIndexer.Infrastructure/Library/DapperMovieRepository.cs`) — all `QueryAsync`, `QuerySingleAsync`, `QuerySingleOrDefaultAsync`, `ExecuteAsync` calls.
2. **`DapperMovieCatalogRepository`** (`src/VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs`) — same.
3. **`DapperFilterSlotRepository`** (`src/VideoIndexer.Infrastructure/Library/DapperFilterSlotRepository.cs`) — same.
4. **`DapperTagsRepository`** (`src/VideoIndexer.Infrastructure/Library/DapperTagsRepository.cs`) — same (applies to all methods including those added in WP-B and WP-C).
5. **`SpdbConfigRepository`** (`src/VideoIndexer.Infrastructure/Database/SpdbConfigRepository.cs`) — same.

Pattern for each call site:
```csharp
// Before
await connection.QueryAsync<T>(sql, param).ConfigureAwait(false);
// After
await connection.QueryAsync<T>(new CommandDefinition(sql, param, cancellationToken: cancellationToken)).ConfigureAwait(false);
```

Calls that already pass a transaction (`ownedTx`) use the overload that accepts both:
```csharp
new CommandDefinition(sql, param, ownedTx, cancellationToken: cancellationToken)
```

No test changes required; cancellation propagation is a contract improvement with no observable behavioural delta under normal conditions.

### WP-J — Minor debt cleanup

**J1 — `ImpactRow` consolidation** (`DapperTagsRepository.cs`):
- `DeleteImpactRow` and `MergeImpactRow` are private sealed classes with identical shape (`AffectedMovies`, `AffectedBookmarks`, `AffectedGrants`, `ChildrenToReparent`). After WP-C adds `OverlapMovies` to `MergeImpactRow`, the shapes diverge, so consolidation should happen *before* WP-C. Rename `DeleteImpactRow` to `ImpactRow`, add an optional `int OverlapMovies` to it (with a default of 0), and remove `MergeImpactRow`.

**J2 — `MoviesListViewModel` factory lambda** (`src/VideoIndexer.App/Program.cs`):
- Register `MoviesListViewModel` with an explicit factory lambda to resolve the ambiguous constructor selection: `services.AddSingleton(sp => new MoviesListViewModel(sp.GetRequiredService<IMovieCatalogRepository>(), …))`. Follow the `MovieEditorViewModel` `Func<Movie, MovieEditorViewModel>` precedent.

**J3 — `TaggingOptions.MostUsedThreshold` range validation** (`src/VideoIndexer.Infrastructure/Library/TagsManager.cs`):
- `MostUsedThreshold` is read in `TagsManager.PublishState` but immediately discarded (`_ = threshold; // suppress unused-variable analysis until filtering is implemented`). Any clamp applied to a discarded value is also discarded — producing dead validation code with no observable effect.
- Replace the two-line discard block (`var threshold = …; _ = threshold; // suppress …`) with a single TODO comment line: `// TODO (WP-J3): clamp MostUsedThreshold to [1, tagCount] and log when filtering is wired up`. Do **not** retain the variable declaration or `_ = threshold;` discard — keeping either without the other introduces a `CS0219` warning (unused variable) which is a build error under `TreatWarningsAsErrors=true`. Do **not** add active clamp or log code in this pass. The guard should be added in the WP that first activates threshold consumption.

**J4 — Count-mismatch defensive guard logging** (`DapperMovieCatalogRepository.cs`):
- The existing defensive count-mismatch guard returns unenriched. Since `ILogger` is not yet injected, add a `// TODO: log Warning when ILogger is injected (WP-J4)` comment at the guard site and update the `constraints.md` entry to track the logger injection as a future WP prerequisite.

**J5 — Pre-existing `CS8765` nullability warnings** (`tests/VideoIndexer.Tests/TagsManagerTests.cs` lines 344, 372):
- Add the appropriate nullability annotation (`?`) or suppress with `#pragma warning disable CS8765` with a rationale comment, whichever is less invasive given the test structure. All warnings are errors; this must be resolved to keep the build green.


## Dependencies

- **WP-J1** (ImpactRow consolidation) must execute **before WP-C** (which adds `OverlapMovies`); otherwise `MergeImpactRow` and `DeleteImpactRow` would need to be reconciled mid-flight.
- **WP-B** (bookmarks cascade) should execute **before WP-H** so that the integration test `DeleteTagAsync_Cascade_RemovesTagsBookmarksRows` can verify the fix.
- **WP-C** should execute **before WP-H** so that merge-path integration tests can verify the transactional implementation.
- **WP-A** should execute **before WP-G** so that `ClearParentTagCommand` exists when test G7 is written.
- **WP-D** should execute **before WP-G/F** if tests for `TaggerViewModel` lifecycle disposal are included.
- All other WPs are independent.

**Recommended sequencing:** J1 → B → A → C → D → E → F → G → H → I → J(2-5)


## Required Components

**Modified files:**
- `src/VideoIndexer.Core/Models/TagMergeImpact.cs` — add `OverlapMovies` property (WP-C)
- `src/VideoIndexer.Core/Abstractions/ITagsRepository.cs` — add `MergeTagAsync` (WP-C)
- `src/VideoIndexer.Infrastructure/Library/DapperTagsRepository.cs` — WP-B, WP-C, WP-I, WP-J1
- `src/VideoIndexer.Infrastructure/Library/TagsManager.cs` — WP-C, WP-J3
- `src/VideoIndexer.Infrastructure/Library/DapperMovieRepository.cs` — WP-I
- `src/VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs` — WP-I, WP-J4
- `src/VideoIndexer.Infrastructure/Library/DapperFilterSlotRepository.cs` — WP-I
- `src/VideoIndexer.Infrastructure/Database/SpdbConfigRepository.cs` — WP-I
- `src/VideoIndexer.App/ViewModels/TagEditorViewModel.cs` — WP-A
- `src/VideoIndexer.App/Views/TagEditorView.axaml` — WP-A
- `src/VideoIndexer.App/Views/TagMergeView.axaml` — WP-C (overlap row)
- `src/VideoIndexer.App/ViewModels/TaggerViewModel.cs` — WP-D, WP-E
- `src/VideoIndexer.App/ViewModels/TaggerTagViewModel.cs` — WP-E
- `src/VideoIndexer.App/Views/TaggerView.axaml` — WP-E (error bar)
- `src/VideoIndexer.App/Views/TaggerView.axaml.cs` — WP-D (Unloaded handler)
- `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs` — WP-D (re-entry guard)
- `src/VideoIndexer.App/Program.cs` — WP-J2
- `tests/VideoIndexer.App.Tests/TaggerViewModelTests.cs` — WP-D (extend — 1 test to add), WP-E (extend — 2 tests to add)
- `tests/VideoIndexer.App.Tests/CategoryEditorViewModelTests.cs` — WP-F (extend — 3 tests to add)
- `tests/VideoIndexer.App.Tests/TagEditorViewModelTests.cs` — WP-G (extend — 2 tests to add)
- `tests/VideoIndexer.Infrastructure.Tests/Database/DapperTagsRepositoryTests.cs` — WP-H (extend)
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeTagsRepository.cs` — WP-C (add `MergeTagAsync` no-op stub to satisfy updated `ITagsRepository`)
- `tests/VideoIndexer.Tests/TagsManagerTests.cs` — WP-C (add `MergeTagAsync` to `StubTagsRepository`; update to trackable no-op for delegation test), WP-J5

**Documentation:**
- `docs/agents/project-manifest/api-surface.md` — WP-C, WP-D, WP-E
- `docs/agents/project-manifest/constraints.md` — WP-B, WP-C, WP-D, WP-J


## Assumptions

- `tags_bookmarks.tag_id` CASCADE presence is **unconfirmed from migration files alone** — the migrations directory contains only m036–m040 and does not include the original `CREATE TABLE tags_bookmarks` DDL. WP-B step 1 must inspect the live database (e.g. `SHOW CREATE TABLE tags_bookmarks`) before proceeding. If `ON DELETE CASCADE` exists, the application-level DELETE is a no-op but harmless.
- Avalonia 11 / FluentAvalonia `Button` with `IsVisible` compiled binding works without additional converters; `ObjectConverters.IsNotNull` is already available in the project.
- `FakeTagsManager` in `tests/VideoIndexer.App.Tests/TestHelpers/FakeTagsManager.cs` already exposes a `Tags` list suitable for constructing `TagEditorViewModel` in unit tests; no new fake infrastructure required.
- `LiveDbFixture` and `[Collection("LiveDB")]` are reusable for WP-H tests without modification.
- `CommandDefinition` is available via `Dapper` 2.1.72 (already referenced); no new package required.


## Constraints

- All warnings are errors (`TreatWarningsAsErrors=true`); every change must compile cleanly.
- No `Version=` attributes on `<PackageReference>` — versions go in `Directory.Packages.props`.
- `.ConfigureAwait(false)` on every `await` in Infrastructure/Core code.
- `AppOptions` mutation always via `with { }` + `ISettingsService.SaveAsync`.
- `DatabaseBootstrapper.ExpectedRevision` sentinel test literal must match `40` — no schema changes in this plan, so no update needed.
- `test-config.json` must not be committed.
- No new NuGet packages are introduced.


## Out of Scope

- `LabelCleanerViewModel.DetectTags` applying detected tag IDs to the movie entity (deferred; requires MovieEditorViewModel save-path refactor).
- `TagsManager.LoadUsageCountsAsync` direct `_connectionFactory` usage (pre-existing; would require adding a usage-count method to `ITagsRepository`).
- `BatchTagSql` gaining a `WHERE movieId IN (…)` filter when pagination is added to `GetMovieListAsync` (deferred to pagination WP).
- `ITagsManager.ConnectMovieTagAsync` / `DisconnectMovieTagAsync` bypassing `ITagsRepository` (pre-existing deviation; left to a future infrastructure refactor WP).
- `HasTags()` semantic upgrade from `StoredTagCount > 0` to `EffectiveTagCount > 0` (noted as post-M7 TODO in evaluator).
- M10 deferred filter identifiers (`HasRatedBookmarks`, `BookmarkContains`, `AmountBookmarks`).
- `FfprobeArchiveEntry` sentinel documentation gap.
- macOS archive layout verification.


## Acceptance Criteria

### WP-A
- [ ] `TagEditorView` has a "Clear" button adjacent to the Parent Tag ComboBox.
- [ ] The button is visible only when `SelectedParentTag` is non-null.
- [ ] Clicking the button sets `SelectedParentTag = null` and the ComboBox reverts to its placeholder state.
- [ ] Compiled build passes (zero warnings/errors).

### WP-B
- [ ] `DapperTagsRepository.DeleteTagAsync` deletes `tags_bookmarks` rows for the deleted tag (either via application-level step 1.5 or confirmed schema-level `ON DELETE CASCADE`).
- [ ] `constraints.md` updated to document the resolution.

### WP-C
- [ ] `ITagsRepository` declares `MergeTagAsync`.
- [ ] `DapperTagsRepository.MergeTagAsync` wraps all five SQL steps in a transaction with ambient-transaction guard.
- [ ] `TagMergeImpact.OverlapMovies` is populated by `GetMergeImpactAsync`.
- [ ] `TagMergeView` displays the overlap count.
- [ ] `TagsManager.MergeTagAsync` no longer contains raw SQL or a direct `_connectionFactory` call.
- [ ] `api-surface.md` updated.

### WP-D
- [ ] `TaggerViewModel` implements `IDisposable`.
- [ ] `TaggerView.axaml.cs` `OnUnloaded` calls `vm.Dispose()`.
- [ ] `MovieEditorViewModel.LoadAsync` disposes any prior `TaggerVm` before re-initialising.
- [ ] `constraints.md` lifecycle entries updated.
- [ ] `TaggerViewModelTests.cs` `Dispose_UnsubscribesFromDataChanged` test passes.

### WP-E
- [ ] `TaggerTagViewModel` toggle callbacks are wrapped in try/catch.
- [ ] `TaggerViewModel.LastOperationError` is populated on exception.
- [ ] `TaggerView.axaml` shows a transient error bar when `LastOperationError` is non-empty.

### WP-F
- [ ] 6 tests total in `CategoryEditorViewModelTests.cs` (3 existing + 3 new); all pass.

### WP-G
- [ ] 8 tests in `TagEditorViewModelTests.cs` (6 existing + 2 new); all pass.

### WP-H
- [ ] 8+ new integration tests in `DapperTagsRepositoryTests.cs`; all self-skip cleanly when no DB is configured; all pass against a live DB.

### WP-I
- [ ] All five repositories use `CommandDefinition` for every Dapper call that accepts a `CancellationToken`.
- [ ] Build passes with zero warnings.

### WP-J
- [ ] `ImpactRow` is a single class; `DeleteImpactRow` and `MergeImpactRow` are removed.
- [ ] `MoviesListViewModel` has an explicit factory lambda in `Program.cs`.
- [ ] The `MostUsedThreshold` discard block in `TagsManager.PublishState` is replaced with a single `// TODO (WP-J3): clamp to [1, tagCount] and log when filtering is wired up` comment (active range validation is deferred to the WP that first consumes the threshold).
- [ ] CS8765 warnings in `TagsManagerTests.cs` are resolved (code-hygiene improvement; `VideoIndexer.Tests.csproj` sets `TreatWarningsAsErrors=false` so these do not currently break the build).
- [ ] `constraints.md` debt table updated to remove resolved entries.


## Testing Strategy

- **WP-A**: Verified by new unit tests (WP-G) and manual smoke-test of the Tag Editor UI (clear button functional).
- **WP-B**: Verified by WP-H integration test `DeleteTagAsync_Cascade_RemovesTagsBookmarksRows`.
- **WP-C**: Verified by WP-H integration tests (merge rewire, no duplicates, overlap count) and a new `TagsManagerTests.cs` unit test covering `TagsManager.MergeTagAsync` delegation to `ITagsRepository` (see Test Plan).
- **WP-E**: Verified by new unit tests in `TaggerViewModelTests.cs` (see Test Plan) and manual smoke-test (error bar visible on simulated DB failure).
- **WP-D**: Verified by new unit test asserting `DataChanged` subscription count drops to zero after `Dispose()`.
- **WP-F, WP-G**: Self-contained unit test WPs.
- **WP-H**: Live-DB integration tests; self-skip when no DB.
- **WP-I**: No functional test needed; verified by existing test suite passing and manual cancellation probing.
- **WP-J**: Verified by build success and existing test suite green.


## Test Plan

**WP-A / WP-G:**
- `tests/VideoIndexer.App.Tests/TagEditorViewModelTests.cs`
  - `ClearParentTagCommand_SetsSelectedParentTagToNull` — asserts `SelectedParentTag == null` after command execution — AC: WP-A.
  - `ConfirmDeleteAsync_FiresCloseRequestedWithNull` — asserts `CloseRequested` raised with `null` — AC: WP-G.
  - *(The following six tests from WP-020 already pass and require no changes: `SaveTag_ClampsWeightAboveMax`, `SaveTag_ClampsWeightBelowMin`, `SaveTag_ParentCycle_ShowsInlineError`, `AddGrant_Valid_AppearsInGrantsList`, `AddGrant_WouldCreateCycle_RejectsWithInlineError`, `DeleteTag_ShowsImpactCountsBeforeExecute`.)*

**WP-C:**
- `tests/VideoIndexer.Tests/TagsManagerTests.cs` (extend — 1 test to add):
  - `MergeTagAsync_DelegatesToRepository_ThenRaisesDataChanged` — sets up `StubTagsRepository.MergeTagAsync` as a trackable no-op (not `throw`); calls `TagsManager.MergeTagAsync`; asserts (1) `_repository.MergeTagAsync` was invoked with the correct IDs and (2) `DataChanged` was raised — AC: WP-C delegation refactor.

**WP-D:**
- `tests/VideoIndexer.App.Tests/TaggerViewModelTests.cs` (extend):
  - `Dispose_UnsubscribesFromDataChanged` — raises `DataChanged` after `Dispose()`; asserts `RebuildCategories` is not invoked — AC: WP-D.

**WP-E:**
- `tests/VideoIndexer.App.Tests/TaggerViewModelTests.cs` (extend — 2 tests to add):
  - `ToggleCallback_Throws_SetsLastOperationError` — constructs `TaggerViewModel` with a `connectTag` callback that throws; checks a stored tag; asserts `LastOperationError` is non-empty — AC: WP-E toggle error surface.
  - `ToggleCallback_SecondAttempt_ClearsLastOperationError` — after producing an error via a throwing callback, performs a second toggle using a **non-throwing** callback (`Task.CompletedTask`); asserts `LastOperationError == string.Empty` after the second toggle completes, proving the first error was cleared and not re-raised — AC: WP-E error cleared on next toggle.

**WP-F:**
- `tests/VideoIndexer.App.Tests/CategoryEditorViewModelTests.cs` (extend — add 3 tests):
  - `EmptyName_DisablesSaveCommand` — AC: WP-F #3.
  - `CreateMode_DisablesDeleteCommand` — AC: WP-F #5.
  - `CancelCommand_FiresCloseRequestedWithNull` — AC: WP-F #6.
  - *(Already passing from WP-020: `Rename_DefaultCategory_Blocked`, `Rename_MostUsedCategory_Blocked`, `Delete_NonEmptyCategory_Blocked`.)*

**WP-H:**
- `tests/VideoIndexer.Infrastructure.Tests/Database/DapperTagsRepositoryTests.cs` (extend — 9 new tests):
  - `UpdateTagAsync_Roundtrip_FieldsUpdated` — AC: WP-H CRUD.
  - `DeleteTagAsync_Cascade_RemovesTagsBookmarksRows` — AC: WP-B fix.
  - `AddGrantAsync_ThenGetAllGrants_ContainsGrant` — AC: WP-H grants.
  - `RemoveGrantAsync_ThenGetAllGrants_DoesNotContainGrant` — AC: WP-H grants.
  - `ConnectBookmarkTagAsync_ThenDisconnectBookmarkTagAsync_Roundtrip` — AC: WP-H bookmarks.
  - `MergeTagAsync_RewiresTags_MoviesToTarget` — AC: WP-C merge.
  - `MergeTagAsync_DeletesSourceTag` — AC: WP-C merge.
  - `MergeTagAsync_DoesNotDuplicateMovies_WhenOverlapExists` — AC: WP-C merge overlap.
  - `GetMergeImpactAsync_OverlapMovies_CountsCorrectly` — AC: WP-C impact query.
  - *(Already passing from WP-021: `CreateCategory_And_GetAll_RoundTrip`, `DeleteCategory_WithTags_Throws`, `CreateTag_And_GetById_RoundTrip`, `DeleteTag_CascadesAssociationsAndReparentsChildren` [covers tags_movies removal, grant removal, reparenting, and tag-row deletion], `AddGrant_WouldCreateCycle_ThrowsBeforeInsert`, `ConnectMovieTag_And_Disconnect_RoundTrip`, `ConnectBookmarkTag_InsertsTags_bookmarksRow`.)*


## Documentation Updates

- `docs/agents/project-manifest/api-surface.md`
  - Add `MergeTagAsync` to `ITagsRepository` section — WP-C.
  - Add `OverlapMovies` to `TagMergeImpact` model entry — WP-C.
  - Remove "MergeTagAsync bypasses ITagsRepository directly" note from `TagsManager` entry — WP-C.
  - Correct the `TagsManager` constructor signature to `(ITagsRepository repository, IDbConnectionFactory connectionFactory, ISettingsService settingsService)` (remove phantom `ILogger<TagsManager>` parameter, add missing `IDbConnectionFactory`) — manifest repair (pre-existing inaccuracy unrelated to WP-C code changes).
  - Correct the `DapperTagsRepository` constructor signature: remove the phantom `ILogger<DapperTagsRepository>` parameter (actual constructor is `DapperTagsRepository(IDbConnectionFactory connectionFactory)`) — manifest repair.
  - Replace the stale `DapperTagsRepository` comment "No transaction wrapping on DeleteTagAsync cascade" with "DeleteTagAsync wraps its 5 SQL steps in a `BeginTransaction`/`Commit`/`Rollback` transaction with ambient-transaction guard (added WP-021)" — manifest repair.
  - Add `ClearParentTagCommand` to `TagEditorViewModel` entry — WP-A.
  - Add `Dispose()` and `IDisposable` to `TaggerViewModel` entry — WP-D.
  - Add `LastOperationError` observable property to `TaggerViewModel` entry — WP-E.
  - Update `TaggerTagViewModel` constructor signature to include `onError` parameter — WP-E.
- `docs/agents/project-manifest/constraints.md`
  - Remove resolved debt: `tags_bookmarks` orphan rows entry — WP-B.
  - Remove `TaggerViewModel` event-subscription leak entry; replace with resolved description — WP-D.
  - Remove `MovieEditorViewModel.LoadAsync` single-call invariant gotcha; replace with resolved note — WP-D.
  - Update `TagEditorView SelectedParentTag` entry: mark resolved, note `ClearParentTagCommand` — WP-A.
  - Add note that `TagsManager.LoadUsageCountsAsync` still uses `_connectionFactory` directly (out-of-scope deviation, retained for documentation).
  - Update `TaggerViewModel event-subscription lifecycle` gotcha to reflect `IDisposable` implemented — WP-D.
  - Remove `TaggingOptions.MostUsedThreshold lacks range validation` entry — WP-J3.
  - Remove `DeleteImpactRow / MergeImpactRow duplication` entry — WP-J1.
  - Remove `MoviesListViewModel ambiguous constructor selection` entry — WP-J2.
  - Remove `Pre-existing CS8765 nullability warnings` entry once resolved — WP-J5.
  - Remove `GetMergeImpactAsync ignores targetTagId` entry — WP-C.
  - Remove `Tag toggle fire-and-forget swallows exceptions` entry — WP-E.


## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **`tags_bookmarks` FK may already have `ON DELETE CASCADE`** | Inspect DDL in migration files first (WP-B step 1). If cascade exists, the application-level DELETE is a no-op. Adding it is still safe; it will simply affect zero rows when the FK fires first. |
| **Ambient-transaction guard in `DapperTagsRepository.MergeTagAsync` may fail differently from `DeleteTagAsync`** | Use identical `try/catch(InvalidOperationException)` pattern. WP-H test `MergeTagAsync_DoesNotDuplicateMovies_WhenOverlapExists` runs inside the shared-rollback fixture, exercising the ambient path. |
| **`TaggerViewModel.Dispose()` called from `Unloaded` before all async operations complete** | Guard with `_disposed` flag at the top of `OnIsCheckedStoredChanged`; if disposed, skip the callback entirely. |
| **`CommandDefinition` breaks existing transaction-passing call sites** | `CommandDefinition` has a `transaction` overload parameter; existing sites that pass `ownedTx` must use the `CommandDefinition(sql, param, ownedTx, cancellationToken: ct)` constructor, not the two-parameter one. Review every call site in WP-I individually. |
| **WP-C changes `TagMergeImpact` record** — any existing `with { }` copy expressions that do not set `OverlapMovies` will use the default value 0, which is safe but may give a misleading impact preview | All `TagMergeImpact` construction is in `DapperTagsRepository.GetMergeImpactAsync`; update it in the same WP-C commit. `FakeTagsManager.GetMergeImpactAsync` in test helpers must also be updated to set a non-zero `OverlapMovies` in tests that test overlap detection. |
| **CS8765 suppression in TagsManagerTests.cs** | Prefer nullable annotation fix over `#pragma`; ensure the fix applies only to the specific lines (344, 372) and does not introduce unintended nullability warnings elsewhere. |
