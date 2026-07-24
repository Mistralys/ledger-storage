# Plan — M7: Tagging Core (Final Completion)

## Plan Audit Cycles
- Audits: 2 — Plan Auditor v1.3.0
- Architectural Reviews: 1 — Plan Architect Reviewer v1.4.0

---

## Summary

Milestone M7 (Tagging Core) is almost entirely implemented. The tag data model, `ITagsRepository`, `ITagsManager`, `TagsManager`, `DapperTagsRepository`, three dialog-service pairs (Tag Editor, Grants Management, Tag Merge), all associated ViewModels, and the `TaggerView` control are all present and wired. The filter DSL correctly evaluates `HasTag`, `TagsWeight`, and `AmountTags` against the effective tag set. `DapperMovieCatalogRepository.GetMovieListAsync` enriches `MovieListItem` with `EffectiveTagIds` / `EffectiveTagWeight` / `EffectiveTagCount`. `MovieEditorView` hosts the live `TaggerView` in Column 2.

One gap remains: **category management is unserviced**. The `TaggerViewModel.AddCategoryAsync`, `EditCategoryAsync`, and `DeleteCategoryAsync` commands are explicit stubs awaiting `ICategoryEditorService`. `CategoryEditorViewModel` and `CategoryEditorView` are fully implemented but have no wiring to show the dialog. This plan closes that gap and delivers M7 as a complete, shippable milestone.

---

## Architectural Context

### Existing dialog-service pattern (established in M5–M6)

Every modal dialog in the application follows an identical four-file pattern:

| Role | Layer | Example (M7 Tag Editor) |
|------|-------|------------------------|
| Interface | `Core/Abstractions/` | `ITagEditorService.cs` |
| Avalonia implementation | `App/Services/` | `AvaloniaTagEditorService.cs` |
| DI registration | `App/Program.cs` | `builder.Services.AddSingleton<ITagEditorService>(...)` |
| VM wiring | `App/ViewModels/` | `TaggerViewModel._tagEditorService` |

`AvaloniaCategoryEditorService` follows this pattern without deviation.

### `CategoryEditorViewModel` constructor contract

```csharp
public CategoryEditorViewModel(
    TagCategory?         existingCategory,   // null = create mode
    IReadOnlyList<string> existingNames,     // all current category names for duplicate-name validation
    int                  tagCount)           // Delete button disabled when > 0
```

Events:
- `CloseRequested<TagCategory?>` — raised on Save (non-null) or Cancel (null)  
- `DeleteRequested` — raised when the user clicks Delete inside the editor

The service must handle `DeleteRequested` by calling `ITagsManager.DeleteCategoryAsync` and closing the window. Tag count and existing names are derived from `ITagsManager` at show time.

### `TaggerViewModel` constructor (current)

```csharp
public TaggerViewModel(
    ITagsManager             tagsManager,
    ITagEditorService        tagEditorService,
    ITagMergeService         tagMergeService,
    IGrantsManagementService grantsManagementService,
    IReadOnlyList<long>      initialStoredTagIds,
    Func<long, CancellationToken, Task> connectTag,
    Func<long, CancellationToken, Task> disconnectTag)
```

Adding `ICategoryEditorService? categoryEditorService = null` (after `disconnectTag`, as the final optional parameter) extends this constructor. `MovieEditorViewModel`, which constructs `TaggerViewModel`, also requires the new parameter.

> **Note:** The parameter must be placed **after** `disconnectTag` (position 8), not after `grantsManagementService` (position 4). C# CS1737 forbids an optional parameter from appearing before required parameters; `initialStoredTagIds`, `connectTag`, and `disconnectTag` are all required and must remain so.

### `TaggerView.axaml` — category tab headers

The `TabControl.ItemTemplate` DataTemplate currently renders only `<TextBlock Text="{Binding Category.Name}" />` with no context menu. A `ContextMenu` with **Edit Category** and **Delete Category** items bound to `EditCategoryCommand` / `DeleteCategoryCommand` needs to be added. The commands receive the `TaggerCategoryViewModel` as a `CommandParameter`.

### Build warnings

