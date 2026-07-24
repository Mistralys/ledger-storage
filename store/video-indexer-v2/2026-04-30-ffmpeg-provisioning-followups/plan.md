# Plan — FFmpeg Provisioning Follow-ups (M2.5.1)


## Summary
Close out the three release-gate items (RG-1…RG-3) and the actionable deferred
items (D-1…D-12) recorded in the M2.5 FFmpeg provisioning synthesis
(`docs/agents/plans/2026-04-28-ffmpeg-provisioning/synthesis.md`). The work
unblocks the M2.5 production release, hardens the provisioning subsystem, and
removes carried-forward tech debt before M3 Library & Indexing depends on it
in earnest. No new user-visible behaviour is introduced; the shell still starts
in `ProvisioningTools` and transitions to `Connecting` on success.


## Architectural Context
The FFmpeg provisioning subsystem (M2.5) lives across three projects in
`src/VideoIndexer.sln`:

- **`src/VideoIndexer.Core`** — contracts only. `IExternalToolProvisioner`,
  `IToolDownloader`, `IArchiveExtractor`, `IChecksumVerifier`,
  `IRuntimeIdentifierProvider`, plus `ToolPaths`, `ProvisioningProgress`,
  `ToolManifestEntry`, `FfmpegToolOptions`, `FfmpegProvisioningOptions`,
  `ExternalToolsOptions`, and `ShellState`.
- **`src/VideoIndexer.Infrastructure/ExternalTools/`** — implementations:
  `FfmpegProvisioner.cs`, `HttpToolDownloader.cs`, `Sha256ChecksumVerifier.cs`,
  `ZipArchiveExtractor.cs`, `TarXzArchiveExtractor.cs`,
  `CompositeArchiveExtractor.cs`, `DefaultRuntimeIdentifierProvider.cs`.
- **`src/VideoIndexer.App`** — UI + DI: `ViewModels/ShellViewModel.cs`,
  `ViewModels/ProvisioningToolsViewModel.cs`, `Views/ProvisioningToolsView.axaml`,
  `Program.cs` (the labelled `// 5.5 — External tools provisioning` block),
  and `Assets/appsettings.json` (manifest with 5-RID `Provisioning.Manifest`).

Settings persistence runs through `ISettingsService.SaveAsync(AppOptions, …)`
implemented by `Settings/JsonSettingsService.cs`. There is **no** `UpdateAsync`
method on the interface (D-10 is a doc-only naming inconsistency).

Tests live in three assemblies under `tests/`:
- `VideoIndexer.Tests` — Core + Infrastructure unit tests, targets `net10.0`
  (inherited from `Directory.Build.props`). Contains `FfmpegProvisionerTests.cs`
  and `FfmpegManifestOptionsTests.cs`.
- `VideoIndexer.Infrastructure.Tests` — Infrastructure + live-download gate, targets `net10.0`.
- `VideoIndexer.App.Tests` — App view-models. A shared `FakeExternalToolProvisioner`
  lives in `TestHelpers/FakeExternalToolProvisioner.cs` and is used by six files:
  `MainWindowViewModelTests.cs`, `ShellViewModelTests.cs`, `LogonViewModelTests.cs`,
  `PasswordSetupViewModelTests.cs`, `DatabaseConnectorViewModelTests.cs`, and
  `ViewLocatorTests.cs`. (D-4 — already consolidated as part of WP-010 delivery;
  the synthesis count of "four copies" was later corrected to "five copies" during
  M2.5 planning, but consolidation was completed before this follow-up plan.)

All three assemblies target `net10.0` via `Directory.Build.props` with no
per-project overrides. The historical `net8.0`/`net10.0` split referenced by
D-6 no longer exists.


## Approach / Architecture
Work is organised into seven thematic work packages. None introduces a new
public contract; all changes are internal refinements, test additions, asset
relocations, or configuration values.

1. **WP-A — Verify-and-close already-fixed items** (D-1, D-2, D-7, D-8). Synthesis
   was authored mid-cycle; the current source already addresses these — including
   D-7, whose `Manifest` initialiser is **already** `new Dictionary<string,
   ToolManifestEntry>()` in `FfmpegProvisioningOptions.cs` (line 42). Add
   regression tests where coverage is genuinely missing, then strike the items
   from the deferred list.
