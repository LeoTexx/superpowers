# Orchestration stack — design

Date: 2026-09-08
Status: approved, not yet implemented
Fork: `LeoTexx/superpowers`, branch `orchestration-stack`, based on `b36e082` (v6.3.0)

## Problem

Superpowers 6.3.0 knows one way to use more than one agent: dispatch several
Agent-tool subagents in the same message. Claude Code now has seven, and they
differ in what they cost, what they isolate, and who coordinates them.

The skills do not merely omit the new mechanisms. Three of them state rules that
are now wrong:

1. `dispatching-parallel-agents` says subagents "should never inherit your
   session's context or history". Forked subagents inherit it by design.
2. `dispatching-parallel-agents` says do not parallelize when agents "would
   interfere with each other (editing same files)". Shared state is now an
   isolation problem, not a reason to serialize.
3. `subagent-driven-development` says "Never dispatch multiple implementation
   subagents in parallel (conflicts)". The parenthesis names the real
   constraint, and worktree isolation removes it.
4. `subagent-driven-development` says to "wait in bounded stretches (five to ten
   minutes)" and never poll. The harness now re-invokes the controller when a
   subagent finishes, so both the polling ban and the timed waiting are obsolete.

A separate personal skill, `maestro`, implements a Manager/Developer/Reviewer
system out of shell scripts, JSON work orders under `.agents/`, and prompts the
user pastes into separate terminals. Agent Teams, `claude agents`, worktrees and
cross-session messaging now provide every part of it natively.

## Verified capability baseline

Checked against Claude Code 2.1.263 on this machine, not against documentation.

| Capability | How it is reached | Evidence |
|---|---|---|
| Parallel subagents | several Agent calls in one message | tool schema |
| Forked subagent | `Agent(subagent_type: "fork")` | tool schema |
| Worktree-isolated subagent | `Agent(isolation: "worktree")` | tool schema |
| Background session | `claude --bg`, `/background` | `claude --help` |
| Background dashboard | `claude agents`, `attach`, `logs`, `stop`, `rm`, `respawn` | `claude --help` |
| Worktree session | `claude -w <name>`, `--tmux` | `claude --help` |
| Cross-session messaging | `ListAgents`, `SendMessage` | tool schema |
| Agent Teams | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` **or** `claude --agent-teams` | binary strings |
| `/batch` | registered command, decomposes into N self-contained units | binary strings |
| Dynamic workflows | `Workflow` tool, opt-in only | tool schema |
| Cloud multi-agent review | `claude ultrareview` | `claude --help` |

Two findings that the public documentation does not carry:

- Agent Teams is reachable through an undocumented `--agent-teams` CLI flag, so
  it can be tried per-session without editing `settings.json`. It is also behind
  a server-side gate that defaults to on.
- `CLAUDE_CODE_EXPERIMENTAL_OBSERVER_AGENTS` exists as a second experimental
  agent mode. It is out of scope here and is recorded only so it is not
  rediscovered as a surprise.

## What this design does not change

`using-git-worktrees` already prefers native worktree tools and names
`EnterWorktree`. `writing-plans`, `test-driven-development`,
`systematic-debugging` and `brainstorming` are unaffected by orchestration and
stay as they are. Not changing them keeps the diff mergeable against upstream.

`writing-plans` is the one deliberate deferral. Plans today emit a linear task
list with no dependency graph and no per-task file ownership, so an executor
cannot prove two tasks are disjoint and must serialize even when the work is
not coupled. Fixing that would unlock parallel execution properly. It is left
out to keep this change reviewable; it is the obvious next spec.

## Architecture

Five skills change. One is new.

```
orchestrating-agents  (NEW)     which mechanism, and what isolation
        |
        +-- dispatching-parallel-agents   (REWRITE)  the subagent tier
        +-- subagent-driven-development   (PATCH)    plan execution
        +-- executing-plans               (PATCH)    plan execution, no subagents
        +-- maestro                       (REWRITE)  the teams tier
