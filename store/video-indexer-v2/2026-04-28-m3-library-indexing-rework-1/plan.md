# Plan — M3 Library & Indexing Rework 1

## Summary

This rework addresses all actionable items identified in the M3 synthesis report dated 2026-05-08. It covers the schema reference debt in `structure.sql`, the live-DB test isolation race, the scoped orphan DELETE improvement, input validation guards at service boundaries, and two view-model hygiene fixes (`RefreshIndexViewModel` subscription leak and unobserved exception in `MainContentViewModel`). A secondary goal is to eliminate duplicated connection-resolution code (the `TryResolveConfig`/`ParseConfigFile`/`ParseEnvVar`/`BuildConnectionString` methods and `TestConnectionConfig` record duplicated verbatim across the four live-DB test classes) into a shared `LiveDbFixture`.

> **Synthesis correction:** The synthesis states the `UNIQUE KEY` on `movies_filenames.hash` is absent from `structure.sql`. A pre-plan inspection of `spdb-indexer/SPDB Indexer/sql/structure.sql` (lines 293–296) confirms it **is already present** as `ADD UNIQUE KEY hash_2 (hash)`. The actual schema gap is the absence of the `year`, `season`, and `episode` nullable columns on the `movies` table, which is documented in `README.md` but not reflected in `structure.sql`.


## Architectural Context

| Layer | Key components |
|---|---|
| Reference schema | `spdb-indexer/SPDB Indexer/sql/structure.sql` — the single-source-of-truth DDL for the `spdb` database; shared by both projects via the multi-root workspace |
| Infrastructure | `VideoIndexer.Infrastructure/Library/SpdbConfigLibraryFolderRepository.cs` — manages library folder persistence and cascade-delete |
| Infrastructure | `VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs` — thin Dapper wrapper around `movies` / `movies_filenames` |
| App ViewModels | `VideoIndexer.App/ViewModels/RefreshIndexViewModel.cs` — subscribes to `IRefreshOrchestrator.StateChanged` |
| App ViewModels | `VideoIndexer.App/ViewModels/MainContentViewModel.cs` — already implements `IDisposable`; fires-and-forgets `RefreshCountAsync` |
| Live-DB tests | `tests/VideoIndexer.Infrastructure.Tests/Library/SpdbConfigLibraryFolderRepositoryTests.cs` — embeds a private `LiveConnectionFactory : IDbConnectionFactory` sealed class that opens a fresh connection per SUT call |
| Live-DB tests | `tests/VideoIndexer.Infrastructure.Tests/Library/DapperMovieCatalogRepositoryTests.cs`, `LibraryScannerIntegrationTests.cs` — each embed `SharedConnectionFactory` + `NonDisposingConnection` inner classes that route all SUT calls through a single shared `MySqlTransaction`, enabling rollback-based cleanup (no `LiveConnectionFactory`) |
| Live-DB tests | `tests/VideoIndexer.Infrastructure.Tests/Database/SpdbConfigRepositoryTests.cs` — uses `[SkippableFact]`-guarded live-DB tests with the same `TryResolveConfig`/`ParseConfigFile`/`ParseEnvVar`/`BuildConnectionString`/`TestConnectionConfig` duplication as the Library classes; uses `_csb` directly via inner `SingleConnectionFactory` + `NonDisposingConnectionWrapper` classes (not connection-resolution code — do not remove); carries no `[Collection]` attribute |

There is no existing shared test-base class or `[Collection]` isolation in `VideoIndexer.Infrastructure.Tests`. The `[CollectionDefinition]` + `[Collection]` mechanism (xUnit) is the established pattern for serializing test-class execution.


## Approach / Architecture

Seven discrete, independent changes — each touching a single concern — delivered in a single rework milestone. Changes are grouped into three work streams:

**Stream A — Schema**
- Read the live `spdb_tests` schema DDL for `movies` (`SHOW CREATE TABLE movies`) to confirm the exact types of `year`, `season`, and `episode`.
- Update `structure.sql` to include those columns.

