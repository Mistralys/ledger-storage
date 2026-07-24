# Synthesis Report — Linux (Fedora Workstation) Support

**Plan:** `2026-05-20-linux-support`  
**Date:** 2026-05-21  
**Status:** COMPLETE — 9/9 work packages, all pipeline stages PASS  
**Synthesized by:** Head of Operations (Synthesis Agent)

---

## Executive Summary

This milestone promoted Linux (`linux-x64` / Fedora Workstation) to a first-class supported platform alongside Windows. The work required no new abstractions, no new layers, and no new NuGet dependencies — four targeted changes to existing code, accompanied by test fixtures, regression tests, and comprehensive documentation updates.

**What was built:**

1. **`OutputType` `WinExe` → `Exe`** (WP-001) — Removed the false signal that the application is Windows-only. The console window briefly visible on Windows desktop launches is an accepted trade-off; the issue can be addressed with an `app.manifest` entry if needed in future.

2. **Cross-platform file launcher** (WP-002, WP-003) — Renamed `WindowsFileLauncherService` to `DesktopFileLauncherService` and added an `OperatingSystem.IsWindows()` branch: Windows continues using `explorer.exe`; Linux uses `xdg-open` (distro-agnostic, present on all major desktop Linux environments including Fedora Workstation with GNOME and KDE Plasma). All argument passing uses `ProcessStartInfo.ArgumentList` exclusively — no string concatenation.

3. **Zip-Slip / Tar-Slip guard hardening** (WP-004, WP-005) — Changed `StringComparison.OrdinalIgnoreCase` to `StringComparison.Ordinal` in both archive extractors. On Linux case-sensitive filesystems, `OrdinalIgnoreCase` was a theoretical traversal-bypass vector; `Ordinal` is unconditionally correct after `Path.GetFullPath` normalisation.

4. **Device-node rejection in `TarXzArchiveExtractor`** (WP-005) — Added a guard that throws `InvalidDataException` for `BlockDevice`, `CharacterDevice`, and `Fifo` archive entry types before any filesystem operation. On Linux, these entry types would otherwise create real kernel device nodes, a privilege-escalation risk if the archive source is untrusted.

5. **Test fixtures and regression gates** (WP-006, WP-007, WP-008) — Created `device.tar.xz` (212 bytes, PAX `BlockDevice` entry `dev/null` major=1 minor=3), added `ExtractAsync_DeviceNodeEntry_ThrowsInvalidDataException`, and added `ExtractAsync_TraversalEntry_OrdinalComparison_RejectsEntry` (a runtime case-divergence regression gate confirming that `Ordinal` semantics are enforced and that reverting to `OrdinalIgnoreCase` would be caught).

6. **Documentation closure** (WP-009) — Removed all stale `WinExe`/`WindowsFileLauncherService` references and the two deferred `README.md` TODO paragraphs; updated `tech-stack.md`, `constraints.md`, `file-tree.md`, `api-surface.md`, `AGENTS.md`, `m1-foundation.md`, and `IFileLauncherService.cs`.

---

## Metrics

| Metric | Value |
|---|---|
| Work packages | 9 total, 9 COMPLETE |
| Pipeline stages passed | 37 / 37 (100%) |
| Pipeline FAILs | 0 |
| Rework cycles | 0 |
| Security findings (Critical / High / Medium) | 0 / 0 / 0 |
| Security findings (Low/Info — informational only) | 9 |
| Tests passing at final state (unit + app) | 643 (282 unit + 361 app) |
| ZipArchiveExtractor tests | 5 / 5 |
| TarXzArchiveExtractor tests | 9 / 9 (1 pre-existing fixture skip, not caused by this work) |
| Infrastructure test skips | 6 (DB-integration tests, expected, pre-existing) |
| Build warnings / errors | 0 / 0 (Release configuration, `TreatWarningsAsErrors=true`) |
| Reviewer Fix-Forward corrections applied | 7 (in-cycle, non-blocking) |
| Documentation-forward items raised | 7 — all addressed within the same cycle |
| Files modified (production source) | 4 |
| Files modified (tests) | 2 |
| Files modified (documentation) | 11 |

