# Field notes

Lessons and numbers from real Jev experiments, mostly legal-research tooling (citation checking, case triage, corpus screening) plus one consumer app. All runs used `jev-1.13.0`. These are one practitioner's results on their own data: treat them as priors, not benchmarks. Patterns in `patterns.md` cite these as **FIELD #n**.

## Standing rules

- **Accuracy is the whole question; cost almost never is.** Every run below cost cents.
- **Window first.** Use located passages or context windows, not whole documents. In #4 the window was half the win; in #7 the window cap was the ceiling.
- **Ask the decision as one plain yes/no** ("Is the proposition, *exactly as written*, fully supported?"). One-of-N Choices broke when the true answer was "both". Jev reads literally, so write the boundaries into the criteria.
- **Use neutral questions, not the user's facts.** Embedding the user's facts in the question made the cases that should have scored high score 0.03 to 0.25; a neutral question separated them at 0.90 vs. 0.15.
- **Reliable at the extremes, mushy in the middle.** Use it to clear or escalate, or to filter or screen. Never use it alone to rank, and never alone to accuse.
- **Fit thresholds on your own labels, out of sample.** In-sample thresholds misled twice. Choice and Score ran overconfident and Noul ran underconfident on unseen material. For a gate, read `probabilities[top_level]`, not the Score expectation.
- **Code catches what Jev is blind to.** A fabricated quote scored 0.99 "supported", and a citation to the wrong case gives Jev no text to judge. Quotes, counts, dates and pincites stay in code, and that code runs first.
- **Keep separate question sets for separate jobs** (clearing vs. grading). Answers shift slightly with the other questions in the same request, so bump a rubric version on any wording change.
- **Seven questions can be one dimension.** In #8, questions that looked independent all scored at chance, and weighting them never beat a simple count.
- **Absolute passage-level judgments are weak; relative ones are strong.** A Choice picking *which* passage worked (the locator in #1); a Noul per passage asking "is this substantive?" was at chance.
- **In a UI, call it a "machine assessment", never a "probability".**
- **Price the best case before building.** In #2, an idea died on paper.
- **Log `response.model` and cache responses** (e.g. keyed by sha256 of the request) so reruns are free and thresholds stay tied to a version.
- **Wrap Jev calls so a failure never stops the run.** Treat Jev as an optional enhancement layer.

## Experiments

### 1. Citation auto-clear gate + locator + phrasing search — SHADOW
- Task: does a cited opinion support the proposition a brief cites it for? State: the top 5 located passages plus the proposition. Gate: Noul "fully supported as written", averaged over two framings. Locator: one Choice over up to 250 numbered passages.
- Locator: top-1 77%, top-3 92%, top-5 96%. The top 5 passages plus their neighbours are 27% of the opinion.
- Gate at 0.90 on four unseen briefs: 22 of 97 cleared, 0 bad. In a 202-claim shadow log, the new wording at 0.50 cleared 38 with 0 bad; the old wording cleared 77 with 13 bad.
- Phrasing search (trying many question wordings against labels) took a held-out set from 22 to 30 cleared, 0 bad. But the nearest bad claim sat at 0.250 against a 0.255 threshold.
- Cost: about $0.0004 and 0.37 s per claim, vs. a frontier LLM at $0.36 to 0.49 and 26 to 41 s.

### 2. Locator feeding a frontier LLM — ABANDONED ON PAPER
- Using the locator to send the LLM only the relevant passages saved 36% at best. With any fallback, savings shrank to little or nothing. A batch API saved 50% with no accuracy risk.
- Side finding: a low `exists` score caught all 13 unsupported claims (all ≤0.23). It worked in one direction only.

### 3. Two-sided claim checker — PROTOTYPE
- 332 claims from 13 briefs. Telling overstated claims from unsupported ones: about 0.86 AUC within a brief. Telling wrong-subject claims from overstated ones: 0.986 on the tuning set, 0.886 held out.
- Rule: clear if `as_written > 0.580`; flag if `min(whose_view, same_issue) < 0.130`. On the held-out set, 36 of 154 claims were cleared (0 bad) and 19 flagged (0 false accusations). Overall, 25% cleared, 11% flagged, and 64% went on to the LLM. Flag recall was only 36 to 43%.
- Live run: 34 claims, 17 s, about 3 cents; it caught a real wrong page cite.

### 4. Citing-case triage — PROBE
- 209 citing cases, each with a ±1,500-character citation-context window. Questions: an `on_issue` Noul, a `depth` Score from 0 to 3, a governing-law Noul, and an `outcome` Choice.
- Jev took 5.0 s and $0.0165; a small LLM took 38.1 s and $0.44. on_issue: 0.947 vs. 0.952. outcome: 0.804 vs. 0.684.
- Filtering with Jev, then sorting by citation count, raised precision@30 from 0.50 to 0.90. Governing law was a weak axis.

### 5. Negative-treatment screen — PROBE
- 1,067 cases citing three overruled Supreme Court cases; 26.7 s, $0.092.
- "No longer good law" at ≥0.5: recall 0.965, precision 0.974. For comparison, a small LLM had recall 0.83, and regex had 0.85 recall at 0.55 precision. Any negative treatment at ≥0.5: recall 0.89, precision 0.85.
- It missed overrulings that had to be inferred, and flagged some "overruled in part" cases that the reference labels called "followed".

### 6. Court-ruling disposition survey — PROBE
- 571 docket entries, 370 of them summary-judgment rulings. "SJ denied?" as one Noul: recall 0.987, precision 0.930. The Choice version failed whenever the answer was "both". Did a specific claim survive: Jev 0.835 vs. a small LLM 0.762.
- End to end: 84 s and $0.06, vs. the small LLM at 216 s and $1.92. Harvesting the documents took 73% of the wall time.

### 7. Corpus screen + question tree + live scorer — LIVE
- Screen: 832 documents, five questions, 22.3 s, $0.0744. It flagged 23 documents, which included every known candidate.
- Question tree: 72 documents, 2.2 s, under a cent. Agreement with a mid-size LLM ranged from 53.5% to 73.6% per attribute. Attributes that depended on how the excerpt was cut scored worst. On the key question, 10 of 11 positives were caught, with 5 false alarms among 15 flagged. One "false alarm" turned out to be Jev right and the LLM wrong.
- The *disagreement* between a broad and a narrow version of a question was what kept all 9 must-have documents.
- Scorer: a neutral question separated 0.90 vs. 0.15 and rescored 136 candidates in about 25 s. The 9,000-character window cap, which 60 of 117 documents hit, was the ceiling.

### 8. Podcast preview + coverage check — IN PRODUCTION (optional layer)
- Episode preview (Choice + Noul) was right on everything checked, with thresholds of YES 0.8 and NO 0.2.
- A per-passage "substance map" was at chance (72% vs. 35% substance).
- A takeaway-coverage check found one real miss. But "is something left out?" scored 0.78 to 0.90 on everything, so it separated nothing.
