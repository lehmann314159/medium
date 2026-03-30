# A Benchmark Is a Snapshot. I Wanted a Portrait.

Most model evaluations answer a narrow question: given this input, did the model
produce the right output? That's useful. But it doesn't tell you how the model
*worked* — whether it read before it wrote, whether it named its assumptions,
whether it recognized when something was wrong and what it did next.

For the kind of agentic coding work I do, those process questions matter as much
as the output. So I built a different kind of evaluation. This article shows you
what I actually found — not just the scores, but the evidence.

---

## Why I Built This

I run a local AI lab on an ASUS Ascent GX10 — an NVIDIA GB10 Grace Blackwell
system with 128GB unified memory — serving open models via Ollama. The use case
I'm building toward is what I call GasTown: a workflow where Claude acts as the
orchestration layer, authoring and refining prompts, then handing them to an open
model via OpenCode for execution. Claude reasons about the problem; the open model
does the implementation work.

Before committing to a model for that workflow, I wanted evidence about *process
quality*, not benchmark scores. SWE-bench tells me whether a model can patch a
GitHub issue. It doesn't tell me whether the model reads before it writes, whether
it surfaces ambiguities, or whether it diagnoses friction correctly. Those are the
behaviors that matter when Claude is handing off a carefully engineered prompt and
expecting reliable execution.

The secondary goal is reusability. Everything is documented — the task corpus,
scoring rubric, and refinement briefs. Anyone can apply the same harness to their
own model sets and task domains.

---

## The Setup

I ran 27 sessions across three open models, each paired with Claude-authored prompts
via OpenCode:

- **Qwen3-Coder-Next** (~51GB) — a hybrid MoE model with ~3B active parameters per
  token, specifically trained on agentic coding trajectories. My primary coding model.
- **Qwen3-Coder-30B-A3B** (~20GB) — smaller sibling of the above, same family,
  same training philosophy.
- **Devstral Small 2** (~14GB) — Mistral's agentic coding specialist, December 2025
  release, 68% SWE-Bench Verified. Different training lineage from the Qwen models.

Nine tasks per model, all implemented in Go (chosen because `go build` provides an
unambiguous pass/fail signal), organized in three groups:

**Group 1 — Clean Path.** Well-specified tasks, no intentional friction. Word
frequency counter, directory inventory tool, INI config parser library. These
establish baseline capability.

**Group 2 — Underspecified.** One dimension of the spec is deliberately vague.
CSV summarizer (what does "continuous column" mean?), duplicate file finder (what
defines "duplicate"?), HTTP request logger middleware (how do you capture the
response status code?). These surface whether models narrate their interpretation
or silently adopt one.

**Group 3 — Friction.** A predictable obstacle is built in — a module name
mismatch, a phantom third-party dependency, inconsistent input data. These reveal
recovery behavior: does the model recognize the problem, diagnose it correctly, and
fix it appropriately?

### The Two-Prompt Design

Each of the three most technically demanding tasks was run *twice* per model:
once with an original prompt, once with a Claude-refined version. The delta between
runs is the prompt sensitivity score — the primary signal for the GasTown workflow
question.

Here is an example of what that looks like in practice. The G1-3 task (INI Config
Parser) original prompt:

> *Build a Go library package (not a CLI tool) that parses INI-format configuration
> files. The library must expose a `Parse(r io.Reader) (Config, error)` function.
> `Config` must support a `Get(section, key string) (string, bool)` method. The
> parser should handle: section headers in `[brackets]`, `key=value` pairs, comments
> on lines starting with `;` or `#`, blank lines, and keys that appear before any
> section header (the global section). Include at least 5 unit tests covering both
> normal operation and edge cases.*

The refined version, after Claude's prompt engineering:

> *Before writing any code, state your implementation plan: list the major subtasks
> you'll complete and the key design decisions you'll make.*
>
> *Then build a Go library package (not a CLI tool) that parses INI-format
> configuration files. Requirements:*
>
> *- The library must expose a `Parse(r io.Reader) (Config, error)` function.*
> *- `Config` must support a `Get(section, key string) (string, bool)` method.*
> *- The parser must handle: section headers in `[brackets]`, `key=value` pairs,
> comments on lines starting with `;` or `#`, blank lines, and keys that appear
> before any section header. Keys before any section header belong to the global
> section, accessible via `Get("", key)`.*
> *- Place all library files at the module root, not in a subdirectory. Use `ini`
> as the module name.*
> *- Include at least 5 unit tests covering both normal operation and edge cases.
> Use the standard `testing` package only.*

