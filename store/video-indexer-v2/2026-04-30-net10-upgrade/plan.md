# Plan

## Summary

The `video-indexer-v2` solution was bootstrapped against `net8.0` even though the
intended long-term target framework was `net10.0`. The build host already runs the
.NET 10 SDK exclusively (10.0.202) and `global.json` rolls forward to `latestMajor`,
so the project happens to build today — but the produced binaries are .NET 8
assemblies that depend on a runtime nobody installs, and one test project
(`VideoIndexer.Infrastructure.Tests`) has already drifted to `net10.0`, creating an
inconsistent TFM matrix flagged in the ffmpeg-provisioning synthesis (D-6).

This plan realigns every production and test project on `net10.0`, updates the
runtime-version-bound NuGet dependencies (`Microsoft.Extensions.*`,
`Serilog.Extensions.Hosting`) to the matching 10.x band, unifies test-project
package versions, retires the now-pointless `<RollForward>` shims, repins
`global.json`, and refreshes forward-facing documentation so future agents see
.NET 10 as the canonical target. Historical plan / synthesis / milestone documents
are left untouched as a faithful record.

## Architectural Context

Relevant files discovered in the repository:

- [global.json](global.json) — currently pins SDK band `8.0.0` with
  `rollForward: latestMajor`.
- [Directory.Build.props](Directory.Build.props) — solution-wide MSBuild defaults
  (`LangVersion=latest`, `Nullable=enable`, `ImplicitUsings=enable`,
  `TreatWarningsAsErrors=true`, `WarningLevel=9999`). **Does not** define a shared
  `TargetFramework`; each project sets its own.
- Production projects (currently `net8.0`):
  - [src/VideoIndexer.Core/VideoIndexer.Core.csproj](src/VideoIndexer.Core/VideoIndexer.Core.csproj)
    — pure model/abstraction project, no NuGet refs.
  - [src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj](src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj)
    — references `Microsoft.Extensions.*` 8.0.0 (except
    `Microsoft.Extensions.Logging.Abstractions` which is already at 8.0.2),
    `Serilog.Extensions.Hosting` 8.0.0, plus framework-agnostic packages
    (`Dapper`, `MySqlConnector`, `SharpCompress`, `Serilog.Sinks.*`).
  - [src/VideoIndexer.App/VideoIndexer.App.csproj](src/VideoIndexer.App/VideoIndexer.App.csproj)
    — `WinExe` Avalonia front-end on `Avalonia` 11.2.5, plus
    `Microsoft.Extensions.Configuration.Json` 8.0.1, `Microsoft.Extensions.Hosting`
    8.0.1, `Serilog.Extensions.Hosting` 8.0.0, `CommunityToolkit.Mvvm` 8.3.2.
- Test projects (currently mixed):
  - [tests/VideoIndexer.Tests/VideoIndexer.Tests.csproj](tests/VideoIndexer.Tests/VideoIndexer.Tests.csproj)
    — `net8.0`; xunit 2.9.3 / Test.Sdk 17.12.0 / FluentAssertions 6.12.2.
  - [tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj](tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj)
    — `net8.0`; xunit 2.9.3 / Test.Sdk 17.12.0 / FluentAssertions 6.12.2;
    `TreatWarningsAsErrors=false`.
  - [tests/VideoIndexer.Infrastructure.Tests/VideoIndexer.Infrastructure.Tests.csproj](tests/VideoIndexer.Infrastructure.Tests/VideoIndexer.Infrastructure.Tests.csproj)
    — **already** `net10.0`; older xunit 2.7.0 / Test.Sdk 17.9.0; also pulls
    `Moq` 4.20.70, `Serilog` 4.0.0, `Xunit.SkippableFact` 1.4.13.
