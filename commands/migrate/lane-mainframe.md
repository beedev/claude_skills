# Lane Knowledge: Mainframe (COBOL/PL1/JCL) → Modern Target

Framework-specific migration guidance for IBM mainframe codebases
(COBOL, PL/1, JCL, VSAM, IMS DL/I, CICS) to Python/Java/.NET.

## Core Transform Table

| Mainframe Concept | Python Equivalent | Java Equivalent | Notes |
|-------------------|-------------------|-----------------|-------|
| COBOL PROGRAM-ID | Python module / class | Java class | Entry point becomes main() or class |
| WORKING-STORAGE SECTION | Class instance variables | Class fields | Preserve state semantics |
| LINKAGE SECTION | Function parameters | Method parameters | USING → params, RETURNING → return |
| COPY <copybook> | from models import <Type> | import <package>.<Type> | Shared record layout → dataclass/POJO |
| PIC X(n) | str (fixed-width) | String | Pad/truncate to n chars |
| PIC 9(n) | int | int | |
| PIC S9(n)V9(m) COMP-3 | Decimal | BigDecimal | CRITICAL: Never use float |
| PERFORM <para> | function call | method call | |
| PERFORM <para> THRU <para> | function call (merge range) | method call | Merge paragraphs into single function |
| PERFORM VARYING | for loop | for loop | COBOL 1-based → 0-based indexing |
| GO TO | (restructure to if/else) | (restructure) | Eliminate — never preserve GOTOs |
| EVALUATE / WHEN | if/elif/else or match | switch/case | |
| STOP RUN | raise SystemExit(0) | System.exit(0) | Terminates entire run unit |
| GOBACK | return | return | Returns to caller |
| CALL <subprogram> USING | function_call(args) | method.invoke(args) | Static linking → import; dynamic → registry |
| FILE SECTION (sequential) | open() / SequentialReader | BufferedReader | |
| FILE SECTION (VSAM KSDS) | SQLite / dict / VsamAdapter | TreeMap / JDBC | Key-sequenced → key-value store |
| REWRITE record | seek + write (sequential) | RandomAccessFile | In-place update |
| AT END | StopIteration / EOF check | hasNext() == false | |
| INVALID KEY | KeyError / except | catch NoSuchElementException | |
| RETURN-CODE | sys.exit(code) / return code | System.exit(code) | Special register → exit code |
| STRING ... DELIMITED BY | ''.join() / f-string | StringBuilder | |
| UNSTRING ... INTO | split() / slicing | String.split() | |
| INSPECT TALLYING | str.count() | String.chars().filter() | |
| INSPECT REPLACING | str.replace() | String.replace() | |
| 88-level condition names | Enum or bool property | enum or boolean | Named conditions |

## JCL Batch Patterns

| JCL Pattern | Shell Equivalent | Target-Lang Fallback |
|-------------|-----------------|---------------------|
| EXEC PGM=SORT, SORT FIELDS | sort -t '' -k pos,len | Python heapq / Java Collections.sort |
| SORT with INCLUDE COND | sort \| awk 'condition' | Python filter() |
| SORT with SUM FIELDS | sort \| awk group-by-sum | Python itertools.groupby |
| SORT with OUTREC FIELDS | sort \| awk reformat | Python string slicing |
| MERGE | sort -m file1 file2 | Python heapq.merge |
| IDCAMS REPRO | cp source dest | streaming copy |
| IEBGENER (SYSIN DUMMY) | cp / cat | streaming copy |
| IEFBR14 | touch / mkdir -p | (no-op) |
| GDG (+1) | naming script G{nnnn}V00 | versioned file naming |
| EXEC PGM=<user_pgm> | #!/bin/bash; python/java -m <module> | N/A (always shell) |
| EXEC COBOLCL / IGYWCL | SKIP (no-op) | N/A |
| DD DSN=...,DISP=SHR | env var / config path | env var / config |
| DD SYSOUT=* | stdout | stdout |
| DD * (instream) | heredoc / temp file | temp file |
| PROC / cataloged proc | shell function / script | shell script |
| PARM='value' | CLI argument / env var | CLI argument |
| COND=(0,NE) / IF-THEN-ELSE | shell exit code check | exit code check |

## VSAM Patterns

| VSAM Concept | Python Equivalent | Java Equivalent |
|-------------|-------------------|-----------------|
| KSDS (Key-Sequenced) | SQLite table with PK | TreeMap or JDBC table |
| RRDS (Relative Record) | list / array indexed by slot | ArrayList |
| ESDS (Entry-Sequenced) | append-only file | append-only file |
| READ KEY IS | dict[key] / SQL SELECT | map.get(key) / SQL |
| REWRITE | dict[key] = updated / SQL UPDATE | map.put(key, updated) |
| DELETE | del dict[key] / SQL DELETE | map.remove(key) |
| START KEY IS >= | bisect / SQL WHERE key >= | NavigableMap.tailMap() |
| BROWSE (sequential) | iterate dict.values() | iterate map.values() |

## IMS DL/I Patterns

