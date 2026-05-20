---
name: subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
---

# Subagent-Driven Development

Execute plan by dispatching fresh subagent per task. Review depth is tiered to task risk — routine tasks rely on implementer self-review, standard tasks get a single combined review, and high-risk tasks get the full two-stage review.

**Why subagents:** You delegate tasks to specialized agents with isolated context. By precisely crafting their instructions and context, you ensure they stay focused and succeed at their task. They should never inherit your session's context or history — you construct exactly what they need. This also preserves your own context for coordination work.

**Core principle:** Fresh subagent per task + review depth matched to task risk = high quality without ceremony tax.

## Review Tiers

Pick the lightest tier the task can defensibly take. Default to **Standard** when unsure.

**Routine** — implementer self-review only:
- 1-2 files, mechanical change (rename, dep bump, typo, format)
- Complete spec, no design judgment
- Cheap to verify by running tests

**Standard** (default) — one combined spec+quality review:
- Multi-file or non-trivial logic
- Implementer is following an unambiguous task in the plan
- Dispatch a single reviewer with both `./spec-reviewer-prompt.md` and `./code-quality-reviewer-prompt.md` concerns merged into one prompt

**High-risk** — full two-stage review (spec first, then code quality):
- Touches security, auth, data integrity, payment, or migrations
- Introduces new architecture or cross-cutting patterns
- Implementer reported `DONE_WITH_CONCERNS`
- Previous tasks in this plan have surfaced quality issues

When you escalate a task to a higher tier, note why in your TodoWrite item so the human can sanity-check your judgment later.

**Continuous execution:** Do not pause to check in with your human partner between tasks. Execute all tasks from the plan without stopping. The only reasons to stop are: BLOCKED status you cannot resolve, ambiguity that genuinely prevents progress, or all tasks complete. "Should I continue?" prompts and progress summaries waste their time — they asked you to execute the plan, so execute it.

## When to Use

Use **subagent-driven-development** when you have an implementation plan, the tasks are mostly independent, and you want to stay in this session. If the tasks are tightly coupled or you don't have a plan yet, fall back to manual execution or brainstorm first. If you want to hand off to a parallel session instead, use **executing-plans**.

**vs. Executing Plans (parallel session):**
- Same session (no context switch)
- Fresh subagent per task (no context pollution)
- Two-stage review after each task: spec compliance first, then code quality
- Faster iteration (no human-in-loop between tasks)

## The Process

1. Read the plan once, extract all tasks with their full text and surrounding context, and create a TodoWrite list. For each task, decide its review tier (Routine / Standard / High-risk — see above) and record it on the todo.
2. For each task in order:
   1. Dispatch the implementer subagent using `./implementer-prompt.md`.
   2. If the implementer asks questions, answer them and re-dispatch with the new context.
   3. Otherwise let the implementer implement, test, commit, and self-review.
   4. Run reviews per the task's tier:
      - **Routine:** no review subagent. Trust implementer self-review + passing tests. If the implementer reported `DONE_WITH_CONCERNS`, escalate this task to Standard.
      - **Standard:** dispatch one combined reviewer covering both spec compliance and code quality (merge `./spec-reviewer-prompt.md` and `./code-quality-reviewer-prompt.md` into a single prompt). If it finds issues, the implementer fixes them and the reviewer re-reviews until approved.
      - **High-risk:** dispatch the spec reviewer first using `./spec-reviewer-prompt.md`. After it approves, dispatch the code quality reviewer using `./code-quality-reviewer-prompt.md`. Re-review until both approve.
   5. Mark the task complete in TodoWrite.
3. After every task is complete, dispatch a final code reviewer subagent over the entire implementation.

## Model Selection

Use the least powerful model that can handle each role to conserve cost and increase speed.

**Mechanical implementation tasks** (isolated functions, clear specs, 1-2 files): use a fast, cheap model. Most implementation tasks are mechanical when the plan is well-specified.

**Integration and judgment tasks** (multi-file coordination, pattern matching, debugging): use a standard model.

**Architecture, design, and review tasks**: use the most capable available model.

**Task complexity signals:**
- Touches 1-2 files with a complete spec → cheap model
- Touches multiple files with integration concerns → standard model
- Requires design judgment or broad codebase understanding → most capable model

## Handling Implementer Status

Implementer subagents report one of four statuses. Handle each appropriately:

**DONE:** Proceed to spec compliance review.

**DONE_WITH_CONCERNS:** The implementer completed the work but flagged doubts. Read the concerns before proceeding. If the concerns are about correctness or scope, address them before review. If they're observations (e.g., "this file is getting large"), note them and proceed to review.

**NEEDS_CONTEXT:** The implementer needs information that wasn't provided. Provide the missing context and re-dispatch.

**BLOCKED:** The implementer cannot complete the task. Assess the blocker:
1. If it's a context problem, provide more context and re-dispatch with the same model
2. If the task requires more reasoning, re-dispatch with a more capable model
3. If the task is too large, break it into smaller pieces
4. If the plan itself is wrong, escalate to the human

**Never** ignore an escalation or force the same model to retry without changes. If the implementer said it's stuck, something needs to change.

