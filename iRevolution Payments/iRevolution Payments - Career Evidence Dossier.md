# iRevolution Payments — Career Evidence Dossier

**Project:** iRevolution Payments (repository `o-payment-gateway`, package `irevolution-payments`)
**Organisation:** Insurance Revolution
**Audit date:** 10 October 2026
**Subject:** Abraham Akintola (`AbrahamNobleOX` / `abraham.akintola@insurancerevolution.co.uk`)
**Audit basis:** Repository source, Git history, Prisma schema and migrations, CI/CD definitions, project documentation, and **read-only analysis of the live production database**.

> **Scope note — production database audited read-only.**
> The production database (Azure Database for MySQL 8.0, `appdb`) was accessed **strictly read-only** on 10 October 2026 via a connection string held commented-out in `.env`. The session ran with `SET SESSION TRANSACTION READ ONLY` (server-confirmed, `@@transaction_read_only = 1`), behind a guard script refusing every write/DDL keyword. Only `SELECT`, aggregate and `SHOW` statements were executed. No credentials, connection strings or customer PII appear anywhere in this dossier; all findings are aggregates. See §9 and §25.

---

## 1. Executive Summary

iRevolution Payments is an internal payment-operations web application used by Insurance Revolution staff to create Worldpay hosted payment links, send them to customers by email and SMS, and track payment and communication status. It is a single Next.js 12 application with frontend and API routes in one codebase, backed by MySQL via Prisma and deployed as a Docker container to Azure App Service.

The application **originally existed from 2022–2023**, built by other engineers. **Abraham's contribution is a distinct 2026 modernisation, security and operations phase** delivered between **25 June and 5 August 2026** — roughly six weeks, 47 commits — which **went live in production on 17 July 2026** and has run continuously since.

**Why the engagement existed — the headline achievement.** Abraham was brought onto the project to resolve a **parameter-tampering vulnerability in the Worldpay payment links**. The generated payment URL carried `amount` as a plain, unsigned query parameter, so a customer sent a link for £500 could edit the URL and pay £5. A pre-existing callback check (built in 2022) refused to *record* such a payment as paid — so the money was taken by the payment provider but the request stayed `pending`, producing unreconcilable accounting and eroding customer trust in the links.

He shipped the fix — **MD5 request signing over the amount and other critical fields, with fail-closed configuration** — on **5 July 2026, as his first substantive commit, roughly ten days after joining**. The repository proves both the flaw and the fix: the pre-fix `getPaymentLink` builds the URL with no signature at all, and all 36 lines of the signing implementation are attributed to him (§14.1).

**Exposure the fix closed: 65,093 payment links totalling £45,049,376 had been issued with the editable amount parameter** between January 2023 and 4 July 2026 (§9.9). *This is the exposure surface, not a loss figure — no evidence establishes how many links were actually tampered with.*

**Other technically significant work** (the engagement then broadened into a full modernisation):

**Strongest authorship evidence:** 47 commits, all in 2026; **103 files changed, +9,029 / −1,110 lines**; **46 files created**; **84.5% of surviving source lines** (8,519 of 10,077) attributed by `git blame` at HEAD.

**Strongest verified production evidence** (all read-only from the live database, 10 Oct 2026):

| Measure | Value | Window |
|---|---|---|
| Production runtime of the 2026 release | **84 days, continuous** | 17 Jul – 9 Oct 2026 |
| Payment requests created on the new codebase | **3,308** | since go-live |
| Value of completed payments since go-live | **£1,792,455** (2,611 paid, 78.9%) | since go-live |
| Email + SMS delivery attempts via the UCP integration | **6,225** (3,361 email / 2,864 SMS) | since go-live |
| Staff users governed by the RBAC system | **71** (68 user, 3 admin), all active, all logged in | live |
| Audit events captured | **8,634**, incl. **31 blocked access attempts** | since 17 Jul |
| Automated daily reports delivered | **81 of 81 (100%), no missed day** | 20 Jul – 8 Oct |
| Bulk resend job | **70 links regenerated, 135 messages, 0 failures, 123s** | 2 Aug |
| Historical estate migrated into Azure | **65,561 payments · 125,444 communications** | pre-go-live |
| Whole-estate scale now under management | **68,869 payments · £47.76M** | 2023–2026 |

**Clearest business impact:** an unaudited, unrestricted internal tool became a role-governed, fully audited one serving 71 staff; a 16,257-record backlog of expired pending links became visible and remediable in bulk; and management gained an automated daily report that has not missed a day in 81 days. Impact that remains unquantified is identified in §24.

## 2. Project Overview

**What it does** (verified from `README.md`, source, and schema):

- Creates Worldpay Business Gateway **hosted payment page** links.
- **MD5-signs** the Worldpay payment URL to protect fields such as `amount`.
- Shortens links via a TinyURL-compatible service.
- Stores payment requests in MySQL and tracks their state (`pending` / `paid`).
- Sends payment links by **email and SMS through UCP**.
- Records a **communication history** per payment.
- Provides a payments dashboard with filtering, sorting, pagination, search, CSV export and KPI summaries.
- Provides **app-owned authorisation**: users, invites, roles, disabled users, audit logs.
- Sends **daily payment activity reports** with CSV attachments.
- Supports **admin-only bulk resend** of pending payment links with optional link regeneration.
- Includes a **legacy sync script** for migrating records from the previous database.

**Who uses it:** Insurance Revolution internal staff (agents creating customer payment requests) and administrators. Access is restricted to approved company Google accounts. **71 users are registered in production** — 68 standard users and 3 administrators, all active and all having signed in — with 46–57 distinct users creating payments in any given month (§9.4, §9.3).

**Production deployment target** (verified from `.github/workflows/deploy-prod.yml`): Azure App Service `o-payment-gateway-prod` in resource group `o-payment-gateway-rg`, image pushed to Azure Container Registry `insurancerevolution.azurecr.io`. A parallel staging pipeline targets `o-payment-gateway-stage`. The production customer-facing hostname referenced in application code is `pay.insurancerevolution.co.uk`.

---

## 3. Business Problem and Context

### 3.1 The originating problem: an editable payment amount

The application generates Worldpay **hosted payment page** links. Worldpay's hosted-page model carries the transaction parameters in the URL itself, which means the URL must be cryptographically signed or the customer can rewrite it.

**It was not signed.** The pre-fix `getPaymentLink` assembled the URL as a plain query string including `amount: values.amount`, with no signature field of any kind (VERIFIED — see the pre-fix source at `af467f9^`). A customer who received a link for £500 could change `&amount=500` to `&amount=5` and reach a legitimate Worldpay checkout for £5.

**The failure mode was worse than simple underpayment.** A safety check added in December 2022 compared the amount reported by Worldpay's callback against the stored amount and rejected the mismatch:

```
if (payment.amount !== amount) → log "insufficient amount" → HTTP 500, do not mark paid
```

So the tampered payment *completed at the provider* — the customer's card was charged the reduced amount — but the application refused to record it. The result was money received that the system could not reconcile against any completed payment request, while the request itself remained `pending` indefinitely.

**Business consequences** (as described by the engineer who was brought in to fix it; the mechanism is verified from code, the business framing is his account):
- Accounting and reconciliation problems from payments taken but never recorded.
- Loss of customer trust in the payment links.
- A related branding problem: the payment emails were generic and unbranded, and read to recipients as though they might be a scam — compounding reluctance to use the links.

**Why it had not been fixed:** the legacy Worldpay integration had **no surviving documentation**, and other developers had previously attempted the fix without success (user-reported; not independently verifiable from the repository). Getting hosted-page signing right is unforgiving — the signature must cover exactly the fields the Worldpay installation is configured to check, in the configured order, or every link breaks.

**Scale of exposure at the point of fix:** 65,093 payment links worth **£45,049,376** had been issued under the unsigned scheme (§9.9).

### 3.2 The problems the engagement then expanded to address

Once the vulnerability was closed, the engagement broadened into a modernisation of the surrounding application (VERIFIED from commit history, schema migrations and documentation):

1. **Uncontrolled access.** Any user with an allowed Google domain could generate payment links. No roles, no way to disable a leaver, no invite control, no record of who did what.
2. **No operational audit trail** — no record of logins, denied attempts, payment creation, exports or bulk operations.
3. **Unbranded, low-trust customer communications** — the generic email problem above.
4. **Expired and unsent payment links at scale**, with no bulk remediation path.
5. **No operational visibility cadence** for management.
6. **Manual deployment and a legacy database** needing repeatable container deployment and migration to Azure.

*Items 2–6 are inferred as motivations from the implemented work and documentation; item 1 and the §3.1 narrative are corroborated by code, git history and the engineer's direct account.*

## 4. My Role and Contribution

### 4.1 Authorship evidence and method

Authorship was established from `git shortlog`, per-commit authorship, and `git blame --line-porcelain` at HEAD.

| Author | Commits (all time) | Active years |
|---|---|---|
| Michal Tesar | 54 | 2022–2023 |
| **AbrahamNobleOX / Abraham** | **47** | **2026** |
| javier | 20 | 2023 |
| dependabot[bot] | 6 | — |
| Roxie Gibson | 5 | 2023–2024 |
| zayan_DevOps | 4 | 2026 |
| Others (Cassidy, Wills, Tesar alt, Göksu) | 7 | 2022–2024 |

**Every one of Abraham's 47 commits falls in 2026** (25 June – 5 August). The project's first commit was 8 June 2022.

