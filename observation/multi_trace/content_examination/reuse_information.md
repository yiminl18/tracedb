# What in a trace helps reuse? (multi_trace)

_Written 2026-11-05. Part of Goal 1 in [`../../goals.md`](../../goals.md). Same question as [`../../harvard_trace/content_examination/reuse_information.md`](../../harvard_trace/content_examination/reuse_information.md), now with full text and many more traces._

## The question

We want to reuse past traces to make a new query cheaper or better. **Which kinds of information inside a trace help a future query, and how often do they actually recur?**

The Harvard note could name the kinds of information but could not measure exact reuse, because Harvard has no text. Here we can. A "reuse" is any computation result that recurs, in any position: a call and its result, a page, a file view, a failure, an answer. A shared prefix is only one case.

## What I looked at

| Source | Traces | Used for |
|---|---|---|
| Sample A: `../<dataset>/traces/` | 800 (8 datasets × 100), spread over tasks | reuse across different tasks |
| Sample B: built by `../../scripts/sample_multi_trace_same_task.py` | 460: DataAgentBench 12 questions × 10 runs, Nebius 10 tasks × 10, OpenResearcher 20 questions × 5 seeds, MATM 20 tasks × 6–8 models | reuse across runs of the same task |
| All DataAgentBench runs in `../_raw/` | 13,452 runs, 54 questions, 12 databases, 84,411 database calls | reuse at full scale |

Scripts in this folder:

- `reuse_scan.py` measures exact call and result repeats, failures, and answer agreement on Samples A and B.
- `dab_scale.py` measures database-call repeats over all DataAgentBench runs.

Both read the common step format from `../../scripts/multi_trace_normalize.py`.

## Short answer

The same eight kinds of information as in the Harvard note, now with measured recurrence:

| # | Information | Seen in the traces | How often it recurs | Risk if reused blindly |
|---|---|---|---|---|
| 1 | **Environment facts** (description, schema, value formats, file layout) | one exact database description per database; schema probes; `view /workspace` | the description: in every question on that database. Probes: 19–79% of probe calls are also made for another question on the same database | low while the environment version holds |
| 2 | **Negative facts** (what fails) | "Error fetching URL" for pages not in the corpus; failing reproduce scripts | for the same task, 52% (Nebius) and 86% (MATM) of failing calls also fail in another run | low, but some failures are temporary |
| 3 | **Intermediate results** (call + its result) | file views, query outputs, search pages | for the same task: 31% (Nebius), 41% (MATM), 12% (DataAgentBench), 11% (OpenResearcher) of calls repeat with an identical result. At full scale, 55% of DataAgentBench database calls | medium: must match the exact call and data version |
| 4 | **Working procedures** (the action path) | the click path in WebArena, query → Python → answer | two successful runs of the same MATM task have action-order similarity 0.85 | medium: element ids and paths change |
| 5 | **Final answers** | `return_answer` text, OpenResearcher "Exact Answer" | low agreement: only 18% (DataAgentBench) and 29% (OpenResearcher) of run pairs on the same question give about the same answer | **high**: answers vary and are often wrong |
| 6 | **Outcome labels** | `resolved`, `success` | present in Nebius, NVIDIA, MATM and AMA-bench only | none, but often missing |
| 7 | **Prompt prefix** | system prompt, tool definitions | 0–13% of trace text, shared by every trace of a harness | none for correctness |
| 8 | **Cost data** | result sizes, LMCache gaps, OpenResearcher latency | sizes in every trace, timing only in LMCache and OpenResearcher | none |

Main findings:

- **Reuse is large within one task and small across tasks.**
  - Across runs of the same task, a large share of tool result characters comes from a call that another run made with an identical result: 60% for DataAgentBench at full scale, 42% for Nebius, 17% for MATM, 10% for OpenResearcher.
  - Across different tasks, even in the same environment, this drops to under 5%.
- **What carries across tasks is environment facts, not results.** The database description and the schema probes recur for every question on a database. A query result rarely does.
- **Run-local handles hide reuse.** Storage keys, call ids and search timestamps make equal results look different. Removing them lifts identical-result repeats a lot:

  | Dataset (same-task sample) | Before | After |
  |---|---|---|
  | DataAgentBench | 1.7% | 12% |
  | OpenResearcher | 3.7% | 11% |

- **Final answers are the worst thing to reuse.** Runs on the same question disagree most of the time.

## Observations by kind of information

### 1. Environment facts

- **The description is fixed per database.** All 42 DataAgentBench questions in Sample A carry one exact description per database, 1.6K to 5.6K characters long. Each run rereads it. It sits after the question, so a prefix cache misses it (see `../cross_traces/`).
- **Schema probes recur across questions.** Over all 13,452 runs, 41% of database calls are probes or listings (`LIMIT n`, `DISTINCT`, `COUNT(`, `list_db`). Per database, 19% (stockmarket) to 79% (agnews) of probe calls are made by runs of **two or more different questions**. These are facts about the database, not about one question.
- **Questions per database bound the gain.** DataAgentBench has 2–13 questions per database, 54 in all. Each question has about 250 runs.
- **Code repair repeats the first look.** `view /workspace` is the most repeated call in Nebius (75 of 100 traces). The same tree view returns for every run on the same repo.

### 2. Negative facts

Failures are found with a text rule (tracebacks, `Error:`, "No such file", "Error fetching URL" and similar). This is an upper bound, because a file view can contain the word "Exception". A stricter count uses exit codes from shell results: 10–17% of shell calls fail in the coding datasets.

- **Within the same task, failures repeat.** In Sample B, a failing call is hit with an error by another run of the same task:

  | Dataset | Share of failing calls |
  |---|---|
  | MATM | 86% |
  | Nebius | 52% |
  | OpenResearcher | 16% |
  | DataAgentBench | 2% |

