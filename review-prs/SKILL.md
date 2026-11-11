---
name: review-prs
description: Review every open PR in the current repo (or the PR numbers given), approve the ones that are correct with a brief comment, and check with the user before requesting changes on any of them. Blocks on any conflict or duplicate work between open PRs, and accounts for auto-merge on PRs that have it enabled.
disable-model-invocation: true
argument-hint: "[PR numbers]"
---

# Review PRs

Review open PRs, approve the correct ones, and run every non-approval by the user before posting it.

## Step 1: Collect PRs

Run everything against the current repo. Get the user's login with `ME=$(gh api user -q .login)`.

Fetch all open PRs in one GraphQL call (`gh` fills in `{owner}` and `{repo}` from the current repo):

```bash
gh api graphql -F owner='{owner}' -F name='{repo}' -f query='
query($owner: String!, $name: String!) {
  repository(owner: $owner, name: $name) {
    pullRequests(states: OPEN, first: 100, orderBy: {field: CREATED_AT, direction: DESC}) {
      nodes {
        number title url isDraft baseRefName headRefOid reviewDecision authorAssociation
        author { login }
        autoMergeRequest { enabledAt }
        files(first: 100) { nodes { path } }
        commits(last: 20) { nodes { commit { oid committedDate parents { totalCount } } } }
        reviews(last: 50) { nodes { id author { __typename login } state commit { oid } } }
        comments(last: 50) { nodes { author { login } createdAt } }
        timelineItems(itemTypes: [REVIEW_DISMISSED_EVENT], last: 50) {
          nodes { ... on ReviewDismissedEvent { previousReviewState review { id } } }
        }
      }
    }
  }
}'
```

Build a compact roster with `jq`:

```bash
jq --arg me "$ME" '.data.repository.pullRequests.nodes | map(. as $pr
  | [.commits.nodes[].commit] as $commits
  | (reduce (.timelineItems.nodes[] | select(.review != null)) as $e ({};
      .[$e.review.id] = $e.previousReviewState)) as $wasState
  | [.reviews.nodes[] | select(.author.__typename != "Bot")
      | .was = (if .state == "DISMISSED" then $wasState[.id] else .state end)] as $reviews
  | ([$reviews[] | select(.was == "APPROVED" or .was == "CHANGES_REQUESTED")] | last) as $prior
  | ($commits | map(.oid) | index($prior.commit.oid)) as $priorAt
  | (if $priorAt != null and $priorAt < ($commits | length) - 1
      then $commits[$priorAt + 1:] else null end) as $newCommits
  | (.reviewDecision == "REVIEW_REQUIRED" and any($reviews[];
      .state == "DISMISSED" and .was == "APPROVED" and .commit.oid == $pr.headRefOid)) as $reapprove
  | {
  number, title, url, author: .author.login, base: .baseRefName,
  headOid: .headRefOid, autoMerge: (.autoMergeRequest != null),
  trusted: (.authorAssociation | IN("OWNER", "MEMBER", "COLLABORATOR")),
  files: [.files.nodes[].path],
  review: (if $reapprove then "re-approve"
    elif $newCommits == null then "full"
    elif $prior.was == "APPROVED" and all($newCommits[]; .parents.totalCount > 1) then "merge-only"
    else "follow-up" end),
  since: (if $newCommits == null then null else {oid: $prior.commit.oid, state: $prior.was} end),
  skip: (if .isDraft then "draft"
    elif .author.login == $me then "own"
    elif $reapprove then null
    elif any($reviews[]; .author.login == $me and .state != "DISMISSED" and .commit.oid == $pr.headRefOid)
      or any(.comments.nodes[]; .author.login == $me and .createdAt > $commits[-1].committedDate)
      then "responded"
    elif any($reviews[]; .commit.oid == $pr.headRefOid
      and (.state == "APPROVED" or .state == "CHANGES_REQUESTED")) then "reviewed"
    else null end)
})'
```

A PR is **eligible** when `skip` is null. That excludes:

- Drafts.
- The user's own PRs.
- PRs the user already responded to since the last commit: any review by the user at the current head that was not dismissed, or a PR comment by the user posted after the head commit.
- PRs that a person already approved or blocked at the current head commit. Bot reviews do not count.

`review` sets how deep Step 3 goes. It looks at the latest approval or change request by a person, counting reviews that were later dismissed as what they were before dismissal:

- `re-approve`: a person's approval at the current head was dismissed, and the PR still needs an approval. This happens when the repo dismisses approvals after the base branch moves. There is no new code to review, so it skips Step 3.
- `full`: no such review, or its commit is no longer in the PR (for example after a rebase) or not among the last 20 commits.
- `follow-up`: the PR has new commits since that review. `since` holds the reviewed commit and the review's state.
- `merge-only`: that review was an approval, and every commit since is a merge.

