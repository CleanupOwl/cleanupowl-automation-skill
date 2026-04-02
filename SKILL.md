---
name: cleanupowl-automation
description: Scans QuickBooks Online books via the CleanupOwl API to detect and fix bookkeeping issues. Use when the user mentions CleanupOwl, QuickBooks cleanup, QBO bookkeeping scan, fixing bookkeeping errors, running accounting scans, or applying solver fixes to flagged records.
---

# CleanupOwl Automation

## Purpose

This skill enables an agent to **automatically clean and maintain QuickBooks books** using the CleanupOwl API.

It allows you to:
- Run bookkeeping **scans**
- Analyze detected **issues**
- Apply **fixes (solvers)** to specific records
- Maintain books via **repeatable workflows**

Overview of where things live: the public site is [cleanupowl.com](https://cleanupowl.com); the signed-in web app and HTTP API host are [app.cleanupowl.com](https://app.cleanupowl.com), with routes under **`/tp/api/`** (see [API Reference](./api-reference.md#base-url)).

---

## When to Use This Skill

### Prerequisites

Before using the CleanupOwl API, ensure the following:

* The user has a **CleanupOwl account** and can use the product in the browser at [app.cleanupowl.com](https://app.cleanupowl.com)
* The user can **create and scope API keys** there (same app; keys are **Bearer** tokens per the reference)
* The user is on a **subscription plan that includes API access**
  *(Not every plan exposes `/tp/api/`. If requests fail on billing or entitlements, the user upgrades or changes plan in [CleanupOwl](https://app.cleanupowl.com).)*
* The user has at least **one QuickBooks Online company connected**
* Full contract detail lives in [API Reference](./api-reference.md)

---

### Verifying Connections

1. Call: `GET /tp/api/connections`

   * Response shape: **`data.connections`** (array); each item’s **`_id`** is **`companyId`** on scan and solve calls—not the QBO realm id (`externalId`)

2. If no connections are found (or the required company is missing):

   * Call: `GET /tp/api/connect`
   * Extract **`data.connectionUrl`**
   * Ask the user to open that URL **while logged into** [CleanupOwl](https://app.cleanupowl.com) so they can finish QuickBooks OAuth

3. After successful authentication:

   * Call `GET /tp/api/connections` again and read **`data.connections[]._id`** for later calls

---

### Use This Skill When

* You need to **scan accounting data for issues**
* You want to **fix bookkeeping errors programmatically**
* You are building **automated cleanup workflows**
* You are assisting users with **bookkeeping maintenance via API**
* Steps should follow the **scan → review → fix** lifecycle (prep → apply, **one row at a time**)

---

### Avoid Using This Skill When

* No **QuickBooks company is connected**
* The task is **unrelated to bookkeeping cleanup**

---

## Key Concepts

- **Company ID (`companyId`)**  
  **`_id`** from **`data.connections`** on `GET /tp/api/connections`

- **Scan (`scanId`)**  
  Background job: pull QBO data (unless sync skipped), run detectors, fill **`data.scan.results`** when **`data.scan.status`** is **`completed`**

- **Issue (`issueId`)**  
  Stable id for a category of problem (keys under **`results`**)

- **Record (`recordId`)**  
  One flagged row; set **`recordId`** to a string matching **`Id`**, **`_id`**, **`entityId`**, or **`id`** on that row—see [Identifying a row](./api-reference.md#identifying-a-row-for-post-tpapisolveapply)

- **Solver (`solverId`)**  
  The fix path chosen after prep (or from solvers list); must match the **issue** or you get **`E_SOLVER_ISSUE_MISMATCH`**

---

## High-Level Workflow

```
1. Get companyId from data.connections
2. POST /tp/api/scan/start
3. Poll GET /tp/api/scan/:scanId until data.scan.status is completed or failed
4. Review data.scan.results by issueId
5. Fix records: POST /tp/api/solve/prep → POST /tp/api/solve/apply
6. Optional: GET /tp/api/scans/:scanId/solves for audit trail

```

---

## Flow 1: Run a Cleanup Scan

### Step 1: Fetch Company ID

```bash
GET /tp/api/connections
```

* Extract **`data.connections[]._id` → `companyId`**
* Never use **`externalId`** (realm) as **`companyId`**

---

### Step 2: Start Scan

```bash
POST /tp/api/scan/start
```

**Required:**

* `companyId`

**Recommended:**

* `period` (or other body flags from the reference) to limit scope

**Important:**

* Scans are **long-running**; **`POST`** returns immediately. Avoid redundant runs.

---

### Step 3: Poll for Completion

```bash
GET /tp/api/scan/:scanId?companyId=...
```

**Polling rules:**

* Wait at least **`data.pollAfterSeconds`** before the first poll (often **300**)
* Then **exponential backoff**: 5s → 10s → 20s → 30s (**max 30s** between tries)
* Stop when **`data.scan.status`** is **`completed`** or **`failed`**

---

### Step 4: Read Results

When **`data.scan.status`** is **`completed`**:

* **`data.scan.results`** is a map **`issueId` → bucket**
* Each bucket may expose **`count`**, **`severity`**, **`records`**, and/or **`table.rows`**

**`recordId`:** prefer the same rules as the reference (list vs table rows). For **`needsInput`**, options often come from **`recordFixOptions` / `metaFixOptions`** after prep, not from the raw scan row alone.

---

## Flow 2: Fix a Record

### Always run PREP before APPLY

---

### Step 1: Prepare Fix

```bash
POST /tp/api/solve/prep
```

**Purpose:**

* Row-level solvers and **`userInput`** hints
* Risk flags before writing to QBO

---

### Check risk flags (critical)

| Flag | Meaning | Agent action |
| --- | --- | --- |
| `hasSolvers` | **`false`** — no solver for this row/issue | Do not apply; read **`message`** if present |
| `hasLinkedTransactions` | Linked QBO txns (e.g. payment ↔ invoice) | Proceed carefully |
| `resolvedByOtherFix` | Already fixed elsewhere | Skip |
| `needsRevalidation` | QBO changed vs scan | New scan before apply |
| `entityConflicts` | Same entity under multiple issues | Resolve conflicts first |

---

### Step 2: Apply Fix

```bash
POST /tp/api/solve/apply
```

Include:

* `scanId`, `issueId`, `recordId`, `companyId`
* `solverId` (from prep when needed)
* Optional `userInput`

This call **writes to QuickBooks Online**.

---

### Step 3: Handle response

| Scenario | Action |
| --- | --- |
| `success: true` with **`data.action`** (typical apply success) | Done; read **`data.message`** |
| `success: true` with **`data.needsInput: true`** | **Not a failure** (HTTP **200**). Build **`userInput`** from **`missingFields` / `userInputSchema`**, then **POST apply again** with the same ids |
| `success: false` | Branch on **`error.code`**; use **`error.message`** for logging only |

---

### Step 4: Audit fixes

```bash
GET /tp/api/scans/:scanId/solves?companyId=...
```

Use to:

* Track solve attempts
* See **`action`**, **`success`**, **`solverId`**, **`executedAt`** per entry

---

## Flow 3: Ongoing cleanup routine

### Weekly routine

1. `GET /tp/api/billing` if fix limits matter
2. Start scan → poll → read **`results`**
3. Prioritize **high** severity, then medium; skip low if appropriate
4. Prep → apply per row; review history with **`/scans/:scanId/solves`**

### Monthly deep clean

* Broader **`period`** or full company scan per product guidance
* Compare severities across runs
* Clear remaining buckets; watch for new **`issueId`** values from **`GET /tp/api/issues`**

---

## Best practices for agents

* Call **`solve/prep`** before **`solve/apply`** unless you already have a validated **`solverId`** and **`userInput`**
* Honor **`pollAfterSeconds`** and backoff caps
* Pass **`companyId`** on every scan GET, history, and solves query as required
* Treat **`needsInput`** as a normal apply round-trip, not an exception path for **`success: false`**

---

## Common mistakes (avoid)

* Polling every second instead of waiting **`pollAfterSeconds`** and backing off
* Using **`externalId`** or a label instead of **`data.connections[]._id`** for **`companyId`**
* Skipping prep when **`recordFixOptions`** matter
* Applying against an old **`scanId`** after QBO changed (**`E_ENTITY_MODIFIED`**, **`needsRevalidation`**)
* Treating **`needsInput`** as failed apply
* **`solverId`** that does not match **`issueId`**

---

## Error handling strategy

1. Read **`error.code`** (stable); avoid logic on **`error.message`** alone
2. Map to recovery: reconnect QBO (**`E_QBO_AUTH_FAILED`**), re-scan (**`E_ENTITY_MODIFIED`**, archived scan), fix payload (**`E_MISSING_COMPANY_ID`**, mismatch codes), billing (**`E_FIX_LIMIT_EXCEEDED`**)
3. Details and HTTP semantics: [Errors](./api-reference.md#errors), [Common `error.code` values](./api-reference.md#common-errorcode-values), [Common mistakes](./api-reference.md#common-mistakes)

---

## Support and bugs

If something cannot be resolved with docs, retries, or reconnecting QuickBooks, or looks like a **defect**, point the user at human support.

**Contact:** [support@cleanupowl.com](mailto:support@cleanupowl.com)

**When:** repeated failures after normal recovery, inconsistent API behavior, access or billing blocked in-app, or suspected product bug.

Optional context for the user: account and in-app help from [app.cleanupowl.com](https://app.cleanupowl.com); general product info at [cleanupowl.com](https://cleanupowl.com).

**Email template**

**Subject:** `[CleanupOwl] <short topic>` — e.g. `[CleanupOwl] API error on solve/apply`

**Body (fill in brackets):**

```
What happened:
[One or two sentences]

What I expected:
[Expected behavior]

What I got instead:
[Actual behavior, including any error message text]

Context (include what you can):
- Endpoint or action: [e.g. POST /tp/api/solve/apply]
- Approximate time (with timezone): [when it occurred]
- error.code or HTTP status (if any): [values]
- companyId / scanId / issueId (only if the user is comfortable sharing): [optional]

Steps to reproduce (if known):
1. [...]
```

Keep the email factual; do not invent details.

---

## Quick reference

Relative paths are under **`/tp/api/`** on [app.cleanupowl.com](https://app.cleanupowl.com).

| Action | Endpoint |
| --- | --- |
| QBO connect URL | `/connect` |
| List companies | `/connections` |
| Issue catalog | `/issues` |
| Start scan | `/scan/start` |
| Poll scan | `/scan/:scanId` |
| Scan history | `/scan/history` |
| Prep | `/solve/prep` |
| Apply | `/solve/apply` |
| Audit solves | `/scans/:scanId/solves` |
| Billing | `/billing` |

---

## Agent decision logic

* No finished **`scanId`** for the task → start scan (then poll)
* Scan not **`completed`** / **`failed`** → poll with backoff
* Scan **`completed`** → read **`results`**, pick row, prep → apply
* Apply returns **`needsInput`** → enrich **`userInput`**, POST apply again
* **`success: false`** → **`error.code`**-driven recovery
* Product or API bug → **Support and bugs** (template + [support@cleanupowl.com](mailto:support@cleanupowl.com))

---

## Companion files in this repo

All paths are valid from the repository root:

| File | Role |
| --- | --- |
| [api-reference.md](./api-reference.md) | Full HTTP reference |
| [README.md](./README.md) | Integration kit overview |
| [cleanupowl-tp-api-openapi.yaml](./cleanupowl-tp-api-openapi.yaml) | OpenAPI 3 spec |
| [cleanupowl-tp-api-postman-collection.json](./cleanupowl-tp-api-postman-collection.json) | Postman collection |
| [examples/basic.md](./examples/basic.md) | Short `curl` examples |
| [examples/advanced.md](./examples/advanced.md) | Prep/apply, errors, targeted scans |
