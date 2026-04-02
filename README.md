# i-got-skills

A collection of interactive skill files for [Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) — Snowflake's AI coding assistant. Each skill walks you through a specific workflow step-by-step, asks you the right questions, and generates real Snowflake artifacts as output.

---

## What are skills?

Skills are markdown files that give Cortex Code structured instructions for a specific task. Instead of prompting from scratch, a skill provides:

- Guided questions that gather your context
- Step-by-step workflows with mandatory checkpoints
- Generated output files you can use immediately (SQL, YAML, config files)

---

## How to use a skill

### Option A: Snowsight (no setup required)

1. Open [Snowsight](https://app.snowflake.com) → **Projects** → **Workspaces**
2. Create or open a Workspace
3. Open the Cortex Code chat panel
4. Copy the contents of a skill file and paste it into the chat
5. Cortex Code will start the workflow — follow the prompts

### Option B: Cortex Code CLI

1. Install the [Cortex Code CLI](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code-cli)
2. Copy the skill file into your skills directory:
   ```
   ~/.snowflake/cortex/skills/<skill-name>/SKILL.md
   ```
3. Start a session:
   ```bash
   cortex code
   ```
4. Reference the skill by name or trigger phrase — Cortex Code auto-loads it based on your request

> **Tip:** Rename the `.md` file to `SKILL.md` when placing it in your skills directory. The skill's `name` and `description` frontmatter controls when it auto-triggers.

---

## Skills

### `cortex-agents/`

Skills for building and operating Cortex Agents in Snowflake Intelligence. Run these in sequence for end-to-end agent deployment, or use individually.

| File | What it does |
|------|-------------|
| [`build-semantic-view.md`](cortex-agents/build-semantic-view.md) | Build a Snowflake Semantic View using the `dbt_semantic_view` package. Handles permissions, table introspection, schema generation, and deployment — no local dbt install required. |
| [`create-cortex-agent.md`](cortex-agents/create-cortex-agent.md) | Create a Cortex Agent end-to-end. Gathers your semantic view, designs orchestration and response instructions, configures tools, deploys to Snowflake Intelligence, and validates with test questions. |
| [`cortex-agent-best-practice.md`](cortex-agents/cortex-agent-best-practice.md) | Interactive best practices guide. Walks through agent purpose, tool mapping, orchestration instructions, response instructions, tool descriptions, and deployment readiness. Generates a complete `agent-config.md` and golden test set. |
| [`evaluate-cortex-agent.md`](cortex-agents/evaluate-cortex-agent.md) | Set up and run Cortex Agent Evaluations. Handles permissions, evaluation dataset creation, YAML configuration, execution, result inspection, and iterative improvement. |

**Recommended sequence for new agents:**

```
build-semantic-view  →  create-cortex-agent  →  cortex-agent-best-practice  →  evaluate-cortex-agent
       ↑                        ↑                           ↑                           ↑
  Build the              Create the agent            Harden your                  Measure and
  data layer             and deploy it               instructions                 improve quality
```

---

## Contributing

Skills follow a standard format. Each file should include:

```yaml
---
name: skill-name
description: "What it does + trigger phrases"
---
```

Keep skills under 500 lines. Use `⚠️ MANDATORY STOPPING POINT` to gate user approval before generating or modifying anything.

---

## License

See [LICENSE](LICENSE).