2. **WP-B — Manifest pinning & live-download enablement** (RG-1, RG-2, RG-3).
   Document a reproducible "refresh-pins" procedure, fill the five SHA-256
   values in `appsettings.json`, verify the win-arm64 BtbN URL exists, and
   confirm the evermeet.cx archives contain `ffprobe`. Run the live-download
   integration test with `VI_LIVE_DOWNLOAD=1` to prove the end-to-end path.
3. **WP-C — Auditable HTTP downloader** (D-12). Inject
   `ILogger<HttpToolDownloader>`; emit structured log events for download
   start, completion (URL, byte total, elapsed), and failure (status code,
   exception type) — matching the logging style already used in
   `FfmpegProvisioner`.
4. **WP-D — Shared test fixtures** (D-11; D-4 already done). Move the inline
   tar fixture byte arrays in `TarXzArchiveExtractorTests.cs` into binary files
   under `tests/VideoIndexer.Infrastructure.Tests/Fixtures/archives/` and load
   them via `EmbeddedResource` or test-content copy. (`FakeExternalToolProvisioner`
   consolidation was completed before this plan — D-4 is a no-op here.)
5. **WP-E — DI hygiene** (D-5 only). Apply `[ActivatorUtilitiesConstructor]`
   to the **parameterless** constructor of `CompositeArchiveExtractor` (the
   2-parameter ctor cannot be resolved by DI because `IArchiveExtractor` is
   the service being registered — wiring DI to it would create a
   self-dependency cycle). Replace the factory lambda registration in
   `Program.cs` with the standard
   `AddSingleton<IArchiveExtractor, CompositeArchiveExtractor>()`.
   **D-3 is deferred** out of this follow-up — see WP-H below for rationale.
6. **WP-F — Coverage** (D-9). Add the missing
   `Provisioning.Enabled = false` short-circuit test to
   `FfmpegProvisionerTests`. (D-7 was found already-fixed during planning and
   moved to WP-A.)
7. **WP-G — Documentation reconciliation** (D-10, D-6 close-out).
   Sweep **all** matches of `ISettingsService.UpdateAsync` under
   `docs/agents/plans/2026-04-28-ffmpeg-provisioning/` (plan.md,
   work-packages-draft.md, work/WP-007.md) and the M2.5 milestone doc, and
   replace with `SaveAsync`. Close D-6 in the synthesis (no TFM split remains;
   both assemblies already target `net10.0`). Update the synthesis and milestone
   Known Deferred Items tables with accurate close-out annotations.

8. **WP-H — D-3 deferred to post-`ShellView`-wiring milestone**. The
   synthesis flagged `ProvisioningToolsViewModel.RunProvisioningAsync` as a
   "redundant" caller of `EnsureAsync`. Verification against the current
   source shows this is incorrect: `ShellViewModel.ProvisionAsync` passes
   `null` for `IProgress<ProvisioningProgress>` and is **not yet invoked
   from anywhere** (per README §M2.5 — `ShellView` wiring is deferred). The
   view-model is therefore the *only* current caller, not a duplicate.
   Removing it would also delete the `_cts` field that backs `CancelCommand`
   and `RetryCommand`, regressing the AC-2 tests. D-3 is deferred to the
   milestone that wires `ShellView` and adds shell-side progress
   forwarding/cancel/retry plumbing; this follow-up records it as
   "Deferred — premise invalid in current code" rather than closing it.


## Rationale
- **Verify-before-redo (WP-A).** The synthesis snapshot lags reality on three
  items. Re-implementing them would be wasteful and risks regression; a
  targeted verification + missing-test pass is enough.
- **Release-gate items first in their own WP (WP-B).** RG-1/2/3 are the only
  items blocking a production cut. They depend on network access and an
  external mirror state that may have shifted (BtbN may have re-tagged; the
  win-arm64 asset name is unverified). Isolating them keeps risk contained.
- **D-3 deferred (WP-H).** Audit found the synthesis premise invalid: the
  shell does not yet forward an `IProgress<>` and is not yet invoked by any
  view, so the VM is the only caller. Reworking it now would delete the
  cancel/retry plumbing that the AC-2 tests depend on. Reopen when
  `ShellView` is wired.
