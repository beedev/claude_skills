# Claude Skills

A collection of Claude Code slash commands for quality-driven development.

Built around the **Bharath persona** — craftsman-engineer standards, approval gates, and SRC-validated planning.

## Skills

| Skill | Description |
|-------|-------------|
| `/orchestrate` | Full development lifecycle orchestrator — Plan → Craft → Test → Review → Ship |
| `/mcp:build` | Scaffold a production-ready custom MCP server from a plain-English description |
| `/mcp:setup` | Add, configure, and validate MCP servers in Claude settings (user or project scope) |
| `/sc:implement` | Feature implementation with intelligent persona activation |
| `/sc:analyze` | Code quality, security, performance, and architecture analysis |
| `/sc:improve` | Systematic improvements with evidence-based recommendations |
| `/sc:design` | System architecture, API, and component interface design |
| `/sc:build` | Build, compile, and package with error handling |
| `/sc:test` | Test execution, coverage reporting, and maintenance |
| `/sc:git` | Git operations with intelligent commit messages |
| `/sc:troubleshoot` | Diagnose and resolve issues in code or system behavior |
| `/sc:explain` | Clear explanations of code, concepts, or system behavior |
| `/sc:document` | Focused documentation for components or features |
| `/sc:cleanup` | Remove dead code, consolidate duplicates, optimize |
| `/sc:workflow` | Structured implementation workflows from PRDs |
| `/sc:task` | Complex task execution with cross-session persistence |
| `/sc:spawn` | Break complex tasks into coordinated subtasks |
| `/sc:estimate` | Development estimates for tasks, features, or projects |
| `/sc:context` | Save and load session context across sessions |
| `/sc:load` | Load and analyze project context and dependencies |
| `/sc:index` | Generate comprehensive project documentation |
| `/sc:project-briefing` | Engaging project briefing in plain language |
| `/sc:code-analyzer` | Analyze code changes against requirements |

## MCP Skills

```bash
# Build a custom MCP server
/mcp:build "tools to query our internal REST API and cache in Redis" --name my-api --lang ts
/mcp:build "read/write Notion pages and databases" --lang py --setup

# Wire up well-known servers (10 in catalog)
/mcp:setup playwright
/mcp:setup postgres --env DATABASE_URL=postgresql://localhost/mydb
/mcp:setup github --env GITHUB_PERSONAL_ACCESS_TOKEN=ghp_xxx
/mcp:setup brave-search

# Project-scoped (shared with team via git)
/mcp:setup playwright --scope project

# Wire up a server you just built
/mcp:setup my-api --path ./mcp-servers/my-api

# Remove
/mcp:setup playwright --remove
```

## Install

### Option A — One-liner (single skill)

```bash
# Install /orchestrate
curl -o ~/.claude/commands/orchestrate.md \
  https://raw.githubusercontent.com/beedev/claude_skills/main/commands/orchestrate.md
```

### Option B — Clone and symlink (stay in sync with updates)

```bash
git clone https://github.com/beedev/claude_skills.git ~/claude_skills

# Link all skills
ln -sf ~/claude_skills/commands/orchestrate.md ~/.claude/commands/orchestrate.md

# Link the sc/ suite
mkdir -p ~/.claude/commands/sc
for f in ~/claude_skills/commands/sc/*.md; do
  ln -sf "$f" ~/.claude/commands/sc/$(basename "$f")
done
```

To update later:
```bash
cd ~/claude_skills && git pull
```

### Option C — Project-level (shared with your whole team)

Copy into your project repo so everyone gets the skills automatically:

```bash
mkdir -p .claude/commands/sc
cp ~/claude_skills/commands/orchestrate.md .claude/commands/
cp ~/claude_skills/commands/sc/*.md .claude/commands/sc/
git add .claude/commands && git commit -m "feat: add claude skills"
```

## Usage

```
/orchestrate add OAuth2 login to the API --pr
/orchestrate fix the memory leak in the retriever --dev alice
/orchestrate list
/orchestrate status

/sc:analyze --focus security
/sc:implement add dark mode to settings panel
/sc:improve --quality
/sc:git
```

## `/orchestrate` workflow

```
Phase 1 — Ultrathink & Plan    → Explore agent + SRC gap validation (≥95%) + approval gate
Phase 2 — Craft                → Implementation agent in isolated worktree
Phase 3 — Prove It Works       → Test engineer agent + RCA fix cycles (max 3)
Phase 4 — Leave It Better      → Code reviewer agent
Phase 5 — Ship                 → Final approval gate → push / PR
```

Non-negotiables: plan approved before code, tests pass before ship, root cause not symptoms, codebase better than you found it.