**Stream B — Infrastructure correctness**
- Scope `RemoveAsync` orphan DELETE to the set of hashes collected in step 3 rather than a global `WHERE NOT EXISTS` scan.
- Add `ArgumentException.ThrowIfNullOrWhiteSpace` guards to every public method that accepts a `string` parameter in `SpdbConfigLibraryFolderRepository` and `DapperMovieCatalogRepository`.

**Stream C — Test quality**
- Add a single `[CollectionDefinition("LiveDB", DisableParallelization = true)]` declaration in a new `LiveDbCollection.cs` file under `VideoIndexer.Infrastructure.Tests`.
- Apply `[Collection("LiveDB")]` to all four live-DB test classes.
- Extract the four connection-resolution methods (`TryResolveConfig`, `ParseConfigFile`, `ParseEnvVar`, `BuildConnectionString`) and the `TestConnectionConfig` record — duplicated verbatim across all four test classes — into a shared `LiveDbFixture`, eliminating ~60–80 lines of duplication per class. `SpdbConfigLibraryFolderRepositoryTests` retains a slim `LiveConnectionFactory` inner class that wraps `LiveDbFixture.Csb`; the other three classes retain their `SharedConnectionFactory` and `NonDisposingConnection` inner classes (where applicable).

**Stream D — View-model hygiene**
- Implement `IDisposable` on `RefreshIndexViewModel`; unsubscribe from `StateChanged` in `Dispose`.
- Update `MainContentViewModel.Dispose` to call `RefreshIndex?.Dispose()`.
- Add a logging `ContinueWith` continuation to the fire-and-forget `_ = RefreshCountAsync()` call in `MainContentViewModel.OnRefreshStateChanged`.


## Rationale

- **Scoped DELETE over global scan:** The global `WHERE NOT EXISTS` touches every `movies` row under concurrent test execution, causing lock contention. *(Note: the synthesis attributed `ER_CHECKREAD` failures specifically to MyISAM tables; however, `structure.sql` declares `movies` as `ENGINE=InnoDB`. The engine used in the live `spdb_tests` schema has not been verified — this is a judgment, not a confirmed fact. The scoped DELETE is the correct approach under either engine.)* Collecting the affected hashes before the `DELETE` from `movies_filenames` and then issuing a targeted `DELETE FROM movies WHERE hash IN (...)` eliminates the contention and makes the DELETE intent explicit to future readers.
- **`[Collection("LiveDB")]` before CI:** The test isolation race reproduces in the full test suite today. Serializing live-DB classes is the correct xUnit-idiomatic fix and requires no schema or code changes.
- **Input guards as one-liners:** `ArgumentException.ThrowIfNullOrWhiteSpace` was introduced in .NET 8. Given `TreatWarningsAsErrors=true` and the .NET 10 target, these guards are idiomatic and prevent deferred null-pointer failures that are harder to trace than a clean `ArgumentException` at the call site.
- **`IDisposable` on `RefreshIndexViewModel`:** `MainContentViewModel` already applies this pattern and already calls `Dispose` on itself. Extending it to cover the child view-model is consistent and defensive.
- **Shared `LiveDbFixture`:** Eliminates byte-for-byte code duplication across three test classes and provides a single place to evolve the connection-resolution logic (e.g., adding retry support or a connection-pool override).


## Detailed Steps

1. **Confirm `movies` live schema** — Connect to `spdb_tests` and run `SHOW CREATE TABLE movies` to obtain the exact type/nullability/default for `year`, `season`, `episode`.

2. **Update `structure.sql`** — In `spdb-indexer/SPDB Indexer/sql/structure.sql`, add the three nullable columns to the `CREATE TABLE movies` block immediately after the `cache` column, matching the types confirmed in step 1. (If live DB access is unavailable, infer types from `spdb-indexer/SPDB Indexer/Classes/Movie.cs` and `spdb-indexer/SPDB Indexer/Classes/DBHelper.cs`; mark inferred values with a `-- TODO: verify` comment in the DDL.)

