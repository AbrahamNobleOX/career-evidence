# Qutii (TiiQu PDF2QA) — Career Evidence Dossier

**Audit date:** 17 October 2026 · **Revision 2** (adds first-hand context from the subject and a second evidence pass)
**Subject:** Akintola Abraham (`abramseyakid@gmail.com`, GitHub `AbrahamNobleOX`)
**Engagement type:** **Volunteer / unpaid contributor**, using his own equipment — stated by the subject and consistent with the evidence
**Engagement period:** **31 January 2024 – 29 October 2024 (9 months)** — verified from first and last commit

**Repositories audited:** `TiiQu-Network/FrontEnd-PDF2QA` (and the fork `AbrahamNobleOX/FrontEnd-PDF2QA`); `TiiQu-Network/2024-backend-pdf2qa` (branches `main`, `websocket`, `internet-tasks`, `agmodified`, `deleteuser`); a local WebSocket working copy; and an archived pre-May-2024 API snapshot (`frontend_api_layer-*.zip`). A clone of `hashicorp/learn-terraform-rds` is **unmodified upstream tutorial content** and is excluded from every claim.

> ⚠️ **Handle before sharing anything:** the archived zip contains a **hardcoded production database connection string with credentials in source** (`src/app.ts`). Do not send that zip to anyone, do not attach it to a portfolio, and consider deleting it or scrubbing the file. Its existence is itself evidence (see §11, Problem 1) but the file must not circulate.

---

## 1. Executive Summary

**Qutii** (internally "PDF2QA", package name `qutii`, domain `qutii.com`) was a document-processing SaaS built by TiiQu Network. Users authenticated, uploaded a PDF, flagged its copyright status, and an upstream pipeline extracted question-and-answer pairs from the document's paragraph structure. Users reviewed and edited those pairs in-browser, marked editing complete, and downloaded the result as a Word document. Identity ran through Auth0, payments through Stripe.

**The headline for career purposes: this was unpaid volunteer work in which one contributor ended up owning the production application.** That combination is unusual and is the most persuasive thing the evidence shows. Over nine months:

- He became the **largest contributor to the frontend** — 92 of 278 commits (33%) across 19 contributor identities — and the **sole active frontend developer from April to October 2024**, with 68 of 71 commits in that window (96%).
- He **created and owned the backend API repository outright** — first commit 22 May 2024, 49 of 61 commits (80%).
- He **rebuilt the API layer**, replacing a hybrid Sequelize/raw-SQL implementation that ran `sequelize.sync()` on every Lambda cold start and carried database credentials hardcoded in source, with a layered Fastify service using parameterised SQL and environment configuration.
- He was sole author of the **AWS deployment tooling**, **Lambda layer packaging**, the **WebSocket status service**, and the **Auth0 + PostgreSQL account-deletion cascade**.

**This was his entry point into AWS and into owning production infrastructure** — stated by the subject, and consistent with the repository, where his first cloud commits (22–25 May 2024) are also the repository's first.

**Production statistics: none exist, and none can be recovered from here.** He no longer has access to the organisation's cloud account, repository or database. Independently, the production RDS hostname returns `NXDOMAIN` and all four production API Gateway endpoints are unreachable — the environment has been torn down. Every figure in this dossier is a repository metric, labelled as such. §22 lists what might still be recoverable and from whom.

**On AI positioning — read §16 before writing any CV line.** This project contains **no AI or ML code**: no vendor SDK, no model call, no NLP library, on any branch. The extraction pipeline was a separate system owned by someone else. What he built and owned was **the production platform that delivered an AI product to users** — human-in-the-loop review of generated output, asynchronous orchestration of slow inference jobs, model-failure states, and rights governance over the document corpus. That is a real and currently sought-after role (AI *platform* engineering, not ML engineering), and it is fully evidenced. The claim "integrated AI into their system" is not, and should come from his other freelance engagement instead.

**One further correction: NestJS was not used on this project.** See §3.4. This matters because it is checkable.

---

## 2. Project Overview

| Attribute | Value | Evidence |
|---|---|---|
| Product / internal name | Qutii / PDF2QA | `package.json` `"name": "qutii"`; backend README |
| Owner | TiiQu Network | GitHub org `TiiQu-Network`; `tiiqu.com` live |
| Type | Authenticated web SaaS | Auth0 gates, Stripe pricing table and customer portal |
| Subject's engagement | Volunteer, unpaid, own equipment | Stated by subject; consistent with the fork-based workflow |
| Subject's period | 31 Jan 2024 – 29 Oct 2024 (9 months) | First and last commit, `git log` |
| Region | AWS `eu-west-2` | `.env`, GitHub Actions workflow |
| Repo first commit | 10 May 2023 (frontend, predates subject) | `git log` |
| Production status | Decommissioned | RDS NXDOMAIN; 4/4 gateways unreachable |
| Project generation | An **early version** of the product | Stated by subject; the org has likely iterated since |

**End-to-end workflow.** Auth0 login → select PDF and make a mandatory *shareable / non-shareable copyrighted* declaration → request a presigned URL (`GET /NewDoc`) and `PUT` the file **directly to S3**, bypassing the API → an upstream pipeline parses the document into `Article` → `Paragraph` → `Qna` rows and advances a `Task` status → the documents table surfaces an eight-state machine (*Processing → Processed / CreateQnA → Edit ready → PDF ready → Download ready*, plus *Processing failed* and *QnADeleted*) → at *Edit ready* the user edits pairs inline, each save issuing `PUT /QnA` and setting `Document.qnasEdited` → "complete edit" calls a separate Lambda that advances to *Download ready* → download issues a 30-second presigned S3 URL, fetched as a blob with a `.pdf` → `.docx` filename rewrite. Users can delete a single Q&A, a whole Q&A set (which also removes the S3 object), a document, or their account.

**Users:** authenticated subscribers, plus an `admin` level read from an Auth0 custom claim (`user.accessLevel[0] === "admin"`) that unlocks an analytics dashboard.

**Business problem (INFERRED).** Producing Q&A sets from long documents by hand is slow and expensive; the product automates extraction and reserves human effort for review. The mandatory copyright flag, the `isCopyrighted` column, and the separation of shareable from non-shareable documents strongly suggest an intent to accumulate a shareable corpus from open-source material. No requirements document exists in the repositories, so the commercial motive is inferred from the data model and UI copy, not proven.

---

## 3. My Role and Contribution

### 3.1 Quantified authorship

| Repository / window | Subject's commits | Total | Share | Contributors |
|---|---|---|---|---|
| FrontEnd-PDF2QA (all time) | 92 | 278 | 33% | 19 identities |
| FrontEnd-PDF2QA, Apr–Oct 2024 | 68 | 71 | 96% | 3 identities |
| 2024-backend-pdf2qa | 49 | 61 | 80% | 2 identities |

*Method: `git shortlog -sne --all` and `git log --all --author=abramseyakid`, summing two Git identities ("Akintola Abraham", "Abraham") on one email. Commit counts measure activity, not lines or difficulty.*

The handover is visible month by month. February 2024: Benneh 41, Ted 29, Abraham 18. From April 2024 the project is his — April 8/10, May 1/1, June 2/2, July 28/28, August 15/15, October 14/15. **He arrived as one of several contributors and left as the only one.**

### 3.2 Verified personal contributions

Each item is traceable to commits authored by Abraham with no competing authorship in the same files.

**Rebuilt the backend API layer (May 2024) — the strongest single piece of work.** The pre-existing API (preserved in the archived zip, dated to 19 May 2024) was a hybrid: it imported both `sequelize-typescript` **and** `@fastify/postgres` and used both in the same file; it called `sequelize.sync()` inside a floating, unawaited `Promise` at module load, so every Lambda cold start performed a schema sync; it carried the **production database connection string hardcoded in source**; and it contained explicit timing instrumentation (`const t0 = …; const t1 = …; console.log(t1 - t0)`) wrapped around the Sequelize constructor — someone was actively measuring how slow initialisation was. Abraham's repository, created three days later, **drops Sequelize entirely**, uses `@fastify/postgres` with parameterised SQL throughout, moves the connection string to `POSTGRES_CONNECTION_STRING`, and separates `server.ts`, `config/db.ts`, `routes/app.ts` and `aws/`. (*See §11, Problem 1 for the full account and its qualification.*)

