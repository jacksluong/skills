# skills

My personal AI skills, usable in both Claude Code and Codex (or any harness that follows the [Agent Skills](https://code.claude.com/docs/en/skills) format).

## Skills

My own skills:

- `address-pr-comments`: address the review comments on a PR, fix the ones that hold up, and check with me before pushing back on any.
- `create-kb-md`: write a standalone Markdown reference note for a personal knowledge base.
- `create-plan`: interview, then write a comprehensive plan document to `.plans/` in the current repo.
- `orchestrate`: settle a plan, then have subagents implement it, review the result, and address the review in a separate worktree.
- `review-prs`: review open PRs in the current repo, approve the sound ones, and check with me before requesting changes.
- `self-review`: have four independent reviewer agents, one per lens, check the agent's work before its last commit, then fix what holds up and write the findings to a report.

Adapted skills from [mattpocock/skills](https://github.com/mattpocock/skills):

- `grill-me`: relentless interview to sharpen a plan or design.
- `handoff`: compact the current conversation into a handoff document for another agent.
- `teach`: guided, stateful teaching workspace for learning a topic over multiple sessions, saved to `.learning/<topic>/` in the current repo.

Adapted skills from [petergyang/no-ai-slop](https://github.com/petergyang/no-ai-slop):

- `no-ai-slop`: edit a draft to remove AI writing patterns while keeping the writer's voice, or flag those patterns without rewriting.

I also install [Impeccable](https://impeccable.style/) and [Playwright CLI](https://playwright.dev/) skills via my [dotfiles](https://github.com/jacksluong/dotfiles) repo.

## Install

Symlink each skill folder into the local skill directory for your harness:

```bash
mkdir -p ~/.claude/skills ~/.agents/skills
for skill in skills/*/; do
  name="$(basename "$skill")"
  ln -sfn "$(pwd)/$skill" ~/.claude/skills/"$name"
  ln -sfn "$(pwd)/$skill" ~/.agents/skills/"$name"
done
```

- `~/.claude/skills`: Claude Code.
- `~/.agents/skills`: Codex and other Agent Skills-compatible harnesses.

Since each entry is a symlink into this repo, `git pull` keeps installed skills current.