- Forward-facing documentation that still advertises .NET 8:
  - [README.md](README.md) (lines 3, 9 — "Built with Avalonia UI, .NET 8, …",
    ".NET 8 SDK" prerequisite).
  - [docs/projects/rebuild/rebuild.md](docs/projects/rebuild/rebuild.md) (line 51 —
    "Avalonia UI + LibVLCSharp on .NET 8+").
  - [docs/projects/rebuild/milestones/m1-foundation.md](docs/projects/rebuild/milestones/m1-foundation.md)
    (lines 10, 44, 60 — establishment goals and acceptance criteria).
- Historical plan / synthesis documents under
  `docs/agents/plans/2026-04-27-m1-foundation/`,
  `docs/agents/plans/2026-04-28-ffmpeg-provisioning/`,
  `docs/agents/plans/2026-04-28-m3-library-indexing/` — left untouched per user
  decision (faithful historical record).
- Build-host SDK availability: only `10.0.202` is installed; .NET 8 runtime is
  absent. The current build only succeeds because `global.json` rolls forward and
  `<RollForward>LatestMajor</RollForward>` is set on test projects.

## Approach / Architecture

A coordinated, **single-PR-style** TFM bump across the solution. Because no
production code currently depends on .NET-9- or .NET-10-only APIs and Avalonia
11.2.5 is forward-compatible with later .NET runtimes, the upgrade is
predominantly a metadata change accompanied by a synchronized NuGet bump for the
runtime-version-bound `Microsoft.Extensions.*` and `Serilog.Extensions.Hosting`
families.

Three structural improvements are included to avoid leaving the codebase in a
half-migrated state:

1. **Promote `TargetFramework` to `Directory.Build.props`.** Today every project
   restates `<TargetFramework>net8.0</TargetFramework>` — this is exactly what
   produced the inconsistency (one test project drifted to `net10.0` unnoticed).
   Setting `<TargetFramework>net10.0</TargetFramework>` once in
   `Directory.Build.props` removes the drift surface; individual projects can
   still override (none currently need to).
2. **Repin `global.json` to the installed 10.0.2xx band** with
   `rollForward: latestFeature` so the build is reproducible against the actual
   intended runtime, not whichever future major happens to be installed.
3. **Retire `<RollForward>LatestMajor</RollForward>`** from
   `VideoIndexer.Tests` and `VideoIndexer.App.Tests` — it was a workaround for the
   "no .NET 8 runtime installed" gap and becomes pointless once the projects
   target `net10.0` natively.

Test-project package versions are unified on the newer bar already used by
`VideoIndexer.Tests` and `VideoIndexer.App.Tests` (xunit 2.9.3, Test.Sdk 17.12.0).
`Moq` and `Xunit.SkippableFact` remain only on `VideoIndexer.Infrastructure.Tests`
because they are actually used there; they are not blanket-added elsewhere. The
stray `Serilog` 4.0.0 direct reference in `VideoIndexer.Infrastructure.Tests` is
verified for actual usage and removed if redundant (Serilog comes transitively
through `Serilog.Extensions.Hosting`).

## Rationale

- **Why a single coordinated bump instead of project-by-project?** The pain point
  is the inconsistent TFM matrix (D-6). Migrating one project at a time would
  prolong that inconsistency and complicate cross-project test/runtime
  dependencies. The bump itself is mechanically tiny.
- **Why `Directory.Build.props` for `TargetFramework`?** Same reason MSBuild
  conventions like `LangVersion` are centralized there: it makes accidental
  drift impossible to introduce silently. The current state is direct evidence
  of why centralization matters.
- **Why bump `Microsoft.Extensions.*` to 10.x?** Mixing an `Extensions.Hosting`
  8.0.x package on a `net10.0` TFM works (NuGet picks the netstandard2.0 asset)
  but pulls in transitive 8.0 dependencies and surfaces NU1903/NU1605 noise on
  some restores; aligning the band with the runtime is the standard guidance.
