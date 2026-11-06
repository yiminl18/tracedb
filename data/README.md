# Data

This folder holds **links, descriptions and metadata only**. The traces themselves are not stored here. Download them from the links below.

## Datasets

| Dataset | Link | Application | What one trace is | Text? | Metadata here |
|---|---|---|---|---|---|
| Harvard FreeInference Agentic Trace | [Hugging Face](https://huggingface.co/datasets/harvardMadsys/freeinference_agentic_trace) · [code](https://github.com/HarvardMadSys/freeinference_agentic_trace) | real coding and assistant agents (14 harnesses) | one agent session: LLM requests, tool calls, timing, tokens, prefix block ids | no | `harvard_freeinference/` |
| DataAgentBench | [GitHub](https://github.com/ucbepic/DataAgentBench) | data analysis, SQL / Mongo + Python | one run of one question (message list) | yes | `multi_trace/dataagentbench/` |
| Nebius SWE-rebench OpenHands | [Hugging Face](https://huggingface.co/datasets/nebius/SWE-rebench-openhands-trajectories) | software repair | one OpenHands run, with patch and `resolved` | yes | `multi_trace/nebius_swe_rebench_openhands/` |
| NVIDIA Open-SWE-Traces | [Hugging Face](https://huggingface.co/datasets/nvidia/Open-SWE-Traces) | software repair, 5 agents | one agent run, with patch and `resolved` | yes | `multi_trace/nvidia_open_swe_traces/` |
| Thoughtworks agentic-coding | [Hugging Face](https://huggingface.co/datasets/thoughtworks/agentic-coding-trajectories) | coding / repair | one session (`messages`) | yes | `multi_trace/thoughtworks_agentic_coding/` |
| MATM | [Hugging Face](https://huggingface.co/datasets/toeunkim/matm-trajectories) | WebArena web tasks, ALFWorld household tasks | one run: goal, then action + observation per step | yes | `multi_trace/matm/` |
| OpenResearcher | [Hugging Face](https://huggingface.co/datasets/OpenResearcher/OpenResearcher-Dataset) | web research QA | one run on one question: search / open / find, answer | yes | `multi_trace/openresearcher/` |
| AMA-bench | [Hugging Face](https://huggingface.co/datasets/AMA-bench/AMA-bench) | SQL, software, web, games, embodied, open QA | one episode, plus QA pairs about it | yes | `multi_trace/ama_bench/` |
| LMCache agentic-traces | [Hugging Face](https://huggingface.co/datasets/sammshen/lmcache-agentic-traces) | repair, research, mixed | one session (all requests with one `session_id`) | yes | `multi_trace/lmcache_agentic_traces/` |
| agentic-trace-samples | [GitHub](https://github.com/LittleHealth/agentic-trace-samples) | 15 hand-picked full-text traces from four of the sources above | one trace with a step-by-step reading page | yes | none (see the link) |

## Harvard FreeInference

16 weeks (2026-05-17 to 2026-09-05). 12,002 sessions from 267 accounts and 14 harnesses, 1.19M LLM requests, 1.21M tool calls. No message text. Tool arguments are anonymised. Inputs are given as chained hashes of 16-token blocks, which show shared prefixes. License: CC BY 4.0.

- `dataset_card.md`: the dataset's own card, with the full field list.
- `manifests/<week>.json`: per-week file list, sizes, checksums, block counts and excluded sessions.
- `sessions_index.csv`: one row per session (week, file, line, user, harness, start / end, request and call counts, size). Built by us.
- `session_types.csv`: the agent type we inferred for each session from its tool use, with the signal shares. Built by us; see `../observation/goals.md` for the rules.
- `exploration_samples/<type>.csv`: the 10 sessions picked per type for close reading.

## Full-text sources (multi_trace)

We drew 100 traces from each source. `multi_trace/sources.md` explains how the samples were built and what is special about each source. `multi_trace/<dataset>/samples.csv` lists the sampled traces (file, group, task / environment, model, outcome where the source has one). The raw traces are not stored; rebuild them from the source links.
