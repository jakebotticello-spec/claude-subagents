# The Agent Team — Project Spec v1.0

**Status:** proposed, unbuilt. MVP is the Scribe, built for measurement.
**Author:** OC (Cowork), 8-23-26. **Grounded in:** four external sources, read in full, cited inline.
**Governs:** how OC farms specialist work out so its context window stays unsaturated.

---

## 1. The why, and the constraint that shapes everything

**Goal:** OC keeps orchestrating. It stops needing to know everything about everything.
Specialists hold the bodies of knowledge; OC holds the plan.

**Jake's economic ruling (8-23-26), and it is the binding constraint:**

> Up to ~1.2x total tokens is worth it if it protects OC's context. 10x is not.

The literature's numbers look fatal to that: Anthropic reports multi-agent systems use
**~15x chat tokens**, and their own guidance says **"3-10x more tokens than single-agent
approaches for equivalent tasks."**

**They are not fatal, and the reason is the whole architecture.** Anthropic names exactly
three sources of that overhead:

> "duplicating context across agents, coordination messages between agents, and
> summarizing results for handoffs."

Two of the three are FAN-OUT costs. This design eliminates them by construction:

| Overhead source | Eliminated how |
|---|---|
| Duplicating context across agents | **No fan-out.** One specialist at a time, serially. |
| Coordination messages between agents | **Star topology.** Agents never talk to each other. |
| Summarizing results for handoffs | Unavoidable — bounded by the return contract (§4). |

What remains is **cold-start**: a subagent that has to rediscover context OC already had.
That is a briefing problem, and §3 is the fix. **A well-briefed single serial specialist
is ~1.0-1.3x total tokens.** The 15x figure describes 3-5 parallel researchers, which
this system never does.

**Corroborating measurement (Grover/Chanl, real repo, not fan-out):** rules overhead for
one agent spanning all layers **16,800 tokens; a scoped backend subagent 4,300.** ~4x
reduction on the rules alone, before any code is read.

**Therefore, a hard rule of this system: NO PARALLEL FAN-OUT.** If a task seems to want
three specialists at once, that is a signal the task is wrongly scoped, not a signal to
spawn three. Fan-out is where the 10x lives.

---

## 2. Architecture

**Orchestrator:** the Cowork/Desktop window. Ruled by Jake — it has held this seat for
eight months and keeps it.

**Topology:** star. **OC invokes every specialist, and only when needed.** Specialists
never invoke each other. This is not merely preferred — `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH=1`
is set in the Cowork container, so subagents *cannot* spawn subagents. Verified live.

**Distribution — this is settled and it is narrower than we hoped.** Verified by reading
the Cowork container's own filesystem:

- `CLAUDE_CODE_SYNC_SKILLS=1` — claude.ai **account skills** sync in, to `~/.claude/skills/synced/`.
- `CLAUDE_CODE_SYNC_PLUGINS=1` — but `/root/.claude/plugins/synced/` is **EMPTY**. Jake's
  three locally-installed plugins are not there.
- **No agents directory exists anywhere.** Custom subagent files do not reach Cowork.

**Consequence: specialists reaching a Cowork orchestrator must be claude.ai ACCOUNT SKILLS
with `context: fork`.** Not agent files. Not local plugins.

**The cost of that, stated honestly.** Per the docs' own comparison:

| Shape | System prompt | Task |
|---|---|---|
| Skill with `context: fork` | from the generic agent type | the SKILL.md body |
| Subagent with `skills:` | the subagent's own body | OC's delegation message |

A forked skill **cannot carry a persona as a system prompt.** The persona has to live in
the body as an instruction preamble. Weaker, and it is the only channel that reaches OC.

**⚠ And a fork warning that kills the naive version of every seat here:**

> "`context: fork` only makes sense for skills with explicit instructions. If your skill
> contains guidelines like 'use these API conventions' without a task, the subagent
> receives the guidelines but no actionable prompt, and returns without meaningful output."

**Every seat must be a TASK, not a body of knowledge.** "You are a database expert" returns
nothing. "Given this question, answer it from the schema and return X" works.

---

## 3. The dispatch packet — what OC sends

Anthropic's four mandatory fields, verbatim: **"an objective, an output format, guidance on
the tools and sources to use, and clear task boundaries."** Without these, *"agents duplicate
work, leave gaps, or fail to find necessary information."*

Konishi on why the packet is the entire inbound channel:

> "The subagent does not see your conversation history" or "the files you already read."
> "If the subagent needs a file path, an error message, a branch name, or a decision already
> made in the main conversation, that information has to be in the delegation prompt, because
> nothing else crosses the boundary automatically."

**Template (adapted from Grover, plus the piece no external source has — the librarian):**

