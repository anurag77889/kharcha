# ADR-0005: Sessions + hashed personal API tokens; no JWT in the MVP

- **Status:** Accepted
- **Date:** 2026-11-26

## Context

Kharcha has two kinds of API client:

- **First-party**: the browsable API, and possibly a web UI later.
- **Integrations and scripts.**

The candidate mechanisms are Django sessions, DRF `authtoken`, JWT (simplejwt) and OAuth2.

## Decision

**First-party clients** use DRF `SessionAuthentication`, which enforces CSRF:

- Cookies are `Secure`, `HttpOnly` and `SameSite=Lax`.
- The session is rotated on login.

**Integrations** use a custom `BaseAuthentication`:

- Requests send `Authorization: Bearer kh_<prefix>_<secret>`.
- The token is looked up by its prefix. Only SHA-256(secret) is stored, and comparison uses `hmac.compare_digest`.
- Each token is scoped to one org, has scopes and an expiry, and can be revoked.
- `last_used_at` is updated at most once per minute.

## Consequences

**Benefits**

- Teaches Django's real authentication machinery: backends, `login()`, `cycle_key`, the session auth hash and CSRF.
- Tokens can be revoked instantly.
- A database leak does not expose usable tokens.

**Costs**

- Each token request needs one database lookup. This can be cached later if needed.

## Why not DRF authtoken

- It stores token keys in plaintext.
- It allows only one token per user.
- It has no scopes and no expiry.

## Why not JWT

- Revoking a JWT needs a denylist, which recreates server-side state anyway.
- Token size and key rotation add complexity.
- No current requirement needs stateless verification across services.

Revisit this if a separate service must verify tokens without calling Kharcha.