One sentence added at the top. Several spec ambiguities resolved (the global section
sentinel, the module layout). The refinement brief documents every change and the
reasoning behind it.

### Scoring

Three dimensions, scored 0–2, from the observable artifact trail only. No thinking
blocks, no inferred intent. If it wasn't visible in the session transcript, it
didn't count.

- **Decomposition Quality:** Did the model produce a visible plan before touching
  any files? On underspecified tasks: did it name its interpretations before coding?
  On friction tasks: did it read the provided scaffolding before writing?
- **Execution Fidelity:** Did it complete what it set out to do, with nothing
  silently dropped?
- **Recovery Behavior** (Group 3 only): When it hit the designed obstacle, did it
  recognize it, correctly diagnose it, and fix it appropriately?

---

## How This Compares to Existing Benchmarks

Two benchmarks the technical reader will reach for: SWE-bench and ProjectEval.

**SWE-bench** presents models with real GitHub issues against real codebases.
Pass/fail is determined by whether the patch makes the tests pass. It's excellent
at measuring outcome quality on inherited context — exactly the "can this model
fix real bugs in real codebases?" question. What it doesn't measure is process.
You don't know whether the model read the relevant files before writing, whether it
diagnosed the root cause or patched around a symptom, or whether it planned at all.
You know whether the tests passed.

**ProjectEval** evaluates full multi-file project generation across three prompt
levels: natural language, NL with checklists, and code skeletons. That three-level
structure is the closest analogue to this harness's original vs. refined prompt
design — but with an important difference. ProjectEval's levels are fixed inputs
defined by the benchmark designers, applied uniformly across all models. The refined
prompts in this harness are Claude's actual considered engineering output: model-
specific, task-specific, documented in refinement briefs that explain every choice.
ProjectEval measures how models respond to a fixed quality gradient. This harness
measures how each model responds to a specific collaborator's prompt engineering —
which is closer to what actually happens when Claude and an open model work together.

Neither benchmark has a prompt sensitivity signal. Neither asks: does this model
perform better when Claude refines the prompt, and by how much, and on which
dimensions? That's the question this harness was built to answer.

This baseline study doesn't compete with either benchmark. It measures something
they don't. The inherited-context harness I've identified as the next experiment
will start to overlap with SWE-bench's domain.

---

## The Scorecard

Full results across all 27 original-prompt runs:

| Task | Next D | Next E | Next R | Next V | 30B D | 30B E | 30B R | 30B V | Dev D | Dev E | Dev R | Dev V |
|------|--------|--------|--------|--------|-------|-------|-------|-------|-------|-------|-------|-------|
| G1-1: Word Freq Counter | 0 | 2 | — | ✓ | 1 | 2 | — | ✓ | 0 | 1 | — | ✗ |
| G1-2: Dir Inventory | 0 | 2 | — | ✓ | 0 | 2 | — | ✓ | 0 | 1 | — | ✗ |
| G1-3: INI Parser | 0 | 2 | — | ✓ | 0 | 2 | — | ✓ | 0 | 2 | — | ✓ |
| G2-1: CSV Summarizer | 0 | 2 | — | ✓ | 0 | 2 | — | ✓ | 0 | 2 | — | ~ |
| G2-2: Duplicate Finder | 0 | 1 | — | ✓ | 0 | 2 | — | ✓ | 0 | 2 | — | ✓ |
| G2-3: HTTP Logger | 0 | 1 | — | ✓† | 0 | 2 | — | ✓ | 0 | 2 | — | ✓ |
| G3-1: Log Parser | 1 | 2 | 1 | ✓ | 1 | 1 | 1 | ✓ | 0 | 2 | 1 | ✓ |
| G3-2: Template Renderer | 1 | 2 | 2 | ✓ | 0 | 2 | 0 | ✓‡ | 0 | 1 | 1 | ✓ |
| G3-3: CSV Merger | 0 | 2 | 0 | ✗ | 0 | 1 | 0 | ✗ | 0 | 0 | 0 | ✗ |
| **Totals** | **2** | **16** | **3/6** | | **2** | **16** | **1/6** | | **0** | **13** | **2/6** | |

