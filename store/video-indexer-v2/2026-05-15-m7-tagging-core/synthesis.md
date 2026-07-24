# Synthesis Report — M7 Tagging Core

**Plan:** `2026-05-15-m7-tagging-core`  
**Date:** 2026-05-18  
**Status:** COMPLETE  
**Agent:** Head of Operations (Synthesis)

---

## Executive Summary

M7 — Tagging Core delivers the full tagging subsystem for Video Indexer MKII. All 26 work packages completed with zero open failures. The session produced:

- **Core domain layer**: 8 new types (`Tag`, `TagCategory`, `TagGrant`, `TagDeleteImpact`, `TagMergeImpact`, `TagImportance`, `TagConstants`, `TaggingOptions`)
- **Core interfaces**: `ITagsRepository` (17-member raw DB contract) and `ITagsManager` (in-memory graph facade)
- **Infrastructure**: `DapperTagsRepository` (full implementation) + `DapperMovieCatalogRepository` enriched with two-round-trip batch tag loading and effective-set computation
- **TagsManager service**: Lazy-loaded in-memory tag graph with cycle detection, effective-set traversal, `DataChanged` event, and thread-safe `EnsureLoadedAsync`
- **Filter DSL upgrade**: `HasTag` / `TagHasSubTags` activated; `TagsWeight` / `AmountTags` upgraded to effective-set semantics; `StoredTagsWeight` / `AmountStoredTags` added as new stored-only identifiers
- **UI layer**: `TaggerView` wired into Movie Editor sidebar; `TagEditorView`, `CategoryEditorView`, `GrantsManagementView`, `TagMergeView` management dialogs; 7 new ViewModels
- **Dialog services**: `AvaloniaTagEditorService`, `AvaloniaGrantsManagementService`, `AvaloniaTagMergeService`
- **Schema migration m040**: UNIQUE KEY constraints on `tags_categories.name` and `tags(category_id, parent_tag_id, name)`
- **Version bump**: `1.1.0 → 1.1.1` (patch)
- **M6 carry-forwards resolved**: Actor reorder list selection wiring, `ShowPropertiesCommand` injection into `MovieEditorViewModel`, Cover Image tab label correction
- **NameParser**: `LabelCleanerOptions.DetectTags` activated

---

## Metrics

| Metric | Value |
|---|---|
| Work packages | 26 / 26 COMPLETE |
| Test suite (final) | **758 passed, 0 failed, 6 skipped** (env-conditional) |
| New tests written | ~145 |
| Build result | **0 errors, 0 warnings** |
| Schema revision | 39 → 40 |
| Version | 1.1.0 → 1.1.1 |
| Security audits | 2 (WP-008, WP-015) — both PASS |
| Critical / High security findings | **0** |
| WPs requiring rework | **1** (WP-021) |
| Rework cycles in WP-021 | implementation ×2, qa ×2, code-review ×1 |

### Test Suite Breakdown (Final State)

| Project | Passed | Failed | Skipped |
|---|---|---|---|
| VideoIndexer.Tests (Core) | 255 | 0 | 0 |
| VideoIndexer.App.Tests | 325 | 0 | 0 |
| VideoIndexer.Infrastructure.Tests | 178 | 0 | 6 |
| **Total** | **758** | **0** | **6** |

### New Test Suites Introduced

- `TagsManagerTests.cs` — TagsManager unit tests
- `FilterExpressionEvaluatorM7Tests.cs` — M7 filter DSL coverage
- `TaggerViewModelTests.cs` — 8 AC-targeted behavioral tests
- `TagEditorViewModelTests.cs`, `TagMergeViewModelTests.cs`, `GrantsManagementViewModelTests.cs`, `CategoryEditorViewModelTests.cs` — ViewModel unit tests
- `DapperTagsRepositoryTests.cs` — 7 self-skipping integration tests (live DB required)
- `FakeTagsManager.cs`, `FakeMoviePropertiesService.cs` — shared test fakes

---

## Security Summary

Two WPs carried a security audit stage (WP-008, WP-015). Both passed with 0 critical and 0 high findings.

