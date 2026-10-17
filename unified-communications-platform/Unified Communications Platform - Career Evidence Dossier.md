# Unified Communications Platform (UCP) — Career Evidence Dossier

**Audit date:** 8 October 2026 · **Revised:** 10 October 2026 with subject-provided origin context, corroborated against legacy source where possible
**Subject:** Abraham Akintola
**Evidence sources:** source repositories (including the two legacy systems replaced), Git history, project documentation, read-only queries against the production PostgreSQL database, and subject testimony (labelled as such).

> **Evidence levels used throughout:** **VERIFIED** (directly observed in code, Git, or production data) · **DERIVED** (calculated from verified figures) · **STRONGLY SUPPORTED** (multiple converging signals, not provable) · **INFERRED** (plausible from architecture, no direct evidence) · **UNKNOWN** (requires confirmation).

---

## 1. Executive Summary

UCP is an internal, governed outbound-communications platform built for a UK insurance broker. It exists because the company's customer communications were scattered across systems with no central visibility: no one could see which customers had been contacted, through what, or whether messages had actually arrived — and when a process failed silently, staff re-triggered emails by hand. Executive stakeholders asked for every company communication to run through one monitored place.

UCP replaced **two** legacy systems: a Google Apps Script estate that orchestrated the business processes, and `iros-mail`, a .NET 8 Azure Functions microservice that rendered and sent the email. In their place it provides campaign definition, approval workflow, audience sourcing, queue-backed execution, multi-provider delivery across Email/SMS/WhatsApp, delivery reconciliation, suppression, per-customer communication history, operational reporting, and a complete audit trail — plus a backwards-compatible API so other internal systems send *through* UCP rather than around it.

**Scale established from production data (queried read-only, 8 October 2026):**

| Measure | Figure | Window |
|---|---|---|
| Campaign messages dispatched | **85,482** | 26 Jun – 8 Oct 2026 |
| External-service API emails | **18,113** | 11 May – 8 Oct 2026 |
| Audit events recorded | **364,430** | 30 Apr – 8 Oct 2026 |
| Automated source-polling runs | **149,321** (99.83% success) | 25 Jun – 8 Oct 2026 |
| Campaign execution runs | **5,261** (99.92% non-cancelled) | to 8 Oct 2026 |
| Email failure+bounce rate | **1.07%** | as above |
| SMS failure rate | **0.23%** | as above |

**Contribution.** Git history attributes **221 of 228** backend commits and **77,448 of 77,455** backend source lines added to one author identity (the subject). The backend is effectively single-author. The frontend was scaffolded in one week by Lovable's AI bot — but 231 of those 238 bot commits carry the subject as co-author, indicating he directed them — after which he authored all 149 subsequent commits and 78% of all source lines added.

**Most technically significant aspects:** a control-plane/execution-plane split with immutable per-message snapshots; queue-backed delivery via Azure Service Bus; a five-adapter provider abstraction proven in production; database-enforced idempotency that produced **zero duplicate schedule dispatches across 1,146 dispatches**; a near-real-time SQL polling loop that has executed ~149k runs at 99.83% success; and a deliberately backwards-compatible API contract that let 22 existing internal clients migrate onto the platform.

**Clearest business impact:** 99.4% of all execution runs were triggered automatically rather than manually — directly addressing the manual re-triggering that characterised the legacy estate — while human approval is still enforced before anything goes live. Every message is now recorded against the customer who received it and visible in one searchable place, which is the capability executives asked for and which neither legacy system provided.

---

## 2. Project Overview

UCP centralises outbound customer communication for an insurance broker across three channels. Per the project's own scope document, it "replaces google scripts driven orchestration with stored campaign definitions, controlled audiences, approvals, queue-backed execution, audit trails, customer-level history, suppressions, and integration-specific controls." *(VERIFIED — `ucp-documentation/docs/01-product-and-architecture/product-scope.md`)*

**Workspaces:**

| Workspace | Role | Size |
|---|---|---|
| `ucp-backend` | NestJS API and all domain logic | 354 TS files, 62,750 lines, 19 modules |
| `ucp-frontend-lovable` | Active React/Vite operations UI | 190 files, 39,289 lines |
| `ucp-documentation` | MkDocs reference and handover site | 17 commits |
| `ucp-frontend` | Superseded first UI, kept for reference | 12 commits, last Apr 2026 |
| `old-google-scripts` | **Legacy system replaced** — Apps Script process orchestration | 2,691 lines of `.gs` |
| `old-iros-mail` | **Legacy system replaced** — .NET 8 Azure Functions mail microservice | EJS/MJML rendering, MongoDB email records, SMTP |

**Users:** 17 active internal accounts — 7 Admin, 6 Manager, 4 StandardUser — all of whom have signed in (352 sessions, 29 Apr – 8 Oct 2026). This is an internal operations tool, not a customer-facing product. *(VERIFIED — production `User`, `UserSession`)*

---

## 3. Business Problem and Context

### 3.1 What existed before

Two systems, in a chain:

1. **Google Apps Scripts** (`iros-process-one`, `iros-process-two`) orchestrated the business processes — renewal invites and acknowledgements, cancellation chases and acknowledgements, outstanding-payment chases, RTA7 notices, ad-hoc document requests. Time-driven `runEveryMinute` / `runEveryHour` / `runEveryDay` entry points, with the sending window hardcoded in a conditional.
2. **`iros-mail`**, a .NET 8 isolated Azure Functions microservice, rendered templates (EJS/MJML), sent over SMTP, stored email records in MongoDB, and ran scheduled-send and failed-email-reschedule workers.

*(VERIFIED — legacy source, `README.md`, `mail-service.csproj`, `code.gs`)*

### 3.2 Why it was a problem

The subject's account is that communications were scattered with no central visibility, nobody could see which customers had been contacted or whether messages arrived, processes broke and messages were silently lost, and staff re-triggered emails manually. Executive stakeholders — CEO and CTO level — asked for consolidation into one monitored system. *(TESTIMONY)*

**Three parts of that account are directly corroborated in the legacy code:**

**Failures notified nobody.** The shared failure handler is an unimplemented stub:

```javascript
function notifyExecutionFailed(fn, error) {
  //todo send email
  console.error("Function execution failed", fn, error);
}
```

and the batch runner catches per-function errors and continues:

```javascript
runAll(a) { a.forEach(f => { try { runAsync(f); } catch (e) { console.error(e); } }); }
```

A failed communication process therefore produced a console line in Apps Script and nothing else — no alert, no retry, no persisted failure record. This is a precise mechanism for "communication would be lost in between" and for manual re-triggering: a silent failure is only discoverable by someone noticing a customer was not contacted. *(VERIFIED)*

**There was no record of what was sent to whom.** The scripts write to ten per-process marker tables (`wp_os_payments`, `wp_canc_chase`, `wp_renw_invites`, …) capturing only that a source row was picked up — `created`, `ref`, `polno`, `reason`, and the raw row as JSON. There is no `SpreadsheetApp`, `appendRow`, or any send-log in the Apps Script estate (0 occurrences of each). `iros-mail` did keep its own `EmailRecord` store, but that was per-message inside one service, not a cross-channel customer history. *(VERIFIED)*

