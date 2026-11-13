# Setting up your Shopify store

The Shopify side of the connection, step by step. The technical details are in the [README](./README.md).

## Checklist

1. **SKUs.** Put the Breakmark SKU from the CSV in the `Variant SKU` field of every matching variant. Staging list: [`staging/bam-2027-skus-staging.csv`](./staging/bam-2027-skus-staging.csv).
2. **Name and number.** Player jersey pages send two line-item properties named exactly `Name on back` and `Number`, filled in automatically from the player (hidden fields). Products where fans type their own name keep visible fields with the same two names.
3. **Checkout.** Card checkout only. Turn off "Buy it now" (dynamic checkout) buttons on product pages; some themes skip the name and number fields with them.
4. **Webhook.** Settings → Notifications → Webhooks → Create webhook. Event **Order creation**, format **JSON**, URL from Breakmark.
5. **Signing secret.** Copy the "Your webhooks will be signed with …" value into your Breakmark account. Never email or text it.
6. **Test.** Test payments on (Bogus Gateway), then place the test orders in [README §7](./README.md#7-testing).

## AI setup prompt

Paste this into an AI assistant (ChatGPT, Claude, or Shopify's Sidekick). Attach the SKU CSV and, if you can, your product export (Products → Export → CSV).

```text
You are helping me set up my Shopify store to send orders to Breakmark, the company that makes and ships our team jerseys. Walk me through it one step at a time. Wait for me to confirm each step before moving on, and ask me for screenshots when you're unsure what my admin or theme looks like. Don't change anything I didn't ask for.

Background:
- Breakmark receives every paid order through a Shopify "Order creation" webhook (JSON). It matches each line item to a product only by the variant SKU. It ignores my option names and labels.
- Breakmark gave me a CSV with one row per variant: Team, Product, Jersey Type, Fit, Size, SKU, Price add-on. Every variant in my store that Breakmark makes needs the matching SKU.
- Player jerseys must send the player's name and number as line-item properties named exactly "Name on back" and "Number".

Steps:

1. SKUs. Using the Breakmark CSV and my product export, match each of my variants to its Breakmark row by team, jersey type, fit and size. My labels may differ from Breakmark's (for example "Long Sleeve Hood" vs "Long Sleeve with Hood", "Women's" vs "Womens", "XS" vs "Extra Small"); match on meaning and list any variant you can't match confidently instead of guessing. Give me a product CSV I can import with "Overwrite products with matching handles" that changes only the "Variant SKU" column. Remind me to keep SKUs as text so leading zeros survive.

2. Name and number. On player jersey product pages, help me add two hidden inputs inside the product form, named properties[Name on back] and properties[Number], filled from the player's data on that product (for example a metafield). Show me exactly which theme file and where, for my theme. Products where fans type their own name and number should keep visible fields with the same two names. Then have me add one jersey to the cart and confirm both properties show on the cart line.

3. Checkout. Have me turn off "Show dynamic checkout buttons" (Buy it now) on product page templates, and confirm only card payments are enabled, with no manual or pay-later methods.

4. Webhook. Settings → Notifications → Webhooks → Create webhook: event "Order creation", format JSON, URL = the address Breakmark gave me. Then tell me where the "Your webhooks will be signed with …" value is shown. Tell me to enter it only in my Breakmark account, never in this chat, email or text. Do not ask me to paste it here.

5. Test orders. Help me turn on Shopify test payments (Bogus Gateway) and place: one of each jersey style, a Women's and a Youth size, one jersey with a name and number, and one order on a variant whose SKU I've temporarily removed. Give me a list of the order numbers to send Breakmark, then remind me to put the removed SKU back.
```

The prompt never asks for your signing secret. If an assistant asks for it, don't share it.
