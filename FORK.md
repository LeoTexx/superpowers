# This is a fork

`LeoTexx/superpowers`, forked from [`obra/superpowers`](https://github.com/obra/superpowers)
at `b36e082` (v6.3.0). Upstream's README follows in `README.md`; this file
records what diverged and how to keep it in sync.

## Why fork

The upstream plugin installs into `~/.claude/plugins/cache/`, which a plugin
update overwrites. Any local edit is silently lost. Forking is the only way to
carry changes.

## Installed as

Marketplace `superpowers-leo` → `enabledPlugins["superpowers@superpowers-leo"]`
in `~/.claude/settings.json`, with `superpowers@claude-plugins-official` set to
`false`. Both must not run: they ship the same SessionStart hook and the same
skill names.

**A marketplace installs from the default branch.** A change is not live until
it reaches `main` and Claude Code restarts.

## What diverged

Superpowers knew one way to use more than one agent. Claude Code now has seven,
and three skills stated rules the newer harness makes false.

| File | Change |
|---|---|
| `skills/orchestrating-agents/` | **New.** The routing decision — which mechanism, what isolation — keyed to predicates checkable before dispatch. |
| `skills/dispatching-parallel-agents/` | Rewritten. Context is a choice (fresh vs forked). Overlapping files mean *isolate*, not serialize. |
| `skills/subagent-driven-development/` | Parallel implementers allowed under proven-disjoint ownership or worktrees. Polling ban and timed waits removed — a finished subagent notifies you. `ultrareview` named for whole-branch review. |
| `skills/executing-plans/` | Background-session and Agent Teams routes named. |
| `skills/writing-plans/` | Survey before decomposing; a `Foundations` reuse budget; required `Reuses:`/`New:` per task; a subtraction check. |
| 14 skill files (2026-09-22) | Edits made directly in the installed plugin and reconciled here on 2026-10-04: the brief every agent gets (goal, done, stop-and-ask, evidence, report shape), no thinking-instructions, plan files as the durable checklist, and related wording across the review, debugging, verification and skill-writing skills. See the commit for the full diff. |
| `skills/brainstorming/`, `skills/writing-plans/`, `skills/orchestrating-agents/` | One sentence each: a user-facing flow gets a flow contract (`flow-audit contract`) in the design, its acceptance criteria become plan constraints and agent Done conditions, and `flow-audit check` runs after the build. `flow-audit` lives in `~/.claude/skills/`, outside this plugin; the sentences are conditional on it being present. |
| `.claude-plugin/` | Marketplace renamed `superpowers-leo`, version `6.4.0-leo.2`. |

**Do not edit the installed copy.** `~/.claude/plugins/cache/` and `~/.claude/plugins/marketplaces/superpowers-leo` are overwritten by a plugin update. The September edits were made there and sat uncommitted for two weeks; one update would have erased them. Edit this repo, push `main`, then update the plugin.

Three corrections worth knowing, because upstream still has them:

- **"Never dispatch multiple implementation subagents in parallel (conflicts)"**
  — the parenthesis named the real constraint, and worktree isolation removes
  it. Now conditional on stated file ownership.
- **"Wait in bounded stretches of five to ten minutes"** — the harness
  re-invokes the controller on completion, so both the polling ban and the
  timed waiting are obsolete.
- **A worktree does not dissolve a shared-manifest conflict.** Two agents both
  adding dependencies to one `package.json` defer the conflict to merge time
  and add a lockfile to reconcile. That file needs one owner. This correction
  came from a clean baseline run that *rejected* worktrees with a better
  argument than the spec had.

## Evidence

Every change was tested RED/GREEN with `claude --safe-mode` as the control arm
and `--append-system-prompt` as the treatment arm, five reps, one scenario per
agent.

- `docs/superpowers/specs/2026-09-08-orchestration-stack-design.md` — decisions
- `docs/superpowers/plans/2026-09-08-orchestration-stack.md` — the plan
- `docs/superpowers/plans/2026-09-08-orchestration-baselines.md` — baselines and
  results, including the method's own flaw: Agent-tool subagents inherit the
  operator's memory and cannot produce a clean control.

Headline numbers: `writing-plans` went from **0/5** plans naming any existing
module to **5/5** with no duplication. The loosened parallel-implementer rule
held at **5/5** against an adversarial collision case.

## Syncing upstream

```bash
git fetch upstream
git merge upstream/main
```

Conflicts land in the four rewritten skills; `orchestrating-agents` is new and
will not conflict. Take upstream's structure and re-apply the corrections above
— the spec explains each one's reasoning, so a mechanical re-application is
possible without re-deriving it.

Then:

```bash
git push origin main   # the marketplace installs from the default branch
```

and restart Claude Code.

## Contributing back

Nothing has been sent upstream. The three corrections listed above are general
rather than personal, and would be reasonable pull requests. The marketplace
rename and version bump are local and must not be included in one.