- **Why `latestFeature` instead of `latestMajor` for `global.json` rollForward?**
  The original intent was "build with .NET 10". `latestMajor` would silently
  accept .NET 11+, reproducing the same class of mistake this plan exists to
  fix. `latestFeature` allows patch/feature SDK rolls within 10.x (e.g.
  10.0.202 → 10.0.300) without slipping a major.
- **Why leave Avalonia at 11.2.5?** It is forward-compatible and not the subject
  of this request. Bundling an Avalonia bump would expand scope, risk regressions
  in compiled-bindings behaviour, and obscure which change caused any failure.
  An Avalonia bump can be a future, isolated work package.

## Detailed Steps

1. **Repin SDK in `global.json`.**
   - Replace `"version": "8.0.0"` with `"version": "10.0.202"`.
   - Replace `"rollForward": "latestMajor"` with `"rollForward": "latestFeature"`.

2. **Centralize `TargetFramework` in `Directory.Build.props`.**
   - Add `<TargetFramework>net10.0</TargetFramework>` to the existing
     `<PropertyGroup>` in [Directory.Build.props](Directory.Build.props).

3. **Remove per-project `<TargetFramework>` lines.**
   - Delete the `<TargetFramework>net8.0</TargetFramework>` line from each of:
     - [src/VideoIndexer.Core/VideoIndexer.Core.csproj](src/VideoIndexer.Core/VideoIndexer.Core.csproj)
     - [src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj](src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj)
     - [src/VideoIndexer.App/VideoIndexer.App.csproj](src/VideoIndexer.App/VideoIndexer.App.csproj)
     - [tests/VideoIndexer.Tests/VideoIndexer.Tests.csproj](tests/VideoIndexer.Tests/VideoIndexer.Tests.csproj)
     - [tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj](tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj)
   - Delete the `<TargetFramework>net10.0</TargetFramework>` line from
     [tests/VideoIndexer.Infrastructure.Tests/VideoIndexer.Infrastructure.Tests.csproj](tests/VideoIndexer.Infrastructure.Tests/VideoIndexer.Infrastructure.Tests.csproj)
     (now redundant).

4. **Retire the `<RollForward>` shims.**
   - Remove `<RollForward>LatestMajor</RollForward>` from
     `VideoIndexer.Tests.csproj` and `VideoIndexer.App.Tests.csproj`.

5. **Bump runtime-version-bound NuGet packages to the 10.x band.**
   For every package listed below, the Engineer must query `nuget.org` at
   implementation time and pin the **highest stable 10.x version** available
   (do not hard-pin to `10.0.0` — the .NET 10 GA shipped months before this
   plan and patch bands almost certainly exist).

   In `VideoIndexer.Infrastructure.csproj`:
   - `Microsoft.Extensions.Configuration` → highest stable 10.x
   - `Microsoft.Extensions.Configuration.Json` → highest stable 10.x
   - `Microsoft.Extensions.Configuration.Binder` → highest stable 10.x
   - `Microsoft.Extensions.Http` → highest stable 10.x
   - `Microsoft.Extensions.Options.ConfigurationExtensions` → highest stable 10.x
   - `Microsoft.Extensions.Logging.Abstractions` → highest stable 10.x
   - `Serilog.Extensions.Hosting` → highest stable that declares `net10.0` or
     `netstandard2.0` compatibility (currently 9.0.0; pick the newest available).

   In `VideoIndexer.App.csproj`:
   - `Microsoft.Extensions.Configuration.Json` → highest stable 10.x
   - `Microsoft.Extensions.Hosting` → highest stable 10.x
   - `Serilog.Extensions.Hosting` → same version chosen above.

