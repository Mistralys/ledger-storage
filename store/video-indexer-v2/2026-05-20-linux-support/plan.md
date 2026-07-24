# Plan: Linux (Fedora Workstation) Support

## Plan Audit Cycles
- Audits: 2 — Plan Auditor v1.3.1
- Architectural Reviews: 1 — Plan Architect Reviewer v1.4.0


## Summary

The rebuild specification targets "Windows + Linux + MacOS" (see [rebuild.md](../../../projects/rebuild/rebuild.md)), but this intent was never translated into a milestone or acceptance criterion. M1 codified `OutputType=WinExe` and described the project as "Windows desktop", and a handful of known cross-platform correctness issues were deferred with README TODO notes. This plan closes the gap: it hardens the two archive extractors for Linux case-sensitive filesystem semantics, adds Linux shell-open support to `IFileLauncherService`, changes `OutputType` from `WinExe` to `Exe` for cross-platform clarity, and updates all documentation and acceptance criteria to reflect Linux (`linux-x64` / Fedora Workstation) as a first-class supported platform alongside Windows.


## Architectural Context

The solution is layered: `VideoIndexer.Core` (domain) → `VideoIndexer.Infrastructure` (services) → `VideoIndexer.App` (Avalonia UI). Cross-platform posture differs by layer:

- **Core** — pure IL; no OS-specific code; already cross-platform.
- **Infrastructure** — partially cross-platform:  
  - `DefaultRuntimeIdentifierProvider` already recognises `linux-x64`.  
  - `appsettings.json` already contains a `linux-x64` FFmpeg manifest entry.  
  - `FfmpegProvisioner` already sets the executable bit on non-Windows (`File.SetUnixFileMode`).  
  - `MysqlDumpBackupService` already branches on `OperatingSystem.IsWindows()` for the binary name.  
  - `AppPaths` uses `Environment.SpecialFolder.ApplicationData` which maps to `~/.config` on Linux — correct.  
  - `ZipArchiveExtractor` ([ExternalTools/ZipArchiveExtractor.cs](../../../../src/VideoIndexer.Infrastructure/ExternalTools/ZipArchiveExtractor.cs)) and `TarXzArchiveExtractor` ([ExternalTools/TarXzArchiveExtractor.cs](../../../../src/VideoIndexer.Infrastructure/ExternalTools/TarXzArchiveExtractor.cs)) use `StringComparison.OrdinalIgnoreCase` for path-traversal guards — incorrect on Linux case-sensitive filesystems.  
  - `TarXzArchiveExtractor` does not reject device-node entry types (`BlockDevice`, `CharacterDevice`, `Fifo`) — a known deferred security gap documented in `README.md` §"TAR device-node entry types".
- **App** — one Windows-specific element remains:  
  - `WindowsFileLauncherService` ([Services/WindowsFileLauncherService.cs](../../../../src/VideoIndexer.App/Services/WindowsFileLauncherService.cs)) delegates to `explorer.exe` unconditionally; registered as `AddSingleton<IFileLauncherService, WindowsFileLauncherService>` in `Program.cs`.  
  - `VideoIndexer.App.csproj` declares `<OutputType>WinExe</OutputType>`, which the M1 milestone accepted as an acceptance criterion and which all project documentation echoes as a "Windows desktop" constraint.


## Approach / Architecture

Four targeted changes — no new abstractions, no new layers:

1. **`OutputType`: `WinExe` → `Exe`**  
   The `WinExe` PE subsystem flag only affects whether a console window is suppressed on Windows. It has no effect on Linux (the .NET AppHost on Linux is always an ELF binary). Changing to `Exe` removes the false documentation signal that the project is Windows-only and keeps the csproj honest for cross-platform builds. The cosmetic console-window flash on Windows is acceptable for a developer tool invoked from the terminal; it can be addressed later with an `app.manifest` if required.

2. **`IFileLauncherService` — Linux branch**  
   Rename `WindowsFileLauncherService` → `DesktopFileLauncherService` and add an internal `OperatingSystem.IsWindows()` branch following the established `MysqlDumpBackupService` pattern. Windows keeps `explorer.exe`. Linux uses `xdg-open <path>` (available on all major desktop Linux distros, including Fedora Workstation). For `ShowInExplorer`, Linux opens the containing directory (no universal "select file" command exists across Linux file managers). `ProcessStartInfo.ArgumentList` is used for all argument passing — no `Arguments` string concatenation.

