---
name: cerebelly-product-discovery
description: >-
  Search and resolve products in the Cerebelly catalog over the anonymous UCP
  commerce MCP endpoint, and get back the product-variant ids a cart actually
  needs. Use for "what Cerebelly purees are available", "find the smart bars",
  "price and stock for this product".
generated: '2026-08-09'
method: generated
source: mcp/cerebelly-ucp-tools-list.json
api: cerebelly-ucp-commerce-mcp
endpoint: https://cerebelly.com/api/ucp/mcp
authentication: none
operations:
  - search_catalog
  - lookup_catalog
  - get_product
---

# Discover products in the Cerebelly catalog

Cerebelly's catalog is readable with no credential. The endpoint is an MCP server
at `https://cerebelly.com/api/ucp/mcp`, speaking JSON-RPC 2.0 over HTTP.

## Before you start

Every tool on this surface requires `meta.ucp-agent.profile` — a URI identifying
your agent. There is no exception; `search_catalog` will not run without it.

```json
{
  "meta": { "ucp-agent": { "profile": "https://example.com/agents/my-agent" } }
}
```

Send buyer context whenever you have it. Cerebelly's own `llms.txt` asks for it
directly: *"Pass `context.address_country` and `context.currency` for accurate
pricing and availability."* Without it you may quote the wrong price.

## Step 1 — search

Call `search_catalog` with a free-text query.

```json
{
  "jsonrpc": "2.0", "id": 1, "method": "tools/call",
  "params": {
    "name": "search_catalog",
    "arguments": {
      "meta": { "ucp-agent": { "profile": "https://example.com/agents/my-agent" } },
      "catalog": {
        "query": "organic veggie puree",
        "context": { "address_country": "US", "currency": "USD", "language": "en" }
      }
    }
  }
}
```

`catalog.filters` narrows the result set and `catalog.signals` carries
`dev.ucp.buyer_ip` and `dev.ucp.user_agent` if you have them.

**There is no pagination.** `search_catalog` exposes no cursor, page or offset
parameter. If you need breadth, issue several narrower queries rather than trying
to walk a result set — see `conventions/cerebelly-conventions.yml`.

## Step 2 — resolve ids in bulk

`lookup_catalog` resolves known ids. It accepts **1 to 10 ids** (`minItems: 1`,
`maxItems: 10`) — chunk anything larger.

```json
{ "catalog": { "ids": ["gid://shopify/Product/..."] }, "meta": { "...": "..." } }
```

## Step 3 — get full detail and the variant id

`get_product` returns complete detail for one product. Pass
`catalog.selected[]` to resolve a specific variant by its options.

```json
{
  "catalog": {
    "id": "gid://shopify/Product/...",
    "selected": [{ "name": "Size", "label": "4 oz" }]
  },
  "meta": { "...": "..." }
}
```

## The thing that trips agents up

**Carts key on the product-VARIANT id, not the product id.** `create_cart` takes
`cart.line_items[].item.id` and that field is documented as *"The Product Variant
ID to add or update."* Passing a product id here fails. Always take the variant id
out of `get_product` before you build a cart.

Ids are Shopify Global IDs: `gid://shopify/{Type}/{numeric_id}`.

## Errors and limits

- The endpoint is rate-limited **per IP**. Back off on `429`.
- Errors arrive as the JSON-RPC `error` member — not RFC 9457 problem details.
- Full code families: `errors/cerebelly-problem-types.yml`.

## Alternative read paths

If you only need catalog data and not commerce, two other anonymous surfaces exist:

- Storefront GraphQL: `https://cerebelly.com/api/2026-01/graphql.json`
  (cursor-paginated, introspectable — see `skills/cerebelly-catalog-sync.md`)
- Plain JSON: `https://cerebelly.com/products.json?limit=250&page=1`
