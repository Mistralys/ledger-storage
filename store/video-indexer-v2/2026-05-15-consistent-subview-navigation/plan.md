# Plan

## Summary

The navigation specification (`docs/projects/rebuild/navigation-specification.md`) defines a complete navigation architecture: a collapsible FluentAvalonia `NavigationView` rail for primary destinations, a workspace `ContentControl` backed by a navigation stack with breadcrumb projection, and a standardised `PageHeaderView` embedded in every routable screen. The original tactical plan (dual context-sensitive toolbars + `LibraryFolders.DoneCommand`) is superseded by this specification. This plan implements the full navigation architecture, which also resolves the original problem: users can no longer be stranded in `LibraryFoldersView` because Library Folders becomes a first-class primary rail destination rather than a sub-view of the Movies area.


## Architectural Context

`MainContentView.axaml` is a three-row `Grid`: row 0 holds a persistent horizontal toolbar, row 1 holds the error banner, row 2 holds a `ContentControl` bound to `MainContentViewModel.CurrentChildViewModel`. The toolbar is always fully visible regardless of which area is active.

`MainContentViewModel` drives navigation through a `MainContentPage` enum (`Default`, `LibraryFolders`, `RefreshOverlay`, `MovieEditor`) and directly sets `CurrentChildViewModel`. Library Folders and Refresh are treated as sub-views of the Movies area, reached by toolbar buttons; `MovieEditorView` is a further sub-view opened on row selection.

`MovieEditorViewModel` raises `CloseRequested`, subscribed by `MainContentViewModel`, which calls `CloseSubView()` to return to the movies list. `LibraryFoldersViewModel` has no exit mechanism of its own — users are stranded without restarting.

The `MainContentPage` enum and manual `CurrentChildViewModel` assignment are the entirety of the current navigation model. There is no navigation stack, no breadcrumb, and no shared page-header component.

Key files in scope:
- `src/VideoIndexer.App/ViewModels/MainContentViewModel.cs`
- `src/VideoIndexer.App/ViewModels/MainContentPage.cs`
- `src/VideoIndexer.App/ViewModels/LibraryFoldersViewModel.cs`
- `src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs`
- `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs`
- `src/VideoIndexer.App/Views/MainContentView.axaml`
- `src/VideoIndexer.App/Views/LibraryFoldersView.axaml`
- `src/VideoIndexer.App/Views/MovieEditorView.axaml`
- `src/VideoIndexer.App/Views/RefreshIndexView.axaml`
- `src/VideoIndexer.App/Views/MoviesListView.axaml`
- `src/VideoIndexer.App/Program.cs`
- `tests/VideoIndexer.App.Tests/MainContentViewModelTests.cs`


## Approach / Architecture

The implementation follows the navigation specification's recommended shape verbatim (section 5):

### Two Zones

**Primary zone — collapsible rail.** `MainContentView.axaml` is restructured around a FluentAvalonia `NavigationView` that lists primary destinations in a collapsible side rail. The current primary destinations are: **Movies**, **Library Folders**, **Refresh**, and two stubs (**Filters**, **Settings**) whose views are not yet implemented. Selecting a rail item clears the workspace stack and pushes the root view-model for that area.

**Workspace zone — navigation stack.** The content surface to the right of the rail is driven by a new `INavigationService`. It maintains a stack of `NavigationEntry` objects (view-model + title + optional parameter). `ContentControl.Content` is bound to `INavigationService.Current`, which is proxied by `MainContentViewModel.CurrentChildViewModel` (preserving the existing compiled binding in the AXAML). The `ViewLocator` continues to resolve views from the VM type — no `FluentAvalonia.Frame` is used (per spec section 5).

### Navigation Stack Semantics

- `INavigationService.NavigateToRoot(vm, title)` — clears the stack and sets the root entry. Called when a primary destination is selected.
- `INavigationService.NavigateTo(vm, title)` — pushes a new entry. Called when opening Movie Editor from the Movies list.
- `INavigationService.GoBack()` — pops the top entry. Called by the Back button in `PageHeaderView`.
- View-model lifetime: **fresh on push, restored on pop**. The previous root VM instance is preserved when Back pops a drill-down entry.

### PageHeaderView

A new `PageHeaderView.axaml` user control is embedded at the top of every routable view. It exposes the following Avalonia `StyledProperty` entries that the host view sets in AXAML:
- `Title` (`string`) — the page title shown in the header.
- `BackCommand` (`ICommand?`) — the command executed by the Back button. Defaults to `null`; when `null`, the Back button invokes `INavigationService.GoBack()` directly via `App.Services`. Movie Editor sets it to `MovieEditorViewModel.CloseCommand` to route through the existing cleanup/guard path. Root views do not set it.
- `PrimaryAction` (`object?`) — content slot for at most one accent-styled primary button; rendered via `ContentPresenter`.
- `SecondaryActions` (`object?`) — content slot for 0–N standard buttons; rendered via `ContentPresenter`.

