# Synthesis — M1 Foundation
**Plan:** [2026-04-27-m1-foundation/plan.md](plan.md)  
**Date:** 2026-04-27  
**Status:** COMPLETE — all 8 work packages passed all pipeline stages  
**Pipeline health:** 8/8 WPs with all stages passing · 0 missing stages · 0 blocked WPs

---

## Executive Summary

M1 delivered a fully functional application shell for Video Indexer MKII on Avalonia/.NET 8.
Starting from a documentation-only repository with no source code, the milestone produced a
four-project layered solution that boots cleanly, loads and persists settings from
`%APPDATA%\VideoIndexer\appsettings.json`, writes rolling log files under
`%APPDATA%\VideoIndexer\logs\`, applies FluentTheme in System/Light/Dark modes, and ships
41 passing unit tests. Every acceptance criterion across all 8 work packages was met. The
build is clean at zero warnings under `TreatWarningsAsErrors=true`.

---

## Work Package Outcomes

### WP-001 · Repository Scaffold
**Stages:** implementation → code-review  
**Deliverables:** `src/`, `tests/`, `.gitignore`, `.editorconfig`, `global.json`,
`Directory.Build.props`, `src/VideoIndexer.sln`

Established the repository skeleton with inheritable build settings (`Nullable=enable`,
`TreatWarningsAsErrors=true`, `LangVersion=latest`). One environment constraint was noted:
the machine lacks a .NET 8 SDK; `global.json` is authored correctly with
`rollForward: latestMajor`, which resolves to .NET 10 on this host. The classic `.sln`
format was created manually because `dotnet new sln` now defaults to `.slnx` under
.NET 10.

**Open items for M2:**
- Consider constraining `global.json` to `latestFeature` once the team standardises on
  .NET 8, or document the `latestMajor` intent explicitly.
- Evaluate migrating to `.slnx` once VS 2022 17.x adoption is confirmed team-wide.
- `Directory.Build.props` does not set `<WarningLevel>`. Adding
  `<WarningLevel>9999</WarningLevel>` would surface all available warnings alongside
  `TreatWarningsAsErrors=true`.

---

### WP-002 · Core Domain Layer
**Stages:** implementation → code-review  
**Deliverables:** `VideoIndexer.Core` project — `AppOptions` records, `ThemeMode` enum,
`IAppPaths` / `ISettingsService` / `IThemeService` interfaces

The Core layer is intentionally free of all I/O, Avalonia, and Microsoft.Extensions.*
dependencies. The five option records (`AppOptions`, `LoggingOptions`, `AppearanceOptions`,
`WindowOptions`, `ExternalToolsOptions`) match the documented `appsettings.json` schema and
use `init`-only properties with full nullable annotation. All three abstractions carry
`CancellationToken` on async operations.

**Design decisions:**
- `AppearanceOptions.Theme` is typed as `ThemeMode` (enum); the configuration binder
  handles string-to-enum conversion automatically. This is implicit — future WPs should
  document it or add a JSON converter attribute.
- `ISettingsService` exposes `SaveAsync` but no `LoadAsync`. If reload-on-change is ever
  activated, `Current` can silently become stale. A `SettingsChanged` event or
  `IObservable<AppOptions>` should be tracked for a future WP.
- `IThemeService.ThemeChanged` is typed `EventHandler<ThemeMode>` (non-idiomatic; .NET
  convention expects `TEventArgs : EventArgs`). A thin `ThemeModeChangedEventArgs` wrapper
  would be idiomatic but is not blocking.

---

### WP-003 · Infrastructure — AppPaths & Settings
**Stages:** implementation → qa → security-audit → code-review  
**Deliverables:** `VideoIndexer.Infrastructure` project, `AppPaths`, `JsonSettingsService`

`AppPaths` resolves `Root` to `%APPDATA%\VideoIndexer`, creates `LogsDirectory` on
construction. `JsonSettingsService` handles three scenarios: first-run default write,
corrupted-file fallback to defaults, and atomic save via `.tmp` + `File.Move(overwrite:true)`.
`IAppPaths.RootDirectory` was renamed to `Root` during implementation to match the AC
wording — a safe breaking change as no consumer existed yet.

**Security findings (all Low):**
- `ExternalToolsOptions` path fields (`VlcExecutablePath`, `MySqlBinaryFolder`,
  `ExternalXmlEditorPath`) and `LoggingOptions.FilePath` are persisted without validation.
  When future milestones launch processes or write to these paths, inputs must be
  canonicalized and prefix-checked against safe roots.
- `File.Move` atomicity holds for same-volume NTFS moves (`%APPDATA%`) but is not
  guaranteed on cross-volume or network paths.

**Technical debt:**
- `JsonSettingsService` constructor calls `LoadOrInitialise()` synchronously. Acceptable
  for a sub-1 KB file at M1; revisit if startup performance becomes a concern.
- `SaveAsync` has no `try/finally` cleanup for the `.tmp` file on cancellation or I/O
  error. The target file is never corrupted, but the temp file is orphaned. A
  `try/finally { File.Delete(tempFile) }` guard should be added.
- Concurrent `SaveAsync` calls race on a single `.tmp` filename. Not a concern for a
  single-user desktop app but should be flagged if the service is ever accessed from
  multiple threads.
- `UnauthorizedAccessException` is not caught; it propagates to the caller. Decide whether
  access-denied scenarios should fall back to defaults or surface to the user.

---

### WP-004 · Infrastructure — ThemeService & Logging
**Stages:** implementation → qa → security-audit → code-review  
**Deliverables:** `ThemeService`, `LoggingSetup.ConfigureSerilog`, 21 unit tests

`ThemeService` reads the initial theme from `ISettingsService.Current` on construction,
`Set(mode)` updates `Current`, persists via `SaveAsync`, and raises `ThemeChanged` exactly
once. `LoggingSetup.ConfigureSerilog` maps verbosity 1–4 to Warning / Information / Debug /
Verbose with `Math.Clamp` for out-of-range values; the rolling file sink targets
`<LogsDirectory>/videoindexer-.log` with 14-day retention and a `RollingInterval.Day`
pattern. `MapVerbosity` is `public static` specifically to enable unit testing in isolation.

21 unit tests were created in `VideoIndexer.Infrastructure.Tests` (targeting `net10.0` to
work around the absent .NET 8 runtime on this host; production code remains `net8.0`). All
21 pass.

**Technical debt:**
- `ThemeService.Set` calls `SaveAsync().GetAwaiter().GetResult()` — the sync-over-async
  anti-pattern. Acceptable at M1 (user-initiated, <1 KB write); if ReactiveUI or async
  MVVM is adopted, `ThemeService` should expose `SetAsync`.

---

### WP-005 · App Shell
**Stages:** implementation → qa → security-audit → code-review → documentation  
**Deliverables:** `VideoIndexer.App` project — `Program.cs`, `App.axaml/cs`, `ViewLocator`,
`MainWindowViewModel`, `MainWindow.axaml/cs`, embedded `appsettings.json`

`Program.cs` uses a two-phase init: `AppPaths` and `EnsureDefaultSettingsFile` are
constructed before the Generic Host to guarantee the settings file exists before
`IConfiguration` reads it. The Host wires `IAppPaths`, `ISettingsService`, `IThemeService`,
Serilog, and the MVVM types via DI. `App.axaml.cs` subscribes to `ThemeChanged` and applies
the `ThemeVariant` to `RequestedThemeVariant` before `MainWindow` is shown. `ViewLocator`
resolves views from DI via namespace-convention string replacement.

All 6 acceptance criteria were met. AC2 / 4 / 5 are runtime-only (require a display) and
were verified by code inspection.

**Design decisions:**
- `App.Services` is a static `IServiceProvider` property — the standard Avalonia+DI bridge.
  Makes `App` harder to unit test; a pure constructor-injection pattern should be preferred
  if the App class grows in complexity.
- `Program.cs` early-init uses `NullLogger<JsonSettingsService>.Instance`. If
  `LoadOrInitialise()` encounters a corrupted settings file in the narrow window after
  `EnsureDefaultSettingsFile` completes, the warning log is silently dropped. Using a
  bootstrap logger or deferring `JsonSettingsService` construction until after Serilog is
  configured would close this gap.
- Dual first-run init: `EnsureDefaultSettingsFile` in `Program.cs` and
  `WriteDefaultsSync` in `JsonSettingsService` are both idempotent (guarded by
  `File.Exists`), but the redundancy adds cognitive load. Consider removing one path in M2.

---

### WP-006 · Debug Theme Cycle (Ctrl+T)
**Stages:** implementation → qa → code-review  
**Deliverables:** `#if DEBUG`-gated `CycleTheme` command in `MainWindowViewModel`,
Ctrl+T `KeyBinding` in `MainWindow.axaml.cs`