| IMS Concept | Python Equivalent | Java Equivalent |
|------------|-------------------|-----------------|
| DBD (Database Description) | SQLAlchemy model / schema | JPA Entity |
| PSB (Program Spec Block) | database session / connection | EntityManager |
| PCB (Program Comm Block) | session with status | EntityManager status |
| GU (Get Unique) | session.get(Model, key) | em.find(Entity.class, key) |
| GN (Get Next) | iterator.next() | iterator.next() |
| GNP (Get Next within Parent) | filtered child query | JOIN + WHERE parent_id |
| ISRT (Insert) | session.add(record) | em.persist(entity) |
| REPL (Replace) | session.merge(record) | em.merge(entity) |
| DLET (Delete) | session.delete(record) | em.remove(entity) |
| Segment hierarchy | Table relationships (FK) | @OneToMany / @ManyToOne |
| SSA (Segment Search Arg) | WHERE clause | JPQL WHERE |

## Decimal Arithmetic (CRITICAL)

COBOL packed decimal (COMP-3) and display numeric (PIC S9(n)V9(m)):
- **Python**: ALWAYS use `decimal.Decimal`, NEVER `float`
- **Java**: ALWAYS use `java.math.BigDecimal`, NEVER `double`
- **V99 implied decimal**: `PIC S9(3)V99` means 3 integer + 2 decimal digits
  - Source stores as integer (e.g., 12345 = 123.45)
  - Target must: `Decimal(str(raw_value)) / Decimal('100')` or parse with scale
- **ROUNDED**: `Decimal.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)`
- **ON SIZE ERROR**: try/except OverflowError or check magnitude

## Record Layout Mapping

COBOL Data Division → target type system:
```
01 EMPLOYEE-RECORD.
   05 EMP-ID        PIC X(5).      →  emp_id: str      # fixed 5 chars
   05 EMP-NAME      PIC X(20).     →  emp_name: str    # fixed 20 chars
   05 EMP-SALARY    PIC 9(5)V99.   →  emp_salary: Decimal  # 5.2 digits
   05 EMP-DEPT      PIC X(3).      →  emp_dept: str    # fixed 3 chars
```

Python: `@dataclass` with `from_line(line: str)` and `to_line() -> str` for fixed-width serialization.
Java: POJO/record with `fromLine(String)` and `toLine()` static methods.

**OCCURS**: `05 ITEM PIC X(10) OCCURS 5 TIMES` → `List[str]` (Python) / `String[]` (Java)
**REDEFINES**: `05 FIELD-B REDEFINES FIELD-A` → union type or separate parsing based on condition

## Quality Gate Checklist

Before approving the transform phase, verify:
- [ ] All COMP-3 / implied decimal fields use Decimal/BigDecimal (NEVER float/double)
- [ ] Comparison operators match source exactly (> vs >= — verify each program)
- [ ] STOP RUN → SystemExit/System.exit (not return — terminates run unit)
- [ ] GOBACK → return (returns to caller, not system exit)
- [ ] Copybook record layouts have exact field widths matching PIC declarations
- [ ] from_line()/to_line() fixed-width serialization matches COBOL record length
- [ ] VSAM KSDS key operations → key-value store with proper error handling
- [ ] AT END → proper EOF/StopIteration handling
- [ ] INVALID KEY → proper KeyError/exception handling
- [ ] PERFORM THRU ranges merged into single functions (no GO TO simulation)
- [ ] JCL compile steps (COBOLCL, IGYWCL) SKIPPED — not migrated
- [ ] JCL SORT steps use shell sort where possible, target-lang streaming for complex ops
- [ ] 88-level conditions → named constants or enum values
- [ ] RETURN-CODE special register → proper exit codes
- [ ] FILE STATUS codes → exception handling (not status code checking)

## Common Pitfalls

1. **Float for money**: COMP-3/V99 fields MUST use Decimal/BigDecimal. Float loses precision on financial math.
2. **> vs >=**: COBOL programs vary — some use `>`, others `>=` for the same business logic. Verify EACH comparison operator against source.
3. **STOP RUN vs GOBACK**: STOP RUN terminates the entire run unit (all callers). GOBACK returns to the calling program. Using `return` for STOP RUN is WRONG — the caller will continue looping.
4. **1-based vs 0-based**: COBOL arrays (OCCURS) are 1-based. SORT FIELDS positions are 1-based. Convert all to 0-based in target.
5. **Fixed-width records**: COBOL files are fixed-width (LRECL). Every field has exact width. Trailing spaces matter. Don't strip/trim unless the business logic does.
6. **REDEFINES**: Two names for the same memory. Target must parse the right interpretation based on a condition variable — not both simultaneously.
7. **PERFORM THRU with GO TO**: Legacy pattern uses GO TO within a THRU range as flow control. Must be restructured as if/elif/else — never simulate GO TO.
8. **Copybook fan-out**: One copybook used by 20 programs. The target type definition MUST be shared (single source of truth), never duplicated per program.
9. **CICS interactions**: If COBOL uses EXEC CICS, the program is online (not batch). Online programs need REST API endpoints, not batch scripts.
10. **IMS DL/I hierarchy**: Segment navigation (GN/GNP) is hierarchical traversal. Don't flatten to SQL without preserving parent-child relationships.
