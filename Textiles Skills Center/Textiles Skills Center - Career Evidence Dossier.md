# Textiles Skills Center — Career Evidence Dossier

**Audit date:** 17 October 2026 · **Revised:** 17 October 2026 with first-hand engagement context from the author
**Repositories audited:**
- `txs-bk` — https://github.com/AbrahamNobleOX/textiles-backend (ASP.NET Core Web API)
- `txs-fd` — https://github.com/AbrahamNobleOX/textiles-frontend (Next.js 14 / TypeScript)

**Local path:** `~/Documents/Coding/freelance/textiles-skills-center`

> **Evidence classification used throughout:** **VERIFIED** (directly evidenced in code, Git history or data) · **DERIVED** (calculated from verified evidence) · **USER-ATTESTED** (first-hand account from the author; not independently verifiable from the repository, but noted where code evidence is consistent with it) · **STRONGLY SUPPORTED** · **INFERRED** · **UNKNOWN**.

---

## 1. Executive Summary

Textiles Skills Center was a **UK volunteer engagement** (the author's second) to build a web platform allowing a textile-training organisation to sell and manage courses online. It consists of an **ASP.NET Core Web API** backend and a **Next.js 14 + TypeScript** frontend, built across **10 August – 29 December 2024**.

**The project was never completed, never demonstrated and never launched.** The client's product owners disengaged during the build; the point of contact went on a two-week holiday and never made contact again. The planned demo never happened. The root cause was a **requirements mismatch**: the organisation had briefed a bespoke course-selling-and-delivery platform (supplying a competitor site to model and a Miro architecture board), but what they appear to have actually wanted was a WordPress site that listed courses and took membership payments, with course delivery remaining on their existing third-party platform. *(USER-ATTESTED)*

**Role and contribution.** Authorship is unusually clean: **all 61 backend commits** and **22 of 23 frontend commits** are the author's. One frontend commit — the initial login/signup/welcome pages — came from another contributor. The architecture (React + .NET) was agreed with the introducing technical lead; .NET was chosen deliberately as a vehicle for the author to learn C# and ASP.NET Core in depth. *(VERIFIED authorship; USER-ATTESTED context.)*

**What was actually built** — all verifiable in code:
- A **dual Stripe integration**: one-off card payments via Payment Intents *and* recurring subscriptions via Stripe Checkout + Billing Portal, reconciled through **three signature-verified webhook endpoints** including cancellation-versus-reactivation detection.
- **Anonymous-to-authenticated basket merging** via a cookie-carried GUID buyer identity reconciled server-side at login.
- **Role-based authorisation** end to end: ASP.NET Identity roles → JWT claims → server-side `[Authorize]` → client route guards.
- A **SQLite → PostgreSQL migration** for deployment, including runtime `DATABASE_URL` parsing, containerisation and a GitHub Actions image pipeline.

**Production evidence: none, and correctly so.** The system never reached production. The author self-hosted a PostgreSQL instance as a **staging/development environment**; no launch occurred, no real users existed, no transactions were processed. The only database committed to the repository holds **20 seeded placeholder courses, 2 test users, 0 orders, 0 subscriptions**. **This dossier contains no usage, revenue or adoption metrics, and none should ever be claimed.** *(VERIFIED by inspection; USER-ATTESTED as to why.)*

**Why this project is still worth having on record.** Two reasons. First, the engineering is real, substantial and entirely verifiable — a complete commerce domain, a genuinely non-trivial payments integration, and a deployment taken through real infrastructure friction. Second, and more valuable in interviews: this is a **clean, well-understood requirements-validation failure with a lesson the author can articulate precisely**. Three months of correct engineering against an incorrect brief is a story most mid-level engineers cannot tell credibly, because most have never examined one honestly. See §13 and §17.

---

## 2. Project Overview and Engagement Context

| Attribute | Detail |
|---|---|
| Human-readable name | Textiles Skills Center |
| Engagement type | **Volunteer / unpaid** — the author's second UK volunteering role *(USER-ATTESTED)* |
| How it came about | Introduced by a former colleague from the **TiiQu / Qutii** volunteering engagement, who had moved on and later recommended the author to this organisation *(USER-ATTESTED)* |
| Client | Textile Skills Center — a textile-training organisation *(USER-ATTESTED)* |
| Brief | Build a web app to sell courses, modelled on a competitor site the client supplied; architecture sketched on a **Miro board** shared with the author *(USER-ATTESTED)* |
| Stack decision | React + .NET, agreed with the introducing technical lead; **.NET chosen deliberately by the author to learn C#/ASP.NET Core in depth** *(USER-ATTESTED)* |
| Duration | ~3 months of engagement *(USER-ATTESTED)*; repository activity spans 141 days / ~4.7 months *(VERIFIED)* — see note below |
| Outcome | **Incomplete.** Client disengaged pre-demo; no demo, no review, no launch *(USER-ATTESTED)* |
| Hosting | Author self-hosted a PostgreSQL instance — **staging/dev only** *(USER-ATTESTED; consistent with the `DATABASE_URL` work in code)* |
| Backend package | `API` (`Textiles.sln`), .NET target `net9.0`, ASP.NET packages 8.0.x |
| Frontend package | `textile-frontend` v0.1.0 |
| Commit window | 10 Aug 2024 → 29 Dec 2024 |
| Total commits | 84 (61 backend across all refs, 23 frontend) |
| Automated tests | **0** |

**Date note (pin this down before putting it on a CV).** The author recalls roughly three months with the client; repository activity spans 10 Aug – 29 Dec 2024 (~4.7 months). The difference is most likely self-directed learning and scaffolding before the engagement formally began, and/or continued work after contact lapsed. **Before stating dates publicly, settle on an engagement window you can defend** — see §21.

**What the system does.** Visitors browse a paginated, searchable, filterable course catalogue. They add courses to a basket anonymously (cookie-tracked), then register or log in — at which point the anonymous basket is merged into their account. They check out via Stripe with a shipping address, and order status is confirmed asynchronously by webhook. Separately, users can buy a **recurring membership** via Stripe Checkout, manage it through the Stripe Billing Portal, and have cancellation or reactivation reflected back into the application by webhook. Administrators can create, update and delete courses, with images uploaded to Cloudinary.

**The requirements gap.** The brief — reinforced by the competitor site supplied as a model — described a platform that would both *sell* courses and *host and manage* course delivery. The author built toward that. What the organisation appears to have actually wanted was a marketing/listing site with membership payments, leaving course delivery on their existing third-party platform. *(USER-ATTESTED.)* A corroborating detail from the code: orders carry a required `ShippingAddress` and a delivery-fee rule (`subtotal > 10000 ? 0 : 500`) — physical-goods concepts that fit a generic e-commerce model rather than either interpretation of the client's actual need. The domain model was shaped by the reference material and scaffold, not by a validated job-to-be-done.

---

## 3. My Role and Contribution

### Verified personal contribution (Git authorship)

| Repository | Commits by Akintola Abraham | Commits by others | Period |
|---|---|---|---|
| `textiles-backend` | 61 / 61 (100%) | 0 | 10 Aug – 29 Dec 2024 |
| `textiles-frontend` | 22 / 23 (21 + 1 merge) | 1 (another contributor) | 10 Aug – 25 Nov 2024 |

**Evidence level: VERIFIED** (`git shortlog -sne --all` on both repositories).

Commit-message evidence attributes the following to the author:

- **API foundation** — course controller, exception middleware, Swagger with JWT security scheme (Aug 2024).
- **Catalogue query layer** — sorting, searching, category filtering, pagination as composable `IQueryable` extensions (14–15 Aug 2024).
- **Basket** — cookie-based anonymous basket with quantity semantics (14 Aug 2024).
- **Authentication and authorisation** — ASP.NET Identity, JWT issuance with role claims, login-time basket reconciliation (30 Aug 2024).
- **Checkout and orders** — order aggregate, creation from basket, address persistence (13 Nov 2024).
- **Payments** — Stripe Payment Intents, webhook reconciliation, hosted Checkout, Billing Portal (22–23 Nov 2024).
- **Admin course management** — CRUD with Cloudinary upload and orphan-image cleanup (24 Nov 2024).
- **Subscriptions** — subscription entity, controller, Checkout-session and cancellation webhooks (28 Dec 2024).
- **Deployment engineering** — Dockerfile, GitHub Actions Docker Hub workflow, SQLite→PostgreSQL migration, `DATABASE_URL` parsing, environment-driven CORS and Stripe URLs (Oct–Dec 2024).
- **Frontend** — Redux Toolkit store and slices, Axios agent with interceptors, catalogue, basket, checkout, dashboard, order views, admin role gating (Nov 2024).

### Non-code contribution *(USER-ATTESTED)*

- Participated in architecture discussion with the introducing technical lead and worked from a client-supplied **Miro architecture board**.
- Translated a competitor reference site into a domain model and build plan.
- Maintained progress communication with the client's point of contact, including proposing a demo (the email proposing a demo date went unanswered).

### Contribution by another person

One frontend commit (31 Aug 2024) delivered initial login/signup pages, a welcome page and several shadcn/ui primitives (~19 files; insertion count inflated by `package-lock.json`), merged via pull request #1. **This should not be claimed as personal work.** Subsequent auth work by the author (`990867a` "Added auth and reconfigured redux", `b5fd56b` "finished auth") substantially replaced the wiring.

### Still unestablished

- Whether any written brief, contract or volunteering agreement exists beyond the Miro board and correspondence.
- The organisation's internal reasons for disengaging (the WordPress conclusion is the author's reasonable reconstruction, not a stated position).

