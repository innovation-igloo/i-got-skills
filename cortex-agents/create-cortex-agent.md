# Skill: Create a Cortex Agent

> **Run this skill inside a Snowsight Workspace.** Open a Workspace in Snowsight, start a Cortex Code chat, and paste this file's contents directly into the prompt.

## Purpose

Walk the user through creating a Snowflake Cortex Agent that uses a Semantic View (Cortex Analyst) and optionally a Cortex Search service to answer natural language questions. The agent is deployed to Snowflake Intelligence so users can interact with it via chat.

This skill assumes the semantic view already exists (built via the `build-semantic-view` skill or manually). It handles agent design, tool configuration, instruction authoring, deployment, and validation.

---

## Workflow Overview

```
Phase 1: GATHER CONTEXT
  ├── Ask: semantic view location
  ├── Ask: agent name + display name + color/avatar
  ├── Ask: target audience + domain scope (in/out)
  ├── Ask: orchestration model (default: openai-gpt-5)
  ├── Ask: orchestration budget (optional: seconds + tokens)
  ├── Ask: Cortex Search service (optional)
  └── Ask: warehouse

Phase 2: DESIGN INSTRUCTIONS
  ├── Generate system instructions (agent identity/persona)
  ├── Generate orchestration instructions (6 sections)
  │     ├── Identity & Scope
  │     ├── Domain Context
  │     ├── Tool Selection (with examples per tool)
  │     ├── Boundaries & Limitations
  │     ├── Business Rules & Conditional Logic
  │     └── Domain-Specific Workflows
  ├── Generate response instructions (4 sections)
  │     ├── Tone & Style
  │     ├── Data Presentation
  │     ├── Response Structure Templates
  │     └── Error & Edge Case Messaging
  ├── Generate tool descriptions (3-step framework)
  └── Generate sample questions

Phase 3: DEPLOY
  ├── Set up SNOWFLAKE_INTELLIGENCE.AGENTS schema (if needed)
  ├── CREATE AGENT with full specification
  └── GRANT access

Phase 4: VALIDATE
  ├── SHOW AGENTS / DESCRIBE AGENT
  ├── Test in Snowflake Intelligence UI
  └── Iterate on instructions based on results
```

---

## Phase 1: Gather Context

### Question 1: Semantic View

> **What is the fully qualified name of the semantic view this agent should use?**
>
> Format: `DATABASE.SCHEMA.SEMANTIC_VIEW_NAME`
>
> This is the semantic view built in the previous skill (or an existing one).

Store as `SEMANTIC_VIEW`.

### Question 2: Agent Identity

> **What should the agent be named?**
>
> - **Agent object name** (SQL identifier, no spaces): e.g., `PUBLISHING_REVENUE_AGENT`
> - **Display name** (shown in Snowflake Intelligence UI): e.g., `Publishing Revenue Assistant`
> - **Description** (1-2 sentences for the UI): e.g., `Answers questions about publishing royalty revenue from the Common Revenue Model.`
> - **Color** (optional, for the SI UI): e.g., `blue`, `green`, `red`
> - **Avatar** (optional, image file name or identifier)

Store as `AGENT_NAME`, `DISPLAY_NAME`, `AGENT_DESCRIPTION`, `COLOR`, `AVATAR`.

### Question 3: Domain Scope

> **What domain does this agent cover, and what is out of scope?**
>
> - **In scope**: What data and questions can the agent answer?
> - **Out of scope**: What should the agent explicitly decline?
> - **Target audience**: Who will use this agent?
>
> Example:
> - In scope: Publishing royalty revenue by song, source, territory, period
> - Out of scope: Recorded music, theatrical revenue, contract details, real-time streaming
> - Audience: Publishing team, finance, business analysts

Store as `IN_SCOPE`, `OUT_OF_SCOPE`, `AUDIENCE`.

### Question 4: Orchestration Model

> **Which LLM should orchestrate the agent?**
>
> Available models for Cortex Agents:
> - `openai-gpt-5` — **Recommended.** Best OpenAI model available. (Preview)
> - `openai-gpt-4-1` — OpenAI GPT-4.1, generally available.
> - `auto` — Let Snowflake select the highest quality model for your account.
> - `claude-4-sonnet` — Best overall quality. (Requires Claude approval.)
> - `claude-sonnet-4-5` — Latest Claude. (Requires Claude approval.)
> - `claude-haiku-4-5` — Fast + cost-efficient Claude. (Preview, requires Claude approval.)
>
> **If your organization restricts Claude models, use `openai-gpt-5`.**