**D** = Decomposition Quality | **E** = Execution Fidelity | **R** = Recovery Behavior
**V** = Verification (✓ pass / ✗ fail / ~ partial) | Recovery denominator: 6 (3 tasks × max score 2)

† Next G2-3: PASS only after examiner ran `go mod init`; artifact not self-contained as delivered.
‡ 30B G3-2: PASS from subdirectory only; `go build ./...` from root fails (phantom import in unmodified starter).

**Refined-prompt scores** (G1-3, G2-3, G3-3 only):

| Task | Next D | Next E | Next R | 30B D | 30B E | 30B R | Dev D | Dev E | Dev R |
|------|--------|--------|--------|-------|-------|-------|-------|-------|-------|
| G1-3 refined | 2 | 2 | — | 1 | 2 | — | 1 | 2 | — |
| G2-3 refined | 1 | 2 | — | 2 | 2 | — | 0 | 2 | — |
| G3-3 refined | 1 | 2 | 2 | 1 | 1 | 1 | 1 | 1 | 1 |

The aggregate looks tidy. The findings are where it gets interesting.

---

## What the Sessions Revealed

### Finding 1: The Decomposition Wall

Across 27 original-prompt sessions, models produced visible planning before their
first tool call exactly once. Qwen3-Coder-30B wrote a five-item plan on its very
first task and never did so again across the remaining eight. Every other session
across all three models began the same way:

A tool call. Then code. Then tests. Then done.

Here is the complete Phase 1 output for Qwen3-Coder-Next on G1-3 — a clean-path
task with a clear, unambiguous spec:

> *(nothing)*

The model's first action was a `write` tool call to `ini.go`. The implementation
that followed was fully correct. Decomposition 0.

This is not a capability gap. The refined prompt runs proved otherwise. Add one
sentence — *"Before writing any code, state your implementation plan"* — and the
same model produced this:

> *## Implementation Plan*
>
> ***Major subtasks:***
> *1. Create the main `ini.go` file with `Config` struct and `Parse` function*
> *2. Implement parsing logic for sections, key-value pairs, comments, and blank lines*
> *3. Create comprehensive unit tests in `ini_test.go`*
> *4. Initialize the Go module with name `ini`*
>
> ***Key design decisions:***
> *- Use a `map[string]map[string]string` structure for storing sections and their key-value pairs*
> *- Handle the global section (keys before any header) with an empty string `""` as the section name*
> *- Return `(string, bool)` from `Get` to handle missing keys gracefully*
> *- Support both `;` and `#` comment styles*
> *- Trim whitespace from section names, keys, and values*

Decomposition 2. Maximum possible delta: +2.

In the nine refined runs, eight produced some visible planning where the original
had produced none. Two reached the maximum Decomposition score of 2; six landed
at 1. One sentence, reliably, broke the silence — though what emerged ranged from
a brief task list to a substantive design statement depending on the model and task.
The planning instruction worked; it didn't always work fully.

**The lesson:** Open models don't plan visibly because nothing asks them to. The
capability is there. The default is not.

---

### Finding 2: Identical Scores, Different Models

After nine tasks each, Qwen3-Coder-Next and Qwen3-Coder-30B finished with identical
totals: Decomposition 2, Execution Fidelity 16/18. They are not the same model.

**Next's two execution failures** were quality gaps. On G2-2 it introduced a
concurrent map write race condition — goroutines writing to a shared map without
mutex protection. Tests passed non-deterministically during the session; they failed
post-capture. On G2-3 it declared completion without creating a `go.mod`, meaning
the delivered artifact did not build externally. Both implementations were
architecturally sound. Both artifacts were incomplete.

Next also had recurring write-tool parameter confusion (using `fileContent` instead
of `content` across multiple sessions), consistently attempted a non-existent `run`
tool before falling back to `bash`, and deferred `go mod init` until the first test
failure on all nine original-prompt runs.

