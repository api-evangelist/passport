---
name: passport-order-sync
description: >-
  Keep commercial order data in sync with Passport so customs documentation, duty/tax attribution and compliance
  reporting are correct — submit, update, retrieve and delete orders with the Passport Global API v3.15 /order
  endpoints, including the one-year update window and the orderNames query contract.
api: Passport Global API
version: '3.15'
base_url: https://api.passportshipping.com/v3
operations:
  - POST /order
  - PUT /order
  - GET /order
  - DELETE /order
generated: '2026-08-04'
method: generated
source: openapi/passport-public-api-openapi.yml
---

# Sync orders into Passport

`/order` is the only resource-shaped surface in the Passport Global API, and even it is addressed by an
`orderNames` **query parameter** rather than a path id. The identifier is yours: `order_name` is whatever your
store calls the order, capped at 100 characters.

Authenticate every call with `X-Access-Token`.

## Submit an order: `POST /order`

Required: `total_value`, `currency_code`, `order_name`, `created`, `shipping`.
Also accepted: `value`, `items[]`, `address_to`, `customer_tax_id`, `order_url`.

- `created` is a Unix timestamp (integer), not an ISO date.
- `currency_code` is a three-letter ISO 4217 code.
- `items[]` carry `id`, `name`, `sku`, `requires_shipping`, `value`, `value_discounted`, `quantity`, `hs_code`,
  `country_of_origin`, `country_of_fulfillment`, `description`, `weight`. The HS code and origin country are what
  customs actually consumes — send them.
- `shipping` carries `id`, `rate`, `duty`, `duty_source`, `tax`, `tax_source`, `service_name` (and `service_code`
  on update). Set `service_name` to the option the shopper chose from `POST /cart` or `POST /rate`.

`400` with `"Brand has not been onboarded within Passport's system"` means the account is not provisioned — this
is an onboarding problem, not a payload problem. `400 "Integration is not configured for this merchant"` is the
same class.

## Update an order: `PUT /order?orderNames=<name>`

Same body shape as the POST. `orderNames` is required on the query string.

- `403 "Denied: Returned when orders are past 1 year"` — **orders older than one year cannot be updated.** Treat
  the one-year mark as a hard write boundary in your reconciliation jobs.
- `404` — the order name was never submitted; POST it instead.

## Read orders: `GET /order?orderNames=<name>[,<name>]`

`orderNames` is required and must be a non-empty string with at least one value — an empty or missing parameter
returns `400`. There is **no pagination and no list-all**: you can only read orders you can name.

The `200` returns an array of order documents, which include fields the request shape does not expose —
`integration_id`, `presentment_currency`, `value_discounted_presentment_currency`.

## Delete orders: `DELETE /order?orderNames=<name>`

Returns `{ "deleted": ["<order_name>", ...] }`. Destructive and not undoable through the API — confirm with a
human before an agent calls it.

## Failure modes to program against

| Status | Cause | Action |
|---|---|---|
| 400 | Bad order-name format, brand not onboarded, integration not configured | Escalate to Passport; do not retry |
| 401 | Missing/invalid `X-Access-Token` | Stop |
| 403 | Brand blocked, or order past one year | Account/window state — stop |
| 404 | Order not found | POST it, or correct the name |
| 422 | Field validation — `details.body` / `details.items` | Fix and resubmit |
| 500 | `message` contains `RequestId` | Retry with backoff, escalate the RequestId |

There is no idempotency key. A retried `POST /order` for the same `order_name` has no documented dedupe
behavior — read first with `GET /order?orderNames=<name>` before re-submitting.
