---
name: commit-properly
description: Commit the changes on the current branch as tested, coherent commits with conventional messages
---
I want to commit changes to the current git branch, making sure they are correct and subdivided into coherent units of change. 

# Branch

Check the current branch before committing anything. If it is `main`, `master`, `devel` or
`develop`, stop and warn me that the commit would land on a shared branch, then ask whether
to continue there or create a feature branch first. Branch-based development is the standard;
committing to a shared branch is a deliberate exception, not the default.

# Commit Content

Analyze changes:
 - If changes are coherent, and of appropriate size for a commit, use a single commit
 - If changes are not homogeneous, propose splitting into multiple homogeneous commits and as for approval.

# Testing

If the codebase includes a minimal set of tests, run the test on the whole set of commits. 
If they do not pass, report and ask for input.
If these pass, re-run the tests on each proposed commit and adjust the subdivision of changes accordingly.
 

# Commit Message
For each commit, propose a commit message following this structure:

<type>: <subject>

<Optional body>

- <type> can be one of among [feat, fix, docs, style, refactor, test, chore]
- <subject> is a 50 character max description. Use imperative tone to describe what commit does. 
- <body> is omitted by default. Write one only when 50 characters genuinely cannot carry
  the point; can be multi-line, wrap at 72 characters.

Use plain words in the subject. Avoid tooling jargon: write "tests", not "fixtures".

