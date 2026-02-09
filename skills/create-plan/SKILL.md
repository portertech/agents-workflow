---
name: create-plan
description: Create a markdown plan optimized for beads epic conversion and agent execution, output directly to chat.
---

# Create Plan

Create structured markdown plans optimized for beads epic conversion and autonomous agent execution. Plans are always output directly to the chat conversation, never written to a file.

Every plan becomes a beads epic: the plan title becomes the epic title, the full plan text becomes the epic description, and each task heading becomes a beads subtask title. Design accordingly.

## Instructions

When the user asks to create a plan:

1. **Gather context**: Understand the goal by reading relevant code, docs, and conversation history. Do not ask clarifying questions unless truly ambiguous — make reasonable assumptions and state them in the plan.

2. **Analyze scope**: Determine what needs to change, which files are involved, and identify dependencies between tasks.

3. **Write the plan**: Output a markdown plan to chat following the format below.

## Plan Format

```markdown
# <concise goal — becomes the beads epic title>

## Context

<1-3 sentences: what exists today, why the change is needed, key assumptions>

## Tasks

### 1. <self-contained action — becomes a beads subtask title>
- **Files**: `path/to/file.go`, `path/to/other.go`
- **Details**: <what to do, not how — agents decide implementation>
- **Acceptance**: <observable criteria: tests pass, endpoint returns X, etc.>

### 2. <self-contained action — becomes a beads subtask title>
- **Files**: `path/to/file.go`
- **Details**: <what to do>
- **Acceptance**: <observable criteria>
- **Depends on**: Task 1

### 3. <self-contained action — becomes a beads subtask title>
- **Files**: `path/to/file.go`
- **Details**: <what to do>
- **Acceptance**: <observable criteria>

### 4. <self-contained action — becomes a beads subtask title>
- **Files**: `path/to/wire.go`
- **Details**: <what to do>
- **Acceptance**: <observable criteria>
- **Depends on**: Tasks 2, 3

## Verification

<final checks to confirm the entire plan is complete: test commands, build commands, manual spot-checks>
```

## Plan Rules

### Beads Compatibility

- **Plan title becomes the epic title** — make it concise and descriptive
- **Full plan text becomes the epic description** — the entire markdown is preserved verbatim
- **Each task heading becomes a beads subtask title** — titles must be fully self-descriptive without reading the body
- **Keep tasks flat** — no sub-subtasks; beads subtasks are a flat list under the epic
- **Number tasks sequentially** across all sections (including parallel groups) for unambiguous reference

### Task Titles

- **Titles must stand alone**: an agent reading only the title should know what to do (e.g., "Add Validate method to UserInput struct in pkg/models/user.go" not "Add validation")
- **Start with an action verb**: Add, Create, Refactor, Update, Remove, Extract, Implement, Fix, Wire, Configure
- **Include the key identifier**: function name, struct name, endpoint, or file
- **One logical change per task** — if a task touches unrelated concerns, split it

### Task Body

- **Include file paths** so agents can navigate directly without searching
- **Acceptance criteria must be machine-verifiable** when possible (test passes, compiles, endpoint responds)
- **Each task must be independently executable** by an agent with no human input
- **Avoid implementation prescriptions** unless a specific approach is required — describe *what*, let agents decide *how*

### Ordering and Dependencies

- **Tasks are sequential by default** — list them in execution order
- **Mark explicit dependencies** only when a task depends on a non-adjacent prior task
- **Put foundational changes first**: types/interfaces, then implementation, then wiring, then tests

### Scope

- **Include all necessary work** — don't leave implicit steps for the agent to discover
- **Include test updates** as part of the task that changes behavior, or as a dedicated task
- **Include config/migration/docs changes** if the feature requires them
- **Omit trivial steps** like "save file" or "run formatter" — agents handle these automatically

## Example Invocations

The user might say:
- "Create a plan for adding user authentication"
- "Plan out the refactor of the database layer"
- "Make a plan for this feature"
- "Help me plan the migration to the new API"

## Notes

- Always output the plan directly in chat, never write to a file
- Keep plans concise — agents work better with clear, brief instructions than verbose descriptions
- If the codebase is unfamiliar, read key files first to produce accurate file paths and identifiers
