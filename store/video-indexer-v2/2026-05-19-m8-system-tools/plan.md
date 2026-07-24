# Plan — M8 System Tools


## Plan Audit Cycles
- Audits: 2 — Plan Auditor v1.3.0
- Architectural Reviews: 1 — Plan Architect Reviewer v1.4.0


## Summary

M8 delivers the System Tools milestone: a full six-tab Preferences settings page
(General, Database, Thumbnails, Appearance, Logging, Obfuscation), a Database Backup
workflow accessible from a new File menu bar, and a file-rename Obfuscation toggle with
background progress reporting. It also ships schema migration **m041** that adds the
`movies.original_filename` column required to safely round-trip file renames, bumping
`DatabaseBootstrapper.ExpectedRevision` from 40 → 41. The "settings" primary-navigation
destination, currently backed by `StubViewModel`/`StubView`, is replaced by real
`SettingsViewModel`/`SettingsView` implementations. Everything else in the current system
(shell auth, library scan, movies list, editor, tagger) is unaffected.


## Architectural Context

| Concern | Existing asset |
|---------|----------------|
| Settings persistence (machine-local) | `ISettingsService` / `JsonSettingsService` (`src/VideoIndexer.Infrastructure/Settings/JsonSettingsService.cs`); atomic write via `.tmp` + `File.Move`; fires `SettingsChanged` event |
| `AppOptions` record tree | `src/VideoIndexer.Core/Options/AppOptions.cs`; child records `LoggingOptions`, `AppearanceOptions`, `ExternalToolsOptions`, `WindowOptions`, `DatabaseOptions`, `LibraryOptions`, `MoviesListOptions`, `LabelCleanerOptions`, `TaggingOptions` |
| Theme | `IThemeService` / `ThemeService` (`src/VideoIndexer.Infrastructure/Theme/ThemeService.cs`); persists to `AppOptions.Appearance.Theme` and fires `ThemeChanged` |
| Obfuscation map | `IObfuscationMap` / `ExtensionObfuscationMap` (`src/VideoIndexer.Infrastructure/Library/ExtensionObfuscationMap.cs`); hardcoded 10-entry insertion-ordered map |
| Key-value DB config | `ISpdbConfigRepository` (`GetAsync` / `SetAsync` on `spdb_config` table); used by filter slots and library folders |
| Movie filenames & paths | `IMovieRepository` / `DapperMovieRepository`; `GetOriginalFilenameAsync` returns oldest `movies_filenames` row — will be updated in M8 |
| Folder picker | `IFolderPickerService` / `AvaloniaFolderPickerService` (already registered) |
| Connection store | `IDatabaseConnectionStore` / `JsonDatabaseConnectionStore`; `GetActiveAsync()` returns active `DatabaseConnectionOptions` |
| Settings nav stub | `StubViewModel` / `StubView` currently at the "settings" primary destination; `MainContentViewModel` constructs both stubs inline with `new StubViewModel()` |
| Primary nav | `IPrimaryNavigationService`; `MainContentViewModel` constructor registers 5 destinations and calls `NavigateTo("movies")` |
| Dialog service pattern | Modal windows via `IXxxService` / `AvaloniaXxxService` (e.g., `ILabelCleanerService`, `ITagEditorService`); `try/finally` unsubscription; named `CloseRequested` event pattern |


## Approach / Architecture

### 1 — Preferences as a primary-nav page

The "settings" rail destination (currently `StubViewModel`) is replaced by
`SettingsViewModel` + `SettingsView`. A six-tab `TabControl` hosts one tab per settings
group. `MainContentViewModel` is updated to accept `SettingsViewModel` as a new
constructor parameter instead of constructing `new StubViewModel()` for the settings
destination.

`SettingsView.axaml.cs` calls `SettingsViewModel.LoadAsync(token)` from its `OnLoaded`
handler (same pattern as `MoviesListView.axaml.cs`). `LoadAsync` reads the current
`AppOptions` snapshot plus the database-side obfuscation state.

### 2 — Obfuscation tab sub-ViewModel

The Obfuscation tab is stateful (background worker, progress bar, error list). It gets
its own `ObfuscationSettingsViewModel` nested inside `SettingsViewModel` as a read-only
property. This VM exposes `ToggleCommand` (async relay), `IsEnabled`, `Progress`, and
`Errors`, isolating the worker lifecycle from the rest of the settings form.

### 3 — Settings save strategy

The five "local" tabs (General, Database, Thumbnails, Appearance, Logging) share a
single **Save** command on `SettingsViewModel` that produces an updated `AppOptions` via
`with { }` and calls `ISettingsService.SaveAsync`. A **Cancel** command reverts
in-memory fields to `ISettingsService.Current`. The Database tab is read-only (shows
active connection details); no mutation occurs there.

The Obfuscation tab saves its enabled/disabled state to `spdb_config` via
`IObfuscationService.ToggleAsync`; it does not participate in the shared
Save/Cancel cycle.

### 4 — Database Backup as a menu-triggered modal

A new Avalonia `Menu` control is added at the top of `MainContentView.axaml`'s
`DockPanel` (above the `NavigationView`). The File menu has a single "Database Backup…"
item bound to `MainContentViewModel.BackupDatabaseCommand`. That command calls
`IDatabaseBackupDialogService.ShowAsync()`, which opens a `DatabaseBackupView` modal
window. `DatabaseBackupViewModel` owns the folder picker, backup execution, and
result display.

### 5 — Schema migration m041

Adds `movies.original_filename VARCHAR(255) NULL`. The obfuscation worker writes the
pre-obfuscation filename here before renaming the file; the unobfuscation worker reads it
to restore the original name and clears the column.
`DapperMovieRepository.GetOriginalFilenameAsync` is updated to prefer
`movies.original_filename` when non-null, falling back to the oldest
`movies_filenames.filename` row (existing behavior for non-obfuscated movies).

### 6 — New services and interfaces

| Interface (Core) | Implementation (Infrastructure) | Purpose |
|---|---|---|
| `IObfuscationService` | `ObfuscationService` | Read/write obfuscation enabled state; drive bulk file-rename worker |
| `IDatabaseBackupService` | `MysqlDumpBackupService` | Invoke `mysqldump`, stream output, write `.sql` backup file |
| `IDatabaseBackupDialogService` (App) | `AvaloniaDatabaseBackupDialogService` | Open `DatabaseBackupView` modal window |

`IMovieRepository` gains one new method:
`Task<IReadOnlyList<MovieForObfuscation>> GetAllMoviesForObfuscationAsync(CancellationToken)`