---

## Security Highlights

Three WPs included formal security audit pipelines (WP-003, WP-004, WP-005). All audits returned **PASS** with 0 Critical / High / Medium findings.

| Finding | Severity | Status |
|---|---|---|
| WP-004: `OrdinalIgnoreCase` → `Ordinal` in Zip-Slip guard — theoretical bypass on Linux | Resolved (was Low pre-fix) | **Closed by this plan** |
| WP-005: `OrdinalIgnoreCase` → `Ordinal` in Tar-Slip guard — same bypass | Resolved (was Low pre-fix) | **Closed by this plan** |
| WP-005: Device-node/FIFO entry creation on Linux — privilege escalation risk | Resolved (was Medium pre-fix) | **Closed by this plan** |
| WP-003 (informational): `Process.Start()` return discarded — silent failure if `xdg-open` absent | Low/Info | Accepted; consistent with Windows branch behaviour |
| WP-003 (informational): `explorer.exe /select,<file>` with a comma in the file path may be misparsed by explorer internally | Low/Info | Accepted; no code execution risk, `ArgumentList` prevents OS injection |
| WP-003 (informational): `xdg-open` resolved from `$PATH` — malicious binary placed earlier in `$PATH` would be invoked | Low/Info | Accepted; requires prior full account compromise |
| WP-004 (informational): No decompressed-size cap in `ZipArchiveExtractor` — Zip Bomb | Low/Info | Pre-existing; acceptable (controlled FFmpeg distribution source) |
| WP-005 (informational): `overwrite: true` in `TarXzArchiveExtractor.ExtractToFileAsync` — silent overwrite | Low/Info | Pre-existing; acceptable (idempotent FFmpeg re-extraction) |
| WP-005 (informational): No extraction audit logging in `TarXzArchiveExtractor` | Low/Info | Pre-existing; TODO documented in class XML doc |

---

## Strategic Recommendations

### Gold Nuggets — Insights for the Next Planning Cycle

1. **Zip Bomb protection is unaddressed** — Both `ZipArchiveExtractor` and `TarXzArchiveExtractor` have no upper bound on decompressed size or entry count. While the current call sites use controlled FFmpeg distribution archives, if either extractor is ever exposed to user-supplied archives, a Zip Bomb could exhaust disk space. A lightweight guard (check `entry.Length` against a configurable limit before streaming) would be the appropriate fix. This is a `constraints.md` TODO candidate.

2. **`Process.Start()` silent failures in `DesktopFileLauncherService`** — Both `OpenFolder` and `ShowInExplorer` discard the `Process.Start()` return value with `_ =`. On Linux, if `xdg-open` is absent (minimal desktop environments, server installs), the file-launch silently fails with no log entry. The Security Auditor and QA both noted this. A future WP could wrap the call in a null-check and log a `Warning` via `ILogger`, consistent with the logging groundwork already in place in other Infrastructure services.

3. **`TarXzArchiveExtractor` has no audit logging** — The class XML doc already contains a detailed TODO with specific `LogDebug`/`LogWarning`/`LogInformation` integration points. When the provisioning-pipeline logging feature is added, this class should be the first beneficiary: it processes untrusted (or semi-trusted) archive data and currently emits no observable events.

4. **CharacterDevice and Fifo guard coverage is implicit only** — The device-node rejection in `TarXzArchiveExtractor` covers `BlockDevice | CharacterDevice | Fifo`, but only `BlockDevice` has a dedicated test fixture. This is acceptable for now (same guard code path), but if the guard is ever refactored into separate branches, the implicit coverage disappears silently. Consider adding minimal `character.tar.xz` and `fifo.tar.xz` fixtures in a future infrastructure hardening WP for completeness.

5. **`ShowInExplorer` on Linux opens a directory, not a file** — `xdg-open` opens the parent directory; there is no universally available "select file in file manager" command across Linux desktop environments. This behaviour is now accurately documented in `IFileLauncherService.cs` and `README.md`. The Planner should record this as a known UX limitation for Linux users so it is not re-investigated in future planning cycles.

