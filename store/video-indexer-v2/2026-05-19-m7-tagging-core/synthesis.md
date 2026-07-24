# Synthesis Report — M7: Tagging Core (Final Completion)

**Plan:** `2026-05-19-m7-tagging-core`
**Date:** 2026-05-19
**Status:** COMPLETE
**Work Packages:** 9 / 9 COMPLETE
**Pipeline Health:** 9/9 WPs with all stages passing — 0 missing stages

---

## Executive Summary

This session completed Milestone M7 (Tagging Core) by closing the final gap in the category management flow. `CategoryEditorViewModel` and `CategoryEditorView` were already implemented but had no wiring to make them visible. Nine targeted work packages delivered the full dialog-service pipeline — interface, concrete implementation, DI registration, ViewModel integration, view context menus, and comprehensive tests — bringing M7 to a fully shippable state.

The work followed the established four-file dialog-service pattern (`IXxxService` → `AvaloniaXxxService` → `Program.cs` registration → ViewModel wiring) without introducing any architectural deviation or new dependencies.

### Features Delivered

| WP | Deliverable |
|----|-------------|
| WP-001 | `ICategoryEditorService` interface in `VideoIndexer.Core/Abstractions/` |
| WP-002 | `AvaloniaCategoryEditorService` implementing the interface (create/edit/delete flows) |
| WP-003 | DI singleton registration in `Program.cs` (section 5.75, after `ITagMergeService`) |
| WP-004 | `TaggerViewModel` wired with optional `ICategoryEditorService?` parameter (position 8); three stub commands activated |
| WP-005 | `MovieEditorViewModel` propagates `ICategoryEditorService` through its DI factory and into `TaggerViewModel` via `LoadAsync` |
| WP-006 | `TaggerView.axaml` category tab-header context menu (Edit Category / Delete Category) using `#Root` element-name binding |
| WP-007 | CS8765 warning elimination in `TagsManagerTests.cs` via `[AllowNull]` on two stub property overrides |
| WP-008 | 5 new unit tests covering all three category command behaviors (add, edit, delete) including guard conditions |
| WP-009 | Full manifest documentation sweep + M7 milestone document created |

---

## Metrics

### Test Results

| Checkpoint | Tests Passed | Tests Failed | Notes |
|------------|-------------|-------------|-------|
| WP-001 QA (full suite) | 777 | 0 | 6 skipped (live integration) |
| WP-002 QA (full suite) | 777 | 0 | 6 skipped |
| WP-004 QA (full suite) | 777 | 0 | — |
| WP-005 QA (full suite) | **782** | 0 | +5 from WP-008 |
| WP-008 QA (App tests) | 340 | 0 | All 5 new category tests pass |
| **Final full suite** | **782** | **0** | **6 skipped (pre-existing)** |

### Build Quality

- `dotnet build src/VideoIndexer.sln` — **0 errors, 0 warnings** across all WPs
- Pre-existing CS8765 warnings (2) resolved in WP-007 — solution is now fully clean
- `TreatWarningsAsErrors=true` enforced throughout; no suppressions added to production code

### Files Delivered

**New source files (2):**
- `src/VideoIndexer.Core/Abstractions/ICategoryEditorService.cs`
- `src/VideoIndexer.App/Services/AvaloniaCategoryEditorService.cs`

**Modified source files (5):**
- `src/VideoIndexer.App/Program.cs`
- `src/VideoIndexer.App/ViewModels/TaggerViewModel.cs`
- `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs`
- `src/VideoIndexer.App/Views/TaggerView.axaml`
- `tests/VideoIndexer.Tests/TagsManagerTests.cs`

**New/modified test files (2):**
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeTagDialogServices.cs` (FakeCategoryEditorService spy added)
- `tests/VideoIndexer.App.Tests/TaggerViewModelTests.cs` (5 new tests)

**Documentation files (5):**
- `docs/agents/project-manifest/api-surface.md`
- `docs/agents/project-manifest/file-tree.md`
- `docs/agents/project-manifest/constraints.md`
- `docs/projects/rebuild/milestones/m7-tagging-core.md` *(new)*
- `docs/agents/plans/2026-05-19-m7-tagging-core/synthesis.md` *(this file)*

---

## Strategic Recommendations

### Gold Nuggets

1. **Interface asymmetry to align on next touch (low priority)**
   The `ICategoryEditorService.ShowAsync` signature uses `existingCategory = null` as a default parameter, while `ITagEditorService.ShowAsync` intentionally omits the default. The reviewer flagged this minor asymmetry as plan-spec correct but worth aligning if `ITagEditorService` is ever revisited.
   > *Action: Align both interfaces to use `= null` default when `ITagEditorService` is next touched.*

2. **`ITagEditorService` has the same "null-after-delete" documentation gap as `ICategoryEditorService`**
   WP-001 Documentation clarified `ICategoryEditorService.ShowAsync`'s return-value contract (implementation deletes before returning null; callers must not double-delete). The identical ambiguity exists in `ITagEditorService.ShowAsync` and was explicitly flagged as out-of-scope.
   > *Action: In a future WP or documentation pass, apply the same `<returns>` clarification to `ITagEditorService.ShowAsync`.*

3. **Stale constraints.md entry: `TagEditorViewModel` unit tests**
   WP-009 removed resolved M7 backlog entries but one related item was noted as potentially stale: the `TagEditorViewModel` "no unit tests" constraint. Code inspection suggests 8 tests already exist in `TagEditorViewModelTests.cs`.
   > *Action: Verify `TagEditorViewModelTests.cs` coverage and remove or update the constraint entry if resolved.*

4. **`#Root` element-name binding pattern now a documented convention**
   The reviewer identified this pattern as an undocumented idiom used in three places in `TaggerView.axaml`. WP-006 Documentation added it to `constraints.md` as the official project convention for binding parent-ViewModel commands from nested `DataTemplate`s. This proactively reduces future confusion.

5. **`MovieEditorViewModel` constructor arity (informational)**
   The full constructor now takes 13 parameters. While this is consistent with the established codebase pattern and the DI factory approach mitigates call-site friction, arity growth is worth monitoring. If M8 adds further optional services, consider whether a builder/options object pattern would improve maintainability.

---

## Next Steps

1. **M8 Planning:** Milestone M7 is fully complete and documented. The planning cycle can now focus on the next milestone in `docs/projects/rebuild/` roadmap.
2. **`ITagEditorService` null-after-delete doc (low effort):** Apply the clarified `<returns>` contract from WP-001 to `ITagEditorService.ShowAsync` in a standalone documentation WP.
3. **`TagEditorViewModel` constraints.md cleanup (low effort):** Verify and remove or update the "no unit tests" constraint entry.
4. **Regression monitoring:** The 6 consistently skipped tests (live infrastructure integration) should be reviewed periodically to confirm they remain intentionally skipped and have not become silently broken.