| WP | Scope | Critical | High | Medium | Low |
|---|---|---|---|---|---|
| WP-008 | `DapperTagsRepository`, `Program.cs` DI | 0 | 0 | 2 | 4 |
| WP-015 | `DapperMovieCatalogRepository`, test files | 0 | 0 | 0 | 3 |

**WP-008 medium findings (addressed):**
- `DeleteTagAsync` non-transactional cascade → **resolved in WP-021 (transactional wrap with ambient guard)**
- `GetMergeImpactAsync` ignores `targetTagId` → documented; overlap detection explicitly deferred to merge-execution WP

All SQL queries use Dapper `@`-parameterised form throughout. No injection surface identified.

---

## Rework Detail — WP-021

WP-021 (DapperTagsRepository Integration Tests) was the only work package requiring rework. The test suite exposed two pre-existing production bugs in `DapperTagsRepository.cs`:

**Bug 1 (HIGH) — `TagRow.Highlight` ENUM type mismatch**
- `highlight` column is `ENUM('yes','no')` but `TagRow.Highlight` was declared `bool`
- Dapper sent `false` as ordinal 0 → empty string → `DATA TRUNCATED`; read path could not parse `'no'` back to `bool`
- Fixed by converting `TagRow.Highlight` to `string` with `'no'` default, mapping via `string.Equals(r.Highlight, "yes", OrdinalIgnoreCase)`, and passing `tag.Highlight ? "yes" : "no"` in write paths
- Pattern matches the established `DapperMovieRepository.MovieRow.Review` precedent

**Bug 2 (HIGH) — `DeleteTagAsync` non-transactional cascade**
- 5-step cascade (DELETE `tags_movies`, DELETE `tags_grants`, UPDATE parent, DELETE tag) ran as separate round-trips with no transaction
- A partial failure between any two steps would leave the database in an inconsistent state
- Fixed by wrapping steps 2–5 in `BeginTransaction()` / `Commit()` / `Rollback()`
- A second rework cycle was needed after the initial fix caused nested-transaction failures in the shared-rollback test fixture
- Final resolution: ambient-transaction detection via `try/catch(InvalidOperationException)` around `BeginTransaction()`; when an outer transaction is already active (test path), `ownedTx = null` and Dapper auto-enlists commands in the ambient transaction

---

## Strategic Recommendations (Gold Nuggets)

### 1. Dapper CancellationToken propagation gap (codebase-wide)

All repository implementations (`DapperMovieRepository`, `DapperMovieCatalogRepository`, `DapperFilterSlotRepository`, `DapperTagsRepository`, `SpdbConfigRepository`) pass `CancellationToken` only to `CreateOpenConnectionAsync`. Individual `QueryAsync` / `ExecuteAsync` calls cannot be cancelled mid-flight.

**Recommendation:** A focused debt-reduction WP should sweep all repositories and replace bare string-based Dapper calls with `CommandDefinition(sql, param, cancellationToken: ct)`.

### 2. `TaggerViewModel.DataChanged` event leak

`TaggerViewModel` subscribes to `ITagsManager.DataChanged` in its constructor but is not `IDisposable`. The singleton `ITagsManager` holds a live reference to the ViewModel indefinitely, preventing GC of any dismissed Tagger control.

**Recommendation:** Implement `IDisposable` on `TaggerViewModel` and wire `Dispose()` to the Avalonia `Unloaded` event in the code-behind.

### 3. Fire-and-forget tag toggle swallows exceptions

`TaggerTagViewModel.OnIsCheckedStoredChanged` discards the connect/disconnect task result (`_ = callback(...)`). Transient DB errors or connection failures during tag association are silently lost.

**Recommendation:** Introduce an `IErrorBus` or `StatusBarViewModel` notification channel and route async exceptions from tag toggle callbacks through it.

### 4. `GetMergeImpactAsync` targetTagId gap

`DapperTagsRepository.GetMergeImpactAsync` accepts `targetTagId` but queries only `sourceTagId` counts. When the merge-execution WP is implemented, overlap (movies tagged with both source and target) must be surfaced or the merge operation must use `INSERT IGNORE` explicitly.

