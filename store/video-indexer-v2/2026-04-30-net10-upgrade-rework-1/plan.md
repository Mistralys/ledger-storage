# Plan

## Summary

This plan addresses all four actionable items carried forward from the `.NET 10 Upgrade` synthesis (2026-05-05). In priority order: (1) upgrade Avalonia from 11.2.5 to 11.3.14 to resolve the GHSA-xrw6-gwf8-vvr9 transitive vulnerability and remove the `NuGetAuditMode=direct` debt shield; (2) write `ViewLocator` unit tests that confirm view resolution for all five shell-state content views; (3) correct misleading `// DI constructor` comments in five content view code-behinds; and (4) perform a focused package audit of the framework-agnostic libraries left unchanged during the upgrade cycle.


## Architectural Context

The solution is structured under `src/VideoIndexer.sln` with three production projects and three test projects:

- **`src/VideoIndexer.App/VideoIndexer.App.csproj`** — Avalonia desktop entry point. Owns `Program.cs`, `ViewLocator.cs`, all `Views/`, and all `ViewModels/`. All four Avalonia packages (`Avalonia`, `Avalonia.Desktop`, `Avalonia.Themes.Fluent`, `Avalonia.Fonts.Inter`) are declared here at version `11.2.5`.
- **`src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj`** — Houses Dapper, MySqlConnector, SharpCompress, and the two Serilog sinks (Console + File), all on pinned minor/patch versions.
- **`Directory.Build.props`** — Central build config. Currently contains `<NuGetAuditMode>direct</NuGetAuditMode>` as a debt shield. Removing this property is the mandatory exit criterion of WP-A.

Views and their constructors follow a two-constructor pattern throughout `src/VideoIndexer.App/Views/`:
- **Parameterless constructor** — called by Avalonia's XAML runtime and by the factory registrations in `Program.cs`.
- **Typed ViewModel constructor** — present on all six views but only actively used by DI for `MainWindow`. The five content views use the parameterless constructor in production (via `AddTransient<XView>(_ => new XView())`); the typed constructors are currently labelled with a misleading `// DI constructor — DataContext is injected at construction time.` comment.

`ViewLocator.Build()` resolves view types from DI using `App.Services.GetRequiredService(viewType)` — a static service locator. The five content views are registered as factory-built `AddTransient<>` services. No `ViewLocator` tests currently exist. The existing `VideoIndexer.App.Tests` project references `VideoIndexer.App` but carries no Avalonia headless infrastructure.

**Vulnerability posture:** Avalonia 11.2.5 pulls in `Tmds.DBus.Protocol 0.20.0` (GHSA-xrw6-gwf8-vvr9, HIGH). Avalonia 11.3.14 bumps this transitive to `0.21.3`, resolving the advisory. The 12.x series (latest: 12.0.2, which bumps to 0.92.0) is a separate major-version track with documented breaking changes and is explicitly out of scope here.

**Framework-agnostic packages in `VideoIndexer.Infrastructure`** deferred from the upgrade cycle:

| Package | Current version |
|---|---|
| Dapper | 2.1.35 |
| MySqlConnector | 2.3.7 |
| SharpCompress | 0.38.0 |
| Serilog.Sinks.Console | 6.0.0 |
| Serilog.Sinks.File | 6.0.0 |


## Approach / Architecture

Four independent work packages are sequenced by risk and dependency:

- **WP-A (Avalonia 11.3.14 upgrade)** — Bump the four `Avalonia*` packages in `VideoIndexer.App.csproj` to 11.3.14, verify compilation and tests pass, then remove `<NuGetAuditMode>direct</NuGetAuditMode>` from `Directory.Build.props`. No source code changes beyond version strings are expected given the 11.2→11.3 minor version boundary.
- **WP-B (ViewLocator tests)** — Add `Avalonia.Headless.XUnit 11.3.14` to `VideoIndexer.App.Tests.csproj`. Create `ViewLocatorTests.cs` with a test fixture that builds a minimal `ServiceCollection` with all five content views registered and sets `App.Services` before invoking `ViewLocator.Build()` with an instance of each corresponding ViewModel. Assert the returned control is of the expected view type.
- **WP-C (comment cleanup)** — Mechanical one-liner change in five files: replace the misleading `// DI constructor — DataContext is injected at construction time.` comment with accurate language clarifying that the typed constructor is available for direct instantiation (e.g., test scenarios) and that production startup uses the parameterless constructor.
- **WP-D (package audit)** — Research and apply version upgrades to the five framework-agnostic packages. Evaluate release notes for breaking changes before bumping. Run the full test suite after each package change.