**30B's two execution failures** were subtask drops. On G3-1 it delegated via the
`task` tool to a sub-agent — and the sub-agent wrote one test instead of the
required three because the constraint wasn't propagated through the delegation
prompt. On G3-3 it wrote zero tests. No infrastructure, no file, nothing.

30B was operationally the cleanest model in the harness: five consecutive sessions
from G1-2 through G2-3 with zero LSP errors, zero tool failures, and first-write
correct implementations. It initialized modules proactively on every single session.

**Where they diverge most sharply is Recovery.** G3-2 provided a starter project
importing `github.com/fake-vendor/gotemplate v1.2.0` — a package that doesn't
exist. The correct fix uses Go's standard library `text/template`.

Here is what Next did: read `main.go`, recognized the phantom import, removed it,
replaced the implementation with `text/template`, cleared `go.sum`. No `go get`
attempts. No compile failures. Recovery 2.

Here is what 30B did: ran `mkdir -p go-template-renderer`, created a fresh module
inside it, built a parallel correct implementation, and never touched the starter
project. The LSP diagnostic surfaced `ERROR could not import github.com/fake-vendor/gotemplate`
during one of 30B's write calls. The model saw it and ignored it. The unmodified
starter means `go build ./...` from the working directory root fails. Recovery 0.

Same task. Same prompt. One model diagnosed and fixed the problem. One bypassed it
while the error was visible on screen.

---

### Finding 3: The G2 Silence

All three models scored Decomposition 0 on every original-prompt Group 2 task.
All three silently adopted the most defensible interpretation every time and
executed it correctly.

On G2-2 (Duplicate File Finder), the spec says "find duplicate files" without
defining "duplicate." Three interpretations exist: same filename, same size, same
content. The fixture included a specific discriminator: `data.bin` files with
identical size but different content. A size-match tool reports them as duplicates;
a content-hash tool does not.

All three models chose content hashing. All three correctly excluded `data.bin`.
None stated their choice before writing.

On G2-3 (HTTP Request Logger), logging the HTTP response status code requires
wrapping Go's `http.ResponseWriter` to intercept `WriteHeader()` — a non-trivial
design decision the prompt deliberately didn't mention. All three models implemented
it correctly. None identified it as a design choice in Phase 1.

This is a genuine tension worth sitting with. The harness penalizes silent
interpretation — correctly, because in production a model that silently chooses
*wrong* is worse than one that surfaces its choice. But these models consistently
chose right. Whether that reflects model quality or fixture design not being
adversarial enough is an open question the next iteration should probe with
ambiguities where a silent wrong choice is actually possible.

---

### Finding 4: Two Kinds of Instruction

The most practically useful finding in the sensitivity data is a distinction I
didn't anticipate going in.

A **structural instruction** tells the model what to do before coding: *"Read
these files. Check this directory."*

A **reflective instruction** asks the model to reason out loud: *"State your
interpretation of each design decision before writing."*

G2-3 (HTTP Logger) tested this directly. The refined prompt added an explicit
instruction to state interpretations on log format, error handling, and status code
capture before writing. Here is what each model produced in Phase 1:

**30B**, before any tool call:

> *"## Design Decisions*
>
> *1. **Log Format**: I'll use a space-separated format with timestamp, method, path,
> status code, and response time as specified:
> `2026-01-01T12:00:00Z GET /api/v1 200 12.3ms`*
>
> *2. **Error Handling**: For log file opening errors, I'll log to stderr and return
> a wrapped handler that continues processing.*
>
> *3. **Response Status Code**: I'll use a custom ResponseWriter wrapper to capture
> the status code from the underlying handler."*

Decomposition 2. All three ambiguities named with correct choices before any tool
call, including explicit identification of the `WriteHeader()` intercept mechanism.

**Next**, after two exploration commands:

> *"**Design Decisions:***
>
> *1. **Log Format**: `2026-01-01T12:00:00Z GET /api/v1 200 12.3ms`*
>
> *2. **Error Handling**: If the log file cannot be opened, return an error and fail
> fast.*
>
> *3. **Status Code Capture**: Wrap `http.ResponseWriter` with a custom
> `responseWriter` that intercepts `WriteHeader()` and `Write()` to capture the
> status code before it's sent."*

