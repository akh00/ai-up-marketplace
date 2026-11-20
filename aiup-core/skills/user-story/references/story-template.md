<!--
Copyright 2026 Andrei Kishkevich.
Licensed under the Apache License, Version 2.0. See LICENSE and NOTICE.
-->

# Story Template

The Jira story description to fill for each new user story. The story is a tracking ticket: all
substantive content stays in the documentation tree; only references go here.

```markdown
*As a [role], I want [goal] so that [benefit].*

## Acceptance Criteria

- [ ] All automated tests pass (unit, integration, and E2E where applicable)
- [ ] Documentation updated: requirements, entity model, glossary, and use case specification are complete and consistent with the implementation
- [ ] No open blocking questions in `docs/open_questions.md` for this story's FR-* or use case
- [ ] Use case coverage audit passes — no implementation or test gaps reported by `/use-case-coverage`
- [ ] Code review approved by at least one reviewer
- [ ] The use case reaches **Tested** status (via `/use-case-test`) before the story is closed

## Documentation

| Type | Reference | Location |
|------|-----------|----------|
| Requirement | FR-XXX | docs/requirements.md |
| Entity model | ENTITY_A, ENTITY_B | docs/entity_model.md |
| Glossary | Term A, Term B | docs/glossary.md |
| Use case | UC-XXX Title | docs/use_cases/UC-XXX-title.md |
| Architecture | ADR-XXX Title | docs/architecture/decisions/ADR-XXX.md |

## Incomplete

⚠️ [Document type] not yet created — run `/[skill]` to add it.
```

**Rules:**
- Replace every placeholder with the real value; never leave `XXX` or `[role]` in the final story.
- The `## Acceptance Criteria` section is standard for every story — copy it verbatim; do not remove or reword items.
- Omit a row from the Documentation table when the document is confirmed missing and the user deferred it — that row goes to Incomplete instead.
- Omit the `## Incomplete` section entirely when all checks passed.
- The `## Documentation` table always appears, even when all rows are present.
- Entity model and Glossary rows list the specific entities and terms the story names, comma-separated.
- Architecture row is advisory: include it when at least one Accepted ADR links an NFR or constraint the story implies; omit it otherwise (do not add an Incomplete row for a missing ADR).
