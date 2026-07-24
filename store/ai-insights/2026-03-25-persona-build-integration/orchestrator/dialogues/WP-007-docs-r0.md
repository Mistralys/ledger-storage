# Dialogue — docs / WP-007

| Field | Value |
| ----- | ----- |
| Stage | `docs` |
| WP ID | `WP-007` |
| Captured | 2026-03-25T21:45:54.973415+00:00 |

## Human

**Project:** `/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration`
**Work package:** WP-007

Always use the project path above for all ledger tool calls.

CRITICAL: Every MCP tool call MUST use `work_package_id=WP-007`. Do NOT reference or operate on any other work package.


## Assistant

```json
{
  "id": "toolu_01R9vdXY48eFsG6UJzYNjCGT",
  "caller": {
    "type": "direct"
  },
  "input": {
    "agent_role": "Documentation",
    "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration"
  },
  "name": "ledger_get_next_action",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_next_action` (id: `toolu_01R9vdXY48eFsG6UJzYNjCGT`)

```json
{
  "agent_role": "Documentation",
  "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration"
}
```

## Tool Result

{
  "action": "WRITE_DOCS",
  "work_package_id": "WP-007",
  "reason": "Work package WP-007 has PASS release-engineering pipeline. Write or update documentation.",
  "next_steps": [
    "1. Call ledger_begin_work (work_package_id: \"WP-007\", type: \"documentation\", agent_role: \"Documentation\").",
    "2. Call ledger_get_work_package to review implementation artifacts and review comments.",
    "3. Update documentation, README files, and inline docs as needed.",
    "4. Call ledger_complete_pipeline (work_package_id: \"WP-007\", type: \"documentation\", status: PASS/FAIL, summary, artifacts, comments, acceptance_criteria_updates).",
    "5. Call ledger_get_handoff_status (current_agent: \"Documentation\")."
  ],
  "handoff_notes": [
    "Documentation agent: two acceptance criteria remain unmet and are your responsibility:\n\n1. README.md ledger plugin section (AC1 + AC2): The current README.md references docs/plugins.md in a documentation table but has no inline ledger plugin section. You need to add a ## Ledger Plugin section to README.md that includes: (a) a working usage code example matching the config in the plan (the personas/persona-build.config.js example from the plan.md Architectural Context section), and (b) documents all ledgerPlugin() options — manifestRoles (ReadonlyArray<string>, optional, defaults to [] which skips validation) and warnOnUnknownRole (boolean, default: true). The full option docs are already in docs/plugins.md §LedgerPluginOptions — adapt them for the README inline section.\n\n2. AGENTS.md 'how to add a new plugin' section (AC3): AGENTS.md has test/build commands and references file-tree.md for repo layout, but lacks an explicit contributor guide section covering: repo layout overview, test command (npm test), build command (npm run build), and how to add a new plugin (create src/plugins/your-plugin/index.ts implementing PersonaBuildPlugin, add to tsup.config.ts entry points for sub-path export, add to package.json exports field, write tests in tests/plugins/). See docs/plugins.md for the PersonaBuildPlugin interface reference."
  ]
}

## Assistant

```json
{
  "id": "toolu_01FbqARzCGVzPx1NpxZjsZ26",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-007",
    "type": "documentation",
    "agent_role": "Documentation",
    "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration"
  },
  "name": "ledger_begin_work",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01Y8WDpYaHHAdQuJtJSjxwyg",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-007",
    "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration"
  },
  "name": "ledger_get_work_package",
  "type": "tool_use"
}
```

**Tool call:** `ledger_begin_work` (id: `toolu_01FbqARzCGVzPx1NpxZjsZ26`)

```json
{
  "work_package_id": "WP-007",
  "type": "documentation",
  "agent_role": "Documentation",
  "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration"
}
```

**Tool call:** `ledger_get_work_package` (id: `toolu_01Y8WDpYaHHAdQuJtJSjxwyg`)

```json
{
  "work_package_id": "WP-007",
  "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration"
}
```

## Tool Result

{
  "work_package_id": "WP-007",
  "work_package_file": "work/WP-007.md",
  "status": "IN_PROGRESS",
  "assigned_to": "Documentation",
  "dependencies": [
    "WP-003"
  ],
  "acceptance_criteria": [
    {
      "criterion": "README.md contains a ledger plugin section with a working usage code example matching the config shown in the plan",
      "met": false
    },
    {
      "criterion": "README.md documents all ledgerPlugin() options (manifestRoles, warnOnUnknownRole) with types and descriptions",
      "met": false
    },
    {
      "criterion": "AGENTS.md exists at the repo root and covers: repo layout, test command, build command, and how to add a new plugin",
      "met": false
    },
    {
      "criterion": "npm pack --dry-run output includes the compiled ledger plugin files under the exports path and excludes *.test.* files and src/ TypeScript sources",
      "met": true
    },
    {
      "criterion": "A git tag v1.0.0 exists on the commit that represents the final state of the library after all WP-001–WP-003 work",
      "met": true
    }
  ],
  "active_pipeline_stages": [
    "release-engineering",
    "documentation"
  ],
  "revision": 0,
  "pipelines": [
    {
      "type": "release-engineering",
      "status": "PASS",
      "started_at": "2026-03-25T21:29:05Z",
      "completed_at": "2026-03-25T21:35:16Z",
      "duration_ms": 371000,
      "summary": [
        "Bumped version from 0.2.0 to 1.0.0 in package.json (first stable release).",
        "Updated version references in AGENTS.md, docs/agents/project-manifest/README.md, and docs/api.md.",
        "Curated CHANGELOG.md: promoted [Unreleased] content to [1.0.0] entry (2026-03-25), added factory/export and unit-test bullets, added Keep-a-Changelog link references block.",
        "Verified package manifest: 'files': ['dist'] covers dist/plugins/ledger/index.{js,cjs,d.ts,d.cts,.map} — ledger plugin exports are present; no *.test.* files or src/ TypeScript sources in dist/.",
        "Committed all pending changes (18 files, 2342 insertions) in commit ae93c2b: 'Release v1.0.0: ledger plugin, unit tests, publish prep'.",
        "Created annotated git tag v1.0.0 on commit ae93c2b.",
        "No migration guide required — v1.0.0 adds new capabilities (ledger plugin sub-path export) without removing or breaking any existing public API surface."
      ],
      "artifacts": {
        "files_modified": [
          "package.json",
          "CHANGELOG.md",
          "AGENTS.md",
          "docs/agents/project-manifest/README.md",
          "docs/api.md",
          "src/plugins/ledger/frontmatter-templates.ts",
          "src/plugins/ledger/index.ts",
          "src/plugins/ledger/mcp-tools-renderer.ts",
          "src/plugins/ledger/role-validator.ts",
          "src/plugins/ledger/roster-renderer.ts",
          "tests/plugins/ledger.test.ts"
        ],
        "commit_hash": "ae93c2b"
      },
      "comments": [
        {
          "type": "version-decision",
          "priority": "medium",
          "timestamp": "2026-03-25T21:35:16Z",
          "note": "Version bumped to 1.0.0 (not 0.3.0). Rationale: this is the first stable public release per the plan's explicit 'Tag version — v1.0.0 release' instruction. The ledger plugin sub-path export (@mistralys/persona-builder/plugins/ledger) completes the full public API surface described in the plan. No existing API was removed or changed, so there is no breaking change — 1.0.0 is a marketing/stability statement, not a semver breaking-change indicator."
        },
        {
          "type": "release-note",
          "priority": "low",
          "timestamp": "2026-03-25T21:35:16Z",
          "note": "Package manifest is clean for npm publish: 'files': ['dist'] in package.json means only dist/ ships. dist/plugins/ledger/ contains index.js, index.cjs, index.d.ts, index.d.cts, and source maps. No .test. files, no src/ TypeScript sources, no fixtures in the tarball. No .npmignore required."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-25T21:35:16Z",
          "note": "Node.js was not available in the sandbox shell (npm pack --dry-run could not be executed). Package manifest correctness was verified through direct file inspection: 'files' field, dist/ directory listing, and absence of test/source files in dist/. This should be re-verified manually before actual npm publish."
        }
      ]
    },
    {
      "type": "documentation",
      "status": "IN_PROGRESS",
      "started_at": "2026-03-25T21:46:01Z",
      "summary": []
    }
  ],
  "status_changed_at": "2026-03-25T21:29:05Z",
  "last_updated": "2026-03-25T21:46:01Z",
  "handoff_notes": [
    {
      "from_agent": "Release Engineer",
      "to_agent": "Documentation",
      "timestamp": "2026-03-25T21:35:16Z",
      "notes": [
        "Documentation agent: two acceptance criteria remain unmet and are your responsibility:\n\n1. README.md ledger plugin section (AC1 + AC2): The current README.md references docs/plugins.md in a documentation table but has no inline ledger plugin section. You need to add a ## Ledger Plugin section to README.md that includes: (a) a working usage code example matching the config in the plan (the personas/persona-build.config.js example from the plan.md Architectural Context section), and (b) documents all ledgerPlugin() options — manifestRoles (ReadonlyArray<string>, optional, defaults to [] which skips validation) and warnOnUnknownRole (boolean, default: true). The full option docs are already in docs/plugins.md §LedgerPluginOptions — adapt them for the README inline section.\n\n2. AGENTS.md 'how to add a new plugin' section (AC3): AGENTS.md has test/build commands and references file-tree.md for repo layout, but lacks an explicit contributor guide section covering: repo layout overview, test command (npm test), build command (npm run build), and how to add a new plugin (create src/plugins/your-plugin/index.ts implementing PersonaBuildPlugin, add to tsup.config.ts entry points for sub-path export, add to package.json exports field, write tests in tests/plugins/). See docs/plugins.md for the PersonaBuildPlugin interface reference."
      ]
    }
  ],
  "claimed": false
}

## Tool Result

