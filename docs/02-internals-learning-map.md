# Part 3: Django Internals Learning Map

Each row links one Django mechanism to three things:
- the **business situation in Kharcha** that forces you to use it,
- the **skill** it builds,
- the **roadmap phase** where it shows up.

Source paths point at the `stable/6.1.x` branch and were checked to exist on 2026-11-26. The [source guide](06-django-source-guide.md) walks through each one.

## A. Request–response lifecycle

| Mechanism | Kharcha situation that forces it | Skill built | Phase |
|---|---|---|---|
| Startup: `django.setup()`, the `apps.populate()` app registry, `AppConfig.ready()` | Registering signal receivers and checks once at startup, not on every import | Knowing why "Apps aren't loaded yet" happens and what import side effects cost | 1 |
| `manage.py`, `ManagementUtility`, `BaseCommand` | A custom `seed_demo_tenant` command and a `backfill_budget_ledger` command | Running code outside the request cycle with full settings and ORM access | 1, 2 |
| Settings loading (`LazySettings`) | Splitting settings into `base`, `dev`, `test` and `prod` read from the environment | Why settings are lazy, and why importing settings at module level in apps is risky | 1 |
| `wsgi.py` / `WSGIHandler`, `asgi.py` | Choosing Gunicorn sync workers (WSGI) and documenting why ASGI isn't needed yet | The concurrency model per process, and what blocks what | 1, 6 |
| URL resolution (`URLResolver`, `URLPattern`, converters) | Nested tenant routes such as `/api/v1/orgs/<slug>/reports/<uuid:id>/` | How `resolve()` walks patterns, and how converters type-cast and reject input | 1 |
| Middleware chain (`BaseHandler.load_middleware`) | Middleware for correlation IDs, request timing, and tenant resolution | The onion model: why order matters, and how short-circuiting works | 1, 3 |
| `HttpRequest` / `HttpResponse`, `StreamingHttpResponse` | A CSV export of 100k rows that must not build the whole file in memory | Response types and memory behaviour | 4 |
| Exception handling (`convert_exception_to_response`, the DRF `exception_handler`) | A consistent error envelope that carries the correlation ID; no stack traces leaking | Where exceptions become responses, at both the Django layer and the DRF layer | 1, 3 |
| Templates and context processors | Email templates for approval notifications, and admin templates | How templates are compiled into nodes and how context is layered | 4 |

## B. ORM internals