| Metric | Value | Source |
|---|---|---|
| Commits authored | 47 | `git log --author="Abraham"` |
| Contribution window | 2026-06-25 → 2026-08-05 | commit dates |
| Files changed in phase | 103 | `git diff --shortstat af467f9^ HEAD` |
| Lines added / removed | +9,029 / −1,110 | same |
| New files created | 46 | `git diff --diff-filter=A` |
| Surviving lines attributed | **8,519 of 10,077 (84.5%)** | `git blame` across `src/`, `scripts/`, `prisma/*.prisma`, `.github/workflows/`, `*.md` |

**Per-file authorship at HEAD** (all lines, `git blame`):

| File | Abraham's share |
|---|---|
| `src/pages/admin/index.tsx` | 1,551 / 1,551 (100%) |
| `src/services/bulk-payment-resend.service.ts` | 458 / 458 (100%) |
| `src/services/authorization.service.ts` | 374 / 374 (100%) |
| `src/services/daily-report.service.ts` | 362 / 362 (100%) |
| `src/services/ucp.service.ts` | 143 / 143 (100%) |
| `src/components/payment/index.tsx` | 1,105 / 1,115 (99%) |
| `README.md` | 426 / 487 (87%) |
| `src/pages/api/cred/payment/new.ts` | 95 / 165 (58%) |

### 4.2 Verified personal contribution

**Engagement origin — payment link security (the reason he was brought in):**
- Diagnosed and fixed a parameter-tampering vulnerability allowing customers to alter the amount payable on any generated payment link (§14.1).
- Implemented MD5 request signing with a configurable ordered field list, missing-field refusal, and fail-closed configuration validation — **all 36 lines attributed to him** in commit `af467f9` (2026-07-05), his first substantive commit.
- Worked **without any surviving documentation** for the legacy Worldpay integration.
- Retained the pre-existing callback amount check as an independent second layer rather than replacing it.

**Authorisation and security system** — designed and built from scratch:
- Five new models: `AuthUser`, `Invite`, `AuditLog` (plus the settings models below).
- Three roles (`admin`, `user`, `read_only`) mapped to seven granular permissions (`users.manage`, `invites.manage`, `audit.view`, `payments.create`, `payments.view`, `payments.export`, `payments.send`) in `src/domain/auth.ts`.
- `requirePermission(permission)` middleware enforcing permissions per API route.
- Single-use, **SHA-256-hashed** invite tokens with a 48-hour TTL, revocation, and bulk invite creation.
- Environment-flag overrides for domain enforcement and invite requirements (`GOOGLE_ALLOWED_DOMAINS_ENABLED`, `GOOGLE_INVITE_REQUIRED`, `INVITE_ALLOWED_DOMAINS_ENABLED`) enabling controlled rollout.
- Server-side session validation for admin access; configurable session max age.
- Audit logging service capturing actor, action, resource, metadata, IP and user agent.

**Bulk payment resend subsystem** — designed and built from scratch (`bulk-payment-resend.service.ts`, 458 LOC, plus two models and two API routes):
- Two selection modes: explicit payment selection, or "all matching current filters".
- Optional Worldpay **link regeneration** (re-signs and re-shortens before resend).
- Job and per-item state machines; per-item isolation so one failure does not abort the run.
- In-process guard (`runningJobs` Set) preventing duplicate concurrent execution of the same job.
- A 5,000-record cap on filtered selections to bound runaway jobs.
- Completion notification email to the initiating user.

**UCP communications migration** — `ucp.service.ts` and `payment-link-message.service.ts`:
- Typed client for UCP mail and SMS APIs with bearer authentication and error normalisation.
- Database-backed, admin-editable sender configuration (`UcpSetting`), with explicit `MissingUcpSenderConfigError` when unconfigured rather than silent failure.
- Extracted the payment-link email/SMS message bodies into a shared service reused by the detail page, the bulk resend job and the Worldpay callback.
- Removed the deprecated private logger package and replaced it with a local logger helper.

**Daily reporting system** — `daily-report.service.ts` (362 LOC), four API routes, three models:
- Previous-day payment activity report with a generated **CSV attachment** and per-payment communication counts.
- Scheduling via `schedule_time` + `timezone` with next-due calculation, triggered by an API-key-protected `run-due` endpoint.
- Persistent `DailyReportSendLog` recording mode, state, recipients, counts, timing, errors and the raw UCP response.

**Payments dashboard and operations UI** — `src/components/payment/index.tsx` (1,115 LOC) and `src/pages/admin/index.tsx` (1,551 LOC):
- Server-side filtering, searching across five fields, sorting, pagination.
- Payment **link expiry status** (`active` / `expired` / `unknown` / `completed`) as a first-class filter, backed by a new `link_expires_at` column and index.
- CSV export with selectable fields, dynamic date-ranged filenames and a `payment.exported` audit entry.
- KPI summary cards; breadcrumbs; icon set; responsive layout.

**Infrastructure and data migration:**
- Two GitHub Actions workflows (production and staging) building and pushing Docker images to ACR and deploying to Azure App Service, including a migration step that pulls the connection string from the target Web App's own settings rather than from repository secrets.
- `scripts/sync-legacy-payments.ts` (220 LOC) — batched, idempotent legacy→Azure sync with `--since`, `--all`, `--dry-run` and `--batch-size` options.
- Service health check endpoint (`/api/health`) returning status, service, version, timestamp and uptime, with `HEAD` support and no-store caching.
- Seven Prisma migrations authored in 2026 (the repository had two pre-existing from 2023).

### 4.3 Work by others / not individually attributable

The **core payment flow predates this contribution** and was built in 2022–2023: the original Payment and Communication models, the Worldpay hosted-page link construction, the TinyURL integration, the payment response callback, and the original Next.js scaffolding. Abraham **modified** several of these (e.g. 58% of current lines in `payment/new.ts`, including the MD5 signing hardening and link expiry) but did not originate them.

Four 2026 commits are attributed to `zayan_DevOps`; their content was not separately analysed and should not be claimed.

---

## 5. System Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│  BROWSER (internal staff)                                                │
│  Next.js 12 pages · React 18 · Tailwind · Headless UI · Formik           │
│  /payments  /payments/new  /payments/detail/[uuid]  /admin  /auth/signin │
└───────────────┬──────────────────────────────────────────────────────────┘
                │ HTTPS
