# user-story Skill Family Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add three skills to `aiup-core` — `/user-story` (create), `/user-story-review` (validate), `/user-story-fix` (repair) — that together manage Jira stories as lightweight tracking tickets whose content lives entirely in the documentation tree.

**Architecture:** All three skills are stack-agnostic and live in `aiup-core/skills/`. `/user-story` orchestrates existing skills (`/requirements`, `/entity-model`, `/use-case-spec`, `/architecture-decision`) rather than duplicating their logic; it calls them by slash command and reads their output files. The Jira story holds only references to the docs, not their content; missing docs are marked as incomplete rows in the story. `/user-story-review` is read-only. `/user-story-fix` fetches an existing issue, re-runs the same checks as `/user-story`, and updates the issue in place.

**Tech Stack:** Bash (CI script patch), Markdown (three SKILL.md files and reference files).

**Spec:** This document and the brainstormed design confirmed in the conversation that produced it.

## Global Constraints

- Copyright header on every new `.md` file: `Copyright 2026 Andrei Kishkevich.` — not the upstream author line.
- Skill names must match folder names exactly: `user-story`, `user-story-review`, `user-story-fix`.
- Each skill `description` ≤ 1024 characters.
- No `../` links in any SKILL.md or references file; cross-skill files referenced by prose + glob, never Markdown links.
- Only `.md` files in each skill folder.
- User approval required before every Jira create or update call.
- The Jira story is a tracking ticket: all substantive content stays in `docs/`.

## Review Focus

- **Duplicate FR:** a story semantically matching an existing FR must reuse it, not create a new row.
- **All docs missing:** when none of requirements, entity model, or use cases exist yet, the skill must offer to run each skill in turn rather than halting on the first missing doc.
- **Partial incomplete story reviewed:** `/user-story-review` must report each incomplete row distinctly — "⚠️ Use case not yet written" is a different finding from "❌ Referenced FR-007 does not exist in requirements.md".
- **Fix with nothing to fix:** `/user-story-fix` on a story whose docs are all present and correct must report "No gaps found" and not update the issue.
- **Jira MCP absent:** all three skills must degrade gracefully — review prints its report to the terminal, create/fix print the payload as JSON.

---

### Task 1: Loosen the copyright-header CI check

**Files:**
- Modify: `scripts/check-copyright-headers.sh`

**Interfaces:**
- Produces: a check script that accepts any `Copyright <year(s)> <author>` line, not only the upstream one.

- [ ] **Step 1: Read the full check script**

  Read `scripts/check-copyright-headers.sh` to confirm the per-file check at the line containing `grep -qF "$COPYRIGHT_LINE" "$file"`.

- [ ] **Step 2: Replace the per-file check with a pattern match**

  Change:
  ```bash
  if grep -qF "$COPYRIGHT_LINE" "$file"; then
  ```
  to:
  ```bash
  if grep -qE 'Copyright [0-9]{4}' "$file"; then
  ```
  `COPYRIGHT_LINE`, `HEADER`, and `--fix` logic are unchanged — they still insert the upstream header into files that have no copyright at all.

- [ ] **Step 3: Verify existing files still pass**

  ```bash
  bash scripts/check-copyright-headers.sh
  ```
  Expected: `OK: copyright headers, LICENSE and NOTICE present`

- [ ] **Step 4: Smoke-test Andrei's header format**

  ```bash
  tmp=$(mktemp --suffix=.md)
  printf '<!--\nCopyright 2026 Andrei Kishkevich.\nLicensed under the Apache License, Version 2.0.\n-->\n\n# Test\n' > "$tmp"
  grep -qE 'Copyright [0-9]{4}' "$tmp" && echo "PASS" || echo "FAIL"
  rm "$tmp"
  ```
  Expected: `PASS`

- [ ] **Step 5: Commit**

  ```bash
  git add scripts/check-copyright-headers.sh
  git commit -m "fix: accept any copyright author in the header check"
  ```

---

### Task 2: Create the `/user-story` skill

**Files:**
- Create: `aiup-core/skills/user-story/SKILL.md`
- Create: `aiup-core/skills/user-story/references/jira-payload.md`
- Create: `aiup-core/skills/user-story/references/story-template.md`

**Interfaces:**
- Produces: a skill that creates a Jira story as a tracking ticket, orchestrating other skills for each doc type.

- [ ] **Step 1: Create SKILL.md — front matter and copyright**

  ```markdown
  ---
  name: user-story
  description: >
    Creates a Jira story as a lightweight tracking ticket that references the
    project's documentation (requirements, entity model, glossary, use case
    specifications, and architecture decisions) rather than duplicating it.
    Runs the corresponding skill for any missing document, or marks it
    incomplete in the story if the user defers. Use when the user asks to
    "create a user story", "add a story to Jira", "write a feature ticket",
    or describes a desired feature as "As a … I want … so that …".
  ---

  <!--
  Copyright 2026 Andrei Kishkevich.
  Licensed under the Apache License, Version 2.0. See LICENSE and NOTICE.
  -->
  ```

  Verify: `echo -n "<description text>" | wc -c` is under 1024.