---

## 4. System Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  Browser                                                             │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Next.js 14 (App Router, TypeScript)                            │  │
│  │  • Redux Toolkit: account / basket / catalog slices            │  │
│  │  • Axios "agent" — request interceptor injects Bearer token,   │  │
│  │    response interceptor parses pagination header + maps        │  │
│  │    400/401/403/500 to toast notifications                      │  │
│  │  • ProtectedRoute — client-side role guard                     │  │
│  │  • MUI + shadcn/ui + Tailwind; react-hook-form + yup/zod       │  │
│  │  • @stripe/react-stripe-js for card element                    │  │
│  └───────────────┬──────────────────────────────┬─────────────────┘  │
└──────────────────┼──────────────────────────────┼────────────────────┘
                   │ JWT Bearer + buyerId cookie  │ Stripe.js
                   ▼                              ▼
┌──────────────────────────────────────┐   ┌──────────────────────────┐
│  ASP.NET Core Web API (Docker)       │   │  Stripe                  │
│  Controllers:                        │   │  • Payment Intents       │
│   Account · Courses · Basket ·       │◄──┤  • Checkout Sessions     │
│   Orders · Payments · Subscriptions  │ 3 │  • Billing Portal        │
│   (+ Error test controller)          │webhooks  • Signed webhooks   │
│  Services: Token · Basket ·          │   └──────────────────────────┘
│            Payment · Image           │
│  ExceptionMiddleware (global 500)    │   ┌──────────────────────────┐
│  AutoMapper · ASP.NET Identity       │──►│  Cloudinary              │
│  JWT Bearer (HMAC-SHA512, 7-day)     │   │  course image upload /   │
│  Fallback controller → SPA index     │   │  delete by publicId      │
└───────────────┬──────────────────────┘   └──────────────────────────┘
                │ EF Core 8
                ▼
┌──────────────────────────────────────┐
│  PostgreSQL — self-hosted STAGING    │   ← never promoted to production
│  SQLite (local development)          │
│  Courses · Baskets · BasketItems ·   │
│  Orders · OrderItem · Subscriptions ·│
│  UserAddress · AspNetIdentity tables │
└──────────────────────────────────────┘

CI/CD: GitHub Actions → docker buildx → Docker Hub (noblexeyi/textiles:latest)
       Trigger: workflow_dispatch (manual); push-on-main trigger is commented out.
