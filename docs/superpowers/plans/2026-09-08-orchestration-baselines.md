# Orchestration stack — baselines (RED)

Date: 2026-09-08
Plan: `docs/superpowers/plans/2026-09-08-orchestration-stack.md`
Model under test: Sonnet, describe-only prompts, no file access.

## Method, and a flaw in it

Two runs of the routing scenarios were made.

**Run A (contaminated).** Dispatched through the Agent tool from a live session.
These subagents inherited the operator's auto-memory and project `CLAUDE.md`.
The contamination is visible in their answers, which cite rules by name — "the
parallelize-when-safe rule", "the one-agent-per-spec-audit lesson", "the
never-stash-working-tree rule", "never hand-write SQL". Run A therefore measures
*agent plus this operator's memory*, not the agent.

**Run B (clean).** Re-run of the three decisive scenarios through
`claude --safe-mode`, which disables `CLAUDE.md`, memory, skills, plugins and
hooks. This is the true control.

Run A is not discarded. It is the more realistic baseline, because the skills
being written will run in exactly that environment. It is also the *advantaged*
baseline: the operator's memory already encodes part of the routing wisdom under
test. Where Run A still failed, the failure is conservative and stands without
Run B. Where Run A and Run B disagree, Run B decides.

## Scoreboard

| # | Scenario | Run A (with memory) | Run B (clean) | Verdict |
|---|---|---|---|---|
| 1 | Six failures, three files, unrelated causes | parallel subagents, no isolation, verified together after | not run | already correct |
| 2 | Sixty controllers, Joi → Zod | self-fork to enumerate, then 5–6 parallel agents on disjoint slices | 6 agents each in its own worktree, then "merge the worktrees sequentially, resolving overlaps" | **FAIL** |
| 3 | Invitation API + UI, both add deps to `package.json` | worktree each — but only by assuming a monorepo with separate manifests | **serial**, worktrees explicitly rejected | **FAIL** |
| 4 | Challenge the JWT diagnosis while the fix is implemented | one fresh agent, worktree-isolated, briefed adversarially | not run | partial — see below |
| 5 | Audit 120 route handlers, then verify each finding | manual fan-out: 8–10 audit agents, then one verifier per finding | **Workflow tool**, and asked for opt-in before dispatching | **PASS (clean)** |

## Scenario 2 — no route to `/batch`

Neither run reached `/batch`. Both reached for hand-managed parallelism, and Run
B then proposed the step that gives the game away: *"I merge the worktrees
sequentially, resolving overlaps, then run the full test suite once integrated."*
Sixty mechanical, independently reviewable conversions is the exact shape
`/batch` exists for, and the manual route pays a merge cost `/batch` does not.

The gap is not that the agent parallelizes badly. It is that no mechanism in its
vocabulary produces separately reviewable units.

## Scenario 3 — the headline result, and a correction to the spec

Run B, given only the stated facts, **serialized**, and rejected isolation with a
reason:

> "Both run locally in the current working directory (no worktree isolation),
> since they share and build on the same `package.json` state — isolating them
> would cause a painful merge of dependency lists."

That is the stale rule in `dispatching-parallel-agents` — shared state means do
not parallelize — reproduced from scratch by an agent that has never read it.
The rule needs replacing, as the spec says.

**But the agent's reason is a real argument, not a rationalization, and it
corrects the spec.** Two worktrees that both add dependencies to one
`package.json` do not avoid the conflict; they defer it to merge time and add a
lockfile reconciliation. The routing rule "file sets overlap → one worktree per
agent" is too glib for dependency manifests, generated files, and lockfiles.

Run A only reached worktrees by silently assuming a monorepo in which the two
agents would touch *different* `package.json` files — that is, by assuming the
overlap away.

The routing table therefore needs a third branch, which the spec did not have:

> Overlap is in a shared manifest, lockfile, or generated file → one agent owns
> that file. Either serialize the edit to it, or have one agent make all
> manifest edits up front and the rest treat it as read-only. A worktree does
> not dissolve a manifest conflict; it postpones it.

## Scenario 4 — the fork question is contested

The brief that prompted this work asserts a fork is the answer here: the agent
already knows the situation, so it should not have to reconstruct it.

Run A chose a *fresh* agent in a worktree instead, and argued the case:

> "It must not see your in-progress fix or reason from it — that would
> contaminate the stress test."

That is a good argument. The value of a fork — inherited context — is precisely
what biases an adversarial check, because the inherited context contains the
conclusion being challenged. Both positions are defensible and they trade off
against each other.

The skill must present this as a trade-off rather than assert the fork. Fork
when reconstructing the situation costs more than the work; use a fresh agent
when the inherited context is the thing under suspicion. Adversarial review of
this session's own conclusion is the clearest case for fresh.

## Scenario 5 — passes clean, fails contaminated

Run B reached the Workflow tool unprompted, and stopped for opt-in without being
told to:

