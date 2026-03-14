# /migrate — CodeLoom Full-Autonomy Migration Skill

Claude Code CLI is the orchestrator. CodeLoom is a read-only intelligence source (AST, ASG, code units, RAG).
Claude drives: discovery, architecture design, MVP clustering, functional specs, and automated transforms.

## Prerequisites

The CodeLoom MCP server must be running. Verify by calling `codeloom_list_projects`.
If not found: `./dev.sh setup-mcp` in the CodeLoom directory, then restart Claude Code.

---

## Usage

```
/migrate [init]              — Full interactive init: discover → architect → plan → begin transforms
/migrate run [--mvp <name>]  — Transform all MVPs, then compare + auto-fix automatically
/migrate compare [--mvp <name>]  — Re-run accuracy comparison + auto-fix on completed output
/migrate status              — Show plan status (+ accuracy score if report exists)
/migrate resume              — Pick up an existing plan
```

After a full `run` (all MVPs complete), compare and auto-fix fire automatically using source
units already in CodeLoom. Results are persisted via `codeloom_save_accuracy_report` and shown
in the CodeLoom migration dashboard.

**Argument parsing**:
1. First word (if not a flag) is the subcommand. Default: `init`.
2. `--mvp <name>` targets a specific MVP by name (fuzzy match). Omit to run all pending MVPs in priority order.

---

## Workflow: `init` (default — `/migrate` or `/migrate init`)

### Step 1 — Project Selection

1. Call `codeloom_list_projects` — display formatted table (name, ID, file count).
2. If exactly 1 project: auto-select, confirm with user.
3. If >1 projects: use `AskUserQuestion` — pick project by number.

### Step 2 — Migration Goal Prompts (`AskUserQuestion`)

Collect in a **single** multi-question call:

```
Q1: What is your migration goal? (free text — what are you migrating to and why?)
Q2: Target stack
    Options: TypeScript + Express / Python + FastAPI / Java + Spring Boot / .NET Core / Other
Q3: Migration style
    Options: Strangler Fig (incremental, low risk) / Big Bang (full rewrite) / Hybrid
Q4: Output directory
    Options: migration-output/<project_id>/ (Recommended) / Custom path / Other
```

### Step 3 — Intelligence Gathering

Run all in parallel:

1. `codeloom_get_project_intel(project_id)` — language breakdown, unit count, existing plans
2. `codeloom_get_import_graph(project_id)` — **import fan-in analysis**: shared files + full import edge list
3. `codeloom_search_codebase(project_id, "entry point main application startup")`
4. `codeloom_search_codebase(project_id, "authentication middleware API routes")`
5. `codeloom_search_codebase(project_id, "database configuration data models")`
6. `codeloom_list_units(project_id)` — **paginate until exhausted** (call until `page * page_size >= total`)
   - Group units by: language, unit_type, file path prefix (first 2 path segments)
   - Build `all_source_files` = complete set of every `file_path` across all pages

After gathering, extract from `codeloom_get_import_graph` result:
- **`shared_files`**: files with `importer_count >= 3` — these are shared infrastructure.
  They MUST go into Foundation (MVP 0) regardless of directory location.
  Examples: `utils.ts`, `types.ts`, `constants.ts`, `helpers.ts` at module roots.
- **`import_edges`**: used in Step 5.5 to verify every file is covered.

7. **Check existing asset strategies**: If the most recent plan has `asset_strategies`:
   - Extract per-language strategies (especially JCL sub-type classifications)
   - Log: "Asset strategies found: {summary of strategies per language}"
   - These are the user's confirmed classifications — MUST be honored in Step 5

8. **Auto-detect migration lanes**: From `codeloom_get_project_intel` response:
   - Check `detected_lanes` array for lane matches (confidence > 0.3)
   - Check `file_breakdown` for mainframe languages (cobol, pli, jcl → load mainframe lane)
   - For each match, the lane sub-skill's transform tables, quality gates, and pitfalls
     become active guidance for Steps 4-5
   - Log which lanes were loaded and why

### Step 4 — Architecture Reasoning

Analyze gathered data and produce a concise architecture document covering:

