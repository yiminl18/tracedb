# How alike are traces of the same type? (multi_trace)

_Written 2026-11-05. Part of Goal 1 and Goal 2 in [`../../goals.md`](../../goals.md). Same question as [`../../harvard_trace/cross_traces/similarity_within_type.md`](../../harvard_trace/cross_traces/similarity_within_type.md), now on full-text traces._

## The question

Do traces of the same type share structure or content? If they do, which part is shared, and where in the trace does it sit?

Here a "type" is one dataset, which is one application (data analysis, code repair, web tasks, web research, games). Inside a dataset, traces can also share a harness, a model, an environment (database, repo, site) or the exact task.

## Data and method

- **Sample A, spread over tasks.** The 800 traces in `../<dataset>/traces/`, 100 per dataset. Few traces share a task here.
- **Sample B, same task many times.** 460 extra traces built from `../_raw/` by `../../scripts/sample_multi_trace_same_task.py`:

  | Dataset | Tasks | Runs per task |
  |---|---|---|
  | DataAgentBench | 12 questions, one per database | 10 (2 per model) |
  | Nebius | 10 repair tasks with mixed outcomes | 10 |
  | OpenResearcher | 20 questions | 5 (one per seed) |
  | MATM | 10 WebArena tasks and 10 ALFWorld tasks | all model runs (8 and 6) |

- **One step format.** `../../scripts/multi_trace_normalize.py` turns all 8 formats into one list of steps per trace. A step is system, task, think, call, result or answer.
- **Content.** `cross_scan.py` splits each step into lines of at least 20 characters ("chunks"). A chunk counts as shared when another trace of the same dataset has the same chunk **anywhere**. Position does not matter, so this is not only prefix sharing. Separately, I measure the longest common prefix of the whole trace text with any other trace.
- **Structure.** Each call is mapped to an action word: read, search, edit, test, run, query, probe, click, go, and so on. I compare action mixes (Jensen-Shannon distance, 0 = same, 1 = disjoint) and order (share of shared action trigrams). `struct_scan.py` gives phases and start and end templates per harness.
- **Pairs are grouped by their closest relation.** In order: same task, same environment, same harness and model, same harness, same model, dataset only.

## Summary: five layers

| Layer | What it shares | How strong |
|---|---|---|
| **Same task** | tool results (file views, query outputs, search pages), the task text, often the same calls | **50–66% of all text** is also in another run of the same task |
| **Same environment** (database, repo, site) | the environment description, schema probes, the first page | small on top of the harness: DataAgentBench 34% vs 21% for other databases |
| **Same harness** | system prompt, tool definitions, fixed tool messages | the main shared layer across tasks (NVIDIA 16% vs 0.4% across harnesses) |
| **Same model** | almost nothing on its own | under 1% |
| **Same dataset (type)** | the skeleton: which actions exist, the rough phase order | content near 0% |

The type gives the skeleton. The task, the environment and the harness give the content. This matches the Harvard finding (user > harness > environment > type). The full text adds one more layer, the same task, and it is by far the strongest.

## Content observations

### C1. Runs of the same task share half their text, and most of it is not a prefix

Sample B. Share of each trace's text that also appears in another run of the same task:

| Dataset | Shared with any trace | Shared from the same task | Without the system prompt | Longest shared prefix |
|---|---|---|---|---|
| Nebius (code repair) | 66% | 65% | 50% | 19% |
| MATM (web, household) | 55% | 51% | 55% | 40% |
| DataAgentBench (SQL + Python) | 59% | 59% | 40% | 34% |
| OpenResearcher (web research) | 50% | 50% | 46% | **4%** |

The gap between the "shared" and "prefix" columns is content that a prefix cache cannot reuse. In OpenResearcher, runs on the same question share half their text, yet they share almost no prefix. They hit the same search pages, but in a different order.

### C2. What is shared inside one task is mostly tool results

Sample B. For each tool, the first number is the share of its result text that also appears in another run of the same task. The second is that tool's share of all result text.

| Dataset | Result text shared, by tool |
|---|---|
| Nebius | file viewer 89% (40% of results), shell 69% (59%) |
| OpenResearcher | page open 72% (17%), search 48% (82%) |
| DataAgentBench | database query 52% (82%), Python 13% (15%) |
| MATM WebArena | click 38%, type 40%, scroll 8% |

Read-only tools with a fixed environment (file views, page opens) are shared most. Results that depend on the agent's own code (Python, scroll position) are shared least.

### C3. Across different tasks, almost all sharing is the harness

Sample A, where traces rarely share a task. Share of text shared with another trace, split by the closest relation:

