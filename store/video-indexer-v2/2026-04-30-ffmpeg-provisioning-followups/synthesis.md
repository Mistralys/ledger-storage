# Synthesis Report — FFmpeg Provisioning Follow-ups (M2.5.1)

**Project:** `2026-04-30-ffmpeg-provisioning-followups`  
**Plan file:** `plan.md`  
**Date generated:** 2026-05-06  
**Status:** ALL 6 WORK PACKAGES COMPLETE — ZERO BLOCKING ISSUES  

---

## 1. Executive Summary

This plan closed all three M2.5 release-gate items (RG-1, RG-2, RG-3) and the
full set of actionable deferred items (D-1, D-2, D-3 deferred, D-4/D-6 already
closed, D-7, D-8, D-9, D-10/D-11/D-12 via documentation). No new user-visible
behaviour was introduced. The M2.5 FFmpeg provisioning subsystem is now
production-ready and unblocks the M3 Library & Indexing milestone.

**Key deliverables:**

| Deliverable | WP | Result |
|---|---|---|
| Real SHA-256 values in manifest (RG-1) | WP-002 | Closed — n7.1.4 BtbN autobuild |
| BtbN archive URLs corrected (RG-2, RG-3) | WP-002 | Closed — URL + checksum verified live |
| macOS two-archive model (evermeet.cx) | WP-002 | Closed — `FfprobeArchive` model added |
| D-1/D-2/D-7/D-8 source verification | WP-001 | Closed — all confirmed in-source |
| Provisioning-disabled regression test | WP-001 | Closed — new test added |
| `HttpToolDownloader` structured logging (D-9) | WP-003 | Closed — 4 log events documented |
| `.tar.xz` test fixtures from files (D-5) | WP-004 | Closed — 5 binary fixtures + Python regen script |
| `CompositeArchiveExtractor` DI cleanup (D-11) | WP-005 | Closed — `[ActivatorUtilitiesConstructor]` |
| Cross-plan doc consistency (D-10, UpdateAsync) | WP-006 | Closed — 3 plan docs corrected |
| Milestone doc + synthesis closure annotations | WP-006 | Closed — all RG/D rows annotated |

---

## 2. Metrics

### 2.1 Pipeline health

| WP | Stages | Rework | Final result |
|---|---|---|---|
| WP-001 | impl → qa → code-review | 0 | PASS / PASS / PASS |
| WP-002 | impl → qa → impl → qa → sec-audit → code-review → docs | 1 (QA bounce) | All PASS |
| WP-003 | impl → qa → sec-audit → code-review → docs | 0 | All PASS |
| WP-004 | impl → qa → code-review | 0 | All PASS |
| WP-005 | impl → qa → code-review → docs | 0 | All PASS |
| WP-006 | documentation | 0 | PASS |

**Total pipeline stages executed:** 21  
**Stages failed:** 1 (WP-002 QA-1 — bounced for rework)  
**Reviewer Fix-Forwards applied:** 1 (WP-003 — `OperationCanceledException` log level)  
**Critical/High blocking issues:** 0

### 2.2 Test results (final state)

| Assembly | Passed | Failed | Skipped |
|---|---|---|---|
| `VideoIndexer.Tests` | 50 | 0 | 0 |
| `VideoIndexer.Infrastructure.Tests` | 63 | 0 | 2 (live gate + tar.xz fixture) |
| `VideoIndexer.App.Tests` | 48 | 0 | 0 |
| **Total** | **161** | **0** | **2** |

All skips are expected: `VI_LIVE_DOWNLOAD` gate (requires network + x64 Windows) and
the tar.xz fixture skip guard. Build: 0 errors, 0 warnings (`TreatWarningsAsErrors=true`).

### 2.3 Security

| WP | Audited files | Critical | High | Medium | Low |
|---|---|---|---|---|---|
| WP-002 | 6 | 0 | 0 | 0 | 2 |
| WP-003 | 3 | 0 | 0 | 0 | 2 |
| **Total** | 9 | **0** | **0** | **0** | **4** |

All low-severity security observations are forward-looking notes, not active
vulnerabilities. See §4 for details.

---

## 3. Work Package Summaries

### WP-001 — Verify-and-close already-fixed items (D-1, D-2, D-7, D-8)

Verified by source inspection that all four items were already correctly
implemented: `IsCacheHitAsync` is `private static Task<bool>` with
`Task.FromResult` (no blocking calls); both FFprobe and FFmpeg paths are
null/empty-guarded with `File.Exists`; `Manifest` initialised with
`new Dictionary<string, ToolManifestEntry>()`; symlink/hard-link rejection
in `TarXzArchiveExtractor` placed before path operations.

One new test added: `EnsureAsync_ProvisioningDisabled_ShortCircuitsAndReturnsCachedPaths`
asserts the `Enabled=false` path bypasses both downloader and extractor and
returns cached tool paths verbatim.

