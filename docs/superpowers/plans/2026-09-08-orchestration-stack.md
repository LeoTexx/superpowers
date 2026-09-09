# Orchestration Stack Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Teach the superpowers skills — and the personal `maestro` skill — to route work across Claude Code's seven orchestration mechanisms instead of assuming subagents are the only one.

**Architecture:** One new routing skill (`orchestrating-agents`) owns the "which mechanism, what isolation" decision. Four existing skills are corrected to stop stating rules that are now false, and each defers routing to the new skill. `maestro` loses its hand-rolled machinery and becomes a driver for native Agent Teams.

**Tech Stack:** Markdown skills with YAML frontmatter; Claude Code 2.1.263; the Agent tool for test dispatch; `gh`/`git` for the fork.

**Spec:** `docs/superpowers/specs/2026-09-08-orchestration-stack-design.md`

## Global Constraints

- **Do not commit.** Every change stays in the working tree on branch `orchestration-stack` so it can be reviewed with `git diff` first. No `git commit`, no `git push`, no PR.
- Fork repo root: `~/Dev/Zeratus/code/superpowers`, branch `orchestration-stack`, base `b36e082`.
- Personal-skill root: `~/.claude/skills/`, mirrored to `~/.agents/skills/`. The two must end identical.
- Every skill description is **triggering conditions only**. Never summarize a skill's process or workflow in its `description` — a description that summarizes a workflow becomes the shortcut agents take instead of reading the skill.
- Every routing rule keys to a predicate an agent can check before dispatching, never to a judgment about task difficulty.
- Claimed capabilities must match the verified baseline table in the spec. Do not write a capability into a skill that was not verified on 2.1.263.
- `claude ultrareview` is user-triggered and separately billed. Skills may recommend it to the human; no skill may instruct an agent to launch it.
- Test dispatch uses one subagent per scenario. Never batch scenarios into one agent — batched runs produce agreeable, false results.
- Record baseline rationalizations verbatim. Rationalization tables are built from what runs actually produce, not from what seems likely.

---

### Task 1: Baseline the routing failure (RED)

Observe what an agent does with no routing guidance. This is the failing test; nothing may be written until it has run.

**Files:**
- Create: `docs/superpowers/plans/2026-09-08-orchestration-baselines.md`

**Interfaces:**
- Consumes: nothing.
- Produces: the baselines file, whose "Scenario N — baseline" sections Tasks 2, 4 and 6 read to target their guidance.

- [ ] **Step 1: Write the five scenario prompts**

Each is dispatched to a fresh general-purpose subagent, alone, with this exact wrapper and no superpowers context:

```
You are working in a git repository. Describe how you would carry out the
task below: what you would dispatch, how many agents, and where each would
run. Do not do the work. Answer in under 150 words.

TASK: <scenario>
```

Scenarios, verbatim:

1. `Six tests are failing across three files. The three files' failures have unrelated root causes. Fix them.`
2. `Migrate all sixty controllers under src/controllers from Joi validation to our existing Zod helpers. Preserve API behaviour.`
3. `Implement the invitation API and the invitation UI. Both will add dependencies to package.json.`
4. `We think this 401 is a JWT expiry race. I am about to implement that fix. Separately, stress-test whether the diagnosis is wrong.`
5. `Audit every one of the 120 route handlers under src/routes for missing authorization checks, then verify each finding.`

- [ ] **Step 2: Run the five scenarios**

Five Agent dispatches, one scenario each, in a single message so they run concurrently. They only read and describe, so no isolation is needed.

- [ ] **Step 3: Record the baselines**

For each scenario write: the mechanism chosen, whether isolation was mentioned at all, and any rationalization quoted verbatim. Expected failure — the reason this is the RED step — is that scenarios 2, 4 and 5 all collapse to "parallel subagents", and that scenario 3 either serializes or ignores the collision.

- [ ] **Step 4: Checkpoint**

