# Architecture Overview

## System Purpose

This system automates SEO content production end-to-end using two sequential n8n workflows.
It receives a content request via webhook, researches the topic across multiple data sources,
builds a structured briefing document, and then writes, reviews, and delivers a final article —
all without human intervention between steps.

---

## Workflow 1 — Briefing

### Entry point

The workflow is triggered by a **POST webhook** receiving a JSON payload with:
`Cliente`, `Domínio`, `Tópico`, `Palavra-chave principal`, `Objetivo`, `Tom de voz`, `Squad`, and `num_task_id`.

Before any processing begins, the system checks API credit availability for both
**Google Serper** and **Firecrawl**. If either falls below the defined threshold,
the workflow responds with a failure message and sends a Discord alert.
Only if both APIs have sufficient credits does execution continue.

---

### Stage 1 — Setup

| Step | What happens |
|---|---|
| Credit check | Serper balance ≥ 70 and Firecrawl remaining credits ≥ 100 |
| Squad routing | Switch node maps the squad name to its corresponding Google Drive folder ID |
| Temporary documents | Creates a Google Docs briefing draft and a Google Sheets spreadsheet with 4 tabs: `SERP_concorrentes`, `Dados_Mercado`, `Posicionamento_da_marca`, `links_internos` |
| Webhook response | Returns `"Workflow iniciado com sucesso"` immediately — the rest runs async |

---

### Stage 2 — Internal links

A lightweight agent identifies 5–10 semantically relevant internal links for the target domain.
It reads pre-selected URLs from the `links_internos` sheet and generates anchor texts and
insertion positions for each link. Uses **Claude Haiku** (lightweight model for a deterministic task).
Output is written to the briefing Google Doc.

---

### Stage 3 — SERP analysis

The main keyword is first translated to English using a **Basic LLM Chain** (Claude Haiku).
Then two parallel SERP calls are made — **Brazil (pt-br)** and **Global (en-us)** — each returning 5 results.

All 10 results are combined and crawled via **Firecrawl** to extract:
headings (H1–H4), meta descriptions, page summaries, topics, and questions per URL.

A **SERP Extractor agent** (Claude Sonnet) structures this data into a clean JSON.
A **SERP Analysis agent** (Claude Sonnet) then interprets the combined dataset, identifying:
dominant search intent, competitor patterns, content gaps, and 10–20 real user questions.

Structured data is saved to the `SERP_concorrentes` sheet.
Analysis output is written to the briefing Google Doc.

---

### Stage 4 — Semrush analysis

An **AI Semrush Analyzer agent** (Claude Sonnet) calls three MCP tools:
`buscar_dados_palavra_chave_semrush`, `palavras_chave_relacionadas_semrush`, and `encontrar_concorrentes_semrush`.

It returns volume, CPC, competition, trend, 25 secondary keywords, competitors, and semantic variations —
all scoped to Brazil. Output is written to the briefing Google Doc.

---

### Stage 5 — Data statistics

An **AI Query Generator agent** (Claude Haiku) produces 5 optimized search queries
targeting market statistics related to the topic. Each query runs against **Google Serper**
in a loop, collecting up to 50 organic results across all queries.

A **Data Extractor agent** (Claude Sonnet) structures every result — title, snippet, link,
date, position, and statistics — into a normalized JSON (one item per source, no skipping).

A second set of queries searches the client's own domain to extract their published
statistics and positioning. Results are saved to the `Dados_Mercado` and `Posicionamento_da_marca` sheets.

A **Market Data agent** (Claude Sonnet) consolidates results, identifies statistical consensus
(≥ 3 sources citing the same theme = consensus), deduplicates, and filters by recency and source authority.

A **Brand Positioning agent** (Claude Sonnet) extracts concepts and statistics the client
has already published publicly.

A **Market vs Client agent** (Claude Sonnet) produces a single diagnostic paragraph comparing
where the brand is strong versus where competitors dominate.

All outputs are sequentially written to the briefing Google Doc.

---

### Stage 6 — Final briefing structure

An **AI Content Structure agent** (Claude Sonnet, 15k tokens) reads the entire briefing
Google Doc and produces a final structured JSON containing:

- `briefing_markdown` — full editorial brief with SERP analysis, competitor table, search intent, market data, Semrush data, FAQ, internal links, CTAs, and visual recommendations
- `h1`, `meta_title`, `meta_description`
- `content_structure` — H2/H3 hierarchy with detailed writing instructions per section
- `faq_section`
- `estimated_total_words`

This output is written to a **new, permanent Google Doc** named with the pattern:
`[Brand] BRIEFING - "keyword" - YYYY-MM`

Status is updated in the DataTable. Temporary files (briefing draft + Sheets spreadsheet)
are deleted from Google Drive. Workflow 2 is then called asynchronously.

---

### Workflow 1 — Pipeline diagram