**AWS deployment pipeline and Lambda layers (May 2024) — sole author.** Five coordinated npm scripts plus `uploadZipToS3.ts` and `updateLambdaFunction.ts` written against the AWS SDK: compile → zip function code → build and zip a dependency layer → upload to S3 → `UpdateFunctionCode`. Routine deploys ship kilobytes rather than tens of megabytes.

**Fastify-to-Lambda proxy with a warm-start guard.** `src/aws/lambda.ts` calls `routes()` and `app.ready()` exactly once per container behind an `isInitialized` flag.

**WebSocket status service, including a documented false start (July 2024) — sole author.** He first attempted in-process WebSockets inside the Fastify/Lambda application, adding `@fastify/websocket` and `fastify-websocket`. On **13 July 2024** he removed both dependencies ("removed a library"), and the following day moved the work into a **standalone `ws` service** that accepts a client access token, polls the documents API every 5 seconds, diffs against an in-memory map and pushes only changed documents. The pivot is the interesting part: AWS Lambda cannot hold persistent WebSocket connections, so the in-process approach could not work.

**Account-deletion Lambda (Aug 2024) — sole author.** Auth0 client-credentials token → delete in Auth0 → resolve the internal id from `authProviderId` → cascade across `Task`, `UserUpload`, `User`.

**Auth0 login and callback handling (Jul 2024) — sole author.** Replaced the default `handleAuth()` with explicit `login` and `callback` handlers, setting a post-login `returnTo: "/redirect"` destination and adding error handling with proper status propagation to both. This is the fix to the login/redirect behaviour.

**Frontend feature ownership.** Access-token integration with the backend API and the `/api/get-access-token` route (Apr 2024). Upload flow: presigned-URL request, direct-to-S3 `PUT`, copyright gate, MIME validation (Jun 2024). `MyDocumentsTable.tsx` (704 lines) — the full status machine with per-state colour/icon mapping and conditional action gating. Q&A editor with per-row `PUT` save (Jul 2024). Q&A download with presigned URL, blob handling and filename rewrite (Jul 2024). Status polling (Jul 2024) and the edit-complete stage advance (Oct 2024). Account-deletion UI. **Home-page responsiveness** ("Added responsiveness to home page", 5 Apr 2024). Earlier ticketed work: admin dashboard and access gating (`PDF-186`), custom 404, Q&A line chart (`PDF-188`), pie/doughnut chart (`PDF-189`), dashboard right panel.

### 3.3 Where authorship cannot be established

- **The PDF parsing and Q&A generation pipeline is not in these repositories — and contains no AI/ML code he wrote.** An exhaustive search of every branch, the compiled output and the archive found no AI vendor SDK, model invocation or NLP library. He built the platform *around* the extraction and never called a model. State this boundary in interviews before being asked; §16 gives the wording.
- `GET /NewDoc` and `POST /editcomplete` are called by the frontend but implemented in neither repository.
- Backend task-status, user-details and parts of the document-delete cascade were authored by `avstxn98` (12 commits, Jun–Jul 2024).
- The 2023 frontend — design system, home page, navigation, early auth scaffolding — predates him.
- **Authorship of the pre-May-2024 API in the zip is not established.** The improvement it was replaced by is definitively his; who wrote the original is not evidenced.
- No infrastructure-as-code for the AWS account, RDS, API Gateway or ECS exists in these repositories.

### 3.4 Correction: NestJS was not used on this project

The subject recalled this as the first project where he used NestJS. **A search for `@nestjs` across all four local repositories and the archived zip returns zero matches.** The backend is **Fastify** throughout — both the pre-May-2024 version and his rebuild. The ORM in the earlier version was **Sequelize** (`sequelize-typescript`), which he removed.

This is worth getting right because a hiring manager can ask about it and because the true story is better: *removing* an ORM and a half-finished abstraction in favour of a lean, explicitly-queried service is a more senior-sounding decision than adopting a framework. **Do not list NestJS under this project.** If NestJS belongs on the CV, it needs to come from a different piece of work.

---

## 4. System Architecture

```
                       ┌──────────────────────────────┐
                       │           Auth0              │
                       │  OIDC · JWT · Management API │
                       └──────┬───────────────┬───────┘
                              │ login         │ delete user
                              ▼               │
┌──────────────────────────────────────────┐  │
│  Next.js 12 web app (React 17 / TS)      │  │
│  Auth0 session · MUI DataGrid · Chart.js │  │
│  Stripe pricing table + customer portal  │  │
│  Docker → ECR → ECS (eu-west-2)          │  │
│  CI: GitHub Actions on push to `dev`     │  │
└─┬───────────┬──────────────┬─────────────┘  │
  │ JWT       │ presigned    │ ws://          │
  ▼           ▼ PUT/GET      ▼                │
┌──────────────────┐ ┌──────────┐ ┌───────────┴────────┐
│  API Gateway     │ │   S3     │ │ internet-tasks     │
│       ▼          │ │ landing/ │ │ Lambda (Fastify)   │
│ Fastify on Lambda│ │processed/│ │ DELETE /v1/user    │
│ 10 REST routes   │ │ runtime_ │ │ Auth0 + cascade    │
└────────┬─────────┘ └────┬─────┘ └────────┬───────────┘
         │ pg pool        │                │
         ▼                ▼                ▼
┌────────────────────────────────────────────────────┐
│  PostgreSQL on Amazon RDS (eu-west-2)              │
│  User · Document · Task · Article · Paragraph ·    │
│  Qna · Download · UserUpload                       │
└────────────────────────────────────────────────────┘
      ▲                             ▲
      │ writes status / Q&A         │ polls every 5s
┌─────┴───────────────────┐  ┌──────┴──────────────────┐
│ Upstream PDF→Q&A        │  │ WebSocket status service│
│ pipeline (NOT in repos) │  │ `ws` · in-memory diff   │
└─────────────────────────┘  └─────────────────────────┘
```

**Data model** (DERIVED — reconstructed from SQL in `src/routes/app.ts`; the schema itself is defined upstream, no migrations exist here):

`User` (`authProviderId` = the Auth0 `sub`, the join key between identity and relational store) · `Document` (`fileName`, `title`, `author`, `keywords`, `isCopyrighted`, `qnasEdited`, `uploadedBy`) · `Task` (`documentId`, `status`) · `Article` → `Paragraph` → `Qna` (a four-level hierarchy below the document) · `Download` (`documentId`, S3 key of the generated artefact) · `UserUpload`.

**API surface** — 10 routes on the main Lambda (473-line route module), plus one on the internet-tasks Lambda. All paths prefixed `/api/web/v1`:

| Method | Path | Purpose |
|---|---|---|
| GET | `/alldocuments` | All documents joined to task status (admin) |
| GET | `/document` | Caller's documents, scoped by JWT `sub` |
| DELETE | `/document` | Transactional cascade: Qna → Paragraph → Article → Task → Document |
| GET | `/document/download` | Presigned URL for the original upload |
| GET | `/document/task` | Single-document status (polling endpoint) |
| GET | `/QnA` | Q&A pairs with paragraph context |
| GET | `/QnA/download` | 30-second presigned URL for the generated file |
| PUT | `/QnA` | Batch update; sets `qnasEdited` |
| DELETE | `/QnA` | Delete all Q&As + S3 object + status `QnADeleted` |
| DELETE | `/QnA/single` | Delete one pair |
| DELETE | `/user` | *(internet-tasks Lambda)* Auth0 delete + relational cascade |

---

## 5. Technology Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 12, React 17, TypeScript 5, Tailwind 3, MUI 5 + `x-data-grid` 6, Material-UI 4, Reactstrap, Chart.js 4, `pdfjs-dist` 3, `@react-pdf/renderer` 3 — 18 pages, 21 components, ~5,300 lines TS/TSX |
| Backend | Fastify 4, TypeScript, `@fastify/postgres`, `@fastify/aws-lambda` 4, `ws` 8, `aws-sdk` v2, Auth0 Node SDK 4 |
| Removed during the rebuild | `sequelize`, `sequelize-typescript`, `@fastify/websocket`, `fastify-websocket` |
| Data & storage | PostgreSQL on Amazon RDS; S3 (`landing/`, `processed/`, `runtime_dependencies/`) |
| Compute | AWS Lambda + API Gateway (backend, 4 gateway stages referenced); ECR + ECS (frontend) |
| Identity & payments | Auth0 (OIDC, Management API, custom claims); Stripe (pricing table, customer portal) |
| CI/CD | GitHub Actions → ECR → ECS; bespoke npm + AWS SDK scripts for Lambda |
| Containers | Multi-stage Dockerfile, `node:18-alpine`, non-root user, cache and bind mounts |
| Process | Jira-style `PDF-###` ticket branches (10+ on origin) |
| **Not used** | **NestJS** — see §3.4 |

