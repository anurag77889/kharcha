# Kharcha — Multi-tenant Spend & Approval Platform

> Working name. *Kharcha* (खर्चा) means "expense" in Hindi.

Kharcha is a B2B SaaS backend. Employees of a company submit expense reports with receipts. Each report is checked against the company's policy, approved by one or more people, and charged against a departmental budget. Approved reports are paid back in reimbursement batches.

The project is built with **Django 6.1**, Django REST Framework, PostgreSQL, Redis, Celery and S3. It is a **modular monolith**: one deployable application split into well-separated Django apps. Microservices are deliberately avoided (see [ADR-0001](docs/adr/0001-modular-monolith.md)).

The project has two goals:

1. **Understand Django by building with it.** Every feature is chosen to force a specific Django mechanism into the open: the request lifecycle, middleware, lazy QuerySets, transactions and row locks, authentication backends, object-level permissions, `transaction.on_commit`, cache invalidation, and migrations that keep existing data intact.
2. **Look like a real product.** The project handles tenant isolation, concurrent approvals, overspending, double-submitted requests, retried background jobs, malicious uploads and slow dashboards, and each of these is backed by tests.

## Status

Phase 0 is done: research, product definition and architecture. Phase 1 (foundations) is next. See the [roadmap](docs/05-roadmap.md).

**Working mode.** The project follows a mentor model. Each feature has a learning brief in the roadmap. The developer writes the implementation, then reviews it against the brief's verification steps and reflection questions.

## Documentation map

| Doc | What it covers |
|---|---|
| [01 — Executive summary & Django in industry](docs/01-executive-summary-industry.md) | Why Django works in production, where it fits and where it doesn't, real-world patterns, common production problems |
| [02 — Django internals learning map](docs/02-internals-learning-map.md) | Each Django mechanism, the business situation that requires it, and the skill it builds |
| [03 — Project ideas, scoring & top three](docs/03-project-ideas-scoring.md) | 17 project ideas, a weighted scoring matrix and a ranking |
| [04 — Flagship technical design](docs/04-flagship-design.md) | Requirements, architecture, app structure, schema, ER diagram, API, stack |
| [05 — Development roadmap](docs/05-roadmap.md) | Phases 0–7. Each feature uses an 8-step learning format and includes deliberate engineering challenges |
| [06 — Django source-code guide](docs/06-django-source-guide.md) | A selective tour of the Django 6.1 source, with an experiment for each topic |
| [07 — Testing, security & performance](docs/07-testing-security-performance.md) | Test strategy, concurrency testing, security checklist, performance method |
| [08 — Deployment & operations](docs/08-deployment-operations.md) | Moving from local to production, AWS stage 1 and stage 2, CI/CD, observability |
| [09 — Portfolio & interview preparation](docs/09-portfolio-interviews.md) | Repository practices, documentation set, interview questions and senior-level follow-ups |
| [10 — Final recommendation](docs/10-final-recommendation.md) | A one-page decision summary |
| [ADRs](docs/adr/) | Architecture decision records |

## Planned stack (versions verified 2026-11-26)

- Python 3.13
- Django 6.1.x
- DRF 3.18.x
- drf-spectacular
- PostgreSQL 17
- Redis
- Celery 5.x
- django-storages with S3 (MinIO for local development)
- Gunicorn and WhiteNoise
- pytest-django and factory_boy
- uv, ruff and pre-commit
- GitHub Actions
- Docker

Setup instructions will be added in Phase 1.
