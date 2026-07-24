# Plan

## Plan Audit Cycles
- Audits: 1 — Plan Auditor v1.3.0
- Architectural Reviews: 2 — Plan Architect Reviewer v1.4.0


## Summary

This plan addresses all 9 actionable items surfaced in the M8 System Tools synthesis report (`docs/agents/plans/2026-05-19-m8-system-tools/synthesis.md`). The tenth item ("original filename in movie editor") is explicitly deferred to M9. The 9 items are organised into 7 work packages covering: shared test infrastructure, new `SettingsViewModel` unit tests, `DatabaseBackupViewModel` CanExecute guard + service-level security hardening, a cancel-while-busy feature for the backup dialog, two AXAML view UX fixes, the medium-priority `--defaults-extra-file` password-exposure security fix, and `constraints.md` documentation additions.


## Architectural Context

All changes touch three existing layers:

- **Infrastructure** (`src/VideoIndexer.Infrastructure/Database/MysqlDumpBackupService.cs`) — two security fixes: path traversal in output filename and password exposure via process listing.
- **App / ViewModels** (`src/VideoIndexer.App/ViewModels/DatabaseBackupViewModel.cs`, `SettingsViewModel.cs`) — CanExecute guard, cancel command, and new unit test coverage.
- **App / Views** (`src/VideoIndexer.App/Views/SettingsView.axaml`, `DatabaseBackupView.axaml`) — error banner and Errors-label visibility binding corrections.
- **Tests** (`tests/VideoIndexer.App.Tests/`, `tests/VideoIndexer.Tests/`) — shared fake promotion, new tests.
- **Docs** (`docs/agents/project-manifest/constraints.md`) — two new convention entries.

Key integration points:
- `MysqlDumpBackupService.BackupAsync` / `BuildStartInfo` (internal static, directly called from tests) — any signature change propagates to `MysqlDumpBackupServiceTests.cs`.
- `DatabaseBackupViewModel` is instantiated by `AvaloniaDatabaseBackupDialogService` (`src/VideoIndexer.App/Services/AvaloniaDatabaseBackupDialogService.cs`) via direct `new`; no DI factory. CanExecute and Cancel changes are transparent to that service.
- `SettingsViewModel.cs` + `SettingsViewModelTests.cs` — the `_loadedTheme` guard is already implemented; only tests are missing.
- `StubObfuscationMap` exists as a private nested class in three test files: `SettingsViewModelTests.cs` (line 123), `ObfuscationSettingsViewModelTests.cs` (line 72), `MainContentViewModelTests.cs` (line 379).


## Approach / Architecture

Each work package is self-contained and minimally scoped:

1. **WP-001** — pure refactor: extract `StubObfuscationMap` into `tests/VideoIndexer.App.Tests/TestHelpers/FakeObfuscationMap.cs` and update three call sites. No logic change.
2. **WP-002** — new tests only: add three `[Fact]` methods to `SettingsViewModelTests.cs` covering `_loadedTheme` guard (second-save optimisation), `CancelCommand` contract (no `ObfuscationVm.LoadAsync`), and `SetAsync`-throws recovery.
3. **WP-003** — `DatabaseBackupViewModel` `CanExecute` guard (one-line attribute change) + two `MysqlDumpBackupService.BackupAsync` fixes (filename sanitization, partial-file cleanup on cancellation) + corresponding test additions.
4. **WP-004** — cancel-while-busy feature: change `BackupCommand` attribute to `IncludeCancelCommand = true`, update `DatabaseBackupView.axaml` to surface the generated `CancelBackupCommand`, always-enable the Close button, update tests.
5. **WP-005** — pure AXAML: add an error banner row to `SettingsView.axaml` bound to `HasLoadError`/`LoadErrorMessage`; fix the Errors-label `IsVisible` binding from `IsBusy` to `Errors.Count > 0`.
6. **WP-006** — security fix: write the MySQL password to a per-run temp file in the user's `%TEMP%` folder using `--defaults-extra-file`, removing `-p{password}` from the argument list; delete the temp file in a `finally` block; update `BuildStartInfo` signature and all related tests.
7. **WP-007** — docs: add two entries to `constraints.md` (MoviesListView view-lifecycle template; DapperMovieRepository INNER JOIN vs correlated-subquery rule) and update the api-surface.md note for `DatabaseBackupViewModel.CancelBackupCommand` (added in WP-004).

The `--defaults-extra-file` approach (WP-006) is the only widely-supported way to pass a MySQL password without exposing it in the process argument list. The file is created in `Path.GetTempPath()` (per-user directory on Windows), written with a minimal `[mysqldump]\npassword=<value>` body, and deleted immediately after `WaitForExitAsync`. Full ACL restriction to owner-read-only (via `FileSecurity`) is noted as a stretch goal but is not required; the per-user `%TEMP%` boundary is sufficient for the desktop-only threat model.


## Rationale