`trusted` is true when the PR's author is an owner, member, or collaborator of the repo. Reviewers run a PR's code only when it is trusted, since code from anyone else may be unsafe to run.

If the query returns exactly 100 PRs, there may be more. Page through the rest with `pageInfo { hasNextPage endCursor }` and an `after:` cursor. If a PR lists exactly 100 files, its list may be cut off. Fetch the full list with `gh pr view <n> --json files`.

If the user passed PR numbers, review only those, but keep the full roster for the overlap check. If a requested PR is not eligible, say why and skip it.

If there are no eligible PRs, say so and stop. If there are more than 20, list them (number, title, author) and ask the user to confirm or narrow the set before going on.

## Step 2: Shared context

Do this once in the main agent so the reviewers do not each repeat it.

1. **Fetch PR heads.** Run `git fetch origin <base>` for each base branch, then `git fetch origin refs/pull/<n>/head ...` for every eligible PR in a single command. This downloads the commits without touching the working tree or creating branches. Reviewers then read the code with `git show <headOid>:<path>` and `git diff origin/<base>...<headOid>`.
2. **Repo conventions.** Read CLAUDE.md, AGENTS.md, and CONTRIBUTING.md at the repo root (whichever exist on the base branch). Condense the rules that matter for code review into at most 20 bullets.
3. **Overlap map.** Find every pair of open, non-draft PRs (eligible or not) that touch the same files:

   ```bash
   jq -c '[.[] | select(.skip != "draft")] | . as $all
     | [range(length) as $i | range($i+1; length) as $j
       | ($all[$i].files - ($all[$i].files - $all[$j].files)) as $s
       | select($s | length > 0)
       | {a: $all[$i].number, b: $all[$j].number, shared: $s}]'
   ```

   This uses file paths only. No diffs are fetched at this stage.

## Step 3: Review each PR

Start one subagent per eligible PR except `re-approve` PRs, all in parallel. Pass the full review instructions for `full` PRs and the follow-up review instructions for `follow-up` and `merge-only` PRs. Run every subagent on the same model as yours. If the harness lets you pick a model, set it explicitly instead of relying on a default, and use a general-purpose agent type, not a lightweight search agent. If your harness has no subagents, review the PRs one at a time with the same instructions.

Give each subagent:

- The PR number, URL, base branch, head commit, whether auto-merge is enabled, and `trusted`.
- For follow-up reviews: the review type and `since`.
- The condensed conventions from Step 2.
- The roster of all other open, non-draft PRs as one line each: number, title, author, file count. This lets it spot duplicate work across PRs, even when they share no files.
- Its entries from the overlap map: which PRs share which files.
- The matching review instructions below, and the comment guidance in "Write comments people can understand".

### Full review instructions

Do not post anything to GitHub. Only report back.

1. **Read the intent.** `gh pr view <n> --json title,body,closingIssuesReferences,comments,reviews`. Read the linked issues. Note points earlier reviewers already made so you do not repeat them.
2. **Read the change in depth.** Diff the head against the base, then read each changed file in full at the head commit. Trace the code around every change: callers, callees, types, config, schemas, and how inputs reach the changed code. Take the time a careful senior reviewer would. Do not check out the branch. Run code only as step 5 describes.

   For each change, check at least the following:
   - Inputs: empty, missing, very large, malformed, or duplicate values that can reach it.
   - Error paths: what happens when a call fails, times out, or is retried, and whether a partial failure leaves bad state behind.
   - Concurrency: two requests, jobs, or users hitting the same code or data at once.
   - Existing data: whether records, caches, or messages written by the old code still work with the new code.
   - Callers: code that relies on the old behavior, return shape, or side effects.
   - Security: auth and permission checks, untrusted input reaching queries, commands, or output, and secrets.
   - Completeness: whether the change does everything the PR description and linked issues say.

   The bar in step 6 and YAGNI decide what blocks and what goes in the comment. They do not limit how hard you look. Follow every suspicion until you confirm it or rule it out.
3. **Check CI.** Run `gh pr checks <n>`. If a check fails, read only the tail of its log (`gh run view <run-id> --log-failed | tail -100`) and decide whether the PR's own changes caused it. Failures unrelated to the PR do not block.
4. **Check overlap.** For each overlapping PR, fetch only the shared files' patches: `gh api repos/{owner}/{repo}/pulls/<other>/files --paginate --jq '.[] | select(.filename == "<path>") | .patch'`. Classify:
   - **Independent:** the PRs change unrelated parts of the shared files, and neither change affects the logic the other touches. Ignore.
   - **Expected merge conflict:** the PRs change the same lines, so git will report a conflict for whichever merges second, but they do separate things and neither change affects the logic the other relies on. Do not block.
   - **Conflict:** the PRs change the same logic, or one change affects code the other relies on. Block, whether or not git can merge both cleanly.
   - **Duplicate work:** both PRs solve the same problem. Block.

   Also check the roster for duplicate work in PRs that share no files, using their titles. Fetch a candidate's description with `gh pr view <other> --json body` only when the title suggests the same goal.

   Read both changes until you can classify the pair. If you still cannot tell, mark the PR unsure and say what is unclear.
