# Breakmark Shopify order webhook

How a partner's Shopify store sends paid orders to Breakmark for production and shipping.

**Status:** draft v0.1, 2026-11-03. The endpoint is in development. Breakmark issues each store its URL when the store is connected; until then, nothing here is live.

## How it works

1. A fan buys on your Shopify store and pays you.
2. Shopify sends the order to Breakmark with an **Order creation** webhook.
3. Breakmark checks the signature, matches each line item to a Breakmark product by SKU, and creates a production order.
4. Breakmark makes the items and ships them to the address on the order.

No Shopify app or API key is involved. The connection is a standard Shopify notification webhook.

## 1. Connection

| | |
|---|---|
| Event | `orders/create` (Shopify admin: **Order creation**) |
| Format | JSON |
| URL | `https://<host>/webhook/shopify/orders/<connectionId>`, issued by Breakmark, one per store (separate URLs for test and live) |
| Method | `POST` |

Set it up in Shopify admin under **Settings → Notifications → Webhooks → Create webhook**.

The URL identifies your store's connection. It is not a password. Breakmark only accepts orders from a connection staff have activated.

## 2. Authentication

Breakmark verifies every request with Shopify's webhook signature:

- `X-Shopify-Hmac-Sha256` must be the base64 HMAC-SHA256 of the exact request body, keyed with your store's webhook signing secret. The secret is shown on the Shopify Webhooks page ("Your webhooks will be signed with …").
- `X-Shopify-Shop-Domain` must be the `.myshopify.com` domain registered for this connection.
- `X-Shopify-Topic` must be `orders/create`.

Share the signing secret with Breakmark only through the secure link Breakmark provides, never by email, chat or text. If you think it has leaked, tell Breakmark: we disable the connection and you rotate the secret in Shopify.

## 3. Products

### SKU

- Each Shopify product carries **one SKU: the Breakmark product id** (a UUID such as `00000000-0000-4000-8000-000000000001`).
- Put the same SKU on **every variant** of that product.
- Breakmark sends a spreadsheet with one row per product (`Handle`, `Title`, `SKU`). Load it with Shopify's product CSV export and import: fill `Variant SKU` on every variant row of each handle.
- SKUs are per season. Each season Breakmark sends a new list.

### Option values

Shopify sends option **values** in each line item's `variant_title` (for example `Short Sleeve / Women's M`), not option names. So:

- Option **names** are yours to choose.
- Option **values** must exactly match the values on the Breakmark product, including capitalization. Breakmark's product list states the allowed values for each product.
- Never put ` / ` (space, slash, space) inside a value. Shopify uses it to separate options.

### Size

Size is the one option Breakmark translates. Use fit, then size, in one value:

| Fit | Size |
|---|---|
| `Women's`, `Men's`, `Youth` | `XS`, `S`, `M`, `L`, `XL`, `2XL`, `3XL`, `4XL` |

Examples: `Women's M`, `Men's 2XL`, `Youth L`. Only the sizes the product offers are accepted.

### Name and number

Customizations come from line-item properties:

| Property | Use |
|---|---|
| `Name on back` (or `Name`) | Printed name |
| `Number` | Printed number |

Blank values print nothing. An option used only to set a customization price (for example "Name & Number") may be agreed with Breakmark and is then ignored. Tell Breakmark before changing it.

## 4. Order requirements

| Field | Requirement |
|---|---|
| `financial_status` | `paid` when the order is created. Use card checkout. Orders paid later are not picked up automatically. |
| `cancelled_at` | null |
| `currency` | `USD` |
| `test` | `true` on the test connection, `false` on the live connection |
| `shipping_address` | `name`, `address1`, `city`, `zip`, `country_code` required. `province_code` required for US and Canada. `address2` and `phone` optional. |
| `email` | the buyer's email |
| `line_items[].sku` | a Breakmark product id from your current list (§3) |
| `line_items[].quantity` | a positive whole number |
| `line_items[].variant_title` | option values as in §3 |
| `line_items[].properties` | name and number as in §3 |
| `id`, `name` | Shopify's order id and number (`#1001`), used to identify the order and prevent duplicates |

See [`examples/orders-create.json`](./examples/orders-create.json) for a trimmed example with only the fields Breakmark reads.

## 5. Responses

| Status | Meaning | Shopify retries? |
|---|---|---|
| `200` | Accepted. The order was created, was already received, or is **held** for staff review (see below). | No |
| `400` | Wrong topic or malformed body | No, fix the webhook |
| `401` | Signature, shop domain or secret doesn't match | No, fix the webhook |
| `404` | Unknown or inactive connection | No, ask Breakmark |
| `429`, `500`, `503` | Temporary problem on Breakmark's side | Yes |

Shopify retries a failed delivery 8 times over 4 hours. A webhook that keeps failing may be removed by Shopify. Breakmark compares your weekly order export with received orders to catch anything missed.

Sending the same order twice never creates two Breakmark orders.

### Held orders

Breakmark answers `200` but creates no order until staff fix the cause and re-run it, when:

- the order isn't paid, is cancelled, or isn't in USD
- a line item has no SKU, or the SKU isn't in your current Breakmark product list
- an option value doesn't match the product, or a value is missing
- the shipping address is incomplete
- a test order arrives on the live connection, or a live order on the test connection

Breakmark staff are alerted on every held order and contact you if they need something.

## 6. Not synced

These don't reach Breakmark automatically. Email Breakmark instead:

- cancellations and refunds
- address changes after the order is placed
- orders paid after they were created

Tracking numbers are not sent back to Shopify.

## 7. Testing

1. Breakmark activates a test connection for your test store and sends its URL.
2. You create the webhook there and share the signing secret securely.
3. Load the test SKU list, turn on Shopify test payments (Bogus Gateway), and place test orders: one per product type, one with a name and number, one Youth size, and one with a missing SKU (it should be held).
4. Send Breakmark the order numbers. Breakmark confirms each one.

The live connection is activated after testing passes and the season's SKUs are loaded.

## Changes

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-11-03 | First draft |

Questions: support@breakmark.com