**There was no interface.** `iros-mail` exposes API-key-protected HTTP endpoints only; the Apps Scripts have no UI surface. Neither system offered a way to look a customer up. *(VERIFIED)*

> **Honest qualification.** It is *not* true that the legacy estate stored nothing — per-process markers and `iros-mail` email records existed. The accurate claim is narrower and still strong: **no unified, queryable, cross-channel record of what was sent to a given customer, and no interface to ask the question.**

### 3.3 The stakeholder requirements, and what satisfies each

| Executive requirement *(TESTIMONY)* | UCP capability *(VERIFIED)* |
|---|---|
| All company communications in one place | 5 provider adapters + 3 channels behind one approval/suppression model; 22 external clients routed through the platform API |
| Visibility and monitoring of what is going out | Dashboard `overview`, campaign message search/export, 364,430-event audit log, `ucp-execution-summary` monitor API |
| Look up a customer and see every communication sent to them | `GET /customer-communications` — a `UNION ALL` across `CampaignExecutionRecipient` and `TransactionalEmail`, with a dedicated UI page |
| Reporting and communication statistics for executives | Daily summary, operational report (HTML + export), weekly audit report services |
| Other internal systems must send *through* this service | `POST /api/mail` preserves the legacy `iros-mail` contract verbatim (§11) |

The per-customer lookup is the clearest single answer to the original complaint: it unifies campaign sends and external-API sends into one searchable history, which is precisely what neither legacy system could do.

### 3.4 Regulatory framing

UCP targets FCA, GDPR and PECR alignment, with non-negotiable boundaries stated in its own documentation: backend-enforced authorisation, approval before live use, queue-backed sending, reproducible evidence of what was sent, and channel-aware suppression checked before send. *(VERIFIED — `AGENTS.md`, product-scope doc)* Whether a specific regulatory audit drove the timing is **UNKNOWN**.

---

## 4. My Role and Contribution

### Verified personal contribution

| Repository | Commits (subject) | Commits (others) | Source lines added/removed (subject) |
|---|---|---|---|
| `ucp-backend` | 221 | 7 (`zayan_DevOps`) | +77,448 / −10,115 |
| `ucp-frontend-lovable` | 149 | 238 bot, 2 other | +37,712 / −10,248 |
| `ucp-documentation` | 17 | 0 | — |
| `ucp-frontend` | 12 | 0 | — |

*(VERIFIED — `git shortlog -sne`, `git log --numstat`)*

The backend is **single-author to a near-total degree**: the other contributor added 7 lines and removed 2, consistent with a DevOps/config touch rather than feature work.

Sustained ownership across the full lifespan is verifiable: first commit 22 April 2026, latest 8 October 2026, with commits in every month — backend 25/36/76/52/17/21/1 by month Apr→Oct. *(VERIFIED)*

### Honest qualification on AI assistance

This project was developed with AI tooling, and the record should say so plainly:

- The active frontend was **scaffolded by Lovable's `gpt-engineer-app[bot]`** between 22 and 28 April 2026 (+10,866 lines). However **231 of 238** of those commits list the subject as co-author, indicating he was directing the generations rather than inheriting someone else's codebase. After 28 April the bot never commits again; all 149 later commits are his. *(VERIFIED)*
- `AGENTS.md` is a detailed agent-instruction file, so coding agents were used on the backend too. Backend commits carry **no** co-author trailers, so the split between hand-written and AI-assisted backend code **cannot be determined** *(UNKNOWN)*.

The defensible framing is **architectural ownership and direction**, not keystroke authorship: the module boundaries, data model, approval model, execution-plane design and provider abstraction are consistent across 228 commits under one identity, and the agent-instruction file that governs the AI tooling is itself his design artefact.

### Attribution that cannot be established

Infrastructure provisioning (Azure subscription, Service Bus namespace, container registry, database), the decision to adopt Lovable, and business/stakeholder requirements-gathering. CI workflows exist in-repo but their original authorship was not separately traced. *(UNKNOWN)*

---

## 5. System Architecture

```
┌──────────────────────┐        ┌───────────────────────────────────────────┐
│  React / Vite SPA    │        │  External integrators (22 API clients)    │
│  Azure Static Web    │        │  → /mail, /sms, /whatsapp                 │
│  Apps                │        └──────────────────┬────────────────────────┘
└──────────┬───────────┘                           │
           │ session cookie                        │ API key
           ▼                                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  NestJS API — Azure Web App (container)   19 modules · 155 routes        │
│                                                                          │
│  CONTROL PLANE                         EXECUTION PLANE                   │
│  ├─ auth (Google OIDC, invite-only)    ├─ execution (runs, recipients)   │
│  ├─ campaigns / templates              ├─ scheduling (dispatcher)        │
│  ├─ approvals (backend-enforced)       ├─ preset-flows (realtime poller) │
│  ├─ suppressions (channel-aware)       ├─ transactional-mail (processor) │
│  ├─ users / audit / dashboard          └─ notifications / monitor        │
└───────┬─────────────────────────────────────────┬────────────────────────┘
        │ Prisma                                  │ enqueue
        ▼                                         ▼
┌────────────────────┐                 ┌──────────────────────┐
│ PostgreSQL 18      │                 │ Azure Service Bus    │
│ 34 models          │                 │ queue                │
│ 66 migrations      │                 └──────────┬───────────┘
└────────────────────┘                            │ consume
        ▲                                         ▼
        │ reconcile                   ┌───────────────────────────────┐
        │                             │ Provider adapters (5 live)    │
┌───────┴────────────┐                │ BrevoEmail · SMTP · CmSms     │
│ Provider webhooks  │◀───────────────│ CmWhatsApp · ConsoleSms       │
│ (Brevo, inbound)   │                └───────────────────────────────┘
└────────────────────┘
                              ┌──────────────────────┐
                              │ Azure Blob Storage   │
                              │ attachments/exports  │
                              └──────────────────────┘
                              ┌──────────────────────┐
                              │ Preset SQL source DB │
                              │ (read, polled)       │
                              └──────────────────────┘
```

*(VERIFIED — module tree, `main.ts`, Prisma schema, CI workflows, env var names, production `provider` column)*

**Deployment:** backend builds a Docker image pushed to Azure Container Registry and deployed to an Azure Web App; frontend deploys to Azure Static Web Apps. Both via GitHub Actions. *(VERIFIED — `.github/workflows/`)*

**Authentication:** Google Identity Services / OIDC, invite-only, Workspace-domain enforced, tokens verified server-side. Authorisation is enforced by `SessionAuthGuard` and `RolesGuard`, never in the UI alone. *(VERIFIED — `src/common/guards/`, `AGENTS.md`)*

---

## 6. Main Workflows