```

**Architecture notes (all VERIFIED from source):**

- **Single-deployable option.** `UseDefaultFiles()` + `UseStaticFiles()` + `MapFallbackToController("Index","Fallback")` means the API can serve a built Next.js bundle from `wwwroot` — the design supports deploying both halves as one container. A built frontend bundle was committed to `wwwroot` at one point and later removed.
- **Thin service layer:** four scoped services (`TokenService`, `BasketService`, `PaymentService`, `ImageService`); remaining logic sits in controllers and entity methods (`Basket.AddItem/RemoveItem`, `Order.GetTotal`).
- **Query composition** via extension methods on `IQueryable<Course>` so filtering translates to SQL, with a custom `PagedList<T>` and a `Pagination` response header.
- **Centralised error handling** in `ExceptionMiddleware`, plus a dedicated `ErrorController` exposing seven deliberate error endpoints used to exercise frontend error paths.

---

## 5. Main Workflows

**A. Browse and filter catalogue.** `GET /api/courses` composes `Sort → Search → Filter → PagedList`, returning courses plus a `Pagination` response header that the Axios interceptor converts into a typed `PaginatedResponse`. `GET /api/courses/filters` returns distinct categories.

**B. Anonymous basket → authenticated basket.** `POST /api/basket` resolves buyer identity as `User.Identity?.Name ?? Cookies["buyerId"]`; absent both, a GUID is minted into a 30-day essential cookie. At login, `AccountController.Login` retrieves both the cookie basket and any stored user basket; where an anonymous basket exists it **deletes the stored basket, re-keys the anonymous one to the username, and clears the cookie** — a deliberate "latest intent wins" policy.

**C. One-off checkout.** `POST /api/payments` creates or updates a Stripe Payment Intent sized to subtotal plus delivery fee, persisting `PaymentIntentId` and `ClientSecret` on the basket. `POST /api/orders` converts basket items into immutable `OrderItem`/`CourseItemOrdered` snapshots (title, image, price at purchase time), deletes the basket, and optionally saves the shipping address. `POST /api/payments/webhook` later flips `OrderStatus` to `PaymentReceived` on `charge.succeeded`, matched by `PaymentIntentId`.

**D. Subscription membership.** `POST /api/payments/hostedpage` creates a Stripe Checkout Session in `subscription` mode. `POST /api/subscriptions` records a pending subscription keyed by `SessionId`, rejecting duplicates. `POST /api/payments/hpwebhook` re-fetches the session from Stripe (expanding `line_items`), verifies `payment_status == "paid"`, then writes back `SubscriptionId`, `CollectionMethod`, `Renews`, `CanceledAt`, `CurrentPeriodEnd`. `POST /api/payments/subcanwebhook` handles cancellation **and reactivation**, distinguished by `CanceledAt == null && status == "active"`. `GET /api/payments/portalsession/{sessionId}` issues a Billing Portal URL.

**E. Admin course management.** `POST/PUT/DELETE /api/courses`, all `[Authorize(Roles="Admin")]`, accepting `multipart/form-data`. On update and delete the previous Cloudinary asset is destroyed by `PublicId`, preventing orphaned media.

---

## 6. Technology Stack

**Backend:** C#, ASP.NET Core Web API (`net9.0` target, ASP.NET packages 8.0.x), EF Core 8 (SQLite + Npgsql providers), ASP.NET Core Identity with `int` keys and custom `Role`, JWT Bearer (HMAC-SHA512, 7-day), AutoMapper, Swashbuckle/Swagger with Bearer scheme, Stripe.net 47.1.0, CloudinaryDotNet 1.26.2, Docker.

**Frontend:** Next.js 14.2.5 (App Router), React 18, TypeScript 5, Redux Toolkit 2, react-redux, Axios, MUI 6 + MUI X DataGrid, Radix/shadcn-style components, Tailwind CSS 3, react-hook-form with yup *and* zod resolvers, @stripe/react-stripe-js 3, react-dropzone, react-toastify, date-fns, moment, lodash.

**Infrastructure:** Docker multi-stage build (SDK 8.0 → ASP.NET runtime 8.0), GitHub Actions with `docker/build-push-action@v3` → Docker Hub (`noblexeyi/textiles:latest`), self-hosted PostgreSQL via `DATABASE_URL`, Cloudinary, Stripe.

---

## 7. Engineering Implementation Highlights

- **Composable, SQL-translated query pipeline.** `CourseExtensions.Sort/Search/Filter` each take and return `IQueryable<Course>`, pushing the full predicate into the database; `PagedList<T>.ToPagedList` issues `CountAsync` plus `Skip/Take`. Pagination metadata travels in a custom header, keeping the response body a clean array.
- **Price snapshotting.** `CourseItemOrdered` copies title, image URL and price onto the order at creation, so later catalogue edits cannot retroactively alter historical orders.
- **Webhook idempotency and defensive nulls.** Subscription creation rejects duplicate `SessionId`s. The deployed handlers null-check both the deserialised Stripe object and the matched record before dereferencing — hardening added *after* the first implementation, suggesting it was prompted by a real failure.
- **Server-side verification over client trust.** The hosted-page webhook re-fetches the Checkout Session and Subscription from Stripe rather than trusting the event payload's status.
- **Role propagation.** Roles become claims in `TokenService`, are decoded client-side from the JWT in `accountSlice`, and are enforced by `ProtectedRoute` for UX — with `[Authorize(Roles="Admin")]` as the actual gate. Correct split.
- **Environment-driven configuration.** CORS origins, Stripe return URLs and the database connection all moved from hard-coded values to configuration/environment during deployment work.

---

## 8. Production Evidence Audit — Result: Never Reached Production

This section records precisely what does and does not exist, and why.

**Why there is no production evidence.** The project was abandoned by the client before demo, review or launch. The author self-hosted a PostgreSQL instance to run the system, but it was a **staging/development environment** — no launch, no real users, no transactions. *(USER-ATTESTED; entirely consistent with the repository state, which simply stops mid-deployment-work on 29 December 2024.)*

| Check | Result |
|---|---|
| `.env` file in either repository or parent directory | **Not present** |
| Production connection string | **Not present**; the deployed configuration reads a `DATABASE_URL` environment variable at runtime |
| Production database reachable from this session | **No** — and none exists; the hosted instance was staging |
| Committed database | `API/store.db` on `origin/main` — a **development SQLite file** |
| Rows in committed database | Courses **20**, AspNetUsers **2**, Baskets **0**, Orders **0**, OrderItem **0**, Subscriptions **0** |
| Nature of that data | `DbInitializer` seed data: lorem-ipsum descriptions, placeholder titles, test users `bob@test.com` / `admin@test.com` |
| Logging / monitoring / APM | None beyond default ASP.NET `ILogger` |
| Analytics | None |
| Deployment artefact | Docker Hub target `noblexeyi/textiles:latest` |
| CI trigger | `workflow_dispatch` only — the push-on-main trigger is **commented out** |

**Conclusion (VERIFIED):** No claim about users, revenue, transactions, uptime, latency or adoption has any basis. The counts above are development seed data and must never be presented as production figures.

**What *is* evidenced about reaching a running environment (VERIFIED code + USER-ATTESTED context):** `origin/main` contains a dense deployment sequence — fourteen `prepare for deployment` commits (14–20 Dec 2024), a `fix: CORS Error` commit, five `fix: datetime to timestamp issues` commits, and seven migration-reset commits culminating in `PG_BackToNormal` (29 Dec 2024). This is the signature of deploying against a **real, live PostgreSQL host** and iterating against real failures — which the author confirms was his own self-hosted staging instance. **Defensible claim:** "deployed to a self-hosted staging environment." **Not defensible:** "deployed to production" or "launched."

### Verifiable code and repository metrics (VERIFIED)

| Metric | Value | Source |
|---|---|---|
| Repository activity window | 10 Aug 2024 – 29 Dec 2024 (141 days) | `git log` both repos |
| Client engagement duration | ~3 months *(USER-ATTESTED — reconcile before CV use)* | Author's account |
| Total commits | 84 | `git rev-list --all --count` (61 + 23) |
| Commits authored by the author | 83 of 84 (99%) | `git shortlog -sne --all` |
| Backend C# lines (excl. `bin`/`obj`) | ~4,065, of which ~2,140 are EF-generated migrations | `wc -l` on `*.cs` |
| Frontend TS/TSX lines (`src/`) | ~9,051 | `wc -l` |
| HTTP endpoints (deployed branch) | 25 functional (+7 deliberate error-test endpoints) | `git grep` on `origin/main` |
| API controllers | 6 functional + 2 infrastructure | `origin/main` tree |
| Third-party integrations | 2 (Stripe, Cloudinary) | `API.csproj`, service classes |
| Stripe webhook endpoints | 3 | `PaymentsController` on `origin/main` |
| Domain entities | 11 | `API/Entities` |
| EF Core migrations authored | 5 distinct migrations across the history | `API/Data/Migrations` |
| Automated tests | 0 | no test project or test files found |

---

## 9. Outcome and Impact

**Business impact: none delivered.** The platform was never demonstrated, reviewed, launched or used. No value reached the client. Stating this plainly is the only honest position, and it is a better position than hedging.

**Value to the author (the real return on this project):**

- **Deliberate skill acquisition.** .NET and C# were chosen specifically to learn them in depth *(USER-ATTESTED)*. The result is a complete, working ASP.NET Core application covering Identity, EF Core, JWT, AutoMapper, middleware, DI, containerisation and provider migration — a far stronger learning artefact than tutorial work, because it was built against a real brief with real deployment constraints.
- **A genuine payments integration.** Dual-mode Stripe with three webhook reconciliation paths is commercially relevant experience that many engineers acquire only on the job.
- **A requirements-validation lesson, learned expensively and understood clearly.** See §13.

**Capability built (describable without claiming outcomes):** an end-to-end sales channel for courses supporting one-off purchase and recurring membership, with self-service subscription management and an admin content surface. Cloudinary offloads media; Stripe Checkout and Billing Portal reduce PCI scope and supply dunning and cancellation UI; webhooks maintain consistency without polling.

---

## 10. Significant Engineering Challenges

**1. Two payment models in one codebase.** One-off purchases use Payment Intents with a client-side card element; memberships use hosted Checkout Sessions in `subscription` mode. These have different lifecycles, different Stripe objects (`Charge` / `Session` / `Subscription`) and different reconciliation points, requiring three webhook handlers each deserialising a different payload and resolving a different local record by a different key. Getting it wrong means lost payments or unpaid access.

**2. Subscription state is owned by Stripe, mirrored locally.** The application must represent `Pending → Active → Canceled` *plus reactivation* from out-of-band, possibly out-of-order, retried events. The cancellation webhook distinguishes cancel from reactivate by inspecting `CanceledAt` and `status` together rather than trusting the event type.

**3. Identity continuity across the anonymous/authenticated boundary.** Shopper intent lives in a cookie before login and a database row after it; the login path must decide what happens when both exist.

**4. SQLite-to-PostgreSQL portability.** EF Core abstracts the provider, not its semantics. `DateTime` handling differs (`timestamp with time zone` requires UTC `DateTimeKind`), and the entities use `DateTime.Now`. The five consecutive `datetime to timestamp issues` commits and subsequent migration resets are the visible cost.

**5. Operating a webhook that cannot be invoked locally.** Stripe signature verification (`EventUtility.ConstructEvent`, 300-second tolerance) means the handler can only be exercised against a real Stripe environment with a real signing secret. The later addition of null guards indicates debugging against live traffic.

---

## 11. Architectural Decisions and Trade-offs

**Decision: React + .NET, agreed with the introducing technical lead.**
*Reasoning* — the architecture was discussed collaboratively and sketched on a client-supplied Miro board; .NET was additionally chosen by the author as a deliberate learning investment *(USER-ATTESTED)*. *Trade-offs* — a bespoke two-tier build was heavier than the client's actual need, which a discovery conversation would likely have surfaced. *Result* — strong learning outcome, wrong tool for the client's real problem. **This is the project's central trade-off and should be owned openly.**

**Decision: JWT bearer tokens over cookie/session auth.**
*Problem* — client and API on separate origins. *Reasoning* — stateless tokens avoid server-side session storage and work cross-origin. *Trade-offs* — tokens in `localStorage` (XSS-exposed), **no refresh-token flow**, and a 7-day lifetime that cannot be revoked server-side. *Result* — works; expiry is handled reactively by catching the rejected `fetchCurrentUser` and redirecting with a "session expired" toast.

**Decision: anonymous basket in a cookie, merged at login.**
*Decision* — GUID `buyerId` in a 30-day essential cookie; merge at login with "anonymous wins". *Trade-offs* — per-browser rather than per-person; the merge discards the stored basket rather than unioning. *Result* — simple, predictable, explicitly commented.

**Decision: hosted Stripe Checkout + Billing Portal for subscriptions, custom card element for one-off payments.**
*Trade-offs* — hosted pages minimise PCI scope and supply payment-method and cancellation UI free, at the cost of leaving the app's visual flow; the custom element keeps branding but takes on more responsibility. *Result* — the right tool chosen per flow rather than one pattern forced on both.

**Decision: snapshot course data onto order items.**
*Reasoning* — historical orders must not change when a course is edited. *Trade-offs* — duplicated data, no FK integrity to the live course. *Result* — correct for commerce.

**Decision: SQLite in development, PostgreSQL in deployment.**
*Trade-offs* — **provider-semantic drift discovered only at deploy time**; migrations generated against one provider do not transfer, which is why the migration folder was reset. *Result* — local convenience repaid with interest.

**Decision: no automated tests.**
*Result* — manual verification only; the deliberate `ErrorController` endpoints show error paths were exercised by hand. A genuine gap, and the deployment-era commit churn is partly what untested code costs.

---

## 12. Problems Solved

**P1 — Lost basket on login.** *Situation:* shoppers add courses before authenticating. *Problem:* login replaces browser identity, orphaning the cookie basket. *Investigation:* surfaced immediately after Identity was introduced. *Solution:* unify buyer identity behind `User.Identity?.Name ?? Cookies["buyerId"]` and reconcile both baskets inside the login handler. *Result:* anonymous carts survive authentication. *Evidence:* `AccountController.Login`, `BasketController`, commit `be3bc7b` (30 Aug 2024).

**P2 — Unconfirmed payments.** *Problem:* an order is created before Stripe confirms the charge; treating creation as payment would grant access to unpaid orders. *Solution:* persist `PaymentIntentId` through basket and order; reconcile asynchronously in a signature-verified webhook flipping `OrderStatus` to `PaymentReceived` on `charge.succeeded`. *Evidence:* `PaymentsController.StripeWebhook`, commit `782ee14`.

**P3 — Webhook handler crashing on unmatched events.** *Problem:* the first implementation dereferenced `charge` and `order` unguarded — a null-reference 500, which Stripe then retries. *Solution:* explicit null checks returning `BadRequest`/`NotFound` before any dereference, across all three handlers. *Result:* unmatched events fail fast with definitive statuses instead of erroring into retry loops. *Evidence:* `PaymentsController.cs` diff, `main` → `origin/main`.

**P4 — Orphaned Cloudinary assets.** *Problem:* replacing or deleting a course image left the old asset accruing storage. *Solution:* persist `PublicId` on `Course`; call `DeleteImageAsync(publicId)` on update-with-new-image and on delete. *Evidence:* `CoursesController`, commit `e68555e`.

**P5 — SQLite → PostgreSQL deployment.** *Problem:* (a) the host supplies a `DATABASE_URL` URL, not an ADO.NET connection string; (b) `DateTime` values rejected by `timestamp with time zone`; (c) SQLite-generated migrations would not apply. *Investigation:* iterative deploy-and-observe across ~20 commits, 14–29 Dec 2024. *Solution:* parse `DATABASE_URL` with `System.Uri` into host/port/user/password/database and compose the Npgsql connection string at startup, environment-switched; reset and regenerate migrations against PostgreSQL. *Result:* a PostgreSQL migration set is the final repository state. *Evidence:* `Program.cs` on `origin/main`, migration filenames, commit messages. *Qualification:* proves the work; the target was a self-hosted staging instance, not production.

**P6 — Cross-origin failure in the deployed environment.** *Problem:* CORS hard-coded to `http://localhost:3000`. *Solution:* move the allowed origin to `CORSSettings:AllowedOrigins`; likewise move Stripe success/cancel/return URLs to `StripeSettings:ReturnUrl`. *Evidence:* commit `e6526fb`.

