# Plan

## Summary

This plan addresses all actionable items carried forward from the M5 Filters & Search synthesis report. The synthesis identified one **High** priority blocker (filter slot loading is never triggered from the view, making the DSL filter feature inert at runtime), plus six lower-priority tech debt items spanning input validation, UI polish, a test file misplacement, an over-specified test assertion, an implicit repository insert/update convention, and a DSL evaluator function-dispatch pattern that will become unwieldy in M7/M10. This plan resolves all items in priority order.

---

## Architectural Context

The relevant modules after M5 are:

- **`src/VideoIndexer.App/Views/MoviesListView.axaml.cs`** — code-behind; currently only calls `vm.LoadAsync()` in the `Loaded` handler; `vm.LoadFilterSlotsAsync()` is never called.
- **`src/VideoIndexer.App/ViewModels/MoviesListViewModel.cs`** — holds `FilterSlots`, `ActiveFilterSlot`, `LoadFilterSlotsAsync()`; the method loads and reconciles active-slot persistence.
- **`src/VideoIndexer.App/Views/FiltersManagerView.axaml`** — binds `SaveSlotCommand` to both "Add" and "Save Slot" buttons; no "New Slot / Clear Editor" command exists.
- **`src/VideoIndexer.App/Views/MainContentView.axaml`** — ComboBox for `ActiveFilterSlot` shows blank when no slot is active; no null-item placeholder DataTemplate.
- **`src/VideoIndexer.App/ViewModels/FiltersManagerViewModel.cs`** — no `ClearSelectionCommand`; editor fields must be cleared manually via selection change.
- **`src/VideoIndexer.Infrastructure/Library/DapperFilterSlotRepository.cs`** — `SaveAsync` does not guard against `slot.Name`/`slot.Expression` being null; a null value propagates to a DB exception.
- **`src/VideoIndexer.Core/Filtering/FilterExpressionEvaluator.cs`** — function dispatch uses a linear `if`-chain; all 8 functions are hardcoded.
- **`tests/VideoIndexer.Tests/FilterExpressionEvaluatorTests.cs`** — sits at the test-project root instead of the `Filtering/` subfolder where `FilterExpressionParserTests.cs` already lives.
- **`tests/VideoIndexer.Infrastructure.Tests/Database/DatabaseBootstrapperTests.cs`** — `ExpectedRevision_IsThirtySeven` is a magic-number constant assertion that must be manually renamed on every schema revision.
- **`src/VideoIndexer.Core/Abstractions/IFilterSlotRepository.cs`** — `SaveAsync` insert/update convention (`SlotId == 0` → insert) is implicit and not expressed in the type system.

---

## Approach / Architecture

Each action is a targeted, minimal change; no new abstractions or layers are introduced beyond what the synthesis recommends.

1. **Wire `LoadFilterSlotsAsync`** — one additional `await` call added alongside the existing `LoadAsync()` call in `MoviesListView.axaml.cs`. No ViewModel changes required; the VM already implements the method.

2. **`SaveAsync` null-guard** — add `ArgumentException.ThrowIfNullOrEmpty` guards for `slot.Name` and `slot.Expression` at the top of `DapperFilterSlotRepository.SaveAsync`, consistent with the `ArgumentNullException.ThrowIfNull(slot)` guard already present. Companion test added to `DapperFilterSlotRepositoryTests`.

3. **`ClearSelectionCommand` in `FiltersManagerViewModel`** — add a `[RelayCommand]` `ClearSelection()` method that sets `SelectedSlot = null` (which already triggers `OnSelectedSlotChanged` to blank the editor fields). Update `FiltersManagerView.axaml` to rename the existing "Add" button to "New" and bind it to `ClearSelectionCommand`. The "Save Slot" button remains the sole trigger for `SaveSlotCommand`. Companion unit test added.

4. **`ActiveFilterSlot` null-item placeholder** — add a `<ComboBox.PlaceholderText>` attribute (Avalonia 11 `PlaceholderText`) to the `ActiveFilterSlot` ComboBox in `MainContentView.axaml` so the user sees "— no filter —" when `ActiveFilterSlot` is null.

5. **`FilterExpressionEvaluator` dispatch table** — replace the linear `if`-chain in `EvaluateFunction` and `ResolveNumericIdentifier` with `static readonly Dictionary<string, ...>` dispatch tables using `StringComparer.OrdinalIgnoreCase`. No public API changes; no behaviour changes; existing 57-test suite remains the regression guard.

