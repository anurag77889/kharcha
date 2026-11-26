# Parts 4–6: Project Ideas, Scoring, and the Top Three

## Part 4: Seventeen industry-level project ideas

Each idea is profiled on the 18 attributes from the brief. To keep it scannable, each attribute gets one line.

**Difficulty levels**
- **I**: Intermediate
- **A**: Advanced
- **E**: Expert

**Effort** assumes one developer working about 15 hours a week, to an MVP-plus-hardening level.

---

### P1. Multi-tenant Spend & Approval Platform ("Kharcha")
- **Problem:** Companies lose money and time on manual expense claims. That means emailed receipts, untracked approvals, and budgets blown silently.
- **Users:** Employees, managers/approvers, finance admins, and the vendor's internal ops/support team.
- **Industry:** Spend-management SaaS (Expensify, Ramp, Brex, Zoho Expense, SAP Concur). Internal finance tooling at most mid-size companies.
- **Business value:** Policy compliance, an audit trail, budget control, and faster reimbursement.
- **Core features:**
  - Orgs and departments
  - Expense reports with line items and receipts
  - Policy checks
  - Multi-level approval
  - Budgets per department and period
  - Reimbursement batches
  - Audit log
  - Dashboard
  - CSV export
- **Django relevance:** Relational, role-heavy, workflow-heavy, and ops-admin-heavy. This is Django's sweet spot.
- **Internals:** Every area A–K in the learning map.
- **Database:** About 15 entities. Partial unique constraints, check constraints, composite indexes, and an append-only ledger.
- **Backend challenges:**
  - A state machine under concurrency
  - Budget locking
  - Idempotent submission
  - `on_commit` side effects
  - Payout retries
- **Infrastructure:** PostgreSQL, Redis, Celery, S3, and SMTP or a mock email service.
- **Security:**
  - Tenant isolation
  - Object-level authorization
  - Separation of duties (you cannot approve your own report)
  - Upload safety
  - Signed URLs
  - Token hashing
- **Testing:**
  - A permission matrix
  - Race-condition tests
  - Transaction and `on_commit` tests
  - Task idempotency tests
  - Query-count budgets
- **Scalability:** Inbox and dashboard queries grow with tenant size, there are noisy-neighbour tenants, and the payout batch volume grows.
- **Difficulty:** A
- **Portfolio value:** High. Interviewers immediately understand the domain and can probe its hard parts.
- **Learning value:** Very high, and spread across almost all of Django.
- **Effort:** 14–18 weeks for the MVP plus hardening. Phase 7 is optional on top.

### P2. Multi-vendor Marketplace OMS + Vendor Settlement
- **Problem:** Marketplaces must split orders across vendors, reserve stock, take commission, and pay vendors out correctly.
- **Users:** Buyers, vendors, and marketplace operations staff.
- **Industry:** Flipkart, Amazon, and Etsy sellers; Shopify-app builders.
- **Business value:** Revenue correctness and vendor trust.
- **Core features:**
  - Catalogue and SKUs
  - Cart and checkout
  - Order split per vendor
  - Stock reservation
  - Payment webhook (mock)
  - Commission rules
  - Settlement ledger
  - Payouts
  - Returns
- **Django relevance:** Strong. Transactional and relational, with an admin back-office.
- **Internals:** ORM, transactions, locking, signals vs services, Celery, caching (catalogue).
- **Database:** About 25 entities. Inventory and ledger invariants.
- **Backend challenges:**
  - Overselling under concurrency
  - Payment webhook idempotency
  - Partial refunds
  - Settlement reconciliation
- **Infrastructure:** PostgreSQL, Redis, Celery, S3, and a payment sandbox.
- **Security:** Vendor isolation, payment-webhook signature verification, and price tampering.
- **Testing:** Checkout races, refund paths, settlement totals.
- **Scalability:** Flash-sale hot rows and catalogue read load.
- **Difficulty:** A–E
- **Portfolio value:** High.
- **Learning value:** High, but you spend a lot of time on commerce edge cases that teach little Django.
- **Effort:** 24+ weeks. The scope tends to explode (catalogue UI, search, returns).

