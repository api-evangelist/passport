---
name: passport-quote-and-ship
description: >-
  Price an international parcel with Passport's landed-cost rating and purchase the shipping label, using the
  Passport Global API v3.15. Covers the rate → choose service → ship → (void on failure) flow, including the
  single-currency rule, the non-idempotency of label purchase, and the error paths that are not retryable.
api: Passport Global API
version: '3.15'
base_url: https://api.passportshipping.com/v3
operations:
  - POST /rate
  - POST /ship
  - POST /void/{code}
generated: '2026-08-04'
method: generated
source: openapi/passport-public-api-openapi.yml
---

# Quote and ship an international parcel with Passport

## Before you start

- You need a Passport API key. It is issued by Passport's onboarding team, not by self-service signup. Send it on
  every request as the header `X-Access-Token`. There is no OAuth and no scopes.
- Use `https://api-stg.passportshipping.com/v3` for testing and `https://api.passportshipping.com/v3` for
  production. Keys are per-environment. **The published OpenAPI lists only the staging server**, so a generated
  client will point at staging unless you override the base URL.
- Everything is HTTPS + `application/json`.

## Step 1 — Rate the shipment: `POST /rate`

Send `address_from`, `address_to`, `parcel`, and `items` (all four are required). Optional: `reference`,
`duty_paid`, `contents_type`.

- Each item carries `description`, `hs_code`, `sku`, `quantity`, `value`, `weight`, `units_weight`, `origin`,
  and `currency`.
- **Every item must use the same currency.** Mixed currencies return `400` with
  `"Passport only accepts a single currency for items at a time. Currencies received: [...]"`.
- Values must be between `0.01` and `99999999.99`. Currency is a three-letter ISO 4217 code.

A `200` returns `{ rate, duty, tax, insurance, currency, serviceName }` — the landed cost. Keep `serviceName`;
it is the input to step 2.

## Step 2 — Buy the label: `POST /ship`

Same body as `/rate` plus a required `service_name`, set to the `serviceName` you got back from `/rate`.
`address_to` may also carry `customer_tax_id`; `label_image_format` selects the label format.

A `200` returns `{ code, rate, duty, tax, insurance, currency, label, tracking_url }`:

- `code` is the Passport tracking code (e.g. `PG3450126846CA`) — the only Passport-issued identifier in the API.
- `label` is a URL to the label image in S3.
- `tracking_url` is the branded tracking page for the shipment.

**This operation is not idempotent.** Passport documents no `Idempotency-Key` header. If the call times out you
cannot safely retry it — you may buy a second label. Treat a timeout as "unknown", surface it to a human, and if a
duplicate label was created, void it (step 3).

## Step 3 — Void a label you should not have bought: `POST /void/{code}`

Path parameter `code` is the tracking code from step 2.

- `200` — label voided.
- `400` — the shipment is not voidable (it has already moved). Not retryable.
- `404` — invalid tracking code.

## Error handling

Passport does not use RFC 9457. Errors are `{ message, details?, code? }`.

| Status | Meaning | What to do |
|---|---|---|
| 400 | Mixed currency, or brand not onboarded | Fix the payload, or contact Passport — not retryable by the agent |
| 401 | Missing/incorrect `X-Access-Token` | Stop. Do not retry with the same key |
| 404 | Bad URL, or invalid tracking code on void | Check the `/v3` prefix and the code |
| 422 | Validation — walk `details.body` and `details.items` for per-field messages | Fix and resubmit |
| 500 | Message contains `RequestId: ...` | Retry with backoff; escalate the RequestId to Passport support |

There are no documented rate limits and no `429` semantics — back off conservatively on your own initiative.

See `errors/passport-problem-types.yml` for the full catalog and
`conventions/passport-conventions.yml` for the cross-cutting rules.