6. **Unify all shared package versions across production and test projects.**
   General principle: any package referenced by both a production project and a
   test project (or by multiple projects of either kind) must resolve to the
   same version. Bump to the **highest stable** version when aligning, and pull
   the same version into every project that references it — including
   framework-agnostic packages.

   Concrete actions for the current state:
   - In `VideoIndexer.Infrastructure.Tests.csproj`:
     - `Microsoft.NET.Test.Sdk` 17.9.0 → highest stable (≥ 17.12.0; align with
       the other two test projects, bumping all three together if a newer
       stable exists).
     - `xunit` 2.7.0 → highest stable (≥ 2.9.3; align all three test projects).
     - `xunit.runner.visualstudio` 2.5.7 → highest stable (≥ 2.8.2; align all
       three test projects).
     - `Moq` 4.20.70 → highest stable (kept; used by this project).
     - `Xunit.SkippableFact` 1.4.13 → highest stable (kept; used by this
       project).
     - **Retain** the `Serilog` direct reference but bump to the highest stable
       version that matches the `Serilog` transitively pulled in by
       `Serilog.Extensions.Hosting` (chosen in step 5). Confirmed used by
       `Logging/LoggingSetupTests.cs` (`using Serilog;` / `using Serilog.Events;`).
     - Do **not** add `FluentAssertions` — no usage found in this project's
       test code.
     - Add `<IsTestProject>true</IsTestProject>` to the `<PropertyGroup>` — the
       other two test projects already declare it; this file is missing it.
   - In `VideoIndexer.Tests.csproj` and `VideoIndexer.App.Tests.csproj`:
     - Bump `Microsoft.NET.Test.Sdk`, `xunit`, `xunit.runner.visualstudio`,
       `FluentAssertions` to the highest stable versions chosen above (so all
       three test projects share an identical test-runner stack).
   - Framework-agnostic production packages (`Dapper`, `MySqlConnector`,
     `SharpCompress`, `Serilog.Sinks.Console`, `Serilog.Sinks.File`,
     `CommunityToolkit.Mvvm`) may also be bumped to their highest stable
     versions when this clarifies version alignment; deep upgrades that imply
     behavioral changes (e.g. major-version jumps with documented breaking
     changes) are out of scope and should be deferred to a dedicated plan.
   - Avalonia stays at 11.2.5 (explicit user decision; see Out of Scope).

   **Align `TreatWarningsAsErrors` posture across the three test projects.**
   Today `VideoIndexer.Tests` inherits `true` while `VideoIndexer.App.Tests`
   and `VideoIndexer.Infrastructure.Tests` set it to `false`. Pick one
   posture and apply it uniformly — recommended: keep `false` for all three
   test projects (xunit/Moq analyzers periodically emit informational
   warnings that do not warrant blocking the build), while production
   projects keep `true` from `Directory.Build.props`.

7. **Verify build & tests on .NET 10.**
   - Wipe stale build outputs first: `dotnet clean src/VideoIndexer.sln` and
     manually remove any leftover `bin/Debug/net8.0/` and `obj/` folders so the
     restore graph is rebuilt cleanly against the new TFM.
   - `dotnet restore src/VideoIndexer.sln`
   - `dotnet build src/VideoIndexer.sln -c Debug` — must pass with
     `TreatWarningsAsErrors=true` honored where currently set (production
     projects + `VideoIndexer.Tests`); test projects that opt out continue to
     emit warnings without failing.
   - As a brand-new project without external consumers, the baseline
     expectation is **zero warnings** across the solution. Any warnings that
     surface (typically `[Obsolete]` notices from the `Microsoft.Extensions.*`
     8 → 10 jump) must be fixed in-place. Warnings that genuinely cannot be
     fixed without scope creep (e.g. requiring an Avalonia bump) are
     documented in the synthesis with a tracked follow-up.
   - `dotnet test tests/VideoIndexer.Tests --no-build`
   - `dotnet test tests/VideoIndexer.App.Tests --no-build`
   - `dotnet test tests/VideoIndexer.Infrastructure.Tests --no-build`
   - Sanity-check the App: `dotnet run --project src/VideoIndexer.App` opens the
     shell window without runtime-binding errors.
   - Inspect a build output (e.g. `src/VideoIndexer.App/bin/Debug/net10.0/`) to
     confirm the per-TFM folder is now `net10.0`.

