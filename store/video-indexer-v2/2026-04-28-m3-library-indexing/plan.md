# Plan — M3 Library & Indexing

## Summary

Stand up the **library and indexing** layer of Video Indexer MKII on top of the M2
database/auth shell. M3 introduces (a) per-database **registered library folders**
persisted server-side in `spdb_config`, (b) the **path-based hashing** pipeline driven
by `ffprobe` and the 41-key flat-dictionary algorithm, (c) an **obfuscation-aware
scanner** that skips already-renamed files and reconciles them via `movies_filenames`,
(d) a **Refresh Index** background worker that walks every registered folder, computes
hashes, and inserts/updates rows in `movies` and `movies_filenames`, and (e) the
minimum **Library Folders** UI plus a *Refresh Index* command bound into the
post-`Ready` shell content. Done means: from a fresh clone, after the M2
`Connecting → LoggingOn → Ready` flow completes, the user can register the
[`test-folder/`](../../../../test-folder/) directory, click *Refresh Index*, and observe
that the database `movies` table is populated with rows for the obfuscated files in
that folder, with no duplicates on a second refresh, no exceptions, and no movie editor
or grid UI yet (those are M4–M7).

## Architectural Context

The repository contains the M1 + M2 shell at [src/](../../../../src/):

- **`VideoIndexer.Core`** — pure abstractions / options / enums / events.
  M2 added [`Options/DatabaseOptions.cs`](../../../../src/VideoIndexer.Core/Options/DatabaseOptions.cs),
  [`Abstractions/IDbConnectionFactory.cs`](../../../../src/VideoIndexer.Core/Abstractions/IDbConnectionFactory.cs),
  [`Abstractions/ISpdbConfigRepository.cs`](../../../../src/VideoIndexer.Core/Abstractions/ISpdbConfigRepository.cs),
  [`Enums/ShellState.cs`](../../../../src/VideoIndexer.Core/Enums/ShellState.cs).
- **`VideoIndexer.Infrastructure`** — Dapper + `MySqlConnector`. M2 added
  [`Database/MySqlConnectionFactory.cs`](../../../../src/VideoIndexer.Infrastructure/Database/MySqlConnectionFactory.cs),
  [`Database/SpdbConfigRepository.cs`](../../../../src/VideoIndexer.Infrastructure/Database/SpdbConfigRepository.cs),
  [`Database/DatabaseBootstrapper.cs`](../../../../src/VideoIndexer.Infrastructure/Database/DatabaseBootstrapper.cs).
- **`VideoIndexer.App`** — Avalonia + DI bridge.
  [`ViewModels/ShellViewModel.cs`](../../../../src/VideoIndexer.App/ViewModels/ShellViewModel.cs)
  is the state machine; [`ViewModels/MainContentViewModel.cs`](../../../../src/VideoIndexer.App/ViewModels/MainContentViewModel.cs)
  is the empty placeholder M3 is allowed to fill.
- **Tests** — [`tests/VideoIndexer.Tests/`](../../../../tests/VideoIndexer.Tests/),
  [`tests/VideoIndexer.Infrastructure.Tests/`](../../../../tests/VideoIndexer.Infrastructure.Tests/),
  [`tests/VideoIndexer.App.Tests/`](../../../../tests/VideoIndexer.App.Tests/) — 110/110
  passing on xUnit + FluentAssertions + `Xunit.SkippableFact`. Shared fixtures live
  under `tests/VideoIndexer.Tests/Fixtures/`.

Authoritative inputs for M3:

- [docs/projects/rebuild/management-areas/movie-management-specification.md](../../../projects/rebuild/management-areas/movie-management-specification.md)
  §1 (Library Management) — Add Folder / Remove Folder / Refresh Index workflow,
  supported video extensions, "creation time, not last-write" rule.
- [docs/projects/rebuild/hashing-specification.md](../../../projects/rebuild/hashing-specification.md)
  — full 41-key MD5 hashing algorithm and the obfuscation early-exit.
- [docs/projects/rebuild/obfuscation-specification.md](../../../projects/rebuild/obfuscation-specification.md)
  — extension map, insertion-order reverse-lookup rule, `Obfuscation_Enabled`
  global setting (read-only in M3; the *toggle* UI ships in M8).
- [docs/projects/rebuild/milestones/roadmap.md](../../../projects/rebuild/milestones/roadmap.md)
  — M3 scope: "Folder registration, refresh worker, path-based hashing,
  obfuscation-aware scan." Roadmap rule: prior milestones must still launch cleanly.

Legacy reference (read-only, used to confirm SQL shapes — never imported):

- [`spdb-indexer/SPDB Indexer/sql/structure.sql`](../../../../../spdb-indexer/SPDB%20Indexer/sql/structure.sql)
  lines 54–115: `movies (movie_id, hash, studio, name, label, description, rating,
  watch_count, open_count, last_watched, review, review_message, cache)` and
  `movies_filenames (filename, hash)`.
- [`spdb-indexer/SPDB Indexer/Classes/Folder.cs`](../../../../../spdb-indexer/SPDB%20Indexer/Classes/Folder.cs)
  `RefreshIndex` / `_AddMovie` — duplicate detection by hash, "file disappeared"
  handling, `Movie.HasValidMovieExtension` gate.
- [`spdb-indexer/SPDB Indexer/Classes/Indexer.cs`](../../../../../spdb-indexer/SPDB%20Indexer/Classes/Indexer.cs)
  lines 39–115 — `AddFolder`, `NextFolderID` (uses `spdb_config.folder_counter`),
  `RemoveFolders` semantics.
- The legacy `Movie.cs` / `Obfuscator.cs` extension maps and obfuscation flag.
- The legacy `Hash.cs` / ffprobe wrapper used to hash files. The rebuild
  re-implements per the spec — it does **not** port the legacy code.

### Conventions inherited from M1 + M2 (MUST be preserved)

- Sealed records with `init`-only properties for option/DTO types.
- Full nullable annotation, `TreatWarningsAsErrors=true`,
  `<WarningLevel>9999</WarningLevel>`.
- Atomic file writes via `.tmp` + `File.Move(overwrite:true)` (M3 does not write
  user files on the JSON path; this still applies if any new local persistence is
  added).
- `xUnit` + `FluentAssertions`; naming `Subject_Scenario_ExpectedBehavior`;
  in-memory fakes; `IDisposable` temp-directory fixtures.
- Dependency direction: Core → ∅; Infrastructure → Core; App → Core +
  Infrastructure; Tests → Core + Infrastructure.
- Numbered 7-step DI registration block in
  [`Program.cs`](../../../../src/VideoIndexer.App/Program.cs); insert M3 services
  inside the existing block, not in a new one.