- **Source summary**: Framework, patterns, key design choices identified
- **Target architecture**: How the codebase maps to the target stack
- **Key transforms**: Specific high-value mappings (e.g., "LangGraph nodes → Express routers")
- **DI strategy**: How dependencies will be wired in the target
- **Target file structure**: Unified `src/` layout — all MVPs write into the same tree, not separate directories
- **Risks**: Anti-patterns to avoid, tricky patterns needing manual attention

### Step 5 — MVP Cluster Definition + Functional Specs

**Before clustering**, extract shared files from Step 3:
- Any file with `importer_count >= 3` belongs in MVP 0 unconditionally.
- Do not assign these files to any other MVP.

Propose MVP clusters based on:
- Natural module groupings from unit analysis (file path prefixes, shared imports)
- Dependency order: independent MVPs first, orchestration/integration last
- Priority ladder: `Foundation & Setup → Core Domain → Integrations → Orchestration → Tests`
- **Tests are a dedicated final MVP** — all `*.test.ts`, `*.spec.ts`, `*.test.py`, `evals/` files belong here

**Asset-Strategy-Aware Clustering** (when asset_strategies exist):

1. For each language in asset_strategies:
   - `no_change` → EXCLUDE entirely from all MVPs
   - `keep_as_is` → synthetic "Keep As-Is" MVP (copy only)
   - Active strategies → include in clustering normally

2. For JCL with sub_type strategies:
   - compile_link: no_change → SKIP entirely
   - sort_merge: rewrite → "Batch Utilities" MVP (early priority, after Foundation)
   - data_mgmt: rewrite → Group with sort_merge in "Batch Utilities" MVP
   - application_run: convert → Group each run step with its program's MVP
   - proc_invoke: convert → "Job Orchestration" MVP or with programs

3. For COBOL with program_category metadata:
   - batch_program → standard batch MVP clustering
   - cics_online → "Online Services" MVP (REST API endpoints)
   - ims_dli → group with ORM/database MVP
   - batch_db2 → group with database integration MVP
   - copybook → Foundation MVP (shared type definitions)

4. Fallback (no asset_strategies): classify client-side using metadata from unit listing

**JCL Transform Patterns** (when JCL files present):
- sort_merge → Python streaming (heapq.merge, SORT FIELDS 1-based→0-based, INCLUDE/OMIT predicates) or shell sort
- data_mgmt → Python streaming (REPRO=copy, IEBGENER=copy, IEFBR14=skip, GDG=versioned naming) or shell cp
- application_run → Shell script (export DD vars, call migrated Python/Java module)
- compile_link → SKIP (do not generate)
- proc_invoke → Shell script calling resolved proc equivalent

**Lane Quality Gates** (if lane sub-skill is active):
- Run each quality gate from the lane's checklist against the generated output
- Log pass/fail for each gate
- Blocking gates prevent MVP completion (same as compile failure)
- Advisory gates are logged but don't block

For each MVP, produce a **functional spec** in this format:

```
MVP <N>: <Name>
  Description: <what this cluster covers>
  Source files: [list — use exact paths from codeloom_list_units]
  Target path: <output_dir>/src/<target-subpath>/
  Depends on: [MVP names or "none"]
  Why this order: <rationale>

  Functional Spec:
    Purpose: <one sentence — what this MVP achieves in the migrated system>
    Key transforms:
      - <source pattern> → <target pattern>
      - <source pattern> → <target pattern>
    Renamed exports (consumed by dependent MVPs):
      | Old symbol                  | New symbol              | Old import path                     | New import path                  |
      |-----------------------------|-------------------------|-------------------------------------|----------------------------------|
      | fooGraph.invoke(state, cfg) | runFoo(params, supabase?)| foo/foo-graph.js                    | foo/index.js                     |
      | BarGraph (StateGraph)       | (removed — plain fn)    | bar/bar-graph.js                    | (deleted — do not import)        |
    Acceptance criteria:
      - All source functionality preserved
      - No framework-specific imports from source stack remain
      - Compiles without errors (tsc --noEmit or equivalent)
      - Exports match interface contracts expected by dependent MVPs
```

The **Renamed exports table is mandatory** for any MVP that migrates orchestrators, graphs, or shared modules.
Be explicit: if `fooGraph.invoke(state, config)` becomes `runFoo(params)`, write that down.
Dependent MVPs use this table to know exactly how to rewrite their import statements.

### Step 5.5 — Uncovered File Audit

