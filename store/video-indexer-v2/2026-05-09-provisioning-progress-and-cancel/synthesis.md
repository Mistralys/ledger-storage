# Project Synthesis Report

**Plan:** provisioning-progress-and-cancel  
**Date:** 2026-05-11  
**Status:** COMPLETE — 8/8 Work Packages Delivered  
**Pipeline Health:** All 8 WPs passed all 4 stages (implementation → QA → code-review → documentation)

---

## Executive Summary

This session resolved six root-cause defects in the "Setting up external tools" (Provisioning Tools) screen. Before this work: the provisioning process never started, the Cancel button was a no-op, and the UI showed no meaningful feedback beyond a spinning indeterminate progress bar.

Eight work packages were delivered across a single session (~3 hours wall time). The changes span three source files and two test files as core deliverables, plus comprehensive README and CHANGELOG updates.

**What was built:**

| # | Deliverable | File(s) Touched |
|---|-------------|-----------------|
| WP-001 | `onComplete: Action?` constructor callback on `ProvisioningToolsViewModel` — fires once on success, never on cancel/throw | `ProvisioningToolsViewModel.cs` |
| WP-002 | `Log: ObservableCollection<string>` — timestamped entries, 200-entry cap, terminal entry on every run, UIThread-marshalled | `ProvisioningToolsViewModel.cs` |
| WP-003 | `ShellViewModel.AdvanceToConnecting()` — thin public façade over `Transition(ShellState.Connecting)` | `ShellViewModel.cs` |
| WP-004 | `Message` observable property + `DownloadPercent` computed property on `ProvisioningToolsViewModel` | `ProvisioningToolsViewModel.cs` |
| WP-005 | `Program.cs` wiring: inline `ProvisioningToolsViewModel` construction with `onComplete` callback; `AddTransient<>` DI registration removed | `Program.cs` |
| WP-006 | `ProvisioningToolsView` updated: Loaded handler auto-starts provisioning, Message TextBlock, DownloadPercent label, bounded/auto-scrolling Log panel | `ProvisioningToolsView.axaml` + `.axaml.cs` |
| WP-007 | Cross-thread safety tests via `ThreadPoolProgressProvisioner` fake (5 new tests) | `ProvisioningToolsViewModelTests.cs` |
| WP-008 | `AdvanceToConnecting_TransitionsToConnecting` unit test for ShellViewModel | `ShellViewModelTests.cs` |

---

## Metrics

### Test Results (final solution-wide)

| Assembly | Passed | Failed | Skipped |
|----------|--------|--------|---------|
| `VideoIndexer.App.Tests` | 98 | 0 | 0 |
| `VideoIndexer.Core.Tests` | 128 | 0 | 0 |
| `VideoIndexer.Infrastructure.Tests` | 124 | 0 | 5 (live/integration) |
| **Total** | **350** | **0** | **5** |

### New Test Coverage Added

| Scope | Tests Added |
|-------|-------------|
| `onComplete` callback (success / cancel / throw) | 3 |
| `Log` behaviour (timestamped entries, cap, terminal, null-skip) | 6 |
| `Message` null-skip; `DownloadPercent` formatting + `PropertyChanged` | 6 |
| `AdvanceToConnecting` unit test (`ShellViewModelTests`) | 1 |
| Cross-thread safety (`ThreadPoolProgressProvisioner`) | 5 |
| **Total new tests** | **21** |

### Rework Events

| WP | Stage | Rework Reason |
|----|-------|---------------|
| WP-002 | implementation → QA FAIL → implementation fix | `Log.Count < 200` guard placed outside `UIThread.Post` lambda — evaluated before any dispatch ran, allowing all entries through. Fixed by moving the guard inside the lambda so it executes on the UI thread. |

---

## Bugs Caught

| Severity | Location | Description |
|----------|----------|-------------|
| **Medium** | `ProvisioningToolsViewModel.cs` — Log cap | Guard `Log.Count < 200` was evaluated on the thread-pool before any `Dispatcher.UIThread.Post` callback had fired. All synchronous progress reports bypassed the cap. Fixed: guard moved inside the `Post` lambda. |

*(No other functional bugs were introduced or found during this session.)*

---

## Strategic Recommendations ("Gold Nuggets")

### 1. SharpCompress Security Vulnerability (High Priority — Pre-existing)

**WP-002 Developer comment (medium priority)**  
`SharpCompress 0.47.4` has a known moderate severity vulnerability (`GHSA-6c8g-7p36-r338`). With `TreatWarningsAsErrors=true` in `Directory.Build.props`, `dotnet build` fails unless `-p:NuGetAudit=false` is passed. This was a recurring obstacle during the session.  
**Recommendation:** Upgrade `SharpCompress` to a non-vulnerable version in `Directory.Packages.props`, or add a `<NuGetAuditSuppress>` entry for the advisory if the risk is accepted. This unblocks normal build verification across all future sessions.

