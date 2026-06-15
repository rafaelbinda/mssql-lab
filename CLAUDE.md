# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

SQL Server DBA study repository — structured lab scripts and reference queries for learning SQL Server on-premises and Azure. Two parallel hierarchies exist with different purposes and patterns.

## Current Script Numbering

Always check the highest-numbered file in the target directory before assigning a new number.

| Series | Last used | Next |
|--------|-----------|------|
| Module scripts `Q####` (across all `module-01/` subfolders) | Q0030 | Q0031 |
| Module notes `A####` (across all `module-01/` subfolders) | A0035 | A0036 |
| Instance info `INST-Q####` (`dba-scripts/SQL-instance-information/`) | INST-Q0022 | INST-Q0023 |

## File Naming

- Module scripts: `Q####-lowercase-hyphen-description.sql` (4-digit, zero-padded)
- Module notes: `A####-lowercase-hyphen-description.md`
- DBA utility scripts: `{CATEGORY}-Q####-lowercase-hyphen-description.sql`
  - `INST` → `dba-scripts/SQL-instance-information/`
  - `CONN` → `dba-scripts/SQL-connections/`
  - `TRAN` → `dba-scripts/SQL-transactions-and-concurrency/`
  - `FUNC`, `PROC`, `VIEW`, `TRIG` → `dba-scripts/SQL-programming-objects/`
- Q numbers and A numbers are **globally sequential across all modules** in `module-01/`

## SQL Script Header

### Module scripts (`Q####`)

```sql
/*
===============================================================================
Author      : Rafael Binda
Created     : yyyy-mm-dd
Version     : 1.0
Task        : Q#### - Human Readable Title
Object      : Script
Description : Description text aligned at column 16.
              Continuation line also at column 16.
Notes       : relative/path/to/notes/A####-topic.md
===============================================================================
INDEX
1  - First section title
2  - Second section title
===============================================================================
*/
```

### DBA utility scripts (`INST-Q####`, `CONN-Q####`, etc.)

After `Notes :`, add:

```sql
Examples    : relative/path/to/scripts/Q####-example.sql
Related     : INST-Q#### - Related Script Title
```

**Header rules:**
- `Task` value must be `{PREFIX}#### - {Title that matches the filename}`
- Banner `=` lines are exactly 79 characters wide
- Multi-line field values align continuation text at column 16
- `Created` in `yyyy-mm-dd` format; `Version` starts at `1.0`
- `Notes`, `Examples`, `Related` use paths relative to the repo root

## T-SQL Code Style

**Keywords:** UPPERCASE — `SELECT`, `FROM`, `WHERE`, `INNER JOIN`, `LEFT JOIN`, `DECLARE`, `SET`, `CREATE TABLE`, `DROP TABLE IF EXISTS`, `INSERT INTO`, `ORDER BY`, `GO`, `USE`, `BEGIN`, `END`

**Column aliases:** `lowercase_underscore` (e.g., `database_name`, `page_address`, `event_type_desc`)

**Table aliases:** meaningful abbreviations — `AS sp` for `suspect_pages`, `AS mf` for `master_files`, `AS bs` for `backupset`

**Variables:** UPPERCASE — `@DATABASE`, `@SCHEMA`, `@OBJECT`, `@SQL`, `@OBJECT_ID`

**Indentation:** 4 spaces (no tabs)

**Section separators and headers:**

```sql
-------------------------------------------------------------------------------
-- 1 - Section title
-------------------------------------------------------------------------------
```

**Dynamic SQL:**
- Always use `sys.sp_executesql` for parameterized queries (never `EXEC(@SQL)` with parameters)
- Always use `QUOTENAME()` when concatenating identifiers
- Name input params `@{VAR}_IN`, output params `@{VAR}_OUT` in `sp_executesql` calls

**Other patterns:**
- `DROP TABLE IF EXISTS` before any temp table creation (not `IF OBJECT_ID(...)`)
- `GO` as batch separator after DDL statements
- `CASE` expressions for human-readable status/description columns
- Status string literals in ALLCAPS: `'ACTIVE INVESTIGATION'`, `'NO RESULTS FOUND'`
- `CAST(x AS VARCHAR(20))` for numeric-to-string in concatenation; `CONCAT()` for formatted output

## dba-scripts/ vs module-01/ — Key Differences

| | `dba-scripts/` | `module-01/` |
|--|----------------|--------------|
| Purpose | Reusable reference/monitoring utilities | Hands-on lab exercises |
| Section name column | Every SELECT has `'N - ...' AS section_name` as first column | Not required |
| Parameters | UPPERCASE `@DATABASE`, `@OBJECT`, etc. with `RAISERROR` validation | May hardcode values |
| Destructive ops | Never auto-execute; use `@EXECUTE_X BIT = 0` flag; generate commands as strings | DO execute (own `ExamplesDB_` sandbox) |
| Temp table naming | `#{CATEGORY}Q####_{Name}` (e.g., `#INSTQ0021_SuspectPages`) | `#Q####_{Name}` |
| Lab databases | None — run against any instance | `ExamplesDB_{Topic}`: always drop/recreate at script start using `SET SINGLE_USER WITH ROLLBACK IMMEDIATE` then `DROP DATABASE` |
| Expected output | Not included | `/* Result: ... */` comment after each section |
| Header extra fields | `Examples :`, `Related :` | `Notes :` only |

## Notes (.md) Format

```markdown
# A#### – Topic Name

> **Author:** Rafael Binda  
> **Created:** yyyy-mm-dd  
> **Version:** 1.0  

---

## Descrição

[Portuguese prose]

---

## 1 - Section title

[Content]
```

- **Written in Portuguese (Brazilian)**; English technical terms kept as-is
- Title uses em dash with spaces: `# A#### – Topic Name`
- Section headers numbered: `## N - Section title`
- Bullet points: `- item` (not `*`); no trailing period after list items
- `→` for sequential steps or emphasis
- Code blocks: triple backtick with language hint (` ```sql `, ` ```text `)