`git status` shows exactly one new file under `docs/superpowers/plans/`. Do not commit.

---

### Task 2: Write orchestrating-agents (GREEN)

**Files:**
- Create: `skills/orchestrating-agents/SKILL.md`
- Read: `docs/superpowers/plans/2026-09-08-orchestration-baselines.md`

**Interfaces:**
- Consumes: Task 1's baselines.
- Produces: skill name `orchestrating-agents`, referenced by Tasks 3, 5 and 6 as `superpowers:orchestrating-agents`.

- [ ] **Step 1: Write the frontmatter**

```yaml
---
name: orchestrating-agents
description: Use when work might need more than one agent — several independent investigations, changes that would collide in one checkout, a long job you want off your main session, the same mechanical edit across many files, or a fan-out too large to hold in one context
---
```

- [ ] **Step 2: Write the body**

Required sections, in order:

1. **Overview** — the core principle, stated first: *who does the work and where they do it are two separate decisions.* The current guidance folds them together and concludes "do not parallelize"; the correct conclusion is "parallelize, in worktrees".
2. **Routing table** — the eight predicate/mechanism rows from the spec, verbatim, with a note that "provably disjoint" means the plan states file ownership, and that unknown ownership routes to worktrees.
3. **Isolation** — one flowchart, the only one in the skill, covering: do the tasks write? → are the file sets disjoint? → one checkout or one worktree each. Graphviz per `graphviz-conventions.dot`.
4. **Cost ladder** — each level costs more than the one above in tokens, wall-clock and the user's attention; take the lightest mechanism whose predicate holds.
5. **Reaching each mechanism** — one line each, only the verified forms: several Agent calls in one message; `Agent(subagent_type: "fork")`; `Agent(isolation: "worktree")`; `claude --bg` and `/background`; `claude agents`; `claude -w <name>`; `ListAgents`/`SendMessage`; `claude --agent-teams` or `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`; `/batch`; the Workflow tool.
6. **Opt-in gates** — Workflows and `/batch` spend the user's budget at a scale they must agree to. Never start either without being asked. `claude ultrareview` is user-triggered; recommend, never launch.
7. **Waiting** — a finished agent notifies you. Do local work meanwhile. Never fabricate a pending agent's result.
8. **Rationalization table** — built from the Task 1 baselines, aimed at over-orchestration.

Target under 900 words. Tables and lists, not prose.

- [ ] **Step 3: Re-run the five scenarios (GREEN)**

Same five dispatches, same wrapper, with the skill text prepended as context. Expected: scenario 2 reaches `/batch`, 3 reaches worktrees, 4 reaches a fork, 5 reaches a workflow with an opt-in caveat, 1 stays parallel subagents.

- [ ] **Step 4: Close the loopholes (REFACTOR)**

Any scenario that still picks wrong: add the specific counter its answer revealed, then re-run that scenario only. Repeat until all five route correctly. Append every iteration to the baselines file.

- [ ] **Step 5: Checkpoint**

`git status` shows the new skill and the updated baselines. Do not commit.

---

### Task 3: Rewrite dispatching-parallel-agents

**Files:**
- Modify: `skills/dispatching-parallel-agents/SKILL.md`

**Interfaces:**
- Consumes: `superpowers:orchestrating-agents` from Task 2.
- Produces: nothing later tasks depend on.

- [ ] **Step 1: Delete the two false claims**

Remove, from the Overview: `They should never inherit your session's context or history — you construct exactly what they need.`

Remove, from both "Don't use when" and "When NOT to Use": the `Shared state: Agents would interfere (editing same files, using same resources)` line and the `Agents would interfere with each other` line.

- [ ] **Step 2: Replace them**

Context becomes a choice: a fresh subagent gets constructed context and is right when the work is self-contained; a fork inherits this conversation and is right when reconstructing the situation would cost more than the work itself. Both preserve the main context.

