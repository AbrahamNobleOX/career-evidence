# Lead Reporting Platform — Career Evidence Dossier

**Prepared:** 12 October 2026
**Subject:** Abraham Akintola (`AbrahamNobleOX` / `Abraham`)
**Organisation:** Insurance Revolution
**Repositories:** `Insurance-Revolution/lead-reporting` (backend), `Insurance-Revolution/lead-reporting-frontend` (frontend)
**Audit method:** Repository inspection, full Git history review, and read-only aggregate queries against the production PostgreSQL database (session forced to `default_transaction_read_only = on`).

> **Evidence levels used throughout:** VERIFIED (directly observed), DERIVED (calculated from verified data), STRONGLY SUPPORTED (multiple indicators), INFERRED (likely, not proven), UNKNOWN (requires confirmation).

---

## 1. Executive Summary

**Lead Reporting** is an internal business-intelligence web application built for Insurance Revolution, an insurance brokerage. It gives internal staff a filterable, real-time view of inbound insurance enquiry ("lead") data that was previously locked inside a production database shared with an external telephony/dialer system.

The platform is live in production at `bi.insurancerevolution.co.uk`, backed by an API at `bi-backend.insurancerevolution.co.uk`, both deployed on Microsoft Azure via GitHub Actions.

**Most significant verified facts:**

| Finding | Value | Evidence level |
|---|---|---|
| Lead rows queryable in production | **642,348** (1,213 MB table) | VERIFIED |
| Leads created in 12 months to Sep 2026 | **150,030** | DERIVED |
| Average / peak daily lead volume (last 90 days) | **332 / day**, peak **572** | VERIFIED |
| Distinct lead sources & types surfaced (90 days) | **23 sources, 40 lead types** | VERIFIED |
| Near-duplicate rows collapsed by canonical-session logic | **21.3%** (10,587 → 8,327 over 30 days) | VERIFIED |
| Internal users onboarded via SSO | **9 accounts, 8 distinct users logged in** | VERIFIED |
| Production logins recorded | **26** (28 Sep – 11 Oct 2026) | VERIFIED |
| Unauthorised sign-in attempts correctly denied | **9** | VERIFIED |
| Project active development span | **18 Feb 2026 – 12 Oct 2026** (~8 months) | VERIFIED |

**The strongest career evidence this project provides** is not raw volume. It is four things:

1. **An architectural judgement call at the project's origin** — the business had a revenue-critical lead-ingestion service with no UI. Rather than bolt reporting onto it, this was built as a separate application so that high-churn reporting work never redeploys the system that captures leads (§8, Decision 0).
2. **Re-engineering an inherited data layer** that fetched up to 50 pages of rows into the browser to aggregate them — and, critically, silently truncated beyond 50,000 rows, meaning executive dashboards over ranges longer than ~4 months had been quietly reporting about a third of the data as complete (§7.0).
3. **A complete Google SSO + role-based authorisation system** demonstrably in daily use by a real internal team, with enforcement provable from the audit log.
4. **A genuinely hard data-quality problem solved properly** — duplicate lead submissions collapsed with window-function SQL rather than a naive filter that would have discarded the richer records (§7.2).

