# Synthesis Report — M9 Images Rework 2 · Post-Synthesis Rework 1

**Plan:** `2026-05-22-m9-images-rework-2-rework-1`
**Date:** 2026-05-31
**Status:** COMPLETE — all 5 work packages passed all pipeline stages

---

## Executive Summary

This plan resolved four concrete defects surfaced by the previous synthesis cycle:

1. **FfprobeRunner priority inversion (WP-001)** — `ResolveBinaryPath()` evaluated the provisioner path before the user's library override, silently ignoring user configuration. The branch order was corrected to match `FfmpegRunner`'s documented three-step chain (user override → provisioner → PATH fallback). The method was promoted from `private` to `internal` to enable pure-unit testing without spawning a process.

2. **Missing SettingsViewModel round-trip tests (WP-002)** — Two new `[Fact]` tests were added to `SettingsViewModelTests.cs` covering the `FfmpegOverridePath` ↔ `Library.FfmpegPath` bidirectional mapping (load and save paths). The open item was removed from `constraints.md`.

3. **MovieEditorViewModel 22-parameter constructor (WP-003 + WP-005)** — The constructor was refactored from 22 positional parameters to 3 (Movie, IMovieRepository, MovieEditorOptions?) via a new `MovieEditorOptions` sealed record. All 6 test factory helpers and the Program.cs DI factory were updated. WP-005 (documentation verification) was pre-satisfied during WP-003 implementation.

4. **DapperBookmarkRepository UTC timestamp inconsistency (WP-004)** — `IncrementClickSql` was changed from `NOW()` (server local time) to `UTC_TIMESTAMP()`. A UTC timestamp convention rule was codified in `constraints.md`.

---

## Metrics

| Metric | Value |
|---|---|
| Work packages completed | 5 / 5 |
| Pipeline stages passed | 15 / 15 |
| Release build warnings / errors | 0 / 0 |
| Unit + App tests (final) | 861 (326 unit + 535 app) |
| Test failures | 0 |
| Integration tests (binary/DB-gated) | 6 skipped (previously skipped) |
| New tests added | +2 (SettingsViewModel FfmpegOverridePath round-trip) |
| New production files created | 1 (`MovieEditorOptions.cs`) |
| Production files modified | 3 (`FfprobeRunner.cs`, `MovieEditorViewModel.cs`, `DapperBookmarkRepository.cs`, `Program.cs`) |

---

## Completed Artifacts

| WP | Files Modified |
|---|---|
| WP-001 | `src/VideoIndexer.Infrastructure/Library/FfprobeRunner.cs` |
| WP-001 | `tests/VideoIndexer.Infrastructure.Tests/Library/FfprobeRunnerTests.cs` |
| WP-002 | `tests/VideoIndexer.App.Tests/SettingsViewModelTests.cs` |
| WP-003 | `src/VideoIndexer.App/ViewModels/MovieEditorOptions.cs` *(new)* |
| WP-003 | `src/VideoIndexer.App/ViewModels/MovieEditorViewModel.cs` |
| WP-003 | `src/VideoIndexer.App/Program.cs` |
| WP-003 | `tests/VideoIndexer.App.Tests/MovieEditorViewModelTests.cs` |
| WP-003 | `tests/VideoIndexer.App.Tests/LabelCleanerViewModelTests.cs` |
| WP-004 | `src/VideoIndexer.Infrastructure/Database/DapperBookmarkRepository.cs` |
| WP-001/003/004 | `docs/agents/project-manifest/api-surface.md` |
| WP-001/002/004 | `docs/agents/project-manifest/constraints.md` |
| WP-003 | `docs/agents/project-manifest/file-tree.md` |

---

## Open Tech Debt

One tech-debt item was identified and documented in `constraints.md` during this cycle:

**Pure-unit ResolveBinaryPath tests in the wrong test project.** The three new `[Fact]` tests for `FfprobeRunner.ResolveBinaryPath()` were placed in `VideoIndexer.Infrastructure.Tests` (the integration suite) during WP-001 implementation. Since `ResolveBinaryPath()` is a pure in-memory method with no external dependencies, these tests belong in `VideoIndexer.Tests` (the unit suite). All three pipeline stages (Developer, QA, Reviewer) independently flagged this. Deferred to a future test project boundary cleanup pass.

---

## Strategic Recommendations

1. **Test project boundary cleanup.** The consistent cross-agent flag on misplaced pure-unit tests suggests the boundary between `VideoIndexer.Tests` and `VideoIndexer.Infrastructure.Tests` is not intuitive. A dedicated cleanup pass to move all non-integration tests out of the infrastructure suite would reduce confusion for future agents.

2. **Runner abstraction.** `FfprobeRunner` and `FfmpegRunner` are structurally near-identical (binary name constant, constructor, main async method, `internal ResolveBinaryPath()`, three-step resolution chain). If a third runner is added, a shared `BinaryRunner` base class or `BinaryResolver` helper would eliminate duplication. The WP-001 Developer explicitly noted this and deferred it to a future tools-layer cleanup.

3. **Integration test coverage of UTC_TIMESTAMP.** The `NOW()` → `UTC_TIMESTAMP()` fix in `IncrementClickSql` (WP-004) has no automated regression guard — the integration tests are DB-gated and run only in configured environments. If a future schema or SQL change reverts this, it will only be caught at runtime. Consider adding a static analysis or string-search guard in the test suite.

---

## Next Steps

- **Planner / PM:** Advance to M10 milestone planning. The Release Engineer PR for `manifest.md` (deferred from the previous cycle) remains unaddressed and should be included in the next plan.
- **Developer:** Address the `ResolveBinaryPath` test placement debt (move 3 tests from infrastructure to unit suite) in the next available cleanup window.
- **Developer:** Evaluate the `BinaryRunner` abstraction when a third external-tool runner is needed.
