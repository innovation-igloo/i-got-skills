---
name: cortex-agent-best-practice
description: "Interactive build guide for production-quality Cortex Agents. Walks through agent purpose, tool mapping, orchestration instructions, response instructions, tool descriptions, and deployment readiness. Generates a complete agent-config.md output. Use when: building a Cortex Agent, writing agent instructions, configuring Snowflake Intelligence, agent best practices, improve my agent, tool descriptions, orchestration instructions, response instructions, cortex agent setup."
---

# Cortex Agent Best Practices — Interactive Build Guide

## Setup

On load:
1. Say: "Let's build your Cortex Agent the right way. I'll guide you through each layer of configuration, ask you focused questions, and write a complete `agent-config.md` as we go. By the end you'll have something ready to paste into the Snowflake Agent UI."
2. Create `agent-config.md` in the working directory with these section stubs:
   ```
   # Agent Configuration
   ## Agent Identity
   ## Tool Inventory
   ## Orchestration Instructions
   ## Response Instructions
   ## Tool Descriptions
   ## Deployment Checklist
   ```
3. Ask the user how they want to proceed:

```
How would you like to start?

1. Full walkthrough — go through all 6 modules in order (recommended for new agents)
2. Jump to a specific module:
   1. Agent Purpose
   2. Tool Mapping
   3. Orchestration Instructions
   4. Response Instructions
   5. Tool Descriptions
   6. Deploy & Evaluate
```

**Route based on selection:**
- Full walkthrough → proceed to Module 1, chain through 2→3→4→5→6
- Specific module → jump to that section

---

## Module 1: Agent Purpose

**Key principle:** Narrow beats broad. Before writing a single line of instructions, lock in who the agent serves, what it answers, and what it does NOT answer. This shapes every decision that follows.

**Ask the user:**

```
Module 1 of 6: Agent Purpose

1. What is your agent's display name?
2. In one sentence — what specific job does it do?
3. Who are the primary users? (role or team, e.g. "Sales AEs and SEs")
4. List the top 5 questions these users would ask your agent.
5. What is explicitly out of scope? What should the agent refuse or redirect?
```

**⚠️ MANDATORY STOPPING POINT** — wait for user responses before proceeding.

**After receiving answers**, generate the following block and show it to the user:

```markdown
## Agent Identity

**Name:** {name}
**Purpose:** {purpose}
**Primary Users:** {users}

**Example Questions:**
- {q1}
- {q2}
- {q3}
- {q4}
- {q5}

**Out of Scope:**
- {restriction1}
- {restriction2}
```

**⚠️ MANDATORY CHECKPOINT** — ask: "Does this capture your agent's purpose accurately? Approve or tell me what to change."

Wait for explicit approval. Append approved block to `agent-config.md` under `## Agent Identity`. Then proceed to Module 2 (if full walkthrough) or stop.

---

## Module 2: Tool Mapping

**Key principle:** Your agent needs exactly the tools required to answer its scoped questions — no more. Start from the Snowflake objects you already have, then layer in Cortex Search and custom tools as needed. Inspecting each object now builds the context you'll need for tool descriptions in Module 5.

### Step 2a: Semantic Views (Cortex Analyst)

```
Module 2 of 6: Tool Mapping

Let's start with your structured data.

1. What semantic view(s) will this agent use?
   For each one, provide the fully qualified name: DATABASE.SCHEMA.VIEW_NAME
```

**⚠️ MANDATORY STOPPING POINT** — wait for user response.

**For each semantic view named**, immediately inspect it:

```sql
SELECT GET_DDL('semantic_view', '<DATABASE>.<SCHEMA>.<VIEW_NAME>');
```

Read the output and extract:
- Entities and their source tables
- Dimensions (categorical columns)
- Metrics (numeric measures)
- Time dimensions
- Any relationships between entities

Summarize findings to the user:
```
I found the following in <VIEW_NAME>:
- Entities: {list}
- Key metrics: {list}
- Key dimensions: {list}
- Date range / time dimension: {field}
```

Store this summary — it will be used to pre-populate the tool description in Module 5.

Generate a tool name using the `[Domain][Function]` pattern and confirm with the user:

```
✅ CustomerConsumptionAnalytics   ← domain + function, unambiguous
✅ SalesPipelineAnalytics
❌ DataTool  ❌ Query  ❌ MyView   ← too generic, avoid these
```

### Step 2b: Cortex Search Services (optional)

```
Do you plan to use a Cortex Search service?
(e.g., for searching documents, PDFs, support tickets, knowledge bases)

Yes / No
```

**If Yes:**

