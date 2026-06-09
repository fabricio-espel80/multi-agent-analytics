# Multi-Agent Analytics

A conversational analytics system for marketing data built with **Google ADK 2.0**. A root coordinator agent receives questions in natural language and delegates to specialized sub-agents, each with deep expertise in a specific data domain.

## Architecture

```
User (Claude / MCP Client)
        │
        ▼
   MCP Server  (FastMCP + OAuth 2.1)
        │
        ▼
   root_coordinator  (gemini-2.5-pro)
        │
        ├── google_ads_specialist    ──► BigQuery (campaigns, keywords, search terms)
        ├── meta_ads_specialist      ──► BigQuery (campaigns, adsets, creatives)
        ├── cross_channel_specialist ──► BigQuery (ROAS, CPA, unified performance)
        ├── analytics_specialist     ──► BigQuery (GA4 sessions, funnel, transactions)
        ├── seo_specialist           ──► BigQuery (Search Console, rankings, CTR)
        └── institutional_specialist ──► Vertex AI RAG (brand identity, company docs)
```

The user always interacts with a single entry point. The coordinator decides which specialist to invoke, consolidates the response, and replies in natural language.

## Key Design Decisions

**Semantic layer as single source of truth**  
Each specialist reads its tables, metrics and dimensions from YAML files — never hardcoded in Python. This separates business logic from agent logic and makes the system maintainable as the data model evolves.

**Factory functions for BigQuery specialists**  
Every BigQuery specialist is a `create_<domain>_specialist(**kwargs)` factory that receives shared toolsets and semantic content at initialization. One BigQuery toolset instance is shared across all agents.

**Gold over Silver**  
Specialists query `gld_*` tables by default. Silver tables are only used where campaign/ad-level granularity is required (Google Ads, Meta Ads).

**RAG for unstructured content**  
Institutional knowledge (brand voice, company docs) lives in a Vertex AI RAG corpus, not in BigQuery. The `institutional_specialist` uses `VertexAiRagRetrieval` instead of BigQuery toolset.

## Project Structure

```
agent/
  agent.py                  # Root coordinator + specialist initialization
  sub_agents/
    google_ads/
      semantic/             # YAML: tables, metrics, dimensions
      skills/               # SKILL.md: behavioral instructions
      google_ads_specialist.py
    meta_ads/
    cross_channel/
    analytics/
    seo/
    institutional/          # RAG-based, no semantic YAML
  tools/
    chart_generator.py      # Chart image generation
    datetime_tool.py        # Current date/time context
  business_rules/
    business_rules.yaml     # Global rules injected into all specialists
    constraints.yaml        # Hard constraints (never do X)
mcp_server/
  main.py                   # FastMCP tools exposed per specialist
  oauth.py                  # OAuth 2.1 Authorization Server
  session_manager.py        # InMemory or Vertex AI session backend
eval/
  generate_evalsets.py      # Pulls test cases from Google Sheets
  evalsets_generated/       # Auto-generated .evalset.json files
docs/
  architecture.md
  sub-agents.md
  mcp-server.md
  eval.md
  deployment.md
```

## Semantic Layer

Each specialist's behavior is defined by YAML files, not Python strings:

```yaml
# example: agent/sub_agents/google_ads/semantic/tables.yaml
tables:
  - name: gld_gads_campaign_daily
    description: Daily aggregated campaign performance
    columns:
      - name: campaign_name
        description: Campaign display name
      - name: cost_brl
        description: Media spend in BRL (already converted from micros)
      - name: clicks
      - name: impressions
      - name: conversions
      - name: roas
        description: Return on ad spend (revenue / cost)
```

## Evaluation Framework

Test cases are managed in a Google Sheets spreadsheet and pulled via script into ADK-compatible `.evalset.json` files. CI runs evals on every push to `main` — the branch protection gate only passes if all cases pass.

```bash
# Generate evalsets from spreadsheet
python eval/generate_evalsets.py --sheet-id <SHEET_ID>

# Run a specific eval case
adk eval agent eval/evalsets_generated/cases/<eval_id>.evalset.json \
  --config_file_path eval/evalsets_generated/cases/<eval_id>.config.json \
  --print_detailed_results
```

## Deployment

The system runs on **Cloud Run** (containerized via Docker). The MCP server implements OAuth 2.1 with Google login, restricting access to authorized domains.

```
Cloud Run
  └── FastMCP server (uvicorn)
        ├── OAuth 2.1 (Google login → JWT)
        └── Agent runner (ADK InMemoryRunner or Vertex AI Agent Engine)
```

## Stack

- **Agent Framework:** Google ADK 2.0
- **LLM:** Gemini 2.5 Pro (Vertex AI)
- **Data:** Google BigQuery · BigQueryToolset
- **RAG:** Vertex AI RAG Engine
- **MCP:** FastMCP · OAuth 2.1
- **Deployment:** Cloud Run · Docker
- **Eval:** ADK Eval · Google Sheets

## Related Projects

- [marketing-data-lake](https://github.com/fabricio-espel80/marketing-data-lake) — The data infrastructure this system queries
- [mcp-server-template](https://github.com/fabricio-espel80/mcp-server-template) — Standalone MCP server template extracted from this project
