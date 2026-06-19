# code-analysis — Claude Code Plugin

A Claude Code skill that automatically produces structured Markdown code analysis reports.

## What it does

When you ask Claude to analyze code in a folder or file, this skill instructs it to:

- Save the report to `.claude/analysis/<descriptive-name>.md` inside your project by default
- Follow a consistent report structure: Overview, Architecture, Key Components, Dependencies, Code Quality, Summary
- Respect any custom output path you specify (e.g. "save it in docs/review.md")

## Installation

### 1. Add the marketplace

```bash
claude plugin marketplace add github:enesisrl/claude-skills
```

### 2. Install the plugin

```bash
claude plugin install code-analysis
```

## Usage

Just ask Claude naturally — no slash command needed:

- "Analizza il codice in src/"
- "Analyze the auth module and focus on security"
- "Give me a full code analysis of this project"
- "Analyze src/utils/ and save it to docs/review.md"

The skill triggers automatically. The report lands in `.claude/analysis/` unless you say otherwise.

## Update

```bash
claude plugin update code-analysis
```

## Plugin structure

```
code-analysis-plugin/
  .claude-plugin/
    plugin.json
  skills/
    code-analysis/
      SKILL.md
```