After proposing all MVPs, compute:
```
all_source_files = {all file_paths from paginated codeloom_list_units}
covered_files    = union of all source_file_paths across every MVP spec
uncovered_files  = all_source_files − covered_files
```

Auto-assign uncovered files by pattern (no user prompt needed):
- `*.test.ts`, `*.spec.ts`, `*.test.py`, `*.spec.py` → Tests MVP
- `evals/` prefix → Tests MVP
- `tsconfig.json`, `package.json`, `*.config.js`, `*.env*`, `Dockerfile`, `*.yml` → Foundation MVP

For any remaining uncovered files, show the user:
```
The following source files are not assigned to any MVP:
  - src/agents/utils.ts        (imported by 12 files — suggest Foundation)
  - src/utils/firecrawl.ts     (suggest: add to existing MVP or create new)
  ...

Choose: assign each to an MVP / confirm skip (will not be migrated)
```

**Do not proceed to Step 6 until every uncovered file has an explicit assignment or confirmed skip.**

### Step 6 — User Confirmation (`AskUserQuestion`) ← **ONLY APPROVAL GATE**

Show the full MVP list with functional specs and uncovered file decisions. Ask:
```
Options:
  - Approve as-is → proceed to save and begin transforms
  - Adjust MVPs → user describes changes, re-propose, ask again
  - Cancel
```

If "Adjust": apply changes and re-present. Ask again.

### Step 7 — Save to CodeLoom DB + Write Artifacts to Disk

1. `codeloom_save_plan(project_id, target_brief, target_stack_json, architecture_doc, discovery_doc, output_dir)`
2. `codeloom_save_mvps(plan_id, mvps[...])`
   - For each MVP: `{name, description, priority, source_file_paths, depends_on_names}`
3. **Write `SPEC.md` per MVP** to `<output_dir>/<mvp-slug>/SPEC.md`
   - Content: full functional spec from Step 5 (including Renamed exports table)
4. **Write `SYMBOLS.md`** to `<output_dir>/SYMBOLS.md`:
   - One section per MVP, pre-populated with headers, tables filled in as run completes:
   ```markdown
   # Migration Symbols Map
   Generated by /migrate init. Updated after each MVP transform completes.
   Read by /migrate run to resolve renamed exports from prior MVPs.

   ## MVP 0: Foundation & Setup
   _(populated when transform completes)_

   ## MVP 1: <Name>
   _(populated when transform completes)_
   ```
5. Report: "Plan saved. `<N>` MVPs created. Specs written to `<output_dir>/*/SPEC.md`. Starting transforms."

### Step 7.5 — Write CHECKPOINT.md

Write `<output_dir>/CHECKPOINT.md` — the resume bookmark that survives `/clear`:

```markdown
# Migration Checkpoint
<!-- Auto-generated by /migrate. Do not edit manually. -->

## Identity
- project_name: <project_name>
- project_id: <project_id>
- plan_id: <plan_id>
- output_dir: <output_dir>

## Target
- brief: <target_brief>
- languages: <languages>
- frameworks: <frameworks>

## Progress
- current_phase: transform
- last_completed_step: init-complete
- timestamp: <ISO timestamp>

## MVPs
| # | Name | ID | Status | Priority |
|---|------|----|--------|----------|
| 0 | <name> | <mvp_id> | pending | 0 |
| 1 | <name> | <mvp_id> | pending | 1 |
...

## Active MVP
- index: 0
- name: <first MVP name>
- mvp_id: <first MVP id>
- compile_result: untested

## Architecture Summary
<5-10 line compact summary of key architecture decisions from Step 4>

## Blockers
(none)

## Accuracy
(not yet run)

## Resume Instruction
Init complete. Run `/migrate run` to start transforming MVP 0: <name>.
Read SYMBOLS.md + <mvp-slug>/SPEC.md before generating.
```

**This file is critical**: after `/clear`, Claude reads ONLY this file to fully reconstruct its working state. Keep it under 120 lines.

### Step 8 — Begin Transforms Immediately

No prompt. Proceed directly to the `run` workflow starting from MVP 0.

---

## Workflow: `run [--mvp <name>]`

**No approval gates.** Transforms run fully automatically from start to finish.
If `--mvp` is omitted, run ALL pending MVPs in priority order (lowest priority number first).

### Step 1 — Load Context (Checkpoint-First)