### P3. Double-entry Ledger + Payment Reconciliation
- **Problem:** Internal money movements must balance, and must be matched against bank and PSP statements.
- **Users:** Finance and treasury operations.
- **Industry:** Fintechs, PSPs, and marketplaces (Stripe, Razorpay, and Modern Treasury-style tooling).
- **Business value:** Financial correctness and audit readiness.
- **Core features:**
  - Accounts
  - Journal entries (with debits equal to credits)
  - Immutable postings
  - Statement import
  - Matching rules
  - Exceptions queue
  - Balances as of a date
- **Django relevance:** Good. Postgres and the ORM fit well. The UI is mostly admin.
- **Internals:** Transactions, constraints, bulk import, aggregation, data migrations.
- **Database:** About 10 entities, but with deep invariants: balanced entries, immutability, and idempotent imports.
- **Backend challenges:** Exactly-once import, the matching algorithm, and concurrency on balances.
- **Infrastructure:** PostgreSQL, Celery (imports), S3 (statement files).
- **Security:** Segregation of duties and immutability.
- **Testing:** Property-based tests that check balances always sum to zero, and import idempotency.
- **Scalability:** Ledger growth, balance queries (snapshots), and partitioning.
- **Difficulty:** A
- **Portfolio value:** High for fintech roles.
- **Learning value:** Very high for databases. Weaker on auth, middleware, forms, and files.
- **Effort:** 10–14 weeks.

### P4. Multi-warehouse Inventory & Reservation System
- **Problem:** Stock spread across warehouses must be reserved, allocated, and replenished without being oversold.
- **Users:** Ops staff, warehouse staff, and integrating storefronts.
- **Industry:** D2C brands, 3PLs, and quick-commerce companies.
- **Business value:** Avoiding oversell and stockouts.
- **Core features:**
  - SKUs and warehouses
  - Stock levels
  - Reservations with TTL
  - Allocation rules
  - Transfers
  - Cycle counts
  - Low-stock alerts
- **Django relevance:** Strong.
- **Internals:** `select_for_update` / `skip_locked`, `F()` expressions, constraints, Celery beat (reservation expiry).
- **Database:** About 10 entities. Quantity invariants.
- **Backend challenges:** Hot-row contention and reservation expiry races.
- **Infrastructure:** PostgreSQL, Redis, Celery.
- **Security:** Moderate (API tokens for storefronts).
- **Testing:** Concurrency tests and expiry tests.
- **Scalability:** Hot SKUs and sharded counters.
- **Difficulty:** A
- **Portfolio value:** Medium–high.
- **Learning value:** High for concurrency, low for auth and security.
- **Effort:** 8–12 weeks.

### P5. Subscription Billing Engine
- **Problem:** SaaS companies need plans, trials, proration, invoices, dunning, and usage add-ons.
- **Users:** SaaS finance teams and developers integrating via API.
- **Industry:** Every SaaS company (Chargebee, Recurly, Stripe Billing).
- **Business value:** Revenue.
- **Core features:**
  - Plans and prices
  - Subscriptions and their lifecycle
  - Proration
  - Invoices
  - Payment retries
  - Credit notes
  - Webhooks
- **Django relevance:** Good.
- **Internals:** Transactions, Celery beat, data migrations (price changes).
- **Database:** About 15 entities. Temporal data.
- **Backend challenges:** Time and calendar edge cases, proration maths, and dunning state.
- **Infrastructure:** PostgreSQL, Celery, a payment sandbox.
- **Security:** Webhook signatures and PCI scope avoidance.
- **Testing:** Time-travel tests (freezegun/time-machine).
- **Scalability:** End-of-month invoice spikes.
- **Difficulty:** A–E
- **Portfolio value:** High.
- **Learning value:** Medium. Calendar maths dominates over Django.
- **Effort:** 16–20 weeks.

### P6. Resource Scheduling & Booking (no double-booking)
- **Problem:** Clinics, labs, and meeting rooms need conflict-free bookings with rules.
- **Users:** Staff, customers, and admins.
- **Industry:** Healthcare scheduling, coworking spaces, and service businesses.
- **Business value:** Utilisation and fewer no-shows.
- **Core features:**
  - Resources
  - Availability rules
  - Bookings
  - Holds with TTL
  - Cancellation policies
  - Reminders