┌───────────────▼──────────────────────────────────────────────────────────┐
│  NEXT.JS APP (single container, custom server src/server.ts)             │
│                                                                          │
│  ── AUTH ───────────────────────────────────────────────────────────┐    │
│   NextAuth + Google OAuth → signIn callback:                        │    │
│     domain allow-list → invite check → AuthUser lookup → status     │    │
│   Session (JWT, configurable max age)                               │    │
│  ─────────────────────────────────────────────────────────────────── ┘   │
│                                                                          │
│  ── API ROUTES (next-connect + validate + permission middleware) ──┐     │
│   /api/cred/*    session-authenticated payment APIs                │     │
│   /api/admin/*   users · invites · audit · reports · bulk-resend   │     │
│   /api/key/*     API-key protected (machine integrations)          │     │
│   /api/public/*  Worldpay payment response callback                │     │
│   /api/health    liveness/readiness probe                          │     │
│  ──────────────────────────────────────────────────────────────────┘     │
│                                                                          │
│  ── SERVICES ──────────────────────────────────────────────────────┐     │
│   authorization · audit · ucp · payment-link-message               │     │
│   daily-report · bulk-payment-resend · communication               │     │
│  ──────────────────────────────────────────────────────────────────┘     │
└───┬──────────────┬───────────────┬────────────────┬──────────────────────┘
    │ Prisma 4     │ HTTPS         │ HTTPS          │ HTTPS
┌───▼──────────┐ ┌─▼───────────┐ ┌─▼────────────┐ ┌─▼────────────────────┐
│  MySQL       │ │  Worldpay   │ │  UCP         │ │  TinyURL-compatible  │
│  11 models   │ │  hosted     │ │  mail + SMS  │ │  shortener           │
│              │ │  pages      │ │  platform    │ │                      │
└──────────────┘ └──┬──────────┘ └──────────────┘ └──────────────────────┘
                    │ customer pays → POST callback
                    └──────────► /api/public/payment/response
                                 (verifies transStatus + amount,
                                  marks paid, notifies creator)

DEPLOYMENT:  GitHub Actions → docker build → Azure Container Registry
             → prisma migrate deploy → Azure App Service (stage / prod)
```

**Data model (11 Prisma models).** Pre-existing: `Payment`, `Communication`. Added in 2026: `AuthUser`, `Invite`, `AuditLog`, `DailyReportSetting`, `DailyReportRecipient`, `DailyReportSendLog`, `BulkPaymentResendJob`, `BulkPaymentResendJobItem`, `UcpSetting`. Indexing is deliberate — composite indexes on `(state, updated_at)` for both `Payment` and `Communication`, and a dedicated index on `link_expires_at` supporting the expiry filter.

---

## 6. Main Workflows

**Create and send a payment link.** Agent submits name, reference, amount (≤ £5,000), optional email/tel → yup validation client and server side → `requirePermission('payments.create')` → Worldpay parameters assembled including a 7-day `authValidTo` expiry → **MD5 signature computed** over the configured signature fields → URL shortened → `Payment` persisted with `link_expires_at` → audit entry → agent redirected to the detail page → sends link by email and/or SMS via UCP → each attempt recorded as a `Communication`.

**Customer payment and callback.** Customer opens the shortened link → Worldpay hosted page → on completion Worldpay POSTs to `/api/public/payment/response` → handler verifies `transStatus === 'Y'`, resolves the payment by `CM_uuid`, and **re-checks the paid amount against the stored amount** as a second safety layer → marks `paid` → emails a confirmation to the creating agent.

**Bulk resend.** Admin selects payments explicitly or by current filter set → job and item rows created (capped at 5,000) → job started asynchronously → per item: optionally regenerate and re-sign the Worldpay link, then send email and/or SMS via UCP, recording `Communication` rows and per-item outcome → job finalises as `completed` or `completed_with_errors` → completion notification emailed.

**Daily report.** External scheduler calls API-key-protected `/api/admin/reports/daily/run-due` → service checks whether the configured send time is due → builds the previous-day payment dataset with communication counts → renders HTML summary plus CSV attachment → sends via UCP to active recipients → writes a `DailyReportSendLog`.

**Access control.** Google sign-in → domain allow-list (toggleable) → invite requirement (toggleable) → `AuthUser` lookup; disabled users rejected; first admin bootstrapped from `INITIAL_ADMIN_EMAIL` → login and denial both audited.

---

## 7. Technology Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Next.js 12 (pages router), TypeScript 4.7, Tailwind CSS 3, Headless UI 1.7, Formik, react-query, react-paginate, date-fns |
| Backend | Next.js API routes, `next-connect`, custom Node server (`src/server.ts`), yup + `yup-phone` validation |
| Auth | NextAuth 4, Google OAuth provider, app-owned RBAC |
| Data | MySQL, Prisma 4.16 |
| Integrations | Worldpay Business Gateway (hosted pages, MD5 signing), UCP mail/SMS API, TinyURL-compatible shortener |
| Infrastructure | Docker, Azure Container Registry, Azure App Service, GitHub Actions |
| Tooling | Yarn 1.22, ESLint, nodemon, ts-node |

Codebase size at audit: **8,738 lines** across `src/`, plus a 220-line migration script.

---

## 8. Engineering Implementation Notes

Selected details that demonstrate depth (expanded where relevant in §12 and §14):

- **Signature integrity.** `getWorldpaySignature` throws if any configured signature field is missing or empty, rather than silently signing a partial parameter set — a missing `amount` would otherwise yield a valid signature over the wrong data.
- **Fail-closed configuration.** `assertWorldpayMd5Config()` blocks link generation when MD5 settings are absent, returning a validation-shaped 400 so the UI shows a field error instead of the app failing at startup. The same pattern guards UCP senders via `MissingUcpSenderConfigError`.
- **Defence in depth on money.** The amount is MD5-signed outbound *and* re-verified inbound in the callback before a payment is marked paid.
- **Token security.** Invite tokens are stored only as SHA-256 hashes (unique `token_hash`), so database access alone yields no usable invite links.
- **Partial-failure isolation.** Email and SMS outcomes are tracked independently per bulk-resend item, so a valid email with an invalid phone records `email: sent` / `sms: failed` rather than failing wholesale.
- **Idempotent legacy sync.** The migration script upserts by `uuid` and supports `--dry-run` and `--since`, making repeated runs safe and enabling incremental cutover.
- **Deploy-time configuration resolution.** The workflow reads `MYSQL_CONNECTION_STRING` from the target Azure Web App's own settings, so the production connection string never lives in GitHub secrets or the repository.

## 9. Production Evidence and Statistical Analysis

### 9.1 Method and integrity

The live production database (Azure Database for MySQL 8.0, database `appdb`, host `internal-payment-gateway-db-01.*`) was queried **read-only** on 10 October 2026. Every session ran `SET SESSION TRANSACTION READ ONLY` — confirmed by the server returning `@@transaction_read_only = 1`, meaning the database itself would have rejected any write. A wrapper script additionally refused any statement matching write or DDL keywords. Only `SELECT`, aggregate and `SHOW` statements were executed. Connection established over TLS (`--ssl-mode=REQUIRED`).

**Production go-live is precisely datable.** The 2026 migrations were applied to production on **2026-07-17 at 18:26**, and the first `AuthUser` and `AuditLog` rows were written at **19:33** the same evening. The most recent activity at audit time was **2026-10-09 17:39** — the system is in active daily use.

| Migration | Applied in production |
|---|---|
| `add_authorization_tables` | 2026-07-17 18:26 |
| `add_daily_report_settings` | 2026-07-17 18:26 |
| `add_ucp_settings` | 2026-07-17 23:19 |
| `add_daily_report_send_logs` | 2026-07-20 12:12 |
| `add_bulk_payment_resend_jobs` | 2026-07-20 12:12 |
| `add_bulk_resend_link_regeneration_columns` | 2026-07-20 12:12 |
| `add_payment_link_expiry` | 2026-07-28 13:24 |

### 9.2 Whole-estate figures (2023 → present)

| Measure | Value | Notes |
|---|---|---|
| Payment requests | **68,869** | 2023-01-13 → 2026-10-09 |
| Distinct creating users (lifetime) | 185 | `COUNT(DISTINCT created_by)` |
| Completed (paid) | 52,556 (**76.3%**) | `state = 'paid'` |
| Value of completed payments | **£34,954,889** | `SUM(amount) WHERE state='paid'` |
| Total requested value | £47,756,322 | all states |
| Communication records | **131,669** | email + SMS |

**Critical attribution caveat:** this estate is **mostly historical and predates the contribution**. Only 3,308 of the 68,869 payments were created on the 2026 codebase. The £34.95M figure is cumulative since 2023 and must **never** be presented as processed by this contribution. It is reported because the 2026 work now *manages and governs* this estate.

### 9.3 Activity on the 2026 codebase (since go-live, 17 Jul – 9 Oct 2026)

This is the properly attributable production evidence — 84 days of continuous operation.

| Measure | Value | Source / calculation |
|---|---|---|
| Payment requests created | **3,308** | `Payment WHERE created_at >= '2026-07-17'` |
| Distinct creating users | **71** | `COUNT(DISTINCT created_by)` over same window |
| Completed (paid) | **2,611 (78.9%)** | `SUM(state='paid')` |
| Value of completed payments | **£1,792,455** | `SUM(amount) WHERE state='paid'` |
| Total requested value | £2,369,846 | all states |
| Email + SMS delivery attempts | **6,225** (3,361 / 2,864) | `Communication` same window |
| Messages per payment request | 1.88 | 6,216 joined sends ÷ 3,308 payments |
| Mean time, link creation → payment | **3.3 hours** | `AVG(TIMESTAMPDIFF(...))` over 2,611 paid |
| Derived daily averages | ~39 payments/day · ~74 messages/day | ÷ 84 days |

**Monthly trend** (July and October are partial months):

| Month | Payments | Active users | Paid | Paid % | Paid value |
|---|---|---|---|---|---|
| 2026-07 (from 17th) | 591 | 46 | 454 | 76.8% | £303,095 |
| 2026-08 | 1,031 | 57 | 823 | 79.8% | £548,622 |
| 2026-09 | **1,318** | 55 | 1,044 | 79.2% | £745,809 |
| 2026-10 (to 9th) | 368 | 46 | 290 | 78.8% | £194,930 |

Busiest single day: **82 payment requests** (2026-09-01). August → September growth: **+27.8%** in volume. The paid conversion rate held between 76.8% and 79.8% throughout.

**Important qualification on message counts.** `Communication` rows are written with `state = 'sent'` **immediately before** the provider call, and the table contains **no `failed` state at all** (all 131,669 rows are `sent`). These figures therefore represent **delivery attempts logged by the application**, not provider-confirmed deliveries. True delivery rates would have to come from UCP's own reporting (§24).

### 9.4 Authorisation system in production

| Measure | Value |
|---|---|
| Users under RBAC | **71** (68 `user`, 3 `admin`) |
| Active / disabled | 71 active, 0 disabled |
| Users who have signed in | **71 of 71** |
| Successful logins recorded | **906** |
| **Blocked access attempts** (`auth.denied`) | **31**, from 6 distinct accounts |
| Self-registrations via the open-registration toggle | 68 |
| Invites issued / accepted | 2 / 2 |

The 31 `auth.denied` events between 23 Aug and 15 Sep are the clearest single proof that the access-control system does real work in production rather than existing only as code.

### 9.5 Audit trail in production

**8,634 audit events** captured between 2026-07-17 and 2026-10-09:

| Action | Events | Distinct actors |
|---|---|---|
| `payment.created` | 2,707 | 65 |
| `payment.link_emailed` | 2,656 | 64 |
| `payment.link_texted` | 2,253 | 62 |
| `auth.login` | 906 | 71 |
| `user.self_registered` | 68 | 68 |
| `auth.denied` | 31 | 6 |
| `payment.exported` | 7 | 1 |
| `user.invited` / `user.invite_accepted` | 2 / 2 | 1 / 2 |
| `payment.bulk_resend_started` | 1 | 1 |
| `user.updated` | 1 | 1 |

*Note:* `payment.created` audit events (2,707) are fewer than payments created since go-live (3,308) because payment-level audit logging began on 2026-08-02, two weeks after the initial release.

### 9.6 Daily reporting reliability

| Measure | Value |
|---|---|
| Reports sent | **81** |
| Reports failed | **0** |
| Success rate | **100%** |
| Distinct days covered | **81** (2026-07-20 → 2026-10-08) |
| Mode | `automatic` (scheduled, not manual) |
| Recipient deliveries | 223 |
| Payment records reported | 2,819 |

81 reports across 81 distinct consecutive days with zero failures — **the scheduled job has not missed a single day since launch.** This is the strongest reliability evidence in the project.

### 9.7 Bulk resend in production

One production job, executed 2026-08-02:

| Measure | Value |
|---|---|
| Selection | `filtered`, with link regeneration enabled |
| Payments processed | **70 of 70** |
| Worldpay links regenerated and re-signed | **70** |
| Emails sent | 70 |
| SMS sent | 65 |
| SMS skipped (no mobile number held) | **5** |
| Failed | **0** |
| Duration | **123 seconds** |

This validates the partial-failure design precisely as intended: 65 items completed both channels, and 5 recorded `email: sent` / `sms: skipped` because no phone number existed — a clean, legible outcome rather than an opaque failure.

### 9.8 Legacy migration and the operational backlog

**65,561 payments and 125,444 communications** dated before go-live are present in the Azure production database, covering 2023-01-13 onward. The historical estate was migrated into Azure as part of this phase; `scripts/sync-legacy-payments.ts` (authored by Abraham) is the documented tool for it. *Evidence level: the data's presence in Azure is VERIFIED; that all 65,561 rows passed through this specific script rather than partly via a database restore is **STRONGLY SUPPORTED** but not provable from the database alone (§24).*

The backlog the expiry-tracking and bulk-resend features target is real and large: of 16,313 pending payments, **16,257 have expired links** and only 56 remain active. This quantifies the operational problem those features address.

### 9.9 Exposure surface closed by the signing fix

| Measure | Value |
|---|---|
| Payment links issued **before** the fix (unsigned, editable amount) | **65,093** |
| Total value represented by those links | **£45,049,376** |
| Period of exposure | 2023-01-13 → 2026-07-04 |
| Fix deployed | commit `af467f9`, **2026-07-05** |

*Calculation:* `COUNT(*)` and `SUM(amount)` over `Payment` where `created_at < '2026-07-05'`.

**Essential qualification:** this is the **exposure surface** — every link generated in that period carried an editable amount parameter. It is **not** a loss figure. Nothing in the available data establishes how many links were actually tampered with, because a tampered payment was rejected by the callback and left no record of the amount actually charged. Any claim must say "closed a vulnerability affecting £45M of issued payment links", never "recovered" or "prevented the loss of" £45M.

## 10. Scale, Usage and Reliability

**Observed production scale** (see §9 for derivations): ~39 payment requests and ~74 outbound messages per day sustained over 84 days, peaking at 82 requests in a day, across 71 staff users, against a managed estate of 68,869 payment records and 131,669 communications.

**Observed reliability:**
- Daily report: **81/81 successful sends, 0 failures, no missed day.**
- Bulk resend: **70/70 items processed, 0 failures**, 123 seconds (≈1.8s per item including link re-signing and two channel sends).
- No `failed` state exists anywhere in `Communication` — but see the §9.3 caveat; this reflects how the record is written, not a measured delivery success rate.
- Payment conversion stable at 76.8–79.8% month over month.

**Designed capacity limits:** bulk resend caps at 5,000 payments per job (the production job used 70, ~1.4% of the cap); CSV export is bounded by `MAX_EXPORT_ROWS`; payment queries are paginated and served by composite indexes on `(state, updated_at)` plus a dedicated `link_expires_at` index — the latter now supporting filtering across 68,869 rows.

## 11. Business Impact

**Governance and control — now quantified.** The application moved from domain-based implicit access to explicit role-based access covering **71 staff users** (68 standard, 3 admin), with **906 logins** and **31 blocked access attempts** recorded. **8,634 audit events** now answer who created, sent, exported or bulk-resent what, and when — for a tool that has generated **£47.76M** in payment requests across its lifetime. Before this work, none of that was recorded.

**Operational capability — demonstrated in production.** Bulk resend with link regeneration remediated **70 expired payment links in 123 seconds with zero failures**, work that previously had no mechanism at all and would otherwise be done one request at a time. The scale of the latent problem is **16,257 pending payments with expired links**.

**Operational visibility — proven reliable.** The automated daily report has delivered **81 of 81 reports across 81 consecutive days with no failures and no missed day**, covering 2,819 payment records to 5 recipients.

**Platform consolidation — carrying real volume.** The UCP integration has carried **6,225 email and SMS delivery attempts in 84 days** (~74/day), with sender configuration now changeable by an administrator without a redeployment.

**Sustained adoption and growth.** Monthly volume rose from 1,031 payments in August to **1,318 in September (+27.8%)**, with 46–57 distinct users active each month and paid conversion steady at ~79%. The system processed **£1,792,455** of completed payments on the new codebase in its first 84 days.

**Deployment repeatability.** Nine migrations were applied cleanly to production through the automated pipeline between 17 and 28 July, with no production credentials present in the repository or CI configuration.

**Still not established:** staff hours saved, cost impact, and whether payment conversion changed *as a result of* this work (the 76–80% rate has no pre-go-live baseline computed here). See §24.

## 12. Significant Engineering Challenges

**Retrofitting authorisation into a live authenticated application.** The hard part was not defining roles but inserting enforcement without locking out existing users mid-rollout. The design bootstraps a first admin from `INITIAL_ADMIN_EMAIL`, and puts three independent behaviours behind environment flags (`GOOGLE_ALLOWED_DOMAINS_ENABLED`, `GOOGLE_INVITE_REQUIRED`, `INVITE_ALLOWED_DOMAINS_ENABLED`) so domain checks and invite requirements can be relaxed or tightened independently at runtime. That is a deliberate migration-safety design, not an afterthought.

**Bulk operations over a payment provider with partial failure.** Each item involves up to three external calls (Worldpay link re-signing, TinyURL shortening, UCP send) across two channels. Any can fail independently. The service models link state, email state and SMS state separately per item, aggregates counters on the parent job, and distinguishes `completed` from `completed_with_errors` — so an operator can see precisely which of 500 customers did not receive a message and why.

**Signing correctness with an undocumented legacy payment integration** *(the hardest problem in the project)*. Worldpay hosted pages carry payment parameters in the URL, so an unsigned link lets the customer choose what they pay — which is exactly what was happening in production across 65,093 issued links. Fixing it required determining the precise MD5 scheme the Worldpay installation expected: which fields, in which order, concatenated with the secret in which form. **No documentation for the integration survived**, and the failure mode is unforgiving in both directions — get it wrong and either every link breaks in production, or worse, the signature covers the wrong fields and appears to work while protecting nothing. The implementation derives the field list from configuration (so it can be matched to the installation without redeploying code), refuses to sign when any field is missing or empty, fails closed when configuration is absent, and leaves the independent callback amount check in place as a second layer. Previous attempts by other developers had reportedly not succeeded; this one shipped within roughly ten days of joining.

**Working within Next.js 12 constraints.** No App Router, no route handlers, no built-in background jobs. Background processing is implemented with an in-process deferred execution pattern and a job table for durability — a pragmatic choice for the deployment model, with known limits (§13).

**Legacy data migration without downtime.** The sync script had to be re-runnable, incremental, and verifiable before committing — hence upsert-by-uuid, `--since`, batching, and `--dry-run`.

---

## 13. Architectural Decisions and Trade-offs

**Application-owned authorisation rather than Google Workspace groups or an external IdP role system.**
*Problem:* needed roles, invites and disablement tied to app-specific permissions.
*Decision:* store `AuthUser`, `Invite`, `AuditLog` in the application database; keep Google purely for identity.
*Reasoning:* permissions are domain-specific (`payments.export`, `payments.send`) and would not map cleanly onto directory groups; it also keeps the audit trail adjacent to the payment data it describes.
*Trade-off:* a second user store to maintain, and offboarding is not automatic when a Google account is disabled.
*Result:* granular per-route permission enforcement; `auth.denied` events in the local database confirm the path is exercised.

**In-process asynchronous job execution rather than a queue or worker service.**
*Problem:* bulk resend can involve thousands of external calls and cannot run inside a request.
*Decision:* persist job and item rows, then trigger processing via deferred in-process execution guarded by a `runningJobs` Set.
*Reasoning:* avoids introducing Redis/a queue broker/a separate worker deployment for a single internal workflow on a single App Service.
*Trade-off (explicitly acknowledged):* the duplicate-run guard is per-process, so it would not prevent concurrent processing across multiple instances, and an instance restart mid-job leaves items `pending`. The item-level state table makes recovery *possible* but no automatic resumption is implemented.
*Result:* working bulk processing with no new infrastructure; the state model leaves a clean upgrade path to a real queue.

**Database-backed sender configuration rather than environment variables.**
*Problem:* UCP sender addresses change and must match senders pre-approved in UCP.
*Decision:* store in `UcpSetting`, editable under Admin → Settings.
*Reasoning:* changing a sender should not require a redeploy.
*Trade-off:* configuration split between env vars and database.
*Result:* misconfiguration surfaces as an explicit, actionable error rather than a silent delivery failure.

**Pull the production connection string from Azure at deploy time.**
*Problem:* migrations need production credentials.
*Decision:* the workflow queries the target Web App's app settings via `az webapp config appsettings list`.
*Reasoning:* avoids duplicating the production connection string into GitHub secrets; single source of truth.
*Trade-off:* the pipeline needs a service principal with configuration read rights.
*Result:* no database credentials in the repository or CI configuration — corroborated by this audit finding only local strings in `.env`.

**Keeping the Next.js 12 pages router rather than upgrading.**
*Decision:* modernise within the existing framework version.
*Reasoning:* the goal was security and operational capability, not a framework migration; an upgrade would have risked the payment flow.
*Trade-off:* no App Router features; legacy `next/link` semantics requiring child `<a>` elements.
*Result:* substantial capability added with the core payment path untouched.

---

## 14. Problems Solved

### 14.1 The originating problem — unsigned, editable payment links *(flagship)*

**Situation.** The platform had generated Worldpay hosted-payment links since January 2023. The engagement began specifically because customers were able to alter the amount they paid.

**Problem.** `getPaymentLink` built the payment URL as a plain query string containing `amount`, with **no signature of any kind**. A recipient could rewrite `&amount=500` to `&amount=5` and complete a genuine Worldpay checkout for the lower sum. A 2022 callback check then refused to mark the payment paid because the returned amount did not match the stored one — so the money was taken at the provider but never recorded, leaving unreconcilable receipts and permanently `pending` requests. Customer confidence in the links suffered, compounded by unbranded emails that recipients found untrustworthy.

**Investigation.** The legacy Worldpay integration had **no surviving documentation**, and previous attempts by other developers had not succeeded (user-reported). The problem was not recognising that signing was needed, but determining *exactly* what Worldpay's hosted-page MD5 scheme required: which fields, in which order, concatenated with the shared secret in which form — where any error silently breaks every link in production rather than failing loudly.

**Solution.** Implemented MD5 request signing (commit `af467f9`, 2026-07-05):
- `getWorldpaySignature` builds the digest from a **configurable, ordered field list** (`WP_SIGNATURE_FIELDS`, default `instId:amount:currency:cartId`) joined with the shared secret, so the app can be aligned to whatever the Worldpay installation is configured to verify without a code change.
- It **throws if any signature field is missing or empty**, rather than signing a partial parameter set — a subtle but critical detail, since a missing `amount` would otherwise produce a technically valid signature over the wrong data.
- `assertWorldpayMd5Config()` makes the system **fail closed**: if signing configuration is absent the app refuses to generate links at all, returning a validation-shaped error the UI surfaces as a field message rather than failing at startup.
- The pre-existing callback amount check was deliberately **retained as a second, independent layer** — defence in depth rather than replacement.

**Result.** A tampered URL no longer reaches a payable checkout: Worldpay rejects it because the signature no longer matches the altered parameters. The fix was delivered as his **first substantive commit, roughly ten days after joining the project**, and closed an exposure surface of **65,093 previously-issued links worth £45.0M** (§9.9). The engineer reports the CTO commended the fix, noting that previous developers had attempted it without success *(user-reported; not independently verifiable)*.

**Evidence.**
- Pre-fix source at `af467f9^` shows `getPaymentLink` with no signature — the vulnerability is directly demonstrable.
- `git blame` attributes **all 36 lines** of the signing implementation to `af467f9` (AbrahamNobleOX, 2026-07-05).
- The callback check originates in `ef1ff7f` (Michal Tesar, 2022-12-23), confirming it pre-existed and was retained, not authored, by him.
- Production: 65,093 links / £45,049,376 issued before the fix date.
- `AGENTS.md` and `CLAUDE.md` both record the signing as a non-removable constraint for future work.

### 14.2 Other problems solved

**Payment links expiring unnoticed.****Payment links expiring unnoticed.**
*Situation:* Worldpay links carry a 7-day validity window; pending payments accumulate.
*Problem:* no way to see which links were still valid, and no way to refresh them in bulk.
*Solution:* added `link_expires_at` to `Payment` with a dedicated index, derived a four-state link status (`active`/`expired`/`unknown`/`completed`), exposed it as a dashboard filter, and built bulk resend with optional regeneration so expired links are re-signed before being resent.
*Result:* expired links became visible and remediable as a batch operation.
*Evidence:* migration `20260728133000_add_payment_link_expiry`; commit `32b035d`; `EPaymentLinkStatus` logic in `bulk-payment-resend.service.ts` and the list API.

**Silent delivery failures after the UCP migration.**
*Problem:* UCP requires senders to be pre-registered; an unregistered sender fails at the provider.
*Solution:* `getUcpMailConfig` / `getUcpSmsSender` throw a typed `MissingUcpSenderConfigError` with remediation text naming the admin screen, surfaced to the operator instead of logged and swallowed.
*Result:* configuration errors became self-describing.
*Evidence:* `ucp.service.ts`; commit `ad29cf6`.

**Unsigned payment links allowing amount tampering (hardening).**
*Problem:* hosted-page parameters are client-visible and alterable.
*Solution:* MD5 signing over configured fields, refusal to sign on missing fields, fail-closed on absent configuration, plus independent amount verification in the callback.
*Result:* defence in depth on the financially sensitive field.
*Evidence:* `payment/new.ts`; `public/payment/response.ts`; documented as a non-removable constraint in `AGENTS.md` and `CLAUDE.md`.

**Full page reloads between screens (front-end performance).**
*Situation:* navigating the dashboard reloaded the whole document.
*Investigation:* several links were plain `<a href>` elements, bypassing the Next.js client-side router.
*Solution:* converted navigation, the table's row links and dropdown items to `next/link`.
*Result:* client-side transitions with prefetching, removing full-document reloads.
*Evidence:* commit `d825ab5`.

**Validation errors arriving as HTTP 400s with no client-side guard.**
*Problem:* empty form submissions produced server round-trips and console errors, because the form had no `validationSchema`.
*Solution:* added client-side yup schemas mirroring the server rules plus a `mergeFieldErrors` helper reconciling client and server errors, while deliberately retaining server validation as authoritative for all callers.
*Result:* immediate feedback; 400s reserved for genuine anomalies.
*Evidence:* commit `d825ab5`; `src/utils/react.ts`.

---

## 15. Security, Reliability and Operational Considerations

- **Four distinct API trust zones:** session-authenticated (`/api/cred`), permission-gated admin (`/api/admin`), API-key machine access (`/api/key`, and the report scheduler), and public (Worldpay callback only).
- **Secrets never in the repository** — confirmed by this audit; production configuration resides in Azure App Service settings.
- **Invite tokens stored hashed**; 48-hour expiry; single use; revocable.
- **Audit trail** capturing actor, action, resource, metadata, IP and user agent.
- **Fail-closed** behaviour on both Worldpay MD5 and UCP sender configuration.
- **Callback validation** on transaction status and amount before state change.
- **Health endpoint** for platform probes.
- **Known gaps (stated honestly):** automated test coverage is effectively absent — the three `.test.ts` files are empty placeholders inherited from the original project. There is no retry/backoff on UCP calls, and no automatic resumption of interrupted bulk jobs. These are real limitations, not oversights to hide.

---

## 16. Lessons Learned

- **Retrofitting access control is a migration problem, not a modelling problem.** The permission table was straightforward; the valuable design work was the bootstrap admin and the three independent feature flags that allowed enforcement to be switched on without stranding users.
- **Model partial failure explicitly when fanning out to external providers.** Separate per-channel state fields turned "the bulk job failed" into "these three customers' SMS failed for this reason," which is the difference between an operator being stuck and being able to act.
- **Fail closed and say why.** Both the Worldpay and UCP configuration paths raise typed errors with remediation text. This consistently converted silent misconfiguration into a visible, fixable message.
- **Configuration belongs where its change cadence lives.** Secrets and endpoints in environment variables; operationally-tuned values (senders, report schedule, recipients) in the database behind an admin screen.
- **An in-process job runner is a legitimate interim choice, but only if the state is durable.** Persisting job items made the shortcut defensible and left a clear migration path to a queue.
- **Undocumented legacy integrations are a research problem before they are a coding problem.** The signing fix was perhaps forty lines of code; nearly all the risk sat in establishing *which* fields the installation verified and in what order, with no documentation and a failure mode that can look like success. Designing it to fail closed, and to refuse partial signatures, was the safeguard against being wrong.
- **Fixing the real problem earns the mandate for the rest.** The engagement began as a single security fix and expanded into access control, auditing, reporting and bulk remediation. Credibility from the first fix is what made the wider modernisation possible.
- **What I would do differently:** add automated tests around the signing logic, permission middleware and bulk-resend state transitions — the highest-risk code in the system is currently untested; add retry with backoff on UCP calls; and add job resumption after restart.

---

## 17. Verified Career Claim Bank

> Evidence levels are stated per claim. Supporting detail is cross-referenced to §9 rather than repeated.

**C1 — Identified and fixed a parameter-tampering vulnerability in a live payment platform that allowed customers to alter the amount they paid, implementing MD5 request signing as the first substantive commit, roughly ten days after joining.**
*Level:* **VERIFIED.** *Evidence:* pre-fix source at `af467f9^` shows the payment URL built with no signature; all 36 lines of the signing implementation attributed to `af467f9` (2026-07-05) by `git blame`; §14.1. *Qualification:* the pre-existing callback amount check (2022, another author) was retained as a second layer, not authored by him.

**C1a — Closed an exposure surface of 65,093 previously-issued payment links representing £45,049,376 of transaction value.**
*Level:* VERIFIED as exposure measurement (§9.9). *Qualification — critical:* this is the value of links issued under the unsigned scheme, **not** money lost or recovered. No evidence establishes how many were tampered with. Phrase as "closed a vulnerability affecting £45M of issued payment links"; never "saved" or "prevented the loss of".

**C1b — Solved a payment-integration problem that previous developers had attempted unsuccessfully, working without any surviving documentation for the legacy integration.**
*Level:* **STRONGLY SUPPORTED.** *Evidence:* absence of integration documentation is consistent with the repository state; the prior-attempts detail and the CTO's commendation are the engineer's direct account, not independently verifiable. Use in interview narrative; confirm before stating as fact on a CV.

**C1c — Delivered a six-week modernisation of a payment-operations platform running in production continuously since 17 July 2026, serving 71 staff users.**
*Level:* VERIFIED. *Evidence:* 47 commits; production migrations 2026-07-17; activity through 2026-10-09. *Qualification:* modernisation of an existing 2022-built application, not greenfield.

**C2 — Contributed 47 commits and +9,029 lines across 103 files, authoring 84.5% of the application's surviving source.**
*Level:* VERIFIED (§4.1). *Qualification:* `git blame` favours recent contributors; the original payment flow was authored by others.

**C3 — Designed and shipped a role-based access control and audit system now governing 71 production users, which has recorded 906 logins, blocked 31 unauthorised access attempts and captured 8,634 audit events in its first 84 days.**
*Level:* VERIFIED (§9.4, §9.5). *Qualification:* replaced domain-based implicit access; identity remains Google SSO.

**C4 — Built a bulk payment-link resend engine that regenerated and re-signed 70 Worldpay links and dispatched 135 messages across two channels in 123 seconds with zero failures.**
*Level:* VERIFIED (§9.7). *Qualification:* one production job to date, at ~1.4% of the engine's 5,000-record design cap.

**C5 — Migrated all transactional email and SMS onto the company's UCP platform; the integration has carried 6,225 production delivery attempts (~74/day) over 84 days.**
*Level:* VERIFIED (§9.3). *Qualification:* counts are application-logged send attempts, not provider-confirmed deliveries.

**C6 — Built an automated daily reporting system that has delivered 81 of 81 scheduled reports across 81 consecutive days with a 100% success rate and no missed day.**
*Level:* VERIFIED (§9.6). *The single strongest reliability claim in the project.*

**C7 — Introduced payment-link expiry tracking across a 68,869-record estate, surfacing a previously invisible backlog of 16,257 pending payments with expired links.**
*Level:* VERIFIED (§9.8). *Qualification:* the backlog's existence is measured; its prior business cost is not.

**C8 — Delivered a payment platform that processed £1,792,455 in completed customer payments in its first 84 days of production, at a 78.9% completion rate across 3,308 requests.**
*Level:* VERIFIED (§9.3). *Qualification:* the platform's lifetime estate is £34.95M paid since 2023, the large majority of which predates this contribution and must not be claimed as processed by it.

**C9 — Migrated a historical estate of 65,561 payment records and 125,444 communication records into Azure as part of a cloud database migration.**
*Level:* VERIFIED (data present in Azure production) / **STRONGLY SUPPORTED** (that it passed through the sync script authored for the purpose, rather than partly via a database restore). *Evidence:* §9.8; `scripts/sync-legacy-payments.ts`.

**C10 — Created Dockerised CI/CD pipelines to Azure Container Registry and App Service; nine database migrations were applied cleanly to production through them.**
*Level:* VERIFIED (§9.1, workflow definitions). *Qualification:* workflow runs themselves were not inspected.

**C11 — Hardened a Worldpay hosted-payment integration with MD5 request signing, fail-closed configuration checks and independent server-side amount verification.**
*Level:* VERIFIED (implementation) / **STRONGLY SUPPORTED** (personal authorship — 58% of lines in `payment/new.ts`, including the fail-closed path, but the file predates the contribution).

**C12 — Authored 7 production database migrations introducing 9 models, with composite indexing supporting dashboard queries over a 68,869-row payment table.**
*Level:* VERIFIED (§9.1).

**C13 — Rebuilt the operations dashboard (server-side filtering, multi-field search, sorting, pagination, link-expiry filtering, CSV export, KPIs) now used daily by 46–57 distinct staff per month.**
*Level:* VERIFIED (§9.3 monthly active users).

**C14 — Supported sustained production growth of +27.8% in monthly payment volume (1,031 → 1,318, Aug → Sep 2026) with stable ~79% completion.**
*Level:* VERIFIED as measurement (§9.3). *Qualification:* this is business growth the platform carried; **not** growth caused by the engineering work.

---

## 18. CV Achievement Candidates

### High-confidence quantified

- **Identified and remediated a parameter-tampering vulnerability in a live payment platform** that allowed customers to alter the amount payable on any generated payment link, implementing MD5 request signing with fail-closed configuration — shipped as the first substantive commit **~10 days after joining**, closing an exposure surface of **65,093 issued links representing £45M of transaction value**.
- Modernised and now own the production codebase of an internal payment platform **serving 71 staff users and managing 68,869 payment records worth £47.8M**, authoring **84.5%** of its surviving source across 47 commits in a six-week engagement.
- Designed and shipped **role-based access control and audit logging** for a financial payment-link platform, governing **71 users** and capturing **8,634 audit events**, **906 logins** and **31 blocked access attempts** in the first 84 days of production.
- Built an **automated daily reporting service** achieving **81/81 successful scheduled deliveries across 81 consecutive days — 100% reliability, zero missed days**.
- Engineered a **bulk payment-link resend subsystem** that regenerated and re-signed **70 Worldpay payment links and dispatched 135 messages in 123 seconds with zero failures**, against a design capacity of 5,000 requests per job.
- Migrated all transactional messaging to a central communications platform, **carrying 6,225 email and SMS delivery attempts (~74/day) across 84 days in production**.
- Delivered a platform release that **processed £1.79M of completed customer payments at a 78.9% completion rate across 3,308 requests** in its first 84 days.
- Introduced **payment-link expiry tracking across 68,869 records**, exposing a previously invisible operational backlog of **16,257 expired pending payment links**.
- Led a **cloud database migration of 65,561 payment and 125,444 communication records** into Azure Database for MySQL, and authored **7 schema migrations introducing 9 data models** applied cleanly to production.

### High-confidence non-quantified

- Hardened a legacy Worldpay hosted-payment integration with MD5 request signing, fail-closed configuration validation and independent server-side amount verification on the payment callback.
- Built Dockerised GitHub Actions pipelines deploying to Azure App Service via Azure Container Registry for staging and production, resolving database configuration from the target environment so no production credentials exist in source control or CI secrets.
- Designed an idempotent, batched data-migration tool supporting incremental, full and dry-run execution.
- Eliminated full-page reloads across the application by migrating to client-side routing, and added mirrored client-side validation without weakening server-side enforcement.

### Requires confirmation before use

- **Delivery success rates.** Message counts are application-logged attempts; the `Communication` table records no failures by design. Confirm true delivery rates from UCP reporting before claiming them.
- **Causal impact on payment conversion — not yet assessable.** Do not use the lifetime-average comparison (post-release 78.9% vs lifetime 76.3%); the lifetime figure is depressed by a 2025 dip that had already recovered before the release, so that ~2.6 pp reading is an artefact and would not survive scrutiny. A proper baseline and re-check method are held in the separate working note (see §25); revisit once more post-release data has accumulated.
- **"Sole engineer on the 2026 phase."** Four 2026 commits are attributed to `zayan_DevOps`.
- **Attribution of the full £34.95M lifetime value.** Only £1.79M arose on the new codebase; the rest predates the contribution.
- **Time or cost saved.** No baseline exists for manual effort displaced (§24).

## 19. Portfolio Case Study Material

**Overview.** iRevolution Payments lets insurance staff generate a signed Worldpay payment link for a specific customer, amount and reference, deliver it by email or SMS, and track it to completion.

**Context and challenge.** The application had run since 2022 but had accumulated control and operational gaps: anyone with a company Google account could create payment links; there was no audit trail; delivery ran through a legacy mail path; expired links could only be fixed one at a time; and there was no automated reporting or repeatable deployment. The binding constraint: none of the payment or Worldpay behaviour could change — this is live financial infrastructure.

**Role.** Led and implemented the 2026 modernisation end to end — data modelling, backend services, API design, front-end rebuild, infrastructure, and data migration (§4).

**Solution, architecture and workflows.** See §5 (component diagram, 11 models, four API trust zones, three external integrations) and §6.

**Interesting engineering problems.** Retrofitting authorisation without locking out live users; modelling partial failure across a two-channel bulk fan-out; signing correctness against a legacy hosted-payment page; background jobs inside Next.js 12 without a queue — §12 and §13.

**Evidence and results.** 84 days of continuous production operation: 3,308 payment requests worth £1.79M completed, 6,225 messages delivered, 71 users under RBAC, 8,634 audit events, 81/81 daily reports with zero failures, and a 70-link bulk remediation run completed in 123 seconds (§9, §11).

**What I would improve next.** Test coverage on signing, permissions and job state transitions; retry with backoff on provider calls; job resumption after restart; a real queue if volume grows (§16).

## 20. Interview Story Bank

**S0 — The payment links customers could rewrite** *(flagship: security, ownership, working without documentation)*
*Situation:* I was brought into a four-year-old internal payment platform for one reason — customers were paying less than they were invoiced. The app generated Worldpay hosted-payment links, and the amount travelled in the URL as a plain query parameter. Someone sent a link for £500 could edit it to £5 and complete a genuine checkout.
*Task:* Close it in a live financial system without breaking the payment flow the business ran on daily.
*Action:* I traced the whole failure path first, because the symptom was strange. A 2022 callback check compared Worldpay's reported amount against the stored one and refused to mark mismatched payments paid — so the money *was* taken at the provider but never recorded. That's why it surfaced as an accounting reconciliation mess rather than clean underpayment. The fix was MD5 request signing, but the difficulty was that **no documentation for the legacy integration survived**, and the scheme is unforgiving: the signature must cover exactly the fields the Worldpay installation checks, in order. Get it wrong and either every link in production breaks, or — far worse — it covers the wrong fields and looks like it works while protecting nothing. So I made the field list configuration-driven, so it could be aligned to the installation without a code change; made the signer **throw if any signature field is missing** rather than sign a partial set; made the app **fail closed** and refuse to generate links at all when signing config is absent; and deliberately **kept the existing callback check** as an independent second layer.
*Result:* Shipped as my first substantive commit, about ten days after joining. A tampered URL no longer reaches a payable checkout. It closed an exposure surface of 65,093 previously-issued links worth £45M. Previous developers had reportedly attempted this without success, and the CTO commended the fix. The engagement then expanded into the wider modernisation — access control, auditing, reporting and bulk remediation.
*Evidence:* the pre-fix source has no signature at all; `git blame` attributes all 36 lines of the signing implementation to my 2026-07-05 commit; production data sizes the pre-fix exposure.
*If asked what I'd do differently:* write tests around the signing logic — it is still the highest-risk untested code in the system.

**S1 — Retrofitting access control into a live financial tool (ownership, migration safety).**
*Situation:* an internal tool generating customer payment links allowed any company Google account full access, with no audit trail.
*Task:* introduce roles, invites and disablement without locking out active users or disrupting payment operations.
*Action:* designed an app-owned authorisation model (3 roles, 7 permissions) with hashed single-use invite tokens and a 48-hour TTL; added permission middleware per route; bootstrapped a first admin from configuration; and put domain enforcement and the invite requirement behind independent environment flags so enforcement could be tightened progressively.
*Result:* live since 17 July 2026 across **71 users**, with **906 logins**, **8,634 audit events** and **31 blocked access attempts** recorded — the denials being the proof it does real work rather than existing only as code.
*Evidence:* §9.4, §9.5; migration `20260716152000`.

**S2 — Designing for partial failure in a bulk external fan-out (systems design).**
*Situation:* operators needed to resend hundreds of pending payment links at once.
*Task:* build bulk processing where each item makes up to three external calls across two channels, any of which can fail.
*Action:* modelled a parent job and per-item rows with independent link/email/SMS state; isolated failures per item; aggregated counters; distinguished `completed` from `completed_with_errors`; capped jobs at 5,000 and guarded against duplicate concurrent runs.
*Result:* operators can see exactly which recipients failed and why, instead of a single opaque failure.
*Result in production:* the single live job regenerated 70 Worldpay links and sent 135 messages in 123 seconds with **zero failures**; 5 items correctly recorded `email: sent` / `sms: skipped` because no mobile number was held — exactly the legible partial outcome the design exists to produce.
*Evidence:* `bulk-payment-resend.service.ts`; §9.7.

**S3 — A deliberate shortcut, and knowing its limits (trade-offs, technical honesty).**
*Situation:* bulk jobs needed background execution; the stack was Next.js 12 on a single App Service with no queue infrastructure.
*Action:* chose durable job/item tables plus in-process deferred execution with a duplicate-run guard, rather than introducing a broker and worker deployment.
*Result:* delivered the capability with no new infrastructure.
*Reflection (the substance of the story):* the guard is per-process and would not hold across multiple instances, and a restart mid-job leaves items pending with no automatic resumption. The durable item state was the deliberate hedge that keeps a queue migration cheap.
*Evidence:* §13.

**S4 — Debugging full-page reloads (investigation, front-end performance).**
*Situation:* every navigation reloaded the entire application.
*Task:* diagnose; a proposal to adopt React Router was on the table.
*Action:* identified that several links were plain anchors bypassing the Next.js router, and that React Router is incompatible with the pages router in any case; converted navigation, table row links and menu items to `next/link`.
*Result:* client-side transitions with prefetching; the correct fix was a handful of components, not a routing library.
*Evidence:* commit `d825ab5`.

**S5 — Reading a 400 correctly before "fixing" it (diagnostic discipline).**
*Situation:* submitting an empty form produced `400 Bad Request` errors in the console.
*Action:* traced the flow and established the 400 *was* the validation mechanism — the server schema returned per-field errors the UI rendered. Rather than suppressing it, added mirrored client-side validation for instant feedback while deliberately retaining server validation as authoritative for all API callers.
*Result:* immediate feedback; 400s reserved for genuine anomalies; no weakening of the security boundary.
*Evidence:* §14.

**S6 — Securing a legacy hosted-payment integration (security thinking).**
*Situation:* Worldpay hosted pages carry the amount in a client-visible URL.
*Action:* MD5 signing over configured fields; refusal to sign when any field is missing; fail-closed when signing configuration is absent; independent amount re-verification in the callback.
*Result:* defence in depth on the financially sensitive field.
*Evidence:* §8, §14.

---

## 21. Technical Article / Blog Opportunities

**A1 — "Retrofitting RBAC into a live application without locking anyone out."** The modelling is easy; the rollout is the engineering. Covers bootstrap admins, independent feature flags per enforcement rule, and check ordering in the sign-in callback. *Anonymise:* company, domains, admin email. *Do not publish:* production flag names and defaults.

**A2 — "Partial failure is a product feature: bulk operations that touch external providers."** Argues for per-item, per-channel state as the difference between an actionable and an opaque system; covers job/item schema design and counter aggregation. *Keep providers generic.* *Do not publish:* internal API shapes or sender configuration.

**A3 — "You probably don't need a queue yet — but you do need durable job state."** An honest treatment of a pragmatic shortcut, its failure modes (multi-instance duplication, restart mid-job), and the schema choice that keeps the upgrade cheap. *Do not publish:* deployment topology.

**A4 — "The payment link your customer can rewrite"** *(flagship article)*. The strongest story in the project: hosted payment pages put transaction parameters in the URL, so an unsigned link lets the customer choose the price. Covers why the failure mode was *worse* than underpayment (a downstream amount check rejected the callback, so money was taken but never recorded — an accounting problem, not a clean error), why a partially-populated signature set is more dangerous than no signature at all, and why signing configuration should fail closed. Genuinely educational: this class of flaw exists in any legacy hosted-payment integration and is widely under-appreciated. *Must anonymise:* employer, merchant/installation identifiers, and all monetary figures. *Must NOT publish:* the signature field list or order, secret handling specifics, live endpoint configuration, or anything that would help someone identify and attack a still-unpatched installation elsewhere. **Write this as a defensive pattern piece, never as a reproduction recipe — and have it reviewed before publication, since it describes a real vulnerability in a named employer's payment flow.**

**A5 — "That 400 in your console might be your validation working."** Short practical piece on client/server validation as complementary layers, with a concrete error-merging approach. *Safe to publish broadly.*

## 22. LinkedIn / Social Content Opportunities

Each is grounded in a specific, verifiable event from this project.

1. **"The fix was three components, not a new library."** The full-page-reload investigation (S4) — a short post on diagnosing before adopting. *Credible because:* a concrete wrong turn (React Router) was considered and rejected on evidence.
2. **"We shipped a background job system without a queue — and wrote down exactly when that stops working."** (S3) *Credible because:* it names the real limitations rather than presenting the shortcut as a triumph.
3. **"Adding permissions to a live tool is a migration problem."** (S1) *Credible because:* the feature-flag rollout design is visible in the code.
4. **"Partial failure should be visible."** (S2) — why "which three of these 500 failed, and why" is a design requirement.
5. **"A 400 isn't always a bug."** (S5) — short, useful, broadly relatable.
6. **Retrospective: "Six weeks on a four-year-old codebase."** What modernising an inherited application teaches about scope discipline — the payment path was deliberately left untouched throughout. *Credible because:* the constraint is documented in the project's own engineering notes.
7. **"Configuration belongs where its change cadence lives."** Secrets in environment variables; operational values in the database behind an admin screen (§13).

8. **"81 days, 81 reports, zero failures."** A short post on what it takes for a boring scheduled job to simply never miss — and why that is a real engineering outcome. *Credible because:* §9.6 measures it directly.

9. **"I was brought in to fix one bug. It turned into six weeks."** The arc from a single security fix to a full modernisation — a post about how fixing the real problem earns the mandate to fix the surrounding ones. *Credible because:* the git history shows exactly that progression, starting with the signing commit. **Caution:** describe the flaw generically ("a parameter-tampering issue in a payment integration"); do not name the employer alongside it, and omit monetary figures.

*Before posting any figure:* use only the post-go-live numbers (§9.3), never the £34.95M lifetime estate, and describe message counts as send attempts rather than confirmed deliveries.

---

## 23. Private vs Public-Safe Information

**PRIVATE — do not publish:**
- **The vulnerability narrative tied to this employer.** That a named insurance company issued 65,093 payment links worth £45M with an editable amount parameter is commercially and reputationally sensitive. The *engineering pattern* is publishable; the *attribution* is not. Use it freely in interviews and in this private record; anonymise completely for any public writing.
- **The CTO's commendation and the detail that previous developers had attempted the fix unsuccessfully.** Legitimate for interviews and your own record, but it characterises identifiable colleagues — keep it verbal, not published.
- **AI-assisted research on the Worldpay signing scheme.** You used AI to work out the correct approach in the absence of documentation. That is a legitimate and increasingly unremarkable engineering practice — solving, without docs, a problem others could not — and this record notes it for accuracy. Whether to surface it is entirely your call: some interviewers read it as resourcefulness, others fixate on it. The defensible framing if you do raise it is that you validated the scheme against the live integration and designed the fail-closed and missing-field safeguards yourself, which is what the code shows.
- Azure resource names (`o-payment-gateway-rg`, `o-payment-gateway-prod/stage`, registry name), the **production database hostname**, the production application hostname, and the UCP base URL.
- **Absolute business volumes and monetary values** (£47.76M lifetime estate, £34.95M paid, 68,869 payment records, monthly payment counts) — these are commercially sensitive. Relative and operational figures (81/81 reports, 71 users, 70-link job in 123 seconds, 0 failures) are safer but should still be stripped of employer identification.
- Environment variable names and defaults relating to Worldpay signing, and any signature field ordering.
- The internal admin email used as the bootstrap admin default.
- Local development database contents and all per-record data.
- The permission taxonomy as applied to real named staff, and any audit log contents.
- Internal platform names (UCP) in a context identifying the employer.
- Repository source code.

**PUBLIC-SAFE (with the genericisation noted):**
- Architectural patterns: app-owned RBAC with hashed invite tokens; job/item modelling for bulk operations; fail-closed configuration; database-backed sender settings.
- Technology stack as a list (Next.js, React, TypeScript, Prisma, MySQL, Docker, Azure, GitHub Actions).
- The contribution metrics in §4.1 (commits, lines, authorship share) — these describe personal output, not business data.
- Reliability and operational outcomes stated without employer context: "81 consecutive scheduled reports with zero failures", "70 links regenerated and 135 messages sent in 123 seconds with no failures", "an RBAC system serving ~70 internal users".
- Lessons and trade-offs in §13 and §16, described generically ("a payment provider", "a communications platform").
- Stories S1–S6 with employer-identifying detail removed.

*Where uncertain, default to private.* In particular, anything describing how payment links are signed should be generalised before publication.

---

## 24. Evidence Gaps and Further Metrics Worth Collecting

The production audit in §9 closed most gaps identified earlier — volume, value, users, go-live date, reporting reliability, bulk-resend usage and migration scale are all now measured. What remains:

| Gap | Why it matters | Where it likely exists | Question to ask |
|---|---|---|---|
| **True message delivery rates** | §9.3 counts are application-logged *attempts*; `Communication` has no failure state, so bounces and provider rejections are invisible to the app. This is the most important remaining caveat on the strongest volume claim. | UCP platform reporting / provider dashboards | "What are the actual delivery, bounce and failure rates for the payments app's email and SMS since July 2026?" |
| **Longer-run effect on payment conversion** | The release has only ~3 months of post-deployment data, below the threshold at which a modest effect could be distinguished from normal month-to-month variation. A baseline and a repeatable method are already prepared (separate working note, §25) so this can be re-run cleanly. | Production `Payment` table, once 6–12 months of post-release data exist | Re-run the prepared comparison around **March 2027**; until then make no conversion claim either way. |
| **Cause of the 2025 paid-rate movement** | The rate dipped during 2025 and recovered through early 2026, entirely before this work. Unexplained, and it is why a lifetime-average comparison is not a valid baseline. | Business/ops records for 2025; pricing, product or customer-mix changes | "Does anyone know what moved payment conversion during 2025, and what drove the recovery?" |
| **Manual effort displaced by bulk resend** | The only responsible basis for a time-saving claim. 70 links took 123 seconds; the manual equivalent is unknown. | Operations team | "Before bulk resend, how long did it take to chase and re-issue an expired payment link?" |
| **Whether the 16,257 expired-link backlog is being worked** | Shows ongoing business value rather than a single use — only one bulk job has run to date. | `BulkPaymentResendJob` over time | "Is bulk resend part of a regular operational routine, or was August a one-off?" |
| **Exact mechanism of the legacy migration** | Determines whether C9 is VERIFIED or STRONGLY SUPPORTED for personal attribution. | Sync script run logs; whoever performed the Azure cutover | "Were all 65,561 legacy rows moved via the sync script, or partly by database restore?" |
| **Infrastructure cost** | Cloud migration cost impact is entirely unknown. | Azure Cost Management | "What does the App Service + MySQL footprint cost per month?" |
| **Uptime / HTTP error rates** | Reliability beyond the job-level evidence in §9.6–9.7. | Azure App Service metrics; `/api/health` probe history | "Can I see availability and HTTP error rates since 17 July 2026?" |
| **Scope of the other 2026 contributor's work** | Required before claiming sole ownership of the phase. | The four `zayan_DevOps` commits | "What did the DevOps contributions cover?" |
| **Any record of actual exploitation of the link vulnerability** | Would move the £45M from "exposure surface" to a measured incident. Currently unknowable from the app: a tampered payment was rejected by the callback and left no record of the amount actually charged. | Worldpay merchant transaction reports vs application records for the same period; finance reconciliation records | "Can finance or Worldpay reporting show how many transactions settled at an amount that did not match the issued payment request before July 2026?" |
| **Written confirmation of the CTO's commendation and prior failed attempts** | Would upgrade C1b from STRONGLY SUPPORTED to VERIFIED and is strong CV material. | Slack/Teams messages, email, performance review notes, the ticket that initiated the engagement | "Could I get that feedback in writing for my record?" — worth doing soon, before it becomes hard to retrieve. |
| **Stakeholder feedback on the wider modernisation** | Qualitative colour for interviews and portfolio (the security fix feedback is already captured). | Operations/finance team, line manager | "Has the team given feedback on the admin, reporting or bulk resend features?" |

---

## 25. Appendix — Evidence Notes and Methodology

**Repository audited:** `/Users/welldunn/Documents/iRev_Projects/iRev_Projects_New/o-payment-gateway`, remote `github.com/Insurance-Revolution/o-payment-gateway`, branch `main`, HEAD `8f334e1` (2026-08-05). 134 commits total; first commit 2022-06-08.

**Commands used for authorship figures (reproducible):**
- `git shortlog -sne --all`
- `git log --author="Abraham" --format="%h|%ad|%an|%s" --date=short`
- `git diff --shortstat af467f9^ HEAD` → `103 files changed, 9029 insertions(+), 1110 deletions(-)`
- `git diff --diff-filter=A --name-only af467f9^ HEAD | wc -l` → `46`
- Per-file and aggregate `git blame --line-porcelain <file> | grep "^author "` counted against `grep -ci abraham`.

`af467f9` (2026-07-05) is the first commit of the contribution phase after the 2026-06-25 merge commit; using its parent as the baseline captures the full phase.

**Database access methodology.** Two databases were examined.

*Production* — Azure Database for MySQL 8.0.46, database `appdb`, reached over TLS using a connection string held **commented out** in `.env`. Every session first executed `SET SESSION TRANSACTION READ ONLY`, and the server confirmed `@@transaction_read_only = 1` — meaning MySQL itself would have rejected any write, independently of application-side safeguards. A wrapper script additionally refused any statement matching write or DDL keywords, parsing credentials without ever printing them. Only read statements (`SELECT`, aggregates, `SHOW`) were executed. No schema, data or configuration was modified, and no migrations or application commands were run. All queries were bounded aggregates or `GROUP BY` rollups; no bulk extraction of raw rows was performed.

*Local development* — a stopped Docker container (`o-payment-gateway-mysql`) was started, queried under the same guard, and **returned to its stopped state**. Its contents (14 payments, 2 users) are development data and are not cited as evidence in this dossier beyond noting the distinction.

**Key derivations.** Production runtime of 84 days = 2026-07-17 (migrations applied 18:26; first `AuthUser`/`AuditLog` rows 19:33) → 2026-10-09 (latest activity). Daily averages = totals ÷ 84. The go-live boundary `created_at >= '2026-07-17'` separates records created on the 2026 codebase from the migrated historical estate; every figure in §9.3 uses that filter.

**PII and secrets handling.** No credentials, connection strings, tokens or keys appear in this dossier. Only environment variable *names* were read. No customer names, emails, phone numbers, references or payment amounts from any database record are reproduced; all database findings are aggregate counts.

**Separate working note.** A private working note, `Paid-rate baseline and re-check method.md`, is held alongside this dossier. It contains the prepared pre/post baseline figures and the exact repeatable method (adjacent equal-length windows, year-over-year seasonal control, payment-size stratification, maturity exclusion) for assessing conversion impact once sufficient post-release data has accumulated. It is working material, deliberately not part of this dossier.

**Limitations of this audit.**
1. Message volume counts application-logged send *attempts*, not provider-confirmed deliveries (§9.3) — the most significant remaining caveat.
2. The 68,869-record estate and its £34.95M paid value largely **predate** this contribution; only post-17-July-2026 figures are attributable to it, and the two must never be conflated.
3. `git blame` measures surviving lines at HEAD and therefore favours the most recent contributor; it is reported alongside raw commit and diff counts for balance.
4. Business motivations in §3 are inferred from code and documentation; no requirements or ticket records were available.
5. Deployment *definitions* and their database effects were verified (migrations are timestamped in production), but GitHub Actions run history and Azure platform metrics were not accessible.
6. The post-release window (~3 months) is too short to assess any effect on payment conversion; no causal claim is made in either direction, and the lifetime average is not a valid baseline for that comparison.
7. The repository's three `.test.ts` files are empty placeholders; no test suite was available to assess quality.
