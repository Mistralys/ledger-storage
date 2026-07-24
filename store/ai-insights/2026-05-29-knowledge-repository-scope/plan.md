# Plan

## Plan Audit Cycles
- Audits: 3 — Plan Auditor v1.4.0
- Architectural Reviews: 1 — Plan Architect Reviewer v1.5.0

## Summary

Replace the current `global` + `project` knowledge scope model with `global` + `repository`. Project-scoped insights (one file per plan slug) are redundant — plan-specific knowledge is already captured by the project synthesis and the Insights tab. The knowledge store should hold codebase-level knowledge (repository scope) and cross-codebase knowledge (global scope). A file like `2026-02-27-perceval-phase1-insights.json` becomes `hcp-editor-insights.json` — knowledge about the hcp-editor repository, discoverable by any future plan touching that repo. The `origin_plan` field is added as optional provenance metadata on repository-scoped insights, allowing the GUI to link back to the originating plan. No migration of data is necessary, as there are no project storage files present yet.

---

## Architectural Context

The knowledge subsystem is self-contained across four layers:

| Layer | Files |
|-------|-------|
| Schema (types + validation) | `src/schema/knowledge.ts` — `InsightScope`, `InsightSchema`, `KnowledgeStoreSchema`, `PROJECT_SLUG_REGEX` |
| Storage (CRUD + file I/O) | `src/storage/knowledge-store.ts` — `KnowledgeStoreManager` |
| MCP tools (agent-facing API) | `src/tools/knowledge.ts` — `ledger_add_insight`, `ledger_search_insights`, `ledger_list_insights`, `ledger_update_insight` |
| GUI REST API + frontend | `gui/api-knowledge.ts` (handlers + Zod schemas), `gui/server.ts` (route wiring), `gui/public/views/knowledge.js` (SPA view) |

**Current storage layout** (relative to `ledgerRoot`):

```
.knowledge/
  .lock
  global-insights.json          ← scope: 'global'
  {slug}-insights.json          ← scope: 'project', one file per project slug
```

**Scope enum**: `z.enum(['global', 'project'])`.

**Key invariant (constraint 1 in constraints.md)**: all file writes must go through `atomicWriteJson()`; all read-modify-write sequences must be wrapped in `withLock(knowledgeDir(), …)`.

**GUI Knowledge page**: renders two tabs — "Global" and "Project". Client-side filtering by category, project slug, and query. Edit, Delete, Promote to Global, and Move to Project actions per card.

---

## Approach / Architecture

### New storage layout

```
.knowledge/
  .lock
  global-insights.json             ← scope: 'global' (unchanged)
  {repo-name}-insights.json        ← scope: 'repository' (replaces project stores)
```

**Key decisions:**
- Repository stores live **flat** in `.knowledge/` (same location as current project stores). This is viable because we are *removing* project stores, not coexisting with them. No enumeration ambiguity arises.
- The resulting filename is exactly what the user wants: `hcp-editor-insights.json`.
- The existing `global-insights.json` filename has a well-known fixed name and cannot collide with a repository name (it would require a repository called `global`, which is a valid slug — guarded by rejecting `'global'` as a repository name at the storage layer).

### Schema changes

1. **Replace** `InsightScope` with `z.enum(['global', 'repository'])`.
2. **Remove** `project_slug` from `InsightSchema` as a scope-discriminator field.
3. **Add** `repository_name: z.string().regex(PROJECT_SLUG_REGEX)` to `InsightSchema` — optional in the schema, enforced by storage layer when `scope === 'repository'`.
4. **Add** `origin_plan: z.string().regex(PROJECT_SLUG_REGEX).optional()` as **provenance metadata** — records which plan originally discovered this insight. Not used for storage routing. Optional for both scopes.

### Storage changes (`KnowledgeStoreManager`)

**Renamed/repurposed public helpers:**
- `projectStorePath(slug)` → **removed**
- `repositoryStorePath(repoName: string): string` → `join(knowledgeDir(), '{repoName}-insights.json')` (guards: `_validateSlug(repoName)` + reject `'global'` reserved name)
- `readProjectStore(slug)` → **removed**
- `writeProjectStore(slug, data)` → **removed**
- `readRepositoryStore(repoName: string): Promise<KnowledgeStore>` — **new**
- `writeRepositoryStore(repoName: string, data: KnowledgeStore): Promise<void>` — **new** (acquires own lock — top-level only)

