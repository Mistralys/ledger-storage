# Plan — FFmpeg/FFprobe Provisioning (M2.5 prerequisite for M3)

## Summary

Introduce an **automatic, cross-platform FFmpeg/FFprobe provisioning subsystem**
that runs **before the M2 `Connecting` state** so that by the time the M3
*Library & Indexing* milestone begins implementation, every developer and end
user has a verified `ffprobe` binary on disk at a known path. The provisioning
flow ships as a new shell state (`ProvisioningTools`) with a small Avalonia
view that displays download progress, and it persists the resolved binary paths
back into the app settings tree for later consumption by the M3
[`FfprobeRunner`](../2026-04-28-m3-library-indexing/plan.md). All downloads are
governed by a **manifest baked into `appsettings.json`** that names a URL and a
**SHA-256 checksum per Runtime Identifier**, so binaries are tamper-verified
and the user can override the source for offline / mirror scenarios. No
third-party FFmpeg NuGet packages are introduced; the existing CLI-wrapping
approach (the same pattern used by the legacy
[`SPDB Indexer`](../../../../../spdb-indexer/SPDB%20Indexer/) project's
`ffmpeg/` folder) is preserved.

Done means: from a fresh clone with no `ffprobe` on `PATH`, launching
`VideoIndexer.App` shows a *Provisioning external tools* screen, downloads the
correct binaries for the host OS into
`%APPDATA%/VideoIndexer/tools/ffmpeg/<version>/`, verifies them against the
manifest's SHA-256, writes the resolved paths into `appsettings.json` under a
new `ExternalTools.Ffmpeg` section, and then advances the shell to
`Connecting`. On every subsequent launch, if the cached binaries exist and
match the recorded checksum, provisioning is a sub-second no-op and the user
sees the existing `Connecting` screen immediately.

## Architectural Context

The M1 + M2 shell at [src/](../../../../src/) is the integration target.
Relevant existing pieces:

- **Shell state machine** —
  [`src/VideoIndexer.Core/Enums/ShellState.cs`](../../../../src/VideoIndexer.Core/Enums/ShellState.cs)
  defines `Connecting → LoggingOn|SettingPassword → Ready → Error`, driven by
  [`src/VideoIndexer.App/ViewModels/ShellViewModel.cs`](../../../../src/VideoIndexer.App/ViewModels/ShellViewModel.cs)
  and the `viewModelFactory` switch in
  [`src/VideoIndexer.App/Program.cs`](../../../../src/VideoIndexer.App/Program.cs)
  (lines ~74–84). New states must extend both, and the factory is the single
  insertion point for the new view-model.
- **Settings persistence** —
  [`src/VideoIndexer.Infrastructure/AppPaths.cs`](../../../../src/VideoIndexer.Infrastructure/AppPaths.cs)
  resolves `Root` to `%APPDATA%/VideoIndexer/`. The new `tools/` subtree lives
  alongside the existing `logs/`. `JsonSettingsService` (M1) handles atomic
  `.tmp + File.Move` writes — the provisioner reuses it to persist resolved
  paths.
- **Options tree** — additions go through
  [`src/VideoIndexer.Core/Options/AppOptions.cs`](../../../../src/VideoIndexer.Core/Options/AppOptions.cs)
  and extend the existing
  [`ExternalToolsOptions`](../../../../src/VideoIndexer.Core/Options/ExternalToolsOptions.cs)
  record (which already holds `VlcExecutablePath`, `MySqlBinaryFolder`,
  `ExternalXmlEditorPath`).
- **DI bootstrap** — the numbered 7-step block in
  [`Program.cs`](../../../../src/VideoIndexer.App/Program.cs) is the single
  composition root. A new step **5.5: External tools provisioning** slots
  between the M2 infrastructure registration (step 5) and the shell state
  machine registration (step 6). The shell factory must take a dependency on
  the new provisioner so `ProvisioningTools` is the boot-time initial state.
- **Tests** — [`tests/VideoIndexer.Tests/`](../../../../tests/VideoIndexer.Tests/)
  for unit tests against in-memory fakes;
  [`tests/VideoIndexer.Infrastructure.Tests/`](../../../../tests/VideoIndexer.Infrastructure.Tests/)
  for live-on-disk integration tests (no live network in CI; integration tests
  use a local HTTP fixture or `[SkippableFact]`).
- **Legacy reference (read-only)** —
  [`spdb-indexer/SPDB Indexer/ffmpeg/`](../../../../../spdb-indexer/SPDB%20Indexer/ffmpeg/)
  shows the legacy bundled-binary layout. The legacy
  [`Hash.cs`](../../../../../spdb-indexer/SPDB%20Indexer/Classes/Hash.cs)
  invoked `ffprobe.exe` via `Process.Start` with no checksum verification —
  the rebuild adds the verification step but otherwise keeps the wrapping
  approach.

