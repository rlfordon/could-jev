# Jev factsheet

Checked 2026-10-24.

**Sources:**
- docs.typesafe.ai: models, api (including openapi.json), primitives (including the Box page), images, confidence, state, patterns, cookbooks, SDK changelogs, and the jaggedness page for jev-1.13 (last reviewed 2026-10-02). As of 2026-10-24 there is still no 1.14 jaggedness page and no product changelog.
- TypeSafe's official agent skill (`typesafe-ai/skills`, v0.6.2, 2026-10-23), which has newer design guidance than the docs.
- TypeSafe's 1.14 announcement (typesafe.ai/blog/jev-1-14), the only official source for the `jev-latest` move and the deprecation timeline.
- The SDK packages themselves (Python 0.7.5, JS 0.7.0, `@ai-sdk/typesafe-ai` 3.0.26, `ai` 7.0.152).
- The `typesafe-ai` GitHub org.
- The OpenRouter Jev guide.
- The Vercel AI Gateway docs and model listings, and the DigitalOcean Gradient models page.
- Community repos: aaddrick/building-with-typesafe-jev and JyothiKumar03/just-jev-it.

Community numbers are self-reported. Most cookbook numbers were measured on **jev-1.12**, not 1.13; only the self-consistency cookbooks ran on 1.13. Almost nothing here has been re-measured on **1.14**, which is now the default (`jev-latest`). If something here looks stale, say so and re-check docs.typesafe.ai.

## What it is

TypeSafe's "System One" model. You send a `state` plus named, typed questions. You get back typed answers with full probability distributions.

There's no text, no rationale, no citations, no retrieval, no memory between requests, and no fine-tuning. "Typed output guarantees the interface, not truth."

In text, it can only "point" at things your code enumerates as options: spans, line IDs, candidate values, function names. The one exception is the experimental Box question, which returns a region of an image.

## Question types

| Type | Returns | Limits / gotchas |
|---|---|---|
| **Choice** | `choice`, `probabilities` (sum to 1), `confidence` | Max 255 options. Always picks a winner, so pair it with a Noul asking whether anything fits; TypeSafe's skill says that on 1.14 this "usually works better than a 'none of the above' option". Use descriptive option names: positional ones (`A`, `option_1`) "can bias the answer". Leans toward the first option; reorder to check. Probabilities are *relative*, so only that companion Noul says whether anything is relevant at all. |
| **Score** | `score` (probability-weighted, can fall between levels), `legend`, `probabilities`, `confidence` | 2 to 10 ordered levels; `criteria` is an ordered list (JS SDK 0.6.0+). Don't read the expectation as an exact magnitude: the position "can get pulled toward the middle and hide two-peaked answers", so look at the level probabilities. Describe each level as a situation that makes sense on its own. |
| **Noul** | `noul` = P(yes) | No `confidence` field. 0.5 means unsure, not "medium." TypeSafe's suggested starting thresholds are 0.7 (yes) and 0.3 (no), with the band between treated as unsettled. Describe `false` as a concrete situation, not "not X". Instructions are optional if `criteria` describes an outcome (SDK 0.7.5+). |
| **Box** *(experimental, images only, jev-1.14+)* | `box` = `top`, `bottom`, `left`, `right` as fractions (0 to 1) of image height/width, with (0,0) at top-left; `confidence` 0 to 1 (the JS type allows `null` "when the API didn't report it") | `instructions` only, no `criteria`; it can be a description ("the red mug"). One box per question and no "not found" answer: ask a presence Noul in the same request and ignore the box when it's low. For several matches, describe the one you want, or add a Choice (none/one/several). Small targets in big images get missed: pick a tile with a Choice first, then Box the tile. No distribution over positions, and no recommended confidence threshold. In the Vercel AI SDK provider since 3.0.21 (`box` question type; confidence in `providerMetadata.typesafe.confidence`). |

