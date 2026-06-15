---
name: new-module-script
description: Scaffold a new numbered lab script (Q####) in module-01-sql-server-on-premises with the correct header, section structure, ExamplesDB_ database setup, and temp table naming. Use when creating a new hands-on lab exercise script.
disable-model-invocation: true
---

When the user invokes `/new-module-script` or asks to create a new module lab script, follow these steps.

## Step 1 — Determine the next Q number

Check all `module-01-sql-server-on-premises/` scripts directories and find the highest-numbered `Q####` file. The next number is that value + 1, zero-padded to 4 digits. Cross-reference with CLAUDE.md to confirm. Do not reuse numbers.

## Step 2 — Gather inputs

From `$ARGUMENTS` or by asking the user:
- Target module subfolder (e.g., `04-database-recovery/scripts/`)
- Script title (human-readable, becomes part of `Task :` and filename)
- Short description for the `Description :` field
- Associated notes file (e.g., `A0036-topic.md`) — if it doesn't exist yet, note that it should be created
- Lab database topic name (the `ExamplesDB_{Topic}` suffix)
- Section list (what numbered sections will the INDEX contain)

## Step 3 — Create the file

File path: `module-01-sql-server-on-premises/{module}/scripts/Q####-lowercase-hyphen-description.sql`

Use this exact structure:

```sql
/*
===============================================================================
Author      : Rafael Binda
Created     : {today yyyy-mm-dd}
Version     : 1.0
Task        : Q#### - {Title}
Object      : Script
Description : {Description, aligned at column 16.}
              {Continuation if needed.}
Notes       : {module-path}/notes/A####-topic.md
===============================================================================
INDEX
1  - {First section title}
2  - {Second section title}
===============================================================================
*/

USE master;
GO

-------------------------------------------------------------------------------
-- 1 - {First section title}
-------------------------------------------------------------------------------

-- Drop and recreate lab database
IF DB_ID('ExamplesDB_{Topic}') IS NOT NULL
BEGIN
    ALTER DATABASE ExamplesDB_{Topic} SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
    DROP DATABASE ExamplesDB_{Topic};
END
GO

CREATE DATABASE ExamplesDB_{Topic};
GO

USE ExamplesDB_{Topic};
GO

/* Result:

*/

-------------------------------------------------------------------------------
-- 2 - {Second section title}
-------------------------------------------------------------------------------

/* Result:

*/
```

## Code rules to follow

- 4-space indentation, no tabs
- UPPERCASE keywords (`SELECT`, `FROM`, `WHERE`, `CREATE TABLE`, `DROP TABLE IF EXISTS`, `INSERT INTO`, `GO`, etc.)
- Column aliases: `lowercase_underscore`
- Variable names: UPPERCASE (`@OBJECT`, `@SQL`, `@ROW_COUNT`)
- Temp tables: `#Q####_{Name}` — always `DROP TABLE IF EXISTS` before creating
- `sys.sp_executesql` for parameterized dynamic SQL; `QUOTENAME()` for identifiers
- `GO` as batch separator after DDL
- Include `/* Result: ... */` comment after each section showing the expected output shape
- Section separators: 79-dash line (`-------------------------------------------------------------------------------`) before and after `-- N - Section title`
