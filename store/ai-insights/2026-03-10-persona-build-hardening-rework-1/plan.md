# Plan

## Summary

Address the three actionable strategic recommendations surfaced during the `2026-03-10-persona-build-hardening` synthesis. All items are low-priority documentation and code-comment hygiene fixes: (1) correct the stale `build-personas.js` description in root `AGENTS.md`, (2) formalize a log-prefix severity convention in the personas constraints manifest, and (3) add a `@note` to the `ccFrontmatterFields()` JSDoc documenting the monomorphism assumption and suite-divergence risk.

## Architectural Context

- **Root `AGENTS.md`** ([AGENTS.md](AGENTS.md)) — workspace-wide agent operating manual. Line 249 contains a "Root-Level Tooling" table describing `scripts/build-personas.js` as _"Assemble 7 ledger persona files from `personas/ledger/src/` templates"_. This is stale: the script now builds 34 personas across ledger + standalone suites and two IDE targets.
- **Personas constraints manifest** ([personas/docs/agents/project-manifest/constraints.md](personas/docs/agents/project-manifest/constraints.md)) — authoritative rules for the persona template system. Currently contains no log-prefix convention, though the build script already uses `[ERROR]`, `[WARN]`, `[STRICT]`, and `[info]` prefixes.
- **`ccFrontmatterFields()`** in [scripts/build-personas.js](scripts/build-personas.js) (line 426) — pure helper returning three shared CC frontmatter fields. Its JSDoc describes what it does but does not document the implicit assumption that ledger and standalone CC frontmatter remain identical.

## Approach / Architecture

Three independent, documentation-only changes touching three files. No runtime behaviour changes. Each can be implemented and verified in isolation.

## Rationale

These are direct follow-ups from the synthesis Gold Nuggets (§1, §2, §3). Fixing them now prevents documentation drift from compounding and gives future agents accurate mental models for log severity and frontmatter divergence risk.

## Detailed Steps

### Step 1 — Fix root `AGENTS.md` build script description

In [AGENTS.md](AGENTS.md), update the Root-Level Tooling table row for `scripts/build-personas.js`:

**Before:**
```
| `scripts/build-personas.js` | Assemble 7 ledger persona files from `personas/ledger/src/` templates |
```

**After:**
```
| `scripts/build-personas.js` | Assemble 34 persona files (7 ledger + 10 standalone × 2 IDE targets) from `personas/ledger/src/` and `personas/standalone/src/` templates |
```

### Step 2 — Formalize log-prefix severity convention

In [personas/docs/agents/project-manifest/constraints.md](personas/docs/agents/project-manifest/constraints.md), add a new section after the existing "Template Engine Limitations" section (after constraint 9's notes):

Add a **"Log-Prefix Convention"** section documenting the four severity levels already in use:

| Prefix | Meaning | Example usage |
|--------|---------|---------------|
| `[info]` | Informational — runtime context, no action needed | Suite default announcement at startup |
| `[WARN]` | Warning — recoverable issue, output may still be valid | Unresolved template markers (non-strict mode) |
| `[STRICT]` | Strict-mode failure — gates CI exit code | Unresolved markers when `--strict` is active |
| `[ERROR]` | Fatal — build cannot continue | Missing content file, invalid YAML |

Instruct future contributors to use these prefixes consistently for all `console.log` / `console.error` output.

### Step 3 — Add `@note` to `ccFrontmatterFields()` JSDoc

In [scripts/build-personas.js](scripts/build-personas.js) at the `ccFrontmatterFields()` function (line 426), extend the existing JSDoc block with a `@note` tag:

```javascript
/**
 * Shared CC-specific frontmatter fields.
 * Used by both FRONTMATTER_LEDGER_CC and FRONTMATTER_STANDALONE_CC
 * to avoid verbatim duplication of these three fields.
 *
 * @note This helper is intentionally monomorphic — it returns the same
 * fields regardless of suite context (ledger vs. standalone). If ledger
 * and standalone CC frontmatter ever diverge (e.g., different
 * permissionMode defaults, or a suite-specific field), this function
 * will need to accept a suite parameter or be split into per-suite
 * variants. See 2026-03-10-persona-build-hardening synthesis §3.
 *
 * @returns {string} Multi-line YAML fragment (no leading/trailing newline)
 */
```

## Dependencies

- None. All three steps are independent and touch different files.

## Required Components

- [AGENTS.md](AGENTS.md) (existing — edit)
- [personas/docs/agents/project-manifest/constraints.md](personas/docs/agents/project-manifest/constraints.md) (existing — edit)
- [scripts/build-personas.js](scripts/build-personas.js) (existing — edit)

## Assumptions

- The current persona count (7 ledger + 10 standalone = 17 personas × 2 IDE targets = 34 output files) is accurate as of 2026-03-10.
- The log prefixes `[info]`, `[WARN]`, `[STRICT]`, `[ERROR]` represent the complete set currently in use.

## Constraints

- Root `AGENTS.md` is a high-traffic agent reference — changes must be precise and factual.
- Personas constraints manifest follows existing section numbering — new content must integrate without disrupting numbering.
- `ccFrontmatterFields()` JSDoc must not change the function's behaviour or signature.

## Out of Scope

- Recommendation §4 (pre-existing `default_cc_model` inaccuracy) was already resolved during the original session.
- Any runtime changes to `build-personas.js`.
- Building or syncing personas (documentation-only plan).

## Acceptance Criteria

1. Root `AGENTS.md` build script description accurately reflects 34 personas, both suites, and both source directories.
2. `personas/docs/agents/project-manifest/constraints.md` contains a "Log-Prefix Convention" section with a table documenting `[info]`, `[WARN]`, `[STRICT]`, and `[ERROR]`.
3. `ccFrontmatterFields()` JSDoc includes a `@note` tag documenting the monomorphism assumption and referencing the synthesis recommendation.
4. `node scripts/build-personas.js` still runs without errors (no regression from JSDoc change).

## Testing Strategy

- **Step 1 & 2:** Manual review — verify the updated text is accurate and well-formatted.
- **Step 3:** Run `node scripts/build-personas.js` from the workspace root to confirm no regression from the JSDoc edit.
- All three files should render correctly in Markdown preview.

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Persona count changes before implementation** | Verify current count with `node scripts/build-personas.js` before finalising the AGENTS.md wording. |
| **Additional log prefixes missed** | Grep `build-personas.js` for `\[` bracket patterns to catch any unlisted prefixes. |
