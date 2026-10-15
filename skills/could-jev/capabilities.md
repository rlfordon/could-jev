# Jev factsheet

Checked 2026-10-15.

**Sources:**
- docs.typesafe.ai: models, api, primitives, confidence, state, patterns, cookbooks, SDK changelogs, and the jaggedness page for jev-1.13 (last reviewed 2026-10-02). It wasn't reachable on 2026-10-15; the 1.14 and "danger zone" facts below come from the SDK packages themselves (Python 0.7.5, JS 0.7.0, `@ai-sdk/typesafe-ai` 3.0.19) and from coverage of TypeSafe's 1.14 announcement.
- The `typesafe-ai` GitHub org.
- The OpenRouter Jev guide.
- The Vercel AI Gateway docs.
- Community repos: aaddrick/building-with-typesafe-jev and JyothiKumar03/just-jev-it.

Community numbers are self-reported. Most cookbook numbers were measured on **jev-1.12**, not 1.13; only the self-consistency cookbooks ran on 1.13. Nothing here has been re-measured on **1.14**. If something here looks stale, say so and re-check docs.typesafe.ai.

## What it is

TypeSafe's "System One" model. You send a `state` plus named, typed questions. You get back typed answers with full probability distributions.

There's no text, no rationale, no citations, no retrieval, no memory between requests, and no fine-tuning. "Typed output guarantees the interface, not truth."

In text, it can only "point" at things your code enumerates as options: spans, line IDs, candidate values, function names. The one exception is the experimental Box question, which returns a region of an image.

## Question types

| Type | Returns | Limits / gotchas |
|---|---|---|
| **Choice** | `choice`, `probabilities` (sum to 1), `confidence` | Max 255 options. Always picks a winner, so add `none`/`other`. Leans toward the first option; reorder to check. Probabilities are *relative*: add an absolute Noul for "is anything relevant?" |
| **Score** | `score` (probability-weighted, can fall between levels), `legend`, `probabilities`, `confidence` | 2 to 10 ordered levels; `criteria` is an ordered list (JS SDK 0.6.0+). Don't read the expectation as an exact magnitude. |
| **Noul** | `noul` = P(yes) | No `confidence` field. 0.5 means unsure, not "medium." Instructions are optional if `criteria` describes an outcome (SDK 0.7.5+). |
| **Box** *(experimental, images only, jev-1.14+)* | `box` = `top`, `bottom`, `left`, `right` as fractions (0 to 1) of image height/width, each the *expected* edge position; `confidence` (may be `null`) | One box per question, so no "not present" answer: pair it with a Noul ("is there a ...?"). No distribution over positions. Not exposed by the Vercel AI SDK provider. |

**How questions work:**
- You can ask many named questions per request, of mixed types. They run in parallel over the same state and are blind to each other.
- The model never sees the IDs, so the full question goes in `instructions`.
- Instructions, options and levels can be JSON objects, and (experimentally) images.
- You can reference nested state with backticked paths (`doc.paragraphs[3]`).
- The confidence formulas are published at /confidence.