Shared state becomes an isolation problem: overlapping file sets mean each agent needs `isolation: "worktree"`, not that the work must serialize. Only genuinely coupled reasoning — where fixing one may fix the others — still argues for a single agent.

- [ ] **Step 3: Add the fork section**

Its real use: challenge a conclusion this session already reached, or explore an alternative, from a context the agent already holds. The worked example is scenario 4 from Task 1.

- [ ] **Step 4: Keep what is still right**

The "several dispatches in one message run concurrently" rule, the Agent Prompt Structure section, and the Common Mistakes table all stay. Add one closing line handing off to `superpowers:orchestrating-agents` when the fan-out outgrows this tier.

- [ ] **Step 5: Verify**

Re-run Task 1 scenarios 3 and 4 against this skill alone, without `orchestrating-agents`. Both must reach the right mechanism from this skill's text by itself.

- [ ] **Step 6: Checkpoint** — do not commit.

---

### Task 4: Patch subagent-driven-development

The riskiest task: it loosens a prohibition. Test the loosening adversarially before writing it.

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md`

**Interfaces:**
- Consumes: nothing.
- Produces: nothing.

- [ ] **Step 1: Write the adversarial test (RED)**

Dispatch one subagent with the proposed loosened wording plus this plan fragment:

```
Task 3: add the invitation endpoint. Files: src/api/routes.ts, src/api/types.ts
Task 4: add the invitation UI.       Files: src/ui/invite.tsx, src/api/types.ts
```

Ask: may Tasks 3 and 4 run in parallel? The correct answer is no — both write `src/api/types.ts` — or yes only under one worktree each. An agent that says plain yes is the failure this wording must prevent.

- [ ] **Step 2: Replace the parallel ban**

Old text, in "1. Dispatch the implementer":

```
- Never dispatch multiple implementation subagents in parallel (conflicts).
```

New text — a prohibition with its exception keyed to an observable predicate, not soft guidance:

```
- Dispatch implementers in parallel ONLY when one of these is true, checked
  before dispatch: the plan states each task's file ownership and the sets do
  not intersect; or each implementer runs under `isolation: "worktree"`.
  Neither true — including "the plan does not say" — means serial.
  Reviewers never write, so reviews of separate tasks always parallelize.
