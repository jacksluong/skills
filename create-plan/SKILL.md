---
name: create-plan
description: Interview the user until every decision is settled, then write a comprehensive Markdown plan document for whatever they describe (a feature, migration, refactor, project, or process) to `.plans/<topic>.md` at the root of the current repo. Use when the user asks to "plan this out", "write a plan", "make a plan doc", "spec this", or wants a design or implementation plan saved in the repo.
disable-model-invocation: true
argument-hint: "What should the plan cover?"
---

# Create plan document

Produce one Markdown file in `.plans/` that a person or agent can pick up cold and execute without asking follow-up questions.

## Step 1: Gather context

Treat the arguments as the plan's subject. If there are none, use the subject discussed so far in the session. If neither exists, ask what the plan should cover.

Before asking anything, learn what you can on your own. Read the repo (README, CLAUDE.md or AGENTS.md, relevant source) and search the web for facts that change over time, such as library versions and API behavior. Dispatch sub-agents for broad searches. Never ask the user for a fact you could look up.

## Step 2: Interview

Interview the user until you reach a shared understanding. Map the plan as a **design tree**: every decision branches into the decisions that hang off it.

Work the tree in **rounds**. The **frontier** is every decision whose prerequisites are already settled. Ask the whole frontier in one round: number each question and give your recommended answer. Then wait for the user's answers before the next round.

Format a round like so:

```
❓ **Q1** - **<question title>**: <question body, including choices when there are any>

➡️ <your recommended answer>

---

❓ **Q2** - **<question title>**: <question body>

➡️ <your recommended answer>
```

After each round, recompute the frontier and ask the next round. A question that depends on another open question belongs to a later round.

Cover at least: goals, scope and non-goals, constraints, approach, sequencing, verification, and risks. Stop when the frontier is empty and the user confirms you share an understanding.

## Step 3: Write the plan

Pick the structure that best fits the subject. The layout below is a starting point. Reorder, merge, rename, drop, or add sections as the content calls for.

```markdown
# <Plan title>

> Status: Draft
>
> Created: <YYYY-MM-DD>

## Summary
<2-4 sentences: what will be built or changed, and the outcome.>

## Goals
- <Measurable outcome>

## Non-goals
- <Explicitly out of scope>

## Background
<Current state, relevant existing code or systems, constraints.>

## Decisions
| Decision | Choice | Alternatives considered |
|---|---|---|

## Design
<How it works: components, data flow, interfaces, schemas. Add a mermaid diagram when a flow or structure is easier to see than read.>

## Implementation steps
### Phase 1: <name>
1. <Concrete step, naming the files, modules, or systems it touches>

## Verification
<Tests, checks, and acceptance criteria that prove each phase works.>

## Rollout
<Deployment, migration, feature flags, rollback.>

## Risks
| Risk | Impact | Mitigation |
|---|---|---|

## Open questions
- <Anything the user chose to defer>
```

Writing guidance:

- **Self-contained.** A reader who missed the interview understands every section. Define project-specific terms on first use.
- **Concrete.** Name real files, functions, commands, and versions from the repo. Each implementation step is small enough to do and verify on its own.
- **Final decisions only.** Record what was decided, not the back-and-forth of the interview.
- **Accurate.** Every file path, API, and command in the plan exists or is explicitly marked as new.

## Step 4: Save and deliver

- Location: `.plans/` at the repo root (`git rev-parse --show-toplevel`). Outside a git repo, use the current directory. Create `.plans/` if it doesn't exist.
- File name: the topic in lowercase kebab-case, for example `.plans/auth-session-migration.md`.
- If a file with that name exists, ask whether to update it or pick a new name.
- In the reply, give the file path and one line on what the plan covers. Don't paste the plan into the chat.