| Mechanism | Kharcha situation | Skill | Phase |
|---|---|---|---|
| Model metaclass (`ModelBase`) and `_meta` (`Options`) | A generic audit helper that iterates over `_meta.get_fields()` to compute diffs | How the class body becomes fields, managers and relations | 2 |
| Lazy QuerySets and the result cache | An approvals inbox that is filtered, then paginated, then serialized | Exactly when SQL runs, and why `if qs:` followed by `for x in qs` costs one query, not two | 2 |
| SQL compilation (`Query`, `SQLCompiler`) | Reading `str(qs.query)` and running `EXPLAIN` on the dashboard query | Translating ORM calls into SQL in your head | 2, 5 |
| Custom managers and QuerySets | `TenantScopedQuerySet.for_org(org)` and `ExpenseReport.objects.pending_for(approver)` | Encapsulating data-access rules where they cannot be forgotten | 2, 3 |
| FK, M2M, reverse relations, `related_name` | Report → Expenses → Receipts; Approver ↔ Department | Descriptors, and what `report.expenses.all()` really does | 2 |
| `select_related` / `prefetch_related` / `Prefetch()` | An inbox that shows the submitter, the department, and the current approval step | JOIN versus a second query, and when each wins | 2, 5 |
| **6.1 fetch modes** (`FETCH_ONE`, `FETCH_PEERS`, `FETCH_RAISE`) | Making N+1 queries **raise** in tests | Turning a silent performance bug into a loud test failure | 5 |
| Aggregation and annotation, `Subquery`, `OuterRef`, `Coalesce` | A spend dashboard by department, category and month, with budget utilisation | Doing the work in SQL instead of Python loops | 2, 5 |
| `F()` expressions | Incrementing `Budget.consumed` without a lost update | Atomic updates done in the database | 2 |
| `Q` objects | Inbox filters: "assigned to me OR to my delegate AND NOT expired" | Composing predicates dynamically | 2 |
| `atomic()`, savepoints, `on_commit` | Approving a report means changing its state, consuming budget, writing to the ledger and sending a notification, all or nothing | Transaction boundaries, and deferring side effects until commit | 2, 4 |
| `select_for_update()` (`nowait`, `skip_locked`, `of`) | Two finance approvers acting at the same time; a reimbursement batch picking reports | Pessimistic locking, deadlocks, and `skip_locked` work queues | 2, 4 |
| `bulk_create` / `bulk_update` / `.update()` / `iterator()` | Importing 10k corporate-card transactions; nightly batch updates | What bulk operations skip (signals, `save()`), and memory use | 4 |
| Constraints (`UniqueConstraint` with `condition`, `CheckConstraint`) and indexes (partial, composite) | One open approval step per report; `amount > 0`; `consumed <= limit` | Letting the database guarantee what the application cannot | 2 |
| **6.1 DB-level `on_delete`** (`DB_CASCADE`, `DB_SET_NULL`) | Deciding where cascades run for audit and child rows | Application-level versus database-level referential actions | 2 |
| Migrations: autodetector, executor, `RunPython`, `SeparateDatabaseAndState` | Splitting `amount` into `amount` and `currency` while keeping existing rows; adding a NOT NULL column safely | Migrations that keep production data intact and avoid long table locks | 2 |

## C. Authentication and authorization

| Mechanism | Kharcha situation | Skill | Phase |
|---|---|---|---|
| Custom user model (`AbstractBaseUser` / `AbstractUser`, `USERNAME_FIELD`) | Email login with no usernames | Why the user model must be set before the first migration | 1 |
| Authentication backends (`authenticate()` and `get_user()` per backend) | Email + password backend; per-org deactivation | How `authenticate()` iterates over backends, and how the session stores the backend path | 3 |
| Sessions and cookies (`SessionMiddleware`, `SessionBase`, `cycle_key`) | Rotating the session on login; a "log out all devices" feature | Session fixation, and the trade-offs between storage backends | 3 |
| Password hashing (`hashers.py`, `PBKDF2PasswordHasher`, upgrade on login) | Raising the work factor without forcing password resets | How hashes encode algorithm, iterations and salt, and how they are upgraded on login | 3 |
| Permissions and Groups (`ModelBackend.has_perm`) | Internal staff roles in the admin (support versus finance-ops) | Global model permissions, and why they don't model per-tenant roles | 3 |
| Object-level authorization (DRF `has_object_permission`, a policy module) | "Approver X may approve report R only if R is in their department, under their limit, and not their own" | Consistent enforcement in one place, with the same check reused by queries and actions | 3 |
| Token authentication (a custom DRF `BaseAuthentication`) | Personal API tokens for integrations: prefixed, hashed and revocable | Token storage design and constant-time comparison | 3 |
| `AuthenticationMiddleware` and the lazy `request.user` | Understanding why `request.user` costs a query only when it is touched | `SimpleLazyObject` | 3 |

## D. Middleware

| Middleware | Belongs in middleware? | Why |
|---|---|---|
| Correlation ID (read or generate `X-Request-ID`, bind to logs, echo in the response) | **Yes** | It is cross-cutting, applies to every request, and contains no business rules |
| Request timing and structured access logs | **Yes** | The same reasons |
| Tenant resolution (load org from URL or header, set a `contextvar`) | **Yes, partially** | Resolving and attaching the tenant is cross-cutting. *Authorising* membership belongs in the permission layer. |
| "Approver must be in department" | **No** | This is a business rule. Put it in a policy or service, where it can be tested without HTTP. |
| Rate limiting | **Either** | A DRF throttle at the view level is often better because it knows the user and the endpoint |

