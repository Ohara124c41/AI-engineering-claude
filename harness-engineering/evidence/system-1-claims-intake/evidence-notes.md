# System 1 — Insurance Claims Intake Agent — Evidence Notes

**System path:** `Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/`
**Environment:** Python 3.12.10, dedicated venv (`C:\Users\<user>\venvs\hx1`), Windows 11 (no WSL), Vocareum proxy (`ANTHROPIC_BASE_URL=https://claude.vocareum.com`)

## What was run

| Step | Command | Result |
|---|---|---|
| Test suite | `pytest tests/ -v` | **29 passed** (see `test-log.txt`) |
| Full run (final) | `python -m claims_intake.run --all --model claude-opus-4-7` | **exit 0; 7 routed + 1 escalated**, run dir `runs/20260820_113839/` (see `run-command.txt`, `summary.md`) |

## Environment problems hit and fixed (Task 1 requirement)

1. **Windows 260-char path limit** — `pip install -e ".[dev]"` failed inside the deeply nested solution folders (`OSError` on a long `dist-info` path; `LongPathsEnabled=0` and enabling requires admin). Fix: venvs created at short paths (`C:\Users\<user>\venvs\hx1..hx4`) instead of in-project `.venv`. No source changes.
2. **`anthropic==0.39.0` vs modern httpx** — first live run crashed with `TypeError: Client.__init__() got an unexpected keyword argument 'proxies'` (httpx 0.28 removed the kwarg the pinned SDK passes). Fix: `pip install "httpx<0.28"` (installed 0.27.2) in the System 1 venv only.
3. **Transient Vocareum 500** — one Sonnet run aborted with `anthropic.InternalServerError` mid-loop; a retry succeeded.

## Model comparison (stand-out; see `model-comparison/`)

| Model | Outcome | Notes |
|---|---|---|
| `claude-haiku-4-5-20251001` (default) | 4–5 of 8 routed/escalated; 3–4 `incomplete`, exit 1 | Failure mode diagnosed: on incomplete claims the model asked its follow-up question **in plain text and stopped with `end_turn`** instead of calling `request_clarification` (probe transcript: "Do you have an initial estimate of the repair costs…?" as final text, `terminal_called: False`). The loop honored the `stop_reason` contract and terminated; the harness correctly reported `incomplete` rather than hanging. |
| `claude-sonnet-4-6` | 4 of 8 `incomplete`, exit 1 | Same failure mode. |
| `claude-opus-4-7` | **8/8 terminated (7 routed, 1 escalated), exit 0** | Matches the README expectation. Higher cost: est. $3.66 vs $0.13 for a Haiku attempt. |

This is direct evidence that the *harness* is deterministic while the *model* is the variable: identical loop, tools, and prompt produced different termination rates purely by model choice.

## Key run numbers (from `summary.md`, run `20260820_113839`)

- claim_03_water_damage: routed (property_damage, high), **7 turns, 2 clarifications**, 37,190 in / 1,802 out tokens, est. $0.693 — the dynamic-decomposition showcase.
- claim_06_low_confidence_escalation: **escalated**, 5 turns, reason: ambiguous across property_damage/auto/liability.
- Queues written: `queues/auto.jsonl`, `liability.jsonl`, `property_damage.jsonl`, `theft.jsonl`; escalation in `escalations.jsonl`.

## stop_reason sequences (from the submitted traces)

`trace-claim_03_water_damage.jsonl`:
```
turn 1 tool_use  [lookup_policy, record_claim_fact ×4]
turn 2 tool_use  [request_clarification]
turn 3 tool_use  [record_claim_fact, request_clarification]
turn 4 tool_use  [record_claim_fact, classify_claim]
turn 5 tool_use  [assess_severity]
turn 6 tool_use  [route_to_adjuster]
turn 7 end_turn  []
```
`trace-claim_06_low_confidence_escalation.jsonl`: `tool_use ×4` (ending in `escalate_to_human`) then `end_turn`.

Loop control lives in `claims_intake/loop.py`, function `run()`: returns on `stop_reason == "end_turn"` (line 103), continues only on `stop_reason == "tool_use"` (line 113), raises `UnexpectedStopReason` otherwise (line 130).
