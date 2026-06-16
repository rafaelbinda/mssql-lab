---
name: sql-docs
description: Query official Microsoft Learn documentation for SQL Server topics. Returns verified content from docs.microsoft.com/learn to enrich notes, validate configurations, or find official references. Use when the user asks to look up, verify, or add official references to any SQL Server topic.
disable-model-invocation: true
---

When the user invokes `/sql-docs` or asks to consult official Microsoft documentation for a SQL Server topic, follow these steps.

## Step 1 — Determine the search query

From `$ARGUMENTS` or the current context (open file, selected text, recent conversation), identify:
- The SQL Server topic to search (e.g., "MaxDOP configuration", "collation Latin1_General_100_CI_AI_SC_UTF8", "Instant File Initialization")
- The SQL Server version if relevant (default to SQL Server 2022 / `view=sql-server-ver16`)

If no topic is provided via `$ARGUMENTS`, ask the user before proceeding.

## Step 2 — Search Microsoft Learn

Call `microsoft_docs_search` with the topic as query. Review the returned excerpts and titles.

- If results are sufficient (title + excerpt clearly answers the question), proceed to Step 4.
- If a specific page looks like the authoritative reference, call `microsoft_docs_fetch` with that URL to get the full content.
- If the user needs code examples (T-SQL examples, configuration scripts), also call `microsoft_code_sample_search`.

## Step 3 — Extract what matters

From the official content, identify:
- The authoritative recommendation or definition
- Any specific version constraints or prerequisites
- Official example values, formulas, or commands
- The canonical URL for the `## Conteúdo adicional` section in notes

## Step 4 — Present findings

Respond with:

1. **Summary in Portuguese** — 2–4 sentences summarizing the official recommendation, matching the style of the project notes (concise, technical, using `→` for emphasis)
2. **Official URL** — the direct link to add under `## Conteúdo adicional`
3. **Key details** — bullet list of facts worth adding to the note, formatted ready to paste

If the topic matches an existing note in the project (`A####-*.md`), identify it and suggest where to insert the content.

## Step 5 — Optional: update the note

If the user asks to update a note with the found content:
- Read the target `.md` file first
- Insert the URL under `## Conteúdo adicional` if not already present
- Add or correct technical content in the relevant section
- Follow the notes format: Portuguese prose, `→` emphasis, numbered sections, no trailing periods on list items

## Examples

```
/sql-docs MaxDOP recommendations SQL Server 2022
/sql-docs Instant File Initialization security implications
/sql-docs FILESTREAM enable T-SQL access
/sql-docs Latin1_General_100_CI_AI_SC_UTF8 collation
/sql-docs TempDB number of files best practices
```