**Role:** Abraham is the author of record and technical owner of both repositories — 51 of 52 backend commits and 39 of 56 frontend commits (see §4 for the full attribution breakdown, including a colleague's contributions and AI-assisted development).

---

## 2. Project Overview

### What it does

The application presents internal staff with three primary views over insurance lead data:

1. **Dashboard** — aggregate KPIs (total leads, contactability, top source/type, Google CPC volume), plus breakdowns by lead type, traffic source and campaign, an hourly arrival histogram, and a volume trend chart.
2. **Leads Table** — a server-side filtered, sorted, cursor-paginated table of individual leads with CSV export, supporting filters on free-text search, lead type, source, company, UTM medium/source/campaign, query-param presence, and date range.
3. **Lead Details** — a full record view including the raw quote-form submission (`info`) rendered dynamically, since its JSON structure differs per source and customer journey.

Plus a **Noetica Campaigns** reference view (outbound dialer campaign configuration) and an **Admin** area (users, invites, audit log).

### Who uses it

VERIFIED: 9 user accounts exist, all on the `insurancerevolution.co.uk` email domain. 8 distinct users have successfully logged in. Role distribution: 3 admins, 5 standard users (active), 1 pending invite.

INFERRED: Based on the role model and feature set (read-only reporting, CSV export), the user base is internal operations/marketing staff rather than customers. No customer-facing surface exists in the codebase.

### Business problem it solves

VERIFIED, from the sibling `lead-api-dotnet` repository, schema evidence and commit history:

Before this project existed, the business ran **`lead-api-dotnet`** — a .NET Core service (first commit 24 September 2025, itself a C# reimplementation of an earlier Go API) that **ingests inbound leads, validates and normalises UK phone numbers, and forwards them to Noetica (outbound dialer), Marketplace, and Leela**, alongside Brevo contact creation and fleet SMS dispatch. It writes to the `leads` table this reporting platform reads.

That service is **business-critical on the revenue path**: if it fails, leads are not captured or not enrolled into the dialer. But it had **no user interface whatsoever**. Directors and managers had no way to see lead volumes, sources, or performance — answering "how many convicted-driver leads came from Quotezone last week?" required someone with direct database access.

**The decision that created this project** (STRONGLY SUPPORTED — stated by the project owner, corroborated by repository structure and the complete absence of any UI in `lead-api-dotnet`): rather than extend the ingestion service with reporting endpoints and a UI, build reporting as a **separate full-stack application reading the same database**. Reporting is high-churn work — new filters, columns, charts and date ranges get requested constantly — and every such change would otherwise mean redeploying the service that captures revenue-generating leads. Separating them means reporting can iterate freely while the ingestion path stays untouched.

This is the architectural reason behind the read-only posture described throughout this document. It is not merely *"we don't own the table"* — it is a **deliberate blast-radius decision** to decouple a volatile reporting surface from a critical writer. See §8 for the full decision record.

VERIFIED constraint (documented in `AGENTS.md`, enforced in code): the application contains **no `create`, `update`, `delete` or `upsert` calls against `leads`** — it is strictly read-only against data another service owns and writes.

The `leads` table itself carries 21 columns, most of which this application introduced nothing to; fields like `queue_id`, `noetica_customer_id`, `leela_id`, `fail_reason` and `cancelled_at` are written by the ingestion service and the dialer. Data begins 9 January 2024 — over a year before this project's first commit (18 February 2026).

---

## 3. System Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│  BROWSER                                                           │
│  React 18 + TypeScript + Vite SPA                                  │
│  TanStack Query (cache/persist) · TanStack Table · Recharts        │
│  Radix/shadcn UI · React Router                                    │
└───────────┬──────────────────────────────────┬─────────────────────┘
            │ HTTPS (Bearer JWT)               │ WebSocket (Socket.IO)
            │                                  │ auth: session token
            ▼                                  ▼
┌────────────────────────────────────────────────────────────────────┐
│  Azure App Service — bi-backend.insurancerevolution.co.uk          │
│  NestJS 11 API (global prefix /api)                                │
│                                                                    │
│  SessionAuthGuard (global APP_GUARD)                               │
│    └─ verifies JWT → re-reads AuthUser from DB on EVERY request    │
│  PermissionsGuard → @RequirePermissions('records.view' …)          │
│                                                                    │
│  Modules: internal-leads · auth · users · invites ·                │
│           audit-log · mail · noetica-campaigns                     │
│                                                                    │
│  InternalLeadsGateway — polls MAX(id), MAX(updated_at) every 10s,  │
│    broadcasts `leads:changed` to connected clients                 │
└───────┬──────────────────────────────┬──────────────┬──────────────┘
        │ Prisma 7 + raw SQL           │ OAuth2       │ HTTPS
        │ (READ-ONLY on leads)         │              │
        ▼                              ▼              ▼
┌──────────────────────┐   ┌──────────────┐   ┌────────────────────┐
│ Azure PostgreSQL     │   │ Google       │   │ UCP internal mail  │
│ (pre-existing, NOT   │   │ Identity     │   │ service (invite    │
│  owned by project)   │   │ (OIDC)       │   │ notifications)     │
│                      │   └──────────────┘   └────────────────────┘
│ SHARED (read-only):  │
│  leads (642k, 1.2GB) │
│  api_keys (86)       │   ┌──────────────────────────────────────┐
│  companies (6)       │   │ Azure Static Web Apps                │
│  noetica_campaigns   │   │ bi.insurancerevolution.co.uk         │
│ OWNED by project:    │   │ (GitHub Actions deploy)              │
│  auth_users (9)      │   └──────────────────────────────────────┘
│  invites (10)        │
│  audit_logs (55)     │
└──────────────────────┘
```

**Key architectural property (VERIFIED):** the database is split into tables the project *reads but does not own* (`leads`, `api_keys`, `companies`, `noetica_campaigns`) and tables the project *created and owns* (`auth_users`, `invites`, `audit_logs`). This separation is deliberate and documented, and the auth tables were created in production by a reviewed SQL script rather than an automatic migration — precisely so no migration tool could ever alter the shared tables.

---

## 4. My Role and Contribution

### Attribution evidence (VERIFIED from `git log`)

**Backend repository — 52 commits total:**

| Author | Commits | Notes |
|---|---|---|
| AbrahamNobleOX | 51 | All application code |
| gsp8181 | 1 | Azure deployment workflow scaffold (auto-generated) |

**Frontend repository — 56 commits total:**

| Author | Commits | Notes |
|---|---|---|
| AbrahamNobleOX / Abraham | 39 | Feature work + all PR merges (#1–#12) |
| gpt-engineer-app[bot] / Lovable | 11 | Initial UI scaffold (Feb 2026, pre-dates feature work) |
| Mati-Sh / Mati-Shaikh | 6 | Colleague contributions (see below) |

**Colleague contributions (explicitly NOT mine)** — Mati-Sh authored: date-filter query-param fix, the initial dedicated Lead Details page, responsive layout work across dashboard/tables/details, the "Has Query Params" filter, and the Export Report popup with CSV/XLSX downloads. These were merged by Abraham via PRs #6, #7, #8, #11 and #12.

**Initial UI scaffold (NOT mine)** — the first dashboard component shell was generated by the Lovable/gpt-engineer tool in February 2026. Substantially all of that scaffold's data layer was later replaced by backend-driven endpoints.

### Verified personal contribution

Based on commit authorship and the content of those commits:

- **Entire backend application** — NestJS service architecture, Prisma data layer, all seven feature modules, all API endpoints, DTO validation, error handling, and test suite.
- **Backend-driven reporting re-architecture (Sept 2026)** — moved dashboard aggregation, CSV export and pagination from the client to raw-SQL endpoints.
- **Google SSO + app-owned authorisation system** — complete: OAuth flow, session JWTs, global auth guard, permission model, invites (single + bulk CSV), audit logging, and three admin UI pages.
- **Real-time updates** — Socket.IO gateway and the client-side invalidation bridge.
- **Multi-source filtering** — removal of the hardcoded source restriction, `company_id` resolution via `api_keys`, source/provider enrichment with caching, and the metadata endpoints behind the filter dropdowns.
- **Noetica Campaigns feature** — read-only endpoint plus frontend page.
- **Local development database tooling** — Docker Compose Postgres, a safety-guarded production→local sampling script, and the `with-local-db.sh` wrapper that makes it structurally impossible to point a migration command at production.
- **Production deployment and configuration** — Azure App Service and Static Web Apps deployment, environment configuration, custom domain wiring.
- **Project documentation and engineering standards** — `AGENTS.md` and `CLAUDE.md` (~54 KB combined) documenting database ownership rules, architecture decisions, and a dated change log.
- **Performance re-engineering of an inherited data layer** — see §7.0; the single largest technical improvement in the project.

### Cross-system contribution: `lead-api-dotnet` (VERIFIED)

Abraham also contributed **16 of 119 commits** to the upstream .NET ingestion service (predominantly authored by Geoffrey Prytherch, 94 commits). His contributions there are notable because they are **security and integration work on the critical revenue path**:

| Contribution | Date | Significance |
|---|---|---|
| PII redaction and conditional logging for sensitive request data (API + Function middleware) | Aug 2026 | Prevents customer PII leaking into application logs — a data-protection control |
| `x-api-key` header authentication across API and Function projects, with Swagger docs and tests | Sep 2026 | Access control on the ingestion endpoints |
| Noetica enrolment service and API controller, with models and tests | Aug–Sep 2026 | New integration feature; later extended for type-specific handling and existing-customer identity reuse |
| Noetica list timing rules (list 150 / list 53 latest-time behaviour) | Sep 2026 | Business-rule work affecting when leads are dialled |
| Brevo contact strategy and fleet SMS message changes | Nov 2025 | Marketing/communication integration |

**Why this matters for the career record:** it establishes that the separation-of-concerns decision in §8 was made by someone with **first-hand working knowledge of the critical service** — not an outsider guessing at its risk profile. It also demonstrates competence across a second language and stack (C#/.NET alongside TypeScript/Node).

**Qualification:** `lead-api-dotnet` is not Abraham's project. He is a contributor (16/119 commits), not its owner or architect.

### Important qualification (declare honestly)

A portion of the implementation across both repositories was **AI-assisted**: 9 backend and 13 frontend commits carry `Co-Authored-By: Claude` trailers, and the "canonical lead sessions" feature (commit `44e8d4f`, 29 Sep 2026) was implemented by an AI coding agent under Abraham's direction and review.

This does not diminish the contribution — Abraham specified the requirements, made the architectural decisions, reviewed and integrated the work, and owns it in production — but any claim should be defensible as *"I designed, directed, reviewed and shipped this"* rather than *"I hand-wrote every line."* This is standard modern practice and is increasingly expected, but it is better to be precise about it in an interview than to be caught out.

---

## 5. Production Evidence and Statistical Analysis

All figures below were obtained on **12 October 2026** via read-only aggregate queries. No raw customer data was retrieved; no PII appears in this document.

### 5.1 Scale of data served

| Metric | Value | Source / method | Level |
|---|---|---|---|
| Lead rows in production | 642,348 | `pg_stat_user_tables.n_live_tup` (planner estimate) | VERIFIED |
| `leads` table size | 1,213 MB | `pg_total_relation_size` | VERIFIED |
| Earliest / latest lead | 2024-01-09 / 2026-10-12 | `MIN/MAX(created_at)` | VERIFIED |
| Reference data served | 86 API keys, 6 companies, 106 campaigns | `COUNT(*)` | VERIFIED |

> **Critical qualification:** the 642,348 figure is the size of the dataset the application **queries**, not data it created. The `leads` table predates this project by ~13 months and is populated by other systems. The defensible claim is *"built reporting over a 640k-row, 1.2 GB production dataset"* — never *"processed 640k leads."*

### 5.2 Lead volume over time (VERIFIED)

Monthly `COUNT(*)` on `created_at`, October 2025 – October 2026:

| Month | Leads | Month | Leads |
|---|---|---|---|
| 2025-10 | 13,985 | 2026-04 | 10,323 |
| 2025-11 | 16,982 | 2026-05 | 9,694 |
| 2025-12 | 13,509 | 2026-06 | 10,873 |
| 2026-01 | 16,702 | 2026-07 | 10,232 |
| 2026-02 | 14,845 | 2026-08 | 9,686 |
| 2026-03 | 12,331 | 2026-09 | 10,868 |
| | | 2026-10 (partial) | 3,700 |

**DERIVED:** the 12 complete months October 2025 – September 2026 total **150,030 leads**, a mean of **12,503 per month**.

**VERIFIED (90-day window, July–October 2026):** 30,255 rows; mean **332 leads/day**; peak **572 leads/day**; spanning **23 distinct sources** and **40 distinct lead types**; 21,837 rows flagged `processed`; 8,003 flagged `duplicate`.

### 5.3 Duplicate-collapse effectiveness (VERIFIED — strongest engineering metric)

The canonical-session algorithm was executed against 30 days of real production data:

| Measure | Value |
|---|---|
| Raw lead rows in window | 10,587 |
| Distinct canonical sessions | 8,327 |
| **Reduction in rows shown to users** | **21.3%** |

**Method:** group rows sharing a normalised mobile/email identity *and* the same lead `type` *and* falling within a 30-minute session window; count distinct sessions versus raw rows. This is reproducible — the query is in the appendix.

**Why this matters:** 8,003 of the last 90 days' rows carry `duplicate = true`. The naive fix — filtering those out — would have been **wrong**: investigation showed those rows frequently contain the *richer* final-submission payload, while their non-duplicate partners carry the dialer metadata. The implemented solution keeps both and merges them.

### 5.4 Authorisation system adoption (VERIFIED)

Live since 28 September 2026 — roughly two weeks of production data at time of audit:

| Metric | Value | Period |
|---|---|---|
| User accounts provisioned | 9 (3 admin, 5 user, 1 pending) | 28 Sep – 7 Oct 2026 |
| Distinct users who logged in | 8 | 28 Sep – 11 Oct 2026 |
| Successful logins | 26 | 28 Sep – 11 Oct 2026 |
| Invites issued / accepted / revoked | 10 / 7 / 2 | 29 Sep – 7 Oct 2026 |
| Sign-in attempts denied | 9 | 29 Sep – 7 Oct 2026 |
| Audit events recorded | 55 across 6 action types | 28 Sep – 11 Oct 2026 |
| Email domains represented | 1 (`insurancerevolution.co.uk`) | — |

**DERIVED:** invite acceptance rate **70%** (7/10). **DERIVED:** 9 denied attempts against 26 successful logins confirms the authorisation layer is actively rejecting access, not merely passing everyone through.

---

## 6. Business Impact

Claims are limited to what the evidence supports.

**VERIFIED / DERIVED:**
- **Self-service access to a 642k-row production dataset** for 8 internal staff who previously had no application-level route to it.
- **21.3% reduction in duplicate rows surfaced**, directly improving data quality in every view (list, detail, dashboard, export) without discarding data.
- **Controlled, auditable access** — 55 audit events and 9 denied attempts demonstrate enforced access control over commercially sensitive enquiry data, replacing an API-key model where a single shared static key granted full access.
- **Expanded reporting coverage** — removal of the hardcoded `source IN (1,47)` restriction opened reporting from 2 sources to the 23 sources present in recent data, including third-party data partners.

**STRONGLY SUPPORTED (direction clear, magnitude unmeasured):**
- Reduced load on whoever previously answered ad-hoc data questions manually.
- Faster time-to-answer for lead-performance questions (dashboard + filters vs. raw SQL).

**UNKNOWN — do not claim:** time saved per staff member, cost savings, revenue impact, or any before/after performance benchmark. No baseline measurement exists (see §11).

---

## 7. Significant Engineering Challenges

### 7.0 Re-engineering an inherited client-side data layer (largest technical improvement)

**What was inherited.** The original data layer was generated by an AI scaffolding tool (`gpt-engineer-app[bot]`, Feb 2026) and subsequently extended by a teammate. Its core was `src/hooks/use-leads.ts`, containing a function called `fetchAllLeadsForRange`.

**What it actually did (VERIFIED by reading the pre-change source at commit `c65476b^`):**

```
while (hasMorePages) {
  fetch 1,000 leads from the API
  filter them client-side by date
  append to an in-memory array
  stop if the last row predates the range, or after 50 pages
}
```

Every dashboard view therefore:
1. Issued **N sequential HTTP requests** (`ceil(rows / 1000)`), each blocking the next.
2. **Over-fetched**, then discarded rows outside the range in the browser.
3. Held **the entire result set in browser memory**.
4. Aggregated it client-side — six `useMemo` passes (`getTrendData`, `getLeadsByType`, `getLeadsBySource`, `getLeadsByCampaign`, `getLeadsByHour`, recent-leads sort) plus three full-array `.filter().length` scans for the KPI cards.
5. CSV/XLSX export (`exportLeadsToExcel(leads: Lead[], …)`, authored by the teammate) consumed that same fully-materialised array, so exporting required loading everything into the browser first.

**The defect nobody had noticed.** The loop carried a hard stop: `if (currentPage > 50) break;` — a 50,000-row ceiling. At the verified production rate of ~12,500 leads/month, **any range beyond roughly four months silently truncated**. A full-year dashboard view would have aggregated only the most recent 50,000 of 150,030 leads — about a third of the data — and presented the result to directors as fact, with no error, no warning, and no visible difference from a correct result.

This reframes the work: it was not only a performance problem, it was a **silent data-correctness problem in executive reporting**.

**What replaced it.** Aggregation moved into backend SQL (`GET /internal-leads/dashboard-summary`), returning computed summary values rather than rows. Export became a backend endpoint generating CSV server-side in 2,000-row batches. The leads table moved to cursor pagination, requesting an exact total only on page 1 and caching counts for 30 seconds.

**Measured effect (DERIVED from verified production volumes):**

| | Before | After |
|---|---|---|
| HTTP requests for a one-month dashboard (~12,500 leads) | ~13 sequential | **1** |
| Rows transferred to the browser | ~12,500 full lead records | **0** (summary values only) |
| Aggregation location | Browser JS, 6 passes + 3 scans | **PostgreSQL** |
| Maximum correct range | **~50,000 rows, then silent truncation** | No ceiling |
| Export memory footprint | Entire dataset in browser | **2,000-row server-side batches** |

**Qualification:** the request-count and row-count figures are arithmetic derived from the verified code path (1,000 rows per request) and verified production volumes (~12,500 leads/month). **No before/after wall-clock latency was measured** — no baseline was captured before the change, which is recorded as an evidence gap in §17.

**Why this is senior-level evidence.** The visible symptom was slowness. The engineer's job was to recognise that (a) the fix was architectural rather than a matter of tuning the existing loop, (b) the same inherited design was simultaneously producing silently wrong numbers, and (c) correcting it meant moving work across a system boundary into SQL and replacing a teammate's recently-shipped export feature — a change that had to be justified rather than simply made.

### 7.1 Building on a production database you don't own

**Why it was hard:** the `leads` table is actively written to by an external dialer system. Any schema change, index, trigger or write could have disrupted a live business system. Standard tooling actively fights this — Prisma wants to migrate, `db push` wants to reconcile schema drift.

**How it was solved:** a documented read-only posture; a local Docker Postgres for all schema work; a `with-local-db.sh` wrapper that forces `DATABASE_URL` to the local database for the duration of any migration command, making it *structurally impossible* to point a migration at production through that path; and a production→local sampling script with read-only enforcement at both application and Postgres-session level, plus guards refusing to run if the target hostname looks like production.

When the new auth tables did need to reach production, they were applied by a reviewed, inspected SQL file (`prisma/manual-sql/auth_tables.sql`) — create-only, touching no existing table — not by a migration tool.

### 7.2 Duplicate lead submissions (the canonical-session problem)

**Why it was hard:** the obvious solution (hide `duplicate = true`) loses the richest data. The correct grouping key isn't the customer either — a genuine repeat enquiry six months later must stay visible as a separate lead.

**Solution:** a per-query SQL CTE chain implementing gap-and-islands sessionization:
1. Normalise identity (strip non-digits from mobile; lowercase/trim email).
2. Compute each row's gap from the previous row for the same identity+type via `LAG(...) OVER (PARTITION BY ... ORDER BY created_at, id)`.
3. Convert gaps > 30 minutes into session boundaries with a running `SUM(CASE WHEN gap > interval '30 minutes' THEN 1 ELSE 0 END) OVER (...)`.
4. Within each session, rank rows **twice** — once by data richness (`length(info::text) + length(query_params::text)`) to pick the display row, once by operational completeness (`processed`, `queue_id`, `noetica_customer_id` present) to pick the metadata source.
5. Emit one row per session combining both.

Because sessions are time-windowed rather than customer-global, historical repeat enquiries are preserved. The same CTE powers list, detail, dashboard and export, so every surface agrees.

*(Implemented by an AI agent under Abraham's direction, 29 Sep 2026 — see §4.)*

### 7.3 Cross-origin session auth without cookie complexity

**Why it was hard:** frontend (`bi.insurancerevolution.co.uk`) and backend (`bi-backend.insurancerevolution.co.uk`) are different hosts. Cookie-based sessions would require cross-site `SameSite=None` cookies, domain alignment and CORS credential handling.

**Decision:** return the session JWT via **URL fragment** after the OAuth callback (fragments are never sent to servers, so the token stays out of server logs and referrers), store it client-side, and attach `Authorization: Bearer` manually. Short-lived httpOnly cookies are still used — but only for the OAuth `state`/CSRF handshake, which is same-origin during the redirect dance.

**Critical security property:** the guard does **not** trust token claims for authorisation. Every request re-reads the user from the database and rejects anyone whose status is no longer `active` — so disabling a user takes effect immediately rather than at token expiry.

### 7.4 A production bug class: stale client cache masking correct backend behaviour

Three separate user-reported bugs during this audit period traced to the same root cause: `PersistQueryClientProvider` (localStorage query-cache persistence) combined with `refetchOnMount: false`.

- **Symptom A:** full-page reload loop after login. During cache *restoration*, TanStack Query reports `isLoading: false` with `data: undefined` — which the route guard read as "logged out", redirecting to `/login`, whose own effect hard-redirected back, ~10×/second.
- **Symptom B:** "Matching Leads" count stuck at 0 on the default view but correct after changing any filter — a stale cached total that `refetchOnMount: false` prevented from ever revalidating.
- **Symptom C:** after switching accounts, the previous user's email persisted in the header — the `/auth/me` cache key was a counter that reset to `0` on every page load, so a different user collided with the prior user's cached identity.

**Fixes:** incorporate `useIsRestoring()` into loading state; re-enable `refetchOnMount` so stale data self-heals; key the identity query by the **token itself** so two users can never share a cache entry.

**Lesson worth retelling:** `refetchOnMount: false` does not mean "skip refetch when fresh" — it means *never refetch on mount, stale or not*. The code comment asserted the former. The library's default (`true`) already implements the intended behaviour.

### 7.5 Search that matches how people actually search

Lead search originally used prefix matching (`startsWith`) per field. Three distinct failures surfaced, each fixed:
- Searching the last digits of a mobile number failed → switched to substring matching, with `+44` prefix handling.
- Pasting a full name ("Ilie Alexandru Buga") returned nothing, because the displayed Name column joins `first_name` and `surname` but neither field alone contains the joined string → multi-word searches now require each word to match across either column.
- Keystroke-per-request load → 400 ms debounce across the leads, campaigns, users and invites tables, deliberately **excluding** the audit log, which is expected to grow much larger.

---

## 8. Architectural Decisions and Trade-offs

### Decision 0 — Build a separate reporting application rather than extending the ingestion service

This is the project's founding decision and the one most worth being able to explain.

- **Problem:** Directors and managers needed visibility over lead data. The only system that touched that data, `lead-api-dotnet`, had no UI — but it sits directly on the revenue path, ingesting leads and enrolling them into the Noetica dialer, Marketplace and Leela.
- **Decision:** Build a separate full-stack reporting application that reads the same database, rather than adding reporting endpoints and a UI to the existing service.
- **Reasoning:** Reporting is inherently high-churn — filters, columns, charts and date ranges change constantly as business questions change. Coupling it to the ingestion service would mean frequently redeploying revenue-critical infrastructure for cosmetic or analytical reasons. Separating them confines reporting changes to a system where failure means *a chart is wrong*, not *leads are lost*.
- **Trade-offs accepted:** two codebases and two deployment pipelines; no shared validation or domain model between them; the reporting side can only read, so anything requiring a write must go back to the owning service; data access logic is effectively duplicated in a second language.
- **Result (VERIFIED):** ~8 months and 108 commits of frequent reporting iteration — including a complete authentication system, real-time updates, a deduplication engine and repeated UI changes — with **zero deployments to the ingestion service for reporting reasons**. The ingestion service's own commit history over the same period contains only ingestion concerns.

| Decision | Reasoning | Trade-off | Result |
|---|---|---|---|
| **Move aggregation to backend raw SQL** (Sept 2026) | Inherited client-side layer fetched up to 50 pages of rows to aggregate in the browser, and silently truncated beyond 50,000 rows | Lost client-side flexibility; SQL harder to test | Dashboard, export and counts scale with the dataset; silent truncation eliminated (§7.0) |
| **Cursor pagination for `id`/`created_at`, offset fallback for name sorts** | Cursors can't express arbitrary sort columns | Two code paths; name-sorted views silently use offset | Deep pagination fast on the common cases |
| **30s count cache; `include_total` only on page 1** | `COUNT(*)` over 642k rows with filters is expensive per page | Total can lag live data up to 30s | Documented as a known, accepted limitation |
| **App-side 10s polling for live updates, not `LISTEN`/`NOTIFY`** | Triggers would mean altering a table the project doesn't own | Up to 10s latency; a wasted query every 10s | Real-time UX with zero schema footprint on shared infrastructure |
| **JWT in URL fragment, not cross-site cookies** | Different hosts; fragments never reach servers or logs | Token in localStorage (XSS-exposed) | Avoided cross-site cookie complexity; mitigated by always re-checking the DB |
| **Google SSO + app-owned authorisation tables** | Google proves *identity*; the app decides *access* | New tables to create in production | Immediate revocation; full audit trail; 9 denied attempts logged |
| **Manual reviewed SQL for production auth tables** | Migration tooling could touch shared tables | Manual step; no migration history | Zero risk to the dialer system's tables |
| **In-memory 5-min lookup caches** (sources, companies) | ~86 and ~6 rows, rarely changing | Up to 5 min staleness; per-instance | Removed a join from every lead query |

---

## 9. Problems Solved (Interview-ready accounts)

### Google sign-in failed with "session expired" on every production attempt

- **Situation:** SSO worked locally; every production sign-in failed.
- **Problem:** The frontend's hardcoded production API base pointed at the raw `*.azurewebsites.net` hostname, while `GOOGLE_CALLBACK_URL` pointed at the custom domain `bi-backend.insurancerevolution.co.uk`.
- **Investigation:** Traced the OAuth state check. The CSRF `state` cookie is set on whichever host *starts* the flow; Google then redirects to the *callback URL's* host. Two different hostnames → the browser never sends the cookie back → state mismatch → `invalid_state`.
- **Solution:** Pointed the frontend's production API and WebSocket base at the same custom domain as the callback URL.
- **Result:** Sign-in worked; 26 production logins recorded since. (Commit `30e75a8`.)
- **Why it's a good story:** the error message said "expired" but nothing had expired — it was a cookie-scope problem. Good illustration of reading what the code actually checks rather than what the message says.

### Deployment succeeded but the feature was dead

- **Situation:** Auth code deployed to production; tables didn't exist.
- **Problem:** Deploying application code never creates tables — and this project deliberately has no automatic migration step, because migrations could touch the shared `leads` table.
- **Solution:** A reviewed, create-only SQL file applied manually to production after confirming by read-only query that none of the three tables already existed.
- **Result:** Tables live; 9 users, 10 invites, 55 audit events since.
- **Why it's a good story:** demonstrates treating "I can't safely automate this" as a legitimate engineering decision rather than a gap.

### A clean merge silently deleted a colleague's shipped feature

- **Situation:** Merging a long-running branch into `main`. Git reported no conflicts.
- **Problem:** Review of the merge diff showed `src/lib/export-excel.ts` deleted and the `exceljs` dependency gone. A colleague had merged client-side Excel export (PR #12) while the other branch had deliberately removed client-side export in favour of backend-driven CSV. Because one side *deleted* rather than edited, Git saw no textual conflict and silently applied the deletion.
- **Solution:** Stopped before pushing, reconstructed what each side had done, and escalated it as a product decision rather than silently picking a winner.
- **Result:** Decision taken to keep backend CSV only; merge pushed on that explicit basis.
- **Why it's a good story:** "no conflicts" is not "no conflict." Reviewing a merge's *effect*, not just its exit code, prevented destroying a teammate's work.

---

## 10. Verified Career Claim Bank

> **Claim 1** — Built and shipped an internal reporting platform providing 8 internal staff with filtered, real-time access to a 642,348-row (1.2 GB) production insurance-lead dataset.
> **Level:** VERIFIED · **Evidence:** `pg_stat_user_tables`; 8 distinct `auth.login` actors · **Period:** Feb–Oct 2026
> **Qualification:** The dataset pre-dates the project and is populated by other systems; the platform reads it, it did not generate it.

> **Claim 2** — Designed and implemented a canonical lead-session algorithm that collapses near-duplicate submissions, reducing rows shown to users by 21.3% without discarding any data.
> **Level:** VERIFIED · **Evidence:** 10,587 raw rows → 8,327 sessions over a 30-day production window · **Period:** Sept 2026
> **Qualification:** Implemented by an AI agent under my direction and review. Measured on 30 days; rate will vary by period and source mix.

> **Claim 3** — Delivered a complete Google SSO and role-based authorisation system now in daily production use: 9 accounts, 26 logins, 10 invites (70% accepted), 9 unauthorised attempts denied, 55 audit events.
> **Level:** VERIFIED · **Evidence:** `auth_users`, `invites`, `audit_logs` aggregates · **Period:** 28 Sep – 11 Oct 2026
> **Qualification:** ~2 weeks of production data at time of audit; small absolute numbers reflect a small internal team.

> **Claim 4** — Re-architected dashboard, export and pagination from client-side aggregation to backend SQL endpoints, reducing a one-month dashboard view from ~13 sequential HTTP requests and ~12,500 lead records held in browser memory to a single request returning computed values.
> **Level:** VERIFIED (architecture) / DERIVED (request and row counts) · **Evidence:** Pre-change source at `c65476b^`; commits `98286e5`, `c65476b`; production volume of ~12,500 leads/month · **Period:** Sept 2026
> **Qualification:** Request/row counts are arithmetic from the verified code path and verified volumes. No wall-clock latency benchmark was captured.

> **Claim 4b** — Identified and eliminated a silent data-truncation defect in executive reporting: the inherited client-side data layer capped fetching at 50,000 rows, so any date range beyond ~4 months silently aggregated only part of the data and presented it as complete.
> **Level:** VERIFIED · **Evidence:** `if (currentPage > 50) break;` in `use-leads.ts` at `c65476b^`, against a verified rate of ~12,500 leads/month · **Period:** Sept 2026
> **Qualification:** No evidence establishes whether an incorrect figure was actually acted upon by a stakeholder — the claim is that the defect existed and was removed.

> **Claim 4c** — Took ownership of a data layer originating from an AI scaffold and extended by a teammate, re-engineering it across the client/server boundary — including replacing a colleague's recently shipped client-side Excel export with a backend-generated CSV export processed in 2,000-row batches.
> **Level:** VERIFIED · **Evidence:** File authorship via `git log --diff-filter=A`; commit `c65476b`; `exportCsv` batch size in `internal-leads.service.ts` · **Period:** Jul–Sept 2026
> **Qualification:** The export replacement removed XLSX support; this was later confirmed as an accepted product trade-off rather than an oversight.

> **Claim 5** — Expanded reporting coverage from 2 hardcoded lead sources to all sources present in the data (23 distinct sources, 40 lead types in a 90-day window), adding source/company filtering.
> **Level:** VERIFIED · **Evidence:** Commit `25c45a7`; production aggregates · **Period:** Sept 2026

> **Claim 6** — Delivered real-time updates via a Socket.IO gateway with zero schema footprint on a shared production table, by polling rather than installing database triggers.
> **Level:** VERIFIED · **Evidence:** `internal-leads.gateway.ts`; `AGENTS.md` rationale · **Period:** Sept 2026
> **Qualification:** Up to ~10s propagation latency by design.

> **Claim 7** — Established and documented a production-safety engineering standard for a database owned by another system: read-only posture, local Docker Postgres for schema work, a guarded production→local sampling script, and a wrapper making it structurally impossible to run a migration against production.
> **Level:** VERIFIED · **Evidence:** `scripts/with-local-db.sh`, `sync-from-prod.ts`, `docker-compose.local-db.yml`, `AGENTS.md` · **Period:** Sept 2026

> **Claim 8** — Owned both repositories end-to-end across ~8 months: 51/52 backend commits and 39/56 frontend commits, including Azure App Service and Static Web Apps deployment via GitHub Actions.
> **Level:** VERIFIED · **Evidence:** `git log`; workflow files · **Period:** Feb–Oct 2026
> **Qualification:** A colleague contributed 6 frontend commits; the initial UI shell was tool-generated; a portion of later work was AI-assisted.

> **Claim 9** — Diagnosed and fixed a client-side cache-restoration race causing a ~10 requests/second full-page reload loop in production authentication.
> **Level:** VERIFIED · **Evidence:** Commit `ca40b77`; `auth-context.tsx` · **Period:** Sept 2026

> **Claim 10** — Made and justified the architectural decision to build lead reporting as a separate application rather than extending a revenue-critical ingestion service, keeping ~8 months of high-churn reporting iteration off the lead-capture path.
> **Level:** STRONGLY SUPPORTED · **Evidence:** Absence of any UI in `lead-api-dotnet`; its integration surface (Noetica, Marketplace, Leela, Brevo, SMS); 108 reporting commits with no reporting-driven deploys to the ingestion service; stated rationale from the project owner · **Period:** Feb–Oct 2026
> **Qualification:** The reasoning is reported by the owner and corroborated by structure; no written design document predating the decision was found.

> **Claim 11** — Contributed security and integration work to the upstream .NET ingestion service, including PII redaction in request logging, API-key authentication across two deployment front-ends, and a Noetica enrolment service.
> **Level:** VERIFIED · **Evidence:** 16 commits in `lead-api-dotnet` · **Period:** Nov 2025 – Sept 2026
> **Qualification:** Contributor, not owner — 16 of 119 commits; the service is primarily authored by a colleague.

---

## 11. CV Achievement Candidates

### High-confidence quantified

- Built an internal lead-reporting platform (NestJS, React, PostgreSQL, Azure) serving filtered real-time analytics over a **642,000-row, 1.2 GB** production dataset averaging **~12,500 new leads/month**.
- Designed a SQL window-function deduplication algorithm that collapsed near-duplicate lead submissions by **21.3%** (10,587 → 8,327 rows over 30 days) while preserving every underlying record and all historical repeat enquiries.
- Shipped a Google SSO and role-based authorisation system — invites, bulk CSV onboarding, and audit logging — adopted by **9 internal users** with **26 logins** and **9 unauthorised attempts correctly denied** in its first two weeks.
- Expanded reporting coverage from **2 hardcoded data sources to 23**, adding server-side source, company and campaign filtering across **40 lead types**.
- Authored **51 of 52** backend and **39 of 56** frontend commits across an 8-month build, owning architecture, deployment and production operation.
- Re-engineered an inherited client-side data layer into backend SQL endpoints, cutting a one-month dashboard view from **~13 sequential HTTP requests and ~12,500 records held in browser memory to a single request** returning computed values.
- Eliminated a **silent 50,000-row truncation defect** in executive reporting, where any date range beyond ~4 months had been aggregating roughly a third of the available data and presenting it as complete.

### High-confidence non-quantified

- Architected lead reporting as a separate full-stack application rather than extending a revenue-critical lead-ingestion service, isolating high-churn reporting changes from the system that captures and dials inbound leads.
- Took ownership of a data layer originating from an AI scaffold and extended by a teammate, re-engineering it across the client/server boundary — including replacing a colleague's client-side Excel export with a backend-generated CSV export processed in 2,000-row batches.
- Contributed PII redaction in request logging, API-key authentication, and a Noetica dialer enrolment service to the upstream .NET ingestion API, working across C#/.NET alongside TypeScript/Node.
- Implemented real-time dashboard updates over WebSockets with **zero schema footprint** on a production table shared with an external telephony system, deliberately polling instead of installing database triggers.
- Established a production-safety engineering standard for a non-owned database — read-only posture, local Docker Postgres, and tooling that makes running a migration against production structurally impossible.
- Diagnosed and resolved a production authentication reload loop caused by a query-cache restoration race, plus two related stale-cache defects including cross-user identity leakage.
- Debugged an OAuth `invalid_state` failure traced to a cookie-scope mismatch between two hostnames in the sign-in redirect chain.

### Requiring confirmation before use

- Any claim about staff hours saved, cost reduction, or decision-making speed.
- Any before/after performance or latency figure.
- Total leads ever processed through the business (the table pre-dates the project).

---

## 12. Portfolio Case Study Material

**Challenge.** An insurance brokerage had years of inbound enquiry data sitting in a PostgreSQL table — but the table was shared with, and actively written to by, an external telephony/dialer system. Staff had no application-level way to analyse it, and no change could risk disrupting the live dialer.

**Role.** Sole backend engineer and primary frontend engineer; architecture, implementation, deployment and production ownership.

**Solution.** A NestJS API and React SPA deployed on Azure, built with a strict read-only posture toward the shared tables, and its own separate tables for authentication and auditing. Reporting aggregation lives in backend SQL so it scales with the dataset. Real-time updates use application-side polling rather than database triggers. A SQL window-function sessionization algorithm collapses near-duplicate submissions without discarding data.

**Three problems worth writing up in depth:**
1. *Deduplication without data loss* — why `WHERE NOT duplicate` was the wrong answer, and how gap-and-islands sessionization with dual ranking solved it.
2. *Operating safely on infrastructure you don't own* — tooling that makes unsafe operations structurally impossible rather than merely discouraged.
3. *The stale-cache bug class* — one library misconfiguration producing three unrelated-looking user-facing bugs.

**What I'd improve next.** API-level tests for dashboard correctness, cursor pagination and export (currently 23 unit tests, no integration coverage); a trigram index to support substring search at 1M+ rows; capturing baseline metrics *before* optimising so improvements are measurable.

---

## 13. Interview Story Bank

| Story | Best for | Core point |
|---|---|---|
| **Inherited data layer re-engineering (§7.0)** | **Performance, code review, seniority** | **The slowness was the symptom; silent truncation was the real defect** |
| **Separate service vs. extending the critical API (§8 Decision 0)** | **Architecture, risk judgement** | **Confine volatile work away from the revenue path** |
| Canonical lead sessions (§7.2) | Complex problem solving, data modelling | The obvious fix destroys data; investigate before filtering |
| OAuth `invalid_state` (§9) | Production debugging | The error message named the wrong cause |
| Merge deleted a colleague's feature (§9) | Collaboration, judgement | "No conflicts" ≠ "no conflict"; escalated rather than deciding unilaterally |
| Read-only posture on shared DB (§7.1) | Ownership, risk management | Making the unsafe path impossible beats documenting it |
| Reload loop / cache races (§7.4) | Debugging under ambiguity | Distinguishing a live loop from cumulative log noise |
| Tables missing after deploy (§9) | Deployment, ops | Deliberately un-automated steps are a valid decision |
| Backend re-architecture (§8) | Architecture, scale | Moving work to where the data is |

**Worked example — STAR, inherited data layer (strongest seniority story):**
- **Situation:** The dashboard was slow. The data layer had been generated by an AI scaffolding tool and extended by a teammate; nobody had revisited its design.
- **Task:** Make reporting performant against a 642k-row table.
- **Action:** Read the inherited code rather than tuning around it, and found it fetched up to 50 pages of 1,000 rows in a sequential loop, filtered them in the browser, held everything in memory, and aggregated with six passes plus three full-array scans. It also carried a hard `break` at 50 pages — so beyond ~50,000 rows it silently returned partial data. At ~12,500 leads/month that meant any range over ~4 months was quietly wrong. I moved aggregation into SQL behind a summary endpoint, moved export to a server-side batched CSV endpoint (replacing a colleague's recently merged client-side Excel export, which I raised rather than silently dropped), and switched the table to cursor pagination with a cached count.
- **Result:** A one-month dashboard went from ~13 sequential requests and ~12,500 records in browser memory to one request returning computed values, with no row ceiling and therefore no silent truncation.
- **Evidence:** Pre-change source at `c65476b^`; commit `c65476b`; production volumes in §5.2.
- **If asked what you'd do differently:** capture a latency baseline before the change — the improvement is architecturally obvious but not measurable after the fact (§17).

**Worked example — STAR, canonical sessions:**
- **Situation:** Users saw the same customer twice in the leads table minutes apart.
- **Task:** Remove the duplication without losing data.
- **Action:** Investigated the duplicate rows and found the `duplicate = true` rows often held the *richer* payload while their partners held dialer metadata — so filtering was wrong. Specified a time-windowed sessionization approach: normalise identity, detect >30-minute gaps as session boundaries, rank rows twice within a session (data richness for display, operational completeness for metadata), and emit one merged row. Applied the same CTE across list, detail, dashboard and export for consistency.
- **Result:** 21.3% fewer rows surfaced over 30 days of production data, with every underlying record retained and genuine repeat enquiries months apart preserved.
- **Evidence:** Commit `44e8d4f`; reproducible query in the appendix.

---

## 14. Technical Article / Blog Opportunities

1. **"The dashboard wasn't slow. It was lying."** — A client-side fetch-all loop with a 50-page safety break that silently truncated results past 50,000 rows. Covers why "safety limits" in data-fetching code are correctness bugs in disguise, and why aggregation belongs next to the data. Strongest article in the set; fully public-safe when generalised. *Anonymise:* employer, volumes (use ratios).
2. **"Don't put the reporting UI on the revenue path"** — Choosing a separate reporting application over extending a critical ingestion service: blast radius, deployment cadence mismatch, and the duplication you accept in exchange. *Omit:* downstream integration names.
3. **"Your duplicates might be your best data"** — Why `WHERE NOT duplicate` was the wrong fix, and gap-and-islands sessionization in PostgreSQL. *Anonymise:* company, source names. *Omit:* the production SQL verbatim.
2. **"`refetchOnMount: false` doesn't mean what you think"** — One TanStack Query misconfiguration, three unrelated-looking production bugs. Highly generalisable; fully public-safe.
3. **"Building on a database you don't own"** — Engineering patterns for read-only tenancy on shared production infrastructure: tooling that makes mistakes structurally impossible. *Omit:* hostnames, schema specifics.
4. **"Google SSO authenticates; your database authorises"** — Why re-reading the user on every request beats trusting token claims, and what a minimal invite/audit model looks like.
5. **"It said the session expired. Nothing had expired."** — Debugging OAuth state failures caused by cookie scope across two hostnames.

---

## 15. LinkedIn / Social Content Opportunities

Each grounded in a real incident from this project:

- **A `break` statement was quietly costing us a third of our data** — the 50-page cap that turned a performance shortcut into a silent reporting defect. *Evidence:* the inherited loop; verified monthly volumes. Strongest post in the set.
- **Reporting doesn't belong on the revenue path** — why the lead dashboard became its own application instead of another endpoint on the ingestion API. *Evidence:* the separation decision and 8 months of iteration without touching ingestion.
- **The 21.3% that wasn't noise** — on investigating duplicates before deleting them. *Evidence:* measured collapse rate.
- **Inheriting AI-generated code** — what it's like to own, audit and re-engineer a data layer a scaffolding tool produced and a teammate extended. Topical and credible because it's specific. *Evidence:* file authorship history; the re-architecture.
- **"No merge conflicts" is not the same as "no conflict"** — on reviewing what a merge *did*, not whether it exited cleanly. *Evidence:* the deleted Excel export.
- **Make the dangerous thing impossible, not just discouraged** — on the wrapper that forces migrations to the local database. *Evidence:* `with-local-db.sh`.
- **The error message was lying** — OAuth "session expired" that was actually a cookie-scope bug. *Evidence:* commit `30e75a8`.
- **Shipping the schema is not shipping the feature** — code deployed, tables absent. *Evidence:* the manual SQL application step.
- **On AI-assisted engineering** — an honest reflection on directing, reviewing and owning AI-written code in production. *Evidence:* the co-authored commits; genuinely topical and credible because it's specific.

---

## 16. Private vs Public-Safe Information

### PRIVATE — internal records, interviews, CV decisions only
Production hostnames and custom domains; `leads` schema specifics and the external dialer's column names; internal company/source identifiers (`source IN (1,47)`, company prefixes); absolute user counts tied to a named employer; exact production row counts attributed to a named company; the UCP internal mail service and its API shape; audit-log action taxonomy; the verbatim canonical-session SQL.

### PUBLIC-SAFE — portfolio, LinkedIn, articles
The engineering patterns in §7 and §8 described generically ("an insurance brokerage", "a shared production table"); the deduplication *technique* (gap-and-islands sessionization is a well-known public pattern); the TanStack Query cache lessons in full; the OAuth cookie-scope lesson in full; technology stack; percentage-based metrics (21.3% collapse, 70% invite acceptance) without naming absolute counts or the employer; the "database you don't own" patterns.

**When uncertain, classify as private.** In particular: do not publish absolute lead volumes alongside the employer's name — that is commercially sensitive enquiry-volume information.

---

## 17. Evidence Gaps and Further Metrics Worth Collecting

| Missing evidence | Why it matters | Where it might exist | Question to ask |
|---|---|---|---|
| Pre-project baseline — how lead questions were answered before | Converts "built a reporting tool" into a quantified efficiency claim | Team recollection; predecessor spreadsheets/tooling | "How did we answer lead-performance questions before this existed?" |
| Dashboard load time before/after the backend re-architecture | Would turn §7.0's architectural argument into a hard performance number — the single most valuable missing metric | Azure App Service / Static Web Apps metrics around 27 Sep 2026; browser performance traces if anyone captured one | "Can I pull API response-time or page-load metrics either side of the September release?" |
| Whether anyone acted on a truncated (>4-month) dashboard figure before Sept 2026 | Would escalate the truncation fix from "defect removed" to "incorrect reporting corrected" | Stakeholder recollection; any saved reports or screenshots predating the fix | "Did anyone pull a quarterly or annual lead figure from the dashboard before late September?" |
| Any written record of the separate-service decision | Would upgrade Decision 0 from STRONGLY SUPPORTED to VERIFIED | Slack/Teams history, tickets, or meeting notes from ~Feb 2026 | "Is there a ticket or thread where we agreed not to put reporting in the .NET API?" |
| Number of staff who *should* have access vs. 9 provisioned | Distinguishes full rollout from partial pilot | Management / rollout plan | "Is 9 users the full intended audience, or phase one?" |
| Exact production go-live date of the first version | Currently inferred from commit dates | Azure deployment history; first production login | "When was this first used by staff for real work?" |
| Usage depth — most-used filters, export frequency | Demonstrates actual adoption beyond login counts | Not currently instrumented | *Action:* add lightweight usage logging |
| Full-table aggregate statistics | Annual totals timed out at 60s | Requires a maintenance window or longer timeout | *Action:* re-run during low-traffic hours |
| Whether the canonical-session change altered reported KPIs | Stakeholder-visible impact of the dedup work | Compare dashboard totals either side of 29 Sep 2026 | "Did reported lead counts shift when dedup shipped, and was that communicated?" |

---

## 18. Appendix — Evidence Notes

**Audit date:** 12 October 2026. **Database access:** read-only session (`default_transaction_read_only = on`, 60s statement timeout). No `INSERT`/`UPDATE`/`DELETE`/DDL executed. No raw customer records retrieved; all queries aggregate. No credentials appear in this document.

**Reproducible duplicate-collapse query** (the §5.3 metric):

```sql
WITH filtered AS (SELECT l.* FROM leads l WHERE created_at >= now()-interval '30 days'),
keyed AS (SELECT filtered.*, regexp_replace(COALESCE(mobile,''),'[^0-9]','','g') mk,
                 lower(btrim(COALESCE(email,''))) ek FROM filtered),
identified AS (SELECT keyed.*, CASE WHEN mk<>'' THEN 'mobile:'||mk
                                    WHEN ek<>'' THEN 'email:'||ek
                                    ELSE 'lead:'||id::text END ik,
                      COALESCE(type,'') tk FROM keyed),
ordered AS (SELECT identified.*, LAG(created_at) OVER (PARTITION BY ik,tk ORDER BY created_at,id) prev FROM identified),
sessioned AS (SELECT ordered.*, SUM(CASE WHEN prev IS NULL THEN 1
                                         WHEN created_at-prev > interval '30 minutes' THEN 1
                                         ELSE 0 END)
                                OVER (PARTITION BY ik,tk ORDER BY created_at,id) si FROM ordered)
SELECT count(*) raw_rows, count(DISTINCT (ik,tk,si)) canonical_sessions,
       round(100.0*(count(*)-count(DISTINCT (ik,tk,si)))/count(*),1) pct_collapsed
FROM sessioned;
-- Result 2026-10-12: 10,587 | 8,327 | 21.3
```

**Codebase size (VERIFIED):** backend 3,731 LOC application + 466 LOC tests; frontend 6,280 LOC excluding the generated shadcn/Radix component library. Backend test suite: 4 spec files, 23 test cases.

**Known measurement limitations:**
- `leads` row count (642,348) is a planner estimate from `pg_stat_user_tables`, not an exact `COUNT(*)` — exact counts timed out at 60s. Accurate to within a fraction of a percent for a table of this size.
- Annual volume aggregates could not be completed (full-table scan exceeded the statement timeout); 13 months of monthly data was obtained instead.
- The 21.3% collapse rate reflects one 30-day window and will vary with source mix.
- Auth adoption figures cover ~2 weeks only.

**Qualifications to carry into any interview:**
1. The `leads` dataset pre-dates the project (earliest row 9 Jan 2024 vs. first commit 18 Feb 2026) and is populated by other systems. Claim *reporting over* it, never *processing* it.
2. A colleague (Mati-Sh) authored 6 frontend commits — notably the initial Lead Details page, responsive layout work, and the Export Report modal.
3. The initial dashboard UI shell was tool-generated (Lovable/gpt-engineer) in Feb 2026 and substantially replaced later.
4. A portion of implementation was AI-assisted (22 co-authored commits across both repos), including the canonical-sessions feature. Claim direction, architecture, review, integration and production ownership.
