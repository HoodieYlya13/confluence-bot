# confluence-bot

An RBAC-enforced **Model Context Protocol** documentation RAG system, end to end: a Python MCP server with a four-layer security model deployed on Hugging Face Spaces, and a Next.js console that lets you experience the access-control story live — no MCP client required.

```
confluence-bot
├── mcp-confluence-documentation-rag   Python MCP server — secure Confluence connector,
│                                      LlamaIndex + ChromaDB retrieval with ACL pushdown,
│                                      LangGraph agent, eval gates, Prometheus metrics
└── confluence-bot-app                 Next.js 16 console — live health & metrics dashboard,
                                       dual-role RBAC playground over MCP streamable HTTP
```

## Try it

- **Live MCP server:** [`/health`](https://hoodieylya13-mcp-confluence-documentation-rag.hf.space/health) · [`/metrics`](https://hoodieylya13-mcp-confluence-documentation-rag.hf.space/metrics) — the MCP endpoint itself requires a bearer token; identical questions yield different answers per authorization level.
- **Console:** ask the same accelerator-operations question as `JUNIOR_OP` and as `ATS_CORE_LEAD` side by side, and watch the document ACL filter withhold restricted chunks from the lower-privileged session — in real time, against the real index.

## Clone

```bash
git clone --recurse-submodules https://github.com/HoodieYlya13/confluence-bot.git
```

Each submodule has its own README, and design rationale lives in each `TAD.md` (house convention: no code comments).
