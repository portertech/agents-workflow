---
name: using-beads
description: Commands for using beads epics and working on assigned tasks.
---

# Using Beads

Commands for using beads epics and working on assigned tasks.

## Critical Rules

**You MUST follow these rules for every task:**

1. **MUST** run `bd update <id> -s in_progress` before starting work
2. **MUST** run `bd close <id>` immediately after completing work
3. **MUST** run `bd sync` after closing to persist changes
4. **MUST** verify closure with `bd show <id>` (status should be `closed`)

Failure to close tasks leaves work untracked and blocks epic completion.

## Quick Reference

| Action | Command |
|--------|---------|
| Show epic | `bd show <epic-id>` |
| Show subtasks | `bd children <epic-id>` |
| Show task | `bd show <id>` |
| **Start work** | `bd update <id> -s in_progress` |
| **Complete** | `bd close <id>` |
| **Persist** | `bd sync` |
| **Verify** | `bd show <id>` |
| List open | `bd list -s open` |

## Investigating an Epic

When assigned an epic, understand it before starting:

```bash
# Read the epic details and description
bd show EPIC-1

# List all subtasks under the epic
bd children EPIC-1

# Show a specific subtask
bd show TASK-42
```

## Working on a Task

When given a task ID (e.g., `TASK-42`):

```bash
# 1. Read the task details
bd show TASK-42

# 2. Mark as in progress before starting
bd update TASK-42 -s in_progress

# 3. Do the work...

# 4. MUST: Close when complete
bd close TASK-42

# 5. MUST: Persist changes
bd sync

# 6. MUST: Verify closure succeeded
bd show TASK-42
# Confirm status shows "closed"
```

## Notes

- Always run `bd show <id>` first to understand the task
- For epics, use `bd children` to see all subtasks
- Update status to `in_progress` before starting work
- **Never skip the close/sync/verify steps** — they are mandatory