**P7 — The problem that was not solved: validating the brief.** *Situation:* the client supplied a competitor site to model and a Miro architecture board; the stack was agreed with the introducing technical lead. *Problem:* nobody validated the underlying job-to-be-done. The organisation's actual need appears to have been a listing-and-membership site with course delivery remaining on a third-party platform — achievable with WordPress and a membership plugin. *Investigation:* only reconstructed after the client disengaged, by re-reading what they had asked for against what they already had. *Result:* roughly three months of correct engineering against an incorrect specification; the work was never demoed. *Evidence:* USER-ATTESTED, corroborated by the repository stopping mid-deployment with no demo artefact, and by domain-model artefacts (shipping address, delivery fee) that fit neither reading of the real need. **This is the most instructive item in this dossier.**

---

## 13. Lessons Learned

**On requirements — the primary lesson:**

- **A reference site is not a specification.** "Build something like this" describes an output, not a problem. The questions that were never asked: *where do your courses live today? who delivers them? what happens the day after someone buys? what would you do if we built nothing?* Any one of those would likely have surfaced the WordPress answer in week one.
- **Demo in week two, not month three.** The single highest-leverage change. An ugly, partial, clickable demo early would have exposed the mismatch while it was cheap. Going quiet-then-big is a failure mode the engineer owns, not the client — the author proposed a demo only once the build was substantially done.
- **A technical intermediary is not a substitute for the product owner.** The architecture was agreed with the introducing technical lead; the people who understood the business need were a step removed from that conversation.
- **Silence is signal, and it arrives earlier than you think.** Diminishing responsiveness preceded the final silence. Treat a slipping response cadence as a project risk requiring escalation, not a scheduling inconvenience.
- **For volunteer and unpaid work, agree checkpoints in writing.** Not for enforcement, but because a shared date creates the moment where misalignment surfaces.