Store as `ORCHESTRATION_MODEL`. Default to `openai-gpt-5` if the user has no preference or has Claude restrictions.

### Question 5: Orchestration Budget (Optional)

> **Do you want to set a budget limit for agent orchestration?**
>
> - **seconds**: Max wall-clock time per agent turn (e.g., `30`)
> - **tokens**: Max token budget per agent turn (e.g., `16000`)
>
> If not specified, Snowflake defaults apply.

Store as `BUDGET_SECONDS`, `BUDGET_TOKENS` (or null).

### Question 6: Cortex Search (Optional)

> **Does this agent need access to unstructured knowledge (glossary, docs, business context)?**
>
> If yes, provide the Cortex Search service name: `DATABASE.SCHEMA.SEARCH_SERVICE_NAME`
>
> If no Cortex Search service exists yet and you want one, we can build it.
> If not needed, skip this — the agent will use only the semantic view.

Store as `SEARCH_SERVICE` (or null).

### Question 7: Warehouse

> **Which warehouse should the agent use for executing queries?**

Store as `WAREHOUSE`.

---

## Phase 2: Design Instructions

Every Cortex Agent combines your custom instructions with Snowflake's built-in base system instructions. The base instructions handle general workflows for tool usage, data analysis, validation, visualization, citation, and safety guardrails.

**You do NOT need to instruct the agent on general reasoning.** For example:

```
❌ DON'T include:
"When you receive a question, first analyze it carefully, then
select appropriate tools, call them in sequence, and format results properly..."
```

Snowflake handles that. Your job is to provide **domain-specific context** across the instruction layers below.

### Where Does Each Instruction Go?

Use this decision table to categorize instructions:

| If the instruction affects... | Put it in... |
|-------------------------------|-------------|
| Which tool to select | Orchestration |
| What data to retrieve | Orchestration |
| How to interpret user intent | Orchestration |
| How to sequence tool calls | Orchestration |
| Conditional logic and rules | Orchestration |
| What to do in specific scenarios | Orchestration |
| How to format the answer | Response |
| What tone to use | Response |
| How to structure text | Response |
| What to say when errors occur | Response |
| The agent's core identity/persona | System |

### System Instructions

System instructions define the agent's core identity. Keep this brief — 1-2 sentences.

**Template:**

```
You are a friendly agent that helps {AUDIENCE} with {IN_SCOPE} questions.
```

Store as `SYSTEM_INSTRUCTIONS`.

### Orchestration Instructions

Generate from the user's domain context. Structure into **6 sections**:

**Template:**

```
IDENTITY & SCOPE:
You are the {DISPLAY_NAME} for {ORGANIZATION}.
Your Scope: You answer questions about {IN_SCOPE}.
Your Users: {AUDIENCE} who need {what they need}.

DOMAIN CONTEXT:
{Key business terms and definitions}
{Date format notes (e.g., "DISTRIBUTION_DATE is YYYYMM integer, not a real date")}
{Default behaviors (e.g., "When no date specified, default to last 3 periods")}
{Disambiguation rules (e.g., "'revenue' means STATEMENT_GROSS_AMOUNT unless they say 'net'")}

TOOL SELECTION:
- For {category 1} questions: Use {ANALYST_TOOL_NAME}
    Examples: "{example question 1}", "{example question 2}"
{IF SEARCH_SERVICE}
- For {category 2} questions: Use {SEARCH_TOOL_NAME}
    Examples: "{example question 3}", "{example question 4}"
- If a question mentions a term you're unsure about, search for the definition first, then query the data
{ENDIF}

BOUNDARIES & LIMITATIONS:
- You do NOT have access to: {OUT_OF_SCOPE}
- Do NOT make financial forecasts or predictions
- Do NOT provide {restricted data types} for privacy reasons
- If data is not available: "{specific message about what to do}"
{IF DATA_HAS_LAG}
- Data is refreshed {frequency}. If asked about real-time data, clarify: "My data is current as of {lag description}."
{ENDIF}

BUSINESS RULES:
{IF LOOKUP_REQUIRED}
- When a user asks about {entity} by name (not ID), ALWAYS use {lookup_tool} first to get the ID before calling other tools
{ENDIF}
- If a query result returns more than {N} rows, aggregate or filter before presenting
{IF DATE_AMBIGUITY}
- When "{ambiguous term}" is used, default to {specific behavior}
{ENDIF}
- {Additional domain-specific rules}

WORKFLOWS:
{IF MULTI_STEP_WORKFLOW}
{Workflow Name}: When a user asks to "{trigger phrase}":
1. Use {tool_1} to get {data_1}
2. Use {tool_2} to get {data_2}
3. Present findings with {format requirements}
{ENDIF}
```

