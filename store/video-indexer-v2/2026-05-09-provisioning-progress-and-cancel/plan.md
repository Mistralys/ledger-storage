# Plan

## Summary

The "Setting up external tools" screen has two distinct categories of defect:
**First**, the provisioning process never actually starts — `ProvisioningToolsViewModel.StartAsync()` is never called from anywhere, so the progress bar spins indefinitely in indeterminate mode without any real work happening behind it.
**Second**, even when it does start, the UI gives no meaningful feedback: the `Message` field carried by every `ProvisioningProgress` snapshot is never surfaced, there is no running log, and there is no quantitative download progress shown during the download phase.
Finally, the Cancel button is a no-op in the current production wiring because `_cts` is only created inside `RunProvisioningAsync()`, which is never invoked.

This plan fixes the startup trigger, completes the shell-navigation callback that transitions the app forward after provisioning succeeds, surfaces `Message` detail and a scrollable running log in the view, and adds appropriate unit-test coverage.

---

## Architectural Context

```
src/
  VideoIndexer.App/
    App.axaml.cs                      — OnFrameworkInitializationCompleted: sets MainWindow only
    Program.cs                        — DI wiring; ShellViewModel factory; lazyShell pattern
    ViewModels/
      ShellViewModel.cs               — State machine; ProvisionAsync() passes null for progress
      ProvisioningToolsViewModel.cs   — Orchestrates provisioning, has StartAsync(); NEVER CALLED
      MainWindowViewModel.cs
    Views/
      ProvisioningToolsView.axaml     — Binds Stage, progress bar, Cancel/Retry/Provide buttons
      ProvisioningToolsView.axaml.cs  — Parameterless + DI constructors; NO OnLoaded hook
  VideoIndexer.Core/
    Models/ProvisioningProgress.cs    — Stage, BytesReceived, BytesTotal, Message
  VideoIndexer.Infrastructure/
    ExternalTools/
      FfmpegProvisioner.cs            — 13-step provisioner; reports Stage + Message but no steps
      HttpToolDownloader.cs           — Reports BytesReceived/BytesTotal per 64 KB chunk
tests/
  VideoIndexer.App.Tests/
    ProvisioningToolsViewModelTests.cs  — Tests StartAsync, Cancel, Retry
    ShellViewModelTests.cs              — Tests ProvisionAsync (which bypasses the VM)
```

### Root-cause analysis

| # | Symptom | Root Cause |
|---|---------|------------|
| 1 | Spinner runs forever | `ProvisioningToolsViewModel.StartAsync()` is never called; `IsProgressIndeterminate` defaults to `true` |
| 2 | Cancel button does nothing | `_cts` is only created inside `RunProvisioningAsync()`, which is never invoked |
| 3 | No detail text | `ProvisioningProgress.Message` is never mapped to a VM property or bound in the view |
| 4 | No running log | No log collection exists in the VM |
| 5 | Indeterminate outside download | Non-download stages (Verifying, Extracting) report `BytesTotal = null`, keeping bar indeterminate |
| 6 | No forward navigation | `ProvisioningToolsViewModel` has no reference to `ShellViewModel`; there is no callback wired to advance to `ShellState.Connecting` on success |

---

## Approach / Architecture

### A — Startup trigger (View → VM)

Add an `OnLoaded` handler in `ProvisioningToolsView.axaml.cs` that casts `DataContext` to `ProvisioningToolsViewModel` and calls `StartAsync()`. Using the `Loaded` event (rather than `OnDataContextChanged`) is idiomatic for Avalonia — the DataContext is set before `Loaded` fires, and the call is safe because `RunProvisioningAsync` guards against a second in-flight call via `IsProvisioning`.

### B — Shell navigation on completion (VM → Shell callback)

Add an optional `Action? onComplete` constructor parameter to `ProvisioningToolsViewModel`. At the end of a successful `RunProvisioningAsync()` call (after `Stage = "Complete"`), invoke `onComplete?.Invoke()`. The caller is responsible for triggering the shell transition.

In `Program.cs`, wire this inside the `ProvisioningTools` factory branch:

```csharp
ShellState.ProvisioningTools => new ProvisioningToolsViewModel(
    sp.GetRequiredService<IExternalToolProvisioner>(),
    onComplete: () => lazyShell!.AdvanceToConnecting()),
```

Because `onComplete` is invoked asynchronously (after provisioning finishes), `lazyShell` is guaranteed to be non-null by the time the callback fires — it is assigned synchronously immediately after the shell constructor returns.