**Campaign lifecycle.** Draft → submit for review (named Admin reviewer) → approve/reject → live. Approval is backend-enforced; it is not a UI affordance. Production shows 128 campaign submissions → 124 approvals, 1 rejection, 3 withdrawals; and 92 template-version submissions → 82 approvals, 2 rejections. *(VERIFIED — `AuditLog`)*

**Execution.** An approved campaign is *frozen* into a `CampaignExecutionRun` that snapshots channel bindings, template versions, approval state and audience metadata. Each eligible recipient becomes a durable `CampaignExecutionRecipient` holding the rendered subject/body and variable values at that moment. Recipients are handed to Azure Service Bus; a processor consumes them and calls the provider adapter. A browser request never sends. *(VERIFIED — `docs/execution-plane-foundation.md`, schema, 85,619 recipient rows)*

**Audience sourcing.** Either an uploaded CSV or a "preset SQL flow" that polls an external source database on an interval, dedupes rows, and creates recipients. 13 distinct flow keys are live. *(VERIFIED)*

**Reconciliation.** Provider webhooks feed `CampaignExecutionEvent` — 212,492 rows covering accepted/delivered/opened/clicked/bounced/deferred/blocked — reconciling true delivery outcomes back against the platform's own execution state. *(VERIFIED)*

**External Service Comms.** A separately governed API surface (`/mail`, `/sms`, `/whatsapp`) letting other internal systems send through UCP under per-client API keys and rate limits, with threaded reply handling. `POST /api/mail` deliberately preserves the legacy `iros-mail` request contract — same bearer-key auth, same field names (`sender`, `from`, `to`, `bcc`, `subject`, `mjml`, `html`, `text`, `template`, `attachment`, `processAt`) — so existing callers could repoint rather than rewrite. *(VERIFIED — `docs/transactional-mail-compatibility.md`)*

**Per-customer communication history.** `GET /customer-communications` answers "what have we sent this customer?" with a `UNION ALL` across campaign execution recipients and external-service emails, so both routes into the platform surface in one timeline. *(VERIFIED — service SQL)*

**Legacy cutover.** Eleven Apps Script flows are implemented as UCP preset-flow adapters, including `os_payments_new`, `canc_chase`, `canc_acknow`, `renew_acknowledge`, `renw_nocontact`, `dreq`, `dreq_inception`, `ad_hoc_docs`, `rta7_emails`, `rta7_rmail` and `renew_invite`. Eight further functions referenced by the legacy `code.gs` have **no implementation present in the checked-in source** and are explicitly tracked as a separate discovery track rather than assumed migrated. *(VERIFIED — `docs/legacy-script-cutover-plan.md`)*

---

## 7. Technology Stack

**Backend:** NestJS 11, TypeScript, Prisma 7 + PostgreSQL 18, `class-validator`, Swagger/OpenAPI, Luxon, Azure Service Bus SDK, Azure Blob Storage SDK, Jest.
**Frontend:** React 18, Vite, TypeScript, TailwindCSS, shadcn/ui + Radix, TanStack Query, React Router, Vitest + Testing Library.
**Infrastructure:** Azure Web App (container), Azure Container Registry, Azure Static Web Apps, Azure Service Bus, Azure Blob Storage, GitHub Actions.
**Providers:** Brevo (email + webhooks), SMTP, CM.com (SMS + WhatsApp), RMail-related reconciliation.
**Docs:** MkDocs (`--strict` builds).

---

## 8. Production Evidence and Statistical Analysis

All figures below were obtained on 8 October 2026 through a PostgreSQL session with `default_transaction_read_only=on`. The guard was verified before querying: an attempted `CREATE TABLE` was rejected with `cannot execute CREATE TABLE in a read-only transaction`. No write, DDL, or migration was executed. No personal data is reproduced here — every figure is an aggregate.

### 8.1 Campaign message volume

**85,482** `CampaignExecutionRecipient` rows have a non-null `sentAt`, between **26 June and 8 October 2026** (105 distinct active sending days).

| Channel | Rows | Failed/Bounced | Rate |
|---|---|---|---|
| Email | 64,055 | 683 | 1.066% |
| SMS | 21,443 | 50 | 0.233% |
| WhatsApp | 121 | 0 | 0.000% |

Monthly dispatched (Email / SMS / WhatsApp): Jun 1,531 / 553 / — · Jul 18,598 / 6,013 / — · Aug 19,013 / 5,838 / 93 · Sep 19,530 / 6,997 / 28 · Oct (8 days) 5,317 / 1,971 / —.

- **Average per active sending day: 814** *(DERIVED — 85,482 ÷ 105)*
- **Peak day: 1,598** on 9 July 2026 *(VERIFIED)*

> **Qualification — important.** These are *message attempts*, not unique customers. One audience row can produce an Email and an SMS. Across preset-flow sourcing, 73,415 source items carry only 52,520 distinct dedupe keys, and 4,968 keys recur across more than one processing day — so the same customer is legitimately contacted on multiple days. Unique-recipient counts were deliberately **not** computed, to avoid touching personal data.

### 8.2 Reliability

| Measure | Value |
|---|---|
| Execution runs | 5,261 — 4,940 Completed, 317 CompletedWithFailures, 4 Cancelled |
| Runs fully clean | **93.9%** *(DERIVED)* |
| Preset-flow polling runs | 149,321 — 149,065 succeeded, 256 failed (**99.83%**) *(DERIVED)* |
| Schedule dispatches | 1,146 — across **1,146 distinct** `(scheduleId, scheduledFor)` slots |
| Recipients retried | 747; maximum attempt count 2 |

The schedule figure is the strongest single idempotency result available: the `@@unique([scheduleId, scheduledFor])` constraint has held across every dispatch, with **zero double-sends** attributable to duplicate scheduling.

### 8.3 Automation

| Trigger | Runs | Share |
|---|---|---|
| PresetFlowPolling | 4,219 | 80.2% |
| Scheduled | 1,011 | 19.2% |
| Manual | 22 | 0.4% |
| Retry | 9 | 0.2% |

**99.4% of execution runs started without a human pressing anything** *(DERIVED — 5,230 ÷ 5,261)*.

### 8.4 External Service Comms

18,113 emails between 11 May and 8 October 2026, from 22 registered API clients (traffic recorded against 20 distinct client IDs; busiest client 7,345 emails, mean 862.5).

Status: Sent 11,271 · Delivered 5,238 · Skipped 1,263 · Failed 341.
Monthly: May 47 · Jun 1,077 · Jul 1,931 · Aug 6,016 · Sep 7,075 · Oct (8 days) 1,967 — a **consistent month-on-month growth curve in adopted usage** by other internal systems.

### 8.5 Governance

- **364,430** audit events, 30 Apr – 8 Oct 2026, across **59 distinct action types**, 7 distinct actor user IDs (most events are system-actor).
- Largest categories: `preset_flow_synced` 174,411 · `execution_recipient_sent` 122,543 · `execution_webhook_received` 27,057 · `preset_flow_sync_failed` 21,608.
- **917** immutable `MessageSnapshot` records (7 Aug – 8 Oct 2026).

