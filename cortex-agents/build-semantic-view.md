# Skill: Build a Semantic View with dbt

> **Run this skill inside a Snowsight Workspace.** Open a Workspace in Snowsight, start a Cortex Code chat, and paste this file's contents directly into the prompt.

## Purpose

Walk the user through building a Snowflake Semantic View using the `dbt_semantic_view` package. The skill handles permissions, project scaffolding, table introspection, semantic view generation, deployment, and iterative refinement — all from within Snowsight / Cortex Code.

The user does NOT need a local dbt installation or a cloned repo. Everything runs in Snowflake.

---

## Workflow Overview

```
Phase 1: PERMISSIONS CHECK
  └── Verify role can create EAI, stages, semantic views, and run dbt

Phase 2: GATHER CONTEXT
  ├── Ask: source table(s)
  ├── Ask: target database.schema for semantic view
  ├── Ask: warehouse
  └── Ask: verified business questions (top 20) + sample SQL queries

Phase 3: INTROSPECT
  ├── DESCRIBE TABLE → column names, types
  ├── SELECT * LIMIT 10 → sample values
  └── Propose draft: classify columns as dimensions, facts, time dimensions

Phase 4: BUILD & DEPLOY VIA DBT
  ├── Load dbt skills (MANDATORY)
  ├── Scaffold project following dbt conventions + add dbt_semantic_view package
  ├── Generate semantic view model from Phase 3 classification
  └── dbt deps + dbt run to deploy

Phase 5: REFINE
  ├── Ask: custom instructions (business rules for Cortex Analyst)
  ├── Ask: synonyms (business terms → column mappings)
  ├── Ask: sample values to highlight
  ├── Ask: descriptions for key columns
  └── Redeploy with refinements

Phase 6: VALIDATE
  ├── SHOW SEMANTIC VIEWS IN SCHEMA
  ├── DESCRIBE SEMANTIC VIEW
  └── Test question in Cortex Analyst chat
```

---

## Phase 1: Permissions Check

Before anything else, verify the user's role has the grants needed. Run these checks and report any gaps:

```sql
-- Check current context
SELECT CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_DATABASE(), CURRENT_SCHEMA();

-- Check if the role can create the objects we need
-- The user needs:
--   CREATE SEMANTIC VIEW on the target schema
--   CREATE STAGE on a working schema (for the dbt project)
--   USAGE on a warehouse
--   SELECT on the source table(s)
```

If the user's role is missing grants, generate the exact GRANT statements needed and ask them to have an admin run them.

### External Access Integration (EAI) for dbt packages

The `dbt_semantic_view` package is hosted on hub.getdbt.com. To run `dbt deps` inside Snowflake (via notebook or Git integration), the environment needs network access to download the package.

Check if an EAI already exists that allows access to hub.getdbt.com:

```sql
SHOW EXTERNAL ACCESS INTEGRATIONS;
```

If none exist that cover `hub.getdbt.com` and `github.com`, create one:

```sql
CREATE OR REPLACE NETWORK RULE dbt_hub_network_rule
  MODE = EGRESS
  TYPE = HOST_PORT
  VALUE_LIST = ('hub.getdbt.com', 'github.com', 'codeload.github.com');

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION dbt_hub_access
  ALLOWED_NETWORK_RULES = (dbt_hub_network_rule)
  ENABLED = TRUE;
```

> **Note:** Creating EAIs requires ACCOUNTADMIN or a role with CREATE INTEGRATION privilege. If the user doesn't have this, flag it and provide the SQL for their admin.

---

## Phase 2: Gather Context

Ask the user these questions **before** doing any introspection. Collect all answers first, then proceed.

### Question 1: Source Table(s)

> **What table(s) should the semantic view be built on?**
>
> Provide the fully qualified name(s): `DATABASE.SCHEMA.TABLE_NAME`
>
> If multiple tables, list them all — we'll define relationships between them.

Store the answer as `SOURCE_TABLES`.

### Question 2: Target Location

> **Where should the semantic view be created?**
>
> Provide the target: `DATABASE.SCHEMA`
>
> The semantic view and dbt project artifacts will be created here.

Store as `TARGET_DB_SCHEMA`.

### Question 3: Warehouse

> **Which warehouse should be used for dbt runs and Cortex Analyst queries?**

Store as `WAREHOUSE`.

### Question 4: Verified Business Questions