### Response Instructions

Structure into **4 sections**:

**Template:**

```
TONE & STYLE:
- Be concise and direct — lead with the answer, then supporting detail
- Be direct with data. Avoid hedging language like "it seems" or "it appears"
- Use active voice and clear statements
{IF DOMAIN_SPECIFIC_TONE}
- {Additional tone guidance}
{ENDIF}

DATA PRESENTATION:
- Use tables for multi-row results (3+ items)
- Use charts for comparisons, trends, and rankings
- For single values, state them directly without tables
- Always include units ({currency}, %) with numbers
- Format large numbers with commas (e.g., $1,234,567.89)
{IF DATE_IS_YYYYMM}
- Display distribution dates as readable periods (e.g., "March 2025") not raw integers (202503)
{ENDIF}
{IF DATA_HAS_LAG}
- Include data freshness timestamp in responses when relevant
{ENDIF}

RESPONSE STRUCTURE:

For "What is X?" questions:
    - Lead with the direct answer
    - Follow with supporting context if relevant

For "Show me X" questions:
    - Brief summary sentence
    - Table or chart with data
    - Key insights or highlights

For "Compare X and Y" questions:
    - Summary of comparison result
    - Chart or table showing comparison
    - Notable differences highlighted

ERRORS & EDGE CASES:
- When data is unavailable: "I don't have access to [data type]. You can find this in [alternative] or contact [team]."
- When query is ambiguous: "To provide accurate data, I need clarification: [specific question]. Did you mean [option A] or [option B]?"
- When results are empty: "No records found for [criteria]. This could mean [possible reason]. Would you like to try [alternative]?"
- When out of scope: "I don't have access to [topic]. I can help with {IN_SCOPE}."
```

### Tool Descriptions (3-Step Framework)

Tool descriptions are the **#1 driver of agent quality**. Agents choose tools based on name and description context. Bad descriptions create cascading failures — wrong tool selection leads to wrong data, which leads to wrong answers.

#### Step 1: Clear, specific tool name

Combine domain + function. The name is loaded into the agent's context and influences selection.

```
✅ GOOD: "PublishingRevenueAnalytics"     ❌ BAD: "DataTool" or "Tool1"
✅ GOOD: "RevenueKnowledgeSearch"         ❌ BAD: "Search" or "Docs"
```

#### Step 2: Purpose-driven description — WHAT + DATA + WHEN + WHEN NOT

"When NOT to Use" is critical. Without it, agents try to use tools for everything remotely related. It creates clear boundaries and redirects the agent to alternatives.

**Tip for Cortex Analyst tools:** Start with "Generate with Cortex" in the Agent UI to auto-generate a baseline description from your semantic model, then enhance it using this framework.

**Cortex Analyst tool template:**

```
Name: {domain}Analytics (e.g., PublishingRevenueAnalytics)

Description:
Analyzes {domain} data from {table/model description}. Covers {list key facts and dimensions}.

Data Coverage:
- {What data is available, time range, granularity}
- {Update frequency and lag}
- {Data sources}

When to Use:
- {Use case 1 from business questions}
- {Use case 2}
- {Use case 3}

When NOT to Use:
- Do NOT use for {out of scope item 1} (use {alternative tool} instead)
- Do NOT use for {out of scope item 2}
- Do NOT use for {real-time/lag caveat}
```

**Cortex Search tool template (if applicable):**

