# 🧠 CHE MCP Architecture

> Deep dive into the gateway that routes natural language queries to 80+ Argentine data sources.

---

## Overview

CHE MCP is not a single MCP server — it's an **intelligent gateway** that classifies queries, routes them to the correct data source, and aggregates results. The architecture follows a layered design with graceful degradation at every stage.

```
┌──────────────────────────────────────────────────────────────┐
│                     MCP CLIENT                                │
│  (Claude Desktop / opencode / Cursor / custom agent)         │
└───────────────────────────┬──────────────────────────────────┘
                            │
              ┌─────────────▼─────────────┐
              │     HTTP Server            │
              │  (Streamable HTTP MCP)     │
              │  Port 3456                 │
              └─────────────┬─────────────┘
                            │
              ┌─────────────▼─────────────┐
              │   5-Stage Classifier       │
              │  ┌─────────────────────┐  │
              │  │ 1. Keyword Match    │  │
              │  │ 2. WMA Weighted     │  │
              │  │ 3. Embeddings       │  │
              │  │ 4. Data Node        │  │
              │  │ 5. LLM Fallback     │  │
              │  └─────────────────────┘  │
              └─────────────┬─────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
  ┌───────▼──────┐  ┌──────▼──────┐  ┌───────▼──────┐
  │  Data Node   │  │  Dispatcher │  │  MCP Bridge  │
  │ DuckDB+SQL   │  │  Fan-out    │  │  stdio wrap  │
  └───────┬──────┘  └──────┬──────┘  └──────────────┘
          │                │
  ┌───────▼──────┐  ┌──────▼──────┐
  │ 748 Parquet  │  │ Individual  │
  │  datasets    │  │   MCPs      │
  └──────────────┘  └──────┬──────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
  ┌───────▼──────┐  ┌──────▼──────┐  ┌───────▼──────┐
  │  DolarAPI    │  │  Weather    │  │   Transit    │
  └──────────────┘  └─────────────┘  └──────────────┘
  │  ... 10+ real MCPs connected at runtime           │
  └───────────────────────────────────────────────────┘
```

---

## 1. The 5-Stage Classifier

The classifier is the brain of CHE MCP. It takes a natural language query and determines which domain(s) to query. Each stage has a specific role and degrades gracefully.

### Stage 1: Keyword Matching

**How it works:** A pre-built domain hierarchy with 3,000+ keywords mapped to 182 classified domains. The keyword engine scans the query text for domain-specific terms.

```
Input: "cuanto sale el dolar blue"

Matches found:
  finanzas/dolar    → score: 2.0  ("dolar", "blue")
  finanzas/cotizaciones → score: 0.5 ("cuanto sale")

Parent consolidation → finanzas: 2.5
```

**Strengths:** Fast, zero-dependency, handles 95% of common queries.
**Limitations:** Fails on ambiguous or out-of-vocabulary queries.

**Implementation:** `packages/che-pro/src/router/intent.ts`
- `DOMAIN_HIERARCHY`: tree structure with 16 parent domains covering 182 classified domains
- `DOMAIN_KEYWORDS`: weighted keyword dictionary per child domain
- Negative keywords: 30 anti-match rules prevent catch-all domains from stealing queries

---

### Stage 2: WMA Weighted Routing

**How it works:** The Weighted Majority Algorithm (WMA) is an **online learning** system. Every domain starts with equal weight (1.0). When a query succeeds, the winning domain gets reinforced (+0.1). When it fails, the domain gets penalized (-0.1). Weights are bounded at [0.1, 5.0].

```
Before query "dolar blue":
  finanzas/dolar: weight 1.0
  clima/actual:   weight 1.0

Query succeeds via finanzas/dolar:
  finanzas/dolar: 1.0 + 0.1 = 1.1  ✓ reinforced
  clima/actual:   1.0              (unchanged)

After 100+ queries:
  finanzas/dolar:  4.2  (proven performer)
  clima/actual:    2.8  (mostly reliable)
  justicia/buscar: 0.3  (rarely used, low confidence)
```

**Persistence:** WMA weights are saved to disk as JSON. The router starts warm — it doesn't forget what it learned across restarts.

**Parent-level consolidation:** Instead of tracking weights at the individual domain level, WMA operates at the parent level (16 parent groups). This prevents overfitting and improves generalization.