8. **Refresh forward-facing documentation.**
   - [README.md](README.md):
     - Line 3 — change "Built with Avalonia UI, .NET 8, …" → ".NET 10".
     - Line 9 — change ".NET 8 SDK" prerequisite + link → ".NET 10 SDK"
       (`https://dotnet.microsoft.com/download/dotnet/10.0`). Replace the
       existing `rollForward` aside with text equivalent to: _"`global.json`
       pins the 10.0.2xx feature band with `rollForward: latestFeature`,
       accepting any 10.x SDK ≥ 10.0.202 within the same major"_. The
       Engineer may polish wording but must not widen the rollForward
       semantics back to `latestMajor`.
   - [docs/projects/rebuild/rebuild.md](docs/projects/rebuild/rebuild.md):
     - Line 51 — change "Avalonia UI + LibVLCSharp on .NET 8+" → ".NET 10+".
   - [docs/projects/rebuild/milestones/m1-foundation.md](docs/projects/rebuild/milestones/m1-foundation.md):
     - Line 10 — "Avalonia/.NET 8 solution" → "Avalonia/.NET 10 solution".
     - Line 44 — acceptance criterion "All projects target `net8.0`" →
       "All projects target `net10.0`".
     - Line 60 — risk row about ".NET 8 runtime missing on build machine" is now
       resolved; either delete the row or rewrite it to note historical context.
   - Add a brief entry to the project changelog (path determined during
     implementation — there is no top-level `CHANGELOG.md` in `video-indexer-v2`
     today; if absent, skip rather than invent a new convention).

## Dependencies

- .NET 10 SDK already installed on the build host (verified: `10.0.202`).
- NuGet feed `nuget.org` reachable to resolve the 10.x packages.
- No other tooling (CI, container images, deployment scripts) currently
  depends on `net8.0` — confirmed by absence of `Dockerfile`, GitHub Actions,
  or similar in the repository at planning time.

## Required Components

Files modified (all existing):

- [global.json](global.json)
- [Directory.Build.props](Directory.Build.props)
- [src/VideoIndexer.Core/VideoIndexer.Core.csproj](src/VideoIndexer.Core/VideoIndexer.Core.csproj)
- [src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj](src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj)
- [src/VideoIndexer.App/VideoIndexer.App.csproj](src/VideoIndexer.App/VideoIndexer.App.csproj)
- [tests/VideoIndexer.Tests/VideoIndexer.Tests.csproj](tests/VideoIndexer.Tests/VideoIndexer.Tests.csproj)
- [tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj](tests/VideoIndexer.App.Tests/VideoIndexer.App.Tests.csproj)
- [tests/VideoIndexer.Infrastructure.Tests/VideoIndexer.Infrastructure.Tests.csproj](tests/VideoIndexer.Infrastructure.Tests/VideoIndexer.Infrastructure.Tests.csproj)
- [README.md](README.md)
- [docs/projects/rebuild/rebuild.md](docs/projects/rebuild/rebuild.md)
- [docs/projects/rebuild/milestones/m1-foundation.md](docs/projects/rebuild/milestones/m1-foundation.md)

No new files are introduced (besides this plan and its eventual synthesis).

## Assumptions

- Avalonia 11.2.5 runs without modification on the .NET 10 runtime; the user has
  confirmed bumping Avalonia is **out of scope** for this work.
- No production source file currently uses an API removed or behavior-changed
  between `net8.0` and `net10.0` (none was identified during planning; if the
  build surfaces one, it is treated as an in-scope minor fix because it is a
  direct consequence of the TFM bump).
- The `Microsoft.Extensions.*` 10.0.0 GA packages are available on `nuget.org`
  (true at the planning date, .NET 10 having shipped).
- No external consumer outside the repository depends on `net8.0`-flavoured
  `VideoIndexer.Core`/`Infrastructure` assemblies.

