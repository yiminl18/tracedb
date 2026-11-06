# Similarity of traces within the same type

_Written 2026-11-05. Covers step 2 of the plan in [`../../goals.md`](../../goals.md)._

**Question.** Do traces of the same type (coding, data analysis, web research …) look alike? If so, in what way? The answer decides what a reuse system can rely on.

**Short answer.** Yes, but the similarity is in the *shape* of the work, not in the exact text. Exact shared text is rare across users. When it exists, it comes from the harness, the user, or a shared environment (same database, same website, same repo). It does not come from the type.

## Data and method

- **Harvard release, all 12,002 sessions**, not only the 80 samples. Each session uses the type label from `../index/session_types.csv`.
- **Full-text benchmark traces**: the 15 traces in `../exploration/by_type/*/sample_traces/` (coding 3, data analysis 6, web research 3, web browsing 3).
- **Content similarity** uses `block_ids`. Each id stands for one 16-token block. Ids are chained hashes, so two sessions share a block only if their whole input up to that block is the same. In practice, a shared block means a shared prefix.
- **Structure similarity** uses tool calls, token counts and context changes. Pairs of sessions are compared in five groups: same type, same harness, both, same user, and random.

**Pitfall found while doing this.** Block ids are numbered per week, so the same id in two different weeks means nothing. Most of the "overlap" first seen between the 80 samples came from this. All block numbers below compare sessions within one week only.

## Summary: four layers of similarity

| Layer | What is shared | How strong | Where it comes from |
|---|---|---|---|
| **User** | exact text and work habits | strongest: about 20% of a session's blocks also appear in the same user's other sessions | the same person working on the same project, across types |
| **Harness** | system prompt, tool definitions, tool names, how context is changed | medium: tool sets overlap about 0.55 (Jaccard), but the shared text prefix breaks after about 2K tokens | the agent product |
| **Environment** | database schema, web pages, repo layout | strong when present, but only visible in full-text traces | tasks that run on the same data or site |
| **Type** | which actions are used, in what order and in what phase | clear but abstract; no exact text | the kind of task |

So the type gives a shared **skeleton**: the same kinds of steps in a similar order. The exact **content** to reuse comes from the user, the harness or the environment.

## Observations: content similarity

### C1. Exact shared text across users is tiny

On average, 21% of a session's distinct blocks appear in some other session in the same week. Almost all of that comes from the same user.

| Type | Shared with any other session | Shared with the same user only | Shared with another user | Another user, same type |
|---|---|---|---|---|
| assistant-workflow | 50% | 48% | 2.8% | 1.6% |
| web-browsing | 39% | 36% | 2.5% | 1.7% |
| data-analysis | 36% | 34% | 2.0% | 1.5% |
| web-research-qa | 28% | 25% | 2.4% | 1.4% |
| ops-sysadmin | 22% | 20% | 2.7% | 1.9% |
| mixed-other | 20% | 19% | 1.0% | 0.4% |
| code-qa-exploration | 19% | 17% | 2.3% | 1.5% |
| coding | 19% | 18% | 1.1% | 0.9% |

(Shares are weighted by blocks.)

### C2. All text shared across users comes from the same harness

For every type, "shared with another user" equals "shared with another user of the same harness". Same type alone adds nothing: the same-type share is always *smaller* than the same-harness share. No block is ever shared between two different harnesses.

### C3. The static harness text is mostly not shared, because it is personalised early

The first request is mostly static text: system prompt plus tool definitions make up 50–90% of it (10K–25K tokens). Yet the prefix shared with another user is short.

| Harness | System + tool defs (tokens, p50) | Prefix shared with another user (tokens, p50) |
|---|---|---|
| claude-code | 25,147 | 2,000 |
| hermes | 20,669 | 2,080 |
| copilot | 20,161 | 2,832 |
| kilo | 18,524 | 1,984 |
| codex | 13,569 | 2,368 |
| zcode | 11,898 | 5,920 |
| opencode | 9,546 | 224 |
| openclaw, cline, zoo | 5K–17K | 0 |

