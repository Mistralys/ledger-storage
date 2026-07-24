# Dialogue — docs / WP-004

| Field | Value |
| ----- | ----- |
| Stage | `docs` |
| WP ID | `WP-004` |
| Captured | 2026-03-26T12:46:28.709283+00:00 |

## Human

**Project:** `/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights`
**Work package:** WP-004

Always use the project path above for all ledger tool calls.

CRITICAL: Every MCP tool call MUST use `work_package_id=WP-004`. Do NOT reference or operate on any other work package.


## Assistant

I'll start by getting the project status and work package details to understand what documentation needs to be done.

```json
{
  "id": "toolu_01Qe9xTKmris3CDuNQmF17Qj",
  "caller": {
    "type": "direct"
  },
  "input": {
    "agent_role": "Documentation",
    "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights"
  },
  "name": "ledger_get_next_action",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01YZVRxJAwqqMxTm73bdyY9u",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-004",
    "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights"
  },
  "name": "ledger_get_work_package",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_next_action` (id: `toolu_01Qe9xTKmris3CDuNQmF17Qj`)

```json
{
  "agent_role": "Documentation",
  "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights"
}
```

**Tool call:** `ledger_get_work_package` (id: `toolu_01YZVRxJAwqqMxTm73bdyY9u`)

```json
{
  "work_package_id": "WP-004",
  "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights"
}
```

## Tool Result

{
  "action": "WRITE_DOCS",
  "work_package_id": "WP-004",
  "reason": "Work package WP-004 has no prior documentation pipeline and documentation is the first active stage. Write documentation.",
  "next_steps": [
    "1. Call ledger_begin_work (work_package_id: \"WP-004\", type: \"documentation\", agent_role: \"Documentation\").",
    "2. Call ledger_get_work_package to review implementation artifacts and review comments.",
    "3. Update documentation, README files, and inline docs as needed.",
    "4. Call ledger_complete_pipeline (work_package_id: \"WP-004\", type: \"documentation\", status: PASS/FAIL, summary, artifacts, comments, acceptance_criteria_updates).",
    "5. Call ledger_get_handoff_status (current_agent: \"Documentation\")."
  ]
}

## Tool Result

{
  "work_package_id": "WP-004",
  "work_package_file": "work/WP-004.md",
  "status": "READY",
  "assigned_to": null,
  "dependencies": [
    "WP-003"
  ],
  "acceptance_criteria": [
    {
      "criterion": "docs/plugins.md removes the ledger plugin section but retains the plugin interface documentation, and includes a note that the ledger plugin has migrated to ai-insights-dev",
      "met": false
    },
    {
      "criterion": "All three project-manifest files (api-surface.md, file-tree.md, data-flows.md) no longer reference the ledger plugin",
      "met": false
    },
    {
      "criterion": "CHANGELOG.md contains an entry for the upcoming 2.0.0 release documenting the ledger plugin removal as a breaking change",
      "met": false
    }
  ],
  "active_pipeline_stages": [
    "documentation"
  ],
  "revision": 0,
  "pipelines": [],
  "status_changed_at": "2026-03-26T12:46:10Z",
  "last_updated": "2026-03-26T12:46:10Z"
}

## Assistant

Now let me begin work on WP-004 and simultaneously read the relevant files:

```json
{
  "id": "toolu_015kDmvSzZ3mZPpJb7Q4YXXr",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-004",
    "type": "documentation",
    "agent_role": "Documentation",
    "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights"
  },
  "name": "ledger_begin_work",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_0151nskgJkVgg1JajbpYZ392",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights/work/WP-004.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `ledger_begin_work` (id: `toolu_015kDmvSzZ3mZPpJb7Q4YXXr`)

```json
{
  "work_package_id": "WP-004",
  "type": "documentation",
  "agent_role": "Documentation",
  "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights"
}
```

**Tool call:** `read_file` (id: `toolu_0151nskgJkVgg1JajbpYZ392`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights/work/WP-004.md"
}
```

## Tool Result

{
  "work_package_id": "WP-004",
  "work_package_file": "work/WP-004.md",
  "status": "IN_PROGRESS",
  "assigned_to": "Documentation",
  "dependencies": [
    "WP-003"
  ],
  "acceptance_criteria": [
    {
      "criterion": "docs/plugins.md removes the ledger plugin section but retains the plugin interface documentation, and includes a note that the ledger plugin has migrated to ai-insights-dev",
      "met": false
    },
    {
      "criterion": "All three project-manifest files (api-surface.md, file-tree.md, data-flows.md) no longer reference the ledger plugin",
      "met": false
    },
    {
      "criterion": "CHANGELOG.md contains an entry for the upcoming 2.0.0 release documenting the ledger plugin removal as a breaking change",
      "met": false
    }
  ],
  "active_pipeline_stages": [
    "documentation"
  ],
  "revision": 0,
  "pipelines": [
    {
      "type": "documentation",
      "status": "IN_PROGRESS",
      "started_at": "2026-03-26T12:46:38Z",
      "summary": []
    }
  ],
  "status_changed_at": "2026-03-26T12:46:38Z",
  "last_updated": "2026-03-26T12:46:38Z",
  "claimed": true
}

## Tool Result

     1	# WP-004: Update persona-builder documentation for ledger removal
     2	
     3	## Overview
     4	
     5	Update all documentation files in persona-builder to reflect that the ledger plugin has been removed and migrated to ai-insights-dev. Remove the ledger section from the plugins guide while preserving the plugin interface documentation, add a migration note, and update the project manifest files and CHANGELOG.
     6	
     7	## Assigned To
     8	
     9	Documentation
    10	
    11	## Status
    12	
    13	BLOCKED (depends on WP-003)
    14	
    15	## Dependencies
    16	
    17	- WP-003 (ledger plugin source must be removed before docs reflect that state)
    18	
    19	## Active Pipeline Stages
    20	
    21	- documentation
    22	
    23	## Scope
    24	
    25	### Modified files (persona-builder)
    26	- `docs/plugins.md` — Remove ledger plugin section, retain plugin interface documentation, add note that ledger plugin has migrated to ai-insights-dev
    27	- `docs/agents/project-manifest/api-surface.md` — Remove ledger plugin exports from the API surface
    28	- `docs/agents/project-manifest/file-tree.md` — Remove `src/plugins/ledger/` entries
    29	- `docs/agents/project-manifest/data-flows.md` — Remove ledger-specific references if any
    30	- `CHANGELOG.md` — Add entry for upcoming 2.0.0 release documenting ledger plugin removal as a breaking change
    31	
    32	## Technical Notes
    33	
    34	- The 2.0.0 version entry in CHANGELOG.md should be a Breaking Change entry
    35	- Example CHANGELOG entry:
    36	  ```
    37	  ## [2.0.0] - YYYY-MM-DD
    38	  ### Breaking Changes
    39	  - Removed the `./plugins/ledger` sub-path export. The ledger plugin has been migrated to the ai-insights-dev workspace as a local CommonJS module at `personas/plugins/ledger/`. No external consumers are affected (the library has no published consumers).
    40	  ```
    41	- The plugins.md should explain how to build a custom plugin (keep the interface docs) but remove the ledger-specific example section
    42	- persona-builder location: find the library project (sibling directory)
    43	
    44	## Acceptance Criteria
    45	
    46	1. `docs/plugins.md` removes the ledger plugin section but retains the plugin interface documentation, and includes a note that the ledger plugin has migrated to ai-insights-dev
    47	2. All three project-manifest files (`api-surface.md`, `file-tree.md`, `data-flows.md`) no longer reference the ledger plugin
    48	3. `CHANGELOG.md` contains an entry for the upcoming 2.0.0 release documenting the ledger plugin removal as a breaking change

## Assistant

Now let me find the persona-builder project location and read all relevant files:

```json
{
  "id": "toolu_01GTN2htiKqn8RQYxtu3wB6w",
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

**Tool call:** `ls` (id: `toolu_01GTN2htiKqn8RQYxtu3wB6w`)

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
  "id": "toolu_01NstgZfeovjbzd3LpaEGnRL",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE"
  },
  "name": "ls",
  "type": "tool_use"
}
```

**Tool call:** `ls` (id: `toolu_01NstgZfeovjbzd3LpaEGnRL`)

```json
{
  "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE"
}
```

## Tool Result

['/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/.git/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/.gitignore', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/AGENTS.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/CHANGELOG.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/CLAUDE.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/LICENSE', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/README.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/coverage/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/dist/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/fixtures/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/node_modules/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/package-lock.json', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/package.json', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/src/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tsconfig.json', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tsup.config.ts', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/vitest.config.ts']

## Assistant

```json
{
  "id": "toolu_015veEvUyBrvAdfCb7yjYUFp",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs"
  },
  "name": "ls",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01CGSNDqVAbkBn58W4LdYVro",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/CHANGELOG.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `ls` (id: `toolu_015veEvUyBrvAdfCb7yjYUFp`)

```json
{
  "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs"
}
```

**Tool call:** `read_file` (id: `toolu_01CGSNDqVAbkBn58W4LdYVro`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/CHANGELOG.md"
}
```

## Tool Result

['/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/api.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/cli.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/configuration.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/directory-convention.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/template-syntax.md']

## Tool Result

     1	# Changelog
     2	
     3	All notable changes to `@mistralys/persona-builder` will be documented in this file.
     4	
     5	The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
     6	and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
     7	
     8	## [Unreleased]
     9	
    10	## [1.0.1] - 2026-03-26
    11	
    12	Patch release addressing known tech-debt items, documentation fixes, and two consumer-side bug fixes identified in the post-integration synthesis.
    13	
    14	### Changed (library)
    15	
    16	- **Fix `warnOnUnknownRole` documentation** — Removed the stale "not yet wired" known-limitation blockquote from `docs/plugins.md` and replaced it with accurate documentation of the `warnOnUnknownRole` escalation contract (`true` → `warning`, `false` → `error`). Added a "Validator Severity Escalation Pattern" subsection for future plugin authors. Updated the corresponding JSDoc in `src/plugins/ledger/index.ts` and the `api-surface.md` manifest.
    17	- **Resolve `TargetType` dual re-export** — Eliminated the duplicate `TargetType` re-export from `src/builders/types.ts` and `src/builders/index.ts`, resolving a tech-debt item flagged in `constraints.md` before 1.0. `TargetType` is now exported exclusively via `src/plugins/types.ts` → `src/plugins/index.ts` → `src/index.ts`.
    18	- **Extract `escapeRegExp` to shared utility** — Moved the previously private `escapeRegExp` function from `src/plugins/ledger/role-validator.ts` into a new shared module at `src/utils/regex.ts` with a barrel at `src/utils/index.ts`. The function is now a named export of the library's main barrel, available to all future validators and plugins without duplication.
    19	- **Improve `renderedOutputCache` keying** — Extended the `onValidate` hook signature with an optional `target?: TargetType` parameter and propagated it through the plugin runner (`src/plugins/runner.ts`) and persona builder (`src/builders/persona-builder.ts`). The ledger plugin now uses a composite cache key `${persona.name}:${target}` (with `'unknown'` fallback) so that multi-target builds cache and validate output correctly per target.
    20	
    21	### Fixed (consumer — `ai-insights-dev`)
    22	
    23	- **`scripts/build-personas.js` version log bug** — Captured `oldVersion` before mutating `pkg.version`, so the console message now correctly shows `oldVersion → newVersion` instead of the old `newVersion → newVersion`.
    24	- **`scripts/build-personas.js` catch block exit code** — The catch block now propagates the library's own exit code via `err.status ?? 1` instead of always exiting with `1`.
    25	
    26	### Removed (consumer — `ai-insights-dev`)
    27	
    28	- Deleted the orphaned empty directories `scripts/lib/` and `scripts/tests/`.
    29	
    30	## [1.0.0] - 2026-03-25
    31	
    32	First stable public release. The plugin architecture is complete and the built-in ledger plugin is fully implemented, tested, and publicly exported.
    33	
    34	### Added
    35	
    36	- **Ledger plugin — core helpers** (`src/plugins/ledger/`) — four internal TypeScript modules forming the foundation of the built-in ledger plugin:
    37	  - `roster-renderer.ts` — `renderRoster(roster: RosterEntry[], activeNumber: number): string` — renders the agent roster as a numbered Markdown list with `(YOU)` suffix on the active entry. Ported from `scripts/lib/persona-helpers.js`.
    38	  - `mcp-tools-renderer.ts` — `renderMcpToolsTable(tools: McpToolEntry[]): string` — renders MCP tools as Markdown table rows, filtering out `note_only: true` entries. Ported from `scripts/lib/persona-helpers.js`.
    39	  - `role-validator.ts` — `validateRole(role, manifestRoles)` and `validateNoteOnlyGuard(output, mcpTools)` — pure validation helpers compatible with the `onValidate` plugin hook. `validateRole` warns when a persona role is absent from the workflow manifest; `validateNoteOnlyGuard` errors when a `note_only` tool leaks into rendered output (second-line defence after the renderer filter).
    40	  - `frontmatter-templates.ts` — `FRONTMATTER_LEDGER_VSCODE` and `FRONTMATTER_LEDGER_CC` string constants — ledger-suite frontmatter templates for VS Code and Claude Code targets respectively. Structurally identical to the originals in `build-personas.js`; `ccFrontmatterFields()` is inlined as a named constant.
    41	  - All four modules are pure functions with no file-system I/O, no side effects, and no global state. All exports carry explicit TypeScript types; no use of `any`.
    42	- **Ledger plugin — factory & public export** (`src/plugins/ledger/index.ts`):
    43	  - `ledgerPlugin(options?: LedgerPluginOptions): PersonaBuildPlugin` — factory function that composes the core helpers into a fully wired `PersonaBuildPlugin` instance.
    44	  - `LedgerPluginOptions` interface — `manifestRoles?: ReadonlyArray<string>` (scopes role validation to a known set) and `warnOnUnknownRole?: boolean` (default: `true`).
    45	  - Sub-path export `@mistralys/persona-builder/plugins/ledger` registered in the `exports` field of `package.json`; compiled artefacts `dist/plugins/ledger/index.{js,cjs,d.ts,d.cts}` present in dist.
    46	- **Ledger plugin — unit tests** (`tests/plugins/ledger.test.ts`) — comprehensive test suite covering the roster renderer, MCP tools renderer, role validator, `note_only` guard, plugin hook composition, and `LedgerPluginOptions` defaults. Brings the total test count to 227 tests across 14 test files.
    47	- **Ledger plugin documentation** (`docs/plugins.md`) — full Ledger Plugin section covering installation, configuration, `LedgerPluginOptions` reference, and usage examples.
    48	
    49	## [0.2.0] - 2026-03-25
    50	
    51	### Added
    52	
    53	- **Template engine** (`src/engine/`) — pure functions for template rendering:
    54	  - `resolvePartials(template, partials)` — injects `{{> partial}}` blocks
    55	  - `resolveConditionals(template, flags)` — evaluates `{{#if flag}}…{{/if}}` blocks
    56	  - `resolveVariables(template, vars)` — substitutes `{{variable}}` tokens
    57	  - Post-processor passes: `collapseBlankLines`, `ensureBlankLineBeforeHeadings`, `normalizeNewlines`
    58	  - Serializers: `serializeTools`, `serializeToolsList`
    59	- **File I/O layer** (`src/loaders/`) — file discovery and loading:
    60	  - `loadPartials(dir)` — reads all `.md` files from a directory, keyed by filename stem
    61	  - `discoverPersonaYamls(root)` — recursively discovers all `**/*.yaml` persona files
    62	  - `loadMetadata(yamlPath)` — parses YAML into a typed `PersonaMetadata` object
    63	  - `loadContent(mdPath)` — reads a persona Markdown template as a raw string
    64	  - `PersonaMetadata` type
    65	- **Plugin architecture** (`src/plugins/`) — extension interface for the build pipeline:
    66	  - `PersonaBuildPlugin` interface with hooks: `onSuiteInit`, `onBuildContext`, `onPostRender`, `onValidate`, `frontmatterTemplates`
    67	  - `ValidationResult` type (`severity: 'error' | 'warning' | 'info'`, `message: string`)
    68	  - Plugin runner functions: `runSuiteInit`, `runBuildContext`, `runPostRender`, `runValidate` — all invoke hooks in registration order
    69	- **Built-in validators** (`src/validators/`):
    70	  - `validateFileName(filePath)` — enforces kebab-case naming convention; returns `ValidationResult[]` with `severity: 'error'` for violations
    71	  - Strict-marker validator — scans rendered output for unresolved `{{…}}` tokens outside code fences
    72	- **Builder core** (`src/builders/`) — the primary build orchestration layer:
    73	  - `build(config: BuildConfig): Promise<BuildSummary>` — top-level entry point; orchestrates suite discovery, rendering, plugin hooks, validation, and file writes
    74	  - `buildSuite(config, plugins)` — processes all personas in a single suite
    75	  - `buildPersona(personaYamlPath, config, plugins)` — processes a single persona through the full pipeline
    76	  - Frontmatter registry (`src/builders/frontmatter.ts`) with built-in templates for `vscode` and `claude-code` targets; overridable via plugin `frontmatterTemplates`
    77	  - `BuildConfig`, `BuildResult`, `BuildSummary`, `TargetType` types
    78	- **CLI entry point** (`src/cli.ts`) — `persona-build` executable:
    79	  - `--config <path>` — load build config from `.js` (ESM), `.cjs`, or `.json`
    80	  - `--check` — render personas but skip writing output files; always exits 0 alone
    81	  - `--strict` — exit 1 if any `ValidationResult` has severity `error` or `warning`; combine with `--check` for a safe CI dry-run
    82	  - `--help` — print usage and exit 0
    83	  - `--version` — print package version (sourced from `package.json`) and exit 0
    84	- **Public API barrel** (`src/index.ts`) — named exports: `build`, `BuildConfig`, `BuildSummary`, `PersonaBuildPlugin`, `TargetType`, `ValidationResult`, `VERSION`
    85	- `VERSION` export — reads from `package.json` at runtime (single source of truth)
    86	- Integration test suite in `tests/integration/` — calls `build(config)` against `fixtures/` and asserts output files are written with correct content
    87	- Comprehensive `README.md` — installation, quick-start (programmatic + CLI), `BuildConfig` reference, `SuiteConfig` reference, `BuildSummary` reference, CLI flags reference, `PersonaBuildPlugin` interface with three code examples, directory conventions, and template syntax guide
    88	
    89	## [0.1.0] - 2026-03-25
    90	
    91	### Added
    92	
    93	- Initial repository scaffold for `@mistralys/persona-builder` TypeScript library
    94	- `package.json` with `@mistralys/persona-builder` package name, dual CJS + ESM exports, and `persona-build` CLI bin entry
    95	- `js-yaml` as the sole production dependency
    96	- `tsup` build pipeline producing both CJS (`dist/index.cjs`) and ESM (`dist/index.js`) outputs with TypeScript declarations
    97	- `tsconfig.json` with `strict: true`, `ES2022` target, `bundler` module resolution, and `node18` engine target
    98	- `vitest.config.ts` configured for Node environment with `passWithNoTests: true` so the test suite exits cleanly before test files are written
    99	- Placeholder `src/index.ts` exporting `VERSION = "0.1.0"` — future module exports are scaffolded as commented-out barrel re-exports with WP references
   100	- Placeholder `src/cli.ts` documenting all planned CLI flags (`--config`, `--suite`, `--target`, `--check`, `--dry-run`, `--strict`) and exiting cleanly