### Conventions inherited (MUST be preserved)

- Sealed records with `init`-only properties for option / DTO types; full
  nullable annotations; `TreatWarningsAsErrors=true`; `<WarningLevel>9999</WarningLevel>`.
- Atomic file writes (`.tmp` + `File.Move(overwrite:true)`) on every settings
  mutation.
- xUnit + FluentAssertions; `Subject_Scenario_ExpectedBehavior` naming;
  in-memory fakes; `IDisposable` temp-directory fixtures.
- Dependency direction: `Core → ∅`, `Infrastructure → Core`, `App → Core +
  Infrastructure`, `Tests → Core + Infrastructure`.
- Cross-platform constraint from `rebuild.md`: no Windows-only APIs; binary
  selection driven by `RuntimeInformation.RuntimeIdentifier` /
  `IsOSPlatform`.

## Approach / Architecture

### New solution shape (additions only)

```
src/
├── VideoIndexer.Core/
│   ├── Abstractions/
│   │   ├── IExternalToolProvisioner.cs       (NEW: EnsureAsync(IProgress<...>, ct) → ToolPaths)
│   │   ├── IToolDownloader.cs                (NEW: stream URL → file with progress)
│   │   ├── IArchiveExtractor.cs              (NEW: extract zip / tar.xz to a target dir)
│   │   ├── IChecksumVerifier.cs              (NEW: SHA-256 compare)
│   │   └── IRuntimeIdentifierProvider.cs     (NEW: resolves the active RID for manifest lookup)
│   ├── Enums/
│   │   ├── ProvisioningOutcome.cs            (NEW: AlreadyPresent | Downloaded | Failed)
│   │   └── ShellState.cs                     (MODIFIED: prepend ProvisioningTools)
│   ├── Events/
│   │   └── ProvisioningProgressChangedEventArgs.cs   (NEW)
│   ├── Models/                               (NEW folder)
│   │   ├── ToolManifestEntry.cs              (NEW: Rid, Url, Sha256, ArchiveType, FfprobeRelativePath, FfmpegRelativePath, Version)
│   │   ├── ToolPaths.cs                      (NEW: FfprobePath, FfmpegPath, Version)
│   │   └── ProvisioningProgress.cs           (NEW: Stage, BytesReceived, BytesTotal?, Message)
│   └── Options/
│       ├── FfmpegProvisioningOptions.cs      (NEW: Manifest dictionary, Enabled flag, AutoDownload flag)
│       ├── ExternalToolsOptions.cs           (MODIFIED: add Ffmpeg property = FfmpegToolOptions { FfprobePath?, FfmpegPath?, ResolvedVersion? })
│       └── (AppOptions unchanged — provisioning options live under ExternalTools)
└── VideoIndexer.Infrastructure/
    └── ExternalTools/                        (NEW folder)
        ├── HttpToolDownloader.cs             (NEW: HttpClient + IProgress, resumable not required)
        ├── Sha256ChecksumVerifier.cs         (NEW)
        ├── ZipArchiveExtractor.cs            (NEW: handles .zip — Windows builds)
        ├── TarXzArchiveExtractor.cs          (NEW: handles .tar.xz — Linux builds; uses System.Formats.Tar + XZ via SharpCompress)
        ├── CompositeArchiveExtractor.cs      (NEW: dispatches by ArchiveType)
        ├── DefaultRuntimeIdentifierProvider.cs (NEW)
        └── FfmpegProvisioner.cs              (NEW: orchestrates ensure → download → verify → extract → persist)

src/VideoIndexer.App/
├── ViewModels/
│   └── ProvisioningToolsViewModel.cs         (NEW: observable Stage/Progress/Error/Retry)
├── Views/
│   └── ProvisioningToolsView.axaml(.cs)      (NEW: progress bar, stage label, retry/cancel buttons)
├── Program.cs                                (MODIFIED: step 5.5 registrations; shell factory adds ProvisioningTools branch and starts there)
└── Assets/appsettings.json                   (MODIFIED: add ExternalTools.Ffmpeg manifest section)

tests/
├── VideoIndexer.Tests/
│   ├── FfmpegProvisionerTests.cs             (NEW: in-memory fakes for downloader/extractor/verifier; covers AlreadyPresent, Downloaded, ChecksumMismatch retry, missing-RID failure, cancellation)
│   ├── Sha256ChecksumVerifierTests.cs        (NEW)
│   └── Fixtures/
│       ├── FakeToolDownloader.cs             (NEW: returns canned bytes + progress)
│       └── FakeArchiveExtractor.cs           (NEW: drops fake ffprobe[.exe] in target dir)
└── VideoIndexer.Infrastructure.Tests/
    └── ExternalTools/
        ├── HttpToolDownloaderTests.cs        (NEW: spins up an in-process Kestrel server serving a canned 1 KB blob; verifies progress + checksum)
        ├── ZipArchiveExtractorTests.cs       (NEW)
        ├── TarXzArchiveExtractorTests.cs     (NEW: SkippableFact if SharpCompress unavailable)
        └── FfmpegProvisionerLiveTests.cs     (NEW: SkippableFact — actually downloads from manifest URL when env var VI_LIVE_DOWNLOAD=1)
```

