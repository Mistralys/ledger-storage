# Plan: Add Target Model Field to Ledger Personas

## Summary

Add a per-persona `model` field to the ledger persona YAML metadata and the VS Code frontmatter template, and update the Claude Code `cc_model` from the global `"inherit"` default to per-persona model values. The Planner (Agent 1) and Project Manager (Agent 2) will target **Claude Opus 4.6**; the remaining five agents (3–7) will target **Claude Sonnet 4.6**.

## Architectural Context

The persona build system ([personas/docs/agents/project-manifest/](../../../personas/docs/agents/project-manifest/)) assembles generated persona files from YAML metadata + Markdown templates using `scripts/build-personas.js`.

**Frontmatter templates** (constants in [scripts/build-personas.js](../../../../scripts/build-personas.js)):
- `FRONTMATTER_LEDGER_VSCODE` — currently has **no** `model` field
- `FRONTMATTER_LEDGER_CC` — already has `model: {{cc_model}}`, currently resolved from `_shared.yaml` → `cc_model: "inherit"`

**Context assembly** in `buildForTarget()` ([scripts/build-personas.js](../../../../scripts/build-personas.js#L596-L632)):
- Shared fields from `_shared.yaml` are loaded first, then per-persona fields are spread on top (`...persona`), so per-persona keys override shared defaults.
- `cc_model` is already wired: `context.cc_model = sharedMeta.cc_model` — and the spread adds any per-persona `cc_model` if present.

**Per-persona YAML files** (7 files in [personas/ledger/src/meta/](../../../../personas/ledger/src/meta/)):
- None currently have a `model` or `cc_model` field. All Claude Code builds resolve `cc_model` from the shared `"inherit"` value.

**Standalone suite** ([personas/standalone/src/meta/_shared.yaml](../../../../personas/standalone/src/meta/_shared.yaml)) also has `cc_model: "inherit"` and its VS Code frontmatter (`FRONTMATTER_STANDALONE_VSCODE`) also lacks a `model` field. This plan scopes changes to the **ledger suite only** — standalone is out of scope.

## Approach / Architecture

### Design: Per-persona `model` field with `default_model` shared fallback

Introduce a new pattern that mirrors the existing `version` / `default_version` override pattern:

1. Add **`default_model`** to [personas/ledger/src/meta/_shared.yaml](../../../../personas/ledger/src/meta/_shared.yaml) — set to `"Claude Sonnet 4.6"` (the majority case).
2. Add a per-persona **`model`** field to personas that need a different value (Agents 1 & 2 → `"Claude Opus 4.6"`). Personas without a `model` field inherit `default_model`.
3. In `buildForTarget()`, resolve `model` using the same precedence pattern as `version`:
   ```js
   const model = persona.model !== undefined
     ? persona.model
     : sharedMeta.default_model;
   ```
4. Add `model` to the `context` object so both `{{model}}` and `{{cc_model}}` can reference it.
5. Add `model: {{model}}` to `FRONTMATTER_LEDGER_VSCODE`.
6. **Re-map `cc_model`**: change the context assembly so `cc_model` resolves from the per-persona `model` field (or `default_model`) instead of the current `cc_model: "inherit"` shared field. This unifies the model selection across both targets. The old `cc_model` field in `_shared.yaml` becomes the fallback for suites that don't use per-persona models (e.g., standalone).

### Why a shared `default_model` instead of per-persona everywhere

Five of seven agents share the same model. Duplicating `model: "Claude Sonnet 4.6"` seven times violates DRY and creates maintenance overhead when the model is upgraded. The `default_model` + per-persona override pattern is already established for `default_version` and `default_cc_tools`, so this is consistent.

### Why unify `model` and `cc_model`

Currently `cc_model` is a separate concern because Claude Code may need different models. However, the user wants the same model selection for both targets. Keeping two independent fields (`model` + `cc_model`) would create confusion. Instead, the computed `cc_model` context variable will be derived from the resolved `model` value, unless a per-persona `cc_model` is explicitly set (escape hatch for future divergence).

Resolution order for `cc_model`:
```
persona.cc_model → persona.model → sharedMeta.default_model → sharedMeta.cc_model (legacy fallback)
```

This preserves backward compatibility: if `default_model` is absent (standalone suite), the existing `cc_model: "inherit"` path still works.

## Rationale

- **Token-efficient reasoning models for planning**: Opus 4.6 excels at complex multi-step reasoning required by the Planner and PM roles — these agents produce architectural plans and decompose work, benefiting from deeper reasoning.
- **Cost/speed-optimized for execution roles**: Sonnet 4.6 provides excellent coding and analysis capabilities at lower cost/latency, appropriate for the five execution-oriented roles.
- **Existing pattern compliance**: The `default_X` + per-persona override pattern is established (`default_version`, `default_cc_tools`) — no new architectural concept is introduced.

## Detailed Steps

### 1. Add `default_model` to ledger `_shared.yaml`

**File:** [personas/ledger/src/meta/_shared.yaml](../../../../personas/ledger/src/meta/_shared.yaml)

Add a new field `default_model: "Claude Sonnet 4.6"` alongside the existing shared fields. Add a comment explaining the override pattern.

### 2. Add `model` field to Agents 1 and 2 YAML

**Files:**
- [personas/ledger/src/meta/1-planner.yaml](../../../../personas/ledger/src/meta/1-planner.yaml)
- [personas/ledger/src/meta/2-project-manager.yaml](../../../../personas/ledger/src/meta/2-project-manager.yaml)

Add `model: "Claude Opus 4.6"` to each. Agents 3–7 intentionally omit the field and inherit `default_model`.

### 3. Update `buildForTarget()` in build script

**File:** [scripts/build-personas.js](../../../../scripts/build-personas.js)

In the context-building section (around the existing `version` resolution):

a. Resolve `model`:
```js
const model = persona.model !== undefined
  ? persona.model
  : (sharedMeta.default_model || sharedMeta.cc_model || 'inherit');
```

b. Compute unified `cc_model`:
```js
const ccModel = persona.cc_model !== undefined
  ? persona.cc_model
  : model;
```

c. Add `model` and override `cc_model` in the context object:
```js
const context = {
  ...
  cc_model: ccModel,    // override the spread value from sharedMeta
  ...
  model,
};
```

### 4. Add `model` to `FRONTMATTER_LEDGER_VSCODE`

**File:** [scripts/build-personas.js](../../../../scripts/build-personas.js)

Add `model: {{model}}` to the `FRONTMATTER_LEDGER_VSCODE` template. Place it after the `description` line (matching the position shown in Gemini's pseudo-example, and logically grouping identity fields together):

```yaml
---
id: {{id}}
name: '{{number}} - {{role}} v{{version}}'
description: 'Step {{number}}/{{total}} in the agent workflow.'
model: {{model}}
role: {{role}}
author: {{author}}
version: {{version}}
last_updated: {{last_updated}}
vs_file_name: {{vs_file_name}}
tools: {{tools_json}}
---
```

No change needed to `FRONTMATTER_LEDGER_CC` — it already has `model: {{cc_model}}`.

### 5. Update manifest documentation

**Files to update:**

a. [personas/docs/agents/project-manifest/api-surface.md](../../../../personas/docs/agents/project-manifest/api-surface.md):
   - Add `default_model` to the `_shared.yaml` schema table
   - Add `model` to the per-persona YAML schema table (optional field, overrides `default_model`)
   - Update the `FRONTMATTER_LEDGER_VSCODE` template listing to include `model: {{model}}`
   - Add `{{model}}` to the computed variables table (explain resolution chain)

b. [personas/docs/agents/project-manifest/constraints.md](../../../../personas/docs/agents/project-manifest/constraints.md):
   - Add a constraint documenting the `default_model` + per-persona `model` override pattern
   - Document the `cc_model` resolution chain (`persona.cc_model → persona.model → sharedMeta.default_model → sharedMeta.cc_model`)

### 6. Rebuild and verify

Run the build and verify:
```bash
node scripts/build-personas.js --suite ledger --strict
```

Verify that:
- Agents 1 and 2 have `model: Claude Opus 4.6` in VS Code output and `model: Claude Opus 4.6` in Claude Code output
- Agents 3–7 have `model: Claude Sonnet 4.6` in both outputs
- `--strict` passes (no unresolved markers)
- `--check` will fail (expected — output is now updated)

### 7. Sync to IDEs

```bash
node scripts/sync-personas.js
```

## Dependencies

- No external dependencies. All changes are within the personas build system.
- Steps 1–4 must be completed before Step 6 (build/verify).
- Step 5 (docs) can be done in parallel with Steps 1–4.

## Required Components

| Component | Status | Path |
|-----------|--------|------|
| Ledger shared YAML | **existing — modify** | `personas/ledger/src/meta/_shared.yaml` |
| Planner YAML | **existing — modify** | `personas/ledger/src/meta/1-planner.yaml` |
| Project Manager YAML | **existing — modify** | `personas/ledger/src/meta/2-project-manager.yaml` |
| Build script | **existing — modify** | `scripts/build-personas.js` |
| API surface doc | **existing — modify** | `personas/docs/agents/project-manifest/api-surface.md` |
| Constraints doc | **existing — modify** | `personas/docs/agents/project-manifest/constraints.md` |

No new files are created.

## Assumptions

- "Claude Opus 4.6" and "Claude Sonnet 4.6" are the exact model identifier strings expected by VS Code's `model` frontmatter field. If VS Code uses different identifiers (e.g., `claude-opus-4-6`), the values will need adjustment.
- The standalone suite does not need model pinning at this time (remains `cc_model: "inherit"`).
- The `model` field in VS Code agent frontmatter is a supported field (based on the Gemini pseudo-example provided).

## Constraints

- **No changes to standalone suite** — this plan only touches the ledger suite.
- **No changes to `FRONTMATTER_LEDGER_CC` template** — it already has `model: {{cc_model}}` which will now resolve to the unified model value.
- **Backward compatibility**: if `default_model` is absent from `_shared.yaml` (e.g., standalone suite), the fallback chain gracefully degrades to `cc_model` → `"inherit"`.
- Follow the Edit → Build → Sync workflow (constraint 3 in [constraints.md](../../../../personas/docs/agents/project-manifest/constraints.md)).

## Out of Scope

- Standalone persona model assignment
- Model-specific behavioral tuning in persona content templates
- Automated model version detection or validation
- Changes to the MCP server or orchestrator

## Acceptance Criteria

- [ ] `FRONTMATTER_LEDGER_VSCODE` template includes `model: {{model}}`
- [ ] `_shared.yaml` has `default_model: "Claude Sonnet 4.6"`
- [ ] `1-planner.yaml` and `2-project-manager.yaml` have `model: "Claude Opus 4.6"`
- [ ] Agents 3–7 YAML files do NOT have a `model` field (inherit default)
- [ ] Generated VS Code output for Agents 1–2 contains `model: Claude Opus 4.6`
- [ ] Generated VS Code output for Agents 3–7 contains `model: Claude Sonnet 4.6`
- [ ] Generated Claude Code output for Agents 1–2 contains `model: Claude Opus 4.6`
- [ ] Generated Claude Code output for Agents 3–7 contains `model: Claude Sonnet 4.6`
- [ ] `node scripts/build-personas.js --suite ledger --strict` passes (exit 0)
- [ ] Manifest documents (`api-surface.md`, `constraints.md`) updated
- [ ] No regressions in standalone suite (`node scripts/build-personas.js --suite standalone --check`)

## Testing Strategy

1. **Build verification**: Run `node scripts/build-personas.js --suite ledger --strict` — confirms no unresolved markers.
2. **Output inspection**: Spot-check generated files in `personas/ledger/vs-code/` and `personas/ledger/claude-code/` for correct `model` values.
3. **Standalone regression**: Run `node scripts/build-personas.js --suite standalone --check` — confirms no side effects on standalone suite.
4. **Sync dry-run**: Run `node scripts/sync-personas.js --dry-run` to verify sync detects the updated files.

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **VS Code doesn't recognize `model` field** | Field is ignored if unsupported — no breakage. Can be verified by invoking an agent after sync. |
| **Model name string mismatch** | Verify the exact model identifier string after implementation by checking VS Code's model picker or documentation. Field value can be trivially updated in YAML. |
| **`cc_model` resolution change breaks standalone** | Fallback chain degrades gracefully: standalone has no `default_model`, so `sharedMeta.cc_model` (= `"inherit"`) is used. Verified by the standalone `--check` test. |
| **Context spread order matters** | The `cc_model` override must be placed AFTER the `...persona` spread in the context object, or use explicit assignment. The plan's step 3 addresses this with an explicit `cc_model: ccModel` property. |