`CycleTheme` is a `[RelayCommand]`-annotated private method that cycles System → Light →
Dark → System via `IThemeService.Set`. The `KeyBinding` is added programmatically in
code-behind (XAML has no `#if` directive support). Binary inspection confirmed the symbol
is absent from the Release DLL. All 4 ACs met; 41 unit tests pass (20 from WP-007 +
21 from Infrastructure).

**Convention note:** The DI constructor of `MainWindow` could be `internal` with
`[assembly: InternalsVisibleTo]` to prevent accidental direct instantiation outside the
entry point. Not a concern for the current single-project scope.

---

### WP-007 · Unit Test Project
**Stages:** implementation → code-review  
**Deliverables:** `VideoIndexer.Tests` project — 20 tests across 4 test classes

| Test class | Tests | Coverage |
|---|---|---|
| `AppPathsTests` | 3 | Root resolves under AppData; SettingsFile inside Root; LogsDirectory created |
| `JsonSettingsServiceTests` | 5 | First-run write; defaults; save/load round-trip; corrupted-file fallback; temp-file cleanup |
| `ThemeServiceTests` | 4 | Set updates Current; raises event exactly once; persisted value re-emits on reload; initial Current from ISettingsService |
| `LoggingVerbosityMappingTests` | 8 | All 4 in-range mappings + 4 out-of-range clamp cases via `[Theory]` |

