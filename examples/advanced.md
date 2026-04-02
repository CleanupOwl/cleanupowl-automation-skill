# Advanced examples

Multi-step workflows beyond [basic.md](basic.md): **prep** and **apply** (solvers fix **one row at a time**), **`needsInput`**, targeted scans, and **`error.code`** handling—as in the [API reference](../api-reference.md).

Assume the same **`TP_BASE_URL`**, **`TP_API_KEY`**, and **`COMPANY_ID`** as basic examples, plus:

```bash
export SCAN_ID="paste-scanId-here"
export ISSUE_ID="issue-bucket-id"
export RECORD_ID="row-id-from-scan-results"
export SOLVER_ID="solver-id-from-prep"
```

**`solve`** scope is required for **`POST /tp/api/solve/prep`**, **`POST /tp/api/solve/apply`**, and **`GET /tp/api/scans/:scanId/solves`**.

---

## Prep → apply (full fix flow)

### 1. Prep the fix

**Scope:** `solve`

Before applying, call prep to get the available **solvers**, row-level options, and risk flags for that row—see [`POST /tp/api/solve/prep`](../api-reference.md#post-tpapisolveprep).

```bash
curl -sS -X POST "$TP_BASE_URL/tp/api/solve/prep" \
  -H "Authorization: Bearer $TP_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"scanId\": \"$SCAN_ID\",
    \"issueId\": \"$ISSUE_ID\",
    \"recordId\": \"$RECORD_ID\",
    \"companyId\": \"$COMPANY_ID\"
  }"
```

From the response, branch roughly like the quick start:

- **`data.solvers`** — choose a **`solverId`**
- **`data.recordFixOptions`** and **`data.metaFixOptions`** — use when building **`userInput`**
- **`data.resolvedByOtherFix: true`** — skip; already fixed elsewhere
- **`data.needsRevalidation: true`** — QBO data changed; run a new scan before applying
- **`data.entityConflicts`** — same entity flagged under multiple issues; resolve before applying
- **`data.hasSolvers: false`** — no solver for this issue; do not apply
- **`data.hasLinkedTransactions: true`** — linked QBO records (e.g. payment ↔ invoice); proceed carefully

### 2. Apply the fix

**Scope:** `solve`

This is the call that **writes to QuickBooks Online (QBO)**.

```bash
curl -sS -X POST "$TP_BASE_URL/tp/api/solve/apply" \
  -H "Authorization: Bearer $TP_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"scanId\": \"$SCAN_ID\",
    \"issueId\": \"$ISSUE_ID\",
    \"recordId\": \"$RECORD_ID\",
    \"companyId\": \"$COMPANY_ID\",
    \"solverId\": \"$SOLVER_ID\"
  }"
```

Read the response:

- **`success: true`** with **`data.action`** — fix applied; check **`data.message`**
- **`success: true`** with **`data.needsInput: true`** — still **HTTP 200**; add **`userInput`** from **`data.userInputSchema`** and **POST again** with the same ids
- **`success: false`** — use **`error.code`** and **`error.message`** (integrations should branch on **`error.code`**)

### 3. `needsInput` loop

When **`data.needsInput`** is **`true`**, supply the missing fields and call **`POST /tp/api/solve/apply`** again with the same **`scanId`**, **`issueId`**, **`recordId`**, **`companyId`**, and **`solverId`**.

```bash
curl -sS -X POST "$TP_BASE_URL/tp/api/solve/apply" \
  -H "Authorization: Bearer $TP_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"scanId\": \"$SCAN_ID\",
    \"issueId\": \"$ISSUE_ID\",
    \"recordId\": \"$RECORD_ID\",
    \"companyId\": \"$COMPANY_ID\",
    \"solverId\": \"$SOLVER_ID\",
    \"userInput\": {
      \"targetItemId\": \"235\",
      \"quantity\": 1
    }
  }"
```

Use **`data.missingFields`** and **`data.userInputSchema`** from the previous response when building **`userInput`**.

### 4. Review what was fixed

**Scope:** `solve`

Audit trail of solve attempts on the scan:

```bash
curl -sS "$TP_BASE_URL/tp/api/scans/$SCAN_ID/solves?companyId=$COMPANY_ID" \
  -H "Authorization: Bearer $TP_API_KEY"
```

Filter to one issue:

```bash
curl -sS "$TP_BASE_URL/tp/api/scans/$SCAN_ID/solves?companyId=$COMPANY_ID&issueId=$ISSUE_ID" \
  -H "Authorization: Bearer $TP_API_KEY"
```

Each entry in **`data.solves`** includes **`action`**, **`success`**, **`solverId`**, and **`executedAt`** for one attempt.

---

## Recovering from errors

Integrations should branch on **`error.code`**; use **`message`** for display and logging. Typical cases:

### `E_QBO_AUTH_FAILED`

```json
{ "success": false, "error": { "code": "E_QBO_AUTH_FAILED" } }
```

**Meaning:** QuickBooks Online connection is invalid (often **401**). **Action:** call **`GET /tp/api/connect`**, open **`data.connectionUrl`**, complete authorization, then retry.

### `E_FIX_LIMIT_EXCEEDED`

```json
{ "success": false, "error": { "code": "E_FIX_LIMIT_EXCEEDED" } }
```

**Meaning:** plan fix limit reached (often **402**). **Action:** call **`GET /tp/api/billing`** for **`usage.fixesRemaining`** and upgrade if needed.

### `E_ENTITY_MODIFIED`

```json
{ "success": false, "error": { "code": "E_ENTITY_MODIFIED" } }
```

**Meaning:** QBO data changed since the scan. **Action:** start a new scan and retry with the new **`scanId`**.

### `E_SCAN_ARCHIVED`

```json
{ "success": false, "error": { "code": "E_SCAN_ARCHIVED" } }
```

**Meaning:** scan is no longer usable for apply. **Action:** run **`POST /tp/api/scan/start`** again.

---

## Targeted scan (`only` / `skip`)

**Scope:** `scan`

Restrict the scan to specific **issue ids**, or skip some—see [`POST /tp/api/scan/start`](../api-reference.md#post-tpapiscanstart).

`only` — only run these detectors:

```bash
curl -sS -X POST "$TP_BASE_URL/tp/api/scan/start" \
  -H "Authorization: Bearer $TP_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"companyId\": \"$COMPANY_ID\",
    \"only\": [\"inventory-purchases-booked-to-expense\", \"duplicate-bank-feed-date-amount\"],
    \"period\": {
      \"startDate\": \"2024-01-01\",
      \"endDate\": \"2024-12-31\"
    }
  }"
```

`skip` — omit issue ids; optional **`skipSync`**: skip QBO sync before the scan when **`true`**:

```bash
curl -sS -X POST "$TP_BASE_URL/tp/api/scan/start" \
  -H "Authorization: Bearer $TP_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"companyId\": \"$COMPANY_ID\",
    \"skip\": [\"low-priority-issue-id\"],
    \"skipSync\": true
  }"
```
