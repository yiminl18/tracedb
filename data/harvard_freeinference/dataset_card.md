---
pretty_name: FreeInference Agentic Trace
license: cc-by-4.0
tags:
  - agents
  - llm-inference
  - tool-use
  - prefix-caching
  - traces
size_categories:
  - 1M<n<10M
configs:
  - config_name: traces
    default: true
    data_files:
      - split: week_2026_05_17
        path: "viewer/traces/2026-05-17_2026-05-23.parquet"
      - split: week_2026_05_24
        path: "viewer/traces/2026-05-24_2026-05-30.parquet"
      - split: week_2026_05_31
        path: "viewer/traces/2026-05-31_2026-06-06.parquet"
      - split: week_2026_06_07
        path: "viewer/traces/2026-06-07_2026-06-13.parquet"
      - split: week_2026_06_14
        path: "viewer/traces/2026-06-14_2026-06-20.parquet"
      - split: week_2026_06_21
        path: "viewer/traces/2026-06-21_2026-06-27.parquet"
      - split: week_2026_06_28
        path: "viewer/traces/2026-06-28_2026-07-04.parquet"
      - split: week_2026_07_05
        path: "viewer/traces/2026-07-05_2026-07-11.parquet"
      - split: week_2026_07_12
        path: "viewer/traces/2026-07-12_2026-07-18.parquet"
      - split: week_2026_07_19
        path: "viewer/traces/2026-07-19_2026-07-25.parquet"
      - split: week_2026_07_26
        path: "viewer/traces/2026-07-26_2026-08-01.parquet"
      - split: week_2026_08_02
        path: "viewer/traces/2026-08-02_2026-08-08.parquet"
      - split: week_2026_08_09
        path: "viewer/traces/2026-08-09_2026-08-15.parquet"
      - split: week_2026_08_16
        path: "viewer/traces/2026-08-16_2026-08-22.parquet"
      - split: week_2026_08_23
        path: "viewer/traces/2026-08-23_2026-08-29.parquet"
      - split: week_2026_08_30
        path: "viewer/traces/2026-08-30_2026-09-05.parquet"
  - config_name: requests
    data_files:
      - split: week_2026_05_17
        path: "release/2026-05-17_2026-05-23/processed/requests.parquet"
      - split: week_2026_05_24
        path: "release/2026-05-24_2026-05-30/processed/requests.parquet"
      - split: week_2026_05_31
        path: "release/2026-05-31_2026-06-06/processed/requests.parquet"
      - split: week_2026_06_07
        path: "release/2026-06-07_2026-06-13/processed/requests.parquet"
      - split: week_2026_06_14
        path: "release/2026-06-14_2026-06-20/processed/requests.parquet"
      - split: week_2026_06_21
        path: "release/2026-06-21_2026-06-27/processed/requests.parquet"
      - split: week_2026_06_28
        path: "release/2026-06-28_2026-07-04/processed/requests.parquet"
      - split: week_2026_07_05
        path: "release/2026-07-05_2026-07-11/processed/requests.parquet"
      - split: week_2026_07_12
        path: "release/2026-07-12_2026-07-18/processed/requests.parquet"
      - split: week_2026_07_19
        path: "release/2026-07-19_2026-07-25/processed/requests.parquet"
      - split: week_2026_07_26
        path: "release/2026-07-26_2026-08-01/processed/requests.parquet"
      - split: week_2026_08_02
        path: "release/2026-08-02_2026-08-08/processed/requests.parquet"
      - split: week_2026_08_09
        path: "release/2026-08-09_2026-08-15/processed/requests.parquet"
      - split: week_2026_08_16
        path: "release/2026-08-16_2026-08-22/processed/requests.parquet"
      - split: week_2026_08_23
        path: "release/2026-08-23_2026-08-29/processed/requests.parquet"
      - split: week_2026_08_30
        path: "release/2026-08-30_2026-09-05/processed/requests.parquet"
  - config_name: tool_calls
    data_files:
      - split: week_2026_05_17
        path: "release/2026-05-17_2026-05-23/processed/tool_calls.parquet"
      - split: week_2026_05_24
        path: "release/2026-05-24_2026-05-30/processed/tool_calls.parquet"
      - split: week_2026_05_31
        path: "release/2026-05-31_2026-06-06/processed/tool_calls.parquet"
      - split: week_2026_06_07
        path: "release/2026-06-07_2026-06-13/processed/tool_calls.parquet"
      - split: week_2026_06_14
        path: "release/2026-06-14_2026-06-20/processed/tool_calls.parquet"
      - split: week_2026_06_21
        path: "release/2026-06-21_2026-06-27/processed/tool_calls.parquet"
      - split: week_2026_06_28
        path: "release/2026-06-28_2026-07-04/processed/tool_calls.parquet"
      - split: week_2026_07_05
        path: "release/2026-07-05_2026-07-11/processed/tool_calls.parquet"
      - split: week_2026_07_12
        path: "release/2026-07-12_2026-07-18/processed/tool_calls.parquet"
      - split: week_2026_07_19
        path: "release/2026-07-19_2026-07-25/processed/tool_calls.parquet"
      - split: week_2026_07_26
        path: "release/2026-07-26_2026-08-01/processed/tool_calls.parquet"
      - split: week_2026_08_02
        path: "release/2026-08-02_2026-08-08/processed/tool_calls.parquet"
      - split: week_2026_08_09
        path: "release/2026-08-09_2026-08-15/processed/tool_calls.parquet"
      - split: week_2026_08_16
        path: "release/2026-08-16_2026-08-22/processed/tool_calls.parquet"
      - split: week_2026_08_23
        path: "release/2026-08-23_2026-08-29/processed/tool_calls.parquet"
      - split: week_2026_08_30
        path: "release/2026-08-30_2026-09-05/processed/tool_calls.parquet"
