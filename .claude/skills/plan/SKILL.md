---
name: plan
description: Create or update the project implementation plan in Plan.md at the repo root. Use whenever the user asks to "plan", "make a plan", "update the plan", "revise the plan", or discusses architectural/implementation decisions that should be recorded. The plan must always include a Pros & Cons section reflecting the current approach.
---

# Plan skill

Maintain a single source of truth for the project's implementation plan at `Plan.md` in the repo root (`/Users/rmli/code_stuff/ingredient_chart_viz/Plan.md`).

## When invoked

1. **Check if `Plan.md` exists.**
   - If it does NOT exist: generate it from scratch using the structure below.
   - If it DOES exist: read it first, then update it in place — preserve sections the user hasn't asked to change, and revise the sections affected by the new request. Never silently discard prior content; if removing something, note it under a "Changed in this revision" line at the top of the affected section or in a short changelog at the bottom.

2. **Always include a Pros & Cons section** reflecting the *current* approach. When updating, re-evaluate the pros and cons against the revised plan rather than copying the old list verbatim — stale tradeoffs are worse than none.

3. **Write the file with the Write tool** (or Edit tool if only targeted changes are needed). Do not print the full plan back to the user in chat — they can read the file. Summarize the delta in 1–3 sentences.

## Required structure for `Plan.md`

```
# <Project name> — Plan

## Overview
One paragraph: what we're building and why.

## Stack
Key technologies and the reason each was picked.

## File layout
Tree of the files/directories the plan creates or touches.

## Data model
(If relevant.) Shape of the core data.

## Behavior
What the thing does — interactions, views, key UX.

## How to run
Exact commands or steps to see it working locally.

## Implementation phases
Break work into 2–4 sequential, independently-runnable phases. Each phase MUST include:
- A short **Goal** line: what the phase delivers in one sentence.
- A **numbered step list** (5–10 steps) that is concrete enough to implement — name files, functions, config values, not vague verbs like "set up" or "handle."
- An **Exit criterion**: a sentence a tester could verify against the running app before moving to the next phase.

Phases are sequential. Do not start phase N+1 until phase N passes its exit criterion. The first phase is always the smallest shippable MVP; each later phase adds one coherent capability. If the project is trivial (a single script, a one-file fix), collapse this into a single "Steps" list instead of forcing phases.

## Out of scope
Things deliberately deferred.

---

## Pros & Cons

### Pros
- Bolded one-liner, then a sentence of detail. 3–6 items.

### Cons
- Bolded one-liner, then a sentence of detail. 3–6 items. Include at least one con that reflects the phased plan itself (scope creep risk, phase-boundary handoffs, state that carries between phases).

### Bottom line
1–3 sentences naming the biggest risks or decisions to lock down before moving on. When the plan has phases, sequence the decisions by the phase that forces them (e.g. "lock X before Phase 2, lock Y before Phase 3").
```

## Rules

- **One plan file.** Do not create `Plan-v2.md`, `Plan-old.md`, etc. Update in place.
- **Pros & Cons are mandatory** on every write, and must match the current plan, not a previous version.
- **No filler.** Skip any section that doesn't apply to the current plan rather than padding it.
- **Match existing tone.** Terse, concrete, no marketing language.
- **If the user asks a planning question without asking to save**, answer in chat first; only write `Plan.md` when they confirm or explicitly ask to record it.
- **If the user asks for a plan revision**, diff mentally against the existing file: what changed, what stayed, what got dropped. Reflect the drop in the pros/cons if it materially changes tradeoffs.