WP-A must complete before WP-B because the headless package version must match the Avalonia runtime version in use. WP-C and WP-D are independent and may be executed in any order.


## Rationale

- **11.3.x instead of 12.x for Avalonia:** The 11.2→11.3 minor version boundary carries minimal API risk compared to the documented breaking changes in the 11.x→12.x major version upgrade. 11.3.14 is confirmed to resolve the vulnerability (release note: "Security: Bump Tmds.DBus.Protocol to 0.21.3"). A 12.x upgrade is a separate, larger sprint with its own breaking-change migration guide and should not be conflated with this security fix.
- **`Avalonia.Headless.XUnit` for ViewLocator tests:** `ViewLocator.Build()` calls `App.Services.GetRequiredService(viewType)` which instantiates Avalonia `UserControl` objects via `new XView()` → `InitializeComponent()`. Avalonia must be initialized before `InitializeComponent()` can run; headless mode is the standard approach for Avalonia unit tests that touch controls.
- **Comment cleanup scope limited to five content views:** `MainWindow`'s typed constructor (line 15 of `MainWindow.axaml.cs`) IS correctly DI-invoked at startup via `builder.Services.AddTransient<MainWindow>()`, so its existing comment is accurate and must not be changed.


## Detailed Steps

### WP-A — Avalonia 11.3.14 Upgrade

1. In `src/VideoIndexer.App/VideoIndexer.App.csproj`, bump all four Avalonia package references from `11.2.5` to `11.3.14`:
   - `Avalonia`
   - `Avalonia.Desktop`
   - `Avalonia.Themes.Fluent`
   - `Avalonia.Fonts.Inter`
2. Run `dotnet build src/VideoIndexer.sln` — expect zero warnings, zero errors.
3. Run `dotnet test src/VideoIndexer.sln` — expect approximately 147 tests passed and 6 skipped (the 6 `[SkippableFact]` integration/live tests in `SpdbConfigRepositoryTests.cs`, `TarXzArchiveExtractorTests.cs`, and `FfmpegProvisionerLiveTests.cs` are expected to report as Skipped in a standard environment without a live database or network connection), 0 failed.
4. Smoke-run the application on .NET 10 to confirm the initial `ProvisioningToolsView` window state still launches successfully.
5. In `Directory.Build.props`, remove the `<NuGetAuditMode>direct</NuGetAuditMode>` line (and its associated XML comment). Full transitive audit is restored.
6. Run `dotnet build src/VideoIndexer.sln` once more after the property removal and confirm no new NuGet audit warnings surface. If any audit warnings appear, investigate and resolve before proceeding.
7. Update `CHANGELOG.md` with an entry for the Avalonia upgrade and the restored audit mode.

### WP-B — ViewLocator ShellState Coverage Tests

1. Add `Avalonia.Headless.XUnit` version `11.3.14` to `tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj`.
2. Add `[assembly: InternalsVisibleTo("VideoIndexer.App.Tests")]` to `src/VideoIndexer.App/Properties/AssemblyInfo.cs` (create the file if it does not exist). This is required because `App.Services` has `internal set` and the test fixture must write to it.
3. Create `tests/VideoIndexer.App.Tests/ViewLocatorTests.cs`.
4. Add a test assembly-level Avalonia headless app setup attribute (`[assembly: AvaloniaTestApplication(typeof(TestApp))]`) and a `TestApp` class with the static entry-point method required by `Avalonia.Headless.XUnit`:

   ```csharp
   public class TestApp
   {
       public static AppBuilder BuildAvaloniaApp()
           => AppBuilder.Configure<Application>()
                        .UseHeadless(new AvaloniaHeadlessOptions { UseHeadlessDrawing = true });
   }
   ```

   `TestApp` does not need to inherit from `Application`.
5. Implement five `[AvaloniaFact]` tests, one per shell-state content view:

   | Test name | ViewModel instance | Expected control type |
   |---|---|---|
   | `Build_ProvisioningToolsViewModel_ReturnsProvisioningToolsView` | `new ProvisioningToolsViewModel(...)` | `ProvisioningToolsView` |
   | `Build_DatabaseConnectorViewModel_ReturnsDatabaseConnectorView` | `new DatabaseConnectorViewModel(...)` | `DatabaseConnectorView` |
   | `Build_LogonViewModel_ReturnsLogonView` | `new LogonViewModel(...)` | `LogonView` |
   | `Build_PasswordSetupViewModel_ReturnsPasswordSetupView` | `new PasswordSetupViewModel(...)` | `PasswordSetupView` |
   | `Build_MainContentViewModel_ReturnsMainContentView` | `new MainContentViewModel()` | `MainContentView` |

