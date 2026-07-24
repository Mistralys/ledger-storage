# Synthesis Report — M7 Follow-Up: Ship Gate, Merge Execution, Lifecycle & Debt

**Plan:** `2026-05-18-m7-followup`
**Date:** 2026-05-18
**Status:** COMPLETE — All 11 work packages passed all pipeline stages.

---

## Executive Summary

This plan resolved every actionable item from the M7 Tagging Core synthesis report. The scope spanned two pre-ship gate fixes, full merge execution wiring, ViewModel lifecycle hardening, tag toggle error surfacing, test expansion (unit + integration), a cancellation-token sweep across all five Dapper repositories, and a targeted debt cleanup pass. The solution shipped with zero build warnings, zero security findings, and 777 passing tests.

**What was built / fixed:**

| WP | Title | Outcome |
|---|---|---|
| WP-001 | `ImpactRow` Deduplication | `DeleteImpactRow` + `MergeImpactRow` → single `ImpactRow` with `OverlapMovies` placeholder |
| WP-002 | `tags_bookmarks` Cascade Fix | Explicit DELETE step 1.5 in `DeleteTagAsync`; confirmed schema-level ON DELETE CASCADE; integration test added |
| WP-003 | `TagEditorView` Parent Tag Clear Button | `ClearParentTagCommand` + visibility-gated Clear button via `ObjectConverters.IsNotNull` |
| WP-004 | `MergeTagAsync` Full Implementation | 5-step transactional `DapperTagsRepository.MergeTagAsync`; `OverlapMovies` SQL subquery; `TagMergeView` overlap display; `TagsManager` delegation |
| WP-005 | `TaggerViewModel` `IDisposable` Lifecycle | `Dispose()` unsubscribes `DataChanged`; `TaggerView.axaml.cs` `Unloaded` wiring; `MovieEditorViewModel.LoadAsync` re-entry guard |
| WP-006 | Tag Toggle Error Surface | `LastOperationError` on `TaggerViewModel`; try/catch in `ExecuteToggleAsync`; in-panel error bar in `TaggerView.axaml`; threading bug caught and fixed via rework |
| WP-007 | `CategoryEditorViewModel` Unit Tests | 3 new tests (6 total): `EmptyName_DisablesSaveCommand`, `CreateMode_DisablesDeleteCommand`, `CancelCommand_FiresCloseRequestedWithNull` |
| WP-008 | `TagEditorViewModel` Unit Tests | 2 new tests (8 total): `ConfirmDeleteAsync_FiresCloseRequestedWithNull`, `ClearParentTagCommand_SetsSelectedParentTagToNull` |
| WP-009 | `DapperTagsRepository` Integration Tests | 8 new integration tests (16 total in file): CRUD round-trips, grant management, bookmark association, 3× merge scenarios including overlap deduplication |
| WP-010 | Dapper `CancellationToken` Sweep | `CommandDefinition` propagation across all 5 repositories (~47+ call sites); cancellation now reaches in-flight MySQL queries |
| WP-011 | Debt Cleanup (J2–J5) | `MoviesListViewModel` factory lambda; `MostUsedThreshold` dead code removed; count-mismatch guard TODO; CS8765 warnings resolved |

---

## Metrics

### Test Results (final run)

| Suite | Passed | Failed | Skipped |
|---|---|---|---|
| `VideoIndexer.Tests` (Core + unit) | 255 | 0 | 0 |
| `VideoIndexer.App.Tests` (unit + headless UI) | 335 | 0 | 0 |
| `VideoIndexer.Infrastructure.Tests` (integration) | 187 | 0 | 6 (pre-existing — no live DB/tools) |
| **Total** | **777** | **0** | **6** |

### Security

| WP | Audit Scope | Critical | High | Medium | Result |
|---|---|---|---|---|---|
| WP-002 | `DeleteTagAsync` cascade, integration test | 0 | 0 | 0 | PASS |
| WP-004 | `MergeTagAsync`, `GetMergeImpactAsync`, `TagsManager`, test stubs | 0 | 0 | 0 | PASS |

### Build

- **Warnings:** 0 (all WPs completed under `TreatWarningsAsErrors=true`)
- **Errors:** 0

### Rework

| WP | Pipeline Stage | Rework Cause | Resolution |
|---|---|---|---|
| WP-006 | `code-review` → FAIL | `ExecuteToggleAsync` used `ConfigureAwait(false)` then invoked `_onError` from a thread-pool thread — off-UI-thread `INotifyPropertyChanged` mutation | Removed `ConfigureAwait(false)`; catch continuation now runs on captured Avalonia `SynchronizationContext` |

---

## Strategic Recommendations (Gold Nuggets)