- Grouping WP-003 and WP-004 as separate packages allows the simpler CanExecute fix and low-security-severity cleanup to ship before the more complex cancel-while-busy feature, reducing the rework surface.
- Keeping WP-005 as a pure AXAML WP avoids coupling view-layer changes to Infrastructure changes. It can run in parallel with WP-006.
- WP-001 precedes WP-002 so the new `SettingsViewModel` tests can use the shared `FakeObfuscationMap` from the start, avoiding yet another private nested copy.
- The synthesis explicitly flags item 10 (original-filename in movie editor) as M9 scope. It is excluded here.


## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|---|---|---|---|
| Cancel mechanism for BackupCommand | `IncludeCancelCommand = true` on `[RelayCommand]` | Manual `CancellationTokenSource` field + explicit `[RelayCommand]` cancel method | CommunityToolkit.Mvvm auto-generates `CancelBackupCommand` wired to the correct token; manual CTS is more code with no benefit. |
| Password security: temp-file placement | `Path.GetTempPath()` (per-user `%TEMP%`) | Owner-ACL via `FileSecurity`; named pipe | ACL manipulation requires P/Invoke on Windows; named pipe requires bidirectional coordination. `%TEMP%` is already user-scoped. Full ACL can be added later. |
| Path traversal fix: sanitize vs reject | Sanitize with `Path.GetInvalidFileNameChars()` | Reject if invalid chars present and return an error | Sanitizing preserves UX (partial match still works) and is consistent with how most utilities handle the case. The resulting filename is deterministic. |
| Partial-backup cleanup on cancellation | Catch `OperationCanceledException`, delete file, re-throw | Return a `DatabaseBackupResult(false, ...)` on cancel | Re-throwing preserves the standard cancellation contract; the ViewModel's `catch` in its `finally` already sets `IsBusy = false`. CommunityToolkit.Mvvm `AsyncRelayCommand` handles `OperationCanceledException` silently (does not propagate to the UI thread unhandled). |
| SettingsView error banner placement | New `Grid.Row="0"` error banner above `TabControl` | Overlay within each tab | A single top-level banner is consistent with `ErrorView.axaml` and avoids duplicating the binding in every tab. |
| Credentials file cleanup scope | Initialize path to `null` before `try`; write and use inside `try`; null-check in `finally` | Write file before entering `try`, clean in `finally` | Writing before the `try` leaves an orphaned `.cnf` file if `WriteAllTextAsync` is cancelled mid-write, because the outer `try` has not been entered yet. The null-initialized pattern closes this gap: the `finally` always runs once entered, and the null-check prevents a spurious delete if an exception is thrown before the path is assigned. |


## Pattern Alignment

| Pattern | File | Status |
|---|---|---|
| Shared test helpers in `TestHelpers/` | `tests/VideoIndexer.App.Tests/TestHelpers/` | WP-001 follows this pattern by promoting `StubObfuscationMap` |
| `FakeXxx` naming convention for test doubles | `FakeObfuscationService.cs`, `FakeDatabaseBackupService.cs`, etc. | WP-001 names the new file `FakeObfuscationMap.cs` — consistent |
| `[RelayCommand(IncludeCancelCommand = true)]` pattern | CommunityToolkit.Mvvm usage throughout the codebase | WP-004 follows this pattern instead of a manual CTS |
| `MoviesListView.axaml.cs` lifecycle template (OnLoaded/OnUnloaded) | `src/VideoIndexer.App/Views/MoviesListView.axaml.cs` | WP-007 codifies this as a `constraints.md` rule |
| Private row classes at top of `DapperMovieRepository.cs` | `src/VideoIndexer.Infrastructure/Library/DapperMovieRepository.cs` | Already documented in constraints.md |
| ArgumentList (no string concatenation) for process arguments | `MysqlDumpBackupService.cs`, `WindowsFileLauncherService.cs` | WP-006 maintains this pattern; temp-file path is passed as a discrete entry |
| `try/finally` temp-resource cleanup | Pattern used in `AvaloniaDatabaseBackupDialogService.cs` (CloseRequested unsubscription) | WP-006 follows `try/finally` for temp file deletion |


## Detailed Steps

### WP-001 — Promote `StubObfuscationMap` to shared test helper

1. Create `tests/VideoIndexer.App.Tests/TestHelpers/FakeObfuscationMap.cs` implementing `IObfuscationMap` with a fixed empty dictionary return (identical logic to the current private nested classes).
2. In `SettingsViewModelTests.cs` (line ~123): delete the private `sealed class StubObfuscationMap`, update `BuildSut` to use `new FakeObfuscationMap()` (no import change needed — same namespace).
3. In `ObfuscationSettingsViewModelTests.cs` (line ~72): delete the private `sealed class StubObfuscationMap`, update `BuildSut` parameter default.
4. In `MainContentViewModelTests.cs` (line ~379): delete the private `sealed class StubObfuscationMap`, update the `new StubObfuscationMap()` call site at line ~42 to `new FakeObfuscationMap()`.
5. Run `dotnet build` — verify zero warnings/errors.
6. Run `dotnet test` — verify all 814 tests still pass.

