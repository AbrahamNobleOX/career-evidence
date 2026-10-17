# iRev SSP Portal — Career Evidence Dossier

**Subject:** Abraham Akintola (GitHub: `AbrahamNobleOX`)
**Repository:** `Insurance-Revolution/nexus-renew` (local checkout: `ssp-portal-frontend`)
**Audit date:** 10 October 2026
**Evidence sources:** repository source, Git history (198 commits), `git blame` line attribution, API specification document, build/deploy configuration, and read-only queries against the live production REST API.

> **Scope note.** This repository is the **frontend** of the system. The backend REST API and the underlying legacy data store are separate services not present in this checkout. Production figures below were obtained by read-only `GET` requests to the live API; no database credentials were available or used, and no write operation of any kind was performed.

---

## 1. Executive Summary

The iRev SSP Portal is a web application that gives insurance-broking staff at BG Insurance / Insurance Revolution read access to a very large legacy **SSP broking system** dataset — clients, policies, policy events, financial transactions, and versioned underwriting "risk" snapshots — through a modern browser UI instead of the third-party platform the data previously lived on.

**Why it existed (REPORTED — see §3).** Per the CTO, the company was exiting a third-party vendor whose platform held ~25 years of its client and policy data. The vendor set a deadline of roughly one month to extract the data, with a reported penalty in excess of **£100,000** if the deadline passed. The portal was the destination for the extracted data: the place the business could hold, browse and verify it once it was off the vendor's system. This reframes the project from "a nice internal viewer" to **the delivery vehicle for a time-boxed, six-figure-exposure data exit** — and the production data profile corroborates it (a 25-year archive, 99% of it lapsed, entirely read-only, with no write path anywhere in the codebase).

**Scale verified against production (10 Oct 2026):** the API exposes **779,778 client records** and **401,339 policies** (4,249 currently Live, 397,090 Lapsed/Cancelled). Sampled policies carry rich per-policy histories — a random sample of 30 live policies returned **620 policy events**, and a random sample of 25 historical policies returned **1,521 events** spanning **2000 to 2023**. The dataset is therefore a ~25-year broking archive, not a greenfield application.

**Contribution.** Abraham built the portal end to end: he produced the initial UI using an AI scaffolding tool he directed, then did the engineering that made it a real, authenticated, production-connected application. The verified contributions are:

- **The entire API integration layer** (`src/lib/api.ts`) — 315 of 488 current lines attributed by `git blame`; every client/policy/event/transaction/risk/diary fetch, pagination model and defensive field-mapping function.
- **The API contract document** (`API_Specifications.md`) — 337 of 337 lines; the specification the backend was built against.
- **Firebase authentication and the user-administration subsystem** — `ForgetPassword.tsx` (171/171 lines), `ManageUsers.tsx` (458/613), `AuthContext.tsx` (110/159), `SignIn.tsx` (146/235), including invite-by-email, suspend/re-enable, and a suspended-user pre-check at sign-in.
- **Large-scale reduction of the prototype** — four "clean up" commits removing roughly **19,300 lines** of speculative features that had no backing data.
- **Secret hygiene remediation** — removing committed `.env` values and introducing `.env.example` plus `.gitignore` coverage.
- **Architectural influence outside his own codebase** — he did **not** build the backend (he was deliberately given no backend or database access), but he authored the contract it was built to and argued the team off a failing Firebase Data Connect approach onto the Node/PostgreSQL REST architecture that shipped and still runs. The pivot is datable in the repository to 2 Nov 2025 (§4.3).
- **Protecting the integrity of his own work under review** — reverted a colleague's AI-assisted refactor that had memoized a security check (weakening account suspension) without fixing the bug it targeted, then resolved the real defect in one line (§P7).

**Clearest business impact.** Two layers, held apart deliberately:

- *Verified:* staff can search and read a 779k-client / 401k-policy archive — including document references and 25 years of event history — from a browser behind authentication with per-user suspension control. Abraham reports the application has been **in production use for over a year**; the API was confirmed live on 10 Oct 2026, ~11 months after the last commit.
- *Reported, not independently verified:* the portal enabled a vendor exit that avoided a penalty reported at **£100,000+**. Attribute this to the CTO when using it, and never state it as a figure you measured (§3, §11).

**Strongest single career claim:** *Specified and built the complete API integration layer for a production insurance portal serving a 779,778-client / 401,339-policy legacy broking dataset, and authored the REST contract the backend was implemented against — delivered as the destination system for a time-boxed vendor data exit.* (VERIFIED, with the exit context REPORTED — see §17.)

---

## 2. Project Overview

| Attribute | Value | Evidence |
|---|---|---|
| Human-readable name | **iRev SSP Portal** | `<title>` in `index.html`; sidebar heading "SSP Portal" |
| Type | Internal B2B single-page web application (read-only CRM/archive browser) | Routes & code |
| Business driver | Destination system for a time-boxed data exit from a third-party platform | CTO, relayed verbally (REPORTED — §3.1) |
| Owner organisation | Insurance Revolution / BG Insurance | GitHub org `Insurance-Revolution`; Firebase project `bg-ssp`; og:image on `bginsurance.co.uk` |
| Repo | `Insurance-Revolution/nexus-renew` | `git remote -v` |
| Active development window | **18 Sep 2025 → 21 Nov 2025** | first and last commit dates |
| Commits | 198 total — 133 scaffolding-tool commits (directed by Abraham), 44 Abraham hand-written, 27 Abbas Ali | `git shortlog -sne --all` |
| Deployment | Firebase Hosting, region `europe-west1` | `firebase.json`, `.firebaserc`, `npm run deploy` |
| Branches | `main`, `data-sdk-integration`, `files-linking` | `git branch -a` |

**What it does.** Authenticated staff sign in, search a client list, open a client to see their details and policies, open a policy, and inspect five tabbed views of that policy: **Events** (full chronological audit trail with document links), **Transactions** (ledger entries — premiums, receipts, debits/credits), **Risk Details** (versioned underwriting snapshots: vehicle, cover, drivers, claims, convictions), **Diary** entries, and policy header/summary cards. A separate **Manage Users** screen lets an administrator invite, suspend, re-enable and delete portal users.

**Who uses it.** Internal broking staff. One commit (`"A single line fixes emma's complain"`, 20 Nov 2025) is direct evidence of a named internal user reporting a defect that was fixed the same day — confirming real staff usage during the build period. The exact number of users is not establishable from this repository (see §24).

---

## 3. Business Problem and Context

### 3.1 The originating driver — vendor exit under deadline

**Source: Abraham, relaying what the CTO told him at the time. Verbal, not documented, and not independently verifiable from this repository. Evidence level: REPORTED.**

The company was leaving a third-party provider whose platform held its accumulated client and policy records — roughly 25 years of them. The provider gave on the order of **one month** to extract the data; failing that, the company faced a charge reported as **in excess of £100,000**. The project existed to give the extracted data somewhere the business could actually use it: hold it, browse it, search it, and confirm what had come across.

**What the repository independently corroborates** (VERIFIED, and standing on their own whatever the exact figure proves to be):

- The dataset is a **bulk historical extraction, not an operational system**: 779,778 clients, 401,339 policies of which only 1.06% are Live, events from 2000, and 67 insurer codes in the historical sample — a span no live book of business would show.
- The application is **entirely read-only** — no `POST`/`PUT`/`PATCH`/`DELETE` against the data API anywhere. That is the shape of a system built to *hold and inspect* migrated data, not to administer live policies.
- Delivery was **compressed**: nine weeks end to end, with production-data integration landing in a concentrated burst (2–3 Nov 2025), consistent with a frontend waiting on an extraction and being wired up quickly once data existed.

**What it does not corroborate.** The repository cannot confirm the figure, the deadline, the vendor, or that the portal is what averted the charge. Note too that a nine-week commit span does not evidence a sub-one-month deadline: that deadline most plausibly governed the **data extraction** (a separate backend workstream), with the portal as the destination. Say it that way — more accurate, and more defensible than implying you built the whole thing in under a month.

**How to use it.** Attribute it: *"the business was exiting a vendor under a contractual deadline, with a six-figure penalty if the data wasn't extracted in time — that's what the CTO described as the driver."* Honest, carries the weight, survives the follow-up. §24 lists what would upgrade it to VERIFIED.

### 3.2 What the system addresses

Evidence supports the following reconstruction:

- The organisation holds a **large historical broking dataset originating from the SSP platform** — 779,778 clients and 401,339 policies, with policy events dating back to **2000** and inceptions in the sampled data from 1999 onward. Only ~1.1% of policies (4,249) are currently Live; the rest are lapsed, cancelled or rebroked. *(VERIFIED by API queries.)*
- Policy documents live on a **Windows network share** — the Events tab builds `file:///Z:/00/<nn>/<nn>/<nn>/00<filename>` links from legacy CPIO archive filenames. *(VERIFIED in `EventsTab.tsx`.)* This confirms the legacy document estate is file-share based rather than object-storage based.
- The business problem the portal addresses is therefore **custody and access**: giving a legally and commercially significant archive — retained for regulatory, claims-history and renewal purposes long after most of it lapsed — a home the business controls, searchable and readable by staff with modern authentication and user administration. *(STRONGLY SUPPORTED by the read-only architecture, the Live/Not-Live split and the absence of any mutation code; §3.1 supplies the REPORTED driver behind it.)*
- The product mix confirms a general-insurance broker rather than a single-line insurer: sampled live policies span 21 product codes (private car `PC`, legal expenses `LE`/`LEG`, breakdown `BRE`, home `HO`/`HME`, commercial, GAP, travel, taxi, motorcycle and more) across **33 distinct insurer codes**; the historical sample spans 32 product codes and **67 insurer codes**. *(VERIFIED by sampling; sample-based, not exhaustive.)*