6. Each test must:
   a. Build a `ServiceCollection`, register all five content views with factory lambdas identical to those in `Program.cs`.
   b. Build the provider and assign it to `App.Services`.
   c. For the three tests that require a `ShellViewModel` (`DatabaseConnectorViewModel`, `LogonViewModel`, `PasswordSetupViewModel`), first construct a `ShellViewModel` using the fakes already defined in `ShellViewModelTests.cs` (`FakeBootstrapper`, `FakePasswordService`, `FakeExternalToolProvisioner`, and a no-op `viewModelFactory` stub), then pass the resulting instance to the view-model constructor.
   d. Instantiate a `ViewLocator` and call `Build(viewModel)`.
   e. Assert the result is an instance of the expected view type using `FluentAssertions`.
   f. Reset `App.Services` to `null!` in a `finally` block to prevent test-ordering side effects. (`null!` is required because the property type is `IServiceProvider` in a nullable-enabled context.)
7. Run `dotnet test tests/VideoIndexer.App.Tests/` — expect all new tests plus all existing tests to pass.

### WP-C — Content View Constructor Comment Cleanup

In each of the five content view code-behinds, replace the misleading typed-constructor comment:

```csharp
// DI constructor — DataContext is injected at construction time.
```

with:

```csharp
// Available for direct instantiation (e.g., in test scenarios).
// In production, DataContext is set via Avalonia's ContentPresenter
// through the CurrentViewModel binding; the parameterless constructor above is used.
```

**Files to update:**
- `src/VideoIndexer.App/Views/ProvisioningToolsView.axaml.cs`
- `src/VideoIndexer.App/Views/DatabaseConnectorView.axaml.cs`
- `src/VideoIndexer.App/Views/LogonView.axaml.cs`
- `src/VideoIndexer.App/Views/PasswordSetupView.axaml.cs`
- `src/VideoIndexer.App/Views/MainContentView.axaml.cs`

**Do not change** `src/VideoIndexer.App/Views/MainWindow.axaml.cs` — its typed constructor IS used by DI production startup.

### WP-D — Framework-Agnostic Package Audit

1. For each of the five packages, check NuGet for the latest stable release and review release notes for breaking changes:
   - `Dapper` (current: 2.1.35)
   - `MySqlConnector` (current: 2.3.7)
   - `SharpCompress` (current: 0.38.0)
   - `Serilog.Sinks.Console` (current: 6.0.0)
   - `Serilog.Sinks.File` (current: 6.0.0)
2. Apply version bumps that are safe (no breaking API changes relative to the current usage surface in `VideoIndexer.Infrastructure`).
3. After each bump (or as a batch if all are safe), run `dotnet build src/VideoIndexer.sln` and `dotnet test src/VideoIndexer.sln` to confirm no regressions.
4. Update `CHANGELOG.md` with an entry per bumped package.


## Dependencies

- WP-B depends on WP-A completing first (Avalonia.Headless.XUnit must match the Avalonia runtime version).
- WP-C and WP-D are independent of each other and of WP-A/WP-B.


## Required Components

**Modified (existing files):**
- `src/VideoIndexer.App/VideoIndexer.App.csproj` — Avalonia package version bumps (WP-A)
- `Directory.Build.props` — removal of `NuGetAuditMode=direct` (WP-A)
- `tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj` — add Avalonia.Headless.XUnit (WP-B)
- `src/VideoIndexer.App/Views/ProvisioningToolsView.axaml.cs` — comment update (WP-C)
- `src/VideoIndexer.App/Views/DatabaseConnectorView.axaml.cs` — comment update (WP-C)
- `src/VideoIndexer.App/Views/LogonView.axaml.cs` — comment update (WP-C)
- `src/VideoIndexer.App/Views/PasswordSetupView.axaml.cs` — comment update (WP-C)
- `src/VideoIndexer.App/Views/MainContentView.axaml.cs` — comment update (WP-C)
- `src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj` — package version bumps (WP-D)
- `CHANGELOG.md` — release note entries (WP-A, WP-D)

**New (files to create):**
- `src/VideoIndexer.App/Properties/AssemblyInfo.cs` — `[assembly: InternalsVisibleTo("VideoIndexer.App.Tests")]` attribute (WP-B)
- `tests/VideoIndexer.App.Tests/ViewLocatorTests.cs` — ViewLocator coverage tests (WP-B)


## Assumptions