### WP-002 — Add `SettingsViewModel` unit tests

Add three new `[Fact]` methods to `tests/VideoIndexer.App.Tests/SettingsViewModelTests.cs`, inside the existing `SettingsViewModelTests` class:

**Test A — `_loadedTheme` guard (second-save optimisation):**
```
SaveCommand_DoesNotCallThemeService_OnSecondSave_WhenThemeUnchanged
```
- Start with `ThemeMode.System`. `LoadAsync`. Change to `ThemeMode.Dark`. `SaveCommand`. Assert `SetCallCount == 1`.
- Execute `SaveCommand` again without changing `SelectedTheme`. Assert `SetCallCount` is still `1` (second save did not call `SetAsync`).

**Test B — `CancelCommand` does not reload `ObfuscationVm`:**
```
CancelCommand_DoesNotCallObfuscationVmLoadAsync
```
- Introduce a `LoadCallCount` property on `FakeObfuscationService` that increments in `IsEnabledAsync` (not in `ToggleAsync`). Use a local factory for this test that exposes the `FakeObfuscationService` instance directly (`SutBundle` does not expose it, so do not use `BuildSut` here). `LoadAsync`. Assert `LoadCallCount == 1` (initial load). `CancelCommand.Execute(null)`. Assert `LoadCallCount` is still `1` (Cancel must not trigger a second `IsEnabledAsync` call).

**Test C — `SetAsync` throws; `_loadedTheme` not advanced:**
```
SaveCommand_RetriesThemeService_IfPreviousSetAsyncThrew
```
- Use a `ThrowingThemeService` stub (inline, private nested class) whose first `SetAsync` call throws `InvalidOperationException`; second call succeeds.
- `LoadAsync`. Change `SelectedTheme`. First `SaveCommand` — assert exception is NOT propagated (command swallows it? — check source). Actually, looking at `SettingsViewModel.Save`, the theme call is `await _themeService.SetAsync(...)` — if it throws, the exception propagates out of `Save`, which propagates out of the `AsyncRelayCommand`. The ViewModel does not swallow it. The test needs to verify that `_loadedTheme` was not updated. Approach: catch the exception from `SaveCommand.ExecuteAsync`, then call `SaveCommand.ExecuteAsync` a second time and assert `ThemeService.SetCallCount == 2`.

Run `dotnet test` — verify total passes increase by 3.

### WP-003 — `DatabaseBackupViewModel` CanExecute guard + `MysqlDumpBackupService` security lows

**3a. `DatabaseBackupViewModel` CanExecute guard:**
- In `src/VideoIndexer.App/ViewModels/DatabaseBackupViewModel.cs`, change:
  ```csharp
  [RelayCommand(IncludeCancelCommand = false)]
  private async Task Backup(CancellationToken cancellationToken)
  ```
  to:
  ```csharp
  [RelayCommand(IncludeCancelCommand = false, CanExecute = nameof(CanBackup))]
  private async Task Backup(CancellationToken cancellationToken)
  
  private bool CanBackup() => !string.IsNullOrWhiteSpace(DestinationFolder);
  ```
- Add `OnDestinationFolderChanged` partial method (generated by `[ObservableProperty]`) to notify the command:
  ```csharp
  partial void OnDestinationFolderChanged(string value) => BackupCommand.NotifyCanExecuteChanged();
  ```

**3b. `MysqlDumpBackupService` filename sanitization:**
- In `BackupAsync`, replace:
  ```csharp
  var databaseName = conn.Database ?? "backup";
  ```
  with:
  ```csharp
  var rawName      = conn.Database ?? "backup";
  var invalidChars = Path.GetInvalidFileNameChars();
  var databaseName = string.Concat(rawName.Where(c => !invalidChars.Contains(c)));
  if (string.IsNullOrEmpty(databaseName)) databaseName = "backup";
  ```

**3c. `MysqlDumpBackupService` partial-file cleanup on cancellation:**
- In `BackupAsync`, replace the existing `try/catch` block so that `OperationCanceledException` triggers file cleanup before re-throwing:
  ```csharp
  catch (OperationCanceledException)
  {
      if (File.Exists(outputFilePath))
          File.Delete(outputFilePath);
      throw;
  }
  ```
  Insert this catch clause _before_ the existing `catch (Exception ex) when (ex is not OperationCanceledException)` clause.

