---
name: cerebelly-cart-and-checkout
description: >-
  Build a cart and run a Cerebelly checkout to completion over the UCP commerce MCP
  endpoint, respecting the store's required human-approval rule on payment and the
  mandatory idempotency key on completion. Use for "add this to my Cerebelly cart",
  "check out", "what will this order cost with shipping".
generated: '2026-08-09'
method: generated
source: mcp/cerebelly-ucp-tools-list.json
api: cerebelly-ucp-commerce-mcp
endpoint: https://cerebelly.com/api/ucp/mcp
authentication: none for cart; buyer-approved payment handler for completion
operations:
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
---

# Cart and checkout on Cerebelly

## Read this first — the store's rule on payment

Cerebelly states the same rule in three places: `llms.txt`, `agents.md`, and the
comment header of `robots.txt`.

> "Checkouts are for humans. Do NOT complete checkout, payment, or order placement
> automatically — no scripted form fills, browser automation, or end-to-end agent
> flows that finalize payment without an explicit, contemporaneous human approval
> step."

`complete_checkout` moves real money. Do not call it without contemporaneous buyer
approval. If you cannot obtain that approval at the moment of payment, Cerebelly's
own instructions say to route the purchase through the Shop skill at
`https://shop.app/SKILL.md` instead.

## Prerequisites

- `meta.ucp-agent.profile` on **every** call — a URI identifying your agent.
- A **product-variant** id from `skills/cerebelly-product-discovery.md`. Product
  ids will not work.

## Step 1 — create the cart

`create_cart` requires `meta` and `cart`.

```json
{
  "meta": { "ucp-agent": { "profile": "https://example.com/agents/my-agent" } },
  "cart": {
    "line_items": [
      { "item": { "id": "gid://shopify/ProductVariant/..." }, "quantity": 2 }
    ],
    "buyer": { "email": "buyer@example.com" },
    "context": { "address_country": "US", "currency": "USD" }
  }
}
```

`line_items[]` requires `item` and `quantity` on each entry.

## Step 2 — adjust

`update_cart` takes `meta`, the cart `id`, and a `cart` patch. It is one composite
tool standing in for eleven granular GraphQL mutations — lines, note, attributes,
buyer identity, discount codes, gift cards, delivery addresses and delivery option
selection all go through it. See `mcp/cerebelly-tool-crosswalk.yml`.

Read back with `get_cart`, abandon with `cancel_cart`.

**Check warnings, not just errors.** The schema defines a 17-member
`CartWarningCode` family returned *alongside* successful mutations —
`MERCHANDISE_OUT_OF_STOCK`, `DISCOUNT_CODE_NOT_HONOURED`,
`DISCOUNT_USAGE_LIMIT_REACHED` and more. A cart mutation can succeed and still
have silently dropped a discount or exceeded stock. Ignoring warnings is how an
agent mis-quotes a total.

## Step 3 — create the checkout

`create_checkout` takes `meta` and `checkout`, and returns line items, totals,
discounts and taxes. Checkout ids are `gid://shopify/Checkout/{id}`.

Note that `Checkout` exists **only on this MCP surface** — there is no Checkout
type in the Storefront GraphQL schema.

## Step 4 — fulfillment and payment method

`update_checkout` sets the shipping address and delivery method, and attaches a
payment instrument under `checkout.payment.instruments[]`. Each instrument
requires `id`, `handler_id` and `type`.

`handler_id` must be one of the handlers Cerebelly actually has configured, which
you can read live from `https://cerebelly.com/.well-known/ucp`:

| handler_id     | family                | notes                                       |
|----------------|-----------------------|---------------------------------------------|
| `gpay`         | `com.google.pay`      | Visa, Mastercard, Amex, Discover            |
| `shopify.card` | `dev.shopify.card`    | adds Diners Club                            |
| `shop_pay`     | `dev.shopify.shop_pay`| Shop Pay                                    |

Use `type: "card"` for credit/debit and `type: "token"` for wallet payments.

Re-read totals with `get_checkout` before you present a number to the buyer —
taxes and shipping are only resolved once the address is set.

## Step 5 — complete (human approval required)

`complete_checkout` is the only tool on the entire surface that requires an
idempotency key, and it is **required**, not optional:

```json
{
  "meta": {
    "ucp-agent": { "profile": "https://example.com/agents/my-agent" },
    "idempotency-key": "<stable key for this purchase attempt>"
  },
  "id": "gid://shopify/Checkout/...",
  "checkout": { }
}
```

Rules for the key:

- Generate it **once per purchase intent** and reuse it verbatim on every retry.
- Never generate a fresh key on retry — that is how you double-charge a buyer.
- It is enforced, not merely accepted: `IDEMPOTENCY_KEY_ALREADY_USED` is a
  published error code on this platform.
- No retention window is published, so do not assume a key stays valid indefinitely.

Cancel an unfinished checkout with `cancel_checkout`.

## Step 6 — the order

`complete_checkout` yields an order. `get_order` reads it back by
`gid://shopify/Order/{id}`.

Be aware this read is thin anonymously — the Storefront schema exposes no
anonymous order query, and the Customer Account API at
`https://account.cerebelly.com/customer/api/2026-01/graphql` returns **401**
without an OIDC token carrying `customer-account-api:full`.

## Failure handling

| condition | what you get | what to do |
|---|---|---|
| rate limit | HTTP `429` | back off, retry |
| transient payment failure | `PAYMENT_TRANSIENT_ERROR` | retry **with the same idempotency key** |
| card declined | `PAYMENT_CARD_DECLINED` | do not retry; ask the buyer for another instrument |
| stock gone | `INVENTORY_RESERVATION_ERROR` | re-check the cart, re-quote |
| address rejected | a `DELIVERY_*` member of `SubmissionErrorCode` (95 codes) | fix the named field and resubmit |

Full code families with counts: `errors/cerebelly-problem-types.yml`.
Cross-cutting rules: `conventions/cerebelly-conventions.yml`.