- [ ] **Step 2: Write Instructions, Scope, and DO NOT**

  **Instructions** (one sentence): "Create a Jira story for $ARGUMENTS as a tracking ticket whose description references the project's documentation; run the appropriate skill for each missing document or mark the section incomplete if the user defers."

  **Scope — Consumes:**
  - $ARGUMENTS — story description (rough prose or user-story format)
  - `docs/requirements.md` — FR-* catalog
  - `docs/entity_model.md` and `docs/entity_model/*.md` — entity model
  - `docs/glossary.md` — domain terms
  - `docs/use_cases/` — use case specs (glob `docs/use_cases/**/UC-*.md`)
  - `docs/architecture/decisions/*.md` and `docs/architecture/adr/*.md` — ADRs

  **Scope — Produces:**
  - One Jira story (via MCP or printed as JSON)
  - Optionally: new documents created by delegated skills

  **Scope — Never touches:**
  - Existing documentation directly — delegation to `/requirements`, `/entity-model`, `/use-case-spec`, `/architecture-decision` only
  - More than one Jira issue per run

  **DO NOT:**
  - Duplicate logic from other skills; call them instead
  - Write to `docs/` directly — delegated skills do that
  - Push to Jira without user approval
  - Omit incomplete markers when a user defers a missing document

- [ ] **Step 3: Write the Story Format section**

  Format: `As a [role], I want [goal] so that [benefit].`

  Quality rules (defined by `/requirements`; this skill enforces them, not redefines them):
  - Singular: one goal per story; split compound stories before proceeding
  - Unambiguous: no subjective terms
  - Testable: a pass/fail criterion exists

- [ ] **Step 4: Write the Documentation Checks section**

  For each document type, in this order:

  | Document | Skill | Check |
  |---|---|---|
  | Functional requirement | `/requirements` | FR-* row matching the story's role + goal exists in `docs/requirements.md` |
  | Entity model | `/entity-model` | Every domain noun in the story has a `### ENTITY_NAME` section in the entity model |
  | Glossary | `/requirements` | Every domain noun appears as a Term row in `docs/glossary.md` |
  | Use case specification | `/use-case-spec` | A `UC-*.md` file covering the story's goal exists under `docs/use_cases/` |
  | Architecture decisions | `/architecture-decision` | At least one `Accepted` ADR links an NFR or constraint the story implies (optional — flag only, never block) |

  For each check:
  - **Present and consistent:** add a ✅ reference row in the story's Documentation table (file path, ID).
  - **Missing, user agrees to create:** hand off to the corresponding skill by slash command; return to this skill when the user is done and re-run the check.
  - **Missing, user defers:** add a ⚠️ row in the story's Incomplete section (see story template in [references/story-template.md](references/story-template.md)).

- [ ] **Step 5: Write the Jira Integration section**

  After completing all checks:
  1. Present the filled story template (complete and incomplete sections) for the user's review.
  2. Ask: *"Approve creating this Jira story? (yes / no / skip)"*
  3. **On yes:** call `mcp__jira__create_issue` with the payload from [references/jira-payload.md](references/jira-payload.md). On success: report the issue key and URL. On MCP absent: print the JSON payload.
  4. **On no or skip:** print the story content to the terminal; no Jira action.

- [ ] **Step 6: Write the Workflow section**

  Numbered steps:
  1. Shape $ARGUMENTS into the story format (validate quality rules; show to user before proceeding).
  2. Run each documentation check in table order. For each gap: offer to run the skill or mark as deferred. If the user runs a skill, re-check that document before moving to the next.
  3. Build the story description using [references/story-template.md](references/story-template.md): fill the Documentation table (✅ rows) and the Incomplete section (⚠️ rows).
  4. Present the full draft story and ask for Jira approval.
  5. Act on the response (see Jira Integration).