Three new lightweight Core models: `MovieForObfuscation`, `ObfuscationProgress`,
`DatabaseBackupResult`.


## Rationale

- **Preferences as a page, not a modal dialog** — the rail already has a "settings"
  destination slot. Using it avoids a second navigation pattern for settings, keeps the
  UI consistent with the other destinations, and requires only minimal plumbing changes to
  `MainContentViewModel`.
- **Dedicated `ObfuscationSettingsViewModel`** — the obfuscation tab carries mutable
  async worker state (progress, errors, IsBusy) that is out of place inside a flat
  settings form VM. Separation mirrors the M6/M7 pattern of giving complex sub-features
  their own ViewModels.
- **Single Save for the five "local" tabs** — all five tabs read and write the same
  `AppOptions` record. A single command avoids per-tab save buttons and makes the
  cancel/revert semantics clear.
- **Database tab read-only** — connection editing is already handled by the startup
  `DatabaseConnectorView` / `ConnectionEditorViewModel`. Duplicating that form in
  Preferences would create two code paths for the same operation.
- **Schema migration for `movies.original_filename`** — the obfuscation spec §3.2
  explicitly requires preserving the original filename "in a separate column." The
  existing `GetOriginalFilenameAsync` workaround (oldest `movies_filenames` row) is
  fragile under multiple-file-per-movie scenarios.
- **Menu bar in `MainContentView`, not `MainWindow`** — the File menu is only meaningful
  in `ShellState.Ready`. Embedding it in `MainContentView` avoids enable/disable
  complexity driven by shell state from `MainWindow`.


## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|----------|--------------|-------------------------|-------------------|
| Preferences UI placement | Primary-nav page (`SettingsView`) replacing the `StubViewModel` | Modal dialog opened from a menu item | A page uses the existing navigation slot; a modal would add a second dialog-service and no persistent nav context |
| Obfuscation tab VM | Dedicated `ObfuscationSettingsViewModel` | All tabs inline in one flat `SettingsViewModel` | The background worker + progress state warrants separation; flat VM would bloat `SettingsViewModel` with worker lifecycle concerns |
| Database tab behaviour | Read-only display of active connection | Inline edit form duplicating `ConnectionEditorViewModel` | Avoids code duplication; connection editing already exists at startup; the tab fulfils the spec requirement of "showing" the DB settings |
| Original filename storage | New `movies.original_filename` column (migration m041) | Keep using oldest `movies_filenames` row | Spec §3.2 explicitly requires a separate column; the oldest-row proxy fails when multiple obfuscation cycles occur or rows are deleted |
| Database Backup access | File menu bar in `MainContentView` | Button inside Preferences General tab | Spec §4.1 says "Main Menu → File → Database Backup"; a menu bar matches this expectation and keeps backup out of Preferences |
| Obfuscation state persistence | `spdb_config["obfuscation_enabled"]` | New `AppOptions` field in `appsettings.json` | The spec §2.2 designates cross-machine settings for `spdb_config`; the obfuscation state must survive reinstallation and affect all users of the same DB |
| Extension map display in `ObfuscationSettingsViewModel` | Build `IReadOnlyDictionary` from `IObfuscationMap.RealExtensions` + `TryGetObfuscatedExtension` in `LoadAsync` (Option B) | Add `NativeToObfuscated` property to `IObfuscationMap` interface (Option A) | Option B requires no interface change and no additional Modified Files entry. Option A exposes a richer surface for future display consumers. Option B is the narrower, lower-friction choice. |
| `ObfuscationSettingsViewModel` DI lifetime | `AddSingleton` | `AddTransient` | `SettingsViewModel` (singleton) holds `ObfuscationSettingsViewModel` as a constructor parameter; a transient captured inside a singleton is a classic captive-dependency anti-pattern. `AddSingleton` is the honest contract. |
| `DatabaseBackupView` DI registration | Not registered — constructed directly in `AvaloniaDatabaseBackupDialogService` via `new DatabaseBackupView()` | `AddTransient<DatabaseBackupView>` | No DI consumer exists; registering it risks confusion about whether it is ViewLocator-managed. Follows the `AvaloniaTagEditorService` / `new TagEditorView()` precedent. |


## Pattern Alignment

| Pattern | Status |
|---------|--------|
| `LoadAsync(CancellationToken)` called from `OnLoaded` in view code-behind | Followed — `SettingsView.axaml.cs` and `DatabaseBackupView.axaml.cs` follow `MoviesListView.axaml.cs` |
| Dialog service `IXxxService` / `AvaloniaXxxService` with modal window, `CloseRequested` event, `try/finally` unsubscription | Followed — `IDatabaseBackupDialogService` / `AvaloniaDatabaseBackupDialogService` |
| `AppOptions` immutability: `with { }` + `ISettingsService.SaveAsync` | Followed — `SettingsViewModel.SaveCommand` produces a new record snapshot |
| `[ObservableProperty]` and `[RelayCommand]` from CommunityToolkit.Mvvm | Followed throughout all new ViewModels |
| Compiled Avalonia bindings (`x:DataType`) | Followed in all new `.axaml` files |
| No `Version=` on `<PackageReference>` | Followed — no new NuGet packages needed |
| `TreatWarningsAsErrors` — all warnings must be resolved | Followed — new code must compile clean |
| Cooperative cancellation in background workers | Followed — `ObfuscationService.ToggleAsync` honours the `CancellationToken` and returns normally on cancellation (does not throw `OperationCanceledException`) |
| `.ConfigureAwait(false)` in Infrastructure and Core async paths | Followed in `ObfuscationService` and `MysqlDumpBackupService` |
| `StubViewModel` / `StubView` must not gain logic | Followed — the "filters" destination stub is left unchanged; only the "settings" stub is replaced |
| `ViewLocator` naming convention: `FooViewModel` → `FooView` | Followed — `SettingsViewModel` → `SettingsView`, `DatabaseBackupViewModel` → `DatabaseBackupView` |
| Views use parameterless constructors in DI | Followed for `SettingsView`; `DatabaseBackupView` is **not** registered in DI — it is constructed directly by `AvaloniaDatabaseBackupDialogService` (same as `TagEditorView` in `AvaloniaTagEditorService`) |
| `PageHeaderView` dynamic values via `OnDataContextChanged` | Followed in `SettingsView.axaml.cs` for the title |


## Detailed Steps

### Phase 1 — AppOptions extensions & schema migration

1. **`ThumbnailsOptions` record** (new file `src/VideoIndexer.Core/Options/ThumbnailsOptions.cs`):
   ```csharp
   public sealed record ThumbnailsOptions
   {
       public int ThumbnailSize { get; init; } = 150;
       public bool KeepOriginalSize { get; init; }
   }
   ```

