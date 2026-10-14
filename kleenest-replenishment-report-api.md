# Kleenest — External API

**Audience:** Kleenest teams and tools (Power BI, Google Sheets, scripts) that need FilFlo reports without downloading them by hand.

This document covers two read-only endpoints, both returning JSON or CSV:

- **Replenishment SKU-wise report:** the Replenishment page's **"Download SKU-wise"** report, with the same 97 columns and one row per order line.
- **Consolidated inventory:** current stock per casepack SKU per warehouse, as on the Stock → Consolidated Inventory page.

> This API runs on the Kleenest FilFlo deployment. Its base URL, keys and limits are separate from the general [FilFlo External API](./external-api.md).

Live docs (same content, always matches the deployed version): https://backend.kleenest.filflo.in/api/v1/external/docs

---

## Base URL

```
https://backend.kleenest.filflo.in/api/v1/external
```

## Authentication

Send your API key in either header:

- `X-API-Key: kln_live_...`
- or `Authorization: Bearer kln_live_...`

Notes:

- Keys are issued by FilFlo. A key is shown once and emailed to its recipient; it cannot be viewed again.
- Keys are **not** accepted in the query string, so they stay out of URLs and access logs.
- Keys can have an expiry date. Revoked, expired or invalid keys return `401`.
- A lost or exposed key is replaced by **rotating** it: a new key is issued and emailed, and the old one stops working immediately.
- Super admins manage keys in FilFlo under **Admin Settings → API Keys**.

## API Key Scopes

| Scope | Endpoint |
|---|---|
| `replenishment_sku_report` | `GET /external/replenishment-sku-report` |
| `consolidated_inventory` | `GET /external/consolidated-inventory` |

A key can hold several scopes. A key without the required scope gets `403`.

## Rate Limiting

Per API key:

- **60 requests per minute**
- **One request at a time.** Fetch pages one after another, not in parallel.

Exceeding either limit returns `429`. For `rate_limited`, wait the number of seconds in `Retry-After`. For `concurrent_request`, wait for your previous request to finish.

Headers on every response:

- `X-RateLimit-Limit`
- `X-RateLimit-Remaining`

## Endpoints

### 1) Get Replenishment SKU-wise Report

`GET /external/replenishment-sku-report`

Returns the SKU-wise report for orders matching the filters, paginated **by order** (every line of an order is on the same page). Orders are sorted by creation, oldest first, so pages are stable while you page through.

Requires scope: `replenishment_sku_report`

#### Query parameters