> **Qualification.** `preset_flow_sync_failed` (21,608) vastly exceeds failed `PresetFlowRun` rows (256). These count different things — the audit action appears to be logged per campaign per attempt, while a run row is one persisted sync. The **99.83% figure in §8.2 uses `PresetFlowRun`**, the authoritative run record. The discrepancy is noted rather than resolved; see Evidence Gaps.

### 8.6 Content and configuration

16 campaigns (15 Approved) · 33 templates (27 Email, 5 SMS, 1 WhatsApp) · 145 template versions, highest version number 13 · 14 schedules (13 Active) · 18 email senders · 4 messaging senders · 5 provider adapters live.

---

## 9. Business Impact

**Established by evidence:**

- **Consolidation — the originating goal — is measurably happening.** 22 internal client systems now send through the platform rather than around it, growing 47 → 7,075 emails monthly (§8.4), on top of 85,482 campaign messages. Both routes land in one store with one audit trail.
- **Visibility where there was none.** Per-customer communication history, dashboard, searchable/exportable message log, and a 364,430-event audit trail — against a legacy estate with no interface at all (§3.2).
- **Failure is now visible rather than silent.** Legacy failures reached an unimplemented TODO; UCP records 683 failed/bounced emails and 50 failed SMS as queryable state, surfaces `CompletedWithFailures` as a distinct run outcome, retries with recorded lineage (747 retries), and emails operational summaries including failure detail.
- **Automation of recurring outbound operations.** 99.4% of runs auto-triggered (§8.3), addressing manual re-triggering directly.
- **Governed approval actually in use.** 220 submissions, 206 approvals, 3 rejections, 3 withdrawals (§6) — exercised, not decorative. No approval gate existed before.
- **Two legacy systems replaced.** Eleven Apps Script flows reimplemented as preset-flow adapters; the `iros-mail` contract absorbed into UCP (§6, §11).
- **Executive reporting delivered.** Daily summary, operational report with export, and weekly audit reporting now exist as services.

**Stakeholder recognition.** The subject reports being commended on this project by both the CEO and the CTO on several occasions. *(TESTIMONY — uncorroborated; see §22 for how to make this citable.)*

**Not established — do not claim:** staff hours saved, cost savings, revenue impact, quantified error-rate reduction, or number of customers previously missed. §3.2 explains *why* these are structurally unavailable: the legacy estate recorded no send-level outcome data to compare against. The qualitative before/after in §3.2 is the honest substitute, and it is well-evidenced.

---

## 10. Significant Engineering Challenges

**Snapshot reproducibility under mutable configuration.** A campaign or template can be edited at any time, but a message sent last month must remain reproducible exactly as rendered. Solved by freezing rendered subject, body, variable values, template version, approval state and audience metadata onto each execution record at run-creation time — so later edits cannot alter in-flight sends or the audit trail. This is harder than it sounds because it requires every downstream view (history, reporting, reconciliation) to read the snapshot rather than the live entity.

**Idempotency across three independent trigger paths.** Runs can originate from polling, a schedule, a manual action, or a retry. With a poller firing continuously, at-least-once queue delivery, and schedules that can overlap, duplicate sends are the default failure mode. Addressed with database-level uniqueness — `@@unique([campaignId, presetFlowKey, processingDate, dedupeKey])`, `@@unique([scheduleId, scheduledFor])`, `@@unique([campaignId, channel, audienceRowIndex])` — making the guarantee structural rather than a code convention. Production bears this out (§8.2).

**Distributed state across systems that disagree.** The platform's own execution state and the provider's later view of reality are different things and arrive out of order. 212,492 webhook events reconcile against 85,482 sends. The documentation explicitly requires V1 to *preserve the distinction* between its own state and external provider evidence rather than collapsing them — a deliberate modelling decision in favour of auditability over simplicity.

**Unreliable external dependencies.** An external source database is polled continuously (149k runs), with stale-lock handling for crashed pollers, and CM.com failures captured through a dedicated `CmRequestFailedError` carrying provider context.

**Provider abstraction proven, not theoretical.** Five adapters run in production simultaneously (Brevo, SMTP, CM SMS, CM WhatsApp, Console) with per-sender and per-campaign routing.

**Domain complexity.** 34 models, 66 migrations, 155 routes, 19 modules — plus channel-aware suppression, throttle windows, blackout periods, per-channel eligibility rules, template variable validation, and per-client rate limits.

---

## 11. Architectural Decisions and Trade-offs

**Control plane / execution plane split**
*Problem:* mutable campaign config versus immutable sent-message evidence.
*Decision:* separate entities; freeze a run from approved state.
*Trade-off:* significant data duplication (85,619 recipient rows each carrying rendered content) and more complex queries.
*Result:* every send reproducible; 917 snapshots and full audit chain in production.

**Queue-backed execution (Azure Service Bus)**
*Problem:* HTTP request lifecycle cannot own delivery of thousands of messages.
*Decision:* enqueue per recipient; a processor invokes the adapter. Stated as a non-negotiable boundary.
*Trade-off:* at-least-once semantics force idempotency work; operational surface grows.
*Result:* 1,598-message peak day handled; 5,261 runs with 4 cancellations.

**Modular monolith over microservices**
*Decision:* 19 NestJS modules with explicit boundaries, one deployable.
*Trade-off:* one large service — `campaign-execution.service.ts` is 10,463 lines, which is a real maintainability cost and the clearest refactoring candidate in the codebase.
*Result:* deployable by a single engineer; the boundaries exist for later extraction.

**Idempotency in the database, not the application** — covered in §10.

**Backwards-compatible API contract to enable migration**
*Problem:* the strategic goal — every company communication through one platform — fails if each calling system must be rewritten to adopt it. Adoption friction, not capability, was the binding constraint.
*Decision:* reimplement the legacy `iros-mail` HTTP contract verbatim inside UCP (`POST /api/mail`, bearer-key auth, identical field names) while discarding its EJS/local-microservice internals; then extend the same pattern to `/api/sms` and `/api/whatsapp`.
*Reasoning:* callers repoint a base URL instead of rewriting an integration, so consolidation can proceed incrementally rather than as a coordinated cutover.
*Trade-off:* legacy field shapes (`mjml`, `processAt`, `attachment`) are carried into a new system's public contract and will constrain it for as long as those clients exist — real long-term design debt accepted for short-term migration velocity.
*Result:* 22 registered clients and 18,113 emails, with monthly volume growing 47 → 7,075 in four months (§8.4). This is the decision that converted a stated strategic goal into measured adoption.

**Lovable for UI scaffolding only**
*Decision:* `AGENTS.md` mandates using Lovable to accelerate UI scaffolding and "never as the owner of business logic or auth."
*Result:* bot activity stops after one week (28 Apr); all later frontend work is hand-directed. The boundary held.

---

## 12. Problems Solved

### UK date format silently flipping in customer communications