All 20 tests pass. Tests use isolated `TempDirectory` + `FakeAppPaths` /
`InMemorySettingsService` helper classes.

**Note:** `AppPathsTests.LogsDirectory_ShouldBe_InsideRoot_AndExist` touches the real
`%APPDATA%\VideoIndexer\logs\` directory (the production side-effect of `AppPaths`
construction). Acceptable given that is the unit under test.

**Open items for M2:**
- Extract `TempDirectory`, `FakeAppPaths`, and `InMemorySettingsService` to a shared
  `tests/VideoIndexer.Tests/Fixtures/` folder when a second test file needs them.
- Add `GlobalUsings.cs` to centralise common `using Xunit;`, `using FluentAssertions;`
  imports.
- Consider whether `AppPaths` should accept the root as a constructor parameter (via
  `IOptions<AppPathsOptions>` or similar) to avoid touching real AppData in tests.

---

### WP-008 · Release Verification & Documentation
**Stages:** qa (rework×1) → code-review → release-engineering → documentation  
**Deliverables:** `docs/projects/rebuild/milestones/m1-foundation.md`, updated `README.md`

The first QA pass found two documentation gaps (AC4: missing milestone doc; AC5: README
lacked a `## Build & Run` section). Both were resolved before the second QA pass, which
passed all 5 ACs. Release engineering confirmed zero warnings on a clean restore + Release
build with `TreatWarningsAsErrors=true`, 20/20 tests passing, and no forbidden packages.
The documentation agent confirmed all content was accurate with no corrections required.

**README.md verbosity table** was found by the Reviewer to be inverted vs the
implementation; this was corrected before Release Engineering. Final confirmed scale:
1 = Warning, 2 = Information, 3 = Debug, 4 = Verbose.

---

## Test Metrics

| Project | Tests | Pass | Fail |
|---|---|---|---|
| `VideoIndexer.Tests` | 20 | 20 | 0 |
| `VideoIndexer.Infrastructure.Tests` | 21 | 21 | 0 |
| **Total** | **41** | **41** | **0** |

Build: `dotnet build -c Release` — **0 warnings, 0 errors** across all 4 projects.

---

## Security Summary

No Critical or High findings were identified across any WP. Open Low-severity items:

| Finding | Location | Status |
|---|---|---|
| Path-traversal risk on `ExternalToolsOptions` fields | `AppOptions` records | Open — deferred to the WP that first launches processes |
| `LoggingOptions.FilePath` not validated (UNC redirect) | `LoggingSetup.cs` | Open — Low risk in single-user desktop threat model |
| Console sink unconditionally active at Verbosity 4 | `LoggingSetup.cs` | Open — Info severity; acceptable for desktop |