**Answers vary from run to run.** On 1.13, the mean per-question SD was about 0.01, but a Noul near the line can swing across 0.5 (0.43 to 0.53 in TypeSafe's cookbook), and Choice labels repeated about 91% of the time. Don't trust a threshold sitting right at a single run's value. Cache responses, or repeat-and-vote near the line.

## Input, cost, speed, access

**State:** a string, a JSON object, or an **array of texts**, so two documents fit side by side in one request.

**Budget:** 64k tokens per request direct, of which 32k is for the state plus the longest question. OpenRouter allows 32k total.

**Price:** $0.042 per million input tokens; output is free. Rule of thumb: a whole question tree over 72 long documents costs under a cent.

**"Danger zone" (experimental, opt-in per request by header, as of 2026-10-14).** TypeSafe says there's "still way too much jaggedness" to vouch for these in production, and that multimodal results vary more than text. Treat every number as unmeasured until you test on your own data.
- **Images** (`TypeSafe-Danger-Zone-Images: true`; jev-1.13.0 and 1.14.0):
  - PNG or JPEG only (WebP and GIF are rejected). Sent as `multipart/form-data`.
  - An image can go anywhere in `state`, `instructions` or `criteria`, by a placeholder reference listed in an `assets` map. So you can ask about one photo, compare two ("is A the product in B?"), or use reference images as Choice options.
  - Billed as input tokens at the normal rate. One third-party report put a 1181×1568 photo at about 5,200 tokens (about $0.0002). **Every mention of the reference inserts the image again**, and is billed again.
  - Reported (not in official docs seen): 16 MiB per image, and 256 to 1,568 px per side. Smaller images return 400 (one user hit this at 278 px, so upscale small crops) and larger ones return 413. EXIF orientation is ignored, so rotate first. One guide says any request with an image runs in long-context mode.
  - Many people send screenshots, so Choice over numbered on-screen elements is the common shape for UI agents.
  - Python SDK: `Image(path_or_bytes)`, `Image.from_base64`, `Image.from_pil` (`pil` extra). It doesn't add the opt-in header for you, so pass it in `default_headers`. JS SDK: `image()`, the `withAssets` template tag. The Vercel AI SDK provider adds the header itself. The **Vercel AI Gateway rejects files**, so use TypeSafe direct.
- **Box questions:** see the table above. Only on 1.14 (`jev-preview` for now); 1.13 rejects it with a 400. An anecdotal test located 30 of 36 targets when it asked four per-edge questions instead, vs. 20 of 36 for the raw box. Draw it and check before you trust it.
- **Long context** (`TypeSafe-Danger-Zone-Long-Context: true`; 1.13 and 1.14): raises both limits to 262k tokens at the same per-token price. TypeSafe calls it slower and *less accurate*, which fits jaggedness item 3 (distractors). Windowing is still the better move.

**Speed:**
- About 0.1 to 0.4 s per request.
- Direct rate limits are 80 req/s and 100K tok/s, and TypeSafe says these "can change without notice".
- In practice, expect 429s beyond about 8 concurrent workers per key.
- The API can also return `529 Overloaded`, so retry with backoff.

**Version:**
- **`jev-1.14.0` was released about 2026-10-14** behind `jev-preview` ("latest release, including previews"). `jev-latest` ("latest stable") still pointed to `jev-1.13.0` at that point, and was announced to move to 1.14 about a week later (around 10-21). Check `GET /v1/models`.
- Deprecation notice: 60 days for 1.13.0, 30 days for 1.14.0.
- Log `response.model`, and pin the version once thresholds are tuned. **If you used `jev-latest` and tuned thresholds on 1.13, pin `jev-1.13.0` now and re-fit on 1.14 before switching.**

**Access:**
- **TypeSafe direct:** `POST https://api.typesafe.ai/v1/systemone`.
- **OpenRouter:**
  - Model `typesafe/jev-1.13` (the SDK docs use `~typesafe/jev-latest`).
  - Use `https://openrouter.ai/api/v1/systemone` as the base URL for the TypeSafe SDK, or the Decisions API at `/api/alpha/decisions`.
  - Separately, `typesafe/jev-router` is a chat-completions *model router* built on Jev. It isn't the decision API.
- **Vercel AI Gateway:**
  - Model `typesafe-ai/jev`, plus a generic Decision API (`/v1/evaluate`).
  - The AI SDK provider is `@ai-sdk/typesafe-ai` (`experimental_decide()`). It **rounds probabilities to 2 decimals**.
  - It has built-in **decision fallbacks**: rerun on an LLM when `confidenceBelow` or `probabilityBetween` matches. A fallback answer comes back with `confidence: 0, probabilities: {}`.
- **Pydantic AI Gateway:** `https://gateway-us.pydantic.dev/proxy/typesafe`.
- **n8n:** a community node with Evaluate and Route operations.

**SDKs:**
- Python `typesafe-sdk` 0.7.x (`TypeSafeClient`, `AsyncTypeSafeClient`). 0.7.0 moved to pydantic and added `response_model=`. 0.7.5 added `Image`, `Box`, `response.boxes`, and zstd request compression, which is on by default (Python 3.14+ or the `zstd` extra). Extras: `http2`, `pil`, `zstd`.
- JS `@typesafe-ai/sdk` 0.7.x (needs Node 22+). 0.7.0 added images, `box()`, `withAssets`, and zstd compression for bodies of 1 KiB or more (Node 22.15+).
- Vercel `@ai-sdk/typesafe-ai` 3.0.19 has model IDs `jev-1.13.0`, `jev-1.14.0`, `jev-latest` and `jev-preview`, plus image file parts and `longContext`. It has no Box question. `TYPESAFE_AI_BASE_URL` points it at a proxy or a compatible server.
- `system-one-adapter-python` serves the same API from an LLM, which is handy for A/B testing Jev against an LLM.

**Data:** zero data retention is enterprise-only.

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

**Asking:**
- **Speculative fan-out:** ask every question you might need in one request (about 10x cheaper and faster than sequential).
- **Self-consistency:** repeat the request and keep answers that agree. On 1.13, a 0.60 top-probability floor auto-labelled 74% of items at 99.2% agreement.

**Extracting and locating:**
- **Pre-parsed value extraction:** regex over-finds candidates, one Choice picks, and code copies the value verbatim. Jev can't invent or transpose a value.
- **Line-by-line search:** one Choice over numbered line IDs (218 in the cookbook) finds where something is.
- **Structure recovery:** ask one Noul per adjacent line pair ("same paragraph?") to stitch hard-wrapped text, then classify each block.
- **Date extraction:** a Choice over month, day and year parts plus "not stated"; date math in code.

**Ranking and classifying:**
- **Rerank a shortlist:** a Noul per (query, candidate) pair on a BM25 top 30 of CLERC legal queries raised top-1 from 5% to 18% and top-10 from 38% to 62%. Use a shortlist only, never the whole corpus.
- **Hierarchical taxonomy:** walk the tree level by level with beam search (K=3 beat greedy). Needed past 255 options.
- **Confidence back-off:** below a confidence floor, return the *parent* label instead of the leaf. On industry codes, this took the uncertain half from 40% to 70% accuracy, with no extra call.
- **Entity alignment:** one Score for "same entity?" plus one Noul per field to show *which* fields disagree.
- **Jev answers as features:** feed the distributions into a gradient-boosted model (TypeSafe's "autoresearch" cookbook).

**Gating and routing:**
- **Gates:** hard rules in code first; Jev can only make a gate stricter. Combine flags with max, not mean.
- **Composite scoring:** atomic Scores weighted in code. The policy runs on cached probabilities, so reweighting costs nothing.
- **Cascades:** Jev decides the easy bulk and an LLM gets the uncertain residue. Variant: Jev as the *verifier between LLM rungs*.
- **Routing and function calling:**
  - Intent routing and confidence-gated routing.
  - A function name, with closed-set arguments as enums.
  - Skill suggestion: wrong skill loads fell from 16.8% to 7.3% over 488 requests.
- **RAG passage triage:** per retrieved passage, ask relevant / usable / contradicts / looks like an injection.

## Where it breaks

TypeSafe's official jaggedness list for 1.13 (9 items). The 1.14 page wasn't reachable on 2026-10-15; assume these all still apply until it's checked.

1. **Literal reading.** Indirection, double negatives and inverted Noul criteria hurt.
2. **Arithmetic, counting, dates, comparisons.** Do these in code. To count, ask one Noul per item and sum in code. Numeric encodings also do worse than names (e.g. hex or RGB values vs. color names).
3. **Distractors ("context rot").** Long state with irrelevant detail lowers accuracy. Window or filter first.
4. **Adversarial or injected state.** Not a security boundary.
5. **Order bias** toward the first Choice option.
6. **Score magnitudes.** Don't interpolate exact magnitudes from a Score.
7. **Contradictory instructions and criteria.** When `instructions` and `criteria` disagree, results degrade.
8. **Text only.** English is strongest. *Since 1.14 (2026-10-14), images are an experimental opt-in; audio and video are still unsupported.* No official image jaggedness list was seen; TypeSafe says image results vary more than text.
9. **Generation.** It isn't trained to generate text.

Field reports, not on TypeSafe's list:

- **Calibration taken on trust.** It runs overconfident out of distribution and on contested items, and one temperature doesn't fix both. Fit thresholds on your own labels.
- **Flops reported by users:**
  - Chess and search-like tasks.
  - Code review as the only reviewer.
  - Record dedupe.
  - A personal prompt router ("OK at best").
- **Losing to cheaper or narrower alternatives:**
  - A tiny distilled specialist beat Jev on a narrow form-filling task (99.7% vs. 83.6%).
  - Batched offline LLM prompts can match its cost.
- **Adversarial text:** in one guardrail test (1,000 items), Jev caught 78% of adversarial rewrites vs. 94% for Haiku 4.5, at about 6x lower latency and 60x lower cost.
- **1.14 vs. 1.13 (early, community):** one 1,136-question benchmark went from 80.1% to 82.0%, with the largest gain on verbal spatial tracking and a small drop on exception and priority rules. Multi-step sequential procedures stayed very weak (6 of 42). One migration test saw identical failures on both versions.
- **Benchmark gap:** on TypeSafe's own WorkflowEvals, Jev scored about 68% vs. about 74% for a frontier LLM, at a tiny fraction of the cost and latency.

## Fit signals

- **Jev:** the answer space is known in advance; it's one narrow judgment; the evidence fits in the state; an expert could judge it in seconds; code branches on the result; you'd want to ask it thousands of times or in real time.
- **LLM:** you need prose output or a rationale/audit trail; the options aren't known; it needs a retrieval or tool loop mid-decision; or it takes multi-hop reasoning.
- **Code:** arithmetic, dates, exact lookups, known rules, regex-findable units, security rules, threshold policy.
- **Images (experimental):** the judgment is visible at a glance (damaged? same product? is the element on screen?), and a crop or a pair of images holds the evidence. Not for pixel-exact geometry, and nothing should go to production before TypeSafe drops the "danger zone" label.
- **Always:** add `none` or an absolute Noul; validate on in-domain labels, including hard cases; log the model version.
