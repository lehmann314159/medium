# How Ratchet Works: A Technical Overview

## The Big Picture

Ratchet is a fully autonomous, local coding pipeline. You give it a design document; it produces working, tested code. The whole thing runs on consumer hardware using Ollama-served LLMs — no cloud APIs, no human in the loop except when the pipeline gets genuinely stuck.

The central abstraction is the **bead**: a scoped unit of work with a fixed title, a set of output files it's allowed to write, and an exit criterion (a shell command that must pass, like `go test -v . -run=TestApplyMove`). Beads are produced by decomposing the design doc, then executed sequentially in order. The pipeline doesn't move to the next bead until the current one passes its exit criterion.

Each bead moves through an eight-verb FSM. The key insight is that this FSM isn't just a wrapper around a single model call — it mixes mechanical analysis, structured model calls, and a compress/summarize loop to manage context across many attempts.

---

## The Decomposition Phase (runs once; produces all bead specs)

**DECOMPOSE** runs once per project. A strong reasoning model (`qwen3:32b`) reads the entire design doc and produces a JSON array of bead specs. The bead count emerges from this call — it's not predetermined. Each spec includes: a title, the list of output files the model is allowed to write, an exit criterion, an execution budget in seconds, and a monitor override setting. The prompt tells the model to search the whole document for function signatures — not just a formal API section — and to produce exact Go `var _ func(...)` compile-time assertion lines for the layout bead.

A design doc can strongly constrain the outcome: if it contains a `## Decomposition Notes` section with an explicit bead table, DECOMPOSE essentially transcribes it and AUDIT typically finds nothing to flag. But the pipeline has no concept of "the spec says N beads" — the notes are hints, not a schema. An AUDIT pass with zero findings is a quality signal for the design doc, not a sign the pipeline skipped work.

**AUDIT** runs next. A second model (`gemma4:31b`) independently reviews the decomposition for structural problems: test beads with no test files in `output_files`, incorrect exit criteria syntax, independence violations between beads. AUDIT emits a structured list of findings.

**RECONCILE** runs only if AUDIT found issues. The same model that did DECOMPOSE looks at each finding in turn and produces an `updated_bead` — a revised spec for just that bead. This is a subtle design: RECONCILE sees one finding at a time and emits one updated bead, not a full re-decomposition.

**Mechanical fixup at commit time** *(Go-specific)*: After RECONCILE (or AUDIT if no findings), Ratchet runs a deterministic fixup pass over every bead spec. If a bead has a `go test -run=TestFoo` exit criterion but no `*_test.go` in its output files, it adds the derived test file name (e.g. `game.go` → `game_test.go`) to `output_files`. Only if there are no `.go` files at all (a content-only bead, like HTML templates) does it downgrade the criterion to `go build ./...`. This prevents a recurring failure mode where the executor writes the implementation but ADJUDICATE can't verify anything because the test was never written.

---

## The Execution Loop (runs once per bead, up to N attempts)

Once the decomposition is committed, beads execute sequentially. Each bead goes through its own loop of up to N attempts (configurable per project).

**EXECUTE** is the live coding step. A `gemma4:31b` model runs in a tool-use loop with three tools: `read_file`, `run_command`, and `write_file`. It reads the project directory and existing source files, writes code, runs tests, and iterates. All tool calls are logged turn-by-turn to a trace file. Meanwhile, a concurrent **MONITOR** process (`mistral-small3.2:24b`) reads the growing trace every 30 seconds and fires SIGTERM if it detects the model is stuck — defined as repeated identical tool calls or more than ~10 turns with no `write_file`. An **orientation-phase exception** prevents false fires: if no `write_file` has appeared yet, repeated reads are treated as expected preparation, not recurrence. This exception was discovered empirically — four of five beads in an early project had at least one attempt killed by the monitor before the first write appeared. Trace data showed the first `write_file` appearing between turns 6–9 in successful attempts, so the threshold of 10 turns was chosen to cover the observed range while still catching genuinely stuck models at 19–28 turns.

**ANALYZE** runs after each execution, whether it succeeded or timed out. A reasoning model (`qwen3:32b`) reads the trace and produces structured findings: what the model did, what the test output said, whether the exit criterion passed, and what might have gone wrong. ANALYZE also runs mechanical pre-checks before calling the model *(Go-specific)*: it parses the trace for `write_file` calls, verifies the compile-time assertion file is present and has `var _` lines using a Go AST parser, and injects those findings mechanically.

**COMPRESS** condenses the history of all previous attempts into a rolling summary. On the first two attempts it's a passthrough — no model call, raw ANALYZE output is used directly. The model call only starts when the history is long enough to justify summarization. On attempt 3+, `mistral-small3.2:24b` reads the current compressed history and the new ANALYZE findings and produces an updated summary with `[NEW]`/`[RECURRING]`/`[RESOLVED]` tags on each observation. This gives ADJUDICATE a concise picture of what has and hasn't worked across all attempts.