**How questions work:**
- You can ask many named questions per request, of mixed types. They run in parallel over the same state and are blind to each other.
- The model never sees the IDs, so the full question goes in `instructions`.
- Instructions, options and levels can be JSON objects, and (experimentally) images.
- You can reference nested state with backticked paths (`doc.paragraphs[3]`).
- The confidence formulas are published at /confidence.

**Answers vary from run to run.** On 1.13, the mean per-question SD was about 0.01, but a Noul near the line can swing across 0.5 (0.43 to 0.53 in TypeSafe's cookbook), and Choice labels repeated about 91% of the time. Don't trust a threshold sitting right at a single run's value. Cache responses, or repeat-and-vote near the line.

## Input, cost, speed, access

**State:** a string, a JSON object, or an **array of texts**, so two documents fit side by side in one request.

**Budget:** 64k tokens per request direct, of which 32k is for the state plus the longest question. There is no limit on the number of questions. OpenRouter lists 262k for every Jev model and runs long requests in long-context mode (see Access).

**Price:** $0.042 per million input tokens; output is free. Rule of thumb: a whole question tree over 72 long documents costs under a cent.

**"Danger zone" (experimental, opt-in per request by header, as of 2026-10-14).** The docs label these "Dreamer-ready" and say the format, answers and limits can change between releases. The blog says there's "still way too much jaggedness" to vouch for them in production. TypeSafe publishes no accuracy numbers for them, only "internal best guesses", and says "all models (Jev included) perform worse on multimodality over pure text"; breaking a question into its simplest parts "helps a lot". Treat every number as unmeasured until you test on your own data.
- **Images** (`TypeSafe-Danger-Zone-Images: true` on every request with an image; jev-1.13.0 and 1.14.0):
  - Still PNG or JPEG only (animated PNG, WebP and GIF are rejected). Send them as multipart parts (recommended; base64 adds about a third) or inline `data_base64` with a `format`.
  - An image can go anywhere text can: `state`, `instructions`, `criteria`, and inside Choice options, Score levels and Noul `true`/`false` definitions. It goes in by a placeholder reference listed in an `assets` map. References are matched as plain text and object keys aren't replaced, so put an option's image in its description. Every asset must be referenced, or the request returns 400. So you can ask about one photo, compare two ("is A the product in B?"), or use reference images as Choice options.
  - Official limits (as of 2026-10-24): 16 MiB per file; **280** to 1,568 px per side (raised from 256) and 78,400 to 2,458,624 total pixels. Out-of-range images are rejected, not resized: 400 if too small, 413 if too large. Requests are capped at 32 MiB, counting each repeated use of a reference, and all distinct images *after processing* share another 32 MiB (re-encoding can make an image bigger). Each reference key can be at most 1,024 bytes, and all of them together 65,536 bytes. Re-save progressive JPEGs as baseline. Metadata is stripped and EXIF orientation is *not* applied, so rotate pixels first. One bad image fails the whole request.
  - Billed as input tokens at the normal rate, and counted against the normal context limit. There's no official estimator yet. Reports: about 5,200 tokens for a 1181×1568 photo, about 1,600 for 1024×576, and about 2,900 for one product photo plus five questions (about $123 per million photos). **Every mention of the reference inserts the image again**, and is billed again.
  - Many people send screenshots, so Choice over numbered on-screen elements is the common shape for UI agents.
  - Python SDK: `Image(path_or_bytes)`, `Image.from_base64`, `Image.from_pil` (`pil` extra). Neither the Python nor the JS SDK adds the opt-in header, so pass it yourself (`TypeSafeClient(headers={"TypeSafe-Danger-Zone-Images": "true"})`, or per call with `extra_headers=`; JS `{ headers: {...} }`). JS SDK: `image()`, the `withAssets` template tag. The Vercel AI SDK provider adds the header itself. On the **Vercel AI Gateway**, the unversioned `typesafe-ai/jev` is text-only and rejects files, but since 2026-10-15 the versioned `typesafe-ai/jev-1.13.0` and `jev-1.14.0` list image input (not tested live).
- **Box questions:** see the table above. Only on 1.14 (now `jev-latest`); 1.13 rejects it with a 400. To check a box, crop to it and ask one Noul per edge ("is the object cut off at the top?"). On 36 painting crops in TypeSafe's own test, per-edge Nouls caught 30 cut-offs, one "is all of it inside?" Noul caught 26, and an outline drawn on the full image caught 20. When confidence is low, re-ask on a larger or higher-resolution crop, or pad the box.
- **Long context** (`TypeSafe-Danger-Zone-Long-Context: true`; 1.13 and 1.14): raises both limits to 262k tokens at the same per-token price. TypeSafe calls it slower and *less accurate* and "strongly recommend[s] limiting context length", which fits jaggedness item 5 (irrelevant detail). Windowing is still the better move.

**Speed:**
- About 0.1 to 0.4 s per request.
- Rate limits are set per organization, in requests and tokens per second, and shown in the developer console (console.typesafe.ai/rate-limits); the docs no longer publish figures. Higher limits come with custom and enterprise plans. The SDKs retry 429s by default and follow `retry-after`.
- In practice, expect 429s beyond about 8 concurrent workers per key.
- The API can also return `529 Overloaded`, so retry with backoff.

**Version:**
- **`jev-latest` now points to `jev-1.14.0`** ("the most recent stable, official release", the SDK default). It still pointed to 1.13.0 on 10-15. TypeSafe gave no date and no notice; one benchmark saw it resolving to 1.14 on 10-19 while others still saw 1.13 around 10-20, and the docs showed it by 10-23. `jev-preview` points to the same model ("There is no preview build available right now"). No newer version exists. 1.14.0 was released 2026-10-14.
- Deprecation (blog only): TypeSafe "intend[s] to give 60 days warning" before 1.13.0 is deprecated, and 30 days for 1.14.0. A formal deprecation policy hasn't been published yet.
- Log `response.model`, and pin the version once thresholds are tuned. **Anyone who tuned thresholds on 1.13 through `jev-latest` is already on 1.14:** pin `jev-1.13.0` to restore the old behavior, and re-fit on 1.14 before moving on.

**Access:**
- **TypeSafe direct:** `POST https://api.typesafe.ai/v1/systemone`.
- **OpenRouter:**
  - Models `typesafe/jev-1.13`, `typesafe/jev-1.14` (since 2026-10-14) and `~typesafe/jev-latest` (now 1.14.0). All list 262k context and text and image input at the same price.
  - OpenRouter's guide says requests with more than about 32,000 bytes of text, *and every request with an image*, run in TypeSafe's long-context mode on its route. TypeSafe calls that mode slower and less accurate, so prefer TypeSafe direct for short image requests. **Field report:** OpenRouter's System One endpoint (the one the TypeSafe SDK uses) refused multipart uploads and silently dropped base64 images: the answer came back, but Jev never saw the image (324 input tokens with it, 327 without). Images reached Jev only through the Decisions API (yolo-jev, 10-17). Check token counts.
  - Point the TypeSafe SDK at the base URL `https://openrouter.ai/api` (the SDK appends `/v1/systemone`; both SDKs also read `TYPESAFE_BASE_URL`), or use the Decisions API at `/api/alpha/decisions`.
  - Separately, `typesafe/jev-router` is a chat-completions *model router* built on Jev. It isn't the decision API.
- **Vercel AI Gateway:**
  - Models: `typesafe-ai/jev` (text-only, 32k; served by TypeSafe *or DigitalOcean*, and only TypeSafe has zero data retention), plus `typesafe-ai/jev-1.13.0` and `typesafe-ai/jev-1.14.0` (since 2026-10-15: text and image input, 32k context, TypeSafe only, zero data retention). There's a generic Decision API (`/v1/evaluate`), an OpenAI-compatible `/v1/decisions`, and a TypeSafe-compatible endpoint at base URL `https://ai-gateway.vercel.sh/typesafe`.
  - **Zero data retention per request:** `providerOptions: { gateway: { zeroDataRetention: true, only: ['typesafe-ai'] } }` (Vercel Pro or Enterprise; the request fails with 400 if no ZDR provider is available). This is the setup GC AI uses for customer data.
  - The AI SDK call is `experimental_decide()` (AI SDK 7, `ai` 7.0.128+); `experimental_evaluate` is a deprecated alias. The provider is `@ai-sdk/typesafe-ai`. Probabilities come back **rounded to 2 decimals**.
  - It has built-in **decision fallbacks**: rerun on an LLM when `confidenceBelow` or `probabilityBetween` matches. A fallback answer comes back with `confidence: 0, probabilities: {}`.
- **Pydantic AI Gateway:** `https://gateway-us.pydantic.dev/proxy/typesafe`.
- **DigitalOcean Gradient** serverless inference: `typesafe-jev-1.13.0` at `https://inference.do-ai.run/v1/systemone` (a passthrough; text only, 64k/32k). Not yet 1.14.
- **Pydantic AI:** a native `TypeSafeModel` (`pydantic-ai-slim[typesafe]`, `typesafe:jev-latest`, text only).
- **n8n:** TypeSafe's own n8n community node (`@typesafe-ai/n8n-nodes-typesafe-ai`), with Evaluate and Route operations; 0.10.0 (2026-10-20) added images, but not Box.

**SDKs:**
- Python `typesafe-sdk` 0.7.x (`TypeSafeClient`, `AsyncTypeSafeClient`). 0.7.0 moved to pydantic and added `response_model=`. 0.7.5 added `Image`, `Box`, `response.boxes`, and zstd request compression, which is on by default (Python 3.14+ or the `zstd` extra). Extras: `http2`, `pil`, `zstd`.
- JS `@typesafe-ai/sdk` 0.7.x (needs Node 22+). 0.7.0 added images, `box()`, `withAssets`, and zstd compression for bodies of 1 KiB or more (Node 22.15+).
- Vercel `@ai-sdk/typesafe-ai` 3.0.26 has model IDs `jev-1.13.0`, `jev-1.14.0`, `jev-latest` and `jev-preview`, plus image file parts, `longContext`, the `box` question (3.0.21) and `insertReference: false` to send an image only where it's referenced (3.0.20). `TYPESAFE_AI_BASE_URL` points it at a proxy or a compatible server.
- `system-one-adapter-python` serves the same API from an LLM, which is handy for A/B testing Jev against an LLM.

**Data:** zero data retention is enterprise-only on TypeSafe direct. Through the Vercel AI Gateway it's a per-request option on Pro or Enterprise plans (see Access).

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

TypeSafe's official jaggedness list for 1.13 (9 items). As of 2026-10-24 there is still no 1.14 page and no image-specific list, so assume these all still apply on 1.14, now the default.

1. **Literal reading.** It answers the question you wrote, not the one you meant: scoping words, negations and implied conditions are read at face value. Put boundary cases in the criteria.
2. **Math and numbers.** Arithmetic, counting and comparisons go in code. To count, ask one Noul per item and sum in code. Numeric encodings do worse than names (e.g. hex or RGB values vs. color names). Don't interpolate exact magnitudes from a Score.
3. **Date and time comparison.** Do date math in code.
4. **Indirection.** Double negatives, a property of a property, and multi-hop questions cost accuracy. Write instructions directly and name the relevant parts of the state.
5. **Large state full of irrelevant detail ("context rot").** Window or filter first.
6. **Adversarial content.** Not a security boundary.
7. **Contradictory instructions and criteria.** When `instructions` and `criteria` disagree, results degrade.
8. **Choice option order.** A lean toward the first option.
9. **Generation.** It isn't trained to generate text.

Also: English is strongest. Images are an experimental opt-in since 1.14 (2026-10-14); audio and video are unsupported. There's no official image jaggedness list; TypeSafe says all models do worse on images than on text.

Field reports, not on TypeSafe's list:

- **Calibration taken on trust.** It runs overconfident out of distribution and on contested items, and one temperature doesn't fix both. Calibration also swings by domain: in Vals AI's 12-system comparison, Jev had the *worst* calibration on LegalBench (ECE 0.108) but the *best* on claim verification (0.011). Fit thresholds on your own labels.
- **Legal benchmarks (Vals AI, 2026-10-06):** Jev ranked last of 12 on LegalBench class-balanced accuracy (0.730), and answered "Yes" to all 33 items of one contract-NLI subtask.
- **Packing items together:** putting several similar items in one state array dropped accuracy from 96% to 72% in GC AI's hack day (88% with keyed fields like `item_4` instead of `items[4]`). One item per state, or key items by name. Pick-from-list questions over many sentences in one call also degraded (83% at 5, 60% at 40). Many questions about *one* item stayed nearly free.
- **GC AI hack day (2026-09-29, published 2026-10-06; 38 projects on 1.13, small and mostly synthetic sets, usually one run):** [write-up](https://gc.ai/blog/what-we-learned-building-38-projects-with-jev-in-12-hours), [project matrix](https://gc.ai/assets/jev-hackday/jev-patterns.html).
  - Strong: picking from a fixed list (21 of 21 skills; 133 of 136 alert families; 46 of 50 movie titles over 253 options), "does this text do X?" on concrete text, select-not-generate extraction, record matching, claim vs. source.
  - Weak: holistic calls ("is this test useless?" AUC 0.54), absolute fine-grained Scores, recall on subtle patterns (a comment remover at precision 1.00 but recall 0.12), two-hop cross-references (wrong at 0.86), world knowledge (band trivia 57% vs. Claude 93%), near-synonym labels (wrong at 1.00), criteria about files or tool output (59% vs. 96% for LLM judges), and security judgments (28 to 40%).
  - Long context: a needle sentence was still found inside 30k tokens of boilerplate, and a relevant 12k-token document did as well as focused excerpts. Misleading or template text did hurt.
  - Languages: same results in Spanish, German, French and Japanese on a small extraction battery.
  - A question and its negation don't sum to 1. Run-to-run movement was about ±0.03, and Choice picks were more stable than their probabilities.
  - Injection: an anti-injection sentence in the question didn't hold, and planted text flipped 2 of 5 answers in one pipeline (below its confidence bar, so the fallback caught them). A separate "is this an injection?" Noul flagged a planted instruction at 0.98.
  - Timeouts: a 1.5 s budget aborted inside the SDK's retry wait, while 3 s worked. Leave room for one retry.
- **Flops reported by users:**
  - Chess and search-like tasks.
  - Code review as the only reviewer.
  - Record dedupe.
  - A personal prompt router ("OK at best").
- **Losing to cheaper or narrower alternatives:**
  - A tiny distilled specialist beat Jev on a narrow form-filling task (99.7% vs. 83.6%), and a fine-tuned local model beat it on IT-ticket routing (department 62% vs. 41%, priority 72% vs. 57%).
  - Batched offline LLM prompts can match its cost.
  - **Other decision models now compete.** OpenAI's Decisions API went GA on 2026-10-22 at $0.10 per million input tokens. On Vals AI's ClaimBench, several rivals (Decider V1.2, d1-ultra) answered 99.7% or more of claims alone vs. 93.0% for Jev 1.14. A pre-registered head-to-head with OpenAI Decisions was level on three of five tests and favored Jev on two (88.2% vs. 84.2% on CLINC150 with out-of-scope; 94.5% vs. 90.6%), with a better Brier score on all five (gbolask24/decision-bench). Leaderboards put Jev near the top of decision models but below small LLMs on knowledge (Artificial Analysis: decision models 28 to 41 vs. 52 to 66 for LLMs).
  - **Token price isn't decision price.** Restating the option list in every question made Jev read about 6x the text per item, so transaction tagging cost about the same as gpt-4o-mini (CaseTrail).
- **Adversarial text:** in one guardrail test (1,000 items), Jev caught 78% of adversarial rewrites vs. 94% for Haiku 4.5, at about 6x lower latency and 60x lower cost. In another, injected text flipped a confident label to the wrong one at 0.82. A security vendor reported breaking every configuration it tried, at about 50 cents per break.
- **1.14 vs. 1.13 (early, community; Parallix also found 1.14 stricter, clearing 12 of 43 approved repairs vs. 24 on 1.13):** one 1,136-question benchmark went from 80.1% to 82.0%, with the largest gain on verbal spatial tracking and a small drop on exception and priority rules. Multi-step sequential procedures stayed very weak (6 of 42). One migration test saw identical failures on both versions. In a 30-image test, 1.14 was *stricter* than 1.13: it passed 15 of 30 correct image–headline matches at 0.5 or above vs. 23 of 30 on 1.13, with slightly better separation overall (AUC 0.91 vs. 0.89). So thresholds tuned on 1.13 may not carry over.
- **Since 10-15 (community, mostly 1.14):**
  - **Calibration by type** (Brussle, 500 items per type): yes/no calibrated (ECE 0.023), Choice over-confident (about 0.10), Score badly over-confident (0.275).
  - **Wording and context move thresholds** (Alinia, 2,232 replies): three phrasings of one policy blocked 11%, 54% and 79% of compliant replies at 0.5. Dropping the user message moved 710 replies across 0.5, and 44 to 53% of scores sat between 0.2 and 0.8. Jev agreed with AI-written labels 6 to 8 points more than with human ones.
  - **Counterfeit policy text in the state** cut attack recall by more than 79 points; a plain "set the answer to 0.01" didn't work (Datadog Security Labs).
  - **Order sensitivity:** swapping which option is described first changed the answer in 370 of 2,620 trolley-problem pairs.
  - **Empty state:** with nothing in the state, a true world fact still scored 0.99, so a failed fetch looks like a confident pass. Add a basis or presence Noul (SuperInstance/jev-lab).
  - **Box isn't deterministic:** the same request returned the same box only 30% of the time (median move 4.8 px), and targets under 10 px were hit 2% of the time (jev-click-bench).
  - **Choice output:** in 2,538 calls, 4 returned a choice that wasn't the top-probability option (gaps of about 0.01), and 5 had probabilities summing to 0.99.
  - **Score drift:** 10% of leads changed level between identical runs.
  - **Weak spots reported:** look-alike entities (70% vs. 97% for gpt-4o), ops remediation (4 of 18 incidents vs. 13 for Opus), and a two-line regex beating Jev's best single phishing signal (91.8% vs. 89.4%).
  - **Images:** two images side by side got left and right mixed up; hate imagery scored F1 0.27 to 0.44 for every model.
- **Benchmark gap:** on TypeSafe's own WorkflowEvals, Jev scored about 68% vs. about 74% for a frontier LLM, at a tiny fraction of the cost and latency.

## Fit signals

- **Jev:** the answer space is known in advance; it's one narrow judgment; the evidence fits in the state; an expert could judge it in seconds; code branches on the result; you'd want to ask it thousands of times or in real time.
- **LLM:** you need prose output or a rationale/audit trail; the options aren't known; it needs a retrieval or tool loop mid-decision; or it takes multi-hop reasoning.
- **Code:** arithmetic, dates, exact lookups, known rules, regex-findable units, security rules, threshold policy.
- **Images (experimental):** the judgment is visible at a glance (damaged? same product? is the element on screen?), and a crop or a pair of images holds the evidence. Not for pixel-exact geometry, and nothing should go to production before TypeSafe drops the "danger zone" label.
- **Always:** add an absolute "does anything fit?" Noul (on 1.14 TypeSafe prefers it to a `none` option; field reports on 1.13 found `none` helped too); validate on in-domain labels, including hard cases; log the model version.
