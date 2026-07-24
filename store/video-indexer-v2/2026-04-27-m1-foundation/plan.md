# Plan — M1 Foundation

## Summary

Establish the cross-platform Avalonia/.NET 8 solution that every later milestone of the
Video Indexer MKII rebuild will inherit. M1 ships no end-user features; it ships an
**application shell that boots cleanly**, with a layered project structure, a generic-host
DI/configuration/logging stack, an `appsettings.json` loaded from
`Environment.SpecialFolder.ApplicationData`, FluentTheme with a runtime light/dark switch,
the CommunityToolkit.Mvvm binding stack, and a test project wired to xUnit. Done means:
running the app from a fresh clone produces an empty themed main window, a log file at the
configured location, and `dotnet test` passes against placeholder smoke tests.

## Architectural Context

The `video-indexer-v2` repository currently contains only documentation
([docs/projects/rebuild/](../../../projects/rebuild/)), `LICENSE`, and `README.md`. There
is **no existing source code, no `.sln`, no `AGENTS.md`, and no prior project layout to
follow** — M1 is a true greenfield bootstrap.

Authoritative inputs for M1 come from:

- [docs/projects/rebuild/rebuild.md](../../../projects/rebuild/rebuild.md) — tech stack
  (Avalonia UI + LibVLCSharp on .NET 8+, Dapper + MySqlConnector), deployment
  (framework-dependent, no installer), and configuration storage
  (`appsettings.json` in `Environment.SpecialFolder.ApplicationData`).
- [docs/projects/rebuild/management-areas/system-management-specification.md](../../../projects/rebuild/management-areas/system-management-specification.md)
  §2 — Preferences taxonomy (General, Database, Thumbnails, Appearance, Logging,
  Obfuscation) and the split between `spdb_config` (DB-side, M2+) and the local
  `app.config` file (machine-specific paths, M1 scope).
- [docs/projects/rebuild/milestones/roadmap.md](../../../projects/rebuild/milestones/roadmap.md)
  — milestone scope and the rule that *the app must still launch cleanly with all prior
  milestones intact* after every milestone.

The legacy WinForms implementation under
[../spdb-indexer/SPDB Indexer/](../../../../../spdb-indexer/SPDB%20Indexer/) is referenced
**only** for behavioural parity in later milestones. M1 does not port any code from it.

## Approach / Architecture

### Solution layout (layered, four projects)

```
video-indexer-v2/
├── src/
│   ├── VideoIndexer.Core/              # Domain types, abstractions, no I/O
│   ├── VideoIndexer.Infrastructure/    # Config, logging, file system, future DB
│   ├── VideoIndexer.App/               # Avalonia UI host (entry point)
│   └── VideoIndexer.sln
└── tests/
    └── VideoIndexer.Tests/             # xUnit, references Core + Infrastructure
```

- **`VideoIndexer.Core`** owns pure abstractions (`IAppPaths`, `IThemeService`,
  `AppOptions` records, future domain models). No dependencies on Avalonia or the .NET
  Generic Host. Keeps the door open for swapping the UI layer.
- **`VideoIndexer.Infrastructure`** implements those abstractions: filesystem-backed
  `appsettings.json` loader, the Serilog logger configuration, `AppPaths` (resolves
  `%APPDATA%/VideoIndexer/` and equivalents on Linux/macOS via
  `Environment.SpecialFolder.ApplicationData`).
- **`VideoIndexer.App`** hosts Avalonia. Composes the host in `Program.cs`, registers
  services, sets up the Avalonia `App` to resolve the `MainWindowViewModel` from DI, and
  wires the FluentTheme switch.
- **`VideoIndexer.Tests`** uses xUnit + FluentAssertions. M1 ships smoke tests only
  (config round-trip, theme service mode toggling, app paths resolution).

### Composition root

`Program.cs` builds a `Microsoft.Extensions.Hosting.HostApplicationBuilder`:

1. Resolve `IAppPaths` early (synchronously) to know where `appsettings.json` lives.
2. `builder.Configuration.AddJsonFile(appPaths.SettingsFile, optional: true, reloadOnChange: true)`
   — and on first run, write the bundled defaults to that path before adding it.
3. `builder.Services.Configure<AppOptions>(builder.Configuration)`.
4. Register Serilog via `builder.Services.AddSerilog(...)`, sinks: Console + rolling
   file under `appPaths.LogsDirectory`. Verbosity bound to `Logging:Verbosity` (1–4 →
   `LogLevel`).