**3d. Test updates in `tests/VideoIndexer.Tests/MysqlDumpBackupServiceTests.cs`:**
- Add `BackupAsync_SanitizesPathSeparatorsInDatabaseName` — calls `BackupAsync` with a `DatabaseConnectionOptions` where `Database = "../../evil"`, asserts the generated output file name does not contain `/` or `\`.  
  _Caveat: This requires running the full process or mocking. Since process execution cannot be unit-tested, test the sanitization logic extracted as an internal helper, or verify via integration with a `FakeBackupService`. Alternative: extract the sanitization to a private static method `SanitizeDatabaseName` and test it directly (it is `private` — test via `InternalsVisibleTo` or make it `internal static`)._  
  **Chosen approach:** make `SanitizeDatabaseName` `internal static` (consistent with `BuildStartInfo` being `internal static`) and test it directly.
- Add `BackupCommand_CanExecute_IsFalseWhenDestinationFolderIsEmpty` to `tests/VideoIndexer.App.Tests/DatabaseBackupViewModelTests.cs`.
- Add `BackupCommand_CanExecute_IsTrueWhenDestinationFolderIsSet` to `tests/VideoIndexer.App.Tests/DatabaseBackupViewModelTests.cs`.

### WP-004 — `DatabaseBackupView` cancel-while-busy feature

**4a. `DatabaseBackupViewModel`:**
- Change `[RelayCommand(IncludeCancelCommand = false, CanExecute = ...)]` → `[RelayCommand(IncludeCancelCommand = true, CanExecute = nameof(CanBackup))]`. CommunityToolkit.Mvvm generates `CancelBackupCommand`.

**4b. `DatabaseBackupView.axaml`:**
- Add a Cancel button between the Backup button and Close button, bound to `CancelBackupCommand`, visible/enabled only while `IsBusy`:
  ```xml
  <Button Content="Cancel"
          Command="{Binding CancelBackupCommand}"
          IsVisible="{Binding IsBusy}" />
  ```
- Remove `IsEnabled="{Binding !IsBusy}"` from the Close button so it is always enabled. The user can close the dialog at any time; the `CancelBackupCommand` handles in-progress cancellation.

**4c. `DatabaseBackupViewModelTests.cs`:**
- Add `CancelBackupCommand_CancelsInProgressBackup` — use a `DelayedDatabaseBackupService` (blocking fake) to observe that calling `CancelBackupCommand` causes `BackupCommand`'s task to complete (via `OperationCanceledException` handling), and that `IsBusy` returns to `false`.
  `DelayedDatabaseBackupService` must be a **private nested class inside `DatabaseBackupViewModelTests.cs`**, following the identical pattern as `DelayedObfuscationService` in `ObfuscationSettingsViewModelTests.cs` (line 96). Do not add it to `TestHelpers/`.

### WP-005 — `SettingsView.axaml` error banner + Obfuscation Errors label fix

**5a. Error banner — `src/VideoIndexer.App/Views/SettingsView.axaml`:**
- Change the root `Grid` from `RowDefinitions="Auto,*,Auto"` to `RowDefinitions="Auto,Auto,*,Auto"`.
- Add a new row (Grid.Row="1") containing an error banner `Border`/`TextBlock` visible only when `HasLoadError = true`:
  ```xml
  <Border Grid.Row="1"
          Background="#FFB4B4"
          Padding="8,4"
          IsVisible="{Binding HasLoadError}">
    <TextBlock Text="{Binding LoadErrorMessage}"
               Foreground="#8B0000"
               TextWrapping="Wrap" />
  </Border>
  ```
- Update all existing `Grid.Row` values: `TabControl` moves to `Grid.Row="2"`, bottom toolbar moves to `Grid.Row="3"`.

**5b. Errors-label visibility — Obfuscation tab in `SettingsView.axaml`:**
- Find:
  ```xml
  <TextBlock Text="Errors" FontWeight="SemiBold"
             IsVisible="{Binding ObfuscationVm.IsBusy}"
             Margin="0,8,0,0" />
  ```
- Change `IsVisible` to use `Errors.Count`:
  ```xml
  <TextBlock Text="Errors" FontWeight="SemiBold"
             IsVisible="{Binding ObfuscationVm.Errors.Count, Converter={x:Static ObjectConverters.IsNotNull}}"
             Margin="0,8,0,0" />
  ```
  _Note: `ObservableCollection<T>.Count` is an `int`, not nullable; `IsNotNull` is not appropriate. Correct binding: since `Errors` is an `ObservableCollection<string>`, use a converter or bind to a computed bool property. The simplest zero-dependency approach is to add a computed `bool HasErrors => Errors.Count > 0` property to `ObfuscationSettingsViewModel` backed by a `CollectionChanged` subscription. Alternatively use `{Binding ObfuscationVm.HasErrors}` after adding the property._
  **Chosen approach:** Add `public bool HasErrors => Errors.Count > 0;` to `ObfuscationSettingsViewModel` and subscribe to `Errors.CollectionChanged` to raise `OnPropertyChanged(nameof(HasErrors))`. Bind `IsVisible="{Binding ObfuscationVm.HasErrors}"`. Also apply the same fix to the `ListBox` visibility (currently unconditionally visible).

### WP-006 — `MysqlDumpBackupService` `--defaults-extra-file` password security fix

**6a. `MysqlDumpBackupService.BackupAsync` changes:**
1. Initialize `credentialsFilePath` to `null` *before* the outer `try`, then write the file as the **first statement inside** the `try` body:
   ```csharp
   string? credentialsFilePath = null;
   try
   {
       credentialsFilePath = Path.Combine(Path.GetTempPath(), $"vindex_mysql_{Guid.NewGuid():N}.cnf");
       await File.WriteAllTextAsync(
           credentialsFilePath,
           $"[mysqldump]{Environment.NewLine}password={conn.Password ?? string.Empty}{Environment.NewLine}",
           cancellationToken).ConfigureAwait(false);
       /* … existing process start + stream copy + WaitForExitAsync … */
   }
   finally
   {
       if (credentialsFilePath is not null && File.Exists(credentialsFilePath))
           File.Delete(credentialsFilePath);
   }
   ```
   This ensures the entire credentials-file lifecycle — creation, use, deletion — is within one cleanup scope. If `WriteAllTextAsync` is cancelled or throws before the file is fully written, `credentialsFilePath` is non-null and the `finally` block will still delete the (possibly partial) file. If an exception is thrown before the assignment to `credentialsFilePath`, the null-check prevents a spurious `File.Delete` call.  
   The existing `catch (OperationCanceledException)` (added in WP-003) and `catch (Exception ex) when ...` clauses nest inside this same `try`.
2. Pass `credentialsFilePath` to `BuildStartInfo` as a new parameter.

**6b. `BuildStartInfo` signature change:**
- Add parameter: `string credentialsFilePath`
- Remove the `psi.ArgumentList.Add($"-p{conn.Password ?? string.Empty}");` line entirely.
- Add `--defaults-extra-file` as the **very first `Add` call** in the method body — before `--single-transaction`, `-u`, `-P`, `-h`, and the database name — by inserting the following as the opening line of the argument-building block:
  ```csharp
  psi.ArgumentList.Add($"--defaults-extra-file={credentialsFilePath}");
  ```
  Do **not** append it at the end and then reorder; write it first. mysqldump requires `--defaults-*` options to precede all other flags in some server versions, and the unit test assertion (`ArgumentList[0]` starts with `--defaults-extra-file`) enforces this contract.

**6c. `MysqlDumpBackupServiceTests.cs` updates:**
- Update `BuildStartInfo_PopulatesExpectedArguments`: pass a `credentialsFilePath` argument (e.g., `"/tmp/test.cnf"`); assert `--defaults-extra-file=/tmp/test.cnf` appears in `ArgumentList`; assert no entry starts with `-p`.
- Update `BuildStartInfo_DoesNotUseArgumentsStringForm`: add `credentialsFilePath` arg.
- Update `BuildStartInfo_ConfiguresRedirection`: add `credentialsFilePath` arg.
- Update `BuildStartInfo_NullConnectionFields_UsesEmptyStrings`: add `credentialsFilePath` arg; remove the existing `psi.ArgumentList.Should().Contain("-p")` assertion (no `-p` entry exists after WP-006); replace it with `psi.ArgumentList.Should().NotContain(x => x.StartsWith("-p"))` plus a positive `--defaults-extra-file` presence check.
- Update `BuildStartInfo_AddsArgumentsAsDiscreteEntries`: add `credentialsFilePath` arg; replace the `psi.ArgumentList.Should().Contain("-ps3cr3t")` and `psi.ArgumentList.Should().Contain("-p")` assertions with `psi.ArgumentList.Should().NotContain(x => x.StartsWith("-p"))` plus a positive `--defaults-extra-file` prefix check.
- Update `BackupAsync_ReturnsError_WhenMySqlBinaryFolderNotConfigured`, etc.: these tests do not call `BuildStartInfo` directly; they call `BackupAsync` which now writes a temp file. Ensure the temp file is cleaned up even in the error-early-exit path (add a `File.Exists` guard in test cleanup, or note the early-return paths do not create the file yet).

### WP-007 — `constraints.md` additions + manifest updates

**7a. Add to `constraints.md` → "UI & MVVM" section — view lifecycle template rule:**
```
**View code-behind lifecycle template (`MoviesListView.axaml.cs` pattern) — mandatory for all code-behind with async loads** — any `UserControl` code-behind that calls `vm.LoadAsync` must follow this pattern exactly:
(1) `OnLoaded`: create a fresh `CancellationTokenSource`; store as `_cts`; call `await vm.LoadAsync(_cts.Token)` inside a `try/catch(OperationCanceledException)` + `catch(Exception ex)` block — the general catch must set `vm.HasLoadError`/`vm.LoadErrorMessage`; (2) `OnUnloaded`: unsubscribe all event handlers subscribed in `OnDataContextChanged` or `OnLoaded`; call `_cts?.Cancel()`; call `_cts?.Dispose()`; set `_cts = null`. Deviating from this template (missing general catch, or missing event unsubscription) is a code-review blocker. See `MoviesListView.axaml.cs` as the canonical reference.
```

**7b. Add to `constraints.md` → "Database" section — INNER JOIN preference for aggregation queries:**
```
**`DapperMovieRepository` per-movie aggregation must use INNER JOIN + derived tables, not correlated subqueries** — a correlated subquery form (e.g., `(SELECT COUNT(*) FROM ... WHERE movie_id = m.movie_id)`) re-executes once per row and degrades to O(n) at scale. All per-movie sub-aggregation in `DapperMovieRepository` must use an INNER JOIN against a derived table or GROUP BY subquery (e.g., `INNER JOIN (SELECT movie_id, COUNT(*) AS cnt FROM ... GROUP BY movie_id) agg ON agg.movie_id = m.movie_id`). This was established in WP-004 of the M8 cycle (replaced a correlated-subquery form in `GetAllMoviesForObfuscationAsync`).
```

**7c. Update `api-surface.md`:**
- Add `CancelBackupCommand` to `DatabaseBackupViewModel` surface entry (added by WP-004).
- Add `HasErrors` computed property to `ObfuscationSettingsViewModel` surface entry (added by WP-005).
- Update `BuildStartInfo` to its new 4-parameter signature: `internal static ProcessStartInfo BuildStartInfo(string mysqldumpPath, DatabaseConnectionOptions conn, string outputFilePath, string credentialsFilePath)`. Update the accompanying note to describe the `credentialsFilePath` role (passed as `--defaults-extra-file`; password is no longer in the argument list). (WP-006)
- Add `SanitizeDatabaseName` as a new `internal static` test-accessible method entry under `MysqlDumpBackupService`: `internal static string SanitizeDatabaseName(string rawName)`. (WP-003)

**7d. Update the `SettingsViewModel` open-item note in `constraints.md`** — the note "SettingsViewModel has no unit tests" (in the Open Items section) must be updated once WP-002 is complete. Remove or revise the `## Open Items / Known Gotchas` entry accordingly.