**On engineering:**

- **Develop on the database engine you deploy on.** SQLite-in-dev / PostgreSQL-in-prod moved an entire defect class into deployment day.
- **Use `DateTime.UtcNow` by default** in anything that may meet PostgreSQL. `DateTime.Now` on `Order.OrderDate` and `Subscription.OrderDate` is the proximate cause of the timestamp failures.
- **Externalise environment-coupled values before the first deploy.** Hard-coded CORS origins and Stripe URLs each cost a deploy-fail-fix round.
- **Webhooks are untrusted, unordered and duplicate-prone.** Null-guard payload *and* record lookup, re-read the provider's API rather than trusting the event body, and make handlers idempotent on the provider's identifier.
- **Client-side route guards are UX, not security.** Correctly implemented here; worth being able to articulate.
- **Zero tests is affordable until deployment, then it is not.** The volume of "prepare for deployment" commits is a legible record of that cost.

---

## 14. Verified Career Claim Bank

**C1.** Designed and built a full-stack course-commerce platform (ASP.NET Core Web API + Next.js 14/TypeScript) as sole author of the backend and near-sole author of the frontend, during a UK volunteer engagement.
*Evidence:* **VERIFIED** — 61/61 backend and 22/23 frontend commits (`git shortlog -sne --all`); engagement type USER-ATTESTED. *Period:* Aug–Dec 2024. *Qualification:* one frontend commit (initial login/signup/welcome pages) was another contributor's. **The project was never completed or launched.**

**C2.** Implemented a dual Stripe integration covering one-off card payments (Payment Intents) and recurring subscriptions (Checkout Sessions + Billing Portal), reconciled through three signature-verified webhook endpoints handling payment success, subscription activation, cancellation and reactivation.
*Evidence:* **VERIFIED** — `PaymentsController`, `SubscriptionsController` on `origin/main`; Stripe.net 47.1.0. *Qualification:* exercised in Stripe test mode; no live-mode transactions.

**C3.** Built a commerce domain model with purchase-time price and product snapshotting so historical orders remain immutable when catalogue items change.
*Evidence:* **VERIFIED** — `CourseItemOrdered`, `OrderItem`, `OrdersController.CreateOrder`.

**C4.** Implemented end-to-end role-based access control: ASP.NET Identity roles issued as JWT claims, enforced server-side with `[Authorize(Roles=...)]` and mirrored by client-side route guards.
*Evidence:* **VERIFIED** — `TokenService`, `StoreContext` role seeding, `CoursesController`, `ProtectedRoute.tsx`, `accountSlice.ts`.

**C5.** Designed anonymous-to-authenticated basket continuity using a cookie-carried GUID buyer identity reconciled server-side at login.
*Evidence:* **VERIFIED** — `BasketController`, `AccountController.Login`.

**C6.** Built a database-side catalogue query pipeline (sort, search, multi-category filter, pagination) using composable `IQueryable` extensions, with pagination metadata in a custom response header consumed by a typed Axios interceptor.
*Evidence:* **VERIFIED** — `CourseExtensions`, `PagedList<T>`, `HttpExtensions`, `agent.ts`. *Qualification:* search uses `ToLower().Contains()`, which does not use an index.

**C7.** Migrated the persistence layer from SQLite to PostgreSQL for deployment, implementing runtime `DATABASE_URL` parsing and regenerating the EF Core migration history against the new provider.
*Evidence:* **VERIFIED** — `Program.cs` on `origin/main`; PostgreSQL migrations dated 29 Dec 2024; five `datetime to timestamp` commits.

**C8.** Containerised the API with a multi-stage Docker build and automated image publication to Docker Hub via GitHub Actions; deployed to a self-hosted PostgreSQL-backed staging environment.
*Evidence:* **VERIFIED** (artefacts) + **USER-ATTESTED** (self-hosting). *Qualification:* the workflow runs on manual dispatch; **staging only — never promoted to production.**