5. Register MVVM types (`MainWindowViewModel` etc.) and services (`IThemeService`,
   `ISettingsService`) as singletons / transients as appropriate.
6. Build the host, then call `BuildAvaloniaApp().StartWithClassicDesktopLifetime(args)`
   passing the host's `IServiceProvider` to the `App` instance via a static accessor (the
   standard Avalonia + Generic Host bridge).

### Configuration model

A single strongly-typed `AppOptions` record tree mapped from `appsettings.json`:

```jsonc
{
  "Logging":     { "Verbosity": 2, "FilePath": null },              // null ⇒ use AppPaths.LogsDirectory
  "Appearance":  { "Theme": "System", "Language": "en" },           // System | Light | Dark
  "Window":      { "Width": 1280, "Height": 800, "X": null, "Y": null, "Maximized": false },
  "ExternalTools": {
    "VlcExecutablePath":     null,
    "MySqlBinaryFolder":     null,
    "ExternalXmlEditorPath": null
  }
}
```

- `null` placeholders for external tool paths are intentional — they signal "not yet
  configured" to later milestones (M2 needs MySQL, M8 surfaces the editors in
  Preferences, M10 needs VLC).
- `ExternalTools` and `Window` are **machine-local only**; their existence in M1 reserves
  the schema for M8 without committing UI to it.

### Theming

- Reference `Avalonia.Themes.Fluent`. `App.axaml` declares the `FluentTheme` resource.
- `IThemeService` exposes `Current`, `Set(ThemeMode)`, and `ThemeChanged` event. It
  mutates `Application.Current.RequestedThemeVariant` and persists the choice through
  `ISettingsService`.
- `System` mode honours the OS theme via Avalonia's `ActualThemeVariant`.
- A debug-only menu item (or keyboard shortcut `Ctrl+T`) cycles modes in M1 so the
  toggle is exercisable without the Preferences dialog (which lands in M8).

### Logging

- **Serilog** chosen over plain `Microsoft.Extensions.Logging` because M.E.L has no
  built-in file sink; Serilog with `Serilog.Sinks.Console` + `Serilog.Sinks.File` is the
  lightest practical way to satisfy the spec's "log to disk with verbosity levels" intent
  without writing a custom provider. Consumers still depend on `ILogger<T>` from
  `Microsoft.Extensions.Logging.Abstractions`, so Serilog can be swapped later without
  touching call sites.
- Verbosity mapping: `1 → Warning`, `2 → Information`, `3 → Debug`, `4 → Verbose`.
- Rolling file: one file per day under `<AppData>/VideoIndexer/logs/videoindexer-.log`, retained
  for 14 days.

### MVVM

- `CommunityToolkit.Mvvm` with source generators (`[ObservableProperty]`,
  `[RelayCommand]`).
- `ViewLocator` resolves `FooView` from `FooViewModel` by naming convention; instantiates
  views via DI to allow constructor injection.
- M1 ships exactly one VM/View pair: `MainWindowViewModel` / `MainWindow.axaml`. The
  window contains a placeholder welcome message, the app version, and the theme cycle
  shortcut hint.

## Rationale

- **Layered split (Core / Infrastructure / App / Tests)** chosen by the user. It costs
  modest ceremony now but avoids a painful mid-rebuild refactor when M2 introduces the
  Dapper data layer and M9 introduces the ffmpeg pipeline — both fit naturally into
  Infrastructure without polluting the UI project.
- **Generic Host over a hand-rolled `ServiceCollection`** because configuration binding,
  logging integration, and lifetime management come for free and match what every M2+
  background worker (refresh worker, obfuscation worker, thumbnail generator) will need.
- **Serilog over NLog/M.E.L-only** for the file-sink reason above. It is ubiquitous
  enough to count as a standard dependency, not a heavy one.
- **CommunityToolkit.Mvvm over ReactiveUI** per user choice. Source generators keep VM
  code idiomatic and discoverable for AI agents working on later milestones.
- **FluentTheme with runtime switch** so the Preferences → Appearance tab in M8 has
  something to bind against; building the switch later would risk re-architecting how
  resources are loaded.
- **Bundle defaults + write on first run** rather than embedding defaults in code only.
  This gives users a discoverable, editable file from the very first launch, matching
  the behaviour of the legacy app.

## Detailed Steps