`ShellViewModel.AdvanceToConnecting()` is a new thin public method that wraps the private `Transition(ShellState.Connecting)` call. This avoids giving the VM any broader shell reference.

> **Note on `ShellViewModel.ProvisionAsync()`**: This method is left intact to preserve existing unit-test coverage. It becomes dead code in the production path but continues to serve as a contract-level integration test of the shell's transition logic.

### C — Progress detail: Message property

Add a `Message` observable property to `ProvisioningToolsViewModel`, populated from `ProvisioningProgress.Message` in the progress callback. Bind it in `ProvisioningToolsView.axaml` as a subtitle beneath `Stage`, visible only when non-empty.

### D — Running log

Add an `ObservableCollection<string> Log` property to `ProvisioningToolsViewModel`. Every time the progress callback fires with a non-empty `Message`, append a timestamped entry (e.g., `"14:51:02 — Downloading FFmpeg (7.1, win-x64)…"`). On `Stage = "Complete"` or `Stage = "Cancelled"`, append a final entry.

Bind this in the view as a read-only, auto-scrolling `ItemsControl` (or `ListBox` with `SelectionMode="None"`) with a fixed maximum height.

> **Thread-safety note:** The `SyncProgress<T>` callback fires on the ThreadPool thread (after `ConfigureAwait(false)`). Avalonia 11 auto-marshals scalar `PropertyChanged` events but does not guarantee auto-marshaling of `INotifyCollectionChanged` from non-UI threads. Collection mutations (`Log.Add(...)`) must therefore be dispatched to the UI thread explicitly via `Dispatcher.UIThread.Post(() => Log.Add(entry))`. This is the only exception to the no-`Dispatcher` constraint; scalar property assignments are unaffected.

### E — Determinate progress during Verifying / Extracting

For the Verifying and Extracting stages, `BytesTotal` is always null, keeping the bar indeterminate. These stages are not long-running enough to warrant a step counter, but we can improve the UX by: showing the stage name prominently and resetting `BytesReceived`/`BytesTotal` to zero/null at stage boundaries. No change to `ProvisioningProgress` model is required.

Optionally — if step tracking is desired — add `StepNumber` (int) and `TotalSteps` (int) properties to `ProvisioningProgress` and have `FfmpegProvisioner` populate them. This is listed as a stretch goal below.

### F — Percentage label during download

Add a computed `DownloadPercent` string property to the VM (e.g., `"47 %"`, or `string.Empty` when `BytesTotal` is null). Bind it as a small label inside the progress bar row.

---

## Rationale

- Using a constructor callback for shell navigation keeps `ProvisioningToolsViewModel` free of any direct reference to `ShellViewModel`. This is a deliberate departure from the direct-reference pattern used by `DatabaseConnectorViewModel` and `LogonViewModel` (which accept `ShellViewModel` as a constructor parameter); the callback achieves the same navigation goal while keeping the provisioning VM fully decoupled from the shell.
- Using `View.Loaded` as the startup trigger is the simplest approach and avoids adding lifecycle complexity to the VM.
- The existing `SyncProgress<T>` pattern is preserved — no `Dispatcher.UIThread.Post` needed.
- Keeping `ShellViewModel.ProvisionAsync()` intact avoids breaking existing shell tests.

---

## Detailed Steps

1. **`ProvisioningToolsViewModel` — add `onComplete` callback**
   - Add `Action? _onComplete` field; accept it as an optional constructor parameter.
   - Invoke `_onComplete?.Invoke()` immediately after `Stage = "Complete"` in `RunProvisioningAsync()`.
   - Existing single-arg constructor `ProvisioningToolsViewModel(IExternalToolProvisioner)` stays valid for unit tests (callback defaults to null).

2. **`ProvisioningToolsViewModel` — add `Message` observable property**
   - Add `[ObservableProperty] private string? _message`.
   - Populate from `p.Message` inside the `SyncProgress<ProvisioningProgress>` callback.

3. **`ProvisioningToolsViewModel` — add `Log` collection**
   - Add `public ObservableCollection<string> Log { get; } = new()`.
   - In the progress callback, if `p.Message` is non-null and `Log.Count < 200`, dispatch to the UI thread: `Dispatcher.UIThread.Post(() => Log.Add($"{DateTime.Now:HH:mm:ss} — {p.Message}"))`. If `Log.Count` has already reached 200, skip the append (cap enforced before dispatching).
   - In the `finally` block, dispatch a terminal entry to the UI thread: `Dispatcher.UIThread.Post(() => Log.Add($"{DateTime.Now:HH:mm:ss} — {(Error is not null ? "Failed." : Stage + ".")}"))`. This correctly distinguishes `"Complete."`, `"Cancelled."`, and `"Failed."` across all three exit paths.
   - **Note:** All `Log` mutations must use `Dispatcher.UIThread.Post` — `INotifyCollectionChanged` is not auto-marshaled by Avalonia 11 from non-UI threads, unlike scalar `PropertyChanged`. This is the only place in this plan where `Dispatcher.UIThread.Post` is required.