```

`orchestrating-agents` holds the routing decision and nothing else. The tier
skills hold the mechanics of their own tier. This keeps each skill answerable
for one question, and it means a future mechanism adds a row to one table
instead of a paragraph to four skills.

### Two decisions, not one

The central idea, and the thing the current skills conflate: **who does the work
and where they do it are separate choices.** Today's guidance folds isolation
into the parallelism decision and concludes "don't parallelize". The correct
conclusion is "parallelize, in worktrees".

Every routing rule is therefore keyed to a predicate the agent can actually
check before dispatching, never to a judgment about task difficulty.

### Routing table

| Predicate | Mechanism |
|---|---|
| Tasks only read | parallel subagents, one checkout, no isolation |
| Tasks write, file sets provably disjoint | parallel subagents, one checkout |
| Tasks write, file sets overlap or are unknown | one worktree per agent |
| Agent needs this conversation's reasoning | `Agent(subagent_type: "fork")` |
| Work is independent of your next step | background session |
| Workers must agree on a contract mid-flight | Agent Teams |
| Same mechanical change, many separately reviewable units | `/batch` |
| Fan-out too large to hold, or needs verify/repeat loops | Workflow, explicit opt-in |

"Provably disjoint" is deliberate. A plan that does not state which files a task
owns has not proved anything, and the unknown case routes to worktrees.

### Cost ladder

Each level costs more than the one above it in tokens, wall-clock, and the
user's attention. The rule is to take the lightest mechanism whose predicate
holds, and the failure mode the skill guards against is reaching for a heavier
one because it feels more thorough. Spawning a team for a three-file change
loses to doing it in the session.

## Per-skill changes

### orchestrating-agents (new)

A routing skill. Contents: the two-decisions principle, the routing table, the
cost ladder, the isolation rules, cross-session messaging (`ListAgents` /
`SendMessage`, and the rule that a finished agent notifies you), and a
rationalization table aimed at over-orchestration.

One flowchart, for the isolation decision only, because that is the one branch
where the wrong answer corrupts a working tree. Everything else is a table.

Description field carries triggering conditions only, no workflow summary, per
the `writing-skills` rule that a description summarizing a process becomes a
shortcut agents take instead of reading the skill.

### dispatching-parallel-agents (rewrite)

Keeps its name; upstream compatibility and the name is right. Changes:

- Context is a choice, not a rule. Constructed context for a fresh subagent,
  inherited context for a fork. State when each is correct.
- "Do not use when agents would interfere" becomes "interference means you need
  isolation" and routes to `isolation: "worktree"`.
- Adds forks with their real use case: challenge a conclusion, or explore an
  alternative, from a context the agent already holds.
- Keeps the existing rule that several dispatches in one message run
  concurrently, and the existing prompt-quality guidance, which is still good.
- Hands off to `orchestrating-agents` when the fan-out outgrows the tier.

### subagent-driven-development (patch)

Three edits. The rest of the skill is sound and stays untouched.

1. The parallel-implementer ban becomes conditional: implementers may run in
   parallel when the plan states disjoint file ownership, or when each runs
   under `isolation: "worktree"`. Otherwise serial. Reviewers never write, so
   reviews of separate tasks always parallelize.
2. The waiting paragraph is replaced. A finished subagent notifies the
   controller; the controller does local work meanwhile and never invents a
   pending agent's result.
3. The final whole-branch review names `claude ultrareview` as an option,
   worded as a recommendation to the human. A controller session cannot launch
   it: it is user-triggered and separately billed.

Edit 1 loosens a prohibition, which is the risky direction. It is written as a
prohibition with red flags, not as soft guidance, because the failure it must
prevent — parallelizing implementers whose files collide — is a discipline
failure, and `writing-skills` is explicit that soft guidance loses to a
competing incentive.

### executing-plans (patch)

Its note currently says: if you have subagents, use subagent-driven-development.
It gains the other two routes — hand a self-contained plan to a background
session, or a multi-stream plan to a team — and points at
`orchestrating-agents` for the choice.

### maestro (rewrite)

Personal skill at `~/.claude/skills/maestro/`, mirrored to `~/.agents/skills/`.
Not part of the fork.

Deleted: `manager-tools.sh`, `developer-tools.sh`, `reviewer-tools.sh`,
`work-orders/`, `submissions/`, `reviews/`, `feedback/`, `manager-state.json`,
`dev-workspace/`, the three copy-paste terminal prompts, and the whole `.agents/`
protocol. Every one of them is now native.

Written: how to run a team once `orchestrating-agents` has said a team is the
answer.

1. Preflight — `claude --agent-teams`, or the env var for a durable setting.
2. Shape — roles come from the dependency graph, not from job titles.
3. Ownership — exactly one owner per file or package. This is the collision rule
   and it is the reason a team is safe without worktrees.
4. Contracts — name what each teammate hands the next before spawning, because
   a contract discovered mid-flight is a rewrite.
5. Spawn — the prompt template.
6. Drive — `claude agents`, `SendMessage`, the shared task list.
7. Fallback — if teams is unavailable, `claude --bg -w <stream>` per stream plus
   cross-session messaging. Same shape, more manual.

Frontmatter is corrected. It currently carries `tools:` and `color:`, which are
agent fields and do nothing in a skill, and `disable-model-invocation: true`,
which is why the skill has never surfaced. It becomes discoverable, with an
explicit gate in the body: never spawn a team without the user's go-ahead.

## Install

The fork must replace the official plugin, not sit beside it. Both ship the same
SessionStart hook and the same skill names, and two copies would double-inject.

- `.claude-plugin/marketplace.json`: marketplace name `superpowers-dev` →
  `superpowers-leo`.
- `.claude-plugin/plugin.json`: version `6.3.0` → `6.4.0-leo.1`.
- `settings.json`: add `superpowers-leo` to `extraKnownMarketplaces`; set
  `superpowers@claude-plugins-official` to `false`; add
  `superpowers@superpowers-leo: true`.
- `upstream` remote stays, so later releases merge rather than being
  re-forked.

## Testing

`writing-skills` imposes its Iron Law: observe the failure before writing the
guidance. Each skill is tested at the tier that matches what it is.

`orchestrating-agents` is a routing skill, so it is tested by retrieval and
application: give a fresh subagent a scenario and record which mechanism it
picks, without the skill and then with it. Five scenarios, one agent each, never
batched:

1. Six independent test failures across three files.
2. Migrate sixty controllers from one validation library to another.
3. Two streams of work that would both edit `package.json`.
4. Challenge the current diagnosis while the main session implements the fix.
5. Audit a hundred-plus route handlers for missing authorization.

The `subagent-driven-development` change is a discipline loosening, so it is
tested adversarially in the opposite direction: give an agent a plan whose tasks
share a file and confirm it still refuses to parallelize them. That test failing
is worse than any of the routing tests failing, because it corrupts a tree
rather than wasting tokens.

`maestro` is tested by application: does the agent shape a sane team and assign
non-overlapping ownership.

Baselines are recorded verbatim, including the rationalizations, and the
rationalization tables are built from what the runs actually produce rather than
from what seems likely.

## Out of scope

- `writing-plans` dependency graphs and file ownership. Deferred, named above.
- `CLAUDE_CODE_EXPERIMENTAL_OBSERVER_AGENTS`.
- Any change to TDD, debugging, brainstorming, or worktree skills.
- Upstream pull requests. The fork is the deliverable; contributing back is a
  later decision.
