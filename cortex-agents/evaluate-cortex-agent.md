# Skill: Evaluate a Cortex Agent

> **Run this skill inside a Snowsight Workspace.** Open a Workspace in Snowsight, start a Cortex Code chat, and paste this file's contents directly into the prompt.

## Purpose

Walk the user through setting up and running Cortex Agent Evaluations in Snowflake. The skill handles permissions, dataset creation, YAML configuration, evaluation execution, result inspection, and iterative improvement — all from within Snowsight / Cortex Code.

This skill assumes a Cortex Agent already exists (built via the `create-cortex-agent` skill or manually).

---

## Workflow Overview

```
Phase 1: PREREQUISITES
  ├── Verify permissions (CORTEX_USER, EXECUTE TASK, USAGE/MONITOR on agent)
  ├── Confirm model availability (claude-4-sonnet or claude-3-5-sonnet required)
  └── Ask: agent name + location, evaluation schema, warehouse

Phase 2: BUILD EVALUATION DATASET
  ├── Ask: business questions + expected outputs (or reuse existing VQRs)
  ├── Create evaluation table (input_query VARCHAR, expected_output VARIANT)
  └── Review dataset with user before proceeding

Phase 3: CONFIGURE & RUN
  ├── Select metrics (answer_correctness, logical_consistency, optional custom)
  ├── Generate YAML configuration
  ├── Upload to stage
  └── Start evaluation (Snowsight UI or SQL)

Phase 4: INSPECT & IMPROVE
  ├── Check run status
  ├── Review per-record results (scores, thread details, traces)
  ├── Identify failures — classify root cause
  └── Apply improvement levers and re-evaluate
```

---

## Phase 1: Prerequisites

### 1a: Verify Permissions

Run the following checks for the user's current role:

```sql
-- Check CORTEX_USER database role
SELECT COUNT(*) FROM INFORMATION_SCHEMA.APPLICABLE_ROLES
  WHERE ROLE_NAME = 'SNOWFLAKE.CORTEX_USER';

-- Check EXECUTE TASK
SHOW GRANTS TO ROLE <CURRENT_ROLE>;
-- Look for: EXECUTE TASK ON ACCOUNT
```

The role running evaluations needs ALL of the following:
- `SNOWFLAKE.CORTEX_USER` database role
- `EXECUTE TASK ON ACCOUNT`
- `USAGE` on the database and schema containing the agent
- `CREATE FILE FORMAT`, `CREATE TASK`, `EXECUTE TASK` on the agent's schema
- `USAGE` on the database containing evaluation data
- `USAGE`, `EXECUTE TASK`, `CREATE DATASET ON SCHEMA` on the evaluation data schema
- `USAGE` or `OWNERSHIP` on the agent
- `MONITOR` or `OWNERSHIP` on the agent
- If using a YAML config on a stage: `READ` on that stage
- If the agent has tools: access to all tools (semantic view SELECT, search service USAGE, etc.)
- `USAGE` on the warehouse used in Snowsight

If any permissions are missing, generate the appropriate `GRANT` statements and present them to the user.

### 1b: Model Availability

> **Important:** Cortex Agent Evaluations currently only support `claude-4-sonnet` and `claude-3-5-sonnet` via cross-region inference.
>
> If your organization has restrictions on Claude models, evaluations cannot run programmatically. In that case, scope to **manual testing** in the Snowflake Intelligence UI and skip to Phase 4's manual testing guidance.

Confirm with the user whether Claude models are approved for internal tooling (even if not for the agent itself).

### 1c: Gather Context

> **What is the fully qualified name of the agent to evaluate?**
>
> Format: `DATABASE.SCHEMA.AGENT_NAME`

Store as `AGENT_DB`, `AGENT_SCHEMA`, `AGENT_NAME`.

> **Where should the evaluation dataset and config live?**
>
> This can be the same schema as the agent or a separate one. Format: `DATABASE.SCHEMA`

Store as `EVAL_DB`, `EVAL_SCHEMA`.

> **Which warehouse should be used for evaluation tasks?**

Store as `WAREHOUSE`.

---

## Phase 2: Build Evaluation Dataset

### 2a: Collect Questions and Expected Outputs

