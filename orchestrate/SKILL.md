---
name: orchestrate
description: Plan a complex task, then hand it to subagents in a git worktree the user is not working in. An implementer carries out the plan, a reviewer checks the result, and a fixer addresses the review. Use only when the user asks for this skill by name, or asks to have subagents implement, review, and fix a task. Not for tasks small enough to do directly, for reviewing PRs, or for only writing a plan.
argument-hint: "[plan file or task]"
---

# Orchestrate

You are the planner and orchestrator. Settle the plan, then have three subagents in turn implement it, review the result, and address the review. Leave the code to them: do not implement, review, or fix anything yourself. Nothing gets pushed and no PR gets opened unless the user asks.

## Subagents

| Subagent | Model | Effort | Instructions |
| --- | --- | --- | --- |
| Implementer | Opus 5.5 | max, or xhigh for simpler work (Step 3) | [implementer.md](./implementer.md) |
| Reviewer | Opus 5.5 | xhigh | [reviewer.md](./reviewer.md) |
| Fixer | Opus 5.5 | xhigh | [fixer.md](./fixer.md) |

- Start each subagent fresh, never as a fork of your conversation. It works only from what you give it.
- Set the model (`claude-opus-5-5`) and effort explicitly if the harness lets you, and use a general-purpose agent type, not a lightweight search agent. If the harness cannot run Opus 5.5, use its most capable model at the same effort, and say so in the summary.
- Tell each subagent to read and follow its instructions file, given by its absolute path in this skill's folder. Do not paraphrase the file.
- Run them one at a time. Do not change the worktree while one runs.
- If the harness has no subagents, do each step yourself with the same instructions, and say in the summary that the review was not independent.

## Step 1: Settle the plan

Find the plan, in this order:

1. A plan file named in the arguments, or one written earlier in the session, such as `.plans/<topic>.md`.
2. A plan agreed on earlier in the session, in chat or in plan mode.
3. The task described in the arguments or in the session so far. Draft a plan from it.

Before you draft a plan or fill gaps in one, read the repo's instruction files (CLAUDE.md, AGENTS.md, and CONTRIBUTING.md, whichever exist) and the code the task touches. Look up facts that change over time, such as library versions and API behavior, instead of relying on memory.

The plan is ready when someone who missed the session could carry it out without asking questions. It states:

- The goal and the scope, including what is out of scope.
- The approach, and every decision already made, including the user's decisions from the session.
- Steps that name the files, modules, or commands they touch.
- How to verify the work: tests, checks, and acceptance criteria.

If the plan leaves open a decision only the user can make, such as a product choice or a tradeoff they have not weighed in on, ask before going on. Settle the rest yourself. Otherwise, do not wait for approval of the plan.

If the plan is not in a file, or the session settled decisions its file lacks, write the full plan to `plan.md` in a new temp folder (`mktemp -d`). Leave the user's plan file as it is. Pass the plan to every subagent by its absolute path, so all three read the same text. If the task covers only part of the plan, such as one milestone, name that part for every subagent.

## Step 2: Pick the checkout

If the user said where to work, work there. Otherwise, use the current checkout only when both of these hold:

- It is a linked worktree: `git rev-parse --git-dir` differs from `git rev-parse --git-common-dir`.
- Every commit on its branch that the default branch lacks, and every uncommitted change, came from you in this session. A new branch with neither also counts.

In every other case, create a new worktree. This covers the main checkout, and any branch that another session or the user worked on.

1. Pick a branch name that follows the convention in `git branch -a`.
2. Run `git fetch origin`. Start from the default branch, or from another branch if the plan builds on its work, such as the current branch. If the plan needs uncommitted changes from another checkout, ask the user first.
3. Put the worktree where the repo's instructions say worktrees go. If they say nothing, use `.worktrees/<branch>` at the main checkout's root, the parent of `git rev-parse --path-format=absolute --git-common-dir`. If the path already exists, ask the user what to do with it.

   ```bash
   git worktree add -b <branch> <path> <start point>
   ```

4. Install dependencies the way the repo's docs describe, and copy the ignored files the code needs, such as `.env`, from the main checkout.

Switch the session into the worktree if your harness can. Otherwise run every later command from it.

Record the base commit: `git rev-parse HEAD` in a new worktree. If you made changes for this task earlier in the session, use the commit before them instead, and tell the implementer what they cover. The review covers everything after the base.

If the work is not in a git repo, say so and ask the user how to go on.

## Step 3: Implement

Pick the implementer's effort:

- **max** by default, for long or complex work.
- **xhigh** when the plan is small and well bounded: a few files in one area, following patterns the code already has, no data migrations, concurrency, or security-sensitive changes, and checks that show clearly whether it works.

When unsure, use max.

Give the implementer:

- The worktree path, branch, and base commit.
- The plan's absolute path, and the part to do if not all of it.
- The paths to the repo's instruction files.

When it reports:

- If it stopped, or left steps undone, settle the cause. Update the plan yourself when the fix is clear and stays within the plan's goal. Ask the user when it needs their decision. Then have it go on: send the answer to the same implementer if your harness can resume it, or start a new one with the updated plan and a note on what is already committed.
- Check that `git status --porcelain` prints nothing and that `git log <base>..HEAD` shows the work. If changes are left uncommitted, have the implementer commit them.
- Keep its Deviations and Choices sections for the fixer.

## Step 4: Review

Start the reviewer. Give it:

- The worktree path and base commit.
- The plan's absolute path, and the part in scope.
- The paths to the repo's instruction files.

Do not give it the implementer's report or your own view of the change. It forms its own view from the plan and the code.

When it reports, check each required change:

- Drop it if it does not say how the problem happens.
- Move it to the questions if it does not say how it was reproduced. Failures of the repo's checks that the change caused are exempt.

If no required changes, questions, or minor points remain, skip Step 5.

## Step 5: Address the review

Start the fixer. Give it:

- The worktree path, branch, and base commit.
- The plan's absolute path, and the part in scope.
- The paths to the repo's instruction files.
- The implementer's Deviations and Choices sections. They hold reasons the reviewer did not see.
- The review report, with your changes from Step 4.

When it reports, check that `git status --porcelain` prints nothing and that each Fix and Fix differently item names its commit.

## Step 6: Summary

End with:

- The worktree's path and branch, and the commits since the base (`git log --oneline <base>..HEAD`).
- The implementer's effort, with a one-line reason.
- What was built, measured against the plan: deviations, choices the plan left open, and anything not done.
- One bullet per review item: what the reviewer found, how the fixer handled it, and the commit or the reason. Leave out No action items.
- The final results of the repo's checks.
- Problems the subagents noticed but left alone.
- A question for each Needs input item. When the user answers, have a fixer apply the answers: resume the same one if your harness can, or start a new one with the same inputs plus the answers.
