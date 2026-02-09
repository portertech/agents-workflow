# Agents Workflow

Patterns and tools for incorporating local models into software development workflows.

## Philosophy

**If it works with local models, it works even better with SOTA.**

Local models like Qwen3 Coder are fast, free, and private—but they have limits. This project explores workflow designs that succeed within those constraints:

- **Structured plans** with explicit file paths and acceptance criteria reduce ambiguity
- **Scope-locked prompts** prevent models from wandering outside allowed files
- **Small, self-contained tasks** fit within context windows and attention spans
- **Sequential execution** lets each task build on verified prior work
- **Automatic escalation** catches failures early and retries with more capable models

The result: workflows that succeed with a local 32B model running on your laptop. When you need more capability, the same workflows scale up seamlessly to cloud models.

---

## Tool Strategy

Local models struggle with complex, multi-purpose tools. They hallucinate parameters, confuse similar tools, and make mistakes that larger models avoid. The solution: replace problematic built-in tools with simpler, sandboxed MCP alternatives.

### Disabled Built-in Tools

The [Crush](https://github.com/charmcli/crush) configuration disables several built-in tools:

| Tool | Why Disabled |
|------|--------------|
| `view` | Replaced by MCP filesystem with explicit allowed directories |
| `edit` | Local models struggle with find/replace precision |
| `multiedit` | Too complex — multiple edits compound errors |
| `write` | Replaced by sandboxed MCP filesystem |
| `file_path` | Ambiguous interface, replaced by explicit MCP tools |

### MCP Replacements

Three MCP servers provide safer, more reliable alternatives:

**[Filesystem MCP](https://github.com/portertech/filesystem-mcp-server)** — Sandboxed file operations with explicit directory allowlists:
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

Benefits:
- Docker container isolation — can only access mounted paths
- Explicit allowed directories passed as arguments
- Separate tools for read vs write operations
- Simpler interfaces that local models handle reliably

**[Skills MCP](https://github.com/portertech/skills-mcp-server)** — Serves skill documents as retrievable tools:
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

Benefits:
- Read-only mount — skills cannot be modified
- Each skill becomes a callable tool
- Agents retrieve instructions on-demand
- Tool calls are visible in the UI — you see when a skill is invoked

**[LM Studio MCP](https://github.com/portertech/lm-studio-mcp-server)** — Model management without shell access:
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

Benefits:
- Query available models without bash
- Load/unload models programmatically
- Health checks for orchestration

### Why This Helps Local Models

1. **Fewer tools** — Smaller tool set means less confusion
2. **Simpler interfaces** — One purpose per tool, obvious parameters
3. **Hard boundaries** — Can't access files outside allowed directories
4. **Fail-safe** — Errors are explicit, not silent corruption
5. **Containerized** — Network and filesystem isolation by default

---

## Core Workflow

```
create-plan → plan-to-beads → epic-exec
```

### 1. Create Plan

Generate a structured markdown plan optimized for agent execution. Plans are designed to work within local model constraints:

- **Explicit file paths** — no searching required
- **Self-contained tasks** — each task has everything needed to execute
- **Observable acceptance criteria** — models can verify their own work
- **Dependency ordering** — foundational changes first

### 2. Plan to Beads

Convert the plan into a trackable epic with subtasks using [beads](https://github.com/charmcli/beads). Each task becomes an independently executable unit of work.

### 3. Epic Exec

Execute subtasks sequentially, delegating each to a model via `crush run`. Start with the cheapest model and escalate on failure:

| Tier | Model | When |
|------|-------|------|
| **local** | `lmstudio/qwen/qwen3-coder-next` | Default — try this first |
| **balanced** | `openai/gpt-5.2-codex` | Local model failed |
| **capable** | `anthropic/claude-opus-4-6` | Balanced model failed |

Most tasks complete at the local tier. When escalation happens, the calling model states it explicitly and reports which tier succeeded.

---

## Why This Works

### Constraints That Help Local Models

**Scope locking** — Every task prompt includes:
```
Allowed files (ONLY modify these):
- path/to/file.go

Constraints:
- DO NOT modify any files other than those listed above.
- DO NOT delete or remove existing functions or tests.
```

Local models are prone to scope creep. Explicit boundaries keep them focused.

**Small tasks** — Plans break work into single-concern tasks. A local model can implement one function, add one test, or wire one component. It struggles with "refactor the authentication system."

**Verified foundations** — Sequential execution means task N runs only after task N-1 succeeds. Later tasks can assume prior work is correct.

**Machine-verifiable acceptance** — "Tests pass" and "endpoint returns 200" are checkable. "Code is clean" is not.

### Benefits at Scale

These same constraints help larger models too:
- Less hallucination (explicit paths, no searching)
- Faster execution (smaller context, focused scope)
- Better auditability (one task, one change)
- Cheaper (most work done by local/balanced tiers)

---

## Example: Local Model Success

A typical task delegated to `qwen3-coder-next`:

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

This works because:
1. **One file** — no navigation needed
2. **One method** — fits in attention span
3. **Clear inputs/outputs** — Email, Name → error or nil
4. **Testable** — model can verify its own implementation

---

## Skills

The `skills/` directory contains reference documents that agents can retrieve during execution. Each skill provides domain knowledge or workflow instructions.

### Workflow Skills

| Skill | Purpose |
|-------|---------|
| [create-plan](skills/create-plan/SKILL.md) | Structured plans optimized for local model execution |
| [plan-to-beads](skills/plan-to-beads/SKILL.md) | Convert plans to trackable epics with subtasks |
| [epic-exec](skills/epic-exec/SKILL.md) | Execute subtasks with automatic model escalation |
| [using-beads](skills/using-beads/SKILL.md) | Task lifecycle: start, complete, sync, verify |
| [crush-run-delegate](skills/crush-run-delegate/SKILL.md) | Delegate to models via `crush run` |

### Domain Skills

| Skill | Purpose |
|-------|---------|
| [effective-go](skills/effective-go/SKILL.md) | Go best practices for idiomatic code |
| [github-pr-review](skills/github-pr-review/SKILL.md) | PR review with GitHub CLI |
| [git-worktree](skills/git-worktree/SKILL.md) | Parallel workspaces for isolated agent tasks |
| [otel-mdatagen](skills/otel-mdatagen/SKILL.md) | OpenTelemetry metadata generator reference |

Skills are exposed as MCP tools when using [Crush](https://github.com/charmcli/crush). The [AGENTS.md](crush/AGENTS.md) memory file triggers skill retrieval based on context.

---

## AGENTS.md

The `crush/AGENTS.md` file is loaded as global context, providing explicit rules that guide skill invocation:

```markdown
YOU MUST ALWAYS:

* Use the mcp filesystem tools for file operations
* Use the mcp_skills_effective_go tool before reviewing, writing, or refactoring Go code
* Use the mcp_skills_create_plan tool before creating or making a plan
* Use the mcp_skills_plan_to_beads tool before creating a beads epic from a plan
...
```

This removes ambiguity about when to retrieve skills. Instead of hoping the model recognizes context, explicit rules trigger the right tool calls.

---

## Local Models

The Crush configuration includes several local models that work well with these workflows:

| Model | Context | Notes |
|-------|---------|-------|
| [Qwen3 Coder Next (80B-A3B)](https://huggingface.co/Qwen/Qwen3-Coder-Next) | 128K | Latest Qwen coder, Q8_0 quantization |
| [Qwen3 Coder Flash (30B-A3B)](https://huggingface.co/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF) | 128K | Smaller and faster, Unsloth BF16 |
| [GLM 4.7 Flash (30B-A3B)](https://huggingface.co/unsloth/GLM-4.7-Flash-GGUF) | 128K | Reasoning support, Unsloth BF16 |
| [GLM 4.5 Air (106B-A12B)](https://huggingface.co/unsloth/GLM-4.5-Air-GGUF) | 128K | Reasoning support, Unsloth Q4_K_XL |
| [Devstral 2 (24B)](https://huggingface.co/unsloth/Devstral-Small-2-24B-Instruct-2512-GGUF) | 200K | Dense, slower but good for long tasks, Unsloth BF16 |

All are served via [LM Studio](https://lmstudio.ai/) with the OpenAI-compatible API, running on an NVIDIA RTX 6000 Pro Workstation Edition (96GB VRAM) for fast prompt processing and token generation.

---

## Qwen Template

The `qwen/template.jinja` file provides a simple, reliable chat template for Qwen3 Coder models (Flash and Next) in LM Studio.

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

## Current State

Evaluation of this philosophy is ongoing. Experimenting with local models has been fun and has resulted in general workflow improvements that benefit SOTA models too.

That said, the hardware required to run local models capable enough for day-to-day professional development has become expensive. Until 30-80B parameter models make another leap in capability, I cannot recommend investing thousands of dollars on local hardware for this purpose.

The workflows here remain valuable—they work even better with cloud models—but the "local-first" aspiration is currently ahead of what's practical for most developers.

---

## Directory Structure

```
agents/
├── README.md
├── skills/
│   ├── create-plan/SKILL.md
│   ├── plan-to-beads/SKILL.md
│   ├── epic-exec/SKILL.md
│   ├── using-beads/SKILL.md
│   ├── crush-run-delegate/SKILL.md
│   ├── effective-go/SKILL.md
│   ├── github-pr-review/SKILL.md
│   ├── git-worktree/SKILL.md
│   └── otel-mdatagen/SKILL.md
└── crush/
    ├── AGENTS.md
    └── crush.json
```