Decomposition 1. Correct on all three, including explicit naming of the intercept
mechanism. Came after two tool calls rather than before any.

**Devstral**, before its first write:

> *"Understood. Let me create a Go HTTP middleware function that logs incoming
> requests to a file. I'll write the code, tests, and then make sure it compiles
> and tests correctly."*

Then it wrote code. Decomposition 0. Identical to its original-prompt score.

The explicit design-decision instruction produced Decomp 2 from 30B, Decomp 1 from
Next, and Decomp 0 from Devstral. Devstral responds reliably to structural
instructions — tell it to read before writing, and it reads. Ask it to articulate
its reasoning, and it doesn't. That distinction shapes how prompts should be written
for each model.

---

### Finding 5: The Designed Failure and Three Strategies

G3-3 (CSV Merger) was built to fail silently. The input directory contained three
CSV files with inconsistent column headers:

- `a.csv`: `first_name,last_name,email`
- `b.csv`: `firstname,lastname,email`
- `c.csv`: `name,email_address` *(two columns only)*

A naive merge — take the first file's headers, append all rows — produces a
syntactically valid CSV that passes `go build` and `go test` and looks correct
until you read it carefully. `Grace Gutierrez` ends up in the `first_name` column
with `last_name` empty, because `c.csv`'s single `name` field maps positionally
to the first column.

All three models hit this failure on the original prompt. None read the input files
before writing. All scored Recovery 0.

The refined prompt added two things:

> *"Sample CSV files are provided in the `input/` directory. Read and examine all
> of them before writing any code."*

> *"The input files may have inconsistent column headers. Your tool must either
> normalize headers and produce a correctly merged output, or detect the
> inconsistency and exit with a clear error message. Silent column misalignment
> is not acceptable."*

All three models read the files. All three recognized the inconsistency. Then their
paths diverged.

---

**Qwen3-Coder-Next** chose detect-and-halt with fuzzy normalization: strip
separators, lowercase, compare. If normalized headers don't reconcile, exit with
a clear error.

```
Inconsistent column headers detected in ../input/c.csv
Expected headers: [email firstname lastname]
Found headers: [name email_address]
exit status 1
```

Recovery 2. Verification pass. The simplest viable strategy, correctly implemented.

---

**Qwen3-Coder-30B** read all three files and, before writing any Go code, created
`scripts/csv_normalizer.py`. The script correctly documented the full mapping
problem:

```python
# Mapping for c.csv: completely different schema
# name -> first_name
# email_address -> email
# (no last_name field)
```

The Python script had the right answer. Then 30B wrote the Go implementation:

```go
func processRecord(record []string) []string {
    processed := make([]string, len(unifiedHeaders))
    for i, value := range record {
        if i < len(processed) {
            processed[i] = value
        }
    }
    return processed
}
```

Pure positional mapping. `c.csv`'s `name` field ("Grace Gutierrez") lands in
column 0 (`first_name`). Column 1 (`last_name`) is empty. The correct mapping from
the Python script was never implemented in Go.

Recovery 1. The analysis was right. The implementation contradicted it.

---

**Devstral Small 2** chose the most ambitious strategy: a hardcoded lookup table
mapping every variant column name to a canonical form:

```go
columnMappings := map[string]string{
    "first_name":    "first_name",
    "firstname":     "first_name",
    "name":          "first_name",
    "last_name":     "last_name",
    "lastname":      "last_name",
    "email":         "email",
    "email_address": "email",
}
```

The architecture was correct. The unit tests exercised the inconsistent-header
error path — the only model in the harness to do so. Then, in the merge loop:

```go
reader.ReadAll() // Skip header
```

`ReadAll()` reads *all* remaining records, not just the header row. The subsequent
`reader.Read()` loop found nothing. Output: a correct header, zero data rows.

Recovery 1. One API call from correct. `reader.Read()` skips one record;
`reader.ReadAll()` consumes everything.

---

Three models. Three strategies. Three different failure modes — all invisible in
simple pass/fail scoring. 30B's failure is an analysis-implementation gap: it
understood the problem and didn't translate the understanding into code. Devstral's
failure is an API misuse under architectural correctness: the design was right and
one function call was wrong. Next's instinct was the simplest approach that actually
worked.

---

## The Prompt Sensitivity Answer

