# Dialogue — qa / WP-016

| Field | Value |
| ----- | ----- |
| Stage | `qa` |
| WP ID | `WP-016` |
| Captured | 2026-04-07T21:14:05.891471+00:00 |

## Human

Please start using the project path: `/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend`.

> NOTE: You can use this project path for all ledger tool calls to identify the current project.


## Assistant

```json
{
  "id": "toolu_01MwdXKsqDPGuuhswNB88kHL",
  "caller": {
    "type": "direct"
  },
  "input": {
    "agent_role": "QA",
    "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
  },
  "name": "ledger_get_next_action",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_next_action` (id: `toolu_01MwdXKsqDPGuuhswNB88kHL`)

```json
{
  "agent_role": "QA",
  "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
}
```

## Tool Result

{
  "action": "RUN_QA",
  "work_package_id": "WP-016",
  "reason": "Work package WP-016 has PASS implementation pipeline but no QA pipeline. Run QA.",
  "next_steps": [
    "1. Call ledger_begin_work (work_package_id: \"WP-016\", type: \"qa\", agent_role: \"QA\").",
    "2. Call ledger_get_work_package to review implementation artifacts and acceptance criteria.",
    "3. Execute the Verification Stack: build check, AC verification, regression tests, edge-case stress tests.",
    "4. Call ledger_complete_pipeline (work_package_id: \"WP-016\", type: \"qa\", status: PASS/FAIL, summary, metrics, comments, acceptance_criteria_updates).",
    "5. Call ledger_get_handoff_status (current_agent: \"QA\")."
  ],
  "handoff_notes": [
    "All 6 acceptance criteria are met and marked as such in the ledger.",
    "New file: gui/public/js/views/workspace-detail.js — exports renderWorkspaceDetail(container, params) and setRouter(router).",
    "Modified file: gui/public/js/app.js — imports renderWorkspaceDetail and setWorkspaceDetailRouter; replaces the placeholder stub for '#/projects/:id/workspaces/:wid'.",
    "Cleanup/polling: the view returns a cleanup function; the router's existing _render() mechanism handles it — no router.js changes required.",
    "Key integration points for QA: (1) polling interval cleared on navigation, (2) STABLE workspace disables Rename and Delete, (3) rename navigates to the new workspace URL, (4) delete navigates back to project detail, (5) failed API load renders an error state with a back link."
  ]
}

## Assistant

```json
{
  "id": "toolu_01DjG31QyESePjkWucRvvJBd",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-016",
    "type": "qa",
    "agent_role": "QA",
    "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
  },
  "name": "ledger_begin_work",
  "type": "tool_use"
}
```

**Tool call:** `ledger_begin_work` (id: `toolu_01DjG31QyESePjkWucRvvJBd`)

```json
{
  "work_package_id": "WP-016",
  "type": "qa",
  "agent_role": "QA",
  "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
}
```

## Tool Result

{
  "work_package_id": "WP-016",
  "work_package_file": "work/WP-006.md",
  "status": "IN_PROGRESS",
  "assigned_to": "QA",
  "dependencies": [
    "WP-001",
    "WP-011",
    "WP-014"
  ],
  "acceptance_criteria": [
    {
      "criterion": "Navigating to #/projects/:id/workspaces/:wid renders the workspace header and a repository status table with one row per repository.",
      "met": true
    },
    {
      "criterion": "Each repository row displays the current branch name, a color-coded Git status badge showing modified files, commits ahead/behind, last activity, and conflict indicator.",
      "met": true
    },
    {
      "criterion": "The status table auto-refreshes via setInterval calling GET /api/projects/:id/workspaces/:wid/status and updates badges in-place without re-rendering the entire view.",
      "met": true
    },
    {
      "criterion": "The polling interval is cleared when navigating away from the workspace detail view (no leaked intervals after navigation).",
      "met": true
    },
    {
      "criterion": "The 'Switch Branches' button navigates to #/projects/:id/workspaces/:wid/branch-switch.",
      "met": true
    },
    {
      "criterion": "Rename and Delete actions are disabled for STABLE workspace and functional with confirmation dialogs for other workspaces.",
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
      "started_at": "2026-04-07T21:10:29Z",
      "completed_at": "2026-04-07T21:13:34Z",
      "duration_ms": 185000,
      "summary": [
        "Created gui/public/js/views/workspace-detail.js — the full workspace detail view with header, breadcrumb, repository status table, and actions section.",
        "Implemented in-place polling: setInterval (10 s) calls api.status.get() and calls updateStatusTable() which locates rows/badge wrappers by data-repo-id and replaces only badge DOM nodes — no full re-render.",
        "Cleanup contract: renderWorkspaceDetail returns a cleanup function that clears the pollingInterval. The router's existing _render() already stores and calls any function returned by a view, so no router.js changes were needed.",
        "Actions section: 'Switch Branches' navigates to #/projects/:id/workspaces/:wid/branch-switch. 'Rename Workspace' shows an inline form with /^[A-Z]{2,6}$/ validation and calls api.workspaces.rename(); on success navigates to the new workspace URL. 'Delete Workspace' shows showConfirm() dialog, calls api.workspaces.delete(), then navigates back to project detail. Both Rename and Delete are disabled (with title tooltip) when wid === 'STABLE'.",
        "Modified app.js: replaced the placeholderView stub for '#/projects/:id/workspaces/:wid' with the real renderWorkspaceDetail import; registered setWorkspaceDetailRouter(router) so programmatic navigation works.",
        "All 517 server-side tests continue to pass. Node --check confirmed no syntax errors in both modified/new files."
      ],
      "artifacts": {
        "files_modified": [
          "gui/public/js/views/workspace-detail.js",
          "gui/public/js/app.js"
        ]
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:13:34Z",
          "note": "workspace-detail.js: The in-place badge update in updateStatusTable() uses CSS.escape() for the [data-repo-id] attribute selector, which is correct and defensive. However, if a repo ID contains characters that are valid in data attributes but unusual in CSS selectors, a fallback via querySelectorAll('[data-repo-id]') with a manual filter loop would be marginally more robust. Low priority as repo IDs are currently validated to safe patterns."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:13:34Z",
          "note": "app.js: The placeholderView() factory function and its comment block are now only used by the Branch Switch stub. Once WP-017 (Branch Switch view) ships, placeholderView will be dead code and should be removed along with the comment block above it."
        },
        {
          "type": "debt",
          "priority": "low",
          "timestamp": "2026-04-07T21:13:34Z",
          "note": "workspace-detail.js: The Rename Workspace inline form validates the workspace ID with /^[A-Z]{2,6}$/ — the same pattern used in project-detail.js buildAddWorkspaceForm(). This pattern is duplicated in two places. A shared validator function in a utils module or form-helpers.js would eliminate the duplication."
        }
      ]
    },
    {
      "type": "qa",
      "status": "IN_PROGRESS",
      "started_at": "2026-04-07T21:14:11Z",
      "summary": []
    }
  ],
  "status_changed_at": "2026-04-07T21:10:29Z",
  "last_updated": "2026-04-07T21:14:11Z",
  "handoff_notes": [
    {
      "from_agent": "Developer",
      "to_agent": "QA",
      "timestamp": "2026-04-07T21:13:34Z",
      "notes": [
        "All 6 acceptance criteria are met and marked as such in the ledger.",
        "New file: gui/public/js/views/workspace-detail.js — exports renderWorkspaceDetail(container, params) and setRouter(router).",
        "Modified file: gui/public/js/app.js — imports renderWorkspaceDetail and setWorkspaceDetailRouter; replaces the placeholder stub for '#/projects/:id/workspaces/:wid'.",
        "Cleanup/polling: the view returns a cleanup function; the router's existing _render() mechanism handles it — no router.js changes required.",
        "Key integration points for QA: (1) polling interval cleared on navigation, (2) STABLE workspace disables Rename and Delete, (3) rename navigates to the new workspace URL, (4) delete navigates back to project detail, (5) failed API load renders an error state with a back link."
      ]
    }
  ],
  "claimed": false
}

## Assistant

Now let me read the work package spec and the implementation files in parallel.

```json
{
  "id": "toolu_01VzQxptFRDccHvmMFxu8Gx2",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend/work/WP-006.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01YShcUDWJgSou67WYnytjwt",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/workspace-detail.js"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01VzQxptFRDccHvmMFxu8Gx2`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend/work/WP-006.md"
}
```

**Tool call:** `read_file` (id: `toolu_01YShcUDWJgSou67WYnytjwt`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/workspace-detail.js"
}
```

## Tool Result

     1	# WP-006: Workspace Detail View with Git Status Polling
     2	
     3	## Description
     4	
     5	Create the workspace detail view that displays a workspace's repository status table with live-updating Git status badges, and integrate frontend polling to keep the status current. The view shows each repository's current branch, Git status (modified files, ahead/behind counts, conflicts), and error indicators. Status auto-refreshes at a polling interval using `setInterval`, with the interval cleared on navigation away. This WP also includes the "Switch Branches" navigation button that links to the branch switch wizard (implemented in the next WP).
     6	
     7	## Scope
     8	
     9	### `gui/public/js/views/workspace-detail.js` (NEW)
    10	
    11	Exports a `renderWorkspaceDetail(container, params)` function. Uses `params.id` (project ID) and `params.wid` (workspace ID).
    12	
    13	- **Data fetching**:
    14	  - Fetches workspace via `api.workspaces.get(projectId, wid)` for metadata (description, etc.).
    15	  - Fetches project via `api.projects.get(projectId)` for the `Repositories` array (repo IDs and names).
    16	  - Fetches initial status via `api.status.get(projectId, wid)` → returns `Record<repoId, GitStatusInfo | null>` keyed by **repository ID** (not path — the backend builds the status map using repo IDs directly).
    17	
    18	- **Header**: Workspace ID, description, project context breadcrumb link (`#/projects/:id`).
    19	
    20	- **Repository Status Table**: One row per repository from the project's `Repositories` list showing:
    21	  - Repository name/ID (from the global repo list or directly from the project's Repositories array).
    22	  - Current branch name (from `status[repoId].currentBranch`).
    23	  - Git status badge via `createStatusBadge(status[repoId])` — shows modifiedFiles, localCommits (ahead), unfetchedCommits (behind), lastActivity, hasConflicts.
    24	  - Error/loading indicator when `status[repoId]` is `null` (repo not yet polled or clone failed).
    25	
    26	- **Polling**: Sets a `setInterval` (e.g., every 10 seconds) that calls `api.status.get(projectId, wid)` and updates status badges **in-place** using DOM manipulation (finding existing badge elements by a data attribute like `data-repo-id` and replacing their contents) without re-rendering the entire view. The interval ID is stored so it can be cleared.
    27	
    28	- **Cleanup**: The router must clear the polling interval when navigating away. Strategy: store the interval ID on the `#app` container element (e.g., via a `_cleanup` function property) and have the router call it before clearing the container. Alternatively, use `MutationObserver` or check DOM existence in the interval callback.
    29	
    30	- **Actions**:
    31	  - "Switch Branches" button: navigates to `#/projects/:id/workspaces/:wid/branch-switch`.
    32	  - "Rename Workspace" action: disabled (visually and functionally) for the STABLE workspace. For other workspaces: shows a text input + confirmation dialog, calls `api.workspaces.rename(projectId, wid, newId)` (which sends `PUT /api/projects/:id/workspaces/:wid/rename` with `{ newId }`). On success, navigates to `#/projects/:id/workspaces/:newId`.
    33	  - "Delete Workspace" button: disabled for STABLE. For others: `showConfirm()` with warning, calls `api.workspaces.delete(projectId, wid)` (which sends `DELETE /api/projects/:id/workspaces/:wid`). On success, navigates to `#/projects/:id`.
    34	
    35	### `gui/public/js/app.js` (MODIFY)
    36	
    37	Replace the placeholder route for `#/projects/:id/workspaces/:wid` with the actual `renderWorkspaceDetail` import and registration.
    38	
    39	### `gui/public/js/router.js` (MODIFY — if needed)
    40	
    41	Ensure the router supports a cleanup/teardown mechanism so views can clear intervals or remove event listeners when navigated away from. Recommended approach: before clearing `#app` and rendering the next view, check if the container has a `_cleanup` function and call it. This avoids leaked `setInterval` timers.
    42	
    43	## Plan Steps Covered
    44	
    45	- Step 9 (Workspace Detail View)
    46	- Step 13 (Git Status Polling — Frontend)
    47	
    48	## Ledger Metadata
    49	
    50	- **Ledger WP ID**: WP-016
    51	- **Dependencies**: WP-001, WP-011, WP-014
    52	- **Pipeline Stages**: implementation, qa, code-review, documentation
    53	
    54	## Acceptance Criteria
    55	
    56	1. Navigating to `#/projects/:id/workspaces/:wid` renders the workspace header and a repository status table with one row per repository.
    57	2. Each repository row displays the current branch name, a color-coded Git status badge showing modified files, commits ahead/behind, last activity, and conflict indicator.
    58	3. The status table auto-refreshes via `setInterval` calling `GET /api/projects/:id/workspaces/:wid/status` and updates badges in-place without re-rendering the entire view.
    59	4. The polling interval is cleared when navigating away from the workspace detail view (no leaked intervals after navigation).
    60	5. The "Switch Branches" button navigates to `#/projects/:id/workspaces/:wid/branch-switch`.
    61	6. Rename and Delete actions are disabled for STABLE workspace and functional with confirmation dialogs for other workspaces.

## Tool Result

     1	/**
     2	 * Workspace Detail View — Repo Parallelizer GUI.
     3	 *
     4	 * Renders the full detail page for a single workspace inside a project:
     5	 *   - Workspace header: ID, description, breadcrumb link back to the project.
     6	 *   - Repository status table: one row per repository, showing current branch,
     7	 *     a color-coded Git status badge, and an error/loading indicator for repos
     8	 *     with no status data yet.
     9	 *   - Live polling: status badges refresh in-place every 10 seconds via
    10	 *     `setInterval`. The interval is cleared via the cleanup function returned
    11	 *     from `renderWorkspaceDetail`, which the router calls before navigating
    12	 *     away.
    13	 *   - Actions: "Switch Branches" navigation button, "Rename Workspace" (disabled
    14	 *     for STABLE), "Delete Workspace" (disabled for STABLE).
    15	 *
    16	 * ## Router integration
    17	 *
    18	 * The view uses the same router-injection pattern as `project-detail.js`:
    19	 * `app.js` calls `setRouter(router)` before `router.start()`. The `_router`
    20	 * variable is null-guarded at every navigation site so the view remains
    21	 * functional in test contexts.
    22	 *
    23	 * ## Cleanup contract
    24	 *
    25	 * `renderWorkspaceDetail` returns a cleanup function. The router's `_render`
    26	 * method already stores and calls any function returned by a view. No changes
    27	 * to `router.js` are needed.
    28	 *
    29	 * @module workspace-detail
    30	 */
    31	
    32	import { api }               from '../api.js';
    33	import { showToast }         from '../components/toast.js';
    34	import { showConfirm }       from '../components/confirm-dialog.js';
    35	import { createStatusBadge } from '../components/status-badge.js';
    36	import { createFormField, validateRequired } from '../components/form-helpers.js';
    37	
    38	// ---------------------------------------------------------------------------
    39	// Router reference — injected from app.js via setRouter()
    40	// ---------------------------------------------------------------------------
    41	
    42	/** @type {import('../router.js').Router|null} */
    43	let _router = null;
    44	
    45	/**
    46	 * Inject the router instance so workspace-detail can navigate on rename/delete.
    47	 * Called from app.js before the router starts.
    48	 *
    49	 * @param {import('../router.js').Router} router
    50	 */
    51	export function setRouter(router) {
    52	    _router = router;
    53	}
    54	
    55	// ---------------------------------------------------------------------------
    56	// Constants
    57	// ---------------------------------------------------------------------------
    58	
    59	/** Polling interval in milliseconds. */
    60	const POLL_INTERVAL_MS = 10_000;
    61	
    62	/** The workspace ID that cannot be renamed or deleted. */
    63	const STABLE_WS_ID = 'STABLE';
    64	
    65	// ---------------------------------------------------------------------------
    66	// Normalisation helpers
    67	// ---------------------------------------------------------------------------
    68	
    69	/**
    70	 * Normalise a project object from the backend (Go-style keys or lowercase).
    71	 *
    72	 * @param {Object} project
    73	 * @returns {{ id: string, name: string, description: string, repositories: Array }}
    74	 */
    75	function normaliseProject(project) {
    76	    return {
    77	        id:           project.Id          || project.id          || '',
    78	        name:         project.Name        || project.name        || '',
    79	        description:  project.Description || project.description || '',
    80	        repositories: Array.isArray(project.Repositories)
    81	            ? project.Repositories
    82	            : (Array.isArray(project.repositories) ? project.repositories : []),
    83	    };
    84	}
    85	
    86	/**
    87	 * Normalise a workspace object from the backend.
    88	 *
    89	 * @param {Object} ws
    90	 * @returns {{ id: string, description: string }}
    91	 */
    92	function normaliseWorkspace(ws) {
    93	    return {
    94	        id:          ws.Id          || ws.id          || '',
    95	        description: ws.Description || ws.description || '',
    96	    };
    97	}
    98	
    99	/**
   100	 * Extract a repository's ID from either a plain string or an object.

## Assistant

```json
{
  "id": "toolu_01PnxQwfQJTjytJ3FmMisJy6",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/workspace-detail.js",
    "offset": 99,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01XN3vygpaUBL39cZHTTENnN",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/app.js"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01PnxQwfQJTjytJ3FmMisJy6`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/workspace-detail.js",
  "offset": 99,
  "limit": 200
}
```

**Tool call:** `read_file` (id: `toolu_01XN3vygpaUBL39cZHTTENnN`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/app.js"
}
```

## Tool Result

   100	 * Extract a repository's ID from either a plain string or an object.
   101	 * The backend may return Repositories as an array of strings, an array of
   102	 * objects with `Id`/`id`, or an array of objects with `repositoryId`.
   103	 *
   104	 * @param {string|Object} repo
   105	 * @returns {string}
   106	 */
   107	function extractRepoId(repo) {
   108	    if (typeof repo === 'string') return repo;
   109	    return repo.Id || repo.id || repo.RepositoryId || repo.repositoryId || '';
   110	}
   111	
   112	/**
   113	 * Extract a human-readable repository name from a repository entry.
   114	 * Falls back to the ID when no name is available.
   115	 *
   116	 * @param {string|Object} repo
   117	 * @returns {string}
   118	 */
   119	function extractRepoName(repo) {
   120	    if (typeof repo === 'string') return repo;
   121	    return repo.Name || repo.name || extractRepoId(repo);
   122	}
   123	
   124	// ---------------------------------------------------------------------------
   125	// Loading helper
   126	// ---------------------------------------------------------------------------
   127	
   128	/**
   129	 * Render a loading spinner into `el`.
   130	 *
   131	 * @param {HTMLElement} el
   132	 * @param {string} [label]
   133	 */
   134	function showLoading(el, label = 'Loading…') {
   135	    el.innerHTML = `
   136	        <div class="loading-indicator" aria-live="polite">
   137	            <span class="spinner" aria-hidden="true"></span>
   138	            <span>${label}</span>
   139	        </div>
   140	    `;
   141	}
   142	
   143	// ---------------------------------------------------------------------------
   144	// Status table helpers
   145	// ---------------------------------------------------------------------------
   146	
   147	/**
   148	 * Build the status `<tbody>` row for a single repository.
   149	 *
   150	 * The row uses `data-repo-id` on the badge container so the polling update
   151	 * can locate and replace badge contents in-place.
   152	 *
   153	 * @param {string} repoId
   154	 * @param {string} repoName
   155	 * @param {Object|null} statusInfo - GitStatusInfo or null.
   156	 * @returns {HTMLTableRowElement}
   157	 */
   158	function buildRepoStatusRow(repoId, repoName, statusInfo) {
   159	    const tr = document.createElement('tr');
   160	    tr.dataset.repoId = repoId;
   161	
   162	    // Repository name / ID
   163	    const nameCell = document.createElement('td');
   164	    nameCell.className = 'repo-name-cell';
   165	    const nameEl = document.createElement('span');
   166	    nameEl.className = 'repo-name';
   167	    nameEl.textContent = repoName !== repoId ? `${repoName}` : repoId;
   168	    if (repoName !== repoId) {
   169	        const idHint = document.createElement('span');
   170	        idHint.className = 'text-muted repo-id-hint';
   171	        idHint.textContent = ` (${repoId})`;
   172	        nameEl.appendChild(idHint);
   173	    }
   174	    nameCell.appendChild(nameEl);
   175	    tr.appendChild(nameCell);
   176	
   177	    // Branch name
   178	    const branchCell = document.createElement('td');
   179	    branchCell.className = 'repo-branch-cell';
   180	    branchCell.textContent = (statusInfo && statusInfo.currentBranch)
   181	        ? statusInfo.currentBranch
   182	        : '—';
   183	    tr.appendChild(branchCell);
   184	
   185	    // Status badge cell
   186	    const badgeCell = document.createElement('td');
   187	    badgeCell.className = 'repo-badge-cell';
   188	
   189	    const badgeWrapper = document.createElement('div');
   190	    badgeWrapper.dataset.repoId = repoId;
   191	    badgeWrapper.appendChild(createStatusBadge(statusInfo || null));
   192	    badgeCell.appendChild(badgeWrapper);
   193	    tr.appendChild(badgeCell);
   194	
   195	    return tr;
   196	}
   197	
   198	/**
   199	 * Update an existing status table in-place by replacing badge contents and
   200	 * branch text for each repository whose status has changed.
   201	 *
   202	 * Rows are located via `[data-repo-id]` on both the `<tr>` and the badge
   203	 * wrapper `<div>` inside it. No full re-render of the table is performed.
   204	 *
   205	 * @param {HTMLElement}           tableBody - The `<tbody>` to update.
   206	 * @param {Record<string, Object|null>} statusMap - Keyed by repository ID.
   207	 */
   208	function updateStatusTable(tableBody, statusMap) {
   209	    for (const [repoId, statusInfo] of Object.entries(statusMap)) {
   210	        const row = tableBody.querySelector(`tr[data-repo-id="${CSS.escape(repoId)}"]`);
   211	        if (!row) continue;
   212	
   213	        // Update branch cell (second cell)
   214	        const branchCell = row.cells[1];
   215	        if (branchCell) {
   216	            branchCell.textContent = (statusInfo && statusInfo.currentBranch)
   217	                ? statusInfo.currentBranch
   218	                : '—';
   219	        }
   220	
   221	        // Update badge wrapper (third cell → div[data-repo-id])
   222	        const badgeWrapper = row.querySelector(`div[data-repo-id="${CSS.escape(repoId)}"]`);
   223	        if (badgeWrapper) {
   224	            badgeWrapper.innerHTML = '';
   225	            badgeWrapper.appendChild(createStatusBadge(statusInfo || null));
   226	        }
   227	    }
   228	}
   229	
   230	// ---------------------------------------------------------------------------
   231	// Section builders
   232	// ---------------------------------------------------------------------------
   233	
   234	/**
   235	 * Build the workspace header section.
   236	 *
   237	 * @param {string} projectId
   238	 * @param {{ id: string, description: string }} workspace
   239	 * @returns {HTMLElement}
   240	 */
   241	function buildHeaderSection(projectId, workspace) {
   242	    const header = document.createElement('div');
   243	    header.className = 'page-header workspace-detail-header';
   244	
   245	    // Breadcrumb
   246	    const breadcrumb = document.createElement('nav');
   247	    breadcrumb.className = 'breadcrumb';
   248	    breadcrumb.setAttribute('aria-label', 'Breadcrumb');
   249	
   250	    const projectLink = document.createElement('a');
   251	    projectLink.href      = `#/projects/${encodeURIComponent(projectId)}`;
   252	    projectLink.textContent = projectId;
   253	    projectLink.className = 'breadcrumb-link';
   254	    if (_router) {
   255	        projectLink.addEventListener('click', (e) => {
   256	            e.preventDefault();
   257	            _router.navigate(`#/projects/${encodeURIComponent(projectId)}`);
   258	        });
   259	    }
   260	
   261	    const separator = document.createElement('span');
   262	    separator.className   = 'breadcrumb-sep';
   263	    separator.textContent = ' / ';
   264	    separator.setAttribute('aria-hidden', 'true');
   265	
   266	    const currentPage = document.createElement('span');
   267	    currentPage.className   = 'breadcrumb-current';
   268	    currentPage.textContent = workspace.id;
   269	    currentPage.setAttribute('aria-current', 'page');
   270	
   271	    breadcrumb.appendChild(projectLink);
   272	    breadcrumb.appendChild(separator);
   273	    breadcrumb.appendChild(currentPage);
   274	    header.appendChild(breadcrumb);
   275	
   276	    // Title
   277	    const titleEl = document.createElement('h1');
   278	    titleEl.className   = 'workspace-detail-title';
   279	    titleEl.textContent = `Workspace: ${workspace.id}`;
   280	    header.appendChild(titleEl);
   281	
   282	    // Description
   283	    if (workspace.description) {
   284	        const descEl = document.createElement('p');
   285	        descEl.className   = 'workspace-detail-description text-secondary';
   286	        descEl.textContent = workspace.description;
   287	        header.appendChild(descEl);
   288	    }
   289	
   290	    return header;
   291	}
   292	
   293	/**
   294	 * Build the repository status table section.
   295	 *
   296	 * @param {Array<{ repoId: string, repoName: string }>} repos
   297	 * @param {Record<string, Object|null>} statusMap
   298	 * @returns {{ section: HTMLElement, tbody: HTMLTableSectionElement }}
   299	 */

## Tool Result

     1	/**
     2	 * Application bootstrap for Repo Parallelizer GUI.
     3	 *
     4	 * Instantiates the hash-based router, registers all view routes, and starts
     5	 * listening for navigation events.
     6	 *
     7	 * Route registry:
     8	 *   #/                                           → Dashboard        (WP-013)
     9	 *   #/repositories                               → Repositories     (WP-015)
    10	 *   #/projects/:id                               → Project Detail   (WP-014)
    11	 *   #/projects/:id/workspaces/:wid               → Workspace Detail (WP-016)
    12	 *   #/projects/:id/workspaces/:wid/branch-switch → Branch Switch    (future WP stub)
    13	 */
    14	
    15	import { Router }                                        from './router.js';
    16	import { renderDashboard, setRouter }                    from './views/dashboard.js';
    17	import { renderRepositories }                            from './views/repositories.js';
    18	import { renderProjectDetail, setRouter as setProjectDetailRouter } from './views/project-detail.js';
    19	import { renderWorkspaceDetail, setRouter as setWorkspaceDetailRouter } from './views/workspace-detail.js';
    20	
    21	// ---------------------------------------------------------------------------
    22	// Placeholder stub factory
    23	//
    24	// Generates a minimal view function for routes whose full view has not yet
    25	// been implemented. Remove the stub and import the real view when the
    26	// corresponding WP ships.
    27	//
    28	// NOTE: The label string is trusted internal code only — no XSS risk.
    29	// ---------------------------------------------------------------------------
    30	
    31	/**
    32	 * Create a placeholder view function for a not-yet-implemented route.
    33	 *
    34	 * @param {string} label - Human-readable label shown in the placeholder.
    35	 * @returns {function(HTMLElement, Object): void}
    36	 */
    37	function placeholderView(label) {
    38	    return function (container, _params) {
    39	        const wrapper = document.createElement('div');
    40	        wrapper.innerHTML = `
    41	            <div class="page-header">
    42	                <h1>${label}</h1>
    43	            </div>
    44	            <div class="empty-state">
    45	                <p>Coming soon — ${label}</p>
    46	            </div>
    47	        `;
    48	        container.appendChild(wrapper);
    49	    };
    50	}
    51	
    52	// ---------------------------------------------------------------------------
    53	// Router instantiation & route registration
    54	// ---------------------------------------------------------------------------
    55	
    56	const router = new Router();
    57	
    58	// Inject router into views that need programmatic navigation.
    59	setRouter(router);
    60	setProjectDetailRouter(router);
    61	setWorkspaceDetailRouter(router);
    62	
    63	// Dashboard (WP-013)
    64	router.register('#/', renderDashboard);
    65	
    66	// Repositories list (WP-015)
    67	router.register('#/repositories', renderRepositories);
    68	
    69	// Project detail (WP-014)
    70	router.register('#/projects/:id', renderProjectDetail);
    71	
    72	// Workspace detail (WP-016)
    73	router.register('#/projects/:id/workspaces/:wid', renderWorkspaceDetail);
    74	
    75	// Branch switch (future WP — stub)
    76	router.register('#/projects/:id/workspaces/:wid/branch-switch', placeholderView('Branch Switch'));
    77	
    78	// ---------------------------------------------------------------------------
    79	// Start the router — must be called after all routes are registered
    80	// ---------------------------------------------------------------------------
    81	
    82	router.start();

## Assistant

```json
{
  "id": "toolu_01V3MkPqjb5PUFTqYUVnspsJ",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/workspace-detail.js",
    "offset": 299,
    "limit": 250
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01Se6CxHTVAsvCT1UZsfRKKh",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/router.js"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01V3MkPqjb5PUFTqYUVnspsJ`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/workspace-detail.js",
  "offset": 299,
  "limit": 250
}
```

**Tool call:** `read_file` (id: `toolu_01Se6CxHTVAsvCT1UZsfRKKh`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/router.js"
}
```

