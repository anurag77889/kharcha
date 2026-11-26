# ADR-0004: Celery for background work; study django.tasks

- **Status:** Accepted
- **Date:** 2026-11-26

## Context

Kharcha needs to run four kinds of work in the background:

- notifications
- receipt validation
- exports
- payouts to an unreliable provider

It also needs scheduled jobs.

Django 6.0 added `django.tasks`, an API for defining and enqueuing tasks. However, it ships only development and test backends, and **no production worker** ([6.0 release notes](https://docs.djangoproject.com/en/6.0/releases/6.0/)). Third-party backends exist, such as `django-tasks-celery`.

## Decision

Use **Celery 5.x with a Redis broker**, plus Celery beat for scheduled jobs. Run three queues: `default`, `files` and `payouts`.

Every task follows these rules:

- It accepts IDs and `org_id`, never model instances.
- It is enqueued through `transaction.on_commit`.
- It is safe to run twice. State guards, unique constraints and provider idempotency keys make this hold.
- Payout tasks set `acks_late=True`.
- `autoretry_for` with backoff covers transient errors only.

In Phase 4, port one task to `django.tasks` using the `ImmediateBackend` in tests, and record the comparison in this ADR.

## Consequences

**Benefits**

- Celery's retry, routing and scheduling are mature.
- Celery is the industry-standard skill.

**Costs and risks**

- Celery adds its own operational surface: the broker's visibility timeout, prefetch behaviour and worker memory.
- Beat must run as a single replica. Because scheduled jobs are idempotent, an accidental second beat process does no harm.

## Revisit when

Revisit this decision when `django.tasks` gains a mature production backend and worker. At that point, using the framework API could reduce our coupling to Celery.