**C9.** Integrated Cloudinary for course media with lifecycle management — upload on create, replace-and-destroy-previous on update, destroy on delete — preventing orphaned assets.
*Evidence:* **VERIFIED** — `ImageService`, `CoursesController`.

**C10.** Chose ASP.NET Core and C# deliberately as a learning investment and delivered a complete, deployable application in an unfamiliar stack — including Identity, EF Core, JWT, middleware, DI, containerisation and a database provider migration.
*Evidence:* **USER-ATTESTED** (intent) + **VERIFIED** (the resulting application). *Why it's strong:* self-directed capability acquisition demonstrated by a working artefact rather than a certificate.

**C11.** Identified, after the engagement lapsed, that three months of engineering had been built against an unvalidated brief — and can articulate precisely which discovery questions would have surfaced the mismatch, and why demoing in week two rather than month three was the decisive missed step.
*Evidence:* **USER-ATTESTED**, corroborated by the repository stopping mid-deployment with no demo artefact. *Why it's strong:* self-aware post-mortem of a real failure, with the engineer's own responsibility owned rather than deflected onto the client.

**C12.** Production usage, revenue, user counts, uptime, adoption.
*Evidence:* **UNKNOWN / NON-EXISTENT — never claim.** The system never launched.

---

## 15. CV Achievement Candidates

> **Framing guidance:** list this as a **volunteer / pro-bono engagement**, with a **non-production** qualifier visible in the entry. Do not lead your CV with it if you have paid production work — it is a supporting entry, and its job is to evidence .NET/payments capability and UK engagement history, not impact.

### High-confidence, quantified

- Built a full-stack course-commerce platform (ASP.NET Core Web API + Next.js 14/TypeScript) as **sole backend author — 61 of 61 backend and 22 of 23 frontend commits** — delivering **25 REST endpoints across 6 controllers** and **11 domain entities** for a UK non-profit volunteer engagement.
- Integrated **Stripe in two payment modes** (Payment Intents for one-off purchases; Checkout Sessions and Billing Portal for recurring memberships) reconciled by **3 signature-verified webhook handlers** covering payment success, subscription activation, cancellation and reactivation.
- Delivered a **~9,000-line TypeScript frontend** (Next.js App Router, Redux Toolkit, MUI/Tailwind) over a **~1,900-line hand-written C# API**, with a centralised Axios client handling auth injection, pagination headers and typed error-to-toast mapping.
- Migrated the data layer from **SQLite to PostgreSQL** for deployment, implementing runtime `DATABASE_URL` parsing and regenerating the EF Core migration history — resolving provider-level `DateTime`/`timestamp` incompatibilities across **5 targeted fixes**.

### High-confidence, non-quantified

- Self-taught C# and ASP.NET Core by delivering a complete, containerised, deployable application — Identity, EF Core, JWT, AutoMapper, middleware, DI and a database provider migration — rather than through tutorial work.
- Designed anonymous-to-authenticated shopping-basket continuity, merging cookie-scoped carts into user accounts server-side at login.
- Implemented end-to-end role-based authorisation: ASP.NET Identity roles issued as JWT claims, enforced by `[Authorize(Roles=...)]` on the API and mirrored by client-side route guards.
- Modelled orders with purchase-time price and product snapshots so historical transactions remain immutable when catalogue content changes.
- Built a database-side catalogue query pipeline using composable `IQueryable` extensions, surfacing pagination metadata via a custom response header.
- Containerised the API with a multi-stage Docker build, automated image publication through GitHub Actions, and deployed to a self-hosted PostgreSQL-backed staging environment.
- Integrated Cloudinary for course media with full asset lifecycle management, eliminating orphaned images on update and deletion.
- Hardened third-party webhook handlers against malformed payloads, unmatched records and duplicate delivery, verifying payment state by re-reading the provider's API rather than trusting event bodies.

### Honest framing options for the outcome

Pick one and use it consistently. All three are defensible; none hides anything.

- *"Engagement concluded before launch when the client's requirements proved to be for a simpler listing-and-membership site than the bespoke platform briefed."*
- *"Built to demo stage; project discontinued by the client before review."*
- *"Volunteer engagement; platform reached a deployed staging environment and was not taken to production."*

### Do not use

- Any claim of launch, users, revenue, transactions, uptime or business impact.
- "Production" as a descriptor for the hosted environment — use "staging".
- Any framing that attributes the outcome solely to the client.

---

## 16. Portfolio Case Study Material

**Overview.** A volunteer engagement to build a platform for a UK textile-training organisation to sell and deliver courses online — and a case study in what happens when a well-executed build meets an unvalidated brief.

**Context.** Introduced by a former colleague from a previous volunteering engagement. The client supplied a competitor site to model and a Miro architecture board; React + .NET was agreed with the introducing technical lead. The author additionally chose .NET as a deliberate learning investment.

**Role.** Sole architect and developer of the ASP.NET Core API; near-sole developer of the Next.js frontend.

**Solution and architecture.** See §4 and §5. Headline design points: composable `IQueryable` query extensions with header-carried pagination; cookie-GUID buyer identity merged at login; purchase-time order snapshots; three distinct Stripe webhook reconciliation paths; environment-switched SQLite/PostgreSQL persistence; optional single-container deployment with the API serving the built SPA.

**Most interesting engineering problem.** Mirroring subscription state that Stripe owns — deriving `Pending → Active → Canceled → Active` from unordered, retried webhook events of three different object types, resolved against three different local keys, distinguishing cancellation from reactivation by inspecting `CanceledAt` and `status` together rather than trusting the event name.

**Outcome.** The client disengaged before the demo. The platform was never reviewed or launched. The probable cause was a requirements mismatch: a listing-and-membership site, not a bespoke delivery platform, was what the organisation actually needed.

**What I'd do differently.** Lead with discovery, not architecture — *where do your courses live today, and what happens the day after someone buys?* Demo an ugly clickable slice in week two. Get the product owner, not only the technical intermediary, into the architecture conversation. Treat a slipping response cadence as a risk to escalate.

**What I'd improve technically.** PostgreSQL from day one; `DateTime.UtcNow` throughout; a refresh-token flow with short-lived access tokens held outside `localStorage`; a webhook event log with idempotency keys and replay; integration tests around the payment and basket-merge paths; drop the inherited shipping-address and delivery-fee concepts from a digital-goods domain; structured logging and health checks before deployment rather than after.

**Presentation note.** This case study is strongest when the retrospective is the headline rather than a footnote. A polished build that shipped is unremarkable; a polished build that *shouldn't have been built that way*, explained by the person who built it, is memorable.

---

## 17. Interview Story Bank

