# skills

Personal AI skills, usable in both Claude Code and Codex (or any harness that follows the [Agent Skills](https://code.claude.com/docs/en/skills) format).

## Skills

Vendored skills from [mattpocock/skills](https://github.com/mattpocock/skills):

- `grill-me`: relentless interview to sharpen a plan or design.
- `handoff`: compact the current conversation into a handoff document for another agent.
- `teach`: guided, stateful teaching workspace for learning a topic over multiple sessions.

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
