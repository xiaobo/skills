---
name: dml-expert
description: Guides assistants through DML (Dato Measure Language) and Model Query SQL workflows: generate/update `.dml` models from CSV/XLSX, write `.sql/.dsql`, validate/execute queries, and fix compilation/runtime errors by editing `model.dml`.
license: Apache-2.0
metadata:
  author: datoai.com
  version: "1.0"
---

# DML Expert

Use this skill when working with **CSV/XLSX → `.dml` models → Model Query SQL** (`.sql` / `.dsql`) or when debugging **DML/query validation/compilation errors**.

## When to use (triggers)

- Generate or refresh `base_model.dml` / `model.dml` from data files
- Write or update Model Query SQL (`.sql` / `.dsql`)
- Fix validation/compilation errors: unknown column/measure, ambiguous refs, type mismatch, circular refs, YAML/formula parse errors

## Success criteria

- Reuse existing fields/measures first; add new definitions only when truly missing
- Validate-only before execute, and after every model edit
- Fix one error at a time, re-validate, repeat

## Setup & installation

**Before using this skill, ensure the `dmlc-mcp` MCP server is configured.**

### Check if MCP is available

Try calling `list_models` or `list_connections`. If you get an error like "Tool not found" or "MCP server not configured", follow the installation steps below.

### Installation steps (Cursor IDE)

1. **Prerequisites (recommended: bundled app-image)**
   - `dmlc` app-image built via jpackage at `target/jpackage/dmlc/` (contains `dmlc.exe` on Windows)

2. **Create/update `.cursor/mcp.json`**
   - Location: `.cursor/mcp.json` in your workspace root (project-specific) or `~/.cursor/mcp.json` (global)
   - Windows global: `%APPDATA%\Cursor\User\globalStorage\mcp.json`

3. **Add configuration (using bundled `dmlc` app)**
   ```json
   {
     "mcpServers": {
       "dmlc-mcp": {
         "command": "D:\\absolute\\path\\to\\target\\jpackage\\dmlc\\dmlc.exe",
         "args": ["--mcp"],
         "cwd": "D:\\absolute\\path\\to\\your\\workspace",
         "env": {
           "WORKSPACE_FOLDER": "D:\\absolute\\path\\to\\your\\workspace"
         }
       }
     }
   }
   ```
   **Important:**
   - Use **absolute paths** (not `${workspaceFolder}` — it may not work in Cursor)
   - Set `WORKSPACE_FOLDER` in `env` — required for file path resolution
   - For project-specific config, use `.cursor/mcp.json` in workspace root

4. **Restart Cursor** after configuration changes

5. **Verify installation**
   - Try calling `list_connections` — should return available database connections
   - If errors persist, check Cursor's MCP server logs

### Troubleshooting

- **"Tool not found"**: MCP server not configured or not restarted
- **"File paths don't resolve"**: Check `WORKSPACE_FOLDER` is set correctly in `env`
- **"Executable not found"**: Verify the path to `target/jpackage/dmlc/dmlc.exe` is correct and absolute

### Building the app-image (if needed)

```bash
cd dmlc_jdbc
mvn clean package -DskipJpackage=false
# App-image will be at: target/jpackage/dmlc/ (use dmlc.exe as the MCP command)
```

## Quick start (default workflow)

- **Generate / refresh models**
  - Run `list_models` with `dataFiles` (comma-separated paths) to generate/update `base_model.dml` + `model.dml`.
  - Read `base_model.dml` to learn datasets/columns; prefer editing `model.dml` for new logic.

- **Write the query**
  - Convert the request into Model Query SQL.
  - Reuse existing fields/measures first; only add definitions when missing.
  - Pick the correct query type:
    - **Aggregation**: query from `MODEL` and reference measures via `AGG(...)`
    - **Detail**: query from `MODEL.<dataset>` (no `AGG`, no `GROUP BY`)

- **Validate first, then execute**
  - Always run `run_query` with `validateOnly: true` before execution.
  - Fix **one** error at a time in `model.dml`, then re-validate.

## Example prompts (copy/paste)

- “Generate DML models from `data/sales.xlsx` and `data/customers.csv`, then summarize available datasets/fields.”
- “Write a Model Query SQL query for total sales by month for the last 12 months; validate-only first.”
- “Validation fails with `UNKNOWN_COLUMN: customer` — update `model.dml` with the correct column-macro and re-validate.”
- “We need a reusable ‘gross margin’ measure; define it in `model.dml` and then query it by month.”

## What goes where

- **`base_model.dml`**: generated schema (datasets/columns). Avoid putting measures/column-macros here.
- **`model.dml`**: the “working” file for:
  - `column-macros` (aliases + computed row-level fields)
  - measures (aggregations / business metrics)
- **`references/`**: read-only docs you can pull in on demand (syntax, patterns, troubleshooting).

## Error → fix (fast lookup)

| Symptom (typical) | Likely error code | Fix |
|---|---:|---|
| Unknown column `X` | `UNKNOWN_COLUMN` | Add a `column-macros` mapping in `model.dml` (use the real column name from `base_model.dml`). |
| Unknown measure `Y` | `UNKNOWN_MEASURE` | Define measure `Y` in `model.dml` (start simple: `SUM(...)`, `COUNT(DISTINCT ...)`). |
| Ambiguous reference `Z` | `AMBIGUOUS_REF` | Fully-qualify: `{DATASET_NAME}.Z` (often via `column-macros`). |
| Type mismatch | `TYPE_MISMATCH` | Fix formula types or add casts/conversions in a column-macro. |
| Circular reference | `CIRCULAR_REF` | Break the dependency chain; build derived measures from atomic ones. |
| YAML / formula parse errors | `YAML_ERROR` / `SQL_ERROR` | Fix indentation/quotes; keep formulas minimal; re-validate. |

## Guardrails

- **Do**: validate with `validateOnly: true` before execution and after every model edit.
- **Do**: add measure descriptions for business meaning.
- **Don’t**: edit `base_model.dml` for measures/column-macros.
- **Don’t**: “shotgun fix” multiple issues before re-validating.

## Deep dives (read only when needed)

- **References**
  - **Syntax**
    - [Column macros](references/syntax/column-macros.md)
    - [Measures](references/syntax/measures.md)
    - [DML functions & modifiers](references/syntax/dml-functions.md)
    - [Model Query SQL spec](references/syntax/model%20query%20sql-specification.md)

  - **Examples**
    - [Common query patterns](references/examples/common-patterns.md)
    - [Business logic rules](references/examples/business-logic-rules.md)

  - **Troubleshooting**
    - [Compilation errors](references/troubleshooting/compilation-errors.md)