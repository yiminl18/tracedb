# Use cases for agent traces (multi_trace)

_Written 2026-11-05. Part of Goal 1 in [`../../goals.md`](../../goals.md). Same question as [`../../harvard_trace/use_case/README.md`](../../harvard_trace/use_case/README.md), now checked on full-text traces._

This note asks: **why is a large set of agent traces useful?** It keeps the 13 use cases and the database lens of the Harvard note. For each one, it says what the full-text data shows, and where it agrees or disagrees with Harvard.

## What I looked at

- **Sample A.** 800 traces in `../<dataset>/traces/`, 100 from each of 8 datasets: data analysis, 3 code-repair sets, web and household tasks, web research, mixed games and tasks, and LMCache sessions.
- **Sample B.** 460 extra traces with many runs of the same task (see `../content_examination/`).
- **All 13,452 DataAgentBench runs**, for the full-scale reuse number.

Scripts:

```bash
python3 -I scripts/multi_trace_normalize.py multi_trace norm.pkl          # from anytrace/
python3 -I multi_trace/use_case/usecase_scan.py norm.pkl
python3 -I multi_trace/content_examination/reuse_scan.py norm.pkl norm_same.pkl
python3 -I multi_trace/content_examination/dab_scale.py multi_trace/_raw/dataagentbench/all_trajectories
```

Sizes are in characters. Divide by about 4 for tokens.

## Numbers at a glance

| What | Value | Harvard value | Source |
|---|---|---|---|
| Calls repeated **with identical result** in another run of the **same task** | 11–41% (Sample B); 55% over all DataAgentBench runs | not measurable | Sample B, full DataAgentBench |
| Same, but only across **different tasks** | under 1%, except MATM 12% and DataAgentBench 6% | not measurable | Sample A |
| Calls that exactly repeat an earlier call in the **same trace** | 1% (DataAgentBench) to 37% (AMA-bench) | 59%, upper bound | Sample A |
| Model output / model input | 0.2–2.4% (about 1 : 40 to 1 : 500) | 0.42% (1 : 240) | Sample A |
| System prompt and tool definitions, share of trace text | median 0–13% | median 13% | Sample A |
| Input messages already sent in the previous request | 95% | 66–88% of blocks | LMCache |
| Traces with 5 or more identical calls in a row | 21% MATM, 18% AMA-bench, 3% LMCache, 0% elsewhere | 37 of 80 | Sample A |
| Calls that fail | 10–17% of shell calls by exit code; 9–29% by error text | 3.7% | Sample A |
| A failure followed by the same tool | 49–85% | 185 retries | Sample A |
| Offered tools that a trace uses | median 80% (Nebius), 100% (NVIDIA) | median 18% | Sample A |
| Repair traces that read a file after editing it | 91% Nebius, 63% NVIDIA, 33% Thoughtworks | paths hidden | Sample A |

Notes:

- **Exact numbers, not upper bounds.** Unlike Harvard, calls are matched on their real text. The in-trace repeat numbers are exact, which is why they are much lower than Harvard's 59%.
- **Benchmark tool sets are small.** They offer 3–10 tools, so "tools used / offered" is high here. Harvard sessions are offered many more tools.
- **The error-text rule overcounts.** It also fires on file views that contain the word "Exception". The exit-code rate is the safer one.

## Use cases

### 1. Reuse of results: materialized views for agents

**Database idea.** Store computed results so later queries can use them.

**What I saw.**
- **Same task: large.** Repeats with identical results, as a share of calls:

  | Dataset | Share of calls |
  |---|---|
  | MATM | 41% |
  | Nebius | 31% |
  | DataAgentBench | 12% |
  | OpenResearcher | 11% |

  These carry 42% of all result characters in Nebius. Over all 13,452 DataAgentBench runs (about 250 per question), 55% of database calls repeat with an identical result. They carry 60% of query result characters.
- **Different tasks: small.** Under 1% in code repair and web research.
- **Run-local handles hide reuse.** Removing storage keys, call ids and timestamps lifts identical-result hits from 1.7% to 12% (DataAgentBench) and from 3.7% to 11% (OpenResearcher).

**What it enables.** A result store keyed by task or environment, with canonical call text. This confirms Harvard's view that reuse is local: per user there, per task here. It also adds an exact size for the opportunity.

