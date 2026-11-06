# Goals

_Started 2026-11-05_

## Goal 1: Why traces matter. What can they be used for?

Understand what agent traces are useful for, and why.

**Main use case: reuse.** Take an agent that answers natural-language questions by querying a database. Each trace records the intermediate computation results the agent produced while solving a query. Across many past queries, these traces likely contain a large amount of duplicated work. When a new query arrives, the agent may be able to reuse results from earlier traces during its own iterations instead of recomputing them.

- How much duplication actually exists across traces?
- What kinds of results can be reused (tool calls, query results, intermediate steps, context prefixes)?
- Beyond reuse, what other uses do traces support?

> **Note:** Reuse is just one example of why traces are useful. While going through the trace data, record any other use cases below.

### Other use cases seen in the data

_Add to this list as you look through traces: what you saw, where, and what it would enable._

Full write-up of 13 use cases, mostly seen through a database lens, with evidence: [`harvard_trace/use_case/README.md`](harvard_trace/use_case/README.md).

- **KV / prefix-cache reuse at the serving layer.** _(harvard_trace)_ Each request mostly extends the one before it: in the sample day, ~90% of requests are `append` transitions and most prompt tokens are cache hits. With `block_ids`, cache policies can be replayed and evaluated without the original text. This is reuse below the agent, at the inference layer.
- **Serving and capacity planning.** _(harvard_trace)_ Each request records `ttft_ms`, its duration, and token counts. Tool-call `latency_ms` shows the gaps between LLM requests. Together these describe the real workload shape for scheduling and sizing inference systems.
- **Benchmark vs. real usage.** _(harvard_trace)_ The dataset contains both human-driven sessions and `eval:swe-bench` runs. Comparing them shows where benchmarks misrepresent real agent behaviour.
- **Context management.** _(harvard_trace)_ `role_tokens` and `mutation` show how context grows and when it gets rewritten. In some sessions tool definitions alone are 10–33K tokens, so traces show where context is wasted.
- **Tool-use and failure analysis.** _(harvard_trace)_ Tool calls carry a category, the programs launched, an outcome, and result size. These show which tools dominate, which fail, and which produce large outputs.

## Goal 2: How similar are traces from different agents and tasks?

Compare traces from different agents and applications. Find in what ways they are similar or different, and produce a breakdown.

**Motivation:** design an algorithm for a *class* of traces, not one tailored to the traces of a single application or agent. If traces from a class of problems or agents share certain properties, one algorithm can handle any trace with those properties. That is a much stronger result.

- In what ways can traces be similar (structure, tool-use patterns, context growth, repetition, timing)?
- Which agents and applications fall into the same class under each kind of similarity?
- What properties define a class that an algorithm could rely on?

### How we define similarity (updated 2026-11-06)

**Similarity means shared work, not shared text or a shared prefix.**

- **Start from the bottleneck.** To finish a task, some steps cost the most time or money. In agentic QA over a database, these are the expensive computations: large queries, joins, long Python jobs.
- **A trace is a tree of computations.** If a subtree records a computation result, and a later query needs the same result, the later query can reuse it. That gives large savings.
- **Position does not matter.** A result can be reused wherever it sits in the trace. It does not need to be part of a shared prefix. So prefix sharing is **not** the measure of similarity.
- **So two traces are similar when they contain the same computation results**, for example:
  - the same call on the same environment that returns the same result;
  - the same intermediate result reached by different calls;
  - the same environment fact or failure.
- **Weight similarity by cost.** A shared result that took 10 minutes matters far more than one that took 10 ms. All shared results that could be reused should be managed for future reuse. The expensive ones come first.

**What to measure:**

1. **Bottleneck:** which actions or sub-traces take most of the cost (latency, tokens) to finish a task, per type.
2. **Reusable cost:** how much of that cost is spent on results that also appear in other traces (same task, same environment, same user, other users), ignoring position.

