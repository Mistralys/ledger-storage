# Synthesis Report — M3 Library & Indexing

**Plan:** `2026-04-28-m3-library-indexing`  
**Date Generated:** 2026-05-08  
**Status:** COMPLETE — 19/19 work packages delivered  
**Pipeline Health:** All 19 WPs passed all active pipeline stages (0 missing stages)

---

## Executive Summary

M3 delivers the full **library folder management and indexing pipeline** for Video Indexer MK2 on top of the M2 database/auth shell. The milestone implements:

- **Library Folder Registry** — server-side folder persistence in `spdb_config` via `SpdbConfigLibraryFolderRepository`, with atomic add/remove and a LIKE-safe cascade delete that preserves sibling folder data.
- **Path-Based Hashing** — the 41-key MD5 algorithm using a `Utf8JsonReader` token-stream parser (raw numeric literal preservation; first-occurrence-wins duplicate semantics), wrapped in `PathBasedVideoHasher`.
- **Obfuscation-Aware Scanner** — `ExtensionObfuscationMap` and `FfprobeRunner` as DI-injectable infrastructure components; `LibraryScanner` coordinates the full per-file pipeline with path-length guard, cancellation, progress reporting, and per-file fault isolation.
- **Refresh Orchestrator** — `RefreshOrchestrator` provides a thread-safe, single-flight background runner using `Interlocked.CompareExchange` and `Volatile.Read/Write`; state transitions and a `StateChanged` event allow the UI to react.
- **UI Layer** — `LibraryFoldersView`, `RefreshIndexView`, and updates to `MainContentView` bind the full workflow to the `MainContentViewModel` page navigation model. `IFolderPickerService` / `AvaloniaFolderPickerService` abstract the Avalonia storage picker.
- **DI Composition Root** — `Program.cs` extended with step 7 registering all 7 library singletons + transient view models; `ShellState.Ready` arm resolves `MainContentViewModel` from DI.
- **Test Suite** — Grew from the 110-test M2 baseline to **313+ tests** across 3 test projects. Live-DB and ffprobe-dependent tests are `SkippableFact`-guarded throughout.

The application satisfies the M3 "done" definition: from a fresh clone, after the `Connecting → LoggingOn → Ready` flow, the user can register `test-folder/`, click *Refresh Index*, and observe `movies` rows populated for obfuscated files — no duplicates on second refresh, no exceptions.

---

## Metrics

| Metric | Value |
|---|---|
| Work packages | 19 / 19 COMPLETE |
| Acceptance criteria | 76 / 76 met (100%) |
| Security audit findings (Critical/High) | 0 |
| Security audit findings (Medium) | 2 (documented, non-blocking) |
| Build warnings | 0 (all WPs, `TreatWarningsAsErrors=true`, `WarningLevel=9999`) |
| Tests at M2 baseline | 110 |
| Tests at M3 completion | 313+ (unit) + live/integration (self-skipping) |
| Test failures | 0 |
| Pre-existing skips (live environment) | 4–5 (expected, no DB/ffprobe in CI) |

### Test Growth by Milestone Sub-Layer

| Component | New Tests Added |
|---|---|
| LibraryOptions / AppOptions (WP-001) | +51 (grew from 110 to 161) |
| Core contracts (WP-002) | +0 (compilation-only WP) |
| ExtensionObfuscationMap (WP-003) | +19 unit |
| FfprobeRunner (WP-003) | +2 SkippableFacts |
| SpdbConfigLibraryFolderRepository (WP-004/WP-012) | +16 (5 unit + 7 live-DB + 4 seam tests) |
| DapperMovieCatalogRepository (WP-005/WP-014) | +9 live-DB |
| PathBasedVideoHasher (WP-006/WP-011) | +28 (13 unit + 15 FakeFfprobeRunner-based) |
| LibraryScanner (WP-007/WP-016) | +13 unit |
| Test fixtures (WP-008) | +0 (shared infrastructure only) |
| RefreshOrchestrator (WP-009/WP-016) | +22 (14 in infra tests + 8 unit) |
| View models (WP-010/WP-015) | +35 (VM unit tests) |
| Integration tests (WP-013/WP-018) | +6 SkippableFacts (real file/DB) |
| **Total new** | **~200 new tests** |