5. **Reproduce.** Before a finding can block, show that it happens. Do this for every finding that passes the rest of the bar in step 6. CI failures and overlap findings already carry their evidence, so they skip this step.
   - **Run it** when `trusted` is true and a run is cheap. Export the files at the head commit to a new temp folder with `dir=$(mktemp -d) && git archive <headOid> | tar -x -C "$dir"`, set up only what the run needs, and run a focused test or a short script there that shows the problem. Never run code in the user's working tree. Keep the run local: no deploys, migrations, or calls to shared services. Delete the folder when you are done.
   - **Trace it** when the PR is not trusted, or when setup would take more than a few minutes. Pick a concrete input and follow it from where it enters the system (a request, a command, a job) to the failure, naming the file and line at each step.

   Note how you reproduced each finding, since the comment includes it.
6. **Decide.** Block only on findings you are highly confident in. A finding blocks only if all of these hold:
   - You confirmed it in the code and can point to the lines.
   - You reproduced it in step 5, unless it is a CI failure or an overlap finding.
   - It can happen given the current state of the system: the real callers, inputs, config, and data. If upstream validation, types, or the deployment setup already rule it out, it cannot happen.
   - It has a meaningful effect when it happens.

   Apply YAGNI ("you aren't gonna need it"): do not ask for handling of cases that cannot occur today, extra abstraction or configurability for future needs, or defensive checks the system already makes unnecessary. A minor issue that is very unlikely to happen does not block.

   - **Request changes** for findings that pass the bar above: bugs, security issues, data loss, broken behavior, missing parts of what the PR says it does, CI failures caused by the PR, any conflict or duplicate work with another open PR, and significant departures from the repo conventions.
   - **Never block on:** style, naming, small cleanups, missing tests, minor convention slips, or anything that fails the bar above.
   - **Unsure** when you cannot judge correctness without context you do not have, or when the only findings that would block are ones you could not reproduce.
   - **Approve** otherwise.

   A finding you could not reproduce does not block. Ask the author about it instead. If the PR also has findings that block, request changes and add the questions after the required changes.
7. **Report** in this format:

   ```
   PR: #<n> <title>
   Summary: <one sentence on what the PR does and why, for the user only>
   Verdict: approve | request-changes | unsure
   Head: <headOid>
   CI: <passing | failing (caused by PR) | failing (unrelated) | pending>
   Overlap: <none, or one line per overlapping PR with its class>
   Reason: <one line, for request-changes or unsure>
   Comment:
   <the exact review body to post>
   ```

   **Approve comment:** 1 to 3 sentences on what you checked. You may add a minor point the author could improve. If auto-merge is enabled, the PR will likely merge on approval and the author will not push a follow-up, so phrase any minor point as a note for future work ("Worth a follow-up: ..."). Anything that must change before merge is not minor, so it cannot go in an approval. If the PR has an expected merge conflict, add one line naming the other PR and the shared files, and say that whichever merges second needs a rebase.

   **Request-changes comment:** one sentence of context, then a short list of the required changes. Each item names the file and line, says what is wrong, says how it happens, and says how you reproduced it. For a conflict or duplicate, link the other PR and say how the two interact.

   Write each reproduction so the author, or an agent working for them, can rerun it and later confirm the fix: the exact command and test or input, and what happened compared with what should happen. For a trace, list the steps with their files and lines. Put a test or script longer than a few lines in a collapsed `<details>` block under its item. For example:

   ````markdown
   An empty cart breaks the new total, and the new cancel route lets users cancel each other's orders.

   - `orders/total.ts:42`: an empty cart makes the total `NaN`, because the discount step divides by the item count. Checkout then shows "NaN" and the payment call fails.
     Reproduced by adding this test to `orders/total.test.ts` and running `pnpm vitest run orders/total.test.ts` on the PR head. It fails with `expected NaN to be 0`.
     <details><summary>Test</summary>

     ```ts
     it("totals an empty cart", () => {
       expect(total([])).toBe(0);
     });
     ```

     </details>
   - `api/orders.ts:88`: any signed-in user can cancel another user's order, because the new route skips the ownership check that the other order routes run.
     Reproduced by trace: `POST /orders/:id/cancel` goes from the router (`api/routes.ts:31`) straight to `cancelOrder` (`api/orders.ts:88`), and neither one calls `requireOwner` (`api/middleware.ts:20`).
   ````

   **Unsure comment:** the specific questions the author needs to answer. For a finding you could not reproduce, say what it is, what you tried, and what would confirm it.

   Comment style: neutral and lean, first person only when needed. Say only what the author needs to act on or know. Skip praise, summaries of the diff, and points earlier reviewers already made. No em dashes. No mention of AI. Follow "Write comments people can understand" below.