- **Logger in `HttpToolDownloader` (WP-C).** Matches the convention already
  established by `FfmpegProvisioner` (`ILogger<FfmpegProvisioner>` injected,
  structured logs around each external operation). Closes the only Low-severity
  audit-logging gap that the security audit explicitly flagged as deferred.
- **Fixture consolidation (WP-D).** Five copies of `FakeExternalToolProvisioner`
  is a clear DRY violation that will multiply as new view-model tests appear
  in M3. Doing this now, before M3 starts adding more tests, is cheaper.
- **TFM split deferred (WP-G).** Unifying the TFMs requires touching MSBuild
  central props and may cascade into framework reference changes. Until shared
  test helpers actually need to be extracted across the boundary, the cost is
  not yet justified — we document the rationale instead.


## Detailed Steps

### WP-A — Verify-and-close already-fixed deferred items
0. **D-7 verification.** Re-read
   `src/VideoIndexer.Core/Options/FfmpegProvisioningOptions.cs` line ~42.
   Confirm `Manifest` initialiser is already
   `new Dictionary<string, ToolManifestEntry>()` (verified during planning).
   No code change. Mark D-7 closed.
1. **D-1 verification.** Re-read
   `src/VideoIndexer.Infrastructure/ExternalTools/FfmpegProvisioner.cs` lines
   ~232–260. Confirm `IsCacheHitAsync` is `private static Task<bool>` with
   `Task.FromResult(...)` returns and **no** `GetAwaiter().GetResult()` calls.
   No code change. Mark D-1 closed in the follow-up tracker.
2. **D-2 verification.** Same file: confirm `IsCacheHitAsync` checks both
   `options.FfprobePath` and `options.FfmpegPath` for null/empty + `File.Exists`.
   No code change. Mark D-2 closed.
3. **D-8 verification.** In
   `src/VideoIndexer.Infrastructure/ExternalTools/TarXzArchiveExtractor.cs`,
   confirm the `SymbolicLink`/`HardLink` rejection branch (`throw new
   InvalidDataException(...)`) is present. No code change. Mark D-8 closed.
4. **D-9 (test gap).** Add a unit test
   `EnsureAsync_ProvisioningDisabled_ShortCircuitsAndReturnsCachedPaths` to
   `tests/VideoIndexer.Tests/FfmpegProvisionerTests.cs`
   (the existing test class in the `VideoIndexer.Tests` project, which references
   both Core and Infrastructure and already contains all other `FfmpegProvisioner`
   unit tests). The test sets `Provisioning.Enabled = false`, populates
   `FfprobePath`/`FfmpegPath`/`ResolvedVersion` in the fake `ISettingsService`,
   asserts no downloader/extractor methods are invoked, and asserts the returned
   `ToolPaths` mirror the persisted values.
5. ~~**TarXz symlink coverage (test for D-8).**~~ **Already done — no-op.**
   `ExtractAsync_SymlinkEntry_ThrowsInvalidDataException` already exists at line 186
   of `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/TarXzArchiveExtractorTests.cs`,
   backed by the inline `SymlinkTarXzBytes` constant. Confirm it passes in the
   baseline `dotnet test` run and mark D-8 closed (verified: implementation +
   test both present).

### WP-B — Manifest pinning & live-download enablement
1. ~~**Refresh-pins procedure.**~~ **Already done — no-op.** The numbered runbook
   is already present in `docs/projects/rebuild/milestones/m2-5-ffmpeg-provisioning.md`
   under "How to Refresh Manifest Pins" (steps 1–7, with PowerShell and shell
   commands) and under "Manual Smoke Test". No doc change needed.
2. **RG-2 — verify win-arm64 URL.** HEAD request the documented URL
   `https://github.com/BtbN/FFmpeg-Builds/releases/download/n7.1/ffmpeg-n7.1-winarm64-gpl.zip`.
   If it 404s, search the BtbN release page for the actual asset name and
   update `Manifest["win-arm64"].Url` plus the two `…RelativePath` fields.
3. **RG-3 — verify macOS archive contents.** Download the two evermeet.cx
   archives once and inspect the file list. If either does not contain
   `ffprobe`, add a separate `ffprobe` URL/SHA pair to the manifest (requires
   extending `ToolManifestEntry` with an optional `FfprobeArchive` sub-record;
   apply only if needed). Document the outcome in the milestone doc.
