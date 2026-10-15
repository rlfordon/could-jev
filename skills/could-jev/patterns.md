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

- **Window first.** Send located passages or context windows, not whole documents. Irrelevant text lowers accuracy, and the window is often half the win. The same goes for images: crop to the region, and TypeSafe's own Box guidance says the same (tile first for small targets, re-ask on a tighter crop when unsure). In one study, a nonsense sentence scored 0.90 on a paragraph crop but only 0.60 on the whole page.
- **Code first for anything checkable.** Quotes, counts, dates, arithmetic, pincites and exact lookups belong in code, and that code runs before Jev. Jev will happily call a fabricated quote "supported", and it has nothing to judge when the text is missing.
- **Trust the extremes, not the middle.** Use Jev to clear or escalate, or to filter or screen. The 0.3 to 0.7 band is where the hard cases live; route that band to an LLM or a person rather than ranking on it.
- **Fit thresholds on your own labels, out of sample, and refit when the data shifts.** In-sample thresholds mislead, thresholds don't carry over between datasets, and a gate tuned at one prevalence can miss badly at another. Pin the model version once thresholds are tuned. For a gate, read the probability of the level you care about, not the Score's expectation.
- **Request shape changes answers.** Answers vary a little from run to run, and putting several items in one state or changing the question set can move them more (packing similar items into one array took one test from 96% to 72%). Keep one item per state, version your question sets, and repeat or cache any answer that sits near a threshold.
- **Make options mutually exclusive, and spell out the boundaries.** A Choice assumes exactly one answer. If "both" is possible, ask separate yes/no Nouls instead. Jev reads literally, so put the edge cases in the criteria.

## Worth testing

Hunches with thin or mixed evidence. Suggest them as a **Cheapest test**; don't state them as fact.

- **Neutral questions may beat questions that include your situation.** Asking the general question ("does this case hold X?") separated far better than asking whether it fit the user's facts.
- **Relative judgments may beat absolute ones over passages.** "Which passage decides X?" (a Choice) worked, while "is this passage substantive?" (a Noul per passage) was at chance. That's confounded, though: the failing question was also vague.
- **Calibration may lean by type.** In one set of tests, Choice and Score ran overconfident and Noul ran underconfident on unseen data. Others have independently found Choice probabilities poorly calibrated: they don't match stated odds and barely track how split human annotators were. Don't treat a Choice probability as a frequency.
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
DOCUMENTED (about 10x cheaper than sequential). TESTED.

## B. Cutting a big set down