### 2. Choosing what to keep: workload-driven view selection

**What I saw.**
- **Probes are 41% of database calls.** Over all DataAgentBench runs, 41% of database calls are schema or value probes. Per database, 19–79% of probe calls are also made for another question. These are the best candidates to keep.
- **The opening tree view repeats.** In code repair, `view /workspace` is the most repeated call (75 of 100 Nebius traces).
- **Reading dominates.** In repair, it is 24–62% of calls.

**What it enables.** Choose which results to keep from the trace log, by how many tasks share them and how large they are.

### 3. Keeping lessons: catalog notes

**What I saw.**
- **One description per database.** Every DataAgentBench question on a database carries the same exact description (1.6K–5.6K characters), and each run reads it again.
- **Failures repeat within a task.** 52% (Nebius) and 86% (MATM) of failing calls fail again in another run of the same task.
- **OpenResearcher retries missing pages.** It retries pages that are not in its corpus, up to 11 times for one page across the runs of one question.

**What it enables.** Short notes per environment ("page X is not in the corpus", "column Y has two formats"). They are cheaper than results and stay valid longer.

### 4. Optimizing agent plans: a query rewriter

**What I saw.**
- **Large results move through files and Python.** 145 of 483 DataAgentBench query results (30%) are too large to show and are stored to a file. Later Python reads 104 of them (72%). The agent pulls whole tables up and filters in Python. Harvard samples showed only 10 of 44 results read by code, so this sample is more code-driven.
- **Most Python failures are in the agent's own glue code:** 84 `SyntaxError` and 49 `TypeError` in Sample B.

**What it enables.** Push filters into the query and return handles instead of text. Python glue code is the main source of failures, so it is also the main place to save work.

### 5. Provenance: which result led to the answer

**What I saw.**
- **Answers disagree across runs.** Only 18% (DataAgentBench) and 29% (OpenResearcher) of run pairs on the same question give about the same answer.
- **"Finished" is not "correct".** OpenResearcher marks every run "success", yet in a rough check only 23% of answers contain all the words of the reference.

**What it enables.** Link each answer to the results it used, so that diverging runs can be compared and the wrong choice found.

### 6. Staleness and conflicts: view maintenance

**What I saw.**
- **Edited files are read again.** 91% of Nebius traces read a file again after editing it (6 such reads per trace on average). NVIDIA: 63%, Thoughtworks: 33%. Any stored view of that file is stale after the edit.
- **Full text gives exact read and write sets.** Harvard could not measure this, because its paths are hidden.

**What it enables.** Invalidate stored results by file. Measure how often a reused result would be stale.

### 7. Caching below the agent: KV cache

**What I saw.**
- **Each request mostly repeats the last one.** In LMCache, 95% of the input messages of a request were already sent in the previous request.
- **The model waits on tools briefly.** The gap before a request is 0.72 s at the median and 6.6 s at p95. The median output is 107 tokens.
- **The prefix is small across traces.** System prompts are 0–13% of trace text. Across runs of the same task, the shared prefix is 4–40% of text, while 50–66% is shared somewhere (see `../cross_traces/`).

**What it enables.** Keep a session's cache warm through short tool gaps. Beyond the prefix, a cache that can reuse a block at any position (for example, the same file view at a new place) would catch much more.

### 8. Stopping runaway sessions: a watchdog

**What I saw.**
- **Loops mark failure in web and game tasks.** The share of traces with 5 or more identical calls in a row:

  | Dataset | Failed runs | Successful runs |
  |---|---|---|
  | MATM | 38% | 9% |
  | AMA-bench | 28% | 7% |

  The longest runs: 100 identical calls (AMA-bench) and 30 (MATM, the step limit).
- **Repair agents do not loop this way.** No Nebius or NVIDIA trace has 5 identical calls in a row. Their waste is different: rereading edited files and rerunning the same test.

**What it enables.** A loop detector for web and game agents, trained on the success labels. Repair agents need a different signal.

### 9. Workload characterization: a TPC for agents

**What I saw.**
- **Session sizes vary a lot.** Median calls per trace: 8 (DataAgentBench), 15 (MATM), 22 (LMCache), 31 (NVIDIA), 35 (Thoughtworks), 41 (OpenResearcher), 50 (AMA-bench), 66 (Nebius). The p95 goes up to 206.
- **Text per trace is large.** The median is 12K characters (MATM) to 230K (Nebius).
- **The workload is read-heavy.** The model writes 1 character for every 40 to 500 it reads.