6. **Move `FilterExpressionEvaluatorTests.cs`** — move from `tests/VideoIndexer.Tests/` root to `tests/VideoIndexer.Tests/Filtering/` and update the `namespace` declaration from `VideoIndexer.Tests` to `VideoIndexer.Tests.Filtering` to match `FilterExpressionParserTests.cs`.

7. **Rename `ExpectedRevision_IsThirtySeven`** — rename the test method to `ExpectedRevision_MatchesCurrentSchemaRevision` and keep the existing assertion body (`Assert.Equal(37, DatabaseBootstrapper.ExpectedRevision)`) unchanged. The rename removes the manual-rename burden on every schema migration; the body continues to anchor the constant value. The sibling test `CheckAsync_RevisionMatchesExpected_ReturnsOk` (which hardcodes `dbRevision: "37"`) already validates the bootstrap happy-path for revision 37.

8. **`IFilterSlotRepository.SaveAsync` insert/update clarity** — add XML doc improvements to the interface signature clarifying the `SlotId == 0` → insert contract (the synthesis deferred the discriminated-union shape change; this plan adopts the lighter doc-improvement approach to reduce risk in a rework pass). This is a documentation-only change with no runtime impact.

---

## Rationale

- **Wiring `LoadFilterSlotsAsync` first** — this is the only change that makes M5 functionally complete at runtime. All other items are quality improvements.
- **`ArgumentException.ThrowIfNullOrEmpty`** (not `ArgumentNullException.ThrowIfNull`) for the name/expression guards: `""` is as invalid as `null` for a filter slot name; using the stricter check prevents empty-string rows in the DB.
- **`ClearSelectionCommand` over binding "Add" to `SaveSlotCommand`** — the current binding is semantically confusing ("Add" appears to save, which it accidentally does only when nothing is selected). A dedicated clear-selection command makes intent explicit and avoids accidental saves.
- **`PlaceholderText`** for the null-slot ComboBox — Avalonia 11 supports `PlaceholderText` natively; this is the least invasive fix and avoids adding a synthetic null-sentinel item to the collection.
- **Dispatch table** — replaces an O(n) linear scan with O(1) lookup and removes the need to touch `EvaluateFunction` or `ResolveNumericIdentifier` when M7/M10 add new functions. Dictionary entry addition is a one-liner, consistent with M7/M10 parser hook removal.
- **Move evaluator tests** — cosmetic but ensures the `Filtering/` folder is the canonical home for all DSL tests, reducing future contributor confusion.
- **Rename-only `ExpectedRevision` test** — removes the manual rename burden from every schema migration engineer. The assertion body (`Assert.Equal(37, DatabaseBootstrapper.ExpectedRevision)`) is preserved unchanged; a self-referential round-trip body would pass for any constant value (including `0`) and provides no regression protection for the constant itself.

---

## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|----------|--------------|-------------------------|-------------------|
| `LoadFilterSlotsAsync` wiring location | `MoviesListView.axaml.cs` `Loaded` handler, alongside `LoadAsync` | Wire in `MoviesListViewModel.LoadAsync` directly | View-side wiring keeps the VM independently testable; `LoadAsync` is already tested without filter-slot loading, and the synthesis explicitly named the view code-behind as the target location. |
| Null-slot ComboBox placeholder | `PlaceholderText` attribute | Add a null sentinel `FilterSlot` to the `FilterSlots` collection | A sentinel item requires ViewModel logic to exclude it from saving/activating; `PlaceholderText` is pure XAML with no VM impact. |
| `ClearSelectionCommand` | New `[RelayCommand]` `ClearSelection()` on `FiltersManagerViewModel` | Separate "New" and "Save" commands as distinct RelayCommands | A single `ClearSelection` that sets `SelectedSlot = null` reuses the existing `OnSelectedSlotChanged` callback that already blanks all editor fields; minimal new code. |
| DSL dispatch table | `static readonly Dictionary<string, Func<...>>` per dispatch site | Keep linear if-chain | Dictionary lookup is O(1); extension for M7/M10 is a single `Add()` call rather than a new `if` branch; no behaviour change means zero test risk. |
| `IFilterSlotRepository.SaveAsync` insert/update | XML doc improvement only | Introduce separate `InsertAsync`/`UpdateAsync` methods | The discriminated-union split would require updating all callers (`FiltersManagerViewModel`, integration tests, fakes) — disproportionate scope for a rework plan. Doc improvement achieves the discoverability goal at zero risk. |
| `ExpectedRevision` test fix | Rename method only; keep `Assert.Equal(37, DatabaseBootstrapper.ExpectedRevision)` body | (a) Replace body with self-referential `ExpectedRevision.ToString()` round-trip; (b) delete the test entirely | A self-referential round-trip body passes for any constant value (including `0`), providing zero regression protection for the constant. Rename-only achieves the "no magic number in method name" goal without sacrificing the constant-value regression guard. Option (b) loses the regression signal entirely. |