**3. Funnel before the LLM.** Jev clears the obvious yeses and noes; the LLM only sees the uncertain middle. The gains come from the extremes. Set two thresholds (clear and flag) and send the band between them to the LLM.
TESTED (citation checking: about a quarter cleared and a tenth flagged with no bad calls; the rest went to the LLM). COMMUNITY:
- Conformal thresholds (jev-certify) auto-routed about 85% of queries with about 2% loss.
- Entity resolution with an LLM on the uncertain residue matched 199 of 200 at a tiny fraction of frontier-LLM cost ([Southbridge](https://southbridge.ai/blog/jev-entity-resolution)).
- Incident diagnosis ([SREGym](https://sregym.com)): 80 of 105 passed (76% vs. 78% for an LLM agent), about 7x faster, $0.15 in total.

Breaks: using one threshold; trusting in-sample thresholds.

**4. Census instead of sample.** When the universe can be listed (every court opinion matching a broad query, every ticket this quarter), score *all* of it on one coarse question and audit a random sample to measure the miss rate. That turns "we searched carefully" into "we checked every document; measured miss rate is X%".
UNTRIED as a census with a miss-rate audit. Whole-corpus screens are TESTED and COMMUNITY: about 1,200 pages judged in under 3 minutes; 1,500 posts per channel profiled month by month. Breaks: the universe is only as complete as the query that listed it.

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
TESTED implicitly (the partial-support cases clustered in the middle). COMMUNITY variant: ask the user a clarifying question when the top two options are close. COMMUNITY, on images ([jev-vision-benchmarks](https://github.com/MaceTenth/jev-vision-benchmarks)): five yes/no policy questions per catalog photo; sending the 0.3 to 0.7 band to a person put 4 of 294 answers (1.4%) in review, including both errors, and the other 290 were all right. It caught 7 of 8 problems with 1 false alarm, at about 0.4 s and $123 per million photos (60 photos, Claude-written labels, 8 positives). Warning: Choice probabilities are poorly calibrated on contested items, so rank by closeness or spread, not by the raw value.

**11. Disagreement mining.** Ask a broad and a narrow version of a question, or compare Jev with an LLM or a human. The items where they diverge are the ones to read.
TESTED (the broad-vs-narrow gap kept every must-have document; some "false alarms" were Jev right and the LLM wrong).

**12. Question-wording search.** Because asking is free, *search over phrasings* against a labeled set and keep the wording that separates best. It's prompt engineering with a measurable objective.
TESTED. Breaks: overfitting to the held-out set; bump the rubric version.

**13. Gold-set hygiene.** Run Jev over your gold labels. Where it confidently disagrees, re-check the label first.
UNTRIED.

## E. Making it point

**14. Locator.** One Choice over numbered passages ("which passage decides X?") finds where something is. Jev can't search by itself, but it can point at an option you number.
TESTED (the right passage was usually in the top 3). DOCUMENTED (line-ID search). COMMUNITY: routing callers' questions to the right website page ([Sayora](https://sayora.ai)) got the right page first 88% of the time vs. 70% for embeddings, and correctly said "none of these" 95% of the time vs. 23%, for $0.45 over 1,449 requests. That's the case for always offering `none`.

**15. Paragraph heat map.** Ask the same Noul of every paragraph to get a probability curve through the document, showing where an issue is decided and where it's only mentioned. Use it for windowing, for reading UIs, and to jump straight to the key passage.
COMMUNITY: jev-skip paints a sponsor-segment probability on the YouTube seek bar (caught about 77% of sponsor seconds, under $0.001 a video). Per-line "grep by meaning" tools do the same over code. For images, fan the same Noul out over tiles or crops to get a heat map of the picture (see #32). Warning: a vague per-passage question ("is this substantive?") was at chance. Prefer a relative judgment (a Choice over passages) or very concrete Nouls.

**16. Remove-a-sentence attribution.** Delete one sentence at a time, re-ask, and watch the probability move. The sentences that move it most are what drove the judgment. Cheap explainability for a model that can't explain itself.
UNTRIED.

**17. Pre-parsed extraction.** Code finds candidate values (dates, ID numbers, names, statutes cited); Jev picks which one is *the* answer; code copies it verbatim. It can't hallucinate a value.
DOCUMENTED. COMMUNITY variant: for spans, code nominates candidates, Jev verifies each one, and code fixes the boundaries (named-entity recognition at about 74 strict F1 averaged over 12 benchmarks).

## F. Two texts at once (state as an array)

**18. Pairwise judgments.** "Same issue?" "Does B apply A's rule?" "Does this passage support this claim?" "Did this edit change the meaning?" These power deduplication, clustering, treatment checks, version diffs, and matching a question to candidate sources.
TESTED (claim plus passage). DOCUMENTED (reranking a BM25 shortlist on legal queries: top-10 from 38% to 62%). Generic record dedupe has been reported as underwhelming, but funnelled entity resolution worked well (see #3). With images (experimental), the pair can be two pictures or a picture plus a claim, for example "is this upload the product in the catalog photo?" or "is the product in this returned-item photo damaged, as the claim says?". COMMUNITY (demo): a reference photo plus video frames, with a Noul for "visible" and then a Box ([video-image-tagger](https://github.com/botheaj/video-image-tagger)). COMMUNITY (numbers): "does this featured image fit this headline?" separated own vs. random headlines at AUC 0.91 over 30 images, and rejected all 60 off-topic pairings (Thorsten Meyer, read via a republished copy); look-alike stock photos were the misses.

**19. Tournament ranking.** Many cheap pairwise "which is more on point?" judgments add up to a ranking, which sidesteps poor absolute calibration.
UNTRIED. Watch Choice order bias; ask both orders.

## G. Watching processes

**20. LLM-output check.** A cheap screen on a draft before anything expensive runs: does this sentence's citation support it; is this summary faithful to that passage?
TESTED. COMMUNITY: Jev assertions inside test suites (pytest-jev). Breaks: vague "is anything missing?" questions, which score high on everything.

**21. Agent tripwires.** Inside a long agent run: is this step still on the task? Is this draft answering what was asked? Is this work actually done? The answer becomes a gate in a hook.
COMMUNITY: a tool-permission gate tested on about 7,900 labelled calls made 1 unsafe allow among 3,664 risky ones and settled 62% of real work at about 160 ms ([jev-permission-gate](https://github.com/madisonrickert/jev-permission-gate)). Others: wake-or-sleep gates for idle agents, and skill routers. Breaks: injected text didn't push dangerous calls through, but it caused about 10% false denials, and "authority" framing moved a few.

**22. Novelty and saturation detector.** "Does this new item add something not already in the set?" Plot the number of new items per round and stop when the curve flattens.
COMMUNITY, as a stopping rule for search: asking "is there enough context to answer?" matched an LLM's recall at about a quarter of the latency (AUROC about 0.90 vs. 0.69 for small local models).

## H. Speed-dependent

**23. Live feedback.** At 0.1 to 0.4 s, Jev can run as you type or click. For example: flag a sentence that states a conclusion without a source; badge each item in a reader with its attributes; check a student's answer against a rubric before they submit.
UNTRIED.

**24. Query to filters.** Break a plain-language question into attribute constraints (jurisdiction, date range, category, fact branch), then filter the collection before retrieval.
UNTRIED.

## I. Teaching and personal tools

**25. Misconception classifier.** Map a student's answer onto a taxonomy of known errors. Use the result to give targeted feedback, pick the next hint, or tally misconceptions across a class.
UNTRIED.

**26. Feed triage.** For new reports, new filings, new podcast episodes, or new papers: "is this in my interests, and which project does it belong to?" Ask before anything downloads, transcribes, or summarizes.
TESTED.

## J. New shapes from the community

**27. Generate-and-test recognizer.** A brute-force or search process produces candidates, and Jev judges which output is "real" (plausible, valid, on-target). Jev is a strong *recognizer* even where it's a weak *policy*: let code explore and Jev recognize.
COMMUNITY: [enigma-jev](https://github.com/agodoy21/enigma-jev) judges whether each candidate decryption is real German (159 of 160, and it held up under distribution shift where XGBoost collapsed). Breaks: using Jev to *choose the next move* in the search had no skill there, and a compiler-optimization policy lost to the baseline.

**28. Distill-and-defer.** Jev labels live traffic. A small local model trains on those labels and answers when its confidence bound allows; otherwise it defers to Jev. Over time, most traffic never leaves the machine.
COMMUNITY: [Jevstiller](https://jevstiller.pages.dev/posts/the-guarantee/) answered 25 to 90% of requests locally at about 15 ms, with 1 to 1.5% disagreement under a 2% budget, across four intent and topic datasets. A good fit when the task is fixed and traffic is high, because specialists can beat Jev there.

**29. Legal-move picker.** Code enumerates the legal actions (page controls, state-machine transitions, game moves), and one Choice per step picks one. It's fast enough to drive a UI or agent loop in real time.
COMMUNITY: [jev-ultrafast](https://github.com/browser-use/jev-ultrafast) completed a flight search in about 7 s; [jevspresso](https://github.com/statelyai/jevspresso) picks transitions in a state machine. Small numbers so far. With images, the state can be a screenshot with numbered elements, and the Choice picks a number; coordinates come from the UI tree, not from Jev. [jev-phone](https://github.com/dongxinsuperman/jev-phone) took about 0.7 s per step (demo, no accuracy reported). Prefer the accessibility tree when there is one, and fall back to the screenshot. Breaks: long-horizon planning, which needs a real planner.

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

Breaks:
- Box returned zero-width boxes, repeated the same region after masking, and gave one giant box for "find anything".
- Table misalignments were missed.
- Run-to-run spread on the same crop was 0.57 to 0.67.
- Crops under the minimum image size were rejected (officially 256 px a side and 78,400 pixels in total, so about 280 px for a square).

## Not a Jev job

- Prose, a rationale, or an audit trail: use an LLM.
- Anything that needs retrieval or a tool loop mid-decision: use an LLM.
- Multi-hop reasoning as the *final* word: use an LLM.
- Arithmetic, dates, counts, pincites, quote matching, known rules: use code.
- Pixel-exact geometry in images (alignment, measurements, counts of small objects, precise boxes): use code or classical vision, or verify a Box before you act on it.
- Confidential data in the state: don't. For screens or documents, one option is local OCR that sends only the needed text, never the image. Open-weight look-alikes exist but trail Jev, so they're unproven as a local substitute.
- A fixed, high-volume task where you have plenty of labels: a small fine-tuned classifier or an embedding classifier may beat Jev on both accuracy and latency. Consider Jev for bootstrapping the labels (#28).
