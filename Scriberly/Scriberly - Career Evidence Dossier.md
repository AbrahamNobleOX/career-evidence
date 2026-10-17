# Scriberly — Career Evidence Dossier

**Subject:** Abraham Akintola (`abramseyakid@gmail.com`, Git identities `AbrahamNobleOX` / `Abraham Akintola` / `abramseyakid`)
**Project:** Scriberly — AI-assisted CV and cover-letter platform
**Client / delivery context:** Fixed-scope freelance contract to build the AI and payments layer of version 1 of the product, for a startup working with the agency Fifteen (`@fifteen.co.uk` domains dominate the committer list); public product domain `scriberly.co.uk`. Engagement completed and paid; the product went live after the contract ended.
**Evidence window for this contributor:** 5 November 2025 – 21 January 2026
**Dossier compiled:** 15 October 2026
**Evidence base:** full working copies of three Git repositories (`api`, `frontend`, `demo-ii`), their complete commit histories including remote branches, source code, configuration, migrations, and in-repo engineering documentation.

> **Production database access: not available.** The brief anticipated a `.env` containing production database credentials. The `.env` present in `api/` is a **local development** configuration (`APP_ENV=local`, `APP_DEBUG=true`, `DB_HOST=127.0.0.1`, `DB_DATABASE` pointing at a local schema, `MAIL_MAILER=log`). No production, staging, or remote database host appears in any `.env`, config file, or infrastructure definition in the workspace, and no local MySQL server is running. **No production usage statistics could therefore be measured.** The reason is structural, not an oversight: this was a fixed-scope contract for version 1, and repository and database access were revoked when it ended — before the product went live (§2A). No production metrics exist that could be attributed to this contributor, and none are claimed. Every quantitative figure in this dossier is derived from code and Git history only, and is labelled accordingly. **§21 sets out how to present this project credibly without them.**

---

## 1. Executive Summary

Scriberly is a commercial SaaS product that helps job-seekers build, optimise and ATS-tune CVs and cover letters using generative AI. The platform is a Laravel 11 / PHP 8.3 REST API (with Statamic 5 as CMS and Laravel Sanctum for auth) consumed by a Next.js front end.

**Abraham's contribution was the AI and monetisation layer of the product** — not the CV builder itself, which predates him. Between November 2025 and January 2026 he designed, built and shipped:

1. **The entire OpenAI integration** — six AI generation features (personal summary rewrite, cover letter generation, ATS cover letter generation, cover letter optimisation, full CV optimisation, ATS CV optimisation, plus single-job-entry and single-answer optimisation) built on OpenAI's Responses API with `json_schema` strict structured outputs.
2. **A token-credit economy ("Scriberly Tokens" / "Ink Coins")** — a per-user wallet with debit/credit, lifetime accounting, pre-flight cost quotation, transactional charging, and a configurable margin model.
3. **Stripe monetisation end to end** — Checkout sessions for tiered top-up packs, webhook handling, an idempotent fulfilment service, a billing portal, receipt emails, and a transaction ledger.
4. **A database-driven AI control plane** — AI model/pricing records, system-instruction records, output-token limit multipliers, per-feature fixed prices and a global margin setting, all editable at runtime without a deploy.
5. **An observability layer** — an `ai_request_stats` table and API capturing the full token, cost, margin and response breakdown of every AI call.
6. **A standalone React 19 demo/QA console** (`demo-ii`, 139 commits, sole author) that exercises every AI feature, cost-confirmation flow, Stripe top-up and admin pricing control.

**Strongest verified evidence:**

