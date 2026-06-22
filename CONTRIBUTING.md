# 🤝 Contributing to CHE MCP

> ¡Gracias por querer contribuir! CHE MCP es un proyecto de comunidad — construido en Argentina, para el mundo.

---

## Ways to Contribute

### 🗺️ Add a New Data Domain

Each domain follows a standard structure. To add a new data source:

1. **Identify the source** — official API, CKAN dataset, or structured data feed
2. **Add keywords** in `packages/che-pro/src/router/intent.ts` — Spanish keywords that activate this domain
3. **Register the domain** in `packages/che-pro/src/types.ts` — add to `DOMAIN_TO_MCP` with tool name and description
4. **Create a handler** in `packages/che-pro/src/data-pipeline.ts` — fetch, cache, and circuit-breaker logic
5. **Add tests** — verify the handler with real and edge-case inputs
6. **Document** — add to `docs/DOMAINS.md` with description, source, and example queries

### 🐛 Report a Bug

Open an issue with:
- What you were trying to do (query text)
- What happened vs what you expected
- CHE MCP version or commit hash
- Any error messages from the trace endpoint (`/api/trace`)

### 🔧 Improve the Classifier

The 5-stage classifier gets better with real-world feedback:
- **Keyword coverage:** missing keywords for existing domains → add to `intent.ts`
- **WMA tuning:** domain consistently misrouting → report with trace data
- **False positives:** wrong domain catching queries → add negative keywords
- **Embeddings:** poor semantic matching → suggest better example phrases

### ✍️ Improve Documentation

- Fix typos, unclear explanations, or outdated numbers
- Translate docs to other languages (Portuguese, English improvements)
- Add diagrams, screenshots, or video tutorials
- Write blog posts or tutorials about using CHE MCP

### 🧪 Write Tests

We use Vitest. Every domain handler, classifier stage, and utility should have tests.

```bash
npm test                 # Run all tests
npm test -- --watch      # Watch mode
npm test -- --coverage   # Coverage report
```

---

## Development Setup

```bash
git clone https://github.com/Albano-schz/che-mcp.git
cd che-mcp
npm install

# Start in dev mode
npm run dev

# Run tests
npm test

# Benchmark the classifier
npx tsx scripts/benchmark-router.ts
```

---

## Code Style

- TypeScript strict mode
- Zod validation for all external inputs
- Descriptive variable names (Spanish is OK for domain-specific terms)
- Comments in English
- Functions should be small and single-purpose
- Async/await over raw promises

---

## Pull Request Process

1. **Fork** the repo and create a feature branch
2. **Write tests** for new functionality
3. **Update docs** if you changed behavior
4. **Run `npm test`** — all tests must pass
5. **Open a PR** with a clear description of what changed and why
6. **Wait for review** — we review PRs within 48 hours

---

## Community Guidelines

- 🇦🇷 Spanish and English are both welcome — use whatever you're comfortable with
- Be respectful and constructive
- Focus on the technical merit of ideas
- Beginners welcome — ask questions, we'll help

---

## Questions?

Open an issue with the `question` label or reach out to the maintainers.

🧉 *Gracias por ser parte de CHE MCP.*