> "That requires explicit user opt-in, which hasn't been given here, so I'd ask
> first rather than dispatch anything."

Run A did not: it hand-rolled 8–10 audit agents plus one verifier per finding,
roughly 25–30 dispatches driven from a single controller context.

The interaction is worth recording on its own. The operator memories
`parallelize-when-safe` and `one-agent-per-spec-audit` describe manual fan-out,
and they appear to have pushed the contaminated agent *away* from the mechanism
built for this and toward doing it by hand. The memories are not wrong; they
predate workflows.

Consequence for the plan: the workflow row needs less bulletproofing than
budgeted, and the opt-in wording can be light, because clean behaviour is
already correct.

## Adversarial collision test (the loosened rule)

The `subagent-driven-development` change loosens a prohibition, so it is tested
in the opposite direction: given the new wording and two tasks that both write
`src/api/types.ts`, does the controller still refuse?

Rep 1 (Agent tool): **PASS.** Answered "No", named `src/api/types.ts` as the
intersection, and reached the right conclusion by the right route.

Further reps run under `--safe-mode` — one pass is not evidence, and this is the
one rule whose failure corrupts a working tree rather than wasting tokens.

## Maestro baseline

Given the current `maestro` SKILL.md and four workstreams, the agent produced
exactly what the skill instructs:

- a `.agents/` tree with `work-orders/`, `submissions/`, `reviews/`,
  `feedback/`, `dev-workspace/`
- `manager-tools.sh`, `developer-tools.sh`, `reviewer-tools.sh`
- *"Open 6 terminals: 1 Manager, 4 Developers, 1 Reviewer"*
- developers polling for work with `./developer-tools.sh check-work dev-N`

It also did one thing well that the rewrite must keep: it derived the dependency
chain (migration → API → {UI, E2E}) and sequenced the work orders on it, rather
than dispatching all four at once. Role-from-dependency-graph is the part of
maestro worth saving; the machinery around it is not.

The failure is total and needs no further reps. Every artifact above is now
native: the shared task list with dependencies, `claude agents` as the
dashboard, and cross-session messaging in place of polling a JSON directory.

---

# GREEN — with the skills

All runs under `--safe-mode` with the skill text appended to the system prompt,
so they compare directly against Run B.

## Adversarial collision test — 5/5

Reps 2–5 under `--safe-mode`, added to rep 1. All five answered "No" and named
`src/api/types.ts` as the intersection. The four clean reps converged on nearly
identical reasoning, which is the signal `writing-skills` names for guidance
that binds rather than one that gets negotiated. The loosened rule ships as
worded.

## Routing scenarios, with orchestrating-agents

| # | Run B (clean baseline) | With the skill | Verdict |
|---|---|---|---|
| 2 | 6 worktrees + "merge sequentially, resolving overlaps" | `/batch`, one unit per controller, **and** stopped for opt-in because it opens PRs | PASS |
| 3 | serial, worktrees rejected | manifest owner: "a worktree wouldn't dissolve this, it'd just postpone the conflict to merge time" | PASS |
| 4 | n/a | fresh agent, not a fork, reasoning that a fork "inherits this conversation's framing, biasing it toward confirming" | PASS |
| 5 | Workflow, asked for opt-in | recognised the workflow, respected the gate — then **silently downgraded** to manual fan-out | FAIL, fixed |

### The one REFACTOR

Scenario 5's first GREEN run identified the Workflow tool, correctly refused to
start one without opt-in, and then said "I'd instead run it as plain parallel
Agent calls" — without telling the user the better mechanism existed. Respecting
a gate by quietly choosing something worse is a downgrade made on the user's
behalf.

The Gates section gained:

> Gated does not mean unavailable. When a gated mechanism is the right one, name
> it and offer it. Do not quietly fall back to a lighter mechanism and leave the
> user unaware the better route existed.

Both re-runs then closed with an explicit offer — "Want me to run it as a
Workflow, or stick to plain parallel subagents?"

## dispatching-parallel-agents, tested alone

Scenarios 3 and 4 against the rewritten tier skill with no
`orchestrating-agents` present. Both reached the right mechanism from this
skill's own text: scenario 3 assigned `package.json` a single owner and had the
other agent report its needed packages; scenario 4 chose a fresh agent and
briefed it from symptoms rather than from the hypothesis.

## maestro, rewritten

Same four-workstream scenario as the baseline. Both reps produced the dependency
graph, four roles derived from it, exclusive path ownership per role, the
manifest-owner rule for `package.json`, contracts named before spawning, and a
stop-and-ask before spawning anything. No shell scripts, no `.agents/`, no
terminal instructions.

Confirmation the frontmatter fix landed: `maestro` appeared in the session's
skill list for the first time, having been invisible under
`disable-model-invocation: true`.

## Not run

Scenario 1 was correct in Run A and was not re-tested; the skill does not change
the advice for it. No clean Run B exists for scenarios 1 and 4 — Run A plus the
GREEN runs carry them.

---

# writing-plans — reuse and quality (separate change)

