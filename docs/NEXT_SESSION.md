# Next Session Handoff

Last updated: 2026-10-21. Rewrite this file at the end of every session. Keep it short; detail lives in [`STATUS.md`](STATUS.md), [`IMPLEMENTATION_PLAN.md`](IMPLEMENTATION_PLAN.md) and the specs.

## 1. Current milestone

**Phase 0, repository setup and decisions** (plan: `IMPLEMENTATION_PLAN.md` §4, Phase 0). **Implementation has not started.** There is no application code, no tests and no tooling config. Do not describe the project as built or verified.

## 2. Last completed task and evidence

- **P0-06 (part): `docs/STATUS.md` created and committed**, `3e13b0b` (local, **not pushed**; `develop` is 1 commit ahead of `origin/develop`).
- Before it: six planning docs committed (`61ac6a3`, `3e8c5de`, `203ae69`, `daf8dc5`, `696b7c6`, `29f3ab4`), pushed through `29f3ab4`.
- Evidence: `git log` and `git ls-files` (verified this session). No tests exist or were run.

## 3. Current unfinished task

**P0-06 (remainder): `docs/NEXT_SESSION.md`**, this file. It is written but **uncommitted** until the owner approves a commit. Once committed, P0-06 is complete.

## 4. Files to inspect first (in order)

1. `git status -sb` and `git log --oneline -5` (confirm the state described here is still true).
2. `CLAUDE.md` (rules; note its "Commands" section is still empty).
3. `docs/STATUS.md` (current truth).
4. `docs/IMPLEMENTATION_PLAN.md` §1.2–1.3 (evidence rule) and Phase 0 (tasks P0-07 onward).
5. `docs/ARCHITECTURE.md` §17 (decisions D-1…D-15, spikes S-1…S-6) only if working on ADRs.

## 5. Proposed scope for the next session (confirm with the owner at the start)

**One task: P0-07, make `docs/ADR/` tracked and add an ADR template.**
- Add `docs/ADR/README.md` (what ADRs are, status values, naming `NNNN-title.md`, how to supersede) and `docs/ADR/0000-template.md`.
- Do **not** write real decisions yet (Shelf vs Dart Frog, platform, etc.); those need spikes and owner input.
- Also fix the stale line in `docs/STATUS.md` §9 ("Uncommitted: docs/STATUS.md") and the stale P0-05 note in the plan, and refresh this file and STATUS at session end.

## 6. Acceptance criteria

- `git ls-files docs/ADR` lists the new files (the directory is tracked).
- Template contains: title, status, date, context, decision, alternatives considered, consequences, links to spec sections.
- STATUS.md git section and plan P0-05 reflect reality; STATUS "last updated" and this file updated.
- Nothing outside `docs/` changed. No code or config added.

## 7. Verification to run

No tests, analyzer or build exist yet; do **not** claim any passed. Run and report:
- `git status -sb`, `git ls-files docs/ADR`, `git diff --stat` before committing.
- Search for stale references: `grep -rn "docs/ADR" docs CLAUDE.md` to confirm links resolve to real files.
- Report plainly that no automated tests exist.

## 8. Blockers and decisions pending (owner input needed; not for Claude to decide)

- **Unpushed commit** `3e13b0b`: ask before pushing.
- **Phase 1–3 blockers:** sign-in methods; one vs many workspaces per user; permission matrix (member delete rights); monitor types for MVP; check intervals, failure threshold, recovery rule; MVP boundary; Flutter state management and routing.
- **Platform choices:** backend framework (Shelf recommended, unratified), runtime/region/IaC, CI provider, email provider; cloud projects are **not created** and need owner approval.
- **Unrun spikes:** S-1 (Dart access to Firebase) and S-2 (IP-pinned TLS in `dart:io`) gate the backend and `SafeFetcher`.
- Doc inconsistencies to reconcile later: `API_CONTRACTS.md` §15.1 (items D-1…D-9).

## 9. Out of scope (do not change)

- No application code, scaffolding, dependencies, `pubspec.yaml`, Firebase/Docker/CI config.
- No cloud resources, no Firestore rules or indexes deployed.
- Do not edit the substance of `PRODUCT_SPEC.md`, `ARCHITECTURE.md`, `DATABASE_SCHEMA.md` or `API_CONTRACTS.md` to "resolve" open questions without owner approval.
- Do not rewrite git history or touch the initial commit `2d38465` or the `master` branch.
- Do not mark any plan task done without repository evidence.

## 10. First recommended action

Run `git status -sb` and `git log --oneline -5`, read `STATUS.md`, then **confirm with the owner** whether to (a) push `3e13b0b`, (b) proceed with P0-07 as scoped above, or (c) start the decision pass (P0-10) first. Do not begin scaffolding or coding.
