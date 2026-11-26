# Part 9: Development Roadmap (learning-first)

## How this roadmap works

**Mentor mode.** You write the code, and I write the briefs and review your work. Each feature brief follows the same eight steps:

1. **Business requirement**: the problem the feature solves.
2. **Django mechanism**: which part of Django is involved.
3. **How it works internally**: the execution flow inside Django.
4. **Implementation task**: what to build. I won't give you the solution up front.
5. **Investigation**: which docs and source to read. File paths point at the [source guide](06-django-source-guide.md).
6. **Debugging exercise**: a realistic bug for you to find and fix.
7. **Verification**: how you prove it works, using tests, SQL, or logs.
8. **Engineering reflection**: questions to answer in writing in `notes/` before moving on.

Phases 0–2 have full briefs below. Phases 3–7 have condensed briefs. I'll expand each one into the full eight steps when you reach that phase, because the details will depend on what you've built by then.

**Timeline.** Plan on about 15 hours a week, which gives 16–20 weeks to the end of Phase 6. Phase 7 is optional. Don't move to the next phase until the current phase's completion criteria pass.

**Git rule from day 1.** Do one feature per branch, then open a PR against `main` that I review. Write conventional commit messages, and make every PR description answer three questions: *what*, *why*, and *how verified*.

---

## Phase 0: Product and architecture understanding (week 1)

**Learning objectives**

- Explain the domain without looking at notes.
- Defend every table and constraint in [04-flagship-design](04-flagship-design.md).

**Tasks**

1. Read docs 01–04. Write `notes/phase0-domain.md` covering:
   - each persona's day,
   - the report state machine drawn by hand (states, transitions, who may trigger each one),
   - three ways money could be lost or double-paid.
2. Redraw the ER diagram from memory. Compare it with the doc and write down every difference.
3. Map each constraint in D.3 to the bug it prevents.
4. Install Python 3.13, uv, Docker Desktop, and PostgreSQL client tools. Clone Django's `stable/6.1.x` branch somewhere outside the project as a reading copy: `git clone --branch stable/6.1.x --depth 1 https://github.com/django/django`.

**Deliverables**

- `notes/phase0-domain.md`
- the state-machine diagram

**Completion criteria.** You can answer the following without notes:

- Why is `Payout.report` unique?
- Why is `org_id` on `Expense` and not only on `ExpenseReport`?
- Why must the notification not be sent inside the approval transaction?

**Common mistakes**

- Starting to code before the state machine is settled.
- Treating roles as global rather than per-org.

**Interview questions**

- Walk me through your domain model.
- Why did you denormalise `org_id`?

---

## Phase 1: Django foundations through the project (weeks 2–4)

**Learning objectives**

- Understand startup, settings, the app registry, the request lifecycle, URL routing, and views (both DRF and plain Django).
- Understand the custom user model.
- Customise the admin.
- Set up the testing harness and CI.

**Django concepts:** `django.setup()`, `Apps.populate`, `AppConfig.ready`, `LazySettings`, `WSGIHandler`, `BaseHandler.load_middleware`, `URLResolver`, `View.as_view`, DRF `APIView.dispatch`, `AbstractBaseUser`, `ModelAdmin`.

### Feature 1.1: Project skeleton, settings, and Docker Compose

1. **Requirement.** Everyone, including CI and future colleagues, must be able to run the same stack with one command. Secrets must never be committed.
2. **Mechanism.** The settings module is chosen by `DJANGO_SETTINGS_MODULE`. Setting `django.conf.settings` is a `LazySettings` object, `manage.py` runs `ManagementUtility`, and the app registry is filled by `django.setup()`.
3. **Internals.** Startup runs in this order:
   1. `manage.py` sets `DJANGO_SETTINGS_MODULE` and calls `execute_from_command_line`.
   2. `ManagementUtility.execute()` parses the arguments, then calls `django.setup()`.
   3. `setup()` configures logging, then calls `apps.populate(settings.INSTALLED_APPS)`.
   4. `populate()` runs in three passes: it imports each `AppConfig`, then imports each app's `models` module, then calls every `ready()`.

   Settings themselves are not read until the first attribute access (`settings.X`), which calls `LazySettings._setup()` and imports your module. That is why importing a model at the top of `settings.py` fails.
4. **Task.**
   - Create the `config/` package with settings split into `base`, `dev`, `test`, and `prod`, each reading the environment through a tiny helper. Write it yourself; don't add django-environ yet.
   - Add `compose.yaml` with `web`, `db` (postgres:17), `redis`, and `minio` services.
   - Add `/healthz`, which only returns 200, and `/readyz`, which checks the database and Redis.
   - Add `.env.example`.
   - Set up the uv project with ruff and pre-commit.
5. **Investigation.** Read the following:
   - `django/core/management/__init__.py`: `execute_from_command_line` and `ManagementUtility.execute`.
   - `django/__init__.py`: `setup`.
   - `django/apps/registry.py`: `Apps.populate`.
   - `django/conf/__init__.py`: `LazySettings._setup`.
