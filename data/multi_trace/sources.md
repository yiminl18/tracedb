# Multi-source trace samples

_Built 2026-11-05._ 100 full-text traces from each public source in [trace-data-sources.md](https://github.com/LittleHealth/agentic-trace-samples/blob/main/reports/trace-data-sources.md). Harvard FreeInference is not here. It has no text, and it lives in `../harvard_trace/`.

```
multi_trace/
├── <dataset>/
│   ├── samples.csv      one row per trace: file name, group, and key fields
│   └── traces/          100 files, NNN-<id>.json, one trace per file
├── cross_traces/            how alike traces of the same type are (content and structure)
├── content_examination/    which information in a trace recurs and could be reused
├── use_case/               why a large trace set is useful, 13 use cases with numbers
└── _raw/                downloaded source files the samples came from (see the end)
```

The three analysis folders repeat the studies in `../harvard_trace/` with the same names. They read a common step format made by `../scripts/multi_trace_normalize.py`. Some numbers also use a second sample with many runs of the same task, built by `../scripts/sample_multi_trace_same_task.py` (not stored here).

## Datasets

| Folder | Source | Application | One trace is | Sample is spread over |
|---|---|---|---|---|
| `dataagentbench` | [DataAgentBench](https://github.com/ucbepic/DataAgentBench) | data analysis, SQL / Mongo + Python | one run of one question (message list) | 5 models × 12 databases, 2 per pair (42 distinct questions) |
| `nebius_swe_rebench_openhands` | [Nebius SWE-rebench OpenHands](https://huggingface.co/datasets/nebius/SWE-rebench-openhands-trajectories) | software repair | one OpenHands run, with final patch and `resolved` | resolved 50 / not resolved 50 (99 distinct tasks) |
| `nvidia_open_swe_traces` | [NVIDIA Open-SWE-Traces](https://huggingface.co/datasets/nvidia/Open-SWE-Traces) | software repair | one agent run, with patch and `resolved` | 5 agents × 20 (claudecode, minisweagent, opencode, openhands, sweagent) |
| `thoughtworks_agentic_coding` | [Thoughtworks agentic-coding](https://huggingface.co/datasets/thoughtworks/agentic-coding-trajectories) | coding / repair | one session (`messages`) | 3 sources × 33–34 |
| `matm` | [MATM](https://huggingface.co/datasets/toeunkim/matm-trajectories) | WebArena web tasks, ALFWorld household tasks | one run: goal, then action + observation per step | WebArena 40 model runs + 10 expert runs, ALFWorld the same. Model runs are spread over model × success. |
| `openresearcher` | [OpenResearcher](https://huggingface.co/datasets/OpenResearcher/OpenResearcher-Dataset) | web research QA | one run on one question: search / open / find, final answer | 5 seeds × 20 |
| `ama_bench` | [AMA-bench](https://huggingface.co/datasets/AMA-bench/AMA-bench) | SQL, software, web, games, embodied, open QA | one episode, plus QA pairs about it | 6 domains × 16–17 |
| `lmcache_agentic_traces` | [LMCache agentic-traces](https://huggingface.co/datasets/sammshen/lmcache-agentic-traces) | repair, research, mixed | one session (all requests with the same `session_id`) | SWE-bench 46, GAIA 46, WildClaw 8 (all of them) |

## Notes per dataset

- **Most files keep the source record as it is.** Two changes: Thoughtworks `messages_json` and MATM `trajectory` are parsed from JSON strings into objects.
- **LMCache is stored in a compact form.** The source has one row per LLM request, and each input repeats the whole history. So each trace stores the longest input once (`longest_input`). `requests` lists every request in order, with message count, output length and gap. A request whose input is *not* a prefix of the longest one keeps its own full `input`; this happens in 1 of the 100 sessions. The last model output has no text in the source.
- **The same task appears many times in some sources.** DataAgentBench has up to 50 runs per question, Nebius about 10 per task, and OpenResearcher one per seed. This sample spreads over tasks. To study reuse across runs of one task, draw a second sample grouped by task.
- **Thoughtworks overlaps Nebius.** Its `nebius-swe-rebench-openhands` part is copied from Nebius.
- **NVIDIA samples come from 5 small files only**, one per agent (the smallest file each). They cover only some of the models and task sets in the source.
- **OpenResearcher samples come from seeds 42–46 only**, from the 13 of 60 files that were downloaded.
- **Success labels are the source's own.** OpenResearcher `status=success` means the run finished, not that the answer is right.

## Rebuild

`python3 scripts/sample_multi_trace.py <dataset function> …` (needs `pyarrow`). It reads `_raw/` and uses a fixed seed (20261105). The functions are `dataagentbench nebius nvidia thoughtworks matm openresearcher ama lmcache`.

## `_raw/`

The files downloaded before the download was stopped, about 7.6 GB. All are complete except OpenResearcher (13 of 60 files, the last one cut off) and NVIDIA (5 small files). They are kept only to rebuild the samples and can be deleted.
