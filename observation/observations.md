# Observations

_Personal notes made while reading results from the explorations. Goals and plan: [`goals.md`](goals.md)._

## Exploration results to look at

- Trace classes (BFS step 1): [`harvard_trace/exploration/README.md`](harvard_trace/exploration/README.md)
- What in a trace helps reuse: [`harvard_trace/content_examination/reuse_information.md`](harvard_trace/content_examination/reuse_information.md)
- Similarity within type, full-text traces: [`multi_trace/cross_traces/similarity_within_type.md`](multi_trace/cross_traces/similarity_within_type.md)
- What recurs and could be reused, full-text traces: [`multi_trace/content_examination/reuse_information.md`](multi_trace/content_examination/reuse_information.md)

## Notes

### 2026-11-05: from the content examination

**Observation 1: different applications reuse traces in different ways.**

- Each type of application has its own setup and its own patterns.
  - Data analysis probes a schema, runs queries, and computes in Python.
  - Coding reads files, sets up the environment, edits, and runs tests.
  - Web research searches, opens pages, and finds facts.
  - Browsing moves through pages and fills in forms.
- Because the logic differs, what can be reused also differs. So does how to reuse it.

**Open question: how can we model traces at a higher level?**

- We want one abstraction that fits traces from many applications.
- With it, we could build a meta approach to reuse that works across applications, not one method per application.

**Follow-up: start from one database application, then generalize.**

- Without a shared abstraction, it is hard to state the reuse problem precisely across many applications.
- So we should first fix one or two major applications from the database community. With the application logic fixed, we can model the traces and state the problem concretely.
- This gives two steps:
  1. **First work:** pick one major database application and study trace reuse there.
  2. **Second work:** build a trace abstraction that covers many applications, and extend the approach to it.

### 2026-11-06: from similarity within type

**Observation 1: trace similarity comes from environment similarity.**

- When two queries work on the same environment, they see the same data.
  - Two queries on the same database.
  - Two queries on the same web page.
  - Two queries on the same repo.
- Because the data is the same, their traces are very similar.

**Observation 2: trace similarity comes from user behavior.**

- Many shared parts between traces come from the same user.
- Traces from different users share very little.

**Observation 3: trace similarity comes from the same harness.**

- Traces from the same harness share many parts.
  - The system prompt.
  - The tool names.
  - The tool definitions.

**Observation 4: each type has one main action.**

- In each type, one action takes up most of the workflow.
  - Coding: read files.
  - Web research: search.
  - Browsing: click.
  - Household tasks: move to a place.
- So the structure of traces in one type is built around this action.

**Observation 5: each type has one bottleneck action, measured by cost.**

- Here the bottleneck is the action that takes the most cost or latency to finish a task.
  - Data analysis: running code. It is 71% of the cost in Harvard and 52% of the LLM token cost in DataAgentBench.
  - Web research: search, 63–70% of the cost.
  - Browsing: click or browser actions, 34–46%.
  - Ops: network and wait / poll, about half of the cost together.
- This is a direct chance to optimize: reuse the results of the most expensive computations first.
- Cost is also concentrated in a few calls. In Harvard, the most expensive 10% of calls hold about half of the cost. So handling the costly calls alone already covers much of the possible savings.

### 2026-11-06: from the content examination (full-text traces)

**Observation 1: the same mistakes repeat across runs, and they come from the environment.**

- Within one task, many runs make the same mistake.
  - MATM (browsing): 86% of failing calls also fail in another run of the same task.
  - Nebius (coding): 52%.
  - OpenResearcher (web research): 16%. One page that is not in the corpus failed 11 times across 5 runs of one question.
- These mistakes come from the rules of the environment, such as a page that does not exist or a path that is wrong. They do not come from one run's own choices.
- The contrast supports this. In DataAgentBench, most failures are bugs in the agent's own Python code. Each run writes different code, so only 2% of failing calls repeat.
- This fits the similarity observation above: the environment is the root of much of what traces share. That includes both the shared steps and the shared mistakes.
- So a note about a mistake can be stored once per environment and reused by every later run.