Only 0–16% of sessions share 90% or more of their static part with another user. So something user-specific (date, working folder, user rules, skill list, MCP tools …) is placed early and breaks the prefix. Most of the static text is the same in meaning but not in position. A prefix cache cannot reuse it. A cache that does not depend on position could.

### C4. A user's text is reused across types

Same-user sharing is split between same-type and other-type sessions. For assistant-workflow, 25% of blocks are shared with the user's same-type sessions and 44% with the user's other-type sessions. Only coding leans clearly to same type (16% vs 8%). The user and the project matter more than the type.

### C5. The environment is the real source of shared content (full-text traces)

- **Data analysis (DataAgentBench).** All 6 traces have the same 4,362-char system prompt. D2 and D4 run on the same two stock databases, and their database descriptions are the same 2,623 chars. But the question comes *before* the description in the user message, so the shared prefix ends at the question. Putting stable content first would make it reusable.
- **Web browsing (WebArena).** Two of the three tasks start on the same page; the first observation is the same 1,201 chars. After that, the pages share almost no exact lines (2.5%), because element ids such as `[5225]` change from page to page. The page structure is the same, but the text is not.
- **Coding (OpenHands).** All 3 traces share an 11.5K-char system prompt. Repos differ, so file content does not overlap.

### C6. Inside one trace, duplication is high

- **Web browsing:** 5,495 observation lines, only 856 distinct. The two failed tasks spend 20 of 30 steps repeating one action (`click [14551]`, `tab_focus 0`) and getting the same page back.
- **Web research:** many searches are near copies of earlier ones: 2 of 8, 7 of 23, and 53 of 74 searches share at least half their words with an earlier search. One trace walks a template through a slot: "Collingwood night series 1978", "… 1979" …
- **Coding:** the same command runs again and again (`pip install -e .` 8 times, `python reproduce_issue.py` 3–4 times), and one file was viewed 15 times.
- **Harvard data:** 6–22% of tool calls repeat an earlier call in the same session (same tool, same argument pattern, same result size). They hold only 2–8% of result tokens. Arguments are anonymised, so this is an upper bound.

## Observations: structure similarity

### S1. Each type has one dominant action, done in long runs

Calls are mapped to coarse actions: R = read/search, E = edit/write, X = run code, W = web search/fetch, B = browser control, N = network/ssh, A = memory/skills. Each type has one three-step run that appears in most of its sessions:

| Type | Most common 3-step pattern (share of sessions) | Same action as the step before (p50) |
|---|---|---|
| code-qa-exploration | RRR 0.90 | 71% |
| data-analysis | XXX 0.88 | 79% |
| web-browsing | BBB 0.84 | 61% |
| web-research-qa | WWW 0.82 | 70% |
| coding | RRR 0.78, RRE 0.65, REE 0.63 | 50% |
| ops-sysadmin | NNN 0.77 | 57% |
| assistant-workflow | AAA 0.76 | 62% |

Coding is the only type that switches often, between reading and editing.

### S2. Sessions share a phase pattern

- **Read first.** In every type, reading has its highest share in the first 20% of the session.
- **The main action fills the middle.** In data analysis, running code goes from 60% to 69%. In browsing, browser use goes from 31% to 49%.
- **Wrap-up at the end.** In the last 20%, memory/skill actions jump from about 2% to 11–20% in most types (web research 20%, data analysis 15%). In coding, git goes from 4% to 7%.
- In the full-text coding traces, the phases are written in the prompt itself ("Phase 1. READING … Phase 8"). All three traces open with the same moves: view the workspace, `cd repo && pwd`, read the README, run pytest, grep the error text, write `reproduce_issue.py`, run it, edit, run again. In benchmarks, the shared skeleton comes from the prompt template.
- The data-analysis traces all follow: list tables, probe the schema (`SELECT * LIMIT k`, `DISTINCT`, `COUNT(*)`), run the targeted query, join in Python, return the answer.

### S3. Type explains *which* actions, not *in what order*

Distance between action distributions (Jensen-Shannon, lower = more alike):