2. **`AppOptions`** — add `public ThumbnailsOptions Thumbnails { get; init; } = new();`

3. **`ExternalToolsOptions`** — add `public bool UseVlc { get; init; }` (default `false`).

4. **`Assets/appsettings.json`** (the embedded defaults) — add the new sections:
   ```json
   "Thumbnails": { "ThumbnailSize": 150, "KeepOriginalSize": false },
   ```
   and in `ExternalTools`: `"UseVlc": false`.

5. **Migration `m040` → `m041`**:
   - Create `src/VideoIndexer.Infrastructure/Database/migrations/m041_add_original_filename.sql`:
     ```sql
     ALTER TABLE movies ADD COLUMN original_filename VARCHAR(255) NULL;
     UPDATE spdb_config SET config_value = '41' WHERE config_name = 'db_revision';
     ```
   - `DatabaseBootstrapper.ExpectedRevision` 40 → 41; add `m041` to the embedded
     migration resource list and the auto-migration range check.
   - In `tests/VideoIndexer.Infrastructure.Tests/Database/DatabaseBootstrapperTests.cs`,
     update the sentinel literal in `ExpectedRevision_MatchesCurrentSchemaRevision` from
     `40` to `41`. This literal is an intentional migration tripwire — do **not** replace
     it with `DatabaseBootstrapper.ExpectedRevision`.

6. **`DapperMovieRepository.GetOriginalFilenameAsync`** — update the SQL to prefer
   `movies.original_filename` when non-null:
   ```sql
   SELECT COALESCE(m.original_filename, mf.filename)
   FROM movies m
   LEFT JOIN movies_filenames mf ON mf.hash = m.hash
   WHERE m.movie_id = @MovieId
   ORDER BY mf.created_at ASC
   LIMIT 1
   ```

---

### Phase 2 — Core models and interfaces

7. **`src/VideoIndexer.Core/Models/MovieForObfuscation.cs`** (new):
   ```csharp
   public sealed record MovieForObfuscation(
       long    MovieId,
       string  Hash,
       string  CurrentFilename,      // from movies_filenames.filename (full path)
       string? OriginalFilename);    // from movies.original_filename (basename+ext, or null)
   ```

8. **`src/VideoIndexer.Core/Models/ObfuscationProgress.cs`** (new):
   ```csharp
   public sealed record ObfuscationProgress(
       int    Processed,
       int    Total,
       string CurrentFile);
   ```

9. **`src/VideoIndexer.Core/Models/DatabaseBackupResult.cs`** (new):
   ```csharp
   public sealed record DatabaseBackupResult(
       bool    Success,
       string? OutputFilePath,
       string? ErrorMessage);
   ```

10. **`IMovieRepository`** — add:
    ```csharp
    Task<IReadOnlyList<MovieForObfuscation>> GetAllMoviesForObfuscationAsync(
        CancellationToken cancellationToken = default);
    ```

11. **`IObfuscationService`** (new `src/VideoIndexer.Core/Abstractions/IObfuscationService.cs`):
    ```csharp
    public interface IObfuscationService
    {
        Task<bool> IsEnabledAsync(CancellationToken cancellationToken = default);
        // Renames all indexed files and persists the enabled/disabled state.
        // Returns normally on cancellation — does NOT throw OperationCanceledException.
        Task ToggleAsync(
            bool enable,
            IProgress<ObfuscationProgress>? progress = null,
            CancellationToken cancellationToken = default);
    }
    ```

12. **`IDatabaseBackupService`** (new `src/VideoIndexer.Core/Abstractions/IDatabaseBackupService.cs`):
    ```csharp
    public interface IDatabaseBackupService
    {
        Task<DatabaseBackupResult> BackupAsync(
            string destinationFolder,
            CancellationToken cancellationToken = default);
    }
    ```

---

### Phase 3 — Infrastructure implementations

13. **`DapperMovieRepository`** — implement `GetAllMoviesForObfuscationAsync`:
    ```sql
    SELECT m.movie_id AS MovieId,
           m.hash     AS Hash,
           mf.filename AS CurrentFilename,
           m.original_filename AS OriginalFilename
    FROM movies m
    INNER JOIN movies_filenames mf ON mf.hash = m.hash
    ORDER BY m.movie_id, mf.filename
    ```
    Returns one `MovieForObfuscation` per `(movie_id, filename)` pair. The `ORDER BY`
    clause ensures deterministic row ordering for progress reporting and integration test
    assertions.

14. **`ObfuscationService`** (new `src/VideoIndexer.Infrastructure/Library/ObfuscationService.cs`):
    - Constructor: `IMovieRepository, IObfuscationMap, ISpdbConfigRepository, IDbConnectionFactory, ILogger<ObfuscationService>`
    - `IsEnabledAsync`: reads `spdb_config["obfuscation_enabled"]`; returns `true` if value
      is `"1"`.
    - `ToggleAsync(enable, progress, ct)`:
      1. Load all movies via `GetAllMoviesForObfuscationAsync`.
      2. For each `MovieForObfuscation` row, with `ct.IsCancellationRequested` checked
         after each file:
         - **Enable path**: skip if `CurrentFilename` stem is already the movie hash
           (already obfuscated). Otherwise derive the obfuscated path, `File.Move`, then
           `UPDATE movies_filenames SET filename = @New WHERE filename = @Old` and
           `UPDATE movies SET original_filename = @BaseName WHERE movie_id = @Id`.
         - **Disable path**: skip if `OriginalFilename` is null (not obfuscated). Otherwise
           derive the restore path from `OriginalFilename`, `File.Move`, then
           `UPDATE movies_filenames SET filename = @Restored WHERE filename = @Old` and
           `UPDATE movies SET original_filename = NULL WHERE movie_id = @Id`.
         - File errors (missing file, permissions): collected in a local list; operation
           continues to next file.
      3. On normal completion (or cancellation): write `spdb_config["obfuscation_enabled"]
         = enable ? "1" : "0"`.
      4. Report `ObfuscationProgress` after each file via `progress?.Report(...)`.

