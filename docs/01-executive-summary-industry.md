# Parts 1–2: Executive Summary and Django in the Industry

Every claim below carries one of three labels:

- **[Verified]** comes from a primary source, which is linked.
- **[Inferred]** is a reasonable reading of public technical material.
- **[Recommendation]** is my own engineering judgement.

---

## Part 1: Executive Summary

### What makes Django valuable in production?

1. **One coherent stack for relational, workflow-heavy software.** Django ships these pieces designed to work together:
   - the ORM and migrations
   - authentication and sessions
   - the admin
   - forms and validation
   - the cache framework
   - security middleware

   In a microframework you assemble these yourself, and that is where integration bugs live. **[Recommendation]**

2. **A conservative, predictable release process.** **[Verified]**
   - Django 6.1 (released 2026-08-05) supports Python 3.12–3.14.
   - 5.2 is the current long-term-support (LTS) release, and 6.2 LTS is due in April 2027.
   - In August 2026 the project announced a move to an annual release cycle.
   - Deprecated features are removed on a published schedule.
   - Sources: [release process](https://docs.djangoproject.com/en/6.1/internals/release-process/), [6.1 announcement](https://djangoproject.com/weblog/2026/aug/05/django-61-released), [annual cycle](https://djangoproject.com/weblog/2026/aug/10/annual-release-cycle).

3. **Secure defaults.** **[Verified]**
   - CSRF protection, auto-escaping templates, parameterised SQL, clickjacking protection, and PBKDF2 password hashing are all on by default.
   - Django 6.0 added built-in Content Security Policy support (`SECURE_CSP`, `django.middleware.csp`).
   - Source: [Django 6.0 release notes](https://docs.djangoproject.com/en/6.0/releases/6.0/).

4. **The admin as an internal operations tool.** Real companies run support and operations teams on a customised Django admin before they build bespoke back-office UIs. **[Inferred]**

5. **It scales further than its reputation suggests.** **[Verified]**
   - Instagram describes its server as a "several-million-line Python monolith" running Django, deployed many times a day ([Instagram Engineering](https://instagram-engineering.com)).
   - At that scale, the hard problems are databases, caches and process models. The framework is rarely the bottleneck. **[Inferred]**

### What should you understand before building something serious?

Learn these mechanisms, in roughly this order. Each one is a place where "it works" and "it's correct" diverge.

1. **The request lifecycle.** WSGI handler → middleware chain (built as nested callables at startup) → URL resolver → view → response → middleware unwinding.
2. **QuerySet laziness.** A QuerySet is a *description* of SQL. SQL runs only when you iterate, slice with a step, call `len()`/`list()`/`bool()`, and so on. Results are cached on the QuerySet instance. Most N+1 bugs follow from this.
3. **Transactions and connections.**
   - By default each query runs in autocommit mode.
   - `atomic()` opens a transaction or a savepoint.
   - `select_for_update()` takes row locks, which only work inside a transaction.
   - `on_commit()` defers side effects until the transaction commits.
4. **Authentication ≠ authorisation.**
   - Authentication decides *who you are*: backends, sessions, tokens.
   - Authorisation decides *what you may do*: permissions, object-level policy, query scoping.
5. **Where business logic lives.** It belongs in models, services, forms or serializers. Signals and middleware are poor places for it, and you need to be able to say why.

### Which projects give the greatest learning value?

Projects where **correctness under concurrency and multi-user authorisation are the product**, not features bolted on afterwards. Good examples:

- money moving between states
- approvals
- budgets
- inventory reservations
- multi-tenant data

These force transactions, locks, constraints, permission design and idempotency. Pure CRUD, content sites and "dashboard over an API" projects don't. **[Recommendation]**

---

## Part 2: Django in the Industry

### 2.1 Industry use cases, and how well Django fits each

| Category | Fit | Why |
|---|---|---|
| SaaS / B2B business apps | **Strong** | Relational data, roles, workflows, admin, forms. This is Django's core use case. |
| E-commerce and marketplace back-offices | **Strong** | Orders, inventory and settlement are transactional and relational. Payment, search and recommendations are often separate services. |
| Financial and transactional internal systems | **Strong** | PostgreSQL plus `atomic()`, `select_for_update()` and constraints cover most needs. Ultra-low-latency trading is a poor fit. |
| CMS and publishing | **Strong** | Wagtail and django CMS are mature. It is a solved problem, so it teaches less. |
| Enterprise and internal tools | **Strong** | The admin plus auth plus the ORM delivers value fastest. |
| Healthcare and admin systems | **Reasonable** | A good technical fit. Compliance work (audit, encryption, consent) dominates. |
| Logistics and workflow | **Reasonable** | Fine for orchestration and state. Real-time tracking fan-out needs extra infrastructure. |
| Multi-tenant platforms | **Strong** | Possible with shared-schema plus scoping, or schema-per-tenant using django-tenants. |
| Data-heavy analytics | **Weak–reasonable** | Use Django for metadata and APIs. Heavy aggregation belongs in a warehouse or OLAP store. |
| Real-time, high fan-out (chat, live games) | **Weak** | Channels works, but each connection holds a process resource. Go, Elixir or Node are more natural. |
| CPU-bound ML inference serving | **Weak** | Use FastAPI or a dedicated model server, with Django as the control plane. You already have FastAPI experience here. |
| Identity platforms | **Reasonable** | Django auth is a good base. Full OIDC provider work is specialised (django-oauth-toolkit). |

### 2.2 Real-world architecture patterns

| Company / codebase | What is publicly known | Label |
|---|---|---|
| Instagram | Very large Django monolith. Investment in Python runtime efficiency. Custom work around the ORM at scale. ([Instagram Engineering](https://instagram-engineering.com), [Django forum discussion](https://forum.djangoproject.com/t/is-django-useful-without-the-orm-instagram-case-study/23834)) | Verified (architecture) / Inferred (ORM details) |
| Sentry | Open-source Django application ([github.com/getsentry/sentry](https://github.com/getsentry/sentry)). Large single-package layout under `src/sentry`. Uses Celery and PostgreSQL. | Verified (source is public) |
| Eventbrite | Long-running Django codebase, part monolith and part services ([InfoWorld interview](https://www.infoworld.com/article/2179076/expert-interview-how-to-scale-django-2.html)) | Verified (interview) |
| DoorDash | 2017 post "Tips for Building High-Quality Django Apps at Scale". Later moved core services to Kotlin microservices. | Verified (post exists) / Inferred (migration reasons) |
| Zapier | Reported to use Django for HTTP and Celery for async. Only secondary sources were found. | **Unverified**, so do not repeat it in interviews |

**Recurring patterns** **[Inferred]**, from Sentry's source, conference talks such as "High Performance Django", and the posts above:

1. **Modular monolith first.** Teams extract services only when team size or scaling needs force it. DoorDash's migration is the counter-example: it happened at very large organisational scale.
2. **Thin views, a service layer, explicit query modules.** Business rules move out of views and serializers into plain Python functions that you can unit-test.
3. **A Celery (or similar) worker fleet** for anything slow or unreliable: email, third-party APIs, exports, scheduled jobs.
4. **PostgreSQL as the system of record**, with database constraints as the last line of defence.
5. **Redis for cache, rate limiting and the broker.** Teams often move to separate instances later, once eviction policies conflict.
6. **Gunicorn (WSGI) behind a load balancer.** Teams adopt ASGI only when they need WebSockets or long-lived I/O.

### 2.3 Strengths and limitations

| Strengths | Limitations |
|---|---|
| Batteries included: auth, admin, ORM, migrations, forms, cache, security | The ORM makes N+1 queries easy to write by accident. Laziness hides query cost. |
| Mature migration system with dependency graphs | Large migrations on big tables need manual zero-downtime techniques |
| Huge ecosystem (DRF, Celery, django-storages, Wagtail) | Async support is partial. Much of the ORM still runs sync under the hood, using `sync_to_async` thread pools. |
| Clear security defaults | Global settings and the app registry make some patterns (multi-tenant config) awkward |
| Strong conventions, so it's easy to onboard engineers | "Fat models" vs "services" is unresolved. Teams must choose a structure deliberately. |
| Long-term support releases | Signals and `save()` overrides can hide behaviour and surprise readers |

### 2.4 Common production problems (all of them appear in this project's roadmap)

1. **N+1 queries** from lazy related access in serializers and templates. Django 6.1's `QuerySet.fetch_mode(FETCH_RAISE)` can now make these fail loudly. ([fetch modes docs](https://docs.djangoproject.com/en/6.1/topics/db/fetch-modes/))
2. **Lost updates and race conditions.** The usual shape is read-modify-write in Python instead of an `F()` expression, a conditional `UPDATE`, or `select_for_update()`.
3. **Side effects before commit.** An email or task is fired, then the transaction rolls back, or the worker reads data that isn't committed yet. The fix is `transaction.on_commit()`.
4. **Non-idempotent background tasks.** Retries cause double emails or double payouts.
5. **Cross-tenant data leaks.** One `.objects.all()` that forgot the tenant filter is enough.
6. **Migrations that lock large tables**, and data migrations that load whole tables into memory.
7. **Connection exhaustion.** Too many Gunicorn workers × `CONN_MAX_AGE` × Celery concurrency exceeds Postgres `max_connections`.
8. **Stale caches** after writes, and cache keys that aren't namespaced per tenant (a security bug as well as a correctness bug).
9. **Misconfigured production settings.** Typical examples are `DEBUG=True`, a missing `ALLOWED_HOSTS` or `CSRF_TRUSTED_ORIGINS`, and wrong proxy headers. `manage.py check --deploy` catches many of these.

## Sources

- [Django download and supported versions](https://www.djangoproject.com/download/)
- [Django 6.1 released](https://djangoproject.com/weblog/2026/aug/05/django-61-released)
- [Django moving to an annual release cycle](https://djangoproject.com/weblog/2026/aug/10/annual-release-cycle)
- [Django 6.0 release notes (CSP, Tasks framework)](https://docs.djangoproject.com/en/6.0/releases/6.0/)
- [Django 6.1 release notes](https://docs.djangoproject.com/en/6.1/releases/6.1/)
- [endoflife.date: Django](https://endoflife.date/django)
- [Instagram Engineering](https://instagram-engineering.com)
- [Django forum: Instagram ORM case study](https://forum.djangoproject.com/t/is-django-useful-without-the-orm-instagram-case-study/23834)
- [Django forum: who uses Django for large projects](https://forum.djangoproject.com/t/who-is-using-django-for-really-large-projects/166)
- [InfoWorld: Eventbrite on scaling Django](https://www.infoworld.com/article/2179076/expert-interview-how-to-scale-django-2.html)
- [Sentry source](https://github.com/getsentry/sentry)
