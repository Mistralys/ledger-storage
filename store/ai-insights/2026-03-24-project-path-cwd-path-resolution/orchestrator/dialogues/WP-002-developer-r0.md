# Dialogue — developer / WP-002

| Field | Value |
| ----- | ----- |
| Stage | `developer` |
| WP ID | `WP-002` |
| Captured | 2026-03-24T21:09:47.584540+00:00 |

## Human

**Project:** `/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-project-path-cwd-path-resolution`
**Work package:** WP-002

Always use the project path above for all ledger tool calls.

**Step 1 — BEFORE writing any code:** Call `ledger_begin_work` with work_package_id=WP-002, type="implementation", agent_role="Developer".

**Pipeline to start:** `implementation`

**SCOPE RESTRICTION — You must ONLY operate on work package WP-002. Do NOT call any MCP tool with a different work_package_id.**


## Assistant

```json
{
  "id": "toolu_012gPwgzwyg1PeiE1aNNtGzG",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-002",
    "type": "implementation",
    "agent_role": "Developer",
    "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-project-path-cwd-path-resolution"
  },
  "name": "ledger_begin_work",
  "type": "tool_use"
}
```

**Tool call:** `ledger_begin_work` (id: `toolu_012gPwgzwyg1PeiE1aNNtGzG`)

```json
{
  "work_package_id": "WP-002",
  "type": "implementation",
  "agent_role": "Developer",
  "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-project-path-cwd-path-resolution"
}
```

## Tool Result

{
  "work_package_id": "WP-002",
  "work_package_file": "work/WP-003.md",
  "status": "IN_PROGRESS",
  "assigned_to": "Developer",
  "dependencies": [],
  "acceptance_criteria": [
    {
      "criterion": "help-content.ts no longer contains any text describing project_path/cwd_path as mutually exclusive",
      "met": false
    },
    {
      "criterion": "help-content.ts documents that project_path takes precedence when both parameters are provided",
      "met": false
    },
    {
      "criterion": "constraints.md no longer states that supplying both parameters is an error",
      "met": false
    },
    {
      "criterion": "constraints.md describes the project_path-wins precedence rule clearly",
      "met": false
    }
  ],
  "active_pipeline_stages": [
    "implementation",
    "documentation"
  ],
  "revision": 0,
  "pipelines": [
    {
      "type": "implementation",
      "status": "IN_PROGRESS",
      "started_at": "2026-03-24T21:09:50Z",
      "summary": []
    }
  ],
  "status_changed_at": "2026-03-24T21:09:50Z",
  "last_updated": "2026-03-24T21:09:50Z",
  "claimed": true
}

## Assistant

```json
{
  "id": "toolu_01QUugSLrPAWLmKHJFftSZWT",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-002",
    "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-project-path-cwd-path-resolution"
  },
  "name": "ledger_get_work_package",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_work_package` (id: `toolu_01QUugSLrPAWLmKHJFftSZWT`)

```json
{
  "work_package_id": "WP-002",
  "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-project-path-cwd-path-resolution"
}
```

## Tool Result

{
  "work_package_id": "WP-002",
  "work_package_file": "work/WP-003.md",
  "status": "IN_PROGRESS",
  "assigned_to": "Developer",
  "dependencies": [],
  "acceptance_criteria": [
    {
      "criterion": "help-content.ts no longer contains any text describing project_path/cwd_path as mutually exclusive",
      "met": false
    },
    {
      "criterion": "help-content.ts documents that project_path takes precedence when both parameters are provided",
      "met": false
    },
    {
      "criterion": "constraints.md no longer states that supplying both parameters is an error",
      "met": false
    },
    {
      "criterion": "constraints.md describes the project_path-wins precedence rule clearly",
      "met": false
    }
  ],
  "active_pipeline_stages": [
    "implementation",
    "documentation"
  ],
  "revision": 0,
  "pipelines": [
    {
      "type": "implementation",
      "status": "IN_PROGRESS",
      "started_at": "2026-03-24T21:09:50Z",
      "summary": []
    }
  ],
  "status_changed_at": "2026-03-24T21:09:50Z",
  "last_updated": "2026-03-24T21:09:50Z"
}

## Assistant

```json
{
  "id": "toolu_015ALNVJEa8uiumE1Ww2Tru7",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-project-path-cwd-path-resolution/work/WP-003.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_015ALNVJEa8uiumE1Ww2Tru7`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-project-path-cwd-path-resolution/work/WP-003.md"
}
```

## Tool Result

     1	# WP-002: Documentation — update help content and constraints docs
     2	
     3	> **Ledger ID:** WP-002 (spec stored in work/WP-003.md)
     4	
     5	## Description
     6	Update the two human-facing documentation artefacts that previously described mutual-exclusivity enforcement to now document the precedence rule. `help-content.ts` drives the in-tool help output visible to agents at runtime; `constraints.md` is the project manifest documentation read by developers and orchestrators.
     7	
     8	## Scope
     9	- `mcp-server/src/tools/help-content.ts` — replace mutual-exclusivity paragraph with precedence-rule description
    10	- `mcp-server/docs/agents/project-manifest/constraints.md` — update section on path parameter mutual exclusivity
    11	
    12	**New help text paragraph:**
    13	`**Most tools accept \`project_path\` and/or \`cwd_path\`.** If you have \`project_path\` (the plan folder), use it — it's the fastest path. If you only know your workspace directory, pass \`cwd_path\` and the server auto-detects the active project. If you pass both, \`project_path\` takes precedence and \`cwd_path\` is ignored.`
    14	
    15	## Dependencies
    16	None
    17	
    18	## Acceptance Criteria
    19	1. `help-content.ts` no longer contains any text describing `project_path`/`cwd_path` as mutually exclusive
    20	2. `help-content.ts` documents that `project_path` takes precedence when both parameters are provided
    21	3. `constraints.md` no longer states that supplying both parameters is an error
    22	4. `constraints.md` describes the `project_path`-wins precedence rule clearly
    23	
    24	## Active Pipeline Stages
    25	implementation → documentation

## Assistant

Now let me find and read the two files I need to modify:

```json
{
  "id": "toolu_0174p5DhhSqT8v5RpQd5uv1j",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "**/help-content.ts",
    "path": "/Users/smordziol/Webserver/tools/ai-insights"
  },
  "name": "glob",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_014N6HrsfkGHtPTS9LFU6KgU",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "**/constraints.md",
    "path": "/Users/smordziol/Webserver/tools/ai-insights"
  },
  "name": "glob",
  "type": "tool_use"
}
```

**Tool call:** `glob` (id: `toolu_0174p5DhhSqT8v5RpQd5uv1j`)

```json
{
  "pattern": "**/help-content.ts",
  "path": "/Users/smordziol/Webserver/tools/ai-insights"
}
```

**Tool call:** `glob` (id: `toolu_014N6HrsfkGHtPTS9LFU6KgU`)

```json
{
  "pattern": "**/constraints.md",
  "path": "/Users/smordziol/Webserver/tools/ai-insights"
}
```

## Tool Result

['/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts']

## Tool Result

['/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md', '/Users/smordziol/Webserver/tools/ai-insights/orchestrator/docs/agents/project-manifest/constraints.md', '/Users/smordziol/Webserver/tools/ai-insights/personas/docs/agents/project-manifest/constraints.md']

## Assistant

```json
{
  "id": "toolu_01HjrZJC2rWDLmJT4ttbueZW",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_016efTqiJukVstBt1bM5AuK3",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01HjrZJC2rWDLmJT4ttbueZW`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts"
}
```

**Tool call:** `read_file` (id: `toolu_016efTqiJukVstBt1bM5AuK3`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md"
}
```

