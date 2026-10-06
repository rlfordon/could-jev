# Jev factsheet

Checked 2026-10-05. Sources: docs.typesafe.ai (primitives, models, state, jaggedness/jev-1.13), the official `typesafe-ai/skills` SKILL.md, the OpenRouter Jev guide, aaddrick/building-with-typesafe-jev prior-art (snapshot 2026-09-25), JyothiKumar03/just-jev-it fit-heuristics. Community numbers are self-reported. If something here looks stale, say so and re-check docs.typesafe.ai.

## What it is

TypeSafe's "System One" model. Input: a `state` plus named typed questions. Output: typed answers with full probability distributions. No text, no rationale, no citations, no retrieval, no memory between requests. "Typed output guarantees the interface, not truth."

It can only "point" at things your code enumerates as options (spans, line IDs, candidate values).

## Question types

| Type | Returns | Limits / gotchas |
|---|---|---|
| **Choice** | `choice`, `probabilities` (sum to 1), `confidence` | Max 255 options. Always picks a winner, so add `none`/`other`. Leans toward the first option; reorder to check. Probabilities are *relative*: add an absolute Noul for "is anything relevant?" |
| **Score** | `score` (probability-weighted, can fall between levels), `legend`, `probabilities`, `confidence` | 2 to 10 ordered levels. |
| **Noul** | `noul` = P(yes) | No `confidence` field. 0.5 means unsure, not "medium." |

Many named questions per request, mixed types, run in parallel over the same state, blind to each other. The model never sees the IDs, so the full question goes in `instructions`. Nested state can be referenced with backticked paths (`doc.paragraphs[3]`).

## Input, cost, speed, access

- **State:** a string, JSON object, or **array of texts**, so two documents fit side by side in one request.
- **Budget:** 64k tokens per request direct (32k for state plus the longest question). The OpenRouter guide says 32k total; assume 32k on OpenRouter.
- **Price:** $0.042 per million input tokens; output is free. Rule of thumb: a whole question tree over 72 opinions cost under a cent.
- **Speed:** about 0.1 to 0.4 s per request. Rate limits direct: 80 req/s and 100K tok/s; in practice about 8 concurrent workers per key before 429s.
- **Version:** `jev-1.13.0` (`jev-latest`). Log `response.model` and pin it when thresholds are tuned.
- **Access:** TypeSafe direct `POST https://api.typesafe.ai/v1/systemone`; OpenRouter `typesafe/jev-1.13` (Decisions API, or `https://openrouter.ai/api/v1/systemone` so the TypeSafe SDK works with a changed base URL); Vercel AI Gateway `typesafe-ai/jev`.
- **SDKs:** Python `typesafe-sdk` (`TypeSafeClient`, `AsyncTypeSafeClient`), JS `@typesafe-ai/sdk`.

```python
from typesafe_sdk import TypeSafeClient, Choice, Score, Noul
with TypeSafeClient() as client:   # reads TYPESAFE_API_KEY
    r = client.system_one(state={"text": doc}, questions={
        "relevant": Noul(instructions="Does `text` discuss X?"),
        "kind": Choice(instructions="...", criteria={"a": "...", "none": "..."}),
    })
r.answers["relevant"].noul; r.answers["kind"].choice; r.answers["kind"].probabilities
```

## Documented patterns worth stealing

- **Speculative fan-out:** ask every question you might need in one request (about 10x cheaper and faster than sequential).
- **Pre-parsed value extraction:** regex over-finds candidates; one Choice picks; code copies the value verbatim. Jev cannot invent or transpose a value.
- **Line-by-line search:** one Choice over numbered line IDs (218 in the cookbook) finds where something is.
- **Rerank a shortlist:** Noul per (query, candidate) pair on a BM25 top-30 of CLERC legal queries raised top-1 from 5% to 18% and top-10 from 38% to 62%. Shortlist only; never the whole corpus.
- **Hierarchical taxonomy:** walk the tree level by level with beam search (K=3 beat greedy). Needed past 255 options.
- **Gates:** hard rules in code first; Jev can only make a gate stricter. Combine flags with max, not mean.
- **Composite scoring:** atomic Scores weighted in code; policy runs on cached probabilities, so reweighting costs nothing.
- **Cascades:** Jev decides the easy bulk, an LLM gets the uncertain residue.
- **Date extraction:** Choice over month, day, year parts plus "not stated"; date math in code.

## Where it breaks (TypeSafe's jaggedness list for 1.13, plus field reports)

1. Reads instructions literally. Indirection and double negatives hurt; inverted Noul criteria hurt.
2. Arithmetic, counting, dates, comparisons: do these in code.
3. Distractors ("context rot"): long state with irrelevant detail lowers accuracy. Window or filter first.
4. Adversarial or injected state: not a security boundary.
5. Choice order bias toward the first option.
6. Calibration taken on trust: overconfident out of distribution and on contested items; one temperature does not fix both. Fit thresholds on your own labels.
7. Field flops: chess and search-like tasks, code review as the only reviewer, record dedupe, a personal prompt router ("OK at best"). A tiny distilled specialist beat it on a narrow form-filling task (99.7% vs 83.6%), and batched offline LLM prompts can match its cost.
8. Text only; English strongest.

## Fit signals

- **Jev:** the answer space is known in advance; one narrow judgment; the evidence fits in the state; an expert could judge it in seconds; code branches on the result; you'd want to ask it thousands of times or in real time.
- **LLM:** prose output, a rationale or audit trail is needed, the options aren't known, it needs a retrieval or tool loop mid-decision, or it is multi-hop reasoning.
- **Code:** arithmetic, dates, exact lookups, known rules, regex-findable units, security rules, threshold policy.
- **Always:** add `none` or an absolute Noul; validate on in-domain labels including hard cases; log the model version.