**Checkpoint resume** (preferred — fast, no questions):
1. `Glob: migration-output/*/CHECKPOINT.md` — look for existing checkpoint
2. If found: Read it → extract `plan_id`, `project_id`, active MVP, resume instruction
3. `codeloom_list_mvps(plan_id)` → cross-check MVP statuses (DB is source of truth; update checkpoint if stale)
4. If `--mvp <name>` specified: find that MVP; otherwise use the active MVP from checkpoint
5. Read `<output_dir>/SYMBOLS.md` + active MVP's `SPEC.md`
6. Follow the `Resume Instruction` from checkpoint — no user questions needed

**Fallback** (no checkpoint exists):
1. `codeloom_list_projects` → find/confirm project
2. `codeloom_get_project_intel(project_id)` → get plan list → pick most recent `in_progress` plan
3. `codeloom_list_mvps(plan_id)` → find MVP by name (fuzzy: case-insensitive contains match), or collect all pending

Print status line before each MVP:
```
→ Starting MVP <N>: <name> (<file_count> source files, <unit_count> units)
```

### Step 2 — Mark In-Progress + Update Checkpoint

1. Call `codeloom_start_transform(plan_id, mvp_id)` immediately.
   This makes the MVP show as "in_progress" in the CodeLoom UI while generation runs.

2. **Update `CHECKPOINT.md`**: set active MVP to current, status `in_progress`, update `last_completed_step` to `mvp-<N>-transform-started`, update `Resume Instruction` to: "Transform of MVP N (<name>) is in progress. Re-read SPEC.md and SYMBOLS.md, then continue generating files."

### Step 3 — Pull Source Units + Build Import Context

1. For each `unit_id` in MVP: `codeloom_get_source_unit(unit_id)` → full source + file_path + signature
2. Group by file_path to understand the file structure
3. **Read `<output_dir>/SYMBOLS.md`** — load the full renamed-exports map from all prior completed MVPs
4. Read this MVP's `SPEC.md` for transform guidance (key transforms, renamed exports table)

**Import audit** — for each source file in this MVP, identify every import statement.
Cross-check each import target against:
- Files within this MVP's `source_file_paths` → ✅ OK
- Files in Foundation (MVP 0) → ✅ OK
- Files from any prior completed MVP → ✅ OK — look up new symbol name in `SYMBOLS.md`, rewrite
- External packages (`node_modules` / pip / maven) → ✅ OK, keep as-is
- Files not covered by any of the above → ⚠️ **Gap**

For each gap:
- If the missing file is small (< 50 lines) and has no further dependencies: **migrate it inline** into this MVP
- Otherwise: **create a typed stub** (interfaces + function signatures, no implementation) so the MVP compiles
- Log prominently: "Gap: `<source_file>` imports `<missing_file>` which was not assigned to any MVP. Migrated inline / stubbed."

### Step 4 — Batch Transform (Fully Automatic)

Generate all output files for the MVP:

1. **Generate all output files** in-context using:
   - Full source from Step 3
   - `SYMBOLS.md` for resolving all import renames from prior MVPs
   - Architecture doc (from session or re-query `codeloom_get_project_intel`)
   - Functional spec from `SPEC.md`
   - Target stack conventions
   - **Batch processing rules**: source programs often process 10K–100K+ records. Migrated code must use streaming/chunked I/O (generators, line-by-line readers, iterators) — never load entire datasets into memory. Indexed file access (VSAM) → SQLite or DB-backed storage. Preserve open→read→process→write→close sequential semantics.
   - **External API docs via chub**: before writing any file that calls an external service, run `chub get <provider>/<api> --lang <lang>` to fetch current docs. Use returned spec as the authoritative reference for request shapes, auth headers, and error codes. Run `chub search <term>` first if unsure of the exact ID.
   - Lane-specific transforms if applicable (@~/.claude/commands/migrate/lane-struts.md, etc.)

2. **When rewriting imports**: use `SYMBOLS.md` exhaustively. Never leave a `graph.invoke(state, config)` call — always replace with the migrated orchestrator function. Never import from a source path that no longer exists.

3. **Write all files to disk** using the Write tool:
   - Path: `<output_dir>/src/<target-path>` — all MVPs write into a single unified `src/` tree
   - `mkdir -p` for parent directories as needed