---
# FreeInference Agentic Trace

**New: Interactive Session Explorer!** We've launched an interactive session explorer to help you explore the dataset interactively. [Try it out here!](https://trace.agentic-system.org/)

This dataset contains 16 weeks (2026-05-17 to 2026-09-05) of coding and assistant agents interacting with LLMs through the FreeInference gateway: **12,002 agent sessions** from 267 accounts and 14 agent harnesses, with **1,186,582 LLM requests** and **1,213,347 tool calls**. 

The dataset accompanies the paper *From Requests to Sessions: A Large-Scale Characterization of Human-Driven Agentic Workloads*. The code that reads it, replays its prefix cache and reproduces the paper's figures is at [HarvardMadSys/freeinference_agentic_trace](https://github.com/HarvardMadSys/freeinference_agentic_trace).

The trace includes request timing and serving latency, models and anonymized providers, token usage, tool definitions and calls, input mutations, and block-level prefixes for cache replay. **No message text is released.**

## Traces

The primary release is under:

```text
release/<week>/traces/<day>.jsonl.zst
```

Each file is zstd-compressed JSONL with **one agent session per line**. A session contains its LLM requests in order. Each request records the model invocation and the tool calls produced by the response. If a tool call launches a subagent, the complete subagent session is nested under that call.

A trace has the following structure:

```jsonc
{
  "week": "2026-05-17_2026-05-23",
  "day": "2026-05-17",

  // Session field
  "session_id": "...",
  "user_id": "...",
  "harness": "claude-code",
  "start_ms": 1778977459236,
  "end_ms": 1778984067198,

  // LLM requests in chronological order.
  "requests": [
    {
      // Position in the session and user turn.
      "step": 0,
      "turn_index": 0,

      // What triggered the request and how it connects
      // to the previous request.
      "step_trigger": "user",
      "link_reason": null,

      // Request timing at the gateway.
      "start_ms": 1778977459236,
      "end_ms": 1778977468828,
      "ttft_ms": 7889,

      // Requested model and anonymized upstream provider.
      "model": "glm-5.1",
      "provider": "B",
      "stream": true,

      // Token counts reported by the provider.
      "provider_tokens": {
        "prompt": 52731,
        "completion": 230,
        "cached": 1280
      },

      // Input tokens broken down by message role, tokenized with tiktoken.
      "role_tokens": {
        "system": 1790,
        "user": 1110,
        "assistant": 15916,
        "tool": 25725,
        "tool_definitions": 8638
      },

      // Shape of the input and tools exposed to the model.
      "n_messages": 179,
      "tool_definition_names": [
        "read_file",
        "write_file",
        "terminal",
        "execute_code"
      ],
      "finish_reason": "tool_calls",

      // How the input differs from the previous request.
      // Null for the first request in a session.
      "mutation": null,

      // Input represented as chained 16-token blocks,
      // tokenized with tiktoken.
      "block_ids": [123, 456, 789, "..."],

      // Tool calls produced by the model response.
      "tool_calls": [
        {
          "call_index": 0,
          "name": "terminal",

          // Normalized tool type. Shell calls additionally record
          // the programs found in the command.
          "category": "*Interpreter",
          "programs": ["python"],
          "parse_status": "success",

          // Arguments preserve their structure while sensitive
          // values are replaced with typed placeholders.
          "arguments": [
            [
              "command",
              "python <path> --status"
            ],
            [
              "timeout",
              "<num>"
            ]
          ],

          // Tool outcome and the gap until the next LLM request.
          "outcome": "error",
          "result_tokens": 842,
          "latency_ms": 5272,

          // If this call starts another agent, its complete
          // session is nested here.
          "subagent": null
        }
      ]
    },

    {
      "step": 1,
      "turn_index": 0,
      "step_trigger": "tool",
      "link_reason": "tool_call_id",

      // Later requests contain the same fields.
      // Here the previous input was extended without changing
      // any earlier content.
      "mutation": {
        "transition": "append",
        "cause": null
      },

      "tool_calls": ["..."]
    }
  ]
}
```