### Composition root changes

In the existing numbered DI block in
[`Program.cs`](../../../../src/VideoIndexer.App/Program.cs), insert
**step 5.5: External tools provisioning** between step 5 (M2 infrastructure)
and step 6 (shell state machine). All registrations are singletons:

- `IRuntimeIdentifierProvider` → `DefaultRuntimeIdentifierProvider`
- `IToolDownloader` → `HttpToolDownloader` (uses an injected
  `IHttpClientFactory` so tests can swap `HttpClient`)
- `IChecksumVerifier` → `Sha256ChecksumVerifier`
- `IArchiveExtractor` → `CompositeArchiveExtractor` (which fans out to
  `ZipArchiveExtractor` / `TarXzArchiveExtractor`)
- `IExternalToolProvisioner` → `FfmpegProvisioner`
- `ProvisioningToolsViewModel` (transient)

The shell-factory switch in step 6 gains a `ProvisioningTools` branch that
returns a `ProvisioningToolsViewModel`. The `ShellViewModel` constructor
takes an additional `IExternalToolProvisioner` dependency and **its initial
state is `ProvisioningTools`**, not `Connecting`. The provisioner runs once
per process; on success the shell transitions itself to `Connecting`. Generic
Host structure is unchanged.

### State-machine extension

`ShellState` becomes:

```
ProvisioningTools → Connecting → LoggingOn | SettingPassword → Ready → Error
```

Transitions out of `ProvisioningTools`:

- **Success path:** `EnsureAsync` returns `AlreadyPresent` or `Downloaded` →
  `ShellViewModel` calls `TransitionTo(Connecting)`.
- **Failure path:** `EnsureAsync` throws or returns `Failed` → `ShellViewModel`
  calls `TransitionTo(Error)` with the provisioner's message preserved on
  `LastError` (already supported by the M2 shell).
- **Retry:** the `ProvisioningToolsView` exposes a *Retry* button that
  re-invokes `EnsureAsync`. *Cancel* exits the application
  (`IClassicDesktopStyleApplicationLifetime.Shutdown(1)`) — there is no
  meaningful Ready-without-ffprobe mode for M3+.

If `FfmpegProvisioningOptions.AutoDownload == false` and the binaries are
missing, the view surfaces a *"Provide a path to ffprobe / ffmpeg manually"*
fallback (file-picker over `IStorageProvider`). The chosen paths are validated
(file exists + `--version` exits 0) and persisted directly to
`ExternalTools.Ffmpeg`.

### Manifest model

`FfmpegProvisioningOptions`:

```csharp
public sealed record FfmpegProvisioningOptions
{
    public bool Enabled { get; init; } = true;
    public bool AutoDownload { get; init; } = true;
    public string DownloadVersion { get; init; } = "n7.1";   // pin
    public IReadOnlyDictionary<string, ToolManifestEntry> Manifest { get; init; }
        = new Dictionary<string, ToolManifestEntry>();   // keyed by RID
}

public sealed record ToolManifestEntry
{
    public required string Url { get; init; }
    public required string Sha256 { get; init; }              // hex, lowercase
    public required string ArchiveType { get; init; }         // "zip" | "tar.xz"
    public required string FfprobeRelativePath { get; init; } // path inside archive
    public required string FfmpegRelativePath { get; init; }
}
```

Bundled defaults in
[`Assets/appsettings.json`](../../../../src/VideoIndexer.App/Assets/appsettings.json)
ship four entries: `win-x64`, `win-arm64`, `linux-x64`, `osx-x64`,
`osx-arm64`. URLs point at a pinned BtbN release tag for Windows/Linux and
an evermeet.cx pinned build for macOS. SHA-256 values are committed to the
repo so any tamper-with-the-mirror attack fails locally.

> **Note for the user:** the actual SHA-256 hashes will be computed and
> committed by the engineer at implementation time by downloading the
> pinned URLs once and running `Get-FileHash`. The Planner does not
> fabricate hashes.