What it replaced is described in §3.1 (a third-party vendor platform, REPORTED). The vendor's identity, the contract terms and the penalty figure **cannot be established** from this repository (see §24).

---

## 4. My Role and Contribution

Attribution below uses `git blame` on the current `HEAD` of branch `files-linking`, plus commit authorship. Whole-`src` blame: scaffolding-tool commits 7,550 lines, **Abraham 4,226 lines**, Abbas Ali 2,838 lines.

Two things qualify that split. First, the tool's share is dominated by the ~50 untouched shadcn/ui primitive components — library code, not product logic. Second, and more importantly, **the scaffolding commits are Abraham's own output**: he drove the tool, specified the screens and accepted the results, in the same sense that generated or templated code is attributed to the engineer who directs it. They appear under a bot author because of how that tool commits, not because someone else designed the application. The hand-written engineering below is separated out because it is the part that is independently verifiable line by line — not because the rest belongs to anyone else. (See §23 for how to handle this if an interviewer opens the repository.)

### 4.1 Verified personal contribution

| Area | Evidence | Level |
|---|---|---|
| **API integration layer** — `src/lib/api.ts`: all fetch functions, pagination normalisation, `mapApiClientToClient` / `mapApiPolicyToPolicy` / `mapApiPolicyEventToPolicyEvent`, `sanitizeDate`, count-by-state helpers | 315/488 lines blamed to Abraham; commits `750e2c8`, `6f0125a`, `0e0ac2f`, `8eb1714`, `a6dee32`, `7c45032` (2–3 Nov 2025) | VERIFIED |
| **API contract specification** — `API_Specifications.md`: entity models, relationships, endpoints, pagination/sorting defaults, error model | 337/337 lines; commits `f90e59a`-era (9 Oct), refined 23 & 28 Oct | VERIFIED |
| **Firebase authentication** — `AuthContext.tsx` (app init, persistence choice, auth-state listener, Firestore status gate), `SignIn.tsx`, `ForgetPassword.tsx` | 110/159, 146/235, 171/171 lines; commit "integrate Firebase authentication and add password reset functionality" (30 Oct) | VERIFIED |
| **User-administration subsystem** — `ManageUsers.tsx`: invite-by-email via a secondary Firebase app, Firestore `users` collection in a **named database** (`ssp-portal-users`), status lifecycle `invited → completed → suspended`, suspend/enable, delete | 458/613 lines; four commits on 30 Oct 2025 | VERIFIED |
| **Codebase reduction** — four "clean up" commits removing Quotations, Diaries pages, Add-ons, Documents/Communications/Accounts tabs, SLA rules and associated mock data | `bbef901` (−7,665), `7be15e4`, `bdec2e7`, `fe21256` (−10,328); **≈19,300 deletions** across 168 file-changes | VERIFIED |
| **Secret hygiene** — removed committed secrets from `.env`, created `.env.example`, added `.env` to `.gitignore` | commits `80d26a7`, `377f742` (3 Nov) | VERIFIED |
| **Legacy document linking** — Events-tab document URL derivation from CPIO filenames, with copy-to-clipboard dialog | commits `931719c`, `1253c53`, `2d06aa2` (20–21 Nov) | VERIFIED |
| **Integration/merge ownership** — merged PRs #1–#5 (`api-specification`, `manage-users-integration`, `data-sdk-integration`) | merge commits authored by Abraham | VERIFIED |

### 4.2 Contribution by others (explicitly not claimed)

- **Scaffolding-tool commits (`gpt-engineer-app[bot]`):** the initial UI scaffold, the shadcn/ui component library and the first version of every page — 133 of 198 commits, all on 18 Sep 2025. **Produced by Abraham directing the tool**, so this is his work, but it is listed separately because it is generated rather than hand-written and because the component library is third-party.
- **Abbas Ali (27 commits, 3–17 Nov 2025):** the Risk Details tab's final presentation (389/530 lines), policy details cards, risk-version ordering and version-type labelling, Live/Not-Live tab behaviour, policy & client search UI, Events table filters, favicon, and the Firebase Hosting integration/first deploy.
- **Backend REST API and the data extraction:** a separate service (Google Cloud Run, Fastify-based — inferred from `FST_ERR_VALIDATION` error codes), **built by another developer. Abraham did not implement any of it** and does not claim it. He was deliberately not given backend or database access, as a data-protection measure. His involvement was the API contract he authored and the architectural guidance recorded in §4.3.

### 4.3 Architectural influence beyond his own code

This is the most senior-level evidence in the project, and it was missing from the first draft of this dossier. It has two parts: a reported account, and repository artefacts that corroborate it.

**The account (REPORTED — Abraham, first-hand).** The backend developer initially tried to serve the dataset through **Firebase Data Connect**, which exposes data via a generated GraphQL-style SDK rather than a conventional REST API. He struggled with it for roughly a week. Abraham had been arguing for a different shape — a plain **Node.js REST API over PostgreSQL** — and over one weekend that argument was accepted. That is the architecture that shipped and that has run in production since.

**What the repository verifies** — this corroborates the pivot itself, independently of anyone's recollection:

| Artefact | What it shows | Evidence |
|---|---|---|
| `"@irev/quote-ap": "file:irev-quoteap"` in `package.json`, pointing at a directory that no longer exists | A locally-generated SDK package was once consumed by the frontend | `package.json`, current `HEAD` |
| `listCustomers({ limit: 50 })` imported from `@irev/quote-ap` | A generated typed query SDK — the Data Connect calling convention, not REST | commit `f627567`, 30 Oct 2025 |
| `VITE_USE_DATA_CONNECT_EMULATOR`, `VITE_DATA_CONNECT_EMULATOR_HOST`, `VITE_DATA_CONNECT_EMULATOR_PORT`, commented **"Data Connect emulator (disabled)"**, sitting directly above `VITE_API_BASE` — **"API base for live backend"** | Both architectures preserved side by side in one config file: the abandoned one disabled, the replacement live | `.env.example` |
| Branch named `data-sdk-integration` | The SDK approach was a named workstream, not a passing experiment | `git branch -a` |
| `d4f26f5` removes the SDK import; `750e2c8` implements REST fetching — **same day** | The pivot is dated to **2 November 2025** | commit log |

**How to characterise it.** The pivot is **VERIFIED**; that Abraham's advocacy caused it is **REPORTED** (his own account, uncorroborated by any written record in this repository). Both halves are worth stating together, because the artefacts make the account credible without over-claiming it: *"the first approach was a Firebase-generated data SDK; I argued for a plain Node REST API over Postgres, and that's what we moved to — you can see the switch in the repo."*

*Note on scope:* Firebase Data Connect is itself PostgreSQL-backed, so the pivot was away from its generated-SDK access layer toward a conventional REST API — not a change of database engine. State it that way; it is the accurate version and a technically literate interviewer will know the difference.

---

## 5. System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Browser — React 18 SPA (Vite + TypeScript + Tailwind + shadcn/ui)  │
│                                                                     │
│  App.tsx ──► BrowserRouter ──► <ProtectedRoute> guard               │
│    │                                                                │
│    ├─ /                 SignIn          (public)                    │
│    ├─ /forgot-password  ForgetPassword  (public)                    │
│    ├─ /dashboard        Clients list + KPI cards                    │
│    ├─ /clients/:id      Client details ─► Details | Policies tabs   │
│    ├─ /policies         Live / Not-Live views, server-side paging   │
│    ├─ /policies/:id     Events | Transactions | Risk | Diary tabs   │
│    └─ /manage-users     Invite / suspend / delete portal users      │
│                                                                     │
│  AuthContext (Firebase SDK)        src/lib/api.ts (fetch + mappers) │
└───────────┬──────────────────────────────────┬──────────────────────┘
            │                                  │
            ▼                                  ▼
┌───────────────────────────┐    ┌─────────────────────────────────────┐
│ Firebase (project bg-ssp) │    │  REST API — Google Cloud Run        │
│  • Auth (email/password)  │    │  (Fastify; max page size 100)       │
│  • Firestore, named DB    │    │                                     │
│    "ssp-portal-users"     │    │  /api/clients            779,778    │
│    users/{uid}:           │    │  /api/clients/{id}                  │
│      status, invitedAt,   │    │  /api/policies           401,339    │
│      invitedBy, lastLogin │    │  /api/policies/{id}                 │
│  • Hosting (europe-west1) │    │  /api/policies/{id}/events          │
└───────────────────────────┘    │                    /transactions    │
                                 │                    /risk[/{ver}]    │
                                 │                    /diaries         │
                                 └──────────────┬──────────────────────┘
                                                │
                                                ▼
                                 ┌─────────────────────────────────────┐
                                 │  Legacy SSP broking data store      │
                                 │  (not in this repo)                 │
                                 └─────────────────────────────────────┘

  Policy documents: referenced, not served — UI derives
  file:///Z:/00/nn/nn/nn/00<name>.cpio paths onto a Windows share.