**Updated private helpers:**
- `_enumerateProjectStorePaths()` → **removed**
- `_enumerateRepositoryStorePaths(): Promise<string[]>` — **new**: scans `knowledgeDir()` for `*-insights.json`, excludes `global-insights.json`
- `_enumerateStorePaths()` — returns `[globalStorePath(), ...repositoryStorePaths]`
- `_storePathsForFilter()` — simplified (no project-slug routing):

| scope | repository_name | Stores searched |
|-------|-----------------|-----------------|
| `'global'` | — | `global-insights.json` |
| `'repository'` | present | `{name}-insights.json` |
| `'repository'` | absent | all repository stores |
| absent | present | `{name}-insights.json` |
| absent | absent | global + all repository stores |

- `_loadInsights()` — same selection rules as above + category filter post-load
- `addInsight()` — handle `scope === 'repository'` (requires `repository_name`); resolve store path via `repositoryStorePath()`; no longer handles `scope === 'project'`
- `moveInsight()` — only moves between `global` ↔ `repository` and `repository` ↔ `repository`
- `updateInsight()`, `deleteInsight()` — unchanged logic, just use the new `_storePathsForFilter()`

### MCP tool changes (`src/tools/knowledge.ts`)

- `AddInsightSchema`:
  - `scope`: change enum to `['global', 'repository']`; update description
  - Add `repository_name` (optional, validated by `PROJECT_SLUG_REGEX`)
  - Add `origin_plan` (optional, provenance metadata — "which plan originally discovered this?")
  - Remove `project_slug` store-routing usage entirely (scope enum no longer includes 'project')
- `SearchInsightsSchema`: replace `project_slug` filter with `repository_name`; update `scope` enum
- `ListInsightsSchema`: replace `project_slug` filter with `repository_name`; update `scope` enum
- `UpdateInsightSchema`: replace `project_slug` filter with `repository_name`; update `scope` enum
- All four handler functions: forward `repository_name` to storage calls; remove project-slug store routing

### GUI API changes (`gui/api-knowledge.ts`)

- `KnowledgeListParams`: replace `project_slug?: string` with `repository_name?: string`
- `KnowledgeUpdateBodySchema`: replace `project_slug` with `repository_name`; update scope enum
- `KnowledgeMoveBodySchema` — simplified (no project stores):

```typescript
KnowledgeMoveBodySchema = z.object({
  source_scope: InsightScope,                                // 'global' | 'repository'
  source_repository_name: z.string().regex(…).optional(),   // required if source_scope === 'repository'
  target_repository_name: z.string().regex(…),              // destination repository (move always targets a repo)
}).strict();
```

Move is now: `global → repository` or `repository → repository`. Moving to global is done via `promote`.

- `handleListKnowledge()`: parse `repository_name` from query string
- `handleDeleteKnowledge()`: accept `scope === 'repository'`; require `repository_name`
- `handlePromoteKnowledge()`: accept `scope === 'repository'` (only valid source; global-to-global is rejected)
- `handleMoveKnowledge()`: move from `source_scope` → `target_repository_name`
- `handleUpdateKnowledge()`: pass `repository_name` for store disambiguation

### GUI frontend changes (`gui/public/views/knowledge.js`)

1. **Replace tabs**: "Global" | "Repository" (drop the "Project" tab entirely).
2. Replace `filterProject` with `filterRepository` state variable.
3. Update `applyFilters()` to handle `repository` scope.
4. Update `getDistinctValues()` to collect `repository_name` values.
5. Update `buildFilterBarHtml()` — show "Repository" dropdown on the repository tab.
6. Update `buildKnowledgeHtml()` — render `repository_name` on cards; render `origin_plan` as a provenance link (clickable, navigates to `#/projects/{slug}`).
7. "Promote to Global" button on repository-scoped insights.
8. "Move" action: move form shows a `target_repository_name` text input (simple free-text, since the set of repository stores is open-ended and not pre-enumerable).

### Help content changes (`src/tools/help-content.ts`)

Update all four knowledge tool help strings:
- Replace `"project"` scope description with `"repository"`
- Replace `project_slug` store-routing parameter with `repository_name`
- Document `origin_plan` as optional provenance metadata

### Persona knowledge-collection partial