| Measure | Value | Source |
|---|---|---|
| Non-merge commits to the API repo | **176** (174 reachable from `origin/master`) | `git log --all --author=abramseyakid --no-merges` |
| Share of all non-merge commits on `origin/master` | **174 of 692 ≈ 25.1%** | `git log origin/master --no-merges` |
| Net code contribution to the API | **+18,072 / −7,033 lines** | `git log --numstat` |
| Pull requests merged (Bitbucket, branch `dev/abraham`) | **35** (PR #402 – #452) | merge commit subjects |
| New application files authored | **33** under `app/` | `--diff-filter=A` |
| Database migrations authored | **35** | `--diff-filter=A` on `database/migrations/` |
| Demo console repo | **139 commits, 100% sole authorship, +16,203 / −3,150 lines** | `demo-ii` Git history |

All of this work is merged into `origin/master` (reached via `staging`; `origin/master` head dated 2026‑02‑11 contains all 174 commits), which is the strongest available evidence that it shipped.

**The shape of the engagement is itself career evidence.** Abraham was brought onto an existing product by personal referral from a former manager, for one reason: the team needed AI and payments integrated and did not have someone who could do it. He was given the backend; when the remote front-end developer could not integrate from written documentation alone, the client extended his scope to building a working reference front end as well. He delivered both, the contract completed, and the product subsequently went live. See §2A for the engagement narrative and §21 for how to present this project when no production metrics are available.

---

## 2. Project Overview

**What it does.** Scriberly lets a user build a structured CV (profile, employments with Q&A-driven job descriptions, education and qualifications, skills, languages, references, hobbies, web/social links, "outside of work" activities), generate PDFs from multiple templates, track job applications, and — the part Abraham built — apply AI to rewrite and optimise that content for specific roles and for ATS (Applicant Tracking System) parsing.

**Who uses it.** Individual job-seekers (B2C). Registered users authenticate by email/password or social login (Google, LinkedIn, Apple, Facebook). A HubSpot integration (built by other contributors) syncs user profiles, CVs, cover letters and job records into a CRM, implying a commercial/marketing operation behind the product.

**The business problem Abraham's work solved.** Generative AI features have a direct, variable, per-request cost. Shipping "free" AI into a consumer product is financially unbounded. Scriberly needed:
- AI features good enough to be the product's differentiator;
- a way to charge users for them that was **predictable to the user** and **margin-positive to the business**;
- a way to change prices, prompts, models and limits **without a code deploy**;
- a way to see, per request, what the business actually paid OpenAI versus what the user was charged.

Abraham built all four as a single coherent subsystem.

**Repositories in the workspace**

| Repo | Role | Stack | Abraham's commits |
|---|---|---|---|
| `api` | Backend REST API + CMS | Laravel 11, PHP 8.3, Statamic 5, Sanctum, MySQL | 176 / 1,128 total |
| `frontend` | Production web app (`scriberfy`) | Next.js 14, React 18, TanStack Query, Tailwind, Radix | **0** |
| `demo-ii` | AI/billing demo & QA console (`scrib-demo`) | React 19, Vite 7, TypeScript 5.9, Ant Design 6, Formik, Yup | 139 / 139 (sole author) |

---

## 2A. Engagement Context, Scope and Outcome

> **Evidence level: ATTESTED.** This section records first-hand account from the contributor. It is not independently verifiable from the repository, but each item below notes where repository evidence is consistent with it. Keep this distinction when reusing the material: the *engineering* claims in this dossier are VERIFIED; the *engagement* claims are ATTESTED and corroborated.

**Nature of the engagement.** A fixed-scope freelance contract to deliver version 1 of the AI and payments capability for an early-stage product. Not an employment relationship, not an open-ended retainer — a defined deliverable with a defined end.
*Corroboration:* the commit history is a clean, continuous 11-week block (5 Nov 2025 – 21 Jan 2026) confined almost entirely to AI, token and payment files, with no activity before or after. That is the signature of a scoped contract, not a staff role.

**How the work was won.** Direct referral from a former manager, who recommended Abraham by name as the person capable of doing the AI integration. The client engaged him on that recommendation.
*Why this matters professionally:* being named by someone who has previously managed your work is a stronger quality signal than an interview. It is legitimate to reference this in interviews and, with permission, as a reference contact — it is not a CV bullet, but it is a story.

**The mandate.** The client had a working CV platform and no in-house capability to integrate generative AI or payments into it. That gap was the entire reason for the engagement.
*Corroboration:* before 5 Nov 2025 the repository contains no OpenAI integration, no token wallet, no Stripe code and no AI cost accounting. Every one of those systems appears in this contributor's commits. The gap in the codebase matches the stated gap in the team.

**Original scope: backend plus documentation.** Build the API, document it, and hand the documentation to the front-end developer for integration.
*Corroboration:* OpenAPI/Swagger annotations appear on every endpoint he added, and `storage/docs/openapi.json` is touched across six commits — documentation was treated as a deliverable, not an afterthought.

**Inherited documentation was poor.** The documentation already in the project when he joined was not usable as an onboarding aid; understanding the existing system required reading the code directly. He wrote proper documentation for his own work but did not take on remediating the legacy documentation, which was outside scope.
*Corroboration:* `COPILOT_INSTRUCTIONS.md` exists and is detailed on conventions, but there is no architectural or integration guide for the pre-existing CV/cover-letter domain. The 1,100-commit codebase he joined has no README beyond social-login setup notes.

**Scope extension: a reference front end.** The front-end developer — working remotely from a different country and timezone — was unable to integrate the new API from written documentation alone, including after documentation was provided. Rather than continue iterating on documents, the client asked Abraham, as a full-stack engineer, to build a working front end against his own API so there was functioning integration code to work from.
*Corroboration:* `demo-ii` begins on 30 Nov 2025, three weeks into the engagement — after the backend work was underway, consistent with a mid-engagement scope change rather than an original deliverable. It is 139 commits, sole-authored, and covers every AI, pricing and payment flow end to end: the behaviour of a reference implementation, not a throwaway test page.
*Note on framing:* record this as "the written API contract was not enough for a remote integrator, so I built a working reference implementation". That is the professional substance. Characterisations of the other developer should stay out of written records entirely — they add nothing to your evidence and read badly to any future reader.

**Outcome.** The contract completed on delivery. Repository and production-database access were revoked at that point, which is standard for a contractor rolling off. The product subsequently went live. Abraham was paid in full for the engagement.
*Consequence for this dossier:* the absence of production metrics is a direct result of the contract ending before launch — not of the work being unused or unmeasured. This is a materially better position than "the data exists and I never looked at it", and should be stated plainly rather than avoided (§21).

---

## 3. My Role and Contribution

**Verified personal contribution** (direct Git authorship):

*Backend — AI engine*
- `app/Services/OpenAiService.php` (1,125 lines) — created and iterated across 55 commits.
- `app/Services/OpenAiServiceContinuation.php` (1,748 lines) — ATS CV optimisation and refinement.
- `app/Services/OpenAiServiceContinuationTwo.php` (500 lines) — single-answer-entry optimisation.
- `app/Http/Controllers/Api/CvOptimiseController.php` (733), `CoverLetterAiController.php` (466), `OpenAiController.php` (234).
- `app/Services/SaveAiResponseService.php` (424) — persists AI output back into a new CV/cover-letter record.

*Backend — token economy and pricing*
- `app/Http/Controllers/Api/AiTokenController.php` (656) — five pre-flight cost-quotation endpoints.
- `app/Services/AiTokenCalculatorService.php`, `AiSettingsService.php`, `app/Models/ScriberlyToken.php`, `AiPricingSetting`, `AiFeaturePrice`, `AiLimit`, `AiSystemInstruction`, `AiModel`, `TopupLevel`.
- `app/Http/Controllers/Api/ScriberlyTokenController.php` (206), `AiSettingsController.php` (117), `AiModelController.php` (299).

*Backend — payments*
- `app/Http/Controllers/Api/StripeTokenController.php` (171), `StripeWebhookController.php` (62), `StripeBillingPortalController.php` (61), `PaymentTransactionController.php`.
- `app/Services/StripeService.php`, `app/Services/StripeFulfillmentService.php` (151), `app/Mail/PaymentReceiptMail.php` and its Blade template, `app/Models/PaymentTransaction.php`.

*Backend — observability*
- `app/Models/AiRequestStat.php`, `app/Services/AiRequestStatsService.php`, `app/Http/Controllers/Api/AiRequestStatsController.php`.

Across the 18 primary files listed above: **approximately 6,990 lines of application code** of which he is the originating and dominant author.

*Schema* — 35 migrations creating or altering: `scriberly_tokens`, `ai_models`, `ai_pricing_settings`, `ai_request_stats`, `ai_system_instructions`, `ai_limits`, `ai_feature_prices`, `payment_transactions`, `topup_levels`, and `users.stripe_customer_id`.

*Frontend* — the `demo-ii` console in full: 14 pages, 15 components, typed API client, Formik/Yup validation, auth context, cost-confirmation modals, Stripe top-up and success flows, request-stats inspector and admin margin controls.

*Also evidenced* — OpenAPI/Swagger annotations on all new endpoints (`storage/docs/openapi.json` touched in 6 commits), CORS configuration work including adding the custom `X-AI-Debug` header, and two early refactors (`WebLinksController`, `SocialMediaAuthController`) in his first days on the project.

**Project functionality where individual authorship cannot be attributed to Abraham:** the CV/cover-letter data model and builder, PDF rendering and templates, the HubSpot integration and queue jobs, the Next.js production front end, authentication and social login (he refactored but did not author it), Statamic CMS content, and the job tracker. These were authored principally by `CrisCrassus`/`Cristiano Crassus`, `dwpember`/`David Pember`, `deepfifteen`/`Deep Shah` and `Stephen Medd`.

---

## 4. System Architecture

```
                         ┌───────────────────────────────────────┐
   Browser               │  Next.js 14 web app  (frontend)       │
   (job-seeker)  ───────▶│  React 18 · TanStack Query · Tailwind │
                         └──────────────┬────────────────────────┘
                                        │ Bearer (Sanctum PAT) + CORS
   Internal / QA         ┌──────────────▼────────────────────────┐
   (demo console) ──────▶│   Laravel 11 API   (api)  PHP 8.3     │
                         │                                       │
                         │  Middleware: api.auth (Sanctum), CORS │
                         │  Policies: Cv, CoverLetter, User      │
                         ├───────────────────────────────────────┤
                         │  ── AI & MONETISATION LAYER ──        │
                         │  AiTokenController    (quote cost)    │
                         │  OpenAiController     (summary)       │
                         │  CoverLetterAiController              │
                         │  CvOptimiseController                 │
                         │        │                              │
                         │        ├─ AiTokenCalculatorService    │
                         │        ├─ AiSettingsService (cached)  │
                         │        ├─ ScriberlyToken (wallet)     │
                         │        ├─ OpenAiService(+Cont.,+Two)  │────▶ OpenAI
                         │        ├─ SaveAiResponseService       │      /v1/responses
                         │        └─ AiRequestStat (audit)       │      /v1/responses/
                         │                                       │       input_tokens
                         │  StripeTokenController ──┐            │
                         │  StripeWebhookController─┼─ Stripe    │────▶ Stripe
                         │  StripeFulfillmentService┘  Fulfil    │      Checkout ·
                         │  StripeBillingPortalController        │      Webhooks ·
                         │  PaymentReceiptMail                   │      Billing Portal
                         ├───────────────────────────────────────┤
                         │  Pre-existing: CV/CL CRUD, PDF render │────▶ Api2PDF
                         │  HubSpot sync jobs (database queue)   │────▶ HubSpot
                         │  Statamic 5 CMS · Socialite logins    │────▶ Google/LinkedIn/
                         └──────────────┬────────────────────────┘       Apple/Facebook
                                        │
                         ┌──────────────▼────────────────────────┐
                         │ MySQL  ·  database cache  ·  database │
                         │ queue  ·  local/S3 filesystem         │
                         │ Runtime control tables: ai_models,    │
                         │ ai_pricing_settings, ai_limits,       │
                         │ ai_system_instructions,               │
                         │ ai_feature_prices, topup_levels       │
                         └───────────────────────────────────────┘
```

Diagram derived from `routes/api.php`, `composer.json`, `config/*`, and the service/controller classes named. Components not evidenced in the repository (web server, host, CDN, APM) are deliberately omitted.

---

## 5. Main Workflows

**A. Priced AI generation (the core flow Abraham designed)**

1. **Quote.** The client calls a `calculate-*-tokens` endpoint. The API assembles the exact prompt it would send, then calls OpenAI's `POST /v1/responses/input_tokens` to obtain a **real, not estimated, input-token count**. It derives a maximum output-token budget by multiplying input tokens by a per-feature multiplier held in `ai_limits`, adds a configurable margin percentage from `ai_pricing_settings`, converts the total to Scriberly Tokens at a configurable AI-tokens-per-Scriberly-token ratio, and returns the breakdown alongside the feature's **fixed price** from `ai_feature_prices`.
2. **Confirm.** The UI shows the user the cost before they commit (implemented in `demo-ii`'s `CostConfirmationModal`).
3. **Charge.** Inside a `DB::beginTransaction()`, `OpenAiService::chargeUserTokens()` takes a `lockForUpdate()` row lock on the user's wallet, verifies balance, and debits. Insufficient balance returns `422` with `required_tokens`, and no AI call is made.
4. **Generate.** The OpenAI Responses API is called with a `json_schema` response format set to `strict: true`, a system instruction loaded from `ai_system_instructions` (with a hard-coded fallback), and the computed `max_output_tokens`.
5. **Rollback on failure.** Any throwable inside the transaction rolls back the debit and returns *"AI generation failed; no tokens have been charged."*
6. **Persist.** `SaveAiResponseService` writes the AI output into a **new** CV or cover-letter record, carrying over every un-optimised field from the original so nothing is lost.
7. **Audit.** `OpenAiService::logRequestStat()` writes an `ai_request_stats` row recording model, real input/output tokens, OpenAI's actual fee, tokens charged to the user, the margin tokens and margin fee, the full request payload and the full response.

**B. Token top-up (Stripe).** Client posts a `topup_level` → API ensures a Stripe customer exists on the user → resolves tokens and `stripe_price_id` from the `topup_levels` table (falling back to config) → creates a Checkout session carrying `user_id`, `tokens` and `topup_level` in metadata → returns the hosted URL. Fulfilment runs through `StripeFulfillmentService`, reachable from **two** paths: the `checkout.session.completed` webhook (signature-verified) and a direct `fulfill-checkout` call from the success page. The service is idempotent on both `event_id` and `session_id`, credits the wallet and writes a `payment_transactions` row inside a `DB::transaction`, then attempts a receipt email outside the transaction so a mail failure cannot roll back a paid credit.

**C. Runtime pricing administration.** `GET/PUT /ai-settings` (margin %, AI-tokens-per-Scriberly-token), `GET/POST/PUT /ai-models` (provider, model, per-million input/output costs, active flag), plus the `ai_feature_prices`, `ai_limits` and `ai_system_instructions` tables — all changeable in the database with no deploy.

**D. Granular optimisation.** Beyond whole-CV optimisation, the system supports optimising a **single employment entry** or a **single Q&A answer**, each with its own price, system instruction and token quote — added December 2025 after the whole-CV flow shipped.

---

## 6. Technology Stack

**Backend:** PHP 8.3, Laravel 11, Laravel Sanctum 4, Laravel Socialite 5 (+ Apple provider), Statamic 5 CMS, Stripe PHP SDK 19, Api2PDF + barryvdh/laravel-dompdf, l5-swagger (OpenAPI), spatie/laravel-searchable, rinvex/countries, lcobucci/jwt, MySQL, database-backed cache and queue, Redis configured.
**Frontend (production):** Next.js 14, React 18, TanStack Query 5, Tailwind, Radix UI, React-Quill, date-fns.
**Frontend (Abraham's demo console):** React 19, Vite 7, TypeScript 5.9, Ant Design 6, Formik + Yup, Axios, React Router 7, react-toastify.
**AI:** OpenAI Responses API (`/v1/responses`, `/v1/responses/input_tokens`), `gpt-4o-2024-08-06` default with `gpt-4o`, `gpt-4o-mini`, `gpt-4.1-mini` whitelisted; `json_schema` structured outputs with `strict: true`.
**Tooling / process:** Git on Bitbucket, pull-request workflow with per-developer `dev/*` branches into `staging` then `master`, Laravel Pint, PHPUnit 11 (a Bitbucket Pipelines SQLite test job exists but is **fully commented out**; `COPILOT_INSTRUCTIONS.md` states "No automated tests are currently used by project choice").

---

## 7. Engineering Complexity — and why it was hard

**Charging real money for a non-deterministic, variable-cost operation.** You cannot know an LLM call's cost until after it completes, but a consumer product must quote before it charges. Abraham's resolution: price on *inputs you can measure exactly* (OpenAI's own input-token endpoint) plus a *bounded* output budget (`max_output_tokens` = input × per-feature multiplier), then apply margin to that ceiling. The user is quoted a number that cannot be exceeded; the business is protected against a long generation. The `ai_request_stats` table then records what was actually spent versus charged, so the margin model can be validated against reality.

**Charge-then-generate without stealing from users.** Charging after generation risks giving away paid output to users with no balance; charging before risks taking money for a failed call. The chosen design wraps the debit and the OpenAI call in one database transaction with a `lockForUpdate()` row lock on the wallet — the lock also prevents two concurrent requests from double-spending the same balance — and rolls back on any throwable.

**Getting structurally valid output back from an LLM.** A CV is a nested object graph (employments, each with job roles and a variable-length Q&A list carrying stable `id`/`question_id` values). The model must rewrite the prose and preserve every identifier, count and ordering. The code attacks this on three fronts: `json_schema` with `strict: true` on the API call; explicit, forceful system instructions ("You must rewrite **every** employment entry… Do not omit or add employments. Keep JSON strictly valid and closed"); and a defensive `safeJsonDecode()` that strips markdown fences, extracts the first balanced `{…}` and trims stray trailing characters before giving up. The commit history shows this hardening was iterative and driven by real misbehaviour.

**Payment idempotency across two fulfilment paths.** Because fulfilment can be triggered by a Stripe webhook *and* by the browser hitting the success page, double-crediting is a live risk. `StripeFulfillmentService` guards on both `event_id` and `session_id` before entering its transaction, and `payment_transactions` is the durable record of what was already honoured. A later migration made `event_id` nullable precisely because the direct path has no event ID.

**Making the economics changeable without deploys.** Every pricing lever — model, per-million costs, margin %, conversion ratio, per-feature fixed price, output-token multiplier, and even the system prompts — reads from a database table with a config fallback and a tolerant `try/catch`. A pricing mistake is a database update, not a release. The trade-off (extra queries on every request) is partially mitigated by a short cache on the settings singleton.

**Operating inside someone else's architecture.** The repository's `COPILOT_INSTRUCTIONS.md` prescribes route-path shapes, response envelopes, validation style, migration naming and model conventions in detail. Abraham's 35 migrations and all new endpoints follow those conventions, and his OpenAPI annotations match the existing documentation style — adaptation to an established house style rather than imposition of a personal one.

---

## 8. Architectural Decisions and Trade-offs

| # | Problem | Decision | Reasoning (as evidenced) | Trade-off | Result |
|---|---|---|---|---|---|
| 1 | LLM cost is unknown before the call | Price on exact input tokens from OpenAI's `input_tokens` endpoint × a per-feature output multiplier, + margin | Gives a quotable upper bound; `config/ai.php` comments name the multipliers as cost/quality controls | Extra network round-trip per quote; quote exceeds typical actual cost | Users see a firm price; actual-vs-charged is recorded per request |
| 2 | Charge before or after generation? | Before, inside a DB transaction with a row lock, rolled back on failure | Code comment: "Charge user tokens before generation"; explicit rollback message to the user | Transaction held open for the duration of an external API call | No unpaid generation, no charge for a failed one, no concurrent double-spend |
| 3 | Pure usage-based pricing vs fixed prices | **Both** — a usage breakdown is computed, but a per-feature *fixed price* (`ai_feature_prices`, Jan 2026) is what is charged | Added two months after usage pricing; fixed prices are simpler to communicate | Margin now varies per request instead of being guaranteed | Predictable UX; `ai_request_stats.fixed_price` records what was charged alongside what it cost |
| 4 | Where should prompts, limits and prices live? | In database tables, each with a config fallback | Migrations seed defaults; every read is `optional(...)` with a `config()` fallback | More queries; prompts become untracked state outside Git | Prompts, models and prices are tunable in production without a deploy |
| 5 | How to deliver AI output into the CV? | Write a **new** CV/cover-letter record, copying all un-optimised fields from the original | A dense run of commits on 2 Jan 2026 adds employments, skills, education, activities, additional details and profile fields one by one | Duplicates data; more records per user | The original CV is never mutated; the user compares and keeps what they want |
| 6 | Only the webhook, or also a direct fulfilment call? | Both, behind one idempotent service | Code comment: "mirrors the webhook processing to guarantee immediate fulfillment when customer is present" | Two entry points to keep consistent | Tokens appear immediately on return from Stripe without waiting on webhook delivery |
| 7 | How to expose cost internals without leaking them to end users | An opt-in `X-AI-Debug` request header that toggles the verbose breakdown; responses are rebuilt minimally when absent | The final week of work (20–21 Jan 2026) is almost entirely this | Response shape varies by header; must be handled everywhere | Internal tooling keeps full visibility; public responses expose only the content and the fixed price |
| 8 | One 1,000-line service, or many? | Three sequential classes: `OpenAiService` → `…Continuation` → `…ContinuationTwo` | Names and chronology show file growth drove the split, not domain boundaries | Boundaries are historical, not conceptual; discoverability suffers | Honest limitation — see *Lessons Learned* |

---

## 9. Problems Solved

**9.1 — The model rewrote only the first employment**

*Situation.* CV optimisation sends a user's full employment history for rewriting.
*Problem.* The model returned a complete-looking JSON object in which only the first employment had actually been rewritten; later entries came back unchanged or were dropped, and output was sometimes truncated mid-JSON.
*Investigation.* Evidenced by the countermeasures in code rather than by a written post-mortem: the fix combines schema enforcement, ID enumeration and parse-level recovery, which is the signature of repeated failures at all three levels.
*Solution.* (a) `json_schema` with `strict: true` on the Responses API call; (b) the system instruction is appended at runtime with the explicit list of employment IDs to process and the directive not to omit or add entries and to keep the JSON "strictly valid and closed"; (c) `safeJsonDecode()` recovers from markdown fences, leading/trailing noise and stray terminators.
*Result.* A deterministic contract for a non-deterministic generator; multi-employment optimisation became reliable enough to ship.
*Evidence.* `OpenAiServiceContinuation.php` lines ~82, 134–189, 497, 704, 1120; `OpenAiService::safeJsonDecode()`; commit "Enhance CV optimisation feature to enforce input-output consistency in experience entries" (2025‑11‑10).

**9.2 — Optimisation silently discarded parts of the user's CV**

*Situation.* AI-optimised CVs are saved as new records.
*Problem.* Only the fields the AI returned were being persisted, so anything the AI did not touch — education, skills, additional details, external activities, profile and contact fields, template and section preferences — was lost from the optimised copy.
*Solution.* `SaveAiResponseService` was extended to take the **original** CV as a second argument and backfill every untouched field from it, field by field.
*Result.* An optimised CV is a complete CV.
*Evidence.* Nine commits on 2 January 2026 each adding one category of persistence ("…add persistence for additional details…", "…for external activities…", "…education persistence…", "…skill persistence…", "…additional user profile fields…"); the `?array $originalCv = null` parameter and the `$originalCv[...] ?? null` fallbacks throughout `SaveAiResponseService::createCvFromAi()`.

**9.3 — Double-crediting paid top-ups**

*Situation.* Stripe can deliver `checkout.session.completed` more than once, and the success page also triggers fulfilment directly.
*Problem.* Without guards, one payment credits tokens two or three times — a direct revenue leak.
*Solution.* A single `StripeFulfillmentService` used by every path, checking `event_id` **and** `session_id` against `payment_transactions` before crediting, performing the credit and the ledger write in one `DB::transaction`, and moving the receipt email outside it. Webhook signature verification rejects forged events.
*Result.* Fulfilment is safe to retry from any path.
*Evidence.* "Implement PaymentTransaction model and migration; enhance StripeWebhookController for duplicate event handling" (2025‑12‑13); `StripeFulfillmentService::fulfillSession()`; migration `2026_01_06_000100_make_payment_transactions_event_id_nullable`.

**9.4 — Token balances could not represent real costs**

*Situation.* Wallet balances were created as `unsignedInteger`, and Scriberly-token totals were rounded up to whole numbers.
*Problem.* Integer-only balances forced rounding on every charge, which both distorted pricing and created an unintended "rounding profit" the team had to track as its own field.
*Solution.* Migrations converted the balance and lifetime columns to decimal and then set scale to 2; `AiTokenCalculatorService` moved from `ceil()` to `round(…, 2)`, and `rounding_profit_tokens` was explicitly zeroed (the original line is retained as a comment).
*Result.* Charges reflect computed cost rather than a rounding artefact.
*Evidence.* Migrations `2026_01_05_000002`, `2026_01_05_000003`, `2026_01_06_000000`; the commented-out `ceil()` and `$roundingProfit = 0;` in `AiTokenCalculatorService`.

**9.5 — Internal cost data leaking into public API responses**

*Situation.* AI responses carried the full breakdown — OpenAI model, real token usage, OpenAI's fee, margin percentage, margin tokens — because internal tooling needed it.
*Problem.* That is the company's cost structure and margin, exposed to any authenticated user.
*Solution.* An `X-AI-Debug` request header, parsed by `OpenAiController::parseAiDebugHeader()`; when absent or false, each controller rebuilds a minimal response containing only the generated content and the fixed price. Applied consistently across the AI controllers, the token calculators and the stats endpoint, with the header added to the CORS allow-list.
*Result.* One codebase serves both a safe public response and a full internal one.
*Evidence.* Commits 20–21 January 2026 across `CoverLetterAiController`, `CvOptimiseController`, `OpenAiController`, `AiTokenController`, `AiRequestStatsController`; `config/cors.php` `allowed_headers`.

---

## 10. Business Impact

Impact here is **qualitative and capability-based**, because no production data was available. Each item is a capability the codebase demonstrably provides, not a measured outcome.

- **Monetisation of AI.** The product gained a complete paid-AI path — quote, charge, generate, fulfil, refund-by-rollback, receipt, ledger. Before this work there was no token wallet, no Stripe integration and no AI cost accounting in the repository. *(VERIFIED as a capability; revenue impact UNKNOWN.)*
- **Cost control.** Per-feature output-token ceilings and a configurable margin make the unit economics of every AI feature explicit and boundable. Two separate commits are titled "Update AI usage limits for improved cost control and output quality". *(VERIFIED as intent and mechanism; realised savings UNKNOWN.)*
- **Commercial agility without engineering.** Prices, margin, models, prompts and top-up packs are database rows. A pricing change no longer needs a developer, a release or downtime. *(VERIFIED from code.)*
- **Financial observability.** `ai_request_stats` records, for every single AI call, what the business paid and what the user was charged — the data a finance or product owner needs to verify margin. *(VERIFIED that the mechanism exists; no data read.)*
- **New revenue surface.** Four top-up tiers (`free_on_signup` 350 tokens, `start_pack` 500, `job_hunter_pack` 1,200, `pro_searcher_pack` 3,100) and ten individually priced AI features. A free grant on signup creates a try-before-you-buy funnel. *(VERIFIED from `config/ai.php` and `RegisterController` sync logic; conversion UNKNOWN.)*
- **Product differentiation.** ATS-specific optimisation for both CVs and cover letters, plus granular single-job and single-answer optimisation, materially widen what the product does versus a plain CV builder. *(VERIFIED from endpoints and prompts.)*

---

## 11. Security, Reliability and Operational Considerations

**Implemented and verified**
- Stripe webhook signature verification via `Webhook::constructEvent`; missing secret fails closed with a 500 and an error log.
- Idempotent payment fulfilment keyed on `event_id` and `session_id`.
- Wallet debits under `lockForUpdate()` inside a transaction, with rollback on generation failure.
- Sanctum bearer-token auth on every AI, wallet and payment endpoint; `CvPolicy`/`CoverLetterPolicy`/`UserPolicy` for ownership; explicit `user_id` ownership checks in `AiTokenController` before a CV is read.
- `ai_request_stats` is paginated **per authenticated user** with a hard `per_page` cap of 100 — a user cannot page through other users' AI history, and payload size is bounded.
- Cost internals suppressed from responses unless `X-AI-Debug` is sent.
- OpenAI calls are configured with a 30-second timeout and 2 retries with a 200 ms backoff; transport errors, non-2xx responses and JSON decode failures are each logged distinctly with duration, HTTP status, response size and token usage.
- Structured application logging throughout (`openai.responses`, `openai.responses.transport_error`, `openai.input_tokens.non_2xx`, Stripe fulfilment events), giving usable production forensics.
- Secrets held in environment variables; the code fails fast with a clear message when the OpenAI key is absent.

**Weaknesses and risks visible in the repository** (recorded honestly; useful interview material)
- **No automated tests for this subsystem.** PHPUnit is installed, the Bitbucket Pipelines file is entirely commented out, and `COPILOT_INSTRUCTIONS.md` records the no-tests stance as a project decision. Payment and billing logic without tests is the single largest risk in this work.
- **`CORS_ALLOWED_ORIGIN=*` in the local `.env`**, and the app reads a differently-named key (`CORS_ALLOWED_ORIGINS`) in `config/cors.php` — a latent misconfiguration. Production values are unknown.
- The settings cache TTL is 0.6 seconds, which is close to no caching; every AI request performs several small control-table lookups.
- AI calls are synchronous inside a request/transaction rather than queued, so a slow generation holds both an HTTP worker and a database transaction.
- Prompts live in database rows, so prompt changes are outside version control and have no review trail.

---

## 12. Lessons Learned

1. **Price what you can measure.** Quoting from OpenAI's own input-token endpoint plus a bounded output ceiling turned an unknowable cost into a quotable one. This is the transferable idea from the whole project.
2. **Record charged-vs-actual from day one.** `ai_request_stats` arrived in week three (22 Nov 2025) and every later pricing change — margin precision, decimal balances, fixed prices — was only arguable because that table existed. Instrument the economics before tuning them.
3. **Put the business levers in the database.** Seven weeks of commits consist of moving constants (prices, prompts, limits, models, pack sizes) out of `config/` and into tables. Doing that first would have saved a long tail of deploy-only changes.
4. **Usage-based pricing is honest; fixed pricing is sellable.** The December usage model was replaced in January by fixed per-feature prices while keeping the usage calculation for internal accounting. Users want a price, not a formula.
5. **Constrain LLMs at three levels.** Schema, instruction and parser. Any one alone proved insufficient for nested, ID-bearing JSON.
6. **Never mutate the user's original.** Writing optimised output to a new record made the feature safe to iterate on and gave users a comparison view for free.
7. **Splitting a class by line count is not refactoring.** `OpenAiService` → `Continuation` → `ContinuationTwo` is the clearest thing to do differently: the boundaries should have been per feature or per concern (prompt construction, transport, pricing, persistence) from the start.
8. **Ship billing with tests, whatever the house policy.** The absence of tests around idempotency and wallet debits is the part of this work that would be hardest to defend to a senior reviewer.

---

## 13. Verified Career Claim Bank

> Evidence levels: **VERIFIED** (direct from code/Git) · **DERIVED** (calculated from verified evidence) · **STRONGLY SUPPORTED** · **ATTESTED** (contributor's first-hand account, corroborated by but not provable from the repository) · **INFERRED** · **UNKNOWN**.
> Universal qualification: all figures are Git- and code-derived. No production usage data was accessible, so none of these claims asserts user numbers, revenue or runtime volume.

**C1.** Designed and built the complete AI generation layer of a commercial CV platform — eight distinct AI operations (personal-summary rewrite, cover-letter generation, ATS cover-letter generation, cover-letter optimisation, CV optimisation, ATS CV optimisation, single-job-entry optimisation, single-answer optimisation) on OpenAI's Responses API using strict `json_schema` structured outputs.
*Level:* VERIFIED. *Evidence:* `routes/api.php` lines 307–314; `OpenAiService`/`OpenAiServiceContinuation`/`…Two`. *Period:* Nov 2025 – Jan 2026.

**C2.** Built a token-credit economy from scratch — per-user wallet with transactional debit/credit, lifetime spend/credit accounting, row-level locking against concurrent double-spend, pre-flight cost quotation and automatic rollback when generation fails.
*Level:* VERIFIED. *Evidence:* `ScriberlyToken` model, `OpenAiService::chargeUserTokens()`, migration `create_scriberly_tokens_table`, transaction blocks in `OpenAiController`/`CvOptimiseController`.

**C3.** Delivered Stripe payments end to end — tiered Checkout top-ups, signature-verified webhooks, an idempotent fulfilment service guarding on both event and session ID, Billing Portal access, automated receipt emails and a queryable transaction ledger.
*Level:* VERIFIED. *Evidence:* `StripeTokenController`, `StripeWebhookController`, `StripeFulfillmentService`, `StripeBillingPortalController`, `PaymentTransactionController`, `PaymentReceiptMail`, `payment_transactions` migrations. *Period:* 13 Dec 2025 – 21 Jan 2026.

**C4.** Contributed 176 non-merge commits (+18,072 / −7,033 lines) to the production API across 35 merged pull requests in an 11-week engagement; 174 of those commits are merged into `origin/master`, representing ~25% of all non-merge commits on the main branch.
*Level:* VERIFIED (counts) / DERIVED (the 25% share = 174 ÷ 692). *Qualification:* commit and line counts measure activity, not value; the 25% share counts commits, not functionality.

**C5.** Authored 35 database migrations introducing nine new tables for the AI and billing domain (`scriberly_tokens`, `ai_models`, `ai_pricing_settings`, `ai_request_stats`, `ai_system_instructions`, `ai_limits`, `ai_feature_prices`, `payment_transactions`, `topup_levels`).
*Level:* VERIFIED. *Evidence:* `git log --diff-filter=A` on `database/migrations/`.

**C6.** Designed a database-driven AI control plane allowing models, per-million token costs, margin percentage, per-feature prices, output-token limits, system prompts and top-up pack sizes to be changed in production without a code deploy, each with a config fallback.
*Level:* VERIFIED. *Evidence:* `AiSettingsService`, `AiModel`, `AiFeaturePrice`, `AiLimit`, `AiSystemInstruction`, `TopupLevel` and their seeding migrations.

**C7.** Built per-request AI cost observability capturing model, real input/output tokens, provider fee, tokens charged, margin tokens and margin fee, plus the full request and response payloads, exposed through a paginated, per-user, size-capped API.
*Level:* VERIFIED. *Evidence:* `ai_request_stats` migrations, `OpenAiService::logRequestStat()`, `AiRequestStatsController`.

**C8.** Solved structured-output reliability for nested, identifier-bearing JSON by combining `strict` JSON-schema enforcement, runtime-injected ID enumeration in the system prompt, and a defensive multi-strategy JSON recovery parser.
*Level:* VERIFIED. *Evidence:* `json_schema`/`strict: true` blocks at four call sites in `OpenAiServiceContinuation`; `safeJsonDecode()`; commit of 10 Nov 2025.

**C9.** Independently designed and built a React 19 / TypeScript single-page console (139 commits, sole author, +16,203 lines) covering all AI features, cost-confirmation flows, Stripe top-up and success handling, a request-stats inspector and admin margin controls.
*Level:* VERIFIED. *Evidence:* `demo-ii` repository; `git shortlog -sne` shows one author. *Qualification:* an internal demo/QA tool, not a customer-facing product.

**C10.** Worked to an established team architecture — Bitbucket PR workflow, per-developer branches merged through `staging` to `master`, documented house conventions for routing, response envelopes, validation and migrations, with OpenAPI annotations maintained for every new endpoint.
*Level:* STRONGLY SUPPORTED. *Evidence:* 35 PR merge commits, branch topology, `COPILOT_INSTRUCTIONS.md`, Swagger annotations in new controllers, `storage/docs/openapi.json` updates.

**C11.** Identified and closed a commercial data-exposure gap by making internal cost and margin detail opt-in behind a request header while keeping public responses minimal.
*Level:* VERIFIED. *Evidence:* `parseAiDebugHeader()` and the response-rebuild branches added 20–21 Jan 2026.

**C12.** Engaged by direct referral from a former manager to deliver a capability the client had no in-house ability to build — generative-AI and payment integration — and completed the contract on delivery of version 1, which subsequently went live.
*Level:* ATTESTED (engagement facts) / VERIFIED (the capability gap). *Evidence:* no OpenAI, token-wallet, Stripe or AI-cost code exists in the repository before 2025‑11‑05; all of it is authored by this contributor. *Qualification:* the referral, payment and go-live are the contributor's account; the codebase corroborates the capability gap but cannot evidence the commercial relationship.

**C13.** Had scope extended mid-engagement from backend-only to include a full reference front end, after the remote front-end developer could not integrate the API from written documentation alone — then delivered that front end solo in parallel with continuing backend work.
*Level:* ATTESTED, strongly corroborated. *Evidence:* `demo-ii` begins 2025‑11‑30, three weeks into the engagement, and runs concurrently with API commits through 2026‑01‑21; 139 commits, single author, covering every AI, pricing and payment flow. *Qualification:* the reason for the extension is the contributor's account.

**C14.** Produced API documentation as a first-class deliverable in a codebase whose existing documentation was not sufficient to onboard from.
*Level:* ATTESTED (the quality judgement) / VERIFIED (the documentation work). *Evidence:* OpenAPI annotations on every endpoint added; `storage/docs/openapi.json` updated across six commits; the pre-existing repository has no architectural or integration guide beyond social-login setup notes and `COPILOT_INSTRUCTIONS.md`.

**Claims that cannot be made from this evidence:** number of users served, AI requests processed in production, revenue or tokens sold, latency or uptime figures, cost savings, conversion impact, or team-leadership scope.

---

## 14. CV Achievement Candidates

> **See §21.2 for the recommended CV block** — context line plus three bullets, written for a contract engagement with no production metrics. The bullets below are the raw pool to draw from and adapt.

**High-confidence, quantified** (all quantities are code/Git-verified)

- Designed and shipped the full AI layer of a commercial CV platform — eight generative features on OpenAI's Responses API with strict JSON-schema outputs — contributing 176 commits and ~18,000 lines across 35 merged pull requests in 11 weeks.
- Built a token-credit billing system from scratch for a Laravel 11 SaaS product: per-user wallet, transactional charge-with-rollback under row-level locking, pre-flight cost quotation and nine new database tables across 35 migrations.
- Integrated Stripe end to end — tiered Checkout top-ups, signature-verified webhooks and an idempotent fulfilment service guarding on both event and session identifiers — eliminating double-credit risk across two independent fulfilment paths.
- Authored a 1,750-line AI optimisation service and a per-request cost-audit pipeline recording provider fee, tokens charged and margin for every one of ten individually priced AI features.
- Independently delivered a React 19 / TypeScript console (139 commits, sole author, 16,000+ lines) exercising every AI, pricing and payment flow, used to validate the backend before front-end integration.

**High-confidence, non-quantified**

- Made generative-AI features commercially viable by pricing on exact provider-measured input tokens plus a bounded output ceiling, so users receive a firm quote before any spend is committed.
- Designed a database-driven AI control plane letting the business change models, prices, margin, prompts and usage limits in production with no code deploy.
- Solved LLM structured-output reliability for nested, identifier-bearing JSON by combining schema enforcement, runtime prompt constraints and a defensive recovery parser.
- Re-architected AI result persistence so optimised CVs are written as new records that inherit every un-optimised field from the original, making the feature non-destructive.
- Closed a commercial-data exposure gap by putting internal cost and margin detail behind an opt-in debug header.
- Delivered within an established agency codebase, following documented house conventions for routing, responses, validation and migrations, with OpenAPI documentation maintained for every endpoint.

**Requires confirmation before use** (do not state without the data)

- Any claim of production volume, user numbers, revenue, token sales, latency or uptime.
- "Reduced AI costs by X%" — the mechanism is verified, the outcome is not measured.
- Any claim of leading or mentoring — no supporting evidence in the repository.

---

## 15. Portfolio Case Study Material

**Overview.** Scriberly is a consumer SaaS CV and cover-letter builder with AI optimisation (§2). The brief: make the AI features sellable.

**Challenge.** Generative AI costs real money per request, and the cost is unknown until after the request completes. A consumer product cannot quote "about £0.03, we'll tell you after". The platform needed a pricing model users could trust, margin the business could rely on, and controls that could be retuned without shipping code.

**Role.** Backend/full-stack engineer on contract, owning the AI and monetisation subsystem end to end — design, schema, implementation, API documentation and a purpose-built demo console — inside an existing agency team of five to six engineers using a Bitbucket PR workflow.

**Solution, in four parts.** A quotation engine (§5A step 1); a transactionally charged token wallet (§5A steps 3–5); Stripe top-ups with idempotent fulfilment (§5B); and a database-backed control plane plus per-request cost audit (§5C, §8 #4). Architecture in §4, stack in §6.

**Interesting engineering problems.** §7 and §9 — in short: quoting a non-deterministic cost; charging before generating without stealing from the user; forcing an LLM to return nested JSON that preserves every identifier; keeping a payment idempotent across two fulfilment paths; separating public from internal responses in one codebase.

**Results.** The platform moved from having no AI monetisation in the codebase to a complete, auditable, runtime-tunable paid-AI path, merged to the main branch. *Outcome metrics (users, requests, revenue) were not accessible and are not claimed.*

**What I would improve.** Automated tests around wallet debits and payment idempotency, before anything else. Decomposing the AI services by concern rather than by file size. Moving generation onto a queue so a slow LLM call cannot hold an HTTP worker and a database transaction. Versioning prompts in Git with a database overlay. Reconciling the CORS environment-variable naming mismatch (§11).

## 16. Interview Story Bank

Six of these stories are the problems documented in §9 and the decisions in §8; they are listed here with the angle each one answers, rather than restated.

| Story | Interview theme | Where the detail is | Headline result |
|---|---|---|---|
| **S1 — Pricing an unpriceable operation** | Complex problem solving, commercial thinking | §8 #1, §7 | Exact input-token count from the provider + a bounded output ceiling + margin = a firm quote before the user commits |
| **S2 — Charge-then-generate** | Correctness under concurrency and failure | §8 #2, §5 step 3–5 | Transaction + `lockForUpdate()` + rollback: no free generation, no charge for a failure, no double-spend |
| **S3 — "It only rewrote the first job"** | Production debugging, working with non-deterministic systems | §9.1 | Three layers — strict JSON schema, runtime-enumerated IDs in the prompt, recovery parser — made multi-entry optimisation shippable |
| **S4 — Payment idempotency** | Protecting revenue, integration design | §9.3 | One fulfilment service guarding on event **and** session ID; safe to retry from webhook or success page |
| **S5 — The response was leaking our margin** | Judgement, security awareness | §9.5 | Opt-in `X-AI-Debug` header; one codebase serves a public and an internal response |
| **S6 — Usage pricing → fixed pricing** | Technical trade-offs, responding to the business | §8 #3 | Kept the usage computation for accounting, charged a fixed price for clarity; margin per request is no longer guaranteed — an honest trade-off to discuss |
| **S7 — Optimisation was dropping users' data** | Ownership, data-design care | §9.2 | Nine commits in one day backfilling every un-optimised field; the original CV is never mutated |

**S8 — Working inside someone else's conventions.** Joined a 1,100-commit codebase with five prior contributors and a written conventions document (`COPILOT_INSTRUCTIONS.md`) prescribing route shapes, response envelopes, validation style and migration naming. Followed all of it, maintained Swagger annotations for every new endpoint, and used the team's per-developer branch and PR workflow for all 35 PRs — work merged through `staging` to `master` without architectural conflict.

**S9 — A weakness to own.** Asked what he would do differently: ship billing with tests. PHPUnit is installed, the Bitbucket Pipelines job is entirely commented out, and the project's documented position is that no automated tests are used. Wallet debits and payment idempotency are exactly the code that should not rely on manual verification — part of why the `demo-ii` console exists, though a harness is not a substitute for tests.

## 17. Technical Article and Blog Opportunities

| Proposed title | Core engineering problem | Why it's interesting | Must anonymise | Must NOT publish |
|---|---|---|---|---|
| **Quoting a price for something you can't measure yet: billing for LLM calls** | Per-request cost is unknown before the call, but consumers need a price up front | Nearly every team shipping paid AI hits this and reaches for post-hoc billing or flat subscriptions; the input-tokens + bounded-ceiling approach is concrete and generalisable | Client name, feature names | The pricing table, margin %, conversion ratio, per-million costs |
| **Charge before you generate: transactional credits around a third-party API** | Ordering a debit against an external call that can fail or hang | The transaction + row-lock + rollback pattern, and the real cost of holding a transaction across network I/O (queueing as the alternative) | Schema names | — (pattern is generic) |
| **Three layers of defence for structured LLM output** | A model returning *valid* JSON that is nonetheless wrong — entries skipped, IDs invented, output truncated | Schema alone is not enough for nested, identifier-bearing objects; shows why all three layers are needed | Domain field names | Production prompt text |
| **Idempotent payment fulfilment across two paths** | Webhook + success-page fulfilment, retries, duplicate events | Keying idempotency on two identifiers, the ledger-as-guard pattern, and why the receipt email belongs outside the transaction | — | Internal table names |
| **Put your business levers in the database** | Every price change becoming a deploy | The config-fallback pattern, its query cost, the caching decision, and the genuine downside: prompts living outside version control | Actual values | Prices, margins, prompts |

## 18. Professional Content Opportunities (LinkedIn / X)

Each is anchored to something that actually happened — the anchor is what makes it credible.

| Post angle | Anchor in the evidence |
|---|---|
| "The hardest part of shipping AI features wasn't the AI" — it was pricing a non-deterministic cost before the user commits | The quotation engine (§8 #1) |
| "Charge first or generate first?" — the ordering problem and how a transaction with a row lock resolves it | `chargeUserTokens` inside a transaction (§9, §8 #2) |
| "The model returned perfect JSON. It was still wrong." | The "only rewrote the first employment" bug (§9.1) |
| "Nine commits in one day, each adding one field" — discovering that AI optimisation was quietly dropping users' own data | The 2 Jan 2026 commit run (§9.2) |
| "We shipped usage-based pricing. Two months later we shipped fixed prices." | `ai_feature_prices`, Jan 2026 (§8 #3) |
| "Your API response was leaking your margin" | The `X-AI-Debug` header (§9.5) |
| "Why I built a second front end nobody will ever use" — the case for a disposable console that exercises every backend path | `demo-ii`, 139 commits, sole author |
| "Instrument the economics before you tune them" | `ai_request_stats` landing in week three |
| "Splitting a class because it got long is not refactoring" | `Service` → `ServiceContinuation` → `ServiceContinuationTwo` (§8 #8) |

*Caution for all of the above:* say "a CV platform" rather than naming the client unless permission is granted; never quote prompt text, prices or margin figures.

## 19. Private vs Public-Safe Information

**PRIVATE — internal record only**
- Client and agency identities (Fifteen, `scriberly.co.uk`, `*.fifteen.dev` hosts) unless permission to name them is obtained.
- All pricing economics: the 15% default margin, the 1,000 AI-tokens-per-Scriberly-token ratio, the ten per-feature fixed prices, the four top-up pack sizes, and the per-million model costs.
- System-prompt text in `ai_system_instructions` and the service classes.
- Internal table, column and endpoint names; the `X-AI-Debug` header name.
- Colleague names and the authorship distribution in §3.
- Repository contents, commit messages and branch names.
- The security weaknesses listed in §11 — useful for interview candour, not for publication.

**PUBLIC-SAFE — portfolio, LinkedIn, articles, CV**
- The architectural patterns, described generically: pre-flight token quotation, bounded output ceilings, transactional charge-with-rollback, idempotent payment fulfilment, database-driven configuration, three-layer structured-output defence, non-destructive AI persistence.
- Technology names: Laravel 11, PHP 8.3, MySQL, OpenAI Responses API, Stripe Checkout/Webhooks/Billing Portal, Sanctum, React 19, TypeScript, Vite.
- Contribution volume: commits, pull requests, lines, migrations, duration — these are about Abraham's work, not the client's secrets.
- Feature descriptions at the level of "AI CV optimisation with ATS variants" — the product is publicly marketed.
- The lessons in §12.

*Uncertain items are classified private.* No credentials, API keys, connection strings or customer data appear anywhere in this dossier, and none were read beyond identifying variable names.

---

## 20. Evidence Gaps and Further Metrics Worth Collecting

| # | Missing evidence | Why it matters | Where it probably exists | Question to ask |
|---|---|---|---|---|
| 1 | Production `ai_request_stats` row count, date range and breakdown by `request_type` | Would convert the entire AI subsystem from "built" to "processed N requests over M months" — the single highest-value gap | Production MySQL, `ai_request_stats` | "Can I get a read-only count of `ai_request_stats` by `request_type` and month?" |
| 2 | Actual-versus-charged totals (`total_fee` vs `token_charged_user_fee`) | Would evidence that the margin model worked in practice | Same table | "What did the aggregate provider spend versus tokens charged look like?" |
| 3 | `payment_transactions` count and total `amount_paid` | Would evidence that the Stripe integration processed real revenue | Production MySQL / Stripe dashboard | "How many top-ups have been fulfilled since launch?" |
| 4 | Count of users with a `scriberly_tokens` wallet and with non-zero `lifetime_spent` | Adoption of the paid AI features | Production MySQL | "How many users have spent tokens?" |
| 5 | Exact production launch date | The contributor confirms the product is now live, but not the date; a date would let you say "shipped in v1, live since X" | The former manager who made the referral; the client; the public site | "When did v1 go live?" — worth one message to the referrer |
| 6 | AI request latency and OpenAI failure/retry rates | Would support reliability claims | Application logs (`openai.responses` entries carry `duration_ms`) | "Can I get p50/p95 duration from the `openai.responses` log entries?" |
| 7 | Whether the `demo-ii` console was used by anyone besides Abraham | Would upgrade it from a personal harness to a team tool | The team; Abraham's own recollection | "Did QA or the front-end team use the demo console?" |
| 8 | Before/after on manual CV editing time, or any user-facing outcome | Would give the only genuine business-impact number in the dossier | Product analytics, user research, HubSpot | "Is there any measurement of what AI optimisation did for users?" |
| 9 | Production CORS, queue and cache configuration | Would resolve whether the issues noted in §11 are real in production | Production `.env` / deployment config | "What are the production CORS origins and queue driver?" |
| 10 | Code-review feedback on the PRs | Would evidence collaboration and code quality from a third party | Bitbucket PR comments on #402–#452 | — |
| 11 | Whether tests were ever added to this subsystem after January 2026 | Affects how the testing weakness should be framed | Current `master` | — |

**None of these gaps has been filled with an estimate anywhere in this dossier.**

**Important qualification on gaps 1–4, 6 and 9:** these are not gaps the contributor can close. Repository and database access were revoked when the contract ended (§2A), and any production figures now reflect a period after his involvement — so they would not be attributable to him even if obtained. Gaps 5, 7, 10 and 11 are the only ones realistically worth pursuing, and only gap 5 (launch date) would change how the project is presented. **Do not treat the rest as homework; treat them as permanently out of reach and present the project accordingly (§21).**

---

## 21. Presenting This Project Without Production Metrics

This project has no usage, revenue or performance figures and never will, because the contract ended before launch and access was revoked (§2A). That constraint is normal for contract work and it is not a weakness — but it does mean the project has to be presented on a different axis from a "I grew conversion 23%" story. The rule is simple:

> **When you cannot show what the system did, show what the mandate was, what you built, and that it shipped.**

Hiring managers reading a contract engagement already expect the first two. The third is the one most contractors fail to evidence, and you can: 174 of your commits are merged into `master`, and the product is live.

### 21.1 The four substitutes for production metrics

| Proof type | What it evidences | What you have |
|---|---|---|
| **Mandate** | That a company had a problem they could not solve internally and chose you for it | Brought in specifically to integrate AI and payments; neither existed in the codebase before you and both were authored entirely by you |
| **Scope and complexity** | The size and difficulty of what you built | 8 AI operations, 9 new tables, 35 migrations, ~7,000 lines of subsystem code, 35 merged PRs, 11 weeks |
| **Engineering judgement** | That you made defensible decisions, not just working code | The eight documented decisions in §8 and five problems in §9 — each with a trade-off you can argue either side of |
| **Delivery** | That it was accepted, merged and shipped | 174 commits on `master`; contract completed and paid; product subsequently went live |

Use all four. Any one alone reads thin; together they are a complete account of a professional engagement.

### 21.2 CV — how to write it

Give this project a **context line** plus two or three bullets. The context line does work that bullets cannot: it tells the reader this was scoped contract work, so they stop looking for tenure or runtime outcomes.

> **Backend / AI Engineer (Contract)** — Scriberly (CV & cover-letter SaaS) · Nov 2025 – Jan 2026
> *Engaged on referral to deliver the AI and payments capability for v1 of the platform; now live.*
>
> - Built the platform's entire generative-AI layer — 8 AI operations on OpenAI's Responses API with strict JSON-schema outputs — and the token-credit billing system that made them sellable: per-user wallet, transactional charge-with-rollback under row-level locking, and pre-flight cost quotation so users see a firm price before any spend.
> - Integrated Stripe end to end (tiered Checkout top-ups, signature-verified webhooks, idempotent fulfilment across two independent paths) and a runtime pricing control plane letting the business change models, prices, margin and prompts without a deploy — 9 new tables across 35 migrations.
> - Delivered 35 merged pull requests in 11 weeks, plus a React 19 reference front end (sole author, 139 commits) built so the remote front-end developer could integrate the API from working code.

Why this works without metrics: every bullet opens with a verb and closes with either a **capability the business did not have before** ("made them sellable", "without a deploy") or a **quantity of work** (8, 9, 35, 139). Neither requires production data.

**Do not** write: "Worked on AI features for a CV platform." **Do not** invent: "reduced costs by 40%", "served 10,000 users". If pressed in an interview on numbers you cannot produce, the honest answer is strong (§21.5).

### 21.3 Portfolio — how to structure it

Lead with the **problem**, not the stack. The most interesting thing about this project to another engineer is not that it uses Laravel and OpenAI; it is that *you had to put a price on a non-deterministic operation before it ran*.

1. **One-line hook:** "How do you charge a user for an AI request when you don't know what it will cost until after it finishes?"
2. **Context:** an existing CV platform, a startup, no in-house AI or payments capability, a fixed-scope contract to build both.
3. **Constraint:** the price shown to the user has to be firm *before* they commit, and the business has to stay margin-positive.
4. **Solution:** the quotation engine → wallet → generation → audit chain (§5A), with the architecture diagram from §4.
5. **Two deep dives:** pick the structured-output problem (§9.1) and payment idempotency (§9.3). These are the two a senior engineer will actually read.
6. **What you'd do differently:** §12 items 7 and 8. Including this is what separates a portfolio piece from a brochure.
7. **Honest close:** "Delivered as v1 under contract; the product went live after my engagement ended, so I have no production figures to share."

That last line costs you nothing and buys credibility for everything above it.

### 21.4 Public writing — what is safe

Your strongest public material is **the patterns, not the project** (§17, §18). Write "how to bill for LLM calls" with your own code examples reconstructed generically; do not write "how Scriberly bills for LLM calls". Specifically:

- **Safe:** the pre-flight quotation pattern, charge-then-generate with rollback, three-layer structured-output defence, idempotent fulfilment, database-driven config. All are general engineering patterns you can illustrate with your own simplified code.
- **Not safe without permission:** the client or agency name, prompt text, actual prices, margin percentage, pack sizes, internal table and header names (§19).
- **Judgement call:** whether to name the client at all. Default to "a UK CV-building SaaS" until you have explicit permission. If you want to name it — and it is a reasonable thing to ask — the person to ask is the former manager who referred you.

### 21.5 Interview — the script for "what was the impact?"

You will be asked. Do not hedge or pad; answer in three moves.

> *"It was a fixed-scope contract for version one, so I don't have production numbers — my access ended at delivery and the product went live afterwards. What I can tell you is what they didn't have before and what they had after: no AI integration and no payments at all, and at the end, eight AI features, a token-credit billing system, Stripe top-ups with idempotent fulfilment, and pricing the business could change without a deploy. All of it merged to master. If it's useful I can walk you through the hardest part, which was pricing an LLM call before you know what it costs."*

Three things are happening in that answer: you resolve the metrics question immediately and without embarrassment, you replace it with a before/after that is just as concrete, and you hand the interviewer a better question to ask next. Then let §9 carry the conversation.

**The one thing to avoid:** apologising for the absence of data, or volunteering it before you are asked. Contract engineers routinely ship and leave. Treat it as the ordinary professional fact it is.

---

## Appendix A — Evidence Notes and Reproduction

All commands were run read-only inside the three repositories.

```
# Contributor identity and volume (api)
git log --all --author=abramseyakid --no-merges --oneline | wc -l          → 176
git log --all --author=abramseyakid --no-merges --numstat --format='' \
  | awk '{a+=$1;d+=$2} END {print a,d}'                                     → 18072 7033
git log origin/master --author=abramseyakid --no-merges --oneline | wc -l   → 174
git log origin/master --no-merges --oneline | wc -l                         → 692
git log --all --merges --format='%s' \
  | grep -oE 'Merged in dev/abraham \(pull request #[0-9]+\)' | sort -u | wc -l → 35
git log --all --author=abramseyakid --no-merges --diff-filter=A --name-only \
  --format='' | grep '^database/migrations/' | sort -u | wc -l              → 35
git log --all --author=abramseyakid --no-merges --diff-filter=A --name-only \
  --format='' | grep '^app/' | sort -u | wc -l                              → 33

# Engagement window
git log --all --author=abramseyakid --no-merges --format='%ad' --date=short \
  | sort | sed -n '1p;$p'                                                   → 2025-11-05 … 2026-01-21

# Demo console
cd demo-ii && git shortlog -sne --all     → 139  AbrahamNobleOX  (only author)
cd demo-ii && git log --no-merges --numstat --format='' \
  | awk '{a+=$1;d+=$2} END {print a,d}'                                     → 16203 3150

# Front end (production Next.js app)
cd frontend && git shortlog -sne --all    → no commits by this contributor
```

**Derived figures and their calculations**
- *Share of main-branch commits:* 174 ÷ 692 = 25.14%. Counts non-merge commits reachable from `origin/master` only. Measures commit count, not scope or value.
- *~6,990 lines of primary-authored application code:* the sum of `wc -l` over the 18 files in §3 for which Git shows Abraham as the creating and dominant author. Includes OpenAPI docblocks and comments. Does not include the 35 migrations or `demo-ii`.
- *Engagement length:* 5 Nov 2025 to 21 Jan 2026 = 11 weeks of commit activity. Contract start and end dates are not evidenced.

**Scope and limitations**
1. No production database, staging database, log store, APM or analytics platform was reachable; **no production data of any kind was read**.
2. `origin/*` refs reflect a fetch dated 12 February 2026; work in the repository after that date is not visible here.
3. Commit authorship proves who wrote a change, not who designed it; pairing, specification and review by others would not appear.
4. Line counts include generated OpenAPI JSON and documentation.
5. Four Git identities map to this contributor (`AbrahamNobleOX`, `Abraham Akintola`, `abramseyakid`, all on `abramseyakid@gmail.com`); all queries matched on the shared email.
6. No credentials, keys, connection strings or personal data from any source are reproduced in this dossier.

---

*Compiled 15 October 2026 from repository and Git evidence only. Every quantitative claim above is reproducible with the commands in Appendix A.*
