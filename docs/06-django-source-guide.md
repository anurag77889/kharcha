# Part 10: Django Internals Investigation Plan (Django 6.1 source)

**How to use this guide.** Clone a read-only copy of the 6.1 source:

```
git clone --branch stable/6.1.x --depth 1 https://github.com/django/django
```

You can also read the source inside your virtualenv at `.venv/Lib/site-packages/django/`. Every file and symbol named below was checked against `stable/6.1.x` on 2026-11-26.

**Don't read whole files.** For each topic:

1. Start at the entry point.
2. Follow one execution path.
3. Set a breakpoint and step through it.
4. Run the experiment.
5. Write down what you saw in `notes/internals/<topic>.md`.

### Investigation techniques

| Technique | How |
|---|---|
| Breakpoints inside Django | `breakpoint()` in your code, then `s` (step) into Django. Or set an IDE breakpoint directly in the `site-packages` file. |
| Print the SQL for a QuerySet | `print(qs.query)`, or `str(qs.query)`. The parameters are interpolated for display only. |
| Capture executed SQL | `from django.test.utils import CaptureQueriesContext` → `with CaptureQueriesContext(connection) as ctx: ...` → `ctx.captured_queries` |
| See every query in dev | Set the `django.db.backends` logger to DEBUG (this only works with `DEBUG=True`) |
| Find who calls a function | `traceback.print_stack()` inside a receiver or method |
| Search the source | `rg "def select_for_update" django/` |
| Read the tests | `tests/<feature>/tests.py` in the Django repo. These are often the clearest specification. |

---

## 1. Application initialisation

**What it does.** It loads settings, configures logging, imports every installed app and its models, then calls each app's `ready()`.

**Why it exists.** Models need a complete registry before they can resolve relations to each other, and that includes string references such as `"organizations.Membership"`.

**Start reading at:**
- `django/core/management/__init__.py`: `execute_from_command_line`, `ManagementUtility.execute`
- `django/__init__.py`: `setup`
- `django/apps/registry.py`: `Apps.populate`
- `django/apps/config.py`: `AppConfig.create`, `import_models`
- `django/conf/__init__.py`: `LazySettings._setup`

**Path to follow.** `manage.py` → `execute_from_command_line` → `ManagementUtility.execute` → `django.setup()` → `configure_logging` → `apps.populate()`.

`populate()` runs in three phases:
- **Phase 1** creates the AppConfigs.
- **Phase 2** imports the models.
- **Phase 3** calls `ready()`.

**Experiment.** Add `print("models imported")` at the top of an app's `models.py` and `print("ready")` inside `ready()`. Run `manage.py check` and `manage.py runserver`. Why does `runserver` print everything twice? (The autoreloader runs a child process.)

## 2. Request handling and the WSGI entry point

**What it does.** It turns a WSGI `environ` into an `HttpRequest`, runs it through the middleware chain, and returns an `HttpResponse`.

**Start reading at:**
- `django/core/wsgi.py`: `get_wsgi_application`
- `django/core/handlers/wsgi.py`: `WSGIHandler.__call__`, `WSGIRequest`
- `django/core/handlers/base.py`: `BaseHandler.get_response`, `_get_response`, `resolve_request`, `make_view_atomic`

**Path to follow.**
1. `WSGIHandler.__call__` sends the `request_started` signal.
2. It builds a `WSGIRequest` and calls `get_response`.
3. That calls `_middleware_chain`. The outermost middleware is the first entry in your `MIDDLEWARE` setting.
4. The chain ends in `_get_response`, which resolves the URL, runs view middleware, calls the view, and runs template-response middleware.
5. The response unwinds back out through the chain.
6. `request_finished` fires when the WSGI server closes the response.

**Experiment.**
- Put a breakpoint in a view and print the stack (`traceback.print_stack()`). Count the middleware frames and match them to your `MIDDLEWARE` setting.
- Set `ATOMIC_REQUESTS=True` temporarily, then read `make_view_atomic` to see how it wraps your view.