## Constraints

- `TreatWarningsAsErrors=true` (from `Directory.Build.props`) and
  `WarningLevel=9999` are **not** relaxed by this plan; the bumped packages must
  build cleanly under those settings.
- `global.json` must continue to produce reproducible builds; rollForward is
  narrowed, not widened.
- Historical plan / synthesis / milestone documents dated before 2026-04-30 are
  **not** edited.

## Out of Scope

- Avalonia version upgrade (e.g. 11.2.5 → 11.3.x).
- Any C# language-version exploitation of new .NET 10 BCL APIs.
- **Major-version** jumps for framework-agnostic packages (`Dapper`,
  `MySqlConnector`, `SharpCompress`, `CommunityToolkit.Mvvm`,
  `Serilog.Sinks.*`) that imply documented breaking changes. Within-major
  alignment to the highest stable version is in scope per step 6.
- Introducing a `Directory.Packages.props` / Central Package Management.
- CI configuration, Dockerfiles, or release-pipeline changes (none exist yet).
- Refactoring or behavior changes in production code beyond what the TFM bump
  forces.

## Acceptance Criteria

- [ ] `global.json` pins SDK band `10.0.202` with `rollForward: latestFeature`.
- [ ] `Directory.Build.props` defines `<TargetFramework>net10.0</TargetFramework>`
      in its single `<PropertyGroup>`.
- [ ] No `.csproj` in the solution contains `<TargetFramework>` after this work
      (each inherits from `Directory.Build.props`); confirmed by
      `Select-String -Path src,tests -Pattern '<TargetFramework' -Recurse` returning
      zero matches.
- [ ] No `.csproj` references `net8.0`; confirmed by repo-wide grep returning
      zero matches in `src/` and `tests/`.
- [ ] `<RollForward>LatestMajor</RollForward>` is removed from both
      `VideoIndexer.Tests.csproj` and `VideoIndexer.App.Tests.csproj`.
- [ ] All `Microsoft.Extensions.*` packages in `VideoIndexer.Infrastructure` and
      `VideoIndexer.App` are on a 10.x version.
- [ ] All three test projects use the same versions of `Microsoft.NET.Test.Sdk`,
      `xunit`, and `xunit.runner.visualstudio`.
- [ ] `dotnet build src/VideoIndexer.sln -c Debug` succeeds with no warnings.
      Project is greenfield with no external consumers; warnings that genuinely
      cannot be fixed without out-of-scope work (e.g. Avalonia bump) are
      enumerated in the synthesis with a tracked follow-up rather than left
      silent.
- [ ] All three test projects share an identical `TreatWarningsAsErrors`
      posture (recommended: `false` on all three, `true` inherited from
      `Directory.Build.props` on production projects).
- [ ] All three test projects share identical versions of
      `Microsoft.NET.Test.Sdk`, `xunit`, `xunit.runner.visualstudio`, and
      (where referenced) `FluentAssertions`.
- [ ] Any package referenced by more than one project resolves to the same
      version everywhere it is referenced.
- [ ] `dotnet test` succeeds on each of the three test projects.
- [ ] Build outputs land in `bin/Debug/net10.0/` (not `net8.0/`) for every
      project.
- [ ] `README.md`, `docs/projects/rebuild/rebuild.md`, and
      `docs/projects/rebuild/milestones/m1-foundation.md` no longer advertise
      `.NET 8` / `net8.0` as the project target.
- [ ] Historical plan documents under `docs/agents/plans/2026-04-2[7-8]-*/`
      remain unchanged.

## Testing Strategy

- **Build verification**: full solution `dotnet build` under `Debug` and
  `Release` with `TreatWarningsAsErrors=true` enforced — this catches any
  package-version incompatibility, removed APIs, or new analyzer warnings
  introduced by the 10.x packages.
