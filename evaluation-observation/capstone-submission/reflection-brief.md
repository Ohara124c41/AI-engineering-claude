# Reflection Brief — Evaluation and Observability Capstone

**Name:** _(submitted via Udacity classroom account)_
**Date:** 2026-08-20

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Windows 11 Home, build 10.0.26200 |
| Python version | 3.12.10 |
| Date run | 2026-08-20 |
| Ran any system live? (which) | Yes — System 1 full pipeline + System 1 and System 2 perturbations (Anthropic API via Vocareum proxy). Everything else offline. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped (`01-policy-pipeline/tests.txt` — skips are live-marked tests, "skipped is not failed") |
| Routing output file | `01-policy-pipeline/routing_decisions.json` (9 decisions) |
| auto_approve / human_review / spot_check counts | 0 / 8 / 1 (plus 1 escalation — POL-2025-009, `endorsements_absent`, category `missing_source`; see `pipeline-run.txt` and the screenshot) |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

> From `01-policy-pipeline/perturbation-run.txt` (premium line deleted from a copy of POL-2025-001):
> `{"category": "missing_source", "detected_pattern": "premium_amount_absent", "field": "premium_amount",
> "kind": "escalation", "policy_id": "POL-2025-001", "reason": "Field 'premium_amount' returned null —
> the source document does not contain this information. Retry is futile; escalate to human review."}`
> The log shows exactly **one** `POST /v1/messages` before the escalation. Retrying a `missing_source`
> failure is worse than escalating because the error is in the *document*, not in the model's attempt:
> every retry costs money and latency and, worse, pressures the model toward inventing a premium to
> satisfy the schema. `format` and `consistency` failures are retried (the model can fix its own
> output given the error); absence of source information cannot be fixed by asking again.

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

> POL-2025-001 in `routing_decisions.json`: `confidence_summary` reports a flat 1.0 on `deductible`,
> `exclusions`, `policy_type`, and `premium_amount`, 0.95 on `coverage_limit`, 0.92 on
> `endorsements`, with `fields_below_threshold: []` and `integration_failures: []` — yet
> `decision: "human_review"` with `reason: "reviewer_disagreement=['coverage_limit',
> 'endorsements']"`. The **reviewer** signal — an independent second extraction pass — sent it to a
> human; the other two signals were green. Trusting the model's stated confidence alone, this policy
> auto-approves (every field at or above threshold, four of them at a perfect 1.0) while the two
> fields the independent pass read differently ship unchecked. Confidence is the model grading its
> own work; the reviewer is a second grader — and in this run the second grader dissented on 8 of
> the 9 routed documents.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

> From `01-policy-pipeline/calibration-report.txt`:
> `umbrella  exclusions  n=2 conf=0.93 acc=0.00 brier=0.865` against `OVERALL brier=0.291`.
> The overall Brier score is moderate — the kind of number a dashboard shows green — because the
> well-calibrated `auto/premium_amount` cell (n=3, conf 0.95, acc 1.00, brier 0.003) averages the
> disaster away. The slice catches that for one specific policy-type × field combination the model
> is *confidently wrong every single time* (93% claimed, 0% delivered). An auto-approve threshold
> tuned on the aggregate would wave umbrella exclusions through at 0.93 — the exact cell where
> stated confidence is worthless.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed (`02-mortgage-extraction/tests.txt`) |
| Document run | `fixtures/documents/appraisal_informal_sqft.txt` (`extract-run.txt`) |
| Classified type | `appraisal` (log line: `classify: model=claude-haiku-4-5 in=2169 out=108 type=appraisal`) |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

> From `02-mortgage-extraction/discrepancy-run.txt`: `"validation": {"consistent": false,
> "discrepancies": [{"field": "total_monthly_income", "calculated": 9642.17, "stated": 10892.17,
> "delta": -1250.0}]}`, exit code 1. Tool use guarantees the output is *well-formed*: the right
> fields, the right types, nulls only where the schema's union types allow. The validator guarantees
> the output is *internally coherent*: the numbers bear the arithmetic relationship the domain
> requires. `10892.17` is a perfectly schema-valid number — the schema cannot know it isn't the sum
> of the line items; only the validator's `5416.67+1250.00+2140.00+385.50+450.00 = 9642.17` check
> catches that. Conversely, the validator cannot catch a response that omits the income object or
> returns a string where a number belongs — it never sees it, because the schema gate rejects
> malformed output before validation runs. Structure and semantics are separate failure modes and
> need separate guards.

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