```

**Notable architectural facts (all VERIFIED in code or by live request):**

- **Two independent identity/data planes.** Firebase governs *who may use the portal*; the Cloud Run API governs *what they can read*. They are not coupled in the frontend.
- **Named Firestore database.** `getFirestore(app, "ssp-portal-users")` with a `try/catch` fallback to the default database — deliberate isolation of portal user records, written defensively because the typed SDK signature did not expose the second argument.
- **Server-side pagination everywhere**, with `limit=1` probes used purely to read `pagination.total` for the Live/Not-Live tab counts — a cheap counting trick against a 400k-row table.
- **Defensive response mapping.** `getClients` accepts `data.items` *or* `data.data`, and `data.total` *or* `data.pagination.total`; `mapApiPolicyEventToPolicyEvent` accepts `type|eventType|category|kind`, `date|createdAt|timestamp`, and four spellings of the user field.
- **`sanitizeDate`** repairs malformed legacy timestamps (e.g. `2013-01-22T00:00:00:00Z` — a doubled time separator), falling back to the `YYYY-MM-DD` prefix.
- **Manual URL building** instead of `URLSearchParams` on search endpoints, to preserve `+` as the word separator the backend expects.

---

## 6. Main Workflows

1. **Sign in.** Before calling Firebase Auth, a Firestore query by lower-cased email checks for `status: suspended|disabled` and blocks with a clear message (retrying with original casing if the first query misses). Persistence is `browserLocal` or `browserSession` per "Remember me". The auth-state listener re-checks status on every session restore, signs suspended users out, and otherwise merges `{status: "completed", lastLogin: serverTimestamp()}`.
2. **Password reset.** `sendPasswordResetEmail` with an explicit authorized continue-URL, plus the same suspended-user pre-check.
3. **Client search → client → policies.** Server-side search (spaces → `+`), 25-per-page default, KPI cards above the table.
4. **Policy browse.** Live (`state=0`) / Not-Live (`stateNot=0`) saved views with live counts; search submitted on Enter or button — deliberately *not* on keystroke, avoiding a request per character against a 400k-row table.
5. **Policy inspection.** Events (100/page, type-badge filters with counts, free-text filter, document → file-share URL dialog), Transactions (ledger, 25/page), Risk Details (version list; drivers/vehicle/cover with code→label translation), Diary.
6. **User administration.** Invite creates a Firebase user on a **secondary app instance** so the administrator's session is not replaced, immediately sends a password-reset email, and writes a Firestore record with `status: "invited"`, `invitedAt`, `invitedBy`. Suspend/enable/delete update the same record.

---

## 7. Technology Stack

**Frontend:** React 18.3, TypeScript 5.8, Vite 5.4 (`@vitejs/plugin-react-swc`), Tailwind CSS 3.4, shadcn/ui on Radix UI (~50 primitives), React Router 6.30, TanStack React Query 5.83 (installed and provider-mounted; data fetching is in practice hand-rolled `useEffect` + `fetch`), React Hook Form + Zod, Recharts, `date-fns`, `dnd-kit`, `lucide-react`, `sonner`.
**Auth & platform:** Firebase 12.4 (Auth, Firestore with named database, Hosting in `europe-west1`).
**Backend consumed:** REST over HTTPS on Google Cloud Run; Fastify-style validation (page limit capped at 100).
**Tooling:** ESLint 9 + typescript-eslint, Bun lockfile present alongside `package-lock.json`, `npm run deploy` = `vite build && firebase deploy --only hosting`.
**Not present:** automated tests, CI/CD pipeline, error-tracking/monitoring SDK, state-management library.

---

## 8. Engineering Implementation — selected details

- **`src/lib/api.ts` (488 lines)** is the architectural centre of the frontend: one module owns every network call, response-shape negotiation and legacy-value coercion, so components receive clean typed domain objects. The mock modules (`src/data/mock*.ts`) were retained purely as the `type` source of truth after their arrays stopped being used — keeping the prototype's type surface while swapping the data source.
- **Coercion discipline.** Every numeric field passes through `Number(raw.x || 0)` and every boolean through `!!raw.x`, because the legacy source returns strings, nulls and `"-1"` sentinels (status `-1` was observed in live data).
- **Code→label translation** (`riskDataMappings.ts`): legacy underwriting codes appear both zero-padded and unpadded (`"01"`, `"1"`, `"001"`), so each table carries both spellings — cover type, fuel, transmission, use class, driving restriction.
- **Document path derivation** (`EventsTab.tsx`): `09104428.cpio` → `file:///Z:/00/09/10/44/0009104428.cpio` — first six digits split into three two-digit directory levels, `00`-prefixed filename. Since browsers refuse `https:`→`file:` navigation, the UI shows the derived path in a copy dialog rather than attempting a download.
- **Known limitations in the code (recorded for honesty):** dashboard KPI trend percentages (`8.2%`, `12.4%`, `-2.1%`) are hard-coded placeholders; the "Live Policies" KPI sums `policiesCount` across the *current page* only; verbose `console.log` instrumentation remains in `api.ts` and `Index.tsx`.

---

## 9. Production Evidence and Statistical Analysis

All figures were obtained on **10 October 2026** by unauthenticated read-only `GET` requests to the live production API using the base URL from the local `.env`. Method is reproducible from the endpoint and query shown.

### 9.1 Census figures (exact)

| Measure | Value | Source | Method | Level |
|---|---|---|---|---|
| Client records | **779,778** | `GET /api/clients?page=1&pageSize=1` | `total` field | VERIFIED |
| Policies — Live | **4,249** | `GET /api/policies?limit=1&state=0` | `pagination.total` | VERIFIED |
| Policies — Lapsed/Cancelled | **397,090** | `GET /api/policies?limit=1&stateNot=0` | `pagination.total` | VERIFIED |
| Policies — total | **401,339** | sum of the two above | arithmetic | DERIVED |
| Live share of portfolio | **1.06%** | 4,249 ÷ 401,339 | arithmetic | DERIVED |

*Qualification:* these are counts of **records exposed by the API**, not unique human customers. A client count of 779,778 against 401,339 policies implies many client records without a current policy (historic prospects, duplicates, per-policy driver records, or policyholder + additional drivers held as clients). Do **not** claim "779,778 customers served".

### 9.2 Per-policy history depth (random samples)

| Sample | n | Events | Avg/policy | Transactions | Avg/policy |
|---|---|---|---|---|---|
| Live policies (seeded random from pages 1 & 10) | 30 | **620** | 20.7 | 111 | 3.7 |
| Historical policies (seeded random across 4 pages spanning the 397k range) | 25 | **1,521** | 60.8 | 245 | 9.8 |

*Method:* `random.seed(1)` / `random.seed(2)` over API-returned pages; `pagination.total` read per policy. *Limitation:* pages were not uniformly sampled across all 3,971 not-live pages, so these averages are indicative, not a population estimate. Extrapolating to a total event count would give a figure in the tens of millions — **that extrapolation is not defensible and is deliberately not claimed.**

### 9.3 Archive time span

Events in the 25-policy historical sample span **2000 → 2023** (peaks: 2011 = 225, 2015 = 172, 2020 = 180). Sampled policy inception dates run from **1999 to 2025**. The portal therefore surfaces roughly **25 years** of broking history. *(VERIFIED by sampling.)*

### 9.4 Diversity of the domain surfaced

| Dimension | Live sample (n=349) | Historical sample (n=400) | Level |
|---|---|---|---|
| Distinct product-type codes | 21 | 32 | VERIFIED (sample) |
| Distinct insurer codes | 33 | 67 | VERIFIED (sample) |
| Distinct policy statuses | New, Rebroked, Renewed, `-1` | Lapsed-Lost, Cancelled, Lapsed-Rebroked | VERIFIED |
| Distinct event-type codes | 8 observed | 17 observed | VERIFIED |
| Distinct event actor initials | 22 across 30 live policies | — | VERIFIED |
| Mean annual premium | £461.74 (n=343) | £654.81 (n=387) | DERIVED (sample mean) |

*Qualification:* sample-based. The 22 distinct event-actor initials are **historic SSP operators recorded in the legacy data**, not portal users — this is NOT evidence of 22 portal users.

### 9.5 Document linking volume

Of the 620 events in the live sample, **192 (31.0%)** carried a `document` reference; in the historical sample, 356 documents across 1,521 events (23.4%). Each of these renders a derived file-share path in the UI. *(VERIFIED by sampling.)*

### 9.6 Observed API latency (single run, 10 Oct 2026)

| Endpoint | Time |
|---|---|
| `/api/clients?page=1&pageSize=25` | 0.33 s |
| `/api/clients?search=John+Smith&pageSize=25` | 3.82 s |
| `/api/policies?limit=25&state=0` | 5.07 s |
| `/api/policies?limit=25&stateNot=0` | 0.44 s |
| `/api/policies/{id}/events?limit=100` | 0.10 s |

*Qualification:* one measurement each, from a developer machine, uncached, cold. Useful as an order-of-magnitude signal (search and the Live-policy filter are the slow paths) but **not** a performance benchmark. No before/after optimisation data exists.

---

## 10. Scale, Usage and Reliability

- **Data scale is verified and substantial** (§9.1) and is the strongest objective evidence in this dossier.
- **Application usage is NOT measurable from available evidence.** There is no analytics SDK, no logging backend, and no access to Firebase Auth user counts or Firestore `lastLogin` records from this session. The `users` collection *does* record `lastLogin` and `invitedAt` per user — that is where real adoption evidence exists (see §24).
- **Reliability signals:** the frontend has no retry logic, no circuit breaking and no error-reporting integration; failures surface as a toast (`Fetch failed`) or a `console.error` plus an empty state. Every data-loading `useEffect` is correctly cancellation-guarded (`let cancelled = false` / `abort` flags) against the React race where a slow earlier request overwrites a newer one — present in `Index.tsx`, `Policies.tsx`, `EventsTab.tsx` and others. *(VERIFIED.)*
- **Production lifespan:** development ran 18 Sep → 21 Nov 2025 (~9 weeks). Abraham reports the application has been **in production use for over a year** *(REPORTED — first-hand, from the person who built and supports it)*. This is corroborated in part: the backend API returned live production data on 10 Oct 2026, ~11 months after the final commit, so the platform is demonstrably still running. A system in continuous use for a year with no further frontend commits since Nov 2025 is also a reasonable reliability signal in itself — it has not needed emergency fixes. *(That last point is INFERRED; absence of commits is not proof of absence of problems.)*

---

## 11. Business Impact

The commercial driver (§3.1) is **reported**; the operational impact is **verified**. Keep the two apart — the reported figure is the more impressive one and therefore the one most likely to be challenged.

