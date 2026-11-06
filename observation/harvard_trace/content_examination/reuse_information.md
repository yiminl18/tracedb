# What in a trace helps reuse?

_Started 2026-11-05. Part of Goal 1 in [`../../goals.md`](../../goals.md)._

## The question

We want to reuse past traces to make a new query cheaper or better. Not every part of a trace is worth keeping. This note asks: **which kinds of information inside a trace help a future query, and under what conditions?**

## What I looked at

| Source | Traces | What they contain |
|---|---|---|
| data analysis (DataAgentBench) | 6 | full text: SQL / Mongo queries, Python code, results, answer |
| coding (OpenHands on SWE-rebench) | 3 | full text: shell commands, file views, edits, test output, patch, `resolved` label |
| web research QA (OpenResearcher) | 3 | full text: searches, page opens, finds, answer, reference answer |
| web browsing (WebArena) | 3 | full text: page tree per step, action, reasoning, `success` label |
| Harvard FreeInference | 80 sample sessions (8 types × 10), plus one full week scanned | metadata only: token counts, prefix block ids, tool names, anonymized arguments, outcome, latency |

Full-text traces are in `../exploration/by_type/<type>/sample_traces/`. Harvard samples are in `../exploration/by_type/<type>/p*/`. The week scan script is described in [Method notes](#method-notes).

## Short answer

A trace holds eight kinds of information that can help a later query. They differ a lot in how often they apply and how risky they are to reuse.

| # | Information | Example from the traces | When it helps a new query | Risk if reused blindly |
|---|---|---|---|---|
| 1 | **Environment facts** (schema, value formats, join keys, file layout, site map) | the `Date` column mixes two formats (D1); `businessid_X` maps to `businessref_X` (D6); test command and repo layout (coding) | any later query on the **same database, repo or site**, even a very different question | low, as long as the environment version is the same |
| 2 | **Negative facts** (what failed and why) | a date parse that failed (D1); `/r/earthporn/submit` returns 404 (browsing); 34 of 109 page fetches were "URL not in corpus" (research) | avoids repeating the same dead end | low; but some failures are temporary |
| 3 | **Intermediate results** (query outputs, computed tables) | the 25 troubled NASDAQ symbols (D2); average rating per business (D5) | a later query that needs the **same sub-result** | medium: must match the exact filter and the data version |
| 4 | **Working procedures** (the sequence that solved it) | search → submit form → fill title, pick forum → create (browsing success) | a later query of the **same task template** | medium: element ids and paths change |
| 5 | **Final answers** | "399001.SZ" (D1); "18.44" (D4) | only the **same or almost the same question** | high: some answers are wrong or depend on a choice |
| 6 | **Outcome labels** (resolved, success, reference answer) | `resolved=0`; `success=false`; reference "Hawthorn" vs agent "Fitzroy" | tells us **which traces to trust** | none; but often missing in real use |
| 7 | **Prompt prefix** (system prompt and tool definitions) | Harvard block ids shared across sessions | serving layer: KV cache across sessions | none for correctness |
| 8 | **Cost data** (tokens, latency, result size) | Harvard `result_tokens`, `latency_ms`, `ttft_ms` | decides whether reuse is **worth it** | none |

The main finding: **the most widely reusable content is not the final answer. It is what the agent learned about the environment on the way.** In every full-text application, a large share of steps is spent on discovery: probing the schema, reading files, finding the right page. These facts are true for many future questions, not just the one that was asked.

Two more findings shape where to look:

- **Trust must be judged per piece of information, not per trace.** Two of three coding traces failed, yet their repo facts are correct. One research trace reports "success" with a wrong answer.
- **In real usage, past traces that share content almost always come from the same user.** In one Harvard week, 32% of sessions start by repeating more than 256 tokens of the same user's earlier content. Only 1 session shares such content with another user.

## Observations by application

### Data analysis (6 DataAgentBench traces)

All six traces follow the same shape: probe the schema → run the main query → post-process in Python → answer. In 5 of 6 traces, schema and value probes (`list_db`, `LIMIT 5`, `DISTINCT`, `COUNT(*)`) are 1 to 4 of the 3 to 12 tool calls.

What a later query could reuse:

- **Value formats that the description does not state.** The benchmark gives each agent a database description and hints. The trace still learns things the description leaves out:
  - D1: `Date` has at least two formats, `"31 Dec 1986, 00:00"` and `"January 02, 1987 at 12:00 AM"`. A strict parse failed. Any later query on `indextrade_database` that filters by date needs this.
  - D6: addresses use `"Indianapolis, IN"`. A filter on `"Indianapolis, Indiana"` returned nothing.
  - D2: `Financial Status` in the data only holds `D`, `H`, `N`, `None`. The hint lists more codes than are present.
- **Join keys.** D6 found that `businessid_52` in Mongo matches `businessref_52` in SQL. D3 matched `bookid_N` to `purchaseid_N` by the number after `_`. These are expensive to find and valid for every query that joins the same two databases.
- **Database layout.** In `stocktrade_database`, every ticker is its own table (D2 had to list 2,753 tables). D4 queried table `REAL` directly. D2 and D4 use the same two databases, so the layout found in one helps the other.
- **Shared sub-results.** D2 computed the list of financially troubled NASDAQ symbols. Any later question about "troubled NASDAQ companies" can start from that list. D5 computed the average rating for every business. That table answers many ranking questions on the same data.
- **Harness protocol.** D5 first opened `var_function-call-....json` as a file name and failed. The right way was to read the path stored in that variable. This is a fact about the agent's own tool system. It helps every later run.

Cautions seen in the data:

- **Answers can depend on a hidden choice.** D6 computed two different averages: 3.86 over all reviews, 3.98 over per-business means. It answered 3.98. A reused answer must carry which definition was used.
- **Some learned facts are guesses.** D3's join by ID number follows a hint that says "fuzzy join". The trace does not check it. A reused fact should carry whether it was checked.
- **Answers are not labeled.** None of the six traces records whether the answer is correct.

### Coding (3 SWE-rebench traces)

Each trace is 45 to 92 tool calls. Most of them read and search; few of them edit.

| Trace | View file / dir | Search / inspect | Env setup | Run tests | Run scripts | Edit / create | Resolved |
|---|---|---|---|---|---|---|---|
| posix-path | 9 | 8 | 0 | 9 | 8 | 5 | yes |
| netcdf | 24 | 6 | 13 | 5 | 18 | 20 | no |
| androidtv | 16 | 20 | 0 | 11 | 7 | 9 | no |

What a later query could reuse:

- **Repo orientation.** Every trace starts by viewing `/workspace`, the README, the test folder, and how to run tests. For a later issue in the same repo at the same commit, this is pure repeat work.
- **Environment setup fixes.** netcdf: `pip install -e .` failed until the agent installed `scikit-build-core` and `setuptools_scm` and used `--no-build-isolation`. This is costly to discover and true for every later task on that repo.
- **Where the relevant code lives.** posix-path read `path_utilities.py` 4 times; netcdf read `__init__.py` 15 times; androidtv read `basetv.py` 5 times. A later issue on the same module starts here.
- **Tool output is the bulk of the context.** Bash output and file views make up 93% to 99% of observation characters in the three traces. Reusing reads saves the most.

Cautions:

- **Two of three traces failed** (`resolved=0`). Their patches should not be reused. Their orientation facts (layout, test command, build fix) are still correct. So **trust must be judged per kind of information, not per trace.**
- netcdf ran `pip install -e .` again after each edit, because the code is compiled. That repeat is needed, not wasted. A reuse system must not skip it.
- All facts depend on the repo commit. The trace records the repo name and version folder, so a key exists.

### Web research QA (3 OpenResearcher traces)

| Trace | Tool calls | Searches (all distinct) | Fetch errors | Same page opened again | Agent answer vs reference |
|---|---|---|---|---|---|
| pharaoh | 22 | 8 | 2 | Fourth Dynasty page × 5 | matches |
| image personalization | 32 | 23 | 3 | ACL paper × 3 | "DreamBooth" vs a longer list of methods |
| football | 109 | 74 | 34 | none | **"Fitzroy" vs reference "Hawthorn"** |

What a later query could reuse:

- **Which sources are reachable.** Most fetch errors are "URL not found in corpus" for guessed Wikipedia URLs. The football task's required Excel file was never reachable. A list of reachable and unreachable URLs saves many steps.
- **Good pages for a topic.** The pharaoh trace kept going back to one page. A later question on the same topic could start there.
- **Facts with citations.** The final answer cites line ranges of pages (for example `【28†L68-L71】`). A fact plus its source can be checked and reused.

Cautions:

- **The run status says "success" even when the answer is wrong.** The football trace has `status=success`, but its answer differs from the reference. Reusing its answer would spread the error.
- **Searches are not repeated word for word.** Every search string is unique, but many are near rewrites (15 of the 23 searches in the image trace mention DreamBooth). Exact-match caching of search queries would miss them. Reuse here needs **matching by meaning**.
- **Cursors and search ids are local.** `cursor: 4`, `web-search://ts=...` only mean something inside one run.

### Web browsing (3 WebArena traces)

What a later query could reuse:

- **Site map and page structure.** The home page is byte-for-byte the same in two traces. The "Create submission" form has the same fields in all three traces: URL/Image radio, Title, Body, Forum combobox, Create button.
- **Site conventions.** Forums live at `/f/<name>`, not `/r/<name>`. The repost trace tried `/r/earthporn/submit` and got 404 three times. The working path to post is `/submit`.
- **Widget quirks.** In the promote trace, the Forum combobox only showed options up to "books". Typing and pressing Enter did not pick "funny". The agent never learned a working way.
- **The successful procedure.** The success trace took 9 steps: search → open the submit form → fill title → pick forum → create → add a comment → stop. This is a plan for the whole task template "post to a forum".

Cautions:

- **Element ids are not stable.** The same "Title" textbox is `[5225]` in one visit and `[799]` in another. Reuse must address elements by role and name, not by id.
- **Failed traces contain loops.** The two failed traces repeat one action 20 or more times (`click [14551]`, `tab_focus 0`). These steps hold no new information. They do show a failure pattern worth storing as a negative fact.

### Harvard FreeInference (metadata only)

These traces have no message text. They cannot tell us *what* was computed, so they cannot show which results to reuse. They do show *where* the work goes and *how much* of the input repeats.

What they contain that helps reuse:

- **Block ids** (16-token chained hashes) show exactly which part of an input was seen before, within and across sessions. See the week numbers below.
- **Tool name, category, programs, and argument shape.** For example `shell_command[*File Ops]` with `cd <path> ; git status ; Get-ChildItem ...`. These show what kinds of steps repeat.
- **Outcome, `result_tokens`, `latency_ms`.** These show the cost of each step, which is the possible saving.
- **User id and harness.** These allow grouping by user. The week scan below shows this matters: cross-session reuse happens almost only within one user.

What they lack:

- **Arguments are anonymized** (`<path>`, `<str>`). Many different calls look the same. A test on the 80 sample sessions shows the problem: in a claude-code session, 17 `webfetch` calls share the same template and the same result size (50 tokens), but they are very likely different pages. So exact repeat detection of tool calls is not possible from this data.
- **No outcome label** for the task as a whole.

#### One full week scanned (2026-08-09 to 2026-08-15)

1,353 sessions, about 164K LLM requests with block ids (subagents included), 2.02 billion input blocks of 16 tokens, 168K tool calls.

**Prompt prefix reuse.** For each request I asked: how long is the leading part of its input that had already appeared before?

| Seen before in | Share of all input blocks |
|---|---|
| this session or any earlier session | 98.8% |
| an earlier session (longest prefix that another session also had) | 18.4% |
| an earlier session **of a different user** | 0.9% |

By harness, the share found in an earlier session ranges from 1.7% (openhands) and 1.9% (zcode) to 26.6% (openclaw) and 35.6% (hermes).

The first request of each session shows cross-session reuse most clearly, because it has no history of its own:

- Median: 85% of the first request was already seen in an earlier session.
- 69% of sessions start with at least half of their first input already seen. 42% start with at least 90% seen.
- Almost all of the system prompt and tool definitions was seen before (median 99.8%).
- In 427 sessions (32%), the shared part goes **more than 256 tokens past** the system prompt and tool definitions. So it includes user messages or history. This looks like resumed or repeated work. Only 1 of these 427 shares that content with a different user.

What this means for reuse:

- **Cross-trace reuse is almost always within one user.** Different users rarely share even the first block. Their tool lists and system prompts differ (installed tools, skills, paths, dates). The block hash uses one salt for the whole week (`code/export/request_inputs/digests.py`), so this is not an effect of anonymization.
- **The user id is a strong key for finding useful past traces.** One third of sessions start by repeating content from the same user's earlier sessions.

**Tool-call repeats.** I matched calls on (tool name, anonymized arguments, outcome, `result_tokens`). This is an upper bound and it is noisy. Small fixed-size results make different calls look the same: by this measure 54% of Edit calls and 61% of Write calls "repeat" inside a session, which is clearly wrong. For reads and searches the numbers are more believable: 10% of Read calls and 2.4% of Search calls repeat inside their session. Exact cross-session repeat detection is not possible with this data.

**Error rates by category**, which show where negative facts would save the most: Interpreter 7.3%, Wait/Poll 7.2%, Other 5.4%, Network 5.1%, Build/Test 4.6%, Read 1.1%, Write 0.9%.

## What is not worth reusing

- **Run-local handles:** storage keys (`var_function-call-...`), browser cursors, search ids with timestamps, element ids, temp file names.
- **Loops and retries** that did not change state.
- **Results of steps that must be redone** after a change, such as a rebuild after an edit.
- **Answers from traces with no correctness label**, or with a wrong one.

## What this means for a reuse design

1. **Store facts, not just answers.** Split each trace into small facts: "column X has formats A and B", "table per ticker", "test command is Y", "URL Z is not in the corpus". These apply to many future queries.
2. **Key each fact by its environment and version.** For example (database, table), (repo, commit), (site, page). A fact is only valid while the version holds.
3. **Keep provenance and status.** For each fact: which trace, which step, was it checked, did the task succeed. Judge trust per fact. A failed trace can still give correct facts.
4. **Match by meaning, not exact text.** Searches and questions are rarely repeated word for word.
5. **Keep the choices behind an answer.** For example, which average was used. Without it, a reused answer can be wrong for a slightly different question.
6. **Use cost data to decide.** Reuse pays off for large, slow results (big query outputs, long builds, many page loads). It does not pay off for small, fast ones (D4 returned one number in 2 calls).

## Open questions

- How often do real future queries hit the same database, repo or site? The full-text samples are too few to say. In Harvard, one third of sessions start by repeating the same user's earlier content, but without text we cannot tell if it is the same repo or the same task.
- How do we detect that an environment changed (new commit, new data) so stored facts expire?
- Can facts be extracted automatically from a trace, and how do we check them?
- For data analysis, how many different questions exist per database in DataAgentBench? This bounds how much environment facts can save.

## Method notes

- Full-text traces were read step by step with small Python scripts; counts in the tables come from those scripts.
- Harvard week scan: `reuse_scan.py` (in this folder). It streams every session of one week in time order. Block ids are chained, so the longest prefix seen before is the largest position whose id was already seen. It counts this against (a) earlier sessions of any user and (b) earlier sessions of other users. Sessions are read in file order, which is roughly start order. Each session is handled as a whole, so sessions that overlap in time are counted approximately. The run took about 20 minutes for one week.
- "Static" prefix = `role_tokens.system + role_tokens.tool_definitions`, divided by 16.
