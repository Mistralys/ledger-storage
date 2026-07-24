# Synthesis Report — M5 Filters & Search Rework (Pass 1)

**Plan:** `2026-05-12-m5-filters-search-rework-1`
**Date:** 2026-05-13
**Status:** COMPLETE — 8/8 work packages PASS across 3 pipeline stages each (or 1 for WP-008)

---

## Executive Summary

This plan resolved all actionable carry-forwards from the M5 Filters & Search synthesis, converting a feature that was architecturally complete but **inert at runtime** into a fully functional filter slot system. The single High-priority blocker — `LoadFilterSlotsAsync` was never called from the view — is resolved in WP-001. The remaining seven work packages address input validation, UI polish, test organisation, a test naming convention, and a DSL evaluator refactor that unblocks M7/M10 extension.

**What was built:**
1. Filter slot loading now fires on app startup — `FilterSlots` populates and `ActiveFilterSlot` reflects the persisted selection.
2. `DapperFilterSlotRepository.SaveAsync` validates `Name` and `Expression` before any SQL executes — null/empty values are rejected with `ArgumentException`.
3. `FiltersManagerViewModel` has a `ClearSelectionCommand`; the "Add" button is renamed "New" and wired to it.
4. The `ActiveFilterSlot` ComboBox shows "— no filter —" placeholder when no slot is selected.
5. `FilterExpressionEvaluator` now dispatches via `static readonly Dictionary` tables — O(1) lookup, zero closure allocations, M7/M10 extension is a single `Add()` call per new function.
6. `FilterExpressionEvaluatorTests.cs` is correctly located in `tests/VideoIndexer.Tests/Filtering/` with namespace `VideoIndexer.Tests.Filtering`.
7. `ExpectedRevision_IsThirtySeven` is renamed `ExpectedRevision_MatchesCurrentSchemaRevision` — future schema migrations no longer require a test rename.
8. `IFilterSlotRepository.SaveAsync` XML doc and `api-surface.md` are updated to reflect the input validation contract and the new `ClearSelectionCommand`.

---

## Metrics

| WP | Scope | Tests Passed | Tests Failed | Files Modified |
|---|---|---|---|---|
| WP-001 | Wire `LoadFilterSlotsAsync` in Loaded handler | 500 (full suite) | 0 | 2 |
| WP-002 | Move `FilterExpressionEvaluatorTests.cs` to `Filtering/` | 56 (evaluator) / 197 (project) | 0 | 1 |
| WP-003 | Add `ClearSelectionCommand`, rename "Add"→"New" | 17 (FiltersManager) | 0 | 3 |
| WP-004 | Add `PlaceholderText` to `ActiveFilterSlot` ComboBox | 0 (AXAML-only) | 0 | 1 |
| WP-005 | Rename `ExpectedRevision_IsThirtySeven` test | 6 (DatabaseBootstrapper) | 0 | 1 |
| WP-006 | `FilterExpressionEvaluator` dispatch table refactor | 56 (evaluator) | 0 | 1 |
| WP-007 | `SaveAsync` null/empty guards + 4 guard tests | 14 (DapperFilterSlotRepository) | 0 | 2 |
| WP-008 | XML doc + `api-surface.md` updates | 0 (doc-only) | 0 | 2 |

**Full suite at close of session:** 500 pass / 0 fail / 6 skip (confirmed by WP-001 QA).
**Rework cycles:** 0 across all 8 WPs. Every pipeline passed on first attempt.
**Pipeline health:** 8/8 WPs with all stages passing; 0 missing stages.

---

## Debt & Observations Carried Forward

All items are low priority. No blockers remain.

### 1. Async Void Exception Propagation in `MoviesListView.axaml.cs`
**Flagged by:** Developer (WP-001), QA (WP-001), Reviewer (WP-001)
The `Loaded` event handler is `async void` (required by Avalonia). If `LoadAsync()` or `LoadFilterSlotsAsync()` throws after the first `await`, the exception propagates into Avalonia's unhandled-exception path rather than being caught locally. The previous fire-and-forget pattern was strictly worse (exceptions were fully silenced), so this is an improvement, not a regression.
**Recommendation:** In a future lifecycle-hardening WP, add a `try/catch` around the awaited calls in the `Loaded` handler and surface failures to the ViewModel's `HasLoadError` / `LoadErrorMessage` state.