4. **RG-1 — fill SHA-256 hashes.** Replace each `"TODO_FILL_AT_IMPLEMENTATION"`
   in two locations:
   - `src/VideoIndexer.App/Assets/appsettings.json` (5 occurrences).
   - `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/FfmpegProvisionerLiveTests.cs`
     (5 occurrences inside the in-test manifest builder, lines ~116, 124,
     132, 140, 148). Either inline the real hashes or, preferably, refactor
     the live test to load the manifest from the production
     `appsettings.json` so there is a single source of truth.
5. **Update existing smoke-test gate.** `FfmpegManifestOptionsTests` already
   exists at `tests/VideoIndexer.Tests/FfmpegManifestOptionsTests.cs` with four
   tests. The test `AppsettingsJson_Manifest_Sha256ValuesAreAllPlaceholders`
   currently asserts that every `Sha256` equals `"TODO_FILL_AT_IMPLEMENTATION"`.
   After step 4 fills real hashes, **update this test** to instead assert that
   every `Manifest[*].Sha256` matches the regex `^[0-9a-f]{64}$` (rename to
   `AppsettingsJson_Manifest_Sha256ValuesAreHexStrings`). Do **not** create a
   second class in a different assembly. This step depends on step 4 (RG-1)
   completing first.
6. **Run the live-download test.** Set `$env:VI_LIVE_DOWNLOAD = "1"` and run
   `dotnet test tests/VideoIndexer.Infrastructure.Tests --filter
   FullyQualifiedName~FfmpegProvisionerLiveTests`. Confirm `ffprobe -version`
   exits with code 0 against the freshly downloaded binary.

### WP-C — Auditable HTTP downloader (D-12)
1. Add `ILogger<HttpToolDownloader>` constructor parameter to
   `src/VideoIndexer.Infrastructure/ExternalTools/HttpToolDownloader.cs`;
   guard with `ArgumentNullException` per existing convention.
2. Emit `LogInformation` on download start (URL, destination, content-length
   when known) and on success (URL, bytes received, elapsed milliseconds).
3. Emit `LogError` in the `catch` block before re-throwing (URL, exception
   type, partial bytes received).
4. Update existing `HttpToolDownloaderTests.cs` to pass
   `NullLogger<HttpToolDownloader>.Instance` (or a capturing
   `ListLogger<T>` if one already exists in the test project).
5. No DI change is needed — `Microsoft.Extensions.Logging` is already wired in
   `Program.cs` for the rest of Infrastructure.

### WP-D — Shared test fixtures (D-4, D-11)
1. ~~**Consolidate `FakeExternalToolProvisioner`.**~~ **Already done — no-op.**
   The shared fake already exists at
   `tests/VideoIndexer.App.Tests/TestHelpers/FakeExternalToolProvisioner.cs`
   (note: the correct folder is `TestHelpers/`, not `Fixtures/`). All six
   consumer files (`MainWindowViewModelTests.cs`, `ShellViewModelTests.cs`,
   `LogonViewModelTests.cs`, `PasswordSetupViewModelTests.cs`,
   `DatabaseConnectorViewModelTests.cs`, `ViewLocatorTests.cs`) already use the
   shared type with no private nested copies. No code change needed.
2. **Externalise tar fixtures.**
   - Create `tests/VideoIndexer.Infrastructure.Tests/Fixtures/archives/`
     (`Fixtures/` already exists containing `TempDirectory.cs`; create the
     `archives/` subdirectory within it).
   - For each inline `byte[]` fixture in `TarXzArchiveExtractorTests.cs`,
     write a `.tar.xz` file under `Fixtures/archives/`. There are **five**
     fixtures matching the existing constants:
     - `FlatTarXzBytes`    → `flat.tar.xz`
     - `NestedTarXzBytes`  → `nested.tar.xz`
     - `SlipTarXzBytes`    → `slip1.tar.xz`
     - `Slip2TarXzBytes`   → `slip2.tar.xz`
     - `SymlinkTarXzBytes` → `symlink.tar.xz`
   - Mark the files `<None CopyToOutputDirectory="PreserveNewest" />` in the
     `.csproj`.
   - Replace the inline byte arrays in the tests with `File.ReadAllBytes` (or
     direct path use in `ExtractAsync`).
   - **Out of scope:** the three behaviour-specialised provisioner fakes in
     `ProvisioningToolsViewModelTests.cs` (`ProgressReportingProvisioner`,
     `FailingProvisioner`, `BlockingProvisioner`) have distinct semantics and
     remain local to that test file — do **not** fold them into the shared
     `FakeExternalToolProvisioner`.