## Assistant

```json
{
  "id": "toolu_011mmy6zCPzhriDAbsr7mMtP",
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

```json
{
  "id": "toolu_012ZEsiLtWagdiA6ZKz5EZM9",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents"
  },
  "name": "ls",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_011mmy6zCPzhriDAbsr7mMtP`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md"
}
```

**Tool call:** `ls` (id: `toolu_012ZEsiLtWagdiA6ZKz5EZM9`)

```json
{
  "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents"
}
```

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
    38	  onValidate?(persona: PersonaMetadata, suite: SuiteConfig, target?: TargetType): ValidationResult[];
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

## Tool Result

['/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/']

## Assistant

```json
{
  "id": "toolu_01WwNUqw1yDf83PRBrk6nh1a",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
    "offset": 100,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_017EM6zuwpT7g5isNWmrw4xy",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest"
  },
  "name": "ls",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01WwNUqw1yDf83PRBrk6nh1a`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
  "offset": 100,
  "limit": 200
}
```

**Tool call:** `ls` (id: `toolu_017EM6zuwpT7g5isNWmrw4xy`)

```json
{
  "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest"
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
   203	   * Controls the severity emitted when a persona's `role` field is not found
   204	   * in `manifestRoles`.
   205	   *
   206	   * - `true` (default): unknown role → `warning` severity. The build continues
   207	   *   and the issue is surfaced as an advisory.
   208	   * - `false`: unknown role → `error` severity. The build treats the unknown
   209	   *   role as a hard failure.
   210	   *
   211	   * @default true
   212	   */
   213	  warnOnUnknownRole?: boolean;
   214	}
   215	```
   216	
   217	---
   218	
   219	### Validator Severity Escalation Pattern
   220	
   221	The ledger plugin uses a deliberate separation between **validator logic** and **severity policy** that plugin authors can adopt in their own validators.
   222	
   223	**Contract:**
   224	
   225	- Validator functions (e.g. `validateRole`) always return `warning`-severity results. They detect the condition and surface it — they do not decide how serious it is.
   226	- The plugin factory's `onValidate` hook applies severity policy based on plugin options. When `warnOnUnknownRole: false`, every `warning` emitted by `validateRole` is escalated to `error` before the result is returned.
   227	
   228	**Why this pattern?**
   229	
   230	It keeps validator logic pure and independently testable. Validators only answer "did this condition occur?". The factory decides "how bad is it?" based on runtime configuration.
   231	
   232	**Implementation sketch:**
   233	
   234	```ts
   235	onValidate(persona, _suite, _target): ValidationResult[] {
   236	  const raw = validateRole(persona.role, manifestRoles);
   237	  return raw.map(r => ({
   238	    ...r,
   239	    severity: (r.severity === 'warning' && !warnOnUnknownRole)
   240	      ? 'error'
   241	      : r.severity,
   242	  }));
   243	}
   244	```
   245	
   246	Plugin authors can apply the same pattern to any option that should toggle an advisory into a hard failure:
   247	
   248	```ts
   249	// Validator always returns 'warning'
   250	function validateCustomField(value: unknown): ValidationResult[] {
   251	  if (!value) {
   252	    return [{ severity: 'warning', message: 'customField is missing' }];
   253	  }
   254	  return [];
   255	}
   256	
   257	// Factory escalates based on options
   258	const raw = validateCustomField(persona.customField);
   259	const results = raw.map(r => ({
   260	  ...r,
   261	  severity: (r.severity === 'warning' && options.strictCustomField)
   262	    ? 'error'
   263	    : r.severity,
   264	}));
   265	```
   266	
   267	---
   268	
   269	### Cache Key Conventions for Multi-Target Builds
   270	
   271	When a plugin caches per-persona state (e.g. rendered output) for use across
   272	hooks, the cache key **must** include the build target to prevent collisions
   273	in multi-target builds. Use the composite key pattern:
   274	
   275	```ts
   276	const cacheKey = `${persona.name}:${target}`;
   277	```
   278	
   279	In `onValidate`, the `target` parameter is optional (`target?: TargetType`).
   280	When absent (e.g. in unit-test contexts where `onValidate` may be called
   281	directly), fall back to a sentinel value using nullish coalescing:
   282	
   283	```ts
   284	const cacheKey = `${persona.name}:${target ?? 'unknown'}`;
   285	const cached = renderedOutputCache.get(cacheKey) ?? '';
   286	```
   287	
   288	**Why `??` and not `||`?** Nullish coalescing (`??`) only triggers on
   289	`null`/`undefined`, preserving any falsy-but-valid value. While `TargetType`
   290	is currently always a non-empty string, the `??` pattern is semantically
   291	correct and defensive against future type changes.
   292	
   293	---
   294	
   295	### Exported types
   296	
   297	#### `RosterEntry`
   298	
   299	```ts
   300	interface RosterEntry {

## Tool Result

['/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/README.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/constraints.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/data-flows.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/tech-stack.md']

## Assistant

```json
{
  "id": "toolu_01Th9YEXPtV25Seghh8KutoG",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
    "offset": 300,
    "limit": 200
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_013XA7Pduvqziv8cXLRShTPc",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01Th9YEXPtV25Seghh8KutoG`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
  "offset": 300,
  "limit": 200
}
```

**Tool call:** `read_file` (id: `toolu_013XA7Pduvqziv8cXLRShTPc`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md"
}
```

## Tool Result

   301	  /** Sequential agent number (1-based) */
   302	  number: number;
   303	  /** Full display title for this agent role */
   304	  title: string;
   305	  /** Short description / label shown in parentheses */
   306	  short: string;
   307	}
   308	```
   309	
   310	Represents one entry in the ledger agent roster as declared in `_shared.yaml`.
   311	
   312	#### `McpToolEntry`
   313	
   314	```ts
   315	interface McpToolEntry {
   316	  /** Tool identifier as used in the MCP server */
   317	  tool: string;
   318	  /** Human-readable description of what the tool does */
   319	  purpose: string;
   320	  /**
   321	   * When true, this entry appears in documentation notes only and must NOT
   322	   * appear in the rendered persona output.
   323	   */
   324	  note_only?: boolean;
   325	}
   326	```
   327	
   328	Represents one MCP tool entry from the persona YAML `mcp_tools` field.
   329	
   330	---
   331	
   332	### Exported renderer functions
   333	
   334	#### `renderRoster(roster, activeNumber)`
   335	
   336	```ts
   337	function renderRoster(roster: RosterEntry[], activeNumber: number): string
   338	```
   339	
   340	Renders the agent roster as a numbered Markdown list. Each entry is formatted as `{number}. **{title}[ (YOU)]** ({short})`. The `(YOU)` suffix is appended to the entry whose `number` matches `activeNumber`.
   341	
   342	```ts
   343	renderRoster([
   344	  { number: 1, title: 'Planner', short: 'plans the work' },
   345	  { number: 2, title: 'Developer', short: 'writes code' },
   346	], 1)
   347	// => "1. **Planner (YOU)** (plans the work)\n2. **Developer** (writes code)"
   348	```
   349	
   350	#### `renderMcpToolsTable(tools)`
   351	
   352	```ts
   353	function renderMcpToolsTable(tools: McpToolEntry[]): string
   354	```
   355	
   356	Renders the MCP tools array as Markdown table rows (`| \`tool\` | purpose |`). Entries with `note_only: true` are filtered out and will not appear in the output.
   357	
   358	```ts
   359	renderMcpToolsTable([
   360	  { tool: 'ledger_get_status', purpose: 'Read project status' },
   361	  { tool: 'internal_tool',    purpose: 'Internal use only', note_only: true },
   362	])
   363	// => "| `ledger_get_status` | Read project status |"
   364	```
   365	
   366	---
   367	
   368	### Exported validator functions
   369	
   370	#### `validateRole(role, manifestRoles)`
   371	
   372	```ts
   373	function validateRole(
   374	  role: string | undefined,
   375	  manifestRoles: ReadonlyArray<string> | ReadonlySet<string>,
   376	): ValidationResult[]
   377	```
   378	
   379	Checks that a persona's `role` field is present in the workflow manifest. Returns an empty array when the role is valid or absent (non-ledger personas have no `role` field), and a single `warning`-level result when the role is not found.
   380	
   381	```ts
   382	validateRole('Developer', ['Planner', 'Developer', 'QA'])
   383	// => []
   384	
   385	validateRole('Coder', ['Planner', 'Developer', 'QA'])
   386	// => [{ severity: 'warning', message: 'Role "Coder" is not in the workflow manifest. Known roles: ...' }]
   387	
   388	validateRole(undefined, ['Planner', 'Developer'])
   389	// => []  — role absent, not a ledger persona
   390	```
   391	
   392	> `manifestRoles` is normalised to a `Set` internally, so both `Array` and `Set` inputs are accepted with O(1) lookup behaviour.
   393	
   394	#### `validateNoteOnlyGuard(output, mcpTools)`
   395	
   396	```ts
   397	function validateNoteOnlyGuard(
   398	  output: string,
   399	  mcpTools: ReadonlyArray<McpToolEntry> | undefined,
   400	): ValidationResult[]
   401	```
   402	
   403	Second-line defence against `note_only: true` tools leaking into published persona output. Even if `renderMcpToolsTable` is bypassed, this guard detects any note-only tool name appearing as a Markdown table cell (`| \`toolName\` |`) in the rendered string and returns one `error`-level result per violation.
   404	
   405	```ts
   406	validateNoteOnlyGuard('| `internal_tool` | does stuff |', [
   407	  { tool: 'internal_tool', purpose: 'Internal', note_only: true },
   408	])
   409	// => [{ severity: 'error', message: 'note_only tool "internal_tool" appears in rendered output.' }]
   410	```
   411	
   412	> The `onValidate` hook supplies both arguments via its closure over the plugin's `renderedOutputCache`. Tool names containing regex special characters (`.`, `+`, `*`, etc.) are correctly escaped before pattern matching.
   413	
   414	---
   415	
   416	### Exported frontmatter constants
   417	
   418	#### `FRONTMATTER_LEDGER_VSCODE`
   419	
   420	```ts
   421	const FRONTMATTER_LEDGER_VSCODE: string
   422	```
   423	
   424	VS Code frontmatter template for the ledger persona suite. Resolved template variables: `{{id}}`, `{{number}}`, `{{role}}`, `{{version}}`, `{{total}}`, `{{model}}`, `{{author}}`, `{{last_updated}}`, `{{vs_file_name}}`, `{{tools_json}}`.
   425	
   426	#### `FRONTMATTER_LEDGER_CC`
   427	
   428	```ts
   429	const FRONTMATTER_LEDGER_CC: string
   430	```
   431	
   432	Claude Code frontmatter template for the ledger persona suite. Resolved template variables: `{{cc_name}}`, `{{cc_description}}`, `{{role}}`, `{{author}}`, `{{version}}`, `{{last_updated}}`, `{{cc_tools_json}}`, `{{cc_permission_mode}}`, `{{cc_model}}`, `{{cc_memory}}`, `{{mcp_server_name}}`. The `mcpServers` block is conditionally included via `{{#if has_mcp}}`.
   433	
   434	Both templates are structurally identical to the originals in `build-personas.js` and are registered automatically by `ledgerPlugin()` via the `frontmatterTemplates` map, overriding the library's built-in defaults for the ledger suite.

## Tool Result

     1	# Public API Surface
     2	
     3	All public symbols are exported from the package entry point `@mistralys/persona-builder` (via `src/index.ts`).
     4	
     5	---
     6	
     7	## Constants
     8	
     9	### `VERSION`
    10	
    11	```ts
    12	export const VERSION: string;
    13	```
    14	
    15	Package version string sourced from `package.json` at runtime via `createRequire`.
    16	
    17	---
    18	
    19	## Top-Level Functions
    20	
    21	### `build(config)`
    22	
    23	```ts
    24	export async function build(config: BuildConfig): Promise<BuildSummary>;
    25	```
    26	
    27	Main entry point. Iterates all suites × targets, orchestrates the full pipeline (discover → load → render → validate → write), and returns an aggregated summary. Respects `check` (no writes) and `strict` (fail on warnings/errors) flags.
    28	
    29	### `buildSuite(suiteName, suiteConfig, config, plugins)`
    30	
    31	```ts
    32	export async function buildSuite(
    33	  suiteName: string,
    34	  suiteConfig: SuiteConfig,
    35	  config: BuildConfig,
    36	  plugins: PersonaBuildPlugin[],
    37	): Promise<BuildResult[]>;
    38	```
    39	
    40	Builds all personas in a single suite for all configured targets. Loads `_shared.yaml`, merges partials, fires `onSuiteInit`, discovers persona YAMLs, and delegates to `buildPersona()`.
    41	
    42	### `buildPersona(personaYamlPath, suiteName, suiteConfig, sharedMeta, partialsMap, config, plugins, target)`
    43	
    44	```ts
    45	export async function buildPersona(
    46	  personaYamlPath: string,
    47	  suiteName: string,
    48	  suiteConfig: SuiteConfig,
    49	  sharedMeta: Record<string, unknown>,
    50	  partialsMap: Record<string, string>,
    51	  config: BuildConfig,
    52	  plugins: PersonaBuildPlugin[],
    53	  target: 'vscode' | 'claude-code',
    54	): Promise<BuildResult>;
    55	```
    56	
    57	Builds a single persona for a single target. Runs the full rendering pipeline: load metadata → build context → plugin hooks → frontmatter → body rendering → post-processing → validation → write.
    58	
    59	---
    60	
    61	## Engine Functions
    62	
    63	All engine functions are **pure** — zero imports, no side effects, no file I/O.
    64	
    65	### `resolvePartials(text, partialsMap, depth?)`
    66	
    67	```ts
    68	export function resolvePartials(
    69	  text: string,
    70	  partialsMap: Record<string, string>,
    71	  depth?: number,
    72	): string;
    73	```
    74	
    75	Replaces `{{> name}}` markers with content from `partialsMap`. Recursion capped at depth 2. Missing partials emit `console.warn` and are preserved as-is.
    76	
    77	### `resolveConditionals(text, context)`
    78	
    79	```ts
    80	export function resolveConditionals(
    81	  text: string,
    82	  context: Record<string, unknown>,
    83	): string;
    84	```
    85	
    86	Evaluates `{{#if flag}}…{{/if}}` and `{{#if flag}}…{{else}}…{{/if}}` blocks. Unknown flags treated as falsy.
    87	
    88	### `resolveVariables(text, context, filename)`
    89	
    90	```ts
    91	export function resolveVariables(
    92	  text: string,
    93	  context: Record<string, unknown>,
    94	  filename: string,
    95	): string;
    96	```
    97	
    98	Substitutes `{{varName}}` tokens with `String(context[varName])`. Unresolved variables emit `console.warn` and are preserved.
    99	
   100	### `collapseBlankLines(text)`

## Assistant

```json
{
  "id": "toolu_017QNAZW1bRCvuWCCsTcRCXB",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md",
    "offset": 100,
    "limit": 300
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_018j6FtUdsFzy42HtQuCcGam",
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

**Tool call:** `read_file` (id: `toolu_017QNAZW1bRCvuWCCsTcRCXB`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md",
  "offset": 100,
  "limit": 300
}
```

