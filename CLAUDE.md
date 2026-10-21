# UptimePilot

SaaS platform for website/API uptime monitoring, SSL certificate health and expiry alerts, technical SEO audits with deterministic scoring, AI-assisted explanations and recommendations, PDF/CSV reports, email notifications and subscription plans.

> The repository directory is `eazy-monitor-360`; the product name is UptimePilot.

## Repository status

Greenfield: at the time of writing there is no application code, `pubspec.yaml`, or specification. Update this section and the sections below as the project takes shape.

## Technology direction

| Area | Choice |
|---|---|
| Frontend | Flutter Web |
| Backend | Dart. **Undecided: Dart Frog or Shelf.** Check the repo; if still unset, record the choice as an ADR in `docs/ADR/` before building. |
| Auth | Firebase Auth |
| Database | Cloud Firestore |
| Background work | Scheduled monitoring and SEO worker jobs |
| Storage | Private Cloud Storage for generated reports |
| Deployment | **Undecided.** Propose options after reviewing requirements and record in an ADR. |

Architecture decisions go in `docs/ADR/`. Specifications live in `docs/`; link to them, don't duplicate them here.

## Commands

None established yet. Once the project is scaffolded, list the exact format, analyze, test and run commands here (for Flutter: `dart format`, `flutter analyze`, `flutter test`; for the backend: its own equivalents).

## Working rules

1. Inspect the existing repository before modifying files or adding dependencies.
2. Never overwrite working code or existing configuration without explaining why.
3. Follow the existing architecture and coding conventions.
4. Implement one approved feature at a time; do not build outside the approved scope.
5. For multi-file changes, explain the plan before implementing.
6. Keep business logic testable and separate from UI code.
7. Add tests for important logic, permissions, error handling and security controls.
8. Run the available formatter, analyzer and relevant tests. Report actual results; never claim a check passed unless it was executed.

## Product and security invariants

- **Deterministic SEO:** findings and scores come from deterministic rules. AI may explain findings but must not invent evidence or alter verified results.
- **Tenant isolation:** enforce it, and authorization, server-side. Never rely on client checks or UI hiding.
- **SSRF protection:** all user-submitted URLs (monitors, SEO audits, anything fetched on a user's behalf) must be guarded against SSRF, including redirects, DNS rebinding, private/loopback/link-local IPs and cloud metadata endpoints. Validate the resolved IP at connect time and re-validate on every redirect hop.
- **Secrets:** never commit credentials, API keys, tokens or production secrets. Use environment/secret manager configuration.
- **Reports:** generated PDF/CSV files live in private storage and are served only through authorized, tenant-scoped access.

## Documentation

- Update `docs/STATUS.md` and `docs/NEXT_SESSION.md` after each completed feature or work session.
- Keep docs concise; link to the detailed spec rather than copying it.

## Finishing a task

Before finishing, summarize: changed files, implementation details, test results, known issues, and the next recommended task.
