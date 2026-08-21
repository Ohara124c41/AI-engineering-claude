# Claude AI Engineering — Course Projects

Completed coursework for the Udacity **Claude AI Engineering** track. Four projects, each
built, run against real APIs, verified, and submitted. All four passed review.

Every project keeps the course's original repository structure. What was added is the
implementation work, the evidence produced by running it, and a set of implementation notes
recording where current library and service versions diverged from the written instructions.

---

## The four projects

### 1. [Harness Engineering](harness-engineering/) — capstone

Build, run, and verify four reference systems, then defend their design in an evidence-grounded
reflection brief.

- Agentic loop driven by `stop_reason`; context strategy that halves token load; Claude Code
  configuration hierarchy; Layer 3 orchestrator with tiered state and crash recovery
- Test counts: **29 + 30 + 35 + 33** passing
- Evidence in [`evidence/`](harness-engineering/evidence/), brief in
  [`reflection-brief.md`](harness-engineering/reflection-brief.md)
- Notable: a three-model comparison showing the *same harness* completes 8/8 claims on one model
  and leaves 3–4 incomplete on another — the harness is deterministic, the model is the variable

### 2. [Agentic Analyst](agentic-analyst/) — PriceScout MCP competitor analyst

A chatbot that scrapes LLM-inference pricing pages through a custom MCP server and stores
structured results in SQLite via a second MCP server.

- Custom Python MCP scraper (Firecrawl), pre-built SQLite and filesystem MCP servers, and an
  Anthropic-driven client that orchestrates all three
- Evidence in [`evidence.md`](agentic-analyst/evidence.md)
- Notable: the agent follows a **memory-first policy** — after the initial scrape, follow-up and
  comparison questions are answered entirely from stored data (verified: exactly one
  `scrape_websites` call per session)

### 3. [Evaluation and Observability](evaluation-observation/) — capstone

Reproduce, run, deliberately break, and analyze three systems that give Claude work that must
be right.

- Validated/routed insurance extraction; schema-enforced two-pass mortgage extraction with
  arithmetic validation; provenance-preserving supply-chain synthesis
- Test counts: **45 (+3 skipped) + 25 + 34**; mypy and ruff clean on all three
- Evidence in [`capstone-submission/`](evaluation-observation/capstone-submission/)
- Notable: the calibration report exposes a `umbrella × exclusions` cell at **0.93 confidence,
  0.00 accuracy** while the aggregate Brier score stays comfortable — the clearest illustration
  in this repo of why aggregate metrics lie

> Selected reviewer feedback: *"You debugged your environment instead of working around it.
> Blocked mypy DLLs, numpy stubs on the wrong Python version, cp1252 encoding — each diagnosed
> and fixed without touching source files. […] You found something the brief didn't ask for —
> that a source going dark makes a conflict invisible rather than resolved, and the availability
> banner is the only warning."*

### 4. [Enterprise Multi-Agent Review](enterprise-multiagent-review/) — Claude Agent SDK

A production-shaped multi-agent code reviewer: an orchestrator dispatches three specialized
subagents at a GitHub pull request and aggregates their findings into a validated report.

- Claude Agent SDK, MCP (GitHub + ESLint), Zod-enforced structured output, retry/timeout/rate
  limiting, 78 passing tests
- Implementation in [`project/starter/`](enterprise-multiagent-review/project/starter/),
  reports in [`reports/`](enterprise-multiagent-review/project/starter/reports/)
- Notable: on one test PR the code-quality analyzer found a **planted supply-chain attack** in
  `jest.config.js` — obfuscated code using an Ethereum wallet address for command-and-control
  discovery — and escalated it as the top critical finding

---

## Outside the project scope — what had to be solved in real time

Coursework isolates a concept by removing everything around it. The environment doesn't
cooperate. Roughly all of the difficulty in this repo came from things no assignment mentioned:
upstream libraries that moved after the material was written, defects in the supplied
instructions, platform constraints, and model behavior that had to be constrained by code.

