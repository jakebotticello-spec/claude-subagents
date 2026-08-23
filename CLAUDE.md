@C:/claude-reference/universal/JAKE-RULES.md
@C:/claude-reference/universal/JAKE-STACK.md

# CLAUDE.md — Subagents

**Repo root:** `c:\claude-subagents\code` · **Librarian group:** `claude-subagents`
(alias `subagents`) · **Track name:** Subagents.

The universal layer above is the floor. This file is the project layer; where they
disagree for this project's scope, this file wins (JAKE-RULES §15).

---

## What this project is

A role-based specialist team — Scribe, Database, UI, API Integrator, Walker,
Relay-checker — orchestrated by the Cowork/Desktop window (OC), so OC keeps the plan
and stops needing to hold every body of knowledge itself.

**Spec of record:** `canon/build-specs/Agent_Team_Spec_v1.0.md`. **Status: proposed and
entirely unbuilt.** Everything in that spec that is not in its §3 is a TARGET, not a
finding. Do not cite a spec number as a measured result.

---

## The constraint that shapes every design here

Read this before proposing anything, and re-read it if you find yourself drawing a
diagram with agents calling agents:

| Fact (verified in-container 8-23-26) | Consequence |
|---|---|
| `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH=1` | Subagents cannot spawn subagents. Star topology is enforced, not chosen. |
| `~/.claude/skills/synced/` holds only claude.ai account skills | The ONLY channel that reaches a Cowork session. |
| `/root/.claude/plugins/synced/` is EMPTY | Locally-installed plugins do not sync in. |
| No `agents` dir exists anywhere in the Cowork container | Custom subagent files do not reach Cowork. |

**Therefore: a specialist that OC can invoke must be a claude.ai ACCOUNT SKILL with
`context: fork` — not an agent file, not a local plugin.** A seat authored as
`.agents/*.md` is a **CC-side** seat and cannot be called by OC.

**`background: false` is mandatory on any seat that writes.** A backgrounded fork
auto-denies tool calls that would otherwise prompt for permission — so a delegated
write that needs approval **fails silently while reporting success.** On a canon-write
seat that is the worst available outcome.

Economic ceiling, Jake's ruling: **~1.2× total tokens is acceptable if it protects OC's
context. 10× is not.** No parallel fan-out — that rule is what makes 1.2× reachable.

---

## Layout

```
c:\claude-subagents\code\
  CLAUDE.md            this file
  .claude\             Claude Code permissions bundle
  .codex\              Codex permissions bundle (config.toml, not settings.local.json)
  .agents\             CC-side subagent definitions — do NOT reach Cowork, see above
  canon\
    BOOT.md            ignition doc, served by `boot claude-subagents`
    build-specs\       Agent_Team_Spec_v1.0.md
    changelog\         per-session dated files
    handoffs\          session handoffs
    foundation\        (empty — foundation doc not yet authored)
    graveyard\         killed ideas, with reasons
```

## Write scope

· **OC authors canon** (`canon/**`). CC executes and does not author canon without
  explicit instruction (JAKE-RULES §7.6).
· CC writes inside `c:\claude-subagents\code` only. Never repo root of another project.
· ⚠ **This tree is NOT a git repository** (observed 8-23-26 — no `.git` anywhere).
  Disk is the only state. There is no commit to fall back to, so a destructive edit is
  final — back up to `c:\claude-MCP\_scratch\` before any structural change.
· The librarian serves this group live off disk. A save is instantly visible: no git,
  no cache, no propagation delay.

## Standing invariants — flag any violation as a finding

· A seat that authors content is out of scope. **The Scribe lands supplied text and
  verifies at bytes** — it does not write prose of its own.
· Every role file pulls the universal layer before acting.
· Specialists never call each other. OC invokes every specialist. Star, always.
· No seat ships without its guard fired against a real failure first — JAKE-RULES §10:
  a guard asserted in docs but never fired is a comment, not a guard.

*Last updated: 8-23-26 — group stood up: librarian registration, watch, and the CC/Codex/.agents bundles.*
