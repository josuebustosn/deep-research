---
name: deep-research
description: >-
  Multi-engine deep research orchestrator that replicates the claude.ai "Research"
  experience inside Claude Code. Fires Exa Deep Researcher and NotebookLM DeepResearch
  in parallel (with optional papersflow for academic queries and context7 for library
  docs), polls async, pulls full reports, writes a project-relevant SYNTHESIS.md
  cross-validated between engines, and auto-persists outputs to
  .planning/deep-research/YYYY-MM-DD-<slug>/.

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

Claude.ai has a "Research" button that autonomously visits 50+ sources, synthesizes, and returns a structured report. Claude Code has the same underlying capability distributed across separate MCP tools (Exa, NotebookLM, papersflow, context7) — but without orchestration, most users never hit it.

Each engine alone returns useful output. **Running two in parallel and synthesizing the convergences is where the 10x lives** — cross-validation catches hallucinations, complementary coverage expands source diversity, and divergences reveal where the topic is genuinely contested.

Per-engine character (matters for dispatch):

- **Exa `research-pro`** — strong on GitHub issues, community threads, provider docs, specific URLs. Returns structured tables with direct citations. Best when the user wants "what's the evidence?"
- **NotebookLM `deep`** — strong on narrative synthesis, theoretical mechanisms, academic papers, benchmark numbers, *why* something happens. Returns analyst-style long-form. Best when the user wants "help me understand?"
- **papersflow** — academic papers only. Narrow but deep.
- **context7** — official library docs. For API specifics, not research.

## Engine dispatch

**Default for any "deep research" request:** Exa `research-pro` + NotebookLM `deep` in parallel. Extra cost (~$1.30) is trivial vs. the value of cross-validation.

Override or expand based on query type:

| Query contains... | Dispatch |
|---|---|
| "paper", "arxiv", "peer-reviewed", "academic study" | Default + papersflow |
| Specific library/framework/SDK name (React, Next.js, Django, AWS Lambda...) | Try context7 first via find-docs; fallback to default if insufficient |
| "benchmark", "SOTA", "compare models", "which is best", "landscape" | Default (the bread-and-butter case) |
| "community reports", "Reddit discussions", "production experience", "real-world issues" | Default — but Exa is primary (better GitHub issues coverage) |
| "market", "pricing", "providers", "competitive analysis" | Default + consider Exa's `company_research_exa` |
| Query mentions a specific person/founder/company | Default + consider Exa's `linkedin_search_exa` |

If the user provides a flag like `--source=exa`, `--source=nlm`, `--source=all`, honor it.

## Preflight

### 1. NotebookLM auth check (CRITICAL — don't skip)

Call `mcp__notebooklm-mcp__notebook_list` to verify auth. **Do NOT use `server_info`** — that's a local call that returns success even with expired cookies and will silently pass auth check.

If `notebook_list` returns an auth error:

1. Tell the user: "La sesión de NotebookLM expiró. Voy a relanzar el login — se abrirá Chrome."
2. Launch `nlm login` via Bash with `run_in_background=true`
3. Wait for user confirmation that they completed the OAuth flow in the browser
4. Call `mcp__notebooklm-mcp__refresh_auth` — **this step is mandatory**. The MCP caches credentials in memory at startup and won't auto-reload from disk after `nlm login` writes new tokens. Without `refresh_auth`, `notebook_list` keeps returning "Authentication expired" even though cookies are valid on disk.
5. Re-verify with `notebook_list`
6. Only then proceed with research

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

### Exa `deep_researcher_start`

Exa likes **structured, explicit instructions**. Template:

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

Use `model=exa-research-pro` by default. Fall back to `exa-research` (balanced, 15-45s) if the user wants faster/cheaper, or `exa-research-fast` for simple queries.

### NotebookLM `research_start`

NotebookLM prefers **narrative, exploratory queries** — write like you're briefing an analyst. Single flowing question with sub-points baked in. Example:

```
Which <things to compare> support <capability>, and why? Evaluate specifically
<items>. For each: confirmed support, known issues, evidence from <source types>.
Critical question: <the "why" question that forces synthesis>.
```

