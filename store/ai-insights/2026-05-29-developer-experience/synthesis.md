# Synthesis Report — Developer Experience Improvements

**Plan:** 2026-05-29-developer-experience  
**Completed:** 2026-05-30  
**Status:** ALL 12 WORK PACKAGES COMPLETE

---

## Executive Summary

This cycle delivered all seven Tier 1 developer-experience improvements identified in the research
paper (`docs/agents/research/2026-05-29-developer-experience.md`). The work was implemented across
three repositories (`ai-insights`, `cli-menu`, `ai-persona-builder`) with no regressions.

The deliverables fall into four layers:

1. **Foundation** — A shared, cost-tiered health-check registry (`scripts/lib/health-checks.js`)
   with 9 annotated checks across three cost tiers, eliminating duplicate detection logic across
   three consumers.

2. **Library** — `@mistralys/cli-menu` v1.1.0 gained two new optional `MenuConfig` properties:
   `statusLines` (header status rendering) and `firstRunRedirect` / `onFirstRun` (first-run
   wizard trigger). Released with a clean production build.

3. **Consumer** — `scripts/cli.js` gained: a live health status block in the menu header, a
   `doctor` command, a `install-mcp` command for global MCP registration, a first-run wizard
   handler, and an `--skip-setup-check` CI bypass flag.

4. **Shim** — `scripts/install-mcp-global.js` implements a stable-shim strategy for user-level
   MCP registration that survives repo moves without touching IDE config.

---

## Metrics

| Category | Value |
|---|---|
| Work packages | 12 / 12 COMPLETE |
| Acceptance criteria | 49 / 49 MET |
| ai-insights test suite | 80 pass, 0 fail |
| cli-menu test suite | 253 pass, 0 fail (↑ from 236) |
| Combined tests | 333 pass, 0 fail |
| Reviewer Fix-Forwards applied | 2 |
| Security issues (critical / high) | 0 / 0 |
| Security issues (medium — noted) | 1 |
| New npm production dependencies | 0 |

### Fix-Forwards Applied

| WP | Fix | Nature |
|---|---|---|
| WP-001 | Renamed `@typedef InstantCheck` → `SyncCheck` | JSDoc-only; the typedef covered both instant and fast tiers, making the name misleading |
| WP-011 | Removed redundant `async` keyword from `handleFirstRun()` | The function body was `return new Promise(...)` — `async` was a no-op antipattern |

### Security Findings

| Severity | Location | Description |
|---|---|---|
| Medium (noted, not blocking) | `scripts/install-mcp-global.js` (shim) | `distPath` is constructed from `config.json`'s `repoPath` without a path-containment check. A manually edited config pointing to an arbitrary `.js` file would execute it in MCP server context. Exploitability is LOW (developer tool; config writable only by the local user). Recommended remediation: add `path.resolve(distPath).startsWith(path.resolve(repoPath) + path.sep)` guard before spawn. |

---

## Deliverables by Work Package

| WP | Feature | Key Files |
|---|---|---|
| WP-001 | Health-check registry | `scripts/lib/health-checks.js`, `scripts/tests/health-checks.test.js` |
| WP-002 | `cli-menu` `statusLines` property | `cli-menu/src/types.ts`, `cli-menu/src/menu/renderer.ts` |
| WP-003 | Global MCP registration script | `scripts/install-mcp-global.js`, `scripts/cli.js` |
| WP-004 | npm `bin` field | `package.json` |
| WP-005 | Bootstrap staleness detection + missing-repo guidance | `scripts/preflight-bootstrap.js` |
| WP-006 | `cli-menu` first-run wizard support (v1.1.0) | `cli-menu/src/types.ts`, `cli-menu/src/menu/interactive.ts` |
| WP-007 | Status lines wired into `scripts/cli.js` menu | `scripts/cli.js`, `package.json` |
| WP-008 | `doctor` command | `scripts/cli.js` |
| WP-009 | install-mcp CLI wiring (verified complete via WP-003) | `scripts/cli.js` |
| WP-010 | Migrate `preflight-orchestrator.js` to health-checks | `scripts/preflight-orchestrator.js` |
| WP-011 | First-run wizard handler in `scripts/cli.js` | `scripts/cli.js` |
| WP-012 | Cross-project documentation integration | `cli-menu/docs/configuration.md`, `cli-menu/docs/agents/project-manifest/data-flows.md`, `mcp-server/docs/agents/project-manifest/README.md`, `personas/docs/agents/project-manifest/constraints.md` |

