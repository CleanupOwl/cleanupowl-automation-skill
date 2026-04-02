# API Reference

> **Beta:** This API is in **beta**. Endpoints, response shapes, and documentation may change without a version bump; watch release notes and update integrations accordingly.

The CleanupOwl API is organized around [REST](https://en.wikipedia.org/wiki/Representational_State_Transfer). It uses predictable, resource-oriented URLs under **`/tp/api/`**, accepts [JSON](https://www.json.org/) request bodies on `POST` requests (`Content-Type: application/json`), returns JSON responses in a consistent envelope, and uses standard HTTP response codes, [authentication](#authentication), and verbs (`GET`, `POST`).

There is **no `/v1`-style path prefix**. **Breaking changes** and behavior updates are communicated **out of band** (for example, release notes or partner channels); keep an eye on those if you maintain an integration.

**Scans run in the background.** After you start a scan, poll for results until the run finishes or fails—see [Quick start](#quick-start) for the full polling flow. Solvers fix **one row at a time** using prep and apply. There is no “fix every row in one call” option.

## Not a developer?

This API is for **integrators**: scripts, partner tools, internal automation, and **LLM agents**. To use CleanupOwl without writing to the API, use the main CleanupOwl application and QuickBooks Online (QBO)–connected workflows in the product UI.

## Base URL

```plaintext
https://app.cleanupowl.com
```

Append **`/tp/api/...`** to that host for API requests (for example, `https://app.cleanupowl.com/tp/api/connections`).

## Authentication

The CleanupOwl API uses **API keys** to authenticate requests. You create and manage keys in the CleanupOwl application, where you choose the **scopes** (permissions) each key needs.

Each key is a secret string you send as a **Bearer** token. **Scopes** say what the key may do—for example: run **scans**, list **issues**, use **solvers**, or only open **connections** and **billing**. The full list is under [Scopes](#scopes).

API keys grant access to your CleanupOwl data and actions tied to your account. Keep them private. Do not put them in public git repos, browser-side code, shared docs, or support threads. If a key is exposed, revoke it and create a new one.

All requests must use **HTTPS**. Requests without a valid `Authorization: Bearer <your-api-key>` header fail with **401** or **403**. The API does not accept unauthenticated calls.

```http
Authorization: Bearer <your-api-key>
```

### Authenticated request

```bash
curl -sS "https://app.cleanupowl.com/tp/api/connections" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Use a key from your CleanupOwl account in place of `YOUR_API_KEY`.

### Scopes

| Scope | Required for |
|-------|----------------|
| `scan` | `GET /tp/api/issues`, scan endpoints such as `/tp/api/scan/...` |
| `solve` | `GET /tp/api/solvers`, `GET /tp/api/issues/:issueId/solvers`, `POST /tp/api/solve/prep`, `POST /tp/api/solve/apply`, `GET /tp/api/scans/:scanId/solves` |
| *(none)* | `GET /tp/api/billing`, `GET /tp/api/connections`, `GET /tp/api/connect` |

### Auth error schema

HTTP status is **401** or **403**. The body uses the standard [error envelope](#errors).

| `error.code` | HTTP | Meaning |
|--------------|------|---------|
| `E_MISSING_API_KEY` | 401 | No `Authorization: Bearer` header |
| `E_INVALID_API_KEY` | 401 | Unknown or invalid key |
| `E_API_KEY_REVOKED` | 401 | Key revoked |
| `E_API_KEY_EXPIRED` | 401 | Key expired |
| `E_SCOPE_FORBIDDEN` | 403 | Key missing required scope |

```json
{
  "success": false,
  "error": {
    "code": "E_INVALID_API_KEY",
    "message": "Invalid API key format"
  }
}
```

Integrations should branch on **`error.code`**; use **`message`** for display and logging.

## Quick start

Learn how to scan a connected QuickBooks Online company for bookkeeping issues and apply a fix—using only `curl` from your terminal.

### Introduction

This guide walks you through the full flow: create an API key, connect a company, run a scan, read the results, and fix one row. Each step explains what the call does and what to do with the response before moving on.

Only `curl` is required. Every command prints JSON—read it and copy the values you need into the next command.

For full reference documentation on any endpoint, see the sections below: [Issues](#1-issues), [Scans](#2-scans), [Solvers](#3-solvers), [Connections](#5-connections).

---

### 1. Create an API key

Every request must include an API key in the `Authorization` header. The key also controls what you can do—see [Scopes](#scopes).

Go to your CleanupOwl account and create a key. Choose the scopes you need:

- `scan` — to run scans and list issues
- `solve` — to prep and apply fixes
- No scope needed for connections and billing

> **Keep your key secret.** Do not commit it to git or paste it in public docs. If it leaks, revoke it and create a new one immediately.

Set your key and base URL as environment variables so you can copy the commands below without editing them:

```bash
export TP_BASE_URL="https://app.cleanupowl.com"
export TP_API_KEY="your-api-key-here"
```

To verify your key works, call the issues endpoint:

```bash
curl -sS "$TP_BASE_URL/tp/api/issues" \
  -H "Authorization: Bearer $TP_API_KEY"
```

A `200` with a list of issues means your key is valid. A `401` means the key is wrong, expired, or missing the `scan` scope.

---

### 2. Connect a QuickBooks Online company

Scans and fixes run against a specific QBO company. If no company is connected yet, get the browser link:

```bash
curl -sS "$TP_BASE_URL/tp/api/connect" \
  -H "Authorization: Bearer $TP_API_KEY"
```

Open `data.connectionUrl` in a browser. The user signs in to CleanupOwl and completes QBO authorization. Once done, the company appears in the connections list.

---

### 3. Get your `companyId`

The `companyId` is required on every scan and solve call. It is the `_id` field from the connections list—not the QBO realm id.

```bash
curl -sS "$TP_BASE_URL/tp/api/connections" \
  -H "Authorization: Bearer $TP_API_KEY"
```

In the response, find the company you want and copy its `_id` value.

```bash
export COMPANY_ID="paste-_id-from-connections-here"
```

---

### 4. Start a scan

A scan pulls data from QBO, checks it for bookkeeping issues, and stores the results. You need a completed scan before you can fix anything.

```bash
curl -sS -X POST "$TP_BASE_URL/tp/api/scan/start" \
  -H "Authorization: Bearer $TP_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"companyId\":\"$COMPANY_ID\"}"
```

From the response, copy `data.scanId`. Also note `data.pollAfterSeconds`—**wait at least that many seconds** before your first poll. The server sets this value; it is commonly **300 seconds**.

```bash
export SCAN_ID="paste-scanId-from-response-here"
```

---

### 5. Poll until the scan finishes

Scans run in the background. You need to keep checking until the status changes.

Wait at least `pollAfterSeconds` after starting, then run this command. Repeat it with **exponential backoff** (5s → 10s → 20s → 30s, max 30s between attempts):

```bash
curl -sS "$TP_BASE_URL/tp/api/scan/$SCAN_ID?companyId=$COMPANY_ID" \
  -H "Authorization: Bearer $TP_API_KEY"
```

Check `data.scan.status` in each response:

- `completed` — results are ready, continue to the next step
- `failed` — check `data.scan.error` and decide whether to retry
- anything else — wait and poll again

---

### 6. Read the results

When the scan is `completed`, `data.scan.results` holds every issue found—organized as a map of `issueId → rows`.

Each row represents one flagged record. To fix a row you need its `recordId`, which is one of `Id`, `_id`, `entityId`, or `id` on the row object. For details, see [Identifying a row](#identifying-a-row-for-post-tpapisolveapply).

Pick an issue bucket, then pick a row you want to fix, and note its `issueId` and `recordId`.

---

### 7. Prep the fix

Before applying, call prep to get the available solvers, row-level options, and any risk flags for that specific row.

```bash
curl -sS -X POST "$TP_BASE_URL/tp/api/solve/prep" \
  -H "Authorization: Bearer $TP_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"scanId\":\"$SCAN_ID\",\"issueId\":\"YOUR_ISSUE_ID\",\"recordId\":\"YOUR_RECORD_ID\",\"companyId\":\"$COMPANY_ID\"}"
```

In the response:

- `data.solvers` — pick a `solverId` from this list
- `data.recordFixOptions` and `data.metaFixOptions` — use these to build `userInput` if the solver needs it
- `data.resolvedByOtherFix: true` — skip this row, it was already fixed
- `data.needsRevalidation: true` — QBO data changed; run a new scan before applying

---

### 8. Apply the fix

This is the call that changes data in QuickBooks Online.

```bash
curl -sS -X POST "$TP_BASE_URL/tp/api/solve/apply" \
  -H "Authorization: Bearer $TP_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"scanId\":\"$SCAN_ID\",\"issueId\":\"YOUR_ISSUE_ID\",\"recordId\":\"YOUR_RECORD_ID\",\"companyId\":\"$COMPANY_ID\",\"solverId\":\"YOUR_SOLVER_ID\"}"
```

Read the response:

- `success: true` with `data.action` — the fix was applied. Check `data.message` for a description.
- `success: true` with `data.needsInput: true` — the solver needs more information. Add `userInput` with the fields listed in `data.userInputSchema` and POST again with the same ids.
- `success: false` — something went wrong. Read `error.code` and `error.message` to fix the request.

---

### 9. Review what was fixed

To see an audit trail of every solve attempt on the scan:

```bash
curl -sS "$TP_BASE_URL/tp/api/scans/$SCAN_ID/solves?companyId=$COMPANY_ID" \
  -H "Authorization: Bearer $TP_API_KEY"
```

Each entry in `data.solves` shows the `action`, `success`, `solverId`, and `executedAt` timestamp for one attempt.

---

### 10. Check your plan limits

To see how many fixes you have used and how many remain:

```bash
curl -sS "$TP_BASE_URL/tp/api/billing" \
  -H "Authorization: Bearer $TP_API_KEY"
```

If `usage.fixesRemaining` is `0`, you have reached your plan limit. The apply endpoint returns `E_FIX_LIMIT_EXCEEDED` (402) when this happens.

---

### Next steps

- Read the full endpoint reference: [Issues](#1-issues), [Scans](#2-scans), [Solvers](#3-solvers), [Connections](#5-connections), [Billing](#4-billing)
- Understand the response envelope and error handling: [Response format](#response-format), [Errors](#errors)
- Learn what each `error.code` means and how to handle it: [Common `error.code` values](#common-errorcode-values)

## Table of contents

| Step | Section | What it covers |
|------|---------|----------------|
| 1 | [Authentication](#authentication) | API keys, scopes, and how to authorize requests |
| 2 | [Quick start](#quick-start) | Run your first scan and apply a fix in under 5 minutes |
| 3 | [Response format](#response-format) | The success envelope, error shape, and `needsInput` behavior |
| 4 | [Errors](#errors) | HTTP codes, error object attributes, and common `error.code` values |
| 5 | [Common mistakes](#common-mistakes) | The most frequent integration mistakes and how to avoid them |
| 6 | [Issues](#1-issues) | List available issue types |
| 7 | [Scans](#2-scans) | Start a scan, poll for results, and read scan history |
| 8 | [Solvers](#3-solvers) | Prep and apply fixes row by row |
| 9 | [Billing](#4-billing) | Check usage and plan limits |
| 10 | [Connections](#5-connections) | List connected QBO companies and get a connection URL |
| 11 | [Postman collection](#postman-collection) | Run requests quickly with the bundled Postman collection |

## Response format

Every response—success or failure—uses the same outer shape:

```json
{
  "success": true,
  "data": { }
}
```

When a request fails, `success` is `false` and the body contains an `error` object instead of `data`:

```json
{
  "success": false,
  "error": {
    "code": "E_ERROR_CODE",
    "message": "Human-readable description"
  }
}
```

The `data` object shape is endpoint-specific. See each endpoint section for its full attribute list.

### Solver needs more input — `POST /tp/api/solve/apply`

`POST /tp/api/solve/apply` has a special case: the solver may need additional fields before it can run. When this happens the response is still HTTP **200** with `success: true`—it is **not** a failure. The signal is `data.needsInput: true`:

```json
{
  "success": true,
  "data": {
    "needsInput": true,
    "missingFields": ["targetAccountId"],
    "userInputSchema": { }
  }
}
```

When you receive `data.needsInput: true`, add a `userInput` object with the fields listed in `missingFields` and POST to `/tp/api/solve/apply` again using the same `scanId`, `issueId`, `recordId`, `companyId`, and `solverId`. Do not treat this as an error.

### Error

All error responses share the same shape regardless of endpoint:

```json
{
  "success": false,
  "error": {
    "code": "E_MISSING_COMPANY_ID",
    "message": "companyId is required"
  }
}
```

`error.code` is a stable string you can branch on in code. `error.message` is a human-readable explanation intended for logs and dashboards—its wording may change between releases, so do not match on it in production logic.

## Errors

The CleanupOwl API uses conventional HTTP status codes together with a JSON body to show whether a request succeeded. In general: codes in the **`2xx`** range mean the HTTP request was accepted and you should read **`success`** and either **`data`** or **`error`** in the body. Codes in the **`4xx`** range mean the request failed because of the client input, auth, or permissions (wrong or missing parameters, invalid key, wrong scope, unknown resource, and similar). Codes in the **`5xx`** range mean something went wrong on the server side; treat these as uncommon and retry with backoff when safe.

Many **`4xx`** responses include an **`error.code`** string you can branch on in code. **`error.message`** is meant for humans—logs, dashboards, or support—not as a stable contract.

### Error object attributes

When **`success`** is **`false`**, the body includes an **`error`** object:

- **`code`** (string)  
  Stable machine-readable code, for example **`E_MISSING_COMPANY_ID`**. Use this in `switch` / `if` logic.

- **`message`** (string)  
  Human-readable explanation. Wording may change with update; do not match on the full string in production logic.

Successful responses use **`success: true`** and **`data`**. For **`POST /tp/api/solve/apply`**, a **`200`** with **`success: true`** can still mean the solver needs more fields—see [Solver needs more input](#solver-needs-more-input--post-tpapisolveapply) under [Response format](#response-format) (`data.needsInput`), which is **not** the same as **`success: false`**.

### HTTP status code summary

| HTTP | Meaning | Typical case |
|------|---------|----------------|
| **200** | OK | **`success: true`** with **`data`**. For apply, may include **`data.needsInput`** (still success). |
| **400** | Bad Request | Invalid or incomplete payload; wrong solver/issue pairing; business validation failed. Check **`error.code`**. |
| **401** | Unauthorized | Missing or invalid API key, expired/revoked key, or QuickBooks Online (QBO) authentication failure for the company. |
| **403** | Forbidden | Key lacks scope, or **`companyId`** is not allowed for this user. |
| **404** | Not Found | Unknown scan, record, or route resource. |
| **500** | Internal Server Error | Unexpected server-side failure; retry later if appropriate. |
| **503** | Service Unavailable | Dependency unavailable (for example billing not configured in this environment). |

Other status codes may appear for specific routes; each endpoint section lists its **`error.code`** values where relevant.

### Common `error.code` values

These appear across many endpoints; additional codes are documented per route below.

| `error.code` | Typical HTTP | When | What to do |
|--------------|--------------|------|------------|
| `E_MISSING_API_KEY` | 401 | No Bearer token | Send `Authorization: Bearer ...` |
| `E_INVALID_API_KEY` | 401 | Bad key | Fix or rotate key |
| `E_SCOPE_FORBIDDEN` | 403 | Wrong scope | Use a key with `scan` / `solve` as needed |
| `E_MISSING_COMPANY_ID` | 400 | Required `companyId` missing | Pass body or query `companyId` |
| `E_COMPANY_ACCESS_DENIED` | 403 | Company not linked to key user | Use `GET /tp/api/connections` for allowed company ids |
| `E_QBO_AUTH_FAILED` | 401 | QuickBooks Online (QBO) connection invalid | Reconnect QBO in CleanupOwl |

## Common mistakes

| Mistake | What goes wrong | Fix |
|---------|-----------------|-----|
| Using anything other than **`connections[]._id`** for **`companyId`** | **`pollUrl`**, scan payloads, and solve calls will not line up | Set **`companyId`** to **`_id`** from **`GET /tp/api/connections`** every time |
| Omitting **`companyId`** on scan GET, history, or solves | 400 `E_MISSING_COMPANY_ID` | Always pass query or body **`companyId`** |
| Wrong **`recordId`** | 404 `E_RECORD_NOT_FOUND` | Use **`Id`**, **`_id`**, **`entityId`**, or **`id`** from the row in **`results`** — see scan section |
| **`solverId`** does not match **`issueId`** | 400 `E_SOLVER_ISSUE_MISMATCH` | Call **`GET /tp/api/issues/:issueId/solvers`** first |
| Polling every second | Slow scans, noisy traffic | Wait **`pollAfterSeconds`**, then backoff — 5→10→20→30s max |
| Treating **`needsInput`** as a failure | Apply never completes | HTTP **200** with **`data.needsInput`**: add **`userInput`** and **POST apply again** |
| Ignoring **400** responses on apply | Solver requirements not met | Read **`error.code`** and **`message`**, then supply **`userInput`** or fix the payload and retry |

## What we find

When you run a scan, CleanupOwl checks the connected QuickBooks Online company against a set of detectors. Each detector looks for one specific type of bookkeeping problem. The table below lists every detector and what it checks for.

### Duplicate transactions

| Detector | What it finds |
|----------|---------------|
| **Duplicate Payment in Undeposited Funds** | Receive Payment entries sitting in Undeposited Funds with the same customer, amount, and date (±1 day). Often caused by POS sync issues or double-entry mistakes. |
| **Duplicate Bank Feed Transactions (Same Date, Amount, Account)** | Categorized bank feed transactions that share the same date, amount, and bank account. The later-created transaction in each group is flagged as the auto-delete candidate. |
| **Duplicate Bank Feed Transactions (Date/Amount/Account in Review)** | Transactions in the For Review stage that share the same date, amount, and account. You choose which one to keep; the rest are excluded. |
| **Duplicate Bank Feed Transactions (Memo Match in Review)** | Transactions in the For Review stage that share the same memo or description. You choose which one to keep; the rest are excluded. |
| **Duplicate Bank Feed Transactions Based on Memo** | Categorized bank feed transactions grouped by matching memo within the same account and direction (money in vs. out). Groups of two or more are flagged as potential duplicates. |
| **Merge Duplicate Customers** | Customers with very similar names and matching contact details (email, address, or phone) that are likely the same person entered twice. |

### Undeposited funds & cash handling

| Detector | What it finds |
|----------|---------------|
| **Undeposited Funds Balance & Bypassed Deposits** | Two issues in one: payments posted directly to a bank account bypassing Undeposited Funds, and items stuck in Undeposited Funds that were never deposited. |
| **Old Items in Undeposited Funds** | Customer payments and sales receipts posted to Undeposited Funds that were never moved to a bank account. |
| **Cash Received but Not Deposited** | Cash payments and sales receipts (excluding cheques) in Undeposited Funds with no matching bank deposit. |
| **Payments Deposited Directly to Bank (Bypassing Undeposited Funds)** | Receive Payments where the Deposit To field was set to a bank account instead of Undeposited Funds while Undeposited Funds still carries a balance. Can cause reconciliation issues. |
| **Cheque Received but Not Deposited** | Cheque payments in Undeposited Funds that have not been deposited to a bank account. |

### Vendor & bill management

| Detector | What it finds |
|----------|---------------|
| **Vendor Payments Not Applied to Bills** | Vendor payments that exist but have not been linked to any open bill, leaving the bill balance unpaid and the payment floating. |
| **Vendor Payments to Expenses When No Open Bills Exist** | Payments recorded as expenses for a vendor that has no open bills, which can distort vendor balances and accounts payable. |
| **Aged Unapplied Vendor Credits and Debits** | Vendor credits that have never been applied to a bill or payment, meaning your business is leaving money on the table. |

### Customer & receivables

| Detector | What it finds |
|----------|---------------|
| **Unapplied Customer Credits** | Customer credits that have never been applied to an invoice or refunded, causing receivable balances to appear higher than they are. |
| **Open Customer Receivables** | Customers with outstanding invoice balances where payment has not been received. |
| **Sales Receipt Used Instead of Receive Payment** | Sales Receipts created for customers who already have open invoices. This records duplicate revenue and leaves the original invoice unpaid. |
| **Sales Recorded Without Invoice** | Revenue transactions (Sales Receipts, Deposits, Journal Entries) that credit income accounts without a corresponding invoice to the customer. |

### Inventory

| Detector | What it finds |
|----------|---------------|
| **Sales Before Purchase — Negative Inventory** | Items sold before they were purchased, resulting in negative inventory quantities that make financial reports unreliable. |
| **Wrong Item Selection — Negative Inventory** | Transactions where the wrong inventory item was selected, causing one item to go negative while another is overstated. |
| **Inventory Purchases Booked to Expense** | Inventory purchases recorded as expenses instead of assets, understating inventory value and overstating expenses. |
| **Missing Inventory Opening Balance** | Inventory items with no opening balance set, meaning the starting quantity and value are unknown. |

### Expense & income classification

| Detector | What it finds |
|----------|---------------|
| **Incorrect Expense Classification** | Expenses booked to the wrong category—for example, legal fees recorded as travel. The amount is correct but it appears on the wrong line of the Profit & Loss. |
| **Sales Tax Payment Recorded as Expense** | Sales tax payments to a tax agency booked as a P&L expense instead of clearing the Sales Tax Payable liability account. Overstates expenses and leaves the liability balance wrong. |
| **Personal Expenses Charged to Business** | Personal purchases recorded in the business books, making it impossible to see the true cost of running the business. |
| **Personal Income Charged to Business** | Personal income recorded as business revenue, inflating business income and potentially causing incorrect tax filings. |

### Assets, liabilities & compliance

| Detector | What it finds |
|----------|---------------|
| **Depreciation Not Charged** | Fixed assets (equipment, vehicles, etc.) with no depreciation recorded, causing profit to be overstated and asset values on the balance sheet to be too high. |
| **Missing Opening Balances for Assets** | Asset accounts with no opening balance entered, leaving the balance sheet incomplete from the start date. |
| **Missing Opening Balances for Liabilities** | Liability accounts with no opening balance entered, making the financial snapshot incomplete. |
| **Contractor Payment Not Flagged for 1099** | Payments to contractors that are not marked for 1099 reporting, which can result in missed tax filing obligations. |

### Bank statement reconciliation

| Detector | What it finds |
|----------|---------------|
| **Missing Transactions from Bank Statement** | Transactions that appear in an uploaded bank or credit card statement but are absent from QBO. Requires an uploaded statement file. |
| **Bank Statement Transactions** | Displays all transactions from an uploaded bank statement so you can verify the file was parsed correctly before running reconciliation checks. |

### Onboarding

| Detector | What it finds |
|----------|---------------|
| **Onboarding Health Check Diagnostic** | A comprehensive diagnostic for new clients. Checks uncoded transaction backlogs, reconciliation status using activity patterns, and balance sheet sanity (unusual liability patterns, credit cards, payroll liabilities, retirement accounts). |

## 1. Issues

An **issue** is a category of bookkeeping problem, identified by a stable string **`id`** (for example `duplicate-bank-feed-date-amount`). The list endpoint returns **`id`**, **`name`**, and **`description`** only—no solver list.

**Endpoints:**
- [`GET /tp/api/issues`](#get-tpapiissues)

### `GET /tp/api/issues`

Returns the catalog of issue types your integration can scan for and solve against.

**Scope:** `scan`

#### Parameters

No path or query parameters.

#### Returns

On success, HTTP **200** with `success: true` and a `data` object. See **Response attributes**.

#### Response attributes

- **`issues`** (array of objects) — Issue summaries. Each element:
  - **`id`** (string) — Use as `issueId` on solve/apply and as keys under `results` on a completed scan.
  - **`name`** (string) — Short title.
  - **`description`** (string or array of string, optional) — Plain text or bullet-style strings.

#### Example response

```json
{
  "success": true,
  "data": {
    "issues": [
      {
        "id": "cheque-not-deposited",
        "name": "Cheque Received but Not Deposited",
        "description": "Identifies cheque payments that were posted to Undeposited Funds but have not been deposited to a bank account."
      },
      {
        "id": "contractor-payment-not-flagged-for-1099",
        "name": "Contractor Payment Not Flagged for 1099",
        "description": [
          "First paragraph explaining the issue.",
          "Second paragraph with impact or how to fix it."
        ]
      }
    ]
  }
}
```

#### Request sample

```bash
curl -sS "$TP_BASE_URL/tp/api/issues" \
  -H "Authorization: Bearer $TP_API_KEY"
```

#### Error codes

| `error.code` | HTTP | Meaning |
|--------------|------|---------|
| `E_ISSUES_LOAD_FAILED` | 500 | Issue definitions could not be loaded |

---

## 2. Scans

A **scan** pulls QuickBooks Online (QBO) data when sync is not skipped, runs issue checks, and stores **results** by **`issueId`**. **Scans run in the background:** `POST /tp/api/scan/start` comes back right away; then poll [`GET /tp/api/scan/:scanId`](#get-tpapiscanscanid) until `data.scan.status` is `completed`, or handle `failed` / still running. A full company can take **minutes**—see [step 5 in Quick start](#5-poll-until-the-scan-finishes) for the polling flow.

**Endpoints:**
- [`POST /tp/api/scan/start`](#post-tpapiscanstart)
- [`GET /tp/api/scan/:scanId`](#get-tpapiscanscanid)
- [`GET /tp/api/scan/history`](#get-tpapiscanhistory)

### `POST /tp/api/scan/start`

Starts a background scan for one company.

**Scope:** `scan`

#### Parameters

JSON body:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `companyId` | `string` | **Yes** | Same as connection **`_id`** from [`GET /tp/api/connections`](#get-tpapiconnections) |
| `skipSync` | `boolean` | No | If `true`, skip QuickBooks Online sync before scan; default **`false`** |
| `period` | `object` | No | Date range for analysis |
| `period.startDate` | `string` | No | ISO date, e.g. `"2024-01-01"` |
| `period.endDate` | `string` | No | ISO date |
| `reportYear` | `number` | No | Focus year |
| `materialityThreshold` | `number` | No | Minimum amount to flag |
| `skip` | `string[]` | No | Issue ids to skip |
| `only` | `string[]` | No | Only run these issue ids |
| `overrides` | `object` | No | Per-detector option overrides |
| `syncPeriod` | `object` | No | Sync window for QBO pull |

#### Returns

On success, HTTP **200** with `success: true` and `data` containing the new scan id and polling hints.

#### Response attributes

| Field | Type | Description |
|-------|------|-------------|
| `scanId` | `string` | Use in `GET /tp/api/scan/:scanId` and solve endpoints |
| `status` | `string` | e.g. `"started"` |
| `pollAfterSeconds` | `number` | Minimum suggested delay before the first poll (often **`300`**) |
| `pollUrl` | `string` | Relative URL including `companyId` query for convenience |

#### Example request

```json
{
  "companyId": "YOUR_COMPANY_DOCUMENT_ID",
  "period": {
    "startDate": "2024-01-01",
    "endDate": "2024-12-31"
  }
}
```

#### Example response

```json
{
  "success": true,
  "data": {
    "scanId": "65f1a2b3c4d5e6f7a8b9c0d1",
    "status": "started",
    "pollAfterSeconds": 300,
    "pollUrl": "/tp/api/scan/65f1a2b3c4d5e6f7a8b9c0d1?companyId=YOUR_COMPANY_DOCUMENT_ID"
  }
}
```

#### Request sample

```bash
curl -sS -X POST "$TP_BASE_URL/tp/api/scan/start" \
  -H "Authorization: Bearer $TP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"companyId":"YOUR_COMPANY_DOCUMENT_ID"}'
```

#### Error codes

| `error.code` | HTTP | Meaning |
|--------------|------|---------|
| `E_MISSING_COMPANY_ID` | 400 | Missing `companyId` |
| `E_COMPANY_ACCESS_DENIED` | 403 | No access to company |
| `E_QBO_AUTH_FAILED` | 401 | QBO session invalid |

---

### `GET /tp/api/scan/:scanId`

Gets **one scan**. You see **`results`** when the scan is done. Put the **`scanId`** from `POST /tp/api/scan/start` in the URL. To **list past scans**, use **`history`** where **`scanId`** would go—or call [`GET /tp/api/scan/history`](#get-tpapiscanhistory).

**Scope:** `scan`

#### Parameters

**Path**

| Parameter | Type | Description |
|-----------|------|-------------|
| `scanId` | `string` | The scan id, or **`history`** to list scans |

**Query**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `companyId` | `string` | **Yes** | Company id |
| `limit` | `number` | No | For **`history`** lists: page size |
| `skip` | `number` | No | For **`history`** lists: offset |

#### Returns

On success, HTTP **200** with `success: true` and `data.scan`. When **`data.scan.status`** is **`completed`**, read **`data.scan.results`** (object keyed by **`issueId`**). For [`POST /tp/api/solve/apply`](#post-tpapisolveapply) you need **`scanId`**, **`companyId`**, **`issueId`**, and **`recordId`** from a row in the matching bucket.

#### Response attributes

**`data.scan`** (object) — selected fields:

| Field | Type | Description |
|-------|------|-------------|
| `_id` | `string` | Scan id |
| `companyId` | `string` | Internal company id |
| `externalId` | `string` | QBO realm / company id |
| `startedAt` | `string` | ISO timestamp |
| `completedAt` | `string` or `null` | When the scan finished |
| `status` | `string` | e.g. `pending`, `running`, `completed`, `failed` |
| `issueIds` | `string[]` | For **history**: every issue in the job. For **single scan**: issues that have rows in **`results`** |
| `totalResults` | `number` | Total flagged rows |
| `severity` | `object` | Counts with **`high`**, **`medium`**, **`low`** |
| `executionTimeMs` | `number` | Optional |
| `error` | `any` | Optional failure payload |
| `isLatest` | `boolean` | Whether this is the latest completed full scan for the company |
| `results` | `object` | Map **issue id → per-issue result** (see below) |

Issue buckets with **`count: 0`** are **omitted** from **`results`**. Only the fields in this section are treated as a stable integration contract; other properties may appear for the product UI—**ignore** undocumented fields.

**`results[issueId]`** (object):

| Field | Type | Description |
|-------|------|-------------|
| `category` | `string` | Optional grouping |
| `label` | `string` | Human-readable title |
| `description` | `string` or `string[]` | Optional |
| `message` | `string` | Optional |
| `count` | `number` | Flagged records |
| `severity` | `object` | Optional per-bucket breakdown |
| `params` | `object` | Optional run parameters |
| `meta` | `object` | Extra context (totals, solver options, etc.) |
| `executionTimeMs` | `number` | Optional |
| `records` | `array` | List-shaped rows; see **Identifying a row** |
| `table` | `object` | Tabular shape |
| `table.columns` | `array` | `{ key, label, format?, width? }` |
| `table.rows` | `array` | Rows with **`entityId`**, **`entityType`**, **`_id`** when present, **`fixOptions`** |

#### Identifying a row for `POST /tp/api/solve/apply`

Set **`recordId`** to a string equal to one of **`Id`**, **`_id`**, **`entityId`**, or **`id`** on the target row. For **table**-shaped issues, use **`entityId`** or row **`_id`** on **`table.rows`**. For **list**-shaped issues (**`records`**), use **`Id`** or **`entityId`**. Leading-underscore fields are primarily for the app UI.

#### Example response

```json
{
  "success": true,
  "data": {
    "scan": {
      "_id": "65f1a2b3c4d5e6f7a8b9c0d1",
      "companyId": "YOUR_COMPANY_DOCUMENT_ID",
      "externalId": "9341455852565781",
      "startedAt": "2026-03-31T09:17:27.328Z",
      "completedAt": "2026-03-31T09:18:48.518Z",
      "status": "completed",
      "issueIds": ["open-customer-receivables", "inventory-purchases-booked-to-expense"],
      "totalResults": 23,
      "severity": { "high": 19, "medium": 3, "low": 1 },
      "isLatest": true,
      "results": {
        "inventory-purchases-booked-to-expense": {
          "category": "Inventory",
          "label": "Inventory Purchases Booked to Expense",
          "description": ["…"],
          "count": 4,
          "severity": { "high": 0, "medium": 1, "low": 3 },
          "table": {
            "columns": [
              { "key": "txnType", "label": "Transaction Type", "format": null },
              { "key": "purchaseAmount", "label": "Amount", "format": "currency" }
            ],
            "rows": [
              {
                "txnType": "Bill",
                "purchaseAmount": "$830.09",
                "_id": "3871",
                "_entity": "bill",
                "entityId": "3871",
                "entityType": "bill"
              }
            ]
          },
          "params": { "aiConfidenceThreshold": 0.8 },
          "meta": { "totalFlagged": 4 }
        }
      }
    }
  }
}
```

#### Request sample

```bash
SCAN_ID="65f1a2b3c4d5e6f7a8b9c0d1"
COMPANY_ID="YOUR_COMPANY_DOCUMENT_ID"

curl -sS "$TP_BASE_URL/tp/api/scan/$SCAN_ID?companyId=$COMPANY_ID" \
  -H "Authorization: Bearer $TP_API_KEY"
```

#### Error codes

| `error.code` | HTTP | Meaning |
|--------------|------|---------|
| `E_MISSING_COMPANY_ID` | 400 | Missing `companyId` query |
| `E_COMPANY_ACCESS_DENIED` | 403 | No access |
| `E_SCAN_NOT_FOUND` | 404 | Unknown scan id, or scan does not belong to the company for this **`companyId`** |

---

### `GET /tp/api/scan/history`

Lists recent scans for one company. The URL is **`GET /tp/api/scan/history`**—same idea as get scan, but the path uses **`history`** instead of a scan id.

**Scope:** `scan`

#### Parameters

**Query**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `companyId` | `string` | **Yes** | — | Company id |
| `limit` | `number` | No | `20` | Page size |
| `skip` | `number` | No | `0` | Offset |

#### Returns

On success, HTTP **200** with `success: true` and `data.scans` plus `data.latestScanId`.

#### Response attributes

| Field | Type | Description |
|-------|------|-------------|
| `scans` | `array` | Summary rows |
| `latestScanId` | `string` | Latest completed scan id in this list window |

Each **`scans[]`** element:

| Field | Type | Description |
|-------|------|-------------|
| `_id` | `string` | Scan id |
| `startedAt` | `string` | ISO |
| `completedAt` | `string` or `null` | ISO |
| `status` | `string` | Scan status |
| `issueIds` | `string[]` | Issues that ran in this job |
| `totalResults` | `number` | Total flagged rows |
| `severity` | `object` | Optional |
| `executionTimeMs` | `number` | Optional |
| `error` | `any` | Optional |
| `isLatest` | `boolean` | Whether this row is the latest completed scan in the list |

#### Example response

```json
{
  "success": true,
  "data": {
    "scans": [
      {
        "_id": "65f1a2b3c4d5e6f7a8b9c0d1",
        "startedAt": "2026-03-31T09:17:27.328Z",
        "completedAt": "2026-03-31T09:18:48.518Z",
        "status": "completed",
        "issueIds": ["open-customer-receivables", "duplicate-bank-feed-date-amount"],
        "totalResults": 45,
        "isLatest": true,
        "error": null,
        "executionTimeMs": 24132,
        "severity": { "high": 32, "medium": 7, "low": 6 }
      }
    ],
    "latestScanId": "65f1a2b3c4d5e6f7a8b9c0d1"
  }
}
```

#### Request sample

```bash
curl -sS "$TP_BASE_URL/tp/api/scan/history?companyId=YOUR_COMPANY_DOCUMENT_ID&limit=10&skip=0" \
  -H "Authorization: Bearer $TP_API_KEY"
```

## 3. Solvers

**Solvers** change data in QuickBooks Online (QBO) to fix a single **record** surfaced by a **scan**. Always inspect **`requiresInput`** and **`userInputSchema`** before calling apply.

> A solver only fixes the issues it is built for. The wrong pairing returns **`E_SOLVER_ISSUE_MISMATCH`**. Discover valid solvers with [`GET /tp/api/issues/:issueId/solvers`](#get-tpapiissuesissueidsolvers) or [`GET /tp/api/solvers`](#get-tpapisolvers).

**Endpoints:**
- [`GET /tp/api/solvers`](#get-tpapisolvers)
- [`GET /tp/api/issues/:issueId/solvers`](#get-tpapiissuesissueidsolvers)
- [`POST /tp/api/solve/prep`](#post-tpapisolveprep)
- [`POST /tp/api/solve/apply`](#post-tpapisolveapply)
- [`GET /tp/api/scans/:scanId/solves`](#get-tpapiscansscanidsolves)

### `GET /tp/api/solvers`

Returns every registered solver and the issue types each one supports.

**Scope:** `solve`

#### Parameters

No path or query parameters.

#### Returns

On success, HTTP **200** with `success: true` and `data.solvers`.

#### Response attributes

**`data.solvers`** (array). Each element:

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | Pass as `solverId` on apply when required |
| `name` | `string` | Display name |
| `description` | `string` | Optional |
| `forIssue` | `string`, `string[]`, or `"*"` | Issue id(s) this solver handles, or universal |
| `requiresInput` | `boolean` | If `true`, expect `userInput` or a `needsInput` round-trip |
| `userInputSchema` | `object` or `null` | Field specs: `select`, `text`, `number`, `hidden`; use **`optionsFrom`** when options come from paths on the scan record |

#### Example response

```json
{
  "success": true,
  "data": {
    "solvers": [
      {
        "id": "reclassify-expense-to-inventory",
        "name": "Reclassify to Inventory Item",
        "description": "Reclassify an expense line to an inventory item or inventory asset account.",
        "requiresInput": true,
        "userInputSchema": {
          "targetItemId": {
            "type": "select",
            "label": "Select Inventory Item",
            "required": false,
            "optionsFrom": "record.fixOptions.inventoryItems"
          }
        },
        "forIssue": "inventory-purchases-booked-to-expense"
      }
    ]
  }
}
```

#### Request sample

```bash
curl -sS "$TP_BASE_URL/tp/api/solvers" \
  -H "Authorization: Bearer $TP_API_KEY"
```

---

### `GET /tp/api/issues/:issueId/solvers`

Returns solvers registered for one issue id.

**Scope:** `solve`

#### Parameters

**Path**

| Parameter | Type | Description |
|-----------|------|-------------|
| `issueId` | `string` | Same as **`id`** from [`GET /tp/api/issues`](#get-tpapiissues) |

#### Returns

On success, HTTP **200** with `success: true`, `data.solvers`, and `data.hasSolvers`.

#### Response attributes

| Field | Type | Description |
|-------|------|-------------|
| `solvers` | `array` | Same element shape as [`GET /tp/api/solvers`](#get-tpapisolvers) |
| `hasSolvers` | `boolean` | `false` when no solvers exist for this issue |

#### Example response

```json
{
  "success": true,
  "data": {
    "solvers": [
      {
        "id": "reclassify-expense-to-inventory",
        "name": "Reclassify to Inventory Item",
        "requiresInput": true,
        "userInputSchema": { },
        "forIssue": "inventory-purchases-booked-to-expense"
      }
    ],
    "hasSolvers": true
  }
}
```

#### Request sample

```bash
curl -sS "$TP_BASE_URL/tp/api/issues/inventory-purchases-booked-to-expense/solvers" \
  -H "Authorization: Bearer $TP_API_KEY"
```

---

### `POST /tp/api/solve/prep`

Returns solver metadata, row-level options, and risk flags for **one** scan row—everything you need to choose **`solverId`** and build **`userInput`** before [`POST /tp/api/solve/apply`](#post-tpapisolveapply). This mirrors the internal **`get_fixing_prep_data`** tool shape.

**Scope:** `solve`

#### Parameters

JSON body:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `scanId` | `string` | **Yes** | Scan that contains the record |
| `issueId` | `string` | **Yes** | Issue bucket id |
| `recordId` | `string` | **Yes** | Row id—see [Identifying a row](#identifying-a-row-for-post-tpapisolveapply) |
| `companyId` | `string` | **Yes** | Company id |

#### Returns

On success, HTTP **200** with `success: true` and the fields below.

#### Response attributes

| Field | Type | Description |
|-------|------|-------------|
| `issueId` | `string` | Echo of request **`issueId`** |
| `scanId` | `string` | Scan id |
| `hasSolvers` | `boolean` | `false` when no solver is registered for this issue |
| `message` | `string` | Present when **`hasSolvers`** is `false` |
| `solvers` | `array` | When **`hasSolvers`** is `true`, same shape as [`GET /tp/api/issues/:issueId/solvers`](#get-tpapiissuesissueidsolvers) |
| `recordFixOptions` | `object` or `null` | Row-level options (bills, suggested ids, etc.) |
| `metaFixOptions` | `object` or `null` | Issue- or scan-level options (accounts, items, bank accounts, …) |
| `hasLinkedTransactions` | `boolean` | QBO links (e.g. payment ↔ invoice) |
| `linkedTransactions` | `array` | When linked: `{ txnType, txnId }` entries |
| `resolvedByOtherFix` | `boolean` | Do not apply; already resolved elsewhere |
| `resolvedBy` | `object` or `null` | Context when **`resolvedByOtherFix`** |
| `needsRevalidation` | `boolean` | QBO data changed; run a new scan before applying |
| `modifiedBy` | `object` | Present when **`needsRevalidation`** |
| `entityConflicts` | `object` or `null` | Same entity flagged under multiple issues |

#### Request sample

```bash
curl -sS -X POST "$TP_BASE_URL/tp/api/solve/prep" \
  -H "Authorization: Bearer $TP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "scanId": "65f1a2b3c4d5e6f7a8b9c0d1",
    "issueId": "inventory-purchases-booked-to-expense",
    "recordId": "3871",
    "companyId": "YOUR_COMPANY_DOCUMENT_ID"
  }'
```

#### Error codes

| `error.code` | HTTP | Meaning |
|--------------|------|---------|
| `E_MISSING_PARAMS` | 400 | Missing `scanId`, `issueId`, or `recordId` |
| `E_MISSING_COMPANY_ID` | 400 | Missing `companyId` |
| `E_SCAN_NOT_FOUND` | 404 | Unknown scan |
| `E_RECORD_NOT_FOUND` | 404 | Row not in this issue’s results for the scan |
| `E_DETECTOR_NOT_FOUND` | 404 | No results bucket for this **`issueId`** on the scan |
| `E_COMPANY_ACCESS_DENIED` | 403 | No access to company |

---

### `POST /tp/api/solve/apply`

**Runs a solver on one row** from a finished scan. Use it after [`POST /tp/api/solve/prep`](#post-tpapisolveprep), or when you already know the solver and **`userInput`**. CleanupOwl applies the fix in QuickBooks Online (QBO). The response says what changed or was skipped. If more fields are needed, you still get **HTTP 200**, **`success: true`**, and **`data.needsInput`**—**POST again** with the same ids plus **`userInput`**.

**Scope:** `solve`

#### Parameters

JSON body:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `scanId` | `string` | **Yes** | Scan containing the record |
| `issueId` | `string` | **Yes** | Issue bucket id |
| `recordId` | `string` | **Yes** | Row id—see [Identifying a row](#identifying-a-row-for-post-tpapisolveapply) |
| `companyId` | `string` | **Yes** | Company id |
| `solverId` | `string` | No | Solver to run; a default applies if omitted |
| `userInput` | `object` | No | Values matching `userInputSchema` |
| `externalId` | `string` | No | Optional; **`companyId`** is enough to resolve the company |

#### Returns

On success, HTTP **200** with `success: true` and solver-specific **`data`** (see **Response attributes**). When more fields are required, **`data.needsInput`** is **`true`** instead—see **Needs more input response**.

#### Response attributes

Typical success **`data`** fields:

| Field | Type | Description |
|-------|------|-------------|
| `solverId` | `string` | Solver that ran |
| `solverName` | `string` | Display name |
| `issueId` | `string` | Issue id |
| `action` | `string` | e.g. `deleted`, `updated`, `skip` |
| `message` | `string` | Human-readable outcome |
| `success` | `boolean` | Operation outcome inside `data` |

#### Example request

```json
{
  "scanId": "65f1a2b3c4d5e6f7a8b9c0d1",
  "issueId": "inventory-purchases-booked-to-expense",
  "recordId": "3871",
  "solverId": "reclassify-expense-to-inventory",
  "companyId": "YOUR_COMPANY_DOCUMENT_ID",
  "userInput": {
    "targetItemId": "235",
    "quantity": 1
  }
}
```

#### Example response

```json
{
  "success": true,
  "data": {
    "solverId": "reclassify-expense-to-inventory",
    "solverName": "Reclassify to Inventory Item",
    "issueId": "inventory-purchases-booked-to-expense",
    "action": "updated",
    "message": "Expense reclassified to inventory"
  }
}
```

#### Needs more input response

```json
{
  "success": true,
  "data": {
    "needsInput": true,
    "missingFields": ["targetAccountId"],
    "userInputSchema": {
      "targetAccountId": {
        "type": "select",
        "label": "Select New Account",
        "required": true,
        "optionsFrom": "record.fixOptions.targetAccounts"
      }
    }
  }
}
```

Send **`userInput`** with the requested fields and **POST** again with the same **`scanId`**, **`issueId`**, **`recordId`**, **`companyId`**, and **`solverId`**.

#### Request sample

```bash
curl -sS -X POST "$TP_BASE_URL/tp/api/solve/apply" \
  -H "Authorization: Bearer $TP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "scanId": "65f1a2b3c4d5e6f7a8b9c0d1",
    "issueId": "inventory-purchases-booked-to-expense",
    "recordId": "3871",
    "companyId": "YOUR_COMPANY_DOCUMENT_ID",
    "solverId": "reclassify-expense-to-inventory"
  }'
```

#### Error codes

| `error.code` | HTTP | Meaning |
|--------------|------|---------|
| `E_MISSING_PARAMS` | 400 | Missing `scanId`, `issueId`, or `recordId` |
| `E_MISSING_COMPANY_ID` | 400 | Missing `companyId` |
| `E_SOLVER_ISSUE_MISMATCH` | 400 | Solver not valid for this issue |
| *(solver validation)* | 400 | Preconditions not met (for example required **`userInput`**) — **`success: false`** with **`error`** |
| `E_SCAN_NOT_FOUND` | 404 | Scan not found |
| `E_RECORD_NOT_FOUND` | 404 | No matching row in scan results |
| `E_SCAN_ARCHIVED` | 400 | Scan archived; run a new scan |
| `E_ENTITY_FROZEN` | 400 | Record already resolved |
| `E_ENTITY_MODIFIED` | 400 | QBO data changed; re-scan |
| `E_FIX_LIMIT_EXCEEDED` | 402 | Plan fix limit reached |
| `E_QBO_AUTH_FAILED` | 401 | Reconnect QuickBooks Online (QBO) |
| `E_COMPANY_ACCESS_DENIED` | 403 | No access to company |

---

### `GET /tp/api/scans/:scanId/solves`

Lists recent solve attempts for a scan (newest first, capped server-side).

**Scope:** `solve`

#### Parameters

**Path**

| Parameter | Type | Description |
|-----------|------|-------------|
| `scanId` | `string` | Scan id |

**Query**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `companyId` | `string` | **Yes** | Company id |
| `issueId` | `string` | No | Filter to one issue |

#### Returns

On success, HTTP **200** with `success: true` and `data.solves`.

#### Response attributes

**`data.solves`** (array). Each entry includes at least:

| Field | Type | Description |
|-------|------|-------------|
| `_id` | `string` | Log id |
| `scanId` | `string` | Scan |
| `recordId` | `string` | Row |
| `issueId` | `string` | Issue |
| `solverId` | `string` or `null` | Solver used |
| `solverName` | `string` or `null` | Display name |
| `action` | `string` | e.g. `skip`, `updated` |
| `success` | `boolean` | Whether the fix succeeded |
| `executedAt` | `string` | ISO timestamp |
| `entity` | `object` | Optional `{ entityType, entityId }` |

Other properties may appear; **integrate only against the fields above**.

#### Example response

```json
{
  "success": true,
  "data": {
    "solves": [
      {
        "_id": "65f1a2b3c4d5e6f7a8b9c0d2",
        "scanId": "65f1a2b3c4d5e6f7a8b9c0d1",
        "recordId": "3699",
        "issueId": "vendor-expenses-without-open-bills",
        "action": "skip",
        "success": true,
        "executedAt": "2026-03-31T09:33:16.025Z",
        "solverId": null,
        "solverName": null,
        "entity": { "entityType": "purchase", "entityId": "3699" }
      }
    ]
  }
}
```

#### Request sample

```bash
curl -sS "$TP_BASE_URL/tp/api/scans/$SCAN_ID/solves?companyId=$COMPANY_ID" \
  -H "Authorization: Bearer $TP_API_KEY"
```

### Applying solves

**Recommended:** call [`POST /tp/api/solve/prep`](#post-tpapisolveprep) with the same **`scanId`**, **`companyId`**, **`issueId`**, and **`recordId`** you will use for apply. Use **`data.solvers`**, **`recordFixOptions`**, **`metaFixOptions`**, and the risk flags to build **`userInput`** and decide whether apply is safe.

**Lightweight path:** [`GET /tp/api/issues/:issueId/solvers`](#get-tpapiissuesissueidsolvers) or [`GET /tp/api/solvers`](#get-tpapisolvers) only exposes schemas—they omit row-level **`recordFixOptions`** and **`metaFixOptions`**.

Call [`POST /tp/api/solve/apply`](#post-tpapisolveapply) with **`scanId`**, **`companyId`**, **`issueId`**, **`recordId`**, optional **`solverId`**, and optional **`userInput`**. If **`data.needsInput`** is true, add **`userInput`** and POST apply again.

---

## 4. Billing

**Endpoints:**
- [`GET /tp/api/billing`](#get-tpapibilling)

### `GET /tp/api/billing`

Returns plan name, limits, and usage counters for the authenticated account.

**Scope:** none (valid API key only)

#### Parameters

No path or query parameters.

#### Returns

On success, HTTP **200** with `success: true` and `data` as below.

#### Response attributes

| Field | Type | Description |
|-------|------|-------------|
| `plan` | `object` | Current plan |
| `plan.name` | `string` | Display name |
| `plan.limits` | `object` | e.g. `{ "maxCompanies": 2 }` |
| `usage` | `object` | Usage counters |
| `usage.fixesUsed` | `number` | Fixes used in the billing period |
| `usage.fixesLimit` | `number` | Monthly cap, or **`-1`** for unlimited |
| `usage.fixesRemaining` | `number` | Remaining fixes, or **`-1`** if unlimited |
| `usage.canRunScan` | `boolean` | Whether scans are allowed on the plan |
| `upgrade` | `object` | Upgrade hint |
| `upgrade.available` | `boolean` | Whether a higher tier exists |
| `upgrade.nextPlan` | `object` or `null` | `{ id, name }` |
| `upgrade.message` | `string` or `null` | Prompt text |

Use **`plan.name`** and **`plan.limits`** in billing UI. **`plan.id`** is not always returned.

#### Example response

```json
{
  "success": true,
  "data": {
    "plan": {
      "name": "All-Access Pass",
      "limits": { "maxCompanies": 2 }
    },
    "usage": {
      "fixesUsed": 0,
      "fixesLimit": -1,
      "fixesRemaining": -1,
      "canRunScan": true
    },
    "upgrade": {
      "available": false,
      "nextPlan": null,
      "message": null
    }
  }
}
```

#### Error codes

| `error.code` | HTTP | Meaning |
|--------------|------|---------|
| `E_PAYMENTS_DISABLED` | 503 | Billing subsystem not configured in this deployment |

---

## 5. Connections

**Endpoints:**
- [`GET /tp/api/connections`](#get-tpapiconnections)
- [`GET /tp/api/connect`](#get-tpapiconnect)

### `GET /tp/api/connections`

Lists QuickBooks Online companies linked to the API key’s user. Use each object’s **`_id`** as **`companyId`** in scan and solve calls.

**Scope:** none

#### Parameters

No path or query parameters.

#### Returns

On success, HTTP **200** with `success: true` and `data.connections`.

#### Response attributes

**`data.connections`** (array of QuickBooks Online (QBO) connections). Each element:

| Field | Type | Description |
|-------|------|-------------|
| `_id` | `string` | **Use as `companyId`** |
| `externalId` | `string` | QBO company / realm id |
| `companyName` | `string` | Display name |
| `accountingSystem` | `string` | e.g. `qbo` |
| `isSandbox` | `boolean` | Sandbox realm |
| `status` | `string` | e.g. `active` |
| `syncStatus` | `string` | Sync pipeline status |
| `syncLocked` | `boolean` | Sync lock |
| `syncUpdatedAt` | `string` | ISO |
| `syncError` | `any` | Optional |
| `createdAt` | `string` | ISO |
| `lastAccessedAt` | `string` | ISO |

#### Example response

```json
{
  "success": true,
  "data": {
    "connections": [
      {
        "_id": "69b2df85dc1974c5e28656ae",
        "externalId": "9341455852565781",
        "accountingSystem": "qbo",
        "companyName": "Example Co.",
        "isSandbox": true,
        "status": "active",
        "syncStatus": "ready",
        "syncLocked": false,
        "syncUpdatedAt": "2026-03-31T06:31:25.689Z",
        "syncError": null,
        "createdAt": "2026-03-12T15:45:09.699Z",
        "lastAccessedAt": "2026-03-26T08:41:58.959Z"
      }
    ]
  }
}
```

#### Request sample

```bash
curl -sS "$TP_BASE_URL/tp/api/connections" \
  -H "Authorization: Bearer $TP_API_KEY"
```

---

### `GET /tp/api/connect`

Returns a **browser URL** that starts QuickBooks Online OAuth. The user must already be **signed in** to CleanupOwl in that browser session.

**Scope:** none

#### Parameters

No path or query parameters.

#### Returns

On success, HTTP **200** with `success: true` and `data.connectionUrl` / `data.instructions`.

#### Response attributes

| Field | Type | Description |
|-------|------|-------------|
| `connectionUrl` | `string` | Absolute URL to open |
| `instructions` | `string` | Human-readable steps |

#### Example response

```json
{
  "success": true,
  "data": {
    "connectionUrl": "https://app.cleanupowl.com/auth/qbo/init?returnTo=/v2/scans",
    "instructions": "Open this URL in your browser to connect your QuickBooks Online company. You must be logged in to CleanupOwl."
  }
}
```

## Postman collection

The Postman collection gives you a runnable set of requests for every endpoint so you can test the API without writing any code first. It includes pre-configured paths and collection variables—fill in your values and run.

**Collection file:** [`cleanupowl-tp-api-postman-collection.json`](./cleanupowl-tp-api-postman-collection.json) in this repository root.

The collection name includes **Beta** in Postman.

**Suggested variables to set in your Postman environment:**

| Variable | What to put there |
|----------|-------------------|
| `baseUrl` | `https://app.cleanupowl.com` |
| `apiToken` | Your API key |
| `companyId` | `_id` from `GET /tp/api/connections` |
| `scanId` | `scanId` from `POST /tp/api/scan/start` |
| `issueId` | An issue id from the issues list |
| `recordId` | A row id from the scan results |
| `solverId` | A solver id from prep or the solvers list |

The collection does not duplicate schemas, error codes, or polling rules—use this reference for those details and Postman as a quick way to run requests.

## Companion files

These files complement this HTTP reference with additional context for specific use cases:

- **`THIRD_PARTY_API_AGENT_WORKFLOW.md`** — For external agents. Covers recommended call order, guardrails, and field mapping when building automated integrations.
- **`SKILL.md`** — For Cursor and IDE agents (`cleanupowl-automation`). Details when to use the API, prerequisites, connections, key concepts, and tool patterns.

## Contact Us

If you need help or think you have found a bug, reach out using the support template available online. We typically respond within 24 hours.

[support@cleanupowl.com](mailto:support@cleanupowl.com)