**Outstanding doc note (documentation-forward):** The `Enabled=false` branch in
`FfmpegProvisioner.EnsureAsync` should have an inline comment noting that path
validation is the caller's responsibility in this mode.

### WP-002 — Real SHA-256 values + FfprobeArchive support (RG-1/2/3)

The BtbN `n7.1` tag did not exist; all three URLs were 404. Corrected to the
pinned autobuild `autobuild-2026-05-06-13-32` with archives named
`ffmpeg-n7.1.4-{platform}-gpl-7.1.{ext}`. SHA-256 values sourced from BtbN's
`checksums.sha256` and independently verified by live download.

evermeet.cx `ffmpeg-7.1.zip` contains only the `ffmpeg` binary. The
`ToolManifestEntry` model was extended with an optional `FfprobeArchiveEntry`
sub-record. `FfmpegProvisioner` was updated with Step 8b: download, verify, and
extract from the secondary archive when `FfprobeArchive` is set.

QA found two defects on first pass: (1) the `FfprobeArchive` code path had zero
unit tests; (2) `targetDir` was not cleaned up on `FfprobeArchive` SHA-256
failure. Both were fixed in rework. Final test count: 161 passed, 0 failed.

**Ongoing technical debt:**
- osx-arm64 falls back to Intel binary via Rosetta 2 — no native Apple Silicon
  source available from evermeet.cx at this time.
- BtbN URLs are pinned to a specific autobuild for SHA-256 stability. A periodic
  refresh procedure is needed to pick up future FFmpeg 7.1.x security patches.

### WP-003 — HttpToolDownloader structured logging (D-9)

`ILogger<HttpToolDownloader>` constructor parameter added with
`ArgumentNullException` guard. Four structured log events emitted:

| Event | Level | Key fields |
|---|---|---|
| Download starting | Information | URL, destination, content-length |
| Download succeeded | Information | URL, bytes, elapsed (stream+write) |
| Download cancelled | **Information** | URL, bytes, exception type |
| Download failed | Error | URL, bytes, exception type |

The Reviewer applied a Fix-Forward promoting `OperationCanceledException` from
`LogError` to `LogInformation` to prevent user-initiated cancellations from
polluting error monitoring dashboards. README.md updated with a structured
logging table and notes on `ElapsedMs` scope and cancellation log level.

### WP-004 — TarXzArchiveExtractor test fixture refactoring

Five binary `.tar.xz` fixtures (flat, nested, slip1, slip2, symlink) extracted
from inline `byte[]` constants in `TarXzArchiveExtractorTests.cs` and written to
`Fixtures/archives/`. The `.csproj` uses a glob `<None Include="Fixtures/archives/*.tar.xz" ... />`
so future fixtures are auto-included. `FixturePath()` helper uses
`AppContext.BaseDirectory` — the standard pattern for test output directories.

An inline `ARCHIVE CONTENTS SUMMARY` block and Python regeneration script were
added as code comments — an excellent maintainability investment for future
contributors.

### WP-005 — CompositeArchiveExtractor DI cleanup

`[ActivatorUtilitiesConstructor]` attribute added to the parameterless
constructor of `CompositeArchiveExtractor`. `Program.cs` updated from a factory
lambda to the idiomatic `services.AddSingleton<IArchiveExtractor, CompositeArchiveExtractor>()`.
README.md DI registration section updated to reflect the new pattern and explain
why the parameterless ctor is selected (avoiding circular resolution of both
`IArchiveExtractor` parameters in the 2-parameter overload).

### WP-006 — Cross-plan documentation consistency

`ISettingsService.UpdateAsync` (which does not exist — the method is `SaveAsync`)
replaced in three plan documents: `plan.md`, `work-packages-draft.md`, and
`work/WP-007.md`. Closure annotations added to all deferred-item rows in
`synthesis.md` (D-3 marked as deferred with rationale). Milestone document
`m2-5-ffmpeg-provisioning.md` Known Deferred Items table updated for D-3, D-4,
and the logging row.

---

## 4. Security Observations

**All low severity — no active vulnerabilities.**

| Ref | Area | Observation | Status |
|---|---|---|---|
| A06 | Outdated components | BtbN URLs pinned to `autobuild-2026-05-06-13-32`; no CVEs at time of review | Carried — needs periodic refresh |
| A09 | Logging | `TarXzArchiveExtractor` emits no audit log for extraction events (pre-existing TODO, tracked in source) | Carried — pre-existing |
| A10 | URL allowlist | `HttpToolDownloader.DownloadAsync` passes `url` directly to `HttpClient` without scheme/host validation; fully mitigated by appsettings-controlled manifest | Noted — no active risk |
| A09/A03 | URL logging | Full manifest URLs logged at `LogInformation`; no credentials currently — forward-looking risk if authenticated URLs are ever used | Noted — no active risk |

**Positive security assurances confirmed:**