15. **`MysqlDumpBackupService`** (new `src/VideoIndexer.Infrastructure/Database/MysqlDumpBackupService.cs`):
    - Constructor: `ISettingsService, IDatabaseConnectionStore, ILogger<MysqlDumpBackupService>`
    - Extract argument-building into an `internal static ProcessStartInfo BuildStartInfo(
      string mysqldumpPath, DatabaseConnectionOptions conn, string outputFilePath)` method.
      This method is `internal` so `VideoIndexer.Tests` can call it directly to verify
      injection safety without spawning a process. Add
      `[assembly: InternalsVisibleTo("VideoIndexer.Tests")]` to
      `src/VideoIndexer.Infrastructure/Properties/AssemblyInfo.cs` (or the
      `.csproj` `<InternalsVisibleTo>` item if the file does not exist).
    - `BackupAsync(destinationFolder, ct)`:
      1. Check `ISettingsService.Current.ExternalTools.MySqlBinaryFolder` is configured;
         return `DatabaseBackupResult(false, null, "MySQL binary folder is not configured…")`
         if absent.
      2. Build `mysqldump` path from folder + `"mysqldump"` / `"mysqldump.exe"`.
      3. Obtain active connection via `IDatabaseConnectionStore.GetActiveAsync()`.
      4. Call `BuildStartInfo(mysqldumpPath, conn, outputPath)` — it populates
         `ProcessStartInfo.ArgumentList` with `--single-transaction`,
         `--default-character-set=utf8mb4`, `-h`, `{host}`, `-P`, `{port}`,
         `-u`, `{user}`, `-p{password}`, `{database}` as discrete entries.
         **Never use `ProcessStartInfo.Arguments` string concatenation** — it bypasses
         shell-quoting and allows injection from connection field values.
      5. Redirect stdout to a timestamped file
         `Path.Combine(destinationFolder, $"backup-{DateTime.Now:yyyy-MM-dd}.sql")`.
      6. On exit code 0: return `DatabaseBackupResult(true, outputPath, null)`.
      7. On non-zero exit: return `DatabaseBackupResult(false, null, captured stderr)`.

---

### Phase 4 — App layer: new dialog service

16. **`IDatabaseBackupDialogService`** (new `src/VideoIndexer.App/Services/IDatabaseBackupDialogService.cs`):
    ```csharp
    public interface IDatabaseBackupDialogService
    {
        Task ShowAsync(CancellationToken cancellationToken = default);
    }
    ```

17. **`DatabaseBackupViewModel`** (new `src/VideoIndexer.App/ViewModels/DatabaseBackupViewModel.cs`):
    - Properties: `DestinationFolder`, `IsBusy`, `ResultMessage`, `IsSuccess`, `HasResult`.
    - Commands: `BrowseFolderCommand` (calls `IFolderPickerService`), `BackupCommand`
      (async, calls `IDatabaseBackupService.BackupAsync`), `CloseCommand` (fires
      `CloseRequested`).
    - `CloseRequested` event.

18. **`DatabaseBackupView.axaml / .axaml.cs`** (new):
    - `DockPanel` with a bottom toolbar (`Backup`, `Close`) and a simple form:
      folder-picker row, result area (green/red status text).
    - NOT ViewLocator-registered (shown by `AvaloniaDatabaseBackupDialogService`).
    - `axaml.cs` does **not** subscribe `CloseRequested` — that subscription belongs
      exclusively in `AvaloniaDatabaseBackupDialogService.ShowAsync` (see Step 19).

19. **`AvaloniaDatabaseBackupDialogService`** (new `src/VideoIndexer.App/Services/AvaloniaDatabaseBackupDialogService.cs`):
    - Pattern identical to `AvaloniaTagEditorService`: constructor accepts
      `IDatabaseBackupService`, `IFolderPickerService`, and `Func<Window?> ownerFactory`;
      `ShowAsync` constructs `DatabaseBackupViewModel` directly (not via DI), opens a
      modal `Window` with `DatabaseBackupView`, and uses `try/finally` unsubscription of
      `CloseRequested`.

---

### Phase 5 — App layer: Preferences ViewModel and View

20. **`ObfuscationSettingsViewModel`** (new `src/VideoIndexer.App/ViewModels/ObfuscationSettingsViewModel.cs`):
    - Constructor: `IObfuscationService, IObfuscationMap`.
    - Properties: `IsEnabled` (bool), `IsBusy` (bool), `Progress` (string showing
      "N / Total" or "Idle"), `Errors` (ObservableCollection\<string\>),
      `ExtensionMap` (IReadOnlyDictionary\<string, string\>).
    - `LoadAsync(ct)`: reads `IObfuscationService.IsEnabledAsync()`; builds `ExtensionMap`
      from `IObfuscationMap` interface members only — no access to the concrete class:
      ```csharp
      ExtensionMap = _obfuscationMap.RealExtensions
          .ToDictionary(
              e => e,
              e => { _obfuscationMap.TryGetObfuscatedExtension(e, out var o); return o ?? e; });
      ```
    - `ToggleCommand` (async relay): calls `IObfuscationService.ToggleAsync(!IsEnabled, …)`;
      sets `IsBusy`; handles cancellation by not throwing.

21. **`SettingsViewModel`** (new `src/VideoIndexer.App/ViewModels/SettingsViewModel.cs`):
    - Constructor: `ISettingsService, IThemeService, IDatabaseConnectionStore,
      ObfuscationSettingsViewModel, IFolderPickerService`.
    - Tab-local observable properties (populated from `ISettingsService.Current` in
      `LoadAsync`):
      - **General**: `MySqlBinaryFolder`, `ExternalXmlEditorPath`, `VlcExecutablePath`,
        `UseVlc`.
      - **Database** (read-only): `ActiveConnectionHost`, `ActiveConnectionDatabase`,
        `ActiveConnectionUsername` — loaded from `IDatabaseConnectionStore.GetActiveAsync()`.
      - **Thumbnails**: `ThumbnailSize` (int), `KeepOriginalSize` (bool).
      - **Appearance**: `SelectedTheme` (ThemeMode), `SelectedLanguage` (string).
      - **Logging**: `Verbosity` (int, 1–4).
    - **`ObfuscationVm`** — read-only property exposing the nested
      `ObfuscationSettingsViewModel`.
    - `LoadAsync(CancellationToken)`: populate all above properties; call
      `ObfuscationVm.LoadAsync(ct)`.
    - **`SaveCommand`** (async relay): produce updated `AppOptions` via `with { }` and call
      `ISettingsService.SaveAsync`; also call `IThemeService.SetAsync` if theme changed.
    - **`CancelCommand`**: re-run the field-population logic from `ISettingsService.Current`
      to revert in-memory changes.
    - **`BrowseMySqlFolderCommand`**: calls `IFolderPickerService.PickFolderAsync` to
      update `MySqlBinaryFolder` (a folder path — folder picker is correct here).
    - **`BrowseVlcPathCommand`** and **`BrowseXmlEditorCommand`** are **not implemented
      in M8**. `VlcExecutablePath` and `ExternalXmlEditorPath` are editable directly via
      their `TextBox` fields. The Browse button affordance for these two fields is deferred
      to a future milestone that will introduce `IFilePickerService`. Remove the Browse
      buttons for VLC and XML editor from `SettingsView.axaml`; leave only the `TextBox`.