Update `personas/shared/partials/synthesis-knowledge-collection.md` — replace the scope table with two rows (global + repository); add guidance on `origin_plan` provenance.

### Migration

No migration needed. Existing project-scoped `{slug}-insights.json` files have been manually deleted (they were mostly empty). The new code simply does not create or read project-scoped stores.

---

## Rationale

**Why drop project scope entirely?** Plan-bounded knowledge is already captured by: (a) the project synthesis report (human-readable summary of decisions, pivots, and outcomes), and (b) the pipeline observations visible in the Insights tab. Duplicating this in the knowledge store adds no retrieval value — agents searching the knowledge store want codebase truths, not plan logistics.

**Why flat in `.knowledge/` instead of a `repos/` subdirectory?** With project stores removed, there is no naming collision. All `*-insights.json` files (except `global-insights.json`) are repository stores. The enumeration logic is trivially `endsWith('-insights.json') && !== 'global-insights.json'`. A subdirectory would add path complexity for no benefit.

**Why add `origin_plan` as provenance?** It provides traceability — the GUI can show "Discovered in plan 2026-02-27-perceval-phase1" with a clickable link to the project detail page. This is useful for understanding *when* and *why* an insight was recorded without needing to trawl synthesis reports.

**Why reject `'global'` as a repository name?** The `global-insights.json` filename is reserved. A repository named `"global"` would produce `global-insights.json`, colliding with the global store. Rejecting it at the storage layer is cleaner than special-casing it everywhere.

---

## Considered Alternatives

| Decision | Chosen Shape | Alternatives Considered | Trade-Off Summary |
|----------|--------------|-------------------------|-------------------|
| Scope model | `global` + `repository` (drop `project`) | Keep all three (`global` + `project` + `repository`) | Two-scope model is simpler; project-specific knowledge is already available via synthesis and Insights tab. |
| Repository store location | Flat in `.knowledge/` | `repos/` subdirectory | With project stores removed, flat is safe and produces the cleanest filenames. Subdirectory would be needed only if project stores coexisted. |
| `origin_plan` field | Optional provenance field (renamed from `project_slug`) | Keep `project_slug` name; remove entirely; use `source_plan` | With old files deleted, migration cost is zero. Renaming eliminates semantic overlap with `repository_name` in the same schema. |
| Move endpoint shape | `source_scope + source_repo → target_repo` | Generic `source → target_scope + target_name` | Only two move directions exist (global→repo, repo→repo). Promote handles repo→global. Simpler schema. |
| Reserved name guard | Reject `'global'` in `repositoryStorePath()` | Use a `repos/` subdirectory | Single reserved name is trivial to enforce; avoids subdirectory overhead. |
| Migration approach | No migration (files manually deleted) | Interactive script; automatic migration on start | Old files were mostly empty — manual deletion is simplest. No tooling overhead. |

---

## Pattern Alignment

| Pattern | Alignment |
|---------|-----------|
| `src/schema/knowledge.ts` — optional schema fields, storage-layer enforcement | Followed: `repository_name` is `z.string().optional()` in schema; `addInsight()` throws if absent when scope is `'repository'`. |
| `src/storage/knowledge-store.ts` — atomic writes via `atomicWriteJson()` | Followed: all repository store writes use `atomicWriteJson()` inside `withLock(knowledgeDir(), …)`. |
| `src/storage/knowledge-store.ts` — pure-read methods skip the lock | Followed: `readRepositoryStore()` does not acquire a lock. |
| `gui/api-knowledge.ts` — Zod `.strict()` schemas | Followed: updated `KnowledgeMoveBodySchema` keeps `.strict()`. |
| `gui/public/views/knowledge.js` — vanilla ES5 JS, no build step | Followed: all frontend changes use `var`, IIFE-style, no modern syntax. |
| `tests/gui/knowledge-api.test.ts` — real temp directories, no storage mocks | Followed: new tests use `mkdtemp`/`rm` and `KnowledgeStoreManager`. |


---

## Detailed Steps

1. **`src/schema/knowledge.ts`** — change `InsightScope` to `z.enum(['global', 'repository'])`; add `repository_name` field; rename `project_slug` to `origin_plan` with description "provenance metadata — which plan originally discovered this insight" (keep it optional on the schema).

