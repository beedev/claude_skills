---
allowed-tools: [Read, Write, Bash]
description: "Save and load session context - persistent memory across sessions"
---

# /sc:context - Session Context Management

## Purpose
Persist important decisions and instructions across sessions. Restore context after compact or new session start.

## Usage

Parse the command argument to determine which subcommand to run:

### If argument starts with "save "
Extract the message after "save " and save it to persistent memory.

**Example:** `/sc:context save "Always use ./dev.sh local for development"`

**Action:**
1. Create ~/.claude/memory/ directory if it doesn't exist
2. Append timestamped entry to ~/.claude/memory/decisions.md
3. Format: `## YYYY-MM-DD HH:MM` followed by the message on next line
4. Confirm what was saved

### If argument is "load"
Load all saved context into current session.

**Example:** `/sc:context load`

**Action:**
1. Read ~/.claude/memory/decisions.md (global decisions)
2. Read ./context/context.md if it exists (project-specific context)
3. Read ./context/decisions.md if it exists (project-specific decisions)
4. Display content with clear section headers
5. Acknowledge what was loaded and summarize key points

### If argument is "show"
Display saved decisions in compact format.

**Example:** `/sc:context show`

**Action:**
1. Read ~/.claude/memory/decisions.md
2. Display last 20 entries in compact format
3. Show count of total entries

### If argument is "clear"
Clear saved decisions after confirmation.

**Example:** `/sc:context clear`

**Action:**
1. Ask for confirmation before clearing
2. If confirmed, delete ~/.claude/memory/decisions.md
3. Confirm deletion

### If no argument or "help"
Show usage information.

## File Locations

```
~/.claude/memory/
└── decisions.md    # Global decisions (persists across all projects)

<project>/context/
├── context.md      # Auto-saved by PreCompact hook (project-specific)
└── decisions.md    # Project-specific decisions (optional)
```

## File Format (decisions.md)

```markdown
# Session Decisions

## 2025-01-31 10:30
Always use `./dev.sh local` for development, never raw python commands

## 2025-01-31 10:45
In plan mode, never edit files - only the plan file is allowed

## 2025-01-31 11:00
This project uses PostgreSQL on port 5432 for local dev
```

## Implementation Notes

- Use Bash to create directories: `mkdir -p ~/.claude/memory`
- Use Write tool to append to decisions.md (read first, then write with new content appended)
- Use Read tool to display content
- Format timestamps using: `date '+%Y-%m-%d %H:%M'`
- When loading, clearly separate global vs project-specific context
- If files don't exist, report that gracefully (not an error)