## Dependencies

- WP-001 must complete before WP-002 (WP-002 uses `FakeObfuscationMap`).
- WP-003 must complete before WP-004 (both modify `DatabaseBackupViewModel.cs`; WP-003 adds `CanExecute`, WP-004 changes `IncludeCancelCommand`; processing them in sequence avoids merge conflicts).
- WP-007 should run after WP-002, WP-004, and WP-005 complete, as it records the final API surface produced by those WPs.
- All other WPs (WP-005, WP-006) are independent and can run in parallel with WP-001/WP-002.


## Required Components

**Modified files:**
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeObfuscationMap.cs` — **new file** (WP-001)
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeObfuscationService.cs` — add `LoadCallCount` (WP-002)
- `tests/VideoIndexer.App.Tests/SettingsViewModelTests.cs` — remove private `StubObfuscationMap`; add 3 tests (WP-001, WP-002)
- `tests/VideoIndexer.App.Tests/ObfuscationSettingsViewModelTests.cs` — remove private `StubObfuscationMap` (WP-001)
- `tests/VideoIndexer.App.Tests/MainContentViewModelTests.cs` — remove private `StubObfuscationMap` (WP-001)
- `tests/VideoIndexer.App.Tests/DatabaseBackupViewModelTests.cs` — add CanExecute tests; add cancel test; add `DelayedDatabaseBackupService` as a private nested class (WP-003, WP-004)
- `tests/VideoIndexer.Tests/MysqlDumpBackupServiceTests.cs` — update `BuildStartInfo` call sites; add sanitization + `--defaults-extra-file` tests (WP-003, WP-006)
- `src/VideoIndexer.App/ViewModels/DatabaseBackupViewModel.cs` — CanExecute guard; `IncludeCancelCommand = true` (WP-003, WP-004)
- `src/VideoIndexer.App/ViewModels/ObfuscationSettingsViewModel.cs` — add `HasErrors` computed property (WP-005)
- `src/VideoIndexer.App/Views/DatabaseBackupView.axaml` — Cancel button; Close always-enabled (WP-004)
- `src/VideoIndexer.App/Views/SettingsView.axaml` — error banner row; `HasErrors` binding (WP-005)
- `src/VideoIndexer.Infrastructure/Database/MysqlDumpBackupService.cs` — filename sanitization; partial-file cleanup; `--defaults-extra-file` (WP-003, WP-006)
- `docs/agents/project-manifest/constraints.md` — two new rules; revise open-item note (WP-007)
- `docs/agents/project-manifest/api-surface.md` — `CancelBackupCommand`; `HasErrors` (WP-007)