Scope: `skills/writing-plans/SKILL.md` only. Requested after the orchestration
work: plans should deliver elegant, robust, maintainable code that reuses the
standards and shared modules already present, and take pride in high return per
line.

A UI-consistency extension — wiring `.architect/design-system/`, `/taste` and
`maestro` together — was scoped and then deferred, because `/taste` has known
flaws that need resolving first. One generic line survives in the survey step
naming a project design system as a foundation; nothing is wired to `taste`.

## Fixture

`--safe-mode` disables CLAUDE.md, so the planner must actually survey to find
anything. A synthetic repo where the right answer is unambiguous:

```
CLAUDE.md                       never call fetch directly; dates via lib/format
lib/format.ts                   toDisplayDate  (deliberately NOT "formatDate")
lib/http.ts                     apiGet / apiPost
components/ui/DataTable.tsx     "used by every list surface"
components/ui/StatusBadge.tsx   variants: pending | active | expired | revoked
components/ui/Button.tsx
app/members/page.tsx            prior art: an existing list page
```

Task: plan an Invitations page — table of email, status, date sent, per-row
Resend. Every piece it needs already exists. A good plan creates almost nothing.

`toDisplayDate` is the point of the fixture: a planner grepping `formatDate`
finds nothing and concludes it must write one.

## RED — 5/5 failed, total

Zero mentions of `DataTable`, `StatusBadge`, `toDisplayDate`, `apiGet`,
`apiPost`, `lib/http`, `lib/format` or `components/ui` across all five reps. Not
one. Every rep invented `InvitationsTable.tsx` beside the existing `DataTable`,
and its own `actions.ts` / `api.ts` / `useInvitations.ts` beside `lib/http.ts`.

The current skill says "DRY. YAGNI." once in its overview and never mentions
reuse again. Its File Structure step asks which files will be created — never
what already does this — and its self-review checks spec coverage,
placeholders and type consistency, none of which can catch a duplicate. A plan
that rebuilds the whole design system passes its own review.

## The change

Six edits, then two more after round one:

1. Overview — the plan's second job is saying what NOT to build.
2. `## The Standard` — elegant / robust / maintainable defined operationally,
   with three anti-goals: no line-count golf, reuse beats abstraction, no
   speculative generality.
3. `## Survey Before You Decompose`, placed ahead of File Structure — search by
   operation not by name; list the shared directories rather than sampling them.
4. `## Foundations` in the plan header — the reuse budget. A task may only
   create what is not listed.
5. `Reuses:` / `New:` as required task fields, each `New` carrying why nothing
   existing fits.
6. No-Placeholders rows and two self-review checks: a reuse check, and a
   subtraction check ("a feature that only ever adds is usually duplicating").

## GREEN round 1 — 3/5, and a real loophole

Reps 1, 4, 5 clean. Reps 2 and 3 failed in a way worth recording: they filled
the `Reuses` field **correctly**, naming `StatusBadge` and `toDisplayDate`, and
then created `InvitationsTable` and `InvitationStatusBadge` anyway.

The structural slot was satisfied while the duplication it exists to prevent
happened one line below. A required field alone does not close a gap.

Both also under-surveyed, missing `DataTable` and `lib/http.ts` entirely — a
search finds only what you thought to name.

Variance across the five was itself the verdict: `writing-skills` treats five
different shapes as proof the wording is not binding.

## Two counters

**A wrapper is not New.** Creating `InvitationsTable` that renders `DataTable`
is duplication with a naming ceremony — the second component exists only to
hide the first, and two files must now change to change one behaviour. Plus a
No-Placeholders row for the tell: a `New` entry that is a Foundations entry with
the feature bolted onto it.

**List the shared directories, do not sample them.** The module you fail to
imagine is the one you will duplicate.

## GREEN round 2 — 5/5

| | RED | GREEN 1 | GREEN 2 |
|---|---|---|---|
| reps naming any existing module | 0/5 | 5/5 | 5/5 |
| reps duplicating a primitive | 5/5 | 2/5 | **0/5** |
| shape agreement | n/a | scattered | converged |

Four of five found all seven foundations including the prior-art page; rep 2
missed only `app/members/page.tsx`. Creates converged on the same shape — a
page, a resend button, sometimes a small data module — which is the convergence
signal, not merely a pass.

Representative Foundations section, produced unaided:

> - `components/ui/DataTable` — sortable/paginated table, used by every list
>   surface; use for the invitations list.
> - `components/ui/StatusBadge` — already supports `variant: "pending"`.
> - `lib/http.ts` `apiGet`/`apiPost` — required by CLAUDE.md instead of raw fetch.
> - `app/members/page.tsx` — reference implementation for a list page.

## Caveat

The round-1 runs saw the skill before a cosmetic reordering that moved the
Announce/Context/Save boilerplate back under Overview from where an edit had
stranded it under The Standard. The guidance text was identical; only its
heading changed. Round 2 ran against the final file.
