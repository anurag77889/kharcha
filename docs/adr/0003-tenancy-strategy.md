# ADR-0003: Shared-schema multi-tenancy with explicit scoping

- **Status:** Accepted
- **Date:** 2026-11-26

## Context

Every customer's data must be isolated from every other customer's. The standard options are:

1. A separate database per tenant.
2. A separate schema per tenant (for example django-tenants, which sets the Postgres `search_path`).
3. One shared schema, with an `org_id` on every tenant-owned row.

## Decision

Use option 3, the **shared schema**:

- `org_id` is denormalised onto every tenant-owned table, including child tables.
- Middleware resolves the org from the URL (`/orgs/{slug}/`) and stores it in a `contextvar`.
- QuerySets are tenant-scoped (`.for_org()`). Request-path code that queries without an org context raises an error.
- Policy functions check membership.
- Requests for another tenant's resources return 404.
- Postgres row-level security (RLS) is deferred to Phase 7, as defence in depth.

## Consequences

**Benefits**

- There is one set of migrations.
- Cross-tenant queries for the internal admin stay simple.
- It is cheap at any tenant count.
- It teaches managers, middleware and authorization directly.

**Costs and risks**

- Isolation depends on application discipline. Scoped managers, a cross-tenant test matrix for every endpoint, and RLS later all reduce that risk.
- Noisy-neighbour queries need composite indexes that lead with `org_id`.
- Scoping does **not** protect these paths, so each needs explicit care:
  - raw SQL
  - the admin
  - management commands
  - Celery tasks, which must receive `org_id` and re-scope

## Alternatives rejected

- **Schema per tenant**: migrations multiply per tenant, connection and search-path handling is subtle, and cross-tenant reporting is hard.
- **Database per tenant**: too expensive at this stage.