- **Django relevance:** Strong.
- **Internals:** Postgres `ExclusionConstraint` with range fields (`django.contrib.postgres`), transactions, and Celery beat.
- **Database:** About 8 entities. Range types and exclusion constraints.
- **Backend challenges:** Double booking, time zones, and recurring rules.
- **Infrastructure:** PostgreSQL, Celery, email/SMS mocks.
- **Security:** Moderate.
- **Testing:** Conflict tests and time-zone tests.
- **Scalability:** Availability search.
- **Difficulty:** I–A
- **Portfolio value:** Medium–high.
- **Learning value:** High, but narrow.
- **Effort:** 8–10 weeks.

### P7. API Key & Usage Metering Platform
- **Problem:** API businesses must issue keys, enforce quotas, meter usage, and expose usage to customers.
- **Users:** Developers and the API provider's operations team.
- **Industry:** API-first companies (Twilio, OpenAI-style platforms).
- **Business value:** Monetisation and abuse prevention.
- **Core features:**
  - Projects
  - Keys (hashed)
  - Scopes
  - Rate limits
  - Usage events
  - Aggregation
  - Quotas
  - Overage alerts
- **Django relevance:** Good for the control plane. The hot path may outgrow Django.
- **Internals:** Middleware, auth backends, caching, and Redis counters.
- **Database:** About 8 entities, plus a high-volume events table.
- **Backend challenges:** Counter accuracy versus speed, and rolling up aggregates.
- **Infrastructure:** PostgreSQL, Redis, Celery.
- **Security:** Strong (key handling).
- **Testing:** Rate-limit tests and aggregation tests.
- **Scalability:** Event ingestion volume.
- **Difficulty:** A
- **Portfolio value:** Medium–high.
- **Learning value:** Medium–high. The domain model is thin.
- **Effort:** 8–10 weeks.

### P8. Multi-tenant Helpdesk with SLAs
- **Problem:** Support teams need ticket routing, SLA timers, escalation, and email ingestion.
- **Users:** Agents, customers, and admins.
- **Industry:** Zendesk/Freshdesk-type SaaS and internal IT.
- **Business value:** Customer retention.
- **Core features:**
  - Tickets
  - Queues
  - Assignment
  - SLA policies
  - Escalations
  - Inbound email
  - Canned responses
- **Django relevance:** Strong.
- **Internals:** Tenancy, permissions, Celery beat (SLA breaches), and templates.
- **Database:** About 12 entities.
- **Backend challenges:** SLA computation across business hours, and assignment races.
- **Infrastructure:** PostgreSQL, Redis, Celery, inbound email.
- **Security:** Tenant isolation and HTML email sanitisation.
- **Testing:** SLA clock tests.
- **Scalability:** Ticket search.
- **Difficulty:** I–A
- **Portfolio value:** Medium (common in portfolios).
- **Learning value:** Medium–high.
- **Effort:** 10–12 weeks.

### P9. Configurable Workflow Automation Engine
- **Problem:** Businesses want no-code approval and automation flows.
- **Users:** Ops admins.
- **Industry:** Zapier/n8n-like tools and enterprise BPM.
- **Business value:** Automation.
- **Core features:**
  - Workflow definitions (a DAG)
  - Triggers
  - Steps
  - Runs
  - Retries
  - Versioning of definitions
- **Django relevance:** Reasonable.
- **Internals:** JSONField, Celery chains, transactions.
- **Database:** Generic or EAV-ish models (a hazard).
- **Backend challenges:** Building an interpreter, versioning, and step idempotency.
- **Infrastructure:** PostgreSQL, Redis, Celery.
- **Security:** Sandboxing user-defined logic.
- **Testing:** Hard (combinatorial).
- **Scalability:** Run volume.
- **Difficulty:** E
- **Portfolio value:** Medium–high if finished.
- **Learning value:** High, but it is a rabbit hole.
- **Effort:** 20+ weeks.

