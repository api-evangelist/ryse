---
name: ryse-track-and-return-an-order
description: >-
  Check a RYSE customer's order status, read their store credit, and open a return — using RYSE's
  OAuth-protected customer-account MCP server.
api: RYSE Customer Account MCP
endpoint: https://account.helloryse.com/customer/api/mcp
operations:
  - get_most_recent_order_status
  - get_order_status
  - get_store_credit_balances
  - request_return
generated: '2026-08-26'
method: generated
source: >-
  Grounded in the tool names and JSON Schemas returned by anonymous tools/list on
  https://account.helloryse.com/customer/api/mcp, RYSE's /.well-known/oauth-authorization-server and
  /.well-known/oauth-protected-resource (all probed 2026-08-26), and
  https://www.helloryse.com/policies/refund-policy.
---

# Post-purchase: tracking and returning a RYSE order

This is the only RYSE surface that requires a credential, and the only one with a reversal.

## Get authorized first

`tools/list` answers anonymously. **Every `tools/call` does not** — an unauthenticated call returns
HTTP 401 with `{"errors":[{"message":"Unauthorized"}]}` and a `www-authenticate` header pointing at
`https://account.helloryse.com/authentication/.well-known/openid-configuration`.

Run the authorization code flow against RYSE's own authorization server:

- authorize: `https://account.helloryse.com/authentication/oauth/authorize`
- token: `https://account.helloryse.com/authentication/oauth/token`
- PKCE: **required**, `S256` only
- scope: `customer-account-mcp-api:full` (add `openid email` if you need the identity claims)

Send the access token as `Authorization: Bearer <token>`; `bearer_methods_supported` is `["header"]`.

**Scope warning.** `customer-account-mcp-api:full` is the only granularity offered. It is not
possible to request read-only access — the same scope that lets you check an order status also
authorizes `request_return`. Do not treat this token as harmless.

## Steps

### Check the latest order
`get_most_recent_order_status` takes **no arguments**. It resolves against whoever the token belongs
to. Prefer it when the user says "my order" without a number.

### Check a specific order
`get_order_status` requires `order_number` — the human-facing number from the confirmation email,
not a `gid://shopify/Order/...` GID. (The GID form is what the *anonymous* UCP `get_order` takes;
the two order lookups are not interchangeable.)

### Check store credit
`get_store_credit_balances` takes no arguments and returns the customer's balances.

### Open a return
`request_return` requires `line_items` and accepts `order_id` and `order_number`.

Before calling it, confirm with the user, and tell them the terms RYSE publishes:

- The return must be initiated **within 30 days of the original purchase**.
- Refunds are processed **within 30 days of RYSE receiving the unit** at its warehouse.
- A **$10.00 restocking fee** applies to every return from helloryse.com.
- Missing packaging/parts are charged at RYSE's discretion — $10.00 for SmartShade /
  SmartShade + BatteryPack, $5.00 for SmartBridge / BatteryPack accessories.
- Expedited shipping fees are **not** refundable.
- If the item was bought from Amazon, Best Buy or another retailer, **RYSE cannot accept it** — it
  goes back to that retailer, and `request_return` will not help.

Source: https://www.helloryse.com/policies/refund-policy

`request_return` **opens** a return. It does not issue a refund and there is no idempotency key on
it — calling it twice is not safe. Confirm before you call, and do not retry blindly.
