---
allowed-tools: [Bash, Read, TodoWrite]
description: "Run a Codex CLI code review on changes made by Claude. Reviews staged, unstaged, and uncommitted changes by default, or against a base branch."
---

# /codex-review — Codex Code Review

Run `codex review` on current git changes. Uses OpenAI's Codex CLI to provide an independent, AI-powered code review of what Claude just wrote.

## Usage
```
/codex-review [--base <branch>] [--commit <sha>] [--focus <area>] [--title <title>]
```

## Arguments
- `--base <branch>` — Review changes against a specific base branch (e.g. `--base main`)
- `--commit <sha>` — Review a specific commit's changes
- `--focus <area>` — Custom review focus: security, performance, correctness, style, tests
- `--title <title>` — Title to display in the review summary
- (no args) — Reviews all uncommitted changes (staged + unstaged)

---

## Execution

### Step 1: Check Prerequisites

```bash
# Verify codex CLI is available
which codex || echo "MISSING"
```

If `codex` is not found, tell the user to install it: `npm install -g @openai/codex` and stop.

### Step 2: Assess Git State

```bash
git status --short
git diff --stat
git diff --cached --stat
```

Summarize what's about to be reviewed:
- How many files changed
- Insertions / deletions count
- Which files are staged vs unstaged

If there are **no changes at all**, report: "Nothing to review — working tree is clean." and stop.

### Step 3: Determine Review Scope

Parse arguments from `$ARGUMENTS`:

- If `--base <branch>` is present → use `codex review --base <branch>`
- If `--commit <sha>` is present → use `codex review --commit <sha>`
- Default → use `codex review --uncommitted`

If `--title` is provided, append `--title "<value>"`.

Build the focus prompt:
- If `--focus security` → prompt = "Focus on security vulnerabilities, injection risks, authentication issues, and data exposure"
- If `--focus performance` → prompt = "Focus on performance bottlenecks, inefficient algorithms, memory leaks, and unnecessary re-computation"
- If `--focus correctness` → prompt = "Focus on logic errors, edge cases, null/undefined handling, and incorrect assumptions"
- If `--focus style` → prompt = "Focus on code style, naming conventions, consistency, and readability"
- If `--focus tests` → prompt = "Focus on test coverage gaps, missing edge case tests, and brittle test patterns"
- Default → prompt = "Review for correctness, security, performance, and code quality. Highlight any bugs, vulnerabilities, or improvements."

### Step 4: Run Codex Review

Execute with a 120 second timeout:

```bash
codex review --uncommitted "<prompt>"
# or
codex review --base <branch> "<prompt>"
# or
codex review --commit <sha> "<prompt>"
```

Capture the full output.

### Step 5: Present Results

Format the review output clearly:

```
## Codex Review Results

**Scope**: [uncommitted changes / changes vs <branch> / commit <sha>]
**Files reviewed**: [count]
**Focus**: [area or "general"]

---

[codex output verbatim]

---

### Next Steps
- Address any critical issues before committing
- Run `yarn test` to verify no regressions
- Use `/sc:improve` to fix specific issues found
```

If codex exits with an error (non-zero exit code), show the error and suggest:
1. Check `codex --help` for usage
2. Verify you are logged in: `codex login`
3. Check if OPENAI_API_KEY is set in the environment

### Step 6: Summary Verdict

After presenting results, give a one-line verdict:

- ✅ **Clean** — No significant issues found. Safe to commit.
- ⚠️ **Review suggested** — Minor issues found, review before committing.
- ❌ **Issues found** — Critical issues detected. Fix before committing.

## Claude Code Integration

- Uses `Bash` to run git commands and `codex review`
- Uses `Read` to inspect specific files flagged by the review
- Uses `TodoWrite` to track issues found if `--track` flag is provided
- Never modifies files — this is a read-only review command