### WP-E — DI hygiene (D-5)
1. **D-5.** Add `[ActivatorUtilitiesConstructor]` to the **parameterless**
   constructor of
   `src/VideoIndexer.Infrastructure/ExternalTools/CompositeArchiveExtractor.cs`.
   The 2-parameter ctor takes two `IArchiveExtractor` arguments — the same
   service being registered — and DI cannot resolve a self-dependency. The
   parameterless ctor delegates to `new ZipArchiveExtractor()` and
   `new TarXzArchiveExtractor()`, which is the intended runtime composition.
2. In `src/VideoIndexer.App/Program.cs` (the `// 5.5 — External tools
   provisioning` block), replace
   `AddSingleton<IArchiveExtractor>(_ => new CompositeArchiveExtractor())`
   with
   `services.AddSingleton<IArchiveExtractor, CompositeArchiveExtractor>();`.
3. Verify by running `dotnet test` — the existing
   `Program`/`Provision`-touching tests must continue to pass.

### WP-H — D-3 deferral note (no code change)
1. Append to the M2.5 milestone doc and the synthesis deferred-items table:
   `D-3: Deferred — synthesis premise invalid in current code (ShellViewModel
   does not forward IProgress<>; ProvisioningToolsViewModel is the sole
   caller). Reopen after ShellView wiring lands.`

### WP-F — Coverage (D-9)
1. **D-9.** Test added in WP-A step 4. No-op here.
2. **D-7.** Verified already-fixed in WP-A step 0. No code change.

### WP-G — Documentation reconciliation (D-10, D-6 note)
1. **D-10.** Search the M2.5 plan folder
   (`docs/agents/plans/2026-04-28-ffmpeg-provisioning/`) and the milestone doc
   `docs/projects/rebuild/milestones/m2-5-ffmpeg-provisioning.md` for any
   reference to `ISettingsService.UpdateAsync` and replace with `SaveAsync`.
   Known matches at planning time:
   - `docs/agents/plans/2026-04-28-ffmpeg-provisioning/plan.md` line ~259
   - `docs/agents/plans/2026-04-28-ffmpeg-provisioning/work-packages-draft.md` line ~178
   - `docs/agents/plans/2026-04-28-ffmpeg-provisioning/work/WP-007.md` lines ~25, ~32
   - `docs/agents/plans/2026-04-28-ffmpeg-provisioning/synthesis.md` line ~204 (descriptive — leave or annotate)
2. ~~**D-6 note.**~~ **D-6 closed — no action.** Both test assemblies target
   `net10.0` inherited from `Directory.Build.props` with no per-project overrides;
   the historical `net8.0`/`net10.0` split has been resolved. Annotate the
   synthesis D-6 row as "Closed — both assemblies now target `net10.0`; no
   documentation needed." No milestone-doc subsection is required.
3. Update `docs/agents/plans/2026-04-28-ffmpeg-provisioning/synthesis.md`
   §5 deferred-items table:
   - Strike RG-1/2/3 and D-1/D-2/D-5/D-7/D-8/D-9/D-10/D-11/D-12 with a
     "Closed in 2026-04-30 follow-up" note.
   - Mark D-3 as "Deferred — synthesis premise invalid in current code;
     reopen after `ShellView` wiring".
   - Mark D-4 as "Closed before follow-up plan was written — shared fake
     in `tests/VideoIndexer.App.Tests/TestHelpers/FakeExternalToolProvisioner.cs`;
     six consumer files (not four as originally counted)."
   - Mark D-6 as "Closed — both assemblies target `net10.0`; no split to
     document."
4. Update `docs/projects/rebuild/milestones/m2-5-ffmpeg-provisioning.md`
   Known Deferred Items table:
   - "Consolidated `FakeExternalToolProvisioner`" row → "Closed — shared fake
     in `TestHelpers/FakeExternalToolProvisioner.cs`; six consumer files."
   - "Logging in provisioning pipeline" row → update to distinguish:
     `HttpToolDownloader` logging is closed by WP-C (this follow-up); logging
     for `TarXzArchiveExtractor` and other pipeline classes remains deferred.