```
What is the fully qualified name of your Cortex Search service?
DATABASE.SCHEMA.SERVICE_NAME
```

**⚠️ STOP** — wait for name, then inspect:

```sql
DESCRIBE CORTEX SEARCH SERVICE <DATABASE>.<SCHEMA>.<SERVICE_NAME>;
```

Extract and summarize:
- Source table and indexed column(s)
- Available filter columns
- What content it searches (infer from column names + table name)

Store summary for Module 5. Generate tool name using `[Domain]Search` pattern (e.g., `ProductDocumentationSearch`, `SupportTicketSearch`).

**If No:** skip to Step 2c.

### Step 2c: Custom Tools (optional)

```
Do you plan to use any custom tools?
(e.g., stored procedures, UDFs, or external API lookups)

Yes / No
```

**If Yes:**

```
For each custom tool, provide:
1. The fully qualified procedure or function name
2. What action it performs
```

**⚠️ STOP** — wait for response, then inspect each:

```sql
-- For stored procedures:
DESCRIBE PROCEDURE <DATABASE>.<SCHEMA>.<NAME>(<ARG_TYPES>);

-- For UDFs:
DESCRIBE FUNCTION <DATABASE>.<SCHEMA>.<NAME>(<ARG_TYPES>);
```

Extract signature, argument names/types, and return type. Store for Module 5.

**If No:** skip to Step 2d.

### Step 2d: Coverage Check

Display the tool inventory alongside the 5 example questions from Module 1:

```
Here's your tool inventory:
{table of tools}

Your example questions from Module 1:
{list}

Does each question have a tool that can answer it?
Any gaps — questions no tool currently covers?
```

**⚠️ MANDATORY CHECKPOINT** — ask: "Does this tool inventory look right? Any tools to add, remove, or rename?"

Wait for explicit approval. Append approved block to `agent-config.md` under `## Tool Inventory`:

```markdown
## Tool Inventory

| Tool Name | Type | Covers |
|---|---|---|
| {ToolName} | Cortex Analyst | {summary from inspection} |
| {ToolName} | Cortex Search | {summary from inspection} |
| {ToolName} | Custom Tool | {summary from inspection} |
```

Proceed to Module 3.

---

## Module 3: Orchestration Instructions

**Key principle:** Orchestration = WHAT to do, WHICH tools to call, in what order, under what rules. It does NOT control how answers are formatted — that belongs in Module 4. Mixing the two is one of the most common agent configuration mistakes.

Ask yourself: *Does this instruction affect which tool to select, what data to retrieve, or how to sequence tool calls?* → Orchestration. If it affects formatting, tone, or output structure → that's Module 4.

**Walk through 5 sub-sections. Ask each in sequence.**

### 3a. Identity & Scope

```
Sub-section 3a: Identity & Scope

Write 2-3 sentences that define:
- Your agent's name and role
- Its specific scope (what it covers)
- Its primary users

Example:
"You are 'SalesBot', a sales intelligence assistant for the Snowflake sales team.
You answer questions about customer accounts, pipeline opportunities, deal history,
and product usage. Your users are AEs, SEs, and Sales Leaders."
```

**⚠️ STOP** — capture response.

### 3b. Domain Context

```
Sub-section 3b: Domain Context

What terminology, definitions, or business logic does your agent need to
interpret questions correctly?

Examples to consider:
- Industry-specific terms (e.g., "ARR", "GMV", "DTS")
- Your fiscal year calendar
- Pricing model (e.g., consumption-based, subscription)
- Internal abbreviations or product names
```

**⚠️ STOP** — capture response.

### 3c. Tool Selection Rules

For each tool in the inventory from Module 2, ask:

```
Sub-section 3c: Tool Selection Rules

For each tool, give me 2-3 question patterns that should trigger it.
I'll generate explicit routing rules.

Example output format:
- For questions about CURRENT usage metrics → use CustomerConsumptionAnalytics
  e.g. "What's Acme's credit usage?", "Show active accounts"
- For questions about DOCUMENTATION → use ProductDocumentationSearch
  e.g. "How do Streams work?", "How do I configure SSO?"
```

**⚠️ STOP** — generate routing rules block and show for approval.

### 3d. Boundaries

```
Sub-section 3d: Boundaries & Limitations

What data does your agent NOT have access to?
What questions should it refuse or redirect — and to where?

Generated format:
- You do NOT have access to [X]. If asked, respond: "[redirect message]"
- Your data refreshes [cadence]. Clarify when users ask about real-time data.
- Do NOT [action]. Direct users to [alternative].
```