| Pair | Action mix | Action transitions | Order beyond the mix |
|---|---|---|---|
| random | 0.44 | 0.64 | 1.01 |
| same type, different harness | 0.33 | 0.54 | 0.99 |
| same harness, different type | 0.48 | 0.69 | 1.09 |
| same type and harness, different user | 0.29 | 0.51 | 1.00 |
| same user, same type | 0.20 | 0.38 | 0.82 |

The action mix is clearly more alike within a type. Part of this is built in, because types were defined from the action mix. Once the mix is removed, the order is not more alike within a type. Only the same user makes the order more alike.

### S4. Harness and user shape the rest of the trace more than type does

Distance on features that were *not* used to define types (session size, user-turn share, calls per request, error rate, result size, token mix by role, context growth). 1.00 = random pairs, lower = more alike.

| Pair | Size and shape | Context mix | Tool I/O | All |
|---|---|---|---|---|
| same type, different harness | 0.91 | 0.99 | 0.95 | 0.96 |
| same harness, different type | 0.98 | 0.87 | 0.86 | 0.91 |
| same type and harness, different user | 0.83 | 0.82 | 0.73 | 0.81 |
| same user, same type | 0.66 | 0.68 | 0.61 | 0.67 |

Type mainly shapes session size: coding p50 is 67 requests, web research is 17. Harness shapes the token mix and tool I/O. Web-browsing is the most alike type (0.78), code-qa-exploration the least (1.14, less alike than random).

### S5. Tool sets and context changes belong to the harness

- **Tool definition sets** (Jaccard overlap of tool names): different harness 0.04–0.07, even for the same type. Same harness 0.53 (different type) and 0.56 (same type). Same user and harness 0.86.
- **Context changes** (`mutation` instead of `append`) depend on the harness. openclaw changes 51% of its requests (mostly injections), kilo 15%, swe-bench almost none. By type, the rate only ranges from 3.5% to 11%.
- **The same action has different names** in each harness: `bash`, `terminal`, `shell_command`, `exec_command`, `run_in_terminal`. Even argument order differs: `web_search(query, limit)` vs `web_search(limit, query)`. Any cross-harness matching needs a normalising step first.

## Observations: action templates

### A1. Action templates recur across users within a type

A template is a tool call with its values removed (the release already replaces paths and strings with `<path>`, `<str>` …).

| Type | Calls whose coarse template (category + first 3 programs) is also used by another user of the same type | Same, with the full argument pattern | Share of calls covered by the top 20 full templates |
|---|---|---|---|
| data-analysis | 93% | 73% | 57% |
| web-browsing | 91% | 72% | 47% |
| web-research-qa | 91% | 72% | 50% |
| coding | 95% | 65% | 31% |
| assistant-workflow | 88% | 60% | 34% |
| code-qa-exploration | 90% | 52% | 23% |
| ops-sysadmin | 83% | 35% | 16% |
| mixed-other | 83% | 34% | 23% |

Data, web research and browsing have small, shared action sets. Ops and mixed sessions are long-tailed.

### A2. Each type has signature templates

Lift = how much more likely a session is to be of this type when it uses the template.

- **web-browsing:** `browser_type`, `browser_click`, `browser_snapshot`, `browser_console` (lift 16–35).
- **web-research-qa:** `web_search`, `web_extract`, `webfetch` (lift 5–11).
- **assistant-workflow:** `memory`, `skill_manage`, `cronjob`, `session_search` (lift 5–7).
- **data-analysis:** `execute_code`, `python <heredoc>`, `vision_analyze` (lift 2–6).
- **coding:** `cd <path> && git diff`, `git status --short`, grep then edit (lift about 1.9, the highest possible because coding is half the data).

These partly repeat the rules used to define the types, so they confirm the labels more than they reveal new facts.

### A3. The same mistakes repeat across tasks

In the data-analysis traces, mistakes come from the environment's rules, not from the task: reading a stored result by guessing its file name instead of using `locals()` (D5), Python quoting errors twice in a row (D2), a date-format guess that fails (D1). In coding, most extra steps were missing build tools (`scikit-build-core`, `setuptools_scm`). These lessons hold for every task in the same environment.

