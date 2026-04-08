---
description: Run selected phases of the orchestrator workflow with flexible input
argument-hint: [clickup-list-url] [clickup-task-url] [prompt or comments]
---

# Task

Run any combination of orchestrator workflow phases. Accepts ClickUp URLs, a free-form prompt, or no arguments at all.

## Arguments

| Argument               | Required | Description                                                    |
| ---------------------- | -------- | -------------------------------------------------------------- |
| `[clickup-list-url]`   | No       | ClickUp list URL containing User Stories for the epic          |
| `[clickup-task-url]`   | No       | ClickUp task URL or ID (e.g., DEV-53174)                      |
| `[prompt or comments]` | No       | Free-form task description, or additional context if URLs given |

## Usage

```
/task

/task Add a new endpoint for patient sleep reports

/task https://app.clickup.com/t/3015277/DEV-53174

/task DEV-53174

/task https://app.clickup.com/3015277/v/l/li/901711067888 https://app.clickup.com/t/3015277/DEV-53174

/task DEV-53174 Only run code review and delivery on existing implementation
```

## What This Command Does

1. **Parses arguments** to determine input mode (ClickUp URLs, free-form prompt, or interactive)
2. **Presents phase picker** for user to select which workflow phases to run
3. **Executes selected phases** in order, skipping unselected ones

## Stack

- **Backend:** Express/Node.js (`express-explorer`, `story-planner`, `express-implementer`, `express-code-reviewer`)

Full backend development including routes, controllers, actions, models, migrations, and tests.

## Implementation Steps

When this command is invoked:

### 1. Parse Arguments

Analyze the provided arguments to determine the input mode:

**Mode A — No arguments provided:**

Ask the user: "What would you like to work on? Provide a description, a ClickUp task URL, or both."
Wait for the user's response, then re-parse using the rules below.

**Mode B — ClickUp URLs detected:**

If any argument contains `clickup.com` OR matches a task ID pattern like `DEV-NNNNN`:

- Scan for a URL containing `/v/l/li/` or `/v/li/` → extract as **list URL** (optional)
- Scan for a URL containing `/t/` or a standalone ID matching pattern like `DEV-53174` → extract as **task URL/ID**
- Everything remaining after extracting URLs and IDs → treat as **comments/context**

Extract IDs from URLs:

- List URL example: `https://app.clickup.com/3015277/v/l/li/901711067888` → List ID: `901711067888`
- Task URL example: `https://app.clickup.com/t/3015277/DEV-53174` → Task ID: `DEV-53174`

**Mode C — Free-form prompt:**

If no argument contains `clickup.com` and no argument matches a task ID pattern (`DEV-NNNNN`), treat the entire argument string as a **free-form task description**.

Store the parsed result:

- `input_mode`: `clickup` | `prompt` | `interactive`
- `list_url` / `list_id`: (if available)
- `task_url` / `task_id`: (if available)
- `prompt`: free-form description (if available)
- `comments`: additional context (if available)

### 2. Present Phase Picker

**Immediately after parsing arguments**, present the phase picker. Do NOT begin any work before the user selects phases.

Display this exact multiselect prompt:

```
Which phases should I run? Pick by number (comma-separated), or type "all":

  1. Exploration         — Investigate codebase (express-explorer sub-agent)
  2. Requirements        — Ask clarifying questions with codebase context (grill-me skill)
  3. Planning            — Draft high-level implementation plan (write-prd skill)
  4. Story Planning      — Create atomic checklist from plan (break-into-tasks skill)
  5. Implementation      — Code the backend feature (express-implementer)
  6. Code Review         — Review and fix violations (express-code-reviewer)
  7. Delivery            — Create PR, update ClickUp status

Example: "1,2,3,4,5" or "all" or "5,6,7"
```

Wait for the user's response. Parse the selected numbers into a set of active phases.

- If user types `all` → select phases 1 through 7.
- If user types comma-separated numbers (e.g., `1,2,4,5`) → select those phases.
- If user types a range or natural language (e.g., "just implementation and review") → interpret and confirm the selection before proceeding.

**Confirm the selection** back to the user before starting:

```
✅ Running phases: [list selected phase names]
⏭️ Skipping: [list skipped phase names]

Proceed? (y/n)
```

Wait for user confirmation before executing any phase.

### 3. Fetch ClickUp Context (if applicable)

**Condition:** Only run this step if `input_mode` is `clickup`.

If a **list URL** was provided:

- Extract the list ID from the URL
- Use ClickUp MCP tools to retrieve all tasks from the list
- Summarize as **Epic User Stories context** (read-only, for awareness)

If a **task URL/ID** was provided:

- Fetch the task details using ClickUp MCP tools
- Write the task description, acceptance criteria, and any attachments to `orchestrator/context/[feature]-clickup.md`
- Derive `feature` name from the task title (lowercase, hyphens). Example: "Add patient sleep reports endpoint" → `patient-sleep-reports`

If `input_mode` is `prompt`:

- Skip ClickUp fetch entirely
- Use the free-form prompt as the task description for all downstream phases
- Derive the feature name by slugifying the first few meaningful words of the prompt (lowercase, hyphens). Example: "Add patient sleep reports endpoint" → `patient-sleep-reports`

