<!--
Copyright 2026 Andrei Kishkevich.
Licensed under the Apache License, Version 2.0. See LICENSE and NOTICE.
-->

# Jira Update Issue Payload

Template for the `mcp__jira__update_issue` call. Fill the placeholders before sending.

## Fields

| Field | Value | Notes |
|-------|-------|-------|
| `issueIdOrKey` | The issue key from $ARGUMENTS, e.g. `PROJ-42` | Never create a new issue |
| `fields.description` | The rebuilt story description | Full content from the story template |
| `fields.summary` | Updated only when the story sentence itself changed | Omit this field if the summary is unchanged |

## Example JSON

```json
{
  "issueIdOrKey": "PROJ-42",
  "fields": {
    "description": "*As a team member, I want to filter tasks by status so that I can focus on relevant items.*\n\n## Documentation\n\n| Type | Reference | Location |\n|------|-----------|----------|\n| Requirement | FR-003 | docs/requirements.md |\n| Entity model | TASK | docs/entity_model.md |\n| Glossary | Task, Status | docs/glossary.md |\n| Use case | UC-004 Filter Tasks | docs/use_cases/UC-004-filter-tasks.md |"
  }
}
```

## MCP Tool

This skill calls `mcp__jira__update_issue`. If the Jira MCP server uses a different tool name,
replace it in the SKILL.md Update section. Configure the server in `.mcp.json` under a `jira` key.

This payload updates only the `description` field (and `summary` when the story sentence changed).
It does not touch labels, priority, assignee, or any other field.
