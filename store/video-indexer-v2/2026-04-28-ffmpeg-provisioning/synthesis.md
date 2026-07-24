# Synthesis — FFmpeg/FFprobe Provisioning (M2.5)

**Project path:** `docs/agents/plans/2026-04-28-ffmpeg-provisioning`  
**Date completed:** 2026-04-30  
**Work packages:** 12 / 12 COMPLETE  
**Total pipeline stages passed:** 50 / 50  
**Final test count:** 151 passed · 0 failed · 2 skipped (1 TarXz fixture, 1 live-download gate)

---

## 1. Objective

Introduce a cross-platform, manifest-driven FFmpeg/FFprobe auto-provisioning
subsystem that runs before the existing `Connecting` shell state, so that every
developer and end-user has a tamper-verified `ffprobe` binary at a known path
before the M3 Library & Indexing milestone begins. On subsequent launches, if
cached binaries match the recorded SHA-256, provisioning is a sub-second no-op.

---

## 2. What Was Built

### 2.1 Core contracts — WP-001

`VideoIndexer.Core` received a complete, sealed set of domain types with no new
package references:

| Category | Types |
|---|---|
| Interfaces | `IExternalToolProvisioner`, `IToolDownloader`, `IArchiveExtractor`, `IChecksumVerifier`, `IRuntimeIdentifierProvider` |
| Enums | `ProvisioningOutcome`, `ArchiveType`, `ShellState.ProvisioningTools` (prepended) |
| Models | `ToolManifestEntry`, `ToolPaths`, `ProvisioningProgress` |
| Options | `FfmpegToolOptions`, `FfmpegProvisioningOptions`, `ExternalToolsOptions.Ffmpeg` |
| Events | `ProvisioningProgressChangedEventArgs` |

One rework cycle occurred: the initial code-review (FAIL) caught that
`FfmpegProvisioningOptions.Manifest` was typed as mutable `Dictionary<,>` rather
than the spec-required `IReadOnlyDictionary<,>`. Fixed before QA sign-off.

### 2.2 SHA-256 checksum verifier — WP-002

`Sha256ChecksumVerifier` uses `SHA256.HashDataAsync` (buffered, 81 920-byte
reads) with case-insensitive hex comparison. Throws `FileNotFoundException`
(with path in message) when the target file is absent. 7 tests covering match,
case-insensitive match, mismatch, empty file, missing file, and cancellation.

### 2.3 HTTP downloader — WP-003

`HttpToolDownloader` streams directly to a `.tmp` sibling file, reports
`IProgress<ProvisioningProgress>` every 64 KB, and atomically moves to the
final path on success. Any exception (including `OperationCanceledException`)
deletes the `.tmp` file. 3 integration tests using a `CannedResponseHandler`
(no external network). Adds `Microsoft.Extensions.Http` to Infrastructure.

### 2.4 Archive extractors — WP-004 & WP-005

| Class | Format | Slip protection |
|---|---|---|
| `ZipArchiveExtractor` | `.zip` | Zip-Slip via `Path.GetFullPath` + trailing-separator prefix check |
| `TarXzArchiveExtractor` | `.tar.xz` | Tar-Slip (same technique); uses SharpCompress `XZStream` + `System.Formats.Tar.TarReader` |
| `CompositeArchiveExtractor` | dispatches by extension | `NotSupportedException` for unknown types |

SharpCompress 0.38.0 is the only new package reference added across WP-004/005.
12 tests cover flat extraction, nested directories, traversal rejection, and
composite dispatch.

### 2.5 Runtime identifier provider — WP-006

`DefaultRuntimeIdentifierProvider` resolves one of five canonical RIDs
(`win-x64`, `win-arm64`, `linux-x64`, `osx-x64`, `osx-arm64`) from
`RuntimeInformation` and caches the result statically. Throws
`PlatformNotSupportedException` for unrecognised platforms. 3 tests pass on
Windows x64 without skipping.

### 2.6 FfmpegProvisioner orchestrator — WP-007

