# skills

Agent skills for repository setup and git workflow.

| Skill | Purpose |
|---|---|
| `setup-repo` | Sets up CI and development tools in a new or existing repository: pre-commit, Ruff, gitleaks, zizmor, Dependabot, branch protection, OpenSpec, CodeRabbit, license. |
| `commit-properly` | Commits the changes on the current branch as tested, coherent commits with conventional messages. |
| `start-task` | Creates a git worktree and branch for a Linear or GitHub issue. |
| `open-issue` | Files a bug or task found in the current session as a GitHub issue. |

## Install

With the [skills CLI](https://github.com/vercel-labs/skills):

```
npx skills add csggnn/skills
```

From a clone, linked so that edits in the clone take effect:

```
git clone https://github.com/csggnn/skills ~/Projects/skills
for s in ~/Projects/skills/skills/*/; do
    ln -s "$s" ~/.agents/skills/
    ln -s ~/.agents/skills/"$(basename "$s")" ~/.claude/skills/
done
```

## Layout

```
skills/<name>/SKILL.md   frontmatter name and description, then the instructions
skills/<name>/*          files the skill reads, e.g. setup-repo/templates/ci.yml
```

## License

MIT
