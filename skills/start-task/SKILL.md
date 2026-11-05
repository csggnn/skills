---
name: start-task
description: Create a git worktree and branch to work on a Linear or GitHub issue
argument-hint: <Linear issue id, e.g. CSG-15, or GitHub issue number, e.g. #12>
allowed-tools: mcp__linear-server__get_issue, mcp__linear-server__save_issue, Bash(gh issue view:*), Bash(gh issue edit:*), Bash(git:*), Bash(cp:*), Bash(ls:*), Bash(cmp:*)
---

Set up a worktree and branch for issue `$ARGUMENTS`.

# Issue

The argument selects the tracker:
- `<KEY>-<number>`, e.g. `CSG-15`: a Linear issue. Fetch it with `get_issue`.
- `#<number>` or `<number>`: a GitHub issue of the current repository. Fetch it with
  `gh issue view <number> --json number,title,body,state,assignees`.

If the issue does not exist or a GitHub issue is closed, stop and report.

# Names

Branch: `<type>/<id>-<slug>`.
- `<type>` is one of `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, chosen from the
  issue title and description.
- `<id>` is the Linear identifier in lowercase, e.g. `csg-15`, or `gh-<number>` for a GitHub
  issue, e.g. `gh-12`.
- `<slug>` is 2 to 5 kebab-case words taken from the title.

Do not use Linear's `gitBranchName`. It uses a different type prefix and the full title.

Worktree path: `.trees/<id>-<slug>`, relative to the main checkout.

# Preconditions

Stop and report if any of these fails:
- The current directory is the main checkout: `git rev-parse --git-dir` equals
  `git rev-parse --git-common-dir`.
- `.trees/` is ignored: `git check-ignore -q .trees/`.
- No local or remote branch already matches `<id>-`: check `git branch -a --list '*<id>-*'`.
  If one exists, report it and ask whether to attach a worktree to it instead.
- The worktree path does not exist.

# Create

1. `git fetch --prune origin`.
2. Resolve the default branch with `git symbolic-ref --short refs/remotes/origin/HEAD`. If
   that ref is not set, run `git remote set-head origin --auto` and retry.
3. `git worktree add --no-track -b <branch> .trees/<id>-<slug> <ref>`, where `<ref>` is the
   output of step 2, e.g. `origin/master`.
   `--no-track` leaves the branch without an upstream, so the first push sets one with the
   branch's own name.

# Git-invisible files

The skip-worktree bit lives in each checkout's index and a new worktree starts without it.
List the main checkout's skip-worktree files with `git ls-files -v | grep '^S'`. For each:
1. Set the bit in the new worktree: `git -C <worktree> update-index --skip-worktree <file>`.
2. If the file is a `.env` file, copy the main checkout's version into the worktree, so the
   worktree has working keys.
3. Leave every other file at its committed version.

# Issue status

Linear: if the issue's status type is `backlog` or `unstarted`, set it to `In Progress` with
`save_issue`. Leave any other status unchanged.

GitHub: if the issue has no assignee, assign it with
`gh issue edit <number> --add-assignee @me`. Leave existing assignees unchanged.

# Report

State:
- The branch name and the base commit (short hash and subject).
- The worktree path.
- Each skip-worktree file and whether it was copied or left at the committed version.
- The issue status or assignee, and whether it was changed.
- The command to start a session there: `cd <worktree> && claude`.