## Work-based similarity (added 2026-11-06)

`goals.md` now defines similarity as **shared work, weighted by cost**, not shared text or prefixes. This section redoes the analysis under that definition. The sections above stay as background.

**Method.** `work_scan.py` reads all 12,002 sessions (1.28M tool calls).

- **Cost of a step** = the LLM request that asked for the call + the tool run that answered it. Serving a stored result skips the tool run. Supplying the result before the agent asks (or reusing a whole sub-trace) also skips the LLM request.
  - LLM time = request end − start, split evenly over the calls in that request.
  - Tool time = the gap until the next LLM request (`latency_ms`), split evenly over parallel calls.
  - "Ask User" time is human wait, so it is kept apart. Subagent calls are not counted twice: their nested requests are counted instead.
- **Idle gaps are left out.** 895 tool gaps longer than 30 minutes (0.07% of calls) hold about half of all raw tool time. They are the user being away or an approval wait, not computation.
- **Same work** = the same fingerprint: tool name, anonymised arguments, outcome and result size. Position does not matter: any earlier call anywhere counts. Arguments are anonymised, so every match is an **upper bound**. A **strict** match also needs a result of at least 500 tokens, where the same exact size is a stronger sign of the same result.

### W1. LLM time and tool time are about equal; tools dominate in ops and data work

| Type | LLM time | Tool time | Tool share | Idle gaps left out |
|---|---|---|---|---|
| coding | 2,238 h | 2,534 h | 53% | 2,617 h |
| code-qa-exploration | 285 h | 264 h | 48% | 270 h |
| data-analysis | 113 h | 144 h | 56% | 83 h |
| ops-sysadmin | 152 h | 233 h | 61% | 121 h |
| web-research-qa | 51 h | 36 h | 41% | 12 h |
| assistant-workflow | 39 h | 27 h | 41% | 0 h |
| web-browsing | 59 h | 40 h | 40% | 20 h |
| mixed-other | 182 h | 219 h | 55% | 185 h |

So both sides matter. A stored result skips the tool run. To also skip the LLM step, reuse must happen before the agent asks, for example by reusing a whole sub-trace.

### W2. Each type has a clear bottleneck action

Share of each type's step cost by tool category (top three). The number after the category is the mean step cost per call.

| Type | Bottleneck (share of cost, mean per call) | Next | Next |
|---|---|---|---|
| **data-analysis** | **Interpreter 71% (25 s)** | Other 10% | Wait/Poll 7% (97 s) |
| ops-sysadmin | Network 25% (20 s) | Wait/Poll 23% (44 s) | Other 19% |
| web-browsing | Browser 34% (12 s) | Other 16% | Interpreter 12% |
| web-research-qa | Browser / web fetch 30% (8 s) | Other 19% | Interpreter 13% (30 s) |
| coding | Other 20% (29 s; mostly `bash`, `hub`, `process`) | Interpreter 15% (23 s) | Edit 11% |
| code-qa-exploration | Other 21% | File Ops 17% | Search 13% |
| assistant-workflow | Other 48% (memory, skills, todo) | Interpreter 14% | File Ops 7% |
| mixed-other | Other 40% | Interpreter 13% | MCP 9% |

- **Data analysis has the sharpest bottleneck.** Running code (Python, SQL) is 71% of the cost. This matches the database QA case in `goals.md`: the expensive part is the computation.
- **Cost is not in the same place as tokens.** In coding, Read is 41% of result tokens but only 9% of cost. Reads are big but fast. A context-size view and a cost view point at different steps.

### W3. Cost is concentrated in few calls

The most expensive 1% of calls hold 17–24% of a type's cost. The top 10% hold 45–58%. In data analysis, the top 1% of calls each cost 5 minutes or more, and 72% of them are Interpreter runs. So a reuse system that only handles the most expensive calls already covers about half of the cost.

### W4. How much of the cost is work seen before

Share of each type's step cost spent on a call whose fingerprint was seen earlier (all calls / strict, results ≥ 500 tokens):

