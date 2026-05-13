# dev-plan — Claude Code Plugin

A Claude Code skill that automatically produces structured Markdown development plans.

## What it does

When you ask Claude for a development plan (feature, refactor, bug fix, migration, new project, etc.), this skill instructs it to:

- Save the plan to `.claude/dev_plans/<descriptive-name>.md` inside your project by default
- Follow a consistent plan structure: Goal & context, Scope, Requirements, Current state, Proposed approach, Implementation plan (phases & tasks), Risks, Testing, Open questions, References
- Ask focused clarifying questions before writing when the scope is ambiguous
- Respect any custom output path you specify (e.g. "save it in docs/plan.md")

## Installation

### 1. Add the marketplace

```bash
claude plugin marketplace add github:emanueletoffolon/claude-skills
```

### 2. Install the plugin

```bash
claude plugin install dev-plan
```

## Usage

Just ask Claude naturally — no slash command needed:

- "Fammi un piano di sviluppo per la nuova area pagamenti"
- "Draft an implementation plan for migrating to Postgres 15"
- "Piano dettagliato per risolvere la race condition sulla coda"
- "How would you implement multi-tenant auth here?"
- "Piano per rifare il modulo di reportistica, salvalo in docs/reporting-plan.md"

The skill triggers automatically. The plan lands in `.claude/dev_plans/` unless you say otherwise.

## Update

```bash
claude plugin update dev-plan
```

## Plugin structure

```
dev-plan-plugin/
  .claude-plugin/
    plugin.json
  skills/
    dev-plan/
      SKILL.md
```