{
  "work_package_id": "WP-007",
  "work_package_file": "work/WP-007.md",
  "status": "IN_PROGRESS",
  "assigned_to": "Release Engineer",
  "dependencies": [
    "WP-003"
  ],
  "acceptance_criteria": [
    {
      "criterion": "README.md contains a ledger plugin section with a working usage code example matching the config shown in the plan",
      "met": false
    },
    {
      "criterion": "README.md documents all ledgerPlugin() options (manifestRoles, warnOnUnknownRole) with types and descriptions",
      "met": false
    },
    {
      "criterion": "AGENTS.md exists at the repo root and covers: repo layout, test command, build command, and how to add a new plugin",
      "met": false
    },
    {
      "criterion": "npm pack --dry-run output includes the compiled ledger plugin files under the exports path and excludes *.test.* files and src/ TypeScript sources",
      "met": true
    },
    {
      "criterion": "A git tag v1.0.0 exists on the commit that represents the final state of the library after all WP-001–WP-003 work",
      "met": true
    }
  ],
  "active_pipeline_stages": [
    "release-engineering",
    "documentation"
  ],
  "revision": 0,
  "pipelines": [
    {
      "type": "release-engineering",
      "status": "PASS",
      "started_at": "2026-03-25T21:29:05Z",
      "completed_at": "2026-03-25T21:35:16Z",
      "duration_ms": 371000,
      "summary": [
        "Bumped version from 0.2.0 to 1.0.0 in package.json (first stable release).",
        "Updated version references in AGENTS.md, docs/agents/project-manifest/README.md, and docs/api.md.",
        "Curated CHANGELOG.md: promoted [Unreleased] content to [1.0.0] entry (2026-03-25), added factory/export and unit-test bullets, added Keep-a-Changelog link references block.",
        "Verified package manifest: 'files': ['dist'] covers dist/plugins/ledger/index.{js,cjs,d.ts,d.cts,.map} — ledger plugin exports are present; no *.test.* files or src/ TypeScript sources in dist/.",
        "Committed all pending changes (18 files, 2342 insertions) in commit ae93c2b: 'Release v1.0.0: ledger plugin, unit tests, publish prep'.",
        "Created annotated git tag v1.0.0 on commit ae93c2b.",
        "No migration guide required — v1.0.0 adds new capabilities (ledger plugin sub-path export) without removing or breaking any existing public API surface."
      ],
      "artifacts": {
        "files_modified": [
          "package.json",
          "CHANGELOG.md",
          "AGENTS.md",
          "docs/agents/project-manifest/README.md",
          "docs/api.md",
          "src/plugins/ledger/frontmatter-templates.ts",
          "src/plugins/ledger/index.ts",
          "src/plugins/ledger/mcp-tools-renderer.ts",
          "src/plugins/ledger/role-validator.ts",
          "src/plugins/ledger/roster-renderer.ts",
          "tests/plugins/ledger.test.ts"
        ],
        "commit_hash": "ae93c2b"
      },
      "comments": [
        {
          "type": "version-decision",
          "priority": "medium",
          "timestamp": "2026-03-25T21:35:16Z",
          "note": "Version bumped to 1.0.0 (not 0.3.0). Rationale: this is the first stable public release per the plan's explicit 'Tag version — v1.0.0 release' instruction. The ledger plugin sub-path export (@mistralys/persona-builder/plugins/ledger) completes the full public API surface described in the plan. No existing API was removed or changed, so there is no breaking change — 1.0.0 is a marketing/stability statement, not a semver breaking-change indicator."
        },
        {
          "type": "release-note",
          "priority": "low",
          "timestamp": "2026-03-25T21:35:16Z",
          "note": "Package manifest is clean for npm publish: 'files': ['dist'] in package.json means only dist/ ships. dist/plugins/ledger/ contains index.js, index.cjs, index.d.ts, index.d.cts, and source maps. No .test. files, no src/ TypeScript sources, no fixtures in the tarball. No .npmignore required."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-25T21:35:16Z",
          "note": "Node.js was not available in the sandbox shell (npm pack --dry-run could not be executed). Package manifest correctness was verified through direct file inspection: 'files' field, dist/ directory listing, and absence of test/source files in dist/. This should be re-verified manually before actual npm publish."
        }
      ]
    }
  ],
  "status_changed_at": "2026-03-25T21:29:05Z",
  "last_updated": "2026-03-25T21:35:16Z",
  "handoff_notes": [
    {
      "from_agent": "Release Engineer",
      "to_agent": "Documentation",
      "timestamp": "2026-03-25T21:35:16Z",
      "notes": [
        "Documentation agent: two acceptance criteria remain unmet and are your responsibility:\n\n1. README.md ledger plugin section (AC1 + AC2): The current README.md references docs/plugins.md in a documentation table but has no inline ledger plugin section. You need to add a ## Ledger Plugin section to README.md that includes: (a) a working usage code example matching the config in the plan (the personas/persona-build.config.js example from the plan.md Architectural Context section), and (b) documents all ledgerPlugin() options — manifestRoles (ReadonlyArray<string>, optional, defaults to [] which skips validation) and warnOnUnknownRole (boolean, default: true). The full option docs are already in docs/plugins.md §LedgerPluginOptions — adapt them for the README inline section.\n\n2. AGENTS.md 'how to add a new plugin' section (AC3): AGENTS.md has test/build commands and references file-tree.md for repo layout, but lacks an explicit contributor guide section covering: repo layout overview, test command (npm test), build command (npm run build), and how to add a new plugin (create src/plugins/your-plugin/index.ts implementing PersonaBuildPlugin, add to tsup.config.ts entry points for sub-path export, add to package.json exports field, write tests in tests/plugins/). See docs/plugins.md for the PersonaBuildPlugin interface reference."
      ]
    }
  ]
}

## Assistant

I have a clear picture. Let me read the relevant files now — plan.md for the config example, and the existing README.md, AGENTS.md, and docs/plugins.md.