- [ ] **Step 7: Create references/story-template.md**

  Copyright header (Andrei's), then the template as a Markdown fenced block showing the exact structure an implementer fills in:

  ```
  *As a [role], I want [goal] so that [benefit].*

  ## Documentation

  | Type | Reference | Location |
  |------|-----------|----------|
  | Requirement | FR-XXX | docs/requirements.md |
  | Entity model | ENTITY_A, ENTITY_B | docs/entity_model.md |
  | Glossary | Term A, Term B | docs/glossary.md |
  | Use case | UC-XXX Title | docs/use_cases/UC-XXX-title.md |
  | Architecture | ADR-XXX Title | docs/architecture/decisions/ADR-XXX.md |

  ## Incomplete

  ⚠️ [Document type] not yet created — run `/<skill>` to add it.
  ```

  Note: omit the `## Incomplete` section entirely when all checks passed.

- [ ] **Step 8: Create references/jira-payload.md**

  Copyright header (Andrei's), then:
  - A Fields table: `project.key` (placeholder `YOUR_PROJECT_KEY`), `issuetype.name` (`Story`), `summary` (the goal clause, ≤ 255 chars), `description` (the filled story template), `labels` (`["FR-XXX"]`), `priority.name` (`Medium` or FR row value).
  - An example JSON block.
  - A note: this skill calls `mcp__jira__create_issue`; configure the Jira MCP server in `.mcp.json`.

- [ ] **Step 9: Verify CI**

  ```bash
  bash scripts/check-skill-frontmatter.sh
  bash scripts/check-skill-links.sh
  bash scripts/check-copyright-headers.sh
  ```
  Expected: all exit 0.

- [ ] **Step 10: Commit**

  ```bash
  git add aiup-core/skills/user-story/
  git commit -m "feat: add /user-story skill"
  ```

---

### Task 3: Create the `/user-story-review` skill

**Files:**
- Create: `aiup-core/skills/user-story-review/SKILL.md`

**Interfaces:**
- Consumes: Jira issue key from $ARGUMENTS; uses `mcp__jira__get_issue` (or prints instructions if absent).
- Produces: a findings report (terminal only — read-only skill).

- [ ] **Step 1: Create SKILL.md — front matter and copyright**

  ```markdown
  ---
  name: user-story-review
  description: >
    Validates an existing Jira user story created by /user-story: fetches the
    issue, checks that every referenced document exists and is internally
    consistent, and reports findings without modifying anything. Use when the
    user asks to "review a story", "validate story PROJ-42", "check if the
    story is complete", or mentions story correctness or stale references.
  ---

  <!--
  Copyright 2026 Andrei Kishkevich.
  Licensed under the Apache License, Version 2.0. See LICENSE and NOTICE.
  -->
  ```

- [ ] **Step 2: Write Instructions, Scope, and DO NOT**

  **Instructions:** "Fetch the Jira story for $ARGUMENTS and report whether every documentation reference is present, correct, and consistent — without writing or updating anything."

  **Scope — Consumes:** Jira issue (via MCP or pasted description), the documentation files it references.

  **Scope — Produces:** A findings report on the terminal. Never writes any file or updates the Jira issue.

  **DO NOT:**
  - Modify `docs/`, `docs/requirements.md`, or any other file
  - Update the Jira issue (that is `/user-story-fix`)
  - Report findings as errors when a document is intentionally deferred (⚠️ rows are advisory, not errors)

- [ ] **Step 3: Write the Findings section**

  Three finding levels:
  - **✅ OK** — document exists, the referenced ID is present, and it links back to the same story context.
  - **⚠️ Incomplete** — the story already marks this row as deferred; no action needed beyond noting it.
  - **❌ Error** — the story references a document or ID that does not exist, or exists but does not cover the story's intent.

  For each document type (requirement, entity model, glossary, use case, architecture), state what constitutes each level.

  Locate the story template used by `/user-story` with a glob for `**/*user-story/references/story-template.md` to understand the Documentation table format.

- [ ] **Step 4: Write the Jira Integration section**

  - Call `mcp__jira__get_issue` with the issue key from $ARGUMENTS to fetch the description.
  - If MCP is absent: ask the user to paste the story description and proceed with the pasted text.

- [ ] **Step 5: Write the Workflow section**

  1. Fetch or receive the story description.
  2. Parse the Documentation table: extract each row's Type, Reference, and Location.
  3. For each row: check the referenced file and ID; classify as ✅, ⚠️, or ❌.
  4. Parse the Incomplete section: list each ⚠️ row as advisory.
  5. Report all findings grouped by level, most severe first.
  6. If any ❌ findings: suggest running `/user-story-fix <issue-key>` to repair them.

- [ ] **Step 6: Verify CI**

  ```bash
  bash scripts/check-skill-frontmatter.sh
  bash scripts/check-skill-links.sh
  bash scripts/check-copyright-headers.sh
  ```
  Expected: all exit 0.

- [ ] **Step 7: Commit**

  ```bash
  git add aiup-core/skills/user-story-review/SKILL.md
  git commit -m "feat: add /user-story-review skill"
  ```

---

### Task 4: Create the `/user-story-fix` skill

**Files:**
- Create: `aiup-core/skills/user-story-fix/SKILL.md`
- Create: `aiup-core/skills/user-story-fix/references/jira-update-payload.md`

**Interfaces:**
- Consumes: Jira issue key from $ARGUMENTS; fetches the existing story via `mcp__jira__get_issue`.
- Produces: an updated Jira issue (via `mcp__jira__update_issue`) or a printed update payload.

- [ ] **Step 1: Create SKILL.md — front matter and copyright**

  ```markdown
  ---
  name: user-story-fix
  description: >
    Fetches an existing Jira user story, identifies gaps or errors in its
    documentation references, runs the appropriate skill for each missing
    document (or marks it deferred), and updates the issue in place with the
    corrected content. Use when the user asks to "fix story PROJ-42", "repair
    an incomplete story", "update the story with missing docs", or "complete
    an unfinished story".
  ---

  <!--
  Copyright 2026 Andrei Kishkevich.
  Licensed under the Apache License, Version 2.0. See LICENSE and NOTICE.
  -->
  ```

- [ ] **Step 2: Write Instructions, Scope, and DO NOT**

  **Instructions:** "Fetch the Jira story for $ARGUMENTS, re-run all documentation checks from `/user-story`, resolve gaps where possible, and update the issue in place with the corrected content after user approval."

  **Scope — Consumes:** existing Jira issue, `docs/` tree.

  **Scope — Produces:** updated Jira issue (same key); optionally new documents created by delegated skills.

  **DO NOT:**
  - Create a new Jira issue — always update the existing one
  - Update the issue without user approval
  - Write to `docs/` directly — delegate to the appropriate skill

- [ ] **Step 3: Write the Fetching the Story section**

  - Call `mcp__jira__get_issue` with the issue key from $ARGUMENTS.
  - If MCP absent: ask the user to paste the story description; proceed with that text; at the end print the update payload instead of calling the MCP.
  - Parse: extract the story sentence, Documentation table rows, and Incomplete rows.

- [ ] **Step 4: Write the Gap Resolution section**

  Identical logic to `/user-story`'s Documentation Checks step (locate `/user-story`'s SKILL.md with a glob for `**/*user-story/SKILL.md`). For each gap found:
  - **❌ Error row** (reference exists in story but doc is missing): offer to run the corresponding skill; if the user defers, convert to a ⚠️ Incomplete row.
  - **⚠️ Incomplete row** (already deferred): offer to run the skill now; if still deferred, leave as-is.
  - **✅ OK row**: no action.

  After all gaps are addressed, rebuild the full story description from the template (locate it with a glob for `**/*user-story/references/story-template.md`).

- [ ] **Step 5: Write the Update section**

  - If nothing changed: report "No gaps found — story is already correct" and stop.
  - Otherwise: show a diff of the old vs new description; ask *"Approve updating PROJ-42? (yes / no)"*.
  - On yes: call `mcp__jira__update_issue` with the corrected description field, using the payload from [references/jira-update-payload.md](references/jira-update-payload.md). Report success.
  - On no: print the updated description to the terminal.

- [ ] **Step 6: Write the Workflow section**

  1. Fetch the story (see Fetching the Story).
  2. Parse the existing Documentation table and Incomplete section.
  3. Re-run documentation checks for every row; identify ❌ errors and address ⚠️ deferrals (see Gap Resolution).
  4. Rebuild the story description.
  5. If nothing changed, stop and report.
  6. Show diff and ask for approval; act on the response (see Update section).

- [ ] **Step 7: Create references/jira-update-payload.md**

  Copyright header (Andrei's), then:
  - Fields table: `issueIdOrKey` (from $ARGUMENTS), `description` (the rebuilt story template content).
  - Example JSON: `{ "fields": { "description": "..." } }` (Jira REST v3 update format).
  - Note: this skill calls `mcp__jira__update_issue`.

- [ ] **Step 8: Verify CI**

  ```bash
  bash scripts/check-skill-frontmatter.sh
  bash scripts/check-skill-links.sh
  bash scripts/check-copyright-headers.sh
  ```
  Expected: all exit 0.

- [ ] **Step 9: Commit**

  ```bash
  git add aiup-core/skills/user-story-fix/
  git commit -m "feat: add /user-story-fix skill"
  ```

---

### Task 5: Register all three skills in CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Produces: an updated core skills table listing all three new skills.

- [ ] **Step 1: Locate the Core skills table**

  Find the `### Core (stack-agnostic)` section with the `Phase | Skill (slash command) | Description` table.

- [ ] **Step 2: Add the three rows**

  Insert in the Inception phase (before the Elaboration rows):

  | Inception | `/user-story` | Create a Jira story as a tracking ticket with references to requirements, entity model, use cases, and architecture |
  | Inception | `/user-story-review` | Validate that all documentation references in a Jira story are present and consistent (read-only) |
  | Inception | `/user-story-fix` | Fetch an incomplete or incorrect Jira story, resolve gaps by running the appropriate skills, and update the issue in place |

- [ ] **Step 3: Commit**

  ```bash
  git add CLAUDE.md
  git commit -m "docs: register /user-story, /user-story-review, /user-story-fix in core skills table"
  ```