Use `mode=deep` (5 min, 40 sources, web only) by default. Use `mode=fast` only when the user explicitly needs it in 30s.

Use `source=web` unless the research is against Google Drive sources the user has configured.

Title the notebook descriptively: `<Project Name> - <Topic> <Q/Year>`. NotebookLM creates a new notebook per research — the title helps the user find it later in notebooklm.google.com.

### papersflow

Academic keywords + domain framing. Less verbose than Exa. Focus on the theoretical question.

### context7 (via find-docs)

Only for library API / framework documentation. Not general research.

## Launching and polling

**Launch in a single message with parallel tool calls.** All engines are async — don't wait for one before starting the next.

### Polling strategy

- **Exa:** `deep_researcher_check` returns instantly with current status. Poll every ~40 seconds (inside the 5-min prompt cache window). Research-pro takes 45s to 3 min typically.
- **NotebookLM:** `research_status` with `max_wait=180` blocks server-side for up to 3 minutes polling internally — this is MORE efficient than local polling, since it holds the connection open. In `deep` mode, the task_id may change partway through the run — always pass `query` as fallback for task matching per the MCP docs.
- **Ideal pattern:** fire both checks in the same message. NLM blocks up to 180s; Exa returns instant status. If Exa not done when NLM returns, do another pair.

### CRITICAL: compact mode trap

`research_status` defaults to `compact=true`, which **truncates the report to the first few hundred characters**. Once status is `completed`, call it again with `compact=false` to pull the full report. Skipping this loses 70%+ of the content silently and is the most common way to undervalue NotebookLM.

### Timeout handling

- Exa >5 min: likely stuck. Decide with user whether to proceed with partial or abort.
- NotebookLM deep >8 min: check `notebooklm.google.com/notebook/<notebook_id>` manually — occasionally the web UI finishes while the API hangs. User can copy-paste as fallback.

## Persistence

Write four files to `<workspace>/<date>-<slug>/`:

### 1. `exa-report.md` — raw Exa output

Header block:

```markdown
# Exa research-pro Report — <Title>

**Engine:** Exa Deep Researcher (`<model>`)
**Research ID:** <researchId>
**Date:** <YYYY-MM-DD>
**Duration:** ~<N> minutes
**Cost:** $<X.XX> USD
**Pages browsed:** <N>
**Searches performed:** <N>
**Citations:** <N>

## Query
<the full instructions block sent to Exa>

## Report
<the raw report text>

## Citations
<numbered list of all URLs with titles>
```

### 2. `nlm-report.md` — raw NotebookLM output

Same header style. Include `Notebook URL: https://notebooklm.google.com/notebook/<id>` so the user can inspect sources in the UI.

### 3. `SYNTHESIS.md` — project-relevant distillation

**This is the file humans read.** Structure:

```markdown
# Synthesis — <Topic> (<date>)

**Investigación:** <one-line question>
**Engines:** Exa <model> (<N> sources, $<cost>) + NotebookLM <mode> (<N> sources, free)
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
3. **Cost** (Exa $ + NLM $0)
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
- `--mode=fast` — NotebookLM fast mode (30s, 10 sources) instead of deep
- `--model=exa-research` — balanced Exa model instead of `exa-research-pro`
- `--model=exa-research-fast` — fastest Exa, 15s
- `--no-persist` — skip file persistence, just return findings inline
- `--slug=<custom>` — override auto-generated slug

## Edge cases

**Ambiguous query** — if the query is vague ("investiga esto a fondo"), ask ONE clarifying question before firing engines. Don't waste $1.30 on a bad query.

**Missing `.planning/` folder** — fall back to `./deep-research/` at cwd. Don't fail.

**One engine returns empty/error** — proceed with the one that worked. Note the failure under SYNTHESIS.md section "Engines used" with the error message. Do not silently pretend both ran.

**Both engines fail** — abort persistence, tell the user, suggest either `--source=fast`, a more specific query, or manual fallback via `WebSearch` + `WebFetch`.

**Query involves proprietary/private info** — do NOT send identifying details (internal project names, customer names, secret keys) to Exa/NLM. Both index or cache queries. Anonymize before firing. If the user insists on including private info, warn them first and get explicit consent.

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
