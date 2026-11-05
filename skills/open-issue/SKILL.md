---
name: open-issue
description: File the just-found bug/task as a GitHub issue from current context
argument-hint: [what you noticed]
allowed-tools: Bash(gh issue create:*), Bash(gh issue view:*)
---
Create a GitHub issue using the context loaded in this session, via
`gh issue create --title "<concise title>" --body "<structured body>"`.
Body: summary, suspected root cause / location, reproduction,
expected vs actual, how to verify. Report the issue number and URL.
Ask at most one question, only if genuinely ambiguous.