---

## Pattern Alignment

| Pattern | Status |
|---------|--------|
| Code-behind wires view lifecycle hooks | `MoviesListView.axaml.cs` `Loaded` pattern — followed exactly (adding one `await` call alongside the existing `LoadAsync` call). |
| `[RelayCommand]` with `[ObservableProperty]` in ViewModels | `FiltersManagerViewModel.cs` — followed; `ClearSelectionCommand` uses the same `[RelayCommand]` attribute as all existing commands. |
| `ArgumentNullException.ThrowIfNull` / `ArgumentException.ThrowIfNullOrEmpty` guards at service boundaries | `DapperFilterSlotRepository.SaveAsync` — aligning with the existing `ThrowIfNull(slot)` guard already on the same method. |
| `static readonly Dictionary` dispatch tables | No prior example in `FilterExpressionEvaluator`; introduces a new pattern consistent with standard C# best practice. Justified by M7/M10 extension story. |
| Test namespace matches subfolder | `tests/VideoIndexer.Tests/Filtering/FilterExpressionParserTests.cs` uses `VideoIndexer.Tests.Filtering` — `FilterExpressionEvaluatorTests.cs` will be brought into alignment. |

---

## Detailed Steps

1. **Wire `LoadFilterSlotsAsync` in `MoviesListView.axaml.cs`**
   - Open `src/VideoIndexer.App/Views/MoviesListView.axaml.cs`.
   - In the `Loaded` lambda, after `_ = vm.LoadAsync();`, add `_ = vm.LoadFilterSlotsAsync();`.

2. **Add `SaveAsync` null/empty guards in `DapperFilterSlotRepository`**
   - Open `src/VideoIndexer.Infrastructure/Library/DapperFilterSlotRepository.cs`.
   - After the existing `ArgumentNullException.ThrowIfNull(slot);` at the top of `SaveAsync`, add:
     ```csharp
     ArgumentException.ThrowIfNullOrEmpty(slot.Name, nameof(slot.Name));
     ArgumentException.ThrowIfNullOrEmpty(slot.Expression, nameof(slot.Expression));
     ```
   - Update the `<inheritdoc />` XML comment on the interface to document that `ArgumentException` is thrown for null/empty `Name` or `Expression`.

3. **Add `ClearSelectionCommand` to `FiltersManagerViewModel`**
   - Add a new `[RelayCommand]` method `ClearSelection()` to `FiltersManagerViewModel.cs`:
     ```csharp
     [RelayCommand]
     private void ClearSelection() => SelectedSlot = null;
     ```
   - This reuses `OnSelectedSlotChanged` to blank all editor fields.

4. **Update `FiltersManagerView.axaml`**
   - Change the "Add" `<Button>` in the left column's `<StackPanel>`:
     - Change `Content="Add"` to `Content="New"`
     - Change `Command="{Binding SaveSlotCommand}"` to `Command="{Binding ClearSelectionCommand}"`

5. **Add `PlaceholderText` to the `ActiveFilterSlot` ComboBox in `MainContentView.axaml`**
   - Locate the `<ComboBox ItemsSource="{Binding MoviesListVm.FilterSlots}" ...>` element.
   - Add the attribute `PlaceholderText="— no filter —"`.

6. **Replace linear if-chain with dispatch tables in `FilterExpressionEvaluator.cs`**
   - Replace `ResolveNumericIdentifier` with a `static readonly Dictionary<string, Func<MovieListItem, decimal>>` keyed on identifier name (OrdinalIgnoreCase), populated with all 5 numeric identifiers.
   - Replace `EvaluateFunction` with a `static readonly Dictionary<string, Func<FunctionCallNode, MovieListItem, bool>>` keyed on function name (OrdinalIgnoreCase), populated with all 8 functions.
   - Both methods become single-expression lookups that throw `FilterExpressionException` when the key is absent.

7. **Move `FilterExpressionEvaluatorTests.cs` to `Filtering/` subfolder**
   - Move `tests/VideoIndexer.Tests/FilterExpressionEvaluatorTests.cs` → `tests/VideoIndexer.Tests/Filtering/FilterExpressionEvaluatorTests.cs`.
   - Update `namespace VideoIndexer.Tests;` → `namespace VideoIndexer.Tests.Filtering;`.