6. **Debugging exercise.** Put `from apps.organizations.models import Organization` at module level in `apps/common/apps.py`, then run `manage.py check`. Explain the exact traceback: *which pass of `populate()` were you in?* Then move the import into `ready()` and explain why that works.
7. **Verification.**
   - `docker compose up` brings everything up.
   - `/readyz` returns 503 when you stop the `db` container and recovers when you start it again.
   - `manage.py check` passes, and `manage.py check --deploy --settings=config.settings.prod` lists the warnings you expect at this stage.
8. **Reflection.**
   - Why is settings access lazy?
   - What breaks if two settings modules both set `INSTALLED_APPS` but in a different order?
   - What does `ready()` guarantee that module import time doesn't?

### Feature 1.2: Custom user model (before the first migration)

1. **Requirement.** Users log in with email. A username field would only confuse people.
2. **Mechanism.** `AUTH_USER_MODEL`, `AbstractBaseUser`, `PermissionsMixin`, `BaseUserManager`, `USERNAME_FIELD`, and `REQUIRED_FIELDS`.
3. **Internals.**
   - `get_user_model()` resolves the user model through the app registry using `AUTH_USER_MODEL`.
   - Foreign keys to `settings.AUTH_USER_MODEL` are resolved lazily as string references.
   - The swappable model affects migration dependencies (`migrations.swappable_dependency`). That is why changing the user model after the first migration is painful: every FK to it, and the migration graph itself, depend on it.
4. **Task.**
   - Create `accounts.User` with `email` unique (normalised to lowercase), `full_name`, `is_active`, `is_staff`, and `date_joined`.
   - Write a manager with `create_user` and `create_superuser`.
   - Register it in the admin with `UserAdmin`, adapting the fieldsets.
   - Implement case-insensitive uniqueness with `UniqueConstraint(Lower("email"))`.
5. **Investigation.** Read:
   - `django/contrib/auth/base_user.py`: `AbstractBaseUser.set_password`, `check_password`, `normalize_username`.
   - `django/contrib/auth/models.py`: `PermissionsMixin`.
   - `django/contrib/auth/__init__.py`: `get_user_model`.
6. **Debugging exercise.**
   - Create a user as `A@x.com`, then try to create `a@x.com`. Did your constraint catch it, or only your manager's normalisation? Prove which one by calling `User.objects.bulk_create` directly, which skips the manager.
   - Then run `sqlmigrate` and read the generated index.
7. **Verification.** Write tests for:
   - duplicate emails in different case → `IntegrityError`,
   - `create_superuser` flags,
   - a password that is stored hashed, never in plaintext. Check that the stored string starts with `pbkdf2_sha256$`.
8. **Reflection.**
   - Why does Django recommend a custom user model even when you don't need one yet?
   - What is the risk of putting profile fields on `User` versus on a separate table?

### Feature 1.3: Organizations, memberships, departments, and the admin as an ops tool

1. **Requirement.** Internal ops staff must be able to inspect and fix tenant setup. Customers manage their own setup through the API (built in Phase 3).
2. **Mechanism.** Models, `Meta.constraints`, `ModelAdmin` (`list_display`, `list_select_related`, `search_fields`, `raw_id_fields`, `get_queryset`, actions), and `InlineModelAdmin`.
3. **Internals.**
   - The admin builds a changelist QuerySet through `ModelAdmin.get_queryset()` → `ChangeList.get_queryset()` (which applies filters and search), then paginates it.
   - Every callable in `list_display` that touches a relation runs one query per row unless you use `list_select_related` or an annotation.
4. **Task.**
   - Build `Organization`, `Department`, and `Membership` with the constraints from D.3.
   - In the admin, the Membership changelist shows the user's email, the org, the department, and the manager's email.
   - Add an admin action, "deactivate memberships", that writes an audit event (a stub function for now).
5. **Investigation.** Read `django/contrib/admin/options.py` (`ModelAdmin.get_queryset`, `changelist_view`) and `django/contrib/admin/views/main.py` (`ChangeList`).
6. **Debugging exercise.**
   - Seed 500 memberships and open the changelist with debug-toolbar enabled. Count the queries.
   - Fix it, then count again.
   - Explain why `list_select_related = True` isn't always the right fix.
