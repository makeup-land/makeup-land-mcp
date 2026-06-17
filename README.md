# makeup.land MCP server

> Public metadata + integrator guide for the [makeup.land](https://makeup.land) Model Context Protocol server.

makeup.land is an Israeli professional cosmetics retailer. Our MCP server lets AI agents (Claude Desktop, Cursor, ChatGPT browsing, LangGraph apps, custom integrations) browse our catalog, look up customers, fetch carts, and check gift cards — all through the Model Context Protocol's standard JSON-RPC interface.

The MCP server is hosted at **`https://makeup.land/api/mcp`** as a Streamable-HTTP endpoint (protocol version 2025-06-18, stateless). The catalog tools work anonymously; the customer-data tools require a bearer token issued via `shop@makeup.land`.

This repo is the **public face** of the integration — the server runs from our private storefront codebase, but everything an integrator needs (tool inventory, auth model, connection snippets, error envelopes) lives here.

## What you can do with it

| Tool | What it returns | Auth |
|---|---|---|
| `list_products` | Catalog browse: cross-lingual semantic search (`q`), ΔE-ranked shade matching (`near_hex`, returns `shade_match: {hex, delta_e}` per product), brand / exact-Hebrew tag filters, sort by price / popularity / Bayesian-shrunk rating | Anonymous |
| `validate_gift_card` | Gift card balance | Public (gated on the gift-card code) |
| `list_brands` | Every brand we carry — Yossi Bitton's B Cosmic, da Vinci (Defet) brushes, INGLOT, NYX, and dozens more | Bearer |
| `get_customer` | Customer profile, ℳ-credit wallet, M Club tier | Bearer |
| `get_cart` | Customer's most-recent cart with per-line + total reward projection | Bearer + phone |
| `list_orders` | Customer's recent orders with 6-axis status | Bearer + phone |
| `list_payment_links` | Pending payment requests on unpaid orders | Bearer + phone |
| `get_customer_best_deals` | Personalised deal projections based on tags + M Club tier | Bearer + phone |

## Quickstart

### From the terminal

```bash
# Anonymous catalog browse — find lipsticks similar to a target shade
curl -X POST https://makeup.land/api/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "list_products",
      "arguments": { "near_hex": "#C2185B", "q": "lipstick", "limit": 5 }
    }
  }'

# List all Yossi Bitton (B Cosmic) products — bearer required
curl -X POST https://makeup.land/api/mcp \
  -H 'Authorization: Bearer ml_<your-token>' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "list_products",
      "arguments": { "brand": "yossi-bitton", "limit": 20 }
    }
  }'
```

### From Claude Desktop

Use the `mcp-remote` bridge (Claude Desktop reads MCP via stdio; this bridges to our HTTP endpoint):

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

Anonymous catalog-only access (skip the bearer):

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

### From Cursor

Cursor reads from `~/.cursor/mcp.json` (or per-workspace `.cursor/mcp.json`). Same shape as Claude Desktop above.

## Brands we carry

The catalog spans ~150 brands. Some highlights:

- **Yossi Bitton (B Cosmic)** — Israeli professional makeup line by stylist [Yossi Bitton](https://es.wikipedia.org/wiki/Yossi_Bitton). makeup.land is the official online store.
- **da Vinci (Defet)** — Authorised Israeli distributor for [da Vinci's](https://www.davinci-makeupbrushes.com/) German-made professional cosmetic brushes.
- **Yarin Shahaf** — Cross-catalog partner ([yarin-shahaf.co.il/מייקאפלנד](https://www.yarin-shahaf.co.il/%D7%9E%D7%99%D7%99%D7%A7%D7%90%D7%A4%D7%9C%D7%A0%D7%93)).
- **INGLOT, NYX, Bourjois, Maybelline, L'Oréal**, and 140+ more.

Use `list_brands` to enumerate all of them at runtime.

## ΔE shade matching

The flagship feature. Every variant in our catalog carries swatch hex colours; the `list_products` tool's `near_hex` argument runs a perceptual-distance ranking (CIE ΔE 2000) so an agent can answer "find a lipstick that looks like #C2185B" in one call. Pair with `hue_family=warm|cool|neutral` to refine.

Each returned product carries a **`shade_match: {hex, delta_e}`** field identifying the closest variant swatch and its perceptual distance — so on a 40-shade palette, you know which specific shade was the match (and how close it is), not just which palette.

This is the surface that makes makeup.land's MCP server uniquely useful for shopping agents — no other Israeli cosmetics retailer exposes shade matching through MCP, and very few retailers globally expose it at all.

## Cross-lingual catalog search

Send `q` (natural language) instead of `tag` for category lookups. `q` is a cross-lingual semantic search — `q="lipstick"`, `q="שפתון"`, and `q="lápiz labial"` each return Hebrew-tagged lipsticks. The surfaced product set may differ across query languages (the system ranks by semantic relevance, not by language-canonicalisation), but each is a valid lipstick page. `tag` is a literal-string filter against Hebrew-stored tags only; English category words will not match it.

## Authentication

| Mode | When required | How |
|---|---|---|
| Anonymous | `list_products` (catalog filters only), `validate_gift_card` | Nothing — just call |
| Bearer | All customer-data tools, `list_brands` | `Authorization: Bearer ml_<hex>` |
| Bearer + phone | `get_cart`, `list_orders`, `list_payment_links`, `get_customer_best_deals` | Bearer authenticates the caller (partner integration); phone (E.164) selects the customer |

The bearer auth is **NOT OAuth** despite Smithery's autodetection — tokens are issued out-of-band via email. Request one by writing to `shop@makeup.land` with your integration use case.

### Why bearer + phone?

The phone identifier (`?phone=+972...`) selects **which customer's** resources to return. The bearer authenticates **who** is calling. Neither alone is sufficient — the bearer alone can't enumerate customer records, and a phone alone returns `401 Unauthorized`. This is the V1 REST API's contract; the MCP server matches it exactly.

## Error envelope

Tool errors propagate from our V1 REST error envelope as text content with `isError: true`:

```json
{
  "content": [{
    "type": "text",
    "text": "{ \"error\": \"...\", \"error_code\": \"...\" }"
  }],
  "isError": true
}
```

Stable `error_code` values are documented in the OpenAPI spec (`components.schemas.Error`). Common codes:

- `unauthorized` — missing or invalid bearer
- `scope_mismatch` — token has wrong scope for the tool
- `read_only_token` — write attempted with a read-only token
- `customer_not_found` — phone doesn't match any customer
- `endpoint_not_found` — typo or removed V1 path
- `insufficient_stock` — variant out of stock (relevant once mutating tools ship)
- `insufficient_credits` — wallet balance can't cover a credits-tender purchase

## Limitations of v1

- **Read-only.** No cart writes, no gift-card redeem, no customer registration. v2 adds these.
- **Stateless.** Every request initialises a fresh MCP session.
- **Synchronous.** `tools/call` blocks on the V1 internal fetch. Most calls return <500ms; semantic search can take up to 2s.
- **No SSE streaming** for partial results.

## Discovery surfaces

| Surface | URL |
|---|---|
| MCP discovery manifest | https://makeup.land/.well-known/mcp.json |
| A2A agent card | https://makeup.land/.well-known/agent-card.json |
| OpenAPI 3.1 spec | https://makeup.land/openapi.json |
| Auth skill manifest | https://makeup.land/auth.md |
| Long-form handbook | https://makeup.land/llms-full.txt |
| OAuth resource metadata | https://makeup.land/.well-known/oauth-protected-resource |
| OAuth server metadata | https://makeup.land/.well-known/oauth-authorization-server |

## Listed on

- [Smithery](https://smithery.ai/?q=makeup.land)
- [Official MCP Registry](https://registry.modelcontextprotocol.io/v0/servers?search=makeup) (as `land.makeup/v1`)
- [makeup.land integrator guide](https://makeup.land/llms-full.txt)

## Contact

- **Token requests:** `shop@makeup.land`
- **Bug reports + feature requests:** open an issue on this repo
- **General inquiries:** `shop@makeup.land`

## License

The metadata and documentation in this repo are released under [MIT](./LICENSE) so they can be redistributed by MCP catalog aggregators.

The underlying MCP server, the makeup.land storefront, and our product catalog remain proprietary — see [makeup.land/terms-of-service](https://makeup.land/terms-of-service).
