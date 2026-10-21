# UptimePilot API Contracts

| | |
|---|---|
| Status | **PROPOSED**, draft v0.1. No endpoint is implemented; no contract is confirmed by tests |
| Last updated | 2026-10-21 |
| Related | [`PRODUCT_SPEC.md`](PRODUCT_SPEC.md) (requirement IDs, `AC-n`, `Q-n`), [`ARCHITECTURE.md`](ARCHITECTURE.md) (§3 backend, §4 auth, §8 jobs, §11 billing, §15 security), [`DATABASE_SCHEMA.md`](DATABASE_SCHEMA.md) (collections, `V-n`), [`IMPLEMENTATION_PLAN.md`](IMPLEMENTATION_PLAN.md), [`CLAUDE.md`](../CLAUDE.md) |

> **Every contract here is proposed until implementation and tests confirm it.** When an endpoint is built, its section is updated from "Proposed" to "Confirmed by tests" with the evidence (commit and test names). No payment provider, email provider or AI provider is assumed to be configured. No credentials appear in this document; all values are placeholders.

## Reading guide

- **Notation.** Request and response shapes use a compact *schema notation* (not code): `field: type` is required, `field?: type` is optional, `| null` is nullable, `enum(a, b)` is a closed string set. Strings in angle brackets, e.g. `<ISO-8601 UTC>`, describe a format.
- **Path shorthand.** `{ws}` means `/v1/workspaces/{workspaceId}`. It is a documentation shorthand; real paths are written out in full.
- **Roles.** `O` = account owner, `A` = workspace administrator, `M` = member, `PA` = platform administrator. "Any member" = O, A or M. The role matrix is PRODUCT_SPEC §4.1; items marked ⚠ depend on its open question Q-3.
- **⚠ Validate** marks a number, behavior or choice to confirm before it is relied on. All collected in [§15](#15-deviations-schema-additions-and-open-questions).
- **Numeric limits** (sizes, rates, ranges) are proposals, not measured values.

---

## 1. Scope

### 1.1 Covered

Auth verification and workspace authorization; workspaces and memberships; monitors; monitor tests and check history; incidents and acknowledgement; SSL status; SEO audits, pages and findings; reports; notification preferences; subscription and usage; payment webhooks; platform administration.

### 1.2 Not covered

- Implementation, OpenAPI generation, SDKs (none created in this task).
- Internal worker endpoints are listed only for completeness ([§12.2](#122-internal-endpoints-not-a-client-contract)).
- AI explanation endpoints appear only as a short optional appendix ([§11.13](#1113-optional-ai-explanations-phase-7)).
- Browsers never call Firestore or Storage directly (ARCHITECTURE §5.1); this API is the only client interface.

---

## 2. Conventions

### 2.1 Protocol and format

| Topic | Convention |
|---|---|
| Transport | HTTPS only. Plain HTTP is redirected or refused at the edge |
| Base path | `/v1`. Breaking changes ship as `/v2`; additive changes (new optional fields, new endpoints, new enum values clients must tolerate) stay in `/v1` |
| Format | JSON, UTF-8. Requests with a body require `Content-Type: application/json`, else `415 unsupported_media_type`. Responses are `application/json` |
| Field naming | `lowerCamelCase` (matches DATABASE_SCHEMA) |
| Timestamps | ISO-8601 UTC with `Z` and millisecond precision at most, e.g. `2026-10-21T12:00:00Z`. **Server-generated** fields (`createdAt`, `updatedAt`, …) can never be supplied by clients |
| Money | Integer minor units (paise) plus ISO currency code. Never floats |
| Booleans/nulls | Absent optional fields are omitted; explicitly cleared fields are `null` |
| Enumerations | Lowercase snake_case strings. **Clients must tolerate unknown enum values** in responses |
| Unknown request fields | Rejected with `422 validation_failed` (`unknown_field`), to surface mass-assignment attempts and typos |
| Request size | Body ≤ 64 KiB by default ⚠ Validate; webhooks have their own cap ([§13](#13-payment-webhooks-signature-verification-and-duplicates)) |
| Compression | Response compression allowed; request bodies must not be compressed (bomb avoidance) |
| Caching | Authenticated responses: `Cache-Control: no-store`. Conditional requests use `ETag`/`If-None-Match` only where noted |
| Security headers | `X-Content-Type-Options: nosniff`, `Strict-Transport-Security`; CORS allows only the app's origin(s); **no cookies are used**, so CORS credentials are off |

### 2.2 Headers

| Header | Direction | Purpose |
|---|---|---|
| `Authorization: Bearer <Firebase ID token>` | request | Authentication on every endpoint except those in [§12.1](#121-unauthenticated--special-auth-endpoints) |
| `X-Request-Id` | request (optional) / response (always) | Correlation. If the client sends a well-formed value it is echoed; otherwise the server generates one. Included in every error body and every log line |
| `Idempotency-Key` | request | See [§6](#6-idempotency) |
| `Idempotent-Replayed: true` | response | The response replays a previously completed request |
| `If-Match` / `ETag` | request / response | Optimistic concurrency on `PATCH` of monitors and notification settings (optional for clients; see each endpoint) |
| `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset` | response | Current rate-limit state for the request's class. Header names follow the IETF rate-limit-headers draft ⚠ Validate |
| `Retry-After` | response | On `429`, `503`, and `202` polling hints (seconds) |
| `Location` | response | On `201` (new resource) and `202` (status resource) |
| `X-Admin-Reason` | request | Required on admin endpoints that read customer data or write ([§11.12](#1112-platform-administration)) |

### 2.3 Identifiers

- Resource IDs are **opaque, stable, server-generated strings** (DATABASE_SCHEMA §1.6). Clients must not parse them, assume length, or construct them. They never contain emails, URLs or secrets.
- Every workspace-scoped representation includes `id` and `workspaceId`.
- Some IDs are deterministic (e.g. a check's `id` is its slot's epoch seconds; a page's `id` is a hash of its normalized URL). They remain opaque to clients; the stability guarantee is that the same logical entity keeps the same ID across retries.
- IDs appear in paths; **an ID alone never authorizes access** ([§9](#9-tenant-isolation-rules)).
- The reserved path segment `current` may stand in for an ID where noted (e.g. `snapshots/current`).

### 2.4 Resource representations never expose

`purgeAt`, `schemaVersion`, `leaseUntil`/`leaseToken`, `tokenHash`, monitor secret values or ciphertext, provider customer/subscription references, storage bucket/object names or signed URLs (except in the one download response), `inputDigest` internals, other tenants' identifiers, stack traces, resolved IP addresses of customer targets.

### 2.5 Versioning of resources

Where concurrent edits matter, a representation carries a weak `ETag` (monitors: derived from `configVersion`; notification settings: from `updatedAt`). `PATCH`/`PUT` with a stale `If-Match` → `412 stale_version`. Sending no `If-Match` means last-write-wins ⚠ Validate (Q-A5).

---

## 3. Authentication and authorization model

### 3.1 Authentication (every non-public request)

1. Extract `Authorization: Bearer <ID token>`. Missing, malformed or non-Bearer → `401 unauthenticated`.
2. Verify the Firebase ID token: signature against the identity provider's published keys, issuer, audience (the project), expiry/not-before, non-empty subject. Failure → `401 unauthenticated` (body does not say which check failed; a `reason` hint of `expired` is allowed so the client knows to refresh).
3. Read `email_verified` from the verified token. Endpoints that create monitors, audits or invitations require `true`, else `403 email_not_verified`.
4. Build the `Principal(uid, emailVerified, isPlatformAdmin?)`. **Identity is taken only from the verified token**, never from request bodies, query strings or other headers.

### 3.2 Workspace authorization (every `{ws}` request)

1. Parse `workspaceId` from the path.
2. Load `workspaces/{wid}/members/{uid}` (live; not from token claims). **No membership → `404 not_found`** (indistinguishable from a nonexistent workspace).
3. Load workspace status and `entitlements` (cache ≤ a few seconds per instance ⚠ Validate): `suspended` or `deleting` rejects per [§3.4](#34-workspace-state-gates).
4. Check the caller's role against the endpoint's required role. Insufficient → `403 forbidden`. A member may learn that a resource exists in their own workspace; a non-member never can.
5. Run endpoint-specific checks (resource exists under the tenant path, plan quota, state preconditions).
6. Authorization failures return no resource data.

### 3.3 Minimum-role summary (authoritative per endpoint in §11)

| Capability | O | A | M |
|---|:--:|:--:|:--:|
| Read monitors, history, incidents, SSL, audits, reports, usage | ✓ | ✓ | ✓ |
| Create/edit monitors, pause/resume, run tests | ✓ | ✓ | ✓ |
| Delete monitors | ✓ | ✓ | ⚠ Q-3 |
| Acknowledge incidents | ✓ | ✓ | ✓ |
| Start/cancel audits; request/download reports | ✓ | ✓ | ✓ |
| Notification settings (workspace/monitor) | ✓ | ✓ | – |
| Own notification preferences | ✓ | ✓ | ✓ (self only) |
| Members, invitations, role changes | ✓ | ✓ (not owner) | – |
| Workspace rename, audit-events view | ✓ | ✓ | – |
| Subscription, checkout, billing history | ✓ | – | – |
| Ownership transfer, workspace deletion | ✓ | – | – |
| Reports download history | ✓ | ✓ | – |

### 3.4 Workspace state gates

| Workspace `status` | Reads | Writes / job starts |
|---|---|---|
| `active` | allowed | allowed (subject to role/quota) |
| `suspended` | allowed for `GET {ws}`, `GET {ws}/access`, `GET {ws}/usage`; otherwise `403 workspace_suspended` | `403 workspace_suspended` |
| `deleting` | `403 workspace_deleting` | `403 workspace_deleting` |

⚠ Validate (Q-A10): whether suspended workspaces may read their own data/exports.

### 3.5 Authorization is server-side; UI gating is cosmetic

`GET {ws}/access` (below) exists so the UI can hide controls. The API enforces every rule regardless of what the UI hides.

---

## 4. Errors

### 4.1 Envelope (all error responses)

```
{
  "error": {
    "code": "validation_failed",            // stable, machine-readable
    "message": "One or more fields are invalid.",   // human-readable, safe, may change
    "requestId": "req_…",
    "details": [                              // optional; present for validation_failed and some conflicts
      { "field": "url", "code": "url_blocked_by_policy", "message": "This address cannot be monitored." }
    ],
    "retryAfterSec": 30                       // optional; mirrors Retry-After
  }
}
```

- `code` values are a **stable registry** (§4.2); clients branch on `code`, never on `message`.
- Messages are safe to show to users and **never** contain secrets, internal hostnames, resolved IP addresses, stack traces, other tenants' data, or the validity of a token's individual checks.
- Content type `application/json`. (An RFC 9457 `application/problem+json` shape was considered; this envelope was chosen for a single predictable shape. ⚠ Validate, Q-A1.)

### 4.2 Status codes and error registry

| Status | `code` | When |
|---|---|---|
| 400 | `invalid_request` | Malformed JSON, wrong types at the syntax level, invalid query-parameter syntax, malformed cursor |
| 401 | `unauthenticated` | Missing/invalid/expired token |
| 401 | `invalid_signature` | Webhook signature failed ([§13](#13-payment-webhooks-signature-verification-and-duplicates)) |
| 403 | `forbidden` | Authenticated member lacks the required role |
| 403 | `email_not_verified` | Action requires a verified email |
| 403 | `plan_limit_reached` | A plan quota would be exceeded (`details` names the quota, limit and current usage) ⚠ Q-A2 (vs `402`) |
| 403 | `workspace_suspended` / `workspace_deleting` | [§3.4](#34-workspace-state-gates) |
| 403 | `admin_required` | Not a platform administrator, or MFA not satisfied |
| 404 | `not_found` | Resource missing **or** not visible to the caller (including cross-tenant and non-member). One code for both, by design |
| 405 | `method_not_allowed` | Route exists, method doesn't |
| 409 | `conflict` | Generic state conflict (e.g. duplicate unique constraint) |
| 409 | `state_conflict` | Operation invalid in the resource's current state (e.g. cancel a completed audit, request a report for an unfinished audit) |
| 409 | `sole_owner` | Would leave the workspace without exactly one owner |
| 409 | `idempotency_conflict` | Same key reused with a different request body |
| 409 | `request_in_progress` | Same key, original request still running; includes `Retry-After` |
| 412 | `stale_version` | `If-Match` mismatch |
| 413 | `payload_too_large` | Body exceeds the limit |
| 415 | `unsupported_media_type` | Not `application/json` |
| 422 | `validation_failed` | Semantic field errors; `details` lists them |
| 429 | `rate_limited` | Rate limit exceeded; `Retry-After` set |
| 500 | `internal_error` | Unexpected failure; body contains only `requestId` |
| 503 | `dependency_unavailable` | A required dependency (datastore, queue) is down; `Retry-After` set |
| 503 | `feature_disabled` | A kill switch is active (dispatcher, crawler, email, AI, reports, sign-ups) |
| 503 | `billing_unavailable` | No payment provider is configured or it is unreachable |

`details[].code` registry (field-level): `required`, `invalid_type`, `invalid_format`, `too_short`, `too_long`, `out_of_range`, `invalid_enum`, `unknown_field`, `duplicate`, `url_invalid`, `url_blocked_by_policy`, `not_allowed_for_plan`, `feature_unavailable`, `range_too_large`, `unsupported_filter_combination`, `invalid_reference`, `domain_unverified`.

### 4.3 Rules

- **Validation precedence:** syntax (400) → authentication (401) → tenancy/role (404/403) → state gates → field validation (422) → quota (403 `plan_limit_reached`) → conflict (409). Authentication and tenancy are checked **before** body validation so unauthorized callers learn nothing about field rules.
- **SSRF rejections** are `422` with field code `url_blocked_by_policy` and a generic message; the response never says why (private range, metadata endpoint, rebinding) or what the address resolved to.
- Existence is never revealed across tenants: cross-tenant IDs, unknown IDs and deleted IDs are all `404 not_found`.
- `Retry-After` accompanies `429`, `503` and `409 request_in_progress`.

---

## 5. Pagination and filtering

### 5.1 Pagination (cursor-based only)

Query: `limit` (integer, default **25**, max **100**; some endpoints state other limits) and `cursor` (opaque string from a previous response). **No offset, no page numbers, no total counts.**

Response envelope for every list:

```
{
  "items": [ … ],
  "page": { "limit": 25, "hasMore": true, "nextCursor": "…" }   // nextCursor omitted when hasMore is false
}
```

- Cursors are opaque, signed (HMAC), and **bound to the workspace, route, filters and ordering** they were issued for. A cursor replayed on another tenant, route or filter → `400 invalid_request`. Tampered or expired cursors → `400 invalid_request`. (DATABASE_SCHEMA §5.1.)
- Ordering is fixed per endpoint (documented); results are stable under concurrent inserts (the sort key includes the document ID as tiebreaker).
- `limit` above the maximum is clamped or rejected (`422 out_of_range`); this contract **rejects** so clients notice.
- Counts shown in the UI come from maintained counters (`usage`, audit progress), not list totals.

### 5.2 Filtering

Filters are query parameters. **Only filter combinations backed by a declared index are supported** (DATABASE_SCHEMA §6); an unsupported combination returns `422 validation_failed` with `unsupported_filter_combination`. Time filters (`from`, `to`) are ISO-8601 UTC, inclusive `from`, exclusive `to`. Unknown query parameters → `422 unknown_field`.

---

## 6. Idempotency

### 6.1 Header

`Idempotency-Key: <1–128 chars, [A-Za-z0-9_.:-]>`, a client-generated unique value per logical operation (a UUID is typical).

### 6.2 Semantics

Keys are scoped to `(uid, route template, key)` and stored for 24 h (DATABASE_SCHEMA W28). For a keyed `POST`:

| Situation | Result |
|---|---|
| First request | Processed normally; the outcome (resource pointer) is recorded |
| Retry, same key and same canonical body, original **completed** | Replays the original outcome: same status code, current representation of the created resource, header `Idempotent-Replayed: true` |
| Retry while the original is **in progress** | `409 request_in_progress` + `Retry-After` |
| Same key, **different** body | `409 idempotency_conflict` |
| Key from another user/route | Independent (scoped), no cross-talk |

"Same body" means equal after canonicalization (key order and whitespace ignored). Failed requests that did not create a resource may be retried with the same key.

### 6.3 Where it applies

| Endpoint class | Header |
|---|---|
| Creates that start work or consume quota: audits, reports, monitor tests, checkout sessions | **Required** (`400 invalid_request` if missing) |
| Other creates: workspaces, monitors, invitations, blocked hosts | Supported, strongly recommended |
| `PUT`, `PATCH`, `DELETE`, pause/resume, cancel, acknowledge | Naturally idempotent by definition (below); header ignored |
| Webhooks | Deduplicated by the **provider event ID**, not this header ([§13](#13-payment-webhooks-signature-verification-and-duplicates)) |

### 6.4 Natural idempotency of other methods

`PUT` replaces to a target state; `PATCH` of the same values is a no-op; `pause` on a paused monitor returns `200` unchanged; `DELETE` repeated returns `404 not_found` after the first success (clients treat as success). Quota counters change only on the first effective call (counters update in the same transaction as the resource write).

---

## 7. Rate limits and quotas

These are **different things**: rate limits protect the service from bursts and abuse (`429 rate_limited`); quotas enforce plan entitlements (`403 plan_limit_reached`). Both are server-side (CLAUDE.md #8).

### 7.1 Rate-limit classes (proposed; numbers are placeholders ⚠ Validate, V-18)

| Class | Applies to | Proposed limit | Keyed by |
|---|---|---|---|
| `R` read | `GET` endpoints | 300 / min | user (and workspace) |
| `W` write | non-expensive `POST/PUT/PATCH/DELETE` | 60 / min | user (and workspace) |
| `X` expensive | start audit, generate report, monitor test / check-now, AI explanation, test email | 10 / min, plus per-endpoint hourly caps below | user and workspace |
| `A` auth-adjacent | `PUT /v1/me`, invitation preview/accept, unsubscribe | 20 / min | IP and user |
| `H` webhook | provider webhooks | generous (e.g. 600 / min) | source/provider |
| `ADM` | `/v1/admin/**` | 120 / min | admin user |

Per-endpoint caps (examples, ⚠ Validate): invitations 20 / hour / workspace; test emails 5 / hour / user; monitor tests 10 / min / workspace; audits 10 / hour / workspace (in addition to plan quota); reports 20 / hour / workspace.

Responses carry `RateLimit-*` headers; `429` carries `Retry-After`. Limits are enforced before expensive work. The counter store is **not** per-request Firestore writes (V-18; undecided).

### 7.2 Quotas

Quotas come from the workspace `entitlements` snapshot (plan configuration; provisional values per PRODUCT_SPEC §5.10; `null` = undecided and treated as the most restrictive safe default). Enforced atomically with the guarded write. A rejected request returns:

```
403 { "error": { "code": "plan_limit_reached", "message": "…",
      "details": [ { "field": "quota", "code": "monitors", "limit": 5, "current": 5 } ] } }
```

Quota-governed actions: create monitor (`maxMonitors`), check interval below plan minimum (`422 not_allowed_for_plan` on the field), audit start (`maxAuditsPerMonth`, page cap snapshot `maxPagesPerAudit`), report request (`maxReportsPerMonth`), AI (`maxAiRequestsPerMonth`), invitations (`maxMembers`). Which actions count (paused monitors, failed audits) is undecided (PRODUCT_SPEC Q-6).

---

## 8. Synchronous requests vs background jobs

**Principle.** No HTTP request waits for a monitoring check, a crawl, a report render or an email. API handlers do bounded work (validate, authorize, read/write a few documents, enqueue) and respond. Long work happens in workers, which keep running when the browser closes (ARCHITECTURE principle 1). Every API request has a server-side deadline (default 10 s, hard cap 30 s ⚠ Validate); exceeding it returns an error, not a longer wait. **The API does not fetch customer URLs itself**; only fetch workers do (ARCHITECTURE §3.3).

### 8.1 Classification

| Operation | Mode | Completion signal |
|---|---|---|
| Auth/identity, workspace/member/invitation CRUD, monitor CRUD, pause/resume, notification settings, usage, listings and detail reads | **Synchronous** | Immediate response |
| Monitor scheduled checks | **Background** (dispatcher → queue → probe) | New check/incident/notification records |
| Monitor test / check now | **Async job** `monitor-tests` | `GET` poll (seconds) |
| SSL inspection | **Background**, scheduled | `GET {ws}/monitors/{id}/ssl` |
| Start SEO audit | **Async job** `audits` | Poll audit; email on completion/failure |
| Cancel audit | Sync request that sets a flag; stopping is async | Poll audit (`cancelling` → `cancelled`) |
| Generate report | **Async job** `reports` | Poll report; optional email |
| Report download | Sync (issues a short-lived URL) | Immediate |
| Delete monitor / workspace | Sync request marks deletion; cascade is async | `204` / `202` |
| Plan changes, billing | Driven by **webhooks + async processor** | Subscription state updates |
| Admin plan "apply to all" | **Async job** | Poll |
| Notification delivery | **Background** outbox | `GET {ws}/notifications` |

### 8.2 Job pattern for `202 Accepted`

```
POST …                              → 202 Accepted
Location: /v1/workspaces/{wid}/audits/{auditId}
Retry-After: 2
{ "id": "…", "status": "queued", … full resource representation … }

GET  …/audits/{auditId}             → 200  { "status": "running", "progress": { … } }
```

- The job **is** the resource (an audit, a report, a monitor test); its `status` field is the job status. There is no separate generic jobs endpoint.
- Clients poll with backoff (suggested 2 s → 10 s) and honor `Retry-After`. Conditional `GET` with `ETag`/`If-None-Match` may return `304`.
- Terminal statuses are stable and carry safe failure info (`failureReason`, `errorCode`), never raw errors.
- Closing the browser never cancels a job; only the explicit cancel endpoint does.
- Server-sent events or WebSockets for progress are **not** part of this contract (⚠ Q-A13; polling is sufficient for the MVP).

---

## 9. Tenant isolation rules

Applicable to every `{ws}` endpoint; each endpoint in §11 names which rules it adds.

| Rule | Statement |
|---|---|
| **T1** | The tenant is taken from the **path**, and authorization requires live membership in that workspace (§3.2). Bodies and query strings never carry authoritative `workspaceId`; a supplied `workspaceId` field is rejected as `unknown_field`. |
| **T2** | Resource IDs are looked up **under the tenant path** only. An ID belonging to another workspace is simply not found: `404 not_found`, byte-identical to a nonexistent ID (including timing within reason ⚠). |
| **T3** | Lists are always scoped by construction to the tenant path; no filter can widen scope. |
| **T4** | Cursors are bound to workspace + route + filters (§5.1). |
| **T5** | Cross-resource references in bodies (`monitorId`, `auditId`, `snapshotId`, recipient `uid`s) are validated to belong to the **same workspace**; otherwise `422 validation_failed` (`invalid_reference`) without revealing whether the ID exists elsewhere. |
| **T6** | Workspace-state gates apply (§3.4). |
| **T7** | Server-managed fields (`ownerUid`, `role` on non-role endpoints, `planId`, `entitlements`, `status` on monitors, counters, lease fields, timestamps) are not writable through any customer endpoint. |
| **T8** | Responses contain only the caller's tenant data; error bodies never echo other tenants' identifiers. |
| **T9** | Platform admin access is a separate router and audited (§11.12); it grants no implicit access on customer routes. |
| **T10** | Worker (OIDC) credentials and user tokens are not interchangeable: user routes reject worker tokens and vice versa. |

Required tests: DATABASE_SCHEMA §11.1 cross-tenant matrix, generated for every route in §11 (§14).

---

## 10. Shared resource representations

Fields are server-produced unless a request section says otherwise. `<ts>` = ISO-8601 UTC timestamp.

```
User {
  id: string                      // Firebase Auth uid
  email: string; emailVerified: boolean
  displayName?: string; locale?: string; timeZone?: string
  defaultWorkspaceId?: string | null
  createdAt: <ts>
}

Workspace {
  id: string; name: string; status: enum(active, suspended, deleting)
  ownerUid: string; planId: string
  createdAt: <ts>; updatedAt: <ts>
}

Member { uid: string; displayName?: string; role: enum(owner, admin, member); addedAt: <ts> }
   // email is NOT included; an admin sees displayName and uid only

Invitation {
  id: string; inviteeEmail: string; role: enum(admin, member)
  status: enum(pending, accepted, revoked, expired)
  invitedBy: string; expiresAt: <ts>; createdAt: <ts>
}

Entitlements {                    // snapshot from plan configuration (+ admin override)
  maxMonitors: int | null; maxPagesPerAudit: int | null; minCheckIntervalSec: int | null
  historyRetentionDays: int | null; maxMembers: int | null; maxAuditsPerMonth: int | null
  maxReportsPerMonth: int | null; maxAiRequestsPerMonth: int | null
}                                 // null = not yet defined (restrictive default applies)

Monitor {
  id: string; workspaceId: string; name: string; type: enum(website, api)
  url: string; host: string; method: string
  headerNames: string[]           // names only; values never returned
  hasBody: boolean
  expectedStatusRanges: { from: int, to: int }[]
  intervalSec: int; timeoutSec: int; maxRedirects: int; failureThreshold: int
  status: enum(active, paused, quota_suspended)
  state: enum(pending, up, down, unknown); stateSince?: <ts>; consecutiveFailures: int
  lastCheck?: { checkId, scheduledAt, ok, httpStatus?, latencyMs?, errorCategory? }
  openIncidentId?: string | null
  nextCheckAt?: <ts> | null
  sslEnabled: boolean
  certState?: { fingerprint32, notAfter, status } | null
  configVersion: int; createdBy: string; createdAt: <ts>; updatedAt: <ts>
}

Check {
  id: string                      // slot epoch seconds, opaque to clients
  monitorId: string; scheduledAt: <ts>; startedAt: <ts>
  ok: boolean; state: enum(up, down)
  httpStatus?: int | null; latencyMs?: int | null
  errorCategory?: enum(dns_failure, connect_timeout, tls_error, unexpected_status,
                       redirect_limit, blocked_by_policy, response_too_large, internal_error) | null
  redirectCount?: int; responseBytes?: int; configVersion: int
}

Incident {
  id: string; monitorId: string; monitorName: string
  kind: enum(downtime, certificate); status: enum(open, resolved)
  openedAt: <ts>; resolvedAt?: <ts> | null; durationSec?: int | null
  failedCheckCount: int; firstErrorCategory: string; lastErrorCategory: string
  certFingerprint32?: string
  acknowledged: boolean; acknowledgedAt?: <ts>; acknowledgedBy?: string; acknowledgementNote?: string
}                                 // acknowledgement fields: proposed schema addition (§15)

IncidentEvent { seq: int; type: enum(opened, error_category_changed, notification_queued,
                acknowledged, resolved, manually_closed); at: <ts>; checkId?: string; detail?: object }

Audit {
  id: string; startUrl: string; siteKey: string
  status: enum(queued, running, cancelling, analyzing, completed, failed, cancelled)
  failureReason?: enum(start_unreachable, blocked_by_robots, blocked_by_policy, internal_error, timeout) | null
  failureDetail?: string
  pageCap: int; maxDepth: int; options: { respectRobots: boolean, externalLinkCheck: boolean }
  progress: { pagesDiscovered, pagesFetched, pagesFailed, pagesBlocked: int, capReached: boolean }
  ruleSetVersion: string; currentSnapshotId?: string | null; latestScore?: int | null
  cancelRequested: boolean; requestedBy: string
  createdAt: <ts>; startedAt?: <ts>; finishedAt?: <ts>
}

Snapshot {
  id: string; auditId: string; ruleSetVersion: string
  scoreName: "UptimePilot Technical SEO Score"
  score: int                      // 0–100
  disclaimer: { key: string, text: string }   // MUST be displayed with the score
  weights: { [category: string]: int }
  categoryScores: { [category: string]: { score, weight, evaluated, passed, failed: int } }
  findingCounts: { total: int, bySeverity: { [s]: int }, byCategory: { [c]: int } }
  pagesAnalyzed: int; pageCap: int; capReached: boolean
  crawlStartedAt: <ts>; crawlFinishedAt: <ts>; computedAt: <ts>
}

PageSummary { id, url, depth: int, fetchState, httpStatus?: int | null, statusClass: int,
              finalUrl?: string, inSitemap?: boolean, errorCategory?: string | null, fetchedAt?: <ts> }
Page extends PageSummary { redirectChain: { url, status }[], contentType?: string,
              responseBytes?: int, fetchMs?: int, source: string, facts?: object /* bounded, untrusted text */ }

Finding {
  id: string; ruleId: string; category: string
  severity: string; severityRank: int
  scope: enum(page, site); pageId?: string | null; url?: string | null
  relatedPageIds: string[]; affectedCount: int
  evidence: { kind: string, observed: string, expected?: string, location?: object }[]   // never empty
  ruleSetVersion: string
}                                 // evidence text is untrusted third-party content: render as text only

Report {
  id: string; auditId: string; snapshotId: string; format: enum(pdf, csv)
  status: enum(pending, generating, ready, failed, expired, deleted)
  requestedBy: string; includesAi: boolean; generatorVersion: string
  contentType?: string; sizeBytes?: int; errorCode?: string | null
  createdAt: <ts>; readyAt?: <ts>; expiresAt: <ts>
}

ReportDownload { id: string; reportId: string; method: enum(signed_url, stream)
                 url?: string; expiresAt?: <ts> }   // url present only in the creating response

NotificationSettings {
  memberRecipientUids: string[]; externalRecipients: []   // MVP: must be empty (reserved)
  eventToggles: { [eventType: string]: boolean }; enabled: boolean; minIntervalSec?: int
  updatedAt: <ts>
}
NotificationPreferences { mutedTypes: string[]; updatedAt: <ts> }
Notification { id, type, category: enum(alert, product, account_security),
               status: enum(pending, sending, retry, sent, failed, suppressed, cancelled),
               monitorId?, incidentId?, auditId?, reportId?, attempts: int, sentAt?: <ts>, createdAt: <ts> }
               // no recipient email addresses

Plan { id: string; displayName: string; provisional: boolean
       price: { currency: string, amountMinor: int, interval: "month" }; quotas: Entitlements; version: int }
Subscription { planId: string; planVersion: int
               status: enum(none, trialing, active, past_due, canceled, incomplete)
               currentPeriodStart?: <ts>; currentPeriodEnd?: <ts>; cancelAtPeriodEnd: boolean; pendingPlanId?: string | null }
Usage { entitlements: Entitlements
        gauges: { monitorsActive, monitorsTotal, membersCount: int }
        period: { periodStart: <ts>, periodEnd: <ts>, auditsStarted, pagesCrawled,
                  reportsGenerated, aiRequests, emailsSent, externalLinkChecks: int } }

MonitorTest {
  id: string; status: enum(queued, running, completed, failed)
  target: { monitorId?: string } | { config: MonitorTestConfig }
  result?: { ok: boolean, httpStatus?: int | null, latencyMs?: int | null, errorCategory?: string | null,
             redirectCount?: int, certificate?: { notAfter, daysRemaining, hostnameMatches, chainValid } }
  createdAt: <ts>; finishedAt?: <ts>
}                                 // proposed new resource (§15); tests never create checks or incidents
```

Example `Monitor` (illustrative):

```json
{
  "id": "Mn8vB2qX7rTc4LdZ9aPe", "workspaceId": "Wk3pQ9xT2mR7vLc4HnZa",
  "name": "Marketing site", "type": "website", "url": "https://example.com/", "host": "example.com",
  "method": "GET", "headerNames": [], "hasBody": false,
  "expectedStatusRanges": [{ "from": 200, "to": 399 }],
  "intervalSec": 300, "timeoutSec": 10, "maxRedirects": 5, "failureThreshold": 3,
  "status": "active", "state": "up", "stateSince": "2026-10-21T09:05:00Z", "consecutiveFailures": 0,
  "lastCheck": { "checkId": "1792584000", "scheduledAt": "2026-10-21T12:00:00Z", "ok": true, "httpStatus": 200, "latencyMs": 182 },
  "openIncidentId": null, "nextCheckAt": "2026-10-21T12:05:00Z", "sslEnabled": true,
  "certState": { "fingerprint32": "4d9a…", "notAfter": "2027-01-15T00:00:00Z", "status": "valid" },
  "configVersion": 1, "createdBy": "7fK2…", "createdAt": "2026-10-21T09:00:00Z", "updatedAt": "2026-10-21T12:00:01Z"
}
```

---

## 11. Endpoint reference

**Defaults for every endpoint** (not repeated):
- **Auth:** Bearer ID token ([§3.1](#31-authentication-every-non-public-request)).
- **Standard errors (SE):** `401 unauthenticated`, `429 rate_limited`, `500 internal_error`, `503 dependency_unavailable`.
- **Tenant errors (TE)** on every `{ws}` endpoint: `404 not_found` (non-member/unknown/cross-tenant), `403 workspace_suspended`/`workspace_deleting` on non-exempt routes ([§3.4](#34-workspace-state-gates)). Role failures return `403 forbidden`.
- **Tenant isolation:** rules T1–T10 ([§9](#9-tenant-isolation-rules)); each endpoint lists only what it adds.
- **Rate-limit class:** listed per endpoint (`R`, `W`, `X`, `A`, `ADM`).
- **Pagination/idempotency:** stated per endpoint; "n/a" means not applicable.

---

### 11.1 Authentication verification and identity

#### EP-ME-1 `GET /v1/me`
- **Purpose:** Verify the token and return the caller's profile and workspaces (used at app start).
- **Roles:** any authenticated user (no workspace; email verification not required).
- **Request:** none.
- **Success `200`:** `{ user: User, workspaces: { workspaceId, name, role, status }[] }` (≤ 50, newest membership first; larger sets paginate via `GET /v1/workspaces`).
- **Errors:** SE.
- **Pagination:** none (bounded to 50). **Idempotency:** n/a (safe). **Limits:** class `R`.
- **Tenant:** workspaces come from the caller's own memberships only (collection-group `members.uid == caller`, server-side). The caller's `users` document is the only user data returned.

#### EP-ME-2 `PUT /v1/me`
- **Purpose:** Bootstrap/sync the profile after sign-in. Idempotent upsert of the `users` document from the verified token. (Replaces the `POST /me` wording in IMPLEMENTATION_PLAN P1-04; see §15.)
- **Roles:** any authenticated user.
- **Request:** `{ displayName?: string(1–100), locale?: string(BCP-47), timeZone?: string(IANA) }`. `email` and `emailVerified` are **always taken from the token**; supplying them → `422 unknown_field`.
- **Success:** `201` on first creation, `200` thereafter, body `User`.
- **Errors:** `422 validation_failed`; SE.
- **Idempotency:** naturally idempotent (PUT). **Limits:** class `A`.
- **Tenant:** operates only on `users/{callerUid}`.

#### EP-ME-3 `PATCH /v1/me`
- **Purpose:** Update profile fields.
- **Request:** any of `{ displayName?: string | null, locale?: string | null, timeZone?: string | null, defaultWorkspaceId?: string | null }`. `defaultWorkspaceId` must be a workspace the caller belongs to (else `422 invalid_reference`).
- **Success `200`:** `User`. **Errors:** `404` if profile not bootstrapped (`not_found`); `422`; SE. **Limits:** `W`.
- **Tenant:** self only; `email`/`status` not writable.

#### EP-ME-4 `DELETE /v1/me`  *(optional for MVP ⚠ Q-A12)*
- **Purpose:** Delete the caller's account and profile.
- **Preconditions:** caller must not be the owner of any workspace (transfer or delete first) → `409 sole_owner` listing blocking workspaces (ids/names the caller owns).
- **Success:** `202` (account removal runs async; memberships removed; opaque `uid` references in audit logs remain). **Limits:** `W`. **Idempotency:** repeated call after completion → `401`.
- **Tenant:** removes the caller's memberships only.

#### EP-ME-5 `GET /v1/workspaces/{workspaceId}/access`
- **Purpose:** Workspace authorization introspection for UI gating (cosmetic; not a security boundary).
- **Roles:** any member. **Exempt from suspended gate.**
- **Success `200`:** `{ workspaceId, role, workspaceStatus, emailVerified: boolean, permissions: string[] /* e.g. "monitor:create" */, entitlements: Entitlements }`.
- **Errors:** TE; SE. **Limits:** `R`. **Tenant:** T1–T2; derived from live membership.

---

### 11.2 Workspaces and membership

#### EP-WS-1 `POST /v1/workspaces`
- **Purpose:** Create a workspace; caller becomes owner. Creates workspace, owner membership, `usage/current`, and free-plan entitlements in one transaction.
- **Roles:** any authenticated user. Per-user workspace limit ⚠ Q-A15 (spec Q-2).
- **Request:** `{ name: string(1–80, trimmed) }`.
- **Success `201`:** `Workspace` + `Location: /v1/workspaces/{id}`.
- **Errors:** `403 email_not_verified`; `403 plan_limit_reached` (workspace-per-user cap, if any); `422`; SE.
- **Idempotency:** `Idempotency-Key` supported (recommended). **Limits:** `W`.
- **Tenant:** creates a new tenant; no cross-tenant data touched.

#### EP-WS-2 `GET /v1/workspaces`
- **Purpose:** List the caller's workspaces.
- **Query:** `limit` (≤ 50), `cursor`. **Order:** membership creation, newest first.
- **Success `200`:** list of `{ id, name, role, status }`. **Errors:** `400` bad cursor; SE. **Limits:** `R`.
- **Tenant:** server-only collection-group query on `members.uid == caller`; each result re-validated against its path (rule R4 of the schema).

#### EP-WS-3 `GET {ws}`
- **Roles:** any member. **Exempt from suspended gate.** **Success `200`:** `Workspace`. **Errors:** TE; SE. **Limits:** `R`.

#### EP-WS-4 `PATCH {ws}`
- **Roles:** O, A. **Request:** `{ name?: string(1–80) }` (only `name` is writable). **Success `200`:** `Workspace`. **Errors:** `403 forbidden`; `412 stale_version`; `422`; TE; SE. **Limits:** `W`. **Tenant:** T7 (ownerUid/planId/entitlements/status not writable). Writes `auditEvents`.

#### EP-WS-5 `POST {ws}/ownership-transfer`
- **Purpose:** Transfer ownership to an existing member (atomic).
- **Roles:** O only. **Request:** `{ newOwnerUid: string }`. Must be a current member (`422 invalid_reference` otherwise).
- **Success `200`:** `{ ownerUid, previousOwnerRole: "admin" }` (previous owner becomes admin).
- **Errors:** `403 forbidden`; `409 state_conflict` if already owner; `422`; TE; SE. **Idempotency:** retry with same target is a no-op `200`. **Limits:** `W`.
- **Tenant:** transaction updates `workspaces.ownerUid` and two member documents; invariant "exactly one owner" holds at every commit. Audited.

#### EP-WS-6 `DELETE {ws}`
- **Purpose:** Request workspace deletion (grace period, then irreversible cascade).
- **Roles:** O only. **Request:** `{ confirmName: string }` must equal the workspace name (`422 validation_failed` otherwise).
- **Success `202`:** `{ status: "deleting", deletionScheduledFor: <ts> }`. Workspace immediately rejects other requests (`workspace_deleting`).
- **Errors:** `403 forbidden`; `409 state_conflict` if already deleting; TE; SE. **Idempotency:** repeated → `409 state_conflict`/`403 workspace_deleting`. **Limits:** `W`. **Async:** cascade job (DATABASE_SCHEMA §7).
- **Tenant:** T1–T2; the cascade deletes only paths under the workspace.

#### EP-MB-1 `GET {ws}/members`
- **Roles:** any member. **Query:** `limit` (≤ 100), `cursor`. **Order:** `createdAt` asc.
- **Success `200`:** list of `Member`. **Errors:** TE; SE. **Limits:** `R`. **Tenant:** T3; no email addresses returned.

#### EP-MB-2 `PATCH {ws}/members/{uid}`
- **Purpose:** Change a member's role.
- **Roles:** O, A. Admins cannot modify the owner or grant `owner`; use ownership transfer.
- **Request:** `{ role: enum(admin, member) }`.
- **Success `200`:** `Member`. **Errors:** `403 forbidden` (insufficient privilege); `404 not_found` (not a member); `409 sole_owner` (target is owner); `422`; TE; SE.
- **Idempotency:** same role → no-op `200`. **Limits:** `W`. **Tenant:** T1–T2; transaction preserves invariants; effective on the member's next request; audited.

#### EP-MB-3 `DELETE {ws}/members/{uid}`
- **Purpose:** Remove a member, or leave (when `uid` is the caller).
- **Roles:** O, A for others (A cannot remove the owner); any member may remove **themselves** unless sole owner.
- **Success `204`.** **Errors:** `403 forbidden`; `409 sole_owner`; `404 not_found`; TE; SE.
- **Idempotency:** repeated → `404`. **Limits:** `W`. **Tenant:** removal takes effect on the next request; their authored data keeps the opaque `uid`; audited.

#### EP-MB-4 `POST {ws}/invitations`
- **Roles:** O, A. **Requires** verified email.
- **Request:** `{ email: string(valid, ≤ 254), role: enum(admin, member) }` (never `owner`).
- **Validation:** email format; not already a member; no duplicate pending invitation for that email (`409 conflict`); member quota (`maxMembers`).
- **Success `201`:** `Invitation` (the token is **only** emailed, never returned). `Location` header.
- **Errors:** `403 email_not_verified`, `403 plan_limit_reached`, `409 conflict`, `422`; TE; SE.
- **Idempotency:** `Idempotency-Key` supported. **Limits:** `W`, cap 20 / hour / workspace.
- **Tenant:** T1; invitation stored under the workspace; email goes through the outbox (no address stored in delivery records).

#### EP-MB-5 `GET {ws}/invitations`
- **Roles:** O, A. **Query:** `status` (default `pending`), `limit`, `cursor`. **Order:** `createdAt` desc (index `(status, createdAt desc)`). **Success `200`:** list of `Invitation`. **Limits:** `R`.

#### EP-MB-6 `DELETE {ws}/invitations/{inviteId}`
- **Roles:** O, A. **Success `204`** (status → `revoked`). **Errors:** `404`; `409 state_conflict` if already accepted. **Limits:** `W`.

#### EP-MB-7 `POST /v1/invitations/preview`  *(special auth, [§12.1](#121-unauthenticated--special-auth-endpoints))*
- **Purpose:** Let a recipient see which workspace invited them before signing in.
- **Auth:** none required (token-gated). **Request:** `{ token: string }` (POST so the token never lands in URLs/logs).
- **Success `200`:** `{ workspaceName: string, role: string, expiresAt: <ts>, inviteeEmailHint: "d***@example.com" }` (masked).
- **Errors:** `404 not_found` for unknown/expired/revoked/accepted tokens (identical response); `429` (class `A`, tight per-IP limit); `400`.
- **Tenant:** the token hash is the only key; the response leaks only the minimum above and nothing about members or other invitations.

#### EP-MB-8 `POST /v1/invitations/accept`
- **Auth:** Bearer; **the verified token's email must equal the invitation's `inviteeEmail`** and `emailVerified` must be true.
- **Request:** `{ token: string }`.
- **Success `200`:** `{ workspaceId, role }` (creates `members/{uid}` in a transaction; invitation becomes `accepted`; single-use).
- **Errors:** `404 not_found` (unknown/expired/revoked/used, **or email mismatch**; identical, so tokens can't be probed); `403 email_not_verified`; `409 conflict` if already a member; `400`; SE. **Limits:** `A`.
- **Idempotency:** a second accept with the same token → `404` (single-use); accepting concurrently creates exactly one membership.
- **Tenant:** workspace resolved by token hash (server-only collection-group query); membership created only for the verified caller.

#### EP-MB-9 `GET {ws}/audit-events`
- **Roles:** O, A. **Query:** `action?` (index `(action, at desc)`), `limit`, `cursor`. **Order:** `at` desc. **Success `200`:** list of `{ id, at, actorType, actorUid?, action, targetType, targetId, summary? }`. **Limits:** `R`. **Tenant:** T3.

---

### 11.3 Monitor CRUD

#### EP-MON-1 `POST {ws}/monitors`
- **Roles:** O, A, M. Requires verified email.
- **Request:**

  | Field | Type | Req | Validation |
  |---|---|:--:|---|
  | `name` | string | ✓ | 1–100, trimmed, no control characters |
  | `type` | enum(website, api) | ✓ | |
  | `url` | string | ✓ | absolute `http`/`https`, ≤ 2048, no userinfo, valid host, IDN normalized, port in the allowed set (80/443 default ⚠ Q-25); **SSRF target validation** (all resolved addresses must be public/routable) → else `422 url_blocked_by_policy` |
  | `method` | string | | website: `GET`; api: allow-list (GET, HEAD, POST, PUT, PATCH, DELETE) ⚠; default `GET` |
  | `headers` | map<string,string> | | api only, feature-gated; ≤ 20; name `[A-Za-z0-9-]`; forbidden names (Host, Content-Length, Transfer-Encoding, Connection, …) → `422`; value ≤ 1 KiB; **stored encrypted, write-only** |
  | `requestBody` | string | | api only, feature-gated; ≤ 10 KiB; encrypted |
  | `expectedStatusRanges` | `{from,to}[]` | | ≤ 10; 100–599; default `[{200,399}]` |
  | `intervalSec` | int | ✓ | must be an allowed value ≥ plan minimum (`entitlements.minCheckIntervalSec`; undecided, spec Q-5) → `422 not_allowed_for_plan` |
  | `timeoutSec` | int | | 1–30; default 10 |
  | `maxRedirects` | int | | 0–10; default 5 |
  | `failureThreshold` | int | | 1–10; default 3 |

  Numeric bounds are proposals ⚠ Validate. `status`, `state`, `sslEnabled` (derived from the URL scheme), counters and timestamps are not accepted.
- **Success `201`:** `Monitor` (`state: "pending"`, `nextCheckAt` = now so the first check runs promptly, async) + `Location`.
- **Errors:** `403 email_not_verified`; `403 plan_limit_reached` (monitors); `409 conflict` (duplicate name+URL policy ⚠ optional); `422` (incl. `url_blocked_by_policy`, `not_allowed_for_plan`, `feature_unavailable`); TE; SE.
- **Idempotency:** `Idempotency-Key` supported (recommended). **Limits:** `W`. **Quota:** monitors, atomic with creation (`usage/current` transaction).
- **Tenant:** T1, T7; the monitor is created under the tenant path only; secrets stored in a separate write-only document.
- **Async note:** the first check is a background job; the response does not wait for it.

#### EP-MON-2 `GET {ws}/monitors`
- **Roles:** any member. **Query:** `status?` (active/paused/quota_suspended) **or** `state?` (pending/up/down/unknown): only one of the two, else `422 unsupported_filter_combination`; `limit`, `cursor`.
- **Order:** with `status`: `createdAt` desc; with `state`: `stateSince` desc; default `createdAt` desc (indexes in DATABASE_SCHEMA §6.1).
- **Success `200`:** list of `Monitor`. **Limits:** `R`. **Tenant:** T3, T4. Name search is not supported in the MVP (no index).

#### EP-MON-3 `GET {ws}/monitors/{monitorId}`
- **Roles:** any member. **Success `200`:** `Monitor` + `ETag`. **Errors:** TE; SE. **Limits:** `R`. **Tenant:** T2.

#### EP-MON-4 `PATCH {ws}/monitors/{monitorId}`
- **Roles:** O, A, M.
- **Request:** any subset of `name`, `url`, `method`, `headers` (replaces secret headers; `null` clears), `requestBody`, `expectedStatusRanges`, `intervalSec`, `timeoutSec`, `maxRedirects`, `failureThreshold`. Omitted = unchanged; `null` clears optional fields. Same validation as create; a `url` change re-runs SSRF validation and bumps `configVersion`. `status`/`state` are not writable here.
- **Headers:** optional `If-Match: "<etag>"` → `412 stale_version` on mismatch.
- **Success `200`:** `Monitor`. **Errors:** `404`; `412`; `422`; TE; SE. **Idempotency:** same values → no-op. **Limits:** `W`.
- **Tenant:** T2, T7.

#### EP-MON-5 `POST {ws}/monitors/{monitorId}/pause` and `POST …/resume`
- **Roles:** O, A, M. **Request:** empty body.
- **Success `200`:** `Monitor` (`status: paused` / `active`; `nextCheckAt` cleared / rescheduled). Resume rechecks plan limits (`403 plan_limit_reached` if the monitor now exceeds active-monitor quota under the counting rule, spec Q-6).
- **Errors:** `404`; TE; SE. **Idempotency:** natural (already paused → `200` unchanged). **Limits:** `W`. **Tenant:** T2.

#### EP-MON-6 `DELETE {ws}/monitors/{monitorId}`
- **Roles:** O, A; M ⚠ Q-3.
- **Success `204`** (status → `deleting`, disappears from lists immediately, usage decremented; subcollection cascade is async). **Errors:** `403`; `404`; TE; SE. **Idempotency:** repeated → `404`. **Limits:** `W`. **Tenant:** T2. Open incidents are closed with a `monitor_deleted` event; historical incidents are retained per policy.

#### EP-MON-7 `PUT {ws}/monitors/{monitorId}/secrets`  *(feature-gated: API monitor secrets; cut candidate in the plan)*
- **Roles:** O, A, M. **Request:** `{ headers?: map<string,string>, requestBody?: string }` (same limits as create). **Replace semantics.**
- **Success `200`:** `{ headerNames: string[], hasBody: boolean, secretVersion: int }`. Values are **never** returned by any endpoint.
- **Errors:** `404`; `422`; `422 feature_unavailable`; TE; SE. **Idempotency:** PUT. **Limits:** `W`.
- **Tenant:** T2; stored encrypted in `monitors/{id}/secrets/current` (write-only for the API); secrets never logged or echoed in errors.

#### EP-MON-8 `DELETE {ws}/monitors/{monitorId}/secrets`
- **Roles:** O, A, M. **Success `204`.** Clears stored headers/body. Same error/limit/tenant notes as EP-MON-7.

---

### 11.4 Monitor tests and check history

#### EP-MT-1 `POST {ws}/monitor-tests`  *(proposed new resource, §15)*
- **Purpose:** Run a **single, isolated** check, either of a proposed (unsaved) configuration or of an existing monitor ("check now"), without affecting uptime history, incidents or notifications. **Asynchronous:** the API never fetches the URL itself; a fetch worker does.
- **Roles:** O, A, M. Requires verified email.
- **Request (one of):**
  - `{ monitorId: string }`: uses the saved config and stored secrets.
  - `{ config: { type, url, method?, expectedStatusRanges?, timeoutSec?, maxRedirects? } }`: no secrets accepted on unsaved tests (`headers`/`requestBody` → `422 feature_unavailable`).
  Same URL/SSRF validation as monitor creation.
- **Success `202`:** `MonitorTest` (`status: queued`) + `Location` + `Retry-After: 2`.
- **Errors:** `403 plan_limit_reached` (test quota ⚠ if adopted); `404` (unknown `monitorId` under this tenant; T5); `422` (incl. `url_blocked_by_policy`); `503 feature_disabled`; TE; SE.
- **Idempotency:** `Idempotency-Key` **required**. **Limits:** class `X`, 10 / min / workspace. Tests are counted toward abuse limits to prevent use as a probing tool.
- **Tenant:** T1, T5; test documents live under the workspace with a short TTL; results contain no resolved IPs or internal detail.

#### EP-MT-2 `GET {ws}/monitor-tests/{testId}`
- **Roles:** any member. **Success `200`:** `MonitorTest` (poll until `completed` or `failed`). **Errors:** `404`; TE; SE. **Limits:** `R`. **Caching:** `ETag` supported. **Tenant:** T2. Tests expire (TTL ≈ 1 h) → `404`.

#### EP-MON-9 `GET {ws}/monitors/{monitorId}/checks`
- **Roles:** any member.
- **Query:** `from?`, `to?` (time range on `scheduledAt`), `ok?` (boolean), `limit` (≤ 100), `cursor`. All three filters may combine (index `(ok, scheduledAt desc)` plus the automatic time index). Range larger than a configurable maximum (e.g. 31 days) → `422 range_too_large`.
- **Order:** `scheduledAt` desc.
- **Success `200`:** list of `Check`. Slots with no recorded check are **absent** (clients render gaps as unknown, never as up).
- **Errors:** `404`; `422`; TE; SE. **Limits:** `R`. **Tenant:** T2 (monitor exists under tenant), T3.

#### EP-MON-10 `GET {ws}/monitors/{monitorId}/checks/{checkId}`
- **Roles:** any member. **Success `200`:** `Check`. **Errors:** `404`. **Limits:** `R`. **Tenant:** T2.

#### EP-MON-11 `GET {ws}/monitors/{monitorId}/uptime`
- **Roles:** any member. **Query:** `windows?` comma list from `24h,7d,30d,90d` (default `24h,7d,30d`). Windows beyond the plan's history retention return `uptimePercent: null` with `reason: "outside_retention"`.
- **Success `200`:** `{ monitorId, asOf: <ts>, formulaVersion: string, windows: [{ window, from, to, uptimePercent: number | null, expectedChecks, upChecks, downChecks, unknownChecks, avgLatencyMs?: int | null }] }`. `uptimePercent` has up to 3 decimals and is `null` when there is no data. The formula (treatment of gaps and paused periods) is the one defined in `packages/domain` (HIST-2).
- **Errors:** `404`; `422`; TE; SE. **Limits:** `R`. **Tenant:** T2. Computed from rollups.

#### EP-MON-12 `GET {ws}/monitors/{monitorId}/rollups`
- **Roles:** any member. **Query:** `granularity` (`hour`|`day`, required), `from` and `to` (required). Hard cap 400 buckets per request → `422 range_too_large`. **No cursor** (bounded range).
- **Success `200`:** `{ items: [{ id, granularity, bucketStart, expectedCount, checkCount, upCount, downCount, unknownCount, latencyAvgMs?, latencyMinMs?, latencyMaxMs?, approxP95Ms?, closed }] }`. `approxP95Ms` is a histogram estimate ⚠ Validate (DATABASE_SCHEMA W7 `latencyHistogram`).
- **Errors:** `404`; `422`; TE; SE. **Limits:** `R`. **Tenant:** T2.

---

### 11.5 Incident history and acknowledgement

#### EP-INC-1 `GET {ws}/incidents`
- **Roles:** any member.
- **Query (single filter family, index-backed):** `status?` (open/resolved) **or** `monitorId?` (optionally with `status`), plus optional `from`/`to` on `openedAt` when neither is set; `limit`, `cursor`. A `kind` filter is **not supported** without a new index (§15); `unsupported_filter_combination` otherwise.
- **Order:** `openedAt` desc.
- **Success `200`:** list of `Incident`. **Errors:** `404` (unknown `monitorId` under tenant → `422 invalid_reference`, T5); TE; SE. **Limits:** `R`. **Tenant:** T3, T5.

#### EP-INC-2 `GET {ws}/incidents/{incidentId}`
- **Roles:** any member. **Success `200`:** `Incident`. **Errors:** `404`. **Limits:** `R`. **Tenant:** T2.

#### EP-INC-3 `GET {ws}/incidents/{incidentId}/events`
- **Roles:** any member. **Query:** `limit` (≤ 100), `cursor`. **Order:** `seq` asc. **Success `200`:** list of `IncidentEvent` (≤ 200 per incident). **Errors:** `404`. **Limits:** `R`. **Tenant:** T2.

#### EP-INC-4 `POST {ws}/incidents/{incidentId}/acknowledge`  *(requires schema addition, §15)*
- **Purpose:** Record that a human has seen the incident. **Does not close it, change monitor state, or affect uptime math**; closing remains the worker's job on recovery (INC-1). In the MVP acknowledgement has no effect on notifications (no reminder emails exist); ⚠ Validate (Q-A14) if reminders are added.
- **Roles:** O, A, M.
- **Request:** `{ note?: string(≤ 500) }`. The note may contain personal data; it is stored only on the incident and is not echoed to logs.
- **Success `200`:** `Incident` with `acknowledged: true`, `acknowledgedAt`, `acknowledgedBy`.
- **Errors:** `404`; `409 state_conflict` if the incident is `resolved` (⚠ or allowed with no effect; Q-A14); `422`; TE; SE.
- **Idempotency:** natural. Re-acknowledging an acknowledged incident returns `200` unchanged (first acknowledger and time are preserved). **Limits:** `W`.
- **Tenant:** T2; writes only the acknowledgement fields and appends an `acknowledged` incident event; other incident fields remain worker-owned (T7).

---

### 11.6 SSL status and expiry

#### EP-SSL-1 `GET {ws}/monitors/{monitorId}/ssl`
- **Roles:** any member.
- **Success `200`:**

  ```
  {
    "monitorId": "…", "enabled": true, "status": "valid",     // enum(valid, expiring, expired, invalid, unknown, not_applicable)
    "lastCheckedAt": "<ts>", "nextCheckAt": "<ts>",
    "current": { "fingerprint32": "4d9a…", "subjectCn": "example.com", "issuer": "CN=Example CA",
                 "notBefore": "<ts>", "notAfter": "<ts>", "daysRemaining": 85,
                 "sans": ["example.com"], "sanCount": 1, "hostnameMatches": true,
                 "chainValid": true, "trustError": null, "protocol": "TLSv1.3" },
    "alertThresholdsDays": [30, 14, 7, 1],                    // defaults undecided (spec Q-5)
    "recentAlerts": [ { "thresholdDays": 30, "triggeredAt": "<ts>" } ]
  }
  ```

  `daysRemaining` is computed at response time. HTTP monitors return `status: "not_applicable"`. Before the first inspection `status: "unknown"` and `current` is omitted.
- **Errors:** `404`; TE; SE. **Limits:** `R`. **Tenant:** T2. Contains only public certificate data.

#### EP-SSL-2 `GET {ws}/monitors/{monitorId}/ssl/certificates`
- **Roles:** any member. **Query:** `limit` (≤ 20), `cursor`. **Order:** `lastSeenAt` desc. **Success `200`:** list of certificate observations (the `current` shape plus `firstSeenAt`, `lastSeenAt`, `status`). **Errors:** `404`. **Limits:** `R`. **Tenant:** T2.

#### EP-SSL-3 `GET {ws}/monitors/{monitorId}/ssl/alerts`
- **Roles:** any member. **Query:** `limit` (≤ 20), `cursor`. **Order:** `triggeredAt` desc. **Success `200`:** list of `{ id, fingerprint32, thresholdDays, notAfter, triggeredAt }`. **Limits:** `R`. **Tenant:** T2.

*(A workspace-wide "certificates expiring soon" listing is deferred: it needs an index not yet in the schema, §15.)*

---

### 11.7 Starting an SEO audit; status and results

#### EP-AUD-1 `POST {ws}/audits`
- **Purpose:** Start an audit. **Asynchronous:** validates, reserves quota, enqueues, and returns immediately. Crawling runs in background workers and continues if the browser closes.
- **Roles:** O, A, M. Requires verified email.
- **Request:** `{ startUrl: string, options?: { externalLinkCheck?: boolean } }`.

  | Field | Validation |
  |---|---|
  | `startUrl` | absolute `http`/`https`, ≤ 2048, no userinfo, valid host, allowed port; **SSRF target validation** → `422 url_blocked_by_policy`; domain-ownership verification if adopted (spec Q-11) → `422 domain_unverified` ⚠ |
  | `options.externalLinkCheck` | boolean; feature-gated (cut candidate) → `422 feature_unavailable` |

  `pageCap`, `maxDepth`, `respectRobots` and quota values are **server-decided** and not accepted from the client (`unknown_field`).
- **Success `202`:** `Audit` (`status: "queued"`, `pageCap` = the plan cap **snapshotted now**) + `Location` + `Retry-After: 2`.
- **Errors:** `403 email_not_verified`; `403 plan_limit_reached` (audits per month, if defined); `422`; `503 feature_disabled` (crawler kill switch); TE; SE.
- **Idempotency:** `Idempotency-Key` **required**. **Limits:** class `X` (10 / hour / workspace) **and** plan quota (atomic with creation; usage and the audit document are written in one transaction).
- **Tenant:** T1, T7; audit stored under the workspace; workers re-validate tenant/plan state before each step.

#### EP-AUD-2 `GET {ws}/audits`
- **Roles:** any member. **Query:** `status?` **or** `siteKey?` (not both), `limit`, `cursor`. **Order:** `createdAt` desc. **Success `200`:** list of `Audit`. **Limits:** `R`. **Tenant:** T3.

#### EP-AUD-3 `GET {ws}/audits/{auditId}`
- **Purpose:** Poll status/progress. **Roles:** any member. **Success `200`:** `Audit` + `ETag`; `Retry-After` hint while non-terminal. **Errors:** `404`. **Limits:** `R`. **Tenant:** T2.

#### EP-AUD-4 `POST {ws}/audits/{auditId}/cancel`
- **Roles:** O, A, M. **Request:** empty.
- **Success `202`:** `Audit` with `cancelRequested: true` and `status: "cancelling"`; the crawler stops at the next step boundary and the status becomes `cancelled` (partial-results policy: spec Q-20).
- **Errors:** `404`; `409 state_conflict` if the audit is already terminal; TE; SE. **Idempotency:** natural (already cancelling → `200`/`202` unchanged). **Limits:** `W`. **Tenant:** T2.

#### EP-AUD-5 `GET {ws}/audits/{auditId}/snapshots`
- **Roles:** any member. **Query:** `limit` (≤ 20), `cursor`. **Order:** `computedAt` desc. **Success `200`:** list of `Snapshot` (one per rule-set version analyzed). **Limits:** `R`. **Tenant:** T2.

#### EP-AUD-6 `GET {ws}/audits/{auditId}/snapshots/{snapshotId}`
- **Purpose:** Score and category breakdown. `snapshotId` may be `current` (the audit's current snapshot).
- **Roles:** any member. **Success `200`:** `Snapshot` (immutable; safe to cache by `ETag`). **Errors:** `404` (also while the audit has no snapshot yet); `409 state_conflict` is **not** used for "not ready": returns `404` plus `Audit.status` is how clients know. **Limits:** `R`. **Tenant:** T2.
- **Contract guarantee:** every response carrying a score includes `scoreName` and the `disclaimer` text; the score is a proprietary diagnostic, not a Google score or ranking prediction. Clients must display it with the score.

---

### 11.8 Crawled pages and findings

#### EP-AUD-7 `GET {ws}/audits/{auditId}/pages`
- **Roles:** any member. **Query:** `fetchState?` **or** `statusClass?` (not both; `statusClass` ∈ 0,2,3,4,5), `limit` (≤ 100), `cursor`. **Order:** `discoverSeq` asc.
- **Success `200`:** list of `PageSummary`. Page URLs and text come from third-party sites: treat as untrusted, render as text only. URL substring search is not supported (no index).
- **Errors:** `404`; `422 unsupported_filter_combination`; TE; SE. **Limits:** `R`. **Tenant:** T2, T3.

#### EP-AUD-8 `GET {ws}/audits/{auditId}/pages/{pageId}`
- **Roles:** any member. **Success `200`:** `Page` (bounded `facts`). **Errors:** `404`. **Limits:** `R`. **Tenant:** T2.

#### EP-AUD-9 `GET {ws}/audits/{auditId}/snapshots/{snapshotId}/findings`
- **Roles:** any member. **`snapshotId` may be `current`.**
- **Query (one filter family):** exactly one of `category`, `ruleId`, `pageId`, or none; `category` may be combined with `minSeverityRank`. Other combinations → `422 unsupported_filter_combination` (index-backed per DATABASE_SCHEMA §6.1). `limit` (≤ 100), `cursor`.
- **Order:** `severityRank` desc, then rule/ID (stable).
- **Success `200`:** list of `Finding`. Every finding includes non-empty `evidence`. Findings are produced only by the deterministic rule engine; no endpoint creates or edits them.
- **Errors:** `404`; `422`; TE; SE. **Limits:** `R`. **Tenant:** T2, T3, T5 (`pageId` belongs to this audit).

#### EP-AUD-10 `GET {ws}/audits/{auditId}/snapshots/{snapshotId}/findings/{findingId}`
- **Roles:** any member. **Success `200`:** `Finding`. **Errors:** `404`. **Limits:** `R`. **Tenant:** T2.

---

### 11.9 Reports

#### EP-RPT-1 `POST {ws}/reports`
- **Purpose:** Request a PDF or CSV generated from an immutable audit snapshot. **Asynchronous.**
- **Roles:** O, A, M.
- **Request:** `{ auditId: string, snapshotId?: string /* default "current" */, format: enum(pdf, csv) }`.
- **Validation:** audit and snapshot belong to this workspace (T5) and the audit is `completed`; else `409 state_conflict` (not finished) or `422 invalid_reference`.
- **Success `202`:** `Report` (`status: pending`) + `Location` + `Retry-After: 2`.
- **Errors:** `403 plan_limit_reached` (`maxReportsPerMonth`); `409 state_conflict`; `422`; `503 feature_disabled`; TE; SE.
- **Idempotency:** `Idempotency-Key` **required**; the report ID derives from it, so retries return the same report. **Limits:** class `X`, 20 / hour / workspace, plus quota.
- **Tenant:** T1, T5; output stored in private storage under the workspace prefix; the API never returns bucket or object names.

#### EP-RPT-2 `GET {ws}/reports`
- **Roles:** any member. **Query:** `auditId?`, `limit`, `cursor`. **Order:** `createdAt` desc (index `(auditId, createdAt desc)`). **Success `200`:** list of `Report`. **Limits:** `R`. **Tenant:** T3.

#### EP-RPT-3 `GET {ws}/reports/{reportId}`
- **Purpose:** Poll generation status. **Roles:** any member. **Success `200`:** `Report` + `ETag`. **Errors:** `404`. **Limits:** `R`. **Tenant:** T2.

#### EP-RPT-4 `POST {ws}/reports/{reportId}/downloads`
- **Purpose:** Obtain a **short-lived, single-purpose download URL**. Authorization is re-evaluated **on every call**; access is never inferred from knowing a report ID or storage path.
- **Roles:** O, A, M (⚠ spec Q-3). **Request:** empty.
- **Preconditions:** report belongs to the workspace and `status == ready` and not expired; else `404 not_found` (cross-tenant/unknown) or `409 state_conflict` (not ready / `expired`).
- **Success `201`:** `ReportDownload` with `method: "signed_url"`, `url` (valid for minutes, e.g. 300 s ⚠ Validate) and `expiresAt`. The URL is returned **only here**; it is never stored, logged or listed. Response headers: `Cache-Control: no-store`. The object is served with `Content-Disposition: attachment`, explicit `Content-Type`, `nosniff`.
- **Alternative (⚠ Q-A4 / ARCHITECTURE S-3):** `method: "stream"` where the client fetches `GET {ws}/reports/{reportId}/content` with its bearer token. The contract allows either; the choice is made after spike S-3.
- **Errors:** `404`; `409 state_conflict`; TE; SE. **Idempotency:** each call creates a new download record and URL (not idempotent by design; rate-limited). **Limits:** class `X`-light (e.g. 30 / min / user). **Tenant:** T2; a download record is written (who, when).

#### EP-RPT-5 `GET {ws}/reports/{reportId}/downloads`
- **Roles:** O, A. **Query:** `limit`, `cursor`. **Order:** `at` desc. **Success `200`:** list of `{ id, at, actorUid, method }` (no URLs). **Limits:** `R`. **Tenant:** T2.

#### EP-RPT-6 `DELETE {ws}/reports/{reportId}`
- **Roles:** O, A. **Success `204`:** the file is deleted and status becomes `deleted`. **Errors:** `404`. **Idempotency:** repeated → `404`. **Limits:** `W`. **Tenant:** T2.

---

### 11.10 Notification preferences

#### EP-NTF-1 `GET {ws}/notification-settings`
- **Roles:** O, A. **Success `200`:** `NotificationSettings` (workspace default) + `ETag`. If none stored, returns defaults (owner as recipient). **Limits:** `R`. **Tenant:** T2.

#### EP-NTF-2 `PUT {ws}/notification-settings`
- **Roles:** O, A.
- **Request:** `NotificationSettings` fields: `memberRecipientUids` (≤ 50; each must be a current member, T5), `eventToggles` (keys from the known event set), `enabled`, `minIntervalSec?`. `externalRecipients` must be empty in the MVP (`422 feature_unavailable`) ⚠ spec/DB V-10.
- **Success `200`:** `NotificationSettings`. **Errors:** `412 stale_version` (optional `If-Match`); `422`. **Idempotency:** PUT. **Limits:** `W`. **Tenant:** T1, T5; audited.

#### EP-NTF-3 `GET|PUT|DELETE {ws}/monitors/{monitorId}/notification-settings`
- Per-monitor override. Same shapes, roles (O, A) and validation as EP-NTF-1/2. `DELETE` removes the override (`204`); `GET` returns the effective settings with `source: "monitor" | "workspace"`. **Errors:** `404` (unknown monitor under tenant). **Limits:** `R`/`W`. **Tenant:** T2, T5.

#### EP-NTF-4 `GET {ws}/notification-preferences`
- **Purpose:** The caller's **own** opt-outs in this workspace. **Roles:** any member (self only). **Success `200`:** `NotificationPreferences`. **Limits:** `R`. **Tenant:** the document key is the caller's uid; there is no way to address another user's preferences.

#### EP-NTF-5 `PUT {ws}/notification-preferences`
- **Roles:** any member (self only). **Request:** `{ mutedTypes: string[] }`, non-mandatory event types only; mandatory security/account mail types → `422 invalid_enum`. **Success `200`.** **Limits:** `W`. **Tenant:** self only.

#### EP-NTF-6 `POST {ws}/notification-settings/test`
- **Purpose:** Send a test email to the caller's own verified address (NOTIF-9).
- **Roles:** any member. **Request:** empty (the recipient is always the caller; arbitrary addresses are not accepted).
- **Success `202`:** `{ notificationId: string, status: "pending" }`.
- **Errors:** `403 email_not_verified`; `503 feature_disabled` (email kill switch); TE; SE. **Idempotency:** `Idempotency-Key` supported; each call is a distinct notification otherwise. **Limits:** class `X`, 5 / hour / user.
- **Tenant:** T1; the notification record lives under the workspace and stores no email address.

#### EP-NTF-7 `GET {ws}/notifications`
- **Purpose:** Delivery history/failures. **Roles:** O, A. **Query:** `status?`, `limit`, `cursor`. **Order:** `createdAt` desc (index `(status, createdAt desc)`). **Success `200`:** list of `Notification` (no addresses). **Limits:** `R`. **Tenant:** T3.

---

### 11.11 Subscription and usage

#### EP-BIL-1 `GET /v1/plans`
- **Auth:** any authenticated user. **Success `200`:** `{ items: Plan[] }` (active plans, ≤ 20, `provisional` reflects the unvalidated pricing hypotheses: **clients must not present provisional plans as committed offers**). **Limits:** `R`. **Tenant:** global configuration; no tenant data.

#### EP-BIL-2 `GET {ws}/usage`
- **Roles:** any member. **Exempt from suspended gate.** **Success `200`:** `Usage` (current gauges and the current period's counters next to entitlements). **Limits:** `R`. **Tenant:** T2; counters are the authoritative enforcement counters.

#### EP-BIL-3 `GET {ws}/usage/periods`
- **Roles:** any member. **Query:** `limit` (≤ 12). **Order:** `periodStart` desc. **Success `200`:** list of period counters. **Limits:** `R`.

#### EP-BIL-4 `GET {ws}/subscription`
- **Roles:** O. **Success `200`:** `Subscription`. **Provider customer/subscription references are never returned.** For a Free workspace with no provider subscription, `status: "none"`. **Limits:** `R`. **Tenant:** T2.

#### EP-BIL-5 `GET {ws}/subscription/events`
- **Roles:** O. **Query:** `limit` (≤ 50), `cursor`. **Order:** `appliedAt` desc. **Success `200`:** list of `{ id, type, result, fromPlanId?, toPlanId?, appliedAt }`. **Limits:** `R`.

#### EP-BIL-6 `POST {ws}/subscription/checkout-sessions`  *(only when a payment provider is configured)*
- **Purpose:** Create a provider-hosted checkout session for a plan change. Card data never touches UptimePilot (BILL-7).
- **Roles:** O. **Request:** `{ planId: string, returnPath: string }`. `planId` must be an active, non-current plan; `returnPath` must be a relative app path on an allow-list (no open redirects, no absolute URLs) → `422 invalid_format`.
- **Success `201`:** `{ id: string, url: string, expiresAt: <ts> }` (hosted page URL).
- **Errors:** `503 billing_unavailable` (no provider configured or unreachable); `409 state_conflict` (already on that plan / change pending); `422`; TE; SE.
- **Idempotency:** `Idempotency-Key` **required**. **Limits:** class `X`.
- **Important:** this endpoint **does not change the plan**. Plan state changes only when a **verified webhook** is processed (BILL-8). The client must not assume success from the redirect.

#### EP-BIL-7 `POST {ws}/subscription/portal-sessions`
- **Purpose:** Open the provider's customer portal (payment method, cancel, invoices). **Roles:** O. **Request:** `{ returnPath: string }` (same rules). **Success `201`:** `{ url, expiresAt }`. **Errors:** `503 billing_unavailable`; `409 state_conflict` (no provider subscription to manage); `422`. **Limits:** `X`.

*(There is deliberately no `PUT /subscription` that sets a plan. Admin-assigned plans for the pilot go through the admin API, which is audited.)*

---

### 11.12 Platform administration

**Router:** `/v1/admin/**`, separate from customer routes (T9). **Access:** Bearer token **and** an enabled `platformAdmins` record **and** the admin custom claim **and** MFA satisfied; otherwise `403 admin_required` (never `404`, so staff can tell what's wrong). Admin accounts are provisioned **out-of-band, not through this API**.

**Admin rules (all `/v1/admin` endpoints)**
- **`X-Admin-Reason` header (1–500 chars) is required** on every write and on every read of customer data (workspace contents). Missing → `422 validation_failed` (`required`).
- **Every request is recorded in `adminAuditEvents` (including denials) before success is returned**; if the audit write fails, the operation fails.
- Admin APIs **never return secrets** (monitor secret values, tokens, signed URLs, provider references).
- Rate class `ADM`. Admin endpoints are not exposed to customer clients and are not part of the public API surface.

#### EP-ADM-1 `GET /v1/admin/me`
- **Success `200`:** `{ uid, role: enum(support, operator, admin), mfaSatisfied: true }`. **Errors:** `403 admin_required`.

#### EP-ADM-2 `GET /v1/admin/workspaces`
- **Roles:** PA (any role). **Query:** `workspaceId?` (exact) **or** `status?` **or** `memberUid?` (via server-side collection-group `members`), `limit` (≤ 50), `cursor`. **Success `200`:** list of `{ id, name, status, planId, ownerUid, createdAt }`. **Limits:** `ADM`. **Tenant:** summary fields only; no customer content.

#### EP-ADM-3 `GET /v1/admin/workspaces/{workspaceId}`
- **Roles:** PA. **Success `200`:** summary + `entitlements`, `entitlementsOverride?`, `usage`, `subscription` (without provider refs), counts (members/monitors/audits). Requires `X-Admin-Reason` (customer-data read). **Limits:** `ADM`.

#### EP-ADM-4 `GET /v1/admin/workspaces/{workspaceId}/monitors` and `/audits`
- **Roles:** PA `operator`/`admin` ⚠. Read-only lists (same shapes and pagination as the customer endpoints; secrets omitted). `X-Admin-Reason` required; logged as `customerData.view`. **Limits:** `ADM`.

#### EP-ADM-5 `POST /v1/admin/workspaces/{workspaceId}/suspend` and `POST …/reactivate`
- **Roles:** PA `operator`/`admin`. **Request:** `{ reasonCode: string, note?: string(≤ 500) }`. **Success `200`:** `Workspace`. Suspension stops dispatching new checks/steps for the workspace and applies the §3.4 gate. **Errors:** `404`; `409 state_conflict`; `422`. **Idempotency:** natural. **Audited.**

#### EP-ADM-6 `PUT /v1/admin/workspaces/{workspaceId}/plan`
- **Purpose:** Assign a plan administratively (pilot / manual billing). **Roles:** PA `admin`. **Request:** `{ planId: string }`. **Success `200`:** `Subscription`/`Workspace` summary with recomputed entitlements. **Errors:** `404` (workspace or plan); `422`. **Idempotency:** PUT. **Audited.**

#### EP-ADM-7 `PUT /v1/admin/workspaces/{workspaceId}/entitlements-override` and `DELETE …`
- **Roles:** PA `admin`. **Request (PUT):** a partial `Entitlements` object. **Success:** `200` (effective entitlements) / `204`. Overrides are explicit, visible in admin views, and audited.

#### EP-ADM-8 `GET /v1/admin/plans`, `PUT /v1/admin/plans/{planId}`, `POST /v1/admin/plans/{planId}/apply`
- **Roles:** PA `admin`. `PUT` creates a new plan `version` (`{ displayName, status, price, quotas, provisional }`); existing workspaces are **not** changed. `POST …/apply` recomputes entitlements for workspaces on that plan: **async job**, `202` with a status resource (`GET /v1/admin/jobs/{jobId}`). **Errors:** `422` (a `null` quota is allowed only if `provisional: true`); `404`. **Audited.**

#### EP-ADM-9 `GET /v1/admin/system/health`
- **Roles:** PA. **Success `200`:** `{ asOf, dispatcher: { lagSec, lastTickAt }, queues: { name, depth, oldestAgeSec }[], deadLetters: int, notificationFailures24h: int, killSwitches: { name, enabled }[] }`. Source: metrics store ⚠ undecided. **Limits:** `ADM`.

#### EP-ADM-10 `PUT /v1/admin/kill-switches/{name}`
- **Roles:** PA `admin`. **Request:** `{ enabled: boolean }`. Names: `dispatcher`, `crawler`, `email`, `ai`, `reports`, `signups`. **Success `200`.** Effects surface to customers as `503 feature_disabled`. **Audited.**

#### EP-ADM-11 `GET|PUT|DELETE /v1/admin/blocked-hosts[/{hostHash}]`
- **Roles:** PA `operator`/`admin`. `PUT` body: `{ host: string, matchType: enum(exact, suffix), reason: string, expiresAt?: <ts> | null }`. `GET` lists (≤ 100, cursor). Blocked hosts are consulted by the SSRF-guarded fetcher (ADM-6). **Audited.**

#### EP-ADM-12 `GET /v1/admin/audit-events`
- **Roles:** PA `admin`. **Query:** `workspaceId?` **or** `actorUid?`, `limit` (≤ 100), `cursor`. **Order:** `at` desc. **Success `200`:** list of audit events. Reading the audit log is itself audited.

#### EP-ADM-13 `GET /v1/admin/payment-events` and `POST /v1/admin/payment-events/{eventKey}/retry`
- **Roles:** PA `operator`/`admin`. **Query (GET):** `status?` (e.g. `failed`), `workspaceId?`, `limit`, `cursor`. **Order:** `receivedAt`. **Success `200`:** list of `{ eventKey, provider, type, status, attempts, workspaceId?, lastErrorCode? }` (no payloads). **Retry:** re-queues processing of a `failed` event (`202`); idempotent because processing is idempotent. **Audited.**

#### EP-ADM-14 `GET /v1/admin/jobs/{jobId}`
- **Success `200`:** `{ id, kind, status: enum(queued, running, completed, failed), progress?, startedAt?, finishedAt? }` for admin-initiated async jobs (plan apply). **Errors:** `404`.

---

### 11.13 Optional: AI explanations (Phase 7)

*Included for completeness only; not required for the MVP. The audit, findings and score work fully without it.*

- `POST {ws}/audits/{auditId}/snapshots/{snapshotId}/findings/{findingId}/explanation`: roles O, A, M; empty body; returns `200` with a cached `{ text, aiGenerated: true, modelId?, generatedAt }` or `202` while generating; `403 plan_limit_reached` (`maxAiRequestsPerMonth`); `503 feature_disabled` when AI is off. `Idempotency-Key` required; class `X`.
- `GET …/explanation`: returns the cached explanation or `404`.
- Contract guarantees: the response is labeled `aiGenerated: true` and is **advisory text only**; it never modifies a finding, severity, evidence or score; failure leaves everything else usable (AI-2, AI-4). Only the finding's rule and evidence are sent to the provider; no secrets or other-tenant data.

---

## 12. Endpoint inventory by authentication type

### 12.1 Unauthenticated / special-auth endpoints

| Endpoint | Auth | Notes |
|---|---|---|
| `GET /healthz`, `GET /readyz` | none | No sensitive detail; not versioned |
| `POST /v1/invitations/preview` | invitation token | EP-MB-7; masked data, uniform `404`, strict per-IP limit |
| `POST /v1/webhooks/payments/{providerKey}` | **signature** | [§13](#13-payment-webhooks-signature-verification-and-duplicates) |
| `POST /v1/webhooks/email/{providerKey}` | **signature** | Bounce/complaint events (Phase 8); same verification and dedupe pattern as §13 |
| `POST /v1/unsubscribe` | signed unsubscribe token | One-click unsubscribe for non-mandatory mail (Phase 8); token embeds scope; never unsubscribes mandatory security mail |

Everything else requires a Bearer ID token.

### 12.2 Internal endpoints (not a client contract)

Called by Cloud Scheduler/queue only, authenticated by **Google-signed OIDC tokens** from expected service accounts with the worker URL as audience (ARCHITECTURE §8.4); never reachable with user tokens (T10). Listed so the surface is explicit; request shapes are internal.

| Endpoint | Caller | Purpose |
|---|---|---|
| `POST /internal/dispatch` | scheduler | Claim due monitors/certs, enqueue tasks |
| `POST /internal/tasks/probe` | queue | Run one check |
| `POST /internal/tasks/ssl` | queue | Inspect a certificate |
| `POST /internal/tasks/crawl-step` | queue | One crawl step |
| `POST /internal/tasks/analyze` | queue | Analyze an audit |
| `POST /internal/tasks/notify` | queue/scheduler | Send a notification |
| `POST /internal/tasks/report` | queue | Generate a report |
| `POST /internal/tasks/monitor-test` | queue | Run a monitor test |
| `POST /internal/tasks/billing-event` | queue | Process a stored payment event |
| `POST /internal/maintenance/*` | scheduler | Reapers, retention, expiry, reconciliation |

Handlers are idempotent and re-validate their payload's tenant, state and quotas before acting.

---

## 13. Payment webhooks: signature verification and duplicates

### 13.1 Endpoint

`POST /v1/webhooks/payments/{providerKey}`

- `providerKey` selects a registered provider adapter. **No provider is assumed to be configured.** An unregistered or unconfigured key → `404 not_found`. With no provider configured, this endpoint accepts nothing.
- **Auth:** none other than the signature. Not browser-callable; no CORS.
- **Content-Type:** `application/json` only (`415` otherwise). **Body cap:** 1 MiB ⚠ Validate (`413` above).
- **Rate class:** `H`.

### 13.2 Signature verification (provider-neutral requirements)

The adapter for each provider implements the provider's documented scheme; the endpoint enforces these rules regardless of provider. Which header names, algorithm and signed-payload layout apply is **provider-specific and is not assumed here**; many providers use an HMAC over a timestamp-prefixed raw body, which is why the rules are stated this way.

1. **Verify before parsing.** The server reads the **raw request bytes exactly as received** and verifies the signature over those bytes (plus any timestamp component the provider specifies). The body is parsed as JSON **only after** verification succeeds. Re-serialized JSON must never be used for verification (a test enforces this).
2. **Required material present:** missing or malformed signature/timestamp header → `401 invalid_signature`.
3. **Constant-time comparison** of the computed and supplied signatures.
4. **Multiple signatures/secrets supported** to allow secret rotation: accept if any currently-valid secret matches; secrets are per environment, held in the secret manager, never in the repo, config files, logs or error output.
5. **Replay window:** where the scheme provides a timestamp, reject events outside a tolerance (default ±5 minutes ⚠ Validate) → `401 invalid_signature`. Where it does not, rely on event-ID deduplication (§13.3).
6. **Failure handling:** respond `401 invalid_signature` with a fixed generic body; **do not** reveal which check failed; **do not** persist the payload; increment an aggregated `securityEvents` counter (`webhook_signature_failed`); log only the request ID, provider key and a reason code (never the body, signature or secret).
7. **Optional network allow-listing** of provider source IPs may be added as defense in depth but is **never a substitute** for signature verification.
8. **Test mode vs live mode:** the adapter and secrets are environment-specific; non-production environments use sandbox credentials only, and a live event arriving in a non-production environment is rejected.

### 13.3 Duplicate and out-of-order handling

After verification:

1. Extract `providerEventId`, `type`, `providerCreatedAt` from the verified payload.
2. Compute `eventKey = {providerKey}__{sha256(providerEventId)[:40]}`.
3. **Create-if-absent** the inbox document `paymentWebhookEvents/{eventKey}` (status `received`; stores `payloadHash` and a **minimal sanitized `summary`**, never the full payload, never card or personal customer data). This single atomic create is the idempotency gate.
   - **Created** → enqueue processing, return `200`.
   - **Already exists** (duplicate delivery or concurrent twin) → do **not** enqueue again; return `200` with `{ "status": "duplicate" }`. Two simultaneous deliveries race on the atomic create; exactly one wins and the other sees "exists".
4. **Respond `200` quickly after persisting**, not after processing, so provider timeouts do not trigger retry storms. Processing is asynchronous.
5. **Processor** (async, idempotent): resolves the workspace from the provider's customer/subscription reference; applies the event inside a transaction that also writes `workspaces/{wid}/subscriptionEvents/{eventKey}` (create-if-absent, a second idempotency layer), updates `subscriptions/current`, and recomputes `entitlements`.
6. **Out-of-order events:** if the event's `providerCreatedAt` is older than the workspace's `lastEventProviderCreatedAt`, the result is `ignored_stale` (recorded, state unchanged).
7. **Unknown event types** are recorded as `ignored` (so deliveries stop being retried) and logged; they never error.
8. **Unresolvable workspace:** the event stays `failed`/`pending` for reconciliation and admin review (EP-ADM-13), not silently dropped.
9. **Retry policy:** processing failures retry with backoff; after the limit the event is `failed` and surfaces in admin views and alerts. Retrying is safe by construction.

### 13.4 Responses

| Condition | Response |
|---|---|
| Verified, new event persisted | `200 {"status":"accepted"}` |
| Verified, duplicate | `200 {"status":"duplicate"}` |
| Signature missing/invalid/stale | `401 invalid_signature` (generic) |
| Unknown/unconfigured provider key | `404 not_found` |
| Wrong content type / too large | `415` / `413` |
| Persistence failed (datastore down) | `503 dependency_unavailable` (provider will retry) |
| Verified event, processing error later | Still `200` at delivery time; failure handled asynchronously |

### 13.5 Security notes

- Billing state changes come **only** from verified events or audited admin actions (BILL-8, AC-39); a forged request cannot change a plan.
- Client-initiated flows (checkout redirect) never change plan state.
- The email-provider webhook (`/v1/webhooks/email/{providerKey}`) uses the same verify-first, create-if-absent pattern for bounces/complaints and updates the suppression list.
- No provider credentials or sample real secrets appear in this document or in tests (tests use generated throwaway secrets).

---

## 14. Contract testing requirements

Applies when implementation begins; each endpoint ID in §11 maps to tests:

1. **Per-endpoint contract tests:** success shape; every documented error status and `code`; validation of each field rule; unknown-field rejection; method and content-type handling.
2. **Auth tests:** missing/expired/wrong-project/unverified-email tokens; the validation precedence in §4.3.
3. **Role matrix:** every endpoint × role (O, A, M, non-member, unauthenticated, removed member, platform admin on customer routes, worker token on user routes).
4. **Cross-tenant matrix** (DATABASE_SCHEMA §11.1): for every `{ws}` endpoint with tenant A's IDs under tenant B's identity: `404`, no data, byte-identical to a nonexistent ID.
5. **Cursor tests:** replay across tenant/route/filter, tamper, over-limit `limit`, unsupported filter combinations.
6. **Idempotency tests:** replay, in-progress, conflicting body, concurrent duplicates; quota counted once.
7. **Quota and rate-limit tests:** `plan_limit_reached` details; `429` with `Retry-After`; limits enforced before expensive work.
8. **Async tests:** `202` + status resource; polling to terminal states; no request ever blocks on a check/crawl/render; jobs complete with no client connected.
9. **SSRF tests:** `url_blocked_by_policy` for the full corpus at monitor create/update, monitor tests and audit start; generic error text; no resolved IPs in responses.
10. **Secret-leak tests:** canary secrets never appear in any response, log, error or audit record.
11. **Webhook tests:** valid, tampered body, re-serialized body, wrong secret, stale timestamp, rotated secret, duplicate delivery, concurrent twins, out-of-order, unknown type, oversized, unconfigured provider.
12. **Admin tests:** non-admin and non-MFA rejection; reason-header requirement; audit record written before success; secrets never returned.
13. **Contract artifact:** an OpenAPI 3.1 description (or equivalent) will be authored with the implementation and checked in CI against the server's behavior ⚠ tooling undecided (Q-A8). None exists today.

Confirmation process: when an endpoint passes these tests, update its section here from "Proposed" to "Confirmed by tests" with the commit and test names.

---

## 15. Deviations, schema additions and open questions

### 15.1 Additions or changes this proposal needs elsewhere

| # | Item | Where |
|---|---|---|
| D-1 | **Incident acknowledgement:** add `acknowledgedAt`, `acknowledgedBy`, `acknowledgementNote` to `incidents`, event type `acknowledged`, and a **narrow user-writable field set** (DATABASE_SCHEMA W8 says incidents are worker-written only) | DATABASE_SCHEMA W8, W9; Q-A |
| D-2 | **`monitorTests`** workspace-scoped collection (`workspaces/{wid}/monitorTests/{testId}`, TTL ≈ 1 h, status/result fields) | DATABASE_SCHEMA (new W-entry); IMPLEMENTATION_PLAN Phase 3 task |
| D-3 | **Expected status as ranges:** the schema's `expectedStatus` (`array<int>`, ≤ 20) cannot hold the default 200–399; replace with `expectedStatusRanges: {from,to}[]` (≤ 10) | DATABASE_SCHEMA W4 |
| D-4 | `PUT /v1/me` replaces the `POST /me` wording | IMPLEMENTATION_PLAN P1-04 |
| D-5 | Incident `kind` filter and a workspace-wide "certificates expiring soon" list need new indexes; both are **excluded** from this contract until the schema adds them | DATABASE_SCHEMA §6 |
| D-6 | Unsubscribe tokens and email-webhook endpoints are not described in the schema beyond `emailSuppressions` | DATABASE_SCHEMA G8 |
| D-7 | Monitor `status` enum here lacks `deleting` (not exposed; hidden immediately on delete) | consistent with W4 |
| D-8 | Kill-switch state storage and `GET /v1/admin/system/health` data source are undecided | ARCHITECTURE §13 |
| D-9 | Rate-limit counter store undecided | DATABASE_SCHEMA V-18 |

### 15.2 Open questions

| ID | Question |
|---|---|
| Q-A1 | Custom error envelope (§4.1) vs RFC 9457 `application/problem+json`? |
| Q-A2 | `403 plan_limit_reached` vs `402` for quota rejections? |
| Q-A3 | Should monitor tests ever write to history (a "check now" that counts)? Current proposal: no, tests are isolated. |
| Q-A4 | Signed URL vs streamed report download (ARCHITECTURE S-3 / Q-17)? |
| Q-A5 | Is `If-Match` optional (last-write-wins) or required on `PATCH`? |
| Q-A6 | Final rate-limit numbers per class and endpoint (V-18). |
| Q-A7 | Should `X-Admin-Reason` be a header or body field; is a ticket reference mandatory? |
| Q-A8 | OpenAPI tooling and where the contract artifact lives (`packages/api_contracts`). |
| Q-A9 | Is `DELETE` of an already-deleted resource `404` (proposed) or `204`? |
| Q-A10 | May suspended workspaces read their own data (export)? |
| Q-A11 | Allowed outbound ports and HTTP methods for API monitors (ARCHITECTURE Q-25). |
| Q-A12 | Is self-service account deletion (EP-ME-4) in the MVP? |
| Q-A13 | Server-sent events for progress after the MVP? |
| Q-A14 | May acknowledged incidents be acknowledged after resolution; do acknowledgements ever suppress notifications once reminders exist? |
| Q-A15 | Which workspace-per-user cap, if any (spec Q-2)? |

### 15.3 Validation items from this document

Numeric limits (body size, string lengths, ranges, rate limits, URL TTLs, timestamps tolerance); rate-limit header names; cache duration of membership lookups; 30 s request cap; masked-email format; the exact field set of `Snapshot` and `Finding` until `docs/SEO_RULES.md` exists; plan-quota semantics while quotas are `null`.

### 15.4 Before implementation

1. Resolve D-1…D-3 in the schema (they affect stored data).
2. Decide Q-A1, Q-A2, Q-A5 (they affect every endpoint's wire format).
3. Add per-endpoint tasks to the implementation plan referencing the `EP-*` IDs.
4. Do not create handlers, routes or an OpenAPI file until the phase that needs them, per one-small-task-per-session.