2. **`src/storage/knowledge-store.ts`** — remove `projectStorePath()`, `readProjectStore()`, `writeProjectStore()`, `_enumerateProjectStorePaths()`; add `repositoryStorePath()`, `readRepositoryStore()`, `writeRepositoryStore()`, `_enumerateRepositoryStorePaths()`; add `'global'` reserved-name guard in `repositoryStorePath()`; update `addInsight()`, `moveInsight()`, `_storePathsForFilter()`, `_loadInsights()`, `_enumerateStorePaths()` to use repository logic; remove all project-scope branching.

3. **`src/tools/knowledge.ts`** — update all four schemas: change `scope` enum to `['global', 'repository']`; replace `project_slug` store-routing with `repository_name`; add `origin_plan` as optional provenance in `AddInsightSchema`; forward `repository_name` in all handlers.

4. **`src/tools/help-content.ts`** — rewrite four knowledge tool help strings for the new two-scope model.

5. **`gui/api-knowledge.ts`** — update `KnowledgeListParams`, `KnowledgeUpdateBodySchema`, redesign `KnowledgeMoveBodySchema`; update all five handler functions.

6. **`gui/server.ts`** — update three query-parameter extraction sites: (a) in the list route, change `project_slug: sp.get('project_slug')` → `repository_name: sp.get('repository_name')`; (b) in the delete route, rename the extracted variable and `sp.get` key from `project_slug` to `repository_name`; (c) same change in the promote route. Handler signatures are unchanged; only the query-string key names change.

7. **`gui/public/views/knowledge.js`** — replace "Project" tab with "Repository" tab; update filtering, rendering, and action forms; add provenance link rendering for `origin_plan`.

8. **`personas/shared/partials/synthesis-knowledge-collection.md`** — update scope documentation.

9. **`personas/standalone/src/content/knowledge-archiver.md`** — replace all occurrences of `scope: "project"` with `scope: "repository"`; replace `project_slug` usage with `repository_name`; update the scope guidance paragraph and the workflow step that references the removed scope. Also update any prose references to `` `project` `` scope (e.g., "for `project` insights" style sentences) that are not caught by a literal `scope: "project"` search.

10. **Tests** — see Test Plan section.

11. **Documentation** — see Documentation Updates section.

---

## Dependencies

- None on other work packages. The knowledge subsystem is self-contained within the MCP server.

---

## Required Components

**Modified files:**

| File | Change type |
|------|-------------|
| `mcp-server/src/schema/knowledge.ts` | Replace enum, add `repository_name`, rename `project_slug` to `origin_plan` |
| `mcp-server/src/storage/knowledge-store.ts` | Remove project methods, add repository methods, simplify routing |
| `mcp-server/src/tools/knowledge.ts` | Update four schemas + four handlers |
| `mcp-server/src/tools/help-content.ts` | Rewrite four help strings |
| `mcp-server/gui/api-knowledge.ts` | Update schemas + five handlers |
| `mcp-server/gui/server.ts` | Update query-string key names at three extraction sites |
| `mcp-server/gui/public/views/knowledge.js` | Replace Project tab with Repository tab |
| `personas/shared/partials/synthesis-knowledge-collection.md` | Update scope table |
| `personas/standalone/src/content/knowledge-archiver.md` | Replace `scope: "project"` with `scope: "repository"`; replace `project_slug` with `repository_name` |
| `mcp-server/tests/gui/api-knowledge.test.ts` | Update all `scope: 'project'` usages to `scope: 'repository'`; add `repository_name`; update Move/Promote schemas |
| `mcp-server/tests/gui/knowledge-api.test.ts` | Update all `scope: 'project'` / `project_slug` references (57 matches) |
| `mcp-server/tests/tools/knowledge.test.ts` | Update scope enum and `project_slug` references |
| `mcp-server/tests/storage/knowledge-store.test.ts` | Update scope enum and `project_slug` references (63 matches) |
| `mcp-server/tests/schema/knowledge.test.ts` | Update scope enum and field references |
| `mcp-server/tests/gui/server-knowledge-routes.test.ts` | Update query-string key assertions to `repository_name` |

**New files:**

| File | Purpose |
|------|---------|
| `mcp-server/tests/gui/knowledge-repository-scope.test.ts` | New test suite |

---

## Assumptions

- Repository names follow the same character set as project slugs (`^[a-zA-Z0-9][a-zA-Z0-9_-]*$`).
- `'global'` is the only reserved repository name.
- Existing project-scoped insight files have been manually deleted. No migration is required.
- The Synthesis and Knowledge Archiver persona partials are the only persona documents referencing knowledge tools.

