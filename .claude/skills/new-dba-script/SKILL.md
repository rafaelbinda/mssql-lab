---
name: new-dba-script
description: Scaffold a new DBA utility script (INST-Q####, CONN-Q####, TRAN-Q####, etc.) in dba-scripts/ with the correct rich header, parameter validation, section_name column, and defensive patterns. Use when creating a reusable instance-level monitoring or investigation query.
disable-model-invocation: true
---

When the user invokes `/new-dba-script` or asks to create a new DBA utility script, follow these steps.

## Step 1 — Determine the category and next number

From `$ARGUMENTS` or by asking the user, identify the script category:

| Category | Subfolder |
|----------|-----------|
| `INST` | `dba-scripts/SQL-instance-information/` |
| `CONN` | `dba-scripts/SQL-connections/` |
| `TRAN` | `dba-scripts/SQL-transactions-and-concurrency/` |
| `FUNC`, `PROC`, `VIEW`, `TRIG` | `dba-scripts/SQL-programming-objects/` |

Check the highest-numbered `{CATEGORY}-Q####` file in that subfolder. Next number = highest + 1, zero-padded to 4 digits.

## Step 2 — Gather inputs

From `$ARGUMENTS` or by asking the user:
- Script title (human-readable, becomes part of `Task :` and filename)
- Description
- Related notes files (A#### paths, relative to repo root)
- Related example scripts (Q#### paths, relative to repo root) — what scripts demonstrate the concepts queried here
- Related DBA scripts (other `{CATEGORY}-Q####` in the same series)
- Main parameters the script will accept (e.g., `@DATABASE`, `@SCHEMA`, `@OBJECT`)

## Step 3 — Create the file

File path: `dba-scripts/{subfolder}/{CATEGORY}-Q####-lowercase-hyphen-description.sql`

Use this exact structure:

```sql
/*
===============================================================================
Author      : Rafael Binda
Created     : {today yyyy-mm-dd}
Version     : 1.0
Task        : {CATEGORY}-Q#### - {Title}
Object      : Script
Description : {Description aligned at column 16.}
              {Continuation if needed.}
Notes       : {relative/path/to/notes/A####-topic.md}
Examples    : {relative/path/to/scripts/Q####-example.sql}
Related     : {CATEGORY}-Q#### - {Related Script Title}
===============================================================================

INDEX
1  - {First section title}
2  - {Second section title}

===============================================================================
*/

USE master;
GO

-------------------------------------------------------------------------------
-- 1 - {Parameter declaration and validation}
-------------------------------------------------------------------------------

DECLARE @DATABASE  SYSNAME  = N'';
DECLARE @SCHEMA    SYSNAME  = N'dbo';
DECLARE @OBJECT    SYSNAME  = N'';

-- Parameter validation
IF @DATABASE IS NULL OR LTRIM(RTRIM(@DATABASE)) = N''
BEGIN
    RAISERROR('Parameter @DATABASE cannot be NULL or empty.', 16, 1);
    RETURN;
END
GO

-------------------------------------------------------------------------------
-- 2 - {Investigation or output section}
-------------------------------------------------------------------------------

SELECT
    '2 - {Section title}'  AS section_name,
    {column}               AS {alias}
FROM
    {source_table} AS {alias}
WHERE
    {condition};
GO
```

## Defensive patterns — always apply to INST-Q and investigation scripts

- **`RAISERROR` on invalid parameters** — never silently continue with bad input
- **Non-destructive by default** — when a script could run DBCC or restore commands, generate them as strings and print; use `@EXECUTE_{OPERATION} BIT = 0` flag to optionally execute
- **`section_name` column** — every `SELECT` must have `'N - {Section title}' AS section_name` as its first column
- **Status strings in ALLCAPS** — `'ACTIVE INVESTIGATION'`, `'NO RESULTS FOUND'`, `'EXECUTE MANUALLY'`
- **Parameterized dynamic SQL** — `sys.sp_executesql` with `@{VAR}_IN` (input) and `@{VAR}_OUT` (output) parameter names; `QUOTENAME()` for all identifier concatenation
- **Temp tables** — named `#{CATEGORY}Q####_{Name}` (e.g., `#INSTQ0023_Results`); always `DROP TABLE IF EXISTS` before `CREATE TABLE`

## Code rules

- 4-space indentation, no tabs
- UPPERCASE keywords; `lowercase_underscore` column aliases; UPPERCASE variable names
- Meaningful table aliases (`AS sp` for `suspect_pages`, `AS mf` for `master_files`)
- `DROP TABLE IF EXISTS` (not `IF OBJECT_ID(...)`)
- `GO` as batch separator after DDL and after major sections
- `CASE` expressions for human-readable status columns
- `CAST(x AS VARCHAR(20))` for numeric-to-string; `CONCAT()` for formatted output
