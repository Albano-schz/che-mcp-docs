# ✨ CHE MCP Feature Showcase

> Every capability, with technical context — why we built it and what makes it special.

---

## 🎯 Intelligent Query Routing

**What it is:** A 5-stage classifier that takes any natural language query and routes it to the correct data source — no manual configuration, no tool selection required from the user.

**Why it matters:** Without intelligent routing, an MCP with 80+ data sources would either require the user to specify which tool to use (terrible UX) or produce irrelevant results. CHE MCP's classifier understands intent automatically.

**Technical highlights:**
- 3,000+ weighted Spanish keywords across 182 domains
- Weighted Majority Algorithm (WMA) with online learning — improves over time
- 384-dim semantic embeddings (all-MiniLM-L6-v2)
- Graceful degradation: if keyword fails → WMA → embeddings → Data Node → LLM

**Benchmark:** 95.45% Top-First-Score accuracy on MCPAgentBench (66 diverse queries covering finance, transit, climate, sports, and government data).

```typescript
// Example: how the classifier sees a query
const result = await classifier.classify("dolar blue");
// → {
//     domain: "finanzas/dolar",
//     confidence: 0.94,
//     stage: "keyword",
//     alternatives: ["finanzas/cotizaciones"]
//   }
```

---

## 🧠 WMA — A Router That Learns

**What it is:** The Weighted Majority Algorithm is an online machine learning system embedded directly in the router. Every domain starts with equal weight. Successful queries reinforce the winning domain; failed queries penalize it.

**Why it matters:** Static routing tables are fragile. They break when sources change, when new domains are added, or when queries evolve. WMA adapts automatically — no retraining, no manual weight tuning.

**How it works:**
```
Query succeeds → reinforce winner (+0.1)
Query fails    → penalize loser   (-0.1)
Weights capped → [0.1, 5.0] range

After 500 queries:
  finanzas/dolar          → 4.2 ★★★★  (proven performer)
  clima/actual            → 3.1 ★★★✩  (reliable)
  justicia/buscar         → 0.5 ★✩✩✩  (rarely queried)
  transporte/colectivos   → 0.1 ★✩✩✩  (poor performance → degraded)
```

**Persistence:** Weights save to JSON on disk. Restart the server and it remembers what it learned.

---

## 🗄️ Data Node — SQL, But Natural

**What it is:** A DuckDB-powered query engine that converts natural language questions into SQL queries on 748 Argentine Parquet datasets.

**Why it matters:** Raw datasets are opaque. Without the Data Node, an AI agent would need to download CSVs, understand their schemas, and write SQL manually. The Data Node does all of that automatically.

**Example workflow:**
```
User: "¿Cuánto aumentó la inflación en 2024?"

1. NL-to-SQL:
   → SELECT AVG(valor) as inflacion_promedio
     FROM indice_precios_consumidor
     WHERE fecha BETWEEN '2024-01-01' AND '2024-12-31'

2. DuckDB executes on Parquet → loads column stats from profile (17.68 MB index)

3. Result: 117.8% anual

4. Response formatted for the AI agent
```

**Safety guardrails:**
- Read-only queries enforced
- No DDL / DML / destructive operations
- Column name validation against dataset schemas
- 5-second query timeout
- 1,000-row result limit
- SQL injection prevention via prepared statements

**Performance:**
- 748 datasets, 404 MB compressed (9.92× vs CSV)
- 17 ms single aggregation query
- 553 ms warm start (54× faster than cold)
- 66% DuckDB hit rate overall, 83% on refinement round

---

## 💾 3-Tier Cache — Data When Sources Fail

**What it is:** A layered caching system that ensures responses are available even when external sources (CKAN, APIs) are unreachable.

| Tier | Speed | Freshness | When It's Used |
|------|-------|-----------|----------------|
| Memory | <1 ms | Minutes | Hot queries, recent fetches |
| Disk | ~5 ms | Hours/Days | Persistent cache, post-restart |
| Live | 50-2000 ms | Real-time | Cache miss, TTL expired |

**Why it matters:** Argentina's government servers aren't always reliable. datos.gob.ar has downtime. APIs have rate limits. The 3-tier cache means CHE MCP works even when the source doesn't.

