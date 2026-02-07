---
description: Check current session token usage and suggest optimizations.
---
Run /cost and analyze the output.

Report:
1. Current session cost
2. Model distribution (how much Opus vs Sonnet vs Haiku)
3. Context window usage percentage
4. If context > 60%, suggest /clear or compaction
5. If Opus usage > 30% of tokens, suggest delegating more to Sonnet/Haiku

Be concise. Three lines max.
