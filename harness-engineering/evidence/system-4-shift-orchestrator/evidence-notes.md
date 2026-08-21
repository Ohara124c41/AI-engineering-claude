# System 4 — Multi-Shift Quality Monitoring Orchestrator — Evidence Notes

**System path:** `Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/`
**Environment:** Python 3.12.10, dedicated venv (`C:\Users\<user>\venvs\hx4`), Windows 11 (no WSL)
**Mode:** fully offline via `--recorded-response` (no API spend for this system)

## What was run

| Step | Command | Result |
|---|---|---|
| Test suite | `pytest tests/ -v` | **33 passed** (see `test-log.txt`) |
| Warm-tier seed | `WarmStore(...).initialize(); insert_many(fixtures/defects.json)` | 40 defect rows inserted (see `warm-seed-output.txt`) |
| Shift run | `python -m shift_monitor run-shift --shift C --warm-db data/warm.sqlite --since 2026-04-20T00:00:00Z --recorded-response fixtures/recorded_responses/shift_C_2026-04-30.json` | exit 0, summary printed (see `shift-output.txt`) |

**Note on `--since`:** the CLI default window is "8 hours before now" (`_default_since()` in `shift_monitor/__main__.py`). The fixture defects are dated March–April 2026, so a first run with the default window correctly returned a 0-row slice. The documented re-run pins `--since 2026-04-20T00:00:00Z` so the SQL pre-filter demonstrably selects a **5-of-40** slice matching the recorded response's shift date (2026-04-30). No code was edited — `--since` is a first-class CLI flag.

## Rubric evidence

- **SQL-filtered slice, not full history** — `sql-slice-evidence.txt`: warm-tier total = **40** defects; the query `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?` (`defects_since` in `shift_monitor/warm.py`, backed by index `idx_defects_shift_ts`) returned **5** defects to the model, via `gather_new_defects(warm, since_ts, limit=50)` in `shift_monitor/pipeline.py`.
- **Hot state within budget** — `hot-state-size.txt`: `data/hot_state.json` = **752 bytes** after the run (budget ~5 KB = ~6.6× headroom).
- **Scratchpad append** — one JSONL line appended per shift run to `data/shift_scratchpad.jsonl` (copy in this folder; `hypothesis_id: "shift-C"`, `evidence: "5 new defects analyzed since 2026-04-20T00:00:00Z"`).
- **Resume-vs-fresh recovery** — `shift_monitor/recovery.py`, function `decide(state, now)`: resumes only if the manifest's last step is within `STALE_RESUME_THRESHOLD_MINUTES = 30` of now and the run is incomplete; otherwise returns `"fresh"` (fresh session with manifest findings injected as a summary). The module docstring notes 30 min ≈ 1/16 of an 8-hour shift cycle.
- **Fork isolation** — `fork-isolation-evidence.txt`: two hypothesis forks created with `fork_for_hypothesis()` (`shift_monitor/fork.py`), each getting a *copy* of `hot_state.json` in `data/forks/<hypothesis_id>/` and its own `scratchpad.jsonl`. Main scratchpad stayed at 2 lines while both forks wrote (isolation), and only an explicit `merge_findings()` call brought the two findings back (4 lines after merge).

## Test-count discrepancy note

Observed: **33 passed**. The capstone task list also says 33; the rubric text says 28. Actual output preserved unmodified in `test-log.txt`; nothing was forced to match either number.
