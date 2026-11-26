# Parts 13–14: Portfolio Strategy and Interview Preparation

## Part 13: GitHub and Portfolio Strategy

### 13.1 Repository

- **Structure.** The repository follows the layout in [04 §C](04-flagship-design.md#c-django-project-structure). The root contains:
  - `README.md`, `docs/`, `docs/adr/`, `docs/perf/`, and `notes/`. `notes/` holds your learning journal. It is optional to keep public, but it is impressive when it is.
  - `compose.yaml`, `Dockerfile`, `.env.example`, `.github/`, `pyproject.toml`, and `uv.lock`.
- **Commits.** Use Conventional Commits (`feat(approvals): conditional update for step decisions`). Each commit should be a small, coherent change. The commit body says *why*. Never squash away the story of a bug fix. "test: reproduce lost update on budget" followed by "fix: lock budget row" is a great pair to have in history.
- **Branching.** Trunk-based development. A short-lived branch per feature (`feat/2.5-budget-locking`), merged to `main` through a pull request. `main` stays deployable. Releases are tagged `v0.1.0`, and so on, at the end of each phase.
- **Pull requests.** Use a template with these sections:
  - What
  - Why
  - How verified (tests, SQL output, screenshots of query counts)
  - Risks / rollback
  - Django internals touched

  Even when you are working alone, write the PRs: they are evidence of engineering judgement.
- **Issues.** Keep one GitHub Project board, with an issue per roadmap feature. Label planted challenges `challenge`, and close each one with a link to its fix and its regression test.

### 13.2 README structure (final form)

1. A one-paragraph pitch and the architecture diagram.
2. **"Engineering highlights."** Five bullets, each linking to code and to a test. For example: "Concurrent approvals can't overspend budgets: `select_for_update` + `F()` + `CheckConstraint` → [service] [race test]".
3. Quick start: `cp .env.example .env && docker compose up`, a seed command, and a demo login.
4. API docs link (`/api/schema/swagger/`), plus a few example `curl` calls.
5. Testing: how to run the suite, how to run the race tests, how to run Locust.
6. Deployment: a link to doc 08 and the runbook.
7. Performance: a table of measured before/after results from `docs/perf/`.
8. Known limitations and future work. Being honest here builds credibility.

### 13.3 Engineering documentation set

| Document | Content |
|---|---|
| ADRs (`docs/adr/`) | Monolith, Django version, tenancy, Celery vs `django.tasks`, API auth. Add new ones as decisions come up (CASCADE vs DB_CASCADE, keyset pagination, cache invalidation strategy). |
| Database design | Doc 04 §D, kept current, plus `sqlmigrate` excerpts for the constraints |
| API contract | The OpenAPI schema committed as `docs/openapi.yaml`, with a CI check that it is up to date |
| AuthN/AuthZ | The policy functions, the permission matrix, and the tenant isolation strategy, including its limits |
| Background processing | The task list, queues, retry policy, the idempotency approach, and the failure modes |
| Deployment | Doc 08 plus the runbook |
| Performance investigations | One file per investigation: symptom → measurement → plan → fix → result |
| Security | Threat model (STRIDE-lite), the controls table, and known gaps |
| Known limitations | Examples: single currency, no RLS yet, the hot budget row caps throughput per department |

## Part 14: Interview Preparation

Answers must come from **your** code and measurements. For each question, practise a 60–90 second answer that follows this structure:

1. The context.
2. The decision you made.
3. The trade-off.
4. The evidence (a test or a measurement).

### 14.1 Core questions with senior follow-ups

| Question | What a strong answer covers | Senior follow-ups |
|---|---|---|
| **Why Django?** | Relational, workflow- and role-heavy domain. Admin for ops. Mature migrations and auth. You chose FastAPI elsewhere for a different reason. | "What would you lose with FastAPI + SQLAlchemy here?" "Where is Django fighting you?" (global settings, partial async, signals) |
| **How does Django process a request?** | WSGI handler → middleware onion built at startup → resolver → view → back out. Where exceptions are converted. Where `request.user` is set lazily. | "Your timing middleware sits above `CommonMiddleware`. Does it see the APPEND_SLASH redirect?" "What runs at import time versus per request?" |
| **How does QuerySet laziness work?** | Clone-on-chain. `Query` tree. Compiled on evaluation. `_result_cache`. `.count()` and `.exists()` don't fill the cache. | "Why might `if qs:` followed by `qs.count()` be worse than `len(qs)`, and when is it better?" "What does `iterator()` give up?" |
| **How did you prevent N+1, and keep it prevented?** | Selectors, `select_related`/`Prefetch`/annotations, query-count tests, `FETCH_RAISE` in tests. | "When is `prefetch_related` slower than a JOIN?" "Two `Count`s gave wrong numbers. Why?" |
| **How did you handle concurrent updates?** | A conditional UPDATE for state transitions (optimistic). `select_for_update` on budget rows (pessimistic). `F()` for arithmetic. A `CheckConstraint` as the backstop. The race tests. | "Why not `SERIALIZABLE`?" "Draw the deadlock you hit, and how lock ordering fixed it." "How does this scale if one department gets 100 approvals/s?" (sharded budget counters, reservation tokens, accepting soft limits) |
| **Where did you use transactions?** | Service-level `atomic()` per use case. Never across HTTP or S3 calls. `on_commit` for side effects. | "What happens if you catch `IntegrityError` inside `atomic()`?" "What does `ATOMIC_REQUESTS` cost?" "Payout: the provider succeeded, then your database write failed. Now what?" (reconciliation) |
| **How did you implement permissions?** | AuthN vs AuthZ. Per-org `Membership.role` vs Django Groups for staff. Policy functions shared by the queries and the actions. 404 for cross-tenant requests. The permission matrix test. | "Where can tenant scoping *not* protect you?" (raw SQL, admin, management commands, Celery tasks without org context) "Would RLS help? What breaks with connection reuse?" |
| **How did you prevent duplicate background task execution?** | At-least-once delivery + `acks_late` means duplicates *will* happen. State guards, unique constraints, and idempotency keys passed to the provider. | "Visibility timeout shorter than the task duration: what happens?" "Why isn't a Redis lock enough?" |
| **How did you approach caching?** | Measured first. Per-tenant versioned keys. Bump the version on commit. A TTL as the safety net. What is deliberately *not* cached. | "What causes a thundering herd here, and how would you prevent it?" "Why namespace by tenant? Give the security bug." |
| **How did you find and fix bottlenecks?** | Seeded data, `EXPLAIN ANALYZE`, a specific index you added (or removed), before/after numbers. | "Why a partial index on PENDING steps?" "What did `pg_stat_statements` show under Locust?" |
| **How did you test critical workflows?** | The layer table. `TransactionTestCase` only for races. `captureOnCommitCallbacks`. Migration tests. Mocking only at process boundaries. | "How do you know your race test actually raced?" "What's your flakiest test, and why?" |
| **What changes at 10× traffic?** | Connection math → PgBouncer (transaction pooling, and what breaks: session-level `SET`, server-side cursors). Read replicas for the dashboard. Separate Redis instances for cache and broker. Queue separation. Budget hot-row redesign. Partitioning the audit table. | "Which of these would you do *first*, and what metric tells you?" |

### 14.2 Deeper questions drawn from the project

1. Why does `Model.save()` not call `full_clean()`? How did that shape your validation layers?
2. You chose `CASCADE` or `DB_CASCADE` for expenses. What does the other option change for signals and the audit trail?
3. Explain `get_session_auth_hash` and what happens to other sessions when a password changes.
4. Why are your API tokens hashed with SHA-256 and not bcrypt? When would that be wrong?
5. Your idempotency key row is written in the same transaction as the effect. Why not store it in Redis?
6. How does `on_commit` behave inside nested `atomic()` blocks when the inner block rolls back?
7. How did you make the data migration safe for 200k rows? What would change at 50M rows?
8. What does `contextvars` give you over `threading.local`, and where could your tenant context leak? (Celery tasks, threads, and the test client)
9. Why is Celery beat a single replica? What made your scheduled jobs safe even with two?
10. Walk through a receipt from the `upload` request to `CLEAN`. Point out every place an attacker could interfere.
11. Your dashboard is cached per tenant. Explain the invalidation path, and the race between invalidation and recomputation.
12. Explain one Django source file you read and what it changed in your code.

### 14.3 The "mainframe to Django" story (prepare it)

Mainframe batch systems (COBOL, JCL, DB2) taught you three things:

- commit boundaries (`COMMIT` frequency in batch jobs)
- restartability
- audit trails

Kharcha applies the same ideas in modern form:

- `atomic()` scope
- idempotent, restartable tasks
- reimbursement batches with reconciliation
- an append-only ledger

This is a real, differentiated narrative. Use it.
