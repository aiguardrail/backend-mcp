# Collar Guardrail

**Deterministic pre-trade risk layer for AI agents on Robinhood Chain.**

Collar Guardrail evaluates proposed trades before execution and returns
`allow` / `warn` / `deny` with reasons, a 0–100 risk score, and a
tamper-evident SHA-256 audit hash.

[![MCP Registry](https://img.shields.io/badge/MCP%20Registry-v1.0.2-brightgreen)](https://registry.modelcontextprotocol.io/servers/io.github.aiguardrail/backend)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)

---

## Overview

Autonomous trading agents can move capital faster than any human can review.
Collar Guardrail sits between an agent and the chain as a deterministic
pre-trade checkpoint: every proposed trade is scored against live oracle
prices, market-hours rules, tier-based policy limits, slippage bounds, and
contract-safety signals before it is allowed to proceed.

The result is a single, auditable verdict — never a probabilistic guess.

---

## Endpoints

| Service | URL |
| :--- | :--- |
| **MCP Server** | `https://backendai-x4m1.onrender.com/mcp-http/mcp` |
| **MCP Server Card** | `https://backendai-x4m1.onrender.com/.well-known/mcp/server-card.json` |
| **Agent Card (A2A)** | `https://backendai-x4m1.onrender.com/.well-known/agent-card.json` |
| **x402 Discovery** | `https://backendai-x4m1.onrender.com/.well-known/x402.json` |
| **llms.txt** | `https://backendai-x4m1.onrender.com/llms.txt` |
| **Live Terminal (UI)** | `https://collar-b46l.onrender.com` |
### MCP Metadata

| Field | Value |
| :--- | :--- |
| **Transport** | `streamable-http` |
| **MCP Protocol** | `2025-06-18` |
| **Authentication** | None required (fixed Tier 1) |

---

## Installation

### MCP Client Configuration

Add the following to your MCP client configuration (Claude Desktop, Cursor,
VS Code, or any MCP-compatible client):

```json
{
  "mcpServers": {
    "collar-guardrail": {
      "url": "https://backendai-x4m1.onrender.com/mcp-http/mcp"
    }
  }
}
curl -X POST https://backendai-x4m1.onrender.com/mcp-http/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
Available Tools
The MCP server exposes five tools. Each declares an inputSchema,
outputSchema, and behavioral annotations.

Tool	Purpose
evaluate_trade	Pre-trade risk check — returns allow / warn / deny with reasons and audit hash
check_token_safety	Honeypot / contract safety check for any ERC-20
simulate_balance	Read-only balance simulation via eth_call state override
get_supported_assets	Official Robinhood Chain asset registry
verify_audit_trail	Verify the hash-chain integrity of past decisions
Important: Call get_supported_assets first to resolve a symbol to
its canonical contract address. A mismatched address is treated as a
fake-token attempt and denied.

**Yapıştır → Commit → Sonraki parça.**

---

### 🔹 PARÇA 3/6 — Capabilities (Pricing + Market Hours)

```markdown
---

## Capabilities

### Pricing

- Chainlink oracle prices for 34+ Stock Tokens on Robinhood Chain
- `oraclePaused()` corporate-action detection for tokenized equities
- Staleness bounds: 120 s during market hours, 5 days market-closed
- **Uniswap V4 fallback** — when the oracle is stale, paused, or unconfigured,
  reads the V4 pool directly
- **No fabricated fallback** — if neither oracle nor V4 yields a price, the
  verdict is `deny`
- 10-second V4 price cache to keep RPC traffic bounded under load

### US Market Hours

- Full US holiday calendar: New Year, MLK, Presidents, Memorial, Juneteenth,
  Independence, Labor, Thanksgiving, Christmas
- Observed-on-Friday / observed-on-Monday shift for fixed holidays
- Early close: day after Thanksgiving and Christmas Eve (weekday) → 18:00 UTC
### Policy Enforcement

- Tier system (REST API): **1,000 / 2,500 / 5,000 COLR** → **$5K / $25K / $100K USD** limits
- Per-tier rate limiting for runaway-agent detection
- **Cluster-based rate limit** — keyed on wallet fingerprint, so rotating to a
  fresh wallet funded from the same source does not reset the counter
- **Cluster-based daily-loss guard** — 24 h on-chain net flow, keyed on the
  same fingerprint
- **Slippage / MEV guard** — compares oracle reference to live V4 pool spot;
  hard 500 bps floor
- Idempotency via `request_id` (5-minute cache)
- Global kill switch
- Contract address mismatch detection
- Per-wallet blocklist (Tier 2+)

### Contract Safety

- Honeypot detection: bytecode scan, owner check, and sell simulation via
  `eth_call`
- Multi-token `balanceOf` slot discovery with Redis cache

### Authentication & Infrastructure

- JWT authentication via EIP-191 wallet signature (REST API)
- Auth rate limiting per IP (20 req/min)
- WalletConnect v2 + EIP-6963 wallet discovery
- 200+ token registry
- **Hash-chained audit trail** — every verdict is written with an `audit_hash`
  and `audit_seq`; the chain can be verified end-to-end
---

## Payments (x402)

For paid, high-volume agent traffic, Collar exposes an x402 payment endpoint:

| Field | Value |
| :--- | :--- |
| **Endpoint** | `POST /api/x402/analyze` |
| **Price** | $0.01 USDG per call |
| **Network** | Robinhood Chain (`eip155:4663`) |
| **Discovery** | `/.well-known/x402.json` |
| **Facilitator** | Ultravioleta DAO (`facilitator.ultravioletadao.xyz`) |
| **Listing** | Registered in the x402 Bazaar |

No signup, no API key. The payment header is verified by the facilitator
and the verdict is returned in the same response.

---

## Response Format

Every `evaluate_trade` verdict returns a consistent, machine-readable payload:

```json
{
  "decision": "allow",
  "reasons": ["price source: oracle", "within tier 1 ceiling"],
  "tier": 1,
  "max_trade_usd": 5000,
  "calculated_notional_usd": 1452.30,
  "price_usd": 145.23,
  "price_source": "oracle",
  "risk_score": 12,
  "request_id": "uuid-v4",
  "audit_hash": "sha256:...",
  "audit_seq": 12345,
  "daily_pnl_usd": 0.0
}
Field	Type	Description
decision	string	allow, warn, or deny
reasons	array	Human-readable explanations. Entries prefixed with ADVISORY: are non-blocking
tier	integer	Tier the wallet was evaluated at (1–3)
max_trade_usd	number	USD ceiling for this tier
calculated_notional_usd	number	USD value of the proposed trade
price_usd	number	Asset price used for valuation
price_source	string	oracle, uniswap_v4, fallback_default, or unavailable
risk_score	integer	0–100; higher means more risk
request_id	string	Echoed idempotency key, if provided
audit_hash	string	SHA-256 hash chaining this decision
audit_seq	integer	Monotonic sequence number
daily_pnl_usd	number	Realized 24 h PnL for the wallet, in USD
error	string	If present, no verdict was produced. Treat as a hard stop


**Yapıştır → Commit → Sonraki parça.**

---

### 🔹 PARÇA 6/6 — Discovery + Registry + Documentation + License

```markdown
---

## Discovery Files

Collar exposes a standard set of discovery documents so autonomous agents
can find and use the service without human intervention.

| Path | Purpose |
| :--- | :--- |
| `/.well-known/mcp/server-card.json` | MCP tool schemas and transport metadata |
| `/.well-known/agent-card.json` | A2A skills and capability declarations |
| `/.well-known/x402.json` | x402 payment resources |
| `/llms.txt` | Full API contract written for LLM-based agents |
| `/skill.md` | Concise skill index |
| `/robots.txt` | AI crawler allow-list |
| `/sitemap.xml` | Site map |

---

## Registry Listings

| Registry | Entry |
| :--- | :--- |
| **Official MCP Registry** | `io.github.aiguardrail/backend` |
| **Smithery** | Search "collar" |
| **x402 Bazaar** | `/api/x402/analyze` |

---

## Documentation

| Resource | URL |
| :--- | :--- |
| **Live Terminal (UI)** | `https://collar-b46l.onrender.com` |
| **Agent Integration Guide** | `https://backendai-x4m1.onrender.com/agent-docs.html` |
| **API Reference** | `https://backendai-x4m1.onrender.com/docs.html` |
| **MCP Server Card** | `https://backendai-x4m1.onrender.com/.well-known/mcp/server-card.json` |

---

## Contact

- **X (Twitter):** [@collarfamily](https://x.com/collarfamily)

---

## Disclosure

Collar is independent third-party infrastructure. Not built, operated,
or endorsed by Robinhood. COLR is a separate token, not affiliated with
Robinhood Markets, Inc.

---

## License

This project is licensed under the MIT License. See [LICENSE](./LICENSE)
for the full text.
