# Write-up (MATACS-TH-2026-02)

## 1. AI vs. rules
**AI is used only for reading messy text:** extracting structured facts (licence held by bidder?, install count, price, weeks, warranty years, team items) from each proposal, each with a quoted evidence snippet. It also flags suspicious text aimed at the reviewer.
**Plain code handles everything that decides outcomes:** file validation, mandatory pass/fail, score formulas, ranking, tie-breaks, and the one-line reasons (templated from the score breakdown).
I drew the line there because language understanding is where AI adds value, while arithmetic, thresholds and ordering must be exact, repeatable and auditable.

## 2. Consistency and explainability
- Scoring is deterministic code, so the same extracted facts always give the same score.
- Extraction runs at temperature 0 with a strict JSON schema, and the result is cached by document hash, so re-ranking the same files reuses it. A second extraction runs only when fields are missing, low-confidence or unverified, and any disagreement between runs goes to the reviewer.
- Every extracted fact must carry a quote; code checks that the quote really appears in the document, otherwise the field is marked "unverified".
- Reviewers see a per-criterion table: extracted value, evidence quote, points earned and formula. Rank reasons are generated from that table.
- Missing information is never guessed: it is "not stated" and scores 0 (e.g. a proposal with no stated schedule).

## 3. Human in the loop
- **Checkpoint 1, after extraction:** the reviewer confirms or corrects facts that are low-confidence, unverified or missing.
- **Checkpoint 2, exceptions:** ineligible and flagged proposals are shown with reasons; the reviewer can clear a flag or reinstate an ineligible bid (reason required).
- **Checkpoint 3, final ranking:** the reviewer can override a rank with a comment. The tool never awards; a human makes the award decision.
- **Audit log:** user, timestamp, document hash, model name and version, prompt version, criteria version, raw extraction, corrections, scores, flags, overrides and comments.

## 4. Changing criteria
Criteria are versioned. An edit creates a new version (weights must total 100%) and never changes old results. Each ranking stores the version it used. Because extraction stores facts, not scores, re-scoring under the new version is a cheap deterministic recompute; AI is re-run only if a new criterion needs a fact not yet extracted. The reviewer sees a before/after rank diff and must confirm the new ranking. If an award has already been recorded, it stays tied to the old version.
If a new mandatory criterion is added, proposals already scored are re-checked against it automatically; any that fail move to ineligible, and the reviewer is shown the list before the new ranking is confirmed.

## 5. Risks
| Risk | Safeguard |
|---|---|
| Wrong extraction (bad PDF/OCR, misread) | Evidence quotes verified in code, schema validation, second extraction on uncertain fields, reviewer checkpoint 1 |
| Made-up facts | "Not stated" is allowed and scores 0; unverified quotes are flagged |
| Prompt injection / manipulation (C) | Proposal text is wrapped as untrusted data; the model only fills a schema and cannot influence scoring; injected text is detected and quoted, and the bid is flagged and not auto-ranked |
| Bias or unfair scoring | Scores come only from extracted criteria facts, never from bidder name or reputation; the extraction prompt asks for facts, not judgement; rubrics are fixed and visible; humans can override. (Optional: redact company names before extraction.) |
| Incomplete or non-compliant bids (e.g. subcontracted licence, no stated schedule) | Mandatory rules in code; missing data penalised and surfaced |
| Relative scoring changes with the bid set | Documented, shown to the reviewer; absolute formulas are an admin option |
| Cost / scale | One extraction call per proposal (a second only for uncertain ones), cached by document hash; scoring is free code; re-scoring needs no AI |
| Privacy | Only the uploaded proposals are sent; no keys in the repo |

**Assumptions:** "last 5 years" is counted from review date; unclear dates (C's "since 2020") are flagged; D's missing schedule is scored 0 with a clarification request, not auto-rejected, because schedule is not a mandatory criterion.