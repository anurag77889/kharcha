# Part 11: Testing, Security, and Performance

The aim is that the application can be trusted. A coverage percentage is not the goal. Every test below exists because it guards a bug that could cost someone money, or let them see data they shouldn't.

## 1. Testing strategy

### 1.1 Layers

| Layer | What to test | Database? | Base class / tools |
|---|---|---|---|
| Pure functions (policy evaluation, money helpers, routing rules) | Edge cases: limits, boundaries, rounding | No | Plain pytest, optionally `hypothesis` for money properties |
| Models / constraints | One test per constraint. Asserts `IntegrityError` | Yes | `TestCase` plus `with transaction.atomic():` around the failing statement |
| Services | Happy paths, every domain error, rollback on failure | Yes | `TestCase`, plus factories |
| Selectors | Correct rows, correct tenant scope, **query count** | Yes | `django_assert_num_queries` / `assertNumQueries`, plus `fetch_mode(FETCH_RAISE)` |
| API | Status codes, error envelope, auth classes, pagination, idempotency headers | Yes | DRF `APIClient`, pytest parametrisation |
| Permissions | Full role × action × tenant matrix | Yes | Parametrised API tests |
| Transactions / `on_commit` | Side effects happen only after commit | Yes | `TestCase.captureOnCommitCallbacks(execute=True)` |
| Concurrency | Races, lost updates, deadlocks | **Real commits** | `TransactionTestCase` with threads (see 1.3) |
| Migrations | Data migrations forward and backward | Yes | `MigrationExecutor` |
| Tasks | Task logic, retries, idempotency | Yes | Call the task function directly. Use `task.apply()` for retry semantics. Use a real worker in one CI job. |
| External services | Payout provider, S3 | Mocked, plus one real-ish integration | `respx`/`responses` for HTTP. MinIO in CI for S3 (prefer this over `moto`, because it is closer to the real thing). |
| Security | Headers, CSRF, throttles, upload validation, log scrubbing | Yes | API tests, `check --deploy` in CI |
| Performance | Query budgets, benchmark thresholds, load tests | Seeded | pytest benchmarks, Locust (manual or nightly) |

### 1.2 Real transactions vs. mocks

- **`TestCase`** wraps each test in a transaction and rolls it back afterwards. It is fast. But `on_commit` callbacks never fire on their own (you need `captureOnCommitCallbacks`). Row locks are also meaningless here, because there is only one connection.
- **`TransactionTestCase`** really commits, then truncates tables after each test. It is slow. Use it **only** for concurrency tests and for checking true commit behaviour.
- **Mock** only at process boundaries: HTTP to the provider, email sending, and the clock (`time-machine`). Never mock the ORM or your own services in service tests, because then the test proves nothing about the database.

### 1.3 Testing race conditions

Here is the pattern for running two requests at exactly the same moment:

```python
# Illustrative skeleton (Phase 2.4/2.5) — you write the real one.
class ApproveRaceTest(TransactionTestCase):
    def test_two_approvers_one_winner(self):
        barrier = threading.Barrier(2)
        results = []
        def worker(member):
            try:
                barrier.wait()                       # start together
                results.append(approve_step(member, step_id, key=uuid4().hex))
            except DomainError as e:
                results.append(e.code)
            finally:
                connection.close()                   # each thread has its own connection
        threads = [threading.Thread(target=worker, args=(m,)) for m in (fin1, fin2)]
        ...
        assert sorted(map(str, results)).count("STEP_ALREADY_DECIDED") == 1
```

Rules for these tests:

1. Every thread must use its own database connection. Django provides this automatically, because connections are per-thread.
2. Close the connection in a `finally` block.
3. Run the test in a loop (`pytest --count=50` with pytest-repeat) before you trust it.
4. To force the bad interleaving on purpose, inject a sleep or a barrier between the read and the write.

### 1.4 Meaningful test scenarios (examples)

- An employee in org A requests a report in org B and gets **404**. The same request with an org B admin token also gets 404, because the token is scoped to org A.
- A manager with `approval_limit = 5000` tries to approve a 6000 report and gets **403 APPROVAL_LIMIT_EXCEEDED**. The step stays PENDING.
- A submitter who also holds the finance role tries to approve their own report's finance step and gets **403 SELF_APPROVAL_FORBIDDEN**.
- A budget has 10,000 remaining. Three concurrent final approvals of 4,000 each arrive. Exactly two succeed. `consumed = 8000`, and the ledger sums to 8000.
- `submit` is sent twice with the same `Idempotency-Key`. Both calls return an identical 200 response, and there is one approval step. Sending the same key with a different body returns **409**.
- The payout task gets a timeout from the provider (which actually processed the payment), then retries. The provider sees the same idempotency key, and there is exactly one payment.
- An approval transaction rolls back. No notification task is enqueued.
- The receipt is a PNG file declared as `application/pdf`, and it is REJECTED. A 25 MB upload is refused by the S3 policy.
- The dashboard is cached. A report is approved, and the next dashboard read reflects it.
- The inbox runs the same number of queries at 1 row and at 100 rows.
- The data migration that splits merchant names preserves all 200k rows. A backward migration restores them.

## 2. Security checklist

