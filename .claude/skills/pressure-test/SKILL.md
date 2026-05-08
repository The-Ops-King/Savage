---
name: pressure-test
description: Adversarially review a plan, design, or change-in-progress to surface blind spots, risks, overengineering, and future regret before it ships. Use when the user wants to pressure-test their thinking, asks what they're missing, or mentions "pressure test".
---

Pressure-test the plan, design, or code the user is working on (use `$ARGUMENTS` if supplied, otherwise whatever is currently in context — the active plan, recent diff, or file under discussion).

Goal: surface what's missing, what's overcomplicated, and what will hurt later — before any of it ships. Output is a punch list, not a redesign.

## Rules

- Ask one question at a time. Wait for the answer before the next one.
- Always include your recommended answer with the question, with reasoning.
- If the codebase or docs can answer the question, go look — don't ask.
- Don't redesign. Don't write code. Don't propose new features.
- Bias toward subtraction. The default verdict on new complexity is "no".
- Stop when the punch list is concrete and the user is satisfied — not on a fixed count.

## Lenses

Rotate through these, picking whichever is most load-bearing for the current decision. Not all of them, not in order — pick the one most likely to expose a real problem next.

- **Blind spots** — what is the user not seeing or not thinking through?
- **Risk** — what breaks under load, edge cases, partial failure, or six months from now?
- **Overengineering** — what abstraction, flag, handler, config knob, or "future-proofing" can be deleted right now?
- **Future regret** — what will they wish they had built, or wish they hadn't?
- **Senior-engineer lens** — what would an expert do here, and why?
- **Prior art** — what's the standard pattern for this situation, and is there a real reason to deviate?
- **Simplification** — what's the smallest version that still ships the value?
- **Reversibility** — is this a one-way door? If so, is the cost of being wrong proportionate to the confidence?

## Output

End the session with a short punch list. One line per item, grouped:

- **KEEP** — decisions that survived scrutiny
- **CUT** — complexity to remove now
- **FIX** — concrete changes before shipping
- **DEFER** — explicitly chosen "not now, maybe later"

If a `WATCH` item belongs (something to revisit at a known trigger), add it.