---

## Constraints

- All file writes to repository stores must use `atomicWriteJson()` inside `withLock(knowledgeDir(), …)` — Constraint 1.
- Repository store paths constructed with `path.join()` — never hardcoded separators (cross-platform policy).
- `'global'` must be rejected as a repository name — prevents collision with `global-insights.json`.
- This is a **breaking change** to the knowledge system. No migration is needed (old files were manually deleted).
- `origin_plan` on insights is **provenance metadata only** — it must never be used for store routing or file-path derivation in the new system.

---

## Out of Scope

- Automatic repository name inference from `cwd_path` or `.meta.json`. Agents pass `repository_name` explicitly.
- Changes to the orchestrator — it does not call knowledge tools directly.
- Changes to the Insights tab or project synthesis — those remain the source of plan-specific knowledge.

---

## Acceptance Criteria

1. `ledger_add_insight` with `scope: 'repository'`, `repository_name: 'hcp-editor'`, and optional `origin_plan` creates `{ledgerRoot}/.knowledge/hcp-editor-insights.json` and returns the insight with correct fields.
2. `ledger_add_insight` with `scope: 'repository'` and no `repository_name` returns an error.
3. `ledger_add_insight` with `scope: 'repository'` and `repository_name: 'global'` returns an error (reserved name).
4. `ledger_list_insights` with no filters returns insights from both global and all repository stores.
5. `ledger_list_insights` with `scope: 'repository'` returns only repository-scoped insights.
6. `ledger_list_insights` with `repository_name: 'hcp-editor'` returns only that store's insights.
7. `ledger_search_insights` with `repository_name: 'hcp-editor'` searches only that repository store.
8. `ledger_update_insight` with `scope: 'repository'` and `repository_name` updates only the matching store.
9. `POST /api/knowledge/:id/promote` with `scope=repository&repository_name=hcp-editor` promotes the insight to global.
10. `POST /api/knowledge/:id/move` moves global→repository and repository→repository.
11. `DELETE /api/knowledge/:id` with `scope=repository&repository_name=hcp-editor` deletes from the correct store.
12. The GUI Knowledge page shows "Global" and "Repository" tabs (no "Project" tab).
13. Repository insight cards display `repository_name` and a provenance link to `origin_plan` (when present).
14. Repository insights in the GUI have working Edit, Delete, Promote to Global, and Move actions.
15. Existing `global-insights.json` is unaffected.
16. `scope: 'project'` is no longer accepted by any MCP tool or REST endpoint.
17. `scope: 'project'` is rejected at the schema level by all four MCP tools and by all REST handlers with a VALIDATION_ERROR response.

---

## Testing Strategy

All tests use real temp directories and `KnowledgeStoreManager` for fixture setup — no mocks of the storage layer. This follows the pattern in `tests/gui/knowledge-api.test.ts`. New tests are in a dedicated file. Existing `knowledge-api.test.ts` will need updating (scope enum changed) — treated as part of the implementation, not tested separately.

---

## Test Plan

All new tests live in `mcp-server/tests/gui/knowledge-repository-scope.test.ts`.

### Storage layer (`KnowledgeStoreManager`)

- `repositoryStorePath('hcp-editor')` → `{knowledgeDir}/hcp-editor-insights.json` — AC-1
- `repositoryStorePath('global')` throws (reserved name guard) — AC-3
- `addInsight({ scope: 'repository', repository_name: 'hcp-editor', origin_plan: 'p1', … })` creates file with correct content — AC-1
- `addInsight({ scope: 'repository' })` (no `repository_name`) throws — AC-2
- `readRepositoryStore('hcp-editor')` returns empty store when file absent
- `readRepositoryStore('hcp-editor')` returns stored insights after add
- `listInsights({})` returns global + repository insights — AC-4
- `listInsights({ scope: 'repository' })` returns only repository insights — AC-5
- `listInsights({ scope: 'repository', repository_name: 'hcp-editor' })` narrows to one store — AC-6
- `searchInsights('query', { repository_name: 'hcp-editor' })` searches only that store — AC-7
- `updateInsight(id, updates, { scope: 'repository', repository_name: 'hcp-editor' })` updates correctly — AC-8
- `deleteInsight(id, { scope: 'repository', repository_name: 'hcp-editor' })` removes insight
- `moveInsight` global → repository succeeds — AC-10
- `moveInsight` repository → repository (different names) succeeds — AC-10
- `moveInsight` repository → repository (same name) throws (identity check)
- `origin_plan` metadata is preserved through add, update, and move operations — AC-1, AC-13

