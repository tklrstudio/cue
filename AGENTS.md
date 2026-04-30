# AGENTS.md

This is the neutral instruction entrypoint for any agent working in this repo.

Read these files in order:
1. `.living-systems/GLOBAL_AI_INSTRUCTIONS.md` — universal rules
2. `LOCAL_AI_INSTRUCTIONS.md` — workspace-specific context

Follow the Open Brain policy defined in the instruction stack through the best compliant path available to this agent.

Agent-specific shims such as `CLAUDE.md` may add startup mechanics, but they do not replace the constitutional instruction stack above.
