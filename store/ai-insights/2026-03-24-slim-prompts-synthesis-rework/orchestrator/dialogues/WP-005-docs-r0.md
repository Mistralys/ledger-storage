# Dialogue — docs / WP-005

| Field | Value |
| ----- | ----- |
| Stage | `docs` |
| WP ID | `WP-005` |
| Captured | 2026-03-24T15:44:47.882298+00:00 |

## Human

**Project path:** /Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework
**Work package:** WP-005

**CRITICAL — EVERY MCP TOOL CALL MUST include `project_path='/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework'`.**
Omitting `project_path` from any tool call will cause it to fail immediately.


## Assistant

I'll start by checking the project status and getting my next action.

```json
{
  "id": "toolu_01HfHZJ3SjfdvLBWt2QarJNS",
  "caller": {
    "type": "direct"
  },
  "input": {
    "agent_role": "Documentation",
    "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework"
  },
  "name": "ledger_get_next_action",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_next_action` (id: `toolu_01HfHZJ3SjfdvLBWt2QarJNS`)

```json
{
  "agent_role": "Documentation",
  "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework"
}
```

## Tool Result

{
  "action": "WRITE_DOCS",
  "work_package_id": "WP-005",
  "reason": "Work package WP-005 has PASS code-review pipeline. Write or update documentation.",
  "next_steps": [
    "1. Call ledger_begin_work (work_package_id: \"WP-005\", type: \"documentation\", agent_role: \"Documentation\").",
    "2. Call ledger_get_work_package to review implementation artifacts and review comments.",
    "3. Update documentation, README files, and inline docs as needed.",
    "4. Call ledger_complete_pipeline (work_package_id: \"WP-005\", type: \"documentation\", status: PASS/FAIL, summary, artifacts, comments, acceptance_criteria_updates).",
    "5. Call ledger_get_handoff_status (current_agent: \"Documentation\")."
  ],
  "handoff_notes": [
    "All four acceptance criteria are met and verified. A minor Fix-Forward was applied to align Manifest Sections table order with the file tree (constraints before api-surface). Documentation agent: check whether the constraints.md file itself has adequate introductory prose or needs a brief overview paragraph added."
  ]
}

## Assistant

```json
{
  "id": "toolu_01UQrvtn63D2X3QHhTXXfFCV",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-005",
    "type": "documentation",
    "agent_role": "Documentation",
    "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework"
  },
  "name": "ledger_begin_work",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01FHPYp75pecPXJxpMv2A4Ss",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-005",
    "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework"
  },
  "name": "ledger_get_work_package",
  "type": "tool_use"
}
```

**Tool call:** `ledger_begin_work` (id: `toolu_01UQrvtn63D2X3QHhTXXfFCV`)

```json
{
  "work_package_id": "WP-005",
  "type": "documentation",
  "agent_role": "Documentation",
  "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework"
}
```

**Tool call:** `ledger_get_work_package` (id: `toolu_01FHPYp75pecPXJxpMv2A4Ss`)

```json
{
  "work_package_id": "WP-005",
  "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework"
}
```

## Tool Result

