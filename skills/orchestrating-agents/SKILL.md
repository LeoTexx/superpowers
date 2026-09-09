---
name: orchestrating-agents
description: Use when work might need more than one agent - several independent investigations, changes that would collide in one checkout, a long job you want off your main session, the same mechanical edit repeated across many files, or a fan-out too large to hold in one context
---

# Orchestrating Agents

## Overview

**Who does the work and where they do it are two separate decisions.** Older
guidance folds them together, finds that two agents would touch the same file,
and concludes "do not parallelize." That is the wrong conclusion from the right
observation. Collision is an isolation problem. Decide the mechanism first, then
decide the isolation.

**Announce at start:** "I'm using the orchestrating-agents skill to choose how to
run this."

Every rule below keys to a predicate you can check *before* dispatching. None
key to how hard the task feels.

## Routing

Take the first row that matches.

| Predicate | Mechanism |
|---|---|
| One agent can hold the whole task | Do it yourself. No dispatch. |
| Tasks only read | Parallel subagents, one checkout, no isolation |
| Tasks write, and the plan states file ownership with no intersection | Parallel subagents, one checkout |
| Tasks write, and ownership overlaps or is unstated | One worktree per agent — but read Isolation first |
| An agent needs reasoning this conversation already did | Forked subagent |
| The work is independent of your next step | Background session |
| Workers must agree on a contract while they work | Agent Teams — see the `maestro` skill |
| The same mechanical change across many separately reviewable units | `/batch` |
| Fan-out larger than you can hold, or a check-fix-recheck loop | Workflow — **ask first** |

"States file ownership" means the plan says so. Your own impression that two
tasks look unrelated is not ownership, and unstated ownership takes the overlap
row.

## Isolation

```dot
digraph isolation {
    "Do the agents write files?" [shape=diamond];
    "Do their file sets intersect?" [shape=diamond];
    "Is the shared file a manifest, lockfile, or generated file?" [shape=diamond];
    "One checkout, no isolation" [shape=box];
    "One worktree per agent" [shape=box];
    "One agent owns that file" [shape=box];

    "Do the agents write files?" -> "One checkout, no isolation" [label="no"];
    "Do the agents write files?" -> "Do their file sets intersect?" [label="yes"];
    "Do their file sets intersect?" -> "One checkout, no isolation" [label="no"];
    "Do their file sets intersect?" -> "Is the shared file a manifest, lockfile, or generated file?" [label="yes"];
    "Is the shared file a manifest, lockfile, or generated file?" -> "One worktree per agent" [label="no"];
    "Is the shared file a manifest, lockfile, or generated file?" -> "One agent owns that file" [label="yes"];
}
```

**A worktree does not dissolve a manifest conflict. It postpones it to merge
time and adds a lockfile to reconcile.** When two agents would both add
dependencies to one `package.json`, both edit one migration directory, or both
regenerate one lockfile or schema, isolation makes the problem worse, not
better. Give that file a single owner: either make all its edits up front and
let the others treat it as read-only, or serialize just the tasks that touch it.

This is the one branch where the wrong answer corrupts a working tree rather
than wasting tokens, which is why it is a diagram and not a sentence.

## Fresh context or inherited

A fresh subagent gets only what you construct for it. A forked subagent inherits
this conversation. Neither is the default.

**Fork** when reconstructing the situation would cost more than the work itself
— you have spent an hour narrowing a bug and want an alternative approach
explored from where you already are.

**Fresh** when the inherited context is the thing under suspicion. Asking a fork
to challenge a conclusion hands it the conclusion first; that is the bias you
were trying to escape. Adversarial checks on this session's own reasoning get a
fresh agent and a brief written from evidence, not from your conclusion.

## Cost ladder

The table is ordered. Each mechanism costs more than the one above it in tokens,
in wall-clock, and in the user's attention. Take the lightest mechanism whose
predicate holds.

The failure this guards against is reaching down the ladder because a heavier
mechanism feels more rigorous. A team of five for a three-file change loses to
doing it in the session, and it loses on quality too — coordination overhead
grows faster than throughput.

## Reaching each mechanism

| Mechanism | How |
|---|---|
| Parallel subagents | Several Agent calls in one message. One per message runs them serially. |
| Fork | `Agent(subagent_type: "fork")` |
| Worktree-isolated agent | `Agent(isolation: "worktree")` |
| Worktree session | `claude -w <name>`, add `--tmux` for panes |
| Background session | `claude --bg "<task>"`, or `/background` in session |
| Background dashboard | `claude agents`; then `attach`, `logs`, `stop`, `rm`, `respawn` |
| Cross-session messaging | `ListAgents` to find them, `SendMessage` to reach one |
| Agent Teams | `claude --agent-teams`, or `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |
| Batch | `/batch <instruction>` |
| Workflow | The Workflow tool |

## Gates

Three mechanisms spend the user's budget at a scale they have to agree to
first. Never start one on your own judgment:

- **Workflows** can run dozens of agents. They need explicit opt-in — the user
  saying "use a workflow", "ultracode", or naming a saved workflow.
- **`/batch`** opens pull requests. It is the user's repository and the user's
  reviewers.
- **`claude ultrareview`** is user-triggered and separately billed. You cannot
  launch it. Recommend it; let them run it.

Gated does not mean unavailable. When a gated mechanism is the right one, **name
it and offer it.** Do not quietly fall back to a lighter mechanism and leave the
user unaware the better route existed — "I'd run this as plain parallel agents
instead" is a downgrade made on their behalf without telling them. Say which
mechanism fits, say why it is gated, and let them decide.

Proposing one of these and waiting is always correct. Starting one is not.

## Waiting

A finished agent notifies you. Do not poll for it, do not schedule wake-ups to
check on it, and do not sit in a silent open-ended wait.

Keep doing local work while children run. **Never state or infer a pending
agent's result before its notification arrives** — a predicted result that turns
out wrong is worse than no result, because it looks like evidence.

## Rationalizations

| Excuse | Reality |
|---|---|
| "More agents means more thorough" | Coordination cost grows faster than throughput. Take the lightest mechanism that matches. |
| "They look independent, so parallel is fine" | Looking independent is not stated ownership. Unstated ownership takes the overlap row. |
| "Worktrees make any collision safe" | Not for manifests, lockfiles, or generated files. Those need one owner. |
| "This is big, so it needs a team" | Big and coupled needs a team. Big and separable needs `/batch`. Big and repetitive needs a workflow. |
| "I'll fork it so it has full context" | Full context includes your conclusion. For an adversarial check that is the bug. |
| "A workflow would obviously help here" | Then say so and ask. Workflows are opt-in. |
| "I'll start the batch and show them the PRs" | The PRs are the thing they needed to agree to. |

## Handing off

Once the mechanism is chosen, the tier skills own the mechanics:

- Subagents — **superpowers:dispatching-parallel-agents**
- Executing a plan with subagents — **superpowers:subagent-driven-development**
- Executing a plan without them — **superpowers:executing-plans**
- Isolated workspaces — **superpowers:using-git-worktrees**
- Agent Teams — **the `maestro` skill**