```
Webhook (POST)
  │
  ├── Credit check: Serper + Firecrawl
  │     └── Fail → Discord alert + error response
  │
  ├── Squad routing → Google Drive folder assignment
  ├── Create: Google Docs (draft) + Google Sheets (4 tabs)
  ├── Respond to webhook: "iniciado com sucesso" (async from here)
  │
  ├── [Stage 2] Internal links agent
  │     └── Writes anchor texts + positions → briefing doc
  │
  ├── [Stage 3] SERP BR (5 results) + SERP Global (5 results)
  │     └── Firecrawl crawl → headings, meta, topics, questions
  │     └── SERP Extractor → SERP_concorrentes sheet
  │     └── SERP Analysis agent → briefing doc
  │
  ├── [Stage 4] Semrush MCP agent (3 tools)
  │     └── Volume, CPC, keywords, competitors → briefing doc
  │
  ├── [Stage 5] Query generator (5 queries) → Serper loop
  │     ├── Data Extractor (50 results) → Dados_Mercado sheet
  │     ├── Brand SERP loop → Posicionamento_da_marca sheet
  │     ├── Market Data agent (consensus + dedup)
  │     ├── Brand Positioning agent
  │     └── Market vs Client agent → briefing doc
  │
  └── [Stage 6] Content Structure agent
        ├── Reads full briefing doc
        ├── Writes final BRIEFING doc (permanent)
        ├── Deletes temp Google Docs + Sheets
        └── Calls Workflow 2 (async)
```

---

## Workflow 2 — Writing

### Entry point

Called by Workflow 1 via `executeWorkflow`, receiving:
`idbriefing`, `Marca`, `tom_de_voz`, `task_id`, `Tópico`, and `id_folder`.

---

### Stage 1 — Writing

A **Redator agent** (Claude Sonnet 4, 14k tokens) reads the final briefing doc via the `ler_briefing` tool
and produces a complete article in natural Brazilian Portuguese, including:

- Title (≤ 60 chars), meta description (140–156 chars), H1, H2/H3 structure
- Narrative and technical depth per section, following the briefing's `content_structure`
- Internal links delivered in `anchor text (URL)` format — never bare URLs
- A mandatory "Fontes e Referências" section at the end
- Zero external research — only what the briefing contains

Output is saved to a new **"Texto para revisar"** Google Doc.

---

### Stage 2 — Voice and tone review

A **Revisor de Tom de Voz agent** (Claude Sonnet 4, 14k tokens) reads the draft
via the `ler_texto` tool and applies the brand's tone of voice guide (provided as a Google Docs URL).

Rules enforced: no anglicisms, natural logical connectors, no content removal,
paragraph cohesion, mandatory introductory paragraph after every heading, link format preserved.

Output overwrites the draft content in the same Google Doc.

---

### Stage 3 — Fluency review

A **Revisor de Fluidez agent** (Claude Sonnet 4, 14k tokens) reads the tone-reviewed text
via the `ler_texto1` tool and polishes sentence rhythm, transitions, and readability.

Rules enforced: no content changes whatsoever, no new sections created, heading capitalization
corrected (only first word capitalized), all links and references preserved intact.

Output is written to a new **"Texto final"** Google Doc — the final deliverable.

---

### Workflow 2 — Pipeline diagram

```
Called by Workflow 1 (async)
  │
  ├── [Stage 1] Redator agent (Claude Sonnet 4, 14k tokens)
  │     ├── Reads: final BRIEFING doc via ler_briefing tool
  │     └── Creates + writes: "Texto para revisar" Google Doc
  │
  ├── [Stage 2] Revisor de Tom de Voz agent (Claude Sonnet 4, 14k tokens)
  │     ├── Reads: "Texto para revisar" doc + brand tone guide
  │     └── Overwrites: same doc with tone-adjusted content
  │
  └── [Stage 3] Revisor de Fluidez agent (Claude Sonnet 4, 14k tokens)
        ├── Reads: tone-adjusted doc via ler_texto1 tool
        └── Creates: "Texto final" Google Doc ← final deliverable
```

---

## Model selection rationale

| Task | Model | Reason |
|---|---|---|
| Internal link queries, keyword translation | Claude Haiku | Deterministic output, no reasoning needed — lowest cost |
| SERP extraction, SERP analysis | Claude Sonnet 4.5 | Structured interpretation across 50+ data points |
| Semrush analysis, Market Data, Brand Positioning | Claude Sonnet 4.5 | Multi-source reasoning with MCP tool calls |
| Content Structure (briefing) | Claude Sonnet 4.5, 15k tokens | Long structured JSON generation requiring consistency |
| Writing, Tone review, Fluency review | Claude Sonnet 4, 14k tokens | Complex long-form generation — highest quality model for final output |

This tiered model strategy — lighter models for deterministic tasks, heavier models for
generative and interpretive tasks — is the primary driver of the 66% cost reduction
compared to the original architecture where all agents used the same model.

---

## Tech Stack

| Layer | Tools |
|---|---|
| Workflow orchestration | n8n (cloud-hosted) |
| LLM provider | Anthropic (Claude Haiku 4.5, Claude Sonnet 4.5, Claude Sonnet 4) |
| Search data | Google Serper API |
| Web crawling | Firecrawl API |
| SEO data | Semrush via MCP server |
| Document storage | Google Docs + Google Drive |
| Structured data | Google Sheets |
| Notifications | Discord (webhook) |
| Entry point | Bubble (external app) → POST webhook |

---

## Workflow Files

The full n8n workflows are available at:
- [`workflows/briefing_workflow.json`](./workflows/briefing_workflow.json)
- [`workflows/writing_workflow.json`](./workflows/writing_workflow.json)

To import: open n8n → top-right menu → Import from file → select the JSON.

> Note: credentials and sensitive data have been removed from the exported workflows.
> Replace placeholder credential nodes with your own API keys before running.