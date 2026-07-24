
# Plan — Post-Implementation Cleanup (model field rework)

## Summary

Address the three low-priority cleanup items and one follow-on consideration identified in the `2026-03-10-persona-model-field` synthesis. These items were independently flagged by multiple agents (QA, Reviewer, Documentation) and relate to dead code, defensive quoting, legacy configuration, and cross-suite model-field consistency.

## Architectural Context

The persona build system ([scripts/build-personas.js](scripts/build-personas.js)) assembles persona Markdown files from YAML metadata in `personas/<suite>/src/meta/` and content templates in `personas/<suite>/src/content/`. The recent `model` field introduction added a three-tier fallback chain:

```
persona.model → sharedMeta.default_model → sharedMeta.cc_model → 'inherit'
```

**Key files involved:**

| File | Role |
|------|------|
| [scripts/build-personas.js](scripts/build-personas.js) | Build engine — frontmatter templates, context assembly, model resolution |
| [personas/ledger/src/meta/_shared.yaml](personas/ledger/src/meta/_shared.yaml) | Ledger suite shared metadata (has `default_model` + legacy `cc_model`) |
| [personas/standalone/src/meta/_shared.yaml](personas/standalone/src/meta/_shared.yaml) | Standalone suite shared metadata (has `cc_model` only, no `default_model`) |
| [personas/docs/agents/project-manifest/api-surface.md](personas/docs/agents/project-manifest/api-surface.md) | Documents template variables and YAML schema |
| [personas/docs/agents/project-manifest/data-flows.md](personas/docs/agents/project-manifest/data-flows.md) | Documents model resolution chain |
| [personas/docs/agents/project-manifest/constraints.md](personas/docs/agents/project-manifest/constraints.md) | Constraints 28b (default_model pattern) and 28c (cc_model resolution chain) |

**Frontmatter templates in `build-personas.js`** — four templates emit `model:` or `cc_model:` values:

- `FRONTMATTER_LEDGER_VSCODE` (~line 416) — `model: {{model}}` (unquoted)
- `FRONTMATTER_LEDGER_CC` (~line 429) — `model: {{cc_model}}` (unquoted)
- `FRONTMATTER_STANDALONE_VSCODE` (~line 448) — no model field
- `FRONTMATTER_STANDALONE_CC` (~line 458) — `model: {{cc_model}}` (unquoted)

**Context object** (~line 607–636) — the dead assignment `cc_model: sharedMeta.cc_model` at line 611 is overridden first by the persona spread (`...persona`) and then explicitly by `cc_model: ccModel` at line 618.

## Approach / Architecture

Four surgical changes, each independently verifiable:

1. **Remove dead `cc_model` assignment** from the context object in `buildForTarget()`.
2. **Add defensive single-quoting** to `model:` / `cc_model:` values in all four frontmatter templates.
3. **Remove the legacy `cc_model` field** from `personas/ledger/src/meta/_shared.yaml`. The fallback chain still works without it because `default_model` is set.
4. **Introduce `default_model` to the standalone suite** and remove `cc_model` from its `_shared.yaml`, bringing both suites to the same pattern. Update the `cc_model` resolution in the build script to remove the legacy `cc_model` fallback entirely, simplifying the chain to `persona.model → sharedMeta.default_model → 'inherit'`.

## Rationale

- **Item 1** eliminates a dead assignment that obscures intent and was flagged by three independent agents.
- **Item 2** is defensive programming: current model names are YAML-safe, but future names containing colons, brackets, or other special characters would silently produce invalid frontmatter. Single-quoting is consistent with how `name:` and `description:` are already rendered in the templates.
- **Item 3** removes dead configuration from the ledger suite now that `default_model` is the canonical source of truth.
- **Item 4** is the follow-on consideration from the synthesis. It completes the migration by bringing the standalone suite onto the same `default_model` pattern, after which `cc_model` is no longer needed anywhere. This eliminates the legacy fallback path entirely, reducing cognitive load and removing a potential source of confusion.

## Detailed Steps

### Step 1 — Remove dead `cc_model` assignment (build script)