For convenience, `processed/` contains Parquet tables derived by parsing and flattening the traces into sessions, requests, and tool calls.

## Notes

## Notes

- Request inputs are tokenized with `o200k_base`, not the provider's tokenizer and grouped into 16-token blocks. Inputs are rendered using the following order: **tool definitions → system prompt → messages**, Partial final blocks are dropped.
- Block IDs are local to each week. When replaying multiple weeks, make them globally unique, e.g., by prefixing them with the week.
- Sessions are reconstructed by linking related requests through repeated tool-call IDs and assistant messages.
- Sessions do not cross week boundaries. A session that continues into the next week is released as a new session.
- Rare tool names and commands used by fewer than 3 accounts are replaced with `other` or typed placeholders.
- Each week's `manifest.json` reports sessions excluded from the release and their exclusion reasons.

## Usage

### DuckDB

Query the derived Parquet tables directly from the Hugging Face Hub:

```sql
SELECT
    sessions.harness,
    count(*) AS requests,
    approx_quantile(requests.ttft_ms, 0.5) AS p50_ttft_ms
FROM 'hf://datasets/harvardMadsys/freeinference_agentic_trace/release/*/processed/requests.parquet' AS requests
JOIN 'hf://datasets/harvardMadsys/freeinference_agentic_trace/release/*/processed/sessions.parquet' AS sessions
    USING (session_id)
GROUP BY sessions.harness
ORDER BY requests DESC;
```

### Hugging Face Datasets

```python
from datasets import load_dataset
traces = load_dataset(
    "harvardMadsys/freeinference_agentic_trace",
    "traces",
    split="week_2026_08_30",
)
```

The derived request and tool-call tables are also available as separate configurations for convenience:

```python
requests = load_dataset(
    "harvardMadsys/freeinference_agentic_trace",
    "requests",
    split="week_2026_08_30",
    streaming=True,
)
tool_calls = load_dataset(
    "harvardMadsys/freeinference_agentic_trace",
    "tool_calls",
    split="week_2026_08_30",
    streaming=True,
)
```

### Download

Download the full dataset:

```bash
hf download harvardMadsys/freeinference_agentic_trace \
    --repo-type dataset \
    --local-dir freeinference_trace
```

Or download one week:

```bash
hf download harvardMadsys/freeinference_agentic_trace \
    --repo-type dataset \
    --local-dir freeinference_trace \
    --include "release/2026-08-30_2026-09-05/*"
```

## License

This dataset is released under the [Creative Commons Attribution 4.0 International License (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

If you use this dataset, please cite the paper below.

## Citation

If you use the dataset or artifact in your research, please cite our paper:

```bibtex
@misc{nixon2026requests,
  title        = {From Requests to Sessions: A Large-Scale Characterization
                  of Human-Driven Agentic Workloads},
  author       = {Nixon, William and Tian, Muxin and Zheng, Yunjia
                  and Gunawi, Haryadi S. and Yang, Juncheng},
  year         = {2026},
  month        = nov,
  publisher    = {Zenodo},
  note         = {Preprint},
  doi          = {10.5281/zenodo.23903440},
  url          = {https://doi.org/10.5281/zenodo.23903440}
}
```