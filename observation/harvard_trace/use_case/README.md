# Use cases for agent traces

_Written 2026-11-05. Part of Goal 1 in [`../../goals.md`](../../goals.md)._

This note asks one question: **why is a large set of agent traces useful?**
Reuse is the main use case in `goals.md`. Here I look for more, mostly through a database lens. Each use case says what I saw in the sample data, what it would enable, and which data can support it.

## What I looked at

- **80 Harvard sessions** in `../exploration/by_type/<type>/p*/` (10 per agent type). No message text. Tool arguments are anonymized to shapes, such as `cd <path> | python <str>`.
- **18 full-text benchmark traces** in `../exploration/by_type/<type>/sample_traces/`: 6 data-analysis (DataAgentBench), 3 research QA, 3 WebArena browsing, 3 SWE repair.
- **3 full days of the release** (2026-06-09, 2026-07-22, 2026-08-12), to measure sharing *across* sessions.

Scripts, run from `anytrace/`:

```bash
python3 -I scripts/usecase_sample_stats.py harvard_trace/exploration/by_type out.json
zstd -dc harvard_trace/release/<week>/traces/<day>.jsonl.zst | python3 -I scripts/usecase_cross_session.py
```

## Numbers at a glance

| What | Value | Source |
|---|---|---|
| Input blocks already sent earlier in the **same session** | 66–88% | 3 days, all sessions |
| Input blocks already sent in an **earlier session of the same user** | 10–31% | 3 days |
| Input blocks already sent by **another user** | 0.01–0.35% | 3 days |
| Share of a new session's first input already seen in another session | median 72–84% | 3 days |
| Tool calls that repeat an earlier call shape in the same session | 59% (upper bound) | 80 samples |
| Output tokens / input tokens | 0.42% (about 1 : 240) | 80 samples |
| Tool time / LLM time | 0.9 (coding) to 2.7 (mixed) | 80 samples |
| Offered tools that a session ever uses | median 18% | 80 samples |
| Tool definitions as share of prompt tokens | median 13%, p75 29% | 80 samples |
| Sessions with 5 or more identical calls in a row | 37 of 80 | 80 samples |
| Tool calls that fail | 3.7% | 80 samples |

Notes on these numbers:
- A "block" is 16 tokens. Block ids are chained, so two equal ids mean the whole input up to that point is equal.
- The 59% is an upper bound. Anonymized arguments make different calls look the same (two different files are both `<path>`).
- Sessions are rebuilt by the dataset authors. Some "new sessions" may be one conversation split in two, which inflates the same-user number.

## Use cases

### 1. Reuse of results: materialized views for agents

**Database idea.** A materialized view stores a computed result so later queries can use it. The hard parts are: which views to keep, how to match a new query to a view, and when a view goes stale.

**What I saw.**
- **Reuse lives inside one user, not across users.** On each of 3 days, 10–31% of all input blocks were already sent in an earlier session *of the same user*. Only 0.01–0.35% came from other users. This makes reuse work like a per-tenant view, not a global one.
- **A new session starts mostly warm.** Its first input is, at the median, 72–84% the same as an input seen before. This is tool definitions, system prompt and project notes. Harnesses differ a lot: hermes 35–43% cross-session hits, codex 14–18%, opencode under 5%.
- **Exact matching is not enough.** In the football research trace (`web-research-qa/sample_traces/03-football-finalist`), the agent ran 74 searches. 73 have distinct word sets, yet most ask the same thing in new words ("1977 night series Collingwood", "VFL Night Series Wikipedia", "Night Premiership VFL winners"). A cache keyed on exact text would miss almost all of them.
- **Different tasks touch the same data.** Data tasks D2 and D4 both query `stockinfo_database` and `stocktrade_database`. Each one rediscovers the schema and the symbol lookup from scratch.

**What it enables.** A reuse layer that keeps tool results per user or project, and matches new requests by meaning. The database theory of *answering queries using views* (query containment, rewriting) is a good fit. For SQL tools it applies directly. For web search and file reads it needs a fuzzy version.

**Data fit.** Harvard data can size the opportunity (block reuse, call shapes). It cannot check if two results are truly equal, because text is removed. Full-text traces (DataAgentBench) are needed to show real result reuse.

### 2. Choosing what to keep: workload-driven view and index selection

**Database idea.** Index and view advisors (for example AutoAdmin in SQL Server) read a query log and pick which structures to build under a space budget.

**What I saw.**
- **Schema probing repeats in every data task.** 5 of 6 data-analysis traces start with `list_db`, `SELECT * ... LIMIT 5`, `SELECT DISTINCT`, or `COUNT(*)`. These are the same small facts every time.
- **Coding agents reread the same files.** In SWE traces, one file was viewed 4 times (resolved run) and another 15 times (unresolved run, `coding/sample_traces/02-netcdf-variable-names`).
- **Most tool activity is reading.** Top programs in the Harvard samples: `head` 954, `grep` 584, `cat` 183, `ls` 252, versus `Edit` 329 and `Write` 122 calls.

