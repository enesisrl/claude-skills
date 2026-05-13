---
name: dev-plan
description: >
  Produce a structured development plan (Markdown) for a feature, refactor, bug fix, migration,
  new project, or any other coding initiative, and save it under .claude/dev_plans/ in the current
  project. Use this skill whenever the user asks for a development plan, an implementation plan,
  a roadmap, a technical plan, a step-by-step plan, or a "how would you build this" — phrases like
  "piano di sviluppo", "piano di implementazione", "piano tecnico", "fammi un piano per",
  "preparami un piano", "come implementeresti", "come svilupperesti", "roadmap tecnica",
  "piano dettagliato", "scrivi un piano per", "implementation plan", "development plan",
  "technical plan", "technical roadmap", "step-by-step plan", "how would you implement",
  "draft a plan for", "plan the work for", "outline the implementation". Always use this skill
  when the user asks for any kind of development/implementation plan, even if they don't
  explicitly say "piano" or "plan" — intent to plan technical work is enough to trigger it.
  By default, saves the plan as a Markdown file under .claude/dev_plans/ in the current project.
---

# Dev Plan Skill

When the user asks for a development plan (a feature, refactor, bug fix, migration, new project, etc.), follow this workflow.

## Output location

**Default behavior** — unless the user specifies otherwise, save the plan here:

```
<project-root>/.claude/dev_plans/<descriptive-name>.md
```

Where `<descriptive-name>` reflects what is being planned (e.g., `auth-rewrite.md`, `payments-integration.md`, `migration-postgres-15.md`, `fix-race-condition-queue.md`). Create the `.claude/dev_plans/` directory if it doesn't exist.

**Override** — if the user provides an explicit output path or format (e.g., "save it in docs/", "write it to plan.md", "just show it here", "non serve salvarlo"), use that instead.

## Planning workflow

1. **Understand the goal** — clarify what the user wants to build/change and why. If the request is ambiguous (scope, target users, must-have vs. nice-to-have, deadlines, constraints), ask focused questions BEFORE writing the plan.
2. **Explore the relevant context** — if the plan touches existing code, scan the repo: project structure, key modules, conventions, frameworks in use. Use `find`/`ls`, read key files (entry points, configs, the modules that will be touched). For a brand-new project, skip this step.
3. **Identify constraints & dependencies** — runtime, language version, framework, infra, external services, performance/security/compliance requirements, team conventions, deadlines.
4. **Design the approach** — pick an architectural direction. If there are real trade-offs, write down 1–2 alternatives and explain why you chose one.
5. **Decompose into phases & tasks** — break the work into ordered, shippable phases. Each phase should have concrete tasks, clear deliverables, and (where useful) a rough estimate.
6. **Write the plan** — follow the structure below.
7. **Save it** — write the `.md` file to the default location (or user-specified location), then share a link to it.

## Plan structure

Use this template as a starting point. Adapt section depth to the size of the initiative — a single bug fix warrants a lighter plan than a full feature or migration.

```markdown
# Dev Plan: [Name of the initiative]

**Date**: YYYY-MM-DD
**Author**: [user / team]
**Status**: Draft | In review | Approved | In progress | Done
**Target paths**: `path/to/folder`, `path/to/other` (if applicable)

## 1. Goal & context

What we're building or changing, and why. Business motivation, user need, or technical driver
behind the work. Keep it short — 1–2 paragraphs.

## 2. Scope

**In scope**: what this plan covers.
**Out of scope**: what is explicitly NOT covered (so expectations are clear).

## 3. Requirements

Functional and non-functional requirements: what the result must do, performance targets,
security/compliance constraints, accessibility, etc.

## 4. Current state (if applicable)

How things work today and where the gaps/problems are. Cite file paths or modules where relevant.
Skip this section for greenfield work.

## 5. Proposed approach

The chosen architectural/technical direction. Include a brief diagram or component breakdown if
useful. Explain the key design decisions.

### Alternatives considered

If there are meaningful trade-offs, list 1–2 alternatives and why they were rejected.

## 6. Implementation plan

Break the work into ordered phases. Each phase should be independently shippable when possible.

### Phase 1 — [name]

**Goal**: what this phase achieves.

**Tasks**:

- [ ] Task A — short description (touches `path/to/file`)
- [ ] Task B — short description

**Deliverable**: what exists at the end of the phase.

**Estimate**: rough size (e.g., S / M / L, or days).

### Phase 2 — [name]

…

## 7. Risks & mitigations

Things that could go wrong (technical risk, unknowns, external dependencies, data migration
hazards) and how we plan to address them.

## 8. Testing & validation

How we'll verify the work: unit tests, integration tests, manual QA, staging rollout, metrics
to monitor, rollback plan.

## 9. Open questions

Things still to be decided or that need input from someone else. List them so they don't get lost.

## 10. References

Links to related issues, PRs, design docs, prior analyses (e.g., a `.claude/analysis/*.md` file),
external documentation.
```

## A few tips

- **Adjust depth to scope**: a small bug fix needs a 1-page plan (Goal, Approach, Tasks, Validation). A multi-month migration deserves the full template.
- **Be concrete**: name files, modules, and APIs that will be touched. Vague plans are useless plans.
- **Order matters**: phases should be sequenced so each one leaves the codebase in a working, shippable state when possible.
- **Surface unknowns**: if a step depends on a decision that hasn't been made, put it in "Open questions" rather than inventing an answer.
- **Cross-link with analysis**: if there is a relevant report under `.claude/analysis/`, reference it in section 10 — it gives the plan grounded context.
- **Match the user's language**: if the user writes in Italian, write the plan in Italian; if in English, write it in English. Keep the section headings consistent with the chosen language.
- **After saving**, always share the file link so the user can open it directly.