## 3. Middleware

**Start reading at:**
- `django/core/handlers/base.py`: `load_middleware`
- `django/core/handlers/exception.py`: `convert_exception_to_response`, `response_for_exception`
- `django/utils/deprecation.py`: `MiddlewareMixin`

**Path to follow.** `load_middleware` iterates over `reversed(settings.MIDDLEWARE)`. At each step it builds `handler = middleware(handler)`, then wraps it as `convert_exception_to_response(handler)`. It also collects any `process_view`, `process_template_response` and `process_exception` hooks.

**Experiment.** Write a middleware that raises `MiddlewareNotUsed` in `__init__` and confirm it disappears from the chain. Then write one that returns a response early, without calling `get_response`, and note which inner middleware never runs.

## 4. URL resolution

**Start reading at:**
- `django/urls/resolvers.py`: `URLResolver.resolve`, `URLPattern.resolve`, `RoutePattern.match`
- `django/urls/converters.py`

**Path to follow.** `resolve_request` calls `get_resolver().resolve(path)`. That walks each `url_patterns` entry in order. An `include()` becomes a nested `URLResolver`, which matches its prefix and then recurses into the rest of the path. When a pattern matches, it returns a `ResolverMatch` holding the view function, the args and kwargs, and the route.

**Experiment.**
- In the shell, run `from django.urls import resolve; resolve("/api/v1/orgs/acme/reports/<uuid>/")` and inspect `.route`, `.kwargs` and `.func`.
- Pass a value that isn't a valid UUID and watch the resolver fall through to `Resolver404`.

## 5. View execution (Django and DRF)

**Start reading at:**
- `django/views/generic/base.py`: `View.as_view`, `View.dispatch`
- DRF `rest_framework/views.py`: `APIView.as_view`, `APIView.dispatch`, `APIView.initial`, `handle_exception`
- DRF `rest_framework/request.py`: `Request.user`, `_authenticate`

**Path to follow.** `as_view()` returns a closure. Each request creates a new instance of the view class, then `dispatch` looks up the handler method that matches the HTTP verb.

DRF adds three things:
- It wraps the request in its own `Request` object.
- `initial()` runs `perform_authentication`, `check_permissions` and `check_throttles`.
- `handle_exception()` passes errors through your `EXCEPTION_HANDLER`.

**Experiment.**
- Find where DRF calls `csrf_exempt` (in `APIView.as_view`) and where it re-applies CSRF (`SessionAuthentication.enforce_csrf`).
- Explain why token-authenticated requests skip CSRF.

## 6. ORM: model metadata

**Start reading at:**
- `django/db/models/base.py`: `ModelBase.__new__`
- `django/db/models/options.py`: `Options.contribute_to_class`, `get_fields`, `get_field`

**Path to follow.** The class body is processed by `ModelBase.__new__`. Each field's `contribute_to_class` attaches it to `_meta` and installs its descriptor. Relations to models that don't exist yet are resolved later, through `lazy_related_operation`.

**Experiment.** `[f.name for f in ExpenseReport._meta.get_fields()]`. Find which entries are reverse relations, and explain where they came from.

## 7. ORM: QuerySet evaluation and SQL generation

**Start reading at:**
- `django/db/models/query.py`: `QuerySet.__iter__`, `_fetch_all`, `_clone`, `filter`, `_filter_or_exclude`, `ModelIterable`, `fetch_mode`
- `django/db/models/sql/query.py`: `Query.add_q`, `build_filter`, `setup_joins`
- `django/db/models/sql/compiler.py`: `SQLCompiler.as_sql`, `execute_sql`, `results_iter`
- `django/db/models/expressions.py`: `F`, `Subquery`, `OuterRef`
- `django/db/models/query_utils.py`: `Q`

