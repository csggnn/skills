---
name: setup-repo
description: Set up CI and development tools in a new or existing repository after asking which ones to add
argument-hint: [path to repo, default current directory]
---
Set up CI and development tools in the target repository. The target is `$ARGUMENTS` or the
current directory.

## 1. Detect

Collect these facts. Do not modify anything in this step.

- Git state: `git rev-parse --show-toplevel`, `git remote -v`, current branch.
  No git repo: offer `git init`.
- GitHub remote: `gh repo view --json nameWithOwner,visibility,defaultBranchRef`.
- Languages, from manifests and file extensions:
  `pyproject.toml`, `setup.py`, `*.py` → Python.
  `package.json`, `*.ts`, `*.js` → JS/TS.
  `go.mod` → Go. `Cargo.toml` → Rust.
- Code expected: true unless the repo holds only docs, data or config, or the user said so.
- Existing setup: `.github/workflows/`, `.pre-commit-config.yaml`, `.coderabbit.yaml`,
  `openspec/`, `.claude/settings*.json`, `[tool.ruff]`, `.editorconfig`, `.gitignore`,
  `.github/dependabot.yml`, `CLAUDE.md`, `LICENSE*`, `COPYING*`, the `license` field in
  `pyproject.toml` or `package.json`.

Print a short table of the findings.

## 2. Ask

Compute defaults from the rules below. Ask with AskUserQuestion, `multiSelect: true`, one
question per group. Append " (Recommended)" to the label of each default-on option. Skip
options that are already set up and list them as "present" in the question text. Skip
options whose language does not match.

| Group | Tool | Default on when |
|---|---|---|
| Review & AI | CodeRabbit | GitHub remote exists |
| Review & AI | OpenSpec | code expected |
| Review & AI | simplicity-rules plugin | Python, or no language detected |
| Review & AI | CLAUDE.md via `/init` | code exists and no CLAUDE.md |
| Lint & test | Ruff (lint + format) | Python, or no language detected |
| Lint & test | Type checker (pyright) | off |
| Lint & test | pytest scaffold | Python and no tests directory |
| Lint & test | Language linter for non-Python (Biome, golangci-lint, clippy) | matching language |
| CI | GitHub Actions CI workflow | GitHub remote exists, or always if the user plans one |
| CI | Dependabot (github-actions + package ecosystem) | GitHub remote exists |
| CI | zizmor (workflow security lint) | CI workflow selected |
| CI | Branch protection requiring CI | off (outward-facing change) |
| Hygiene | pre-commit hooks | on |
| Hygiene | gitleaks secret scan | on |
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
existing config instead of overwriting it. Resolve current versions at run time
(`pre-commit autoupdate`, `uv add`, release pages); do not use versions from memory.

### Ruff
- Python with uv: `uv add --dev ruff`. Otherwise `uvx ruff` in hooks and CI.
- Add to `pyproject.toml`:
  ```toml
  [tool.ruff]
  line-length = 100

  [tool.ruff.lint]
  select = ["E", "F", "W", "I", "B", "UP", "SIM", "C90", "RUF"]

  [tool.ruff.lint.mccabe]
  max-complexity = 10
  ```
- Run `ruff check --fix` and `ruff format`. Report remaining findings; do not hand-fix them
  unless asked.

### Type checker
- `uv add --dev pyright`. Add `[tool.pyright]` with `typeCheckingMode = "standard"`.

### pytest
- `uv add --dev pytest`. Create `tests/test_smoke.py` importing the package.

### pre-commit
- `.pre-commit-config.yaml` with hooks for the selected tools:
  `pre-commit-hooks` (trailing-whitespace, end-of-file-fixer, check-yaml, check-toml,
  check-added-large-files), `ruff-pre-commit` (ruff-check `--fix`, ruff-format),
  `gitleaks`, `zizmor` when selected.
- Run `pre-commit autoupdate`, `pre-commit install`, `pre-commit run --all-files`.

### GitHub Actions CI
- `.github/workflows/ci.yml`:
  - Triggers: `push` to the default branch, `pull_request`.
  - `permissions: contents: read`.
  - `concurrency` keyed on workflow and ref, `cancel-in-progress: true`.
  - Python job: `actions/checkout`, `astral-sh/setup-uv`, `uv sync --locked`,
    `uv run ruff check`, `uv run ruff format --check`, type check if selected,
    `uv run pytest` if tests exist.
  - Non-Python jobs: the equivalent lint, format check and test commands.
  - `persist-credentials: false` on checkout.
- Run `uvx zizmor .github/workflows` when zizmor is selected.

### Dependabot
- `.github/dependabot.yml` with `github-actions` and the package ecosystem (`uv`, `npm`,
  `gomod`, `cargo`), weekly schedule, grouped minor and patch updates.