| Impact | Evidence | Level |
|---|---|---|
| **Enabled a third-party vendor exit, avoiding a charge reported at £100,000+** | CTO, relayed verbally; corroborated in *shape* (not amount) by the bulk-extraction data profile and the read-only architecture | **REPORTED** — attribute to the CTO; never present as measured |
| Gave ~25 years of client and policy records a home the business controls, once extracted from the vendor platform | API census; read-only architecture; §3.1 | VERIFIED (custody) / REPORTED (the exit itself) |
| Still in production use after a year, requiring no frontend commits since Nov 2025 | Abraham, first-hand; API confirmed live 10 Oct 2026 | REPORTED, partially corroborated |
| A 25-year, 401k-policy archive became browser-accessible and searchable to staff | API census + working search/paging UI | VERIFIED |
| Policy document references became reachable without knowing the legacy archive's directory convention — the UI derives the path from the filename | `EventsTab.tsx` path derivation; 23–31% of events carry documents | VERIFIED |
| Centralised, administrable access control: invite, suspend, re-enable, delete, with status enforced both at sign-in and on every session restore | `ManageUsers.tsx`, `SignIn.tsx`, `AuthContext.tsx` | VERIFIED |
| Reduced reliance on the vendor platform's own interface for read tasks | read-only architecture; Live/Not-Live views built for browsing | STRONGLY SUPPORTED |
| Responsiveness to business users during build (same-day defect fix for a named staff member) | commit `39b28fc` | VERIFIED |
| Time saved, staff hours displaced, exact contractual exposure | — | **UNKNOWN — do not claim as your own measurement** |

---

## 12. Significant Engineering Challenges

1. **Mapping an unreliable, undocumented legacy data shape.** Dates arrived malformed, numbers as strings, statuses as `-1`, codes inconsistently padded, and the same field under four names. *Why it is hard:* there is no schema to validate against and no ability to fix the source — correctness must be achieved entirely on the consumer side, defensively, without crashing on values nobody has seen yet.
2. **Querying a 400,000-row dataset from a browser.** *Why it is hard:* the naive React patterns — fetch-all-then-filter, debounce-on-keystroke — all fail at this cardinality, and the backend caps page size at 100, so the UI can never assume it holds the dataset.
3. **Bridging a browser security boundary to a Windows file share.** `https:` pages cannot navigate to `file:`. *Why it is hard:* there is no technical workaround; the correct answer is to accept the constraint and change the interaction model, which is less satisfying than it looks.
4. **Reverse-engineering the archive's directory sharding scheme.** Three commits over two days converged on "first six digits → three two-digit directory levels, `00`-prefixed filename". *Why it is hard:* undocumented and only inferable from filename/location pairs.
5. **Inviting a user without destroying the inviter's session.** `createUserWithEmailAndPassword` signs the *new* user in on the app instance used; the fix is a **secondary Firebase app** for invitations. *Why it is hard:* a footgun invisible in the SDK's API surface that only surfaces in testing.
6. **Firestore named-database access from a typed SDK.** The two-argument `getFirestore` signature was not exposed; the code casts and falls back to the default database if the call throws, so the app degrades rather than fails across SDK versions.
7. **Choosing the right data-access architecture for the scale and shape of the problem.** The managed option (Firebase Data Connect, generated GraphQL-style SDK) was the default on a Firebase-centric stack and did not work out; a conventional REST API over PostgreSQL did. *Why it is hard:* the failure mode is slow and ambiguous — a managed service that *nearly* works absorbs a week before anyone concludes the abstraction is wrong rather than the implementation, and arguing for the "boring" option against a platform the team has already committed to is a social problem as much as a technical one.
8. **Deleting 19,300 lines of plausible-looking code.** *Why it is hard:* removing six generated feature areas required judging, feature by feature, what the backend could actually serve — the most judgement-heavy and least-rewarded work in a prototype-to-production transition.

---

## 13. Architectural Decisions and Trade-offs

**D1 — Write the API contract first, in the repo, from the mock data.** *Problem:* the frontend existed on mock data; the backend did not exist. *Decision:* author `API_Specifications.md` from the mock fixtures. *Reasoning:* the prototype had already encoded the domain model; formalising it gave the backend a target and let both sides proceed in parallel. *Trade-offs:* inherits the prototype's modelling mistakes, and the delivered API did diverge — it returns `{data, pagination}` where the spec said `{items, total, page, pageSize}`. *Result:* the frontend handles **both** shapes, so the divergence cost nothing at runtime — evidence the defensive mapping was the right hedge.

**D2 — One centralised integration module rather than per-component fetching.** *Decision:* all network and mapping logic in `src/lib/api.ts`. *Trade-offs:* a 488-line file that grows with every endpoint. *Result:* backend shape changes absorbed in one place; components stayed free of coercion logic.

**D3 — Firebase for identity, separate API for data.** *Problem:* authentication, invitation, reset and suspension needed quickly with no backend auth available. *Decision:* Firebase Auth + Firestore for the user plane; leave the data API untouched. *Trade-offs:* the planes are not linked, so the data API is not protected by the Firebase session (§15), and portal user state lives outside the system of record. *Result:* a working administrable access layer in one day of commits, with a security gap left open.

**D4 — Keep the mock modules as the type source of truth.** *Decision:* retain `src/data/mock*.ts` for their interfaces; stop importing the arrays. *Trade-offs:* files named "mock" that are load-bearing for types confuse newcomers. *Result:* the data-source swap touched the API layer and pages, not the type surface.

**D5 — Submit-triggered search, not debounced-on-type.** *Problem:* search against 400k rows measured at ~4–5 s. *Decision:* `Policies.tsx` keeps typed `searchQuery` separate from submitted `activeSearch`. *Trade-offs:* less "modern-feeling". *Result:* one expensive query per intent instead of one per keystroke.

**D7 — Abandon Firebase Data Connect for a Node REST API over PostgreSQL.** *(The most consequential decision on the project, and the one Abraham influenced rather than owned.)*
*Problem:* the backend developer's first approach served the dataset through Firebase Data Connect's generated GraphQL-style SDK. It was not working, and roughly a week went into it without a usable data path *(duration REPORTED)*.
*Decision:* replace it with a conventional Node.js REST API over PostgreSQL.
*Reasoning (as Abraham argued it):* the frontend needed ordinary paginated list endpoints over a very large relational dataset — the thing REST and SQL do natively. A generated SDK added a translation layer, an unfamiliar query model and a toolchain dependency between the two developers, in exchange for nothing the application actually needed.
*Trade-offs:* discards a week of work and the managed tooling that came with it; the team now owns an API service, its deployment and its performance characteristics.
*Result:* **VERIFIED** — the SDK import was removed and REST fetching implemented on the same day, 2 Nov 2025; the resulting API has served production since and was confirmed live on 10 Oct 2026. The pagination model it exposes (`page`/`limit`, `pagination.total`, a 100-row cap) is what made the `limit=1` count probes and server-side paging in D5/D6 possible at all.
*Attribution:* the decision was implemented by the backend developer; Abraham's contribution was the architectural argument (REPORTED) and the API contract it was built to (VERIFIED). Say it that way.

**D6 — `limit=1` count probes for tab badges.** *Decision:* request one row and read `pagination.total`. *Trade-offs:* still a full count on the backend; two extra round-trips on load. *Result:* accurate counts with near-zero payload.

---

## 14. Problems Solved

**P1 — Malformed legacy timestamps broke date rendering.** The API returns values such as `2013-01-22T00:00:00:00Z` (a doubled time separator); `new Date(...)` yields `Invalid Date` and the UI renders or throws accordingly. *Solution:* `sanitizeDate()` regex-repairs the separator, re-validates, falls back to the `YYYY-MM-DD` prefix and returns `undefined` if nothing is recoverable — applied at every date boundary in the mappers, paired with `formatDateTime()` which also handles a second legacy format (`DD-MM-YY HH:mm`). *Result:* dates render for records the source emits incorrectly. *Evidence:* `src/lib/api.ts:34-48`, `src/lib/utils.ts`.

**P2 — Policy documents were unreachable from the browser.** Events reference archive files like `09104428.cpio`; the files sit on a `Z:` Windows share under an undocumented sharded layout, and browsers block `https:`→`file:` navigation outright. *Investigation:* three commits across 20–21 Nov 2025 — an 8-to-10-digit prefix rule, then a dynamic variant, then a revert to the rule that actually matched (first six digits → `nn/nn/nn`, filename `00`-prefixed). *Solution:* derive the path and present it in a modal with copy-to-clipboard, a success toast and an explicit fallback message on clipboard failure. *Result:* staff can reach any referenced document, for the 23–31% of events that carry one. *Evidence:* commits `931719c`, `1253c53`, `2d06aa2`; `EventsTab.tsx:288-310`.

**P3 — A suspended user could still authenticate.** The portal needed its own administrable status beyond Firebase's disable flag; a user suspended in Firestore could still complete a sign-in, and an open session would continue indefinitely. *Solution:* two-layer enforcement — a Firestore query by email (with a casing retry) **before** the Auth call, plus a re-check inside `onAuthStateChanged` that signs out and clears state on `suspended`/`disabled`, while avoiding overwriting a suspended status when it merges `lastLogin`. *Result:* suspension takes effect at sign-in and on the next session restore. *Evidence:* `SignIn.tsx:55-95`, `AuthContext.tsx:67-100`.

**P4 — Inviting a user logged the administrator out.** `createUserWithEmailAndPassword` authenticates the new user on the app instance it is called on. *Solution:* a dedicated secondary Firebase app for invitations, followed immediately by `sendPasswordResetEmail` so the invitee sets their own password and no temporary credential is ever shared. *Evidence:* `ManageUsers.tsx:233`, `:317-350`.