**Implementation:** `packages/che-pro/src/router/wma-router.ts`
- Bounded additive scoring: reinforce (+0.1), penalize (-0.1)
- Weight range: 0.1–5.0
- Parent-level weight consolidation
- JSON file persistence at `/var/data/che-pro/wma-state.json`

---

### Stage 3: Semantic Embeddings

**How it works:** Two-mode operation — when Transformer.js is available, uses **all-MiniLM-L6-v2** to generate 384-dimensional vector embeddings. When unavailable, falls back to **Jaccard similarity** (keyword overlap, zero dependencies).

```
Query: "necesito saber el valor del dolar"
Keyword score: finanzas/dolar → 0.8 (partial match)
Embedding:     cos_sim(query, "cotizacion dolar blue") = 0.94 ← wins
Jaccard:       keyword overlap with finanzas keywords = 0.6
```

Both modes work — embeddings are more accurate but require downloading the model (~80MB). Jaccard is instant and works offline.

**Implementation:** `packages/che-pro/src/router/embedding-store.ts`
- Primary: all-MiniLM-L6-v2 via `@xenova/transformers`
- Fallback: Jaccard similarity coefficient
- Vector dimension: 384
- Used for MCP-Zero two-stage retrieval
- Lazy model loading (download on first use)

---

### Stage 4: Data Node Search

**How it works:** When a query involves statistics, trends, or numbers (*"inflation last 12 months"*, *"unemployment by province"*), the Data Node takes over. It converts natural language to SQL using pattern matching and guardrails, executes against DuckDB, and returns structured results.

```
Input: "inflacion ultimos 12 meses"

NL-to-SQL:
  → SELECT month, value FROM indice_precios
    WHERE date >= '2026-01-01'
    ORDER BY month

DuckDB result:
  Jun: 2.4%  Jul: 4.0%  Ago: 4.2%  ...
  Average: 3.7%

Formatted response
```

**Safety:** The NL-to-SQL engine has built-in guardrails:
- Read-only queries enforced
- No DDL, DML, or destructive operations
- Column name validation against dataset schemas
- Query timeout (5 seconds)
- Maximum result rows (1000)

**Implementation:** `scripts/data-node/`
- DuckDB query engine
- 748 Parquet datasets (Zstd compression, 9.92× — 404 MB vs 3.92 GB CSV)
- 17.68 MB of dataset profiles with column statistics
- Index cache: 553 ms warm start (54× faster than cold)
- DuckDB hit rate: 66% (83% in last refinement round)

---

### Stage 5: LLM Fallback

**How it works:** Optional. When all previous stages fail to classify a query confidently, an external LLM endpoint can attempt classification. Configured via `CHE_LLM_ENDPOINT` environment variable.

```
All stages exhausted:
  Keyword → no match (< threshold)
  WMA     → no domain above 0.5 confidence
  Embedding → cos_sim < 0.3
  Data Node → no matching dataset

→ LLM fallback classifies intent
→ Returns best-effort domain routing
```

When the LLM endpoint is not configured, the system returns a graceful degradation response listing available domains instead of an error.

---

## 2. Data Pipeline

The data pipeline manages all external data sources — CKAN datasets, real-time APIs, and individual MCP servers.

### Architecture

```
query("dolar blue")
      │
┌─────▼──────────────────────────────────────────┐
│              Data Pipeline                      │
│                                                 │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │ Cache   │  │Circuit  │  │ Rate    │        │
│  │ Check   │→ │Breaker  │→ │Limiter  │→ Fetch │
│  └────┬────┘  └────┬────┘  └────┬────┘        │
│       │            │            │              │
│  ┌────▼────┐  ┌────▼────┐  ┌────▼────┐        │
│  │ Memory  │  │ 3 fails │  │Max 3    │        │
│  │ 200 LRU │  │→ stale  │  │concurr  │        │
│  └─────────┘  └─────────┘  └─────────┘        │
└────────────────────────────────────────────────┘
      │
┌─────▼─────┐  ┌──────────┐  ┌──────────┐
│ Disk Cache│  │   CKAN   │  │   APIs   │
│ /var/data │  │ 80+ sets │  │  real-   │
│  .tmp+mv  │  │ datos.gob│  │  time    │
└───────────┘  └──────────┘  └──────────┘
```