4. **`ProvisioningToolsViewModel` — add `DownloadPercent` computed property**
   - Add `public string DownloadPercent => BytesTotal.HasValue && BytesTotal.Value > 0 ? $"{BytesReceived * 100 / BytesTotal.Value} %" : string.Empty;`
   - Add `[NotifyPropertyChangedFor(nameof(DownloadPercent))]` on both `_bytesReceived` and `_bytesTotal`.

5. **`ShellViewModel` — add `AdvanceToConnecting()` method**
   - Add `public void AdvanceToConnecting() => Transition(ShellState.Connecting);` as a thin public façade.

6. **`Program.cs` — wire up callback**
   - Change the `ProvisioningTools` factory arm to use the two-argument constructor:
     ```csharp
     ShellState.ProvisioningTools => new ProvisioningToolsViewModel(
         sp.GetRequiredService<IExternalToolProvisioner>(),
         onComplete: () =>
         {
             Debug.Assert(lazyShell is not null, "lazyShell must be non-null before onComplete fires.");
             lazyShell!.AdvanceToConnecting();
         }),
     ```
   - Remove the `builder.Services.AddTransient<ProvisioningToolsViewModel>()` registration (the VM is now constructed manually in the factory, not resolved from DI).

7. **`ProvisioningToolsView.axaml.cs` — add `Loaded` trigger**
   - In the parameterless constructor, subscribe to `Loaded`:
     ```csharp
     Loaded += (_, _) => (DataContext as ProvisioningToolsViewModel)?.StartAsync();
     ```
   - No change to the DI constructor.

8. **`ProvisioningToolsView.axaml` — update view layout**
   - Add a `TextBlock` for `Message` beneath the `Stage` label (visible when `Message` is non-null/non-empty).
   - Add a `TextBlock` for `DownloadPercent` aligned to the right of the progress bar row.
   - Add a bordered `ScrollViewer` containing an `ItemsControl` bound to `Log`, with `MaxHeight="200"` and auto-scroll behaviour (via a Behavior or code-behind scroll-to-bottom logic).

9. **Tests — `ProvisioningToolsViewModelTests`**
   - Add test: `StartAsync_OnSuccess_InvokesOnCompleteCallback` — verifies the callback fires.
   - Add test: `StartAsync_OnSuccess_AppendsLogEntries` — verifies `Log` is populated.
   - Add test: `StartAsync_OnSuccess_SetsMessage` — verifies `Message` reflects last progress message.
   - Add test: `StartAsync_WhenCancelled_DoesNotInvokeOnCompleteCallback`.
   - Add test: `StartAsync_WhenProvisionerThrows_DoesNotInvokeOnCompleteCallback`.

10. **Tests — `ShellViewModelTests`**
    - Add test: `AdvanceToConnecting_TransitionsToConnecting` — verifies the new public method.

---

## Dependencies

- `CommunityToolkit.Mvvm` (already referenced) — no new packages.
- `System.Collections.ObjectModel.ObservableCollection<T>` — part of the BCL.

---

## Required Components

| Component | Action |
|-----------|--------|
| `src/VideoIndexer.App/ViewModels/ProvisioningToolsViewModel.cs` | Modify — add callback, Message, Log, DownloadPercent |
| `src/VideoIndexer.App/ViewModels/ShellViewModel.cs` | Modify — add `AdvanceToConnecting()` |
| `src/VideoIndexer.App/Views/ProvisioningToolsView.axaml` | Modify — add Message label, DownloadPercent label, Log panel |
| `src/VideoIndexer.App/Views/ProvisioningToolsView.axaml.cs` | Modify — add `Loaded` trigger |
| `src/VideoIndexer.App/Program.cs` | Modify — wire `onComplete` callback; remove AddTransient for ProvisioningToolsViewModel |
| `tests/VideoIndexer.App.Tests/ProvisioningToolsViewModelTests.cs` | Modify — add new AC tests |
| `tests/VideoIndexer.App.Tests/ShellViewModelTests.cs` | Modify — add AdvanceToConnecting test |

---

## Assumptions