- Zip-Slip and Tar-Slip guards present and correct in both extractors.
- `TarXzArchiveExtractor` additionally rejects symbolic links and hard links unconditionally.
- SHA-256 verification applied to **both** the main `ffmpeg` archive and the secondary `FfprobeArchive` before extraction.
- Cleanup invariant enforced: both `partialDir` and `targetDir` are deleted on any checksum failure.
- Archive filename extraction uses `Path.GetFileName(new Uri(url).LocalPath)` — prevents path traversal in staged filenames.

---

## 5. Strategic Recommendations (Gold Nuggets)

### 5.1 Add audit logging to TarXzArchiveExtractor [Priority: Medium]

`TarXzArchiveExtractor` emits no log events for extraction start, Tar-Slip guard
triggers, or extraction completion. A Security Auditor finding (A09) documents
this as a pre-existing gap with a TODO comment in the source (referencing a prior
Security Auditor finding). Recommend injecting `ILogger<TarXzArchiveExtractor>`
and adding `LogDebug`/`LogWarning`/`LogInformation` events as the TODO describes.
This is the most actionable security item remaining in the provisioning subsystem.

### 5.2 Periodic BtbN manifest refresh procedure [Priority: Medium]

The BtbN archive URLs are pinned to a specific autobuild date for SHA-256
stability. A documented procedure (and ideally a CI check) should exist to
refresh the manifest to the latest FFmpeg 7.1.x point release when security
patches are issued. The current `AppsettingsJson_Manifest_Sha256ValuesAreHexStrings`
test is the ideal place to gate this: a PR that updates the checksums will turn
it red if the format changes, preventing silent corruption.

### 5.3 Resolve native osx-arm64 FFmpeg source [Priority: Low]

The current osx-arm64 manifest entry uses the Intel x64 binary from evermeet.cx
via Rosetta 2 compatibility. This works today but is suboptimal for Apple Silicon
performance. A dedicated ARM64 build source should be monitored and the manifest
updated when one becomes available (e.g., a future evermeet.cx ARM64 release or
a static build from another trusted source).

### 5.4 Add HttpToolDownloader URL allowlist [Priority: Low]

`HttpToolDownloader.DownloadAsync` currently passes URLs directly to
`HttpClient.GetAsync` without scheme or host validation. All callers are
application-controlled (appsettings.json manifest), so no SSRF risk exists today.
A scheme-check (`https://` only) and optional host-allowlist (e.g., `github.com`,
`evermeet.cx`) would provide defence-in-depth if provisioning URLs ever originate
from less-trusted input in future.

### 5.5 D-3 (ShellView wiring) remains deferred [Priority: Low]

The D-3 deferred item (consolidate ShellView state machine wiring) had an invalid
synthesis premise — the code it referenced does not exist in the current
implementation. It should be reopened as a fresh work package once the ShellView
wiring work package ships and the relevant ViewModel code is in place.

---

## 6. Outstanding Documentation-Forward Items

These items were logged by Reviewer agents and are not blocking — they represent
incremental improvements to inline documentation:

| File | Item |
|---|---|
| `FfmpegProvisioner.cs` | Add inline comment to `Enabled=false` branch noting that path validation is the caller's responsibility when provisioning is disabled |
| `FfmpegProvisioner.cs` | Update `Step 8b` cancellation handler comment to note that `targetDir` contains a partial result (ffmpeg present, ffprobe absent) after cancellation |
| `ToolManifestEntry.cs` | Class-level doc should explain `FfprobeRelativePath` required-but-ignored design trade-off when `FfprobeArchive` is set |
| `HttpToolDownloader.cs` | XML doc on `DownloadAsync` should clarify that `ElapsedMs` in the success log measures streaming/file-write only, not the full HTTP round-trip |
| `FfmpegProvisionerTests.cs` | `ComputeHex` uses `Convert.ToHexString()` (uppercase) while production convention is lowercase; functionally harmless due to case-insensitive SHA-256 comparison |

---

## 7. Next Steps for Planner / Project Manager

1. **Open TarXzArchiveExtractor logging WP** — inject `ILogger<TarXzArchiveExtractor>`,
   add the four structured log events described in the TODO comment. The Security
   Auditor A09 finding has been carried twice; this should be the next
   infrastructure hardening item.

2. **Schedule BtbN manifest refresh** — establish a recurring review cadence
   (e.g., monthly or with each FFmpeg 7.1.x point release). The smoke test
   (`AppsettingsJson_Manifest_Sha256ValuesAreHexStrings`) acts as the gating
   check.

3. **Reopen D-3 after ShellView wiring WP** — once the ShellView state machine
   wiring is implemented, revisit the consolidation item with accurate source
   references.

4. **Address documentation-forward items** — the five inline documentation items
   in §6 can be batched into a single small documentation WP or folded into the
   next developer touching those files.

5. **Proceed to M3 Library & Indexing** — all M2.5 release gates are now closed.
   The provisioning subsystem is stable, tested (161/161 non-gated tests pass),
   and security-audited. M3 may depend on it without concern.
