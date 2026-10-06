# Jev patterns: what it makes possible

Jev has four levers:
- **Nearly free:** cents per thousand documents.
- **Fast:** 0.1 to 0.4 s per request.
- **Full distributions:** probabilities, not just a label.
- **Many named questions per state:** and the state can hold several texts.

Every pattern below is some combination of those. "Classify X" is the least interesting use.

Status tags:
- **FIELD #n:** an experiment in `field-notes.md`.
- **DOCUMENTED:** TypeSafe or the community shows it working.
- **UNTRIED:** an idea.

When the user's own history has a matching experiment, cite that instead.

## A. Getting a sense of things

**1. Quick read / profile.** Ask 10 to 20 questions of one document at once and get its profile: what kind of document it is, its stance, its outcome, whether it reaches the issue, and how deep it goes. A "what is this?" read for a fraction of a cent, before you decide whether to read it.
FIELD #8 (episode preview, right on everything checked), #4 (citing-case profile). Breaks: questions that need inference across the whole document.

**2. Fan-out first, decide later.** Ask every question you *might* need in one request and let code decide which answers matter. Speculation is free, so the design question becomes "what would I want to know?", not "what can I afford to ask?".
DOCUMENTED (about 10x cheaper than sequential). FIELD #7 (a whole question tree in one call).

## B. Cutting a big set down

**3. Funnel before the LLM.** Jev clears the obvious yeses and noes; the LLM only sees the uncertain middle. The gains come from the extremes. Set two thresholds (clear and flag) and send the band between them to the LLM.
FIELD #1, #3 (25% cleared, 11% flagged, 64% to the LLM), #6, #4 (precision@30 from 0.50 to 0.90). Breaks: using one threshold; trusting in-sample thresholds.

**4. Census instead of sample.** When the universe can be listed (every court opinion matching a broad query, every ticket this quarter), score *all* of it on one coarse question and audit a random sample to measure the miss rate. That turns "we searched carefully" into "we checked every document; measured miss rate is X%".
UNTRIED. Closest: FIELD #7's 832-document screen. Breaks: the universe is only as complete as the query that listed it.

**5. Treatment screen.** Run one question over every citing or referencing document's context window: does it follow, distinguish, limit, or overrule (or, outside law, endorse, dispute, or supersede)?
FIELD #5 (recall 0.965 on "no longer good law"). Breaks: treatment you have to infer.

## C. Taxonomies and structure

**6. Question tree.** Encode a decision tree as many questions in one call; code walks it. The tree *is* a schema for the facts.
FIELD #7. Breaks: attributes that depend on how the excerpt is cut (worst attribute: 53.5% agreement).

**7. Big taxonomy, beam search.** Past 255 options, or for a deep hierarchy, descend level by level keeping the top K branches.
DOCUMENTED (K=3 beat greedy). Candidates include subject headings, legal topic outlines, product catalogs, support-ticket routing, and content tagging.

**8. Taxonomy debugger.** Run a draft taxonomy over a few hundred items and look at the *distributions*:
- Two categories that keep splitting the probability overlap.
- A category that never wins is dead.
- A high `none` rate shows a gap.

Fix the taxonomy, not the labels.
UNTRIED.

**9. Metadata layer and changed-fact re-sort.** Store per-item attributes along with their distributions. Then change one fact in a hypothetical and re-rank the collection by how well each item matches. That becomes a query instead of a re-read.
UNTRIED.

## D. Using the probabilities themselves

**10. Uncertainty spotlight.** Spend human reading where Jev is *least* sure. The 0.3 to 0.7 band is where the interesting, contested, or badly worded items live.
FIELD #1 implicitly (all the "partial support" claims sat in the middle). UNTRIED as a reading-order tool.

**11. Disagreement mining.** Ask a broad and a narrow version of a question, or compare Jev with an LLM or a human. The items where they diverge are the ones to read.
FIELD #7 (the broad-vs-narrow gap kept all 9 must-have documents; one "false alarm" was Jev right and the LLM wrong).