**What it enables.** Use the trace log as a query log. Pick which results to precompute or keep (schema summaries, file contents, common lookups), given a token or storage budget. This is the "view selection" half of use case 1.

### 3. Keeping lessons, not only results: catalog notes and data profiling

**Database idea.** A database keeps statistics and catalog facts so the planner does not rediscover them.

**What I saw.**
- In D1 (`data-analysis/sample_traces/01-asian-index`) a SQL date filter failed. The `Date` column mixes formats: `31 Dec 1986, 00:00` and `January 02, 1987 at 12:00 AM`. The agent then pulled the raw rows into Python to work around it.
- In D5 the first Python call failed with a wrong file path. The agent had to learn how stored results are accessed.
- In D6 a search for "Indianapolis, Indiana" returned nothing, while "Indianapolis, IN" worked.

**What it enables.** Save these facts as short notes per database or tool ("Date has mixed formats", "city is stored as IN"). The next agent reads the note instead of failing first. This is cheaper than storing results, and it stays valid longer.

### 4. Optimizing agent plans: a query rewriter for agents

**Database idea.** Query optimizers push filters and aggregates down to the data, and avoid moving data that is not needed.

**What I saw.**
- **Data pulled up instead of work pushed down.** D1 and D5 fetched whole tables (results over 10K characters, stored to files) and did the filter and aggregation in pandas.
- **Data copied through the model.** In D2, the agent typed a list of 23 symbols by hand into Python code, then built a 23-way `UNION ALL` query in Python. It hit two syntax errors on the way.
- **Only a few results are read by later code.** Across the 6 data traces, 10 of 44 stored results were read by key in a later step. The rest were read only by the model, through its context. This is a lower bound on use.

**What it enables.** Mine traces for these patterns, then give agents better tools (a query tool that returns a handle, not text) or rewrite plans. Compare cost before and after on the same tasks.

### 5. Provenance: which result led to the answer

**Database idea.** Data provenance (lineage) records which inputs produced an output.

**What I saw.**
- The `mapping.md` files for the data traces already link each query to the steps that use its result.
- D6 computed two different answers: 3.86 (average over all reviews) and 3.98 (average of per-business averages). It submitted 3.98. Only the lineage shows which choice was made.
- Many results are never used again (see use case 4).

**What it enables.**
- Explain an answer and spot wrong aggregation choices.
- Drop unused results from context, which shrinks the prompt.
- Know which base data a reused result depends on, so it can be invalidated (use case 6).

### 6. Staleness and conflicts: view maintenance and concurrency control

**Database idea.** A view must be refreshed when its base data changes. Concurrent writers need conflict checks.

**What I saw.**
- Coding sessions mix reads and writes on the same workspace: read, edit, read again. A file read before an edit is stale after it.
- Some sessions run subagents in parallel (`p55-subagents-*` samples). They may read and write the same files.

**What it enables.** Traces give each step's read set and write set. From these one can measure how often a reused result would be stale, and how often parallel agents would conflict.

**Data fit.** Harvard paths are anonymized (`<path>`), so exact read/write sets are not available. This needs full-text traces such as SWE repair.

### 7. Caching below the agent: buffer pool management for the KV cache

This is already in `goals.md`. The new evidence adds two points:
- **Route by user.** Because cross-session reuse stays within one user, sending a user's sessions to the same server is like partition affinity in a distributed database.
- **Prune the tool catalog.** A session uses a median of 18% of the tools it is offered, yet tool definitions are a median of 13% of prompt tokens. It is like a query that scans columns it never uses. Trimming the tool list per agent type would cut that.

### 8. Stopping runaway sessions: a watchdog for agents

**Database idea.** Databases kill runaway queries that exceed a time or cost limit.

**What I saw.**
- 37 of 80 Harvard sessions have 5 or more identical calls in a row. The worst: a copilot session called `task_complete` **105 times in a row** (`mixed-other/p95-copilot-263d0bbb`). Others: `execute_code` 31 times, `webfetch` 20 times, `read` 20 times.
- Failed WebArena runs loop: one clicked the same element 20 times, one switched to tab 0 21 times. Both hit the 30-step limit. The successful run had no repeats.
- Unresolved SWE runs repeat more than the resolved one: 44 of 92 and 23 of 68 calls are exact repeats, versus 15 of 45.
- 3.7% of tool calls fail, and 185 failures are followed by a retry of the same tool.

**What it enables.** Detect loops early, then stop, warn, or switch strategy. Traces give the labels to tune a detector and to estimate the tokens saved. This is one of the cheapest demos, because it works on the Harvard metadata alone.

### 9. Workload characterization and benchmarks: a TPC for agents

**Database idea.** TPC benchmarks capture a typical workload so systems can be compared fairly.