### P10. Webhook Delivery & Retry Platform
- **Problem:** SaaS products must deliver events to customer endpoints reliably.
- **Users:** Developers.
- **Industry:** Svix/Hookdeck-type products, and every platform that has webhooks.
- **Business value:** Integration reliability.
- **Core features:**
  - Endpoints
  - Event types
  - Signing
  - Delivery attempts
  - Exponential backoff
  - Dead-letter queue
  - Replay
  - Delivery logs
- **Django relevance:** Reasonable. Most of the work happens in workers.
- **Internals:** Celery, the outbox pattern, `on_commit`, and bulk writes.
- **Database:** About 6 entities, plus a large attempts table.
- **Backend challenges:** At-least-once delivery, ordering, and per-endpoint rate limits.
- **Infrastructure:** PostgreSQL, Redis, Celery.
- **Security:** HMAC signing and SSRF protection.
- **Testing:** Retry tests and replay tests.
- **Scalability:** Fan-out.
- **Difficulty:** A
- **Portfolio value:** Medium–high.
- **Learning value:** High for async work, low for Django breadth.
- **Effort:** 6–8 weeks. Good as a *feature* of P1 (Phase 7).

### P11. Clinic Administration System (EHR-lite)
- **Problem:** Small clinics need patients, appointments, notes, billing, and consent tracking.
- **Users:** Doctors, front desk, and admins.
- **Industry:** Healthtech (Practo-type products).
- **Business value:** Operational efficiency.
- **Core features:**
  - Patients
  - Encounters
  - Prescriptions
  - Appointments
  - Invoices
  - Access logs
- **Django relevance:** Strong.
- **Internals:** Permissions, audit, encryption at rest (field-level), and forms.
- **Database:** About 15 entities.
- **Backend challenges:** Mostly compliance rather than engineering.
- **Infrastructure:** PostgreSQL, S3.
- **Security:** Very high (PHI). The burden is regulatory.
- **Testing:** Access-control tests.
- **Scalability:** Low.
- **Difficulty:** A
- **Portfolio value:** Medium.
- **Learning value:** Medium.
- **Effort:** 12–16 weeks.

### P12. Audit Log & Event Management Service
- **Problem:** Enterprises need tamper-evident activity logs that they can search and export.
- **Users:** Security and compliance teams.
- **Industry:** Enterprise SaaS (as a feature).
- **Business value:** Compliance (SOC 2).
- **Core features:**
  - Ingest API
  - Append-only store
  - Hash chain
  - Search
  - Retention
  - Export
- **Django relevance:** Reasonable.
- **Internals:** Bulk inserts, partitioning, middleware context.
- **Database:** Few entities, high volume.
- **Backend challenges:** Write throughput and search.
- **Infrastructure:** PostgreSQL (maybe OpenSearch).
- **Security:** Integrity.
- **Testing:** Tamper-detection tests.
- **Scalability:** Volume.
- **Difficulty:** I–A
- **Portfolio value:** Medium.
- **Learning value:** Medium. Better as P1's audit feature.
- **Effort:** 5–6 weeks.

### P13. Shipment Tracking System
- **Problem:** Merchants want unified tracking across couriers.
- **Users:** Merchants and end customers.
- **Industry:** Logistics aggregators (Shiprocket-type companies).
- **Business value:** Fewer "where is my order?" support tickets.
- **Core features:**
  - Shipments
  - Courier adapters
  - Status polling and webhooks
  - Notifications
- **Django relevance:** Reasonable.
- **Internals:** Celery beat and adapters.
- **Database:** About 6 entities.
- **Backend challenges:** Normalising third-party statuses.
- **Infrastructure:** PostgreSQL, Celery.
- **Security:** Low–moderate.
- **Testing:** Adapter mocks.
- **Scalability:** Polling volume.
- **Difficulty:** I
- **Portfolio value:** Low–medium.
- **Learning value:** Medium–low.
- **Effort:** 5–6 weeks.

### P14. Analytics & Reporting Platform
- **Problem:** Business users want self-serve reports.
- **Users:** Analysts and managers.
- **Industry:** BI tools.
- **Business value:** Insight.
- **Core features:**
  - Datasets
  - Saved queries
  - Scheduled reports
  - Exports
