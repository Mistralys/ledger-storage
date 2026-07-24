# Plan

## Summary

This sprint addresses all six actionable items and strategic recommendations carried forward from the `2026-04-30-net10-upgrade-rework-1` synthesis (the synthesis artefacts from sprint 1 are not retained on disk; all carried-forward findings are reproduced in full in the Architectural Context section below). The work is partitioned into five focused work packages targeting test infrastructure quality (WP-001 through WP-004) and build system hygiene (WP-005). The expected outcome is a codebase with a discoverable headless test bootstrap, consolidated test fakes eliminating duplication across six files, documented `ViewLocator` edge-case contracts, latent Avalonia theme-resolution risk eliminated in the test project, and all NuGet package versions centrally managed via `Directory.Packages.props`.

---

## Architectural Context

**Solution layout** (root: `c:\Webserver\tools\video-indexer-v2\`):

| Layer | Path |
|---|---|
| Root build props | `Directory.Build.props` |
| Solution file | `src/VideoIndexer.sln` |
| App | `src/VideoIndexer.App/` |
| Core abstractions | `src/VideoIndexer.Core/` |
| Infrastructure | `src/VideoIndexer.Infrastructure/` |
| App unit tests | `tests/VideoIndexer.App.Tests/` |
| Infrastructure tests | `tests/VideoIndexer.Infrastructure.Tests/` |
| Cross-layer tests | `tests/VideoIndexer.Tests/` |

**Relevant existing files:**

- `Directory.Build.props` — shared properties (`TargetFramework=net10.0`, `TreatWarningsAsErrors=true`, `WarningLevel=9999`). No `ManagePackageVersionsCentrally` setting yet.
- `src/VideoIndexer.App/ViewLocator.cs` — `IDataTemplate` implementation. `Build(null)` returns `null` immediately; an unresolvable view type falls back to a `TextBlock`; an unregistered-but-resolvable type causes `GetRequiredService` to throw `InvalidOperationException`.
- `src/VideoIndexer.App/Properties/AssemblyInfo.cs` — `[InternalsVisibleTo("VideoIndexer.App.Tests")]` only.
- `tests/VideoIndexer.App.Tests/ViewLocatorTests.cs` — contains `[assembly: AvaloniaTestApplication(typeof(TestApp))]` and the `TestApp` class inline (lines 13–21), plus four private fake classes. Five `[AvaloniaFact]` tests cover the five registered content views.
- `tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj` — references `Avalonia.Headless.XUnit 11.3.14` but **not** `Avalonia.Themes.Fluent`.

**Fake duplication scope** (broader than synthesis reported):

| Fake | Files containing a private copy |
|---|---|
| `FakeBootstrapper` | `ViewLocatorTests.cs`, `ShellViewModelTests.cs`, `LogonViewModelTests.cs`, `PasswordSetupViewModelTests.cs`, `DatabaseConnectorViewModelTests.cs`, `MainWindowViewModelTests.cs` |
| `FakeExternalToolProvisioner` | `ViewLocatorTests.cs`, `ShellViewModelTests.cs`, `LogonViewModelTests.cs`, `PasswordSetupViewModelTests.cs`, `DatabaseConnectorViewModelTests.cs`, `MainWindowViewModelTests.cs` |
| `FakePasswordService` | `ViewLocatorTests.cs`, `ShellViewModelTests.cs`, `DatabaseConnectorViewModelTests.cs`, `MainWindowViewModelTests.cs` |
| `FakeDatabaseConnectionStore` (various names) | `ViewLocatorTests.cs` (`FakeDatabaseConnectionStore`), `DatabaseConnectorViewModelTests.cs` (`FakeStore`) |

The `ShellViewModelTests.cs` fakes are the most capable (configurable construction, throw modes), making them the authoritative baseline for the consolidated shared variants. `FakeThemeService` appears only in `MainWindowViewModelTests.cs` and is not duplicated; it should remain private to that file.

**Current NuGet package inventory** across all six projects (pre-WP-005):

| Package | Version | Projects |
|---|---|---|
| Avalonia | 11.3.14 | App |
| Avalonia.Desktop | 11.3.14 | App |
| Avalonia.Themes.Fluent | 11.3.14 | App |
| Avalonia.Fonts.Inter | 11.3.14 | App |
| Avalonia.Headless.XUnit | 11.3.14 | App.Tests |
| CommunityToolkit.Mvvm | 8.3.2 | App |
| Dapper | 2.1.72 | Infrastructure |
| FluentAssertions | 6.12.2 | App.Tests, Infrastructure.Tests, Tests |
| Microsoft.Extensions.Configuration | 10.0.7 | Infrastructure |
| Microsoft.Extensions.Configuration.Json | 10.0.7 | App, Infrastructure |
| Microsoft.Extensions.Configuration.Binder | 10.0.7 | Infrastructure |
| Microsoft.Extensions.Hosting | 10.0.7 | App |
| Microsoft.Extensions.Http | 10.0.7 | Infrastructure |
| Microsoft.Extensions.Logging.Abstractions | 10.0.7 | Infrastructure |
| Microsoft.Extensions.Options.ConfigurationExtensions | 10.0.7 | Infrastructure |
| Microsoft.NET.Test.Sdk | 17.12.0 | App.Tests, Infrastructure.Tests, Tests |
| Moq | 4.20.70 | Infrastructure.Tests |
| MySqlConnector | 2.5.0 | Infrastructure |
| Serilog | 4.3.0 | Infrastructure.Tests |
| Serilog.Extensions.Hosting | 10.0.0 | App, Infrastructure |
| Serilog.Sinks.Console | 6.1.1 | Infrastructure |
| Serilog.Sinks.File | 7.0.0 | Infrastructure |
| SharpCompress | 0.47.4 | Infrastructure |
| xunit | 2.9.3 | App.Tests, Infrastructure.Tests, Tests |
| xunit.runner.visualstudio | 2.8.2 | App.Tests, Infrastructure.Tests, Tests |
| Xunit.SkippableFact | 1.4.13 | Infrastructure.Tests |

---

## Approach / Architecture

Five independent work packages are executed in priority order:

1. **WP-001** (trivial csproj patch) — eliminates a latent headless test risk by adding `Avalonia.Themes.Fluent` to the test project before any deeper test refactoring.
2. **WP-002** (file extraction) — separates the Avalonia headless bootstrap from `ViewLocatorTests.cs` into a dedicated `TestBootstrap.cs`, improving discoverability for future contributors.
3. **WP-003** (new tests) — documents `ViewLocator.Build()` null-input and unregistered-view contracts; depends on WP-002 having extracted `TestBootstrap.cs` so that `ViewLocatorTests.cs` is not edited twice.
4. **WP-004** (test refactor) — consolidates all duplicated fakes into `tests/VideoIndexer.App.Tests/TestHelpers/`; depends on WP-002 so `ViewLocatorTests.cs` is in its final form before being refactored again.
5. **WP-005** (build system) — introduces `Directory.Packages.props` at the repo root, enabling NuGet Central Package Management across all six projects. This is done last so that WP-001's version addition is already confirmed correct before being centralised.

No new production source files are created. No third-party libraries beyond those already referenced are introduced.

---

## Rationale

- **WP ordering:** WP-001 is done first (one-line, zero risk) so that subsequent test additions (WP-003) run against a theme-aware headless setup. WP-002 must precede WP-003 and WP-004 to avoid editing `ViewLocatorTests.cs` three separate times in the same sprint.
- **Consolidated fakes baseline:** `ShellViewModelTests.cs` fakes have the richest capability (configurable results, throw-on-call flags). Promoting these to the shared module avoids down-grading test expressiveness in `ShellViewModelTests.cs` while upgrading it everywhere else.
- **`FakeThemeService` exception:** It is not duplicated across files; adding it to `TestHelpers/` would be premature generalisation.
- **`Directory.Packages.props` at repo root:** `Directory.Build.props` already lives at the repo root and is picked up by MSBuild traversal for all six projects. Placing `Directory.Packages.props` at the same level ensures consistent discovery without requiring SDK version workarounds.
- **`ManagePackageVersionsCentrally` in `Directory.Build.props`:** The opt-in property must be set globally (not per-project) so the SDK recognises the central file. Adding it to the existing `Directory.Build.props` is the correct location.

---

## Detailed Steps

### WP-001 — Add `Avalonia.Themes.Fluent` to Test Project

1. Open `tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj`.
2. In the Avalonia `ItemGroup`, add:
   ```xml
   <PackageReference Include="Avalonia.Themes.Fluent" Version="11.3.14" />
   ```
3. Build and run the full test suite to confirm 0 regressions.

---

### WP-002 — Extract `TestApp` to `TestBootstrap.cs`

1. Create `tests/VideoIndexer.App.Tests/TestBootstrap.cs` with content:
   ```csharp
   using Avalonia;
   using Avalonia.Headless;
   using Avalonia.Headless.XUnit;

   [assembly: AvaloniaTestApplication(typeof(VideoIndexer.App.Tests.TestBootstrap))]

   namespace VideoIndexer.App.Tests;

   public sealed class TestBootstrap
   {
       public static AppBuilder BuildAvaloniaApp()
           => AppBuilder.Configure<Application>()
                        .UseHeadless(new AvaloniaHeadlessPlatformOptions { UseHeadlessDrawing = true });
   }
   ```
   > **Note:** The class is renamed from `TestApp` to `TestBootstrap` to make its role immediately clear.

2. In `tests/VideoIndexer.App.Tests/ViewLocatorTests.cs`:
   - Remove the `[assembly: AvaloniaTestApplication(...)]` attribute line.
   - Remove the `TestApp` class definition (lines ~14–21).
   - Remove the `using Avalonia;`, `using Avalonia.Headless;`, and `using Avalonia.Headless.XUnit;` imports that are no longer needed in this file (they move to `TestBootstrap.cs`). Retain `using Avalonia.Headless.XUnit;` only if `[AvaloniaFact]` requires it — `AvaloniaFact` lives in `Avalonia.Headless.XUnit` so it must be kept.

3. Build and run the test suite — the `[AvaloniaTestApplication]` attribute is assembly-scoped, so all `[AvaloniaFact]` tests in the assembly continue to work regardless of which file the attribute is declared in.

---

### WP-003 — Add `ViewLocator.Build()` Edge-Case Tests

Add two new `[AvaloniaFact]` tests to `tests/VideoIndexer.App.Tests/ViewLocatorTests.cs`:

**Test 1 — `Build(null)` returns `null`:**
```csharp
[AvaloniaFact]
public void Build_NullData_ReturnsNull()
{
    var locator = new ViewLocator();
    var result = locator.Build(null);
    result.Should().BeNull();
}
```

**Test 2 — View type resolved but not registered in DI throws `InvalidOperationException`:**
```csharp
[AvaloniaFact]
public void Build_ViewTypeNotRegisteredInServices_ThrowsInvalidOperationException()
{
    // Arrange: DI container with no view registrations
    App.Services = new ServiceCollection().BuildServiceProvider();
    try
    {
        var locator = new ViewLocator();
        var viewModel = new MainContentViewModel();

        // Act & Assert
        locator.Invoking(l => l.Build(viewModel))
               .Should().Throw<InvalidOperationException>();
    }
    finally
    {
        App.Services = null!;
    }
}
```
> This test documents the contract that an unregistered-but-resolvable view type will throw `InvalidOperationException` (from `GetRequiredService`) at navigation time with no fallback, catching silent regression if a new ViewModel is added without a corresponding DI registration.

The `Build(null)` test does not interact with `App.Services` and requires no arrange/teardown.

---

### WP-004 — Consolidate Shared Test Fakes into `TestHelpers/`

#### 4a. Create shared fake files

Create the following files under `tests/VideoIndexer.App.Tests/TestHelpers/`. All files use `namespace VideoIndexer.App.Tests.TestHelpers;`.

**`FakeBootstrapper.cs`**
- Implements `IDatabaseBootstrapper`.
- Constructor accepts `DatabaseBootstrapResult result` (default: `(Ok, 35)`) and `bool throws` flag.
- When `throws` is `true`, `CheckAsync` throws `InvalidOperationException("Simulated connection error.")`.
- This unified class replaces both `FakeBootstrapper` (fixed) and `ThrowingBootstrapper` (throw-only) found in `ShellViewModelTests.cs`.

**`FakeExternalToolProvisioner.cs`**
- Implements `IExternalToolProvisioner`.
- Constructor accepts `Exception? throws = null`.
- `EnsureAsync` returns a fixed `ToolPaths` (`/fake/ffmpeg`, `/fake/ffprobe`, `7.0`). When `throws` is non-null, throws it before returning.
- This unified class replaces the fixed `FakeExternalToolProvisioner` variants across all six files.

**`FakePasswordService.cs`**
- Implements `IPasswordService`.
- Exposes settable properties: `bool HasPassword`, `bool VerifyResult`, `bool ThrowOnVerify`, `bool ThrowOnSet` — matching the richest variant from `ShellViewModelTests.cs`.
- Default values: `HasPassword = true`, `VerifyResult = true`, all throw flags `false`.

**`FakeDatabaseConnectionStore.cs`**
- Implements `IDatabaseConnectionStore`.
- Simple no-op implementation (returns empty list, completes tasks without side effects).
- Replaces the fixed `FakeDatabaseConnectionStore` in `ViewLocatorTests.cs`.
- Does **not** replace `FakeStore` in `DatabaseConnectorViewModelTests.cs` — that fake has meaningful test-specific behaviour (it is the SUT's collaborator under test) and should remain private to that file.

#### 4b. Update consuming test files

For each of the following files, remove the private fake class definitions that are now covered by `TestHelpers/` and add a `using VideoIndexer.App.Tests.TestHelpers;` directive:

- `tests/VideoIndexer.App.Tests/ViewLocatorTests.cs` — remove `FakeBootstrapper`, `FakePasswordService`, `FakeExternalToolProvisioner`, `FakeDatabaseConnectionStore`.
- `tests/VideoIndexer.App.Tests/ShellViewModelTests.cs` — remove `FakeBootstrapper`, `ThrowingBootstrapper`, `FakeExternalToolProvisioner`, `FakePasswordService`; update call sites that construct `new FakeBootstrapper(result)` to the new signature; update call sites that use `ThrowingBootstrapper` to use `new FakeBootstrapper(throws: true)`.
- `tests/VideoIndexer.App.Tests/LogonViewModelTests.cs` — remove `FakeBootstrapper`, `FakeExternalToolProvisioner`. In `BuildShell()`, replace `new FakeBootstrapper()` with `new FakeBootstrapper(new DatabaseBootstrapResult(DatabaseBootstrapStatus.ConfigTableMissing, null))` to preserve the original intent explicitly (the shared fake defaults to `Ok`; no `ConnectAsync` call is made in this file so correctness is unaffected today, but the explicit default guards against future regression).
- `tests/VideoIndexer.App.Tests/PasswordSetupViewModelTests.cs` — remove `FakeBootstrapper`, `FakeExternalToolProvisioner`. In `BuildShell()`, replace `new FakeBootstrapper()` with `new FakeBootstrapper(new DatabaseBootstrapResult(DatabaseBootstrapStatus.ConfigTableMissing, null))` for the same reason as `LogonViewModelTests.cs`.
- `tests/VideoIndexer.App.Tests/DatabaseConnectorViewModelTests.cs` — remove `FakeBootstrapper`, `FakePasswordService`, `FakeExternalToolProvisioner` (retain `FakeStore`). In `BuildShell()`, replace `new FakeBootstrapper()` with `new FakeBootstrapper(new DatabaseBootstrapResult(DatabaseBootstrapStatus.ConfigTableMissing, null))` — the test `ConnectCommand_InvokesShellConnectAsyncExactlyOnce` asserts `shell.Current.Should().Be(ShellState.Error)` based on `ConfigTableMissing`; using the shared fake's default of `(Ok, 35)` would change the transition to `LoggingOn` and break that assertion.
- `tests/VideoIndexer.App.Tests/MainWindowViewModelTests.cs` — remove `FakeBootstrapper`, `FakePasswordService`, `FakeExternalToolProvisioner` (retain `FakeThemeService`). In `BuildShell()` and all inline usages, replace `new FakeBootstrapper { StatusToReturn = status }` with `new FakeBootstrapper(new DatabaseBootstrapResult(status, null))` and `new FakeBootstrapper { StatusToReturn = DatabaseBootstrapStatus.Ok }` with `new FakeBootstrapper(new DatabaseBootstrapResult(DatabaseBootstrapStatus.Ok, null))` — the shared fake uses a constructor parameter, not a settable property.

#### 4c. Verify

Run the full test suite. Expected: 158 tests (156 passed + 2 skipped) plus the 2 new tests from WP-003 (158 total passed, 2 skipped). No regressions.

---

### WP-005 — Adopt Central Package Management (`Directory.Packages.props`)

#### 5a. Enable CPM in `Directory.Build.props`

Add the following property to the existing `<PropertyGroup>` in `Directory.Build.props`:
```xml
<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
```

#### 5b. Create `Directory.Packages.props` at the repo root

Create `Directory.Packages.props` with one `<PackageVersion>` entry per unique package, covering all six projects:

```xml
<Project>

  <!-- ─── Avalonia UI ─────────────────────────────────────────────── -->
  <ItemGroup>
    <PackageVersion Include="Avalonia" Version="11.3.14" />
    <PackageVersion Include="Avalonia.Desktop" Version="11.3.14" />
    <PackageVersion Include="Avalonia.Themes.Fluent" Version="11.3.14" />
    <PackageVersion Include="Avalonia.Fonts.Inter" Version="11.3.14" />
    <PackageVersion Include="Avalonia.Headless.XUnit" Version="11.3.14" />
  </ItemGroup>

  <!-- ─── Community Toolkit ───────────────────────────────────────── -->
  <ItemGroup>
    <PackageVersion Include="CommunityToolkit.Mvvm" Version="8.3.2" />
  </ItemGroup>

  <!-- ─── Data access ─────────────────────────────────────────────── -->
  <ItemGroup>
    <PackageVersion Include="Dapper" Version="2.1.72" />
    <PackageVersion Include="MySqlConnector" Version="2.5.0" />
  </ItemGroup>

  <!-- ─── Microsoft.Extensions ────────────────────────────────────── -->
  <ItemGroup>
    <PackageVersion Include="Microsoft.Extensions.Configuration" Version="10.0.7" />
    <PackageVersion Include="Microsoft.Extensions.Configuration.Binder" Version="10.0.7" />
    <PackageVersion Include="Microsoft.Extensions.Configuration.Json" Version="10.0.7" />
    <PackageVersion Include="Microsoft.Extensions.Hosting" Version="10.0.7" />
    <PackageVersion Include="Microsoft.Extensions.Http" Version="10.0.7" />
    <PackageVersion Include="Microsoft.Extensions.Logging.Abstractions" Version="10.0.7" />
    <PackageVersion Include="Microsoft.Extensions.Options.ConfigurationExtensions" Version="10.0.7" />
  </ItemGroup>

  <!-- ─── Serilog ──────────────────────────────────────────────────── -->
  <ItemGroup>
    <PackageVersion Include="Serilog" Version="4.3.0" />
    <PackageVersion Include="Serilog.Extensions.Hosting" Version="10.0.0" />
    <PackageVersion Include="Serilog.Sinks.Console" Version="6.1.1" />
    <PackageVersion Include="Serilog.Sinks.File" Version="7.0.0" />
  </ItemGroup>

  <!-- ─── Compression ──────────────────────────────────────────────── -->
  <ItemGroup>
    <!-- SharpCompress is in 0.x space: semver guarantees do not apply across minor versions.
         Treat every bump as requiring explicit build + test validation. -->
    <PackageVersion Include="SharpCompress" Version="0.47.4" />
  </ItemGroup>

  <!-- ─── Test infrastructure ─────────────────────────────────────── -->
  <ItemGroup>
    <PackageVersion Include="FluentAssertions" Version="6.12.2" />
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.12.0" />
    <PackageVersion Include="Moq" Version="4.20.70" />
    <PackageVersion Include="xunit" Version="2.9.3" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="2.8.2" />
    <PackageVersion Include="Xunit.SkippableFact" Version="1.4.13" />
  </ItemGroup>

</Project>
```

#### 5c. Strip `Version` attributes from all `.csproj` files

Remove the `Version="..."` attribute from every `<PackageReference>` element in:

- `src/VideoIndexer.App/VideoIndexer.App.csproj`
- `src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj`
- `tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj`
- `tests/VideoIndexer.Infrastructure.Tests/VideoIndexer.Infrastructure.Tests.csproj`
- `tests/VideoIndexer.Tests/VideoIndexer.Tests.csproj`

`src/VideoIndexer.Core/VideoIndexer.Core.csproj` has no `<PackageReference>` elements and requires no changes.

#### 5d. Verify

Run `dotnet restore` at the repo root and confirm all packages resolve without errors. Then `dotnet build src/VideoIndexer.sln` to confirm 0 errors. Run the full test suite one final time.

---

## Dependencies

- WP-002 must complete before WP-003 and WP-004 (all three touch `ViewLocatorTests.cs`).
- WP-001 must complete before WP-005 (WP-005 centralises the version added by WP-001).
- WP-003 and WP-004 may execute in parallel after WP-002, but editing the same file (`ViewLocatorTests.cs`) means sequential execution is safer — WP-003 first, WP-004 second.

Execution order: **WP-001 → WP-002 → WP-003 → WP-004 → WP-005**

---

## Required Components

### New files
- `tests/VideoIndexer.App.Tests/TestBootstrap.cs` (WP-002)
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeBootstrapper.cs` (WP-004)
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeExternalToolProvisioner.cs` (WP-004)
- `tests/VideoIndexer.App.Tests/TestHelpers/FakePasswordService.cs` (WP-004)
- `tests/VideoIndexer.App.Tests/TestHelpers/FakeDatabaseConnectionStore.cs` (WP-004)
- `Directory.Packages.props` (WP-005)

### Modified files
- `tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj` (WP-001, WP-005)
- `tests/VideoIndexer.App.Tests/ViewLocatorTests.cs` (WP-002, WP-003, WP-004)
- `tests/VideoIndexer.App.Tests/ShellViewModelTests.cs` (WP-004)
- `tests/VideoIndexer.App.Tests/LogonViewModelTests.cs` (WP-004)
- `tests/VideoIndexer.App.Tests/PasswordSetupViewModelTests.cs` (WP-004)
- `tests/VideoIndexer.App.Tests/DatabaseConnectorViewModelTests.cs` (WP-004)
- `tests/VideoIndexer.App.Tests/MainWindowViewModelTests.cs` (WP-004)
- `src/VideoIndexer.App/VideoIndexer.App.csproj` (WP-005)
- `src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj` (WP-005)
- `tests/VideoIndexer.Infrastructure.Tests/VideoIndexer.Infrastructure.Tests.csproj` (WP-005)
- `tests/VideoIndexer.Tests/VideoIndexer.Tests.csproj` (WP-005)
- `Directory.Build.props` (WP-005)

---

## Assumptions

- The `ProgressReportingProvisioner` in `ProvisioningToolsViewModelTests.cs` is a specialised fake with test-specific behaviour distinct from `FakeExternalToolProvisioner`. It should remain private to that file and is excluded from WP-004.
- `FakeStore` in `DatabaseConnectorViewModelTests.cs` has meaningful test-specific state/behaviour; it remains private to that file.
- `FakeThemeService` in `MainWindowViewModelTests.cs` is not duplicated and remains private.
- The Avalonia headless platform does not require a full theme to be loaded in order for `ViewLocator.Build()` tests to pass. Adding `Avalonia.Themes.Fluent` (WP-001) is a precautionary measure for deeper headless tests added in the future.
- `dotnet restore` and `dotnet build` commands are run from `c:\Webserver\tools\video-indexer-v2\` (the repo root).

---

## Constraints

- `TreatWarningsAsErrors` is `true` in `Directory.Build.props` for production projects; test projects override to `false`. Do not remove or weaken this for production projects.
- `SharpCompress` must not be bumped in this sprint — the `0.x` versioning note in `Directory.Packages.props` is documentary only.
- Avalonia 12.x migration is explicitly out of scope.
- No new production dependencies are introduced.

---

## Out of Scope

- Avalonia 12.x migration planning.
- Package version bumps for any package not directly required by the work packages.
- Creation of a separate `VideoIndexer.TestHelpers` assembly — a `TestHelpers/` folder within `VideoIndexer.App.Tests` is sufficient at current scale.
- Scheduled package audit tooling or CI pipeline changes.
- Documentation updates to `README.md` or `CHANGELOG.md` beyond what release engineering may produce as a standard pipeline stage.

---

## Acceptance Criteria

- **WP-001:** `dotnet list package` for `VideoIndexer.App.Tests.csproj` shows `Avalonia.Themes.Fluent 11.3.14`. Build is clean.
- **WP-002:** `TestBootstrap.cs` exists at `tests/VideoIndexer.App.Tests/TestBootstrap.cs`. `ViewLocatorTests.cs` no longer contains the `TestApp` class or `[assembly: AvaloniaTestApplication]`. All existing `[AvaloniaFact]` tests pass.
- **WP-003:** Two new tests (`Build_NullData_ReturnsNull`, `Build_ViewTypeNotRegisteredInServices_ThrowsInvalidOperationException`) exist in `ViewLocatorTests.cs` and pass. Total passing tests rises from 156 to 158.
- **WP-004:** `TestHelpers/` directory contains four fake files. No private fake class in `ViewLocatorTests.cs`, `ShellViewModelTests.cs`, `LogonViewModelTests.cs`, `PasswordSetupViewModelTests.cs`, `DatabaseConnectorViewModelTests.cs`, or `MainWindowViewModelTests.cs` duplicates a class now in `TestHelpers/`. Full test suite passes (158 passed, 2 skipped, 0 failed).
- **WP-005:** `Directory.Packages.props` exists at the repo root. No `<PackageReference>` element in any `.csproj` file contains a `Version` attribute. `dotnet restore` and `dotnet build src/VideoIndexer.sln` succeed with 0 errors, 0 warnings. Full test suite passes.

---

## Testing Strategy

Each work package includes a mandatory build-and-test verification gate:

- After WP-001: `dotnet build src/VideoIndexer.sln` + `dotnet test`.
- After WP-002: `dotnet test` — confirm all 156 tests still pass.
- After WP-003: `dotnet test` — confirm 158 tests pass (net +2).
- After WP-004: `dotnet test` — confirm 158 tests pass, 0 regressions.
- After WP-005: `dotnet restore` then `dotnet build src/VideoIndexer.sln` then `dotnet test` — full green board.

No new test frameworks or tooling are required beyond what is already in the solution.

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **Assembly-attribute conflict after WP-002** — if `[assembly: AvaloniaTestApplication]` is accidentally left in `ViewLocatorTests.cs` and also present in `TestBootstrap.cs`, the compiler will emit an error. | Remove the attribute from `ViewLocatorTests.cs` in the same commit that creates `TestBootstrap.cs`. Verify with a build before proceeding to WP-003. |
| **Fake capability regression in WP-004** — the consolidated `FakeBootstrapper` must support both fixed-result and throw modes; an under-specified shared fake could silently break `ShellViewModelTests.cs` scenarios. | Use `ShellViewModelTests.cs` fakes as the authoritative implementation baseline. Validate all 158 tests after WP-004. |
| **CPM restore failure in WP-005** — any `<PackageReference>` that still has a `Version` attribute will cause MSBuild to error with `NU1009` (version specified both centrally and locally). | After stripping `Version` attributes, run `dotnet restore` first and fix `NU1009` errors before attempting `dotnet build`. |
| **CPM and `PrivateAssets`/`IncludeAssets` on test runner** — `xunit.runner.visualstudio` uses `IncludeAssets` and `PrivateAssets` metadata. CPM preserves these attributes; only `Version` is removed. | Verify that `xunit.runner.visualstudio` entries in all three test projects retain their `IncludeAssets`/`PrivateAssets` child elements after the strip. |
| **`SharpCompress` 0.x false confidence** — centralising the version in `Directory.Packages.props` without documentation could mislead a future maintainer into treating it as semver-safe. | Include the inline comment in `Directory.Packages.props` (as shown in WP-005b) to make the 0.x caveat visible at the point of version declaration. |
| **`FakeBootstrapper` default-value shift in WP-004** — the shared `FakeBootstrapper` defaults to `(Ok, 35)`, whereas every local fake it replaces (except `ShellViewModelTests.cs`) defaults to `ConfigTableMissing`. Any consuming test that constructs `new FakeBootstrapper()` and then calls `ConnectAsync` (or expects a `ConfigTableMissing`-driven path) will silently change behaviour. | Follow the per-file call-site migration guidance in step 4b. After the refactor, run the full test suite (`dotnet test`) and confirm 158 passed, 2 skipped, 0 failed before proceeding to WP-005. |