### CodeRabbit
- Write `.coderabbit.yaml`:
  ```yaml
  # yaml-language-server: $schema=https://coderabbit.ai/integrations/schema.v2.json
  language: en-US
  reviews:
    profile: chill
    request_changes_workflow: false
    high_level_summary: true
    poem: false
    auto_review:
      enabled: true
      drafts: false
  ```
- Check installation: `gh api repos/{owner}/{repo}/installation` fails without app admin
  rights, so ask the user instead. If the app is not installed, give the link
  `https://github.com/apps/coderabbitai/installations/new` and state that selecting
  "All repositories" covers repos created later.

### OpenSpec
OpenSpec is a pinned dev dependency. Dependabot bumps it and CI regenerates its files.
`csggnn/simplicity-rules` is the reference setup.

- Pin the CLI: `npm install --save-dev --save-exact --ignore-scripts @fission-ai/openspec@latest`.
  Commit `package.json` and `package-lock.json`. Add `node_modules/` to `.gitignore`.
- `npx openspec init --tools claude --no-animation` in the repo root.
- Merge the `rules:` block of `openspec-rules.yaml` (in this skill's folder) into
  `openspec/config.yaml`. Repo-specific rules go after the shared rules of the same artifact.
- The `.claude/skills/openspec-*` files are generated. Do not edit them.
- CI workflow, after the other checks:
  ```yaml
      - name: Install OpenSpec
        if: ${{ !cancelled() }}
        run: npm ci --ignore-scripts --no-audit --no-fund

      - name: Validate OpenSpec changes and specs
        if: ${{ !cancelled() }}
        run: npx openspec validate --all --no-interactive

      - name: Check generated OpenSpec files match the pinned version
        if: ${{ !cancelled() }}
        run: |
          npx openspec update --force > /dev/null
          git diff --exit-code -- .claude
  ```
- Dependabot: add an `npm` entry (directory `/`, weekly, `cooldown: default-days: 7`).
- Copy `.github/workflows/openspec-regenerate.yml` from `csggnn/simplicity-rules`
  (`gh api repos/csggnn/simplicity-rules/contents/.github/workflows/openspec-regenerate.yml -q .content | base64 -d`).
  It regenerates the files on a Dependabot bump, approves the CI runs of its commit and
  enables auto-merge.
- Auto-merge needs "Allow auto-merge"
  (`gh api -X PATCH repos/{owner}/{repo} -F allow_auto_merge=true`) and branch protection
  that requires the CI check. Without a required check, auto-merge merges before CI runs.
  Select "Branch protection requiring CI" when OpenSpec is selected. Branch protection on
  a private repo needs GitHub Pro.
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
- `.editorconfig`: `root = true`, utf-8, lf, final newline, trim trailing whitespace,
  4-space indent for Python, 2-space for YAML, JSON, JS/TS.
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

### GitHub settings (outward-facing, confirm before each)
- Secret scanning and push protection:
  `gh api -X PATCH repos/{owner}/{repo} -f security_and_analysis[secret_scanning][status]=enabled -f security_and_analysis[secret_scanning_push_protection][status]=enabled`.
- Branch protection: run only after CI has passed once on the default branch, so the check
  name exists. Require that check and pull requests before merge.

### Documentation
Runs after all tools are applied and before CLAUDE.md.
- Read the repo docs: `README*`, `CONTRIBUTING*`, `docs/`, `DEVELOPMENT*`, `HACKING*`,
  `CLAUDE.md`, `AGENTS.md`.
- Pick one target, first match wins:
  1. An existing section on development, contributing, testing or CI, in any of these files.
  2. `CONTRIBUTING.md` or a development page under `docs/`.
  3. A new `## Development` section at the end of `README.md`.
  4. A new `README.md` with the repo name and that section.
- Write a short section, at most about 15 lines:
  - One-time setup command (e.g. `uv sync`, `pre-commit install`).
  - Command that runs all local checks (e.g. `pre-commit run --all-files`).
  - What CI runs and on which events.
  - Other automation in use, one line each: Dependabot, CodeRabbit, OpenSpec,
    simplicity-rules.
- Match the target file's heading levels, tone and formatting. Reuse the commands from the
  config files; do not invent commands.
- Describe only tools that are present after this run. If the target already describes CI,
  update that text instead of adding a second description.
- In `CLAUDE.md` or `AGENTS.md`, when present and not the target, add the local check
  command if it is missing.

### CLAUDE.md
- Invoke the `init` skill last, so it sees the final setup and docs.

## 4. Verify and report

- Run each installed check locally: pre-commit, ruff, type checker, tests, zizmor.
- Print a table: tool, status (added / present / skipped / failed), files touched.
- Name the doc file and section that holds the CI description.
- List manual steps left to the user (CodeRabbit app install, secrets, first CI run).
- Do not commit. Offer the `commit-properly` skill, one commit per tool.