### Provisioner algorithm

`FfmpegProvisioner.EnsureAsync(IProgress<ProvisioningProgress>?, CancellationToken)`:

1. If `Options.Enabled == false` → return `AlreadyPresent` immediately
   (manual mode; user is on their own).
2. If `ExternalTools.Ffmpeg.FfprobePath` is set **and** the file exists **and**
   its SHA-256 matches the manifest entry for the active RID → return
   `AlreadyPresent`. (Skip checksum if the path was set manually, i.e., not
   under the managed `tools/ffmpeg/` subtree.)
3. Resolve `rid = IRuntimeIdentifierProvider.Current`. If absent from the
   manifest → throw `PlatformNotSupportedException` with a clear "no FFmpeg
   manifest entry for {rid}" message.
4. Compute target directory: `Path.Combine(appPaths.Root, "tools", "ffmpeg",
   options.DownloadVersion, rid)`.
5. If a previous partial download exists at `<target>.partial`, delete it.
6. Stream the URL into `<target>.partial/<archive-name>` via
   `IToolDownloader.DownloadAsync` reporting progress.
7. Verify SHA-256 of the downloaded archive against
   `ToolManifestEntry.Sha256`. On mismatch → delete and throw
   `InvalidDataException` ("checksum mismatch").
8. Extract via `IArchiveExtractor.ExtractAsync` into `<target>/`.
9. Compose the absolute paths from `FfprobeRelativePath` /
   `FfmpegRelativePath`.
10. On non-Windows, `chmod +x` the binaries
    (`File.SetUnixFileMode(path, ExecutableMask)`).
11. Persist the resolved paths through `ISettingsService.UpdateAsync` so the
    `appsettings.json` `ExternalTools.Ffmpeg` section reflects the result.
    Atomic write per the existing convention.
12. Delete the `.partial` directory.
13. Return `Downloaded`.

Cancellation is observed at every `await`. A cancelled run leaves the
`.partial` directory intact for cleanup on the next attempt — never the final
target directory.

### `ProvisioningToolsView` minimum surface

```
┌─────────────────────────────────────────────────────┐
│ Setting up external tools                           │
│                                                     │
│  Stage:    Downloading FFmpeg (n7.1, win-x64)…      │
│  [████████████████░░░░░░░░░] 64% (38.2 / 59.7 MB)   │
│                                                     │
│           [Cancel]   (on error: [Retry] [Provide…]) │
└─────────────────────────────────────────────────────┘
```

Bindings only; ViewLocator picks up the new view by convention.

## Rationale

- **Pre-`Connecting` provisioning** matches the user's roadmap (M3 needs
  `ffprobe` to exist) without coupling provisioning to the database tier.
  The shell state machine is the natural sequencing primitive — adding one
  state is cheaper than introducing a parallel boot pipeline.
- **Manifest with pinned URL + SHA-256 per RID** trades a small amount of
  manual maintenance (refreshing pins on FFmpeg upgrades) for full control
  over what gets installed on the user's machine and a verifiable supply
  chain. Third-party downloader packages either lag upstream or cede
  checksum control.
- **Storing under `AppPaths.Root/tools/`** means the binaries follow the
  per-user data directory the rest of the app already uses — no privileged
  install dir writes, no leakage between OS users on a shared machine,
  trivially replaceable by deleting one folder.
- **Reuse `ExternalToolsOptions`** rather than a new top-level options
  section. The record already exists for VLC and MySQL client paths;
  FFmpeg fits the same pattern (managed external tooling).
- **No FFmpeg NuGet dependency** keeps the solution surface small and
  honours the legacy app's CLI-wrapping approach. The M3
  [`FfprobeRunner`](../2026-04-28-m3-library-indexing/plan.md) plan already
  assumes a `Process.Start` wrapper — this milestone simply guarantees the
  binary is on disk before that runner needs it.
- **`ProvisioningTools` as a real shell state** (not a hidden splash) lets
  the UI surface progress, allow retry, and offer manual-path fallback —
  all without inventing a parallel UI subsystem.

## Detailed Steps

1. **Extend Core abstractions, models, enums, options.**
   - All new files under `Abstractions/`, `Models/`, `Enums/`, `Events/`,
     and `Options/` per the *New solution shape* table.
   - `ShellState` gains `ProvisioningTools` as the **first** member.
   - `ExternalToolsOptions` gains an `Ffmpeg` property of a new
     `FfmpegToolOptions` record holding resolved paths and version.
   - `FfmpegProvisioningOptions` is registered under
     `ExternalTools:Ffmpeg:Provisioning`.