22. **`SettingsView.axaml / .axaml.cs`** (new):
    - Top-level layout: `PageHeaderView` (title "Settings") + `TabControl` with six tabs.
    - **General tab**: MySQL folder row with `TextBox` + "Browse…" button (calls
      `BrowseMySqlFolderCommand`); XML editor and VLC path rows with `TextBox` only
      (no Browse button in M8 — see Step 21); `CheckBox` for `UseVlc`.
    - **Database tab**: four read-only `TextBlock` rows (Host, Database, Username, Port);
      info text "Changes take effect after application restart."
    - **Thumbnails tab**: `NumericUpDown` (bound to `ThumbnailSize`), `CheckBox`
      (`KeepOriginalSize`). `NumericUpDown` uses `IntDecimalConverter.Instance` (existing
      singleton converter) for the compiled binding.
    - **Appearance tab**: `ComboBox` for theme (Light / Dark / System), `ComboBox` for
      language (English / Deutsch / Français).
    - **Logging tab**: `ComboBox` or `Slider` for verbosity (1–4).
    - **Obfuscation tab**: status label, toggle button ("Enable" / "Disable"), progress
      label, `ListBox` for errors, read-only `DataGrid` for the extension map
      (Original / Obfuscated columns).
    - Bottom toolbar: "Save" and "Cancel" buttons bound to `SaveCommand` /
      `CancelCommand` (these affect all non-obfuscation tabs; the Obfuscation toggle is
      independent).
    - `axaml.cs`: `OnLoaded` creates a `CancellationTokenSource`, wires `vm.LoadAsync`;
      `OnUnloaded` cancels/disposes it. `OnDataContextChanged` wires `PageHeaderView.Title`.

---

### Phase 6 — Menu bar and navigation wiring

23. **`MainContentView.axaml`** — wrap the existing root content in a `DockPanel`; add a
    `Menu` docked to the top:
    ```xml
    <Menu DockPanel.Dock="Top">
      <MenuItem Header="_File">
        <MenuItem Header="Database _Backup…"
                  Command="{Binding BackupDatabaseCommand}" />
      </MenuItem>
    </Menu>
    ```
    The existing `NavigationView` occupies the remaining `DockPanel` area.

24. **`MainContentViewModel`** — changes:
    - Add constructor parameter `SettingsViewModel settingsVm`.
    - In `primaryNavService.Register(…)` replace `new StubViewModel()` for the "settings"
      destination with `settingsVm`.
    - Add `IDatabaseBackupDialogService? _backupDialogService` field and matching
      constructor parameter.
    - Add `[RelayCommand] private async Task BackupDatabase(CancellationToken ct)` which
      calls `_backupDialogService?.ShowAsync(ct)`.

25. **`Program.cs`** (DI registrations — insert a new section **before** the existing
    `// 9. Start Avalonia` block, e.g. `// 8.5 — M8 System Tools services`):
    - `AddSingleton<IObfuscationService, ObfuscationService>()`
    - `AddSingleton<IDatabaseBackupService, MysqlDumpBackupService>()`
    - `AddSingleton<IDatabaseBackupDialogService>(sp => new AvaloniaDatabaseBackupDialogService(sp.GetRequiredService<IDatabaseBackupService>(), sp.GetRequiredService<IFolderPickerService>(), ownerFactory))` — captures the existing `ownerFactory` closure defined in section 5.75, matching the registration pattern used by all other dialog services.
    - `AddSingleton<ObfuscationSettingsViewModel>()` *(singleton — matches the lifetime of
      `SettingsViewModel` which holds it; using `AddTransient` here would create a captive
      dependency)*
    - `AddSingleton<SettingsViewModel>()`
    - `AddTransient<SettingsView>(_ => new SettingsView())`
    - **Do NOT register `DatabaseBackupView` in DI.** `AvaloniaDatabaseBackupDialogService`
      constructs the view directly via `new DatabaseBackupView()`, identical to how
      `AvaloniaTagEditorService` uses `new TagEditorView`. Registering it would imply
      ViewLocator management and create confusion about ownership.
    - Replace the plain `AddTransient<MainContentViewModel>()` with an explicit factory
      lambda that injects `SettingsViewModel` and `IDatabaseBackupDialogService` alongside
      the existing parameters.

---

### Phase 7 — Tests

26. **`FakeObfuscationService`** (new `tests/VideoIndexer.App.Tests/TestHelpers/FakeObfuscationService.cs`):
    configurable `IsEnabled` result, captures `ToggleAsync` call arguments.

27. **`FakeDatabaseBackupService`** (new `tests/VideoIndexer.App.Tests/TestHelpers/FakeDatabaseBackupService.cs`):
    configurable `BackupResult`, captures `destinationFolder`.

28. **`SettingsViewModelTests`** (new `tests/VideoIndexer.App.Tests/SettingsViewModelTests.cs`):
    - `LoadAsync_PopulatesFieldsFromCurrentOptions` — verifies all tab fields reflect
      `ISettingsService.Current` after `LoadAsync`.
    - `SaveCommand_ProducesUpdatedAppOptions` — mutates a field, asserts
      `ISettingsService.SaveAsync` receives an `AppOptions` with the expected change.
    - `CancelCommand_RevertsFields` — mutates a field, calls `CancelCommand`, verifies
      field reverts.
    - `SaveCommand_CallsThemeService_WhenThemeChanged` — verifies `IThemeService.SetAsync`
      is called when `SelectedTheme` is changed before save.

29. **`ObfuscationSettingsViewModelTests`** (new `tests/VideoIndexer.App.Tests/ObfuscationSettingsViewModelTests.cs`):
    - `LoadAsync_SetsIsEnabled_FromService` — `IsEnabledAsync` returns true → `IsEnabled`
      is true.
    - `ToggleCommand_CallsToggleAsync_WithInvertedState` — `IsEnabled = true` → calls
      `ToggleAsync(false, …)`.
    - `ToggleCommand_SetsBusy_DuringExecution` — `IsBusy` is true while
      `ToggleAsync` runs.

