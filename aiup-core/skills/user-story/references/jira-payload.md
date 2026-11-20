<!--
Copyright 2026 Andrei Kishkevich.
Licensed under the Apache License, Version 2.0. See LICENSE and NOTICE.
-->

# Jira Create Issue Payload

Template for the `mcp__jira__create_issue` call. Fill the placeholders with the story details before
sending.

## Fields

| Field | Value | Notes |
|-------|-------|-------|
| `project.key` | `YOUR_PROJECT_KEY` | Replace with the Jira project key, e.g. `PROJ` |
| `issuetype.name` | `Story` | Use `Story` or the type your Jira project defines |
| `summary` | Goal clause of the story (≤ 255 chars) | Short imperative: "Filter tasks by status" |
| `description` | Filled story template content | Full "As a … I want … so that …" + Documentation table |
| `labels` | `["FR-XXX"]` | The FR-* ID, so the issue traces back to the requirements catalog |
| `priority.name` | `Medium` (or the FR row's Priority value) | Maps to the FR's Priority column |

## Example JSON

```json
{
  "project": { "key": "YOUR_PROJECT_KEY" },
  "issuetype": { "name": "Story" },
  "summary": "Filter tasks by status",
  "description": "*As a team member, I want to filter tasks by status so that I can focus on relevant items.*\n\n## Documentation\n\n| Type | Reference | Location |\n|------|-----------|----------|\n| Requirement | FR-003 | docs/requirements.md |\n| Entity model | TASK | docs/entity_model.md |\n| Glossary | Task, Status | docs/glossary.md |\n| Use case | UC-004 Filter Tasks | docs/use_cases/UC-004-filter-tasks.md |",
  "labels": ["FR-003"],
  "priority": { "name": "Medium" }
}
```

## MCP Tool

This skill calls `mcp__jira__create_issue`. If the Jira MCP server uses a different tool name,
replace it in the SKILL.md Jira Integration section. Configure the server in `.mcp.json` under
a `jira` key.
