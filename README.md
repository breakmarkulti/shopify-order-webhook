# Breakmark Shopify order webhook

How a partner's Shopify store sends paid orders to Breakmark for production and shipping.

**Status:** v0.2, 2026-11-13. Testing on Breakmark's staging (test) system. Production is not live yet.

**Setting up your store?** Start with [`SETUP.md`](./SETUP.md): a step-by-step checklist and a prompt you can paste into an AI assistant to walk you through the Shopify side.

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

Enter the signing secret only on the Shopify page of your Breakmark account (or a secure link Breakmark sends), never by email, chat or text. Breakmark can store it but nobody can read it back. If you think it has leaked, tell Breakmark: we disable the connection and you rotate the secret in Shopify.

## 3. Products

### SKU

- **Every variant gets its own SKU from Breakmark.** The SKU says which team product it is and which style, fit and size was picked, for example `1008-001716-040201`.
- Format: `CCCC-IIIIII-VVVVVV`: catalog product, team product (item number), then two digits per option (style, size, fit). Breakmark generates them; never build or edit one by hand.
- Your option **names and values can say anything you like**. Breakmark reads only the SKU, so it never translates your option labels.
- SKUs are per team product, not per player. Every player's jersey for a team uses that team's SKUs, so roster changes never need new SKUs.
- Breakmark sends a CSV with one row per variant (team, product, style, fit, size, SKU). Put each SKU in Shopify's `Variant SKU` field for the matching variant.
- Staging and live SKUs are different. The staging list is in [`staging/`](./staging/). Swap in the live list before going live.
- If you add a style or size, ask Breakmark for its SKU first. An unknown SKU is held, not guessed.

### Name and number

Customizations come from line-item properties:

| Property | Use |
|---|---|
| `Name on back` (or `Name`) | Printed name (exact spelling) |
| `Number` | Printed number |

On player jersey pages, fill these in automatically from the player (hidden fields in the product form) so fans don't type them. A jersey that needs a name or number and arrives without one is held for staff rather than printed blank.

## 4. Order requirements

| Field | Requirement |
|---|---|
| `financial_status` | `paid` when the order is created. Use card checkout. Orders paid later are not picked up automatically. |
| `cancelled_at` | null |
| `currency` | `USD` |
| `test` | `true` on the test connection, `false` on the live connection |
| `shipping_address` | `name`, `address1`, `city`, `zip`, `country_code` required. `province_code` required for US and Canada. `address2` and `phone` optional. |
| `email` | the buyer's email |
| `line_items[].sku` | a Breakmark variant SKU from your current list (§3) |
| `line_items[].quantity` | a positive whole number |
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
- a jersey that needs a name or number arrives without one
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

1. Use a Shopify test store if you can. Turn on test payments (Bogus Gateway); staging only accepts orders with `test: true`.
2. Load the staging SKUs from [`staging/`](./staging/) onto the matching variants.
3. Breakmark gives you the staging webhook URL. Create the webhook (§1) and enter the signing secret in your Breakmark account.
4. Send one test order. Breakmark confirms it arrived and activates the connection.
5. Place test orders: each jersey style, a Women's and a Youth size, one with a name and number, and one variant with a SKU deliberately removed (it should be held).
6. Send Breakmark the order numbers. Breakmark confirms each one.

The live connection is activated after testing passes and the live SKUs are loaded.

## Changes

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-11-03 | First draft |
| 0.2 | 2026-11-13 | One SKU per variant (option labels no longer matter); setup guide and AI prompt; staging SKU list |

Questions: support@breakmark.com
