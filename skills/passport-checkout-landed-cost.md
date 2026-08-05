---
name: passport-checkout-landed-cost
description: >-
  Show a shopper the true landed cost of an international order at checkout — shipping options with duty, tax and
  insurance, and product prices converted into their local currency — using Passport Global API v3.15 cart rating,
  tax-and-duty calculation, and product-price conversion.
api: Passport Global API
version: '3.15'
base_url: https://api.passportshipping.com/v3
operations:
  - POST /cart
  - POST /tax-and-duty
  - POST /product-price
generated: '2026-08-04'
method: generated
source: openapi/passport-public-api-openapi.yml
---

# Show landed cost and localized pricing at checkout

Authenticate every call with the `X-Access-Token` header (see `authentication/passport-authentication.yml`).

## Localize product prices before the cart: `POST /product-price`

Required: `base_currency`, `presentment_currency`, `country`, and `values[]` — each value is
`{ reference, value }`, where `reference` is your own product/variant handle.

The `200` returns `base_currency`, `presentment_currency`, `currency_symbol`, `currency_code`, `currency_text`,
`country`, and `converted_values[]`. Each converted value carries `reference`, `value`, `converted_value`,
`adjustment`, `adjusted_value`, `conversion_rate`, `converted_value_with_fee`, `conversion_fee`,
`converted_value_with_fee_rounded`, and `converted_value_formatted`.

Render `converted_value_formatted` — it is the display string Passport already rounded and formatted for that
market. Match rows back to your catalog on `reference`.

Unsupported currencies return `400` with `"presentment_currency EUR is not supported."` or
`"base_currency XYZ is not supported."` — fall back to your base currency rather than failing checkout.

## Rate the whole cart: `POST /cart`

Required: `address_from`, `address_to`, `items`. Optional: `reference`, `service_name`, `contents_type`,
`presentment_currency`, `parcel`.

The `200` returns `request_id`, `currency`, and `rates[]`. Each rate carries `service_name`, `service_code`,
`rate`, `tax`, `duty`, `insurance`, `formatted`, `description`, `total`, `edd` (estimated delivery date), and
`duty_tax_breakdown`.

- Present `rates[]` as the shopper's shipping choices; `total` is the landed total, `formatted` is the ready-made
  display string, and `edd` is the delivery promise.
- Keep `request_id` — it is the only correlation handle Passport returns for the quote, and it is what support
  will ask for.
- Carry the chosen `service_name` forward into `POST /ship` and into the `shipping` block of `POST /order`.

## Duty and tax alone: `POST /tax-and-duty`

When you already have a shipping rate from another source, send `shipping_rate`, `address_from`, `address_to`,
and `items` (plus optional `reference`, `presentment_currency`, `insurance`). The `200` returns `request_id`,
`currency`, and `rates[]` with `tax`, `duty`, `total`, `formatted`, and `duty_tax_breakdown`.

## Rules that bite

- **One currency per request.** All items in a single `/cart`, `/rate` or `/tax-and-duty` call must share one ISO
  4217 currency.
- **HS codes matter.** `hs_code` on each item drives the duty calculation; a missing or wrong code produces a
  wrong landed cost, not an error.
- `403 "Merchant is blocked"` is an account state, not a transient error — stop and escalate.
- `422` returns field-level errors under `details.body`; surface them, do not retry unchanged.
- Quotes carry no documented expiry. Re-rate rather than caching a landed cost across a session.