## E. Signals

| Use in Kharcha | Decision |
|---|---|
| `post_save` on `User`, used to create a profile | **Avoid.** Do it explicitly in the `register_user()` service. |
| Audit log on every model change | **Partially.** Signals catch admin edits, but `.update()` and `bulk_*` skip them. The primary audit lives in services; a signal-based audit of admin edits is kept as a teaching contrast. |
| `request_finished`, used to clear the tenant `contextvar` | **Yes.** This is framework-level plumbing. |
| A custom `report_approved` signal | **Avoid** for critical effects. Prefer explicit service calls plus `on_commit`. |

Lesson to take away: signals run **synchronously, inside the same transaction**, and in an undefined order across receivers. They hide control flow from someone reading the code, and bulk operations skip them.

## F. Forms, serializers and validation

| Layer | Kharcha example |
|---|---|
| Field validation | `amount` must be a positive `Decimal` with 2 decimal places. `currency` must be in the org's allowed set. |
| Cross-field validation (`validate()` / `clean()`) | `expense_date` must be on or before today and within the policy's lookback window |
| Service-level validation (needs DB state) | "Report total exceeds the remaining budget" is a business-rule error, not a serializer error |
| Model `full_clean()` | Admin edits run model validation. Note that `save()` does **not** call `full_clean()`. |
| Database constraints | The final guarantee for `amount > 0`, uniqueness, and `consumed <= limit` |
| Admin `ModelForm`s | Custom admin actions such as "force-reject" and "re-run policy check" |

## G. Background processing

| Need | Mechanism | Phase |
|---|---|---|
| Notify the approver after a report is submitted | Celery task dispatched through `transaction.on_commit` | 4 |
| Process a receipt: sniff MIME type, build a thumbnail, stub a virus scan | Celery task with retries; status column on `Receipt` | 4 |
| Monthly budget rollover | Celery beat, plus an idempotency guard (unique `(budget, period)`) | 4 |
| Payout to a reimbursement provider (mock HTTP service) | Retries with exponential backoff, an idempotency key sent to the provider, and a reconciliation job | 4, 7 |
| Comparison with `django.tasks` (Django 6.0+) | Write an ADR and an experiment using the `ImmediateBackend` in tests | 4 |

## H. Caching and performance

| Need | Mechanism |
|---|---|
| A dashboard aggregate that is expensive and read often | Low-level `cache.get_or_set` with a **per-tenant versioned key**, bumped on write |
| Policy and category lookups on every expense submission | A per-request memo plus a short-TTL cache |
| An inbox list that grows with tenant size | Keyset (cursor) pagination, a composite index on `(org_id, state, created_at)`, and `select_related` |
| Connection usage | `CONN_MAX_AGE`, `CONN_HEALTH_CHECKS`, and a sizing formula for Gunicorn workers + Celery concurrency against Postgres `max_connections` |

## I. Files

Receipts are uploaded straight from the client to S3 using a presigned POST. The upload has size and content-type conditions attached. After upload, the server validates the file by magic bytes (not by extension) and by size. Downloads go through an authorization check first, then the user gets a 60-second presigned GET URL. Thumbnails are generated by a background task.

## J. Security

These map to doc 07: CSRF on session-authenticated API calls, CSP, an object-level authorization test matrix, throttling, upload validation, hashed tokens, secrets kept out of the repository, and `check --deploy`.

## K. Testing

Testing is designed in from Phase 1. Doc 07 covers:

- which tests use `TestCase` and which use `TransactionTestCase`,
- `captureOnCommitCallbacks`,
- thread-based race tests,
- `django_assert_max_num_queries`,
- `FETCH_RAISE`.
