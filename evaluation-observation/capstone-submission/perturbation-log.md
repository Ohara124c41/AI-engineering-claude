# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** Copied `data/policies/POL-2025-001.txt` to
  `perturbations/POL-2025-001_no_premium.txt` and deleted the single line
  `Total Policy Premium ........................  $ 1,847.62`, so the required `premium_amount`
  field is genuinely absent from the document. The bundled data was not touched.
- **Command I ran:** `policy-extractor extract perturbations/POL-2025-001_no_premium.txt --policy-id POL-2025-001` (live API)
- **What I predicted:** The extractor would return `premium_amount: null`, the validator would
  classify it `missing_source`, and the system would escalate immediately — exactly one API call,
  no retry, no invented premium.
- **What actually happened (paste the key output line):** One `POST /v1/messages` line in the log,
  then: `"category": "missing_source", "detected_pattern": "premium_amount_absent", "kind": "escalation",
  "reason": "Field 'premium_amount' returned null — the source document does not contain this
  information. Retry is futile; escalate to human review."` Exit code 1.
  (Full capture: `01-policy-pipeline/perturbation-run.txt`.)
- **How this differs from the unperturbed run:** In the full pipeline run over the intact bundled
  set (`01-policy-pipeline/pipeline-run.txt`), POL-2025-001 extracts `premium_amount` with
  confidence 1.0 and proceeds through review and routing (it lands in `human_review` for
  reviewer disagreement, not for a missing field). The perturbed copy never reaches routing: the
  `missing_source` category short-circuits the retry loop after a single call, because re-asking
  the model cannot conjure information the document does not contain.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** The inverse experiment. Copied
  `fixtures/documents/income_sum_mismatch.txt` to `perturbations/income_sum_fixed.txt` and
  *corrected* the stated monthly total from `10,892.17` to `9,642.17` — which is what the line
  items actually sum to (5416.67 + 1250.00 + 2140.00 + 385.50 + 450.00). Bundled fixture untouched.
- **Command I ran:** `mortgage-extract perturbations/income_sum_fixed.txt --mode record --verbose`
  (live API — a perturbed document has no recorded response to replay; the two new cache files it
  recorded are preserved in `perturbations/`)
- **What I predicted:** Identical extraction to the bundled mismatch document, except
  `stated_monthly_total: 9642.17`, and the consistency validator would now report
  `consistent: true` with no discrepancies — proving the flag responds to the arithmetic itself,
  not to anything else about the document.
- **What actually happened (paste the key output line):** `"stated_monthly_total": 9642.17` and
  `"validation": {"consistent": true, "discrepancies": []}`, exit code 0.
  (Full capture: `02-mortgage-extraction/perturbation-run.txt`.)
- **How this differs from the unperturbed run:** The bundled `income_sum_mismatch.txt`
  (`02-mortgage-extraction/discrepancy-run.txt`) produces the same field-by-field extraction but
  `"consistent": false` with `{"field": "total_monthly_income", "calculated": 9642.17, "stated":
  10892.17, "delta": -1250.0}` and exit code 1. Same document type, same schema, same model — a
  one-number edit flips only the validator verdict. (The −1250.00 delta in the original equals the
  bonus line exactly: the document's stated total double-counts or mis-adds the bonus, which is
  precisely the class of upstream arithmetic error the validator exists to catch.)

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** Configuration change, no files edited: added the
  `--simulate-timeout` flag, which forces the logistics source to fail mid-run.
- **Command I ran:** `supply-chain-investigate meridian --offline --simulate-timeout`
  (contrast run: same command without the flag)
- **What I predicted:** The run would complete rather than abort; logistics-only metrics would
  move to Incomplete annotated as a timeout; and metrics where logistics was one of two voices
  would lose that voice.
- **What actually happened (paste the key output line):** The briefing gained the banner
  `> Sources unavailable: logistics unavailable (timeout)` and completed with exit code 0.
  `late_shipment_count` moved to Incomplete as `missing source: timeout reading logistics` —
  wording that is deliberately different from `production_capacity_utilization`'s
  `missing source: no source reported this metric` ("source was unreachable" vs "sources had
  nothing to say"). (Full captures: `03-supply-chain/timeout-run.txt`, contrast in
  `03-supply-chain/perturbation-run.txt`.)
- **How this differs from the unperturbed run:** The most interesting difference is what
  *disappears*: in the unperturbed briefing (`03-supply-chain/investigation-run.txt`),
  `on_time_delivery_rate` sits in **Contested** — 95.0% (supplier_audit, 2026-04-10) vs 78.0%
  (logistics, 2026-04-05) with an ESCALATE flag. With logistics dark, the same metric shows as
  single-source 95.0% under Well-Established and Contested reads `_none_`. The conflict didn't
  get resolved — it became invisible because the dissenting source vanished. The availability
  banner at the top is what tells the reader to distrust that calm: a "healthy-looking" briefing
  with a sources-unavailable banner is a different document from a healthy briefing.
