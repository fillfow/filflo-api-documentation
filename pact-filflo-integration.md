# Pact ↔ FilFlo Integration

**Audience:** Pact ERP technical team integrating with FilFlo for Dr. Jackfruit.

This document covers the complete bidirectional integration between Pact ERP and FilFlo:

- **Part 1 — FilFlo → Pact (Outbound Sales Orders):** FilFlo pushes approved orders from Blinkit / Zepto / Instamart into Pact as Sales Orders. This flow is **live and operational**.
- **Part 2 — Pact → FilFlo (Invoice Webhook):** Pact pushes invoice events to FilFlo on invoicing. This flow is the **new integration being set up**.

---

## System Overview

```
Blinkit / Zepto / Instamart
        │
        ▼ (order received)
      FilFlo
        │  ──── approves & punches Sales Order ──▶  Pact ERP
        │                                               │
        │  ◀──── pushes Invoice event on invoicing ─────┘
        │
      Stored in FilFlo (PactWebhookLog)
```

FilFlo acts as the middleware layer. Pact is the brand ERP for Dr. Jackfruit.

---

# Part 1 — FilFlo → Pact: Sales Order Push (Live)

When an order from a quick-commerce platform (Blinkit, Zepto, Instamart) is approved on the FilFlo dashboard, FilFlo automatically creates a corresponding Sales Order in Pact via the Pact API.

**Status:** Live and operational. No action required from the Pact team for this flow.

### How it works

1. Order arrives in FilFlo from the platform
2. FilFlo operator reviews and approves the order
3. FilFlo calls the Pact API to create a Sales Order
4. The Sales Order is now visible in Pact against the relevant PO

---

# Part 2 — Pact → FilFlo: Invoice Webhook (New)

When Pact generates an invoice, it should push the invoice payload to FilFlo. FilFlo stores this verbatim, keyed by `invoice_number` and `po_number`.

This is a **push model** — Pact calls FilFlo on each invoicing event. No polling required.

## Endpoint

```
POST https://backenddo.drjackfruit.filflo.in/api/v1/pact/webhook/invoice
```

## Authentication

Use a static Bearer token in the `Authorization` header:

```
Authorization: Bearer <token>
```

The token will be shared with the Pact team directly over a secure channel. Do not hardcode it in documentation or source code.

## Headers

| Header | Value |
|--------|-------|
| `Authorization` | `Bearer <token>` |
| `Content-Type` | `application/json` |

## Request Body

Send the invoice payload as a JSON object. FilFlo is **schema-agnostic** — the full payload is stored verbatim regardless of structure. However, the following fields are used for indexing if present:

| Field | Type | Description |
|-------|------|-------------|
| `invoice_number` | string | Invoice reference (e.g. `I-TF/26-27/438`) |
| `po_number` | string | Purchase order number (e.g. `P4296604`) |

All other fields are stored as-is. Future additions to the Pact payload schema require no changes on FilFlo's side.

### Example Payload

```json
{
  "po_number": "P4296604",
  "invoice_number": "I-TF/26-27/438",
  "invoice_date": "09/05/2026",
  "delivery_date": "01-04-2026",
  "basic_price": "0",
  "landing_price": "0",
  "quantity": "string",
  "item_count": "4",
  "tax_distribution": {
    "gst_type": "CGST | SGST",
    "gst_percentage": "20",
    "gst_total": "1469.98",
    "taxable_value": "100000"
  },
  "supplier_details": {
    "gstin": "29AAGCD6653F1ZG",
    "name": "Tumkur Factory",
    "supplier_address": {
      "address_line_1": "Dr Jackfruit India Pvt. Ltd.",
      "address_line_2": "Plot 10&11, India Food Park,",
      "city": "Kiadb Kora. Tumkur,",
      "state": "29",
      "postal_code": "572138"
    }
  },
  "buyer_details": {
    "gstin": "29AAICK4821A1ZR"
  },
  "shipment_details": {
    "ewaybill_number": "0",
    "delivery_type": "",
    "delivery_partner": "-",
    "delivery_tracking_code": "string"
  },
  "items": [
    {
      "item_id": "Original Style 26G",
      "sku_code": "Original Style 26G",
      "sku_description": "Original Style 26G",
      "batch_number": "A2O26E0526",
      "quantity": "1260",
      "mrp": "20",
      "unit_discount_amount": "0",
      "unit_discount_percentage": "0",
      "unit_basic_price": "0",
      "unit_landing_price": "0",
      "expiry_date": "04/Nov/2026",
      "mfg_date": "05/May/2026",
      "upc": "23",
      "value": "1",
      "cgst_percentage": "2.5",
      "sgst_percentage": "2.5",
      "igst_percentage": "0",
      "cess_percentage": "0"
    }
  ]
}
```

## Response

### Success (200)

```json
{
  "success": true,
  "message": "Invoice event received and stored",
  "id": "<mongo_document_id>"
}
```

### Error Responses

| HTTP Code | Cause |
|-----------|-------|
| `400` | Request body is not a valid JSON object |
| `401` | `Authorization` header missing, malformed, or token is invalid |
| `500` | Server-side error (token not configured, or DB write failed) |

## Example curl

```bash
curl -X POST https://backenddo.drjackfruit.filflo.in/api/v1/pact/webhook/invoice \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "po_number": "P4296604",
    "invoice_number": "I-TF/26-27/438",
    "invoice_date": "09/05/2026",
    "items": [...]
  }'
```

## Reliability

- FilFlo stores the raw payload immediately on receipt. If you receive a `200`, the record is persisted.
- In case of a `5xx` response (transient server error), retry with a short delay.
- `4xx` responses indicate a client-side issue (auth or bad payload) — retrying without fixing the request will not help.
- The endpoint is idempotent-friendly: re-sending the same invoice will create a new log entry (FilFlo does not deduplicate at this layer). If deduplication is required in future, `invoice_number` can be used as a natural key.

---

## Setup Checklist

### FilFlo side (Himanshu / DevOps)

- [x] Endpoint implemented: `POST /api/v1/pact/webhook/invoice`
- [ ] Set `PACT_WEBHOOK_TOKEN` in `.env` on prod server (`backenddo.drjackfruit.filflo.in`)
- [ ] Merge PR #36 and deploy
- [ ] Share the token value with Pact team over a secure channel

### Pact side

- [ ] Receive the Bearer token from FilFlo
- [ ] Configure Pact to call the webhook URL on each invoicing event
- [ ] Confirm the `Content-Type: application/json` header is set
- [ ] Send a test invoice payload and confirm `200` response

---

## Support

For integration questions, contact **shubham@filflo.in** or reach out to Himanshu Rajauria on the FilFlo dev team.

Include the following in any support request:

- Relevant `invoice_number` and `po_number`
- Approximate timestamp of the call
- Full request/response details (with the Bearer token redacted)