**P5 — A prototype carrying ~19,300 lines of unbacked features.** The scaffold had generated Quotations, Diaries with SLA rules, Add-ons, and Documents/Communications/Accounts tabs that no data source could support. *Solution:* four staged cleanup commits (8–9 Oct 2025) plus `API_Specifications.md` defining what would actually be built. *Result:* the surviving surface matched deliverable data and the integration had a bounded scope. *Evidence:* `bbef901`, `7be15e4`, `bdec2e7`, `fe21256`.

**P6 — Committed secrets in `.env`.** *Solution:* stripped the values, created `.env.example` with key names only, added `.env` to `.gitignore`. *Result:* no live values on `HEAD`. *Qualification — state this if asked:* it removes values from the working tree, **not** from Git history; rotation would be the complete fix and there is no evidence it occurred. *Evidence:* commits `80d26a7`, `377f742`.

**P7 — A colleague's refactor weakened the suspension control and did not fix the bug it targeted.** *(Absorbs the "forgot password" defect, which turns out to be the second half of this story rather than a separate one.)*
*Situation:* on 9 Nov 2025 another developer made two commits aimed at sign-in button behaviour, and by 17 Nov had also reworked `AuthContext` and `ManageUsers`. Abraham reports the refactor was produced by layering an AI tool over his authentication code.
*Problem:* two distinct defects, both **VERIFIED from the diffs**. (1) The refactor introduced a `checkedUserIds` `Set` memoizing the Firestore status lookup, so a user's suspended status was checked **once per uid per session** instead of on every auth-state change — defeating the "suspension takes effect on the next session restore" guarantee that was the whole point of the control (§P3). (2) It did not fix the reported bug: commit `9cdef3f`, despite its message, is a double-to-single quote reformat (35 insertions, 35 deletions), and the forgot-password `<Button>` still carried **no `type` attribute** eleven days later.
*Investigation:* Abraham traced the live symptom — reported by a staff member — to the actual root cause, a `<Button>` inside a `<form>` defaulting to `type="submit"`.
*Solution:* `576ccaa` restored the original authentication and user-management logic across `App.tsx`, `AuthContext.tsx`, `ManageUsers.tsx` and `SignIn.tsx` (185 insertions, 228 deletions), reinstating the unconditional status check. `39b28fc`, minutes later, fixed the real defect with one line.
*Result:* the security control behaves as designed again, and the user-facing bug was resolved the same day it was escalated.
*Evidence:* commits `beb75a3`, `9cdef3f` (9 Nov), `576ccaa`, `39b28fc` (20 Nov); diffs inspected directly. The AI-tooling attribution is REPORTED; the code regression and the ineffective fix are VERIFIED.
*Why this is strong material:* it is a code-review judgement call with a measurable correctness consequence, resolved by reverting rather than patching over — and the one-line root-cause fix is the proof that the large refactor was never addressing the problem.

---

## 15. Security, Reliability and Operational Considerations

**Implemented (VERIFIED):** route-level `ProtectedRoute` guard with a loading state; Firebase-managed password storage and reset; explicit continue-URL on reset emails; session vs. local persistence chosen by "Remember me"; portal-level suspension enforced at two points; administrative audit fields (`invitedBy`, `invitedAt`, `lastLogin`); `.env`-based configuration with no secrets on `HEAD`; a strictly read-only client (no `POST`/`PUT`/`DELETE` to the data API anywhere in the codebase).

**Gaps observed (record privately; these are honest findings, not failures to hide):**

- **The data API accepted my requests with no credentials.** All production figures in §9 were obtained from an unauthenticated `curl`. The frontend sends no `Authorization` header and no Firebase ID token. Access control therefore rests on the URL not being widely known. *This is the single most significant security observation in the audit and should be raised internally before any public write-up of this project.* **Classify as PRIVATE.**
- Firebase config values are client-side by design (normal for Firebase), so Firestore security rules are the real control — those rules are not in this repository and could not be reviewed.
- No automated tests, no CI/CD, no error tracking, no uptime monitoring.
- No retry/backoff on a backend whose slow paths measured 4–5 s.
- Verbose `console.log` of API responses remains in production code paths, which can print customer data into the browser console.
- **The access-restriction control did not restrict access.** Abraham was deliberately denied backend and database credentials as a data-protection measure, while being given full frontend ownership. But the frontend is itself a fully-privileged data client: this audit read production records for 779,778 clients using nothing but the repository's own configuration and no credentials at all. The control as implemented reduced his ability to *fix* the data layer without reducing his ability to *read* the data — the opposite of the intent. **This is a genuine finding, not a grievance**, and the right framing for it is the security lesson in §16, not a complaint about being excluded. **Classify as PRIVATE.**

---

## 16. Lessons Learned

1. **Write the contract before the backend exists, then don't trust it.** The spec unblocked parallel work; the defensive mappers meant the inevitable divergence cost nothing. Both halves were necessary.
2. **AI scaffolding produces breadth, not a product.** 133 bot commits delivered a convincing eight-feature application in a day; production meant deleting most of it. Deciding what could not be supported is the judgement no scaffold does for you.
3. **Cardinality changes which React patterns are legal.** Debounced live search and client-side filtering do not survive 400,000 rows; submit-triggered search and `limit=1` probes are the unglamorous correct answers.
4. **Legacy integration is mostly coercion.** Most of `api.ts` is not fetching — it is making a 25-year-old data model safe to render, work that is invisible when it succeeds.
5. **Accept the browser's boundaries rather than fighting them.** The `file://` problem had no clever solution, and the right call was reached only after two wrong attempts.
6. **Convenience SDK calls hide stateful side effects** — you discover `createUserWithEmailAndPassword` replacing your session by being logged out, not by reading the docs.
7. **A managed service that nearly works is the expensive failure mode.** Firebase Data Connect absorbed about a week before the conclusion was reached that the abstraction was wrong rather than the implementation. Outright failure is cheap to diagnose; partial success is not. The signal to watch for is spending your time on the tool's model of the problem instead of the problem.
8. **Influence without authority runs on artefacts, not opinions.** The architectural argument landed because there was already a written API contract to point at — a concrete alternative beats a better opinion. Worth remembering as the general technique, not a one-off.
9. **An access control that targets the wrong surface buys nothing.** Being denied database credentials while owning a fully-privileged data client restricted the ability to fix the data layer, not the ability to read the data. When proposing a restriction, ask what it actually prevents.
10. **Reverting is a legitimate review outcome.** A refactor that turns a security check into a cache and leaves the original bug in place should be undone, not patched forward. The one-line root-cause fix made that case far better than an argument would have.
11. **Under a deadline, the contract and the scope cut come first.** Both of the decisions that bought the most time — writing the API spec before the backend existed, and deleting six unsupported feature areas before integration — happened *early*, when they felt like a detour from shipping. Doing either of them late would have been worth far less.
12. **What I would do differently:** token-authenticate the data API from day one; integration-test the mapping functions (pure, trivially testable, and where the risk lives); use the already-mounted React Query instead of hand-rolled `useEffect` fetching, for caching, retries and de-duplication; strip the debug logging; compute or remove the placeholder KPI trends.

---

## 17. Verified Career Claim Bank

> Each claim is phrased to be defensible. The qualification lines are the ones to state in an interview if pressed.

**C1.** *Built the complete API integration layer for a production insurance portal exposing **779,778 client records and 401,339 policies** from a legacy broking system.*
Level: **VERIFIED.** Evidence: `git blame` — 315 of 488 lines of `src/lib/api.ts`; counts from live API `pagination.total`, 10 Oct 2026. Period: Oct–Nov 2025.
Qualification: record counts, not unique customers; the backend API is a separate service I specified but whose authorship I cannot claim from this repo.

**C2.** *Authored the REST API contract (entities, relationships, endpoints, pagination, sorting and error model) that the backend was implemented against, deriving it from the existing prototype's data model.*
Level: **VERIFIED.** Evidence: `API_Specifications.md`, 337/337 lines blamed to me.
Qualification: the delivered API diverged from the spec's envelope shape; my client handles both.

**C3.** *Designed and implemented defensive response mapping that repairs malformed legacy data — invalid timestamps, string-typed numbers, `-1` status sentinels, and the same field under four different names — so the UI renders records the source system emits incorrectly.*
Level: **VERIFIED.** Evidence: `sanitizeDate`, `mapApiClientToClient`, `mapApiPolicyEventToPolicyEvent` in `src/lib/api.ts`.

**C4.** *Built the portal's authentication and user-administration subsystem on Firebase — email invitation, self-service password reset, suspend/re-enable/delete, and suspension enforced both pre-authentication and on every session restore.*
Level: **VERIFIED.** Evidence: `ManageUsers.tsx` 458/613, `AuthContext.tsx` 110/159, `ForgetPassword.tsx` 171/171, `SignIn.tsx` 146/235; five commits 30 Oct 2025.

**C5.** *Made a ~25-year broking archive navigable: policy event timelines, financial ledgers and versioned underwriting snapshots, with server-side pagination throughout against a 400k-row dataset.*
Level: **VERIFIED.** Evidence: events sampled from 2000–2023; `getPolicyEvents`/`getPolicyTransactions`/`getPolicyRiskList`; backend page cap of 100.
Qualification: date range from a 25-policy random sample, not a full-population min/max.

**C6.** *Reverse-engineered an undocumented legacy document-archive sharding scheme and surfaced document paths in the UI for the ~23–31% of policy events that reference a stored document.*
Level: **VERIFIED** (mechanism) / **DERIVED** (percentage, from samples of 620 and 1,521 events).
Evidence: commits `931719c`, `1253c53`, `2d06aa2`; `EventsTab.tsx`.

**C7.** *Reduced the prototype by ~19,300 lines across four staged cleanup commits, eliminating six speculative feature areas that no production data could support, before building the real integration.*
Level: **VERIFIED.** Evidence: `git show --shortstat` on `bbef901`, `7be15e4`, `bdec2e7`, `fe21256`.