3. **Scope orphan DELETE in `RemoveAsync`** — In `SpdbConfigLibraryFolderRepository.RemoveAsync`:
   a. Before the existing `DELETE FROM movies_filenames` statement, issue `SELECT DISTINCT hash FROM movies_filenames WHERE filename = @ExactPath OR filename LIKE @LikePrefix ESCAPE '\\\\'` on the same transaction and materialise the results into a `List<string> deletedHashes`.
   b. Execute the existing `DELETE FROM movies_filenames` statement as currently written.
   c. If `deletedHashes` is non-empty, replace the current global `DELETE m FROM movies m WHERE NOT EXISTS (...)` with `DELETE FROM movies WHERE hash IN @Hashes` using a Dapper `IN` expansion (anonymous parameter `new { Hashes = deletedHashes }`).
   d. If `deletedHashes` is empty, skip the `movies` DELETE entirely.

4. **Add input guards to `SpdbConfigLibraryFolderRepository`** — In `AddAsync(string path, ...)`, add `ArgumentException.ThrowIfNullOrWhiteSpace(path, nameof(path))` as the first line of the public method body (before the `_configRepository` branch check).

5. **Add input guards to `DapperMovieCatalogRepository`** — Add `ArgumentException.ThrowIfNullOrWhiteSpace` guards at the start of:
   - `MovieExistsAsync(string hash, ...)`
   - `InsertMovieAsync(string hash, ...)`
   - `FilenameExistsAsync(string filename, ...)`
   - `AddFilenameAsync(string filename, string hash, ...)`

6. **Create `LiveDbCollection.cs`** — Add `tests/VideoIndexer.Infrastructure.Tests/LiveDbCollection.cs` containing `[CollectionDefinition("LiveDB", DisableParallelization = true)]` on a marker class.

7. **Create `LiveDbFixture.cs`** — Extract the connection-resolution logic (env-var → config-file walk-up → `MySqlConnectionStringBuilder` build) from `SpdbConfigLibraryFolderRepositoryTests` into a new `tests/VideoIndexer.Infrastructure.Tests/LiveDbFixture.cs` class that implements `IDisposable`. This class exposes:
   - `string? SkipReason` — non-null when no config was found.
   - `MySqlConnectionStringBuilder? Csb` — the resolved connection string builder.
   - A factory method `CreateConnection()` — wraps `new MySqlConnection(Csb!.ConnectionString)`.

8. **Update live-DB test classes** — Apply the following changes to each class:

   **All four classes (`SpdbConfigLibraryFolderRepositoryTests`, `DapperMovieCatalogRepositoryTests`, `LibraryScannerIntegrationTests`, `SpdbConfigRepositoryTests`):**
   - Add `[Collection("LiveDB")]` attribute.
   - Add `IClassFixture<LiveDbFixture>` to the class declaration and accept `LiveDbFixture fixture` in the constructor.
   - Remove: the `TryResolveConfig`, `ParseConfigFile`, `ParseEnvVar`, `BuildConnectionString` static methods and the `TestConnectionConfig` record (replaced by `LiveDbFixture`).
   - Do **not** wholesale-replace all `_skipReason` field references: classes that assign `_skipReason` at runtime for per-class conditions must **retain a private `_skipReason` field** that is first initialised from `fixture.SkipReason` and then augmented with the class-specific reason. Specifically: `LibraryScannerIntegrationTests` sets `_skipReason` when `test-folder/` is missing (before the connection attempt); `DapperMovieCatalogRepositoryTests` and `LibraryScannerIntegrationTests` set it when `MySqlException` is thrown during connection setup. Replace `_csb` usages with `fixture.Csb` directly.

   **`SpdbConfigLibraryFolderRepositoryTests` only:**
   - Retain the private `LiveConnectionFactory : IDbConnectionFactory` inner class, but update it to accept `MySqlConnectionStringBuilder` from `fixture.Csb` rather than from a locally resolved config. The constructor call at usage sites becomes `new LiveConnectionFactory(fixture.Csb!)`.
   - `_skipReason` is only ever set from `TryResolveConfig`; replace it entirely with `fixture.SkipReason` (no per-class augmentation needed).

   **`DapperMovieCatalogRepositoryTests` and `LibraryScannerIntegrationTests`:**
   - Retain the `SharedConnectionFactory`, `NonDisposingConnection`, and (for `LibraryScannerIntegrationTests`) `FakeFolderRepository` inner classes — these encode shared-transaction isolation, not connection resolution, and must not be moved to `LiveDbFixture`.
   - In the constructor, replace `BuildConnectionString(cfg).ConnectionString` with `fixture.Csb!.ConnectionString` when opening the shared `MySqlConnection`.
   - Retain a private `_skipReason` field; initialise it from `fixture.SkipReason` in the constructor and then augment with per-class skip conditions as described above.

   **`SpdbConfigRepositoryTests` only:**
   - This class uses `_csb` directly (no `SharedConnectionFactory` or `LiveConnectionFactory`). Replace `_csb` usages with `fixture.Csb` throughout.
   - `_skipReason` is only ever set from `TryResolveConfig`; replace it entirely with `fixture.SkipReason` (no per-class augmentation needed).

