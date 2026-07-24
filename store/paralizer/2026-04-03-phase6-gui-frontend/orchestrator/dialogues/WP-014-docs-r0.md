# Dialogue — docs / WP-014

| Field | Value |
| ----- | ----- |
| Stage | `docs` |
| WP ID | `WP-014` |
| Captured | 2026-04-07T21:07:39.714731+00:00 |

## Human

Please start using the project path: `/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend`.

> NOTE: You can use this project path for all ledger tool calls to identify the current project.


## Assistant

```json
{
  "id": "toolu_01QXupwUmT3i6xBhdN6yVtwK",
  "caller": {
    "type": "direct"
  },
  "input": {
    "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend",
    "agent_role": "Documentation"
  },
  "name": "ledger_get_next_action",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_next_action` (id: `toolu_01QXupwUmT3i6xBhdN6yVtwK`)

```json
{
  "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend",
  "agent_role": "Documentation"
}
```

## Tool Result

{
  "action": "WRITE_DOCS",
  "work_package_id": "WP-014",
  "reason": "Work package WP-014 has PASS code-review pipeline. Write or update documentation.",
  "next_steps": [
    "1. Call ledger_begin_work (work_package_id: \"WP-014\", type: \"documentation\", agent_role: \"Documentation\").",
    "2. Call ledger_get_work_package to review implementation artifacts and review comments.",
    "3. Update documentation, README files, and inline docs as needed.",
    "4. Call ledger_complete_pipeline (work_package_id: \"WP-014\", type: \"documentation\", status: PASS/FAIL, summary, artifacts, comments, acceptance_criteria_updates).",
    "5. Call ledger_get_handoff_status (current_agent: \"Documentation\")."
  ],
  "handoff_notes": [
    "WP-014 code review PASS. One Fix-Forward applied: removed dead assignment in cancelDescBtn handler (project-detail.js). Two documentation-forward items for the Documentation agent: (1) note normaliseRepo duplication across repositories.js and project-detail.js, recommend future consolidation into utils/normalise.js; (2) add module-level note explaining the full-refresh-on-mutation strategy in project-detail.js. All 6 AC confirmed met. No blocking issues."
  ]
}

## Assistant

```json
{
  "id": "toolu_01TZG8gkopCwPVzgHYS2ppc4",
  "caller": {
    "type": "direct"
  },
  "input": {
    "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend",
    "work_package_id": "WP-014",
    "type": "documentation",
    "agent_role": "Documentation"
  },
  "name": "ledger_begin_work",
  "type": "tool_use"
}
```

**Tool call:** `ledger_begin_work` (id: `toolu_01TZG8gkopCwPVzgHYS2ppc4`)

```json
{
  "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend",
  "work_package_id": "WP-014",
  "type": "documentation",
  "agent_role": "Documentation"
}
```

## Tool Result

{
  "work_package_id": "WP-014",
  "work_package_file": "work/WP-005.md",
  "status": "IN_PROGRESS",
  "assigned_to": "Documentation",
  "dependencies": [
    "WP-011",
    "WP-013"
  ],
  "acceptance_criteria": [
    {
      "criterion": "Navigating to #/projects/:id fetches the project via GET /api/projects/:id and displays the project's metadata (ID, name, description), repositories list, and workspaces list.",
      "met": true
    },
    {
      "criterion": "Adding a repository presents a selection from the global repository list (filtered to exclude already-added repos) and calls POST /api/projects/:id/repositories with { repositoryId }.",
      "met": true
    },
    {
      "criterion": "Removing a repository from the project shows a confirmation dialog and calls DELETE /api/projects/:id/repositories/:repoId on confirm.",
      "met": true
    },
    {
      "criterion": "The 'Add Workspace' form validates the workspace ID format (2-6 uppercase A-Z letters) and creates the workspace via POST /api/projects/:id/workspaces.",
      "met": true
    },
    {
      "criterion": "The STABLE workspace's delete button is visually disabled and non-functional.",
      "met": true
    },
    {
      "criterion": "Renaming a project shows a confirmation dialog, calls PUT /api/projects/:id/rename with { newId }, and navigates to #/projects/:newId on success.",
      "met": true
    }
  ],
  "active_pipeline_stages": [
    "implementation",
    "qa",
    "code-review",
    "documentation"
  ],
  "revision": 0,
  "pipelines": [
    {
      "type": "implementation",
      "status": "PASS",
      "started_at": "2026-04-07T20:55:45Z",
      "completed_at": "2026-04-07T20:57:18Z",
      "duration_ms": 93000,
      "summary": [
        "Verified full implementation of gui/public/js/views/project-detail.js (901 lines) covering all AC.",
        "Verified app.js correctly imports renderProjectDetail and setRouter, registers #/projects/:id route, and calls setProjectDetailRouter(router) before router.start().",
        "Ran the pre-existing qa-wp014-tests.mjs test suite: 53/53 tests passed with exit code 0.",
        "All six acceptance criteria verified by static analysis and runtime tests."
      ],
      "artifacts": {
        "files_modified": [
          "gui/public/js/views/project-detail.js",
          "gui/public/js/app.js"
        ]
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T20:57:18Z",
          "note": "project-detail.js: The `refresh()` helper re-renders the whole view by calling renderProjectDetail recursively. For mutation-heavy workflows (e.g., adding several repos in sequence) this triggers three parallel API calls each time. A targeted section re-render (only repos or only workspaces) would be more efficient, but is a deferred optimisation — the current full-refresh approach is correct and readable."
        },
        {
          "type": "refactor",
          "priority": "low",
          "timestamp": "2026-04-07T20:57:18Z",
          "note": "project-detail.js: The normalise* functions (normaliseProject, normaliseRepo, normaliseWorkspace) handle both Go-capitalised and lowercase keys. These patterns are repeated across multiple view files. A shared normalisation utility in a separate module (e.g., gui/public/js/utils/normalise.js) would reduce duplication if more views need the same treatment."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T20:57:18Z",
          "note": "project-detail.js buildMetaSection: The cancelDescBtn click handler sets editRow.hidden to false before setting it to true on the next line (lines 197-200). The first assignment is dead code and can be removed for clarity."
        },
        {
          "type": "convention",
          "priority": "low",
          "timestamp": "2026-04-07T20:57:18Z",
          "note": "project-detail.js: The module-level `_router` variable uses a leading underscore by convention to signal 'private', which is a reasonable pattern. However app.js also exports setRouter from dashboard.js under the same alias — the naming is consistent across all views and is not a problem, just worth noting for documentation."
        }
      ]
    },
    {
      "type": "qa",
      "status": "PASS",
      "started_at": "2026-04-07T20:59:54Z",
      "completed_at": "2026-04-07T21:01:53Z",
      "duration_ms": 119000,
      "summary": [
        "All 53 tests in qa-wp014-tests.mjs pass (exit code 0).",
        "Regression: WP-011 (73 tests), WP-013 (54 tests), WP-015 (40 tests) all pass — no regressions introduced.",
        "4 additional manual edge cases all pass: null Repositories field (graceful empty state), cancelDescBtn dead-code line (UI state is correct despite redundant assignment on line 197), missing createdAt renders '—', renaming to same ID is blocked client-side without calling the API.",
        "AC1-AC6 all verified by runtime DOM tests and static analysis.",
        "app.js correctly imports renderProjectDetail and setRouter (as setProjectDetailRouter), calls setProjectDetailRouter(router) before router.start(), and registers the #/projects/:id route."
      ],
      "metrics": {
        "test_coverage": "53/53 suite tests + 4 manual edge cases + 167 regression tests across WP-011, WP-013, WP-015",
        "tests_passed": 53,
        "tests_failed": 0
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:01:53Z",
          "note": "project-detail.js line 197: cancelDescBtn handler sets editRow.hidden = false before immediately setting it to true on line 199. The first assignment is dead code and has no visible effect on runtime behaviour (confirmed by manual edge-case test). Low risk — safe to remove for clarity in a follow-up clean-up."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:01:53Z",
          "note": "The rename guard (newId === project.id) correctly prevents calling the API when the same ID is entered — confirmed by manual edge-case test. Good defensive UX."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:01:53Z",
          "note": "Workspace links use encodeURIComponent() for both projectId and workspaceId — correct for IDs that may contain spaces or special characters. No issues found."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:01:53Z",
          "note": "No cross-module regressions: all 167 tests across WP-011 (shared components), WP-013 (dashboard/router), and WP-015 (repositories view) continue to pass after this implementation."
        }
      ]
    },
    {
      "type": "code-review",
      "status": "PASS",
      "started_at": "2026-04-07T21:04:46Z",
      "completed_at": "2026-04-07T21:05:56Z",
      "duration_ms": 70000,
      "summary": [
        "Code review PASS for WP-014 (Project Detail View). Reviewed project-detail.js (901 lines) and app.js across all four dimensions: maintainability, best practices, performance, and future alignment. One Fix-Forward applied (dead assignment in cancelDescBtn handler, lines 197–199). Two Documentation-Forward items tagged. No blocking issues found."
      ],
      "artifacts": {
        "files_modified": [
          "gui/public/js/views/project-detail.js",
          "gui/public/js/app.js"
        ]
      },
      "comments": [
        {
          "type": "reviewer-applied-fix",
          "priority": "low",
          "timestamp": "2026-04-07T21:05:56Z",
          "note": "project-detail.js lines 196–200 (cancelDescBtn handler): Removed dead assignment `editRow.hidden = false` that was immediately overwritten by `editRow.hidden = true` on the next line. The redundant assignment had no behavioral effect (QA-confirmed), but created a misleading read. The handler now reads: `descRow.hidden = false; editRow.hidden = true;` — clear and correct."
        },
        {
          "type": "documentation-forward",
          "priority": "medium",
          "timestamp": "2026-04-07T21:05:56Z",
          "note": "[documentation-forward] The `normaliseRepo()` function is duplicated verbatim in both `repositories.js` (line 30) and `project-detail.js` (line 67). Both handle Go-capitalised vs lowercase keys identically. The README or architecture doc should note this as a known duplication and flag `gui/public/js/utils/normalise.js` as the suggested location for a future consolidation — so the next developer adding a view does not create a third copy."
        },
        {
          "type": "documentation-forward",
          "priority": "low",
          "timestamp": "2026-04-07T21:05:56Z",
          "note": "[documentation-forward] The `refresh()` pattern (recursive call to `renderProjectDetail` on every mutation) is used in project-detail.js. This is intentional and correct for current scale, but the module-level docstring at the top of the file does not mention it. A brief note in the module JSDoc explaining the full-refresh-on-mutation strategy (and the trade-off against targeted section re-renders) would help future contributors understand the design intent without reading the implementation."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:05:56Z",
          "note": "app.js: The `setRouter` alias import (`setRouter as setProjectDetailRouter`) is clean and avoids a naming collision with dashboard.js's `setRouter`. The inline comment on line 18 adequately explains the aliasing. No changes needed — recording as a positive pattern for consistency."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:05:56Z",
          "note": "project-detail.js: The `_router` null-guard pattern (checking `if (_router)` before calling `.navigate()`) is applied consistently in three places: backLink click handler, workspace link click handler, and rename/delete navigation. This is correct defensive coding — the view remains usable even if called without router injection (e.g., in test contexts). No changes needed."
        }
      ]
    },
    {
      "type": "documentation",
      "status": "IN_PROGRESS",
      "started_at": "2026-04-07T21:07:44Z",
      "summary": []
    }
  ],
  "status_changed_at": "2026-04-07T20:55:45Z",
  "last_updated": "2026-04-07T21:07:44Z",
  "handoff_notes": [
    {
      "from_agent": "Developer",
      "to_agent": "QA",
      "timestamp": "2026-04-07T20:57:18Z",
      "notes": [
        "WP-014 (Project Detail View) implementation is fully verified. All 53 QA tests in qa-wp014-tests.mjs pass. The view implements: parallel data fetching with loading state, project metadata with inline description editing, repository section with filtered add-picker and per-repo remove+confirm, workspaces section with STABLE-disabled delete and validated Add Workspace form (/^[A-Z]{2,6}$/), rename with confirmation+navigation, and delete with confirmation+navigation. One minor dead-code line exists in buildMetaSection cancelDescBtn handler (low priority, noted in comments)."
      ]
    },
    {
      "from_agent": "QA",
      "to_agent": "Reviewer",
      "timestamp": "2026-04-07T21:01:53Z",
      "notes": [
        "WP-014 (Project Detail View) passes QA. 53/53 suite tests + 4 manual edge cases pass. All 6 AC verified. Notable for reviewer: (1) cancelDescBtn dead-code on line 197 (editRow.hidden = false immediately overwritten — low priority clean-up only); (2) normalise* helpers are duplicated across view files — candidate for a shared utility module in a future refactor; (3) refresh() re-renders the full view on every mutation — acceptable for current scale. No functional issues found."
      ]
    },
    {
      "from_agent": "Reviewer",
      "to_agent": "Documentation",
      "timestamp": "2026-04-07T21:05:56Z",
      "notes": [
        "WP-014 code review PASS. One Fix-Forward applied: removed dead assignment in cancelDescBtn handler (project-detail.js). Two documentation-forward items for the Documentation agent: (1) note normaliseRepo duplication across repositories.js and project-detail.js, recommend future consolidation into utils/normalise.js; (2) add module-level note explaining the full-refresh-on-mutation strategy in project-detail.js. All 6 AC confirmed met. No blocking issues."
      ]
    }
  ],
  "claimed": false
}

## Assistant

Good. I have the full WP-014 context. Let me now survey the existing documentation and the relevant source files.

```json
{
  "id": "toolu_01CubQuhEhZS7Qp6iYFAiLc2",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE"
  },
  "name": "ls",
  "type": "tool_use"
}
```

**Tool call:** `ls` (id: `toolu_01CubQuhEhZS7Qp6iYFAiLc2`)

```json
{
  "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE"
}
```

## Tool Result