Older analyses that measure shared text or prefixes stay useful as background. New observations follow this definition.

Results under this definition: the "Work-based similarity" sections in [`harvard_trace/cross_traces/similarity_within_type.md`](harvard_trace/cross_traces/similarity_within_type.md) (cost from timing, all 12,002 sessions) and [`multi_trace/cross_traces/similarity_within_type.md`](multi_trace/cross_traces/similarity_within_type.md) (cost from LLM tokens, exact call and result matching).

## Data

- `sample_traces/`: agentic-trace-samples (browser, data-analysis, freeinference, research-qa, software-repair)
- `harvard_trace/release/`: FreeInference raw traces, 16 weeks, 12K sessions, 1.2M requests (`.jsonl.zst`, one session per line)
- `harvard_trace/code/`: reader, cache simulation, and figure scripts from the paper
- `harvard_trace/paper.pdf`: the paper that describes the dataset

## Plan

- [x] **Step 1, breadth first (BFS):** split traces into classes by *agent type* (the kind of task: coding, QA, browsing…), list every class, and pick 10 sample traces for each. See [Exploration: trace classes](#exploration-trace-classes-bfs-step-1).
- [ ] Step 2: look at each class's samples in depth: structure, tool use, context growth, reuse opportunities.
- [ ] Step 3: compare across classes (Goal 2) and record use cases (Goal 1).

## Exploration: trace classes (BFS step 1)

**Class = agent type**: the kind of task an agent does (coding, question answering, browsing…), not the specific agent product. Both datasets are mapped onto one shared set of types.

- **`sample_traces`** is labelled by application already. Each application is a public benchmark run by one agent, and the traces include message text.
- **`harvard_trace`** has no task label, only the agent name and metadata. Each session's type is **inferred from what its tools did** (`scripts/classify_session_types.py`). The rules use the share of tool calls in each signal group, checked in order:

| Rule (first match wins) | Type |
|---|---|
| browser control (navigate / click / playwright / computer use) ≥ 25% | web-browsing |
| web search / fetch ≥ 20% and code edits < 15% | web-research-qa |
| file edits + writes ≥ 10%, or git / build / test / packages ≥ 10% | coding |
| network, wait/poll, ssh / docker / curl / adb … ≥ 25% | ops-sysadmin |
| Python / SQL interpreter and data tools ≥ 30% | data-analysis |
| assistant tools (memory, skills, cronjob, session_search, clarify …) ≥ 20% | assistant-workflow |
| read / search / shell file inspection (ls, cat, head…) ≥ 35% | code-qa-exploration |
| none of the above | mixed-other |

### The 8 types

| Type | What the agent does | Harvard sessions (share) | Requests / session p50 · p95 | Main agents | Full-text samples (`sample_traces`) |
|---|---|---|---|---|---|
| **coding** | changes code: edits files, git, build, tests | 6,348 (53%) | 67 · 488 | hermes 24%, opencode 19%, eval:swe-bench 13% | software repair, 3 (OpenHands on SWE-rebench) |
| **code-qa-exploration** | answers questions about code or files by reading and searching, no edits | 2,105 (18%) | 25 · 171 | hermes 23%, opencode 16%, pi 14% | none |
| **data-analysis** | computes answers with Python / SQL, little code editing | 810 (7%) | 28 · 169 | hermes 47%, pi 37% | data analysis, 6 (DataAgentBench SQL/Mongo + Python) |
| **ops-sysadmin** | operates machines and services: ssh, docker, curl, processes | 758 (6%) | 33 · 235 | hermes 59%, oh-my-pi 10% | none |
| **web-research-qa** | answers questions by searching and reading the web | 565 (5%) | 17 · 97 | hermes 54%, pi 11%, kilo 11% | research QA, 3 (OpenResearcher) |
| **assistant-workflow** | personal-assistant work: memory, skills, scheduled jobs | 557 (5%) | 24 · 80 | hermes 93% | none |
| **web-browsing** | operates web pages: navigate, click, type | 229 (2%) | 55 · 372 | hermes 63%, opencode 12% | browser, 3 (WebArena agent, gpt-oss-20b) |
| **mixed-other** | no signal dominates (mostly generic shell / MCP use) | 630 (5%) | 33 · 328 | hermes 25%, pi 18%, codex 12% | none |

Per-session labels and signal shares: `harvard_trace/index/session_types.csv`.

### Sample traces per type

Folder: `harvard_trace/exploration/by_type/<type>/`. Each type holds:
- **10 Harvard sessions**, `<pXX>-<agent>-<session8>/` with `trace.json` and `steps.csv`. They are spread from short to long sessions (p05 … p95 by request count) and across as many different agents as possible. `samples.csv` lists them.
- **`sample_traces/`**, for the 4 types that have them: the full-text benchmark traces, moved here from `sample_traces/samples/`. The original repo is unchanged.

| Type | Harvard samples (agent, requests) | Full-text samples |
|---|---|---|
| coding | pi 9, swe-bench 19, hermes 29, opencode 42, zoo 59, openclaw 78, copilot 105, kilo 145, oh-my-pi 233, claude-code 486 | `posix-path` (resolved), `netcdf-variable-names`, `androidtv-adb` |
| code-qa-exploration | oh-my-pi 4, hermes 8, opencode 11, swe-bench 15, codex 21, copilot 30, zoo 43, claude-code 64, pi 93, kilo 171 | none |
| data-analysis | hermes 5, pi 9, claude-code 13, opencode 17, oh-my-pi 24, hermes 100 (subagents), cline 44, codex 61, openclaw 97, hermes 169 | `asian-index`, `trading-volume`, `perfect-rated-books`, `realreal-max-price`, `evening-businesses`, `indianapolis-ratings` |
| ops-sysadmin | pi 5, hermes 10, oh-my-pi 16, zcode 23, opencode 29, opencode 129 (subagents), codex 52, claude-code 71, openclaw 108, kilo 163 | none |
| web-research-qa | hermes 4, kilo 6, pi 9, oh-my-pi 11, openclaw 15, opencode 19, codex 25, copilot 31, cline 47, claude-code 104 | `pharaoh-order`, `image-personalization`, `football-finalist` |
| assistant-workflow | claude-code 4, hermes 7, pi 11, copilot 15, hermes 21, hermes 98 (subagents), hermes 30, copilot 41, hermes 50, copilot 130 | none |
| web-browsing | oh-my-pi 6, hermes 16, opencode 25, openclaw 35, kilo 41, opencode 180 (subagents), pi 83, zcode 104, copilot 205, pi 372 | `relationship-post` (success), `repost-eagle`, `promote-repository` |
| mixed-other | codex 3, hermes 7, pi 10, claude-code 18, oh-my-pi 27, hermes 57 (subagents), opencode 60, openclaw 99, zcode 178, copilot 201 | none |

More samples per type: `python3 -I scripts/extract_class_samples.py harvard_trace/release harvard_trace/index <out> <N>`.

### Open points from step 1

- **The type labels are a first cut, inferred from tool use only**, because the release has no message text. Spot checks of random sessions look right for coding, browsing, web research and assistant work. Two kinds of labels are least reliable:
  - **data-analysis:** running Python can't be told apart from data work. One SWE-bench debugging session landed here.
  - **very short sessions:** labels rest on just a few tool calls.

  Rules and thresholds are easy to revise in the script.
- Types with no full-text sample: code-qa-exploration, ops-sysadmin, assistant-workflow, mixed-other.
- Full-text types have only 3–6 traces each, fewer than 10.
- hermes dominates several types (assistant 93%, browsing 63%, ops 59%), so in those types agent and type are partly confounded.
- My request total (1.22M) is about 3% higher than the official 1.19M. My count includes nested subagent requests, which may explain the difference; not yet confirmed.

## Findings

- 

## Open questions

- 