## Prompt Templates

- `./implementer-prompt.md` - Dispatch implementer subagent
- `./spec-reviewer-prompt.md` - Dispatch spec compliance reviewer subagent
- `./code-quality-reviewer-prompt.md` - Dispatch code quality reviewer subagent

## Example Workflow

```
You: I'm using Subagent-Driven Development to execute this plan.

[Read plan file once: docs/superpowers/plans/feature-plan.md]
[Extract all 5 tasks with full text and context]
[Create TodoWrite with all tasks]

Task 1: Hook installation script

[Get Task 1 text and context (already extracted)]
[Dispatch implementation subagent with full task text + context]

Implementer: "Before I begin - should the hook be installed at user or system level?"

You: "User level (~/.config/superpowers/hooks/)"

Implementer: "Got it. Implementing now..."
[Later] Implementer:
  - Implemented install-hook command
  - Added tests, 5/5 passing
  - Self-review: Found I missed --force flag, added it
  - Committed

[Dispatch spec compliance reviewer]
Spec reviewer: ✅ Spec compliant - all requirements met, nothing extra

[Get git SHAs, dispatch code quality reviewer]
Code reviewer: Strengths: Good test coverage, clean. Issues: None. Approved.

[Mark Task 1 complete]

Task 2: Recovery modes

[Get Task 2 text and context (already extracted)]
[Dispatch implementation subagent with full task text + context]

Implementer: [No questions, proceeds]
Implementer:
  - Added verify/repair modes
  - 8/8 tests passing
  - Self-review: All good
  - Committed

[Dispatch spec compliance reviewer]
Spec reviewer: ❌ Issues:
  - Missing: Progress reporting (spec says "report every 100 items")
  - Extra: Added --json flag (not requested)

[Implementer fixes issues]
Implementer: Removed --json flag, added progress reporting

[Spec reviewer reviews again]
Spec reviewer: ✅ Spec compliant now

[Dispatch code quality reviewer]
Code reviewer: Strengths: Solid. Issues (Important): Magic number (100)

[Implementer fixes]
Implementer: Extracted PROGRESS_INTERVAL constant

[Code reviewer reviews again]
Code reviewer: ✅ Approved

[Mark Task 2 complete]

...

[After all tasks]
[Dispatch final code-reviewer]
Final reviewer: All requirements met, ready to merge

Done!
```

## Advantages

**vs. Manual execution:**
- Subagents follow TDD naturally
- Fresh context per task (no confusion)
- Parallel-safe (subagents don't interfere)
- Subagent can ask questions (before AND during work)

**vs. Executing Plans:**
- Same session (no handoff)
- Continuous progress (no waiting)
- Review checkpoints automatic

**Efficiency gains:**
- No file reading overhead (controller provides full text)
- Controller curates exactly what context is needed
- Subagent gets complete information upfront
- Questions surfaced before work begins (not after)

**Quality gates:**
- Self-review catches issues before handoff
- Review depth matched to task risk: Routine (self-review only), Standard (one combined review), High-risk (two-stage spec then quality)
- Review loops ensure fixes actually work
- Spec compliance prevents over/under-building
- Code quality ensures implementation is well-built

**Cost:**
- Tiered reviews keep cost proportional to risk — most tasks get 0-1 reviewer dispatches, not 2
- Controller does more prep work (extracting all tasks + assigning tiers upfront)
- Review loops add iterations when issues surface
- Still catches issues early on the tasks where it matters (cheaper than debugging later)

## Red Flags

**Never:**
- Start implementation on main/master branch without explicit user consent
- Skip the review tier the task earned (Routine is a tier, not a skip — but Standard/High-risk tasks MUST get their review)
- Proceed with unfixed issues
- Dispatch multiple implementation subagents in parallel (conflicts)
- Make subagent read plan file (provide full text instead)
- Skip scene-setting context (subagent needs to understand where task fits)
- Ignore subagent questions (answer before letting them proceed)
- Accept "close enough" on spec compliance (reviewer found issues = not done)
- Skip review loops (reviewer found issues = implementer fixes = review again)
- **In High-risk tier: start code quality review before spec compliance is ✅** (wrong order)
- Move to next task while review has open issues
- Downgrade a task's tier to dodge work — if you're tempted to call a Standard task "Routine," it isn't

**If subagent asks questions:**
- Answer clearly and completely
- Provide additional context if needed
- Don't rush them into implementation

**If reviewer finds issues:**
- Implementer (same subagent) fixes them
- Reviewer reviews again
- Repeat until approved
- Don't skip the re-review

**If subagent fails task:**
- Dispatch fix subagent with specific instructions
- Don't try to fix manually (context pollution)

## Integration

**Required workflow skills:**
- **superpowers:using-git-worktrees** - Ensures isolated workspace (creates one or verifies existing)
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:requesting-code-review** - Code review template for reviewer subagents

**Subagents should use:**
- **superpowers:test-driven-development** - Subagents follow TDD for each task

**Alternative workflow:**
- **superpowers:executing-plans** - Use for parallel session instead of same-session execution
