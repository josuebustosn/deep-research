---
name: deep-research
description: >-
  Multi-engine deep research orchestrator that replicates the claude.ai "Research"
  experience inside Claude Code. Fires the Exa Agent (MCP tool `agent_run`) and
  NotebookLM Deep Research (via the `notebooklm-py` CLI) in parallel (with optional
  papersflow for academic queries and context7 for library docs), polls async, pulls
  full reports, writes a project-relevant SYNTHESIS.md cross-validated between engines,
  and auto-persists outputs to .planning/deep-research/YYYY-MM-DD-<slug>/.

  Use this skill whenever the user says "deep research", "investigación profunda",
  "investigación exhaustiva", "research exhaustivo", "investiga a fondo",
  "lanza investigación", "comprehensive research", "multi-source research",
  "quiero investigación profunda", "investigación cruzada", or types /deep-research.

  Also use proactively when the user asks a question that cannot be answered well
  by a single WebSearch because it requires cross-validating evidence from 20+
  sources (provider docs + Reddit + GitHub issues + benchmarks + papers), or when
  the conversation is about choosing between technologies / models / providers /
  MCPs / tools and needs empirical backing from multiple independent sources, or
  when the user wants a "second opinion" on a research question.

  Do NOT use for: quick factual lookups (single WebSearch suffices), library API
  questions (use find-docs / context7 directly), questions answerable from project
  files, code review, or debugging specific errors.
---

# Deep Research

Multi-engine deep research orchestrator. You are the conductor — the engines do the heavy lifting, and your job is to dispatch the right combination, synthesize cross-validated findings, and persist everything cleanly.

## Why this exists

Claude.ai has a "Research" button that autonomously visits 50+ sources, synthesizes, and returns a structured report. Claude Code has the same underlying capability distributed across separate tools (the Exa MCP, the `notebooklm-py` CLI, papersflow, context7) — but without orchestration, most users never hit it.

Each engine alone returns useful output. **Running two in parallel and synthesizing the convergences is where the 10x lives** — cross-validation catches hallucinations, complementary coverage expands source diversity, and divergences reveal where the topic is genuinely contested.

Per-engine character (matters for dispatch):

- **Exa Agent (`agent_run`)** — strong on GitHub issues, community threads, provider docs, pricing pages, specific URLs. Returns structured tables with direct citations (`grounding`). Best when the user wants "what's the evidence?"
- **NotebookLM `deep`** — strong on narrative synthesis, theoretical mechanisms, academic papers, benchmark numbers, *why* something happens. Returns analyst-style long-form. Best when the user wants "help me understand?"
- **papersflow** — academic papers only. Narrow but deep.
- **context7** — official library docs. For API specifics, not research.

## Engine dispatch

**Default for any "deep research" request:** Exa `agent_run` (`effort=high`) + NotebookLM `deep` in parallel. Extra cost (Exa ≈ $0.50 USD per run at `effort=high`, measured; NotebookLM $0) is trivial vs. the value of cross-validation.

Override or expand based on query type:

| Query contains... | Dispatch |
|---|---|
| "paper", "arxiv", "peer-reviewed", "academic study" | Default + papersflow |
| Specific library/framework/SDK name (React, Next.js, Django, AWS Lambda...) | Try context7 first via find-docs; fallback to default if insufficient |
| "benchmark", "SOTA", "compare models", "which is best", "landscape" | Default (the bread-and-butter case) |
| "community reports", "Reddit discussions", "production experience", "real-world issues" | Default — but Exa is primary (better GitHub issues coverage) |
| "market", "pricing", "providers", "competitive analysis" | Default (consider `effort=xhigh`) + Exa's `web_search_advanced_exa` with category filters for targeted company lookups |
| Query mentions a specific person/founder/company | Default + Exa's `web_search_advanced_exa` (people/company categories) |

`company_research_exa`, `linkedin_search_exa` and `deep_search_exa` were deprecated upstream and replaced by `web_search_advanced_exa`. The old `deep_researcher_start` / `deep_researcher_check` tools no longer exist — the research product is now the Exa Agent (`agent_run`).

If the user provides a flag like `--source=exa`, `--source=nlm`, `--source=all`, honor it.

## Preflight

### 0. Engine discovery (every user's setup is different — detect, don't assume)

Tool and server names depend on each user's config. Detect what's available before dispatching:

