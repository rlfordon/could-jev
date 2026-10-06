# Jev state-of-the-art sweep

A repeatable procedure for keeping `skills/could-jev/` current. Run it monthly, either by hand or from a scheduled agent. A sweep produces a dated report in `maintenance/sweeps/` and a small, reviewable set of edits to the skill.

## 0. Know what's already known

Read these files first, so you report only what's new:
- `skills/could-jev/capabilities.md`. Note its "Checked" date.
- `skills/could-jev/patterns.md`. These are the patterns already catalogued.
- The most recent report in `maintenance/sweeps/`. Its "Seen" list holds the sources already reviewed.

## 1. Official changes

Check each of these. For each, record what changed since the last "Checked" date, or "no change".
- **docs.typesafe.ai:** model versions (anything newer than the one pinned in `capabilities.md`?), the changelog, the primitives and question types, limits (options, token budget, rate limits), pricing, and the jaggedness page for the current version.
- **The `typesafe-ai` GitHub org:** the SDK releases (`typesafe-sdk` on PyPI, `@typesafe-ai/sdk` on npm), the official skills repo, and the cookbooks.
- **Access routes:** OpenRouter (`typesafe/jev-*`), the Vercel AI Gateway, and any new ones.
- **TypeSafe's blog and announcements:** new features (new question types? spans? batching? streaming?) and official benchmarks.

## 2. What people are building

Search widely; recent and concrete beats general. Places to look:
- **GitHub:** code search for `typesafe_sdk`, `TypeSafeClient`, `systemone`, `@typesafe-ai/sdk`, `jev-1.`, plus repo search for "jev" and "typesafe". Note the star count, last commit date, and whether the repo reports numbers.
- **Discussion sites:** Hacker News (Algolia search), Reddit, X/Bluesky, and blogs and Substack ("Jev" together with "TypeSafe").
- **Model directories and forums:** OpenRouter model page activity, plus the Discord or forum if public.
- **Evals:** any public benchmark or head-to-head comparison against small LLMs, embeddings, or fine-tuned classifiers.

For each candidate use, record:
- **What it does,** in one line.
- **Which levers it uses:** cheap, fast, distributions, fan-out, multi-text state.
- **Evidence:** reported numbers / demo only / claim only.
- **Novelty:** a new pattern, a new variant or domain for an existing pattern (name it), or nothing new.
- **Link and date.**

Also collect **failures and limits** people report. They matter as much as wins.

## 3. Judge

- **Promote a use to `patterns.md`** only if it's a genuinely different *shape* of use, not just a new domain for an existing pattern. Tag it **COMMUNITY** with a link. A new domain or variant gets one clause added to the existing pattern instead.
- **Prefer reported numbers.** A claim with no numbers can go in the report, but doesn't go in the skill.
- **Update `capabilities.md`** for any official change, and bump its "Checked" date. If the model version changed, say in the report which jaggedness items or rules of thumb might no longer hold.
- **Promote a "worth testing" hunch to a rule of thumb** only when independent reports confirm it. Demote or remove one that turns out to be contradicted.
- **Never add** personal or confidential data, or vendor marketing claims stated as fact.

## 4. Write up

Create `maintenance/sweeps/YYYY-MM-DD.md` with these sections:
- **Official changes**
- **New patterns:** what was promoted to the skill.
- **Variants / domains:** what was folded into existing patterns.
- **Reported failures / limits**
- **Interesting but unproven:** in the report only.
- **Seen:** every source reviewed, so the next sweep can skip it.

Then make the skill edits, bump `version` in `.claude-plugin/plugin.json` (patch for factsheet-only changes, minor for new patterns), and open a PR titled `Sweep YYYY-MM-DD` whose body is the report. If nothing material changed, still commit the report, but don't bump the version.