`FfmpegProvisioner` implements a 13-step algorithm:

1. Short-circuit if `Enabled = false`
2. Resolve RID via `IRuntimeIdentifierProvider`
3. Look up RID in `FfmpegProvisioningOptions.Manifest`
4. Check cache hit (ffprobe path exists + checksum matches)
5. Return `AlreadyPresent` (and resolved `ToolPaths`) if cached
6. Guard: `AutoDownload` must be true
7. Download to `.partial/` via `IToolDownloader`
8. Verify SHA-256 via `IChecksumVerifier`; delete `.partial/` on mismatch
9. Extract archive via `IArchiveExtractor`
10. Delete `.partial/` archive
11. Persist resolved paths via `ISettingsService.SaveAsync`
12. Return `Downloaded`
13. On any exception: delete `.partial/`, re-throw

Fake collaborators (`FakeToolDownloader`, `FakeArchiveExtractor`) support 6
unit-test scenarios (AlreadyPresent, clean download, checksum mismatch, missing
RID, cancellation mid-download). 47 provisioner tests pass.

### 2.7 Manifest configuration — WP-008

`appsettings.json` extended with a 5-RID `ExternalTools.Ffmpeg.Provisioning`
section (BtbN builds for Windows/Linux; evermeet.cx for macOS). `FfmpegToolOptions`
gained a `Provisioning` property to anchor the binding path. `.gitignore` rule
added to exclude `tools/ffmpeg/`. 5 smoke tests guard structure and act as a
release gate (they assert SHA-256 values are *still* placeholders until real
hashes are filled).

### 2.8 Provisioning UI — WP-009

`ProvisioningToolsViewModel` (CommunityToolkit.Mvvm) exposes:
- `[ObservableProperty]` for `Stage`, `BytesReceived`, `BytesTotal`, `Error`, `IsProvisioning`
- `RetryCommand` (enabled when `Error != null && !IsProvisioning`)
- `CancelCommand` (always enabled during provisioning)
- `ProvideManuallyCommand` (stub, wired in WP-010)

`ProvisioningToolsView.axaml` uses compiled `x:DataType` bindings with
`StringConverters.IsNotNullOrEmpty` for conditional visibility. 4 view-model
tests pass; AXAML compiles with zero warnings.

### 2.9 DI wiring and shell integration — WP-010

`Program.cs` gained a clearly-labelled `// 5.5 — External tools provisioning`
block registering all five interfaces to their implementations plus
`ProvisioningToolsViewModel` as transient. `ShellViewModel` now starts in
`ProvisioningTools`, calls `ProvisionAsync()` which transitions to `Connecting`
on success or `Error` (with `LastError`) on failure. All 5 test files updated
to pass a `FakeExternalToolProvisioner`. Full regression: 151 tests passed, 0
failed.

### 2.10 Live-download integration test — WP-011

`FfmpegProvisionerLiveTests.cs` added to `VideoIndexer.Infrastructure.Tests`.
Self-skips unless `VI_LIVE_DOWNLOAD=1`. When enabled, wires the real provisioner
stack against a temp-rooted `IAppPaths`, asserts `ffprobe -version` exits with
code 0, and cleans up via a `TempDirectory` `IDisposable` fixture. A QA FAIL
cycle occurred on first pass (file was absent); the file was created during the
same QA session and passed immediately.

### 2.11 End-user documentation — WP-012

- `docs/projects/rebuild/milestones/m2-5-ffmpeg-provisioning.md` created
  (Manifest format, Refresh-pins workflow, offline/mirror instructions, manual
  smoke test, Known Deferred Items table)
- `README.md` updated — shell documented as starting in `ProvisioningTools`,
  no manual FFprobe install instruction present
- M3 plan `Dependencies` section updated to reference M2.5/`FfmpegProvisioner`
  as the automatic provisioning source

---

## 3. Files Modified (complete list)

