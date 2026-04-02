# CleanupOwl API — integration kit

> **Beta:** This API is in **beta**. Endpoints, response shapes, and documentation may change without a version bump; watch release notes and update integrations accordingly.

This repository bundles materials for integrating with the **CleanupOwl API**: human-readable reference, machine-readable spec, runnable examples, Postman assets, and an agent-oriented workflow guide. The API is organized around [REST](https://en.wikipedia.org/wiki/Representational_State_Transfer), uses resource-oriented URLs under **`/tp/api/`**, accepts [JSON](https://www.json.org/) on `POST` requests, and returns responses in a consistent envelope—see [`api-reference.md`](./api-reference.md) for the full picture.

**Scans run in the background.** After you start a scan, poll until the run finishes or fails. Solvers fix **one row at a time** with **prep** and **apply**; there is no “fix every row in one call” option.

This kit is aimed at **integrators**: scripts, partner tools, internal automation, and **LLM agents**. To use CleanupOwl without calling the API, use the main application and QuickBooks Online (QBO)–connected workflows in the product UI.

---

## Contents

| Asset | Description |
|-------|-------------|
| [`api-reference.md`](./api-reference.md) | Full HTTP reference — authentication, request/response shapes, **`error.code`** values, and endpoint details |
| [`cleanupowl-tp-api-openapi.yaml`](./cleanupowl-tp-api-openapi.yaml) | OpenAPI 3.0.3 — import into Swagger, codegen tools, or Postman |
| [`cleanupowl-tp-api-postman-collection.json`](./cleanupowl-tp-api-postman-collection.json) | Postman collection with environment-style variables |
| [`SKILL.md`](./SKILL.md) | Agent skill — how to run scans, read issues, prep/apply fixes, and handle errors in agent workflows |
| [`examples/basic.md`](./examples/basic.md) | Short **`curl`** examples by endpoint |
| [`examples/advanced.md`](./examples/advanced.md) | Prep → apply, **`needsInput`**, targeted scans, and error recovery |

Start with **`api-reference.md`** (especially [Quick start](./api-reference.md#quick-start)) for a step-by-step first scan and fix.

---

## Base URL

```plaintext
https://app.cleanupowl.com
```

Append **`/tp/api/...`** to that host (for example, `https://app.cleanupowl.com/tp/api/connections`). There is **no `/v1`-style path prefix**. Breaking changes are communicated **out of band** (release notes, partner channels); plan to watch those if you maintain an integration.

---

## Authentication

The CleanupOwl API uses **API keys** as **Bearer** tokens. Create and scope keys in the CleanupOwl application. All requests must use **HTTPS**; unauthenticated calls are rejected (**401** / **403**).

```http
Authorization: Bearer <your-api-key>
```

Keep keys out of public repos, browser code, and shared docs. Integrations should branch on **`error.code`** in failures; use **`message`** for logging and display.

### Scopes

| Scope | Required for |
|-------|----------------|
| `scan` | `GET /tp/api/issues`, scan endpoints such as `/tp/api/scan/...` |
| `solve` | `GET /tp/api/solvers`, `GET /tp/api/issues/:issueId/solvers`, `POST /tp/api/solve/prep`, `POST /tp/api/solve/apply`, `GET /tp/api/scans/:scanId/solves` |
| *(none)* | `GET /tp/api/billing`, `GET /tp/api/connections`, `GET /tp/api/connect` |

---

## Verify connectivity

Set variables and call connections (no `scan` scope required):

```bash
export TP_BASE_URL="https://app.cleanupowl.com"
export TP_API_KEY="your-api-key-here"

curl -sS "$TP_BASE_URL/tp/api/connections" \
  -H "Authorization: Bearer $TP_API_KEY"
```

Use each connection object’s **`_id`** as **`companyId`** on scan and solve calls—it is **not** the QBO realm id.

For **`scan`**/**`solve`** flows, copy-paste commands live in [`examples/basic.md`](./examples/basic.md) and [`examples/advanced.md`](./examples/advanced.md).

---

## Endpoints (summary)

| Method | Path | Role |
|--------|------|------|
| GET | `/tp/api/connections` | List connected QBO companies |
| GET | `/tp/api/connect` | Get URL to link QBO in the browser |
| GET | `/tp/api/issues` | List issue types |
| POST | `/tp/api/scan/start` | Start a background scan |
| GET | `/tp/api/scan/:scanId` | Poll status and read **`results`** when `completed` |
| GET | `/tp/api/scan/history` | List past scans |
| GET | `/tp/api/solvers` | List solvers |
| GET | `/tp/api/issues/:issueId/solvers` | Solvers for one issue |
| POST | `/tp/api/solve/prep` | Prep one row (solvers, flags, options) |
| POST | `/tp/api/solve/apply` | Apply a solver to one row in QBO |
| GET | `/tp/api/scans/:scanId/solves` | Solve audit trail for a scan |
| GET | `/tp/api/billing` | Plan and usage (for example **`usage.fixesRemaining`**) |

Parameter names, bodies, and error codes are defined in [`api-reference.md`](./api-reference.md).

---

## Typical integration flow

1. **`GET /tp/api/connections`** — resolve **`companyId`** (connection **`_id`**).
2. **`POST /tp/api/scan/start`** — obtain **`scanId`**; wait at least **`data.pollAfterSeconds`** before the first poll (often **300** seconds).
3. **`GET /tp/api/scan/:scanId`** — poll with backoff until **`data.scan.status`** is **`completed`** or **`failed`**.
4. **`POST /tp/api/solve/prep`** — choose **`solverId`** and inspect risk flags for the row.
5. **`POST /tp/api/solve/apply`** — write the fix to QBO; handle **`needsInput`** (still HTTP 200, **`success: true`**) by posting again with **`userInput`**.
6. **`GET /tp/api/scans/:scanId/solves`** — review what ran.

---

## Support

Email [support@cleanupowl.com](mailto:support@cleanupowl.com) for bugs or integration help.
