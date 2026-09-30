---
description: Implement a feature or fix using TDD workflow. Delegates to the `implement` agent.
---
Invoke the `implement` agent with the user's request: $ARGUMENTS

The agent handles entry-mode parsing (card / issue / free-form), classification, branching, TDD cycle, review, and PR prep.

The agent runs in its own git worktree and moves the Glacier card explicitly via the `glacier-sync` skill (In Progress at branch creation, In Review after the PR opens). To run several cards in parallel, use `/triage` instead.