## Assumptions

- The `ObservableCollection<string> Errors` property in `ObfuscationSettingsViewModel` already exists and is populated from `ToggleAsync` error reporting (confirmed from the M8 implementation).
- `FakeObfuscationService.IsEnabledAsync` is called exactly once by `ObfuscationSettingsViewModel.LoadAsync`; adding a `LoadCallCount` counter to `FakeObfuscationService` captures this.
- `IObfuscationMap` is in `VideoIndexer.Core.Abstractions` — confirmed from grep results.
- On Windows, temp files in `Path.GetTempPath()` (`%LOCALAPPDATA%\Temp` or `%TEMP%`) are accessible only to the current user account, providing sufficient isolation for the desktop-only threat model.
- CommunityToolkit.Mvvm 8.3.2 (as pinned in `Directory.Packages.props`) generates a `CancelBackupCommand` property when `IncludeCancelCommand = true` on an `AsyncRelayCommand`.
- `OperationCanceledException` thrown from an `AsyncRelayCommand` body is swallowed by CommunityToolkit.Mvvm (does not propagate as an unhandled exception).


## Constraints

- All warnings are errors (`TreatWarningsAsErrors=true`) — zero warnings in all modified projects.
- No `Version=` attributes on `<PackageReference>` in `.csproj` files.
- Core project must remain free of external NuGet dependencies.
- `ObfuscationService.ToggleAsync` cancellation contract must not change — it still returns normally (no `OperationCanceledException` thrown).
- `BuildStartInfo` in `MysqlDumpBackupService` is `internal static` — this accessibility level must be preserved so existing unit tests can call it directly.
- The `--defaults-extra-file` entry must appear first in the `ArgumentList` (before `--single-transaction`) because some MySQL versions require option-file arguments to precede other options.