**Key features:**
- **Atomic writes:** Disk cache uses `.tmp` files + `mv` — no corrupted cache on crash
- **Request collapsing:** 50 concurrent queries for the same dataset → 1 upstream fetch, 49 shared
- **Predictive pre-fetch:** Top 10 hot datasets refresh every 15 minutes
- **Stale-serving:** When circuit breaker opens, serve cached data instead of erroring

---

## ⚡ Circuit Breaker — Resilience per Dataset

**What it is:** Each CKAN dataset gets its own circuit breaker with a 3-failure threshold and 60-second cooldown.

**Why it matters:** Without circuit breakers, one failing dataset blocks all queries that depend on it. With per-dataset breakers, only that domain is affected — everything else works normally.

```
Dataset: "indice_precios_consumidor"
                ┌──────────┐
                │  CLOSED  │ ← Normal operation
                └────┬─────┘
                     │ 3 consecutive fetch failures
                ┌────▼─────┐
                │   OPEN   │ ← Serves cached data
                └────┬─────┘
                     │ 60 second cooldown
                ┌────▼─────┐
                │ HALF_OPEN│ ← Tests with 1 request
                └────┬─────┘
                ┌────┴─────┐
           success       failure
                │            │
          ┌────▼─────┐  ┌───▼──────┐
          │  CLOSED  │  │   OPEN   │
          └──────────┘  └──────────┘
```

---

## 🔐 Auth & Rate Limiting

**What it is:** Production-grade authentication and multi-tenant isolation.

**Auth:**
- JWT-based authentication with scope validation
- API key authentication for headless clients
- Per-key rate limiting: 100 requests/minute
- Scoped access per domain (optional)

**Rate Limiting:**
- Token bucket algorithm per API key
- 100 req/min default, configurable per tier
- 429 response with `Retry-After` header
- Noisy neighbor isolation — high-usage keys don't affect others

**Why it matters:** Public MCP servers need auth. CHE MCP includes authentication and rate limiting in the core — ready for multi-tenant use.

---

## 🔍 Observability & Debugging

### Trace Endpoint
`POST /api/trace` — diagnostic endpoint that shows the classifier's reasoning step by step:

```json
{
  "query": "dolar blue",
  "stages": [
    { "stage": "keyword", "matches": ["finanzas/dolar: 2.0", "finanzas/cotizaciones: 0.5"] },
    { "stage": "wma", "weights": { "finanzas/dolar": 4.2, "finanzas/cotizaciones": 1.8 } },
    { "stage": "final", "domain": "finanzas/dolar", "confidence": 0.94 }
  ],
  "latency_ms": 45
}
```

### Health Endpoints
- `GET /health` — server status, uptime, datasets, cache hits
- `GET /health/embeddings` — embedding mode, model loaded
- `GET /health/wma` — current WMA weights, total queries learned

### Audit Trail
Every query is logged: timestamp, user, query text, routed domain, classifier stage, latency, and result status. Feeds into WMA learning.

### OpenTelemetry
Distributed tracing for production debugging across the HTTP server, bridge, and individual MCP executions.

---

## 🔌 MCP Bridge — Universal Client Support

**What it is:** A bridge that wraps the HTTP REST API as a standard MCP stdio server — so clients that expect local MCP transport can use CHE MCP without changes.

**Supported clients:**
- Claude Desktop (stdio)
- opencode (stdio)
- Cursor (stdio + HTTP)
- Any MCP-compatible client

**Implementation:** The bridge reads tool definitions from the HTTP API and translates them into MCP SDK tools dynamically. OpenTelemetry spans flow through the bridge for end-to-end tracing.

---

## 🗺️ What's Under the Hood

| Layer | Implementation |
|-------|---------------|
| Language | TypeScript 5.4 |
| Runtime | Node.js 24 |
| MCP SDK | `@modelcontextprotocol/sdk` ^1.29.0 |
| Query Engine | DuckDB (columnar, embeddable) |
| Embeddings | `@xenova/transformers` (all-MiniLM-L6-v2) |
| Validation | Zod (compile-time + runtime) |
| Testing | Vitest (280+ tests) |
| Cache | Custom 3-tier (memory, disk, live) |
| Observability | OpenTelemetry + structured audit trail |
| Data Format | Parquet (Zstd compression, 9.92×) |
| Data Sources | CKAN (datos.gob.ar), REST APIs, WebSocket feeds |

---

→ [Back to README](../README.md)
→ [Architecture Deep Dive](./ARCHITECTURE.md)
