# Project Synthesis Report

**Project:** `2026-05-15-consistent-subview-navigation`
**Date:** 2026-05-15
**Status:** COMPLETE
**WPs:** 16 / 16 COMPLETE — 4-stage pipeline (implementation → qa → code-review → documentation)

---

## Executive Summary

This session delivered a complete, consistent navigation architecture for Video Indexer v2. The work replaced an ad-hoc `MainContentPage` enum-driven navigation system with a fully type-safe, service-based navigation layer built around two new interfaces (`INavigationService` and `IPrimaryNavigationService`) and a FluentAvalonia `NavigationView` rail in `MainContentView`.

The feature delivers:
- **A hierarchical sub-view navigation stack** (`INavigationService`) with breadcrumb trail, back-navigation, and event-driven change notification.
- **A primary rail navigation service** (`IPrimaryNavigationService`) managing the five top-level destinations (Movies, Library Folders, Refresh, Filters (stub), Settings (stub)).
- **A reusable `PageHeaderView` control** providing consistent title, breadcrumb bar, and back-button UX across all content views.
- **FluentAvaloniaUI 2.5.1 integration** replacing `FluentTheme` with `FluentAvaloniaTheme` as the application theme.
- **Full DI wiring** in `Program.cs` with navigation singletons and updated view registrations.
- **Comprehensive test coverage** — 680 tests passing with 0 failures.

The `MainContentPage.cs` enum file was deleted. The `MainContentViewModel` was entirely rewritten. Five views (`MainContentView`, `MoviesListView`, `MovieEditorView`, `RefreshIndexView`, and the new `PageHeaderView`) were restructured.

---

## Metrics

| Metric | Value |
|---|---|
| Work packages completed | 16 / 16 |
| Pipeline stage passes | 64 / 64 (100%) |
| Build errors | 0 |
| Build warnings | 0 (TreatWarningsAsErrors=true) |
| Final test suite | **680 passed, 0 failed, 6 skipped** |
| Tests at session start | 629 |
| **Net new tests added** | **+51** |
| Rework cycles (QA bounce) | 1 (WP-008) |
| Rework cycles (code-review FAIL) | 1 (WP-010) |
| Reviewer Fix-Forwards applied | 3 (WP-007, WP-008, WP-013) |

### New Test Files

| File | Tests |
|---|---|
| `NavigationServiceTests.cs` | 22 |
| `PrimaryNavigationServiceTests.cs` | 12 |
| `PageHeaderViewTests.cs` | 13 |
| `FakeNavigationService.cs` | (test helper) |
| `FakePrimaryNavigationService.cs` | (test helper) |
| `MainContentViewModelTests.cs` | +4 new, existing migrated to fakes |

---

## Work Package Summary

| WP | Spec File | Title | Rework | Key Outcome |
|---|---|---|---|---|
| WP-001 | WP-001.md | FluentAvaloniaUI 2.5.1 | — | Theme migration; constraints.md updated |
| WP-002 | WP-002.md | Navigation interfaces & records | — | 5 new types in `Services/Navigation/` |
| WP-003 | WP-005.md | StubViewModel + StubView | — | Placeholder ViewModel/View pair |
| WP-004 | WP-003.md | NavigationService implementation | — | Stack-based navigation; 22 unit tests |
| WP-005 | WP-004.md | PrimaryNavigationService | — | Rail destination manager; 12 unit tests |
| WP-006 | WP-006.md | PageHeaderView | — | Reusable header control; 13 headless tests |
| WP-007 | WP-007.md | MainContentViewModel rewrite | — | Deleted MainContentPage.cs; DI wired |
| WP-008 | WP-011.md | MainContentView NavView restructure | 1 QA bounce | Header binding + IsBackButtonVisible fix |
| WP-009 | WP-012.md | RefreshIndexView PageHeaderView | — | Applies standard 2-row Grid pattern |
| WP-010 | WP-013.md | MovieEditorView PageHeaderView | 1 code-review FAIL | DataContext shadowing fix; bottom toolbar retained |
| WP-011 | WP-008.md | MainContentView verification | — | Confirms WP-008 implementation already complete |
| WP-012 | WP-009.md | Program.cs DI registration | — | StubView AddTransient added |
| WP-013 | WP-010.md | MoviesListView PageHeaderView | — | IndexedMovieCount migrated; Search/Filter moved |
| WP-014 | WP-014.md | Navigation service unit tests | — | Fakes + 32 navigation tests |
| WP-015 | WP-015.md | MainContentViewModelTests migration | — | SutBundle switched to fakes; 4 new tests |
| WP-016 | WP-016.md | Final manifest update | — | All 5 manifest docs comprehensive and consistent |

---

## Failures and Rework

### WP-008 — QA Bounce (1 rework cycle)
- **Issue 1 (High):** `NavView.Header` was never set — FluentAvalonia does not auto-populate it from the selected item. Fix: `UpdateSelectedItem()` performs a `Destinations` lookup and assigns `NavView.Header` directly.
- **Issue 2 (Medium):** `IsBackButtonVisible` was not set. FluentAvalonia exposes this as a plain `bool` (not the WinUI `NavigationViewBackButtonVisible` enum). Fix: `IsBackButtonVisible="False"` added to AXAML. Attempting `Collapsed` caused AVLN1000.
- **Code-Review Fix-Forward:** Duplicate `NavView.SelectionChanged += OnNavViewSelectionChanged` subscription in constructor body removed (AXAML attribute already covers it).