**⚠️ STOP** — capture response and generate Limitations block.

### 3e. Business Rules

```
Sub-section 3e: Business Rules

Any conditional logic your agent must always follow?

Examples from real agents:
- "Always call CustomerLookup before querying metrics — never assume the ID"
- "If result > 100 rows, aggregate before displaying"
- "For data within last 7 days, warn about 24-hour refresh delay"
- "When multiple regions match, always ask for clarification"
```

**⚠️ STOP** — capture response.

**After all 5 sub-sections**, assemble the full Orchestration Instructions block and show the complete draft. Ask: "Does this orchestration block capture your agent's logic? Approve or edit."

Wait for explicit approval. Append to `agent-config.md` under `## Orchestration Instructions`. Proceed to Module 4.

---

## Module 4: Response Instructions

**Key principle:** Response instructions control HOW answers look — tone, format, structure, and error language. Keep these completely separate from orchestration. A clear test: if removing this instruction wouldn't change which tool gets called, it belongs here.

**Ask the user:**

```
Module 4 of 6: Response Instructions

1. Tone: What communication style fits your users?
   a) Professional & data-driven (analysts, technical users)
   b) Concise & direct (busy sales/ops teams)
   c) Conversational & approachable (general business users)

2. Data format: When showing multi-row results (>3 items), use:
   a) Tables  b) Bullet lists  c) Agent decides based on data type

3. Trends & comparisons: Prefer charts or text summaries?

4. When data is unavailable — what should the agent say?
   (e.g., "I don't have access to X. You can find this in Y or contact Z.")

5. When a question is ambiguous — how should it clarify?
   (e.g., "To give you accurate data, I need clarification: did you mean A or B?")
```

**⚠️ MANDATORY STOPPING POINT** — wait for user responses.

**Generate Response Instructions block:**

```markdown
## Response Instructions

**Style:**
- {tone based on selection}
- Lead with the direct answer, then provide supporting details
- Use active voice and clear statements
- Avoid hedging language ("it seems", "it appears")

**Data Presentation:**
- Use tables for multi-row data (>3 items)
- Use {chart type} for trends and comparisons
- State single values directly without tables
- Always include units (credits, dollars, %) with numbers

**Response Structure:**
- "What is X?" → Direct answer + supporting context
  Example: "{agent-specific example from their purpose}"
- "Show me X" → Brief summary + table/chart + key insights
- "Compare X and Y" → Summary of comparison + visual + differences highlighted

**Error Handling:**
- Unavailable data: "{their message from Q4}"
- Ambiguous query: "{their message from Q5}"
- Empty results: "No results found for [criteria]. This could mean [reason]. Try [alternative]."
```

**⚠️ MANDATORY CHECKPOINT** — show block and ask for approval.

Append approved block to `agent-config.md`. Proceed to Module 5.

---

## Module 5: Tool Descriptions

**Key principle:** Tool descriptions are the #1 cause of agent quality failures. Agents select tools based on name and description alone — they cannot inspect your data model. A vague description cascades into wrong tool calls, bad inputs, and hallucinations. Every description must answer: *What does it do? What data does it cover? When should I call it? When should I NOT call it?*

**For each tool in your inventory, run this prompt (one tool at a time).**

First, display what was learned during Module 2 inspection:

```
Working on: {Tool Name}

From my inspection of {object name}, I found:
{summary captured in Module 2 — entities/metrics/dimensions or indexed columns}

I'll use this to pre-populate the description. A few gaps to fill:
```

Then ask only what the inspection couldn't answer:

```
Module 5 of 6: Tool Description — {Tool Name}

1. Give 3 example questions this tool should handle.
   (I'll use these for the "When to Use" section)

2. Give 3 question types this tool should NOT handle.
   For each: which other tool should handle it instead?

3. Are there required parameter formats the agent needs to know?
   (e.g., customer IDs must come from a lookup first, date format must be ISO 8601)
```

**⚠️ MANDATORY STOPPING POINT** per tool — wait for user responses before generating.

**Generate description block** using inspection output + user answers:

```
Name: {ToolName}

Description: {what it does} + {data it accesses — from inspection summary}

Data Coverage:
- {source table/entity — from inspection}
- {key metrics or indexed columns — from inspection}
- {date range or refresh cadence — from inspection or user}

When to Use:
- {scenario 1 from user answers}
- {scenario 2}
- {scenario 3}

When NOT to Use:
- Do NOT use for {q-type 1} — use {other tool} instead
- Do NOT use for {q-type 2} — use {other tool} instead
- Do NOT use for {q-type 3} — use {other tool} instead

Key Parameters:
- {param_name}: {type} — {format, e.g. "ISO 8601 date (YYYY-MM-DD)"}
  {how to obtain if not obvious, e.g. "Use CustomerLookup tool first if only name is known"}
```