- **Django relevance:** Weak–reasonable. Heavy aggregation belongs in a warehouse.
- **Internals:** Aggregation, raw SQL, caching.
- **Database:** Moderate.
- **Backend challenges:** Query performance and multi-tenant SQL safety.
- **Infrastructure:** PostgreSQL, Redis, Celery.
- **Security:** SQL injection via user-defined queries.
- **Testing:** Report correctness.
- **Scalability:** OLAP load on an OLTP database.
- **Difficulty:** A
- **Portfolio value:** Medium.
- **Learning value:** Medium.
- **Effort:** 10 weeks.

### P15. CRM
- **Problem:** Sales teams track leads, deals, and activities.
- **Users:** Sales reps and managers.
- **Industry:** Salesforce/HubSpot/Zoho-type products.
- **Business value:** Pipeline visibility.
- **Core features:**
  - Contacts
  - Companies
  - Deals
  - Pipelines
  - Activities
  - Import
- **Django relevance:** Strong but shallow.
- **Internals:** CRUD, permissions, import.
- **Database:** About 10 entities.
- **Backend challenges:** Few hard problems beyond deduplication.
- **Infrastructure:** PostgreSQL.
- **Security:** Moderate.
- **Testing:** Standard.
- **Scalability:** Low.
- **Difficulty:** I
- **Portfolio value:** Low–medium.
- **Learning value:** Medium–low.
- **Effort:** 8 weeks.

### P16. Document Processing Platform
- **Problem:** Extract data from invoices and IDs.
- **Users:** Ops teams.
- **Industry:** IDP vendors and fintech KYC.
- **Business value:** Fewer manual data-entry hours.
- **Core features:**
  - Upload
  - OCR pipeline
  - Extraction
  - Review queue
- **Django relevance:** Reasonable for the control plane. The pipeline is ML-heavy.
- **Internals:** File handling and Celery.
- **Database:** About 6 entities.
- **Backend challenges:** Pipeline orchestration.
- **Infrastructure:** S3, Celery, an OCR service.
- **Security:** PII handling.
- **Testing:** Pipeline tests.
- **Scalability:** Processing throughput.
- **Difficulty:** A
- **Portfolio value:** Medium.
- **Learning value:** Medium-low for Django.
- **Effort:** 8–10 weeks.

### P17. Data Quality Monitoring Platform
- **Problem:** Detect bad data in pipelines.
- **Users:** Data engineers.
- **Industry:** Data platform teams.
- **Business value:** Trust in data.
- **Core features:**
  - Checks
  - Schedules
  - Results
  - Alerts
- **Django relevance:** Weak–reasonable.
- **Internals:** Celery beat.
- **Database:** About 6 entities.
- **Backend challenges:** Connector sprawl.
- **Infrastructure:** PostgreSQL, Celery.
- **Security:** Credential storage.
- **Testing:** Check-engine tests.
- **Scalability:** Number of checks.
- **Difficulty:** I–A
- **Portfolio value:** **Low for you**. It overlaps your existing FastAPI ML-monitoring project.
- **Learning value:** Low–medium.
- **Effort:** 6–8 weeks.

---

## Part 5: Scoring and ranking

### Framework

Each dimension is scored from 1 to 10. Weights sum to 100.

**Weighted score = Σ(weight × score) / 100**, which gives a result out of 10.

| # | Dimension | Weight | Rationale for the weight |
|---|---|---|---|
| D1 | Django internals coverage | **15** | Your primary objective (Objective A) |
| D2 | Backend engineering depth | 10 | Objective B |
| D3 | Database & ORM complexity | **12** | Most production Django pain is database pain |
| D4 | System design exposure | 6 | Useful, but should not drive the choice toward over-architecture |
| D5 | Security engineering | 8 | Required by the brief for every project |
| D6 | Asynchronous processing | 6 | Use only where justified |
| D7 | Performance optimisation | 7 | |
| D8 | Production architecture relevance | 8 | |
| D9 | Testing & reliability | 7 | |
| D10 | Portfolio & interview value | 8 | |
| D11 | Feasibility for one developer | **8** | Penalises endless projects |
| D12 | Long-term learning value | 5 | |