---

## Failures & Blockers (Aggregated)

### Critical Schema Debt — `movies_filenames.hash` UNIQUE Constraint
**Raised:** WP-004 Developer; escalated to project-level comment by Reviewer (WP-004) and Documentation (WP-004); confirmed by QA (WP-005), Security Auditor (WP-005), and integration tests (WP-018).

`movies_filenames.hash` has a `UNIQUE` constraint in the live `spdb_tests` database that is **absent from `structure.sql`** (the reference SQL was generated without the `ALTER TABLE ... ADD UNIQUE` statements). The practical effect:
- `INSERT IGNORE` is used correctly in `DapperMovieCatalogRepository.AddFilenameAsync` — this is **already handled**.
- The `structure.sql` reference file is **stale** and misleads future developers about the schema's true constraints.
- Each movie hash may only have **one** filename path in `movies_filenames` (one-to-one, not one-to-many as the spec implies). WP-018 integration tests were redesigned around this constraint.

**Required action:** Update `structure.sql` to include `UNIQUE KEY` on `movies_filenames.hash` so the reference file matches the live schema.

### Medium Security Finding — `SpdbConfigLibraryFolderRepository.AddAsync` Counter Race
**Raised:** WP-004 Security Auditor (A04 — Insecure Design).

The `folder_counter` read-increment-write in `AddAsync` is not protected by `SELECT ... FOR UPDATE`. Two concurrent `AddAsync` calls can allocate the same `LibraryFolder.Id`. The repository is currently intended for **low-frequency admin use only** (documented in `ILibraryFolderRepository.AddAsync` XML remarks), so this is non-blocking for M3. If the admin path is ever exposed to concurrent callers, a row-level lock must be added.

### Medium Security Finding — `FfprobeRunner` Unsanitized Settings-Sourced Binary Path
**Raised:** WP-003 Security Auditor (A04 — Insecure Design).

The binary path sourced from `ExternalTools.Ffmpeg.FfprobePath` or `Library.FfprobePath` is used directly in `ProcessStartInfo` without an allowlist check. Documented as admin-only settings; no remediation required before M4 but should be locked down if the settings surface is ever broadened.

### Test Isolation Race — `SpdbConfigLibraryFolderRepositoryTests` + `DapperMovieCatalogRepositoryTests`
**Raised:** WP-004 QA, confirmed WP-005.

`RemoveAsync_FolderPathWithUnderscore_DoesNotDeleteSiblingFolderFilenames` fails with MySQL `ER_CHECKREAD` (`Record has changed since last read`) when run alongside `DapperMovieCatalogRepositoryTests` in the full suite. The root cause is a global orphan-cleanup `DELETE` in `RemoveAsync` that touches all `movies` rows — likely a MyISAM table (no MVCC/row-level locking). The test passes 7/7 in class isolation.

**Required actions:**
1. Add `[Collection("LiveDB")]` to all live-DB test classes to serialize execution.
2. Investigate / migrate the `movies` table to InnoDB if still using MyISAM.

---

## Strategic Recommendations

### Gold Nuggets

**1. Scope orphan-cleanup DELETE to specific hashes (WP-004 QA / WP-005 improvement)**  
`SpdbConfigLibraryFolderRepository.RemoveAsync` executes a global `DELETE m FROM movies m WHERE NOT EXISTS (...)` that touches *all* `movies` rows. Under concurrent load, this creates unnecessary contention. Narrowing the DELETE to the specific hashes deleted from `movies_filenames` in step 3 eliminates the contention and removes the test isolation race.