| Dataset | Shared | Same task | Same env | Same harness (+model) | Longest prefix |
|---|---|---|---|---|---|
| DataAgentBench | 50% | 41% | 9% | 0.2% | 27% |
| LMCache | 31% | 21% | 10% | - | 18% |
| Nebius | 31% | 1% | 4% | 26% | 16% |
| Thoughtworks | 19% | - | 9% | 10% | 3% |
| MATM | 18% | 0.4% | 5% | 13% | 0.4% |
| AMA-bench | 18% | - | 18% | - | 0.8% |
| NVIDIA | 17% | - | 1% | 16% | 12% |
| OpenResearcher | 7% | - | 7% (one corpus) | - | 5% |

- DataAgentBench and LMCache still show "same task" sharing, because Sample A has repeats there (29 DataAgentBench questions run more than once, 8 LMCache tasks run by several models).
- For the coding datasets, most cross-task sharing is the harness: the system prompt, tool definitions, and fixed messages such as "Your thought has been logged."
- In NVIDIA, two traces from the same harness share 16% of text on average. Two traces from different harnesses share 0.4%, even with the same model.

### C4. The environment description is reused exactly, but it sits after the question

All 42 DataAgentBench questions in Sample A use **one exact database description per database** (1.6K to 5.6K characters). Questions on the same database share it word for word. The description comes after the question in the first user message. So the shared prefix ends at the question, and the description is shared but not as a prefix. The Harvard note saw the same layout (C5 there).

### C5. A small model-specific line breaks the shared prefix early

The DataAgentBench system prompt (about 4.6K characters) is the same for every run, except for one sentence that explains how to read stored results. That sentence differs by model (`call_1` vs `functions.tool:id`). So two runs of the same question with different models share only the first 1,074 characters as a prefix. Two runs with the same model share 4,502. After that, almost everything matches again. A prefix cache loses nearly the whole prompt to one line.

### C6. When all runs fail or all succeed, they look alike; mixed outcomes look different

Sample B. Average similarity of two runs of the same task, by outcome:

| Dataset | Pair | Content shared | Action-order similarity | Action-mix distance |
|---|---|---|---|---|
| MATM | both succeed | 0.47 | 0.85 | 0.11 |
| MATM | both fail | 0.31 | 0.65 | 0.26 |
| MATM | one succeeds, one fails | 0.13 | 0.24 | 0.61 |
| Nebius | both succeed | 0.54 | 0.50 | 0.19 |
| Nebius | both fail | 0.55 | 0.51 | 0.18 |
| Nebius | mixed | 0.54 | 0.47 | 0.21 |

- In web and household tasks, success follows one path. Runs that succeed are almost copies of each other. A failing run leaves that path early.
- In code repair, success is decided by a few lines of the patch. Failing and succeeding runs read the same files and run the same tests, so they look the same.
- ALFWorld success is set by the task, not the model. In 8 of 10 tasks, all 6 models succeed or all 6 fail.

## Structure observations

### S1. The type fixes the action set and the phase order

From `struct_scan.py`. Each line is "first action, then middle, then end".

- **DataAgentBench:** query or probe the schema, then Python, then answer. The first query comes at 0–15% of the run, the first Python call at 12–50%, the answer at 80–94%.
- **Code repair (all harnesses):** search and read, then edit, then test, then submit. The first edit is at 25–49% of the run. Reading is 24–62% of all calls.
- **OpenResearcher:** search at step 0, the first page open by 5–10%. Searches are 55–71% of calls.
- **WebArena:** click from step 0, type at about 17%, stop at about 83%.
- **ALFWorld:** go, open, look, take, then put.

### S2. Inside a type, the harness and model shape the details

- **NVIDIA, same model, 5 harnesses.** openhands runs a median 67 calls, minisweagent 21, opencode 22. Only openhands and sweagent end with an explicit submit. claudecode and minisweagent mostly end with a read.
- **Thoughtworks.** mini-swe-agent and swe-agent search right at the start (the median first search is call 0). OpenHands starts with think and then read (22 of 33). OpenHands tests early (first test at 12%), swe-agent late (73%).
- **DataAgentBench, one harness, 5 models.**
  - gemini-3-pro probes the schema most (22% of calls) and always answers.
  - gemini-2.5-flash makes only 4 calls at the median and answers in only 9 of 24 runs.
  - gpt-5-mini starts by listing tables.

### S3. Action-mix and order distance by relation

Mean over pairs (Sample A, with Sample B for same task). Lower distance and higher order similarity mean more alike.

