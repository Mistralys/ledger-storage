# Synthesis Report — M3 Library & Indexing Rework 1

**Date:** 2026-05-08  
**Plan:** `2026-04-28-m3-library-indexing-rework-1`  
**Status:** COMPLETE  
**Work Packages:** 12 / 12 COMPLETE  
**Pipeline Health:** 12 / 12 WPs with all stages passing — 0 missing stages

---

## Executive Summary

This milestone addressed all actionable items surfaced in the previous M3 synthesis report. Seven discrete engineering concerns were delivered across 12 work packages in a single focused session:

1. **Schema sync** — `structure.sql` updated to reflect the live schema (year/season/episode columns) with a `db_revision` annotation establishing a versioning convention.
2. **Scoped cascade delete** — `SpdbConfigLibraryFolderRepository.RemoveAsync` replaced an unscoped `WHERE NOT EXISTS` orphan-delete with a two-phase hash-scoped `DELETE … WHERE hash IN @Hashes`, transactionally safe thanks to the `movies_filenames.hash UNIQUE KEY` invariant discovered during QA.
3. **Test infrastructure DRY** — A shared `LiveDbFixture` / `LiveDbCollection` was introduced, consolidating five identical copies of config-resolution helpers from four test classes into a single fixture.
4. **Input validation guards** — `ArgumentException.ThrowIfNullOrWhiteSpace` guards added to all public boundary methods of `DapperMovieCatalogRepository` (4 methods) and `SpdbConfigLibraryFolderRepository.AddAsync`.
5. **ViewModel memory safety** — `RefreshIndexViewModel` now implements `IDisposable` with correct `StateChanged` event deregistration.
6. **Fire-and-forget error logging** — `MainContentViewModel.OnRefreshStateChanged` chains `ContinueWith(OnlyOnFaulted)` for silent exception capture, and cascades `RefreshIndex?.Dispose()`.
7. **Test coverage** — 14 new tests added covering guard clauses, Dispose safety, and integration verification of the cascade delete.

All 314 tests pass (5 env-skipped), 0 failures, 0 build warnings across the full solution.

---

## Metrics

### Test Results (final full-suite run)

| Suite | Passing | Skipped | Failing |
|---|---|---|---|
| VideoIndexer.Infrastructure.Tests | 124 | 5 (env-dependent) | 0 |
| VideoIndexer.App.Tests | 77 | 0 | 0 |
| VideoIndexer.Tests (Core) | 128 | 0 | 0 |
| **Total** | **329** | **5** | **0** |

> The 5 skipped tests are live-DB tool tests that require environment configuration (`test-config.json` / `VI_TEST_CONNECTION`). They are expected skips and unchanged from before this milestone.

### New Tests Added

| WP | Test(s) | Type | Notes |
|---|---|---|---|
| WP-006 | `AddAsync_NullPath_ThrowsArgumentException`, `AddAsync_WhitespaceOnlyPath_ThrowsArgumentException` | Unit | InMemoryConfigRepository seam; no live DB |
| WP-009 | `AddAsync_NullPath_ThrowsArgumentException`, `AddAsync_WhitespacePath_ThrowsArgumentException` | Unit | SpdbConfigLibraryFolderRepositoryUnitTests.cs |
| WP-010 | 10 guard tests for `MovieExistsAsync`, `InsertMovieAsync`, `FilenameExistsAsync`, `AddFilenameAsync` | Unit | NeverCalledConnectionFactory stub |
| WP-011 | `Dispose_ThenStateChanged_IsRunningDoesNotChange`, `Dispose_ThenStateChanged_RefreshIndexIsRunningDoesNotChange` | Unit | FakeRefreshOrchestrator; validates event deregistration |
| WP-012 | `RemoveAsync_ScopedDelete_DeletesOnlyMoviesInTargetFolder` | Integration (live DB) | [SkippableFact]; validates two-phase cascade delete |

### Security Audit (WP-002)

| Severity | New in This Milestone | Pre-existing |
|---|---|---|
| Critical | 0 | 0 |
| High | 0 | 0 |
| Medium | 0 | 0 |
| Low | 0 | 2 (see below) |

Pre-existing accepted findings (not introduced by this milestone):
- **A02 — Plaintext password**: `DatabaseConnectionOptions.Password` stored as plain text in settings JSON. Accepted trade-off for a local desktop app; upgrade path available if network-facing surface is added.
- **A02 — Fixed-salt password hash**: `Sha256FixedSaltPasswordHasher` uses a compile-time fixed salt (`spdb_42`) instead of Argon2id/bcrypt/PBKDF2. Documented; `IPasswordHasher` interface provides a clean upgrade path.

### Rework