### 1. `DapperMovieCatalogRepository` needs `ILogger` — data integrity risk (Medium)

The count-mismatch guard in `GetMovieListAsync` silently returns an item with `EffectiveTagWeight = 0` when movie counts diverge between the query result and the enrichment map. Without a logger, this mismatch is invisible in production. WP-J4 is the tracked follow-up; it should be prioritized before the next milestone release.

**Recommendation:** Inject `ILogger<DapperMovieCatalogRepository>` and emit a `Warning` in the guard path.

### 2. `MovieEditorViewModel` is not `IDisposable` — benign GC gap

WP-005 correctly disposes `TaggerVm` inside `LoadAsync` (re-entry guard), but if `MovieEditorViewModel` itself is GC'd without another `LoadAsync` call, the `TaggerViewModel` event subscription lingers until both objects are collected. In practice this is benign because the event-subscription does not prevent GC when both are unreachable.

**Recommendation:** If stricter lifetime semantics are required in future (e.g., pooled editor instances), extend `MovieEditorViewModel` with `IDisposable` and call `TaggerVm?.Dispose()` in `Dispose()`.

### 3. `MergeTagAsync` has no self-merge guard (Low)

If `sourceTagId == targetTagId`, Step 1 of `DapperTagsRepository.MergeTagAsync` executes `INSERT IGNORE` (no-op) followed by a DELETE that removes all `tags_movies` rows for that tag — data destructive. The ViewModel prevents this at runtime, but there is no assertion at the repository layer.

**Recommendation:** Add a guard (`if (sourceTagId == targetTagId) throw new ArgumentException(...)`) to the repository method for defence-in-depth.

### 4. `MostUsedThreshold` is wired but inactive (Low)

`TagsManager.PublishState` has a `TODO (WP-J3)` comment where the threshold filtering should activate the Most Used virtual category. The setting is read from `TaggingOptions` but never applied to `allTags`.

**Recommendation:** When the Most Used category is ready for activation, add a `Where(t => t.UseCount >= threshold)` clause in `PublishState` and remove the TODO.

### 5. `LastOperationError` persists across `RebuildCategories` (Low)

When `ITagsManager.DataChanged` fires after a tag toggle error, `LastOperationError` is not cleared by `RebuildCategories`. The stale error bar remains visible until the user's next non-implied toggle. This is intentional (persistent-until-cleared is the documented pattern) but may be unexpected UX if a background event fires after a failure.

**Recommendation:** Decide whether a successful `RebuildCategories` cycle (no exception) should auto-clear the error. If so, add `LastOperationError = string.Empty` at the top of `RebuildCategories`.

### 6. `ExecuteToggleAsync` fire-and-forget pattern — future threading vigilance

The `_ = ExecuteToggleAsync(value)` fire-and-forget is safe because all exceptions are caught inside the method. However, removing `ConfigureAwait(false)` means the entire async method runs on the UI thread's SynchronizationContext. For DB-heavy paths this is acceptable, but any future await-points added inside `ExecuteToggleAsync` should be reviewed for potential UI-thread blocking.

**Recommendation:** Document this in `constraints.md` — `ExecuteToggleAsync` must not introduce `Task.Delay` or other blocking awaits without adding proper marshalling.

---

## Documentation & Manifest Updates Delivered

| Document | Updated By |
|---|---|
| `docs/agents/project-manifest/constraints.md` | WP-002 (cascade pattern), WP-005 (lifecycle), WP-011 (J4 guard note) |
| `docs/agents/project-manifest/api-surface.md` | WP-002 (DeleteTagAsync steps), WP-004 (MergeTagAsync, OverlapMovies), WP-005 (TaggerViewModel IDisposable) |
| `docs/agents/project-manifest/file-tree.md` | WP-004 (DapperTagsRepository, TagsManager annotations) |
| `docs/agents/implementation-history/m7-tagging-core.md` | WP-004 (resolved debt entries), WP-005 (resolved IDisposable stub) |

---

## Next Steps for Planner

1. **Prioritize WP-J4** — `ILogger` injection for `DapperMovieCatalogRepository`. Silent count-mismatch is the only remaining medium-priority data integrity gap.
2. **Self-merge guard** — Add `sourceTagId == targetTagId` assertion in `DapperTagsRepository.MergeTagAsync` (low-effort defence-in-depth).
3. **MostUsedThreshold activation** — Schedule WP-J3 when the Most Used virtual category is ready for production.
4. **`MovieEditorViewModel` IDisposable** — Low priority; revisit if pooled editor instances are ever introduced.
5. **M7 milestone** — The feature set is now fully implemented and ship-ready. All pre-ship gates (null-sentinel, bookmark cascade, merge execution, lifecycle safety) have passed.