1. **Repository skeleton**
   - Create `src/` and `tests/` directories at repo root.
   - Add `.gitignore` (Visual Studio template) and `.editorconfig`
     (4-space C#, `dotnet_diagnostic` defaults, file-scoped namespaces, `nullable=enable`).
   - Add `global.json` pinning the .NET 8 SDK band.
   - Add `Directory.Build.props` at repo root setting `LangVersion=latest`,
     `Nullable=enable`, `TreatWarningsAsErrors=true`, and a shared `AssemblyVersion`.

2. **Create `VideoIndexer.Core` (`netstandard2.1` or `net8.0`; pick `net8.0` for
   simplicity)**
   - No package references.
   - Add: `AppOptions`, `LoggingOptions`, `AppearanceOptions`, `WindowOptions`,
     `ExternalToolsOptions` records.
   - Add: `IAppPaths`, `ISettingsService`, `IThemeService` interfaces.
   - Add: `ThemeMode` enum (`System`, `Light`, `Dark`).

3. **Create `VideoIndexer.Infrastructure` (`net8.0`)**
   - Packages: `Microsoft.Extensions.Configuration`,
     `Microsoft.Extensions.Configuration.Json`,
     `Microsoft.Extensions.Configuration.Binder`,
     `Microsoft.Extensions.Options.ConfigurationExtensions`,
     `Microsoft.Extensions.Logging.Abstractions`,
     `Serilog.Extensions.Hosting`, `Serilog.Sinks.Console`, `Serilog.Sinks.File`.
   - Implement `AppPaths` (root: `Environment.SpecialFolder.ApplicationData/VideoIndexer/`;
     subfolders: `logs/`).
   - Implement `JsonSettingsService` (load, save, atomic write via temp file + replace,
     bundled defaults written on first run).
   - Implement `ThemeService` (in-memory state + persistence delegated to
     `ISettingsService`; raises `ThemeChanged`).
   - Implement `LoggingSetup.ConfigureSerilog(LoggerConfiguration, AppOptions, IAppPaths)`.

4. **Create `VideoIndexer.App` (`net8.0`, `OutputType=WinExe`)**
   - Packages: `Avalonia`, `Avalonia.Desktop`, `Avalonia.Themes.Fluent`,
     `Avalonia.Fonts.Inter`, `Avalonia.ReactiveUI` **excluded**,
     `CommunityToolkit.Mvvm`, `Microsoft.Extensions.Hosting`,
     `Serilog.Extensions.Hosting`.
   - `Program.cs`: build host, build Avalonia, hand the `IServiceProvider` to `App`.
   - `App.axaml` + `App.axaml.cs`: declare `FluentTheme`, set `DataTemplates` with the
     custom `ViewLocator`, on framework init resolve `MainWindow` from DI.
   - `Views/MainWindow.axaml` + `ViewModels/MainWindowViewModel.cs`.
   - `ViewLocator.cs` (DI-aware).
   - `Assets/` folder placeholder (icon TBD; ship blank for M1).

5. **Bundle default `appsettings.json`**
   - Embed the defaults JSON as a resource in `VideoIndexer.App`.
   - On startup, if `<AppData>/VideoIndexer/appsettings.json` does not exist, write the
     defaults to disk before the host configuration system reads it.

6. **Wire the theme cycle shortcut**
   - `MainWindowViewModel` exposes `CycleThemeCommand`; `MainWindow` binds it to
     `Ctrl+T`. Persists via `IThemeService` → `ISettingsService`.

7. **Create `VideoIndexer.Tests` (`net8.0`)**
   - Packages: `Microsoft.NET.Test.Sdk`, `xunit`, `xunit.runner.visualstudio`,
     `FluentAssertions`.
   - Tests:
     - `AppPathsTests` — root resolves under `ApplicationData`; `logs/` created on
       demand.
     - `JsonSettingsServiceTests` — round-trip save/load; corrupted file falls back to
       defaults; first-run writes bundled defaults.
     - `ThemeServiceTests` — `Set` raises `ThemeChanged`; persisted value re-emits on
       reload.
     - `LoggingVerbosityMappingTests` — 1→Warning, 2→Information, 3→Debug, 4→Verbose;
       out-of-range clamps to nearest.

8. **Solution file & build verification**
   - `dotnet new sln -n VideoIndexer` at repo root, add all four projects.
   - `dotnet build -c Release` succeeds with zero warnings.
   - `dotnet test` succeeds.
   - `dotnet run --project src/VideoIndexer.App` opens the themed main window.

9. **Lightweight documentation**
   - Update `README.md` with a "Build & Run" section (prereqs: .NET 8 SDK; `dotnet run`
     command; where the settings file lands).
   - Add `docs/projects/rebuild/milestones/m1-foundation.md` per the roadmap's
     "Milestone Document Template", referencing this plan.

## Dependencies

- **External tooling:** .NET 8 SDK installed on the developer machine.
- **NuGet packages** (versions resolved at implementation time, all stable):
  - `Avalonia`, `Avalonia.Desktop`, `Avalonia.Themes.Fluent`, `Avalonia.Fonts.Inter`
  - `CommunityToolkit.Mvvm`
  - `Microsoft.Extensions.Hosting`, `Microsoft.Extensions.Configuration.Json`,
    `Microsoft.Extensions.Configuration.Binder`,
    `Microsoft.Extensions.Options.ConfigurationExtensions`
  - `Serilog.Extensions.Hosting`, `Serilog.Sinks.Console`, `Serilog.Sinks.File`
  - Test stack: `xunit`, `xunit.runner.visualstudio`, `Microsoft.NET.Test.Sdk`,
    `FluentAssertions`
- **Specifications that must remain stable for M1:** the tech-stack and configuration
  sections of [rebuild.md](../../../projects/rebuild/rebuild.md). No other spec is on
  the critical path because M1 ships no domain features.

## Required Components

All components below are **new** (the repository has no existing source code).

- `src/VideoIndexer.sln` — solution file at `src/` root (or repo root; pick repo root for
  discoverability).
- `Directory.Build.props`, `global.json`, `.gitignore`, `.editorconfig` at repo root.
- `src/VideoIndexer.Core/` project with:
  - `Configuration/AppOptions.cs`, `LoggingOptions.cs`, `AppearanceOptions.cs`,
    `WindowOptions.cs`, `ExternalToolsOptions.cs`
  - `Abstractions/IAppPaths.cs`, `ISettingsService.cs`, `IThemeService.cs`
  - `Theming/ThemeMode.cs`
- `src/VideoIndexer.Infrastructure/` project with:
  - `Paths/AppPaths.cs`
  - `Settings/JsonSettingsService.cs`, `Settings/DefaultSettings.cs`
  - `Theming/ThemeService.cs`
  - `Logging/LoggingSetup.cs`
- `src/VideoIndexer.App/` project with:
  - `Program.cs`, `App.axaml`, `App.axaml.cs`, `ViewLocator.cs`
  - `Views/MainWindow.axaml`, `Views/MainWindow.axaml.cs`
  - `ViewModels/MainWindowViewModel.cs`, `ViewModels/ViewModelBase.cs`
  - `Assets/appsettings.default.json` (embedded resource)
- `tests/VideoIndexer.Tests/` project with one test class per service listed in step 7.

No external services or infrastructure are required.

## Assumptions

- The user has a .NET 8 SDK install path resolvable on PATH; CI is **out of scope** for
  M1 (the user opted for Windows-only manual verification).
- `Environment.SpecialFolder.ApplicationData` is writable on the developer's machine
  (true on all targeted desktop OSes by default).
- LibVLCSharp is **not** added in M1 — the player ships in M10.
- `MySqlConnector` and `Dapper` are **not** added in M1 — the database layer ships in M2.
- The legacy `videoindexer-indexer/` solution under the sibling workspace folder is read-only
  reference material; nothing in M1 imports from it.
- We can rename the assembly/namespace from `VideoIndexer` to something else later
  without breaking the persisted `appsettings.json` (because the JSON keys are stable and
  unrelated to assembly name).

## Constraints

- Must target **.NET 8** (LTS).
- Must build and run cleanly on **Windows** in M1; nothing in M1 may use a
  Windows-only API. Filesystem code uses `Path.Combine` and `Environment.SpecialFolder`
  exclusively, so Linux/macOS verification can land in a later milestone without rework.
- Must persist configuration to `Environment.SpecialFolder.ApplicationData` per
  [rebuild.md](../../../projects/rebuild/rebuild.md).
- `TreatWarningsAsErrors=true` and `Nullable=enable` from day one.

## Out of Scope

- **Database access**, schema-revision check, `DatabaseConnector`/`Logon` overlays
  (M2).
- **Library Management**, indexing, hashing, obfuscation runtime (M3).
- **Movies grid, columns, context menu** (M4).
- **Filters, search, filter codes** (M5).
- **Movie editor and tagging** (M6, M7).
- **Preferences dialog UI** (M8) — M1 only models the underlying options and a
  developer-facing theme shortcut.
- **Image/thumbnail pipeline** (M9) and **player/bookmarks** (M10).
- **Localization runtime** — `Appearance.Language` is a settings key only; no resx /
  translation files are added in M1.
- **CI pipeline** and **Linux/macOS verification** — deferred to a later milestone.
- **Application icon and branding assets** — placeholder only.

## Acceptance Criteria

- [ ] `dotnet restore && dotnet build -c Release` from a fresh clone produces zero
      warnings and zero errors.
- [ ] `dotnet run --project src/VideoIndexer.App` opens an empty themed Avalonia main
      window.
- [ ] On first launch, `<ApplicationData>/VideoIndexer/appsettings.json` is created with
      the documented default schema.
- [ ] Editing `Appearance.Theme` to `Light` or `Dark` and relaunching applies the
      chosen variant.
- [ ] `Ctrl+T` cycles `System → Light → Dark → System`, persists the choice, and the
      window repaints immediately.
- [ ] A log file appears under `<ApplicationData>/VideoIndexer/logs/videoindexer-YYYYMMDD.log`
      after launch, and changing `Logging.Verbosity` from 2 to 3 produces visibly more
      entries on the next run.
- [ ] `dotnet test` runs all M1 unit tests green.
- [ ] All four projects target `net8.0`; the App project's `OutputType` is `WinExe`.
- [ ] No package references to `MySqlConnector`, `Dapper`, `LibVLCSharp`, or `FFMpeg*`
      exist anywhere in the solution.
- [ ] [docs/projects/rebuild/milestones/m1-foundation.md](../../../projects/rebuild/milestones/m1-foundation.md)
      exists and conforms to the roadmap's milestone-doc template.

## Testing Strategy

- **Unit tests (xUnit)** for the four service classes listed in step 7. These are pure
  in-memory tests using a temporary directory for the filesystem-touching paths.
- **Manual smoke test** documented in `m1-foundation.md`:
  1. Delete `<ApplicationData>/VideoIndexer/` if present.
  2. `dotnet run --project src/VideoIndexer.App`.
  3. Confirm the settings file and log file are created.
  4. Cycle the theme three times via `Ctrl+T`; relaunch; confirm the last theme
     persisted.
  5. Set `Logging.Verbosity` to `4`, relaunch, confirm verbose entries appear.
- **No integration tests** in M1 — there is nothing to integrate with yet (no DB, no
  ffmpeg, no VLC).

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Avalonia + Generic Host wiring is more fragile than the standard Avalonia template.** | Follow the canonical pattern: build the host first, capture `IServiceProvider`, then `BuildAvaloniaApp().StartWithClassicDesktopLifetime(args)`. Document the bridge in `Program.cs` comments. |
| **`appsettings.json` written by the app overwrites user edits during reload.** | `JsonSettingsService.Save` writes via temp file + atomic replace; the configuration system uses `reloadOnChange: true` for **reading** only. The service never rewrites the file unless `Save` is called explicitly. |
| **`TreatWarningsAsErrors=true` blocks Avalonia's generated XAML code.** | Suppress the well-known Avalonia analyzer warnings on the App project only, never project-wide. Document each suppression. |
| **CommunityToolkit.Mvvm source generators silently fail on certain SDK band mismatches.** | Pin the SDK band via `global.json` and add a CI-equivalent local check (`dotnet --version`) to the README's Build & Run section. |
| **Choosing Serilog now locks us in.** | Code depends only on `ILogger<T>` from `Microsoft.Extensions.Logging.Abstractions`. Swapping to NLog or a future M.E.L file provider requires changing only `LoggingSetup.cs`. |
| **Layered split adds friction for tiny early changes.** | Accept it; the split is the user's chosen layout and prevents a far costlier refactor in M2/M3 when Infrastructure grows the data and indexing layers. |
| **Theme-cycle shortcut overlaps with a future shortcut chosen in M4+.** | `Ctrl+T` is debug-only and gated behind `#if DEBUG` — to be removed when the Preferences dialog (M8) supplies the real toggle. |

AGENT: Planning
STATUS: READY_FOR_PM
