---
name: explore-names
description: Explore and select names for a project, repository or agent skill through rounds that mix exploitation of liked names with exploration of new directions. Use when asked to name, rename or brainstorm names for a project, repo, package or skill.
argument-hint: [path, repo or skill to name, or a description of it]
---

Explore names for `$ARGUMENTS`. The target is a project, a repository or an agent skill. With no
argument, the target is the repository in the current directory.

# 1. Purpose statement

Classify the target's role:
- **Product**: delivers a capability to users.
- **Library**: code other projects import.
- **Wrapper**: binds or adapts one named implementation.
- **Sandbox**: exists to learn or try out a subject. Its tasks are examples.
- **Skill**: an action an agent performs on request.

Write one sentence. For a sandbox: "<target> explores <subject> through <example tasks>".
For every other role: "<target> <verb>s <object> for <user>". Build it from evidence, in this
order:
- The target's files: README, package metadata, entry points, tests, sample data, skill body.
  Tests and sample data show the domain when the README is empty.
- Sibling names: other repositories of the same owner (`gh repo list`), installed skills
  (`ls ~/.claude/skills ~/.agents/skills`), skills in the same collection.
- Domain vocabulary: the established terms for the category of the subject or object, and 5
  to 10 neighboring projects in that category. Pick the narrowest category term that stays
  true for every part the user expects to add. That term and its synonyms are candidate
  tokens in step 3.
- The current name. Split it into tokens and classify each token as purpose, domain,
  implementation, owner/client or filler.

Ask the user, in one message, only the facts the files cannot give:
- The meaning of tokens that are neither words nor known libraries (client, product, team).
- The role, when the files fit more than one. For a sandbox: the subjects being learned, and
  which parts are expected to be swapped or extended, and which features are planned.
- Which subjects are practices the user applies to every project, e.g. a development method
  or a toolchain. Practices stay out of the purpose statement and the name.
- Whether the target is published (PyPI, npm, public GitHub, a skill registry) or private.
- The audience: who reads the name. Users who install or search for it, contributors, or
  visitors of the owner's profile, e.g. interviewers reading a portfolio.
- For a skill: the verbs the user types when asking for the action.
- Constraints the user already has: a required prefix, a language, a length limit.

The step is done when the role is known, the purpose statement names the subject or the
action, and every token of the current name is classified.

# 2. Rules

Every candidate satisfies the rules for its kind.

All kinds:
- Lowercase kebab-case, ASCII letters, digits and hyphens.
- Describes purpose or domain. An implementation token (model, framework, language) appears
  when the target is a wrapper of that implementation, or when the user states the
  implementation stays for the target's lifetime, e.g. the framework of a sandbox. Other
  implementation tokens stay out, because the implementation can be replaced while the
  purpose stays.
- A sandbox name describes the subject at the level that stays true when the parts the user
  expects to swap are swapped, e.g. "web frameworks" when a second framework is planned.
  Example tasks stay out of the name. A sandbox name may mark the role with one word: `lab`,
  `playground`, `sandbox`, `kata`.
- An owner or client token appears only when the target exists for that client alone, and
  the user allows it. Names of published targets omit client names unless the user asks.
- 1 to 3 words. 4 words only for a skill whose action needs a qualifier.
- Distinct from every sibling name by more than a suffix.
- Every word appears in names of established projects the audience knows, e.g. for a
  portfolio, repositories of the domain's companies and labs. A word with a second meaning in
  the domain carries that meaning too, e.g. `playground` suggests an interactive UI.
- No time-relative tokens: `sota`, `new`, `modern`, `next`, `latest`. They go stale while
  the target stays.
- No generic filler: `tool`, `utils`, `helper`, `project`, `app`, `manager`, `system`.

Repository and project:
- The repository name, the distribution name and the import package (hyphens replaced by
  underscores) are the same word sequence.
- When published, the name is free on the target registry.