**What it enables.** Per-application workload profiles. Harvard gives real-use distributions, and these benchmarks give the text behind them.

### 10. Predicting cost and next steps

**What I saw.**
- **Huge results are rare.** Results of 20K characters or more are 0–2.2% of calls. They come mostly from shell output in repair tasks.
- **The phase order is fixed per type** (see `../cross_traces/`, S1). This makes the next action easy to predict.

**What it enables.** Predict result size and next action from the type and phase. Warn before a call that would flood the context.

### 11. Security and audit

**What I saw.** Share of traces using each command:

| Dataset | `rm -rf` | `git reset/checkout/stash/clean` | `pip install` | `curl` / `wget` |
|---|---|---|---|---|
| NVIDIA | 14% | 32% | 10% | 5% |
| Nebius | 5% | 11% | 26% | 2% |
| Thoughtworks | 3% | 2% | 19% | 0% |
| LMCache | 1% | 1% | 53% | 21% |

**What it enables.** Per-harness allowlists. The NVIDIA harnesses use destructive git commands far more than the others.

### 12. Replay and what-if

**What I saw.**
- **Timing exists in two datasets.** LMCache has request timing and OpenResearcher has run latency. The others have only the text.
- **Full text makes counterfactuals possible.** Replace a repeated call with its stored result and count the saved characters. That is how use case 1's numbers were made.

### 13. Post-training

| Need | What the data has |
|---|---|
| Successful traces for SFT | Nebius, NVIDIA, MATM, AMA-bench have labels. About half the runs succeed. |
| Preference pairs (same task, success and failure) | Nebius: all 10 tasks in Sample B have both. MATM WebArena: 6 of 10 tasks. ALFWorld: only 1 of 10, because outcomes depend on the task. |
| Step labels | failures (exit codes), loops, re-reads after edits |
| Same-task variety | DataAgentBench 250 runs per question; Nebius about 10 per task |

**Link to reuse.** A model trained on successful runs learns the same paths that the reuse store would replay. Comparing the two on the same tasks is a fair experiment.

## Which data supports which use case

| Use case | Harvard metadata | multi_trace full text |
|---|---|---|
| 1. Result reuse | sizes the opportunity | **measures it exactly** |
| 2. View selection | call shapes | exact calls and probes |
| 3. Lessons and notes | no | yes |
| 4. Plan rewriting | partly | yes (DataAgentBench) |
| 5. Provenance | no | yes |
| 6. Staleness | no | **yes (read and write sets)** |
| 7. KV cache | yes (block ids) | prefix and in-place sharing; LMCache timing |
| 8. Runaway detection | yes | yes, with labels |
| 9. Benchmarks | real use | per-application text |
| 10. Cost prediction | yes (latency) | sizes only, plus LMCache |
| 11. Security | program names | full commands |
| 12. Replay | caching | agent behaviour |
| 13. Post-training | indirect | labels and pairs |

## Suggestions for what to demonstrate

1. **Same-task result reuse on DataAgentBench (main story).**
   - The setting has 12 databases, 54 questions and about 250 runs per question.
   - Replay runs in time order. Serve a database call from a store keyed by (database, canonical query).
   - Measure the saved calls and characters as the store grows. The early number is 12% of calls (10 runs per question); at full size it is 55%.
   - Then add environment notes for new questions.
2. **Cross-task environment reuse.** Same databases, new questions: reuse probes and the description, not results.
3. **Loop detection for web agents (cheap second demo).** In MATM, failed runs loop 4× as often as successful ones.

## Open points

- **Partial samples.** OpenResearcher covers 13 of 60 files and NVIDIA 5 small files. The numbers for these two may shift.
- **Exact match only.** Near-duplicate results (a file view after a small edit, a search with new wording) are not counted.
- **Thin labels.** Correctness labels are missing for DataAgentBench, OpenResearcher, Thoughtworks and LMCache in this format. Answer checks here are rough word matches.
- **Missing per-user patterns.** These benchmarks have no users. The Harvard finding that reuse lives within one user cannot be checked here. The closest analog is "within one task".
