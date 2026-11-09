# QMe API quickstart

Create a quote in QMe from your own system in about ten minutes.

The full reference is [`openapi.yaml`](./openapi.yaml). Examples below use the staging address; production is announced at launch.

```bash
export QME=https://bjodkoppwjyoyabolchf.supabase.co/functions/v1/v1
```

## 1. Make a key

In QMe, open **API keys** in the sidebar (admins only) and choose **Create key**.

| Permission | What it lets the key do | Default |
|---|---|---|
| Read quotes | `GET /quotes/{id}` | on |
| Create and update draft quotes | `POST /quotes` | on |
| Send quotes to customers | `POST /quotes/{id}/send` | **off** |
| Update the catalogue | `POST /catalogue/import` (admin keys only) | off |

The key is shown **once**. Copy it and keep it safe; QMe stores only a fingerprint. A key acts as the person who made it, so quotes it creates belong to that person, and if that person is switched off the key stops working.

```bash
export QME_KEY=qme_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## 2. Preview a quote (nothing is saved)

```bash
curl -s $QME/quotes -H "X-API-Key: $QME_KEY" -H "Content-Type: application/json" -d '{
  "dry_run": true,
  "external_ref": "ERP-1001",
  "customer": { "name": "Acme Labs", "phone": "9876543210", "email": "buyer@acme.in", "state": "Tamil Nadu" },
  "lines": [
    { "sku": "PIP-10", "quantity": 10 },
    { "name": "Beaker 250ml", "quantity": 2, "discount_pct": 10 },
    { "name": "Custom fabrication", "quantity": 1, "unit_price": 5000, "gst_rate": 12 }
  ]
}'
```

You get each line with what it was matched to, the tax type (same-state CGST+SGST or other-state IGST), the totals, and any `warnings`.

## 3. Create the draft

Same request without `dry_run`. Add an `Idempotency-Key` so a retry never makes two quotes:

```bash
curl -s $QME/quotes -H "X-API-Key: $QME_KEY" -H "Idempotency-Key: ERP-1001-v1" \
  -H "Content-Type: application/json" -d @quote.json
```

Response `201`: the quote `id`, its `quote_number`, the lines, totals, and whether the buyer was saved as a new client. The quote now shows in QMe as a draft, owned by the key's creator. **Nothing has been sent to the customer.**

Send the same `external_ref` again (changed lines, say) and the **same draft is updated**, with the same number (`200`, `created: false`). Once a quote has been sent it can no longer be changed this way.

### How lines are matched to your catalogue

1. `product_id`, then `sku`, then the name.
2. A name matches only when it is very close. Spacing, capitals and units are ignored (`pipette 10 ML` = `Pipette 10ml`). A different size never matches: `Beaker 100ml` is not `Beaker 250ml`.
3. A line that cannot be placed and has no `unit_price` refuses the **whole** request (`422`, nothing saved) and lists the closest names, so you can fix it and retry. With a `unit_price` it becomes a custom line.

### Tax

Worked out like the QMe app does: your GSTIN against the buyer's state (`customer.state`, or the GSTIN's own state). If the buyer's state is unknown, same-state tax is assumed and `warnings` says so. A business that is not a regular GST taxpayer charges no tax.

## 4. Send it

```bash
curl -s $QME/quotes/$ID/send -H "X-API-Key: $QME_KEY" -H "Content-Type: application/json" \
  -d '{ "channel": "email" }'
```

* **email**: goes only to the customer email saved on the quote, in your company's name; replies reach you. 100 emails a day per key.
* **whatsapp**: QMe does not send WhatsApp messages. You get `message_text` and a `wa_link` to send yourself.
* A key made by a **member** can send only after an admin approves the quote in QMe. You get `202 pending_approval`, an admin is alerted, and you send again once it is approved.
* If the email provider fails after the quote was marked sent, you get `502 email_failed` **with** the `public_url`, so nothing is lost.

## 5. Check a quote

```bash
curl -s $QME/quotes/$ID -H "X-API-Key: $QME_KEY"
```

A sent quote shows the total that was frozen when it was sent.

## 6. Keep your catalogue in step (optional)

```bash
curl -s $QME/catalogue/import -H "X-API-Key: $QME_KEY" -H "Content-Type: application/json" -d '{
  "dry_run": true,
  "mark_missing_inactive": false,
  "products": [ { "sku": "PIP-10", "name": "Pipette 10ml", "unit_price": 120, "gst_rate": 18 } ]
}'
```

Rows are matched by SKU, then exact name; only the fields you send change; **nothing is ever deleted**. `dry_run` shows exactly what a real import would do. The import is all or nothing.

## Errors

Always `{ "error": { "code", "message", "details"? } }`; `details` lists each problem with its path (`lines[2].quantity`).

| Status | `code` | Meaning |
|---|---|---|
| 400 | `validation_error` | the request is not valid |
| 401 | `invalid_api_key` | missing, wrong or revoked key |
| 403 | `missing_scope`, `key_creator_inactive` | the key may not do this |
| 404 | `not_found` | no such quote (or not yours) |
| 409 | `conflict`, `request_in_progress` | wrong state (e.g. already sent) |
| 413 | `payload_too_large` | over 1 MB |
| 422 | `unprocessable`, `idempotency_key_reuse` | cannot be done as asked |
| 429 | `rate_limited`, `daily_email_limit` | slow down; see `Retry-After` |
| 502 / 503 | `email_failed`, `email_not_configured` | email problems |
| 500 | `internal_error` | our side; retry (a 5xx is never stored by an `Idempotency-Key`) |

Limits: 60 requests a minute per key, 1 MB per request, 200 lines per quote, 5000 products per import.

## Good habits

* Always send an `Idempotency-Key` on `POST`.
* Revoke a key you no longer use. Revoking works immediately.
* Keep keys out of browsers and public code: a key can create quotes in your account.