| WP | Pipeline | Bounce Reason | Resolution |
|---|---|---|---|
| WP-002 | QA FAIL → Implementation rework → QA PASS | QA identified an over-delete bug in the `DELETE FROM movies WHERE hash IN @Hashes` for shared-hash cross-folder scenarios | Schema investigation confirmed `movies_filenames.hash` has a `UNIQUE KEY (hash_2)` — the shared-hash scenario is physically impossible. NOT EXISTS guard reverted; original implementation restored with clarifying inline comment and architectural note in README. |

---

## Work Package Outcomes

| WP | Title / Scope | Key Files | All ACs Met |
|---|---|---|---|
| WP-001 | `structure.sql` — add year/season/episode columns + db_revision annotation | `spdb-indexer/sql/structure.sql`, `spdb-indexer/changelog.md` | ✓ |
| WP-002 | `SpdbConfigLibraryFolderRepository.RemoveAsync` — two-phase hash-scoped delete | `Library/SpdbConfigLibraryFolderRepository.cs` | ✓ |
| WP-003 | `LiveDbFixture` + `LiveDbCollection` — shared test fixture infrastructure | `Infrastructure.Tests/LiveDbCollection.cs`, `LiveDbFixture.cs` | ✓ |
| WP-004 | `RefreshIndexViewModel` — implement `IDisposable` | `App/ViewModels/RefreshIndexViewModel.cs` | ✓ |
| WP-005 | `DapperMovieCatalogRepository` — input validation guards (4 methods) | `Library/DapperMovieCatalogRepository.cs` | ✓ |
| WP-006 | `SpdbConfigLibraryFolderRepository.AddAsync` — input validation guard | `Library/SpdbConfigLibraryFolderRepository.cs`, `ILibraryFolderRepository.cs` | ✓ |
| WP-007 | Migrate 4 live-DB test classes to `LiveDbFixture` | 4 test class files + `LiveDbFixture.cs` | ✓ |
| WP-008 | `MainContentViewModel` — logger injection, fire-and-forget error logging, cascade Dispose | `App/ViewModels/MainContentViewModel.cs`, `MainContentViewModelTests.cs` | ✓ |
| WP-009 | `SpdbConfigLibraryFolderRepository.AddAsync` — guard unit tests | `VideoIndexer.Tests/SpdbConfigLibraryFolderRepositoryUnitTests.cs` | ✓ |
| WP-010 | `DapperMovieCatalogRepository` — guard unit tests (10 tests) | `Infrastructure.Tests/Library/DapperMovieCatalogRepositoryTests.cs` | ✓ |
| WP-011 | Dispose safety tests — `RefreshIndexViewModel` + `MainContentViewModel` | `App.Tests/RefreshIndexViewModelTests.cs`, `MainContentViewModelTests.cs` | ✓ |
| WP-012 | `RemoveAsync` scoped-delete integration test | `Infrastructure.Tests/Library/SpdbConfigLibraryFolderRepositoryTests.cs` | ✓ |

---

## Strategic Recommendations (Gold Nuggets)

### 1. The `movies_filenames.hash` UNIQUE KEY Is an Architectural Load-Bearing Constraint

**Origin:** WP-002 rework cycle — QA correctly flagged a theoretical over-delete bug; the rework investigation proved the schema constraint renders the scenario physically impossible.

**Implication:** The correctness of `RemoveAsync`'s step-5 `DELETE FROM movies WHERE hash IN @Hashes` is entirely dependent on `movies_filenames.hash` carrying a `UNIQUE KEY`. If this constraint is ever relaxed (e.g., to allow the same physical file to be indexed from two different library folders), the delete predicate will silently over-delete movies that are still referenced from the surviving folder. This is now documented in `README.md` with an explicit architectural invariant block and a live-DB integration test (`RemoveAsync_ScopedDelete_DeletesOnlyMoviesInTargetFolder`).

**Recommendation:** Before any schema migration that touches `movies_filenames`, require a review of `RemoveAsync` step 5. The `README.md` architectural invariant note should be treated as a constraint — not a recommendation.

---

### 2. Guard-Clause Tests Should Not Inherit Live-DB Fixture Overhead

**Origin:** WP-010 (code review observation)

`DapperMovieCatalogRepositoryTests` is decorated with `[Collection("LiveDB")]` and `IClassFixture<LiveDbFixture>`. The 10 new guard-clause tests (using `NeverCalledConnectionFactory`) are pure unit tests that open no database connection, but they still incur the LiveDB collection serialization constraint and fixture initialization overhead simply by being in the same class.

**Recommendation:** For the next round of guard-clause coverage, create a separate `DapperMovieCatalogRepositoryGuardTests` class (no `[Collection]`, no fixture) in `VideoIndexer.Infrastructure.Tests` for all pure-unit guard tests. This eliminates unnecessary serialization with live-DB tests and makes the CI pipeline faster.

---