- **Exa (preferred: MCP).** Run `ToolSearch` with the query `agent_run`. The tool is named `mcp__<server-name>__agent_run` (usually `mcp__exa__agent_run`). Load it with `ToolSearch select:<exact name>` before calling it.
  - **Only an `authenticate` tool shows up for the Exa server** → the hosted MCP (`https://mcp.exa.ai/mcp`) needs OAuth. Call its `authenticate` tool and give the URL to the user **immediately** — the localhost callback listener is short-lived and a stale link silently fails. If the browser lands on a connection error after authorizing, ask the user to paste the full `http://localhost:<port>/callback?...` URL from the address bar and pass it to the server's `complete_authentication` tool. The real tools appear automatically once auth completes.
  - **No Exa MCP, but `EXA_API_KEY` is set** → use the REST fallback (see "Launching and polling").
  - **Neither** → skip Exa, run NotebookLM only, and note the missing engine in SYNTHESIS.md.
  - If the user's MCP URL restricts tools with `?tools=...`, `agent_run` must be in that list (enabling optional tools replaces the default set).
- **NotebookLM (via the `notebooklm-py` CLI).** Run `command -v notebooklm && notebooklm --version`. If missing, offer to install it (`uv tool install notebooklm-py`, `pipx install notebooklm-py`, or `pip install notebooklm-py`) and then run the auth flow below. If the user declines, fall back to `--source=exa`.
  - This skill does **not** use the legacy `notebooklm-mcp-cli` package (`nlm`, `mcp__notebooklm-mcp__*`). Don't call those tools here even if they're configured.

### 1. NotebookLM auth check (CRITICAL — don't skip)

```bash
notebooklm auth check --test --json
```

Require **both** `"status": "ok"` and `"checks": {"token_fetch": true}`. This is the only check that makes a network call and proves the cookies still authenticate against Google.

**Do NOT use `notebooklm doctor` or `notebooklm status` as an auth check** — both are local. `doctor` can print "All checks passed" while the session is expired (observed 2026-09-18: `doctor` passed, then `notebooklm list` failed with "Authentication expired or invalid").

If the check fails:

1. Try the cheap path first: `notebooklm auth refresh` (server-side cookie refresh), then re-run the check.
2. If that fails too, tell the user: "La sesión de NotebookLM expiró. Voy a relanzar el login — se abrirá una ventana del navegador."
3. Launch `notebooklm login` via Bash with `run_in_background=true`. It opens a browser with a persistent profile and waits up to 5 minutes; the only human step is completing the Google sign-in. It exits by itself once login is detected (`Authentication saved to ...storage_state.json`).
4. Re-run `notebooklm auth check --test --json`. No extra "reload" step is needed — the CLI reads the stored session on every call.
5. Only then proceed with research.

If the user declines to re-auth, fall back to `--source=exa` and proceed with a single engine. Note the missing engine in SYNTHESIS.md.

### 2. Generate slug + date

**Slug rules:** kebab-case, 3-6 words, describes the question not the answer.

Examples:
- "investiga si Qwen2.5 abliterated soporta tool calling" → `abliterated-tool-calling`
- "Bifrost vs LiteLLM para producción" → `bifrost-vs-litellm-bench`
- "¿Qué MCPs de Reddit existen?" → `reddit-mcp-landscape`
- "Latencia WireGuard + Hetzner" → `hetzner-wireguard-latency`

**Date:** today in `YYYY-MM-DD`.

### 3. Resolve workspace path

Preferred: `<project-root>/.planning/deep-research/<date>-<slug>/`

Fallback (no `.planning/` in project): `./deep-research/<date>-<slug>/`

Use `Glob` to check for `.planning/` in the current working directory. If found, use it. If not, use the fallback. Create the folder with `mkdir -p` via Bash.

## Query formatting per engine

Engines respond better to different prompt styles. Adapt the user's intent into the right format for each.

### Exa `agent_run`

Exa likes **structured, explicit instructions**. Pass this template as `query`:

```
<THE RESEARCH QUESTION IN ONE LINE>

CONTEXTO DEL PROYECTO: <1-2 sentences explaining what the user is building,
why this matters, and any relevant constraints (provider pool, tech stack, etc.)
Copy from .planning/PROJECT.md if applicable>

ITEMS TO EVALUATE: <numbered list if the query is a comparison>

QUESTIONS TO ANSWER:
- <sub-question 1>
- <sub-question 2>
- <sub-question 3>

SOURCES TO PRIORITIZE:
- <type 1: official docs, arXiv, GitHub issues, r/LocalLLaMA, etc.>

OUTPUT FORMAT: <describe the ideal structure — table by item with columns X/Y/Z,
synthesis with ranking, concrete evidence bullets>
```

