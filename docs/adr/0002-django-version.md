# ADR-0002: Target Django 6.1, upgrade to 6.2 LTS

- **Status:** Accepted
- **Date:** 2026-11-26

## Context

Release status as of 2026-11-26:

| Version | Status |
|---|---|
| 6.1 | Current feature release (6.1.3 on PyPI). Supports Python 3.12–3.14. |
| 6.0 | Past mainstream support. Security fixes until April 2027. |
| 5.2 | The LTS. |
| 6.2 | Next LTS, due April 2027. |

Django has also announced a move to an annual release cycle.

Third-party support:
- DRF 3.18.3 declares support for Django 6.1.
- drf-spectacular 0.30.0 declares support only up to 6.0, so check it at install time.

Sources: [download page](https://www.djangoproject.com/download/), [6.1 release](https://djangoproject.com/weblog/2026/aug/05/django-61-released), [annual cycle](https://djangoproject.com/weblog/2026/aug/10/annual-release-cycle).

## Decision

- Use **Django 6.1.x** on Python 3.13.
- Upgrade to **6.2 LTS** shortly after it is released. Treat the upgrade as a planned learning exercise:
  - read the release notes,
  - run with deprecation warnings turned into errors,
  - rely on CI as the safety net.

## Consequences

**What we gain**

These 6.x features are used for teaching:
- CSP middleware
- `django.tasks`
- 6.1 fetch modes, including `FETCH_RAISE` for catching N+1 queries
- database-level `on_delete`

The source-code guide also matches current documentation.

**What it costs**

- A shorter support window than 5.2 LTS, which forces the 6.2 upgrade.
- Some third-party packages may lag behind. Check each one's declared support before adding it.