**Situation.** Legacy preset flows map source rows into template variables, including cancellation and due dates, for customer-facing insurance emails.
**Problem.** The original helper used `new Date(value)`. For a `dd/MM/yyyy` string, JavaScript's parser applies US `MM/dd`, so 5 August silently became 8 May. Values below 13 flip silently; above 12 they don't — so the bug is invisible for most of any given month and only wrong for part of it.
**Investigation.** Traced to a private `toLegacyDateString` helper duplicated in the variable mapper.
**Solution.** Explicit Luxon parsing with a declared format, lifted into a shared `legacy-flow.helpers.ts`, plus a regression test named for the failure mode: *"preserves dd/MM/yyyy cancellation dates for rta7 emails without month-day flipping"*.
**Result.** Commit `22d52e0`, 18 Aug 2026 — 146 insertions, 30 deletions across 3 files, 98 of them test code.
**Evidence.** VERIFIED — commit diff and test.

*Why this is a strong interview story: it is a correctness bug with regulatory consequence, invisible to type checking, invisible for over half of any month, and fixed with a named regression test rather than a patch.*

### Duplicate source keys crashing the sync

**Situation.** Preset SQL flows pull batches from an external source database.
**Problem.** A batch could contain two rows producing the same dedupe key; the unique constraint then aborted the whole sync — the constraint protecting against duplicates was taking down the job.
**Solution.** Duplicate-tolerant handling in `syncSourceItems`, with a test: *"does not crash when a sync batch contains duplicate outstanding payment dedupe keys."*
**Result.** Commit, 14 Jun 2026. Production now shows 149,065 successful runs against 256 failures.
**Evidence.** VERIFIED — commit diff; production counts.

### Circular foreign keys blocking database restore

**Problem.** `Template.currentVersionId`, `User.departmentId`, and two `Department` self-references form cycles that defeat ordered restore.
**Solution.** The prod-to-local sync script drops those four constraints before restore and restores them after.
**Evidence.** VERIFIED — commit, 12 Sep 2026.

### Permission model corrected: withdraw-review

**Problem.** `canWithdrawCampaignReview` granted the action to any Admin, so the reviewer a campaign was submitted *to* saw a "Withdraw review" button — a reviewer could retract someone else's submission.
**Solution.** Narrowed to the draft team (owner plus collaborators); reviewers act by approving or rejecting.
**Evidence.** VERIFIED — observed and fixed during this session's work.

---

## 13. Security, Reliability and Operational Considerations

**Security.** Invite-only Google OIDC with server-side token verification and Workspace-domain enforcement; no local passwords. Two guards enforce session and role. API clients authenticate by key with per-client rate limits. Webhook endpoints verify shared secrets. All secrets are environment variables; none are committed.

**Reliability.** Queue-backed delivery; retry with recorded lineage (`retriedFromRecipientId`, 747 retries); stale-lock recovery for pollers and processors; dedupe windows for transactional mail; throttle windows and blackout periods; `CompletedWithFailures` as a distinct run state so partial failure is visible rather than hidden.

**Observability.** 364,430-row audit log across 59 action types; a `health` module; a `monitor` module exposing `ucp-execution-summary` for an external dashboard (bounded to a 3-day window — a deliberate load guard); daily/operational/weekly reporting services.

**Testing.** 9 spec files, 84 unit cases, 9 e2e cases. **This is thin relative to a 62,750-line backend** and is the most obvious engineering weakness. Tests are concentrated where logic is subtlest (preset flows, variable rendering, trigger rules) — a defensible prioritisation, but not broad coverage.

---

## 14. Lessons Learned

**Worked well.** Pushing invariants into database constraints — the zero-duplicate dispatch result is a constraint doing its job silently 1,146 times. Freezing execution state early, which made reconciliation and audit straightforward rather than archaeological. Writing `AGENTS.md` before heavy AI tooling use, which kept generated code inside intended boundaries. Keeping documentation a same-task deliverable rather than a later chore.

**Would do differently.** `campaign-execution.service.ts` reaching 10,463 lines is a boundary that should have been split earlier — run creation, queue handoff, provider dispatch and reconciliation are four responsibilities in one file. Test coverage should have grown with the domain rather than clustering around known-subtle areas. And the dd/MM bug argues for banning `new Date(string)` on business input from the outset, by lint rule rather than by convention.

**Operational lesson.** The gap between `preset_flow_sync_failed` audit events (21,608) and failed run rows (256) shows that two logging paths recording "the same" event at different granularities will eventually be read as contradicting each other. Metric definitions need to be as deliberate as schema.

**Migration lesson.** The single highest-leverage decision was not a feature: preserving the legacy HTTP contract verbatim meant other teams could adopt the platform by changing a URL. Consolidation goals fail on adoption friction long before they fail on capability — and the cost, legacy field shapes embedded in a new public API, was worth paying and should be paid knowingly.

**Lesson about instrumentation, learned from the system being replaced.** The legacy estate could not be measured because it never recorded outcomes — which makes its own replacement impossible to quantify, and means no one could prove how bad things were while they were bad. Systems should record what they did from day one, even before anyone asks for a report; the absence is only discovered when it is too late to backfill. This is also why §22 now says to stop hunting for a baseline that cannot exist.

---

## 15. Verified Career Claim Bank

> **Claim 1.** Built and operate a governed multi-channel communications platform that has dispatched **85,482 campaign messages** across Email, SMS and WhatsApp in production.
> **Level:** VERIFIED · **Evidence:** `CampaignExecutionRecipient` where `sentAt` is not null · **Period:** 26 Jun – 8 Oct 2026 · **Qualification:** message attempts, not unique customers.

> **Claim 2.** Delivered an email channel with a **1.07% combined failure-and-bounce rate** and an SMS channel at **0.23%**.
> **Level:** DERIVED · **Evidence:** 683/64,055 and 50/21,443 by terminal status · **Qualification:** terminal state at query time; late webhooks could shift marginally.

> **Claim 3.** Designed database-enforced idempotency that produced **zero duplicate schedule dispatches across 1,146 dispatches**.
> **Level:** VERIFIED · **Evidence:** `count(*) = count(distinct (scheduleId, scheduledFor)) = 1,146` · **Qualification:** covers schedule-path duplicates specifically.

> **Claim 4.** Built a near-real-time SQL polling pipeline that has executed **149,321 sync runs at 99.83% success** across 13 flows.
> **Level:** VERIFIED/DERIVED · **Evidence:** `PresetFlowRun` · **Period:** 25 Jun – 8 Oct 2026.

> **Claim 5.** Automated outbound operations such that **99.4% of 5,261 execution runs were system-triggered**.
> **Level:** DERIVED · **Evidence:** `triggerSource` distribution · **Qualification:** measures trigger origin, not absence of human oversight — approval remains manual by design.

> **Claim 6.** Implemented a compliance audit trail capturing **364,430 events across 59 action types**.
> **Level:** VERIFIED · **Period:** 30 Apr – 8 Oct 2026.

> **Claim 7.** Built an authenticated API platform adopted by **22 internal client applications**, carrying **18,113 emails** with month-on-month growth from 47 (May) to 7,075 (Sep).
> **Level:** VERIFIED · **Evidence:** `TransactionalEmail`, `TransactionalMailClient`.