**Tool call:** `read_file` (id: `toolu_018j6FtUdsFzy42HtQuCcGam`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md"
}
```

## Tool Result

   101	
   102	```ts
   103	export function collapseBlankLines(text: string): string;
   104	```
   105	
   106	Collapses 3+ consecutive blank lines into 2.
   107	
   108	### `ensureBlankLineBeforeHeadings(text)`
   109	
   110	```ts
   111	export function ensureBlankLineBeforeHeadings(text: string): string;
   112	```
   113	
   114	Inserts a blank line before Markdown headings and horizontal rules when missing.
   115	
   116	### `normalizeNewlines(text)`
   117	
   118	```ts
   119	export function normalizeNewlines(text: string): string;
   120	```
   121	
   122	Converts CRLF/CR to LF.
   123	
   124	### `serializeTools(tools)`
   125	
   126	```ts
   127	export function serializeTools(tools: string[]): string;
   128	```
   129	
   130	Returns YAML flow-sequence with outer brackets: `['tool1', 'tool2']`.
   131	
   132	### `serializeToolsList(tools)`
   133	
   134	```ts
   135	export function serializeToolsList(tools: string[]): string;
   136	```
   137	
   138	Returns comma-separated quoted tool names without brackets: `'tool1', 'tool2'`.
   139	
   140	---
   141	
   142	## Loader Functions
   143	
   144	All loaders perform async file I/O via `node:fs/promises`.
   145	
   146	### `loadPartials(dir)`
   147	
   148	```ts
   149	export async function loadPartials(dir: string): Promise<Record<string, string>>;
   150	```
   151	
   152	Reads all `.md` files in `dir` and returns a map from filename stem to content string.
   153	
   154	### `discoverPersonaYamls(root)`
   155	
   156	```ts
   157	export async function discoverPersonaYamls(root: string): Promise<string[]>;
   158	```
   159	
   160	Recursively discovers all `*.yaml` files under `root`. Returns sorted absolute paths. Uses `readdir({ recursive: true })` (Node ≥ 18.17).
   161	
   162	### `loadMetadata(yamlPath)`
   163	
   164	```ts
   165	export async function loadMetadata(yamlPath: string): Promise<PersonaMetadata>;
   166	```
   167	
   168	Parses a YAML file into a typed `PersonaMetadata` object. Throws if the file is not a valid object or is missing the required `name` field.
   169	
   170	### `loadContent(mdPath)`
   171	
   172	```ts
   173	export async function loadContent(mdPath: string): Promise<string>;
   174	```
   175	
   176	Reads a Markdown content template as a raw UTF-8 string. No parsing or template resolution.
   177	
   178	---
   179	
   180	## Utility Functions
   181	
   182	### `escapeRegExp(str)`
   183	
   184	```ts
   185	export function escapeRegExp(str: string): string;
   186	```
   187	
   188	Escapes all regex special characters in `str` for safe use inside a `new RegExp(...)` constructor. Pure function — no I/O, no side effects.
   189	
   190	---
   191	
   192	## Validator Functions
   193	
   194	Both validators are pure functions — no I/O, no side effects.
   195	
   196	### `validateFileName(filePath)`
   197	
   198	```ts
   199	export function validateFileName(filePath: string): ValidationResult[];
   200	```
   201	
   202	Validates a filename against kebab-case naming convention. Returns one `ValidationResult` (severity `'error'`) per violated rule. Rules: no uppercase, no spaces, kebab-case segments only.
   203	
   204	### `validateStrictMarkers(renderedContent, requiredMarkers)`
   205	
   206	```ts
   207	export function validateStrictMarkers(
   208	  renderedContent: string,
   209	  requiredMarkers: string[],
   210	): ValidationResult[];
   211	```
   212	
   213	Checks that every marker in `requiredMarkers` appears verbatim in `renderedContent`. Returns one error per missing marker.
   214	
   215	---
   216	
   217	## Frontmatter Functions
   218	
   219	### `resolveFrontmatterTemplate(target, plugins, configTemplates?)`
   220	
   221	```ts
   222	export function resolveFrontmatterTemplate(
   223	  target: 'vscode' | 'claude-code',
   224	  plugins: PersonaBuildPlugin[],
   225	  configTemplates?: Partial<Record<'vscode' | 'claude-code', string>>,
   226	): string;
   227	```
   228	
   229	Resolves the frontmatter template for a target. Precedence: plugin `frontmatterTemplates` (first plugin wins) → config-level templates → library defaults.
   230	
   231	### `renderFrontmatter(template, context, filename)`
   232	
   233	```ts
   234	export function renderFrontmatter(
   235	  template: string,
   236	  context: Record<string, unknown>,
   237	  filename: string,
   238	): string;
   239	```
   240	
   241	Renders a frontmatter template string by applying conditionals then variable substitution.
   242	
   243	### `DEFAULT_FRONTMATTER_VSCODE`
   244	
   245	```ts
   246	export const DEFAULT_FRONTMATTER_VSCODE: string;
   247	```
   248	
   249	Built-in VS Code frontmatter template (`name`, `description`, `tools`).
   250	
   251	### `DEFAULT_FRONTMATTER_CLAUDE_CODE`
   252	
   253	```ts
   254	export const DEFAULT_FRONTMATTER_CLAUDE_CODE: string;
   255	```
   256	
   257	Built-in Claude Code frontmatter template (`name`, `permissionMode`, `model`, `memory`, `allowedTools`).
   258	
   259	---
   260	
   261	## Plugin Runner Functions
   262	
   263	All runner functions are synchronous.
   264	
   265	### `runSuiteInit(plugins, suite, sharedMeta)`
   266	
   267	```ts
   268	export function runSuiteInit(
   269	  plugins: PersonaBuildPlugin[],
   270	  suite: SuiteConfig,
   271	  sharedMeta: Record<string, unknown>,
   272	): void;
   273	```
   274	
   275	Invokes `onSuiteInit` on each plugin in order. `sharedMeta` is mutable (passed by reference).
   276	
   277	### `runBuildContext(plugins, ctx, persona, suite)`
   278	
   279	```ts
   280	export function runBuildContext(
   281	  plugins: PersonaBuildPlugin[],
   282	  ctx: Record<string, unknown>,
   283	  persona: PersonaMetadata,
   284	  suite: SuiteConfig,
   285	): Record<string, unknown>;
   286	```
   287	
   288	Accumulating hook — each plugin receives the previous plugin's returned context. Returns the final context.
   289	
   290	### `runPostRender(plugins, rendered, persona, target)`
   291	
   292	```ts
   293	export function runPostRender(
   294	  plugins: PersonaBuildPlugin[],
   295	  rendered: string,
   296	  persona: PersonaMetadata,
   297	  target: TargetType,
   298	): string;
   299	```
   300	
   301	Accumulating hook — each plugin receives the previous plugin's returned output string.
   302	
   303	### `runValidate(plugins, persona, suite)`
   304	
   305	```ts
   306	export function runValidate(
   307	  plugins: PersonaBuildPlugin[],
   308	  persona: PersonaMetadata,
   309	  suite: SuiteConfig,
   310	): ValidationResult[];
   311	```
   312	
   313	Collecting hook — concatenates all `ValidationResult[]` from all plugins into a flat array.
   314	
   315	---
   316	
   317	## Types
   318	
   319	### `BuildConfig`
   320	
   321	```ts
   322	export interface BuildConfig {
   323	  suites: Record<string, SuiteConfig>;
   324	  sharedPartialsDir?: string;
   325	  plugins?: PersonaBuildPlugin[];
   326	  targets?: Array<'vscode' | 'claude-code'>;
   327	  check?: boolean;
   328	  strict?: boolean;
   329	  frontmatter?: Partial<Record<'vscode' | 'claude-code', string>>;
   330	}
   331	```
   332	
   333	### `SuiteConfig`
   334	
   335	```ts
   336	export interface SuiteConfig {
   337	  srcDir: string;
   338	  outVscode: string;
   339	  outClaudeCode: string;
   340	  personaMode?: string;
   341	  partialsSubdir?: string;   // default: 'partials'
   342	  metaSubdir?: string;       // default: 'meta'
   343	  contentSubdir?: string;    // default: 'content'
   344	}
   345	```
   346	
   347	### `BuildResult`
   348	
   349	```ts
   350	export interface BuildResult {
   351	  suite: string;
   352	  target: 'vscode' | 'claude-code';
   353	  personaYamlPath: string;
   354	  outputPath: string;
   355	  content: string;
   356	  validationResults: ValidationResult[];
   357	  written: boolean;
   358	}
   359	```
   360	
   361	### `BuildSummary`
   362	
   363	```ts
   364	export interface BuildSummary {
   365	  success: boolean;
   366	  results: BuildResult[];
   367	  strictFailures: ValidationResult[];
   368	  totalBuilt: number;
   369	  totalWritten: number;
   370	}
   371	```
   372	
   373	### `PersonaMetadata`
   374	
   375	```ts
   376	export interface PersonaMetadata {
   377	  name: string;
   378	  displayName?: string;
   379	  description?: string;
   380	  version?: string;
   381	  tools?: string[];
   382	  [key: string]: unknown;
   383	}
   384	```
   385	
   386	### `PersonaBuildPlugin`
   387	
   388	```ts
   389	export interface PersonaBuildPlugin {
   390	  name: string;
   391	  onSuiteInit?(suite: SuiteConfig, sharedMeta: Record<string, unknown>): void;
   392	  onBuildContext?(
   393	    context: Record<string, unknown>,
   394	    persona: PersonaMetadata,
   395	    suite: SuiteConfig,
   396	  ): Record<string, unknown>;
   397	  onPostRender?(output: string, persona: PersonaMetadata, target: TargetType): string;
   398	  onValidate?(persona: PersonaMetadata, suite: SuiteConfig, target?: TargetType): ValidationResult[];
   399	  frontmatterTemplates?: Partial<Record<TargetType, string>>;
   400	}

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
    48	│   ├── validators/                 # Validation functions
    49	│   │   ├── index.ts                # Barrel re-export
    50	│   │   ├── filename-validator.ts   # Kebab-case filename validation
    51	│   │   └── strict-validator.ts     # Required-marker presence validation
    52	│   │
    53	│   └── utils/                      # Shared utility functions
    54	│       ├── index.ts                # Barrel re-export
    55	│       └── regex.ts                # escapeRegExp() — safe RegExp string escaping
    56	│
    57	├── tests/
    58	│   ├── README.md                   # Test suite documentation
    59	│   ├── engine/                     # Engine module tests (74 tests)
    60	│   │   ├── partials.test.ts
    61	│   │   ├── conditionals.test.ts
    62	│   │   ├── variables.test.ts
    63	│   │   ├── postProcessor.test.ts
    64	│   │   └── serializer.test.ts
    65	│   ├── loaders/                    # Loader tests (40 tests)
    66	│   │   ├── partials-loader.test.ts
    67	│   │   ├── metadata-loader.test.ts
    68	│   │   └── content-loader.test.ts
    69	│   ├── plugins/                    # Plugin system tests (27 tests)
    70	│   │   └── plugin-runner.test.ts
    71	│   ├── builders/                   # Builder tests (33 tests)
    72	│   │   ├── persona-builder.test.ts
    73	│   │   └── persona-builder-edge-cases.test.ts
    74	│   ├── validators/                 # Validator tests (46 tests)
    75	│   │   ├── filename-validator.test.ts
    76	│   │   └── strict-validator.test.ts
    77	│   └── integration/                # End-to-end integration tests (7 tests)
    78	│       └── build.test.ts
    79	│
    80	├── fixtures/                       # Test fixtures
    81	│   ├── sample-suite/
    82	│   │   ├── meta/
    83	│   │   │   ├── _shared.yaml        # Suite-level shared defaults
    84	│   │   │   └── example-persona.yaml
    85	│   │   ├── content/
    86	│   │   │   └── example-persona.md  # Markdown content template
    87	│   │   └── partials/
    88	│   │       └── suite-specific.md   # Suite-local partial
    89	│   └── shared/
    90	│       └── partials/
    91	│           └── greeting.md         # Cross-suite shared partial
    92	│
    93	├── dist/                           # Build output (gitignored)
    94	└── docs/
    95	    └── agents/
    96	        └── project-manifest/       # This manifest
    97	```

## Assistant

```json
{
  "id": "toolu_01NboNCFsjW8thg29HpE9Fvn",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/data-flows.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_018R12r2N7wxVmwHHRyWyGnQ",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md",
    "offset": 400,
    "limit": 100
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01NboNCFsjW8thg29HpE9Fvn`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/data-flows.md"
}
```

**Tool call:** `read_file` (id: `toolu_018R12r2N7wxVmwHHRyWyGnQ`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md",
  "offset": 400,
  "limit": 100
}
```

## Tool Result

     1	# Key Data Flows
     2	
     3	## 1. Full Build Pipeline (`build()`)
     4	
     5	The main `build(config)` entry point orchestrates the entire pipeline:
     6	
     7	```
     8	build(config)
     9	  │
    10	  ├─ For each suite in config.suites:
    11	  │     │
    12	  │     ├─ buildSuite(suiteName, suiteConfig, config, plugins)
    13	  │     │     │
    14	  │     │     ├─ Load _shared.yaml → sharedMeta
    15	  │     │     ├─ Load partials (shared → suite-local overlay) → partialsMap
    16	  │     │     ├─ Run onSuiteInit hooks on all plugins
    17	  │     │     ├─ Discover persona YAML files (meta/*.yaml, excluding _*.yaml)
    18	  │     │     │
    19	  │     │     └─ For each persona × each target:
    20	  │     │           │
    21	  │     │           └─ buildPersona(yamlPath, …, target)
    22	  │     │                 │
    23	  │     │                 ├─ 1. Load persona YAML → personaMeta
    24	  │     │                 ├─ 2. Merge context (sharedMeta + personaMeta + derived fields)
    25	  │     │                 ├─ 3. Run onBuildContext hooks (context accumulation)
    26	  │     │                 ├─ 4. Resolve frontmatter template (plugin → config → default)
    27	  │     │                 ├─ 5. Render frontmatter (conditionals → variables)
    28	  │     │                 ├─ 6. Load content template (.md file)
    29	  │     │                 ├─ 7. Render body:
    30	  │     │                 │     ├─ resolvePartials (depth-2 recursion)
    31	  │     │                 │     ├─ resolveConditionals
    32	  │     │                 │     ├─ resolveVariables
    33	  │     │                 │     ├─ collapseBlankLines
    34	  │     │                 │     └─ ensureBlankLineBeforeHeadings
    35	  │     │                 ├─ 8. Assemble output (frontmatter + body)
    36	  │     │                 ├─ 9. Run onPostRender hooks (output chain)
    37	  │     │                 ├─ 10. Run onValidate hooks (collect ValidationResults)
    38	  │     │                 ├─ 11. Determine output path (vs_file_name / cc_file_name)
    39	  │     │                 └─ 12. Write file (unless check mode)
    40	  │     │
    41	  │     └─ Collect BuildResult[]
    42	  │
    43	  ├─ Aggregate results → BuildSummary
    44	  ├─ If strict: check for error/warning ValidationResults → strictFailures
    45	  └─ Return BuildSummary
    46	```
    47	
    48	## 2. Context Merge Order
    49	
    50	Template variables are resolved from a merged context object. Later values win:
    51	
    52	```
    53	1. _shared.yaml defaults         (suite-level base)
    54	   ↓ overridden by
    55	2. Per-persona YAML fields       (persona-specific values)
    56	   ↓ augmented by
    57	3. Derived convenience fields    (version, tools_list, cc_file_name_stem, etc.)
    58	   ↓ augmented by
    59	4. Plugin onBuildContext hooks    (each plugin mutates/extends context)
    60	```
    61	
    62	### Derived Fields (auto-computed)
    63	
    64	| Field | Source |
    65	|-------|--------|
    66	| `version` | `personaMeta.version` → `sharedMeta.default_version` → `'0.0.0'` |
    67	| `tools_list` | `serializeToolsList(tools)` |
    68	| `tools_json` | `serializeTools(tools)` |
    69	| `cc_tools_list` | `serializeToolsList(cc_tools ?? tools)` |
    70	| `cc_tools_json` | `serializeTools(cc_tools ?? tools)` |
    71	| `cc_file_name_stem` | `cc_file_name` with `.md` extension stripped |
    72	
    73	Derived fields are only set when not already present in the merged context — explicit YAML values always win.
    74	
    75	## 3. Frontmatter Template Precedence
    76	
    77	```
    78	Plugin frontmatterTemplates     (first registered plugin with target key wins)
    79	   ↓ fallback
    80	BuildConfig.frontmatter         (config-level override)
    81	   ↓ fallback
    82	Library defaults                (DEFAULT_FRONTMATTER_VSCODE / DEFAULT_FRONTMATTER_CLAUDE_CODE)
    83	```
    84	
    85	## 4. Partials Resolution
    86	
    87	```
    88	Shared partials dir (sharedPartialsDir)
    89	   ↓ merged with (suite-local overrides shared on name collision)
    90	Suite-local partials dir (srcDir/partials/)
    91	   ↓ result
    92	Combined partialsMap
    93	   ↓ used by
    94	resolvePartials(template, partialsMap)  ← depth-2 recursion
    95	```
    96	
    97	## 5. Plugin Hook Execution Order
    98	
    99	Per persona, hooks fire in this order:
   100	

## Tool Result

   401	```
   402	
   403	### `TargetType`
   404	
   405	```ts
   406	export type TargetType = 'vscode' | 'claude-code';
   407	```
   408	
   409	### `ValidationResult`
   410	
   411	```ts
   412	export interface ValidationResult {
   413	  severity: 'error' | 'warning' | 'info';
   414	  message: string;
   415	}
   416	```
   417	
   418	---
   419	
   420	## Ledger Plugin — `@mistralys/persona-builder/plugins/ledger`
   421	
   422	Sub-path export. Not included in the main `@mistralys/persona-builder` barrel.
   423	
   424	### `ledgerPlugin(options?)`
   425	
   426	```ts
   427	export function ledgerPlugin(options?: LedgerPluginOptions): PersonaBuildPlugin;
   428	```
   429	
   430	Factory function that returns a `PersonaBuildPlugin` for the ledger persona suite. The plugin wires `renderRoster`, `renderMcpToolsTable`, `validateRole`, and `validateNoteOnlyGuard` into the standard build hooks. Also registers `FRONTMATTER_LEDGER_VSCODE` and `FRONTMATTER_LEDGER_CC` as target-specific frontmatter templates.
   431	
   432	### `LedgerPluginOptions`
   433	
   434	```ts
   435	export interface LedgerPluginOptions {
   436	  manifestRoles?: ReadonlyArray<string>;
   437	  warnOnUnknownRole?: boolean;  // default: true
   438	}
   439	```
   440	
   441	### `renderRoster(roster, activeNumber)`
   442	
   443	```ts
   444	export function renderRoster(roster: RosterEntry[], activeNumber: number): string;
   445	```
   446	
   447	Renders the agent roster as a numbered Markdown list. Appends `(YOU)` to the entry matching `activeNumber`.
   448	
   449	### `RosterEntry`
   450	
   451	```ts
   452	export interface RosterEntry {
   453	  number: number;
   454	  title: string;
   455	  short: string;
   456	}
   457	```
   458	
   459	### `renderMcpToolsTable(tools)`
   460	
   461	```ts
   462	export function renderMcpToolsTable(tools: McpToolEntry[]): string;
   463	```
   464	
   465	Renders the MCP tools array as Markdown table rows. Entries with `note_only: true` are filtered out.
   466	
   467	### `McpToolEntry`
   468	
   469	```ts
   470	export interface McpToolEntry {
   471	  tool: string;
   472	  purpose: string;
   473	  note_only?: boolean;
   474	}
   475	```
   476	
   477	### `validateRole(role, manifestRoles)`
   478	
   479	```ts
   480	export function validateRole(
   481	  role: string | undefined,
   482	  manifestRoles: ReadonlyArray<string> | ReadonlySet<string>,
   483	): ValidationResult[];
   484	```
   485	
   486	Validates a persona's `role` field against the workflow manifest. Returns an empty array when the role is valid or absent; a single warning when the role is not found.
   487	
   488	### `validateNoteOnlyGuard(output, mcpTools)`
   489	
   490	```ts
   491	export function validateNoteOnlyGuard(
   492	  output: string,
   493	  mcpTools: ReadonlyArray<McpToolEntry> | undefined,
   494	): ValidationResult[];
   495	```
   496	
   497	Asserts that `note_only: true` MCP tools do not appear in the rendered persona output. Returns one error per leaking tool name.
   498	
   499	### `FRONTMATTER_LEDGER_VSCODE`
   500	

## Assistant

```json
{
  "id": "toolu_01DqMGv7SWgDgfRQr3WxAxZW",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md",
    "offset": 500,
    "limit": 100
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01NribQUpramz5HZ2xduouRp",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/data-flows.md",
    "offset": 100,
    "limit": 100
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01DqMGv7SWgDgfRQr3WxAxZW`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md",
  "offset": 500,
  "limit": 100
}
```

**Tool call:** `read_file` (id: `toolu_01NribQUpramz5HZ2xduouRp`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/data-flows.md",
  "offset": 100,
  "limit": 100
}
```

## Tool Result

   501	```ts
   502	export const FRONTMATTER_LEDGER_VSCODE: string;
   503	```
   504	
   505	VS Code frontmatter template for the ledger persona suite.
   506	
   507	### `FRONTMATTER_LEDGER_CC`
   508	
   509	```ts
   510	export const FRONTMATTER_LEDGER_CC: string;
   511	```
   512	
   513	Claude Code frontmatter template for the ledger persona suite (includes conditional `mcpServers` block).

## Tool Result

   101	```
   102	1. onSuiteInit(suite, sharedMeta)          ← once per suite (before any persona)
   103	2. onBuildContext(context, persona, suite)  ← per persona, before rendering
   104	3. onPostRender(output, persona, target)   ← per persona, after rendering
   105	4. onValidate(persona, suite, target?)     ← per persona, after post-render
   106	```
   107	
   108	Within each hook, plugins are invoked in **registration order** (array index in `config.plugins`).
   109	
   110	- `onBuildContext` and `onPostRender` are **accumulating** — each plugin receives the prior plugin's output.
   111	- `onValidate` is **collecting** — results are concatenated into a flat array.
   112	
   113	## 6. CLI Flow
   114	
   115	```
   116	persona-build [flags]
   117	  │
   118	  ├─ Parse args (--config, --check, --strict, --help, --version)
   119	  ├─ Load config file (dynamic import: .js ESM / .cjs / .json)
   120	  ├─ Merge CLI flags into BuildConfig
   121	  ├─ Call build(config)
   122	  ├─ Report results to stdout
   123	  └─ Exit code: 0 (success) or 1 (strict failure)
   124	```
   125	
   126	## 7. Output File Naming
   127	
   128	```
   129	For each persona × target:
   130	  1. Check context['vs_file_name'] (VS Code) or context['cc_file_name'] (Claude Code)
   131	  2. If present → use as the output basename
   132	  3. If absent  → fall back to content filename (persona-name.md)
   133	  4. Output path = outputDir / basename
   134	```

## Assistant

I now have a complete picture of all files. Let me also check the current package.json version to use in the CHANGELOG:

```json
{
  "id": "toolu_01E18Z8tnWNXmfK8SjywMJU8",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/package.json",
    "limit": 30
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01E18Z8tnWNXmfK8SjywMJU8`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/package.json",
  "limit": 30
}
```

## Tool Result

     1	{
     2	  "name": "@mistralys/persona-builder",
     3	  "version": "1.0.1",
     4	  "description": "Standalone library for building AI persona documents from YAML metadata and Markdown content templates.",
     5	  "type": "module",
     6	  "main": "./dist/index.cjs",
     7	  "module": "./dist/index.js",
     8	  "types": "./dist/index.d.ts",
     9	  "exports": {
    10	    ".": {
    11	      "types": "./dist/index.d.ts",
    12	      "import": "./dist/index.js",
    13	      "require": "./dist/index.cjs"
    14	    }
    15	  },
    16	  "bin": {
    17	    "persona-build": "./dist/cli.js"
    18	  },
    19	  "files": [
    20	    "dist"
    21	  ],
    22	  "scripts": {
    23	    "build": "tsup",
    24	    "dev": "tsup --watch",
    25	    "test": "vitest run",
    26	    "test:watch": "vitest",
    27	    "typecheck": "tsc --noEmit"
    28	  },
    29	  "dependencies": {
    30	    "js-yaml": "^4.1.0"

## Assistant

I have everything I need. Now let me make all three documentation updates:

```json
{
  "id": "toolu_01DQsGBxPS4KQ2JSjUuXzZYF",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Update docs/plugins.md \u2014 remove ledger plugin section, add migration note",
        "status": "in_progress"
      },
      {
        "content": "Update api-surface.md \u2014 remove ledger plugin section",
        "status": "pending"
      },
      {
        "content": "Update file-tree.md \u2014 remove ledger plugin entries",
        "status": "pending"
      },
      {
        "content": "Update data-flows.md \u2014 remove ledger-specific references if any",
        "status": "pending"
      },
      {
        "content": "Update CHANGELOG.md \u2014 add 2.0.0 breaking change entry",
        "status": "pending"
      },
      {
        "content": "Complete documentation pipeline in ledger",
        "status": "pending"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_01DQsGBxPS4KQ2JSjUuXzZYF`)

```json
{
  "todos": [
    {
      "content": "Update docs/plugins.md \u2014 remove ledger plugin section, add migration note",
      "status": "in_progress"
    },
    {
      "content": "Update api-surface.md \u2014 remove ledger plugin section",
      "status": "pending"
    },
    {
      "content": "Update file-tree.md \u2014 remove ledger plugin entries",
      "status": "pending"
    },
    {
      "content": "Update data-flows.md \u2014 remove ledger-specific references if any",
      "status": "pending"
    },
    {
      "content": "Update CHANGELOG.md \u2014 add 2.0.0 breaking change entry",
      "status": "pending"
    },
    {
      "content": "Complete documentation pipeline in ledger",
      "status": "pending"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Update docs/plugins.md — remove ledger plugin section, add migration note', 'status': 'in_progress'}, {'content': 'Update api-surface.md — remove ledger plugin section', 'status': 'pending'}, {'content': 'Update file-tree.md — remove ledger plugin entries', 'status': 'pending'}, {'content': 'Update data-flows.md — remove ledger-specific references if any', 'status': 'pending'}, {'content': 'Update CHANGELOG.md — add 2.0.0 breaking change entry', 'status': 'pending'}, {'content': 'Complete documentation pipeline in ledger', 'status': 'pending'}]

## Assistant

Now let me update `docs/plugins.md` — remove the full Ledger Plugin section (lines 119–433) and replace it with a migration note:

```json
{
  "id": "toolu_01KQVSi8QFnjwn5cWCNYetX6",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
    "old_string": "---\n\n## Ledger Plugin \u2014 `@mistralys/persona-builder/plugins/ledger`\n\nThe ledger plugin is a first-party plugin shipped as a **sub-path export** from the library. It wires ledger-specific rendering and validation into the standard build hooks and is the recommended way to build the ai-insights ledger persona suite.\n\n### Import\n\n```ts\n// ESM\nimport { ledgerPlugin } from '@mistralys/persona-builder/plugins/ledger';\n\n// CJS\nconst { ledgerPlugin } = require('@mistralys/persona-builder/plugins/ledger');\n```\n\nThe sub-path resolves to `dist/plugins/ledger/index.{js,cjs,d.ts}` \u2014 all three artefacts (ESM, CJS, DTS) are included in the package.",
    "new_string": "---\n\n## Ledger Plugin \u2014 Migrated\n\n> **Moved in v2.0.0** \u2014 The ledger plugin has been removed from this library and migrated to the\n> [ai-insights-dev](https://github.com/mistralys/ai-insights-dev) workspace as a local CommonJS\n> module at `personas/plugins/ledger/`. The `@mistralys/persona-builder/plugins/ledger` sub-path\n> export no longer exists. See the ai-insights-dev repository for current ledger plugin source and\n> documentation."
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_01KQVSi8QFnjwn5cWCNYetX6`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
  "old_string": "---\n\n## Ledger Plugin \u2014 `@mistralys/persona-builder/plugins/ledger`\n\nThe ledger plugin is a first-party plugin shipped as a **sub-path export** from the library. It wires ledger-specific rendering and validation into the standard build hooks and is the recommended way to build the ai-insights ledger persona suite.\n\n### Import\n\n```ts\n// ESM\nimport { ledgerPlugin } from '@mistralys/persona-builder/plugins/ledger';\n\n// CJS\nconst { ledgerPlugin } = require('@mistralys/persona-builder/plugins/ledger');\n```\n\nThe sub-path resolves to `dist/plugins/ledger/index.{js,cjs,d.ts}` \u2014 all three artefacts (ESM, CJS, DTS) are included in the package.",
  "new_string": "---\n\n## Ledger Plugin \u2014 Migrated\n\n> **Moved in v2.0.0** \u2014 The ledger plugin has been removed from this library and migrated to the\n> [ai-insights-dev](https://github.com/mistralys/ai-insights-dev) workspace as a local CommonJS\n> module at `personas/plugins/ledger/`. The `@mistralys/persona-builder/plugins/ledger` sub-path\n> export no longer exists. See the ai-insights-dev repository for current ledger plugin source and\n> documentation."
}
```

## Tool Result

Successfully replaced 1 instance(s) of the string in '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md'

## Assistant

Now remove everything from the `### ledgerPlugin(options?)` heading through to the end of the file (which is all the ledger-specific details):

```json
{
  "id": "toolu_019yaKNQ91XU6gbgtGbcZaU9",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
    "offset": 117,
    "limit": 10
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_019yaKNQ91XU6gbgtGbcZaU9`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
  "offset": 117,
  "limit": 10
}
```

## Tool Result

   118	
   119	## Ledger Plugin — Migrated
   120	
   121	> **Moved in v2.0.0** — The ledger plugin has been removed from this library and migrated to the
   122	> [ai-insights-dev](https://github.com/mistralys/ai-insights-dev) workspace as a local CommonJS
   123	> module at `personas/plugins/ledger/`. The `@mistralys/persona-builder/plugins/ledger` sub-path
   124	> export no longer exists. See the ai-insights-dev repository for current ledger plugin source and
   125	> documentation.
   126	
   127	---

## Assistant

Good. Now I need to remove the remaining ledger plugin detail content that starts after `---`. Let me check what follows:

```json
{
  "id": "toolu_01Xt54XaMLqQwf98fG3Tro7J",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
    "offset": 126,
    "limit": 20
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01Xt54XaMLqQwf98fG3Tro7J`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
  "offset": 126,
  "limit": 20
}
```

## Tool Result

   127	---
   128	
   129	### `ledgerPlugin(options?)`
   130	
   131	```ts
   132	function ledgerPlugin(options?: LedgerPluginOptions): PersonaBuildPlugin
   133	```
   134	
   135	Factory function. Returns a fully configured `PersonaBuildPlugin` for the ledger persona suite. The returned plugin object implements:
   136	
   137	| Hook / field | Behaviour |
   138	|---|---|
   139	| `name` | `'ledger'` |
   140	| `onBuildContext` | Injects `roster_rendered` and `mcp_tools_table` into the template context |
   141	| `onPostRender` | Caches the rendered output per-persona for use by `onValidate` |
   142	| `onValidate` | Runs role validation and the `note_only` guard; returns `ValidationResult[]` |
   143	| `frontmatterTemplates` | Registers `FRONTMATTER_LEDGER_VSCODE` (`vscode`) and `FRONTMATTER_LEDGER_CC` (`claude-code`) |
   144	
   145	#### Context variables injected by `onBuildContext`
   146	

## Assistant

The ledger detail content starts at line 127 (`---`) and continues to end of file. I need to remove everything from that `---` onwards. Let me read the very end of the file to understand the full extent:

```json
{
  "id": "toolu_01CLMQUibwRmeYq9xzshT9RY",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
    "offset": 420,
    "limit": 20
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01CLMQUibwRmeYq9xzshT9RY`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
  "offset": 420,
  "limit": 20
}
```

## Tool Result

   421	const FRONTMATTER_LEDGER_CC: string
   422	```
   423	
   424	Claude Code frontmatter template for the ledger persona suite. Resolved template variables: `{{cc_name}}`, `{{cc_description}}`, `{{role}}`, `{{author}}`, `{{version}}`, `{{last_updated}}`, `{{cc_tools_json}}`, `{{cc_permission_mode}}`, `{{cc_model}}`, `{{cc_memory}}`, `{{mcp_server_name}}`. The `mcpServers` block is conditionally included via `{{#if has_mcp}}`.
   425	
   426	Both templates are structurally identical to the originals in `build-personas.js` and are registered automatically by `ledgerPlugin()` via the `frontmatterTemplates` map, overriding the library's built-in defaults for the ledger suite.

## Assistant

The file ends at line 426. I need to remove everything from the `---` separator (line 127) to the end. I'll do this with an edit that replaces from the separator onwards with nothing:

```json
{
  "id": "toolu_01NuTF7zLVPybt8xvkKreV3N",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
    "offset": 124,
    "limit": 5
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01NuTF7zLVPybt8xvkKreV3N`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
  "offset": 124,
  "limit": 5
}
```

## Tool Result

   125	> documentation.
   126	
   127	---
   128	
   129	### `ledgerPlugin(options?)`

## Assistant

```json
{
  "id": "toolu_01CCTem3rWP8KKb2dJWAw8DN",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
    "offset": 422,
    "limit": 10
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01CCTem3rWP8KKb2dJWAw8DN`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
  "offset": 422,
  "limit": 10
}
```

## Tool Result

   423	
   424	Claude Code frontmatter template for the ledger persona suite. Resolved template variables: `{{cc_name}}`, `{{cc_description}}`, `{{role}}`, `{{author}}`, `{{version}}`, `{{last_updated}}`, `{{cc_tools_json}}`, `{{cc_permission_mode}}`, `{{cc_model}}`, `{{cc_memory}}`, `{{mcp_server_name}}`. The `mcpServers` block is conditionally included via `{{#if has_mcp}}`.
   425	
   426	Both templates are structurally identical to the originals in `build-personas.js` and are registered automatically by `ledgerPlugin()` via the `frontmatterTemplates` map, overriding the library's built-in defaults for the ledger suite.

## Assistant

The entire ledger detail section is large, so I'll use a strategy of replacing the `---\n\n### \`ledgerPlugin(options?)\`` ... through the final line. Let me do this by reading the full section first and then making a targeted replacement. Since the file is large, I'll use the approach of replacing from `\n\n---\n\n### \`ledgerPlugin` to the last line:

```json
{
  "id": "toolu_01TuZLA284zRdgA1opKm2YA7",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
    "offset": 126,
    "limit": 300
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01TuZLA284zRdgA1opKm2YA7`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
  "offset": 126,
  "limit": 300
}
```

## Tool Result

   127	---
   128	
   129	### `ledgerPlugin(options?)`
   130	
   131	```ts
   132	function ledgerPlugin(options?: LedgerPluginOptions): PersonaBuildPlugin
   133	```
   134	
   135	Factory function. Returns a fully configured `PersonaBuildPlugin` for the ledger persona suite. The returned plugin object implements:
   136	
   137	| Hook / field | Behaviour |
   138	|---|---|
   139	| `name` | `'ledger'` |
   140	| `onBuildContext` | Injects `roster_rendered` and `mcp_tools_table` into the template context |
   141	| `onPostRender` | Caches the rendered output per-persona for use by `onValidate` |
   142	| `onValidate` | Runs role validation and the `note_only` guard; returns `ValidationResult[]` |
   143	| `frontmatterTemplates` | Registers `FRONTMATTER_LEDGER_VSCODE` (`vscode`) and `FRONTMATTER_LEDGER_CC` (`claude-code`) |
   144	
   145	#### Context variables injected by `onBuildContext`
   146	
   147	| Variable | Type | Value when absent |
   148	|---|---|---|
   149	| `roster_rendered` | `string` | `''` (empty — persona has no `roster` or `number` field) |
   150	| `mcp_tools_table` | `string` | `''` (empty — persona has no `mcp_tools` field) |
   151	
   152	Both keys are **always** set (even to an empty string) so templates can reference `{{roster_rendered}}` and `{{mcp_tools_table}}` on non-ledger personas without triggering an unresolved-variable warning.
   153	
   154	#### Usage
   155	
   156	```js
   157	// personas/persona-build.config.js
   158	const { ledgerPlugin } = require('@mistralys/persona-builder/plugins/ledger');
   159	const manifest = require('../shared/workflow-manifest.json');
   160	
   161	module.exports = {
   162	  rootDir: __dirname,
   163	  sharedPartialsDir: './shared/partials',
   164	  suites: {
   165	    ledger: {
   166	      srcDir: './ledger/src',
   167	      outVscode: './ledger/vs-code',
   168	      outClaudeCode: './ledger/claude-code',
   169	      personaMode: 'numbered',
   170	    },
   171	  },
   172	  plugins: [
   173	    ledgerPlugin({
   174	      manifestRoles: manifest.roles.map(r => r.name),
   175	      warnOnUnknownRole: true,
   176	    }),
   177	  ],
   178	};
   179	```
   180	
   181	---
   182	
   183	### `LedgerPluginOptions`
   184	
   185	```ts
   186	interface LedgerPluginOptions {
   187	  /**
   188	   * List of canonical role names from the project's workflow manifest.
   189	   * Every persona's `role` field is validated against this list.
   190	   * When omitted or empty, role validation is skipped.
   191	   */
   192	  manifestRoles?: ReadonlyArray<string>;
   193	
   194	  /**
   195	   * Controls the severity emitted when a persona's `role` field is not found
   196	   * in `manifestRoles`.
   197	   *
   198	   * - `true` (default): unknown role → `warning` severity. The build continues
   199	   *   and the issue is surfaced as an advisory.
   200	   * - `false`: unknown role → `error` severity. The build treats the unknown
   201	   *   role as a hard failure.
   202	   *
   203	   * @default true
   204	   */
   205	  warnOnUnknownRole?: boolean;
   206	}
   207	```
   208	
   209	---
   210	
   211	### Validator Severity Escalation Pattern
   212	
   213	The ledger plugin uses a deliberate separation between **validator logic** and **severity policy** that plugin authors can adopt in their own validators.
   214	
   215	**Contract:**
   216	
   217	- Validator functions (e.g. `validateRole`) always return `warning`-severity results. They detect the condition and surface it — they do not decide how serious it is.
   218	- The plugin factory's `onValidate` hook applies severity policy based on plugin options. When `warnOnUnknownRole: false`, every `warning` emitted by `validateRole` is escalated to `error` before the result is returned.
   219	
   220	**Why this pattern?**
   221	
   222	It keeps validator logic pure and independently testable. Validators only answer "did this condition occur?". The factory decides "how bad is it?" based on runtime configuration.
   223	
   224	**Implementation sketch:**
   225	
   226	```ts
   227	onValidate(persona, _suite, _target): ValidationResult[] {
   228	  const raw = validateRole(persona.role, manifestRoles);
   229	  return raw.map(r => ({
   230	    ...r,
   231	    severity: (r.severity === 'warning' && !warnOnUnknownRole)
   232	      ? 'error'
   233	      : r.severity,
   234	  }));
   235	}
   236	```
   237	
   238	Plugin authors can apply the same pattern to any option that should toggle an advisory into a hard failure:
   239	
   240	```ts
   241	// Validator always returns 'warning'
   242	function validateCustomField(value: unknown): ValidationResult[] {
   243	  if (!value) {
   244	    return [{ severity: 'warning', message: 'customField is missing' }];
   245	  }
   246	  return [];
   247	}
   248	
   249	// Factory escalates based on options
   250	const raw = validateCustomField(persona.customField);
   251	const results = raw.map(r => ({
   252	  ...r,
   253	  severity: (r.severity === 'warning' && options.strictCustomField)
   254	    ? 'error'
   255	    : r.severity,
   256	}));
   257	```
   258	
   259	---
   260	
   261	### Cache Key Conventions for Multi-Target Builds
   262	
   263	When a plugin caches per-persona state (e.g. rendered output) for use across
   264	hooks, the cache key **must** include the build target to prevent collisions
   265	in multi-target builds. Use the composite key pattern:
   266	
   267	```ts
   268	const cacheKey = `${persona.name}:${target}`;
   269	```
   270	
   271	In `onValidate`, the `target` parameter is optional (`target?: TargetType`).
   272	When absent (e.g. in unit-test contexts where `onValidate` may be called
   273	directly), fall back to a sentinel value using nullish coalescing:
   274	
   275	```ts
   276	const cacheKey = `${persona.name}:${target ?? 'unknown'}`;
   277	const cached = renderedOutputCache.get(cacheKey) ?? '';
   278	```
   279	
   280	**Why `??` and not `||`?** Nullish coalescing (`??`) only triggers on
   281	`null`/`undefined`, preserving any falsy-but-valid value. While `TargetType`
   282	is currently always a non-empty string, the `??` pattern is semantically
   283	correct and defensive against future type changes.
   284	
   285	---
   286	
   287	### Exported types
   288	
   289	#### `RosterEntry`
   290	
   291	```ts
   292	interface RosterEntry {
   293	  /** Sequential agent number (1-based) */
   294	  number: number;
   295	  /** Full display title for this agent role */
   296	  title: string;
   297	  /** Short description / label shown in parentheses */
   298	  short: string;
   299	}
   300	```
   301	
   302	Represents one entry in the ledger agent roster as declared in `_shared.yaml`.
   303	
   304	#### `McpToolEntry`
   305	
   306	```ts
   307	interface McpToolEntry {
   308	  /** Tool identifier as used in the MCP server */
   309	  tool: string;
   310	  /** Human-readable description of what the tool does */
   311	  purpose: string;
   312	  /**
   313	   * When true, this entry appears in documentation notes only and must NOT
   314	   * appear in the rendered persona output.
   315	   */
   316	  note_only?: boolean;
   317	}
   318	```
   319	
   320	Represents one MCP tool entry from the persona YAML `mcp_tools` field.
   321	
   322	---
   323	
   324	### Exported renderer functions
   325	
   326	#### `renderRoster(roster, activeNumber)`
   327	
   328	```ts
   329	function renderRoster(roster: RosterEntry[], activeNumber: number): string
   330	```
   331	
   332	Renders the agent roster as a numbered Markdown list. Each entry is formatted as `{number}. **{title}[ (YOU)]** ({short})`. The `(YOU)` suffix is appended to the entry whose `number` matches `activeNumber`.
   333	
   334	```ts
   335	renderRoster([
   336	  { number: 1, title: 'Planner', short: 'plans the work' },
   337	  { number: 2, title: 'Developer', short: 'writes code' },
   338	], 1)
   339	// => "1. **Planner (YOU)** (plans the work)\n2. **Developer** (writes code)"
   340	```
   341	
   342	#### `renderMcpToolsTable(tools)`
   343	
   344	```ts
   345	function renderMcpToolsTable(tools: McpToolEntry[]): string
   346	```
   347	
   348	Renders the MCP tools array as Markdown table rows (`| \`tool\` | purpose |`). Entries with `note_only: true` are filtered out and will not appear in the output.
   349	
   350	```ts
   351	renderMcpToolsTable([
   352	  { tool: 'ledger_get_status', purpose: 'Read project status' },
   353	  { tool: 'internal_tool',    purpose: 'Internal use only', note_only: true },
   354	])
   355	// => "| `ledger_get_status` | Read project status |"
   356	```
   357	
   358	---
   359	
   360	### Exported validator functions
   361	
   362	#### `validateRole(role, manifestRoles)`
   363	
   364	```ts
   365	function validateRole(
   366	  role: string | undefined,
   367	  manifestRoles: ReadonlyArray<string> | ReadonlySet<string>,
   368	): ValidationResult[]
   369	```
   370	
   371	Checks that a persona's `role` field is present in the workflow manifest. Returns an empty array when the role is valid or absent (non-ledger personas have no `role` field), and a single `warning`-level result when the role is not found.
   372	
   373	```ts
   374	validateRole('Developer', ['Planner', 'Developer', 'QA'])
   375	// => []
   376	
   377	validateRole('Coder', ['Planner', 'Developer', 'QA'])
   378	// => [{ severity: 'warning', message: 'Role "Coder" is not in the workflow manifest. Known roles: ...' }]
   379	
   380	validateRole(undefined, ['Planner', 'Developer'])
   381	// => []  — role absent, not a ledger persona
   382	```
   383	
   384	> `manifestRoles` is normalised to a `Set` internally, so both `Array` and `Set` inputs are accepted with O(1) lookup behaviour.
   385	
   386	#### `validateNoteOnlyGuard(output, mcpTools)`
   387	
   388	```ts
   389	function validateNoteOnlyGuard(
   390	  output: string,
   391	  mcpTools: ReadonlyArray<McpToolEntry> | undefined,
   392	): ValidationResult[]
   393	```
   394	
   395	Second-line defence against `note_only: true` tools leaking into published persona output. Even if `renderMcpToolsTable` is bypassed, this guard detects any note-only tool name appearing as a Markdown table cell (`| \`toolName\` |`) in the rendered string and returns one `error`-level result per violation.
   396	
   397	```ts
   398	validateNoteOnlyGuard('| `internal_tool` | does stuff |', [
   399	  { tool: 'internal_tool', purpose: 'Internal', note_only: true },
   400	])
   401	// => [{ severity: 'error', message: 'note_only tool "internal_tool" appears in rendered output.' }]
   402	```
   403	
   404	> The `onValidate` hook supplies both arguments via its closure over the plugin's `renderedOutputCache`. Tool names containing regex special characters (`.`, `+`, `*`, etc.) are correctly escaped before pattern matching.
   405	
   406	---
   407	
   408	### Exported frontmatter constants
   409	
   410	#### `FRONTMATTER_LEDGER_VSCODE`
   411	
   412	```ts
   413	const FRONTMATTER_LEDGER_VSCODE: string
   414	```
   415	
   416	VS Code frontmatter template for the ledger persona suite. Resolved template variables: `{{id}}`, `{{number}}`, `{{role}}`, `{{version}}`, `{{total}}`, `{{model}}`, `{{author}}`, `{{last_updated}}`, `{{vs_file_name}}`, `{{tools_json}}`.
   417	
   418	#### `FRONTMATTER_LEDGER_CC`
   419	
   420	```ts
   421	const FRONTMATTER_LEDGER_CC: string
   422	```
   423	
   424	Claude Code frontmatter template for the ledger persona suite. Resolved template variables: `{{cc_name}}`, `{{cc_description}}`, `{{role}}`, `{{author}}`, `{{version}}`, `{{last_updated}}`, `{{cc_tools_json}}`, `{{cc_permission_mode}}`, `{{cc_model}}`, `{{cc_memory}}`, `{{mcp_server_name}}`. The `mcpServers` block is conditionally included via `{{#if has_mcp}}`.
   425	
   426	Both templates are structurally identical to the originals in `build-personas.js` and are registered automatically by `ledgerPlugin()` via the `frontmatterTemplates` map, overriding the library's built-in defaults for the ledger suite.

