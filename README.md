# 🧉 CHE MCP

> *The first national MCP ecosystem. Argentina, at your AI's fingertips.*

**CHE MCP** is an intelligent gateway that gives AI agents real-time access to Argentina's data — from government statistics and finance to climate, transit, football, and tax compliance. One MCP server. Eighty-plus data sources. Zero configuration needed.

[![MCP SDK](https://img.shields.io/badge/MCP_SDK-^1.29.0-blue)](https://github.com/modelcontextprotocol/typescript-sdk)
[![Tests](https://img.shields.io/badge/tests-280+-brightgreen)]()
[![Datasets](https://img.shields.io/badge/datasets-748-orange)]()
[![npm](https://img.shields.io/badge/npm-coming_soon-orange)]()
[![License](https://img.shields.io/badge/license-MIT-blue)](./LICENSE)

---

## Why CHE MCP?

Instead of installing 80+ individual MCP servers for every Argentine data source, you install **one**. CHE MCP understands your query in natural language and routes it to the correct source automatically:

* 💵 *"How much is the dólar blue?"* → DolarAPI
* 🌦️ *"What's the weather in Buenos Aires?"* → National Weather Service
* ⚖️ *"Validate this ARCA invoice"* → ARCA tax validator
* 🚍 *"When's the next colectivo to Palermo?"* → Transit API
* ⚽ *"Boca Juniors' last result?"* → AFA football data
* 📊 *"Inflation trend last 12 months?"* → DuckDB on 748 datasets

---

## 🚀 Launching Soon

CHE MCP is in active deployment. The **Data Node** — 748 Parquet datasets, DuckDB query engine with NL-to-SQL — is being deployed to production this week.

> ⭐ **Star this repo** to get notified when the public endpoint and one-line install go live.
> 
> See the **[Roadmap →](./docs/ROADMAP.md)** for the full launch timeline.

---

## 📊 Project Status

| Area | Status | Notes |
|------|:------:|-------|
| **Core Gateway** (routing, cache, dispatch) | ✅ Production | 280+ tests, 95.45% accuracy |
| **80+ CKAN datasets** (datos.gob.ar) | ✅ Live | Real data, circuit-breaker protected |
| **DuckDB + NL-to-SQL** Query Engine | ✅ Live | 748 Parquet datasets, 9.92× compressed |
| **Weather, Transit, Sports, Finance** APIs | ✅ Live | Real-time from official sources |
| **ARCA tax validation** (RG 5616) | ✅ Live | Invoice validation, CUIT lookup |
| **npm package** `@artificio/che-mcp` | 🔲 Coming | Phase 3 on roadmap |
| **Billing & payments** | 🔲 In dev | Currently free for public testing |
| **LLM fallback endpoint** | 🔲 Optional | Requires external endpoint config |

> **Transparency:** Some features (billing, LLM fallback) require external configuration. The gateway runs fully self-contained for core routing — zero external API dependencies required.

---

## 🧠 How It Works — The Intelligent Gateway

CHE MCP uses a **5-stage classification pipeline** that degrades gracefully. If one stage can't route the query, the next one catches it:

```
Query: "dolar blue hoy"
         │
    ┌────▼─────┐   Stage 1 — Keyword matching
    │  Keyword  │   3,000+ keywords across 182 classified domains
    └────┬─────┘
         │
    ┌────▼─────┐   Stage 2 — WMA weighted routing
    │   WMA     │   Weighted Majority Algorithm: learns from every query
    └────┬─────┘   (online reinforcement learning, bounded additive scoring)
         │
    ┌────▼─────┐   Stage 3 — Semantic embeddings
    │ Embedding │   384-dim vectors (all-MiniLM-L6-v2) with Jaccard fallback
    └────┬─────┘
         │
    ┌────▼─────┐   Stage 4 — Data Node search
    │ Data Node │   DuckDB SQL over 748 Parquet datasets + NL-to-SQL
    └────┬─────┘
         │
    ┌────▼─────┐   Stage 5 — LLM fallback
    │   LLM     │   External endpoint (optional, configurable)
    └────┬─────┘
         │
    ┌────▼─────┐
    │  Response │   "Dólar blue: $1,245 / $1,265 compra/venta"
    └──────────┘
```

### Under the Hood

| Component | Technology | What It Does |
|-----------|-----------|--------------|
| **Intelligent Router** | Weighted Majority Algorithm (WMA) | Learns from every query — reinforces good routes, penalizes bad ones — 95.45% accuracy |
| **3-Tier Cache** | Memory → Disk → Live Source | Atomic file writes, request collapsing (concurrent queries share fetches), predictive pre-fetch on hot datasets |
| **Circuit Breaker** | Per-dataset, 3-failure threshold | When a source is down, serves stale data instead of errors — fails gracefully |
| **Query Engine** | DuckDB + NL-to-SQL | Natural language → SQL on 748 Parquet datasets with 9.92× Zstd compression |
| **Rate Limiting** | Per-API-key, 100 req/min | Production-grade multi-tenant isolation |
| **Auth** | JWT + API key, scope validation | Authentication and rate limiting built-in |
| **Observability** | OpenTelemetry + audit trail | Full per-request tracing and diagnostics |
| **Classifier** | 5-tier with graceful degradation | Never returns an unhandled error — always falls back to the next tier |

---

## 🇦🇷 Real Data, Official Sources

Every domain connects to **verified Argentine data sources**:

| Sector | Sources |
|--------|---------|
| 💰 **Finance** | BCRA (Central Bank), DolarAPI, UVA mortgage calculator, INDEC inflation |
| 🌤️ **Climate** | SMN (National Weather Service), satellite data, UV index |
| 🚌 **Transit** | Colectivos, trenes, subte — real-time and schedules |
| 🏛️ **Government** | 80+ datasets from datos.gob.ar (CKAN), official gazette |
| ⚽ **Sports** | AFA (Argentine Football Association), league tables, results |
| ⚖️ **Tax & Legal** | ARCA (ex-AFIP) invoice validation, tax compliance |
| 🌾 **Agriculture** | Grain prices, livestock markets, agro forecasts |
| 🏥 **Health** | Public health statistics, epidemiology |
| 📚 **Education** | Enrollment data, academic statistics |
| 🏗️ **Infrastructure** | Public works, energy, logistics |

→ **[Full Domain Catalog →](./docs/DOMAINS.md)** (🇦🇷 in Spanish)

---

## 🔬 Pushing MCP Boundaries

CHE MCP isn't just a data aggregator — it incorporates production-grade patterns rarely seen in MCP servers:

* **Online learning classifier** — the WMA router adjusts weights from real user feedback. Not static routing. It learns.
* **Parallel multi-MCP dispatch** — one query fires across multiple MCPs simultaneously via fan-out dispatch.
* **Embedding-based semantic routing** — 384-dim vectors from all-MiniLM-L6-v2 via @xenova/transformers catch intent when keywords miss.
* **NL-to-SQL query generation** — ask *"average inflation 2023–2025"* and DuckDB generates + executes the SQL on 748 real Parquet datasets.
* **Request collapsing** — concurrent identical queries share a single upstream CKAN fetch.
* **Predictive pre-fetch** — the 10 most-requested datasets refresh every 15 minutes before you even ask.
* **Stale-serving with circuit breakers** — if datos.gob.ar is down, CHE MCP returns cached data instead of failing.

All running on **TypeScript + Node.js 24** with the official `@modelcontextprotocol/sdk` v1.29.0.

---

## 🏗️ Built for the Next MCP Standard

The Model Context Protocol is undergoing its biggest architectural update in July 2026 — new mandatory **Streamable HTTP transport**, **stateless server architecture**, and **standardized authentication**. Most MCP servers are still migrating from the legacy stdio + in-process model.

**CHE MCP was architected for the new protocol from day one:**

| Standard | Status | Implementation |
|----------|:------:|----------------|
| **Streamable HTTP Transport** | ✅ Ready | HTTP server with bidirectional streaming, REST API, dashboard |
| **SDK v1.29.0 + `registerTool` API** | ✅ Adopted | Dynamic tool registration, latest MCP SDK |
| **Stateless Architecture** | ✅ Designed | No server-side session state, JWT-based authentication |
| **API Key + Scope Validation** | ✅ Built-in | Per-key rate limiting, domain-level access control |
| **OpenTelemetry Tracing** | ✅ Integrated | Distributed tracing across HTTP server + stdio bridge |

While the MCP ecosystem migrates to the new standard, CHE MCP is already running on it — production-ready for the July 2026 cutoff.

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | TypeScript 5.4 + Node.js 24 |
| MCP SDK | `@modelcontextprotocol/sdk` ^1.29.0 (`server.registerTool` API) |
| Query Engine | DuckDB (Parquet + NL-to-SQL with SQL injection guardrails) |
| Embeddings | all-MiniLM-L6-v2 via @xenova/transformers + Jaccard fallback |
| Validation | Zod (types, inputs, outputs, config) |
| Tests | Vitest — 280+ test suite |
| Cache | 3-tier: in-memory LRU → persistent disk → live CKAN |
| Observability | OpenTelemetry + structured audit trail |

---

## 📚 Documentation

| Document | Language | Content |
|----------|----------|---------|
| [Architecture](./docs/ARCHITECTURE.md) | 🇬🇧 EN | Deep dive: 5-stage classifier, WMA learning, cache, pipeline |
| [Domains](./docs/DOMAINS.md) | 🇦🇷 ES | Complete catalog of 80+ Argentine data domains |
| [Features](./docs/FEATURES.md) | 🇬🇧 EN | Technical showcase of every capability |
| [Roadmap](./docs/ROADMAP.md) | 🇬🇧 EN | What's coming next, community priorities |
| [Example Prompts](./examples/prompts.md) | 🇦🇷 ES | Queries you'll be able to run at launch |

---

## 🤝 Community

CHE MCP is built for the Argentine developer community first.

* 🇦🇷 **Spanish-first support** — documentation, examples, and community in Spanish
* 🌍 **English technical docs** — for the global MCP ecosystem and AI community
* 🐛 **Issues & feature requests** — [open an issue](https://github.com/Albano-schz/che-mcp/issues)
* 🧉 **Become a contributor** — new domains, better classifiers, improved data sources

---

## 📦 From the Makers of

**ARTIFICIO Soluciones Inteligentes** — Bahía Blanca, Argentina.

We build AI infrastructure, MCP ecosystems, and data intelligence for Latin America.

---

## 📜 License

MIT — free to use, modify, and distribute.

---

🧉 *"Che, ¿tu IA ya conoce Argentina?"*