```
OBJECTIVE
  <one sentence, the outcome not the activity>

CONTEXT YOU DO NOT HAVE
  <paths, error text, decisions already made, the constraint that must not be violated>

RULES THAT BIND YOU
  Pull the universal layer before you act: boot universal, or
  fetch('universal','JAKE-RULES.md'). You are bound by it. Specifically <the 2-3 sections
  that bite on THIS task>.

TOOLS AND SOURCES
  <which tools, which groups, what NOT to touch>

BOUNDARIES
  <what is out of scope; what to do instead of guessing>

RETURN FORMAT
  <exact shape — see §4>
```

**The librarian pull is the piece that makes these Jake's specialists rather than generic
hires.** A stock database agent does not know `first_name` + `last_name` is a hard universal
(JAKE-RULES §9), does not hold the Governor, and will break both on turn one.

**Grover's dependency bridge**, for when task 2 needs task 1's output — OC extracts and
injects, because nothing else carries it:

```
CONTEXT FROM PREVIOUS TASK (backend):
  New field: lastActive (Date, optional, indexed). Returned in GET /agents/:id.
  Format: ISO 8601 string after JSON serialization.
```

---

## 4. The return contract — what comes back

Konishi, on the only outbound channel:

> "Only the subagent's final message returns to the parent." Every intermediate tool call,
> every file read, every line of test output "stays inside the subagent's context and is
> never injected into yours."

And the honest framing of what delegation actually buys:

> "a filter, not a wall with no cost on either side" — the summary still lands in OC's context.

**So the return contract IS the savings.** A specialist that returns everything it read has
moved zero context and added a hop. Konishi's rule: *"Instruct each subagent to return a
summary, not a transcript"* — report the failing tests with their errors, not the test run.

**Universal return shape for every seat in this system:**

```
RESULT      <the answer/outcome, bounded — no transcripts, no full file bodies>
RECEIPT     <what proves it: byte counts, hashes, a diff, a command's exit>
ROUTING     <one line: "touched schema - recommend SECR" | "no review needed">
UNKNOWNS    <what it could not determine, named as unknowns per JAKE-RULES §5.1>
```

**ROUTING is what makes the star topology work.** OC does not need to hold the work to know
it needs review — it needs one line. The routing intelligence travels in the return, not in
OC's memory.

---

## 5. The roster, scored against the actual test

Anthropic's rule, verbatim: **"group work by what context it requires, not by what kind of
work it is."** And their named anti-pattern:

> "Dividing by type of work (one agent writes features, another writes tests, a third reviews
> code) creates constant coordination overhead."
> "Planning, implementation, and testing of the same feature share too much context" and
> belong in one agent.

Their quantitative threshold for context protection: it pays when a subtask generates
**over 1,000 tokens of largely irrelevant information.**

| Seat | Context boundary? | >1k irrelevant tokens? | Verdict |
|---|---|---|---|
| **Scribe** | canon corpus + mechanical edit | Yes — a 185KB doc | **PASS — MVP** |
| **Database** | the schema | Yes | PASS |
| **API Integrator** | third-party docs | Yes — purest case in the roster | PASS |
| **UI** | design system + FE conventions | Yes | PASS |
| **Relay-checker** | mailbox poll output | Yes | PASS |
| **Walker** | live-path output, logs, clicks | Yes | PASS |
| **SECR** | the codebase's shape + the Governor | **Borderline** | **FLAGGED — §5.1** |

### 5.1 The SECR problem, stated honestly

"Reviews every other agent's work" is review-as-a-phase, which is the named anti-pattern.

**The counter-argument, which I think holds:** Anthropic is warning about splitting ONE task
into sequential agent phases. The SECR is not a phase of one task — it is a standing role
holding a body of knowledge (the Governor; how the whole codebase fits) that genuinely differs
from any builder's. That is a context boundary.

**But the failure mode they describe is the one the Governor was written about.** A reviewer
whose standing job is finding more to fix is how forty sessions happened.

**Condition of build:** the SECR ships only with Governor classing in its RETURN contract —
every finding tagged *protects money/data/a user* or *protects against a hypothetical* —
and a hard instruction that a clean review is a valid, expected result. Without that it is
a spiral engine.

### 5.2 Rejected, with reasons

- **System Architect** — that is OC's seat. Two things designing, neither holding the whole.
- **QA Automation** — a standing mandate to "try and break the software" against a
  pre-revenue system is the spiral with a job title.
- **Performance Optimizer** — Governor bait. Zero-traffic.
- **Technical Writer** — collides with §7.6; docs are canon and canon has an author.
- **Frontend/UX split** — creates a handoff OC must mediate, putting context back.

---

## 6. MVP — THE SCRIBE

**Why this one first:** highest mechanical cost removed, and the only seat whose saving is
trivially measurable, because its return is a diff.

**What it is:** the seat that lands canon and reference edits so OC never loads the file.

**⚠ THE RULE THAT KEEPS IT LEGAL (JAKE-RULES §7.6):** *the Scribe authors nothing.* OC or a
specialist supplies the exact text. The Scribe locates, edits surgically, verifies at bytes,
and reports the diff. **Typesetter, not writer.** OC still authors, so §7.6 holds; OC just
stops paying the mechanical cost, which was the actual expense.