- Live-DB integration tests use the rollback-only discipline established in M2:
  every write inside a `MySqlTransaction` that is **always rolled back** in
  `Dispose`; never `CREATE`/`DROP`/`ALTER`/`TRUNCATE`; row keys namespaced with a
  `vi_test_` prefix; `[SkippableFact]` when no test config is present.

## Approach / Architecture

### New solution shape (additions only)

```
src/
├── VideoIndexer.Core/
│   ├── Abstractions/
│   │   ├── ILibraryFolderRepository.cs        (NEW)
│   │   ├── IMovieCatalogRepository.cs         (NEW: Insert movie by hash, GetByHash, Count, list/insert/delete movies_filenames rows — one aggregate for movies + movies_filenames)
│   │   ├── IFfprobeRunner.cs                  (NEW: returns raw JSON output for a file)
│   │   ├── IVideoHasher.cs                    (NEW: input = path + obfuscation map; output = uppercase MD5 hex or filename-stem)
│   │   ├── IObfuscationMap.cs                 (NEW: two-way extension lookup, immutable)
│   │   ├── ILibraryScanner.cs                 (NEW: RefreshAsync(IProgress<RefreshProgress>, ct))
│   │   └── IRefreshOrchestrator.cs            (NEW: background-refresh coordinator that owns the worker task; ensures only one refresh at a time)
│   ├── Enums/
│   │   ├── RefreshOutcome.cs                  (NEW: Inserted | FilenameAdded | SkippedAlreadyKnown | SkippedUnsupportedExtension | Failed)
│   │   └── RefreshState.cs                    (NEW: Idle | Running | Completed | Faulted)
│   ├── Models/                                (NEW folder)
│   │   ├── LibraryFolder.cs                   (NEW: Id (long), Path, AddedUtc)
│   │   ├── RefreshProgress.cs                 (NEW: CurrentFolder, CurrentFile, Processed, Total, Outcomes counters)
│   │   └── RefreshSummary.cs                  (NEW: end-of-run roll-up)
│   └── Options/
│       ├── LibraryOptions.cs                  (NEW: FfprobePath, RescanExistingFiles flag)
│       └── AppOptions.cs                      (MODIFIED: add Library property)
├── VideoIndexer.Infrastructure/
│   ├── Library/
│   │   ├── SpdbConfigLibraryFolderRepository.cs   (NEW: serialises folders + counter into spdb_config)
│   │   ├── DapperMovieCatalogRepository.cs        (NEW: implements IMovieCatalogRepository)
│   │   ├── ExtensionObfuscationMap.cs             (NEW: hard-coded insertion-ordered map)
│   │   ├── FfprobeRunner.cs                       (NEW: System.Diagnostics.Process wrapper)
│   │   ├── PathBasedVideoHasher.cs                (NEW: implements the 41-key spec)
│   │   └── LibraryScanner.cs                      (NEW: walks folders, dispatches to hasher + repos)
│   └── (no existing files modified except DI)
└── VideoIndexer.App/
    ├── ViewModels/
    │   ├── LibraryFoldersViewModel.cs         (NEW: list / add / remove)
    │   ├── RefreshIndexViewModel.cs           (NEW: progress + cancel)
    │   ├── MainContentViewModel.cs            (MODIFIED: hosts Library Folders entry + Refresh button + indexed-movie count text)
    │   └── ShellViewModel.cs                  (UNCHANGED — M3 plugs into Ready content only)
    ├── Views/
    │   ├── LibraryFoldersView.axaml(.cs)      (NEW: folder list, Add Folder, Delete Selected)
    │   ├── RefreshIndexView.axaml(.cs)        (NEW: progress bar overlay)
    │   └── MainContentView.axaml(.cs)         (MODIFIED: minimal toolbar + content host)
    └── Assets/appsettings.json                (MODIFIED: add Library section with default null Ffprobe path)

tests/
├── VideoIndexer.Tests/
│   ├── ExtensionObfuscationMapTests.cs        (NEW)
│   ├── PathBasedVideoHasherTests.cs           (NEW: feeds canned ffprobe JSON via fake IFfprobeRunner)
│   ├── LibraryScannerTests.cs                 (NEW: in-memory repos + fake hasher + temp-dir of fake files)
│   ├── SpdbConfigLibraryFolderRepositoryUnitTests.cs (NEW: serialisation round-trip on the spdb_config repo via in-memory ISpdbConfigRepository)
│   ├── RefreshOrchestratorTests.cs            (NEW: single-flight + cancellation semantics)
│   └── Fixtures/
│       ├── FakeFfprobeRunner.cs               (NEW: returns canned JSON keyed by absolute path)
│       └── TempVideoFolder.cs                 (NEW: creates dummy files with valid extensions for scanner tests)
└── VideoIndexer.Infrastructure.Tests/
    ├── Library/
│   ├── DapperMovieCatalogRepositoryTests.cs       (NEW: live-DB, rollback-only, vi_test_ namespaced rows)
    │   ├── SpdbConfigLibraryFolderRepositoryTests.cs  (NEW: live-DB, rollback-only)
    │   ├── FfprobeRunnerTests.cs                      (NEW: probes one of the test-folder/.pkg files, asserts JSON shape; SkippableFact when ffprobe binary missing)
    │   ├── PathBasedVideoHasherIntegrationTests.cs    (NEW: hashes a known test-folder file end-to-end and asserts determinism across two runs; SkippableFact when ffprobe binary missing)
    │   └── LibraryScannerIntegrationTests.cs          (NEW: registers test-folder as a folder, runs RefreshAsync against live DB, asserts inserts + idempotent re-run; SkippableFact)
```

### Composition root changes

Inside the existing numbered DI block in
[`Program.cs`](../../../../src/VideoIndexer.App/Program.cs), insert a new step
*"7: Library & Indexing"* between the database-bootstrapper registration (step 6)
and the view-model registration (now step 8). All registrations are singletons except
the view-models (transient):

- `IObfuscationMap` → `ExtensionObfuscationMap` (singleton, immutable).
- `IFfprobeRunner` → `FfprobeRunner` (singleton, resolves the `ffprobe` binary via a three-step chain: `ExternalTools.Ffmpeg.FfprobePath` → `LibraryOptions.FfprobePath` → `ffprobe[.exe]` on `PATH`).
- `IVideoHasher` → `PathBasedVideoHasher` (singleton).
- `ILibraryFolderRepository` → `SpdbConfigLibraryFolderRepository` (singleton).
- `IMovieCatalogRepository` → `DapperMovieCatalogRepository` (singleton).
- `ILibraryScanner` → `LibraryScanner` (singleton — stateless; same instance is safe across calls).
- `IRefreshOrchestrator` → singleton (single-flight guard).
- `LibraryFoldersViewModel`, `RefreshIndexViewModel` — transient.
- `MainContentViewModel` — transient (registering it in DI is required because
  M3 injects `IMovieCatalogRepository` and `IRefreshOrchestrator` into it).