Skill:
- At most 64 characters. The skill directory has the same name.
- Verb first, then object: `<verb>-<object>`, e.g. `open-issue`, `setup-repo`. The verb is
  one the user types when asking for the action, or one the user likes in a round.
- Follows the verb style of the sibling skills.

# 3. Candidates

Generate at least 12 candidates, at least 2 from each strategy. The examples are for a tool
that shrinks photos for upload.
- **Literal**: purpose statement words, e.g. `resize-photos`.
- **Domain noun**: the thing operated on or produced, e.g. `web-thumbnails`.
- **Action**: verb-object from the user's view, e.g. `shrink-images`.
- **Metaphor**: a real word with a related meaning in another field, e.g. `darkroom`.
- **Compound**: two short words fused, e.g. `pixpress`.

Drop every candidate that breaks a rule in step 2.

# 4. Checks

For each remaining candidate run the checks that apply:
- Siblings: no collision with the lists gathered in step 1.
- GitHub: `gh search repos <name> in:name --limit 5 --json fullName,stargazersCount,description`.
  When the audience installs or searches for the target, a repository with more than 100
  stars and the same name in the same domain is a collision. When the audience reads the
  target from the owner's profile, only the owner's repositories count.
- PyPI, when the target is a published Python package:
  `curl -s -o /dev/null -w '%{http_code}' https://pypi.org/pypi/<name>/json`. `404` means free.
- Skill registry, when the target is a published skill: `npx skills find <name>`. A skill with
  the same name and many installs is a collision.
- npm, when the target is a published JavaScript package: `npm view <name> name`. An error
  means free.

# 5. Score

Score each remaining candidate 0 to 2 on each criterion:
- **Guessable**: a reader who sees only the name states the purpose statement.
- **Durable**: the name stays true after the likely changes (model swap, new client, scope
  growth by one feature) and after the planned features.
- **Typeable**: 2 for at most 16 characters, 1 for 17 to 24, 0 for more than 24. Subtract 1
  when the spelling does not follow from the pronunciation.
- **Specific**: the name separates the target from what the audience compares it with: the
  owner's other repositories and the common names in the domain. A name that fits most
  projects in the domain scores 0.
- **Distinct**: no collision in step 4, distinct from siblings.

Show a table of the top 5 with the scores and the total. Present the top 3 with one sentence
each on what the name tells a new reader. Add a **wildcard**: the best-scoring candidate from
a direction the top 3 do not cover, with one sentence on the direction it opens. Recommend
one. On a tied total, prefer the candidate that covers more parts of the purpose statement.
When the current name passes the rules in step 2 and scores within 1 point of the best
candidate, recommend keeping it.

From the second round on, add a comparison table: the top 2 of every earlier round, the
current round's best and the current name, rescored against the latest purpose statement,
with the round each came from. Recommend from this table.

End with a question with two parts: is one name final, and if not, which candidates or tokens
the user likes and which they dislike, and why. The step is done when the user confirms one
name as final. Any other reply starts a new round, including approval of several names ("X and
Y are close"). Before starting the round, add the likes and dislikes to the purpose statement.
Disliked candidates and tokens stay out of later rounds. Each new round has at least 12
candidates:
- 60% **exploitation**: variants of the liked candidates and tokens, each adding, swapping or
  combining one token.
- 40% **exploration**: candidates built from tokens, strategies or category terms no earlier
  round used.

# 6. Rename

Rename only after the user confirms the name as final. List the places that hold the name:
- Repository: directory, `gh repo rename`, git remote URL, package metadata (`pyproject.toml`
  `name` and `[project.scripts]`, `package.json` `name`), import package directory, imports,
  README title, CI files, editor configs with absolute paths.
- Skill: skill directory, frontmatter `name`, symlinks in `~/.agents/skills` and
  `~/.claude/skills`, the collection README, references in other skills.

Find the remaining occurrences with `grep -rn <old-name>` and `grep -rn <old_name>`. Report each
place changed and each place left for the user.