**Layout follows spec section 3:** Left zone — FluentAvalonia `BreadcrumbBar` and the Back button (both left-anchored); Centre zone — optional title `TextBlock` (hidden when `Title` is `string.Empty`, matching the spec's guidance that the title may be omitted when the breadcrumb terminal item already conveys it); Right zone — `PrimaryAction` and `SecondaryActions` content slots.

**`IsBackVisible` is not a host-configured property.** `PageHeaderView.axaml.cs` resolves `INavigationService` from `App.Services` at construction, stores it as `_navService`, and exposes `IsBackVisible { get => _navService.CanGoBack; }` as a plain CLR property. The Back button's `IsVisible` binds to this property via `RelativeSource Self`. The code-behind subscribes to `_navService.NavigationChanged` and raises `PropertyChanged(nameof(IsBackVisible))` on each event so the binding refreshes.

`PageHeaderView` contains a FluentAvalonia `BreadcrumbBar` on the LEFT (alongside the Back button), bound to `INavigationService.BreadcrumbItems` (resolved via `App.Services`). `PageHeaderView.axaml.cs` subscribes to `BreadcrumbBar.ItemClicked` and calls `_navService.GoBackTo(item.StackIndex)` — after first checking `BackCommand?.CanExecute(null) != false` so that breadcrumb clicks are suppressed when the Back button's command guard is active (consistent with the Back button disable). This avoids breaking `x:DataType` compiled bindings on the host view.

### Primary Navigation Service

A new `IPrimaryNavigationService` in `src/VideoIndexer.App/Services/Navigation/` exposes:
- `IReadOnlyList<PrimaryDestination> Destinations` — registered primary destinations.
- `PrimaryDestination? ActiveDestination` — currently selected rail item.
- `void NavigateTo(string destinationId)` — selects a destination and fires `ActiveDestinationChanged`.
- `event EventHandler<PrimaryDestination?>? ActiveDestinationChanged`

`MainContentViewModel` subscribes to `ActiveDestinationChanged` and calls `_navigationService.NavigateToRoot(dest.RootViewModel, dest.Title)` in response, keeping the two services loosely coupled.

### Workspace Navigation Service

`INavigationService` in the same directory exposes:
- `object? Current` — top-of-stack VM; `null` when stack is empty.
- `IReadOnlyList<BreadcrumbItem> BreadcrumbItems` — read-only stack projection.
- `bool CanGoBack` — `true` when stack depth > 1.
- `void NavigateToRoot(object vm, string title)` — clears stack and pushes root.
- `void NavigateTo(object vm, string title)` — pushes entry onto stack.
- `void GoBack()` — pops the topmost entry.
- `event EventHandler? NavigationChanged` — fired on every stack change.

### MovieEditor Close Handshake (preserved)

`MovieEditorViewModel.CloseRequested` is kept as the exit signal — this event is subscribed by `MainContentViewModel.OnEditorCloseRequested`, which calls `_navigationService.GoBack()` followed by `MoviesListVm.LoadAsync`. This preserves the existing cleanup responsibility in `MainContentViewModel` and avoids injecting `INavigationService` into `MovieEditorViewModel`.

`PageHeaderView.BackCommand` for `MovieEditorView` is bound to `MovieEditorViewModel.CloseCommand` (not to a generic GoBack). This routes Back through the existing CloseCommand → CloseRequested → `OnEditorCloseRequested` path, preserving all cleanup logic and the `HasChanges`-based `CanExecute` guard.

### Auto-navigation for Refresh

`MainContentViewModel.OnRefreshStateChanged` continues to auto-navigate when the orchestrator transitions to `Running`. It calls `_primaryNavService.NavigateTo("refresh")`, which fires `ActiveDestinationChanged`, which triggers `NavigateToRoot(RefreshIndex, "Refresh")`. On Completed/Faulted it calls `_primaryNavService.NavigateTo("movies")` and reloads the movies list.

### IndexedMovieCount

The indexed-count display moves from the old toolbar into `MoviesListView`'s `PageHeaderView.SecondaryActions` slot, bound to `MoviesListViewModel.IndexedMovieCount` via the view's existing `x:DataType="vm:MoviesListViewModel"` compiled binding. Count ownership moves fully into `MoviesListViewModel`: `LoadAsync` loads the count via `IMovieCatalogRepository` and updates `IndexedMovieCount` as part of the same database round-trip. `MainContentViewModel.RefreshCountAsync` and `_indexedMovieCount` are deleted. The count stays current because `MoviesListVm.LoadAsync` is already called at every point that previously called `RefreshCountAsync` (orchestrator Completed/Faulted, editor close, initial activation).

### MainContentPage Enum

`MainContentPage.cs` is deleted. All navigation state is held in `INavigationService` (workspace stack) and `IPrimaryNavigationService` (active rail destination). `MainContentViewModel` no longer needs the enum.

### Navigation Lifecycle Hooks (Deferred)

The navigation specification (section 4) defines optional `OnNavigatedTo(parameter)` and `OnNavigatedFrom()` lifecycle hooks. These are deferred. The Library Folders folder-list load is handled explicitly in `OnPrimaryDestinationChanged` (Step 9) rather than via a generic lifecycle hook, because that is the only VM currently requiring navigation-entry notification. General-purpose `OnNavigatedTo`/`OnNavigatedFrom` hooks are deferred until a second VM requires them. The `INavigationService` interface leaves room to add lifecycle callback invocations alongside `NavigateTo`/`GoBack` without breaking callers.


## Rationale

**Why implement the full navigation spec instead of the dual-toolbar patch?** The patch addressed the symptom (no exit from Library Folders) but produced a new inconsistency: a bespoke context-sensitive toolbar that must be maintained separately from every screen that adds a new sub-view. The navigation spec solves the structural problem once and provides a framework that scales to Settings, Filters, and future areas without per-screen toolbar surgery.

**Why FluentAvalonia `NavigationView` and `BreadcrumbBar`?** The spec explicitly names these. Both components are purpose-built for the navigation pattern described, they match the Fluent design idiom already in use, and they are the only new dependency introduced. The alternative — hand-rolling a rail and breadcrumb over the Fluent theme — is more work for the same visual result.

**Why keep `CloseRequested` on `MovieEditorViewModel` rather than injecting `INavigationService`?** The existing event-handshake pattern is already tested, the cleanup logic (LoadAsync after close) belongs in `MainContentViewModel`, and injecting `INavigationService` into `MovieEditorViewModel` would require updating the `Func<Movie, MovieEditorViewModel>` factory and all existing tests that construct the editor. Preserving the event produces less churn for equivalent correctness.

**Why proxy `INavigationService.Current` through `MainContentViewModel.CurrentChildViewModel`?** The AXAML binding `{Binding CurrentChildViewModel}` is compiled; re-wiring it to `{Binding NavigationService.Current}` would require either exposing `INavigationService` directly from the VM (adding a public dependency that is not a domain concern) or opting out of compiled bindings for that binding. Updating `CurrentChildViewModel` from a `NavigationChanged` event keeps the AXAML unchanged and the VM cohesive.


## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|---|---|---|---|
| Adopt full navigation spec vs. targeted toolbar patch | Full spec implementation | Dual context-sensitive StackPanels (original plan) | Patch fixes one symptom; spec fixes the structural problem and scales to future areas. The patch accumulates technical debt; the spec pays it off now. |
| Primary navigation chrome | FluentAvalonia `NavigationView` | Hand-rolled rail over Avalonia Fluent theme | `NavigationView` is purpose-built, already matches the design idiom, and brings `BreadcrumbBar` for free. Hand-rolling is additional styling work for identical visual output. |
| Navigation stack driver | Custom `INavigationService` + `IPrimaryNavigationService` | ReactiveUI `RoutingState`; Prism.Avalonia regions | ReactiveUI adds a second MVVM framework alongside CommunityToolkit.Mvvm. Prism is disproportionate for a single-window utility app. Custom services are small, testable, and own no MVVM opinions. |
| FluentAvalonia Frame vs. ContentControl | `ContentControl` bound to `INavigationService.Current` | `FluentAvalonia.Frame` | `Frame` imposes its own navigation stack, conflicting with the custom `INavigationService`. `ContentControl` + `ViewLocator` is the existing convention; no new pattern required. |
| MovieEditor Back mechanism | `PageHeaderView.BackCommand` bound to `MovieEditorViewModel.CloseCommand` | Inject `INavigationService` into `MovieEditorViewModel`; expose `GoBackCommand` relay on `MainContentViewModel` | Binding to CloseCommand reuses the proven event-handshake pattern with zero constructor churn. Injection adds factory and test complexity. Host relay adds indirection. |
| Unsaved-change guard for Movie Editor back | Back button disabled when `HasChanges == true` | Confirmation dialog on Back | A dialog is the spec's ideal (section 4.1) but requires a general-purpose confirmation dialog service not yet present. Disabling the button enforces the guard mechanically with no new service needed; dialog can be added later. This is explicitly called out as a spec departure. |
| Unsaved-change guard for primary-zone (rail) switch | Unguarded — documented spec departure | Intercept `SelectionChanged`, check `HasChanges`, block selection or show dialog | Intercepting requires a navigation-cancellation API (e.g., `INavigationGuard`) or blocking a `NavigationView` selection event — neither is trivial and both need the same `IConfirmationDialogService` that is absent. The departure is the same root cause as the Back-button dialog departure and will be addressed together in a future iteration. |
| `INavigationService.Current` surfacing in AXAML | Proxy via `MainContentViewModel.CurrentChildViewModel` | Expose `INavigationService` as a public property of `MainContentViewModel` and bind via `{Binding NavigationService.Current}` | Public property approach breaks the single-DataContext-per-view convention and exposes an implementation detail. Proxy keeps AXAML unchanged and the compiled binding intact. |
| `PageHeaderView` breadcrumb data source | Resolved from `App.Services` static provider inside `PageHeaderView.axaml.cs` | Passed as `StyledProperty<INavigationService>` from each host view | The static provider is an established escape hatch already used by `ViewLocator`. Passing the service as a styled property requires every host view to wire it, adding boilerplate per routable view. |
| `PageHeaderView` Back button visibility | Automatically derived from `INavigationService.CanGoBack` (plain CLR property, no styled property) | `IsBackVisibleProperty : StyledProperty<bool>` set by each host view | Automatic derivation is verbatim from spec section 3 ("Back is visible whenever the workspace stack has more than one entry"), removes per-view boilerplate, and prevents accidental misconfiguration. A per-view styled property adds complexity while the navigation service already owns the authoritative state. |


## Pattern Alignment

| Pattern | Status |
|---|---|
| `ViewLocator` convention-based VM→View resolution | **Followed** — navigation stack uses `ContentControl.Content = INavigationService.Current`; `ViewLocator` resolves the view as before. No `FluentAvalonia.Frame`. |
| Compiled bindings (`x:DataType`) in AXAML | **Followed** — `MainContentView.axaml` retains `x:DataType="vm:MainContentViewModel"`; all new bindings in that file are to properties of `MainContentViewModel`. `PageHeaderView.axaml` uses `x:DataType="local:PageHeaderView"` for its own StyledProperties. |
| `[ObservableProperty]` + `[NotifyPropertyChangedFor]` source generation | **Followed** — new observable properties on `MainContentViewModel` use the same pattern. |
| `MovieEditorViewModel.CloseRequested` event → `MainContentViewModel` subscription | **Followed** — event-handshake is preserved. `MainContentViewModel.OnEditorCloseRequested` now calls `_navigationService.GoBack()` instead of `CloseSubView()`. |
| `Dispose()` unsubscribes events | **Followed** — `MainContentViewModel.Dispose()` unsubscribes from `_orchestrator.StateChanged`, `moviesListVm.EditMovieRequested`, editor `CloseRequested` (if active), and `_navigationService.NavigationChanged` / `_primaryNavService.ActiveDestinationChanged`. |
| `App.Services` static bridge for non-injected resolution | **Followed** — `PageHeaderView.axaml.cs` uses `App.Services.GetRequiredService<INavigationService>()` to resolve the breadcrumb source, following the same pattern as `ViewLocator`. |
| `MainContentPage` enum as navigation state | **Departed** — enum is deleted; navigation state lives in `INavigationService` + `IPrimaryNavigationService`. Tests that assert on `CurrentPage` are updated to assert on navigation service state or on `CurrentChildViewModel` type. |
| Sub-view VM knows nothing about `MainContentViewModel` | **Followed** — no VM references `MainContentViewModel` directly. |
| Interface-first cross-layer design | **Followed** — `INavigationService` and `IPrimaryNavigationService` are App-layer interfaces; `NavigationService` and `PrimaryNavigationService` are concrete implementations registered in `Program.cs`. No VM references a concrete navigation class. |


## Detailed Steps

### Step 1 — Add FluentAvalonia dependency
In `Directory.Packages.props`, add under the Avalonia UI `ItemGroup`:
```xml
<PackageVersion Include="FluentAvalonia" Version="2.1.0" />
```
> ⚠ Verify the latest FluentAvalonia 2.x release that is compatible with Avalonia 11.3.14 before committing the version number. FluentAvalonia 2.x targets Avalonia 11; minor version compatibility must be confirmed against the FluentAvalonia release notes.

In `src/VideoIndexer.App/VideoIndexer.App.csproj`, add:
```xml
<PackageReference Include="FluentAvalonia" />
```
In `App.axaml` (and/or `App.axaml.cs`), register the FluentAvalonia styles so that `NavigationView` and `BreadcrumbBar` receive their default templates.

---

### Step 2 — Create navigation model types
Create `src/VideoIndexer.App/Services/Navigation/NavigationEntry.cs`:
```csharp
sealed record NavigationEntry(object ViewModel, string Title, object? Parameter = null);
```

Create `src/VideoIndexer.App/Services/Navigation/BreadcrumbItem.cs`:
```csharp
sealed record BreadcrumbItem(string Title, int StackIndex);
```

Create `src/VideoIndexer.App/Services/Navigation/PrimaryDestination.cs`:
```csharp
sealed record PrimaryDestination(string Id, string Title, object RootViewModel);
```

---

### Step 3 — Create `INavigationService`
Create `src/VideoIndexer.App/Services/Navigation/INavigationService.cs`:
```csharp
public interface INavigationService
{
    object? Current { get; }
    IReadOnlyList<BreadcrumbItem> BreadcrumbItems { get; }
    bool CanGoBack { get; }
    void NavigateToRoot(object viewModel, string title);
    void NavigateTo(object viewModel, string title, object? parameter = null);
    void GoBack();
    void GoBackTo(int stackIndex);   // pops the stack down to the entry at stackIndex (0 = root)
    event EventHandler? NavigationChanged;
}
```

---

### Step 4 — Create `IPrimaryNavigationService`
Create `src/VideoIndexer.App/Services/Navigation/IPrimaryNavigationService.cs`:
```csharp
public interface IPrimaryNavigationService
{
    IReadOnlyList<PrimaryDestination> Destinations { get; }
    PrimaryDestination? ActiveDestination { get; }
    void Register(IReadOnlyList<PrimaryDestination> destinations);
    void NavigateTo(string destinationId);
    event EventHandler<PrimaryDestination?>? ActiveDestinationChanged;
}
```

---

### Step 5 — Create `NavigationService` implementation
Create `src/VideoIndexer.App/Services/Navigation/NavigationService.cs`. Use a `Stack<NavigationEntry>` internally. `NavigateToRoot` clears the stack and pushes. `NavigateTo` pushes. `GoBack` pops (guard: no-op if stack depth ≤ 1 to avoid emptying it entirely). `GoBackTo(int stackIndex)` removes all entries above `stackIndex` from the stack (guard: if `stackIndex >= stack depth - 1` it is a no-op; `stackIndex <= 0` navigates to root — the root entry is never removed). After each mutation, fire `NavigationChanged` and recompute `BreadcrumbItems` as a projection of the stack from bottom to top.

---

### Step 6 — Create `PrimaryNavigationService` implementation
Create `src/VideoIndexer.App/Services/Navigation/PrimaryNavigationService.cs`. Holds an `IReadOnlyList<PrimaryDestination>` set via `Register`. `NavigateTo(id)` finds the matching destination, sets `ActiveDestination`, and fires `ActiveDestinationChanged`. No-op if the requested destination is already active.

---

### Step 7 — Create `PageHeaderView`
Create `src/VideoIndexer.App/Views/PageHeaderView.axaml` and `PageHeaderView.axaml.cs`.

The control defines the following Avalonia `StyledProperty` entries (registered in code-behind):
- `TitleProperty : StyledProperty<string>` (default `string.Empty`)
- `BackCommandProperty : StyledProperty<ICommand?>` (default `null`)
- `PrimaryActionProperty : StyledProperty<object?>` (default `null`)
- `SecondaryActionsProperty : StyledProperty<object?>` (default `null`)

`IsBackVisible` is **not** a `StyledProperty`. The code-behind resolves `INavigationService` from `App.Services` at construction, stores it as `_navService`, and exposes `IsBackVisible { get => _navService.CanGoBack; }` as a plain CLR property. The code-behind subscribes to `_navService.NavigationChanged` and raises `PropertyChanged(nameof(IsBackVisible))` so the binding refreshes on every stack change.

AXAML layout (inside a horizontal `DockPanel` or `Grid`) — matching spec section 3:
- **Left:** FluentAvalonia `BreadcrumbBar` bound to breadcrumb items sourced from `INavigationService` (obtained via `App.Services` in code-behind, stored as `_navService`, exposed as a property `BreadcrumbItemsSource`), followed by a `Button` labelled "← Back" adjacent to it. The Back button's `IsVisible` is bound to `IsBackVisible` (plain CLR, reflective binding acceptable for a control's own property). Its `Command` binds to `BackCommand` via `RelativeSource Self`; when `BackCommand` is `null`, the code-behind substitutes a `RelayCommand` that calls `_navService.GoBack()`. `IsEnabled` is governed by `BackCommand.CanExecute` automatically. `BreadcrumbBar.ItemClicked` in code-behind calls `_navService.GoBackTo(item.StackIndex)` after checking `BackCommand?.CanExecute(null) != false` — click is suppressed when the Back guard is active.
- **Centre:** optional `TextBlock` bound to `Title`. Hidden when `Title` is `string.Empty` (the default) — this accommodates the spec's note that the title "may be omitted when the breadcrumb's terminal item already conveys it."
- **Right:** two `ContentPresenter` elements for `PrimaryAction` and `SecondaryActions` slots.