- Avalonia 11.2 → 11.3 introduces no breaking changes to the APIs in use by this application (AppBuilder, UserControl, Window, IDataTemplate, compiled bindings, Fluent theme, Inter font).
- The typed ViewModel constructors on `DatabaseConnectorViewModel`, `LogonViewModel`, and `PasswordSetupViewModel` accept the same dependencies as those used in `ShellViewModelTests.cs` fakes, making it straightforward to construct minimal test instances for WP-B.
- `App.Services` has `internal set` (`App.axaml.cs`). The test fixture can write to it from `VideoIndexer.App.Tests` only after adding `[assembly: InternalsVisibleTo("VideoIndexer.App.Tests")]` to `VideoIndexer.App` (WP-B step 2). This is a minimal, one-line production-code change required to enable testing.
- NuGet audit is clean (no new advisories) after the transitive Tmds.DBus.Protocol is bumped to 0.21.3 via Avalonia 11.3.14.


## Constraints

- The `global.json` SDK pin (`10.0.202`, `rollForward: latestFeature`) must not change.
- `TreatWarningsAsErrors=true` (inherited by all production projects from `Directory.Build.props`) must continue to be satisfied — zero build warnings remain a hard gate.
- The Avalonia 12.x upgrade track is explicitly out of scope for this plan.
- No changes to XAML files, styles, or themes — only `.csproj`, `.cs` source files, and `Directory.Build.props`.


## Out of Scope

- Avalonia 12.x upgrade (separate sprint; has documented breaking changes and requires its own migration assessment).
- `NonDisposingConnectionWrapper` extraction to shared test utilities (flagged as "no action required now" in the synthesis; no usage growth since that note).
- Any new feature work or non-package-related refactoring.


## Acceptance Criteria

- `Directory.Build.props` no longer contains `<NuGetAuditMode>direct</NuGetAuditMode>`.
- `dotnet build src/VideoIndexer.sln` completes with 0 warnings and 0 errors.
- `dotnet test src/VideoIndexer.sln` reports ≥ 152 tests passed (147 existing + 5 new ViewLocator tests), 0 failed, with the 6 skippable integration tests reported as Skipped (not Failed) in a standard environment.
- A `ViewLocatorTests.cs` file exists in `tests/VideoIndexer.App.Tests/` with one test per content view.
- The five content view code-behinds no longer contain the phrase "DI constructor — DataContext is injected at construction time." in their typed-constructor comments.
- `CHANGELOG.md` is updated with entries covering WP-A (Avalonia version + audit mode restoration) and any package bumps from WP-D.


## Testing Strategy

- **WP-A:** Full `dotnet test` after each package version bump; smoke run of the application to the initial `ProvisioningToolsView` window state.
- **WP-B:** `[AvaloniaFact]`-decorated tests using `Avalonia.Headless.XUnit`. Each test independently configures `App.Services` and tears it down, making the suite order-independent. Assertion via `FluentAssertions`'s `Should().BeOfType<T>()` / `Should().BeAssignableTo<T>()`.
- **WP-C:** No functional test required — pure comment change. Review-only confirmation that none of the five view files now contain the old misleading comment text.
- **WP-D:** Full `dotnet test` after each package bump batch; review of breaking-change release notes prior to bumping.


## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Avalonia 11.3.x introduces an API break** affecting compiled bindings or the Fluent theme | Check the Avalonia 11.2→11.3 migration notes and build immediately after bumping; revert to 11.2.5 and document if a breaking change is found |
| **NuGet audit surfaces new advisories** after removing `NuGetAuditMode=direct` beyond Tmds.DBus.Protocol | Investigate each advisory; if any is unresolvable without a major upgrade, re-add a scoped `NuGetAuditMode` suppression with an explicit comment and timeline |
| **`Avalonia.Headless.XUnit` initialization conflicts with xUnit runner isolation** | Use `[assembly: AvaloniaTestApplication]` at the test project level; each `[AvaloniaFact]` test runs inside the headless app context as documented in the Avalonia headless testing guide |
| **ViewLocator tests are fragile due to `App.Services` being static** | Ensure every test resets `App.Services` in a `finally` block; run tests with `--no-parallel` if isolation issues arise |
| **Framework-agnostic package bumps introduce breaking changes** (e.g., Dapper 3.x, MySqlConnector 3.x) | Review changelogs before bumping; defer any package requiring source-level changes to a separate, explicitly-scoped work package |
| **`Serilog.Sinks.File` 7.0.0 major version bump** — a new stable major version (6.0.0 → 7.0.0) is already available on NuGet (published 2025-04-28); major bumps carry higher API-break risk than patch bumps | Review the serilog-sinks-file 7.0.0 release notes and diff the `WriteTo.File()` call signature in `LoggingSetup.cs` before bumping; if breaking changes affect the usage surface, defer to a separate, explicitly-scoped work package |