30. **`DatabaseBackupViewModelTests`** (new `tests/VideoIndexer.App.Tests/DatabaseBackupViewModelTests.cs`):
    - `BackupCommand_CallsService_WithSelectedFolder` — verifies `IDatabaseBackupService.BackupAsync`
      is called with the folder from `DestinationFolder`.
    - `BackupCommand_SetsResultMessage_OnSuccess` — service returns `Success = true` →
      `ResultMessage` reflects the output path.
    - `BackupCommand_SetsResultMessage_OnFailure` — service returns `Success = false` →
      `ResultMessage` reflects the error.
    - `CloseCommand_FiresCloseRequested`.

31. **`ObfuscationServiceTests`** (new `tests/VideoIndexer.Tests/ObfuscationServiceTests.cs`):
    Define private stub classes (`StubMovieRepository`, `StubSpdbConfigRepository`,
    `StubDbConnectionFactory`) inside the test file, following the inline pattern used by
    `TagsManagerTests.cs` in the same project. No shared test-helper files exist in
    `VideoIndexer.Tests`.
    - `IsEnabledAsync_ReturnsTrue_WhenConfigKeyIsOne` — `ISpdbConfigRepository` stub
      returns `"1"` → service returns `true`.
    - `IsEnabledAsync_ReturnsFalse_WhenConfigKeyIsNull` — returns `false` when key absent.
    - `ToggleAsync_Enable_SkipsAlreadyObfuscatedFile` — file whose stem equals hash is
      skipped (no `File.Move` call).
    - `ToggleAsync_Disable_SkipsFilesWithNullOriginalFilename` — `OriginalFilename = null`
      → no rename attempted.
    - `ToggleAsync_WritesObfuscationState_ToConfig` — after run, verifies
      `ISpdbConfigRepository.SetAsync` called with key `"obfuscation_enabled"` and correct
      value.

32. **`MysqlDumpBackupServiceTests`** (new `tests/VideoIndexer.Tests/MysqlDumpBackupServiceTests.cs`):
    - `BackupAsync_ReturnsError_WhenMySqlBinaryFolderNotConfigured` — `ISettingsService`
      returns options with `MySqlBinaryFolder = null` → `BackupAsync` returns
      `DatabaseBackupResult(false, null, …)` without spawning a process.
    - `BuildStartInfo_AddsArgumentsAsDiscreteEntries` — calls the `internal static
      MysqlDumpBackupService.BuildStartInfo(...)` method directly and asserts that host,
      port, user, password, and database are each present as a discrete entry in
      `ProcessStartInfo.ArgumentList` (not assembled into a single `Arguments` string).
      This approach does not require spawning a real process.
    - `BackupAsync_ReturnsError_OnNonZeroExitCode` — process exits with code 1 →
      `DatabaseBackupResult(false, null, capturedStderr)` is returned.

33. **`MainContentViewModelTests.cs`** (modified — `tests/VideoIndexer.App.Tests/MainContentViewModelTests.cs`):
    - Update `BuildSut()` to pass a `SettingsViewModel` instance (constructed with fake
      services including a `FakeSettingsService`, `FakeThemeService`,
      `FakeDatabaseConnectionStore`, and a minimal `ObfuscationSettingsViewModel`) and
      `null` for the `IDatabaseBackupDialogService?` parameter.
    - Update the `Constructor_AllDestinationsHaveNonNullRootViewModel` assertion to
      verify that the "settings" destination is now backed by the injected
      `SettingsViewModel` (not a `StubViewModel`).

---

## Dependencies

- Phase 1 must complete before Phases 2–7 (schema migration needed to write tests
  against the updated `GetOriginalFilenameAsync`).
- Phase 2 (interfaces) must complete before Phases 3–5 (implementations and ViewModels
  depend on them).
- Phases 3 and 4 are independent of each other; both must complete before Phase 5.
- Phase 5 must complete before Phase 6 (wiring).
- Phase 7 (tests) can be written in parallel with Phases 3–6 (test-helpers and
  unit tests only; integration tests require a running DB).


## Required Components