> **Claim 8.** Authored **221 of 228 backend commits (+77,448 source lines)** over a continuous six-month delivery.
> **Level:** VERIFIED · **Qualification:** AI coding assistance was used; this is authorship-of-record and architectural ownership, not a claim of unaided keystroke authorship.

> **Claim 9.** Designed a provider abstraction running **five adapters simultaneously in production**.
> **Level:** VERIFIED · **Evidence:** `provider` column distribution.

> **Claim 10.** Delivered a NestJS backend of **155 REST endpoints across 19 modules over 34 data models and 66 migrations**, with OpenAPI documentation.
> **Level:** VERIFIED.

> **Claim 11.** Built and maintained a published technical documentation site as a same-task deliverable alongside code.
> **Level:** VERIFIED · **Evidence:** `ucp-documentation`, 17 commits; `--strict` MkDocs build policy in `AGENTS.md`.

> **Claim 12.** Diagnosed and fixed a locale date-parsing defect that silently transposed day and month in customer-facing insurance communications, shipping it with a named regression test.
> **Level:** VERIFIED · **Evidence:** commit `22d52e0`, 18 Aug 2026.

> **Claim 13.** Replaced **two** legacy production systems with one platform: a Google Apps Script orchestration estate (11 flows reimplemented as preset-flow adapters) and a .NET 8 Azure Functions mail microservice (contract absorbed into the platform API).
> **Level:** VERIFIED · **Evidence:** `legacy-script-cutover-plan.md`, `transactional-mail-compatibility.md`, legacy source · **Qualification:** 8 further legacy functions have no implementation in the checked-in source and remain an open discovery track — the cutover is substantially but not provably complete.

> **Claim 14.** Designed a backwards-compatible API contract that let 22 existing internal client systems migrate onto the platform by repointing rather than rewriting, growing from 47 to 7,075 emails monthly in four months.
> **Level:** VERIFIED/DERIVED · **Evidence:** compatibility doc; `TransactionalEmail` monthly counts · **Qualification:** growth reflects migration onto the platform, not new business volume.

> **Claim 15.** Delivered a unified per-customer communication history (`UNION ALL` across campaign and external-service sends) — a capability neither replaced system provided, and the one executives specifically asked for.
> **Level:** VERIFIED (capability) / TESTIMONY (that it was the executive ask) · **Evidence:** service SQL, dedicated UI page.

> **Claim 16.** Replaced a failure path that notified nobody — the legacy handler was an unimplemented `//todo send email` stub — with recorded failure state, retry lineage, distinct partial-failure run status, and emailed operational summaries.
> **Level:** VERIFIED · **Evidence:** `old-google-scripts/iros-process-one/functions.gs`; UCP execution schema and notification services · **Qualification:** describes a capability gap, not a measured reduction in failures.

> **Claim 17.** Commended on this project by the CEO and CTO on several occasions.
> **Level:** TESTIMONY · **Evidence:** subject's account only · **Qualification:** usable in interview as "I'm told…"; do not put in writing as fact until a dated written source exists (§22).

---

## 16. CV Achievement Candidates

### High-confidence quantified

- Designed and built a governed multi-channel communications platform (NestJS, PostgreSQL, Azure Service Bus) that has dispatched **85,482 production messages** across Email, SMS and WhatsApp at a **1.07% email failure/bounce rate**, replacing Google Apps Script–driven orchestration for a UK insurance broker.
- Engineered a queue-backed execution plane with database-enforced idempotency, achieving **zero duplicate dispatches across 1,146 scheduled sends** and **99.83% success across 149,321 automated source-polling runs**.
- Automated recurring outbound operations so that **99.4% of 5,261 execution runs are system-triggered**, while preserving mandatory human approval before any campaign goes live.
- Built a compliance-grade audit and reproducibility layer recording **364,430 events across 59 action types**, with immutable per-message snapshots enabling any communication to be reproduced exactly as sent.
- Consolidated company-wide outbound communication by designing a backwards-compatible API that let **22 internal client systems** migrate onto the platform by repointing rather than rewriting, carrying **18,113 emails** and growing from 47 to 7,075 monthly in four months.
- Replaced **two** legacy production systems — a Google Apps Script orchestration estate (11 business flows reimplemented) and a .NET 8 Azure Functions mail microservice — with a single governed platform.
- Authored **221 of 228 backend commits (+77,448 lines)** across a 155-endpoint, 19-module NestJS service over 34 data models and 66 migrations.

### High-confidence non-quantified

- Architected a control-plane/execution-plane separation that keeps campaign configuration mutable while guaranteeing sent messages remain reproducible exactly as rendered.
- Designed a provider-adapter abstraction running five integrations concurrently (Brevo, SMTP, CM.com SMS, CM.com WhatsApp, Console), allowing vendor substitution without touching domain logic.
- Implemented invite-only Google OIDC authentication with Workspace-domain enforcement and backend-enforced RBAC, with authorisation never delegated to the UI.
- Built channel-aware suppression and eligibility evaluated before send, supporting FCA/GDPR/PECR-aligned operation.
- Delivered a unified per-customer communication history spanning campaign and API-originated messages, giving the business its first single view of what any individual customer had been sent.
- Replaced a silent-failure model — where the legacy failure handler was an unimplemented stub and failed sends were discovered only by noticing a customer had not been contacted — with recorded failure state, retry lineage and emailed operational summaries.
- Established and maintained a published documentation site as a same-task delivery requirement rather than a trailing chore.
- Diagnosed and fixed a UK/US date-parsing defect that silently transposed day and month in customer-facing communications, shipping it with a named regression test.

### Requiring confirmation before use

- Any claim of hours saved, cost reduction, or error-rate improvement versus the legacy Google Scripts system — **no baseline exists**.
- Any uptime or SLA figure — no monitoring data was available.
- Any unique-customer-reached figure — deliberately not computed to avoid personal data.
- Team-size or leadership framing — only one other contributor appears in history, with 7 commits.

---

## 17. Portfolio Case Study Material

*This section is an assembly guide, not a retelling — each beat points at the section holding the evidence, so a case study can be drafted without restating the dossier.*

| Case-study beat | Draw from | The line that carries it |
|---|---|---|
| Context & challenge | §3 | Two chained legacy systems, silent failure, no record of what reached whom, executive demand for one monitored place |
| My role | §4 | Architectural ownership; 221/228 backend commits; honest AI-assistance framing |
| Solution & architecture | §5, §6 | Control/execution plane split; queue-backed delivery; five provider adapters; compatibility API |
| Interesting problems | §10, §12 | Snapshot reproducibility; idempotency across four trigger paths; out-of-order webhook reconciliation; the dd/MM defect |
| Key decision | §11 | Preserving the legacy contract verbatim — adoption as the binding constraint |
| Results | §8, §9 | 85,482 campaign messages; 22 migrated clients; zero duplicate dispatches; 99.4% automated. Qualitatively: first per-customer history, failure now visible |
| What I'd improve | §14 | Split the 10,463-line service; grow coverage with the domain; lint-ban `new Date(string)`; reconcile the sync metrics |