**Path to follow.**
1. `filter()` calls `_filter_or_exclude`, which calls `_clone`, which calls `query.add_q(Q(...))`.
2. `build_filter` turns each lookup into joins plus a `WhereNode`.
3. Nothing executes until iteration. Iteration calls `_fetch_all`, which calls `ModelIterable.__iter__`, which calls `compiler.execute_sql`.
4. Each returned row becomes a model instance via `Model.from_db`.

**Experiment.** Build a chain of 4 filters. Confirm that no queries have run yet. Then evaluate it and read the SQL. Change one filter so it spans a relation and see the JOIN appear in the SQL.

## 8. ORM: relations, prefetching and fetch modes

**Start reading at:**
- `django/db/models/fields/related_descriptors.py`: `ForwardManyToOneDescriptor.__get__`, `ReverseManyToOneDescriptor`, `create_reverse_many_to_one_manager`
- `django/db/models/query.py`: `prefetch_related_objects`, `RelatedPopulator`
- `django/db/models/fetch_modes.py`: `FETCH_ONE`, `FETCH_PEERS`, `FETCH_RAISE`

**Experiment.** Run the same inbox loop under each fetch mode and count the queries each time:

```python
qs.fetch_mode(FETCH_ONE)
qs.fetch_mode(FETCH_PEERS)
qs.fetch_mode(FETCH_RAISE)
```

## 9. Database connections and transactions

**Start reading at:**
- `django/db/utils.py`: `ConnectionHandler`
- `django/db/__init__.py`: `close_old_connections`, `reset_queries` (connected to the request signals)
- `django/db/backends/base/base.py`: `ensure_connection`, `set_autocommit`, `on_commit`, `run_and_clear_commit_hooks`, `validate_no_broken_transaction`, `close_if_unusable_or_obsolete`
- `django/db/transaction.py`: `Atomic.__enter__`, `Atomic.__exit__`, `on_commit`

**Path to follow.**
- **Connections.** Each thread gets its own connection through an `asgiref.local.Local` in `ConnectionHandler`. The connection is opened lazily on first use. At the start and end of every request, `close_old_connections` decides whether to keep it, based on `CONN_MAX_AGE` and its health.
- **Atomic blocks.**
  - The outermost `atomic` turns off autocommit, which begins a transaction.
  - A nested `atomic` creates a savepoint.
  - `on_commit` callbacks queue up and run only when the outermost commit happens.

**Experiment.**
- Turn on query logging and run nested `atomic()` blocks. You should see `SAVEPOINT`, `RELEASE SAVEPOINT` and `ROLLBACK TO SAVEPOINT` in the log.
- Register an `on_commit` callback inside a block that rolls back, and confirm it never runs.

## 10. Migrations

**Start reading at:**
- `django/db/migrations/autodetector.py`: `MigrationAutodetector.changes`
- `django/db/migrations/loader.py`: `MigrationLoader.build_graph`
- `django/db/migrations/executor.py`: `MigrationExecutor.migrate`, `apply_migration`
- `django/db/migrations/operations/special.py`: `RunPython`, `SeparateDatabaseAndState`
- `django/contrib/postgres/operations.py`: `AddIndexConcurrently`

**Experiment.**
- Run `manage.py makemigrations --dry-run -v 3`.
- Run `manage.py showmigrations --plan`.
- Run `manage.py sqlmigrate <app> <n>`.
- Inside a `RunPython` function, print `apps.get_model("expenses", "Expense")._meta.get_fields()` and compare it with your current model.

## 11. Authentication

**Start reading at:**
- `django/contrib/auth/__init__.py`: `authenticate`, `login`, `logout`, `get_user`, `update_session_auth_hash`
- `django/contrib/auth/backends.py`: `ModelBackend.authenticate`, `has_perm`, `_get_permissions`
- `django/contrib/auth/middleware.py`: `AuthenticationMiddleware` (sets a `SimpleLazyObject`)
- `django/contrib/auth/hashers.py`: `make_password`, `check_password`, `PBKDF2PasswordHasher`, `must_update`
- `django/contrib/auth/base_user.py`: `AbstractBaseUser.set_password`, `get_session_auth_hash`