In the existing **step 8** block make two additional changes:
1. Update the `ShellState.Ready` arm of the view-model factory from
   `new MainContentViewModel()` to `sp.GetRequiredService<MainContentViewModel>()`
   so injected services are resolved from DI instead of being silently null.
2. Add parameterless-constructor factory registrations for the two new content
   views alongside the existing ones:
   `builder.Services.AddTransient<LibraryFoldersView>(_ => new LibraryFoldersView())`
   `builder.Services.AddTransient<RefreshIndexView>(_ => new RefreshIndexView())`

The Generic Host is unchanged. `MainContentView` now owns a small toolbar (Library
Folders, Refresh Index) + a text line showing "Indexed movies: N" — this is the only
M3 surface the user sees inside the Ready state.

### Library-folder persistence model (DB-side)

Folders are stored **server-side in `spdb_config`** so they survive across machines
that point at the same database (matches the legacy `folder_counter` convention and
the spec's promise that the catalog lives in the DB):

| `config_name`     | `config_value`                                                                    |
| ----------------- | --------------------------------------------------------------------------------- |
| `library_folders` | JSON array of `{ "id": <long>, "path": "<absolute>", "addedUtc": "<ISO 8601>" }`  |
| `folder_counter`  | Last allocated folder id, monotonically increasing (parsed as `long`)             |

`SpdbConfigLibraryFolderRepository` reads/writes both rows inside a single `MySqlTransaction`
(via `IDbConnectionFactory.CreateOpenConnectionAsync` + `BeginTransaction`) so an `Add` cannot leak an id
without persisting the folder, and a `Remove` cannot leave a stale counter. The repository
injects `IDbConnectionFactory` directly — **not** `ISpdbConfigRepository`, which creates a
new connection per call and cannot share a transaction across multiple write operations. The XML
*library snapshot* file format from the spec §1.4 is **deferred to M8** — M3 ships only
the in-database folder list, which is sufficient for the Refresh-Index acceptance
criteria.

### Hashing pipeline

`PathBasedVideoHasher.ComputeAsync(string absolutePath, CancellationToken)`:

1. Lower-case the file's extension.
2. If it matches a value in `IObfuscationMap.ObfuscatedExtensions` → return
   `Path.GetFileNameWithoutExtension(absolutePath).ToUpperInvariant()` (per the spec's
   "obfuscated files exception"). **No ffprobe invocation.**
3. Otherwise call `IFfprobeRunner.ProbeAsync(absolutePath, ct)`. Parse the JSON as a
   token stream via `Utf8JsonReader`. Build a flat dictionary, capturing the **first
   occurrence only** of each scalar key (string / number). Each value is stored as the
   **raw JSON token text** (`Utf8JsonReader.ValueSpan` decoded as ASCII) — numeric
   tokens are **never** re-rendered via `ToString()` or numeric parsing, so the literal
   `1.0` is stored as `"1.0"`, not `"1"`.
4. Iterate the spec's 41 keys *in order*; collect each present raw token text string.
5. Join with `|`, ASCII-encode, MD5, return uppercase hex.

`FfprobeRunner` shells out to the binary resolved via a three-step chain:
(1) `ExternalTools.Ffmpeg.FfprobePath` (M2.5 provisioner output); (2)
`LibraryOptions.FfprobePath` (manual per-machine override); (3) `ffprobe[.exe]` on `PATH`.
It uses the exact arguments from the spec. Stderr is captured and surfaced via
`InvalidOperationException` only if the exit code is non-zero. Standard output is
returned as a `string`. The runner times out at 30 s per file (configurable via
`LibraryOptions.FfprobeTimeoutSeconds`, default `30`).

### Library scanner

`LibraryScanner.RefreshAsync(IProgress<RefreshProgress>?, CancellationToken)`:

1. Load registered folders via `ILibraryFolderRepository.ListAsync`.
2. For each folder, walk recursively with `Directory.EnumerateFiles(path, "*",
   SearchOption.AllDirectories)`.
3. Filter by extension: an extension is processed if it is either a key
   *(supported native)* or a value *(known obfuscated)* in the obfuscation map. Any
   other file → `RefreshOutcome.SkippedUnsupportedExtension`. The supported-native
   list comes from the obfuscation map keys (ten entries, including `.mpeg` in
   addition to the nine extensions listed in the movie-management spec §1 —
   this discrepancy is intentional: `.mpeg` is present in the obfuscation map
   and safe to include).
4. For each candidate file:
   - `var hash = await hasher.ComputeAsync(path, ct);`
   - `var existing = await movies.GetByHashAsync(hash, ct);`
   - If `existing is null` → `INSERT INTO movies` with empty metadata
     (`studio=''`, `name=''`, `label=''`, `description=''`, `rating=0`, etc.) and
     `INSERT INTO movies_filenames(filename, hash)` for the absolute path. Outcome:
     `Inserted`.
   - Else, ensure the absolute path is present in `movies_filenames`; if not,
     insert it. Update `movies` only if the spec calls for "file size /
     modification time" tracking — for M3, the legacy `movies` table has **no**
     such columns; therefore "update" is limited to ensuring the
     `movies_filenames` row exists. Outcome: `FilenameAdded` if a filename row
     was inserted, otherwise `SkippedAlreadyKnown`.
   - On any per-file exception (ffprobe failure, malformed JSON, IO error):
     log at `Warning`, count as `Failed`, continue with the next file. The scan
     never aborts on a single bad file.
5. Report progress after every file (current path, processed count, totals counted
   eagerly before the loop so the progress bar is accurate).
6. Return a `RefreshSummary` with per-outcome counts and the duration.

The scanner does **not** delete movies whose files no longer exist — that is the
*Remove Folder* flow (which removes the entire folder) and a future "library cleanup"
WP. The legacy "Movie does not exist anymore" log is preserved as an `Information`
log entry only, with no DB side-effect.

### Folder removal semantics

Per spec §1, *Remove Folder* deletes "all movies belonging to removed folders" from
the catalog. M3's `ILibraryFolderRepository.RemoveAsync(long folderId)` therefore:

1. Deletes the folder row from the `library_folders` JSON array in `spdb_config`.
2. Identifies every `movies_filenames.filename` whose absolute path begins with the
   folder's path + `Path.DirectorySeparatorChar`. The SQL pattern is written
   explicitly to prevent `_` and `%` in real paths from over-matching:
   ```sql
   WHERE filename = @folder
      OR filename LIKE CONCAT(REPLACE(REPLACE(@folder, '\\', '\\\\'), '%', '\\%'), @sep, '%') ESCAPE '\\'
   ```
   The case-sensitivity contract is **byte-exact** — paths are matched against the
   value stored on insert; no collation-level folding is applied. A unit test must
   assert that a folder path containing `_` does not match sibling folder paths.
3. For each such filename, deletes the `movies_filenames` row.
4. For any `movies` row whose hash is no longer referenced by *any*
   `movies_filenames` row, deletes the `movies` row.
5. All four steps run inside a single `MySqlTransaction`.

This matches the spec ("source video files on disk are **not** deleted") and is
covered end-to-end by an integration test on `test-folder/`.

### Refresh orchestration & cancellation

`IRefreshOrchestrator` is the **single-flight guard** that the UI binds to:

- `Task<RefreshSummary> StartAsync(IProgress<RefreshProgress>, CancellationToken)`
  — if a run is already in progress throws `InvalidOperationException("A refresh is
  already running.")` immediately; otherwise schedules `LibraryScanner.RefreshAsync`
  on a background `Task.Run` and returns the in-flight `Task<RefreshSummary>` for the
  caller to `await`. The state is flipped `Idle → Running` atomically via
  `Interlocked.CompareExchange<int>` on a backing `int` field (0 = Idle, 1 =
  Running), preventing TOCTOU races without a `lock`.
- **`CancellationTokenSource` lifecycle:** a fresh `CancellationTokenSource` is
  allocated at the start of each run; on completion (success, fault, or
  cancellation) it is disposed and the backing state field is flipped back to
  `Idle` before `StateChanged` is fired.
- `RefreshState State { get; }` and a `StateChanged` event so the view-model can
  toggle the Refresh button.
- `Cancel()` — signals the run's `CancellationTokenSource`. The scanner observes
  the token between files; an in-flight ffprobe call is allowed to finish.

This isolates Avalonia's UI thread from the worker and keeps the scanner itself
trivially unit-testable (no threading concerns inside the scanner).

### `MainContentView` minimum surface

```
┌──────────────────────────────────────────────────────────┐
│ Library Folders…    [Refresh Index]    Indexed: 0        │
├──────────────────────────────────────────────────────────┤
│ (Refresh progress overlay when running)                  │
└──────────────────────────────────────────────────────────┘
```

- *Library Folders…* opens `LibraryFoldersView` (modal-style content swap inside
  `MainContentView`; no second window). The navigation state is tracked by a
  `MainContentPage` enum (`Default | LibraryFolders | RefreshOverlay`) on
  `MainContentViewModel`; `MainContentView` uses an Avalonia `ContentControl` bound
  to `CurrentPage` with a `DataTemplates` block that maps each enum value to the
  appropriate embedded view (`LibraryFoldersView` or `RefreshIndexView`). This avoids
  a second window and keeps navigation state fully in the view-model.
- *Refresh Index* invokes `IRefreshOrchestrator.StartAsync` and shows the
  progress overlay until completion.
- *Indexed: N* is bound to `IMovieCatalogRepository.CountAsync` and refreshes after every
  Refresh-Index completion.

No movies-grid, no editor, no context menu, no playback. Those are M4–M10.

## Rationale

- **DB-side folder list** matches the legacy convention (`folder_counter` already
  lives in `spdb_config`) and lets M3 ship without a JSON-snapshot file format.
  Snapshot import/export is its own well-scoped UX feature for M8.
- **Path-based hash, ffprobe-driven** is the spec's mandated mechanism. Re-using
  the legacy `ffprobe.exe` binary by *path configuration* (rather than vendoring
  it into the rebuild) keeps the cross-platform constraint intact.
- **Obfuscation-aware scan** is required by the spec ("Refresh Index recognizes
  obfuscated files and does not treat them as new entries"). Skipping ffprobe
  for already-obfuscated files matches the spec's *Obfuscated Files Exception*
  and is what makes scanning the seeded `test-folder/` round-trip cleanly.
- **`movies_filenames` as the identity table** is the legacy contract: one
  movie (one hash) can have many on-disk filenames over its lifetime
  (rename, obfuscation toggle, snapshot reuse). M3 honours that table from day
  one so the obfuscation-toggle WP in M8 can land without reshuffling the
  scanner.
- **Single-flight orchestrator** prevents the user from queueing duplicate
  refresh runs, which would otherwise produce concurrent inserts and break the
  `Inserted` vs `FilenameAdded` outcome accounting.
- **Defer the on-disk XML snapshot, the Startup Selection screen, and the
  obfuscation *toggle* UI** — none of these are needed to satisfy the M3 roadmap
  scope, and bundling them invites the same scope creep that produced the
  WP-016/WP-017 closure gap in M2.
- **Use the seeded `test-folder/`** (already on disk, already obfuscated, already
  has DB rows from the production data copy) as the live integration target.
  This gives M3 an honest end-to-end test against real-world data without
  inventing fixtures.

## Detailed Steps

1. **Extend the options tree.**
   - Add `LibraryOptions(FfprobePath?: string, FfprobeTimeoutSeconds: int = 30,
     RescanExistingFiles: bool = false)`.
   - Add `Library` property to `AppOptions`. Update bundled defaults
     ([`Assets/appsettings.json`](../../../../src/VideoIndexer.App/Assets/appsettings.json)).
   - `RescanExistingFiles` reserved for M4+; the M3 scanner ignores it.

2. **Add Core abstractions and models.**
   - Interfaces and enums per the *New solution shape* table above.
   - `LibraryFolder`, `RefreshProgress`, `RefreshSummary` records (sealed, init-only).
     `IndexedMovie` is deferred to M4 (the movie editor is the first consumer).
   - `IObfuscationMap` exposes `IReadOnlyDictionary<string, string> NativeToObfuscated`
     (insertion-ordered) and reverse-lookup methods that respect the spec's
     insertion-order rule (`.mpg`/`.mpeg` → `.res`; reverse `.res` → `.mpg`).

3. **Add Infrastructure implementations.**
   - `ExtensionObfuscationMap` — static, hard-coded.
   - `FfprobeRunner` — `Process.Start` with `UseShellExecute=false`,
     `RedirectStandardOutput=true`, `RedirectStandardError=true`. Honours the
     timeout. Resolves `FfprobePath`; falls back to `ffprobe[.exe]` on `PATH`.
   - `PathBasedVideoHasher` — `Utf8JsonReader` for the token-stream parse
     (avoids materialising the full `JsonDocument`); MD5 via
     `System.Security.Cryptography.MD5.HashData`.
   - `SpdbConfigLibraryFolderRepository` — injects `IDbConnectionFactory` directly
     (not `ISpdbConfigRepository`, which creates a new connection per call and
     cannot share a transaction). Opens one connection, begins a `MySqlTransaction`,
     executes all `spdb_config` reads and writes as Dapper SQL on that single
     connection + transaction, then commits or rolls back atomically. Uses
     `System.Text.Json` for the JSON round-trip on the folder array.
   - `DapperMovieCatalogRepository` — implements `IMovieCatalogRepository`, combining
     both `movies` and `movies_filenames` operations on the same aggregate:
     `INSERT INTO movies (hash, studio, name, label, description, rating, watch_count,
     open_count, last_watched, review, review_message, cache) VALUES (@hash, '', '',
     '', '', 0, 0, 0, NULL, 'no', NULL, '');` + `SELECT * FROM movies WHERE hash =
     @hash;` + `SELECT COUNT(*) FROM movies;` + `movies_filenames` INSERT,
     SELECT-by-hash, SELECT-by-prefix (folder removal), DELETE-by-filename. Keeping
     both tables in one repository preserves the aggregate boundary and makes the
     folder-removal cascade naturally one transaction on one repository.
   - `LibraryScanner` — pure orchestration; takes all dependencies via DI;
     no `Task.Run`, no threading inside.

4. **Register all new services in `Program.cs`** inside the existing numbered
   block. No host-builder structural changes. Specifically:
   - Insert step "7: Library & Indexing" per the *Composition root changes* section above.
   - In the existing step 7 block: register `MainContentViewModel` as transient;
     update `ShellState.Ready` to `sp.GetRequiredService<MainContentViewModel>()`;
     add factory registrations for `LibraryFoldersView` and `RefreshIndexView`.

5. **Build the M3 view-models and views.**
   - `LibraryFoldersViewModel` — `ObservableCollection<LibraryFolder>` plus
     `AddFolderAsync(string path)` and `RemoveSelectedAsync()`. Backed by
     `ILibraryFolderRepository`. Surfaces a folder-picker call through an
     `IFolderPickerService` (small Avalonia adapter, defined in the App project).
   - `RefreshIndexViewModel` — observable `IsRunning`, `Progress`, `Summary`;
     `RunCommand` calls `IRefreshOrchestrator.StartAsync`; `CancelCommand`
     gated on `IsRunning`.
   - `MainContentViewModel` — minimal: `OpenFoldersCommand`, `RefreshCommand`,
     `IndexedMovieCount` (refreshed after each run).
   - Views are bindings-only; ViewLocator picks them up.

6. **Wire startup.**
   - `MainContentViewModel` resolves `IMovieCatalogRepository.CountAsync` lazily on
     activation (so an unreachable DB never crashes the UI before the user has
     seen `MainContentView`).
   - The `Ctrl+T` debug theme cycle from M1/M2 still works; no changes to the
     shell state machine.

7. **Tests — unit (xUnit + FluentAssertions).**
   - `ExtensionObfuscationMapTests` — every native-to-obfuscated mapping;
     reverse lookup of `.res` returns `.mpg` (insertion order); unknown
     extension returns `null`.
   - `PathBasedVideoHasherTests` — fed canned JSON from a `FakeFfprobeRunner`,
     asserts: (a) obfuscated extension short-circuits to filename stem with no
     ffprobe call; (b) the 41-key spec produces a known hex value for a fixture
     JSON; (c) duplicate keys in JSON only capture the first; (d) missing keys
     are skipped (no empty placeholders); (e) the join character is `|`;
     (f) a JSON input containing `"x": 1.0` captures the dictionary entry as
     `"1.0"`, not `"1"` (raw token text, not re-rendered numeric).
   - `LibraryScannerTests` — synthetic temp folder of empty-but-correct-extension
     files plus an in-memory `IMovieCatalogRepository`
     and a fake hasher; asserts: (a) first run emits one `Inserted` per
     supported file; (b) second run emits `SkippedAlreadyKnown` for the same
     files; (c) unsupported extensions produce
     `SkippedUnsupportedExtension`; (d) a hasher exception produces `Failed`
     and does not abort the run; (e) cancellation token is honoured between
     files; (f) progress is reported with monotonically increasing counts.
   - `RefreshOrchestratorTests` — `StartAsync` while running throws
     `InvalidOperationException`; `Cancel` flips state to `Idle` after the
     scanner observes the token.
   - `SpdbConfigLibraryFolderRepositoryUnitTests` — round-trip add/list/remove
     against an in-memory `ISpdbConfigRepository`; counter monotonicity;
     remove-then-add yields a fresh id; folder path containing `_` is stored and
     retrieved byte-exact without matching sibling folder paths.

8. **Tests — live integration (`Xunit.SkippableFact`, rollback-only).**
   - `DapperMovieCatalogRepositoryTests`,
     `SpdbConfigLibraryFolderRepositoryTests` — INSERT/SELECT/DELETE inside
     a transaction that is always rolled back; rows use `vi_test_` prefixes
     where applicable; never `CREATE`/`DROP`/`ALTER`/`TRUNCATE`. **Note:**
     for `DapperMovieCatalogRepositoryTests`, the `vi_test_` discipline applies to
     the `hash` column (use a hash like `vi_test_<guid>` rather than a real
     32-hex string so a missed rollback is unambiguous).
   - `FfprobeRunnerTests` / `PathBasedVideoHasherIntegrationTests` — probe one
     file from [`test-folder/`](../../../../test-folder/) (e.g.
     `CA8C1E214046C8F725A1F46ADBAA61E3.raz`); assert JSON has expected top-level
     fields; assert hash determinism across two runs. `SkippableFact` when
     `ffprobe` cannot be located on `PATH` *and* `LibraryOptions.FfprobePath`
     is unset.
   - `LibraryScannerIntegrationTests` — register `test-folder/` as a folder,
     run `RefreshAsync`, assert that every supported-extension file in the
     folder appears in `movies_filenames` after the run, and that a second
     run inserts zero new rows. **All inserts and updates are wrapped in a
     transaction that is always rolled back**, so the production-copy data
     remains untouched.

9. **Documentation.**
   - Add [docs/projects/rebuild/milestones/m3-library-indexing.md](../../../projects/rebuild/milestones/m3-library-indexing.md)
     per the roadmap's milestone-doc template. Reference §1 of the movie
     management spec, the hashing spec, and the obfuscation spec rather than
     duplicating them. Include the manual smoke test below.
   - Update `README.md`:
     - "Build & Run" — **no** ffprobe prerequisite paragraph needed. M2.5 provisions
       FFmpeg/FFprobe automatically on first launch; update the README note at that
       section to reference `docs/projects/rebuild/milestones/m2-5-ffmpeg-provisioning.md`
       instead of directing users to install manually.
     - "Running the tests" — note that `FfprobeRunnerTests` and the live
       scanner test self-skip when neither `ffprobe` is on `PATH` nor
       `LibraryOptions:FfprobePath` is configured.
   - Record the WP-016/WP-017 *ledger-closure* recovery procedure as a
     repository-memory note so the M3 documentation WP cannot repeat the same
     mistake.

## Dependencies

- **Prior milestone:** M2 Database & Authentication (15 of 17 WPs complete; the
  outstanding WP-016 ledger-status closure and WP-017 milestone-doc work
  carried into M3 as **prerequisite tasks** — see *Out of Scope*).
- **External tooling:**
  - `ffprobe` (FFmpeg ≥ 4.x): provisioned automatically by M2.5 on first launch
    (see [`docs/projects/rebuild/milestones/m2-5-ffmpeg-provisioning.md`](../../../projects/rebuild/milestones/m2-5-ffmpeg-provisioning.md)).
    `FfprobeRunner` resolves the binary via a three-step chain: (1)
    `ExternalTools.Ffmpeg.FfprobePath` (written by `FfmpegProvisioner`); (2)
    `LibraryOptions.FfprobePath` (manual per-machine override); (3) `ffprobe[.exe]`
    resolved on `PATH`. Required for live hasher tests and manual smoke
    testing; not required for unit tests (which use `FakeFfprobeRunner`).
  - The MariaDB instance from M2 (`spdb_tests` schema seeded with a copy of
    production data, per
    [`tests/test-config.json`](../../../../tests/test-config.json)). Live tests
    self-skip when no config is present.
- **Test data:** [`test-folder/`](../../../../test-folder/) — 100+ obfuscated
  files (`.pkg`, `.raz`, `.dic`) already present in the workspace and
  already represented in the seeded test database. Used as-is by the live
  scanner integration test. **No file in `test-folder/` is renamed, deleted,
  or unobfuscated by any test.**
- **NuGet packages** — none new in Core, none new in Infrastructure (Dapper and
  MySqlConnector are already referenced; `System.Text.Json` and `Utf8JsonReader`
  are part of the .NET 10 BCL — no new `PackageReference` is required). No new
  App-side packages. No new test packages.
- **Specifications that must remain stable for M3:**
  [hashing-specification.md](../../../projects/rebuild/hashing-specification.md),
  [obfuscation-specification.md](../../../projects/rebuild/obfuscation-specification.md),
  and §1 of
  [movie-management-specification.md](../../../projects/rebuild/management-areas/movie-management-specification.md).

## Required Components

All paths workspace-relative. **(NEW)** items do not exist; **(MODIFIED)** items
already exist and gain content.

- `src/VideoIndexer.Core/Options/LibraryOptions.cs` (NEW)
- [`src/VideoIndexer.Core/Options/AppOptions.cs`](../../../../src/VideoIndexer.Core/Options/AppOptions.cs) (MODIFIED)
- `src/VideoIndexer.Core/Abstractions/ILibraryFolderRepository.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/IMovieCatalogRepository.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/IFfprobeRunner.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/IVideoHasher.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/IObfuscationMap.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/ILibraryScanner.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/IRefreshOrchestrator.cs` (NEW)
- `src/VideoIndexer.Core/Enums/RefreshOutcome.cs` (NEW)
- `src/VideoIndexer.Core/Enums/RefreshState.cs` (NEW)
- `src/VideoIndexer.Core/Models/LibraryFolder.cs` (NEW)
- `src/VideoIndexer.Core/Models/RefreshProgress.cs` (NEW)
- `src/VideoIndexer.Core/Models/RefreshSummary.cs` (NEW)
- `src/VideoIndexer.Infrastructure/Library/ExtensionObfuscationMap.cs` (NEW)
- `src/VideoIndexer.Infrastructure/Library/FfprobeRunner.cs` (NEW)
- `src/VideoIndexer.Infrastructure/Library/PathBasedVideoHasher.cs` (NEW)
- `src/VideoIndexer.Infrastructure/Library/SpdbConfigLibraryFolderRepository.cs` (NEW)
- `src/VideoIndexer.Infrastructure/Library/DapperMovieCatalogRepository.cs` (NEW)
- `src/VideoIndexer.Infrastructure/Library/LibraryScanner.cs` (NEW)
- `src/VideoIndexer.Infrastructure/Library/RefreshOrchestrator.cs` (NEW)
- `src/VideoIndexer.App/ViewModels/LibraryFoldersViewModel.cs` (NEW)
- `src/VideoIndexer.App/ViewModels/RefreshIndexViewModel.cs` (NEW)
- [`src/VideoIndexer.App/ViewModels/MainContentViewModel.cs`](../../../../src/VideoIndexer.App/ViewModels/MainContentViewModel.cs) (MODIFIED)
- `src/VideoIndexer.App/Views/LibraryFoldersView.axaml(.cs)` (NEW)
- `src/VideoIndexer.App/Views/RefreshIndexView.axaml(.cs)` (NEW)
- [`src/VideoIndexer.App/Views/MainContentView.axaml`](../../../../src/VideoIndexer.App/Views/MainContentView.axaml) (MODIFIED)
- `src/VideoIndexer.App/Services/IFolderPickerService.cs` (NEW; interface defined in `VideoIndexer.App` — depends on Avalonia's `IStorageProvider` and must **not** be placed in Core)
- `src/VideoIndexer.App/Services/AvaloniaFolderPickerService.cs` (NEW; implements `IFolderPickerService`)
- [`src/VideoIndexer.App/Program.cs`](../../../../src/VideoIndexer.App/Program.cs) (MODIFIED — DI registrations)
- [`src/VideoIndexer.App/Assets/appsettings.json`](../../../../src/VideoIndexer.App/Assets/appsettings.json) (MODIFIED — Library section)
- `tests/VideoIndexer.Tests/ExtensionObfuscationMapTests.cs` (NEW)
- `tests/VideoIndexer.Tests/PathBasedVideoHasherTests.cs` (NEW)
- `tests/VideoIndexer.Tests/LibraryScannerTests.cs` (NEW)
- `tests/VideoIndexer.Tests/RefreshOrchestratorTests.cs` (NEW)
- `tests/VideoIndexer.Tests/SpdbConfigLibraryFolderRepositoryUnitTests.cs` (NEW)
- `tests/VideoIndexer.Tests/Fixtures/FakeFfprobeRunner.cs` (NEW)
- `tests/VideoIndexer.Tests/Fixtures/TempVideoFolder.cs` (NEW)
- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperMovieCatalogRepositoryTests.cs` (NEW)
- `tests/VideoIndexer.Infrastructure.Tests/Library/SpdbConfigLibraryFolderRepositoryTests.cs` (NEW)
- `tests/VideoIndexer.Infrastructure.Tests/Library/FfprobeRunnerTests.cs` (NEW)
- `tests/VideoIndexer.Infrastructure.Tests/Library/PathBasedVideoHasherIntegrationTests.cs` (NEW)
- `tests/VideoIndexer.Infrastructure.Tests/Library/LibraryScannerIntegrationTests.cs` (NEW)
- `docs/projects/rebuild/milestones/m3-library-indexing.md` (NEW)
- [`README.md`](../../../../README.md) (MODIFIED — ffprobe prereq + tests section update)

No external services, no infrastructure provisioning, no CI changes.

## Assumptions

- The MariaDB `spdb_tests` schema seeded with a copy of production data is
  available throughout M3 development. Live tests are encouraged for daily dev,
  required before declaring an integration WP done, and self-skip in CI.
- The `test-folder/` files are valid video containers wearing obfuscated
  extensions (per the M2 + M3 user note that they "should have entries in the
  DB"). The scanner will short-circuit them without invoking ffprobe, and the
  `movies` rows for them already exist in the test DB.
- `ffprobe` is installable on every developer machine. Where it is not (e.g.
  CI runners), the live hasher tests skip rather than fail.
- The legacy `movies` schema is unchanged in M3 — no DB migrations, no
  `db_revision` bump. The empty-metadata insert (`label = ''`, `studio = ''`)
  is acceptable; M6 (the editor) is what populates those fields.
- The user is content to defer the spec's *Startup Selection* screen, the
  XML library snapshot file, and the obfuscation-toggle UI to later
  milestones (per the *Out of Scope* list below).
- The single-user desktop trust model from M2 still holds; folder paths are
  not validated against any allow-list.

## Constraints

- Must target **.NET 10** (the framework declared in `Directory.Build.props` and `global.json`); do not change the target framework.
- Must build cleanly under `TreatWarningsAsErrors=true` and
  `<WarningLevel>9999</WarningLevel>`.
- Must not break any M1 or M2 acceptance criterion: the
  `Connecting → LoggingOn → Ready` flow, the empty-content fall-through, the
  `Ctrl+T` debug theme cycle, and all existing test counts must all
  still pass.
- Must not introduce a Windows-only API dependency (cross-platform constraint
  from `rebuild.md`). `Process.Start` for `ffprobe`, `Path.*`, and Dapper are
  all cross-platform; folder-picker UX uses Avalonia's `IStorageProvider`.
- Must use Dapper for SQL access; no Entity Framework.
- Live integration tests must follow the M2 rollback-only discipline. **No
  test issues `CREATE`, `DROP`, `ALTER`, or `TRUNCATE`. No test calls
  `MySqlTransaction.Commit()`. No test renames, deletes, or unobfuscates any
  file in `test-folder/`.**
- Hashing must produce **identical** output to the spec across operating
  systems (uppercase hex, ASCII encoding, `|` separator, first-occurrence
  capture). This is non-negotiable: future obfuscation-toggle and
  thumbnail-generator features rely on hash stability.

## Out of Scope

- **Movie editor / movies grid / context menu** — M4–M6.
- **Obfuscation toggle UI** — M8 (the obfuscation *map* and the *skip-during-scan*
  behaviour ship in M3 because they are required by the spec, but the global
  *Turn On / Turn Off* worker that renames files lives in M8).
- **XML library snapshot file** (Open Library / Open Last) — deferred to M8 or
  later. M3 ships only the in-database folder list.
- **Startup Selection screen** (Open Library / Add Folders / Open Last) —
  deferred. After M3, a `Ready`-state user with zero registered folders
  simply sees an empty `MainContentView` with a *Library Folders…* button.
- **File deletion / "Delete on Disk"** — M4+.
- **Multi-folder progress aggregation refinements** (per-folder ETA, cancel-this-
  folder-only, throttled UI) — basic single-progress is enough for M3.
- **Thumbnails / cover images** — M9.
- **Schema migrations** — M3 does not alter any table or bump `db_revision`.
- **Movie metadata sync** (label cleaner, name parser) — M6.
- **Carry-over from M2:**
  - WP-016 ledger-status closure must be performed by a Project Manager
    *before* the M3 plan is decomposed into work packages.
  - WP-017 (M2 milestone doc + README updates) should be executed *before*
    M3 starts coding, so the M3 documentation WP can pattern-match cleanly.
  These two carry-overs are listed here so the Planner / PM does not lose
  them; they are **not** themselves M3 work.

## Acceptance Criteria

- [ ] `dotnet restore && dotnet build -c Release` from a fresh clone produces
      zero warnings and zero errors across all projects.
- [ ] All pre-M3 unit tests still pass (no regressions in the existing test suite).
- [ ] All new M3 unit tests pass under `dotnet test`. Live-MariaDB and
      `ffprobe`-dependent tests self-skip when their prerequisites are absent
      and pass when present.
- [ ] On launch, after the M2 `Connecting → LoggingOn → Ready` flow completes,
      `MainContentView` shows the *Library Folders…* and *Refresh Index*
      controls and an "Indexed: N" label reflecting the current `movies`
      row count.
- [ ] *Library Folders…* opens a list, an *Add Folder* picker registers the
      chosen absolute path, and the entry is persisted to the database
      (`spdb_config.library_folders` JSON array). Restarting the app shows
      the same folder list.
- [ ] *Delete Selected* removes the folder, removes every
      `movies_filenames` row whose path lives under that folder, and removes
      every `movies` row whose hash is no longer referenced.
- [ ] *Refresh Index* against the registered
      [`test-folder/`](../../../../test-folder/) completes without exception,
      reports per-file progress, and ends with a summary whose
      `SkippedAlreadyKnown + FilenameAdded` count equals the number of
      supported files in the folder (because every file is already in the
      seeded production-copy DB). A **second** *Refresh Index* with no
      filesystem changes inserts **zero** new rows.
- [ ] An obfuscated file (e.g. `*.pkg`, `*.raz`) is hashed via the
      filename-stem path, **not** by invoking ffprobe (verified by the
      `LibraryScannerTests` fake-runner expectation that no ffprobe call
      occurs for those extensions).
- [ ] Adding a brand-new file with a supported native extension to a registered
      folder, then running *Refresh Index*, inserts one new row in `movies`
      and one new row in `movies_filenames`. (Manual smoke test, documented in
      the milestone doc; the test never alters `test-folder/` and is performed
      against a temp folder created by the developer.)
- [ ] Cancelling an in-flight refresh stops processing within one file
      boundary; partial results are **not** rolled back (each file's inserts are
      autocommit — already-inserted rows are persisted when cancellation arrives).
- [ ] `MainContentView` is the only Ready-state surface; no other UI from
      M4–M10 leaks in.
- [ ] No file in [`test-folder/`](../../../../test-folder/) is renamed,
      deleted, or otherwise mutated by any test or by the manual smoke test.
- [ ] [docs/projects/rebuild/milestones/m3-library-indexing.md](../../../projects/rebuild/milestones/m3-library-indexing.md)
      exists and conforms to the milestone-doc template; README has the
      ffprobe prereq paragraph.

## Testing Strategy

- **Unit tests** (xUnit + FluentAssertions) — the bulk of M3 coverage; all use
  in-memory fakes (`FakeFfprobeRunner`, in-memory repos, `TempVideoFolder`),
  no DB, no ffprobe binary.
- **Live integration tests** (`Xunit.SkippableFact`, rollback-only) — exercise
  Dapper SQL against the seeded `spdb_tests` schema and the real `ffprobe`
  binary against `test-folder/`. Discipline (mandatory):
  - Every write inside a `MySqlTransaction` that is **always rolled back** in
    `Dispose` (success, failure, cancellation).
  - **Never** `CREATE`/`DROP`/`ALTER`/`TRUNCATE`.
  - **Never** `MySqlTransaction.Commit()`.
  - Test row keys (`hash`, folder paths, `config_name`s) namespaced with
    `vi_test_` so a forgotten rollback would leave an obviously-test row
    rather than overwriting production-copy data.
  - The fixture self-skips with an actionable message if `spdb_config` is
    missing or `ffprobe` cannot be located.
  - **Never** mutate any file in [`test-folder/`](../../../../test-folder/).
- **Manual smoke test** documented in `m3-library-indexing.md`:
  1. Drop a temp folder with one or two real video files (e.g. an `.mp4`)
     into a known location.
  2. Launch — complete the M2 flow.
  3. *Library Folders…* → *Add Folder* → pick the temp folder.
  4. *Refresh Index* — confirm progress reaches 100 %, the *Indexed* count
     increases by the number of files, and `SELECT * FROM movies WHERE
     hash = '<computed>'` returns one row.
  5. *Refresh Index* a second time — confirm zero new rows.
  6. *Library Folders…* → *Delete Selected* on the temp folder — confirm the
     `movies` and `movies_filenames` rows for those hashes disappear (and
     that no other rows are affected).
  7. Repeat steps 3–4 against [`test-folder/`](../../../../test-folder/) and
     confirm zero inserts (every file is already in the seeded DB).
- **No Avalonia Headless tests** in M3 — still tracked as a separate future WP.

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Hash drift across OSes (line endings in JSON, locale-sensitive number formatting).** | The hashing pipeline parses ffprobe's UTF-8 JSON via `Utf8JsonReader` and captures each scalar value as its **raw JSON token text** (`Utf8JsonReader.ValueSpan` decoded as ASCII) — numeric tokens are never re-rendered, so `1.0` stays `1.0`. A determinism unit test asserts that a JSON input containing `"x": 1.0` produces the dictionary entry `"1.0"`, not `"1"`; a live integration test hashes the same physical file twice and compares. |
| **`ffprobe` not installed on a developer machine.** | M2.5 provisions the binary automatically on first launch via `FfmpegProvisioner`. The runner reads `ExternalTools.Ffmpeg.FfprobePath` and falls back to `ffprobe[.exe]` on `PATH` when null; live tests `Skip` with a clear message. No manual installation step is required. |
| **Concurrent refresh runs corrupt counts / cause duplicate inserts.** | `IRefreshOrchestrator` is a single-flight guard; `StartAsync` while a run is in flight throws `InvalidOperationException`; the *Refresh Index* button is `IsRunning`-gated. |
| **Live integration tests accidentally mutate the production-copy DB.** | M2-style rollback discipline, `vi_test_` namespaced keys, fixture-level self-skip on missing config, code-review checklist item that *no test calls `Commit()` and no test issues a DDL statement*. |
| **A rogue test renames or unobfuscates a `test-folder/` file.** | The obfuscator-toggle code path is **out of scope** for M3 and is not exercised by any M3 test. The scanner only **reads** files. A code-review checklist item explicitly forbids any test from calling `File.Move`, `File.Delete`, or any obfuscator method against a path under `test-folder/`. |
| **Folder-removal cascade leaves orphan `movies` rows or, worse, deletes shared rows.** | The cascade explicitly runs in two stages inside one transaction: delete `movies_filenames` rows for the folder; then delete `movies` rows whose hash is no longer referenced by *any* `movies_filenames` row. A targeted unit test (in-memory repos) and a live integration test (rollback-only, namespaced rows) cover both stages. |
| **Long ffprobe call hangs the UI.** | The runner enforces a 30 s per-file timeout; the orchestrator runs the scanner on a background `Task.Run`; Avalonia bindings dispatch back to the UI thread via `Dispatcher.UIThread.Post`. |
| **Plan scope creep into M4 surfaces (movies grid, editor stubs).** | The *Out of Scope* list is explicit; the `MainContentView` mock-up is intentionally one toolbar row. PR review enforces. |
| **WP-016 ledger-closure pattern repeats on the M3 documentation WP.** | The PM agent must call `ledger_update_work_package_status(<wp>, COMPLETE)` as the final step of every Documentation pipeline; recorded as a `/memories/repo/` note before M3 work begins. |
| **Absolute paths exceeding 255 characters are silently truncated or rejected by MariaDB** (the `movies_filenames.filename` column is `varchar(255)`; on Windows with long-path support enabled, or on Linux/macOS, real paths can exceed this). | In `LibraryScanner`, before each `movies_filenames` insert, check `Encoding.UTF8.GetByteCount(path) > 250`; if so, log at `Warning` and count the file as `RefreshOutcome.Failed`. This measures the actual storage cost under `utf8mb4` rather than UTF-16 code units. Document the 255-byte path limit in the milestone doc as a known M3 constraint. |