```json
{
  "id": "toolu_01SrcTmqyJuXEUwwA9L3Upwg",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration/plan.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_011SjLLK9nHs9vMofqhePUDW",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/libraries/ai-insights-dev"
  },
  "name": "ls",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01SrcTmqyJuXEUwwA9L3Upwg`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration/plan.md"
}
```

**Tool call:** `ls` (id: `toolu_011SjLLK9nHs9vMofqhePUDW`)

```json
{
  "path": "/Users/smordziol/Webserver/libraries/ai-insights-dev"
}
```

## Tool Result

     1	# Plan: Persona Build — Ledger Plugin & ai-insights Migration
     2	
     3	> **Prerequisite:** `2026-03-25-persona-build-core-library/plan.md` (Plan 1) must be completed and committed first.
     4	> **Sequence:** Plan 2 of 2
     5	
     6	## Summary
     7	
     8	Build the ledger-specific plugin for `@mistralys/persona-builder`, migrate ai-insights' persona build system to use the library, verify byte-identical output across all 48 persona files, update project manifests and documentation, and prepare for npm publishing.
     9	
    10	## Architectural Context
    11	
    12	### Prerequisites from Plan 1
    13	
    14	At the start of this plan, the following must be true:
    15	
    16	- `ai-persona-builder-STABLE` contains a working library with: template engine, loaders, builder core, plugin architecture, CLI, and tests.
    17	- The library is symlinked into `ai-insights-dev` via `npm link` (or `"link:../ai-persona-builder-STABLE"` in `package.json`).
    18	- The library builds and tests pass independently.
    19	
    20	### Current ai-insights Build System (To Be Replaced)
    21	
    22	| Component | File | What Happens to It |
    23	|-----------|------|--------------------|
    24	| Build script | `scripts/build-personas.js` (~560 lines) | Rewritten to thin wrapper calling library API |
    25	| Helpers | `scripts/lib/persona-helpers.js` (~350 lines) | Deprecated — all generic functions moved to library |
    26	| Helper tests | `scripts/tests/persona-helpers.test.js` (~160 lines) | Deprecated — tests moved to library |
    27	| Sync script | `scripts/sync-personas.js` (~504 lines) | **Unchanged** — no coupling to helpers, calls build script as subprocess |
    28	
    29	### Ledger-Specific Code to Extract into Plugin
    30	
    31	| Function / Logic | Current Location | Plugin Hook |
    32	|------------------|------------------|-------------|
    33	| `renderRoster()` | `persona-helpers.js` lines ~240–280 | `onBuildContext` — adds `roster_rendered` to context |
    34	| `renderMcpToolsTable()` | `persona-helpers.js` lines ~290–340 | `onBuildContext` — adds `mcp_tools_table` to context |
    35	| Role validation against `_MANIFEST_ROLE_NAMES` | `build-personas.js` lines ~350–360 | `onValidate` — checks persona `role` against manifest |
    36	| `ccFrontmatterFields()` | `build-personas.js` lines ~50–55 | Absorbed into ledger frontmatter templates |
    37	| `FRONTMATTER_LEDGER_VSCODE` template | `build-personas.js` lines ~60–80 | `frontmatterTemplates.vscode` |
    38	| `FRONTMATTER_LEDGER_CC` template | `build-personas.js` lines ~85–110 | `frontmatterTemplates['claude-code']` |
    39	| `note_only` guard (check mode) | `build-personas.js` lines ~480–510 | `onValidate` — verify `note_only` fields aren't exposed |
    40	
    41	### Config File to Create
    42	
    43	```javascript
    44	// personas/persona-build.config.js
    45	const { ledgerPlugin } = require('@mistralys/persona-builder/plugins/ledger');
    46	const manifest = require('../shared/workflow-manifest.json');
    47	
    48	module.exports = {
    49	  rootDir: __dirname,
    50	  sharedPartialsDir: './shared/partials',
    51	  suites: {
    52	    ledger: {
    53	      srcDir: './ledger/src',
    54	      outVscode: './ledger/vs-code',
    55	      outClaudeCode: './ledger/claude-code',
    56	      personaMode: 'numbered',
    57	    },
    58	    standalone: {
    59	      srcDir: './standalone/src',
    60	      outVscode: './standalone/vs-code',
    61	      outClaudeCode: './standalone/claude-code',
    62	      personaMode: 'standalone',
    63	    },
    64	  },
    65	  plugins: [
    66	    ledgerPlugin({
    67	      manifestRoles: manifest.roles.map(r => r.name),
    68	      warnOnUnknownRole: true,
    69	    }),
    70	  ],
    71	};
    72	```
    73	
    74	### Files That Must Stay in ai-insights
    75	
    76	| Function | Why | Location |
    77	|----------|-----|----------|
    78	| `syncPersonasVersion()` | Reads `personas/changelog.md`, writes `personas/package.json` — project-specific CI glue | `scripts/build-personas.js` (retained) |
    79	| Role derivation + frontmatter validation | `sync-personas.js` has its own independent parsing — no coupling to helpers | `scripts/sync-personas.js` (unchanged) |
    80	| `_MANIFEST_ROLE_NAMES` | Used by ledger plugin, but sourced from `shared/workflow-manifest.json` — project config, not library code | Passed to plugin via config |
    81	
    82	---
    83	
    84	## Approach / Architecture
    85	
    86	### Ledger Plugin Structure
    87	
    88	The ledger plugin ships as a sub-path export from `@mistralys/persona-builder`:
    89	
    90	```
    91	ai-persona-builder-STABLE/
    92	├── src/
    93	│   └── plugins/
    94	│       └── ledger/
    95	│           ├── index.ts              # Plugin factory: ledgerPlugin(options)
    96	│           ├── roster-renderer.ts    # renderRoster() — ported from persona-helpers.js
    97	│           ├── mcp-tools-renderer.ts # renderMcpToolsTable() — ported from persona-helpers.js
    98	│           └── role-validator.ts     # Role validation + note_only guard
    99	```
   100	

## Tool Result

['/Users/smordziol/Webserver/libraries/ai-insights-dev/.claude/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/.context/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/.git/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/.gitattributes', '/Users/smordziol/Webserver/libraries/ai-insights-dev/.githooks/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/.github/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/.gitignore', '/Users/smordziol/Webserver/libraries/ai-insights-dev/.mcp.dist.json', '/Users/smordziol/Webserver/libraries/ai-insights-dev/.mcp.json', '/Users/smordziol/Webserver/libraries/ai-insights-dev/.ruff_cache/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/.vscode/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/AGENTS.md', '/Users/smordziol/Webserver/libraries/ai-insights-dev/CLAUDE.md', '/Users/smordziol/Webserver/libraries/ai-insights-dev/README.md', '/Users/smordziol/Webserver/libraries/ai-insights-dev/changelog.md', '/Users/smordziol/Webserver/libraries/ai-insights-dev/context.yaml', '/Users/smordziol/Webserver/libraries/ai-insights-dev/discussions/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/history/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/mcp-server/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/node_modules/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/orchestrator/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/package-lock.json', '/Users/smordziol/Webserver/libraries/ai-insights-dev/package.json', '/Users/smordziol/Webserver/libraries/ai-insights-dev/personas/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/scripts/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/shared/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/vitest.config.ts']

## Assistant

This is the `ai-insights-dev` repo. The library is at a different path. Let me find where `ai-persona-builder-STABLE` is.

```json
{
  "id": "toolu_01UE7oN4ieMLz2y9a1zhM7L7",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/libraries"
  },
  "name": "ls",
  "type": "tool_use"
}
```

**Tool call:** `ls` (id: `toolu_01UE7oN4ieMLz2y9a1zhM7L7`)

```json
{
  "path": "/Users/smordziol/Webserver/libraries"
}
```

## Tool Result

['/Users/smordziol/Webserver/libraries/.DS_Store', '/Users/smordziol/Webserver/libraries/HTML_QuickForm2/', '/Users/smordziol/Webserver/libraries/ai-insights-dev/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/', '/Users/smordziol/Webserver/libraries/application-framework-DEV-Bifrost/', '/Users/smordziol/Webserver/libraries/application-framework-DEV-Fenrir/', '/Users/smordziol/Webserver/libraries/application-framework-DEV-Odin/', '/Users/smordziol/Webserver/libraries/application-framework-STABLE/', '/Users/smordziol/Webserver/libraries/application-framework-docs/', '/Users/smordziol/Webserver/libraries/application-localization/', '/Users/smordziol/Webserver/libraries/application-utils-collections/', '/Users/smordziol/Webserver/libraries/application-utils-core/', '/Users/smordziol/Webserver/libraries/application-utils-image/', '/Users/smordziol/Webserver/libraries/application-utils-result-handling/', '/Users/smordziol/Webserver/libraries/application-utils.wiki/', '/Users/smordziol/Webserver/libraries/application-utils/', '/Users/smordziol/Webserver/libraries/builder-content-generator/', '/Users/smordziol/Webserver/libraries/changelog-parser/', '/Users/smordziol/Webserver/libraries/column-widths-calculator/', '/Users/smordziol/Webserver/libraries/composer-local-switcher/', '/Users/smordziol/Webserver/libraries/currency-parser/', '/Users/smordziol/Webserver/libraries/icon-svg-generator/', '/Users/smordziol/Webserver/libraries/mail-body-extractor/', '/Users/smordziol/Webserver/libraries/mailcode/', '/Users/smordziol/Webserver/libraries/maileditor-serializers-DEV-Bifrost/', '/Users/smordziol/Webserver/libraries/maileditor-serializers-DEV-Fenrir/', '/Users/smordziol/Webserver/libraries/maileditor-serializers-DEV-Odin/', '/Users/smordziol/Webserver/libraries/maileditor-serializers-STABLE/', '/Users/smordziol/Webserver/libraries/mailings-icon-library/', '/Users/smordziol/Webserver/libraries/markdown-viewer/', '/Users/smordziol/Webserver/libraries/nexus-connector-pigeon/', '/Users/smordziol/Webserver/libraries/puki-vpn/', '/Users/smordziol/Webserver/libraries/rygnarok/', '/Users/smordziol/Webserver/libraries/simple_html_dom/', '/Users/smordziol/Webserver/libraries/text-diff/', '/Users/smordziol/Webserver/libraries/time-track-helper/', '/Users/smordziol/Webserver/libraries/tiny-markdown-editor/', '/Users/smordziol/Webserver/libraries/version-parser/']

## Assistant

```json
{
  "id": "toolu_018ceQ6ZNici8tCvVRYE1tnJ",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/README.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01FN7b35mLjBLtxVaBxxhSVR",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/AGENTS.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01FFeP3TM7dDE8FaqFv6x791",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_018ceQ6ZNici8tCvVRYE1tnJ`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/README.md"
}
```

**Tool call:** `read_file` (id: `toolu_01FN7b35mLjBLtxVaBxxhSVR`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/AGENTS.md"
}
```

**Tool call:** `read_file` (id: `toolu_01FFeP3TM7dDE8FaqFv6x791`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md"
}
```

## Tool Result

     1	# AI Persona Builder
     2	
     3	Build AI persona instruction files for **VS Code Chat** and **Claude Code** from YAML metadata and Markdown templates — with zero configuration friction.
     4	
     5	Define your personas once as simple YAML + Markdown sources, and the library generates correctly formatted instruction files for both IDEs. A plugin system lets you inject custom frontmatter, run validators, or post-process output without touching the core engine.
     6	
     7	## ✨ Features
     8	
     9	- **Dual-target output** — generates both `.agent.md` (VS Code) and `.md` (Claude Code) from a single source
    10	- **YAML + Markdown templating** — separate metadata from content; merge them at build time with `{{variables}}`, `{{> partials}}`, and `{{#if}}` conditionals
    11	- **Shared + per-suite partials** — reuse content fragments across personas with local overrides
    12	- **Plugin architecture** — hook into context building, post-rendering, validation, and frontmatter generation
    13	- **CI-friendly** — `--check` mode renders without writing; `--strict` exits non-zero on warnings
    14	- **Programmatic & CLI** — use the `build()` API in scripts or run `persona-build` from the command line
    15	- **Single dependency** — only `js-yaml` at runtime
    16	
    17	## 📋 Requirements
    18	
    19	- **Node.js** ≥ 18
    20	
    21	## 🚀 Quick Start
    22	
    23	```bash
    24	npm install @mistralys/persona-builder
    25	```
    26	
    27	### Programmatic API
    28	
    29	```ts
    30	import { build } from '@mistralys/persona-builder';
    31	import path from 'node:path';
    32	
    33	const summary = await build({
    34	  suites: {
    35	    'my-suite': {
    36	      srcDir: path.resolve('./personas/my-suite'),
    37	      outVscode: path.resolve('./dist/vscode'),
    38	      outClaudeCode: path.resolve('./dist/claude-code'),
    39	    },
    40	  },
    41	  sharedPartialsDir: path.resolve('./personas/shared/partials'),
    42	});
    43	
    44	console.log(`Built ${summary.totalBuilt} persona(s), wrote ${summary.totalWritten} file(s).`);
    45	```
    46	
    47	### CLI
    48	
    49	```bash
    50	# Create a persona-build.config.js, then:
    51	npx persona-build
    52	
    53	# CI staleness check
    54	npx persona-build --check --strict
    55	```
    56	
    57	See the [CLI docs](docs/cli.md) for config file format and all flags.
    58	
    59	## 📖 Documentation
    60	
    61	| Guide | Description |
    62	|-------|-------------|
    63	| [Directory Convention](docs/directory-convention.md) | Expected source layout (`meta/`, `content/`, `partials/`) |
    64	| [Template Syntax](docs/template-syntax.md) | Variables, partials, conditionals, and built-in context variables |
    65	| [Configuration Reference](docs/configuration.md) | `BuildConfig`, `SuiteConfig`, and `BuildSummary` fields |
    66	| [CLI Reference](docs/cli.md) | Command-line flags, config file format, and common patterns |
    67	| [Plugins](docs/plugins.md) | `PersonaBuildPlugin` interface, examples, and the built-in Ledger Plugin |
    68	| [Public API](docs/api.md) | All exported types and functions |
    69	
    70	## 📄 License
    71	
    72	MIT

## Tool Result

     1	# AI Agents Operating System — @mistralys/persona-builder
     2	
     3	> **Purpose:** Authoritative entry point for AI agents entering the `@mistralys/persona-builder` codebase. Defines how agents discover, navigate, and interact with the library to ensure architectural integrity and token efficiency.
     4	
     5	---
     6	
     7	## 📚 Project Manifest — Start Here!
     8	
     9	**Core Philosophy:** The Project Manifest is the canonical documentation of this codebase. If implementation code contradicts the manifest, the **code is likely wrong**.
    10	
    11	**Manifest location:** [`docs/agents/project-manifest/`](docs/agents/project-manifest/README.md)
    12	
    13	| Document | Contents |
    14	|----------|----------|
    15	| [README.md](docs/agents/project-manifest/README.md) | Project overview, version, and manifest index. |
    16	| [tech-stack.md](docs/agents/project-manifest/tech-stack.md) | Runtime, frameworks, architectural patterns, build tooling, distribution format. |
    17	| [file-tree.md](docs/agents/project-manifest/file-tree.md) | Annotated directory structure (22 source files, 14 test files, fixtures). |
    18	| [api-surface.md](docs/agents/project-manifest/api-surface.md) | All exported types, functions, and constants — signatures only. |
    19	| [data-flows.md](docs/agents/project-manifest/data-flows.md) | Build pipeline, context merge order, plugin hooks, CLI flow. |
    20	| [constraints.md](docs/agents/project-manifest/constraints.md) | Architectural invariants, naming rules, known limitations. |
    21	
    22	### Quick Start Workflow
    23	
    24	Follow this sequence before making any changes:
    25	
    26	1. **Read [README.md](docs/agents/project-manifest/README.md)** — Understand project purpose and scope.
    27	2. **Read [tech-stack.md](docs/agents/project-manifest/tech-stack.md)** — Understand layered architecture, zero-dependency engine, plugin patterns.
    28	3. **Read [constraints.md](docs/agents/project-manifest/constraints.md)** — **MANDATORY** before writing any code.
    29	4. **Consult [file-tree.md](docs/agents/project-manifest/file-tree.md) + [api-surface.md](docs/agents/project-manifest/api-surface.md)** — Find files and public interfaces.
    30	5. **Read source code** — Only when implementation details are needed.
    31	
    32	---
    33	
    34	## 📝 Manifest Maintenance Rules
    35	
    36	When you change the codebase, update the corresponding manifest documents:
    37	
    38	| Change Made | Documents to Update |
    39	|-------------|---------------------|
    40	| Add/modify engine function | `api-surface.md`, verify zero-dependency invariant in `constraints.md` |
    41	| Add/modify loader function | `api-surface.md`, `file-tree.md` (if new file) |
    42	| Add/modify builder function | `api-surface.md`, `data-flows.md` (if pipeline changes) |
    43	| Add/modify plugin hook | `api-surface.md`, `data-flows.md` (hook execution order) |
    44	| Add/modify validator | `api-surface.md`, `file-tree.md` (if new file) |
    45	| Add/modify exported type | `api-surface.md` |
    46	| Add/remove dependency | `tech-stack.md` |
    47	| Add new file or directory | `file-tree.md` |
    48	| Change build pipeline flow | `data-flows.md` |
    49	| Change frontmatter defaults | `api-surface.md`, `data-flows.md` (precedence section) |
    50	| Change naming convention | `constraints.md` |
    51	| Add CLI flag | `api-surface.md` (CLI section in README.md too) |
    52	| Change architectural pattern | `tech-stack.md`, `constraints.md` |
    53	| Discover new limitation | `constraints.md` (Known Limitations section) |
    54	
    55	---
    56	
    57	## ⚡ Efficiency Rules — Search Smart, Read Less
    58	
    59	**Token efficiency is critical. Follow this search hierarchy:**
    60	
    61	| What You Need | Search Here FIRST | Then Here | Read Source LAST |
    62	|---------------|-------------------|-----------|------------------|
    63	| Find a file location | `file-tree.md` | `grep` / file search | Never needed |
    64	| Understand a function/type | `api-surface.md` | Source code | Only for implementation logic |
    65	| Trace data flow | `data-flows.md` | Source code | Only for edge cases |
    66	| Check a rule or convention | `constraints.md` | Source comments | Only if ambiguous |
    67	| Identify dependencies | `tech-stack.md` | `package.json` | Never needed |
    68	| Understand patterns | `tech-stack.md` | Source code | Only for complex logic |
    69	
    70	### Anti-Patterns
    71	
    72	| ❌ Inefficient | ✅ Efficient |
    73	|---------------|-------------|
    74	| Grep the entire codebase for a type name | Search `api-surface.md` |
    75	| Read 5 engine files to understand template rendering | Read `data-flows.md` §7 (Render body) |
    76	| Read source to check if a function is pure | Check `constraints.md` §1 (Zero-Dependency Engine) |
    77	| Read `package.json` for dependencies | Check `tech-stack.md` |
    78	| Read all builder files to understand plugin order | Read `data-flows.md` §5 (Plugin Hook Execution) |
    79	
    80	---
    81	
    82	## 🚨 Failure Protocol & Decision Matrix
    83	
    84	| Scenario | Action | Priority |
    85	|----------|--------|----------|
    86	| **Manifest vs. code conflict** | Trust manifest. Flag code for correction. | MUST |
    87	| **Ambiguous requirement** | Use most restrictive interpretation. Document assumption. | MUST |
    88	| **Missing manifest documentation** | Flag gap. Do not invent facts. Draft entry for review. | MUST |
    89	| **Untested code path** | Proceed with caution. Add test recommendation. | SHOULD |
    90	| **New engine function needs an import** | It does NOT belong in `src/engine/`. Move it to `src/loaders/` or `src/builders/`. | MUST |
    91	| **Plugin hook needs async** | Do NOT add `async` to existing runner functions without a plan. Flag for discussion. | MUST |
    92	| **Adding a new npm dependency** | Justify in writing. Update `tech-stack.md`. Never add to `src/engine/`. | MUST |
    93	| **Output file naming mismatch** | Check `vs_file_name` / `cc_file_name` context fields first, then fall back to content basename. See `data-flows.md` §7. | SHOULD |
    94	| **Template rendering produces wrong output** | Verify processing order: partials → conditionals → variables. This order is mandatory. | MUST |
    95	| **CI validation writes partial files** | Always combine `strict: true` with `check: true` in validation mode. | MUST |
    96	| **Path traversal concern** | Acceptable for build-time use with developer-controlled paths. Add a containment guard before any HTTP/CLI exposure. | SHOULD |
    97	| **Breaking change proposed** | Document before implementing. Flag for review. Never implement silently. | MUST |
    98	
    99	### Escalation Path
   100	

## Tool Result

     1	# Plugins
     2	
     3	Plugins extend the build pipeline without modifying the core engine. Register plugins via `BuildConfig.plugins`.
     4	
     5	## PersonaBuildPlugin Interface
     6	
     7	```ts
     8	interface PersonaBuildPlugin {
     9	  /** Unique name used for logging and identification */
    10	  name: string;
    11	
    12	  /**
    13	   * Called once per suite before any persona is built.
    14	   * Use this to read shared metadata and set up plugin state.
    15	   */
    16	  onSuiteInit?(suite: SuiteConfig, sharedMeta: Record<string, unknown>): void;
    17	
    18	  /**
    19	   * Called for each persona before template rendering.
    20	   * Mutate and return the context to inject additional template variables.
    21	   */
    22	  onBuildContext?(
    23	    context: Record<string, unknown>,
    24	    persona: PersonaMetadata,
    25	    suite: SuiteConfig,
    26	  ): Record<string, unknown>;
    27	
    28	  /**
    29	   * Called after the full output is rendered.
    30	   * Mutate and return the output string (e.g. to append a footer).
    31	   */
    32	  onPostRender?(output: string, persona: PersonaMetadata, target: TargetType): string;
    33	
    34	  /**
    35	   * Called during the validation phase.
    36	   * Return an array of ValidationResult entries (empty = no issues).
    37	   */
    38	  onValidate?(persona: PersonaMetadata, suite: SuiteConfig): ValidationResult[];
    39	
    40	  /**
    41	   * Register custom frontmatter templates, keyed by target type.
    42	   * These override the library defaults and config-level overrides.
    43	   */
    44	  frontmatterTemplates?: Partial<Record<TargetType, string>>;
    45	}
    46	```
    47	
    48	## Examples
    49	
    50	### Adding a custom frontmatter field
    51	
    52	```ts
    53	import { build, type PersonaBuildPlugin } from '@mistralys/persona-builder';
    54	
    55	const timestampPlugin: PersonaBuildPlugin = {
    56	  name: 'timestamp',
    57	
    58	  onBuildContext(context) {
    59	    // Inject a build-time variable that templates can use as {{build_date}}
    60	    return { ...context, build_date: new Date().toISOString().slice(0, 10) };
    61	  },
    62	};
    63	
    64	const summary = await build({
    65	  suites: {
    66	    docs: {
    67	      srcDir: './personas/docs',
    68	      outVscode: './dist/vscode',
    69	      outClaudeCode: './dist/cc',
    70	    },
    71	  },
    72	  plugins: [timestampPlugin],
    73	});
    74	```
    75	
    76	### Custom frontmatter template via plugin
    77	
    78	```ts
    79	const ledgerPlugin: PersonaBuildPlugin = {
    80	  name: 'ledger',
    81	
    82	  frontmatterTemplates: {
    83	    'claude-code': `---
    84	name: {{cc_file_name_stem}}
    85	description: '{{description}}'
    86	permissionMode: {{cc_permission_mode}}
    87	model: {{cc_model}}
    88	memory: {{cc_memory}}
    89	mcpServers:
    90	  - central_pm
    91	---`,
    92	  },
    93	};
    94	```
    95	
    96	### Validation plugin
    97	
    98	```ts
    99	import type { PersonaBuildPlugin, ValidationResult } from '@mistralys/persona-builder';
   100	

## Assistant

```json
{
  "id": "toolu_019WNJbek82eLRWDdjkNZ41u",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
    "offset": 100,
    "limit": 150
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01CLvTwfrcD1NAXt3uQi9dGy",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/AGENTS.md",
    "offset": 100,
    "limit": 100
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_019WNJbek82eLRWDdjkNZ41u`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
  "offset": 100,
  "limit": 150
}
```

**Tool call:** `read_file` (id: `toolu_01CLvTwfrcD1NAXt3uQi9dGy`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/AGENTS.md",
  "offset": 100,
  "limit": 100
}
```

## Tool Result

   101	const requiredFieldsPlugin: PersonaBuildPlugin = {
   102	  name: 'required-fields',
   103	
   104	  onValidate(persona): ValidationResult[] {
   105	    const errors: ValidationResult[] = [];
   106	    if (!persona.description) {
   107	      errors.push({ severity: 'error', message: `${persona.name}: missing "description" field` });
   108	    }
   109	    if (!persona.version) {
   110	      errors.push({ severity: 'warning', message: `${persona.name}: missing "version" field` });
   111	    }
   112	    return errors;
   113	  },
   114	};
   115	```
   116	
   117	---
   118	
   119	## Ledger Plugin — `@mistralys/persona-builder/plugins/ledger`
   120	
   121	The ledger plugin is a first-party plugin shipped as a **sub-path export** from the library. It wires ledger-specific rendering and validation into the standard build hooks and is the recommended way to build the ai-insights ledger persona suite.
   122	
   123	### Import
   124	
   125	```ts
   126	// ESM
   127	import { ledgerPlugin } from '@mistralys/persona-builder/plugins/ledger';
   128	
   129	// CJS
   130	const { ledgerPlugin } = require('@mistralys/persona-builder/plugins/ledger');
   131	```
   132	
   133	The sub-path resolves to `dist/plugins/ledger/index.{js,cjs,d.ts}` — all three artefacts (ESM, CJS, DTS) are included in the package.
   134	
   135	---
   136	
   137	### `ledgerPlugin(options?)`
   138	
   139	```ts
   140	function ledgerPlugin(options?: LedgerPluginOptions): PersonaBuildPlugin
   141	```
   142	
   143	Factory function. Returns a fully configured `PersonaBuildPlugin` for the ledger persona suite. The returned plugin object implements:
   144	
   145	| Hook / field | Behaviour |
   146	|---|---|
   147	| `name` | `'ledger'` |
   148	| `onBuildContext` | Injects `roster_rendered` and `mcp_tools_table` into the template context |
   149	| `onPostRender` | Caches the rendered output per-persona for use by `onValidate` |
   150	| `onValidate` | Runs role validation and the `note_only` guard; returns `ValidationResult[]` |
   151	| `frontmatterTemplates` | Registers `FRONTMATTER_LEDGER_VSCODE` (`vscode`) and `FRONTMATTER_LEDGER_CC` (`claude-code`) |
   152	
   153	#### Context variables injected by `onBuildContext`
   154	
   155	| Variable | Type | Value when absent |
   156	|---|---|---|
   157	| `roster_rendered` | `string` | `''` (empty — persona has no `roster` or `number` field) |
   158	| `mcp_tools_table` | `string` | `''` (empty — persona has no `mcp_tools` field) |
   159	
   160	Both keys are **always** set (even to an empty string) so templates can reference `{{roster_rendered}}` and `{{mcp_tools_table}}` on non-ledger personas without triggering an unresolved-variable warning.
   161	
   162	#### Usage
   163	
   164	```js
   165	// personas/persona-build.config.js
   166	const { ledgerPlugin } = require('@mistralys/persona-builder/plugins/ledger');
   167	const manifest = require('../shared/workflow-manifest.json');
   168	
   169	module.exports = {
   170	  rootDir: __dirname,
   171	  sharedPartialsDir: './shared/partials',
   172	  suites: {
   173	    ledger: {
   174	      srcDir: './ledger/src',
   175	      outVscode: './ledger/vs-code',
   176	      outClaudeCode: './ledger/claude-code',
   177	      personaMode: 'numbered',
   178	    },
   179	  },
   180	  plugins: [
   181	    ledgerPlugin({
   182	      manifestRoles: manifest.roles.map(r => r.name),
   183	      warnOnUnknownRole: true,
   184	    }),
   185	  ],
   186	};
   187	```
   188	
   189	---
   190	
   191	### `LedgerPluginOptions`
   192	
   193	```ts
   194	interface LedgerPluginOptions {
   195	  /**
   196	   * List of canonical role names from the project's workflow manifest.
   197	   * Every persona's `role` field is validated against this list.
   198	   * When omitted or empty, role validation is skipped.
   199	   */
   200	  manifestRoles?: ReadonlyArray<string>;
   201	
   202	  /**
   203	   * When `true`, an unknown `role` emits a warning-level ValidationResult.
   204	   * @default true
   205	   */
   206	  warnOnUnknownRole?: boolean;
   207	}
   208	```
   209	
   210	> **Known limitation — `warnOnUnknownRole` is not yet wired.** The option is accepted and preserved in the public interface, but the underlying `validateRole` always emits a warning when a role is not in the manifest, regardless of this flag. Setting `warnOnUnknownRole: false` currently has no observable effect. This will be resolved before the 1.0 release. In the meantime, role warnings cannot be suppressed via this option.
   211	
   212	---
   213	
   214	### Exported types
   215	
   216	#### `RosterEntry`
   217	
   218	```ts
   219	interface RosterEntry {
   220	  /** Sequential agent number (1-based) */
   221	  number: number;
   222	  /** Full display title for this agent role */
   223	  title: string;
   224	  /** Short description / label shown in parentheses */
   225	  short: string;
   226	}
   227	```
   228	
   229	Represents one entry in the ledger agent roster as declared in `_shared.yaml`.
   230	
   231	#### `McpToolEntry`
   232	
   233	```ts
   234	interface McpToolEntry {
   235	  /** Tool identifier as used in the MCP server */
   236	  tool: string;
   237	  /** Human-readable description of what the tool does */
   238	  purpose: string;
   239	  /**
   240	   * When true, this entry appears in documentation notes only and must NOT
   241	   * appear in the rendered persona output.
   242	   */
   243	  note_only?: boolean;
   244	}
   245	```
   246	
   247	Represents one MCP tool entry from the persona YAML `mcp_tools` field.
   248	
   249	---
   250	

## Tool Result

   101	```
   102	Issue Detected
   103	    ↓
   104	Can I resolve with manifest + constraints?
   105	    ↓ YES → Proceed
   106	    ↓ NO  →
   107	Is it an architectural concern (engine purity, async runner, new dependency)?
   108	    ↓ YES → Pause and request user input
   109	    ↓ NO  →
   110	Is it a breaking change to the public API?
   111	    ↓ YES → Pause and request user input
   112	    ↓ NO  →
   113	Is it a missing manifest entry?
   114	    ↓ YES → Draft entry + request review
   115	    ↓ NO  →
   116	Unclear → Pause and request user clarification
   117	```
   118	
   119	---
   120	
   121	## 📊 Project Stats
   122	
   123	| Property | Value |
   124	|----------|-------|
   125	| **Package** | `@mistralys/persona-builder` |
   126	| **Version** | 1.0.0 |
   127	| **Language** | TypeScript 5.8.2 (ES2022) |
   128	| **Runtime** | Node.js ≥ 18.17 (ESM) |
   129	| **Architecture** | Layered: builders → plugins → engine / loaders / validators |
   130	| **Package Manager** | npm |
   131	| **Build Tool** | tsup (dual CJS + ESM) |
   132	| **Test Framework** | Vitest (227 tests across 14 files) |
   133	| **Production Dependency** | `js-yaml` (sole) |
   134	| **License** | MIT |
   135	
   136	### npm Scripts
   137	
   138	| Script | Purpose |
   139	|--------|---------|
   140	| `npm run build` | Production build via tsup. |
   141	| `npm run dev` | Watch-mode build. |
   142	| `npm test` | Run all tests once. |
   143	| `npm run test:watch` | Run tests in watch mode. |
   144	| `npm run typecheck` | Type-check without emitting (`tsc --noEmit`). |
   145	
   146	---
   147	
   148	## 🧭 Navigation Quick Reference
   149	
   150	| I Need To… | Go Here |
   151	|------------|---------|
   152	| Understand the project | [README.md](README.md) |
   153	| See the full manifest | [docs/agents/project-manifest/](docs/agents/project-manifest/README.md) |
   154	| Find a source file | [file-tree.md](docs/agents/project-manifest/file-tree.md) |
   155	| Look up a function signature | [api-surface.md](docs/agents/project-manifest/api-surface.md) |
   156	| Understand the build pipeline | [data-flows.md](docs/agents/project-manifest/data-flows.md) |
   157	| Check naming rules or invariants | [constraints.md](docs/agents/project-manifest/constraints.md) |
   158	| Understand the tech stack | [tech-stack.md](docs/agents/project-manifest/tech-stack.md) |
   159	| See test fixtures | `fixtures/sample-suite/` and `fixtures/shared/` |
   160	| Run tests | `npm test` |
   161	| Build the library | `npm run build` |

## Assistant

```json
{
  "id": "toolu_01StnXmy4sxeNSbQ59qbJYUZ",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/AGENTS.md",
    "offset": 160,
    "limit": 50
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01StnXmy4sxeNSbQ59qbJYUZ`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/AGENTS.md",
  "offset": 160,
  "limit": 50
}
```

## Tool Result

   161	| Build the library | `npm run build` |

## Assistant

Good — that's the end of AGENTS.md. Now I have everything I need. Let me also check the file-tree.md for the repo layout section to reference accurately.

```json
{
  "id": "toolu_01HTkViohvCyTs5YcFKsYvkE",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01HTkViohvCyTs5YcFKsYvkE`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md"
}
```

## Tool Result

     1	# File Tree
     2	
     3	```
     4	@mistralys/persona-builder/
     5	├── package.json                    # npm package config (dual CJS + ESM exports, bin entry)
     6	├── tsconfig.json                   # TypeScript config (strict, ES2022, bundler resolution)
     7	├── tsup.config.ts                  # Build config (dual format, three entry points: index, cli, plugins/ledger/index)
     8	├── vitest.config.ts                # Test runner config
     9	├── CHANGELOG.md                    # Version history
    10	├── README.md                       # User-facing documentation
    11	├── LICENSE                         # MIT license
    12	│
    13	├── src/
    14	│   ├── index.ts                    # Public API barrel export + VERSION constant
    15	│   ├── cli.ts                      # CLI entry point (persona-build executable)
    16	│   │
    17	│   ├── engine/                     # Pure template rendering functions (zero dependencies)
    18	│   │   ├── index.ts                # Barrel re-export
    19	│   │   ├── partials.ts             # {{> name}} resolution (depth-2 recursion)
    20	│   │   ├── conditionals.ts         # {{#if flag}}…{{/if}} resolution
    21	│   │   ├── variables.ts            # {{varName}} substitution
    22	│   │   ├── postProcessor.ts        # Blank-line collapsing, heading spacing, newline normalization
    23	│   │   └── serializer.ts           # Tool list serialization (YAML flow format)
    24	│   │
    25	│   ├── loaders/                    # File I/O layer
    26	│   │   ├── index.ts                # Barrel re-export
    27	│   │   ├── partials-loader.ts      # Load .md files from a directory as a partials map
    28	│   │   ├── metadata-loader.ts      # YAML discovery + parsing into PersonaMetadata
    29	│   │   └── content-loader.ts       # Read raw Markdown content templates
    30	│   │
    31	│   ├── plugins/                    # Plugin system
    32	│   │   ├── index.ts                # Barrel re-export (types + runner functions)
    33	│   │   ├── types.ts                # Core types: TargetType, PersonaMetadata, SuiteConfig, etc.
    34	│   │   ├── runner.ts               # Hook invocation: runSuiteInit, runBuildContext, etc.
    35	│   │   └── ledger/                 # Ledger plugin (sub-path: @mistralys/persona-builder/plugins/ledger)
    36	│   │       ├── index.ts            # ledgerPlugin(options) factory — LedgerPluginOptions type
    37	│   │       ├── roster-renderer.ts  # renderRoster() — numbered Markdown list with (YOU) marker
    38	│   │       ├── mcp-tools-renderer.ts # renderMcpToolsTable() — filters note_only entries
    39	│   │       ├── role-validator.ts   # validateRole() + validateNoteOnlyGuard()
    40	│   │       └── frontmatter-templates.ts # FRONTMATTER_LEDGER_VSCODE, FRONTMATTER_LEDGER_CC
    41	│   │
    42	│   ├── builders/                   # Build orchestration
    43	│   │   ├── index.ts                # Barrel re-export
    44	│   │   ├── types.ts                # BuildConfig, BuildResult, BuildSummary types
    45	│   │   ├── frontmatter.ts          # Default frontmatter templates + resolution + rendering
    46	│   │   └── persona-builder.ts      # build(), buildSuite(), buildPersona() orchestrators
    47	│   │
    48	│   └── validators/                 # Validation functions
    49	│       ├── index.ts                # Barrel re-export
    50	│       ├── filename-validator.ts   # Kebab-case filename validation
    51	│       └── strict-validator.ts     # Required-marker presence validation
    52	│
    53	├── tests/
    54	│   ├── README.md                   # Test suite documentation
    55	│   ├── engine/                     # Engine module tests (74 tests)
    56	│   │   ├── partials.test.ts
    57	│   │   ├── conditionals.test.ts
    58	│   │   ├── variables.test.ts
    59	│   │   ├── postProcessor.test.ts
    60	│   │   └── serializer.test.ts
    61	│   ├── loaders/                    # Loader tests (40 tests)
    62	│   │   ├── partials-loader.test.ts
    63	│   │   ├── metadata-loader.test.ts
    64	│   │   └── content-loader.test.ts
    65	│   ├── plugins/                    # Plugin system tests (27 tests)
    66	│   │   └── plugin-runner.test.ts
    67	│   ├── builders/                   # Builder tests (33 tests)
    68	│   │   ├── persona-builder.test.ts
    69	│   │   └── persona-builder-edge-cases.test.ts
    70	│   ├── validators/                 # Validator tests (46 tests)
    71	│   │   ├── filename-validator.test.ts
    72	│   │   └── strict-validator.test.ts
    73	│   └── integration/                # End-to-end integration tests (7 tests)
    74	│       └── build.test.ts
    75	│
    76	├── fixtures/                       # Test fixtures
    77	│   ├── sample-suite/
    78	│   │   ├── meta/
    79	│   │   │   ├── _shared.yaml        # Suite-level shared defaults
    80	│   │   │   └── example-persona.yaml
    81	│   │   ├── content/
    82	│   │   │   └── example-persona.md  # Markdown content template
    83	│   │   └── partials/
    84	│   │       └── suite-specific.md   # Suite-local partial
    85	│   └── shared/
    86	│       └── partials/
    87	│           └── greeting.md         # Cross-suite shared partial
    88	│
    89	├── dist/                           # Build output (gitignored)
    90	└── docs/
    91	    └── agents/
    92	        └── project-manifest/       # This manifest
    93	```

## Assistant

I now have all the information I need. Let me make both updates simultaneously:

1. Add a `## Ledger Plugin` section to `README.md`
2. Add a contributor guide section to `AGENTS.md`

```json
{
  "id": "toolu_01SPiC3fscgboWE9woTp7W8y",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Add Ledger Plugin section to README.md (AC1 + AC2)",
        "status": "in_progress"
      },
      {
        "content": "Add contributor guide section to AGENTS.md (AC3)",
        "status": "pending"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_01SPiC3fscgboWE9woTp7W8y`)

```json
{
  "todos": [
    {
      "content": "Add Ledger Plugin section to README.md (AC1 + AC2)",
      "status": "in_progress"
    },
    {
      "content": "Add contributor guide section to AGENTS.md (AC3)",
      "status": "pending"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Add Ledger Plugin section to README.md (AC1 + AC2)', 'status': 'in_progress'}, {'content': 'Add contributor guide section to AGENTS.md (AC3)', 'status': 'pending'}]

## Assistant

```json
{
  "id": "toolu_01HJbCDbA5SnzJnA8QBji55y",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/README.md",
    "old_string": "## \ud83d\udcc4 License\n\nMIT",
    "new_string": "## \ud83d\udd0c Ledger Plugin\n\nThe ledger plugin is a first-party plugin shipped as a sub-path export. It adds ledger-specific rendering (roster table, MCP tools table) and role validation into the standard build hooks.\n\n### Installation\n\nThe plugin ships with the library \u2014 no extra install needed.\n\n```bash\nnpm install @mistralys/persona-builder\n```\n\n### Usage\n\n```js\n// personas/persona-build.config.js\nconst { ledgerPlugin } = require('@mistralys/persona-builder/plugins/ledger');\nconst manifest = require('../shared/workflow-manifest.json');\n\nmodule.exports = {\n  rootDir: __dirname,\n  sharedPartialsDir: './shared/partials',\n  suites: {\n    ledger: {\n      srcDir: './ledger/src',\n      outVscode: './ledger/vs-code',\n      outClaudeCode: './ledger/claude-code',\n      personaMode: 'numbered',\n    },\n    standalone: {\n      srcDir: './standalone/src',\n      outVscode: './standalone/vs-code',\n      outClaudeCode: './standalone/claude-code',\n      personaMode: 'standalone',\n    },\n  },\n  plugins: [\n    ledgerPlugin({\n      manifestRoles: manifest.roles.map(r => r.name),\n      warnOnUnknownRole: true,\n    }),\n  ],\n};\n```\n\n### Options \u2014 `LedgerPluginOptions`\n\n| Option | Type | Default | Description |\n|--------|------|---------|-------------|\n| `manifestRoles` | `ReadonlyArray<string>` | `[]` | Canonical role names from your workflow manifest. Each persona's `role` field is validated against this list. When omitted or empty, role validation is skipped. |\n| `warnOnUnknownRole` | `boolean` | `true` | When `true`, an unknown `role` field emits a warning-level validation result. |\n\nSee the [Plugins reference](docs/plugins.md#ledger-plugin----mistralys-persona-builderpluginsleeger) for full hook documentation and exported types (`RosterEntry`, `McpToolEntry`).\n\n---\n\n## \ud83d\udcc4 License\n\nMIT"
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_01HJbCDbA5SnzJnA8QBji55y`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/README.md",
  "old_string": "## \ud83d\udcc4 License\n\nMIT",
  "new_string": "## \ud83d\udd0c Ledger Plugin\n\nThe ledger plugin is a first-party plugin shipped as a sub-path export. It adds ledger-specific rendering (roster table, MCP tools table) and role validation into the standard build hooks.\n\n### Installation\n\nThe plugin ships with the library \u2014 no extra install needed.\n\n```bash\nnpm install @mistralys/persona-builder\n```\n\n### Usage\n\n```js\n// personas/persona-build.config.js\nconst { ledgerPlugin } = require('@mistralys/persona-builder/plugins/ledger');\nconst manifest = require('../shared/workflow-manifest.json');\n\nmodule.exports = {\n  rootDir: __dirname,\n  sharedPartialsDir: './shared/partials',\n  suites: {\n    ledger: {\n      srcDir: './ledger/src',\n      outVscode: './ledger/vs-code',\n      outClaudeCode: './ledger/claude-code',\n      personaMode: 'numbered',\n    },\n    standalone: {\n      srcDir: './standalone/src',\n      outVscode: './standalone/vs-code',\n      outClaudeCode: './standalone/claude-code',\n      personaMode: 'standalone',\n    },\n  },\n  plugins: [\n    ledgerPlugin({\n      manifestRoles: manifest.roles.map(r => r.name),\n      warnOnUnknownRole: true,\n    }),\n  ],\n};\n```\n\n### Options \u2014 `LedgerPluginOptions`\n\n| Option | Type | Default | Description |\n|--------|------|---------|-------------|\n| `manifestRoles` | `ReadonlyArray<string>` | `[]` | Canonical role names from your workflow manifest. Each persona's `role` field is validated against this list. When omitted or empty, role validation is skipped. |\n| `warnOnUnknownRole` | `boolean` | `true` | When `true`, an unknown `role` field emits a warning-level validation result. |\n\nSee the [Plugins reference](docs/plugins.md#ledger-plugin----mistralys-persona-builderpluginsleeger) for full hook documentation and exported types (`RosterEntry`, `McpToolEntry`).\n\n---\n\n## \ud83d\udcc4 License\n\nMIT"
}
```

## Tool Result

Successfully replaced 1 instance(s) of the string in '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/README.md'

## Assistant

```json
{
  "id": "toolu_01E7sVtKWj1JRCBbAs8eJQ6C",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Add Ledger Plugin section to README.md (AC1 + AC2)",
        "status": "completed"
      },
      {
        "content": "Add contributor guide section to AGENTS.md (AC3)",
        "status": "in_progress"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_01E7sVtKWj1JRCBbAs8eJQ6C`)

```json
{
  "todos": [
    {
      "content": "Add Ledger Plugin section to README.md (AC1 + AC2)",
      "status": "completed"
    },
    {
      "content": "Add contributor guide section to AGENTS.md (AC3)",
      "status": "in_progress"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Add Ledger Plugin section to README.md (AC1 + AC2)', 'status': 'completed'}, {'content': 'Add contributor guide section to AGENTS.md (AC3)', 'status': 'in_progress'}]

## Assistant

Now add the contributor guide section to AGENTS.md. I'll insert it before the Navigation Quick Reference section:

```json
{
  "id": "toolu_01289qvUq77RH67cQYqP2oES",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/AGENTS.md",
    "old_string": "## \ud83e\udded Navigation Quick Reference",
    "new_string": "## \ud83d\udee0\ufe0f Contributor Guide\n\n### Repo Layout\n\nThe repo follows a layered architecture. Key directories:\n\n| Path | Purpose |\n|------|---------|\n| `src/engine/` | Pure template rendering functions (zero external dependencies) |\n| `src/loaders/` | File I/O \u2014 YAML metadata, Markdown content, partials |\n| `src/builders/` | Build orchestration (`build()`, `buildSuite()`, `buildPersona()`) |\n| `src/plugins/` | Plugin runner, types, and first-party plugins |\n| `src/plugins/ledger/` | Ledger plugin \u2014 sub-path export `@mistralys/persona-builder/plugins/ledger` |\n| `src/validators/` | Filename and strict-mode validators |\n| `src/cli.ts` | `persona-build` CLI entry point |\n| `tests/` | Vitest test suites mirroring the `src/` structure |\n| `fixtures/` | Test fixtures (`sample-suite/`, `shared/`) |\n| `dist/` | Build output \u2014 gitignored, generated by `npm run build` |\n| `docs/` | User-facing documentation and the agent project manifest |\n\nSee [`docs/agents/project-manifest/file-tree.md`](docs/agents/project-manifest/file-tree.md) for the full annotated directory listing.\n\n### Test Command\n\n```bash\nnpm test\n```\n\nRuns all Vitest tests once (227 tests across 14 files). Use `npm run test:watch` during development.\n\n### Build Command\n\n```bash\nnpm run build\n```\n\nProduces dual CJS + ESM output via tsup into `dist/`. Three entry points are compiled: `index`, `cli`, and `plugins/ledger/index`.\n\n### How to Add a New Plugin\n\n1. **Create the plugin source** at `src/plugins/<your-plugin>/index.ts`.  \n   Implement the `PersonaBuildPlugin` interface (see [`docs/plugins.md`](docs/plugins.md#personabuildplugin-interface)):\n\n   ```ts\n   import type { PersonaBuildPlugin } from '../../plugins/types.js';\n\n   export function myPlugin(options?: MyPluginOptions): PersonaBuildPlugin {\n     return {\n       name: 'my-plugin',\n\n       onBuildContext(context, persona, suite) {\n         return { ...context, my_variable: 'value' };\n       },\n     };\n   }\n   ```\n\n2. **Add an entry point to `tsup.config.ts`** so the plugin gets its own sub-path export:\n\n   ```ts\n   entry: [\n     'src/index.ts',\n     'src/cli.ts',\n     'src/plugins/ledger/index.ts',\n     'src/plugins/my-plugin/index.ts',   // \u2190 add this\n   ],\n   ```\n\n3. **Add the sub-path export to `package.json`**:\n\n   ```json\n   \"exports\": {\n     \"./plugins/my-plugin\": {\n       \"import\": \"./dist/plugins/my-plugin/index.js\",\n       \"require\": \"./dist/plugins/my-plugin/index.cjs\",\n       \"types\": \"./dist/plugins/my-plugin/index.d.ts\"\n     }\n   }\n   ```\n\n4. **Write tests** in `tests/plugins/<your-plugin>.test.ts`, following the pattern in `tests/plugins/plugin-runner.test.ts`.\n\n5. **Document** the plugin in `docs/plugins.md` and update `README.md` if the plugin is first-party.\n\n---\n\n## \ud83e\udded Navigation Quick Reference"
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_01289qvUq77RH67cQYqP2oES`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/AGENTS.md",
  "old_string": "## \ud83e\udded Navigation Quick Reference",
  "new_string": "## \ud83d\udee0\ufe0f Contributor Guide\n\n### Repo Layout\n\nThe repo follows a layered architecture. Key directories:\n\n| Path | Purpose |\n|------|---------|\n| `src/engine/` | Pure template rendering functions (zero external dependencies) |\n| `src/loaders/` | File I/O \u2014 YAML metadata, Markdown content, partials |\n| `src/builders/` | Build orchestration (`build()`, `buildSuite()`, `buildPersona()`) |\n| `src/plugins/` | Plugin runner, types, and first-party plugins |\n| `src/plugins/ledger/` | Ledger plugin \u2014 sub-path export `@mistralys/persona-builder/plugins/ledger` |\n| `src/validators/` | Filename and strict-mode validators |\n| `src/cli.ts` | `persona-build` CLI entry point |\n| `tests/` | Vitest test suites mirroring the `src/` structure |\n| `fixtures/` | Test fixtures (`sample-suite/`, `shared/`) |\n| `dist/` | Build output \u2014 gitignored, generated by `npm run build` |\n| `docs/` | User-facing documentation and the agent project manifest |\n\nSee [`docs/agents/project-manifest/file-tree.md`](docs/agents/project-manifest/file-tree.md) for the full annotated directory listing.\n\n### Test Command\n\n```bash\nnpm test\n```\n\nRuns all Vitest tests once (227 tests across 14 files). Use `npm run test:watch` during development.\n\n### Build Command\n\n```bash\nnpm run build\n```\n\nProduces dual CJS + ESM output via tsup into `dist/`. Three entry points are compiled: `index`, `cli`, and `plugins/ledger/index`.\n\n### How to Add a New Plugin\n\n1. **Create the plugin source** at `src/plugins/<your-plugin>/index.ts`.  \n   Implement the `PersonaBuildPlugin` interface (see [`docs/plugins.md`](docs/plugins.md#personabuildplugin-interface)):\n\n   ```ts\n   import type { PersonaBuildPlugin } from '../../plugins/types.js';\n\n   export function myPlugin(options?: MyPluginOptions): PersonaBuildPlugin {\n     return {\n       name: 'my-plugin',\n\n       onBuildContext(context, persona, suite) {\n         return { ...context, my_variable: 'value' };\n       },\n     };\n   }\n   ```\n\n2. **Add an entry point to `tsup.config.ts`** so the plugin gets its own sub-path export:\n\n   ```ts\n   entry: [\n     'src/index.ts',\n     'src/cli.ts',\n     'src/plugins/ledger/index.ts',\n     'src/plugins/my-plugin/index.ts',   // \u2190 add this\n   ],\n   ```\n\n3. **Add the sub-path export to `package.json`**:\n\n   ```json\n   \"exports\": {\n     \"./plugins/my-plugin\": {\n       \"import\": \"./dist/plugins/my-plugin/index.js\",\n       \"require\": \"./dist/plugins/my-plugin/index.cjs\",\n       \"types\": \"./dist/plugins/my-plugin/index.d.ts\"\n     }\n   }\n   ```\n\n4. **Write tests** in `tests/plugins/<your-plugin>.test.ts`, following the pattern in `tests/plugins/plugin-runner.test.ts`.\n\n5. **Document** the plugin in `docs/plugins.md` and update `README.md` if the plugin is first-party.\n\n---\n\n## \ud83e\udded Navigation Quick Reference"
}
```

## Tool Result

Successfully replaced 1 instance(s) of the string in '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/AGENTS.md'

## Assistant

```json
{
  "id": "toolu_019YdUQsA347GVtZPtyc2xPp",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Add Ledger Plugin section to README.md (AC1 + AC2)",
        "status": "completed"
      },
      {
        "content": "Add contributor guide section to AGENTS.md (AC3)",
        "status": "completed"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_019YdUQsA347GVtZPtyc2xPp`)

```json
{
  "todos": [
    {
      "content": "Add Ledger Plugin section to README.md (AC1 + AC2)",
      "status": "completed"
    },
    {
      "content": "Add contributor guide section to AGENTS.md (AC3)",
      "status": "completed"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Add Ledger Plugin section to README.md (AC1 + AC2)', 'status': 'completed'}, {'content': 'Add contributor guide section to AGENTS.md (AC3)', 'status': 'completed'}]

## Assistant

Let me quickly verify the final state of both files:

```json
{
  "id": "toolu_017rFMTNm4jKVB3KW8aCpMBN",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/README.md",
    "offset": 60,
    "limit": 80
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01NzM1DrvSWrB5pS2umW19AL",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/AGENTS.md",
    "offset": 118,
    "limit": 100
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_017rFMTNm4jKVB3KW8aCpMBN`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/README.md",
  "offset": 60,
  "limit": 80
}
```

**Tool call:** `read_file` (id: `toolu_01NzM1DrvSWrB5pS2umW19AL`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/AGENTS.md",
  "offset": 118,
  "limit": 100
}
```

## Tool Result

    61	| Guide | Description |
    62	|-------|-------------|
    63	| [Directory Convention](docs/directory-convention.md) | Expected source layout (`meta/`, `content/`, `partials/`) |
    64	| [Template Syntax](docs/template-syntax.md) | Variables, partials, conditionals, and built-in context variables |
    65	| [Configuration Reference](docs/configuration.md) | `BuildConfig`, `SuiteConfig`, and `BuildSummary` fields |
    66	| [CLI Reference](docs/cli.md) | Command-line flags, config file format, and common patterns |
    67	| [Plugins](docs/plugins.md) | `PersonaBuildPlugin` interface, examples, and the built-in Ledger Plugin |
    68	| [Public API](docs/api.md) | All exported types and functions |
    69	
    70	## 🔌 Ledger Plugin
    71	
    72	The ledger plugin is a first-party plugin shipped as a sub-path export. It adds ledger-specific rendering (roster table, MCP tools table) and role validation into the standard build hooks.
    73	
    74	### Installation
    75	
    76	The plugin ships with the library — no extra install needed.
    77	
    78	```bash
    79	npm install @mistralys/persona-builder
    80	```
    81	
    82	### Usage
    83	
    84	```js
    85	// personas/persona-build.config.js
    86	const { ledgerPlugin } = require('@mistralys/persona-builder/plugins/ledger');
    87	const manifest = require('../shared/workflow-manifest.json');
    88	
    89	module.exports = {
    90	  rootDir: __dirname,
    91	  sharedPartialsDir: './shared/partials',
    92	  suites: {
    93	    ledger: {
    94	      srcDir: './ledger/src',
    95	      outVscode: './ledger/vs-code',
    96	      outClaudeCode: './ledger/claude-code',
    97	      personaMode: 'numbered',
    98	    },
    99	    standalone: {
   100	      srcDir: './standalone/src',
   101	      outVscode: './standalone/vs-code',
   102	      outClaudeCode: './standalone/claude-code',
   103	      personaMode: 'standalone',
   104	    },
   105	  },
   106	  plugins: [
   107	    ledgerPlugin({
   108	      manifestRoles: manifest.roles.map(r => r.name),
   109	      warnOnUnknownRole: true,
   110	    }),
   111	  ],
   112	};
   113	```
   114	
   115	### Options — `LedgerPluginOptions`
   116	
   117	| Option | Type | Default | Description |
   118	|--------|------|---------|-------------|
   119	| `manifestRoles` | `ReadonlyArray<string>` | `[]` | Canonical role names from your workflow manifest. Each persona's `role` field is validated against this list. When omitted or empty, role validation is skipped. |
   120	| `warnOnUnknownRole` | `boolean` | `true` | When `true`, an unknown `role` field emits a warning-level validation result. |
   121	
   122	See the [Plugins reference](docs/plugins.md#ledger-plugin----mistralys-persona-builderpluginsleeger) for full hook documentation and exported types (`RosterEntry`, `McpToolEntry`).
   123	
   124	---
   125	
   126	## 📄 License
   127	
   128	MIT

## Tool Result

   119	---
   120	
   121	## 📊 Project Stats
   122	
   123	| Property | Value |
   124	|----------|-------|
   125	| **Package** | `@mistralys/persona-builder` |
   126	| **Version** | 1.0.0 |
   127	| **Language** | TypeScript 5.8.2 (ES2022) |
   128	| **Runtime** | Node.js ≥ 18.17 (ESM) |
   129	| **Architecture** | Layered: builders → plugins → engine / loaders / validators |
   130	| **Package Manager** | npm |
   131	| **Build Tool** | tsup (dual CJS + ESM) |
   132	| **Test Framework** | Vitest (227 tests across 14 files) |
   133	| **Production Dependency** | `js-yaml` (sole) |
   134	| **License** | MIT |
   135	
   136	### npm Scripts
   137	
   138	| Script | Purpose |
   139	|--------|---------|
   140	| `npm run build` | Production build via tsup. |
   141	| `npm run dev` | Watch-mode build. |
   142	| `npm test` | Run all tests once. |
   143	| `npm run test:watch` | Run tests in watch mode. |
   144	| `npm run typecheck` | Type-check without emitting (`tsc --noEmit`). |
   145	
   146	---
   147	
   148	## 🛠️ Contributor Guide
   149	
   150	### Repo Layout
   151	
   152	The repo follows a layered architecture. Key directories:
   153	
   154	| Path | Purpose |
   155	|------|---------|
   156	| `src/engine/` | Pure template rendering functions (zero external dependencies) |
   157	| `src/loaders/` | File I/O — YAML metadata, Markdown content, partials |
   158	| `src/builders/` | Build orchestration (`build()`, `buildSuite()`, `buildPersona()`) |
   159	| `src/plugins/` | Plugin runner, types, and first-party plugins |
   160	| `src/plugins/ledger/` | Ledger plugin — sub-path export `@mistralys/persona-builder/plugins/ledger` |
   161	| `src/validators/` | Filename and strict-mode validators |
   162	| `src/cli.ts` | `persona-build` CLI entry point |
   163	| `tests/` | Vitest test suites mirroring the `src/` structure |
   164	| `fixtures/` | Test fixtures (`sample-suite/`, `shared/`) |
   165	| `dist/` | Build output — gitignored, generated by `npm run build` |
   166	| `docs/` | User-facing documentation and the agent project manifest |
   167	
   168	See [`docs/agents/project-manifest/file-tree.md`](docs/agents/project-manifest/file-tree.md) for the full annotated directory listing.
   169	
   170	### Test Command
   171	
   172	```bash
   173	npm test
   174	```
   175	
   176	Runs all Vitest tests once (227 tests across 14 files). Use `npm run test:watch` during development.
   177	
   178	### Build Command
   179	
   180	```bash
   181	npm run build
   182	```
   183	
   184	Produces dual CJS + ESM output via tsup into `dist/`. Three entry points are compiled: `index`, `cli`, and `plugins/ledger/index`.
   185	
   186	### How to Add a New Plugin
   187	
   188	1. **Create the plugin source** at `src/plugins/<your-plugin>/index.ts`.  
   189	   Implement the `PersonaBuildPlugin` interface (see [`docs/plugins.md`](docs/plugins.md#personabuildplugin-interface)):
   190	
   191	   ```ts
   192	   import type { PersonaBuildPlugin } from '../../plugins/types.js';
   193	
   194	   export function myPlugin(options?: MyPluginOptions): PersonaBuildPlugin {
   195	     return {
   196	       name: 'my-plugin',
   197	
   198	       onBuildContext(context, persona, suite) {
   199	         return { ...context, my_variable: 'value' };
   200	       },
   201	     };
   202	   }
   203	   ```
   204	
   205	2. **Add an entry point to `tsup.config.ts`** so the plugin gets its own sub-path export:
   206	
   207	   ```ts
   208	   entry: [
   209	     'src/index.ts',
   210	     'src/cli.ts',
   211	     'src/plugins/ledger/index.ts',
   212	     'src/plugins/my-plugin/index.ts',   // ← add this
   213	   ],
   214	   ```
   215	
   216	3. **Add the sub-path export to `package.json`**:
   217	
   218	   ```json

## Assistant

Both files look correct. Now complete the pipeline:

```json
{
  "id": "toolu_01DqiER2QKi97eZLXqjsdHeJ",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-007",
    "type": "documentation",
    "agent_role": "Documentation",
    "status": "PASS",
    "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration",
    "summary": [
      "Added ## Ledger Plugin section to README.md covering: install note, full working usage code example (matching plan.md Architectural Context config), and an options table documenting manifestRoles (ReadonlyArray<string>, default []) and warnOnUnknownRole (boolean, default true) with types and descriptions. Link to docs/plugins.md for extended reference.",
      "Added ## Contributor Guide section to AGENTS.md covering: repo layout table (all key src/ directories with purpose), test command (npm test), build command (npm run build), and step-by-step 'How to Add a New Plugin' guide (create src/plugins/<your-plugin>/index.ts implementing PersonaBuildPlugin, add tsup.config.ts entry, add package.json exports field, write tests in tests/plugins/)."
    ],
    "artifacts": {
      "files_modified": [
        "README.md",
        "AGENTS.md"
      ]
    },
    "acceptance_criteria_updates": [
      {
        "criterion": "README.md contains a ledger plugin section with a working usage code example matching the config shown in the plan",
        "met": true
      },
      {
        "criterion": "README.md documents all ledgerPlugin() options (manifestRoles, warnOnUnknownRole) with types and descriptions",
        "met": true
      },
      {
        "criterion": "AGENTS.md exists at the repo root and covers: repo layout, test command, build command, and how to add a new plugin",
        "met": true
      }
    ],
    "comments": [
      {
        "type": "improvement",
        "priority": "low",
        "note": "README.md \u00a7Ledger Plugin anchor link in the 'See Plugins reference' line uses a manually constructed fragment (#ledger-plugin----mistralys-persona-builderpluginsleeger) which may not render correctly in all Markdown renderers. Consider simplifying the anchor to match GitHub's auto-generated fragment ID once the docs are published."
      }
    ]
  },
  "name": "ledger_complete_pipeline",
  "type": "tool_use"
}
```

**Tool call:** `ledger_complete_pipeline` (id: `toolu_01DqiER2QKi97eZLXqjsdHeJ`)

```json
{
  "work_package_id": "WP-007",
  "type": "documentation",
  "agent_role": "Documentation",
  "status": "PASS",
  "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration",
  "summary": [
    "Added ## Ledger Plugin section to README.md covering: install note, full working usage code example (matching plan.md Architectural Context config), and an options table documenting manifestRoles (ReadonlyArray<string>, default []) and warnOnUnknownRole (boolean, default true) with types and descriptions. Link to docs/plugins.md for extended reference.",
    "Added ## Contributor Guide section to AGENTS.md covering: repo layout table (all key src/ directories with purpose), test command (npm test), build command (npm run build), and step-by-step 'How to Add a New Plugin' guide (create src/plugins/<your-plugin>/index.ts implementing PersonaBuildPlugin, add tsup.config.ts entry, add package.json exports field, write tests in tests/plugins/)."
  ],
  "artifacts": {
    "files_modified": [
      "README.md",
      "AGENTS.md"
    ]
  },
  "acceptance_criteria_updates": [
    {
      "criterion": "README.md contains a ledger plugin section with a working usage code example matching the config shown in the plan",
      "met": true
    },
    {
      "criterion": "README.md documents all ledgerPlugin() options (manifestRoles, warnOnUnknownRole) with types and descriptions",
      "met": true
    },
    {
      "criterion": "AGENTS.md exists at the repo root and covers: repo layout, test command, build command, and how to add a new plugin",
      "met": true
    }
  ],
  "comments": [
    {
      "type": "improvement",
      "priority": "low",
      "note": "README.md \u00a7Ledger Plugin anchor link in the 'See Plugins reference' line uses a manually constructed fragment (#ledger-plugin----mistralys-persona-builderpluginsleeger) which may not render correctly in all Markdown renderers. Consider simplifying the anchor to match GitHub's auto-generated fragment ID once the docs are published."
    }
  ]
}
```

## Tool Result

{
  "work_package_id": "WP-007",
  "work_package_file": "work/WP-007.md",
  "status": "COMPLETE",
  "assigned_to": "Documentation",
  "dependencies": [
    "WP-003"
  ],
  "acceptance_criteria": [
    {
      "criterion": "README.md contains a ledger plugin section with a working usage code example matching the config shown in the plan",
      "met": true
    },
    {
      "criterion": "README.md documents all ledgerPlugin() options (manifestRoles, warnOnUnknownRole) with types and descriptions",
      "met": true
    },
    {
      "criterion": "AGENTS.md exists at the repo root and covers: repo layout, test command, build command, and how to add a new plugin",
      "met": true
    },
    {
      "criterion": "npm pack --dry-run output includes the compiled ledger plugin files under the exports path and excludes *.test.* files and src/ TypeScript sources",
      "met": true
    },
    {
      "criterion": "A git tag v1.0.0 exists on the commit that represents the final state of the library after all WP-001–WP-003 work",
      "met": true
    }
  ],
  "active_pipeline_stages": [
    "release-engineering",
    "documentation"
  ],
  "revision": 0,
  "pipelines": [
    {
      "type": "release-engineering",
      "status": "PASS",
      "started_at": "2026-03-25T21:29:05Z",
      "completed_at": "2026-03-25T21:35:16Z",
      "duration_ms": 371000,
      "summary": [
        "Bumped version from 0.2.0 to 1.0.0 in package.json (first stable release).",
        "Updated version references in AGENTS.md, docs/agents/project-manifest/README.md, and docs/api.md.",
        "Curated CHANGELOG.md: promoted [Unreleased] content to [1.0.0] entry (2026-03-25), added factory/export and unit-test bullets, added Keep-a-Changelog link references block.",
        "Verified package manifest: 'files': ['dist'] covers dist/plugins/ledger/index.{js,cjs,d.ts,d.cts,.map} — ledger plugin exports are present; no *.test.* files or src/ TypeScript sources in dist/.",
        "Committed all pending changes (18 files, 2342 insertions) in commit ae93c2b: 'Release v1.0.0: ledger plugin, unit tests, publish prep'.",
        "Created annotated git tag v1.0.0 on commit ae93c2b.",
        "No migration guide required — v1.0.0 adds new capabilities (ledger plugin sub-path export) without removing or breaking any existing public API surface."
      ],
      "artifacts": {
        "files_modified": [
          "package.json",
          "CHANGELOG.md",
          "AGENTS.md",
          "docs/agents/project-manifest/README.md",
          "docs/api.md",
          "src/plugins/ledger/frontmatter-templates.ts",
          "src/plugins/ledger/index.ts",
          "src/plugins/ledger/mcp-tools-renderer.ts",
          "src/plugins/ledger/role-validator.ts",
          "src/plugins/ledger/roster-renderer.ts",
          "tests/plugins/ledger.test.ts"
        ],
        "commit_hash": "ae93c2b"
      },
      "comments": [
        {
          "type": "version-decision",
          "priority": "medium",
          "timestamp": "2026-03-25T21:35:16Z",
          "note": "Version bumped to 1.0.0 (not 0.3.0). Rationale: this is the first stable public release per the plan's explicit 'Tag version — v1.0.0 release' instruction. The ledger plugin sub-path export (@mistralys/persona-builder/plugins/ledger) completes the full public API surface described in the plan. No existing API was removed or changed, so there is no breaking change — 1.0.0 is a marketing/stability statement, not a semver breaking-change indicator."
        },
        {
          "type": "release-note",
          "priority": "low",
          "timestamp": "2026-03-25T21:35:16Z",
          "note": "Package manifest is clean for npm publish: 'files': ['dist'] in package.json means only dist/ ships. dist/plugins/ledger/ contains index.js, index.cjs, index.d.ts, index.d.cts, and source maps. No .test. files, no src/ TypeScript sources, no fixtures in the tarball. No .npmignore required."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-25T21:35:16Z",
          "note": "Node.js was not available in the sandbox shell (npm pack --dry-run could not be executed). Package manifest correctness was verified through direct file inspection: 'files' field, dist/ directory listing, and absence of test/source files in dist/. This should be re-verified manually before actual npm publish."
        }
      ]
    },
    {
      "type": "documentation",
      "status": "PASS",
      "started_at": "2026-03-25T21:46:01Z",
      "completed_at": "2026-03-25T21:47:26Z",
      "duration_ms": 85000,
      "summary": [
        "Added ## Ledger Plugin section to README.md covering: install note, full working usage code example (matching plan.md Architectural Context config), and an options table documenting manifestRoles (ReadonlyArray<string>, default []) and warnOnUnknownRole (boolean, default true) with types and descriptions. Link to docs/plugins.md for extended reference.",
        "Added ## Contributor Guide section to AGENTS.md covering: repo layout table (all key src/ directories with purpose), test command (npm test), build command (npm run build), and step-by-step 'How to Add a New Plugin' guide (create src/plugins/<your-plugin>/index.ts implementing PersonaBuildPlugin, add tsup.config.ts entry, add package.json exports field, write tests in tests/plugins/)."
      ],
      "artifacts": {
        "files_modified": [
          "README.md",
          "AGENTS.md"
        ]
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-25T21:47:26Z",
          "note": "README.md §Ledger Plugin anchor link in the 'See Plugins reference' line uses a manually constructed fragment (#ledger-plugin----mistralys-persona-builderpluginsleeger) which may not render correctly in all Markdown renderers. Consider simplifying the anchor to match GitHub's auto-generated fragment ID once the docs are published."
        }
      ]
    }
  ],
  "status_changed_at": "2026-03-25T21:47:26Z",
  "last_updated": "2026-03-25T21:47:26Z",
  "handoff_notes": [
    {
      "from_agent": "Release Engineer",
      "to_agent": "Documentation",
      "timestamp": "2026-03-25T21:35:16Z",
      "notes": [
        "Documentation agent: two acceptance criteria remain unmet and are your responsibility:\n\n1. README.md ledger plugin section (AC1 + AC2): The current README.md references docs/plugins.md in a documentation table but has no inline ledger plugin section. You need to add a ## Ledger Plugin section to README.md that includes: (a) a working usage code example matching the config in the plan (the personas/persona-build.config.js example from the plan.md Architectural Context section), and (b) documents all ledgerPlugin() options — manifestRoles (ReadonlyArray<string>, optional, defaults to [] which skips validation) and warnOnUnknownRole (boolean, default: true). The full option docs are already in docs/plugins.md §LedgerPluginOptions — adapt them for the README inline section.\n\n2. AGENTS.md 'how to add a new plugin' section (AC3): AGENTS.md has test/build commands and references file-tree.md for repo layout, but lacks an explicit contributor guide section covering: repo layout overview, test command (npm test), build command (npm run build), and how to add a new plugin (create src/plugins/your-plugin/index.ts implementing PersonaBuildPlugin, add to tsup.config.ts entry points for sub-path export, add to package.json exports field, write tests in tests/plugins/). See docs/plugins.md for the PersonaBuildPlugin interface reference."
      ]
    }
  ],
  "auto_finalized": true
}

--- NEXT STEP ---
Pipeline PASS. WP WP-007 was auto-finalized to COMPLETE (all acceptance criteria met). Call ledger_get_handoff_status (current_agent: "Documentation") to confirm handoff.

## Assistant

WP-007 is `COMPLETE` — all 5 acceptance criteria met and the WP auto-finalized. Now check handoff:

```json
{
  "id": "toolu_01LvwfjdGcfYBgCtqJAcfL69",
  "caller": {
    "type": "direct"
  },
  "input": {
    "agent_role": "Documentation",
    "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration"
  },
  "name": "ledger_get_next_action",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_next_action` (id: `toolu_01LvwfjdGcfYBgCtqJAcfL69`)

```json
{
  "agent_role": "Documentation",
  "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration"
}
```

## Tool Result

{
  "action": "WAIT",
  "reason": "All work packages are COMPLETE. Project is ready for Synthesis agent.",
  "handoff_status": {
    "current_agent": "Documentation",
    "next_agent": "Developer",
    "status": "READY_FOR_DEVELOPER",
    "details": "Documentation complete for all reviewed work packages. 2 work package(s) still need earlier stages: WP-006, WP-007. Hand back to Developer.",
    "auto_handoff": {
      "agent_name": "3 - Developer v3.6.1",
      "agent_id": "ledger-3-dev",
      "prompt": "@ledger-3-dev\nProject path: /Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration"
    }
  }
}

## Assistant

```json
{
  "id": "toolu_01PiHXezNMptaFhrpHLiAiGo",
  "caller": {
    "type": "direct"
  },
  "input": {
    "subagent_type": "general-purpose",
    "description": "3-developer"
  },
  "name": "task",
  "type": "tool_use"
}
```

**Tool call:** `task` (id: `toolu_01PiHXezNMptaFhrpHLiAiGo`)

```json
{
  "subagent_type": "general-purpose",
  "description": "3-developer"
}
```

## Tool Result

All work packages in the current project are already **COMPLETE** — there's nothing left for the Developer to do.

**Status:** `WAIT` — All WPs are in a terminal state.

**Next step:** The project is ready for the **Synthesis** agent.

> 📋 **Handoff to:** `9 - Synthesis v3.5.3`
> **Project path:** `/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration`

You can invoke the Synthesis agent with:
```
@ledger-7-synthesis
Project path: /Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-25-persona-build-integration
```

## Assistant

WP-007 documentation is complete. Here's a summary of what was done:

**`README.md`** — Added a `## 🔌 Ledger Plugin` section containing:
- Installation note (bundled with the library)
- Full working `persona-build.config.js` usage example matching the plan's Architectural Context config exactly (both `ledger` and `standalone` suites, `manifestRoles`, `warnOnUnknownRole`)
- `LedgerPluginOptions` table documenting both options with types, defaults, and descriptions

**`AGENTS.md`** — Added a `## 🛠️ Contributor Guide` section covering:
- Repo layout table (all key `src/` directories and `tests/`, `fixtures/`, `dist/`, `docs/`)
- Test command (`npm test`) with description
- Build command (`npm run build`) with description
- Step-by-step "How to Add a New Plugin" guide (create `src/plugins/<your-plugin>/index.ts`, add `tsup.config.ts` entry, add `package.json` exports field, write tests in `tests/plugins/`)

All 5 acceptance criteria are now met — WP-007 auto-finalized to **COMPLETE**.

CURRENT AGENT: Documentation
NEXT AGENT: Developer
STATUS: READY_FOR_DEVELOPER

---

## Token Usage

| Metric | Count |
| ------ | ----- |
| Input Tokens | 631909 |
| Output Tokens | 4813 |
| Total Tokens | 636722 |