**ADJUDICATE** makes the go/no-go decision. `gemma4:31b` sees: (1) the bead spec with actual execution budget, (2) the ANALYZE findings from the last attempt, (3) the compressed history. It produces a JSON verdict: `trend` (improving/declining/stalled), `bead_spec_fit` (does the implementation match the spec), `decision` (one of: `declare_success`, `execute_revised`, `declare_escalation`), and an `execute_revised` bead spec if retrying. The revised spec may include stronger hints via a **specificity ratchet** that escalates across attempts: "here's the general approach" → "here's a skeleton implementation" → "here's verbatim code, copy it exactly." ADJUDICATE's output passes through the same mechanical fixup as decomposition before committing.

Two important principles encoded in the ADJUDICATE prompt:
- **Vacuous-pass principle**: if the exit criterion is `go test -run=TestFoo` but no test file was written, don't declare success — the test didn't run, it just didn't fail.
- **Workspace repair principle**: if prior attempts left broken files (e.g., a partial implementation that doesn't compile), instruct the next attempt to fix or replace them before adding new code.

If ADJUDICATE issues `declare_escalation` or the attempt cap is hit, the bead enters `escalated` state and the project pauses for human intervention.

---

## Prompt Engineering vs. Mechanical Steps

The split is clean and deliberate: **the prompt engineering is largely language-agnostic; the mechanical layer is where language-specific knowledge lives.**

**Language-agnostic:**
- All model calls: DECOMPOSE, AUDIT, RECONCILE, ANALYZE, COMPRESS, ADJUDICATE
- COMPRESS passthrough on attempts 1–2
- Monitor orientation-phase exception
- ADJUDICATE vacuous-pass principle and workspace repair principle
- Specificity ratchet
- Post-execution reports
- The attempt cap, budget enforcement, FSM transitions

**Go-specific (mechanical layer):**
- `applyMechanicalBeadFixes`: detects `go test -run=TestFoo` without `*_test.go` in output files; derives the test file name from the source file name; falls back to `go build ./...` for non-Go beads
- `api_check_test.go` with `var _ func(...) = FuncName` compile-time assertions
- ANALYZE pre-check: Go AST parser to verify the assertion file has `var _` lines
- `detectLang()`: checks for `go.mod` on disk; falls back to scanning `output_files` extensions at decompose time (when `go.mod` doesn't exist yet)
- Language guidance injected into the EXECUTE prompt

Adding Python support would mean implementing a `detectLang` extension, a `pytest`-equivalent exit criterion fixer, and replacing the AST check — the model calls wouldn't change at all.

---

## Three Rules That Took Failures to Discover

Some of the most important mechanical rules only became visible as failure modes:

1. **Test file missing from `output_files`**: The model writes a correct implementation, but because the test file wasn't in its write-permission list, it doesn't write one. ADJUDICATE can't see any test run. The pipeline wastes multiple attempts on already-correct code, each time failing vacuously. Fix: add the test file to `output_files` at commit time.

2. **`execution_budget=0` in the spec JSON**: DECOMPOSE writes `execution_budget: 0` meaning "use the project default." ADJUDICATE reads the raw JSON and reasons "the budget was zero, that's why it timed out." Fix: prepend "Actual execution budget: Ns" to ADJUDICATE's input before the model call.

3. **Cryptic `write_file` error for missing path**: Without a `path` argument, the tool resolved to the project directory and returned "open /path/to/project: is a directory." The model would run `pwd` and try to diagnose the error. Fix: return "write_file requires a 'path' argument specifying the filename (e.g. path="game.go")." The model self-corrects on the next turn.

---

## The Attempt-Cap Bypass on Server Restart

One known gap: when the server restarts while an `EXECUTE_BEAD` job is running, the orchestrator resets it to `failed_retry` and re-queues the execution directly — bypassing ANALYZE → ADJUDICATE entirely. The attempt cap check only lives in ADJUDICATE, so restart-killed attempts accumulate without triggering it. One bead accumulated 9 executions against a cap of 7 because of two server restart cycles during development. The design intention is that ADJUDICATE controls all retry decisions; the restart path is a maintenance shortcut that skips it.

---

## Post-Execution Reports

Each bead at terminal state (succeeded, escalated, full_stopped) writes `traces/bead-{id}-report.md` automatically — no model call, purely mechanical. The report contains: every spec revision, an attempt table (write_file ok/total, last test result), ADJUDICATE decisions with actual budgets, compressed history, full output file contents, and a 60-line trace tail. This makes escalation diagnosis fast: instead of grepping multiple log files, you read one document.

---

## Model Fleet

| Role | Model | Notes |
|---|---|---|
| DECOMPOSE + RECONCILE | qwen3:32b | Strong structured JSON, natural-language reasoning |
| AUDIT | gemma4:31b | Fast cross-checker |
| EXECUTE | gemma4:31b | Tool-use loop; reads 3–6 files before first write |
| MONITOR | mistral-small3.2:24b | Fast inference needed for 30s ticks |
| ANALYZE | qwen3:32b | Hedged interpretation |
| COMPRESS | mistral-small3.2:24b | [NEW/RECURRING/RESOLVED] tagging |
| ADJUDICATE | gemma4:31b | Binary decision + revised spec on retry |

All three models run resident on a single machine (Ollama, `OLLAMA_KEEP_ALIVE=-1`, 55 GB total VRAM). No cloud API calls.
