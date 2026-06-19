<p align="center">
  <img src="assets/logo.png" alt="confluence-bot" width="96" height="96" />
</p>

# 🌌 confluence-bot — RBAC-Enforced MCP Documentation RAG, End to End

[![Live MCP server](https://img.shields.io/badge/Live-HF%20Spaces-yellow)](https://hoodieylya13-mcp-confluence-documentation-rag.hf.space/health)
[![Model Context Protocol](https://img.shields.io/badge/Protocol-MCP%20streamable%20HTTP-orange)](https://modelcontextprotocol.io/)
[![Security](https://img.shields.io/badge/Security-4--layer%20RBAC-red)](https://github.com/HoodieYlya13/mcp-confluence-documentation-rag/blob/main/SECURITY.md)
[![Python](https://img.shields.io/badge/python-3.11%20%7C%203.12-blue)](https://www.python.org/)
[![Next.js](https://img.shields.io/badge/Next.js-16-black)](https://nextjs.org/)
[![Tauri](https://img.shields.io/badge/Desktop-Tauri%202%20%C2%B7%20Rust-24C8DB)](https://tauri.app/)

A complete, production-deployed **Model Context Protocol** system built around one idea: *the same question must yield different answers depending on who is asking — and nothing else must ever leak.*

It spans the full stack of an applied-AI knowledge system:

- a **Python MCP server** that turns a live Atlassian Confluence instance into an RBAC-enforced RAG substrate, secured by a four-layer enforcement model and gated by an automated evaluation suite — deployed 24/7 on Hugging Face Spaces;
- a **Next.js 16 console** that lets anyone *experience* the access-control story in a browser — live health and metrics, plus a playground that asks one question through two real MCP sessions with different bearer tokens and shows, side by side, what each authorization level is allowed to retrieve;
- a **Tauri desktop spotlight** (Rust) that summons a frameless command bar on a global hotkey, asks the server's agent in one keystroke, and returns a grounded answer with links straight to the source Confluence pages — the same RBAC-governed server, driven from a third language, with the bearer token held in the trusted process and never exposed to the webview.

```
confluence-bot                             ← you are here (umbrella / showcase repo)
├── mcp-confluence-documentation-rag       Python MCP server — secure Confluence connector,
│                                          LlamaIndex + ChromaDB retrieval with ACL pushdown,
│                                          LangGraph agent, eval gates, Prometheus metrics
├── confluence-bot-app                     Next.js 16 console — live health & metrics dashboard,
│                                          dual-role RBAC playground over MCP streamable HTTP
└── confluence-spotlight                   Tauri desktop spotlight (Rust) — global-hotkey command
                                           bar, real rmcp client, token held server-side
```

---

## Try it live

| What | Where |
|---|---|
| **Console** | [`confluence-bot.hy13dev.com`](https://confluence-bot.hy13dev.com) — live dashboard + RBAC playground, no setup required |
| Server health | [`/health`](https://hoodieylya13-mcp-confluence-documentation-rag.hf.space/health) — corpus size, retriever backend, sync status |
| Prometheus metrics | [`/metrics`](https://hoodieylya13-mcp-confluence-documentation-rag.hf.space/metrics) — tool calls, latency, RBAC denials by layer |
| MCP endpoint | `https://hoodieylya13-mcp-confluence-documentation-rag.hf.space/mcp` — requires a bearer token; tokens map to roles **server-side** |

The console's [RBAC playground](https://confluence-bot.hy13dev.com/playground) asks the same accelerator-operations question as `JUNIOR_OP` and as `ATS_CORE_LEAD` simultaneously, in two modes: **compare retrieval** highlights every chunk the document ACL filter withheld from the lower-privileged session, while **compare answers** runs the server-side LangGraph agent and contrasts the grounded answer each role receives — in real time, against the real index. (The free-tier Space sleeps when idle; the console treats "waking up" as a state, not an error.)

```bash
claude mcp add --transport http accelerator-ops \
  https://hoodieylya13-mcp-confluence-documentation-rag.hf.space/mcp \
  --header "Authorization: Bearer <token>"
```

---

## The system in one picture

```mermaid
flowchart LR
    CONF["Atlassian Confluence Cloud
    space ATSOPS · ACL labels per page"]

    subgraph SRV["MCP server · Hugging Face Docker Space (Python)"]
        SYNC["Connector & sync
        fail-closed ACL · version diff"]
        IDX[("ChromaDB
        MiniLM embeddings
        role metadata")]
        MW["Bearer-token middleware
        token → role"]
        TOOLS["Retrieval tools
        list · fetch · semantic_search"]
        AGENT["LangGraph agent · ask tool
        retrieve → verify → generate
        + leak scan"]
        OBS["/health · /metrics"]
        SYNC --> IDX
        MW -->|raw tools · client reasons| TOOLS
        MW -->|ask tool · server reasons| AGENT
        TOOLS -->|ACL filter inside the vector query| IDX
        AGENT --> TOOLS
    end

    subgraph APP["Console · Next.js 16 (TypeScript)"]
        OV["Overview
        health + parsed Prometheus metrics"]
        PG["RBAC playground
        one query · two role sessions"]
        RL["Upstash rate limiting
        per-IP + global daily budget"]
        PG --> RL
    end

    CLIENT["Any MCP client
    Claude Desktop / Claude Code"]

    SPOT["Spotlight desktop · Tauri (Rust)
    global hotkey · rmcp client
    token held in-process"]

    CONF --> SYNC
    CLIENT -->|"Authorization: Bearer …"| MW
    RL -->|"MCP TypeScript SDK
    streamable HTTP · server-side tokens"| MW
    SPOT -->|"rmcp streamable HTTP
    server-side role · ask tool"| MW
    OV --> OBS
```

---

## Half 1 — the MCP server ([`mcp-confluence-documentation-rag`](https://github.com/HoodieYlya13/mcp-confluence-documentation-rag))

A zero-trust RAG pipeline over real Confluence content:

- **Secure connector** — Confluence REST sync with fail-closed ACL mapping (a page with no recognized ACL label is restricted, never public) and incremental version-diff updates.
- **Structure-preserving ingestion** — XHTML → Markdown with scientific tables kept atomic and heading context carried into every sub-chunk.
- **ACL pushdown** — role filters are evaluated *inside* the ChromaDB vector query, so unauthorized chunks never enter the candidate set, no matter what the calling code does.
- **Four-layer enforcement** — bearer auth → ACL pushdown → LangGraph context-verifier node → post-generation leak scanner. A compromised retriever or a prompt injection embedded in the corpus cannot leak restricted content. Full threat model in [SECURITY.md](https://github.com/HoodieYlya13/mcp-confluence-documentation-rag/blob/main/SECURITY.md).
- **Two deployment modes, one server** — capable clients (Claude Desktop / Claude Code) call the raw retrieval tools and do their own reasoning (layers 1–2); thin or untrusted clients call the single `ask` tool, which runs the full LangGraph agent server-side so the verify gate and leak scanner (layers 3–4) are enforced regardless of caller. Identical RBAC either way.
- **Gated evaluation suite** — 8 scenarios run in CI with exit-code gates: golden-set retrieval, adversarial probes (including a permanent prompt-injection fixture *inside the live corpus*), LLM-as-judge faithfulness, and a 0.00% leakage target.
- **MLOps** — two-speed GitHub Actions (seconds-fast offline gates per push, full semantic + LLM pipeline nightly), Trivy scans, self-healing nightly sync, custom Prometheus exposition. Runs at **$0/month**.

Latest live evaluation: **0.00% RBAC leakage · 0 adversarial probes leaked · 100% golden-set hit rate @3 · 100% faithfulness.**

## Half 2 — the console ([`confluence-bot-app`](https://github.com/HoodieYlya13/confluence-bot-app))

A deliberately thin, security-conscious window over the live server — no MCP client required:

- **Overview** — `/health` plus Prometheus exposition parsed server-side into semantic cards: indexed corpus, tool calls, RBAC denials *by enforcement layer*, latency, sync status.
- **RBAC playground** — one query fans out to two genuine MCP sessions (official [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) over streamable HTTP, full initialize → tool-call → close handshake) holding different bearer tokens, in two modes: **compare retrieval** badges the chunks withheld from the junior operator, while **compare answers** calls the server-side `ask` agent tool and contrasts the grounded answer each role receives.
- **Server-first Next.js 16** — Cache Components / Partial Prerendering, React Compiler, server actions, and a `next/form` GET flow: the playground works with JavaScript disabled, and the only client component in the app is a pending-state submit button.
- **No JSON API surface** — tokens are `server-only`, the browser receives rendered HTML, and the single path to the MCP server is guarded by Upstash rate limiting (per-IP sliding window + global daily budget, fail-closed in production).

## Half 3 — the desktop spotlight ([`confluence-spotlight`](https://github.com/HoodieYlya13/confluence-spotlight))

A Tauri v2 desktop client that puts the assistant one keystroke away — and proves the security model holds outside the browser:

- **Summon-on-hotkey UX** — a frameless, always-on-top, centered command bar (Raycast/Spotlight style) bound to a global shortcut (`Cmd+Shift+Space` by default); ask, read the grounded answer, click through to the Confluence page (opened in the system browser), Esc/blur to dismiss.
- **Token never leaves the trusted process** — sign-in is a deep-link **OAuth Authorization-Code + PKCE** round trip through the console, and the **Rust** process (never the webview) redeems the one-time code for the bearer token, which is then held in memory only. The shipped binary contains no secrets. This is the desktop analog of the console's `server-only` tokens and of the server's own `STDIO_ROLE` model — the client never sends its own role.
- **A real Rust MCP client** — the call uses the official [`rmcp`](https://crates.io/crates/rmcp) SDK over streamable HTTP (full initialize → `ask_accelerator_operations` → close), so one server is now exercised from clients in **three languages**.
- **Single role with an honest badge** — runs as one configured clearance, shown in the bar; the same query as a different role is a second sign-in, not a hidden toggle.
- **Downloadable on every OS** — a GitHub Actions `tauri-action` matrix builds unsigned macOS (universal), Windows, and Linux installers per release tag; the macOS-only `NSPanel` overlay is `cfg`-gated behind a portable always-on-top-window fallback, so one codebase ships everywhere.

---

## Run it yourself

```bash
git clone --recurse-submodules https://github.com/HoodieYlya13/confluence-bot.git
```

**Server — fully offline, no accounts needed:**

```bash
cd mcp-confluence-documentation-rag
make build && make test && make run-eval && make run-agent
```

**Console:**

```bash
cd confluence-bot-app
bun install && cp .env.example .env.local && bun dev
```

**Desktop spotlight:**

```bash
cd confluence-spotlight
cp .env.example .env   # set MCP_TOKEN_* for dev sign-in; URLs default to the live deploy
bun install && bun run tauri dev
```

Each submodule's README has the full quickstart (live Confluence pipeline, Claude Desktop wiring, Vercel deployment, the spotlight's GUI-free data-path probe).

---

## Documentation map

| Document | What's in it |
|---|---|
| [server README](https://github.com/HoodieYlya13/mcp-confluence-documentation-rag#readme) | Architecture diagram, quickstarts, evaluation report, repository map |
| [server SECURITY.md](https://github.com/HoodieYlya13/mcp-confluence-documentation-rag/blob/main/SECURITY.md) | Identity model, the four enforcement layers, threat model |
| [server TAD.md](https://github.com/HoodieYlya13/mcp-confluence-documentation-rag/blob/main/TAD.md) | Every server design decision with rationale |
| [console README](https://github.com/HoodieYlya13/confluence-bot-app#readme) | Console setup, environment variables, deployment |
| [console TAD.md](https://github.com/HoodieYlya13/confluence-bot-app/blob/main/TAD.md) | Every console design decision with rationale |
| [spotlight README](https://github.com/HoodieYlya13/confluence-spotlight#readme) | Desktop spotlight setup, env vars, hotkey, data-path probe |
| [spotlight TAD.md](https://github.com/HoodieYlya13/confluence-spotlight/blob/main/TAD.md) | Every desktop design decision with rationale |

House convention across the project: **no code comments or docstrings** — all design rationale lives in each repo's `TAD.md`.
