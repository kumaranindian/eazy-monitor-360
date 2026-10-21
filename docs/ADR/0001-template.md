# ADR-NNNN: <Short, specific title of the decision>

<!--
How to use this template
- Copy this file to docs/ADR/NNNN-short-kebab-title.md using the next unused number (0001 is this template).
- Delete these comments and every placeholder in angle brackets before committing.
- One decision per ADR. If you are deciding two things, write two ADRs.
- Write for a reader who has none of the conversation history.
- Link to the specs instead of copying them (docs/PRODUCT_SPEC.md, ARCHITECTURE.md, DATABASE_SCHEMA.md, API_CONTRACTS.md).
- Never put secrets, credentials or real customer data in an ADR.
- Accepted ADRs are not edited to change the decision. Write a new ADR and mark the old one Superseded.
-->

| | |
|---|---|
| Status | <Proposed / Accepted / Superseded by ADR-NNNN / Rejected> |
| Date | <YYYY-MM-DD, when the status last changed> |
| Deciders | <who decided or approved; roles are fine> |
| Related spec items | <e.g. PRODUCT_SPEC Q-4, ARCHITECTURE S-1, DATABASE_SCHEMA V-16> |

## Context and problem

<What situation forces a decision? What is undecided, and why does it matter now? State facts and constraints, and what is already fixed by earlier decisions or the brief. Mark anything unverified as unverified.>

## Decision drivers

<The criteria that matter, in priority order. For example: security, tenant isolation, testability, delivery effort, running cost, operational burden, reversibility, team skill.>

- <driver 1>
- <driver 2>

## Options considered

<Include "do nothing" or "defer" when it is a real option. For each option give a short description and its main pros and cons against the drivers.>

### Option A: <name>
- <what it is>
- Pros: <...>
- Cons: <...>

### Option B: <name>
- <what it is>
- Pros: <...>
- Cons: <...>

## Decision

<The chosen option, stated in one or two clear sentences, in the active voice: "We will ...". Leave this section empty while the status is Proposed if no option is yet recommended.>

## Reasons

<Why this option beats the others against the drivers. Cite evidence: a spike result, a test, a measurement, a document link. Say what is assumed and not yet proven.>

## Trade-offs and consequences

- Gains: <...>
- Costs and limitations: <...>
- Follow-up work this creates: <tasks, with IDs from docs/IMPLEMENTATION_PLAN.md where they exist>
- Open risks: <...>

## Security and operational implications

- Security: <trust boundaries, tenant isolation, SSRF, secrets, authentication or authorization effects. Write "none identified" only after checking.>
- Privacy and data: <personal data stored, retention, third-party processors>
- Operations: <deployment, monitoring, cost, scaling, on-call or support effects>
- Testing: <what tests must exist to keep this decision honest>

## Migration or rollback plan

- Adoption steps: <how the change is introduced, in small reversible steps>
- Rollback: <how to undo it, what data or contracts would be affected, and the point after which rollback is no longer cheap>
- Review trigger: <the condition or date at which this decision should be revisited>

## Links

- Supersedes: <ADR-NNNN or none>
- Superseded by: <ADR-NNNN or none>
- Related decisions: <ADR-NNNN, ...>
- References: <spikes, benchmarks, issues, external documentation (URLs only, no secrets)>