{
  "work_package_id": "WP-005",
  "work_package_file": "work/WP-005.md",
  "status": "IN_PROGRESS",
  "assigned_to": "Documentation",
  "dependencies": [
    "WP-001"
  ],
  "acceptance_criteria": [
    {
      "criterion": "orchestrator/docs/agents/project-manifest/README.md Manifest Sections table includes a row for constraints.md with a working relative link",
      "met": true
    },
    {
      "criterion": "The file tree in the README lists constraints.md under the project-manifest/ directory",
      "met": true
    },
    {
      "criterion": "The inline \"Constraints & Conventions\" numbered list has been replaced with a forward reference to the standalone constraints.md file (no longer duplicated inline)",
      "met": true
    },
    {
      "criterion": "The ## Constraints & Conventions section heading is preserved (anchor not broken)",
      "met": true
    }
  ],
  "active_pipeline_stages": [
    "implementation",
    "code-review",
    "documentation"
  ],
  "revision": 0,
  "pipelines": [
    {
      "type": "implementation",
      "status": "PASS",
      "started_at": "2026-03-24T15:42:10Z",
      "completed_at": "2026-03-24T15:42:56Z",
      "duration_ms": 46000,
      "summary": [
        "Added missing `constraints.md` row to the Manifest Sections table in orchestrator/docs/agents/project-manifest/README.md. The other three acceptance criteria (file tree entry, forward-reference body replacing inline numbered list, preserved heading) were already satisfied by prior work (likely WP-001 edits). The single edit inserts a new table row with Section=`**Constraints & Conventions**`, Document=`[project-manifest/constraints.md](constraints.md)` (relative link resolves correctly from the README's directory), and a brief Contents description matching the WP spec."
      ],
      "artifacts": {
        "files_modified": [
          "orchestrator/docs/agents/project-manifest/README.md"
        ]
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-24T15:42:56Z",
          "note": "The file tree in the README shows constraints.md before api-surface.md (alphabetical/insertion order), but the Manifest Sections table places the constraints row after api-surface.md. The ordering is not wrong, but making both consistent (e.g., constraints before api-surface in both) would aid readability. Out of scope for this WP."
        }
      ]
    },
    {
      "type": "code-review",
      "status": "PASS",
      "started_at": "2026-03-24T15:43:45Z",
      "completed_at": "2026-03-24T15:44:25Z",
      "duration_ms": 40000,
      "summary": [
        "All four acceptance criteria verified against the live file. The Developer's single table-row insertion is correct and complete. The relative link `constraints.md` resolves correctly (constraints.md is a sibling of README.md in the same directory, confirmed by ls). Applied one Fix-Forward fix: reordered the Manifest Sections table so that Constraints & Conventions appears before API Surface (manifest), matching the file tree order. This is a purely cosmetic, non-behavioral change that improves consistency between the two sections."
      ],
      "artifacts": {
        "files_modified": [
          "orchestrator/docs/agents/project-manifest/README.md"
        ]
      },
      "comments": [
        {
          "type": "reviewer-applied-fix",
          "priority": "low",
          "timestamp": "2026-03-24T15:44:25Z",
          "note": "Reordered Manifest Sections table rows so Constraints & Conventions (constraints.md) appears immediately before API Surface (manifest) (api-surface.md). This matches the file tree ordering in the same README (lines 90–91: constraints.md listed before api-surface.md). No links, anchors, or behavioral logic changed — purely cosmetic consistency improvement."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-24T15:44:25Z",
          "note": "No blocking issues found. The implementation is minimal, focused, and correct. The Developer correctly identified that three of four ACs were already satisfied by WP-001 and made only the one necessary change."
        }
      ]
    },
    {
      "type": "documentation",
      "status": "IN_PROGRESS",
      "started_at": "2026-03-24T15:44:53Z",
      "summary": []
    }
  ],
  "status_changed_at": "2026-03-24T15:42:10Z",
  "last_updated": "2026-03-24T15:44:53Z",
  "handoff_notes": [
    {
      "from_agent": "Developer",
      "to_agent": "Reviewer",
      "timestamp": "2026-03-24T15:42:56Z",
      "notes": [
        "All four acceptance criteria are met. The only actual change needed was inserting the Manifest Sections table row — the other three criteria were already satisfied by prior work on WP-001. Reviewer should verify the relative link `[project-manifest/constraints.md](constraints.md)` is correct from the README location (it is — constraints.md is a sibling of README.md in the same directory)."
      ]
    },
    {
      "from_agent": "Reviewer",
      "to_agent": "Documentation",
      "timestamp": "2026-03-24T15:44:25Z",
      "notes": [
        "All four acceptance criteria are met and verified. A minor Fix-Forward was applied to align Manifest Sections table order with the file tree (constraints before api-surface). Documentation agent: check whether the constraints.md file itself has adequate introductory prose or needs a brief overview paragraph added."
      ]
    }
  ],
  "claimed": false
}

## Tool Result

{
  "work_package_id": "WP-005",
  "work_package_file": "work/WP-005.md",
  "status": "IN_PROGRESS",
  "assigned_to": "Reviewer",
  "dependencies": [
    "WP-001"
  ],
  "acceptance_criteria": [
    {
      "criterion": "orchestrator/docs/agents/project-manifest/README.md Manifest Sections table includes a row for constraints.md with a working relative link",
      "met": true
    },
    {
      "criterion": "The file tree in the README lists constraints.md under the project-manifest/ directory",
      "met": true
    },
    {
      "criterion": "The inline \"Constraints & Conventions\" numbered list has been replaced with a forward reference to the standalone constraints.md file (no longer duplicated inline)",
      "met": true
    },
    {
      "criterion": "The ## Constraints & Conventions section heading is preserved (anchor not broken)",
      "met": true
    }
  ],
  "active_pipeline_stages": [
    "implementation",
    "code-review",
    "documentation"
  ],
  "revision": 0,
  "pipelines": [
    {
      "type": "implementation",
      "status": "PASS",
      "started_at": "2026-03-24T15:42:10Z",
      "completed_at": "2026-03-24T15:42:56Z",
      "duration_ms": 46000,
      "summary": [
        "Added missing `constraints.md` row to the Manifest Sections table in orchestrator/docs/agents/project-manifest/README.md. The other three acceptance criteria (file tree entry, forward-reference body replacing inline numbered list, preserved heading) were already satisfied by prior work (likely WP-001 edits). The single edit inserts a new table row with Section=`**Constraints & Conventions**`, Document=`[project-manifest/constraints.md](constraints.md)` (relative link resolves correctly from the README's directory), and a brief Contents description matching the WP spec."
      ],
      "artifacts": {
        "files_modified": [
          "orchestrator/docs/agents/project-manifest/README.md"
        ]
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-24T15:42:56Z",
          "note": "The file tree in the README shows constraints.md before api-surface.md (alphabetical/insertion order), but the Manifest Sections table places the constraints row after api-surface.md. The ordering is not wrong, but making both consistent (e.g., constraints before api-surface in both) would aid readability. Out of scope for this WP."
        }
      ]
    },
    {
      "type": "code-review",
      "status": "PASS",
      "started_at": "2026-03-24T15:43:45Z",
      "completed_at": "2026-03-24T15:44:25Z",
      "duration_ms": 40000,
      "summary": [
        "All four acceptance criteria verified against the live file. The Developer's single table-row insertion is correct and complete. The relative link `constraints.md` resolves correctly (constraints.md is a sibling of README.md in the same directory, confirmed by ls). Applied one Fix-Forward fix: reordered the Manifest Sections table so that Constraints & Conventions appears before API Surface (manifest), matching the file tree order. This is a purely cosmetic, non-behavioral change that improves consistency between the two sections."
      ],
      "artifacts": {
        "files_modified": [
          "orchestrator/docs/agents/project-manifest/README.md"
        ]
      },
      "comments": [
        {
          "type": "reviewer-applied-fix",
          "priority": "low",
          "timestamp": "2026-03-24T15:44:25Z",
          "note": "Reordered Manifest Sections table rows so Constraints & Conventions (constraints.md) appears immediately before API Surface (manifest) (api-surface.md). This matches the file tree ordering in the same README (lines 90–91: constraints.md listed before api-surface.md). No links, anchors, or behavioral logic changed — purely cosmetic consistency improvement."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-24T15:44:25Z",
          "note": "No blocking issues found. The implementation is minimal, focused, and correct. The Developer correctly identified that three of four ACs were already satisfied by WP-001 and made only the one necessary change."
        }
      ]
    }
  ],
  "status_changed_at": "2026-03-24T15:42:10Z",
  "last_updated": "2026-03-24T15:44:25Z",
  "handoff_notes": [
    {
      "from_agent": "Developer",
      "to_agent": "Reviewer",
      "timestamp": "2026-03-24T15:42:56Z",
      "notes": [
        "All four acceptance criteria are met. The only actual change needed was inserting the Manifest Sections table row — the other three criteria were already satisfied by prior work on WP-001. Reviewer should verify the relative link `[project-manifest/constraints.md](constraints.md)` is correct from the README location (it is — constraints.md is a sibling of README.md in the same directory)."
      ]
    },
    {
      "from_agent": "Reviewer",
      "to_agent": "Documentation",
      "timestamp": "2026-03-24T15:44:25Z",
      "notes": [
        "All four acceptance criteria are met and verified. A minor Fix-Forward was applied to align Manifest Sections table order with the file tree (constraints before api-surface). Documentation agent: check whether the constraints.md file itself has adequate introductory prose or needs a brief overview paragraph added."
      ]
    }
  ]
}