The `BackCommand.CanExecute` state automatically disables the Back button when the command's `CanExecute` returns `false` — no additional `IsEnabled` binding required.

---

### Step 8 — Restructure `MainContentView.axaml`
Replace the current three-row `Grid` with a two-column layout:
- Column 0: FluentAvalonia `NavigationView` (`IsPaneOpen="{Binding IsPaneOpen}"`) with named `NavigationViewItem` entries for Movies (`x:Name="NavMovies"`), Library Folders (`x:Name="NavLibraryFolders"`), Refresh (`x:Name="NavRefresh"`), Filters stub (`x:Name="NavFilters"`), Settings stub (`x:Name="NavSettings"`). **No `SelectedItem` AXAML binding is used.** Because inline `NavigationViewItem` declarations make `SelectedItem` resolve to a `NavigationViewItem` control reference rather than a data object, the programmatic selection highlight is driven from code-behind: `MainContentView.axaml.cs` subscribes to `IPrimaryNavigationService.ActiveDestinationChanged` and sets `IsSelected = true` on the matching named item, `IsSelected = false` on all others (spec section 5: "treat the navigation service as the single writer").
- Column 1: the existing workspace `ContentControl` bound to `{Binding CurrentChildViewModel}` (unchanged binding), preceded by the load-error `Border` (moved into this column's `Grid`).

The old toolbar `StackPanel` (Library Folders button, Refresh Index button, Search, Filter, Indexed count) is removed entirely. These controls migrate to their respective views:
- Search `TextBox` and Filter `ComboBox` move into `MoviesListView.axaml` (below the PageHeaderView).
- Indexed count moves into `MoviesListView`'s `PageHeaderView.SecondaryActions` slot.
- Refresh is now triggered from `RefreshIndexView` (Start Refresh button stays there) or auto-triggered by the orchestrator.

`MainContentView.axaml.cs` resolves `IPrimaryNavigationService` via `App.Services.GetRequiredService<IPrimaryNavigationService>()` in the constructor (after `InitializeComponent()`), following the same pattern as `ViewLocator.cs`. The code-behind contains two pieces of logic:
1. `NavigationView.SelectionChanged` handler — calls `_primaryNavService.NavigateTo(selectedDestinationId)` to propagate user rail clicks to the service.
2. `_primaryNavService.ActiveDestinationChanged` handler — sets `IsSelected = true` on the matching named `NavigationViewItem` (e.g. `NavMovies`, `NavLibraryFolders`) and `IsSelected = false` on all others, so that programmatic navigation (auto-navigate to Refresh, auto-return to Movies) is reflected in the rail highlight.

---

### Step 9 — Update `MainContentViewModel.cs`
**Constructor:** Accept `INavigationService navigationService` and `IPrimaryNavigationService primaryNavService` as new parameters. Remove the `_currentPage` field and all `MainContentPage`-based logic.

**Initialization:**
```csharp
_primaryNavService.Register(new[]
{
    new PrimaryDestination("movies",          "Movies",          moviesListVm),
    new PrimaryDestination("library-folders", "Library Folders", libraryFolders),
    new PrimaryDestination("refresh",         "Refresh",         refreshIndex),
    // Filters and Settings: use new StubViewModel() — never null (see Constraints)
});
_primaryNavService.NavigateTo("movies");  // default on Ready state entry
```

**Event subscriptions:**
- `_navigationService.NavigationChanged += OnNavigationChanged;` → `CurrentChildViewModel = _navigationService.Current;`
- `_primaryNavService.ActiveDestinationChanged += OnPrimaryDestinationChanged;` → `_navigationService.NavigateToRoot(dest.RootViewModel, dest.Title);` When `dest.Id == "library-folders"`, also calls `LibraryFolders.LoadFoldersCommand.ExecuteAsync(null)` (fire-and-forget) to populate the folder list, mirroring the loading behaviour that was previously in `ShowLibraryFolders()`.
- `_orchestrator.StateChanged += OnRefreshStateChanged;` (kept, updated — see below)
- `moviesListVm.EditMovieRequested += OnEditMovieRequested;` (kept, updated — see below)

**`OnRefreshStateChanged`:** When `Running` → `_primaryNavService.NavigateTo("refresh")`. On Completed/Faulted when active destination is Refresh → `_primaryNavService.NavigateTo("movies")`. Movies list reload on Completed/Faulted — unchanged; count is refreshed inside `MoviesListVm.LoadAsync`, so no separate `RefreshCountAsync` call is required.

**`OnEditMovieRequested`:** Create `editorVm` via factory, subscribe `editorVm.CloseRequested += OnEditorCloseRequested`, then call `_navigationService.NavigateTo(editorVm, "Edit Movie")` (was `CurrentPage = MovieEditor; CurrentChildViewModel = editorVm`).

**`OnEditorCloseRequested`:** Unsubscribe from editor, call `_navigationService.GoBack()` (was `CloseSubView()`), then fire `MoviesListVm.LoadAsync` — the post-close reload is kept unchanged.

**Remove:** `ShowLibraryFolders()` relay command (confirmed present — `[RelayCommand]` in `MainContentViewModel.cs`) and `CloseSubView()` relay command (confirmed present — `[RelayCommand]` in `MainContentViewModel.cs`). Also remove `MainContentPage`-based computed properties (`IsDefaultPage`, `IsSubViewPage`, `IsBackVisible`, `CurrentPageTitle`, `NavigateBackCommand`) — these were referenced in the previous plan draft; verify each individually against the current source before deleting, as they may not be present. Also remove `_indexedMovieCount` field, `IndexedMovieCount` observable property, and `RefreshCountAsync` method — count ownership moves to `MoviesListViewModel` (see IndexedMovieCount section).

**Add:**
- `[ObservableProperty] private bool _isPaneOpen = true;` — exposed for the `IsPaneOpen="{Binding IsPaneOpen}"` binding on the `NavigationView`.

**`Dispose()`:** Unsubscribe `_navigationService.NavigationChanged`, `_primaryNavService.ActiveDestinationChanged`, `_orchestrator.StateChanged`, `moviesListVm.EditMovieRequested`, and any active `editorVm.CloseRequested`.

---

### Step 10 — Delete `MainContentPage.cs`
The file `src/VideoIndexer.App/ViewModels/MainContentPage.cs` is deleted. All usages (currently only in `MainContentViewModel.cs` and its tests) are removed as part of Steps 9 and 16.

---

### Step 11 — Update `LibraryFoldersViewModel.cs`
No changes required to `LibraryFoldersViewModel` itself. The `CloseRequested` event proposed in the original plan was never implemented. Library Folders is now a primary rail destination — navigation away from it is handled by the rail, not by any action inside the view. The existing Add Folder / Delete Selected commands are kept unchanged.

> **Note:** The folder list is populated by calling `LibraryFolders.LoadFoldersCommand.ExecuteAsync(null)` inside `MainContentViewModel.OnPrimaryDestinationChanged` whenever `dest.Id == "library-folders"` (see Step 9 event-subscriptions). No `OnLoaded` hook is needed in `LibraryFoldersView.axaml.cs` for this purpose.

---

### Step 12 — Update `LibraryFoldersView.axaml`
Add `<views:PageHeaderView Title="Library Folders" />` as the first row of the view's root `Grid` (row definition `Auto`). No `BackCommand`, `PrimaryAction`, or `SecondaryActions` need to be set — Library Folders is a primary destination root; the Back button is automatically hidden because `INavigationService.CanGoBack == false` at root stack depth.

---

### Step 13 — Update `MoviesListView.axaml`
Add `<views:PageHeaderView Title="Movies">` as the first row. Set `SecondaryActions` slot to a `TextBlock` bound to `MoviesListViewModel.IndexedMovieCount` via `{Binding IndexedMovieCount, StringFormat='Indexed: {0}'}` (compiled binding against the view's existing `x:DataType="vm:MoviesListViewModel"` data context). `MoviesListViewModel` adds `[ObservableProperty] private long _indexedMovieCount;` and refreshes it from `IMovieCatalogRepository` inside `LoadAsync` — no public setter, external push, or constructor parameter required. Move the Search `TextBox` and Filter `ComboBox` (and orange filter indicator + Filters button) from the old toolbar into `MoviesListView.axaml` directly below the `PageHeaderView`.

---

### Step 14 — Update `RefreshIndexView.axaml`
Add `<views:PageHeaderView Title="Refresh Index" />` as the first row. The Back button is automatically hidden because `INavigationService.CanGoBack == false` at primary destination root. The existing Start Refresh and Cancel buttons stay in the view body.

---

### Step 15 — Update `MovieEditorView.axaml`
Add `<views:PageHeaderView>` as the first row of the editor's root `Grid`. The Back button is automatically shown because `INavigationService.CanGoBack == true` when the editor is on the stack. Set:
- `BackCommand="{Binding CloseCommand}"` — routes Back through the existing close/cleanup path.
- `PrimaryAction` slot: the existing "Save & Exit" button (moved from the bottom action toolbar into the header slot, or left in place with PrimaryAction as an alias pointing to the same command).
- `SecondaryActions` slot: "Discard" button.

The in-view bottom toolbar buttons ("Close", "Save & Exit", "Discard") may be kept as-is if product preference is to retain redundant affordances, or removed if the PageHeaderView slots are deemed sufficient. Mark this as a product decision in the plan and default to **keeping** them in the view body to avoid breaking the existing UX.

---

### Step 16 — Update `Program.cs`
Register the navigation services as singletons:
```csharp
services.AddSingleton<INavigationService, NavigationService>();
services.AddSingleton<IPrimaryNavigationService, PrimaryNavigationService>();
```
Both must be registered before `MainContentViewModel`, which receives them via constructor injection.

Also register `StubView` so `ViewLocator` can resolve it for the Filters and Settings stubs:
```csharp
services.AddTransient<StubView>();
```
`StubViewModel` requires no DI registration — it is constructed inline when `PrimaryNavigationService.Register` is called with the Filters and Settings destinations.

---

### Step 17 — Update tests

**`MainContentViewModelTests.cs`:**
- Update `BuildSut` / `BuildSutExtended` factory helpers: replace `FakeMovieCatalogRepository` etc. with also constructing `FakeNavigationService` and `FakePrimaryNavigationService` (new fakes to be added). Pass them to `MainContentViewModel` constructor.
- Remove tests that assert on `sut.CurrentPage` (enum deleted). Replace with assertions on `fakeNavService.Current` type and `fakePrimaryNavService.ActiveDestination.Id`.
- Add tests for: primary destination registration at construction, auto-navigation to Refresh on `Running` state, auto-return to Movies on Completed/Faulted, MovieEditor push on `EditMovieRequested`, GoBack called on editor `CloseRequested`.

**Add `FakeNavigationService.cs`** in `tests/VideoIndexer.App.Tests/TestHelpers/`:
- Implements `INavigationService`; records `NavigateToRoot`, `NavigateTo`, `GoBack`, `GoBackTo` calls; exposes `Current` and `BreadcrumbItems`; fires `NavigationChanged` on mutations.

**Add `FakePrimaryNavigationService.cs`** in the same directory:
- Implements `IPrimaryNavigationService`; records `Register`, `NavigateTo` calls; fires `ActiveDestinationChanged` on `NavigateTo`; exposes `ActiveDestination`.

**`LibraryFoldersViewModelTests.cs`:**
- No changes required (no new behaviour on `LibraryFoldersViewModel`).

**New `NavigationServiceTests.cs`** in `tests/VideoIndexer.App.Tests/`:
- `NavigateTo_PushesEntry_CurrentIsUpdated`
- `GoBack_PopsEntry_PreviousViewModelRestored`
- `GoBack_WhenAtRoot_DoesNothing`
- `NavigateToRoot_ClearsStack_CurrentIsNewRoot`
- `BreadcrumbItems_ReflectsStackOrderFromBottomToTop`
- `CanGoBack_True_WhenStackDepthGreaterThan1`
- `CanGoBack_False_WhenAtRoot`

**New `PrimaryNavigationServiceTests.cs`** in `tests/VideoIndexer.App.Tests/`:
- `NavigateTo_SetsActiveDestination`
- `NavigateTo_WhenAlreadyActive_DoesNotFireChangedEvent`
- `ActiveDestinationChanged_FiredOnDestinationChange`


## Dependencies

- **FluentAvalonia** must be added to `Directory.Packages.props` before any other step.
- Navigation model types (Step 2) must be created before service interfaces (Steps 3–4).
- Service interfaces (Steps 3–4) must be created before implementations (Steps 5–6).
- `PageHeaderView` (Step 7) must be created before any view is updated to embed it (Steps 12–15).
- `MainContentViewModel` update (Step 9) depends on navigation services (Steps 3–6).
- `Program.cs` update (Step 16) depends on all service implementations (Steps 5–6).
- Tests (Step 17) depend on all implementation steps.


## Required Components

**New files:**
- `src/VideoIndexer.App/Services/Navigation/NavigationEntry.cs` (new)
- `src/VideoIndexer.App/Services/Navigation/BreadcrumbItem.cs` (new)
- `src/VideoIndexer.App/Services/Navigation/PrimaryDestination.cs` (new)
- `src/VideoIndexer.App/Services/Navigation/INavigationService.cs` (new)
- `src/VideoIndexer.App/Services/Navigation/IPrimaryNavigationService.cs` (new)
- `src/VideoIndexer.App/Services/Navigation/NavigationService.cs` (new)
- `src/VideoIndexer.App/Services/Navigation/PrimaryNavigationService.cs` (new)
- `src/VideoIndexer.App/Views/PageHeaderView.axaml` (new)
- `src/VideoIndexer.App/Views/PageHeaderView.axaml.cs` (new)
- `tests/VideoIndexer.App.Tests/NavigationServiceTests.cs` (new)
- `tests/VideoIndexer.App.Tests/PrimaryNavigationServiceTests.cs` (new)
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeNavigationService.cs` (new)
- `tests/VideoIndexer.App.Tests/TestHelpers/FakePrimaryNavigationService.cs` (new)
- `src/VideoIndexer.App/ViewModels/StubViewModel.cs` (new — empty `ObservableObject` subclass used as `RootViewModel` for Filters and Settings rail stubs)
- `src/VideoIndexer.App/Views/StubView.axaml` (new — empty `UserControl`; ViewLocator resolves `StubViewModel` → `StubView`)
- `src/VideoIndexer.App/Views/StubView.axaml.cs` (new)

**Modified files:**
- `Directory.Packages.props` — FluentAvalonia version entry
- `src/VideoIndexer.App/VideoIndexer.App.csproj` — FluentAvalonia package reference
- `src/VideoIndexer.App/App.axaml` or `App.axaml.cs` — FluentAvalonia styles registration
- `src/VideoIndexer.App/ViewModels/MainContentViewModel.cs` — navigation service integration
- `src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs` — add `IndexedMovieCount` observable property; load it via `IMovieCatalogRepository` inside `LoadAsync`
- `src/VideoIndexer.App/Views/MainContentView.axaml` — NavigationView rail + workspace layout
- `src/VideoIndexer.App/Views/MainContentView.axaml.cs` — SelectionChanged handler
- `src/VideoIndexer.App/Views/LibraryFoldersView.axaml` — PageHeaderView embed
- `src/VideoIndexer.App/Views/MoviesListView.axaml` — PageHeaderView embed + Search/Filter migration
- `src/VideoIndexer.App/Views/RefreshIndexView.axaml` — PageHeaderView embed
- `src/VideoIndexer.App/Views/MovieEditorView.axaml` — PageHeaderView embed
- `src/VideoIndexer.App/Program.cs` — navigation service registrations
- `tests/VideoIndexer.App.Tests/MainContentViewModelTests.cs` — test updates
- `docs/agents/project-manifest/api-surface.md` — new navigation types + updated `MainContentViewModel`
- `docs/agents/project-manifest/file-tree.md` — new `Services/Navigation/` directory + `PageHeaderView`
- `docs/agents/project-manifest/tech-stack.md` — FluentAvalonia entry
- `docs/agents/project-manifest/constraints.md` — navigation rules
- `docs/agents/project-manifest/data-flows.md` — updated navigation flows

**Deleted files:**
- `src/VideoIndexer.App/ViewModels/MainContentPage.cs`


## Assumptions

- FluentAvalonia 2.x is compatible with Avalonia 11.3.14 and .NET 10. The exact `2.x` patch version must be verified before `Directory.Packages.props` is updated.
- `FiltersManagerView.axaml` does not yet exist (per `constraints.md` open items). The "Filters" rail item is implemented as a disabled or no-op stub with no routable VM. It must not call `_navigationService.NavigateToRoot` with a null VM — use a guard.
- Settings as a primary destination is similarly unbuilt. The "Settings" rail item is a stub. Same null-guard applies.
- `MoviesListView.axaml.cs` already creates and cancels a `CancellationTokenSource` in `OnLoaded`/`OnUnloaded`. Migrating Search/Filter into `MoviesListView.axaml` does not require any code-behind changes.
- The in-view bottom action toolbar on `MovieEditorView` (Close, Save & Exit, Discard buttons) is kept alongside the new `PageHeaderView` — duplicate affordance, both valid. Removal is a separate product decision out of scope here.
- `App.Services` static bridge is safe to use in `PageHeaderView.axaml.cs` for resolving `INavigationService`, following the same pattern as `ViewLocator.cs`.


## Constraints

- `TreatWarningsAsErrors=true` — all new code must compile clean.
- Nullable reference types enabled — new service interfaces, model records, and VM properties must satisfy nullability contracts.
- **No `Version=` on `<PackageReference>`** — FluentAvalonia must follow NuGet CPM: version in `Directory.Packages.props` only; no `Version=` attribute in `.csproj`.
- **`NavigationView` rail selection is code-behind-driven** — rail item highlighting is set by `MainContentView.axaml.cs` in response to `IPrimaryNavigationService.ActiveDestinationChanged`; no `SelectedItem` AXAML binding is used. The navigation service remains the single writer of rail state (spec section 5).
- **No `FluentAvalonia.Frame`** — workspace ContentControl + ViewLocator is the navigation host; no Frame is used (spec section 5).
- **`INavigationService.GoBack()` is a no-op at root** — when stack depth ≤ 1, `GoBack` must not clear the stack; the root VM must always remain.
- **Filters and Settings stubs must guard against null VM** — `PrimaryDestination.RootViewModel` must not be `null` at construction time. Use a dedicated `StubViewModel` (an empty `ObservableObject` subclass) for unimplemented areas, and register a matching `StubView` (empty UserControl) in ViewLocator.
- **`MovieEditorViewModel.CloseCommand` CanExecute guards the Back button** — when `HasChanges == true`, `CloseCommand.CanExecute` returns `false`, which propagates to `PageHeaderView.BackCommand` and disables the Back button. This is a departure from spec section 4.1 (which prescribes a confirmation dialog); a dialog guard can be added in a future iteration once a general-purpose `IConfirmationDialogService` is available.
- **Primary-zone (rail) switch is unguarded when MovieEditor has unsaved changes** — selecting a different rail item while `HasChanges == true` will clear the workspace stack via `IPrimaryNavigationService.NavigateTo`, discarding the editor. This is a known spec departure (section 4.1). A guard can be added to `MainContentView.axaml.cs` `SelectionChanged` in a future iteration.
- **`BreadcrumbBar.ItemClicked` is suppressed when BackCommand guard is active** — `PageHeaderView.axaml.cs` checks `BackCommand?.CanExecute(null) != false` before forwarding a breadcrumb click to `INavigationService.GoBackTo`. When `CloseCommand.CanExecute` returns `false`, breadcrumb clicks are suppressed consistently with the Back button disable.
- **CommunityToolkit.Mvvm remains the sole MVVM framework** — FluentAvalonia is a UI controls library only; it must not be used as an MVVM or navigation framework.
- **Core and Infrastructure layers are untouched** — all changes are within `VideoIndexer.App` and the test projects.
- **Rail and workspace are Ready-state-only** — The `NavigationView` rail and workspace `ContentControl` are children of `MainContentView`, which is instantiated only in the shell's Ready state. The navigation services are inactive in all other shell states. Auth-flow surfaces (`DatabaseConnectorView`, `PasswordSetupView`, error state) are owned exclusively by `ShellViewModel` and are never pushed onto the workspace stack (spec section 2.4).


## Out of Scope

- Animated workspace transitions (slide-in/out, cross-fade) — may be added later as opt-in behaviour on the workspace `ContentControl`.
- Deep linking / URI-addressable navigation.
- A confirmation dialog for leaving Movie Editor with unsaved changes — requires a `IConfirmationDialogService` not yet present; Back-button disable is the interim guard.
- Settings screen implementation — stub rail item only.
- Migrating FiltersManager from a modal dialog to a routable workspace screen — remains modal as today.
- Any changes to the Shell state machine (`ShellViewModel`, `DatabaseConnectorView`, auth flow).
- Any changes to Core or Infrastructure layers.
- Forward navigation history (browser-style Forward button).
- Navigation lifecycle hooks (`OnNavigatedTo` / `OnNavigatedFrom`) — defined in spec section 4 but deferred. The Library Folders folder-list load is the only current case requiring navigation-entry notification; it is handled explicitly in `OnPrimaryDestinationChanged` (see Step 9). General-purpose hooks are deferred until a second VM requires them.
- `SecondaryActions` overflow menu (collapsing beyond three items into a `…` menu) — spec recommendation only; deferred until a screen accumulates more than three secondary actions.
- Navigation telemetry and analytics — explicitly excluded per spec section 7.


## Acceptance Criteria

- AC-1: The Ready state renders a collapsible side rail listing Movies, Library Folders, Refresh, and stub entries for Filters and Settings.
- AC-2: Selecting a primary rail item clears the workspace stack and shows that area's root view in the workspace area.
- AC-3: Library Folders is no longer a sub-view of Movies; navigating to it via the rail replaces the workspace content and the user can leave by selecting another rail item — there is no "stranded" state.
- AC-4: Opening Movie Editor from the movies list pushes it onto the workspace stack. A `← Back` button appears in the `PageHeaderView`. Clicking it closes the editor and returns to the movies list (with list reload), following the existing `CloseRequested` → `OnEditorCloseRequested` path. The `MoviesListViewModel` instance (including its current search text and active filter) is the same object that was on the stack before the editor was pushed — navigation does not reconstruct it (spec section 2.2: "same instance is restored when the user returns to it via Back").
- AC-5: When `MovieEditorViewModel.HasChanges == true`, the `PageHeaderView` Back button is disabled (bound to `CloseCommand.CanExecute`).
- AC-6: Every routable view (Movies, Library Folders, Refresh, Movie Editor) displays a `PageHeaderView` at the top with the correct page title.
- AC-7: The `BreadcrumbBar` in `PageHeaderView` reflects the workspace stack — one item when at root, two items when Movie Editor is open. Clicking a breadcrumb item navigates back to that stack level via `INavigationService.GoBackTo`; when the Back button guard is active (`HasChanges == true`), breadcrumb clicks are also suppressed.
- AC-7b: The Back button in `PageHeaderView` is shown and hidden automatically based on `INavigationService.CanGoBack`; no per-view `IsBackVisible` attribute is set on any host view.
- AC-8: All existing `MainContentViewModelTests` either pass without modification or are updated to pass under the new navigation model. No tests are deleted without a replacement.
- AC-9: `NavigationServiceTests` and `PrimaryNavigationServiceTests` pass, covering the navigation stack invariants in the Test Plan.
- AC-10: The Search `TextBox` and Filter `ComboBox` are visible when the Movies area is active and absent when any other primary area is active (they now live inside `MoviesListView`).
- AC-11: `MainContentViewModel.Dispose()` unsubscribes from all navigation service events; no post-dispose navigation side effects occur.
- AC-12: Modal dialogs (file/folder pickers, Connection Editor, Password Setup prompt, Label Cleaner result dialog) are not pushed onto the workspace stack and do not appear in the breadcrumb. They continue to be opened as separate top-level windows, outside the navigation system entirely (spec section 2.4).
- AC-13: The primary navigation rail and workspace zone are only rendered when the shell is in the Ready state. Database connector, password setup prompt, and error state surfaces are owned entirely by `ShellViewModel` and are never pushed onto the workspace stack — they are invisible to `INavigationService` and `IPrimaryNavigationService` (spec section 2.4).


## Testing Strategy

All tests are unit tests in `VideoIndexer.App.Tests` (no DB, no UI thread). New fakes (`FakeNavigationService`, `FakePrimaryNavigationService`) stand in for the navigation services in `MainContentViewModel` tests, replacing the now-deleted `MainContentPage`-based assertions. New dedicated test classes cover the navigation stack and primary navigation services in isolation. Headless Avalonia tests are not required for AXAML layout changes — those are covered by AC manual smoke-test at review.

`PageHeaderView` is **explicitly out of scope for headless UI-thread tests**: `PageHeaderView.axaml.cs` resolves `INavigationService` from `App.Services` at construction, so testing it in isolation requires a populated `App.Services`. This is deferred until a general test-fixture pattern for control-level headless tests is established.


## Test Plan

### `NavigationServiceTests.cs` (new)
- `NavigateTo_PushesEntry_CurrentIsUpdated` — pushes a VM, asserts `Current == vm`. **Covers AC-4, AC-7.**
- `NavigateTo_FiresNavigationChanged` — pushes, asserts event fired. **Covers AC-4.**
- `GoBack_PopsEntry_PreviousViewModelRestored` — pushes two VMs, GoBack, asserts `Current == firstVm`. **Covers AC-4, AC-7.**
- `GoBack_WhenAtRoot_DoesNothing` — NavigateToRoot then GoBack, asserts `Current` unchanged and `CanGoBack == false`. **Covers AC-11.**
- `NavigateToRoot_ClearsStack_CurrentIsNewRoot` — NavigateTo two items, NavigateToRoot, asserts stack depth 1 and `Current == rootVm`. **Covers AC-2.**
- `BreadcrumbItems_AtRoot_ContainsSingleItem` — NavigateToRoot, asserts `BreadcrumbItems.Count == 1`. **Covers AC-7.**
- `BreadcrumbItems_AfterDrillDown_ContainsTwoItems` — NavigateToRoot + NavigateTo, asserts `BreadcrumbItems.Count == 2`. **Covers AC-7.**
- `CanGoBack_False_WhenAtRoot` — asserts `CanGoBack == false` after NavigateToRoot. **Covers AC-5.**
- `CanGoBack_True_WhenDrilledDown` — NavigateToRoot + NavigateTo, asserts `CanGoBack == true`. **Covers AC-5.**
- `GoBackTo_NavigatesBackToSpecifiedEntry` — NavigateToRoot + NavigateTo twice, GoBackTo(0), asserts `Current == rootVm` and `BreadcrumbItems.Count == 1`. **Covers AC-7.**
- `GoBackTo_NoOp_WhenIndexIsCurrentTop` — NavigateToRoot + NavigateTo, GoBackTo(1), asserts `Current` unchanged. **Covers AC-7.**
- `GoBackTo_AtRoot_DoesNotEmptyStack` — NavigateToRoot then GoBackTo(0), asserts `Current == rootVm` and `CanGoBack == false`. **Covers AC-7.**
- `GoBack_RestoresSameViewModelInstance` — NavigateToRoot(vm1), NavigateTo(vm2), GoBack(), asserts `object.ReferenceEquals(Current, vm1) == true`. **Covers spec section 2.2 "same instance is restored when the user returns to it via Back"; covers AC-4.**

### `PrimaryNavigationServiceTests.cs` (new)
- `NavigateTo_SetsActiveDestination` — register destinations, NavigateTo("library-folders"), asserts `ActiveDestination.Id == "library-folders"`. **Covers AC-2, AC-3.**
- `NavigateTo_FiresActiveDestinationChanged` — asserts event fired with new destination. **Covers AC-2.**
- `NavigateTo_WhenAlreadyActive_DoesNotFireChangedEvent` — navigate to same destination twice, asserts event fired only once. **Covers AC-2.**

### `MainContentViewModelTests.cs` (updated)
- `Constructor_PrimaryNavService_RegisteredWithFourDestinations` — asserts `fakePrimaryNavService.Destinations.Count >= 3` (Movies, Library Folders, Refresh). **Covers AC-1.**
- `Constructor_DefaultDestination_IsMovies` — asserts `fakePrimaryNavService.LastNavigatedId == "movies"`. **Covers AC-2.**
- `OnPrimaryDestinationChanged_Movies_NavigatesWorkspaceToMoviesListVm` — fire ActiveDestinationChanged for Movies, asserts `fakeNavService.LastNavigateToRootVm == moviesListVm`. **Covers AC-2.**
- `OnPrimaryDestinationChanged_LibraryFolders_NavigatesWorkspaceToLibraryFoldersVm` — fire for Library Folders, asserts root VM is LibraryFolders. **Covers AC-3.**
- `OnRefreshStateChanged_Running_PrimaryNavNavigatesToRefresh` — simulate `RefreshState.Running`, asserts `fakePrimaryNavService.LastNavigatedId == "refresh"`. **Covers AC-2.**
- `OnRefreshStateChanged_Completed_PrimaryNavNavigatesToMovies` — simulate Completed, asserts navigated to movies. **Covers AC-2.**
- `OnEditMovieRequested_PushesEditorOntoWorkspaceStack` — trigger edit, asserts `fakeNavService.NavigateToCalled == true` and `fakeNavService.LastNavigatedVm is MovieEditorViewModel`. **Covers AC-4.**
- `OnEditorCloseRequested_CallsGoBack` — raise CloseRequested on editor, asserts `fakeNavService.GoBackCallCount == 1`. **Covers AC-4.**
- `OnEditorCloseRequested_FiresMoviesListReload` — raise CloseRequested, yield, asserts `catalog.GetCountAsyncCallCount > 0` or movies list reload triggered. **Covers AC-4.**
- `Dispose_UnsubscribesNavigationServiceEvents_NoSideEffects` — dispose SUT, fire NavigationChanged, asserts `CurrentChildViewModel` did not change. **Covers AC-11.**


## Documentation Updates

- `docs/agents/project-manifest/tech-stack.md` — Add FluentAvalonia to the UI Framework table (package name + version + purpose: NavigationView rail + BreadcrumbBar).
- `docs/agents/project-manifest/file-tree.md` — Add `src/VideoIndexer.App/Services/Navigation/` directory with all new files. Add `PageHeaderView.axaml / .cs` under `Views/`. Add `StubView.axaml / .cs` under `Views/`. Add `StubViewModel.cs` under `ViewModels/`. Remove `MainContentPage.cs` entry. Add new test files under `VideoIndexer.App.Tests/`.
- `docs/agents/project-manifest/api-surface.md` — Add section "Navigation Services" with interfaces `INavigationService`, `IPrimaryNavigationService` and record types `NavigationEntry`, `BreadcrumbItem`, `PrimaryDestination`. Update `MainContentViewModel` section: remove `CurrentPage`, `ShowLibraryFoldersCommand`, `CloseSubViewCommand`, `IndexedMovieCount`, `RefreshCountAsync`; note `CurrentChildViewModel` is now driven by `INavigationService`; add `IsPaneOpen` observable property. Update `MoviesListViewModel` section: add `IndexedMovieCount` observable property (owned and loaded by `MoviesListViewModel` inside `LoadAsync`). Remove `LibraryFoldersViewModel` `CloseRequested` / `DoneCommand` stubs (never implemented; not added). Add `PageHeaderView` styled properties (`TitleProperty`, `BackCommandProperty`, `PrimaryActionProperty`, `SecondaryActionsProperty`) and note the header layout: Left = BreadcrumbBar + Back button; Centre = optional title (hidden when empty); Right = PrimaryAction + SecondaryActions. Add `StubViewModel` (empty `ObservableObject` subclass) entry.
- `docs/agents/project-manifest/constraints.md` — Add navigation rules: *"Primary navigation (rail) is driven exclusively by `IPrimaryNavigationService.NavigateTo` — never by directly setting `CurrentChildViewModel`. The rail `SelectedItem` binding is `Mode=OneWay`."* *"Filters and Settings rail stubs must provide a non-null `StubViewModel` as `RootViewModel`."* *"Back for MovieEditor routes through `MovieEditorViewModel.CloseCommand` (event-handshake path) — not directly to `INavigationService.GoBack()`."* *"No `FluentAvalonia.Frame` — workspace uses `ContentControl` + `ViewLocator` only."* *"The `NavigationView` rail and workspace zone are only rendered in the Ready shell state; auth-flow surfaces remain entirely outside the navigation system."*
- `docs/agents/project-manifest/data-flows.md` — Replace the Library Folder Management section's close path with: *"User selects another primary destination in the rail → `NavigationView.SelectionChanged` → `IPrimaryNavigationService.NavigateTo(id)` → `ActiveDestinationChanged` fired → `MainContentViewModel.OnPrimaryDestinationChanged` → `INavigationService.NavigateToRoot(vm, title)` → `NavigationChanged` fired → `CurrentChildViewModel = INavigationService.Current`."* Update Movie Editor open/close flows to show `INavigationService.NavigateTo(editorVm, "Edit Movie")` and `INavigationService.GoBack()` replacing the old `CurrentPage` assignments.


## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **FluentAvalonia version incompatible with Avalonia 11.3.14** | Verify the FluentAvalonia 2.x changelog for supported Avalonia minor versions before committing. If incompatible, implement rail and breadcrumb manually using Avalonia Fluent theme (spec section 5.1 fallback). |
| **`NavigationView` styled property conflicts with `Avalonia.Themes.Fluent`** | FluentAvalonia is designed to layer on top of Avalonia.Themes.Fluent. Register FluentAvalonia styles after the Fluent theme in `App.axaml`. Run the build and smoke-test visually at the earliest opportunity. |
| **`PageHeaderView` breadcrumb access via `App.Services` causes issues in headless unit tests** | The headless test fixture initialises the Avalonia application. Ensure `App.Services` is populated in the test fixture setup (it already is for `Avalonia.Headless.XUnit`). If not, refactor `PageHeaderView.axaml.cs` to accept `INavigationService` via a public property set by the test. |
| **MovieEditor Back button double-fires (GoBack from CloseRequested + direct navigation service call)** | `OnEditorCloseRequested` guards with `if (CurrentPage != MainContentPage.MovieEditor)` — this guard is removed when the enum is deleted. Replace with a guard on the navigation service: only call `GoBack()` if `Current is MovieEditorViewModel`. |
| **Search/Filter migration breaks CancellationTokenSource lifecycle in `MoviesListView.axaml.cs`** | Moving Search/Filter AXAML into `MoviesListView` is purely a view layout change; the code-behind CTS lifecycle is not affected. The bindings reference `MoviesListViewModel` properties which are unchanged. |
| **Stubs for Filters/Settings crash when selected** | `StubViewModel` + `StubView` pair registered in ViewLocator before the first rail render. `PrimaryNavigationService.NavigateTo("filters")` navigates to `StubViewModel`, which renders as an empty view. No crash. |
| **`MainContentViewModelTests` entirely broken by enum deletion** | Tests are updated in the same plan step (Step 17). `FakeNavigationService` and `FakePrimaryNavigationService` replace the deleted `MainContentPage` assertions. Existing test helper setup is extended, not replaced. |
| **Breadcrumb item click bypasses MovieEditor unsaved-change guard** | `PageHeaderView.axaml.cs` checks `BackCommand?.CanExecute(null) != false` before calling `GoBackTo`. When `HasChanges == true` and `BackCommand == CloseCommand`, breadcrumb clicks are suppressed, consistent with the Back button disable. The guard is implemented in the `BreadcrumbBar.ItemClicked` handler in Step 7. |
| **Primary-zone (rail) switch discards Movie Editor state silently** | This is a spec departure (section 4.1). The `SelectionChanged` handler in `MainContentView.axaml.cs` does not intercept selections when the editor has unsaved changes. Document as a known limitation; add an `INavigationGuard` or equivalent in a future iteration. |