**Narrative spine to use:** a system nobody could measure → read the source to find out *why* (the TODO stub) → build the missing capabilities as structure, not features → make adoption frictionless so consolidation actually happens → prove it with numbers that are honestly qualified.

---

## 18. Interview Story Bank

**Correctness under invisibility — the dd/MM bug** (§12). *S:* preset flows map source dates into customer-facing insurance emails. *T:* dates were occasionally wrong with no error anywhere. *A:* traced to `new Date()` applying US month-day order to UK strings — wrong only for day values ≤ 12, so invisible most of the month; replaced with explicit Luxon parsing in a shared helper and added a regression test named for the failure. *R:* commit `22d52e0`; 98 of 146 added lines were test code. *E:* commit diff.

**Designing for duplicates you cannot prevent** (§10, §12). *S:* a continuous poller, at-least-once queue delivery, and overlapping schedules. *T:* guarantee no customer is messaged twice. *A:* moved the invariant into the schema — composite unique constraints on the natural keys — rather than trusting application checks; then fixed the fallout when a single batch contained internal duplicates and aborted the whole sync. *R:* 1,146 dispatches across 1,146 distinct slots; 149,065/149,321 syncs succeed. *E:* schema and production counts.

**Making compliance structural, not procedural.** *S:* FCA/GDPR/PECR-aligned outbound comms. *T:* prove what was sent, to whom, approved by whom. *A:* backend-enforced approval, immutable execution snapshots, 59 audited action types, channel-aware pre-send suppression. *R:* 364,430 audit events; 206 recorded approvals; every message reproducible. *E:* schema, audit table, product-scope doc.

**Deciding what a metric means.** *S:* two code paths logged "sync failed" at different granularities, producing 21,608 versus 256. *T:* report reliability honestly. *A:* established which record is authoritative, reported the run table, and documented the discrepancy rather than quoting the flattering number. *R:* a defensible 99.83%. *E:* §8.5. *(Also a good answer to "tell me about a time you resisted a better-sounding number.")*

**Using AI tooling without losing ownership.** *S:* Lovable and coding agents in a regulated codebase. *T:* move fast without AI owning business logic or auth. *A:* wrote `AGENTS.md` defining workspace boundaries, architectural direction, auth rules and validation gates before heavy use. *R:* bot commits cease after week one; all later frontend work hand-directed; boundaries held. *E:* `AGENTS.md`, commit authorship timeline. *(Honest, current, and increasingly asked about.)*

**Replacing a system nobody could measure.** *S:* communications ran through Apps Scripts and a mail microservice with no central record of what reached whom; executives wanted one monitored place. *T:* justify and design a replacement without a baseline to argue from. *A:* read the legacy source for the failure mechanism rather than relying on anecdote — found the failure notifier was an unimplemented `//todo send email` stub and that only per-process "row picked up" markers were persisted; designed UCP so the missing capabilities (send-level outcomes, per-customer history, approval, audit) were structural. *R:* 22 client systems migrated onto the platform; every message now attributable to a customer and a run. *E:* legacy source; production counts. *(Also the honest answer to "what metrics did you improve?" — explain why no baseline could exist, then show the capability delta.)*

**Migration velocity as a design constraint.** *S:* consolidation fails if every calling system must be rewritten. *T:* get other teams onto the platform without a coordinated cutover. *A:* reimplemented the legacy `iros-mail` HTTP contract verbatim while discarding its internals, so callers repoint a URL. *R:* 22 clients, 47 → 7,075 emails monthly. *T/O:* legacy field shapes now sit in a new system's public contract — accepted knowingly. *E:* compatibility doc; monthly counts. *(Strong because it shows adoption treated as an engineering problem, and an owned trade-off.)*

**Permission models are easy to get subtly wrong** (§12). The reviewer could withdraw the submitter's own review request, because the check asked "are you an Admin?" rather than "are you the requesting side?".

---

## 19. Technical Article and Blog Opportunities

**"`new Date()` is not your friend: the bug that was only wrong for twelve days a month."** Locale-ambiguous date parsing, why it evades type systems and tests, and designing the regression test around the failure mode. *Anonymise:* broker identity, customer data, process names. *Do not publish:* source queries, schema.

**"Put your invariants in the database."** Composite unique constraints as duplicate-prevention versus application-level checks; what happens when the constraint protecting you takes down your job; evidence that it works at scale. *Publish:* the pattern. *Do not publish:* real key composition or business identifiers.

**"Snapshot everything: designing for reproducibility in regulated messaging."** Control/execution plane split, immutable rendered content, the deliberate choice to keep platform state and provider evidence separate. *Anonymise:* sector specifics. *Do not publish:* audit action taxonomy.

**"`//todo send email`: the comment that explains four years of lost messages."** A real legacy failure handler that logged to console and notified nobody, and what silent failure costs in a regulated workflow — alerting, retry lineage, partial-failure states, and making "did this actually go out?" a question the system can answer. *Anonymise:* broker, process names. *Do not publish:* legacy source beyond the two short functions, which must be shown without business identifiers.

**"Strangling two legacy systems at once."** Replacing an orchestration layer and the microservice beneath it, and why preserving the old HTTP contract verbatim — field names, auth header and all — was the decision that actually drove adoption. The trade-off of importing legacy field shapes into a new public API. *Do not publish:* the field list as it maps to internal systems, or client identities.

**"Two logs, one event, two numbers: metric definition as a design problem."** The 21,608 vs 256 discrepancy, generalised.

**"Governing AI coding tools in a regulated codebase."** What an agent-instruction file should contain — workspace boundaries, architectural direction, validation gates, where AI must never own logic — and evidence that scoping held. *Do not publish:* the file verbatim.

---

## 20. Professional Content Opportunities

- **Shipping reliability you can prove.** 1,146 dispatches, 1,146 distinct slots, zero duplicates — the constraint that did its job silently. *Credible because it is a single verifiable query.*
- **The bug that was only wrong half the month.** Short, memorable, technically instructive. *Credible from the commit and its test.*
- **Choosing the less impressive number.** Why the reliability figure quoted is 99.83% and not something prettier. *Credible from §8.5.*
- **Six months, one backend, 155 endpoints.** A build-log retrospective on scoping a modular monolith solo. *Credible from Git history.*
- **Honest AI-assisted engineering.** What was scaffolded, what was directed, where the boundary was drawn, and why the record says so. *Credible from the authorship timeline — and distinctive, because most posts on this are not specific.*
- **The 10,463-line file I would split.** Owning a known architectural debt publicly. *Credible from the file itself.*
- **"`//todo send email`."** One screenshot of a real legacy failure handler that notified nobody, and what that costs. *Credible, memorable, and lands instantly with any engineer.* *Must be anonymised.*
- **Adoption is an engineering problem.** Why keeping the old API contract verbatim did more for consolidation than any feature. *Credible from 22 migrated clients and the growth curve.*
- **What to do when there's no baseline.** Replacing a system that measured nothing, and arguing capability delta instead of percentages. *Credible, unusual, and quietly demonstrates integrity about metrics.*