> **Provide the questions you want to evaluate the agent against, along with expected outputs.**
>
> Expected outputs are natural language descriptions — they're fed to an LLM judge, not matched exactly. You can describe:
> - The expected answer content ("Net revenue for In the Air Tonight in 2025, summed across all sources")
> - The expected format ("A table with columns: month, gross_revenue, net_revenue")
> - Boundary behavior ("The agent should decline, saying it doesn't have contract data")
>
> **Sources for questions:**
> - Your top 20 business questions from the semantic view exercise
> - Existing verified queries (VQRs) from the semantic view
> - Edge cases: out-of-scope questions, ambiguous terms, empty result scenarios
>
> **Ground truth best practice:** Use absolute dates ("revenue between January and March 2025"), not relative ("last quarter"). Relative dates drift and produce inconsistent eval results.

If the user has existing VQRs or a question list, help them transform those into evaluation rows.

### 2b: Create the Dataset Table

```sql
USE WAREHOUSE <WAREHOUSE>;

CREATE TABLE IF NOT EXISTS <EVAL_DB>.<EVAL_SCHEMA>.AGENT_EVAL_DATA (
    input_query VARCHAR,
    expected_output VARIANT
);
```

For each question/expected-output pair, generate an INSERT:

```sql
INSERT INTO <EVAL_DB>.<EVAL_SCHEMA>.AGENT_EVAL_DATA
SELECT
    '<INPUT_QUERY>',
    PARSE_JSON('{
        "ground_truth_output": "<EXPECTED_OUTPUT>"
    }');
```

**Important:** Use `PARSE_JSON` — not `OBJECT_CONSTRUCT` — to ensure the column is `VARIANT` type.

### 2c: Validate the Dataset

```sql
SELECT input_query, expected_output:ground_truth_output::VARCHAR AS ground_truth
FROM <EVAL_DB>.<EVAL_SCHEMA>.AGENT_EVAL_DATA;
```

Present the full dataset to the user for review before proceeding. Confirm:
- All questions are phrased with absolute dates (no "last month", "this quarter")
- Expected outputs describe the answer, not exact values
- Edge cases are included (out-of-scope, empty results, ambiguous terms)

---

## Phase 3: Configure and Run

### 3a: Select Metrics

Present the available metrics:

| Metric | What It Measures | Needs Ground Truth? |
|--------|-----------------|-------------------|
| `answer_correctness` | How closely the agent's answer matches expected output | Yes |
| `logical_consistency` | Consistency across instructions, planning, and tool calls | No (reference-free) |

Ask if the user wants a **custom metric**. If yes, gather:
- Metric name (e.g., `revenue_accuracy`)
- Scoring prompt (what should the LLM judge evaluate?)
- Score ranges: min_score, median_score, max_score

**Available replacement strings for custom metric prompts:**

| String | Maps To |
|--------|---------|
| `{{input}}` | The input query |
| `{{output}}` | The agent's response |
| `{{ground_truth}}` | The ground truth from the dataset |
| `{{tool_info}}` | Tool invocation details |
| `{{duration}}` | Response time in ms |
| `{{error}}` | Error information (if any) |

### 3b: Generate YAML Configuration

```yaml
dataset:
  dataset_type: "CORTEX AGENT"
  table_name: "<EVAL_DB>.<EVAL_SCHEMA>.AGENT_EVAL_DATA"
  dataset_name: "<AGENT_NAME>_EVAL"
  column_mapping:
    query_text: "INPUT_QUERY"
    ground_truth: "EXPECTED_OUTPUT"

evaluation:
  agent_params:
    agent_name: "<AGENT_NAME>"
    agent_type: "CORTEX AGENT"
  run_params:
    label: "<RUN_LABEL>"
    description: "<RUN_DESCRIPTION>"
  source_metadata:
    type: "DATASET"
    dataset_name: "<AGENT_NAME>_EVAL"

metrics:
  - "answer_correctness"
  - "logical_consistency"
# {IF CUSTOM_METRIC}
#  - name: "<CUSTOM_METRIC_NAME>"
#    score_ranges:
#      min_score: [1, 3]
#      median_score: [4, 6]
#      max_score: [7, 10]
#    prompt: |
#      <CUSTOM_PROMPT with {{output}}, {{ground_truth}}, etc.>
# {ENDIF}
```

Present the YAML to the user for review.

### 3c: Upload to Stage