### GUI REST handlers (`gui/api-knowledge.ts`)

- `handleListKnowledge(root, { repository_name: 'hcp-editor' })` returns only that store's insights
- `handleUpdateKnowledge` with `scope: 'repository', repository_name` updates correctly
- `handleDeleteKnowledge` with `scope: 'repository', repository_name` deletes — AC-11
- `handleDeleteKnowledge` with `scope: 'repository'` and no `repository_name` → VALIDATION_ERROR
- `handlePromoteKnowledge` with `scope: 'repository', repository_name` promotes to global — AC-9
- `handlePromoteKnowledge` with `scope: 'global'` still → VALIDATION_ERROR
- `handleMoveKnowledge` global → repository succeeds — AC-10
- `handleMoveKnowledge` repository → repository (same name) → VALIDATION_ERROR
- `handleMoveKnowledge` missing `target_repository_name` → VALIDATION_ERROR
- Passing `scope: 'project'` to any handler → VALIDATION_ERROR — AC-17

### Existing test updates

The following existing test files use `scope: 'project'` or `project_slug` and must be updated in the same changeset. All scope references change to `'repository'` with appropriate `repository_name`; all `project_slug` field references become `repository_name` (routing use) or `origin_plan` (provenance use).

- **`tests/gui/api-knowledge.test.ts`** (86 matches) — update scope enum, `project_slug` filter, Promote source scope, Move body schema.
- **`tests/gui/knowledge-api.test.ts`** (57 matches) — update all `scope: 'project'` and `project_slug` references.
- **`tests/storage/knowledge-store.test.ts`** (63 matches) — update scope enum, all `project_slug` store-routing calls.
- **`tests/tools/knowledge.test.ts`** — update scope enum and `project_slug` field references.
- **`tests/schema/knowledge.test.ts`** — update scope enum and field references.
- **`tests/gui/server-knowledge-routes.test.ts`** — update query-string key assertions from `project_slug` to `repository_name`.
- **`tests/gui/knowledge-repository-scope.test.ts`** (new file) — see Test Plan section above.

---

## Documentation Updates

Per Manifest Maintenance Rules in `AGENTS.md`:

- `mcp-server/docs/agents/project-manifest/api-surface.md` — update Knowledge Tools section: `InsightScope` now `['global', 'repository']`; `repository_name` replaces `project_slug` as store discriminator; `origin_plan` added as provenance metadata; `KnowledgeStoreManager` section updated; GUI API section updated for new schemas and handlers.
- `mcp-server/docs/agents/project-manifest/file-tree.md` — update `.knowledge/` listing: remove `{slug}-insights.json` project store line; add `{repo-name}-insights.json` repository store line.
- `mcp-server/docs/agents/project-manifest/constraints.md` — add constraint: `'global'` is a reserved repository name; `origin_plan` is provenance only, not a storage key.
- `mcp-server/changelog.md` — new MINOR version entry.
- `personas/shared/partials/synthesis-knowledge-collection.md` — two-scope table (global + repository); document `origin_plan` provenance usage.
- Root `AGENTS.md` — update the "Knowledge Collection (Synthesis persona)" cross-system dependency row to reflect the removal of project scope.

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Reserved name `'global'` not immediately obvious** | Explicit error message: `"'global' is a reserved name and cannot be used as a repository name."` Documented in constraints and help content. |
| **Existing tests break** | `tests/gui/knowledge-api.test.ts` is updated in the same changeset (Step 10 in Detailed Steps). All test changes are enumerated in the Test Plan. |
| **`moveInsight()` signature change** | Rename `targetProjectSlug` → `targetRepositoryName` and `sourceFilter.project_slug` → `sourceFilter.repository_name`. Both call sites in `gui/api-knowledge.ts` are updated in Step 5 — no transitional `undefined` state. |
| **`project_slug` semantic ambiguity** | Resolved: field renamed to `origin_plan` — unambiguous name for "which plan produced this insight." Schema JSDoc updated. Help content updated. |

