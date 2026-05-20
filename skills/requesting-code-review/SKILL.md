---
name: requesting-code-review
description: Use when completing tasks, implementing major features, or before merging to verify work meets requirements
---

# Requesting Code Review

Dispatch a code reviewer subagent to catch issues before they cascade. The reviewer gets precisely crafted context for evaluation — never your session's history. This keeps the reviewer focused on the work product, not your thought process, and preserves your own context for continued work.

**Core principle:** Review early, review often.

## When to Request Review

**Mandatory:**
- After each task in subagent-driven development
- After completing major feature
- Before merge to main

**Optional but valuable:**
- When stuck (fresh perspective)
- Before refactoring (baseline check)
- After fixing complex bug

## Specialist Reviewers

Superpowers ships six specialist review subagents under `agents/`. Each focuses on one aspect of code quality. Dispatch them by `subagent_type`.

| Aspect | Subagent | Use when |
|---|---|---|
| General code quality, CLAUDE.md compliance, bugs | `superpowers:code-reviewer` | Every review. Always applicable. |
| Error handling, silent failures, fallback abuse | `superpowers:silent-failure-hunter` | Diff touches try/catch, fallbacks, or error paths |
| Test coverage, edge cases, behavioral vs implementation tests | `superpowers:pr-test-analyzer` | Diff adds or modifies tests or testable logic |
| Type design, invariants, encapsulation | `superpowers:type-design-analyzer` | Diff adds or refactors types / data models |
| Comment accuracy and comment rot | `superpowers:comment-analyzer` | Diff adds non-trivial docs or comments |
| Simplification and clarity polish | `superpowers:code-simplifier` | After review passes, to polish before merge |

The first five are advisory — they report findings, they don't modify code. `code-simplifier` is also advisory; apply its suggestions yourself.

## How to Request — Single Reviewer (default)

For routine reviews, dispatch `superpowers:code-reviewer` against a git range.

**1. Get git SHAs:**
```bash
BASE_SHA=$(git rev-parse HEAD~1)   # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. Dispatch the reviewer.** Use the Agent (Task) tool with `subagent_type: "superpowers:code-reviewer"`. Fill the template at `code-reviewer.md`.

**Placeholders:**
- `{DESCRIPTION}` — brief summary of what you built
- `{PLAN_OR_REQUIREMENTS}` — what it should do
- `{BASE_SHA}` — starting commit
- `{HEAD_SHA}` — ending commit

**3. Act on feedback:**
- Fix Critical issues immediately
- Fix Important issues before proceeding
- Note Minor issues for later
- Push back if reviewer is wrong (with reasoning)

## How to Request — Multi-Aspect Fan-Out (high-stakes review)

For PRs touching critical code, dispatch multiple specialists **in parallel** in a single message. Each returns an independent report; you aggregate.

**When to fan out:**
- Before opening a PR for a large feature
- Before merging anything touching auth, payments, data integrity, or migrations
- When the diff spans multiple concerns (new types + new tests + new error handling)
- When your human partner explicitly asks for thorough review

**Pick the relevant specialists** based on what the diff actually contains. Don't run all six on every PR — most reviews only need 2-3.

**Example fan-out** for a PR that adds new types, error handling, and tests:

```
[In a single message, dispatch in parallel:]

Agent(subagent_type: "superpowers:code-reviewer",
      prompt: <fill template at code-reviewer.md>)

Agent(subagent_type: "superpowers:silent-failure-hunter",
      prompt: "Audit error handling in {BASE_SHA}..{HEAD_SHA}. Focus on
               <files-or-modules>. Report silent failures, broad catches,
               unjustified fallbacks.")

Agent(subagent_type: "superpowers:pr-test-analyzer",
      prompt: "Analyze test coverage in {BASE_SHA}..{HEAD_SHA}. Report
               critical gaps (rated 8-10) for <feature being added>.")

Agent(subagent_type: "superpowers:type-design-analyzer",
      prompt: "Review new types added in {BASE_SHA}..{HEAD_SHA}: <list
               of types>. Rate each on encapsulation, invariant expression,
               usefulness, enforcement.")
```

**Aggregating results:** After all specialists return, write a single consolidated report:

```
### Critical (must fix before merge)
- [code-reviewer] file:line — description
- [silent-failure-hunter] file:line — description

### Important (should fix)
- [pr-test-analyzer] file:line — description

### Minor / Nice-to-have
- ...

### Cross-specialist agreement
[Note when multiple specialists flagged the same area — high-confidence signal]
```

Fix Critical items, then re-dispatch the relevant specialist(s) to confirm.

## Post-Review Polish

After review passes and Critical/Important issues are fixed, optionally dispatch `superpowers:code-simplifier` to polish for clarity. This is **after** review — not a substitute for it.

```
Agent(subagent_type: "superpowers:code-simplifier",
      prompt: "Simplify recently-modified code in {BASE_SHA}..{HEAD_SHA}.
               Preserve all functionality. Focus on reducing nesting,
               eliminating redundancy, and improving naming.")
```

The simplifier returns suggestions; apply them by hand and re-run tests.

## Example — Single Reviewer

```
[Just completed Task 2: Add verification function]

You: Let me request code review before proceeding.

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[Dispatch superpowers:code-reviewer]
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types
  PLAN_OR_REQUIREMENTS: Task 2 from docs/superpowers/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661

[Subagent returns]:
  Critical: (none)
  Important: Missing progress indicators (indexer.ts:130)
  Minor: Magic number (100) for reporting interval
  Assessment: Ready to proceed with fixes

You: [Fix progress indicators]
[Continue to Task 3]
```

## Integration with Workflows

**Subagent-Driven Development:**
- Review after EACH task with `superpowers:code-reviewer`
- Fan out specialists at major checkpoints
- Catch issues before they compound

**Executing Plans:**
- Review after each task or at natural checkpoints
- Get feedback, apply, continue

**Ad-Hoc Development:**
- Review before merge
- Fan out specialists when stakes are high

## Red Flags

**Never:**
- Skip review because "it's simple"
- Ignore Critical issues
- Proceed with unfixed Important issues
- Argue with valid technical feedback
- Fan out all six specialists by default — pick what the diff actually needs

**If reviewer wrong:**
- Push back with technical reasoning
- Show code/tests that prove it works
- Request clarification

See:
- Template for single-reviewer dispatch: `requesting-code-review/code-reviewer.md`
- Specialist agent definitions: `agents/*.md` in the superpowers plugin