| Type | Same session | Same user, other session | Other user | New |
|---|---|---|---|---|
| data-analysis | 12% / 1% | **26% / 16%** | 10% / 8% | 53% / 75% |
| web-browsing | 21% / 6% | 14% / 7% | 11% / 9% | 54% / 78% |
| coding | 21% / 4% | 10% / 7% | 8% / 10% | 61% / 79% |
| mixed-other | 18% / 5% | 11% / 11% | 4% / 3% | 67% / 81% |
| assistant-workflow | 5% / 2% | 14% / 9% | 13% / 8% | 67% / 81% |
| web-research-qa | 8% / 2% | 14% / 9% | 10% / 6% | 68% / 83% |
| ops-sysadmin | 15% / 15% | 8% / 4% | 5% / 3% | 72% / 77% |
| code-qa-exploration | 12% / 3% | 10% / 7% | 6% / 6% | 73% / 85% |

- **Data analysis repeats the most work across sessions of the same user.** Up to 26% of its cost (16% strict) goes to calls the same user already ran in an earlier session. Inside its bottleneck (Interpreter), the same-user share is 32%. A user returning to the same data reruns the same expensive code.
- **Repeats inside a session are mostly small calls.** The same-session share drops from 12–21% to 1–6% under the strict rule, except in ops (15% both ways, likely the same poll or network call run again).
- **Other users matter more here than in the prefix view.** Under the prefix view, other users shared under 3% of blocks (C1). Under the work view, 3–10% of cost (strict) matches another user's call. In coding, these are mostly `read` of large files with the same size, such as the same library file or doc. Position-free matching finds this sharing; prefix matching cannot.
- These are upper bounds. Without the argument text we cannot confirm that two calls returned the same result.

### W5. Failed calls are a large part of the cost in some types

Share of step cost spent on calls that ended in an error: data-analysis 18%, assistant-workflow 16%, ops-sysadmin 12%, the other types 4–6%. In data analysis, the failures sit in the bottleneck: 18% of Interpreter cost is on failing runs. Notes about known failures (see A3) would save expensive steps, not just cheap ones.

### What the work view adds

1. **The target is clear for database QA:** expensive code runs. They are 71% of the cost, and up to a third of that cost repeats across the same user's sessions.
2. **Rank by cost, not by size or frequency.** Large reads are cheap; long code runs and polls are expensive.
3. **Position-free matching finds more cross-user sharing** than the prefix view, but it is still small. Same-user reuse stays the main source.
4. **Harvard cannot confirm a match.** Full-text traces are needed to check that the same call returned the same result. The multi_trace pass does this.

## What this means for reuse

1. **Type is a good key for plans, not for content.** A type gives a skeleton: which actions, in what phases. Exact content (prefixes, query results, file reads) should be keyed by user and project, harness, and environment (database, site, repo).
2. **Most static text could be shared but is not.** 10K–25K tokens of system prompt and tool definitions are the same in meaning across users of a harness. A user-specific line placed early breaks the shared prefix. Moving stable text first, or using a cache that does not depend on position, would unlock it.
3. **Environment probes are the best content to share across tasks.** Schema discovery, README and test layout, and the first page of a site repeat whenever tasks share an environment.
4. **Loops and near-duplicate calls are the easiest wins inside one trace.** Examples: stuck browser actions, repeated searches with one changed slot, rebuilds and re-runs.
5. **Matching across harnesses needs normalising first:** tool names, argument order, and the anonymised placeholders.

## Limits

- Types are inferred from tool use. Any similarity in the action mix is partly built in by the labels.
- Block ids show only shared prefixes. The same text at different positions is invisible, so C1 to C3 understate how much text is truly shared.
- Arguments are anonymised. Repeats and template matches are upper bounds.
- Full-text traces are few (3–6 per type) and come from benchmarks, where one prompt template makes traces look more alike than real use.

## Open questions

- How much of the static harness text is the same across users once position is ignored? This needs the text, or hashes that do not depend on position.
- In the Harvard data, do different users of the same type ever work on the same environment (same repo, same site)? This cannot be seen from this release.
- Can the type skeleton (S1, S2) be written as a small grammar, and does it predict the next action better than the harness alone?