The current build produces two **CS8765** warnings in `tests/VideoIndexer.Tests/TagsManagerTests.cs` at lines 346 and 374. Both are nullable-annotation mismatches on overridden interface members in test fake types. They are non-blocking today (test project does not apply `TreatWarningsAsErrors=true`), but eliminating them is good hygiene and should be done as part of closing M7.

---

## Approach / Architecture

1. Create `ICategoryEditorService` in `Core/Abstractions/` following the same shape as `ITagEditorService`.
2. Implement `AvaloniaCategoryEditorService` in `App/Services/`, injecting `ITagsManager` (for tag-count lookup and `DeleteCategoryAsync`) and `ownerFactory`.
3. Register the service in `Program.cs` in the existing "Section 5.75 — tag-related dialog services" block.
4. Extend `TaggerViewModel` to accept `ICategoryEditorService` and implement the three category commands.
5. Extend `MovieEditorViewModel` to inject and forward `ICategoryEditorService` to the `TaggerViewModel` constructor in `LoadAsync`.
6. Update `TaggerView.axaml` to add a right-click context menu on category tab headers.
7. Fix the two CS8765 build warnings in `TagsManagerTests.cs`.
8. Add unit-test coverage for the newly wired category commands.
9. Update manifest documents.

---

## Rationale

**Single service per dialog pair.** The existing codebase has a strict one-interface-one-implementation pattern for all dialogs. Adding a second dialog for "just delete" would introduce an unnecessary second surface. The `CategoryEditorViewModel` already contains a Delete button and the guard logic; the service simply exposes it.

**`ITagsManager` injected into `AvaloniaCategoryEditorService`.** The service needs `ITagsManager.Tags` to compute tag count for the VM constructor, and `ITagsManager.DeleteCategoryAsync` to handle `DeleteRequested`. Injecting the manager (a singleton) directly avoids a round-trip to `ITagsRepository` for count data that is already cached in memory.

**`DeleteCategoryAsync` in `TaggerViewModel` opens the editor (not a direct call).** This matches the spec §3.7 ("Right-click → Delete") and is consistent with how tag deletion works — the editor is the confirmation surface. The Delete button inside the editor is guarded against non-empty categories. Calling the manager directly from the VM without a dialog would bypass the user confirmation step.

---

## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|----------|--------------|-------------------------|-------------------|
| Category delete from VM | Open category editor (via service) | Direct `_tagsManager.DeleteCategoryAsync` from `DeleteCategoryAsync` | Spec §3.7 requires a confirmation surface; editor provides that without a second dialog. Direct call would silently fail on non-empty categories with no user feedback. |
| Tag count source in service | `ITagsManager.Tags.Count(t => t.CategoryId == id)` | Extra `ITagsRepository` query | Manager cache already holds all tags; extra DB round-trip unnecessary and increases latency of opening the dialog. |
| `ICategoryEditorService` location | `Core/Abstractions/` | `App/Services/` | Other service interfaces (`ITagEditorService`, `ITagMergeService`, `IGrantsManagementService`) live in Core/Abstractions for cross-layer contract visibility. Consistency requires the same. |

---

## Pattern Alignment

| Pattern | File | Status |
|---------|------|--------|
| Dialog service: interface in Core/Abstractions | `src/VideoIndexer.Core/Abstractions/ITagEditorService.cs` | Followed |
| Dialog service: Avalonia impl in App/Services | `src/VideoIndexer.App/Services/AvaloniaTagEditorService.cs` | Followed |
| DI registration via `AddSingleton` factory lambda with `ownerFactory` | `src/VideoIndexer.App/Program.cs` section `// 5.75 — UI dialog services` | Followed |
| Modal Window via `ShowDialog(owner)` with named-handler try/finally unsubscription | All `Avalonia*Service.cs` impls | Followed |
| `CancellationToken` declared but not observed (Avalonia limitation) | XML doc on all Avalonia dialog service methods | Followed |

---

## Detailed Steps

### WP-001 — `ICategoryEditorService` interface