| Dataset | Same task | Same env | Same harness/model | Dataset only |
|---|---|---|---|---|
| Nebius: distance / order | 0.19 / 0.49 | 0.17 / 0.42 | 0.25 / 0.41 | - |
| MATM | 0.25 / 0.67 | 0.48 / 0.31 | 0.57 / 0.18 | 0.97 / 0.00 |
| OpenResearcher | 0.16 / 0.74 | - | 0.29 / 0.62 | - |
| DataAgentBench | 0.39 / 0.39 | 0.49 / 0.26 | 0.47 / 0.28 | 0.51 / 0.22 |
| NVIDIA | - | 0.28 / 0.65 | 0.30 / 0.52 | 0.34 / 0.52 |

- The same task always gives the most alike structure.
- In code repair, order similarity is about 0.4–0.5 for any pair, so the type alone already fixes a lot.
- MATM mixes two environments (WebArena and ALFWorld) with no shared actions, so its "dataset only" pairs are near 1.

### S4. Repeats inside one trace depend on the type

The share of calls that exactly repeat an earlier call in the same trace:

| Datasets | Repeat share |
|---|---|
| AMA-bench | 37% |
| MATM | 26% |
| Thoughtworks | 12% |
| Nebius | 9% |
| LMCache | 7% |
| NVIDIA | 3% |
| OpenResearcher | 2% |
| DataAgentBench | 1% |

Game and web agents repeat moves (`look`, `scroll [down]`). Data agents almost never repeat a query.

## Work-based similarity (added 2026-11-06)

`goals.md` now defines similarity as **shared work, weighted by cost**, not shared text or prefixes. This section redoes the analysis under that definition. The sections above stay as background.

**Method.** `work_scan.py`, on Sample B (same task many times) and Sample A (spread over tasks).

- **Unit of work** = one call and the result it got back.
- **Cost of a unit** = the LLM tokens of the request that issued it: everything the model read (tool definitions, system prompt, task, history) plus what it wrote, at about 4 characters per token, split over the calls in that request. No source records tool run time per step, so the tool side is only seen through result size.
- **Same work** = the same call (tool + arguments) with the same result, after masking run-local handles (storage keys, call ids, timestamps, element ids, temp names). **Position does not matter.** Each match takes its closest relation: same task, then same environment, then other.
- **Same result, other call** = a result of 200+ characters that another trace got from a differently written call.
- **Reusable sub-trace** = a run of consecutive units in one trace that are all same work found elsewhere. A long run could be skipped as one block, which also saves the LLM steps inside it.

Exact matching is strict, so all shares are **lower bounds**.

### W1. Each type has a bottleneck action

Share of LLM token cost by action (Sample A; DataAgentBench and MATM also checked on Sample B):

| Dataset | Bottleneck | Next | Note |
|---|---|---|---|
| DataAgentBench | **python 52%** | query 26% | python is 15–22% of result text but half the cost, and 35–39% of its cost is on runs that end in an error |
| OpenResearcher | **search 63–70%** | open 25–32% | search is also 77–82% of result text |
| MATM (WebArena, ALFWorld) | **click 35–46%** | none / scroll / type | |
| Nebius | read 23%, test 21%, edit 21% | run 15% | no single bottleneck |
| NVIDIA | **read 50%** | edit 20% | read is 77% of result text |
| Thoughtworks | edit 24%, read 22%, test 19% | run 15% | |
| LMCache | **read 40%** | search 15% | |

- **Steps get more expensive as the trace goes on**, because each request reads the whole history. In Nebius, a `submit` step costs 62–64K tokens and a `read` 26K. Work reused late in a trace saves more.
- **Size is not cost.** In DataAgentBench, `query` returns 60–66% of the result text but takes 26% of the cost. `python` returns little text but takes 52%. Reasoning over results costs more than fetching them.

### W2. Across runs of the same task, a large part of the cost is the same work

Sample B. Share of cost spent on a unit that another run of the same task also did, with the same result:

| Dataset | Same call, same result | Same result, other call | Result text that is reusable |
|---|---|---|---|
| MATM | **44%** | 7% | 52% |
| Nebius | **21%** | 4% | 42% |
| OpenResearcher | 7% | **14%** | 10% |
| DataAgentBench | 5% | **13%** | 9% |

- **In web tasks and code repair, runs redo the same calls.** MATM starts from the same page and clicks through the same pages. Nebius reads the same files (48% of read cost is reusable) and runs the same tests (18%).
- **In database QA and web research, runs reach the same result by different calls.** In DataAgentBench, 38% of query cost returns a result that another run got from a **differently written query**. In OpenResearcher, 34% of page-open cost and 25% of find cost do the same. Exact call matching misses this. **Reuse in database QA must match on the result or on the meaning of the query, not on its text.**
- **DataAgentBench python is almost never reused (0%).** Each run writes its own code. Its result is also usually new, so the savings in database QA must come from the queries and the data they return, not from the agent's code.