All OWASP Top 10 categories were assessed for each security-audited WP (WP-003,
WP-004, WP-005). No new CVEs identified in the dependency set at the pinned versions.

---

## Technical Debt Register

Items carried forward for M2 consideration, in priority order:

| Priority | Item | Source |
|---|---|---|
| Medium | `ISettingsService` has no change-notification mechanism; `Current` can silently become stale if reload-on-change is active | WP-002 review |
| Medium | `SaveAsync` lacks `try/finally` temp-file cleanup on cancellation / I/O error | WP-003 QA |
| Low | `ThemeService.Set` uses sync-over-async (`GetAwaiter().GetResult()`) | WP-004 implementation |
| Low | Dual first-run init (both `EnsureDefaultSettingsFile` and `JsonSettingsService.WriteDefaultsSync`) | WP-008 review |
| Low | `App.Services` static property acts as a service locator; makes App untestable | WP-005 implementation |
| Low | `ViewLocator` namespace-convention resolution breaks silently on namespace rename | WP-005 implementation |
| Low | No CI pipeline or publish profile; Release artifacts are not reproducible from a clean agent | WP-008 release-engineering |
| Low | `global.json` `rollForward: latestMajor` should be tightened or documented | WP-001 review |
| Low | `Directory.Build.props` missing `<WarningLevel>9999</WarningLevel>` | WP-001 review |
| Low | `IThemeService.ThemeChanged` event args type is non-idiomatic (`EventHandler<ThemeMode>`) | WP-002 review |

---

## Patterns & Conventions Established

The following conventions were agreed upon and applied consistently across M1; all
subsequent milestones should follow them:

- **Namespace layout:** `VideoIndexer.Core.Options`, `VideoIndexer.Core.Abstractions`,
  `VideoIndexer.Core.Enums` — pure types only, no I/O.
- **Sealed records with `init`-only properties** for all option/DTO types.
- **Full nullable annotation** (`Nullable=enable`) project-wide.
- **Atomic file write** via `.tmp` + `File.Move(overwrite:true)` for any settings write.
- **`#if DEBUG` guards** on all debug-only UI scaffolding (commands, key bindings) with
  binary verification before shipping.
- **xUnit + FluentAssertions** as the test stack. Test naming:
  `Subject_Scenario_ExpectedBehavior`. Isolation via `IDisposable` temp-directory
  fixtures and in-memory fakes.
- **`TreatWarningsAsErrors=true`** enforced at solution level; zero-warning builds are
  required to merge.
- **Dependency direction enforced:** Core → (nothing); Infrastructure → Core;
  App → Core + Infrastructure; Tests → Core + Infrastructure.

---

## Environment Notes

- The development machine has .NET 10.0.6 installed; .NET 8 SDK/runtime is absent.
  `global.json` is correctly authored but resolves to .NET 10 on this host.
- Test projects target `net10.0` (Infrastructure.Tests) or `net8.0` with
  `RollForward=LatestMajor` (VideoIndexer.Tests) to execute on the available runtime.
  Production source projects remain `net8.0`.
- No `.NET 8.0 runtime` is required for M1 verification on this host; all pipelines passed.

---

## Recommendations for M2

1. **Add UNC-path guard to `LoggingSetup`** — one-line fix: validate
   `options.Logging.FilePath` is not a UNC path before passing to Serilog.
2. **Add `try/finally` temp-file cleanup in `SaveAsync`** — prevents orphaned `.tmp`
   files on disk-full or cancellation.
3. **Consider a `SettingsChanged` notification on `ISettingsService`** before any M2 WP
   depends on reload-on-change being reactive.
4. **Extract test fixtures** (`TempDirectory`, `FakeAppPaths`, `InMemorySettingsService`)
   to `tests/VideoIndexer.Tests/Fixtures/` before the next test file is added.
5. **Add a CI pipeline** (GitHub Actions or Azure Pipelines) with restore → build -c
   Release → test steps and a `dotnet publish` profile for `win-x64`.
6. **Add Avalonia Headless smoke tests** for runtime-only ACs (window launch, theme
   application, log file creation) to remove dependency on manual verification.