7. **Verification.** The admin changelist query count is constant (it doesn't grow with the number of rows). Assert this with `django_assert_max_num_queries`.
8. **Reflection.**
   - Should the admin enforce tenant rules?
   - Who uses the admin, and what is the blast radius of an admin bug?

### Feature 1.4: Request lifecycle made visible (correlation ID and timing middleware)

1. **Requirement.** When a customer reports "approval failed at 10:42", ops must be able to find every log line from that request.
2. **Mechanism.** Function-style middleware, `BaseHandler.load_middleware`, the `get_response` chain, `process_exception`, and `contextvars`.
3. **Internals.**
   - At startup, `load_middleware()` walks `MIDDLEWARE` **in reverse** and wraps each middleware around the previous handler. The result is a single callable onion.
   - Each request enters the outermost layer first. `_get_response()` resolves the URL, applies view middleware, then calls the view.
   - Exceptions are converted to responses by `convert_exception_to_response` at *each* layer. That is why a middleware further out still sees a response object, even when a view raised.
4. **Task.**
   - Write `RequestIdMiddleware`. It reads `X-Request-ID`, or generates a ULID/UUID if there isn't one. It stores the value in a `contextvar` and on `request.request_id`, and adds the response header.
   - Write `TimingMiddleware`, which logs method, path, status, duration in ms, and the request ID as JSON.
   - Add a logging `Filter` that injects `request_id` into every record.
5. **Investigation.** Read:
   - `django/core/handlers/base.py`: `load_middleware`, `_get_response`.
   - `django/core/handlers/exception.py`: `convert_exception_to_response`.
   - `django/utils/deprecation.py`: `MiddlewareMixin`.
6. **Debugging exercise.**
   - Put `TimingMiddleware` *before* `SecurityMiddleware` in `MIDDLEWARE`, then *after* `CommonMiddleware`.
   - Make a request to a URL without a trailing slash, which triggers an `APPEND_SLASH` redirect.
   - In each configuration, which middleware logs the 301, and which never sees the request? Explain why.
   - Finally, raise an exception in a view and check that your timing log still records a 500.
7. **Verification.** Write tests that check:
   - the response header echoes a provided ID,
   - an ID is generated when none is provided,
   - log records captured with `caplog` carry the ID,
   - the `contextvar` is reset after the request. Test this with two sequential requests in one test.
8. **Reflection.**
   - Why `contextvars` instead of `threading.local`?
   - What would break under ASGI if you used a thread-local?
   - Which work should *not* go in middleware?

### Feature 1.5: First endpoints, built twice

1. **Requirement.** `GET /api/v1/me/` returns the current user and their memberships.
2. **Mechanism.** URL resolution, a plain Django `View` with `JsonResponse`, and then DRF's `APIView` and `Serializer`.
3. **Internals.**
   - `path()` builds a `URLPattern`, and `include()` builds a `URLResolver`.
   - `resolve()` walks the patterns in order. Converters (`<uuid:id>`) call `to_python`; if that raises `ValueError`, the URL doesn't match and resolution continues.
   - DRF's `APIView.as_view()` wraps the view in `csrf_exempt`, then re-applies CSRF inside `SessionAuthentication.enforce_csrf`. **This surprises a lot of people.**
4. **Task.**
   - Implement the endpoint first as a plain Django `View` that returns `JsonResponse`, with manual auth checks.
   - Then implement it again with DRF.
   - Write down the differences: content negotiation, auth classes, exception handling, and the browsable API.
   - Keep only the DRF version.
5. **Investigation.** Read:
   - `django/urls/resolvers.py`: `URLResolver.resolve`, `RoutePattern.match`.
   - `django/views/generic/base.py`: `View.as_view`, `dispatch`.
   - DRF `rest_framework/views.py`: `APIView.dispatch`, `initial`, `handle_exception`.
6. **Debugging exercise.** A POST to a DRF endpoint works with `curl` but returns 403 from the browser while you're logged in. Find out why (CSRF only applies when `SessionAuthentication` is used) and fix it properly. Don't use `csrf_exempt`.
7. **Verification.** Write API tests for anonymous (401), logged in (200), and the correct shape. Write a URL test showing that an invalid UUID returns 404 from the resolver, not 500 from the view.
8. **Reflection.**
   - Where exactly is `request.user` set?
   - Why is it lazy?

### Feature 1.6: Test harness and CI

- **Task.**
  - Set up pytest-django and factory_boy factories for `User`, `Organization`, and `Membership`.
  - Use `config.settings.test`, which sets the MD5 hasher for speed. Explain why that is safe *only* in tests.
  - Add a GitHub Actions workflow that runs ruff, runs pytest against Postgres 17 and Redis service containers, and runs `makemigrations --check`.
- **Debugging exercise.** A test passes locally but fails in CI because of test ordering. Plant this deliberately: one test mutates a module-level dict. Find the cause with `pytest -p no:randomly` versus `pytest-randomly`.

**Phase 1 completion criteria**

- CI is green.
- `docker compose up` works from a clean clone.
- 100% of Phase 1 features have tests.
- `notes/` has a written answer for every reflection question.
- You can draw the middleware onion for your `MIDDLEWARE` setting from memory.

**Common mistakes**

- Migrating before the custom user model exists.
- Logic in `AppConfig.ready()` that queries the database.
- Not pinning dependencies.
- Using SQLite for tests. It is not acceptable here, because you need row locks and partial indexes.

**Interview questions**

- What happens between Gunicorn accepting a socket and your view running?
- Why does middleware order matter? Give a concrete example from your project.

---

## Phase 2: ORM and database engineering (weeks 5–8)

**Learning objectives**

- Understand QuerySet laziness and caching.
- Learn to read SQL generation.
- Use relations, constraints, indexes, and aggregation.
- Learn transactions, locks, and migrations that preserve existing data.

### Feature 2.1: Categories, policies, reports, and expenses with constraints

1. **Requirement.** Invalid money states must be impossible, even through bugs or the admin.
2. **Mechanism.** `Meta.constraints` (`CheckConstraint`, `UniqueConstraint` with `condition` and expressions), `Meta.indexes`, `DecimalField`, choices compared with database checks, and 6.1 `on_delete=DB_CASCADE`.
3. **Internals.**
   - The migration autodetector compares model state with the migration state and emits `AddConstraint` operations.
   - The schema editor translates each operation into DDL.
   - `Model.full_clean()` calls `validate_constraints()`, which evaluates constraints *in Python* by running queries. But `save()` does **not** call `full_clean()`. So the database is the only guarantee that holds for every code path.
4. **Task.**
   - Implement the models and every constraint from D.3 that applies to them.
   - For `Expense.report`, choose between `CASCADE` and `DB_CASCADE`. Justify the choice in an ADR. Hint: think about signals, the audit log, and bulk deletes.
5. **Investigation.** Read:
   - `django/db/models/constraints.py`: `CheckConstraint.validate`, `UniqueConstraint.validate`.
   - `django/db/models/deletion.py`: `Collector`, and the `DatabaseOnDelete` objects (`DB_CASCADE` and friends).
   - Then run `manage.py sqlmigrate expenses 0001` and read the DDL it produces.
6. **Debugging exercise.**
   - Create an expense with `amount=Decimal("0.00")` through `Expense.objects.create(...)` and through the admin. One fails with a nice form error, the other with `IntegrityError`. Explain the difference.
   - Then try `amount=0.1 + 0.2` as a float and inspect what's stored.
7. **Verification.**
   - Constraint tests: one test per constraint, asserting `IntegrityError` inside `transaction.atomic()`. Explain why the `atomic()` is needed inside a `TestCase`.
   - A `sqlmigrate` snapshot is reviewed in the PR.
8. **Reflection.**
   - Application-level versus database-level validation: give one rule that can only live in each.
   - What does `DB_CASCADE` give up compared with `CASCADE`?

### Feature 2.2: QuerySets, laziness, and the "my reports" list

1. **Requirement.** Employees list their reports, with filtering by state and a date range.
2. **Mechanism.** `QuerySet` chaining, `Query` (the SQL tree), `SQLCompiler`, and `_result_cache`.
3. **Internals.**
   - `Report.objects.filter(...)` clones the QuerySet and adds a `WhereNode` to its `Query`. No SQL runs.
   - SQL is compiled only on evaluation: iteration, `len()`, `list()`, `bool()`, slicing with a step, or `repr()` in the shell. At that point `SQLCompiler.as_sql()` builds the SQL and `execute_sql()` runs it.
   - Results land in `_result_cache`. Iterating the same QuerySet again doesn't query again, but calling `.filter()` on it creates a new, uncached clone.
   - `.exists()` and `.count()` issue their own queries and don't fill the cache.
4. **Task.**
   - Write a custom `ExpenseReportQuerySet` with `for_org(org)`, `submitted_by(member)`, and `in_state(*states)`.
   - Expose it through `ExpenseReport.objects = ExpenseReportQuerySet.as_manager()`.
   - Write the `reports_for_member()` selector.
5. **Investigation.** Read:
   - `django/db/models/query.py`: `QuerySet.__iter__`, `_fetch_all`, `_clone`, `filter`, `_filter_or_exclude`.
   - `django/db/models/sql/query.py`: `Query.add_q`.
   - `django/db/models/sql/compiler.py`: `SQLCompiler.as_sql`, `execute_sql`.
6. **Debugging exercise.** This view runs **3** queries when you expected 1. Find each one:

   ```python
   qs = reports_for_member(m)
   if qs: ...
   total = qs.count()
   data = [r for r in qs.filter(state="DRAFT")]
   ```

7. **Verification.** Use `django_assert_num_queries` tests, and use `CaptureQueriesContext` to print the SQL. Write a test asserting that building the QuerySet without evaluating it runs **zero** queries.
8. **Reflection.**
   - Why is laziness a good default?
   - When does it bite?
   - What does `iterator()` give up, and what does it gain?

### Feature 2.3: Relations and the approvals inbox (N+1 queries on purpose)

1. **Requirement.** Approvers see pending steps with the report title, submitter name, department, total, and the number of receipts.
2. **Mechanism.** FK forward and reverse descriptors, `select_related` (JOIN), `prefetch_related` and `Prefetch` (a second query with `IN`), `annotate(Count(...))`, and 6.1 `fetch_mode`.
3. **Internals.**
   - `step.report` is a `ForwardManyToOneDescriptor.__get__`. On a cache miss it runs a query.
   - `select_related` adds JOINs, and `RelatedPopulator` fills the related instances from the same row.
   - `prefetch_related` runs a second query (`WHERE report_id IN (...)`) and stitches the results together in Python through `prefetch_related_objects`.
   - In 6.1, `FETCH_PEERS` batches lazy loads across instances from the same QuerySet, and `FETCH_RAISE` raises `FieldFetchBlocked` instead of querying.
4. **Task.**
   - First, write the inbox selector naïvely and serialize it with nested serializers.
   - Measure it.
   - Fix it with `select_related` and an annotation. Decide whether the receipt count needs `prefetch` or `annotate(Count)`.
   - Make tests run inbox selectors under `fetch_mode(FETCH_RAISE)`.
5. **Investigation.** Read:
   - `django/db/models/fields/related_descriptors.py`: `ForwardManyToOneDescriptor.__get__`.
   - `django/db/models/query.py`: `prefetch_related_objects`, `RelatedPopulator`.
   - `django/db/models/fetch_modes.py`.
6. **Debugging exercise.**
   - Seed 50 pending steps. The naïve version runs about 151 queries. Explain the arithmetic.
   - Then plant a subtler bug: `annotate(Count("expenses"), Count("approval_steps"))` gives inflated counts. Explain why (JOIN multiplication) and fix it with `distinct=True` or a `Subquery`.
7. **Verification.** Assert that the query count is constant at 5 and at 50 rows. The `FETCH_RAISE` test passes.
8. **Reflection.**
   - When is `prefetch_related` better than `select_related`?
   - Why can two `Count`s over different joins be wrong?

### Feature 2.4: Transactions and the report state machine

1. **Requirement.** Submitting a report must atomically validate the policy, change state, create the first approval step, and write an audit row. A double click must not create two steps.
2. **Mechanism.** `transaction.atomic` (outermost block versus savepoint), autocommit, conditional `.update()`, the `version` field, and partial unique constraints.
3. **Internals.**
   - `Atomic.__enter__`: if the connection isn't already in an atomic block, it turns autocommit off (`BEGIN`). If it is already in one, it creates a savepoint.
   - `__exit__` commits or rolls back at the outermost level, or releases or rolls back to the savepoint when nested.
   - Catching a database exception *inside* an `atomic` block without a nested `atomic` leaves the connection in a broken-transaction state. The symptom is `TransactionManagementError` on the next query.
4. **Task.**
   - Implement `services.submit_report(member, report_id, expected_version, idempotency_key)`. Use a conditional update for the state transition.
   - Map `IntegrityError` from `one_pending_step_per_report` to a domain error.
   - Write `recall_report`.
5. **Investigation.** Read `django/db/transaction.py` (`Atomic.__enter__` and `__exit__`, `savepoint`, `on_commit`) and `django/db/backends/base/base.py` (`set_autocommit`, `on_commit`, `run_and_clear_commit_hooks`, `validate_no_broken_transaction`).
6. **Debugging exercise.** A planted bug:

   ```python
   with atomic():
       try:
           ApprovalStep.objects.create(...)
       except IntegrityError:
           pass
       AuditEvent.objects.create(...)
   ```

   Explain the error, and give two correct fixes.
7. **Verification.**
   - A **race test**: two threads, each with its own DB connection, call `submit_report` at the same time using `TransactionTestCase`. Exactly one succeeds and the other gets `INVALID_STATE` or `STALE_VERSION`. Exactly one step exists.
   - A rollback test: patch audit writing to raise, then assert that nothing persisted.
8. **Reflection.**
   - Optimistic locking (version) versus pessimistic locking (`select_for_update`): which one did you use here, and why?
   - What isolation level does Postgres use by default?
   - Could your conditional update still allow anomalies under READ COMMITTED?

### Feature 2.5: Budgets with row locks and an append-only ledger

1. **Requirement.** Concurrent final approvals must never overspend a department budget.
2. **Mechanism.** `select_for_update()`, `F()` expressions, `CheckConstraint` as a backstop, and lock ordering.
3. **Internals.**
   - `select_for_update()` compiles to `SELECT … FOR UPDATE`. It must run inside `atomic()`, otherwise you get `TransactionManagementError`.
   - The lock is held until the transaction commits.
   - `F("consumed_amount") + amount` compiles to `SET consumed = consumed + %s`. The database does the arithmetic on the current row, so no value read in Python is ever stale.
4. **Task.** Implement `budgets.services.consume(budget_period, report)`, which is called from the final step of `approve_step`. It should:
   - lock the budget,
   - check what remains,
   - increment with `F()`,
   - insert a ledger entry, relying on the unique `(budget, report, kind)` constraint.

   Then write a `reconcile_budget` management command that compares `consumed_amount` with `SUM(ledger)`.
5. **Investigation.** Read the `select_for_update` path in `django/db/models/query.py` and the `for_update` handling in `sql/compiler.py`. Also read the PostgreSQL docs on [explicit locking](https://www.postgresql.org/docs/17/explicit-locking.html) and [transaction isolation](https://www.postgresql.org/docs/17/transaction-iso.html).
6. **Debugging exercise.** Three planted bugs, one at a time:
   - (a) Read-modify-write: `b.consumed_amount += amt; b.save()`, without a lock. Make the race test fail, and explain the lost update.
   - (b) Lock budgets in report order in one code path and department order in another, then trigger a **deadlock** with two threads. Read the Postgres deadlock log line, then fix it with consistent lock ordering.
   - (c) Remove the Python check and confirm that the `CheckConstraint` still prevents overspending.
7. **Verification.**
   - The race test: N threads approve N reports against a budget that fits only N−1 of them. Exactly one gets `BUDGET_EXCEEDED`, and `consumed == SUM(ledger)`.
   - The `reconcile_budget` command reports zero drift.
8. **Reflection.**
   - Why lock the budget pessimistically but use optimistic locking for reports?
   - What is the throughput cost of a hot budget row, and how would you redesign if one department received 100 approvals per second?

### Feature 2.6: Migrations that preserve existing data

1. **Requirement.** Product changes are coming:
   - (a) the "merchant" text field must become a normalised `Merchant` table,
   - (b) a new NOT NULL `cost_center` column must be added to a table holding production data.
2. **Mechanism.** `RunPython` with the historical `apps.get_model`, `SeparateDatabaseAndState`, the multi-step deploy pattern (add a nullable column → backfill in batches → add NOT NULL), `atomic = False` on migrations, and `AddIndexConcurrently` (from `django.contrib.postgres.operations`).
3. **Internals.**
   - `MigrationLoader` builds a graph from the files on disk plus the `django_migrations` table.
   - `MigrationExecutor` plans the operations and applies them.
   - Each operation mutates a *project state*. `RunPython` receives historical models built from that state, which is why importing your real models inside a migration is wrong.
4. **Task.** Seed 200k expenses. Write the multi-migration sequence for (a) and (b). Time each step, and note which ones lock tables.
5. **Investigation.** Read:
   - `django/db/migrations/executor.py`: `migrate`, `apply_migration`.
   - `django/db/migrations/loader.py`.
   - `django/db/migrations/operations/special.py`: `RunPython`, `SeparateDatabaseAndState`.
6. **Debugging exercise.** A planted bug: the data migration imports `from apps.expenses.models import Expense`, and it works today. Add a field to `Expense` in a later migration, then run all migrations on an empty database. Explain the failure.
7. **Verification.**
   - A migration test using `MigrationExecutor`: migrate to the old state, create data with historical models, migrate forward, and assert the data is correct. Then migrate back.
   - CI runs `makemigrations --check`.
8. **Reflection.**
   - Which operations take an `ACCESS EXCLUSIVE` lock in Postgres?
   - How would you run this migration on a table with 50M rows while it's serving traffic?

**Phase 2 completion criteria**

- Every constraint has a test.
- The inbox and list endpoints run a constant number of queries.
- The race tests for submit and budget pass 50 times in a loop.
- `reconcile_budget` reports zero drift.
- The migration test passes forwards and backwards.

**Common mistakes**

- `select_for_update` outside `atomic`.
- Catching `IntegrityError` inside an atomic block.
- Importing real models in migrations.
- `Count` over multiple joins without `distinct`.
- Using floats for money.

**Interview questions**

- How does QuerySet lazy evaluation work?
- How did you prevent N+1 queries, and how do you stop them from coming back?
- How did you handle concurrent updates?
- Walk me through the transaction for the final approval.

---

## Phase 3: Authentication, authorization, and tenant isolation (weeks 9–10), condensed

| Feature | Mechanism | Task | Planted challenge |
|---|---|---|---|
| 3.1 Login/logout and session security | `authenticate()` iterates over `AUTHENTICATION_BACKENDS`. `login()` calls `cycle_key()` and stores `_auth_user_id`, `_auth_user_backend`, and `_auth_user_hash`. `SessionMiddleware`. Cookie flags. | Session login API, CSRF endpoint, "log out all devices" (rotate the session auth hash). | A session-fixation demonstration: what if `cycle_key()` weren't called? Then change a password and observe that other sessions are invalidated through `get_session_auth_hash`. |
| 3.2 Password hashing | `PBKDF2PasswordHasher`, `check_password` → `must_update` → re-hash on login | Raise the iteration count. Show that old hashes upgrade on the next login. | Read a stored hash and identify its algorithm, iteration count, and salt. |
| 3.3 Personal API tokens | Custom DRF `BaseAuthentication`. Prefix lookup plus `hashlib.sha256(secret)`, compared with `hmac.compare_digest`. Scopes. | Tokens look like `kh_<8-char prefix>_<32-byte secret>`. The secret is shown once. Implement `last_used_at` (throttled writes), expiry, and revocation. | A timing-safe comparison exercise. Why not store bcrypt for tokens? Hint: tokens have high entropy and lookup must be fast. |
| 3.4 Tenant context and scoped managers | Middleware resolves `org_slug` → `Organization` and sets a `contextvar`. `TenantScopedQuerySet` with explicit `.for_org()`. A default manager that **raises** if it is used without an org context in request code. | Scope every selector, and every viewset `get_queryset`. | **Cross-tenant leak:** a planted `ExpenseReport.objects.get(public_id=…)` in the receipt download view. Find it with the cross-tenant test matrix. |
| 3.5 Object-level policies | Policy functions shared by DRF `has_permission`/`has_object_permission` and by selectors | Write `can_view_report`, `can_edit_report`, `can_decide_step` (including the separation-of-duties rule and approval limits), and `can_manage_members` (including "cannot demote the last admin"). | A manager approves a report they submitted themselves, through a pooled finance step, because they also hold the finance role. Fix the rule and add the regression test. |
| 3.6 Staff and admin permissions | Django Groups and model perms for *internal* staff. `ModelAdmin.has_change_permission`. | "Support" staff can view but not edit money fields. Only "finance-ops" can run force actions. Admin actions are audited. | Explain why per-tenant roles do not use Django Groups. |

- **Security tests.** A parametrised matrix covers role × endpoint × same/other org, with the expected status for each combination (see doc 07).
- **Interview questions.**
  - Authentication vs authorization in your system?
  - How do you guarantee tenant isolation?
  - Why 404 and not 403?

## Phase 4: Production backend features (weeks 11–14), condensed

| Feature | Mechanism | Planted challenge |
|---|---|---|
| 4.1 Celery setup and notification emails | `config/celery.py`, `autodiscover_tasks`, `transaction.on_commit(lambda: notify.delay(...))`, email templates and context | **Notification before commit:** call `.delay()` inside `atomic()`. The worker reads stale data or a rolled-back step. Reproduce it with a worker that sleeps. Fix it with `on_commit`, and test with `captureOnCommitCallbacks(execute=True)`. |
| 4.2 Idempotency keys | `IdempotencyKey` rows written in the same transaction as the effect. Request hash comparison. Replay of the stored response. | A client retry after a timeout must not create a second batch. Two concurrent requests with the same key: the unique constraint makes one of them fail, so map that to 409 or a replay. |
| 4.3 Receipt uploads | django-storages, boto3 `generate_presigned_post` with conditions, a validation task (magic bytes, size, SHA-256), and short-TTL GET URLs | **Malicious upload:** an `.html` file renamed `receipt.pdf` with a spoofed `Content-Type`. Show that the presigned POST conditions *and* the magic-byte check stop it. Also try an oversized upload (rejected by `content-length-range`). |
| 4.4 Reimbursement batches and payouts | `select_for_update(skip_locked=True)`, Celery `autoretry_for`, `retry_backoff`, `acks_late`, an idempotency key sent to the provider, and no transaction held across HTTP | **Task executes twice:** kill the worker mid-task with `acks_late` enabled. The task re-runs, and the provider's idempotency key plus the `Payout` state guard prevent a double payment. **Provider down:** return 503 for 5 minutes. Observe backoff and the reconciliation sweep. |
| 4.5 Scheduled jobs | Celery beat: budget rollover (idempotent through the unique constraint), stuck-payout reconciliation, idempotency-key cleanup | Run two beat processes by accident, and show why the jobs are still safe. |
| 4.6 Caching the dashboard | `cache.get_or_set`, per-tenant versioned keys (`org:{id}:v{n}:dash:{hash}`), and a version bump via `on_commit` | **Stale cache:** approve a report and the dashboard doesn't change. Fix it with the version bump. **Key leak:** a key without the org ID serves org A's numbers to org B. Find it with a test. |
| 4.7 Streaming CSV export | `StreamingHttpResponse` versus an async export to S3, and `QuerySet.iterator(chunk_size=…)` | **Memory blow-up:** `list(qs)` over 200k rows. Measure it with `tracemalloc`, then fix it. Decide when a synchronous stream is fine and when to use an async job. |
| 4.8 `django.tasks` experiment | Django 6.0+ `@task`, `ImmediateBackend` in tests | Port one task, compare it with the Celery version, and record the outcome in ADR-0004. |

## Phase 5: Testing and performance (weeks 15–16), condensed

- **Seed realistic data.** Write a management command using `bulk_create` that creates 20 orgs, one of them a noisy tenant with 30k reports, plus 200k expenses.
- **Query profiling.**
  - Run `EXPLAIN (ANALYZE, BUFFERS)` on the inbox, the dashboard, and the audit log.
  - Add or remove indexes, and record before and after timings in `docs/perf/`.
- **Keyset pagination.** Replace offset pagination on the audit log and the inbox. Measure page 1 against page 500.
- **Load testing.**
  - Write a Locust scenario mix: 70% reads, 20% submits, 10% approvals.
  - Find the first bottleneck. Expect a hot budget row, or connection exhaustion.
- **Connection sizing.**
  - Compute `web_tasks × gunicorn_workers + celery_concurrency + beat + admin` and compare it with `max_connections`.
  - Run an experiment with `CONN_MAX_AGE` set to 0 versus 60.
  - Learn when PgBouncer becomes necessary.
- **Regression guards.** Keep the query-count tests, `FETCH_RAISE` in tests, and a CI job that fails on new unapplied migrations or missing indexes on foreign keys you've marked as filtered.

## Phase 6: Deployment and operations (weeks 17–19), condensed

See [08-deployment-operations](08-deployment-operations.md). Deliverables:

- the production Dockerfile,
- the `prod` settings passing `check --deploy`,
- stage 1 deployed (EC2 + RDS + S3),
- the CD workflow,
- Sentry wired up,
- the health endpoints used by the load balancer,
- a written runbook covering deploy, roll back, run a migration, and rotate a secret.

## Phase 7: Advanced engineering (only when justified)

| Topic | When it is justified | Concept |
|---|---|---|
| Postgres row-level security as defence in depth | After tenant isolation is solid in the application | `SET app.current_org` per transaction, RLS policies, and the interaction with connection reuse |
| Outbound webhooks (`report.approved`) | When an "integration" persona appears | Transactional outbox table → relay task → HMAC signing → retries and a dead-letter queue |
| ECS Fargate and ALB (stage 2) | Once stage 1 is stable | Blue/green or rolling deploys, and migrations as a one-off task |
| API versioning | When a breaking change is needed | URL versioning plus a deprecation header |
| Audit hash chain | For the compliance story | Tamper evidence |
| Upgrade to Django 6.2 LTS (around April 2027) | When it is released | Read the release notes, fix deprecation warnings (run with `-W error::DeprecationWarning`), and use CI as the safety net |

---

## Engineering challenge catalogue (Part 12 of the brief)

Every challenge uses the same six steps: reproduce, investigate, tools, root cause, fix, and prevent regression.

| # | Challenge | Reproduce | Investigate / tools | Root cause | Fix | Prevent regression |
|---|---|---|---|---|---|---|
| C1 | Inbox has N+1 queries | Seed 50 steps and call the inbox | debug-toolbar, `CaptureQueriesContext`, `FETCH_RAISE` | Lazy FK access in the serializer | `select_related`, annotation | `assert_max_num_queries` plus `FETCH_RAISE` in tests |
| C2 | Two approvers decide the same step | Two threads, a barrier, `TransactionTestCase` | Postgres logs, the race test | Read-then-write without a guard | Conditional `UPDATE … WHERE state='PENDING'` | A race test running in a loop |
| C3 | Budget overspent | N concurrent final approvals | Ledger vs `consumed` drift | Lost update | `select_for_update` + `F()` + `CheckConstraint` | Race test plus nightly `reconcile_budget` |
| C4 | Deadlock | Opposite lock order across two code paths | Postgres `deadlock detected` log, `pg_locks` | Inconsistent lock ordering | Lock in sorted primary-key order | Code review rule plus a test |
| C5 | Transaction fails halfway | Patch the audit write to raise | Assert no partial rows | Missing `atomic()` boundary | Service-level `atomic()` | Rollback tests |
| C6 | Data migration breaks on a fresh database | Import real models in `RunPython` | Run migrations from zero in CI | Historical vs current model mismatch | `apps.get_model` | CI migrates from an empty database |
| C7 | Slow query as data grows | Seed 200k rows, time the dashboard | `EXPLAIN ANALYZE`, `pg_stat_statements` | Sequential scan, or a sort without an index | A composite or partial index, or rewriting the query | Perf notes and a benchmark test with a threshold |
| C8 | Cross-tenant access | Fetch org B's receipt ID while logged into org A | Cross-tenant test matrix | Unscoped `.objects.get()` | Scoped manager plus policy | A matrix test for every endpoint |
| C9 | Unauthorised operation | An employee calls `/approve/` | Permission matrix | A missing or incorrect permission class | Policy function | Matrix test |
| C10 | Request flood | Locust or a loop on the login endpoint | Logs, 429 counts | No throttling | DRF throttles (scoped login throttle) backed by Redis | A throttle test |
| C11 | Malicious file upload | A spoofed MIME type, or a polyglot file | The validation task's logs | Trusting the client's `Content-Type` | Presigned conditions plus a magic-byte check | Upload tests with fixture files |
| C12 | Token mishandling | Grep the logs for `Authorization` | Log review | Headers logged in plaintext | Scrub in the logging filter and in Sentry `before_send` | A log-scrubbing test |
| C13 | Task fails after it was queued | A worker raises in a non-retryable path | Celery events, Flower (optional) | Unhandled error | Classify errors as retryable or not, plus a dead-letter record | Task tests |
| C14 | Task runs twice | Kill the worker mid-task with `acks_late` | Provider mock logs | At-least-once delivery | An idempotency key plus a state guard | A duplicate-execution test |
| C15 | External service down | The mock provider returns 503 | Retry logs | — | Backoff, a circuit breaker (optional), reconciliation | A test using `responses` or `respx` |
| C16 | Notification before commit | `.delay()` inside `atomic()`, then a rollback | Worker logs show "not found" | Side effect before commit | `on_commit` | A `captureOnCommitCallbacks` test |
| C17 | Slow dashboard | 30k reports in the noisy tenant | Timing logs | Aggregation in Python | SQL aggregation plus cache | A query budget test |
| C18 | Stale cache | Approve, then read the dashboard | — | No invalidation | A versioned key bumped on commit | Test |
| C19 | Too many DB connections | Scale Gunicorn and Celery up | `pg_stat_activity` | Connections per process × number of processes | Sizing, `CONN_MAX_AGE`, PgBouncer later | A documented sizing formula |
| C20 | Large response memory | Export 200k rows | `tracemalloc` | `list(qs)` | `iterator()`, streaming, or an async export | A memory assertion test |