4. **Show summary table** after writing:
   ```
   ✅ MVP <N>: <name>
   | File                              | Lines | Key Transforms Applied              |
   |-----------------------------------|-------|-------------------------------------|
   | src/agents/generate-post/index.ts |   89  | StateGraph → runGeneratePost()      |
   | src/agents/utils.ts               |  140  | LangGraphRunnableConfig → plain args|
   ```

### Step 4.5 — Compile Check (Hard Gate)

Determine compile command from target stack:
- **TypeScript**: `cd <output_dir> && npx tsc --noEmit 2>&1`
  - If `tsconfig.json` is missing: create a minimal one targeting `src/`
- **Python**: `cd <output_dir> && python -m py_compile $(find src -name "*.py" | tr '\n' ' ') 2>&1`
- **Java/Maven**: `cd <output_dir> && mvn compile -q 2>&1`
- **.NET**: `cd <output_dir> && dotnet build -q 2>&1`

**Run via Bash tool. Treat exit code 0 as pass, anything else as fail.**

**If compile passes**: continue to Step 5.

**If compile fails**:
1. Parse errors — identify each failing file and line number
2. Fix each error in-context. Common patterns:
   - `Cannot find module 'X'` → check SYMBOLS.md for a path rename, update import
   - `Module has no exported member 'Y'` → symbol was renamed in a prior MVP, look it up in SYMBOLS.md
   - `Type error / argument mismatch` → function signature changed when migrated, align the call site
   - `Cannot find name 'Z'` → missing type import, check which MVP owns the type and import from there
3. Re-run compile check after fixes (up to **3 rounds total**)
4. If still failing after round 3: **stop the chain**. Do not mark MVP complete. Report:
   ```
   ❌ MVP <N>: <name> — compile failed after 3 fix rounds.
   Remaining errors:
     src/agents/foo.ts:42 — Cannot find module '../bar/bar-graph.js'
     ...
   Fix manually or assign missing files, then resume with:
     /migrate run --mvp "<name>"
   ```
   Call `codeloom_complete_transform` with `status=failed` to log the state in CodeLoom.
   **Update `CHECKPOINT.md`**: set MVP status to `needs_review`, populate `Blockers` with error output (truncated to 20 lines), set `compile_result: failed`, update `Resume Instruction` to: "MVP N (<name>) failed compile after 3 rounds. Fix errors below, then `/migrate run --mvp '<name>'` to retry."
   Do not proceed to the next MVP.

### Step 5 — Validate, Update Symbols, Save to CodeLoom

1. `codeloom_validate_output(project_id, "transform", <concatenated file summaries>)` — surface warnings
   - Print warnings prominently but **do not pause** — log and continue

2. **Update `SYMBOLS.md`**: replace the `_(populated when transform completes)_` placeholder for this MVP
   with the actual renamed exports from the MVP's `SPEC.md` Renamed exports table.
   This makes them immediately available to all subsequent MVPs.

3. `codeloom_complete_transform(plan_id, mvp_id, transform_summary, output_files)`:
   - `transform_summary`: summary table from Step 4 + gap notes + compile result
   - `output_files`: list of `{file_path, language}` — **no code content**

4. **Update `CHECKPOINT.md`**: set this MVP to `migrated`, set `compile_result: pass`, advance `Active MVP` to next pending MVP, set `last_completed_step` to `mvp-<N>-transform-complete`, clear `Blockers`, update `Resume Instruction` to: "MVP N complete. Next: run MVP N+1 (<name>), or `/clear` safely — will resume from this checkpoint."

5. **Print safe-to-clear message**:
   ```
   ✅ MVP <N>: <name> complete (compile passed). CHECKPOINT.md updated.
   Safe to /clear if context is getting full — I'll resume automatically from CHECKPOINT.md.
   ```

### Step 6 — Chain to Next MVP

If more pending MVPs remain: loop back to Step 1 for the next one (by priority order).
Print progress after each:
```
✅ MVP <N> complete (compile passed) → starting MVP <N+1>: <name>
```

When all done:
```
✅ All <N> MVPs complete. View in CodeLoom at /migration.
Run /migrate status for a summary.
```

### Step 7 — Compare + Auto-Fix (always, after full run)

Fires automatically when all pending MVPs have completed.
Skipped if `--mvp` targeted a single MVP (partial run — full picture not ready).