**New files:**
- `src/VideoIndexer.Core/Options/ThumbnailsOptions.cs`
- `src/VideoIndexer.Core/Models/MovieForObfuscation.cs`
- `src/VideoIndexer.Core/Models/ObfuscationProgress.cs`
- `src/VideoIndexer.Core/Models/DatabaseBackupResult.cs`
- `src/VideoIndexer.Core/Abstractions/IObfuscationService.cs`
- `src/VideoIndexer.Core/Abstractions/IDatabaseBackupService.cs`
- `src/VideoIndexer.Infrastructure/Database/migrations/m041_add_original_filename.sql`
- `src/VideoIndexer.Infrastructure/Library/ObfuscationService.cs`
- `src/VideoIndexer.Infrastructure/Database/MysqlDumpBackupService.cs`
- `src/VideoIndexer.App/Services/IDatabaseBackupDialogService.cs`
- `src/VideoIndexer.App/Services/AvaloniaDatabaseBackupDialogService.cs`
- `src/VideoIndexer.App/ViewModels/ObfuscationSettingsViewModel.cs`
- `src/VideoIndexer.App/ViewModels/SettingsViewModel.cs`
- `src/VideoIndexer.App/ViewModels/DatabaseBackupViewModel.cs`
- `src/VideoIndexer.App/Views/SettingsView.axaml`
- `src/VideoIndexer.App/Views/SettingsView.axaml.cs`
- `src/VideoIndexer.App/Views/DatabaseBackupView.axaml`
- `src/VideoIndexer.App/Views/DatabaseBackupView.axaml.cs`
- `tests/VideoIndexer.App.Tests/SettingsViewModelTests.cs`
- `tests/VideoIndexer.App.Tests/ObfuscationSettingsViewModelTests.cs`
- `tests/VideoIndexer.App.Tests/DatabaseBackupViewModelTests.cs`
- `tests/VideoIndexer.Tests/ObfuscationServiceTests.cs`
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeObfuscationService.cs`
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeDatabaseBackupService.cs`
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeThemeService.cs`
- `tests/VideoIndexer.Tests/MysqlDumpBackupServiceTests.cs`
- `docs/projects/rebuild/milestones/m8-system-tools.md`

**Modified files:**
- `src/VideoIndexer.Core/Options/AppOptions.cs`
- `src/VideoIndexer.Core/Options/ExternalToolsOptions.cs`
- `src/VideoIndexer.Core/Abstractions/IMovieRepository.cs`
- `src/VideoIndexer.App/Assets/appsettings.json`
- `src/VideoIndexer.Infrastructure/Database/DatabaseBootstrapper.cs`
- `src/VideoIndexer.Infrastructure/Library/DapperMovieRepository.cs`
- `src/VideoIndexer.App/ViewModels/MainContentViewModel.cs`
- `src/VideoIndexer.App/Views/MainContentView.axaml`
- `src/VideoIndexer.App/Program.cs`
- `tests/VideoIndexer.App.Tests/MainContentViewModelTests.cs`
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeMovieRepository.cs`
- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperMovieRepositoryTests.cs`
- `tests/VideoIndexer.Infrastructure.Tests/Database/DatabaseBootstrapperTests.cs`
- `docs/agents/project-manifest/api-surface.md`
- `docs/agents/project-manifest/file-tree.md`
- `docs/agents/project-manifest/constraints.md`
- `docs/agents/project-manifest/data-flows.md`
- `docs/agents/project-manifest/tech-stack.md`


## Assumptions

- No new NuGet packages are needed. `System.Diagnostics.Process` (already in .NET BCL) is
  sufficient for spawning `mysqldump`. No file-system abstraction library is required.
- The `IFolderPickerService` interface already supports folder-only selection (it does —
  `AvaloniaFolderPickerService` is registered as a singleton).
- `IObfuscationMap.NativeToObfuscated` is a property only on the concrete
  `ExtensionObfuscationMap`, not on the `IObfuscationMap` interface. The extension map
  table in `ObfuscationSettingsViewModel` is therefore built from the interface's existing
  members in `LoadAsync`: `_obfuscationMap.RealExtensions.ToDictionary(e => e, e =>
  { _obfuscationMap.TryGetObfuscatedExtension(e, out var o); return o ?? e; })` — no
  interface change required (Design Review Concern #1, Option B).
- The `DatabaseBackupViewModel` factory pattern (constructing via `new` in the service,
  not via DI factory) is acceptable since `DatabaseBackupViewModel` has no per-instance
  parameters beyond its injected services — same as `MoviePropertiesViewModel`.
- Language switching (Appearance tab) will persist the value to `AppOptions.Appearance.Language`
  but localization itself is not implemented in M8. The ComboBox is wired; the actual
  locale switch is deferred to a future milestone. The tab will display a note accordingly.
- The `mysqldump` password is passed via a command-line argument (standard practice).
  `ProcessStartInfo.ArgumentList` prevents shell injection, though the password will be
  briefly visible in the process list on Linux/macOS. This is accepted behaviour for a
  desktop application (matches the original app's behaviour).


## Constraints

- **`TreatWarningsAsErrors`** — all new code must compile warning-free.
- **No new NuGet packages** — all functionality achievable with BCL + existing packages.
- **`Core` has no external NuGet dependencies** — `IObfuscationService` and
  `IDatabaseBackupService` are Core interfaces with no library references.
- **Schema revision must be 41 after M8** — `DatabaseBootstrapper.ExpectedRevision = 41`.
- **`AppOptions` immutability** — `SettingsViewModel.SaveCommand` must produce a new
  record via `with { }` and call `ISettingsService.SaveAsync`; no in-place mutation.
- **Cancellation contract** — `ObfuscationService.ToggleAsync` must return normally on
  cancellation; must not propagate `OperationCanceledException`.
- **`ProcessStartInfo.ArgumentList`** — `MysqlDumpBackupService` must use the argument
  list API (not string concatenation) to avoid injection from user-supplied connection
  field values.
- **Theme change side-effect** — when the user saves a new theme in Preferences,
  `IThemeService.SetAsync` must be called (in addition to saving `AppOptions`) so that
  `ThemeService` fires `ThemeChanged` and `App.axaml.cs` applies the new variant
  immediately without requiring a restart.


## Out of Scope

- Localization / string translation (Appearance → Language field is wired but
  locale switching does not take effect until a dedicated localization milestone).
- In-app connection editing from the Database tab (handled by the startup connection
  picker; read-only display only in M8).
- Database **restore** (spec §4 explicitly excludes it).
- The "extension mapping" in the Obfuscation tab is **read-only** — the map is hardcoded
  in `ExtensionObfuscationMap`; no UI to change it is planned.
- Visual Obfuscation per-file error dialog — errors are collected in a list shown on
  the Obfuscation tab after the worker completes; no per-file interrupts.
- M9 thumbnail size application — `ThumbnailsOptions` is persisted by M8 but the actual
  thumbnail rendering respects it only when M9 is implemented.
- M10 player launch — `UseVlc` and `VlcExecutablePath` are persisted and displayed but
  the launch logic is implemented in M10.


## Acceptance Criteria

1. **Preferences page visible** — clicking the "Settings" rail item navigates to
   `SettingsView`; all six tabs are present.
2. **General tab round-trips** — entering a path in the MySQL folder field, clicking
   Save, then reopening Settings shows the persisted value.
3. **Appearance tab applies theme immediately** — changing the theme ComboBox and saving
   changes the application theme without restart.
4. **Logging tab persists verbosity** — changing verbosity and saving writes the new
   value to `appsettings.json`.
5. **Thumbnails tab persists size and flag** — `ThumbnailSize` and `KeepOriginalSize`
   round-trip through `appsettings.json`.
6. **Database tab shows active connection** — host, database name, and username from the
   active `DatabaseConnectionOptions` are displayed read-only.
7. **Cancel reverts fields** — changing fields and clicking Cancel without saving reverts
   all non-Obfuscation tab fields to prior values.
8. **Database Backup — happy path** — with `MySqlBinaryFolder` configured, opening
   File → Database Backup…, selecting a folder, and clicking Backup runs the backup
   worker; success message and output path are shown.
9. **Database Backup — missing binary** — without `MySqlBinaryFolder` configured, the
   Backup button produces an error message describing the missing configuration.
10. **Obfuscation toggle — enable** — clicking Enable triggers `IObfuscationService.ToggleAsync(true)`;
    progress updates are visible; status label shows "Obfuscation: Enabled" after
    completion.
11. **Obfuscation toggle — disable** — clicking Disable triggers `ToggleAsync(false)`;
    status label shows "Obfuscation: Disabled" after completion.
12. **Extension map table** — the Obfuscation tab shows a read-only table of the 10
    extension pairs.
13. **Schema at revision 41** — a fresh DB that passes migration from revision 35
    includes the `movies.original_filename` column and `db_revision = '41'`.
14. **Zero build warnings** — `dotnet build` produces no warnings.
15. **All tests pass** — `dotnet test` succeeds.


## Testing Strategy

Unit tests cover the new ViewModels using fake services. The `ObfuscationService` is
unit-tested with a fake `IMovieRepository` (returning controlled `MovieForObfuscation`
lists) and a fake filesystem (or by using temporary directories in an integration test).
`MysqlDumpBackupService` is unit-tested for argument construction and error handling
without invoking the actual `mysqldump` binary (use a fake `Process` or test only the
argument-list preparation logic). Integration tests (in `VideoIndexer.Infrastructure.Tests`)
cover `DapperMovieRepository.GetAllMoviesForObfuscationAsync` and the m041 migration
against the real test database.


## Test Plan

- `tests/VideoIndexer.App.Tests/SettingsViewModelTests.cs`
  - `LoadAsync_PopulatesGeneralTabFromOptions` — asserts `MySqlBinaryFolder` matches
    `AppOptions.ExternalTools.MySqlBinaryFolder` — AC 2
  - `LoadAsync_PopulatesToggles` — asserts `UseVlc` and `ThumbnailSize` match current
    options — AC 5
  - `SaveCommand_PersistsAllTabFields` — mutates multiple fields; asserts
    `SaveAsync` receives a correctly shaped `AppOptions` — AC 2, 4, 5
  - `SaveCommand_CallsThemeService_OnThemeChange` — AC 3
  - `CancelCommand_RevertsToCurrentOptions` — AC 7

- `tests/VideoIndexer.App.Tests/ObfuscationSettingsViewModelTests.cs`
  - `LoadAsync_SetsIsEnabled_WhenServiceReturnsTrue` — AC 10, 11
  - `ToggleCommand_CallsToggleWithInvertedEnabled` — AC 10, 11
  - `ToggleCommand_SetsBusyTrue_DuringExecution` — AC 10

- `tests/VideoIndexer.App.Tests/DatabaseBackupViewModelTests.cs`
  - `BackupCommand_CallsServiceWithFolder` — AC 8
  - `BackupCommand_ShowsSuccessMessage_OnSuccess` — AC 8
  - `BackupCommand_ShowsErrorMessage_OnFailure` — AC 9
  - `CloseCommand_FiresCloseRequested` — defensive

- `tests/VideoIndexer.Tests/ObfuscationServiceTests.cs`
  - `IsEnabledAsync_ReturnsTrue_WhenKeyIsOne` — AC 10
  - `IsEnabledAsync_ReturnsFalse_WhenKeyIsAbsent` — AC 11
  - `ToggleAsync_Enable_SkipsAlreadyObfuscatedFile` — AC 10 (idempotency)
  - `ToggleAsync_Disable_SkipsUnobfuscatedFile` — AC 11 (idempotency)
  - `ToggleAsync_Enable_WritesEnabledKeyToConfig` — AC 10
  - `ToggleAsync_Disable_WritesDisabledKeyToConfig` — AC 11

- `tests/VideoIndexer.Infrastructure.Tests/` *(integration)*
  - `DapperMovieRepositoryTests` — `GetAllMoviesForObfuscationAsync_ReturnsRowsWithHashAndFilename`
    — AC 13 (requires test DB at revision 41)
  - `DatabaseBootstrapperTests` — `MigratesFrom40To41_AddsOriginalFilenameColumn` — AC 13

- `tests/VideoIndexer.Tests/MysqlDumpBackupServiceTests.cs`
  - `BackupAsync_ReturnsError_WhenMySqlBinaryFolderNotConfigured` — AC 9
  - `BackupAsync_BuildsProcessWithArgumentList_NotStringConcatenation` — AC 8 (injection safety)
  - `BackupAsync_ReturnsError_OnNonZeroExitCode` — AC 9


## Documentation Updates

Per the AGENTS.md maintenance rules:

- `docs/agents/project-manifest/api-surface.md` — add `IObfuscationService`,
  `IDatabaseBackupService`, `IMovieRepository.GetAllMoviesForObfuscationAsync`;
  new models `MovieForObfuscation`, `ObfuscationProgress`, `DatabaseBackupResult`;
  `ThumbnailsOptions`, `ExternalToolsOptions.UseVlc`, `AppOptions.Thumbnails`; update
  `IDatabaseBootstrapper` note on `ExpectedRevision`
- `docs/agents/project-manifest/file-tree.md` — add all new files under their respective
  directories; update "settings" stub annotation; add menu-bar note to
  `MainContentView.axaml`
- `docs/agents/project-manifest/constraints.md` — add schema revision note (rev 40 → 41
  with m041 rollback procedure); add `ProcessStartInfo.ArgumentList` rule for
  `MysqlDumpBackupService`; add `ObfuscationService` cancellation note
- `docs/agents/project-manifest/data-flows.md` — add "8. Preferences Save" and
  "9. Obfuscation Toggle" flow diagrams; update "ShellState.Ready" section to note that
  the "settings" destination now routes to `SettingsViewModel`
- `docs/agents/project-manifest/tech-stack.md` — update schema revision row from
  `Expected revision 40` to `Expected revision 41`
- `docs/projects/rebuild/milestones/m8-system-tools.md` — new milestone document
  following the template in `roadmap.md`


## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **`mysqldump` process spawning is platform-specific** — path separator, binary extension (`.exe` on Windows), and process-table visibility of the password differ across OSes | The plan targets Windows initially (the existing app is Windows-only for now); add `RuntimeInformation.IsOSPlatform` branching in `MysqlDumpBackupService` with a clear TODO comment for Linux/macOS |
| **Obfuscation toggle on a large library is slow** — renaming thousands of files synchronously on the UI thread | The toggle runs via `Task.Run` in `ObfuscationSettingsViewModel.ToggleCommand`; progress is reported per file; cancellation is supported |
| **`movies.original_filename` column conflicts with future schema changes** — m041 adds a nullable column that other parts of the app do not yet write | The column is nullable; all existing queries that do not know about it are unaffected; the only writer is `ObfuscationService.ToggleAsync` |
| **Partial obfuscation run (cancelled mid-way)** — some files renamed, some not; `spdb_config` state written at end | The service writes `spdb_config` only after completing (or on cancellation-after-partial); the partial state is accepted — the user can re-run the toggle to complete. Document this in the Obfuscation tab UI ("If cancelled, some files may already be renamed; run again to complete.") |
| **`IThemeService.SetAsync` and `ISettingsService.SaveAsync` must both succeed on theme change** — partial failure leaves settings and theme out of sync | `SettingsViewModel.SaveCommand` calls `ISettingsService.SaveAsync` first (persistent), then `IThemeService.SetAsync` (in-memory + re-saves); if the second call fails the theme reverts at next launch (disk wins). This is acceptable for a desktop app. |
| **`DapperMovieRepository.GetAllMoviesForObfuscationAsync` returns one row per filename** — a movie with no filenames in `movies_filenames` would be skipped silently | This is correct behaviour — a movie with no filenames has nothing to rename. The progress count should be based on filename rows, not movie rows. |