- Auto-scroll on the log panel is implemented via code-behind (subscribing to `Log.CollectionChanged` and calling `ScrollViewer.ScrollToEnd()`) rather than a third-party behaviour library, keeping dependencies minimal.
- Cancellation after completion (user clicks Cancel while Stage is "Complete") is a no-op — `_cts` has already been disposed or is already cancelled.
- The `ProvisioningTools` factory arm in `Program.cs` is only called once per application launch (no re-entry into `ProvisioningTools` state after it transitions away), so removing the `AddTransient` registration is safe.

---

## Constraints

- No new NuGet packages.
- `ShellViewModel.ProvisionAsync()` must remain public and functional (existing tests depend on it).
- The `SyncProgress<T>` inner class must not be changed to `System.Progress<T>` (thread-scheduling implications for tests).
- Scalar `[ObservableProperty]` changes (`Stage`, `BytesReceived`, `BytesTotal`, `Message`, `Error`, `IsProvisioning`) must NOT use `Dispatcher.UIThread.Post` — Avalonia 11 marshals `PropertyChanged` internally for these.
- `ObservableCollection<string>` mutations (`Log.Add`) are the sole exception: they MUST be dispatched via `Dispatcher.UIThread.Post` because `INotifyCollectionChanged` is not auto-marshaled by Avalonia 11 from non-UI threads (see Step 3).

---

## Out of Scope

- Step-number tracking (`StepNumber`/`TotalSteps` fields on `ProvisioningProgress` and corresponding `FfmpegProvisioner` changes) — listed as a future stretch goal.
- The `ProvideManually` command implementation — already explicitly deferred.
- Download resume / `.partial` directory reuse on retry.
- Accessibility (screen-reader) annotations on the new log panel.

---

## Acceptance Criteria

- When the app starts, provisioning begins automatically within 500 ms of the "Setting up external tools" screen being shown (observable via the Stage label changing from empty string to a stage name).
- The Cancel button cancels the in-flight provisioning operation; `Stage` transitions to "Cancelled" and the Retry button appears.
- During the Download phase, the progress bar shows a determinate fill reflecting bytes received vs total, and a percentage label is visible.
- During Verifying and Extracting phases, the progress bar is indeterminate but the Stage label clearly indicates which phase is active, and the Message label shows the detailed step description.
- Every progress update appends a timestamped entry to the Log panel.
- After successful provisioning, the app automatically transitions to the `Connecting` (database) screen without user interaction.
- All existing unit tests continue to pass without modification.

---

## Testing Strategy

- **Unit**: New `ProvisioningToolsViewModelTests` cases exercise the callback, log population, Message propagation, and the cancel-does-not-call-callback invariant using the existing fake provisioner pattern.
- **Unit**: New `ShellViewModelTests` case for `AdvanceToConnecting()`.
- **Unit (cross-thread)**: Add a `ThreadPoolProgressProvisioner` fake that uses `await Task.Yield()` before reporting progress, ensuring the callback fires on a ThreadPool thread. Run the existing log-population and Message tests through this provisioner to validate that `Log` and `Message` are updated correctly across thread boundaries without an `InvalidOperationException`.
- **Manual smoke**: Run the application against a clean environment (no cached FFmpeg binaries) and verify all stages are visible, log scrolls, and the app advances automatically.
- **Manual cancel**: Run against a slow/unreachable URL, click Cancel, verify Stage becomes "Cancelled" and Retry works.

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **`lazyShell` is null when `onComplete` fires** | `onComplete` executes asynchronously on a ThreadPool thread, well after `lazyShell = shell` is assigned synchronously. Add a null-guard assertion in debug builds. |
| **`Loaded` fires before `DataContext` is set** | Avalonia guarantees `DataContext` is set before `Loaded` in a `ContentPresenter` binding scenario. If this assumption is violated (e.g., in a designer host), the null-conditional cast `(DataContext as ProvisioningToolsViewModel)?` safely does nothing. |
| **Log panel growing unbounded** | Cap `Log` at 200 entries in the progress callback before appending. |
| **Removing `AddTransient<ProvisioningToolsViewModel>()` breaks DI resolution elsewhere** | Grep for all `GetRequiredService<ProvisioningToolsViewModel>()` usages (currently only one, in the old `ProvisioningTools` factory arm). Remove that call when wiring the new manual constructor. |
| **`Log.Add()` from a ThreadPool thread throws `InvalidOperationException`** | Avalonia 11 does not auto-marshal `INotifyCollectionChanged` from non-UI threads. All `Log` mutations must be wrapped in `Dispatcher.UIThread.Post`. Validated by the cross-thread unit test added in the Testing Strategy. |
