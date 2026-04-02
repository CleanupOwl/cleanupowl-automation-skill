# Basic examples

Minimal, copy-paste-ready requests against the CleanupOwl API under **`/tp/api/`**. Requests use JSON bodies on `POST` where noted and the usual success/error envelope—see [API reference](../api-reference.md).

Set these shell variables first (same pattern as [Quick start](../api-reference.md#quick-start)):

```bash
export TP_BASE_URL="https://app.cleanupowl.com"
export TP_API_KEY="your-api-key-here"
export COMPANY_ID="your-company-document-id"
```

Use an API key from your CleanupOwl account. **`companyId`** on scan and solve calls is the **`_id`** from the connections list—not the QBO realm id (see [`GET /tp/api/connections`](../api-reference.md#get-tpapiconnections)).

---

## List connected QBO companies

**Scope:** none

```bash
curl -sS "$TP_BASE_URL/tp/api/connections" \
  -H "Authorization: Bearer $TP_API_KEY"
```

Copy the connection document’s **`_id`** and use it as **`COMPANY_ID`**.

---

## List issue types

**Scope:** `scan`

```bash
curl -sS "$TP_BASE_URL/tp/api/issues" \
  -H "Authorization: Bearer $TP_API_KEY"
```

---

## Start a scan

**Scope:** `scan`

A **scan** pulls QuickBooks Online (QBO) data (unless sync is skipped), runs issue checks, and stores results. **`POST /tp/api/scan/start`** returns immediately; then poll **`GET /tp/api/scan/:scanId`** until the run finishes.

```bash
curl -sS -X POST "$TP_BASE_URL/tp/api/scan/start" \
  -H "Authorization: Bearer $TP_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"companyId\": \"$COMPANY_ID\",
    \"period\": {
      \"startDate\": \"2024-01-01\",
      \"endDate\": \"2024-12-31\"
    }
  }"
```

From the response, copy **`data.scanId`** as **`SCAN_ID`**. Note **`data.pollAfterSeconds`**—wait at least that many seconds before the first poll (often **300**).

---

## Poll scan status

**Scope:** `scan`

Scans run in the background. Wait at least **`pollAfterSeconds`** after starting, then poll with **exponential backoff** (5s → 10s → 20s → 30s, max 30s between attempts) until **`data.scan.status`** is **`completed`** or **`failed`**.

```bash
export SCAN_ID="paste-scanId-from-response-here"

curl -sS "$TP_BASE_URL/tp/api/scan/$SCAN_ID?companyId=$COMPANY_ID" \
  -H "Authorization: Bearer $TP_API_KEY"
```

---

## Scan history

**Scope:** `scan`

```bash
curl -sS "$TP_BASE_URL/tp/api/scan/history?companyId=$COMPANY_ID&limit=10" \
  -H "Authorization: Bearer $TP_API_KEY"
```

---

## Billing

**Scope:** none

```bash
curl -sS "$TP_BASE_URL/tp/api/billing" \
  -H "Authorization: Bearer $TP_API_KEY"
```

---

## Connection URL (link QBO)

**Scope:** none

```bash
curl -sS "$TP_BASE_URL/tp/api/connect" \
  -H "Authorization: Bearer $TP_API_KEY"
```

Open **`data.connectionUrl`** in a browser. The user signs in to CleanupOwl and completes QBO authorization; the company then appears in the connections list.
