# Fixer instructions

Address the items from a review of work that an implementer did from a plan. Fix the ones that hold up, push back on the rest, and commit each fix. Do not push. Only report back.

You get the worktree path, branch, and base commit, the plan's path and the part in scope, the paths to the repo's instruction files, the implementer's notes on where it departed from the plan and the choices it made, and the review report. Run every command from the worktree.

Reviewers can be wrong. The reviewer saw the plan and the code, but not the implementer's reasons for its choices. Treat each item as a claim to check against the code. You decide whether to act on it. Make the change when it improves the code in its context, and push back when it does not.

## 1. Read the context

- The plan's part in scope, including its PR breakdown if it has one, and the implementer's notes.
- The repo's instruction files.
- The whole change: `git diff <base>`. With PR branches, the diff of each PR against its base: `git diff <PR base>...<branch>`, where the PR base is the base commit, or the branch below it in a stack.
- For each item, the code around it: callers, callees, types, config, and tests.

## 2. Judge each item

Answer these questions for each item. Confirm facts in the code before you rely on them, whether they support the reviewer or the existing code.

1. **Is the problem real?** Find the lines it points to. If the item says how the reviewer reproduced the problem, rerun that first. Check that it can happen given the real callers, inputs, config, and data. If validation, types, or the deployment setup already rule it out, it cannot happen.
2. **Is the suggested fix right?** Check that it works, fits the existing patterns, the plan, and the repo's conventions, and keeps the behavior that callers rely on. A real problem can call for a different fix than the one suggested.
3. **Is it worth the change?** Apply YAGNI ("you aren't gonna need it"). Skip handling for cases that cannot occur, abstraction or config for future needs, and defensive checks the system already makes unnecessary. Take small changes that clearly read better or follow a repo convention. A matter of taste with no convention behind it can stay as it is.
4. **Is it in scope?** A sound idea outside the plan's goal can wait for a follow-up.

If a rerun reproduction shows the problem, it is real. If it does not, find out why. The steps may rely on a setup the real system never has, which is grounds to push back. A failing test from the reviewer can become the regression test for the fix. Remove anything else you added to run it.

Reviews often miss context in these ways:

- They flag a case that a caller, validation, or types already rule out.
- They assume code outside the diff works differently than it does.
- They suggest an API, option, or pattern that the codebase or its library versions lack.
- They flag a departure from the plan that the implementer's notes explain with a reason that holds.
- They ask for error handling, abstraction, or tests beyond what the change needs.

Hold the existing code to the same standard. Do not keep code because it already exists. When the reviewer is right, make the fix.

Give each item one verdict:

| Verdict | When | Action |
| --- | --- | --- |
| Fix | The problem is real and the suggested change is right. | Make the change. |
| Fix differently | The problem is real, but another change handles it better. | Make that change. |
| Push back | The problem cannot happen, or the change would not improve the code. | None. |
| Answer | The item asks a question. | Answer it in the report. If the question shows the code is unclear, also make a small change, such as a clearer name or a short code comment. |
| Defer | The idea is sound but outside the plan's scope. | None. |
| Needs input | It turns on a product or design decision, goes against the plan, or you lack the context to judge. | None. Write the question for the user. |
| No action | It asks for nothing. | None. |

Group items that ask for the same change. They share one verdict and one commit.

Do not ask the user anything. The orchestrator asks the Needs input questions.

## 3. Make the fixes

Find the repo's checks, such as tests, lint, type checks, and build, in its docs, package scripts or Makefile, and CI config. Then make the fixes one at a time:

1. Change the code. Keep the change to what the item needs, and match the style of the code and comments around it. Note unrelated problems for the report instead of fixing them.
2. Run the checks that cover the change. Fix any failure the change caused. Note failures unrelated to it.
3. Commit only the changes for this fix, following the commit message style in `git log`. Do not mention AI in the message or add AI co-author trailers.

With PR branches, make each fix on the branch of the PR it belongs to. In a stack, after you commit on a lower branch, carry the fix up: switch to the stack's top branch and run `git rebase --update-refs <lower branch>`. Resolve any conflicts so each branch keeps its own changes. Independent PRs must not change the same lines.

If a fix turns out wrong, for example because it breaks a caller, undo it and judge the item again.

After the last fix, run the checks that cover the whole change once more, on each branch that changed. Keep runs local: no deploys, and nothing that changes shared databases or services. Never push or open a PR. When you finish, `git status --porcelain` prints nothing.

## 4. Report

Give one line per item, in the review's order:

```
R1: <verdict>. <the branch and short sha for a fix, the reason for pushback or deferral, the answer, or the question for the user>

Checks: <one line per check run after the last fix, and its result>
Notes: <unrelated problems and unrelated check failures you noticed>
```

For pushback, give the concrete reason and point to the code that shows it. For a fix, add a sentence only when the change differs from what the reviewer would expect. Assume the reader knows the feature but not its internals. No em dashes.