```sql
CREATE FILE FORMAT IF NOT EXISTS <EVAL_DB>.<EVAL_SCHEMA>.yaml_file_format
  TYPE = 'CSV'
  FIELD_DELIMITER = NONE
  RECORD_DELIMITER = '\n'
  SKIP_HEADER = 0
  FIELD_OPTIONALLY_ENCLOSED_BY = NONE
  ESCAPE_UNENCLOSED_FIELD = NONE;

CREATE STAGE IF NOT EXISTS <EVAL_DB>.<EVAL_SCHEMA>.eval_config_stage
  FILE_FORMAT = <EVAL_DB>.<EVAL_SCHEMA>.yaml_file_format;
```

Upload the YAML. If in a Snowsight workspace:

```sql
-- From workspace to stage
COPY FILES INTO @<EVAL_DB>.<EVAL_SCHEMA>.eval_config_stage
  FROM 'snow://workspace/USER$.PUBLIC.DEFAULT$/versions/live'
  FILES=('agent_evaluation_config.yaml');
```

Or guide the user to upload via **Snowsight → Ingestion → Add Data → Load files into a Stage**.

### 3d: Start the Evaluation

**Option A — Snowsight UI:**
1. Navigate to **AI & ML → Agents**
2. Select the agent → **Evaluations** tab
3. Click **New evaluation run**
4. Name the run, select or create the dataset, select metrics
5. If using custom metrics: select the stage and YAML file
6. Click **Create**

**Option B — SQL:**

```sql
CALL EXECUTE_AI_EVALUATION(
  'START',
  OBJECT_CONSTRUCT('run_name', '<RUN_NAME>'),
  '@<EVAL_DB>.<EVAL_SCHEMA>.eval_config_stage/agent_evaluation_config.yaml'
);
```

**Check status:**

```sql
CALL EXECUTE_AI_EVALUATION(
  'STATUS',
  OBJECT_CONSTRUCT('run_name', '<RUN_NAME>'),
  '@<EVAL_DB>.<EVAL_SCHEMA>.eval_config_stage/agent_evaluation_config.yaml'
);
```

---

## Phase 4: Inspect and Improve

### 4a: Review Results

**Snowsight UI:**
1. AI & ML → Agents → select agent → Evaluations tab
2. Click the run → see per-input scores, output, duration
3. Click a record → **Evaluation results** (metric breakdown), **Thread details** (planning + tool calls), **Trace details** (per-step input/output)

**SQL:**

```sql
-- Full evaluation results
SELECT * FROM TABLE(SNOWFLAKE.LOCAL.GET_AI_EVALUATION_DATA(
    '<AGENT_DB>', '<AGENT_SCHEMA>',
    '<AGENT_NAME>', 'CORTEX AGENT', '<RUN_NAME>'
));

-- Drill into a single record trace
SELECT * FROM TABLE(SNOWFLAKE.LOCAL.GET_AI_RECORD_TRACE(
    '<AGENT_DB>', '<AGENT_SCHEMA>',
    '<AGENT_NAME>', 'CORTEX AGENT',
    '<RECORD_ID>'
));

-- Check for errors and warnings
SELECT * FROM TABLE(SNOWFLAKE.LOCAL.GET_AI_OBSERVABILITY_LOGS(
    '<AGENT_DB>', '<AGENT_SCHEMA>',
    '<AGENT_NAME>', 'CORTEX AGENT'
))
WHERE record:"severity_text" IN ('ERROR', 'WARN')
  AND record_attributes:"snow.ai.observability.run.name" = '<RUN_NAME>';
```

**Cortex Code sub-skills:**
- `investigate-cortex-agent-evals` — inspect evaluation results and find issues
- `optimize-cortex-agent` — take results from completed evaluations and improve agent performance

### 4b: Classify Failures

For each low-scoring record, examine the trace to identify the root cause:

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| Wrong tool selected | Weak tool description | Update "When to Use / When NOT to Use" in tool description |
| Correct tool, wrong SQL | Missing VQR or poor column descriptions | Add verified query or improve semantic view COMMENTs |
| Correct data, wrong interpretation | Orchestration instructions incomplete | Add domain context or disambiguation rules |
| Correct answer, wrong format | Response instructions incomplete | Update response formatting rules |
| Agent declines valid question | Boundaries too restrictive | Relax scope in orchestration instructions |
| Agent answers out-of-scope question | Boundaries too loose | Tighten scope, add explicit "do NOT" rules |
| Timeout or error | Complex query or service issue | Simplify the question, check warehouse size, split eval dataset |

