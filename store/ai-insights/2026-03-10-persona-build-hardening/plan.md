# Plan

## Summary

Address four strategic recommendations from the `2026-03-10-persona-model-field-rework-1` synthesis. The changes harden the persona build system by: (1) defaulting `--suite` to `all` so that both suites are always built and checked unless explicitly narrowed, (2) extending `--check` coverage to standalone personas (achieved as a direct consequence of item 1), (3) adding an inline comment to the `ccModel` escape hatch to prevent accidental removal, and (4) introducing a template-composition helper to eliminate verbatim duplication between CC frontmatter templates.

Synthesis recommendation 5 (raw string interpolation has no YAML escaping) is a pre-existing limitation tracked in this plan's Assumptions section but not addressed with code changes — the risk is negligible given the current model name corpus.

## Architectural Context

The persona build system ([scripts/build-personas.js](scripts/build-personas.js)) is the central engine that assembles persona Markdown files from YAML metadata and content templates. It supports two suites (`ledger`, `standalone`) and two IDE targets (`vscode`, `claude-code`).

**Current `--suite` default behavior (line 94):**
```javascript
let SUITE_ARG = 'ledger';
```
When invoked without `--suite`, only the 14 ledger output files are built/checked. The 20 standalone files are silently skipped. This default propagates through `--check` mode and through the pre-commit hook (`.githooks/pre-commit`), which runs `node scripts/build-personas.js --check` without `--suite`.

**Template duplication:** `FRONTMATTER_LEDGER_CC` (line 429) and `FRONTMATTER_STANDALONE_CC` (line 456) share four verbatim fields: `permissionMode`, `model`, `memory`, and framing `---` delimiters. As template count or field count grows, drift between these duplicates becomes a maintenance risk.

**Key files involved:**

| File | Role |
|------|------|
| [scripts/build-personas.js](scripts/build-personas.js) | Build engine — `--suite` default, frontmatter templates, model resolution |
| [.githooks/pre-commit](.githooks/pre-commit) | Pre-commit freshness guard (runs `build-personas.js --check`) |
| [personas/docs/agents/project-manifest/constraints.md](personas/docs/agents/project-manifest/constraints.md) | Constraints 3, 43 (document `--suite` default and pre-commit scope) |
| [personas/docs/agents/project-manifest/api-surface.md](personas/docs/agents/project-manifest/api-surface.md) | Documents CLI flags, template variables, and frontmatter templates |
| [personas/docs/agents/project-manifest/data-flows.md](personas/docs/agents/project-manifest/data-flows.md) | Documents build pipeline phases |

**Upstream callers of `build-personas.js`:**
- `.githooks/pre-commit` — runs `node scripts/build-personas.js --check` (no `--suite` flag)
- `scripts/sync-personas.js` (line 498) — runs with `--suite ledger,standalone` (already covers both suites; unaffected by this change)

## Approach / Architecture

Four changes, grouped into three logical units:

1. **Default-to-all + console note** — Change the `SUITE_ARG` default from `'ledger'` to `'all'`. Emit an informational line at startup when the default is used, explaining how to narrow with `--suite ledger` or `--suite standalone`. This single change also resolves synthesis recommendation 2: the pre-commit hook and bare `--check` invocations now cover all 34 output files without any hook modification.

2. **Inline comment on `ccModel` ternary** — Add a one-line JSDoc-style comment above the `ccModel` assignment explaining its purpose as a per-persona CC model override escape hatch.

3. **CC frontmatter partial helper** — Extract the shared CC-specific fields (`permissionMode`, `model`, `memory`) into a helper function that returns a multi-line string fragment. Both `FRONTMATTER_LEDGER_CC` and `FRONTMATTER_STANDALONE_CC` call this helper instead of duplicating the fields verbatim. This keeps the templates readable while eliminating drift risk.

## Rationale

- **Defaulting to `all`** aligns the build script with contributor expectations. Every contributor who reported the silent-skip behavior expected a full build. The `--suite ledger` escape hatch remains for focused work. The informational note at startup ensures contributors are never confused about which suites are being processed.
- **The pre-commit hook needs no modification.** It runs `node scripts/build-personas.js --check` without `--suite`. Once the default changes, it automatically covers both suites. This eliminates the "Ledger-only scope" caveat from constraint 43.
- **The inline comment** costs nothing and prevents a recurring flag across agent sessions where the `ccModel` ternary is mistaken for dead code.
- **The template helper** is the minimal-intervention approach to deduplication. A full partial/fragment system would be over-engineered for three fields. A function returning a string fragment integrates cleanly with the existing template-string approach.

## Detailed Steps

### Step 1 — Change `--suite` default to `all`