1. Create `src/VideoIndexer.Core/Abstractions/ICategoryEditorService.cs`.
2. Declare one method:
   ```csharp
   Task<TagCategory?> ShowAsync(TagCategory? existingCategory = null,
       CancellationToken cancellationToken = default);
   ```
   Returns the created/renamed `TagCategory` on save, `null` on cancel or delete.  
   XML doc: note that `CancellationToken` is declared but not observed (Avalonia `ShowDialog` limitation, per existing convention).

---

### WP-002 — `AvaloniaCategoryEditorService`

1. Create `src/VideoIndexer.App/Services/AvaloniaCategoryEditorService.cs`.
2. Constructor: `(ITagsManager tagsManager, Func<Window?> ownerFactory)`.
3. `ShowAsync` implementation:
   - Derive `existingNames` from `tagsManager.Categories` (excluding the virtual Most Used sentinel and, when editing, excluding the category being renamed).
   - Derive `tagCount` from `tagsManager.Tags.Count(t => t.CategoryId == existingCategory.CategoryId)` (zero in create mode).
   - Construct `CategoryEditorViewModel(existingCategory, existingNames, tagCount)`.
   - Create a `Window` hosting `new CategoryEditorView { DataContext = vm }`, sized ~360 × 200, `CanResize = false`.
   - Subscribe to `vm.CloseRequested` → capture result, close window.
   - Subscribe to `vm.DeleteRequested` → **synchronous** one-liner: set `bool deletePending = true`, call `dialog.Close()`. Do **not** use an `async void` lambda here — `TagsManager.DeleteCategoryAsync` uses `ConfigureAwait(false)` internally (line 395 of `TagsManager.cs`), so any continuation after `await` would run on a thread-pool thread and calling `dialog.Close()` from there would violate Avalonia's UI-thread requirement.
   - `try { await window.ShowDialog(owner); } finally { unsubscribe handlers; }`.
   - After the `try/finally` block: if `deletePending`, `await tagsManager.DeleteCategoryAsync(existingCategory!.CategoryId).ConfigureAwait(false)` and set `result = null`. All async work is in the existing `async Task ShowAsync` method body, after the dialog has already closed — consistent with every other `Avalonia*Service` in the project.
   - Return captured result.

---

### WP-003 — DI registration

1. In `src/VideoIndexer.App/Program.cs`, add to the "Section 5.75 — tag-related dialog services" block:
   ```csharp
   builder.Services.AddSingleton<ICategoryEditorService>(sp =>
       new AvaloniaCategoryEditorService(
           sp.GetRequiredService<ITagsManager>(),
           ownerFactory));
   ```
   Position: after `ITagMergeService` registration, within the `// 5.75 — UI dialog services` block.

---

### WP-004 — Extend `TaggerViewModel`

1. Add `private readonly ICategoryEditorService? _categoryEditorService;` field (nullable — no null-guard throw).
2. Add `ICategoryEditorService? categoryEditorService = null` parameter to the full constructor, positioned **after `disconnectTag`** (as the last parameter — position 8). No `?? throw ArgumentNullException` — the service is optional by design (stub behaviour when absent in design-time/test scenarios). **Do not place it before the required parameters `initialStoredTagIds`, `connectTag`, or `disconnectTag`** — doing so would produce CS1737 (optional parameter before required).
3. Implement `AddCategoryAsync`:
   ```csharp
   await (_categoryEditorService?.ShowAsync(null) ?? Task.CompletedTask).ConfigureAwait(false);
   ```
   (`ITagsManager.DataChanged` handles the `RebuildCategories` refresh automatically.)
4. Implement `EditCategoryAsync(TaggerCategoryViewModel? categoryVm)`:
   - Guard: if `categoryVm is null`, return.
   - `await (_categoryEditorService?.ShowAsync(categoryVm.Category) ?? Task.CompletedTask).ConfigureAwait(false)`.
5. Implement `DeleteCategoryAsync(TaggerCategoryViewModel? categoryVm)`:
   - Guard: if `categoryVm is null || categoryVm.Category.IsMostUsed`, return.
   - `await (_categoryEditorService?.ShowAsync(categoryVm.Category) ?? Task.CompletedTask).ConfigureAwait(false)` — the editor is the confirmation surface; the Delete button is disabled when `tagCount > 0`. When `_categoryEditorService` is null, the command is a no-op (stub behaviour preserved).