**12. Question-wording search.** Because asking is free, *search over phrasings* against a labeled set and keep the wording that separates best. It's prompt engineering with a measurable objective.
FIELD #1 (22 to 30 cleared, 0 bad). Breaks: overfitting to the held-out set; bump the rubric version.

**13. Gold-set hygiene.** Run Jev over your gold labels. Where it confidently disagrees, re-check the label first.
UNTRIED (FIELD #7's Jev-right "false alarm" suggests it pays).

## E. Making it point

**14. Locator.** One Choice over numbered passages ("which passage decides X?") finds where something is. Jev can't search by itself, but it can point at an option you number.
FIELD #1 (top-3 92%). DOCUMENTED (line-ID search over 218 lines).

**15. Paragraph heat map.** Ask the same Noul of every paragraph to get a probability curve through the document, showing where an issue is decided and where it's only mentioned. Use it for windowing, for reading UIs, and to jump straight to the key passage.
UNTRIED. Warning: FIELD #8's per-passage absolute "substance" question was at chance. Prefer a relative judgment (a Choice over passages) or very concrete Nouls.

**16. Remove-a-sentence attribution.** Delete one sentence at a time, re-ask, and watch the probability move. The sentences that move it most are what drove the judgment. Cheap explainability for a model that can't explain itself.
UNTRIED.

**17. Pre-parsed extraction.** Code finds candidate values (dates, ID numbers, names, statutes cited); Jev picks which one is *the* answer; code copies it verbatim. It can't hallucinate a value.
DOCUMENTED.

## F. Two texts at once (state as an array)

**18. Pairwise judgments.** "Same issue?" "Does B apply A's rule?" "Does this passage support this claim?" "Did this edit change the meaning?" These power deduplication, clustering, treatment checks, version diffs, and matching a question to candidate sources.
FIELD #1, #3 (claim plus passage is pairwise). DOCUMENTED (rerank: top-10 from 38% to 62% on CLERC legal queries). Field warning: generic record dedupe was "underwhelming".

**19. Tournament ranking.** Many cheap pairwise "which is more on point?" judgments add up to a ranking, which sidesteps poor absolute calibration.
UNTRIED. Watch Choice order bias; ask both orders.

## G. Watching processes

**20. LLM-output check.** A cheap screen on a draft before anything expensive runs: does this sentence's citation support it; is this summary faithful to that passage?
FIELD #1, #3, #8. Breaks: vague "is anything missing?" questions (#8 scored 0.78 to 0.90 on everything).

**21. Agent tripwires.** Inside a long agent run: is this step still on the task? Is this draft answering what was asked? Is this work actually done? The answer becomes a gate in a hook.
UNTRIED (the community has shown agent-flow gates and skill routers).

**22. Novelty and saturation detector.** "Does this new item add something not already in the set?" Plot the number of new items per round and stop when the curve flattens.
UNTRIED.

## H. Speed-dependent

**23. Live feedback.** At 0.1 to 0.4 s, Jev can run as you type or click. For example: flag a sentence that states a conclusion without a source; badge each item in a reader with its attributes; check a student's answer against a rubric before they submit.
UNTRIED.

**24. Query to filters.** Break a plain-language question into attribute constraints (jurisdiction, date range, category, fact branch), then filter the collection before retrieval.
UNTRIED.

## I. Teaching and personal tools

**25. Misconception classifier.** Map a student's answer onto a taxonomy of known errors. Use the result to give targeted feedback, pick the next hint, or tally misconceptions across a class.
UNTRIED.

**26. Feed triage.** For new reports, new filings, new podcast episodes, or new papers: "is this in my interests, and which project does it belong to?" Ask before anything downloads, transcribes, or summarizes.
FIELD #8.

## Not a Jev job

- Prose, a rationale, or an audit trail: use an LLM.
- Anything that needs retrieval or a tool loop mid-decision: use an LLM.
- Multi-hop reasoning as the *final* word: use an LLM.
- Arithmetic, dates, counts, pincites, quote matching, known rules: use code.
- Confidential data in the state: don't.