```
Name: {domain}KnowledgeSearch (e.g., PublishingKnowledgeSearch)

Description:
Searches {domain} business documentation including {list content types: glossary, definitions, standards, etc.}. Uses semantic search to find relevant documents even when exact keywords don't match.

Data Sources:
- {Source 1} (update frequency)
- {Source 2} (update frequency)

When to Use:
- Questions about terminology, definitions, or business concepts
- "What is" or "what does X mean" questions
- When you need to understand a term before querying data

When NOT to Use:
- Revenue amounts, totals, or aggregations → use {analyst_tool_name} instead
- Questions requiring computation or data aggregation
- Real-time system status

Search Query Best Practices:
1. Use specific terms:
    ✅ "{example good query with domain terms}"
    ❌ "{generic term}" (too broad)
2. Include multiple related keywords:
    ✅ "{example multi-keyword query}"
3. If first search returns low relevance, rephrase with synonyms or expanded acronyms
4. No-results handling: "I couldn't find documentation about that term. This may not be in our knowledge base yet."
```

#### Step 3: Be explicit about tool inputs

This is where most tool descriptions fail. Ambiguous inputs lead to incorrect tool calls and errors.

| Common Pitfall | Fix |
|----------------|-----|
| Generic name: "date" | Specific: "distribution_date in YYYYMM integer format (e.g., 202503)" |
| Unclear how to get values: "Provide work_title" | "Use knowledge search first to resolve song title if user gives partial name" |
| Inconsistent terminology: instructions say "source" but tool says "exploitation_source_name" | Pick one term, define it, use it everywhere |
| Unclear optionality: "region (optional)" | "region (optional, defaults to 'ALL', returns data for all regions)" |

### Sample Questions

Select 3-5 business questions from the user's list that best showcase the agent's capabilities. These help users understand the agent's purpose in Snowflake Intelligence. Each question should be independent and connect back to the agent's predefined purpose.

```yaml
sample_questions:
  - question: "{business question 1}"
    answer: "{brief description of how agent would answer}"
  - question: "{business question 2}"
    answer: "{brief description}"
  - question: "{business question 3}"
    answer: "{brief description}"
```

**Present all generated instructions to the user for review before deploying.**

---

## Phase 3: Deploy

### 3a: Set Up Snowflake Intelligence Schema

```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
GRANT USAGE ON DATABASE SNOWFLAKE_INTELLIGENCE TO ROLE PUBLIC;

CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;
GRANT USAGE ON SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS TO ROLE PUBLIC;

GRANT CREATE AGENT ON SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS TO ROLE <CURRENT_ROLE>;
```

### 3b: Create the Agent

Generate and execute the full `CREATE AGENT` statement:

```sql
CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.<AGENT_NAME>
  COMMENT = '<AGENT_DESCRIPTION>'
  PROFILE = '{"display_name": "<DISPLAY_NAME>", "color": "<COLOR>"}'
  FROM SPECIFICATION $$
  models:
    orchestration: <ORCHESTRATION_MODEL>

  orchestration:
    budget:
      seconds: <BUDGET_SECONDS>
      tokens: <BUDGET_TOKENS>

  instructions:
    system: |
      <SYSTEM_INSTRUCTIONS>
    orchestration: |
      <ORCHESTRATION_INSTRUCTIONS>
    response: |
      <RESPONSE_INSTRUCTIONS>
    sample_questions:
      - question: "<question 1>"
        answer: "<answer 1>"
      - question: "<question 2>"
        answer: "<answer 2>"
      - question: "<question 3>"
        answer: "<answer 3>"

  tools:
    - tool_spec:
        type: cortex_analyst_text_to_sql
        name: <ANALYST_TOOL_NAME>
        description: |
          <ANALYST_TOOL_DESCRIPTION>
{IF SEARCH_SERVICE}
    - tool_spec:
        type: cortex_search
        name: <SEARCH_TOOL_NAME>
        description: |
          <SEARCH_TOOL_DESCRIPTION>
{ENDIF}

  tool_resources:
    <ANALYST_TOOL_NAME>:
      semantic_view: "<SEMANTIC_VIEW>"
{IF SEARCH_SERVICE}
    <SEARCH_TOOL_NAME>:
      name: "<SEARCH_SERVICE>"
      max_results: 5
      title_column: "<TITLE_COLUMN>"
      id_column: "<ID_COLUMN>"
{ENDIF}
  $$;
```