9. **Implement `IDisposable` on `RefreshIndexViewModel`** — Change `public sealed partial class RefreshIndexViewModel` to implement `IDisposable`. Add `Dispose()` that calls `_orchestrator.StateChanged -= OnStateChanged`.

10. **Propagate dispose to `RefreshIndexViewModel` from `MainContentViewModel`** — In `MainContentViewModel.Dispose()`, after the existing `_orchestrator.StateChanged -= OnRefreshStateChanged` line, add `RefreshIndex?.Dispose()`.

11. **Add exception logging to fire-and-forget `RefreshCountAsync` call** — In `MainContentViewModel.OnRefreshStateChanged`, replace:
    ```csharp
    _ = RefreshCountAsync();
    ```
    with:
    ```csharp
    _ = RefreshCountAsync().ContinueWith(
        t => _logger.LogError(t.Exception!.GetBaseException(), "RefreshCountAsync failed after state change"),
        CancellationToken.None,
        TaskContinuationOptions.OnlyOnFaulted,
        TaskScheduler.Default);
    ```
    (Use the 4-argument overload to avoid capturing `TaskScheduler.Current`, which can be the UI dispatcher in Avalonia contexts and would trigger CA2008. `GetBaseException()` unwraps the `AggregateException` wrapping produced by `ContinueWith`, logging the root cause rather than the wrapper.)
    This requires injecting `ILogger<MainContentViewModel>` into the full constructor and storing it as a field. In the **parameterless constructor**, initialize `_logger = NullLogger<MainContentViewModel>.Instance;` (add `using Microsoft.Extensions.Logging.Abstractions;` if not already present) — this is required because `dotnet_diagnostic.CS8618.severity = error` is set in `.editorconfig`, which treats non-nullable uninitialized fields as a build error. Add `using Microsoft.Extensions.Logging;` where needed and register nothing new in DI (the logger is already wired by `AddLogging`).

12. **Update unit tests for changed behaviour** — Add or update tests for:
    - `ArgumentException` thrown from `SpdbConfigLibraryFolderRepository.AddAsync` with null/whitespace path (unit test via `ISpdbConfigRepository` seam).
    - `ArgumentException` thrown from `DapperMovieCatalogRepository` methods with null/whitespace arguments.
    - `RefreshIndexViewModel.Dispose` unsubscribes from `StateChanged` (assert no handler is called after dispose).
    - Scoped DELETE test: verify that removing a folder with hashes `{A, B}` does NOT delete a `movies` row with hash `C`.


## Dependencies

- `TreatWarningsAsErrors=true` on production projects (`VideoIndexer.Infrastructure`, `VideoIndexer.App`) — all added production code must compile warning-free. Test projects (`VideoIndexer.Infrastructure.Tests`) override this to `false` by design.
- .NET 10 — `ArgumentException.ThrowIfNullOrWhiteSpace` is available (.NET 8+).
- xUnit `[Collection]` + `IClassFixture<T>` — standard xUnit v2 mechanisms not yet used in `VideoIndexer.Infrastructure.Tests`; Steps 6–8 introduce them for the first time.
- Dapper `IN` expansion via anonymous `new { Hashes = list }` — supported by Dapper 2.x (already a dependency).
- Live `spdb_tests` DB access — required for step 1 only; all other changes are local.