In [scripts/build-personas.js](scripts/build-personas.js) ~line 611, remove the line:
```javascript
cc_model:           sharedMeta.cc_model,
```
from the context object. The computed `ccModel` value at line 618 is the only assignment that matters.

### Step 2 — Add defensive quoting to frontmatter templates (build script)

In [scripts/build-personas.js](scripts/build-personas.js), update the four frontmatter template strings:

| Template | Current | New |
|----------|---------|-----|
| `FRONTMATTER_LEDGER_VSCODE` (~line 425) | `model: {{model}}` | `model: '{{model}}'` |
| `FRONTMATTER_LEDGER_CC` (~line 438) | `model: {{cc_model}}` | `model: '{{cc_model}}'` |
| `FRONTMATTER_STANDALONE_CC` (~line 464) | `model: {{cc_model}}` | `model: '{{cc_model}}'` |

### Step 3 — Remove legacy `cc_model` from ledger `_shared.yaml`

In [personas/ledger/src/meta/_shared.yaml](personas/ledger/src/meta/_shared.yaml), remove the line:
```yaml
cc_model: "inherit"                  # Legacy fallback; cc_model is now derived from model/default_model
```

### Step 4 — Introduce `default_model` to standalone `_shared.yaml`

In [personas/standalone/src/meta/_shared.yaml](personas/standalone/src/meta/_shared.yaml):
- Add `default_model: "inherit"` (standalone personas deliberately defer model selection to the user, so `inherit` is the correct value).
- Remove the `cc_model: "inherit"` line.

### Step 5 — Simplify the fallback chain in the build script

In [scripts/build-personas.js](scripts/build-personas.js) ~line 553, simplify the model resolution:

**Current:**
```javascript
const model = persona.model !== undefined
  ? persona.model
  : (sharedMeta.default_model || sharedMeta.cc_model || 'inherit');
```

**New:**
```javascript
const model = persona.model !== undefined
  ? persona.model
  : (sharedMeta.default_model || 'inherit');
```

And ~line 555, simplify `ccModel` resolution:

**Current:**
```javascript
const ccModel = persona.cc_model !== undefined
  ? persona.cc_model
  : model;
```

This can remain unchanged — the `persona.cc_model` escape hatch is still useful if a persona ever needs a different model for Claude Code specifically. No persona currently sets it, but the flexibility costs nothing.

### Step 6 — Rebuild and verify

1. Run `node scripts/build-personas.js --suite all --strict` — must exit 0.
2. Inspect at least one generated VS Code and one Claude Code file from each suite to confirm that the `model:` value is now single-quoted.
3. Run `node scripts/build-personas.js --check` — must confirm no stale output.
4. Run `node scripts/check-known-roles.js` — must pass.

### Step 7 — Update manifest documentation

Update three manifest files to reflect the changes:

- **[personas/docs/agents/project-manifest/api-surface.md](personas/docs/agents/project-manifest/api-surface.md):**
  - Remove `cc_model` from the `_shared.yaml` schema section (for the ledger suite).
  - Add `default_model` to the standalone `_shared.yaml` schema section (if documented).
  - Update the `FRONTMATTER_LEDGER_VSCODE` template listing to show `model: '{{model}}'` (quoted).
  - Update the `FRONTMATTER_LEDGER_CC` and `FRONTMATTER_STANDALONE_CC` template listings to show `model: '{{cc_model}}'` (quoted).

- **[personas/docs/agents/project-manifest/data-flows.md](personas/docs/agents/project-manifest/data-flows.md):**
  - Update the model resolution chain to remove the `sharedMeta.cc_model` fallback step.
  - New chain: `persona.model → sharedMeta.default_model → 'inherit'`

- **[personas/docs/agents/project-manifest/constraints.md](personas/docs/agents/project-manifest/constraints.md):**
  - Update constraint 28c to remove the `sharedMeta.cc_model` fallback from the resolution chain.
  - Note that the standalone suite now uses `default_model` instead of `cc_model`.

## Dependencies

- Steps 1–5 are code changes that can be done in any order but should all be completed before Step 6.
- Step 6 (verification) must follow all code changes.
- Step 7 (manifest updates) should follow Step 6 to ensure documentation matches verified behavior.

## Required Components

