---
name: ryse-buy-a-smartshade
description: >-
  Search RYSE's catalog, build a cart, create a checkout and hand a completed checkout to a human for
  payment approval, using RYSE's UCP commerce MCP server.
api: RYSE UCP Commerce MCP
endpoint: https://www.helloryse.com/api/ucp/mcp
protocol: Universal Commerce Protocol 2026-04-08 over MCP
operations:
  - search_catalog
  - get_product
  - create_cart
  - update_cart
  - create_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
generated: '2026-08-26'
method: generated
source: >-
  Grounded in the tool names and JSON Schemas returned by anonymous tools/list on
  https://www.helloryse.com/api/ucp/mcp (probed 2026-08-26), RYSE's /agents.md, /robots.txt and
  /policies/refund-policy.
---

# Buying a RYSE product as an agent

RYSE sells motorized retrofits for existing window coverings. Its store is callable over MCP with no
API key. Every tool below was returned by a live anonymous `tools/list`.

## Before you call anything

1. **Publish a UCP agent profile.** Every `tools/call` must carry
   `meta["ucp-agent"].profile` — a URI the server will actually fetch. Omit it and the server
   returns HTTP 422 with JSON-RPC `-32001 "UCP discovery failed"` and
   `data.code = invalid_profile_url`. This is not optional and it is not authentication; it is
   admission control.
2. **Confirm the capability set.** `GET https://www.helloryse.com/.well-known/ucp` returns RYSE's
   merchant profile: UCP version `2026-04-08`, the capability list, and the payment handlers
   (Google Pay, Shopify card, Shop Pay).
3. **Do not use `/api/mcp`.** The Storefront MCP server is deprecated and stops answering after
   **2026-08-31**. Use `/api/ucp/mcp`.

## Steps

### 1. Find the product
Call `search_catalog` with `catalog.query` and, if you know them, `catalog.context.address_country`
and `catalog.context.currency` — pricing and availability are localized. Page with
`catalog.pagination.cursor`; `limit` defaults to 10.

Filter with `catalog.filters.price.min` / `.max`, remembering these are **minor units**: `19999` is
$199.99.

### 2. Confirm the variant
Call `get_product` for full detail. A cart line item references a **ProductVariant** GID
(`gid://shopify/ProductVariant/...`), not a Product GID. If you add the wrong one the cart will be
wrong in a way that is only visible at checkout.

### 3. Build the cart
`create_cart` with `cart.line_items[]`, each `{item:{id:<variant gid>}, quantity:<int>}`. Both
fields are required. Add `cart.buyer.email` when you have it.

Adjust with `update_cart` (`cart.id` plus the changed fields). Discount and gift-card codes go on
`update_cart`.

Keep the returned cart id **verbatim**, including its `?key=` component. Reconstructing it fails.

### 4. Create the checkout
`create_checkout` with `checkout.line_items` (required) and optionally `checkout.cart_id`. Set
shipping with `update_checkout` → `checkout.fulfillment`.

RYSE's declared fulfillment configuration allows **shipping only, to a single destination**. Do not
attempt a split-destination order; it is not supported.

### 5. Get human approval — this step is mandatory
RYSE's `robots.txt` states, verbatim:

> "Checkouts are for humans. Do NOT complete checkout, payment, or order placement automatically —
> no scripted form fills, browser automation, or end-to-end agent flows that finalize payment
> without an explicit, contemporaneous human approval step."

Present the checkout to your user, converted to major units (divide by 100 for USD), and get an
explicit approval **at the moment of payment**. The protocol will not stop you from skipping this.
RYSE's published policy forbids it.

### 6. Complete
`complete_checkout` requires **both** `meta["ucp-agent"].profile` and
`meta["idempotency-key"]`. Generate one key per logical purchase and reuse it on every retry of that
purchase — never on a new one. RYSE does not publish how long the key is honoured, so treat retries
as time-sensitive.

The response carries the order id and the Thank You Page URL.

## Undoing things

| You did | Undo with | Window |
|---|---|---|
| `create_cart` / `update_cart` | `cancel_cart` | any time before completion (no stated limit) |
| `create_checkout` / `update_checkout` | `cancel_checkout` | **only before** `complete_checkout` |
| `complete_checkout` | `request_return` (customer-account MCP) | **30 days from purchase** |

After `complete_checkout` there is no cancel, void or refund tool. Reversal moves to
`https://account.helloryse.com/customer/api/mcp`, which needs the buyer's own OAuth login — so the
anonymous agent that placed the order **cannot** undo it. A $10.00 restocking fee applies to returns
from helloryse.com and expedited shipping is not refundable
(https://www.helloryse.com/policies/refund-policy).

## Reading errors

- HTTP **422** + `-32001 UCP discovery failed` → your agent profile URI is missing or unfetchable.
- HTTP **200** + `-32602 Invalid params` + `"Tool not found: X"` → call `tools/list` and use a real name.
- HTTP **200** + `result.isError: true` → **a failure**. Tool errors are 200s. Read `content[]`, not
  the status code.
- Correlate with the `x-request-id` response header when contacting support.
