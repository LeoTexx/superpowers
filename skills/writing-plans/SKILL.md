---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

A plan's second job is saying what NOT to build. An implementer following a plan
faithfully cannot produce better code than the plan describes, so the plan is
where quality is decided.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** If working in an isolated worktree, it should have been created via the `superpowers:using-git-worktrees` skill at execution time.

**Save plans to:** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- (User preferences for plan location override this default)

## The Standard

**Elegant** is the smallest change that fully solves the problem, built on what
is already there. Measure it by what you did not have to write. A task that
wires three existing pieces together and adds twenty lines beats one that adds
four hundred and owns them forever.

**Robust** is failure paths named in the task: which inputs are invalid, what
happens when the request fails, what the empty state renders. "Add appropriate
error handling" is not a plan, it is a wish.

**Maintainable** is the next person finding it. It follows the conventions
already in the repo, sits where its siblings sit, and is named the way the
domain names it.

Three anti-goals, because each is a plausible-sounding road to the opposite:

- **No line-count golf.** Few lines is a consequence of reuse, never a target.
  Dense, clever code is not elegant; it is expensive.
- **Reuse beats abstraction.** Do not invent a framework to avoid repeating
  yourself twice. Misapplied DRY builds exactly the unmaintainable code this is
  meant to prevent — two similar blocks are cheaper than one wrong abstraction.
- **No speculative generality.** Build for the requirement in the spec, not the
  one you imagine next quarter.

## Scope Check

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during brainstorming. If it wasn't, suggest breaking this into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

## Survey Before You Decompose

Find out what already exists before mapping files. This is the step that decides
whether the plan adds a second date formatter or uses the one that is there.

Search by **operation, not by name.** A planner grepping `formatDate` misses
`toDisplayDate`. Search for what the code does: the data it transforms, the
endpoint it calls, the element it renders.

Cover four things:

1. **Helpers and modules** — does something already do this transformation,
   this request, this validation?
2. **Components and primitives** — for UI work, does a component for this
   already exist? If the project keeps a design system — a component contract
   in `docs/`, a `.architect/design-system/`, a tokens file — it is a
   foundation, and its patterns bind the plan the way an API contract does.
3. **House conventions** — what do CLAUDE.md, AGENTS.md, or the sibling module
   already mandate? "Migrations are generated, never hand-written" is a
   constraint, not a preference.
4. **Prior art** — the nearest existing feature of the same shape. Its files
   are the template for yours.

**List the shared directories, do not sample them.** Read what is actually in
`components/`, `lib/`, `packages/*` — a grep only finds what you thought to
name, and the module you fail to imagine is exactly the one you will duplicate.

Surveying is read-only, so it parallelizes freely: Explore agents in one
checkout, no isolation. See superpowers:orchestrating-agents.

What the survey finds goes in **Foundations**. What it fails to find is what you
are allowed to create.

## File Structure

Before defining tasks, map out which files will be created or modified and what each one is responsible for. This is where decomposition decisions get locked in.

- Design units with clear boundaries and well-defined interfaces. Each file should have one clear responsibility.
- You reason best about code you can hold in context at once, and your edits are more reliable when files are focused. Prefer smaller, focused files over large ones that do too much.
- Files that change together should live together. Split by responsibility, not by technical layer.
- In existing codebases, follow established patterns. If the codebase uses large files, don't unilaterally restructure - but if a file you're modifying has grown unwieldy, including a split in the plan is reasonable.

This structure informs the task decomposition. Each task should produce self-contained changes that make sense independently.

## Task Right-Sizing

A task is the smallest unit that carries its own test cycle and is worth a
fresh reviewer's gate. When drawing task boundaries: fold setup,
configuration, scaffolding, and documentation steps into the task whose
deliverable needs them; split only where a reviewer could meaningfully
reject one task while approving its neighbor. Each task ends with an
independently testable deliverable.

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

**Spec:** [path to the spec/design doc this plan implements — the plan
argues from the spec, so the spec travels with it; executors read both]

## Foundations

