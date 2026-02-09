---
name: exec-epic
description: Execute beads epic subtasks sequentially using crush run, delegating to a local model by default.
---

# Epic Sequential Execution

Executes beads epic subtasks one at a time in the current working directory, delegating each to a model via `crush run`.

## Required Skills

| Skill | Tool | Purpose |
|-------|------|---------|
| `using-beads` | `mcp_skills_using_beads` | Read epics, manage task status, close tasks |
| `crush-run-delegate` | `mcp_skills_crush_run_delegate` | Delegate tasks to models via `crush run` — includes model tiers and complexity routing |

Call the tool to retrieve detailed reference documentation for each skill.

## Prerequisites

- Beads initialized (`bd init --stealth` for personal use)
- An epic with subtasks created in beads
- Git repository

## Workflow Overview

```
[Epic] → [Batch Read Subtasks] → [Loop: Delegate → On Fail: Escalate Model → Close] → [Sync] → [Summary]
```

## Instructions

### Phase 1: Batch Read Epic

Read the epic and all subtask metadata in one call:

```bash
bd show <epic-id>
bd children <epic-id>
```

Capture all task IDs, titles, and descriptions upfront. No per-task reads needed.

### Phase 2: Execute Tasks Sequentially

Default model is `lmstudio/qwen/qwen3-coder-next` (local, free). See `crush-run-delegate` skill for model tiers and when to escalate.

The calling model drives the loop — **not** a single bash script — so it can escalate on failure.

For each task, try each model tier in order until one succeeds (local → balanced → capable):

```bash
crush run -q -m <model> -c . "<task-prompt>" > /tmp/task-<task-id>.txt
echo $?
```

**On success** (exit 0): close the task with `bd close <task-id>` and move on.

**On failure** (non-zero exit): escalate to the next model tier and retry with the same prompt. If all tiers fail, log the failure and continue to the next task.

### Phase 3: Sync and Summary

Sync once after all tasks, then summarize:

```bash
bd sync
```

Output a summary noting which tasks succeeded and at which model tier, plus any that failed all tiers.

## Prompt Constraints

Every task prompt MUST include scope-locking directives:

```
Task: <task-description>

Allowed files (ONLY modify these):
- path/to/file1.go

Constraints:
- DO NOT modify any files other than those listed above.
- DO NOT delete or remove any existing functions, methods, or test cases.

Acceptance criteria:
- <observable criteria>
```

## Tips

1. **Agent-driven loop** — the calling model controls the loop so it can escalate on failure
2. **Each task sees prior results** — later tasks build on changes from earlier ones
3. **Default to local models** — free and fast; escalate only when the cheaper tier fails
4. **Always scope-lock prompts** — allowed file lists and DO NOT directives prevent scope creep
5. **Escalate, don't skip** — retry with increasingly capable models before giving up
6. **Run `crush run` synchronously** — do NOT use `run_in_background=true`; for parallel execution use shell `&` + `wait` (see `crush-run-delegate` skill)
