# Exploration: sample traces per agent type

Step 1 of the BFS plan in [`../../goals.md`](../../goals.md): one folder per **agent type**, meaning the kind of task the agent does, not the specific agent product.

```
exploration/by_type/
├── coding/                 changes code: edits, git, build, tests
├── code-qa-exploration/    reads / searches files to answer questions, no edits
├── data-analysis/          computes answers with Python / SQL
├── ops-sysadmin/           ssh, docker, curl, process control
├── web-research-qa/        searches and reads the web to answer questions
├── assistant-workflow/     memory, skills, scheduled jobs (personal assistant)
├── web-browsing/           navigates and clicks through web pages
└── mixed-other/            no dominant signal
    each type:
    ├── samples.csv                         the 10 Harvard picks: agent, session_id, size, models
    ├── <pXX>-<agent>-<session8>/
    │   ├── trace.json                      full session as released (metadata only, no text)
    │   └── steps.csv                       one row per LLM request, for reading
    └── sample_traces/                      full-text benchmark traces of this type (4 types only)
```

## Where the types come from

- **Harvard sessions** have no task label. Their type is inferred from tool use by `../../scripts/classify_session_types.py`; see the rule table in `goals.md`. Labels for all 12,002 sessions are in `../index/session_types.csv`.
- **`sample_traces/` subfolders** keep their benchmark's own label:

| Type | Moved from `sample_traces/samples/` |
|---|---|
| coding | `software-repair` (OpenHands on SWE-rebench) |
| data-analysis | `data-analysis` (DataAgentBench) |
| web-research-qa | `research-qa` (OpenResearcher) |
| web-browsing | `browser` (WebArena) |

## How the 10 Harvard samples were picked

Sessions of a type are sorted by request count. One pick is made near each of p05, p15 … p95, so the picks run from short to long. Around each point, the agent picked least so far is preferred, so a type's samples span different agents. If no pick uses subagents but the type has such sessions, the median pick is swapped for one (label `pXX-subagents`).

Resample or take more (last two arguments: samples per class, `type` or `harness`):

```bash
python3 -I scripts/extract_class_samples.py harvard_trace/release harvard_trace/index \
    harvard_trace/exploration/by_type 10 type
```

## steps.csv columns

| Column | Meaning |
|---|---|
| `depth`, `parent_call` | 0 = top-level session; ≥1 = subagent request, started by the `step.call_index` in `parent_call` |
| `step`, `turn`, `trigger` | position in the session; `user` = new user turn, `tool` = follows tool results |
| `model`, `provider` | requested model, anonymised provider |
| `dur_ms`, `ttft_ms` | request duration, time to first token |
| `prompt_tok`, `completion_tok`, `cached_tok` | provider-reported tokens (cached = prefix-cache hit) |
| `sys_tok` … `tooldef_tok` | input tokens by role: system, user, assistant, tool results, tool definitions |
| `n_msgs`, `n_tools_defined` | messages in the input; number of tools offered |
| `transition`, `cause` | how the input changed from the previous request: `append` / `mutation` / `switch` |
| `n_blocks` | number of 16-token prefix blocks in the input |
| `finish`, `tool_calls` | finish reason; tool calls as `name[category]->outcome` |
