# Collar Guardrail

**Deterministic pre-trade risk layer for AI agents on Robinhood Chain.**

Collar Guardrail evaluates proposed trades before execution and returns
`allow` / `warn` / `deny` with reasons, a 0–100 risk score, and a
tamper-evident SHA-256 audit hash.

---

## Overview

Autonomous trading agents can move capital faster than any human can review.
Collar Guardrail sits between an agent and the chain as a deterministic
pre-trade checkpoint: every proposed trade is scored against live oracle
prices, market-hours rules, tier-based policy limits, slippage bounds, and
contract-safety signals before it is allowed to proceed.

The result is a single, auditable verdict — never a probabilistic guess.

---

## Endpoint

| Field | Value |
| :--- | :--- |
| **URL** | `https://backendai-x4m1.onrender.com/mcp-http/mcp` |
| **Transport** | `streamable-http` |
| **MCP Protocol** | `2025-06-18` |
| **Discovery** | `https://backendai-x4m1.onrender.com/.well-known/mcp/server-card.json` |

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
```

### Verify the Connection

```bash
curl -X POST https://backendai-x4m1.onrender.com/mcp-http/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

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

- Tier system: **1,000 / 2,500 / 5,000 COLR** → **$5K / $25K / $100K USD** limits
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

- JWT authentication via EIP-191 wallet signature
- Auth rate limiting per IP (20 req/min)
- WalletConnect v2 + EIP-6963 wallet discovery
- 200+ token registry
- **x402 payment endpoint** at `/api/x402/analyze` — USDG, Robinhood Chain,
  $0.01 per call
- **Hash-chained audit trail** — every verdict is written with an `audit_hash`
  and `audit_seq`; the chain can be verified end-to-end

---

## Response Format

Every verdict returns a consistent, machine-readable payload:

```json
{
  "verdict": "allow | warn | deny",
  "risk_score": 0,
  "reasons": ["..."],
  "audit_hash": "sha256:...",
  "audit_seq": 12345,
  "daily_pnl_usd": 0.0
}
```

| Field | Type | Description |
| :--- | :--- | :--- |
| `verdict` | string | `allow`, `warn`, or `deny` |
| `risk_score` | integer | 0–100; higher means more risk |
| `reasons` | array | Human-readable explanations for the verdict |
| `audit_hash` | string | SHA-256 hash of the verdict for tamper evidence |
| `audit_seq` | integer | Monotonic sequence number in the audit chain |
| `daily_pnl_usd` | number | Realized 24 h PnL for the wallet, in USD |

---

## Documentation

| Resource | URL |
| :--- | :--- |
| **Agent Integration Guide** | `https://collar-b46l.onrender.com/agent-docs.html` |
| **MCP Server Card** | `https://backendai-x4m1.onrender.com/.well-known/mcp/server-card.json` |

---

## License

This project is licensed under the MIT License. See [LICENSE](./LICENSE) for
the full text.
