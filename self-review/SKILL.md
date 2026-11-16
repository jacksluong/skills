---
name: self-review
description: Review your own work before its last commit with four independent reviewer agents, one per lens. Use only when the user or the repo's instructions ask for this review, by name or as an independent or multi-agent review of your own work. Not for reviewing PRs or other people's code, or for a quick check. The lenses are plan fidelity and simplicity; correctness and determinism; security, least privilege, and private data; and naming and writing. Verifies and fixes the high and medium findings, records them in a review report, and lists the low ones in a separate file for the handoff.
argument-hint: "[what to review]"
---

# Self-review

Have four independent reviewers check your work before its last commit, one per lens. Verify what they find, fix what holds up, and record the results.

## Step 1: Gather the sources

Treat the arguments as the scope of the work, such as a milestone or a feature. If there are none, review the work you did in this session. If you did none, ask what to review.

Collect what the reviewers need:

1. **The diff.** Find the base: the last commit before the work started, such as the previous milestone's last commit or the commit the branch started from. `git diff <base>` shows every change since then, committed or not, except new files that git does not track yet. List those with `git ls-files --others --exclude-standard`.
2. **The requirements.** The items in the plan, checklist, spec, or issue that this work covers, such as one milestone's rows of a checklist. Leave out items for other work. If nothing is written down, quote the user's requests from the session.
3. **The design.** The sections of the design or architecture docs that describe what this work builds, if the repo has them.
4. **The conventions.** The repo's instruction files, such as CLAUDE.md, AGENTS.md, and CONTRIBUTING.md.

## Step 2: Run the reviewers

Start four reviewer subagents in parallel, one per lens. Run each on the same model as yours. If the harness lets you pick a model, set it explicitly instead of relying on a default, and use a general-purpose agent type, not a lightweight search agent. If your harness has no subagents, run the lenses yourself one at a time, and say in the report that the reviews were not independent.

Give each reviewer:

- Its lens, from the list below.
- The base commit and the list of new files.
- The requirements and design sections, as file paths with section names, or quoted.
- The paths to the repo's instruction files.
- The reviewer instructions below.

Do not give them your conclusions: no summary of the change, no reasons for your choices, and no guesses about what might be wrong. Each reviewer forms its own view from the sources.

Do not change any files until all four have reported.

### Lenses

1. **Plan fidelity and simplicity.** The change does everything its requirements ask, the way the design describes, and nothing more. Look for missing or partial items, departures from the design, and work nobody asked for. Then look for a simpler way to get the same result: less code, fewer moving parts, reuse of what the repo already has, and no abstraction, config, or handling for cases that cannot happen today.
2. **Correctness and determinism.** The code does the right thing for every input and state that can reach it. Check edge cases (empty, missing, very large, malformed, or duplicate values), error paths, partial failures, concurrency, and data written by older code. The same inputs always give the same result: look for hidden dependence on time, randomness, the order of unordered collections, the environment, or test order, and for flaky tests.
3. **Security, least privilege, and private data.** Check auth and permission checks, untrusted input that reaches queries, commands, file paths, or output, and secrets in code, config, or logs. Each process, token, role, and dependency gets only the access it needs. Private data, such as credentials and personal data, stays out of logs, error messages, responses, and third-party services.
4. **Naming and writing.** Names of files, functions, variables, types, and settings say what they hold or do, stay consistent with each other, and follow the repo's conventions. Comments, docs, log and error messages, and user-facing text are accurate, plain, and match the code after the change. Docs that the change affects are updated.

### Reviewer instructions

Review the change through your lens only. Do not change any files in the repo. Only report back.

1. **Read the sources.** Read the requirements, the design sections, and the repo's instruction files. Then read the diff (`git diff <base>`), the new files, and each changed file in full. Trace the code around each change: callers, callees, types, config, and tests.
2. **Look for problems** in your lens that the change introduces or makes worse. Follow every suspicion until you confirm it or rule it out. You may run the repo's tests, or a short script you write in a temp folder outside the repo.
3. **Try to disprove each finding.** Look for a caller, check, type, test, or config that rules it out, and for a reading of the requirements under which the code is right. Drop the findings that do not survive.
4. **Rate** each finding that survives:
   - **High:** breaks a requirement, gives wrong results in normal use, loses or exposes data, or opens a security hole.
   - **Medium:** fails in a plausible edge case, departs from the design or the repo's conventions, adds complexity the requirements do not need, or misleads through a name, comment, or doc.
   - **Low:** anything smaller, such as style, wording, or a small cleanup.
5. **Report.** For each high and medium finding, give the file and line, what goes wrong and when, the evidence, and what you checked while trying to disprove it. The evidence is a test or script you ran, with its code and what it showed, or a trace through the code that names each file and line. List low findings one line each, with the file and line. If you found nothing at a level, say so.

## Step 3: Verify and fix

Merge findings that more than one reviewer reported. Then verify each high and medium finding yourself:

- Find the lines it points to, and rerun the reviewer's test or script if there is one.
- Check that it can happen given the real callers, inputs, config, and data.
- Hold your own code to the same standard as anyone else's. Do not dismiss a finding because you wrote the code or chose otherwise earlier.

Fix every finding you verify. Keep each fix to what the finding needs, and apply YAGNI ("you aren't gonna need it"): no handling for cases that cannot happen and no abstraction for future needs. Run the repo's checks that cover each fix, such as tests, lint, and type checks. If a fix needs a decision only the user can make, such as a change to the plan, do not guess. Leave it unfixed and mark it as needing input.

Do not verify or fix low findings. They go into the low report as written.

## Step 4: Second round

If round one changed any code, repeat Steps 2 and 3 with four new reviewers, and give them the updated list of new files. Never run a third round. Fix what round two verifies, and note in the report that those fixes were not reviewed again.

A finding you rejected in round one stays rejected if a reviewer raises it again without new evidence.

## Step 5: Write the reports

Save both reports where the repo's instructions say review reports go. If they say nothing, use `.reviews/` at the repo root (`git rev-parse --show-toplevel`). Name them `<name>-review.md` and `<name>-review-low.md`, where `<name>` is the work in lowercase kebab-case, for example `.reviews/m3-review.md`. If a report already exists, add this review at the end instead of replacing it.

`<name>-review.md` lists every high and medium finding from both rounds and what was done about it:

```markdown
# Review: <work>

> Date: <YYYY-MM-DD>
>
> Base: <short sha>
>
> Rounds: <1 or 2>

## <High | Medium>: <one-line title>

- Round <n>, <lens>, `<file>:<line>`
- Finding: <what goes wrong and when>
- Evidence: <how the reviewer showed it and how you verified it>
- Outcome: <Fixed: what changed. | Rejected: why it does not hold up. | Needs input: the question for the user.>
```

`<name>-review-low.md` lists the low findings for whoever picks up the work next, one line each with the lens, file, and line. If you write a handoff document, link it there.

Do not commit as part of this review. The fixes and reports go into the work's last commit.

## Step 6: Summary

End with a short summary: how many high, medium, and low findings came up, one line per high and medium finding with its outcome, the questions for any findings that need the user's input, and the paths to both reports.