| Area | Control in Kharcha | How it is verified |
|---|---|---|
| **CSRF** | `CsrfViewMiddleware`. DRF `SessionAuthentication` enforces CSRF. Bearer tokens are exempt, because they are not sent by browsers automatically. | A test where a session POST without a token gets 403. `CSRF_TRUSTED_ORIGINS` is set explicitly in prod. |
| **XSS** | JSON API, so there is no HTML rendering of user data. Email templates are auto-escaped. CSP via `SECURE_CSP` (Django 6.0+) for the admin and browsable API. | A test that a `<script>` in `merchant` is escaped in the email. Response headers are checked. |
| **SQL injection** | ORM parameterisation only. Any `RawSQL` or `extra` requires a review comment. Dashboard `group_by` uses an allow-list. | A grep check in CI for `.raw(`, `.extra(`, `cursor.execute` with f-strings. A test that `group_by=;DROP` returns 400. |
| **Clickjacking** | `XFrameOptionsMiddleware` (`DENY`), plus CSP `frame-ancestors 'none'` | A header test |
| **CORS** | Not enabled. There is no browser SPA in the MVP. If one is added later, use `django-cors-headers` with an explicit origin list and never `*` together with credentials. | — |
| **Cookies and sessions** | `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`, `SESSION_COOKIE_HTTPONLY`, `SameSite=Lax`, rotation on login (`cycle_key`), a short `SESSION_COOKIE_AGE` for staff | `check --deploy` and a login rotation test |
| **Passwords** | PBKDF2 (the default), with Argon2 as an optional upgrade. `AUTH_PASSWORD_VALIDATORS` enabled. Login throttled. | A test that the hash upgrades on login after the iteration count is raised |
| **Rate limiting** | DRF `ScopedRateThrottle` backed by the Redis cache: login 5/min per IP+email, token creation, exports | Throttle tests return 429 |
| **API authentication** | Tokens look like `kh_<prefix>_<secret>`. Only a SHA-256 of the secret is stored. Lookup is by prefix, then `hmac.compare_digest`. Tokens have scopes, expiry and revocation. | Tests cover a revoked token, an expired one, a wrong scope and a wrong org |
| **Authorization** | Policy functions used by both permissions and selectors. Tenant-scoped managers. 404 for cross-tenant requests. | The **permission matrix** (below) |
| **Input validation** | Serializer, then service, then database constraint layers. Decimal quantisation. Date bounds. Allow-lists. | Tests at each layer |
| **File uploads** | Presigned POST with `content-length-range` and an exact `Content-Type`. Keys are generated by the server. The bucket is private with Block Public Access on. A validation task checks magic bytes and size. A short-TTL GET URL is issued only after an authz check. `Content-Disposition: attachment` on download. | Upload tests with polyglot and spoofed files |
| **Secrets** | Environment variables only. `.env` is git-ignored. Production uses AWS SSM or Secrets Manager, injected into the task definition. `gitleaks` runs in pre-commit and CI. | CI secret scan |
| **Logging hygiene** | Strip `Authorization`, `Cookie`, `password` and token values in the logging filter and Sentry `before_send`. | A log-scrubbing test |
| **Production config** | `DEBUG=False`, explicit `ALLOWED_HOSTS`, `SECURE_PROXY_SSL_HEADER` (behind the ALB), `SECURE_SSL_REDIRECT`, HSTS (starting small), `SECURE_CONTENT_TYPE_NOSNIFF`, `SECRET_KEY` from the environment | `manage.py check --deploy --fail-level WARNING` in CI against prod settings |
| **Admin** | It lives at a non-default path, staff-only, with 2FA in Phase 7. Destructive actions are audited. Money fields are read-only for the support group. | Admin permission tests |
| **Dependencies** | `pip-audit` in CI. Renovate or Dependabot. | CI |

### Permission matrix (abbreviated; the real one is a parametrised test)

| Action ↓ / Role → | Employee (own) | Employee (other's) | Manager (team, within limit) | Manager (over limit) | Finance | Org admin | Any role, other org |
|---|---|---|---|---|---|---|---|
| View report | 200 | 404 | 200 | 200 | 200 | 200 | 404 |
| Edit draft | 200 | 404 | 403 | 403 | 403 | 403 | 404 |
| Submit | 200 | 404 | 403 | 403 | 403 | 403 | 404 |
| Approve step | 403 (self) | 404 | 200 | 403 | 200 (pooled) | 403 unless assigned | 404 |
| Create batch | 403 | — | 403 | 403 | 201 | 403 | 404 |
| Manage members | 403 | — | 403 | 403 | 403 | 200 | 404 |

## 3. Performance method

1. **Measure before you change anything.** Use seeded data (§Phase 5) and record a baseline in `docs/perf/<date>-<endpoint>.md`, covering:
   - p50 and p95 latency
   - query count
   - the slowest query, with its `EXPLAIN (ANALYZE, BUFFERS)`
2. **Find the bottleneck.** Look at the query count first (N+1), then each query's plan (sequential scans, sorts, row estimates), then Python time (py-spy or cProfile), then lock waits (`pg_locks`, `pg_stat_activity`).
3. **Fix one thing at a time** and measure again.
4. **Write a regression guard.** That means a query-count test, a benchmark threshold, or an index-usage note.

### Tools

| Tool | Use |
|---|---|
| django-debug-toolbar | Per-request queries in dev |
| `CaptureQueriesContext` / `assertNumQueries` | Tests |
| `FETCH_RAISE` (6.1) | Make lazy loads fail in tests |
| `EXPLAIN (ANALYZE, BUFFERS)` | Query plans |
| `pg_stat_statements` | Top queries by total time, in staging |
| py-spy | Python CPU profiling, with no code changes |
| tracemalloc | Memory used by exports |
| Locust | Load testing |

### Performance targets (from the NFRs)

| Measure | Target |
|---|---|
| Inbox / report detail | p95 < 300 ms at 50k reports per tenant |
| Dashboard | Cold < 500 ms, cached < 50 ms |
| List endpoints | Constant query count |
| Export of 200k rows | Peak memory < 100 MB |