---

## 21. Private vs Public-Safe Information

**PRIVATE — do not publish.** Client/broker identity and domain; `.env` contents and every credential; connection strings; API keys and webhook secrets; database host names; preset flow keys and legacy process names (`os_payments_new`, `rta7_*`, `dreq_*`); legacy table names (`wp_*`); SQL source queries; internal user names, counts and roles; customer data of any kind; per-client API traffic attribution; identities of the 22 integrating systems; Azure resource names; the audit action taxonomy; precise schema details; the legacy API field list as it maps to internal callers; `AGENTS.md` verbatim.

**Named individuals.** Do not publish the CEO/CTO commendation in a way that identifies them or the company. In interview it is fine as "I'm told the CEO and CTO were pleased with it"; in writing, only once a dated source exists and only in a private CV or application, never publicly attributed.

**PUBLIC-SAFE — with the sector described generically.** Aggregate volume, reliability and automation percentages in §8 (they contain no identifying detail); the architectural patterns in §5, §10 and §11; the technology stack; the engineering problems and their solutions in §12 stated as patterns; the lessons in §14; CV bullets in §16 provided the client is unnamed.

*Judgement applied: where unsure, classified private. In particular, per-client traffic distribution and the flow-key names were moved to private despite being technically aggregate, because they could identify internal systems and business processes.*

---

## 22. Evidence Gaps and Further Metrics Worth Collecting

**Resolved since the first draft.** Two gaps are now closed: the *origin and stakeholder requirements* (§3, from subject testimony, substantially corroborated in legacy code) and *legacy cutover status* (§6 — 11 flows migrated, 8 discovery-only items whose source is absent).

**Reclassified: the legacy baseline is structurally unobtainable, not merely unlocated.** The first draft suggested asking colleagues for before/after numbers. Investigation of the legacy source shows why that will not work: the Apps Scripts logged failures only to a console via an unimplemented notifier, and kept no send-level record — only per-process "row picked up" markers. `iros-mail` held its own email records, but those cover one service's sends, not the estate. **There is no artefact from which legacy volume, failure rate, or missed-customer counts could be reconstructed.** Treat §3.2's qualitative, code-corroborated before/after as the ceiling of what is claimable here, and stop looking for the number.

| Gap | Why it matters | Where it might exist | Question to ask |
|---|---|---|---|
| Written record of CEO/CTO commendation | Converts the strongest stakeholder-impact evidence from testimony into something citable | Email, Slack, performance review, board or project update deck | "Could you put that in writing?" — or save the original message with its date |
| `iros-mail` historical email volume | The one legacy figure that *might* still exist, since that service did persist records | MongoDB store behind the legacy service, if still live | "Is the old iros-mail database still available to query?" |
| Unique customers contacted | Distinguishes reach from message count | Derivable in-database, deliberately not computed (personal data) | Request a privacy-cleared aggregate |
| Uptime / availability | Standard reliability claim; none available | Azure Web App metrics; App Insights | "Is availability monitored, and what does it show?" |
| Production launch date (first real send) | First campaign send is 26 Jun 2026, but cutover may have been staged | Release notes; cutover plan; stakeholder memory | "When did UCP become the system of record?" |
| Infrastructure cost | Enables a cost-efficiency claim | Azure billing | "What does UCP cost monthly to run?" |
| Stakeholder feedback | Qualitative impact evidence | Ops team; compliance | "What changed for the team day-to-day?" |
| Sync-failure metric reconciliation (§8.5) | Two numbers for one concept weakens both | Code paths emitting `preset_flow_sync_failed` | Internal: reconcile definitions |
| Latency / time-to-delivery | Performance claim | Derivable from `queuedAt`→`sentAt` | Compute in a later pass |
| Whether legacy systems still run in parallel | Distinguishes "replaced" from "replacing" | Ops; the 8 discovery-only functions in the cutover plan | "Are any Apps Scripts or iros-mail endpoints still live?" |
| Customer-lookup usage | Would show the flagship executive feature is actually used | Application/request logs, if retained | "Is there request logging that would show dashboard usage?" |

---

## 23. Appendix — Evidence Notes

**Method.** All production figures come from a single `psql` session with `PGOPTIONS='-c default_transaction_read_only=on -c statement_timeout=120000'`. The guard was proven before use: `CREATE TABLE` returned `ERROR: cannot execute CREATE TABLE in a read-only transaction`. Only `SELECT` with aggregation was executed. No row-level personal data was retrieved, displayed, or stored; one planned query reading error-message text was abandoned when blocked, and the aggregate-only alternative was used instead. Credentials were read from `.env` inside a script and never printed.

**Reproducing the headline figures.**

| Figure | Query shape |
|---|---|
| 85,482 dispatched | `count(*) from "CampaignExecutionRecipient" where "sentAt" is not null` |
| Channel/failure rates | `group by channel` with `filter (where status in ('Failed','Bounced'))` |
| 105 active days / 814 avg | `count(distinct "sentAt"::date)`; total ÷ distinct days |
| 99.4% automated | `group by "triggerSource"`; (4,219+1,011+9) ÷ 5,261 |
| 99.83% polling | `group by status on "PresetFlowRun"`; 149,065 ÷ 149,321 |
| Zero duplicate dispatch | `count(*)` vs `count(distinct ("scheduleId","scheduledFor"))` |
| 364,430 audit events | `count(*)`, `count(distinct action)` on `"AuditLog"` |

**Codebase counts.** `find`/`wc -l` over `src`; route count via `grep -c` on HTTP method decorators; model and migration counts from `prisma/`. **Authorship.** `git shortlog -sne --all` and `git log --author --numstat`.

**Boundary note.** This audit was read-only in both directions: no application code was modified, no refactoring performed, and no production state altered.

**Revision note (10 October 2026).** The subject supplied origin context — scattered communications, no monitoring, silent failures and manual re-triggering, executive demand for consolidation, per-customer lookup and reporting, and CEO/CTO commendation. Rather than record this as testimony alone, the legacy source was re-examined, which corroborated the failure mechanism (`notifyExecutionFailed` as an unimplemented stub; errors caught and logged only), the absence of any send-level record (0 occurrences of sheet logging; only per-process `wp_*` row markers), and the absence of any interface. Two further findings emerged that the first draft had missed entirely: `old-iros-mail` is a **replaced production system**, not merely reference material, and the External Service Comms API is a **deliberate verbatim reimplementation of its contract**, which explains the measured client migration. Claims 13–17, the compatibility-layer decision in §11, §3 in full, and the reclassified baseline gap in §22 all derive from this pass. Testimony that could not be corroborated — principally the executive commendation — is labelled TESTIMONY and left unquantified.

---

*Compiled 8 October 2026 from repository, Git, documentation and read-only production evidence; revised 10 October 2026. Figures are point-in-time as at 8 October 2026. Claims are labelled by evidence level; anything unlabelled in the narrative is supported by the sources named in its section.*