8. **Rename `DatabaseBootstrapperTests.ExpectedRevision_IsThirtySeven`**
   - Rename the test method to `ExpectedRevision_MatchesCurrentSchemaRevision`.
   - Keep the existing assertion body unchanged: `Assert.Equal(37, DatabaseBootstrapper.ExpectedRevision)`.
   - The rename removes the manual-rename burden on future schema migrations. The sibling test `CheckAsync_RevisionMatchesExpected_ReturnsOk` (which hardcodes `BuildSut(dbRevision: "37")`) continues to anchor the schema revision through the bootstrap happy-path check; no two-speed inconsistency is introduced.

9. **Update `IFilterSlotRepository` XML documentation**
   - Update the `SaveAsync` XML comment to explicitly state:
     > Throws `ArgumentException` when `slot.Name` or `slot.Expression` is null or empty.
   - Update `api-surface.md` to reflect the new thrown exception and the `ClearSelectionCommand` addition to `FiltersManagerViewModel`.

---

## Dependencies

- Steps 3 and 4 are coupled (command added before AXAML is updated).
- Step 6 depends only on the existing `FilterExpressionEvaluator.cs`; all existing 57 tests serve as regression guard and must remain green.
- Step 7 is independent; only the file location and namespace change.
- Step 9 must be done after steps 2 and 3.

---

## Required Components

- `src/VideoIndexer.App/Views/MoviesListView.axaml.cs` — modified (step 1)
- `src/VideoIndexer.Infrastructure/Library/DapperFilterSlotRepository.cs` — modified (step 2)
- `src/VideoIndexer.Core/Abstractions/IFilterSlotRepository.cs` — modified (steps 2, 9)
- `src/VideoIndexer.App/ViewModels/FiltersManagerViewModel.cs` — modified (step 3)
- `src/VideoIndexer.App/Views/FiltersManagerView.axaml` — modified (step 4)
- `src/VideoIndexer.App/Views/MainContentView.axaml` — modified (step 5)
- `src/VideoIndexer.Core/Filtering/FilterExpressionEvaluator.cs` — modified (step 6)
- `tests/VideoIndexer.Tests/Filtering/FilterExpressionEvaluatorTests.cs` — moved from root + namespace updated (step 7)
- `tests/VideoIndexer.Infrastructure.Tests/Database/DatabaseBootstrapperTests.cs` — modified (step 8)
- `docs/agents/project-manifest/api-surface.md` — updated (step 9)

---

## Assumptions

- Avalonia 11.3.14 (already in use) supports `PlaceholderText` on `ComboBox` for the null-slot placeholder. If not supported, the fallback is a `<TextBlock>` overlay bound to `IsVisible="{Binding MoviesListVm.ActiveFilterSlot, Converter={x:Static ObjectConverters.IsNull}}"`.
- The `FilterExpressionEvaluatorTests.cs` move does not require any `.csproj` changes because test discovery is assembly-wide and the file is already included via the default `**/*.cs` glob.
- `ArgumentException.ThrowIfNullOrEmpty` is available in .NET 10 (it was introduced in .NET 7 — confirmed available).

---

## Constraints

- No new NuGet packages.
- No schema changes; schema remains at revision 37.
- No ViewModel constructor or DI changes; `ClearSelectionCommand` is a ViewModel method, not a DI-injected service.
- All warnings are errors (`TreatWarningsAsErrors=true` inherited by all non-test projects). New code must compile cleanly.
- The `FilterExpressionEvaluator` dispatch table refactor must preserve all existing observable behaviors (same 57 tests must pass; no exception messages changed).

---

## Out of Scope

- M7 and M10 tag/bookmark DSL function activation — deferred identifiers remain gated in the parser.
- `ApplyFilter()` performance optimization (rebuild-on-keystroke) — deferred to a large-catalog milestone.
- `SharedConnectionFactory + NonDisposingConnection` pattern changes to other repositories.
- Any schema migration.
- macOS FFmpeg archive layout verification.
- `ProvisioningToolsViewModel` command name verification.

---

## Acceptance Criteria