In [scripts/build-personas.js](scripts/build-personas.js) line 94, change:
```javascript
let SUITE_ARG = 'ledger';
```
to:
```javascript
let SUITE_ARG = 'all';
```

### Step 2 — Add informational note when default is used

After the `--suite` flag parsing block (~line 112, after the closing brace of the `if (suiteArgIdx !== -1)` block), add a note that prints when no `--suite` flag was provided:

```javascript
if (suiteArgIdx === -1) {
  console.log('[info] No --suite flag provided — building all suites (ledger + standalone). Use --suite ledger or --suite standalone to narrow.');
}
```

### Step 3 — Update header comment

In [scripts/build-personas.js](scripts/build-personas.js) lines 12–15, update the usage comment to reflect the new default:

**Current:**
```javascript
 *   node scripts/build-personas.js                              # build ledger (default)
 *   node scripts/build-personas.js --suite standalone           # standalone suite only
 *   node scripts/build-personas.js --suite all                  # both suites (ledger + standalone)
 *   node scripts/build-personas.js --suite ledger,standalone    # comma-separated list
```

**New:**
```javascript
 *   node scripts/build-personas.js                              # build all suites (default)
 *   node scripts/build-personas.js --suite ledger               # ledger suite only
 *   node scripts/build-personas.js --suite standalone           # standalone suite only
 *   node scripts/build-personas.js --suite ledger,standalone    # comma-separated list (equivalent to --suite all)
```

### Step 4 — Update the `--suite` inline comment

In [scripts/build-personas.js](scripts/build-personas.js) line 90, update:

**Current:**
```javascript
// --suite flag: ledger | standalone | all (default: ledger)
```

**New:**
```javascript
// --suite flag: ledger | standalone | all (default: all)
```

### Step 5 — Add inline comment on `ccModel` ternary

In [scripts/build-personas.js](scripts/build-personas.js) above line 555, add:

```javascript
    // Per-persona CC model override: if persona.cc_model is set, it wins;
    // otherwise falls back to the common model value. No persona currently
    // uses this, but the escape hatch is preserved for forward compatibility.
```

### Step 6 — Extract CC frontmatter partial helper

In [scripts/build-personas.js](scripts/build-personas.js), above the frontmatter template constants (~line 415), add a helper function:

```javascript
/**
 * Shared CC-specific frontmatter fields.
 * Used by both FRONTMATTER_LEDGER_CC and FRONTMATTER_STANDALONE_CC
 * to avoid verbatim duplication of these three fields.
 *
 * @returns {string} Multi-line YAML fragment (no leading/trailing newline)
 */
function ccFrontmatterFields() {
  return `permissionMode: {{cc_permission_mode}}
model: '{{cc_model}}'
memory: {{cc_memory}}`;
}
```

Then update `FRONTMATTER_LEDGER_CC` to use the helper. Replace the three verbatim lines:
```
permissionMode: {{cc_permission_mode}}
model: '{{cc_model}}'
memory: {{cc_memory}}
```
with:
```
${ccFrontmatterFields()}
```

Apply the same replacement to `FRONTMATTER_STANDALONE_CC`.

**Important:** The `FRONTMATTER_*` constants must become template literals that invoke the helper via `${}` interpolation. They are already template literals (backtick-delimited), so this is a drop-in replacement.

### Step 7 — Rebuild and verify

1. Run `node scripts/build-personas.js --strict` (no `--suite` — validates the new default). Must exit 0 with 34 personas built.
2. Run `node scripts/build-personas.js --check` — must confirm no stale output for all suites.
3. Run `node scripts/check-known-roles.js` — must pass.
4. Verify the startup info line appears: `[info] No --suite flag provided — building all suites...`
5. Run `node scripts/build-personas.js --suite ledger --check` — verify the narrowing still works.
6. Run a **diff check** on a representative generated CC file from each suite to confirm the helper produces identical output to the previous verbatim templates.

### Step 8 — Update manifest documentation

Update the following manifest files:

- **[personas/docs/agents/project-manifest/constraints.md](personas/docs/agents/project-manifest/constraints.md):**
  - **Constraint 3:** Remove the "Ledger-default pitfall" blockquote. Update the first sentence to reflect the new default. Optionally add a note that `--suite ledger` or `--suite standalone` can be used to narrow the build scope.
  - **Constraint 43:** Remove the "Ledger-only scope" blockquote. The pre-commit hook now covers all suites by default.

- **[personas/docs/agents/project-manifest/api-surface.md](personas/docs/agents/project-manifest/api-surface.md):**
  - Update the CLI flags table to show `--suite` default as `all` instead of `ledger`.
  - Document the new `ccFrontmatterFields()` helper function.
  - Update the `FRONTMATTER_LEDGER_CC` and `FRONTMATTER_STANDALONE_CC` template listings to show `${ccFrontmatterFields()}` instead of verbatim fields.

