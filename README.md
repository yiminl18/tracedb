# tracedb

What agent traces contain, how alike they are, and how their computation results could be reused by later queries.

```
data/          links to each trace dataset, descriptions, and metadata (no raw traces)
observation/   write-ups: goals, personal observations, and one note per exploration task
code/          analysis code (to be added)
```

Start with [`observation/goals.md`](observation/goals.md) for the goals and plan, and [`observation/observations.md`](observation/observations.md) for the main observations so far.

| Exploration task | Harvard FreeInference (metadata only) | Full-text sources |
|---|---|---|
| Trace classes by agent type | [exploration](observation/harvard_trace/exploration/README.md) | |
| What in a trace helps reuse | [content examination](observation/harvard_trace/content_examination/reuse_information.md) | [content examination](observation/multi_trace/content_examination/reuse_information.md) |
| How alike traces of one type are | [similarity within type](observation/harvard_trace/cross_traces/similarity_within_type.md) | [similarity within type](observation/multi_trace/cross_traces/similarity_within_type.md) |
| Why a large trace set is useful | [use cases](observation/harvard_trace/use_case/README.md) | [use cases](observation/multi_trace/use_case/README.md) |

Some links inside the notes point to local trace files or scripts that are not in this repo yet.