> From `02-mortgage-extraction/extract-run.txt`, `income_missing_bonus.txt` (classified
> `income_verification`): `"bonus_monthly": null` — alongside a real `"base_monthly": 5673.08`.
> Null is the honest answer: the document doesn't state a bonus, and underwriting must be able to
> distinguish "the document said zero" from "the document was silent" — an invented 0 (or a
> plausible-looking figure) silently corrupts a lending decision. The schema choice that makes this
> possible is the nullable union-type idiom in `mortgage_extractor/schema.py` (line 7: "Nullable
> fields use the union-type idiom (`type: ["<base>", "null"]`)…", e.g. line 44
> `"coborrower_name": {"type": ["string", "null"]}`). Because `null` is a *legal* value, the model
> is never forced to choose between violating the schema and fabricating data.

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> `appraisal_informal_sqft.txt` says "about 2,400 sq ft"; the extraction in `extract-run.txt` says
> `"gross_living_area_sqft": 2400` — an integer, no comma, no "about". Normalizing at extraction
> time is the right boundary because that is the only moment the messy source and the clean value
> coexist with full context: the extractor knows "about 2,400" is an approximation of a quantity in
> square feet. Downstream consumers would each have to re-implement fuzzy parsing ("2,400",
> "approx. 2400 sqft", "2.4k ft²") — n parsers, n disagreement modes. One normalization point,
> enforced by an integer type in the schema, means everything after the boundary computes on
> numbers instead of prose.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed (`03-supply-chain/tests.txt`) |
| Briefing file | `03-supply-chain/briefing.md` (extracted from `investigation-run.txt`) |
| Section the conflict landed in | Contested (`on_time_delivery_rate`, flagged ⚠️ ESCALATE) |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

> From `briefing.md`, Contested → `on_time_delivery_rate  [2 sources, conflicting]  ⚠️ ESCALATE`:
> `95.0 percent — supplier_audit (as of 2026-04-10)` vs `78.0 percent — logistics (as of 2026-04-05)`.
> Any single reconciled number destroys exactly the information a risk analyst needs: averaging to
> ~86% describes a supplier that exists in neither source's data, and picking either number silently
> discards the other's measurement. The preserved pair tells the reader something no scalar can —
> that the supplier's own audit and the logistics system *disagree by 17 points*, which is itself
> the risk signal (self-reported vs observed performance), and the escalation line routes that
> question to a human instead of burying it.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

> From `03-supply-chain/timeout-run.txt`: banner `> Sources unavailable: logistics unavailable
> (timeout)`, and in Incomplete: `late_shipment_count  [missing source: timeout reading logistics]`.
> Contrast with `production_capacity_utilization  [missing source: no source reported this metric]`
> in the same section. The two annotations are deliberately different claims: "timeout reading
> logistics" means *the data may exist and we failed to get it* (retry later; distrust this
> briefing's logistics-shaped holes), while "no source reported this metric" means *we looked and
> nothing was there* (a coverage gap, not an outage). The run finishes (exit 0) because the
> coordinator treats a source as an input that can fail, not a dependency that must succeed — the
> other three sources' claims are still delivered, with the failure recorded as provenance instead
> of an exception.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

> Same Meridian metric, five days apart in `briefing.md`: `95.0 percent — supplier_audit (as of
> 2026-04-10)` and `78.0 percent — logistics (as of 2026-04-05)`. Without dates these are just "two
> sources disagree by 17 points." With dates, the reader can ask the right follow-up: did on-time
> delivery genuinely improve in the five days between measurements (say, after the Port of Long
> Beach strike of 2026-03-16 cleared — also in the briefing, dated 2026-03-17), or are the sources
> measuring different things? Requiring `as of` on every claim turns "contradiction" into
> "time-series with a gap" whenever that's what it actually is — and keeps genuine same-period
> conflicts, which are the real alarms, distinguishable from staleness.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

> System 1, `routing_decisions.json`, POL-2025-001: the model reported 0.99 confidence on
> `coverage_limit`, and the independent reviewer pass disagreed with it on that exact field
> (`reason: "reviewer_disagreement=['coverage_limit', 'endorsements']"`). A trusting design —
> auto-approve above a confidence threshold — ships that extraction at 0.99. The evaluating design
> re-derived the answer independently and routed the disagreement to a human. Nothing about the
> model's own output signaled a problem; only *checking it against something the model didn't
> produce* did.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> System 1, twice over. In my live run the routed policies carried field confidences between 0.92
> and a flat 1.0, `fields_below_threshold` was empty on records like POL-2025-001 — and 8 of the 9
> still went to `human_review` on reviewer disagreement (the ninth, POL-2025-007, was pulled only
> by the deterministic stratified spot-check, not by any confidence signal). High stated confidence
> coexisted with independent disagreement on nearly every document. And the calibration report
> shows the mechanism at its worst: the `umbrella/exclusions` cell claims 0.93 confidence with 0.00
> observed accuracy (`calibration-report.txt`). Confidence is an input *to be calibrated against
> outcomes*, sliced finely enough to expose the cells where it lies — never a verdict.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> Extracting line items, totals, and vendor fields from supplier invoices and receipts into an
> accounts-payable system — messy scans, inconsistent layouts, occasional missing PO numbers. I'd
> reach for **validated retry with escalation** first: a JSON schema with nullable unions (so a
> missing PO number is `null`, never invented — the System 2 lesson), an arithmetic check that line
> items sum to the stated total (my `delta: -1250.0` catch generalizes directly to invoice math),
> retry only on format/consistency categories, and escalation on missing-source (the System 1
> single-call boundary). To know when it broke, I'd instrument three signals: the escalation rate
> per document source (a rising rate means an upstream template changed), the retry-success rate by
> error category (falling format-retry success means the model or prompt drifted), and a sliced
> confidence-vs-accuracy calibration table by vendor × field, reviewed against a human-audited
> sample — because the aggregate *will* look fine while one vendor's total field goes the way of
> `umbrella/exclusions`.