Each item below blocked a project until it was diagnosed. None of them were solved by
working around the problem — the fixes are in the code and documented in each project's
"Implementation Notes."

### Upstream drift — the libraries moved

| What broke | Symptom | Resolution |
|---|---|---|
| `firecrawl-py` 4.x dropped the `success` field | The instructions' `scrape_result.get('success', False)` marks **every** scrape failed; v4 raises on failure instead | Treat "content returned for a requested format" as success, still honoring `success` when present |
| `mcp-server-sqlite` vs `mcp` 2.x | Server dies at startup: `AttributeError: 'Server' object has no attribute 'list_resources'` | Pin the server's runtime dep: `--with "mcp>=1.10.1,<2"` |
| `anthropic==0.39.0` vs `httpx` ≥ 0.28 | `TypeError: Client.__init__() got an unexpected keyword argument 'proxies'` | Pin `httpx<0.28` in that venv only |
| `numpy` ≥ 2.2 stubs vs mypy at `python_version = 3.11` | Stubs use the 3.12-only `type` statement; mypy aborts before checking any project code | Pin `numpy<2.2`; re-verify tests |
| `mcp-server-sqlite` row format | Returns Python-repr strings, not JSON — `json.loads` rejects them | `ast.literal_eval` fallback |
| MCP `TextContent` access | Instructions subscript `content[0]["text"]`; the objects use attribute access | Read `.text` with a dict fallback |
| `vitest` 4 | Removed the `basic` reporter used for capture | Use the default reporter |

### Defects in the supplied material

- **The prescribed SQL `INSERT` is injectable by apostrophe** — and the project's own required
  test prompt (*"Compare cloudrift ai and **deepinfra's** costs…"*) triggers it, producing
  `sqlite error: near "s": syntax error` on every insert. Fixed by escaping `'` → `''` while
  preserving the specified statement and column order.
- **Extensionless ESM imports in provided files broke the compiled build.** Three supplied
  modules imported siblings without `.js`. Under `"type": "module"` both `tsx` and `tsc` accept
  this, so it is invisible in development — but `npm start`, the documented production path,
  died with `ERR_MODULE_NOT_FOUND`. Only surfaced by testing `npm run build` rather than
  trusting `tsc --noEmit`.
- **Two strict-mode type errors in the provided rate limiter** appear the moment its TODOs are
  implemented (`noUncheckedIndexedAccess`); guarded rather than cast away.
- **Test counts disagree across sources.** One capstone's task list says 29 + 30 + 35 + 33; its
  rubric says 29 + 17 + 35 + 28. Neither matches for two of the systems. Actual observed output
  was preserved verbatim and the discrepancy documented rather than forced to match either.
- Outdated model identifier in a rubric example; a clone URL that 404s; and validation
  checklists whose items were missing from the text materials entirely — visible only in
  screenshots of the classroom UI.

### Platform reality

- **Windows 260-character path limit** — `pip install -e` fails inside the deeply nested course
  folders. Virtual environments relocated to short paths.
- **mypy's compiled wheel blocked by Windows Application Control** —
  `DLL load failed while importing fscache`. Reinstalled from source (`--no-binary mypy`).
- **cp1252 console encoding** vs Unicode in program output (`—`, `⚠️`, `❌`) — `UnicodeEncodeError`
  mid-run; resolved with `PYTHONUTF8=1`.
- **The Agent SDK's bundled CLI hijacks an interactive login.** The SDK spawns Claude Code's
  CLI, which on a machine where Claude Code is already authenticated sends *its* OAuth bearer
  token instead of `ANTHROPIC_API_KEY` — rejected by the proxy with
  `400 "This key was not found"`, while the same key works fine for direct SDK and `curl` calls.
  Fixed with an isolated `CLAUDE_CONFIG_DIR` that pre-approves the environment key. Omitting the
  pre-approval produces a *silent infinite hang* on an invisible consent prompt, which reads as
  a timeout.
- **Skills don't resolve by default** — `settingSources: ['project']` plus an explicit `cwd` are
  required before the `Skill` tool can find `.claude/skills/`.