**S1 — "Tell me about a project that failed." (Your strongest story here.)**
*Situation:* A volunteer engagement to build a course-selling platform for a UK training organisation, briefed with a competitor site to model and a Miro architecture board.
*Task:* Deliver a bespoke platform for selling and delivering courses.
*Action:* Agreed a React + .NET architecture with the introducing technical lead and built it — catalogue, basket, auth, orders, dual-mode Stripe payments, subscriptions, admin CMS — deploying to a self-hosted staging environment.
*Result:* Before the demo, the client went quiet and never returned. Reconstructing it afterwards, their real need was a listing-and-membership site with delivery staying on their existing third-party platform — a WordPress-scale problem.
*What I own:* I validated the *solution* with a technical intermediary and never validated the *problem* with the product owner. And I proposed a demo only once the build was substantially done — three months in. An ugly clickable demo in week two would have cost two days and saved three months.
*What I changed:* I now open engagements with job-to-be-done questions — *where does this live today, who delivers it, what happens the day after someone buys, what would you do if we built nothing?* — and I schedule the first demo before I schedule the first sprint.
*Evidence:* Commit history ending mid-deployment with no demo artefact; the author's own account.
**Why this lands:** it is a real failure, the engineer's share of it is owned without blaming the client, and the behavioural change is specific rather than aspirational. Rehearse the "what I own" paragraph — that is the part interviewers are listening for.

**S2 — Integrating an unreliable, asynchronous third party (Stripe webhooks).**
*Situation:* Payment and subscription truth lives in Stripe; the application mirrors it. *Task:* keep local state correct given signed, retried, possibly unmatched events. *Action:* built three handlers verifying the signature, null-checking both the deserialised object and the matched record, and re-fetching the Checkout Session and Subscription from Stripe before mutating state; added a duplicate-`SessionId` guard; distinguished cancellation from reactivation by `CanceledAt` plus `status`. *Result:* handlers fail fast with definitive statuses instead of 500-ing into retry loops. *Evidence:* `PaymentsController` diff between `main` and `origin/main`.

**S3 — Debugging a deployment-only failure (database portability).**
*Situation:* the API ran on SQLite locally and failed against PostgreSQL. *Action:* identified three distinct causes — `DATABASE_URL` is not a connection string, `DateTime.Now` is rejected by `timestamp with time zone`, and migrations are provider-coupled — then parsed the URL with `System.Uri`, composed an Npgsql connection string at startup, and regenerated the migration history. *Result:* a working PostgreSQL deployment. *Lesson:* develop on the engine you deploy on. *Evidence:* `Program.cs` on `origin/main`; commits of 29 Dec 2024.

**S4 — A deliberate trade-off (basket merge policy).**
*Situation:* a user has one basket in a cookie and another on their account. *Action:* chose "anonymous wins" — delete the stored basket, re-key the cookie basket, clear the cookie — reasoning that just-expressed intent is the live one. *Discussion value:* the alternative (union both) is equally defensible; being able to argue either side is the point.

**S5 — Security design: where the guard actually lives.**
*Action:* enforced authorisation server-side with `[Authorize(Roles="Admin")]`, issued roles as JWT claims, used the client `ProtectedRoute` purely to avoid rendering unusable screens. *Discussion value:* leads naturally into honest weaknesses — tokens in `localStorage`, no refresh flow, 7-day non-revocable lifetime — and what you'd do instead today.

**S6 — Learning a new stack by shipping in it.**
*Situation:* chose .NET and C# deliberately, having not worked in them. *Action:* built a complete application — Identity, EF Core, JWT, AutoMapper, middleware, DI, Docker, provider migration — against a real brief with real deployment constraints. *Result:* a working, containerised application in roughly four months of part-time work. *Discussion value:* evidences how you acquire capability, which matters more to most interviewers than which frameworks you already know.

**S7 — Working with ambiguity and a remote stakeholder.**
*Situation:* requirements arrived as a competitor URL and a Miro board, mediated by a technical intermediary rather than the product owner. *Discussion value:* what you'd now insist on before writing code — written acceptance criteria, direct product-owner access, a dated first demo — and how you'd escalate a slipping response cadence rather than absorb it.

---

## 18. Technical Article and Blog Opportunities

**A1 — "I spent three months building a .NET platform. They wanted WordPress."**
*Core problem:* requirements validation — the difference between the solution someone describes and the job they need done. *Why interesting:* almost every engineer has lived a version of this; very few write it up honestly. *What to explain:* why a reference site is not a specification; the four discovery questions that would have caught it; why demoing in week two beats demoing in month three; how a technical intermediary can accidentally insulate you from the product owner. *Anonymise:* the organisation, the individuals, the competitor site. *Do not publish:* anything identifying the client or the introducer, and any criticism of named people. **Write this one about your own decisions, not theirs — that is what makes it publishable and what makes it land.**

**A2 — "Mirroring Stripe subscription state without lying to your users."**
*Core problem:* the provider owns the truth; your database is a cache updated by unordered, retried webhooks. *What to explain:* re-fetching rather than trusting the event body; detecting reactivation via `CanceledAt` + `status`; idempotency on the provider's identifier; failing fast on unmatched records. *Anonymise:* keys, price IDs, product names, Docker Hub namespace.

**A3 — "SQLite in dev, Postgres in prod: a bill that comes due on deploy day."**
*What to explain:* parsing `DATABASE_URL` safely with `System.Uri`; `DateTime.Now` vs `DateTimeKind.Utc` against `timestamp with time zone`; why migration histories are provider-coupled and what resetting one costs. *Hook:* five consecutive commits with the same message.

**A4 — "Carts that survive login."**
*Core problem:* identity changes mid-session; intent must not. *What to explain:* cookie-GUID buyer identity, the `User.Identity?.Name ?? cookie` resolution, the merge decision and its alternatives.

**A5 — "Pagination in the header, not the body."**
*What to explain:* composable `IQueryable` extensions keeping filtering in SQL; `PagedList<T>` with `CountAsync` + `Skip/Take`; a custom `Pagination` header and typed Axios interceptor. *Honest caveat worth including:* `.ToLower().Contains()` doesn't use an index — discuss `ILIKE`/full-text as the next step.

---

## 19. Professional Content Opportunities (LinkedIn / X)

Every idea below is anchored in evidence; none is generic.

1. **"The most useful thing I built last year never shipped."** The headline post. Three months, a complete .NET platform, no demo — and the four discovery questions that would have prevented it. High authenticity; this is the one most likely to resonate.
2. **"Demo in week two."** Short, single-point post: the cost difference between an ugly clickable slice early and a polished build late, using this project as the receipt.
3. **"A reference site is not a specification."** On the gap between "build something like this" and a job-to-be-done.
4. **"The bug that only exists in production."** The SQLite→PostgreSQL timestamp saga — five commits with the same message.
5. **"Your webhook handler will be called with data that doesn't match anything you have."** Null guards added *after* the first implementation, and why re-reading the provider's API beats trusting the payload.
6. **"Cancelled or reactivated? Same webhook."** Deriving subscription state from `CanceledAt` + `status` instead of event names.
7. **"Client-side route guards are not security."** `ProtectedRoute` (UX) vs `[Authorize(Roles="Admin")]` (the gate).
8. **"I picked the language I didn't know on purpose."** On choosing .NET as a learning investment and what shipping in an unfamiliar stack teaches that tutorials don't.
9. **"I shipped it without tests. Here's the invoice."** The cluster of "prepare for deployment" commits as the visible cost of deferred testing.

