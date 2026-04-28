# FilFlo External API

**Audience:** B2B customers and partner platforms integrating with FilFlo externally.

This document covers two integration surfaces:

- **Part 1 — Customer Order API:** REST endpoints for checking order status, confirming delivery, submitting GRN, downloading invoices, and listing/exporting invoice reports.
- **Part 2 — Order Hook Integration:** Bidirectional webhook system for staying in sync with order lifecycle events in real time.

---

# Part 1 — Customer Order API

## Overview

The Customer Order API lets B2B customers check order status, confirm delivery, submit GRN (Goods Receipt Note) details, and download tax invoices using an API key.

## Base URL

All endpoints below are under:

```
https://app.backend.filflo.in/api/v1
```

## Authentication

Use an API key with either header:

- `X-API-Key: flk_...`
- or `Authorization: Bearer flk_...`

Notes:

- Keep the key secret. Treat it like a password.
- Keys are either **workspace-wide** (no customer restriction) or **customer-scoped** (restricted to a specific customer's orders).
  - Workspace-wide keys can access any order in the workspace.
  - Customer-scoped keys can only access orders belonging to that customer.
- Keys may be restricted by scopes (permissions). If no scopes are set on the key, all scopes are granted.
- Revoked/expired/invalid keys return `401`.

## API Key Scopes

Each API key has one or more scopes (permissions). If your key does not have the required scope, the API returns `403`.

Required scopes per endpoint:

| Scope | Endpoint |
|---|---|
| `orders.status.read` | `GET /external/orders/:orderId/status` |
| `orders.confirm_delivery` | `POST /external/orders/:orderId/confirm-delivery` |
| `orders.grn.write` | `POST /external/orders/:orderId/grn` |
| `orders.invoice.read` | `GET /external/orders/:orderId/invoice` |
| `reports.invoices.read` | `GET /external/reports/invoices`, `GET /external/reports/invoices/export/header`, `GET /external/reports/invoices/export/line` |

## Content Type

For `POST` endpoints:

- `Content-Type: application/json`

## Rate Limiting

All `/external/*` endpoints are rate limited **per API key**:

- Limit: **100 requests per minute**

If you exceed the limit, the API returns `429` with a `Retry-After` header.

Rate limit headers included on responses:

- `X-RateLimit-Limit`
- `X-RateLimit-Remaining`
- `X-RateLimit-Reset` (unix timestamp in seconds)

## Endpoints

### 1) Get Order Status

`GET /external/orders/:orderId/status`

Returns the current order status and relevant tracking fields.

#### Example

```bash
curl -s \
  -H "X-API-Key: flk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  "https://app.backend.filflo.in/api/v1/external/orders/ORD-001/status"
```

#### Response (200)

```json
{
  "order": {
    "orderId": "ORD-001",
    "status": "in_transit",
    "orderReceivedDate": "2026-02-10T00:00:00.000Z",
    "expectedDeliveryDate": "2026-02-15T00:00:00.000Z",
    "trackingLink": "https://...",
    "shippingPartner": "Delhivery",
    "awbNumber": "AWB123",
    "lineItems": [
      {
        "skuCode": "WG-PEN-400",
        "quantity": 100,
        "approvedQuantity": 100,
        "fulfilledQuantity": 95,
        "grnQuantity": null
      }
    ],
    "statusHistory": [
      {
        "from": "invoiced",
        "to": "in_transit",
        "changedAt": "2026-02-12T12:34:56.000Z",
        "note": null
      }
    ]
  }
}
```

### 2) Confirm Delivery

`POST /external/orders/:orderId/confirm-delivery`

Marks an `in_transit` order as `delivered`.

Idempotency:

- If the order is already `delivered` or `grn_entered`, the API returns `200` with the current order.
- If the order is in any other state, the API returns `409`.

#### Request body

- `deliveryDate` (optional, string): Any ISO date/time string. If omitted, server uses current time.
- `proofOfDelivery` (optional, string): If omitted, server uses `confirmed_via_api`.

Example:

```json
{
  "deliveryDate": "2026-02-15T10:30:00.000Z",
  "proofOfDelivery": "received_by_store_manager"
}
```

#### Example

```bash
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "X-API-Key: flk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  -d '{"deliveryDate":"2026-02-15T10:30:00.000Z","proofOfDelivery":"received_by_store_manager"}' \
  "https://app.backend.filflo.in/api/v1/external/orders/ORD-001/confirm-delivery"
```

#### Response (200)

```json
{
  "order": {
    "orderId": "ORD-001",
    "status": "delivered",
    "orderReceivedDate": "2026-02-10T00:00:00.000Z",
    "expectedDeliveryDate": "2026-02-15T00:00:00.000Z",
    "trackingLink": "https://...",
    "shippingPartner": "Delhivery",
    "awbNumber": "AWB123",
    "lineItems": [
      { "skuCode": "WG-PEN-400", "quantity": 100, "approvedQuantity": 100, "fulfilledQuantity": 95, "grnQuantity": null }
    ],
    "statusHistory": []
  }
}
```

### 3) Download Invoice PDF

`GET /external/orders/:orderId/invoice`

Downloads the tax invoice PDF for an order. Only available for orders that have been invoiced (have a generated invoice).

Requires scope: `orders.invoice.read`

#### Example

```bash
curl -s \
  -H "X-API-Key: flk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  --output invoice.pdf \
  "https://app.backend.filflo.in/api/v1/external/orders/ORD-001/invoice"
```

#### Response (200)

Binary PDF file with headers:

```
Content-Type: application/pdf
Content-Disposition: attachment; filename="invoice-ORD-001.pdf"
```

#### Errors

- `404` — Order not found (or not accessible for this API key)
- `409` — Order has not been invoiced yet

### Invoice Lifecycle

Invoicing is handled **internally by FilFlo** after an order is picked — external partners do not trigger invoicing via API.

Typical flow:

1. Your system marks the order as picked (via inbound webhook — see Part 2)
2. FilFlo generates the tax invoice (including IRN/e-invoice where applicable)
3. If an outbound webhook is configured for the `picked → invoiced` transition, FilFlo notifies your endpoint
4. You download the PDF via `GET /external/orders/:orderId/invoice`

**Best practice:** Subscribe to the outbound webhook for the `picked → invoiced` event, then fetch the PDF when notified. If you poll instead, the endpoint returns `409` until the invoice is ready.

### 4) Submit GRN

`POST /external/orders/:orderId/grn`

Submits GRN quantities and transitions a `delivered` order to `grn_entered`.

Idempotency:

- If the order is already `grn_entered` and the submitted quantities match the stored GRN quantities (falling back to `fulfilledQuantity`, `approvedQuantity`, or `quantity` if `grnQuantity` is not set), the API returns `200`.
- If the order is already `grn_entered` but quantities do not match, the API returns `409`.
- If the order is in any other state, the API returns `409`.

#### Request body

- `lineItems` (required): Array of `{ skuCode, quantity }`

Example:

```json
{
  "lineItems": [
    { "skuCode": "WG-PEN-400", "quantity": 95 }
  ]
}
```

Notes:

- `skuCode` must belong to the order. Unknown `skuCode` returns `400`.
- Duplicate `skuCode` entries in the same request return `400`.
- `quantity` must be a non-negative number.
- The request must include quantities for **all** line items on the order. Partial submissions return `400`.

#### Example

```bash
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "X-API-Key: flk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  -d '{"lineItems":[{"skuCode":"WG-PEN-400","quantity":95}]}' \
  "https://app.backend.filflo.in/api/v1/external/orders/ORD-001/grn"
```

#### Response (200)

```json
{
  "order": {
    "orderId": "ORD-001",
    "status": "grn_entered",
    "orderReceivedDate": "2026-02-10T00:00:00.000Z",
    "expectedDeliveryDate": "2026-02-15T00:00:00.000Z",
    "trackingLink": "https://...",
    "shippingPartner": "Delhivery",
    "awbNumber": "AWB123",
    "lineItems": [
      { "skuCode": "WG-PEN-400", "quantity": 100, "approvedQuantity": 100, "fulfilledQuantity": 95, "grnQuantity": 95 }
    ],
    "statusHistory": []
  }
}
```

### 5) List Invoices

`GET /external/reports/invoices`

Returns a JSON list of invoices for the workspace, with summary totals. Customer-scoped API keys see only their own customer's invoices.

Requires scope: `reports.invoices.read`

#### Query parameters

| Param | Required | Description |
|---|---|---|
| `startDate` | optional | ISO 8601 date. Filter by `invoiceDate >= startDate`. |
| `endDate` | optional | ISO 8601 date. Filter by `invoiceDate < endDate + 1 day`. |
| `customerId` | optional (workspace keys only) | MongoDB ObjectId of a customer. Ignored on customer-scoped keys (always clamped to the key's customer). |
| `status` | optional | Order status — one of `open`, `approved`, `picked`, `invoiced`, `in_transit`, `delivered`, `grn_entered`, `rto`. |
| `irnStatus` | optional | One of `active`, `cancelled`, `failed`. |

The response is capped at the **2,000 most recent invoices** (sorted by `invoiceDate` desc). Use the export endpoints below for larger or full-fidelity datasets.

#### Example

```bash
curl -s \
  -H "X-API-Key: flk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  "https://app.backend.filflo.in/api/v1/external/reports/invoices?startDate=2026-04-01&endDate=2026-04-30"
```

#### Response (200)

```json
{
  "items": [
    {
      "_id": "65...",
      "orderId": "ORD-001",
      "status": "invoiced",
      "invoiceId": "INV-2026-0001",
      "invoiceDate": "2026-04-12T00:00:00.000Z",
      "orderReceivedDate": "2026-04-10T00:00:00.000Z",
      "customerID": { "_id": "64...", "name": "Acme Foods", "location_code": "MUM-01" },
      "irnDetails": { "irn": "...", "ackNo": "...", "ackDt": "...", "status": "active" },
      "irnStatus": "active",
      "gstAmount": 1800.0,
      "ewbNo": "EWB-...",
      "invoiceValue": 11800.0
    }
  ],
  "summary": {
    "totalInvoices": 1,
    "totalInvoiceValue": 11800.0,
    "averageInvoiceValue": 11800.0,
    "totalGSTAmount": 1800.0
  }
}
```

### 6) Export Invoices — Header CSV

`GET /external/reports/invoices/export/header`

Streams a CSV with **one row per invoice** including warehouse, dates, IRN/e-way bill numbers, customer + bill-from addresses, totals, and ordered-vs-fulfilled-vs-GRN reconciliation columns.

Requires scope: `reports.invoices.read`

#### Query parameters

`startDate` and `endDate` are **required** (ISO 8601). The span between them must not exceed **90 days**.

`customerId`, `status`, `irnStatus` accept the same values as `GET /external/reports/invoices`.

#### Example

```bash
curl -s \
  -H "X-API-Key: flk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  --output invoices-header.csv \
  "https://app.backend.filflo.in/api/v1/external/reports/invoices/export/header?startDate=2026-04-01&endDate=2026-04-30"
```

#### Response (200)

```
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="invoices-header-<timestamp>.csv"
```

Columns (in order):

`Warehouse, PO Date, Appointment Date, PO Expiry Date, Punch Date, Invoice Date, Dispatch Date, Delivery Date, Voucher Type, PO Number/Order No, Order Status, Order Remarks, Channel, Invoice Number, IRN Number, IRN Date, Acknowledgement Number, Acknowledgement Date, Eway Bill Number, Total Invoice Taxable Value, Total Invoice Amount With Tax, Customer, Customer GST, Shipping Address, Billing Address, State, City, Pincode, Mode of Transport, Carrier, Tracking Number, Tracking Link, Box Count, Bill From, Bill From GST, Bill From Address, Ship From, Distinct SKU - Ordered, Qty Ordered, Distinct SKU - Approved, Qty Approved, Distinct SKU - Fulfilled/Dispatched, Qty Fulfilled / Dispatched, Distinct SKU - GRN, Qty GRN, Short Fulfilled Quantity (SKU), Short Fulfilled Quantity (Units), Short GRN Quantity (SKU), Short GRN Quantity (Units), Ordered vs Approved % (SKU), Ordered vs Approved % (Qty), Approved vs Fulfilled % (SKU), Approved vs Fulfilled % (Qty), Ordered vs Fulfilled % (SKU), Ordered vs Fulfilled % (Qty), Fulfilled vs GRN % (SKU), Fulfilled vs GRN % (Qty), Ordered vs GRN % (SKU), Ordered vs GRN % (Qty), Approval Remarks, WH/Fulfillment Remarks, GRN Remarks`

#### Errors

- `400` — `startDate`/`endDate` missing, malformed, or span exceeds 90 days
- `403` — API key missing the `reports.invoices.read` scope

### 7) Export Invoices — Line CSV

`GET /external/reports/invoices/export/line`

Streams a CSV with **one row per SKU per invoice**, including per-line GST breakdowns (taxable amount, IGST/CGST/SGST), MRP, selling price, and margin.

Requires scope: `reports.invoices.read`

#### Query parameters

Same as the header export — `startDate` and `endDate` required, span ≤ 90 days.

#### Example

```bash
curl -s \
  -H "X-API-Key: flk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  --output invoices-line.csv \
  "https://app.backend.filflo.in/api/v1/external/reports/invoices/export/line?startDate=2026-04-01&endDate=2026-04-30"
```

#### Response (200)

```
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="invoices-line-<timestamp>.csv"
```

Columns (in order):

`Warehouse, PO Date, Appointment Date, PO Expiry Date, Punch Date, Invoice Date, Dispatch Date, Delivery Date, Voucher Type, PO Number, Invoice Number, IRN Number, IRN Date, Acknowledgement Number, Acknowledgement Date, Eway Bill Number, Total Invoice Taxable Value, Total Invoice Amount With Tax, Customer, Customer GST, Shipping Address, Billing Address, State, City, Pincode, Mode of Transport, Carrier, Tracking Number, Tracking Link, Box Count, Channel, Order Status, Bill From, Bill From GST, Bill From Address, Ship From, SKU Code, SKU Name, SKU HSN Code, Order Qty, Approved Qty, Fulfilled/Dispatched Qty, GRN Qty, Approval Remarks, WH/Fulfillment Remarks, GRN Remarks, MRP, MRP (Without tax), Selling Price, Selling Price (Without tax), Margin, GST Rate, Taxable Amount, IGST Amount, SGST Amount, CGST Amount, Total Item Value`

#### Errors

- `400` — `startDate`/`endDate` missing, malformed, or span exceeds 90 days
- `403` — API key missing the `reports.invoices.read` scope

## Status Values

All possible order statuses, in lifecycle order:

| Status | Meaning |
|---|---|
| `open` | Order created, awaiting approval |
| `approved` | Order approved and confirmed |
| `picked` | Items picked from inventory |
| `invoiced` | Tax invoice generated |
| `in_transit` | Handed to carrier |
| `delivered` | Confirmed delivered to recipient |
| `grn_entered` | Goods receipt note submitted |
| `rto` | Return to origin initiated |

## Error Codes

Errors are returned as JSON with an HTTP status code.

| Code | Meaning |
|---|---|
| `400` | Bad Request: invalid/missing fields |
| `401` | Unauthorized: missing/invalid/revoked/expired API key |
| `403` | Forbidden: API key does not have required scope for this endpoint |
| `404` | Not Found: order not found (or not accessible for this API key) |
| `409` | Conflict: order status changed; retry after re-fetching status |
| `429` | Too Many Requests: rate limited (see `Retry-After` header) |

Example error response:

```json
{
  "message": "Invalid API key"
}
```

## Idempotency / Retries

- `GET` is safe to retry.
- `confirm-delivery` and `grn` are designed to be safely retryable:
  - If the state transition already happened, you will get `200` with the current order.
  - If the order is not in an eligible state, you will get `409` (call `GET status` and decide what to do next).
- If you receive `429`, wait for `Retry-After` seconds and retry.

## Security Recommendations

- Store API keys in a secret manager (not in frontend code).
- Rotate keys periodically and revoke keys that are no longer needed.
- Prefer HTTPS only. Do not send keys over plain HTTP.

---

# Part 2 — Order Hook Integration

## Overview

FilFlo's order hook system lets your platform stay in sync with order lifecycle events in real time. The integration has two directions:

| Direction | What it does |
|---|---|
| **Outbound** (FilFlo → You) | FilFlo calls your HTTP endpoint when an order transitions between statuses |
| **Inbound** (You → FilFlo) | Your platform posts status updates to FilFlo via a signed webhook |

Both directions are optional and can be enabled independently.

---

## Outbound — Receiving Order Status Events

When a configured order status transition occurs (e.g. `approved` → `picked`), FilFlo makes an HTTP `POST` to your endpoint.

### What to Implement

Expose an HTTPS endpoint that:

- Accepts `POST` requests with a JSON body
- Returns `HTTP 200` on success
- Responds within **10 seconds** (default timeout)

### Request Format

FilFlo sends a JSON body constructed from the configured request template. A typical payload looks like:

```json
{
  "orderId": "ORD-2024-00123",
  "status": "picked",
  "updatedAt": "2024-11-15T10:32:00.000Z"
}
```

The exact field names depend on what was configured during setup. Your FilFlo integration contact will confirm the payload schema for your connector.

### Authentication

FilFlo authenticates outbound calls using a **Bearer token** sent in the `Authorization` header:

```
Authorization: Bearer <your_api_token>
```

The token value is set during plugin configuration. Validate this token on every incoming request and reject calls that don't match.

### Reliability and Retries

If your endpoint is unavailable or returns a server error, FilFlo retries automatically. The default configuration allows **3 total attempts** (1 initial + 2 retries):

| Attempt | Delay after previous failure |
|---|---|
| 1st retry | 30 seconds |
| 2nd retry | 2 minutes |

After all attempts are exhausted, the event is moved to a dead-letter queue for manual review. The number of attempts and backoff delays are configurable per connector.

Retries are triggered only for transient failures: timeouts, `408 Request Timeout`, `429 Too Many Requests`, and any `5xx` response. Permanent errors (`4xx` except 408/429) are **not** retried.

**Your endpoint should be idempotent.** FilFlo may deliver the same event more than once in edge cases (e.g. network failures where your 200 response wasn't received). Use the `orderId` + `status` combination to detect duplicates.

---

## Inbound — Pushing Status Updates to FilFlo

Your platform can push order status changes back to FilFlo using a signed webhook. This is useful when status updates originate on your side (e.g. a carrier scan marks a shipment as delivered).

### Endpoint

```
POST https://app.backend.filflo.in/api/v1/integrations/webhooks/<webhookKey>
```

Your `webhookKey` is assigned during onboarding (e.g. `atlas-default`). Use it exactly as provided.

### Request Body

Send a JSON payload containing the fields agreed upon during setup. A typical shape:

```json
{
  "orderId": "ORD-2024-00123",
  "status": "shipped",
  "eventId": "evt_abc123",
  "occurredAt": "2024-11-15T09:00:00.000Z"
}
```

| Field | Required | Description |
|---|---|---|
| `orderId` | Yes | FilFlo order identifier |
| `status` | Yes | Your platform's status string (see Status Mapping below) |
| `eventId` | No | A unique ID for this event; used for deduplication. If omitted, FilFlo auto-generates one from a hash of the webhook key, order ID, status, and request body |
| `occurredAt` | No | ISO 8601 timestamp when the event occurred |

> FilFlo deduplicates by `(pluginId, eventId)`. Re-sending the same `eventId` for the same connector returns `200` with `"idempotent": true` and is safe to ignore.

### Signing Requests (HMAC-SHA256)

Every inbound request must be signed. FilFlo will reject requests with a missing or invalid signature.

**Steps:**

1. Take the raw request body bytes (before any encoding)
2. Compute `HMAC-SHA256(body, shared_secret)`
3. Hex-encode the result
4. Prefix with `sha256=`
5. Send in the signature header (e.g. `x-signature`)

```
x-signature: sha256=<hex-encoded-hmac>
```

Your `shared_secret` is provided by FilFlo during onboarding and should be stored securely (e.g. in your secrets manager). FilFlo will rotate secrets on request — see **Secret Rotation** below.

**Example (Node.js):**

```js
const crypto = require('crypto');

function signPayload(body, secret) {
    const hmac = crypto.createHmac('sha256', secret);
    hmac.update(body); // body must be the raw Buffer or string
    return 'sha256=' + hmac.digest('hex');
}

// Usage
const rawBody = JSON.stringify(payload);
const signature = signPayload(rawBody, process.env.FILFLO_WEBHOOK_SECRET);

fetch(webhookUrl, {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'x-signature': signature,
    },
    body: rawBody,
});
```

> **Important:** Compute the HMAC over the exact bytes you send. Do not re-serialize the payload after signing.

### Status Mapping

Your platform's status strings are mapped to FilFlo's internal order statuses. The mapping for a typical Atlas integration:

| Your status | FilFlo status |
|---|---|
| `packed` | `picked` |
| `shipped` | `in_transit` |
| `delivered` | `delivered` |

Statuses outside this mapping are rejected. If you need additional statuses mapped, contact your FilFlo integration contact.

FilFlo also validates that the mapped status represents a **valid transition** from the order's current state. Pushing a status that would result in an invalid transition (e.g. moving a `picked` order directly to `delivered` without `in_transit`) is rejected with a `409` response and logged.

> **Note:** If the order is already at the target status, or has already advanced past it (e.g. order is `delivered` and you send `in_transit`), FilFlo returns `200` and silently accepts the event without changing the order. This makes it safe to replay events without causing errors.

### Passing Item Details (Fulfilled Quantities)

When `lineItems` is configured in your connector's payload map (standard for production integrations), the inbound webhook payload **must** include an `items` array for every status transition. This is how you communicate fulfilled quantities — i.e. how many units of each SKU were actually packed/shipped.

#### Field Reference

| Field | Type | Required | Description |
|---|---|---|---|
| `items` | array | Yes | Array of per-SKU quantities |
| `items[].skuCode` | string | Yes | SKU code matching a line item on the order |
| `items[].quantity` | number | Yes | Fulfilled quantity (≥ 0, ≤ approved quantity) |

#### Validation Rules

- **All SKUs required:** The `items` array must include every SKU on the order. Partial submissions are rejected (`400`).
- **No duplicates:** Each `skuCode` may appear only once. Duplicate entries are rejected (`400`).
- **Quantity ceiling:** `quantity` must not exceed the order's approved quantity for that SKU. Exceeding it is rejected (`400`).
- **Empty or missing array:** If `items` is missing, null, or an empty array, the request is rejected (`400`).

#### When Fulfilled Quantities Are Applied

Fulfilled quantities are recorded during the `approved → picked` transition specifically. If the fulfilled quantity is less than the approved quantity for a SKU, FilFlo automatically records the reason as `short_supply`.

For transitions after `picked` (e.g. `picked → in_transit`), the items array is still required for validation, but quantities are not re-applied — the values from the picking step are preserved.

#### Example: Packed Webhook with Items

```json
{
  "orderId": "ORD-2024-00123",
  "status": "packed",
  "eventId": "evt_pack_456",
  "occurredAt": "2024-11-15T09:00:00.000Z",
  "items": [
    { "skuCode": "WG-PEN-400", "quantity": 95 },
    { "skuCode": "WG-PEN-200", "quantity": 50 }
  ]
}
```

#### Example curl — Inbound Webhook with Items and HMAC Signing

```bash
# 1. Build the payload
PAYLOAD='{"orderId":"ORD-2024-00123","status":"packed","eventId":"evt_pack_456","occurredAt":"2024-11-15T09:00:00.000Z","items":[{"skuCode":"WG-PEN-400","quantity":95},{"skuCode":"WG-PEN-200","quantity":50}]}'

# 2. Compute HMAC-SHA256 signature
SECRET="your_shared_secret"
SIGNATURE="sha256=$(echo -n "$PAYLOAD" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $NF}')"

# 3. Send the request
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "x-signature: $SIGNATURE" \
  -d "$PAYLOAD" \
  "https://app.backend.filflo.in/api/v1/integrations/webhooks/atlas-default"
```

### Response Format

Successful responses return JSON with `"ok": true` and additional fields:

**Normal success** (order status updated):

```json
{
  "ok": true,
  "idempotent": false,
  "receiptId": "6651a...",
  "order": { "orderId": "ORD-001", "status": "in_transit", "..." : "..." }
}
```

**Idempotent success** (duplicate `eventId`, or order already at/past target status):

```json
{
  "ok": true,
  "idempotent": true,
  "receiptId": "6651a..."
}
```

### Response Codes

| Code | Meaning |
|---|---|
| `200` | Event accepted and order status updated. Also returned for duplicate `eventId` (with `"idempotent": true`) and when the order is already at or past the target status |
| `400` | Malformed request body or missing required fields |
| `401` | Invalid or missing HMAC signature |
| `403` | Webhook exists but is currently disabled |
| `404` | `webhookKey` not recognised |
| `409` | Status transition not allowed for this order's current state |
| `5xx` | FilFlo-side error — retry with exponential back-off |

---

## Secret Rotation

If your inbound webhook secret needs to be rotated (e.g. due to suspected compromise):

1. Contact your FilFlo integration contact or use the FilFlo admin portal to trigger a rotation
2. FilFlo generates and stores the new secret
3. Update your secrets manager with the new value before the rotation deadline

Outbound bearer tokens can be updated similarly. There is no automatic expiry — rotation is on-demand.

---

## Testing Your Integration

FilFlo does not provide a sandbox environment at this time. To test end-to-end:

1. Use a tool like [webhook.site](https://webhook.site) or [ngrok](https://ngrok.com) to expose a local endpoint for outbound testing
2. Share the URL with your FilFlo integration contact to point the outbound plugin at it
3. Trigger a test order status change in the FilFlo staging workspace
4. For inbound testing, send signed requests to the production webhook endpoint (`app.backend.filflo.in`)

**Signature verification tip:** If you're getting `401` responses on inbound calls, double-check that:
- You're signing the raw body bytes, not a re-serialised version
- The `Content-Type` is `application/json` and the body is valid JSON
- The secret value matches exactly what was provided (no trailing newline, no extra whitespace)

---

## Onboarding Checklist

Share the following with your FilFlo integration contact to complete setup:

- [ ] HTTPS endpoint URL for receiving outbound order events
- [ ] Bearer token you want FilFlo to use on outbound calls
- [ ] Confirmation of the status strings your platform emits (for status mapping)
- [ ] PGP key or secure channel for receiving your inbound webhook secret

---

## Support

For integration questions, reach out to your FilFlo technical contact or email **shubham@filflo.in**.

Include the following in any support request:
- Your `webhookKey`
- Relevant `orderId` and `eventId` values
- Approximate timestamp of the event
- Full request/response details (with secrets redacted)
