---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# Executing Plans

## Overview

Load plan, review critically, execute all tasks, report when complete.

**Announce at start:** "I'm using the executing-plans skill to implement this plan."

**This skill is the fallback route, not the default one.** It executes a plan
inline, in this session, by hand. Before using it, check whether one of these
fits better:

| Situation | Route |
|---|---|
| Subagents are available | **superpowers:subagent-driven-development** — fresh implementer per task, review between tasks |
| The plan is self-contained and should not hold this session | A background session: `claude --bg`, watched with `claude agents` |
| The plan has several streams that must agree on contracts as they go | Agent Teams — see the `maestro` skill |
| None of the above, or no subagents at all | This skill |

Subagents are available on Claude Code, Codex CLI, Codex App, Copilot CLI and
Gemini CLI; see the per-platform tool refs in
`../using-superpowers/references/`. Use **superpowers:orchestrating-agents** to
choose between the routes; come back here when the answer is "execute it
inline."

## The Process

### Step 1: Load and Review Plan
1. Ensure an isolated workspace: use superpowers:using-git-worktrees to create one or verify the existing one
2. Read plan file
3. Review critically - identify any questions or concerns about the plan
4. If concerns: Raise them with your human partner before starting
5. If no concerns: Create todos for the plan items and proceed

### Step 2: Execute Tasks

For each task:
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Mark as completed

### Step 3: Complete Development

After all tasks complete and verified:
- Announce: "I'm using the finishing-a-development-branch skill to complete this work."
- **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch
- Follow that skill to verify tests, present options, execute choice

## When to Stop and Ask for Help

**STOP executing immediately when:**
- Hit a blocker (missing dependency, test fails, instruction unclear)
- Plan has critical gaps preventing starting
- You don't understand an instruction
- Verification fails repeatedly

**Ask for clarification rather than guessing.**

## When to Revisit Earlier Steps

**Return to Review (Step 1) when:**
- Partner updates the plan based on your feedback
- Fundamental approach needs rethinking

**Don't force through blockers** - stop and ask.

## Remember
- Review plan critically first
- Follow plan steps exactly
- Don't skip verifications
- Reference skills when plan says to
- Stop when blocked, don't guess
- Never start implementation on main/master branch without explicit user consent
