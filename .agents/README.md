# .agents — subagent definitions for this repo

Custom subagent files (`*.md` with frontmatter) live here and are picked up by
**Claude Code** running in this repo.

⚠ **They do NOT reach a Cowork/Desktop session.** Verified in-container 8-23-26:
`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH=1`, no `agents` directory exists anywhere in
the Cowork container, and `/root/.claude/plugins/synced/` is empty. The only channel
that reaches a Cowork orchestrator is a **claude.ai account skill** with
`context: fork`. A seat built as a file in here is a CC-side seat, full stop.

Receipt: `canon/handoffs/Chat_Session_Handoff_2026-08-23_subagents_S0_to_S1.md` §3.
