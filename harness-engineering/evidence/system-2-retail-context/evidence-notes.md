# System 2 — Retail Support Context Strategy — Evidence Notes

**System path:** `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/`
**Environment:** Python 3.12.10, dedicated venv (`C:\Users\<user>\venvs\hx2`), Windows 11 (no WSL), Vocareum proxy
**Model:** `claude-haiku-4-5-20251001` (default)

## What was run

| Step | Command | Result |
|---|---|---|
| Full run | `python -m retail_context.run --all` | exit 0, run ID **20260820-114154** (see `run-command.txt`) |
| Test suite (after run) | `pytest tests/ -v` | **30 passed** (see `test-log.txt`) |

**Test-count note:** on a fresh checkout the suite reports **28 passed, 2 skipped** — `test_assembled_context_active_segment_byte_exact` and `test_budget_json_section_counts_sum_consistently` skip themselves until run artifacts exist ("no run artifacts available — run `python -m retail_context.run --build` first"). After the live run, all **30 pass**. The rubric's "17 tests" figure matches neither state; observed output is preserved unmodified.

## Rubric evidence (run 20260820-114154)

- **≥50% reduction** — `budget.json`: baseline **38,708** tokens → assembled **16,838** tokens = **56.5% reduction**. Token counts come from the Anthropic `messages.count_tokens` endpoint (model-authoritative, recorded in `token_counter_methodology`).
- **Per-section tokens** — case_facts **204**, resolved_refund **381**, resolved_subscription **482**, active **15,789**. The active segment dominates (93.8% of assembled) and is kept **byte-exact verbatim** (`assemble.py` keeps `active_raw_text`; the anti-pattern test verifies byte-exactness).
- **Eval ≥5/6** — `eval.jsonl`: **6/6 passed** (refund amount $22.14, cancellation reason, failure code AVS_MISMATCH, card last-4 7782, proration refund status, structured status token).
- **Control regression** — `eval_control.jsonl` (case-facts block stripped): **Q6 FAILED** — "I cannot find a structured status token for the payment-method update issue in the provided context." Q1 unexpectedly passed (the $22.14 figure also survives in the compressed refund summary). Regression on ≥1 question satisfied by Q6.

## What was summarized vs preserved

- **Summarized (LLM compression):** the two *resolved* issue segments — refund (12,344 tokens in → 368-token summary) and subscription (11,485 in → 469 out), per `budget.json`'s `compression_api` block.
- **Preserved verbatim:** the **active** payment-method issue (15,789 tokens, byte-exact at the bottom of the context, nearest the new user turn) and the **case-facts block** (204 tokens of structured, exact-value facts: IDs, amounts, status tokens) at the top.
- **Why:** resolved threads only need gist; the active thread needs full fidelity because the next turn operates on it, and exact tokens (AVS_MISMATCH, last-4 7782) are precisely what compression tends to destroy — which is what the Q6 control failure demonstrates.