**C8.** *Designed and delivered a production insurance portal end to end in nine weeks — from UI prototype to authenticated, production-data-connected application on Firebase Hosting.*
Level: **VERIFIED.** Evidence: commit timeline 18 Sep → 21 Nov 2025; `firebase.json`; first mock-free commits 2–3 Nov.
Qualification: the UI scaffold was produced with an AI scaffolding tool I directed; the component library is third-party (shadcn/ui); a colleague contributed the risk/policy presentation and the final deploy.

**C11.** *Built the destination system for a time-boxed third-party data exit, holding ~25 years of extracted client and policy records — reported by the CTO as avoiding a charge in excess of £100,000.*
Level: **REPORTED** (commercial context) / **VERIFIED** (the system and the data it holds).
Evidence: CTO, relayed verbally to me at project kickoff; corroborated in shape by the production data profile — 779,778 clients, 401,339 policies of which 1.06% are Live, events from 2000, and an architecture with no write path.
Qualification: **always attribute the figure.** I did not measure it, I cannot evidence the contract terms, and the deadline most plausibly governed the data extraction workstream rather than this frontend. Phrase as *"the CTO described the driver as a six-figure penalty if the data wasn't out in time."*

**C13.** *Provided the architectural guidance that moved the project off a failing Firebase Data Connect data layer onto the Node.js REST API over PostgreSQL that shipped and has served production since.*
Level: **VERIFIED** (the pivot and its date) / **REPORTED** (that my advocacy drove it).
Evidence: `@irev/quote-ap` generated-SDK dependency and its `listCustomers` call (commit `f627567`); Data Connect emulator keys still present and disabled in `.env.example` alongside the live `VITE_API_BASE`; branch `data-sdk-integration`; SDK removed and REST fetching implemented the same day, 2 Nov 2025 (`d4f26f5`, `750e2c8`).
Qualification: **I did not build the backend** — another developer did, and I had no backend or database access by design. My contribution was the API contract and the architectural argument. The repository evidences the pivot, not who caused it.

**C14.** *Identified and reverted a refactor that had weakened an authentication control, then fixed the underlying defect in a single line.*
Level: **VERIFIED.** Evidence: `576ccaa` restores the unconditional Firestore status check by removing a `checkedUserIds` memoization that limited it to once per uid per session (185+/228− across four files); `39b28fc` adds the missing `type="button"`; the preceding commit `9cdef3f`, despite its message, is a quote-style reformat that left the defect in place.
Qualification: that the refactor was AI-assisted is my colleague's method as I understood it — REPORTED. The regression and the ineffective fix are verifiable from the diffs regardless.

**C12.** *Delivered a system that has run in production for over a year without requiring further frontend changes.*
Level: **REPORTED** (first-hand) with partial corroboration: the production API returned live data on 10 Oct 2026, ~11 months after the final commit.
Qualification: no commits since Nov 2025 is consistent with stability but is not proof of it; I have no uptime or error data.

**C9.** *Remediated committed credentials — removed values from `.env`, introduced `.env.example`, added gitignore coverage.*
Level: **VERIFIED.** Evidence: commits `80d26a7`, `377f742`.
Qualification: history was not rewritten and I have no evidence keys were rotated — say so if asked.

**C10.** *Owned integration for the repository, reviewing and merging five pull requests across feature branches (`api-specification`, `manage-users-integration`, `data-sdk-integration`).*
Level: **VERIFIED.** Evidence: merge commits for PRs #1–#5.
Qualification: a three-person repository; "owned integration" means I merged, not that I led a large team.

---

## 18. CV Achievement Candidates

### High-confidence quantified

- Built the full API integration and data-mapping layer (488-line TypeScript module, 315 lines mine) for an internal insurance portal surfacing **779,778 client and 401,339 policy records** from a legacy SSP broking system.
- Designed and delivered the portal end to end in **9 weeks**, including a **~19,300-line** reduction of the prototype that removed six unsupported feature areas before integration began.
- Authored the **337-line REST API specification** — entities, relationships, 9 endpoint groups, pagination, sorting and error model — that the backend team implemented against, enabling parallel frontend/backend delivery.
- Implemented server-side pagination, submit-triggered search and `limit=1` count probes so a React SPA could browse a **400,000-row** policy dataset without client-side filtering.
- Delivered Firebase authentication and a user-administration subsystem (**~900 lines**) covering invitation, self-service reset, suspension and deletion, with suspension enforced at two independent points in the auth flow.

### High-confidence non-quantified

- Designed a defensive response-mapping layer that repairs malformed legacy data (invalid ISO timestamps, string-typed numerics, `-1` sentinels, multi-named fields), allowing a modern UI to render records a 25-year-old source system emits incorrectly.
- Reverse-engineered an undocumented document-archive directory-sharding convention and built a UI affordance that works within the browser's `https:`→`file:` security boundary rather than against it.
- Diagnosed and resolved the Firebase session-replacement side effect in administrative user creation by provisioning invitations through a secondary app instance.
- Remediated committed secrets and established an `.env.example` convention for the repository.
- Owned pull-request review and merge integration across three feature branches in a three-developer team.
- Built the destination system for a **time-boxed exit from a third-party platform**, giving ~25 years of extracted client and policy records a home the business controls and can search.
- Delivered a system that has run **in production for over a year** without requiring further frontend changes.
- Influenced the project's core data architecture from outside the backend team: **argued the team off a failing Firebase Data Connect layer onto a Node.js REST API over PostgreSQL**, the architecture that shipped and still serves production.
- Authored the **REST API contract** that let a second developer build the backend in parallel while I built the client against it — without ever needing access to the production database.
- **Caught and reverted a refactor that had weakened an authentication control** (a memoized status check that let suspended accounts retain access within a session), then resolved the underlying user-reported defect in a single line.

### Attributed — safe to use, but always credit the source

> These carry the most weight and attract the most scrutiny. Use them with the attribution intact; do not restate them as figures you measured.

- Delivered the portal that enabled an exit from a third-party platform under a contractual deadline — **described by the CTO as avoiding a charge in excess of £100,000** if the data was not extracted in time.
- Worked to a business-critical deadline on a project the CTO flagged as a six-figure commercial exposure.

### Requires confirmation before use

- Number of staff actively using the portal (the Firestore `users` collection holds this).
- Exact production launch date.
- Confirmation of the vendor-exit deadline, the penalty figure and whether the portal is what averted it — ideally in writing.
- Any time saved versus the vendor platform's own interface.
- Whether I contributed to the backend API / extraction itself (not evidenced in this repository).

---

## 19. Portfolio Case Study Material

**Overview.** A browser-based portal that gives a 25-year, 400,000-policy insurance archive a home the business controls — built as the destination for data extracted from a third-party platform the company was leaving.

**Context & challenge.** The commercial context was an exit under deadline (§3.1). The engineering context was a prototype whose assumptions were all wrong once real data arrived: a dataset two orders of magnitude larger than any UI pattern in it could handle, a source returning syntactically invalid dates, and a document estate on a Windows file share that browsers are forbidden to touch.

**My role.** Designed and delivered the portal: the API contract, the integration layer, authentication and user administration, the prototype reduction, and PR integration — plus the architectural guidance that moved the backend off a failing Firebase Data Connect layer onto the Node/PostgreSQL REST API that shipped. I did **not** build the backend; another developer did, and I had no backend or database access by design. The initial UI was produced with an AI scaffolding tool I directed; the component library is third-party (shadcn/ui); a second developer contributed much of the policy/risk presentation.

**Solution & architecture.** React/TypeScript SPA on Firebase Hosting with two independent planes — Firebase Auth plus a named Firestore database for the user lifecycle, and a Cloud Run REST API for broking data. All lists server-paginated; all legacy values pass through one mapping module. (Diagram: §5.)

**Engineering problems, evidence and results** are in §12/§14, §9 and §11 respectively; the case study should draw on those rather than restate them. **What I would improve next:** §16.

---

## 20. Interview Story Bank

**S1 — "Debugging something undocumented."** *(reverse-engineering, iteration)* Policy events referenced archive files that filenames alone didn't locate on the share. I inferred the sharding rule from filename/location pairs: shipped an 8–10-digit prefix rule, found it wrong, tried a dynamic variant, then **reverted** to the rule that matched (first six digits → three two-digit directories, `00`-prefixed filename). Separately accepted that browsers block `https:`→`file:` and changed the interaction to a copy-the-path dialog. *Result:* staff reach any referenced document; 23–31% of events carry one. *Evidence:* commits `931719c`, `1253c53`, `2d06aa2`. *Why it lands:* willingness to revert your own work when the evidence says so.

**S2 — "A time you deleted a lot of code."** *(judgement, ownership)* My own prototype had six feature areas with no backing data. I audited each against what the data source could actually serve, removed ~19,300 lines across four staged commits, and wrote the API spec defining real scope. *Result:* integration had a bounded, deliverable scope. *Evidence:* four cleanup commits; `API_Specifications.md`. *Why it lands:* cutting your own work is harder than cutting someone else's.