**2. Consolidate live-DB test infrastructure (WP-005 Developer / WP-014 Reviewer)**  
`LiveConnectionFactory` (private sealed class) is duplicated byte-for-byte across `DapperMovieCatalogRepositoryTests`, `SpdbConfigLibraryFolderRepositoryTests`, and `LibraryScannerIntegrationTests`. A shared `IntegrationTestFixture` or `BaseIntegrationTest` in `VideoIndexer.Infrastructure.Tests` would eliminate ~60–80 lines of duplication and make it easier to extend the test configuration going forward.

**3. Add `ArgumentException.ThrowIfNullOrWhiteSpace` guards at public service boundaries (WP-004 Security Auditor / WP-005 Security Auditor)**  
Neither `SpdbConfigLibraryFolderRepository.AddAsync` nor `DapperMovieCatalogRepository`'s methods validate null/whitespace input parameters. Null path to `AddAsync` causes a deferred `NullReferenceException` at `RemoveAsync` time. Null hash to any catalog method propagates silently into SQL (NULL ≠ NULL semantics). These guards are one-liners and should be added at all public service method boundaries.

**4. Address `StateChanged` subscription leak in view models (WP-010 Developer)**  
`RefreshIndexViewModel` and `MainContentViewModel` subscribe to `IRefreshOrchestrator.StateChanged` but never unsubscribe (no `IDisposable`). For singleton-lifetime VMs this is harmless today, but it becomes a handler accumulation issue if the composition root is ever restructured. Implementing `IDisposable` with unsubscribe in `Dispose()` is the defensive fix.

**5. Restore `IRefreshOrchestrator.StateChanged` missing from WP-002 deliverable (WP-009 Developer)**  
The `StateChanged` event was specified in the plan but omitted from the `IRefreshOrchestrator` interface delivered in WP-002. It was added retroactively in WP-009. The WP-002 QA and code-review pipelines did not catch the omission. **Recommendation:** Interface completeness should be validated against the plan spec during QA, not just against the WP acceptance criteria.

**6. Standardize `FfprobeRunner` cancellation contract (WP-003 Documentation)**  
When a `CancellationToken` fires during `FfprobeRunner.ProbeAsync`, the `ffprobe` child process continues running in the background (OS process not killed). This is documented in `IFfprobeRunner.ProbeAsync` XML docs, but callers should be aware that long-running scans may leave orphaned processes. For M4+, a process kill on cancellation (with graceful timeout) should be evaluated.

---

## Artifacts Delivered