**Notes on the specification:**
- `models.orchestration`: Set to `openai-gpt-5` (or user's choice). If omitted, Snowflake auto-selects.
- `orchestration.budget`: Optional. Omit the entire block if no budget constraints needed.
- `instructions.system`: Brief identity/persona. Separate from orchestration logic.
- `instructions.orchestration`: Business logic, tool selection, boundaries, workflows.
- `instructions.response`: Format, tone, error messaging.
- `tool_resources` keys must match `tool_spec.name` exactly.
- Cortex Search `tool_resources` uses `name` (the service name), not `search_service`.
- Max specification size: 100,000 bytes.

### 3c: Grant Access

```sql
GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.<AGENT_NAME> TO ROLE PUBLIC;

GRANT SELECT ON SEMANTIC VIEW <SEMANTIC_VIEW> TO ROLE PUBLIC;

{IF SEARCH_SERVICE}
GRANT USAGE ON CORTEX SEARCH SERVICE <SEARCH_SERVICE> TO ROLE PUBLIC;
{ENDIF}
```

---

## Phase 4: Validate

### Confirm Deployment

```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;

DESC AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.<AGENT_NAME>;
```

### Test in Snowflake Intelligence

1. Navigate to **AI & ML → Snowflake Intelligence** in Snowsight
2. The agent should appear in the list with the display name and description
3. Sample questions should show as suggested prompts
4. Test with one of the business questions from Phase 1

### Evaluate Results

For each test question, check:
- Did the agent select the correct tool?
- Did the generated SQL use the right columns and filters?
- Is the response format correct (tables, currency, dates)?
- Did it handle edge cases (no results, ambiguous terms)?

### Iterate

If results are poor, use the **4 improvement levers**:

| Lever | When to Use | How |
|-------|------------|-----|
| Fix tool descriptions | Agent picks wrong tool or misuses inputs | Update "When to Use / When NOT to Use" and input specifics |
| Fix orchestration instructions | Agent reasons incorrectly about scope or sequencing | Update domain context, boundaries, or workflows |
| Add verified queries | Common queries generate wrong SQL | Add VQRs in the semantic view |
| Optimize semantic view | Column descriptions are unclear or agent misinterprets data | Update COMMENTs, add synonyms, sample values |

To update the agent:

```sql
CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.<AGENT_NAME>
  COMMENT = '<AGENT_DESCRIPTION>'
  PROFILE = '{"display_name": "<DISPLAY_NAME>", "color": "<COLOR>"}'
  FROM SPECIFICATION $$
  <UPDATED_SPECIFICATION>
  $$;
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Agent not visible in Snowflake Intelligence | Created in wrong schema | Must be in `SNOWFLAKE_INTELLIGENCE.AGENTS` |
| `CREATE AGENT` permission denied | Missing grant | `GRANT CREATE AGENT ON SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS TO ROLE <role>` |
| Agent can't access semantic view | Missing SELECT grant | `GRANT SELECT ON SEMANTIC VIEW <view> TO ROLE PUBLIC` |
| "I don't have access to that data" | Warehouse not running or not specified | Check warehouse is running; user's default warehouse is used |
| Agent gives poor answers | Weak tool descriptions | Improve description with domain-specific WHEN / WHEN NOT |
| Agent picks wrong tool | Tool descriptions overlap or are too generic | Add "When NOT to Use" with redirects to the correct tool |
| YAML parse error in specification | Invalid YAML | Check indentation, quote strings with special chars |
| Sample questions not showing | Wrong YAML structure | `sample_questions` under `instructions`, each needs `question` + `answer` |
| `tool_resources` key mismatch | Name doesn't match `tool_spec.name` | Keys must match exactly |
| Agent too verbose | Response instructions too open | Add "Be concise. No preamble." to response instructions |
| Agent hallucinates data | Missing boundaries | Add explicit "do NOT" rules to orchestration instructions |
| Agent ignores business rules | Rules in wrong instruction layer | Move to orchestration instructions, not response |
| Cortex Search tool_resources error | Using wrong key name | Use `name` (not `search_service`) for the service reference |

---

## Output

After the skill completes, the user will have:

| Artifact | Location | Purpose |
|----------|----------|---------|
| Cortex Agent | `SNOWFLAKE_INTELLIGENCE.AGENTS.<NAME>` | Live agent in Snowflake Intelligence |
| `CREATE AGENT` DDL | Provided as SQL | Version-controllable agent definition |
| System instructions | In agent spec | Agent identity/persona |
| Orchestration instructions | In agent spec | Tool selection + domain logic + workflows |
| Response instructions | In agent spec | Output format + tone + error messaging |
| Tool descriptions | In agent spec | Per-tool routing guidance |
| Sample questions | In agent spec | User-facing suggested prompts |