---

## Notable Technical Decisions

### Health-check cost tiers
`HEALTH_CHECKS` entries carry a `cost` field (`instant` / `fast` / `slow`). `runChecks(costFilter)`
allows callers to request only the checks they need:
- Menu status lines: `instant` tier only (synchronous, zero blocking)
- `doctor` command: `all` tiers (awaits slow checks before exit code)
- Preflight-orchestrator: delegates specific checks by ID

### Accepted tech debt resolved
`latestMtime()` was temporarily duplicated between `scripts/preflight-bootstrap.js` and
`scripts/lib/health-checks.js`. WP-010 eliminated the duplication in
`scripts/preflight-orchestrator.js`. The `preflight-bootstrap.js` copy remains (isolated scope);
a future consolidation can export `latestMtime` from `health-checks.js`.

### cli-menu version jump
`@mistralys/cli-menu` went from `0.1.0` directly to `1.1.0`, skipping `1.0.0`. This was an
explicit planning decision. The three new optional `MenuConfig` properties (`statusLines`,
`firstRunRedirect`, `onFirstRun`) are all additive; no breaking changes were introduced.

### Ledger metadata drift (audit note)
WP-009 and WP-004 exchanged `work_package_file` references during execution (each points to the
other's spec file). The reviewer flagged this for a future ledger audit pass. Code correctness
is unaffected — all acceptance criteria were verified against the correct source.

---

## Strategic Recommendations

### 1. Add path-containment guard to the shim (security)
The medium-severity finding from WP-003's security audit (see Metrics section) is low exploitability
today but worth resolving as a defence-in-depth measure. Add a `path.resolve(distPath).startsWith(path.resolve(repoPath) + path.sep)` check in `_buildShimContent()` before the
`spawn` call, exiting with a diagnostic on failure. One-line fix.

### 2. Add null guards to `HEALTH_CHECKS.find()` references
`scripts/preflight-orchestrator.js` resolves three health-check entries by ID at module load time
(`hcMcpDist`, `hcMcpDistFresh`, `hcOrcVenv`). If a check ID is renamed in `health-checks.js`,
`find()` returns `undefined` and the subsequent `.detect()` call throws a `TypeError` at runtime
rather than startup. Adding assertive guards (e.g. `if (!hcMcpDist) throw new Error(...)`) makes
ID-rename failures loud and immediate.

### 3. Regenerate `.context/` snapshot
`scripts/lib/health-checks.js` is a new file and several existing scripts were modified. Run
`node scripts/cli.js ctx-generate` to refresh `.context/scripts.md` and the workspace-structure
snapshot for downstream LLM/NotebookLM consumers.

### 4. Add fast-tier sync contract test (AC-2b candidate)
WP-001 QA noted that the fast-tier detect() sync contract (returns plain boolean, no Promise) is
documented in the `health-checks.js` module header but has no dedicated test assertion parallel to
the instant-tier AC-2 test. A future AC-2b could explicitly assert fast-tier entries also return
plain booleans, hardening the contract.

### 5. Align `--dry-run` error handling in install-mcp-global.js
`dryRun()` uses `console.log`/`console.error` directly, while `install()` accepts an `opts.log`
callback. This inconsistency makes `dryRun()` harder to test in isolation. Low priority — the
function is display-only — but worth aligning in the next maintenance pass.

---

## Open Items at Session End

| Item | Priority | Source |
|---|---|---|
| Path-containment guard in shim | Medium | WP-003 security audit |
| HEALTH_CHECKS.find() null guards | Low | WP-010 code-review |
| `.context/` snapshot regeneration | Low | WP-001 documentation |
| Fast-tier sync contract test | Low | WP-001 QA |
| `--dry-run` error handling parity | Low | WP-003/WP-009 code-review |
| Ledger WP-004/WP-009 metadata drift audit | Low | WP-009 code-review |
