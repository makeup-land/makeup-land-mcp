# makeup.land MCP server

Live MCP server for AI agents (Claude Desktop, Cursor, LangGraph, custom integrations) to query the makeup.land catalog and authenticated customer data.

## Endpoint

```
POST https://makeup.land/api/mcp
Content-Type: application/json
Authorization: Bearer ml_<your-token>  (optional for catalog tools)
```

Discovery: `https://makeup.land/.well-known/mcp.json`
A2A agent card: `https://makeup.land/.well-known/agent-card.json`
Protocol version: `2025-06-18` (Streamable-HTTP MCP, stateless)

## Authentication

Aligned exactly with the V1 REST API. The server forwards your bearer header to V1 routes; V1's runtime gate enforces.

| Tool | Auth |
|---|---|
| `list_products` (basic catalog browse) | anonymous OK |
| `list_products` (`phone=` rewards projection) | bearer required |
| `validate_gift_card` | public (gated on code) |
| `list_brands` | bearer required |
| `get_customer` | bearer required |
| `get_cart` | bearer required + phone arg |
| `list_orders` | bearer required + phone arg |
| `list_payment_links` | bearer required + phone arg |
| `get_customer_best_deals` | bearer required + phone arg |

Bearer tokens are issued manually — write to `shop@makeup.land` describing your integration. Tokens carry a scope and an optional `read_only` flag; the V1 layer enforces both.

## v1 tool surface

Eight read-only tools. Mutating tools (cart writes, gift-card redeem, customer registration) ship in v2.

```
list_products            Browse / search catalog (anonymous-OK).
list_brands              List all brands with product counts.
validate_gift_card       Check gift-card balance by code (public).
get_customer             Customer profile, wallet balance, M Club tier.
get_cart                 Most-recent cart with reward projections.
list_orders              Recent orders with 6-axis status.
list_payment_links       Pending payment requests on unpaid orders.
get_customer_best_deals  Personalised deals based on tags + tier.
```

## Quick test

### From the terminal (raw JSON-RPC)

```bash
# Initialize
curl -s -X POST https://makeup.land/api/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-06-18",
      "capabilities": {},
      "clientInfo": { "name": "curl", "version": "1.0" }
    }
  }'

# List tools
curl -s -X POST https://makeup.land/api/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{ "jsonrpc": "2.0", "id": 2, "method": "tools/list" }'

# Anonymous catalog browse — no bearer
curl -s -X POST https://makeup.land/api/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0", "id": 3, "method": "tools/call",
    "params": {
      "name": "list_products",
      "arguments": { "near_hex": "#C2185B", "limit": 5 }
    }
  }'

# Customer-keyed call — bearer required
curl -s -X POST https://makeup.land/api/mcp \
  -H "Authorization: Bearer ml_<your-token>" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0", "id": 4, "method": "tools/call",
    "params": {
      "name": "get_customer",
      "arguments": { "phone": "+972501234567" }
    }
  }'
```

### From Claude Desktop

Add to `claude_desktop_config.json` (use the `mcp-remote` bridge since we expose HTTP, not stdio):

```json
{
  "mcpServers": {
    "makeup.land": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote",
        "https://makeup.land/api/mcp",
        "--header", "Authorization:Bearer ml_<your-token>"
      ]
    }
  }
}
```

Anonymous-only setup (omit the bearer):

```json
{
  "mcpServers": {
    "makeup.land": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://makeup.land/api/mcp"]
    }
  }
}
```

## Error envelope

Tool errors propagate from V1's REST error envelope as text content with `isError: true`:

```json
{
  "content": [{ "type": "text", "text": "{ \"error\": \"...\", \"error_code\": \"...\" }" }],
  "isError": true
}
```

Stable `error_code` values are documented in the OpenAPI spec (`components.schemas.Error`). Common codes:

- `unauthorized` — missing or invalid bearer
- `scope_mismatch` — token has wrong scope for the tool
- `read_only_token` — write attempted with a read-only token
- `customer_not_found` — phone doesn't match any customer
- `endpoint_not_found` — typo or removed V1 path

## Limitations of v1

- Read-only — no cart writes, no gift-card redeem, no registration. v2 adds these.
- Stateless — every request initialises a fresh session. Resumable sessions ship if a real use case emerges.
- Synchronous — `tools/call` blocks on the V1 internal fetch. Most calls return <500ms; semantic search may take up to 2s.
- No SSE streaming for partial results.

## See also

- [/openapi.json](https://makeup.land/openapi.json) — full V1 REST spec
- [/llms-full.txt](https://makeup.land/llms-full.txt) — long-form agent handbook
- [/auth.md](https://makeup.land/auth.md) — token acquisition flow
- [/.well-known/mcp.json](https://makeup.land/.well-known/mcp.json) — discovery manifest
- [/.well-known/agent-card.json](https://makeup.land/.well-known/agent-card.json) — A2A agent card