---

## 6. Engineering Implementation Highlights

**Fastify inside Lambda, done carefully.** `src/aws/lambda.ts` guards initialisation behind an `isInitialized` flag. Without it, every warm invocation re-registers routes and the Postgres plugin — and Fastify throws on duplicate registration. A few lines encoding real understanding of the Lambda execution model.

**Dependencies in a layer, not the function.** `package-layer` builds a separate `nodejs/` tree, installs production dependencies and zips it as a Lambda layer; `package-build` zips only compiled `dist/`. `deploy` chains build → S3 upload → `UpdateFunctionCode`.

**Direct-to-S3 transfer both ways.** No file bytes pass through the API: uploads are a presigned `PUT` to `landing/`, downloads a presigned `GET` from `processed/` with a 30-second expiry. Lambda payload limits and cold-start cost stay out of the file path entirely.

**Transactional cascade deletes.** Document deletion wraps `BEGIN`/`COMMIT` around five ordered `DELETE`s descending `Qna → Paragraph → Article → Task → Document`, using correlated subqueries so the whole subtree resolves from one `documentId`, with `ROLLBACK` in the catch.

**Defensive UI state.** Delete is enabled only in terminal or failed states, edit only at *Edit ready*, download only at *Download ready*. A code comment in the delete handler says why: deletion during processing "could break the backend code."

---

## 7. Production Evidence and Statistical Analysis

**This section reports a null result and its causes — both of which are legitimate and worth stating plainly in an interview.**

**Cause one: access ended with the engagement.** The subject contributed as an unpaid volunteer on his own laptop and no longer has access to the organisation's cloud account, GitHub organisation or database. The local code and commit history survive precisely because the work was done on personal equipment.

**Cause two: the environment no longer exists.**

| Check | Method | Result |
|---|---|---|
| Production PostgreSQL | DNS resolution, then `psql` with `default_transaction_read_only` | **NXDOMAIN** against both the system resolver and `8.8.8.8` — no connection possible |
| 4 × production API Gateway | `curl` HTTPS GET, 12s timeout | No response (`000`) on all four |
| `qutii.com` | `curl` | HTTP 200, no readable content extracted |
| `tiiqu.com` | `curl` | HTTP 200 — corporate site, not the product |

**VERIFIED conclusion:** the production environment has been torn down. The RDS instance was *deleted*, not firewalled — a running-but-restricted instance would still resolve in DNS and fail at the TCP or authentication layer. So even with credentials, no data was reachable.

**Consequence:** no production volume, adoption, reliability, latency or business metric can be established. Every quantitative figure in this dossier is a repository metric.

**Two near-misses that must never be cited as production data:**

- `src/sample_response_qna.json` holds 638 `{Paragraph, Question, Answer}` objects under the message *"PDF successfully uploaded, saved, and QNA pairs extracted!"* — apparently a captured real response. **It is a development fixture committed to source control.** It may be described only as "a sample response in the repository contains 638 extracted pairs from one document," never as production volume or a typical document.
- `sample.json` / `sample2.json` are mock data powering the admin dashboard.

**Honest architectural finding.** `pages/dashboard/index.tsx` imports its chart data from those static files; a `useEffect` contains the comment `// Fetch data from API here` and then sets state from the local import. **The analytics dashboard was built against mock data and never connected to a live source** in the audited code. The charting, access gating and layout are real engineering; the numbers were not.

---

## 8. Business Impact

With no production data, impact must be framed as *what the system was built to do*, not what it measurably achieved.

**STRONGLY SUPPORTED by the code:** human-in-the-loop automation with `qnasEdited` tracking which documents received human attention; a fully self-service lifecycle (upload, monitor, edit, download, delete document, delete account) that removes staff from every step; a GDPR right-to-erasure path built deliberately across three systems; copyright governance embedded as a mandatory blocking control rather than enforced by policy; and operational visibility that turns an opaque asynchronous pipeline into an eight-state machine a user can reason about.

**Delivery impact on the organisation (STRONGLY SUPPORTED):** from April 2024 the customer-facing application had exactly one active developer. Whatever shipped in those seven months shipped because he shipped it. For an early-stage product with a rotating volunteer team, continuity of that kind is itself the impact.

**UNKNOWN:** users, documents processed, revenue, conversion, failure rates, processing times, infrastructure cost, time saved per document, and whether the product reached commercial traction.

---

## 9. Significant Engineering Challenges

1. **Asynchronous state in a synchronous UI.** The pipeline is out-of-band with no event feed. Each answer trades differently: load-time fetch is free but stale within seconds; per-document 30-second polling is correct but scales as *documents × users* against a per-invocation-billed Lambda; a WebSocket delta service collapses that to one upstream poll per connection but introduces a stateful long-lived process that does not fit Lambda — which is exactly why it became a separate service after the in-process attempt failed.

2. **Replacing a running API without a specification.** The pre-existing implementation mixed an ORM with raw SQL, synced the schema on every cold start and hardcoded credentials. Rebuilding it meant inferring the contract from the code and the consuming frontend, because no schema definition, migrations or API documentation existed in the repository — and the database was shared with an upstream pipeline he did not own.

3. **Deleting a user across three systems with no distributed transaction.** Auth0 holds identity, PostgreSQL the record, S3 the files. The chosen order — Auth0 first, then the local cascade — fails safe in the common case. It is not atomic; a crash after the Auth0 delete orphans local rows.

4. **Fastify's lifecycle versus Lambda's.** Fastify expects one construction; Lambda reuses containers. The failure is invisible until duplicate-route errors appear on production warm starts.

5. **Lambda package size.** Fastify + `aws-sdk` + `pg` + `auth0` is large enough that bundling dependencies with code makes every deploy slow and risks the unzipped limit. Splitting into a layer meant restructuring the build into five coordinated scripts and writing SDK deployment code by hand.

6. **Leaf-first deletion across a four-level foreign-key hierarchy**, where each level is reachable only through correlated subqueries from one `documentId`.

7. **Inheriting a multi-author codebase.** A rotating team left three overlapping UI libraries (MUI 5, Material-UI 4, Reactstrap) and a duplicated component file. Taking sole ownership in April 2024 and shipping through October is a delivery achievement independent of any feature — and it was unpaid.

---

## 10. Architectural Decisions and Trade-offs

| Decision | Reasoning | Trade-offs | Result |
|---|---|---|---|
| **Drop the ORM; raw parameterised SQL on `@fastify/postgres`** | Sequelize's model registration and `sync()` sat on the Lambda cold path; the codebase was already using both approaches | Loses migrations, model validation and type-safe query building; every query is hand-written and hand-reviewed | A lean service with no ORM boot cost and no hardcoded credentials |
| **Fastify on Lambda** rather than a container | Low, bursty traffic; pay-per-invocation; the team already ran ECS for the frontend and chose per workload | Cold starts on an RDS-backed API; no cross-invocation pooling, so many containers can exhaust RDS connections | Worked, with complexity pushed into deployment. *Reasoning: INFERRED* |
| **Standalone `ws` service** after in-process WebSockets failed | Lambda cannot hold persistent connections; `@fastify/websocket` was added then removed on 13 Jul 2024 | Single point of failure; process-local status map blocks horizontal scaling; still 5s latency and constant upstream load | Built. **`NEXT_PUBLIC_WS_URL` still points at `ws://localhost:3365` — no evidence it reached production** |
| **Account deletion in its own Lambda** (`internet-tasks`) | A VPC-attached Lambda has no internet route to the Auth0 Management API without a NAT gateway | A second deployable, gateway and base URL; the route still queries RDS, so the networking story isn't clean-cut | Shipped Aug 2024. *Reasoning: INFERRED — not documented* |
| **Presigned S3 URLs** for all transfer | Keeps the API stateless and small | The API never validates file contents; a presigned `PUT` is a trusted write; the upload presign sets no explicit expiry | Implemented throughout |
| **Lambda layers** for dependencies | Small, fast code deploys; independent dependency versioning | Layer and code can drift if `package.json` changes without a layer rebuild | Deploys reduced to kilobytes |
| **Frontend to ECS**, not Vercel/Amplify | One AWS account, one IAM model | A container build and an always-on service for an app that could have run serverless | GitHub Actions on push to `dev` |

