---
name: setup-repo
description: Set up CI and development tools in a new or existing repository after asking which ones to add, or sync a repository set up earlier with the current templates
argument-hint: [path to repo, default current directory]
---
Set up CI and development tools in the target repository. The target is `$ARGUMENTS` or the
current directory.

## Principles

- `.pre-commit-config.yaml` defines every check that reads files as text: hygiene, Ruff,
  shellcheck, gitleaks, zizmor. Local commits and CI run the same file.
- Checks that import or run the project's code (tests, type checker, the app) run in the
  project environment: `uv run` for a uv project, the repo's container for a containerized
  repo. pre-commit runs neither.
- Every version is pinned and Dependabot updates it: GitHub Actions by commit hash, pre-commit
  hooks by `rev`, packages by lockfile, OpenSpec by `package.json`.
- Every update reaches the default branch through a pull request that passes the required CI
  checks.
- `templates/` in this skill's folder holds the standard files. Every repo gets the same
  files. A repo-specific deviation from a template carries a comment that starts with
  `# exception:` and states its reason.
- uv and pre-commit run at their latest release. The hooks and lockfiles they run are pinned.

## Templates

| File | Repo path |
|---|---|
| `templates/ci.yml` | `.github/workflows/ci.yml` |
| `templates/pre-commit-config.yaml` | `.pre-commit-config.yaml` |
| `templates/dependabot.yml` | `.github/dependabot.yml` |
| `templates/dependabot-automerge.yml` | `.github/workflows/dependabot-automerge.yml` |
| `templates/openspec-regenerate.yml` | `.github/workflows/openspec-regenerate.yml` |
| `templates/openspec-rules.yaml` | `rules:` block of `openspec/config.yaml` |
| `templates/coderabbit.yaml` | `.coderabbit.yaml` |
| `templates/editorconfig` | `.editorconfig` |

Placeholders, filled when a file is written:
- `PIN`: `<commit sha> # <tag>` of the action's latest release.
- `LATEST`: any tag, then `uvx pre-commit autoupdate` sets the latest.
- `DEFAULT_BRANCH`, `TEST_COMMAND`, `ECOSYSTEM`: from the detected facts.

A `# block: <tool>` ... `# end block` section is written only when that tool is selected.
The marker lines are not written.

## Mode