### 4. Execute Selected Phases

Iterate through phases 1-7 in order. For each phase, check if it is in the selected set. If not, skip it silently. If yes, execute it following the workflow below.

---

**Phase 1 — Exploration** (if selected)

1. Create empty context file: `orchestrator/context/[feature]-backend.md`
2. Delegate to **`express-explorer`** targeting the feature scope
3. After completion, read the context file

If `input_mode` is `prompt`, pass the free-form description as the feature scope to the explorer.

The backend context covers: relevant routes (and which routing system), controllers, services/actions, models, migrations, middlewares, validators, error classes, and existing tests.

---

**Phase 2 — Requirements** (if selected)


1. **Ask which pattern standard to follow:**

```
Which coding patterns should this feature follow?

  A. Current Patterns — Match existing codebase conventions
     (express-validator, SomnoRESTError, MiddleWares class, JS/TS mixed)

  B. Target Patterns — Use greenfield best practices from express-guidelines
     (Zod validation, AppError class hierarchy, standard Express Router, TypeScript-first)
```

2. Use grill-me skill to ask any clarifying questions needed to finalize requirements.
3. Store the answer as `pattern_standard`: `current` | `target` in the plan file.

---

**Phase 3 — Planning** (if selected)

1. Draft a high-level implementation plan covering:
   - Routes / controllers / actions to create or modify
   - Models and migrations needed
   - Middleware and validation changes
   - Error classes needed
   - Test coverage plan
2. Present the plan to the user for approval
3. Include `pattern_standard: current | target` in the plan header (from Phase 2)
4. Save the approved plan to `orchestrator/plans/[feature]-plan.md`

**Review Gate:** Do not proceed to Phase 4 until the user approves the plan.

**ClickUp integration (optional):** If a ClickUp task exists, propose subtask breakdown and create subtasks after user confirms.

---

**Phase 4 — Story Planning** (if selected)

1. Delegate to **`story-planner`** with:
   - Plan file path: `orchestrator/plans/[feature]-plan.md` (includes `pattern_standard`)
   - Backend context: `orchestrator/context/[feature]-backend.md`
   - Story number (default 0 for foundation, or as specified by user)
2. After completion, read the generated checklist
3. Present the checklist summary to the user for review

The story planner creates atomic, one-action-per-checkbox tasks with file paths. Each task maps to a single code change.

Output: `orchestrator/plans/[feature]-story-[N]-checklist.md`

---

**Phase 5 — Implementation** (if selected)

1. If a ClickUp task exists, note it as "In Progress"
2. Delegate to **`express-implementer`** with:
   - The feature description
   - Backend context from `orchestrator/context/[feature]-backend.md`
   - The approved plan from `orchestrator/plans/[feature]-plan.md` (includes `pattern_standard`)
   - The checklist from `orchestrator/plans/[feature]-story-[N]-checklist.md`
3. After implementation completes, verify checklist items against the changes made

---

**Phase 6 — Code Review** (if selected)

1. Delegate to **`express-code-reviewer`** with:
   - The checklist from Phase 4
   - The context file from Phase 1
2. If the reviewer reports decisions needed, present them to the user
3. Confirm all checklist items are marked complete before proceeding

---

**Phase 7 — Delivery** (if selected)

1. Verify all checklist items are marked `[x]`
2. Create a PR using `gh pr create` with:
   - A clear title derived from the feature name
   - Summary of changes and API endpoints affected
3. If a ClickUp task exists:
   - Update the task status to "PR Pending"
   - Add a comment with the PR link
4. Report completion to the user with the PR URL

## Error Handling

If any phase fails:

- Report which phase failed and the specific error
- Ask the user how to proceed:
  - **Retry** the failed phase
  - **Skip** it and continue to the next selected phase
  - **Abort** the workflow entirely

If a phase is selected but its prerequisites are clearly missing (e.g., Implementation selected but no checklist exists):

- Warn the user: "Phase 5 (Implementation) is selected but no checklist was found at `orchestrator/plans/`. Continue anyway?"
- Proceed only on user confirmation

## Important Notes

- **Phase picker is mandatory**: ALWAYS show the phase picker before starting any work, regardless of input mode.
- **No dependency enforcement**: If the user picks Phase 5 without Phase 3-4, trust them. They may have existing plans or checklists.
- **Express backend scope**: All phases use Express-specific agents (`express-explorer`, `story-planner`, `express-implementer`, `express-code-reviewer`).
- **Graceful degradation**: If a phase requires ClickUp context that does not exist, skip the ClickUp parts with a message rather than failing.
- **Feature name derivation**: From ClickUp task title (slugified) or from the first meaningful words of the free-form prompt (lowercase, hyphens). If a feature name already exists in `orchestrator/context/`, reuse it.
- **Comments carry forward**: If comments or prompt text was provided, reference it in Requirements (Phase 2) and Planning (Phase 3) to avoid asking redundant questions.
- **Do not skip exploration when selected**: Even with full task details, if Phase 1 (Exploration) is in the selected set, run it — respect the user's selection.

## Begin Now

Parse the provided arguments, determine input mode, then immediately present the phase picker. Do NOT begin any phase work until the user has selected and confirmed which phases to run.