['/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/.git/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/.gitignore', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/CHANGELOG.md', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/CONTRIBUTING.md', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/LICENSE', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/config.dist.json', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/dist/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/node_modules/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/package-lock.json', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/package.json', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp011-tests.mjs', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp013-tests.mjs', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp014-tests.mjs', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp015-tests.mjs', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/src/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/tsconfig.json']

## Assistant

```json
{
  "id": "toolu_014RX2bf3TrJEP9T1tDaL2PU",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs"
  },
  "name": "ls",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01UtvrEvVgyeNQ7yShv26Ysq",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui"
  },
  "name": "ls",
  "type": "tool_use"
}
```

**Tool call:** `ls` (id: `toolu_014RX2bf3TrJEP9T1tDaL2PU`)

```json
{
  "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs"
}
```

**Tool call:** `ls` (id: `toolu_01UtvrEvVgyeNQ7yShv26Ysq`)

```json
{
  "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui"
}
```

## Tool Result

['/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/projects/']

## Tool Result

['/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/']

## Assistant

```json
{
  "id": "toolu_01L7wvyR1LvVDqZHoJbSjZHq",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public"
  },
  "name": "ls",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01E221WPTPbZDCbJXyKctSU6",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `ls` (id: `toolu_01L7wvyR1LvVDqZHoJbSjZHq`)

```json
{
  "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public"
}
```

**Tool call:** `read_file` (id: `toolu_01E221WPTPbZDCbJXyKctSU6`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md"
}
```

## Tool Result

['/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/css/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/index.html', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/']

## Tool Result

     1	# repo-parallelizer
     2	
     3	Parallelization of VS Code workspaces with multiple local git repositories.
     4	
     5	## Prerequisites
     6	
     7	- **Node.js** >= 18
     8	- **npm** >= 9
     9	- **git** >= 2.28
    10	
    11	## Installation
    12	
    13	```bash
    14	npm install
    15	npm run build
    16	```
    17	
    18	This compiles TypeScript to `dist/` and makes the `paralizer` CLI available.
    19	
    20	## Usage
    21	
    22	### Global install (recommended)
    23	
    24	```bash
    25	npm link
    26	paralizer
    27	```
    28	
    29	### Run directly
    30	
    31	```bash
    32	node dist/index.js
    33	```
    34	
    35	> **Note:** `dist/index.js` does not have the executable bit set after compilation. Use `node dist/index.js` or `npm link` for local execution — not `./dist/index.js` directly.
    36	
    37	### npm scripts
    38	
    39	| Script | Description |
    40	|--------|-------------|
    41	| `npm run build` | Compile TypeScript (`tsc`) |
    42	| `npm run dev` | Watch mode — recompile on save (`tsc --watch`) |
    43	| `npm start` | Run compiled output via `node dist/index.js` |
    44	
    45	## Configuration
    46	
    47	At runtime the tool reads a `config.json` file located at the tool root (next to `package.json`). This file is **not committed** — create it locally before running the tool.
    48	
    49	### Setup
    50	
    51	1. Copy `config.dist.json` to `config.json`:
    52	   ```bash
    53	   cp config.dist.json config.json
    54	   ```
    55	2. Open `config.json` and fill in the two required fields (`projectsFolder` and `storageFolder`).
    56	3. **Remove the `_instructions` key** — it is an editorial note in the template and is not a valid config field. Leaving it in is harmless at runtime but may cause warnings with strict JSON schema validators.
    57	
    58	A minimal `config.json` looks like this:
    59	
    60	```json
    61	{
    62	  "projectsFolder": "/Users/me/projects",
    63	  "storageFolder": "data/storage"
    64	}
    65	```
    66	
    67	### config.json schema
    68	
    69	| Field | Type | Required | Default | Description |
    70	|-------|------|----------|---------|-------------|
    71	| `projectsFolder` | `string` | ✅ | — | Root directory that contains the git repositories to parallelise. |
    72	| `storageFolder` | `string` | ✅ | — | Directory used for internal storage. On first run, `repositories.json` and `projects-index.json` are created here automatically. |
    73	| `cloneDepth` | `number` | | `50` | Depth passed to `git clone --depth`. Use `0` for a full clone. |
    74	| `serverPort` | `number` | | `4200` | TCP port the built-in HTTP server listens on. |
    75	| `gitPollingIntervalSeconds` | `number` | | `30` | How often (in seconds) the tool polls git remotes for new commits. |
    76	
    77	### Storage structure
    78	
    79	On first run, the tool calls `initializeStorage()` automatically. This creates the following structure under `storageFolder` (directories and seed files are created only if they do not already exist):
    80	
    81	```
    82	{storageFolder}/
    83	  repositories.json       # { "Repositories": [], "SchemaVersion": 1 }
    84	  projects-index.json     # { "Projects": [], "SchemaVersion": 1 }
    85	  projects/               # per-project working directories (created by later phases)
    86	{projectsFolder}/         # root directory for git repositories (must exist before first run)
    87	```
    88	
    89	> **Note:** `initializeStorage()` is idempotent — calling it again (e.g. on subsequent runs) does not overwrite or modify existing files.
    90	
    91	### Repository management
    92	
    93	`RepositoryManager` (`src/models/repository/repository.manager.ts`) provides stateless CRUD access to `repositories.json`. Every public method re-reads the file from disk on each call so that concurrent writes from other processes are always reflected.
    94	
    95	#### Constructor
    96	
    97	```typescript
    98	new RepositoryManager(config: AppConfig)
    99	```
   100	

## Assistant

```json
{
  "id": "toolu_01RGeByA73kwMdDUEY6bM5Ht",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
    "offset": 100,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01KUuFG8rmyWpemcZoTC7V4h",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js"
  },
  "name": "ls",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01RGeByA73kwMdDUEY6bM5Ht`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
  "offset": 100,
  "limit": 200
}
```

**Tool call:** `ls` (id: `toolu_01KUuFG8rmyWpemcZoTC7V4h`)

```json
{
  "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js"
}
```

## Tool Result

   101	`AppConfig` comes from `src/config/config.types.ts`; use `loadConfig()` to obtain it at runtime.
   102	
   103	#### Methods
   104	
   105	| Method | Signature | Description |
   106	|--------|-----------|-------------|
   107	| `list` | `list(): Repository[]` | Returns all repositories in the store. |
   108	| `getById` | `getById(id: string): Repository \| undefined` | Returns the repository with the given ID, or `undefined`. |
   109	| `exists` | `exists(id: string): boolean` | Returns `true` when a repository with the given ID is in the store. |
   110	| `add` | `add(params): Repository` | Adds a new repository. See parameters below. |
   111	| `update` | `update(id: string, params: { name: string }): Repository` | Updates the `Name` of an existing repository. Throws if the ID does not exist. |
   112	| `remove` | `remove(id: string): void` | Removes a repository by ID. Throws if the ID does not exist. |
   113	
   114	#### `add()` parameters
   115	
   116	| Parameter | Type | Required | Description |
   117	|-----------|------|----------|-------------|
   118	| `url` | `string` | ✅ | Remote Git URL (HTTPS or SSH). |
   119	| `name` | `string` | | Human-readable display name. Defaults to the resolved ID. |
   120	| `id` | `string` | | Explicit repository ID. Validated via `isValidKebabCase()` after trimming (must be lowercase alphanumeric segments separated by single hyphens). When omitted, the ID is inferred from `url` via `inferSlugFromUrl()`. |
   121	
   122	`add()` throws when:
   123	- The explicit `id` is not valid kebab-case.
   124	- `id` is omitted and the URL produces an empty slug.
   125	- A repository with the same ID already exists.
   126	- A repository with the same URL already exists.
   127	
   128	### Path resolution rules
   129	
   130	Both `storageFolder` and `projectsFolder` accept **relative or absolute paths**:
   131	
   132	- **Relative path** — resolved against the tool root (the directory containing `package.json`), regardless of the current working directory when the tool is invoked.
   133	- **Absolute path** — used as-is; no transformation is applied.
   134	
   135	Examples:
   136	
   137	| Value | Resolved to |
   138	|-------|-------------|
   139	| `"data/storage"` | `<toolRoot>/data/storage` |
   140	| `"../shared/projects"` | `<toolRoot>/../shared/projects` (normalised by `path.resolve`) |
   141	| `"/Users/me/projects"` | `/Users/me/projects` |
   142	
   143	> **Note:** Path traversal sequences (e.g. `"../"`) in relative values are silently normalised by `path.resolve`. They may resolve to a directory outside the tool root — this is intentional for developer flexibility.
   144	
   145	---
   146	
   147	### Project management
   148	
   149	`ProjectManager` (`src/models/project/project.manager.ts`) provides stateless CRUD access to per-project JSON files and the shared project index. Every public method re-reads from disk on each call.
   150	
   151	#### Storage layout
   152	
   153	`ProjectManager` uses a dual-file strategy:
   154	
   155	```
   156	{storageFolder}/
   157	  projects-index.json     # lightweight listing: [{ Id, Name }, ...]
   158	  projects/
   159	    {id}.json             # full project data for each project
   160	```
   161	
   162	On first call the index file is created automatically if it does not exist.
   163	
   164	#### Constructor
   165	
   166	```typescript
   167	new ProjectManager(config: AppConfig, repositoryManager: RepositoryManager)
   168	```
   169	
   170	`RepositoryManager` is injected for repository-existence validation inside `create()` and `addRepository()`.
   171	
   172	#### Methods
   173	
   174	| Method | Signature | Description |
   175	|--------|-----------|-------------|
   176	| `list` | `list(): ProjectIndexEntry[]` | Returns all projects from the index. |
   177	| `getById` | `getById(id: string): ProjectData \| undefined` | Returns full project data, or `undefined` if not found. |
   178	| `create` | `create(name, repositoryIds, description?, id?): ProjectData` | Creates a new project. See parameters below. |
   179	| `update` | `update(id, changes): ProjectData` | Updates `Name` and/or `Description`. Keeps the index in sync. |
   180	| `rename` | `rename(oldId, newId): ProjectData` | Changes the project ID and renames the project file on disk. |
   181	| `remove` | `remove(id): void` | Deletes the project file and removes the index entry. |
   182	| `addRepository` | `addRepository(projectId, repositoryId): ProjectData` | Appends a repository ID to the project. |
   183	| `removeRepository` | `removeRepository(projectId, repositoryId): ProjectData` | Removes a repository ID from the project. |
   184	
   185	#### `create()` parameters
   186	
   187	| Parameter | Type | Required | Description |
   188	|-----------|------|----------|-------------|
   189	| `name` | `string` | ✅ | Human-readable project name. Used to generate the ID when `id` is omitted. |
   190	| `repositoryIds` | `string[]` | ✅ | IDs of repositories to associate with the project. All must exist in RepositoryManager. |
   191	| `description` | `string` | | Optional description text. Defaults to `''`. |
   192	| `id` | `string` | | Explicit project ID. Validated via `isValidKebabCase()` after trimming (must be lowercase alphanumeric segments separated by single hyphens). When omitted, the ID is generated from `name` via `toKebabCase()`. |
   193	
   194	`create()` throws when:
   195	- The explicit `id` is not valid kebab-case.
   196	- `id` is omitted and `name` produces an empty slug.
   197	- Any repository ID in `repositoryIds` does not exist.
   198	- A project with the resolved ID already exists.
   199	
   200	`create()` auto-creates a `STABLE` workspace with the current ISO 8601 timestamp.
   201	
   202	#### `rename()` parameters
   203	
   204	| Parameter | Type | Required | Description |
   205	|-----------|------|----------|-------------|
   206	| `oldId` | `string` | ✅ | ID of the project to rename. |
   207	| `newId` | `string` | ✅ | New project ID. Validated via `isValidKebabCase()` after trimming (must be lowercase alphanumeric segments separated by single hyphens). |
   208	
   209	`rename()` throws when:
   210	- `newId` is not valid kebab-case.
   211	- No project with `oldId` exists.
   212	- A project with `newId` already exists.
   213	
   214	`rename()` writes the new file before deleting the old one, so no data is lost if the process is interrupted between the two disk operations.
   215	
   216	---
   217	
   218	### Workspace management
   219	
   220	`WorkspaceManager` (`src/models/workspace/workspace.manager.ts`) provides stateless CRUD access to the Workspaces collection embedded inside each project's JSON file. All persistence is delegated to `ProjectManager` — `WorkspaceManager` has no storage files of its own.
   221	
   222	#### The STABLE workspace invariant
   223	
   224	Every project is guaranteed to have exactly one workspace with the ID `"STABLE"`. This workspace is auto-created when a project is created and **cannot be removed or renamed**:
   225	
   226	- `remove()` throws if `workspaceId` is `"STABLE"`.
   227	- `rename()` throws if `oldId` is `"STABLE"`.
   228	
   229	The `isStable()` helper captures the definition of the reserved ID in a single place.
   230	
   231	#### Constructor
   232	
   233	```typescript
   234	new WorkspaceManager(projectManager: ProjectManager)
   235	```
   236	
   237	`ProjectManager` is injected for all storage operations.
   238	
   239	#### Methods
   240	
   241	| Method | Signature | Description |
   242	|--------|-----------|-------------|
   243	| `list` | `list(projectId: string): WorkspaceInfo[]` | Returns all workspaces for the project as flat `WorkspaceInfo` objects. |
   244	| `getById` | `getById(projectId: string, workspaceId: string): WorkspaceInfo \| undefined` | Returns a single workspace, or `undefined` if not found. |
   245	| `create` | `create(projectId, workspaceId, description?): WorkspaceInfo` | Creates a new workspace. See parameters below. |
   246	| `update` | `update(projectId, workspaceId, changes): WorkspaceInfo` | Updates the `Description` of an existing workspace. |
   247	| `rename` | `rename(projectId, oldId, newId): WorkspaceInfo` | Renames a workspace by changing its ID. Cannot be used on the STABLE workspace. |
   248	| `remove` | `remove(projectId, workspaceId): void` | Removes a workspace. Cannot be used on the STABLE workspace. |
   249	| `isStable` | `isStable(workspaceId: string): boolean` | Returns `true` if and only if `workspaceId` is `"STABLE"`. |
   250	
   251	#### `create()` parameters
   252	
   253	| Parameter | Type | Required | Description |
   254	|-----------|------|----------|-------------|
   255	| `projectId` | `string` | ✅ | ID of the project to add the workspace to. |
   256	| `workspaceId` | `string` | ✅ | ID for the new workspace. Must be 2–6 uppercase ASCII letters (A–Z), no digits or special characters (e.g. `"DEV"`, `"PROD"`). Validated via `isValidWorkspaceId()`. |
   257	| `description` | `string` | | Optional description text. Defaults to `''`. |
   258	
   259	`create()` throws when:
   260	- `workspaceId` does not match the required format.
   261	- The project does not exist.
   262	- A workspace with the same ID already exists in the project.
   263	
   264	#### `rename()` parameters
   265	
   266	| Parameter | Type | Required | Description |
   267	|-----------|------|----------|-------------|
   268	| `projectId` | `string` | ✅ | ID of the project containing the workspace. |
   269	| `oldId` | `string` | ✅ | Current workspace ID. Must not be `"STABLE"`. |
   270	| `newId` | `string` | ✅ | New workspace ID. Must be 2–6 uppercase ASCII letters; must not already exist in the project. |
   271	
   272	`rename()` throws when:
   273	- `oldId` is `"STABLE"`.
   274	- `newId` does not match the required format.
   275	- The project does not exist.
   276	- The workspace with `oldId` does not exist.
   277	- A workspace with `newId` already exists.
   278	
   279	All mutations (`create`, `update`, `rename`, `remove`) update `DateModified` on the affected workspace entry.
   280	
   281	---
   282	
   283	### ID validation
   284	
   285	All three managers validate explicit IDs at the storage layer:
   286	
   287	- `ProjectManager.create(name, repos, desc, id?)` — the optional `id` parameter is validated via `isValidKebabCase()` after trimming.
   288	- `ProjectManager.rename(oldId, newId)` — the `newId` parameter is validated via `isValidKebabCase()` after trimming.
   289	- `RepositoryManager.add({ url, name, id? })` — the optional `id` parameter is validated via `isValidKebabCase()` after trimming.
   290	- `WorkspaceManager.create()` / `WorkspaceManager.rename()` — workspace IDs are validated via `isValidWorkspaceId()`.
   291	
   292	When an ID is omitted, it is derived automatically from the input string (`toKebabCase()` / `inferSlugFromUrl()`), which guarantees a safe value.
   293	
   294	Path-traversal sequences (e.g. `../../etc/passwd`), uppercase characters, spaces, and other invalid formats are rejected with a descriptive error.
   295	
   296	**Credential redaction:** `RepositoryManager.add()` redacts embedded credentials from URLs before interpolating them into error messages (e.g. `https://token@host/repo.git` → `https://***@host/repo.git`).
   297	
   298	---
   299	
   300	### Git CLI

## Tool Result

['/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/api.js', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/app.js', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/components/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/router.js', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/']

## Assistant

```json
{
  "id": "toolu_019DtT8YaxFPTuy95tjBuZZ1",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
    "offset": 300,
    "limit": 300
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01EgTDuK2wZzt4xLv7HxY9Bn",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views"
  },
  "name": "ls",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_019DtT8YaxFPTuy95tjBuZZ1`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
  "offset": 300,
  "limit": 300
}
```

**Tool call:** `ls` (id: `toolu_01EgTDuK2wZzt4xLv7HxY9Bn`)

```json
{
  "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views"
}
```

## Tool Result

   301	
   302	`src/git/git-cli.ts` provides the low-level interface for spawning Git sub-processes. All commands are executed with `shell: false` — arguments must be supplied as a pre-split array — which eliminates shell injection entirely.
   303	
   304	#### Types (`src/git/git.types.ts`)
   305	
   306	| Type | Description |
   307	|------|-------------|
   308	| `GitResult` | Resolved value from `runGit()`. Contains `exitCode: number`, `stdout: string`, and `stderr: string`. |
   309	| `GitStatusInfo` | Snapshot of a repository's working-tree status: `currentBranch`, `localCommits`, `unfetchedCommits`, `modifiedFiles`, `lastActivity`, `hasConflicts`. |
   310	| `BranchInfo` | Metadata for a single branch: `name`, `isCurrent`, `isRemote`, `upstream?`. |
   311	| `CloneOptions` | Options passed to `cloneRepository()`. Fields: `depth?` (shallow-clone commit limit), `branch?` (branch to check out), `bare?` (bare clone — no working tree). |
   312	
   313	#### `runGit(args, cwd?)`
   314	
   315	```typescript
   316	runGit(args: string[], cwd?: string): Promise<GitResult>
   317	```
   318	
   319	Spawns `git` with the given arguments and returns a `GitResult`.
   320	
   321	- **Resolves** for **all normal outcomes**, including non-zero exit codes. The caller must inspect `exitCode` to decide whether to treat the result as an error.
   322	- **Rejects** only on spawn-level failures — specifically when the `git` binary is not found on `PATH` (rejection value is a `NodeJS.ErrnoException` with `code === 'ENOENT'`).
   323	- When the process exits abnormally without a recorded exit code, `exitCode` falls back to `1` (null-coalesced).
   324	- `stdout` and `stderr` are decoded as UTF-8 using `Buffer.concat` — multi-byte characters that span chunk boundaries are handled correctly.
   325	
   326	#### `runGitOrThrow(args, cwd?)`
   327	
   328	```typescript
   329	runGitOrThrow(args: string[], cwd?: string): Promise<string>
   330	```
   331	
   332	Thin wrapper around `runGit()` that asserts success.
   333	
   334	- **Resolves** with `stdout.trim()` when `exitCode === 0`.
   335	- **Throws** an `Error` when `exitCode !== 0`. The error message has the form:
   336	  ```
   337	  git <subcommand> failed (exit <code>):
   338	  <trimmed stderr>
   339	  ```
   340	  where `<subcommand>` is `args[0]` (e.g. `clone`, `fetch`). The full args array is **not** included to avoid exposing credential-bearing URLs in logs or error reporters.
   341	
   342	#### `cloneRepository(url, destination, options?)`
   343	
   344	```typescript
   345	cloneRepository(url: string, destination: string, options?: CloneOptions): Promise<GitResult>
   346	```
   347	
   348	Clones a Git repository to a local path using `git clone`.
   349	
   350	- **Resolves** for all normal outcomes including non-zero exit codes. Inspect `GitResult.exitCode` to detect failure.
   351	- **Rejects** only on spawn-level failures (e.g. `ENOENT` when `git` is not on `PATH`).
   352	- `options` defaults to `{}` — all fields are optional.
   353	
   354	| Option | Type | Description |
   355	|--------|------|-------------|
   356	| `depth` | `number` | Truncate history to this many commits (`--depth <n>`). Must be a positive integer. Omit for a full clone. |
   357	| `branch` | `string` | Check out this branch instead of the remote default (`--branch <name>`). |
   358	| `bare` | `boolean` | Perform a bare clone (`--bare`). The destination contains only the Git object store with no working tree. Omit or set `false` for a normal clone. |
   359	
   360	> **Note:** `CloneOptions.bare` is implemented but not covered by the current test suite. Bare clone behaviour (no working tree, `HEAD` reference, remote tracking) should be verified before relying on it in production workflows.
   361	
   362	#### Security
   363	
   364	- `shell: false` is always enforced — no shell expansion, globbing, or metacharacter processing occurs.
   365	- Arguments are passed as a typed `string[]` directly to `spawn()`, preventing injection even when values come from user-supplied input.
   366	- Error messages use only `args[0]` (the subcommand name), not the full args array, to avoid leaking credential-bearing URLs.
   367	
   368	---
   369	
   370	### Branch operations (`src/git/git-branch.ts`)
   371	
   372	Seven stateless functions built over `runGit()` / `runGitOrThrow()`. All accept `repoPath: string` as their first argument pointing to a local repository.
   373	
   374	#### `listBranches(repoPath)`
   375	
   376	```typescript
   377	listBranches(repoPath: string): Promise<BranchInfo[]>
   378	```
   379	
   380	Returns all branches (local and remote-tracking) as `BranchInfo[]`. Remote-tracking branches (e.g. `origin/main`) have `isRemote: true`. The currently checked-out branch has `isCurrent: true`. Symbolic remote HEAD pointers (e.g. `origin/HEAD`) are excluded.
   381	
   382	#### `getCurrentBranch(repoPath)`
   383	
   384	```typescript
   385	getCurrentBranch(repoPath: string): Promise<string | null>
   386	```
   387	
   388	Returns the name of the currently checked-out branch, or `null` when the repository is in detached HEAD state.
   389	
   390	#### `getDefaultBranch(repoPath)`
   391	
   392	```typescript
   393	getDefaultBranch(repoPath: string): Promise<string>
   394	```
   395	
   396	Returns the repository's default branch name. Resolution order:
   397	1. Remote HEAD symbolic ref (`refs/remotes/origin/HEAD`)
   398	2. Existence of a local or remote `main` branch
   399	3. Existence of a local or remote `master` branch
   400	4. Falls back to `"main"`
   401	
   402	Always resolves (never rejects or throws).
   403	
   404	#### `createBranch(repoPath, branchName)`
   405	
   406	```typescript
   407	createBranch(repoPath: string, branchName: string): Promise<GitResult>
   408	```
   409	
   410	Creates a new branch and immediately checks it out (`git checkout -b`). Resolves for all normal outcomes; inspect `exitCode` and `stderr` for conflict or validation errors.
   411	
   412	- **Input guard:** returns `{ exitCode: 128, stderr: "fatal: '...' is not a valid branch name" }` immediately (without invoking git) if `branchName` starts with `'-'`. This prevents git from interpreting the name as a flag.
   413	
   414	#### `switchBranch(repoPath, branchName)`
   415	
   416	```typescript
   417	switchBranch(repoPath: string, branchName: string): Promise<GitResult>
   418	```
   419	
   420	Switches to an existing branch (`git checkout`). Resolves for all normal outcomes including non-zero exit codes — the caller inspects `exitCode` and `stderr` (e.g. for conflict detection).
   421	
   422	- **Input guard:** same `-` prefix guard as `createBranch()` — returns `exitCode: 128` immediately if `branchName` starts with `'-'`, preventing silent data-loss scenarios such as `git checkout --force` discarding uncommitted changes.
   423	
   424	#### `branchExists(repoPath, branchName, remote?)`
   425	
   426	```typescript
   427	branchExists(repoPath: string, branchName: string, remote?: string): Promise<boolean>
   428	```
   429	
   430	Checks whether a branch exists by verifying the ref directly via `git rev-parse --verify`.
   431	
   432	- When `remote` is omitted, checks the local ref (`refs/heads/<branchName>`).
   433	- When `remote` is provided, checks the remote-tracking ref (`refs/remotes/<remote>/<branchName>`).
   434	
   435	> **Warning:** `branchName` and `remote` are not validated against a safe refname pattern. A path-traversal value (e.g. `branchName = '../config'`) yields `refs/remotes/origin/../config`, which git resolves as `refs/remotes/config` and may return a false-positive `true` for a non-existent branch. Validate both parameters before passing untrusted input. A future cleanup WP will add the same `'-'` prefix guard already present on `createBranch()` and `switchBranch()`.
   436	
   437	#### `fetchRemote(repoPath, remote?)`
   438	
   439	```typescript
   440	fetchRemote(repoPath: string, remote?: string): Promise<GitResult>
   441	```
   442	
   443	Fetches updates from a remote. When `remote` is omitted, git fetches all configured remotes. Resolves for all normal outcomes including non-zero exit codes.
   444	
   445	> **Note:** Unlike `createBranch()` and `switchBranch()`, `fetchRemote()` does not validate the `remote` parameter against a `'-'` prefix guard. Passing `'--all'` executes `git fetch --all` (fetches all remotes) rather than failing. A future cleanup WP will address this asymmetry.
   446	
   447	#### Input validation asymmetry
   448	
   449	`createBranch()` and `switchBranch()` reject branch names starting with `'-'` (returning `exitCode: 128` immediately) because `git checkout` cannot use `--` to delimit the branch name from flags. `branchExists()` and `fetchRemote()` do not yet have this guard — they are lower-risk (no data-loss path) and the guard is planned for a future cleanup WP.
   450	
   451	---
   452	
   453	### Status operations (`src/git/git-status.ts`)
   454	
   455	Two functions that query the working-tree status of a local repository. All underlying Git sub-commands are run in parallel via `Promise.all()` — the result is a single `GitStatusInfo` snapshot.
   456	
   457	#### `getGitStatus(repoPath)`
   458	
   459	```typescript
   460	getGitStatus(repoPath: string): Promise<GitStatusInfo>
   461	```
   462	
   463	Returns a `GitStatusInfo` snapshot of the repository's current state.
   464	
   465	| Field | Type | Description |
   466	|-------|------|-------------|
   467	| `currentBranch` | `string \| null` | Checked-out branch name; `null` when HEAD is detached. |
   468	| `localCommits` | `number` | Commits the local branch is ahead of its upstream tracking branch. `0` when no upstream is configured. |
   469	| `unfetchedCommits` | `number` | Commits the upstream tracking branch is ahead of the local branch. Reflects the last-fetched remote state — call `fetchAndGetStatus()` for a live count. `0` when no upstream is configured. |
   470	| `modifiedFiles` | `number` | Number of entries reported by `git status --porcelain` (staged, unstaged, and untracked). |
   471	| `lastActivity` | `string \| null` | ISO 8601 timestamp of the most recent commit; `null` for an empty repository. |
   472	| `hasConflicts` | `boolean` | `true` when the working tree contains unresolved merge conflicts. Detected by inspecting the two-character XY codes in the porcelain output (e.g. `UU`, `AA`, `DD`, `AU`, `UA`, `DU`, `UD`). |
   473	
   474	#### `fetchAndGetStatus(repoPath)`
   475	
   476	```typescript
   477	fetchAndGetStatus(repoPath: string): Promise<GitStatusInfo>
   478	```
   479	
   480	Fetches updates from the `origin` remote, then returns the working-tree status via `getGitStatus()`.
   481	
   482	The fetch is best-effort: failures (network error, missing remote, authentication rejection) are silently ignored so the status query always succeeds. When a fetch fails, `unfetchedCommits` reflects the last known remote state rather than the current live count.
   483	
   484	---
   485	
   486	### Branch orchestrator (`src/orchestration/branch-orchestrator.ts`)
   487	
   488	`BranchOrchestrator` composes the stateless git layer (`git-branch.ts`) with the data-model managers to provide high-level branch operations across all repositories in a workspace.
   489	
   490	#### Constructor
   491	
   492	```typescript
   493	new BranchOrchestrator(
   494	    config: AppConfig,
   495	    projectManager: ProjectManager,
   496	    workspaceManager: WorkspaceManager,
   497	)
   498	```
   499	
   500	All three dependencies are injected; there is no internal state beyond the injected references.
   501	
   502	#### Methods
   503	
   504	| Method | Signature | Description |
   505	|--------|-----------|-------------|
   506	| `getAvailableBranches` | `getAvailableBranches(projectId, workspaceId): Promise<Map<string, BranchInfo[]>>` | Fetches from remote and returns all branches for every repository in the workspace. |
   507	| `compileBranchSuggestions` | `compileBranchSuggestions(branchMap): string[]` | Produces a deduplicated, sorted branch-name list from a `getAvailableBranches()` result. |
   508	| `switchBranches` | `switchBranches(projectId, workspaceId, branchAssignments): Promise<BranchSwitchResult>` | Switches (or creates) the specified branch in each repository concurrently. |
   509	
   510	#### `getAvailableBranches(projectId, workspaceId)`
   511	
   512	```typescript
   513	getAvailableBranches(projectId: string, workspaceId: string): Promise<Map<string, BranchInfo[]>>
   514	```
   515	
   516	Fetches from `origin` in every repository belonging to the project, then calls `listBranches()` to build the result map.
   517	
   518	- Fetch failures (no network, no remote configured) are silently ignored so the list always reflects at least the locally known state.
   519	- Returns a `Map` keyed by repository ID — the same IDs used in `ProjectManager`.
   520	
   521	**Throws** `Error` when the project does not exist (`"Cannot get branches: project "…" does not exist."`). Validates existence eagerly before any git operations begin.
   522	
   523	#### `compileBranchSuggestions(branchMap)`
   524	
   525	```typescript
   526	compileBranchSuggestions(branchMap: Map<string, BranchInfo[]>): string[]
   527	```
   528	
   529	Reduces a full branch map to a flat, UI-ready list:
   530	
   531	- Remote-tracking refs (e.g. `origin/main`) are normalised to their short form (`main`) so a branch known both locally and as a remote-tracking ref appears only once.
   532	- Deduplication is **case-insensitive**; the first-seen casing is preserved.
   533	- The returned list is sorted with `localeCompare`.
   534	
   535	#### `switchBranches(projectId, workspaceId, branchAssignments)`
   536	
   537	```typescript
   538	switchBranches(
   539	    projectId: string,
   540	    workspaceId: string,
   541	    branchAssignments: Record<string, string>,
   542	): Promise<BranchSwitchResult>
   543	```
   544	
   545	Switches each repository listed in `branchAssignments` to the specified branch name. All repositories run concurrently via `Promise.all()`.
   546	
   547	For each `repoId → branchName` entry:
   548	- If the branch does not exist locally **or** as a remote-tracking ref, it is created with `git checkout -b`.
   549	- If the branch already exists (locally or remotely), the repository is switched to it with `git checkout`.
   550	
   551	The workspace's `DateModified` timestamp is always updated after all per-repository operations complete, regardless of individual outcomes — including partial failures.
   552	
   553	**Return value** (`BranchSwitchResult`):
   554	
   555	```typescript
   556	{
   557	    results: {
   558	        [repoId: string]: {
   559	            success: boolean;
   560	            conflict: boolean;
   561	            error?: string;   // set when success is false
   562	        };
   563	    };
   564	}
   565	```
   566	
   567	**Throws** `Error` when the project or workspace does not exist. Unlike `getAvailableBranches()`, existence is **not** validated up front — any error surfaces only when `workspaceManager.update()` is called at the end, after all per-repository operations have already completed.
   568	
   569	> **Conflict detection:** When a switch fails, the `stderr` output is scanned for `/conflict/i` and `/overwritten by (checkout|switch)/i` patterns. If either matches, `conflict` is set to `true`; otherwise `conflict` is `false` and the raw git error is available in `error`. Because git error messages vary across versions and platforms, callers should treat `conflict === true` as a strong signal but also check `error` for platform-specific failure modes.
   570	
   571	---
   572	
   573	## Development Notes
   574	
   575	### TypeScript module resolution (Node16)
   576	
   577	This project uses `"module": "Node16"` and `"moduleResolution": "Node16"` in `tsconfig.json`. This is the most accurate emulation of how Node.js ESM natively resolves modules.
   578	
   579	**Consequence:** all relative imports in TypeScript source files **must include the `.js` extension**, even though you are writing `.ts` files:
   580	
   581	```typescript
   582	// Correct
   583	import { foo } from './utils.js';
   584	
   585	// Wrong — will produce a TypeScript error
   586	import { foo } from './utils';
   587	```
   588	
   589	TypeScript resolves `./utils.js` to `./utils.ts` at compile time, then emits `./utils.js` in the output, which is what Node.js requires at runtime. Omitting the extension will cause both a compile error and a runtime module-not-found error.
   590	
   591	### Build output
   592	
   593	Compilation targets ES2022 and outputs to `dist/`. Source maps are generated alongside each file (`*.js.map`). The `dist/` directory is excluded from version control via `.gitignore`.
   594	
   595	### Ignored files
   596	
   597	| Path | Reason |
   598	|------|--------|
   599	| `dist/` | Compiled output — regenerate with `npm run build` |
   600	| `node_modules/` | Dependencies — regenerate with `npm install` |

## Tool Result

['/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/dashboard.js', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/repositories.js']

## Assistant

```json
{
  "id": "toolu_01HKFQPuKHnk82tmp7Z4SAkU",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
    "offset": 600,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_014aXuZyLc4MEEBu6u6kU9Qf",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
    "limit": 100
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01HKFQPuKHnk82tmp7Z4SAkU`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
  "offset": 600,
  "limit": 200
}
```

**Tool call:** `read_file` (id: `toolu_014aXuZyLc4MEEBu6u6kU9Qf`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
  "limit": 100
}
```

## Tool Result

   601	| `config.json` | Local runtime configuration — not committed |
   602	
   603	---
   604	
   605	## GUI Frontend
   606	
   607	The browser-based GUI is a single-page application (SPA) served directly from `gui/public/`. It uses plain HTML, CSS, and vanilla JavaScript (ES modules) with no build step or bundler. The backend HTTP server (Phase 5) serves these files as static assets.
   608	
   609	### Architecture overview
   610	
   611	```
   612	gui/public/
   613	├── index.html                      # SPA shell: layout, nav, #app mount point, #toast-container
   614	├── css/
   615	│   └── styles.css                  # All application styles
   616	└── js/
   617	    ├── app.js                      # Bootstrap: initialises router, registers all routes
   618	    ├── router.js                   # Hash-based client-side router
   619	    ├── api.js                      # Fetch wrapper — all HTTP calls go through here
   620	    ├── components/
   621	    │   ├── confirm-dialog.js       # Promise-based modal confirmation dialog
   622	    │   ├── status-badge.js         # Git status badge DOM component
   623	    │   ├── toast.js                # Transient notification toasts
   624	    │   └── form-helpers.js         # Form field factory and required-field validation
   625	    └── views/
   626	        ├── dashboard.js            # #/ — project list + create-project form
   627	        ├── project-detail.js       # #/projects/:id — project detail & workspace list
   628	        └── repositories.js         # #/repositories — repository list & management
   629	```
   630	
   631	Routing is hash-based (`#/repositories`, `#/projects/my-project`, etc.). The router intercepts `hashchange` events and renders the matching view into the `#app` container. Each view is a JS module that builds DOM elements and binds event handlers.
   632	
   633	### Route registry
   634	
   635	All routes are registered in `app.js`. The full route table is:
   636	
   637	| Hash pattern | View module | Description |
   638	|---|---|---|
   639	| `#/` | `views/dashboard.js` | Project list (default landing page) |
   640	| `#/repositories` | `views/repositories.js` | Repository list and management |
   641	| `#/projects/:id` | `views/project-detail.js` | Project detail and workspace list |
   642	| `#/projects/:id/workspaces/:wid` | *(stub)* | Workspace detail (future WP) |
   643	| `#/projects/:id/workspaces/:wid/branch-switch` | *(stub)* | Branch switch (future WP) |
   644	
   645	The router starts automatically on page load via `router.start()`. Navigating to an unregistered hash renders a 404 message. An empty hash (`""`) is normalised to `#/` so the dashboard always loads as the default view.
   646	
   647	---
   648	
   649	### Dashboard view (`views/dashboard.js`)
   650	
   651	The dashboard is the landing page rendered at `#/`. It displays all projects fetched from `GET /api/projects` and provides a "Create Project" inline form.
   652	
   653	#### Project grid
   654	
   655	Each project is rendered as a card showing:
   656	- **Name** — links to `#/projects/:id`; clicking calls `router.navigate()` (no full page reload).
   657	- **ID** — displayed as secondary metadata below the name.
   658	- **Description** — shown when non-empty.
   659	- **Repository count** — derived from the `Repositories` / `repositories` array on the project object.
   660	- **Workspace count** — fetched in parallel via `api.workspaces.list(id)`. Failures degrade gracefully: the count shows as `0 workspaces` rather than breaking the grid.
   661	
   662	> **Note on key casing:** The Go backend returns project fields with capitalised keys (`Id`, `Name`, `Description`, `Repositories`). The dashboard normalises both forms; see `ProjectResponse` in `api.js` for details.
   663	
   664	#### Create Project form
   665	
   666	The **+ Create Project** button toggles an inline form with:
   667	- **Name** (required) — used as the project display name; the backend derives the kebab-case ID automatically.
   668	- **Description** (optional) — omitted from the API call when left blank (sent as `undefined`, not `""`).
   669	
   670	On success: a success toast is shown, the form is reset and hidden, and the project list refreshes. On failure: an error toast shows the message from the API error response.
   671	
   672	---
   673	
   674	### API client (`gui/public/js/api.js`)
   675	
   676	All communication with the backend REST API goes through the `api` object exported from `api.js`. It is organised into five namespaces, one per resource type. All methods return Promises and throw an `Error` (message taken from the `error` field in the JSON response body) for any non-2xx response.
   677	
   678	```js
   679	import { api } from './api.js';
   680	```
   681	
   682	#### Error handling
   683	
   684	Non-2xx responses throw an `Error` whose message is taken from the `error` field in the JSON response body. When the body is not JSON (e.g. a plain-text proxy error), the HTTP `statusText` is used as the fallback. HTTP 204 No Content responses resolve with `undefined`.
   685	
   686	#### `api.repositories`
   687	
   688	| Method | HTTP | URL | Body | Returns |
   689	|--------|------|-----|------|---------|
   690	| `list()` | GET | `/api/repositories` | — | `Object[]` |
   691	| `get(id)` | GET | `/api/repositories/:id` | — | `Object` |
   692	| `create(data)` | POST | `/api/repositories` | `{ url, name?, id? }` | `Object` (201) |
   693	| `update(id, data)` | PUT | `/api/repositories/:id` | `{ name }` | `Object` |
   694	| `delete(id)` | DELETE | `/api/repositories/:id` | — | `undefined` (204) |
   695	
   696	#### `api.projects`
   697	
   698	| Method | HTTP | URL | Body | Returns |
   699	|--------|------|-----|------|---------|
   700	| `list()` | GET | `/api/projects` | — | `ProjectResponse[]` |
   701	| `get(id)` | GET | `/api/projects/:id` | — | `ProjectResponse` |
   702	| `create(data)` | POST | `/api/projects` | `{ name, repositoryIds?, description?, id? }` | `ProjectResponse` (201) |
   703	| `update(id, data)` | PUT | `/api/projects/:id` | `{ name?, description? }` | `ProjectResponse` |
   704	| `rename(id, newId)` | PUT | `/api/projects/:id/rename` | `{ newId }` | `ProjectResponse` |
   705	| `delete(id)` | DELETE | `/api/projects/:id` | — | `undefined` (204) |
   706	| `addRepository(projectId, repoId)` | POST | `/api/projects/:id/repositories` | `{ repositoryId }` | `ProjectResponse` |
   707	| `removeRepository(projectId, repoId)` | DELETE | `/api/projects/:id/repositories/:repoId` | — | `undefined` (204) |
   708	
   709	**`ProjectResponse` shape:** The backend Go model serialises project fields using capitalised keys (`Id`, `Name`, `Description`, `Repositories`). View code must normalise both casings — see the `@typedef ProjectResponse` JSDoc in `api.js` for the canonical definition and a normalisation example.
   710	
   711	#### `api.workspaces`
   712	
   713	| Method | HTTP | URL | Body | Returns |
   714	|--------|------|-----|------|---------|
   715	| `list(projectId)` | GET | `/api/projects/:id/workspaces` | — | `Object[]` |
   716	| `get(projectId, wid)` | GET | `/api/projects/:id/workspaces/:wid` | — | `Object` |
   717	| `create(projectId, data)` | POST | `/api/projects/:id/workspaces` | `{ workspaceId, description? }` | `Object` (201) |
   718	| `update(projectId, wid, data)` | PUT | `/api/projects/:id/workspaces/:wid` | `{ description }` | `Object` |
   719	| `rename(projectId, wid, newId)` | PUT | `/api/projects/:id/workspaces/:wid/rename` | `{ newId }` | `Object` |
   720	| `delete(projectId, wid)` | DELETE | `/api/projects/:id/workspaces/:wid` | — | `undefined` (204) |
   721	
   722	#### `api.branches`
   723	
   724	| Method | HTTP | URL | Body | Returns |
   725	|--------|------|-----|------|---------|
   726	| `list(projectId, wid)` | GET | `/api/projects/:id/workspaces/:wid/branches` | — | `{ branches: Record<repoId, BranchInfo[]>, suggestions: string[] }` |
   727	| `switch(projectId, wid, assignments)` | POST | `/api/projects/:id/workspaces/:wid/branches/switch` | `{ assignments: Record<repoId, branchName> }` | `{ results: Record<repoId, { success, conflict, error? }> }` |
   728	
   729	`suggestions` is a pre-computed, case-insensitive-deduplicated flat list of branch names across all repositories — ready for use in a branch-name autocomplete or dropdown.
   730	
   731	#### `api.status`
   732	
   733	| Method | HTTP | URL | Body | Returns |
   734	|--------|------|-----|------|---------|
   735	| `get(projectId, wid)` | GET | `/api/projects/:id/workspaces/:wid/status` | — | `Record<repoId, GitStatusInfo \| null>` |
   736	| `refresh(projectId, wid)` | POST | `/api/projects/:id/workspaces/:wid/status/refresh` | — | `Record<repoId, GitStatusInfo \| null>` |
   737	
   738	`refresh()` forces a live git poll before returning; `get()` returns the last cached result. Each `GitStatusInfo` value has: `currentBranch`, `localCommits`, `unfetchedCommits`, `modifiedFiles`, `lastActivity`, `hasConflicts`.
   739	
   740	> **URL encoding:** All path segments (IDs, workspace IDs) are wrapped in `encodeURIComponent()` before being interpolated into URLs. This is handled transparently by the internal `request()` helper; callers pass raw ID strings.
   741	
   742	---
   743	
   744	### Components
   745	
   746	All components are ES modules under `gui/public/js/components/`. They have no external dependencies and manipulate the DOM directly using `textContent` (never `innerHTML`) to prevent XSS.
   747	
   748	---
   749	
   750	#### Confirmation dialog (`confirm-dialog.js`)
   751	
   752	```js
   753	import { showConfirm } from './components/confirm-dialog.js';
   754	
   755	try {
   756	    await showConfirm('Delete project', 'This action cannot be undone.');
   757	    // User clicked Confirm → proceed with deletion
   758	} catch {
   759	    // User clicked Cancel or pressed Escape → abort
   760	}
   761	```
   762	
   763	`showConfirm(title, message)` — renders a modal overlay and returns a `Promise<void>`.
   764	
   765	- **Resolves** when the user clicks **Confirm**.
   766	- **Rejects** (with `new Error('User cancelled')`) when the user clicks **Cancel**, presses **Escape**, or clicks the backdrop.
   767	- The overlay is appended to `document.body` and removed from the DOM on close (all three dismiss paths converge through a shared `cleanup()` function).
   768	- Uses `.modal-overlay` / `.modal` / `.modal-title` / `.modal-body` / `.modal-actions` CSS classes from `styles.css`.
   769	- Accessibility: `role="dialog"`, `aria-modal="true"`, `aria-labelledby`, `aria-describedby`. Focus is moved to the Confirm button on open.
   770	
   771	> **Known limitation:** No focus trap is implemented — pressing Tab can move focus outside the modal. This is acceptable for an internal developer tool; a WCAG 2.1 SC 2.1.2-compliant trap can be added if keyboard accessibility becomes a requirement.
   772	
   773	---
   774	
   775	#### Status badge (`status-badge.js`)
   776	
   777	```js
   778	import { createStatusBadge } from './components/status-badge.js';
   779	
   780	const badge = createStatusBadge(gitStatusInfo); // or null
   781	container.appendChild(badge);
   782	```
   783	
   784	`createStatusBadge(gitStatusInfo)` — accepts a `GitStatusInfo` object (or `null`) and returns an `HTMLElement`.
   785	
   786	The returned element contains:
   787	- A coloured pill showing the branch name (`currentBranch`, or `"detached HEAD"` when `null`).
   788	- Secondary detail chips (only rendered when the value is non-zero / present):
   789	  - Modified file count (`modifiedFiles`)
   790	  - Commits ahead of remote (`localCommits`)
   791	  - Commits behind remote (`unfetchedCommits`)
   792	  - Last activity timestamp, formatted as a human-readable relative string (`"5m ago"`, `"3h ago"`, `"2d ago"`, or a locale date for older commits)
   793	  - Conflict warning chip when `hasConflicts` is `true`
   794	
   795	When `gitStatusInfo` is `null`, a compact `"No data"` element with class `status-badge-error` is returned instead.
   796	
   797	**CSS classes** applied to the primary pill (priority order — highest wins):
   798	
   799	| Class | Condition |
   800	|-------|-----------|

## Tool Result

     1	/**
     2	 * Project Detail View — Repo Parallelizer GUI.
     3	 *
     4	 * Renders the full detail page for a single project:
     5	 *   - Project metadata (ID, name/description with inline description edit).
     6	 *   - Repositories section: list with per-repo Remove, plus "Add Repository" picker.
     7	 *   - Workspaces section: list with links, per-workspace Delete (STABLE disabled),
     8	 *     and "Add Workspace" form.
     9	 *   - Rename Project action (changes project ID).
    10	 *   - Delete Project action.
    11	 *
    12	 * @param {HTMLElement} container - The `#app` root element supplied by the router.
    13	 * @param {Object}      params    - Route params — expects `params.id` (project ID).
    14	 */
    15	
    16	import { api } from '../api.js';
    17	import { showToast } from '../components/toast.js';
    18	import { showConfirm } from '../components/confirm-dialog.js';
    19	import { createFormField, validateRequired } from '../components/form-helpers.js';
    20	
    21	// ---------------------------------------------------------------------------
    22	// Router reference — injected from app.js via setRouter()
    23	// app.js calls setRouter(router) before router.start() to avoid circular deps.
    24	// ---------------------------------------------------------------------------
    25	
    26	/** @type {import('../router.js').Router|null} */
    27	let _router = null;
    28	
    29	/**
    30	 * Inject the router instance so project-detail can navigate on rename/delete.
    31	 * Called from app.js before the router starts.
    32	 *
    33	 * @param {import('../router.js').Router} router
    34	 */
    35	export function setRouter(router) {
    36	    _router = router;
    37	}
    38	
    39	// ---------------------------------------------------------------------------
    40	// Normalisation helpers
    41	// ---------------------------------------------------------------------------
    42	
    43	/**
    44	 * Normalise a project object from the backend (Go-style capitalised keys or
    45	 * lowercase — both are supported).
    46	 *
    47	 * @param {Object} project
    48	 * @returns {{ id: string, name: string, description: string, repositories: string[] }}
    49	 */
    50	function normaliseProject(project) {
    51	    return {
    52	        id:           project.Id          || project.id          || '',
    53	        name:         project.Name        || project.name        || '',
    54	        description:  project.Description || project.description || '',
    55	        repositories: Array.isArray(project.Repositories)
    56	            ? project.Repositories
    57	            : (Array.isArray(project.repositories) ? project.repositories : []),
    58	    };
    59	}
    60	
    61	/**
    62	 * Normalise a repository object from the backend.
    63	 *
    64	 * @param {Object} repo
    65	 * @returns {{ id: string, name: string, url: string }}
    66	 */
    67	function normaliseRepo(repo) {
    68	    return {
    69	        id:   repo.Id   || repo.id   || '',
    70	        name: repo.Name || repo.name || '',
    71	        url:  repo.Url  || repo.url  || repo.URL || '',
    72	    };
    73	}
    74	
    75	/**
    76	 * Normalise a workspace object from the backend.
    77	 *
    78	 * @param {Object} ws
    79	 * @returns {{ id: string, description: string, createdAt: string }}
    80	 */
    81	function normaliseWorkspace(ws) {
    82	    return {
    83	        id:          ws.Id          || ws.id          || '',
    84	        description: ws.Description || ws.description || '',
    85	        createdAt:   ws.CreatedAt   || ws.createdAt   || ws.created_at || '',
    86	    };
    87	}
    88	
    89	// ---------------------------------------------------------------------------
    90	// Loading helper
    91	// ---------------------------------------------------------------------------
    92	
    93	/**
    94	 * Render a loading spinner into `el`.
    95	 *
    96	 * @param {HTMLElement} el
    97	 * @param {string} [label]
    98	 */
    99	function showLoading(el, label = 'Loading…') {
   100	    el.innerHTML = `

## Assistant

```json
{
  "id": "toolu_01R6HbHScPGgRCFUJPHgqgcE",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
    "offset": 100,
    "limit": 120
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01AoS8sWF6DLbmaZqVyHcJ4x",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
    "offset": 800,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01R6HbHScPGgRCFUJPHgqgcE`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
  "offset": 100,
  "limit": 120
}
```

**Tool call:** `read_file` (id: `toolu_01AoS8sWF6DLbmaZqVyHcJ4x`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
  "offset": 800,
  "limit": 200
}
```

## Tool Result

   101	        <div class="loading-indicator" aria-live="polite">
   102	            <span class="spinner" aria-hidden="true"></span>
   103	            <span>${label}</span>
   104	        </div>
   105	    `;
   106	}
   107	
   108	// ---------------------------------------------------------------------------
   109	// Section builders
   110	// ---------------------------------------------------------------------------
   111	
   112	/**
   113	 * Build the project metadata header section.
   114	 * Description is editable inline: clicking Edit shows a textarea; Save calls
   115	 * `api.projects.update()`.
   116	 *
   117	 * @param {{ id: string, name: string, description: string }} project
   118	 * @returns {HTMLElement}
   119	 */
   120	function buildMetaSection(project) {
   121	    const section = document.createElement('section');
   122	    section.className = 'project-meta-section card';
   123	
   124	    // Project ID + Name
   125	    const idRow = document.createElement('div');
   126	    idRow.className = 'project-meta-id-row';
   127	
   128	    const idLabel = document.createElement('span');
   129	    idLabel.className = 'project-meta-id text-muted';
   130	    idLabel.textContent = `ID: ${project.id}`;
   131	
   132	    const nameEl = document.createElement('h2');
   133	    nameEl.className = 'project-meta-name';
   134	    nameEl.textContent = project.name || project.id;
   135	
   136	    idRow.appendChild(nameEl);
   137	    idRow.appendChild(idLabel);
   138	    section.appendChild(idRow);
   139	
   140	    // Description — read-mode
   141	    const descRow = document.createElement('div');
   142	    descRow.className = 'project-meta-desc-row';
   143	
   144	    const descDisplay = document.createElement('p');
   145	    descDisplay.className = 'project-meta-description text-secondary';
   146	    descDisplay.textContent = project.description || 'No description.';
   147	
   148	    const editDescBtn = document.createElement('button');
   149	    editDescBtn.type      = 'button';
   150	    editDescBtn.className = 'btn btn-secondary btn-sm';
   151	    editDescBtn.textContent = 'Edit Description';
   152	
   153	    descRow.appendChild(descDisplay);
   154	    descRow.appendChild(editDescBtn);
   155	    section.appendChild(descRow);
   156	
   157	    // Description — edit-mode (hidden initially)
   158	    const editRow = document.createElement('div');
   159	    editRow.className = 'project-meta-edit-row';
   160	    editRow.hidden = true;
   161	
   162	    const descTextarea = document.createElement('textarea');
   163	    descTextarea.className = 'form-textarea';
   164	    descTextarea.rows  = 3;
   165	    descTextarea.value = project.description;
   166	    descTextarea.setAttribute('aria-label', 'Project description');
   167	    editRow.appendChild(descTextarea);
   168	
   169	    const editActions = document.createElement('div');
   170	    editActions.className = 'form-actions';
   171	
   172	    const saveDescBtn = document.createElement('button');
   173	    saveDescBtn.type      = 'button';
   174	    saveDescBtn.className = 'btn btn-primary btn-sm';
   175	    saveDescBtn.textContent = 'Save';
   176	
   177	    const cancelDescBtn = document.createElement('button');
   178	    cancelDescBtn.type      = 'button';
   179	    cancelDescBtn.className = 'btn btn-secondary btn-sm';
   180	    cancelDescBtn.textContent = 'Cancel';
   181	
   182	    editActions.appendChild(saveDescBtn);
   183	    editActions.appendChild(cancelDescBtn);
   184	    editRow.appendChild(editActions);
   185	    section.appendChild(editRow);
   186	
   187	    // ---- Behaviour ----
   188	
   189	    editDescBtn.addEventListener('click', () => {
   190	        descRow.hidden   = true;
   191	        editRow.hidden   = false;
   192	        descTextarea.value = project.description;
   193	        descTextarea.focus();
   194	    });
   195	
   196	    cancelDescBtn.addEventListener('click', () => {
   197	        descRow.hidden = false;
   198	        editRow.hidden = true;
   199	    });
   200	
   201	    saveDescBtn.addEventListener('click', async () => {
   202	        const newDesc = descTextarea.value.trim();
   203	        saveDescBtn.disabled = true;
   204	        saveDescBtn.textContent = 'Saving…';
   205	
   206	        try {
   207	            await api.projects.update(project.id, { description: newDesc });
   208	            project.description = newDesc;
   209	            descDisplay.textContent = newDesc || 'No description.';
   210	            showToast('Description updated.', 'success');
   211	            editRow.hidden = true;
   212	            descRow.hidden = false;
   213	        } catch (err) {
   214	            showToast(err.message || 'Failed to update description.', 'error');
   215	        } finally {
   216	            saveDescBtn.disabled = false;
   217	            saveDescBtn.textContent = 'Save';
   218	        }
   219	    });
   220	

## Tool Result

   801	| `status-badge-conflict` | `hasConflicts === true` |
   802	| `status-badge-modified` | `modifiedFiles > 0` |
   803	| `status-badge-ahead` | `localCommits > 0` |
   804	| `status-badge-behind` | `unfetchedCommits > 0` |
   805	| `status-badge-clean` | All other cases |
   806	
   807	> **Note:** The spec prose for this component listed different class names (`status-clean`, `status-modified`, etc.). The implementation correctly follows the authoritative `styles.css` class names (`status-badge-clean`, `status-badge-modified`, etc.) listed in the table above.
   808	
   809	---
   810	
   811	#### Toast notifications (`toast.js`)
   812	
   813	```js
   814	import { showToast } from './components/toast.js';
   815	
   816	showToast('Repository saved.', 'success');
   817	showToast('Something went wrong.', 'error');
   818	showToast('Branch list refreshed.', 'info');
   819	```
   820	
   821	`showToast(message, type, duration?)` — appends a transient notification to `#toast-container`.
   822	
   823	| Parameter | Type | Description |
   824	|-----------|------|-------------|
   825	| `message` | `string` | Text to display. |
   826	| `type` | `'success' \| 'error' \| 'info' \| 'warning'` | Visual variant. |
   827	| `duration` | `number` (optional) | Auto-dismiss delay in ms. Defaults to `4000`. |
   828	
   829	Returns the created `HTMLElement`, or `null` if `#toast-container` is absent from the DOM (a warning is logged to the console).
   830	
   831	- Each toast has CSS classes `toast toast-{type}` and includes a manual close button.
   832	- Toasts stack vertically inside `#toast-container`.
   833	- Auto-dismiss uses a CSS slide-out transition (`TOAST_ANIMATION_MS = 200 ms` must match `styles.css`).
   834	- A double-dismiss guard (`dataset.dismissing`) prevents the auto-timer and the close button from racing.
   835	
   836	The `#toast-container` element is declared in `index.html`. The toast component does not create it.
   837	
   838	---
   839	
   840	#### Form helpers (`form-helpers.js`)
   841	
   842	```js
   843	import { createFormField, validateRequired } from './components/form-helpers.js';
   844	
   845	// Build a field
   846	const nameField = createFormField('Project Name', 'text', 'name', {
   847	    required: true,
   848	    placeholder: 'my-project',
   849	});
   850	form.appendChild(nameField);
   851	
   852	// Validate on submit
   853	form.addEventListener('submit', (e) => {
   854	    e.preventDefault();
   855	    if (!validateRequired(form, ['name', 'url'])) return;
   856	    // all required fields are non-empty → proceed
   857	});
   858	```
   859	
   860	##### `createFormField(label, type, name, opts?)`
   861	
   862	Returns a `<div class="form-group">` containing a `<label>` and a form control.
   863	
   864	| Parameter | Type | Description |
   865	|-----------|------|-------------|
   866	| `label` | `string` | Human-readable label text. Appends `" *"` when `required` is true. |
   867	| `type` | `string` | Any `<input>` type (`'text'`, `'url'`, `'email'`, …), `'select'`, or `'textarea'`. |
   868	| `name` | `string` | The `name` attribute on the control. |
   869	| `opts` | `FormFieldOptions` | Optional — see table below. |
   870	
   871	**`FormFieldOptions`**
   872	
   873	| Option | Type | Default | Description |
   874	|--------|------|---------|-------------|
   875	| `required` | `boolean` | `false` | Appends `" *"` to the label. |
   876	| `placeholder` | `string` | `''` | Placeholder text (inputs only). |
   877	| `value` | `string` | `''` | Pre-populated value. |
   878	| `hint` | `string` | `''` | Optional hint text rendered below the control. |
   879	| `choices` | `string[] \| {value, label}[]` | `[]` | Options for `<select>` fields. |
   880	| `rows` | `number` | `3` | Row count for `<textarea>`. |
   881	| `id` | `string` | auto | Override the auto-generated element ID. |
   882	
   883	##### `validateRequired(form, fields)`
   884	
   885	Checks that each named field in `form` is non-empty.
   886	
   887	- Clears all existing `.field-error` inline error elements before re-validating (prevents stale errors on repeated submissions).
   888	- For each empty field: adds `class="error"` to the control and inserts an inline `.field-error` `<span>` below it.
   889	- Input controls also get an `'input'` listener that clears the error as soon as the user starts typing.
   890	- Returns `true` if all listed fields are non-empty, `false` otherwise.
   891	
   892	| Parameter | Type | Description |
   893	|-----------|------|-------------|
   894	| `form` | `HTMLFormElement` | The form to validate. |
   895	| `fields` | `string[]` | Array of `name` attribute values to check. |

## Assistant

```json
{
  "id": "toolu_018yWcjESKu4LJSj12bdUogL",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
    "offset": 892,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01Rg6wFpYnHv1SvDUbmTp1AB",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
    "offset": 220,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_018yWcjESKu4LJSj12bdUogL`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
  "offset": 892,
  "limit": 200
}
```

**Tool call:** `read_file` (id: `toolu_01Rg6wFpYnHv1SvDUbmTp1AB`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
  "offset": 220,
  "limit": 200
}
```

## Tool Result

   893	|-----------|------|-------------|
   894	| `form` | `HTMLFormElement` | The form to validate. |
   895	| `fields` | `string[]` | Array of `name` attribute values to check. |

## Tool Result

   221	    return section;
   222	}
   223	
   224	/**
   225	 * Build the Repositories section for a project.
   226	 *
   227	 * Lists repos currently in the project (cross-referenced with global repo list
   228	 * for name/URL). Provides a Remove button per repo and an "Add Repository"
   229	 * picker that excludes already-added repos.
   230	 *
   231	 * @param {string}   projectId       - Current project ID.
   232	 * @param {string[]} projectRepoIds  - Repo IDs currently in the project.
   233	 * @param {Array<{ id: string, name: string, url: string }>} allRepos
   234	 *   Full global repository list.
   235	 * @param {function(): Promise<void>} onRefresh - Re-renders the entire view.
   236	 * @returns {HTMLElement}
   237	 */
   238	function buildRepositoriesSection(projectId, projectRepoIds, allRepos, onRefresh) {
   239	    const section = document.createElement('section');
   240	    section.className = 'project-repos-section';
   241	
   242	    const heading = document.createElement('h3');
   243	    heading.className = 'section-title';
   244	    heading.textContent = 'Repositories';
   245	    section.appendChild(heading);
   246	
   247	    // Build a map for quick lookup: repoId → { id, name, url }
   248	    const repoMap = new Map(allRepos.map((r) => [r.id, r]));
   249	
   250	    // ---- Repo list ----
   251	    if (projectRepoIds.length === 0) {
   252	        const empty = document.createElement('p');
   253	        empty.className = 'empty-state-inline text-secondary';
   254	        empty.textContent = 'No repositories in this project yet.';
   255	        section.appendChild(empty);
   256	    } else {
   257	        const list = document.createElement('ul');
   258	        list.className = 'repo-list';
   259	
   260	        projectRepoIds.forEach((repoId) => {
   261	            const repo = repoMap.get(repoId);
   262	            const li   = document.createElement('li');
   263	            li.className = 'repo-list-item';
   264	
   265	            const repoInfo = document.createElement('span');
   266	            repoInfo.className = 'repo-list-info';
   267	            repoInfo.textContent = repo ? `${repo.name || repo.id} (${repo.id})` : repoId;
   268	
   269	            const removeBtn = document.createElement('button');
   270	            removeBtn.type      = 'button';
   271	            removeBtn.className = 'btn btn-danger btn-sm';
   272	            removeBtn.textContent = 'Remove';
   273	
   274	            removeBtn.addEventListener('click', async () => {
   275	                const label = repo ? (repo.name || repo.id) : repoId;
   276	                try {
   277	                    await showConfirm(
   278	                        'Remove Repository',
   279	                        `Remove "${label}" from this project? The repository itself is not deleted.`,
   280	                    );
   281	                } catch {
   282	                    return;
   283	                }
   284	
   285	                removeBtn.disabled = true;
   286	                removeBtn.textContent = 'Removing…';
   287	
   288	                try {
   289	                    await api.projects.removeRepository(projectId, repoId);
   290	                    showToast(`Repository "${label}" removed from project.`, 'success');
   291	                    await onRefresh();
   292	                } catch (err) {
   293	                    showToast(err.message || 'Failed to remove repository.', 'error');
   294	                    removeBtn.disabled = false;
   295	                    removeBtn.textContent = 'Remove';
   296	                }
   297	            });
   298	
   299	            li.appendChild(repoInfo);
   300	            li.appendChild(removeBtn);
   301	            list.appendChild(li);
   302	        });
   303	
   304	        section.appendChild(list);
   305	    }
   306	
   307	    // ---- Add Repository picker ----
   308	    const availableRepos = allRepos.filter((r) => !projectRepoIds.includes(r.id));
   309	
   310	    if (availableRepos.length > 0) {
   311	        const addRow = document.createElement('div');
   312	        addRow.className = 'add-repo-picker-row';
   313	
   314	        const selectEl = document.createElement('select');
   315	        selectEl.className = 'form-select repo-picker-select';
   316	
   317	        const defaultOpt = document.createElement('option');
   318	        defaultOpt.value       = '';
   319	        defaultOpt.textContent = '— Select a repository to add —';
   320	        selectEl.appendChild(defaultOpt);
   321	
   322	        availableRepos.forEach((r) => {
   323	            const opt = document.createElement('option');
   324	            opt.value       = r.id;
   325	            opt.textContent = r.name ? `${r.name} (${r.id})` : r.id;
   326	            selectEl.appendChild(opt);
   327	        });
   328	
   329	        const addBtn = document.createElement('button');
   330	        addBtn.type      = 'button';
   331	        addBtn.className = 'btn btn-primary btn-sm';
   332	        addBtn.textContent = 'Add';
   333	
   334	        addRow.appendChild(selectEl);
   335	        addRow.appendChild(addBtn);
   336	        section.appendChild(addRow);
   337	
   338	        addBtn.addEventListener('click', async () => {
   339	            const selectedId = selectEl.value;
   340	            if (!selectedId) {
   341	                showToast('Please select a repository to add.', 'error');
   342	                return;
   343	            }
   344	
   345	            addBtn.disabled = true;
   346	            addBtn.textContent = 'Adding…';
   347	
   348	            try {
   349	                await api.projects.addRepository(projectId, selectedId);
   350	                const label = repoMap.get(selectedId);
   351	                showToast(
   352	                    `Repository "${label ? (label.name || label.id) : selectedId}" added to project.`,
   353	                    'success',
   354	                );
   355	                await onRefresh();
   356	            } catch (err) {
   357	                showToast(err.message || 'Failed to add repository.', 'error');
   358	                addBtn.disabled = false;
   359	                addBtn.textContent = 'Add';
   360	            }
   361	        });
   362	    } else if (allRepos.length > 0) {
   363	        const allAdded = document.createElement('p');
   364	        allAdded.className = 'empty-state-inline text-secondary';
   365	        allAdded.textContent = 'All registered repositories are already in this project.';
   366	        section.appendChild(allAdded);
   367	    }
   368	
   369	    return section;
   370	}
   371	
   372	/**
   373	 * Build the Workspaces section for a project.
   374	 *
   375	 * Lists workspaces with ID, description, creation date, a link to the
   376	 * workspace detail view, and a Delete button (disabled for STABLE).
   377	 * Includes an "Add Workspace" form.
   378	 *
   379	 * @param {string}   projectId  - Current project ID.
   380	 * @param {Array<{ id: string, description: string, createdAt: string }>} workspaces
   381	 * @param {function(): Promise<void>} onRefresh - Re-renders the entire view.
   382	 * @returns {HTMLElement}
   383	 */
   384	function buildWorkspacesSection(projectId, workspaces, onRefresh) {
   385	    const section = document.createElement('section');
   386	    section.className = 'project-workspaces-section';
   387	
   388	    const heading = document.createElement('h3');
   389	    heading.className = 'section-title';
   390	    heading.textContent = 'Workspaces';
   391	    section.appendChild(heading);
   392	
   393	    // ---- Workspace list ----
   394	    if (workspaces.length === 0) {
   395	        const empty = document.createElement('p');
   396	        empty.className = 'empty-state-inline text-secondary';
   397	        empty.textContent = 'No workspaces yet.';
   398	        section.appendChild(empty);
   399	    } else {
   400	        const table = document.createElement('table');
   401	        table.className = 'data-table workspaces-table';
   402	
   403	        const thead = document.createElement('thead');
   404	        const htr   = document.createElement('tr');
   405	        ['ID', 'Description', 'Created', 'Actions'].forEach((label) => {
   406	            const th = document.createElement('th');
   407	            th.textContent = label;
   408	            htr.appendChild(th);
   409	        });
   410	        thead.appendChild(htr);
   411	        table.appendChild(thead);
   412	
   413	        const tbody = document.createElement('tbody');
   414	
   415	        workspaces.forEach((ws) => {
   416	            const tr = document.createElement('tr');
   417	            tr.dataset.workspaceId = ws.id;
   418	
   419	            // ID + link cell
   420	            const idCell = document.createElement('td');

## Assistant

I have enough context from the source and WP detail. Now let me read the end of the README to see exactly where the `## GUI Frontend` section ends:

```json
{
  "id": "toolu_01CYRJeY4FiXj7DxPEdRTYBi",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
    "offset": 880,
    "limit": 50
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_013MsA1GstHmYZ8jGrQ6LJZu",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
    "offset": 420,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01CYRJeY4FiXj7DxPEdRTYBi`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
  "offset": 880,
  "limit": 50
}
```

**Tool call:** `read_file` (id: `toolu_013MsA1GstHmYZ8jGrQ6LJZu`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
  "offset": 420,
  "limit": 200
}
```

## Tool Result

   881	| `id` | `string` | auto | Override the auto-generated element ID. |
   882	
   883	##### `validateRequired(form, fields)`
   884	
   885	Checks that each named field in `form` is non-empty.
   886	
   887	- Clears all existing `.field-error` inline error elements before re-validating (prevents stale errors on repeated submissions).
   888	- For each empty field: adds `class="error"` to the control and inserts an inline `.field-error` `<span>` below it.
   889	- Input controls also get an `'input'` listener that clears the error as soon as the user starts typing.
   890	- Returns `true` if all listed fields are non-empty, `false` otherwise.
   891	
   892	| Parameter | Type | Description |
   893	|-----------|------|-------------|
   894	| `form` | `HTMLFormElement` | The form to validate. |
   895	| `fields` | `string[]` | Array of `name` attribute values to check. |

## Tool Result

   421	            const wsLink = document.createElement('a');
   422	            wsLink.href      = `#/projects/${encodeURIComponent(projectId)}/workspaces/${encodeURIComponent(ws.id)}`;
   423	            wsLink.textContent = ws.id;
   424	            wsLink.className = 'workspace-link';
   425	            if (_router) {
   426	                wsLink.addEventListener('click', (e) => {
   427	                    e.preventDefault();
   428	                    _router.navigate(
   429	                        `#/projects/${encodeURIComponent(projectId)}/workspaces/${encodeURIComponent(ws.id)}`,
   430	                    );
   431	                });
   432	            }
   433	            idCell.appendChild(wsLink);
   434	            tr.appendChild(idCell);
   435	
   436	            // Description cell
   437	            const descCell = document.createElement('td');
   438	            descCell.textContent = ws.description || '—';
   439	            tr.appendChild(descCell);
   440	
   441	            // Created-at cell
   442	            const createdCell = document.createElement('td');
   443	            createdCell.className = 'text-muted';
   444	            if (ws.createdAt) {
   445	                try {
   446	                    createdCell.textContent = new Date(ws.createdAt).toLocaleDateString();
   447	                } catch {
   448	                    createdCell.textContent = ws.createdAt;
   449	                }
   450	            } else {
   451	                createdCell.textContent = '—';
   452	            }
   453	            tr.appendChild(createdCell);
   454	
   455	            // Actions cell
   456	            const actCell = document.createElement('td');
   457	            actCell.className = 'workspace-actions-cell';
   458	
   459	            const isStable = ws.id === 'STABLE';
   460	
   461	            const deleteBtn = document.createElement('button');
   462	            deleteBtn.type      = 'button';
   463	            deleteBtn.className = 'btn btn-danger btn-sm';
   464	            deleteBtn.textContent = 'Delete';
   465	
   466	            if (isStable) {
   467	                deleteBtn.disabled = true;
   468	                deleteBtn.title    = 'The STABLE workspace cannot be deleted.';
   469	                deleteBtn.classList.add('btn-disabled');
   470	            } else {
   471	                deleteBtn.addEventListener('click', async () => {
   472	                    try {
   473	                        await showConfirm(
   474	                            'Delete Workspace',
   475	                            `Delete workspace "${ws.id}"? All cloned repositories in this workspace will be permanently removed. This action cannot be undone.`,
   476	                        );
   477	                    } catch {
   478	                        return;
   479	                    }
   480	
   481	                    deleteBtn.disabled    = true;
   482	                    deleteBtn.textContent = 'Deleting…';
   483	
   484	                    try {
   485	                        await api.workspaces.delete(projectId, ws.id);
   486	                        showToast(`Workspace "${ws.id}" deleted.`, 'success');
   487	                        await onRefresh();
   488	                    } catch (err) {
   489	                        showToast(err.message || 'Failed to delete workspace.', 'error');
   490	                        deleteBtn.disabled    = false;
   491	                        deleteBtn.textContent = 'Delete';
   492	                    }
   493	                });
   494	            }
   495	
   496	            actCell.appendChild(deleteBtn);
   497	            tr.appendChild(actCell);
   498	
   499	            tbody.appendChild(tr);
   500	        });
   501	
   502	        table.appendChild(tbody);
   503	        section.appendChild(table);
   504	    }
   505	
   506	    // ---- Add Workspace form ----
   507	    const addSection = buildAddWorkspaceForm(projectId, onRefresh);
   508	    section.appendChild(addSection);
   509	
   510	    return section;
   511	}
   512	
   513	/**
   514	 * Build the "Add Workspace" collapsible form.
   515	 *
   516	 * Workspace ID must match /^[A-Z]{2,6}$/ (2-6 uppercase letters).
   517	 *
   518	 * @param {string}   projectId
   519	 * @param {function(): Promise<void>} onSuccess
   520	 * @returns {HTMLElement}
   521	 */
   522	function buildAddWorkspaceForm(projectId, onSuccess) {
   523	    const wrapper = document.createElement('div');
   524	    wrapper.className = 'add-workspace-wrapper';
   525	
   526	    const toggleBtn = document.createElement('button');
   527	    toggleBtn.type      = 'button';
   528	    toggleBtn.className = 'btn btn-secondary btn-sm';
   529	    toggleBtn.textContent = '+ Add Workspace';
   530	    wrapper.appendChild(toggleBtn);
   531	
   532	    const formWrapper = document.createElement('div');
   533	    formWrapper.className = 'add-workspace-form-wrapper';
   534	    formWrapper.hidden = true;
   535	    wrapper.appendChild(formWrapper);
   536	
   537	    const form = document.createElement('form');
   538	    form.className = 'add-workspace-form card';
   539	    form.noValidate = true;
   540	
   541	    const formTitle = document.createElement('h4');
   542	    formTitle.className = 'form-section-title';
   543	    formTitle.textContent = 'New Workspace';
   544	    form.appendChild(formTitle);
   545	
   546	    const wsIdField = createFormField('Workspace ID', 'text', 'workspaceId', {
   547	        required: true,
   548	        placeholder: 'e.g. DEV or FEATURE',
   549	        hint: 'Must be 2–6 uppercase letters (A-Z only).',
   550	    });
   551	    form.appendChild(wsIdField);
   552	
   553	    const descField = createFormField('Description', 'textarea', 'description', {
   554	        placeholder: 'Optional — short description.',
   555	        rows: 2,
   556	    });
   557	    form.appendChild(descField);
   558	
   559	    // Inline validation error area for workspaceId format
   560	    const wsIdInput = wsIdField.querySelector('[name="workspaceId"]');
   561	    const wsIdErrorEl = wsIdField.querySelector('.field-error');
   562	
   563	    const actions = document.createElement('div');
   564	    actions.className = 'form-actions';
   565	
   566	    const submitBtn = document.createElement('button');
   567	    submitBtn.type      = 'submit';
   568	    submitBtn.className = 'btn btn-primary btn-sm';
   569	    submitBtn.textContent = 'Create';
   570	
   571	    const cancelBtn = document.createElement('button');
   572	    cancelBtn.type      = 'button';
   573	    cancelBtn.className = 'btn btn-secondary btn-sm';
   574	    cancelBtn.textContent = 'Cancel';
   575	
   576	    actions.appendChild(submitBtn);
   577	    actions.appendChild(cancelBtn);
   578	    form.appendChild(actions);
   579	    formWrapper.appendChild(form);
   580	
   581	    // ---- Behaviour ----
   582	
   583	    toggleBtn.addEventListener('click', () => {
   584	        formWrapper.hidden = !formWrapper.hidden;
   585	        if (!formWrapper.hidden && wsIdInput) wsIdInput.focus();
   586	    });
   587	
   588	    cancelBtn.addEventListener('click', () => {
   589	        form.reset();
   590	        formWrapper.hidden = true;
   591	    });
   592	
   593	    form.addEventListener('submit', async (e) => {
   594	        e.preventDefault();
   595	
   596	        if (!validateRequired(form, ['workspaceId'])) return;
   597	
   598	        const workspaceId = wsIdInput ? wsIdInput.value.trim() : '';
   599	
   600	        // Validate format: 2-6 uppercase A-Z only
   601	        if (!/^[A-Z]{2,6}$/.test(workspaceId)) {
   602	            if (wsIdErrorEl) {
   603	                wsIdErrorEl.textContent = 'Must be 2–6 uppercase letters (A-Z only).';
   604	                wsIdErrorEl.hidden = false;
   605	            }
   606	            if (wsIdInput) {
   607	                wsIdInput.classList.add('error');
   608	                wsIdInput.setAttribute('aria-invalid', 'true');
   609	                wsIdInput.focus();
   610	            }
   611	            return;
   612	        }
   613	
   614	        const description = form.querySelector('[name="description"]').value.trim();
   615	
   616	        submitBtn.disabled = true;
   617	        submitBtn.textContent = 'Creating…';
   618	
   619	        try {
   620	            await api.workspaces.create(projectId, {

## Assistant

```json
{
  "id": "toolu_01Qiox9aWWWa69TFaoYYHijM",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
    "offset": 620,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01Qiox9aWWWa69TFaoYYHijM`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
  "offset": 620,
  "limit": 200
}
```

## Tool Result

   621	                workspaceId,
   622	                description: description || undefined,
   623	            });
   624	            showToast(`Workspace "${workspaceId}" created.`, 'success');
   625	            form.reset();
   626	            formWrapper.hidden = true;
   627	            await onSuccess();
   628	        } catch (err) {
   629	            showToast(err.message || 'Failed to create workspace.', 'error');
   630	        } finally {
   631	            submitBtn.disabled = false;
   632	            submitBtn.textContent = 'Create';
   633	        }
   634	    });
   635	
   636	    return wrapper;
   637	}
   638	
   639	/**
   640	 * Build the "Rename Project" action section.
   641	 *
   642	 * Shows a text input for the new ID plus a confirmation dialog explaining
   643	 * the consequences (filesystem rename).  On success, navigates to the new URL.
   644	 *
   645	 * @param {{ id: string, name: string }} project
   646	 * @returns {HTMLElement}
   647	 */
   648	function buildRenameSection(project) {
   649	    const section = document.createElement('section');
   650	    section.className = 'project-rename-section card danger-zone-section';
   651	
   652	    const heading = document.createElement('h3');
   653	    heading.className = 'section-title danger-title';
   654	    heading.textContent = 'Rename Project';
   655	    section.appendChild(heading);
   656	
   657	    const desc = document.createElement('p');
   658	    desc.className = 'text-secondary';
   659	    desc.textContent =
   660	        'Changing the project ID renames the underlying directory on the filesystem and updates all references. ' +
   661	        'Existing workspace links will stop working until updated.';
   662	    section.appendChild(desc);
   663	
   664	    const row = document.createElement('div');
   665	    row.className = 'rename-row';
   666	
   667	    const newIdInput = document.createElement('input');
   668	    newIdInput.type        = 'text';
   669	    newIdInput.className   = 'form-input rename-input';
   670	    newIdInput.placeholder = 'New project ID';
   671	    newIdInput.setAttribute('aria-label', 'New project ID');
   672	    row.appendChild(newIdInput);
   673	
   674	    const renameBtn = document.createElement('button');
   675	    renameBtn.type      = 'button';
   676	    renameBtn.className = 'btn btn-warning';
   677	    renameBtn.textContent = 'Rename…';
   678	    row.appendChild(renameBtn);
   679	
   680	    section.appendChild(row);
   681	
   682	    renameBtn.addEventListener('click', async () => {
   683	        const newId = newIdInput.value.trim();
   684	        if (!newId) {
   685	            newIdInput.focus();
   686	            showToast('Please enter a new project ID.', 'error');
   687	            return;
   688	        }
   689	
   690	        if (newId === project.id) {
   691	            showToast('The new ID is the same as the current ID.', 'error');
   692	            return;
   693	        }
   694	
   695	        try {
   696	            await showConfirm(
   697	                'Rename Project',
   698	                `Rename project "${project.id}" to "${newId}"? ` +
   699	                `This renames the directory on disk and changes the URL. ` +
   700	                `All existing workspace links will use the new project ID.`,
   701	            );
   702	        } catch {
   703	            return;
   704	        }
   705	
   706	        renameBtn.disabled = true;
   707	        renameBtn.textContent = 'Renaming…';
   708	
   709	        try {
   710	            await api.projects.rename(project.id, newId);
   711	            showToast(`Project renamed to "${newId}".`, 'success');
   712	            if (_router) {
   713	                _router.navigate(`#/projects/${encodeURIComponent(newId)}`);
   714	            }
   715	        } catch (err) {
   716	            showToast(err.message || 'Failed to rename project.', 'error');
   717	            renameBtn.disabled = false;
   718	            renameBtn.textContent = 'Rename…';
   719	        }
   720	    });
   721	
   722	    return section;
   723	}
   724	
   725	/**
   726	 * Build the "Delete Project" action section.
   727	 *
   728	 * Shows a strong warning and confirmation dialog before deletion.
   729	 * On success, navigates back to the dashboard (#/).
   730	 *
   731	 * @param {{ id: string, name: string }} project
   732	 * @returns {HTMLElement}
   733	 */
   734	function buildDeleteSection(project) {
   735	    const section = document.createElement('section');
   736	    section.className = 'project-delete-section card danger-zone-section';
   737	
   738	    const heading = document.createElement('h3');
   739	    heading.className = 'section-title danger-title';
   740	    heading.textContent = 'Delete Project';
   741	    section.appendChild(heading);
   742	
   743	    const desc = document.createElement('p');
   744	    desc.className = 'text-secondary';
   745	    desc.textContent =
   746	        'Permanently deletes this project and all its workspaces from the filesystem. ' +
   747	        'This action cannot be undone.';
   748	    section.appendChild(desc);
   749	
   750	    const deleteBtn = document.createElement('button');
   751	    deleteBtn.type      = 'button';
   752	    deleteBtn.className = 'btn btn-danger';
   753	    deleteBtn.textContent = 'Delete Project…';
   754	    section.appendChild(deleteBtn);
   755	
   756	    deleteBtn.addEventListener('click', async () => {
   757	        try {
   758	            await showConfirm(
   759	                'Delete Project',
   760	                `Permanently delete project "${project.name || project.id}"? ` +
   761	                `All workspaces and cloned repositories will be removed from disk. ` +
   762	                `This action cannot be undone.`,
   763	            );
   764	        } catch {
   765	            return;
   766	        }
   767	
   768	        deleteBtn.disabled = true;
   769	        deleteBtn.textContent = 'Deleting…';
   770	
   771	        try {
   772	            await api.projects.delete(project.id);
   773	            showToast(`Project "${project.name || project.id}" deleted.`, 'success');
   774	            if (_router) {
   775	                _router.navigate('#/');
   776	            }
   777	        } catch (err) {
   778	            showToast(err.message || 'Failed to delete project.', 'error');
   779	            deleteBtn.disabled = false;
   780	            deleteBtn.textContent = 'Delete Project…';
   781	        }
   782	    });
   783	
   784	    return section;
   785	}
   786	
   787	// ---------------------------------------------------------------------------
   788	// Public export — view function
   789	// ---------------------------------------------------------------------------
   790	
   791	/**
   792	 * Render the Project Detail view.
   793	 *
   794	 * @param {HTMLElement} container - The `#app` root element.
   795	 * @param {Object}      params    - Route params — expects `params.id`.
   796	 */
   797	export async function renderProjectDetail(container, params) {
   798	    const projectId = decodeURIComponent(params.id || '');
   799	
   800	    // -----------------------------------------------------------------------
   801	    // Show loading state while fetching data
   802	    // -----------------------------------------------------------------------
   803	    showLoading(container, 'Loading project…');
   804	
   805	    // -----------------------------------------------------------------------
   806	    // Data fetching — all three in parallel
   807	    // -----------------------------------------------------------------------
   808	    let project, workspaces, allRepos;
   809	    try {
   810	        [project, workspaces, allRepos] = await Promise.all([
   811	            api.projects.get(projectId),
   812	            api.workspaces.list(projectId),
   813	            api.repositories.list(),
   814	        ]);
   815	    } catch (err) {
   816	        container.innerHTML = '';
   817	        const errMsg = document.createElement('div');
   818	        errMsg.className = 'empty-state error-state';
   819	        errMsg.textContent = `Failed to load project: ${err.message}`;
   820	        container.appendChild(errMsg);

## Assistant

```json
{
  "id": "toolu_01Nam7g923iLznPRBohEfzem",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
    "offset": 820,
    "limit": 100
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_011U6RuVbqnCq6ZucbyGiaQK",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/repositories.js",
    "limit": 50
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01Nam7g923iLznPRBohEfzem`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
  "offset": 820,
  "limit": 100
}
```

**Tool call:** `read_file` (id: `toolu_011U6RuVbqnCq6ZucbyGiaQK`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/repositories.js",
  "limit": 50
}
```

## Tool Result

   821	        showToast(err.message || 'Failed to load project.', 'error');
   822	        return;
   823	    }
   824	
   825	    const normProject    = normaliseProject(project);
   826	    const normWorkspaces = Array.isArray(workspaces)
   827	        ? workspaces.map(normaliseWorkspace)
   828	        : [];
   829	    const normAllRepos   = Array.isArray(allRepos)
   830	        ? allRepos.map(normaliseRepo)
   831	        : [];
   832	
   833	    // -----------------------------------------------------------------------
   834	    // Re-render helper — re-fetches all data and re-renders the view
   835	    // -----------------------------------------------------------------------
   836	    async function refresh() {
   837	        container.innerHTML = '';
   838	        await renderProjectDetail(container, params);
   839	    }
   840	
   841	    // -----------------------------------------------------------------------
   842	    // Clear loading state; build the real UI
   843	    // -----------------------------------------------------------------------
   844	    container.innerHTML = '';
   845	
   846	    // ---- Page header ----
   847	    const header = document.createElement('div');
   848	    header.className = 'page-header';
   849	
   850	    const backLink = document.createElement('a');
   851	    backLink.href      = '#/';
   852	    backLink.className = 'back-link text-muted';
   853	    backLink.textContent = '← Projects';
   854	    if (_router) {
   855	        backLink.addEventListener('click', (e) => {
   856	            e.preventDefault();
   857	            _router.navigate('#/');
   858	        });
   859	    }
   860	    header.appendChild(backLink);
   861	
   862	    const title = document.createElement('h1');
   863	    title.className = 'page-title';
   864	    title.textContent = normProject.name || normProject.id;
   865	    header.appendChild(title);
   866	
   867	    container.appendChild(header);
   868	
   869	    // ---- Metadata section ----
   870	    container.appendChild(buildMetaSection(normProject));
   871	
   872	    // ---- Repositories section ----
   873	    container.appendChild(
   874	        buildRepositoriesSection(
   875	            normProject.id,
   876	            normProject.repositories,
   877	            normAllRepos,
   878	            refresh,
   879	        ),
   880	    );
   881	
   882	    // ---- Workspaces section ----
   883	    container.appendChild(
   884	        buildWorkspacesSection(normProject.id, normWorkspaces, refresh),
   885	    );
   886	
   887	    // ---- Danger zone ----
   888	    const dangerZone = document.createElement('div');
   889	    dangerZone.className = 'danger-zone';
   890	
   891	    const dangerHeading = document.createElement('h3');
   892	    dangerHeading.className = 'section-title';
   893	    dangerHeading.textContent = 'Danger Zone';
   894	    dangerZone.appendChild(dangerHeading);
   895	
   896	    dangerZone.appendChild(buildRenameSection(normProject));
   897	    dangerZone.appendChild(buildDeleteSection(normProject));
   898	
   899	    container.appendChild(dangerZone);
   900	}