**What I saw.**
- The workload is extremely read-heavy: the model writes about 1 token for every 240 it reads.
- Tool time is about as large as model time (ratio 0.9 to 2.7), so the model is often idle.
- Session sizes have a long tail (coding: p50 67 requests, p95 488).
- The dataset has both human-driven and SWE-bench sessions, so a benchmark can be checked against real use.

**What it enables.** A synthetic workload generator for agent-serving systems, built from real distributions. It can be used without sharing private text.

### 10. Predicting cost and next steps: learned cost models

**Database idea.** Learned cost models and cardinality estimators predict how expensive a plan will be, from past queries.

**What I saw.** Each call records its category, latency and result size. 121 calls in the samples returned 5,000 tokens or more. These are the calls that grow the context fast.

**What it enables.**
- Predict the next tool and its result size, then prefetch or start it early while the model is still writing.
- Warn before a call that will flood the context.
- Plan capacity per agent type.

### 11. Security and audit: mining least-privilege policies from logs

**Database idea.** Audit logs are mined to find normal access patterns and flag unusual ones.

**What I saw.** The samples run powerful programs: `curl` 547, `ssh` 147, `docker exec` 122, `adb` 120, `rm` 77, `base64` 68.

**What it enables.** Learn, per agent type, which commands are normal. Generate permission allowlists from that, and flag rare or risky commands.

### 12. Replay and what-if: the trace as a log

**Database idea.** A write-ahead log lets a database replay history, recover, and go back in time.

**What it enables.**
- **Replay.** Rerun a session with a new cache policy, model, or tool. The paper's cache simulator (`../code/simulation/`) already does this for caching.
- **Checkpoints.** Restart from a mid-session state instead of from the start, for example after a failure.
- **Counterfactuals.** Ask "what if the agent had reused this result?" and measure the saved tokens and time. This is how use case 1 can be evaluated without changing a live agent.

### 13. Post-training: teaching the model from traces

This is a short summary of common practice, for a reader from databases.

| Method | Idea in plain words | What it needs from traces |
|---|---|---|
| Supervised fine-tuning (SFT) | Show the model good traces. It learns to copy them. | Full text of successful traces |
| Preference training | Show a better and a worse trace for the same task. It learns to prefer the better one. | Pairs with labels (success and failure) |
| Reinforcement learning (RL) | The model tries tasks and gets a score. It learns what scores well. | Tasks, a working environment, and a success check |
| Process rewards | Score each step, not only the end. | Step labels: error, retry, loop |
| Distillation and routing | A small model learns the frequent, easy steps. A router sends easy steps to it. | Many traces of the same kinds of steps |

How our data fits:
- **Full-text benchmark traces** carry labels (`resolved`, `success`), and some tasks have both a success and a failure. That is enough for SFT and preference pairs.
- **Harvard traces have no text**, so they cannot train a model directly. They still help: they show the real task mix, tool mix and session length to copy in training environments. Their error, retry and loop signals (use case 8) are weak step labels.
- **Link to reuse.** Use case 3's notes are a form of "learning" that needs no training: lessons are stored and read back. This is a fair baseline to compare against post-training.

## Which data supports which use case

| Use case | Harvard metadata | Full-text samples |
|---|---|---|
| 1. Result reuse | sizes the opportunity | shows real reuse |
| 2. View selection | yes (call shapes, blocks) | yes |
| 3. Lessons and notes | no | yes |
| 4. Plan rewriting | partly (categories, result sizes) | yes |
| 5. Provenance | no | yes (data traces) |
| 6. Staleness and conflicts | no (paths hidden) | yes (SWE) |
| 7. KV cache | yes | not needed |
| 8. Runaway detection | yes | yes, with success labels |
| 9. Benchmarks | yes | yes |
| 10. Cost prediction | yes | yes |
| 11. Security policies | yes (program names) | yes |
| 12. Replay | yes for caching | yes for agent behaviour |
| 13. Post-training | indirect only | yes, but only 18 traces here |

## Suggestions for what to demonstrate

1. **Cross-session reuse as materialized views (main story).** The same-user signal (10–31% of inputs) is strong and fits the database framing. Show it on Harvard data, then show real result reuse on full-text data tasks. Run many DataAgentBench tasks on the same databases, store results and notes, and measure saved calls and tokens.
2. **Runaway detection (cheap second demo).** It works on Harvard metadata alone, and the loops are easy to see.
3. **Plan rewriting with pushdown.** A natural extension of 1, and very familiar to database readers.

## Open points

- The 3 days are a small slice. One day takes about 1 minute to run, so all 112 days are feasible. Block ids are local to a week, so a full run should process one week at a time.
- Same-user reuse is concentrated: on 2026-06-09 one user made 86% of it. On the other two days the top user made 30–42%.
- I have not checked how much of the same-user reuse is session splitting by the dataset's reconstruction, versus truly new sessions.
- The full-text set is small (18 traces). The original benchmark repos have many more runs.