- **Sync** when the repo has `.pre-commit-config.yaml` and `.github/workflows/ci.yml`
  with a `check` job. Skip step 2 and run [Sync](#sync).
- **Setup** otherwise. Run steps 1 to 5.

## 1. Detect

Collect these facts. Do not modify anything in this step.

- Git state: `git rev-parse --show-toplevel`, `git remote -v`, current branch.
  No git repo: offer `git init`.
- GitHub remote: `gh repo view --json nameWithOwner,visibility,defaultBranchRef`.
- Languages, from manifests and file extensions:
  `pyproject.toml`, `setup.py`, `*.py` → Python.
  `package.json`, `*.ts`, `*.js` → JS/TS.
  `go.mod` → Go. `Cargo.toml` → Rust. `*.sh` → shell.
- Package manager: `uv.lock`, `[tool.uv]` or the `uv_build` backend in `pyproject.toml` → uv
  (run `uv lock` when `uv.lock` is missing). `requirements*.txt` otherwise → pip.
- Container: `Dockerfile`, `Containerfile`, `docker-compose.yml`, `compose.yml`, and a
  statement in the docs that tests or the app run in a container.
- Test command: from CI workflows, `README*`, `CLAUDE.md`, `pyproject.toml`.
- Code expected: true unless the repo holds only docs, data or config, or the user said so.
- Existing setup: `.github/workflows/` (and each `uses:` that is not a 40-character commit
  hash), `.pre-commit-config.yaml`, `git config core.hooksPath` and its folder, scripts that
  wrap a linter (e.g. `scripts/ruff.sh`), `.claude/settings*.json` hooks that call such a
  script, `.coderabbit.yaml`, `openspec/`, `package.json`, `[tool.ruff]` or `ruff.toml`,
  `.editorconfig`, `.gitignore`, `.github/dependabot.yml`, `CLAUDE.md`, `LICENSE*`,
  `COPYING*`, the `license` field in `pyproject.toml` or `package.json`.
- GitHub settings: `gh api repos/{owner}/{repo} -q '{auto_merge: .allow_auto_merge, security: .security_and_analysis}'`
  and `gh api repos/{owner}/{repo}/branches/{default}/protection` (404: none; 403: needs a
  public repo or GitHub Pro).

Print a short table of the findings.

## Sync

Brings a repo set up earlier to the current templates.

1. Run step 1.
2. Compare each template with its repo file. Ignore versions: Dependabot owns them.
   Classify each difference:
   - In the template, missing from the repo: add it.
   - In the repo, missing from the template, with an `# exception:` comment: keep it.
   - In the repo, missing from the template, without an `# exception:` comment: ask whether
     to remove it or keep it with an `# exception:` comment.
3. Add missing Dependabot ecosystems, missing templates whose tool applies, and the
   GitHub settings of step 4 that are not set.
4. Update the docs as in [Documentation](#documentation).
5. Publish as in step 4, one commit per tool, and report as in step 5.

## 2. Ask

Compute defaults from the rules below. Ask with AskUserQuestion, `multiSelect: true`, one
question per group. Append " (Recommended)" to the label of each default-on option. Skip
options that are already set up and list them as "present" in the question text. Skip
options whose language or precondition does not match.

| Group | Tool | Default on when |
|---|---|---|
| Review & AI | CodeRabbit | GitHub remote exists |
| Review & AI | OpenSpec | code expected |
| Review & AI | simplicity-rules plugin | Python, or no language detected |
| Review & AI | CLAUDE.md via `/init` | code exists and no CLAUDE.md |
| Checks | pre-commit (hygiene hooks, gitleaks) | always |
| Checks | Ruff (lint + format) | Python, or no language detected |
| Checks | shellcheck | shell scripts exist |
| Checks | Type checker (pyright) | off |
| Checks | pytest scaffold | Python, uv project and no tests directory |
| Checks | Language linter for non-Python (Biome, golangci-lint, clippy) | matching language |
| CI | GitHub Actions CI workflow | GitHub remote exists |
| CI | zizmor and actions pinned by hash | workflows exist or CI selected |
| CI | Dependabot | GitHub remote exists |
| CI | Branch protection requiring CI | public repo, or private repo on GitHub Pro |
| Hygiene | `.editorconfig` + `.gitignore` | missing |
| Hygiene | GitHub secret scanning + push protection | public GitHub repo |

AskUserQuestion allows 4 options per question. Split a group across two questions when it
has more than 4 applicable options.

When no license file exists, ask one single-select question "License" with the repo
visibility in the question text:
- MIT (Recommended): permissive open source license.
- Proprietary, all rights reserved: closed license. No use, copy, modification or
  distribution without written permission from the copyright holder.
- No license file.

Ask one follow-up only when needed: the simplicity-rules scope (`settings.local.json` for
this machine, or `settings.json` committed for all contributors). Default:
`settings.local.json`.

## 3. Apply

Apply selected tools in this order. Read each existing file before changing it. Merge into
existing config instead of overwriting it. Resolve current versions at run time; do not use
versions from memory:
- pre-commit hooks: `uvx pre-commit autoupdate`.
- A GitHub Action: latest tag `gh release view -R <owner>/<action> --json tagName -q .tagName`,
  its commit `gh api repos/<owner>/<action>/commits/<tag> -q .sha`.
- PyPI and npm packages: `uv add`, `npm install`, `npm view <package> version`.

### Ruff
- Configuration in `pyproject.toml`, or `ruff.toml` when the repo has no `pyproject.toml`.
  Keep an existing configuration. A new one:
  ```toml
  [tool.ruff]
  line-length = 100

  [tool.ruff.lint]
  select = ["E", "F", "W", "I", "B", "UP", "SIM", "C90", "RUF"]

  [tool.ruff.lint.mccabe]
  max-complexity = 10
  ```
- Ruff runs from the `ruff-pre-commit` hooks. Do not add it as a project dependency.
- Run the hooks on all files. Report remaining findings; do not hand-fix them unless asked.

### Type checker
- uv project: `uv add --dev pyright`, `[tool.pyright]` with `typeCheckingMode = "standard"`,
  and a CI step `uv run pyright`. Containerized repo: a CI step that runs pyright in the
  container.

### pytest
- `uv add --dev pytest`. Create `tests/test_smoke.py` importing the package.

### pre-commit
- `templates/pre-commit-config.yaml`, with the Ruff, shellcheck and zizmor blocks when
  selected.
- Repo-specific steps that need a project tool, e.g. a diagram render that runs a container,
  are `repo: local` hooks with `language: system` and an `# exception:` comment.
- Replace an existing custom hook setup:
  - Port each step of the `core.hooksPath` hook to a pre-commit hook. Then delete the hook
    folder and run `git config --unset core.hooksPath`. `pre-commit install` refuses to run
    while `core.hooksPath` is set.
  - Replace calls to a linter wrapper script with the pre-commit hook, e.g. a Claude Code
    hook runs `uvx pre-commit run ruff-check --files "$file"` and
    `uvx pre-commit run ruff-format --files "$file"`. Delete the wrapper when nothing calls it.
- Run `uvx pre-commit autoupdate`, `uvx pre-commit install`,
  `uvx pre-commit run --all-files`.

### GitHub Actions CI
- `templates/ci.yml`. The `uv project` block for a uv project, with `TEST_COMMAND` such as
  `uv run pytest`. The type checker step goes in that block when selected.
- A containerized repo runs its tests in a second job of `ci.yml` that builds the container,
  with an `# exception:` comment. Replace a lint workflow that duplicates a pre-commit hook
  with the `check` job.
- A hook that cannot run in CI is listed in `env: SKIP:` of the pre-commit step, with an
  `# exception:` comment.
- Pin every `uses:` in every workflow to a commit hash with the tag as a comment:
  `uses: actions/checkout@<sha> # v7.0.1`. A major-version tag such as `@v4` is pinned to
  the commit of the latest release; run CI before merging that upgrade.

### Dependabot
- `templates/dependabot.yml`, one entry per ecosystem present.
- `templates/dependabot-automerge.yml`. On each push to a Dependabot pull request it enables
  auto-merge for a minor or patch update and disables it for any other update or failed
  metadata. It is the only workflow that sets auto-merge.
- Auto-merge needs "Allow auto-merge"
  (`gh api -X PATCH repos/{owner}/{repo} -F allow_auto_merge=true`) and branch protection
  that requires the CI checks. Without a required check, auto-merge merges before CI runs.
  Select "Branch protection requiring CI" when Dependabot is selected. Without branch
  protection, skip the workflow and state that Dependabot pull requests need a manual merge.

### CodeRabbit
- `templates/coderabbit.yaml`.
- Check installation: `gh api repos/{owner}/{repo}/installation` fails without app admin
  rights, so ask the user instead. If the app is not installed, give the link
  `https://github.com/apps/coderabbitai/installations/new` and state that selecting
  "All repositories" covers repos created later.

### OpenSpec
OpenSpec is a pinned dev dependency. Dependabot bumps it and CI regenerates its files.

- Pin the CLI: `npm install --save-dev --save-exact --ignore-scripts @fission-ai/openspec@latest`.
  Set `"private": true` in `package.json`. Commit `package.json` and `package-lock.json`.
  Add `node_modules/` to `.gitignore`.
- `npx openspec init --tools claude --no-animation` in the repo root.
- Merge the `rules:` block of `templates/openspec-rules.yaml` into
  `openspec/config.yaml`. Repo-specific rules go after the shared rules of the same artifact.
- The `.claude/skills/openspec-*` files are generated. Do not edit them.
- The `OpenSpec` block of `templates/ci.yml`.
- Dependabot: the `npm` entry.
- `templates/openspec-regenerate.yml`. It regenerates the files on a Dependabot bump and
  approves the CI runs of its commit. `dependabot-automerge.yml` sets auto-merge.
- When the approval step in the workflow fails, the Dependabot pull request stays open with
  CI runs awaiting approval. Approve them once in the Actions tab. After that merge, the
  bot's commit is on the default branch and later runs need no approval.
- Local runs use `~/.local/bin/openspec`, which runs the version pinned in the enclosing
  repo's `package.json` through `npx`.

### simplicity-rules plugin
- If `claude plugin marketplace list` lacks it: `claude plugin marketplace add csggnn/simplicity-rules`.
- If not installed: `claude plugin install simplicity-rules@simplicity-rules`.
- Merge into the chosen settings file:
  `{"enabledPlugins": {"simplicity-rules@simplicity-rules": true}}`.
- Run the plugin's `scripts/check-versions.sh` and report mismatches.
- Requires uv. Install uv first if missing, after asking.

### Hygiene files
- `templates/editorconfig` as `.editorconfig`.
- `.gitignore`: language template entries plus `.env`, `.venv/`, `.claude/settings.local.json`.

### License
- Copyright holder: `git config user.name`, confirmed with the user. Year: current year.
- MIT: write `LICENSE` with the standard MIT text from https://opensource.org/license/mit.
  Set `license = "MIT"` in `pyproject.toml` and `"license": "MIT"` in `package.json`.
- Proprietary: write `LICENSE`:
  ```
  Copyright (c) <year> <holder>. All rights reserved.

  This software and its source code are proprietary and confidential. No part of it may
  be used, copied, modified, merged, published, distributed, sublicensed or sold without
  prior written permission from the copyright holder.
  ```
  Set `license = "LicenseRef-Proprietary"` in `pyproject.toml`. In `package.json` set
  `"license": "UNLICENSED"` and `"private": true`.
- If the repo is public on GitHub and the user picks Proprietary, state that the code is
  still visible to anyone and ask whether to keep it public.
- Update an existing License section in `README.md`. Add a one-line `## License` section
  when the README has none.

### Documentation
Runs after all tools are applied and before CLAUDE.md.
- Read the repo docs: `README*`, `CONTRIBUTING*`, `docs/`, `DEVELOPMENT*`, `HACKING*`,
  `CLAUDE.md`, `AGENTS.md`.
- Pick one target, first match wins:
  1. An existing section on development, contributing, testing or CI, in any of these files.
  2. `CONTRIBUTING.md` or a development page under `docs/`.
  3. A new `## Development` section at the end of `README.md`.
  4. A new `README.md` with the repo name and that section.
- Write a short section, at most about 20 lines:
  - Development requirements, one line each: uv (pre-commit and its hooks), Node.js
    (OpenSpec), the container runtime when tests run in a container. A user who only runs
    the project does not need them; keep the user setup separate.
  - One-time setup commands, e.g. `uvx pre-commit install`, `uv sync`,
    `npm ci --ignore-scripts`.
  - Command that runs all local checks, e.g. `uvx pre-commit run --all-files`.
  - What CI runs and on which events, and the required checks.
  - Other automation in use, one line each: Dependabot, CodeRabbit, OpenSpec,
    simplicity-rules.
- Match the target file's heading levels, tone and formatting. Reuse the commands from the
  config files; do not invent commands.
- Describe only tools that are present after this run. If the target already describes CI,
  hooks or lint commands, update that text instead of adding a second description. Remove
  text about deleted hooks and scripts.
- In `CLAUDE.md` or `AGENTS.md`, when present and not the target, add the local check
  command if it is missing.

### CLAUDE.md
- Invoke the `init` skill last, so it sees the final setup and docs.

## 4. Publish

- Commit with the `commit-properly` skill, one commit per tool, on a branch.
- Push the branch and open a pull request. Wait for CI. Fix failures in the same branch.

### GitHub settings (outward-facing, confirm before each)
- Secret scanning and push protection:
  `gh api -X PATCH repos/{owner}/{repo} -f security_and_analysis[secret_scanning][status]=enabled -f security_and_analysis[secret_scanning_push_protection][status]=enabled`.
- Branch protection, after the CI jobs have run once, so their check names exist. Require
  each CI job that runs on every pull request, pull requests before merge, and the same
  rules for administrators:
  ```
  gh api -X PUT repos/{owner}/{repo}/branches/{default}/protection --input - <<'EOF'
  {"required_status_checks": {"strict": false, "contexts": ["check"]},
   "enforce_admins": true,
   "required_pull_request_reviews": {"required_approving_review_count": 0},
   "restrictions": null}
  EOF
  ```
  A job limited by `paths:` cannot be required: a pull request that skips it never merges.
- Allow auto-merge, when Dependabot is selected.

## 5. Verify and report

- Run each check locally: `uvx pre-commit run --all-files`, the tests, the type checker.
- Print a table: tool, status (added / present / skipped / failed), files touched.
- Name the doc file and section that holds the CI description.
- List manual steps left to the user (CodeRabbit app install, secrets, first Dependabot
  approval).