### Score matrix

| Project | D1 | D2 | D3 | D4 | D5 | D6 | D7 | D8 | D9 | D10 | D11 | D12 | **Weighted** |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| P1 Spend & Approval | 9 | 9 | 9 | 7 | 9 | 7 | 8 | 9 | 9 | 9 | 8 | 9 | **8.61** |
| P2 Marketplace OMS | 8 | 9 | 9 | 8 | 8 | 8 | 8 | 9 | 8 | 9 | 5 | 9 | **8.19** |
| P3 Ledger & Reconciliation | 6 | 9 | 10 | 7 | 8 | 7 | 7 | 9 | 9 | 8 | 7 | 8 | **7.92** |
| P4 Inventory & Reservation | 8 | 8 | 9 | 7 | 6 | 7 | 8 | 8 | 8 | 8 | 7 | 8 | **7.76** |
| P5 Subscription Billing | 7 | 9 | 9 | 7 | 7 | 8 | 6 | 9 | 9 | 8 | 5 | 8 | **7.70** |
| P6 Scheduling & Booking | 8 | 8 | 9 | 6 | 7 | 6 | 7 | 8 | 8 | 7 | 8 | 7 | **7.60** |
| P7 API Key & Metering | 6 | 8 | 7 | 8 | 9 | 7 | 9 | 8 | 7 | 8 | 7 | 8 | **7.52** |
| P8 Helpdesk with SLAs | 8 | 7 | 8 | 6 | 8 | 7 | 7 | 8 | 7 | 7 | 8 | 7 | **7.45** |
| P9 Workflow Engine | 8 | 9 | 8 | 8 | 7 | 8 | 6 | 7 | 7 | 8 | 4 | 8 | **7.41** |
| P10 Webhook Delivery | 5 | 8 | 6 | 8 | 7 | 10 | 7 | 9 | 8 | 8 | 8 | 8 | **7.36** |
| P11 Clinic Admin | 8 | 7 | 8 | 6 | 9 | 6 | 6 | 7 | 7 | 7 | 6 | 7 | **7.16** |
| P12 Audit Log Service | 7 | 6 | 7 | 6 | 7 | 6 | 7 | 7 | 6 | 6 | 8 | 6 | **6.66** |
| P13 Shipment Tracking | 6 | 6 | 7 | 6 | 6 | 7 | 7 | 7 | 6 | 6 | 8 | 6 | **6.49** |
| P14 Analytics & Reporting | 6 | 6 | 8 | 6 | 5 | 6 | 9 | 6 | 6 | 6 | 7 | 6 | **6.45** |
| P15 CRM | 7 | 6 | 7 | 5 | 7 | 5 | 6 | 7 | 6 | 5 | 7 | 6 | **6.31** |
| P16 Document Processing | 6 | 6 | 5 | 6 | 7 | 8 | 6 | 7 | 6 | 6 | 7 | 6 | **6.24** |
| P17 Data Quality Monitoring | 5 | 6 | 6 | 6 | 5 | 7 | 7 | 6 | 6 | 4 | 7 | 5 | **5.77** |

### Why the top projects outperform the others

- **P1 has no weak dimension.** Its lowest score is 7. Every other project has at least one serious gap:
  - P3 is thin on Django internals.
  - P4 is thin on security.
  - P10 is thin on Django internals.
  - P2, P5, and P9 fail on feasibility.
- **Domain complexity is not the same as Django learning.** P5 (billing) and P9 (workflow engine) are hard. But a lot of that difficulty is calendar maths or interpreter design, which teaches you nothing about Django.
- **Size was penalised.** P2 would teach about as much as P1, but it carries roughly double the scope. Its catalogue, checkout, and returns work is mostly CRUD.
- **Overlap was penalised.** P17 repeats work you have already done in FastAPI.

---

## Part 6: Top three

### 1. Multi-tenant Spend & Approval Platform (P1), score 8.61

**Why it is valuable.** It solves real problems that companies pay for: policy enforcement, approval routing, budget control, and auditability. The hard parts are what make it more than a CRUD app:

- money in a state machine
- several actors with different powers
- concurrency on shared budgets
- untrusted file uploads

These problems are inherent to the domain, not bolted on.

**Feature → Django concept → what you will understand**

| Feature | Django concept | What you will understand |
|---|---|---|
| Email-based login | Custom User model, `AbstractBaseUser`, auth backends | How `authenticate()` and `login()` and sessions fit together |
| Org isolation | Custom QuerySet/Manager, middleware, `contextvars` | Data-access boundaries that are hard to forget |
| Approve a report | `atomic()`, conditional `.update()`, `select_for_update()` | Atomicity, lost updates, and lock scope |
| Budget consumption | `F()` expressions, `CheckConstraint`, row locks | Database-enforced invariants |
| Approver can't approve own report | DRF object permissions, a policy module | Authorisation as a single source of truth |
| Notifications | Celery plus `transaction.on_commit` | Why "send after commit" matters |
| Receipts | Storage API, presigned URLs, upload validation | Files are untrusted input, and authorisation must happen before access |
| Spend dashboard | `annotate`, `Subquery`, `Coalesce`, indexes, cache | SQL-side aggregation and cache invalidation |
| Approvals inbox | `select_related`, `Prefetch`, fetch modes, keyset pagination | N+1 queries and pagination cost |
| Policy change | Data migration with `RunPython` | Evolving production data safely |
| Audit trail | Middleware context, services, the signal contrast | Request context and where events belong |
| Ops back-office | Admin customisation, actions, `get_queryset` | The admin as a production tool |

**Production problems it introduces:**
- Double-click submit (duplicate requests)
- Two approvers clicking at once (a race)
- A budget overspent under concurrent load
- A notification sent for a rolled-back approval
- A payout task retried after a timeout (risk of double payment)
- Cross-tenant ID guessing
- A spoofed MIME type on upload
- A dashboard that gets slow at 50k reports
- A stale cached dashboard after an approval

**What you will be able to explain in interviews:**
- "Why I used a conditional UPDATE for state transitions, but `select_for_update` for budgets."
- "How my tenant QuerySet makes forgetting the org filter a test failure, and where it *can't* protect me (raw SQL, `.objects` on unscoped managers)."
- "Why my notification test uses `captureOnCommitCallbacks`, and what bug it caught."
- "How I sized database connections across Gunicorn and Celery."

### 2. Multi-vendor Marketplace OMS + Settlement (P2), score 8.19

**Why it is valuable.** It covers the core commerce problems: overselling, payment webhooks, and commission and settlement correctness.

**Concepts:**

| Feature | Django concept |
|---|---|
| Stock reservation | `select_for_update` and `F()` |
| Order split | Transactions across aggregates |
| Payment webhook | Idempotency keys and signature verification middleware |
| Settlement | Aggregation and ledger constraints |
| Catalogue | Caching and `prefetch_related` |

**Production problems:**
- Flash-sale contention
- Duplicated webhook deliveries
- Partial refunds breaking settlement totals

**Interview value.** It is excellent for e-commerce companies. **The risk** is that one developer spends months on catalogue and checkout CRUD before reaching the interesting parts.

### 3. Double-entry Ledger + Reconciliation (P3), score 7.92

**Why it is valuable.** It is the most rigorous database project on the list: immutability, balanced postings, idempotent imports, and as-of-date balances.

**Concepts:**

| Feature | Django concept |
|---|---|
| Balanced postings | `CheckConstraint` plus transactional services |
| Statement import | `bulk_create`, `iterator`, unique constraints |
| As-of-date balances | Window functions and `Subquery` |
| Corrections | Data migrations and reversal entries |

**Production problems:**
- A duplicate import file
- Floating-point money errors
- Locking a balance during concurrent posting

**Interview value.** It is excellent for fintech companies. It is **weak** on auth, middleware, forms, files, and admin, which means it covers too little of Objective A.

## Decision

**P1 is the flagship.** Its strongest ideas from P3 (an append-only budget ledger) and P10 (outbound webhooks, in Phase 7) are folded into P1 as *features*. They are not separate products. This keeps the scope realistic and keeps the learning breadth.