| Param | Required | Description |
|---|---|---|
| `from` | **yes** | Start date, `YYYY-MM-DD`. IST calendar day, inclusive. |
| `to` | **yes** | End date, `YYYY-MM-DD`. IST calendar day, inclusive. The range can cover at most **93 days**. |
| `date_field` | optional | Which date `from`/`to` filter on. Default `createdAt` (punch date). One of `createdAt`, `actualDispatchDate`, `invoiceDate`, `approvedDate`, `scheduledDispatchDate`, `appointmentDate`. |
| `location` | optional | Warehouse, as shown in FilFlo (lowercase). One value or comma-separated. |
| `status` | optional | Order status. One value or comma-separated. See [Status Values](#status-values). Default: all statuses. |
| `order_type` | optional | Channel, exactly as stored, e.g. `Blinkit`, `Zepto`, `Swiggy`, `Modern-Trade`, `General-Trade`. |
| `po_type` | optional | `Local` or `Outstation`. |
| `page` | optional | Page number, from 1. Default `1`. |
| `page_size` | optional | Orders per page, 1 to 100. Default `100`. |
| `format` | optional | `json` (default) or `csv`. |

#### Example — JSON

```bash
curl -s \
  -H "X-API-Key: $KLEENEST_API_KEY" \
  "https://backend.kleenest.filflo.in/api/v1/external/replenishment-sku-report?from=2026-10-01&to=2026-10-07"
```

#### Example — CSV

```bash
curl -s \
  -H "X-API-Key: $KLEENEST_API_KEY" \
  --output report.csv \
  "https://backend.kleenest.filflo.in/api/v1/external/replenishment-sku-report?from=2026-10-01&to=2026-10-07&date_field=invoiceDate&format=csv"
```

#### Response (200) — JSON

```json
{
  "success": true,
  "report": "replenishment_sku_wise",
  "generated_at": "2026-10-13T09:30:00.000Z",
  "filters": {
    "from": "2026-10-01",
    "to": "2026-10-07",
    "date_field": "createdAt",
    "location": null,
    "status": null,
    "order_type": null,
    "po_type": null
  },
  "pagination": {
    "page": 1,
    "page_size": 100,
    "total_orders": 240,
    "total_pages": 3,
    "has_more": true,
    "next_page": 2
  },
  "columns": ["PO Number", "Invoice Number", "..."],
  "row_count": 1180,
  "rows": [
    {
      "PO Number": "4100123456",
      "Invoice Number": "IN-HR-26/27-0412",
      "WH": "Gurgaon Warehouse",
      "Punch Date": "02-Oct-2026",
      "Punch Time": "11:42:10",
      "SKU Code": "KLN-FC-500",
      "Order Qty": 20,
      "Approved Qty": 18,
      "Fulfilled/Dispatched Qty": 18,
      "Order Vs Fulfilled %": "90.00%",
      "...": "..."
    }
  ]
}
```

- `rows` is keyed by column name, in the order of `columns`. Empty cells are `null` or an empty string.
- `row_count` is the number of rows on this page (order lines), not orders.
- Keep requesting `next_page` until it is `null` / `has_more` is `false`.

#### Response (200) — CSV

```
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="replenishment-sku-report_<from>_<to>_p<page>.csv"
X-Page: 1
X-Page-Size: 100
X-Total-Orders: 240
X-Total-Pages: 3
X-Has-More: true
```

The body is the same CSV as the UI download: a header row of the 97 column names, then one row per order line, every cell quoted. Pagination is in the `X-*` headers above.

#### Paging through a full range — Python

```python
import os, requests

URL = "https://backend.kleenest.filflo.in/api/v1/external/replenishment-sku-report"
HEADERS = {"X-API-Key": os.environ["KLEENEST_API_KEY"]}

def fetch_report(date_from, date_to, **filters):
    rows, page = [], 1
    while page:
        params = {"from": date_from, "to": date_to, "page": page, "page_size": 100, **filters}
        res = requests.get(URL, headers=HEADERS, params=params, timeout=120)
        res.raise_for_status()
        body = res.json()
        rows.extend(body["rows"])
        page = body["pagination"]["next_page"]
    return rows

rows = fetch_report("2026-10-01", "2026-10-07", date_field="invoiceDate")
print(len(rows), "rows")
```

#### Paging through a full range — JavaScript

```js
const URL = "https://backend.kleenest.filflo.in/api/v1/external/replenishment-sku-report";

async function fetchReport(from, to, filters = {}) {
  const rows = [];
  let page = 1;
  while (page) {
    const params = new URLSearchParams({ from, to, page, page_size: 100, ...filters });
    const res = await fetch(`${URL}?${params}`, {
      headers: { "X-API-Key": process.env.KLEENEST_API_KEY },
    });
    if (!res.ok) throw new Error((await res.json()).error.message);
    const body = await res.json();
    rows.push(...body.rows);
    page = body.pagination.next_page;
  }
  return rows;
}
```

For periods longer than 93 days, split them into several ranges and call each one in turn.

The replenishment report's 97 columns are listed in the [Column Dictionary](#column-dictionary) at the end of this document.

### 2) Get Consolidated Inventory

`GET /external/consolidated-inventory`

Returns a current snapshot of stock for every active casepack SKU, with **one row per SKU per warehouse**. There are no date filters and no pages: every call returns the full snapshot for the filters you pass. Rows are sorted by SKU code, then warehouse.

Requires scope: `consolidated_inventory`

#### Query parameters

| Param | Required | Description |
|---|---|---|
| `location` | optional | Warehouse, as shown in FilFlo (case-insensitive). One value or comma-separated. Default: all warehouses. An unknown value returns `400` listing the valid ones. |
| `sku` | optional | Casepack SKU code. One value or comma-separated. Default: all active casepack SKUs. |
| `include_pending_inward` | optional | `true` to add the `pending_inward` column. Default `false`. |
| `include_pallets` | optional | `true` to add the `received_not_racked` and `racked` columns. Default `false`. |
| `format` | optional | `json` (default) or `csv`. |

#### Columns

Quantities are in cases. Optional columns are left out completely unless you ask for them.

| Column | Included | Meaning |
|---|---|---|
| `sku_code` | always | Casepack SKU code. |
| `product_name` | always | SKU name. |
| `location` | always | Warehouse. |
| `current` | always | Current B2B stock. |
| `allocated` | always | Stock allocated to approved orders that are not yet picked. |
| `pending_inward` | `include_pending_inward=true` | Pallet-tracked warehouses only: stock received but not yet scanned in. `null` for other warehouses. |
| `received_not_racked` | `include_pallets=true` | Pallet-tracked warehouses only: stock on pallets received but not yet racked. `null` for other warehouses. |
| `racked` | `include_pallets=true` | Pallet-tracked warehouses only: stock on racked pallets. `null` for other warehouses. |

#### Example — JSON

```bash
curl -s \
  -H "X-API-Key: $KLEENEST_API_KEY" \
  "https://backend.kleenest.filflo.in/api/v1/external/consolidated-inventory"
```

#### Example — CSV with all columns

```bash
curl -s \
  -H "X-API-Key: $KLEENEST_API_KEY" \
  --output inventory.csv \
  "https://backend.kleenest.filflo.in/api/v1/external/consolidated-inventory?include_pending_inward=true&include_pallets=true&format=csv"
```

#### Response (200) — JSON

```json
{
  "success": true,
  "report": "consolidated_inventory",
  "generated_at": "2026-10-14T09:30:00.000Z",
  "filters": {
    "location": null,
    "sku": null,
    "include_pending_inward": false,
    "include_pallets": false
  },
  "columns": ["sku_code", "product_name", "location", "current", "allocated"],
  "row_count": 1208,
  "rows": [
    {
      "sku_code": "CP12-KLN-FC-500",
      "product_name": "Floor Cleaner 500ml",
      "location": "<warehouse>",
      "current": 120,
      "allocated": 30
    }
  ]
}
```

- `columns` lists exactly the columns in each row, in order.
- `filters` echoes what was applied. `location` and `sku` are `null` when not filtered.

#### Response (200) — CSV

```
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="consolidated-inventory_<YYYY-MM-DD>.csv"
```

A header row with the column names, then one row per SKU per warehouse.

## Status Values

Accepted by the replenishment report's `status` filter (comma-separate to combine):

| Status | Meaning |
|---|---|
| `open` | Pending approval |
| `approved` | Approved |
| `picked` | Picked (also matches invoiced) |
| `pending_invoicing_approval` | Awaiting invoicing approval |
| `invoiced` | Invoiced |
| `in_transit` | Dispatched |
| `delivered` | Delivered |
| `grn_entered` | GRN entered |
| `rto` | RTO (initiated, received or closed) |
| `cancelled` | Cancelled |
| `on_hold` | On hold |

## Error Codes

Errors are returned as JSON:

```json
{
  "success": false,
  "error": {
    "code": "date_range_too_large",
    "message": "The date range can cover at most 93 days. Split longer periods into several requests."
  }
}
```

| HTTP | `error.code` | What to do |
|---|---|---|
| `400` | `invalid_date_range`, `date_range_too_large`, `invalid_date_field`, `invalid_format`, `invalid_page`, `invalid_page_size`, `invalid_location`, `invalid_include_pending_inward`, `invalid_include_pallets` | Fix the query parameters. |
| `401` | `missing_api_key`, `invalid_api_key`, `expired_api_key` | Check the key, or ask FilFlo for a new one. |
| `403` | `insufficient_scope` | The key is not allowed to use this endpoint. |
| `429` | `rate_limited`, `concurrent_request` | Wait `Retry-After` seconds, or for your previous request to finish. |
| `500` | `internal_error` | Retry with backoff. If it persists, contact FilFlo. |

## Retries

- Both endpoints are read-only and safe to retry.
- On `429` or `500`, wait and retry the same request. Do not retry in parallel.

## Security Recommendations

- Store the key in a secret manager or your tool's secure settings — not in code, shared sheets or chat groups.
- Use a separate key per team or tool, so one can be revoked without affecting the others.
- If a key is exposed, ask FilFlo to rotate it.

## Column Dictionary

Replenishment SKU-wise report columns. They appear in this order in both JSON (`columns`) and CSV (header row). Quantities without "(in Units)" are in cases. Dates are `DD-Mon-YYYY` and times `HH:mm:ss`, in IST.

| # | Column | Meaning |
|---|---|---|
| 1 | PO Number | Customer PO number (FilFlo order ID). |
| 2 | Invoice Number | Tax invoice / delivery note number. Blank until invoiced. |
| 3 | LR Number | Lorry receipt / AWB number. |
| 4 | GRN Qty | Quantity received by the customer, as recorded at GRN. |
| 5 | Invoice Qty | Invoiced quantity, in cases. |
| 6 | Box Count | Boxes outwarded for this SKU on this order. |
| 7 | Weight | Total weight (kg) of the boxes outwarded for this SKU. |
| 8 | WH | Fulfilment warehouse. |
| 9 | Supplier GSTIN | Seller GSTIN on the e-invoice. |
| 10 | Supplier Name | Seller legal name on the e-invoice. |
| 11 | Bill from Address | Seller address on the e-invoice. |
| 12 | Dispatch from Address | Dispatch address on the e-invoice (same as the seller address today). |
| 13 | PO Date | PO date as received from the customer. |
| 14 | PO Time | PO time as received from the customer. |
| 15 | PO Expiry Date | PO expiry date. |
| 16 | PO Expiry Time | PO expiry time. |
| 17 | Punch Date | Date the order was created in FilFlo. |
| 18 | Punch Time | Time the order was created in FilFlo. |
| 19 | Approved Date | Date the order was approved. |
| 20 | Approved Time | Time the order was approved. |
| 21 | Picked Date | Date the order was picked. |
| 22 | Picked Time | Time the order was picked. |
| 23 | Scheduled Dispatch Date | Planned dispatch date. |
| 24 | Scheduled Dispatch Time | Planned dispatch time. |
| 25 | Appointment Date | Delivery appointment date at the customer. |
| 26 | Appointment Time | Delivery appointment time at the customer. |
| 27 | Appointment ID | Customer appointment reference. |
| 28 | Invoice date\* | Invoice date. |
| 29 | Invoice time\* | Invoice time. |
| 30 | Dispatch Date | Actual dispatch date. |
| 31 | Dispatch Time | Actual dispatch time. |
| 32 | Delivery Date | Delivery date. |
| 33 | Delivery Time | Delivery time. |
| 34 | Voucher Type | "Tax Invoice" or "Delivery Note". |
| 35 | IRN Number | E-invoice IRN, or "N/A". |
| 36 | IRN Date | IRN date (reported as the invoice date). |
| 37 | IRN Time | IRN time (reported as the invoice time). |
| 38 | Acknowledgement Number | E-invoice acknowledgement number, or "N/A". |
| 39 | Acknowledgement Date | Acknowledgement date (reported as the invoice date). |
| 40 | Acknowledgement Time | Acknowledgement time (reported as the invoice time). |
| 41 | Total Invoice Taxable Value | Invoice taxable value (all lines), or "N/A". |
| 42 | Total Invoice Amount With Tax | Invoice total including tax (all lines), or "N/A". |
| 43 | Customer | Customer (bill-to party) name. |
| 44 | Customer GST | Customer GSTIN. |
| 45 | Billing Address | Customer billing address. |
| 46 | Bill To State | Billing state. |
| 47 | Bill To City | Billing city. |
| 48 | Bill To Pincode | Billing pincode. |
| 49 | Ship To Partyname | Ship-to party name. |
| 50 | Shipping Address | Ship-to address or delivery location. |
| 51 | Ship To GSTIN | Ship-to GSTIN. |
| 52 | Ship To State | Ship-to state. |
| 53 | Ship To Pincode | Ship-to pincode. |
| 54 | Mode of Transport | Mode of transport. |
| 55 | Eway Bill Number | E-way bill number. |
| 56 | Order ID | FilFlo order ID (same as PO Number). |
| 57 | Order Type | Channel, e.g. Blinkit, Zepto, Swiggy, Modern-Trade, General-Trade. |
| 58 | PO Type | "Local" or "Outstation". |
| 59 | Order Status | Order status, e.g. open, approved, picked, invoiced, in_transit, delivered, grn_entered, cancelled. |
| 60 | Invoice Status | "Invoiced", "Not Invoiced" or "Invoice Cancelled". |
| 61 | SKU Code | SKU code. |
| 62 | SKU Name | SKU name. |
| 63 | Order Qty | Ordered quantity, in cases. |
| 64 | Approved Qty | Approved quantity, in cases. |
| 65 | Fulfilled/Dispatched Qty | Fulfilled / dispatched quantity, in cases. |
| 66 | Order Qty (in Units) | Ordered quantity x case size. |
| 67 | Approved Qty (in Units) | Approved quantity x case size. |
| 68 | Fulfilled/Dispatched Qty (in Units) | Fulfilled quantity x case size. |
| 69 | GRN Qty (in Units) | Same value as GRN Qty (not multiplied by case size). |
| 70 | Approval Remarks | Why the approved quantity differs from the order. |
| 71 | WH/Fulfillment Remarks | Why the fulfilled quantity differs from the approved quantity. |
| 72 | GRN Remarks | Why the GRN quantity differs from the fulfilled quantity. |
| 73 | Reason for Shortfall | Order-level shortfall reason. |
| 74 | MRP | MRP per unit (from the e-invoice). |
| 75 | MRP (Without tax) | MRP excluding GST. |
| 76 | Selling Price | Selling price per unit including GST. |
| 77 | Selling Price (Without tax) | Selling price per unit excluding GST. |
| 78 | Margin | Customer margin on MRP excluding tax, as a percent. |
| 79 | GST Rate | GST rate, as a percent. |
| 80 | Taxable Amount | Line taxable amount. |
| 81 | IGST Amount | Line IGST. |
| 82 | SGST Amount | Line SGST. |
| 83 | CGST Amount | Line CGST. |
| 84 | Total Item Value | Line total including tax. |
| 85 | Order Vs Fulfilled % | Fulfilled / ordered, as a percent ("N/A" when ordered is 0). |
| 86 | Approved Vs Fulfilled % | Fulfilled / approved, as a percent ("N/A" when approved is 0). |
| 87 | Dispatch Status | "True" once the order is in transit, delivered or GRN entered. |
| 88 | Delivery Status | "True" once the order is delivered or GRN entered. |
| 89 | Carrier/Shipping Partner | Carrier / transporter. |
| 90 | Has Proof of Delivery | "True" if a proof of delivery is attached. |
| 91 | Proof of Delivery | Proof of delivery file URL. |
| 92 | Has PO Proof | "True" if the PO copy is attached. |
| 93 | PO Proof | PO copy file URL. |
| 94 | Has Proof of GRN | "True" if a GRN copy is attached. |
| 95 | Proof of GRN | GRN copy file URL. |
| 96 | Has Proof of Dispatch | "True" if a proof of dispatch is attached. |
| 97 | Proof of Dispatch | Proof of dispatch file URL. |
