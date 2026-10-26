---
name: review-prs
description: Review every open PR in the current repo (or the PR numbers given), approve the ones that are reasonably correct with a brief comment, and check with the user before requesting changes on any of them. Blocks on any conflict or duplicate work between open PRs, and accounts for auto-merge on PRs that have it enabled.
disable-model-invocation: true
argument-hint: "[PR numbers]"
---

# Review PRs

Review open PRs, approve the reasonably correct ones, and run every non-approval by the user before posting it.

## Step 1: Collect PRs

Run everything against the current repo. Get the user's login with `ME=$(gh api user -q .login)`.

Fetch all open PRs in one GraphQL call (`gh` fills in `{owner}` and `{repo}` from the current repo):

```bash
gh api graphql -F owner='{owner}' -F name='{repo}' -f query='
query($owner: String!, $name: String!) {
  repository(owner: $owner, name: $name) {
    pullRequests(states: OPEN, first: 100, orderBy: {field: CREATED_AT, direction: DESC}) {
      nodes {
        number title url isDraft headRefName baseRefName headRefOid mergeable
        author { login }
        autoMergeRequest { enabledAt }
        files(first: 100) { nodes { path } }
        reviews(last: 50) { nodes { author { __typename login } state commit { oid } } }
      }
    }
  }
}'
```

Build a compact roster with `jq`:

```bash
jq --arg me "$ME" '.data.repository.pullRequests.nodes | map(. as $pr | {
  number, title, url, author: .author.login, base: .baseRefName,
  headOid: .headRefOid, mergeable, autoMerge: (.autoMergeRequest != null),
  files: [.files.nodes[].path],
  skip: (if .isDraft then "draft"
    elif .author.login == $me then "own"
    elif any(.reviews.nodes[]; .author.__typename != "Bot"
      and .commit.oid == $pr.headRefOid
      and (.state == "APPROVED" or .state == "CHANGES_REQUESTED")) then "reviewed"
    else null end)
})'
```

A PR is **eligible** when `skip` is null. That excludes drafts, the user's own PRs, and PRs that a person already approved or blocked at the current head commit. Bot reviews do not count.

If a PR lists exactly 100 files, its list may be cut off. Fetch the full list with `gh pr view <n> --json files`.

If the user passed PR numbers, review only those, but keep the full roster for the overlap check. If a requested PR is not eligible, say why and skip it.

If there are no eligible PRs, say so and stop. If there are more than 10, list them (number, title, author) and ask the user to confirm or narrow the set before going on.

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

Start one subagent per eligible PR, all in parallel. If your harness has no subagents, review the PRs one at a time with the same instructions.

Give each subagent:

- The PR number, URL, base branch, head commit, and whether auto-merge is enabled.
- The condensed conventions from Step 2.
- The roster of all other open, non-draft PRs as one line each: number, title, author, file count. This lets it spot duplicate work across PRs, even when they share no files.
- Its entries from the overlap map: which PRs share which files.
- The review instructions below.

### Review instructions (pass to each subagent)

Do not post anything to GitHub. Only report back.

1. **Read the intent.** `gh pr view <n> --json title,body,closingIssuesReferences,comments,reviews`. Read the linked issues. Note points earlier reviewers already made so you do not repeat them.
2. **Read the change.** Diff the head against the base. Open related files at the head commit (callers, callees, types, config) when you need them to judge correctness. Do not check out the branch or run tests.
3. **Check CI.** Run `gh pr checks <n>`. If a check fails, read only the tail of its log (`gh run view <run-id> --log-failed | tail -100`) and decide whether the PR's own changes caused it. Failures unrelated to the PR do not block.
4. **Check overlap.** For each overlapping PR, fetch only the shared files' patches: `gh api repos/{owner}/{repo}/pulls/<other>/files --paginate --jq '.[] | select(.filename == "<path>") | .patch'`. Classify:
   - **Independent:** the PRs change unrelated parts of the shared files, and neither change affects the logic the other touches. Ignore.
   - **Expected merge conflict:** the PRs change the same lines, so git will report a conflict for whichever merges second, but neither change affects the logic the other relies on. Do not block if the PRs accomplish separate things, or if either PR mentions the other PR or the conflict in its body, comments, or reviews. Otherwise treat it as a conflict.
   - **Conflict:** the PRs change the same logic, or one change affects code the other relies on. Block, whether or not git can merge both cleanly.
   - **Duplicate work:** both PRs solve the same problem. Block.

   Also check the roster for duplicate work in PRs that share no files, using their titles. Fetch a candidate's description with `gh pr view <other> --json body` only when the title suggests the same goal.

   When in doubt between independent and conflict, choose conflict.
5. **Decide.**
   - **Request changes** for: bugs that are likely to happen or that matter when they do, security issues, data loss, broken behavior, missing parts of what the PR says it does, CI failures caused by the PR, any conflict or duplicate work with another open PR, and significant departures from the repo conventions.
   - **Never block on:** style, naming, small cleanups, missing tests, minor convention slips, or bugs that are unlikely to happen or have little effect when they do.
   - **Unsure** when you cannot judge correctness without context you do not have.
   - **Approve** otherwise.
6. **Report** in this format:

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

   **Request-changes comment:** one sentence of context, then a short list of the required changes. Each item names the file and says what is wrong and why. For a conflict or duplicate, link the other PR and say how the two interact.

   **Unsure comment:** the specific questions the author needs to answer.

   Comment style: neutral and lean, first person only when needed. Say only what the author needs to act on or know. Skip praise, summaries of the diff, and points earlier reviewers already made. No em dashes. No mention of AI.

## Step 4: Verify reviews

Skim each report. If a verdict does not match its own reasoning (for example, an approval whose comment describes a likely bug), move it to unsure.

Make overlap verdicts consistent. If one review found a conflict or duplicate with another eligible PR, the other PR's review must also block on it. Fix any approval that misses this. If one review classed a pair as an expected merge conflict and the other review did not find a conflict or duplicate, neither review blocks on it.

## Step 5: Post approvals

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

End with one table covering every PR in scope. The summary column uses the one-sentence summary from each review report, so the user learns what each PR does. For PRs that were not reviewed, use the PR title instead.

| PR | Summary | Verdict | Status |
| --- | --- | --- | --- |
| [#12](url) | Retries failed S3 uploads up to 3 times with backoff so large exports stop failing on transient errors. | approve | posted |
| [#15](url) | Moves session tokens from localStorage to HTTP-only cookies and adds a refresh endpoint. | request-changes | posted after edit |
| [#16](url) | Bump deps | skipped (own PR) | |
