---
description: Bypass test-gen and verify steps for ui or config issues. Documents the skip reason and reminds to note it in the PR description.
---
You are handling a `[skip-tests]` request.

Follow these steps:

1. **Confirm issue type**: Verify the issue is classified as `ui` or `config`.
   - If classification is `logic`, STOP and refuse: "Cannot skip tests for logic issues. TDD is required."

2. **Prompt for reason**: Ask the user: "What is the reason for skipping tests?" if no reason was provided in the prompt.

3. **Acknowledge skip**: State clearly:
   - Issue type: [ui|config]
   - Skip reason: [reason]
   - Steps bypassed: 4 (test-gen), 5 (verify red), 7 (verify green)

4. **Proceed to implementation**: Jump directly to Step 6 (Implement) from the TDD workflow.

5. **Remind at report stage**: When reporting results, include a `## Skipped Tests` section in the PR description with:
   - Issue type
   - Reason tests were skipped
   - Confirmation that this was a [ui|config] classification

The user's request is: $ARGUMENTS
