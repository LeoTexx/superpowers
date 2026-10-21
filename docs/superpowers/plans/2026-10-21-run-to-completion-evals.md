# Run-to-completion wording: evals (2026-10-21)

## Problem

Long unattended runs (Ralph + orchestrating-agents) ended with a PR covering the
first phase of a large brief, reported as complete. Evidence from Anthropic's
harness posts and Ralph's source: every exit path in plain Ralph is the agent
grading its own finish line, and Ralph's default `OPTIONAL_SECTIONS` lets
unchecked items under a "Future" heading stop blocking exit.

The mechanical fix (frozen feature list, gate, Stop/SubagentStop hooks, restart
wrapper, evaluator) is `ralph-done` in the `ralph` skill. The wording here only
routes agents to it and changes what "Done" means in a brief.

## Method

Same as the September baselines: `claude -p --safe-mode` with the three skills
appended to the system prompt. Control = `HEAD` versions, treatment = edited
versions. 5 reps per arm per scenario. Opus (CLI default), Claude Code 2.1.306.

- **A** — write the scope and definition of done for an overnight Ralph run of a
  six-part SaaS brief.
- **B** — write the Done section of the brief for an agent running the same
  brief under `claude -p` in a loop.
- **C** — "use brainstorming and writing-plans, give me the spec and task list
  you will hand to Ralph", same brief.

Scored by keyword, then every flagged match read by hand.

## Results

| Measure | Control | Treatment |
|---|---|---|
| All six parts in the run's scope (A+B+C) | 15/15 | 15/15 |
| Any part deferred to a later phase or follow-up | 0/15 | 0/15 (all "follow-up"/"phase 2" hits were prohibitions, all "out of scope" hits were extras the brief never asked for) |
| Done enforced outside the agent (frozen list / `ralph-done gate`) | 1/15 | 14/15 |

## Reading

The control did **not** reproduce scope loss at planning time: asked directly,
the model keeps all six parts. Scope loss happens mid-run, when the agent re-plans
and grades its own work, which a wording test cannot show. What the wording
changes is the definition of done: the control writes a checklist the agent
ticks itself (the condition under which runs stop early); the treatment hands the
decision to a frozen list and a gate. The end-to-end check of the mechanism is a
live `ralph-done run`, recorded in the `ralph` skill's history, not here.
