# Jev patterns: what it makes possible

Jev has four levers, plus a fifth that is still experimental:
- **Nearly free:** cents per thousand documents.
- **Fast:** 0.1 to 0.4 s per request.
- **Full distributions:** probabilities, not just a label.
- **Many named questions per state:** and the state can hold several texts.
- **Images (experimental since 2026-10-14):** PNG or JPEG in the state, the question, or the options, plus a **Box** question that returns a region. Most image evidence so far is demos and small studies from launch week.

Every pattern below is some combination of those. "Classify X" is the least interesting use.

## Rules of thumb

These held up repeatedly in practice and agree with TypeSafe's own guidance or with independent reports.

- **Window first.** Send located passages or context windows, not whole documents. Irrelevant text lowers accuracy, and the window is often half the win. The same goes for images: crop to the region, and TypeSafe's own Box guidance says the same (tile first for small targets, re-ask on a tighter crop when unsure). In one study, a nonsense sentence scored 0.90 on a paragraph crop but only 0.60 on the whole page. What hurts is *irrelevant or misleading* text, not length as such: at [GC AI][gcai], a relevant 12k-token document matched focused excerpts, and full-contract context fixed a two-hop cross-reference miss. Alert template text ("the request errored for the user") caused false pages, though, so strip boilerplate. Jev believes what it reads.
- **Code first for anything checkable.** Quotes, counts, dates, arithmetic, pincites and exact lookups belong in code, and that code runs before Jev. Jev will happily call a fabricated quote "supported", and it has nothing to judge when the text is missing. Then put the facts code computed *into the state*: at [GC AI][gcai], adding the observed CPU value took a pager's precision from 0.63 to 0.92, and giving Jev a computed landing point took a Breakout player from 17 to 29 bricks a minute.
- **Ask narrow, factual questions, not holistic ones.** Broad value judgments are close to coin flips: "is this test useless?" scored AUC 0.54, and vague per-passage or "is anything missing?" questions failed the same way (#15, #20). Splitting the judgment into concrete yes/nos mined from real examples, then combining them in code, took the test flagger to 0.72. One overall "is it supported?" missed "may" turned into "must", and three narrow yes/nos (explicit support, mismatched details, overstatement) caught it. [PremiseCheck](https://openlair.github.io/premisecheck/) measured the same gap on 9,267 review comments: Jev judging whole comments scored AUROC 49.9, near chance, and Jev verifying each premise an LLM pulled out of the comment scored 63.1 (more in #36). The exception is a clean single criterion or category. For document status, one Choice with a described option per status beat the same thing split into yes/nos (88% vs. 82%). On a content policy, [Alinia](https://www.alinia.ai/resources) found a two-line summary of the policy did as well as the full text, and splitting it into five factor questions *lost* 6 points. Split when the judgment really is several facts, not when it's one idea stated at length. Hand-write the questions; AI-generated question lists gave no signal.
- **Trust the extremes, not the middle.** Use Jev to clear or escalate, or to filter or screen. The 0.3 to 0.7 band is where the hard cases live; route that band to an LLM or a person rather than ranking on it. TypeSafe's own skill (v0.6.2) now gives the same numbers: 0.7 and 0.3 are "good starting thresholds for a Noul", and the band between means the question isn't settled. It also warns that `confidence` measures how concentrated the distribution is, "not the same as how likely the answer is to be correct", so it's rarely the right gate.
- **Fit thresholds on your own labels, out of sample, and refit when the data shifts.** In-sample thresholds mislead, thresholds don't carry over between datasets, and a gate tuned at one prevalence can miss badly at another. Pin the model version once thresholds are tuned. For a gate, read the probability of the level you care about, not the Score's expectation. Across GC AI's projects, tuned cutoffs ran anywhere from 0.25 to 0.95, so don't assume 0.8; per-pattern thresholds beat one global one; and asymmetric bands help when one error costs more (#3). Refit when the model changes: 1.14 is stricter than 1.13 in two separate reports (on the same 43 approved code repairs, 1.13 cleared 24 and 1.14 cleared 12; [Parallix](https://oxtan-maker.github.io/parallix/articles/jev-review-evidence/)), and `jev-latest` moved to 1.14 without notice.
- **Request shape changes answers.** Answers vary a little from run to run, and putting several items in one state or changing the question set can move them more. At [GC AI][gcai], primes were 72% right packed into an array, 88% as keyed fields, and 96% one per request; SMS spam went from 62% batched to 100%; `citations[4]` gave a wrong answer where `cite_4` was right; and pick-from-list questions over sentences fell from 83% at 5 per call to 60% at 40. Others found the same: asking about `records[17]` in an unlabeled list fell from 94% to 34% at 100 items ([rlm-jev](https://github.com/jhochenbaum/rlm-jev)); one question asked over every line of an invoice scored every line 0.44 to 0.46 until the question named the line ([Unsiloed](https://www.unsiloed.ai/blog/build-automatic-ai-accountant-muse-unsiloed-jev)); and 32 replies per call shifted typical scores by 0.08 and some by 0.4 (Alinia). Wording and context move the threshold too: three phrasings of the same policy blocked 11%, 54% and 79% of compliant replies at 0.5. Many *questions* about one item are nearly free (1 to 150 questions took 0.33 to 0.76 s). TypeSafe's skill agrees: it's better to put each item into its own question "than to give an ID the model has to look up in a shared list". So fan out one item per request, or key items by name if they must share a call; version (or hash) your question sets; and repeat or cache any answer that sits near a threshold.
- **Read Noul probabilities as odds; don't read Choice or Score probabilities that way.** On 500 items per type, [Brussle](https://brussle.com/evals/calibration/) found 1.14's yes/no answers calibrated (ECE 0.023), Choice over-confident (ECE about 0.10, with about 1 in 9 answers wrong at a mean confidence of 0.98), and Score badly over-confident (ECE 0.275: answers given at about 0.8 were right less than half the time). That agrees with earlier tests and other reports on Choice. Noul calibration still degrades on hard items (ECE 0.22 on the hardest 20%), and one earlier test found Nouls *under*-confident, so fit cutoffs on labels anyway. For ranking and routing, the probabilities are still useful: in one GC AI test every wrong answer from gpt-5-mini came at 0.85 confidence or higher, so its self-reported confidence couldn't route, while Jev's could.
- **Make options mutually exclusive, and spell out the boundaries.** A Choice assumes exactly one answer. If "both" is possible, ask separate yes/no Nouls instead. Jev reads literally, so put the edge cases in the criteria. Near-synonym labels break it (Severability vs. Enforceability, confidently wrong at 1.00), and spelling out the rule took one CUAD task from 65% to 90%. Give every option a description and a name that describes it: undescribed options caused a skill router's misses, and TypeSafe warns that positional names (`A`, `option_1`) can bias the answer. Guidance written for people needs rewriting as literal conditions (a playbook's checks scored 41 of 80 as written). A question and its negation don't sum to 1, so ask each decision one way.

## Worth testing

Hunches with thin or mixed evidence. Suggest them as a **Cheapest test**; don't state them as fact.

- **Neutral questions may beat questions that include your situation.** Asking the general question ("does this case hold X?") separated far better than asking whether it fit the user's facts.
- **Relative judgments may beat absolute ones.** "Which passage decides X?" (a Choice) worked, while "is this passage substantive?" (a Noul per passage) was at chance. That's confounded, though: the failing question was also vague. GC AI saw the same lean: absolute Scores bunched in the middle while relative ranking worked, and a sentence-level one-sidedness Score agreed with labels only 47 to 63% of the time and ran half a level harsh. Elsewhere, Score levels changed between runs for 10% of B2B leads (vs. 0% for OpenAI Decisions), and absolute plausibility Scores missed obviously wrong quiz answers (0.53). Against that, Jev landed on the exact lead level 71% of the time vs. 46% for OpenAI Decisions ([evaboot-labs](https://github.com/evaboot-labs/openai-decisions-vs-jev)).
- **Loaded words may tilt the answer.** Naming the thing you're hunting for in the question ("is this AI-generated?", "any hallucinations?") pushed real photos toward yes (7 of 8 flagged), until a neutral confirm question was added. A text report points the same way: asking about "limited liability" flagged ordinary refund clauses, and narrowing the wording to damages, losses or harm halved the false alarms. Test a neutrally worded version too.
- **On images, Jev may read meaning, not exact layout.** It "mentally fixed" misaligned tables (0.89 to 0.96 on misaligned vs. 0.92 to 0.98 on clean, with no separation), and it couldn't tell look-alike stock photos apart. In TypeSafe's own test, judging a box from its outline on the full image caught the fewest cut-offs (20 of 36, vs. 30 for one Noul per edge on a crop). Do exact geometry (alignment, measurement, counting pixels) in code or classical vision.
- **Questions can be one dimension in disguise.** Several questions that looked independent all scored alike, and weighting them never beat a simple count. Check the correlations before building a weighted composite.

## Patterns

Status tags:
- **TESTED:** someone has run it on real data.
- **DOCUMENTED:** TypeSafe shows it working.
- **COMMUNITY:** seen in public work, with a link.
- **UNTRIED:** an idea.

When the user's own history has a matching experiment, cite that.

## A. Getting a sense of things

**1. Quick read / profile.** Ask 10 to 20 questions of one document at once and get its profile: what kind of document it is, its stance, its outcome, whether it reaches the issue, and how deep it goes. A "what is this?" read for a fraction of a cent, before you decide whether to read it.
TESTED (content previews, citing-case profiles). Breaks: questions that need inference across the whole document.

**2. Fan-out first, decide later.** Ask every question you *might* need in one request and let code decide which answers matter. Speculation is free, so the design question becomes "what would I want to know?", not "what can I afford to ask?".
DOCUMENTED (about 10x cheaper than sequential). TESTED. COMMUNITY: going from 1 to 150 questions on one item only took a request from 0.33 s to 0.76 s ([GC AI][gcai]), and 1 to 8 questions stayed at about 260 ms (Alinia).

## B. Cutting a big set down

**3. Funnel before the LLM.** Jev clears the obvious yeses and noes; the LLM only sees the uncertain middle. The gains come from the extremes. Set two thresholds (clear and flag) and send the band between them to the LLM.
TESTED (citation checking: about a quarter cleared and a tenth flagged with no bad calls; the rest went to the LLM). COMMUNITY:
- Conformal thresholds (jev-certify) auto-routed about 85% of queries with about 2% loss.
- Entity resolution with an LLM on the uncertain residue matched 199 of 200 at a tiny fraction of frontier-LLM cost ([Southbridge](https://southbridge.ai/blog/jev-entity-resolution)).
- Incident diagnosis ([SREGym](https://sregym.com)): 80 of 105 passed (76% vs. 78% for an LLM agent), about 7x faster, $0.15 in total.
- [GC AI][gcai]'s hack day ran it six times, and every head-to-head with a stronger-model fallback matched the stronger model at a fraction of the cost:
  - Relationship detection: Jev alone 84.6%, Gemini Flash 91.9%, Jev with bands also 91.9%, with Jev handling 86% of pairs and 43% lower cost. False yeses were the weakness, so the bands were asymmetric (no below 0.40, yes above 0.70).
  - Party matching: Jev 90%, the existing LLM 93%, Jev plus fallback 95%, at $0.019 vs. $0.51 per 1,000.
  - LegalBench, CUAD and LEDGAR: 79% overall but 92% at confidence 0.8 or higher, which covered 72% of items.
- Screen variant, for rare positives: Jev drops most traffic with a *low* bar and an LLM confirms and extracts the rest. A preference-memory feature sent only 1 in 13 messages to Sonnet, at 11.5x lower cost (92% precision, 71% recall). Jev alone found only 1 of 11 real preferences, so it's a screen, not the judge.
- Noisy-alarm variant: Jev clears a noisy CI security check (only 8 of 30 sampled alarms were real) by passing a change only if every file scores under 0.25, and sends the rest to a reviewer.
- Claim checking ([Vals AI ClaimBench](https://www.vals.ai/benchmarks/claimbench), 1,960 claims against 10-K excerpts): in front of Claude Haiku 5.5, Jev 1.14 answered 93.0% of claims alone with 0.39% wrong (1.13: 88.0%), for 99.64% overall at $0.34 per 1,000 vs. $1.19 for Haiku alone. Some competing decision models answered 99.7% or more alone, for less.
- Forensic accounting ([CaseTrail](https://case-trail.com/), synthetic data): sending unsure entity matches to gpt-4o matched its 98.9% at 59% of the cost, with 55% escalated. At 90% confidence or more, Jev was right 99.4 to 100% of the time.
- *Delta re-review* ([Parallix](https://oxtan-maker.github.io/parallix/articles/jev-review-evidence/)): after a coding agent repairs its own approved change, Jev judges only the repair against the approved revision and the failed check (no skipped test, no loosened assertion), with three parallel questions; any doubt goes to the LLM reviewer. It cleared 41 of 114 re-reviews with one confirmed miss, at 1.7 s and $0.0016 vs. 34.6 s and $0.26 for Sonnet. Whole PRs blew the 32k budget and gave useless answers.

Breaks: using one threshold; trusting in-sample thresholds.

**4. Census instead of sample.** When the universe can be listed (every court opinion matching a broad query, every ticket this quarter), score *all* of it on one coarse question and audit a random sample to measure the miss rate. That turns "we searched carefully" into "we checked every document; measured miss rate is X%".
UNTRIED as a census with a miss-rate audit. Whole-corpus screens are TESTED and COMMUNITY: about 1,200 pages judged in under 3 minutes; 1,500 posts per channel profiled month by month; document type and status over a 25k-document corpus at about 119 documents a second and $0.0001 each (type 87% vs. Opus 93%, right type in the top 2 99%; [GC AI][gcai]); 2,140 accounts screened for $0.57 at 94.5% vs. 94.0% for human screeners ([jev-account-filter](https://github.com/caiweihan/jev-account-filter)). Breaks: the universe is only as complete as the query that listed it.

**5. Treatment screen.** Run one question over every citing or referencing document's context window: does it follow, distinguish, limit, or overrule (or, outside law, endorse, dispute, or supersede)?
TESTED (high recall on "no longer good law"). Breaks: treatment you have to infer.

## C. Taxonomies and structure

**6. Question tree.** Encode a decision tree as many questions in one call; code walks it. The tree *is* a schema for the facts.
TESTED. Breaks: attributes that depend on how the excerpt is cut.

**7. Big taxonomy, beam search.** Past 255 options, or for a deep hierarchy, descend level by level keeping the top K branches.
DOCUMENTED (K=3 beat greedy). Also works for *retrieval*. COMMUNITY:
- [jevgrep](https://github.com/dzhng/jevgrep) walks a repo folder, then file, then declaration (8 of 10 SWE-bench tasks, cost down about 26%).
- [jev-doc-search](https://github.com/VectifyAI/jev-doc-search) descends a document's table of contents to get past the option and token limits.

Images: the same descent works over a grid of crops ("which cell holds X?"), then a finer grid inside it. An iOS-simulator driver located elements by grid descent at about $0.005 a run vs. $0.003 for Box ([axe-decision-control](https://github.com/Saik0s/axe-decision-control), demo).

Breaks: multiplying scores down the path can bury the right leaf (a category scored 0.94 at the leaf but 0.091 along its path). Shortlisting with embeddings and one Choice may beat descent (#14).

Label candidates include subject headings, legal topic outlines, product catalogs, support-ticket routing, and content tagging.

**8. Taxonomy debugger.** Run a draft taxonomy over a few hundred items and look at the *distributions*:
- Two categories that keep splitting the probability overlap.
- A category that never wins is dead.
- A high `none` rate shows a gap.

Fix the taxonomy, not the labels.
UNTRIED.

**9. Metadata layer and changed-fact re-sort.** Store per-item attributes along with their distributions. Then change one fact in a hypothetical and re-rank the collection by how well each item matches. That becomes a query instead of a re-read.
UNTRIED as a re-sort. COMMUNITY: weights fit over 12 to 14 Scores beat a single direct question on NLI, but false positives on hard benign cases rose about 25x, so validate on hard negatives.

## D. Using the probabilities themselves

**10. Uncertainty spotlight.** Spend human reading where Jev is *least* sure. The 0.3 to 0.7 band is where the interesting, contested, or badly worded items live.
TESTED implicitly (the partial-support cases clustered in the middle). COMMUNITY variant: ask the user a clarifying question when the top two options are close. COMMUNITY, as a review queue: an agent-plan verifier kept Jev's answers at 0.8 or above and sent the rest to a person, matching gpt-5-mini's 92% ([GC AI][gcai]). COMMUNITY, image moderation ([image-moderation-bench](https://github.com/Luminary-AI-Solutions/image-moderation-bench), 10,146 UnsafeBench images): Jev 1.14's AUROC was 0.866 vs. 0.859 for Haiku 5.5 and 0.698 for Rekognition. At a 5% false-flag budget, it caught 50.7% of unsafe images vs. 31.4% for Haiku, at 7.7¢ per 1,000 and 162 ms. COMMUNITY, on images ([jev-vision-benchmarks](https://github.com/MaceTenth/jev-vision-benchmarks)): five yes/no policy questions per catalog photo; sending the 0.3 to 0.7 band to a person put 4 of 294 answers (1.4%) in review, including both errors, and the other 290 were all right. It caught 7 of 8 problems with 1 false alarm, at about 0.4 s and $123 per million photos (60 photos, Claude-written labels, 8 positives). Warning: Choice probabilities are poorly calibrated on contested items, so rank by closeness or spread, not by the raw value.

**11. Disagreement mining.** Ask a broad and a narrow version of a question, or compare Jev with an LLM or a human. The items where they diverge are the ones to read.
TESTED (the broad-vs-narrow gap kept every must-have document; some "false alarms" were Jev right and the LLM wrong).

**12. Question-wording search.** Because asking is free, *search over phrasings* against a labeled set and keep the wording that separates best. It's prompt engineering with a measurable objective.
TESTED. COMMUNITY: adding few-shot examples to the criteria cut a comment remover's false deletions from 6 to 0, and calling them "sentences" instead of "units" was worth 5 points ([GC AI][gcai]). *Phrasing ensemble:* asking 16 phrasings of one verification question in a single call, half of them as "does the paper contradict this?", raised AUROC from 63.1 to 65.7 ([PremiseCheck](https://openlair.github.io/premisecheck/)). *On images:* two reference images per class, plus a defect description tuned by blind agents and a calibration step, beat CLIP trained on 50 per class on two of four industrial-inspection sets (0.94 vs. 0.57 balanced accuracy on one; roughly even on the other two; [decision-models-for-industrial-image-qc](https://github.com/fmegahed/decision-models-for-industrial-image-qc)). *Check a composite before trusting it:* a backlog ranker only trusted its rubric when Kendall's tau against an independent ranking was 0.4 or more; 15-issue checks were too noisy (0.14 and 0.51) ([feniix](https://feniix-hq.net/blog/ranking-a-backlog-with-jev/)). Breaks: overfitting to the held-out set; bump the rubric version.

**13. Gold-set hygiene.** Run Jev over your gold labels. Where it confidently disagrees, re-check the label first.
UNTRIED.

## E. Making it point

**14. Locator.** One Choice over numbered passages ("which passage decides X?") finds where something is. Jev can't search by itself, but it can point at an option you number.
TESTED (the right passage was usually in the top 3). DOCUMENTED (line-ID search). COMMUNITY: routing callers' questions to the right website page ([Sayora](https://sayora.ai)) got the right page first 88% of the time vs. 70% for embeddings, and correctly said "none of these" 95% of the time vs. 23%, for $0.45 over 1,449 requests. That's the case for always covering "nothing fits" (on 1.13). On 1.14, TypeSafe says a companion "does anything fit?" Noul usually works better than a `none` option. COMMUNITY: embeddings shortlist 30 categories (recall above 99%), then one Choice picks; product categorization went from 81.9% to 97.2% at about $0.0005 per product ([Channel3](https://trychannel3.com/blog/where-jev-fits)). Inside a recursive-LLM harness, Jev located 30 facts in 1.2M tokens with recall@5 of 100%, at about $0.004 per query ([rlm-jev](https://github.com/jhochenbaum/rlm-jev)).

**15. Paragraph heat map.** Ask the same Noul of every paragraph to get a probability curve through the document, showing where an issue is decided and where it's only mentioned. Use it for windowing, for reading UIs, and to jump straight to the key passage.
COMMUNITY: jev-skip paints a sponsor-segment probability on the YouTube seek bar (caught about 77% of sponsor seconds, under $0.001 a video). Per-line "grep by meaning" tools do the same over code. For images, fan the same Noul out over tiles or crops to get a heat map of the picture (see #32). COMMUNITY, contracts ([GC AI][gcai]): a risk simulator scored every sentence with the full contract and the reader's side as context; 8 of its top 10 were planted risks, including a buried cross-reference that swallowed the liability cap (91 sentences, about 1 s, $0.018). A one-sidedness heat map that used absolute 5-level Scores did worse (47 to 63% level agreement). Warning: a vague per-passage question ("is this substantive?") was at chance. Prefer a relative judgment (a Choice over passages) or very concrete Nouls.

**16. Remove-a-sentence attribution.** Delete one sentence at a time, re-ask, and watch the probability move. The sentences that move it most are what drove the judgment. Cheap explainability for a model that can't explain itself.
UNTRIED.

**17. Pre-parsed extraction.** Code finds candidate values (dates, ID numbers, names, statutes cited); Jev picks which one is *the* answer; code copies it verbatim. It can't hallucinate a value.
DOCUMENTED. COMMUNITY ([GC AI][gcai]): 9 of 9 fields in one 0.34 s call, and 14 of 14 extractions on a 12k-token terms of service; an inbox triager had code find date phrases and Jev pick the deadline. Always cover "nothing fits": add a separate "is it present?" Noul (TypeSafe's advice for 1.14) and, at least on 1.13, a `none` option as well: with no escape option Jev picked a wrong date at 0.92, and adding `none_of_these` made it right at 1.00. Merge duplicate candidates first. COMMUNITY variant: for spans, code nominates candidates, Jev verifies each one, and code fixes the boundaries (named-entity recognition at about 74 strict F1 averaged over 12 benchmarks).

## F. Two texts at once (state as an array)

**18. Pairwise judgments.** "Same issue?" "Does B apply A's rule?" "Does this passage support this claim?" "Did this edit change the meaning?" These power deduplication, clustering, treatment checks, version diffs, and matching a question to candidate sources.
TESTED (claim plus passage). DOCUMENTED (reranking a BM25 shortlist on legal queries: top-10 from 38% to 62%). Generic record dedupe has been reported as underwhelming, but funnelled entity resolution worked well (see #3), and so did narrow matching questions: a returning-buyer checker asked two Nouls (same person? same organization?) with clerical equivalences in the criteria and agreed with Opus 5 95.5% of the time at $0.00011 vs. $0.0073 per probe ([GC AI][gcai]). Claim vs. source at GC AI: a citation verifier (supports / contradicts / silent) hid 0 of 151 good held-out citations and 57 of 84 bad ones, at a p90 of 366 ms per message. Repetition detection: comparing the current prompt with 20 earlier chats ("same kind of task? ignore parties, amounts, dates"), with code counting the matches, hit 0.92 precision and 0.92 recall.
COMMUNITY, reranking (Nouls per candidate): roughly level with commercial rerankers, not ahead of them. [Kapa](https://www.kapa.ai/blog/jev-reranker-benchmark) found Jev 1.14 at 0.532 vs. 0.535 for Voyage rerank-3, which was cheaper and faster, and ahead of Cohere v4.0-pro (0.506); Noul beat Score (0.532 vs. 0.525). [Parallel](https://parallel.ai/blog/are-all-system-one-models-created-equal) found a fine-tuned reranker 3.8 points ahead. SciFact was level with the best decision model and ahead of BGE ([evidence-lab](https://github.com/ozalt/evidence-lab)), and Chinese reranking scored nDCG@5 0.546 vs. 0.413 for BM25 ([jev-rag-zh-benchmark](https://github.com/vicfei/jev-rag-zh-benchmark)). Images: front vs. back of the same ID card rejected 112 of 112 cross-country pairs and accepted 55 of 56 genuine ones, but called about 60% of same-country, different-version pairs a match (AUC 0.939; [jev-front-back-id-matching](https://github.com/HRyohni/jev-front-back-id-matching)). With images (experimental), the pair can be two pictures or a picture plus a claim, for example "is this upload the product in the catalog photo?" or "is the product in this returned-item photo damaged, as the claim says?". COMMUNITY (demo): a reference photo plus video frames, with a Noul for "visible" and then a Box ([video-image-tagger](https://github.com/botheaj/video-image-tagger)). COMMUNITY (numbers): "does this featured image fit this headline?" separated own vs. random headlines at AUC 0.91 over 30 images, and rejected all 60 off-topic pairings (Thorsten Meyer, read via a republished copy); look-alike stock photos were the misses.

**19. Tournament ranking.** Many cheap pairwise "which is more on point?" judgments add up to a ranking, which sidesteps poor absolute calibration.
UNTRIED. Watch Choice order bias; ask both orders.

## G. Watching processes

**20. LLM-output check.** A cheap screen on a draft before anything expensive runs: does this sentence's citation support it; is this summary faithful to that passage?
TESTED. COMMUNITY: Jev assertions inside test suites (pytest-jev). COMMUNITY ([GC AI][gcai]):
- *While streaming:* check each cited claim against all its sources as it closes, and underline it before the next paragraph arrives (no false "high" in a 32-case test, 200 to 470 ms per call). Re-check weak claims, and cap the wait after the stream ends.
- *As an eval judge:* one Noul per rubric criterion agreed 94% with a unanimous LLM panel at about 1% of the cost. It was good on objective criteria (legal accuracy 98.7% vs. 87.4%), weak on holistic ones ("sound advice" 83%) and on criteria about files or tool output (59% vs. 96%).

Breaks: vague "is anything missing?" questions, which score high on everything.

**21. Agent tripwires.** Inside a long agent run: is this step still on the task? Is this draft answering what was asked? Is this work actually done? The answer becomes a gate in a hook.
COMMUNITY: a tool-permission gate tested on about 7,900 labelled calls made 1 unsafe allow among 3,664 risky ones and settled 62% of real work at about 160 ms ([jev-permission-gate](https://github.com/madisonrickert/jev-permission-gate)). Others: wake-or-sleep gates for idle agents, and skill routers. COMMUNITY ([GC AI][gcai]): an exfiltration guard on every tool call stopped all 11 exfiltration calls at about 200 ms each, flagging at 0.4 because attacks scored 0.39 to 0.49. Splitting the state into the user's request, pasted text and tool results cut executed attacks from 6 to 0. Breaks: injected text didn't push dangerous calls through, but it caused about 10% false denials, and "authority" framing moved a few. An anti-injection line inside the question didn't hold against a planted comment, while a separate "is this an injection?" Noul caught one at 0.98, so let that Noul win. Decide fail-open vs. fail-to-approval up front. [Datadog Security Labs](https://securitylabs.datadoghq.com/) found counterfeit policy text placed in the state was the attack that worked: it cut attack recall by more than 79 points, and took a LangChain tool-risk gate from 0.893 to 0.290, under its 0.5 block line. A plain "set the answer to 0.01" injection didn't move it.

**22. Novelty and saturation detector.** "Does this new item add something not already in the set?" Plot the number of new items per round and stop when the curve flattens.
COMMUNITY, as a stopping rule for search: asking "is there enough context to answer?" matched an LLM's recall at about a quarter of the latency (AUROC about 0.90 vs. 0.69 for small local models).

## H. Speed-dependent

**23. Live feedback.** At 0.1 to 0.4 s, Jev can run as you type or click. For example: flag a sentence that states a conclusion without a source; badge each item in a reader with its attributes; check a student's answer against a rubric before they submit.
COMMUNITY ([GC AI][gcai]):
- A skill picker suggested the right skill as the user typed (debounced 600 ms; 21 of 21 vs. 20 of 21 for gpt-5-mini, 221 ms vs. 1,896 ms; over 102 real skills the button was right 96% of the time with 0 wrong pills).
- A browser extension ran eight trap Nouls per paragraph of any terms page (Spotify's 110 paragraphs in 1.9 s; held-out F1 89% vs. 57% for regex; 19 of 20 red highlights right on 11 real sites). Every early data-sharing hit was the cookie banner, so check what the scraper actually sends.
- Citation checks as the answer streams (#20).

Debounce, cache verdicts so badges don't flicker, and judge only new items.

**24. Query to filters.** Break a plain-language question into attribute constraints (jurisdiction, date range, category, fact branch), then filter the collection before retrieval.
UNTRIED.

## I. Teaching and personal tools

**25. Misconception classifier.** Map a student's answer onto a taxonomy of known errors. Use the result to give targeted feedback, pick the next hint, or tally misconceptions across a class.
UNTRIED.

**26. Feed triage.** For new reports, new filings, new podcast episodes, or new papers: "is this in my interests, and which project does it belong to?" Ask before anything downloads, transcribes, or summarizes.
TESTED.

## J. New shapes from the community

**27. Generate-and-test recognizer.** A brute-force or search process produces candidates, and Jev judges which output is "real" (plausible, valid, on-target). Jev is a strong *recognizer* even where it's a weak *policy*: let code explore and Jev recognize.
COMMUNITY: [enigma-jev](https://github.com/agodoy21/enigma-jev) judges whether each candidate decryption is real German (159 of 160, and it held up under distribution shift where XGBoost collapsed). COMMUNITY variant, LLM proposes and Jev checks ([GC AI][gcai]): a de-identification loop had an LLM invent replacement names and Jev reject any that echoed the original, retrying up to 3 times. Jev caught 16 of 20 semantic echoes (code 0, Sonnet 19) at 34x Sonnet's speed, but missed world-knowledge echoes such as Nike → "Victoria Sportswear". Code owns the state, the counts and the stop condition. Quiz distractors (Qiita, 40 items): Jev answers with the key hidden to check the item is solvable, scores each distractor, and Claude rewrites the weak ones; mean plausibility rose from 0.42 to 0.68. Breaks: using Jev to *choose the next move* in the search had no skill there, and a compiler-optimization policy lost to the baseline.

**28. Distill-and-defer.** Jev labels live traffic. A small local model trains on those labels and answers when its confidence bound allows; otherwise it defers to Jev. Over time, most traffic never leaves the machine.
COMMUNITY: [Jevstiller](https://jevstiller.pages.dev/posts/the-guarantee/) answered 25 to 90% of requests locally at about 15 ms, with 1 to 1.5% disagreement under a 2% budget, across four intent and topic datasets. A good fit when the task is fixed and traffic is high, because specialists can beat Jev there.

**29. Legal-move picker.** Code enumerates the legal actions (page controls, state-machine transitions, game moves), and one Choice per step picks one. It's fast enough to drive a UI or agent loop in real time.
COMMUNITY: [jev-ultrafast](https://github.com/browser-use/jev-ultrafast) completed a flight search in about 7 s; [jevspresso](https://github.com/statelyai/jevspresso) picks transitions in a state machine. Small numbers so far. With images, the state can be a screenshot with numbered elements, and the Choice picks a number; coordinates come from the UI tree, not from Jev. [jev-phone](https://github.com/dongxinsuperman/jev-phone) took about 0.7 s per step (demo, no accuracy reported). Prefer the accessibility tree when there is one, and fall back to the screenshot. COMMUNITY ([GC AI][gcai]): Jev picking the target and aggression in Breakout, with code doing the physics, cleared 39 bricks a minute vs. 24 for code alone. Picking moves from raw state managed 17, and 29 once code supplied the predicted landing point; slowing the game didn't help. [fastbrowse](https://github.com/agent-labs-dev/fastbrowse): 99.2% vs. 97.8% for a hosted Browser Use agent, at $0.026 vs. $0.28 per result. [jev-browser](https://github.com/zurfyx/jev-browser-skill) also asks a speculative target under every possible operation (#2), so two decisions take one round trip. Breaks: long-horizon planning, which needs a real planner.

**30. Context gardener.** Inside a long agent session, ask one Noul per old tool result: "is this still needed word for word?" Move the dropped ones to a recoverable file instead of deleting them.
COMMUNITY: [jev-compaction-plus](https://github.com/cth9191/jev-compaction-plus) compacted a 591k-token session in under 1 s vs. 35 s for built-in compaction, with equal recall on a 6-question quiz (one session). Breaks: pruning can drop something needed later, so keep it recoverable.

**31. Resolve once, replay free.** Jev maps each plain-English step (a test step, a macro) to a concrete control once. The recording then replays with no model calls, and Jev is called again only when the recording goes stale.
COMMUNITY (demo): [jevwright](https://github.com/Ice-Hazymoon/jevwright).

## K. Images (experimental)

**32. Locate, crop, confirm.** A Box (or a grid Choice) proposes *where*. Code crops that region, upscaled if it's small, and Jev re-asks a narrow, neutrally worded Noul on the crop. Gate on the confirm, not the first pass. Because Box always returns a box (even for an absent target, at low confidence), ask an "is there a ...?" Noul in the same request.
DOCUMENTED: TypeSafe's Box page gives the same recipe: a Choice over tiles before Box for small targets, a presence Noul alongside, and one Noul per edge on the crop to check the box (30 of 36 cut-offs caught vs. 20 from the outline on the full image).
COMMUNITY: [jev-box-study](https://github.com/RINNECODER/jev-box-study) flags errors in AI-generated images.
- Caught 97 to 98% of planted text errors (misspellings, wrong percentages, contradictions).
- Its confirm step cut false flags on real photos from 4 of 26 to 0 of 26.
- Cost about $0.02 per image (170 to 250 calls).

COMMUNITY, after 1.14:
- *Re-locate on the crop:* [jev-click-bench](https://github.com/arthurkatcher/jev-click-bench) used Box as a GUI clicker. On ScreenSpot-Pro (small targets on big screens), one call hit 18.66%, a second Box on a 2x zoom of the first answer 42.76%, and a third at 4x 60.15% (added afterwards), at $0.16 to $0.48 per 1,000. On ScreenSpot-v2 it went from 84.36% to 87.89%, and zoomed answers at confidence 0.9 or more were right 98.7% of the time. Targets under 10 px still failed.
- *Presence Noul, not Box confidence:* a drone camera's "seen" Noul scored 0.54 to 0.98 with the target in frame and 0.13 or less without it, while Box confidence was *higher* on an empty frame (0.46) than on the real one (0.35) ([jev-dronecam](https://github.com/twidtwid/jev-dronecam)).
- *Tile size:* in a Where's Waldo test, 2×2 tiles with a presence Noul and a Box found 37 of 54 hard scenes vs. 28 for one call, but 3×3 tiles did worse (22). Confidence 0.7 or more was right 79 of 79 times ([wheres-waldo-jev](https://github.com/aviz85/wheres-waldo-jev)).
- *Focal points:* a Box centre landed 8.7% of the diagonal from the target, and 89% of crops kept it, vs. 96% for GPT-5.5, at $0.06 vs. about $16 per 1,000 photos ([workflows-of-ai](https://workflows-of-ai.com/blog/can-ai-set-the-focal-point-of-a-photo)).

Breaks:
- Box returned zero-width boxes, repeated the same region after masking, and gave one giant box for "find anything".
- Table misalignments were missed.
- Run-to-run spread on the same crop was 0.57 to 0.67.
- Crops under the minimum image size were rejected (officially 256 px a side and 78,400 pixels in total, so about 280 px for a square).

## L. Routing and composition

These four came out of GC AI's 38-project hack day ([write-up][gcai], [project matrix][gcai-matrix]). It ran on Jev 1.13, mostly on small synthetic sets, usually in one run, so treat the numbers as direction.

**33. Typed switch statement.** Jev picks which case an input falls into, with `none` among the options, and deterministic code does everything after that: state changes, replies, routing. No text generation anywhere. This was GC AI's most successful shape overall. It differs from #29 in that the options are a fixed catalog (skills, agents, alert families, inbox buckets, card layouts), not moves enumerated per step.
COMMUNITY ([GC AI][gcai]):
- A drop-in replacement for an agent-routing classifier gave "identical results, much cheaper".
- An alert pager sorted alerts into families (133 of 136 right).
- A Slack pet picked feed, play or scold, plus a kindness Score, and Postgres and code did the rest.
- With several typed answers feeding one action, put the logic in one readable policy file: an inbox triager asked 9 questions per email and bucketed 26 of 26 right (Sonnet 4.6: 25 of 26) at 7x the speed and 16x lower cost.

More COMMUNITY:
- *Effort routing in a coding agent:* a per-step "routine or hard?" Noul cut cost 64% for mini-swe-agent on Terminal-Bench 2.0, while success went from 66.2% to 63.5% ([JevAgentHarness](https://github.com/JevAgentHarness/JevAgentHarness)).
- *Recovery actions:* picking what a failing coding agent should do next fixed 24 of 27 vs. 21 for rules, for $0.56 ([recoveriq](https://github.com/sanghamitrajohri/recoveriq)).
- *Allow / approve / block:* block decisions went from 72% to 85% with better wording and to 97% once one rule was computed in code, vs. 98% for Haiku, at 229 ms vs. 1,523 ms ([jev-triage](https://github.com/mohammedahmadi1990/jev-triage)).

Describe every option, and check that the top pick beats `none`. Breaks: options without descriptions; near-synonym options.

**34. Classify first, then decide.** Ask what *kind* of thing the input is, put that answer (and what it means) into the state, then ask the real decision. It works when misleading surface text would sway one combined call.
COMMUNITY ([GC AI][gcai]): jev-pager over a replayed week of 136 production alerts paged for all 11 real incidents with 1 false page (precision 0.92, recall 1.00), and turned one 49-alert day into a single page. Alarming template text had made it over-escalate until it asked for the alert family first. Related: a probe → decide → verify chain where any verifier "no" vetoes (51 of 51 on fictional document pairs, up to 7 calls per pair).

**35. Send shape, not values.** When the decision depends on structure, strip the values and send only the shape (field names, types, layout). You get privacy for free, and you can cache the answer per shape.
COMMUNITY ([GC AI][gcai]): Jev picked a card layout for each tool result from its field names alone. The right card went from 17% to 70% and the right title from 22% to 89%, and 2,236 lines of hand-built cards were deleted.

**36. LLM writes the Jev question.** For a messy, open-ended request, an LLM turns it into one typed question with criteria, and Jev answers it fast. Always add a "basis" Noul ("is there real information here, or would any answer be a guess?"): with no context, Jev still picked a winner at 0.81 to 0.89.
COMMUNITY ([GC AI][gcai]): a Slack bot that settles debates got 20 of 20 debates and flagged all 10 no-context cases (planner 2.9 s, Jev 0.2 s). Breaks: AI-generated question *lists* over unit tests gave no signal at all. Check generated questions on labeled items before trusting them.
COMMUNITY variant, *decompose and verify* ([PremiseCheck](https://openlair.github.io/premisecheck/), arXiv 2610.21046; 9,267 review comments on 217 papers): an LLM rewrites a review comment as premises without reading the paper, Jev checks each premise against the paper, and the comment scores as its *weakest* premise. That split keeps the LLM on short text and Jev on the long one. With a no-reasoning decomposer it matched holistic GPT-5.5 on NLP reviews (AUROC 63.1 vs. 63.2) at 24.6% of the cost and 45% of the time; Jev on whole comments scored 49.9. Swapping GPT-5.5 in as the verifier was no more accurate and cost 3.1x.

## Not a Jev job

- Prose, a rationale, or an audit trail: use an LLM.
- Anything that needs retrieval or a tool loop mid-decision: use an LLM.
- Multi-hop reasoning as the *final* word: use an LLM.
- Arithmetic, dates, counts, pincites, quote matching, known rules: use code. (Weekday of a date was right 20% of the time; counts were off by one, while per-item Nouls summed in code were exact; rhymes were judged by spelling, not sound.)
- World knowledge and trivia: use an LLM. Jev scored 57% on band trivia vs. 93% for Claude, and didn't know that Nike is the goddess of victory.
- Security judgments: keep them as coded gates. A PR auto-approver's security question caught only 28 to 40% of security-path files.
- Two-hop cross-references in a window: Jev picked the general 60 days over the applicable 120 at 0.86. Full-document context helped (see Window first).
- Pixel-exact geometry in images (alignment, measurements, counts of small objects, precise boxes): use code or classical vision, or verify a Box before you act on it.
- Confidential data in the state: don't. For screens or documents, one option is local OCR that sends only the needed text, never the image. Open-weight look-alikes exist but trail Jev, so they're unproven as a local substitute.
- A fixed, high-volume task where you have plenty of labels: a small fine-tuned classifier or an embedding classifier may beat Jev on both accuracy and latency. Consider Jev for bootstrapping the labels (#28).

[gcai]: https://gc.ai/blog/what-we-learned-building-38-projects-with-jev-in-12-hours
[gcai-matrix]: https://gc.ai/assets/jev-hackday/jev-patterns.html
