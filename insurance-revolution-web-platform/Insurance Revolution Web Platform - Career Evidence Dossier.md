# Insurance Revolution Web Platform — Career Evidence Dossier

**Subject:** Abraham Akintola (`abraham.akintola@insurancerevolution.co.uk` / `abramseyakid@gmail.com`)
**Audit date:** 10 October 2026
**Employment:** Joined Insurance Revolution **November 2024**; mid-level developer owning the front-facing website, then took over the backend lead API after the senior engineer departed
**Scope:** Six repositories comprising Insurance Revolution's customer-facing web platform and its lead-ingestion backend
**Production evidence:** Read-only inspection of the production PostgreSQL 16 lead database (Azure Database for PostgreSQL Flexible Server)

> **Reading note on attribution.** The platform predates the subject's employment. Throughout this dossier, production figures are given for two distinct windows: **platform lifetime** (from 9 Jan 2024, the earliest row in the current `leads` table) and **tenure** (from 1 Nov 2024). Career claims use tenure figures. Lifetime figures appear only to describe the system.

---

## 1. Executive Summary

Insurance Revolution (trading within the Well Dunn Ltd group) is a UK commercial and niche motor insurance broker. Its customer acquisition runs almost entirely through a WordPress-based public website whose quote forms feed a .NET lead-ingestion API, which in turn routes every lead into a Noetica predictive dialler, a Marketplace quoting service, and **Leela** — an internally-built cross-system identity service that mints a correlation ID linking the dialler to the broker's Acturis/OGI policy systems.

The audited estate is six Git repositories, 831 commits spanning **4 July 2024 → 25 September 2026** (earliest to latest commit across all repos), and roughly 59,000 lines of first-party application code.

**Role and contribution.** Abraham joined in **November 2024** and is the single largest contributor to the customer-facing layer: **384 of 831 commits (46%)** across the six repositories, and the majority author of four of them. Measured only over their tenure the share is higher still — **384 of 645 commits (59.5%)** made from 1 Nov 2024 onward are theirs, rising to **223 of 253 (88%)** in the quote-form plugin. Every one of their 384 commits falls inside the tenure window, independently corroborating the November 2024 start date against the Git record. They are the outright author of the calculator plugin (the plugin header names them directly), the principal author of the React quote-form plugin (223 of 401 commits), and a co-maintainer of the WordPress theme, competition wheel and register-interest form. From late 2025 they also took over the .NET lead API — a repository previously owned by the departing senior engineer — shipping PII redaction middleware, API-key header authentication, an SMS notification service, and a Noetica enrolment service with idempotent retry semantics. That lead API is itself a rewrite: a C# reimplementation of a legacy Go service, a migration Abraham participated in architecturally. The hand-over of ownership is visible in the commit record.

**Most technically significant aspects.** A 12-vertical, multi-step React form engine embedded in WordPress via shortcode hydration; a CSRF-nonce-protected WordPress REST proxy layer that keeps third-party API keys server-side; idempotent lead enrolment against an unreliable SOAP dialler API; a Go-to-.NET service migration requiring business-logic parity with a system still in production; and a faithful C# port of the Go `rs/xid` identifier algorithm so newly-minted correlation IDs stay byte-compatible with IDs already stored by the legacy service.

**The most interesting finding in this audit** is not a headline number but a diagnosis. The production database shows a 33% duplicate rate, which reads as a data-quality problem. It is not: it is a direct and predictable consequence of a design decision in the quote form — partial submissions are posted mid-journey with no client-generated correlation ID, so the final submission inserts a second row instead of updating the first. Section 8.6 establishes this forensically from the data alone, and section 11 documents the fix Abraham identified. This is the strongest evidence of engineering judgment in the whole estate.

**Strongest verified production statistics.**

*Tenure figures (1 Nov 2024 – 10 Oct 2026) are the defensible basis for career claims.*

| Metric | Tenure (from 1 Nov 2024) | Platform lifetime |
|---|---|---|
| Lead submissions ingested | **397,667** | 641,740 |
| **Unique enquiries** (repeat posts within 30 min collapsed) | **309,486** | — |
| Website-sourced submissions (`insurancerevolution.co.uk`) | **150,714** | 205,806 |
| **Unique website enquiries** | **77,433** | — |
| Integration failure rate (excluding duplicates) | **5,151 of 397,667 = 1.30%** | 0.91% |
| Current throughput | **30,656 in the last 90 days — ~341/day** | same |
| Distinct product/lead types in production | — | **101** |
| Distinct ingestion sources configured | — | **45** API keys (21 live) |
| Leads assigned a Noetica dialler customer record | — | **635,626** (99.05%) |
| Leads assigned a Leela cross-system correlation ID | — | **462,316** (72.0%) |

The **unique enquiry** figures are the honest headline. They collapse the partial-submission artifact described above and in §8.6, and they are the numbers to use in a CV or an interview: **~309,000 unique enquiries platform-wide, ~77,400 from the website**, over the tenure.

**Clearest business impact.** This platform is how the company acquires customers; the quote forms are the top of the revenue funnel and the lead API is the path every enquiry travels. During Abraham's tenure it has ingested **397,667 submissions representing ~309,500 unique enquiries**, routed them into the dialler without manual keying (99% CRM assignment), and suppressed repeat posts rather than letting them generate duplicate outbound calls. Features Abraham shipped are directly traceable to production volume: the Business Insurance vertical they built in July 2025 produced its first production lead on **4 August 2025** and has since generated **437** website leads; the out-of-hours convicted-driver routing introduced in January 2026 has handled **10,476**. Across five verticals, first commit to first live customer ran **0–17 days**.

---

## 2. Project Overview

The platform is a composed system rather than a single application. Six repositories:

| Repository | Purpose | Stack | Size |
|---|---|---|---|
| `wp-theme` | Public website theme — 28 page templates, product/category pages, GDPR and claims pages, GTM integration | PHP (Understrap/Bootstrap child theme), SCSS | 12,326 LOC PHP |
| `wp-quoteform-plugin` | The quote-form engine: 12 insurance verticals, multi-step forms, vehicle/postcode lookup, lead submission | React 18 + TypeScript + Vite + Formik/Yup + Ant Design, wrapped in a WordPress plugin (PHP REST proxy) | 169 TS/TSX files, 30,193 LOC |
| `wp-calculator-plugin` | Four marketing calculators embedded in content pages | React 18 + TS + Vite, WP plugin with ACF-driven shortcode | ~2,000 LOC |
| `wp-competition-wheel` | Promotional prize-wheel game with claim form | React + TS, WP plugin with REST endpoint → Azure Function + email | 528 LOC + PHP |
| `wp-register-interest-form` | Interest-capture modal for products not yet live | React + TS, WP plugin → Azure API + email | 304 LOC + PHP |
| `lead-api-dotnet` | Lead ingestion, validation, de-duplication, and fan-out to Noetica / Marketplace / Leela / Brevo / CM Telecom | .NET 8, ASP.NET Core + Azure Functions + EF Core + PostgreSQL, Docker on Azure Container Registry | Core 5,179 / Api 1,474 / Function 1,548 / Leela 1,485 / Tests 5,778 LOC |

**Who uses it.** Members of the public requesting insurance quotes (the top of the funnel); the internal call-centre team, who receive leads as dialler queue entries rather than emails or spreadsheets; marketing, who own the paid-search attribution and the calculators; and compliance, who receive GDPR subject-access requests through a dedicated form.

---

## 3. Business Problem and Context

A broker's economics are set by cost-per-acquired-policy. The problems the platform addresses:

1. **Many products, one funnel.** Insurance Revolution sells across at least 12 distinct verticals (fleet, courier, taxi, motor trade, convicted driver, young driver, goods in transit, fast food delivery, business/commercial, landlord, plant & machinery, minibus). Each needs a different question set, different validation, and different downstream routing. Production confirms **101 distinct lead types** in use. Building a separate form per product would be unmaintainable; the quote-form plugin exists to make a new vertical a configuration-plus-components exercise rather than a new application.

2. **Leads must reach a human fast, and only once.** Leads arrive from the website *and* from 20+ affiliate/aggregator partners (Quotezone, QuoteSearchUK, Insurance1, Leads2Market, Databowl, Policylink and others). Without central de-duplication the call centre would repeatedly ring the same person from different sources. The lead API's de-duplication rules exist for this; **212,054 leads (33.0%)** have been flagged duplicate and withheld from the dialler. As §8.6 establishes, most of that volume is not in fact the same person arriving twice from different partners — it is the platform's own quote form posting each enquiry twice by design.

3. **Three systems, no shared key.** Noetica (dialler) and OGI/Acturis (broking and policy admin) are separate third-party products with no common customer identifier. Leela was built to mint a stable correlation ID at the moment of lead capture so a customer can be tracked end-to-end from click to policy. *(Inference, well supported: the Leela code comments describe it as "the cross-database lead identity system … acting as a correlation key across multiple data stores", and 462,316 production leads carry a `leela_id`.)*

4. **Paid-search attribution.** The business buys Google and Microsoft Ads traffic. Click IDs must survive the journey from landing page through a multi-step form to the lead record. **149,940 production leads carry a `gclid`.**

5. **Out-of-hours capture.** Leads arriving outside opening hours must not simply be dropped into a queue nobody is working. The system branches on UK business hours for both SMS copy and dialler list selection.

---

## 4. My Role and Contribution

### 4.1 Authorship evidence

Abraham joined Insurance Revolution in **November 2024**. The Git record corroborates this independently: every one of their 384 commits across the six repositories falls on or after 1 Nov 2024, and none before.