Its job description is already in JAKE-RULES §6: *"for huge canon/reference docs where full
regen risks clobbering history (a 185KB ANCHOR), surgical `str_replace` is correct."*

**Scope:** reads AND writes canon. History lookup ("did we already try this?" — §18's
graveyard rule) belongs to the same seat. One owner for canon; splitting recreates the
handoff problem.

### 6.1 ⚠⚠ THE HAZARD THAT COULD SINK THIS SEAT — READ BEFORE BUILDING

Konishi's headline pitfall, verbatim:

> "a subagent cannot use AskUserQuestion, and a background subagent auto-denies any tool call
> that would otherwise prompt for permission," so delegated edits requiring approval
> **"will silently fail while the subagent reports as if it succeeded."**

**This is a canon-write seat. Silent failure reporting success is the worst available outcome** —
worse than not building it, because OC would believe canon landed when it did not.

**Mandatory mitigations, all three:**

1. **`background: false`.** Forked skills default to background, and *"a backgrounded fork
   runs with the narrower tool set that applies to background subagents."* The Scribe's whole
   job is a write tool. Foreground, full tool set, permissions can actually prompt.
2. **Byte verification in the RECEIPT, always.** The Scribe re-reads what it wrote and reports
   actual bytes on disk. JAKE-RULES §2 already requires this of OC ("verifies the write at
   bytes afterward"); it transfers with the job.
3. **No receipt = FAILED.** If the Scribe cannot verify, it reports failure. It never reports
   success on an unverified write. This is the guard, and per §10 a guard that never fires is
   a comment — so **fire it once deliberately** during the MVP: hand it an unwritable path and
   confirm it reports failure rather than success.

### 6.2 Measurement protocol — the actual point of the MVP

Pick one real canon edit. Run it twice.

**Baseline (OC does it):** load the file, locate, `str_replace`, verify. Record:
- `B_ctx` — tokens added to OC's context
- `B_total` — total tokens spent
- `B_wall` — wall-clock
- correctness — did the edit land, byte-verified

**Treatment (Scribe does it):** Record:
- `T_ctx` — tokens the return added to OC's context
- `T_total` — OC's dispatch + the subagent's full spend
- `T_wall`, correctness — same

**Success criteria, all three:**

| Metric | Threshold | Why |
|---|---|---|
| `T_ctx / B_ctx` | **< 0.20** | the point of the exercise |
| `T_total / B_total` | **<= 1.2** | Jake's ruling, 8-23-26 |
| correctness | **equal** | non-negotiable |

**If `T_total/B_total` lands between 1.2 and 2.0**, the finding is that cold-start briefing
is too expensive — fix the dispatch packet before condemning the pattern.
**If `T_ctx/B_ctx` > 0.5**, the return contract is leaking and §4 needs tightening.
**If either fails after one dispatch-packet revision, the pattern does not pay and we stop.**

Report the six numbers. No seat two until these exist.

---

## 7. Standing hazards for every seat

- **Silent permission failure** (§6.1) — applies to any seat that writes or runs anything.
- **Cold-start waste** — Konishi: *"a poorly briefed subagent will waste its first several
  turns rediscovering context you already had."* Fix the packet, not the pattern.
- **Verbose result bloat** — *"you fanned out five subagents to protect your context, and your
  context is now fuller than before."* The reason fan-out is banned in §1.
- **Rules-depth blindness** (Grover) — *"rules near the top get followed more reliably than
  rules buried 200 lines deep."* Put the 2-3 binding rules in the dispatch packet directly;
  do not rely on the specialist finding them.
- **Don't delegate small already-decided edits** — *"Spinning up a subagent to make a two-line
  edit costs more in cold-start latency than it saves."*

## 8. Deferred, with named triggers

| Item | Trigger | Home |
|---|---|---|
| Seats 2-7 | The Scribe's six numbers clear §6.2 | this track |
| SECR specifically | Above, PLUS a written Governor-classing return contract | this track |
| Plugin packaging | Three or more seats stable and unedited for two weeks | this track |
| Subagent-shape specialists (CC-side) | If Cowork ever syncs agents or plugin agents | this track |

## 9. Sources

1. Anthropic Engineering — *How we built our multi-agent research system*
2. Anthropic — *When to use multi-agent systems (and when not to)*
3. Hidekazu Konishi — *Claude Code Subagents and Multi-Agent Orchestration Guide*
4. Dean Grover (Chanl) — *Claude Code subagents and the orchestrator pattern*

Rejected: the "Jake Morrison" audiobook — Independently Published, Amazon Virtual Voice
narration, no author identity, five near-identical 2026 titles including the same book twice.
AI-generated. Named here so nobody re-finds it and thinks it is a source.

---

*v1.0 — 8-23-26. Unbuilt. Nothing in this document has been tested except the two
environment facts in §2, which were verified live in the Cowork container.*
