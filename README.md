# could-jev

A Claude skill that answers one question fast: **could [Jev](https://docs.typesafe.ai) help with what I'm building?**

Jev is TypeSafe's "System One" model. It returns typed answers (yes/no, one-of-N, ordinal scores) with full probability distributions, at a fraction of a cent per thousand documents and 0.1 to 0.4 s per call. The obvious use is "classify X". The interesting uses are things like funnels in front of an LLM, censuses instead of samples, locators, paragraph heat maps, pairwise judgments, uncertainty spotlights, and agent tripwires. This skill helps you spot those.

Ask Claude something like *"could jev help here?"* or *"is this a jev thing?"*. You get back about ten lines:

```
**Verdict:** Partly — the easy 80% is a clean yes/no; the ambiguous middle needs an LLM.
**Pattern:** Funnel before the LLM (FIELD #3), Uncertainty spotlight (UNTRIED).
**Sketch:** state = ...; questions = ...; code does ...
**Closest prior:** ...
**Watch out:** ...
**Cheapest test:** ...
**Bigger idea:** ...
```

It doesn't call Jev itself. It's a design gut-check.

## What's inside

| File | What it is |
|---|---|
| `SKILL.md` | The instructions and answer format. |
| `patterns.md` | 26 Jev patterns grouped by what they unlock, each tagged FIELD-tested, DOCUMENTED, or UNTRIED. |
| `field-notes.md` | Standing rules and anonymized numbers from eight real experiments (mostly legal-research tooling). |
| `capabilities.md` | Factsheet: question types, limits, price, speed, access, failure modes. Checked 2026-10-05. Re-check docs.typesafe.ai if it looks stale. |
| `history-template.md` | The starting shape of your personal experiment log. |

## Your own history

Say *"log this jev result"* after an experiment and the skill appends it to `~/.claude/could-jev/history.md`. It creates that file from the template the first time. Your history and standing rules then take priority over the generic field notes. The history lives outside the skill folder, so updating the plugin never overwrites it.

## Install

**Claude Code (plugin):**

```
/plugin marketplace add rlfordon/could-jev
/plugin install could-jev@could-jev
```

**Claude Code (manual):** copy `skills/could-jev/` to `~/.claude/skills/could-jev/`.

**Claude.ai / Claude Desktop:** download `could-jev.zip` from the [latest release](https://github.com/rlfordon/could-jev/releases/latest) and upload it under Settings → Capabilities → Skills. Logging to a history file needs filesystem access, so it works best in Claude Code.

## License

MIT