1. Print: `→ Running accuracy comparison…`
2. Execute the `compare` workflow Steps 1–6 above
3. Print: `→ Compare + fix complete. Score: XX → YY/100. Report: <output_dir>/MIGRATION_ACCURACY.md`

No approval gate — fully automatic.

---

## Workflow: `compare [--mvp <name>]`

**No approval gates.** Runs fully automatically — same policy as `run`.

### Step 1 — Load Context

If no active plan in session:
1. `codeloom_list_projects` → find/confirm project
2. `codeloom_get_project_intel(project_id)` → get plan list → pick most recent plan
3. `codeloom_list_mvps(plan_id)` → load MVP list and `output_dir`

If `--mvp <name>` specified: scope to that MVP only. Otherwise compare all completed MVPs.

Print: `→ Running accuracy comparison for <project_name> (N MVPs)`

### Step 2 — Build Comparison Map

1. Read `<output_dir>/SYMBOLS.md` — maps source symbols → target Python symbols + file paths
2. For each MVP: get `source_file_paths` from `codeloom_list_mvps`
3. For each source file: find corresponding target file(s):
   - Primary: SYMBOLS.md symbol-level mapping
   - Fallback: naming convention (e.g., `ORDERMGMT.cbl` → `src/orders/manager.py`)
4. Build pairs: `[(source_file_path, [target_file_path, ...])]`

### Step 3 — Per-File Comparison (Claude performs in-context)

For each `(source_file, target_files)` pair:

1. `codeloom_get_source_unit(unit_id)` — get original source text for every unit in this file
2. Read each target file from `<output_dir>/src/` via the Read tool
3. For each source construct (paragraph, procedure, function), check the target:

```
CHECK A — Presence:   target function covers this source construct?
CHECK B — Branching:  all IF/EVALUATE/ON branches represented?
CHECK C — Boundaries: comparison operators correct? (>= vs >, <= vs <, etc.)
CHECK D — Calls:      CALL/PERFORM → correct target function calls?
CHECK E — Data:       field/variable references → correct attribute/variable names?
CHECK F — Returns:    RETURN-CODE / output fields → exceptions or return values?
```

4. Classify each construct:
   - `✅ Correct` — logic match confirmed
   - `⚠️ Gap` — source construct has no equivalent in target
   - `❌ Bug` — target has wrong logic (wrong operator, wrong call, missing branch)
   - `📝 Deviation` — intentional architectural change (acceptable, document only)

### Step 4 — Score and Write Report

**Scoring weights**: main paragraphs ×3, subprogram entry points ×2, utility paragraphs ×1

**Score** = (correct + deviation) / total_weighted_constructs × 100

**Write `<output_dir>/MIGRATION_ACCURACY.md`**:
```markdown
# Migration Accuracy Report
Generated: <timestamp>
Overall Score: XX/100

## Summary
| MVP           | Programs | Constructs | Correct | Gaps | Bugs | Score |
|---------------|----------|------------|---------|------|------|-------|
| MVP 0         | N        | N          | N       | N    | N    | N%    |
...

## Per-Program Analysis
### ORDERMGMT.cbl → src/orders/manager.py
| Paragraph       | Status     | Note                                       |
|-----------------|------------|--------------------------------------------|
| 0000-MAIN-PARA  | ✅ Correct  |                                            |
| 1000-READ-TRANS | ❌ Bug      | Uses `>` should be `>=` (line 45)          |
...

## Top Actionable Fixes
1. src/orders/manager.py:45 — change `>` to `>=` in ship quantity check
   Before: `if qty_required > product.qoh:`
   After:  `if qty_required >= product.qoh:`
...

## Intentional Deviations
| Source Pattern        | Target Pattern           | Reason                       |
|-----------------------|--------------------------|------------------------------|
| VSAM KSDS direct I/O | SQLite via ProductCatalog| Python-idiomatic persistence |
```

Print summary table to console.

### Step 5 — Auto-Fix (runs immediately after Step 4)

For each `❌ Bug` or `⚠️ Gap` identified:

1. **Classify complexity**:
   - **Surgical** (auto-apply): ≤ 3 lines changed, deterministic replacement (wrong operator, wrong function name, missing `raise`, off-by-one boundary)
   - **Structural** (manual): new function needed, multi-paragraph rewrite, missing file