## Tool Result

   300	function buildStatusTableSection(repos, statusMap) {
   301	    const section = document.createElement('section');
   302	    section.className = 'workspace-status-section';
   303	
   304	    const heading = document.createElement('h2');
   305	    heading.className   = 'section-title';
   306	    heading.textContent = 'Repository Status';
   307	    section.appendChild(heading);
   308	
   309	    if (repos.length === 0) {
   310	        const empty = document.createElement('p');
   311	        empty.className   = 'empty-state-inline text-secondary';
   312	        empty.textContent = 'No repositories in this project.';
   313	        section.appendChild(empty);
   314	        return { section, tbody: null };
   315	    }
   316	
   317	    const table = document.createElement('table');
   318	    table.className = 'data-table workspace-status-table';
   319	
   320	    const thead = document.createElement('thead');
   321	    const htr   = document.createElement('tr');
   322	    ['Repository', 'Branch', 'Status'].forEach((label) => {
   323	        const th = document.createElement('th');
   324	        th.textContent = label;
   325	        htr.appendChild(th);
   326	    });
   327	    thead.appendChild(htr);
   328	    table.appendChild(thead);
   329	
   330	    const tbody = document.createElement('tbody');
   331	
   332	    repos.forEach(({ repoId, repoName }) => {
   333	        const statusInfo = statusMap[repoId] ?? null;
   334	        tbody.appendChild(buildRepoStatusRow(repoId, repoName, statusInfo));
   335	    });
   336	
   337	    table.appendChild(tbody);
   338	    section.appendChild(table);
   339	
   340	    return { section, tbody };
   341	}
   342	
   343	/**
   344	 * Build the actions section.
   345	 *
   346	 * @param {string} projectId
   347	 * @param {string} wid        - Workspace ID.
   348	 * @param {boolean} isStable  - Whether this is the STABLE workspace.
   349	 * @returns {HTMLElement}
   350	 */
   351	function buildActionsSection(projectId, wid, isStable) {
   352	    const section = document.createElement('section');
   353	    section.className = 'workspace-actions-section';
   354	
   355	    const heading = document.createElement('h2');
   356	    heading.className   = 'section-title';
   357	    heading.textContent = 'Actions';
   358	    section.appendChild(heading);
   359	
   360	    const actionsRow = document.createElement('div');
   361	    actionsRow.className = 'workspace-actions-row';
   362	
   363	    // ---- Switch Branches button ----
   364	    const switchBtn = document.createElement('button');
   365	    switchBtn.type      = 'button';
   366	    switchBtn.className = 'btn btn-primary';
   367	    switchBtn.textContent = 'Switch Branches';
   368	    switchBtn.addEventListener('click', () => {
   369	        const target = `#/projects/${encodeURIComponent(projectId)}/workspaces/${encodeURIComponent(wid)}/branch-switch`;
   370	        if (_router) {
   371	            _router.navigate(target);
   372	        } else {
   373	            location.hash = target;
   374	        }
   375	    });
   376	    actionsRow.appendChild(switchBtn);
   377	
   378	    // ---- Rename Workspace ----
   379	    const renameWrapper = buildRenameWorkspaceAction(projectId, wid, isStable);
   380	    actionsRow.appendChild(renameWrapper);
   381	
   382	    // ---- Delete Workspace button ----
   383	    const deleteBtn = document.createElement('button');
   384	    deleteBtn.type      = 'button';
   385	    deleteBtn.className = 'btn btn-danger';
   386	    deleteBtn.textContent = 'Delete Workspace';
   387	
   388	    if (isStable) {
   389	        deleteBtn.disabled = true;
   390	        deleteBtn.title    = 'The STABLE workspace cannot be deleted.';
   391	        deleteBtn.classList.add('btn-disabled');
   392	    } else {
   393	        deleteBtn.addEventListener('click', async () => {
   394	            try {
   395	                await showConfirm(
   396	                    'Delete Workspace',
   397	                    `Delete workspace "${wid}"? All cloned repositories in this workspace will be permanently removed. This action cannot be undone.`,
   398	                );
   399	            } catch {
   400	                return; // User cancelled.
   401	            }
   402	
   403	            deleteBtn.disabled    = true;
   404	            deleteBtn.textContent = 'Deleting…';
   405	
   406	            try {
   407	                await api.workspaces.delete(projectId, wid);
   408	                showToast(`Workspace "${wid}" deleted.`, 'success');
   409	                if (_router) {
   410	                    _router.navigate(`#/projects/${encodeURIComponent(projectId)}`);
   411	                } else {
   412	                    location.hash = `#/projects/${encodeURIComponent(projectId)}`;
   413	                }
   414	            } catch (err) {
   415	                showToast(err.message || 'Failed to delete workspace.', 'error');
   416	                deleteBtn.disabled    = false;
   417	                deleteBtn.textContent = 'Delete Workspace';
   418	            }
   419	        });
   420	    }
   421	
   422	    actionsRow.appendChild(deleteBtn);
   423	    section.appendChild(actionsRow);
   424	
   425	    return section;
   426	}
   427	
   428	/**
   429	 * Build the Rename Workspace inline action.
   430	 *
   431	 * Returns a wrapper `<div>` containing the "Rename Workspace" button and a
   432	 * hidden inline form. When shown, the form accepts a new workspace ID and
   433	 * calls `api.workspaces.rename()` on submit.
   434	 *
   435	 * @param {string}  projectId
   436	 * @param {string}  wid       - Current workspace ID.
   437	 * @param {boolean} isStable
   438	 * @returns {HTMLElement}
   439	 */
   440	function buildRenameWorkspaceAction(projectId, wid, isStable) {
   441	    const wrapper = document.createElement('div');
   442	    wrapper.className = 'rename-workspace-wrapper';
   443	
   444	    // ---- Toggle button ----
   445	    const renameBtn = document.createElement('button');
   446	    renameBtn.type      = 'button';
   447	    renameBtn.className = 'btn btn-secondary';
   448	    renameBtn.textContent = 'Rename Workspace';
   449	
   450	    if (isStable) {
   451	        renameBtn.disabled = true;
   452	        renameBtn.title    = 'The STABLE workspace cannot be renamed.';
   453	        renameBtn.classList.add('btn-disabled');
   454	        wrapper.appendChild(renameBtn);
   455	        return wrapper;
   456	    }
   457	
   458	    wrapper.appendChild(renameBtn);
   459	
   460	    // ---- Inline form (hidden initially) ----
   461	    const formWrapper = document.createElement('div');
   462	    formWrapper.className = 'rename-workspace-form-wrapper card';
   463	    formWrapper.hidden = true;
   464	    wrapper.appendChild(formWrapper);
   465	
   466	    const formTitle = document.createElement('h4');
   467	    formTitle.className   = 'form-section-title';
   468	    formTitle.textContent = 'Rename Workspace';
   469	    formWrapper.appendChild(formTitle);
   470	
   471	    const newIdField = createFormField('New Workspace ID', 'text', 'newWorkspaceId', {
   472	        required:    true,
   473	        placeholder: 'e.g. DEV or FEATURE',
   474	        hint:        'Must be 2–6 uppercase letters (A-Z only).',
   475	    });
   476	    formWrapper.appendChild(newIdField);
   477	
   478	    const newIdInput   = newIdField.querySelector('[name="newWorkspaceId"]');
   479	    const newIdErrorEl = newIdField.querySelector('.field-error');
   480	
   481	    const formActions = document.createElement('div');
   482	    formActions.className = 'form-actions';
   483	
   484	    const saveBtn = document.createElement('button');
   485	    saveBtn.type      = 'button';
   486	    saveBtn.className = 'btn btn-primary btn-sm';
   487	    saveBtn.textContent = 'Save';
   488	
   489	    const cancelBtn = document.createElement('button');
   490	    cancelBtn.type      = 'button';
   491	    cancelBtn.className = 'btn btn-secondary btn-sm';
   492	    cancelBtn.textContent = 'Cancel';
   493	
   494	    formActions.appendChild(saveBtn);
   495	    formActions.appendChild(cancelBtn);
   496	    formWrapper.appendChild(formActions);
   497	
   498	    // ---- Behaviour ----
   499	
   500	    renameBtn.addEventListener('click', () => {
   501	        formWrapper.hidden = false;
   502	        renameBtn.hidden   = true;
   503	        if (newIdInput) newIdInput.focus();
   504	    });
   505	
   506	    cancelBtn.addEventListener('click', () => {
   507	        formWrapper.hidden = true;
   508	        renameBtn.hidden   = false;
   509	        if (newIdInput) newIdInput.value = '';
   510	        if (newIdErrorEl) newIdErrorEl.hidden = true;
   511	    });
   512	
   513	    saveBtn.addEventListener('click', async () => {
   514	        // Clear previous validation errors.
   515	        if (newIdErrorEl) newIdErrorEl.hidden = true;
   516	        if (newIdInput) {
   517	            newIdInput.classList.remove('error');
   518	            newIdInput.removeAttribute('aria-invalid');
   519	        }
   520	
   521	        if (!validateRequired(formWrapper, ['newWorkspaceId'])) return;
   522	
   523	        const newId = newIdInput ? newIdInput.value.trim() : '';
   524	
   525	        if (!/^[A-Z]{2,6}$/.test(newId)) {
   526	            if (newIdErrorEl) {
   527	                newIdErrorEl.textContent = 'Must be 2–6 uppercase letters (A-Z only).';
   528	                newIdErrorEl.hidden      = false;
   529	            }
   530	            if (newIdInput) {
   531	                newIdInput.classList.add('error');
   532	                newIdInput.setAttribute('aria-invalid', 'true');
   533	                newIdInput.focus();
   534	            }
   535	            return;
   536	        }
   537	
   538	        try {
   539	            await showConfirm(
   540	                'Rename Workspace',
   541	                `Rename workspace "${wid}" to "${newId}"? The page will navigate to the new workspace URL.`,
   542	            );
   543	        } catch {
   544	            return; // User cancelled.
   545	        }
   546	
   547	        saveBtn.disabled    = true;
   548	        saveBtn.textContent = 'Saving…';
   549	

## Tool Result

     1	/**
     2	 * Hash-based client-side router for the Repo Parallelizer SPA.
     3	 *
     4	 * Supports named parameters in patterns (e.g., `#/projects/:id`).
     5	 * Views are functions called with `(container, params)` where container
     6	 * is the `#app` DOM element and params is an object of extracted route
     7	 * parameters.
     8	 *
     9	 * @example
    10	 *   const router = new Router();
    11	 *   router.register('#/', dashboardView);
    12	 *   router.register('#/projects/:id', projectDetailView);
    13	 *   router.start();
    14	 */
    15	
    16	/**
    17	 * @typedef {Object} Route
    18	 * @property {string}   pattern  - The raw hash pattern (e.g., '#/projects/:id').
    19	 * @property {RegExp}   regex    - Compiled regex for matching.
    20	 * @property {string[]} paramNames - Ordered list of parameter names.
    21	 * @property {function(HTMLElement, Object): (void|Promise<void>)} view
    22	 */
    23	
    24	export class Router {
    25	    constructor() {
    26	        /** @type {Route[]} */
    27	        this._routes = [];
    28	
    29	        /** @type {HTMLElement|null} */
    30	        this._container = null;
    31	
    32	        /** @type {function|null} Current view's cleanup callback. */
    33	        this._cleanup = null;
    34	
    35	        // Bind once so we can add/remove the event listener cleanly.
    36	        this._onHashChange = this._onHashChange.bind(this);
    37	    }
    38	
    39	    // ------------------------------------------------------------------
    40	    // Public API
    41	    // ------------------------------------------------------------------
    42	
    43	    /**
    44	     * Register a route.
    45	     *
    46	     * @param {string}   hashPattern - Hash pattern, e.g., '#/projects/:id'.
    47	     * @param {function(HTMLElement, Object): (void|Promise<void>)} viewFunction
    48	     *   Called with (container, params). May return a cleanup function that
    49	     *   will be called before navigating away from this view.
    50	     */
    51	    register(hashPattern, viewFunction) {
    52	        const { regex, paramNames } = this._compilePattern(hashPattern);
    53	        this._routes.push({
    54	            pattern: hashPattern,
    55	            regex,
    56	            paramNames,
    57	            view: viewFunction,
    58	        });
    59	    }
    60	
    61	    /**
    62	     * Programmatic navigation — sets `location.hash` which triggers `hashchange`.
    63	     *
    64	     * @param {string} hash - Target hash, e.g., '#/projects/my-proj'.
    65	     */
    66	    navigate(hash) {
    67	        location.hash = hash;
    68	    }
    69	
    70	    /**
    71	     * Start listening for hash changes and render the current hash.
    72	     * Must be called after all routes have been registered.
    73	     */
    74	    start() {
    75	        this._container = document.getElementById('app');
    76	        if (!this._container) {
    77	            throw new Error('Router: #app container element not found in the DOM.');
    78	        }
    79	        window.addEventListener('hashchange', this._onHashChange);
    80	
    81	        // Render the current hash (or default to #/).
    82	        if (!location.hash || location.hash === '#') {
    83	            location.hash = '#/';
    84	        } else {
    85	            this._resolve(location.hash);
    86	        }
    87	    }
    88	
    89	    /**
    90	     * Stop listening and clean up (useful for testing / teardown).
    91	     */
    92	    stop() {
    93	        window.removeEventListener('hashchange', this._onHashChange);
    94	        this._runCleanup();
    95	    }
    96	
    97	    // ------------------------------------------------------------------
    98	    // Internal
    99	    // ------------------------------------------------------------------
   100	

## Assistant

```json
{
  "id": "toolu_01MR4XuBeYCtCnpLyX7iBCyz",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/router.js",
    "offset": 100,
    "limit": 100
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01TxFiLbbACNcp18UxwZPqb7",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/workspace-detail.js",
    "offset": 549,
    "limit": 150
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01MR4XuBeYCtCnpLyX7iBCyz`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/router.js",
  "offset": 100,
  "limit": 100
}
```

**Tool call:** `read_file` (id: `toolu_01TxFiLbbACNcp18UxwZPqb7`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/workspace-detail.js",
  "offset": 549,
  "limit": 150
}
```

## Tool Result

   101	    /**
   102	     * Compile a hash pattern into a regex and extract parameter names.
   103	     *
   104	     * '#/projects/:id/workspaces/:wid'
   105	     *   → regex: /^#\/projects\/([^/]+)\/workspaces\/([^/]+)$/
   106	     *   → paramNames: ['id', 'wid']
   107	     *
   108	     * **Trailing-slash behaviour:** Each `:param` segment is compiled to the
   109	     * capture group `([^/]+)`, which requires **at least one non-slash
   110	     * character**. This means a trailing-slash URL such as `#/projects/` will
   111	     * **NOT** match a pattern like `#/projects/:id` — the empty string after
   112	     * the final slash fails the `[^/]+` requirement. View authors should
   113	     * ensure navigation links never append a bare trailing slash when a param
   114	     * value is expected (e.g. use `#/projects/my-proj`, not `#/projects/`).
   115	     *
   116	     * @param {string} pattern
   117	     * @returns {{ regex: RegExp, paramNames: string[] }}
   118	     */
   119	    _compilePattern(pattern) {
   120	        const paramNames = [];
   121	        const regexStr = pattern.replace(/:([a-zA-Z_][a-zA-Z0-9_]*)/g, (_match, name) => {
   122	            paramNames.push(name);
   123	            return '([^/]+)';
   124	        });
   125	        // Escape remaining forward slashes and anchor.
   126	        const regex = new RegExp('^' + regexStr.replace(/\//g, '\\/') + '$');
   127	        return { regex, paramNames };
   128	    }
   129	
   130	    /**
   131	     * Match the current hash against registered routes and render.
   132	     *
   133	     * @param {string} hash
   134	     */
   135	    _resolve(hash) {
   136	        // Normalise: empty hash → '#/'
   137	        if (!hash || hash === '#') {
   138	            hash = '#/';
   139	        }
   140	
   141	        for (const route of this._routes) {
   142	            const match = hash.match(route.regex);
   143	            if (match) {
   144	                // Extract named params.
   145	                const params = {};
   146	                route.paramNames.forEach((name, i) => {
   147	                    params[name] = decodeURIComponent(match[i + 1]);
   148	                });
   149	                this._render(route.view, params);
   150	                return;
   151	            }
   152	        }
   153	
   154	        // No route matched — show a simple 404.
   155	        this._runCleanup();
   156	        if (this._container) {
   157	            this._container.innerHTML = '';
   158	            const msg = document.createElement('div');
   159	            msg.className = 'empty-state';
   160	            msg.textContent = `Page not found: ${hash}`;
   161	            this._container.appendChild(msg);
   162	        }
   163	    }
   164	
   165	    /**
   166	     * Clear the container, run the previous view's cleanup, and render
   167	     * the matched view.
   168	     *
   169	     * @param {function} viewFn
   170	     * @param {Object}   params
   171	     */
   172	    _render(viewFn, params) {
   173	        this._runCleanup();
   174	        if (this._container) {
   175	            this._container.innerHTML = '';
   176	            const result = viewFn(this._container, params);
   177	            // If the view returns a function, store it as cleanup.
   178	            if (typeof result === 'function') {
   179	                this._cleanup = result;
   180	            }
   181	        }
   182	    }
   183	
   184	    /** Run and discard the current cleanup callback. */
   185	    _runCleanup() {
   186	        if (this._cleanup) {
   187	            try {
   188	                this._cleanup();
   189	            } catch (_e) {
   190	                // Swallow cleanup errors — don't block navigation.
   191	            }
   192	            this._cleanup = null;
   193	        }
   194	    }
   195	
   196	    /** hashchange handler */
   197	    _onHashChange() {
   198	        this._resolve(location.hash);
   199	    }
   200	}

## Tool Result

   550	        try {
   551	            await api.workspaces.rename(projectId, wid, newId);
   552	            showToast(`Workspace renamed to "${newId}".`, 'success');
   553	            const target = `#/projects/${encodeURIComponent(projectId)}/workspaces/${encodeURIComponent(newId)}`;
   554	            if (_router) {
   555	                _router.navigate(target);
   556	            } else {
   557	                location.hash = target;
   558	            }
   559	        } catch (err) {
   560	            showToast(err.message || 'Failed to rename workspace.', 'error');
   561	            saveBtn.disabled    = false;
   562	            saveBtn.textContent = 'Save';
   563	        }
   564	    });
   565	
   566	    return wrapper;
   567	}
   568	
   569	// ---------------------------------------------------------------------------
   570	// Public view entry point
   571	// ---------------------------------------------------------------------------
   572	
   573	/**
   574	 * Render the workspace detail view.
   575	 *
   576	 * Fetches workspace metadata, project (for the repositories list), and
   577	 * initial Git status in parallel. Then starts a polling interval that
   578	 * updates badges in-place every {@link POLL_INTERVAL_MS} milliseconds.
   579	 *
   580	 * @param {HTMLElement} container - The `#app` DOM element provided by the router.
   581	 * @param {{ id: string, wid: string }} params - Route parameters.
   582	 * @returns {function(): void} Cleanup function — clears the polling interval.
   583	 *   The router stores and calls this before rendering the next view.
   584	 */
   585	export function renderWorkspaceDetail(container, params) {
   586	    const projectId = params.id;
   587	    const wid       = params.wid;
   588	
   589	    let pollingInterval = null;
   590	
   591	    // Return the cleanup function immediately so the router can register it
   592	    // even if the async bootstrap hasn't resolved yet.
   593	    const cleanup = () => {
   594	        if (pollingInterval !== null) {
   595	            clearInterval(pollingInterval);
   596	            pollingInterval = null;
   597	        }
   598	    };
   599	
   600	    // Show loading state immediately.
   601	    showLoading(container, 'Loading workspace…');
   602	
   603	    // Kick off parallel data fetch.
   604	    Promise.all([
   605	        api.workspaces.get(projectId, wid),
   606	        api.projects.get(projectId),
   607	        api.status.get(projectId, wid),
   608	    ]).then(([rawWorkspace, rawProject, statusMap]) => {
   609	        // Guard: if the container was cleared by navigation before we resolved,
   610	        // do nothing and let the cleanup function handle the interval.
   611	        if (!container.isConnected) return;
   612	
   613	        const workspace = normaliseWorkspace(rawWorkspace);
   614	        const project   = normaliseProject(rawProject);
   615	
   616	        // Build repo list: [{ repoId, repoName }, …]
   617	        const repos = project.repositories.map((r) => ({
   618	            repoId:   extractRepoId(r),
   619	            repoName: extractRepoName(r),
   620	        })).filter((r) => r.repoId !== '');
   621	
   622	        // Render the view.
   623	        container.innerHTML = '';
   624	
   625	        const isStable = wid === STABLE_WS_ID;
   626	
   627	        container.appendChild(buildHeaderSection(projectId, workspace));
   628	        const { section: statusSection, tbody } = buildStatusTableSection(repos, statusMap || {});
   629	        container.appendChild(statusSection);
   630	        container.appendChild(buildActionsSection(projectId, wid, isStable));
   631	
   632	        // Start polling only when there are repos to update.
   633	        if (tbody && repos.length > 0) {
   634	            pollingInterval = setInterval(async () => {
   635	                // Stop polling if the container is no longer in the DOM.
   636	                if (!container.isConnected) {
   637	                    cleanup();
   638	                    return;
   639	                }
   640	                try {
   641	                    const fresh = await api.status.get(projectId, wid);
   642	                    if (container.isConnected && fresh) {
   643	                        updateStatusTable(tbody, fresh);
   644	                    }
   645	                } catch {
   646	                    // Silently ignore polling errors — the stale badges remain.
   647	                }
   648	            }, POLL_INTERVAL_MS);
   649	        }
   650	    }).catch((err) => {
   651	        if (!container.isConnected) return;
   652	        container.innerHTML = '';
   653	        const errEl = document.createElement('div');
   654	        errEl.className = 'empty-state';
   655	
   656	        const errTitle = document.createElement('h2');
   657	        errTitle.textContent = 'Failed to load workspace';
   658	        errEl.appendChild(errTitle);
   659	
   660	        const errMsg = document.createElement('p');
   661	        errMsg.className   = 'text-secondary';
   662	        errMsg.textContent = err.message || 'An unexpected error occurred.';
   663	        errEl.appendChild(errMsg);
   664	
   665	        const backLink = document.createElement('a');
   666	        backLink.href      = `#/projects/${encodeURIComponent(projectId)}`;
   667	        backLink.className = 'btn btn-secondary';
   668	        backLink.textContent = '← Back to Project';
   669	        if (_router) {
   670	            backLink.addEventListener('click', (e) => {
   671	                e.preventDefault();
   672	                _router.navigate(`#/projects/${encodeURIComponent(projectId)}`);
   673	            });
   674	        }
   675	        errEl.appendChild(backLink);
   676	
   677	        container.appendChild(errEl);
   678	    });
   679	
   680	    // Return cleanup so the router can call it on navigation away.
   681	    return cleanup;
   682	}

## Assistant

Good — I've read the full implementation. Now let me check the existing test infrastructure and look for the API module to understand how to test.

```json
{
  "id": "toolu_01VKGYppc7RhZoDBpvgmTxqd",
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

```json
{
  "id": "toolu_01FeVBKy1MC335ruHnEHMHxA",
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

**Tool call:** `ls` (id: `toolu_01VKGYppc7RhZoDBpvgmTxqd`)

```json
{
  "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE"
}
```

**Tool call:** `ls` (id: `toolu_01FeVBKy1MC335ruHnEHMHxA`)

```json
{
  "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js"
}
```

## Tool Result

['/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/.git/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/.gitignore', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/CHANGELOG.md', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/CONTRIBUTING.md', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/LICENSE', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/README.md', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/config.dist.json', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/dist/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/node_modules/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/package-lock.json', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/package.json', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp011-tests.mjs', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp013-tests.mjs', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp014-tests.mjs', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp015-tests.mjs', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/src/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/tsconfig.json']

## Tool Result

['/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/api.js', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/app.js', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/components/', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/router.js', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/views/']

## Assistant

```json
{
  "id": "toolu_01AtfcQLPTUr1aVNM1ypWD31",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp014-tests.mjs",
    "limit": 80
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01UnWGbTHtBVRRXCB2uSEbUg",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/components"
  },
  "name": "ls",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01AtfcQLPTUr1aVNM1ypWD31`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp014-tests.mjs",
  "limit": 80
}
```

**Tool call:** `ls` (id: `toolu_01UnWGbTHtBVRRXCB2uSEbUg`)

```json
{
  "path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/components"
}
```

## Tool Result

     1	/**
     2	 * QA test suite for WP-014 (WP-005): Project Detail View.
     3	 *
     4	 * Runs entirely in Node.js using jsdom for DOM-dependent code.
     5	 * Tests map to WP-014 Acceptance Criteria.
     6	 *
     7	 * AC1: Navigating to #/projects/:id fetches the project via GET /api/projects/:id
     8	 *      and displays the project's metadata (ID, name, description), repositories
     9	 *      list, and workspaces list.
    10	 * AC2: Adding a repository presents a selection from the global repository list
    11	 *      (filtered to exclude already-added repos) and calls
    12	 *      POST /api/projects/:id/repositories with { repositoryId }.
    13	 * AC3: Removing a repository from the project shows a confirmation dialog and calls
    14	 *      DELETE /api/projects/:id/repositories/:repoId on confirm.
    15	 * AC4: The "Add Workspace" form validates the workspace ID format (2-6 uppercase
    16	 *      A-Z letters) and creates the workspace via POST /api/projects/:id/workspaces.
    17	 * AC5: The STABLE workspace's delete button is visually disabled and non-functional.
    18	 * AC6: Renaming a project shows a confirmation dialog, calls
    19	 *      PUT /api/projects/:id/rename with { newId }, and navigates to
    20	 *      #/projects/:newId on success.
    21	 */
    22	
    23	import { JSDOM } from 'jsdom';
    24	import { strict as assert } from 'assert';
    25	import { readFileSync } from 'fs';
    26	import { fileURLToPath } from 'url';
    27	import { dirname, resolve } from 'path';
    28	
    29	const __filename = fileURLToPath(import.meta.url);
    30	const __dirname = dirname(__filename);
    31	
    32	// ─────────────────────────────────────────────────────────────────────────────
    33	// Test harness
    34	// ─────────────────────────────────────────────────────────────────────────────
    35	
    36	let passed = 0;
    37	let failed = 0;
    38	const failures = [];
    39	const promises = [];
    40	
    41	function test(name, fn) {
    42	    try {
    43	        const result = fn();
    44	        if (result && typeof result.then === 'function') {
    45	            const p = result.then(() => {
    46	                console.log(`  ✅  ${name}`);
    47	                passed++;
    48	            }).catch((err) => {
    49	                console.error(`  ❌  ${name}`);
    50	                console.error(`      ${err.message}`);
    51	                failed++;
    52	                failures.push({ name, error: err.message });
    53	            });
    54	            promises.push(p);
    55	            return p;
    56	        }
    57	        console.log(`  ✅  ${name}`);
    58	        passed++;
    59	        return Promise.resolve();
    60	    } catch (err) {
    61	        console.error(`  ❌  ${name}`);
    62	        console.error(`      ${err.message}`);
    63	        failed++;
    64	        failures.push({ name, error: err.message });
    65	        return Promise.resolve();
    66	    }
    67	}
    68	
    69	function section(title) {
    70	    console.log(`\n━━ ${title} ━━`);
    71	}
    72	
    73	// ─────────────────────────────────────────────────────────────────────────────
    74	// Helpers: read and strip ES module syntax for jsdom eval
    75	// ─────────────────────────────────────────────────────────────────────────────
    76	
    77	function readAndStrip(filePath) {
    78	    let code = readFileSync(resolve(__dirname, filePath), 'utf8');
    79	    // Remove import statements
    80	    code = code.replace(/^import .+from .+;?\s*$/gm, '');

## Tool Result

['/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/components/confirm-dialog.js', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/components/form-helpers.js', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/components/status-badge.js', '/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/components/toast.js']

## Assistant

```json
{
  "id": "toolu_01VRSU18wbioheEPA1iabp7T",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp014-tests.mjs",
    "offset": 80,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01VRSU18wbioheEPA1iabp7T`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp014-tests.mjs",
  "offset": 80,
  "limit": 200
}
```

## Tool Result

    81	    code = code.replace(/^import .+$/gm, '');
    82	    // export class Foo → class Foo (assign to window at end)
    83	    code = code.replace(/^export class (\w+)/gm, 'class $1');
    84	    const classNames = [];
    85	    const classMatches = [...code.matchAll(/^class (\w+)/gm)];
    86	    for (const m of classMatches) classNames.push(m[1]);
    87	    for (const name of classNames) { code += `\nwindow.${name} = ${name};`; }
    88	    // export async function / export function
    89	    code = code.replace(/^export (async function) (\w+)/gm, 'window.$2 = $1 $2');
    90	    code = code.replace(/^export function (\w+)/gm, 'window.$1 = function $1');
    91	    // export const/let/var — strip export, assign to window at end
    92	    const exportedVars = [];
    93	    code = code.replace(/^export (const|let|var) (\w+)/gm, (match, kw, name) => {
    94	        exportedVars.push(name);
    95	        return `${kw} ${name}`;
    96	    });
    97	    for (const name of exportedVars) { code += `\nwindow.${name} = ${name};`; }
    98	    return code;
    99	}
   100	
   101	function makeDOM() {
   102	    return new JSDOM(
   103	        `<!DOCTYPE html><html><body>
   104	            <div id="app"></div>
   105	            <div id="toast-container"></div>
   106	        </body></html>`,
   107	        { runScripts: 'dangerously', pretendToBeVisual: true }
   108	    );
   109	}
   110	
   111	/**
   112	 * Build a jsdom window with all dependencies loaded (toast, confirm-dialog,
   113	 * form-helpers, api, project-detail).
   114	 *
   115	 * @param {function} fetchImpl - Mock fetch implementation.
   116	 * @returns {Window}
   117	 */
   118	function buildWindowWithDeps(fetchImpl) {
   119	    const dom = makeDOM();
   120	    const w = dom.window;
   121	    w.fetch = fetchImpl;
   122	
   123	    // Polyfill CSS.escape for jsdom
   124	    if (!w.CSS) w.CSS = {};
   125	    if (!w.CSS.escape) {
   126	        w.CSS.escape = (value) => String(value).replace(/[^\w-]/g, c => '\\' + c);
   127	    }
   128	
   129	    w.eval(readAndStrip('gui/public/js/components/toast.js'));
   130	    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));
   131	    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));
   132	    w.eval(readAndStrip('gui/public/js/api.js'));
   133	    w.eval(readAndStrip('gui/public/js/views/project-detail.js'));
   134	
   135	    return w;
   136	}
   137	
   138	// ─────────────────────────────────────────────────────────────────────────────
   139	// Sample data
   140	// ─────────────────────────────────────────────────────────────────────────────
   141	
   142	const sampleProject = {
   143	    Id: 'my-project',
   144	    Name: 'My Project',
   145	    Description: 'A test project',
   146	    Repositories: ['repo-alpha', 'repo-beta'],
   147	};
   148	
   149	const sampleAllRepos = [
   150	    { Id: 'repo-alpha', Name: 'Alpha Repo', Url: 'https://github.com/org/alpha.git' },
   151	    { Id: 'repo-beta',  Name: 'Beta Repo',  Url: 'https://github.com/org/beta.git'  },
   152	    { Id: 'repo-gamma', Name: 'Gamma Repo', Url: 'https://github.com/org/gamma.git' },
   153	];
   154	
   155	const sampleWorkspaces = [
   156	    { Id: 'STABLE', Description: 'Stable workspace', CreatedAt: '2024-01-01T00:00:00Z' },
   157	    { Id: 'DEV',    Description: 'Dev workspace',    CreatedAt: '2024-01-02T00:00:00Z' },
   158	];
   159	
   160	/**
   161	 * Build a standard mock fetch that handles the three data-fetching calls made
   162	 * by renderProjectDetail: project GET, workspaces GET, repositories GET.
   163	 *
   164	 * @param {Object} [overrides] - { project, workspaces, allRepos }
   165	 * @param {function} [onCall]  - Optional callback invoked with (url, opts) for each fetch.
   166	 */
   167	function buildStandardFetch(overrides = {}, onCall = null) {
   168	    const project    = overrides.project    || sampleProject;
   169	    const workspaces = overrides.workspaces || sampleWorkspaces;
   170	    const allRepos   = overrides.allRepos   || sampleAllRepos;
   171	
   172	    return async (url, opts) => {
   173	        if (onCall) onCall(url, opts);
   174	        const method = (opts || {}).method || 'GET';
   175	
   176	        if (method === 'GET') {
   177	            // /api/projects/:id/workspaces
   178	            if (url.includes('/workspaces') && !url.includes('/branches')) {
   179	                return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => workspaces };
   180	            }
   181	            // /api/projects/:id (single project)
   182	            if (/\/api\/projects\/[^/]+$/.test(url)) {
   183	                return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => project };
   184	            }
   185	            // /api/repositories (global list)
   186	            if (url === '/api/repositories') {
   187	                return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => allRepos };
   188	            }
   189	        }
   190	
   191	        // Default: 200 OK with empty object (for mutations)
   192	        return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => ({}) };
   193	    };
   194	}
   195	
   196	// ─────────────────────────────────────────────────────────────────────────────
   197	// Static analysis
   198	// ─────────────────────────────────────────────────────────────────────────────
   199	
   200	section('Static analysis — project-detail.js');
   201	
   202	const pdSrc = readFileSync(resolve(__dirname, 'gui/public/js/views/project-detail.js'), 'utf8');
   203	
   204	test('project-detail.js imports api from api.js', () => {
   205	    assert.ok(pdSrc.includes("from '../api.js'"), 'api.js import missing');
   206	});
   207	
   208	test('project-detail.js imports showToast', () => {
   209	    assert.ok(pdSrc.includes('showToast'), 'showToast import missing');
   210	});
   211	
   212	test('project-detail.js imports showConfirm', () => {
   213	    assert.ok(pdSrc.includes('showConfirm'), 'showConfirm import missing');
   214	});
   215	
   216	test('project-detail.js exports renderProjectDetail', () => {
   217	    assert.ok(pdSrc.includes('export async function renderProjectDetail'), 'renderProjectDetail export missing');
   218	});
   219	
   220	test('project-detail.js exports setRouter', () => {
   221	    assert.ok(pdSrc.includes('export function setRouter'), 'setRouter export missing');
   222	});
   223	
   224	test('project-detail.js calls api.projects.get()', () => {
   225	    assert.ok(pdSrc.includes('api.projects.get('), 'api.projects.get() call missing');
   226	});
   227	
   228	test('project-detail.js calls api.workspaces.list()', () => {
   229	    assert.ok(pdSrc.includes('api.workspaces.list('), 'api.workspaces.list() call missing');
   230	});
   231	
   232	test('project-detail.js calls api.repositories.list()', () => {
   233	    assert.ok(pdSrc.includes('api.repositories.list()'), 'api.repositories.list() call missing');
   234	});
   235	
   236	test('project-detail.js calls api.projects.addRepository()', () => {
   237	    assert.ok(pdSrc.includes('api.projects.addRepository('), 'api.projects.addRepository() call missing');
   238	});
   239	
   240	test('project-detail.js calls api.projects.removeRepository()', () => {
   241	    assert.ok(pdSrc.includes('api.projects.removeRepository('), 'api.projects.removeRepository() call missing');
   242	});
   243	
   244	test('project-detail.js calls api.workspaces.create()', () => {
   245	    assert.ok(pdSrc.includes('api.workspaces.create('), 'api.workspaces.create() call missing');
   246	});
   247	
   248	test('project-detail.js calls api.projects.rename()', () => {
   249	    assert.ok(pdSrc.includes('api.projects.rename('), 'api.projects.rename() call missing');
   250	});
   251	
   252	test('project-detail.js calls api.projects.delete()', () => {
   253	    assert.ok(pdSrc.includes('api.projects.delete('), 'api.projects.delete() call missing');
   254	});
   255	
   256	test('project-detail.js validates workspace ID with /^[A-Z]{2,6}$/ regex', () => {
   257	    assert.ok(
   258	        pdSrc.includes('[A-Z]{2,6}'),
   259	        'Workspace ID regex /^[A-Z]{2,6}$/ not found in project-detail.js'
   260	    );
   261	});
   262	
   263	test('project-detail.js handles STABLE workspace — disables delete button', () => {
   264	    assert.ok(
   265	        pdSrc.includes('STABLE'),
   266	        "STABLE constant not found — STABLE workspace handling missing"
   267	    );
   268	    assert.ok(
   269	        pdSrc.includes('disabled') || pdSrc.includes('.disabled'),
   270	        'No disabled property set for STABLE workspace delete button'
   271	    );
   272	});
   273	
   274	test('app.js registers renderProjectDetail for #/projects/:id', () => {
   275	    const appSrc = readFileSync(resolve(__dirname, 'gui/public/js/app.js'), 'utf8');
   276	    assert.ok(appSrc.includes('renderProjectDetail'), 'renderProjectDetail not imported/used in app.js');
   277	    const lines = appSrc.split('\n');
   278	    const registerCallLine = lines.find(l =>
   279	        (l.includes("'#/projects/:id'") || l.includes('"#/projects/:id"')) &&
   280	        l.includes('renderProjectDetail')

## Assistant

Now I have a solid understanding of the test pattern. Let me look at the status-badge component and API module, then write the test suite.

```json
{
  "id": "toolu_0135xXUtMof4vD9Yi1TJ5onf",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/components/status-badge.js"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01X17EUcUdLBeSEkQvFNoQvj",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/api.js",
    "limit": 80
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_0135xXUtMof4vD9Yi1TJ5onf`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/components/status-badge.js"
}
```

**Tool call:** `read_file` (id: `toolu_01X17EUcUdLBeSEkQvFNoQvj`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/api.js",
  "limit": 80
}
```

## Tool Result

     1	/**
     2	 * Status Badge Component.
     3	 *
     4	 * Creates a DOM element summarising the git status of a single repository
     5	 * inside a workspace.  CSS classes used here are all defined in styles.css.
     6	 *
     7	 * Usage:
     8	 *   import { createStatusBadge } from './components/status-badge.js';
     9	 *
    10	 *   const badge = createStatusBadge(gitStatusInfo);
    11	 *   container.appendChild(badge);
    12	 *
    13	 * @typedef {Object} GitStatusInfo
    14	 * @property {string|null}  currentBranch     - Active branch name, or null for detached HEAD.
    15	 * @property {number}       localCommits      - Commits ahead of remote.
    16	 * @property {number}       unfetchedCommits  - Commits behind remote (unfetched).
    17	 * @property {number}       modifiedFiles     - Number of modified/staged files.
    18	 * @property {string|null}  lastActivity      - ISO timestamp of last commit, or null.
    19	 * @property {boolean}      hasConflicts      - True when merge conflicts exist.
    20	 */
    21	
    22	// ---------------------------------------------------------------------------
    23	// Helpers
    24	// ---------------------------------------------------------------------------
    25	
    26	/**
    27	 * Format an ISO timestamp into a human-readable relative or absolute string.
    28	 * Returns an empty string when the input is falsy.
    29	 *
    30	 * @param {string|null} isoTimestamp
    31	 * @returns {string}
    32	 */
    33	function formatLastActivity(isoTimestamp) {
    34	    if (!isoTimestamp) return '';
    35	
    36	    const date = new Date(isoTimestamp);
    37	    if (isNaN(date.getTime())) return isoTimestamp; // pass through if unparseable
    38	
    39	    const now = Date.now();
    40	    const diffMs = now - date.getTime();
    41	    const diffMinutes = Math.floor(diffMs / 60_000);
    42	    const diffHours = Math.floor(diffMinutes / 60);
    43	    const diffDays = Math.floor(diffHours / 24);
    44	
    45	    if (diffMinutes < 1)  return 'just now';
    46	    if (diffMinutes < 60) return `${diffMinutes}m ago`;
    47	    if (diffHours < 24)   return `${diffHours}h ago`;
    48	    if (diffDays < 7)     return `${diffDays}d ago`;
    49	
    50	    // Fall back to locale date string for older commits.
    51	    return date.toLocaleDateString();
    52	}
    53	
    54	/**
    55	 * Determine the primary CSS modifier class for the badge based on status
    56	 * priority: conflicts > modified > ahead/behind > clean.
    57	 *
    58	 * @param {GitStatusInfo} info
    59	 * @returns {string} One of: 'status-badge-conflict' | 'status-badge-modified' |
    60	 *   'status-badge-ahead' | 'status-badge-behind' | 'status-badge-clean'
    61	 */
    62	function resolveBadgeClass(info) {
    63	    if (info.hasConflicts)                                 return 'status-badge-conflict';
    64	    if (info.modifiedFiles > 0)                            return 'status-badge-modified';
    65	    if (info.localCommits > 0)                             return 'status-badge-ahead';
    66	    if (info.unfetchedCommits > 0)                         return 'status-badge-behind';
    67	    return 'status-badge-clean';
    68	}
    69	
    70	// ---------------------------------------------------------------------------
    71	// Public export
    72	// ---------------------------------------------------------------------------
    73	
    74	/**
    75	 * Create a status badge DOM element for a git repository.
    76	 *
    77	 * The returned element is a `<div>` that contains:
    78	 * - A coloured pill showing the branch name.
    79	 * - Secondary detail chips: modified count, commits ahead/behind, last
    80	 *   activity, and a conflict warning (each only shown when non-zero / present).
    81	 *
    82	 * When `gitStatusInfo` is `null` a compact "No data" element is returned.
    83	 *
    84	 * @param {GitStatusInfo|null} gitStatusInfo
    85	 * @returns {HTMLElement}
    86	 */
    87	export function createStatusBadge(gitStatusInfo) {
    88	    const wrapper = document.createElement('div');
    89	    wrapper.className = 'status-badge-wrapper';
    90	
    91	    // ------------------------------------------------------------------
    92	    // Null / loading state
    93	    // ------------------------------------------------------------------
    94	    if (!gitStatusInfo) {
    95	        const noData = document.createElement('span');
    96	        noData.className = 'status-badge status-badge-error';
    97	
    98	        const dot = document.createElement('span');
    99	        dot.className = 'status-badge-dot';
   100	

## Tool Result

     1	/**
     2	 * API Client for Repo Parallelizer GUI.
     3	 *
     4	 * Centralises all HTTP communication with the backend REST API.
     5	 * All methods return Promises. Non-2xx responses throw an Error whose
     6	 * message is taken from the `error` field in the JSON response body.
     7	 *
     8	 * Usage:
     9	 *   import { api } from './api.js';
    10	 *
    11	 *   const repos = await api.repositories.list();
    12	 *   const project = await api.projects.get('my-project');
    13	 */
    14	
    15	// ---------------------------------------------------------------------------
    16	// Internal HTTP helper
    17	// ---------------------------------------------------------------------------
    18	
    19	/**
    20	 * Perform a fetch request and return the parsed JSON body.
    21	 *
    22	 * For 204 No Content responses the Promise resolves with `undefined`.
    23	 * For non-2xx responses, an Error is thrown whose message comes from
    24	 * the `error` field in the JSON response body (falling back to the HTTP
    25	 * status text if the body cannot be parsed).
    26	 *
    27	 * @param {string} method - HTTP method (GET, POST, PUT, DELETE, …).
    28	 * @param {string} url    - Absolute or relative URL.
    29	 * @param {Object} [body] - Optional request body (serialised as JSON).
    30	 * @returns {Promise<*>}
    31	 */
    32	async function request(method, url, body) {
    33	    /** @type {RequestInit} */
    34	    const options = {
    35	        method,
    36	        headers: {
    37	            'Content-Type': 'application/json',
    38	            'Accept': 'application/json',
    39	        },
    40	    };
    41	
    42	    if (body !== undefined) {
    43	        options.body = JSON.stringify(body);
    44	    }
    45	
    46	    const response = await fetch(url, options);
    47	
    48	    // 204 No Content — nothing to parse.
    49	    if (response.status === 204) {
    50	        return undefined;
    51	    }
    52	
    53	    // Attempt to parse JSON for all other responses.
    54	    let json;
    55	    const contentType = response.headers.get('Content-Type') || '';
    56	    if (contentType.includes('application/json')) {
    57	        json = await response.json();
    58	    }
    59	
    60	    if (!response.ok) {
    61	        const message =
    62	            (json && json.error) ? json.error : response.statusText;
    63	        throw new Error(message);
    64	    }
    65	
    66	    return json;
    67	}
    68	
    69	// ---------------------------------------------------------------------------
    70	// Shared type definitions
    71	// ---------------------------------------------------------------------------
    72	
    73	/**
    74	 * A project object as returned by the backend REST API.
    75	 *
    76	 * The Go backend serialises struct fields using their Go-style capitalised
    77	 * names (`Id`, `Name`, `Description`, `Repositories`). Future serialiser
    78	 * changes may emit lowercase equivalents (`id`, `name`, `description`,
    79	 * `repositories`). View code **must** normalise both casings:
    80	 *

## Assistant

```json
{
  "id": "toolu_01Jxok2QLtvYkrsMxTFWQKK2",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/api.js",
    "offset": 80,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01Jxok2QLtvYkrsMxTFWQKK2`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/api.js",
  "offset": 80,
  "limit": 200
}
```

## Tool Result

    81	 * ```js
    82	 * const id   = project.Id   || project.id   || '';
    83	 * const name = project.Name || project.name || id;
    84	 * ```
    85	 *
    86	 * @typedef {Object} ProjectResponse
    87	 * @property {string}   [Id]            - Project ID (Go-capitalised key).
    88	 * @property {string}   [id]            - Project ID (lowercase key).
    89	 * @property {string}   [Name]          - Human-readable project name (Go-capitalised key).
    90	 * @property {string}   [name]          - Human-readable project name (lowercase key).
    91	 * @property {string}   [Description]   - Optional project description (Go-capitalised key).
    92	 * @property {string}   [description]   - Optional project description (lowercase key).
    93	 * @property {Array}    [Repositories]  - Array of associated repository objects (Go-capitalised key).
    94	 * @property {Array}    [repositories]  - Array of associated repository objects (lowercase key).
    95	 */
    96	
    97	// ---------------------------------------------------------------------------
    98	// API namespaces
    99	// ---------------------------------------------------------------------------
   100	
   101	/**
   102	 * Repository endpoints.
   103	 *
   104	 * @namespace api.repositories
   105	 */
   106	const repositories = {
   107	    /**
   108	     * List all registered repositories.
   109	     * @returns {Promise<Object[]>}
   110	     */
   111	    list() {
   112	        return request('GET', '/api/repositories');
   113	    },
   114	
   115	    /**
   116	     * Get a single repository by ID.
   117	     * @param {string} id
   118	     * @returns {Promise<Object>}
   119	     */
   120	    get(id) {
   121	        return request('GET', `/api/repositories/${encodeURIComponent(id)}`);
   122	    },
   123	
   124	    /**
   125	     * Register a new repository.
   126	     * @param {{ url: string, name?: string, id?: string }} data
   127	     * @returns {Promise<Object>} The created repository (HTTP 201).
   128	     */
   129	    create(data) {
   130	        return request('POST', '/api/repositories', data);
   131	    },
   132	
   133	    /**
   134	     * Update a repository's metadata.
   135	     * @param {string} id
   136	     * @param {{ name: string }} data
   137	     * @returns {Promise<Object>}
   138	     */
   139	    update(id, data) {
   140	        return request('PUT', `/api/repositories/${encodeURIComponent(id)}`, data);
   141	    },
   142	
   143	    /**
   144	     * Delete a repository.
   145	     * @param {string} id
   146	     * @returns {Promise<void>}
   147	     */
   148	    delete(id) {
   149	        return request('DELETE', `/api/repositories/${encodeURIComponent(id)}`);
   150	    },
   151	};
   152	
   153	/**
   154	 * Project endpoints.
   155	 *
   156	 * @namespace api.projects
   157	 */
   158	const projects = {
   159	    /**
   160	     * List all projects.
   161	     * @returns {Promise<ProjectResponse[]>}
   162	     */
   163	    list() {
   164	        return request('GET', '/api/projects');
   165	    },
   166	
   167	    /**
   168	     * Get a single project by ID.
   169	     * @param {string} id
   170	     * @returns {Promise<ProjectResponse>}
   171	     */
   172	    get(id) {
   173	        return request('GET', `/api/projects/${encodeURIComponent(id)}`);
   174	    },
   175	
   176	    /**
   177	     * Create a new project.
   178	     * @param {{ name: string, repositoryIds?: string[], description?: string, id?: string }} data
   179	     * @returns {Promise<ProjectResponse>} The created project (HTTP 201).
   180	     */
   181	    create(data) {
   182	        return request('POST', '/api/projects', data);
   183	    },
   184	
   185	    /**
   186	     * Update a project's metadata.
   187	     * @param {string} id
   188	     * @param {{ name?: string, description?: string }} data
   189	     * @returns {Promise<ProjectResponse>}
   190	     */
   191	    update(id, data) {
   192	        return request('PUT', `/api/projects/${encodeURIComponent(id)}`, data);
   193	    },
   194	
   195	    /**
   196	     * Rename a project (changes its ID).
   197	     * @param {string} id    - Current project ID.
   198	     * @param {string} newId - Desired new project ID.
   199	     * @returns {Promise<ProjectResponse>}
   200	     */
   201	    rename(id, newId) {
   202	        return request('PUT', `/api/projects/${encodeURIComponent(id)}/rename`, { newId });
   203	    },
   204	
   205	    /**
   206	     * Delete a project.
   207	     * @param {string} id
   208	     * @returns {Promise<void>}
   209	     */
   210	    delete(id) {
   211	        return request('DELETE', `/api/projects/${encodeURIComponent(id)}`);
   212	    },
   213	
   214	    /**
   215	     * Add a repository to a project.
   216	     * @param {string} projectId
   217	     * @param {string} repoId
   218	     * @returns {Promise<ProjectResponse>} The updated project.
   219	     */
   220	    addRepository(projectId, repoId) {
   221	        return request(
   222	            'POST',
   223	            `/api/projects/${encodeURIComponent(projectId)}/repositories`,
   224	            { repositoryId: repoId },
   225	        );
   226	    },
   227	
   228	    /**
   229	     * Remove a repository from a project.
   230	     * @param {string} projectId
   231	     * @param {string} repoId
   232	     * @returns {Promise<void>}
   233	     */
   234	    removeRepository(projectId, repoId) {
   235	        return request(
   236	            'DELETE',
   237	            `/api/projects/${encodeURIComponent(projectId)}/repositories/${encodeURIComponent(repoId)}`,
   238	        );
   239	    },
   240	};
   241	
   242	/**
   243	 * Workspace endpoints.
   244	 *
   245	 * @namespace api.workspaces
   246	 */
   247	const workspaces = {
   248	    /**
   249	     * List all workspaces for a project.
   250	     * @param {string} projectId
   251	     * @returns {Promise<Object[]>}
   252	     */
   253	    list(projectId) {
   254	        return request('GET', `/api/projects/${encodeURIComponent(projectId)}/workspaces`);
   255	    },
   256	
   257	    /**
   258	     * Get a single workspace.
   259	     * @param {string} projectId
   260	     * @param {string} wid - Workspace ID.
   261	     * @returns {Promise<Object>}
   262	     */
   263	    get(projectId, wid) {
   264	        return request(
   265	            'GET',
   266	            `/api/projects/${encodeURIComponent(projectId)}/workspaces/${encodeURIComponent(wid)}`,
   267	        );
   268	    },
   269	
   270	    /**
   271	     * Create a new workspace inside a project.
   272	     * @param {string} projectId
   273	     * @param {{ workspaceId: string, description?: string }} data
   274	     * @returns {Promise<Object>} The created workspace (HTTP 201).
   275	     */
   276	    create(projectId, data) {
   277	        return request(
   278	            'POST',
   279	            `/api/projects/${encodeURIComponent(projectId)}/workspaces`,
   280	            data,

## Assistant

```json
{
  "id": "toolu_01FAWiTAz9Awe6mnox5RrGAF",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/api.js",
    "offset": 280,
    "limit": 100
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01FAWiTAz9Awe6mnox5RrGAF`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/api.js",
  "offset": 280,
  "limit": 100
}
```

## Tool Result

   281	        );
   282	    },
   283	
   284	    /**
   285	     * Update a workspace's metadata.
   286	     * @param {string} projectId
   287	     * @param {string} wid
   288	     * @param {{ description: string }} data
   289	     * @returns {Promise<Object>}
   290	     */
   291	    update(projectId, wid, data) {
   292	        return request(
   293	            'PUT',
   294	            `/api/projects/${encodeURIComponent(projectId)}/workspaces/${encodeURIComponent(wid)}`,
   295	            data,
   296	        );
   297	    },
   298	
   299	    /**
   300	     * Rename a workspace (changes its ID).
   301	     * @param {string} projectId
   302	     * @param {string} wid    - Current workspace ID.
   303	     * @param {string} newId  - Desired new workspace ID.
   304	     * @returns {Promise<Object>}
   305	     */
   306	    rename(projectId, wid, newId) {
   307	        return request(
   308	            'PUT',
   309	            `/api/projects/${encodeURIComponent(projectId)}/workspaces/${encodeURIComponent(wid)}/rename`,
   310	            { newId },
   311	        );
   312	    },
   313	
   314	    /**
   315	     * Delete a workspace.
   316	     * @param {string} projectId
   317	     * @param {string} wid
   318	     * @returns {Promise<void>}
   319	     */
   320	    delete(projectId, wid) {
   321	        return request(
   322	            'DELETE',
   323	            `/api/projects/${encodeURIComponent(projectId)}/workspaces/${encodeURIComponent(wid)}`,
   324	        );
   325	    },
   326	};
   327	
   328	/**
   329	 * Branch endpoints.
   330	 *
   331	 * @namespace api.branches
   332	 */
   333	const branches = {
   334	    /**
   335	     * List branches for all repositories in a workspace.
   336	     *
   337	     * @param {string} projectId
   338	     * @param {string} wid
   339	     * @returns {Promise<{
   340	     *   branches: Record<string, Array<{name: string, isCurrent: boolean, isRemote: boolean, upstream?: string}>>,
   341	     *   suggestions: string[]
   342	     * }>}
   343	     */
   344	    list(projectId, wid) {
   345	        return request(
   346	            'GET',
   347	            `/api/projects/${encodeURIComponent(projectId)}/workspaces/${encodeURIComponent(wid)}/branches`,
   348	        );
   349	    },
   350	
   351	    /**
   352	     * Switch branches across repositories in a workspace.
   353	     *
   354	     * @param {string} projectId
   355	     * @param {string} wid
   356	     * @param {Record<string, string>} assignments - Map of repoId → branchName.
   357	     * @returns {Promise<{results: Record<string, {success: boolean, conflict: boolean, error?: string}>}>}
   358	     */
   359	    switch(projectId, wid, assignments) {
   360	        return request(
   361	            'POST',
   362	            `/api/projects/${encodeURIComponent(projectId)}/workspaces/${encodeURIComponent(wid)}/branches/switch`,
   363	            { assignments },
   364	        );
   365	    },
   366	};
   367	
   368	/**
   369	 * Status endpoints.
   370	 *
   371	 * @namespace api.status
   372	 */
   373	const status = {
   374	    /**
   375	     * Get the current git status for all repositories in a workspace.
   376	     *
   377	     * @param {string} projectId
   378	     * @param {string} wid
   379	     * @returns {Promise<Record<string, {
   380	     *   currentBranch: string|null,

## Assistant

```json
{
  "id": "toolu_01RfkY7g5JWDh1Gon4vudY9Y",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/api.js",
    "offset": 380,
    "limit": 40
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01RfkY7g5JWDh1Gon4vudY9Y`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/gui/public/js/api.js",
  "offset": 380,
  "limit": 40
}
```

## Tool Result

   381	     *   localCommits: number,
   382	     *   unfetchedCommits: number,
   383	     *   modifiedFiles: number,
   384	     *   lastActivity: string|null,
   385	     *   hasConflicts: boolean
   386	     * }|null>>} Keyed by repository ID.
   387	     */
   388	    get(projectId, wid) {
   389	        return request(
   390	            'GET',
   391	            `/api/projects/${encodeURIComponent(projectId)}/workspaces/${encodeURIComponent(wid)}/status`,
   392	        );
   393	    },
   394	
   395	    /**
   396	     * Force-refresh git status for all repositories in a workspace.
   397	     * Returns the same shape as {@link api.status.get} but with freshly polled data.
   398	     *
   399	     * @param {string} projectId
   400	     * @param {string} wid
   401	     * @returns {Promise<Record<string, Object|null>>}
   402	     */
   403	    refresh(projectId, wid) {
   404	        return request(
   405	            'POST',
   406	            `/api/projects/${encodeURIComponent(projectId)}/workspaces/${encodeURIComponent(wid)}/status/refresh`,
   407	        );
   408	    },
   409	};
   410	
   411	// ---------------------------------------------------------------------------
   412	// Public export
   413	// ---------------------------------------------------------------------------
   414	
   415	/**
   416	 * Namespaced API client.
   417	 *
   418	 * @type {{
   419	 *   repositories: typeof repositories,
   420	 *   projects:     typeof projects,

## Assistant

I now have everything I need. Let me also check an existing passing test file (qa-wp015) to see the full pattern, then run the existing server tests to get a baseline.

```json
{
  "id": "toolu_01HeGB1q2pe8GVMXHjqe459Y",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp015-tests.mjs",
    "limit": 50
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_013M7eZLouuBM7ERj1b8m8KW",
  "caller": {
    "type": "direct"
  },
  "input": {
    "command": "cd /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE && node --check gui/public/js/views/workspace-detail.js 2>&1; echo \"EXIT:$?\""
  },
  "name": "execute",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01HeGB1q2pe8GVMXHjqe459Y`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp015-tests.mjs",
  "limit": 50
}
```

**Tool call:** `execute` (id: `toolu_013M7eZLouuBM7ERj1b8m8KW`)

```json
{
  "command": "cd /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE && node --check gui/public/js/views/workspace-detail.js 2>&1; echo \"EXIT:$?\""
}
```

## Tool Result

     1	/**
     2	 * QA test suite for WP-015 (WP-004): Repositories View.
     3	 *
     4	 * Runs entirely in Node.js using jsdom for DOM-dependent code.
     5	 * Tests map to WP-015 Acceptance Criteria.
     6	 *
     7	 * AC1: Navigating to #/repositories renders a table listing all repositories
     8	 *      with ID, Name, and URL columns fetched from GET /api/repositories.
     9	 * AC2: The "Add Repository" form validates that URL is provided (required),
    10	 *      accepts optional Name and ID fields, calls POST /api/repositories on
    11	 *      submit, shows a success toast, and refreshes the table.
    12	 * AC3: Inline edit functionality allows changing a repository's Name and
    13	 *      persists the change via PUT /api/repositories/:id.
    14	 * AC4: Clicking Delete shows a confirmation dialog warning that the repo will
    15	 *      be removed from all projects; confirming calls DELETE /api/repositories/:id
    16	 *      and removes the row.
    17	 * AC5: API errors during any CRUD operation are displayed as error toasts via
    18	 *      showToast().
    19	 */
    20	
    21	import { JSDOM } from 'jsdom';
    22	import { strict as assert } from 'assert';
    23	import { readFileSync } from 'fs';
    24	import { fileURLToPath } from 'url';
    25	import { dirname, resolve } from 'path';
    26	
    27	const __filename = fileURLToPath(import.meta.url);
    28	const __dirname = dirname(__filename);
    29	
    30	// ─────────────────────────────────────────────────────────────────────────────
    31	// Test harness
    32	// ─────────────────────────────────────────────────────────────────────────────
    33	
    34	let passed = 0;
    35	let failed = 0;
    36	const failures = [];
    37	const promises = [];
    38	
    39	function test(name, fn) {
    40	    try {
    41	        const result = fn();
    42	        if (result && typeof result.then === 'function') {
    43	            const p = result.then(() => {
    44	                console.log(`  ✅  ${name}`);
    45	                passed++;
    46	            }).catch((err) => {
    47	                console.error(`  ❌  ${name}`);
    48	                console.error(`      ${err.message}`);
    49	                failed++;
    50	                failures.push({ name, error: err.message });

## Tool Result

EXIT:0

[Command succeeded with exit code 0]

## Assistant

```json
{
  "id": "toolu_01XVYhDLefGCP5rqSjUUDneu",
  "caller": {
    "type": "direct"
  },
  "input": {
    "command": "cd /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE && node --check gui/public/js/app.js 2>&1; echo \"EXIT:$?\""
  },
  "name": "execute",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01F7RciBCDcYP1Xufoah1qcA",
  "caller": {
    "type": "direct"
  },
  "input": {
    "command": "cd /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE && npm test 2>&1 | tail -20"
  },
  "name": "execute",
  "type": "tool_use"
}
```

**Tool call:** `execute` (id: `toolu_01XVYhDLefGCP5rqSjUUDneu`)

```json
{
  "command": "cd /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE && node --check gui/public/js/app.js 2>&1; echo \"EXIT:$?\""
}
```

**Tool call:** `execute` (id: `toolu_01F7RciBCDcYP1Xufoah1qcA`)

```json
{
  "command": "cd /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE && npm test 2>&1 | tail -20"
}
```

## Tool Result

EXIT:0

[Command succeeded with exit code 0]

## Tool Result

✔ rename throws when new workspace ID already exists (1.840167ms)
✔ rename returns WorkspaceInfo with the new ID (5.022083ms)
✔ rename updates DateModified on the workspace entry (3.796583ms)
✔ rename removes the old workspace ID from the project (6.097084ms)
✔ remove throws when attempting to delete the STABLE workspace (3.017042ms)
✔ remove throws when project does not exist (2.209125ms)
✔ remove throws NotFoundError when project does not exist (1.794917ms)
✔ remove throws when workspace does not exist (2.232042ms)
✔ remove deletes the workspace so list() no longer includes it (1.14775ms)
✔ remove does not affect other workspaces in the project (1.077292ms)
✔ CRUD round-trip: create → list → getById → update → rename → remove (4.031666ms)
✔ workspace operations on project A do not affect project B workspaces (3.028709ms)
ℹ tests 517
ℹ suites 0
ℹ pass 517
ℹ fail 0
ℹ cancelled 0
ℹ skipped 0
ℹ todo 0
ℹ duration_ms 7940.911583

[Command succeeded with exit code 0]

## Assistant

517 server tests all pass. Now I'll write the QA test suite for WP-016.

```json
{
  "id": "toolu_012kxVLf2CmyMVa8jWXmJhQK",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Write QA test suite qa-wp016-tests.mjs covering all 6 ACs",
        "status": "in_progress"
      },
      {
        "content": "Run the test suite and verify all tests pass",
        "status": "pending"
      },
      {
        "content": "Run full server test regression and verify 517 still pass",
        "status": "pending"
      },
      {
        "content": "Complete the QA pipeline in the ledger",
        "status": "pending"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_012kxVLf2CmyMVa8jWXmJhQK`)

```json
{
  "todos": [
    {
      "content": "Write QA test suite qa-wp016-tests.mjs covering all 6 ACs",
      "status": "in_progress"
    },
    {
      "content": "Run the test suite and verify all tests pass",
      "status": "pending"
    },
    {
      "content": "Run full server test regression and verify 517 still pass",
      "status": "pending"
    },
    {
      "content": "Complete the QA pipeline in the ledger",
      "status": "pending"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Write QA test suite qa-wp016-tests.mjs covering all 6 ACs', 'status': 'in_progress'}, {'content': 'Run the test suite and verify all tests pass', 'status': 'pending'}, {'content': 'Run full server test regression and verify 517 still pass', 'status': 'pending'}, {'content': 'Complete the QA pipeline in the ledger', 'status': 'pending'}]

## Assistant

```json
{
  "id": "toolu_01CtVVtYKwRbaVeUiuLgWuYw",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp016-tests.mjs",
    "content": "/**\n * QA test suite for WP-016 (WP-006): Workspace Detail View with Git Status Polling.\n *\n * Runs entirely in Node.js using jsdom for DOM-dependent code.\n * Tests map to WP-016 Acceptance Criteria.\n *\n * AC1: Navigating to #/projects/:id/workspaces/:wid renders the workspace header\n *      and a repository status table with one row per repository.\n * AC2: Each repository row displays the current branch name, a color-coded Git\n *      status badge showing modified files, commits ahead/behind, last activity,\n *      and conflict indicator.\n * AC3: The status table auto-refreshes via setInterval calling\n *      GET /api/projects/:id/workspaces/:wid/status and updates badges in-place\n *      without re-rendering the entire view.\n * AC4: The polling interval is cleared when navigating away from the workspace\n *      detail view (no leaked intervals after navigation).\n * AC5: The \"Switch Branches\" button navigates to\n *      #/projects/:id/workspaces/:wid/branch-switch.\n * AC6: Rename and Delete actions are disabled for STABLE workspace and functional\n *      with confirmation dialogs for other workspaces.\n */\n\nimport { JSDOM } from 'jsdom';\nimport { strict as assert } from 'assert';\nimport { readFileSync } from 'fs';\nimport { fileURLToPath } from 'url';\nimport { dirname, resolve } from 'path';\n\nconst __filename = fileURLToPath(import.meta.url);\nconst __dirname = dirname(__filename);\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// Test harness\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nlet passed = 0;\nlet failed = 0;\nconst failures = [];\nconst promises = [];\n\nfunction test(name, fn) {\n    try {\n        const result = fn();\n        if (result && typeof result.then === 'function') {\n            const p = result.then(() => {\n                console.log(`  \u2705  ${name}`);\n                passed++;\n            }).catch((err) => {\n                console.error(`  \u274c  ${name}`);\n                console.error(`      ${err.message}`);\n                failed++;\n                failures.push({ name, error: err.message });\n            });\n            promises.push(p);\n            return p;\n        }\n        console.log(`  \u2705  ${name}`);\n        passed++;\n        return Promise.resolve();\n    } catch (err) {\n        console.error(`  \u274c  ${name}`);\n        console.error(`      ${err.message}`);\n        failed++;\n        failures.push({ name, error: err.message });\n        return Promise.resolve();\n    }\n}\n\nfunction section(title) {\n    console.log(`\\n\u2501\u2501 ${title} \u2501\u2501`);\n}\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// Helpers: read and strip ES module syntax for jsdom eval\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nfunction readAndStrip(filePath) {\n    let code = readFileSync(resolve(__dirname, filePath), 'utf8');\n    // Remove import statements\n    code = code.replace(/^import .+from .+;?\\s*$/gm, '');\n    code = code.replace(/^import .+$/gm, '');\n    // export class Foo \u2192 assign to window at end\n    code = code.replace(/^export class (\\w+)/gm, 'class $1');\n    const classNames = [];\n    const classMatches = [...code.matchAll(/^class (\\w+)/gm)];\n    for (const m of classMatches) classNames.push(m[1]);\n    for (const name of classNames) { code += `\\nwindow.${name} = ${name};`; }\n    // export async function / export function\n    code = code.replace(/^export (async function) (\\w+)/gm, 'window.$2 = $1 $2');\n    code = code.replace(/^export function (\\w+)/gm, 'window.$1 = function $1');\n    // export const/let/var \u2014 strip export, assign to window at end\n    const exportedVars = [];\n    code = code.replace(/^export (const|let|var) (\\w+)/gm, (match, kw, name) => {\n        exportedVars.push(name);\n        return `${kw} ${name}`;\n    });\n    for (const name of exportedVars) { code += `\\nwindow.${name} = ${name};`; }\n    return code;\n}\n\nfunction makeDOM() {\n    return new JSDOM(\n        `<!DOCTYPE html><html><body>\n            <div id=\"app\"></div>\n            <div id=\"toast-container\"></div>\n        </body></html>`,\n        { runScripts: 'dangerously', pretendToBeVisual: true }\n    );\n}\n\n/**\n * Build a jsdom window with all dependencies loaded (toast, confirm-dialog,\n * form-helpers, status-badge, api, workspace-detail).\n *\n * @param {function} fetchImpl - Mock fetch implementation.\n * @returns {Window}\n */\nfunction buildWindowWithDeps(fetchImpl) {\n    const dom = makeDOM();\n    const w = dom.window;\n    w.fetch = fetchImpl;\n\n    // Polyfill CSS.escape for jsdom\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) {\n        w.CSS.escape = (value) => String(value).replace(/[^\\w-]/g, c => '\\\\' + c);\n    }\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    return w;\n}\n\n/**\n * Resolve all microtasks / promises in jsdom.\n * @param {Window} w\n */\nasync function flushPromises(w) {\n    // Several rounds to resolve chained promises\n    for (let i = 0; i < 5; i++) {\n        await new Promise((r) => setTimeout(r, 0));\n    }\n}\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// Sample data\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nconst sampleWorkspace = {\n    Id: 'DEV',\n    Description: 'Development workspace',\n};\n\nconst sampleProject = {\n    Id: 'my-project',\n    Name: 'My Project',\n    Description: 'A test project',\n    Repositories: ['repo-alpha', 'repo-beta'],\n};\n\nconst sampleStatus = {\n    'repo-alpha': {\n        currentBranch: 'main',\n        localCommits: 2,\n        unfetchedCommits: 1,\n        modifiedFiles: 3,\n        lastActivity: new Date(Date.now() - 5 * 60_000).toISOString(), // 5 min ago\n        hasConflicts: false,\n    },\n    'repo-beta': {\n        currentBranch: 'feature/x',\n        localCommits: 0,\n        unfetchedCommits: 0,\n        modifiedFiles: 0,\n        lastActivity: new Date(Date.now() - 60 * 60_000).toISOString(), // 1 hr ago\n        hasConflicts: true,\n    },\n};\n\nconst sampleStatusRefreshed = {\n    'repo-alpha': {\n        currentBranch: 'develop',\n        localCommits: 0,\n        unfetchedCommits: 0,\n        modifiedFiles: 0,\n        lastActivity: new Date(Date.now() - 1 * 60_000).toISOString(),\n        hasConflicts: false,\n    },\n    'repo-beta': {\n        currentBranch: 'feature/y',\n        localCommits: 1,\n        unfetchedCommits: 0,\n        modifiedFiles: 1,\n        lastActivity: new Date(Date.now() - 10 * 60_000).toISOString(),\n        hasConflicts: false,\n    },\n};\n\nconst stableWorkspace = {\n    Id: 'STABLE',\n    Description: 'Stable workspace',\n};\n\n/**\n * Build a standard mock fetch for workspace-detail:\n *   GET /api/projects/:id/workspaces/:wid          \u2192 workspace\n *   GET /api/projects/:id                          \u2192 project\n *   GET /api/projects/:id/workspaces/:wid/status   \u2192 statusMap\n *\n * @param {Object} [overrides] - { workspace, project, status }\n * @param {function} [onCall]  - Optional callback invoked with (url, opts) for each fetch.\n */\nfunction buildStandardFetch(overrides = {}, onCall = null) {\n    const workspace  = overrides.workspace  || sampleWorkspace;\n    const project    = overrides.project    || sampleProject;\n    const statusMap  = overrides.status     || sampleStatus;\n\n    return async (url, opts) => {\n        if (onCall) onCall(url, opts);\n        const method = (opts || {}).method || 'GET';\n\n        if (method === 'GET') {\n            // /api/projects/:id/workspaces/:wid/status\n            if (/\\/workspaces\\/[^/]+\\/status$/.test(url)) {\n                return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => statusMap };\n            }\n            // /api/projects/:id/workspaces/:wid (single workspace \u2014 not a list)\n            if (/\\/workspaces\\/[^/]+$/.test(url)) {\n                return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => workspace };\n            }\n            // /api/projects/:id (single project)\n            if (/\\/api\\/projects\\/[^/]+$/.test(url)) {\n                return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => project };\n            }\n        }\n\n        // Default: 200 OK with empty object (for mutations)\n        return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => ({}) };\n    };\n}\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// Static analysis\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('Static analysis \u2014 workspace-detail.js');\n\nconst wdSrc = readFileSync(resolve(__dirname, 'gui/public/js/views/workspace-detail.js'), 'utf8');\nconst appSrc = readFileSync(resolve(__dirname, 'gui/public/js/app.js'), 'utf8');\n\ntest('workspace-detail.js imports api from api.js', () => {\n    assert.ok(wdSrc.includes(\"from '../api.js'\"), 'api.js import missing');\n});\n\ntest('workspace-detail.js imports createStatusBadge from status-badge.js', () => {\n    assert.ok(wdSrc.includes('createStatusBadge'), 'createStatusBadge import missing');\n});\n\ntest('workspace-detail.js imports showToast', () => {\n    assert.ok(wdSrc.includes('showToast'), 'showToast import missing');\n});\n\ntest('workspace-detail.js imports showConfirm', () => {\n    assert.ok(wdSrc.includes('showConfirm'), 'showConfirm import missing');\n});\n\ntest('workspace-detail.js exports renderWorkspaceDetail', () => {\n    assert.ok(wdSrc.includes('export function renderWorkspaceDetail'), 'renderWorkspaceDetail export missing');\n});\n\ntest('workspace-detail.js exports setRouter', () => {\n    assert.ok(wdSrc.includes('export function setRouter'), 'setRouter export missing');\n});\n\ntest('workspace-detail.js calls api.workspaces.get()', () => {\n    assert.ok(wdSrc.includes('api.workspaces.get('), 'api.workspaces.get() call missing');\n});\n\ntest('workspace-detail.js calls api.projects.get()', () => {\n    assert.ok(wdSrc.includes('api.projects.get('), 'api.projects.get() call missing');\n});\n\ntest('workspace-detail.js calls api.status.get()', () => {\n    assert.ok(wdSrc.includes('api.status.get('), 'api.status.get() call missing');\n});\n\ntest('workspace-detail.js uses setInterval for polling', () => {\n    assert.ok(wdSrc.includes('setInterval'), 'setInterval call missing \u2014 polling not implemented');\n});\n\ntest('workspace-detail.js uses clearInterval for cleanup', () => {\n    assert.ok(wdSrc.includes('clearInterval'), 'clearInterval call missing \u2014 polling interval not cleared');\n});\n\ntest('workspace-detail.js defines STABLE workspace constant', () => {\n    assert.ok(wdSrc.includes('STABLE'), \"STABLE workspace ID constant not found\");\n});\n\ntest('workspace-detail.js disables rename for STABLE', () => {\n    assert.ok(\n        wdSrc.includes('isStable') || wdSrc.includes('STABLE_WS_ID'),\n        'No STABLE workspace guard found'\n    );\n    assert.ok(wdSrc.includes('disabled'), 'No disabled property found for STABLE guards');\n});\n\ntest('workspace-detail.js calls api.workspaces.rename()', () => {\n    assert.ok(wdSrc.includes('api.workspaces.rename('), 'api.workspaces.rename() call missing');\n});\n\ntest('workspace-detail.js calls api.workspaces.delete()', () => {\n    assert.ok(wdSrc.includes('api.workspaces.delete('), 'api.workspaces.delete() call missing');\n});\n\ntest('workspace-detail.js validates workspace rename ID with /^[A-Z]{2,6}$/ regex', () => {\n    assert.ok(wdSrc.includes('[A-Z]{2,6}'), 'Rename validation regex /^[A-Z]{2,6}$/ missing');\n});\n\ntest('workspace-detail.js returns cleanup function from renderWorkspaceDetail', () => {\n    // The function must return a callable cleanup\n    assert.ok(wdSrc.includes('return cleanup'), 'return cleanup statement missing');\n});\n\ntest('app.js imports renderWorkspaceDetail', () => {\n    assert.ok(appSrc.includes('renderWorkspaceDetail'), 'renderWorkspaceDetail not imported in app.js');\n});\n\ntest('app.js imports setRouter as setWorkspaceDetailRouter', () => {\n    assert.ok(appSrc.includes('setWorkspaceDetailRouter'), 'setWorkspaceDetailRouter alias not imported in app.js');\n});\n\ntest(\"app.js registers renderWorkspaceDetail for '#/projects/:id/workspaces/:wid'\", () => {\n    const lines = appSrc.split('\\n');\n    const routeLine = lines.find(l =>\n        (l.includes(\"'#/projects/:id/workspaces/:wid'\") || l.includes('\"#/projects/:id/workspaces/:wid\"')) &&\n        l.includes('renderWorkspaceDetail')\n    );\n    assert.ok(routeLine, \"router.register('#/projects/:id/workspaces/:wid', renderWorkspaceDetail) not found\");\n});\n\ntest('app.js calls setWorkspaceDetailRouter(router)', () => {\n    assert.ok(appSrc.includes('setWorkspaceDetailRouter(router)'), 'setWorkspaceDetailRouter(router) not called in app.js');\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// AC1: Renders workspace header and repository status table\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('AC1 \u2014 Renders workspace header and repository status table');\n\ntest('AC1: renders workspace ID in a heading', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const heading = container.querySelector('h1');\n    assert.ok(heading, 'No <h1> heading found');\n    assert.ok(heading.textContent.includes('DEV'), `Heading does not contain workspace ID \"DEV\": \"${heading.textContent}\"`);\n});\n\ntest('AC1: renders breadcrumb link back to the project', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const link = container.querySelector('a[href=\"#/projects/my-project\"]');\n    assert.ok(link, 'Breadcrumb link to project (#/projects/my-project) not found');\n    assert.ok(link.textContent.includes('my-project'), 'Breadcrumb link does not display project ID');\n});\n\ntest('AC1: renders workspace description', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    assert.ok(\n        container.textContent.includes('Development workspace'),\n        'Workspace description \"Development workspace\" not rendered'\n    );\n});\n\ntest('AC1: renders a repository status table', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const table = container.querySelector('table');\n    assert.ok(table, 'No <table> element found for repository status');\n});\n\ntest('AC1: table has one row per repository (2 repos)', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const tbody = container.querySelector('tbody');\n    assert.ok(tbody, 'No <tbody> found');\n    const rows = tbody.querySelectorAll('tr');\n    assert.strictEqual(rows.length, 2, `Expected 2 repository rows, got ${rows.length}`);\n});\n\ntest('AC1: each repository row has data-repo-id attribute', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const rows = container.querySelectorAll('tbody tr');\n    rows.forEach((row) => {\n        assert.ok(row.dataset.repoId, `Row missing data-repo-id: ${row.innerHTML.slice(0, 100)}`);\n    });\n    const repoIds = [...rows].map((r) => r.dataset.repoId);\n    assert.ok(repoIds.includes('repo-alpha'), 'Row for repo-alpha not found');\n    assert.ok(repoIds.includes('repo-beta'), 'Row for repo-beta not found');\n});\n\ntest('AC1: shows \"No repositories\" message when project has empty repos list', async () => {\n    const emptyProject = { Id: 'empty-proj', Name: 'Empty', Repositories: [] };\n    const w = buildWindowWithDeps(buildStandardFetch({ project: emptyProject }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'empty-proj', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Should show an empty-state message, not a table\n    const table = container.querySelector('table');\n    const emptyMsg = container.querySelector('.empty-state-inline, .empty-state');\n    assert.ok(!table || emptyMsg, 'Expected empty-state indicator for project with no repositories');\n});\n\ntest('AC1: fetches workspace, project, and status in parallel (all three URLs called)', async () => {\n    const calledUrls = [];\n    const w = buildWindowWithDeps(buildStandardFetch({}, (url) => calledUrls.push(url)));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const hasWorkspaceFetch = calledUrls.some(u => /\\/workspaces\\/DEV$/.test(u));\n    const hasProjectFetch   = calledUrls.some(u => /\\/api\\/projects\\/my-project$/.test(u));\n    const hasStatusFetch    = calledUrls.some(u => /\\/workspaces\\/DEV\\/status$/.test(u));\n\n    assert.ok(hasWorkspaceFetch, `Workspace fetch not called. URLs: ${calledUrls.join(', ')}`);\n    assert.ok(hasProjectFetch,   `Project fetch not called. URLs: ${calledUrls.join(', ')}`);\n    assert.ok(hasStatusFetch,    `Status fetch not called. URLs: ${calledUrls.join(', ')}`);\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// AC2: Repository rows display branch name and status badge\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('AC2 \u2014 Repository rows: branch name + color-coded status badge');\n\ntest('AC2: repo-alpha row shows current branch \"main\"', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const alphaRow = container.querySelector('tr[data-repo-id=\"repo-alpha\"]');\n    assert.ok(alphaRow, 'Row for repo-alpha not found');\n    assert.ok(alphaRow.textContent.includes('main'), 'Branch \"main\" not shown for repo-alpha');\n});\n\ntest('AC2: repo-beta row shows current branch \"feature/x\"', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const betaRow = container.querySelector('tr[data-repo-id=\"repo-beta\"]');\n    assert.ok(betaRow, 'Row for repo-beta not found');\n    assert.ok(betaRow.textContent.includes('feature/x'), 'Branch \"feature/x\" not shown for repo-beta');\n});\n\ntest('AC2: status badge wrapper exists with data-repo-id for each row', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const alphaWrapper = container.querySelector('div[data-repo-id=\"repo-alpha\"]');\n    const betaWrapper  = container.querySelector('div[data-repo-id=\"repo-beta\"]');\n    assert.ok(alphaWrapper, 'Badge wrapper div[data-repo-id=\"repo-alpha\"] not found');\n    assert.ok(betaWrapper,  'Badge wrapper div[data-repo-id=\"repo-beta\"] not found');\n});\n\ntest('AC2: conflict badge rendered for repo-beta (hasConflicts=true)', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const betaWrapper = container.querySelector('div[data-repo-id=\"repo-beta\"]');\n    assert.ok(betaWrapper, 'Badge wrapper for repo-beta not found');\n    // status-badge conflict class or conflict text\n    assert.ok(\n        betaWrapper.innerHTML.includes('conflict') || betaWrapper.innerHTML.includes('Conflict'),\n        `Expected conflict indicator for repo-beta. Badge HTML: ${betaWrapper.innerHTML.slice(0, 300)}`\n    );\n});\n\ntest('AC2: modified-files badge shown for repo-alpha (modifiedFiles=3)', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const alphaWrapper = container.querySelector('div[data-repo-id=\"repo-alpha\"]');\n    assert.ok(alphaWrapper, 'Badge wrapper for repo-alpha not found');\n    assert.ok(\n        alphaWrapper.textContent.includes('3'),\n        `Expected modified files count \"3\" in badge. Got: ${alphaWrapper.textContent}`\n    );\n});\n\ntest('AC2: null status entry renders error/no-data badge', async () => {\n    const statusWithNull = {\n        'repo-alpha': null,\n        'repo-beta': sampleStatus['repo-beta'],\n    };\n    const w = buildWindowWithDeps(buildStandardFetch({ status: statusWithNull }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const alphaWrapper = container.querySelector('div[data-repo-id=\"repo-alpha\"]');\n    assert.ok(alphaWrapper, 'Badge wrapper for repo-alpha not found');\n    // Should show \"No data\" or an error badge\n    assert.ok(\n        alphaWrapper.textContent.toLowerCase().includes('no data') ||\n        alphaWrapper.innerHTML.includes('error') ||\n        alphaWrapper.innerHTML.includes('badge-error'),\n        `Expected no-data/error indicator for null status. Got: ${alphaWrapper.textContent}`\n    );\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// AC3: Status table auto-refreshes in-place via setInterval\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('AC3 \u2014 In-place polling via setInterval');\n\ntest('AC3: renderWorkspaceDetail calls setInterval after initial render', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n    w.fetch = buildStandardFetch();\n\n    let setIntervalCalled = false;\n    const origSetInterval = w.setInterval.bind(w);\n    w.setInterval = function (...args) {\n        setIntervalCalled = true;\n        return origSetInterval(...args);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    assert.ok(setIntervalCalled, 'setInterval was not called after initial render');\n});\n\ntest('AC3: polling calls GET .../status endpoint again after interval fires', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    let statusCallCount = 0;\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        if (method === 'GET' && /\\/workspaces\\/DEV\\/status$/.test(url)) {\n            statusCallCount++;\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleStatus };\n        }\n        if (method === 'GET' && /\\/workspaces\\/DEV$/.test(url)) {\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleWorkspace };\n        }\n        if (method === 'GET' && /\\/api\\/projects\\/my-project$/.test(url)) {\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleProject };\n        }\n        return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => ({}) };\n    };\n\n    // Capture and immediately fire the interval callback\n    let intervalCallback = null;\n    const origSetInterval = w.setInterval.bind(w);\n    w.setInterval = function (fn, delay) {\n        intervalCallback = fn;\n        return origSetInterval(fn, 999999); // don't auto-fire during test\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const countAfterInit = statusCallCount;\n    assert.ok(countAfterInit >= 1, 'Status endpoint not called on initial render');\n\n    // Manually fire the polling callback\n    assert.ok(intervalCallback, 'setInterval callback not captured');\n    await intervalCallback();\n    await flushPromises(w);\n\n    assert.ok(statusCallCount > countAfterInit, 'Status endpoint not called again on polling tick');\n});\n\ntest('AC3: polling updates branch name in-place (no full re-render)', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    let serveRefreshed = false;\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        if (method === 'GET' && /\\/workspaces\\/DEV\\/status$/.test(url)) {\n            return {\n                ok: true, status: 200,\n                headers: { get: () => 'application/json' },\n                json: async () => serveRefreshed ? sampleStatusRefreshed : sampleStatus,\n            };\n        }\n        if (method === 'GET' && /\\/workspaces\\/DEV$/.test(url)) {\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleWorkspace };\n        }\n        if (method === 'GET' && /\\/api\\/projects\\/my-project$/.test(url)) {\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleProject };\n        }\n        return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => ({}) };\n    };\n\n    let intervalCallback = null;\n    const origSetInterval = w.setInterval.bind(w);\n    w.setInterval = function (fn, delay) {\n        intervalCallback = fn;\n        return origSetInterval(fn, 999999);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Verify initial branch\n    const alphaRow = container.querySelector('tr[data-repo-id=\"repo-alpha\"]');\n    assert.ok(alphaRow, 'Row for repo-alpha not found after initial render');\n    assert.ok(alphaRow.textContent.includes('main'), 'Initial branch \"main\" not shown for repo-alpha');\n\n    // Record heading element to detect full re-render (heading should remain same object)\n    const headingBefore = container.querySelector('h1');\n\n    // Fire polling with updated status\n    serveRefreshed = true;\n    assert.ok(intervalCallback, 'setInterval callback not captured');\n    await intervalCallback();\n    await flushPromises(w);\n\n    // Branch should be updated in-place\n    const alphaRowAfter = container.querySelector('tr[data-repo-id=\"repo-alpha\"]');\n    assert.ok(alphaRowAfter, 'Row for repo-alpha not found after polling');\n    assert.ok(alphaRowAfter.textContent.includes('develop'), `Branch not updated in-place. Got: ${alphaRowAfter.textContent}`);\n\n    // Heading should be the same DOM element (no full re-render)\n    const headingAfter = container.querySelector('h1');\n    assert.strictEqual(headingBefore, headingAfter, 'Heading element was replaced \u2014 full re-render occurred instead of in-place update');\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// AC4: Polling interval cleared on navigation away\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('AC4 \u2014 Polling interval cleared on navigation away');\n\ntest('AC4: renderWorkspaceDetail returns a cleanup function', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    const cleanup = w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    assert.strictEqual(typeof cleanup, 'function', `renderWorkspaceDetail should return a function, got: ${typeof cleanup}`);\n});\n\ntest('AC4: cleanup function calls clearInterval', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n    w.fetch = buildStandardFetch();\n\n    let clearedIntervalId = null;\n    const origClearInterval = w.clearInterval.bind(w);\n    w.clearInterval = function (id) {\n        clearedIntervalId = id;\n        return origClearInterval(id);\n    };\n\n    let capturedIntervalId = null;\n    const origSetInterval = w.setInterval.bind(w);\n    w.setInterval = function (fn, delay) {\n        const id = origSetInterval(fn, 999999);\n        capturedIntervalId = id;\n        return id;\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    const container = w.document.getElementById('app');\n    const cleanup = w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    assert.ok(capturedIntervalId !== null, 'setInterval was not called \u2014 no polling started');\n\n    // Call the cleanup function (simulating navigation away)\n    cleanup();\n\n    assert.ok(clearedIntervalId !== null, 'clearInterval was not called by the cleanup function');\n});\n\ntest('AC4: cleanup function is safe to call multiple times without error', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    const cleanup = w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Should not throw\n    cleanup();\n    cleanup();\n    cleanup();\n});\n\ntest('AC4: polling stops if container is disconnected from DOM', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    let statusCallCount = 0;\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        if (method === 'GET' && /\\/workspaces\\/DEV\\/status$/.test(url)) {\n            statusCallCount++;\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleStatus };\n        }\n        if (method === 'GET' && /\\/workspaces\\/DEV$/.test(url)) {\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleWorkspace };\n        }\n        if (method === 'GET' && /\\/api\\/projects\\/my-project$/.test(url)) {\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleProject };\n        }\n        return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => ({}) };\n    };\n\n    let intervalCallback = null;\n    const origSetInterval = w.setInterval.bind(w);\n    w.setInterval = function (fn, delay) {\n        intervalCallback = fn;\n        return origSetInterval(fn, 999999);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Simulate navigation away \u2014 disconnect container\n    container.innerHTML = '';\n    w.document.body.removeChild(container);\n\n    const countBefore = statusCallCount;\n    // Fire the interval \u2014 it should check isConnected and not call fetch again\n    if (intervalCallback) {\n        await intervalCallback();\n        await flushPromises(w);\n    }\n    // Status call count should not have increased (or at most incremented by the\n    // async fetch that was in-flight when disconnected \u2014 but the result is ignored)\n    // We accept up to 1 extra call due to the async nature, but it should not keep going\n    assert.ok(\n        statusCallCount <= countBefore + 1,\n        `Polling continued after container disconnected: ${statusCallCount} calls (was ${countBefore})`\n    );\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// AC5: \"Switch Branches\" button navigates to branch-switch route\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('AC5 \u2014 \"Switch Branches\" button navigation');\n\ntest('AC5: \"Switch Branches\" button is rendered', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const switchBtn = buttons.find(b => b.textContent.trim().toLowerCase().includes('switch'));\n    assert.ok(switchBtn, 'No \"Switch Branches\" button found');\n});\n\ntest('AC5: clicking \"Switch Branches\" navigates via router.navigate()', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n\n    let navigatedTo = null;\n    const mockRouter = {\n        navigate(hash) { navigatedTo = hash; },\n    };\n    w.setRouter(mockRouter);\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const switchBtn = buttons.find(b => b.textContent.trim().toLowerCase().includes('switch'));\n    assert.ok(switchBtn, 'No \"Switch Branches\" button found');\n\n    switchBtn.click();\n\n    assert.ok(navigatedTo, 'router.navigate() not called after clicking \"Switch Branches\"');\n    assert.ok(\n        navigatedTo.includes('branch-switch'),\n        `Expected navigation to branch-switch route, got: ${navigatedTo}`\n    );\n    assert.ok(\n        navigatedTo.includes('my-project'),\n        `Expected navigation URL to include project ID, got: ${navigatedTo}`\n    );\n    assert.ok(\n        navigatedTo.includes('DEV'),\n        `Expected navigation URL to include workspace ID, got: ${navigatedTo}`\n    );\n});\n\ntest('AC5: branch-switch URL has correct format #/projects/:id/workspaces/:wid/branch-switch', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n\n    let navigatedTo = null;\n    const mockRouter = { navigate(hash) { navigatedTo = hash; } };\n    w.setRouter(mockRouter);\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'proj-x', wid: 'FEAT' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const switchBtn = buttons.find(b => b.textContent.trim().toLowerCase().includes('switch'));\n    switchBtn.click();\n\n    assert.strictEqual(\n        navigatedTo,\n        '#/projects/proj-x/workspaces/FEAT/branch-switch',\n        `Branch-switch URL mismatch. Got: ${navigatedTo}`\n    );\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// AC6: Rename and Delete disabled for STABLE; functional for others\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('AC6 \u2014 Rename and Delete disabled for STABLE, functional for others');\n\ntest('AC6: Rename button is disabled for STABLE workspace', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch({ workspace: stableWorkspace }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'STABLE' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const renameBtn = buttons.find(b => b.textContent.toLowerCase().includes('rename'));\n    assert.ok(renameBtn, 'Rename button not found for STABLE workspace');\n    assert.ok(renameBtn.disabled, 'Rename button is not disabled for STABLE workspace');\n});\n\ntest('AC6: Delete button is disabled for STABLE workspace', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch({ workspace: stableWorkspace }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'STABLE' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const deleteBtn = buttons.find(b => b.textContent.toLowerCase().includes('delete'));\n    assert.ok(deleteBtn, 'Delete button not found for STABLE workspace');\n    assert.ok(deleteBtn.disabled, 'Delete button is not disabled for STABLE workspace');\n});\n\ntest('AC6: Rename button has a title/tooltip explaining STABLE restriction', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch({ workspace: stableWorkspace }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'STABLE' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const renameBtn = buttons.find(b => b.textContent.toLowerCase().includes('rename'));\n    assert.ok(renameBtn && renameBtn.title, 'Rename button has no title attribute for STABLE workspace');\n});\n\ntest('AC6: Delete button has a title/tooltip explaining STABLE restriction', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch({ workspace: stableWorkspace }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'STABLE' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const deleteBtn = buttons.find(b => b.textContent.toLowerCase().includes('delete'));\n    assert.ok(deleteBtn && deleteBtn.title, 'Delete button has no title attribute for STABLE workspace');\n});\n\ntest('AC6: Rename button is enabled for non-STABLE workspace', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const renameBtn = buttons.find(b => b.textContent.toLowerCase().includes('rename'));\n    assert.ok(renameBtn, 'Rename button not found for DEV workspace');\n    assert.ok(!renameBtn.disabled, 'Rename button is unexpectedly disabled for DEV workspace');\n});\n\ntest('AC6: Delete button is enabled for non-STABLE workspace', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const deleteBtn = buttons.find(b => b.textContent.toLowerCase().includes('delete'));\n    assert.ok(deleteBtn, 'Delete button not found for DEV workspace');\n    assert.ok(!deleteBtn.disabled, 'Delete button is unexpectedly disabled for DEV workspace');\n});\n\ntest('AC6: clicking Rename button for non-STABLE shows inline form', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const renameBtn = buttons.find(b => b.textContent.trim() === 'Rename Workspace');\n    assert.ok(renameBtn, 'Rename Workspace button not found');\n    renameBtn.click();\n\n    // Form should now be visible\n    const formWrapper = container.querySelector('.rename-workspace-form-wrapper');\n    assert.ok(formWrapper, '.rename-workspace-form-wrapper not found');\n    assert.ok(!formWrapper.hidden, 'Rename form is still hidden after clicking Rename button');\n});\n\ntest('AC6: rename form validates against invalid ID (lowercase)', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Open rename form\n    const renameBtn = [...container.querySelectorAll('button')].find(b => b.textContent.trim() === 'Rename Workspace');\n    renameBtn.click();\n\n    const input = container.querySelector('[name=\"newWorkspaceId\"]');\n    assert.ok(input, 'newWorkspaceId input not found');\n    input.value = 'lowercase'; // invalid \u2014 should fail validation\n\n    const saveBtn = [...container.querySelectorAll('button')].find(b => b.textContent.includes('Save'));\n    assert.ok(saveBtn, 'Save button not found');\n    saveBtn.click();\n    await flushPromises(w);\n\n    // Error should be shown\n    const errorEl = container.querySelector('.field-error');\n    assert.ok(errorEl && !errorEl.hidden, 'Validation error not shown for invalid rename input \"lowercase\"');\n});\n\ntest('AC6: rename form validates against ID that is too short (1 char)', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const renameBtn = [...container.querySelectorAll('button')].find(b => b.textContent.trim() === 'Rename Workspace');\n    renameBtn.click();\n\n    const input = container.querySelector('[name=\"newWorkspaceId\"]');\n    input.value = 'A'; // too short\n    const saveBtn = [...container.querySelectorAll('button')].find(b => b.textContent.includes('Save'));\n    saveBtn.click();\n    await flushPromises(w);\n\n    const errorEl = container.querySelector('.field-error');\n    assert.ok(errorEl && !errorEl.hidden, 'Validation error not shown for too-short rename input \"A\"');\n});\n\ntest('AC6: valid rename (e.g. \"FEAT\") calls api.workspaces.rename() and shows confirm dialog', async () => {\n    const calledUrls = [];\n    let renamedTo = null;\n    let confirmShown = false;\n\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        calledUrls.push(url);\n        if (method === 'PUT' && url.includes('/rename')) {\n            const body = JSON.parse((opts || {}).body || '{}');\n            renamedTo = body.newId;\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => ({}) };\n        }\n        return buildStandardFetch()(url, opts);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    // Mock showConfirm to auto-accept and track call\n    w.showConfirm = async () => { confirmShown = true; return true; };\n\n    let navigatedTo = null;\n    w.setRouter({ navigate(h) { navigatedTo = h; } });\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Open rename form\n    const renameBtn = [...container.querySelectorAll('button')].find(b => b.textContent.trim() === 'Rename Workspace');\n    renameBtn.click();\n\n    const input = container.querySelector('[name=\"newWorkspaceId\"]');\n    input.value = 'FEAT';\n\n    const saveBtn = [...container.querySelectorAll('button')].find(b => b.textContent.includes('Save'));\n    saveBtn.click();\n    await flushPromises(w);\n\n    assert.ok(confirmShown, 'showConfirm not called before rename');\n    assert.ok(renamedTo === 'FEAT', `api.workspaces.rename not called with \"FEAT\". Got: ${renamedTo}`);\n    assert.ok(\n        navigatedTo && navigatedTo.includes('FEAT'),\n        `Expected navigation to new workspace URL containing \"FEAT\". Got: ${navigatedTo}`\n    );\n});\n\ntest('AC6: Delete workspace shows confirm dialog and calls api.workspaces.delete()', async () => {\n    let deleteUrl = null;\n    let confirmShown = false;\n\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        if (method === 'DELETE' && url.includes('/workspaces/DEV')) {\n            deleteUrl = url;\n            return { ok: true, status: 204, headers: { get: () => '' }, json: async () => undefined };\n        }\n        return buildStandardFetch()(url, opts);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    w.showConfirm = async () => { confirmShown = true; return true; };\n\n    let navigatedTo = null;\n    w.setRouter({ navigate(h) { navigatedTo = h; } });\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const deleteBtn = [...container.querySelectorAll('button')].find(b => b.textContent.toLowerCase().includes('delete'));\n    assert.ok(deleteBtn, 'Delete button not found');\n    deleteBtn.click();\n    await flushPromises(w);\n\n    assert.ok(confirmShown, 'showConfirm not called before delete');\n    assert.ok(deleteUrl, `api.workspaces.delete() not called. DELETE URL not captured.`);\n    assert.ok(deleteUrl.includes('DEV'), `DELETE URL does not include workspace ID \"DEV\": ${deleteUrl}`);\n    assert.ok(\n        navigatedTo && navigatedTo.includes('my-project') && !navigatedTo.includes('DEV'),\n        `Expected navigation back to project (not workspace). Got: ${navigatedTo}`\n    );\n});\n\ntest('AC6: Delete workspace navigates to #/projects/:id after success', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        if (method === 'DELETE') {\n            return { ok: true, status: 204, headers: { get: () => '' }, json: async () => undefined };\n        }\n        return buildStandardFetch()(url, opts);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    w.showConfirm = async () => true;\n\n    let navigatedTo = null;\n    w.setRouter({ navigate(h) { navigatedTo = h; } });\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const deleteBtn = [...container.querySelectorAll('button')].find(b => b.textContent.toLowerCase().includes('delete'));\n    deleteBtn.click();\n    await flushPromises(w);\n\n    assert.strictEqual(\n        navigatedTo,\n        '#/projects/my-project',\n        `Expected navigation to \"#/projects/my-project\" after delete. Got: ${navigatedTo}`\n    );\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// Edge cases\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('Edge Cases');\n\ntest('Edge: renders error state with back link when API fetch fails', async () => {\n    const w = buildWindowWithDeps(async () => {\n        throw new Error('Network error');\n    });\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const backLink = container.querySelector('a');\n    assert.ok(backLink, 'No back link rendered after API failure');\n    assert.ok(\n        backLink.href.includes('my-project') || backLink.textContent.toLowerCase().includes('back'),\n        `Back link does not reference the project. href=\"${backLink.href}\", text=\"${backLink.textContent}\"`\n    );\n    assert.ok(\n        container.textContent.toLowerCase().includes('fail') ||\n        container.textContent.toLowerCase().includes('error'),\n        'No error/fail message rendered after API failure'\n    );\n});\n\ntest('Edge: repository entries as objects with Id/Name are normalised correctly', async () => {\n    const projectWithObjectRepos = {\n        Id: 'obj-proj',\n        Name: 'Object Repos Project',\n        Repositories: [\n            { Id: 'repo-one', Name: 'Repo One' },\n            { Id: 'repo-two', Name: 'Repo Two' },\n        ],\n    };\n    const statusForObjectRepos = {\n        'repo-one': { currentBranch: 'main', localCommits: 0, unfetchedCommits: 0, modifiedFiles: 0, lastActivity: null, hasConflicts: false },\n        'repo-two': { currentBranch: 'dev',  localCommits: 0, unfetchedCommits: 0, modifiedFiles: 0, lastActivity: null, hasConflicts: false },\n    };\n    const w = buildWindowWithDeps(buildStandardFetch({ project: projectWithObjectRepos, status: statusForObjectRepos }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'obj-proj', wid: 'DEV' });\n    await flushPromises(w);\n\n    const rows = container.querySelectorAll('tbody tr');\n    assert.strictEqual(rows.length, 2, `Expected 2 rows for object-based repos, got ${rows.length}`);\n    const repoIds = [...rows].map(r => r.dataset.repoId);\n    assert.ok(repoIds.includes('repo-one'), 'Row for repo-one not found');\n    assert.ok(repoIds.includes('repo-two'), 'Row for repo-two not found');\n});\n\ntest('Edge: rename cancel hides form and re-shows Rename button', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const renameBtn = [...container.querySelectorAll('button')].find(b => b.textContent.trim() === 'Rename Workspace');\n    renameBtn.click();\n\n    const formWrapper = container.querySelector('.rename-workspace-form-wrapper');\n    assert.ok(!formWrapper.hidden, 'Form should be visible after clicking rename');\n\n    const cancelBtn = [...container.querySelectorAll('button')].find(b => b.textContent.trim() === 'Cancel');\n    assert.ok(cancelBtn, 'Cancel button not found');\n    cancelBtn.click();\n\n    assert.ok(formWrapper.hidden, 'Form should be hidden after clicking Cancel');\n    assert.ok(!renameBtn.hidden, 'Rename button should be visible again after Cancel');\n});\n\ntest('Edge: polling silently handles API error without crashing the view', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    let isPollingCall = false;\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        if (method === 'GET' && /\\/workspaces\\/DEV\\/status$/.test(url) && isPollingCall) {\n            throw new Error('Polling network failure');\n        }\n        return buildStandardFetch()(url, opts);\n    };\n\n    let intervalCallback = null;\n    const origSetInterval = w.setInterval.bind(w);\n    w.setInterval = function (fn, delay) {\n        intervalCallback = fn;\n        return origSetInterval(fn, 999999);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Table should still exist before polling error\n    const headingBefore = container.querySelector('h1');\n    assert.ok(headingBefore, 'Heading missing before polling error');\n\n    // Fire polling with simulated error\n    isPollingCall = true;\n    if (intervalCallback) {\n        await intervalCallback().catch(() => {}); // should not throw\n    }\n    await flushPromises(w);\n\n    // View should be intact \u2014 heading still present\n    const headingAfter = container.querySelector('h1');\n    assert.ok(headingAfter, 'View was destroyed after polling error \u2014 should have been resilient');\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// Summary\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nawait Promise.allSettled(promises);\n\nconsole.log('\\n' + '\u2550'.repeat(55));\nconsole.log(`  Total: ${passed + failed}  |  \u2705 Passed: ${passed}  |  \u274c Failed: ${failed}`);\nconsole.log('\u2550'.repeat(55));\n\nif (failures.length > 0) {\n    console.log('\\nFailed tests:');\n    failures.forEach((f) => console.log(`  \u274c ${f.name}\\n     ${f.error}`));\n    process.exit(1);\n}\n\nprocess.exit(0);\n"
  },
  "name": "write_file",
  "type": "tool_use"
}
```

**Tool call:** `write_file` (id: `toolu_01CtVVtYKwRbaVeUiuLgWuYw`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp016-tests.mjs",
  "content": "/**\n * QA test suite for WP-016 (WP-006): Workspace Detail View with Git Status Polling.\n *\n * Runs entirely in Node.js using jsdom for DOM-dependent code.\n * Tests map to WP-016 Acceptance Criteria.\n *\n * AC1: Navigating to #/projects/:id/workspaces/:wid renders the workspace header\n *      and a repository status table with one row per repository.\n * AC2: Each repository row displays the current branch name, a color-coded Git\n *      status badge showing modified files, commits ahead/behind, last activity,\n *      and conflict indicator.\n * AC3: The status table auto-refreshes via setInterval calling\n *      GET /api/projects/:id/workspaces/:wid/status and updates badges in-place\n *      without re-rendering the entire view.\n * AC4: The polling interval is cleared when navigating away from the workspace\n *      detail view (no leaked intervals after navigation).\n * AC5: The \"Switch Branches\" button navigates to\n *      #/projects/:id/workspaces/:wid/branch-switch.\n * AC6: Rename and Delete actions are disabled for STABLE workspace and functional\n *      with confirmation dialogs for other workspaces.\n */\n\nimport { JSDOM } from 'jsdom';\nimport { strict as assert } from 'assert';\nimport { readFileSync } from 'fs';\nimport { fileURLToPath } from 'url';\nimport { dirname, resolve } from 'path';\n\nconst __filename = fileURLToPath(import.meta.url);\nconst __dirname = dirname(__filename);\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// Test harness\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nlet passed = 0;\nlet failed = 0;\nconst failures = [];\nconst promises = [];\n\nfunction test(name, fn) {\n    try {\n        const result = fn();\n        if (result && typeof result.then === 'function') {\n            const p = result.then(() => {\n                console.log(`  \u2705  ${name}`);\n                passed++;\n            }).catch((err) => {\n                console.error(`  \u274c  ${name}`);\n                console.error(`      ${err.message}`);\n                failed++;\n                failures.push({ name, error: err.message });\n            });\n            promises.push(p);\n            return p;\n        }\n        console.log(`  \u2705  ${name}`);\n        passed++;\n        return Promise.resolve();\n    } catch (err) {\n        console.error(`  \u274c  ${name}`);\n        console.error(`      ${err.message}`);\n        failed++;\n        failures.push({ name, error: err.message });\n        return Promise.resolve();\n    }\n}\n\nfunction section(title) {\n    console.log(`\\n\u2501\u2501 ${title} \u2501\u2501`);\n}\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// Helpers: read and strip ES module syntax for jsdom eval\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nfunction readAndStrip(filePath) {\n    let code = readFileSync(resolve(__dirname, filePath), 'utf8');\n    // Remove import statements\n    code = code.replace(/^import .+from .+;?\\s*$/gm, '');\n    code = code.replace(/^import .+$/gm, '');\n    // export class Foo \u2192 assign to window at end\n    code = code.replace(/^export class (\\w+)/gm, 'class $1');\n    const classNames = [];\n    const classMatches = [...code.matchAll(/^class (\\w+)/gm)];\n    for (const m of classMatches) classNames.push(m[1]);\n    for (const name of classNames) { code += `\\nwindow.${name} = ${name};`; }\n    // export async function / export function\n    code = code.replace(/^export (async function) (\\w+)/gm, 'window.$2 = $1 $2');\n    code = code.replace(/^export function (\\w+)/gm, 'window.$1 = function $1');\n    // export const/let/var \u2014 strip export, assign to window at end\n    const exportedVars = [];\n    code = code.replace(/^export (const|let|var) (\\w+)/gm, (match, kw, name) => {\n        exportedVars.push(name);\n        return `${kw} ${name}`;\n    });\n    for (const name of exportedVars) { code += `\\nwindow.${name} = ${name};`; }\n    return code;\n}\n\nfunction makeDOM() {\n    return new JSDOM(\n        `<!DOCTYPE html><html><body>\n            <div id=\"app\"></div>\n            <div id=\"toast-container\"></div>\n        </body></html>`,\n        { runScripts: 'dangerously', pretendToBeVisual: true }\n    );\n}\n\n/**\n * Build a jsdom window with all dependencies loaded (toast, confirm-dialog,\n * form-helpers, status-badge, api, workspace-detail).\n *\n * @param {function} fetchImpl - Mock fetch implementation.\n * @returns {Window}\n */\nfunction buildWindowWithDeps(fetchImpl) {\n    const dom = makeDOM();\n    const w = dom.window;\n    w.fetch = fetchImpl;\n\n    // Polyfill CSS.escape for jsdom\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) {\n        w.CSS.escape = (value) => String(value).replace(/[^\\w-]/g, c => '\\\\' + c);\n    }\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    return w;\n}\n\n/**\n * Resolve all microtasks / promises in jsdom.\n * @param {Window} w\n */\nasync function flushPromises(w) {\n    // Several rounds to resolve chained promises\n    for (let i = 0; i < 5; i++) {\n        await new Promise((r) => setTimeout(r, 0));\n    }\n}\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// Sample data\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nconst sampleWorkspace = {\n    Id: 'DEV',\n    Description: 'Development workspace',\n};\n\nconst sampleProject = {\n    Id: 'my-project',\n    Name: 'My Project',\n    Description: 'A test project',\n    Repositories: ['repo-alpha', 'repo-beta'],\n};\n\nconst sampleStatus = {\n    'repo-alpha': {\n        currentBranch: 'main',\n        localCommits: 2,\n        unfetchedCommits: 1,\n        modifiedFiles: 3,\n        lastActivity: new Date(Date.now() - 5 * 60_000).toISOString(), // 5 min ago\n        hasConflicts: false,\n    },\n    'repo-beta': {\n        currentBranch: 'feature/x',\n        localCommits: 0,\n        unfetchedCommits: 0,\n        modifiedFiles: 0,\n        lastActivity: new Date(Date.now() - 60 * 60_000).toISOString(), // 1 hr ago\n        hasConflicts: true,\n    },\n};\n\nconst sampleStatusRefreshed = {\n    'repo-alpha': {\n        currentBranch: 'develop',\n        localCommits: 0,\n        unfetchedCommits: 0,\n        modifiedFiles: 0,\n        lastActivity: new Date(Date.now() - 1 * 60_000).toISOString(),\n        hasConflicts: false,\n    },\n    'repo-beta': {\n        currentBranch: 'feature/y',\n        localCommits: 1,\n        unfetchedCommits: 0,\n        modifiedFiles: 1,\n        lastActivity: new Date(Date.now() - 10 * 60_000).toISOString(),\n        hasConflicts: false,\n    },\n};\n\nconst stableWorkspace = {\n    Id: 'STABLE',\n    Description: 'Stable workspace',\n};\n\n/**\n * Build a standard mock fetch for workspace-detail:\n *   GET /api/projects/:id/workspaces/:wid          \u2192 workspace\n *   GET /api/projects/:id                          \u2192 project\n *   GET /api/projects/:id/workspaces/:wid/status   \u2192 statusMap\n *\n * @param {Object} [overrides] - { workspace, project, status }\n * @param {function} [onCall]  - Optional callback invoked with (url, opts) for each fetch.\n */\nfunction buildStandardFetch(overrides = {}, onCall = null) {\n    const workspace  = overrides.workspace  || sampleWorkspace;\n    const project    = overrides.project    || sampleProject;\n    const statusMap  = overrides.status     || sampleStatus;\n\n    return async (url, opts) => {\n        if (onCall) onCall(url, opts);\n        const method = (opts || {}).method || 'GET';\n\n        if (method === 'GET') {\n            // /api/projects/:id/workspaces/:wid/status\n            if (/\\/workspaces\\/[^/]+\\/status$/.test(url)) {\n                return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => statusMap };\n            }\n            // /api/projects/:id/workspaces/:wid (single workspace \u2014 not a list)\n            if (/\\/workspaces\\/[^/]+$/.test(url)) {\n                return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => workspace };\n            }\n            // /api/projects/:id (single project)\n            if (/\\/api\\/projects\\/[^/]+$/.test(url)) {\n                return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => project };\n            }\n        }\n\n        // Default: 200 OK with empty object (for mutations)\n        return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => ({}) };\n    };\n}\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// Static analysis\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('Static analysis \u2014 workspace-detail.js');\n\nconst wdSrc = readFileSync(resolve(__dirname, 'gui/public/js/views/workspace-detail.js'), 'utf8');\nconst appSrc = readFileSync(resolve(__dirname, 'gui/public/js/app.js'), 'utf8');\n\ntest('workspace-detail.js imports api from api.js', () => {\n    assert.ok(wdSrc.includes(\"from '../api.js'\"), 'api.js import missing');\n});\n\ntest('workspace-detail.js imports createStatusBadge from status-badge.js', () => {\n    assert.ok(wdSrc.includes('createStatusBadge'), 'createStatusBadge import missing');\n});\n\ntest('workspace-detail.js imports showToast', () => {\n    assert.ok(wdSrc.includes('showToast'), 'showToast import missing');\n});\n\ntest('workspace-detail.js imports showConfirm', () => {\n    assert.ok(wdSrc.includes('showConfirm'), 'showConfirm import missing');\n});\n\ntest('workspace-detail.js exports renderWorkspaceDetail', () => {\n    assert.ok(wdSrc.includes('export function renderWorkspaceDetail'), 'renderWorkspaceDetail export missing');\n});\n\ntest('workspace-detail.js exports setRouter', () => {\n    assert.ok(wdSrc.includes('export function setRouter'), 'setRouter export missing');\n});\n\ntest('workspace-detail.js calls api.workspaces.get()', () => {\n    assert.ok(wdSrc.includes('api.workspaces.get('), 'api.workspaces.get() call missing');\n});\n\ntest('workspace-detail.js calls api.projects.get()', () => {\n    assert.ok(wdSrc.includes('api.projects.get('), 'api.projects.get() call missing');\n});\n\ntest('workspace-detail.js calls api.status.get()', () => {\n    assert.ok(wdSrc.includes('api.status.get('), 'api.status.get() call missing');\n});\n\ntest('workspace-detail.js uses setInterval for polling', () => {\n    assert.ok(wdSrc.includes('setInterval'), 'setInterval call missing \u2014 polling not implemented');\n});\n\ntest('workspace-detail.js uses clearInterval for cleanup', () => {\n    assert.ok(wdSrc.includes('clearInterval'), 'clearInterval call missing \u2014 polling interval not cleared');\n});\n\ntest('workspace-detail.js defines STABLE workspace constant', () => {\n    assert.ok(wdSrc.includes('STABLE'), \"STABLE workspace ID constant not found\");\n});\n\ntest('workspace-detail.js disables rename for STABLE', () => {\n    assert.ok(\n        wdSrc.includes('isStable') || wdSrc.includes('STABLE_WS_ID'),\n        'No STABLE workspace guard found'\n    );\n    assert.ok(wdSrc.includes('disabled'), 'No disabled property found for STABLE guards');\n});\n\ntest('workspace-detail.js calls api.workspaces.rename()', () => {\n    assert.ok(wdSrc.includes('api.workspaces.rename('), 'api.workspaces.rename() call missing');\n});\n\ntest('workspace-detail.js calls api.workspaces.delete()', () => {\n    assert.ok(wdSrc.includes('api.workspaces.delete('), 'api.workspaces.delete() call missing');\n});\n\ntest('workspace-detail.js validates workspace rename ID with /^[A-Z]{2,6}$/ regex', () => {\n    assert.ok(wdSrc.includes('[A-Z]{2,6}'), 'Rename validation regex /^[A-Z]{2,6}$/ missing');\n});\n\ntest('workspace-detail.js returns cleanup function from renderWorkspaceDetail', () => {\n    // The function must return a callable cleanup\n    assert.ok(wdSrc.includes('return cleanup'), 'return cleanup statement missing');\n});\n\ntest('app.js imports renderWorkspaceDetail', () => {\n    assert.ok(appSrc.includes('renderWorkspaceDetail'), 'renderWorkspaceDetail not imported in app.js');\n});\n\ntest('app.js imports setRouter as setWorkspaceDetailRouter', () => {\n    assert.ok(appSrc.includes('setWorkspaceDetailRouter'), 'setWorkspaceDetailRouter alias not imported in app.js');\n});\n\ntest(\"app.js registers renderWorkspaceDetail for '#/projects/:id/workspaces/:wid'\", () => {\n    const lines = appSrc.split('\\n');\n    const routeLine = lines.find(l =>\n        (l.includes(\"'#/projects/:id/workspaces/:wid'\") || l.includes('\"#/projects/:id/workspaces/:wid\"')) &&\n        l.includes('renderWorkspaceDetail')\n    );\n    assert.ok(routeLine, \"router.register('#/projects/:id/workspaces/:wid', renderWorkspaceDetail) not found\");\n});\n\ntest('app.js calls setWorkspaceDetailRouter(router)', () => {\n    assert.ok(appSrc.includes('setWorkspaceDetailRouter(router)'), 'setWorkspaceDetailRouter(router) not called in app.js');\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// AC1: Renders workspace header and repository status table\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('AC1 \u2014 Renders workspace header and repository status table');\n\ntest('AC1: renders workspace ID in a heading', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const heading = container.querySelector('h1');\n    assert.ok(heading, 'No <h1> heading found');\n    assert.ok(heading.textContent.includes('DEV'), `Heading does not contain workspace ID \"DEV\": \"${heading.textContent}\"`);\n});\n\ntest('AC1: renders breadcrumb link back to the project', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const link = container.querySelector('a[href=\"#/projects/my-project\"]');\n    assert.ok(link, 'Breadcrumb link to project (#/projects/my-project) not found');\n    assert.ok(link.textContent.includes('my-project'), 'Breadcrumb link does not display project ID');\n});\n\ntest('AC1: renders workspace description', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    assert.ok(\n        container.textContent.includes('Development workspace'),\n        'Workspace description \"Development workspace\" not rendered'\n    );\n});\n\ntest('AC1: renders a repository status table', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const table = container.querySelector('table');\n    assert.ok(table, 'No <table> element found for repository status');\n});\n\ntest('AC1: table has one row per repository (2 repos)', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const tbody = container.querySelector('tbody');\n    assert.ok(tbody, 'No <tbody> found');\n    const rows = tbody.querySelectorAll('tr');\n    assert.strictEqual(rows.length, 2, `Expected 2 repository rows, got ${rows.length}`);\n});\n\ntest('AC1: each repository row has data-repo-id attribute', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const rows = container.querySelectorAll('tbody tr');\n    rows.forEach((row) => {\n        assert.ok(row.dataset.repoId, `Row missing data-repo-id: ${row.innerHTML.slice(0, 100)}`);\n    });\n    const repoIds = [...rows].map((r) => r.dataset.repoId);\n    assert.ok(repoIds.includes('repo-alpha'), 'Row for repo-alpha not found');\n    assert.ok(repoIds.includes('repo-beta'), 'Row for repo-beta not found');\n});\n\ntest('AC1: shows \"No repositories\" message when project has empty repos list', async () => {\n    const emptyProject = { Id: 'empty-proj', Name: 'Empty', Repositories: [] };\n    const w = buildWindowWithDeps(buildStandardFetch({ project: emptyProject }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'empty-proj', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Should show an empty-state message, not a table\n    const table = container.querySelector('table');\n    const emptyMsg = container.querySelector('.empty-state-inline, .empty-state');\n    assert.ok(!table || emptyMsg, 'Expected empty-state indicator for project with no repositories');\n});\n\ntest('AC1: fetches workspace, project, and status in parallel (all three URLs called)', async () => {\n    const calledUrls = [];\n    const w = buildWindowWithDeps(buildStandardFetch({}, (url) => calledUrls.push(url)));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const hasWorkspaceFetch = calledUrls.some(u => /\\/workspaces\\/DEV$/.test(u));\n    const hasProjectFetch   = calledUrls.some(u => /\\/api\\/projects\\/my-project$/.test(u));\n    const hasStatusFetch    = calledUrls.some(u => /\\/workspaces\\/DEV\\/status$/.test(u));\n\n    assert.ok(hasWorkspaceFetch, `Workspace fetch not called. URLs: ${calledUrls.join(', ')}`);\n    assert.ok(hasProjectFetch,   `Project fetch not called. URLs: ${calledUrls.join(', ')}`);\n    assert.ok(hasStatusFetch,    `Status fetch not called. URLs: ${calledUrls.join(', ')}`);\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// AC2: Repository rows display branch name and status badge\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('AC2 \u2014 Repository rows: branch name + color-coded status badge');\n\ntest('AC2: repo-alpha row shows current branch \"main\"', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const alphaRow = container.querySelector('tr[data-repo-id=\"repo-alpha\"]');\n    assert.ok(alphaRow, 'Row for repo-alpha not found');\n    assert.ok(alphaRow.textContent.includes('main'), 'Branch \"main\" not shown for repo-alpha');\n});\n\ntest('AC2: repo-beta row shows current branch \"feature/x\"', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const betaRow = container.querySelector('tr[data-repo-id=\"repo-beta\"]');\n    assert.ok(betaRow, 'Row for repo-beta not found');\n    assert.ok(betaRow.textContent.includes('feature/x'), 'Branch \"feature/x\" not shown for repo-beta');\n});\n\ntest('AC2: status badge wrapper exists with data-repo-id for each row', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const alphaWrapper = container.querySelector('div[data-repo-id=\"repo-alpha\"]');\n    const betaWrapper  = container.querySelector('div[data-repo-id=\"repo-beta\"]');\n    assert.ok(alphaWrapper, 'Badge wrapper div[data-repo-id=\"repo-alpha\"] not found');\n    assert.ok(betaWrapper,  'Badge wrapper div[data-repo-id=\"repo-beta\"] not found');\n});\n\ntest('AC2: conflict badge rendered for repo-beta (hasConflicts=true)', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const betaWrapper = container.querySelector('div[data-repo-id=\"repo-beta\"]');\n    assert.ok(betaWrapper, 'Badge wrapper for repo-beta not found');\n    // status-badge conflict class or conflict text\n    assert.ok(\n        betaWrapper.innerHTML.includes('conflict') || betaWrapper.innerHTML.includes('Conflict'),\n        `Expected conflict indicator for repo-beta. Badge HTML: ${betaWrapper.innerHTML.slice(0, 300)}`\n    );\n});\n\ntest('AC2: modified-files badge shown for repo-alpha (modifiedFiles=3)', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const alphaWrapper = container.querySelector('div[data-repo-id=\"repo-alpha\"]');\n    assert.ok(alphaWrapper, 'Badge wrapper for repo-alpha not found');\n    assert.ok(\n        alphaWrapper.textContent.includes('3'),\n        `Expected modified files count \"3\" in badge. Got: ${alphaWrapper.textContent}`\n    );\n});\n\ntest('AC2: null status entry renders error/no-data badge', async () => {\n    const statusWithNull = {\n        'repo-alpha': null,\n        'repo-beta': sampleStatus['repo-beta'],\n    };\n    const w = buildWindowWithDeps(buildStandardFetch({ status: statusWithNull }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const alphaWrapper = container.querySelector('div[data-repo-id=\"repo-alpha\"]');\n    assert.ok(alphaWrapper, 'Badge wrapper for repo-alpha not found');\n    // Should show \"No data\" or an error badge\n    assert.ok(\n        alphaWrapper.textContent.toLowerCase().includes('no data') ||\n        alphaWrapper.innerHTML.includes('error') ||\n        alphaWrapper.innerHTML.includes('badge-error'),\n        `Expected no-data/error indicator for null status. Got: ${alphaWrapper.textContent}`\n    );\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// AC3: Status table auto-refreshes in-place via setInterval\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('AC3 \u2014 In-place polling via setInterval');\n\ntest('AC3: renderWorkspaceDetail calls setInterval after initial render', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n    w.fetch = buildStandardFetch();\n\n    let setIntervalCalled = false;\n    const origSetInterval = w.setInterval.bind(w);\n    w.setInterval = function (...args) {\n        setIntervalCalled = true;\n        return origSetInterval(...args);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    assert.ok(setIntervalCalled, 'setInterval was not called after initial render');\n});\n\ntest('AC3: polling calls GET .../status endpoint again after interval fires', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    let statusCallCount = 0;\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        if (method === 'GET' && /\\/workspaces\\/DEV\\/status$/.test(url)) {\n            statusCallCount++;\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleStatus };\n        }\n        if (method === 'GET' && /\\/workspaces\\/DEV$/.test(url)) {\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleWorkspace };\n        }\n        if (method === 'GET' && /\\/api\\/projects\\/my-project$/.test(url)) {\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleProject };\n        }\n        return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => ({}) };\n    };\n\n    // Capture and immediately fire the interval callback\n    let intervalCallback = null;\n    const origSetInterval = w.setInterval.bind(w);\n    w.setInterval = function (fn, delay) {\n        intervalCallback = fn;\n        return origSetInterval(fn, 999999); // don't auto-fire during test\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const countAfterInit = statusCallCount;\n    assert.ok(countAfterInit >= 1, 'Status endpoint not called on initial render');\n\n    // Manually fire the polling callback\n    assert.ok(intervalCallback, 'setInterval callback not captured');\n    await intervalCallback();\n    await flushPromises(w);\n\n    assert.ok(statusCallCount > countAfterInit, 'Status endpoint not called again on polling tick');\n});\n\ntest('AC3: polling updates branch name in-place (no full re-render)', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    let serveRefreshed = false;\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        if (method === 'GET' && /\\/workspaces\\/DEV\\/status$/.test(url)) {\n            return {\n                ok: true, status: 200,\n                headers: { get: () => 'application/json' },\n                json: async () => serveRefreshed ? sampleStatusRefreshed : sampleStatus,\n            };\n        }\n        if (method === 'GET' && /\\/workspaces\\/DEV$/.test(url)) {\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleWorkspace };\n        }\n        if (method === 'GET' && /\\/api\\/projects\\/my-project$/.test(url)) {\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleProject };\n        }\n        return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => ({}) };\n    };\n\n    let intervalCallback = null;\n    const origSetInterval = w.setInterval.bind(w);\n    w.setInterval = function (fn, delay) {\n        intervalCallback = fn;\n        return origSetInterval(fn, 999999);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Verify initial branch\n    const alphaRow = container.querySelector('tr[data-repo-id=\"repo-alpha\"]');\n    assert.ok(alphaRow, 'Row for repo-alpha not found after initial render');\n    assert.ok(alphaRow.textContent.includes('main'), 'Initial branch \"main\" not shown for repo-alpha');\n\n    // Record heading element to detect full re-render (heading should remain same object)\n    const headingBefore = container.querySelector('h1');\n\n    // Fire polling with updated status\n    serveRefreshed = true;\n    assert.ok(intervalCallback, 'setInterval callback not captured');\n    await intervalCallback();\n    await flushPromises(w);\n\n    // Branch should be updated in-place\n    const alphaRowAfter = container.querySelector('tr[data-repo-id=\"repo-alpha\"]');\n    assert.ok(alphaRowAfter, 'Row for repo-alpha not found after polling');\n    assert.ok(alphaRowAfter.textContent.includes('develop'), `Branch not updated in-place. Got: ${alphaRowAfter.textContent}`);\n\n    // Heading should be the same DOM element (no full re-render)\n    const headingAfter = container.querySelector('h1');\n    assert.strictEqual(headingBefore, headingAfter, 'Heading element was replaced \u2014 full re-render occurred instead of in-place update');\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// AC4: Polling interval cleared on navigation away\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('AC4 \u2014 Polling interval cleared on navigation away');\n\ntest('AC4: renderWorkspaceDetail returns a cleanup function', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    const cleanup = w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    assert.strictEqual(typeof cleanup, 'function', `renderWorkspaceDetail should return a function, got: ${typeof cleanup}`);\n});\n\ntest('AC4: cleanup function calls clearInterval', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n    w.fetch = buildStandardFetch();\n\n    let clearedIntervalId = null;\n    const origClearInterval = w.clearInterval.bind(w);\n    w.clearInterval = function (id) {\n        clearedIntervalId = id;\n        return origClearInterval(id);\n    };\n\n    let capturedIntervalId = null;\n    const origSetInterval = w.setInterval.bind(w);\n    w.setInterval = function (fn, delay) {\n        const id = origSetInterval(fn, 999999);\n        capturedIntervalId = id;\n        return id;\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    const container = w.document.getElementById('app');\n    const cleanup = w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    assert.ok(capturedIntervalId !== null, 'setInterval was not called \u2014 no polling started');\n\n    // Call the cleanup function (simulating navigation away)\n    cleanup();\n\n    assert.ok(clearedIntervalId !== null, 'clearInterval was not called by the cleanup function');\n});\n\ntest('AC4: cleanup function is safe to call multiple times without error', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    const cleanup = w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Should not throw\n    cleanup();\n    cleanup();\n    cleanup();\n});\n\ntest('AC4: polling stops if container is disconnected from DOM', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    let statusCallCount = 0;\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        if (method === 'GET' && /\\/workspaces\\/DEV\\/status$/.test(url)) {\n            statusCallCount++;\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleStatus };\n        }\n        if (method === 'GET' && /\\/workspaces\\/DEV$/.test(url)) {\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleWorkspace };\n        }\n        if (method === 'GET' && /\\/api\\/projects\\/my-project$/.test(url)) {\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => sampleProject };\n        }\n        return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => ({}) };\n    };\n\n    let intervalCallback = null;\n    const origSetInterval = w.setInterval.bind(w);\n    w.setInterval = function (fn, delay) {\n        intervalCallback = fn;\n        return origSetInterval(fn, 999999);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Simulate navigation away \u2014 disconnect container\n    container.innerHTML = '';\n    w.document.body.removeChild(container);\n\n    const countBefore = statusCallCount;\n    // Fire the interval \u2014 it should check isConnected and not call fetch again\n    if (intervalCallback) {\n        await intervalCallback();\n        await flushPromises(w);\n    }\n    // Status call count should not have increased (or at most incremented by the\n    // async fetch that was in-flight when disconnected \u2014 but the result is ignored)\n    // We accept up to 1 extra call due to the async nature, but it should not keep going\n    assert.ok(\n        statusCallCount <= countBefore + 1,\n        `Polling continued after container disconnected: ${statusCallCount} calls (was ${countBefore})`\n    );\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// AC5: \"Switch Branches\" button navigates to branch-switch route\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('AC5 \u2014 \"Switch Branches\" button navigation');\n\ntest('AC5: \"Switch Branches\" button is rendered', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const switchBtn = buttons.find(b => b.textContent.trim().toLowerCase().includes('switch'));\n    assert.ok(switchBtn, 'No \"Switch Branches\" button found');\n});\n\ntest('AC5: clicking \"Switch Branches\" navigates via router.navigate()', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n\n    let navigatedTo = null;\n    const mockRouter = {\n        navigate(hash) { navigatedTo = hash; },\n    };\n    w.setRouter(mockRouter);\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const switchBtn = buttons.find(b => b.textContent.trim().toLowerCase().includes('switch'));\n    assert.ok(switchBtn, 'No \"Switch Branches\" button found');\n\n    switchBtn.click();\n\n    assert.ok(navigatedTo, 'router.navigate() not called after clicking \"Switch Branches\"');\n    assert.ok(\n        navigatedTo.includes('branch-switch'),\n        `Expected navigation to branch-switch route, got: ${navigatedTo}`\n    );\n    assert.ok(\n        navigatedTo.includes('my-project'),\n        `Expected navigation URL to include project ID, got: ${navigatedTo}`\n    );\n    assert.ok(\n        navigatedTo.includes('DEV'),\n        `Expected navigation URL to include workspace ID, got: ${navigatedTo}`\n    );\n});\n\ntest('AC5: branch-switch URL has correct format #/projects/:id/workspaces/:wid/branch-switch', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n\n    let navigatedTo = null;\n    const mockRouter = { navigate(hash) { navigatedTo = hash; } };\n    w.setRouter(mockRouter);\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'proj-x', wid: 'FEAT' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const switchBtn = buttons.find(b => b.textContent.trim().toLowerCase().includes('switch'));\n    switchBtn.click();\n\n    assert.strictEqual(\n        navigatedTo,\n        '#/projects/proj-x/workspaces/FEAT/branch-switch',\n        `Branch-switch URL mismatch. Got: ${navigatedTo}`\n    );\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// AC6: Rename and Delete disabled for STABLE; functional for others\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('AC6 \u2014 Rename and Delete disabled for STABLE, functional for others');\n\ntest('AC6: Rename button is disabled for STABLE workspace', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch({ workspace: stableWorkspace }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'STABLE' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const renameBtn = buttons.find(b => b.textContent.toLowerCase().includes('rename'));\n    assert.ok(renameBtn, 'Rename button not found for STABLE workspace');\n    assert.ok(renameBtn.disabled, 'Rename button is not disabled for STABLE workspace');\n});\n\ntest('AC6: Delete button is disabled for STABLE workspace', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch({ workspace: stableWorkspace }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'STABLE' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const deleteBtn = buttons.find(b => b.textContent.toLowerCase().includes('delete'));\n    assert.ok(deleteBtn, 'Delete button not found for STABLE workspace');\n    assert.ok(deleteBtn.disabled, 'Delete button is not disabled for STABLE workspace');\n});\n\ntest('AC6: Rename button has a title/tooltip explaining STABLE restriction', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch({ workspace: stableWorkspace }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'STABLE' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const renameBtn = buttons.find(b => b.textContent.toLowerCase().includes('rename'));\n    assert.ok(renameBtn && renameBtn.title, 'Rename button has no title attribute for STABLE workspace');\n});\n\ntest('AC6: Delete button has a title/tooltip explaining STABLE restriction', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch({ workspace: stableWorkspace }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'STABLE' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const deleteBtn = buttons.find(b => b.textContent.toLowerCase().includes('delete'));\n    assert.ok(deleteBtn && deleteBtn.title, 'Delete button has no title attribute for STABLE workspace');\n});\n\ntest('AC6: Rename button is enabled for non-STABLE workspace', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const renameBtn = buttons.find(b => b.textContent.toLowerCase().includes('rename'));\n    assert.ok(renameBtn, 'Rename button not found for DEV workspace');\n    assert.ok(!renameBtn.disabled, 'Rename button is unexpectedly disabled for DEV workspace');\n});\n\ntest('AC6: Delete button is enabled for non-STABLE workspace', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const deleteBtn = buttons.find(b => b.textContent.toLowerCase().includes('delete'));\n    assert.ok(deleteBtn, 'Delete button not found for DEV workspace');\n    assert.ok(!deleteBtn.disabled, 'Delete button is unexpectedly disabled for DEV workspace');\n});\n\ntest('AC6: clicking Rename button for non-STABLE shows inline form', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const buttons = [...container.querySelectorAll('button')];\n    const renameBtn = buttons.find(b => b.textContent.trim() === 'Rename Workspace');\n    assert.ok(renameBtn, 'Rename Workspace button not found');\n    renameBtn.click();\n\n    // Form should now be visible\n    const formWrapper = container.querySelector('.rename-workspace-form-wrapper');\n    assert.ok(formWrapper, '.rename-workspace-form-wrapper not found');\n    assert.ok(!formWrapper.hidden, 'Rename form is still hidden after clicking Rename button');\n});\n\ntest('AC6: rename form validates against invalid ID (lowercase)', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Open rename form\n    const renameBtn = [...container.querySelectorAll('button')].find(b => b.textContent.trim() === 'Rename Workspace');\n    renameBtn.click();\n\n    const input = container.querySelector('[name=\"newWorkspaceId\"]');\n    assert.ok(input, 'newWorkspaceId input not found');\n    input.value = 'lowercase'; // invalid \u2014 should fail validation\n\n    const saveBtn = [...container.querySelectorAll('button')].find(b => b.textContent.includes('Save'));\n    assert.ok(saveBtn, 'Save button not found');\n    saveBtn.click();\n    await flushPromises(w);\n\n    // Error should be shown\n    const errorEl = container.querySelector('.field-error');\n    assert.ok(errorEl && !errorEl.hidden, 'Validation error not shown for invalid rename input \"lowercase\"');\n});\n\ntest('AC6: rename form validates against ID that is too short (1 char)', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const renameBtn = [...container.querySelectorAll('button')].find(b => b.textContent.trim() === 'Rename Workspace');\n    renameBtn.click();\n\n    const input = container.querySelector('[name=\"newWorkspaceId\"]');\n    input.value = 'A'; // too short\n    const saveBtn = [...container.querySelectorAll('button')].find(b => b.textContent.includes('Save'));\n    saveBtn.click();\n    await flushPromises(w);\n\n    const errorEl = container.querySelector('.field-error');\n    assert.ok(errorEl && !errorEl.hidden, 'Validation error not shown for too-short rename input \"A\"');\n});\n\ntest('AC6: valid rename (e.g. \"FEAT\") calls api.workspaces.rename() and shows confirm dialog', async () => {\n    const calledUrls = [];\n    let renamedTo = null;\n    let confirmShown = false;\n\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        calledUrls.push(url);\n        if (method === 'PUT' && url.includes('/rename')) {\n            const body = JSON.parse((opts || {}).body || '{}');\n            renamedTo = body.newId;\n            return { ok: true, status: 200, headers: { get: () => 'application/json' }, json: async () => ({}) };\n        }\n        return buildStandardFetch()(url, opts);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    // Mock showConfirm to auto-accept and track call\n    w.showConfirm = async () => { confirmShown = true; return true; };\n\n    let navigatedTo = null;\n    w.setRouter({ navigate(h) { navigatedTo = h; } });\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Open rename form\n    const renameBtn = [...container.querySelectorAll('button')].find(b => b.textContent.trim() === 'Rename Workspace');\n    renameBtn.click();\n\n    const input = container.querySelector('[name=\"newWorkspaceId\"]');\n    input.value = 'FEAT';\n\n    const saveBtn = [...container.querySelectorAll('button')].find(b => b.textContent.includes('Save'));\n    saveBtn.click();\n    await flushPromises(w);\n\n    assert.ok(confirmShown, 'showConfirm not called before rename');\n    assert.ok(renamedTo === 'FEAT', `api.workspaces.rename not called with \"FEAT\". Got: ${renamedTo}`);\n    assert.ok(\n        navigatedTo && navigatedTo.includes('FEAT'),\n        `Expected navigation to new workspace URL containing \"FEAT\". Got: ${navigatedTo}`\n    );\n});\n\ntest('AC6: Delete workspace shows confirm dialog and calls api.workspaces.delete()', async () => {\n    let deleteUrl = null;\n    let confirmShown = false;\n\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        if (method === 'DELETE' && url.includes('/workspaces/DEV')) {\n            deleteUrl = url;\n            return { ok: true, status: 204, headers: { get: () => '' }, json: async () => undefined };\n        }\n        return buildStandardFetch()(url, opts);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    w.showConfirm = async () => { confirmShown = true; return true; };\n\n    let navigatedTo = null;\n    w.setRouter({ navigate(h) { navigatedTo = h; } });\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const deleteBtn = [...container.querySelectorAll('button')].find(b => b.textContent.toLowerCase().includes('delete'));\n    assert.ok(deleteBtn, 'Delete button not found');\n    deleteBtn.click();\n    await flushPromises(w);\n\n    assert.ok(confirmShown, 'showConfirm not called before delete');\n    assert.ok(deleteUrl, `api.workspaces.delete() not called. DELETE URL not captured.`);\n    assert.ok(deleteUrl.includes('DEV'), `DELETE URL does not include workspace ID \"DEV\": ${deleteUrl}`);\n    assert.ok(\n        navigatedTo && navigatedTo.includes('my-project') && !navigatedTo.includes('DEV'),\n        `Expected navigation back to project (not workspace). Got: ${navigatedTo}`\n    );\n});\n\ntest('AC6: Delete workspace navigates to #/projects/:id after success', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        if (method === 'DELETE') {\n            return { ok: true, status: 204, headers: { get: () => '' }, json: async () => undefined };\n        }\n        return buildStandardFetch()(url, opts);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    w.showConfirm = async () => true;\n\n    let navigatedTo = null;\n    w.setRouter({ navigate(h) { navigatedTo = h; } });\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const deleteBtn = [...container.querySelectorAll('button')].find(b => b.textContent.toLowerCase().includes('delete'));\n    deleteBtn.click();\n    await flushPromises(w);\n\n    assert.strictEqual(\n        navigatedTo,\n        '#/projects/my-project',\n        `Expected navigation to \"#/projects/my-project\" after delete. Got: ${navigatedTo}`\n    );\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// Edge cases\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nsection('Edge Cases');\n\ntest('Edge: renders error state with back link when API fetch fails', async () => {\n    const w = buildWindowWithDeps(async () => {\n        throw new Error('Network error');\n    });\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const backLink = container.querySelector('a');\n    assert.ok(backLink, 'No back link rendered after API failure');\n    assert.ok(\n        backLink.href.includes('my-project') || backLink.textContent.toLowerCase().includes('back'),\n        `Back link does not reference the project. href=\"${backLink.href}\", text=\"${backLink.textContent}\"`\n    );\n    assert.ok(\n        container.textContent.toLowerCase().includes('fail') ||\n        container.textContent.toLowerCase().includes('error'),\n        'No error/fail message rendered after API failure'\n    );\n});\n\ntest('Edge: repository entries as objects with Id/Name are normalised correctly', async () => {\n    const projectWithObjectRepos = {\n        Id: 'obj-proj',\n        Name: 'Object Repos Project',\n        Repositories: [\n            { Id: 'repo-one', Name: 'Repo One' },\n            { Id: 'repo-two', Name: 'Repo Two' },\n        ],\n    };\n    const statusForObjectRepos = {\n        'repo-one': { currentBranch: 'main', localCommits: 0, unfetchedCommits: 0, modifiedFiles: 0, lastActivity: null, hasConflicts: false },\n        'repo-two': { currentBranch: 'dev',  localCommits: 0, unfetchedCommits: 0, modifiedFiles: 0, lastActivity: null, hasConflicts: false },\n    };\n    const w = buildWindowWithDeps(buildStandardFetch({ project: projectWithObjectRepos, status: statusForObjectRepos }));\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'obj-proj', wid: 'DEV' });\n    await flushPromises(w);\n\n    const rows = container.querySelectorAll('tbody tr');\n    assert.strictEqual(rows.length, 2, `Expected 2 rows for object-based repos, got ${rows.length}`);\n    const repoIds = [...rows].map(r => r.dataset.repoId);\n    assert.ok(repoIds.includes('repo-one'), 'Row for repo-one not found');\n    assert.ok(repoIds.includes('repo-two'), 'Row for repo-two not found');\n});\n\ntest('Edge: rename cancel hides form and re-shows Rename button', async () => {\n    const w = buildWindowWithDeps(buildStandardFetch());\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    const renameBtn = [...container.querySelectorAll('button')].find(b => b.textContent.trim() === 'Rename Workspace');\n    renameBtn.click();\n\n    const formWrapper = container.querySelector('.rename-workspace-form-wrapper');\n    assert.ok(!formWrapper.hidden, 'Form should be visible after clicking rename');\n\n    const cancelBtn = [...container.querySelectorAll('button')].find(b => b.textContent.trim() === 'Cancel');\n    assert.ok(cancelBtn, 'Cancel button not found');\n    cancelBtn.click();\n\n    assert.ok(formWrapper.hidden, 'Form should be hidden after clicking Cancel');\n    assert.ok(!renameBtn.hidden, 'Rename button should be visible again after Cancel');\n});\n\ntest('Edge: polling silently handles API error without crashing the view', async () => {\n    const dom = makeDOM();\n    const w = dom.window;\n    if (!w.CSS) w.CSS = {};\n    if (!w.CSS.escape) w.CSS.escape = (v) => String(v).replace(/[^\\w-]/g, c => '\\\\' + c);\n\n    let isPollingCall = false;\n    w.fetch = async (url, opts) => {\n        const method = (opts || {}).method || 'GET';\n        if (method === 'GET' && /\\/workspaces\\/DEV\\/status$/.test(url) && isPollingCall) {\n            throw new Error('Polling network failure');\n        }\n        return buildStandardFetch()(url, opts);\n    };\n\n    let intervalCallback = null;\n    const origSetInterval = w.setInterval.bind(w);\n    w.setInterval = function (fn, delay) {\n        intervalCallback = fn;\n        return origSetInterval(fn, 999999);\n    };\n\n    w.eval(readAndStrip('gui/public/js/components/toast.js'));\n    w.eval(readAndStrip('gui/public/js/components/confirm-dialog.js'));\n    w.eval(readAndStrip('gui/public/js/components/form-helpers.js'));\n    w.eval(readAndStrip('gui/public/js/components/status-badge.js'));\n    w.eval(readAndStrip('gui/public/js/api.js'));\n    w.eval(readAndStrip('gui/public/js/views/workspace-detail.js'));\n\n    const container = w.document.getElementById('app');\n    w.renderWorkspaceDetail(container, { id: 'my-project', wid: 'DEV' });\n    await flushPromises(w);\n\n    // Table should still exist before polling error\n    const headingBefore = container.querySelector('h1');\n    assert.ok(headingBefore, 'Heading missing before polling error');\n\n    // Fire polling with simulated error\n    isPollingCall = true;\n    if (intervalCallback) {\n        await intervalCallback().catch(() => {}); // should not throw\n    }\n    await flushPromises(w);\n\n    // View should be intact \u2014 heading still present\n    const headingAfter = container.querySelector('h1');\n    assert.ok(headingAfter, 'View was destroyed after polling error \u2014 should have been resilient');\n});\n\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n// Summary\n// \u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\n\nawait Promise.allSettled(promises);\n\nconsole.log('\\n' + '\u2550'.repeat(55));\nconsole.log(`  Total: ${passed + failed}  |  \u2705 Passed: ${passed}  |  \u274c Failed: ${failed}`);\nconsole.log('\u2550'.repeat(55));\n\nif (failures.length > 0) {\n    console.log('\\nFailed tests:');\n    failures.forEach((f) => console.log(`  \u274c ${f.name}\\n     ${f.error}`));\n    process.exit(1);\n}\n\nprocess.exit(0);\n"
}
```

## Tool Result

Updated file /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/qa-wp016-tests.mjs

## Assistant

```json
{
  "id": "toolu_01VojUYph73dV1aW7knWetnZ",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Write QA test suite qa-wp016-tests.mjs covering all 6 ACs",
        "status": "completed"
      },
      {
        "content": "Run the test suite and verify all tests pass",
        "status": "in_progress"
      },
      {
        "content": "Run full server test regression and verify 517 still pass",
        "status": "pending"
      },
      {
        "content": "Complete the QA pipeline in the ledger",
        "status": "pending"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_01VojUYph73dV1aW7knWetnZ`)

```json
{
  "todos": [
    {
      "content": "Write QA test suite qa-wp016-tests.mjs covering all 6 ACs",
      "status": "completed"
    },
    {
      "content": "Run the test suite and verify all tests pass",
      "status": "in_progress"
    },
    {
      "content": "Run full server test regression and verify 517 still pass",
      "status": "pending"
    },
    {
      "content": "Complete the QA pipeline in the ledger",
      "status": "pending"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Write QA test suite qa-wp016-tests.mjs covering all 6 ACs', 'status': 'completed'}, {'content': 'Run the test suite and verify all tests pass', 'status': 'in_progress'}, {'content': 'Run full server test regression and verify 517 still pass', 'status': 'pending'}, {'content': 'Complete the QA pipeline in the ledger', 'status': 'pending'}]

## Assistant

```json
{
  "id": "toolu_01WTRvNHVJUtuydCVTNFWcGA",
  "caller": {
    "type": "direct"
  },
  "input": {
    "command": "cd /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE && node qa-wp016-tests.mjs 2>&1"
  },
  "name": "execute",
  "type": "tool_use"
}
```

**Tool call:** `execute` (id: `toolu_01WTRvNHVJUtuydCVTNFWcGA`)

```json
{
  "command": "cd /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE && node qa-wp016-tests.mjs 2>&1"
}
```

## Tool Result


━━ Static analysis — workspace-detail.js ━━
  ✅  workspace-detail.js imports api from api.js
  ✅  workspace-detail.js imports createStatusBadge from status-badge.js
  ✅  workspace-detail.js imports showToast
  ✅  workspace-detail.js imports showConfirm
  ✅  workspace-detail.js exports renderWorkspaceDetail
  ✅  workspace-detail.js exports setRouter
  ✅  workspace-detail.js calls api.workspaces.get()
  ✅  workspace-detail.js calls api.projects.get()
  ✅  workspace-detail.js calls api.status.get()
  ✅  workspace-detail.js uses setInterval for polling
  ✅  workspace-detail.js uses clearInterval for cleanup
  ✅  workspace-detail.js defines STABLE workspace constant
  ✅  workspace-detail.js disables rename for STABLE
  ✅  workspace-detail.js calls api.workspaces.rename()
  ✅  workspace-detail.js calls api.workspaces.delete()
  ✅  workspace-detail.js validates workspace rename ID with /^[A-Z]{2,6}$/ regex
  ✅  workspace-detail.js returns cleanup function from renderWorkspaceDetail
  ✅  app.js imports renderWorkspaceDetail
  ✅  app.js imports setRouter as setWorkspaceDetailRouter
  ✅  app.js registers renderWorkspaceDetail for '#/projects/:id/workspaces/:wid'
  ✅  app.js calls setWorkspaceDetailRouter(router)

━━ AC1 — Renders workspace header and repository status table ━━

━━ AC2 — Repository rows: branch name + color-coded status badge ━━

━━ AC3 — In-place polling via setInterval ━━

━━ AC4 — Polling interval cleared on navigation away ━━

━━ AC5 — "Switch Branches" button navigation ━━

━━ AC6 — Rename and Delete disabled for STABLE, functional for others ━━

━━ Edge Cases ━━
  ✅  AC4: renderWorkspaceDetail returns a cleanup function
  ✅  AC1: renders workspace ID in a heading
  ✅  AC1: renders breadcrumb link back to the project
  ✅  AC1: renders workspace description
  ✅  AC1: renders a repository status table
  ✅  AC1: table has one row per repository (2 repos)
  ✅  AC1: each repository row has data-repo-id attribute
  ✅  AC1: shows "No repositories" message when project has empty repos list
  ✅  AC1: fetches workspace, project, and status in parallel (all three URLs called)
  ✅  AC2: repo-alpha row shows current branch "main"
  ✅  AC2: repo-beta row shows current branch "feature/x"
  ✅  AC2: status badge wrapper exists with data-repo-id for each row
  ✅  AC2: conflict badge rendered for repo-beta (hasConflicts=true)
  ✅  AC2: modified-files badge shown for repo-alpha (modifiedFiles=3)
  ✅  AC2: null status entry renders error/no-data badge
  ✅  AC3: renderWorkspaceDetail calls setInterval after initial render
  ✅  AC4: cleanup function calls clearInterval
  ✅  AC4: cleanup function is safe to call multiple times without error
  ✅  AC5: "Switch Branches" button is rendered
  ✅  AC5: clicking "Switch Branches" navigates via router.navigate()
  ✅  AC5: branch-switch URL has correct format #/projects/:id/workspaces/:wid/branch-switch
  ✅  AC6: Rename button is disabled for STABLE workspace
  ✅  AC6: Delete button is disabled for STABLE workspace
  ✅  AC6: Rename button has a title/tooltip explaining STABLE restriction
  ✅  AC6: Delete button has a title/tooltip explaining STABLE restriction
  ✅  AC6: Rename button is enabled for non-STABLE workspace
  ✅  AC6: Delete button is enabled for non-STABLE workspace
  ✅  AC6: clicking Rename button for non-STABLE shows inline form
  ✅  Edge: renders error state with back link when API fetch fails
  ✅  Edge: repository entries as objects with Id/Name are normalised correctly
  ✅  Edge: rename cancel hides form and re-shows Rename button
  ✅  AC3: polling calls GET .../status endpoint again after interval fires
  ✅  AC3: polling updates branch name in-place (no full re-render)
  ✅  AC4: polling stops if container is disconnected from DOM
  ✅  AC6: rename form validates against invalid ID (lowercase)
  ✅  AC6: rename form validates against ID that is too short (1 char)
  ✅  AC6: valid rename (e.g. "FEAT") calls api.workspaces.rename() and shows confirm dialog
  ✅  AC6: Delete workspace shows confirm dialog and calls api.workspaces.delete()
  ✅  AC6: Delete workspace navigates to #/projects/:id after success
  ✅  Edge: polling silently handles API error without crashing the view

═══════════════════════════════════════════════════════
  Total: 61  |  ✅ Passed: 61  |  ❌ Failed: 0
═══════════════════════════════════════════════════════

[Command succeeded with exit code 0]

## Assistant

61/61 pass. Now run the full server regression.

```json
{
  "id": "toolu_01UwKAEpyo7X9rvNUoJ7KJcS",
  "caller": {
    "type": "direct"
  },
  "input": {
    "command": "cd /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE && npm test 2>&1 | tail -15"
  },
  "name": "execute",
  "type": "tool_use"
}
```

**Tool call:** `execute` (id: `toolu_01UwKAEpyo7X9rvNUoJ7KJcS`)

```json
{
  "command": "cd /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE && npm test 2>&1 | tail -15"
}
```

## Tool Result

✔ remove throws when project does not exist (1.362416ms)
✔ remove throws NotFoundError when project does not exist (0.712917ms)
✔ remove throws when workspace does not exist (0.747ms)
✔ remove deletes the workspace so list() no longer includes it (0.945958ms)
✔ remove does not affect other workspaces in the project (1.163583ms)
✔ CRUD round-trip: create → list → getById → update → rename → remove (1.386958ms)
✔ workspace operations on project A do not affect project B workspaces (1.959833ms)
ℹ tests 517
ℹ suites 0
ℹ pass 517
ℹ fail 0
ℹ cancelled 0
ℹ skipped 0
ℹ todo 0
ℹ duration_ms 7797.608625

[Command succeeded with exit code 0]

## Assistant

517 server tests still all pass. Now complete the QA pipeline in the ledger.

```json
{
  "id": "toolu_01B4MwpNgSF6k1B6ReNmPJ8o",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Write QA test suite qa-wp016-tests.mjs covering all 6 ACs",
        "status": "completed"
      },
      {
        "content": "Run the test suite and verify all tests pass",
        "status": "completed"
      },
      {
        "content": "Run full server test regression and verify 517 still pass",
        "status": "completed"
      },
      {
        "content": "Complete the QA pipeline in the ledger",
        "status": "in_progress"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_01B4MwpNgSF6k1B6ReNmPJ8o`)

```json
{
  "todos": [
    {
      "content": "Write QA test suite qa-wp016-tests.mjs covering all 6 ACs",
      "status": "completed"
    },
    {
      "content": "Run the test suite and verify all tests pass",
      "status": "completed"
    },
    {
      "content": "Run full server test regression and verify 517 still pass",
      "status": "completed"
    },
    {
      "content": "Complete the QA pipeline in the ledger",
      "status": "in_progress"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Write QA test suite qa-wp016-tests.mjs covering all 6 ACs', 'status': 'completed'}, {'content': 'Run the test suite and verify all tests pass', 'status': 'completed'}, {'content': 'Run full server test regression and verify 517 still pass', 'status': 'completed'}, {'content': 'Complete the QA pipeline in the ledger', 'status': 'in_progress'}]

## Assistant

```json
{
  "id": "toolu_01LSwHjkiVGXEUwqnTfCLSPq",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-016",
    "type": "qa",
    "status": "PASS",
    "agent_role": "QA",
    "summary": [
      "Wrote qa-wp016-tests.mjs: 61 tests covering all 6 AC groups plus edge cases, all executed with jsdom/Node (same pattern as WP-013/014/015 suites).",
      "61/61 tests PASS. 517/517 server-side regression tests still pass \u2014 no regressions introduced.",
      "AC1 (header + table): 7 tests \u2014 workspace header, breadcrumb link, description, table structure, data-repo-id rows, empty-project fallback, and parallel fetch verification.",
      "AC2 (branch + badge): 6 tests \u2014 branch names rendered per row, badge wrapper per repo, conflict badge for hasConflicts=true, modified-files count, null-status error badge.",
      "AC3 (in-place polling): 3 tests \u2014 setInterval called after render, polling tick calls status endpoint again, branch and badge updated in-place without full re-render (same heading DOM node after poll).",
      "AC4 (interval cleanup): 4 tests \u2014 cleanup function returned, clearInterval called by cleanup, idempotent multi-call safety, disconnected-container guard stops polling.",
      "AC5 (Switch Branches): 3 tests \u2014 button rendered, router.navigate() called with correct branch-switch URL, URL format verified for arbitrary project/workspace IDs.",
      "AC6 (STABLE guards + actions): 11 tests \u2014 rename/delete disabled for STABLE (with title tooltips), enabled for DEV, rename form toggle, form validation (lowercase/too-short), valid rename calls api.workspaces.rename() with confirm + navigates to new URL, delete calls api.workspaces.delete() with confirm + navigates back to project.",
      "Edge cases: 4 additional tests \u2014 API fetch failure renders error state with back link, object-shaped repos normalised correctly, rename cancel restores button, polling errors are silently swallowed without destroying the view."
    ],
    "metrics": {
      "tests_passed": 61,
      "tests_failed": 0,
      "test_coverage": "All 6 AC groups + edge cases covered"
    },
    "artifacts": {
      "files_modified": [
        "qa-wp016-tests.mjs"
      ]
    },
    "acceptance_criteria_updates": [
      {
        "criterion": "Navigating to #/projects/:id/workspaces/:wid renders the workspace header and a repository status table with one row per repository.",
        "met": true
      },
      {
        "criterion": "Each repository row displays the current branch name, a color-coded Git status badge showing modified files, commits ahead/behind, last activity, and conflict indicator.",
        "met": true
      },
      {
        "criterion": "The status table auto-refreshes via setInterval calling GET /api/projects/:id/workspaces/:wid/status and updates badges in-place without re-rendering the entire view.",
        "met": true
      },
      {
        "criterion": "The polling interval is cleared when navigating away from the workspace detail view (no leaked intervals after navigation).",
        "met": true
      },
      {
        "criterion": "The 'Switch Branches' button navigates to #/projects/:id/workspaces/:wid/branch-switch.",
        "met": true
      },
      {
        "criterion": "Rename and Delete actions are disabled for STABLE workspace and functional with confirmation dialogs for other workspaces.",
        "met": true
      }
    ],
    "comments": [
      {
        "type": "improvement",
        "priority": "low",
        "note": "The `updateStatusTable()` function only updates rows already in the DOM \u2014 it does not handle repositories added to the project after the initial render. This is fine for current usage since the repo list comes from the project object fetched on load, but a future iteration could refresh the full table on a project-change event."
      },
      {
        "type": "improvement",
        "priority": "low",
        "note": "No visual loading indicator is shown during polling refreshes (only on initial load). A subtle 'Refreshing\u2026' indicator or spinner on the badge wrappers during the in-flight poll would improve perceived responsiveness. Low-priority cosmetic gap."
      },
      {
        "type": "convention",
        "priority": "low",
        "note": "The rename validation regex /^[A-Z]{2,6}$/ is duplicated in both workspace-detail.js (rename form) and project-detail.js (add workspace form). Noted by Developer; a shared validator in form-helpers.js or utils.js would eliminate the duplication. No functional impact."
      }
    ],
    "handoff_notes": [
      "All 6 AC are met and verified by 61 automated tests (qa-wp016-tests.mjs).",
      "Key verified behaviours: parallel data fetch on mount; in-place badge/branch update via updateStatusTable() (same heading DOM node before/after poll); cleanup function clears setInterval and is idempotent; STABLE guards disable both Rename and Delete with title tooltips; Switch Branches navigates to exact #/projects/:id/workspaces/:wid/branch-switch pattern; rename validates /^[A-Z]{2,6}$/ before showing confirm dialog; delete navigates back to #/projects/:id.",
      "No regressions: 517 server tests continue to pass.",
      "Minor low-priority notes for Reviewer: (1) updateStatusTable only handles repos present at load time; (2) no polling-tick loading indicator; (3) rename regex duplicated in project-detail.js \u2014 none are blockers."
    ],
    "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
  },
  "name": "ledger_complete_pipeline",
  "type": "tool_use"
}
```

**Tool call:** `ledger_complete_pipeline` (id: `toolu_01LSwHjkiVGXEUwqnTfCLSPq`)

```json
{
  "work_package_id": "WP-016",
  "type": "qa",
  "status": "PASS",
  "agent_role": "QA",
  "summary": [
    "Wrote qa-wp016-tests.mjs: 61 tests covering all 6 AC groups plus edge cases, all executed with jsdom/Node (same pattern as WP-013/014/015 suites).",
    "61/61 tests PASS. 517/517 server-side regression tests still pass \u2014 no regressions introduced.",
    "AC1 (header + table): 7 tests \u2014 workspace header, breadcrumb link, description, table structure, data-repo-id rows, empty-project fallback, and parallel fetch verification.",
    "AC2 (branch + badge): 6 tests \u2014 branch names rendered per row, badge wrapper per repo, conflict badge for hasConflicts=true, modified-files count, null-status error badge.",
    "AC3 (in-place polling): 3 tests \u2014 setInterval called after render, polling tick calls status endpoint again, branch and badge updated in-place without full re-render (same heading DOM node after poll).",
    "AC4 (interval cleanup): 4 tests \u2014 cleanup function returned, clearInterval called by cleanup, idempotent multi-call safety, disconnected-container guard stops polling.",
    "AC5 (Switch Branches): 3 tests \u2014 button rendered, router.navigate() called with correct branch-switch URL, URL format verified for arbitrary project/workspace IDs.",
    "AC6 (STABLE guards + actions): 11 tests \u2014 rename/delete disabled for STABLE (with title tooltips), enabled for DEV, rename form toggle, form validation (lowercase/too-short), valid rename calls api.workspaces.rename() with confirm + navigates to new URL, delete calls api.workspaces.delete() with confirm + navigates back to project.",
    "Edge cases: 4 additional tests \u2014 API fetch failure renders error state with back link, object-shaped repos normalised correctly, rename cancel restores button, polling errors are silently swallowed without destroying the view."
  ],
  "metrics": {
    "tests_passed": 61,
    "tests_failed": 0,
    "test_coverage": "All 6 AC groups + edge cases covered"
  },
  "artifacts": {
    "files_modified": [
      "qa-wp016-tests.mjs"
    ]
  },
  "acceptance_criteria_updates": [
    {
      "criterion": "Navigating to #/projects/:id/workspaces/:wid renders the workspace header and a repository status table with one row per repository.",
      "met": true
    },
    {
      "criterion": "Each repository row displays the current branch name, a color-coded Git status badge showing modified files, commits ahead/behind, last activity, and conflict indicator.",
      "met": true
    },
    {
      "criterion": "The status table auto-refreshes via setInterval calling GET /api/projects/:id/workspaces/:wid/status and updates badges in-place without re-rendering the entire view.",
      "met": true
    },
    {
      "criterion": "The polling interval is cleared when navigating away from the workspace detail view (no leaked intervals after navigation).",
      "met": true
    },
    {
      "criterion": "The 'Switch Branches' button navigates to #/projects/:id/workspaces/:wid/branch-switch.",
      "met": true
    },
    {
      "criterion": "Rename and Delete actions are disabled for STABLE workspace and functional with confirmation dialogs for other workspaces.",
      "met": true
    }
  ],
  "comments": [
    {
      "type": "improvement",
      "priority": "low",
      "note": "The `updateStatusTable()` function only updates rows already in the DOM \u2014 it does not handle repositories added to the project after the initial render. This is fine for current usage since the repo list comes from the project object fetched on load, but a future iteration could refresh the full table on a project-change event."
    },
    {
      "type": "improvement",
      "priority": "low",
      "note": "No visual loading indicator is shown during polling refreshes (only on initial load). A subtle 'Refreshing\u2026' indicator or spinner on the badge wrappers during the in-flight poll would improve perceived responsiveness. Low-priority cosmetic gap."
    },
    {
      "type": "convention",
      "priority": "low",
      "note": "The rename validation regex /^[A-Z]{2,6}$/ is duplicated in both workspace-detail.js (rename form) and project-detail.js (add workspace form). Noted by Developer; a shared validator in form-helpers.js or utils.js would eliminate the duplication. No functional impact."
    }
  ],
  "handoff_notes": [
    "All 6 AC are met and verified by 61 automated tests (qa-wp016-tests.mjs).",
    "Key verified behaviours: parallel data fetch on mount; in-place badge/branch update via updateStatusTable() (same heading DOM node before/after poll); cleanup function clears setInterval and is idempotent; STABLE guards disable both Rename and Delete with title tooltips; Switch Branches navigates to exact #/projects/:id/workspaces/:wid/branch-switch pattern; rename validates /^[A-Z]{2,6}$/ before showing confirm dialog; delete navigates back to #/projects/:id.",
    "No regressions: 517 server tests continue to pass.",
    "Minor low-priority notes for Reviewer: (1) updateStatusTable only handles repos present at load time; (2) no polling-tick loading indicator; (3) rename regex duplicated in project-detail.js \u2014 none are blockers."
  ],
  "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
}
```

## Tool Result

{
  "work_package_id": "WP-016",
  "work_package_file": "work/WP-006.md",
  "status": "IN_PROGRESS",
  "assigned_to": "QA",
  "dependencies": [
    "WP-001",
    "WP-011",
    "WP-014"
  ],
  "acceptance_criteria": [
    {
      "criterion": "Navigating to #/projects/:id/workspaces/:wid renders the workspace header and a repository status table with one row per repository.",
      "met": true
    },
    {
      "criterion": "Each repository row displays the current branch name, a color-coded Git status badge showing modified files, commits ahead/behind, last activity, and conflict indicator.",
      "met": true
    },
    {
      "criterion": "The status table auto-refreshes via setInterval calling GET /api/projects/:id/workspaces/:wid/status and updates badges in-place without re-rendering the entire view.",
      "met": true
    },
    {
      "criterion": "The polling interval is cleared when navigating away from the workspace detail view (no leaked intervals after navigation).",
      "met": true
    },
    {
      "criterion": "The 'Switch Branches' button navigates to #/projects/:id/workspaces/:wid/branch-switch.",
      "met": true
    },
    {
      "criterion": "Rename and Delete actions are disabled for STABLE workspace and functional with confirmation dialogs for other workspaces.",
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
      "started_at": "2026-04-07T21:10:29Z",
      "completed_at": "2026-04-07T21:13:34Z",
      "duration_ms": 185000,
      "summary": [
        "Created gui/public/js/views/workspace-detail.js — the full workspace detail view with header, breadcrumb, repository status table, and actions section.",
        "Implemented in-place polling: setInterval (10 s) calls api.status.get() and calls updateStatusTable() which locates rows/badge wrappers by data-repo-id and replaces only badge DOM nodes — no full re-render.",
        "Cleanup contract: renderWorkspaceDetail returns a cleanup function that clears the pollingInterval. The router's existing _render() already stores and calls any function returned by a view, so no router.js changes were needed.",
        "Actions section: 'Switch Branches' navigates to #/projects/:id/workspaces/:wid/branch-switch. 'Rename Workspace' shows an inline form with /^[A-Z]{2,6}$/ validation and calls api.workspaces.rename(); on success navigates to the new workspace URL. 'Delete Workspace' shows showConfirm() dialog, calls api.workspaces.delete(), then navigates back to project detail. Both Rename and Delete are disabled (with title tooltip) when wid === 'STABLE'.",
        "Modified app.js: replaced the placeholderView stub for '#/projects/:id/workspaces/:wid' with the real renderWorkspaceDetail import; registered setWorkspaceDetailRouter(router) so programmatic navigation works.",
        "All 517 server-side tests continue to pass. Node --check confirmed no syntax errors in both modified/new files."
      ],
      "artifacts": {
        "files_modified": [
          "gui/public/js/views/workspace-detail.js",
          "gui/public/js/app.js"
        ]
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:13:34Z",
          "note": "workspace-detail.js: The in-place badge update in updateStatusTable() uses CSS.escape() for the [data-repo-id] attribute selector, which is correct and defensive. However, if a repo ID contains characters that are valid in data attributes but unusual in CSS selectors, a fallback via querySelectorAll('[data-repo-id]') with a manual filter loop would be marginally more robust. Low priority as repo IDs are currently validated to safe patterns."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:13:34Z",
          "note": "app.js: The placeholderView() factory function and its comment block are now only used by the Branch Switch stub. Once WP-017 (Branch Switch view) ships, placeholderView will be dead code and should be removed along with the comment block above it."
        },
        {
          "type": "debt",
          "priority": "low",
          "timestamp": "2026-04-07T21:13:34Z",
          "note": "workspace-detail.js: The Rename Workspace inline form validates the workspace ID with /^[A-Z]{2,6}$/ — the same pattern used in project-detail.js buildAddWorkspaceForm(). This pattern is duplicated in two places. A shared validator function in a utils module or form-helpers.js would eliminate the duplication."
        }
      ]
    },
    {
      "type": "qa",
      "status": "PASS",
      "started_at": "2026-04-07T21:14:11Z",
      "completed_at": "2026-04-07T21:18:59Z",
      "duration_ms": 288000,
      "summary": [
        "Wrote qa-wp016-tests.mjs: 61 tests covering all 6 AC groups plus edge cases, all executed with jsdom/Node (same pattern as WP-013/014/015 suites).",
        "61/61 tests PASS. 517/517 server-side regression tests still pass — no regressions introduced.",
        "AC1 (header + table): 7 tests — workspace header, breadcrumb link, description, table structure, data-repo-id rows, empty-project fallback, and parallel fetch verification.",
        "AC2 (branch + badge): 6 tests — branch names rendered per row, badge wrapper per repo, conflict badge for hasConflicts=true, modified-files count, null-status error badge.",
        "AC3 (in-place polling): 3 tests — setInterval called after render, polling tick calls status endpoint again, branch and badge updated in-place without full re-render (same heading DOM node after poll).",
        "AC4 (interval cleanup): 4 tests — cleanup function returned, clearInterval called by cleanup, idempotent multi-call safety, disconnected-container guard stops polling.",
        "AC5 (Switch Branches): 3 tests — button rendered, router.navigate() called with correct branch-switch URL, URL format verified for arbitrary project/workspace IDs.",
        "AC6 (STABLE guards + actions): 11 tests — rename/delete disabled for STABLE (with title tooltips), enabled for DEV, rename form toggle, form validation (lowercase/too-short), valid rename calls api.workspaces.rename() with confirm + navigates to new URL, delete calls api.workspaces.delete() with confirm + navigates back to project.",
        "Edge cases: 4 additional tests — API fetch failure renders error state with back link, object-shaped repos normalised correctly, rename cancel restores button, polling errors are silently swallowed without destroying the view."
      ],
      "artifacts": {
        "files_modified": [
          "qa-wp016-tests.mjs"
        ]
      },
      "metrics": {
        "test_coverage": "All 6 AC groups + edge cases covered",
        "tests_passed": 61,
        "tests_failed": 0
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:18:59Z",
          "note": "The `updateStatusTable()` function only updates rows already in the DOM — it does not handle repositories added to the project after the initial render. This is fine for current usage since the repo list comes from the project object fetched on load, but a future iteration could refresh the full table on a project-change event."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-04-07T21:18:59Z",
          "note": "No visual loading indicator is shown during polling refreshes (only on initial load). A subtle 'Refreshing…' indicator or spinner on the badge wrappers during the in-flight poll would improve perceived responsiveness. Low-priority cosmetic gap."
        },
        {
          "type": "convention",
          "priority": "low",
          "timestamp": "2026-04-07T21:18:59Z",
          "note": "The rename validation regex /^[A-Z]{2,6}$/ is duplicated in both workspace-detail.js (rename form) and project-detail.js (add workspace form). Noted by Developer; a shared validator in form-helpers.js or utils.js would eliminate the duplication. No functional impact."
        }
      ]
    }
  ],
  "status_changed_at": "2026-04-07T21:10:29Z",
  "last_updated": "2026-04-07T21:18:59Z",
  "handoff_notes": [
    {
      "from_agent": "Developer",
      "to_agent": "QA",
      "timestamp": "2026-04-07T21:13:34Z",
      "notes": [
        "All 6 acceptance criteria are met and marked as such in the ledger.",
        "New file: gui/public/js/views/workspace-detail.js — exports renderWorkspaceDetail(container, params) and setRouter(router).",
        "Modified file: gui/public/js/app.js — imports renderWorkspaceDetail and setWorkspaceDetailRouter; replaces the placeholder stub for '#/projects/:id/workspaces/:wid'.",
        "Cleanup/polling: the view returns a cleanup function; the router's existing _render() mechanism handles it — no router.js changes required.",
        "Key integration points for QA: (1) polling interval cleared on navigation, (2) STABLE workspace disables Rename and Delete, (3) rename navigates to the new workspace URL, (4) delete navigates back to project detail, (5) failed API load renders an error state with a back link."
      ]
    },
    {
      "from_agent": "QA",
      "to_agent": "Reviewer",
      "timestamp": "2026-04-07T21:18:59Z",
      "notes": [
        "All 6 AC are met and verified by 61 automated tests (qa-wp016-tests.mjs).",
        "Key verified behaviours: parallel data fetch on mount; in-place badge/branch update via updateStatusTable() (same heading DOM node before/after poll); cleanup function clears setInterval and is idempotent; STABLE guards disable both Rename and Delete with title tooltips; Switch Branches navigates to exact #/projects/:id/workspaces/:wid/branch-switch pattern; rename validates /^[A-Z]{2,6}$/ before showing confirm dialog; delete navigates back to #/projects/:id.",
        "No regressions: 517 server tests continue to pass.",
        "Minor low-priority notes for Reviewer: (1) updateStatusTable only handles repos present at load time; (2) no polling-tick loading indicator; (3) rename regex duplicated in project-detail.js — none are blockers."
      ]
    }
  ]
}

--- NEXT STEP ---
Pipeline PASS. Call ledger_get_handoff_status (current_agent: "QA") to confirm your work is done and hand off to Reviewer.

## Assistant

```json
{
  "id": "toolu_01DTLHzBZcUbcAszfaYCJMUN",
  "caller": {
    "type": "direct"
  },
  "input": {
    "agent_role": "QA",
    "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
  },
  "name": "ledger_get_next_action",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_next_action` (id: `toolu_01DTLHzBZcUbcAszfaYCJMUN`)

```json
{
  "agent_role": "QA",
  "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
}
```

## Tool Result

{"action": "WAIT", "reason": "Pipeline completed for the active work package. The orchestrator will route the next work package."}

## Assistant

```json
{
  "id": "toolu_01UxT7ddnY6zvzuRDn6whXbH",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Write QA test suite qa-wp016-tests.mjs covering all 6 ACs",
        "status": "completed"
      },
      {
        "content": "Run the test suite and verify all tests pass",
        "status": "completed"
      },
      {
        "content": "Run full server test regression and verify 517 still pass",
        "status": "completed"
      },
      {
        "content": "Complete the QA pipeline in the ledger",
        "status": "completed"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_01UxT7ddnY6zvzuRDn6whXbH`)

```json
{
  "todos": [
    {
      "content": "Write QA test suite qa-wp016-tests.mjs covering all 6 ACs",
      "status": "completed"
    },
    {
      "content": "Run the test suite and verify all tests pass",
      "status": "completed"
    },
    {
      "content": "Run full server test regression and verify 517 still pass",
      "status": "completed"
    },
    {
      "content": "Complete the QA pipeline in the ledger",
      "status": "completed"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Write QA test suite qa-wp016-tests.mjs covering all 6 ACs', 'status': 'completed'}, {'content': 'Run the test suite and verify all tests pass', 'status': 'completed'}, {'content': 'Run full server test regression and verify 517 still pass', 'status': 'completed'}, {'content': 'Complete the QA pipeline in the ledger', 'status': 'completed'}]

## Assistant

```json
{
  "id": "toolu_01XVAt3SsFNUXK2XYNvV8muf",
  "caller": {
    "type": "direct"
  },
  "input": {
    "current_agent": "QA",
    "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
  },
  "name": "ledger_get_handoff_status",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_handoff_status` (id: `toolu_01XVAt3SsFNUXK2XYNvV8muf`)

```json
{
  "current_agent": "QA",
  "project_path": "/Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
}
```

## Tool Result

{
  "current_agent": "QA",
  "next_agent": "Developer",
  "status": "READY_FOR_DEVELOPER",
  "details": "QA passed for 7 implemented work package(s). 9 work package(s) ready for implementation: WP-002, WP-003, WP-004, WP-005, WP-006, WP-007, WP-008, WP-009, WP-010. 1 blocked by dependencies.",
  "auto_handoff": {
    "agent_name": "3 - Developer v3.6.1",
    "agent_id": "ledger-3-dev",
    "prompt": "@ledger-3-dev\nProject path: /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
  }
}

## Assistant

```json
{
  "id": "toolu_01Wff3wAUnCi4usA16bR4JpW",
  "caller": {
    "type": "direct"
  },
  "input": {
    "subagent_type": "general-purpose",
    "description": "3-developer\n@ledger-3-dev\nProject path: /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
  },
  "name": "task",
  "type": "tool_use"
}
```

**Tool call:** `task` (id: `toolu_01Wff3wAUnCi4usA16bR4JpW`)

```json
{
  "subagent_type": "general-purpose",
  "description": "3-developer\n@ledger-3-dev\nProject path: /Users/smordziol/Webserver/tools/repo-parallelizer-STABLE/docs/agents/plans/2026-04-03-phase6-gui-frontend"
}
```

## Tool Result

I now have the full picture. Here's the status summary for **Developer (Agent 3)**:

---

## 📊 Current Status — Developer (Agent 3)

**Ledger Action: `WAIT`**

The Developer's next work package is **WP-017 (Branch Switch Wizard)**, but it is currently **BLOCKED** waiting for **WP-016** to reach COMPLETE status.

### What's happening right now:

| WP | Status | Who's working |
|---|---|---|
| **WP-015** (Repositories view) | IN_PROGRESS | **Reviewer** — running code-review pipeline |
| **WP-016** (Workspace Detail view) | IN_PROGRESS | Waiting for **Reviewer** to pick up code-review pipeline |
| **WP-017** (Branch Switch Wizard) | **BLOCKED** | Waiting for WP-016 → COMPLETE |

### What WP-017 needs (ready for when unblocked):

WP-017 is the **3-step Branch Switch Wizard** — the final view in this project:
- **Step 1**: Branch name text input + suggestions from `api.branches.list()`
- **Step 2**: Per-repo branch assignment table with `<select>` dropdowns + text inputs  
- **Step 3**: Results table (success/conflict/error per repo) + "Done" navigates back

**Files to create/modify:**
- `gui/public/js/views/branch-switch.js` *(new)*
- `gui/public/js/app.js` *(replace placeholder stub for `#/projects/:id/workspaces/:wid/branch-switch`)*

**Waiting on:** The Reviewer completing code-review + Documentation completing documentation on WP-016, which will unblock WP-017 from BLOCKED → READY.

No action needed from you right now — standby for WP-017 to become READY.

## Assistant

QA for WP-016 is **complete**. Here's a summary of what was done:

---

## WP-016 QA — PASS ✅

**61 automated tests written and executed** in `qa-wp016-tests.mjs`, covering all 6 acceptance criteria:

| Section | Tests | Result |
|---|---|---|
| Static analysis (imports, exports, structure) | 21 | ✅ All pass |
| **AC1** — Header + status table renders correctly | 7 | ✅ All pass |
| **AC2** — Branch name + color-coded status badge per row | 6 | ✅ All pass |
| **AC3** — In-place polling via `setInterval` | 3 | ✅ All pass |
| **AC4** — Cleanup clears polling interval on navigation | 4 | ✅ All pass |
| **AC5** — "Switch Branches" navigates to correct URL | 3 | ✅ All pass |
| **AC6** — STABLE guards disabled; non-STABLE actions functional | 11 | ✅ All pass |
| Edge cases | 4 | ✅ All pass |

**Regression:** 517/517 server-side tests still pass — no regressions.

**Minor observations** (low priority, none blocking):
1. `updateStatusTable()` only updates repos present at initial load — fine for current scope
2. No visual loading indicator during polling ticks (cosmetic)
3. Rename regex `/^[A-Z]{2,6}$/` duplicated in `workspace-detail.js` and `project-detail.js` (noted by Developer, suggest future refactor to shared util)

Handed off to the **Reviewer** (code-review pipeline is next for WP-016).

---

## Token Usage

| Metric | Count |
| ------ | ----- |
| Input Tokens | 1379881 |
| Output Tokens | 23154 |
| Total Tokens | 1403035 |