**Path to follow.**
1. `authenticate()` tries each backend in `AUTHENTICATION_BACKENDS` in turn.
2. `login()` calls `request.session.cycle_key()`, then stores three keys in the session: the user's primary key, the dotted path of the backend that authenticated them, and the session auth hash.
3. On each later request, `get_user()` loads the user with that same backend and verifies the hash. This is why changing a password logs out other sessions.

**Experiment.**
- Log in, then change the password in the shell, then refresh the page. You are logged out.
- Now call `update_session_auth_hash` after changing the password and compare what happens.

## 12. Sessions

**Start reading at:**
- `django/contrib/sessions/middleware.py`: `SessionMiddleware.process_request`, `process_response`
- `django/contrib/sessions/backends/base.py`: `SessionBase._get_session`, `cycle_key`, `modified`, `accessed`
- `django/contrib/sessions/backends/db.py`

**Experiment.**
- Read a session in a view without modifying it, and check whether a `Set-Cookie` header comes back.
- Then modify a nested dict inside the session and check whether the change is saved. Why not? You need to set `session.modified = True`.

## 13. Signals

**Start reading at:**
- `django/dispatch/dispatcher.py`: `Signal.connect`, `send`, `send_robust`
- `django/db/models/base.py`: `save_base` (where `pre_save` and `post_save` are sent)
- `django/core/signals.py`

**Experiment.**
- Connect a `post_save` receiver that raises an exception, then call `save()` inside `atomic()`. The receiver runs inside your transaction and rolls it back.
- Then call `QuerySet.update()` and confirm the receiver never fires.

## 14. Forms and validation

**Start reading at:**
- `django/forms/forms.py`: `BaseForm.is_valid`, `errors`, `full_clean`, `_clean_fields`, `_clean_form`, `_post_clean`
- `django/forms/models.py`: `BaseModelForm._post_clean` (calls `instance.full_clean` and `validate_unique`)
- `django/db/models/base.py`: `Model.full_clean`, `validate_constraints`
- DRF `rest_framework/serializers.py`: `Serializer.run_validation`, `to_internal_value`, `validate`

**Path to follow.**
1. `is_valid()` reads `errors`, which triggers `full_clean()`.
2. `full_clean()` runs `_clean_fields()`, which calls each field's `clean()` and then your `clean_<field>()` method.
3. Next comes `_clean_form()`, which calls your form's `clean()`.
4. Finally `_post_clean()` runs. For a ModelForm, that is where model validation happens.

**Experiment.** Compare which validation runs in each of these, using one invalid expense:
- a ModelForm,
- a DRF ModelSerializer,
- a direct `Model.objects.create()`.

## 15. Template rendering

**Start reading at:**
- `django/template/engine.py`: `Engine.get_template`
- `django/template/base.py`: `Lexer`, `Parser`, `Template.render`, `Node.render`
- `django/template/context.py`: `RequestContext`, `make_context`

**Path to follow.** Template text is split into tokens by the `Lexer`. The `Parser` turns the tokens into a `NodeList`, and that tree is compiled once and cached when the cached loader is used. On render, the tree walks a stacked `Context`, which the context processors have filled in.

**Experiment.** Render the approval-notification email template with `{{ report.submitter.user.email }}` under `CaptureQueriesContext`. Count the queries that template access causes.

## 16. Django 6 Tasks framework (bonus)

**Start reading at:** `django/tasks/base.py` (`task`, `Task.enqueue`), `django/tasks/backends/immediate.py` and `dummy.py`.

**Experiment.** Define one task with `@task`, enqueue it under `ImmediateBackend`, and compare its lifecycle with a Celery task.

## Reading order (paired with the roadmap)

| Roadmap phase | Topics to read |
|---|---|
| Phase 1 | 1 → 2 → 3 → 4 → 5 |
| Phase 2 | 6 → 7 → 8 → 9 → 10 |
| Phase 3 | 11 → 12 |
| Phase 4 | 13 → 14 → 15 → 16 |
