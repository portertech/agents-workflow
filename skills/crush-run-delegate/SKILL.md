---
name: crush-run-delegate
description: Delegate simple or parallel tasks to other models using crush run.
---

# Crush Run - Model Delegation Reference

Use `crush run` to delegate tasks to different models mid-session. Tasks run non-interactively and return results to the calling session.

> **CRITICAL**: Never use `run_in_background=true` with `crush run`. It returns immediately with no result. Always run `crush run` synchronously—the bash tool will wait for completion.

## Key Concept: Full Tool Access

Models invoked via `crush run` have **full tool access** — they can read/write files, execute bash commands, search the codebase, and use all available MCP tools. They are not limited to text-only responses. Delegate tasks that require file operations, code changes, or system interaction with confidence.

## Model Selection Guide

| Model | Class | Best For | Cost |
|-------|-------|----------|------|
| `lmstudio/qwen/qwen3-coder-next` | local | code generation, diffs, simple tasks | free |
| `openai/gpt-5.2-codex` | balanced | refactoring, code review, analysis, summarization, general tasks | $ |
| `anthropic/claude-opus-4-6` | capable | complex reasoning, architecture, security review, research | $ |

## Quick Reference

```bash
crush run -m MODEL "prompt"              # run with specific model
crush run -m PROVIDER/MODEL "prompt"     # disambiguate model name
crush run -q -m MODEL "prompt"           # quiet mode (no spinner)
echo "data" | crush run -m MODEL "prompt" # pipe input
```

## Model Specification

```bash
# By model name (if unique across providers)
crush run -m "gpt-5.2-codex" "prompt"

# By provider/model (recommended for clarity)
crush run -m "lmstudio/qwen/qwen3-coder-next" "prompt"
crush run -m "openai/gpt-5.2-codex" "prompt"
crush run -m "anthropic/claude-opus-4-6" "prompt"

# List available models
crush models
```

## Single Task Delegation

### Simple delegation

```bash
# Delegate a focused task to another model
result=$(crush run -m "lmstudio/qwen/qwen3-coder-next" "Write a regex to match email addresses")
echo "$result"
```

### With file context

```bash
# Analyze a file with a specific model
crush run -m "openai/gpt-5.2-codex" "Review this code for security issues" < src/auth.go

# Summarize documentation
cat README.md | crush run -m "openai/gpt-5.2-codex" "Summarize in 3 bullet points"
```

### With piped data

```bash
# Analyze command output
git diff HEAD~5 | crush run -m "lmstudio/qwen/qwen3-coder-next" "Summarize these changes"
```

## Parallel Task Execution

Run multiple model tasks simultaneously using shell background jobs (`&` + `wait`).

> **CRITICAL**: Do NOT use the bash tool's `run_in_background` parameter for `crush run` — use shell `&` and `wait` instead.

| Mechanism | Use For | How It Works |
|-----------|---------|---------------|
| Shell `&` + `wait` | Parallel `crush run` tasks | Bash manages jobs; `wait` blocks until all complete |
| `run_in_background=true` | Long-running servers | Returns shell ID; requires `job_output` polling |

### Basic parallel pattern

```bash
# Run tasks in parallel, collect results
crush run -q -m "model-a" "task 1" > /tmp/result1.txt &
crush run -q -m "model-b" "task 2" > /tmp/result2.txt &
crush run -q -m "model-c" "task 3" > /tmp/result3.txt &
wait

# Results available in /tmp/result1.txt, /tmp/result2.txt, /tmp/result3.txt
```

### Parallel code review

```bash
# Different models check different aspects
crush run -q -m "model-a" "Check for security issues" < code.go > /tmp/security.txt &
crush run -q -m "model-b" "Check for performance issues" < code.go > /tmp/perf.txt &
crush run -q -m "model-c" "Check for code style issues" < code.go > /tmp/style.txt &
wait
```

## Integration with Crush Session

When running inside a Crush session, use the bash tool to delegate:

```bash
# From within Crush, delegate to another model
result=$(crush run -q -m "other-model" "subtask prompt")

# Use the result in current session
echo "The other model said: $result"
```

### Parallel delegation from session

```bash
# Spawn parallel tasks from within Crush
crush run -q -m "model-a" "task 1" > /tmp/a.txt &
crush run -q -m "model-b" "task 2" > /tmp/b.txt &
wait

# Combine results
cat /tmp/a.txt /tmp/b.txt
```

## Flags Reference

| Flag | Description |
|------|-------------|
| `-m, --model` | Model to use (`model` or `provider/model`) |
| `-q, --quiet` | Hide spinner (useful for scripting) |
| `-c, --cwd` | Working directory for the task |

## Tips

1. **Always use `-q`** when capturing output to avoid spinner artifacts
2. **Use provider/model format** to avoid ambiguity across providers
3. **Model names may include dates** - Anthropic models sometimes have date suffixes (e.g., `claude-opus-4-5-20251101`)
4. **Check `crush models`** to see available models, their exact names, and providers
5. **Pipe large context** via stdin rather than command arguments
6. **Background jobs** need explicit `wait` before reading results
7. **Temp files** are safer than variables for large outputs
8. **Never use `run_in_background=true`** with `crush run` — see Parallel Task Execution above