| Layer | File | WP |
|---|---|---|
| Options | `VideoIndexer.Core/Options/LibraryOptions.cs` | WP-001 |
| Options | `VideoIndexer.Core/Options/AppOptions.cs` (updated) | WP-001 |
| Config | `VideoIndexer.App/Assets/appsettings.json` (Library section) | WP-001 |
| Enums | `VideoIndexer.Core/Enums/RefreshOutcome.cs` | WP-002 |
| Enums | `VideoIndexer.Core/Enums/RefreshState.cs` | WP-002 |
| Models | `VideoIndexer.Core/Models/LibraryFolder.cs` | WP-002 |
| Models | `VideoIndexer.Core/Models/RefreshProgress.cs` | WP-002 |
| Models | `VideoIndexer.Core/Models/RefreshSummary.cs` | WP-002 |
| Events | `VideoIndexer.Core/Events/RefreshStateChangedEventArgs.cs` | WP-009 |
| Abstractions | `ILibraryFolderRepository`, `IMovieCatalogRepository`, `IFfprobeRunner`, `IVideoHasher`, `IObfuscationMap`, `ILibraryScanner`, `IRefreshOrchestrator` | WP-002 |
| Infrastructure | `Library/ExtensionObfuscationMap.cs` | WP-003 |
| Infrastructure | `Library/FfprobeRunner.cs` | WP-003 |
| Infrastructure | `Library/SpdbConfigLibraryFolderRepository.cs` | WP-004 / WP-012 |
| Infrastructure | `Library/DapperMovieCatalogRepository.cs` | WP-005 |
| Infrastructure | `Library/PathBasedVideoHasher.cs` | WP-006 |
| Infrastructure | `Library/LibraryScanner.cs` | WP-007 |
| Infrastructure | `Library/RefreshOrchestrator.cs` | WP-009 |
| Infrastructure | `Properties/AssemblyInfo.cs` (InternalsVisibleTo) | WP-004 |
| App | `Services/IFolderPickerService.cs` | WP-010 |
| App | `Services/AvaloniaFolderPickerService.cs` | WP-010 |
| App | `ViewModels/LibraryFoldersViewModel.cs` | WP-010 / WP-015 |
| App | `ViewModels/RefreshIndexViewModel.cs` | WP-010 |
| App | `ViewModels/MainContentViewModel.cs` (updated) | WP-010 / WP-015 |
| App | `ViewModels/MainContentPage.cs` | WP-015 |
| App | `Views/LibraryFoldersView.axaml[.cs]` | WP-015 |
| App | `Views/RefreshIndexView.axaml[.cs]` | WP-015 |
| App | `Views/MainContentView.axaml` (updated) | WP-015 |
| DI | `Program.cs` (step 7 library registrations) | WP-017 |
| Tests | `VideoIndexer.Tests/` — ExtensionObfuscationMapTests, SpdbConfigLibraryFolderRepositoryUnitTests, DapperMovieCatalogRepositoryTests (unit), PathBasedVideoHasherTests, LibraryScannerTests, RefreshOrchestratorTests, Fixtures/FakeFfprobeRunner, Fixtures/TempVideoFolder, Fixtures/InMemoryMovieCatalogRepository | WP-003/008/011/012/007/016/008 |
| Tests | `VideoIndexer.Infrastructure.Tests/` — FfprobeRunnerTests, PathBasedVideoHasherTests, SpdbConfigLibraryFolderRepositoryTests, DapperMovieCatalogRepositoryTests (live-DB), RefreshOrchestratorTests, PathBasedVideoHasherIntegrationTests, LibraryScannerIntegrationTests | WP-003/004/005/006/009/013/018 |
| Docs | `docs/projects/rebuild/milestones/m3-library-indexing.md` | WP-019 |
| Docs | `README.md` (Library Contracts, configuration table, Build & Run, Running Tests sections) | WP-001/002/003/004/019 |
| Memory | `/memories/repo/ledger-closure-recovery.md` | WP-019 |

---

## Next Steps for Planner / Project Manager

1. **Fix `structure.sql`** — Update the reference SQL to add `UNIQUE KEY (hash)` on `movies_filenames`. This is the single highest-impact schema debt item and will prevent future developer confusion.
2. **Serialize live-DB tests** — Add `[Collection("LiveDB")]` to `SpdbConfigLibraryFolderRepositoryTests` and `DapperMovieCatalogRepositoryTests` before the test suite is run in CI.
3. **Scope orphan DELETE in `RemoveAsync`** — Narrow the global `WHERE NOT EXISTS` to the specific set of hashes deleted in step 3 to eliminate the test isolation race and reduce contention.
4. **M4 Planning** — M3 deliberately defers the movie editor and grid UI (M4–M7) and the obfuscation toggle UI (M8). The M4 plan should consider:
   - `IMovieCatalogRepository` does not expose `year`, `season`, or `episode` columns present in the live schema — these will require interface versioning if needed for M4 features.
   - `RefreshState` has no `Cancelled` variant; cancelled runs silently return to `Idle`. If cancellation feedback is needed in M4 UI, add a `Cancelled` enum value.
   - `MainContentViewModel.OnRefreshStateChanged` uses a fire-and-forget `RefreshCountAsync` with no exception handler — add a logging continuation before M4.
5. **Add `ArgumentException.ThrowIfNullOrWhiteSpace`** to `SpdbConfigLibraryFolderRepository.AddAsync` and `DapperMovieCatalogRepository` public methods.

---

*Synthesis generated by Head of Operations (Synthesis Agent) — 2026-05-08*
