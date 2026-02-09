---
name: github-pr-review
description: Provides instructions of how to review a github pull-request (using the GitHub CLI).
---

# GitHub CLI - PR Review Reference

## Fork-safe rule

**Always use PR URL for maximum safety.** When inside a fork, `gh` defaults to the fork repo—causing "PR not found" errors. Using the full PR URL works reliably from any clone or directory.

```bash
# Example: Use full PR URL
gh pr view https://github.com/OWNER/REPO/pull/123
gh pr diff https://github.com/OWNER/REPO/pull/123

# Alternatively, use -R OWNER/REPO
gh pr view 123 -R OWNER/REPO
```

## Quick Reference

```bash
gh pr view URL                         # RECOMMENDED: always use PR URL
gh pr diff URL                         # view changes (use URL)
gh pr diff URL --name-only             # files only (use URL)
gh pr checks URL                       # CI status (use URL)
gh pr checks URL --watch               # wait for CI (use URL)
gh pr view 123 -R OWNER/REPO           # by number (fallback)
gh pr list -R OWNER/REPO --search "review-requested:@me"
```

## Inspect a PR

### View metadata

```bash
gh pr view URL                    # basic info (use URL)
gh pr view URL --comments         # include comments (use URL)
gh pr view URL --json FIELDS      # structured output (use URL)
```

**JSON fields:** `title`, `body`, `author`, `state`, `baseRefName`, `headRefName`, `files`, `additions`, `deletions`, `changedFiles`, `commits`, `comments`, `reviews`, `reviewRequests`, `reviewDecision`, `labels`, `mergeable`, `statusCheckRollup`

### View diff

```bash
gh pr diff URL                    # full diff (use URL)
gh pr diff URL --name-only        # file paths only (use URL)
gh pr diff URL --patch            # patch format (use URL)
```

### Extract specific data

```bash
# File paths
gh pr view URL --json files --jq '.files[].path'

# Commit messages
gh pr view URL --json commits --jq '.commits[].messageHeadline'

# Review states
gh pr view URL --json reviews --jq '.reviews[]|{author:.author.login,state:.state}'

# Merge readiness
gh pr view URL --json mergeable,reviewDecision,statusCheckRollup
```

## List PRs

```bash
gh pr list -R OWNER/REPO                # open (default)
gh pr list -R OWNER/REPO --state all|closed|merged
gh pr list -R OWNER/REPO --limit 50
gh pr list -R OWNER/REPO --author @me
gh pr list -R OWNER/REPO --base main
gh pr list -R OWNER/REPO --label bug
gh pr list -R OWNER/REPO --json number,title,author
```

### Search syntax

```bash
gh pr list -R OWNER/REPO --search "QUERY"
```

| Query | Meaning |
|-------|---------|
| `review:required` | needs review |
| `review:approved` | approved |
| `review:changes_requested` | changes requested |
| `review-requested:@me` | assigned to you |
| `reviewed-by:@me` | you reviewed |
| `draft:false` | not draft |
| `status:success\|failure` | CI status |
| `created:>2024-01-01` | date filter |

## API Queries

For data not available via `gh pr`:

```bash
# Changed files with stats
gh api $R repos/{owner}/{repo}/pulls/123/files \
  --jq '.[]|"\(.filename) +\(.additions)/-\(.deletions)"'

# Review comments
gh api $R repos/{owner}/{repo}/pulls/123/comments \
  --jq '.[]|{file:.path,line:.line,body:.body}'

# All commits
gh api $R repos/{owner}/{repo}/pulls/123/commits --jq '.[].commit.message'
```

## Tips

1. **Always use PR URL** for `view`, `diff`, `checks` commands—works from any directory or fork
2. Use `-R OWNER/REPO` only for `list` and `api` commands
3. `--json` + `--jq` for precise extraction
4. `@me` = your username in queries
5. Date ranges: `created:>DATE`, `updated:<DATE`
6. `-R` works with both `gh pr` and `gh api`