**S9 — "A technical disagreement, and how you handled it."** *(influence without authority — the strongest seniority signal in this project)*
*Situation:* I owned the frontend; another developer owned the backend and the data extraction. I had deliberately been given no backend or database access, as a data-protection measure. He chose Firebase Data Connect — a generated GraphQL-style SDK over the data — and spent about a week unable to get a workable data path out of it.
*Task:* The frontend had nothing real to consume, and we were working to a deadline (§3.1). I had an opinion about why, but no authority over his component.
*Action:* Rather than escalate it as a blocker, I kept making the same technical case: what the application actually needed was ordinary paginated list endpoints over a large relational dataset, which is what REST and SQL do natively; the generated SDK was adding a translation layer, an unfamiliar query model and a build-time dependency between us in exchange for nothing we needed. I had already written the API specification, so the alternative wasn't abstract — there was a concrete contract to build against. Over a weekend, he moved to a Node.js REST API over PostgreSQL.
*Result:* The SDK came out and REST fetching went in on the same day, 2 Nov 2025. That architecture shipped and has served production for over a year. Its pagination model is what made server-side paging over 400,000 rows workable on my side.
*Evidence:* the abandoned `@irev/quote-ap` SDK dependency still in `package.json`; `listCustomers` in commit `f627567`; Data Connect emulator keys disabled in `.env.example` directly above the live API base; branch `data-sdk-integration`; commits `d4f26f5` and `750e2c8`.
*How to tell it:* the pivot is in the repo; my role in causing it is my own account. Say both — *"I can show you the switch; you'd have to take my word that I'm the one who pushed for it."* That framing is disarming and makes the rest land.

**S10 — "A time you disagreed with a code change."** *(code review, correctness over politeness)*
*Situation:* A colleague refactored my authentication and user-management code with AI assistance while fixing a reported sign-in bug.
*Problem:* Two things were wrong. The refactor memoized the Firestore account-status lookup in a `Set`, so a suspended user's status was checked once per session instead of on every auth-state change — quietly turning a security control into a cache. And it hadn't fixed the bug: the commit claiming to fix the sign-in button was a quote-style reformat, and the defect was still there eleven days later.
*Action:* Reverted the refactor to restore the unconditional check, then traced the live symptom to its actual root cause — a `<Button>` inside a `<form>` defaulting to `type="submit"` — and fixed it with one line.
*Result:* The suspension control works as designed; the user-facing bug was closed the day it was escalated.
*Evidence:* `576ccaa` (185+/228− across four files), `39b28fc`, and `9cdef3f` for the ineffective fix.
*Why it lands:* the one-line fix is the argument. It demonstrates the large refactor was never addressing the problem, without needing to say so.

**S8 — "Delivering something commercially critical."** *(pressure, prioritisation — the strongest story here)*
*Situation:* The company was exiting a third-party platform that held ~25 years of its client and policy data. The CTO told me the vendor had given us about a month to extract it, with a charge over £100,000 if we missed — and that this was why delivery speed mattered.
*Task:* Build the system that would hold and expose the extracted data, so the business could actually use and verify what came across.
*Action:* Cut scope hard before writing integration code — removed six speculative feature areas so effort went only where data existed. Wrote the API contract up front so the extraction/backend work and the frontend could proceed in parallel rather than in sequence. Built the integration defensively, because the incoming data was a 25-year legacy extract with no schema guarantees and I would not get a second pass at it.
*Result:* The portal shipped; the data has a home the business controls; the application has been in production use for over a year.
*Evidence:* `API_Specifications.md` (contract-first); the four cleanup commits (scope control); the production data profile — 779,778 clients, 401,339 policies, 1.06% Live, events from 2000 — which is the shape of a bulk historical extraction. The deadline and penalty are **as told to me by the CTO**, verbally.
*How to tell it:* attribute the figure explicitly and say the deadline governed the data extraction, with the portal as the destination. Volunteering that boundary makes the rest of the story more credible, not less.

**S3 — "Designing against a system that doesn't exist yet."** *(architecture)* Derived a full REST contract from the prototype's data model and built the client to accept both the specified envelope and alternatives. *Result:* the delivered API diverged — and nothing broke. *Why it lands:* the interesting part is that the spec was *partly wrong* and the design anticipated that.

**S4 — "Making something work at scale."** *(performance, pragmatism)* With ~400k policies and search measured at 4–5 s, separated typed query from submitted query so search fires on intent, kept every list server-paginated, and used `limit=1` probes for tab counts with no payload. *Evidence:* `Policies.tsx`, `getPolicyCountByState`.

**S5 — "A bug that was one line."** *(root cause, responsiveness)* A staff member reported the "forgot password" link triggering a sign-in attempt; root cause was a `<Button>` inside a `<form>` defaulting to `type="submit"`; fixed same day with `type="button"` (commit `39b28fc`). Usable standalone as a short root-cause story — but **S10 is the better version**, because it supplies the context that another developer had spent two commits and eleven days failing to fix the same defect.

**S6 — "An SDK behaving unexpectedly."** Creating a user for an invitation signs that user in and displaces the administrator's session; solved with a secondary Firebase app plus a reset email so the invitee sets their own password. *Evidence:* `ManageUsers.tsx`.

**S7 — "Something you'd do differently."** *(security maturity)* The data API is not token-protected — identity and data were separate planes and the data plane was left open. I would put a Firebase ID token on every request and verify it at the API edge from the first integration commit. Also: no tests around the mapping functions, which are pure, trivially testable and exactly where the risk lives. *Note:* strong **in interview**; keep the specific finding out of any public write-up.

---

## 21. Technical Article / Blog Opportunities

**A1 — "Your React data-fetching defaults break at 400,000 rows."** *Core problem:* debounce-on-keystroke search and client-side filtering against a large server dataset. *Why interesting:* standard tutorial advice is actively wrong at this cardinality. *Explain:* submit-triggered search, server pagination, `limit=1` count probes, cancellation-guarded effects. *Anonymise:* all record counts and the domain. *Never publish:* API base URL, auth posture, insurer/product codes.

**A2 — "Defensive mapping: rendering data a 25-year-old system emits incorrectly."** *Core problem:* invalid timestamps, string-typed numbers, `-1` statuses, dual-padded codes, one field under four names. *Explain:* the mapper-layer pattern, repair-then-fallback date sanitising, and why you negotiate response shape rather than trust a spec. *Anonymise:* use invented values of the same shape. *Never publish:* real record values or the production schema.

**A3 — "You cannot link to `file://` from `https://` — and that's fine."** *Explain:* the browser security rule, why every workaround fails, and designing the interaction around it. *Never publish:* the path algorithm — the sharding scheme is proprietary.

**A4 — "The Firebase invite footgun: creating a user signs you out."** *Explain:* why `createUserWithEmailAndPassword` replaces the session, the secondary-app fix, and why invite + reset-email beats generating temporary passwords. Nothing proprietary involved. **Best first-article candidate — publishable almost as-is.**

**A5 — "Scaffolding got me a UI in a day. Production took nine weeks."** *Thesis backed by this project's own numbers:* generated scaffolding produced eight feature areas immediately; production readiness required deleting ~19,300 lines of them and writing an API contract. *Explain:* a concrete process for auditing generated features against deliverable data — the judgement step the tooling cannot do. *Anonymise:* organisation and dataset. *Tone:* neither anti-AI nor promotional; the scaffold genuinely accelerated the start. *Note:* this angle requires being open that you used scaffolding — publish it only if you're comfortable with that. A4 and A1–A3 do not.

**A7 — "We spent a week on a managed data layer, then replaced it with REST and Postgres in a day."** *Core problem:* a generated GraphQL-style SDK over a large relational dataset, where the application only ever needed paginated list endpoints. *Why interesting:* thousands of teams face this decision on Firebase/Supabase-style stacks, and an honest account of *why the managed abstraction lost* is rare — most are either vendor advocacy or dismissal. *Explain:* what Data Connect optimises for, why a 400k-row paginated archive browser is not that, how a written API contract made the alternative concrete, and the generated-SDK build coupling between two developers. *Anonymise:* employer, dataset size, the week spent. *Care required:* it names a product as the wrong choice — keep it about fit for this problem, not quality. **Strongest technical article in this list.**

**A6 — "Building the landing site for a data migration you don't control."** *Core problem:* you are the destination for a bulk extract from a system you cannot inspect, on a deadline, with one realistic pass at it. *Explain:* contract-first design so extraction and consumption proceed in parallel; defensive mapping as insurance against a schema you'll only see when it arrives; cutting scope to what the extract can actually support. *Anonymise:* vendor, employer, deadline, penalty, all record counts. *Never publish:* the commercial terms or that a penalty existed — that is the employer's business, not yours.

---

## 22. LinkedIn / Social Content Opportunities

1. **"The feature I was proudest of shipping was 19,300 deleted lines."** Grounded in the four cleanup commits. Credible because it names a real trade-off: the prototype had eight features and most had no data behind them, so the work was deciding which to cut.
1. **"A generated SDK is a dependency between two developers, not just between your app and your data."** The Data Connect → REST pivot, told as an architecture lesson. *Underlying evidence:* §4.3's artefacts — you can point at the abandoned SDK still sitting in `package.json`. Credible because it's visible, not asserted.
1. **"The fix was one line. The refactor it replaced was 228 deleted ones."** On reverting rather than patching forward, and on a security check that had quietly been turned into a cache. *Underlying evidence:* commits `576ccaa` and `39b28fc`. **Keep the colleague unidentifiable** — the lesson is about the change, never the person, and a post that reads as blaming a named teammate costs you more than it gains.
1. **"The brief was: this data has to be off their system, and it has to land somewhere we can actually use."** A post on building the destination for a migration you don't control — contract-first, defensive mapping, ruthless scope control. *Underlying evidence:* §3.1 plus the production data profile. **Keep the penalty figure, the vendor and the employer out of it entirely** — the engineering lesson carries the post on its own, and the commercial detail is your employer's to disclose, not yours.
2. **"`type='button'`."** The Emma bug. A short post on form-button defaults and why root-causing a one-line bug beats patching around it.
3. **"Creating a user in Firebase logs you out. Here's why, and the fix."** Pure engineering utility, no proprietary content — the highest-engagement, lowest-risk post in this list.
4. **"Three attempts to work out where a file lived."** Short retrospective on reverse-engineering an undocumented archive layout and reverting your own work when the evidence says the earlier version was right.
5. **"I wrote the API spec before the API existed — and it was partly wrong. That was fine."** On designing clients that negotiate response shape. Credible precisely because it admits the spec diverged.
6. **"Search-as-you-type is a luxury of small datasets."** A note on cardinality changing which UX patterns are affordable, with the honest detail that the slow query measured ~4–5 seconds.