3. **Zip-Slip / Tar-Slip guards: `OrdinalIgnoreCase` → `Ordinal`**  
   `Path.GetFullPath` on Linux produces a consistently-cased path (the filesystem is authoritative). Changing to `Ordinal` is both correct and more restrictive: it eliminates the theoretical false-positive described in `README.md` §"Zip-Slip guard" without any functional regression on Windows (where `Path.GetFullPath` normalises casing before comparison).

4. **`TarXzArchiveExtractor` — device-node type rejection**  
   `BlockDevice`, `CharacterDevice`, and `Fifo` entries pass through the current guard and reach `TarEntry.ExtractToFileAsync`. On Linux, this would create real device nodes (requiring root privileges and producing kernel objects, not files). The fix mirrors the existing symlink/hard-link guard: throw `InvalidDataException` immediately for all three types.


## Rationale

- **Single `DesktopFileLauncherService` class over separate `LinuxFileLauncherService`:** The existing `MysqlDumpBackupService` handles both platforms internally without separate implementations. Consistent with that precedent; avoids a DI branch in `Program.cs`; reduces the class count.
- **`Ordinal` for path guards:** `OrdinalIgnoreCase` was not deliberately chosen for Linux; it was written when Windows was the only target. Switching to `Ordinal` is the correct behaviour on all platforms: paths from `Path.GetFullPath` are already in the canonical form the OS uses.
- **`Exe` over a conditional `OutputType`:** A conditional like `Condition="'$(OS)'=='Windows_NT'"` only evaluates the build machine's OS at compile time, not the target platform. For a framework-dependent deployment (as specified in `rebuild.md`), `Exe` produces identical output on Linux and Windows — the `.NET AppHost` is native per-platform regardless of this flag.


## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|----------|--------------|-------------------------|-------------------|
| Linux file-launch | `DesktopFileLauncherService` (OS branch inside one class) | Separate `LinuxFileLauncherService` + conditional DI registration | Single class is consistent with `MysqlDumpBackupService`; simpler DI; no extra interface entry-point |
| OutputType | `Exe` unconditionally | Conditional `WinExe`/`Exe` based on `$(OS)` MSBuild property | `$(OS)` reflects the *build* machine, not the target; `Exe` is unconditionally correct for a framework-dependent cross-platform binary |
| Zip/Tar-Slip comparison | `StringComparison.Ordinal` | Keep `OrdinalIgnoreCase`; add explicit Linux `[SkippableFact]` test | `Ordinal` is correct on all platforms; `OrdinalIgnoreCase` was never required for the Windows path |
| File-launch on Linux for ShowInExplorer | Open parent directory via `xdg-open` | `nautilus --select <file>` | `xdg-open` is distro-agnostic; nautilus-specific flag would break on KDE Plasma / Fedora Workstation with non-GNOME DE |


## Pattern Alignment

| Pattern | File | Alignment |
|---------|------|-----------|
| OS-branch inside a single service class | `MysqlDumpBackupService.cs` | **Followed** — `DesktopFileLauncherService` uses `OperatingSystem.IsWindows()` internally |
| `ProcessStartInfo.ArgumentList` for shell commands | `WindowsFileLauncherService.cs`, `MysqlDumpBackupService.cs` | **Followed** — Linux `xdg-open` arguments also use `ArgumentList` |
| `InvalidDataException` for archive guard violations | `TarXzArchiveExtractor.cs` (existing symlink guard) | **Followed** — device-node guard uses the same exception type and message pattern |
| `TreatWarningsAsErrors=true` | `Directory.Build.props` | **Followed** — all changes compile without warnings |


## Detailed Steps

1. **`VideoIndexer.App.csproj`** — Change `<OutputType>WinExe</OutputType>` to `<OutputType>Exe</OutputType>`.

