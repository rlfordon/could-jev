---
name: could-jev
description: Quick gut-check on whether Jev (TypeSafe's System One typed-judgment model) could help with whatever the user is building right now, grounded in a catalog of non-obvious Jev patterns (funnels, censuses, taxonomies, locators, heat maps, pairwise judgments, uncertainty spotlights, tripwires, live feedback), rules of thumb from real use, and the user's own logged Jev results. Use when they ask "could jev do this/help here", "would jev work for", "is this a jev thing", "jev for X?", "what would jev be good for in this project", or wonder aloud about using a cheap, fast typed-judgment model in a design. Also use to log the result of a Jev experiment ("log this jev result"). Answers in a few lines; does not call Jev itself.
---

# Could Jev do this?

A fast, opinionated answer to "huh, could Jev help with what I'm building?" Lightweight on purpose: read, match, answer in about ten lines. Don't build anything unless asked.

## Files

In this skill folder:
- `patterns.md`: rules of thumb, worth-testing hunches, and what Jev makes possible, with a status tag on each pattern.
- `capabilities.md`: the factsheet (limits, question types, price, access).
- `history-template.md`: the starting shape of a personal history file.

Outside it:
- **The user's history**, at `~/.claude/could-jev/history.md`. It may not exist yet. It's kept outside the skill folder so plugin updates never overwrite it.

## Steps

1. **Pin down the thing.** Use the conversation and, if needed, a glance at the code or doc in play. Don't ask a clarifying question unless you truly can't tell what's being built; state your assumption instead.
2. **Read `patterns.md` and the user's history if it exists.** Read `capabilities.md` only when a limit matters (token budget, option counts, question-type behavior, price, access), and always glance at its "Checked" date: if it's more than about 60 days old, add a one-line note that the factsheet may be stale and that the user should update the skill or check docs.typesafe.ai.
3. **Look past "is the output a label?"** Ask which of Jev's levers this task could use: nearly free, fast, full distributions, many questions per state, several texts per state. The best answers often reshape the task rather than swapping a model into it: a census instead of a sample, a heat map instead of a search, a funnel in front of an existing LLM step.
4. **Answer in this format**, no preamble:

```
**Verdict:** Yes / Partly / No, plus one line on why.
**Pattern:** <pattern name(s) from patterns.md>, with its status tag.
**Sketch:** state = <what text, windowed how>; questions = <named, typed: Noul/Choice/Score with the gist of each>; code does <the rest>.
**Closest prior:** <the user's own experiment # and the number that matters, else a TESTED/DOCUMENTED/COMMUNITY pattern>, or "nothing close."
**Watch out:** <1 to 2 specific failure modes from the rules of thumb, the user's history, or the jaggedness list>.
**Cheapest test:** <the smallest labeled set and run that would settle it; fold in a "worth testing" hunch when one applies>.
**Bigger idea:** <one non-obvious way Jev could change this design, if there is one>.
```

On a **No**, say what should do it instead (code, an LLM, or nothing), and still give the Bigger idea if there is one.

## Rules of judgment

- Standing rules in the user's history outrank the rules of thumb in `patterns.md`, and both outrank generic advice. Treat the "worth testing" items as hypotheses: suggest a test, don't state them as fact.
- Cost is almost never the deciding factor. Say whether good accuracy is plausible and how the user would find out.
- Be honest about the middle band. If the decision they care about lives in the ambiguous zone (partial support, inferred conclusions), say Jev will mostly escalate it.
- Confidential or private data never goes in the state unless the user has confirmed that's acceptable for their TypeSafe account.
- If something in `capabilities.md` seems stale or contradicted, say so and suggest re-checking docs.typesafe.ai; don't silently trust it.

## Logging

When the user reports a result from a Jev experiment, or says "log this":

1. If `~/.claude/could-jev/history.md` doesn't exist, create it by copying `history-template.md`.
2. Append an entry to the end of "Experiments", following the commented example in the template: number, name, project, date, status, then 2 to 5 bullets with exact numbers and file paths.
3. Add a lesson to "Standing rules" only if it generalizes beyond that experiment.
4. If the user decides to try an idea, add a one-line entry under "Ideas raised but not yet tried".

Never write to files inside the skill folder. Tell the user what you added in one sentence; don't ask permission for these appends.