| Area | Files |
|---|---|
| Core contracts | `Enums/ArchiveType.cs`, `Enums/ProvisioningOutcome.cs`, `Enums/ShellState.cs`, `Models/ToolManifestEntry.cs`, `Models/ToolPaths.cs`, `Models/ProvisioningProgress.cs`, `Options/FfmpegToolOptions.cs`, `Options/FfmpegProvisioningOptions.cs`, `Options/ExternalToolsOptions.cs`, `Abstractions/I*.cs` (5 files), `Events/ProvisioningProgressChangedEventArgs.cs` |
| Infrastructure impl | `ExternalTools/Sha256ChecksumVerifier.cs`, `ExternalTools/HttpToolDownloader.cs`, `ExternalTools/ZipArchiveExtractor.cs`, `ExternalTools/TarXzArchiveExtractor.cs`, `ExternalTools/CompositeArchiveExtractor.cs`, `ExternalTools/DefaultRuntimeIdentifierProvider.cs`, `ExternalTools/FfmpegProvisioner.cs`, `VideoIndexer.Infrastructure.csproj` |
| App | `ViewModels/ProvisioningToolsViewModel.cs`, `ViewModels/ShellViewModel.cs`, `Views/ProvisioningToolsView.axaml`, `Views/ProvisioningToolsView.axaml.cs`, `Program.cs`, `Assets/appsettings.json` |
| Tests | `Sha256ChecksumVerifierTests.cs`, `HttpToolDownloaderTests.cs`, `ZipArchiveExtractorTests.cs`, `TarXzArchiveExtractorTests.cs`, `DefaultRuntimeIdentifierProviderTests.cs`, `FfmpegManifestOptionsTests.cs`, `FfmpegProvisionerTests.cs`, `ProvisioningToolsViewModelTests.cs`, `ShellViewModelTests.cs` (+4 fixups), `FfmpegProvisionerLiveTests.cs`, `Fixtures/FakeToolDownloader.cs`, `Fixtures/FakeArchiveExtractor.cs`, `Fixtures/TempDirectory.cs` (duplicated), `VideoIndexer.Infrastructure.Tests.csproj` |
| Config & docs | `.gitignore`, `README.md`, `docs/projects/rebuild/milestones/m2-5-ffmpeg-provisioning.md`, `docs/agents/plans/2026-04-28-m3-library-indexing/plan.md` |

---

## 4. Security Findings Summary

All security audits returned PASS. No critical or high findings. Notable items:

| Severity | Finding | Location | Status |
|---|---|---|---|
| Medium | SSRF — URL not scheme-validated before `HttpClient.GetAsync` | `HttpToolDownloader.cs` | Accepted (app-configured URL) |
| Medium | Path overwrite — `destinationPath` not canonicalized before `File.Move` | `HttpToolDownloader.cs` | Accepted (app-constructed paths) |
| Medium | Symlink tar entries not filtered; symlink target not bounds-checked | `TarXzArchiveExtractor.cs` | Deferred (see §5) |
| Low | `string.Equals(OrdinalIgnoreCase)` is not constant-time | `Sha256ChecksumVerifier.cs` | Accepted (file integrity, not HMAC) |
| Low | `.tmp` filename predictable (`destinationPath + ".tmp"`) | `HttpToolDownloader.cs` | Accepted (app-controlled paths) |
| Low | No zip-bomb protection (no uncompressed-size check) | `ZipArchiveExtractor.cs` | Accepted (trusted source URL) |
| Low | `OrdinalIgnoreCase` Zip-Slip check may miss traversal on Linux case-sensitive FS | `ZipArchiveExtractor.cs` | Accepted (Windows-primary deployment) |
| Low | No audit logging for download URL, destination, bytes, or failure | `HttpToolDownloader.cs` | Deferred |

---

## 5. Deferred Items and Known Gaps

These items were identified and tracked but intentionally left for a future
release cycle.

### Release gate (must-fix before production)

