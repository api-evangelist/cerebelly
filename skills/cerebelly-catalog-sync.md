---
name: cerebelly-catalog-sync
description: >-
  Pull Cerebelly's full product catalog and editorial content over the anonymous
  Storefront GraphQL API with proper cursor pagination, for syncing, comparison
  shopping, or building a local index. Use for "sync the Cerebelly catalog", "list
  every product and price", "pull their blog posts".
generated: '2026-08-09'
method: generated
source: graphql/cerebelly-storefront.graphql
api: cerebelly-storefront-graphql
endpoint: https://cerebelly.com/api/2026-01/graphql.json
authentication: none
operations:
  - products
  - collections
  - product
  - productByHandle
  - articles
  - blogs
  - pages
  - localization
  - shop
  - publicApiVersions
---

# Sync the Cerebelly catalog over GraphQL

`https://cerebelly.com/api/2026-01/graphql.json` answers **anonymously**, including
full introspection. The captured schema is at
`graphql/cerebelly-storefront.graphql` — 416 types, 35 root queries, 41 mutations.

Use this surface instead of the MCP endpoint when you need bulk reads: MCP has no
pagination at all, GraphQL is fully cursor-paginated.

## Step 0 — pick a version deliberately

Version is a path segment, `/api/{YYYY-MM}/graphql.json`. Ask the API which
versions are live rather than hardcoding:

```graphql
{ publicApiVersions { handle displayName supported } }
```

As of 2026-08-09 that returns `2025-10`, `2026-01`, `2026-04` and `2026-07` as
supported, `2026-10` as a release candidate, plus `unstable`.

Pin a supported version explicitly. There is **no published deprecation policy and
no Sunset header** on this surface (`lifecycle/cerebelly-lifecycle.yml`), so
nothing will warn you before a version stops working — re-run this query on a
schedule.

## Step 1 — page through products

Every list field is a Relay connection: `first`/`after`, `edges { node }`,
`pageInfo { hasNextPage endCursor }`.

```graphql
query Products($cursor: String) {
  products(first: 100, after: $cursor) {
    edges {
      cursor
      node {
        id
        handle
        title
        productType
        vendor
        tags
        availableForSale
        priceRange { minVariantPrice { amount currencyCode } }
        variants(first: 50) {
          edges { node { id sku title price { amount currencyCode } availableForSale quantityAvailable } }
        }
        images(first: 10) { edges { node { url altText } } }
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

Loop while `pageInfo.hasNextPage`, passing `pageInfo.endCursor` as `$cursor`.

## Step 2 — watch your query cost

Every response carries a cost extension:

```json
"extensions": { "cost": { "requestedQueryCost": 3 } }
```

Rate limiting here is **cost-based**, not request-count based, and no rate-limit
headers are returned. Read `requestedQueryCost` off each response and reduce
`first:` or trim requested fields if it climbs. Deeply nested connections multiply
cost quickly — a `products(first:250)` with `variants(first:250)` inside it is a
very different query from the same page size flattened.

## Step 3 — localize correctly

Prices and availability are locale-dependent. Discover what is available:

```graphql
{ localization { availableCountries { isoCode currency { isoCode } } availableLanguages { isoCode } } }
```

Then apply it with the `@inContext` directive — note this is a **directive**, not
an argument, and it is the GraphQL analogue of the `context` object on the MCP
tools:

```graphql
query Products @inContext(country: US, language: EN) { ... }
```

## Step 4 — editorial content

Cerebelly runs a real blog at `/blogs/blog` with an Atom feed at
`/blogs/news.atom`. Over GraphQL:

```graphql
{
  blogs(first: 10) {
    edges { node {
      handle title
      articles(first: 50) {
        edges { node { id handle title publishedAt excerpt authorV2 { name } } }
        pageInfo { hasNextPage endCursor }
      }
    } }
  }
  pages(first: 50) { edges { node { handle title bodySummary } } }
}
```

## Errors

GraphQL splits errors in two, and you must check both:

- **Transport** — a top-level `errors[]` array with `message`, `path`, `extensions`.
- **Domain** — typed `userErrors[]` on mutation payloads, implementing the
  `DisplayableError` interface with `field`, `message` and a typed `code` enum.

A `200 OK` with a populated `errors[]` is a failure. Catalog:
`errors/cerebelly-problem-types.yml`.

## Cheaper alternative for a flat product dump

If you need nothing but products, the plain JSON endpoint avoids GraphQL entirely:

```
GET https://cerebelly.com/products.json?limit=250&page=1
GET https://cerebelly.com/collections/{handle}/products.json
```

It returns a bare `{"products":[...]}` envelope with **no total count and no link
header** — page until you get an empty array. Note that it uses bare numeric ids,
not the `gid://shopify/...` GIDs used by GraphQL and MCP, so the two surfaces'
identifiers are not interchangeable. See
`data-model/cerebelly-data-model.yml`.