2. **Apply surgical fixes**:
   - Read the target file (Read tool)
   - Apply precise Edit (enough surrounding context to be unique)
   - Run compile check: `python -m py_compile <file>` (or stack equivalent)
   - Pass → mark issue `✅ FIXED` in report
   - Fail → revert, mark `⚠️ FIX ATTEMPTED — compile failed`

3. **Report structural fixes** with clear instructions, no auto-apply:
   ```
   📋 Manual: src/orders/manager.py — missing ORDERNEW-style log output
      Fix: call process_orders_with_log() when log_file is set
   ```

4. **Rewrite `<output_dir>/MIGRATION_ACCURACY.md`** with FIXED markers and revised score.

### Step 6 — Persist to CodeLoom

Call `codeloom_save_accuracy_report`:
- `plan_id` — current plan UUID
- `overall_score` — pre-fix score
- `fixed_score` — post-fix score
- `fixes_applied` — count of surgical fixes
- `fixes_pending` — count of manual items
- `report_markdown` — full MIGRATION_ACCURACY.md content
- `per_mvp` — `[{mvp_name, score, fixed_score, programs, constructs, correct, gaps, bugs}]`

Print: `→ Score: XX → YY/100 | Fixed: N | Manual: M | Saved to CodeLoom`

7. **Update `CHECKPOINT.md`**: populate `Accuracy` section with scores, set `last_completed_step` to `compare-complete`, update `Resume Instruction` to: "Migration complete. Review MIGRATION_ACCURACY.md for remaining manual fixes."

---

## Workflow: `status`

1. If no active project: `codeloom_list_projects` → ask user to select.
2. `codeloom_get_project_intel(project_id)` → get plan list.
3. `codeloom_list_mvps(plan_id)` → phase status per MVP.
4. Display:

```
Project: <name> | Plan: <plan_id> | Output: <output_dir>

| #  | MVP Name               | Status      | Transform | Test  |
|----|------------------------|-------------|-----------|-------|
|  0 | Foundation & Setup     | migrated    | ✅        | ⏳    |
|  1 | Core Services          | in_progress | 🔄        | ⏳    |
|  2 | Integrations           | pending     | ⏳        | ⏳    |

Next: /migrate run --mvp "Integrations"
```

If `accuracy_score` is available on the plan (from CodeLoom), also show:
```
Accuracy: XX/100 → YY/100 after fixes  (N fixed, M manual — see MIGRATION_ACCURACY.md)
```

---

## Workflow: `resume`

**Checkpoint resume** (preferred — zero questions):

1. `Glob: migration-output/*/CHECKPOINT.md` — find existing checkpoints.
2. If found (one): Read it → extract `plan_id`, `project_id`, active MVP index/name, `Resume Instruction`.
3. If found (multiple): list them, `AskUserQuestion` — pick which migration to resume.
4. Validate against DB: `codeloom_list_mvps(plan_id)` → cross-check MVP statuses (DB is source of truth; update checkpoint if stale).
5. Read `<output_dir>/SYMBOLS.md` + active MVP's `SPEC.md`.
6. Follow the `Resume Instruction` from checkpoint — proceed directly to `run` workflow. No user questions needed.

**Fallback** (no checkpoint exists):

1. `codeloom_list_projects` → display project list, ask user to select.
2. `codeloom_get_project_intel(project_id)` → show existing plans with status.
3. If >1 plan: `AskUserQuestion` — pick plan by number.
4. `codeloom_list_mvps(plan_id)` → show MVP table.
5. Proceed directly to `run` workflow for all pending MVPs (no prompt).

---

## Quality Standards