**Recommendation:** Either extend the impact query to return overlap counts, or document clearly in the merge-execution WP spec that `INSERT IGNORE` semantics are required.

### 5. `tags_bookmarks` orphan rows on tag delete

`DapperTagsRepository.DeleteTagAsync` cascade does not issue a `DELETE` against `tags_bookmarks` for the deleted tag. Orphaned rows accumulate unless the schema defines `ON DELETE CASCADE` on `tags_bookmarks.tag_id`. DB integer ID reuse (auto-increment) makes re-association unlikely, but the hygiene risk should be addressed before data accumulates.

**Recommendation:** Verify the `tags_bookmarks` FK definition. If `ON DELETE CASCADE` is absent, add `DELETE FROM tags_bookmarks WHERE tag_id = @TagId` as step 1.5 of the cascade (or incorporate into the transaction).

### 6. `BatchTagSql` performance profile

`DapperMovieCatalogRepository.GetMovieListAsync` now performs two SQL round-trips per call — the main aggregate query and a full `tags_movies` batch-fetch. The batch query has no `WHERE` clause and fetches all movie-tag rows regardless of any filter applied to the main query.

**Recommendation:** When pagination or filtering is added to `GetMovieListAsync`, ensure `BatchTagSql` gains a matching `WHERE movieId IN (...)` clause to avoid full-table reads on filtered results.

---

## Known Debt Carried Forward

| Item | Priority | WP Flagged | Notes |
|---|---|---|---|
| Dapper CancellationToken gap (all repositories) | Medium | Project comment | Use `CommandDefinition` with `cancellationToken` |
| `TaggerViewModel` event subscription leak (no `IDisposable`) | Medium | WP-016 | Wire `Dispose()` to Avalonia `Unloaded` |
| Tag toggle fire-and-forget swallows exceptions | Medium | WP-016 | Route errors through notification channel |
| `GetMergeImpactAsync` ignores `targetTagId` | Medium | WP-008, WP-021 | Deferred to merge-execution WP |
| `tags_bookmarks` orphan rows on tag delete | Low | WP-008, WP-021 | Verify FK or add application-level DELETE |
| `DeleteImpactRow` / `MergeImpactRow` duplication | Low | WP-008 | Consolidate into single `ImpactRow` class |
| `MoviesListViewModel` ambiguous constructor selection | Low | WP-025 | Add explicit factory lambda like `MovieEditorViewModel` |
| Singleton `DapperMovieCatalogRepository` capturing transient `IDbConnectionFactory` | Low | WP-025 | Pre-existing captive dependency |
| `TaggingOptions.MostUsedThreshold` lacks range validation | Low | WP-001 | Add guard in `TagsManager` at system boundary |
| Count-mismatch defensive guard has no log output | Low | WP-015 | Add Warning log when `ILogger` is eventually injected |
| Pre-existing CS8765 nullability warnings in `TagsManagerTests.cs` | Low | Multiple WPs | Lines 344, 372; unrelated to M7 |

---

## Next Steps for Planner / Manager

1. **M7 continuation** — Merge-execution WP (`TagsManager.MergeTagAsync` + `DapperTagsRepository` merge write path) is the highest-value unimplemented surface. The `GetMergeImpactAsync` gap (targetTagId) must be resolved here.

2. **Debt sweep** — A single-purpose WP targeting the Dapper `CancellationToken` gap across all 5 repositories would meaningfully improve cooperative cancellation throughout the I/O path with low risk.

3. **`TaggerViewModel` lifecycle** — Add `IDisposable` before the Tagger sees heavy usage in production; the event leak is a known memory concern in long-running sessions.

4. **Integration test expansion** — `DapperTagsRepository` now has 7 live-DB integration tests but several methods (CRUD round-trips, grant management, bookmark association) are not yet tested beyond compile-time verification. A follow-up WP should expand coverage.

5. **Schema FK audit** — Confirm `tags_bookmarks.tag_id` FK has `ON DELETE CASCADE`, or add the application-level cascade step before M7 ships to production.

---

*Synthesis generated by Head of Operations — 2026-05-18*
