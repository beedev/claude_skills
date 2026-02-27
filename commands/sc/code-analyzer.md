---
allowed-tools: [Read, Grep, Glob, Write, Edit, TodoWrite]
description: "Analyze code changes against user requirements and store analysis in context/code-analysis.md"
---

# /sc:code-analyzer - Code Analysis & Requirements Tracking

## Purpose
Analyze what is being done and how it aligns with user requirements. Tracks implementation decisions, patterns used, and deviations from requirements. Stores analysis in `context/code-analysis.md`.

## Usage
```
/sc:code-analyzer [--full|--incremental] [--since <commit>]
```

## Arguments
- `--full` - Complete analysis of all recent changes
- `--incremental` - Only analyze since last analysis (default)
- `--since <commit>` - Analyze changes since specific commit

## Analysis Categories

### 1. Requirements Alignment
- What was requested vs what was implemented
- Feature completeness check
- Scope creep detection
- Missing requirements identification

### 2. Implementation Patterns
- Design patterns used
- Code organization decisions
- Architectural choices made
- Technology/library selections

### 3. Quality Metrics
- Code complexity introduced
- Test coverage changes
- Technical debt added/removed
- Performance implications

### 4. Decision Log
- Why certain approaches were chosen
- Trade-offs considered
- Alternatives rejected and why
- Future considerations noted

## Execution Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                    CODE ANALYSIS FLOW                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. GATHER CONTEXT                                          │
│     ├── Read context/context.md for current task           │
│     ├── Get git diff of recent changes                     │
│     ├── Identify modified files and their purposes         │
│     └── Extract user requirements from conversation        │
│                                                             │
│  2. ANALYZE CHANGES                                         │
│     ├── Map changes to requirements                        │
│     ├── Identify patterns and decisions                    │
│     ├── Detect quality implications                        │
│     └── Note any deviations or scope changes               │
│                                                             │
│  3. GENERATE REPORT                                         │
│     ├── Requirements coverage matrix                       │
│     ├── Implementation summary                             │
│     ├── Decision rationale                                 │
│     └── Recommendations for next steps                     │
│                                                             │
│  4. UPDATE ANALYSIS FILE                                    │
│     └── Append to context/code-analysis.md                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Output Format (context/code-analysis.md)

```markdown
# Code Analysis Log

## Analysis Entry - [YYYY-MM-DD HH:MM]

### Session Summary
- **Task**: [What was being worked on]
- **Compact Count**: [N of 5]

### Requirements Tracking
| Requirement | Status | Implementation Notes |
|-------------|--------|---------------------|
| ... | ✅/🔄/❌ | ... |

### What Was Done
- [Change 1]: [File] - [Description]
- [Change 2]: [File] - [Description]

### How It Was Done
- **Pattern Used**: [Pattern name/approach]
- **Rationale**: [Why this approach]
- **Alternatives Considered**: [What else could have been done]

### Files Modified
| File | Change Type | Purpose |
|------|-------------|---------|
| ... | add/modify/delete | ... |

### Quality Assessment
- **Complexity**: [Low/Medium/High]
- **Test Coverage**: [Covered/Partial/None]
- **Tech Debt**: [Added/Neutral/Reduced]

### Deviations & Notes
- [Any scope changes, issues encountered, or important notes]

### Next Steps
- [ ] [Pending item 1]
- [ ] [Pending item 2]

---
```

## Auto-Trigger Behavior

This skill is automatically triggered:
- After every 5 context compactions
- When explicitly invoked via `/sc:code-analyzer`

The compact counter is tracked in `context/.compact-count`.

## Integration
- Reads from `context/context.md` for current task state
- Appends analysis to `context/code-analysis.md`
- Uses git history for change tracking
- Integrates with TodoWrite for pending items