2. **Add Infrastructure implementations.**
   - `HttpToolDownloader` uses `IHttpClientFactory.CreateClient("vi-tools")`;
     no automatic decompression; reports `IProgress<long>` every 64 KB.
   - `Sha256ChecksumVerifier` streams the file once via `SHA256.HashData`.
   - `ZipArchiveExtractor` uses `System.IO.Compression.ZipFile` with
     traversal protection (rejects entries whose normalised path escapes
     the target).
   - `TarXzArchiveExtractor` adds a single tactical NuGet reference to
     `SharpCompress` (LGPL-compatible) for the XZ stream, then hands the
     decompressed `tar` to `System.Formats.Tar`. Same traversal
     protection.
   - `CompositeArchiveExtractor` dispatches by `ArchiveType`.
   - `FfmpegProvisioner` orchestrates per the algorithm above.

3. **Wire the manifest defaults into `appsettings.json`.**
   - Five RID entries (Windows x64/arm64, Linux x64, macOS x64/arm64).
   - `Enabled = true`, `AutoDownload = true`, `DownloadVersion = "n7.1"`.
   - SHA-256 placeholders written with `"TODO_FILL_AT_IMPLEMENTATION"` and
     a checklist item in the milestone doc to compute and commit the
     hashes before merge.

4. **Register services in `Program.cs`** as a new step 5.5.
   - Add `IExternalToolProvisioner` to the `ShellViewModel` constructor.
   - The shell factory's switch gains the `ProvisioningTools → new
     ProvisioningToolsViewModel(provisioner)` arm.
   - Initial `CurrentViewModel` becomes the `ProvisioningToolsViewModel`,
     not the `DatabaseConnectorViewModel`. The fix-up at the bottom of
     the factory closure changes accordingly.

5. **Build the view-model and view.**
   - `ProvisioningToolsViewModel` exposes `Stage`, `Progress`, `Error`,
     `RetryCommand`, `CancelCommand`, `ProvideManuallyCommand`. On
     activation it `Task.Run`s `EnsureAsync` with a captured
     `IProgress<ProvisioningProgress>` adapter that marshals to the UI
     thread via Avalonia's `Dispatcher`.
   - `ProvisioningToolsView.axaml` — progress bar, stage label, the three
     buttons, error text. Bindings only.

6. **Tests — unit (xUnit + FluentAssertions, no network, no disk besides
   temp dirs).**
   - `Sha256ChecksumVerifierTests` — known-vector hash; mismatch returns
     false; missing file throws.
   - `FfmpegProvisionerTests` — every algorithm branch via in-memory
     fakes: AlreadyPresent (file exists + checksum matches), Downloaded
     (happy path), ChecksumMismatch (downloader returns wrong bytes →
     `InvalidDataException` + `.partial` deleted), MissingRid
     (`PlatformNotSupportedException`), Cancellation (token signalled
     mid-download).
   - `ProvisioningToolsViewModelTests` — Retry resets state and re-invokes
     `EnsureAsync`; success raises `ProvisioningCompleted` so the shell
     can transition.

7. **Tests — integration (`Xunit.SkippableFact`).**
   - `HttpToolDownloaderTests` — in-process Kestrel serves a 1 KB blob;
     asserts progress monotonicity and final file equality. No external
     network.
   - `ZipArchiveExtractorTests` / `TarXzArchiveExtractorTests` — round-trip
     a synthetic archive containing a fake `ffprobe`; asserts traversal
     protection rejects an `..` entry.
   - `FfmpegProvisionerLiveTests` — `Skip.IfNot(env VI_LIVE_DOWNLOAD == 1)`;
     downloads each manifest entry's URL, verifies SHA-256, asserts the
     extracted `ffprobe` exits 0 on `-version`. **Run before merge to
     refresh the bundled hashes.**

8. **Documentation.**
   - Add `docs/projects/rebuild/milestones/m2-5-ffmpeg-provisioning.md`
     describing the manifest format, how to refresh pins, and the manual
     smoke test.
   - Update [README.md](../../../../README.md): replace the M3 plan's
     "install ffprobe yourself" prerequisite paragraph with a one-liner
     stating that the app provisions `ffprobe` automatically on first
     launch, plus a note on offline mirrors.
   - Update the M3 plan's *Dependencies* section to point at this
     milestone instead of asking the user to install FFmpeg manually.

## Dependencies

- **Prior milestone:** M2 Database & Authentication (shell state machine and
  settings persistence are the substrate). M3 *Library & Indexing* is the
  consumer; this plan is its prerequisite.
- **External tooling:** `dotnet 8` SDK; no other prerequisites for build.
  The provisioner's runtime dependency is HTTPS access to the manifest
  URLs (BtbN, evermeet.cx) — air-gapped users override the manifest in
  `%APPDATA%/VideoIndexer/appsettings.json` to point at an internal mirror.
- **NuGet packages (additions):**
  - `SharpCompress` (LGPL) for the XZ decompression path on Linux. No
    Windows/macOS XZ usage.
  - `Microsoft.Extensions.Http` for `IHttpClientFactory`.
  - No FFmpeg-related NuGet packages (no `Xabe.FFmpeg`, no `FFMpegCore`,
    no `FFmpeg.AutoGen`).
- **Specifications that must remain stable:** the M3 plan
  [hashing pipeline](../2026-04-28-m3-library-indexing/plan.md) — its
  `IFfprobeRunner` reads the resolved path from
  `ExternalTools.Ffmpeg.FfprobePath` produced here.

## Required Components

All paths workspace-relative. **(NEW)** items do not exist; **(MODIFIED)**
items already exist.

- `src/VideoIndexer.Core/Abstractions/IExternalToolProvisioner.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/IToolDownloader.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/IArchiveExtractor.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/IChecksumVerifier.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/IRuntimeIdentifierProvider.cs` (NEW)
- `src/VideoIndexer.Core/Enums/ProvisioningOutcome.cs` (NEW)
- [`src/VideoIndexer.Core/Enums/ShellState.cs`](../../../../src/VideoIndexer.Core/Enums/ShellState.cs) (MODIFIED)
- `src/VideoIndexer.Core/Events/ProvisioningProgressChangedEventArgs.cs` (NEW)
- `src/VideoIndexer.Core/Models/ToolManifestEntry.cs` (NEW)
- `src/VideoIndexer.Core/Models/ToolPaths.cs` (NEW)
- `src/VideoIndexer.Core/Models/ProvisioningProgress.cs` (NEW)
- `src/VideoIndexer.Core/Options/FfmpegProvisioningOptions.cs` (NEW)
- `src/VideoIndexer.Core/Options/FfmpegToolOptions.cs` (NEW)
- [`src/VideoIndexer.Core/Options/ExternalToolsOptions.cs`](../../../../src/VideoIndexer.Core/Options/ExternalToolsOptions.cs) (MODIFIED)
- `src/VideoIndexer.Infrastructure/ExternalTools/HttpToolDownloader.cs` (NEW)
- `src/VideoIndexer.Infrastructure/ExternalTools/Sha256ChecksumVerifier.cs` (NEW)
- `src/VideoIndexer.Infrastructure/ExternalTools/ZipArchiveExtractor.cs` (NEW)
- `src/VideoIndexer.Infrastructure/ExternalTools/TarXzArchiveExtractor.cs` (NEW)
- `src/VideoIndexer.Infrastructure/ExternalTools/CompositeArchiveExtractor.cs` (NEW)
- `src/VideoIndexer.Infrastructure/ExternalTools/DefaultRuntimeIdentifierProvider.cs` (NEW)
- `src/VideoIndexer.Infrastructure/ExternalTools/FfmpegProvisioner.cs` (NEW)
- [`src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj`](../../../../src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj) (MODIFIED — `SharpCompress`, `Microsoft.Extensions.Http`)
- `src/VideoIndexer.App/ViewModels/ProvisioningToolsViewModel.cs` (NEW)
- `src/VideoIndexer.App/Views/ProvisioningToolsView.axaml(.cs)` (NEW)
- [`src/VideoIndexer.App/Program.cs`](../../../../src/VideoIndexer.App/Program.cs) (MODIFIED — step 5.5 + shell factory)
- [`src/VideoIndexer.App/Assets/appsettings.json`](../../../../src/VideoIndexer.App/Assets/appsettings.json) (MODIFIED — manifest)
- [`src/VideoIndexer.App/ViewModels/ShellViewModel.cs`](../../../../src/VideoIndexer.App/ViewModels/ShellViewModel.cs) (MODIFIED — initial state + provisioner dependency)
- `tests/VideoIndexer.Tests/Sha256ChecksumVerifierTests.cs` (NEW)
- `tests/VideoIndexer.Tests/FfmpegProvisionerTests.cs` (NEW)
- `tests/VideoIndexer.Tests/Fixtures/FakeToolDownloader.cs` (NEW)
- `tests/VideoIndexer.Tests/Fixtures/FakeArchiveExtractor.cs` (NEW)
- `tests/VideoIndexer.App.Tests/ProvisioningToolsViewModelTests.cs` (NEW)
- `tests/VideoIndexer.App.Tests/ShellViewModelTests.cs` (MODIFIED — assert `ProvisioningTools` is the initial state and that success advances to `Connecting`)
- `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/HttpToolDownloaderTests.cs` (NEW)
- `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/ZipArchiveExtractorTests.cs` (NEW)
- `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/TarXzArchiveExtractorTests.cs` (NEW)
- `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/FfmpegProvisionerLiveTests.cs` (NEW)
- `docs/projects/rebuild/milestones/m2-5-ffmpeg-provisioning.md` (NEW)
- [`README.md`](../../../../README.md) (MODIFIED)
- [`docs/agents/plans/2026-04-28-m3-library-indexing/plan.md`](../2026-04-28-m3-library-indexing/plan.md) (MODIFIED — *Dependencies* section now points here for `ffprobe` provenance; `LibraryOptions.FfprobePath` becomes "resolved from `ExternalTools.Ffmpeg.FfprobePath` if null")

## Assumptions

- HTTPS egress to the configured mirror URLs is acceptable on first launch.
  Air-gapped deployments can override the manifest to point at an internal
  mirror or pre-populate the `tools/ffmpeg/<version>/<rid>/` directory and
  set `AutoDownload = false`.
- The pinned FFmpeg version (`n7.1`) supports every codec the M3 hasher
  needs. (FFmpeg 7.x is available on all five RIDs at the time of writing.)
- The user has write permission to `%APPDATA%/VideoIndexer/`. (Same
  assumption M1 already makes for `appsettings.json` and logs.)
- Disk usage for the extracted FFmpeg suite (~150 MB on Windows full build,
  ~80 MB stripped) is acceptable for a video-indexing application.
- The single-user desktop trust model from M2 still holds; the SHA-256
  verification is the supply-chain control, there is no code-signing of the
  downloaded binary.

## Constraints

- Must target **.NET 8** (LTS); no preview-only APIs.
- Must build under `TreatWarningsAsErrors=true`,
  `<WarningLevel>9999</WarningLevel>`.
- Must not break any M1 or M2 acceptance criterion. The
  `Connecting → LoggingOn → Ready` flow must still work end-to-end after
  the provisioning state completes (and instantly when the binaries are
  cached).
- Must not introduce a Windows-only API (cross-platform constraint from
  `rebuild.md`). `RuntimeInformation`, `HttpClient`, `ZipFile`,
  `System.Formats.Tar`, `File.SetUnixFileMode` are all cross-platform or
  no-op on the wrong OS.
- Must verify SHA-256 of every downloaded archive before extraction.
  An unverifiable archive is **never** unpacked.
- Must reject Zip-Slip / Tar-Slip entries (any extracted path that escapes
  the target directory) with an `InvalidDataException`.
- Must not check downloaded binaries into source control.
  `tools/ffmpeg/` is added to `.gitignore` if any developer accidentally
  runs the app from inside the repo. (The default `AppPaths.Root` is
  outside the repo, so this is a defence-in-depth measure.)
- Live integration tests must self-skip when their prerequisites are
  absent (`VI_LIVE_DOWNLOAD` env var, network access).

## Out of Scope

- **In-place upgrade of FFmpeg version after release.** Version bumps
  require recomputing checksums and shipping an app update; there is no
  background updater.
- **Windows code-signing or macOS notarisation of the downloaded binary.**
  Trust comes from the SHA-256 pin, not from OS-level signature checks.
- **Provisioning of any other external tool** (VLC, MySQL client, XML
  editor). Those slots in `ExternalToolsOptions` remain user-configured for
  now.
- **Resumable / chunked downloads.** A failed download starts over.
  Acceptable for one-time first-launch provisioning of a ~80 MB asset.
- **A proxy-configuration UI.** `HttpClient` honours system proxy settings
  by default; advanced configuration is environment-variable driven.
- **Removing ffprobe support from the M3 `LibraryOptions.FfprobePath`
  override.** That override remains for power users / CI; it just becomes
  optional once provisioning is in place.

## Acceptance Criteria

- [ ] `dotnet restore && dotnet build -c Release` from a fresh clone
      produces zero warnings and zero errors across all projects.
- [ ] All M1 + M2 unit tests still pass; the modified `ShellViewModelTests`
      now also asserts `ProvisioningTools` is the initial state.
- [ ] On first launch with no `tools/ffmpeg/` cache, the app shows the
      *Provisioning external tools* view, downloads the correct archive
      for the host OS into `%APPDATA%/VideoIndexer/tools/ffmpeg/<version>/<rid>/`,
      verifies its SHA-256, extracts it, persists the resolved paths into
      `appsettings.json`, and advances the shell to `Connecting`.
- [ ] On second launch the same flow completes in well under one second
      (no download, no extraction; checksum check on the already-resolved
      ffprobe path passes), and the user transitions straight to
      `Connecting`.
- [ ] If the manifest URL returns a body whose SHA-256 differs from the
      pinned hash, provisioning fails with a clear error, the `.partial`
      directory is deleted, and the *Retry* button is enabled. **The
      target directory is left untouched.**
- [ ] If the active RID has no manifest entry (e.g., `linux-arm64` in the
      initial release), the user sees a clear "Unsupported platform"
      message and a *Provide…* button that lets them select an existing
      `ffprobe` binary manually.
- [ ] The downloaded `ffprobe` binary, invoked with `-version`, exits 0
      on every supported RID (verified by the live integration test).
- [ ] No FFmpeg binary is committed to the repository; `.gitignore`
      excludes any local `tools/ffmpeg/` artefact.
- [ ] M3 *Library & Indexing* can begin implementation immediately
      against `ExternalTools.Ffmpeg.FfprobePath` without any further
      manual setup.
- [ ] `docs/projects/rebuild/milestones/m2-5-ffmpeg-provisioning.md`
      exists, documents the manifest format and the
      "how to refresh the pinned version" runbook, and the README's
      prerequisites paragraph mentions the auto-provisioning behaviour.

## Testing Strategy

- **Unit tests** (xUnit + FluentAssertions, in-memory fakes) cover the
  provisioner state machine, checksum verifier, and view-model. No
  network, no real archives, no real FFmpeg.
- **Integration tests** that need disk + a tiny HTTP server use an
  in-process Kestrel fixture serving canned bytes; these run in CI on
  every push.
- **Live download tests** (`SkippableFact`, gated on `VI_LIVE_DOWNLOAD=1`)
  actually fetch each manifest URL, verify SHA-256, and run
  `<extracted-ffprobe> -version`. Run locally before merging any change to
  the manifest. **CI does not need network egress.**
- **Manual smoke test** documented in
  `m2-5-ffmpeg-provisioning.md`:
  1. Delete `%APPDATA%/VideoIndexer/tools/`.
  2. `dotnet run --project src/VideoIndexer.App`.
  3. Observe the provisioning view, then `Connecting`.
  4. Re-launch — provisioning is instant.
  5. Confirm `ffprobe -version` runs from the resolved path.
- **Negative manual smoke test:** flip a single hex digit in the
  bundled manifest's SHA-256, rerun, observe the failure path with
  *Retry*.

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Pinned mirror URL goes 404 between releases.** | Manifest is overridable in user's `appsettings.json`; README documents the override. CI's live test surfaces the regression early. The provisioner's error message includes the URL and instructions for manual override. |
| **SHA-256 mismatch on legitimate mirror updates** (BtbN republishes a build). | Pins are versioned (`DownloadVersion = "n7.1"` is part of the URL path). Mirror operators do not republish under the same pinned tag. The live test is the early-warning system. |
| **Zip-Slip / Tar-Slip via a malicious archive entry.** | Both extractors normalise each entry's path and reject any whose normalised target escapes the destination directory; covered by a unit test using a hand-crafted hostile archive. |
| **First-launch network egress is blocked / slow.** | The provisioning view shows progress and stage; *Cancel* exits cleanly; offline operators set `AutoDownload = false` and use the *Provide…* manual fallback. |
| **Disk usage of `tools/ffmpeg/` grows across version pins.** | Older `<version>` directories are not pruned automatically (low risk: pins change rarely). The milestone doc lists "delete `tools/ffmpeg/<old-version>/`" as a manual step on upgrade. A future WP can add automatic GC. |
| **`SharpCompress` LGPL adds a license obligation.** | Acceptable per the project's license stance; recorded in the README's third-party section. The dependency is referenced as a regular NuGet package (dynamic linking model) so LGPL compliance is satisfied by attribution. |
| **macOS Gatekeeper quarantines the downloaded binary.** | The macOS evermeet.cx archives ship a self-contained `ffprobe` that runs fine after `chmod +x`; if Gatekeeper nonetheless blocks it, the manifest doc instructs the user to run `xattr -d com.apple.quarantine`. Documented in the milestone runbook. |
| **The provisioner blocks the UI thread during the first download.** | `EnsureAsync` runs on a background `Task.Run`; progress is marshalled to the UI via Avalonia's `Dispatcher.UIThread.Post`. Cancellation is honoured at every `await`. |
| **Tests accidentally mutate the real `%APPDATA%/VideoIndexer/tools/`.** | All tests inject a temp `IAppPaths` and `IHttpClientFactory`; a code-review checklist item forbids any test from constructing a real `AppPaths()`. |

AGENT: Planning
STATUS: READY_FOR_PM
