---
description: Check current session token usage, rate limit status, and suggest optimizations.
---
Run /cost and /usage. Analyze the output.

Report in three lines max:
1. **Cost + distribution**: Session cost and model split (Opus / Sonnet / Haiku %)
2. **Context + rate limits**: Context window % used; 5-hour and weekly usage from /usage
3. **One recommendation**: If context > 60% → suggest /clear. If Opus > 30% of tokens → suggest delegating exploration to Haiku via the explorer agent. If rate limit > 80% → suggest pausing or switching to a lower-effort model via /effort.

Be direct. No preamble.
