---
name: user-story
description: >
  Creates a Jira story as a lightweight tracking ticket that references the
  project's documentation (requirements, entity model, glossary, use case
  specifications, and architecture decisions) rather than duplicating it.
  Runs the corresponding skill for any missing document, or marks the section
  incomplete in the story if the user defers. Use when the user asks to
  "create a user story", "add a story to Jira", "write a feature ticket",
  "add FR to Jira", or describes a desired feature as "As a … I want … so
  that …".
---

<!--
Copyright 2026 Andrei Kishkevich.
Licensed under the Apache License, Version 2.0. See LICENSE and NOTICE.
-->

# User Story

## Instructions

Create a Jira story for $ARGUMENTS as a tracking ticket whose description references the project's
documentation; run the appropriate skill for each missing document or mark the section incomplete
if the user defers.

## Scope

**Consumes:**
- $ARGUMENTS — story description in rough prose or "As a [role], I want [goal] so that [benefit]" form
- `docs/requirements.md` — FR-* catalog
- `docs/entity_model.md` and `docs/entity_model/*.md` — entity model (modular projects)
- `docs/glossary.md` — domain terms
- `docs/use_cases/**/UC-*.md` — use case specifications
- `docs/architecture/decisions/*.md`, `docs/architecture/adr/*.md`, `docs/architecture/adrs/*.md` — ADRs

**Produces:**
- One Jira story (via MCP or printed as JSON payload)
- Optionally: new documents created by delegated skills

**Never touches:**
- `docs/` directly — all documentation changes are delegated to the appropriate skill
- More than one Jira issue per run

## DO NOT

- Duplicate logic from `/requirements`, `/entity-model`, `/use-case-spec`, or `/architecture-decision`; call them by slash command instead
- Write to any `docs/` file directly
- Push to Jira without explicit user approval
- Omit incomplete markers when the user defers a missing document
- Create a second FR-* row when a semantically equivalent one already exists

## Story Format

Every story must follow the user story format defined by `/requirements`:

**"As a [role], I want [goal] so that [benefit]."**

Quality rules (enforced by `/requirements`; this skill validates, not redefines, them):
- **Singular:** one goal per story; split compound stories before proceeding
- **Unambiguous:** no subjective terms (user-friendly, fast, easy)
- **Testable:** a pass/fail criterion exists for the goal

Shape rough input into this format before proceeding. Show the shaped story to the user and confirm
before moving to the documentation checks.

## Documentation Checks

Check each document type in this order. For each gap, offer two options: run the corresponding skill
now, or mark the section as incomplete in the story (a ⚠️ row in the Incomplete section of
[references/story-template.md](references/story-template.md)).

If the user chooses to run a skill, hand off by slash command and return to finish this skill when
the user is done. Re-run the check for that document before moving to the next.

**After each check that passes, record the exact ID and file path** — these replace the `FR-XXX`,
`UC-XXX`, `ADR-XXX`, entity names, and term names in the story template. Never leave a placeholder
in the final story.

| # | Document | How to check | What to do when missing |
|---|----------|-------------|------------------------|
| 1 | **Functional requirement** | Scan FR rows in `docs/requirements.md` for a row whose role and goal match the story. Record the FR-* ID. | Write the FR row directly (see below); hand off to `/requirements` only when the catalog file does not exist yet. |
| 2 | **Entity model** | Every domain noun extracted from the story has a `### ENTITY_NAME` section in the entity model. Record the matching entity names. | Offer to run `/entity-model`. |
| 3 | **Glossary** | Every domain noun appears as a Term row in `docs/glossary.md`. Record the matching terms. | Offer to run `/requirements` (glossary is part of requirements). |
| 4 | **Use case specification** | A `UC-*.md` file covering the story's goal exists under `docs/use_cases/`. Record the UC-* ID and title. | Offer to run `/use-case-spec`. |
| 5 | **Architecture decisions** | At least one `Accepted` ADR links an NFR or constraint the story implies. Record the ADR-* ID. (Advisory — flag only, never block.) | Offer to run `/architecture-decision`. |

**Writing a missing FR row directly:** When `docs/requirements.md` exists but no matching FR row is
found, add one row to the FR table using the user story format defined by `/requirements`:
`| FR-NNN | <title> | As a [role], I want [goal] so that [benefit]. | Medium | Open |`
where `FR-NNN` is the next available ID (highest existing FR number + 1; `FR-001` if none exist).
Record this FR-* ID for use in the story template.
When the catalog file does not exist at all, hand off to `/requirements` — note that it reads
`docs/vision.md` and will generate the full catalog, not just one row; return to this skill when done.

**Domain nouns** to extract: roles, business objects, states, and events from the story sentence.
Exclude generic words: system, user, data, interface, item.

**Missing entity model:** if neither `docs/entity_model.md` nor any `docs/entity_model/*.md` file
exists, note "No entity model found" as a ⚠️ row and offer to run `/entity-model`.

## Story Template

Build the Jira story description from [references/story-template.md](references/story-template.md).

Fill the **Documentation** table with ✅ rows for each check that passed (file path and ID).
Fill the **Incomplete** section with ⚠️ rows for each check the user deferred.
Omit the **Incomplete** section entirely when all checks passed.

## Jira Integration

After all documentation checks are complete:

1. Present the full filled story draft for the user's review.
2. Ask: *"Approve creating this Jira story? (yes / no / skip)"*
3. **On yes:** call `mcp__jira__create_issue` with the payload from
   [references/jira-payload.md](references/jira-payload.md).
   - On success: report the issue key and URL.
   - On MCP absent or failure: print the JSON payload from the reference file; the user can create the issue manually.
4. **On no or skip:** print the story content to the terminal; no Jira action is taken.

**MCP not configured:** add a `jira` server entry to `.mcp.json` before running this skill again.

## Workflow

1. Shape $ARGUMENTS into the story format (see Story Format). Show to the user; confirm before proceeding.
2. Run each documentation check in table order (see Documentation Checks). For each gap: offer to run the skill or mark as deferred. If the user runs a skill, re-check that document before moving to the next.
3. Build the story description from the template (see Story Template).
4. Present the full draft and ask for Jira approval (see Jira Integration).
5. Act on the response (yes / no / skip) as defined in Jira Integration.
