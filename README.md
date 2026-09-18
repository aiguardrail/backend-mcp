# Collar Guardrail

**Deterministic pre-trade risk layer for AI agents on Robinhood Chain.**

Collar evaluates proposed trades before execution and returns
`allow` / `warn` / `deny` with reasons, a 0–100 risk score, and a
tamper-evident SHA-256 audit hash.

[![MCP Registry](https://img.shields.io/badge/MCP%20Registry-v1.0.2-brightgreen)](https://registry.modelcontextprotocol.io/servers/io.github.aiguardrail/backend)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)

## Overview

Collar sits between an autonomous trading agent and the chain as a
deterministic pre-trade checkpoint. Every proposed trade is scored against
live oracle prices, market-hours rules, tier-based limits, slippage bounds,
and contract-safety signals before it is allowed to proceed.

The result is a single, auditable verdict — never a probabilistic guess.

## Endpoints

- **MCP Server:** `https://backendai-x4m1.onrender.com/mcp-http/mcp`
- **MCP Server Card:** `https://backendai-x4m1.onrender.com/.well-known/mcp/server-card.json`
- **Agent Card (A2A):** `https://backendai-x4m1.onrender.com/.well-known/agent-card.json`
- **x402 Discovery:** `https://backendai-x4m1.onrender.com/.well-known/x402.json`
- **llms.txt:** `https://backendai-x4m1.onrender.com/llms.txt`
- **Live Terminal (UI):** `https://collar-b46l.onrender.com`

MCP Transport: `streamable-http` | Protocol: `2025-06-18` | Auth: none (fixed Tier 1)

## Installation

Add to your MCP client config (Claude Desktop, Cursor, VS Code):

```json
{
  "mcpServers": {
    "collar-guardrail": {
      "url": "https://backendai-x4m1.onrender.com/mcp-http/mcp"
    }
  }
}
```

Verify:

```bash
curl -X POST https://backendai-x4m1.onrender.com/mcp-http/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

## Available Tools

| Tool | Purpose |
| :--- | :--- |
| `evaluate_trade` | Pre-trade risk check — returns allow / warn / deny with reasons and audit hash. **Honeypot findings (severity=danger) force a DENY.** |
| `check_token_safety` | Honeypot / contract safety check for any ERC-20. **When severity=danger, evaluate_trade auto-DENYs.** |
| `simulate_balance` | Read-only balance simulation via `eth_call` state override |
| `get_supported_assets` | Official Robinhood Chain asset registry |
| `verify_audit_trail` | Verify the hash-chain integrity of past decisions |

> Call `get_supported_assets` first to resolve a symbol to its canonical
> contract address. A mismatched address is treated as a fake-token attempt
> and denied.

## Capabilities

**Pricing**
- Chainlink oracle prices for 34+ Stock Tokens on Robinhood Chain
- Uniswap V4 fallback when the oracle is stale, paused, or unconfigured
- No fabricated fallback — if neither oracle nor V4 yields a price, verdict is `deny`
- 10-second V4 price cache

**US Market Hours**
- Full US holiday calendar with observed-on-Friday/Monday shift
- Early close on Thanksgiving Friday and Christmas Eve

**Policy Enforcement**
- Tier system (REST API): 1,000 / 2,500 / 5,000 COLR → $5K / $25K / $100K limits
- Per-tier rate limiting
- Cluster-based rate limit and daily-loss guard (keyed on wallet fingerprint)
- Slippage / MEV guard (hard 500 bps floor)
- Idempotency via `request_id` (5-minute cache)
- Global kill switch, contract mismatch detection, per-wallet blocklist

**Contract Safety**
- Honeypot detection: bytecode scan, owner check, sell simulation via `eth_call`
- **Auto-DENY:** when severity=danger, the verdict is `deny` regardless of
  other checks — a hard stop.
- GoPlus intelligence for unregistered tokens (buy/sell tax, mintable,
  proxy, ownership reclaim).
- Excessive tax (buy or sell > 10%) triggers an automatic `deny` via
  GoPlus when `allow_unregistered=true` is passed to the request.

**Auth & Infrastructure**
- JWT authentication via EIP-191 wallet signature (REST API)
- Auth rate limiting per IP (20 req/min)
- WalletConnect v2 + EIP-6963 wallet discovery
- 200+ token registry
- Hash-chained audit trail (`audit_hash` + `audit_seq`)

## Payments (x402)

- **Endpoint:** `POST /api/x402/analyze`
- **Price:** $0.01 USDG per call
- **Network:** Robinhood Chain (`eip155:4663`)
- **Discovery:** `/.well-known/x402.json`
- **Facilitator:** Ultravioleta DAO
- **Listing:** x402 Bazaar

No signup, no API key. Registered in the x402 Bazaar for automatic agent discovery.

## Response Format

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
  "audit_hash": "sha256:...",
  "audit_seq": 12345,
  "daily_pnl_usd": 0.0
}
```

- `decision`: `allow` / `warn` / `deny` — deny is a hard stop
- `reasons` prefixed with `ADVISORY:` are non-blocking
- `price_source`: `oracle` / `uniswap_v4` / `fallback_default` / `unavailable`
- `error`: if present, no verdict was produced — treat as a hard stop

## Discovery Files

- `/.well-known/mcp/server-card.json` — MCP tool schemas
- `/.well-known/agent-card.json` — A2A skills
- `/.well-known/x402.json` — x402 payment resources
- `/llms.txt` — full API contract for LLM agents
- `/skill.md` — concise skill index
- `/robots.txt` — AI crawler allow-list
- `/sitemap.xml` — site map

## Registry Listings

- **Official MCP Registry:** `io.github.aiguardrail/backend`
- **Smithery:** search "collar"
- **x402 Bazaar:** `/api/x402/analyze`

## Documentation

- **Live Terminal (UI):** https://collar-b46l.onrender.com
- **Agent Integration Guide:** https://collar-b46l.onrender.com/agent-docs.html
- **MCP Server Card:** https://backendai-x4m1.onrender.com/.well-known/mcp/server-card.json

## Contact

X (Twitter): [@collarfamily](https://x.com/collarfamily)

## Disclosure

Collar is independent third-party infrastructure. Not built, operated,
or endorsed by Robinhood. COLR is a separate token, not affiliated with
Robinhood Markets, Inc.

## License

MIT — see [LICENSE](./LICENSE).