[What already exists that this plan builds on, from the survey. One line each,
naming the exact module, component or convention and how this feature uses it.
This is the reuse budget: a task may only create what is not listed here.]

- `lib/format.ts` `toDisplayDate` — every user-facing date
- `components/ui/DataTable` — all list surfaces; do not build a second table
- CLAUDE.md — never call `fetch` directly, use `lib/http.ts`

[If the survey genuinely found nothing, say so: "Greenfield; no existing modules
apply." An empty Foundations section without that sentence means the survey was
skipped.]

## Global Constraints

[The spec's project-wide requirements — version floors, dependency limits,
naming and copy rules, platform requirements — one line each, with exact
values copied verbatim from the spec. Every task's requirements implicitly
include this section.]

---
```

## Task Structure

````markdown
### Task N: [Component Name]

**Reuses:** `components/ui/DataTable`, `lib/format.ts` `toDisplayDate`
**New:** `InvitationRow` — nothing existing renders email + status + row action

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Interfaces:**
- Consumes: [what this task uses from earlier tasks — exact signatures]
- Produces: [what later tasks rely on — exact function names, parameter
  and return types. A task's implementer sees only their own task; this
  block is how they learn the names and types neighboring tasks use.]

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

**Reuses** lists the exact modules, components and helpers from Foundations
that this task builds on. **New** lists only what does not exist, each with the
reason nothing in Foundations fits — "nothing existing does X" is the required
shape.

**A wrapper is not New.** Creating `InvitationsTable` that renders `DataTable`,
or `InvitationStatusBadge` that renders `StatusBadge`, is duplication with a
naming ceremony: the second component exists only to hide the first, and now
two files must be changed to change one behaviour. Configure the primitive at
its call site instead. `New` is for behaviour no existing component has — never
for a feature-specific name wrapped around behaviour that already exists.

A task with an empty Reuses field is unfinished, the same way a task with no
test step is unfinished. A task whose New list is longer than its Reuses list
claims this feature is unlike everything already in the repo. Sometimes that is
true. Usually it is a survey that stopped early.

## No Placeholders

Every step must contain the actual content an engineer needs. These are **plan failures** — never write them:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" (without actual test code)
- "Similar to Task N" (repeat the code — the engineer may be reading tasks out of order)
- Steps that describe what to do without showing how (code blocks required for code steps)
- References to types, functions, or methods not defined in any task
- `Reuses: N/A`, or an empty Reuses field with no reason
- "Create a new X" where X already exists in the repo — that is a survey
  failure wearing a task's clothes
- Foundations entries too vague to check ("use the existing utils", "follow the
  design system")
- A `New` entry whose name is a Foundations entry with the feature bolted on
  (`InvitationsTable` beside `DataTable`, `InviteButton` beside `Button`) — a
  wrapper is not new work

## Self-Review

After writing the complete plan, look at the spec with fresh eyes and check the plan against it. This is a checklist you run yourself — not a subagent dispatch.

**1. Spec coverage:** Skim each section/requirement in the spec. Can you point to a task that implements it? List any gaps.

**2. Placeholder scan:** Search your plan for red flags — any of the patterns from the "No Placeholders" section above. Fix them.

**3. Type consistency:** Do the types, method signatures, and property names you used in later tasks match what you defined in earlier tasks? A function called `clearLayers()` in Task 3 but `clearFullLayers()` in Task 7 is a bug.

**4. Reuse check:** For every file the plan creates, ask what already does this.
Re-run the search for the three you are most confident are new — that confidence
is exactly where the survey stopped early. If something exists, replace the task
with one that uses it.

**5. Subtraction check:** Does any task delete, replace or consolidate
something? A feature that only ever adds is usually duplicating. If the plan
adds a fifth variant of something that already has four, the honest task is
consolidating the five.

If you find issues, fix them inline. No need to re-review — just fix and move on. If you find a spec requirement with no task, add the task.

## Execution Handoff

After saving the plan, offer execution choice:

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Two execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?"**

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Fresh subagent per task + two-stage review

**If Inline Execution chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:executing-plans
- Batch execution with checkpoints for review
