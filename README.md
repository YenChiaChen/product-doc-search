# Product Doc Search

A single-file demo of how to search a company's product documents with **metadata filtering + semantic vector search** (Postgres + pgvector design).

- `index.html` — the demo app: search, document library, product tree, and table browser. No build step, no dependencies.
- `docs/search-system-design.md` — the design write-up with Mermaid diagrams (HackMD format).
- `design/claude-design-export.html` — the original Claude Design export the UI is based on.
- `legacy/doc-search-sim.html` — the first prototype.

All data is mock data for demonstration. The "LLM query parsing" and "embedding" steps are simulated with rules and a 12-dimension concept vocabulary.