---

## 11. Problems Solved

**Problem 1 — An API that took seconds to answer, and why.** *(The best story this project offers.)*

- *Situation.* The API served the frontend from AWS Lambda, but it was slow enough to be a recurring topic in team discussions. The cloud architect on the project was an AWS solutions specialist rather than a hands-on Node developer, so the problem sat unowned between disciplines. Abraham volunteered to investigate because Node.js was his area. *(Organisational context is the subject's account; the technical findings below are all verifiable from the archived code.)*

- *Root cause — the old implementation rebuilt its database connections on almost every request, across the public internet.* The pool was configured `min: 0` with `idle: 10000`: nothing held warm, and idle connections evicted after ten seconds. Because the service was being exercised interactively — run locally, called through Postman, against the same cloud database in eu-west-2 — nearly every request fell outside that ten-second window and paid a full TCP + TLS + Postgres authentication handshake before executing any SQL. And it paid it **twice**, because the file constructed *two* independent connection pools to the same database: a Sequelize pool in `db.ts` and a registered `@fastify/postgres` pool in `app.ts`.

- *Five compounding defects on top of that:*

  | # | Defect | Effect |
  |---|---|---|
  | 1 | Two connection pools to one database (`sequelize-typescript` *and* `@fastify/postgres`), used interchangeably in the same handlers | Double connection overhead; no single owner of connection lifecycle |
  | 2 | `sequelize.sync()` called inside a floating, unawaited `Promise` at module load | DDL round trips per registered model at startup, racing with early requests for the same connections |
  | 3 | `/alldocuments` called `app.pg.connect()` **without `await`** and never released the client | Handler broken outright (`.query()` invoked on an unresolved Promise) *and* a pool slot consumed per call, never returned |
  | 4 | Five handlers called `client.release()` **twice** on the same client | Throws — releasing an already-released `pg` client is an error |
  | 5 | `acquire: 30000` against `max: 10` | Converted any pool contention into a **thirty-second stall** rather than a fast failure |

- *Evidence that the slowness was already known and unsolved.* The old `db.ts` wraps the Sequelize constructor in `const t0 = …; const t1 = …; console.log(t1 - t0)` and carries a long commented-out block on Sequelize pooling inside Lambda. Someone had been measuring initialisation cost and reading up on the problem without resolving it. The file is also littered with commented-out `sequelize.close()` calls and dead model files — the signature of a team fighting connection lifecycle issues by trial and error.

- *The fix.* Rather than patch around the symptoms, Abraham rebuilt the service as a new repository on 22 May 2024:
  1. **Removed Sequelize entirely** — one data-access path instead of two, one pool instead of two, no ORM module load, no schema sync at startup.
  2. **Replaced every manual `connect()` / `release()` with the pool-managed `app.pg.query()` form** — 32 call sites, zero `connect()` calls. The pool now acquires *and* returns each connection automatically, which made both the leak (defect 3) and the double-release (defect 4) **structurally impossible rather than merely fixed**.
  3. **Standardised on parameterised SQL** throughout, removing the mixed ORM/raw-query idiom.
  4. **Moved the hardcoded production connection string** into `POSTGRES_CONNECTION_STRING`.
  5. **Separated the codebase** into `server.ts`, `config/db.ts`, `routes/app.ts` and `aws/`.
  6. **Added layer-based packaging** (23–25 May) so the smaller dependency set also shortened deploys.

  The distinction in step 2 is the one worth drawing out in an interview: he did not fix the leaking handler, he removed the category of mistake by eliminating manual connection lifecycle management from the codebase.

- *Result.* One pool, connections correctly reused and returned, no ORM boot, no schema sync, no credentials in source, and a response path that performs one parameterised query over a reused connection. Every defect above is absent from the rebuilt service by construction.

- *Outcome — what each level of evidence supports.*
  - **From the code alone (STRONGLY SUPPORTED, claim C3a):** response time improved from seconds to milliseconds. Six defects each plausibly costing hundreds of milliseconds to seconds under the method used, all removed. This needs no one's memory.
  - **From the subject's recollection (claim C3b):** specifically from ~11 s to ~110 ms average. No benchmark or written record survives — the organisation later removed his access, including the Microsoft Teams history where the slowness was discussed. Both figures were observed the same way (locally, via Postman, against the same remote database, before and after), so the comparison is consistent in method; it simply was not recorded.

- *Qualification.* **The rebuild is definitively his; authorship of the prior implementation is not attributed to anyone**, and the dossier makes no claim about who wrote it. Describe the code, not the author. The mechanisms are verified; the precise figures are recollection — §15.2 gives three phrasings by confidence level.

**Problem 2 — WebSockets that could not work where they were put.**
*Situation:* document status needed pushing to clients. *Problem:* the natural place was the existing Fastify app, but it runs on Lambda, which cannot hold persistent connections. *Investigation:* he added `@fastify/websocket` and `fastify-websocket`, then removed both on 13 July 2024. *Solution:* a standalone `ws` service with a server-side status map, polling upstream every 5 seconds and pushing only diffs. *Result:* a working design, though deployment is unevidenced. *Evidence:* commit `acd6a13` "removed a library" (both packages dropped); `websocket` branch; `websocket/src/server.ts`.

**Problem 3 — Status never changed until refresh.**
Interval polling scoped to non-terminal documents only, then the WebSocket service. *Evidence:* `MyDocumentsTable.tsx:203–255`; commits 20 Jul 2024.

**Problem 4 — Login redirected to the wrong place and swallowed errors.**
Replaced the default `handleAuth()` with explicit `login` and `callback` handlers, setting `returnTo: "/redirect"` and propagating `error.status`/`error.message` on both paths. *Evidence:* commit `14481a0`, 27 Jul 2024, `src/pages/api/auth/[...auth0].ts`. **Sole author.**

**Problem 5 — Documents stuck blank; edit→download never fired.**
Added the `completeEdit` call to the dedicated Lambda, handled blank status explicitly, corrected colour mapping. *Evidence:* commits 1–7 Oct 2024.

**Problem 6 — Document deletion left orphaned rows and S3 objects.**
One transaction deleting leaf-first with `ROLLBACK`; Q&A delete also removes the S3 object and updates task status. *Qualification:* S3 delete is outside the transaction; attribution shared with `avstxn98`.

**Problem 7 — No way to delete an account.**
Dedicated Lambda: Auth0 Management API delete then relational cascade, with a confirmation-gated profile UI. *Evidence:* commits 21 Aug 2024. **Sole author.**

**Problem 8 — Home page did not work on small screens.**
*Evidence:* "Added responsiveness to home page", 5 Apr 2024.

---

## 12. Security, Reliability and Operations

**Implemented:** Auth0 OIDC with server-side sessions; `withPageAuthRequired` on every protected page; access tokens fetched through a server route, never in client bundles; row-level scoping by JWT `sub` on `GET /document`; **parameterised SQL on every query in the rebuilt API — no string interpolation into SQL anywhere**; **credentials moved out of source into environment variables during the rebuild**; 30-second presigned download URLs; transactions with rollback on destructive paths; non-root `USER node` in the production image; secrets in `.env`/Actions secrets, never committed.

**Weaknesses found — state these yourself before anyone else does:**

- **IDOR on `DELETE /document`.** The ownership check is written but **commented out**, so any authenticated caller who knows a `documentID` can delete another user's document. `DELETE /QnA`, `DELETE /QnA/single` and `GET /document/download` take a raw `documentID` with no ownership scoping either. Real vulnerability, still open in the audited code.
- **The archived zip contains a hardcoded production connection string.** Do not circulate that file. (The rebuild removed it — that is the point of Problem 1 — but the artefact still exists on disk.)
- `QnA/download`'s error handler `console.log`s and returns nothing, leaving the client hanging instead of receiving a 5xx.
- **No automated tests** in either repository.
- The Node CI workflow runs `npm build` — **not a valid npm command** — so the gate cannot ever have been meaningful. It also pins Node 14 while the Dockerfile builds on Node 18.
- No observability beyond `console.log`.
- The WebSocket service's process-local status map blocks horizontal scaling.
- No retry logic on Auth0 token acquisition, S3 operations or database calls.

---

## 13. Lessons Learned

- **Where code runs constrains what you can build in it.** The WebSocket attempt failed because Lambda cannot hold connections — a fact better learned before adding the dependency than after. The instinct to reach for the framework plugin was right; checking the runtime's constraints first would have saved a cycle.
- **Deleting code is engineering.** The strongest backend contribution was removing an ORM, a schema sync and a hardcoded credential, not adding features.
- **Polling is a legitimate first answer and a bad last one.** The step not taken — having the pipeline publish events — would have removed polling entirely.
- **Commenting out an authorisation check is how security holes ship.** The ownership filter was written, disabled to unblock something, and never restored.
- **A CI pipeline that cannot fail is worse than none**, because it looks like a safety net.
- **Build dashboards against the real API from day one.** This one looks finished and shows fabricated numbers.
- **Measure before and after, and write the numbers down.** The predecessor instrumented the slow path but recorded nothing durable; as a result a genuine performance improvement cannot be quantified today. Thirty seconds of note-taking would have produced a CV-grade metric.
- **A decommissioned product erases your evidence.** Take a metrics snapshot while the system is alive — and before an engagement ends, while you still have access.

---

## 14. Verified Career Claim Bank

> **VERIFIED** directly supported · **DERIVED** calculated from verified evidence · **STRONGLY SUPPORTED** · **INFERRED** · **UNKNOWN**

| # | Claim | Level | Evidence & qualification |
|---|---|---|---|
| C1 | Created and was principal author of the backend API service for a document-processing SaaS, writing 49 of 61 commits (80%). | VERIFIED / DERIVED | `git shortlog`, May–Aug 2024. *Commit counts measure activity, not difficulty; one other engineer contributed 12.* |
| C2 | Sole active frontend developer on a production Next.js app for seven consecutive months — 68 of 71 commits (96%) — having been the largest of 19 contributors overall at 33%. | VERIFIED / DERIVED | Per-month author breakdown, Apr–Oct 2024. *"Sole active" means no one else committed meaningfully, not that no one else was assigned.* |
| C3 | Rebuilt a Lambda-hosted API layer: removed an ORM and a per-cold-start schema sync from the cold path, eliminated a connection-pool leak that could stall requests for up to 30s, externalised hardcoded production credentials, and standardised on parameterised SQL in a layered service. | VERIFIED (mechanisms) | Archived `src/app.ts`/`db.ts` (un-awaited, never-released `pg.connect()`; `sequelize.sync()` at module load; `acquire: 30000`; credentials in source) vs. `2024-backend-pdf2qa` (32 × pool-managed `app.pg.query()`, zero `connect()` calls). *Authorship of the prior version is unattributed.* |
| C3a | The rebuild reduced API response time from seconds to milliseconds. | **STRONGLY SUPPORTED** | Six independently verifiable defects in the old code (two pools to one database; `min: 0`/`idle: 10000` so no warm connections against remote RDS; `acquire: 30000`; an un-awaited, never-released, broken handler; double `client.release()` in five handlers; unawaited `sequelize.sync()` at startup) each plausibly cost hundreds of ms to seconds under the testing method used. The rewrite removes all six. *Direction and order of magnitude follow from the code itself, independent of memory.* |
| C3b | Specifically, from an average of ~11 seconds to an average of ~110 milliseconds. | **UNVERIFIED RECOLLECTION** (methodologically sound) | Subject's recollection; no benchmark or written record survives — the organisation removed his access, including to the Microsoft Teams history where it was discussed. Both figures were observed the same way (API running locally, called via Postman, against the same remote database, before and after), so the comparison is like-for-like in method. The earlier cold-start/warm-invocation objection does **not** apply. |
| C4 | Designed and implemented a 10-endpoint REST API in Fastify/TypeScript on AWS Lambda behind API Gateway against PostgreSQL on RDS, including a Lambda proxy with a warm-start initialisation guard. | VERIFIED | `src/routes/app.ts` (473 lines, 10 handlers), `aws/lambda.ts`. *Some endpoint bodies came from a second engineer.* |
| C5 | Built a bespoke AWS deployment pipeline in TypeScript that packages code and dependencies separately, publishes a Lambda layer, uploads to S3 and updates the live function via the SDK. | VERIFIED | `package.json` scripts, `aws/*.ts`, commits 23–25 May 2024. **Sole author.** |
| C6 | Diagnosed that AWS Lambda cannot host persistent WebSocket connections, removed the in-process WebSocket dependencies, and delivered a standalone status service with server-side state and delta push. | VERIFIED (code) | Commit `acd6a13` removing `@fastify/websocket` and `fastify-websocket`, 13 Jul 2024; `websocket/src/server.ts`, 14 Jul 2024. **No evidence of production deployment — describe as built, not running.** |
| C7 | Implemented a GDPR-style account-deletion capability spanning Auth0 Management API, PostgreSQL and S3, ordered to fail safe if identity deletion is rejected. | VERIFIED | `internet-tasks` branch; `profile/index.tsx`; commits 21 Aug 2024. *Not atomic — a crash mid-cascade orphans local rows.* |
| C8 | Customised Auth0 login and callback handling to fix post-login redirect routing and add error propagation on both paths. | VERIFIED | Commit `14481a0`, 27 Jul 2024, `src/pages/api/auth/[...auth0].ts`. **Sole author.** |
| C9 | Implemented transactional cascade deletion across a four-level relational hierarchy plus associated S3 objects and task state, with rollback on failure. | VERIFIED | `src/routes/app.ts`. *Shared attribution; the S3 delete sits outside the transaction.* |
| C10 | Built the document lifecycle UI for an eight-state asynchronous pipeline — per-state colour/icon mapping and conditional gating of destructive actions — in a 704-line MUI DataGrid component. | VERIFIED | `MyDocumentsTable.tsx`. *Line count indicates size, not quality.* |
| C11 | Implemented direct browser-to-S3 transfer in both directions with presigned URLs, keeping document bytes out of the API tier, with 30-second download expiry. | VERIFIED | `upload/index.tsx`, `ApiFunctions.ts`, backend presign handlers. |
| C12 | Integrated Auth0 authentication, role-based admin access via custom claims, and Stripe subscription management into a Next.js application. | VERIFIED | `api/auth/*`, `userContext.tsx`, `services/stripe/*`. *Auth0 scaffolding predates him; the token integration and handler customisation are his.* |
| C13 | Delivered responsive layout work on the public landing page. | VERIFIED | Commit "Added responsiveness to home page", 5 Apr 2024. |
| C14 | Delivered feature work under a Jira-style ticketing process across 10+ tracked tickets, merged through pull requests, over a 9-month engagement. | VERIFIED | Branch names (`PDF-145`, `-163`, `-164`, `-186`, `-188`, `-189`, `-193`, `-57`, `-78`); first commit 31 Jan 2024, last 29 Oct 2024. |
| C15 | Containerised a Next.js app with a multi-stage non-root Dockerfile and a GitHub Actions pipeline deploying to Amazon ECR and ECS. | STRONGLY SUPPORTED | `Dockerfile`, `AWS_ECS.yml`, `.aws/task-definition.json`. **Authorship not established** — claim familiarity and operation, not authorship. |
| C16 | Production usage, scale, reliability and business outcomes. | **UNKNOWN** | Environment decommissioned; access ended with the engagement. **Do not cite any volume, user, revenue or performance figure.** |
| C17 | Use of NestJS on this project. | **FALSE — do not claim** | Zero `@nestjs` matches across all repositories and the archive. The stack is Fastify; the ORM removed was Sequelize. |
| C18 | Built and owned the production platform delivering an AI-powered document-processing product — asynchronous inference-job orchestration, model-failure handling, human-in-the-loop review of generated output, and rights governance over the source corpus. | VERIFIED | Status machine in `MyDocumentsTable.tsx`; `PUT /QnA` and `completeEdit` review gate; `Document.qnasEdited`; `isCopyrighted` upload gate; `websocket/src/server.ts`. *This is an AI **platform** claim, not an AI **integration** claim — see §16.* |
| C19 | Integration of AI/ML, use of LLMs, prompt engineering, RAG, embeddings, or development of an NLP pipeline **on this project**. | **FALSE — do not claim** | Exhaustive search across every branch of both repositories, the compiled `dist/`, and the archived snapshot returned zero AI vendor SDKs, model invocations or ML libraries. The extraction pipeline was owned by others and was never called directly from this code. Source any AI-integration claim from the other freelance engagement instead. |

---

## 15. CV Achievement Candidates

### 15.1 How to represent a volunteer engagement honestly

You mentioned you have sometimes presented this as a non-volunteer role. My advice is to stop doing that, and it is practical advice rather than a lecture: employment status is one of the few CV facts that gets verified. A reference check, a background screen, or a casual "so who were you reporting to at TiiQu?" surfaces it, and when it surfaces it retroactively taints the technical claims — which are strong and true. You would be risking genuinely good evidence to dress up the one detail that impresses people least.

The good news is that the honest framing is barely weaker, and in one respect it is stronger: **you did nine months of production engineering, unpaid, and ended up as the only developer on the application.** That reads as initiative and capability. Paid engineers are expected to ship; a volunteer who ends up owning the system is doing it on merit.

Practical options, in rough order of preference:

1. **Experience section, labelled.** `Software Engineer (Volunteer) · TiiQu Network · Jan 2024 – Oct 2024`. One word, and the bullets carry the weight. This is the normal convention and nobody blinks at it.
2. **A "Selected Projects" section** with dates and scope, rather than the Experience section. Appropriate if you want the Experience section reserved for paid UK roles. No status claim is implied, so nothing is misleading.
3. **"Open-source / community contributor"** if the framing fits how TiiQu describes itself.

Avoid: omitting the dates to blur it, inventing a title, or listing it among paid roles with nothing to distinguish it. In interviews, lead with it rather than waiting to be asked — "I did this unpaid because I wanted production AWS experience, and within a few months I was the only developer on it" is a confident line and completely true.

### 15.2 High-confidence quantified bullets

**The API rebuild — pick one of three, by how much scrutiny you can take.**

*Tier 1, strongest — and now defensible.* You have an answer to "how did you measure it?": *"I ran the API locally and called it through Postman against the same database, before and after the changes."* That is a consistent comparison, not a production APM benchmark, and saying so plainly is fine. Add "roughly" or "about" to the numbers and you are on solid ground.
> Rebuilt a serverless API layer on AWS Lambda, removing an ORM schema-sync from the cold-start path and eliminating a connection-pool leak that stalled requests for up to 30 seconds — reducing average response time from ~11s to ~110ms.

*Tier 2, safest option that keeps the magnitude.* Use if you would rather not defend specific figures at all. Fully supported by the code alone (claim C3a).
> Rebuilt a serverless API layer on AWS Lambda, removing an ORM schema-sync from the cold-start path and eliminating a connection-pool leak with a 30-second acquire timeout — cutting average response time from double-digit seconds to sub-second.

*Tier 3, mechanism only, zero risk.* Every word is verifiable from the code.
> Rebuilt a serverless API layer serving a document-processing SaaS — removing an ORM and a per-cold-start schema sync from the cold path, eliminating a connection-pool leak, and externalising hardcoded production credentials — then delivered 10 REST endpoints in Fastify/TypeScript on AWS Lambda, API Gateway, PostgreSQL and S3, authoring 80% of the repository's commits (49 of 61).

**Still never write:** anything implying instrumented production measurement — "benchmarked", "p99 latency reduced to 110ms", "APM showed". You observed this locally through Postman, which is honest and sufficient; claiming production telemetry you never had is the one thing that would turn a true story into a false one.

- Served as sole active frontend developer on a production Next.js/React/TypeScript application for seven months, contributing 96% of commits in that period (68 of 71), after being the largest of 19 contributors overall.
- Built a custom AWS deployment pipeline in TypeScript that packaged function code and dependencies into separate archives, published a Lambda layer and updated live functions via the AWS SDK — cutting routine deploy payloads from tens of megabytes to kilobytes.
- Designed the document lifecycle UI for an eight-state asynchronous pipeline in a 704-line MUI DataGrid component, gating destructive actions by state to prevent users corrupting in-flight processing.
- Delivered full-stack and cloud engineering across 10+ tracked tickets over a 9-month engagement on a team of 19 contributors.

### 15.3 High-confidence non-quantified bullets

- Diagnosed that AWS Lambda cannot host persistent WebSocket connections, removed the in-process dependencies, and delivered a standalone status service maintaining server-side document state and pushing only changed records.
- Implemented a GDPR-style account-deletion flow spanning Auth0's Management API, PostgreSQL and S3, ordered to fail safe when identity deletion is rejected.
- Engineered transactional cascade deletion across a four-level relational hierarchy plus associated object storage, with rollback on failure.
- Eliminated file transfer through the API tier using presigned direct-to-S3 upload and download with short-lived download URLs.
- Resolved Fastify/Lambda lifecycle conflicts with a warm-start initialisation guard, preventing duplicate route registration across reused containers.
- Customised Auth0 login and callback handling to correct post-login redirect routing and propagate authentication errors.
- Integrated Auth0 authentication with role-based admin access via custom claims and Stripe subscription management into a Next.js application.
- Took sole ownership of a frontend codebase previously built by a rotating team of 19, shipping continuously for seven months.

### 15.4 AI-related bullets

Do not write these from memory — **§16 contains the full set of defensible AI phrasings and the ones to avoid.** In short: claim the *platform* (human-in-the-loop review, async inference orchestration, model-failure states, corpus rights governance), never the *integration*.

### 15.5 Do not use without confirmation

Any figure for users, documents processed, revenue, uptime, latency or cost (no production evidence exists) · **any percentage or multiple for the API performance improvement** (the improvement is real; the measurement was never recorded) · authoring the Docker/ECR/ECS pipeline (authorship unconfirmed) · designing the PostgreSQL schema (defined upstream) · any claim about the PDF parsing or Q&A generation algorithms (not in these repositories) · **NestJS**.

---

## 16. Positioning This Project in the AI Era

This section exists because the project is intended to support a claim about AI experience, and the evidence places a firm boundary on which claim is available.

### 16.1 The finding

**No AI or machine-learning code exists in any repository you hold.** An exhaustive search — every branch of both repositories, the compiled `dist/` output, and the archived pre-May-2024 API snapshot — for AI vendor SDKs (OpenAI, Anthropic, Cohere, Gemini, Bedrock, SageMaker, Textract, Comprehend), orchestration frameworks (LangChain, Hugging Face), NLP and ML libraries (spaCy, NLTK, transformers, TensorFlow, PyTorch, scikit-learn), and model-invocation patterns returned **zero genuine matches**. Every apparent hit was either prose inside the sample Q&A fixture or MUI's `paginationModel` / `GridRowModesModel`.

The extraction pipeline — the component that reads a PDF and generates question-and-answer pairs — ran as a separate system owned by someone else. Your application handed a document to S3 and the pipeline collected it asynchronously; there is no direct call from your code into it.

**So the claim "I integrated AI into their system" is not supported by this project.** It is checkable: any interviewer who asks "which model did you use, and how did you call it?" will reach the boundary in one question, and the honest answer is that you did not call one. Making that claim risks the many strong, true claims in §14 for the one that is not.

### 16.2 What you can legitimately claim, and why it is genuinely valuable

You built and owned **the production platform that delivered an AI product to its users**. In the current market that is a distinct and sought-after role — usually called *AI product engineering* or *AI platform engineering*, as opposed to *ML engineering*. Most of the engineering effort in shipping an AI product is not the model; it is everything around the model, and that is precisely what you own here.

Specifically, and all of it verified:

| What AI products need | What you built | Evidence |
|---|---|---|
| **Human-in-the-loop review** — model output is rarely shippable raw | The complete review workflow: inline Q&A editing, per-row persistence, a `qnasEdited` flag recording which documents received human correction, and an explicit "edit complete" gate before output is released for download | `QAResultsTable.tsx`, `PUT /QnA`, `completeEdit`; commits Jul and Oct 2024 |
| **Async orchestration of slow inference jobs** | An eight-state lifecycle, status polling scoped to non-terminal jobs, and a WebSocket delta-push service — three successive designs | `MyDocumentsTable.tsx:203–255`; `websocket/src/server.ts` |
| **Graceful handling of model failure** | A first-class `Processing failed` state, surfaced distinctly from "still processing", with destructive actions re-enabled so users can retry | Status machine in `MyDocumentsTable.tsx` |
| **Handling high-volume, variable model output** | The captured response shows **638 Q&A pairs generated from 279 distinct source paragraphs (2.3 pairs per paragraph) for a single document** — the UI had to paginate, edit and batch-persist that volume | `sample_response_qna.json` |
| **Working with pipeline data structures** | Queried, joined and cascade-deleted across the pipeline's four-level chunking hierarchy — document → article → paragraph → Q&A — which is the document-segmentation structure an NLP pipeline produces | `src/routes/app.ts` joins and delete cascade |
| **Rights governance over an AI corpus** | A mandatory, blocking copyright declaration at upload, persisted as `isCopyrighted`, separating a shareable corpus from non-shareable copyrighted material | `upload/index.tsx`; `Document.isCopyrighted` |
| **Cost-aware serverless delivery of an AI workload** | Presigned direct-to-S3 transfer keeping document payloads out of the compute tier; Lambda layer packaging; polling scoped to reduce invocations | §6 |

The last two are worth dwelling on. **Copyright governance over a document corpus is an unusually current thing to have built** — AI and intellectual property is a live commercial and regulatory issue, and you implemented a control for it in 2024. And **human-in-the-loop review is the single most transferable AI-product skill you have**, because every serious AI deployment needs it and most engineers have never built one.

### 16.3 Phrasings — use these, avoid those

**Defensible (use freely):**
- "Built and owned the production platform for an AI-powered document-processing product, covering upload, asynchronous job orchestration, human review and delivery."
- "Engineered the human-in-the-loop review workflow for AI-generated content, including per-item editing, review-state tracking and an approval gate before output release."
- "Designed asynchronous orchestration and state management for long-running AI inference jobs, progressing from request-time fetch to scoped polling to WebSocket delta push."
- "Built the review and delivery surface for an NLP pipeline producing several hundred extracted question–answer pairs per document."
- "Implemented rights-governance controls over an AI document corpus, classifying shareable versus copyrighted source material at ingestion."
- "Worked within a four-level document-segmentation data model (document → article → paragraph → Q&A) produced by an upstream NLP pipeline."

**Not defensible (do not use for this project):**
- ❌ "Integrated AI into their system" / "Led AI integration"
- ❌ "Built an AI-powered PDF-to-Q&A engine"
- ❌ "Developed an NLP / machine-learning pipeline"
- ❌ "Worked with LLMs" or any named model or vendor
- ❌ "Implemented prompt engineering / RAG / embeddings"

**Weak — avoid for a different reason:** "Worked on an AI product." True, but it claims nothing and invites the question you do not want to answer vaguely. The specific platform claims above are stronger *and* safer.

### 16.4 How to handle it in an interview

Volunteer the boundary early; do not wait to be caught at it. Something like:

> "The extraction model itself was owned by another part of the team — I didn't build or call it. What I owned was everything around it: getting documents in, orchestrating jobs that took minutes rather than milliseconds, surfacing failure states, and building the review workflow, because the generated pairs needed human correction before anyone would trust them. A few hundred pairs per document, all editable, with the approval gate before release."

That answer is stronger than a vague claim to AI integration, because it demonstrates you understand where the hard engineering in an AI product actually sits. Interviewers notice candidates who state their boundaries precisely; it is a seniority signal.

### 16.5 Where your actual AI-integration claim should come from

You mentioned a separate freelance engagement where you genuinely integrated AI into a client's system. **That is where the "I integrated AI" claim belongs, and it should carry the model names, the integration pattern and the problems you solved.** Keeping the two projects distinct is what makes both credible: this one gives you AI *platform* experience with nine months of verifiable ownership, and that one gives you AI *integration* experience. Together they tell a coherent story. Blended, they produce a claim that collapses under one question.

**Recommended next step:** run this same audit against that freelance project. If it has the model calls, prompt handling or retrieval logic this one lacks, it will produce the evidenced AI-integration claims you want — and your CV can then draw the AI narrative from the project that actually supports it.

---

## 17. Portfolio Case Study Material

*Raw material; detail sits in §2, §4, §5, §9–§11 and §13.*

**Framing.** A nine-month unpaid engagement on an early-stage product, in which the contributor arrived as one of nineteen and left as the only developer on the application — and used it as his entry into AWS.

**Challenge.** The product sat on top of an asynchronous pipeline the web tier had no visibility into and no event feed from. Meanwhile the API layer it depended on mixed an ORM with raw SQL, re-synced the database schema on every cold start and carried credentials in source. Everything hard followed from those two facts.

**Role.** Joined January 2024 on ticketed frontend work (admin dashboard, charts). By April he was the only person committing to the frontend. In May he created the backend API repository and rebuilt the service. Through to October he shipped upload, Q&A editing, downloads, status delivery, account deletion and the deployment tooling.

**Solution in one paragraph.** A layered Fastify API in TypeScript on AWS Lambda behind API Gateway, using parameterised SQL against PostgreSQL on RDS and issuing presigned S3 URLs so document bytes never pass through the API; Auth0 identity with every document query scoped by JWT subject; a second, isolated Lambda for account deletion, which needs outbound internet access to Auth0's Management API; a standalone WebSocket service for status, after the in-process approach proved incompatible with Lambda; and a hand-built deployment pipeline packaging code and dependencies separately.

**Results.** The full lifecycle shipped and ran in production. **No production metrics survive** — see §7.

**What I would do differently.** Publish status events from the pipeline instead of polling anything. Check the runtime's constraints before adding a dependency. Never comment out an authorisation check. Put a CI pipeline in place that can actually fail. Connect the admin dashboard to live data before it looks finished. Record before/after numbers when optimising, and take a metrics snapshot before an engagement ends.

---

## 18. Interview Story Bank

**S1 — Inheriting and rebuilding a broken API (lead with this one).** *S:* a Lambda-hosted API mixing Sequelize and raw SQL, syncing the schema on every cold start, with credentials hardcoded in source. *T:* make it fast, safe and maintainable without a spec, against a database shared with a pipeline I didn't own. *A:* inferred the contract from the code and the consuming frontend; rebuilt as a layered service; dropped the ORM; standardised on parameterised SQL; moved config to environment; added layer-based deployment so the smaller dependency set actually sped up deploys. *R:* no ORM on the cold path, no schema sync at boot, no secrets in source. *E:* archived snapshot vs. the repository I created three days later. *Discipline:* say "removed a schema sync from the cold path," never a percentage — the predecessor instrumented the slow path but never recorded figures, and I should have.

**S1a — Volunteering for the problem nobody else could take (ownership / stakeholders).** *S:* the API was slow, and it was a recurring topic in team discussions. The cloud architect on the project was an AWS solutions specialist rather than a hands-on Node developer, so the problem sat unowned. *T:* nobody had picked it up; the subject volunteered because Node.js was his area. *A:* read the existing implementation, found six distinct defects — two connection pools to one database, no warm connections against a remote database, a 30-second acquire ceiling, a broken handler leaking connections, double releases, and an unawaited schema sync — and rebuilt the service rather than patching around them. *R:* response times went from seconds to milliseconds when tested the same way before and after. *E:* the archived pre-rewrite code and the repository he created three days later; the organisational detail is **his account** — the Microsoft Teams history is no longer accessible to him.

*Why this is your best ownership story:* it is not "I was assigned a performance ticket." It is "a problem sat unowned between disciplines, I recognised it was in my area, and I took it." That is the thing interviewers are actually probing for when they ask about initiative. Deliver the organisational context as narrative and the six defects as specifics — the specifics are what make it credible.

**S2 — Learning a runtime's limits the hard way.** Added `@fastify/websocket` to push status from the Lambda API; Lambda cannot hold persistent connections; removed both packages and built a standalone `ws` service instead. A clean "I was wrong, here's how I found out and what I did" story. *E:* commit `acd6a13`, 13 Jul 2024.

**S3 — The same problem, solved three times.** Load-time fetch → per-document 30s polling excluding terminal states → WebSocket delta push. *Follow-up you should volunteer:* the right answer was pipeline-published events; polling was the best available option, not the best possible one.

**S4 — Distributed consistency without transactions.** Deleting a user across Auth0, PostgreSQL and S3: the ordering, why Auth0 goes first, and the residual failure mode you did not close.

**S5 — Ownership, and why unpaid.** Nine months, arrived as one of nineteen, left as the only developer, on your own laptop, to get production AWS experience. Answers "tell me about taking initiative" and the "why volunteer?" question in one.

**S6 — A framework fighting its runtime.** Fastify expects one lifecycle; Lambda reuses containers; duplicate route registration appears only on warm starts.

**S7 — A security finding in my own code.** The commented-out ownership check and the resulting IDOR: how you'd fix it and what practice prevents it. *Caveat: found in this audit, still open — do not claim it was caught and fixed at the time.*

**S8 — Build and deployment ownership.** Lambda packages too large → layer split and SDK-based tooling; why hand-roll rather than adopt SAM, and what that cost.

**S9 — Reflection.** A dashboard rendering real charts over hard-coded JSON with `// Fetch data from API here` still in place.

---

## 19. Technical Article and Blog Opportunities

| Proposed title | Core engineering problem & why it's interesting | Must not publish |
|---|---|---|
| **I deleted the ORM and the API got better** | A hybrid Sequelize/raw-SQL layer running `sync()` on every Lambda cold start. Why ORMs and short-lived function runtimes fight each other, and what you give up (migrations, validation) by going raw. | Real table/column names, the connection string, the client's identity |
| **Your WebSocket server can't live in a Lambda** | Added the plugin, discovered the constraint, removed it, built a standalone service. Short, useful, and the kind of mistake that is rarely written up. | Gateway hostnames, function names |
| **Running Fastify on AWS Lambda: the warm-start trap** | Fastify assumes a single lifecycle; Lambda reuses containers. Fails *only* on warm starts, so it passes local testing and first deploy. | AWS account IDs, hostnames |
| **Deleting a user across three systems you can't put in a transaction** | Identity provider + database + object storage. Fail-safe ordering, what each failure point leaves behind, how an outbox would close the gap. | Real schema; use "identity provider" generically |
| **Three ways to tell a user their job finished** | The cost/latency/complexity curve from fetch to polling to delta push — and why the right answer is often owned by another team. | Internal status vocabulary |
| **Keeping files out of your API: presigned S3 URLs both ways** | Upload and download flows, expiry trade-offs, and the validation you give up when the API never sees the bytes. | Bucket names and key prefixes |
| **The authorisation check I commented out** | A disabled ownership filter became an IDOR. Frame strictly as a lesson from your own code — safe to write because the system is decommissioned. | Any detail tying it to TiiQu |

---

## 20. Professional Content Opportunities (LinkedIn / X)

Each anchored in something real and verifiable.

1. **"I volunteered for nine months and ended up the only developer on the product."** — the contributor breakdown proves it; the best post you have, because it is about initiative, not technology.
2. **"The best code I wrote on that project was the code I deleted."** — removing Sequelize, the cold-start schema sync and a hardcoded credential.
3. **"Your WebSocket server can't live in a Lambda. I found out the expensive way."** — commit `acd6a13` is the receipt.
4. **"I solved the same problem three times in four months."** — three implementations, all dated in the history.
5. **"Your CI pipeline runs `npm build`. That isn't a command."** — literally in this workflow, pinned to Node 14 while Docker built on Node 18.
6. **"Commenting out an auth check is how security holes ship."** — found auditing your own code; self-critique.
7. **"I optimised something and can't tell you by how much."** — the predecessor instrumented the slow path and recorded nothing; measure and write it down.
8. **"The product is gone. So are the metrics."** — take a snapshot while the system is alive, and before your access ends.
9. **Architecture thread** — serverless API + presigned object storage + third-party identity, trade-off at each node.

**Avoid:** any post implying production scale, user numbers or business outcomes.

---

## 21. Private vs Public-Safe Information

**PRIVATE — own records, interviews, CV decisions only.** Client identity (TiiQu Network), product name, repository URLs — usable in interviews and on a CV, but check any agreement you signed before publishing architecture attributed to them by name. AWS resource identifiers (RDS hostname, bucket and prefix names, Lambda function names, gateway host IDs). Database schema specifics: tables, columns, the SQL, the status vocabulary. The IDOR finding and every other weakness in §12 — discuss in interviews as your own retrospective, **never publish tied to an identifiable system**. The faults in the predecessor API: describe the *pattern* publicly, never attributed to a named company or colleague. Contributor names and per-person commit counts. **The archived zip and anything in it.** Internal ticket keys.

**PUBLIC-SAFE — portfolio, LinkedIn, articles, public CV.** Generic architecture ("a serverless REST API in Fastify/TypeScript on AWS Lambda, with PostgreSQL on RDS and presigned S3 transfer"). Stack names and versions. Your contribution percentages and commit counts, stated as repository metrics for "a document-processing SaaS." The volunteer framing and the dates. The engineering problems and solutions described generically. The lessons in §13, de-identified.

**When in doubt, treat it as private** — in particular the data model, the SQL, AWS resource names, and anything tying the security findings or the predecessor code's faults back to TiiQu or a named person.

---

## 22. Evidence Gaps and Further Metrics Worth Collecting

Access ended with the engagement, so most of these now require someone at TiiQu rather than a lookup.

| Missing evidence | Why it matters | Where it might exist | How to get it |
|---|---|---|---|
| Corroboration for the ~11s → ~110ms figure | Would promote C3a from recollection to evidence and unlock the Tier 1 CV bullet — **the highest-value remaining gap by far** | CloudWatch is gone with the account. But engineers almost always *report wins in writing at the time*: **search your own Slack DMs, email, WhatsApp and any Jira or Notion comments from May–July 2024** for the numbers, or a screenshot. These are your records and you still have them | Search your own archives for "11", "110ms", "latency", "faster", "performance" around May–Jul 2024. A single message to a colleague saying "got it down to ~110ms" is contemporaneous evidence and would be enough |
| Documents processed, Q&As generated, users | Would quantify every impact claim | RDS snapshot if taken; **Stripe dashboard — subscriber counts survive an AWS teardown**; Google Analytics | Ask whoever still administers the TiiQu accounts |
| Production launch and decommission dates | Establishes real production lifespan | Team records, Jira releases, DNS history | Ask a former colleague |
| Whether the WebSocket service was deployed | C6 currently has to be hedged | Deployment records; production frontend env config | Ask; or leave hedged — "built" is still a true claim |
| Authorship of the Docker/ECS/CI pipeline | Would upgrade C15 from "operated" to "built" | GitHub blame on the org repo (you no longer have access) | Ask a former colleague, or drop the claim |
| Who wrote the pre-May-2024 API | Affects how you tell the rebuild story | Org repo history | If unknown, keep describing the code rather than the author — which is the better telling anyway |
| A written reference from TiiQu | For a volunteer role, a reference is worth more than a line on a CV: it independently confirms scope and ownership | A former colleague or the project lead | **Worth doing now.** Ask for a short LinkedIn recommendation naming the period and what you owned |
| Your formal title, if any | Distinguishes "most active contributor" from "lead" | Any onboarding email or Slack intro you still have | Search your own email archive |

---

## 23. Appendix — Methodology and Evidence Notes

**Commands used, all read-only:** `git shortlog -sne --all`, `git log --all --author=…`, `git show <ref>:<path>`, `git show --stat`, `git diff --stat`; `find`, `wc -l`, `grep` over source trees excluding `node_modules`; `unzip` of the archived API snapshot into an isolated scratch directory; `nslookup`/`host` against the production RDS hostname; `curl` against four gateways and two domains; `psql` with `SET default_transaction_read_only = on` — **attempted and failed at DNS resolution; no connection was ever established and no query was executed against any production system.**

**Safety confirmations.** No write, DDL, migration or administrative operation was executed anywhere. No production database connection was established. No credential, password, connection string, key or token appears in any output file — environment variables are referenced **by name only**, and the hardcoded credential discovered in the archived zip is described but never reproduced. No customer PII appears in any output. No project file was modified; the only files written are the three in this output folder.

**Counting method and limitations.** Commit counts include all branches and merge commits and are summed across two Git identities on one email address; they measure activity, not lines or difficulty. Line counts cover `.ts`/`.tsx`/`.jsx` under `src/`, excluding fonts, images and lockfiles; one file (`MyDocumentsTable copy.tsx`) was skipped by the shell glob and is excluded from the ~5,300 total. "Sole active developer, Apr–Oct 2024" counts commits in that window only and does not prove no one else was assigned.

**Repository notes.** The engagement period (31 Jan – 29 Oct 2024) is taken from the subject's first and last commits and independently corroborates his recollection of starting in January/February 2024 and contributing for roughly eight months. The archived zip (`frontend_api_layer-4644b582-….zip`, files dated 30 Apr – 19 May 2024) is the immediate predecessor of the Git-tracked backend and is the primary evidence for Problem 1; its authorship is unknown and it contains a hardcoded production credential. The `learn-terraform-rds` clone is unmodified upstream tutorial content and contributes **no evidence**. `FrontEnd-PDF2QA - legacy` is an older clone pointing at the `TiiQu-Network` origin while the active copy points at the subject's fork; counts are taken from the active copy to avoid double-counting.
