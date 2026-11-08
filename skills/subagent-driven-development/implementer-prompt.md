# Implementer Subagent Prompt Template

Use this template when dispatching an implementer subagent. The `coder` agent
(`~/.claude/agents/coder.md`) carries the standing standard: reading
CLAUDE.md and conventions, following an exemplar, test-first, smallest diff,
self-review, escalation, the status contract, the report format, and the
no-subagents rule. This template holds only what is specific to the task.

```
Subagent (coder):
  description: "Implement Task N: [task name]"
  model: [MODEL — REQUIRED: choose per SKILL.md Model Selection; an omitted
         model silently inherits the session's most expensive one]
  prompt: |
    Task N: [task name]

    Read your task brief first: [BRIEF_FILE]
    It contains the full task text from the plan.

    ## Context

    [Scene-setting: where this fits, dependencies, architectural context]

    Work from: [directory]

    **Done means:** every requirement in the brief is implemented, the
    brief's tests (and any you added) pass, the full suite passes once
    before your final commit, [your work is committed / do not commit],
    and your report file is written. Short of all of these, your status
    is not DONE.

    Report file: [REPORT_FILE]

    The coder agent carries the standing coding standard, escalation rules,
    status contract and report format.

    ## After Review Findings

    If the task review finds issues, you will be resumed with the findings.
    Fix them, re-run the tests covering the amended code, and append a fix
    report to your report file: what you changed, the covering tests, the
    command, and the output. Reviewers will not re-run tests for you.
```

## Placeholders

- `[MODEL]` — required; see SKILL.md Model Selection (sonnet by default,
  the session's model for design-heavy tasks).
- `[BRIEF_FILE]` — the file `scripts/task-brief` printed.
- `[REPORT_FILE]` — named after the brief (`task-N-brief.md` →
  `task-N-report.md`).
- `[directory]` — the worktree the task runs in.