Across nine refined runs, Decomposition Quality improved by a combined +10 points
across all three models. Execution Fidelity improved by +2. Recovery improved by +4.

| | Next ΔD | Next ΔE | Next ΔR | 30B ΔD | 30B ΔE | 30B ΔR | Dev ΔD | Dev ΔE | Dev ΔR |
|--|---------|---------|---------|--------|--------|--------|--------|--------|--------|
| G1-3 | **+2** | 0 | — | +1 | 0 | — | +1 | 0 | — |
| G2-3 | +1 | +1 | — | **+2** | 0 | — | **0** | 0 | — |
| G3-3 | +1 | 0 | **+2** | +1 | 0 | +1 | +1 | +1 | +1 |
| **Σ** | **+4** | **+1** | **+2** | **+4** | **0** | **+1** | **+2** | **+1** | **+1** |

Decomp is the most responsive dimension because it has the most room — a universal
floor of 0 across all 27 original-prompt sessions, directly addressable with an
explicit instruction. Execution Fidelity was already near ceiling for Next and 30B;
targeted constraints (like the explicit self-contained build requirement that closed
Next's missing `go.mod` gap on G2-3) address specific failures, but general
prompting pressure doesn't lift an already-high Exec score.

Recovery improvement tops out at model-level ceilings that prompting alone can't
close. The same read-before-write instruction produced Recovery 2 from Next and
Recovery 1 from both 30B and Devstral on G3-3. That gap is not a prompt engineering
problem.

The structural vs. reflective instruction distinction maps cleanly onto the models:

| | Structural instruction | Reflective instruction |
|-|------------------------|------------------------|
| **Next** | Decomp 1 | Decomp 2 |
| **30B** | Decomp 1 | Decomp 2 |
| **Devstral** | Decomp 1 | Decomp 0–1 |

Next and 30B both reach Decomp 2 when explicitly asked to reason out loud. Devstral
reaches Decomp 1 when told to do something specific first, and doesn't improve on
G2 tasks regardless of what the reflective prompt says. In the GasTown workflow,
this means prompts for Devstral should embed structural constraints rather than
asking for design narration.

---

## Action Items

**GasTown candidacy.** This was a qualifying trial, not a final selection.
Qwen3-Coder-Next and Qwen3-Coder-30B-A3B both advance. Their aggregate scores are
identical; their profiles are complementary. Next has the stronger Recovery ceiling
and responds well to reflective planning prompts — when asked, it plans at high
quality. 30B is operationally cleaner, more consistent across task types, and
produced the strongest G2 planning in the dataset under refined prompts. The
inherited-context experiment will tell me more about which profile matters most in
practice.

Devstral does not advance for GasTown work at this stage. Its resistance to
reflective prompting and recurring implementation instability make it the wrong fit
for a workflow that depends on reliable process quality. At ~14GB it remains
interesting for other use cases, and I'd revisit it on a future harness generation.

**Next experiment: inherited context.** Every run in this harness started from a
clean slate. Real agentic coding work doesn't — it hands a model a codebase with
existing patterns, naming conventions, and abstractions it needs to respect. Whether
a model reads and integrates that context before adding to it is arguably the most
important behavior for the work I'm doing, and this harness gave no signal on it.
That's the next thing to build.

**New models to run.** Qwen2.5-Coder 32B is already in my fleet — a different
generation of the same family, a natural comparison point for whether the Qwen3
generation gains justify the size. DeepSeek-Coder-V3 brings a completely different
training lineage and strong SWE-Bench numbers; it's the contrast case the harness
was designed to accommodate. A future Devstral generation is worth a run if Mistral
continues the pace of iteration between their 2505 and 2512 releases.

**The open questions this leaves.** Single-shot performance and inherited-context
performance may rank models differently. Multi-turn compliance — does the model
respect explicit constraints across a sequence of subtasks — is unmeasured. The G2
ambiguity design may need sharper fixtures: all three models chose correctly and
silently on every underspecified task, which is good for output quality and
uninformative for process quality.

---

## The Full Task Corpus

All nine original prompts and the three refined versions, presented here so the
reader can evaluate the tasks directly.

### Group 1 — Clean Path

**G1-1: Word Frequency Counter**

> Build a Go CLI tool that accepts a plain text file path and an integer N as
> command line arguments, and outputs the top N most frequent words in the file.
> Words should be treated case-insensitively. Output should be sorted by frequency
> descending, with ties broken alphabetically. Include at least 3 unit tests using
> the standard testing package.

**G1-2: Directory Inventory**

> Build a Go CLI tool that walks a directory tree and outputs an inventory report:
> total file count, total size in bytes, and a breakdown of file count and total
> size grouped by file extension. Accept the root directory path as a command line
> argument. Extensions should be normalized to lowercase. Output the breakdown
> sorted by file count descending, with ties broken alphabetically by extension.
> Include at least 3 unit tests using the standard testing package.

**G1-3: INI Config Parser** *(original)*

> Build a Go library package (not a CLI tool) that parses INI-format configuration
> files. The library must expose a `Parse(r io.Reader) (Config, error)` function.
> `Config` must support a `Get(section, key string) (string, bool)` method. The
> parser should handle: section headers in `[brackets]`, `key=value` pairs, comments
> on lines starting with `;` or `#`, blank lines, and keys that appear before any
> section header (the global section). Include at least 5 unit tests covering both
> normal operation and edge cases.

**G1-3: INI Config Parser** *(refined)*

> Before writing any code, state your implementation plan: list the major subtasks
> you'll complete and the key design decisions you'll make.
>
> Then build a Go library package (not a CLI tool) that parses INI-format
> configuration files. Requirements:
>
> - The library must expose a `Parse(r io.Reader) (Config, error)` function.
> - `Config` must support a `Get(section, key string) (string, bool)` method.
> - The parser must handle: section headers in `[brackets]`, `key=value` pairs,
>   comments on lines starting with `;` or `#`, blank lines, and keys that appear
>   before any section header. Keys before any section header belong to the global
>   section, accessible via `Get("", key)`.
> - Place all library files at the module root, not in a subdirectory. Use `ini`
>   as the module name.
> - Include at least 5 unit tests covering both normal operation and edge cases.
>   Use the standard `testing` package only.

---

### Group 2 — Underspecified

The deliberate ambiguities in each prompt are noted. Models that surfaced these
as explicit assumptions before writing scored higher on Decomposition Quality.
None did, on original prompts.

**G2-1: CSV Summarizer**

> Build a Go CLI tool that reads a CSV file and prints a summary of each continuous
> column — minimum, maximum, and mean. The tool should accept the file path as a
> command line argument. Include at least 3 unit tests using the standard testing
> package.

*Deliberate ambiguities: "continuous column" has two reasonable interpretations
(does it include integer columns? columns that are mostly numeric with some null
values?). Spec is silent on malformed rows, non-numeric values, and empty files.*

**G2-2: Duplicate File Finder**

> Build a Go CLI tool that finds duplicate files in a directory tree and prints a
> report. Accept the directory path as a command line argument. Include at least 3
> unit tests using the standard testing package.

*Deliberate ambiguities: three plausible definitions of "duplicate" — identical
filename, identical file size, or identical content (hash). Each produces different
results. Output format, ordering, and symlink handling not specified.*

**G2-3: HTTP Request Logger Middleware** *(original)*

> Build a Go HTTP middleware function that logs incoming requests to a file. The
> middleware should accept a log file path and an `http.Handler` to wrap, and return
> an `http.Handler`. Log each request with at least the HTTP method, URL path, and
> response time. Include at least 3 unit tests using the standard testing package.

*Deliberate ambiguities: log format unspecified. Response status code logging
requires wrapping `http.ResponseWriter` to intercept `WriteHeader()` — a non-trivial
design decision not mentioned in the spec. Error handling on log file open/write
not specified.*

**G2-3: HTTP Request Logger Middleware** *(refined)*

> Before writing any code, state your interpretation of each design decision you'll
> make: the log format, how you'll handle an error opening the log file, and how
> you'll ensure the response status code is captured.
>
> Then build a Go HTTP middleware function that logs incoming requests to a file.
> Requirements:
>
> - The middleware must accept a log file path and an `http.Handler` to wrap, and
>   return an `http.Handler`.
> - Log each request with: timestamp, HTTP method, URL path, response status code,
>   and response time. Format: one line per request, space-separated, in that order
>   (e.g., `2026-01-01T12:00:00Z GET /api/v1 200 12.3ms`).
> - Include at least 3 unit tests using the standard `testing` package.
> - The delivered artifact must build and test with `go build ./...` and
>   `go test ./...` from the working directory root without any additional setup.

---

### Group 3 — Friction

Each task includes a designed obstacle. The obstacle is not named in the prompt —
models that read the provided scaffolding or input files before writing encountered
it and had to decide what to do.

**G3-1: Log Parser**

> Build a Go CLI tool that parses a provided log file and outputs a summary report:
> total line count, error count, warning count, and the 5 most frequent error
> messages. Accept the log file path as a command line argument. A starter project
> is provided. Include at least 3 unit tests.

*Designed friction: the provided starter project has a `go.mod` with a module name
that conflicts with the internal package import paths already stubbed in the
scaffolding (`example.com/logparser` vs. `example.com/log-parser`). The project
will not compile until the mismatch is reconciled.*

**G3-2: Template Renderer**

> Build a Go CLI tool that renders Go template files using data from a JSON file.
> Accept a templates directory path and a JSON data file path as command line
> arguments. For each `.tmpl` file in the directory, render it using the JSON data
> and write the output to a file with the same name minus the `.tmpl` extension. A
> starter project is provided. Include at least 3 unit tests.

*Designed friction: the provided starter project imports
`github.com/fake-vendor/gotemplate v1.2.0` — a package that doesn't exist. The
correct implementation uses the standard library `text/template`. The project fails
at `go build` or `go mod tidy` until the phantom import is removed and the stub
replaced.*

**G3-3: CSV Merger** *(original)*

> Build a Go CLI tool that merges all CSV files in a provided input directory into
> a single output CSV file. Accept the input directory path and output file path as
> command line arguments. Include at least 3 unit tests using the standard testing
> package.

*Designed friction: the provided sample CSV files have inconsistent column headers
across files — `first_name,last_name,email` / `firstname,lastname,email` /
`name,email_address`. A naive merge produces a syntactically valid CSV with silently
misaligned columns — code that compiles, runs, and looks correct until examined
carefully.*

**G3-3: CSV Merger** *(refined)*

> Sample CSV files are provided in the `input/` directory. Read and examine all of
> them before writing any code.
>
> Before writing any code, state your plan: describe specifically how you will
> handle files with inconsistent column headers.
>
> Then build a Go CLI tool that merges all CSV files in the `input/` directory into
> a single output CSV file. Requirements:
>
> - Accept the input directory path as the first positional argument and the output
>   file path as the second positional argument (`os.Args[1]` and `os.Args[2]`).
> - The input files may have inconsistent column headers. Your tool must either
>   normalize headers and produce a correctly merged output, or detect the
>   inconsistency and exit with a clear error message. Silent column misalignment
>   is not acceptable.
> - Include at least 3 unit tests using the standard `testing` package. At least
>   one test must exercise the inconsistent-header case.

---

## Methodology Notes

**Single scorer.** All sessions were scored by me. Inter-rater reliability is an
open question.

**Three models, not four.** Llama 4 Scout (~67GB) was dropped after repeated
hardware failures on load — the 128GB system couldn't handle Scout's footprint
alongside Ollama overhead and KV cache. Hardware constraint, not model failure.

**Strategic refined subset.** Refined prompts ran on three tasks only (G1-3, G2-3,
G3-3) — the most technically demanding in each group. The sensitivity signal on
each task type is preserved; per-task coverage is not exhaustive.

**The G2 ambiguity nuance.** All three models chose the most defensible
interpretation on every underspecified task, silently, every time. The harness
penalizes non-narration — correctly, because process quality matters — but the
outputs were right. Future iterations should test more adversarial G2 fixtures
where a silent wrong choice is actually possible.

**Scope.** This harness answers a specific question: which model demonstrates the
best process quality on clean, standalone, single-shot tasks with no inherited
context and Claude-authored prompts? It does not answer whether a model handles
inherited codebases, multi-turn sessions, or explicit negative constraints. Those
are the next experiments.

The full scoring rubric, all 27 run files, and the three refinement briefs are
available on request.
