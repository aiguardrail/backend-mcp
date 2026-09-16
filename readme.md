# Collar Guardrail

Deterministic pre-trade risk layer for AI agents on Robinhood Chain.

Collar Guardrail evaluates proposed trades **before execution** and returns
`allow` / `warn` / `deny` with reasons, a 0–100 risk score, and a
tamper-evident SHA-256 audit hash.

---

## Endpoint

| Field | Value |
|---|---|
| **URL** | `https://backendai-x4m1.onrender.com/mcp-http/mcp` |
| **Transport** | `streamable-http` |
| **MCP Protocol** | `2025-06-18` |
| **Discovery** | `https://backendai-x4m1.onrender.com/.well-known/mcp/server-card.json` |

---

## Installation

### Claude Desktop / Cursor / VS Code

Add the following to your MCP client configuration:

```json
{
  "mcpServers": {
    "collar-guardrail": {
      "url": "https://backendai-x4m1.onrender.com/mcp-http/mcp"
    }
  }
}