### WP-010 — Code-Review FAIL (1 rework cycle)
- **Issue 1 (High/Blocking):** `BackCommand="{Binding CloseCommand}"` in `MovieEditorView.axaml` silently fails because `PageHeaderView.DataContext=this` shadows the inherited `MovieEditorViewModel` context. The back button remained enabled even with unsaved changes. Fix: imperative `DataContextChanged` handler sets `PageHeader.BackCommand = vm.CloseCommand`.
- **Issue 2 (High/Blocking):** `PrimaryAction`/`SecondaryActions` header slot buttons (`SaveAndExitCommand`, `DiscardChangesCommand`) were dead non-functional UI for the same DataContext reason. Fix: slots removed; bottom toolbar remains the canonical action surface.

### Reviewer Fix-Forwards Applied
| WP | Fix |
|---|---|
| WP-007 | Added missing `Should()` assertion to `Dispose_UnsubscribesFromNavigationChanged` test |
| WP-008 | Removed duplicate `NavView.SelectionChanged` constructor subscription |
| WP-013 | Added blank-line separators around new `_indexedMovieCount` ObservableProperty |

---

## Strategic Recommendations (Gold Nuggets)

### 1. PageHeaderView DataContext Shadowing Constraint — Documented
`PageHeaderView` sets `DataContext = this` in its constructor to enable its internal compiled AXAML bindings. This means parent views **cannot** set `PageHeaderView` styled properties (`BackCommand`, `PrimaryAction`, `SecondaryActions`) via `{Binding}` attribute syntax. The required pattern is code-behind `DataContextChanged` → imperative property assignment. This is now documented in `constraints.md` with `MoviesListView.axaml.cs` as the canonical example.
> **Recommendation:** Consider a future refactor using an attached behaviour or helper base class to reduce boilerplate in views that need dynamic `SecondaryActions`.

### 2. Orphaned Avalonia.Themes.Fluent Package Reference
`Avalonia.Themes.Fluent` remains in `VideoIndexer.App.csproj` as an unused `PackageReference`. `App.axaml` now uses `FluentAvaloniaTheme` exclusively. The reference emits no warnings (harmless) but pollutes the dependency graph.
> **Recommendation:** Remove in a dedicated clean-up task.

### 3. DataGrid StyleInclude May Be Redundant
`App.axaml` retains `avares://Avalonia.Controls.DataGrid/Themes/Fluent.xaml` alongside `FluentAvaloniaTheme`. FluentAvaloniaUI bundles DataGrid support and may already inject compatible styles, making this `StyleInclude` redundant or causing duplication.
> **Recommendation:** Verify and remove if redundant.

### 4. ViewLocatorTests Manual Registration Debt
`ViewLocatorTests.BuildViewServices()` manually registers each `UserControl` subclass. With new views added this session, this factory requires manual updates per view. Convention-based auto-registration would eliminate this maintenance burden.
> **Recommendation:** Consider scanning the App assembly for `UserControl` subclasses in a future refactor.

### 5. LoadAsync Error Handling Inconsistency
`OnRefreshStateChanged` calls `MoviesListVm.LoadAsync(CancellationToken.None)` with no error continuation (fire-and-forget). `OnEditorCloseRequested` wraps the same call with `.ContinueWith(...OnlyOnFaulted)` for error logging. Load failures after a scan completes will be silently swallowed.
> **Recommendation:** Align both call sites to use the `.ContinueWith` logging pattern.

### 6. StubViewModel/StubView Are Temporary Placeholders
`StubViewModel` and `StubView` are wired to the "filters" and "settings" primary rail destinations. They must be replaced with dedicated `FilterManagerViewModel`/`FilterManagerView` and `SettingsViewModel`/`SettingsView` pairs before those features are built. This is documented in `constraints.md`.
> **Recommendation:** Create plan WPs for these two feature destinations when those features are scoped.

### 7. FluentAvalonia IsBackButtonVisible Is a Bool, Not a WinUI Enum
Attempting `IsBackButtonVisible="Collapsed"` (WinUI `NavigationViewBackButtonVisible` enum value) produces AVLN1000. The correct declarative syntax is `IsBackButtonVisible="False"`. This is now recorded in `constraints.md`.

### 8. FakeNavigationService Duplicates Real Service Stack Logic
`FakeNavigationService` in `TestHelpers` faithfully mirrors the real `NavigationService` stack manipulation. This is intentional and correct, but if `NavigationService` logic changes, the fake must be kept in sync.
> **Recommendation:** Add a maintenance note to `NavigationService` pointing to `FakeNavigationService`.

---

## Next Steps

1. **Remove `Avalonia.Themes.Fluent`** from `VideoIndexer.App.csproj` — trivial clean-up.
2. **Verify/remove DataGrid StyleInclude** in `App.axaml` — visual regression test recommended.
3. **Build real FilterManager and Settings destinations** — replace `StubViewModel`/`StubView` placeholders.
4. **Add tests for `MovieEditorView.BackCommand` wiring** — headless `AvaloniaFact` test instantiating `MovieEditorView` with a `MovieEditorViewModel` DataContext and asserting `PageHeader.BackCommand == vm.CloseCommand` and button disabled when `HasChanges=true`.
5. **Align `LoadAsync` error-continuation pattern** across `OnRefreshStateChanged` and `OnEditorCloseRequested` in `MainContentViewModel`.
6. **Refactor `PageHeaderView` dynamic property pattern** — reduce `DataContextChanged + PropertyChanged` boilerplate in views with dynamic `SecondaryActions`.
7. **Address `LibraryFoldersView` PageHeaderView integration** — this view was not covered in the current plan; confirm whether it should follow the same 2-row Grid pattern.