```

Add to Common Rationalizations:

```
| "The tasks look unrelated, they can run in parallel" | Looking unrelated is not file ownership. Unstated ownership is serial. |
| "They only both touch one shared file" | One shared file is a conflict. Worktrees, or serial. |
```

- [ ] **Step 3: Replace the waiting paragraph**

Old text, in "The Task Loop" — the paragraph beginning `**Waiting on dispatched subagents:** never poll a wait interface` and ending `noticed within minutes, not at the end of the session.`

New text:

```
**Waiting on dispatched subagents:** a finished subagent notifies you — you
do not poll for it and you do not schedule wake-ups to check. Keep doing
local work while children run: ledger updates, packaging the next review,
reading reports. Never state or infer a pending agent's result before its
notification arrives.
```

- [ ] **Step 4: Add ultrareview to the final review**

In "Final Review", after the existing dispatch instruction, add that for a large or high-risk branch the stronger option is `claude ultrareview` — and that the controller cannot launch it, so it is surfaced to the human as a recommendation alongside the finished work.

- [ ] **Step 5: Re-run the adversarial test (GREEN)**

Same dispatch, final wording. It must refuse the parallel dispatch or require worktrees. If it says plain yes, tighten and re-run — this test failing is worse than any routing test failing, because it corrupts a working tree rather than wasting tokens.

- [ ] **Step 6: Checkpoint** — do not commit.

---

### Task 5: Patch executing-plans

**Files:**
- Modify: `skills/executing-plans/SKILL.md`

**Interfaces:**
- Consumes: `superpowers:orchestrating-agents` from Task 2.
- Produces: nothing.

- [ ] **Step 1: Replace the Note block**

Old text is the `**Note:** Tell your human partner that Superpowers works much better with access to subagents...` paragraph, which offers exactly one alternative.

New text names three routes and defers the choice: subagent-driven-development for a plan executed in this session; a background session (`claude --bg`) for a self-contained plan that should not hold this session; Agent Teams for a plan with several streams that must agree on contracts as they go. Point at `superpowers:orchestrating-agents` to choose, and keep this skill as the route for when none of them are available.

- [ ] **Step 2: Verify**

Confirm the skill still reads as a coherent single-session procedure — the new block adds routes, it does not turn this skill into a router.

- [ ] **Step 3: Checkpoint** — do not commit.

---

### Task 6: Rewrite maestro

**Files:**
- Delete: `~/.claude/skills/maestro/agents/` (inspect `agents/openai.yaml` first and preserve it elsewhere if it is a live cross-runtime definition)
- Rewrite: `~/.claude/skills/maestro/SKILL.md`
- Mirror: `~/.agents/skills/maestro/`

**Interfaces:**
- Consumes: `superpowers:orchestrating-agents` from Task 2, for the routing decision this skill deliberately does not make.
- Produces: nothing.

- [ ] **Step 1: Baseline (RED)**

Dispatch one subagent with the *current* maestro SKILL.md and this task: `Set up coordinated work for: database migration, invitation API, invitation UI, and E2E tests.` Record what it produces. Expected failure: it writes shell scripts and a `.agents/` tree, and tells the user to open four terminals.

- [ ] **Step 2: Fix the frontmatter**

```yaml
---
name: maestro
description: Use when several workstreams must proceed at once AND agree with each other as they go — a feature spanning API, UI and tests where the contract between them is not yet fixed. Not for independent tasks, and not for work that fits in one session.
---
```

`tools:` and `color:` are agent fields and do nothing in a skill — remove both. Remove `disable-model-invocation: true`; the gate moves into the body as an explicit rule.

- [ ] **Step 3: Write the body**

Required sections:

1. **Gate**, first line of the body: never spawn a team without the user's explicit go-ahead. A team is several full sessions and costs accordingly.
2. **When this is the wrong skill** — independent tasks go to `superpowers:dispatching-parallel-agents`; mechanical fan-out goes to `/batch`; anything that fits one session stays there. Defer the full decision to `superpowers:orchestrating-agents`.
3. **Preflight** — `claude --agent-teams` for one session, or `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in `settings.json` to make it durable. Say plainly that it is experimental.
4. **Shape** — roles come from the dependency graph, not from job titles. Cap at four teammates and say why: each is a full session, and coordination cost grows faster than throughput.
5. **Ownership** — exactly one owner per file or package, decided before spawning. This is the collision rule, and it is what makes a team safe without worktrees. If ownership cannot be made exclusive, the work is not team-shaped: use worktrees or serialize.
6. **Contracts** — name what each teammate hands the next before spawning. A contract discovered mid-flight is a rewrite of whoever guessed wrong.
7. **Spawn** — the prompt template: per-teammate role, owned paths, the contract each publishes and each consumes, and the instruction to agree on contracts before the consumer implements.
8. **Drive** — `claude agents` to see state, `SendMessage` to redirect one teammate, the shared task list with dependencies for ordering.
9. **Fallback** — if teams is unavailable: `claude --bg -w <stream>` per stream, plus `ListAgents`/`SendMessage` across sessions. Same shape, coordinated by hand.
10. **Anti-patterns** — a team for tightly coupled single-file work; more than four teammates; two teammates owning one file; spawning before the contracts are named.

Delete every trace of the old machinery: `manager-tools.sh`, `developer-tools.sh`, `reviewer-tools.sh`, `work-orders/`, `submissions/`, `reviews/`, `feedback/`, `dev-workspace/`, `manager-state.json`, `.agents/`, the three terminal prompts, and the ASCII terminal diagram.

