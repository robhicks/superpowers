---
name: using-superpowers
description: Use when starting any conversation - establishes how to find and use skills, requiring Skill tool invocation before ANY response including clarifying questions
---

<SUBAGENT-STOP>
If you were dispatched as a subagent to execute a specific task, skip this skill.
</SUBAGENT-STOP>

<EXTREMELY-IMPORTANT>
If there is even a 1% chance a skill might apply to what you're doing, you MUST invoke it via the `Skill` tool. You do not have a choice. You cannot rationalize your way out.
</EXTREMELY-IMPORTANT>

## Priority Order

1. **User instructions** (CLAUDE.md, direct requests) — highest
2. **Superpowers skills** — override default system behavior where they conflict
3. **Default system prompt** — lowest

If CLAUDE.md says "don't use TDD" and a skill says "always use TDD," follow CLAUDE.md.

## The Rule

**Invoke skills BEFORE any response or action**, including clarifying questions. When you invoke a skill, its content is loaded — follow it directly. Never use Read on skill files. If a skill has a checklist, create a TodoWrite todo per item.

These rationalizations all mean STOP and check skills first: "just a simple question", "let me explore the codebase first", "I remember this skill", "the skill is overkill", "this doesn't count as a task", "I'll just do this one thing first". Knowing a concept is not the same as invoking the skill.

## When Multiple Skills Apply

1. **Process skills first** (brainstorming, systematic-debugging) — they determine HOW to approach the task
2. **Implementation skills second** — they guide execution

"Let's build X" → brainstorming first. "Fix this bug" → systematic-debugging first.

## Skill Types

- **Rigid** (TDD, debugging): Follow exactly. Don't adapt away discipline.
- **Flexible**: Adapt principles to context.

The skill itself tells you which.
