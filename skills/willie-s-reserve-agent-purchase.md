---
name: willie-s-reserve-agent-purchase
description: Search the Willie's Reserve catalog and take a buyer from intent to a human-approved checkout over the store's live UCP/MCP endpoint.
generated: '2026-09-04'
method: generated
source: mcp/willie-s-reserve-mcp-tools-list.json (live tools/list, 2026-09-04) + https://williesreserve.com/agents.md
api: Willie's Reserve Commerce Agent API (UCP/MCP)
endpoint: https://williesreserve.com/api/ucp/mcp
transport: JSON-RPC 2.0 over HTTPS POST (MCP)
operations:
  - search_catalog
  - lookup_catalog
  - get_product
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

# Buying from Willie's Reserve as an agent

Willie's Reserve serves a live Model Context Protocol endpoint at
`https://williesreserve.com/api/ucp/mcp` implementing the Universal Commerce Protocol (UCP)
shopping service, version `2026-08-25`. It needs no API key: a `tools/list` call with no
credentials returns all thirteen tools.

Every tool name below was read from that live manifest. Do not invent tools; call `tools/list`
again if you need the current set.

## Before you start

1. `GET https://williesreserve.com/.well-known/ucp` to confirm the protocol version and which
   capabilities and payment handlers the store supports.
2. Every tool call requires a `meta` object carrying `meta.ucp-agent.profile` — a URI identifying
   your agent profile. Calls without it are rejected by schema.
3. Send `context.address_country` and `context.currency` so prices and availability come back
   correct for your buyer.

## The flow

1. **Find** — `search_catalog` with a natural-language `query`, filters, or both (at least one is
   required). Results are paginated; follow `pagination.cursor` for more. Use `lookup_catalog` when
   you already hold identifiers, and `get_product` for full detail on one product.
2. **Cart** — `create_cart` with `line_items[]`, each `{quantity, item: {id: <variant id>}}`.
   Read it back with `get_cart`, amend with `update_cart`, abandon with `cancel_cart`.
3. **Checkout** — `create_checkout`, either with `line_items` directly or with `cart_id`. When you
   pass `cart_id`, the cart's contents win over overlapping fields in the checkout payload.
4. **Fulfil** — `update_checkout` to set `fulfillment.methods[]` with a destination address and a
   selected shipping option, and to attach `payment.instruments[]`.
5. **Discounts** — only apply `discounts.codes[]` if the buyer says they have a code. Codes are
   case-insensitive and each submission replaces the previous set; send an empty array to clear.
6. **Complete** — `complete_checkout`. Then read the result with `get_order`.

## Rules the store publishes, which you must honor

- **A human approves payment.** `robots.txt` and `llms.txt` both state that checkout, payment and
  order placement must not be completed automatically. Do not call `complete_checkout` without an
  explicit, contemporaneous approval from the buyer at the moment of payment. If you cannot get
  one, the store directs you to route the purchase through the Shop skill
  (`https://shop.app/SKILL.md`) instead.
- **Cannabis products.** This is a regulated consumer product; availability is jurisdictional and
  the buyer's own eligibility is theirs to establish, not yours to assume.

## Money

Prices are integers in ISO 4217 **minor** units, paired with a currency code:
`{"amount": 2500, "currency": "USD"}` is $25.00. Divide by 100 for two-decimal currencies before
quoting a buyer. Zero-decimal currencies such as JPY are already whole units.

## Retries and reversal

- **Only `complete_checkout` is replay-protected.** It accepts `meta.idempotency-key`. Set one and
  reuse it on any retry of the same completion.
- **The other six mutating tools accept no idempotency key.** A retried `create_cart` or
  `create_checkout` may produce a second object; read state back with `get_cart` / `get_checkout`
  before retrying rather than firing again blind.
- **Cancellation exists, a window does not.** `cancel_cart` and `cancel_checkout` are real tools,
  but the store publishes no window inside which cancellation still works, and there is **no
  refund, void or return tool at all**. Once `complete_checkout` succeeds, the agent-facing surface
  is read-only: reversal is a human support path. Treat completion as the point of no return.

## Rate limits

The endpoint is rate limited per IP. No numeric limit is published and no `RateLimit-*` or
`Retry-After` header comes back on success — you will only learn the limit by receiving a `429`.
Back off exponentially when you do.

## Errors

Errors arrive as JSON-RPC 2.0 `error` objects, not `application/problem+json`. The store publishes
no error-code reference, so surface the raw JSON-RPC error message to the buyer rather than
mapping it to a code you invented.