## Assistant

Good. The key task from the Reviewer's handoff note is: **check whether `constraints.md` has adequate introductory prose or needs a brief overview paragraph added.** Let me read the relevant files now.

```json
{
  "id": "toolu_01CpJRqPxtHQxBBDBDy6vKXs",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/orchestrator/docs/agents/project-manifest/constraints.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

```json
{
  "id": "toolu_01GkXjWsU95aHtFnxLcttFcQ",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/orchestrator/docs/agents/project-manifest/README.md"
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01CpJRqPxtHQxBBDBDy6vKXs`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/orchestrator/docs/agents/project-manifest/constraints.md"
}
```

**Tool call:** `read_file` (id: `toolu_01GkXjWsU95aHtFnxLcttFcQ`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/orchestrator/docs/agents/project-manifest/README.md"
}
```

## Tool Result

     1	# Constraints & Conventions
     2	
     3	This document codifies established rules, conventions, and non-obvious gotchas for the **AI Insights Orchestrator**.
     4	
     5	### Constraint Entry Format
     6	
     7	New constraint entries should follow this structure:
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
    19	## Prompt Architecture Constraints
    20	
    21	### 1. Persona Files Are the Source of Truth for Agent Behaviour
    22	
    23	**Rule:** All identity declarations, workflow step enumerations, and MCP tool-call instructions live exclusively in persona system prompts (`personas/ledger/claude-code/`). User-turn prompts in `_build_*_prompt()` functions must contain only runtime context that the persona file cannot know: concrete `project_path`, `wp_id`, plan content, and the injection-safety warning. Any change to agent behaviour must be made in the persona source files, **not** in prompt builder functions.
    24	
    25	**Rationale:** Splitting identity from runtime context keeps persona files reviewable, versionable, and reusable across different orchestration surfaces without coupling them to Python implementation details.
    26	
    27	**Anti-pattern:**
    28	```python
    29	# ❌ WRONG — workflow instructions embedded in the user-turn prompt
    30	def _build_developer_prompt(project_path: str, wp_id: str) -> str:
    31	    return f"""
    32	    CRITICAL — EVERY MCP TOOL CALL MUST include `project_path='{project_path}'`.
    33	
    34	    Your workflow:
    35	    1. Call ledger_get_next_action with agent_role: "Developer"
    36	    2. Read the WP spec
    37	    3. Implement the changes
    38	    ...
    39	    """
    40	```
    41	
    42	**Correct pattern:**
    43	```python
    44	# ✅ CORRECT — user-turn prompt carries only runtime context
    45	def _build_developer_prompt(project_path: str, wp_id: str) -> str:
    46	    return f"""**Project path:** {project_path}
    47	**Work package:** {wp_id}
    48	
    49	**CRITICAL — EVERY MCP TOOL CALL MUST include `project_path='{project_path}'`.**
    50	Omitting `project_path` from any tool call will cause it to fail immediately.
    51	"""
    52	```
    53	
    54	---
    55	
    56	### 2. The `project_path` Injection-Safety Warning Is Permanent
    57	
    58	**Rule:** The user-turn prompt must always include the verbatim injection-safety warning: `**CRITICAL — EVERY MCP TOOL CALL MUST include \`project_path='...'\`.**` This warning must never be removed or weakened. Persona Markdown files are static and cannot contain runtime values, so this runtime reminder lives in the user-turn prompt.
    59	
    60	**Rationale:** Without the injection-safety warning the agent may omit `project_path` from MCP tool calls, causing every ledger operation to fail. The warning is a runtime guard against path manipulation and a reminder of the MCP server's `project_path` requirement.
    61	
    62	---
    63	
    64	### 3. Prompt Templates Are Structurally Uniform Within Their Category
    65	
    66	**Rule:** The six WP-scoped prompt builder functions (`_build_developer_prompt`, `_build_qa_prompt`, `_build_security_auditor_prompt`, `_build_reviewer_prompt`, `_build_release_engineer_prompt`, `_build_docs_prompt`) must remain structurally identical: same f-string layout, same fields (`project_path`, `wp_id`, injection-safety warning), same annotations. Any change to the minimal prompt pattern must be applied consistently across all six. The PM and synthesis templates are documented exceptions with justified divergences (PM adds plan content; synthesis adds a project-scoped summary and omits `wp_id`).
    67	
    68	**Rationale:** Structural uniformity makes the prompt layer auditable at a glance and prevents silent divergence between nodes that should behave identically.
    69	
    70	---
    71	
    72	## Supervisor & Routing Constraints
    73	
    74	### 4. No LLM Calls in the Supervisor
    75	
    76	**Rule:** The supervisor node must not make LLM calls. All routing decisions must come from the MCP server's `ledger_get_next_action` tool. The supervisor is a pure-Python router.
    77	
    78	**Rationale:** LLM-based routing introduces non-determinism into an otherwise deterministic pipeline. Delegating routing to the ledger tools ensures the supervisor's behaviour is fully specified by the workflow manifest.
    79	
    80	---
    81	
    82	### 5. Manifest-Derived Constants
    83	
    84	**Rule:** `PIPELINE_ROLES`, `PIPELINE_SEQUENCE`, and action→role maps in `src/config.py` must be derived from `shared/workflow-manifest.json` at import time. Never hardcode role names or pipeline ordering as bare string literals.
    85	
    86	**Rationale:** The workflow manifest is the canonical source of pipeline ordering and role naming. Hardcoded constants drift silently when the manifest is updated.
    87	
    88	---
    89	
    90	### 6. Circuit-Breaker Threshold: 3 Consecutive Failures
    91	
    92	**Rule:** A work package that accumulates ≥3 consecutive stage failures must be skipped for the remainder of the run. The threshold value must be read from configuration, not hardcoded.
    93	
    94	**Rationale:** Without a circuit-breaker a pathologically failing WP will stall the entire orchestration run indefinitely.
    95	
    96	---
    97	
    98	## Node Implementation Constraints
    99	
   100	### 7. Stage Node Isolation

## Tool Result

     1	# Orchestrator — Project Manifest
     2	
     3	> Manifest hub for the **AI Insights Orchestrator** — a headless, deterministic alternative to IDE-based agent workflows using LangGraph + Deep Agents.
     4	
     5	---
     6	
     7	## Quick Reference
     8	
     9	| Property | Value |
    10	|----------|-------|
    11	| **Language** | Python 3.11+ |
    12	| **Runtime** | CPython |
    13	| **Architecture** | LangGraph StateGraph + Deep Agents |
    14	| **Package Manager** | pip (setuptools) |
    15	| **Test Framework** | pytest (374 tests) |
    16	| **Entry Point** | `orchestrate` CLI (`src/cli.py`) |
    17	
    18	### Development Commands
    19	
    20	```bash
    21	cd orchestrator
    22	pip install -e ".[dev,anthropic]"   # Install with dev + Anthropic extras
    23	pytest                               # Run all tests
    24	pytest -m "not live"                 # Skip tests requiring API keys
    25	ruff check src/ tests/               # Lint
    26	```
    27	
    28	---
    29	
    30	## Manifest Sections
    31	
    32	The orchestrator's documentation lives in `orchestrator/docs/`. The documents below together form its project manifest.
    33	
    34	| Section | Document | Contents |
    35	|---------|----------|----------|
    36	| **Overview & Usage** | [README.md](../../../README.md) | Prerequisites, installation, configuration, CLI reference, architecture overview, troubleshooting |
    37	| **Architecture & Data Flows** | [architecture.md](../../architecture.md) | Stage node lifecycle, MCP tool wrapping, `WorkflowState` fields, JSONL log entry types |
    38	| **Routing Logic** | [supervisor-routing.md](../../supervisor-routing.md) | Deterministic supervisor algorithm, special exits, action sets, circuit-breaker mechanics |
    39	| **Public API Surface** | [public-api.md](../../public-api.md) | CLI entry point, graph construction, supervisor factory, utility functions |
    40	| **Constraints & Conventions** | [project-manifest/constraints.md](constraints.md) | Numbered constraints and conventions governing orchestrator development: prompt architecture rules, LLM boundaries, circuit-breaker, cross-platform policy |
    41	| **API Surface (manifest)** | [project-manifest/api-surface.md](api-surface.md) | Quick-reference: 16 JSONL event types, enriched fields, `_format_duration`, `parse_tool_response`, progress-tracking state fields |
    42	| **Log Schema** | [jsonl-log-schema.md](../../jsonl-log-schema.md) | JSONL schema reference: 16 event types, full field reference, duration conventions, JSON examples |
    43	| **Smoke Testing** | [smoke-testing.md](../../smoke-testing.md) | Dispatch loop verification runbook |
    44	
    45	---
    46	
    47	## Tech Stack
    48	
    49	| Layer | Technology | Purpose |
    50	|-------|-----------|---------|
    51	| Graph framework | LangGraph ≥0.4 | StateGraph-based workflow with deterministic routing |
    52	| Agent execution | Deep Agents ≥0.3 (via LangChain) | Coding-agent execution within each pipeline stage |
    53	| MCP integration | langchain-mcp-adapters ≥0.2 | Wraps MCP tools for LangChain tool interface |
    54	| LLM providers | langchain-anthropic / langchain-google-genai | Claude (Anthropic) or Gemini (Google) |
    55	| Checkpointing | langgraph-checkpoint-sqlite | SQLite-backed run resume via `--resume` |
    56	| Configuration | python-dotenv | `.env`-based config with auto-detected LLM provider |
    57	| Testing | pytest + pytest-asyncio | Async-aware tests with integration and live marks |
    58	| Linting | ruff | Line-length 100, target Python 3.11 |
    59	
    60	### Architectural Patterns
    61	
    62	- **Deterministic supervisor**: Pure-Python router with no LLM calls — delegates all routing to the MCP server's `ledger_get_next_action` tool.
    63	- **Stage node factories**: Each of the 8 stages (pm, developer, qa, security_auditor, reviewer, release_engineer, docs, synthesis) is a factory-generated node that loads a persona prompt, wraps MCP tools, and creates a Deep Agent.
    64	- **Manifest-derived constants**: Pipeline routing maps and role names are derived from `shared/workflow-manifest.json` at import time.
    65	- **Cross-platform file locking**: `msvcrt` (Windows) / `fcntl` (Unix) for the JSONL run log.
    66	
    67	---
    68	
    69	## Constraints & Conventions
    70	
    71	The authoritative constraint list has been promoted to a dedicated file:
    72	
    73	> **[project-manifest/constraints.md](constraints.md)** — 11 numbered constraints covering persona authority, injection-safety, prompt uniformity, LLM routing, manifest-derived constants, MCP pre-build, circuit-breaker, stage isolation, cross-platform locking, documentation-forward convention, and LangGraph config annotations.
    74	
    75	---
    76	
    77	## File Tree
    78	
    79	```
    80	orchestrator/
    81	├── pyproject.toml              # Package metadata, extras, scripts
    82	├── README.md                   # Full user-facing documentation
    83	├── requirements.txt            # Pinned dependencies
    84	├── changelog.md                # Version history
    85	├── module-context.yaml         # CTX Generator config
    86	├── docs/
    87	│   ├── agents/
    88	│   │   └── project-manifest/
    89	│   │       ├── README.md       # ← You are here
    90	│   │       ├── constraints.md  # Numbered constraint catalogue (11 rules)
    91	│   │       └── api-surface.md  # JSONL event types, enriched fields, utility refs
    92	│   ├── architecture.md         # Stage nodes, state management, log types
    93	│   ├── supervisor-routing.md   # Routing algorithm, exits, circuit-breaker
    94	│   ├── public-api.md           # Public functions and entry points
    95	│   ├── jsonl-log-schema.md     # Run log field reference
    96	│   └── smoke-testing.md        # Dispatch loop verification
    97	├── src/
    98	│   ├── __init__.py
    99	│   ├── cli.py                  # CLI entry point (orchestrate command)
   100	│   ├── config.py               # .env loading, provider detection, constants

## Assistant

```json
{
  "id": "toolu_01L1XonCUizvv42gbAGW6R2m",
  "caller": {
    "type": "direct"
  },
  "input": {
    "file_path": "/Users/smordziol/Webserver/tools/ai-insights/orchestrator/docs/agents/project-manifest/constraints.md",
    "offset": 100,
    "limit": 100
  },
  "name": "read_file",
  "type": "tool_use"
}
```

**Tool call:** `read_file` (id: `toolu_01L1XonCUizvv42gbAGW6R2m`)

```json
{
  "file_path": "/Users/smordziol/Webserver/tools/ai-insights/orchestrator/docs/agents/project-manifest/constraints.md",
  "offset": 100,
  "limit": 100
}
```

## Tool Result

   101	
   102	**Rule:** Each stage node must create its own Deep Agent instance per invocation. No state — including LLM client instances, MCP connections, or tool objects — may be shared between stage invocations.
   103	
   104	**Rationale:** Shared state between stage invocations introduces subtle coupling that makes failures hard to diagnose and prevents clean retry semantics.
   105	
   106	---
   107	
   108	### 8. Cross-Platform File Locking
   109	
   110	**Rule:** File locking for the JSONL run log must use `msvcrt` on Windows and `fcntl` on Unix. All path construction must use `pathlib.Path`, never bare string concatenation.
   111	
   112	**Rationale:** The orchestrator must run in CI environments on both Linux and Windows. Platform-specific locking ensures log integrity without blocking on a missing system call.
   113	
   114	---
   115	
   116	## LangGraph-Specific Constraints
   117	
   118	### 9. LangGraph Config Annotations Require `Optional[RunnableConfig]`
   119	
   120	**Rule:** In files that use `from __future__ import annotations`, always annotate LangGraph config parameters as `Optional[RunnableConfig]`, **not** `RunnableConfig | None`.
   121	
   122	**Rationale:** `from __future__ import annotations` causes Python to stringify all type hints at parse time. The union syntax `RunnableConfig | None` becomes the string `"RunnableConfig | None"`, which LangGraph's config injection does not recognise. `Optional[RunnableConfig]` produces `"Optional[RunnableConfig]"`, which is in the allowlist.
   123	
   124	**Symptom:** `get_run_logger: config is None` warnings; JSONL events only flushed at run end rather than incrementally.
   125	
   126	**Anti-pattern:**
   127	```python
   128	# ❌ WRONG — union syntax is stringified to an unrecognised form
   129	from __future__ import annotations
   130	from langchain_core.runnables import RunnableConfig
   131	
   132	async def node(state: WorkflowState, config: RunnableConfig | None = None) -> WorkflowState:
   133	    ...
   134	```
   135	
   136	**Correct pattern:**
   137	```python
   138	# ✅ CORRECT — Optional[] form is in LangGraph's annotation allowlist
   139	from __future__ import annotations
   140	from typing import Optional
   141	from langchain_core.runnables import RunnableConfig
   142	
   143	async def node(state: WorkflowState, config: Optional[RunnableConfig] = None) -> WorkflowState:
   144	    ...
   145	```
   146	
   147	---
   148	
   149	## Review & Documentation Conventions
   150	
   151	### 10. `documentation-forward` Is the Named Review-to-Documentation Handoff Convention
   152	
   153	**Rule:** When a code-review pipeline identifies documentation gaps, the reviewer must record them as structured pipeline comments with type `documentation-forward`. The documentation stage resolves these comments. This is the standard cross-pipeline handoff mechanism for documentation work identified during review.
   154	
   155	**Format:** Comment objects in the code-review pipeline result must use:
   156	```json
   157	{
   158	  "type": "documentation-forward",
   159	  "priority": "medium",
   160	  "note": "[documentation-forward] <description of documentation gap and suggested resolution>"
   161	}
   162	```
   163	
   164	**Rationale:** Naming the convention enforces a consistent, machine-readable handoff signal between the reviewer and documentation agents, preventing documentation gaps from being silently dropped when the code-review pipeline completes.
   165	
   166	**Who resolves it:** The documentation stage agent reads open `documentation-forward` comments from the most recent code-review pipeline and addresses each one before marking the WP complete.
   167	
   168	---
   169	
   170	## MCP Server Dependency
   171	
   172	### 11. MCP Server Must Be Pre-Built
   173	
   174	**Rule:** The orchestrator spawns the MCP server as a subprocess. `mcp-server/dist/index.js` must exist before any orchestration run begins. Use `node scripts/run-orchestrator.js` for automatic build-freshness checks rather than launching `orchestrator` directly.
   175	
   176	**Rationale:** The orchestrator has no fallback if the MCP server subprocess fails to start — all ledger operations will fail silently or with unhelpful errors.

## Assistant

The `constraints.md` file already has a solid introductory paragraph (lines 1–3). Let me assess the full picture:

- **`constraints.md`**: Has a clear `# Constraints & Conventions` heading, a good opening sentence ("This document codifies established rules, conventions, and non-obvious gotchas for the **AI Insights Orchestrator**."), and a well-structured `Constraint Entry Format` table explaining how entries are organized. The prose is adequate — no gap here.
- **`README.md`**: Already has the `constraints.md` table row in the Manifest Sections table (with the correct relative link), the file tree entry, the forward-reference body (replacing the inline list), and the preserved `## Constraints & Conventions` heading.

All four acceptance criteria are met in the live files. The Reviewer's forwarded note (check if `constraints.md` needs an intro paragraph) is addressed — the file already has adequate introductory prose. No documentation changes are needed.

```json
{
  "id": "toolu_01C3EMNcg9XPuT25tZreLnzZ",
  "caller": {
    "type": "direct"
  },
  "input": {
    "work_package_id": "WP-005",
    "type": "documentation",
    "agent_role": "Documentation",
    "status": "PASS",
    "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework",
    "summary": "Reviewed all documentation artifacts for WP-005. All four acceptance criteria are confirmed met in the live files: (1) Manifest Sections table includes a row for constraints.md with a working relative link; (2) the file tree lists constraints.md under project-manifest/; (3) the inline Constraints & Conventions numbered list has been replaced with a forward reference to constraints.md; (4) the ## Constraints & Conventions heading is preserved. Addressed the Reviewer's handoff note: constraints.md already has adequate introductory prose \u2014 the opening sentence clearly states the document's purpose and a Constraint Entry Format table explains the structure for new entries. No documentation changes were required.",
    "comments": [
      {
        "type": "improvement",
        "priority": "low",
        "note": "constraints.md has a clear h1 heading, a concise opening sentence ('This document codifies established rules, conventions, and non-obvious gotchas for the AI Insights Orchestrator.'), and a Constraint Entry Format table \u2014 introductory prose is sufficient. No additions needed."
      }
    ],
    "acceptance_criteria_updates": [
      {
        "criterion": "orchestrator/docs/agents/project-manifest/README.md Manifest Sections table includes a row for constraints.md with a working relative link",
        "met": true
      },
      {
        "criterion": "The file tree in the README lists constraints.md under the project-manifest/ directory",
        "met": true
      },
      {
        "criterion": "The inline \"Constraints & Conventions\" numbered list has been replaced with a forward reference to the standalone constraints.md file (no longer duplicated inline)",
        "met": true
      },
      {
        "criterion": "The ## Constraints & Conventions section heading is preserved (anchor not broken)",
        "met": true
      }
    ]
  },
  "name": "ledger_complete_pipeline",
  "type": "tool_use"
}
```

**Tool call:** `ledger_complete_pipeline` (id: `toolu_01C3EMNcg9XPuT25tZreLnzZ`)

```json
{
  "work_package_id": "WP-005",
  "type": "documentation",
  "agent_role": "Documentation",
  "status": "PASS",
  "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework",
  "summary": "Reviewed all documentation artifacts for WP-005. All four acceptance criteria are confirmed met in the live files: (1) Manifest Sections table includes a row for constraints.md with a working relative link; (2) the file tree lists constraints.md under project-manifest/; (3) the inline Constraints & Conventions numbered list has been replaced with a forward reference to constraints.md; (4) the ## Constraints & Conventions heading is preserved. Addressed the Reviewer's handoff note: constraints.md already has adequate introductory prose \u2014 the opening sentence clearly states the document's purpose and a Constraint Entry Format table explains the structure for new entries. No documentation changes were required.",
  "comments": [
    {
      "type": "improvement",
      "priority": "low",
      "note": "constraints.md has a clear h1 heading, a concise opening sentence ('This document codifies established rules, conventions, and non-obvious gotchas for the AI Insights Orchestrator.'), and a Constraint Entry Format table \u2014 introductory prose is sufficient. No additions needed."
    }
  ],
  "acceptance_criteria_updates": [
    {
      "criterion": "orchestrator/docs/agents/project-manifest/README.md Manifest Sections table includes a row for constraints.md with a working relative link",
      "met": true
    },
    {
      "criterion": "The file tree in the README lists constraints.md under the project-manifest/ directory",
      "met": true
    },
    {
      "criterion": "The inline \"Constraints & Conventions\" numbered list has been replaced with a forward reference to the standalone constraints.md file (no longer duplicated inline)",
      "met": true
    },
    {
      "criterion": "The ## Constraints & Conventions section heading is preserved (anchor not broken)",
      "met": true
    }
  ]
}
```

## Tool Result

{
  "work_package_id": "WP-005",
  "work_package_file": "work/WP-005.md",
  "status": "COMPLETE",
  "assigned_to": "Documentation",
  "dependencies": [
    "WP-001"
  ],
  "acceptance_criteria": [
    {
      "criterion": "orchestrator/docs/agents/project-manifest/README.md Manifest Sections table includes a row for constraints.md with a working relative link",
      "met": true
    },
    {
      "criterion": "The file tree in the README lists constraints.md under the project-manifest/ directory",
      "met": true
    },
    {
      "criterion": "The inline \"Constraints & Conventions\" numbered list has been replaced with a forward reference to the standalone constraints.md file (no longer duplicated inline)",
      "met": true
    },
    {
      "criterion": "The ## Constraints & Conventions section heading is preserved (anchor not broken)",
      "met": true
    }
  ],
  "active_pipeline_stages": [
    "implementation",
    "code-review",
    "documentation"
  ],
  "revision": 0,
  "pipelines": [
    {
      "type": "implementation",
      "status": "PASS",
      "started_at": "2026-03-24T15:42:10Z",
      "completed_at": "2026-03-24T15:42:56Z",
      "duration_ms": 46000,
      "summary": [
        "Added missing `constraints.md` row to the Manifest Sections table in orchestrator/docs/agents/project-manifest/README.md. The other three acceptance criteria (file tree entry, forward-reference body replacing inline numbered list, preserved heading) were already satisfied by prior work (likely WP-001 edits). The single edit inserts a new table row with Section=`**Constraints & Conventions**`, Document=`[project-manifest/constraints.md](constraints.md)` (relative link resolves correctly from the README's directory), and a brief Contents description matching the WP spec."
      ],
      "artifacts": {
        "files_modified": [
          "orchestrator/docs/agents/project-manifest/README.md"
        ]
      },
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-24T15:42:56Z",
          "note": "The file tree in the README shows constraints.md before api-surface.md (alphabetical/insertion order), but the Manifest Sections table places the constraints row after api-surface.md. The ordering is not wrong, but making both consistent (e.g., constraints before api-surface in both) would aid readability. Out of scope for this WP."
        }
      ]
    },
    {
      "type": "code-review",
      "status": "PASS",
      "started_at": "2026-03-24T15:43:45Z",
      "completed_at": "2026-03-24T15:44:25Z",
      "duration_ms": 40000,
      "summary": [
        "All four acceptance criteria verified against the live file. The Developer's single table-row insertion is correct and complete. The relative link `constraints.md` resolves correctly (constraints.md is a sibling of README.md in the same directory, confirmed by ls). Applied one Fix-Forward fix: reordered the Manifest Sections table so that Constraints & Conventions appears before API Surface (manifest), matching the file tree order. This is a purely cosmetic, non-behavioral change that improves consistency between the two sections."
      ],
      "artifacts": {
        "files_modified": [
          "orchestrator/docs/agents/project-manifest/README.md"
        ]
      },
      "comments": [
        {
          "type": "reviewer-applied-fix",
          "priority": "low",
          "timestamp": "2026-03-24T15:44:25Z",
          "note": "Reordered Manifest Sections table rows so Constraints & Conventions (constraints.md) appears immediately before API Surface (manifest) (api-surface.md). This matches the file tree ordering in the same README (lines 90–91: constraints.md listed before api-surface.md). No links, anchors, or behavioral logic changed — purely cosmetic consistency improvement."
        },
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-24T15:44:25Z",
          "note": "No blocking issues found. The implementation is minimal, focused, and correct. The Developer correctly identified that three of four ACs were already satisfied by WP-001 and made only the one necessary change."
        }
      ]
    },
    {
      "type": "documentation",
      "status": "PASS",
      "started_at": "2026-03-24T15:44:53Z",
      "completed_at": "2026-03-24T15:45:20Z",
      "duration_ms": 27000,
      "summary": [
        "Reviewed all documentation artifacts for WP-005. All four acceptance criteria are confirmed met in the live files: (1) Manifest Sections table includes a row for constraints.md with a working relative link; (2) the file tree lists constraints.md under project-manifest/; (3) the inline Constraints & Conventions numbered list has been replaced with a forward reference to constraints.md; (4) the ## Constraints & Conventions heading is preserved. Addressed the Reviewer's handoff note: constraints.md already has adequate introductory prose — the opening sentence clearly states the document's purpose and a Constraint Entry Format table explains the structure for new entries. No documentation changes were required."
      ],
      "comments": [
        {
          "type": "improvement",
          "priority": "low",
          "timestamp": "2026-03-24T15:45:20Z",
          "note": "constraints.md has a clear h1 heading, a concise opening sentence ('This document codifies established rules, conventions, and non-obvious gotchas for the AI Insights Orchestrator.'), and a Constraint Entry Format table — introductory prose is sufficient. No additions needed."
        }
      ]
    }
  ],
  "status_changed_at": "2026-03-24T15:45:20Z",
  "last_updated": "2026-03-24T15:45:20Z",
  "handoff_notes": [
    {
      "from_agent": "Developer",
      "to_agent": "Reviewer",
      "timestamp": "2026-03-24T15:42:56Z",
      "notes": [
        "All four acceptance criteria are met. The only actual change needed was inserting the Manifest Sections table row — the other three criteria were already satisfied by prior work on WP-001. Reviewer should verify the relative link `[project-manifest/constraints.md](constraints.md)` is correct from the README location (it is — constraints.md is a sibling of README.md in the same directory)."
      ]
    },
    {
      "from_agent": "Reviewer",
      "to_agent": "Documentation",
      "timestamp": "2026-03-24T15:44:25Z",
      "notes": [
        "All four acceptance criteria are met and verified. A minor Fix-Forward was applied to align Manifest Sections table order with the file tree (constraints before api-surface). Documentation agent: check whether the constraints.md file itself has adequate introductory prose or needs a brief overview paragraph added."
      ]
    }
  ],
  "auto_finalized": true
}