## Out of Scope

- Surfacing `GetOriginalFilenameAsync` in the movie editor or details panel — deferred to M9 as stated in synthesis item 10.
- Full owner-only ACL restriction (via `FileSecurity`) on the credentials temp file — noted as a stretch goal; out of scope for this plan.
- Evaluating whether `NullDbConnection`/`NullDbCommand` should be promoted to shared infrastructure (`VideoIndexer.Tests/Fixtures/`) — not referenced by any failing or new test; deferred.
- Any changes to the scanner, tag subsystem, or movie editor.


## Acceptance Criteria

- AC-1: `dotnet build` produces zero warnings and zero errors across all projects after all changes.
- AC-2: `dotnet test` passes all existing 814 tests plus the new tests added in WP-002, WP-003, and WP-004 (minimum +8 new tests).
- AC-3: `StubObfuscationMap` private nested class no longer appears in any test file; a single `FakeObfuscationMap.cs` exists in `TestHelpers/`.
- AC-4: `DatabaseBackupViewModel.BackupCommand.CanExecute(null)` returns `false` when `DestinationFolder` is `""` or whitespace, `true` when non-empty.
- AC-5: `DatabaseBackupViewModel` exposes `CancelBackupCommand`; `DatabaseBackupView.axaml` binds a Cancel button to it.
- AC-6: The Close button in `DatabaseBackupView.axaml` is always enabled (no `IsEnabled="{Binding !IsBusy}"`).
- AC-7: `SettingsView.axaml` renders a visible error banner when `HasLoadError = true`.
- AC-8: The Obfuscation tab "Errors" label is hidden when `Errors` is empty and shown when it contains entries (not toggled by `IsBusy`).
- AC-9: `-p{password}` no longer appears in the `ProcessStartInfo.ArgumentList` produced by `MysqlDumpBackupService`; `--defaults-extra-file=<path>` appears as the first entry instead.
- AC-10: A partial `.sql` file is deleted when `BackupAsync` is cancelled.
- AC-11: `constraints.md` contains entries for the MoviesListView lifecycle template and the INNER JOIN aggregation rule.
- AC-12: `api-surface.md` reflects `CancelBackupCommand` on `DatabaseBackupViewModel` and `HasErrors` on `ObfuscationSettingsViewModel`.


## Testing Strategy

All changes are covered by existing xUnit test projects. No new test project is needed. The test strategy is unit-only (no integration tests touch the new features):

- **WP-001:** Verified by the entire existing test suite passing after the refactor (no logic change).
- **WP-002:** Three new `[Fact]` tests in `SettingsViewModelTests.cs`.
- **WP-003:** Two new `[Fact]` tests in `DatabaseBackupViewModelTests.cs` (CanExecute); one new `[Fact]` in `MysqlDumpBackupServiceTests.cs` (sanitization via `internal static SanitizeDatabaseName`).
- **WP-004:** One new `[Fact]` in `DatabaseBackupViewModelTests.cs` (cancel during in-progress backup).
- **WP-005:** AXAML binding changes; verified by the build (compiled bindings catch type errors at build time). Additionally, add two `[Fact]` tests to `ObfuscationSettingsViewModelTests.cs` verifying `HasErrors` transitions (see Test Plan).
- **WP-006:** Updated `BuildStartInfo` call sites in `MysqlDumpBackupServiceTests.cs`; one new assertion that `--defaults-extra-file` is the first argument; one new assertion that `-p` prefix is absent from all arguments.
- **WP-007:** Documentation only; verified by human review.


## Test Plan