### Follow-up review instructions

A person already approved or requested changes on this PR at the `since` commit. Review only what changed after it. Do not repeat the full review of code that has not changed since then. Do not post anything to GitHub. Only report back.

1. **Read the intent and history.** `gh pr view <n> --json title,body,comments,reviews`. If `since` is a change request, list what it asked for.
2. **Read the new changes.**
   - The author's own commits: `git log -p --first-parent --no-merges <since>..<headOid>`.
   - Each merge: list them with `git log --first-parent --merges --format=%H <since>..<headOid>`, then run `git show --remerge-diff <oid>` on each. Empty output means git merged cleanly with no manual edits. Otherwise the output shows how the conflicts were resolved.

   Read the code around the new changes at the head commit as deeply as the full review does, but only for those changes.
3. **Check CI.** Same as step 3 of the full review.
4. **Check overlap.** Same as step 4 of the full review, limited to files the new commits touched. Skip this for `merge-only` PRs.
5. **Reproduce.** Same as step 5 of the full review, for findings in the new changes.
6. **Decide.** Use the same bar as step 6 of the full review, applied to the new changes. If `since` is a change request, also check that each requested change is addressed. Merge commits almost never add logic changes, so a `merge-only` PR should be approved unless a conflict resolution drops or changes logic from either side, or CI fails because of the merge.
7. **Report** in the same format as the full review. Base the summary on the whole PR. Keep the approve comment to what you checked in the new commits. For `merge-only` PRs, one sentence is enough, for example: "Re-approving after the merge from `<base>`. The conflict resolution keeps both sides."

### Write comments people can understand

- Assume the reader understands the feature but does not know its internal implementation.
- Explain what happens, why it matters, and what needs to change.
- Describe behavior before naming functions, fields, or files. Use those names to locate the problem, not as substitutes for explaining it.
- Prefer concrete examples, such as "fetching the same unchanged data twice", over internal or jargon-dense terms such as "distinct observations".
- When an explanation needs several technical definitions, rewrite it around observable behavior instead.
- Keep the explanation accurate. Plain language must preserve the cause and consequence.
- For overlapping PRs, distinguish incompatible behavior from shared files that need merging or regeneration.

## Step 4: Verify reviews

Skim each report. If a verdict does not match its own reasoning (for example, an approval whose comment describes a likely bug), move it to unsure. Drop any blocking item that does not say how the problem happens. Turn any blocking item that does not say how it was reproduced into a question, as in step 6 of the full review. CI failures and overlap findings are exempt. If a request-changes PR has no blocking items left, move it to unsure if it has questions, or to approve if it does not.

Make overlap verdicts consistent. If one review found a conflict or duplicate with another eligible PR, the other PR's review must also block on it. Fix any approval that misses this. If one review classed a pair as an expected merge conflict and the other review did not find a conflict or duplicate, neither review blocks on it.

## Step 5: Post approvals

This covers approved PRs from Step 3 and `re-approve` PRs. For a `re-approve` PR, write the one-sentence summary from its description, and use this comment: "Re-approving. There are no new commits since the earlier approval was dismissed."

For each approved PR, first confirm the head has not moved: `gh pr view <n> --json headRefOid -q .headRefOid`. If it changed, do not post. Treat the PR as unsure and note that new commits arrived during the review.

Then post:

```bash
gh pr review <n> --approve --body "<comment>"
```

## Step 6: Run the rest by the user

For every request-changes and unsure PR, show:

- Link and the one-sentence summary.
- Verdict and the one-line reason.
- The exact comment it would post.

Then ask what to do with each one:

- **Post as written.** Request changes, or comment for unsure.
- **Edit the wording,** then post.
- **Downgrade** to a plain comment that does not block.
- **Approve** instead.
- **Skip** without posting.

Wait for the user's answers. Check the head commit again before posting each one, as in Step 5. Post with `gh pr review <n> --request-changes`, `--comment`, or `--approve`, and pass the final text with `--body`.

## Step 7: Summary

End with one table covering every PR in scope. The summary column uses the one-sentence summary from each review report, so the user learns what each PR does. For PRs that were not reviewed, use the PR title instead. The result column says what was posted: approved, changes requested, commented, not posted, or skipped with the reason.

| PR | Summary | Result |
| --- | --- | --- |
| [#12](url) | Retries failed S3 uploads up to 3 times with backoff so large exports stop failing on transient errors. | approved |
| [#15](url) | Moves session tokens from localStorage to HTTP-only cookies and adds a refresh endpoint. | changes requested |
| [#16](url) | Bump deps | skipped (own PR) |
