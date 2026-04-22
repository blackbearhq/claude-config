---
description: Implement a feature or fix using TDD workflow. Delegates to the `implement` agent.
---
Invoke the `implement` agent with the user's request: $ARGUMENTS

The agent handles entry-mode parsing (card / issue / free-form), classification, branching, TDD cycle, review, and PR prep.

Glacier board sync is automatic via the `glacier-sync` skill hooks — no manual coordination needed here.