Other parameters:
- `systemPrompt` — persona and ground rules (e.g., "cite a URL for every claim, prefer sources from the last 12 months, say so when a price isn't found").
- `effort` — `high` by default. `xhigh` for broad landscapes / many providers; `medium` for narrow questions. (The REST API also offers `max` behind the beta header `agent-max-effort-2026-07-27`; don't use it by default.)
- `outputSchema` — optional JSON Schema when you need a machine-readable table.

### NotebookLM (`notebooklm source add-research`)

NotebookLM prefers **narrative, exploratory queries** — write like you're briefing an analyst. Single flowing question with sub-points baked in. Example:

```
Which <things to compare> support <capability>, and why? Evaluate specifically
<items>. For each: confirmed support, known issues, evidence from <source types>.
Critical question: <the "why" question that forces synthesis>.
```

Write the query to `<workspace>/nlm-query.md` and pass it with `--prompt-file` (long multi-line prompts break shell quoting).

Use `--mode deep` (~40–50 sources, web only) by default. Use `--mode fast` only when the user explicitly needs it in ~30s.

Use `--from web` (the default) unless the research is against Google Drive sources the user has configured.

Title the notebook descriptively: `<Project Name> - <Topic> <Q/Year>`. Each research gets its own notebook — the title helps the user find it later in notebooklm.google.com.

### papersflow

Academic keywords + domain framing. Less verbose than Exa. Focus on the theoretical question.

### context7 (via find-docs)

Only for library API / framework documentation. Not general research.

## Launching and polling

**Launch in a single message with parallel tool calls.** All engines are async — don't wait for one before starting the next.

### Launch

- **Exa (MCP):** call `agent_run` with `query`, `systemPrompt` and `effort`. It returns an `id` (`agent_run_...`) and usually `status: "running"`.
- **NotebookLM:** each Bash call is a fresh shell, so capture the notebook ID from the output and reuse it literally in later calls.

  ```bash
  export PYTHONUTF8=1
  notebooklm create "<Project> - <Topic> <Q/Year>" --json          # → .notebook.id
  notebooklm source add-research --prompt-file "<workspace>/nlm-query.md" \
    --mode deep --no-wait -n <NOTEBOOK_ID> --json                   # → status "started"
  ```

  Then, in a **separate** Bash call with `run_in_background=true`:

  ```bash
  PYTHONUTF8=1 notebooklm research wait -n <NOTEBOOK_ID> --timeout 1800 --import-all --cited-only --json
  ```

- **Exa (REST fallback, only when there's no MCP and `EXA_API_KEY` is set):**

  ```bash
  curl -s -X POST https://api.exa.ai/agent/runs -H "x-api-key: $EXA_API_KEY" \
    -H "Content-Type: application/json" -d @<workspace>/exa-request.json   # {"query": ..., "effort": "high"}
  ```

### Polling strategy

- **Exa (MCP):** call `agent_run` again with **only** `runId` — never resend `query`, that starts a new paid run. Keep calling every 1–2 minutes until `outputReady: true`. The final payload has `output.text` (the report), `output.grounding[].citations` (sources), `usage.searches` and `costDollars.total`. Reference run (2026-09-18, `effort=high`): ~10–15 min, 60 searches, $0.50.
- **Exa (REST):** `GET https://api.exa.ai/agent/runs/<id>` every few seconds until `status` is `completed`, `failed` or `cancelled`.
- **NotebookLM:** the background `research wait` notifies you when it finishes — don't poll in a loop. Reference run (2026-09-18, `--mode deep`): ~6 min, 54 sources found, 30 cited sources imported.
- **Ideal pattern:** launch both, keep working (or talking with the user) while they run, check Exa with `runId` whenever you're back, and pull the NotebookLM report when its background task completes.

### Pull the full NotebookLM report

When `research wait` completes, write a clean JSON with the full report:

```bash
PYTHONUTF8=1 notebooklm research status -n <NOTEBOOK_ID> --json > "<workspace>/nlm-raw.json"
```

Keys: `task_id`, `status`, `query`, `summary`, `report` (the full markdown report, typically 20–40k chars), `sources[]` (`url`, `title`, `result_type`, `report_markdown`), `tasks`. Build `nlm-report.md` from `report` + `sources`.

### CRITICAL: notebooklm-py traps

1. **Encoding.** On Windows, without `PYTHONUTF8=1` the CLI writes JSON through the console code page and every accented character becomes `�` — the report is silently corrupted. Always prefix NotebookLM calls that produce `--json` with `PYTHONUTF8=1` (harmless on macOS/Linux).
2. **Timeout.** `research wait` defaults to `--timeout 300`, which is too short for deep mode. If it gives up early, nothing gets imported and the web UI is left showing an "Add sources?" modal. Use `--timeout 1800`.
3. **Blocking.** `source add-research` without `--no-wait` blocks the whole call. Always `--no-wait` + a separate background `research wait`.
4. **Auth checks.** `doctor` / `status` are local (see Preflight 1).
5. **Binary name collision (only if you also use MCP servers).** Both `notebooklm-py[mcp]` and the legacy `notebooklm-mcp-cli` install an executable named `notebooklm-mcp`. This skill uses the `notebooklm` CLI, so it isn't affected. If you configure the notebooklm-py MCP server, launch it unambiguously with `uvx --from "notebooklm-py[mcp]" notebooklm-mcp`.

### Timeout handling

- Exa still running after ~25 min: ask the user whether to keep waiting or cancel. Cancelling is only exposed on the REST API (`POST /agent/runs/<id>/cancel`) and usage accrued so far is still billed.
- NotebookLM `research wait` times out: run `notebooklm research status -n <NOTEBOOK_ID> --json`; if it's still in progress, relaunch the background wait. Occasionally the web UI finishes while the API lags — the user can check `https://notebooklm.google.com/notebook/<NOTEBOOK_ID>` and copy-paste as fallback.

## Persistence

Write four files to `<workspace>/<date>-<slug>/`:

### 1. `exa-report.md` — raw Exa output

Header block:

```markdown
# Exa Agent Report — <Title>

**Engine:** Exa Agent (`agent_run`, effort=<effort>)
**Run ID:** <agent_run_...>
**Date:** <YYYY-MM-DD>
**Duration:** ~<N> minutes
**Cost:** $<costDollars.total> USD
**Searches performed:** <usage.searches>
**Citations:** <N from output.grounding>

## Query
<the full query sent to Exa, plus the systemPrompt>

## Report
<output.text>

## Citations
<numbered list of all URLs with titles from output.grounding>
```

### 2. `nlm-report.md` — raw NotebookLM output

Same header style. Include `Notebook URL: https://notebooklm.google.com/notebook/<id>` so the user can inspect sources in the UI, plus the task ID and the number of sources found/imported.

### 3. `SYNTHESIS.md` — project-relevant distillation

**This is the file humans read.** Structure:

```markdown
# Synthesis — <Topic> (<date>)

**Investigación:** <one-line question>
**Engines:** Exa agent_run effort=<effort> (<N> searches, $<cost>) + NotebookLM deep (<N> sources, free)
**Triggered by:** <what prompted this — blocker, discussion, decision point>
**Status:** <one of: HIGH CONFIDENCE / MIXED / DIVERGENT / INCOMPLETE>

---

## 1. Key finding(s) — dual-engine agreement

<The 1-3 most important convergent findings, cited with evidence from both engines>

## 2. Ranking / comparison / primary answer

<A table or ordered list answering the core question. Columns include
confidence level per item.>

## 3. Provider-level / context-level quirks

<Any nuances that matter in specific deployment contexts — "Fireworks handles
this fine, NanoGPT doesn't", etc.>

## 4. Impact on project

<Scan PROJECT.md / ROADMAP.md / active phase CONTEXT.md for references to the
topic. If any existing Decision needs revision, call it out here with proposed
delta. Do NOT silently edit project files.>

## 5. New considerations

<Architectural/tooling implications that emerged from the research but weren't
in the original question.>

## 6. Followups / open questions

<What this investigation did NOT resolve — genuine gaps, worth noting for
future research.>

## Cost summary
- Exa: $<cost>
- NotebookLM: $0
- Total: $<total>
```

### 4. `sources.md` — deduplicated + categorized source list

Merge citations from both engines. Categorize by source type:
- Benchmarks & Leaderboards
- Provider Documentation
- Model Cards / Repos
- GitHub Issues
- Academic Papers
- Community Discussions (Reddit / HN / Discord)
- Technical Blogs

Useful for future citation lookup without re-reading full reports.

### 5. Update the parent README index

If `<workspace>/../README.md` exists (e.g., `.planning/deep-research/README.md` with an Index table), append a new row:

```
| <date> | `<slug>` | <topic> | <trigger> |
```

Do not edit the README if it doesn't exist — just skip.

## Project impact scanning

Before writing SYNTHESIS.md, scan the project for references to the research topic:

```bash
# Grep for topic keywords in planning docs
grep -ri "<keyword>" .planning/PROJECT.md .planning/ROADMAP.md 2>/dev/null

# Check active phase artifacts
ls .planning/phases/*/

# Look at the current phase CONTEXT + RESEARCH
```

If a Decision (e.g., "Decision #5 — Primary Coding Model" in PROJECT.md) mentions something the research contradicts, explicitly list it under SYNTHESIS.md section 4 with proposed revision text. **Do NOT silently update PROJECT.md or other canonical docs** — always present the proposed change to the user for approval as a second, explicit step.

## Reporting back to the user

After persistence, give a tight summary:

1. **One-line verdict** — the most important finding, not a hedge
2. **Files written** (absolute paths for clickability)
3. **Cost** (Exa `costDollars.total` + NLM $0)
4. **Decisions that may need revisiting** (surface from project impact scan)
5. **Ask:** "¿Quieres que actualice los archivos del proyecto con los hallazgos, o lo dejamos como deep-research archive por ahora?"

Default to asking for permission before touching `PROJECT.md` / `ROADMAP.md` / canonical research files. SYNTHESIS.md is always written; updates to project docs require explicit user consent.

Match the user's communication style from the conversation — if they've been terse, be terse. If they've been conversational, be conversational. Use the same language they used for the query (Spanish/English).

## Flags

Override defaults with flags:

- `--source=exa` — Exa only (if NotebookLM auth is being painful)
- `--source=nlm` — NotebookLM only (cheapest, if cost matters)
- `--source=papers` — add papersflow (for academic topics)
- `--source=all` — Exa + NLM + papersflow + context7 where applicable
- `--mode=fast` — NotebookLM fast mode (~30s, ~10 sources) instead of deep
- `--effort=<minimal|low|medium|high|xhigh>` — Exa Agent effort (default `high`)
- `--no-persist` — skip file persistence, just return findings inline
- `--slug=<custom>` — override auto-generated slug

## Edge cases

**Ambiguous query** — if the query is vague ("investiga esto a fondo"), ask ONE clarifying question before firing engines. Don't waste an Exa run (~$0.50 at `effort=high`) on a bad query.

**Missing `.planning/` folder** — fall back to `./deep-research/` at cwd. Don't fail.

**One engine returns empty/error** — proceed with the one that worked. Note the failure under SYNTHESIS.md section "Engines used" with the error message. Do not silently pretend both ran.

**Both engines fail** — abort persistence, tell the user, suggest either `--mode=fast`, a more specific query, or manual fallback via `WebSearch` + `WebFetch` (or parallel research subagents).

**Query involves proprietary/private info** — do NOT send identifying details (internal project names, customer names, account numbers, secret keys) to Exa/NLM. Both index or cache queries. Anonymize before firing. If the user insists on including private info, warn them first and get explicit consent.

**Query is about information that changes rapidly** (e.g., "current price of X", "is Y service up") — deep research is wrong tool. Redirect to `WebSearch` or appropriate monitoring MCP.

## What this skill does NOT do

- Quick factual lookups → use `WebSearch` directly
- Library API / framework questions → use `find-docs` (context7)
- Code review / debugging specific errors → wrong tool
- Writing research *content* from scratch → this is investigation, not authoring
- Summarizing already-existing research files in the project → just `Read` + summarize, no engines needed
- Queries about the user's own project state → read project files directly

## Reference

When you need the workspace convention for `.planning/deep-research/`, see the archive README at `.planning/deep-research/README.md` — it documents the folder layout, slug rules, and relationship to canonical `.planning/research/`.

For an end-to-end example of what SYNTHESIS.md + exa-report.md + nlm-report.md + sources.md look like in practice, see the first investigation persisted under that convention: `.planning/deep-research/2026-04-16-abliterated-tool-calling/`.