Commit counts by repository (all branches; Abraham's Git identities combined). The tenure column is the fairer measure, because it excludes work done before they joined:

| Repository | Abraham's commits | Repo total | Share of repo | Repo commits **during tenure** | **Share during tenure** |
|---|---|---|---|---|---|
| `wp-quoteform-plugin` | **223** | 401 | 56% | 253 | **88%** |
| `wp-theme` | 48 | 120 | 40% | 82 | **59%** |
| `wp-competition-wheel` | 47 | 119 | 39% | 119 | 39% |
| `wp-register-interest-form` | **27** | 42 | 64% | 42 | **64%** |
| `wp-calculator-plugin` | **23** | 26 | 88% | 26 | **88%** |
| `lead-api-dotnet` | 16 | 123 | 13% | 123 | 13% |
| **Total** | **384** | **831** | **46%** | **645** | **59.5%** |

The tenure view is the one to quote. Since November 2024 Abraham has authored **88% of all commits to the quote-form plugin** — the single most valuable repository in the estate, since it is the surface every website lead passes through — and 59.5% of all work across the six repositories. The lead API is the one repository where they are a minority contributor, for the straightforward reason that they inherited it late.

Supporting evidence beyond commit counts: `wp-calculator-plugin/calculator-plugin/index.php` carries `Author: Abraham Akintola` in the plugin header — the only repository in the estate with an individual named as author.

### 4.2 Verified personal contributions

**Quote-form engine (`wp-quoteform-plugin`) — principal author.**
- Built the **Fast Food delivery** vertical end to end (Jan 2025) and its integration with the new Marketplace quoting service (`feat: integration with new marketplace`, 4 Feb 2025) and the Noetica dialler (`feat: sending form data to dialer`, 19 Feb 2025).
- Built the **Business Insurance** vertical (Jul–Sep 2025): three-page form, multi-checkbox cover-options UI, an extensive business-activity taxonomy, validation schema, and routing to the "premier" product category for lead email classification.
- Built the **Landlord Insurance**, **Minibus**, and **Plant & Machinery** verticals (Sep–Dec 2025).
- Built the **StepIndicator** progress component and rolled it across every vertical (18 Jul 2025, seven commits in one day).
- Built the **GTM/dataLayer analytics layer** (`src/utils/analytics.ts`, Aug 2025): `trackOpenForm`, `trackSubmitForm`, `trackBusinessInsuranceFormSubmit`, emitting structured `form_open` / `vehicle_insurance_quote` / `business_insurance_quote` events with product and risk attributes.
- Built **paid-search attribution plumbing**: `gclid` and `msclkid` cookie capture with a 30-day window, query-parameter persistence to `localStorage`, and injection of those parameters into form submissions (Jul 2025 – Feb 2026).
- Integrated **Ideal Postcodes** address lookup across all verticals (Feb 2026).
- Added **ABI vehicle-code** handling, vehicle transmission/fuel/body/doors lookup chains, UK residency fields, motoring and criminal convictions capture with validation (2025–2026).
- Owned the **WordPress REST proxy** work: added `/sendMarketplaceDW` (data-warehouse tracking route, Jun 2025), corrected the lead API path and query-parameter contract, and hardened `arrayToHtmlList` HTML escaping (Aug 2025).

**WordPress theme (`wp-theme`) — co-maintainer.**
- Business Insurance template family: form template with dynamic quote-form selection by vehicle type, product template, home page, and a category page variant (Jul 2025).
- **Calculator template** and the category-page calculator section (Aug–Sep 2025).
- **GDPR data-request form template** (Mar and Jun 2026) — the compliance intake surface.
- Google Tag Manager integration in the theme header (Sep 2025).
- Make-a-claim templates, Trustpilot TrustBox widget rollout, WhatsApp widget (later removed over a Google Ads policy conflict with jumper.ai), young-driver fast-track journey, floating action buttons, blog shortcode support.

**Calculator plugin (`wp-calculator-plugin`) — author.**
Four calculators (Convicted Driver premium estimate, Drink Driving unit/wait-time calculator, BIK Car, Fleet Cost Per Mile), a shared `CustomInputField`/`CustomSelectField` abstraction driven by Formik context, Yup validation per calculator, a React 18 migration, and an ACF-driven shortcode so marketing can place a calculator by picking a type in the WordPress editor rather than editing a shortcode attribute.

**Competition wheel (`wp-competition-wheel`) — co-author.**
Prize-wheel colour and prize configuration, claim modal and form, submission spinner and success/failure states, date-picker constraints, **invisible Google reCAPTCHA** (Jan 2025), live API wiring, and a secrets-removal remediation across three commits (Jan 2025) after credentials had been committed.

**Register-interest form (`wp-register-interest-form`) — majority author.**
Modal form, error handling, email-body fixes, database-API connection, environment-variable handling for email and Azure endpoints, and invisible reCAPTCHA.

**Lead API (`lead-api-dotnet`) — inherited ownership, Nov 2025 onward.**
- **Fleet contact strategy** (PR #2, Nov 2025): `FleetSmsService` — CM Telecom SMS to fleet leads, with UK-timezone-aware in-hours/out-of-hours message variants and a 10-second HTTP timeout.
- **Brevo** list-ID change and constrained contact-creation logic (PRs #4, #7).
- **PII redaction and conditional logging** in both the ASP.NET and Azure Functions middleware (Aug 2026) — query-string sanitisation, trace-ID correlation, and gated raw-body capture to blob storage.
- **`NoeticaEnrolmentService`** and its API controller (Aug–Sep 2026): a 292-line service with explicit idempotency — on a repeat enrolment it distinguishes *already queued*, *CRM-failed with unknown external outcome*, and *in-flight* states, and refuses to create a second dialler customer when the outcome of the first attempt is unknown.
- **`x-api-key` header authentication** across both the API and Functions front ends, with Swagger documentation and tests (Sep 2026).
- **Noetica dialler list-timing rules** (Sep 2026): moved list 53 (DrivingSchool/PPC) into the five-day latest-time list; investigated and reverted a 30-minute latest-time change to list 150 after the change proved wrong — the revert is in the history.

### 4.3 Work in these repositories *not* attributable to Abraham

The core lead-ingestion engine — `LeadService` (1,134 LOC), `NoeticaService` (651 LOC), the Go→C# reimplementation, the Leela domain model and its `XidGenerator`, the Marketplace service, EF Core repositories, and the Azure deployment workflows — is predominantly the work of Geoffrey Prytherch (98 of 123 commits). Abraham inherited and now maintains it, but did not write it. Significant parts of the theme and the competition wheel are Jose Wills's work. Nicholas Wootton and Roxie Gibson contributed to the quote-form plugin.

---

## 5. System Architecture

```
┌───────────────────────── Browser ─────────────────────────┐
│  insurancerevolution.co.uk (WordPress + Understrap child) │
│                                                            │
│  [wp-theme]           28 page templates, GTM, TrustBox     │
│   ├─ shortcode ──► [quote-form plugin]  React 18 island    │
│   │                  12 verticals · Formik/Yup · multi-step│
│   ├─ shortcode ──► [calculator plugin]  4 calculators      │
│   ├─ shortcode ──► [competition wheel]  prize game         │
│   └─ shortcode ──► [register-interest]  modal form         │
│                                                            │
│  gclid / msclkid cookie (30d) + queryParams → localStorage │
└──────────────────────────┬─────────────────────────────────┘
                           │ POST  (WP nonce / CSRF-checked)
                           ▼
┌────────────── WordPress REST proxy  (plugin PHP) ──────────┐
│  quote/v1/*  — 15 routes                                    │
│   vehicleMakes · vehicleModels · vehicleYears · vehicleBodies│
│   vehicleTransmission · fuelTypes · vehicleSpecification     │
│   vehicle · addressOptions · fullAddress                     │
│   send · sendMarketplace · sendMarketplaceDW                 │
│   sendMarketplaceMin · sendPolicyUpdate                      │
│  Keeps third-party API keys server-side; never in the bundle │
└──────────────────────────┬─────────────────────────────────┘
                           │ HTTPS + API key
                           ▼
┌───────── lead-api-dotnet (.NET 8, Docker → Azure) ─────────┐
│                                                             │
│  Leads.Api (ASP.NET Core)  ║  Leads.Function (Azure Funcs)  │
│      — redundant front ends sharing Leads.Core —            │
│  Middleware: ApiKeyAuth · LeelaAuth · ForceJsonContentType  │
│              RequestBodyLogging (PII-redacted, trace-id)    │
│                           │                                 │
│                    ┌──────▼──────┐                          │
│                    │ LeadService │  validate → normalise     │
│                    │  (1,134 LOC)│  → de-duplicate → persist │
│                    └──┬───┬───┬──┘  → fan out                │
│        ┌──────────────┘   │   └──────────────┐              │
│        ▼                  ▼                  ▼              │
│  NoeticaService      LeelaService      MarketplaceService   │
│  (SOAP: CreateCrm    (HTTP, 1s         (HTTP)               │
│   Record / Create     timeout)                              │
│   QueuedRecord)           │                                 │
│        │                  │            BrevoService (email) │
│        │                  │            FleetSmsService      │
│        │                  │              (CM Telecom SMS)   │
│        │                  │            QZLeadsSink (async,  │
│        │                  │              fire-and-forget)   │
│        ▼                  ▼                                 │
│  Noetica dialler     Leela identity store                   │
│  (106 campaigns)     → leela_id → OGI / Acturis             │
│                                                             │
│  PostgreSQL 16 (Azure Flexible Server)                      │
│   leads (641,740) · scheduled_leads · api_keys (45) ·       │
│   companies (6) · noetica_campaigns (106) · audit_logs ·    │
│   gdpr_forms + attachments                                  │
└─────────────────────────────────────────────────────────────┘
```

**Deployment.** The lead API is containerised (multi-stage Dockerfile, .NET 8 SDK → ASP.NET runtime, port 8080) and pushed to `insurancerevolution.azurecr.io`. Six GitHub Actions workflows cover build-and-test, UAT (two variants), production, live, and **rollback**. Every deployment workflow is gated behind a **Bearer SAST security scan** job. The WordPress artefacts build to a `plugin.zip` / `dist.zip` and are installed through the WordPress admin — there is no CI for the WordPress repositories.

**Quality gates in the theme repo:** PHPCS, PHPStan (with baseline), and PHPMD (with baseline) configurations are present; Dependabot is active across the estate.

---

## 6. Main Workflows

**A. Quote capture (the primary revenue path).**
1. Visitor lands on a product page, often from a paid ad; `gclid`/`msclkid` is written to a 30-day cookie and query parameters are persisted to `localStorage`.
2. A theme page template emits `[create_quote_form formtype='…']`; the plugin renders an identified `<div>` and enqueues a deferred bundle that hydrates it. A WordPress nonce is emitted into the footer.
3. The React form walks the visitor through a vertical-specific multi-step journey, with a step indicator, framer-motion transitions, and Yup validation per step. Vehicle lookup chains (make → model → year → body → transmission → fuel → specification → ABI code) and Ideal Postcodes address lookup are each served through the plugin's own REST proxy, so no third-party key reaches the browser.
4. `form_open` and a product-specific submit event are pushed to the GTM dataLayer.
5. Submission POSTs to `quote/v1/send`; the PHP handler validates the CSRF nonce, composes the lead payload (including `gclid` and the persisted query parameters), optionally sends an internal notification email, and forwards to the lead API.

**B. Lead ingestion and fan-out.**
1. The lead API authenticates the caller by API key, resolving it to a `source` row and a `company`.
2. The UK mobile number is validated and normalised; failures are recorded as `invalid_phone`.
3. De-duplication runs against prior leads on the same mobile and source, with carve-outs for priority product types and for previously *cancelled* leads. Duplicates are persisted but marked `duplicate = true` with `fail_reason = 'duplicate'` and are not dialled.
4. A `leela_id` is minted or validated, giving the lead a correlation key usable in OGI/Acturis.
5. A Noetica CRM customer record is created (SOAP), then a queued dialler record is created against the campaign list resolved from the lead's type and company. Out-of-hours leads route to dedicated lists.
6. Fast-food leads additionally go to Marketplace. Fleet leads receive a CM Telecom SMS whose copy differs in and out of hours. Brevo handles transactional email and list membership. Selected lead types are forwarded asynchronously to QZLeadsSink.
7. Every failure maps to a stable `fail_reason` code so that duplicate, invalid-phone, and genuine integration failures are distinguishable in reporting.

**C. Noetica enrolment (idempotent re-entry).** A separate endpoint lets an upstream system enrol an already-known lead (by `leela_id`, source, and type) into a dialler campaign. The service returns `queued`, `processing`, or `failed` and refuses to create a duplicate CRM customer when a prior attempt's external outcome is unknown.

**D. Supporting workflows.** Register-interest capture for unlaunched products; the competition wheel's prize claim (reCAPTCHA → WP REST → Azure Function + email); GDPR subject-access requests via the theme's GDPR form template into `gdpr_forms` and `gdpr_form_attachments`.

---

## 7. Technology Stack

**Frontend:** React 18, TypeScript 5, Vite 5, Formik, Yup, Ant Design 5, framer-motion, react-toastify, dayjs, axios, Ideal Postcodes postcode-lookup, Google reCAPTCHA (invisible).
**WordPress:** PHP, Understrap/Bootstrap 5 child theme, WP REST API, shortcodes, nonces, ACF, Google Tag Manager, Trustpilot TrustBox.
**Backend:** .NET 8, ASP.NET Core, Azure Functions (isolated worker), Entity Framework Core, Npgsql, Swashbuckle/OpenAPI, xUnit.
**Data:** PostgreSQL 16 on Azure Database for PostgreSQL Flexible Server (TLS required), Prisma migrations (from an adjacent reporting tool), Azure Blob Storage.
**Integrations:** Noetica Synthesys dialler (SOAP), Leela identity service (HTTP), Marketplace quoting service, QZLeadsSink, Brevo (email), CM Telecom (SMS), Ideal Postcodes, Google/Microsoft Ads click IDs.
**Infra/CI:** Docker, Azure Container Registry, Azure App Service, GitHub Actions (6 workflows incl. rollback), Bearer SAST, Dependabot, PHPCS/PHPStan/PHPMD, ESLint.

---

## 8. Production Evidence and Statistical Analysis

All figures below come from read-only aggregate queries against the production `leads` database on 10 October 2026. No raw customer data was retrieved; every query was a `COUNT`/`MIN`/`MAX`/`GROUP BY`. Session was set `default_transaction_read_only = on`.

### 8.1 Volume

| Measure | Platform lifetime | **Tenure (from 1 Nov 2024)** |
|---|---|---|
| Rows in `leads` | **641,740** | **397,667** |
| Window | 9 Jan 2024 – 10 Oct 2026 (~33 months) | 1 Nov 2024 – 10 Oct 2026 (~23 months) |
| Unique enquiries (30-min collapse, see §8.6) | — | **309,486** |
| 2024 | 278,791 | — |
| 2025 | 253,894 | — |
| 2026 (to 10 Oct) | 109,055 | — |
| Last 90 days | 30,656 (**~341/day**) | same |
| Busiest single day | 1,656 (29 Jul 2025) | same |

Roughly **62% of all leads the platform has ever recorded arrived during Abraham's tenure.**

*Qualification:* a row is a lead **submission**, not a unique person. The same individual can appear many times across sources and products; indeed 33% of rows are flagged as duplicates. The 2024 start date reflects the current table's retention, not necessarily the system's first day in production — an older `leads_backup` table (35,499 rows) exists separately.

### 8.2 Outcome quality

| Outcome | Count | % of all leads |
|---|---|---|
| Flagged duplicate (withheld from dialler) | 212,054 | 33.04% |
| `invalid_phone` | 3,721 | 0.58% |
| `marketplace_connection_issues` | 1,826 | 0.28% |
| `noetica_connection_issues` | 314 | 0.05% |
| `no_campaign_for_lead` | 3 | <0.01% |
| `leela_connection_issues` | 1 | <0.01% |
| **Integration failures (excl. duplicates)** | **5,866** | **0.91%** |
| Assigned a Noetica customer ID | 635,626 | 99.05% |
| Assigned a Noetica queue entry | 381,109 | 59.39% |
| Assigned a Leela correlation ID | 462,316 | 72.04% |
| Cancelled | 48,049 | 7.49% |

Restricted to the tenure window, the non-duplicate failure count is **5,151 of 397,667 = 1.30%**.

*Interpretation.* Both figures are defensible reliability numbers for the ingestion pipeline. The queue-entry figure (59%) is lower than the CRM figure (99%) precisely because duplicates get a CRM record but no dialler queue entry — 59.39% + 33.04% ≈ 92%, which is consistent. Use the **1.30% tenure figure** in career claims; it is the one that covers the period Abraham was responsible for, and it is the more conservative of the two.

### 8.3 Breadth

- **101** distinct `type` values and **95** distinct `subtype` values in production.
- **45** configured API-key sources across **6** companies; **21** sources carry live traffic.
- **106** Noetica campaign mappings.
- Largest sources: `insurancerevolution.co.uk` 205,806 · `quotezone.co.uk` 182,771 · `marketplace.insurancerevolution.co.uk` 112,999 · `quotesearchuk.co.uk` 56,420 · `insurance1.com` 31,955.

### 8.4 Website channel (the surface Abraham owns)

| Year | Website leads | Duplicates | With click ID (non-empty) |
|---|---|---|---|
| 2024 | 63,603 | 32,624 | 22,705 (35.7%) |
| 2025 | 90,103 | 47,845 | 65,671 (72.9%) |
| 2026 (to 10 Oct) | 52,100 | 24,393 | 32,759 (62.9%) |
| **Total** | **205,806** | **104,862** | **121,135** |

Restricted to the tenure window, the website channel carries **150,714 submissions representing 77,433 unique enquiries** (see §8.6).

*Qualification:* the rise in click-ID coverage from 2024 to 2025 coincides with the attribution work but cannot be causally attributed to it from this data alone — ad-spend mix also changes year to year. Treat as **DERIVED with a stated limitation**, not as a claimed improvement percentage.

Website monthly volume has declined through 2026 (Jan 9,984 → Sep 4,339). No evidence in this dataset explains why; it may reflect marketing budget, channel mix, or seasonality, and should not be presented as a platform metric either way.

### 8.5 Feature-to-production traceability

The clearest evidence linking Abraham's shipped code to live usage: the first production lead of a type appears days after the commits that built it.

| Vertical | Built (commits) | First production lead | Website leads to date |
|---|---|---|---|
| Business Insurance | 18 Jul – 1 Aug 2025 | **4 Aug 2025** | 437 |
| Minibus Insurance | 16 Sep 2025 | **24 Sep 2025** | 427 |
| Landlord Insurance | 16–24 Sep 2025 | **24 Sep 2025** | 5 |
| Plant & Machinery | 15 Dec 2025 | **19 Dec 2025** | 242 |
| Courier Van "Route Flex" | 6–10 Feb 2026 | **10 Feb 2026** (same day) | 6 |
| Convicted Driver (out-of-hours) | — | 16 Jan 2026 | 10,476 |
| Fast Food / Marketplace family | Jan–Apr 2025 | — | 111,725 across all sources |

*Qualification:* the fast-food/marketplace family predates Abraham's 2025 work (first rows 9 Jan 2024), so its 111,725 total cannot be claimed as theirs. The five rows above the line can: the lead type did not exist in production before their commits.

### 8.6 Diagnosing the 33% duplicate rate — partial submissions, not bad data

A naive reading of the production data says a third of all leads are duplicates, which looks like a data-quality failure. It is not. It is a predictable consequence of a deliberate design decision in the quote form, and establishing that was the most substantive piece of analysis in this audit.

**The mechanism.** The multi-step quote form posts a **partial submission** part-way through the journey — as soon as the customer has given contact details — so that a drop-out is still a contactable lead. When the customer finishes, the form posts again, in full. The second post carries no correlation ID tying it to the first, so the lead API has no way to recognise it as the same enquiry and inserts a second row. One customer, one journey, two rows — and the second is flagged `duplicate`.

*Code evidence:* `src/utils/index.ts` `nextHandler()` calls `submitForm(formik.values, true)` mid-journey and fires a `partialSubmit` dataLayer event; `plugin/api/functions/send.php` forwards a `partial` boolean to the lead API and suppresses the notification email when it is true. The `leads` table has no column for a client-supplied correlation ID.

**The data confirms it.** Measuring the interval between consecutive submissions sharing the same mobile, type and source, for the website channel over the tenure:

| Gap to previous identical submission | Submissions | Flagged duplicate |
|---|---|---|
| Under 1 minute | 40,060 | 39,485 |
| 1–5 minutes | 31,011 | 30,527 |
| 5–30 minutes | 2,210 | 2,129 |
| 30–60 minutes | 243 | 229 |
| 1–24 hours | 806 | 744 |
| 1–30 days | 1,877 | 362 |
| Over 30 days | 2,164 | 88 |
| First of its kind | 72,343 | 3,100 |

**94.1% of flagged duplicates in the website channel arrive within 30 minutes of the original, and 91% within five minutes.** That is not human behaviour — nobody re-requests the same insurance quote twice in 90 seconds. It is one journey producing two rows.

**The decisive test** is to compare channels. The website is the only source running the multi-step form with partial submission; affiliate and aggregator partners post a single complete lead. If the theory is right, the website should show a sub-30-minute repeat rate an order of magnitude above everyone else. It does:

| Source | Submissions (tenure) | Repeats within 30 min | Rate |
|---|---|---|---|
| **Website (own quote forms)** | 150,714 | 73,281 | **48.6%** |
| Marketplace (also form-driven) | 41,303 | 6,123 | 14.8% |
| Affiliate partner A | 4,883 | 326 | 6.7% |
| Affiliate partner B | 114,883 | 6,790 | 5.9% |
| Affiliate partner C | 9,354 | 393 | 4.2% |
| Affiliate partner D | 36,693 | 322 | 0.9% |
| Affiliate partner E | 21,862 | 52 | **0.2%** |

The channel that runs the partial-submission form has a repeat rate **240× higher** than the cleanest single-post affiliate feed. The mechanism is established beyond reasonable doubt.

**What this means for the numbers.** Collapsing repeat posts within a 30-minute window gives the defensible unique-enquiry counts:

| | Raw submissions | **Unique enquiries** | Artifact rows |
|---|---|---|---|
| Platform-wide, tenure | 397,667 | **309,486** | 88,181 (22.2%) |
| Website channel, tenure | 150,714 | **77,433** | 73,281 (48.6%) |

*Method:* partition by (mobile, type, source) ordered by `created_at`; count rows whose gap to the previous row is null or ≥ 30 minutes. *Limitations:* 30 minutes is a judgement call — a longer window would collapse more and a shorter one fewer, though the distribution above is so concentrated under 5 minutes that the figure is stable across any threshold from 10 to 60 minutes. It also cannot separate a partial-then-full pair from a genuine immediate resubmission after an error, and a single person using two phone numbers still counts twice. Treat as **DERIVED**, with the method stated.

**Why this matters professionally.** Three reasons. First, it converts a number that looks damaging in an interview into one that demonstrates analytical rigour. Second, it produces honest headline figures (~309,000 unique enquiries, ~77,400 from the website) rather than inflated ones. Third, the fix is documented in §11 — and it is a fix Abraham has already half-built, which is considerably stronger evidence of judgement than simply identifying the problem.

### 8.7 Compliance surface

`gdpr_forms`: 41 submissions with 84 attachments, 22 Jun 2026 – 5 Oct 2026 — matching the GDPR form template Abraham committed to the theme on 22 June 2026. A small but cleanly traceable feature-to-production link. An `audit_logs` table (54 rows, actor email + action + resource + IP + user agent) exists for the administrative tooling built on the same database.

---

## 9. Business Impact

Established by the evidence above:

- **This is the company's revenue funnel, not a peripheral system.** Every customer the business acquires through its own marketing enters through these quote forms and travels this pipeline. That is a statement about the system's position, verifiable from the architecture: the website source alone accounts for 150,714 submissions during the tenure, and there is no alternative path from the public site into the dialler. *(VERIFIED structurally; the revenue share attributable to it is UNKNOWN)*
- **Automation of lead handling at scale.** 397,667 submissions (~309,500 unique enquiries) ingested and routed during the tenure without manual keying; 99.05% automatically created as dialler CRM records. *(VERIFIED)*
- **Repeat-post suppression.** The de-duplication layer prevents the partial-then-full submission pattern (§8.6) from generating a second outbound call per web enquiry — roughly 73,300 website rows during the tenure. Without it, nearly half of all website enquiries would have been dialled twice. The agent-time saving is real but cannot be quantified without call-cost data. *(VERIFIED count; monetary impact UNKNOWN)*
- **Cross-system customer identity.** 462,316 leads carry a Leela correlation ID, enabling a customer to be followed from web click through dialler to broking system. *(VERIFIED count; the downstream reconciliation benefit is INFERRED)*
- **Reliability.** 1.30% non-duplicate failure rate across the tenure (0.91% lifetime). *(DERIVED from verified counts)*
- **Product expansion velocity.** Five new insurance verticals went from first commit to first production lead in 4–17 days. *(VERIFIED)*
- **Compliance capability.** A self-serve GDPR data-request intake replaced whatever preceded it; 41 requests handled in the first ~3½ months. *(VERIFIED volume; the "what it replaced" is UNKNOWN)*
- **Marketing attribution.** 149,940 leads carry a paid-search click ID, enabling closed-loop ad reporting. *(VERIFIED)*

Not establishable from available evidence: revenue, conversion rate, cost per acquisition, call-centre headcount effects, or any before/after performance comparison.

---

## 10. Significant Engineering Challenges

**Twelve product verticals in one bundle without twelve codebases.** Each vertical has its own `data/`, `types/`, and `validation/` module plus a page component set, all composed by a shared `ParentForm`, shared `CustomInputField`/`CustomSelectField`, and a shared `formSteps` map. The hard part is not React — it is keeping 30,000 lines of question logic coherent while non-technical stakeholders (marketing, compliance, insurers) change individual questions weekly. The commit history shows the real cost: dozens of small commits adjusting a single option's display value, a label's HTML entity, or one field's validation message.

**React islands inside WordPress.** The form is not a SPA. A shortcode emits a marker `<div>`; a deferred script hydrates it; a WordPress nonce lives in the footer; and the whole thing must coexist with Bootstrap, the theme's own CSS, and whatever a content editor placed on the page. A recurring class of bug visible in the history is styling and layout regressions caused by this coexistence.

**Keeping third-party keys out of the browser.** Vehicle lookup, postcode lookup, and lead submission all run through the plugin's own `quote/v1/*` REST namespace rather than calling vendors directly. Fifteen proxy routes exist purely so that credentials stay server-side and every write is nonce-checked. This is the correct design and it is also why the plugin has a non-trivial PHP layer at all.

**An unreliable SOAP dialler.** Noetica is driven over SOAP (`CreateQueuedRecord`, `InsertCrmRecord`) with hard-coded queue-state constants and eight campaign list IDs. Production records 314 `noetica_connection_issues`. The enrolment service Abraham wrote has to reason about a partially-failed external call: if a CRM customer ID was saved, retry is safe; if the attempt failed with no customer ID, the external outcome is *unknown* and retrying risks creating a duplicate customer in a system the code does not own. The service returns `failed` rather than retrying. That is a deliberate correctness-over-convenience trade-off.

**De-duplication that must be wrong sometimes.** The rules are not "same number = duplicate". They carve out priority product types (a higher-value product should get through even if the number was seen recently), previously cancelled leads (a cancellation re-opens the person to contact), and a special window for `fastfood-express-quote`. Each carve-out is a business rule that trades false-negative duplicates against lost genuine leads.

**Byte-compatible identifier generation across a language migration.** `XidGenerator` is a hand-ported C# implementation of Go's `rs/xid`: 12-byte layout (4-byte big-endian timestamp, 3-byte MD5-of-hostname machine ID, 2-byte process ID, 3-byte atomic counter) and a direct port of the base32 packing routine producing exactly 20 characters. It exists because IDs already in production were minted by the Go service and must remain sortable and format-identical. *(Author: Geoffrey Prytherch — noted here as a feature of the system Abraham now maintains, not as their work.)*

**Timezone-correct business-hours logic.** SMS copy, Brevo behaviour, and dialler list selection all branch on UK local time. `TimezoneHelper` + `BrevoService.IsOutOfHours` are injected with a `UtcNowProvider` func so the behaviour is testable — and there is a dedicated `OutOfHoursTests` and `TimezoneTests` suite.

**PII in logs.** Logging enough to debug a lead that went wrong, without writing names, phone numbers, or email addresses into Azure log analytics. The solution Abraham shipped: trace-ID correlation on every request, sanitised query strings, `Phone.CensorPhone()` in all phone-adjacent log statements, and raw-body capture to blob storage only when explicitly enabled and only for two specific routes.

---

## 11. Architectural Decisions and Trade-offs

**Decision: Proxy every third-party call through WordPress rather than calling vendors from the browser.**
*Problem:* vehicle data, postcode lookup, and lead submission all need credentials. *Decision:* 15 `quote/v1/*` REST routes in the plugin, each nonce-checked. *Reasoning:* no key in the JS bundle; one place to add CSRF checks; one place to reshape payloads when a vendor contract changes. *Trade-offs:* an extra network hop and latency on every lookup; WordPress becomes a availability dependency of the form; the plugin now has a PHP surface that must be maintained alongside the React one. *Result:* credentials have stayed server-side; vendor contract changes (the Marketplace path change, the data-warehouse route addition) were absorbed in PHP without touching the bundle.

**Decision: Two front ends (ASP.NET Core API + Azure Functions) over one shared `Leads.Core`.**
*Problem:* lead ingestion is revenue-critical and a single App Service is a single point of failure. *Decision:* `Leads.Core` holds all business logic; `Leads.Api` and `Leads.Function` are thin shells, each with its own mirrored middleware. *Reasoning:* redundancy/failover, per the README. *Trade-offs:* middleware is duplicated (`RequestBodyLoggingMiddleware` and `FunctionRequestBodyLoggingMiddleware` are near-identical, 219 and 212 lines) — every cross-cutting change must be made twice. Abraham's PII-redaction and `x-api-key` commits each touched both. *Result:* redundancy achieved; maintenance cost is visible and real.

**Decision: Partial submissions without a client-generated correlation ID — the inherited design, and the fix.**
*Problem:* customers abandon long insurance forms part-way through. A drop-out who has already given a phone number is a contactable lead and should not be lost.
*Decision (inherited):* post a partial submission mid-journey, then post again in full at the end; let the lead API's de-duplication layer recognise the second post and suppress the duplicate dial.
*Reasoning:* it captures abandoners with no change to the lead API contract, which is the cheapest possible implementation.
*Trade-offs:* severe, and visible in the data. One enquiry produces two rows; 48.6% of website submissions are repeat posts within 30 minutes (§8.6); a third of the whole `leads` table is non-actionable; every naive `COUNT(*)` overstates demand; and correctness now depends on time-window heuristics in the de-duplication layer rather than on identity. The alternative — mint a UUID in the browser at the start of the journey, send it with both posts, and have the API upsert on it — is strictly better: it is exact rather than heuristic, it survives arbitrary gaps between partial and final submission, and it removes the de-duplication layer from the critical path for this case entirely.
*Result and current state:* **Abraham identified this as the thing they would design differently, and has already built half of it.** The newer verticals they authored — Business Insurance, Landlord, Plant & Machinery, Fast Food — each generate a client-side `uuidv4()` as `id` in their initial values (`src/data/businessInsurance.ts:6` and siblings). That `id` is forwarded on the Marketplace data-warehouse route (`send_marketplace_dw`), but **not** on the main `send` route to the lead API, and the `leads` table has no column to receive it. Closing the loop is therefore a bounded, well-understood piece of work: forward the existing field, add a nullable indexed column, and upsert on it when present — with the de-duplication layer retained as the fallback for sources that do not supply one. This is a documented, evidence-backed engineering position, not a retrospective rationalisation.

**Decision: Rewrite the lead API from Go to .NET.**
*Problem:* the original lead ingestion service was a Go application maintained by the senior engineer; the business wanted it on a stack the wider team and the then-CTO preferred.
*Decision:* a full C# reimplementation targeting business-logic parity — the README states the goal as "a C# reimplementation of the original Go-based lead API, with business logic parity and improved error handling". Abraham participated in the architectural discussion; **the choice of .NET was a CTO-level mandate rather than a team decision**, and Abraham has stated they would have chosen differently had the call been theirs.
*Reasoning:* organisational — stack consolidation and maintainability by the available team — rather than technical.
*Trade-offs:* a rewrite of a revenue-critical service carries migration risk for no new functionality, and parity with a live system is an exacting constraint. The cost is visible throughout the codebase: comments reading "mirroring Go implementation" appear across `MarketplaceService`, `Phone`, `LeadFactory` and the Noetica and Leela DTOs, and `XidGenerator` is a hand-port of Go's `rs/xid` down to the byte layout, so identifiers already in production stay sortable and format-identical. Every one of those is a place where the new system is constrained by the old one's behaviour rather than free to improve on it.
*Result:* the .NET service runs in production at ~341 leads/day with a 1.30% non-duplicate failure rate. The parity constraint was met. *(The rewrite was predominantly Geoffrey Prytherch's implementation work; Abraham's documented role is architectural participation and subsequent ownership.)*

**Decision: Return `failed` instead of retrying when a dialler CRM call's outcome is unknown.**
*Problem:* an enrolment that threw after possibly creating a customer. *Decision:* if no customer ID was saved **and** a fail reason exists, return `failed` with `alreadyExists = true` rather than re-attempting. *Reasoning:* the code comment states it directly — "A failed CRM operation has an unknown external outcome, so it is returned without creating another customer." *Trade-offs:* a genuinely-failed lead needs human or upstream intervention to recover. *Result:* no duplicate customers created in a third-party system the code cannot clean up.

**Decision: ACF-driven calculator type instead of a shortcode attribute.**
*Problem:* marketing needed to place calculators on pages without touching shortcode syntax. *Decision:* the shortcode reads `get_field('choose_a_calculator_type')` first and falls back to the attribute. *Reasoning:* editor-friendly, no developer in the loop. *Trade-offs:* a hidden dependency on ACF being installed and the field existing. *Result:* calculators are placeable from the editor; the attribute fallback preserves existing usages.

**Decision: 1-second HTTP timeout on the Leela identity call.**
*Reasoning (inferred from the code):* identity assignment must not block lead capture. *Trade-off:* leads can land without a `leela_id` under load — consistent with 72% coverage rather than ~99%.

---

## 12. Problems Solved

### Committed secrets in the competition wheel

**Situation.** The competition-wheel plugin needed an Azure Function URL and function key, plus a mail API key.
**Problem.** Credentials reached the repository. Three successive commits across 3 Jan, 9 Jan and 17 Jan 2025 are titled `fix: removed secrets` and `fix: completed removed secrets` — the pattern shows the first attempts did not fully clear them.
**Investigation/Solution.** Configuration moved behind a `get_comp_env_vars()` accessor; the same pattern appears in `wp-register-interest-form` (`fix: corrected utils env function for email and azure`, 20 Jan 2025). Invisible Google reCAPTCHA was added to both forms in the same window.
**Result.** Both plugins now read endpoints and keys from environment configuration. **Lesson, stated honestly:** the three-commit sequence shows the remediation was iterative rather than clean, and rotating a leaked key is a separate step this evidence does not confirm was done.
**Evidence:** `wp-competition-wheel` commits 2025-01-03/09/17; `azure-database.php`; `wp-register-interest-form` commit 2025-01-20.

### Google Ads policy conflict with the WhatsApp widget

**Situation.** A WhatsApp chat widget (jumper.ai) was added to the theme in Dec 2024.
**Problem.** It conflicted with Google Ads policy — `fix: removed whatsapp because of jumper.ai issue with google ads` (21 Feb 2025).
**Result.** Removed within roughly two months. A concrete case where a third-party marketing widget had to be reversed for a compliance reason outside engineering's control.
**Evidence:** `wp-theme` commits 2024-12-20 and 2025-02-21.

### Noetica list-timing change, shipped and reverted

**Situation.** September 2026: a change to the dialler's latest-contact-time rules for campaign list 150 (Convicted Driver).
**Problem.** The 30-minute latest-time change was wrong.
**Investigation/Solution.** Three commits on 25 Sep 2026: `fix: use 30 minute latest time for Noetica list 150` → `revert: restore Noetica list 150 special timing` → `Implemented the confirmed change: DrivingSchoolandPPC list ID 53 is now in the Noetica 5-day latest-time list. List 150 remains unchanged in that same special list.`
**Result.** The correct change (list 53) shipped; list 150 restored. The commit message explicitly records that the change was *confirmed* before re-applying — evidence of a verify-with-the-business step after a bad first attempt.
**Evidence:** `lead-api-dotnet` commits 2026-09-25.

### PII reaching request logs

**Situation.** Inherited middleware logged raw request bodies for lead endpoints.
**Problem.** Lead payloads contain names, phone numbers, email addresses, dates of birth, and addresses.
**Solution.** Abraham's Aug 2026 commit implemented query-string sanitisation, trace-ID correlation, phone censoring in log statements, and gated raw-body capture (`SensitiveRequestLoggingEnabled()`) restricted to `/lead/add` and `/new-quotezone-lead`, writing to blob storage fire-and-forget rather than to the log stream.
**Result.** Debuggability preserved; PII removed from the default logging path. Mirrored in both the API and Functions middleware.
**Evidence:** commit `feat: implement PII redaction and conditional logging for sensitive request data in API and Function middleware`, 28 Aug 2026; `RequestBodyLoggingMiddleware.cs`.

### Diagnosing a 33% duplicate rate as an architecture artifact

**Situation.** The production lead table shows 212,054 rows flagged `duplicate` — a third of everything the platform has recorded. Read cold, it looks like either a broken de-duplication layer or a serious lead-quality problem, and it is the first thing a technical interviewer would challenge.

**Problem.** Neither explanation is correct, and the data does not volunteer the real one. There is no `partial` flag persisted on the lead row, so a partial submission is indistinguishable from a full one after the fact. The cause had to be reconstructed from code and timing.

**Investigation.** Three steps. First, the code: `nextHandler()` in the quote form posts a partial submission mid-journey, `send.php` forwards a `partial` boolean, and the `leads` table has no column for a client-supplied correlation ID — so a second post cannot be matched to the first by identity. Second, the timing distribution: measuring gaps between consecutive submissions sharing mobile, type and source showed **94.1% of website duplicates arriving within 30 minutes of the original and 91% within five** — not plausible human behaviour. Third, the controlled comparison: the website channel, the only source running the multi-step form, shows a 48.6% sub-30-minute repeat rate against 0.2–6.7% for affiliate feeds that post a single complete lead. A 240× difference between the channel that has the mechanism and the channels that do not.

**Solution.** Analytical rather than code: establish the mechanism, quantify it, and restate the platform's numbers in terms of unique enquiries (309,486 platform-wide, 77,433 website, over the tenure) instead of raw rows. The code fix is specified in §11 and is already half-built — the client UUID exists in the newer verticals, it just is not forwarded to the lead API.

**Result.** A metric that read as a defect now reads as a known, bounded, explained trade-off with a costed fix. The honest unique-enquiry figures replace inflated raw counts throughout this dossier.

**Evidence.** `src/utils/index.ts` `nextHandler`; `plugin/api/functions/send.php`; `src/data/businessInsurance.ts:6`; the gap-distribution and channel-comparison tables in §8.6; `leads` schema showing no correlation-ID column.

### Paid-search click IDs lost across a multi-step journey

**Situation.** A visitor arrives with `?gclid=…`, then completes a form several pages and several minutes later.
**Problem.** The click ID must survive navigation, and Microsoft Ads uses `msclkid`, not `gclid`.
**Solution.** A 30-day cookie set on `init` for both parameters (both written to the `gclid` cookie name so downstream code needs one path); a theme-level script persisting the full query string to `localStorage`; and form code reading both back at submit time and injecting them into the payload.
**Result.** 149,940 production leads carry a click ID; website click-ID coverage sits at 63–73% in 2025–2026 against 35.7% in 2024.
**Evidence:** `set-cookie.php`; commits 2025-09-27, 2025-09-29, 2025-10-08, 2026-07-29; production `leads.gclid` counts.

---

## 13. Security, Reliability and Operational Considerations

**Present and verified:** WordPress nonce/CSRF validation on every state-changing plugin route; `x-api-key` authentication and `Authorization: Bearer` for Leela routes in the .NET API; per-source API keys mapped to companies with foreign-key integrity; invisible reCAPTCHA on the public competition and interest forms; PII redaction and phone censoring in logs; TLS-required Postgres with certificate trust configured; Bearer SAST gating every deployment workflow; Dependabot across the estate; PHPCS/PHPStan/PHPMD in the theme; a dedicated rollback workflow; stable `fail_reason` taxonomy enabling failure-mode monitoring; 20 indexes on `leads` including partial and covering indexes for the duplicate-check and dialler-queue access paths; 5,778 lines of xUnit tests covering phone validation, out-of-hours logic, timezones, JSON handling, Noetica enrolment, SMS, Brevo, and Leela contract/integration/regression suites.

**Observed weaknesses (recorded honestly, as they are also interview material):**
- The competition-wheel REST endpoint uses `'permission_callback' => '__return_true'` with no nonce check — it relies on reCAPTCHA alone.
- Four indexes on `leads` are marked `INVALID` in production, meaning failed concurrent index builds left artefacts that consume space without serving queries.
- Middleware duplication between the API and Functions front ends is a standing maintenance hazard.
- Secrets were committed historically and remediated iteratively (see §12).
- The WordPress repositories have no CI; deployment is a manual zip-and-upload.

---

## 14. Lessons Learned

**What worked.** Isolating per-vertical configuration (`data` / `types` / `validation` modules) from shared presentation components made twelve verticals tractable — new verticals reached production within days of the first commit. Proxying third-party calls through WordPress kept credentials out of the bundle and absorbed two vendor contract changes without a front-end release. A stable `fail_reason` taxonomy is why this audit could produce a reliability number at all; without it, "duplicate" and "integration failure" would be indistinguishable in the data.

**What did not.** Secret management was reactive — credentials reached the repository and took three commits to clear. Duplicating middleware across two front ends has cost real effort on every cross-cutting change. Shipping a dialler timing change before confirming it with the business cost a ship-revert-reship cycle; the corrective commit message explicitly records the confirmation step that was missing the first time.

**Operational lessons.** External systems fail partially, and the correct response is often to stop rather than retry — the enrolment service's refusal to re-create a CRM customer when the first attempt's outcome is unknown is worth more than an automatic retry would be. Logging usefully and logging safely are in genuine tension, and resolving it needs a deliberate design (trace IDs plus censoring plus gated raw capture), not a log-level change.

**The clearest lesson, and the one most worth telling.** If a workflow can produce the same logical event more than once, the identity of that event has to be decided at the point it starts — in the browser, at the first keystroke — not reconstructed afterwards by the server. The partial-submission design deferred that decision, and the cost compounds: a third of the production table is duplicate rows, correctness depends on time-window heuristics instead of identity, and every downstream count needs a caveat. The fix is small and was always small; what made it expensive was that the system had been live for years before anyone quantified the cost. Measuring it (§8.6) was the hard part, not fixing it.

**A lesson about inherited constraints.** The .NET rewrite had to mirror a Go service down to the byte layout of its identifiers. Working inside a parity constraint teaches something a greenfield build cannot: the previous system's behaviour becomes your specification, including its mistakes, and "improve it while you port it" is usually the wrong instinct because it destroys the one thing that makes the migration verifiable.

**What I would design differently today.** Generate a correlation UUID in the browser at the start of every quote journey, send it with both the partial and the final submission, and have the lead API upsert on it — reducing the de-duplication layer to a fallback for sources that cannot supply one. Extract the duplicated middleware into `Leads.Core` behind an abstraction both hosts implement. Put CI on the WordPress repositories — at minimum lint, type-check, and a built artefact — so deployment is not a manual zip. Add a nonce or signed-token check to the competition endpoint rather than relying on reCAPTCHA. Introduce a form-definition schema so a new vertical is data rather than a new component tree. Clean up the invalid indexes and consolidate the overlapping duplicate-check indexes.

---

## 15. Verified Career Claim Bank

> **Claim 1.** Principal engineer on the customer-facing web platform of a UK insurance broker, authoring **384 of 645 commits (59.5%)** across six production repositories since joining — including **88% of all commits to the quote-form plugin**, the surface every website lead passes through.
> **Evidence level:** VERIFIED · **Evidence:** `git shortlog -sne --all` and `git log --since=2024-11-01` across all six repositories · **Period:** Nov 2024 – Sep 2026 (~23 months) · **Qualification:** commit count is a proxy for contribution, not a measure of it. The 46%-of-831 lifetime figure is lower only because the repositories predate the employment; the tenure figure is the fair comparison.

> **Claim 2.** Built and maintained the React/TypeScript quote-form engine serving 12 insurance product verticals — ~30,000 lines across 169 files — embedded in WordPress via shortcode hydration.
> **Evidence level:** VERIFIED · **Evidence:** `src/` of `wp-quoteform-plugin`; 223/401 commits · **Period:** Jan 2025 – Mar 2026 · **Qualification:** they are the largest but not sole contributor; the original engine predates their work.

> **Claim 3.** Lead-capture forms they built and maintain feed a production pipeline that has ingested **397,667 submissions representing ~309,500 unique enquiries** during their tenure, of which **150,714 submissions (~77,400 unique enquiries)** originated from the company website.
> **Evidence level:** VERIFIED (raw counts) / DERIVED (unique counts) · **Evidence:** aggregate queries on production `leads` joined to `api_keys`, restricted to `created_at >= 2024-11-01`; unique counts via the 30-minute collapse method in §8.6 · **Period:** Nov 2024 – Oct 2026 · **Qualification:** the pipeline predates their involvement and carries affiliate traffic they did not build. Unique counts collapse the partial-submission artifact; the 30-minute window is a stated judgement call, stable across any threshold from 10 to 60 minutes.

> **Claim 3b.** Diagnosed a 33% production duplicate rate as an architectural artifact of partial form submissions rather than a data-quality failure, establishing the mechanism forensically from timing distributions and a cross-channel comparison (48.6% sub-30-minute repeat rate on the form-driven channel versus 0.2–6.7% on single-post affiliate feeds), and specified the correlation-ID fix.
> **Evidence level:** VERIFIED · **Evidence:** §8.6 gap-distribution and channel-comparison tables; `nextHandler` and `send.php` code paths; `leads` schema · **Qualification:** the analysis was performed during this audit; the design judgement and the half-built fix (client UUIDs in the newer verticals) predate it and are in the commit record.

> **Claim 4.** Shipped five new insurance verticals (Business Insurance, Minibus, Landlord, Plant & Machinery, Courier Van Route Flex) from first commit to first production lead in **4–17 days**.
> **Evidence level:** VERIFIED · **Evidence:** commit dates vs. `MIN(created_at)` per `type` in production · **Qualification:** small per-product volumes (5–437 leads) reflect niche products, not adoption failure.

> **Claim 5.** Business Insurance vertical, built July–August 2025, produced its first production lead on 4 August 2025 and has generated **437** website leads to date.
> **Evidence level:** VERIFIED · **Evidence:** commits 18 Jul–1 Aug 2025; `SELECT MIN(created_at), COUNT(*) … WHERE type='business-insurance'`.

> **Claim 6.** Built end-to-end paid-search attribution (Google `gclid` + Microsoft `msclkid` cookie capture, query-parameter persistence, injection into form payloads); **149,940** production leads carry a click ID.
> **Evidence level:** VERIFIED (count) / STRONGLY SUPPORTED (ownership) · **Evidence:** `set-cookie.php`, commits 2025-09-27 → 2026-07-29; `COUNT(gclid IS NOT NULL)` · **Qualification:** some click-ID capture predates their changes; they extended it to Microsoft Ads and to multi-step persistence.

> **Claim 7.** Implemented GTM dataLayer analytics for the quote journey, emitting structured `form_open` and product-specific submission events with risk and business attributes.
> **Evidence level:** VERIFIED · **Evidence:** `src/utils/analytics.ts`; commits 28–31 Aug 2025 · **Qualification:** downstream marketing impact is not measurable from this data.

> **Claim 8.** Took over maintenance of a production .NET 8 lead-ingestion API from a departing senior engineer and shipped PII-redaction middleware, `x-api-key` authentication across two front ends, an SMS notification service, and an idempotent dialler-enrolment service.
> **Evidence level:** VERIFIED · **Evidence:** 16 commits Nov 2025 – Sep 2026; `NoeticaEnrolmentService.cs` (292 LOC), `FleetSmsService.cs` (143 LOC), middleware diffs · **Period:** Nov 2025 – Sep 2026 · **Qualification:** they authored 16 of 123 commits; the bulk of the service is the prior engineer's work.

> **Claim 9.** Designed idempotent enrolment against an unreliable SOAP dialler API, explicitly distinguishing queued / in-flight / unknown-outcome states so a failed CRM call never produces a duplicate customer in a third-party system.
> **Evidence level:** VERIFIED · **Evidence:** `NoeticaEnrolmentService.Enrol` and its inline rationale comment; `NoeticaEnrolmentServiceTests.cs`.

> **Claim 10.** The pipeline they maintain operates at **~341 leads/day** (30,656 over the last 90 days) with a **1.30%** non-duplicate integration failure rate across 397,667 attempts during their tenure.
> **Evidence level:** DERIVED from verified counts · **Calculation:** 5,151 ÷ 397,667 for the tenure window; the lifetime equivalent is 5,866 ÷ 641,740 = 0.91% · **Qualification:** measures recorded failure reasons; requests that never reached the service are not represented. The tenure figure is quoted because it is the period of responsibility and is the more conservative of the two.

> **Claim 11.** The de-duplication layer withheld roughly **73,300 repeat website posts** during the tenure from generating a second outbound call — without it, close to half of all web enquiries would have been dialled twice.
> **Evidence level:** VERIFIED (count) / DERIVED (the "half" framing, from the 48.6% sub-30-minute rate) · **Qualification:** Abraham maintains but did not author the de-duplication engine, and most of what it suppresses is the partial-submission artifact described in §8.6 rather than genuinely distinct repeat customers. The agent-time saving is not quantifiable from this data.

> **Claim 16.** Participated in the architecture of a Go-to-.NET rewrite of a revenue-critical lead-ingestion service, delivered under a business-logic-parity constraint including byte-compatible identifier generation, and subsequently took over its maintenance.
> **Evidence level:** VERIFIED (the rewrite and parity constraint) / STRONGLY SUPPORTED (architectural participation) · **Evidence:** README "C# reimplementation of the original Go-based lead API, with business logic parity"; "mirroring Go implementation" comments across `MarketplaceService`, `Phone`, `LeadFactory` and the Noetica/Leela DTOs; `XidGenerator` as a byte-exact port of `rs/xid` · **Qualification:** the implementation was predominantly the senior engineer's (98 of 123 commits). The .NET choice was a CTO-level mandate, not Abraham's recommendation — worth stating plainly in interviews, since the interesting answer is what they would have chosen instead and why.

> **Claim 12.** Sole author of a WordPress calculator plugin delivering four marketing calculators, integrated with Advanced Custom Fields so non-technical staff place them from the editor.
> **Evidence level:** VERIFIED · **Evidence:** plugin header `Author: Abraham Akintola`; 23/26 commits; `shortcode.php`.

> **Claim 13.** Built the GDPR subject-access-request intake surface; **41** requests with **84** attachments received between 22 June and 5 October 2026.
> **Evidence level:** VERIFIED · **Evidence:** theme commit 22 Jun 2026; `gdpr_forms` / `gdpr_form_attachments` counts.

> **Claim 14.** Delivered an invisible-reCAPTCHA-protected promotional prize-wheel game and an interest-capture form, both integrated with Azure Functions and transactional email.
> **Evidence level:** VERIFIED · **Evidence:** 47/119 and 27/42 commits; `rest-api.php`, `azure-database.php` · **Qualification:** prize-wheel submission volume is not stored in the audited database and could not be measured.

> **Claim 15.** Works in a system integrating **8+ external services** (Noetica SOAP dialler, Leela identity service, Marketplace, QZLeadsSink, Brevo, CM Telecom, Ideal Postcodes, Google/Microsoft Ads) across **45** configured lead sources and **6** business entities.
> **Evidence level:** VERIFIED · **Evidence:** `Leads.Core/Services/`, `api_keys` and `companies` tables.

---

## 16. CV Achievement Candidates

### High-confidence quantified

*All figures scoped to the Nov 2024 – Oct 2026 tenure.*

- Principal engineer on the customer-facing web platform of a UK insurance broker — the company's primary customer-acquisition channel — authoring **384 of 645 commits (59.5%)** across six production repositories, including **88% of all commits to the quote-form plugin** that every website lead passes through.
- Built and maintained a **React 18 + TypeScript quote-form engine covering 12 insurance verticals** (~30,000 LOC, 169 files) embedded in WordPress via shortcode hydration, feeding a pipeline that has ingested **397,000+ submissions representing ~309,500 unique enquiries**, **~77,400** of them from the website.
- Shipped **five new insurance product verticals** from first commit to first live customer in **0–17 days**, each traceable to production volume in the lead database.
- Diagnosed a **33% production duplicate rate as an architectural artifact** of partial form submissions rather than a data-quality failure, proving the mechanism from timing distributions and a cross-channel comparison (**48.6%** sub-30-minute repeat rate on the form-driven channel versus **0.2–6.7%** on single-post feeds), and specified the correlation-ID fix.
- Engineered **end-to-end paid-search attribution** (Google and Microsoft click IDs, cookie + localStorage persistence through multi-step journeys) now present on **149,940 production leads**.
- Took over a production **.NET 8 / PostgreSQL lead-ingestion API** from a departing senior engineer and shipped PII-redaction middleware, API-key authentication across two deployment front ends, a timezone-aware SMS service, and an idempotent dialler-enrolment service — in a pipeline running **~341 leads/day at a 1.30% integration failure rate**.
- Authored a **WordPress calculator plugin** (four React calculators, ACF-driven placement) deployed to a site handling **~340 daily lead submissions**.

### High-confidence non-quantified

- Designed a **WordPress REST proxy layer (15 routes)** keeping Ideal Postcodes, vehicle-data, and lead-API credentials server-side, with nonce/CSRF validation on every state-changing request.
- Implemented **idempotent enrolment against an unreliable SOAP dialler API**, explicitly modelling unknown-outcome failures so a partially-failed call never creates a duplicate customer in a third-party CRM.
- Delivered **PII-safe observability** — trace-ID correlation, query-string sanitisation, phone censoring, and gated raw-body capture to blob storage — across both an ASP.NET Core and an Azure Functions host.
- Built the **GDPR subject-access-request intake** used by the compliance team, and **GTM dataLayer instrumentation** for the full quote journey.
- Remediated **committed credentials** across two public-facing plugins and added invisible reCAPTCHA to both.
- Took **sole ownership of five production systems** following the departure of the senior engineer, having previously maintained the front-facing site as a mid-level developer — a documented promotion in responsibility visible in the commit record.
- Participated in the architecture of a **Go-to-.NET rewrite** of a revenue-critical ingestion service delivered under a business-logic-parity constraint, including byte-compatible identifier generation so IDs already in production stayed sortable and format-identical.

### Requiring additional confirmation

- Any claim about **conversion rate, revenue, or cost per acquisition** — not derivable from the lead database.
- Any claim about **call-centre time saved** by duplicate suppression — needs agent-cost or call-handling data.
- Any claim about **the manual process replaced** by the platform — no evidence of the pre-platform workflow.
- Prize-wheel and interest-form **submission volumes** — these write to Azure endpoints outside the audited database.
- **Uptime or latency figures** — no monitoring data was accessible.

---

## 17. Portfolio Case Study Material

**Overview.** The public web platform of a UK commercial and niche motor insurance broker: a WordPress site whose React-based quote forms, calculators and promotional tools feed a .NET lead-ingestion API that routes every submission into a predictive dialler, a quoting marketplace, and a cross-system identity service.

**Context and challenge.** The broker sells at least twelve distinct insurance products, each needing a different question set and different downstream routing, and buys traffic from twenty-plus affiliate and aggregator partners as well as paid search. The platform must (a) let marketing launch a new product vertical in days, (b) stop the call centre ringing the same person twice, (c) keep a customer traceable from ad click through dialler to policy across three unrelated third-party systems, and (d) do all of this inside WordPress, where content editors control the pages the forms live on.

**My role.** Joined November 2024 as a mid-level developer owning the front-facing website; from late 2025, also owner of the backend lead API after the senior engineer left. 59.5% of all commits made across the estate during the tenure, and 88% of the quote-form plugin.

**Solution and architecture.** See §5. The defining shapes: React islands hydrated into WordPress pages by shortcode; a PHP REST proxy that keeps every third-party credential server-side; a shared .NET core library behind two redundant hosts; and a lead pipeline that validates, de-duplicates, persists, and fans out to six external services with a stable failure taxonomy.

**Interesting engineering problems.** Twelve verticals in one bundle without twelve codebases. Idempotency against a SOAP API whose partial failures have unknown external outcomes. Attribution that survives a multi-step, multi-minute journey. Business-hours logic that must be timezone-correct and testable. Logging enough to debug a lead without logging the person.

**Measurable production evidence.** Over the Nov 2024 – Oct 2026 tenure: 397,667 submissions representing ~309,500 unique enquiries, ~77,400 of them from the website; 1.30% non-duplicate failure rate; ~341 leads/day currently. Platform-wide: 101 product types; 45 configured sources across 6 business entities; 99.05% dialler-CRM assignment; 72.04% cross-system identity assignment. Five verticals traceable from commit to first live customer in 0–17 days. The 33% duplicate rate diagnosed as a partial-submission artifact rather than a data defect (48.6% form-channel repeat rate versus 0.2–6.7% on single-post feeds).

**Results and lessons.** See §9 and §14.

**What I would improve next.** De-duplicate the middleware; put CI on the WordPress repositories; replace per-vertical component trees with a form-definition schema; add proper request authentication to the competition endpoint; clean up the four invalid Postgres indexes.

---

## 18. Interview Story Bank

**Ownership / inheriting a system (STAR).**
*Situation:* The senior engineer who owned the .NET lead API — the revenue-critical path for every lead the business receives — left. *Task:* Take over a 15,000-line .NET 8 solution with SOAP, HTTP and database integrations, having previously owned only the frontend. *Action:* Started with contained, high-value changes rather than refactors — an SMS notification service with injectable time for testability, then PII redaction across both hosts, then API-key authentication, then a new enrolment service with its own test suite. *Result:* 16 commits over ten months covering security, observability, a new service, and a dialler-routing change; the service continues to run at ~341 leads/day. *Evidence:* commit history Nov 2025 – Sep 2026; `NoeticaEnrolmentServiceTests.cs`.

**Correctness over convenience (STAR).**
*Situation:* Enrolling a lead into the dialler requires creating a CRM customer, then queueing it — two SOAP calls against a system with 314 recorded connection failures in production. *Task:* Make repeat enrolment safe. *Action:* Modelled three distinct prior states — already queued, in flight with no failure, and failed with no customer ID — and chose to return `failed` in the third case rather than retry, because the external outcome is unknown and a retry could create a duplicate customer in a system we cannot clean up. *Result:* no duplicate CRM customers; recovery is an explicit decision rather than an accidental one. *Evidence:* `NoeticaEnrolmentService.Enrol` and its inline comment.

**Shipping, being wrong, and verifying (STAR).**
*Situation:* A change to dialler latest-contact-time rules for the Convicted Driver campaign list. *Task:* Apply the business's requested timing change. *Action:* Shipped a 30-minute latest time for list 150; it was wrong; reverted it the same day; then implemented the actually-confirmed change (list 53 into the five-day list, list 150 untouched) with the confirmation recorded in the commit message. *Result:* correct behaviour restored same-day, and a changed process — confirm the rule with the business before shipping, not after. *Evidence:* three commits, 25 Sep 2026.

**Security remediation (STAR).**
*Situation:* The competition-wheel plugin had Azure Function and mail API credentials in the repository. *Task:* Remove them and prevent recurrence. *Action:* Three commits over two weeks moving configuration behind an environment accessor, with the same fix applied to the sibling interest-form plugin; added invisible reCAPTCHA to both public endpoints. *Result:* credentials out of source. *Honest reflection:* it took three passes, which is the strongest argument for pre-commit secret scanning — and rotating a leaked key is a separate step that removal alone does not cover. *Evidence:* commits 3/9/17 Jan 2025.

**Scaling a frontend without scaling the team (STAR).**
*Situation:* Twelve insurance verticals, each with a distinct question set, all needed in one WordPress-embedded bundle. *Task:* Keep it maintainable while stakeholders change individual questions weekly. *Action:* Split every vertical into `data` / `types` / `validation` modules against a shared `ParentForm`, shared field components, and a `formSteps` map; added a `StepIndicator` across all verticals in a single day. *Result:* five new verticals reached production within 4–17 days of first commit. *Evidence:* repository structure; commit dates vs. first production lead per type.

**Observability under privacy constraints (STAR).**
*Situation:* Inherited middleware logged raw lead payloads containing names, phones, emails, and dates of birth. *Task:* Keep debuggability, remove PII from logs. *Action:* trace-ID correlation on every request, sanitised query strings, `Phone.CensorPhone()` in phone-adjacent logs, and raw-body capture gated behind a flag and restricted to two routes, written fire-and-forget to blob storage rather than the log stream — mirrored in both hosts. *Result:* lead failures remain diagnosable by trace ID without PII in log analytics. *Evidence:* commit 28 Aug 2026; `RequestBodyLoggingMiddleware.cs`.

**Forensic debugging / challenging a metric (STAR).**
*Situation:* The production lead table showed a 33% duplicate rate — the kind of number that reads as a broken system. *Task:* Establish whether it was a defect, and what the real enquiry volume was. *Action:* Traced the quote form's partial-submission path in code; found no client-generated correlation ID, so the final post could not be matched to the partial one. Measured gap distributions between consecutive identical submissions and found 94% of duplicates arriving within 30 minutes, 91% within five — not human behaviour. Then ran the controlled comparison: the form-driven channel showed a 48.6% sub-30-minute repeat rate against 0.2–6.7% for affiliate feeds that post once. *Result:* The mechanism was established beyond doubt, the platform's real figure restated as ~309,500 unique enquiries rather than 397,667 raw rows, and a bounded fix specified — forward the client UUID that the newer forms already generate, add a column, upsert on it. *Evidence:* §8.6 tables; `nextHandler`; `send.php`; `leads` schema.
*Why this is the strongest story in the set:* it shows reading a system through its data rather than its code alone, using a control group to test a hypothesis, and arriving at a number that is lower but defensible.

**Working inside a constraint you disagree with (STAR).**
*Situation:* The lead API was rewritten from Go to .NET. *Task:* Contribute to the architecture of a rewrite of the service every lead depends on. *Action:* Participated in the design discussions; the .NET choice was a CTO-level mandate rather than a technical consensus, and would not have been their own pick. Worked within it, and the parity constraint it implied — identifier generation ported byte-for-byte from Go's `rs/xid` so existing production IDs stayed sortable, and business logic mirrored closely enough that the comments still say so. *Result:* The service runs at ~341 leads/day with a 1.30% non-duplicate failure rate. *Follow-up to be ready for:* "What would you have chosen?" — the honest answer is about the correlation-ID design (§11) more than the language, since that is the decision that actually shaped the data.

**Technical disagreement / trade-off discussion.** The two-front-end redundancy design means every cross-cutting change is made twice — their own PII and auth commits each touched both middleware trees. A good interview answer discusses when duplication-for-isolation is worth it and what the extraction into `Leads.Core` would cost.

---

## 19. Technical Article and Blog Opportunities

**1. "React islands in WordPress: hydrating a 12-step insurance form inside someone else's CSS."**
*Problem:* shipping a large React application into pages that content editors control, alongside Bootstrap and a theme stylesheet. *Why interesting:* most React content assumes you own the page. *What to explain:* shortcode → marker div → deferred hydration, nonce passing through the footer, the specific class of styling regressions this causes. *Anonymise:* product names, internal route names. *Do not publish:* the REST route list, nonce implementation details.

**2. "When retrying is the wrong answer: idempotency against an API whose failures are ambiguous."**
*Problem:* a two-call enrolment where the first call may or may not have succeeded. *Why interesting:* the common advice is "retry with backoff"; here retrying creates uncleanable duplicate records in a third-party CRM. *What to explain:* modelling prior state explicitly, and why returning `failed` beats an automatic retry. *Anonymise:* the dialler vendor, campaign list IDs. *Do not publish:* SOAP action names, queue-state constants.

**3. "Porting an ID generator across languages when the old IDs still have to sort."**
*Problem:* a Go service's `rs/xid` identifiers already in production; a C# replacement must produce byte-compatible, lexicographically sortable IDs. *Why interesting:* a concrete, bounded example of backwards compatibility during a language migration. *Note:* this code is a colleague's work — credit it, write about the problem class, not the authorship.

**4. "Logging enough to debug a lead without logging the person."**
*Problem:* PII-heavy payloads and a need for production diagnosability. *What to explain:* trace IDs as the unit of correlation, censoring at the formatter boundary, gated raw capture to cold storage. *Do not publish:* blob container names, environment variable names, which routes are gated.

**5. "Attribution that survives a ten-minute form."**
*Problem:* click IDs must cross a landing page, several steps, and a page reload. *What to explain:* the cookie-plus-localStorage pattern, why `msclkid` folds into the same storage key, the measurement caveat that correlation between a code change and coverage improvement is not causation. *Anonymise:* absolute lead volumes.

**6a. "Your duplicate rate is not a data-quality problem: a forensic read of 400,000 production rows."**
*Problem:* a third of a production table flagged duplicate, with no field recording why. *Why interesting:* it is a complete worked example of diagnosing a system from its data — code reading to form a hypothesis, a timing distribution to test it, and a cross-channel comparison as the control group. The 240× difference between the channel with the mechanism and the channels without is the kind of result that makes a convincing article. *What to explain:* why partial submissions without a client correlation ID necessarily produce duplicate rows; how to choose a collapse window and show the result is stable across it; why the honest number is the lower one. *Anonymise:* absolute volumes, all partner names (use "affiliate feed A/B/C"). *Do not publish:* the de-duplication rules themselves, or the real channel names.

**6b. "Decide identity at the start, not the end."**
*Problem:* any workflow that can emit the same logical event twice needs that event's identity fixed at the moment it begins. *Why interesting:* a short, opinionated piece with a concrete cost attached — years of duplicate rows and heuristic de-duplication, against a UUID generated in the browser. *What to explain:* the upsert-on-correlation-ID pattern and why the server cannot reconstruct identity reliably after the fact. *Anonymise:* everything business-specific; this one is almost entirely generalisable.

**6. "33% of your leads are the same person: designing de-duplication that is allowed to be wrong."**
*Why interesting:* the rules are full of deliberate carve-outs — priority products, cancelled prior leads, per-product time windows — each trading a false duplicate against a lost genuine lead. *Do not publish:* the specific rules, windows, or product-type priorities; these are commercially sensitive. Write it at the level of the design problem only.

---

## 20. LinkedIn / Social Content Opportunities

Each of these has a real story behind it — no generic motivation posts.

1. **"Five product launches, 4 to 17 days from first commit to first live customer."** The underlying evidence is commit dates against `MIN(created_at)` per lead type. The honest framing is that the architecture — per-vertical config modules against shared components — is what made the speed possible, and that the products are niche so the volumes are small.
2. **"I inherited the revenue-critical service when the senior engineer left. Here's where I started."** Evidence: the first four commits were an SMS service, PII redaction, auth, and a new service with tests — contained, verifiable changes rather than a refactor.
3. **"Sometimes the right response to a failed API call is to stop."** The enrolment idempotency story. Genuinely counter-intuitive, and the code comment makes the reasoning concrete.
4. **"We shipped it, it was wrong, we reverted it the same day, then we confirmed it and shipped it properly."** The Noetica list-timing sequence, visible in three commits. A credible, un-heroic story about process.
5. **"Three commits to remove one secret."** The 2025 remediation. The useful lesson is pre-commit scanning plus key rotation, not "be careful".
6. **"A third of our leads are duplicates — and that's a design output, not a data-quality problem."** Explain why persisting duplicates with a reason code beats rejecting them, and what it enables in reporting.
7. **"Twelve insurance products, one React bundle."** Architecture post on splitting config from presentation. The commit history is the proof that the weekly question-level churn is real.
8. **"A third of our leads looked like duplicates. They weren't."**  The forensic post: hypothesis from code, timing distribution as evidence, affiliate channels as the control group, and a smaller but honest headline number at the end. The 48.6%-versus-0.2% contrast is the hook, and it is real.
9. **"Decide identity at the start, not the end."**  The one-idea version of the same story, with the cost attached.
10. **"Two years maintaining the same site teaches you things a greenfield project never will."** Grounded in the WhatsApp-widget removal (a marketing tool reversed for a Google Ads policy conflict) and the recurring styling regressions from React-in-WordPress.

---

## 21. Private vs Public-Safe Information

### PRIVATE — career evidence only, do not publish

- Database host, credentials, schema details, index names, and table row counts.
- Absolute production lead volumes by source and by partner (641,740; 205,806; the per-source table in §8.3) — partner volumes are commercially sensitive.
- Partner and aggregator names (Quotezone, QuoteSearchUK, Insurance1, Leads2Market, Databowl, Policylink and others) and their relative volumes.
- Internal system names and identifiers: Leela, Noetica campaign list IDs (53, 150, 41, 32, 149, 35, 152, 151), queue-state constants, OGI/Acturis, QZLeadsSink, Marketplace, the company codes in `companies`.
- De-duplication rules, carve-outs, and time windows.
- The `quote/v1/*` route inventory and the nonce/CSRF implementation.
- Environment variable names, blob container names, Azure registry and resource names.
- The specific security weaknesses listed in §13 (invalid indexes, `__return_true` permission callback, historical secret exposure) — these are defect disclosures about a live system.
- Colleague names and their commit shares.
- All lead-record contents; no PII was retrieved and none should ever be.

### PUBLIC-SAFE — adaptable for portfolio, LinkedIn, CV, articles

- Role framing: principal engineer on a UK insurance broker's customer-facing web platform, 2024–2026; majority author across four of six repositories.
- Technology stack as listed in §7, with vendor-specific internal services described generically ("a predictive dialler", "an internal cross-system identity service", "a quoting marketplace").
- Architecture at the pattern level: React islands in WordPress, a server-side REST proxy for third-party credentials, a shared core library behind redundant hosts, a lead pipeline with validation / de-duplication / fan-out and a stable failure taxonomy.
- Rounded, non-attributable scale statements: "a lead pipeline handling hundreds of thousands of submissions"; "~340 leads a day"; "sub-1% integration failure rate"; "twelve insurance product verticals"; "five new verticals shipped in days".
- Engineering stories at the design-problem level: idempotency with ambiguous failures, PII-safe observability, attribution across multi-step journeys, config-vs-presentation separation for many product variants.
- Honest process stories: the ship-revert-confirm-reship cycle; the iterative secrets remediation; the marketing widget reversed for an ads-policy conflict.

**When in doubt, treat it as private.** In particular: do not publish exact lead counts, any partner name, any campaign list ID, or any of the defect findings in §13.

---

## 22. Evidence Gaps and Further Metrics Worth Collecting

| Gap | Why it matters | Where it might exist | Question to ask internally |
|---|---|---|---|
| **Conversion rate — leads to bound policies** | Turns a volume metric into a business-outcome metric; the single strongest missing number | Acturis/OGI policy data joined to `leela_id`; the `lead-reporting` tool in the adjacent workspace | "Can we get quote-to-policy conversion by lead source and product for 2025–2026?" |
| **Call-centre time saved by repeat-post suppression** | ~73,300 suppressed website repeats during the tenure × average call-handling cost would quantify the operational win | Dialler call records; call-centre WFM data | "What's the average handled-call duration and loaded agent cost?" |
| **Pre-platform process** | Lets you claim "replaced X" rather than "built Y" | Institutional memory; pre-2024 documentation | "How were web leads handled before the Go service?" |
| **Partial-submission conversion rate** | How many partial submissions never become full ones is the entire business case for the partial-submission design — and would show whether the trade-off was worth its cost | Derivable from the same gap analysis in §8.6 with GTM `partialSubmit` event data alongside it | "Can we get `partialSubmit` versus full-submit event counts from GTM for a representative month?" |
| **Exact production launch dates** | Current evidence starts at the data, not the deployment | Azure deployment history; `leads_backup` table; GitHub release tags | "When did the current site and lead API first go live?" |
| **Competition-wheel and interest-form volumes** | Two delivered products with zero measurable usage in this audit | The Azure Function's own storage; the promotions mailbox | "How many prize claims and interest registrations have those endpoints received?" |
| **Uptime, latency, error rates** | No reliability evidence beyond `fail_reason` counts | Azure Application Insights / Monitor | "Can I get App Service availability and p95 latency for the lead API?" |
| **Ad-spend and CPA alongside click-ID coverage** | Would let the attribution work be defended causally rather than correlationally | Google/Microsoft Ads accounts; marketing reporting | "Did reported attributable conversions change after Sep 2025?" |
| **Why website volume declined through 2026** | Protects against a hostile interview reading of the monthly trend | Marketing budget history; channel mix reporting | "Was there a budget or channel change in 2026?" |
| **Infrastructure cost** | Cost-efficiency claims | Azure cost management | "What does the lead API + database cost monthly?" |
| **Stakeholder feedback** | Qualitative evidence of ownership and delivery | Managers, marketing, call-centre leads | Ask for a short written note on the handover and on specific launches. |
| **Pre/post performance of any optimisation** | The 20 indexes on `leads` suggest tuning work happened; no before/after exists | Git history of the lead API; any prior incident notes | "Was there a query-performance incident, and is there a record of it?" |

---

## 23. Appendix — Evidence Notes

**Method.** Six repositories were inspected: structure, READMEs, package manifests, source, configuration, CI workflows, Dockerfile, and full `git log --all` with authorship. Production evidence came from the PostgreSQL 16 database named in the `Leads.Api` launch profile, connected with `default_transaction_read_only = on` and a statement timeout. All queries were `SELECT` with aggregate functions; no DDL, DML, migration, or index operation was executed, and no raw lead rows were retrieved.

**Repository identity.** `insurancerevolution/wp-theme`, `insurancerevolution/wp-quoteform-plugin`, `insurancerevolution/wp-register-interest-form`, `insurancerevolution/wp-competition-wheel`, `Insurance-Revolution/wp-calculator-plugin`, `Insurance-Revolution/lead-api-dotnet`.

**Key queries used (reproducible).**

```sql
-- lifetime volume and window
SELECT count(*), min(created_at)::date, max(created_at)::date FROM leads;

-- annual volume and outcomes
SELECT date_trunc('year',created_at)::date, count(*),
       count(*) FILTER (WHERE processed),
       count(*) FILTER (WHERE duplicate),
       count(*) FILTER (WHERE fail_reason IS NOT NULL)
FROM leads GROUP BY 1 ORDER BY 1;

-- integration coverage
SELECT count(*) FILTER (WHERE leela_id IS NOT NULL),
       count(*) FILTER (WHERE noetica_customer_id IS NOT NULL),
       count(*) FILTER (WHERE queue_id IS NOT NULL),
       count(*) FILTER (WHERE cancelled_at IS NOT NULL),
       count(*) FILTER (WHERE gclid IS NOT NULL)
FROM leads;

-- failure taxonomy
SELECT fail_reason, count(*) FROM leads
WHERE fail_reason IS NOT NULL GROUP BY 1 ORDER BY 2 DESC;

-- website channel by product, with first-seen dates
SELECT l.type, count(*), min(l.created_at)::date, max(l.created_at)::date
FROM leads l JOIN api_keys k ON k.id = l.source
WHERE k.source = 'insurancerevolution.co.uk'
GROUP BY 1 ORDER BY 2 DESC;

-- current throughput
SELECT count(*), round(count(*)/90.0,1) FROM leads
WHERE created_at >= now() - interval '90 days';

-- TENURE SCOPING (all career claims use this window)
SELECT count(*), count(*) FILTER (WHERE duplicate),
       count(*) FILTER (WHERE fail_reason IS NOT NULL AND fail_reason <> 'duplicate')
FROM leads WHERE created_at >= '2024-11-01';

-- UNIQUE ENQUIRIES: collapse repeat posts inside a 30-minute window (§8.6)
WITH w AS (
  SELECT created_at - lag(created_at)
           OVER (PARTITION BY mobile, type, source ORDER BY created_at) AS gap
  FROM leads WHERE created_at >= '2024-11-01'
)
SELECT count(*) AS raw_rows,
       count(*) FILTER (WHERE gap IS NULL OR gap >= interval '30 minutes') AS unique_enquiries,
       count(*) FILTER (WHERE gap <  interval '30 minutes') AS sub30min_artifacts
FROM w;

-- THE CONTROL GROUP: repeat rate per channel. The form-driven channel should
-- stand far above single-post affiliate feeds if the partial-submission
-- hypothesis holds. It does: 48.6% versus 0.2-6.7%.
WITH w AS (
  SELECT k.source,
         l.created_at - lag(l.created_at)
           OVER (PARTITION BY l.mobile, l.type, l.source ORDER BY l.created_at) AS gap
  FROM leads l JOIN api_keys k ON k.id = l.source
  WHERE l.created_at >= '2024-11-01'
)
SELECT source, count(*) AS raw,
       count(*) FILTER (WHERE gap < interval '30 minutes') AS sub30,
       round(100.0 * count(*) FILTER (WHERE gap < interval '30 minutes') / count(*), 1) AS pct
FROM w GROUP BY 1 HAVING count(*) > 2000 ORDER BY pct DESC;
```

**Authorship command.** `git -C <repo> shortlog -sne --all`, with the two Git identities `AbrahamNobleOX <abramseyakid@gmail.com>`, `Abraham <abramseyakid@gmail.com>` and `AbrahamAkintola-InsuranceRevolution <abraham.akintola@insurancerevolution.co.uk>` treated as one person.

**Limitations of this audit.**
1. Commit counts measure activity, not impact; a one-line config commit and a 2,000-line feature count the same.
2. The `leads` table's earliest row is 9 Jan 2024, which bounds the measurable period but is not necessarily the system's first production day.
3. Four `INVALID` indexes on `leads` indicate failed concurrent index builds; they were observed, not investigated, and nothing was changed.
4. Outcomes beyond lead ingestion — quotes produced, policies bound, revenue — are in systems not accessible from this database.
5. The competition wheel and register-interest form write to Azure endpoints outside this database; their production usage could not be measured at all.
6. No monitoring, logging, or deployment telemetry was available, so no uptime, latency, or deployment-frequency figures appear anywhere in this report.
7. Causation is never claimed where only correlation is observable — notably the click-ID coverage change and the 2026 volume decline.
8. The unique-enquiry figures depend on a 30-minute collapse window, which is a judgement call. The gap distribution is concentrated enough (91% of repeats inside five minutes) that the result is stable across any threshold from 10 to 60 minutes, but it cannot separate a partial-then-full pair from a genuine immediate resubmission after an error, and one person using two phone numbers still counts twice.
9. No `partial` flag is persisted on the lead row, so partial and full submissions cannot be distinguished directly in the data; §8.6's conclusion rests on code plus timing plus the cross-channel control, not on a stored field.
10. Employment start date (November 2024) was supplied by the subject and corroborated against the Git record, where all 384 of their commits fall on or after 1 Nov 2024. The exact start date within that month is not independently verifiable from the evidence here.

**Security hygiene of this audit.** No credential, connection string, API key, token, customer name, phone number, email address, or any other personal datum appears anywhere in this dossier or its companion files.

---

*End of dossier.*