## Dependencies
- WP-B (RG-1) requires network access to BtbN GitHub Releases and evermeet.cx.
- WP-B step 6 (live-download test) requires WP-B steps 1–4 to complete first.
- WP-B step 5 (updating `FfmpegManifestOptionsTests`) requires WP-B step 4
  (production `appsettings.json` filled) before the updated regex assertion
  can pass.
- WP-D step 2 (externalised tar fixtures) is a prerequisite for the
  `TarXzArchiveExtractorTests.cs` fixture migration; WP-A step 5 is now a
  no-op verification (symlink test already exists).
- WP-G depends on every other WP being complete (it strikes the items from the
  tracker).
- WP-H is a documentation-only deferral note and has no code dependencies.


## Required Components

### Files to modify (existing)
- `src/VideoIndexer.App/Assets/appsettings.json` — RG-1 hash values.
- `src/VideoIndexer.Infrastructure/ExternalTools/HttpToolDownloader.cs` — D-12 logger.
- `src/VideoIndexer.Infrastructure/ExternalTools/CompositeArchiveExtractor.cs` — D-5 attribute on parameterless ctor.
- `src/VideoIndexer.App/Program.cs` — D-5 DI simplification.
- `tests/VideoIndexer.Tests/FfmpegProvisionerTests.cs` — D-9 short-circuit test.
- `tests/VideoIndexer.Tests/FfmpegManifestOptionsTests.cs` — replace `Sha256ValuesAreAllPlaceholders` test with `^[0-9a-f]{64}$` regex assertion (after RG-1).
- `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/TarXzArchiveExtractorTests.cs` — externalised fixtures (inline byte arrays → file reads); symlink test is pre-existing.
- `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/HttpToolDownloaderTests.cs` — null-logger plumbing.
- `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/FfmpegProvisionerLiveTests.cs` — replace 5 inline `TODO_FILL_AT_IMPLEMENTATION` literals (lines ~116/124/132/140/148) or refactor to load `appsettings.json`.
- `tests/VideoIndexer.Infrastructure.Tests/VideoIndexer.Infrastructure.Tests.csproj` — `<None>` items for fixture archives.
- `docs/projects/rebuild/milestones/m2-5-ffmpeg-provisioning.md` — D-10 `SaveAsync` fix, D-3 deferral note, Known Deferred Items table updates (D-4 closed, logging row clarified).
- `docs/agents/plans/2026-04-28-ffmpeg-provisioning/plan.md` — D-10 `UpdateAsync → SaveAsync` (line ~259).
- `docs/agents/plans/2026-04-28-ffmpeg-provisioning/work-packages-draft.md` — D-10 (line ~178).
- `docs/agents/plans/2026-04-28-ffmpeg-provisioning/work/WP-007.md` — D-10 (lines ~25, ~32).
- `docs/agents/plans/2026-04-28-ffmpeg-provisioning/synthesis.md` — close-out annotations including D-4 and D-6.

**Files no longer modified** (removed during audit):
- `src/VideoIndexer.Core/Options/FfmpegProvisioningOptions.cs` — D-7 already correct.
- `src/VideoIndexer.App/ViewModels/ProvisioningToolsViewModel.cs` — D-3 deferred (WP-H).
- `src/VideoIndexer.App/ViewModels/ShellViewModel.cs` — D-3 deferred (WP-H).
- `tests/VideoIndexer.App.Tests/ProvisioningToolsViewModelTests.cs` — D-3 deferred (WP-H).
- `tests/VideoIndexer.App.Tests/MainWindowViewModelTests.cs` — D-4 already consolidated.
- `tests/VideoIndexer.App.Tests/ShellViewModelTests.cs` — D-4 already consolidated.
- `tests/VideoIndexer.App.Tests/LogonViewModelTests.cs` — D-4 already consolidated.
- `tests/VideoIndexer.App.Tests/PasswordSetupViewModelTests.cs` — D-4 already consolidated.
- `tests/VideoIndexer.App.Tests/DatabaseConnectorViewModelTests.cs` — D-4 already consolidated.

