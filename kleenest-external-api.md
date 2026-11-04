# Kleenest — External API

**Audience:** Kleenest teams and tools (Power BI, Google Sheets, scripts) that need FilFlo reports without downloading them by hand.

This document covers four read-only endpoints, all returning JSON or CSV:

- **Replenishment SKU-wise report:** the Replenishment page's **"Download SKU-wise"** report, with the same 97 columns and one row per order line.
- **Replenishment Order-wise report:** the Replenishment page's **"Download Order-wise"** report, with 99 columns and one row per order.
- **Consolidated inventory:** current stock per casepack SKU per warehouse, as on the Stock → Consolidated Inventory page.
- **RTO report:** every Return Inward with its crates and the Inventory POs raised from each crate (one row per Return Inward x crate x PO), or, with `view=pendancy`, the RTO Pendancy page's data (returned goods not back in stock yet).

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
| `replenishment_sku_report` | `GET /external/replenishment-sku-report`, `GET /external/replenishment-order-report` |
| `consolidated_inventory` | `GET /external/consolidated-inventory` |
| `rto_report` | `GET /external/rto-report` |

A key can hold several scopes. A key without the required scope gets `403`. Any key that can read the SKU-wise report can also read the Order-wise report. New keys get every scope; keys issued before the RTO report was added get `rto_report` from FilFlo on request.

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

### 2) Get Replenishment Order-wise Report

`GET /external/replenishment-order-report`

Returns the Order-wise report for orders matching the filters, with **one row per order**. Quantities are totals over the order's SKUs, along with SKU counts, shortfall and fill-rate percentages, box count and total weight. Orders are sorted by creation, oldest first, so pages are stable while you page through.

Requires scope: `replenishment_sku_report` (the same scope as the SKU-wise report).

#### Query parameters