- [scripts/build-personas.js](scripts/build-personas.js) — modified (Steps 1, 2, 5)
- [personas/ledger/src/meta/_shared.yaml](personas/ledger/src/meta/_shared.yaml) — modified (Step 3)
- [personas/standalone/src/meta/_shared.yaml](personas/standalone/src/meta/_shared.yaml) — modified (Step 4)
- [personas/docs/agents/project-manifest/api-surface.md](personas/docs/agents/project-manifest/api-surface.md) — modified (Step 7)
- [personas/docs/agents/project-manifest/data-flows.md](personas/docs/agents/project-manifest/data-flows.md) — modified (Step 7)
- [personas/docs/agents/project-manifest/constraints.md](personas/docs/agents/project-manifest/constraints.md) — modified (Step 7)

## Assumptions

- The standalone suite's intended behavior for `model` is `inherit` (defer to user's configured model). This matches the current `cc_model: "inherit"` value and is confirmed by its inline comment.
- No per-persona YAML files in either suite currently set `cc_model`, so removing the `sharedMeta.cc_model` fallback has no behavioral impact.
- The `FRONTMATTER_STANDALONE_VSCODE` template intentionally has no `model:` field, and this plan does not change that.

## Constraints

- All persona source changes must flow through the Edit → Build → Sync workflow (constraint 3 in [personas/docs/agents/project-manifest/constraints.md](personas/docs/agents/project-manifest/constraints.md)).
- Generated files in `personas/*/vs-code/` and `personas/*/claude-code/` must never be edited directly (constraint 1).
- The `persona.cc_model` per-persona override must be preserved in the build script for forward compatibility, even though no persona currently uses it.

## Out of Scope

- Adding a `model:` field to `FRONTMATTER_STANDALONE_VSCODE`. The standalone VS Code template intentionally omits it.
- Changing the actual model values (e.g., switching agents between Opus and Sonnet).
- Modifying the `personas/ledger/README.md` user-facing workflow guide.
- Any changes to the MCP server sub-project.

## Acceptance Criteria

- `node scripts/build-personas.js --suite all --strict` exits 0.
- `node scripts/build-personas.js --check` confirms no stale output.
- `node scripts/check-known-roles.js` passes.
- All generated ledger VS Code files contain `model: 'Claude ...'` (single-quoted) in frontmatter.
- All generated Claude Code files (both suites) contain `model: 'inherit'` or `model: 'Claude ...'` (single-quoted) in frontmatter.
- The dead `cc_model: sharedMeta.cc_model` line no longer exists in the context object.
- Neither `_shared.yaml` file contains a `cc_model` field.
- The model resolution in `build-personas.js` no longer references `sharedMeta.cc_model`.
- Manifest files (`api-surface.md`, `data-flows.md`, `constraints.md`) reflect the simplified resolution chain.

## Testing Strategy

1. **Build verification:** `--suite all --strict` ensures all templates resolve cleanly with zero unresolved markers.
2. **Freshness check:** `--check` confirms generated output matches source truth.
3. **Role parity:** `check-known-roles.js` ensures no cross-system drift.
4. **Manual spot-check:** Inspect representative generated files from both suites and both IDE targets to confirm correct quoting.
5. **Negative test:** Verify that no generated file contains a bare (unquoted) `model:` value by grepping: `grep -rn '^model: [^'\'']' personas/*/vs-code/ personas/*/claude-code/`.

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Standalone suite personas break after removing `cc_model`** | Adding `default_model: "inherit"` to standalone `_shared.yaml` (Step 4) ensures the fallback still resolves to `inherit`. Build verification (Step 6) catches any failure immediately. |
| **Single-quoting introduces unexpected escaping for model names containing `'`** | Current and foreseeable model names do not contain single quotes. If they ever do, the fix is to switch to double-quoting — a trivial template change. |
| **Per-persona `cc_model` override path silently broken** | The `ccModel` resolution logic (Step 5) preserves the `persona.cc_model` check. No behavioral change occurs for this path. |
| **Manifest documentation diverges from implementation** | Step 7 explicitly updates all three affected manifest files. The verification in Step 6 runs before documentation, ensuring docs reflect tested behavior. |