2. **Rename `WindowsFileLauncherService.cs` → `DesktopFileLauncherService.cs`** — Update the class name, XML doc summary, and DI registration comment to reflect cross-platform scope.

3. **`DesktopFileLauncherService`** — Replace the implementation body:
   - `OpenFolder(string path)`: When Windows, use `explorer.exe` via `ArgumentList` (existing logic). When Linux (or other Unix), use `xdg-open` with `path` as the sole `ArgumentList` entry and `UseShellExecute = false`.
   - `ShowInExplorer(string filePath)`: When Windows, use `explorer.exe /select,<file>` (existing logic). When Linux, extract `Path.GetDirectoryName(filePath)` and pass it to `xdg-open` via `ArgumentList`.

4. **`Program.cs`** — Update `builder.Services.AddSingleton<IFileLauncherService, WindowsFileLauncherService>()` to `AddSingleton<IFileLauncherService, DesktopFileLauncherService>()`.

5. **`ZipArchiveExtractor.cs`** — Change the Zip-Slip guard comparison from `StringComparison.OrdinalIgnoreCase` to `StringComparison.Ordinal`.

6. **`TarXzArchiveExtractor.cs`** — Two changes in the entry-loop:
   - Change the Tar-Slip guard comparison from `StringComparison.OrdinalIgnoreCase` to `StringComparison.Ordinal`.
   - Add a new guard immediately after the existing symlink/hard-link rejection block, covering `TarEntryType.BlockDevice`, `TarEntryType.CharacterDevice`, and `TarEntryType.Fifo`, throwing `InvalidDataException` with a message matching the symlink guard pattern.

7. **Test fixture: `device.tar.xz`** — Create a new `.tar.xz` fixture file under `tests/VideoIndexer.Infrastructure.Tests/Fixtures/archives/` containing a `BlockDevice` entry. Use the Python 3 regeneration script documented in `TarXzArchiveExtractorTests.cs` (with `info.type = tarfile.BLKTYPE`).

8. **`TarXzArchiveExtractorTests.cs`** — Add one new `[Fact]` test in the existing AC-2 section:
   - `ExtractAsync_DeviceNodeEntry_ThrowsInvalidDataException` — uses `device.tar.xz`; asserts `InvalidDataException`.

