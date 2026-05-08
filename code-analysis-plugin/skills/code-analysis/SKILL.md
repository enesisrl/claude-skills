---
name: code-analysis
description: >
  Analyze code in one or more folders or files and produce a structured Markdown report.
  Use this skill whenever the user asks to analyze, review, audit, inspect, or understand code
  — phrases like "analizza questa cartella", "analyze this folder", "review my code",
  "fammi un'analisi del codice", "give me a code analysis", "look at the code in X",
  "analizza la struttura di", "code review of", "audit the codebase", "cosa fa questo codice",
  "what does this code do", "analyze src/", "analizza il progetto".
  Always use this skill when the user asks for any kind of code analysis, even if they don't
  explicitly say "analysis" — intent to understand or review code is enough to trigger it.
  By default, saves the report as a Markdown file under .claude/analysis/ in the current project.
---

# Code Analysis Skill

When the user asks to analyze code in one or more folders or files, follow this workflow.

## Output location

**Default behavior** — unless the user specifies otherwise, save the report here:

```
<project-root>/.claude/analysis/<descriptive-name>.md
```

Where `<descriptive-name>` reflects what was analyzed (e.g., `auth-module.md`, `full-project.md`, `api-layer.md`). Create the `.claude/analysis/` directory if it doesn't exist.

**Override** — if the user provides an explicit output path or format (e.g., "save it in docs/", "write it to analysis.md", "just show it here"), use that instead.

## Analysis workflow

1. **Identify targets** — determine which folders or files the user wants analyzed.
2. **Explore the structure** — scan the file tree, identify languages, entry points, and main modules (a quick `find` or `ls -R` helps).
3. **Read the code** — read the relevant source files. For large projects, focus on key files: entry points, core modules, config files, and any file the user specifically mentioned.
4. **Write the report** — follow the structure below.
5. **Save it** — write the `.md` file to the default location (or user-specified location), then share a link to it.

## Report structure

Use this template as a starting point. Adapt section depth and detail to the size of what's being analyzed — a single module warrants a lighter report than a full codebase.

```markdown
# Code Analysis: [Name of what was analyzed]

**Date**: YYYY-MM-DD
**Analyzed paths**: `path/to/folder`, `path/to/other`

## Overview

What this code does and its purpose in one or two paragraphs.

## Architecture & structure

How the code is organized: folder layout, main modules, and how they relate to each other.

## Key components

For each significant module or area:
- **Purpose**: what it does
- **Key files**: the most important files
- **Notable patterns**: design patterns, idioms, or techniques used

## Dependencies

External libraries, frameworks, or tools the code relies on, and why they're used.

## Code quality observations

Strengths worth keeping, areas that could be improved, potential bugs or risks, and any
technical debt worth noting. Be specific — cite file names and line numbers where relevant.

## Summary & recommendations

Two or three concise takeaways, and any suggested next steps if applicable.
```

## A few tips

- **Adjust depth to scope**: a single utility file needs a short, focused report; a full application deserves the full template.
- **Focus on what's asked**: if the user says "analyze the auth module for security issues", concentrate the report on that angle rather than doing a general survey.
- **Multiple folders → one report** by default. If the scope is very wide and it makes sense to split (e.g., separate frontend and backend), ask the user before splitting.
- **After saving**, always share the file link so the user can open it directly.
