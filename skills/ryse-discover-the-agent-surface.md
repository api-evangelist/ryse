---
name: ryse-discover-the-agent-surface
description: >-
  Discover everything RYSE exposes to agents from a cold start — UCP profile, MCP endpoints, OAuth
  metadata — and pick the right endpoint before the deprecated one is switched off.
api: RYSE UCP Commerce MCP
endpoint: https://www.helloryse.com/.well-known/ucp
operations:
  - tools/list
  - initialize
  - search_shop_policies_and_faqs
generated: '2026-08-26'
method: generated
source: >-
  Grounded in live probes of RYSE's /.well-known/*, /robots.txt, /agents.md, /llms.txt,
  /sitemap_agentic_discovery.xml and all three MCP endpoints on 2026-08-26.
---

# Discovering RYSE's agent surface

RYSE publishes no developer portal, no OpenAPI and no API keys. Everything an agent needs is
discoverable anonymously from four documents.

## The discovery chain

1. `GET https://www.helloryse.com/robots.txt` — names `/agents.md`, `/.well-known/ucp` and the
   UCP/MCP endpoint, and states the human-approval rule for checkout.
2. `GET https://www.helloryse.com/agents.md` — the canonical agent-facing description.
   `/llms.txt` mirrors it. `/sitemap_agentic_discovery.xml` exists and points at `/agents.md`.
3. `GET https://www.helloryse.com/.well-known/ucp` — the machine-readable one. Returns UCP version
   `2026-04-08` (and `2026-01-23`), the `dev.ucp.shopping` service over MCP transport, eight
   capability URNs each bound to a published JSON Schema, and three payment handlers.
4. `GET https://account.helloryse.com/.well-known/oauth-protected-resource` — RFC 9728 metadata for
   the customer surface; `/.well-known/oauth-authorization-server` and
   `/.well-known/openid-configuration` give the flows and scopes.

Nothing else is served: `/.well-known/security.txt`, `/.well-known/api-catalog`,
`/.well-known/ai-plugin.json`, `/.well-known/agent-card.json` and `/.well-known/agent.json` all
return 404 on every RYSE host, as do `/openapi.json` and `/swagger.json`.

## Choosing an endpoint

| Endpoint | Tools | Auth | Use it? |
|---|---|---|---|
| `https://www.helloryse.com/api/ucp/mcp` | 13 | none (needs a UCP agent profile) | **Yes** — this is the current surface |
| `https://www.helloryse.com/api/mcp` | 5 | none | **No** — deprecated, stops working after 2026-08-31 |
| `https://account.helloryse.com/customer/api/mcp` | 4 | customer OAuth | Only for post-purchase, on the buyer's behalf |

Confirm which you are on from the response headers: the UCP server returns
`x-shopify-ucp-mcp-api-version: 2026-04-08`, while the deprecated one returns
`x-shopify-mcp-api-version: unstable` and appends a `DEPRECATION NOTICE` text block to the
`content[]` of every tool response — including successful ones. If you only read `content[0]`, you
will not see it.

## The one capability that is going away

`search_shop_policies_and_faqs` exists only on the deprecated Storefront MCP and has **no UCP
replacement**. After 2026-08-31, an agent that needs RYSE's return, shipping, privacy or terms text
must fetch it as HTML from `https://www.helloryse.com/policies/{refund-policy, shipping-policy,
privacy-policy, terms-of-service}`.

## What RYSE does not expose

There is no device API. Nothing in RYSE's published contract models a SmartShade, a shade position,
a schedule or a group. Controlling hardware goes through the consumer apps, the SmartBridge, or the
Alexa / Google Home / Apple HomeKit integrations — none of which RYSE documents for developers. The
entire machine-readable surface is commerce.