### Files to create (new)
- `tests/VideoIndexer.Infrastructure.Tests/Fixtures/archives/flat.tar.xz`.
- `tests/VideoIndexer.Infrastructure.Tests/Fixtures/archives/nested.tar.xz`.
- `tests/VideoIndexer.Infrastructure.Tests/Fixtures/archives/slip1.tar.xz`.
- `tests/VideoIndexer.Infrastructure.Tests/Fixtures/archives/slip2.tar.xz`.
- `tests/VideoIndexer.Infrastructure.Tests/Fixtures/archives/symlink.tar.xz`.

### External resources
- BtbN FFmpeg-Builds GitHub release page (n7.1 tag) — for win-x64, win-arm64, linux-x64 hashes and URL verification.
- evermeet.cx FFmpeg downloads — for osx-x64 and osx-arm64 hashes and contents inspection.


## Assumptions
- The five manifest URLs documented in the M2.5 plan are still resolvable;
  only RG-2 (win-arm64) has explicit doubt attached.
- The evermeet.cx archives contain `ffprobe` alongside `ffmpeg`; if they do
  not, RG-3 will require a `ToolManifestEntry` extension (scope creep flagged
  in WP-B step 3).
- `[ActivatorUtilitiesConstructor]` is available — i.e.
  `Microsoft.Extensions.DependencyInjection.Abstractions` is referenced
  transitively in the Infrastructure project. (Confirmed: it is, via
  `Microsoft.Extensions.Http`.)


## Constraints
- TreatWarningsAsErrors=true and WarningLevel=9999 must remain green at every
  step.
- No new package references except where strictly required (none expected).
- No changes to public Core contracts; D-7 was verified already correct so
  no Core changes are needed.
- No Git write operations (commit, push, branch creation) are part of this
  plan — the user owns those.


## Out of Scope
- D-6 TFM unification — closed; both assemblies already target `net10.0`.
- Re-evaluating the seven security findings marked "Accepted" in the synthesis
  (SSRF scheme validation, path canonicalisation, constant-time hash compare,
  predictable `.tmp` filename, zip-bomb size cap, case-sensitive Zip-Slip on
  Linux). These remain accepted risks for the M2.5 threat model.
- Adding an automated "refresh-pins" PowerShell script (the runbook is manual).
- Any M3 Library & Indexing work.


## Acceptance Criteria
- All five `Sha256` values in `Assets/appsettings.json` are lowercase 64-char
  hex strings; the updated `AppsettingsJson_Manifest_Sha256ValuesAreHexStrings`
  test in `tests/VideoIndexer.Tests/FfmpegManifestOptionsTests.cs` enforces
  the `^[0-9a-f]{64}$` regex.
- All five `TODO_FILL_AT_IMPLEMENTATION` literals in
  `FfmpegProvisionerLiveTests.cs` are replaced (with real hashes or by
  refactoring to load `appsettings.json`).
- `FfmpegProvisionerLiveTests` passes when `VI_LIVE_DOWNLOAD=1` is set on a
  Windows x64 host.
- `FfmpegProvisionerTests` includes a passing test for the
  `Provisioning.Enabled = false` short-circuit.
- `TarXzArchiveExtractorTests` already contains a passing test
  (`ExtractAsync_SymlinkEntry_ThrowsInvalidDataException`) that asserts a tar
  archive containing a symbolic-link entry is rejected with
  `InvalidDataException` — pre-existing; confirm in baseline run.
- Exactly one definition of `FakeExternalToolProvisioner` exists in
  `tests/VideoIndexer.App.Tests/TestHelpers/`; six general-purpose test files
  consume it (`MainWindowViewModelTests.cs`, `ShellViewModelTests.cs`,
  `LogonViewModelTests.cs`, `PasswordSetupViewModelTests.cs`,
  `DatabaseConnectorViewModelTests.cs`, `ViewLocatorTests.cs`). The three
  behaviour-specialised provisioner fakes in `ProvisioningToolsViewModelTests.cs`
  remain local (intentional). This criterion is already satisfied — confirm
  no regressions.
- The five `Fixtures/archives/*.tar.xz` files are present and used in lieu
  of inline byte arrays.
- `HttpToolDownloader` constructor takes `ILogger<HttpToolDownloader>`;
  download start/completion/failure produce structured log events.