## Tool Result

     1	/**
     2	 * Repositories View — Repo Parallelizer GUI.
     3	 *
     4	 * Renders a full CRUD management page for all registered repositories:
     5	 *   - Table listing all repositories (ID, Name, URL).
     6	 *   - "Add Repository" inline form (URL required, Name optional, ID optional).
     7	 *   - Inline edit for repository Name per row.
     8	 *   - Delete per row with a confirmation dialog.
     9	 *
    10	 * @param {HTMLElement} container - The `#app` root element supplied by the router.
    11	 * @param {Object}      _params   - Route params (none for this route).
    12	 */
    13	
    14	import { api } from '../api.js';
    15	import { showToast } from '../components/toast.js';
    16	import { showConfirm } from '../components/confirm-dialog.js';
    17	import { createFormField, validateRequired } from '../components/form-helpers.js';
    18	
    19	// ---------------------------------------------------------------------------
    20	// Internal helpers
    21	// ---------------------------------------------------------------------------
    22	
    23	/**
    24	 * Normalise a repository object coming from the backend.
    25	 * The backend may use either capitalised or lowercase keys.
    26	 *
    27	 * @param {Object} repo - Raw repository object from the API.
    28	 * @returns {{ id: string, name: string, url: string }}
    29	 */
    30	function normaliseRepo(repo) {
    31	    return {
    32	        id:   repo.Id   || repo.id   || '',
    33	        name: repo.Name || repo.name || '',
    34	        url:  repo.Url  || repo.url  || repo.URL || '',
    35	    };
    36	}
    37	
    38	// ---------------------------------------------------------------------------
    39	// Table rendering
    40	// ---------------------------------------------------------------------------
    41	
    42	/**
    43	 * Build the `<thead>` element for the repository table.
    44	 *
    45	 * @returns {HTMLElement}
    46	 */
    47	function buildTableHead() {
    48	    const thead = document.createElement('thead');
    49	    const tr    = document.createElement('tr');
    50	

## Assistant

I now have all the information I need. The README ends at line 895. I need to:

1. Add a **Project Detail view** section to the README (the existing GUI section has Dashboard, API client, and Components — but is missing the Project Detail view entirely)
2. Add a note about the `normaliseRepo` duplication (reviewer forward item #1)
3. Add a module-level JSDoc note about the `refresh()` / full-refresh-on-mutation strategy in `project-detail.js` (reviewer forward item #2 — but this should go in the source file's module docstring, which currently doesn't mention it)

Let me also check the current module-level docstring in `project-detail.js` to see if the refresh strategy needs to be added there:

The current module-level docstring (lines 1-14) describes what the module renders but says nothing about the refresh strategy. I need to update it.

Let me now make both changes:

```json
{
  "id": "toolu_019CuSjBMFa3c388qBVkKza2",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Add Project Detail view section to README.md (after Dashboard view section)",
        "status": "in_progress"
      },
      {
        "content": "Update project-detail.js module-level JSDoc to document the refresh/full-refresh-on-mutation strategy",
        "status": "pending"
      },
      {
        "content": "Complete the documentation pipeline via ledger_complete_pipeline",
        "status": "pending"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_019CuSjBMFa3c388qBVkKza2`)

```json
{
  "todos": [
    {
      "content": "Add Project Detail view section to README.md (after Dashboard view section)",
      "status": "in_progress"
    },
    {
      "content": "Update project-detail.js module-level JSDoc to document the refresh/full-refresh-on-mutation strategy",
      "status": "pending"
    },
    {
      "content": "Complete the documentation pipeline via ledger_complete_pipeline",
      "status": "pending"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Add Project Detail view section to README.md (after Dashboard view section)', 'status': 'in_progress'}, {'content': 'Update project-detail.js module-level JSDoc to document the refresh/full-refresh-on-mutation strategy', 'status': 'pending'}, {'content': 'Complete the documentation pipeline via ledger_complete_pipeline', 'status': 'pending'}]

## Assistant

```json
{
  "id": "toolu_0114GBomhtvHeR7Kvx52R4Bm",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
    "old_string": "---\n\n### Dashboard view (`views/dashboard.js`)",
    "new_string": "---\n\n### Project Detail view (`views/project-detail.js`)\n\nThe project detail view is rendered at `#/projects/:id`. It fetches all required data in **parallel** (project record, workspace list, and global repository list via `Promise.all`) before rendering the page.\n\n#### Sections rendered\n\n| Section | Description |\n|---------|-------------|\n| **Metadata** | Project ID and name as a page heading; inline description editor. Clicking **Edit Description** reveals a textarea; **Save** calls `PUT /api/projects/:id`; **Cancel** restores the read-mode display. |\n| **Repositories** | Lists repositories currently in the project. Each row shows name and ID, plus a **Remove** button (requires confirmation dialog). An **Add Repository** select picker lists only repos not already in the project and calls `POST /api/projects/:id/repositories`. When all global repositories are already added, the picker is replaced by an informational message. |\n| **Workspaces** | Table of all workspaces (ID, description, creation date, actions). Each workspace ID is a link to `#/projects/:id/workspaces/:wid`. The **STABLE** workspace's Delete button is visually disabled and non-functional \u2014 the `disabled` attribute and `btn-disabled` CSS class are applied, and the button carries a `title` tooltip explaining the restriction. Non-STABLE workspaces can be deleted after confirmation. |\n| **Add Workspace** | Collapsible form toggled by **+ Add Workspace**. Validates workspace ID against `/^[A-Z]{2,6}$/` (2\u20136 uppercase letters, no digits or special characters) before calling `POST /api/projects/:id/workspaces`. |\n| **Danger Zone** | Two actions: **Rename Project** (calls `PUT /api/projects/:id/rename`, then navigates to `#/projects/:newId`) and **Delete Project** (calls `DELETE /api/projects/:id`, then navigates to `#/`). Both require confirmation dialogs. Rename is client-side-guarded against identical IDs. |\n\n#### Refresh strategy\n\nAfter any successful mutation (add/remove repository, add/delete workspace), the view re-renders itself by calling `renderProjectDetail` recursively (`refresh()`). This **full-refresh-on-mutation** approach re-issues all three parallel API calls (`GET /api/projects/:id`, `GET /api/projects/:id/workspaces`, `GET /api/repositories`) and rebuilds the entire DOM from scratch. This is intentional: it guarantees UI consistency without stateful diffing and is correct for current usage scale. A targeted section re-render (e.g. refreshing only the workspace list) is a deferred optimisation.\n\n#### Router injection\n\n`project-detail.js` exports a `setRouter(router)` function (in addition to `renderProjectDetail`) so that it can call `router.navigate()` on rename and delete without creating a circular dependency with `app.js`. `app.js` calls `setProjectDetailRouter(router)` (aliased from `setRouter`) **before** calling `router.start()`.\n\nThe injected `_router` reference is null-guarded in three places \u2014 the back-link handler, the workspace link handler, and post-rename/post-delete navigation \u2014 so the view remains usable in test contexts where no router is injected.\n\n#### Key casing\n\nThe Go backend returns project and workspace fields with capitalised keys (`Id`, `Name`, `Repositories`, etc.). `project-detail.js` normalises both forms via three internal helpers: `normaliseProject()`, `normaliseRepo()`, and `normaliseWorkspace()`. See the [Normalisation helpers note](#normalisation-helpers-note) below.\n\n---\n\n### Dashboard view (`views/dashboard.js`)"
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_0114GBomhtvHeR7Kvx52R4Bm`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
  "old_string": "---\n\n### Dashboard view (`views/dashboard.js`)",
  "new_string": "---\n\n### Project Detail view (`views/project-detail.js`)\n\nThe project detail view is rendered at `#/projects/:id`. It fetches all required data in **parallel** (project record, workspace list, and global repository list via `Promise.all`) before rendering the page.\n\n#### Sections rendered\n\n| Section | Description |\n|---------|-------------|\n| **Metadata** | Project ID and name as a page heading; inline description editor. Clicking **Edit Description** reveals a textarea; **Save** calls `PUT /api/projects/:id`; **Cancel** restores the read-mode display. |\n| **Repositories** | Lists repositories currently in the project. Each row shows name and ID, plus a **Remove** button (requires confirmation dialog). An **Add Repository** select picker lists only repos not already in the project and calls `POST /api/projects/:id/repositories`. When all global repositories are already added, the picker is replaced by an informational message. |\n| **Workspaces** | Table of all workspaces (ID, description, creation date, actions). Each workspace ID is a link to `#/projects/:id/workspaces/:wid`. The **STABLE** workspace's Delete button is visually disabled and non-functional \u2014 the `disabled` attribute and `btn-disabled` CSS class are applied, and the button carries a `title` tooltip explaining the restriction. Non-STABLE workspaces can be deleted after confirmation. |\n| **Add Workspace** | Collapsible form toggled by **+ Add Workspace**. Validates workspace ID against `/^[A-Z]{2,6}$/` (2\u20136 uppercase letters, no digits or special characters) before calling `POST /api/projects/:id/workspaces`. |\n| **Danger Zone** | Two actions: **Rename Project** (calls `PUT /api/projects/:id/rename`, then navigates to `#/projects/:newId`) and **Delete Project** (calls `DELETE /api/projects/:id`, then navigates to `#/`). Both require confirmation dialogs. Rename is client-side-guarded against identical IDs. |\n\n#### Refresh strategy\n\nAfter any successful mutation (add/remove repository, add/delete workspace), the view re-renders itself by calling `renderProjectDetail` recursively (`refresh()`). This **full-refresh-on-mutation** approach re-issues all three parallel API calls (`GET /api/projects/:id`, `GET /api/projects/:id/workspaces`, `GET /api/repositories`) and rebuilds the entire DOM from scratch. This is intentional: it guarantees UI consistency without stateful diffing and is correct for current usage scale. A targeted section re-render (e.g. refreshing only the workspace list) is a deferred optimisation.\n\n#### Router injection\n\n`project-detail.js` exports a `setRouter(router)` function (in addition to `renderProjectDetail`) so that it can call `router.navigate()` on rename and delete without creating a circular dependency with `app.js`. `app.js` calls `setProjectDetailRouter(router)` (aliased from `setRouter`) **before** calling `router.start()`.\n\nThe injected `_router` reference is null-guarded in three places \u2014 the back-link handler, the workspace link handler, and post-rename/post-delete navigation \u2014 so the view remains usable in test contexts where no router is injected.\n\n#### Key casing\n\nThe Go backend returns project and workspace fields with capitalised keys (`Id`, `Name`, `Repositories`, etc.). `project-detail.js` normalises both forms via three internal helpers: `normaliseProject()`, `normaliseRepo()`, and `normaliseWorkspace()`. See the [Normalisation helpers note](#normalisation-helpers-note) below.\n\n---\n\n### Dashboard view (`views/dashboard.js`)"
}
```

## Tool Result

Successfully replaced 1 instance(s) of the string in '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md'

## Assistant

Now add the normalisation helpers note at the end of the GUI Frontend section (after the form-helpers section):

```json
{
  "id": "toolu_018FFN7KVHdQp7MVXw431iRt",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
    "offset": 940,
    "limit": 20
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_018FFN7KVHdQp7MVXw431iRt`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
  "offset": 940,
  "limit": 20
}
```

## Tool Result

Error: Line offset 940 exceeds file length (925 lines)

## Assistant

```json
{
  "id": "toolu_01XxCmV7hd3Hw6M43UAdP4c4",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
    "offset": 910,
    "limit": 15
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01XxCmV7hd3Hw6M43UAdP4c4`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
  "offset": 910,
  "limit": 15
}
```

## Tool Result

   911	| `id` | `string` | auto | Override the auto-generated element ID. |
   912	
   913	##### `validateRequired(form, fields)`
   914	
   915	Checks that each named field in `form` is non-empty.
   916	
   917	- Clears all existing `.field-error` inline error elements before re-validating (prevents stale errors on repeated submissions).
   918	- For each empty field: adds `class="error"` to the control and inserts an inline `.field-error` `<span>` below it.
   919	- Input controls also get an `'input'` listener that clears the error as soon as the user starts typing.
   920	- Returns `true` if all listed fields are non-empty, `false` otherwise.
   921	
   922	| Parameter | Type | Description |
   923	|-----------|------|-------------|
   924	| `form` | `HTMLFormElement` | The form to validate. |
   925	| `fields` | `string[]` | Array of `name` attribute values to check. |

## Assistant

```json
{
  "id": "toolu_018zvRE32JEypLRUF1P5QXti",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
    "old_string": "| Parameter | Type | Description |\n|-----------|------|-------------|\n| `form` | `HTMLFormElement` | The form to validate. |\n| `fields` | `string[]` | Array of `name` attribute values to check. |",
    "new_string": "| Parameter | Type | Description |\n|-----------|------|-------------|\n| `form` | `HTMLFormElement` | The form to validate. |\n| `fields` | `string[]` | Array of `name` attribute values to check. |\n\n---\n\n### Normalisation helpers note\n\nThe Go backend serialises object fields with **capitalised keys** (`Id`, `Name`, `Url`, `Repositories`, `Description`, `CreatedAt`, etc.). All three view modules normalise these before use:\n\n| Helper | Module | Fields normalised |\n|--------|--------|-------------------|\n| `normaliseProject(project)` | `project-detail.js` | `id`, `name`, `description`, `repositories` |\n| `normaliseRepo(repo)` | `repositories.js`, `project-detail.js` | `id`, `name`, `url` |\n| `normaliseWorkspace(ws)` | `project-detail.js` | `id`, `description`, `createdAt` |\n\n> **Known duplication:** `normaliseRepo()` is defined verbatim in both `repositories.js` (line 30) and `project-detail.js` (line 67). Both handle Go-capitalised and lowercase keys identically. If a third view requires the same helper, it should be extracted to a shared module \u2014 the suggested location is `gui/public/js/utils/normalise.js`. No consolidation has been done yet; the two copies are kept in-place to avoid premature abstraction."
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_018zvRE32JEypLRUF1P5QXti`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md",
  "old_string": "| Parameter | Type | Description |\n|-----------|------|-------------|\n| `form` | `HTMLFormElement` | The form to validate. |\n| `fields` | `string[]` | Array of `name` attribute values to check. |",
  "new_string": "| Parameter | Type | Description |\n|-----------|------|-------------|\n| `form` | `HTMLFormElement` | The form to validate. |\n| `fields` | `string[]` | Array of `name` attribute values to check. |\n\n---\n\n### Normalisation helpers note\n\nThe Go backend serialises object fields with **capitalised keys** (`Id`, `Name`, `Url`, `Repositories`, `Description`, `CreatedAt`, etc.). All three view modules normalise these before use:\n\n| Helper | Module | Fields normalised |\n|--------|--------|-------------------|\n| `normaliseProject(project)` | `project-detail.js` | `id`, `name`, `description`, `repositories` |\n| `normaliseRepo(repo)` | `repositories.js`, `project-detail.js` | `id`, `name`, `url` |\n| `normaliseWorkspace(ws)` | `project-detail.js` | `id`, `description`, `createdAt` |\n\n> **Known duplication:** `normaliseRepo()` is defined verbatim in both `repositories.js` (line 30) and `project-detail.js` (line 67). Both handle Go-capitalised and lowercase keys identically. If a third view requires the same helper, it should be extracted to a shared module \u2014 the suggested location is `gui/public/js/utils/normalise.js`. No consolidation has been done yet; the two copies are kept in-place to avoid premature abstraction."
}
```

## Tool Result

Successfully replaced 1 instance(s) of the string in '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md'

## Assistant

```json
{
  "id": "toolu_01BcgyUxAAW3Wqq3HrXXmvXD",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Add Project Detail view section to README.md (after Dashboard view section)",
        "status": "completed"
      },
      {
        "content": "Update project-detail.js module-level JSDoc to document the refresh/full-refresh-on-mutation strategy",
        "status": "in_progress"
      },
      {
        "content": "Complete the documentation pipeline via ledger_complete_pipeline",
        "status": "pending"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_01BcgyUxAAW3Wqq3HrXXmvXD`)

```json
{
  "todos": [
    {
      "content": "Add Project Detail view section to README.md (after Dashboard view section)",
      "status": "completed"
    },
    {
      "content": "Update project-detail.js module-level JSDoc to document the refresh/full-refresh-on-mutation strategy",
      "status": "in_progress"
    },
    {
      "content": "Complete the documentation pipeline via ledger_complete_pipeline",
      "status": "pending"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Add Project Detail view section to README.md (after Dashboard view section)', 'status': 'completed'}, {'content': 'Update project-detail.js module-level JSDoc to document the refresh/full-refresh-on-mutation strategy', 'status': 'in_progress'}, {'content': 'Complete the documentation pipeline via ledger_complete_pipeline', 'status': 'pending'}]

## Assistant

Now update the module-level JSDoc in `project-detail.js` to document the refresh/full-refresh-on-mutation strategy:

```json
{
  "id": "toolu_01Vyr9cRYbLkuEqHim4vXedE",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
    "old_string": "/**\n * Project Detail View \u2014 Repo Parallelizer GUI.\n *\n * Renders the full detail page for a single project:\n *   - Project metadata (ID, name/description with inline description edit).\n *   - Repositories section: list with per-repo Remove, plus \"Add Repository\" picker.\n *   - Workspaces section: list with links, per-workspace Delete (STABLE disabled),\n *     and \"Add Workspace\" form.\n *   - Rename Project action (changes project ID).\n *   - Delete Project action.\n *\n * @param {HTMLElement} container - The `#app` root element supplied by the router.\n * @param {Object}      params    - Route params \u2014 expects `params.id` (project ID).\n */",
    "new_string": "/**\n * Project Detail View \u2014 Repo Parallelizer GUI.\n *\n * Renders the full detail page for a single project:\n *   - Project metadata (ID, name/description with inline description edit).\n *   - Repositories section: list with per-repo Remove, plus \"Add Repository\" picker.\n *   - Workspaces section: list with links, per-workspace Delete (STABLE disabled),\n *     and \"Add Workspace\" form.\n *   - Rename Project action (changes project ID).\n *   - Delete Project action.\n *\n * ## Data fetching\n *\n * On render, `GET /api/projects/:id`, `GET /api/projects/:id/workspaces`, and\n * `GET /api/repositories` are issued in parallel via `Promise.all`. A loading\n * spinner is shown until all three resolve.\n *\n * ## Refresh strategy (full-refresh-on-mutation)\n *\n * After any successful mutation (add/remove repository, add/delete workspace),\n * the view re-renders itself completely by calling `renderProjectDetail`\n * recursively via the internal `refresh()` helper. This triggers three new\n * parallel API calls and rebuilds the full DOM from scratch.\n *\n * Trade-off: simplicity and guaranteed consistency over efficiency. For the\n * current usage scale this is the right default. A targeted section re-render\n * (e.g. refreshing only the repository list) is a deferred optimisation \u2014\n * it would save two redundant requests per mutation but adds stateful diffing\n * complexity.\n *\n * ## Router injection\n *\n * This module exports `setRouter(router)` so that `renderProjectDetail` can\n * call `router.navigate()` on rename and delete without creating a circular\n * import with `app.js`. `app.js` calls `setProjectDetailRouter(router)` (the\n * aliased import) before `router.start()`. The `_router` variable is\n * null-guarded in all three navigation sites so the view remains functional\n * in test contexts where no router is injected.\n *\n * @module project-detail\n */"
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_01Vyr9cRYbLkuEqHim4vXedE`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js",
  "old_string": "/**\n * Project Detail View \u2014 Repo Parallelizer GUI.\n *\n * Renders the full detail page for a single project:\n *   - Project metadata (ID, name/description with inline description edit).\n *   - Repositories section: list with per-repo Remove, plus \"Add Repository\" picker.\n *   - Workspaces section: list with links, per-workspace Delete (STABLE disabled),\n *     and \"Add Workspace\" form.\n *   - Rename Project action (changes project ID).\n *   - Delete Project action.\n *\n * @param {HTMLElement} container - The `#app` root element supplied by the router.\n * @param {Object}      params    - Route params \u2014 expects `params.id` (project ID).\n */",
  "new_string": "/**\n * Project Detail View \u2014 Repo Parallelizer GUI.\n *\n * Renders the full detail page for a single project:\n *   - Project metadata (ID, name/description with inline description edit).\n *   - Repositories section: list with per-repo Remove, plus \"Add Repository\" picker.\n *   - Workspaces section: list with links, per-workspace Delete (STABLE disabled),\n *     and \"Add Workspace\" form.\n *   - Rename Project action (changes project ID).\n *   - Delete Project action.\n *\n * ## Data fetching\n *\n * On render, `GET /api/projects/:id`, `GET /api/projects/:id/workspaces`, and\n * `GET /api/repositories` are issued in parallel via `Promise.all`. A loading\n * spinner is shown until all three resolve.\n *\n * ## Refresh strategy (full-refresh-on-mutation)\n *\n * After any successful mutation (add/remove repository, add/delete workspace),\n * the view re-renders itself completely by calling `renderProjectDetail`\n * recursively via the internal `refresh()` helper. This triggers three new\n * parallel API calls and rebuilds the full DOM from scratch.\n *\n * Trade-off: simplicity and guaranteed consistency over efficiency. For the\n * current usage scale this is the right default. A targeted section re-render\n * (e.g. refreshing only the repository list) is a deferred optimisation \u2014\n * it would save two redundant requests per mutation but adds stateful diffing\n * complexity.\n *\n * ## Router injection\n *\n * This module exports `setRouter(router)` so that `renderProjectDetail` can\n * call `router.navigate()` on rename and delete without creating a circular\n * import with `app.js`. `app.js` calls `setProjectDetailRouter(router)` (the\n * aliased import) before `router.start()`. The `_router` variable is\n * null-guarded in all three navigation sites so the view remains functional\n * in test contexts where no router is injected.\n *\n * @module project-detail\n */"
}
```

## Tool Result

Successfully replaced 1 instance(s) of the string in '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/project-detail.js'

## Assistant

```json
{
  "id": "toolu_01SqdUYvi2jt4ShuktFxvbKS",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Add Project Detail view section to README.md (after Dashboard view section)",
        "status": "completed"
      },
      {
        "content": "Update project-detail.js module-level JSDoc to document the refresh/full-refresh-on-mutation strategy",
        "status": "completed"
      },
      {
        "content": "Complete the documentation pipeline via ledger_complete_pipeline",
        "status": "in_progress"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_01SqdUYvi2jt4ShuktFxvbKS`)

```json
{
  "todos": [
    {
      "content": "Add Project Detail view section to README.md (after Dashboard view section)",
      "status": "completed"
    },
    {
      "content": "Update project-detail.js module-level JSDoc to document the refresh/full-refresh-on-mutation strategy",
      "status": "completed"
    },
    {
      "content": "Complete the documentation pipeline via ledger_complete_pipeline",
      "status": "in_progress"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Add Project Detail view section to README.md (after Dashboard view section)', 'status': 'completed'}, {'content': 'Update project-detail.js module-level JSDoc to document the refresh/full-refresh-on-mutation strategy', 'status': 'completed'}, {'content': 'Complete the documentation pipeline via ledger_complete_pipeline', 'status': 'in_progress'}]

## Assistant

```json
{
  "id": "toolu_01EqJ85P7cTnsCKtqeZ74kZh",
  "caller": {
    "type": "direct"
  },
  "input": {
    "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend",
    "work_package_id": "WP-014",
    "type": "documentation",
    "agent_role": "Documentation",
    "status": "PASS",
    "summary": [
      "Added a full 'Project Detail view' section to README.md (after the Dashboard view section), documenting: parallel data-fetching strategy, all rendered sections (metadata, repositories, workspaces, add-workspace form, danger zone), the full-refresh-on-mutation strategy, router injection pattern, and key-casing normalisation.",
      "Added a 'Normalisation helpers note' section to README.md documenting the normaliseProject/normaliseRepo/normaliseWorkspace helpers across view modules, and flagging the known normaliseRepo duplication between repositories.js and project-detail.js with the suggested future consolidation path (gui/public/js/utils/normalise.js).",
      "Updated the module-level JSDoc in gui/public/js/views/project-detail.js to document the data-fetching strategy, the full-refresh-on-mutation design (and its trade-off), and the router injection pattern \u2014 addressing both documentation-forward items from the code review pipeline."
    ],
    "artifacts": {
      "files_modified": [
        "README.md",
        "gui/public/js/views/project-detail.js"
      ]
    },
    "acceptance_criteria_updates": [
      {
        "criterion": "Navigating to #/projects/:id fetches the project via GET /api/projects/:id and displays the project's metadata (ID, name, description), repositories list, and workspaces list.",
        "met": true
      },
      {
        "criterion": "Adding a repository presents a selection from the global repository list (filtered to exclude already-added repos) and calls POST /api/projects/:id/repositories with { repositoryId }.",
        "met": true
      },
      {
        "criterion": "Removing a repository from the project shows a confirmation dialog and calls DELETE /api/projects/:id/repositories/:repoId on confirm.",
        "met": true
      },
      {
        "criterion": "The 'Add Workspace' form validates the workspace ID format (2-6 uppercase A-Z letters) and creates the workspace via POST /api/projects/:id/workspaces.",
        "met": true
      },
      {
        "criterion": "The STABLE workspace's delete button is visually disabled and non-functional.",
        "met": true
      },
      {
        "criterion": "Renaming a project shows a confirmation dialog, calls PUT /api/projects/:id/rename with { newId }, and navigates to #/projects/:newId on success.",
        "met": true
      }
    ],
    "comments": [
      {
        "type": "improvement",
        "priority": "medium",
        "note": "[documentation-forward addressed] Added 'Normalisation helpers note' section to README.md documenting the normaliseRepo duplication between repositories.js (line 30) and project-detail.js (line 67), with the suggested consolidation path (gui/public/js/utils/normalise.js). Future developers adding a third view are now explicitly guided to that location rather than creating a third copy."
      },
      {
        "type": "improvement",
        "priority": "low",
        "note": "[documentation-forward addressed] Updated the module-level JSDoc in project-detail.js with a dedicated 'Refresh strategy' subsection explaining the full-refresh-on-mutation design, its trade-off (simplicity/consistency over efficiency), and flagging targeted section re-render as the deferred optimisation path. Also added 'Data fetching' and 'Router injection' subsections, and a @module tag."
      },
      {
        "type": "improvement",
        "priority": "low",
        "note": "No changes were needed to the existing Dashboard, API client, or Component sections \u2014 they accurately reflected their implementations. The existing route registry table already listed the #/projects/:id route; the new Project Detail section provides the full narrative for that view."
      }
    ],
    "handoff_notes": "Documentation PASS for WP-014. README.md now has a full 'Project Detail view' section and a 'Normalisation helpers note'. project-detail.js module JSDoc updated with refresh strategy, data-fetching, and router-injection documentation. Both reviewer documentation-forward items addressed."
  },
  "name": "ledger_complete_pipeline",
  "type": "tool_use"
}
```

**Tool call:** `ledger_complete_pipeline` (id: `toolu_01EqJ85P7cTnsCKtqeZ74kZh`)

```json
{
  "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend",
  "work_package_id": "WP-014",
  "type": "documentation",
  "agent_role": "Documentation",
  "status": "PASS",
  "summary": [
    "Added a full 'Project Detail view' section to README.md (after the Dashboard view section), documenting: parallel data-fetching strategy, all rendered sections (metadata, repositories, workspaces, add-workspace form, danger zone), the full-refresh-on-mutation strategy, router injection pattern, and key-casing normalisation.",
    "Added a 'Normalisation helpers note' section to README.md documenting the normaliseProject/normaliseRepo/normaliseWorkspace helpers across view modules, and flagging the known normaliseRepo duplication between repositories.js and project-detail.js with the suggested future consolidation path (gui/public/js/utils/normalise.js).",
    "Updated the module-level JSDoc in gui/public/js/views/project-detail.js to document the data-fetching strategy, the full-refresh-on-mutation design (and its trade-off), and the router injection pattern \u2014 addressing both documentation-forward items from the code review pipeline."
  ],
  "artifacts": {
    "files_modified": [
      "README.md",
      "gui/public/js/views/project-detail.js"
    ]
  },
  "acceptance_criteria_updates": [
    {
      "criterion": "Navigating to #/projects/:id fetches the project via GET /api/projects/:id and displays the project's metadata (ID, name, description), repositories list, and workspaces list.",
      "met": true
    },
    {
      "criterion": "Adding a repository presents a selection from the global repository list (filtered to exclude already-added repos) and calls POST /api/projects/:id/repositories with { repositoryId }.",
      "met": true
    },
    {
      "criterion": "Removing a repository from the project shows a confirmation dialog and calls DELETE /api/projects/:id/repositories/:repoId on confirm.",
      "met": true
    },
    {
      "criterion": "The 'Add Workspace' form validates the workspace ID format (2-6 uppercase A-Z letters) and creates the workspace via POST /api/projects/:id/workspaces.",
      "met": true
    },
    {
      "criterion": "The STABLE workspace's delete button is visually disabled and non-functional.",
      "met": true
    },
    {
      "criterion": "Renaming a project shows a confirmation dialog, calls PUT /api/projects/:id/rename with { newId }, and navigates to #/projects/:newId on success.",
      "met": true
    }
  ],
  "comments": [
    {
      "type": "improvement",
      "priority": "medium",
      "note": "[documentation-forward addressed] Added 'Normalisation helpers note' section to README.md documenting the normaliseRepo duplication between repositories.js (line 30) and project-detail.js (line 67), with the suggested consolidation path (gui/public/js/utils/normalise.js). Future developers adding a third view are now explicitly guided to that location rather than creating a third copy."
    },
    {
      "type": "improvement",
      "priority": "low",
      "note": "[documentation-forward addressed] Updated the module-level JSDoc in project-detail.js with a dedicated 'Refresh strategy' subsection explaining the full-refresh-on-mutation design, its trade-off (simplicity/consistency over efficiency), and flagging targeted section re-render as the deferred optimisation path. Also added 'Data fetching' and 'Router injection' subsections, and a @module tag."
    },
    {
      "type": "improvement",
      "priority": "low",
      "note": "No changes were needed to the existing Dashboard, API client, or Component sections \u2014 they accurately reflected their implementations. The existing route registry table already listed the #/projects/:id route; the new Project Detail section provides the full narrative for that view."
    }
  ],
  "handoff_notes": "Documentation PASS for WP-014. README.md now has a full 'Project Detail view' section and a 'Normalisation helpers note'. project-detail.js module JSDoc updated with refresh strategy, data-fetching, and router-injection documentation. Both reviewer documentation-forward items addressed."
}
```

## Tool Result

{
  "work_package_id": "WP-014",
  "work_package_file": "work/WP-005.md",
  "status": "COMPLETE",
  "assigned_to": "Documentation",
  "dependencies": [
    "WP-011",
    "WP-013"
  ],
  "acceptance_criteria": [
    {
      "criterion": "Navigating to #/projects/:id fetches the project via GET /api/projects/:id and displays the project's metadata (ID, name, description), repositories list, and workspaces list.",
      "met": true
    },
    {
      "criterion": "Adding a repository presents a selection from the global repository list (filtered to exclude already-added repos) and calls POST /api/projects/:id/repositories with { repositoryId }.",
      "met": true
    },
    {
      "criterion": "Removing a repository from the project shows a confirmation dialog and calls DELETE /api/projects/:id/repositories/:repoId on confirm.",
      "met": true
    },
    {
      "criterion": "The 'Add Workspace' form validates the workspace ID format (2-6 uppercase A-Z letters) and creates the workspace via POST /api/projects/:id/workspaces.",
      "met": true
    },
    {
      "criterion": "The STABLE workspace's delete button is visually disabled and non-functional.",
      "met": true
    },
    {
      "criterion": "Renaming a project shows a confirmation dialog, calls PUT /api/projects/:id/rename with { newId }, and navigates to #/projects/:newId on success.",
      "met": true
    }
  ],
  "active_pipeline_stages": [
    "implementation",
    "qa",
    "code-review",
    "documentation"
  ],
  "revision": 0,
  "pipelines": [
    {
      "type": "implementation",
      "status": "PASS",
      "started_at": "2026-04-07T20:55:45Z",
      "completed_at": "2026-04-07T20:57:18Z",
      "duration_ms": 93000,
      "summary": [
        "Verified full implementation of gui/public/js/views/project-detail.js (901 lines) covering all AC.",
        "Verified app.js correctly imports renderProjectDetail and setRouter, registers #/projects/:id route, and calls setProjectDetailRouter(router) before router.start().",
        "Ran the pre-existing qa-wp014-tests.mjs test suite: 53/53 tests passed with exit code 0.",
        "All six acceptance criteria verified by static analysis and runtime tests."
      ],
      "artifacts": {
        "files_modified": [
          "gui/public/js/views/project-detail.js",
          "gui/public/js/app.js"
        ]
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T20:57:18Z",
          "note": "project-detail.js: The `refresh()` helper re-renders the whole view by calling renderProjectDetail recursively. For mutation-heavy workflows (e.g., adding several repos in sequence) this triggers three parallel API calls each time. A targeted section re-render (only repos or only workspaces) would be more efficient, but is a deferred optimisation — the current full-refresh approach is correct and readable."
        },
        {
          "type": "refactor",
          "priority": "low",
          "timestamp": "2026-04-07T20:57:18Z",
          "note": "project-detail.js: The normalise* functions (normaliseProject, normaliseRepo, normaliseWorkspace) handle both Go-capitalised and lowercase keys. These patterns are repeated across multiple view files. A shared normalisation utility in a separate module (e.g., gui/public/js/utils/normalise.js) would reduce duplication if more views need the same treatment."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T20:57:18Z",
          "note": "project-detail.js buildMetaSection: The cancelDescBtn click handler sets editRow.hidden to false before setting it to true on the next line (lines 197-200). The first assignment is dead code and can be removed for clarity."
        },
        {
          "type": "convention",
          "priority": "low",
          "timestamp": "2026-04-07T20:57:18Z",
          "note": "project-detail.js: The module-level `_router` variable uses a leading underscore by convention to signal 'private', which is a reasonable pattern. However app.js also exports setRouter from dashboard.js under the same alias — the naming is consistent across all views and is not a problem, just worth noting for documentation."
        }
      ]
    },
    {
      "type": "qa",
      "status": "PASS",
      "started_at": "2026-04-07T20:59:54Z",
      "completed_at": "2026-04-07T21:01:53Z",
      "duration_ms": 119000,
      "summary": [
        "All 53 tests in qa-wp014-tests.mjs pass (exit code 0).",
        "Regression: WP-011 (73 tests), WP-013 (54 tests), WP-015 (40 tests) all pass — no regressions introduced.",
        "4 additional manual edge cases all pass: null Repositories field (graceful empty state), cancelDescBtn dead-code line (UI state is correct despite redundant assignment on line 197), missing createdAt renders '—', renaming to same ID is blocked client-side without calling the API.",
        "AC1-AC6 all verified by runtime DOM tests and static analysis.",
        "app.js correctly imports renderProjectDetail and setRouter (as setProjectDetailRouter), calls setProjectDetailRouter(router) before router.start(), and registers the #/projects/:id route."
      ],
      "metrics": {
        "test_coverage": "53/53 suite tests + 4 manual edge cases + 167 regression tests across WP-011, WP-013, WP-015",
        "tests_passed": 53,
        "tests_failed": 0
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:01:53Z",
          "note": "project-detail.js line 197: cancelDescBtn handler sets editRow.hidden = false before immediately setting it to true on line 199. The first assignment is dead code and has no visible effect on runtime behaviour (confirmed by manual edge-case test). Low risk — safe to remove for clarity in a follow-up clean-up."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:01:53Z",
          "note": "The rename guard (newId === project.id) correctly prevents calling the API when the same ID is entered — confirmed by manual edge-case test. Good defensive UX."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:01:53Z",
          "note": "Workspace links use encodeURIComponent() for both projectId and workspaceId — correct for IDs that may contain spaces or special characters. No issues found."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:01:53Z",
          "note": "No cross-module regressions: all 167 tests across WP-011 (shared components), WP-013 (dashboard/router), and WP-015 (repositories view) continue to pass after this implementation."
        }
      ]
    },
    {
      "type": "code-review",
      "status": "PASS",
      "started_at": "2026-04-07T21:04:46Z",
      "completed_at": "2026-04-07T21:05:56Z",
      "duration_ms": 70000,
      "summary": [
        "Code review PASS for WP-014 (Project Detail View). Reviewed project-detail.js (901 lines) and app.js across all four dimensions: maintainability, best practices, performance, and future alignment. One Fix-Forward applied (dead assignment in cancelDescBtn handler, lines 197–199). Two Documentation-Forward items tagged. No blocking issues found."
      ],
      "artifacts": {
        "files_modified": [
          "gui/public/js/views/project-detail.js",
          "gui/public/js/app.js"
        ]
      },
      "comments": [
        {
          "type": "reviewer-applied-fix",
          "priority": "low",
          "timestamp": "2026-04-07T21:05:56Z",
          "note": "project-detail.js lines 196–200 (cancelDescBtn handler): Removed dead assignment `editRow.hidden = false` that was immediately overwritten by `editRow.hidden = true` on the next line. The redundant assignment had no behavioral effect (QA-confirmed), but created a misleading read. The handler now reads: `descRow.hidden = false; editRow.hidden = true;` — clear and correct."
        },
        {
          "type": "documentation-forward",
          "priority": "medium",
          "timestamp": "2026-04-07T21:05:56Z",
          "note": "[documentation-forward] The `normaliseRepo()` function is duplicated verbatim in both `repositories.js` (line 30) and `project-detail.js` (line 67). Both handle Go-capitalised vs lowercase keys identically. The README or architecture doc should note this as a known duplication and flag `gui/public/js/utils/normalise.js` as the suggested location for a future consolidation — so the next developer adding a view does not create a third copy."
        },
        {
          "type": "documentation-forward",
          "priority": "low",
          "timestamp": "2026-04-07T21:05:56Z",
          "note": "[documentation-forward] The `refresh()` pattern (recursive call to `renderProjectDetail` on every mutation) is used in project-detail.js. This is intentional and correct for current scale, but the module-level docstring at the top of the file does not mention it. A brief note in the module JSDoc explaining the full-refresh-on-mutation strategy (and the trade-off against targeted section re-renders) would help future contributors understand the design intent without reading the implementation."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:05:56Z",
          "note": "app.js: The `setRouter` alias import (`setRouter as setProjectDetailRouter`) is clean and avoids a naming collision with dashboard.js's `setRouter`. The inline comment on line 18 adequately explains the aliasing. No changes needed — recording as a positive pattern for consistency."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:05:56Z",
          "note": "project-detail.js: The `_router` null-guard pattern (checking `if (_router)` before calling `.navigate()`) is applied consistently in three places: backLink click handler, workspace link click handler, and rename/delete navigation. This is correct defensive coding — the view remains usable even if called without router injection (e.g., in test contexts). No changes needed."
        }
      ]
    },
    {
      "type": "documentation",
      "status": "PASS",
      "started_at": "2026-04-07T21:07:44Z",
      "completed_at": "2026-04-07T21:09:55Z",
      "duration_ms": 131000,
      "summary": [
        "Added a full 'Project Detail view' section to README.md (after the Dashboard view section), documenting: parallel data-fetching strategy, all rendered sections (metadata, repositories, workspaces, add-workspace form, danger zone), the full-refresh-on-mutation strategy, router injection pattern, and key-casing normalisation.",
        "Added a 'Normalisation helpers note' section to README.md documenting the normaliseProject/normaliseRepo/normaliseWorkspace helpers across view modules, and flagging the known normaliseRepo duplication between repositories.js and project-detail.js with the suggested future consolidation path (gui/public/js/utils/normalise.js).",
        "Updated the module-level JSDoc in gui/public/js/views/project-detail.js to document the data-fetching strategy, the full-refresh-on-mutation design (and its trade-off), and the router injection pattern — addressing both documentation-forward items from the code review pipeline."
      ],
      "artifacts": {
        "files_modified": [
          "README.md",
          "gui/public/js/views/project-detail.js"
        ]
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "medium",
          "timestamp": "2026-04-07T21:09:55Z",
          "note": "[documentation-forward addressed] Added 'Normalisation helpers note' section to README.md documenting the normaliseRepo duplication between repositories.js (line 30) and project-detail.js (line 67), with the suggested consolidation path (gui/public/js/utils/normalise.js). Future developers adding a third view are now explicitly guided to that location rather than creating a third copy."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:09:55Z",
          "note": "[documentation-forward addressed] Updated the module-level JSDoc in project-detail.js with a dedicated 'Refresh strategy' subsection explaining the full-refresh-on-mutation design, its trade-off (simplicity/consistency over efficiency), and flagging targeted section re-render as the deferred optimisation path. Also added 'Data fetching' and 'Router injection' subsections, and a @module tag."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:09:55Z",
          "note": "No changes were needed to the existing Dashboard, API client, or Component sections — they accurately reflected their implementations. The existing route registry table already listed the #/projects/:id route; the new Project Detail section provides the full narrative for that view."
        }
      ]
    }
  ],
  "status_changed_at": "2026-04-07T21:09:55Z",
  "last_updated": "2026-04-07T21:09:55Z",
  "handoff_notes": [
    {
      "from_agent": "Developer",
      "to_agent": "QA",
      "timestamp": "2026-04-07T20:57:18Z",
      "notes": [
        "WP-014 (Project Detail View) implementation is fully verified. All 53 QA tests in qa-wp014-tests.mjs pass. The view implements: parallel data fetching with loading state, project metadata with inline description editing, repository section with filtered add-picker and per-repo remove+confirm, workspaces section with STABLE-disabled delete and validated Add Workspace form (/^[A-Z]{2,6}$/), rename with confirmation+navigation, and delete with confirmation+navigation. One minor dead-code line exists in buildMetaSection cancelDescBtn handler (low priority, noted in comments)."
      ]
    },
    {
      "from_agent": "QA",
      "to_agent": "Reviewer",
      "timestamp": "2026-04-07T21:01:53Z",
      "notes": [
        "WP-014 (Project Detail View) passes QA. 53/53 suite tests + 4 manual edge cases pass. All 6 AC verified. Notable for reviewer: (1) cancelDescBtn dead-code on line 197 (editRow.hidden = false immediately overwritten — low priority clean-up only); (2) normalise* helpers are duplicated across view files — candidate for a shared utility module in a future refactor; (3) refresh() re-renders the full view on every mutation — acceptable for current scale. No functional issues found."
      ]
    },
    {
      "from_agent": "Reviewer",
      "to_agent": "Documentation",
      "timestamp": "2026-04-07T21:05:56Z",
      "notes": [
        "WP-014 code review PASS. One Fix-Forward applied: removed dead assignment in cancelDescBtn handler (project-detail.js). Two documentation-forward items for the Documentation agent: (1) note normaliseRepo duplication across repositories.js and project-detail.js, recommend future consolidation into utils/normalise.js; (2) add module-level note explaining the full-refresh-on-mutation strategy in project-detail.js. All 6 AC confirmed met. No blocking issues."
      ]
    },
    {
      "from_agent": "Documentation",
      "to_agent": "Synthesis",
      "timestamp": "2026-04-07T21:09:55Z",
      "notes": [
        "Documentation PASS for WP-014. README.md now has a full 'Project Detail view' section and a 'Normalisation helpers note'. project-detail.js module JSDoc updated with refresh strategy, data-fetching, and router-injection documentation. Both reviewer documentation-forward items addressed."
      ]
    }
  ],
  "auto_finalized": true
}

--- NEXT STEP ---
Pipeline PASS. WP WP-014 was auto-finalized to COMPLETE (all acceptance criteria met). Call ledger_get_handoff_status (current_agent: "Documentation") to confirm handoff.

## Assistant

WP-014 is COMPLETE. Now checking handoff status — this is already embedded in the `ledger_get_next_action` response pattern, but I'll check what's next:

```json
{
  "id": "toolu_01DdwMKz3SdkzzbuiA4iQx5Z",
  "caller": {
    "type": "direct"
  },
  "input": {
    "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend",
    "agent_role": "Documentation"
  },
  "name": "ledger_get_next_action",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_next_action` (id: `toolu_01DdwMKz3SdkzzbuiA4iQx5Z`)

```json
{
  "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend",
  "agent_role": "Documentation"
}
```

## Tool Result

{"action": "WAIT", "reason": "Pipeline completed for the active work package. The orchestrator will route the next work package."}

## Assistant

```json
{
  "id": "toolu_018QH8vUhCHEkC6tGqu4HiZS",
  "caller": {
    "type": "direct"
  },
  "input": {
    "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend",
    "current_agent": "Documentation"
  },
  "name": "ledger_get_handoff_status",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_handoff_status` (id: `toolu_018QH8vUhCHEkC6tGqu4HiZS`)

```json
{
  "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend",
  "current_agent": "Documentation"
}
```

## Tool Result

{
  "current_agent": "Documentation",
  "next_agent": "Documentation",
  "status": "IN_PROGRESS",
  "details": "Documentation work in progress. 1 work package(s) need documentation or re-documentation after upstream changes.",
  "next_action": "Call ledger_get_next_action with agent_role: \"Documentation\" to find the next work package to document. Continue working until all WPs have PASS documentation pipelines and are marked COMPLETE."
}

## Assistant

More documentation work to do. Let me get the next action:

```json
{
  "id": "toolu_016ckjhkQf9L3b6NgzS3ZMge",
  "caller": {
    "type": "direct"
  },
  "input": {
    "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend",
    "agent_role": "Documentation",
    "max_results": 3
  },
  "name": "ledger_get_next_action",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_next_action` (id: `toolu_016ckjhkQf9L3b6NgzS3ZMge`)

```json
{
  "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend",
  "agent_role": "Documentation",
  "max_results": 3
}
```

## Tool Result

{"action": "WAIT", "reason": "Pipeline completed for the active work package. The orchestrator will route the next work package."}

## Assistant

```json
{
  "id": "toolu_014Hn5APqtCVHwwy3BkJDkwm",
  "caller": {
    "type": "direct"
  },
  "input": {
    "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
  },
  "name": "ledger_list_work_packages",
  "type": "tool_use"
}
```

**Tool call:** `ledger_list_work_packages` (id: `toolu_014Hn5APqtCVHwwy3BkJDkwm`)

```json
{
  "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
}
```

## Tool Result

[
  {
    "work_package_id": "WP-001",
    "status": "COMPLETE",
    "assigned_to": "Documentation",
    "dependencies": [],
    "file": "ledger/WP-001.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-002",
    "status": "CANCELLED",
    "assigned_to": null,
    "dependencies": [
      "WP-001"
    ],
    "file": "ledger/WP-002.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-003",
    "status": "CANCELLED",
    "assigned_to": null,
    "dependencies": [
      "WP-001",
      "WP-002"
    ],
    "file": "ledger/WP-003.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-004",
    "status": "CANCELLED",
    "assigned_to": null,
    "dependencies": [
      "WP-003"
    ],
    "file": "ledger/WP-004.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-005",
    "status": "CANCELLED",
    "assigned_to": null,
    "dependencies": [
      "WP-003"
    ],
    "file": "ledger/WP-005.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-006",
    "status": "CANCELLED",
    "assigned_to": null,
    "dependencies": [
      "WP-005"
    ],
    "file": "ledger/WP-006.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-007",
    "status": "CANCELLED",
    "assigned_to": null,
    "dependencies": [
      "WP-006"
    ],
    "file": "ledger/WP-007.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-008",
    "status": "CANCELLED",
    "assigned_to": null,
    "dependencies": [
      "WP-001"
    ],
    "file": "ledger/WP-008.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-009",
    "status": "CANCELLED",
    "assigned_to": null,
    "dependencies": [
      "WP-001"
    ],
    "file": "ledger/WP-009.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "security-audit",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-010",
    "status": "CANCELLED",
    "assigned_to": null,
    "dependencies": [
      "WP-001"
    ],
    "file": "ledger/WP-010.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "security-audit",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-011",
    "status": "COMPLETE",
    "assigned_to": "Documentation",
    "dependencies": [
      "WP-001"
    ],
    "file": "ledger/WP-011.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "security-audit",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-012",
    "status": "COMPLETE",
    "assigned_to": "Documentation",
    "dependencies": [
      "WP-001"
    ],
    "file": "ledger/WP-012.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "security-audit",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-013",
    "status": "COMPLETE",
    "assigned_to": "Documentation",
    "dependencies": [
      "WP-001",
      "WP-011"
    ],
    "file": "ledger/WP-013.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-014",
    "status": "COMPLETE",
    "assigned_to": "Documentation",
    "dependencies": [
      "WP-011",
      "WP-013"
    ],
    "file": "ledger/WP-014.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-015",
    "status": "IN_PROGRESS",
    "assigned_to": "Reviewer",
    "dependencies": [
      "WP-011",
      "WP-013"
    ],
    "file": "ledger/WP-015.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-016",
    "status": "READY",
    "assigned_to": null,
    "dependencies": [
      "WP-001",
      "WP-011",
      "WP-014"
    ],
    "file": "ledger/WP-016.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "code-review",
      "documentation"
    ]
  },
  {
    "work_package_id": "WP-017",
    "status": "BLOCKED",
    "assigned_to": null,
    "dependencies": [
      "WP-016"
    ],
    "file": "ledger/WP-017.json",
    "active_pipeline_stages": [
      "implementation",
      "qa",
      "code-review",
      "documentation"
    ]
  }
]

## Assistant

```json
{
  "id": "toolu_014mgCn46EyEajKb9yjFD1VH",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Add Project Detail view section to README.md (after Dashboard view section)",
        "status": "completed"
      },
      {
        "content": "Update project-detail.js module-level JSDoc to document the refresh/full-refresh-on-mutation strategy",
        "status": "completed"
      },
      {
        "content": "Complete the documentation pipeline via ledger_complete_pipeline",
        "status": "completed"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_014mgCn46EyEajKb9yjFD1VH`)

```json
{
  "todos": [
    {
      "content": "Add Project Detail view section to README.md (after Dashboard view section)",
      "status": "completed"
    },
    {
      "content": "Update project-detail.js module-level JSDoc to document the refresh/full-refresh-on-mutation strategy",
      "status": "completed"
    },
    {
      "content": "Complete the documentation pipeline via ledger_complete_pipeline",
      "status": "completed"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Add Project Detail view section to README.md (after Dashboard view section)', 'status': 'completed'}, {'content': 'Update project-detail.js module-level JSDoc to document the refresh/full-refresh-on-mutation strategy', 'status': 'completed'}, {'content': 'Complete the documentation pipeline via ledger_complete_pipeline', 'status': 'completed'}]

## Assistant

WP-015 is IN_PROGRESS assigned to Reviewer — not yet ready for documentation. WP-016 is READY but not yet assigned to Documentation. My action is WAIT. The handoff status shows the next agent is still "Documentation" because WP-015 will need docs once it completes its pipeline, but I can't do anything until the Reviewer finishes WP-015.

```
CURRENT AGENT: Documentation
NEXT AGENT: Documentation
STATUS: IN_PROGRESS
```

---

**Summary of what was done for WP-014:**

**`README.md`** — two new sections added:

1. **Project Detail view (`views/project-detail.js`)** — full narrative covering: parallel data-fetching strategy, all rendered sections (metadata with inline description edit, repositories with filtered add-picker and remove-with-confirm, workspaces table with STABLE-disabled delete, add-workspace form with format validation, danger zone with rename and delete), the full-refresh-on-mutation strategy and its trade-off, the router injection pattern and null-guard, and a note on key casing.

2. **Normalisation helpers note** — table summarising all three normalise helpers across view modules, plus an explicit callout of the `normaliseRepo` duplication between `repositories.js` and `project-detail.js`, with `gui/public/js/utils/normalise.js` flagged as the suggested consolidation location.

**`gui/public/js/views/project-detail.js`** — module-level JSDoc expanded with dedicated subsections for *Data fetching*, *Refresh strategy (full-refresh-on-mutation)*, and *Router injection* — addressing both reviewer documentation-forward items. A `@module project-detail` tag was also added.

---

## Token Usage

| Metric | Count |
| ------ | ----- |
| Input Tokens | 1349748 |
| Output Tokens | 7696 |
| Total Tokens | 1357444 |