**Do not post:** any claim of users, revenue or scale; anything identifying the client, the introducer or the competitor site; any framing that blames the client.

---

## 20. Private vs Public-Safe Information

### Private (own records, interviews, CV decision-making)
- The client organisation's identity, the introducer's identity, and the competitor site used as a model.
- The TiiQu/Qutii connection as the route in.
- Repository URLs and Docker Hub namespace (`noblexeyi/textiles`).
- The other contributor's identity and the precise scope of their commit.
- Your reconstruction of *why* the client disengaged — defensible in a 1:1 interview, **not** for publication.
- Seed-data contents and test account identifiers committed to the repository (`bob@test.com`, `admin@test.com` and the seed password) — these should not be reproduced publicly or reused anywhere.
- The hard-coded Stripe price ID in `PaymentsController.GenerateURL`.

### Public-safe (portfolio, LinkedIn, articles, public CV)
- Architecture patterns and the component diagram with the client generalised ("a UK training organisation").
- Technology stack.
- All non-quantified achievement bullets in §15.
- Code-level metrics in §8 — these describe your work, not a client's business.
- All engineering lessons (§13) and article angles (§18), **with the failure narrated about your own decisions rather than the client's**.
- The requirements-validation lesson in the abstract.

**When in doubt, private.** Never present the 20 seeded courses or 2 test users as usage figures; never imply production scale; never name the organisation in a post about a project that failed.

---

## 21. Evidence Gaps and Further Metrics Worth Collecting

Most of the original gaps are now closed by the author's account. What remains:

| Gap | Why it matters | Where it might exist | Question to settle |
|---|---|---|---|
| **Precise engagement dates** | Repository activity spans ~4.7 months; recollection is ~3 months. CV dates must be defensible. | Email thread with the point of contact; the Miro board's creation date; the introducer's messages | "When did the first and last client contact occur?" |
| **The demo email and final correspondence** | Documents your professionalism and that you pursued closure — useful if the engagement is ever questioned | Sent mail archive | Locate and archive the thread (don't delete it) |
| **The Miro board** | The only surviving artefact of agreed architecture; evidences collaborative design, not solo guesswork | Miro account, or the introducer's | "Do I still have access to the board?" |
| **Whether a volunteering agreement exists** | A written agreement strengthens the UK volunteering-history record | Email, the volunteering platform used, the organisation | "Was anything signed or confirmed in writing?" |
| **Stripe account state** | Confirms test-mode-only, which is what you'll say in interviews | Stripe Dashboard | "Any live-mode activity? (Expect none — confirming is still worth it.)" |
| **Frontend code matching the deployed backend** | `origin/main` references `/dashboard/membership`, which doesn't exist in this frontend checkout — a later frontend version is missing | Another clone or branch of `textiles-frontend`; the GitHub repo's current state | "Is there a newer frontend branch with the membership UI?" |
| **Screenshots or a recording of the working app** | The single highest-value missing artefact for portfolio use — the code can't show a reviewer what it looked like | Old screenshots; or **run it locally today and record a short walkthrough** | **Recommended:** spin it up against SQLite and capture a 2-minute screen recording |

---

## Appendix — Evidence Notes and Methodology

**Sources.** (1) Direct inspection of both repositories on local disk, including all local and remote-tracking refs — source files read directly; history read with `git log`, `git shortlog`, `git show`, `git diff`. (2) A first-hand account from the author, recorded 17 October 2026, supplying engagement context, outcome and root cause. Statements from source (2) are marked **USER-ATTESTED** and are noted where repository evidence is consistent with them.

**Important ref note.** The backend working copy is **not** at its remote's latest state. `txs-bk` local `main` is at `e68555e` (24 Nov 2024); `origin/main` is 35 commits ahead at `a660fca` (29 Dec 2024) and contains the subscriptions feature, PostgreSQL configuration and deployment work. **All architecture, endpoint and feature statements describe `origin/main`** unless stated otherwise. `txs-fd` local `main` matches its `origin/main` at `cd69ae8` (25 Nov 2024) and therefore predates the backend's membership endpoints.

**Database inspection.** `API/store.db` was extracted from `origin/main` into a session scratch directory and queried read-only with `sqlite3` (`COUNT(*)` only): Courses 20, AspNetUsers 2, Baskets 0, Orders 0, OrderItem 0, Subscriptions 0 — `DbInitializer` seed data. **No production database was accessed because none exists**; the author's self-hosted PostgreSQL instance was a staging environment and no `.env` or connection string is present in the working copy.

**Metric derivations.**
- *Commit counts:* `git rev-list --all --count` → 61 (backend), 23 (frontend); authorship via `git shortlog -sne --all`.
- *Activity window:* earliest commit 2024-08-10, latest 2024-12-29 → 141 days ≈ 4.7 months of repository activity (not effort, and not necessarily the engagement window — see §21).
- *Endpoint count:* `git grep -E '\[Http(Get|Post|Put|Delete)' origin/main -- 'API/Controllers/*.cs'` → 32 attributes, less 7 on the deliberate `ErrorController` test endpoints → **25 functional endpoints**.
- *Line counts:* `wc -l` on `*.cs` excluding `bin`/`obj` → 4,065 total, of which EF-generated migration and snapshot files account for ~2,140 → ~1,925 hand-written C#. Frontend `src/**/*.{ts,tsx}` → 9,051 lines (includes vendored shadcn/ui and MUI theme-customisation files).

**Security handling.** No credential, token, connection string or secret value appears anywhere in this dossier. Configuration keys are named (`JWTSettings:TokenKey`, `StripeSettings:SecretKey`, `StripeSettings:WhSecret`, `Cloudinary:ApiKey`, `DATABASE_URL`) without values. No customer PII exists; the only user records are two seeded test accounts, named here solely to flag them as repository-committed test credentials that should not be reused.

**Known limitations.**
1. No production environment ever existed; all operational claims are and will remain unprovable.
2. The frontend checkout predates the backend's subscription endpoints, so the subscription UI could not be reviewed.
3. The engagement duration conflicts mildly with repository activity span (§21).
4. The root cause of the client's disengagement is the author's reasonable reconstruction, not a position the organisation stated.
5. Line counts conflate hand-written and vendored/generated code; stated splits are approximate.
6. No issue tracker, written brief or client correspondence was available to this audit.