- AC-1: The DSL filter slot ComboBox and active-slot indicator in `MainContentView` are populated on application load (i.e., `LoadFilterSlotsAsync` is called from the `Loaded` handler).
- AC-2: Calling `DapperFilterSlotRepository.SaveAsync` with a slot whose `Name` is null or empty throws `ArgumentException` before any SQL is executed.
- AC-3: Calling `DapperFilterSlotRepository.SaveAsync` with a slot whose `Expression` is null or empty throws `ArgumentException` before any SQL is executed.
- AC-4: `FiltersManagerView` "New" button clears the editor fields and deselects the current slot without saving.
- AC-5: `FiltersManagerView` "Save Slot" is the only button that triggers a save.
- AC-6: The `ActiveFilterSlot` ComboBox in `MainContentView` displays a placeholder when no filter is active (no blank entry).
- AC-7: `FilterExpressionEvaluator` uses dispatch tables; all 57 existing evaluator tests pass without modification.
- AC-8: `FilterExpressionEvaluatorTests.cs` resides in `tests/VideoIndexer.Tests/Filtering/` with namespace `VideoIndexer.Tests.Filtering`.
- AC-9: `DatabaseBootstrapperTests` contains no test method named `ExpectedRevision_IsThirtySeven`; the renamed method retains the `Assert.Equal(37, DatabaseBootstrapper.ExpectedRevision)` assertion body unchanged.
- AC-10: `IFilterSlotRepository.SaveAsync` XML comment documents the `ArgumentException` thrown on null/empty `Name`/`Expression`.
- AC-11: Full `dotnet test` run passes with ≥ 496 tests (the M5 baseline), zero failures, zero regressions.

---

## Testing Strategy

Existing test suites (496 tests across three assemblies) serve as the regression harness for the non-additive refactors (evaluator dispatch table, file move, test rename). New unit tests are added for the two behavioral changes (null guard, `ClearSelectionCommand`). Integration tests for `DapperFilterSlotRepository` already exist in `VideoIndexer.Infrastructure.Tests`; one new test case is added there.

---

## Test Plan

- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperFilterSlotRepositoryTests.cs` — `SaveAsync_NullName_ThrowsArgumentException` — AC-2
- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperFilterSlotRepositoryTests.cs` — `SaveAsync_EmptyName_ThrowsArgumentException` — AC-2
- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperFilterSlotRepositoryTests.cs` — `SaveAsync_NullExpression_ThrowsArgumentException` — AC-3
- `tests/VideoIndexer.Infrastructure.Tests/Library/DapperFilterSlotRepositoryTests.cs` — `SaveAsync_EmptyExpression_ThrowsArgumentException` — AC-3
- `tests/VideoIndexer.App.Tests/FiltersManagerViewModelTests.cs` — `ClearSelectionCommand_ClearsEditorAndDeselects` — AC-4
- `tests/VideoIndexer.Tests/Filtering/FilterExpressionEvaluatorTests.cs` (moved) — all 57 existing evaluator tests — AC-7, AC-8
- `tests/VideoIndexer.Infrastructure.Tests/Database/DatabaseBootstrapperTests.cs` — `ExpectedRevision_MatchesCurrentSchemaRevision` (renamed) — AC-9

---

## Documentation Updates

- `docs/agents/project-manifest/api-surface.md` — Update `IFilterSlotRepository.SaveAsync` note to document `ArgumentException` for null/empty `Name`/`Expression`; add `ClearSelectionCommand` to `FiltersManagerViewModel` signature block.
- `docs/agents/project-manifest/constraints.md` — Update the "Open Items / Known Gotchas" section: remove the "`MoviesListView` — `LoadFilterSlotsAsync` not yet wired" entry (AC-1 closes it); remove the `FiltersManagerView` "view not yet created" entry (already closed by M5 — this is stale). Note that `DapperFilterSlotRepository.SaveAsync` now guards null/empty `Name`/`Expression`.
- `docs/agents/project-manifest/file-tree.md` — Update `FilterExpressionEvaluatorTests.cs` reference from test-project root to `Filtering/` subfolder.

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **`PlaceholderText` not supported on `ComboBox` in Avalonia 11.3.14** | Verify against Avalonia 11 docs before implementation; fallback is a transparent `TextBlock` overlay bound to `IsVisible` with `ObjectConverters.IsNull`. This fallback must be noted as an acceptable alternative in the WP if confirmed unsupported. |
| **Moving `FilterExpressionEvaluatorTests.cs` breaks test discovery** | The test project uses default `**/*.cs` glob inclusion; moving the file within the same project directory keeps it in scope. Verified by confirming `FilterExpressionParserTests.cs` in the `Filtering/` subfolder is already discovered (it is — 13 tests run from that path). |
| **Dispatch table introduces a subtle behaviour change (e.g., different exception message)** | The 57 existing evaluator tests include error-path tests. If any test asserts the exact exception message string, the refactor must preserve the message. Review test assertions before refactoring. |
| **`ClearSelectionCommand` binding breaks compilation** due to Avalonia compiled-binding strict mode | `FiltersManagerView.axaml` uses `x:DataType="vm:FiltersManagerViewModel"` compiled bindings. The new command will be emitted by the CommunityToolkit source generator as `ClearSelectionCommand` — verify the source generator output after adding the method. Build before submitting the WP as done. |