## Assistant

```json
{
  "id": "toolu_01D92tgWg2aryxzy5BcJr97L",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
    "old_string": "---\n\n### `ledgerPlugin(options?)`\n\n```ts\nfunction ledgerPlugin(options?: LedgerPluginOptions): PersonaBuildPlugin\n```\n\nFactory function. Returns a fully configured `PersonaBuildPlugin` for the ledger persona suite. The returned plugin object implements:\n\n| Hook / field | Behaviour |\n|---|---|\n| `name` | `'ledger'` |\n| `onBuildContext` | Injects `roster_rendered` and `mcp_tools_table` into the template context |\n| `onPostRender` | Caches the rendered output per-persona for use by `onValidate` |\n| `onValidate` | Runs role validation and the `note_only` guard; returns `ValidationResult[]` |\n| `frontmatterTemplates` | Registers `FRONTMATTER_LEDGER_VSCODE` (`vscode`) and `FRONTMATTER_LEDGER_CC` (`claude-code`) |\n\n#### Context variables injected by `onBuildContext`\n\n| Variable | Type | Value when absent |\n|---|---|---|\n| `roster_rendered` | `string` | `''` (empty \u2014 persona has no `roster` or `number` field) |\n| `mcp_tools_table` | `string` | `''` (empty \u2014 persona has no `mcp_tools` field) |\n\nBoth keys are **always** set (even to an empty string) so templates can reference `{{roster_rendered}}` and `{{mcp_tools_table}}` on non-ledger personas without triggering an unresolved-variable warning.\n\n#### Usage\n\n```js\n// personas/persona-build.config.js\nconst { ledgerPlugin } = require('@mistralys/persona-builder/plugins/ledger');\nconst manifest = require('../shared/workflow-manifest.json');\n\nmodule.exports = {\n  rootDir: __dirname,\n  sharedPartialsDir: './shared/partials',\n  suites: {\n    ledger: {\n      srcDir: './ledger/src',\n      outVscode: './ledger/vs-code',\n      outClaudeCode: './ledger/claude-code',\n      personaMode: 'numbered',\n    },\n  },\n  plugins: [\n    ledgerPlugin({\n      manifestRoles: manifest.roles.map(r => r.name),\n      warnOnUnknownRole: true,\n    }),\n  ],\n};\n```\n\n---\n\n### `LedgerPluginOptions`\n\n```ts\ninterface LedgerPluginOptions {\n  /**\n   * List of canonical role names from the project's workflow manifest.\n   * Every persona's `role` field is validated against this list.\n   * When omitted or empty, role validation is skipped.\n   */\n  manifestRoles?: ReadonlyArray<string>;\n\n  /**\n   * Controls the severity emitted when a persona's `role` field is not found\n   * in `manifestRoles`.\n   *\n   * - `true` (default): unknown role \u2192 `warning` severity. The build continues\n   *   and the issue is surfaced as an advisory.\n   * - `false`: unknown role \u2192 `error` severity. The build treats the unknown\n   *   role as a hard failure.\n   *\n   * @default true\n   */\n  warnOnUnknownRole?: boolean;\n}\n```\n\n---\n\n### Validator Severity Escalation Pattern\n\nThe ledger plugin uses a deliberate separation between **validator logic** and **severity policy** that plugin authors can adopt in their own validators.\n\n**Contract:**\n\n- Validator functions (e.g. `validateRole`) always return `warning`-severity results. They detect the condition and surface it \u2014 they do not decide how serious it is.\n- The plugin factory's `onValidate` hook applies severity policy based on plugin options. When `warnOnUnknownRole: false`, every `warning` emitted by `validateRole` is escalated to `error` before the result is returned.\n\n**Why this pattern?**\n\nIt keeps validator logic pure and independently testable. Validators only answer \"did this condition occur?\". The factory decides \"how bad is it?\" based on runtime configuration.\n\n**Implementation sketch:**\n\n```ts\nonValidate(persona, _suite, _target): ValidationResult[] {\n  const raw = validateRole(persona.role, manifestRoles);\n  return raw.map(r => ({\n    ...r,\n    severity: (r.severity === 'warning' && !warnOnUnknownRole)\n      ? 'error'\n      : r.severity,\n  }));\n}\n```\n\nPlugin authors can apply the same pattern to any option that should toggle an advisory into a hard failure:\n\n```ts\n// Validator always returns 'warning'\nfunction validateCustomField(value: unknown): ValidationResult[] {\n  if (!value) {\n    return [{ severity: 'warning', message: 'customField is missing' }];\n  }\n  return [];\n}\n\n// Factory escalates based on options\nconst raw = validateCustomField(persona.customField);\nconst results = raw.map(r => ({\n  ...r,\n  severity: (r.severity === 'warning' && options.strictCustomField)\n    ? 'error'\n    : r.severity,\n}));\n```\n\n---\n\n### Cache Key Conventions for Multi-Target Builds\n\nWhen a plugin caches per-persona state (e.g. rendered output) for use across\nhooks, the cache key **must** include the build target to prevent collisions\nin multi-target builds. Use the composite key pattern:\n\n```ts\nconst cacheKey = `${persona.name}:${target}`;\n```\n\nIn `onValidate`, the `target` parameter is optional (`target?: TargetType`).\nWhen absent (e.g. in unit-test contexts where `onValidate` may be called\ndirectly), fall back to a sentinel value using nullish coalescing:\n\n```ts\nconst cacheKey = `${persona.name}:${target ?? 'unknown'}`;\nconst cached = renderedOutputCache.get(cacheKey) ?? '';\n```\n\n**Why `??` and not `||`?** Nullish coalescing (`??`) only triggers on\n`null`/`undefined`, preserving any falsy-but-valid value. While `TargetType`\nis currently always a non-empty string, the `??` pattern is semantically\ncorrect and defensive against future type changes.\n\n---\n\n### Exported types\n\n#### `RosterEntry`\n\n```ts\ninterface RosterEntry {\n  /** Sequential agent number (1-based) */\n  number: number;\n  /** Full display title for this agent role */\n  title: string;\n  /** Short description / label shown in parentheses */\n  short: string;\n}\n```\n\nRepresents one entry in the ledger agent roster as declared in `_shared.yaml`.\n\n#### `McpToolEntry`\n\n```ts\ninterface McpToolEntry {\n  /** Tool identifier as used in the MCP server */\n  tool: string;\n  /** Human-readable description of what the tool does */\n  purpose: string;\n  /**\n   * When true, this entry appears in documentation notes only and must NOT\n   * appear in the rendered persona output.\n   */\n  note_only?: boolean;\n}\n```\n\nRepresents one MCP tool entry from the persona YAML `mcp_tools` field.\n\n---\n\n### Exported renderer functions\n\n#### `renderRoster(roster, activeNumber)`\n\n```ts\nfunction renderRoster(roster: RosterEntry[], activeNumber: number): string\n```\n\nRenders the agent roster as a numbered Markdown list. Each entry is formatted as `{number}. **{title}[ (YOU)]** ({short})`. The `(YOU)` suffix is appended to the entry whose `number` matches `activeNumber`.\n\n```ts\nrenderRoster([\n  { number: 1, title: 'Planner', short: 'plans the work' },\n  { number: 2, title: 'Developer', short: 'writes code' },\n], 1)\n// => \"1. **Planner (YOU)** (plans the work)\\n2. **Developer** (writes code)\"\n```\n\n#### `renderMcpToolsTable(tools)`\n\n```ts\nfunction renderMcpToolsTable(tools: McpToolEntry[]): string\n```\n\nRenders the MCP tools array as Markdown table rows (`| \\`tool\\` | purpose |`). Entries with `note_only: true` are filtered out and will not appear in the output.\n\n```ts\nrenderMcpToolsTable([\n  { tool: 'ledger_get_status', purpose: 'Read project status' },\n  { tool: 'internal_tool',    purpose: 'Internal use only', note_only: true },\n])\n// => \"| `ledger_get_status` | Read project status |\"\n```\n\n---\n\n### Exported validator functions\n\n#### `validateRole(role, manifestRoles)`\n\n```ts\nfunction validateRole(\n  role: string | undefined,\n  manifestRoles: ReadonlyArray<string> | ReadonlySet<string>,\n): ValidationResult[]\n```\n\nChecks that a persona's `role` field is present in the workflow manifest. Returns an empty array when the role is valid or absent (non-ledger personas have no `role` field), and a single `warning`-level result when the role is not found.\n\n```ts\nvalidateRole('Developer', ['Planner', 'Developer', 'QA'])\n// => []\n\nvalidateRole('Coder', ['Planner', 'Developer', 'QA'])\n// => [{ severity: 'warning', message: 'Role \"Coder\" is not in the workflow manifest. Known roles: ...' }]\n\nvalidateRole(undefined, ['Planner', 'Developer'])\n// => []  \u2014 role absent, not a ledger persona\n```\n\n> `manifestRoles` is normalised to a `Set` internally, so both `Array` and `Set` inputs are accepted with O(1) lookup behaviour.\n\n#### `validateNoteOnlyGuard(output, mcpTools)`\n\n```ts\nfunction validateNoteOnlyGuard(\n  output: string,\n  mcpTools: ReadonlyArray<McpToolEntry> | undefined,\n): ValidationResult[]\n```\n\nSecond-line defence against `note_only: true` tools leaking into published persona output. Even if `renderMcpToolsTable` is bypassed, this guard detects any note-only tool name appearing as a Markdown table cell (`| \\`toolName\\` |`) in the rendered string and returns one `error`-level result per violation.\n\n```ts\nvalidateNoteOnlyGuard('| `internal_tool` | does stuff |', [\n  { tool: 'internal_tool', purpose: 'Internal', note_only: true },\n])\n// => [{ severity: 'error', message: 'note_only tool \"internal_tool\" appears in rendered output.' }]\n```\n\n> The `onValidate` hook supplies both arguments via its closure over the plugin's `renderedOutputCache`. Tool names containing regex special characters (`.`, `+`, `*`, etc.) are correctly escaped before pattern matching.\n\n---\n\n### Exported frontmatter constants\n\n#### `FRONTMATTER_LEDGER_VSCODE`\n\n```ts\nconst FRONTMATTER_LEDGER_VSCODE: string\n```\n\nVS Code frontmatter template for the ledger persona suite. Resolved template variables: `{{id}}`, `{{number}}`, `{{role}}`, `{{version}}`, `{{total}}`, `{{model}}`, `{{author}}`, `{{last_updated}}`, `{{vs_file_name}}`, `{{tools_json}}`.\n\n#### `FRONTMATTER_LEDGER_CC`\n\n```ts\nconst FRONTMATTER_LEDGER_CC: string\n```\n\nClaude Code frontmatter template for the ledger persona suite. Resolved template variables: `{{cc_name}}`, `{{cc_description}}`, `{{role}}`, `{{author}}`, `{{version}}`, `{{last_updated}}`, `{{cc_tools_json}}`, `{{cc_permission_mode}}`, `{{cc_model}}`, `{{cc_memory}}`, `{{mcp_server_name}}`. The `mcpServers` block is conditionally included via `{{#if has_mcp}}`.\n\nBoth templates are structurally identical to the originals in `build-personas.js` and are registered automatically by `ledgerPlugin()` via the `frontmatterTemplates` map, overriding the library's built-in defaults for the ledger suite.",
    "new_string": ""
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_01D92tgWg2aryxzy5BcJr97L`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
  "old_string": "---\n\n### `ledgerPlugin(options?)`\n\n```ts\nfunction ledgerPlugin(options?: LedgerPluginOptions): PersonaBuildPlugin\n```\n\nFactory function. Returns a fully configured `PersonaBuildPlugin` for the ledger persona suite. The returned plugin object implements:\n\n| Hook / field | Behaviour |\n|---|---|\n| `name` | `'ledger'` |\n| `onBuildContext` | Injects `roster_rendered` and `mcp_tools_table` into the template context |\n| `onPostRender` | Caches the rendered output per-persona for use by `onValidate` |\n| `onValidate` | Runs role validation and the `note_only` guard; returns `ValidationResult[]` |\n| `frontmatterTemplates` | Registers `FRONTMATTER_LEDGER_VSCODE` (`vscode`) and `FRONTMATTER_LEDGER_CC` (`claude-code`) |\n\n#### Context variables injected by `onBuildContext`\n\n| Variable | Type | Value when absent |\n|---|---|---|\n| `roster_rendered` | `string` | `''` (empty \u2014 persona has no `roster` or `number` field) |\n| `mcp_tools_table` | `string` | `''` (empty \u2014 persona has no `mcp_tools` field) |\n\nBoth keys are **always** set (even to an empty string) so templates can reference `{{roster_rendered}}` and `{{mcp_tools_table}}` on non-ledger personas without triggering an unresolved-variable warning.\n\n#### Usage\n\n```js\n// personas/persona-build.config.js\nconst { ledgerPlugin } = require('@mistralys/persona-builder/plugins/ledger');\nconst manifest = require('../shared/workflow-manifest.json');\n\nmodule.exports = {\n  rootDir: __dirname,\n  sharedPartialsDir: './shared/partials',\n  suites: {\n    ledger: {\n      srcDir: './ledger/src',\n      outVscode: './ledger/vs-code',\n      outClaudeCode: './ledger/claude-code',\n      personaMode: 'numbered',\n    },\n  },\n  plugins: [\n    ledgerPlugin({\n      manifestRoles: manifest.roles.map(r => r.name),\n      warnOnUnknownRole: true,\n    }),\n  ],\n};\n```\n\n---\n\n### `LedgerPluginOptions`\n\n```ts\ninterface LedgerPluginOptions {\n  /**\n   * List of canonical role names from the project's workflow manifest.\n   * Every persona's `role` field is validated against this list.\n   * When omitted or empty, role validation is skipped.\n   */\n  manifestRoles?: ReadonlyArray<string>;\n\n  /**\n   * Controls the severity emitted when a persona's `role` field is not found\n   * in `manifestRoles`.\n   *\n   * - `true` (default): unknown role \u2192 `warning` severity. The build continues\n   *   and the issue is surfaced as an advisory.\n   * - `false`: unknown role \u2192 `error` severity. The build treats the unknown\n   *   role as a hard failure.\n   *\n   * @default true\n   */\n  warnOnUnknownRole?: boolean;\n}\n```\n\n---\n\n### Validator Severity Escalation Pattern\n\nThe ledger plugin uses a deliberate separation between **validator logic** and **severity policy** that plugin authors can adopt in their own validators.\n\n**Contract:**\n\n- Validator functions (e.g. `validateRole`) always return `warning`-severity results. They detect the condition and surface it \u2014 they do not decide how serious it is.\n- The plugin factory's `onValidate` hook applies severity policy based on plugin options. When `warnOnUnknownRole: false`, every `warning` emitted by `validateRole` is escalated to `error` before the result is returned.\n\n**Why this pattern?**\n\nIt keeps validator logic pure and independently testable. Validators only answer \"did this condition occur?\". The factory decides \"how bad is it?\" based on runtime configuration.\n\n**Implementation sketch:**\n\n```ts\nonValidate(persona, _suite, _target): ValidationResult[] {\n  const raw = validateRole(persona.role, manifestRoles);\n  return raw.map(r => ({\n    ...r,\n    severity: (r.severity === 'warning' && !warnOnUnknownRole)\n      ? 'error'\n      : r.severity,\n  }));\n}\n```\n\nPlugin authors can apply the same pattern to any option that should toggle an advisory into a hard failure:\n\n```ts\n// Validator always returns 'warning'\nfunction validateCustomField(value: unknown): ValidationResult[] {\n  if (!value) {\n    return [{ severity: 'warning', message: 'customField is missing' }];\n  }\n  return [];\n}\n\n// Factory escalates based on options\nconst raw = validateCustomField(persona.customField);\nconst results = raw.map(r => ({\n  ...r,\n  severity: (r.severity === 'warning' && options.strictCustomField)\n    ? 'error'\n    : r.severity,\n}));\n```\n\n---\n\n### Cache Key Conventions for Multi-Target Builds\n\nWhen a plugin caches per-persona state (e.g. rendered output) for use across\nhooks, the cache key **must** include the build target to prevent collisions\nin multi-target builds. Use the composite key pattern:\n\n```ts\nconst cacheKey = `${persona.name}:${target}`;\n```\n\nIn `onValidate`, the `target` parameter is optional (`target?: TargetType`).\nWhen absent (e.g. in unit-test contexts where `onValidate` may be called\ndirectly), fall back to a sentinel value using nullish coalescing:\n\n```ts\nconst cacheKey = `${persona.name}:${target ?? 'unknown'}`;\nconst cached = renderedOutputCache.get(cacheKey) ?? '';\n```\n\n**Why `??` and not `||`?** Nullish coalescing (`??`) only triggers on\n`null`/`undefined`, preserving any falsy-but-valid value. While `TargetType`\nis currently always a non-empty string, the `??` pattern is semantically\ncorrect and defensive against future type changes.\n\n---\n\n### Exported types\n\n#### `RosterEntry`\n\n```ts\ninterface RosterEntry {\n  /** Sequential agent number (1-based) */\n  number: number;\n  /** Full display title for this agent role */\n  title: string;\n  /** Short description / label shown in parentheses */\n  short: string;\n}\n```\n\nRepresents one entry in the ledger agent roster as declared in `_shared.yaml`.\n\n#### `McpToolEntry`\n\n```ts\ninterface McpToolEntry {\n  /** Tool identifier as used in the MCP server */\n  tool: string;\n  /** Human-readable description of what the tool does */\n  purpose: string;\n  /**\n   * When true, this entry appears in documentation notes only and must NOT\n   * appear in the rendered persona output.\n   */\n  note_only?: boolean;\n}\n```\n\nRepresents one MCP tool entry from the persona YAML `mcp_tools` field.\n\n---\n\n### Exported renderer functions\n\n#### `renderRoster(roster, activeNumber)`\n\n```ts\nfunction renderRoster(roster: RosterEntry[], activeNumber: number): string\n```\n\nRenders the agent roster as a numbered Markdown list. Each entry is formatted as `{number}. **{title}[ (YOU)]** ({short})`. The `(YOU)` suffix is appended to the entry whose `number` matches `activeNumber`.\n\n```ts\nrenderRoster([\n  { number: 1, title: 'Planner', short: 'plans the work' },\n  { number: 2, title: 'Developer', short: 'writes code' },\n], 1)\n// => \"1. **Planner (YOU)** (plans the work)\\n2. **Developer** (writes code)\"\n```\n\n#### `renderMcpToolsTable(tools)`\n\n```ts\nfunction renderMcpToolsTable(tools: McpToolEntry[]): string\n```\n\nRenders the MCP tools array as Markdown table rows (`| \\`tool\\` | purpose |`). Entries with `note_only: true` are filtered out and will not appear in the output.\n\n```ts\nrenderMcpToolsTable([\n  { tool: 'ledger_get_status', purpose: 'Read project status' },\n  { tool: 'internal_tool',    purpose: 'Internal use only', note_only: true },\n])\n// => \"| `ledger_get_status` | Read project status |\"\n```\n\n---\n\n### Exported validator functions\n\n#### `validateRole(role, manifestRoles)`\n\n```ts\nfunction validateRole(\n  role: string | undefined,\n  manifestRoles: ReadonlyArray<string> | ReadonlySet<string>,\n): ValidationResult[]\n```\n\nChecks that a persona's `role` field is present in the workflow manifest. Returns an empty array when the role is valid or absent (non-ledger personas have no `role` field), and a single `warning`-level result when the role is not found.\n\n```ts\nvalidateRole('Developer', ['Planner', 'Developer', 'QA'])\n// => []\n\nvalidateRole('Coder', ['Planner', 'Developer', 'QA'])\n// => [{ severity: 'warning', message: 'Role \"Coder\" is not in the workflow manifest. Known roles: ...' }]\n\nvalidateRole(undefined, ['Planner', 'Developer'])\n// => []  \u2014 role absent, not a ledger persona\n```\n\n> `manifestRoles` is normalised to a `Set` internally, so both `Array` and `Set` inputs are accepted with O(1) lookup behaviour.\n\n#### `validateNoteOnlyGuard(output, mcpTools)`\n\n```ts\nfunction validateNoteOnlyGuard(\n  output: string,\n  mcpTools: ReadonlyArray<McpToolEntry> | undefined,\n): ValidationResult[]\n```\n\nSecond-line defence against `note_only: true` tools leaking into published persona output. Even if `renderMcpToolsTable` is bypassed, this guard detects any note-only tool name appearing as a Markdown table cell (`| \\`toolName\\` |`) in the rendered string and returns one `error`-level result per violation.\n\n```ts\nvalidateNoteOnlyGuard('| `internal_tool` | does stuff |', [\n  { tool: 'internal_tool', purpose: 'Internal', note_only: true },\n])\n// => [{ severity: 'error', message: 'note_only tool \"internal_tool\" appears in rendered output.' }]\n```\n\n> The `onValidate` hook supplies both arguments via its closure over the plugin's `renderedOutputCache`. Tool names containing regex special characters (`.`, `+`, `*`, etc.) are correctly escaped before pattern matching.\n\n---\n\n### Exported frontmatter constants\n\n#### `FRONTMATTER_LEDGER_VSCODE`\n\n```ts\nconst FRONTMATTER_LEDGER_VSCODE: string\n```\n\nVS Code frontmatter template for the ledger persona suite. Resolved template variables: `{{id}}`, `{{number}}`, `{{role}}`, `{{version}}`, `{{total}}`, `{{model}}`, `{{author}}`, `{{last_updated}}`, `{{vs_file_name}}`, `{{tools_json}}`.\n\n#### `FRONTMATTER_LEDGER_CC`\n\n```ts\nconst FRONTMATTER_LEDGER_CC: string\n```\n\nClaude Code frontmatter template for the ledger persona suite. Resolved template variables: `{{cc_name}}`, `{{cc_description}}`, `{{role}}`, `{{author}}`, `{{version}}`, `{{last_updated}}`, `{{cc_tools_json}}`, `{{cc_permission_mode}}`, `{{cc_model}}`, `{{cc_memory}}`, `{{mcp_server_name}}`. The `mcpServers` block is conditionally included via `{{#if has_mcp}}`.\n\nBoth templates are structurally identical to the originals in `build-personas.js` and are registered automatically by `ledgerPlugin()` via the `frontmatterTemplates` map, overriding the library's built-in defaults for the ledger suite.",
  "new_string": ""
}
```

## Tool Result

Successfully replaced 1 instance(s) of the string in '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md'

## Assistant

```json
{
  "id": "toolu_01LxcvmP511H66vgmRXyqJtv",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Update docs/plugins.md \u2014 remove ledger plugin section, add migration note",
        "status": "completed"
      },
      {
        "content": "Update api-surface.md \u2014 remove ledger plugin section",
        "status": "in_progress"
      },
      {
        "content": "Update file-tree.md \u2014 remove ledger plugin entries",
        "status": "in_progress"
      },
      {
        "content": "Update data-flows.md \u2014 remove ledger-specific references if any",
        "status": "pending"
      },
      {
        "content": "Update CHANGELOG.md \u2014 add 2.0.0 breaking change entry",
        "status": "pending"
      },
      {
        "content": "Complete documentation pipeline in ledger",
        "status": "pending"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_01LxcvmP511H66vgmRXyqJtv`)

```json
{
  "todos": [
    {
      "content": "Update docs/plugins.md \u2014 remove ledger plugin section, add migration note",
      "status": "completed"
    },
    {
      "content": "Update api-surface.md \u2014 remove ledger plugin section",
      "status": "in_progress"
    },
    {
      "content": "Update file-tree.md \u2014 remove ledger plugin entries",
      "status": "in_progress"
    },
    {
      "content": "Update data-flows.md \u2014 remove ledger-specific references if any",
      "status": "pending"
    },
    {
      "content": "Update CHANGELOG.md \u2014 add 2.0.0 breaking change entry",
      "status": "pending"
    },
    {
      "content": "Complete documentation pipeline in ledger",
      "status": "pending"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Update docs/plugins.md — remove ledger plugin section, add migration note', 'status': 'completed'}, {'content': 'Update api-surface.md — remove ledger plugin section', 'status': 'in_progress'}, {'content': 'Update file-tree.md — remove ledger plugin entries', 'status': 'in_progress'}, {'content': 'Update data-flows.md — remove ledger-specific references if any', 'status': 'pending'}, {'content': 'Update CHANGELOG.md — add 2.0.0 breaking change entry', 'status': 'pending'}, {'content': 'Complete documentation pipeline in ledger', 'status': 'pending'}]

## Assistant

Now update `api-surface.md` — remove the entire `## Ledger Plugin` section at the end:

```json
{
  "id": "toolu_01WcxLXxQaqieQodrJik7ThH",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md",
    "old_string": "---\n\n## Ledger Plugin \u2014 `@mistralys/persona-builder/plugins/ledger`\n\nSub-path export. Not included in the main `@mistralys/persona-builder` barrel.\n\n### `ledgerPlugin(options?)`\n\n```ts\nexport function ledgerPlugin(options?: LedgerPluginOptions): PersonaBuildPlugin;\n```\n\nFactory function that returns a `PersonaBuildPlugin` for the ledger persona suite. The plugin wires `renderRoster`, `renderMcpToolsTable`, `validateRole`, and `validateNoteOnlyGuard` into the standard build hooks. Also registers `FRONTMATTER_LEDGER_VSCODE` and `FRONTMATTER_LEDGER_CC` as target-specific frontmatter templates.\n\n### `LedgerPluginOptions`\n\n```ts\nexport interface LedgerPluginOptions {\n  manifestRoles?: ReadonlyArray<string>;\n  warnOnUnknownRole?: boolean;  // default: true\n}\n```\n\n### `renderRoster(roster, activeNumber)`\n\n```ts\nexport function renderRoster(roster: RosterEntry[], activeNumber: number): string;\n```\n\nRenders the agent roster as a numbered Markdown list. Appends `(YOU)` to the entry matching `activeNumber`.\n\n### `RosterEntry`\n\n```ts\nexport interface RosterEntry {\n  number: number;\n  title: string;\n  short: string;\n}\n```\n\n### `renderMcpToolsTable(tools)`\n\n```ts\nexport function renderMcpToolsTable(tools: McpToolEntry[]): string;\n```\n\nRenders the MCP tools array as Markdown table rows. Entries with `note_only: true` are filtered out.\n\n### `McpToolEntry`\n\n```ts\nexport interface McpToolEntry {\n  tool: string;\n  purpose: string;\n  note_only?: boolean;\n}\n```\n\n### `validateRole(role, manifestRoles)`\n\n```ts\nexport function validateRole(\n  role: string | undefined,\n  manifestRoles: ReadonlyArray<string> | ReadonlySet<string>,\n): ValidationResult[];\n```\n\nValidates a persona's `role` field against the workflow manifest. Returns an empty array when the role is valid or absent; a single warning when the role is not found.\n\n### `validateNoteOnlyGuard(output, mcpTools)`\n\n```ts\nexport function validateNoteOnlyGuard(\n  output: string,\n  mcpTools: ReadonlyArray<McpToolEntry> | undefined,\n): ValidationResult[];\n```\n\nAsserts that `note_only: true` MCP tools do not appear in the rendered persona output. Returns one error per leaking tool name.\n\n### `FRONTMATTER_LEDGER_VSCODE`\n\n```ts\nexport const FRONTMATTER_LEDGER_VSCODE: string;\n```\n\nVS Code frontmatter template for the ledger persona suite.\n\n### `FRONTMATTER_LEDGER_CC`\n\n```ts\nexport const FRONTMATTER_LEDGER_CC: string;\n```\n\nClaude Code frontmatter template for the ledger persona suite (includes conditional `mcpServers` block).",
    "new_string": ""
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_01WcxLXxQaqieQodrJik7ThH`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md",
  "old_string": "---\n\n## Ledger Plugin \u2014 `@mistralys/persona-builder/plugins/ledger`\n\nSub-path export. Not included in the main `@mistralys/persona-builder` barrel.\n\n### `ledgerPlugin(options?)`\n\n```ts\nexport function ledgerPlugin(options?: LedgerPluginOptions): PersonaBuildPlugin;\n```\n\nFactory function that returns a `PersonaBuildPlugin` for the ledger persona suite. The plugin wires `renderRoster`, `renderMcpToolsTable`, `validateRole`, and `validateNoteOnlyGuard` into the standard build hooks. Also registers `FRONTMATTER_LEDGER_VSCODE` and `FRONTMATTER_LEDGER_CC` as target-specific frontmatter templates.\n\n### `LedgerPluginOptions`\n\n```ts\nexport interface LedgerPluginOptions {\n  manifestRoles?: ReadonlyArray<string>;\n  warnOnUnknownRole?: boolean;  // default: true\n}\n```\n\n### `renderRoster(roster, activeNumber)`\n\n```ts\nexport function renderRoster(roster: RosterEntry[], activeNumber: number): string;\n```\n\nRenders the agent roster as a numbered Markdown list. Appends `(YOU)` to the entry matching `activeNumber`.\n\n### `RosterEntry`\n\n```ts\nexport interface RosterEntry {\n  number: number;\n  title: string;\n  short: string;\n}\n```\n\n### `renderMcpToolsTable(tools)`\n\n```ts\nexport function renderMcpToolsTable(tools: McpToolEntry[]): string;\n```\n\nRenders the MCP tools array as Markdown table rows. Entries with `note_only: true` are filtered out.\n\n### `McpToolEntry`\n\n```ts\nexport interface McpToolEntry {\n  tool: string;\n  purpose: string;\n  note_only?: boolean;\n}\n```\n\n### `validateRole(role, manifestRoles)`\n\n```ts\nexport function validateRole(\n  role: string | undefined,\n  manifestRoles: ReadonlyArray<string> | ReadonlySet<string>,\n): ValidationResult[];\n```\n\nValidates a persona's `role` field against the workflow manifest. Returns an empty array when the role is valid or absent; a single warning when the role is not found.\n\n### `validateNoteOnlyGuard(output, mcpTools)`\n\n```ts\nexport function validateNoteOnlyGuard(\n  output: string,\n  mcpTools: ReadonlyArray<McpToolEntry> | undefined,\n): ValidationResult[];\n```\n\nAsserts that `note_only: true` MCP tools do not appear in the rendered persona output. Returns one error per leaking tool name.\n\n### `FRONTMATTER_LEDGER_VSCODE`\n\n```ts\nexport const FRONTMATTER_LEDGER_VSCODE: string;\n```\n\nVS Code frontmatter template for the ledger persona suite.\n\n### `FRONTMATTER_LEDGER_CC`\n\n```ts\nexport const FRONTMATTER_LEDGER_CC: string;\n```\n\nClaude Code frontmatter template for the ledger persona suite (includes conditional `mcpServers` block).",
  "new_string": ""
}
```

## Tool Result

Successfully replaced 1 instance(s) of the string in '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md'

## Assistant

Now update `file-tree.md` — remove the `ledger/` directory entries and update the `tsup.config.ts` note (which mentions a third entry point for `plugins/ledger/index`):

```json
{
  "id": "toolu_016yiwrXcttzXo76GRAvtUYE",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md",
    "old_string": "\u251c\u2500\u2500 tsup.config.ts                  # Build config (dual format, three entry points: index, cli, plugins/ledger/index)",
    "new_string": "\u251c\u2500\u2500 tsup.config.ts                  # Build config (dual format, two entry points: index, cli)"
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_016yiwrXcttzXo76GRAvtUYE`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md",
  "old_string": "\u251c\u2500\u2500 tsup.config.ts                  # Build config (dual format, three entry points: index, cli, plugins/ledger/index)",
  "new_string": "\u251c\u2500\u2500 tsup.config.ts                  # Build config (dual format, two entry points: index, cli)"
}
```

## Tool Result

Successfully replaced 1 instance(s) of the string in '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md'

## Assistant

```json
{
  "id": "toolu_01Gak2gBEyjwCjbHqVjw1YSb",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md",
    "old_string": "\u2502   \u251c\u2500\u2500 plugins/                    # Plugin system\n\u2502   \u2502   \u251c\u2500\u2500 index.ts                # Barrel re-export (types + runner functions)\n\u2502   \u2502   \u251c\u2500\u2500 types.ts                # Core types: TargetType, PersonaMetadata, SuiteConfig, etc.\n\u2502   \u2502   \u251c\u2500\u2500 runner.ts               # Hook invocation: runSuiteInit, runBuildContext, etc.\n\u2502   \u2502   \u2514\u2500\u2500 ledger/                 # Ledger plugin (sub-path: @mistralys/persona-builder/plugins/ledger)\n\u2502   \u2502       \u251c\u2500\u2500 index.ts            # ledgerPlugin(options) factory \u2014 LedgerPluginOptions type\n\u2502   \u2502       \u251c\u2500\u2500 roster-renderer.ts  # renderRoster() \u2014 numbered Markdown list with (YOU) marker\n\u2502   \u2502       \u251c\u2500\u2500 mcp-tools-renderer.ts # renderMcpToolsTable() \u2014 filters note_only entries\n\u2502   \u2502       \u251c\u2500\u2500 role-validator.ts   # validateRole() + validateNoteOnlyGuard()\n\u2502   \u2502       \u2514\u2500\u2500 frontmatter-templates.ts # FRONTMATTER_LEDGER_VSCODE, FRONTMATTER_LEDGER_CC",
    "new_string": "\u2502   \u251c\u2500\u2500 plugins/                    # Plugin system\n\u2502   \u2502   \u251c\u2500\u2500 index.ts                # Barrel re-export (types + runner functions)\n\u2502   \u2502   \u251c\u2500\u2500 types.ts                # Core types: TargetType, PersonaMetadata, SuiteConfig, etc.\n\u2502   \u2502   \u2514\u2500\u2500 runner.ts               # Hook invocation: runSuiteInit, runBuildContext, etc."
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_01Gak2gBEyjwCjbHqVjw1YSb`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md",
  "old_string": "\u2502   \u251c\u2500\u2500 plugins/                    # Plugin system\n\u2502   \u2502   \u251c\u2500\u2500 index.ts                # Barrel re-export (types + runner functions)\n\u2502   \u2502   \u251c\u2500\u2500 types.ts                # Core types: TargetType, PersonaMetadata, SuiteConfig, etc.\n\u2502   \u2502   \u251c\u2500\u2500 runner.ts               # Hook invocation: runSuiteInit, runBuildContext, etc.\n\u2502   \u2502   \u2514\u2500\u2500 ledger/                 # Ledger plugin (sub-path: @mistralys/persona-builder/plugins/ledger)\n\u2502   \u2502       \u251c\u2500\u2500 index.ts            # ledgerPlugin(options) factory \u2014 LedgerPluginOptions type\n\u2502   \u2502       \u251c\u2500\u2500 roster-renderer.ts  # renderRoster() \u2014 numbered Markdown list with (YOU) marker\n\u2502   \u2502       \u251c\u2500\u2500 mcp-tools-renderer.ts # renderMcpToolsTable() \u2014 filters note_only entries\n\u2502   \u2502       \u251c\u2500\u2500 role-validator.ts   # validateRole() + validateNoteOnlyGuard()\n\u2502   \u2502       \u2514\u2500\u2500 frontmatter-templates.ts # FRONTMATTER_LEDGER_VSCODE, FRONTMATTER_LEDGER_CC",
  "new_string": "\u2502   \u251c\u2500\u2500 plugins/                    # Plugin system\n\u2502   \u2502   \u251c\u2500\u2500 index.ts                # Barrel re-export (types + runner functions)\n\u2502   \u2502   \u251c\u2500\u2500 types.ts                # Core types: TargetType, PersonaMetadata, SuiteConfig, etc.\n\u2502   \u2502   \u2514\u2500\u2500 runner.ts               # Hook invocation: runSuiteInit, runBuildContext, etc."
}
```

## Tool Result

Successfully replaced 1 instance(s) of the string in '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md'

## Assistant

Now also remove the ledger plugin test file entry from the tests section:

```json
{
  "id": "toolu_01AfVDD6aCRBujzKdf8NDm59",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md",
    "offset": 57,
    "limit": 30
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01AfVDD6aCRBujzKdf8NDm59`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md",
  "offset": 57,
  "limit": 30
}
```

## Tool Result

    58	│   │   └── serializer.test.ts
    59	│   ├── loaders/                    # Loader tests (40 tests)
    60	│   │   ├── partials-loader.test.ts
    61	│   │   ├── metadata-loader.test.ts
    62	│   │   └── content-loader.test.ts
    63	│   ├── plugins/                    # Plugin system tests (27 tests)
    64	│   │   └── plugin-runner.test.ts
    65	│   ├── builders/                   # Builder tests (33 tests)
    66	│   │   ├── persona-builder.test.ts
    67	│   │   └── persona-builder-edge-cases.test.ts
    68	│   ├── validators/                 # Validator tests (46 tests)
    69	│   │   ├── filename-validator.test.ts
    70	│   │   └── strict-validator.test.ts
    71	│   └── integration/                # End-to-end integration tests (7 tests)
    72	│       └── build.test.ts
    73	│
    74	├── fixtures/                       # Test fixtures
    75	│   ├── sample-suite/
    76	│   │   ├── meta/
    77	│   │   │   ├── _shared.yaml        # Suite-level shared defaults
    78	│   │   │   └── example-persona.yaml
    79	│   │   ├── content/
    80	│   │   │   └── example-persona.md  # Markdown content template
    81	│   │   └── partials/
    82	│   │       └── suite-specific.md   # Suite-local partial
    83	│   └── shared/
    84	│       └── partials/
    85	│           └── greeting.md         # Cross-suite shared partial
    86	│
    87	├── dist/                           # Build output (gitignored)

## Assistant

The tests/plugins section only shows `plugin-runner.test.ts` — but the 1.0.0 CHANGELOG mentioned `tests/plugins/ledger.test.ts` existed. Let me check if it's still there or if WP-003 removed it:

```json
{
  "id": "toolu_017xobCdpoD4hbAVdGgt4Eyi",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests"
  },
  "name": "ls",
  "type": "tool_use"
}
```

**Tool call:** `ls` (id: `toolu_017xobCdpoD4hbAVdGgt4Eyi`)

```json
{
  "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests"
}
```

## Tool Result

['/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests/.gitkeep', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests/README.md', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests/builders/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests/engine/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests/integration/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests/loaders/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests/plugins/', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests/validators/']

## Assistant

```json
{
  "id": "toolu_011TFVsn8J8sgBgGEP2W6SCT",
  "caller": {
    "type": "direct"
  },
  "input": {
    "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests/plugins"
  },
  "name": "ls",
  "type": "tool_use"
}
```

**Tool call:** `ls` (id: `toolu_011TFVsn8J8sgBgGEP2W6SCT`)

```json
{
  "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests/plugins"
}
```

## Tool Result

['/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests/plugins/.gitkeep', '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/tests/plugins/plugin-runner.test.ts']

## Assistant

Good — `ledger.test.ts` has already been removed by WP-003. The file-tree manifest already correctly only shows `plugin-runner.test.ts` in that folder. The test count comment says "27 tests" — let me leave that as-is since the WP spec doesn't mention updating test counts. The file-tree manifest is now accurate structurally.

Now check `data-flows.md` for any ledger-specific references. From what I read, it doesn't have direct references to the ledger plugin — it describes the general build pipeline and plugin hook execution. Let me confirm:

```json
{
  "id": "toolu_018nopgGqkcyvaTcVjkkhgw7",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "ledger",
    "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest"
  },
  "name": "grep",
  "type": "tool_use"
}
```

**Tool call:** `grep` (id: `toolu_018nopgGqkcyvaTcVjkkhgw7`)

```json
{
  "pattern": "ledger",
  "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest"
}
```

## Tool Result

/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/constraints.md

## Assistant

No ledger references remain in `data-flows.md` or the other manifest files (only `constraints.md` still has some). The WP only requires updating the three named files (api-surface, file-tree, data-flows). `data-flows.md` has no ledger references — it's already clean.

```json
{
  "id": "toolu_01HAnMsghowGiwExCzFeBBFT",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Update docs/plugins.md \u2014 remove ledger plugin section, add migration note",
        "status": "completed"
      },
      {
        "content": "Update api-surface.md \u2014 remove ledger plugin section",
        "status": "completed"
      },
      {
        "content": "Update file-tree.md \u2014 remove ledger plugin entries",
        "status": "completed"
      },
      {
        "content": "Update data-flows.md \u2014 remove ledger-specific references if any",
        "status": "completed"
      },
      {
        "content": "Update CHANGELOG.md \u2014 add 2.0.0 breaking change entry",
        "status": "in_progress"
      },
      {
        "content": "Complete documentation pipeline in ledger",
        "status": "pending"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_01HAnMsghowGiwExCzFeBBFT`)

```json
{
  "todos": [
    {
      "content": "Update docs/plugins.md \u2014 remove ledger plugin section, add migration note",
      "status": "completed"
    },
    {
      "content": "Update api-surface.md \u2014 remove ledger plugin section",
      "status": "completed"
    },
    {
      "content": "Update file-tree.md \u2014 remove ledger plugin entries",
      "status": "completed"
    },
    {
      "content": "Update data-flows.md \u2014 remove ledger-specific references if any",
      "status": "completed"
    },
    {
      "content": "Update CHANGELOG.md \u2014 add 2.0.0 breaking change entry",
      "status": "in_progress"
    },
    {
      "content": "Complete documentation pipeline in ledger",
      "status": "pending"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Update docs/plugins.md — remove ledger plugin section, add migration note', 'status': 'completed'}, {'content': 'Update api-surface.md — remove ledger plugin section', 'status': 'completed'}, {'content': 'Update file-tree.md — remove ledger plugin entries', 'status': 'completed'}, {'content': 'Update data-flows.md — remove ledger-specific references if any', 'status': 'completed'}, {'content': 'Update CHANGELOG.md — add 2.0.0 breaking change entry', 'status': 'in_progress'}, {'content': 'Complete documentation pipeline in ledger', 'status': 'pending'}]

## Assistant

Now add the `[2.0.0]` entry to `CHANGELOG.md`. The current latest is `[1.0.1]`, so I'll insert the new entry between `[Unreleased]` and `[1.0.1]`:

```json
{
  "id": "toolu_01VSMbcCsc9TxbbVFD2TAfZr",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/CHANGELOG.md",
    "old_string": "## [Unreleased]\n\n## [1.0.1] - 2026-03-26",
    "new_string": "## [Unreleased]\n\n## [2.0.0] - Upcoming\n\n### Breaking Changes\n\n- Removed the `./plugins/ledger` sub-path export. The ledger plugin has been migrated to the ai-insights-dev workspace as a local CommonJS module at `personas/plugins/ledger/`. No external consumers are affected (the library has no published consumers).\n\n## [1.0.1] - 2026-03-26"
  },
  "name": "edit_file",
  "type": "tool_use"
}
```

**Tool call:** `edit_file` (id: `toolu_01VSMbcCsc9TxbbVFD2TAfZr`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/CHANGELOG.md",
  "old_string": "## [Unreleased]\n\n## [1.0.1] - 2026-03-26",
  "new_string": "## [Unreleased]\n\n## [2.0.0] - Upcoming\n\n### Breaking Changes\n\n- Removed the `./plugins/ledger` sub-path export. The ledger plugin has been migrated to the ai-insights-dev workspace as a local CommonJS module at `personas/plugins/ledger/`. No external consumers are affected (the library has no published consumers).\n\n## [1.0.1] - 2026-03-26"
}
```

## Tool Result

Successfully replaced 1 instance(s) of the string in '/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/CHANGELOG.md'

## Assistant

Now let me do a final verification pass on all modified files:

```json
{
  "id": "toolu_01FWYBLMvb1U1mJBPPL5pqwZ",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
    "offset": 115,
    "limit": 20
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01JnC5ehpQzTDp2VkN5X3UU5",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/CHANGELOG.md",
    "offset": 0,
    "limit": 25
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01FWYBLMvb1U1mJBPPL5pqwZ`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/plugins.md",
  "offset": 115,
  "limit": 20
}
```

**Tool call:** `read_file` (id: `toolu_01JnC5ehpQzTDp2VkN5X3UU5`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/CHANGELOG.md",
  "offset": 0,
  "limit": 25
}
```

## Tool Result

   116	
   117	---
   118	
   119	## Ledger Plugin — Migrated
   120	
   121	> **Moved in v2.0.0** — The ledger plugin has been removed from this library and migrated to the
   122	> [ai-insights-dev](https://github.com/mistralys/ai-insights-dev) workspace as a local CommonJS
   123	> module at `personas/plugins/ledger/`. The `@mistralys/persona-builder/plugins/ledger` sub-path
   124	> export no longer exists. See the ai-insights-dev repository for current ledger plugin source and
   125	> documentation.
   126	
   127	

## Tool Result

     1	# Changelog
     2	
     3	All notable changes to `@mistralys/persona-builder` will be documented in this file.
     4	
     5	The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
     6	and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
     7	
     8	## [Unreleased]
     9	
    10	## [2.0.0] - Upcoming
    11	
    12	### Breaking Changes
    13	
    14	- Removed the `./plugins/ledger` sub-path export. The ledger plugin has been migrated to the ai-insights-dev workspace as a local CommonJS module at `personas/plugins/ledger/`. No external consumers are affected (the library has no published consumers).
    15	
    16	## [1.0.1] - 2026-03-26
    17	
    18	Patch release addressing known tech-debt items, documentation fixes, and two consumer-side bug fixes identified in the post-integration synthesis.
    19	
    20	### Changed (library)
    21	
    22	- **Fix `warnOnUnknownRole` documentation** — Removed the stale "not yet wired" known-limitation blockquote from `docs/plugins.md` and replaced it with accurate documentation of the `warnOnUnknownRole` escalation contract (`true` → `warning`, `false` → `error`). Added a "Validator Severity Escalation Pattern" subsection for future plugin authors. Updated the corresponding JSDoc in `src/plugins/ledger/index.ts` and the `api-surface.md` manifest.
    23	- **Resolve `TargetType` dual re-export** — Eliminated the duplicate `TargetType` re-export from `src/builders/types.ts` and `src/builders/index.ts`, resolving a tech-debt item flagged in `constraints.md` before 1.0. `TargetType` is now exported exclusively via `src/plugins/types.ts` → `src/plugins/index.ts` → `src/index.ts`.
    24	- **Extract `escapeRegExp` to shared utility** — Moved the previously private `escapeRegExp` function from `src/plugins/ledger/role-validator.ts` into a new shared module at `src/utils/regex.ts` with a barrel at `src/utils/index.ts`. The function is now a named export of the library's main barrel, available to all future validators and plugins without duplication.
    25	- **Improve `renderedOutputCache` keying** — Extended the `onValidate` hook signature with an optional `target?: TargetType` parameter and propagated it through the plugin runner (`src/plugins/runner.ts`) and persona builder (`src/builders/persona-builder.ts`). The ledger plugin now uses a composite cache key `${persona.name}:${target}` (with `'unknown'` fallback) so that multi-target builds cache and validate output correctly per target.

## Assistant

All files look correct. Let me also verify the `api-surface.md` ends cleanly after removing the ledger section:

```json
{
  "id": "toolu_01JQAr1iraFiE2LgycdhRbhL",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md",
    "offset": 390,
    "limit": 30
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01JQAr1iraFiE2LgycdhRbhL`)

```json
{
  "file_path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md",
  "offset": 390,
  "limit": 30
}
```

## Tool Result

   391	  onSuiteInit?(suite: SuiteConfig, sharedMeta: Record<string, unknown>): void;
   392	  onBuildContext?(
   393	    context: Record<string, unknown>,
   394	    persona: PersonaMetadata,
   395	    suite: SuiteConfig,
   396	  ): Record<string, unknown>;
   397	  onPostRender?(output: string, persona: PersonaMetadata, target: TargetType): string;
   398	  onValidate?(persona: PersonaMetadata, suite: SuiteConfig, target?: TargetType): ValidationResult[];
   399	  frontmatterTemplates?: Partial<Record<TargetType, string>>;
   400	}
   401	```
   402	
   403	### `TargetType`
   404	
   405	```ts
   406	export type TargetType = 'vscode' | 'claude-code';
   407	```
   408	
   409	### `ValidationResult`
   410	
   411	```ts
   412	export interface ValidationResult {
   413	  severity: 'error' | 'warning' | 'info';
   414	  message: string;
   415	}
   416	```
   417	
   418	

## Assistant

The file ends cleanly after `ValidationResult`. Now let me do a final grep to confirm no ledger references remain in the three manifest files:

```json
{
  "id": "toolu_01KPhH58hRX6biBdx3YcUieP",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "ledger",
    "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md"
  },
  "name": "grep",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01Gay3NCgaJpGXMfL8QAHBMu",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "ledger",
    "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md"
  },
  "name": "grep",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01JmKYEsHJQgLqya9cfPgQbr",
  "caller": {
    "type": "direct"
  },
  "input": {
    "pattern": "ledger",
    "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/data-flows.md"
  },
  "name": "grep",
  "type": "tool_use"
}
```

**Tool call:** `grep` (id: `toolu_01KPhH58hRX6biBdx3YcUieP`)

```json
{
  "pattern": "ledger",
  "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/api-surface.md"
}
```

**Tool call:** `grep` (id: `toolu_01Gay3NCgaJpGXMfL8QAHBMu`)

```json
{
  "pattern": "ledger",
  "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/file-tree.md"
}
```

**Tool call:** `grep` (id: `toolu_01JmKYEsHJQgLqya9cfPgQbr`)

```json
{
  "pattern": "ledger",
  "path": "/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/data-flows.md"
}
```

## Tool Result

/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/constraints.md

## Tool Result

/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/constraints.md

## Tool Result

/Users/smordziol/Webserver/libraries/ai-persona-builder-STABLE/docs/agents/project-manifest/constraints.md

## Assistant

All three target files are clean — the grep only returns `constraints.md` (not in scope). Everything is in order. Now complete the pipeline:

```json
{
  "id": "toolu_01NwJbRaKWqKwq77jTjyvoeT",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Update docs/plugins.md \u2014 remove ledger plugin section, add migration note",
        "status": "completed"
      },
      {
        "content": "Update api-surface.md \u2014 remove ledger plugin section",
        "status": "completed"
      },
      {
        "content": "Update file-tree.md \u2014 remove ledger plugin entries",
        "status": "completed"
      },
      {
        "content": "Update data-flows.md \u2014 remove ledger-specific references if any",
        "status": "completed"
      },
      {
        "content": "Update CHANGELOG.md \u2014 add 2.0.0 breaking change entry",
        "status": "completed"
      },
      {
        "content": "Complete documentation pipeline in ledger",
        "status": "in_progress"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_01NwJbRaKWqKwq77jTjyvoeT`)

```json
{
  "todos": [
    {
      "content": "Update docs/plugins.md \u2014 remove ledger plugin section, add migration note",
      "status": "completed"
    },
    {
      "content": "Update api-surface.md \u2014 remove ledger plugin section",
      "status": "completed"
    },
    {
      "content": "Update file-tree.md \u2014 remove ledger plugin entries",
      "status": "completed"
    },
    {
      "content": "Update data-flows.md \u2014 remove ledger-specific references if any",
      "status": "completed"
    },
    {
      "content": "Update CHANGELOG.md \u2014 add 2.0.0 breaking change entry",
      "status": "completed"
    },
    {
      "content": "Complete documentation pipeline in ledger",
      "status": "in_progress"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Update docs/plugins.md — remove ledger plugin section, add migration note', 'status': 'completed'}, {'content': 'Update api-surface.md — remove ledger plugin section', 'status': 'completed'}, {'content': 'Update file-tree.md — remove ledger plugin entries', 'status': 'completed'}, {'content': 'Update data-flows.md — remove ledger-specific references if any', 'status': 'completed'}, {'content': 'Update CHANGELOG.md — add 2.0.0 breaking change entry', 'status': 'completed'}, {'content': 'Complete documentation pipeline in ledger', 'status': 'in_progress'}]

## Assistant

```json
{
  "id": "toolu_01CZFaFNrWPdTgdXXmsxTvtG",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-004",
    "type": "documentation",
    "status": "PASS",
    "agent_role": "Documentation",
    "summary": "Updated all persona-builder documentation to reflect ledger plugin removal. (1) docs/plugins.md: removed the full Ledger Plugin detail section (ledgerPlugin factory, LedgerPluginOptions, validators, renderers, frontmatter constants, severity escalation pattern, cache key conventions) and replaced it with a concise migration notice explaining the plugin has moved to ai-insights-dev at personas/plugins/ledger/. (2) docs/agents/project-manifest/api-surface.md: removed the entire 'Ledger Plugin \u2014 @mistralys/persona-builder/plugins/ledger' section including all exported types, functions, and constants. File now ends cleanly at ValidationResult. (3) docs/agents/project-manifest/file-tree.md: removed the src/plugins/ledger/ subtree entries and updated tsup.config.ts note from 'three entry points: index, cli, plugins/ledger/index' to 'two entry points: index, cli'. (4) docs/agents/project-manifest/data-flows.md: already contained no ledger-specific references \u2014 no changes needed. (5) CHANGELOG.md: added [2.0.0] - Upcoming entry with a Breaking Changes section documenting the removal of the ./plugins/ledger sub-path export and the migration to ai-insights-dev.",
    "artifacts": {
      "files_modified": [
        "docs/plugins.md",
        "docs/agents/project-manifest/api-surface.md",
        "docs/agents/project-manifest/file-tree.md",
        "CHANGELOG.md"
      ]
    },
    "comments": [
      {
        "type": "improvement",
        "priority": "low",
        "note": "The plugins.md 'Validator Severity Escalation Pattern' and 'Cache Key Conventions for Multi-Target Builds' subsections were valuable architectural guidance. Now that the ledger plugin is gone from this library, those patterns are preserved as general advice in the plugin examples section \u2014 both the 'Custom frontmatter template via plugin' and 'Validation plugin' examples still illustrate the same patterns for custom plugin authors."
      },
      {
        "type": "improvement",
        "priority": "low",
        "note": "The tests/plugins section in file-tree.md still shows '27 tests' for plugin system tests. This count included the ledger plugin tests that WP-003 removed. The actual count will now be lower, but updating exact test counts is beyond documentation scope and should be updated when the test suite is next re-run."
      }
    ],
    "acceptance_criteria_updates": [
      {
        "criterion": "docs/plugins.md removes the ledger plugin section but retains the plugin interface documentation, and includes a note that the ledger plugin has migrated to ai-insights-dev",
        "met": true
      },
      {
        "criterion": "All three project-manifest files (api-surface.md, file-tree.md, data-flows.md) no longer reference the ledger plugin",
        "met": true
      },
      {
        "criterion": "CHANGELOG.md contains an entry for the upcoming 2.0.0 release documenting the ledger plugin removal as a breaking change",
        "met": true
      }
    ],
    "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights"
  },
  "name": "ledger_complete_pipeline",
  "type": "tool_use"
}
```

**Tool call:** `ledger_complete_pipeline` (id: `toolu_01CZFaFNrWPdTgdXXmsxTvtG`)

```json
{
  "work_package_id": "WP-004",
  "type": "documentation",
  "status": "PASS",
  "agent_role": "Documentation",
  "summary": "Updated all persona-builder documentation to reflect ledger plugin removal. (1) docs/plugins.md: removed the full Ledger Plugin detail section (ledgerPlugin factory, LedgerPluginOptions, validators, renderers, frontmatter constants, severity escalation pattern, cache key conventions) and replaced it with a concise migration notice explaining the plugin has moved to ai-insights-dev at personas/plugins/ledger/. (2) docs/agents/project-manifest/api-surface.md: removed the entire 'Ledger Plugin \u2014 @mistralys/persona-builder/plugins/ledger' section including all exported types, functions, and constants. File now ends cleanly at ValidationResult. (3) docs/agents/project-manifest/file-tree.md: removed the src/plugins/ledger/ subtree entries and updated tsup.config.ts note from 'three entry points: index, cli, plugins/ledger/index' to 'two entry points: index, cli'. (4) docs/agents/project-manifest/data-flows.md: already contained no ledger-specific references \u2014 no changes needed. (5) CHANGELOG.md: added [2.0.0] - Upcoming entry with a Breaking Changes section documenting the removal of the ./plugins/ledger sub-path export and the migration to ai-insights-dev.",
  "artifacts": {
    "files_modified": [
      "docs/plugins.md",
      "docs/agents/project-manifest/api-surface.md",
      "docs/agents/project-manifest/file-tree.md",
      "CHANGELOG.md"
    ]
  },
  "comments": [
    {
      "type": "improvement",
      "priority": "low",
      "note": "The plugins.md 'Validator Severity Escalation Pattern' and 'Cache Key Conventions for Multi-Target Builds' subsections were valuable architectural guidance. Now that the ledger plugin is gone from this library, those patterns are preserved as general advice in the plugin examples section \u2014 both the 'Custom frontmatter template via plugin' and 'Validation plugin' examples still illustrate the same patterns for custom plugin authors."
    },
    {
      "type": "improvement",
      "priority": "low",
      "note": "The tests/plugins section in file-tree.md still shows '27 tests' for plugin system tests. This count included the ledger plugin tests that WP-003 removed. The actual count will now be lower, but updating exact test counts is beyond documentation scope and should be updated when the test suite is next re-run."
    }
  ],
  "acceptance_criteria_updates": [
    {
      "criterion": "docs/plugins.md removes the ledger plugin section but retains the plugin interface documentation, and includes a note that the ledger plugin has migrated to ai-insights-dev",
      "met": true
    },
    {
      "criterion": "All three project-manifest files (api-surface.md, file-tree.md, data-flows.md) no longer reference the ledger plugin",
      "met": true
    },
    {
      "criterion": "CHANGELOG.md contains an entry for the upcoming 2.0.0 release documenting the ledger plugin removal as a breaking change",
      "met": true
    }
  ],
  "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights"
}
```

## Tool Result

{
  "work_package_id": "WP-004",
  "work_package_file": "work/WP-004.md",
  "status": "COMPLETE",
  "assigned_to": "Documentation",
  "dependencies": [
    "WP-003"
  ],
  "acceptance_criteria": [
    {
      "criterion": "docs/plugins.md removes the ledger plugin section but retains the plugin interface documentation, and includes a note that the ledger plugin has migrated to ai-insights-dev",
      "met": true
    },
    {
      "criterion": "All three project-manifest files (api-surface.md, file-tree.md, data-flows.md) no longer reference the ledger plugin",
      "met": true
    },
    {
      "criterion": "CHANGELOG.md contains an entry for the upcoming 2.0.0 release documenting the ledger plugin removal as a breaking change",
      "met": true
    }
  ],
  "active_pipeline_stages": [
    "documentation"
  ],
  "revision": 0,
  "pipelines": [
    {
      "type": "documentation",
      "status": "PASS",
      "started_at": "2026-03-26T12:46:38Z",
      "completed_at": "2026-03-26T12:50:13Z",
      "duration_ms": 215000,
      "summary": [
        "Updated all persona-builder documentation to reflect ledger plugin removal. (1) docs/plugins.md: removed the full Ledger Plugin detail section (ledgerPlugin factory, LedgerPluginOptions, validators, renderers, frontmatter constants, severity escalation pattern, cache key conventions) and replaced it with a concise migration notice explaining the plugin has moved to ai-insights-dev at personas/plugins/ledger/. (2) docs/agents/project-manifest/api-surface.md: removed the entire 'Ledger Plugin — @mistralys/persona-builder/plugins/ledger' section including all exported types, functions, and constants. File now ends cleanly at ValidationResult. (3) docs/agents/project-manifest/file-tree.md: removed the src/plugins/ledger/ subtree entries and updated tsup.config.ts note from 'three entry points: index, cli, plugins/ledger/index' to 'two entry points: index, cli'. (4) docs/agents/project-manifest/data-flows.md: already contained no ledger-specific references — no changes needed. (5) CHANGELOG.md: added [2.0.0] - Upcoming entry with a Breaking Changes section documenting the removal of the ./plugins/ledger sub-path export and the migration to ai-insights-dev."
      ],
      "artifacts": {
        "files_modified": [
          "docs/plugins.md",
          "docs/agents/project-manifest/api-surface.md",
          "docs/agents/project-manifest/file-tree.md",
          "CHANGELOG.md"
        ]
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-26T12:50:13Z",
          "note": "The plugins.md 'Validator Severity Escalation Pattern' and 'Cache Key Conventions for Multi-Target Builds' subsections were valuable architectural guidance. Now that the ledger plugin is gone from this library, those patterns are preserved as general advice in the plugin examples section — both the 'Custom frontmatter template via plugin' and 'Validation plugin' examples still illustrate the same patterns for custom plugin authors."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-26T12:50:13Z",
          "note": "The tests/plugins section in file-tree.md still shows '27 tests' for plugin system tests. This count included the ledger plugin tests that WP-003 removed. The actual count will now be lower, but updating exact test counts is beyond documentation scope and should be updated when the test suite is next re-run."
        }
      ]
    }
  ],
  "status_changed_at": "2026-03-26T12:50:13Z",
  "last_updated": "2026-03-26T12:50:13Z",
  "auto_finalized": true
}

--- NEXT STEP ---
Pipeline PASS. WP WP-004 was auto-finalized to COMPLETE (all acceptance criteria met). Call ledger_get_handoff_status (current_agent: "Documentation") to confirm handoff.

## Assistant

WP-004 is **COMPLETE** (`auto_finalized: true`). Now checking handoff status:

```json
{
  "id": "toolu_01QNZQTsPipdQeXEzt2WE5K9",
  "caller": {
    "type": "direct"
  },
  "input": {
    "agent_role": "Documentation",
    "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights"
  },
  "name": "ledger_get_next_action",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_next_action` (id: `toolu_01QNZQTsPipdQeXEzt2WE5K9`)

```json
{
  "agent_role": "Documentation",
  "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights"
}
```

## Tool Result

{
  "action": "WRITE_DOCS",
  "work_package_id": "WP-006",
  "reason": "Work package WP-006 has no prior documentation pipeline and documentation is the first active stage. Write documentation.",
  "next_steps": [
    "1. Call ledger_begin_work (work_package_id: \"WP-006\", type: \"documentation\", agent_role: \"Documentation\").",
    "2. Call ledger_get_work_package to review implementation artifacts and review comments.",
    "3. Update documentation, README files, and inline docs as needed.",
    "4. Call ledger_complete_pipeline (work_package_id: \"WP-006\", type: \"documentation\", status: PASS/FAIL, summary, artifacts, comments, acceptance_criteria_updates).",
    "5. Call ledger_get_handoff_status (current_agent: \"Documentation\")."
  ]
}

## Assistant

There's a WP-006 also awaiting documentation. However, the user's instructions are explicit: **CRITICAL: Every MCP tool call MUST use `work_package_id=WP-004`. Do NOT reference or operate on any other work package.**

My work on WP-004 is fully complete. Let me check handoff:

```json
{
  "id": "toolu_014jUsDcSPMDy9HNYJXET8bN",
  "caller": {
    "type": "direct"
  },
  "input": {
    "current_agent": "Documentation",
    "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights"
  },
  "name": "ledger_get_handoff_status",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_handoff_status` (id: `toolu_014jUsDcSPMDy9HNYJXET8bN`)

```json
{
  "current_agent": "Documentation",
  "project_path": "/Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights"
}
```

## Tool Result

{
  "current_agent": "Documentation",
  "next_agent": "Developer",
  "status": "READY_FOR_DEVELOPER",
  "details": "Documentation complete for all reviewed work packages. 4 work package(s) still need earlier stages: WP-004, WP-005, WP-006, WP-007. Hand back to Developer.",
  "auto_handoff": {
    "agent_name": "3 - Developer v3.6.1",
    "agent_id": "ledger-3-dev",
    "prompt": "@ledger-3-dev\nProject path: /Users/smordziol/Webserver/libraries/ai-insights-dev/docs/agents/plans/2026-03-26-move-ledger-plugin-to-ai-insights"
  }
}

## Assistant

```json
{
  "id": "toolu_01L9FY9tcB6g2shRBbWEewy1",
  "caller": {
    "type": "direct"
  },
  "input": {
    "todos": [
      {
        "content": "Update docs/plugins.md \u2014 remove ledger plugin section, add migration note",
        "status": "completed"
      },
      {
        "content": "Update api-surface.md \u2014 remove ledger plugin section",
        "status": "completed"
      },
      {
        "content": "Update file-tree.md \u2014 remove ledger plugin entries",
        "status": "completed"
      },
      {
        "content": "Update data-flows.md \u2014 remove ledger-specific references if any",
        "status": "completed"
      },
      {
        "content": "Update CHANGELOG.md \u2014 add 2.0.0 breaking change entry",
        "status": "completed"
      },
      {
        "content": "Complete documentation pipeline in ledger",
        "status": "completed"
      }
    ]
  },
  "name": "write_todos",
  "type": "tool_use"
}
```

**Tool call:** `write_todos` (id: `toolu_01L9FY9tcB6g2shRBbWEewy1`)

```json
{
  "todos": [
    {
      "content": "Update docs/plugins.md \u2014 remove ledger plugin section, add migration note",
      "status": "completed"
    },
    {
      "content": "Update api-surface.md \u2014 remove ledger plugin section",
      "status": "completed"
    },
    {
      "content": "Update file-tree.md \u2014 remove ledger plugin entries",
      "status": "completed"
    },
    {
      "content": "Update data-flows.md \u2014 remove ledger-specific references if any",
      "status": "completed"
    },
    {
      "content": "Update CHANGELOG.md \u2014 add 2.0.0 breaking change entry",
      "status": "completed"
    },
    {
      "content": "Complete documentation pipeline in ledger",
      "status": "completed"
    }
  ]
}
```

## Tool Result

Updated todo list to [{'content': 'Update docs/plugins.md — remove ledger plugin section, add migration note', 'status': 'completed'}, {'content': 'Update api-surface.md — remove ledger plugin section', 'status': 'completed'}, {'content': 'Update file-tree.md — remove ledger plugin entries', 'status': 'completed'}, {'content': 'Update data-flows.md — remove ledger-specific references if any', 'status': 'completed'}, {'content': 'Update CHANGELOG.md — add 2.0.0 breaking change entry', 'status': 'completed'}, {'content': 'Complete documentation pipeline in ledger', 'status': 'completed'}]

## Assistant

WP-004 documentation is complete. Here's a summary of what was done:

---

**WP-004 — Documentation complete** ✅

**Files updated in `ai-persona-builder-STABLE`:**

1. **`docs/plugins.md`** — Removed the full Ledger Plugin detail section (~300 lines covering `ledgerPlugin()`, `LedgerPluginOptions`, exported types, renderers, validators, frontmatter constants, and plugin patterns). Replaced with a concise `## Ledger Plugin — Migrated` notice pointing consumers to the ai-insights-dev workspace. The `PersonaBuildPlugin` interface documentation and all three plugin examples are preserved.

2. **`docs/agents/project-manifest/api-surface.md`** — Removed the entire `## Ledger Plugin — @mistralys/persona-builder/plugins/ledger` section, including all exported symbols (`ledgerPlugin`, `LedgerPluginOptions`, `RosterEntry`, `McpToolEntry`, `renderRoster`, `renderMcpToolsTable`, `validateRole`, `validateNoteOnlyGuard`, `FRONTMATTER_LEDGER_VSCODE`, `FRONTMATTER_LEDGER_CC`).

3. **`docs/agents/project-manifest/file-tree.md`** — Removed the `src/plugins/ledger/` subtree (5 module files). Updated `tsup.config.ts` annotation from "three entry points" to "two entry points".

4. **`docs/agents/project-manifest/data-flows.md`** — No changes needed; contained no ledger-specific references.

5. **`CHANGELOG.md`** — Added `[2.0.0] - Upcoming` entry with a `### Breaking Changes` section documenting the removal of the `./plugins/ledger` sub-path export and migration to ai-insights-dev.

**Handoff:**

```
CURRENT AGENT: Documentation
NEXT AGENT: Developer
STATUS: READY_FOR_DEVELOPER
```

---

## Token Usage

| Metric | Count |
| ------ | ----- |
| Input Tokens | 1889888 |
| Output Tokens | 12773 |
| Total Tokens | 1902661 |