## Tool Result

     1	/**
     2	 * Static documentation strings for all Project Ledger MCP tools.
     3	 * Exported as TOOL_HELP and consumed by help.ts.
     4	 */
     5	import { PLAN_ARCHIVE_FILENAME, SYNTHESIS_ARCHIVE_FILENAME } from '../utils/constants.js';
     6	
     7	export const TOOL_HELP: Record<string, string> = {
     8	  overview: `
     9	# Project Ledger MCP — Tool Reference
    10	
    11	## Path Parameters
    12	
    13	**Most tools accept either \`cwd_path\` or \`project_path\` — not both.** Use \`cwd_path\` (your workspace root) as the preferred option; the server auto-detects the active project. Only provide \`project_path\` if you already have it from a previous tool response. The one exception is \`ledger_initialize_project\`, which requires \`project_path\` (the plan folder is being created and cannot be detected yet).
    14	
    15	## All Available Tools
    16	
    17	| Tool | Required Params | Purpose |
    18	|------|----------------|---------|
    19	| ledger_get_project_status | cwd_path or project_path | Read project overview |
    20	| ledger_initialize_project | project_path, plan_file | Create new project ledger |
    21	| ledger_list_projects | None (status filter optional) | List all tracked projects with status, dates, and plan paths |
    22	| ledger_complete_synthesis | cwd_path or project_path | Mark synthesis as generated; transitions project to COMPLETE |
    23	| ledger_get_work_package | cwd_path or project_path, work_package_id | Read a work package's full detail |
    24	| ledger_list_work_packages | cwd_path or project_path | List work packages (optional: status, assigned_to filters) |
    25	| ledger_create_work_package | cwd_path or project_path, assigned_to, dependencies, acceptance_criteria, work_package_file | Create a new work package |
    26	| ledger_claim_work_package | cwd_path or project_path, work_package_id, agent | Claim a READY WP → IN_PROGRESS |
    27	| ledger_begin_work | cwd_path or project_path, work_package_id, type, agent_role | Claim + start pipeline in one atomic call |
    28	| ledger_update_work_package_status | cwd_path or project_path, work_package_id, status, agent | Update WP status |
    29	| ledger_start_pipeline | cwd_path or project_path, work_package_id, type | Start a pipeline for a work package (ordering determined by WP's active_pipeline_stages) |
    30	| ledger_complete_pipeline | cwd_path or project_path, work_package_id, type, status, summary | Complete a pipeline |
    31	| ledger_cancel_pipeline | cwd_path or project_path, work_package_id, type, reason | Cancel a stale IN_PROGRESS pipeline (sets to FAIL) |
    32	| ledger_update_pipeline_progress | cwd_path or project_path, work_package_id, type, summary | Update summary of IN_PROGRESS pipeline without completing it |
    33	| ledger_add_observation | cwd_path or project_path, work_package_id, pipeline_type, type, priority, note | Add observation to pipeline |
    34	| ledger_add_project_comment | cwd_path or project_path, type, priority, agent, note | Add project-level comment |
    35	| ledger_get_next_action | cwd_path or project_path, agent_role | Get next recommended action (optional: max_results for batch mode) |
    36	| ledger_get_handoff_status | cwd_path or project_path, current_agent | Check handoff status |
    37	
    38	## Common Mistakes
    39	
    40	1. **Forgetting the "agent" parameter** — ledger_claim_work_package, ledger_update_work_package_status, and ledger_add_project_comment ALL require an "agent" param with your agent name.
    41	2. **Wrong pipeline type names** — Use exactly: "implementation", "qa", "security-audit", "code-review", "release-engineering", "documentation". Only the pipeline types listed in a WP's \`active_pipeline_stages\` are valid for that WP.
    42	3. **Trying to mark COMPLETE as the wrong terminal agent** — Only the agent owning the last active stage of the WP's pipeline can auto-finalize to COMPLETE. For the default 4-stage pipeline this is the Documentation agent. For non-standard compositions (e.g., verification-only \`["qa", "code-review"]\`), it is the agent owning the last active stage (e.g., Reviewer).
    43	4. **Starting a pipeline before claiming the WP** — WP must be IN_PROGRESS before starting a pipeline.
    44	5. **Not updating acceptance_criteria** — Use the acceptance_criteria_updates param in ledger_complete_pipeline to mark criteria as met before marking WP COMPLETE.
    45	6. **Starting pipelines out of order** — Pipelines must follow the WP's active stage order (a subsequence of: implementation → qa → security-audit → code-review → release-engineering → documentation). Starting a stage requires a PASS pipeline on the immediately preceding active stage. Starting a stage not in the WP's \`active_pipeline_stages\` is also rejected.
    46	7. **Setting WP to BLOCKED after a pipeline FAIL** — When QA or Reviewer fails a pipeline, do NOT set the WP to BLOCKED. Leave it as IN_PROGRESS so the Developer can find it via ledger_get_next_action and rework. BLOCKED should only be used for external blockers (missing APIs, pending decisions, etc.).
    47	8. **Test-only WP references non-existent production method** — When creating a WP whose \`active_pipeline_stages\` excludes "implementation" (test-only, verification-only, or documentation-only), verify that all methods/functions referenced in the WP's scope already exist in production code. If they don't, the WP needs the "implementation" stage — otherwise the Developer will silently expand scope by adding production code inside a non-implementation WP.
    48	
    49	## Workflow Order
    50	
    51	1. PM creates work packages (ledger_create_work_package), optionally specifying a custom \`active_pipeline_stages\` to compose the pipeline (defaults to \`["implementation","qa","code-review","documentation"]\`)
    52	2. Developer claims WP and starts pipeline in one call (\`ledger_begin_work\` type="implementation", agent_role="Developer"), completes pipeline (ledger_complete_pipeline). Note: starting a pipeline auto-updates assigned_to on the WP.
    53	3. QA starts pipeline (type="qa"), completes pipeline
    54	3a. *(Optional — only if WP's active_pipeline_stages includes "security-audit")* Security Auditor starts pipeline (type="security-audit"), completes pipeline
    55	4. Reviewer starts pipeline (type="code-review"), completes pipeline
    56	4a. *(Optional — only if WP's active_pipeline_stages includes "release-engineering")* Release Engineer starts pipeline (type="release-engineering"), completes pipeline
    57	5. Documentation starts pipeline (type="documentation"), completes pipeline — if status=PASS and all acceptance criteria are met, the WP is automatically transitioned to COMPLETE (auto-finalize, no separate ledger_update_work_package_status call needed)
    58	
    59	**Note:** The terminal agent (owner of the last active stage) triggers auto-finalize on PASS. For non-standard compositions (e.g., \`["qa","code-review"]\`), the Reviewer is the terminal agent who auto-finalizes the WP to COMPLETE.
    60	
    61	**Important:** Every ledger_complete_pipeline response includes a "--- NEXT STEP ---" guidance block telling you exactly what to do next. Follow it.
    62	
    63	## Rework After Pipeline FAIL
    64	
    65	When a QA or code-review pipeline completes with FAIL:
    66	- The agent who ran the failing pipeline should leave the WP as IN_PROGRESS (do NOT set to BLOCKED)
    67	- Call ledger_get_handoff_status to confirm handoff
    68	- The Developer will automatically see a REWORK action via ledger_get_next_action
    69	- The Developer re-implements, then the pipeline chain continues from QA again
    70	
    71	## Handoff Block Format
    72	
    73	Every agent must end their response with the handoff block returned by ledger_get_handoff_status. The block uses three fields:
    74	
    75	\`\`\`
    76	CURRENT AGENT: <current_agent>
    77	NEXT AGENT: <next_agent>
    78	STATUS: <status>
    79	\`\`\`
    80	
    81	- **current_agent**: The agent that just finished working (you)
    82	- **next_agent**: The agent that should pick up work next (derived from status)
    83	- **status**: The workflow status (e.g., READY_FOR_QA, IN_PROGRESS, COMPLETE)
    84	
    85	All three fields are returned by ledger_get_handoff_status — copy them verbatim.
    86	
    87	## Action Tool max_results
    88	
    89	- **ledger_get_next_action** (singular, default): Returns the first actionable WP for your role. Best for simple projects or when you process one WP at a time.
    90	- **ledger_get_next_action with max_results > 1**: Returns up to N actionable WPs as an array under the "actions" key. Best for projects with many independent WPs.
    91	
    92	## Storage Architecture
    93	
    94	All ledger files are stored **centrally** at \`{mcp-server}/storage/ledger/{slug}/\` — not inside the plan folder.
    95	- Plan folders remain purely human-readable markdown (no \`.ledger/\` subdirectory).
    96	- The storage root can be overridden at server startup with \`--ledger-dir <path>\`.
    97	
    98	### Files per project
    99	- **\`project-ledger.json\`** — root index (WP summaries, counters, status)
   100	- **\`WP-###.json\`** — individual work package detail files

## Tool Result

     1	# Constraints & Conventions
     2	
     3	This document codifies established rules, conventions, and non-obvious gotchas.
     4	
     5	### Constraint Entry Format
     6	
     7	New constraint entries should follow this structure (modelled on Constraint 2):
     8	
     9	| Section | Content |
    10	|---------|---------|
    11	| **Rule** | The specific, actionable rule — include forbidden alternatives inline. |
    12	| **Rationale** | Why the rule exists. One or two sentences. |
    13	| **Anti-pattern** (if applicable) | A concrete ❌ code example showing the wrong approach. |
    14	| **Correct pattern** (if applicable) | A concrete ✅ code example showing the right approach. |
    15	| **Forbidden patterns** (if applicable) | A prose or list summary of every variant that must NOT be used. |
    16	
    17	---
    18	
    19	## Workflow Specification Governance
    20	
    21	### 0. The Workflow Specification Is the Source of Truth for All Workflow Logic
    22	
    23	**Rule:** The [Workflow Specification](../workflow-specification/README.md) is the authoritative definition of all workflow logic — state machines, pipeline routing, status transitions, handoff behavior, recommendation engine behavior, edge cases, and constants. Implementation code must conform to the specification. When code contradicts the specification, the code is wrong.
    24	
    25	**Spec-first development:** Changes to workflow logic MUST be made in the specification first, then implemented in code, then validated by tests, then documented in the project manifest — in that order.
    26	
    27	**Test traceability:** Test descriptions SHOULD reference the workflow specification section they validate (e.g., `// §14.13 row 1: returns true when QA FAIL started after impl PASS completed`). This convention is already practiced in several test files and should be followed consistently.
    28	
    29	**Rationale:** The specification was designed to be a language-agnostic, formally reviewed reference. Treating code as the source of truth defeats this purpose and leads to silent behavioral drift between the TypeScript (MCP server) and Python (orchestrator) implementations.
    30	
    31	**Scope:** This constraint applies to workflow logic only — file I/O, schema validation, concurrency primitives, and other infrastructure concerns are governed by their respective constraints below and the project manifest.
    32	
    33	---
    34	
    35	## File System Constraints
    36	
    37	### 1. All File I/O Must Be Atomic
    38	
    39	**Rule:** Never write directly to target files. Always use the `atomicWriteJson()` function.
    40	
    41	**Rationale:** Ensures readers never see partial writes or corrupt JSON.
    42	
    43	**Implementation:** Write to `{file}.tmp.{pid}`, then atomically rename to target.
    44	
    45	**Anti-pattern:**
    46	```typescript
    47	// ❌ WRONG — direct write; a crash mid-write leaves the target file truncated or corrupt
    48	await fs.writeFile(targetPath, JSON.stringify(data, null, 2) + '\n', 'utf-8');
    49	```
    50	
    51	**Correct pattern:**
    52	```typescript
    53	// ✅ CORRECT — write to .tmp.{pid}, then rename; readers never see a partial file
    54	await atomicWriteJson(targetPath, data);
    55	```
    56	
    57	---
    58	
    59	### 2. Dual-File Updates Require Locking
    60	
    61	**Rule:** When writing both `storage/ledger/{slug}/project-ledger.json` and `storage/ledger/{slug}/WP-###.json`, always use the appropriate high-level method: `LedgerStore.createWorkPackageWithSync()` for creating a new WP, `LedgerStore.updateWorkPackageWithSync()` for updating a single existing WP, or `LedgerStore.batchUpdateWorkPackagesWithSync()` for updating multiple WPs in one operation (see Constraint 2b). Only fall back to a manual `withLock(store.storageDir, ...)` scope when none of these methods covers the use case. **`store.storageDir` is the only acceptable first argument to `withLock` — never pass `projectPath`, `ledgerRoot`, or `ledgerRoot ?? projectPath`.** Once a `LedgerStore` is constructed, use its `.storageDir` property to obtain the canonical lock directory.
    62	
    63	**Extension — Single-File Read-Modify-Write:** Even when updating only the root index, any read-modify-write sequence must also be wrapped in `withLock(store.storageDir, ...)` to prevent TOCTOU races. Example: `completeSynthesis` reads the root index, mutates `synthesis_generated` and project status, then writes it back — this entire sequence must occur inside a single lock scope.
    64	
    65	**Rationale:** Prevents race conditions and dual-file desync when multiple agents run concurrently.
    66	
    67	**Anti-pattern:**
    68	```typescript
    69	// ❌ WRONG — race condition risk
    70	await store.writeWorkPackage(wpId, updatedWp);
    71	await store.writeRootIndex(updatedRoot);
    72	```
    73	
    74	**Correct pattern:**
    75	```typescript
    76	// ✅ CORRECT — atomic dual-file creation (new WP)
    77	await store.createWorkPackageWithSync(async (root) => {
    78	  // ... build new WP detail and updated root ...
    79	  return { wpId, wp: newWpDetail, root: updatedRoot };
    80	});
    81	
    82	// ✅ CORRECT — atomic dual-file update (existing WP)
    83	await store.updateWorkPackageWithSync(wpId, (wp, root) => {
    84	  // ... update both wp and root ...
    85	  return { wp: updatedWp, root: updatedRoot };
    86	});
    87	```
    88	
    89	---
    90	
    91	### 2b. Batch Multi-WP Writes Must Use `batchUpdateWorkPackagesWithSync`
    92	
    93	**Rule:** When updating multiple work packages and the root index in a single operation, always use `LedgerStore.batchUpdateWorkPackagesWithSync()`. Never loop over `updateWorkPackageWithSync()` calls or acquire multiple separate `withLock` scopes to write a batch of WPs — this produces one lock acquisition per WP instead of one per operation.
    94	
    95	**Rationale:** A loop of per-WP lock acquisitions is not atomic at the operation level: a crash or concurrent write between iterations can leave some WPs updated while others are not, desynchronizing WP state and the root index. `batchUpdateWorkPackagesWithSync` consolidates all reads, validation, writes, and the root index sync into a single lock scope.
    96	
    97	**Atomicity invariant (two-pass validate-then-write):** The method validates all WPs via Zod **before** writing any of them. A validation failure on any WP in the batch aborts the entire operation with no disk writes. This is stronger than the per-WP atomicity provided by `updateWorkPackageWithSync`, which validates and writes one WP at a time.
    98	
    99	**Note on lock-scope vs. rollback-scope atomicity:** If a file write succeeds for WP-A but a subsequent I/O error prevents writing WP-B, WP-A's write is not rolled back. This characteristic is shared with `updateWorkPackageWithSync`. Validation failures are fully atomic (no writes); I/O failures after the write phase begin are not.
   100	

## Assistant

```json
{
  "id": "toolu_01NRxgt6o5si6T9kDk9sHHfV",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
    "offset": 100,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01NRxgt6o5si6T9kDk9sHHfV`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
  "offset": 100,
  "limit": 200
}
```

## Tool Result

   101	**Anti-pattern:**
   102	```typescript
   103	// ❌ WRONG — multiple lock acquisitions; not atomic across the batch
   104	for (const wpId of candidateIds) {
   105	  await store.updateWorkPackageWithSync(wpId, (wp, root) => {
   106	    // ...
   107	    return { wp: updatedWp, root: updatedRoot };
   108	  });
   109	}
   110	```
   111	
   112	**Correct pattern:**
   113	```typescript
   114	// ✅ CORRECT — single lock; all WPs validated before any write
   115	await store.batchUpdateWorkPackagesWithSync(async (root, readWp) => {
   116	  const updatedWps = new Map<string, WorkPackageDetail>();
   117	  for (const wpId of candidateIds) {
   118	    const wp = await readWp(wpId);
   119	    // ... mutate wp ...
   120	    updatedWps.set(wpId, wp);
   121	  }
   122	  // ... mutate root ...
   123	  return { updatedWps, root: updatedRoot };
   124	});
   125	```
   126	
   127	**Known callers:** `propagateDependencyUnblock` and `propagateDependencyReblock` in `src/tools/work-package.ts`; `applyProjectReset` and `markProjectComplete` in `src/utils/project-reset.ts`.
   128	
   129	---
   130	
   131	### 2c. `writeWorkPackage` and `writeRootIndex` Are `@internal` — Tool Code Must Not Call Them Directly
   132	
   133	**Rule:** `LedgerStore.writeWorkPackage()` and `LedgerStore.writeRootIndex()` are marked `@internal` in source. Tool functions (`src/tools/`) and shared helpers (`src/utils/`) must never call these methods directly. All WP+root writes must go through one of the three sync methods (Constraints 2 and 2b).
   134	
   135	**Rationale:** Bypassing the sync methods skips `last_updated` auto-stamping, Zod validation, `.meta.json` sync, and the single-lock atomicity guarantee. The `@internal` tag is documentation-only (TypeScript does not enforce it) — this constraint encodes the boundary as a project rule.
   136	
   137	**Legitimate direct callers of `writeRootIndex` (non-tool code):**
   138	- `src/tools/project-lifecycle.ts` — `getProjectStatus()` self-healing: repairs stale counter fields under an explicit `withLock` scope; `initializeProject()` and `completeSynthesis()` for root-index-only transitions that don't involve any WP file write
   139	- `auto-archive.ts` — sets `status: 'ARCHIVED'` with `preserveLastUpdated: true` (root-index write only; sync methods do not apply)
   140	- `observations.ts` — appends a project-level comment (root-index write only; no WP file involved)
   141	- `workflow-handoff.ts` — `buildHandoffResponse()`: increments or caps the `auto_handoff_depth` counter on every handoff-status response; root-index-only write with no WP file involvement
   142	
   143	**`writeWorkPackage` — zero external callers (post-WP-002):** As of the WP-002 migration (consolidate-wp-writes), `writeWorkPackage` has no legitimate external callers. Every previously-direct caller (e.g., `project-reset.ts`) has been migrated to a sync method. The `@internal` boundary for `writeWorkPackage` is now absolute.
   144	
   145	**Anti-pattern:**
   146	```typescript
   147	// ❌ WRONG — bypasses auto-stamping, validation, and .meta.json sync
   148	await store.writeWorkPackage(wpId, updatedWp);
   149	await store.writeRootIndex(updatedRoot);
   150	```
   151	
   152	**Correct pattern:** Use `updateWorkPackageWithSync`, `createWorkPackageWithSync`, or `batchUpdateWorkPackagesWithSync` as shown in Constraints 2 and 2b.
   153	
   154	---
   155	
   156	### 3. Paths Must Be Absolute
   157	
   158	**Rule:** All MCP tool inputs require absolute paths for `project_path`.
   159	
   160	**Rationale:** The server has no concept of "current working directory" — it must be told explicitly where files live.
   161	
   162	---
   163	
   164	### 4. Plan Folders Must Remain Human-Readable Markdown Only
   165	
   166	**Rule:** No machine-generated files (JSON, lock files, etc.) may be written inside plan folders.
   167	
   168	**Rationale:** Plan folders are the authoritative human source-of-truth. Machine output lives in the centralized ledger at `{mcp-server}/storage/ledger/{slug}/`.
   169	
   170	**Archiving clarification:** `archiveDocuments()` copies files **from** the plan folder **into** the centralized storage directory (`storage/ledger/{slug}/`). The direction is one-way: plan folder → ledger. The archived copy is read-only from the agent's perspective — it exists for retrieval by the GUI and tooling, not for editing. The original file in the plan folder remains the authoritative source and is never modified by the server. This is fully consistent with Constraint 4: no writes ever occur inside the plan folder.
   171	
   172	**`plan_file` validation:** the `plan_file` argument accepted by `ledger_initialize_project` is enforced at parse time by a Zod `.refine()` check: `v === PLAN_ARCHIVE_FILENAME`. Calls with any value other than `'plan.md'` are rejected with a Zod validation error before reaching handler logic. This ensures the GUI's `/api/projects/:slug/plan` endpoint can always rely on the archived plan document having the fixed filename `plan.md`.
   173	
   174	**Archive error contract:** `archiveDocuments()` uses a discriminated error strategy:
   175	- Missing source file (`ENOENT`) — the filename is silently added to `skipped[]` and a warning is written to `stderr`. The operation continues with remaining files.
   176	- All other I/O errors (e.g., `EACCES`, `ENOSPC`, `EISDIR`) — the error is **re-thrown** to the caller. Callers must not assume all errors from `archiveDocuments()` are benign; they must be prepared to handle re-thrown non-ENOENT errors.
   177	
   178	---
   179	
   180	### 5. `.meta.json` Must Be Written Under the Project Lock
   181	
   182	**Rule:** `writeProjectMeta()` must always be called inside the same `withLock()` scope as the root index write it synchronizes. Never call it outside a lock context except for the standalone `writeRootIndex()` (which manages its own internal sync). Note: `writeRootIndex` is `@internal` — see Constraint 2c for the list of legitimate direct callers.
   183	
   184	**Rationale:** Prevents `.meta.json` from lagging behind the root index in a concurrent environment.
   185	
   186	---
   187	
   188	### 6. Central Ledger Root Is Resolved Once at Startup
   189	
   190	**Rule:** `resolveLedgerRoot()` is called once at server startup. The `--ledger-dir <path>` CLI argument overrides the default `{mcp-server}/storage/ledger/` location. The resolved path is logged to stderr.
   191	
   192	**Usage:**
   193	```bash
   194	# Override ledger root:
   195	node dist/index.js --ledger-dir /custom/path/to/ledger
   196	```
   197	
   198	**Default:** `{mcp-server}/storage/ledger/` (relative to the server package root).
   199	
   200	---
   201	
   202	### 7. STDIO Logging Discipline
   203	
   204	**Rule:** Never log to `stdout`. All logs must go to `stderr`.
   205	
   206	**Rationale:** `stdout` is reserved for the MCP protocol. Logging to `stdout` breaks protocol communication.
   207	
   208	**Implementation:**
   209	```typescript
   210	// ✅ CORRECT
   211	console.error('[project-ledger-mcp] Server started');
   212	
   213	// ❌ WRONG — breaks MCP protocol
   214	console.log('[project-ledger-mcp] Server started');
   215	```
   216	
   217	---
   218	
   219	## Schema Constraints
   220	
   221	### 8. Work Package IDs Must Follow WP-### Format
   222	
   223	**Rule:** All work package IDs must match the regex `/^WP-\d{3,}$/` (e.g., `WP-001`, `WP-042`, `WP-999`, `WP-1000`). The minimum is three digits; there is no upper bound to future-proof projects beyond WP-999.
   224	
   225	**Enforcement:** Validated by Zod schemas in `GetWorkPackageSchema`, `CreateWorkPackageSchema` (dependencies array), `ClaimWorkPackageSchema`, `StartPipelineSchema`, `CompletePipelineSchema`, `CancelPipelineSchema`, `UpdatePipelineProgressSchema`, and `AddObservationSchema`, as well as utility functions (`formatWpId()`, `parseWpId()`).
   226	
   227	---
   228	
   229	### 9. Timestamps Must Use UTC ISO 8601 Format (YYYY-MM-DDTHH:MM:SSZ)
   230	
   231	**Rule:** All timestamp fields use UTC ISO 8601 format with a trailing `Z`. Always use the `now()` utility function.
   232	
   233	**Anti-pattern:**
   234	```typescript
   235	// ❌ WRONG — local time, inconsistent format
   236	const timestamp = new Date().toLocaleString();
   237	```
   238	
   239	**Correct pattern:**
   240	```typescript
   241	// ✅ CORRECT — UTC with trailing Z
   242	const timestamp = now(); // "2026-02-16T18:00:00Z"
   243	```
   244	
   245	**Backward compatibility:** `parseTimestamp()` accepts legacy formats (`YYYY-MM-DD HH:MM:SS`, `YYYY-MM-DDTHH:MM:SS` without Z) for ledger files written by earlier versions.
   246	
   247	---
   248	
   249	### 10. JSON Must Be Pretty-Printed
   250	
   251	**Rule:** All JSON files written by the server must use 2-space indentation and include a trailing newline.
   252	
   253	**Rationale:** Human readability and clean git diffs.
   254	
   255	**Enforcement:** `atomicWriteJson()` automatically formats as `JSON.stringify(data, null, 2) + '\n'`.
   256	
   257	---
   258	
   259	## Business Rule Constraints
   260	
   261	### 11. Status Transitions Are Enforced
   262	
   263	**Rule:** Work package status transitions must follow the legal transition table:
   264	
   265	| From | To | Special Conditions |
   266	|------|----|--------------------|
   267	| `READY` | `IN_PROGRESS` | Dependencies must be `COMPLETE` or `CANCELLED` |
   268	| `READY` | `BLOCKED` | None |
   269	| `READY` | `CANCELLED` | PM-only agent guard |
   270	| `IN_PROGRESS` | `COMPLETE` | All acceptance criteria must be met; Documentation agent only |
   271	| `IN_PROGRESS` | `BLOCKED` | None |
   272	| `IN_PROGRESS` | `READY` | None (unclaim path, spec §21.13) |
   273	| `IN_PROGRESS` | `CANCELLED` | PM-only agent guard |
   274	| `BLOCKED` | `IN_PROGRESS` | None (implicitly means blocker resolved); clears `blocked_by` |
   275	| `BLOCKED` | `READY` | All dependencies COMPLETE (auto-unblock); clears `blocked_by` |
   276	| `BLOCKED` | `CANCELLED` | PM-only agent guard |
   277	| `COMPLETE` | `IN_PROGRESS` | Triggers revision increment; Project Manager or Documentation agent only |
   278	| `COMPLETE` | `CANCELLED` | PM-only agent guard |
   279	
   280	`CANCELLED` is the only fully **terminal status** — it has no outward transitions. This includes `CANCELLED → CANCELLED` self-transitions — re-cancelling an already-cancelled WP is rejected. `COMPLETE` allows one outward transition (to `CANCELLED`, PM-only).
   281	**Rule:** A work package cannot be marked `COMPLETE` unless all acceptance criteria have `met: true`.
   282	
   283	**Enforcement:** `canCompleteWorkPackage()` validator in `ledger_update_work_package_status` tool.
   284	
   285	**Error message format:**
   286	```
   287	Cannot mark work package as COMPLETE: the following acceptance criteria are not met:
   288	  - Criterion 1
   289	  - Criterion 2
   290	```
   291	
   292	> Full specification: [Workflow Specification §6.2](../workflow-specification/state-machines.md#62-transition-table).
   293	
   294	---
   295	
   296	### 13. Only Documentation Agent Can Set COMPLETE
   297	
   298	**Rule:** The `ledger_update_work_package_status` tool rejects transitions to `COMPLETE` from any agent other than `"Documentation"` or `"Documentation Agent"`.
   299	
   300	**Enforcement:** Hard guard in `updateWorkPackageStatus()`. The error message includes the full workflow reminder (Developer → QA → Reviewer → Documentation → COMPLETE).

## Assistant

```json
{
  "id": "toolu_018gcxZrzYt2MsSu2FH7VkoF",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
    "offset": 300,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_018gcxZrzYt2MsSu2FH7VkoF`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
  "offset": 300,
  "limit": 200
}
```

## Tool Result

   301	
   302	**Rationale:** Enforces the multi-stage workflow at the MCP server level. Previously this was a persona-level convention only; the guard was added after the 2026-02-22 workflow failure where a Developer agent set COMPLETE directly. As of WP-005, auto-finalize on terminal-stage PASS (see Constraint 13b) is the preferred COMPLETE path — `ledger_update_work_package_status` remains registered for PM and edge-case use only.
   303	
   304	> Full specification: [Workflow Specification §6.5, §21.10](../workflow-specification/state-machines.md#65-agent-guards).
   305	
   306	---
   307	
   308	### 13b. Auto-Finalize on Terminal-Stage Pipeline PASS (WP-005)
   309	
   310	**Rule:** When `ledger_complete_pipeline` is called with `status: "PASS"` and the calling agent owns the WP's **last active stage** (terminal stage), the server automatically evaluates whether all acceptance criteria are met **after** applying `acceptance_criteria_updates`. If all criteria are met, the WP is transitioned to `COMPLETE` **within the same lock scope** as the pipeline completion — no separate `ledger_update_work_package_status` call is required.
   311	
   312	The terminal stage is determined dynamically: `CANONICAL_PIPELINE_ORDERING.filter(t => activeStages.includes(t)).at(-1)`. For default WPs (`DEFAULT_PIPELINE_STAGES`), this is `documentation` (Documentation agent). For custom-stage WPs it may be any stage.
   313	
   314	**Conditions (all must apply):**
   315	- `type === lastActiveStage` (the last entry in the WP's ordered active stages)
   316	- `status === 'PASS'`
   317	- `agent_role === PIPELINE_AGENT_MAP[lastActiveStage]` (PM overrides bypass auto-finalize)
   318	- All `wp.acceptance_criteria[*].met === true` after applying `acceptance_criteria_updates`
   319	
   320	**Response signals:**
   321	- `auto_finalized: true` — WP transitioned to COMPLETE; `pending_work_packages` decremented.
   322	- `auto_finalize_blocked: true` + `unmet_criteria: string[]` — criteria check failed; WP stays IN_PROGRESS.
   323	
   324	**Enforcement:** Logic in `completePipeline()` in `src/tools/pipeline.ts` (added in WP-006).
   325	
   326	**Dependency unblocking side-effect (§6.3):** When auto-finalize transitions the WP to `COMPLETE`, `propagateDependencyUnblock` is called **after** the main lock is released (consistent with §12.2, Gotcha 8). This transitions eligible BLOCKED dependents to `READY`. Only dependents whose `blocked_by.type` is `'dependency'` (or absent) are eligible — WPs blocked by `'external'`, `'decision'`, or `'technical'` reasons remain BLOCKED.
   327	
   328	**Rationale:** The Documentation agent always called `ledger_update_work_package_status` immediately after a PASS pipeline — the transition was unconditional and never conditional. Automating it server-side removes a mandatory extra tool call from every Documentation pipeline, shortening the agent loop by one step.
   329	
   330	**`ledger_update_work_package_status` remains registered** for PM and edge-case use (e.g., re-opening a WP, manually completing a WP with prior pipeline history).
   331	
   332	---
   333	
   334	### 14. Claiming a WP Assigned to Another Agent Requires Override
   335	
   336	**Rule:** `ledger_claim_work_package` rejects the claim when the work package's `assigned_to` field differs from the calling `agent` parameter, unless `override: true` is explicitly passed.
   337	
   338	**Authorization:** Only the **Project Manager** (`"Project Manager"`) and the **current assignee** (`wp.assigned_to`) are permitted to use `override: true`. Any other agent passing `override: true` will receive a hard rejection error. The guard is conditional on `wp.assigned_to` being set — unassigned WPs bypass the identity check.
   339	
   340	**Error message (unauthorized override):**
   341	```
   342	override is restricted to "Project Manager" or the current assignee ("Developer"). You are "Reviewer".
   343	```
   344	
   345	**Enforcement:** Hard guard in `claimWorkPackage()` before dependency and status-transition checks.
   346	
   347	**Error message format:**
   348	```
   349	Cannot claim work package WP-002: it is assigned to "Documentation" but you are "Developer".
   350	
   351	If you need to re-assign this WP, pass override: true.
   352	Otherwise, only claim work packages assigned to your role.
   353	```
   354	
   355	**Rationale:** Prevents agents from silently re-assigning WPs outside their remit — the root cause of the 2026-02-22 workflow failure where the Developer agent claimed and completed a Documentation WP.
   356	
   357	---
   358	
   359	### 15. Dependencies Must Exist Before Creation
   360	
   361	**Rule:** When creating a work package, all dependency IDs must already exist in the root index.
   362	
   363	**Enforcement:** `ledger_create_work_package` validates dependencies before creating the work package.
   364	
   365	**Rationale:** Prevents dangling references.
   366	
   367	---
   368	
   369	### 16. BLOCKED Status Requires Blocker Object
   370	
   371	**Rule:** When transitioning a work package to `BLOCKED`, the `blocked_by` field must be provided.
   372	
   373	**Enforcement:** `ledger_update_work_package_status` throws an error if `status: 'BLOCKED'` is passed without `blocked_by`.
   374	
   375	---
   376	
   377	### 17. Pipelines Require IN_PROGRESS Work Package
   378	
   379	**Rule:** A pipeline can only be started on a work package with status `IN_PROGRESS`.
   380	
   381	**Enforcement:** `ledger_start_pipeline` validates WP status before creating pipeline.
   382	
   383	**Rationale:** Prevents starting work before a work package is claimed.
   384	
   385	---
   386	
   387	### 18. No Duplicate IN_PROGRESS Pipelines
   388	
   389	**Rule:** Only one pipeline of a given type can be `IN_PROGRESS` at a time for a work package.
   390	
   391	**Enforcement:** `ledger_start_pipeline` checks for existing `IN_PROGRESS` pipeline of the same type before creating a new one.
   392	
   393	**Rationale:** Forces agents to complete or fail a pipeline before retrying.
   394	
   395	---
   396	
   397	### 19. Pipelines Must Follow the Required Ordering
   398	
   399	**Rule:** Pipelines must be started in the order defined by the work package's `active_pipeline_stages` (defaults to `DEFAULT_PIPELINE_STAGES` — `['implementation', 'qa', 'code-review', 'documentation']` — when omitted). Each stage requires a PASS on its immediately preceding active stage. Attempting to start a pipeline without the **most recent** prerequisite pipeline having a `PASS` status throws a descriptive error. A historical PASS followed by a FAIL is not sufficient — the most recent entry is the only one that counts (per §8.2 most-recent-wins semantics).
   400	
   401	**Enforcement:** `ledger_start_pipeline` calls `resolvePrerequisite(type, activeStages)` — which filters `CANONICAL_PIPELINE_ORDERING` by the WP's `active_pipeline_stages` and returns the immediately preceding active stage — then finds the most recent pipeline of that prerequisite type via `.at(-1)`, and rejects if it is absent or its status is not `PASS`.
   402	
   403	**Error message format:**
   404	```
   405	Cannot start 'qa' pipeline: requires a PASS 'implementation' pipeline first.
   406	Active pipeline order: implementation → qa → code-review → documentation.
   407	```
   408	
   409	**Exception:** The first active stage in the WP's ordering has no prerequisite and can always be started (subject to other constraints). For `DEFAULT_PIPELINE_STAGES`, this is `implementation`.
   410	
   411	> Full specification: [Workflow Specification §8](../workflow-specification/pipeline-routing.md).
   412	
   413	---
   414	
   415	### 20. Pipeline Start Auto-Updates `assigned_to`
   416	
   417	**Rule:** When a pipeline starts, the work package's `assigned_to` field is automatically updated to the responsible agent according to the `PIPELINE_AGENT_MAP`:
   418	
   419	| Pipeline type | Assigned agent |
   420	|---|---|
   421	| `implementation` | `Developer` |
   422	| `qa` | `QA` |
   423	| `security-audit` | `Security Auditor` |
   424	| `code-review` | `Reviewer` |
   425	| `release-engineering` | `Release Engineer` |
   426	| `documentation` | `Documentation` |
   427	
   428	**Enforcement:** `ledger_start_pipeline` applies the map atomically alongside the pipeline creation. Both WP detail and root index summary are updated.
   429	
   430	---
   431	
   432	### 21. Rework Count Increments on Pipeline Retry
   433	
   434	**Rule:** When `ledger_start_pipeline` detects a rework, the work package's rework counters are automatically incremented. Rework is detected when either:
   435	- **Direct rework:** The most recent completed pipeline of the same type has `FAIL` status.
   436	- **Downstream rework:** A prerequisite pipeline type was reworked (re-failed) after the last PASS of the current pipeline type.
   437	
   438	Auto-cancelled pipelines (`.auto_cancelled === true`) are excluded from both rework-detection checks. This exclusion also applies to **temporal comparison functions** such as `checkRevalidationGuard` — a pipeline with `auto_cancelled: true` is invisible to all time-based guard logic. Auto-cancelled pipelines must never be counted by rework detection, circuit breakers, or any temporal comparison function.
   439	
   440	**Primary field:** `rework_counts` — a per-pipeline-type map (`{ implementation?, qa?, code-review?, documentation? }`). This is the authoritative counter going forward.
   441	
   442	**Legacy field:** `rework_count` — a scalar counter that was maintained during a prior transition period. **Fully retired as of 2026-02-28.** No production code path writes this field anymore. The in-memory migration in `LedgerStore.readWorkPackage()` (see below) handles any on-disk files that still contain it, but no new writes are emitted.
   443	
   444	**Backward-compat migration:** `LedgerStore.readWorkPackage()` performs a lazy in-memory migration: if a file contains `rework_count` but no `rework_counts`, it synthesises `rework_counts: { implementation: rework_count, qa: 0, 'code-review': 0, documentation: 0 }` and removes `rework_count`. This migration is **in-memory only** — no write is triggered; the on-disk file is updated lazily on the next `updateWorkPackageWithSync()` call.
   445	
   446	**Enforcement:** `ledger_start_pipeline` applies both rework-detection checks and excludes auto-cancelled pipelines. A history of `[FAIL, PASS]` does **not** trigger an increment because the most recent is `PASS`.
   447	
   448	**Initial value:** Both fields are absent (`undefined`) until the first rework; neither is ever initialised to `0` on creation.
   449	
   450	| Rework condition | rework_counts change |
   451	|---|---|
   452	| None (no prior failure, no downstream rework) | No increment |
   453	| Direct rework (last same-type FAIL) | rework_counts[type] +1 |
   454	| Downstream rework (prerequisite reworked after last PASS) | rework_counts[type] +1 |
   455	
   456	**Circuit breaker:** After incrementing, the effective count is computed as `rework_counts?.[type] ?? 0`. If this value reaches `MAX_REWORK_COUNT` (default: 5, from `workflow-helpers.ts`), `ledger_start_pipeline` rejects with an error guiding the caller to cancel or restructure. The `getDeveloperAction` function also surfaces `BLOCK_FOR_REWORK_LIMIT` as the highest-priority action for affected WPs.
   457	
   458	---
   459	
   460	### 22. Handoff Notes Are Routed via resolveNextAgent / resolveFailAgent
   461	
   462	**Rule:** When `ledger_complete_pipeline` is called with a `handoff_notes` array, a structured `HandoffNote` entry is appended to the work package. The `to_agent` is determined dynamically based on pipeline status and the WP's `active_pipeline_stages`:
   463	
   464	- **On PASS:** `resolveNextAgent(type, activeStages)` returns the owner of the next active stage in canonical order, or `'Synthesis'` when the type is the last active stage.
   465	- **On FAIL:** `resolveFailAgent(type, activeStages)` uses a base routing map extended to all 6 stages. If the base fail-target's stage is absent from `activeStages`, the fallback is the agent that owns the first active stage.
   466	
   467	**Routing for the default 4-stage pipeline (`DEFAULT_PIPELINE_STAGES`):**
   468	
   469	| Pipeline type | PASS → to_agent | FAIL → to_agent |
   470	|---|---|---|
   471	| `implementation` | `QA` | `Developer` |
   472	| `qa` | `Reviewer` | `Developer` |
   473	| `code-review` | `Documentation` | `Developer` |
   474	| `documentation` | `Synthesis` | `Documentation` |
   475	
   476	**Additional types (dynamic, per-WP routing):**
   477	
   478	| Pipeline type | PASS → to_agent (next active stage) | FAIL → to_agent (base routing) |
   479	|---|---|---|
   480	| `security-audit` | `Reviewer` (if `code-review` is next active) or subsequent active stage | `Developer` |
   481	| `release-engineering` | `Documentation` (if `documentation` is next active) or subsequent active stage | `Release Engineer` (self-rework) |
   482	
   483	> `documentation` and `release-engineering` self-rework on FAIL. All other FAIL paths route to the Developer (base routing). When the base fail-target's stage is absent from the WP's `active_pipeline_stages`, routing falls back to the first active stage's agent.
   484	
   485	**Schema:**
   486	```typescript
   487	interface HandoffNote {
   488	  from_agent: string; // PIPELINE_AGENT_MAP[type], or 'Project Manager (PM Override)' when PM override is active
   489	  to_agent: string;   // resolveNextAgent(type, activeStages) on PASS; resolveFailAgent(type, activeStages) on FAIL
   490	  timestamp: string;
   491	  notes: string[];    // The strings passed in handoff_notes
   492	}
   493	```
   494	
   495	**`ledger_complete_pipeline` guards (applied before pipeline lookup):**
   496	1. **WP status guard:** Rejects if `wp.status !== 'IN_PROGRESS'` (defense-in-depth).
   497	2. **Agent role guard:** `agent_role` must match `PIPELINE_AGENT_MAP[type]`. Exception: `agent_role === 'Project Manager'` bypasses this check (PM Override). When PM override is active, `from_agent` is set to `'Project Manager (PM Override)'`.
   498	
   499	**Consumption:** `ledger_get_next_action` and `ledger_get_next_actions` include any handoff notes addressed to the requesting agent in their response, so the next agent sees the notes immediately when they ask for their next action.
   500	

## Assistant

```json
{
  "id": "toolu_01Geb1jxTMwofkrAha1t9Exr",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
    "offset": 500,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01Geb1jxTMwofkrAha1t9Exr`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
  "offset": 500,
  "limit": 200
}
```

## Tool Result

   501	> Full specification: [Workflow Specification §9, §12](../workflow-specification/pipeline-routing.md).
   502	
   503	---
   504	
   505	### 23. Pipeline Comments Have No Agent Field
   506	
   507	**Rule:** Pipeline-level comments do not include an `agent` field. The agent is inferred from the pipeline type.
   508	
   509	**Convention:**
   510	- `implementation` pipeline → Developer
   511	- `qa` pipeline → QA
   512	- `code-review` pipeline → Reviewer
   513	- `documentation` pipeline → Documentation
   514	
   515	**Contrast:** Project-level comments include an explicit `agent` field because they are not tied to a specific pipeline.
   516	
   517	---
   518	
   519	### 24. Incident Comments Require Context
   520	
   521	**Rule:** When adding a project comment with `type: 'incident'`, the `context` field is required.
   522	
   523	**Enforcement:** `ledger_add_project_comment` throws an error if `type === 'incident'` and `context` is missing.
   524	
   525	**Required context fields:**
   526	- `os` — Operating system where incident occurred
   527	- `tool` — Tool or command that caused the incident
   528	- `work_package` (optional) — Associated work package
   529	- `resolved` — Whether the incident is resolved
   530	- `workaround` (optional) — Workaround description
   531	
   532	---
   533	
   534	## Concurrency Constraints
   535	
   536	### 25. Lock Timeout Is 10 Seconds
   537	
   538	**Rule:** File locks have a stale timeout of 10 seconds. Locks older than this are considered abandoned and can be forcibly acquired.
   539	
   540	**Implication:** If a process crashes while holding a lock, other processes will wait up to 10 seconds before retrying.
   541	
   542	---
   543	
   544	### 26. Lock Retry Count Is 50
   545	
   546	**Rule:** Lock acquisition is retried up to 50 times with 200ms–1000ms exponential backoff before failing.
   547	
   548	**Total retry window:** ~10–50 seconds, ensuring coverage of the 10s stale timeout.
   549	
   550	---
   551	
   552	## Testing Constraints
   553	
   554	> **CI gate:** The MCP server Vitest test suite (`npm test` in `mcp-server/`) is enforced on every push and pull request to `main` via `.github/workflows/ci.yml` (`mcp-server-tests` job, Node.js 20). All tests must pass before a PR can be merged.
   555	
   556	### 27. Test Timeout Is 10 Seconds
   557	
   558	**Rule:** All Vitest tests have a default timeout of 10 seconds.
   559	
   560	**Configuration:** Set in `vitest.config.ts`.
   561	
   562	**Rationale:** Integration tests may involve multiple file I/O operations and lock acquisitions.
   563	
   564	---
   565	
   566	### 28. Prefer Real Implementations Over `vi.mock` for Agent Registry and Ledger Tests
   567	
   568	**Rule:** When writing tests that involve the agent registry (`discoverAgents`, `isRegistryLoaded`, `getAgentHandle`, `getAgentId`) or `LedgerStore`, use the real implementations backed by a temporary directory rather than `vi.mock`.
   569	
   570	**Pattern:**
   571	```typescript
   572	import { discoverAgents, resetRegistry } from '../../src/utils/agent-registry.js';
   573	
   574	beforeEach(async () => {
   575	  tempDir = await mkdtemp(join(tmpdir(), 'test-'));
   576	  agentDir = join(tempDir, 'agents');
   577	  await mkdir(agentDir);
   578	  store = new LedgerStore(tempDir);
   579	});
   580	
   581	afterEach(async () => {
   582	  resetRegistry();
   583	  await rm(tempDir, { recursive: true, force: true });
   584	  await rm(agentDir, { recursive: true, force: true });
   585	});
   586	```
   587	
   588	**Rationale:** `vi.mock` creates module-level side-effects that can leak across test files, especially with ES module hoisting. Using real implementations with `resetRegistry()` cleanup eliminates mock side-effects, provides genuine end-to-end coverage, and is consistent with the approach in `tests/utils/agent-registry.test.ts`.
   589	
   590	**Reserve `vi.mock` for:** Code paths that touch the network, spawn child processes, or produce uncontrollable side-effects that cannot be isolated with a temp directory.
   591	
   592	---
   593	
   594	### 29. Always Supply an Isolated Ledger Root When Constructing `LedgerStore` in Tests
   595	
   596	**Rule:** Every test file that constructs a `LedgerStore` **must** pass a `mkdtemp`-based temporary directory as the second `ledgerRoot` argument. Omitting the argument (or passing the real `storage/ledger/` path) causes the store to write to production storage, accumulating stale artifact directories across CI and local runs.
   597	
   598	**Preferred pattern — use the shared helper:**
   599	```typescript
   600	import { createTempStore, cleanupTempStore } from '../helpers/create-temp-store.js';
   601	
   602	let handle: Awaited<ReturnType<typeof createTempStore>>;
   603	
   604	beforeEach(async () => {
   605	  handle = await createTempStore(join(tmpdir(), '2026-01-01-test-project'));
   606	});
   607	
   608	afterEach(async () => {
   609	  await cleanupTempStore(handle);
   610	});
   611	```
   612	
   613	**Why a helper?** `createTempStore(planPath)` in `tests/helpers/create-temp-store.ts` always injects a fresh `mkdtemp` root, making correct isolation the path of least resistance. Never construct `new LedgerStore(path)` with a single argument inside any test.
   614	
   615	**Anti-pattern (forbidden):**
   616	```typescript
   617	// ❌ WRONG — writes to production storage/ledger/
   618	const store = new LedgerStore('/absolute/path/to/my-plan');
   619	```
   620	
   621	---
   622	
   623	### 30. `afterEach` Teardown Variables Must Be Declared in the Same `describe` Scope
   624	
   625	**Rule:** Variables cleaned up in an `afterEach` block (e.g. a temp directory path) must be declared in the same `describe` block's scope, not in an outer scope. Referencing a variable from an outer scope is a silent bug — the inner `afterEach` compiles and runs but cleans up the *outer* variable, leaving the inner temp directory on disk.
   626	
   627	**Pattern:**
   628	```typescript
   629	describe('my feature', () => {
   630	  let tempDir: string;          // ← declared here
   631	  let store: LedgerStore;
   632	
   633	  beforeEach(async () => {
   634	    tempDir = await mkdtemp(join(tmpdir(), 'my-feature-'));
   635	    store = new LedgerStore(MY_PLAN_PATH, tempDir);
   636	  });
   637	
   638	  afterEach(async () => {
   639	    await rm(tempDir, { recursive: true, force: true }); // ← same scope ✅
   640	  });
   641	});
   642	```
   643	
   644	**Anti-pattern:**
   645	```typescript
   646	let tempLedgerRoot: string; // ← outer scope
   647	
   648	describe('nested', () => {
   649	  let tempDir: string;      // ← different name / inner scope
   650	
   651	  beforeEach(async () => { tempDir = await mkdtemp(…); });
   652	
   653	  afterEach(async () => {
   654	    await rm(tempLedgerRoot, { recursive: true, force: true }); // ❌ wrong variable
   655	  });
   656	});
   657	```
   658	
   659	---
   660	
   661	## Module System Constraints
   662	
   663	### 31. All Imports Must Use .js Extensions
   664	
   665	**Rule:** Even when importing TypeScript files, use `.js` extensions.
   666	
   667	**Example:**
   668	```typescript
   669	// ✅ CORRECT
   670	import { LedgerStore } from '../storage/ledger-store.js';
   671	
   672	// ❌ WRONG
   673	import { LedgerStore } from '../storage/ledger-store';
   674	```
   675	
   676	**Rationale:** Node16 module resolution requires explicit file extensions for ESM.
   677	
   678	---
   679	
   680	### 32. No Default Exports
   681	
   682	**Convention:** All exports are named exports. No default exports are used.
   683	
   684	**Rationale:** Improves refactoring and tooling support.
   685	
   686	---
   687	
   688	## Validation Constraints
   689	
   690	### 33. All Reads Are Validated
   691	
   692	**Rule:** Every file read operation validates the JSON against a Zod schema before returning data.
   693	
   694	**Enforcement:** `LedgerStore.readRootIndex()` and `LedgerStore.readWorkPackage()` both parse and validate.
   695	
   696	**Failure modes:**
   697	- File not found → `ENOENT` error
   698	- Malformed JSON → `SyntaxError`
   699	- Schema mismatch → Zod validation error
   700	

## Assistant

```json
{
  "id": "toolu_019pv5QZ1NziAA2hEvD1pq86",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
    "offset": 700,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_019pv5QZ1NziAA2hEvD1pq86`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
  "offset": 700,
  "limit": 200
}
```

## Tool Result

   701	---
   702	
   703	### 34. All Writes Are Validated
   704	
   705	**Rule:** Every file write operation validates data against a Zod schema before writing.
   706	
   707	**Enforcement:** `LedgerStore.writeRootIndex()` and `LedgerStore.writeWorkPackage()` call `Schema.parse()` before writing.
   708	
   709	**Rationale:** Prevents writing invalid data to disk.
   710	
   711	---
   712	
   713	## Counter Self-Healing
   714	
   715	### 35. Project Status Tool Auto-Corrects Counters and Project Status
   716	
   717	**Rule:** `ledger_get_project_status` recomputes `total_work_packages`, `pending_work_packages`, and the project `status` from the `work_packages` array on every invocation.
   718	
   719	**Behavior:**
   720	- If counters are incorrect, they are silently corrected.
   721	- If `status === 'READY'` and any WP is `IN_PROGRESS`, status is healed to `IN_PROGRESS`.
   722	- If `status === 'BLOCKED'` and no WP is actually `BLOCKED`, status is healed to `IN_PROGRESS` (pending WPs exist) or `READY` (no pending WPs).
   723	- If `status === 'IN_PROGRESS'` and all WPs are complete (pending = 0, WPs exist), status is healed to `COMPLETE`.
   724	- If `status === 'COMPLETE'` and pending WPs exist, status is healed back to `IN_PROGRESS`.
   725	- An empty project (no WPs) is never auto-healed to `COMPLETE`.
   726	- Healing rules are mutually exclusive and applied in order; only the first matching rule fires.
   727	- The root index is rewritten only when a correction is made.
   728	
   729	**Rationale:** Provides fault tolerance against bugs that might cause counter or status drift.
   730	
   731	---
   732	
   733	## Development & Build Constraints
   734	
   735	### 36. Changelog Is the Source of Truth for Versioning
   736	
   737	**Rule:** All version changes must be made in `changelog.md` first, then synced to `package.json`.
   738	
   739	**Rationale:** Maintains a single source of truth and ensures version history is documented.
   740	
   741	**Process:**
   742	1. Update `changelog.md` with new version header:
   743	   ```markdown
   744	   ## v1.0.2 - 2026-02-20
   745	   
   746	   ### Added
   747	   - New feature...
   748	   ```
   749	2. Run `npm run sync-version` to extract version and update `package.json`
   750	3. The MCP server will display the version at startup in STDERR
   751	
   752	**Anti-pattern:**
   753	```bash
   754	# ❌ WRONG — manually editing package.json version
   755	vim package.json  # Don't do this!
   756	```
   757	
   758	**Correct pattern:**
   759	```bash
   760	# ✅ CORRECT — update changelog first, then sync
   761	vim changelog.md  # Add new version
   762	npm run sync-version
   763	```
   764	
   765	---
   766	
   767	### 37. Version Sync Runs Automatically Before Dev
   768	
   769	**Rule:** The `predev` hook ensures version is synced before running the development server.
   770	
   771	**Implication:** You can skip manual `npm run sync-version` if running `npm run dev` — it happens automatically.
   772	
   773	**Manual sync needed when:**
   774	- Building for distribution
   775	- Running in production
   776	- CI/CD pipelines
   777	- Testing version display without starting server
   778	
   779	---
   780	
   781	### 38. Server Version Displays at Startup
   782	
   783	**Rule:** The MCP server logs its version to STDERR on startup.
   784	
   785	**Example output:**
   786	```
   787	[project-ledger-mcp] Server v1.0.1 started successfully
   788	[project-ledger-mcp] Transport: STDIO
   789	[project-ledger-mcp] Registered tools: ledger_get_project_status, ...
   790	```
   791	
   792	**Purpose:** Allows users and CI systems to verify which version is running in their project.
   793	
   794	---
   795	
   796	### 39. Reopening a COMPLETE Work Package Requires Project Manager or Documentation Agent
   797	
   798	**Rule:** When transitioning a work package from `COMPLETE` back to `IN_PROGRESS`, the calling `agent` MUST be `"Project Manager"` (or `"Project Manager Agent"`) or `"Documentation"` (or `"Documentation Agent"`). All other agents are rejected.
   799	
   800	**Enforcement:** Hard guard in `updateWorkPackageStatus()` in `src/tools/work-package.ts`, applied before the status mutation.
   801	
   802	**Error message format:**
   803	```
   804	Cannot reopen work package WP-XXX: only the Project Manager or Documentation agent may transition COMPLETE → IN_PROGRESS.
   805	Hand off to the Project Manager or Documentation agent to formally reopen this work package.
   806	```
   807	
   808	**Rationale:** Prevents developer or QA agents from silently reopening completed work, bypassing the formal re-planning and documentation steps.
   809	
   810	**Additional effect:** On `COMPLETE → IN_PROGRESS`, rework state is fully reset: `rework_counts` is set to `{}`, `rework_count` is set to `0`, `root.synthesis_generated` is cleared, and `root.synthesis_generated_at` is set to `null`. This ensures that a reopened WP starts with a clean rework slate and prevents the Synthesis agent from being gated by stale synthesis state.
   811	
   812	---
   813	
   814	### 40. `READY → IN_PROGRESS` Must Use `ledger_claim_work_package`
   815	
   816	**Rule:** `ledger_update_work_package_status` rejects `status: 'IN_PROGRESS'` when the WP is currently `READY`. The caller must use `ledger_claim_work_package` instead.
   817	
   818	**Enforcement:** Early-return guard in `updateWorkPackageStatus()` that throws an actionable error naming `ledger_claim_work_package` as the correct tool.
   819	
   820	**Rationale:** `ledger_claim_work_package` enforces dependency checks and agent identity checks that `ledger_update_work_package_status` does not replicate.
   821	
   822	---
   823	
   824	### 41. `IN_PROGRESS → READY` (Unclaim) Requires No Active Pipelines
   825	
   826	**Rule:** When transitioning a WP from `IN_PROGRESS` back to `READY`, all pipelines must be in a terminal state (non-`IN_PROGRESS`). If any pipeline is currently `IN_PROGRESS`, the transition is rejected with an actionable error.
   827	
   828	**Side effect:** On success, `assigned_to` is cleared in both the WP detail file and the root index summary.
   829	
   830	**Enforcement:** Guard in `updateWorkPackageStatus()` step 4 in `src/tools/work-package.ts`.
   831	
   832	---
   833	
   834	### 42. `BLOCKED → BLOCKED` Replaces the Blocker with Guards
   835	
   836	**Rule:** A `BLOCKED` work package can be re-blocked with a different `blocked_by` object. This early-return path:
   837	1. **Agent guard:** Only the `"Project Manager"` (or `"Project Manager Agent"`) or the current `wp.assigned_to` may replace a blocker.
   838	2. **Type guard:** Changing a `'dependency'`-type blocker to a non-dependency type (or vice versa) is rejected. Dependency blockers are managed automatically by the system; manual replacement of dependency blockers is disallowed.
   839	3. **Side effect:** `status_changed_at` and `root.last_updated` are set; `pending_work_packages` is unchanged (status remains `BLOCKED`).
   840	
   841	**Enforcement:** Early-return guard in `updateWorkPackageStatus()` step 1a.
   842	
   843	---
   844	
   845	### 43. `IN_PROGRESS → BLOCKED` and `IN_PROGRESS → CANCELLED` Auto-Cancel Active Pipelines
   846	
   847	**Rule:** When a WP transitions from `IN_PROGRESS` to `BLOCKED` or `CANCELLED`, all currently `IN_PROGRESS` pipelines are automatically cancelled. Each cancelled pipeline receives `auto_cancelled: true` to distinguish it from deliberate FAIL pipelines.
   848	
   849	**Effect on rework detection:** Auto-cancelled pipelines are excluded from both direct and downstream rework detection in `ledger_start_pipeline` (see constraint 21).
   850	
   851	**Enforcement:** Pipeline auto-cancellation via `autoCancelActivePipelines(wp, reason)` helper called at steps 8a/8b in `updateWorkPackageStatus()` in `src/tools/work-package.ts`.
   852	
   853	---
   854	
   855	### 44. `→ COMPLETE` Freshness Check
   856	
   857	**Rule:** When transitioning a WP to `COMPLETE`, a freshness check is applied: the most recent non-auto-cancelled `documentation` pipeline PASS must have been recorded **after** the most recent `implementation` pipeline start. If the doc PASS predates the impl start (stale doc), the transition is rejected.
   858	
   859	**Exception:** If no `implementation` pipeline exists, or if no `documentation` pipeline has a PASS, the check is skipped (absent timestamps are accepted).
   860	
   861	**Absent timestamp permissive default:** If the most recent `documentation` pipeline lacks a `completed_at` timestamp, or if the most recent `implementation` pipeline lacks a `started_at` timestamp, the freshness check is skipped and the `→ COMPLETE` transition is allowed.
   862	
   863	**Enforcement:** Freshness check in `canCompleteWorkPackage()` or in `updateWorkPackageStatus()` step 2b.
   864	
   865	**Rationale:** Prevents a WP from being completed with documentation that was written before the current implementation cycle, ensuring the docs always reflect the current implementation.
   866	
   867	---
   868	
   869	### 45. `status_changed_at` Is Set on Every Status Transition
   870	
   871	**Rule:** The `status_changed_at` field on a work package is updated on every successful status transition, including `BLOCKED → BLOCKED` blocker replacements (even though the status value itself doesn't change).
   872	
   873	**Field type:** UTC ISO 8601 timestamp string (same format as `now()`).
   874	
   875	**Enforcement:** Set in `updateWorkPackageStatus()` after every mutation path (early-return paths and main path).
   876	
   877	---
   878	
   879	### 46. Work Package `assigned_to` Always Starts as `null`
   880	
   881	**Rule:** When creating a work package via `ledger_create_work_package`, the `assigned_to` input field is accepted silently but **ignored**. Both the WP detail file and the root index summary are written with `assigned_to: null`.
   882	
   883	**Rationale (§9b.1):** Assignment is managed by `ledger_claim_work_package` (transitions to `IN_PROGRESS`) and cleared by `IN_PROGRESS → READY` (unclaim). Pre-populating at creation time bypasses these guards.
   884	
   885	**Enforcement:** `createWorkPackage()` in `src/tools/work-package.ts` overwrites the input value.
   886	
   887	---
   888	
   889	### 47. New BLOCKED Work Packages Receive An Auto-Assigned `blocked_by`
   890	
   891	**Rule:** When a work package's initial status is `BLOCKED` (because at least one dependency is not terminal), `blocked_by` is automatically populated:
   892	```typescript
   893	{ type: 'dependency', description: 'Dependency WP-XXX is not complete', blocking_work_package: 'WP-XXX' }
   894	```
   895	where `WP-XXX` is the first unmet dependency.
   896	
   897	**Enforcement:** Inside `createWorkPackage()` initial status determination.
   898	
   899	---
   900	

## Assistant

Now I have a clear picture of both files. Let me also look for any other mentions of "mutual exclusivity" or related phrases in the constraints file to ensure I catch everything:

```json
{
  "id": "toolu_01VWfyBE3ByyYajYc7HCkvH1",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "mutual",
    "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest",
    "output_mode": "content"
  },
  "name": "grep",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_016jKVB1rZ9bRy5QW3yhcEnv",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "mutually exclusive",
    "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools",
    "output_mode": "content"
  },
  "name": "grep",
  "type": "tool_use"
}
```

**Tool call:** `grep` (id: `toolu_01VWfyBE3ByyYajYc7HCkvH1`)

```json
{
  "pattern": "mutual",
  "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest",
  "output_mode": "content"
}
```

**Tool call:** `grep` (id: `toolu_016jKVB1rZ9bRy5QW3yhcEnv`)

```json
{
  "pattern": "mutually exclusive",
  "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools",
  "output_mode": "content"
}
```

## Tool Result

/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/api-surface.md:
  1230: const mutuallyExclusivePaths: (args: { project_path?: string | null; cwd_path?: string | null }) => boolean;
  1232: // Error message paired with mutuallyExclusivePaths.
/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md:
  726: - Healing rules are mutually exclusive and applied in order; only the first matching rule fires.
  1270: - The predicate `mutuallyExclusivePaths` and the constant `MUTUAL_EXCLUSIVITY_PATH_MSG` remain exported from `src/utils/path-validator.ts` for backward compatibility and test coverage. They are **not used in production tool files**.
  1282:   .refine(mutuallyExclusivePaths, { message: MUTUAL_EXCLUSIVITY_PATH_MSG });
  1287: // ✅ CORRECT — plain ZodObject; mutual exclusivity is enforced inside resolveProjectPath()
  1488: }).refine(mutuallyExclusivePaths, { message: MUTUAL_EXCLUSIVITY_PATH_MSG });

## Tool Result

No matches found

## Assistant

```json
{
  "id": "toolu_01RzH4ViGbdjzZUQ5Fz7WuCu",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
    "offset": 900,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01RzH4ViGbdjzZUQ5Fz7WuCu`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
  "offset": 900,
  "limit": 200
}
```

## Tool Result

   901	### 48. Creating a Work Package Must Not Introduce a Dependency Cycle
   902	
   903	**Rule:** Before persisting, `createWorkPackage` calls `hasCycle(newWpId, deps, allExistingWps)` (BFS) to verify the new dependency edges don't form a circular dependency. If a cycle is detected, the creation is rejected.
   904	
   905	**Error message format:**
   906	```
   907	Dependency cycle detected: WP X would create a circular dependency.
   908	```
   909	
   910	**Scope:** `hasCycle` checks forward-reference cycles among existing WPs. Simultaneous batch creation bypasses cycle detection — WPs should be created sequentially.
   911	
   912	**Enforcement:** `hasCycle()` pure function at module scope in `src/tools/work-package.ts`, called in `createWorkPackage` step 3b.
   913	
   914	---
   915	
   916	### 49. Acceptance Criteria Cannot Be Empty or Whitespace-Only
   917	
   918	**Rule:** Each string in the `acceptance_criteria` array must be non-empty and non-whitespace after trimming. An empty string or a string containing only spaces/tabs/newlines is rejected.
   919	
   920	**Error message format:**
   921	```
   922	Acceptance criteria cannot be empty or whitespace-only.
   923	```
   924	
   925	**Enforcement:** Validation loop in `createWorkPackage()` before WP creation, supplementing the Zod-level `.min(1)` array constraint.
   926	
   927	---
   928	
   929	### 50. Only CLAIMABLE_ROLES Can Claim Work Packages
   930	
   931	**Rule:** The `agent` field passed to `ledger_claim_work_package` must be a claimable role. 
   932	
   933	**Non-claimable roles:** `Planner`, `Planner Agent`, `Synthesis`, `Synthesis Agent` — these orchestrating roles are excluded from claiming WPs.
   934	
   935	**Claimable roles:** `Developer`, `Developer Agent`, `QA`, `QA Agent`, `Reviewer`, `Reviewer Agent`, `Documentation`, `Documentation Agent`, `Project Manager`, `Project Manager Agent`.
   936	
   937	**Guard ordering:** The CLAIMABLE_ROLES guard fires at step 1b — unconditionally, immediately after the `READY` status guard and **before** the assignment guard (step 2) and override-auth guard (step 2b). Consequence: a non-claimable role always receives the role error regardless of the WP's `assigned_to` field or whether `override: true` is passed.
   938	
   939	**Enforcement:** `CLAIMABLE_ROLES` is a named export at module scope in `src/tools/work-package.ts`, checked in `claimWorkPackage` step 1b. It is derived programmatically from `AGENT_ROLES` by filtering out `ORCHESTRATING_ROLES` (defined in `src/utils/constants.ts`), so adding a new orchestrating role automatically removes it from the claimable set without requiring manual updates.
   940	
   941	---
   942	
   943	### 52. `agent_role` Is Required for `ledger_start_pipeline` and `ledger_complete_pipeline`
   944	
   945	**Rule:** Both `ledger_start_pipeline` and `ledger_complete_pipeline` require an `agent_role` parameter. The value must match the pipeline type's owner role (per `PIPELINE_AGENT_MAP`). Calls that omit `agent_role` or provide a mismatched role are rejected with a descriptive error.
   946	
   947	**Exception:** `agent_role: 'Project Manager'` (or `'Project Manager Agent'`) bypasses the type-to-agent match check for any pipeline type (PM Override). When PM override is active, `startPipeline` adds a `[PM Override]` marker to the pipeline summary and `completePipeline` sets the handoff note's `from_agent` to `'Project Manager (PM Override)'`.
   948	
   949	**Enforcement:** Agent role guard in `startPipeline()` and `completePipeline()` in `src/tools/pipeline.ts` (steps 1b and 2b respectively), applied after the WP status guard.
   950	
   951	**Rationale:** Prevents agents from starting or completing pipelines outside their designated stage, ensuring the pipeline type-to-agent assignment invariant is upheld at runtime.
   952	
   953	---
   954	
   955	### 51. `propagateDependencyReblock` Auto-Cancels IN_PROGRESS Pipelines
   956	
   957	**Rule:** When `propagateDependencyReblock` transitions a non-COMPLETE, non-CANCELLED, non-BLOCKED dependent WP back to `BLOCKED`, all currently `IN_PROGRESS` pipelines on that WP are automatically cancelled with `auto_cancelled: true` (consistent with the `IN_PROGRESS → BLOCKED` behavior enforced by `updateWorkPackageStatus`).
   958	
   959	**Additional behaviors:**
   960	- **COMPLETE dependents:** For each `COMPLETE` WP that lists the reopened WP as a dependency, a warning comment is appended to its last pipeline (type: `"warning"`, priority: `"high"`).
   961	- **`synthesis_generated` reset:** If any WP was re-blocked (i.e., `candidates.length > 0`), `root.synthesis_generated` is reset to `false` and `root.synthesis_generated_at` is set to `null` to ensure the Synthesis agent must re-run.
   962	- If no candidates were re-blocked, `synthesis_generated` and `synthesis_generated_at` are **not** changed.
   963	
   964	**Enforcement:** `propagateDependencyReblock()` in `src/tools/work-package.ts`.
   965	
   966	---
   967	
   968	## Manifest Documentation Constraints
   969	
   970	### 53. No Implementation Provenance in Manifest Documents
   971	
   972	**Rule:** Project manifest documents (`api-surface.md`, `constraints.md`, `data-flows.md`, etc.) describe the **current state** of the codebase. They must not contain work package IDs, plan references, or other implementation-history markers (e.g., `WP-003`, `added in WP-005`, `wired in WP-004`).
   973	
   974	**Where provenance belongs:** Plan documents, synthesis reports, and changelog entries — not the manifest.
   975	
   976	**Rationale:** WP IDs are scoped to individual plans. A reader who has not ingested the plan history cannot resolve `WP-006` to a meaningful context. Provenance markers also accumulate over time and add noise without aiding comprehension of current behavior.
   977	
   978	**What is allowed:** References to `WP-###` as a *data format specifier* (e.g., `work_package_id: string // WP-### format`) are fine — these describe the runtime data model, not implementation history.
   979	
   980	---
   981	
   982	## GUI API Constraints
   983	
   984	### 40. All Slug- and WpId-Accepting GUI Handlers Must Call Their Path-Traversal Guard First
   985	
   986	**Rule:** Every GUI API handler in `gui/api.ts` that accepts a path segment parameter must call its corresponding guard as the **first** (slug) or **second** (wpId) statement, before any other processing.
   987	
   988	**Guards:**
   989	
   990	| Guard | Parameter | Placement | Affected handlers |
   991	|-------|-----------|-----------|-------------------|
   992	| `assertSafeSlug(slug)` | project slug | 1st statement | `handleGetProject`, `handleListWorkPackages`, `handleGetWorkPackage`, `handleDeleteProject`, `handleGetPlanDocument` |
   993	| `assertSafeWpId(wpId)` | work-package ID | 2nd statement (after `assertSafeSlug`) | `handleGetWorkPackage` |
   994	
   995	**Rejection criteria (both guards):** throws `ApiError` with code `NOT_FOUND` (HTTP 404) if the value:
   996	- is empty (`''`)
   997	- contains a forward slash (`/`)
   998	- contains a double dot (`..`)
   999	
  1000	**Rationale:** Returning `NOT_FOUND` (rather than `FORBIDDEN`) on traversal attempts is intentional — it avoids leaking structural information about the server's file system to potential attackers. Using HTTP 404 is consistent with the standard "project not found" response.
  1001	
  1002	**Implementation:** Both guards are module-private to `gui/api.ts` (not exported). They must not be bypassed or called after other parameter-dependent operations.
  1003	
  1004	**Acceptance criteria wording:** When writing AC for test cases that exercise `assertSafeSlug` rejection, use:
  1005	> *"Invalid slug (e.g. path-traversal attempt) returns 404 NOT_FOUND."*
  1006	
  1007	Do **not** write `"400 VALIDATION_ERROR"` — the guard deliberately returns `NOT_FOUND` (not `VALIDATION_ERROR`) to mask traversal detection. (See [error-ledger.md](../../../../../history/error-ledger.md) — deviation recorded in 2026-03-04-project-reset-rework-1 synthesis.)
  1008	
  1009	---
  1010	
  1011	## Cross-Platform Constraints
  1012	
  1013	### 54. All Code Must Run on Windows, macOS, and Linux
  1014	
  1015	**Rule:** The MCP server must work on all three supported platforms (Windows, macOS, Linux). Do not introduce OS-specific APIs without a cross-platform fallback. Use `path.join()` / `path.resolve()` for all file paths — never hardcode `/` or `\` separators.
  1016	
  1017	**File locking:** Uses `proper-lockfile` (cross-platform npm package). Do not replace with a platform-specific alternative.
  1018	
  1019	**Rationale:** The workspace-wide cross-platform policy (see root `AGENTS.md` → Cross-Platform Policy) applies to all sub-projects. The MCP server runs alongside the user's IDE on their desktop OS.
  1020	
  1021	---
  1022	
  1023	## Gotchas
  1024	
  1025	### ⚠️ Gotcha 1: Revision Only Increments on COMPLETE → IN_PROGRESS
  1026	
  1027	The `revision` field only increments when a work package transitions from `COMPLETE` back to `IN_PROGRESS`. It does not increment on other status changes.
  1028	
  1029	---
  1030	
  1031	### ⚠️ Gotcha 2: Lock File Persists After Server Exit
  1032	
  1033	The `.lock` file inside `storage/ledger/{slug}/` is not automatically deleted when the server exits. It will be left on disk and overwritten on the next lock acquisition.
  1034	
  1035	**Implication:** Safe to ignore — the lock system handles stale locks automatically.
  1036	
  1037	---
  1038	
  1039	### ⚠️ Gotcha 3: Metrics Object Is Extensible
  1040	
  1041	The `metrics` object in pipelines uses `.passthrough()` in Zod, meaning it accepts additional fields beyond the predefined ones (`test_coverage`, `tests_passed`, etc.).
  1042	
  1043	**Use case:** Custom metrics for different pipeline types (e.g., `build_time`, `bundle_size`).
  1044	
  1045	---
  1046	
  1047	### ⚠️ Gotcha 4: Work Package Summaries Are Duplicates
  1048	
  1049	Work package summaries in the root index duplicate a subset of data from the work package detail files.
  1050	
  1051	**Reason:** Performance — agents can list work packages without loading all detail files.
  1052	
  1053	**Invariant:** Summaries must always match the corresponding detail files. This is enforced by `createWorkPackageWithSync()` (creation) and `updateWorkPackageWithSync()` (updates).
  1054	
  1055	---
  1056	
  1057	### ⚠️ Gotcha 6: REWORK Is Triggered Only by the Most Recent FAIL
  1058	
  1059	The REWORK recommendation in `ledger_get_next_action` is based **only on the most recent pipeline** of a given type, not any historical FAIL. A work package with pipeline history `[FAIL, PASS]` does NOT receive a REWORK recommendation — the PASS pipeline means the issue was resolved.
  1060	
  1061	**Why it matters:** Before this was corrected, a WP that failed and then passed (e.g., tests failed, bugs were fixed, tests re-run and passed) would permanently trigger a REWORK recommendation, even though the work was complete. Now only a WP whose most recent pipeline is still `FAIL` will trigger REWORK.
  1062	
  1063	**Implementation:** `isMostRecentPipelineFail(pipelines, pipelineType)` — see [Internal Testing Utilities](api-surface.md#internal-testing-utilities).
  1064	
  1065	---
  1066	
  1067	### ⚠️ Gotcha 7: Documentation Handoff Skips Dependency-Blocked WPs
  1068	
  1069	`getDocumentationHandoff` (and `getQaHandoff`, `getReviewerHandoff`) treat WPs blocked by incomplete dependencies as ineligible for their stage. If all unreviewed/undocumented WPs are dependency-blocked, the handoff returns `READY_FOR_SYNTHESIS` rather than routing the agent back to the Developer.
  1070	
  1071	**Why it matters:** Without this check, a project where the only remaining WPs are blocked by incomplete dependencies would incorrectly route the Documentation Agent back to the Developer stage, stalling the workflow.
  1072	
  1073	---
  1074	
  1075	### ⚠️ Gotcha 8: Dependency Auto-Unblocking Uses a Separate Lock
  1076	
  1077	When a work package transitions to `COMPLETE`, `propagateDependencyUnblock` automatically transitions eligible downstream dependents from `BLOCKED` to `READY`. This runs **after** the main lock in `updateWorkPackageStatus` is released — it acquires its own lock.
  1078	
  1079	**Eligibility rule:** A BLOCKED WP is auto-unblocked only when **all its dependencies are terminal (COMPLETE or CANCELLED) AND its `blocked_by.type` is `"dependency"` or absent**. WPs blocked by `"external"`, `"decision"`, or `"technical"` reasons are intentionally skipped — their blockers must be resolved manually, even if all WP dependencies complete.
  1080	
  1081	**Implication:** There is a brief window between the COMPLETE write and the unblocking write during which the root index shows the WP as COMPLETE but dependents are still BLOCKED. This is safe for single-user workflows, but would be a race condition risk in a concurrent multi-agent environment.
  1082	
  1083	---
  1084	
  1085	### ⚠️ Gotcha 9: WP ID Generation Is Max-Based, Not Length-Based
  1086	
  1087	Work package IDs are generated by scanning the highest existing numeric suffix and adding 1. This means:
  1088	- Deleting a WP does not cause ID collisions (unlike a length+1 approach)
  1089	- IDs are monotonically increasing but may have gaps (e.g., WP-001, WP-003 if WP-002 was removed)
  1090	- IDs can be 3+ digits: the schema regex `/^WP-\d{3,}$/` supports WP-001 through WP-9999+
  1091	
  1092	---
  1093	
  1094	### ⚠️ Gotcha 5: READY Status After Creation Depends on Dependencies
  1095	
  1096	When creating a work package:
  1097	- If dependencies are empty or all `COMPLETE` → Initial status is `READY`
  1098	- If any dependency is not `COMPLETE` → Initial status is `BLOCKED`
  1099	
  1100	This logic is automatic and transparent to the caller.

## Assistant

```json
{
  "id": "toolu_01CCESbTb1mx8Z8KhkaRpMNw",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
    "offset": 1100,
    "limit": 300
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01CCESbTb1mx8Z8KhkaRpMNw`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
  "offset": 1100,
  "limit": 300
}
```

## Tool Result

  1101	
  1102	---
  1103	
  1104	### ⚠️ Gotcha 10: `acceptance_criteria` Must Have At Least One Entry
  1105	
  1106	The `ledger_create_work_package` tool rejects requests with an empty `acceptance_criteria` array. Zod validation enforces `.min(1)` — at least one criterion string is required. This prevents the degenerate case of a WP that auto-passes all criterion checks.
  1107	
  1108	---
  1109	
  1110	### ⚠️ Gotcha 11: Unknown Criteria Text in `acceptance_criteria_updates` Is Appended
  1111	
  1112	When `ledger_complete_pipeline` is called with `acceptance_criteria_updates`, each update item is matched by exact criterion text:
  1113	- **Matched:** updates the `met` flag on the existing entry.
  1114	- **Not matched (unknown text):** appends a new `AcceptanceCriterion` entry `{ criterion, met }` to the WP's `acceptance_criteria` array.
  1115	
  1116	---
  1117	
  1118	### ⚠️ Gotcha 12: Pre-mutation State Capture in `updateWorkPackageWithSync` Callbacks
  1119	
  1120	**Rule:** Any variable holding pre-mutation WP or root-index state that is needed **after** the `updateWorkPackageWithSync` callback must be declared with `let` in the **outer scope** and assigned inside the callback. Variables declared with `const` inside the callback are lexically scoped to that callback and are invisible at the call site.
  1121	
  1122	**Anti-pattern:**
  1123	```typescript
  1124	// ❌ WRONG — const inside callback is NOT visible at the call site
  1125	await store.updateWorkPackageWithSync(wpId, (wp, root) => {
  1126	  const previousStatus = wp.status; // const → invisible outside callback
  1127	  wp.status = 'IN_PROGRESS';
  1128	  return { wp, root };
  1129	});
  1130	// TS2304: Cannot find name 'previousStatus'  ← compile error
  1131	console.log(previousStatus); // ReferenceError at runtime if somehow not caught by TS
  1132	```
  1133	
  1134	**Correct pattern:**
  1135	```typescript
  1136	// ✅ CORRECT — let declared in outer scope, assigned inside callback
  1137	let previousStatus = '';
  1138	await store.updateWorkPackageWithSync(wpId, (wp, root) => {
  1139	  previousStatus = wp.status; // assigns to outer-scope let
  1140	  wp.status = 'IN_PROGRESS';
  1141	  return { wp, root };
  1142	});
  1143	console.log(previousStatus); // ✅ 'READY' — visible after lock completes
  1144	```
  1145	
  1146	**Rationale:** `updateWorkPackageWithSync` (and `withLock`) discard the callback's return value for the state-capture use case. Any data produced inside the callback that is needed after it completes must be captured via closure by assigning to an outer-scope `let` variable before the callback runs. This pattern appears throughout `work-package.ts` (e.g., `let createdWpId = ''` in `createWorkPackage`). Failure to follow it produces a TS2304 compile error or, if TypeScript somehow does not catch it, a `ReferenceError` at the call site.
  1147	
  1148	**Alternative correct pattern (`| undefined` union):** When the captured value has no meaningful zero value, use `| undefined` union rather than a non-null assertion (`!`):
  1149	
  1150	```typescript
  1151	// ✅ ALSO CORRECT — | undefined union (used in project-lifecycle.ts completeSynthesis)
  1152	let result: { status: string } | undefined;
  1153	await withLock(store.storageDir, async () => {
  1154	  // ... read-modify-write ...
  1155	  result = { status: 'COMPLETE' };
  1156	});
  1157	if (!result) throw new Error('Expected result to be set inside lock');
  1158	// result is narrowed to { status: string } here
  1159	```
  1160	
  1161	Prefer `| undefined` over non-null assertion (`!`) when the accumulator cannot have a meaningful zero state.
  1162	
  1163	---
  1164	
  1165	## Code Style Conventions
  1166	
  1167	### 53. Test-Only Exports Must Use the `_internal` Naming Convention
  1168	
  1169	**Rule:** Any module that exposes private symbols for unit testing must export them under a single named export called `_internal`. Do **not** introduce alternative names such as `_schemas`, `_test`, or `_utils`.
  1170	
  1171	**Pattern:**
  1172	```typescript
  1173	/**
  1174	 * @internal — exported for unit testing only.
  1175	 */
  1176	export const _internal = {
  1177	  MyPrivateClass,
  1178	  MyInternalSchema,
  1179	  myHelperFunction,
  1180	};
  1181	```
  1182	
  1183	**Rationale:** Consistency and grep-ability. A single naming convention makes it trivial to audit test-only surface (`grep -r '_internal'`) and eliminates `_schemas` / `_test` divergence. The convention was introduced in `work-package.ts` and standardised across all modules in 2026-02-28 (WP-009).
  1184	
  1185	**Enforcement:** `_schemas` exports were renamed to `_internal` in `pipeline.ts` and `observations.ts`. Do not re-introduce `_schemas` or any alternate name.
  1186	
  1187	---
  1188	
  1189	### 54. Prefer `for-of` Loops Over Indexed `for` Loops
  1190	
  1191	**Rule:** Use `for-of` loops for array iteration. Avoid `for (let i = 0; i < arr.length; i++)` indexed loops unless the index itself is required for logic, or a performance constraint is documented.
  1192	
  1193	**When an indexed loop is unavoidable** (e.g. pairwise comparison where both `i-1` and `i` are needed), use non-null-asserted access (`arr[i]!`) with an inline comment explaining the in-bounds guarantee:
  1194	
  1195	```typescript
  1196	// TypeScript is compiled with noUncheckedIndexedAccess so array[i] returns T | undefined.
  1197	// The loop invariant (i < arr.length) guarantees arr[i] is defined — safe to assert.
  1198	for (let i = 1; i < pipelines.length; i++) {
  1199	  const prev = pipelines[i - 1]!; // in-bounds: i >= 1
  1200	  const curr = pipelines[i]!;     // in-bounds: i < pipelines.length
  1201	}
  1202	```
  1203	
  1204	**Context:** The project enables `noUncheckedIndexedAccess` in `tsconfig.json`. This means array element access returns `T | undefined`, which requires either a null-check or a `!` assertion. The `for-of` pattern avoids indexed access entirely and is therefore preferred.
  1205	
  1206	---
  1207	
  1208	### 55. Test Helper Infrastructure Mandate
  1209	
  1210	**Rule:** All new test files **must** import shared fixture factories and test utilities from `tests/helpers/fixtures.ts` and `tests/helpers/test-utils.ts`.
  1211	
  1212	**(a)** Any new test file that needs a project root index, WP detail object, or ledger directory must use the canonical factories from `tests/helpers/fixtures.ts` (e.g. `makeProject`, `makeWpDetail`, `injectLedgerDir`, `nowFloor`).
  1213	
  1214	**(b)** Defining a local test-scope fixture factory function is **prohibited** when a canonical equivalent already exists in `tests/helpers/fixtures.ts`. If the helper does not yet exist and is needed by multiple tests, add it to `tests/helpers/` first rather than duplicating it inline.
  1215	
  1216	**(c)** **Rationale:** Prevents per-file fixture divergence, eliminates test-replica maintenance burden, and ensures fixture behaviour (field defaults, schema shape, timestamps) stays consistent across the entire test suite.
  1217	
  1218	**Anti-pattern:**
  1219	```typescript
  1220	// ❌ WRONG — local factory duplicates the canonical makeWpDetail from tests/helpers/fixtures.ts
  1221	function makeTestWp(overrides: Partial<WorkPackageDetail> = {}): WorkPackageDetail {
  1222	  return {
  1223	    work_package_id: 'WP-001',
  1224	    status: 'READY',
  1225	    revision: 0,
  1226	    pipelines: [],
  1227	    assigned_to: null,
  1228	    dependencies: [],
  1229	    acceptance_criteria: [],
  1230	    ...overrides,
  1231	  };
  1232	}
  1233	```
  1234	
  1235	**Correct pattern:**
  1236	```typescript
  1237	// ✅ CORRECT — import the canonical factory; field defaults and schema shape are guaranteed
  1238	import { makeWpDetail } from '../helpers/fixtures.js';
  1239	
  1240	const wp = makeWpDetail({ work_package_id: 'WP-001', status: 'READY' });
  1241	```
  1242	
  1243	---
  1244	
  1245	### 56. JSDoc Convention for Captured-Closure Variables
  1246	
  1247	**Rule:** When using the captured-closure pattern (an outer-scope `let` written inside a `withLock` / `updateWorkPackageWithSync` callback and read after the call returns), add a brief `// captured via closure in lock callback` inline comment on the `let` declaration.
  1248	
  1249	**Example:**
  1250	```typescript
  1251	let autoFinalizeResult: 'finalized' | 'blocked' | null = null; // captured via closure in lock callback
  1252	await store.updateWorkPackageWithSync(wpId, (wp, root) => {
  1253	  // ... logic that may set autoFinalizeResult ...
  1254	  autoFinalizeResult = 'finalized';
  1255	  return { wp, root };
  1256	});
  1257	if (autoFinalizeResult === 'finalized') { /* ... */ }
  1258	```
  1259	
  1260	**Rationale:** The pattern is non-obvious to contributors unfamiliar with the lock-callback design. Without the comment, reviewers may assume the variable is always `null` after the call (it isn't — the callback executed synchronously within the lock and the `let` is live). See Gotcha 12 for a full explanation of the captured-closure mechanics.
  1261	
  1262	---
  1263	
  1264	### 57. Mutual Exclusivity of `project_path` and `cwd_path` — Runtime Guard via `resolveProjectPath()`
  1265	
  1266	**Rule:** Mutual exclusivity of `project_path` and `cwd_path` is enforced at runtime by `resolveProjectPath()`, **not** by a Zod `.refine()` on the outer schema. Do **not** add `.refine()`, `.transform()`, or `.superRefine()` to the outer `z.object()` of any tool schema.
  1267	
  1268	**Enforcement:**
  1269	- `resolveProjectPath()` (`src/utils/path-validator.ts`) throws `Error(MUTUAL_EXCLUSIVITY_PATH_MSG)` at the top of its body when both `project_path` and `cwd_path` are truthy. Every tool handler that accepts both optional path fields calls `resolveProjectPath()` — the guard fires unconditionally.
  1270	- The predicate `mutuallyExclusivePaths` and the constant `MUTUAL_EXCLUSIVITY_PATH_MSG` remain exported from `src/utils/path-validator.ts` for backward compatibility and test coverage. They are **not used in production tool files**.
  1271	- Schemas that only contain `project_path` (mandatory) or only `cwd_path` — but not both as optional fields — are exempt from this consideration. `DetectProjectSchema`, `InitializeProjectSchema`, and `ListProjectsSchema` fall into this category.
  1272	
  1273	**Anti-pattern:**
  1274	```typescript
  1275	// ❌ WRONG — .refine() converts ZodObject → ZodEffects. The MCP SDK cannot extract properties
  1276	// from ZodEffects, resulting in empty { properties: {}, required: [] } in tools/list responses.
  1277	const GetWorkPackageSchema = z.object({
  1278	  project_path: z.string().optional().describe('…'),
  1279	  cwd_path:     z.string().optional().describe('…'),
  1280	  work_package_id: z.string().regex(/^WP-\d{3,}$/),
  1281	})
  1282	  .refine(mutuallyExclusivePaths, { message: MUTUAL_EXCLUSIVITY_PATH_MSG });
  1283	```
  1284	
  1285	**Correct pattern:**
  1286	```typescript
  1287	// ✅ CORRECT — plain ZodObject; mutual exclusivity is enforced inside resolveProjectPath()
  1288	const GetWorkPackageSchema = z.object({
  1289	  project_path: z.string().optional().describe('…'),
  1290	  cwd_path:     z.string().optional().describe('…'),
  1291	  work_package_id: z.string().regex(/^WP-\d{3,}$/),
  1292	});
  1293	```
  1294	
  1295	**Rationale:** `.refine()` (and `.transform()`, `.superRefine()`) on the outer `z.object()` converts it from `ZodObject` to `ZodEffects`. The MCP SDK's `zodToJsonSchema` cannot extract properties from `ZodEffects` — every affected tool emits empty `{ properties: {}, required: [] }` in the `tools/list` response, preventing AI agents from passing arguments. Centralising the check in `resolveProjectPath()` keeps all tool schemas as plain `ZodObject` instances. (Background: 2026-03-05 Zod `.refine()` empty schema fix — 18 of 22 tools were affected.)
  1296	
  1297	**See also:** §63 for the general rule covering all outer-schema uses of `.refine()`, `.transform()`, and `.superRefine()`.
  1298	
  1299	---
  1300	
  1301	### 58. MCP SDK Injects `RequestHandlerExtra` — Handler Registration Must Use Wrapper Functions
  1302	
  1303	**Rule:** Every internal tool handler that has a second positional parameter (`_ledgerRoot?: string`) **must** be registered via an arrow-function wrapper, **not** passed directly as the handler. Additionally, each such handler **must** apply a defensive type guard before using `_ledgerRoot`.
  1304	
  1305	**Root cause:** The MCP SDK (v1.0.4+) calls every registered tool handler as:
  1306	```typescript
  1307	typedHandler(args, extra)   // extra is RequestHandlerExtra
  1308	```
  1309	If the handler has a second positional parameter (`_ledgerRoot?: string`), the `extra` object is captured by it. Because `extra` is truthy, `_ledgerRoot ?? projectPath` resolves to the `extra` object, causing downstream `path.join()` calls to throw:
  1310	```
  1311	TypeError: The "path" argument must be of type string. Received an instance of Object
  1312	```
  1313	
  1314	**Two-layer defence (belt-and-suspenders):**
  1315	
  1316	*Layer 1 — Registration wrapper (primary):*
  1317	```typescript
  1318	// ✅ CORRECT — extra never reaches the internal handler
  1319	server.registerTool('ledger_create_work_package', { ... }, (args) => createWorkPackage(args));
  1320	
  1321	// ❌ WRONG — extra leaks into _ledgerRoot
  1322	server.registerTool('ledger_create_work_package', { ... }, createWorkPackage as any);
  1323	```
  1324	
  1325	*Layer 2 — Defensive type guard inside the handler (secondary):*
  1326	```typescript
  1327	async function createWorkPackage(args: ..., _ledgerRoot?: string) {
  1328	  // ✅ Guard against the MCP SDK injecting a RequestHandlerExtra object
  1329	  const ledgerRoot = typeof _ledgerRoot === 'string' ? _ledgerRoot : undefined;
  1330	  // Use ledgerRoot throughout — never use _ledgerRoot directly after this line
  1331	}
  1332	```
  1333	
  1334	**Affected handlers (both layers applied as of 2026-03-01):**
  1335	- `createWorkPackage` — `src/tools/work-package.ts`
  1336	- `claimWorkPackage` — `src/tools/work-package.ts`
  1337	- `updateWorkPackageStatus` — `src/tools/work-package.ts`
  1338	- `resetReworkCount` — `src/tools/work-package.ts`
  1339	- `updateAcceptanceCriteria` — `src/tools/work-package.ts`
  1340	- `completeSynthesis` — `src/tools/project-lifecycle.ts`
  1341	
  1342	**Why single-argument handlers are unaffected:** Handlers with only one parameter (`initializeProject`, `getProjectStatus`, etc.) silently ignore any surplus arguments passed by the SDK — `extra` is discarded before it can cause harm.
  1343	
  1344	**Rationale:** A bug introduced when the SDK began passing `extra` went undetected because all unit tests call internal functions directly with an explicit string `_ledgerRoot`. The registration layer, where the SDK's extra injection occurs, had no test coverage. The two-layer defence ensures correctness both at the registration boundary and inside the function itself.
  1345	
  1346	---
  1347	
  1348	### 59. Acceptance Criteria Field-Name Verification
  1349	
  1350	**Rule:** Acceptance criteria text that references specific JSON field names, TypeScript parameter names, or object property names (e.g., `store`, `rootIndex`, `wpDetails`, `storageDir`) **must** be verified against the actual implementation source before the AC is committed to a work package. If the implementation uses a different name than what the AC states, the AC text must be updated to match.
  1351	
  1352	**Rationale:** Stale field-name references in ACs cause false-negative review outcomes. When a reviewer checks `wpDetails` against acceptance criteria but the implementation uses `allWpDetails`, the criterion is technically not met — yet neither the agent nor the QA reviewer notices. This constraint formalises the verification step that was retroactively identified in synthesis #4 of the Ledger Tool Simplification rework-1 cycle.
  1353	
  1354	**Anti-pattern:**
  1355	```
  1356	// AC text: "getNextActionsCollector receives `wpDetails` as a pre-loaded array"
  1357	// Implementation: loads wp details internally, no wpDetails parameter
  1358	// → AC text silently passes review because no one checks the parameter name
  1359	```
  1360	
  1361	**Correct pattern:**
  1362	```
  1363	// AC text uses the exact parameter/field name from the source:
  1364	// "getNextActionsCollector receives `rootIndex: RootIndex` and `store: LedgerStore`"
  1365	// Verified against src/tools/workflow-next-action.ts before committing
  1366	```
  1367	
  1368	---
  1369	
  1370	### 60. No Unused Locals (`noUnusedLocals`)
  1371	
  1372	**Rule:** `tsconfig.json` enables `"noUnusedLocals": true`. Every import, variable, parameter, and type alias that is declared must be consumed within its file. Dead imports and unused variables are compile errors — fix, never suppress.
  1373	
  1374	**Rationale:** Unused imports are structural noise left behind by refactors (e.g., when symbols move to a new module). They mislead agents and developers into thinking a dependency exists when it does not, and they obscure intent. The `noUnusedLocals` flag makes these errors hard build failures so they cannot accumulate silently.
  1375	
  1376	**Anti-pattern:**
  1377	```typescript
  1378	// ❌ WRONG — AGENT_PIPELINE_MAP moved to workflow-next-action-batch.ts but was
  1379	// left in the import list of workflow-next-action.ts after a file-split refactor.
  1380	import {
  1381	  PIPELINE_TYPES,
  1382	  AGENT_PIPELINE_MAP,   // ← never referenced in this file
  1383	  type PipelineType,
  1384	} from '../utils/pipeline-maps.js';
  1385	```
  1386	
  1387	**Correct pattern:**
  1388	```typescript
  1389	// ✅ CORRECT — only symbols actually used in this file are imported.
  1390	import {
  1391	  PIPELINE_TYPES,
  1392	  type PipelineType,
  1393	} from '../utils/pipeline-maps.js';
  1394	```
  1395	
  1396	**Forbidden patterns:**
  1397	- Adding `// @ts-ignore` or `// eslint-disable` to suppress unused-local errors.
  1398	- Importing a symbol "for re-export" without an explicit re-export statement.
  1399	- Leaving a symbol in an import group after moving its last consumer to another file.
  1400	

## Assistant

```json
{
  "id": "toolu_01RNfRhv1AN42RitRquhgaDf",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
    "offset": 1400,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01RNfRhv1AN42RitRquhgaDf`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
  "offset": 1400,
  "limit": 200
}
```

## Tool Result

  1401	---
  1402	
  1403	### 61. `assigned_to` Requires a Canonical AgentRole; `project_comments.agent` Does Not
  1404	
  1405	**Rule:** The `assigned_to` field on a work package (`WorkPackageSchema.assigned_to`) must be a value from the `AGENT_ROLES` constant (a validated `AgentRole` union). The `agent` field on a project-level comment (`ProjectCommentSchema.agent`) is typed as `z.string()` and is intentionally **not** constrained to `AGENT_ROLES`.
  1406	
  1407	**Rationale:** `assigned_to` drives workflow routing, gate checks, and pipeline agent-map lookups — it must be a machine-readable canonical role value. `project_comments.agent` is a human-readable audit identifier; it records who wrote the comment as a narrative label, not as a workflow actor, so free-form strings are appropriate.
  1408	
  1409	**Anti-pattern:**
  1410	```typescript
  1411	// ❌ WRONG — using a non-canonical value in the role-validated field
  1412	await claimWorkPackage({ ..., agent: "Developer Agent" });
  1413	// Zod rejects "Developer Agent" — not a member of AGENT_ROLES
  1414	```
  1415	
  1416	**Correct pattern:**
  1417	```typescript
  1418	// ✅ CORRECT — canonical AgentRole value required for assigned_to/agent in claim
  1419	await claimWorkPackage({ ..., agent: "Developer" });
  1420	
  1421	// ✅ ALSO CORRECT — free-text is acceptable in project_comments.agent
  1422	await addProjectComment({ ..., agent: "Developer Agent" });
  1423	// z.string() accepts arbitrary strings here; this is intentional
  1424	```
  1425	
  1426	**Forbidden patterns:**
  1427	- Using `"Developer Agent"` (or any multi-word variant) as the `agent` argument to `ledger_claim_work_package` or `ledger_start_pipeline`.
  1428	- Assuming `project_comments.agent` and `assigned_to` share the same validation rules — they do not.
  1429	- Hardcoding role strings anywhere other than constants. Use `AGENT_ROLES` entries or the `AgentRole` type for `assigned_to`-typed fields.
  1430	
  1431	**Reference:** `AGENT_ROLES` is derived from `shared/workflow-manifest.json` (`roles[].name`) and re-exported from `src/utils/constants.ts`. `ProjectCommentSchema` is in `src/schema/validators.ts`. See [tech-stack.md — Architectural Pattern 10](tech-stack.md#10-manifest-derived-constants) for the full list of manifest-derived constants.
  1432	
  1433	---
  1434	
  1435	### 62. `ledger_begin_work` IN_PROGRESS Guard Accepts Pipeline-Type Owners
  1436	
  1437	**Rule:** When `ledger_begin_work` is called on a work package that is already `IN_PROGRESS`, the call is allowed if **either** condition holds:
  1438	
  1439	1. **Idempotent re-entry:** `wp.assigned_to === args.agent_role` (the same agent is continuing their own work).
  1440	2. **Cross-agent handoff:** `PIPELINE_AGENT_MAP[args.type] === args.agent_role` (the caller is the legitimate pipeline-type owner per the workflow spec).
  1441	
  1442	If neither condition holds, the call is rejected.
  1443	
  1444	**Rationale (§9.1, §16.5):** The `assigned_to` field is a trailing bookkeeping field — a side-effect updated by the pipeline-start phase, not a security gate. Pipeline authorisation is defined by `PIPELINE_AGENT_MAP`. Using `assigned_to` as a hard gate would block every cross-agent handoff where `ledger_begin_work` is used instead of the two-step `ledger_claim_work_package + ledger_start_pipeline` sequence. This constraint restores consistency with `ledger_start_pipeline`, which enforces `PIPELINE_AGENT_MAP` only.
  1445	
  1446	**Contrast with `ledger_claim_work_package`:** Constraint 14 governs `ledger_claim_work_package`, which operates on `READY` WPs and does require an explicit `override: true` for cross-agent claims. The `READY → IN_PROGRESS` transition is a deliberate re-assignment; `ledger_begin_work` on an `IN_PROGRESS` WP is a pipeline-start handoff, not a RE-assignment.
  1447	
  1448	**Enforcement:** `isPipelineOwner` compound check in `beginWork()` in `src/tools/begin-work.ts`.
  1449	
  1450	**Error message (guard fires):**
  1451	```
  1452	Cannot begin work on WP-002: it is IN_PROGRESS and assigned to "Reviewer" but you are "Developer".
  1453	Only the assigned agent or the legitimate pipeline-type owner may start a pipeline on an IN_PROGRESS work package.
  1454	```
  1455	
  1456	---
  1457	
  1458	### 63. Do Not Use `.refine()`, `.transform()`, or `.superRefine()` on Outer Tool Schemas
  1459	
  1460	**Rule:** Never chain `.refine()`, `.transform()`, or `.superRefine()` on the outer `z.object({...})` schema passed as `inputSchema` to `server.registerTool()`. These methods convert a `ZodObject` into a `ZodEffects` wrapper, which the MCP SDK's JSON Schema converter cannot introspect — it emits `{ properties: {}, required: [] }` instead of the actual field list.
  1461	
  1462	**Reason:** The MCP `tools/list` response uses the JSON Schema to populate the tool definition shown to AI clients. An empty `properties` object means the client cannot see any parameters, so agents cannot pass arguments to the tool. This bug silently affects all callers, including VS Code Copilot agent mode.
  1463	
  1464	**Correct pattern:** Move cross-field validation inside the handler function (or a helper it calls, such as `resolveProjectPath()`):
  1465	
  1466	```typescript
  1467	// ✅ CORRECT — plain ZodObject; SDK emits correct properties
  1468	const MyToolSchema = z.object({
  1469	  project_path: z.string().optional(),
  1470	  cwd_path: z.string().optional(),
  1471	});
  1472	
  1473	async function myToolHandler(args: z.infer<typeof MyToolSchema>) {
  1474	  // Mutual exclusivity enforced at runtime by resolveProjectPath()
  1475	  const projectPath = await resolveProjectPath(args);
  1476	  // ...
  1477	}
  1478	```
  1479	
  1480	**Anti-pattern:**
  1481	
  1482	```typescript
  1483	// ❌ WRONG — .refine() converts ZodObject → ZodEffects
  1484	// SDK emits { properties: {}, required: [] } — agent cannot pass arguments
  1485	const MyToolSchema = z.object({
  1486	  project_path: z.string().optional(),
  1487	  cwd_path: z.string().optional(),
  1488	}).refine(mutuallyExclusivePaths, { message: MUTUAL_EXCLUSIVITY_PATH_MSG });
  1489	```
  1490	
  1491	**Exception:** Field-level `.refine()` applied to an individual field definition (e.g., `z.string().refine(...)`, `plan_file: z.string().refine(v => v === 'plan.md', ...)`) is safe — the outer `z.object()` remains a `ZodObject`.
  1492	
  1493	**Regression guard:** `tests/tools/schema-integrity.test.ts` converts all 22 registered tool schemas to JSON Schema and asserts non-empty `properties`. This test fails if a `.refine()` / `.transform()` / `.superRefine()` is re-added to any outer schema.
  1494	
  1495	**Background:** Fixed in plan `2026-03-05-zod-refine-empty-schema`. All 18 affected tools previously emitted empty JSON Schemas due to this pattern.
  1496	
  1497	---
  1498	
  1499	### 64. Mock `McpServer` Intercept Pattern for Tool Metadata Tests
  1500	
  1501	**Rule:** When writing tests that need to inspect tool metadata (input schema shape, parameter constraints, tool descriptions) without spinning up a real MCP server, use the mock `McpServer` intercept pattern: create a plain object with a `registerTool` method that captures schemas into a `Map`, cast it `as unknown as McpServer`, and call each tool module's `register()` function with it in `beforeAll`.
  1502	
  1503	**Rationale:** This pattern exercises the exact production registration path — same `register()` call, same `inputSchema` reference — without a network socket or real server lifecycle. It is safe with `beforeAll` because `register()` calls are synchronous.
  1504	
  1505	**Correct pattern:**
  1506	
  1507	```typescript
  1508	import { beforeAll, describe, it, expect } from 'vitest';
  1509	import { zodToJsonSchema } from 'zod-to-json-schema';
  1510	import type { z } from 'zod';
  1511	import type { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
  1512	import { register as registerPipeline } from '../../src/tools/pipeline.js';
  1513	
  1514	const capturedSchemas = new Map<string, z.ZodTypeAny>();
  1515	
  1516	const mockServer = {
  1517	  registerTool: (
  1518	    name: string,
  1519	    config: { description: string; inputSchema: z.ZodTypeAny },
  1520	    _handler: unknown
  1521	  ) => {
  1522	    capturedSchemas.set(name, config.inputSchema);
  1523	  },
  1524	} as unknown as McpServer;
  1525	
  1526	beforeAll(() => {
  1527	  registerPipeline(mockServer);
  1528	});
  1529	
  1530	describe('pipeline schemas', () => {
  1531	  it('ledger_start_pipeline has non-empty properties', () => {
  1532	    const schema = capturedSchemas.get('ledger_start_pipeline')!;
  1533	    const json = zodToJsonSchema(schema) as { properties?: object };
  1534	    expect(Object.keys(json.properties ?? {})).not.toHaveLength(0);
  1535	  });
  1536	});
  1537	```
  1538	
  1539	**When to use:** Any test that needs to verify tool schema shape, description content, or parameter constraints without full server lifecycle overhead. See `tests/tools/schema-integrity.test.ts` for the canonical usage.
  1540	
  1541	**Note on `zod-to-json-schema`:** This package is currently a transitive dependency (via `@modelcontextprotocol/sdk`) and is not declared as an explicit `devDependency` in `mcp-server/package.json`. Tests relying on it work today, but if the SDK drops the transitive dep in a future update, imports will fail without a clear error. Prefer adding it explicitly when introducing new test files that import it directly.
  1542	
  1543	---
  1544	
  1545	### 65. All Six Pipeline Stages Are PM-Composable — No Mandatory/Optional Distinction
  1546	
  1547	**Rule:** All six pipeline stages (`implementation`, `qa`, `security-audit`, `code-review`, `release-engineering`, `documentation`) are equally composable by the Project Manager. There is no inherent "mandatory" or "optional" designation for any stage. The PM selects any valid subsequence of `CANONICAL_PIPELINE_ORDERING` per work package via the `active_pipeline_stages` field.
  1548	
  1549	**Default:** When `active_pipeline_stages` is omitted, `DEFAULT_PIPELINE_STAGES` (`['implementation', 'qa', 'code-review', 'documentation']`) is used for backward compatibility.
  1550	
  1551	**Rationale:** The former `MANDATORY_PIPELINE_TYPES` and `OPTIONAL_PIPELINE_TYPES` constants are retired. The PM-composable model enables custom workflows (e.g., skipping QA for documentation-only WPs, adding a security audit before code review) without encoding assumptions into the server.
  1552	
  1553	**Extension:** The `CANONICAL_PIPELINE_ORDERING` constant (`['implementation', 'qa', 'security-audit', 'code-review', 'release-engineering', 'documentation']`) defines the only valid execution order — stages may be omitted but not reordered. `resolvePrerequisite`, `resolveNextAgent`, and `resolveFailAgent` derive routing dynamically from the per-WP `active_pipeline_stages` array.
  1554	
  1555	**Enforcement:** `ledger_create_work_package` validates the `active_pipeline_stages` input (see Constraint 66). Pipeline start and completion routing use the dynamic resolve functions, not static maps.
  1556	
  1557	> Full specification: [Workflow Specification §4.2, §9b](../workflow-specification/data-model.md#42-pipeline-stage-constants).
  1558	
  1559	---
  1560	
  1561	### 66. `active_pipeline_stages` Validation: Hard Guardrails (Reject) and Soft Guardrails (Warn)
  1562	
  1563	**Rule:** When `ledger_create_work_package` receives an `active_pipeline_stages` value, it validates the array before persisting the work package.
  1564	
  1565	**Hard guardrails (reject with error — creation is aborted):**
  1566	- Empty array (`[]`)
  1567	- Entries that are not valid `PIPELINE_TYPES` values
  1568	- Duplicate entries
  1569	- Entries that are not a subsequence of `CANONICAL_PIPELINE_ORDERING` (relative ordering must be preserved; gaps are allowed)
  1570	
  1571	**Soft guardrails (warning appended to the success response message — creation is NOT aborted):**
  1572	- `implementation` present without `qa` (unusual composition)
  1573	- Single-stage chain (degenerate case)
  1574	
  1575	**Omitted field:** When `active_pipeline_stages` is omitted (the common case for standard 4-stage workflows), validation is bypassed entirely. The field is absent on the WP detail and dynamic resolve functions substitute `DEFAULT_PIPELINE_STAGES` at runtime.
  1576	
  1577	**Enforcement:** `validateActiveStages()` helper called inside `createWorkPackage()` in `src/tools/work-package.ts`. Hard rejection throws before the WP is written; soft warning is appended to the response string after the WP is written.
  1578	
  1579	> Full specification: [Workflow Specification §9b.2](../workflow-specification/operations.md#9b2-active-pipeline-stages-validation).
  1580	
  1581	---
  1582	
  1583	### 67. Artifact Declaration Expectation — Soft Warning on Empty `files_modified`
  1584	
  1585	**Rule:** When `ledger_complete_pipeline` is called with `status: 'PASS'` and the `artifacts.files_modified` array is either absent or empty, the server appends a soft-warning note **only if the pipeline type is in `ARTIFACT_EXPECTED_PIPELINE_TYPES`** (`implementation`, `code-review`, `release-engineering`, `documentation`). Verification-only pipeline types (`qa`, `security-audit`) are exempt because those agents verify but do not modify files. `code-review` is included because the Reviewer may apply Fix-Forward edits. This is a non-blocking warning — the pipeline completion is still accepted.
  1586	
  1587	**Rationale:** Agents often forget to populate `files_modified`, reducing the value of the pipeline record for auditing and documentation. The soft warning creates a visible signal in the response without blocking legitimate zero-file-change completions. Verification-only agents are exempt to avoid noisy false-positive warnings.
  1588	
  1589	**Exception:** The warning is only emitted on `PASS` completions — `FAIL` pipelines are not expected to declare modified files.
  1590	
  1591	**Enforcement:** Soft check in `completePipeline()` in `src/tools/pipeline.ts` (step 3b), gated by `ARTIFACT_EXPECTED_PIPELINE_TYPES` from `src/utils/pipeline-maps.ts`. Does not reject the call; appended as a text note in the response body only.
  1592	
  1593	---
  1594	
  1595	### 68. Zod `.describe()` Annotations for Pipeline Type Must Use `describePipelineTypes()`
  1596	
  1597	**Rule:** All Zod `.describe()` strings that enumerate pipeline type values MUST be generated by calling `describePipelineTypes(prefix)` from `src/utils/pipeline-maps.ts`. Hardcoding a pipeline type list inline in a `.describe()` string is forbidden.
  1598	
  1599	**Rationale:** `PIPELINE_TYPES` is the single source of truth for the canonical pipeline type list. Hardcoded `.describe()` strings drift silently when a new pipeline type is added — as demonstrated when `observations.ts` still listed only 4 types after `security-audit` and `release-engineering` were introduced. `describePipelineTypes()` derives the annotation from `PIPELINE_TYPES` at schema definition time, so any future addition to `PIPELINE_TYPES` propagates automatically to all MCP JSON Schema annotations.
  1600	

## Assistant

Now I have a thorough understanding of both files. Let me also check the section around Constraint 57 more carefully — the one about mutual exclusivity — since that's what needs to change:

The key section in `constraints.md` is **Constraint 57** (lines 1264–1297), titled "Mutual Exclusivity of `project_path` and `cwd_path` — Runtime Guard via `resolveProjectPath()`". This currently states mutual exclusivity is enforced (throwing an error when both are provided). According to WP-002, this needs to be updated to describe the precedence rule instead.

The key text in `help-content.ts` that needs changing is line 13:
```
**Most tools accept either \`cwd_path\` or \`project_path\` — not both.**
```

This needs to become the new paragraph specified in the WP:
```
**Most tools accept `project_path` and/or `cwd_path`.** If you have `project_path` (the plan folder), use it — it's the fastest path. If you only know your workspace directory, pass `cwd_path` and the server auto-detects the active project. If you pass both, `project_path` takes precedence and `cwd_path` is ignored.
```

Now let me make both changes:

```json
{
  "id": "toolu_01UkrXa6Box2JNgwg6fKmCGd",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
    "old_string": "**Most tools accept either \\`cwd_path\\` or \\`project_path\\` \u2014 not both.** Use \\`cwd_path\\` (your workspace root) as the preferred option; the server auto-detects the active project. Only provide \\`project_path\\` if you already have it from a previous tool response. The one exception is \\`ledger_initialize_project\\`, which requires \\`project_path\\` (the plan folder is being created and cannot be detected yet).",
    "new_string": "**Most tools accept \\`project_path\\` and/or \\`cwd_path\\`.** If you have \\`project_path\\` (the plan folder), use it \u2014 it's the fastest path. If you only know your workspace directory, pass \\`cwd_path\\` and the server auto-detects the active project. If you pass both, \\`project_path\\` takes precedence and \\`cwd_path\\` is ignored. The one exception is \\`ledger_initialize_project\\`, which requires \\`project_path\\` (the plan folder is being created and cannot be detected yet)."
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_01UkrXa6Box2JNgwg6fKmCGd`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
  "old_string": "**Most tools accept either \\`cwd_path\\` or \\`project_path\\` \u2014 not both.** Use \\`cwd_path\\` (your workspace root) as the preferred option; the server auto-detects the active project. Only provide \\`project_path\\` if you already have it from a previous tool response. The one exception is \\`ledger_initialize_project\\`, which requires \\`project_path\\` (the plan folder is being created and cannot be detected yet).",
  "new_string": "**Most tools accept \\`project_path\\` and/or \\`cwd_path\\`.** If you have \\`project_path\\` (the plan folder), use it \u2014 it's the fastest path. If you only know your workspace directory, pass \\`cwd_path\\` and the server auto-detects the active project. If you pass both, \\`project_path\\` takes precedence and \\`cwd_path\\` is ignored. The one exception is \\`ledger_initialize_project\\`, which requires \\`project_path\\` (the plan folder is being created and cannot be detected yet)."
}
```

## Tool Result

Successfully replaced 1 instance(s) of the string in '/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts'

## Assistant

Now update `constraints.md` — Constraint 57. I need to replace the section that describes mutual exclusivity enforcement as an error with one that describes the `project_path`-wins precedence rule:

```json
{
  "id": "toolu_01NkBPu4Fr4vnviRXQ5M96fQ",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
    "old_string": "### 57. Mutual Exclusivity of `project_path` and `cwd_path` \u2014 Runtime Guard via `resolveProjectPath()`\n\n**Rule:** Mutual exclusivity of `project_path` and `cwd_path` is enforced at runtime by `resolveProjectPath()`, **not** by a Zod `.refine()` on the outer schema. Do **not** add `.refine()`, `.transform()`, or `.superRefine()` to the outer `z.object()` of any tool schema.\n\n**Enforcement:**\n- `resolveProjectPath()` (`src/utils/path-validator.ts`) throws `Error(MUTUAL_EXCLUSIVITY_PATH_MSG)` at the top of its body when both `project_path` and `cwd_path` are truthy. Every tool handler that accepts both optional path fields calls `resolveProjectPath()` \u2014 the guard fires unconditionally.\n- The predicate `mutuallyExclusivePaths` and the constant `MUTUAL_EXCLUSIVITY_PATH_MSG` remain exported from `src/utils/path-validator.ts` for backward compatibility and test coverage. They are **not used in production tool files**.\n- Schemas that only contain `project_path` (mandatory) or only `cwd_path` \u2014 but not both as optional fields \u2014 are exempt from this consideration. `DetectProjectSchema`, `InitializeProjectSchema`, and `ListProjectsSchema` fall into this category.\n\n**Anti-pattern:**\n```typescript\n// \u274c WRONG \u2014 .refine() converts ZodObject \u2192 ZodEffects. The MCP SDK cannot extract properties\n// from ZodEffects, resulting in empty { properties: {}, required: [] } in tools/list responses.\nconst GetWorkPackageSchema = z.object({\n  project_path: z.string().optional().describe('\u2026'),\n  cwd_path:     z.string().optional().describe('\u2026'),\n  work_package_id: z.string().regex(/^WP-\\d{3,}$/),\n})\n  .refine(mutuallyExclusivePaths, { message: MUTUAL_EXCLUSIVITY_PATH_MSG });\n```\n\n**Correct pattern:**\n```typescript\n// \u2705 CORRECT \u2014 plain ZodObject; mutual exclusivity is enforced inside resolveProjectPath()\nconst GetWorkPackageSchema = z.object({\n  project_path: z.string().optional().describe('\u2026'),\n  cwd_path:     z.string().optional().describe('\u2026'),\n  work_package_id: z.string().regex(/^WP-\\d{3,}$/),\n});\n```\n\n**Rationale:** `.refine()` (and `.transform()`, `.superRefine()`) on the outer `z.object()` converts it from `ZodObject` to `ZodEffects`. The MCP SDK's `zodToJsonSchema` cannot extract properties from `ZodEffects` \u2014 every affected tool emits empty `{ properties: {}, required: [] }` in the `tools/list` response, preventing AI agents from passing arguments. Centralising the check in `resolveProjectPath()` keeps all tool schemas as plain `ZodObject` instances. (Background: 2026-03-05 Zod `.refine()` empty schema fix \u2014 18 of 22 tools were affected.)\n\n**See also:** \u00a763 for the general rule covering all outer-schema uses of `.refine()`, `.transform()`, and `.superRefine()`.",
    "new_string": "### 57. `project_path` Takes Precedence Over `cwd_path` When Both Are Provided\n\n**Rule:** When a caller supplies both `project_path` and `cwd_path`, `resolveProjectPath()` uses `project_path` and silently ignores `cwd_path`. Supplying both parameters is **not** an error. Do **not** add `.refine()`, `.transform()`, or `.superRefine()` to the outer `z.object()` of any tool schema to enforce exclusivity.\n\n**Precedence rule (in `resolveProjectPath()`, `src/utils/path-validator.ts`):**\n1. If `project_path` is provided (truthy) \u2192 use it directly; `cwd_path` is ignored.\n2. If only `cwd_path` is provided \u2192 auto-detect the active project from the workspace root.\n3. If neither is provided \u2192 throw a missing-path error.\n\n**Guidance for callers:**\n- If you already have `project_path` (the plan folder path from a prior tool response), pass it \u2014 it is the fastest path with no auto-detection overhead.\n- If you only know your workspace root, pass `cwd_path` and let the server detect the project.\n- If you pass both, `project_path` wins; `cwd_path` is a no-op in that call.\n\n**Enforcement:**\n- `resolveProjectPath()` (`src/utils/path-validator.ts`) applies the precedence rule at the top of its body. Every tool handler that accepts both optional path fields calls `resolveProjectPath()`.\n- The predicate `mutuallyExclusivePaths` and the constant `MUTUAL_EXCLUSIVITY_PATH_MSG` remain exported from `src/utils/path-validator.ts` for backward compatibility and test coverage. They are **not used in production tool files**.\n- Schemas that only contain `project_path` (mandatory) or only `cwd_path` \u2014 but not both as optional fields \u2014 are exempt from this consideration. `DetectProjectSchema`, `InitializeProjectSchema`, and `ListProjectsSchema` fall into this category.\n\n**Anti-pattern:**\n```typescript\n// \u274c WRONG \u2014 .refine() converts ZodObject \u2192 ZodEffects. The MCP SDK cannot extract properties\n// from ZodEffects, resulting in empty { properties: {}, required: [] } in tools/list responses.\nconst GetWorkPackageSchema = z.object({\n  project_path: z.string().optional().describe('\u2026'),\n  cwd_path:     z.string().optional().describe('\u2026'),\n  work_package_id: z.string().regex(/^WP-\\d{3,}$/),\n})\n  .refine(mutuallyExclusivePaths, { message: MUTUAL_EXCLUSIVITY_PATH_MSG });\n```\n\n**Correct pattern:**\n```typescript\n// \u2705 CORRECT \u2014 plain ZodObject; project_path-wins precedence is enforced inside resolveProjectPath()\nconst GetWorkPackageSchema = z.object({\n  project_path: z.string().optional().describe('\u2026'),\n  cwd_path:     z.string().optional().describe('\u2026'),\n  work_package_id: z.string().regex(/^WP-\\d{3,}$/),\n});\n```\n\n**Rationale:** `.refine()` (and `.transform()`, `.superRefine()`) on the outer `z.object()` converts it from `ZodObject` to `ZodEffects`. The MCP SDK's `zodToJsonSchema` cannot extract properties from `ZodEffects` \u2014 every affected tool emits empty `{ properties: {}, required: [] }` in the `tools/list` response, preventing AI agents from passing arguments. Centralising the precedence logic in `resolveProjectPath()` keeps all tool schemas as plain `ZodObject` instances and eliminates spurious errors when callers pass both parameters. (Background: 2026-03-05 Zod `.refine()` empty schema fix \u2014 18 of 22 tools were affected.)\n\n**See also:** \u00a763 for the general rule covering all outer-schema uses of `.refine()`, `.transform()`, and `.superRefine()`."
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_01NkBPu4Fr4vnviRXQ5M96fQ`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
  "old_string": "### 57. Mutual Exclusivity of `project_path` and `cwd_path` \u2014 Runtime Guard via `resolveProjectPath()`\n\n**Rule:** Mutual exclusivity of `project_path` and `cwd_path` is enforced at runtime by `resolveProjectPath()`, **not** by a Zod `.refine()` on the outer schema. Do **not** add `.refine()`, `.transform()`, or `.superRefine()` to the outer `z.object()` of any tool schema.\n\n**Enforcement:**\n- `resolveProjectPath()` (`src/utils/path-validator.ts`) throws `Error(MUTUAL_EXCLUSIVITY_PATH_MSG)` at the top of its body when both `project_path` and `cwd_path` are truthy. Every tool handler that accepts both optional path fields calls `resolveProjectPath()` \u2014 the guard fires unconditionally.\n- The predicate `mutuallyExclusivePaths` and the constant `MUTUAL_EXCLUSIVITY_PATH_MSG` remain exported from `src/utils/path-validator.ts` for backward compatibility and test coverage. They are **not used in production tool files**.\n- Schemas that only contain `project_path` (mandatory) or only `cwd_path` \u2014 but not both as optional fields \u2014 are exempt from this consideration. `DetectProjectSchema`, `InitializeProjectSchema`, and `ListProjectsSchema` fall into this category.\n\n**Anti-pattern:**\n```typescript\n// \u274c WRONG \u2014 .refine() converts ZodObject \u2192 ZodEffects. The MCP SDK cannot extract properties\n// from ZodEffects, resulting in empty { properties: {}, required: [] } in tools/list responses.\nconst GetWorkPackageSchema = z.object({\n  project_path: z.string().optional().describe('\u2026'),\n  cwd_path:     z.string().optional().describe('\u2026'),\n  work_package_id: z.string().regex(/^WP-\\d{3,}$/),\n})\n  .refine(mutuallyExclusivePaths, { message: MUTUAL_EXCLUSIVITY_PATH_MSG });\n```\n\n**Correct pattern:**\n```typescript\n// \u2705 CORRECT \u2014 plain ZodObject; mutual exclusivity is enforced inside resolveProjectPath()\nconst GetWorkPackageSchema = z.object({\n  project_path: z.string().optional().describe('\u2026'),\n  cwd_path:     z.string().optional().describe('\u2026'),\n  work_package_id: z.string().regex(/^WP-\\d{3,}$/),\n});\n```\n\n**Rationale:** `.refine()` (and `.transform()`, `.superRefine()`) on the outer `z.object()` converts it from `ZodObject` to `ZodEffects`. The MCP SDK's `zodToJsonSchema` cannot extract properties from `ZodEffects` \u2014 every affected tool emits empty `{ properties: {}, required: [] }` in the `tools/list` response, preventing AI agents from passing arguments. Centralising the check in `resolveProjectPath()` keeps all tool schemas as plain `ZodObject` instances. (Background: 2026-03-05 Zod `.refine()` empty schema fix \u2014 18 of 22 tools were affected.)\n\n**See also:** \u00a763 for the general rule covering all outer-schema uses of `.refine()`, `.transform()`, and `.superRefine()`.",
  "new_string": "### 57. `project_path` Takes Precedence Over `cwd_path` When Both Are Provided\n\n**Rule:** When a caller supplies both `project_path` and `cwd_path`, `resolveProjectPath()` uses `project_path` and silently ignores `cwd_path`. Supplying both parameters is **not** an error. Do **not** add `.refine()`, `.transform()`, or `.superRefine()` to the outer `z.object()` of any tool schema to enforce exclusivity.\n\n**Precedence rule (in `resolveProjectPath()`, `src/utils/path-validator.ts`):**\n1. If `project_path` is provided (truthy) \u2192 use it directly; `cwd_path` is ignored.\n2. If only `cwd_path` is provided \u2192 auto-detect the active project from the workspace root.\n3. If neither is provided \u2192 throw a missing-path error.\n\n**Guidance for callers:**\n- If you already have `project_path` (the plan folder path from a prior tool response), pass it \u2014 it is the fastest path with no auto-detection overhead.\n- If you only know your workspace root, pass `cwd_path` and let the server detect the project.\n- If you pass both, `project_path` wins; `cwd_path` is a no-op in that call.\n\n**Enforcement:**\n- `resolveProjectPath()` (`src/utils/path-validator.ts`) applies the precedence rule at the top of its body. Every tool handler that accepts both optional path fields calls `resolveProjectPath()`.\n- The predicate `mutuallyExclusivePaths` and the constant `MUTUAL_EXCLUSIVITY_PATH_MSG` remain exported from `src/utils/path-validator.ts` for backward compatibility and test coverage. They are **not used in production tool files**.\n- Schemas that only contain `project_path` (mandatory) or only `cwd_path` \u2014 but not both as optional fields \u2014 are exempt from this consideration. `DetectProjectSchema`, `InitializeProjectSchema`, and `ListProjectsSchema` fall into this category.\n\n**Anti-pattern:**\n```typescript\n// \u274c WRONG \u2014 .refine() converts ZodObject \u2192 ZodEffects. The MCP SDK cannot extract properties\n// from ZodEffects, resulting in empty { properties: {}, required: [] } in tools/list responses.\nconst GetWorkPackageSchema = z.object({\n  project_path: z.string().optional().describe('\u2026'),\n  cwd_path:     z.string().optional().describe('\u2026'),\n  work_package_id: z.string().regex(/^WP-\\d{3,}$/),\n})\n  .refine(mutuallyExclusivePaths, { message: MUTUAL_EXCLUSIVITY_PATH_MSG });\n```\n\n**Correct pattern:**\n```typescript\n// \u2705 CORRECT \u2014 plain ZodObject; project_path-wins precedence is enforced inside resolveProjectPath()\nconst GetWorkPackageSchema = z.object({\n  project_path: z.string().optional().describe('\u2026'),\n  cwd_path:     z.string().optional().describe('\u2026'),\n  work_package_id: z.string().regex(/^WP-\\d{3,}$/),\n});\n```\n\n**Rationale:** `.refine()` (and `.transform()`, `.superRefine()`) on the outer `z.object()` converts it from `ZodObject` to `ZodEffects`. The MCP SDK's `zodToJsonSchema` cannot extract properties from `ZodEffects` \u2014 every affected tool emits empty `{ properties: {}, required: [] }` in the `tools/list` response, preventing AI agents from passing arguments. Centralising the precedence logic in `resolveProjectPath()` keeps all tool schemas as plain `ZodObject` instances and eliminates spurious errors when callers pass both parameters. (Background: 2026-03-05 Zod `.refine()` empty schema fix \u2014 18 of 22 tools were affected.)\n\n**See also:** \u00a763 for the general rule covering all outer-schema uses of `.refine()`, `.transform()`, and `.superRefine()`."
}
```

## Tool Result

Successfully replaced 1 instance(s) of the string in '/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md'

## Assistant

Now let me verify both changes look correct:

```json
{
  "id": "toolu_01MX3Syy73MGpD4c4JQcp4V7",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
    "limit": 20
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01MX3Syy73MGpD4c4JQcp4V7`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
  "limit": 20
}
```

## Tool Result

     1	/**
     2	 * Static documentation strings for all Project Ledger MCP tools.
     3	 * Exported as TOOL_HELP and consumed by help.ts.
     4	 */
     5	import { PLAN_ARCHIVE_FILENAME, SYNTHESIS_ARCHIVE_FILENAME } from '../utils/constants.js';
     6	
     7	export const TOOL_HELP: Record<string, string> = {
     8	  overview: `
     9	# Project Ledger MCP — Tool Reference
    10	
    11	## Path Parameters
    12	
    13	**Most tools accept \`project_path\` and/or \`cwd_path\`.** If you have \`project_path\` (the plan folder), use it — it's the fastest path. If you only know your workspace directory, pass \`cwd_path\` and the server auto-detects the active project. If you pass both, \`project_path\` takes precedence and \`cwd_path\` is ignored. The one exception is \`ledger_initialize_project\`, which requires \`project_path\` (the plan folder is being created and cannot be detected yet).
    14	
    15	## All Available Tools
    16	
    17	| Tool | Required Params | Purpose |
    18	|------|----------------|---------|
    19	| ledger_get_project_status | cwd_path or project_path | Read project overview |
    20	| ledger_initialize_project | project_path, plan_file | Create new project ledger |

## Assistant

```json
{
  "id": "toolu_01Pfd1KnYfLy3gp2XsvpPEuN",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
    "offset": 1263,
    "limit": 60
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01Pfd1KnYfLy3gp2XsvpPEuN`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
  "offset": 1263,
  "limit": 60
}
```

## Tool Result

  1264	### 57. `project_path` Takes Precedence Over `cwd_path` When Both Are Provided
  1265	
  1266	**Rule:** When a caller supplies both `project_path` and `cwd_path`, `resolveProjectPath()` uses `project_path` and silently ignores `cwd_path`. Supplying both parameters is **not** an error. Do **not** add `.refine()`, `.transform()`, or `.superRefine()` to the outer `z.object()` of any tool schema to enforce exclusivity.
  1267	
  1268	**Precedence rule (in `resolveProjectPath()`, `src/utils/path-validator.ts`):**
  1269	1. If `project_path` is provided (truthy) → use it directly; `cwd_path` is ignored.
  1270	2. If only `cwd_path` is provided → auto-detect the active project from the workspace root.
  1271	3. If neither is provided → throw a missing-path error.
  1272	
  1273	**Guidance for callers:**
  1274	- If you already have `project_path` (the plan folder path from a prior tool response), pass it — it is the fastest path with no auto-detection overhead.
  1275	- If you only know your workspace root, pass `cwd_path` and let the server detect the project.
  1276	- If you pass both, `project_path` wins; `cwd_path` is a no-op in that call.
  1277	
  1278	**Enforcement:**
  1279	- `resolveProjectPath()` (`src/utils/path-validator.ts`) applies the precedence rule at the top of its body. Every tool handler that accepts both optional path fields calls `resolveProjectPath()`.
  1280	- The predicate `mutuallyExclusivePaths` and the constant `MUTUAL_EXCLUSIVITY_PATH_MSG` remain exported from `src/utils/path-validator.ts` for backward compatibility and test coverage. They are **not used in production tool files**.
  1281	- Schemas that only contain `project_path` (mandatory) or only `cwd_path` — but not both as optional fields — are exempt from this consideration. `DetectProjectSchema`, `InitializeProjectSchema`, and `ListProjectsSchema` fall into this category.
  1282	
  1283	**Anti-pattern:**
  1284	```typescript
  1285	// ❌ WRONG — .refine() converts ZodObject → ZodEffects. The MCP SDK cannot extract properties
  1286	// from ZodEffects, resulting in empty { properties: {}, required: [] } in tools/list responses.
  1287	const GetWorkPackageSchema = z.object({
  1288	  project_path: z.string().optional().describe('…'),
  1289	  cwd_path:     z.string().optional().describe('…'),
  1290	  work_package_id: z.string().regex(/^WP-\d{3,}$/),
  1291	})
  1292	  .refine(mutuallyExclusivePaths, { message: MUTUAL_EXCLUSIVITY_PATH_MSG });
  1293	```
  1294	
  1295	**Correct pattern:**
  1296	```typescript
  1297	// ✅ CORRECT — plain ZodObject; project_path-wins precedence is enforced inside resolveProjectPath()
  1298	const GetWorkPackageSchema = z.object({
  1299	  project_path: z.string().optional().describe('…'),
  1300	  cwd_path:     z.string().optional().describe('…'),
  1301	  work_package_id: z.string().regex(/^WP-\d{3,}$/),
  1302	});
  1303	```
  1304	
  1305	**Rationale:** `.refine()` (and `.transform()`, `.superRefine()`) on the outer `z.object()` converts it from `ZodObject` to `ZodEffects`. The MCP SDK's `zodToJsonSchema` cannot extract properties from `ZodEffects` — every affected tool emits empty `{ properties: {}, required: [] }` in the `tools/list` response, preventing AI agents from passing arguments. Centralising the precedence logic in `resolveProjectPath()` keeps all tool schemas as plain `ZodObject` instances and eliminates spurious errors when callers pass both parameters. (Background: 2026-03-05 Zod `.refine()` empty schema fix — 18 of 22 tools were affected.)
  1306	
  1307	**See also:** §63 for the general rule covering all outer-schema uses of `.refine()`, `.transform()`, and `.superRefine()`.
  1308	
  1309	---
  1310	
  1311	### 58. MCP SDK Injects `RequestHandlerExtra` — Handler Registration Must Use Wrapper Functions
  1312	
  1313	**Rule:** Every internal tool handler that has a second positional parameter (`_ledgerRoot?: string`) **must** be registered via an arrow-function wrapper, **not** passed directly as the handler. Additionally, each such handler **must** apply a defensive type guard before using `_ledgerRoot`.
  1314	
  1315	**Root cause:** The MCP SDK (v1.0.4+) calls every registered tool handler as:
  1316	```typescript
  1317	typedHandler(args, extra)   // extra is RequestHandlerExtra
  1318	```
  1319	If the handler has a second positional parameter (`_ledgerRoot?: string`), the `extra` object is captured by it. Because `extra` is truthy, `_ledgerRoot ?? projectPath` resolves to the `extra` object, causing downstream `path.join()` calls to throw:
  1320	```
  1321	TypeError: The "path" argument must be of type string. Received an instance of Object
  1322	```
  1323	

## Assistant

Both changes look correct. Now let me also check for any other instances in `help-content.ts` that might describe them as mutually exclusive:

```json
{
  "id": "toolu_01QCeHDL8W5iWCyfSmfMtD9b",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "not both",
    "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
    "output_mode": "content"
  },
  "name": "grep",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_016kpVDhetX1SwxnSzZKthup",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "mutually exclusive",
    "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
    "output_mode": "content"
  },
  "name": "grep",
  "type": "tool_use"
}
```

**Tool call:** `grep` (id: `toolu_01QCeHDL8W5iWCyfSmfMtD9b`)

```json
{
  "pattern": "not both",
  "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
  "output_mode": "content"
}
```

**Tool call:** `grep` (id: `toolu_016kpVDhetX1SwxnSzZKthup`)

```json
{
  "pattern": "mutually exclusive",
  "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
  "output_mode": "content"
}
```

## Tool Result

/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts:
  111: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  112: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  161: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  162: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  180: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  181: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  202: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  203: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  244: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  245: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  292: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  293: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  326: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  327: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  373: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  374: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  400: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  401: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  444: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  445: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  468: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  469: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  491: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  492: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  518: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  519: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  552: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  553: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  634: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  635: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
  701: - **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
  702: - **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*

## Tool Result

No matches found

## Assistant

There are many instances of the "not both" language throughout the file — these are in per-tool parameter descriptions. These all describe `project_path` and `cwd_path` as mutually exclusive, which conflicts with the new precedence rule. I need to update all of these too.

Let me read one of the sections to understand the pattern:

```json
{
  "id": "toolu_018mP83crDRhhs9K9uEBCL3a",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
    "offset": 105,
    "limit": 30
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_018mP83crDRhhs9K9uEBCL3a`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
  "offset": 105,
  "limit": 30
}
```

## Tool Result

   106	# ledger_get_project_status
   107	
   108	Read the project overview from the root index.
   109	
   110	## Required Parameters
   111	- **cwd_path** (string): Workspace root (preferred) — auto-detects the active project. *(Provide this OR project_path — not both.)*
   112	- **project_path** (string): Plan folder path — use only if already known. *(Provide this OR cwd_path — not both.)*
   113	
   114	## Example
   115	\`\`\`json
   116	{
   117	  "project_path": "f:\\\\project\\\\docs\\\\agents\\\\plans\\\\2026-02-16-feature"
   118	}
   119	\`\`\`
   120	
   121	## Returns
   122	The full root index including work package summaries, counters, and project status.
   123	`,
   124	
   125	  ledger_initialize_project: `
   126	# ledger_initialize_project
   127	
   128	Create a new project ledger. Call this once at project start.
   129	
   130	This also creates a \`.meta.json\` entry in the centralized ledger so the project
   131	is immediately discoverable via \`ledger_list_projects\`.
   132	
   133	The \`plan_file\` document is automatically archived into the ledger storage directory
   134	at initialization, making it retrievable via the GUI plan endpoint. If the file does
   135	not yet exist, it is silently skipped and reported in \`archive_skipped\`.

## Assistant

These repeated instances describe the same pattern. The new behavior is: both can be provided, but `project_path` takes precedence. The updated description should say something like: `*(project_path takes precedence if both are provided)*`.

Let me do a bulk replacement on the `cwd_path` lines and the `project_path` lines:

```json
{
  "id": "toolu_01QaA5enZncfqiPpHChZYVLR",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
    "old_string": "- **cwd_path** (string): Workspace root (preferred) \u2014 auto-detects the active project. *(Provide this OR project_path \u2014 not both.)*\n- **project_path** (string): Plan folder path \u2014 use only if already known. *(Provide this OR cwd_path \u2014 not both.)*",
    "new_string": "- **cwd_path** (string): Workspace root \u2014 auto-detects the active project. Pass this if you don't have project_path yet.\n- **project_path** (string): Plan folder path \u2014 use if already known; takes precedence over cwd_path if both are provided.",
    "replace_all": true
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_01QaA5enZncfqiPpHChZYVLR`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
  "old_string": "- **cwd_path** (string): Workspace root (preferred) \u2014 auto-detects the active project. *(Provide this OR project_path \u2014 not both.)*\n- **project_path** (string): Plan folder path \u2014 use only if already known. *(Provide this OR cwd_path \u2014 not both.)*",
  "new_string": "- **cwd_path** (string): Workspace root \u2014 auto-detects the active project. Pass this if you don't have project_path yet.\n- **project_path** (string): Plan folder path \u2014 use if already known; takes precedence over cwd_path if both are provided.",
  "replace_all": true
}
```

## Tool Result

Successfully replaced 16 instance(s) of the string in '/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts'

## Assistant

Now let me verify by checking the full file for any remaining "not both" references:

```json
{
  "id": "toolu_01V3mn48CxVDfRkKLobPrVcj",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "not both",
    "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
    "output_mode": "content"
  },
  "name": "grep",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01C6KRHDPfvvJ1hN77aivHGB",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "Provide this OR",
    "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
    "output_mode": "content"
  },
  "name": "grep",
  "type": "tool_use"
}
```

**Tool call:** `grep` (id: `toolu_01V3mn48CxVDfRkKLobPrVcj`)

```json
{
  "pattern": "not both",
  "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
  "output_mode": "content"
}
```

**Tool call:** `grep` (id: `toolu_01C6KRHDPfvvJ1hN77aivHGB`)

```json
{
  "pattern": "Provide this OR",
  "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
  "output_mode": "content"
}
```

## Tool Result

No matches found

## Tool Result

No matches found

## Assistant

No remaining "not both" language. Now let me also check `constraints.md` for any residual statement that supplying both is an error (we updated constraint 57, but let me check the rest of the file for any other references):

```json
{
  "id": "toolu_01WTMpPvmysRk6ki81NaCGZS",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "not both",
    "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
    "output_mode": "content"
  },
  "name": "grep",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01F3e8gbT1vowcMEeiaj9AL2",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "throws.*MUTUAL_EXCLUSIVITY",
    "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
    "output_mode": "content"
  },
  "name": "grep",
  "type": "tool_use"
}
```

**Tool call:** `grep` (id: `toolu_01WTMpPvmysRk6ki81NaCGZS`)

```json
{
  "pattern": "not both",
  "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
  "output_mode": "content"
}
```

**Tool call:** `grep` (id: `toolu_01F3e8gbT1vowcMEeiaj9AL2`)

```json
{
  "pattern": "throws.*MUTUAL_EXCLUSIVITY",
  "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md",
  "output_mode": "content"
}
```

## Tool Result

/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/api-surface.md:
  1233: // Value: "Provide either 'project_path' or 'cwd_path', not both."
/Users/smordziol/Webserver/tools/ai-insights/mcp-server/docs/agents/project-manifest/constraints.md:
  1281: - Schemas that only contain `project_path` (mandatory) or only `cwd_path` — but not both as optional fields — are exempt from this consideration. `DetectProjectSchema`, `InitializeProjectSchema`, and `ListProjectsSchema` fall into this category.

## Tool Result

No matches found

## Assistant

The remaining "not both" in `constraints.md` at line 1281 is fine — it's talking about schemas that contain only one of the two fields (not both as optional fields), which is about schema structure, not about runtime behavior. That's a schema description, not a mutual-exclusivity enforcement rule.

The `api-surface.md` reference is outside our scope (the WP only mentions `constraints.md`).

Now let me do a final check on both files to confirm all AC criteria are met:

1. ✅ `help-content.ts` no longer contains any text describing `project_path`/`cwd_path` as mutually exclusive
2. ✅ `help-content.ts` documents that `project_path` takes precedence when both parameters are provided
3. ✅ `constraints.md` no longer states that supplying both parameters is an error
4. ✅ `constraints.md` describes the `project_path`-wins precedence rule clearly

Let me do one final scan to confirm there's no "not both" or mutual exclusivity error language remaining in the target files:

```json
{
  "id": "toolu_01PFE7a1j87xQtq4fEowA8qF",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "either.*or.*not both",
    "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
    "output_mode": "content"
  },
  "name": "grep",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_016A28WfnxW745qLtyar2KsK",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "precedence",
    "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
    "output_mode": "content"
  },
  "name": "grep",
  "type": "tool_use"
}
```

**Tool call:** `grep` (id: `toolu_01PFE7a1j87xQtq4fEowA8qF`)

```json
{
  "pattern": "either.*or.*not both",
  "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
  "output_mode": "content"
}
```

**Tool call:** `grep` (id: `toolu_016A28WfnxW745qLtyar2KsK`)

```json
{
  "pattern": "precedence",
  "path": "/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts",
  "output_mode": "content"
}
```

## Tool Result

No matches found

## Tool Result

/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/begin-work.ts:
  23:     .describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/help-content.ts:
  13: **Most tools accept \`project_path\` and/or \`cwd_path\`.** If you have \`project_path\` (the plan folder), use it — it's the fastest path. If you only know your workspace directory, pass \`cwd_path\` and the server auto-detects the active project. If you pass both, \`project_path\` takes precedence and \`cwd_path\` is ignored. The one exception is \`ledger_initialize_project\`, which requires \`project_path\` (the plan folder is being created and cannot be detected yet).
  112: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  162: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  181: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  203: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  245: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  293: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  327: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  374: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  401: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  445: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  469: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  492: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  519: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  553: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  635: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
  702: - **project_path** (string): Plan folder path — use if already known; takes precedence over cwd_path if both are provided.
/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/observations.ts:
  18:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
  112:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/pipeline.ts:
  125:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
  296:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
  657:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
  727:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/project-lifecycle.ts:
  99:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
  656:     // Filter by explicit status first (takes precedence over include_archived)
  695:     .describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/work-package.ts:
  98:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
  147:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
  207:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
  448:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
  593:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
  1159:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
  1288:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/workflow-handoff.ts:
  71:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),
/Users/smordziol/Webserver/tools/ai-insights/mcp-server/src/tools/workflow-next-action.ts:
  66:   project_path: z.string().optional().describe('Absolute path to the plan folder. Use this if you already have it from a previous tool response or if it was provided in your instructions. Takes precedence over cwd_path if both are given.'),

## Assistant

Both files look great. The `help-content.ts` now consistently describes the precedence rule throughout — the overview paragraph says "If you pass both, `project_path` takes precedence and `cwd_path` is ignored", and every per-tool parameter entry says "takes precedence over cwd_path if both are provided". No "not both" language remains.

Now let me complete the pipeline:

```json
{
  "id": "toolu_01HndsiJ8MA3FGG7DPBNSYnT",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-002",
    "type": "implementation",
    "status": "PASS",
    "agent_role": "Developer",
    "summary": "Updated help-content.ts and constraints.md to replace mutual-exclusivity language with the project_path-wins precedence rule. In help-content.ts: replaced the overview paragraph (line 13) with the exact new paragraph from the WP spec, and bulk-replaced 16 instances of the per-tool parameter descriptions that said 'Provide this OR \u2026 \u2014 not both.' with descriptions that describe the precedence rule. In constraints.md: rewrote Constraint 57, retitling it from 'Mutual Exclusivity of project_path and cwd_path' to 'project_path Takes Precedence Over cwd_path When Both Are Provided', replacing the error-throw description with a 3-step precedence rule, and adding a 'Guidance for callers' section.",
    "artifacts": {
      "files_modified": [
        "mcp-server/src/tools/help-content.ts",
        "mcp-server/docs/agents/project-manifest/constraints.md"
      ]
    },
    "acceptance_criteria_updates": [
      {
        "criterion": "help-content.ts no longer contains any text describing project_path/cwd_path as mutually exclusive",
        "met": true
      },
      {
        "criterion": "help-content.ts documents that project_path takes precedence when both parameters are provided",
        "met": true
      },
      {
        "criterion": "constraints.md no longer states that supplying both parameters is an error",
        "met": true
      },
      {
        "criterion": "constraints.md describes the project_path-wins precedence rule clearly",
        "met": true
      }
    ],
    "comments": [
      {
        "type": "improvement",
        "priority": "low",
        "note": "help-content.ts contains per-tool parameter descriptions that were repeated verbatim 16 times (once per tool). A future refactor could extract the common cwd_path/project_path parameter description into a shared constant to avoid drift if the wording needs updating again. The bulk-replace pattern used here mitigates risk for now but a DRY helper would be more robust."
      }
    ],
    "handoff_notes": "All four acceptance criteria met. Two files modified: (1) help-content.ts \u2014 overview paragraph and 16 per-tool parameter descriptions updated to remove 'not both' language and document project_path-wins precedence. (2) constraints.md \u2014 Constraint 57 fully rewritten with new title, 3-step precedence rule, caller guidance, and updated rationale. No logic changes \u2014 documentation only.",
    "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-project-path-cwd-path-resolution"
  },
  "name": "ledger_complete_pipeline",
  "type": "tool_use"
}
```

**Tool call:** `ledger_complete_pipeline` (id: `toolu_01HndsiJ8MA3FGG7DPBNSYnT`)

```json
{
  "work_package_id": "WP-002",
  "type": "implementation",
  "status": "PASS",
  "agent_role": "Developer",
  "summary": "Updated help-content.ts and constraints.md to replace mutual-exclusivity language with the project_path-wins precedence rule. In help-content.ts: replaced the overview paragraph (line 13) with the exact new paragraph from the WP spec, and bulk-replaced 16 instances of the per-tool parameter descriptions that said 'Provide this OR \u2026 \u2014 not both.' with descriptions that describe the precedence rule. In constraints.md: rewrote Constraint 57, retitling it from 'Mutual Exclusivity of project_path and cwd_path' to 'project_path Takes Precedence Over cwd_path When Both Are Provided', replacing the error-throw description with a 3-step precedence rule, and adding a 'Guidance for callers' section.",
  "artifacts": {
    "files_modified": [
      "mcp-server/src/tools/help-content.ts",
      "mcp-server/docs/agents/project-manifest/constraints.md"
    ]
  },
  "acceptance_criteria_updates": [
    {
      "criterion": "help-content.ts no longer contains any text describing project_path/cwd_path as mutually exclusive",
      "met": true
    },
    {
      "criterion": "help-content.ts documents that project_path takes precedence when both parameters are provided",
      "met": true
    },
    {
      "criterion": "constraints.md no longer states that supplying both parameters is an error",
      "met": true
    },
    {
      "criterion": "constraints.md describes the project_path-wins precedence rule clearly",
      "met": true
    }
  ],
  "comments": [
    {
      "type": "improvement",
      "priority": "low",
      "note": "help-content.ts contains per-tool parameter descriptions that were repeated verbatim 16 times (once per tool). A future refactor could extract the common cwd_path/project_path parameter description into a shared constant to avoid drift if the wording needs updating again. The bulk-replace pattern used here mitigates risk for now but a DRY helper would be more robust."
    }
  ],
  "handoff_notes": "All four acceptance criteria met. Two files modified: (1) help-content.ts \u2014 overview paragraph and 16 per-tool parameter descriptions updated to remove 'not both' language and document project_path-wins precedence. (2) constraints.md \u2014 Constraint 57 fully rewritten with new title, 3-step precedence rule, caller guidance, and updated rationale. No logic changes \u2014 documentation only.",
  "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-project-path-cwd-path-resolution"
}
```

## Tool Result

{
  "work_package_id": "WP-002",
  "work_package_file": "work/WP-003.md",
  "status": "IN_PROGRESS",
  "assigned_to": "Developer",
  "dependencies": [],
  "acceptance_criteria": [
    {
      "criterion": "help-content.ts no longer contains any text describing project_path/cwd_path as mutually exclusive",
      "met": true
    },
    {
      "criterion": "help-content.ts documents that project_path takes precedence when both parameters are provided",
      "met": true
    },
    {
      "criterion": "constraints.md no longer states that supplying both parameters is an error",
      "met": true
    },
    {
      "criterion": "constraints.md describes the project_path-wins precedence rule clearly",
      "met": true
    }
  ],
  "active_pipeline_stages": [
    "implementation",
    "documentation"
  ],
  "revision": 0,
  "pipelines": [
    {
      "type": "implementation",
      "status": "PASS",
      "started_at": "2026-03-24T21:09:50Z",
      "completed_at": "2026-03-24T21:12:17Z",
      "duration_ms": 147000,
      "summary": [
        "Updated help-content.ts and constraints.md to replace mutual-exclusivity language with the project_path-wins precedence rule. In help-content.ts: replaced the overview paragraph (line 13) with the exact new paragraph from the WP spec, and bulk-replaced 16 instances of the per-tool parameter descriptions that said 'Provide this OR … — not both.' with descriptions that describe the precedence rule. In constraints.md: rewrote Constraint 57, retitling it from 'Mutual Exclusivity of project_path and cwd_path' to 'project_path Takes Precedence Over cwd_path When Both Are Provided', replacing the error-throw description with a 3-step precedence rule, and adding a 'Guidance for callers' section."
      ],
      "artifacts": {
        "files_modified": [
          "mcp-server/src/tools/help-content.ts",
          "mcp-server/docs/agents/project-manifest/constraints.md"
        ]
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-24T21:12:17Z",
          "note": "help-content.ts contains per-tool parameter descriptions that were repeated verbatim 16 times (once per tool). A future refactor could extract the common cwd_path/project_path parameter description into a shared constant to avoid drift if the wording needs updating again. The bulk-replace pattern used here mitigates risk for now but a DRY helper would be more robust."
        }
      ]
    }
  ],
  "status_changed_at": "2026-03-24T21:09:50Z",
  "last_updated": "2026-03-24T21:12:17Z",
  "handoff_notes": [
    {
      "from_agent": "Developer",
      "to_agent": "Documentation",
      "timestamp": "2026-03-24T21:12:17Z",
      "notes": [
        "All four acceptance criteria met. Two files modified: (1) help-content.ts — overview paragraph and 16 per-tool parameter descriptions updated to remove 'not both' language and document project_path-wins precedence. (2) constraints.md — Constraint 57 fully rewritten with new title, 3-step precedence rule, caller guidance, and updated rationale. No logic changes — documentation only."
      ]
    }
  ]
}

--- NEXT STEP ---
Pipeline PASS. Call ledger_get_handoff_status (current_agent: "Developer") to confirm your work is done and hand off to Documentation.

## Assistant

```json
{
  "id": "toolu_01H89vHJS6CQGa1GrTWzwxfL",
  "caller": {
    "type": "direct"
  },
  "input": {
    "current_agent": "Developer",
    "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-project-path-cwd-path-resolution"
  },
  "name": "ledger_get_handoff_status",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_handoff_status` (id: `toolu_01H89vHJS6CQGa1GrTWzwxfL`)

```json
{
  "current_agent": "Developer",
  "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-project-path-cwd-path-resolution"
}
```

## Tool Result

{
  "current_agent": "Developer",
  "next_agent": "Developer",
  "status": "IN_PROGRESS",
  "details": "Implementation work in progress. 3 work package(s) still need implementation or rework.",
  "next_action": "Call ledger_get_next_action with agent_role: \"Developer\" to find the next work package to implement. Continue working until all WPs have PASS implementation pipelines."
}

## Assistant

There are more WPs for the Developer. Let me check what's next:

```json
{
  "id": "toolu_01Bxri39GFZWeyDdsg8eAzVy",
  "caller": {
    "type": "direct"
  },
  "input": {
    "agent_role": "Developer",
    "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-project-path-cwd-path-resolution"
  },
  "name": "ledger_get_next_action",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_next_action` (id: `toolu_01Bxri39GFZWeyDdsg8eAzVy`)

```json
{
  "agent_role": "Developer",
  "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-project-path-cwd-path-resolution"
}
```

## Tool Result

{
  "action": "CLAIM_WP",
  "work_package_id": "WP-003",
  "reason": "Work package WP-003 is READY and assigned to Developer with all dependencies satisfied.",
  "next_steps": [
    "1. Call ledger_begin_work (work_package_id: \"WP-003\", type: \"implementation\", agent_role: \"Developer\") to claim and start the pipeline in one step.",
    "2. Read the WP spec, implement the changes, run tests.",
    "3. Call ledger_complete_pipeline (work_package_id: \"WP-003\", type: \"implementation\", status: PASS/FAIL, summary, artifacts, comments, acceptance_criteria_updates).",
    "4. Call ledger_get_handoff_status (current_agent: \"Developer\")."
  ]
}

## Assistant

The scope restriction in my instructions says I must **only** operate on WP-002. The next action is for WP-003, which is outside my scope. My WP-002 implementation is complete.

---

**WP-002 implementation complete.** Here's a summary of what was done:

## Changes Made

### `mcp-server/src/tools/help-content.ts`
- **Overview paragraph** (line 13): Replaced `"Most tools accept either cwd_path or project_path — not both."` with the exact new paragraph from the WP spec: `"Most tools accept project_path and/or cwd_path. If you have project_path (the plan folder), use it — it's the fastest path. If you only know your workspace directory, pass cwd_path and the server auto-detects the active project. If you pass both, project_path takes precedence and cwd_path is ignored."`
- **16 per-tool parameter descriptions**: Bulk-replaced all occurrences of `*(Provide this OR project_path — not both.)*` / `*(Provide this OR cwd_path — not both.)*` with descriptions that document the precedence rule: `"takes precedence over cwd_path if both are provided"`

### `mcp-server/docs/agents/project-manifest/constraints.md`
- **Constraint 57**: Fully rewritten — retitled from *"Mutual Exclusivity of project_path and cwd_path"* to *"project_path Takes Precedence Over cwd_path When Both Are Provided"*, replaced the error-throw description with a 3-step precedence rule, added a *"Guidance for callers"* section, and updated the rationale to reflect that passing both parameters is now accepted (not an error).

All 4 acceptance criteria are marked **met** in the ledger.

CURRENT AGENT: Developer
NEXT AGENT: Developer
STATUS: IN_PROGRESS

---

## Token Usage

| Metric | Count |
| ------ | ----- |
| Input Tokens | 1320685 |
| Output Tokens | 8099 |
| Total Tokens | 1328784 |