## Required Components

**Modified files:**
- `spdb-indexer/SPDB Indexer/sql/structure.sql`
- `src/VideoIndexer.Infrastructure/Library/SpdbConfigLibraryFolderRepository.cs`
- `src/VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs`
- `src/VideoIndexer.App/ViewModels/RefreshIndexViewModel.cs`
- `src/VideoIndexer.App/ViewModels/MainContentViewModel.cs`
- `tests/VideoIndexer.Infrastructure.Tests/Library/SpdbConfigLibraryFolderRepositoryTests.cs`
- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperMovieCatalogRepositoryTests.cs`
- `tests/VideoIndexer.Infrastructure.Tests/Library/LibraryScannerIntegrationTests.cs`
- `tests/VideoIndexer.Infrastructure.Tests/Database/SpdbConfigRepositoryTests.cs`
- `tests/VideoIndexer.Tests/SpdbConfigLibraryFolderRepositoryUnitTests.cs`
- `tests/VideoIndexer.App.Tests/MainContentViewModelTests.cs`

**New files:**
- `tests/VideoIndexer.Infrastructure.Tests/LiveDbCollection.cs` — `[CollectionDefinition]` marker
- `tests/VideoIndexer.Infrastructure.Tests/LiveDbFixture.cs` — shared connection resolution + factory


## Assumptions

- The live `spdb_tests` database is accessible from the developer's workstation to execute `SHOW CREATE TABLE movies` (step 1). If not, the column types for `year`, `season`, `episode` must be inferred from the SPDB Indexer source code.
- `ILogger<MainContentViewModel>` is already resolvable from the DI container; `ILogger<T>` is registered by `Host.CreateApplicationBuilder` (default logging) and the `AddSerilog(...)` call in `Program.cs` — no further DI registration is required.
- Dapper's `IN` expansion handles an empty list gracefully — callers must guard against empty sets before issuing the DELETE (per step 3d).


## Constraints

- Do not change the public API surface of `ILibraryFolderRepository`, `IMovieCatalogRepository`, `IRefreshOrchestrator`, or any `IFfprobeRunner` interface — this rework is purely an implementation and test quality pass.
- Do not alter the `RemoveAsync` transactional structure (single connection + transaction). Step 3 may add a `SELECT DISTINCT hash` query and an empty-list guard before/around the existing DELETE statements; the overall connection and transaction lifecycle must not change.
- `LiveDbFixture` must preserve the existing schema-name guard (`spdb_tests` only) and the per-test cleanup contracts held by each test class `Dispose`.


## Out of Scope

- `SpdbConfigLibraryFolderRepository.AddAsync` counter race (`SELECT ... FOR UPDATE`) — deferred; see `m3-library-indexing.md` Deferred Technical Debt.
- `FfprobeRunner` binary path allowlist — deferred; see `m3-library-indexing.md` Deferred Technical Debt.
- `FfprobeRunner` process kill on cancellation — deferred; see `m3-library-indexing.md` Deferred Technical Debt.
- `RefreshState.Cancelled` enum value — deferred to M4 if UI requires it.
- `IMovieCatalogRepository` interface extension for `year`/`season`/`episode` — deferred to the M4 WP that first needs those fields.
- M4 feature planning — covered separately in `m4-movies-list.md`.
- MyISAM → InnoDB migration for the `movies` table — the scoped DELETE in step 3 eliminates the acute test isolation issue without requiring a schema migration.


## Acceptance Criteria

- `spdb-indexer/SPDB Indexer/sql/structure.sql` `CREATE TABLE movies` includes `year`, `season`, `episode` columns matching the live schema.
- All four live-DB test classes (`SpdbConfigLibraryFolderRepositoryTests`, `DapperMovieCatalogRepositoryTests`, `LibraryScannerIntegrationTests`, `SpdbConfigRepositoryTests`) carry `[Collection("LiveDB")]`; the `[CollectionDefinition]` marker exists.
- All four live-DB test classes use `LiveDbFixture` for connection resolution — the `TryResolveConfig`, `ParseConfigFile`, `ParseEnvVar`, `BuildConnectionString` static methods and `TestConnectionConfig` record are removed from all four classes. `SpdbConfigLibraryFolderRepositoryTests` retains its `LiveConnectionFactory` inner class, but its constructor no longer accepts a locally resolved `MySqlConnectionStringBuilder` — it accepts `fixture.Csb` instead.
- `SpdbConfigLibraryFolderRepository.RemoveAsync` issues a `DELETE FROM movies WHERE hash IN (...)` scoped to the hashes removed in step 3, not a global `WHERE NOT EXISTS` scan.
- `SpdbConfigLibraryFolderRepository.AddAsync` and all four `DapperMovieCatalogRepository` public methods throw `ArgumentException` for null/whitespace string arguments.
- `RefreshIndexViewModel` implements `IDisposable`; `MainContentViewModel.Dispose` calls `RefreshIndex?.Dispose()`.
- `MainContentViewModel.OnRefreshStateChanged` logs exceptions from the `RefreshCountAsync` fire-and-forget.
- All existing tests pass (`dotnet test`); no new test failures; the pre-existing live-DB skips remain.
- Production build (`VideoIndexer.Infrastructure`, `VideoIndexer.App`) produces zero warnings (`TreatWarningsAsErrors=true`).


## Testing Strategy

Each changed component has a corresponding unit-test change:

| Change | Test |
|---|---|
| `AddAsync` null guard | New test method in `tests/VideoIndexer.Tests/SpdbConfigLibraryFolderRepositoryUnitTests.cs` (seam path via `ISpdbConfigRepository`) |
| `DapperMovieCatalogRepository` null guards | New unit tests in the existing `DapperMovieCatalogRepositoryTests` (unit section) |
| Scoped DELETE | New live-DB integration test in `tests/VideoIndexer.Infrastructure.Tests/Library/SpdbConfigLibraryFolderRepositoryTests.cs` (`[SkippableFact]`): insert `movies` rows for hashes A, B, C; insert `movies_filenames` rows mapping A and B to the target folder path; call `RemoveAsync`; assert the `movies` row for hash C survives and those for A and B are deleted. |
| `RefreshIndexViewModel` dispose | New unit test in `tests/VideoIndexer.App.Tests/RefreshIndexViewModelTests.cs`: construct → dispose → fire `StateChanged` → assert `IsRunning` unchanged |
| `MainContentViewModel.Dispose` propagates to `RefreshIndex` | New unit test in `tests/VideoIndexer.App.Tests/MainContentViewModelTests.cs`: construct full SUT → call `sut.Dispose()` → fire `StateChanged` on the orchestrator → assert `RefreshIndex.IsRunning` is unchanged (i.e., the child VM's handler was unsubscribed). |
| `[Collection("LiveDB")]` | Verify by running `dotnet test --filter "LiveDB"` — tests must run sequentially, no `ER_CHECKREAD` |


## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **Live schema inspection unavailable (step 1)** | Fall back to SPDB Indexer C# source code (`Classes/DBHelper.cs`, `Classes/Movie.cs`) to infer column types; mark inferred types with a `TODO: verify` comment in `structure.sql` |
| **Dapper `IN` with empty list emits invalid SQL** | Guard: skip the DELETE entirely if `deletedHashes.Count == 0` (step 3c already requires this) |
| **`IClassFixture<LiveDbFixture>` conflicts with existing `IDisposable` cleanup in test classes** | Each test class retains its own `Dispose` for test-data teardown; `LiveDbFixture.Dispose` only owns connection-string resources (no data cleanup) — the two responsibilities do not overlap |
| **`ILogger<MainContentViewModel>` injection breaks parameterless constructor** | The parameterless constructor (test/factory path) will require either `NullLogger<MainContentViewModel>.Instance` as default or an optional logger parameter; choose `NullLogger` to keep the parameterless path clean |
