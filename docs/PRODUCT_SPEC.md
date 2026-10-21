# UptimePilot Product Specification

| | |
|---|---|
| Status | Draft v0.1 |
| Last updated | 2026-10-21 |
| Related | [`CLAUDE.md`](../CLAUDE.md) (engineering rules), [`docs/ADR/`](ADR/) (architecture decisions) |

> **Nothing in this document exists yet.** The repository contains no application code. Every module below describes intended behavior, not current behavior.

## How to read this document

Statements are tagged by certainty:

- **[C] Confirmed**: stated by the product owner; treat as a requirement.
- **[A] Assumption**: a reasonable working default proposed in this draft; implement only after it is accepted, and change freely if wrong.
- **[Q] Open question**: needs a decision before the affected work starts. Collected in [section 12](#12-open-product-decisions).

Requirement IDs (e.g. `MON-3`) are stable references for tests, tickets and status docs. Unless a requirement is tagged otherwise, it is **[C]** when it follows directly from the brief and **[A]** when it adds detail. Where a module's requirements are tagged as a block, the tag applies to every item in it unless an item carries its own tag.

---

## 1. Product overview and problem statement

### 1.1 Overview [C]

UptimePilot is a SaaS platform that lets teams monitor the health of websites and APIs and audit their technical SEO in one place. It provides:

- Website and API uptime monitoring with history, response times and incidents.
- SSL certificate health monitoring with expiry alerts.
- Technical SEO crawling and audits with a deterministic score and evidence-backed findings.
- AI-assisted explanations and recommendations layered on top of verified findings.
- PDF and CSV reports.
- Email notifications.
- Subscription plans with quotas.

### 1.2 Problem statement [A]

Small teams and agencies typically stitch together a separate uptime tool, a certificate-expiry reminder and an SEO crawler, each with its own login, pricing and alert channel. Outages and expired certificates go unnoticed until customers report them; SEO problems (broken links, missing metadata, blocked pages) accumulate silently. Existing all-in-one suites are priced and scoped for larger organizations.

UptimePilot aims to give small teams one affordable workspace where "is my site up, is its certificate healthy, and is it technically sound for search?" is answered with verifiable evidence and clear next steps.

---

## 2. Target customers [C]

| Segment | Typical need |
|---|---|
| Small SaaS businesses | Know immediately when the app, API or marketing site is down or a certificate is about to expire. |
| Website owners | Simple, low-effort monitoring and a plain-language view of site health. |
| Freelancers | Monitor several client sites cheaply; send clients proof of work. |
| Web development agencies | Monitor many client sites, hand over delivery/maintenance reports. |
| Digital marketing agencies | Run technical SEO audits across clients and produce shareable reports. |

**[A]** The agency segments imply multi-site, multi-person use; the workspace model (module 5.1) is designed around that. Client-facing/white-label output is not an MVP commitment (see Q-9).

---

## 3. Product goals and non-goals

### 3.1 Goals [C unless tagged]

- **G1.** Detect and communicate downtime and certificate problems quickly and reliably.
- **G2.** Produce a technical SEO score and findings that are deterministic, reproducible and backed by evidence.
- **G3.** Use AI only to explain and prioritize verified findings, never to create or alter them.
- **G4.** Provide exportable PDF and CSV reports suitable for sharing.
- **G5.** Support multi-tenant workspaces with strict data isolation.
- **G6.** Offer tiered subscription plans enforced by server-side quotas.
- **G7 [A].** Be simple enough that a new user reaches a first monitored site and first audit result within minutes, without documentation.

### 3.2 Non-goals

- Replacing full enterprise observability (APM, tracing, log management). **[A]**
- Predicting or guaranteeing search rankings. The Technical SEO Score is diagnostic only.
- Anything in [section 11](#11-explicitly-deferred-features).
- Content writing or automatic modification of customers' sites. **[A]**

---

## 4. User roles

Roles are scoped to a workspace except platform administrator, which is global.

| Role | Scope | Summary |
|---|---|---|
| Account owner | Workspace | Owns the workspace and subscription. Exactly one per workspace. |
| Workspace administrator | Workspace | Manages members, monitors, audits, settings. No ownership/billing transfer. |
| Member | Workspace | Works with monitors, audits and reports within permissions below. |
| Platform administrator | Platform | UptimePilot staff operating the service. Not a customer role. |

### 4.1 Permission matrix [A]

The roles are **[C]**; the exact matrix is a proposal pending Q-3.

| Capability | Owner | Admin | Member | Platform admin |
|---|:--:|:--:|:--:|:--:|
| View monitors, incidents, audits, reports | ✓ | ✓ | ✓ | Support access only (audited) |
| Create/edit/delete monitors | ✓ | ✓ | ✓ (create/edit; delete: Q-3) | – |
| Start SEO audits | ✓ | ✓ | ✓ | – |
| Download reports | ✓ | ✓ | ✓ | – |
| Configure notification recipients | ✓ | ✓ | – | – |
| Invite/remove members, change roles | ✓ | ✓ (not owner) | – | – |
| Manage subscription and billing | ✓ | – | – | – |
| Delete workspace / transfer ownership | ✓ | – | – | – |
| Manage plans, suspend accounts, view platform health | – | – | – | ✓ |

All authorization is enforced server-side. The UI may hide controls, but hiding is never the control.

---

## 5. Core modules and functional requirements

### 5.1 User authentication and workspaces

**Purpose.** Identify users and group them into isolated workspaces that own all data.

**Requirements.**

- **AUTH-1.** Users sign up and sign in with Firebase Auth. **[C]** Supported sign-in methods: Q-1.
- **AUTH-2.** Email/password accounts must verify their email address before creating monitors or audits. **[A]**
- **AUTH-3.** Password reset and sign-out are available. Sign-out ends the client session. **[A]**
- **AUTH-4.** On first sign-in a user creates or is placed into a workspace; the creator becomes account owner. **[A]**
- **WS-1.** Every monitor, check result, incident, audit, report, notification setting and subscription belongs to exactly one workspace. **[C]** (tenant isolation)
- **WS-2.** Every API request is authorized server-side against the caller's workspace membership and role. Client-supplied workspace IDs are never trusted on their own. **[C]**
- **WS-3.** Owners/admins invite members by email; invitations expire and are single-use. **[A]**
- **WS-4.** Owners/admins can change a member's role (within the matrix) and remove members. Removal takes effect on the next request. **[A]**
- **WS-5.** Whether a user can belong to multiple workspaces: Q-2. The data model must not preclude it. **[A]**
- **WS-6.** Workspace deletion removes or irreversibly anonymizes its data per the retention policy (Q-8). **[A]**

### 5.2 Website and API monitoring

**Purpose.** Periodically probe a target and decide whether it is up.

**Requirements.**

- **MON-1.** A monitor targets a user-supplied HTTP(S) URL with a name. **[C]**
- **MON-2.** Monitor types: website (HTTP GET, status/response-time check) and API (configurable method, headers, optional body, expected status code). **[C]** that both exist; request-configuration depth is **[A]**. Non-HTTP checks (ping, TCP port, DNS, keyword/content match) are not in the brief: Q-4.
- **MON-3.** Check interval is set per monitor, bounded by the workspace plan. Allowed intervals: Q-5.
- **MON-4.** Each check records: timestamp, outcome (up/down), HTTP status if any, response time, and an error category when failing (DNS failure, connect timeout, TLS error, unexpected status, redirect-limit, blocked-by-policy). **[A]**
- **MON-5.** A single failed check does not immediately declare an incident. A monitor is considered down after N consecutive failures (default and range: Q-5). **[A]**
- **MON-6.** Monitors can be paused and resumed. Paused monitors run no checks and do not count toward alerts; whether they count toward the quota is Q-6. **[A]**
- **MON-7.** **SSRF protection is mandatory** for every user-supplied URL. The system must reject or neutralize: private, loopback and link-local addresses; cloud metadata endpoints; non-HTTP(S) schemes; credentials in URLs; and must re-validate the resolved IP at connection time and on every redirect hop (defending against redirect and DNS-rebinding bypass). Rejections show a clear, non-leaky message. **[C]**
- **MON-8.** Requests use a bounded timeout, bounded redirect count and bounded response-size read. **[A]**
- **MON-9.** The user-agent identifies UptimePilot so site owners can recognize and allow-list it. **[A]**
- **MON-10.** Creating a monitor beyond the plan quota is rejected server-side with an upgrade-oriented message. **[C]** (quotas)
- **MON-11.** API monitor secrets (auth headers/tokens) are stored encrypted, never returned to the client in full after saving, and never logged. **[A]**

### 5.3 Uptime history, response time and incidents

**Requirements.**

- **INC-1.** An incident opens when a monitor transitions to down (per MON-5) and closes on the first successful check (recovery rule configurable: Q-5). **[A]**
- **INC-2.** An incident records start, end, duration, first and last error category and affected monitor. **[A]**
- **HIST-1.** The monitor detail view shows current status, uptime percentage over standard windows (e.g. 24 h, 7 d, 30 d), a response-time chart and a recent check log. **[A]**
- **HIST-2.** Uptime percentage is computed from recorded checks, and the formula (including how paused periods and gaps in checks are treated) is documented and tested. **[A]**
- **HIST-3.** The incident list is filterable by monitor and date range and shows ongoing incidents first. **[A]**
- **HIST-4.** History retention depends on plan: Q-7 (and Q-8 for data-retention policy).
- **HIST-5.** Raw check data may be aggregated for long-range views; aggregation must not change the reported uptime result beyond a documented tolerance. **[A]**

### 5.4 SSL certificate monitoring and expiry alerts

**Requirements.**

- **SSL-1.** For HTTPS monitors the system periodically inspects the served certificate: expiry date, issuer, subject/SAN coverage of the monitored hostname, chain validity and protocol errors. **[C]** for expiry and health; the exact fields are **[A]**.
- **SSL-2.** Expiry alerts fire at configurable thresholds before expiry. Default thresholds (e.g. 30/14/7/1 days): Q-5. **[A]**
- **SSL-3.** Each threshold alerts at most once per certificate per monitor, so alert fatigue is avoided; renewal (new certificate) resets thresholds. **[A]**
- **SSL-4.** An expired, hostname-mismatched or untrusted certificate is reported as a certificate incident distinct from a downtime incident. **[A]**
- **SSL-5.** Certificate inspection obeys the same SSRF controls as MON-7. **[C]**
- **SSL-6.** Certificate check frequency may be lower than uptime frequency (e.g. daily). **[A]**

### 5.5 Technical SEO crawling and audits

**Requirements.**

- **SEO-1.** A user starts an audit by supplying a start URL (typically a site's home page). **[C]**
- **SEO-2.** The crawler fetches pages starting from the start URL, follows internal links on the same site, and stops at the plan's page cap (provisional: 25/50/250/1,000). **[C]** that audits are page-capped; the numbers are provisional ([section 5.10](#510-subscription-plans-quotas-and-billing)).
- **SEO-3.** The crawler honors robots.txt for its own user-agent by default, and discovers sitemap URLs from robots.txt and common locations. **[A]**
- **SEO-4.** The crawler fetches **server-delivered HTML only**; it does not execute JavaScript. **[C]** (full JS-rendered crawling is deferred). Pages whose content depends on client-side rendering are flagged as a known limitation in results and reports. **[A]**
- **SEO-5.** All fetches use the SSRF controls in MON-7, including redirects across hosts. **[C]**
- **SEO-6.** Crawl politeness: bounded concurrency, per-host rate limiting, request timeouts and response-size limits. **[A]**
- **SEO-7.** Audits run asynchronously in background workers; progress (pages crawled / cap) is visible, and the user can cancel a running audit. **[A]**
- **SEO-8.** Each audit stores an immutable snapshot of its results so reports and comparisons are reproducible. **[A]**
- **SEO-9.** A user can re-run an audit on the same site; a comparison with the previous audit (score delta, new/resolved issues) is shown. **[A]** (comparison UI may follow the MVP: Q-10)
- **SEO-10.** Audit attempts count against plan quotas as defined by Q-6. **[A]**
- **SEO-11.** Whether the site owner must prove ownership/control of a domain before auditing: Q-11.

**Checks performed** (the audit's rule set; each rule has a stable ID, severity and category):

| Category | Example checks |
|---|---|
| Metadata and headings | Missing/empty/duplicate/over-long/too-short title; missing/duplicate meta description; missing H1; multiple H1; skipped heading levels |
| Crawlability and indexability | robots.txt presence and validity; pages blocked by robots.txt; `noindex` via meta robots or `X-Robots-Tag`; orphan or unreachable pages |
| HTTP status and broken links | 4xx/5xx pages; broken internal links; broken external links (limited); long redirect chains; redirect loops; soft redirects |
| Canonical URLs and sitemap | Missing/multiple/conflicting canonical; canonical pointing to non-200/redirected URL; sitemap missing, unparseable, listing non-200/noindex/redirected URLs, or missing crawled indexable pages |
| Image accessibility basics | Images missing `alt`; empty `alt` on non-decorative images where determinable |
| HTTPS and technical configuration | Site served over HTTP / no redirect to HTTPS; mixed content; invalid certificate; missing viewport meta; missing `lang` attribute; presence of HTTP→HTTPS canonical consistency |

**[A]** The check list is a proposed starting set. The final rule catalogue (IDs, thresholds such as title length ranges, severities) must be written as its own spec (`docs/SEO_RULES.md`, to be created) before implementation.

### 5.6 Technical SEO score and evidence-backed findings

**The UptimePilot Technical SEO Score is a proprietary diagnostic score computed by UptimePilot's own rules. It is not a Google score, does not represent how Google evaluates a site, and is not a ranking prediction or guarantee.** This disclaimer must appear wherever the score is displayed or exported. **[C]**

**Requirements.**

- **SCORE-1.** The score is an integer 0–100 computed deterministically: the same crawl data and rule version always produce the same score. **[C]**
- **SCORE-2.** The score is a weighted combination of category sub-scores. Proposed initial weights: **[C]** as the initial proposal, subject to validation.

  | Category | Weight |
  |---|---:|
  | Metadata and headings | 20% |
  | Crawlability and indexability | 20% |
  | HTTP status and broken links | 20% |
  | Canonical URLs and sitemap | 15% |
  | Image accessibility basics | 10% |
  | HTTPS and technical configuration | 15% |
  | **Total** | **100%** |

- **SCORE-3.** Each category sub-score (0–100) is derived from the pass/fail results of that category's rules across audited pages. The sub-score formula (per-page vs per-site rules, severity weighting, handling of categories with nothing to evaluate) is **not yet defined**: Q-12. It must be specified and unit-tested before implementation. **[Q]**
- **SCORE-4.** The scoring rule set carries a version. Each audit records the version used; changing weights or rules creates a new version and does not retroactively alter stored audits. **[A]**
- **SCORE-5.** Every finding includes: rule ID, category, severity, affected URL(s), and **evidence** (e.g. the actual title text and length, the observed status code and redirect chain, the offending element/snippet, the fetched robots.txt line). A finding without verifiable evidence must not be produced. **[C]**
- **SCORE-6.** Findings are grouped by category and severity, filterable, and linked to the affected pages. **[A]**
- **SCORE-7.** The score view shows the category breakdown so users can see what drives the number. **[A]**
- **SCORE-8.** Because the audit is capped at N pages, the score reflects only crawled pages; the UI and reports state the number of pages crawled and whether the cap was hit. **[A]**
- **SCORE-9.** Weights are provisional and must be reviewed against real audit data before being presented as stable. **[A]**

### 5.7 AI-assisted recommendations

**Requirements.**

- **AI-1.** AI may explain a finding in plain language, explain why it matters, and suggest how to fix it, and may help prioritize. **[C]**
- **AI-2.** AI **must not** create findings, alter severities, change scores, or assert evidence not present in the verified data. Its input is the verified finding and its evidence; its output is advisory text. **[C]**
- **AI-3.** AI output is visibly labeled as AI-generated and separate from verified evidence. **[A]**
- **AI-4.** If the AI service is unavailable, slow, over quota or returns unusable output, the audit results and score remain fully usable; the recommendation area shows a recoverable error state with retry. **[A]**
- **AI-5.** Only the minimum necessary data is sent to the AI provider (finding rule, evidence, page URL); no account credentials, API-monitor secrets or unrelated workspace data. Provider and data-handling terms: Q-13. **[A]**
- **AI-6.** AI usage is rate-limited and quota-bound per workspace/plan: Q-6. **[A]**
- **AI-7.** Generated recommendations may be cached per (rule, evidence) to control cost and keep output stable between views. **[A]**
- **AI-8.** AI provider and model are not chosen in this document: Q-13.

### 5.8 PDF and CSV reports

**Requirements.**

- **REP-1.** Users can generate and download an SEO audit report as PDF and the underlying findings as CSV. **[C]**
- **REP-2.** Whether uptime/incident/SSL reports are also in the MVP: Q-14. The brief says "PDF and CSV reports" without naming report types. **[Q]**
- **REP-3.** PDF audit report contents: workspace/site name, audit date and scope (pages crawled, cap), the Technical SEO Score with the required disclaimer and rule-set version, category breakdown, findings with evidence, and (if included) clearly-labeled AI recommendations. **[A]**
- **REP-4.** CSV contains one row per finding or per (page, finding), with stable column names, UTF-8 encoding and spreadsheet-injection-safe escaping of cell values (neutralizing leading `=`, `+`, `-`, `@`). **[A]**
- **REP-5.** Generated files are stored in private storage and delivered only through authorized, tenant-scoped, time-limited access. Files are never publicly readable. **[C]**
- **REP-6.** Reports are generated from the audit's immutable snapshot (SEO-8), so a re-download matches the original. **[A]**
- **REP-7.** Generation runs asynchronously for large reports, with status and failure/retry handling. **[A]**
- **REP-8.** Report retention and deletion follow the data-retention policy (Q-8). **[A]**
- **REP-9.** White-label/branded client reports are not committed for the MVP: Q-9.

### 5.9 Email notifications

**Requirements.**

- **NOTIF-1.** Email is the notification channel. **[C]** Other channels (SMS, Slack, webhooks, push) are not in the brief and are not planned: Q-15.
- **NOTIF-2.** Notable events produce email: monitor down, monitor recovered, certificate expiring (per SSL-2), certificate problem, audit completed/failed, report ready (optional), plan-limit warnings, and transactional account emails (verification, invitation, password reset). **[A]**
- **NOTIF-3.** Recipients are configurable per workspace (and optionally per monitor). By default the account owner is notified. **[A]**
- **NOTIF-4.** Down/recovery alerts are deduplicated: one "down" email per incident and one "recovered" email, not one per failed check. **[A]**
- **NOTIF-5.** Emails state what happened, which monitor/site, when, the error category and a link to the relevant page. They never include secrets. **[A]**
- **NOTIF-6.** Users can mute or unsubscribe from non-critical notification types. Security and account emails cannot be disabled. **[A]**
- **NOTIF-7.** Delivery failures are recorded and retried with backoff; persistent failure is surfaced to workspace admins. **[A]**
- **NOTIF-8.** Email provider is not chosen here (deployment/integration decision): Q-16.
- **NOTIF-9.** A user can send themselves a test notification to verify setup. **[A]**

### 5.10 Subscription plans, quotas and billing

**Provisional plans.** The following prices and quotas are **unvalidated pricing hypotheses**. They are **not** validated market facts, have not been tested with customers, and must not be hard-coded as final or presented as committed offers. They must be implemented as configuration, not constants in code. **[C]** (as hypotheses)

| Plan | Price (₹/month) | Monitors | Pages per audit |
|---|---:|---:|---:|
| Free | 0 | 5 | 25 |
| Starter | 699 | 10 | 50 |
| Agency | 1,999 | 50 | 250 |
| Pro | 3,999 | 150 | 1,000 |

**Observation (not a decision).** In this table Starter allows 10 monitors while Free allows 5, but the price gap is large relative to the quota gap, and Agency/Pro differ by 3× in monitors and 4× in pages for ~2× the price. Whether tiers are coherent should be tested in pricing research: Q-17.

**Requirements.**

- **BILL-1.** Each workspace has exactly one active plan. New workspaces start on Free. **[A]**
- **BILL-2.** Quotas (monitors, pages per audit) are enforced server-side at the moment of the action. UI messaging is advisory. **[C]**
- **BILL-3.** Additional plan dimensions are **not** defined by the brief and need decisions: check interval limits, history retention, member seats, audit frequency/count per month, report count, AI usage allowance, notification recipients. All Q-6. **[Q]**
- **BILL-4.** Plan definitions are data/configuration so prices and quotas can change without a code release. **[A]**
- **BILL-5.** Upgrading takes effect immediately. Downgrading when usage exceeds the new plan's quota must have defined behavior (e.g. block until under quota, or pause excess monitors): Q-18. **[Q]**
- **BILL-6.** Billing: payment provider, currency handling, taxes (e.g. GST), invoicing, trials, refunds, annual billing and failed-payment handling are **undecided**: Q-19. No payment integration is assumed to exist or be chosen.
- **BILL-7.** Payment card data must never be handled or stored by UptimePilot's own servers; a hosted payment provider is expected. **[A]**
- **BILL-8.** Subscription state changes are driven by verified provider events, not client claims. **[A]**
- **BILL-9.** The workspace billing page shows current plan, usage versus quota, next renewal and invoices. **[A]**

### 5.11 Platform administration

**Requirements.**

- **ADM-1.** An internal admin area is available only to platform administrators, enforced server-side and separate from customer roles. **[C]** (role exists)
- **ADM-2.** Capabilities (proposed): look up workspaces/users, view plan and usage, adjust or override a plan/quota, suspend/reactivate a workspace, view system health (worker backlog, failed jobs, notification failures), and manage the plan configuration. **[A]**
- **ADM-3.** Every platform-admin action, including any access to customer data, is written to an immutable audit log (who, what, when, workspace). **[A]**
- **ADM-4.** Support access to customer data is minimal and logged; admins cannot read stored secrets (MON-11). **[A]**
- **ADM-5.** Platform administrator accounts require strong authentication (e.g. MFA) and are provisioned out-of-band, not via public sign-up. **[A]**
- **ADM-6.** Abuse controls: ability to disable a workspace or block a target host/domain (e.g. if monitoring is being abused to probe third parties). **[A]**

---

## 6. User journeys

Journeys describe intended behavior; steps in `[brackets]` are alternative or error paths. **[A]** unless a step restates a confirmed requirement.

### 6.1 Onboarding

1. Visitor opens the app and signs up (AUTH-1) or signs in.
2. Email verification is requested (AUTH-2). `[Unverified: monitoring creation is blocked with a clear prompt and a resend link.]`
3. A workspace is created; the user is the account owner (AUTH-4). They land on an empty dashboard.
4. The dashboard offers two clear first actions: **Add a monitor** and **Run an SEO audit**.
5. `[Optional]` Invite teammates (WS-3).
6. Success: the user has a workspace on the Free plan with a visible usage-versus-quota indicator.

### 6.2 Adding a monitor

1. User chooses **Add monitor** and picks website or API (MON-2).
2. Enters name and URL; for API, method/headers/expected status.
3. Client-side format validation; the server performs authoritative validation including SSRF rules (MON-7). `[Rejected URL: inline error explaining that the address isn't allowed, without disclosing internal network details.]`
4. `[Quota reached: server rejects with an upgrade prompt (MON-10).]`
5. User picks interval within plan limits and confirms notification recipients (defaults to owner).
6. Monitor is created; an immediate first check runs and the result appears within a short time with a loading state.
7. For HTTPS URLs the certificate is inspected and shown on the monitor.
8. Success: the monitor shows a status, response time and next check time.

### 6.3 Running an SEO audit

1. User chooses **New audit** and enters a start URL.
2. Server validates the URL (SSRF) and checks audit quota; shows the page cap for the plan (SEO-2).
3. `[Domain verification, if adopted (Q-11).]`
4. Audit is queued, then runs asynchronously; the user sees queued → crawling (progress) → analyzing → complete. They may leave the page; an email arrives on completion (NOTIF-2).
5. `[Failure: start URL unreachable or blocked by robots.txt → audit ends with a specific, recoverable error and a retry action.]`
6. `[Cancel: user cancels; partial results policy: Q-20.]`
7. Success: the audit result page shows the Technical SEO Score, category breakdown, pages crawled and the disclaimer (5.6).

### 6.4 Reviewing issues

1. From the audit result, user opens a category or the full findings list.
2. Filters by severity/category/URL. Each finding shows evidence (SCORE-5).
3. User opens a finding: sees what was observed, which pages are affected and the verified evidence.
4. User requests/views an AI explanation (AI-1); it is labeled AI-generated and shown apart from the evidence. `[AI unavailable: finding remains fully usable; retry offered (AI-4).]`
5. `[Optional]` Mark a finding as ignored/accepted: Q-21.
6. After fixing the site, user re-runs the audit and sees the delta (SEO-9).

### 6.5 Downloading a report

1. From an audit, user chooses **Download report** → PDF or CSV.
2. If not already generated, generation starts (REP-7) with a visible progress/pending state.
3. When ready, the user receives a time-limited, authorized download link (REP-5).
4. `[Generation failed: error message with retry; audit data unaffected.]`
5. Success: the file opens, shows the score disclaimer, version and scope (REP-3).

### 6.6 Receiving alerts

1. A monitor fails N consecutive checks (MON-5); an incident opens (INC-1).
2. Recipients receive one "down" email with monitor, time, error category and a link (NOTIF-4, NOTIF-5).
3. Monitor recovers; the incident closes; one "recovered" email states duration.
4. Separately, when a certificate crosses an expiry threshold, a one-time expiry email is sent (SSL-2, SSL-3).
5. User opens the link and lands on the incident/monitor detail page (authentication required; workspace authorization enforced).
6. `[User not signed in: sign-in then redirect back to the target.]`
7. `[User lacks access to that workspace: generic not-found/forbidden without revealing existence.]`

---

## 7. Functional requirements by module

The requirements for each module are listed with that module in [section 5](#5-core-modules-and-functional-requirements), using the IDs `AUTH`/`WS`, `MON`, `INC`/`HIST`, `SSL`, `SEO`, `SCORE`, `AI`, `REP`, `NOTIF`, `BILL`, `ADM`. They are kept in one place to avoid duplication and drift. Cross-cutting rules that apply to every module:

- **X-1.** Every data access and mutation is tenant-scoped and authorized server-side (WS-1, WS-2). **[C]**
- **X-2.** Every outbound fetch of a user-supplied URL uses the shared SSRF-protected client (MON-7). **[C]**
- **X-3.** Quotas are checked server-side (BILL-2). **[C]**
- **X-4.** Background jobs are idempotent and safe to retry. **[A]**
- **X-5.** Important state changes (member changes, plan changes, deletions, admin actions) are audit-logged. **[A]**
- **X-6.** No secrets in source, logs, error messages, emails or reports. **[C]**

---

## 8. MVP acceptance criteria

The MVP scope itself is not fully defined (Q-22). The criteria below assume the modules in section 5 minus anything marked as pending a decision. Each is written to be testable. **[A]**

### Authentication and workspaces
- AC-1. A new user can sign up, verify their email and sign in; an unverified user cannot create a monitor or audit.
- AC-2. After sign-up, a workspace exists and the user is its account owner.
- AC-3. A user in workspace A requesting any resource ID belonging to workspace B receives a not-found/forbidden response and no data, for every resource type (monitor, incident, audit, report).
- AC-4. A member cannot perform owner/admin-only actions (per section 4.1); the server rejects them even if the request is crafted manually.
- AC-5. A removed member's next request to the workspace is rejected.

### Monitoring
- AC-6. Creating a website monitor with a valid public HTTPS URL succeeds and records a first check.
- AC-7. Creating a monitor whose URL resolves to a loopback, private, link-local or cloud-metadata address is rejected.
- AC-8. A monitored URL that returns a redirect to a private/metadata address is not followed, and the check records a blocked-by-policy failure.
- AC-9. A hostname whose DNS answer changes to a private IP between validation and connection is not connected to (rebinding test).
- AC-10. A URL containing credentials or a non-HTTP(S) scheme is rejected.
- AC-11. With N consecutive failing checks an incident opens exactly once; the first success after closes it exactly once.
- AC-12. A single isolated failure does not open an incident.
- AC-13. Creating a monitor beyond the plan quota is rejected server-side regardless of client behavior.
- AC-14. A paused monitor performs no checks and sends no alerts.
- AC-15. Uptime percentage for a seeded history matches a hand-calculated expected value.

### SSL
- AC-16. A certificate expiring within a threshold triggers one email for that threshold and no duplicates on subsequent checks.
- AC-17. An expired or hostname-mismatched certificate is reported as a certificate incident.

### SEO audit and score
- AC-18. An audit never fetches more pages than the plan's page cap.
- AC-19. Running the same audit rules on identical crawl data twice yields identical findings and score.
- AC-20. Every finding contains non-empty evidence taken from the crawled data; a rule with no evidence produces no finding.
- AC-21. The score equals the weighted sum of category scores using the active weight set, and the audit records the rule-set version.
- AC-22. Changing the rule-set version does not change the score of previously stored audits.
- AC-23. The score and exports display the proprietary-score disclaimer.
- AC-24. The crawler does not fetch pages disallowed by robots.txt (default behavior).
- AC-25. Crawl redirects to private addresses are blocked as in AC-8.
- AC-26. A user can cancel a running audit and it stops fetching.

### AI
- AC-27. The AI explanation for a finding never changes that finding's severity, evidence or the audit score.
- AC-28. When the AI service fails, the audit result, score and findings still render and a retry option is shown.
- AC-29. AI-generated text is visibly labeled.
- AC-30. Requests to the AI provider contain no secrets or data from other workspaces.

### Reports
- AC-31. A PDF and a CSV can be generated and downloaded for a completed audit.
- AC-32. A report file cannot be fetched by an unauthenticated user or a user in another workspace, and its storage location is not publicly accessible.
- AC-33. A CSV cell beginning with `=`, `+`, `-` or `@` is neutralized.
- AC-34. Re-downloading a report yields content consistent with the original audit snapshot.

### Notifications
- AC-35. A down incident produces exactly one down email and one recovery email.
- AC-36. Emails contain no secrets.
- AC-37. A failed email send is retried and eventually recorded as failed.

### Billing and plans
- AC-38. Plan limits come from configuration; changing the configured limit changes enforcement without a code change.
- AC-39. A workspace's plan changes only via verified billing events or authorized admin action (pending Q-19).

### Platform administration
- AC-40. A non-platform-admin cannot access admin endpoints.
- AC-41. Platform-admin actions create audit-log entries.

### Engineering
- AC-42. Security controls (SSRF, tenant isolation, authorization, CSV escaping) are covered by automated tests, and formatter, analyzer and tests pass in CI. (See `CLAUDE.md`.)

---

## 9. Non-functional requirements

### 9.1 Security
- **NFR-S1.** Server-side authorization and tenant isolation for every request **[C]**; Firestore security rules (or equivalent) defend in depth and are tested.
- **NFR-S2.** SSRF protections per MON-7 on every outbound fetch **[C]**.
- **NFR-S3.** No credentials, keys, tokens or production secrets in source control; secrets are in a managed secret store **[C]**.
- **NFR-S4.** Secrets at rest (e.g. API monitor headers) are encrypted; never logged **[A]**.
- **NFR-S5.** Rate limiting and abuse protection on sign-up, auth-adjacent and expensive endpoints (audit start, report generation, AI) **[A]**.
- **NFR-S6.** Standard web hygiene: HTTPS only, secure headers, input validation, output encoding, dependency vulnerability scanning **[A]**.
- **NFR-S7.** Report storage private; access via short-lived authorized links **[C]**.
- **NFR-S8.** Webhook/billing callbacks verified by signature **[A]**.

### 9.2 Performance [A]
Numeric targets are placeholders to be confirmed (Q-23):
- Dashboard and list views usable within ~2 s on a typical broadband connection for typical data volumes.
- Check results visible in the UI within ~1 minute of being recorded.
- Audit of a plan-capped site completes in a bounded, communicated time; progress updates at least every few seconds of work.
- Monitoring workers scale horizontally; one tenant's load cannot starve others (fair scheduling / per-tenant limits).

### 9.3 Reliability [A]
- Scheduled checks are executed close to their schedule; drift and missed-check conditions are detectable (a missed check is not recorded as "up").
- **Monitoring-the-monitor:** failure of the monitoring system itself must not silently look like all-clear; platform health is observable (ADM-2).
- Worker jobs are idempotent, retried with backoff, and moved to a visible failed state after exhaustion.
- Alert delivery is reliable and deduplicated (NOTIF-4, NOTIF-7).
- Availability target for UptimePilot itself: Q-23.
- Backups and restore procedure for Firestore and report storage: to be defined.

### 9.4 Accessibility [A]
- Target WCAG 2.1 AA for the web app (Q-24 to confirm).
- Full keyboard operability, visible focus, sufficient color contrast, status never conveyed by color alone (important for up/down indicators), semantic labels for screen readers, accessible charts (text alternative or data table), accessible forms and error messages.
- PDF reports should be reasonably accessible (text-based, structured), effort level per Q-24.
- Responsive layout from small laptop to desktop; behavior on small screens: Q-25 (native mobile apps are deferred, section 11).

### 9.5 Privacy [A]
- Collect only data needed to deliver the service: account email, workspace data, monitored URLs and results, audit data.
- Crawled page content is retained only as needed for evidence; retention periods: Q-8.
- Data deletion on user/workspace request, including from storage and backups within a defined window.
- Third-party processors (email, AI, payments, hosting) are disclosed in a privacy policy; a legal review of the applicable regimes (e.g. India's DPDP Act, GDPR for EU users) is required before launch: Q-26.
- No customer data used to train third-party AI models without explicit permission: Q-13.
- Privacy policy, terms of service and acceptable-use policy (prohibiting monitoring/crawling of third parties without authorization) are required launch artifacts.

### 9.6 Maintainability [A]
- Business logic (rule engine, scoring, quota checks, uptime math, SSRF guard) is separate from UI and transport and unit-testable (`CLAUDE.md`).
- SEO rules are data-driven and versioned, each with tests and fixtures.
- Plan definitions are configuration (BILL-4).
- Architecture decisions recorded as ADRs in `docs/ADR/`.
- Structured logging and metrics for workers; no sensitive data in logs.
- Documentation kept current in `docs/STATUS.md` and `docs/NEXT_SESSION.md`.

---

## 10. Error, empty, loading and recovery states [A]

Every screen must define all four. General principles: say what happened in plain language, say what the user can do next, never leak internals or secrets, and never show a blank screen.

| Area | Empty | Loading | Error | Recovery |
|---|---|---|---|---|
| Dashboard | "No monitors or audits yet" with Add monitor / Run audit actions | Skeletons for status cards | Partial failure shows what loaded plus a retry for the failed part | Retry; stale data labeled with its age |
| Monitor list | Empty state with explanation and Add monitor | Skeleton rows | Load error with retry | Retry without losing filters |
| Monitor creation | – | Button busy state; "running first check…" | Field-level validation; SSRF rejection ("this address can't be monitored"); quota reached → upgrade prompt; server error → keep form values | Fix and resubmit; no duplicate monitor on double-submit |
| Monitor detail | "No checks yet" for brand-new monitor | Chart skeleton | Failed to load history → retry; monitor currently down shows last error category | Resume paused monitor; re-run check now |
| Incidents | "No incidents in this period" (positive framing) | Skeleton | Load error with retry | Adjust filter/date range |
| SSL | "Certificate checks start after the first successful HTTPS check"; non-HTTPS monitor → "Not applicable" | Spinner | Cert fetch failed shown as a state, not a silent blank | Re-check now |
| Audit start | – | "Starting…" | URL rejected; quota/cap message; robots disallow; unreachable | Edit URL, retry |
| Audit in progress | – | Queued → crawling (n/cap) → analyzing, with cancel | Worker failure → "audit failed, no charge against quota" (Q-6) with retry | Retry; resume vs. restart: Q-20 |
| Audit results | "No issues found" shown with the pages-crawled scope, plus the score disclaimer | Skeleton | Result load error with retry | Re-run audit |
| Findings | "No findings match your filters" with clear-filters action | Skeleton | Load error | Clear filters |
| AI recommendation | "Get explanation" action | "Generating…" | "Explanation unavailable right now" while evidence remains visible | Retry later; cached result reused |
| Reports | "No reports generated yet" | "Preparing report…" with status | Generation failed with retry; expired link → regenerate/new link | Retry; re-request download |
| Notifications settings | Default recipient shown | Spinner | Save failure keeps form values | Resend test email |
| Billing | Free plan shown with upgrade options | Spinner | Provider error: plan unchanged, clear message | Retry; contact support (channel: Q-27) |
| Team | Only owner listed with invite action | Spinner | Invitation expired/invalid → resend | Resend invite |
| Platform admin | "No matching workspaces" | Spinner | Access denied page for non-admins (no information leak) | – |
| Global | 404 page; 403 shows generic message; session expiry → re-authenticate and return to the page; offline/connection lost banner with automatic retry | | Unexpected error shows a generic message with a reference ID | Reload |

Additional behavior:
- **Idempotency:** repeated clicks/refreshes must not create duplicate monitors, audits or reports.
- **Degraded dependencies:** AI, email, billing or storage being down degrades only the dependent feature (AI-4, NOTIF-7).
- **Data gaps:** periods with no check data are displayed as unknown, never as up.

---

## 11. Explicitly deferred features [C]

Not part of the MVP and not to be designed into it beyond leaving room in the data model where cheap:

- **Keyword rank tracking**
- **Backlink intelligence**
- **Competitor analysis**
- **Full JavaScript-rendered crawling** (the MVP crawls server-delivered HTML only; see SEO-4)
- **Native mobile applications**

**[A]** Other ideas from the brief's silence that are also *not* committed: SMS/Slack/webhook notifications, public status pages, white-label reports, scheduled recurring audits, an API for customers, multi-region probing, SSO/SAML. These appear as questions where relevant and are only features if the owner adds them.

---

## 12. Open product decisions

Resolve these before the affected work starts. Suggested defaults are marked; none is binding.

| ID | Question | Why it matters | Suggested default |
|---|---|---|---|
| Q-1 | Which sign-in methods (email/password, Google, others)? | Auth setup, verification flow | Email/password + Google |
| Q-2 | Can a user belong to multiple workspaces? | Data model, agency use | Yes (model for it from day one) |
| Q-3 | Exact permission matrix, notably member delete rights and who can start audits | Authorization rules, tests | Section 4.1 |
| Q-4 | Monitor types beyond HTTP(S) (ping, TCP, DNS, keyword)? | Scope, SSRF surface | HTTP(S) only for MVP; add keyword match later |
| Q-5 | Allowed check intervals per plan, consecutive-failure threshold, recovery rule, SSL alert thresholds | Cost, alert quality | e.g. 5 min min interval (free: slower), 3 failures, first success recovers, 30/14/7/1 days |
| Q-6 | Which quotas besides monitors and pages/audit (seats, audit count/month, history retention, reports, AI usage, recipients)? Do paused monitors and failed audits count? | Plan design, enforcement | Define seats, audits/month, AI allowance; paused count; failed audits don't |
| Q-7 | History retention per plan | Storage cost, plan value | e.g. 30 d / 90 d / 1 yr by tier |
| Q-8 | Data-retention and deletion policy (checks, audits, reports, deleted workspaces, backups) | Privacy, cost | Written policy before launch |
| Q-9 | White-label / client-facing reports for agencies? | Agency value proposition | Post-MVP |
| Q-10 | Is audit-to-audit comparison in the MVP? | Scope | Basic score delta in MVP, issue-level diff later |
| Q-11 | Must users prove domain ownership before audits/monitoring? | Abuse and legal risk | Lightweight verification, at least for audits beyond the first-page level |
| Q-12 | Precise sub-score formulas, severity weights, page-level versus site-level rule handling, and how empty categories are scored | Scoring is unimplementable without it | Specify in `docs/SEO_RULES.md` with worked examples |
| Q-13 | AI provider, model, data-handling and training terms, cost ceiling | Privacy, cost, AI-5 | Decide with legal/privacy input |
| Q-14 | Which report types are in the MVP (SEO audit only, or also uptime/SSL/incident)? | Scope | SEO audit PDF + CSV first |
| Q-15 | Any notification channels beyond email? | Scope | No for MVP |
| Q-16 | Email delivery provider and sender-domain setup (SPF/DKIM/DMARC) | Deliverability | Choose with deployment decision (ADR) |
| Q-17 | Are the provisional prices/quotas coherent and viable (cost per check/audit/AI call vs. price)? | Business model | Run a unit-economics and pricing study |
| Q-18 | Behavior on downgrade when over quota | Billing correctness | Block downgrade until compliant, or pause excess monitors with notice |
| Q-19 | Payment provider, taxes (GST), invoicing, trials, refunds, annual plans, failed-payment/dunning | Billing | Decide after Q-17; none assumed |
| Q-20 | Cancelled/failed audits: keep partial results? count against quota? resumable? | UX, cost | Keep partial, don't count failures |
| Q-21 | Can users ignore/acknowledge findings? | Score semantics | Post-MVP; if added, ignored findings must be visibly excluded, not silently rescored |
| Q-22 | Final MVP boundary (which modules and depth ship first) | Planning | Propose phased plan after this review |
| Q-23 | Numeric performance and availability targets | Capacity, SLOs | Set measurable targets before build |
| Q-24 | Accessibility conformance target and PDF accessibility effort | Compliance | WCAG 2.1 AA |
| Q-25 | Supported browsers and minimum screen size; mobile web behavior | Flutter Web responsive scope | Modern evergreen browsers; responsive down to tablet at least |
| Q-26 | Legal/compliance needs (DPDP Act, GDPR, terms, DPA, acceptable use) | Launch readiness | Legal review before public launch |
| Q-27 | Customer support channel and platform-admin staffing | Operations | Email support at launch |
| Q-28 | Backend framework (Dart Frog vs Shelf) and deployment platform | Architecture | Record as ADRs; not decided here |

---

## Appendix A: Assumption register

Assumptions that, if wrong, change the design the most:

1. Audits are initiated manually; scheduled recurring audits are not in scope.
2. One account owner per workspace; ownership transfer exists.
3. Probes originate from a single region.
4. Reports cover SEO audits first.
5. Paid billing is possible for Indian customers in INR; other currencies are not assumed.
6. Provisional prices and quotas are configuration, not code.

## Appendix B: Document maintenance

- Keep this file the single source of product intent. Link to it rather than copying it.
- Create companion specs as needed (not yet written): `docs/SEO_RULES.md` (rule catalogue and scoring formulas), `docs/DATA_MODEL.md`, `docs/SECURITY.md` (SSRF and tenant isolation design).
- When a question in section 12 is resolved, update the relevant requirement here and record the decision (ADR for architectural ones).