- **Existing automated tests**: run all three test projects unchanged — they
  serve as the regression net for any behavioral change inadvertently introduced
  by the `Microsoft.Extensions.*` 8 → 10 jump or by the runtime change.
- **Manual smoke**: launch `VideoIndexer.App` once via `dotnet run` and confirm
  the shell view loads (no runtime-binding/JIT failure on the new TFM).
- **Static check**: repo-wide grep for `net8.0` across `src/`, `tests/`,
  `README.md`, and `docs/projects/rebuild/` returning zero matches.
  Exclude `docs/agents/` from this check entirely — the plan and synthesis
  documents under `docs/agents/plans/2026-04-30-net10-upgrade/` legitimately
  contain `net8.0` references describing this very migration, and historical
  plan/synthesis files dated before 2026-04-30 are preserved as a faithful
  record.

No new tests are added; this work is a metadata/infrastructure migration with no
new behavior to specify.

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Avalonia 11.2.5 has a latent issue on the .NET 10 runtime** (e.g. compiled-binding source generator, JIT intrinsics) that only surfaces at runtime | Manual smoke run of `VideoIndexer.App` is part of acceptance; if reproducible, fall back to a targeted Avalonia bump (out of scope of this plan but documented as a follow-up) rather than reverting the whole upgrade. |
| **`TreatWarningsAsErrors=true` blocks the build** because `Microsoft.Extensions.*` 10 introduces new analyzer warnings or obsoletions | Address each warning in-place (most likely: nullable annotation tightening, `[Obsolete]` API replacements). If the volume balloons unexpectedly, scope-creep is contained by quarantining the offending project with a pragma + tracked follow-up issue rather than disabling the rule globally. |
| **Transitive dependency of a framework-agnostic package (e.g. `MySqlConnector`, `Dapper`) doesn't yet declare `net10.0` support** | These packages target `netstandard2.0/2.1` which is forward-compatible. Watch for NU1603 / NU1605 (version downgrades) and NU1608 (version conflicts) during restore — evaluate per-package and record any genuine incompatibility for follow-up. (NU1701 specifically targets `.NETFramework`-only packages and is not expected here.) |
| **Asymmetric `TreatWarningsAsErrors` posture across the three test projects** causes a new analyzer warning (post 8→10 bump) to fail one test project while passing the other two | Step 6 explicitly aligns the posture across all three test projects so any new analyzer warning produces consistent behaviour. |
| **Avalonia 11.2.5 predates .NET 10 GA** and supports it via netstandard fallback rather than native `net10.0` targeting | Manual smoke-test of `VideoIndexer.App` is part of acceptance. If a runtime-binding issue surfaces, document as a follow-up and consider a targeted Avalonia 11.3+ bump in a separate plan. |
| **`Serilog.Extensions.Hosting` lags behind .NET 10 GA** | Confirm latest version on `nuget.org` during implementation; if the latest stable still targets `net8.0`, keep it on its current major and rely on netstandard fallback (the runtime is forward-compatible). Document the choice in the synthesis. |
| **Centralizing `TargetFramework` in `Directory.Build.props` breaks a future need for a per-project override** (e.g. multi-targeting `Core` to `netstandard2.1` for sharing) | Override is still possible by re-declaring `<TargetFramework>` (or `<TargetFrameworks>`) inside the specific `.csproj`; the centralization sets a default, not a ceiling. |
| **`global.json` `latestFeature` rolls forward to a future 10.x SDK that introduces breaking analyzer behavior** | Acceptable trade-off vs. the previous `latestMajor`; SDK feature bands are by policy non-breaking for source. If a specific feature band misbehaves, pin to a known-good band. |
| **Historical references to `net8.0` in old plans confuse future agents** | Step 9 explicitly documents the policy ("docs dated before 2026-04-30 are historical"); the README and milestone docs — the canonical forward-facing surfaces — are updated to remove ambiguity. |

AGENT: Planning
STATUS: READY_FOR_PM