- **`structured_output` isn't on the SDK's result union** — it exists only on the success
  variant, so the documented access pattern doesn't compile.
- **The API proxy's model allowlist flapped mid-session** — the same key accepted a model alias
  at 15:16, returned 400 for it at 15:27, and accepted it again at 15:29. Two class keys had
  different allowlists. Documented so a reviewer hitting it knows to retry rather than debug.

### The world moved underneath the fixtures

The MCP project's assignment is to compare DeepSeek **V3** pricing across providers. By the time
it ran, DeepInfra listed DeepSeek **V4** and CloudRift had stopped publishing per-model pricing
entirely. The correct behavior is to report what the sources actually say rather than what the
assignment expects — so the deliverable answers honestly and the divergence is documented.

### Model behavior that had to be constrained by code

None of these were in any specification; all were found by checking output against its own
detail:

- **Invented provenance.** The model produced confident, plausible, wrong timestamps — one
  report claimed a December 2025 date for an August 2026 run. Both `analyzedAt` and `duration`
  are now stamped from measured values.
- **Arithmetic that contradicted its own findings.** Report summaries claimed 9 critical issues
  over file reviews containing 6; another claimed 15 critical / 11 refactorings against an actual
  13 / 8. All five headline counters are now recomputed from the underlying findings, with a
  warning logged whenever the correction fires — which it still does.
- **Model choice changes outcomes, not just cost.** Identical harness, prompts, and tools: one
  model terminated all 8 fixture claims correctly; another left 3–4 incomplete, answering in
  prose instead of calling the terminal tool.
- **Run-to-run nondeterminism is real and worth stating.** The same pull request scored 85, 92,
  and 82 across three runs; an identical pipeline run split 0/9/0 one time and 0/8/1 the next.
  Documented so nobody mistakes variance for regression.

### Operational tuning from measurement

The multi-agent orchestrator's timeout began at 15 minutes and was raised to 30 after measuring
actual runs (~3.5 minutes for one file, up to ~25 for four under high API latency). The
too-tight budget was discovered the honest way — a real run exhausted it, and the retry path,
typed error, and "reports persist only on success" guarantee all behaved correctly under a
failure nobody planned.

## A theme worth naming

Three of the four projects independently converged on the same lesson, and the last one is
built around it: **evaluate the output; don't trust the model's word for it.**

- The evaluation project's calibration slice catches a model that is confidently wrong in one
  specific cell that the aggregate hides.
- The insurance pipeline routes on an *independent* second extraction pass rather than the
  model's self-reported confidence — which, in the live run, dissented on 8 of 9 documents that
  all carried near-perfect confidence scores.
- The multi-agent reviewer recomputes its own report's headline counters from the underlying
  findings, because the model's arithmetic drifted (claiming 9 critical issues where its own
  file reviews contained 6). It also stamps timestamps from measured values after the model
  invented a date eight months off.

The dividing line that emerged: **prose and judgement come from the model; arithmetic,
provenance, and enforcement come from code.**

---

## Running any of these

Each project's README carries its own setup, prerequisites, and verification commands, plus an
"Implementation Notes" section documenting environment-specific workarounds. In short:

- Python projects: create a virtual environment, `pip install -e ".[dev]"`, then `pytest`
- The MCP project: `uv venv && uv sync`
- The Agent SDK project: `npm install`, then `npm run build` / `npm test`

Credentials are read from environment variables in every project. No `.env` file is committed;
where one is needed, a `.env.example` documents the required variables.

## Note on reproducibility

Runs against live models are not deterministic. Re-running a project will produce different
scores, issue counts, and phrasing than the committed evidence shows — in one case the same
pull request scored 85, 92, and 82 across three runs. The committed artifacts are honest
snapshots of specific runs, not fixtures to reproduce exactly. Where this matters, the
individual project READMEs say so explicitly.

## License

Course materials are © Udacity, Inc. and carry their original licenses — see each project's
`LICENSE.md`. The implementation work, evidence, and notes added here are mine.