- `Program.cs` registers `IArchiveExtractor` via the standard
  `AddSingleton<TService, TImpl>()` overload (no factory lambda); the
  parameterless constructor of `CompositeArchiveExtractor` carries
  `[ActivatorUtilitiesConstructor]`.
- `m2-5-ffmpeg-provisioning.md` has the D-3 deferral note appended, uses
  `SaveAsync` consistently, and has the Known Deferred Items table updated
  (D-4 and logging-in-pipeline rows closed/clarified).
- The synthesis deferred-items table is annotated with closure notes for
  every item except D-3 (Deferred) and D-6 (Documented as deliberate).
- `dotnet build` reports 0 warnings; `dotnet test` reports 0 failures with
  test count ≥ baseline + 1 (the D-9 short-circuit test is the sole net-new
  test; the symlink test and `FfmpegManifestOptionsTests` are pre-existing and
  count toward the baseline). Re-baseline via `dotnet test --list-tests` before
  WP-A to record the exact starting count in the milestone doc.


## Testing Strategy
- **Unit:** New `FfmpegProvisionerTests` short-circuit test in
  `tests/VideoIndexer.Tests/FfmpegProvisionerTests.cs` (D-9); symlink-rejection
  test in `TarXzArchiveExtractorTests.cs` is pre-existing (D-8 — confirm passes).
  Existing `HttpToolDownloaderTests` continues to pass with the new logger
  dependency.
- **Integration:** `FfmpegProvisionerLiveTests` is run once with
  `VI_LIVE_DOWNLOAD=1` after RG-1 to prove end-to-end correctness against
  the real mirrors. Result is recorded in the milestone doc; the gate stays
  skip-by-default in CI.
- **Smoke:** Updated `FfmpegManifestOptionsTests` (existing, in
  `tests/VideoIndexer.Tests/`) asserts every `Manifest[*].Sha256` matches
  `^[0-9a-f]{64}$` after RG-1 fills real hashes, guarding against regressions
  to placeholder values.
- **App-level:** `ShellViewModelTests` and `ProvisioningToolsViewModelTests`
  are unchanged in behaviour; the shared `FakeExternalToolProvisioner` in
  `TestHelpers/` is pre-existing (WP-D D-4 already done).


## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **BtbN re-tagged or removed the n7.1 win-arm64 asset** (RG-2). | WP-B step 2 verifies the URL before hashing; if missing, search the BtbN release page for the current asset name and update both URL and `…RelativePath` fields. Surface as a blocker if the entire `n7.1` line has been removed (re-pin to a newer tag is then required). |
| **evermeet.cx archives ship `ffmpeg` only, not `ffprobe`** (RG-3). | WP-B step 3 verifies contents. If confirmed missing, scope the `ToolManifestEntry` extension (optional `FfprobeArchive` sub-record) as in-WP work; otherwise no schema change. |
| **Live-download test is flaky on first run after pin refresh.** | The skip gate is opt-in; CI is unaffected. A single retry is acceptable; persistent failure means a hash typo — re-derive via `Get-FileHash` and compare. |
| **`[ActivatorUtilitiesConstructor]` applied to wrong constructor.** | The 2-parameter ctor of `CompositeArchiveExtractor` takes two `IArchiveExtractor` arguments — the same service being registered, which DI cannot resolve (self-dependency cycle). WP-E step 1 explicitly attributes the **parameterless** ctor. Verify by running `dotnet test` after the change; any DI resolution failure indicates the wrong ctor was chosen. |
| **`[ActivatorUtilitiesConstructor]` package not present.** | Verified during planning that `Microsoft.Extensions.DependencyInjection.Abstractions` is transitively available via `Microsoft.Extensions.Http`. If a future trim of dependencies removes it, fall back to keeping the factory lambda. |
| **D-1/D-2/D-8 close-outs miss a regression that was actually re-introduced after the synthesis snapshot.** | WP-A step 1–3 are explicit re-reads of the current source, not a documentary tick-box. Any discrepancy is escalated as new in-WP fix work before close-out. |
| **Inverted hash-placeholder gate accidentally allows lowercase-but-wrong hashes.** | The regex `^[0-9a-f]{64}$` only catches structural validity; the live-download test (WP-B step 6) is what catches a wrong hash. Both gates are required. |