---

### WP-005 — Extend `MovieEditorViewModel`

1. Add `private readonly ICategoryEditorService? _categoryEditorService;` field alongside the other tag-service fields.
2. Add `ICategoryEditorService? categoryEditorService = null` parameter to the full constructor (nullable for design-time/test compat).
3. In `LoadAsync`, pass `_categoryEditorService` to the `TaggerViewModel` constructor:
   ```csharp
   TaggerVm = new TaggerViewModel(
       _tagsManager, _tagEditorService, _tagMergeService, _grantsManagementService,
       _originalMovie.StoredTagIds,
       connectTag:            (tagId, token) => _tagsManager.ConnectMovieTagAsync(movieId, tagId, token),
       disconnectTag:         (tagId, token) => _tagsManager.DisconnectMovieTagAsync(movieId, tagId, token),
       categoryEditorService: _categoryEditorService);  // ← new (named arg, optional position 8)
   ```
   Do **not** extend the `LoadAsync` null-guard to include `_categoryEditorService`. Pass it through unconditionally — the field is nullable by design. `TaggerViewModel`'s null-conditional call sites silently no-op when it is absent, which preserves the stub behaviour and ensures the existing `LoadAsync_InitializesTaggerVm_WhenTagsManagerProvided` test continues to pass without modification.
4. The DI factory for `MovieEditorViewModel` in `Program.cs` (`Func<Movie, MovieEditorViewModel>`) must add `ICategoryEditorService` to the resolved dependencies and pass it through.

---

### WP-006 — `TaggerView.axaml` category context menu

1. Replace the current `TabControl.ItemTemplate` DataTemplate:
   ```xml
   <DataTemplate x:DataType="vm:TaggerCategoryViewModel">
     <TextBlock Text="{Binding Category.Name}" />
   </DataTemplate>
   ```
   with a version that wraps the `TextBlock` in a panel with a `ContextMenu`:
   ```xml
   <DataTemplate x:DataType="vm:TaggerCategoryViewModel">
     <TextBlock Text="{Binding Category.Name}">
       <TextBlock.ContextMenu>
         <ContextMenu>
           <MenuItem Header="Edit Category"
                     Command="{Binding #Root.((vm:TaggerViewModel)DataContext).EditCategoryCommand}"
                     CommandParameter="{Binding}" />
           <MenuItem Header="Delete Category"
                     Command="{Binding #Root.((vm:TaggerViewModel)DataContext).DeleteCategoryCommand}"
                     CommandParameter="{Binding}" />
         </ContextMenu>
       </TextBlock.ContextMenu>
     </TextBlock>
   </DataTemplate>
   ```
   This follows the exact same root-binding pattern already used for the tag context menu in the same file.

---

### WP-007 — Fix CS8765 build warnings

1. In `tests/VideoIndexer.Tests/TagsManagerTests.cs` at lines 346 and 374, add `[System.Diagnostics.CodeAnalysis.AllowNull]` to each overriding property. Do **not** change the property type to `string?` — that alters the getter return type and may introduce new warnings. The CS8765 mismatch arises because the BCL abstract properties (`DbConnection.ConnectionString` and `DbCommand.CommandText`) carry `[AllowNull]` on their setters; the override must mirror this annotation. Correct form:
   ```csharp
   [System.Diagnostics.CodeAnalysis.AllowNull]
   public override string ConnectionString { get; set; } = string.Empty;
   ```
   Apply the same attribute to `CommandText` at line 374.

---

### WP-008 — Tests