### 2. No `CancellationToken` on Startup Init Call
**Flagged by:** QA (WP-001), Reviewer (WP-001)
`LoadFilterSlotsAsync` is called with `CancellationToken.None` — there is no cancellation path if the view is unloaded mid-startup.
**Recommendation:** Address alongside item 1 above in the lifecycle-hardening pass.

### 3. Pre-existing Cosmetic Debt in `FiltersManagerView.axaml`
**Flagged by:** Developer (WP-003)
`HorizontalAlignment='Stretch'` on buttons inside a `Horizontal` StackPanel has no visual effect — buttons size to content regardless. Not introduced by this plan.
**Recommendation:** Fix opportunistically during a future UI polish pass.

### 4. `NeverCalledConnectionFactory` Is a Single-Use Inner Class
**Flagged by:** Developer (WP-007), Reviewer (WP-007)
The `NeverCalledConnectionFactory` test helper is currently an inner class in `DapperFilterSlotRepositoryTests.cs`. The pattern is sound and self-documenting, but if other repository tests need it, duplication will accrue.
**Recommendation:** Extract to a shared `TestHelpers/` file when a second usage appears. No action needed now.

### 5. Plan Spec Off-by-One for `FilterExpressionEvaluatorTests` Count
**Flagged by:** QA (WP-002, WP-006), Reviewer (WP-002, WP-006)
The plan spec states 57 evaluator tests in both WP-002 (AC-6d) and WP-006 (AC-5c). The actual count is 56. All 56 pass — no tests were lost. The intent of both ACs is fully satisfied.
**Recommendation:** Update the plan spec for accuracy; no code change needed.

---

## Strategic Recommendations

### Gold Nuggets

**1. Dispatch Table Pattern Is Now Established in `FilterExpressionEvaluator`**
The M7/M10 extension story for the DSL evaluator is straightforward: adding a new boolean function requires one entry in `s_functions`; adding a new numeric identifier requires one entry in `s_numericIdentifiers`. The Reviewer confirmed all lambdas are `static` (no closures), `OrdinalIgnoreCase` is consistent across both tables, and error paths correctly throw `FilterExpressionException(FilterExpressionPhase.Evaluate, ...)`. The architecture is locked in correctly for scale.

**2. `NeverCalledConnectionFactory` Is a High-Value Test Pattern**
The guard test pattern used in WP-007 — a connection factory that throws `InvalidOperationException` if the DB is ever reached — provides proof-positive that `ArgumentException` guards fire before any SQL. This pattern should be documented in the project manifest and reused for any future repository guard tests.

**3. The Filter Slot Feature Is Now Runtime-Complete**
WP-001 is the only change that had direct user-visible impact. After this plan, the entire filter slot lifecycle — load on startup, create/update/delete via `FiltersManagerViewModel`, persist active selection via `IActiveFilterSlotRepository`, reflect in the `MainContentView` ComboBox — is wired and functional. M6 can add new filter functions and UI features against a working foundation.

---

## Next Steps

1. **Lifecycle hardening** — wrap `Loaded` handler awaits in a `try/catch`; add `CancellationToken` support to `LoadFilterSlotsAsync`. Appropriate for a small standalone WP.
2. **M7/M10 DSL extension** — add new filter functions by appending entries to `s_functions` in `FilterExpressionEvaluator`. The dispatch table architecture is ready.
3. **`IFilterSlotRepository.SaveAsync` discriminated-union shape** — the plan doc-improved the `SlotId == 0 → insert` convention. If the schema or caller contract changes, consider the deferred `InsertAsync`/`UpdateAsync` split (tracked but not actioned in this plan).
4. **`FilterExpressionEvaluatorTests.cs` spec count** — update plan spec to reflect 56 (not 57) tests for accuracy in future references.
5. **UI polish** — resolve the `HorizontalAlignment='Stretch'` on buttons in `FiltersManagerView.axaml` Horizontal StackPanel during the next UI pass.