> **What are the top business questions this semantic view should answer?**
>
> List 10-20 questions your team asks most often. These become:
> 1. The design input for which columns matter most
> 2. Verified queries (VQRs) that teach Cortex Analyst exact SQL patterns
> 3. Your evaluation golden test set later
>
> Example format:
> - What is the total gross revenue by source for Q1 2025?
> - Which works earned the most net publisher share in 2024?
> - Show me Spotify revenue by territory for January 2025
>
> If you have existing SQL queries that answer these questions, paste those too — they'll become VQR reference SQL.

Store as `BUSINESS_QUESTIONS` and `SAMPLE_SQL` (if provided).

---

## Phase 3: Introspect

For each table in `SOURCE_TABLES`, run:

```sql
DESCRIBE TABLE <fully_qualified_table_name>;
```

```sql
SELECT * FROM <fully_qualified_table_name> LIMIT 10;
```

### Auto-Classification Rules

Using the DESCRIBE output, classify each column:

| Column Type | Classification | Rule |
|-------------|---------------|------|
| NUMBER, FLOAT, DECIMAL | **Fact** (if it represents a measurable amount) | Columns with names containing: amount, revenue, cost, price, quantity, score, share, rate, count, total, sum |
| NUMBER | **Dimension** (if it's an ID or code) | Columns with names containing: id, code, key, num, number (unless it's clearly a measure) |
| NUMBER | **Time Dimension** (if it's a date stored as integer) | Columns with names containing: date, period, month, year, quarter AND type is NUMBER/INTEGER |
| DATE, TIMESTAMP, DATETIME | **Time Dimension** | Any date/timestamp column |
| VARCHAR, STRING, TEXT | **Dimension** | Default for string columns |
| ARRAY, VARIANT, OBJECT | **Dimension** (with note) | Flag these — they may need special handling or exclusion |
| BOOLEAN | **Dimension** | Boolean flags |

### Cross-Reference with Business Questions

After auto-classification, cross-reference with the user's business questions:

1. Extract column references from the questions (e.g., "revenue by source" → needs a revenue fact + source dimension)
2. Extract column references from sample SQL if provided
3. Promote any columns referenced in questions to "high priority" — these get synonyms, descriptions, and sample values first
4. Flag any questions that reference data NOT in the table — the user may need additional tables or a Cortex Search service

### Present Draft to User

Show the proposed classification as a table:

```
PROPOSED SEMANTIC VIEW: <name>
Base Table: <table>

FACTS (measurable values):
  Column                    | Type          | Proposed Description
  STATEMENT_GROSS_AMOUNT    | NUMBER(38,8)  | Gross revenue amount
  STATEMENT_NET_AMOUNT      | NUMBER(38,8)  | Net revenue amount
  ...

TIME DIMENSIONS (date/period columns):
  Column                    | Type          | Format Notes
  DISTRIBUTION_DATE         | NUMBER(38,0)  | ⚠ Stored as YYYYMM integer, not DATE
  USAGE_PERIOD_START        | DATE          | Standard date
  ...

DIMENSIONS (attributes for filtering/grouping):
  Column                    | Type          | Proposed Description
  WORK_TITLE                | VARCHAR       | Song or work title
  SOURCE_NAME               | VARCHAR       | Royalty collection source
  ...

EXCLUDED (complex types or low-value columns):
  Column                    | Type          | Reason
  EXPLOITATION_PARTIES      | ARRAY         | Complex nested type
  ...

PROPOSED METRICS:
  Name                      | Expression
  TOTAL_REVENUE             | SUM(STATEMENT_GROSS_AMOUNT)
  ...
```

Ask the user to review:

> **Review the proposed classification above.**
>
> - Should any columns move between categories (fact ↔ dimension)?
> - Should any excluded columns be included?
> - Are there metrics you'd like pre-defined beyond what's shown?
> - Any columns that should be renamed or given specific descriptions?

Apply their feedback before proceeding.

---

## Phase 4: Build & Deploy via dbt

### 4a: Load dbt Skills

**MANDATORY:** Before writing any dbt files, load your dbt skills first. Follow those instructions for all project scaffolding decisions (profiles, project structure, CLI commands, testing etc).

### 4b: Scaffold the dbt Project

Using the loaded dbt skills as your guide:

1. **Initialize the project** — Create the dbt project directory structure following the conventions from the system instructions. Use the user's `WAREHOUSE` and `TARGET_DB_SCHEMA` for profile/project configuration.

2. **Add the `dbt_semantic_view` package** — The `packages.yml` must include:
   ```yaml
   packages:
     - package: Snowflake-Labs/dbt_semantic_view
       version: "1.0.3"
   ```
   Then run `dbt deps` to install.

3. **Add the `generate_schema_name` macro** — This prevents dbt from double-prefixing the schema (e.g., `SCHEMA_SCHEMA`). Create `macros/generate_schema_name.sql`:
   ```sql
   {% macro generate_schema_name(custom_schema_name, node) -%}
       {%- if custom_schema_name is none -%}
           {{ default_schema }}
       {%- else -%}
           {{ custom_schema_name | trim }}
       {%- endif -%}
   {%- endmacro %}
   ```

4. **Create the source definition** — `models/sources.yml` referencing the user's source table(s).

5. **Create the semantic view model** — `models/<semantic_view_name>.sql` using the approved classification from Phase 3.

### 4c: Semantic View Model Syntax

The model file uses the `dbt_semantic_view` materialization. This is NOT standard SQL — it's a declarative definition:

```sql
{{ config(materialized='semantic_view') }}

TABLES(
  {{ source('<source_name>', '<TABLE_NAME>') }}
)
FACTS(
  <COLUMN_NAME> COMMENT = '<description>'
  [, ...]
)
DIMENSIONS(
  <COLUMN_NAME> COMMENT = '<description>'
  [, ...]
)
METRICS(
  <METRIC_NAME> AS <EXPRESSION> COMMENT = '<description>'
  [, ...]
)
COMMENT = '<overall semantic view description>'
```

#### Inline COMMENT Best Practices

`persist_docs` is NOT supported by this package. All documentation goes as inline `COMMENT` strings in the DDL:

- **Column descriptions**: `COLUMN_NAME COMMENT = 'What this column represents'`
- **Synonyms**: Include in the comment: `WORK_TITLE COMMENT = 'Title of the musical work. Synonyms: song, track, composition'`
- **Format notes**: `DISTRIBUTION_DATE COMMENT = 'Distribution period as YYYYMM integer (e.g., 202503 = March 2025). NOT a DATE type.'`
- **Sample values**: `SOURCE_NAME COMMENT = 'Royalty source. Examples: ASCAP, BMI, Spotify, YouTube, MLC'`

### 4d: Deploy

Follow the loaded dbt skills for deployment. Run:

```bash
dbt deps                              # install packages (first time)
dbt run --select <semantic_view_name> # deploy the semantic view
```

Refer to the dbt system instructions for connection management, CLI usage, and error handling.

### 4e: Direct DDL Fallback

If the user cannot run dbt (no CLI, no notebook, permissions issues), **always generate the equivalent raw SQL** so they can deploy directly:

```sql
CREATE OR REPLACE SEMANTIC VIEW <TARGET_DB_SCHEMA>.<SEMANTIC_VIEW_NAME>
  TABLES(
    <FULLY_QUALIFIED_TABLE> COMMENT = '<table description>'
  )
  FACTS(
    <COLUMN> COMMENT = '<desc>'
    [, ...]
  )
  DIMENSIONS(
    <COLUMN> COMMENT = '<desc>'
    [, ...]
  )
  METRICS(
    <METRIC> AS <EXPR> COMMENT = '<desc>'
    [, ...]
  )
  COMMENT = '<overall description>';
```

---

## Phase 5: Refine

After the semantic view is deployed, ask follow-up questions to improve quality.

### Question 5: Custom Instructions

> **Are there business rules that Cortex Analyst should always follow when querying this data?**
>
> Examples:
> - "Revenue" means gross revenue (STATEMENT_GROSS_AMOUNT) unless the user says "net"
> - If no date is specified, default to the last 3 distribution periods
> - Exclude rows where REVENUE_TYPE = 'CORRECTION_ADJUSTMENT' unless specifically asked
> - DISTRIBUTION_DATE is stored as YYYYMM integer — convert to readable format in results
>
> These become `custom_instructions` in the semantic view that Cortex Analyst always applies.

### Question 6: Synonyms

> **What business terms do your users use for these columns?**
>
> For each high-priority column, suggest what users might call it:
>
> | Column | Users might say... |
> |--------|--------------------|
> | WORK_TITLE | song, track, composition, title |
> | SOURCE_NAME | source, PRO, collection society |
> | STATEMENT_GROSS_AMOUNT | revenue, collections, gross |
> | DISTRIBUTION_DATE | month, period, distribution month |
>
> Add or correct any mappings.

### Question 7: Sample Values

> **For key columns, what are the most important values to highlight?**
>
> These help Cortex Analyst understand what values exist:
>
> | Column | Important Values |
> |--------|-----------------|
> | SOURCE_NAME | ASCAP, BMI, Spotify, YouTube, MLC |
> | TERRITORY_NAME | UNITED STATES, UNITED KINGDOM, GERMANY |
> | TYPE_OF_RIGHT | PERFORMANCE, MECHANICAL |
>
> Add or correct any values.

### Question 8: Descriptions

> **Review these column descriptions. Are any inaccurate or missing context?**
>
> [Show the current COMMENT values for each column]
>
> Fix any descriptions that are wrong, vague, or missing important context.

After collecting refinements, regenerate the semantic view DDL with updated COMMENTs and redeploy:

```sql
-- Redeploy with refinements
CREATE OR REPLACE SEMANTIC VIEW ...
```

Or if using dbt: update the model `.sql` file, re-upload, and `dbt run --select <model>`.

---

## Phase 6: Validate

### Confirm Deployment

```sql
SHOW SEMANTIC VIEWS IN SCHEMA <TARGET_DB_SCHEMA>;
```

### Inspect the Semantic View

```sql
DESCRIBE SEMANTIC VIEW <TARGET_DB_SCHEMA>.<SEMANTIC_VIEW_NAME>;
```

### Test with Cortex Analyst

Open the Cortex Analyst chat in Snowsight and ask one of the user's business questions. Verify:

1. The question routes to the correct semantic view
2. The generated SQL is valid and returns results
3. The response uses the correct column names and descriptions

### Test with a Verified Query

If sample SQL was provided, compare the Cortex Analyst output against the known-good SQL:

```sql
-- User's original query (known-good)
<SAMPLE_SQL>

-- Cortex Analyst generated query
<ANALYST_SQL>
```

Are the results equivalent? If not, the semantic view may need additional synonyms, descriptions, or verified queries.

---

## Verified Queries (VQRs)

After validation, use the business questions + sample SQL to create verified queries. These are added to the semantic view to teach Cortex Analyst exact query patterns.

VQRs are configured in the Snowsight Semantic View UI:
1. Navigate to the semantic view in Snowsight
2. Go to the "Verified Queries" tab
3. Add each business question with its verified SQL

Or via ALTER SEMANTIC VIEW if supported.

> **Target: 20+ verified queries.** Each of the user's business questions should become a VQR once validated.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `dbt deps` fails — network error | Missing EAI for hub.getdbt.com | Create the EAI from Phase 1 |
| `Materialization 'semantic_view' not found` | Package not installed | Run `dbt deps` first |
| Schema double-prefixed (e.g., `SCHEMA_SCHEMA`) | Missing `generate_schema_name` macro | Add the macro from Phase 4b |
| `CREATE SEMANTIC VIEW` permission denied | Missing grant | `GRANT CREATE SEMANTIC VIEW ON SCHEMA <schema> TO ROLE <role>` |
| Column not found in semantic view | Case sensitivity | Snowflake stores unquoted identifiers as UPPERCASE |
| DISTRIBUTION_DATE queries return wrong results | Integer treated as date | Add COMMENT explaining YYYYMM format + add custom instruction for date handling |
| Cortex Analyst ignores synonyms | Synonyms not in COMMENT | Synonyms must be in the column's COMMENT string, not in a separate config |
| `persist_docs` has no effect | Not supported by this package | Use inline COMMENT syntax instead |

---

## Output Files

After the skill completes, the user will have:

| Artifact | Location | Purpose |
|----------|----------|---------|
| dbt project files | Snowflake stage or local | Version-controlled semantic view definition |
| `CREATE SEMANTIC VIEW` DDL | Executed in Snowflake | Direct deployment fallback |
| Semantic view object | `<TARGET_DB_SCHEMA>.<NAME>` | Live semantic view for Cortex Analyst/Agent |
| Business questions list | Collected during Phase 2 | Input for VQRs and evaluation dataset |