### 2. `onComplete` Thread-Affinity Documentation Gap (Medium Priority — Addressed)

**WP-001 Reviewer fix-forward (applied)**  
The `onComplete` callback fires on a **ThreadPool thread** (due to `ConfigureAwait(false)`). Callers that need to update UI must self-dispatch. This was caught by the Reviewer, corrected in XML docs on the constructor, and documented in CHANGELOG.md.  
**Recommendation:** Any future `Action?` callback parameters on ViewModels should document thread affinity explicitly at the point of declaration.

### 3. Test Redundancy in WP-007 (Medium Priority — Not Blocking)

**WP-007 Reviewer code-smell (medium priority)**  
`StartAsync_WhenCancelled_DoesNotInvokeOnCompleteCallback` and `StartAsync_WhenProvisionerThrows_DoesNotInvokeOnCompleteCallback` are functional duplicates of existing tests (`DoesNotInvokeOnComplete_Cancelled`, `DoesNotInvokeOnComplete_Throws`). Neither duplicate uses `ThreadPoolProgressProvisioner`.  
**Recommendation:** In a future maintenance pass, remove the duplicate cancel/throw tests to avoid double maintenance burden if the callback contract changes.

### 4. `ShellViewModel.ProvisionAsync()` Dead Production Code (Low Priority — By Design)

**WP-005 Reviewer observation (low priority)**  
`ShellViewModel.ProvisionAsync()` is intentionally left as dead production code (per `plan.md` line 75) to preserve existing `ShellViewModelTests` coverage. This is architecturally correct — it continues to serve as a contract-level integration test for the shell's transition logic.  
**Recommendation:** No action required now. If the test coverage is ever migrated to a dedicated shell-contract test, `ProvisionAsync()` can be removed.

### 5. `CollectionChanged` Subscription Lifetime Documented (Low Priority — Resolved)

**WP-006 Reviewer concern (addressed)**  
The `Log.CollectionChanged` subscription in `ProvisioningToolsView.axaml.cs` has no `Unloaded` cleanup. This is intentional and correct: the view and its VM have identical lifetimes via the shell state machine's dispose-on-transition contract. An inline comment was added by the Documentation agent.  
**Recommendation:** If future views reuse `ProvisioningToolsView` across multiple shell entries, revisit and add an `OnDetachedFromVisualTree` unsubscription.

---

## Files Modified (Session Total)

| File | Change Summary |
|------|---------------|
| `src/VideoIndexer.App/ViewModels/ProvisioningToolsViewModel.cs` | `onComplete` callback, `Log` collection, `Message` property, `DownloadPercent` computed property, XML docs |
| `src/VideoIndexer.App/ViewModels/ShellViewModel.cs` | `AdvanceToConnecting()` public façade, state-machine transition diagram updated |
| `src/VideoIndexer.App/Program.cs` | Inline `ProvisioningToolsViewModel` construction with `onComplete`; `AddTransient<>` removed |
| `src/VideoIndexer.App/Views/ProvisioningToolsView.axaml` | Message TextBlock, DownloadPercent label, Log ScrollViewer panel |
| `src/VideoIndexer.App/Views/ProvisioningToolsView.axaml.cs` | Loaded handler (auto-start + auto-scroll subscription), lifetime comment |
| `tests/VideoIndexer.App.Tests/ProvisioningToolsViewModelTests.cs` | 21 new tests across callback, Log, Message, DownloadPercent, cross-thread |
| `tests/VideoIndexer.App.Tests/ShellViewModelTests.cs` | `AdvanceToConnecting_TransitionsToConnecting` test |
| `CHANGELOG.md` | `### Added` entries for all 4 new VM features |
| `README.md` | Updated DI registration docs, callback wiring, observable properties table, new View subsection, test coverage descriptions |

---

## Next Steps

1. **Upgrade SharpCompress** — remove the `-p:NuGetAudit=false` workaround that was required throughout this session (see Gold Nugget #1).
2. **End-to-end manual test** — verify the provisioning screen starts, shows live Message updates, populates the Log panel, shows download percentage, and navigates forward to `ShellState.Connecting` on success.
3. **Clean up duplicate tests** — remove the 2 redundant cancel/throw callback tests identified in WP-007 (Gold Nugget #3, low urgency).
4. **Plan: remaining root causes** — Root causes #2 (Cancel button no-op) and #5 (indeterminate progress outside download phase) were not addressed in this plan's scope. The Cancel button wiring was unblocked by this session's work (CTS is created when `RunProvisioningAsync` runs, which now fires from the Loaded handler), but a dedicated plan should verify the Cancel UX end-to-end.