1. In `tests/VideoIndexer.App.Tests/TaggerViewModelTests.cs`, add an optional `ICategoryEditorService? categoryEditorService = null` parameter to the `MakeTaggerVm` private helper and forward it to the `TaggerViewModel` constructor as the final argument (position 8, after `disconnectTag`). Existing call sites omit the argument (nullable default) and compile unchanged.
2. In the same file, add test cases covering:
   - `AddCategoryCommand` invokes `ICategoryEditorService.ShowAsync(null)`.
   - `EditCategoryCommand` invokes `ICategoryEditorService.ShowAsync(category)` for a non-Most-Used category.
   - `EditCategoryCommand` is a no-op when `categoryVm` is `null`.
   - `DeleteCategoryCommand` invokes `ICategoryEditorService.ShowAsync(category)` for a regular (non-protected) category.
   - `DeleteCategoryCommand` does **not** invoke the service when the category is the virtual Most Used category.
   Use `FakeTagDialogServices.cs` (already in `tests/VideoIndexer.App.Tests/TestHelpers/`) as the model for the new `ICategoryEditorService` fake.

---

### WP-009 — Manifest updates

Update:
- `docs/agents/project-manifest/api-surface.md` — add `ICategoryEditorService` signature under `VideoIndexer.Core — Abstractions`.
- `docs/agents/project-manifest/file-tree.md` — add entries for `ICategoryEditorService.cs` (Core/Abstractions) and `AvaloniaCategoryEditorService.cs` (App/Services); update `TaggerView.axaml` note to mention category context menu; update `TaggerViewModel.cs` note to remove "stub category commands".
- `docs/agents/project-manifest/constraints.md` — under the Tag Subsystem section, remove the two M7-backlog entries that are now resolved: (1) "`CategoryEditorViewModel` has no unit tests (test backlog)" — already resolved in M7 WP-020 when `CategoryEditorViewModelTests.cs` was implemented; the entry is stale and should be removed; (2) "`TagEditorView` `SelectedParentTag` ComboBox lacks a null-sentinel (must fix before M7 ships)" — already resolved by the `ClearParentTagCommand` Clear button that was implemented alongside `TagEditorView`; the constraint is stale.
- `docs/projects/rebuild/milestones/m7-tagging-core.md` — create this file using the milestone document template from `roadmap.md`.

---

## Dependencies

- `ICategoryEditorService` (WP-001) must land before `AvaloniaCategoryEditorService` (WP-002) and `TaggerViewModel` wiring (WP-004).
- `AvaloniaCategoryEditorService` (WP-002) must land before DI registration (WP-003).
- `TaggerViewModel` wiring (WP-004) must land before `MovieEditorViewModel` extension (WP-005).
- `ICategoryEditorService` must be defined before `FakeTagDialogServices` can be extended for tests (WP-008).
- All implementation WPs (001–007) must pass `dotnet build` before the test WP (WP-008).

---

## Required Components

### New files
| File | Layer | Purpose |
|------|-------|---------|
| `src/VideoIndexer.Core/Abstractions/ICategoryEditorService.cs` | Core | Service interface for category create/edit/delete dialog |
| `src/VideoIndexer.App/Services/AvaloniaCategoryEditorService.cs` | App | Avalonia modal dialog implementation |

### Modified files
| File | Change |
|------|--------|
| `src/VideoIndexer.App/Program.cs` | Add `ICategoryEditorService` DI registration |
| `src/VideoIndexer.App/ViewModels/TaggerViewModel.cs` | Add constructor param; implement 3 category commands |
| `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs` | Add `ICategoryEditorService?` field + ctor param + LoadAsync wiring |
| `src/VideoIndexer.App/Views/TaggerView.axaml` | Add category tab context menu |
| `tests/VideoIndexer.Tests/TagsManagerTests.cs` | Fix 2× CS8765 nullable annotation warnings |
| `tests/VideoIndexer.App.Tests/TaggerViewModelTests.cs` | Add category-command tests |
| `tests/VideoIndexer.App.Tests/TestHelpers/FakeTagDialogServices.cs` | Add `ICategoryEditorService` fake |

### Documentation
| File | Change |
|------|--------|
| `docs/agents/project-manifest/api-surface.md` | Add `ICategoryEditorService` |
| `docs/agents/project-manifest/file-tree.md` | Add new files; update notes |
| `docs/agents/project-manifest/constraints.md` | Remove two stale M7 backlog entries |
| `docs/projects/rebuild/milestones/m7-tagging-core.md` | Create milestone document |