**⚠️ MANDATORY CHECKPOINT** after each tool — ask for approval before moving to the next.

Append each approved tool description to `agent-config.md` under `## Tool Descriptions`. After all tools are done, proceed to Module 6.

---

## Module 6: Deploy & Evaluate

**Key principle:** Agents ship in 3 stages — Prototype → Iterate → Production. The transition from prototype to production is gated by systematic evaluation against a golden test set. Without one, you're guessing at quality.

### 6a. Golden Test Set

Using the 5 example questions from Module 1, generate 5 additional edge-case questions (ambiguous input, multi-tool questions, out-of-scope questions). Build a 10-row test table and save as `golden-test-set.md`:

```markdown
# Golden Test Set — {Agent Name}

| # | Question | Expected Tool(s) | Expected Answer Shape | Notes |
|---|---|---|---|---|
| 1 | {q from module 1} | {tool} | {table / number / summary} | |
| 2 | {q from module 1} | {tool} | | |
| ... | | | | |
| 6 | {edge case: ambiguous} | Clarification | Agent asks for clarification | |
| 7 | {edge case: multi-tool} | {tool1 → tool2} | Chained response | |
| 8 | {edge case: out-of-scope} | None | Redirect with boundary message | |
| 9 | {edge case: data freshness} | {tool} | Answer + freshness caveat | |
| 10 | {edge case: empty result} | {tool} | Empty-result handling message | |
```

**⚠️ STOPPING POINT** — show test set and ask: "Does this golden set represent real user behavior? Add or adjust rows before we finalize."

### 6b. Pre-Launch Checklist

Append to `agent-config.md` under `## Deployment Checklist`:

```markdown
## Deployment Checklist

### Configuration
- [ ] Agent purpose is clearly defined and narrow
- [ ] All tool descriptions include a "When NOT to Use" section
- [ ] Orchestration and Response instructions are in separate fields in the Agent UI
- [ ] Boundaries and redirect language are defined for all out-of-scope queries
- [ ] Tool names follow [Domain][Function] pattern — no generic names like "Tool1" or "DataQuery"
- [ ] Parameter formats are explicit (ISO dates, ID formats, enum values)

### Evaluation
- [ ] Golden test set created (`golden-test-set.md`) with ≥10 questions
- [ ] Golden set includes edge cases: ambiguous, multi-tool, out-of-scope, empty result
- [ ] Agent tested against golden set manually before sharing broadly
- [ ] AI Observability permissions configured (required for monitoring traces)

### Launch
- [ ] Example questions added in the Agent UI (helps users discover the agent)
- [ ] Agent description written for the Snowflake Intelligence Agents tab
- [ ] Designated owner identified for ongoing monitoring
- [ ] Monitoring review scheduled for first 2 weeks post-launch

### Post-Launch
- [ ] Review queries with negative user feedback weekly
- [ ] Use agent traces (Monitoring UI) to diagnose wrong tool calls or slow responses
- [ ] For common questions that fail → add verified queries to your semantic views
- [ ] Rebuild golden test set quarterly or when agent scope changes
```

**Final message** — summarize what was built:

```
Agent config complete.

Agent: {name}
Tools configured: {count} ({list of tool names})
Output files:
  - agent-config.md — full configuration ready for Agent UI
  - golden-test-set.md — {N} test questions for pre-launch evaluation

Next step: Open the Snowflake Agent UI, paste your Orchestration Instructions
and Response Instructions into the respective fields, add your tools, and run
your golden test set to validate before sharing.
```

---

## Stopping Points

- ✋ Module 1 — purpose block approved before proceeding
- ✋ Module 2 — tool inventory confirmed before proceeding
- ✋ Module 3 — after each sub-section (3a–3e), and full block approved before proceeding
- ✋ Module 4 — response instructions approved before proceeding
- ✋ Module 5 — one tool description at a time, approved before next tool
- ✋ Module 6 — golden test set reviewed before generating checklist

**Resume rule:** Upon explicit user approval ("looks good", "approved", "proceed"), continue directly to the next step without re-asking.

---

## Output

| File | Contents |
|------|----------|
| `agent-config.md` | Complete agent configuration: identity, tool inventory, orchestration instructions, response instructions, tool descriptions, deployment checklist |
| `golden-test-set.md` | 10-row evaluation table covering core, edge, and out-of-scope questions |

Both written to the customer's working directory.
