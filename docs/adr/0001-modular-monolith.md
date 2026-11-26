# ADR-0001: Modular monolith on WSGI

- **Status:** Accepted
- **Date:** 2026-11-26

## Context

Kharcha is built by one developer. It covers a single domain whose workflows are tightly coupled: approving a report touches budgets, the ledger, the audit log and notifications, and all of those writes must commit together.

## Decision

Kharcha is one Django project made up of domain apps (`organizations`, `expenses`, `approvals`, `budgets`, …).

It ships as a single image that runs in three modes:

- `web`: Gunicorn WSGI sync workers
- `worker`: Celery
- `beat`: Celery beat

Apps talk to each other only through each other's `services` and `selectors` functions. No app reaches into another app's models from outside those modules.

## Consequences

**Benefits**

- Cross-domain transactions stay atomic, with no distributed-transaction or saga machinery.
- There is one deployment and one place to debug.

**Costs and risks**

- App boundaries are kept by discipline rather than enforced. If that becomes a problem, `import-linter` contracts can enforce them in CI.

**Why not ASGI**

- No feature needs WebSockets or long-lived connections.
- The ORM runs synchronously anyway.
- Revisit this if a live inbox or server-sent events become a requirement.

## Alternatives rejected

**Microservices**

- Every approval would become a distributed transaction.
- Operational overhead would grow with no benefit at this scale.
- There is no team-scaling pressure that would justify them.