--- NEXT STEP ---
Pipeline PASS. WP WP-005 was auto-finalized to COMPLETE (all acceptance criteria met). Call ledger_get_handoff_status (current_agent: "Documentation") to confirm handoff.

Note: artifacts.files_modified is empty or absent. If you modified any files during this pipeline, declare them in artifacts.files_modified for a complete audit trail.

## Assistant

WP-005 is COMPLETE. Now checking handoff status.

```json
{
  "id": "toolu_01URoeHMWoEgV3WiC4M65qeX",
  "caller": {
    "type": "direct"
  },
  "input": {
    "agent_role": "Documentation",
    "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework"
  },
  "name": "ledger_get_next_action",
  "type": "tool_use"
}
```

**Tool call:** `ledger_get_next_action` (id: `toolu_01URoeHMWoEgV3WiC4M65qeX`)

```json
{
  "agent_role": "Documentation",
  "project_path": "/Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework"
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
    "details": "Documentation passed for 4 work package(s), but 1 work package(s) still need earlier stages: WP-003. Hand back to Developer.",
    "auto_handoff": {
      "agent_name": "3 - Developer v3.6.1",
      "agent_id": "ledger-3-dev",
      "prompt": "@ledger-3-dev\nProject path: /Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework"
    }
  }
}

## Assistant

WP-005 is complete. However, the ledger indicates WP-003 still needs earlier pipeline stages — handing back to the Developer.

```json
{
  "id": "toolu_01LS8cEBUiz7WSWfi8N9s4sM",
  "caller": {
    "type": "direct"
  },
  "input": {
    "subagent_type": "general-purpose",
    "description": "3-developer\n\n@ledger-3-dev\nProject path: /Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework"
  },
  "name": "task",
  "type": "tool_use"
}
```

**Tool call:** `task` (id: `toolu_01LS8cEBUiz7WSWfi8N9s4sM`)

```json
{
  "subagent_type": "general-purpose",
  "description": "3-developer\n\n@ledger-3-dev\nProject path: /Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework"
}
```

## Tool Result

All work packages on this project are already **COMPLETE**. There's nothing left for the Developer agent to do here. 🎉

The project is ready for the next stage:

---

**➡️ Hand off to Synthesis Agent:**

```
@ledger-7-synthesis
Project path: /Users/smordziol/Webserver/tools/ai-insights/docs/agents/plans/2026-03-24-slim-prompts-synthesis-rework
```

## Assistant

All work packages are complete. The project is ready for the Synthesis agent.

```
CURRENT AGENT: Documentation
NEXT AGENT: Synthesis
STATUS: ALL_COMPLETE — all WPs done, project ready for synthesis report
```

---

## Token Usage

| Metric | Count |
| ------ | ----- |
| Input Tokens | 209382 |
| Output Tokens | 1933 |
| Total Tokens | 211315 |