*For all of the above: no employer-identifying detail, no record counts, no mention of the API's authentication posture.*

---

## 23. Private vs Public-Safe Information

### PRIVATE — career evidence only, never publish

- Employer and product identifiers: Insurance Revolution, BG Insurance, "iRev SSP Portal", `nexus-renew`, Firebase project `bg-ssp`, the Cloud Run API base URL, Firestore database name `ssp-portal-users`.
- All exact production counts (779,778 / 401,339 / 4,249 / 397,090), premium averages, and the Live-vs-lapsed ratio — this is commercially sensitive portfolio information.
- Insurer codes, product-type codes, branch codes, scheme names, event-type taxonomy.
- The legacy document path convention and the `Z:` share layout.
- **The observation that the data API served production records to an unauthenticated request** — operationally sensitive; raise internally, never publish.
- **The entire vendor-exit commercial story: the deadline, the £100,000+ figure, the vendor's identity, and the fact that a penalty existed at all.** This is your employer's commercial information, not yours to disclose. It is excellent interview and CV material *in a private conversation with a prospective employer*; it must not appear in a public portfolio, a LinkedIn post, a blog article or a published CV. Even unnamed, "a company facing a six-figure penalty from a vendor" is identifiable to anyone who knows the sector.
- Internal staff names appearing in commit messages, **and the identity of the colleague whose refactor was reverted (§P7)**. The engineering substance of that story is yours to tell; the person is not. Describe it as "a colleague's refactor" and never name, hint at, or make them identifiable — including by naming the repository.
- **That backend/database access was withheld from you**, and the §15 observation that the restriction did not actually restrict data access. Both are internal security posture. Useful to you; not publishable, and not something to raise as a grievance in an interview either — the §16 lesson version is the professional framing.
- Internal staff names appearing in commit messages.
- Any customer record values seen during this audit (none are reproduced in these outputs).

**Handling the scaffolding question.** The repository's history shows 133 commits under a scaffolding tool's bot account. That work is yours — you directed the tool — and the dossier records it that way. But it is visible to anyone who opens the repo, so decide your answer before you need it rather than being surprised by it. The strong version is simply true: *"I prototyped the UI with a scaffolding tool, then specified the API, built the integration layer, the auth and user-admin subsystem, and cut about 19,300 lines of what the tool had produced because the data couldn't support it."* That answer is more impressive than claiming to have hand-typed a shadcn component library, and it cannot be undercut by someone scrolling your commit log. Do not volunteer the tool's brand name; do not deny the approach if asked directly.

### PUBLIC-SAFE — with the stated abstractions

- "A read-only internal portal over a large legacy insurance dataset" — domain and scale described qualitatively ("hundreds of thousands of records", "~25 years of history") rather than numerically.
- The technology stack: React, TypeScript, Vite, Tailwind, shadcn/ui, Firebase Auth/Firestore/Hosting, a REST API on Cloud Run.
- Every engineering *pattern* in §12–§16: defensive mapping, date sanitising, server-side pagination, count probes, submit-triggered search, the `https:`→`file:` boundary, the Firebase secondary-app invitation fix, cancellation-guarded effects.
- The prototype-to-production narrative, including the ~19,300-line reduction — safe in the abstract, since it describes my own repository activity rather than the business.
- "Built the destination system for a migration off a third-party platform" — **the migration itself is publishable in neutral terms; the deadline and the penalty are not.**
- The **Data Connect → REST/PostgreSQL architecture pivot**, as a technical lesson about managed data layers versus REST over SQL for large paginated datasets. Publishable with employer, dataset size and timeline removed. The product name is fine to mention if the piece is fair about fit rather than quality.
- The **revert-versus-patch-forward lesson** (a memoized security check, and a one-line root-cause fix), told entirely without reference to who wrote the refactor.
- "In production for over a year" — safe, and worth saying.
- The lessons in §16 and the article angles in §21, with the anonymisation each one specifies.

*When in doubt, a figure goes in the private column. All six numbers in §9.1 are private, and so is every element of the commercial story in §3.1.*

---

## 24. Evidence Gaps and Further Metrics Worth Collecting

| Gap | Why it matters | Where it likely exists | Question to ask internally |
|---|---|---|---|
| **Number of portal users and actual usage** | The strongest missing adoption metric; converts "built it" into "it is used" | Firestore `users` collection in database `ssp-portal-users` — every record has `invitedAt`, `invitedBy`, `lastLogin`; also the Firebase Auth user list | "Can I get a count of users in the portal's Firestore `users` collection, and how many have a `lastLogin` in the last 90 days?" |
| **Written confirmation of the vendor-exit deadline and penalty** | **Highest-value gap.** Would move the strongest claim in this dossier from REPORTED to VERIFIED, and is the one a sceptical interviewer would probe | The vendor contract or termination notice; CTO/finance email threads; any project kickoff deck or board paper from Aug–Sep 2025 | "Is there anything in writing about the data-extraction deadline and the charge we'd have faced — a contract clause, a notice, or an email I could reference?" |
| **Production launch date** | Anchors every claim to a real period | Firebase Hosting release history for `bg-ssp`; first non-developer `lastLogin` | "When did the portal go live to staff?" |
| **Confirmation that the extraction completed on time and the charge was avoided** | Closes the loop between the system and the outcome; without it you can claim the system, not the result | CTO; finance; the vendor termination record | "Did we get the data out in time, and did that avoid the charge?" |
| **Corroboration of the architecture pivot from the backend developer** | Would move the strongest seniority claim (C13) from REPORTED to VERIFIED. The repo proves the pivot happened; only he can confirm who argued for it | The backend developer directly; any Slack/Teams thread or ticket from late Oct–early Nov 2025; the backend repo's own commit history | "Do you remember the switch from Data Connect to the Node/Postgres API — would you be happy to confirm I was the one pushing for it? A short note or a LinkedIn recommendation mentioning it would mean a lot." |
| **Time previously spent retrieving this data from the legacy system** | The only credible route to a time-saving claim | Broking/ops team; any pre-existing process documentation | "Before the portal, how did staff look up a lapsed policy's event history, and roughly how long did it take?" |
| **Ongoing cost difference versus the vendor platform** | Would add a recurring-saving dimension alongside the one-off penalty avoided | IT/ops; vendor contract; the Firebase/Cloud Run bill | "What were we paying the vendor annually, versus what the portal and its API cost us to run?" |
| **Exact total event/transaction counts** | Would upgrade §9.2 from sampled to exact | A backend aggregate endpoint or direct database access (neither available here) | "Can the API expose, or can someone run, a total count of policy events and transactions?" |
| **Whether the committed secrets were rotated** | Needed before claiming the remediation was complete | Firebase console key history; team | "Were the Firebase keys rotated after the `.env` cleanup in November 2025?" |
| **Stakeholder feedback** | The only source of qualitative outcome evidence | The staff who requested features, including the reporter of the 20 Nov defect | "Did the team give any feedback on the portal once it was in their hands?" |
| **Whether the data API was later authenticated** | Determines whether §15's finding is historic or live | The backend repository / infrastructure config | "Is the SSP data API behind authentication now?" |

---

## 25. Appendix — Evidence Notes

**A. Method.** Repository inspected at `HEAD` of branch `files-linking` (commit `2d06aa2`). Attribution via `git blame --line-porcelain HEAD` per file, aggregated by author; commit-count attribution via `git shortlog -sne --all`. Both are reported because they disagree in informative ways (the bot's commit count is inflated by 133 same-day scaffold commits; its line count is inflated by untouched UI primitives).

**B. Production queries.** All production evidence came from HTTP `GET` requests to the API base in the local `.env`. No `POST`, `PUT`, `PATCH` or `DELETE` was issued; no database connection was made; no credentials were available or used. Raw samples were written to a session scratch directory and are not retained in these outputs. **No customer PII is reproduced anywhere in this dossier**, and no credential, key, URL or connection string appears in any output file.

**C. Reproducing the headline figures.**
```
GET {API_BASE}/api/clients?page=1&pageSize=1        → total              = 779778
GET {API_BASE}/api/policies?page=1&limit=1&state=0   → pagination.total   = 4249
GET {API_BASE}/api/policies?page=1&limit=1&stateNot=0→ pagination.total   = 397090
```
Sampling: 4 pages × 100 live policies (pages 1, 10, 25, 43 → n=349) and 4 pages × 100 not-live (pages 1, 1000, 2000, 3970 → n=400); sub-resource probes on 30 live policies (`random.seed(1)`) and 25 historical policies (`random.seed(2)`), reading `pagination.total` per policy.

**D. Known limitations of this audit.**
- Sub-resource statistics are sample-based; page selection was convenience-based rather than uniform random across all 3,971 not-live pages.
- Latency figures are single, cold measurements from one machine and one network location.
- The backend service, the legacy data store and the Firestore security rules were not available for inspection.
- Portal usage (as opposed to data scale) could not be measured at all.
- Line-level `git blame` credits the last author to touch a line; it under-credits work later rewritten by others and over-credits reformatting. Commit-level evidence is cited alongside it throughout.

**E. Key file references.**
`src/lib/api.ts` (integration layer) · `API_Specifications.md` (contract) · `src/context/AuthContext.tsx` (auth + status gate) · `src/pages/ManageUsers.tsx` (user admin) · `src/pages/SignIn.tsx` / `ForgetPassword.tsx` · `src/components/policies/tabs/EventsTab.tsx` (document linking) · `src/lib/riskDataMappings.ts` (legacy code translation) · `src/lib/utils.ts` (`formatDateTime`) · `src/pages/Policies.tsx` (Live/Not-Live views) · `firebase.json` / `.firebaserc` (deployment).