9. **`ZipArchiveExtractorTests.cs`** — Add one new `[Fact]` test:
   - `ExtractAsync_TraversalEntry_OrdinalComparison_RejectsEntry` — Creates a zip entry whose fully-resolved path starts with the `canonicalDest` prefix in a different case (e.g., destination directory created as `Path.Combine(tempDir, "output")` but `canonicalDest` captured as the mixed-case result of `Path.GetFullPath`). On Windows, `Path.GetFullPath` does not normalise casing against the filesystem, so a crafted entry resolving to `c:\temp\OUTPUT\file.txt` would pass an `OrdinalIgnoreCase` check against the canonical destination `c:\temp\output\` but fail an `Ordinal` check. The test asserts the entry is rejected, confirming `Ordinal` is the active comparison. If constructing the case-mismatch scenario requires OS-specific filesystem state, the test may alternatively be structured as a code-pattern assertion — verifying the `StringComparison.Ordinal` argument is passed to `StartsWith` in the compiled guard — and clearly documented as such in the test's `[Fact]` XML summary.

10. **Documentation** — Update all affected manifest and milestone documents (see Documentation Updates section).


## Dependencies

- Steps 3–4 depend on step 2 (class rename must happen first to avoid referencing a deleted type).
- Steps 8–9 (tests) depend on step 7 (fixture file must exist).
- Documentation updates (step 10) can proceed in parallel with code changes.


## Required Components

- `src/VideoIndexer.App/VideoIndexer.App.csproj` — modify `OutputType`
- `src/VideoIndexer.App/Services/WindowsFileLauncherService.cs` → `DesktopFileLauncherService.cs` — rename + edit
- `src/VideoIndexer.App/Program.cs` — update DI registration
- `src/VideoIndexer.Infrastructure/ExternalTools/ZipArchiveExtractor.cs` — fix comparison
- `src/VideoIndexer.Infrastructure/ExternalTools/TarXzArchiveExtractor.cs` — fix comparison + add device-node guard
- `tests/VideoIndexer.Infrastructure.Tests/Fixtures/archives/device.tar.xz` — new binary fixture
- `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/TarXzArchiveExtractorTests.cs` — new tests
- `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/ZipArchiveExtractorTests.cs` — new test
- Documentation files (see Documentation Updates)


## Assumptions

- Fedora Workstation ships `xdg-open` as part of `xdg-utils`, which is a default package. No installation step is required.
- The application is run as a normal user (not root), so `xdg-open` has access to the user's desktop environment.
- `dotnet run --project src/VideoIndexer.App` is the intended launch method on Linux (framework-dependent deployment per `rebuild.md`).
- The `linux-x64` FFmpeg manifest entry in `appsettings.json` is already valid and verified (confirmed in README §"FFmpeg provisioning" — all five RID entries carry verified SHA-256 checksums).


## Constraints

- All warnings must remain zero (`TreatWarningsAsErrors=true`; `WarningLevel=9999`).
- `ProcessStartInfo.ArgumentList` must be used for all process argument passing — no reversion to `Arguments` string concatenation (per existing constraints.md rule on `DesktopFileLauncherService`).
- No new NuGet packages. `xdg-open` is invoked via `Process.Start`, requiring only BCL types.
- The `linux-x64` RID manifest entry in `appsettings.json` must not be modified (it carries a verified SHA-256 and is already in production shape).


## Out of Scope

- macOS support — not targeted by this plan. macOS (`osx-x64`, `osx-arm64`) already has manifest entries and provisioner support; a dedicated plan can address its `IFileLauncherService` implementation (which would use `open`).
- CI/CD pipeline changes — no build infrastructure currently exists; Linux build verification is a manual step on a Fedora Workstation machine.
- LibVLCSharp / M10 player — Linux video playback is a separate concern for the M10 milestone.
- `STAThread` attribute on `Main` — this attribute is a no-op on Linux and does not need to be removed. It is required on Windows for COM interop.


## Acceptance Criteria

- [ ] `dotnet build -c Release` produces zero warnings and zero errors on Windows.
- [ ] `dotnet build -c Release` produces zero warnings and zero errors on Fedora Workstation (linux-x64).
- [ ] `dotnet run --project src/VideoIndexer.App` launches the Avalonia UI on Fedora Workstation and reaches the `ProvisioningTools` shell state.
- [ ] All existing unit and app-layer tests pass unchanged (`.\test.ps1` green on Windows).
- [ ] New device-node rejection tests pass.
- [ ] New Zip-Slip `Ordinal`-comparison regression test passes.
- [ ] `VideoIndexer.App.csproj` no longer declares `WinExe`; `OutputType` is `Exe`.
- [ ] `IFileLauncherService` is registered against `DesktopFileLauncherService` in `Program.cs`; no reference to `WindowsFileLauncherService` remains in the codebase.
- [ ] `TarXzArchiveExtractor` throws `InvalidDataException` for `BlockDevice`, `CharacterDevice`, and `Fifo` entries.
- [ ] Both archive extractors use `StringComparison.Ordinal` for path-traversal guards.


## Testing Strategy

All code changes are unit-testable without a live database or filesystem side-effects:
- Archive extractor changes are covered by the existing `VideoIndexer.Infrastructure.Tests` fixture-based test suite plus two new tests.
- `DesktopFileLauncherService` uses `Process.Start` and its behaviour is platform-dependent; it is not unit-tested by convention (consistent with the existing `WindowsFileLauncherService`, which has no dedicated unit tests — only the `FakeFileLauncherService` used in App-layer ViewModel tests). Manual smoke-test on Fedora covers the Linux path.
- The `OutputType` change requires no test — it is a project property.


## Test Plan

- `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/TarXzArchiveExtractorTests.cs`
  — `ExtractAsync_DeviceNodeEntry_ThrowsInvalidDataException` — asserts `InvalidDataException` is thrown when extracting `device.tar.xz` containing a `BlockDevice` entry — covers AC: `TarXzArchiveExtractor` rejects device-node types.

- `tests/VideoIndexer.Infrastructure.Tests/ExternalTools/ZipArchiveExtractorTests.cs`  
  — `ExtractAsync_TraversalEntry_OrdinalComparison_RejectsEntry` — Asserts that an entry whose resolved full path starts with the canonical destination under a case-insensitive comparison but not a case-sensitive comparison is rejected. Constructs `canonicalDest` using `Path.GetFullPath` over a mixed-case temporary path to create the divergence. Alternatively structured as a code-pattern assertion if the filesystem case-divergence is not constructible in the test environment — covers AC: both archive extractors use `StringComparison.Ordinal`.

- (Regression) All 8 existing `ZipArchiveExtractorTests` and all existing `TarXzArchiveExtractorTests` — must continue to pass unchanged — covers AC: no regression in extraction or traversal-guard behaviour.


## Documentation Updates

- `src/VideoIndexer.App/VideoIndexer.App.csproj` — `OutputType` changed; no doc file needed, but triggers the entries below.
- `docs/agents/project-manifest/tech-stack.md` — Update `Output type` row from `WinExe (Windows desktop application)` to `Exe (cross-platform; console window suppression on Windows deferred to future app.manifest hardening if needed)`.
- `docs/agents/project-manifest/constraints.md` — Under "UI & MVVM", update the `WindowsFileLauncherService uses ProcessStartInfo.ArgumentList` rule to reference `DesktopFileLauncherService` and add a note about the Linux `xdg-open` path also using `ArgumentList`. Remove the README-sourced TODO notes about Zip-Slip `OrdinalIgnoreCase` and device-node gaps.
- `README.md` — Remove the two resolved TODO paragraphs: (1) the Zip-Slip security-considerations paragraph (approximately line 793) noting the `OrdinalIgnoreCase` issue and suggesting switching to `Ordinal` if Linux support is added; (2) the `TarXzArchiveExtractor` "Entry filtering" paragraph (approximately line 815) noting that device-node types (`BlockDevice`, `CharacterDevice`, `Fifo`) are not yet rejected. Both issues are resolved by this plan.
- `docs/agents/project-manifest/file-tree.md` — Rename `WindowsFileLauncherService.cs` entry to `DesktopFileLauncherService.cs` and update its annotation, and change the `VideoIndexer.App/` directory annotation comment from `(WinExe)` to `(Exe)`.
- `docs/agents/project-manifest/api-surface.md` — Rename the class section header from `WindowsFileLauncherService : IFileLauncherService` to `DesktopFileLauncherService : IFileLauncherService` and update the annotation to reflect the cross-platform implementation.
- `src/VideoIndexer.App/Services/IFileLauncherService.cs` — Update the `ShowInExplorer` XML doc comment. The current doc reads `"Opens the containing folder of filePath and selects the file."` On Linux this behavior cannot be honored. Update to clarify the Linux behavior (opens parent directory without selecting the file).
- `docs/projects/rebuild/milestones/m1-foundation.md` — Update Acceptance Criterion 2 from `VideoIndexer.App outputs as WinExe` to `VideoIndexer.App outputs as Exe (cross-platform)`.
- `AGENTS.md` — Update `Output type` stat row from `WinExe (Windows desktop)` to `Exe (cross-platform)`.


## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **`xdg-open` not installed on Fedora** | `xdg-utils` is a default package on Fedora Workstation; `Process.Start` returns `null` on failure, which `DesktopFileLauncherService` already discards (same as the existing Windows path — `_ = Process.Start(psi)`). No crash results. |
| **Console window flash on Windows after `WinExe` → `Exe`** | Cosmetic only for a dev tool launched from the terminal. An `app.manifest` can suppress it if the UX becomes a concern; deferred to a future aesthetic hardening pass. |
| **`device.tar.xz` fixture generation requires Python 3** | The Python regeneration script is already documented in `TarXzArchiveExtractorTests.cs`. The fixture is committed as a binary file, so consumers do not need Python to run the tests. |
| **`ShowInExplorer` semantic change on Linux** | On Linux, the method opens the parent directory rather than selecting the file. This is a known limitation of `xdg-open`. The `IFileLauncherService` interface carries no contract for "file selection"; only folder-browsing semantics are expected by callers. |