### W3. Reusable work often comes as whole sub-traces

Share of reusable cost that sits in runs of 3 or more consecutive reusable units (Sample B):

| Dataset | Runs of 1 | Runs of 2 | Runs of 3+ |
|---|---|---|---|
| MATM | 4% | 4% | **92%** (124 runs) |
| Nebius | 32% | 22% | **46%** (232 runs) |
| OpenResearcher | 60% | 26% | 14% |
| DataAgentBench | 75% | 19% | 5% |

- In MATM and Nebius, reusable work is mostly whole sub-traces: a fixed path through the same pages, or the same orientation sequence (view the repo, read the same files, run the same tests). Such a block can be skipped as a whole, which also removes its LLM steps.
- In DataAgentBench and OpenResearcher, reusable units are scattered single steps. There, reuse would serve one result at a time.

### W4. Across different tasks, exact reusable work is almost zero

Sample A, where few traces share a task:

- **Same environment, different task:** about 0–1% of cost in every dataset. In DataAgentBench, `list_db` is 50% reusable within the same database, but it is only 2% of the cost. Schema probes are reworded (`LIMIT 5` vs `LIMIT 3`), so they rarely match exactly.
- **Different environment:** 0% in every dataset except MATM (15%). There, different tasks start from the same site pages, so the first observations match.
- So the large shares in W2 depend on **the same task**. Across tasks, sharing exists mostly at the level of environment facts (see the content examination), not as identical call-and-result pairs.

### W5. Failed work is a large part of the cost

Share of cost on units whose result is an error: DataAgentBench 20–23% (python 35–39%), LMCache 12%, Nebius 9–11%, NVIDIA and Thoughtworks 10%, others 0–3%. In DataAgentBench, the bottleneck action is also where most of the waste is.

### What the work view adds

1. **Database QA:** the bottleneck is the agent's computation (python 52% of LLM cost, plus query execution that is not recorded). The reusable part is in query results: the same intermediate result recurs within a task, but under different query text. **The key for reuse should be the result or the meaning of the query.**
2. **Web tasks and code repair:** reuse comes as whole sub-traces, reached by exactly the same calls.
3. **Across different tasks, exact reuse is near zero.** To reuse across tasks, we must store finer facts (schema, value formats, join keys) or match by meaning.
4. **Limits of this pass:** cost is LLM tokens only, without prompt caching discounts and without tool run time (not recorded). Exact matching gives lower bounds. Sample B has 10–20 tasks per dataset.

## What this means for reuse

1. **Group past traces by task first, then environment, then harness.** The same task gives 50–66% shared text. Other tasks in the same environment give much less, but what they give (environment description, schema probes) is exact and stays valid.
2. **Reuse must work at any position, not only the prefix.** In OpenResearcher, 50% of the text is shared but only 4% of it is a prefix. In Nebius it is 66% shared and 19% prefix. Most reusable content is tool results in the middle of the trace.
3. **Read-only results are the best targets.** File views (89% shared), page opens (72%) and database queries (52%) recur across runs. Results of the agent's own code rarely do.
4. **The type is a skeleton, useful for routing and templates.** It tells which actions and phases to expect. It does not tell which content will recur.
5. **Small prompt differences break prefix reuse.** One model-specific sentence cuts the shared prefix from 4.5K to 1K characters (C5). Layout matters too: put the shared environment text before the question.

## Limits

- **Small samples.** 100 traces per dataset, and Sample B has 10–20 tasks per dataset.
- **Rough action labels.** They come from tool names and simple text rules. Reading a test file can be labelled as testing.
- **Exact line matching.** It misses near-copies, such as a file view with one changed line or a search page with a new timestamp. So the shares are lower bounds for similar content.
- **Missing labels.** Thoughtworks, OpenResearcher, DataAgentBench and LMCache have no correctness label in this format, so C6 uses MATM and Nebius only.
- **Partial LMCache text.** In the SWE-bench part, edits are in text form and often cut off, so its edit counts are low.

## Open questions

- How much of the same-task overlap is **the same result returned to the same call**, so that it could be served without running the tool? `../content_examination/reuse_information.md` measures this.
- Do near-duplicate results (same file after a small edit) give much more reuse than exact ones?
- Can a router pick the right past traces from the task text alone, before any tool is called?