- **[personas/docs/agents/project-manifest/data-flows.md](personas/docs/agents/project-manifest/data-flows.md):**
  - If the build pipeline phase documentation references the default suite, update to reflect `all`.

## Dependencies

- Steps 1–6 are code changes that can be done in any order but should all complete before Step 7.
- Step 7 (verification) must follow all code changes.
- Step 8 (manifest updates) should follow Step 7 to document verified behavior.

## Required Components

- [scripts/build-personas.js](scripts/build-personas.js) — modified (Steps 1–6)
- [personas/docs/agents/project-manifest/constraints.md](personas/docs/agents/project-manifest/constraints.md) — modified (Step 8)
- [personas/docs/agents/project-manifest/api-surface.md](personas/docs/agents/project-manifest/api-surface.md) — modified (Step 8)
- [personas/docs/agents/project-manifest/data-flows.md](personas/docs/agents/project-manifest/data-flows.md) — modified (Step 8, if applicable)
- No new files created.

## Assumptions

- The standalone suite is mature enough that defaulting to `all` will not cause CI slowdowns. Current build time for all 34 personas is sub-second.
- The `sync-personas.js` script already passes `--suite ledger,standalone` explicitly (line 498) and is unaffected by the default change.
- No external CI pipeline depends on the `--suite ledger` default behavior. The only automated consumer is the pre-commit hook.
- Synthesis recommendation 5 (raw string interpolation lacks YAML escaping) is a pre-existing limitation. Current and foreseeable Anthropic model names do not contain YAML-special characters. This is tracked as a known limitation, not addressed with code changes.

## Constraints

- All persona source changes must flow through the Edit → Build → Sync workflow (constraint 3).
- Generated files must never be edited directly (constraint 1).
- The `ccFrontmatterFields()` helper must produce byte-identical output to the current verbatim fields — no whitespace or ordering changes.
- The informational note must go to `console.log` (not `console.warn`) since it's not a warning — it's the expected default behavior.

## Out of Scope

- Modifying `.githooks/pre-commit`. The hook needs no changes — it benefits from the new default automatically.
- Modifying `scripts/sync-personas.js`. It already passes `--suite ledger,standalone` explicitly.
- Adding YAML escaping to the template engine (synthesis recommendation 5). Tracked as known limitation.
- Any changes to the MCP server sub-project.
- Adding a `--check-all` alias (the `--check` flag now covers all suites by default, making a separate alias unnecessary).

## Acceptance Criteria

- `node scripts/build-personas.js --strict` (without `--suite`) exits 0 and builds 34 personas.
- `node scripts/build-personas.js --check` (without `--suite`) confirms no stale output across all 34 files.
- `node scripts/check-known-roles.js` passes.
- Running without `--suite` prints an `[info]` line explaining the default behavior.
- `node scripts/build-personas.js --suite ledger` still narrows to 14 ledger files only.
- The `ccModel` ternary in `build-personas.js` has a multi-line comment explaining its purpose.
- `FRONTMATTER_LEDGER_CC` and `FRONTMATTER_STANDALONE_CC` use `${ccFrontmatterFields()}` instead of verbatim fields.
- A diff of generated CC output before/after confirms byte-identical content (the helper is a refactor, not a behavior change).
- Constraint 3 no longer warns about the "Ledger-default pitfall".
- Constraint 43 no longer has the "Ledger-only scope" caveat.
- `api-surface.md` documents the `ccFrontmatterFields()` helper and the updated `--suite` default.

## Testing Strategy

1. **Build verification:** `--strict` without `--suite` to confirm both suites build cleanly.
2. **Freshness check:** `--check` without `--suite` to confirm all 34 files are fresh.
3. **Role parity:** `check-known-roles.js` to ensure no cross-system drift.
4. **Narrowing regression:** `--suite ledger` must still produce only 14 files.
5. **Diff check:** Compare generated CC output before/after the `ccFrontmatterFields()` refactor to confirm zero behavioral change.
6. **Info line verification:** Confirm the `[info]` line appears when no `--suite` is passed and does not appear when `--suite` is explicit.

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Pre-commit hook becomes slower with all 34 files** | Current build is sub-second for all suites; negligible impact. Monitor if suite count grows significantly. |
| **Existing CI scripts depend on `--suite ledger` default** | The only automated consumer is the pre-commit hook, which benefits from the change. `sync-personas.js` uses explicit `--suite` and is unaffected. |
| **`ccFrontmatterFields()` helper produces subtly different output** | Acceptance criteria require a byte-identical diff check on generated output. The helper is tested via the existing `--strict` and `--check` modes. |
| **Contributors confused by `--suite all` as default** | The `[info]` startup line makes the behavior explicit and documents how to narrow. The header comment lists all options. |
