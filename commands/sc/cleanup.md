---
allowed-tools: [Read, Grep, Glob, Bash, Edit, MultiEdit, TodoWrite, Task]
description: "Clean up code, remove dead code, consolidate duplicates, and optimize for efficiency with safe refactoring"
---

# /sc:cleanup - Code and Project Cleanup

## Purpose
Systematically clean up code with deep analysis: remove dead code, consolidate duplicates, identify spider-web/spaghetti code, and optimize for efficiency. Uses a safe refactoring process for critical code.

## Usage
```
/sc:cleanup [target] [--type code|imports|duplicates|efficiency|spider|all] [--safe|--aggressive] [--dry-run]
```

## Arguments
- `target` - Files, directories, or entire project to clean
- `--type` - Cleanup type:
  - `code` - Dead code removal
  - `imports` - Unused imports
  - `duplicates` - Consolidate duplicate code
  - `efficiency` - Performance and efficiency issues
  - `spider` - Spider-web/spaghetti code detection
  - `all` - Full comprehensive cleanup (default)
- `--safe` - Conservative cleanup with comment-test-delete for critical code (default)
- `--aggressive` - Direct cleanup (use with caution)
- `--dry-run` - Preview changes without applying them

## Cleanup Categories

### 1. Dead Code Detection
- Unused functions/methods
- Unreachable code blocks
- Commented-out code blocks
- Unused variables and constants
- Orphaned imports

### 2. Duplicate Code Consolidation
- Identify repeated code patterns (3+ occurrences)
- Extract to shared utilities/helpers
- Create reusable abstractions
- DRY principle enforcement

### 3. Efficiency Analysis
- O(n²) or worse algorithms in hot paths
- Unnecessary re-renders/re-computations
- Missing memoization opportunities
- Redundant database/API calls
- Inefficient loops and iterations

### 4. Spider-Web Code Detection
- Circular dependencies
- Deep nesting (>4 levels)
- Functions with >5 parameters
- Files with >500 lines
- Classes/modules with >10 dependencies
- Tangled import chains

## Safe Refactoring Process

### For NON-CRITICAL Code (direct cleanup):
```
1. Analyze → 2. Delete → 3. Verify build/lint passes
```

### For CRITICAL Code (comment-test-delete):
```
1. Deep Analysis
   ├── Trace all callers/dependencies
   ├── Identify test coverage
   └── Assess impact radius

2. Comment Out (don't delete yet)
   ├── Add TODO marker: // CLEANUP: [date] - Testing removal of [description]
   └── Keep original code commented

3. Test
   ├── Run existing tests
   ├── Run build/typecheck
   └── Manual verification if needed

4. Delete (only after tests pass)
   └── Remove commented code and TODO marker
```

### Critical Code Indicators (use comment-test-delete):
- Functions called from 3+ locations
- Code in core business logic paths
- Database/API interaction code
- Authentication/authorization code
- Payment/financial calculations
- Code without test coverage
- Shared utilities used across modules

## Execution Workflow

### Phase 1: Deep Analysis
```
┌─────────────────────────────────────────────────────────────┐
│                    DEEP ANALYSIS PHASE                       │
├─────────────────────────────────────────────────────────────┤
│  1. Scan target for all code patterns                       │
│  2. Build dependency graph                                  │
│  3. Identify dead code candidates                           │
│  4. Detect duplicate patterns                               │
│  5. Analyze efficiency hotspots                             │
│  6. Map spider-web connections                              │
│  7. Classify each item as CRITICAL or NON-CRITICAL          │
│  8. Generate cleanup plan with risk assessment              │
└─────────────────────────────────────────────────────────────┘
```

### Phase 2: Cleanup Execution
```
┌─────────────────────────────────────────────────────────────┐
│                   CLEANUP EXECUTION                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  For each cleanup item:                                     │
│                                                             │
│  ┌─────────────┐                                           │
│  │ Is Critical?│                                           │
│  └──────┬──────┘                                           │
│         │                                                   │
│    YES  │  NO                                               │
│    ▼    ▼                                                   │
│  ┌──────────────┐  ┌──────────────┐                        │
│  │ Comment out  │  │ Direct delete│                        │
│  │ with marker  │  │              │                        │
│  └──────┬───────┘  └──────┬───────┘                        │
│         │                 │                                 │
│         ▼                 │                                 │
│  ┌──────────────┐         │                                │
│  │ Run tests    │         │                                │
│  │ Run build    │         │                                │
│  └──────┬───────┘         │                                │
│         │                 │                                 │
│    PASS │  FAIL           │                                │
│    ▼    ▼                 │                                │
│  ┌──────┐ ┌──────────┐    │                                │
│  │Delete│ │Restore & │    │                                │
│  │code  │ │flag for  │    │                                │
│  └──────┘ │review    │    │                                │
│           └──────────┘    │                                │
│                           ▼                                 │
│                    ┌──────────────┐                        │
│                    │ Verify build │                        │
│                    │ passes       │                        │
│                    └──────────────┘                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Phase 3: Validation & Report
```
┌─────────────────────────────────────────────────────────────┐
│                   VALIDATION & REPORT                        │
├─────────────────────────────────────────────────────────────┤
│  1. Run full test suite                                     │
│  2. Run linter and type checker                             │
│  3. Verify no runtime errors                                │
│  4. Generate cleanup report:                                │
│     - Lines removed                                         │
│     - Duplicates consolidated                               │
│     - Efficiency improvements                               │
│     - Spider-web issues resolved                            │
│     - Items flagged for manual review                       │
└─────────────────────────────────────────────────────────────┘
```

## Output Format

### Cleanup Report
```
## Cleanup Summary for [target]

### Dead Code Removed
| File | Lines | Type | Risk Level |
|------|-------|------|------------|
| ... | ... | ... | ... |

### Duplicates Consolidated
| Pattern | Occurrences | Extracted To |
|---------|-------------|--------------|
| ... | ... | ... |

### Efficiency Improvements
| File:Line | Issue | Fix Applied |
|-----------|-------|-------------|
| ... | ... | ... |

### Spider-Web Issues
| Issue | Severity | Resolution |
|-------|----------|------------|
| ... | ... | ... |

### Flagged for Manual Review
| File:Line | Reason |
|-----------|--------|
| ... | ... |

### Statistics
- Total lines removed: X
- Duplicates consolidated: Y
- Efficiency fixes: Z
- Tests: PASS/FAIL
- Build: PASS/FAIL
```

## Claude Code Integration
- Uses Glob for systematic file discovery
- Leverages Grep for pattern detection and dead code identification
- Applies MultiEdit for batch cleanup operations
- Uses Task tool for parallel analysis of large codebases
- Integrates with project test commands (npm test, pytest, etc.)
- Maintains TodoWrite for tracking cleanup progress

## Safety Guarantees
1. Never delete code without analysis
2. Always verify build/tests pass after changes
3. Critical code goes through comment-test-delete cycle
4. Automatic rollback if tests fail
5. Generate detailed report of all changes
6. Flag uncertain items for manual review