6. **Manifest drift was the most common cross-cutting issue** — The Reviewer applied 7 Fix-Forward corrections across the cycle, mostly correcting stale `WinExe`/`WindowsFileLauncherService` references in `tech-stack.md`, `file-tree.md`, `AGENTS.md`, and XML doc comments. The Documentation agent also found that `ZipArchiveExtractorTests.cs` and the `Fixtures/archives/` directory were entirely absent from `file-tree.md`. The pattern suggests that manifest updates during implementation (not deferred to the documentation pipeline) would reduce the Fix-Forward load on Reviewers. Consider adding a manifest-check step to the Developer handoff checklist in `AGENTS.md`.

---

## Next Steps

| Priority | Recommendation |
|---|---|
| High | Verify application launches and operates correctly on a real Linux (`linux-x64`) target — the code changes are complete, but no end-to-end Linux smoke test exists yet. A future QA or release-engineering WP should capture this. |
| Medium | Plan a WP for `DesktopFileLauncherService` `Process.Start()` null-check + `ILogger` warning on launch failure. |
| Medium | Plan a WP for `ZipArchiveExtractor` and `TarXzArchiveExtractor` decompressed-size guard (Zip Bomb protection). |
| Low | Plan a WP for `TarXzArchiveExtractor` audit logging (detailed TODO already documented in the class). |
| Low | Add `CharacterDevice` and `Fifo` test fixtures for completeness of device-node guard coverage. |
| Low | Update `AGENTS.md` Developer handoff checklist to include a manifest-check reminder (update `file-tree.md` / `api-surface.md` before passing to QA). |

---

## Files Modified (Full List)

**Production source (4 files):**
- `src/VideoIndexer.App/VideoIndexer.App.csproj` — `OutputType` `WinExe` → `Exe`
- `src/VideoIndexer.App/Services/DesktopFileLauncherService.cs` — renamed + Linux xdg-open branch + XML doc corrections
- `src/VideoIndexer.App/Services/IFileLauncherService.cs` — `ShowInExplorer` XML doc clarified for Linux
- `src/VideoIndexer.Infrastructure/ExternalTools/ZipArchiveExtractor.cs` — `Ordinal` guard fix + class XML doc updated
- `src/VideoIndexer.Infrastructure/ExternalTools/TarXzArchiveExtractor.cs` — `Ordinal` guard fix + device-node guard + `await using` idiomatic fix

**Tests (2 files + 1 binary fixture):**
- `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/ZipArchiveExtractorTests.cs` — Ordinal regression gate test added
- `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/TarXzArchiveExtractorTests.cs` — device-node rejection test + fixture docs
- `tests/VideoIndexer.Infrastructure.Tests/Fixtures/archives/device.tar.xz` — new 212-byte PAX BlockDevice fixture

**Documentation (11 files):**
- `CHANGELOG.md` — `[Unreleased]` entry for `OutputType` change
- `AGENTS.md` — Output type stat updated
- `README.md` — Archive extractor guard docs updated; Running Tests section corrected
- `docs/agents/project-manifest/tech-stack.md` — Output type row updated
- `docs/agents/project-manifest/constraints.md` — `DesktopFileLauncherService` reference + ArgumentList rule updated
- `docs/agents/project-manifest/file-tree.md` — `DesktopFileLauncherService`, `ZipArchiveExtractorTests.cs`, `device.tar.xz` entries added/corrected
- `docs/agents/project-manifest/api-surface.md` — `DesktopFileLauncherService` section updated (WP-002 → WP-003)
- `docs/agents/project-manifest/program.cs` — DI comment corrected _(via Reviewer Fix-Forward)_
- `docs/projects/rebuild/milestones/m1-foundation.md` — AC2 updated to `Exe (cross-platform)`
- `src/VideoIndexer.App/Program.cs` — DI registration updated (rename)

---

*Report generated by Head of Operations (Synthesis Agent v3.5.3) on 2026-05-21.*