- [ ] **Step 4: Verify (GREEN)**

Re-run Step 1's scenario against the new skill. It must produce: four roles derived from dependencies, exclusive file ownership per role, the API-to-UI contract named before spawning, and no shell scripts. Confirm it does not spawn anything without asking.

- [ ] **Step 5: Mirror**

Copy `~/.claude/skills/maestro/` to `~/.agents/skills/maestro/`. Verify with `diff -r` that the two are identical.

- [ ] **Step 6: Checkpoint** — do not commit. The personal skills directory is not a git repo; note in the final report that these files are unversioned and were changed in place.

---

### Task 7: Switch the install to the fork

Last, so that a broken switch cannot hide a skill-authoring error.

**Files:**
- Modify: `.claude-plugin/marketplace.json`
- Modify: `.claude-plugin/plugin.json`
- Modify: `~/.claude/settings.json`

**Interfaces:**
- Consumes: everything above.
- Produces: nothing.

- [ ] **Step 1: Rename the marketplace**

In `.claude-plugin/marketplace.json`, `"name": "superpowers-dev"` → `"name": "superpowers-leo"`. Leave the plugin entry's own `name` as `superpowers`; the skill namespace `superpowers:` must not change or every cross-reference in every skill breaks.

- [ ] **Step 2: Bump the version**

In both `.claude-plugin/plugin.json` and the plugin entry in `.claude-plugin/marketplace.json`: `"version": "6.3.0"` → `"version": "6.4.0-leo.1"`.

- [ ] **Step 3: Back up settings**

```bash
cp ~/.claude/settings.json ~/.claude/settings.json.bak.$(date +%Y%m%d%H%M%S)
```

- [ ] **Step 4: Edit settings.json**

Add to `extraKnownMarketplaces`:

```json
"superpowers-leo": { "source": { "source": "github", "repo": "LeoTexx/superpowers" } }
```

In `enabledPlugins`, set `"superpowers@claude-plugins-official": false` and add `"superpowers@superpowers-leo": true`. Both copies must not run: they ship the same SessionStart hook and the same skill names.

- [ ] **Step 5: Validate**

```bash
python3 -m json.tool ~/.claude/settings.json > /dev/null && echo "settings OK"
python3 -m json.tool ~/Dev/Zeratus/code/superpowers/.claude-plugin/marketplace.json > /dev/null && echo "marketplace OK"
python3 -m json.tool ~/Dev/Zeratus/code/superpowers/.claude-plugin/plugin.json > /dev/null && echo "plugin OK"
```

- [ ] **Step 6: Report the manual step**

The marketplace installs from GitHub, and nothing has been pushed — that is deliberate. Tell the user plainly: the switch takes effect only after they review the diff, commit, push the branch, and either point the marketplace at that branch or merge it to `main`. Until then the official 6.3.0 plugin keeps running, unchanged.

- [ ] **Step 7: Checkpoint** — do not commit.

---

## Self-Review

**Spec coverage.** Routing skill → Task 2. `dispatching-parallel-agents` → Task 3. The three `subagent-driven-development` edits → Task 4 steps 2, 3, 4. `executing-plans` → Task 5. `maestro` → Task 6. Install → Task 7. Testing regime → Tasks 1, 2 step 3, 3 step 5, 4 steps 1 and 5, 6 steps 1 and 4. Out-of-scope items appear in no task, as intended.

**Placeholders.** None. Every replaced passage is quoted from the current file, and every new passage is either given verbatim or specified as a named required section.

**Consistency.** The skill is `orchestrating-agents` in every reference. The namespace stays `superpowers:` throughout, which Task 7 step 1 explicitly protects. The routing table has one home, Task 2, and the other skills defer to it rather than restating it.

**One deliberate deviation from the skill template.** Every task ends in a checkpoint rather than a commit, because the user asked to review the whole change in git before anything is committed.
