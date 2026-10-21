# UptimePilot Project Status

| | |
|---|---|
| Last updated | 2026-10-21 (session: status baseline) |
| Updated by | Inspection of the repository at commit `29f3ab4` |
| Rule | Update this file after every meaningful session. Keep it short; link to specs instead of repeating them |

**Status vocabulary:** **Not Started** · **In Progress** · **Implemented** (code exists, not yet proven) · **Verified** (code exists *and* tests/checks were actually run and passed, with evidence) · **Blocked** (cannot proceed; reason stated). A specification or a past conversation never counts as Implemented or Verified.

## 1. Current stage

**Pre-implementation. No application code exists.** The repository contains planning documentation only: product spec, architecture, data model, API contracts and an implementation plan. The project is at the start of Phase 0 of [`IMPLEMENTATION_PLAN.md`](IMPLEMENTATION_PLAN.md) (repository setup, decisions, spikes).

## 2. Modules

Evidence: `git ls-files` lists only `CLAUDE.md`, `README.md` and six files under `docs/`. A search for `*.dart`, `pubspec.*`, `firebase.json`, `firestore.*`, `Dockerfile*`, `*.tf`, `*.yml`/`*.yaml` and `.github` found none.

| Module | Status | Evidence / notes |
|---|---|---|
| Repository scaffold (Flutter app, server, shared packages) | Not Started | No `pubspec.yaml` or source tree |
| CI / tooling (format, analyze, tests, secret scan) | Not Started | No workflow files, no `analysis_options.yaml` |
| Environments (dev/test/staging/prod), config | Not Started | No config or IaC |
| Authentication and workspaces | Not Started | |
| Monitor management | Not Started | |
| Uptime worker, history, incidents | Not Started | |
| SSL monitoring and expiry alerts | Not Started | |
| SEO crawler | Not Started | |
| SEO rules, findings, scoring | Not Started | `docs/SEO_RULES.md` does not exist |
| AI recommendations | Not Started | Optional |
| Reports (PDF/CSV) | Not Started | |
| Email notifications | Not Started | |
| Plans, quotas, billing | Not Started | |
| Platform administration | Not Started | |
| `SafeFetcher` / SSRF protection | Not Started | Highest-risk component; spike S-2 not run |
| Firestore rules, indexes, TTL policies | Not Started | No rules/index files; nothing deployed |

## 3. Documentation (written; no owner sign-off recorded)

| Document | Commit | Notes |
|---|---|---|
| `CLAUDE.md` | `61ac6a3` | Project rules. Its "Commands" section is still empty |
| `docs/PRODUCT_SPEC.md` | `3e8c5de` | Proposed requirements; 28 open questions |
| `docs/ARCHITECTURE.md` | `203ae69` | Proposal; 6 spikes (S-1…S-6) unrun |
| `docs/DATABASE_SCHEMA.md` | `daf8dc5` | Proposal; 22 validation items (V-1…V-22) |
| `docs/IMPLEMENTATION_PLAN.md` | `696b7c6` | Phased roadmap |
| `docs/API_CONTRACTS.md` | `29f3ab4` | Proposed contracts; nothing confirmed by tests |
| `docs/STATUS.md` | this file | Created in this session |

Missing documents: `docs/NEXT_SESSION.md` (required by `CLAUDE.md`), `docs/SEO_RULES.md`, ADRs (`docs/ADR/` exists locally but is **empty and untracked** by git), `README.md` (one line).

## 4. Code implemented

None. No lines of application code have been written.

## 5. Tests and checks run

**None.** No tests exist and no formatter, analyzer or test command has been run against project code, because there is no code. Nothing in this project is Verified.

## 6. Known bugs and unresolved issues

No code bugs (no code). Documentation and repository issues:

1. **Docs disagree in places** (listed in `API_CONTRACTS.md` §15.1): incident acknowledgement needs new schema fields; a `monitorTests` collection is needed; the schema's `expectedStatus` (≤ 20 ints) can't hold the default 200–399, so ranges are proposed; plan task P1-04 says `POST /me` while the contract says `PUT /v1/me`.
2. `IMPLEMENTATION_PLAN.md` still marks P0-05 (the plan itself) as "drafted, not committed"; it is committed (`696b7c6`).
3. `docs/ADR/` is not tracked by git (empty directory).
4. The initial commit `2d38465` is authored as `karthikeyan c`; later commits as `kumaranindian`. The initial commit is already on the remote, so it was left unchanged.
5. A stale local tag `backup/pre-identity-rewrite` points at pre-rewrite history and can be deleted.
6. No `.gitignore` exists.
7. **28 product open questions** (spec §12), **28 architecture questions**, **22 schema validation items**, **15 API questions** remain undecided. Those that block early phases are listed under the next task.

## 7. Missing configuration and external services

Nothing is configured. No Firebase/Google Cloud project, Firestore database, Storage bucket, queue/scheduler, secret store, domain/DNS, container registry or CI is present in the repository, and none should be assumed to exist.

| Needed | State |
|---|---|
| Firebase project(s) and Auth sign-in methods | Not created; sign-in methods undecided |
| Runtime platform, region, IaC tool, CI provider | Undecided |
| Email provider and sender-domain authentication (SPF/DKIM/DMARC) | Undecided |
| Payment provider | Undecided (optional for pilot per the plan) |
| AI provider | Undecided (optional) |
| Backend framework (Shelf vs Dart Frog) | Undecided; ADR needed |

**Local tooling found (not verified to work beyond version output):** Dart SDK 3.8.1, Flutter (binary present, version not queried), Firebase CLI 14.2.1, Docker 28.0.4, Node 20.19.1. `gcloud` is **not installed**.

## 8. Security concerns

Nothing is implemented, so nothing is exposed. Risks recorded for when building starts:

- **SSRF protection is unproven.** Whether `dart:io` can connect to a pre-vetted IP while verifying TLS against the original hostname (spike S-2) is unknown; the fallback is an egress proxy.
- **Dart access to Firebase** (token verification, Firestore, Storage) is unproven (spike S-1).
- **No secret protection yet:** no `.gitignore`, no secret scanning, no CI. Add before any credentials or code arrive.
- **Tenant isolation** rests on backend code and data layout, not Firestore rules; it is untested because it is unwritten.
- Pricing, quotas and score weights are unvalidated hypotheses and must stay configuration, not hard-coded.
- Legal/privacy artifacts (policies, DPDP/GDPR review) do not exist.

## 9. Git state (at time of writing)

- Branch: `develop`; remote `origin` = `https://github.com/kumaranindian/eazy-monitor-360.git`.
- Before this file was created: clean and in sync with `origin/develop` at `29f3ab4`.
- **Uncommitted:** `docs/STATUS.md` (new, untracked).
- Local identity: `kumaranindian` / GitHub no-reply address. Default/main branch per repository: `master`.

## 10. Next recommended task

**P0-06 (remainder): create `docs/NEXT_SESSION.md`**, then commit it with this file. `CLAUDE.md` requires both, and the plan treats P0-06 as one task (STATUS.md is now done).

After that, in order (each its own session): P0-07 track `docs/ADR/` and add an ADR template → P0-10 decision pass on the Phase 1–3 blockers (needs owner input: sign-in methods, one vs many workspaces per user, permission matrix, monitor types, check intervals/failure threshold, MVP boundary, state management/routing) → P0-08/P0-09 spikes S-1 and S-2 → P0-11/P0-12 ADRs for backend framework and platform → scaffold.