- **OpenResearcher pages missing from the corpus.** Many opens fail with "Error fetching URL" for Wikipedia pages. One page failed 11 times across the 5 runs of one question. A note saying "not in corpus" would save every later try.
- **DataAgentBench failures are rarely shared.** Its failures are mostly in the agent's own Python (84 `SyntaxError`, 49 `TypeError`), and each run writes different code.

### 3. Intermediate results

Exact match on (tool, whitespace-collapsed arguments), with run-local handles removed.

| Dataset | Same task (Sample B): call repeats / identical result / result chars | Different tasks (Sample A): identical result |
|---|---|---|
| Nebius | 45% / 31% / 42% | 0.4% |
| MATM | 81% / 41% / 17% | 12% (mostly `look`, `scroll`) |
| DataAgentBench | 14% / 12% / 7% | 6% |
| OpenResearcher | 17% / 11% / 10% | 0.1% |
| DataAgentBench, all 13,452 runs | 58% / 55% / 60% | 1% (only calls never made for the same question) |

- **The more runs we keep, the more hits we get.** With 10 runs per question, 12% of DataAgentBench calls repeat with the same result. With about 250 runs per question, it is 55%. A reuse store gains a lot from volume.
- **Repair runs read the same files.** In Nebius, 89% of file-viewer result text is also in another run of the same task.
- **Results with the same text but from a different call are rare.** Only 0.1–4% of results of 200 or more characters, except LMCache, where fixed environment messages inflate it. So keying on the call works almost as well as keying on the result.

### 4. Working procedures

- **WebArena and ALFWorld.** Successful runs of the same task are almost the same path: action-order similarity 0.85, action-mix distance 0.11. Mixed pairs are at 0.24 and 0.61.
- **Code repair.** Successful and failing runs follow the same procedure (similarity 0.50 vs 0.51). The procedure is reusable, but it does not tell whether the fix is right.

### 5. Final answers

- **DataAgentBench.** Only 18% of run pairs on the same question give about the same answer (word-set Jaccard 0.8 or more). One question yields lists of 20 codes that barely overlap from run to run.
- **OpenResearcher.** 29% of run pairs agree. In a rough check, only 23% of answers contain all the words of the reference answer. `status=success` only means the run finished.

So a stored answer needs a correctness label before it is reused.

### 6. Outcome labels

- **Where labels exist.** Nebius, NVIDIA, MATM and AMA-bench carry `resolved` or `success`. DataAgentBench, Thoughtworks, OpenResearcher and LMCache do not in this format.
- **In ALFWorld, the label belongs to the task.** In 8 of 10 tasks, all 6 models succeed or all 6 fail. A label on one run predicts the others.

### 7 and 8. Prefix and cost

- **The system prompt is a small share.** It plus the tool definitions is 0–13% of trace text at the median. Most text is tool results: 41–96%.
- **Most cost is reading.** The model writes 0.2–2.4% of what it reads (see `../use_case/`). So skipping one large repeated result saves far more than skipping the model's own text.

## What is not worth reusing

- **Run-local handles:** storage keys (`var_function-call-…`), tool-call ids, search timestamps (`ts=…`), page cursors (`[3]`, `【12†`), element ids. These are what make equal results look different.
- **Results of the agent's own code.** DataAgentBench Python results are shared only 13% even within one task.
- **Final answers without a label.**
- **Game moves** (`left`, `up`, `look`). They repeat a lot, but each result depends on the game state.

## What this means for a reuse design

1. **Key results by (environment version, tool, canonical arguments).** Strip run-local handles before matching. This one step moves identical-result hits from 2–4% to 11–12%.
2. **Index by task first.** Same-task hits are much higher than cross-task hits: 31% vs 0.4% in Nebius, 11% vs 0.1% in OpenResearcher. For a new task in a known environment, reuse environment facts (description, schema probes, repo tree) instead of results.
3. **Volume matters.** DataAgentBench goes from 12% to 55% identical-result hits when the store grows from 10 to 250 runs per question.
4. **Store failures as notes.** Within one task, half or more of failing calls fail again in other runs (Nebius, MATM).
5. **Keep provenance and labels per fact.** Answers disagree across runs, and success labels are often missing.
6. **Do not rely on the prefix.** The shared environment text often sits after the question, and the most reusable content (tool results) is in the middle of the trace.

## Open questions

- How stale do stored results get? Repair runs edit files, so a file view is only valid until an edit. `../use_case/` counts how often an edited file is read again.
- Can near-duplicate matching (same query in other words, same file with one line changed) add much over exact matching? OpenResearcher searches are a good test, because most searches are new wordings.
- How do we check a stored answer without a reference? The answer disagreement in DataAgentBench and OpenResearcher is a warning.
- Do these patterns hold beyond the first 13 of 60 OpenResearcher files and the 5 small NVIDIA files?

## Method notes

- **Call match.** A call matches another when its tool name and its whitespace-collapsed arguments are equal. Its result matches when the result text is equal after the same normalization. "After removing run-local handles" also replaces storage keys, call ids, timestamps, temp names, long hex ids, clock times and page cursors with a placeholder.
- **Relation.** Each repeat is credited to the closest trace that also made the call, in this order: same task, then same environment, then other.
- **Answer agreement.** Two answers agree when the Jaccard similarity of their word sets is 0.8 or more. This is a rough check.
- **Error rule.** Errors are found with a regular expression, so the rates are upper bounds. Exit codes give a stricter rate for shell calls.
- **Full-scale DataAgentBench.** `dab_scale.py` reads every run file in `../_raw/dataagentbench/all_trajectories/`. It keeps only `query_db` and `list_db` calls. Of the 13,482 runs, 30 had no database call or could not be read.