### 3. Establish `db_revision` Annotation as a Required Convention for DDL Snapshots

**Origin:** WP-001 documentation phase + reviewer documentation-forward.

`structure.sql` has been annotated with `-- db_revision: 33, partial sync`. The file remains a partial snapshot — revisions 34+ (e.g., `movies_bookmarks.description` column type changes from Rev 34) are not yet reflected.

**Recommendation:** Schedule a full DDL regeneration from the live `spdb_tests` schema as a standalone WP, annotated `-- db_revision: 35` (or current value of `DB_REVISION` in `DBHelper.cs`). Going forward, every schema migration that creates a new `db_revision` constant should include a corresponding update to `structure.sql` as part of the migration WP's definition of done.

---

### 4. `AddAsync` Path Normalisation Gap

**Origin:** WP-006 code review.

`SpdbConfigLibraryFolderRepository.AddAsync` now throws `ArgumentException` for null/whitespace paths, but does **not** trim leading/trailing whitespace. A path like `"  /media/movies  "` (with surrounding spaces) will pass the guard and be stored as-is. If another caller passes `"/media/movies"` (trimmed), the two will be treated as distinct entries in `spdb_config`, resulting in duplicate-folder entries in the UI.

**Recommendation:** Add `path = path.Trim()` immediately after the `ThrowIfNullOrWhiteSpace` guard in `AddAsync`, or enforce normalisation at the ViewModel layer (whichever is more appropriate for the architecture).

---

### 5. `LiveDbFixture` / `LiveDbCollection` File Placement Decision Pending

**Origin:** WP-003 code review.

`LiveDbCollection.cs` and `LiveDbFixture.cs` were placed at the `VideoIndexer.Infrastructure.Tests` project root (namespace `VideoIndexer.Infrastructure.Tests`), while the existing `Fixtures/` subfolder (containing `TempDirectory.cs`) uses `VideoIndexer.Infrastructure.Tests.Fixtures`. The migration WP (WP-007) has now consumed the fixture, so there are four test classes importing from the root namespace.

**Recommendation:** Decide before any further fixture expansion: either move `LiveDbCollection.cs` and `LiveDbFixture.cs` into `Fixtures/` (and update four `using` statements in the migrated classes) or establish the root namespace as the convention for cross-cutting test infrastructure. The cost of moving is low right now; it increases with every new consumer.

---

## Failed / Blocked Metrics

None. All 12 WPs completed with all pipelines PASS. The single FAIL (WP-002 QA first pass) was an intra-WP rework cycle resolved in the same session.

---

## Next Steps for Planner / Manager

1. **Full `structure.sql` regeneration** — Regenerate from live `spdb_tests` schema, annotate `-- db_revision: 35`, capturing revisions 34+ (priority: medium). Schedule as a standalone WP.
2. **Path normalisation in `AddAsync`** — Add `path = path.Trim()` after the null guard to prevent duplicate-folder entries from whitespace variations (priority: low, but simple).
3. **Decide `LiveDbFixture` file placement** — Root namespace vs `Fixtures/` subfolder. Low cost to move now; grows with every new consumer (priority: low, time-sensitive).
4. **Separate guard tests from live-DB collection** — Create `DapperMovieCatalogRepositoryGuardTests` as a plain test class to avoid fixture overhead for pure unit tests (priority: low).
5. **`LiveDbFixtureTests.cs`** — Add a test class specifically for `LiveDbFixture` schema guard logic (the schema-name guard was removed from `SpdbConfigRepositoryTests` during WP-007 since it tested a private helper). The invariant should be re-verified at the fixture level (priority: low).
6. **`Sha256FixedSaltPasswordHasher` KDF upgrade** — If the application ever acquires a network-facing authentication endpoint, upgrade from SHA-256 + fixed salt to Argon2id or PBKDF2 via the existing `IPasswordHasher` interface (priority: low; only relevant if the threat model changes).

---

## Cross-Cutting Observations

- **Documentation quality**: Every WP correctly produced a CHANGELOG entry and updated README.md. Reviewer documentation-forward items were all acted on by the Documentation agent in the same WP pipeline. No documentation debt carried forward.
- **Build discipline**: `TreatWarningsAsErrors=true` was in effect across all builds. Zero new warnings were introduced by any WP.
- **Test isolation**: The `LiveDbFixture` introduction (WP-003 + WP-007) is a meaningful improvement to the infrastructure test layer. Eliminating five duplicate copies of config-resolution logic reduces maintenance surface and makes the `test-config.json` / `VI_TEST_CONNECTION` discovery path consistently documented in one place.
- **WP-001 AC wording gap**: Acceptance criterion 2 for WP-001 contained a stale positional reference ("immediately after the cache column"). The implementation correctly followed the live schema (after `label`, not after `cache`). Future planning should verify AC positional references against the live schema before decomposition.