### 4c: Apply Improvements and Re-Evaluate

After fixing the agent (instructions, semantic view, or tool descriptions), run a new evaluation to measure improvement:

```sql
CALL EXECUTE_AI_EVALUATION(
  'START',
  OBJECT_CONSTRUCT('run_name', '<RUN_NAME>-v2'),
  '@<EVAL_DB>.<EVAL_SCHEMA>.eval_config_stage/agent_evaluation_config.yaml'
);
```

Compare runs side-by-side in the Snowsight Evaluations tab. Repeat until scores stabilize.

### 4d: Schedule Recurring Evaluations (Production)

Once the agent is stable, set up a scheduled evaluation using a Snowflake Task:

```sql
CREATE OR REPLACE TASK <EVAL_DB>.<EVAL_SCHEMA>.WEEKLY_AGENT_EVAL
  WAREHOUSE = <WAREHOUSE>
  SCHEDULE = 'USING CRON 0 6 * * MON America/New_York'
AS
  CALL EXECUTE_AI_EVALUATION(
    'START',
    OBJECT_CONSTRUCT('run_name', 'weekly-' || TO_VARCHAR(CURRENT_DATE(), 'YYYY-MM-DD')),
    '@<EVAL_DB>.<EVAL_SCHEMA>.eval_config_stage/agent_evaluation_config.yaml'
  );

ALTER TASK <EVAL_DB>.<EVAL_SCHEMA>.WEEKLY_AGENT_EVAL RESUME;
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `EXECUTE_AI_EVALUATION` permission denied | Missing EXECUTE TASK or CORTEX_USER | Grant `EXECUTE TASK ON ACCOUNT` and `SNOWFLAKE.CORTEX_USER` database role |
| Evaluation run shows warnings | Some inputs failed during agent execution | Check `GET_AI_OBSERVABILITY_LOGS` for errors; split long-running queries |
| All answer_correctness scores are low | Ground truth too specific or uses relative dates | Rewrite ground truth as natural language descriptions with absolute dates |
| Custom metric returns unexpected scores | Prompt doesn't produce numeric output in range | Ensure prompt includes explicit scoring rubric with numeric values |
| Dataset creation fails | `expected_output` column not VARIANT | Use `PARSE_JSON`, not `OBJECT_CONSTRUCT` |
| "Model not available" error | Claude not enabled or cross-region inference off | Enable cross-region inference; evaluations require claude-4-sonnet or claude-3-5-sonnet |
| Evaluation times out | Too many inputs or complex queries | Split dataset into smaller batches; simplify custom metric prompts |
| YAML parse error | Invalid YAML formatting | Check indentation, quote strings with special characters, validate YAML syntax |
| Stage READ permission denied | Role can't read the config stage | `GRANT READ ON STAGE <stage> TO ROLE <role>` |

---

## Cost Considerations

Each evaluation run incurs:
- **Agent execution**: The agent runs against every input query (same as user interaction costs)
- **LLM judges**: AI_COMPLETE calls for each metric × each input (claude-4-sonnet or claude-3-5-sonnet)
- **Warehouse**: Tasks for managing evaluation runs + queries for computing metrics
- **Storage**: Datasets and evaluation results

For a 20-question dataset with 2 built-in metrics + 1 custom metric: expect ~60 LLM judge calls per run plus 20 agent invocations. Budget accordingly for iterative runs.

---

## Output

After the skill completes, the user will have:

| Artifact | Location | Purpose |
|----------|----------|---------|
| Evaluation dataset table | `<EVAL_DB>.<EVAL_SCHEMA>.AGENT_EVAL_DATA` | Ground truth questions + expected outputs |
| YAML config file | `@<EVAL_DB>.<EVAL_SCHEMA>.eval_config_stage/agent_evaluation_config.yaml` | Evaluation configuration |
| Evaluation run results | Snowsight Evaluations tab + SQL functions | Per-record scores, traces, and metrics |
| Improvement action items | From failure classification | Targeted fixes for agent instructions, semantic view, or tool descriptions |
| Scheduled evaluation task (optional) | `<EVAL_DB>.<EVAL_SCHEMA>.WEEKLY_AGENT_EVAL` | Recurring regression testing |
