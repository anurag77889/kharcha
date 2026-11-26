# Part 15: Final Recommendation

## The project to build

**Kharcha** is a multi-tenant Spend & Approval platform. Companies use it to collect expense claims with receipts, enforce spending policy, route claims through multi-level approval, consume departmental budgets atomically, and pay reimbursements in batches. Everything is fully audited.

## Why this project

- **It scored highest of 17 candidates: 8.61 out of 10.** It is also the only candidate with no dimension below 7. See [03](03-project-ideas-scoring.md).
- **Its difficulty is built into the domain.** The following problems are part of the business itself, not features added on top:
  - concurrent approvals
  - shared budgets
  - money states
  - per-tenant, per-object authorization
  - untrusted files
  - retried payouts
- **One developer can finish it.** It has about 15 entities and 35 endpoints. The work is deep rather than wide.
- **It complements what you already know.** It doesn't repeat your FastAPI or monitoring project. It connects to your mainframe background through batch processing, commit boundaries, and audit.

## Django concepts it teaches

**Startup and request handling**
- Startup and the app registry
- The middleware onion and URL resolution
- The DRF dispatch path

**Authentication and authorization**
- The custom user model, authentication backends, sessions, CSRF, and password hashing
- Object-level authorization and tenant-scoped managers

**ORM and database**
- Lazy QuerySets and SQL compilation
- Relations, plus `select_related`, `prefetch_related`, and 6.1 fetch modes
- Aggregation, `F()` expressions, and `Q` objects
- `atomic()` and savepoints, `select_for_update`, and `on_commit`
- Constraints and partial indexes
- Data migrations that preserve existing data

**Other framework features**
- Signals, including when *not* to use them
- The form and serializer validation lifecycle
- The cache framework
- The storage API
- Admin customization

## Technologies

| Area | Choice |
|---|---|
| Language | Python 3.13 |
| Framework | **Django 6.1**. Upgrade to 6.2 LTS around April 2027. |
| API | DRF 3.18, drf-spectacular |
| Database | PostgreSQL 17 |
| Cache and queue | Redis, Celery |
| File storage | S3 via django-storages (MinIO locally) |
| Web server | Gunicorn, WhiteNoise |
| Testing | pytest-django, factory_boy |
| Tooling | uv, ruff |
| Infrastructure and CI | Docker, GitHub Actions |
| Monitoring | Sentry, CloudWatch |
| Hosting | AWS: EC2 + RDS + S3 first, then ECS Fargate + ALB |

The following are **deliberately excluded**: Channels, Kafka, Kubernetes, microservices, JWT, and a frontend SPA.

## Build first (the MVP, Phases 1–4)

1. The skeleton, settings, Compose, the custom user, the CI pipeline, and the correlation-ID middleware
2. Organizations, memberships, and departments in the admin
3. Reports and expenses with constraints, plus the "my reports" list and the approvals inbox, with query budgets
4. Submitting a report with a conditional update and an idempotency key; the approval state machine
5. Budget consumption with row locks and an append-only ledger
6. Authentication (sessions plus hashed API tokens), the policy layer, and tenant isolation, all covered by the permission matrix
7. Celery notifications through `on_commit`, receipts uploaded through presigned URLs, reimbursement payouts with retries, the cached dashboard, and the CSV export

## Postpone (Phase 7, only once you can justify each)

- Postgres row-level security
- Outbound webhooks through an outbox
- ECS Fargate stage 2
- API versioning
- A hash chain for the audit log
- 2FA for the admin
- Multi-currency and FX
- OCR
- SSO
- Channels or a live inbox

## Expected level of understanding after Phase 6

You should be able to do the following **from your own code and measurements**:

- Explain what Django does between the socket and the view.
- Predict how many queries a piece of ORM code will run.
- Choose between optimistic and pessimistic concurrency control and defend the choice.
- Design authorization that holds across both reading and writing data.
- Make background work safe to run more than once.
- Change production data safely through migrations.
- Deploy and operate the app on AWS.

This is the level of a solid mid-level backend engineer on a Django product team, with interview stories that go deeper than "I built a CRUD API".

## Next step

Phase 0 is in [05-roadmap](05-roadmap.md):
1. Write `notes/phase0-domain.md`.
2. Hand-draw the report state machine.
3. Redraw the ER diagram from memory.

Bring all three back for review. Phase 1 starts after that.