---

## Assumptions

- `CategoryEditorView.axaml` is already ViewLocator-registered (`AddTransient<CategoryEditorView>` in `Program.cs`); the service will instantiate it directly, so no additional DI change is needed for the view.
- The `ITagsManager` singleton is always initialized by the time a user opens the Movie Editor (because `LoadAsync` calls `EnsureLoadedAsync` before constructing `TaggerViewModel`); `AvaloniaCategoryEditorService` can rely on `tagsManager.Tags` being populated when `ShowAsync` is called.
- `CategoryEditorView.axaml` already has `x:DataType="vm:CategoryEditorViewModel"` compiled bindings; no AXAML changes are required.
- The `x:Name="Root"` reference used by the tag context menu in `TaggerView.axaml` is on the root `UserControl` element (verified in source). Verify this name is accessible from within the `TabControl.ItemTemplate` data template — if not, the binding path must be adjusted using `ElementName` targeting the `TabControl` itself.

---

## Constraints

- No new NuGet packages.
- No new database migrations; schema revision stays at **40** (`m040_add_tag_uniqueness.sql` is the terminal migration for M7).
- `TreatWarningsAsErrors=true` — WP-007 (fix CS8765) must be resolved before the milestone is declared done, even if only the test project emits them.
- `ICategoryEditorService` must live in `VideoIndexer.Core/Abstractions/`, not `VideoIndexer.App/Services/`, per the interface-in-Core rule.
- `CancellationToken` must be accepted but may remain unobserved (Avalonia `ShowDialog` limitation; follow existing XML-doc convention).

---

## Out of Scope

- Inline category reordering (drag-and-drop tab sort) — not in the tagging spec.
- Category-level tag-count display inside the tab header — deferred; not in the tagging spec.
- Batch tag reassignment when a category is deleted (spec §3.7 requires the user to empty the category first; enforced by the `tagCount > 0` guard on Delete).
- M8 (System Tools), M9 (Images), M10 (Player & Bookmarks) — separate milestones.

---

## Acceptance Criteria

- [ ] The app launches cleanly with zero build errors and zero build warnings.
- [ ] After reaching `ShellState.Ready` and opening the Movie Editor, the right sidebar shows the Tagger with category tabs populated from the database.
- [ ] Clicking **Add Category** in the Tagger toolbar opens the Category Editor dialog in create mode; entering a name and saving creates the new category and the tab appears immediately.
- [ ] Right-clicking a non-protected category tab → **Edit Category** opens the Category Editor with the existing name pre-filled; renaming and saving updates the tab label immediately.
- [ ] Right-clicking a non-protected category tab → **Delete Category** opens the Category Editor; if the category has tags, the Delete button is disabled; if empty, clicking Delete removes the category.
- [ ] Right-clicking the **Most Used** virtual tab does not allow deletion (Delete Category command is inert for `IsMostUsed` categories).
- [ ] Clicking Delete on the Default Category (category with `tag_category_id = 1`) is not permitted (guarded by `IsReadOnly`).
- [ ] `dotnet test` passes with all tag-related test suites green.

---

## Testing Strategy

All new logic is in thin wiring (service construction + event bridging). Unit tests use the existing `FakeTagsManager` and a new `FakeCategoryEditorService` that captures `ShowAsync` call arguments and returns a configurable result. Integration tests (`DapperTagsRepositoryTests`) already cover the data layer. No additional integration tests are needed for this WP set.

---

## Test Plan