### 3-Tier Cache

| Tier | Location | Capacity | Eviction |
|------|----------|----------|----------|
| **Memory** | In-process Map | 200 entries | LRU |
| **Disk** | `/var/data/che-pro/` | Unlimited (CSV) | mtime-based freshness |
| **Live** | datos.gob.ar CKAN | Source of truth | Circuit breaker managed |

**Atomic writes:** Disk cache uses `.tmp` files + atomic rename (`mv`). This prevents corrupted cache files on process crash.

**Request collapsing:** When 10 concurrent queries ask for the same dataset, only 1 HTTP request goes to CKAN. The other 9 share the response.

**Predictive pre-fetch:** The top 10 most-requested datasets refresh automatically every 15 minutes. Hot data stays fresh without user latency.

### Circuit Breaker

Each CKAN dataset has its own circuit breaker:

```
State machine per dataset:

  CLOSED ──3 consecutive failures──→ OPEN
    │                                    │
    │                              (wait 60s)
    │                                    │
    │←─────── 1 success ────── HALF_OPEN │
    │                                    │
    └────────────────────────────────────┘
```

When the circuit is OPEN, CHE MCP serves **stale cached data** instead of returning an error. The user still gets a response, just not the freshest one.

---

## 3. Dispatcher & MCP Bridge

### Dispatcher

The dispatcher handles **fan-out routing** — when a query could match multiple domains, it fires requests to all of them in parallel.

```
query("clima y trafico en Buenos Aires")
      │
      ├──→ clima/actual       ← parallel
      └──→ transporte/buenos-aires ← parallel
      │
      ▼
  Merge results → unified response
```

If the classifier has ≥0.5 confidence in a single domain, dispatch is single-target. Below 0.5, fan-out is used. This avoids unnecessary parallel calls for clear-cut queries.

### MCP Bridge

CHE MCP exposes two interfaces:

| Mode | Protocol | Use Case |
|------|----------|----------|
| **HTTP Server** | Streamable HTTP MCP | Production deployments, REST API, dashboard |
| **STDIO Bridge** | Standard MCP stdio | Claude Desktop, opencode, Cursor (local) |

The bridge wraps the HTTP REST API as a local MCP server for clients that expect stdio transport. It includes OpenTelemetry instrumentation for tracing.

**Implementation:** `packages/che-pro/src/bridge.ts`

---

## 4. Observability

### Health Endpoints

| Endpoint | Response |
|----------|----------|
| `GET /health` | `{ status: "ok", uptime, datasets, cache_hits }` |
| `GET /health/embeddings` | `{ mode: "embeddings" | "jaccard", model_loaded }` |
| `GET /health/wma` | `{ weights: {...}, total_queries }` |

### Audit Trail

Every tool call is logged with structured metadata: timestamp, query, routed domain, classifier stage used, latency, and result status. This feeds into the WMA learning loop and enables debugging.

### Dashboard

An SQLite-backed dashboard tracks usage: queries per domain, classifier stage distribution, cache hit rates, and circuit breaker state over time.

---

## 5. Performance Characteristics

| Metric | Value | Context |
|--------|-------|---------|
| **Classifier accuracy** | 95.45% TFS | MCPAgentBench (66 diverse queries) |
| **P50 latency** | ~50 ms | Keyword + cache hit |
| **P95 latency** | ~100 ms | With embedding stage |
| **DuckDB query** | 17 ms | Single aggregation query |
| **Cache warm start** | 553 ms | Index load (54× faster than cold) |
| **Parquet compression** | 9.92× | 404 MB vs 3.92 GB CSV |
| **DuckDB hit rate** | 66% | 83% on refinement round |

---

## Key Design Decisions

1. **Graceful degradation over perfect routing.** The 5-stage pipeline ensures no query goes unanswered. A weak routing is better than an error.
2. **Online learning over static classification.** WMA means the router improves over time without retraining.
3. **Stale-serving over failure.** Circuit breakers + cache mean users get data even when sources are down.
4. **DuckDB over traditional DB.** Embeddable, zero-config, fast on columnar data — perfect for an MCP server.
5. **Jaccard fallback over mandatory model download.** The embedding stage works offline with zero dependencies.

---

→ [Back to README](../README.md)
→ [Quickstart Guide](./QUICKSTART.md)
