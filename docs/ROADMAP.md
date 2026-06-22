# 🗺️ CHE MCP Roadmap

> Public roadmap — what's built, what's in progress, what's coming.

---

## ✅ Phase 1 — Foundation *(Complete)*

- [x] 80+ CKAN datasets from datos.gob.ar and sector portals
- [x] 748 Parquet datasets with DuckDB NL-to-SQL query engine
- [x] 5-stage intelligent classifier (keyword → WMA → embeddings → Data Node → LLM)
- [x] Weighted Majority Algorithm with online learning and persistence
- [x] 3-tier cache: memory → disk → live with atomic writes
- [x] Circuit breaker per dataset with stale-serving fallback
- [x] Rate limiting per API key (100 req/min)
- [x] JWT + API key authentication with scope validation
- [x] OpenTelemetry instrumentation and audit trail
- [x] Streamable HTTP MCP server + stdio bridge
- [x] 280+ tests (Vitest)
- [x] 95.45% classifier accuracy on MCPAgentBench

---

## ✅ Phase 2 — Data Coverage *(Complete)*

- [x] Finance: DolarAPI (blue, official, MEP, CCL, crypto), BCRA rates, UVA calculator
- [x] Climate: SMN weather, forecasts, UV index
- [x] Transit: colectivos, trenes, subte status and schedules
- [x] Sports: AFA football (Liga Profesional, Copa Argentina, club data)
- [x] Tax: ARCA (ex-AFIP) invoice validation (RG 5616), CUIT lookup
- [x] Agriculture: grain prices (Rosario Board of Trade), livestock (SENASA)
- [x] Energy: power outages (EDET), fuel prices, electricity generation
- [x] Health: epidemiology, vaccination, medications

---

## 🔲 Phase 3 — Community & Distribution *(In Progress)*

- [ ] Public npm package: `@artificio/che-mcp`
- [ ] MCP Registry listing (Glama, Smithery, MCP.so)
- [ ] Quality and trust seals from MCP community sites
- [ ] Community Discord / discussion forum
- [ ] Spanish-language tutorials and video guides
- [ ] Contribution guide for new domain authors
- [ ] Automated dataset freshness monitoring

---

## 🔲 Phase 4 — Advanced Capabilities

- [ ] Integrated LLM for query understanding (local model, optional)
- [ ] Real-time data streaming (WebSocket feeds)
- [ ] Historical data queries: "compare inflation Q1 2023 vs Q1 2024"
- [ ] Cross-domain queries: "impact of fuel prices on grain transport costs"
- [ ] Scheduled reports and alerts
- [ ] Mobile companion app for AI agent interactions
- [ ] Redis cache layer for horizontal scaling

---

## 🔲 Phase 5 — Ecosystem

- [ ] Plugin system: third-party domain handlers
- [ ] CHE Apps marketplace (SEP-1865)
- [ ] x402 payment protocol for monetized data
- [ ] Multi-country expansion (CHE Chile, CHE Uruguay, CHE Brasil)
- [ ] Enterprise SSO and team management
- [ ] SLA-backed uptime guarantees

---

## 🤝 Community Priorities

What gets built next depends on **you**. Open an issue to vote for:

- New data domains you need
- Better accuracy for specific query types
- Client support for your platform
- Performance improvements
- Documentation in your language

→ [Open an issue](https://github.com/Albano-schz/che-mcp/issues)

---

## 📊 How We Prioritize

1. **Community demand** — most-requested features first
2. **Data source availability** — official, stable APIs over scraping
3. **Technical feasibility** — can we build it well?
4. **Impact** — how many users benefit?

---

→ [Back to README](../README.md)