- **Shared infrastructure first**: files with import fan-in ≥ 3 go in Foundation MVP — enforced by import graph analysis, not guesswork
- **Every source file is accounted for**: uncovered file audit runs before plan saves; every file has an explicit assignment or confirmed skip
- **SYMBOLS.md is the source of truth** for cross-MVP renames — always read before generating, always update after completing
- **Compile gate per MVP**: an MVP is not complete until the target stack compiles cleanly; errors are fixed in-place, not deferred
- **Unified output tree**: all MVPs write into a single `src/` — no per-MVP subdirectories that leave callers with broken relative paths
- **Requirement-understanding transforms**: read each source file fully, understand its role in the system, apply architectural transforms — not find-and-replace. Migrated code must reflect the target architecture.
- **chub before external APIs**: any file calling an external service (auth, payments, AI, messaging, etc.) must be preceded by `chub get <provider>/<api> --lang <lang>` — never guess at request shapes or auth headers
- **Preserve parity**: migrated code must maintain functional equivalence with source
- **Batch processing parity**: source programs (especially COBOL, PL/I, mainframe) process large volumes (10K–100K+ records). Migrated code MUST preserve the same processing model:
  - Use streaming/chunked/record-at-a-time I/O — never load entire files into memory
  - Python: generators, `csv.reader` line-by-line, context managers, iterators
  - For indexed files (VSAM KSDS/ESDS): use SQLite or DB-backed indexed storage, not in-memory dicts
  - Batch logic (sort, merge, match) must handle production-scale volumes without memory explosion
  - Preserve sequential file semantics: open → read record → process → write → close
- **Code stays on disk only**: never pass code content to `codeloom_complete_transform`
- **CodeLoom visibility**: call `codeloom_start_transform` before writing any files

## Approval Gates (Summary)

| Phase | Gate |
|-------|------|
| Init Step 6: MVP list + specs + uncovered file decisions | ✅ `AskUserQuestion` — only gate |
| Run: transforms, import audit, compile, fix rounds | 🚫 None — fully automatic |
| Run: MVP fails compile after 3 rounds | ⚠️ Chain stops, reports errors — resume with `--mvp` |
| Run Step 7: compare + auto-fix (after full run) | 🚫 None — fully automatic         |
| compare: standalone compare + auto-fix           | 🚫 None — compile-verified per fix |

## Lane References

When the source codebase uses a known migration lane, apply its transform rules and quality gates:
- Mainframe (COBOL/PL1/JCL): @~/.claude/commands/migrate/lane-mainframe.md
- Struts → Spring Boot: @~/.claude/commands/migrate/lane-struts.md
- Stored Procedures → ORM: @~/.claude/commands/migrate/lane-storedproc.md
- VB.NET → .NET Core: @~/.claude/commands/migrate/lane-vbnet.md

## Lane Auto-Selection

When `codeloom_get_project_intel` returns `detected_lanes`, auto-load matching sub-skills:

| Lane ID / Source Language | Sub-Skill | Auto-Load When |
|--------------------------|-----------|----------------|
| `mainframe_to_modern` OR source in {cobol, pli, jcl} | lane-mainframe.md | COBOL/PL1/JCL files detected |
| `struts_to_springboot` | lane-struts.md | Struts framework detected |
| `storedproc_to_orm` OR source = sql (stored_procedure sub-type) | lane-storedproc.md | SQL stored procedures detected |
| `vbnet_to_dotnetcore` OR source = vbnet | lane-vbnet.md | VB.NET files detected |

**Selection algorithm** (Step 3 of init, Step 1 of run):
1. Read `detected_lanes` from `codeloom_get_project_intel`
2. Also check `file_breakdown` — if COBOL/PL1/JCL present, always load mainframe lane
3. For each detected lane with confidence > 0.3, load the matching sub-skill
4. Multiple lanes can be active simultaneously (e.g., mainframe + stored procs for COBOL+DB2)
5. Log: "Lane sub-skills loaded: mainframe (COBOL/JCL), storedproc (DB2 SQL)"

## Error Handling

- `codeloom_list_projects` fails → MCP server not running. Instruct: `./dev.sh setup-mcp`, restart Claude Code.
- `codeloom_get_import_graph` returns empty `shared_files` → small codebase with no shared infrastructure, or ASG import edges not built (check ingestion).
- `codeloom_list_units` returns 0 → project has no units. Check ingestion via `codeloom_get_project_intel`.
- `codeloom_save_plan` error "No users found" → DB has no users. Run `./dev.sh local` to ensure backend is seeded.
- `codeloom_save_mvps` returns MVP with `file_count: 0` → file path didn't match. Check exact paths from `codeloom_list_units`.
- Compile fails with "Cannot find module" for a file not in any MVP → re-run uncovered file audit, assign missing file to Foundation MVP, re-run this MVP.
- `codeloom_start_transform` error → phase 3 not found. Check MVP was saved correctly via `codeloom_list_mvps`.
- Write tool fails → check output_dir exists and is writable. Create parent dirs with `mkdir -p`.
