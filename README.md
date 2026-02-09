# Agents Workflow

Patterns and tools for incorporating **local models** into software development workflows.

The key idea is simple: **constraints that make local models succeed also make SOTA (state-of-the-art) models faster, cheaper, and more reliable.**

## Who this is for

- You want to use local models for coding tasks.
- You want a repeatable workflow for delegating coding tasks to models (local or cloud).
- You want *machine-checkable* progress (tests/build/run) instead of “looks good”.
- You want safer tool use (sandboxed filesystem, explicit boundaries) to reduce agent mistakes.

## What you’ll get

- A three-step workflow (`create-plan → plan-to-beads → epic-exec`) that turns “do X” into sequential, verifiable tasks.
- A tool strategy that replaces confusing, multi-purpose tools with more robust and secure MCP alternatives.
- A repo of reusable skills you can use with agents that support skills natively or expose via the MCP server. Most skills are runner-agnostic; `crush-run-delegate` is Crush-specific.

## Table of contents

- [TL;DR](#tldr)
- [Quickstart (conceptual)](#quickstart-conceptual)
- [Core workflow](#core-workflow)
- [Why this works](#why-this-works)
- [Tool strategy (MCP-first)](#tool-strategy-mcp-first)
- [Example: end-to-end task](#example-end-to-end-task)
- [Skills](#skills)
- [AGENTS.md](#agentsmd)
- [Local models](#local-models)
- [Qwen template](#qwen-template)
- [Directory structure](#directory-structure)
- [Current state](#current-state)
- [Next steps](#next-steps)

---

## TL;DR

```
create-plan → plan-to-beads → epic-exec
```

1. **Create Plan**: produce a structured markdown plan with explicit file paths and acceptance criteria.
2. **Plan to Beads**: convert that plan into a [beads](https://github.com/charmcli/beads) epic + subtasks.
3. **Epic Exec**: run tasks sequentially via your agent runner (for example, `crush run`), escalating models only when needed.

---

## Quickstart (conceptual)

This repo is intentionally focused on *workflow patterns*, not a single “app” to run.

A typical loop looks like:

1. **Write (or generate) a plan** that breaks work into small, scope-locked tasks.
2. **Convert the plan to a beads epic** so each task is trackable and independently executable.
3. **Execute tasks sequentially** with a default local model, escalating only on failure.

If you’re using an MCP-capable coding agent runner (Crush, OpenCode, Claude Code, etc.), the most relevant references are:

- Planning: [skills/create-plan/SKILL.md](skills/create-plan/SKILL.md)
- Beads conversion: [skills/plan-to-beads/SKILL.md](skills/plan-to-beads/SKILL.md)
- Execution + escalation: [skills/epic-exec/SKILL.md](skills/epic-exec/SKILL.md) and [skills/crush-run-delegate/SKILL.md](skills/crush-run-delegate/SKILL.md)

---

## Core workflow

### 1) Create Plan

Plans are designed to work within local-model constraints:

- **Explicit file paths** — no searching required
- **Self-contained tasks** — each task includes all context it needs
- **Observable acceptance criteria** — models can verify their own work
- **Dependency ordering** — foundational changes first

### 2) Plan to Beads

Convert the plan into a trackable epic with subtasks using [beads](https://github.com/charmcli/beads). Each task becomes an independently executable unit of work.

### 3) Epic Exec

Execute subtasks sequentially, delegating each to a model via your agent runner (for example, `crush run`). Start with the cheapest model and escalate on failure:

| Tier | Model | When |
|------|-------|------|
| **local** | `lmstudio/qwen/qwen3-coder-next` | Default — try this first |
| **balanced** | `openai/gpt-5.2-codex` | Local model failed |
| **capable** | `anthropic/claude-opus-4-6` | Balanced model failed |

Most tasks should complete at the local tier. When escalation happens, the calling model should state it explicitly and report which tier succeeded.

---

## Why this works

Local models are fast, private, and cheap—but more brittle. This workflow succeeds by *designing tasks that are hard to misunderstand*.

### Constraints that help local models (and improve SOTA workflows too)

- **Scope locking**: every task lists the only files that may be modified.
- **Small tasks**: one concern per task (one function, one endpoint, one wiring step).
- **Explicit inputs/outputs**: acceptance criteria are concrete and checkable.
- **Sequential execution**: later tasks rely on verified earlier work.
- **Escalation instead of thrashing**: retry with a more capable model when needed.

### What “good acceptance criteria” looks like

Prefer criteria that a model can verify without judgment calls:

- ✅ “`go test ./...` passes”
- ✅ “Endpoint returns `200` and JSON includes field `X`”
- ✅ “Config loads without error”
- ❌ “Code is clean”

---

## Tool strategy (MCP-first)

Local models struggle with complex, multi-purpose tools. They hallucinate parameters, confuse similar tools, and make mistakes that larger models avoid.

The approach here is:

1. **Disable confusing built-in tools**.
2. **Replace them with simpler MCP tools** that have hard boundaries (sandboxing / allowlists) and single-purpose interfaces.

### Disabled built-in tools

The included [Crush](https://github.com/charmcli/crush) configuration disables several built-in tools:

| Tool | Why disabled |
|------|--------------|
| `view` | Replaced by MCP filesystem with explicit allowed directories |
| `edit` | Local models struggle with find/replace precision |
| `multiedit` | Too complex — multiple edits compound errors |
| `write` | Replaced by sandboxed MCP filesystem |
| `file_path` | Ambiguous interface, replaced by explicit MCP tools |

See `crush/crush.json`.

### MCP replacements

Three MCP servers provide safer, more reliable alternatives.

#### [Filesystem MCP](https://github.com/portertech/filesystem-mcp-server)

Sandboxed file operations with explicit directory allowlists.

<details>
<summary>Example config</summary>

```json
{
  "filesystem": {
    "command": "docker",
    "args": ["run", "-i", "--rm",
      "--mount", "type=bind,src=/projects,dst=/projects",
      "portertech/filesystem-mcp-server:latest",
      "/projects"
    ]
  }
}
```

</details>

Benefits:

- Docker container isolation — can only access mounted paths
- Explicit allowed directories passed as arguments
- Separate tools for read vs write operations
- Simpler interfaces that local models handle reliably

#### [Skills MCP](https://github.com/portertech/skills-mcp-server)

Serves skill documents as retrievable tools.

<details>
<summary>Example config</summary>

```json
{
  "skills": {
    "command": "docker",
    "args": ["run", "-i", "--rm",
      "--mount", "type=bind,src=/skills,dst=/skills,readonly",
      "portertech/skills-mcp-server:latest",
      "/skills"
    ]
  }
}
```

</details>

Benefits:

- Read-only mount — skills cannot be modified
- Each skill becomes a callable tool
- Agents retrieve instructions on-demand
- Tool calls are visible in the UI — clearer UX, easier debugging, and better trust/auditability

#### [LM Studio MCP](https://github.com/portertech/lm-studio-mcp-server)

Model management without shell access.

<details>
<summary>Example config</summary>

```json
{
  "lmstudio": {
    "command": "docker",
    "args": ["run", "-i", "--rm",
      "-e", "LMSTUDIO_HOST=lmstudio",
      "portertech/lm-studio-mcp-server:latest"
    ]
  }
}
```

</details>

Benefits:

- Query available models without bash
- Load/unload models programmatically
- Health checks for orchestration

---

## Example: end-to-end task

This example mirrors the workflow: plan → beads → execution.

### Step 1: create a scope-locked task

A typical task delegated to a local model:

```
Task: Add Validate method to UserInput struct in pkg/models/user.go

Allowed files (ONLY modify these):
- pkg/models/user.go

Constraints:
- DO NOT modify any files other than those listed above.
- DO NOT delete or remove any existing functions, methods, or test cases.

Acceptance criteria:
- Validate() returns error if Email is empty or invalid format
- Validate() returns error if Name is empty
- Validate() returns nil for valid input
```

### Step 2: convert plan to beads tasks

The plan becomes a beads epic; each `###` task heading becomes a subtask title. This makes work auditable and resumable.

### Step 3: execute tasks sequentially with escalation

Execution runs each task via your agent runner (for example, `crush run`) at the local tier first. On failure, it retries the *same scope-locked prompt* with a more capable tier.

Expected behavior:

- Most tasks succeed locally.
- Failures escalate (local → balanced → capable).
- The orchestrator reports which tier succeeded.

---

## Skills

The `skills/` directory contains reference documents that agents can retrieve during execution. Each skill provides domain knowledge or workflow instructions.

### Workflow skills

| Skill | Purpose |
|------|---------|
| [create-plan](skills/create-plan/SKILL.md) | Structured plans optimized for local model execution |
| [plan-to-beads](skills/plan-to-beads/SKILL.md) | Convert plans to trackable epics with subtasks |
| [epic-exec](skills/epic-exec/SKILL.md) | Execute subtasks with automatic model escalation |
| [using-beads](skills/using-beads/SKILL.md) | Task lifecycle: start, complete, sync, verify |
| [crush-run-delegate](skills/crush-run-delegate/SKILL.md) | Delegate to models via `crush run` |

### Domain skills

| Skill | Purpose |
|------|---------|
| [effective-go](skills/effective-go/SKILL.md) | Go best practices for idiomatic code |
| [github-pr-review](skills/github-pr-review/SKILL.md) | PR review with GitHub CLI |
| [git-worktree](skills/git-worktree/SKILL.md) | Parallel workspaces for isolated agent tasks |
| [otel-mdatagen](skills/otel-mdatagen/SKILL.md) | OpenTelemetry metadata generator reference |

---

## AGENTS.md

Many agent runners support a global “agent instructions” file that is loaded as context at runtime. In Crush, this can be `~/.config/crush/AGENTS.md`. This repo also includes an example at `crush/AGENTS.md`.

The goal is to remove ambiguity by making “when to retrieve skills” explicit:

```markdown
YOU MUST ALWAYS:

* Use the mcp filesystem tools for file operations
* Use the mcp_skills_effective_go tool before reviewing, writing, or refactoring Go code
* Use the mcp_skills_create_plan tool before creating or making a plan
* Use the mcp_skills_plan_to_beads tool before creating a beads epic from a plan
...
```

---

## Local models

The included Crush configuration can include several local models that work well with these workflows.

In this repo, see `crush/crush.json` for example LM Studio provider configuration.

| Model | Context | Notes |
|------|---------|------|
| [Qwen3 Coder Next (80B-A3B)](https://huggingface.co/Qwen/Qwen3-Coder-Next) | 128K | Latest Qwen coder, Q8_0 quantization |
| [Qwen3 Coder Flash (30B-A3B)](https://huggingface.co/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF) | 128K | Smaller and faster, Unsloth BF16 |
| [GLM 4.7 Flash (30B-A3B)](https://huggingface.co/unsloth/GLM-4.7-Flash-GGUF) | 128K | Reasoning support, Unsloth BF16 |
| [GLM 4.5 Air (106B-A12B)](https://huggingface.co/unsloth/GLM-4.5-Air-GGUF) | 128K | Reasoning support, Unsloth Q4_K_XL |
| [Devstral 2 (24B)](https://huggingface.co/unsloth/Devstral-Small-2-24B-Instruct-2512-GGUF) | 200K | Dense, slower but good for long tasks, Unsloth BF16 |

All can be served via [LM Studio](https://lmstudio.ai/) with an OpenAI-compatible API.

---

## Qwen template

The `qwen/template.jinja` file provides a simple, reliable chat template for Qwen3 Coder models in LM Studio.

Default Qwen templates can be inconsistent with tool calling. This template uses a straightforward XML format that Qwen local models handle reliably:

```xml
<tool_call>
{"name": "function_name", "arguments": {"arg": "value"}}
</tool_call>
```

Tool responses follow the same pattern:

```xml
<tool_response>
... result ...
</tool_response>
```

To use in LM Studio, set the model's chat template to the contents of `qwen/template.jinja`.

---

## Directory structure

```
.
├── README.md
├── crush/
│   ├── AGENTS.md
│   └── crush.json
├── qwen/
│   └── template.jinja
└── skills/
    ├── create-plan/SKILL.md
    ├── plan-to-beads/SKILL.md
    ├── epic-exec/SKILL.md
    ├── using-beads/SKILL.md
    ├── crush-run-delegate/SKILL.md
    ├── effective-go/SKILL.md
    ├── github-pr-review/SKILL.md
    ├── git-worktree/SKILL.md
    └── otel-mdatagen/SKILL.md
```

---

## Current state

Evaluation of this philosophy is ongoing. Experimenting with local models has been fun and has resulted in general workflow improvements that benefit SOTA models too.

That said, the hardware required to run local models capable enough for day-to-day professional development can be expensive. Until 30–80B parameter models make another leap in capability, it may not be practical for most developers to invest heavily in local hardware for professional use.

The workflows here remain valuable: they work even better with cloud models, and the constraints improve reliability regardless of model tier.

---

## Next steps

- If you want to adopt the workflow: start with [create-plan](skills/create-plan/SKILL.md) → [plan-to-beads](skills/plan-to-beads/SKILL.md) → [epic-exec](skills/epic-exec/SKILL.md).
- If you want to customize tool boundaries: review `crush/crush.json` (disabled tools + allowed MCP tools).
- If you want to improve local tool calling reliability: use `qwen/template.jinja` as the LM Studio chat template for Qwen models.