| Test file | What it asserts | Acceptance criterion |
|-----------|----------------|----------------------|
| `tests/VideoIndexer.App.Tests/TaggerViewModelTests.cs` — `AddCategoryCommand_CallsServiceWithNull` | `ICategoryEditorService.ShowAsync` receives `null` when the add-category toolbar button is invoked | AC: Create mode opens correctly |
| `tests/VideoIndexer.App.Tests/TaggerViewModelTests.cs` — `EditCategoryCommand_CallsServiceWithCategory` | `ICategoryEditorService.ShowAsync` receives the correct `TagCategory` when invoked for a regular category | AC: Edit mode opens with correct category |
| `tests/VideoIndexer.App.Tests/TaggerViewModelTests.cs` — `EditCategoryCommand_NullCategoryVm_DoesNothing` | No service call when `commandParameter` is `null` | AC: Null guard works |
| `tests/VideoIndexer.App.Tests/TaggerViewModelTests.cs` — `DeleteCategoryCommand_CallsServiceWithCategory` | `ICategoryEditorService.ShowAsync` receives the correct `TagCategory` when invoked for a deletable category | AC: Delete delegate opens editor |
| `tests/VideoIndexer.App.Tests/TaggerViewModelTests.cs` — `DeleteCategoryCommand_MostUsed_DoesNotCallService` | No service call when `IsMostUsed == true` | AC: Most Used guard works |
| `tests/VideoIndexer.Tests/TagsManagerTests.cs` — existing suite, 0 warnings | CS8765 warnings at lines 346 and 374 resolved | AC: Zero build warnings |

---

## Documentation Updates

- `docs/agents/project-manifest/api-surface.md` — Four updates required:
  1. Add `ICategoryEditorService` interface block under `VideoIndexer.Core — Abstractions` with the `ShowAsync` signature and return-value semantics.
  2. Add `AvaloniaCategoryEditorService : ICategoryEditorService` constructor entry in the `App/Services` section (following the same pattern as `AvaloniaTagEditorService` and `AvaloniaTagMergeService`).
  3. Update the `TaggerViewModel` constructor signature to add `ICategoryEditorService? categoryEditorService = null` at position 8 (after `disconnectTag`); remove the "stub — pending `ICategoryEditorService`" annotations from the three category command entries.
  4. Update the `MovieEditorViewModel` constructor signature to add `ICategoryEditorService? categoryEditorService = null` after `IGrantsManagementService? grantsManagementService = null`.
- `docs/agents/project-manifest/file-tree.md` — Add `ICategoryEditorService.cs` to the Abstractions directory listing; add `AvaloniaCategoryEditorService.cs` to the Services listing with M7 tag; update `TaggerViewModel.cs` description (remove "stub category commands"); update `TaggerView.axaml` note (add "category tab context menu").
- `docs/projects/rebuild/milestones/m7-tagging-core.md` — Create using the milestone template from `roadmap.md`; mark all work packages as Complete and include the done-means statement.

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **`x:Name="Root"` binding path inaccessible from `TabControl.ItemTemplate`** | `x:Name="Root"` is on the root `UserControl` element (not the `DockPanel`). The binding `#Root.((vm:TaggerViewModel)DataContext)` should be accessible from within the `TabControl.ItemTemplate` because the `UserControl` is an ancestor; verify at test time. If inaccessible, assign `x:Name` to the `TabControl` itself and reference it via `ElementName` in the context-menu command binding. |
| **`ShowDialog` called before owner window is visible** | `ownerFactory` returns `null` when no window is active; existing dialog services already handle this by skipping `ShowDialog` when `owner is null` — follow the same guard in `AvaloniaCategoryEditorService`. |
| **`DeleteRequested` fires while `TagsManager` reload is in progress** | `ITagsManager.DeleteCategoryAsync` acquires the internal `SemaphoreSlim` — it is safe to call from the service; no additional guard needed. |
| **`MovieEditorViewModel` DI factory signature change breaks existing tests** | `ICategoryEditorService?` is nullable with a `null` default; existing tests that construct the editor VM without it will continue to compile and run unmodified. |
| **`DeleteRequested` handler — `async void` and UI-thread-affinity risk** | ~~async void + try/catch~~ **Resolved by WP-002 design.** The handler is a synchronous one-liner (`deletePending = true; dialog.Close()`). The actual `await tagsManager.DeleteCategoryAsync(...)` call is deferred to after `await window.ShowDialog(owner)` returns, inside the already-`async Task ShowAsync` method body. This eliminates the `async void` unhandled-exception risk **and** the thread-affinity risk caused by `TagsManager.DeleteCategoryAsync`'s `ConfigureAwait(false)` continuation running on a thread-pool thread (which would throw if `dialog.Close()` were called there). |