The same as the [SKU-wise report](#1-get-replenishment-sku-wise-report): `from`, `to` (at most **93 days**), `date_field`, `location`, `status`, `order_type`, `po_type`, `page`, `page_size` (orders per page, 1 to 100) and `format`. The `status` filter takes the same [Status Values](#status-values).

#### Example — JSON

```bash
curl -s \
  -H "X-API-Key: $KLEENEST_API_KEY" \
  "https://backend.kleenest.filflo.in/api/v1/external/replenishment-order-report?from=2026-10-01&to=2026-10-07"
```

#### Example — CSV

```bash
curl -s \
  -H "X-API-Key: $KLEENEST_API_KEY" \
  --output orders.csv \
  "https://backend.kleenest.filflo.in/api/v1/external/replenishment-order-report?from=2026-10-01&to=2026-10-07&date_field=invoiceDate&format=csv"
```

#### Example — with filters

```bash
curl -s \
  -H "X-API-Key: $KLEENEST_API_KEY" \
  "https://backend.kleenest.filflo.in/api/v1/external/replenishment-order-report?from=2026-10-01&to=2026-10-31&order_type=Blinkit&status=delivered,grn_entered"
```

#### Response (200) — JSON

```json
{
  "success": true,
  "report": "replenishment_order_wise",
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
  "row_count": 100,
  "rows": [
    {
      "PO Number": "4100123456",
      "Invoice Number": "IN-HR-26/27-0412",
      "Box Count": 42,
      "Total Weight (kg)": "318.40",
      "WH": "Gurgaon Warehouse",
      "Punch Date": "02-Oct-2026",
      "Punch Time": "11:42:10",
      "Order Status": "invoiced",
      "Distinct Sku - Ordered": 5,
      "Qty ordered (case units)": 120,
      "Qty ordered (in units)": 1440,
      "Qty Fulfilled / Dispatched (case units)": 108,
      "Ordered vs Fulfilled % (Qty)": "90.00%",
      "Channel": null,
      "...": "..."
    }
  ]
}
```

- `rows` is keyed by column name, in the order of `columns`. Empty cells are `null` or an empty string.
- `row_count` is the number of orders on this page (one row per order), at most `page_size`.
- Keep requesting `next_page` until it is `null` / `has_more` is `false`.

#### Response (200) — CSV

```
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="replenishment-order-report_<from>_<to>_p<page>.csv"
X-Page: 1
X-Page-Size: 100
X-Total-Orders: 240
X-Total-Pages: 3
X-Has-More: true
```

The body is a header row of the 99 column names, then one row per order, every cell quoted. Pagination is in the `X-*` headers above.

#### Paging through a full range — Python

```python
import os, requests

URL = "https://backend.kleenest.filflo.in/api/v1/external/replenishment-order-report"
HEADERS = {"X-API-Key": os.environ["KLEENEST_API_KEY"]}

def fetch_orders(date_from, date_to, **filters):
    rows, page = [], 1
    while page:
        params = {"from": date_from, "to": date_to, "page": page, "page_size": 100, **filters}
        res = requests.get(URL, headers=HEADERS, params=params, timeout=120)
        res.raise_for_status()
        body = res.json()
        rows.extend(body["rows"])
        page = body["pagination"]["next_page"]
    return rows

orders = fetch_orders("2026-10-01", "2026-10-07", order_type="Blinkit")
print(len(orders), "orders")
```

The JavaScript example above works the same way: change `URL` to `.../replenishment-order-report`.

#### Matches the page download

Cell values match the Replenishment page's Order-wise download, including:

- `Channel`, `Account Managers` and `Tracking Link` are always empty.
- `Qty GRN (in units)` is the plain sum of GRN quantities (same value as `GRN Qty`, not multiplied by case size). The GRN percent and shortfall columns use that value.
- `EDD Date` / `EDD Time` repeat the delivery date and time.
- `State/Zone` and `City` repeat the billing state and city.

Columns that hold the same value as a SKU-wise column use the SKU-wise name, so both reports name shared data the same way. These 7 headers differ from the page download:

| API header (same as SKU-wise) | Header in the page's Order-wise download |
|---|---|
| `PO Number` | PO Number/Order No |
| `Invoice date*` | Invoice date |
| `Invoice time*` | Invoice Time |
| `Order Type` | Platform/Order Type |
| `Total Invoice Amount With Tax` | Total Invoice Value |
| `Mode of Transport` | Dispatch Mode |
| `Carrier/Shipping Partner` | Carrier\* |

- `Total Invoice Amount With Tax` is empty (not `"N/A"` as in SKU-wise) when the order has no e-invoice, like the page download.
- `Total Weight (kg)` keeps its own name because it is the whole order's weight, while the SKU-wise `Weight` is per line.

The Order-wise report's 99 columns are listed in the [Order-wise Column Dictionary](#order-wise-column-dictionary) at the end of this document.

### 3) Get Consolidated Inventory

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

### 4) Get RTO Report

`GET /external/rto-report`

Returns returned-goods (RTO) data. One endpoint, two views:

- **`view=rto` (default): RTO data.** Every Return Inward whose `date_field` falls between `from` and `to`, oldest first, with **one row per Return Inward x crate x Inventory PO**. `Crate Count` shows how many crates were made from the Return Inward. A Return Inward with no crate yet has one row with the crate and PO columns empty (`Remark`: `No crate yet (<status>)`); a crate with no Inventory PO yet has one `No PO raised` row. The crate and PO columns are the same as the Crates page **Crate PO Summary** download.
- **`view=pendancy`: RTO Pendancy data**, the same rows as the RTO Pendancy page: returned goods received at the warehouse but not back in stock yet, one row per SKU. `stage` picks the rows; `stage=short_received` returns units expected but not received yet, with their own columns. The response also carries the page's `summary` totals (computed before the `stage` filter). Dates are optional: without them you get everything pending now.

Requires scope: `rto_report`

#### Query parameters

| Parameter | Views | Required | Description |
|-----------|-------|----------|-------------|
| `view` | both | No | `rto` (default) or `pendancy`. |
| `from` | both | `rto`: **Yes**. `pendancy`: No | Start date, `YYYY-MM-DD`, IST calendar day, inclusive. In the pendancy view pass both `from` and `to` or neither (default: all time). |
| `to` | both | `rto`: **Yes**. `pendancy`: No | End date, `YYYY-MM-DD`, IST calendar day, inclusive. At most **93 days** after `from`. |
| `date_field` | `rto` | No | Which Return Inward date `from`/`to` filter on: `createdAt` (default), `rto_pickup_date` or `rto_delivery_date`. In the pendancy view the dates filter on `Received On` (the crate's inward date; short rows: the last inward). |
| `location` | both | No | Warehouse. One value or comma-separated: `gurgaon losung`, `gurgaon sevenseas`, `mumbai zippee`, `bangalore quiklee`, `chennai quiklee`, `hyderabad quiklee`, `jwl_gurgaon`, `jwl_bangalore`. Default: all. |
| `status` | `rto` | No | Return Inward status. One value or comma-separated: `pending`, `rto_in_transit`, `partially_received`, `received`, `not_received`, `cancelled`. Default: all. |
| `platform` | `rto` | No | Return Inward platform, exactly as stored (e.g. `Blinkit`, `Swiggy`). |
| `stage` | `pendancy` | No | `all` (default: every pending crate and PO row), `crate_pending`, `handover_pending`, `qc_pending`, `po_pending`, or `short_received`. |
| `page` | both | No | Page number, from 1. Default 1. |
| `page_size` | both | No | 1 to 100. Default 100. Counted in **Return Inwards** in the `rto` view (a Return Inward's rows are never split across pages) and in **rows** in the `pendancy` view. |
| `format` | both | No | `json` (default) or `csv`. |

Parameters of the other view are ignored. Dates in the rows are `DD-Mon-YYYY` or `DD-Mon-YYYY HH:mm`, IST. Status and stage values in the rows are shown title-cased, as on the pages (e.g. `Partially Received`).

#### Pendancy stages

| `stage` | Rows |
|---------|------|
| `all` | All pending rows: `crate_pending` + `po_pending`. |
| `crate_pending` | Crate units not yet marked sellable or discarded (`handover_pending` + `qc_pending`). |
| `handover_pending` | Crate not yet handed over to QC. |
| `qc_pending` | Crate handed over, QC not finished. |
| `po_pending` | QC raised a sellable Inventory PO but its QR codes are not fully scanned / inwarded yet. |
| `short_received` | Partially received Return Inwards: units expected but not received yet (different columns). |

#### Response

```json
{
  "success": true,
  "report": "rto_data",
  "generated_at": "2026-11-04T09:30:00.000Z",
  "filters": { "view": "rto", "from": "2026-10-01", "to": "2026-10-31", "date_field": "createdAt",
               "location": null, "status": null, "platform": null },
  "pagination": { "page": 1, "page_size": 100, "total_return_inwards": 140,
                  "total_pages": 2, "has_more": true, "next_page": 2 },
  "columns": ["RTO Inward ID", "RTO Inward Status", "..."],
  "row_count": 236,
  "rows": [
    {
      "RTO Inward ID": "RTO-INWARD-0000000392",
      "RTO Inward Status": "Received",
      "Warehouse": "jwl_gurgaon",
      "Platform": "Swiggy",
      "Credit Note No": "CN-HR-26/27-0419",
      "RI Expected Qty": 21,
      "RI Received Qty": 21,
      "Crate Count": 1,
      "Crate ID": "CRT-0000000013",
      "Crate Status": "Processed",
      "SKU Code": "0000487",
      "Crate Qty (units)": 21,
      "Sellable (units)": 21,
      "Discarded (units)": 0,
      "Inventory PO": "INV-PO/CP2/01102026/9U",
      "PO Type": "B2B - RTO sellable",
      "PO Status": "Fulfilled",
      "PO SKU (Case Pack)": "CP21-0000487",
      "Case Size": 21,
      "Pending (Units)": 0,
      "QRs Linked": 1,
      "QRs Inwarded": 1,
      "Remark": "Inwarded to stock",
      "...": "..."
    }
  ]
}
```

With `view=pendancy`, `report` is `rto_pendancy`, `pagination` has `total_rows` instead of `total_return_inwards`, and there is a `summary`:

```json
{
  "success": true,
  "report": "rto_pendancy",
  "generated_at": "2026-11-04T09:30:00.000Z",
  "filters": { "view": "pendancy", "from": null, "to": null, "location": null, "stage": "all" },
  "summary": { "totalPendingUnits": 749, "pendingSkuLines": 8, "cratesPending": 2,
               "crateHandoverPending": 1, "crateQcPending": 1, "cratePendingUnits": 749,
               "posPending": 0, "poPendingUnits": 0, "returnInwardsPending": 2,
               "oldestPendingDays": 161, "shortReceivedUnits": 1744, "shortReceivedLines": 22 },
  "pagination": { "page": 1, "page_size": 100, "total_rows": 8,
                  "total_pages": 1, "has_more": false, "next_page": null },
  "columns": ["Stage", "Warehouse", "RTO Inward ID", "..."],
  "row_count": 8,
  "rows": [
    { "Stage": "Crate Pending - QC Pending", "Crate ID": "CRT-0000000002", "SKU Code": "0000391",
      "Pending Units": 10, "Received On (IST)": "28-May-2026 15:54", "Age (days)": 159,
      "Action Needed": "QC: mark Sellable or Discard", "...": "..." }
  ]
}
```

| `summary` field | Meaning |
|-----------------|---------|
| `totalPendingUnits` | Units pending across all pending rows. |
| `pendingSkuLines` | Number of pending rows. |
| `cratesPending`, `crateHandoverPending`, `crateQcPending` | Crates with pending units (all / handover pending / QC pending). |
| `cratePendingUnits`, `poPendingUnits` | Pending units at crate stage / on Inventory POs. |
| `posPending` | Inventory POs with QR codes still to scan / inward. |
| `returnInwardsPending` | Return Inwards with pending rows. |
| `oldestPendingDays` | Age of the oldest pending row. |
| `shortReceivedUnits`, `shortReceivedLines` | Units / rows expected but not received yet. |

With `format=csv`, the body is a CSV with a header row and every cell quoted; each page is a complete CSV. Pagination is in the `X-Page`, `X-Page-Size`, `X-Total-Pages`, `X-Has-More` headers, plus `X-Total-Return-Inwards` (`rto` view) or `X-Total-Rows` (`pendancy` view). File name: `rto-report_<view>_<from>_<to>_p<page>.csv`.

#### Examples

```bash
# RTO data for October (JSON)
curl -H "X-API-Key: $KLEENEST_API_KEY" \
  "https://backend.kleenest.filflo.in/api/v1/external/rto-report?from=2026-10-01&to=2026-10-31"

# Received / partially received returns at JWL Gurgaon, as CSV
curl -H "X-API-Key: $KLEENEST_API_KEY" -o rto.csv \
  "https://backend.kleenest.filflo.in/api/v1/external/rto-report?from=2026-10-01&to=2026-10-31&status=received,partially_received&location=jwl_gurgaon&format=csv"

# Everything pending now
curl -H "X-API-Key: $KLEENEST_API_KEY" \
  "https://backend.kleenest.filflo.in/api/v1/external/rto-report?view=pendancy"

# Inventory POs waiting for QR scan at Gurgaon Losung
curl -H "X-API-Key: $KLEENEST_API_KEY" \
  "https://backend.kleenest.filflo.in/api/v1/external/rto-report?view=pendancy&stage=po_pending&location=gurgaon%20losung"

# Short received units
curl -H "X-API-Key: $KLEENEST_API_KEY" \
  "https://backend.kleenest.filflo.in/api/v1/external/rto-report?view=pendancy&stage=short_received"
```

```python
import os, requests

URL = "https://backend.kleenest.filflo.in/api/v1/external/rto-report"
HEADERS = {"X-API-Key": os.environ["KLEENEST_API_KEY"]}

def fetch_all(**params):
    rows, page = [], 1
    while page:
        res = requests.get(URL, headers=HEADERS, params={**params, "page": page, "page_size": 100}, timeout=120)
        res.raise_for_status()
        body = res.json()
        rows.extend(body["rows"])
        page = body["pagination"]["next_page"]
    return rows

rto_rows = fetch_all(**{"from": "2026-10-01", "to": "2026-10-31"})
pending_rows = fetch_all(view="pendancy")
short_rows = fetch_all(view="pendancy", stage="short_received")
```

The columns of each view are listed in the [RTO Column Dictionary](#rto-column-dictionary) at the end of this document.

## Status Values

Accepted by the replenishment reports' `status` filter, SKU-wise and Order-wise (comma-separate to combine):

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
| `400` | `invalid_date_range`, `date_range_too_large`, `invalid_date_field`, `invalid_format`, `invalid_page`, `invalid_page_size`, `invalid_location`, `invalid_include_pending_inward`, `invalid_include_pallets`, `invalid_view`, `invalid_status`, `invalid_stage` | Fix the query parameters. |
| `401` | `missing_api_key`, `invalid_api_key`, `expired_api_key` | Check the key, or ask FilFlo for a new one. |
| `403` | `insufficient_scope` | The key is not allowed to use this endpoint. |
| `429` | `rate_limited`, `concurrent_request` | Wait `Retry-After` seconds, or for your previous request to finish. |
| `500` | `internal_error` | Retry with backoff. If it persists, contact FilFlo. |

## Retries

- All endpoints are read-only and safe to retry.
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

## Order-wise Column Dictionary

Replenishment Order-wise report columns. They appear in this order in both JSON (`columns`) and CSV (header row). "(case units)" columns are in cases; "(in units)" columns multiply each SKU's cases by its case size. Percent columns are "N/A" when the denominator is 0. Dates are `DD-Mon-YYYY` and times `HH:mm:ss`, in IST.

| # | Column | Meaning |
|---|---|---|
| 1 | PO Number | Customer PO number (FilFlo order ID). |
| 2 | Invoice Number | Tax invoice / delivery note number. Blank until invoiced. |
| 3 | LR Number | Lorry receipt / AWB number. |
| 4 | GRN Qty | Total GRN quantity across all SKUs of the order, as recorded at GRN. |
| 5 | Invoice Qty | Total invoiced (fulfilled) quantity across all SKUs, in cases. |
| 6 | Box Count | Boxes outwarded for the order. |
| 7 | Total Weight (kg) | Total weight (kg) of the boxes outwarded for the order, 2 decimals. |
| 8 | WH | Fulfilment warehouse. |
| 9 | Supplier GSTIN | Seller GSTIN on the e-invoice. |
| 10 | Supplier Name | Seller legal name on the e-invoice. |
| 11 | Bill from Address | Seller address on the e-invoice. |
| 12 | Dispatch from Address | Dispatch address on the e-invoice (same as the seller address today). |
| 13 | PO Date | PO date as received from the customer. |
| 14 | PO Time | PO time as received from the customer. |
| 15 | Appointment Date | Delivery appointment date at the customer. |
| 16 | Appointment Time | Delivery appointment time at the customer. |
| 17 | Appointment ID | Customer appointment reference. |
| 18 | PO Expiry Date | PO expiry date. |
| 19 | PO Expiry Time | PO expiry time. |
| 20 | Punch Date | Date the order was created in FilFlo. |
| 21 | Punch Time | Time the order was created in FilFlo. |
| 22 | Approved Date | Date the order was approved. |
| 23 | Approved Time | Time the order was approved. |
| 24 | Picked Date | Date the order was picked. |
| 25 | Picked Time | Time the order was picked. |
| 26 | Scheduled Dispatch Date | Planned dispatch date. |
| 27 | Scheduled Dispatch Time | Planned dispatch time. |
| 28 | Invoice date\* | Invoice date. |
| 29 | Invoice time\* | Invoice time. |
| 30 | Dispatch Date | Actual dispatch date. |
| 31 | Dispatch Time | Actual dispatch time. |
| 32 | Delivery Date | Delivery date. |
| 33 | Delivery Time | Delivery time. |
| 34 | Voucher Type | "Tax Invoice" or "Delivery Note". |
| 35 | Order Status | Order status, e.g. open, approved, picked, invoiced, in_transit, delivered, grn_entered, cancelled. |
| 36 | Invoice Status | "Invoiced", "Not Invoiced" or "Invoice Cancelled". |
| 37 | Order Remarks | Remarks entered on the order. |
| 38 | Order Type | Channel, e.g. Blinkit, Zepto, Swiggy, Modern-Trade, General-Trade. |
| 39 | PO Type | "Local" or "Outstation". |
| 40 | Total Invoice Amount With Tax | Invoice total including tax, from the e-invoice. Empty if there is no e-invoice. |
| 41 | Customer | Customer (bill-to party) name. |
| 42 | Customer GST | Customer GSTIN. |
| 43 | Billing Address | Customer billing address. |
| 44 | Bill To State | Billing state. |
| 45 | Bill To City | Billing city. |
| 46 | Bill To Pincode | Billing pincode. |
| 47 | Ship To Partyname | Ship-to party name. |
| 48 | Shipping Address | Ship-to address or delivery location. |
| 49 | Ship To GSTIN | Ship-to GSTIN. |
| 50 | Ship To State | Ship-to state. |
| 51 | Ship To Pincode | Ship-to pincode. |
| 52 | Channel | Always empty (kept so the columns match the page download). |
| 53 | State/Zone | Billing state (same as Bill To State). |
| 54 | City | Billing city (same as Bill To City). |
| 55 | Account Managers | Always empty (kept so the columns match the page download). |
| 56 | Distinct Sku - Ordered | Number of SKUs on the order. |
| 57 | Qty ordered (case units) | Total ordered quantity, in cases. |
| 58 | Qty ordered (in units) | Total ordered quantity, each SKU's cases x its case size. |
| 59 | Distinct Sku - Approved | Number of SKUs with an approved quantity above 0. |
| 60 | Qty Approved (case units) | Total approved quantity, in cases. |
| 61 | Qty Approved (in units) | Total approved quantity, each SKU's cases x its case size. |
| 62 | Distinct Sku - Fulfilled/ Dispatched | Number of SKUs with a fulfilled quantity above 0. |
| 63 | Qty Fulfilled / Dispatched (case units) | Total fulfilled / dispatched quantity, in cases. |
| 64 | Qty Fulfilled / Dispatched (in units) | Total fulfilled / dispatched quantity, each SKU's cases x its case size. |
| 65 | Distinct Sku - GRN | Number of SKUs with a GRN quantity above 0. |
| 66 | Qty GRN (in units) | Total GRN quantity. Same value as GRN Qty (not multiplied by case size). |
| 67 | Short Fulfilled Quantity (SKU) | Distinct SKUs approved minus distinct SKUs fulfilled. |
| 68 | Short Fulfilled Quantity (Units) | Approved units minus fulfilled units. |
| 69 | Short GRN Quantity (SKU) | Distinct SKUs fulfilled minus distinct SKUs with a GRN quantity. |
| 70 | Short GRN Quantity (Units) | Fulfilled units minus Qty GRN (in units). |
| 71 | Ordered vs Approved % (SKU) | Distinct SKUs approved / ordered, as a percent ("N/A" when ordered is 0). |
| 72 | Ordered vs Approved % (Qty) | Approved units / ordered units, as a percent ("N/A" when ordered is 0). |
| 73 | Approved vs Fulfilled % (SKU) | Distinct SKUs fulfilled / approved, as a percent ("N/A" when approved is 0). |
| 74 | Approved vs Fulfilled % (Qty) | Fulfilled units / approved units, as a percent ("N/A" when approved is 0). |
| 75 | Ordered vs Fulfilled % (SKU) | Distinct SKUs fulfilled / ordered, as a percent ("N/A" when ordered is 0). |
| 76 | Ordered vs Fulfilled % (Qty) | Fulfilled units / ordered units, as a percent ("N/A" when ordered is 0). |
| 77 | Fulfilled vs GRN % (SKU) | Distinct SKUs with GRN / fulfilled, as a percent ("N/A" when fulfilled is 0). |
| 78 | Fulfilled vs GRN % (Qty) | Qty GRN (in units) / fulfilled units, as a percent ("N/A" when fulfilled is 0). |
| 79 | Ordered vs GRN % (SKU) | Distinct SKUs with GRN / ordered, as a percent ("N/A" when ordered is 0). |
| 80 | Ordered vs GRN % (Qty) | Qty GRN (in units) / ordered units, as a percent ("N/A" when ordered is 0). |
| 81 | Approval Remarks | Distinct approval remarks of the order's SKUs, comma-separated. |
| 82 | WH/Fulfillment Remarks | Distinct fulfilment remarks of the order's SKUs, comma-separated. |
| 83 | GRN Remarks | Distinct GRN remarks of the order's SKUs, comma-separated. |
| 84 | Reason for Shortfall | Order-level shortfall reason. |
| 85 | EDD Date | Expected delivery date (reported as the delivery date). |
| 86 | EDD Time | Expected delivery time (reported as the delivery time). |
| 87 | Mode of Transport | Mode of transport. |
| 88 | Carrier/Shipping Partner | Carrier / transporter. |
| 89 | Tracking Link | Always empty (kept so the columns match the page download). |
| 90 | Dispatch Status | "True" once the order is in transit, delivered or GRN entered. |
| 91 | Delivery Status | "True" once the order is delivered or GRN entered. |
| 92 | Has Proof of Delivery | "True" if a proof of delivery is attached. |
| 93 | Proof of Delivery | Proof of delivery file URL. |
| 94 | Has PO Proof | "True" if the PO copy is attached. |
| 95 | PO Proof | PO copy file URL. |
| 96 | Has Proof of GRN | "True" if a GRN copy is attached. |
| 97 | Proof of GRN | GRN copy file URL. |
| 98 | Has Proof of Dispatch | "True" if a proof of dispatch is attached. |
| 99 | Proof of Dispatch | Proof of dispatch file URL. |

## RTO Column Dictionary

RTO report columns. They appear in this order in both JSON (`columns`) and CSV (header row). Dates are `DD-Mon-YYYY` or `DD-Mon-YYYY HH:mm`, in IST. Empty cells are `""`.

### `view=rto` (40 columns)

One row per Return Inward x crate x Inventory PO. The Return Inward columns (1 to 17) repeat on every row of the Return Inward.

| # | Column | Meaning |
|---|---|---|
| 1 | RTO Inward ID | Return Inward number. Every row of a Return Inward is on the same page. |
| 2 | RTO Inward Status | Return Inward status, e.g. Pending, Rto In Transit, Partially Received, Received, Not Received, Cancelled. |
| 3 | Warehouse | Warehouse the return comes back to (lowercase, as in FilFlo). |
| 4 | Platform | Platform / channel of the return, as entered on the Return Inward. |
| 5 | Credit Note No | Credit note linked to the Return Inward. |
| 6 | Credit Note Date | Credit note date, DD-Mon-YYYY (IST). |
| 7 | Credit Note Amount | Credit note total amount. |
| 8 | Invoice No | Original invoice the credit note was raised against. |
| 9 | Customer | B2B customer on the credit note. |
| 10 | AWB Number | Return AWB number. |
| 11 | Vehicle Number | Return vehicle number. |
| 12 | RTO Pickup Date | RTO pickup date, DD-Mon-YYYY (IST). |
| 13 | RTO Delivery Date | RTO delivery date at the warehouse, DD-Mon-YYYY (IST). |
| 14 | RI Expected Qty | Units expected back on the whole Return Inward (all SKUs). Repeated on every row of the Return Inward. |
| 15 | RI Received Qty | Units received so far on the whole Return Inward (all SKUs). Repeated on every row of the Return Inward. |
| 16 | RI Created (IST) | When the Return Inward was created, DD-Mon-YYYY HH:mm (IST). |
| 17 | Crate Count | Number of crates made from this Return Inward. 0 when no crate is made yet. |
| 18 | Crate ID | Crate made at inward. Empty when the Return Inward has no crate yet. |
| 19 | Crate Status | Handover Pending, Handover Done, Partially Processed, Sellable Po Pending or Processed. |
| 20 | Crate Created (IST) | When the crate was made, DD-Mon-YYYY HH:mm (IST). |
| 21 | SKU Code | Crate SKU (unit SKU). On a "No PO raised" row, every SKU in the crate, separated by " \| ". |
| 22 | Product Name | Crate product name(s). |
| 23 | Crate Qty (units) | Units of this SKU in the crate (whole crate on a "No PO raised" row). |
| 24 | Sellable (units) | Units QC marked sellable. |
| 25 | Discarded (units) | Units QC discarded. |
| 26 | Inventory PO | Inventory PO raised when QC marked units sellable, or "No PO raised". |
| 27 | PO Type | "B2B - RTO sellable" (case-pack PO, QR codes) or "B2C - RTO". |
| 28 | PO Status | Inventory PO status. Fulfilled means the stock is back in inventory. |
| 29 | PO SKU (Case Pack) | Case-pack SKU the units were packed into. |
| 30 | Case Size | Units per case pack (1 for B2C). |
| 31 | Ordered (Case Packs) | B2B only: case packs on the PO. |
| 32 | Received (Case Packs) | B2B only: case packs scanned / inwarded. |
| 33 | Ordered (Units) | Units on the PO. |
| 34 | Received (Units) | Units inwarded on the PO. |
| 35 | Pending (Units) | Units still to inward on the PO. |
| 36 | QRs Linked | B2B only: QR codes linked to the PO. |
| 37 | QRs Inwarded | B2B only: linked QR codes already inwarded. |
| 38 | PO Created (IST) | When the PO was raised, DD-Mon-YYYY HH:mm (IST). |
| 39 | PO Fulfilled (IST) | When the PO was fulfilled, DD-Mon-YYYY HH:mm (IST). |
| 40 | Remark | Where the crate / PO stands, e.g. "Inwarded to stock", "QR scan / inward pending", "QC not started", "No crate yet (Rto In Transit)". |

### `view=pendancy` (29 columns)

One row per pending SKU (stages `all`, `crate_pending`, `handover_pending`, `qc_pending`, `po_pending`). Crate rows leave the PO columns empty; PO rows leave `Sellable (units)` and `Discarded (units)` empty.

| # | Column | Meaning |
|---|---|---|
| 1 | Stage | Crate Pending - Handover Pending, Crate Pending - QC Pending, PO Pending (QR not scanned), or Short Received. |
| 2 | Warehouse | Warehouse the return comes back to (lowercase, as in FilFlo). |
| 3 | RTO Inward ID | Return Inward number. Every row of a Return Inward is on the same page. |
| 4 | RTO Inward Status | Return Inward status, e.g. Pending, Rto In Transit, Partially Received, Received, Not Received, Cancelled. |
| 5 | Credit Note No | Credit note linked to the Return Inward. |
| 6 | Invoice No | Original invoice the credit note was raised against. |
| 7 | Customer | B2B customer on the credit note. |
| 8 | Platform | Platform / channel of the return, as entered on the Return Inward. |
| 9 | Crate ID | Crate made at inward. Empty when the Return Inward has no crate yet. |
| 10 | Crate Status | Handover Pending, Handover Done, Partially Processed, Sellable Po Pending or Processed. |
| 11 | Inventory PO | Inventory PO raised when QC marked units sellable, or "No PO raised". |
| 12 | PO Type | "B2B - RTO sellable" (case-pack PO, QR codes) or "B2C - RTO". |
| 13 | PO Status | Inventory PO status. Fulfilled means the stock is back in inventory. |
| 14 | SKU Code | Crate SKU (unit SKU). On a "No PO raised" row, every SKU in the crate, separated by " \| ". |
| 15 | Product Name | Crate product name(s). |
| 16 | Received Qty (units) | Crate rows: units of the SKU in the crate. PO rows: units on the PO (case packs x case size). |
| 17 | Sellable (units) | Units QC marked sellable. |
| 18 | Discarded (units) | Units QC discarded. |
| 19 | PO SKU (Case Pack) | Case-pack SKU the units were packed into. |
| 20 | Case Size | Units per case pack (1 for B2C). |
| 21 | PO Ordered (Case Packs) | PO rows: case packs on the PO. |
| 22 | PO Received (Case Packs) | PO rows: case packs inwarded. |
| 23 | PO Pending (Case Packs) | PO rows: case packs still to inward. |
| 24 | QRs Linked | B2B only: QR codes linked to the PO. |
| 25 | QRs Inwarded | B2B only: linked QR codes already inwarded. |
| 26 | Pending Units | Units not back in stock yet: not yet QC-checked (crate rows) or not yet inwarded (PO rows). |
| 27 | Received On (IST) | When the crate was made (the inward), DD-Mon-YYYY HH:mm (IST). The date filter uses this date. |
| 28 | Age (days) | Days since Received On (short rows: since the last inward). |
| 29 | Action Needed | Next step: Handover crate to QC, QC: mark Sellable or Discard, or Scan QR & inward on Inventory PO. |

### `view=pendancy&stage=short_received` (17 columns)

One row per SKU of a partially received Return Inward that is still short.

| # | Column | Meaning |
|---|---|---|
| 1 | Stage | Crate Pending - Handover Pending, Crate Pending - QC Pending, PO Pending (QR not scanned), or Short Received. |
| 2 | Warehouse | Warehouse the return comes back to (lowercase, as in FilFlo). |
| 3 | RTO Inward ID | Return Inward number. Every row of a Return Inward is on the same page. |
| 4 | RTO Inward Status | Return Inward status, e.g. Pending, Rto In Transit, Partially Received, Received, Not Received, Cancelled. |
| 5 | Credit Note No | Credit note linked to the Return Inward. |
| 6 | Invoice No | Original invoice the credit note was raised against. |
| 7 | Customer | B2B customer on the credit note. |
| 8 | Platform | Platform / channel of the return, as entered on the Return Inward. |
| 9 | AWB Number | Return AWB number. |
| 10 | Crate IDs | Short rows: crates already made from the Return Inward, separated by " \| ". |
| 11 | SKU Code | Crate SKU (unit SKU). On a "No PO raised" row, every SKU in the crate, separated by " \| ". |
| 12 | Product Name | Crate product name(s). |
| 13 | Expected Qty | Short rows: units expected for the SKU. |
| 14 | Received Qty | Short rows: units received for the SKU. |
| 15 | Short Qty | Short rows: units expected but not received yet. |
| 16 | Last Inward On (IST) | Short rows: last inward on the Return Inward. The date filter uses this date. |
| 17 | Age (days) | Days since Received On (short rows: since the last inward). |