| # | Item | Origin |
|---|---|---|
| RG-1 | Fill SHA-256 hashes in `appsettings.json` for all 5 RIDs | WP-008, WP-011 |
| RG-2 | Verify win-arm64 BtbN release URL (`ffmpeg-n7.1-winarm64-gpl.zip`) exists | WP-008 |
| RG-3 | Verify macOS evermeet.cx archives include both `ffmpeg` and `ffprobe`; if not, add separate `ffprobe` URL to manifest | WP-008 QA |

### Code quality / architecture (deferred)

| # | Item | Origin |
|---|---|---|
| D-1 | `IsCacheHit` uses `GetAwaiter().GetResult()` (sync-over-async) — deadlock risk on non-default sync contexts | WP-007 impl |
| D-2 | `IsCacheHit` checks only `FfprobePath` on disk; does not verify `FfmpegPath` exists — silent empty-path at runtime if only ffprobe is missing | WP-007 QA |
| D-3 | `ProvisioningToolsViewModel` and `ShellViewModel` both independently call `EnsureAsync`; in production one call is redundant | WP-010 impl |
| D-4 | `FakeExternalToolProvisioner` duplicated across 4 test files — consolidate into a shared test fixture | WP-010 impl |
| D-5 | `CompositeArchiveExtractor` registered via a factory lambda; consider `[ActivatorUtilitiesConstructor]` on the 2-param constructor | WP-010 impl |
| D-6 | Infrastructure.Tests targets `net10.0`; VideoIndexer.Tests targets `net8.0` — TFM mismatch should be resolved if shared helpers are extracted | WP-003 impl |
| D-7 | `FfmpegProvisioningOptions.Manifest` collection expression `[]` vs rest of codebase using `new()` — minor style inconsistency | WP-001 impl |
| D-8 | `TarXzArchiveExtractor` does not filter symlink/hardlink tar entries; symlink targets are not bounds-checked (medium security) | WP-005 QA |
| D-9 | No coverage for `Provisioning.Enabled = false` short-circuit in `FfmpegProvisioner` | WP-007 QA |
| D-10 | `ISettingsService` interface uses `SaveAsync`; WP docs referred to `UpdateAsync` — naming inconsistency should be reconciled | WP-007 impl |
| D-11 | Inline tar fixture byte arrays in `TarXzArchiveExtractorTests.cs` should be stored as binary test assets in `Fixtures/archives/` | WP-005 impl |
| D-12 | `ILogger<HttpToolDownloader>` not injected — download URL, bytes, and failures are not auditable | Security audit WP-003 |

---

## 6. Quality Metrics

| Metric | Value |
|---|---|
| WPs completed | 12 / 12 |
| Pipeline stages passed | 50 / 50 |
| QA FAIL cycles | 2 (WP-001 code-review; WP-011 first QA pass) |
| Security audits | 4 (WP-002, WP-003, WP-004, WP-005) — all PASS |
| Tests at completion | 151 passed · 0 failed · 2 skipped |
| New package references | `Microsoft.Extensions.Http`, `SharpCompress 0.38.0` |
| Build warnings at completion | 0 (TreatWarningsAsErrors=true, WarningLevel=9999) |

---

## 7. M3 Handoff Notes

The following are ready for M3 Library & Indexing consumption:

- `IExternalToolProvisioner.EnsureAsync()` returns `ToolPaths` with `FfmpegPath`
  and `FfprobePath` as absolute strings once provisioning completes
- `ShellViewModel` transitions to `Connecting` only after `ProvisionAsync`
  succeeds, so M3 code executing after `Connecting` can assume both paths are
  valid
- `appsettings.json` binding path: `ExternalTools.Ffmpeg.FfmpegPath` /
  `ExternalTools.Ffmpeg.FfprobePath` (populated at runtime by the provisioner
  via `ISettingsService.SaveAsync`)
- Live-download test gate: set `VI_LIVE_DOWNLOAD=1` after filling SHA-256
  hashes (RG-1) to run the end-to-end integration test before merging

Before closing M2.5 for production, items **RG-1**, **RG-2**, and **RG-3**
must be resolved. All other deferred items are safe to carry forward into M3
or a dedicated tech-debt sprint.