- `SettingsViewModelTests.SaveCommand_DoesNotCallThemeService_OnSecondSave_WhenThemeUnchanged` — asserts `_loadedTheme` guard: after a successful save with a theme change, a second save with no further change does not call `IThemeService.SetAsync` — AC-2
- `SettingsViewModelTests.CancelCommand_DoesNotCallObfuscationVmLoadAsync` — asserts `FakeObfuscationService.LoadCallCount` does not increment after `CancelCommand.Execute` — AC-2
- `SettingsViewModelTests.SaveCommand_RetriesThemeService_IfPreviousSetAsyncThrew` — asserts that when `SetAsync` throws on first `SaveCommand`, `_loadedTheme` is unchanged so the second `SaveCommand` calls `SetAsync` again — AC-2
- `DatabaseBackupViewModelTests.BackupCommand_CanExecute_IsFalseWhenDestinationFolderIsEmpty` — asserts `BackupCommand.CanExecute(null) == false` for `""` and `"   "` — AC-4
- `DatabaseBackupViewModelTests.BackupCommand_CanExecute_IsTrueWhenDestinationFolderIsSet` — asserts `BackupCommand.CanExecute(null) == true` when `DestinationFolder = @"C:\Backups"` — AC-4
- `DatabaseBackupViewModelTests.CancelBackupCommand_CancelsInProgressBackup` — asserts `IsBusy` transitions correctly and task terminates after cancel — AC-5
- `MysqlDumpBackupServiceTests.SanitizeDatabaseName_RemovesPathSeparators` — asserts `"../../evil"` → `"evil"` (or appropriate sanitized form with no `/` or `\`) — AC-10 (indirectly, confirms sanitization)
- `MysqlDumpBackupServiceTests.BuildStartInfo_UsesDefaultsExtraFile_NotPasswordArg` — asserts `ArgumentList` contains `--defaults-extra-file=<path>` as first argument and no entry begins with `-p` — AC-9
- `ObfuscationSettingsViewModelTests.HasErrors_IsFalse_WhenErrorsIsEmpty` — asserts `HasErrors == false` on a freshly constructed `ObfuscationSettingsViewModel` with no errors — AC-8
- `ObfuscationSettingsViewModelTests.HasErrors_IsTrue_WhenErrorsContainsEntry` — adds an entry to `Errors`, asserts `HasErrors == true`; removes it, asserts `HasErrors == false` — AC-8
- Updated: `MysqlDumpBackupServiceTests.BuildStartInfo_PopulatesExpectedArguments` — updated to pass `credentialsFilePath` and assert new expected order — AC-9


## Documentation Updates

- `docs/agents/project-manifest/constraints.md` — add view lifecycle template rule (WP-007a); add INNER JOIN aggregation rule (WP-007b); revise/remove open-item note about missing `SettingsViewModel` tests (WP-007d); update the `MysqlDumpBackupService` ArgumentList bullet to replace the `-p{password}` entry description with `--defaults-extra-file=<temppath>` and add a sentence noting that the password is now written to a temporary `.cnf` file rather than passed as a process argument (WP-007e)
- `docs/agents/project-manifest/api-surface.md` — add `CancelBackupCommand` to `DatabaseBackupViewModel`; add `HasErrors` to `ObfuscationSettingsViewModel`; update `BuildStartInfo` to 4-parameter signature; add `SanitizeDatabaseName` entry (WP-007c)
- `docs/agents/project-manifest/file-tree.md` — add entry for `TestHelpers/FakeObfuscationMap.cs` (WP-001)


## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **`--defaults-extra-file` must be first argument** — MySQL versions differ on argument ordering requirements | Place `--defaults-extra-file` as the first `ArgumentList` entry in `BuildStartInfo`; add a unit test asserting `ArgumentList[0]` starts with `--defaults-extra-file` |
| **Temp file not deleted on process failure, cancellation, or exception** | Initialize `credentialsFilePath = null` before the outer `try`; write the file as the first statement inside `try`; use `if (credentialsFilePath is not null && File.Exists(credentialsFilePath)) File.Delete(credentialsFilePath)` in `finally`. The null-check ensures the `finally` is a no-op if the write was never reached. |
| **`CanExecute` change silently breaks existing tests that call `BackupCommand` without setting `DestinationFolder`** | Audit `DatabaseBackupViewModelTests.cs` before the change; ensure all existing tests that call `BackupCommand.ExecuteAsync` first set a non-empty `DestinationFolder` |
| **`IncludeCancelCommand = true` changes generated code; tests expecting `IsBusy` timing may need adjustment** | Review `SettingsViewModelTests` and `DatabaseBackupViewModelTests` for IsBusy assertions; the generated cancel command does not change the `IsBusy` observable behaviour |
| **`HasErrors` property on `ObfuscationSettingsViewModel` requires `CollectionChanged` subscription** — forgetting to raise `OnPropertyChanged` after collection mutation would make the banner never appear | Follow the `TaggerViewModel.RebuildCategories` pattern (existing); `HasErrors_IsFalse_WhenErrorsIsEmpty` and `HasErrors_IsTrue_WhenErrorsContainsEntry` in `ObfuscationSettingsViewModelTests.cs` verify the transitions (see Test Plan) |
| **Compiled bindings on `HasErrors`** — if `ObfuscationSettingsViewModel` type annotation is missing on the Obfuscation tab, the binding will fail at build time | Ensure `x:DataType` is set on the parent `UserControl` (already `vm:SettingsViewModel`); `HasErrors` is accessed via `ObfuscationVm.HasErrors` which is typed through the parent — compiled binding chain is resolvable |
