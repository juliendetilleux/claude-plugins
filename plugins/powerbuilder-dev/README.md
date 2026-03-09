# powerbuilder-dev — Claude Code Plugin

A Claude Code plugin providing skills, hooks, and a CLAUDE.md generator for PowerBuilder 2025 development.

## Overview

This plugin integrates with the `@pb-toolkit/mcp-server` (21 tools) to give Claude structured workflows for reading, modifying, debugging, and creating PowerBuilder source files.

## Structure

```
claude-plugin/
├── .claude-plugin/
│   └── plugin.json          # Plugin metadata
├── skills/
│   ├── pb-modify/
│   │   └── SKILL.md         # Modify PB source — complete 5-step workflow
│   ├── pb-debug/
│   │   └── SKILL.md         # Debug PB bugs — systematic diagnosis
│   ├── pb-analyze/
│   │   └── SKILL.md         # Analyze architecture, inheritance, dependencies
│   ├── pb-create/
│   │   └── SKILL.md         # Create new PB objects with correct conventions
│   ├── pmix-onboard/
│   │   └── SKILL.md         # Auto-onboard new PMIX client projects
│   ├── pmix-navigate/
│   │   └── SKILL.md         # Answer PMIX questions via RAG search
│   ├── pmix-flux/
│   │   └── SKILL.md         # Document PMIX business processes via RAG
│   └── pmix-impact/
│       └── SKILL.md         # Analyze modification impact on PMIX
├── commands/
│   └── pb-setup.md          # /pb-setup slash command — CLAUDE.md generator
└── hooks/
    └── hooks.json           # PostToolUse hook — reminds to compile after .sr* edits
```

## Skills

### pb-modify (Priority 1)

The main skill. Activates whenever Claude modifies `.srw`, `.srd`, `.sru`, or `.srm` files.

Enforces a 5-step workflow:
1. Read the target object (pb_read_object)
2. Check inheritance chain (pb_get_inheritance)
3. Check dependencies (pb_get_dependencies)
4. Apply modification (pb_modify_script — creates .bak, verifies uniqueness)
5. Validate syntax (pb_validate_syntax) then compile (pb_compile)

Also documents all PowerScript naming conventions, file structure rules, and the sections that must never be modified (forward, on create, on destroy).

### pb-debug

Systematic debugging workflow:
- Locate code via pb_search_code and pb_get_call_graph
- Analyze context via pb_get_inheritance and pb_get_dependencies
- Common patterns: null reference, SQL errors, event ordering issues
- Fix using pb-modify skill

### pb-analyze

Architecture analysis workflow:
- pb_get_object_summary for quick overview
- pb_read_object for full source
- pb_get_inheritance for hierarchy
- pb_get_dependencies for usage
- pb_get_call_graph for function tracing
- pb_get_project_structure for module layout

### pb-create

New object creation workflow:
- Identify correct ancestor via pb_get_inheritance
- Identify correct library via pb_get_project_structure
- Create via pb_create_object (generates proper .sr* template)
- Verify and compile

### pmix-onboard (NEW in v0.2.0)

Automatic onboarding for new PMIX client projects:
- Triggered when `.pmix-client.json` doesn't exist at project root
- Scans custom libraries (`_sysxtra`, `_cust2`, `Cust_*`)
- Reads `uo_cust_prg_id` to identify the client
- Indexes custom code in the RAG via `pmix_reindex`
- Generates `.pmix-client.json` marker file
- Displays a summary of all client customizations

### pmix-navigate

Answers any PMIX ERP question using the RAG knowledge base:
- Uses `pmix_search` for hybrid full-text + semantic search
- Uses `pmix_lookup` for detailed table/object info
- Checks both standard and custom (client-specific) layers

### pmix-flux

Documents PMIX business processes and workflows:
- Searches RAG for process playbooks (12 macro-flows documented)
- Details tables, windows, DataWindows, and validations per step
- Identifies side effects (accounting entries, stock movements, prints)

### pmix-impact

Analyzes the impact of modifying a PMIX object, table, or process:
- Combines RAG search with PB dependency/inheritance analysis
- Checks client customizations that may be affected
- Classifies risk (critical/medium/low) and recommends tests

## Commands

### /pb-setup [solution-path]

Generates a comprehensive CLAUDE.md for a PowerBuilder project by:
1. Analyzing project structure (pb_get_project_structure)
2. Discovering the inheritance tree from ancestor objects
3. Identifying naming conventions by sampling object names
4. Reading the .pbproj file for build configuration

The generated CLAUDE.md includes: project overview, architecture, naming conventions, build/deploy commands, development workflow, and key objects.

## Hooks

A lightweight PostToolUse hook fires after any Edit or Write tool call. If the modified file matches `.sr[wdumafsjpq]` (any PowerBuilder source extension), it prints a reminder to run `pb_compile` after all modifications are complete.

Heavy validation is deferred to the `pb_validate_syntax` MCP tool, keeping the hook fast and non-blocking.

## MCP Tools Referenced

All 21 tools from `@pb-toolkit/mcp-server`:

| Tool | Used by |
|------|---------|
| pb_list_objects | pb-analyze |
| pb_read_object | pb-modify, pb-debug, pb-analyze |
| pb_search_code | pb-debug |
| pb_get_project_structure | pb-analyze, pb-create, pb-setup |
| pb_refresh_cache | (cache management) |
| pb_get_datawindow_sql | pb-analyze, pb-debug |
| pb_get_inheritance | pb-modify, pb-debug, pb-analyze, pb-create |
| pb_get_dependencies | pb-modify, pb-debug, pb-analyze |
| pb_get_object_summary | pb-modify, pb-analyze |
| pb_get_call_graph | pb-debug, pb-analyze |
| pb_modify_script | pb-modify |
| pb_create_object | pb-create |
| pb_compile | pb-modify, pb-create |
| pb_validate_syntax | pb-modify |
| pb_launch_app | (visual testing) |
| pb_screenshot_window | (visual testing) |
| pb_list_controls | (visual testing) |
| pb_dw_get_columns | (visual testing — DW layout) |
| pb_interact_control | (visual testing — actions) |
| pb_save_reference | (visual regression) |
| pb_visual_compare | (visual regression) |

## Requirements

- Claude Code with plugin support enabled
- `@pb-toolkit/mcp-server` running and connected (provides all pb_* tools)
- `PB_SOLUTION_PATH` environment variable set to the PowerBuilder solution root
- `PB_PROJECT_PATH` set to the .pbproj file path
- `PB_OUTPUT_EXE` set to the target .exe path
