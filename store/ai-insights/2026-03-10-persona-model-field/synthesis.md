# Project Synthesis Report

**Project:** 2026-03-10-persona-model-field  
**Date:** 2026-03-10  
**Status:** COMPLETE  
**Report Author:** Head of Operations (Synthesis)

---

## Executive Summary

This project introduced a `model` field to the **Ledger persona build system**, enabling per-persona and suite-level AI model assignment in generated VS Code and Claude Code frontmatter. The work was clean and surgical: a three-tier fallback chain (`per-persona model` → `default_model` → `cc_model` → `'inherit'`) was implemented in `scripts/build-personas.js`, mirroring the existing `version` resolution pattern. All 14 ledger persona files now carry a `model:` frontmatter value — `Claude Opus 4.6` for the Planner and Project Manager agents, `Claude Sonnet 4.6` for all others. The standalone suite was left untouched. No regressions were introduced.

Three work packages were completed across a full pipeline cycle (Implementation → QA → Code Review → Documentation):

| WP | Title | Owner | Result |
|----|-------|-------|--------|
| WP-001 | Implement model field in persona build system | Documentation | PASS |
| WP-002 | Build/sync verification of generated output | Documentation | PASS |
| WP-003 | Manifest documentation updates | Documentation | PASS |

---

## Metrics

| WP | QA Tests Passed | QA Tests Failed | All Pipelines |
|----|----------------|-----------------|---------------|
| WP-001 | 9 | 0 | 4 × PASS |
| WP-002 | 6 | 0 | 4 × PASS |
| WP-003 | 6 | 0 | 4 × PASS |
| **Total** | **21** | **0** | **12 × PASS** |

- **Build regression status:** None. Both `--suite ledger --strict` and `--check` passed cleanly; standalone suite output unchanged.
- **Stale output:** None. `--check` confirmed all 14 ledger files are current.
- **Unresolved template markers:** None found in any generated file.
- **Security issues:** 0
- **Cross-system parity:** `check-known-roles.js` passes — `KNOWN_ROLES` and `AGENT_ROLES` remain in sync.

---

## Artifacts

**Source files modified:**

- `personas/ledger/src/meta/_shared.yaml` — added `default_model: 'Claude Sonnet 4.6'`; updated legacy `cc_model` comment
- `personas/ledger/src/meta/1-planner.yaml` — added `model: 'Claude Opus 4.6'`
- `personas/ledger/src/meta/2-project-manager.yaml` — added `model: 'Claude Opus 4.6'`
- `scripts/build-personas.js` — updated `buildForTarget()` with model resolution logic and `{{model}}` in `FRONTMATTER_LEDGER_VSCODE`

**Generated output (14 files):** All files in `personas/ledger/vs-code/` and `personas/ledger/claude-code/`.

**Manifest documentation updated:**

- `personas/docs/agents/project-manifest/api-surface.md` — `{{model}}` computed variable, `default_model` in `_shared.yaml` schema, `model` in per-persona YAML schema, `FRONTMATTER_LEDGER_VSCODE` template listing
- `personas/docs/agents/project-manifest/data-flows.md` — `default_model` in Layer 1 shared metadata; `model` in Layer 3 computed values with resolution chain
- `personas/docs/agents/project-manifest/constraints.md` — constraints 28b (default_model + per-persona override pattern) and 28c (cc_model resolution chain)

---

## Strategic Recommendations (Gold Nuggets)

Three low-priority cleanup items were independently flagged by multiple agents (QA, Reviewer, Documentation) across WP-001 and WP-002. None are blocking, but all are worth scheduling in a future tidy-up pass:

### 1. Dead assignment in `build-personas.js` context object (Low Priority)

In `scripts/build-personas.js` (~line 611), the property `cc_model: sharedMeta.cc_model` is set inside the context object but is **always overridden** — first by the `...persona` spread, then explicitly by `cc_model: ccModel`. The dead assignment adds noise and obscures the three-step intent (shared defaults → persona override → computed final).

**Recommendation:** Remove `cc_model: sharedMeta.cc_model` from the context object declaration.

### 2. Unquoted `model:` value in VS Code frontmatter template (Low Priority)

`FRONTMATTER_LEDGER_VSCODE` emits `model: {{model}}` as a bare, unquoted YAML scalar. Current model names (`Claude Opus 4.6`, `Claude Sonnet 4.6`) are YAML-safe. However, any future model name containing a colon or other YAML special character would silently produce invalid frontmatter.

**Recommendation:** Change to `model: '{{model}}'` in the template. Also consider applying the same defensive quoting to `cc_model` in `FRONTMATTER_LEDGER_CC` and standalone templates for consistency.

### 3. Legacy `cc_model` field in `_shared.yaml` (Low Priority)

The `cc_model: inherit` field in `personas/ledger/src/meta/_shared.yaml` is no longer functionally used — the build script now derives `cc_model` from `model`/`default_model`. It is documented inline but is dead configuration.

**Recommendation:** Remove `cc_model` from `_shared.yaml` in a future cleanup pass, once confident all dependent paths have been validated (e.g., the standalone suite which does not have `default_model`).

---

## Next Steps

1. **Immediate:** No blockers — all WPs are COMPLETE and all generated output is current and correct.
2. **Short-term tidy-up:** Address the three cleanup items above (dead assignment, unquoted template value, legacy `cc_model` in `_shared.yaml`).
3. **Follow-on consideration:** Evaluate whether the `cc_model` field in `FRONTMATTER_LEDGER_CC` and standalone suite templates should also adopt the `model`/`default_model` pattern for full consistency, or whether the `inherit` behavior is intentional for those targets.
4. **Planner:** No new planning work is required from this cycle — the feature is feature-complete and fully documented.
