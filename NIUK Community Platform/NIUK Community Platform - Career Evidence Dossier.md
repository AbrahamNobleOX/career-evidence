# NIUK Community Platform — Career Evidence Dossier

**Audit date:** 17 October 2026
**Repository audited:** `/Users/abrahamakintola/Documents/Coding/portfolio-projects/niuk`
**Sub-repositories:** `anonapp-project` (55 commits, Feb–Apr 2024) and `demoapp-project` (85 commits, Feb–Dec 2024). The second is the continuation of the first; `demoapp-project` is the authoritative codebase.
**Author of 100% of commits:** Akintola Abraham (`abramseyakid@gmail.com`) — verified via `git shortlog -sne --all`.

---

## 0. Important framing: this is a portfolio project, not a production system

This section exists because it changes how every claim in this dossier must be used.

The platform was conceived as a real product for an active Nigerians-in-the-UK community on Twitter/X, with the intention of approaching the community's organisers for adoption. That approach **never happened** — at the point the build reached a demonstrable state, the community was going through a leadership dispute, and the owner chose not to pitch it. The project was then deliberately repositioned as a **portfolio and skills-demonstration project**.

Consequences for this audit:

- **There is no production database.** The active `MONGO_URI` points at `mongodb://localhost:27017/demoapp-project`; a commented-out Atlas URI exists but the live connection target is local. A connection attempt during this audit was refused (`ECONNREFUSED 127.0.0.1:27017`) — no local Mongo instance is running.
- **There are no real users, no real payments, no usage metrics, and no uptime history.** Stripe and GoCardless are both wired against **test/sandbox** credentials.
- **Therefore no adoption, volume, revenue, reliability or business-impact statistic is claimed anywhere in this document.** Every quantitative figure below is a *codebase and version-control* measurement (lines of code, endpoints, models, commit dates), which is verifiable by anyone who opens the repository.

**How to describe this project professionally, in one line:** *"A full-stack community-management platform built end to end as a self-directed project, including authentication, role-based access control, elections/voting, event and group management, and dual payment-provider integration — built to production patterns, deployed publicly, but not launched to real users."*

Do **not** describe it as "used by the Nigerian community in the UK", "live platform serving X members", or anything implying adoption. The engineering is real and defensible; the adoption is not, and a competent interviewer will find that out in one question.

---

## 1. Executive summary

NIUK is a MERN-stack community-management platform for a diaspora community organisation. It gives community administrators the tools a Twitter/X-based community outgrows: a member register, role-based permissions, scheduled Twitter Spaces and physical events, sub-groups, paid membership and event ticketing, and — the most substantial feature — a **structured online election system** with multiple concurrent polls, candidate management, verified-voter gating, single-transferable-vote semantics (one vote per voter per poll, changeable until close), and results hidden from ordinary members until a poll closes.

**Scale of the build (all VERIFIED by file inspection):** ~4,150 lines of backend JavaScript (7 controllers, 8 Mongoose models, 7 routers, 59 registered route handlers plus a raw-body Stripe webhook), ~10,650 lines of React/JSX across 79 source files, 24 Redux Toolkit slice/service files, and a Docker/Docker-Compose deployment path. Built solo over **10 months, 19 February 2024 – 22 December 2024**, across 85 commits in the final repository.

**Most technically significant aspects:**

1. A hand-rolled authentication stack with **six sign-in paths** (email/password, email verification, password reset, one-time login code, Google OAuth, Twitter/X OAuth via Passport) plus **device-fingerprint-triggered 2FA** — unknown user-agent forces an emailed, encrypted six-digit code before a session is issued.
2. An **election/poll domain model** with non-trivial invariants: a candidate may appear in only one poll per election, so adding a candidate atomically removes them from sibling polls; re-voting moves the vote and decrements the previous candidate's count; vote counts are concealed from `user`-role accounts while a poll is open.
3. **Optimistic-concurrency handling**: the `addCandidates` handler catches Mongoose `VersionError` and retries with freshly-loaded state (3 attempts) — a genuine response to a concurrent-write failure, not boilerplate.
4. **A provider migration under failure**: the email layer was built on Gmail OAuth2 + Nodemailer, then re-implemented against Brevo's transactional API when mail stopped sending on the deployed host — both implementations survive in `helpers/email.helper.js`, which is unusually good evidence of a real debugging episode.
5. **Two payment providers** (Stripe Checkout + subscriptions with a 7-day trial, webhook-driven fulfilment; GoCardless Direct Debit billing requests) behind one application payment model.

**Clearest career value:** this project evidences independent, end-to-end ownership — data modelling, API design, auth and authorisation, third-party integration, state management, containerisation and deployment — on a domain (community elections) with real correctness constraints.

---

## 2. Project overview and business problem

**The problem (INFERRED from the owner's account + STRONGLY SUPPORTED by the feature set):** A diaspora community operating inside Twitter/X Communities and Spaces has no durable infrastructure. Membership is implicit, leadership elections are informal, Spaces and events are announced and then lost in the timeline, sub-groups are unmanaged, and there is no way to collect dues or event fees. The platform's feature set maps one-to-one onto those gaps — **Spaces**, **Events**, **Groups**, **Elections/Polls**, **Users with roles**, **Payments** — which is strong circumstantial evidence for that motivation even independent of the owner's own account.

**Intended users (INFERRED from role model):** four roles are defined in `models/user.model.js` (`user`, `author`, `admin`, plus a `suspended` state enforced in middleware). Ordinary members browse, attend and vote. `author` accounts are community organisers who create elections, polls, events, groups and Spaces and verify voters. `admin` accounts additionally delete users and change roles.

**Why it did not reach production:** the community was in a leadership dispute at the time the build became demonstrable; the owner declined to pitch it and repositioned the work as a portfolio piece. (Source: owner's direct account. No repository evidence contradicts or corroborates this.)

**The Twitter/X branding history is visible in version control:** commit `1c6ff9c`, *"removed niuk tags"* (2 April 2024), strips NIUK branding across 15 files — README, Jumbotron, Footer, Menu, Login, Register, `package.json` and the server. The app was subsequently branded `DemoApp` (`COMPANY_NAME=DemoApp`). This confirms the pivot from community-specific product to neutral demonstrable artefact, and dates it precisely.

---

## 3. My role and contribution

**VERIFIED: sole author.** `git shortlog -sne --all` across `demoapp-project` returns one contributor for all 85 commits; `anonapp-project` likewise. There is no evidence of any other contributor, no PR workflow, no co-author trailers. Every line of this system is attributable to Abraham Akintola.

This means the usual caution about over-attribution does not apply here — but the corresponding caution does: **solo project, no code review, no team process.** That is honest and perfectly respectable for a portfolio project; do not imply otherwise.

Work performed, all VERIFIED by code and commit history:

| Area | Evidence |
|---|---|
| Data modelling | 8 Mongoose schemas with validation, indexes via `unique`, soft-delete flags, audit trails (`createdBy`, `updatedBy[]` with per-edit user + timestamp) |
| Authentication | `controllers/auth.controller.js`, 1,409 lines — register, login, logout, verify, forgot/reset/change password, login-code 2FA, Google signup + login, Twitter OAuth |
| Authorisation | `middleware/auth.middleware.js` — `protect`, `adminOnly`, `authorOnly`, `verifiedOnly`; mirrored client-side in `ConditionalRendering.jsx` |
| REST API design | 7 routers, 59 route handlers + 1 raw-body webhook |
| Elections domain | `poll.controller.js` (473 lines) + `election.controller.js` — candidate assignment, vote casting, vote transfer, concurrency retry |
| Payments | `payment.controller.js` (430 lines) — Stripe PaymentIntents, Checkout Sessions, subscriptions with trial, Billing Portal, signed webhook; GoCardless billing requests/flows |
| Email | `helpers/email.helper.js` — Handlebars templates, Gmail OAuth2/Nodemailer implementation and Brevo API implementation |
| Frontend | 79 React files, ~10,650 LOC: routing, protected routes, profile-completeness gate, search/filter, pagination, modals, toasts, dashboards |
| State management | Redux Toolkit store with 16 reducers; per-domain `*Service.js` (axios) + `*Slice.js` (async thunks) + `*FilterSlice.js` (client-side search) |
| Build & deploy | Vite build piped into the Express `dist/` served by the same Node process; Dockerfile (node:20-alpine); docker-compose with a Mongo service; deployed to Render |

---

## 4. System architecture

```
                         ┌─────────────────────────────────────┐
                         │  Browser (React 18 SPA, Vite build) │
                         │  react-router-dom v6                │
                         │  Redux Toolkit: 16 reducers         │
                         │   auth · election · poll · space ·  │
                         │   event · group · payment · email   │
                         │   + per-domain client-side filters  │
                         │  Bootstrap 5 / react-bootstrap / antd│
                         └───────────────┬─────────────────────┘
                                         │ axios, withCredentials:true
                                         │ (HTTP-only JWT cookie)
                                         ▼
      ┌───────────────────────────────────────────────────────────────┐
      │                Express 4 (ESM, Node 20) — single process      │
      │                                                               │
      │  morgan · cookie-parser · express-session · passport          │
      │  cors(allow-list: localhost:5173 + FRONTEND_URL, credentials) │
      │                                                               │
      │  POST /api/payment/webhook  ← express.raw()  (signature check)│
      │  ── express.json() below this line ──                         │
      │                                                               │
      │  /api/auth      protect │ adminOnly │ authorOnly              │
      │  /api/election  /api/poll  /api/space  /api/event  /api/group │
      │  /api/payment                                                 │
      │                                                               │
      │  errorHandler middleware (stack in dev only)                  │
      │  SPA fallback: non-asset paths → dist/index.html (no-cache)   │
      └───┬───────────────┬──────────────┬─────────────┬──────────────┘
          │               │              │             │
          ▼               ▼              ▼             ▼
   ┌────────────┐  ┌────────────┐  ┌──────────┐  ┌──────────────┐
   │  MongoDB   │  │  Stripe    │  │GoCardless│  │ Email        │
   │ (Mongoose) │  │ Intents ·  │  │ Billing  │  │ Brevo API    │
   │            │  │ Checkout · │  │ Requests │  │ (current)    │
   │ User Token │  │ Subs+trial │  │ + Flows  │  │ Gmail OAuth2 │
   │ Election   │  │ Portal ·   │  │ (sandbox)│  │ + Nodemailer │
   │ Poll Event │  │ Webhook    │  └──────────┘  │ (legacy)     │
   │ Group Space│  └────────────┘                │ Handlebars   │
   │ Payment    │   ┌──────────────┐             └──────────────┘
   └────────────┘   │ Google OAuth │
                    │ Twitter OAuth│  (passport-twitter)
                    └──────────────┘

   Deployment: Vite `build` copies dist/ to server root → Node serves API + SPA
               from one origin. Dockerfile (node:20-alpine, EXPOSE 3002).
               docker-compose: app + mongo with named volume.
               Public deploy: https://niuk-project.onrender.com (HTTP 200 on
               17 Oct 2026 — VERIFIED).
```

**Data model relationships:** `User` is the hub. `Election 1─n Poll`; `Poll.options[]` embeds candidates (denormalised `candidateName`/`Email`/`Photo` snapshots plus `candidateUserId` ref) and each option embeds a `vote[]` array of `{voter, votedAt}` with a maintained `voteCount`. `Payment` references `User` and carries a Stripe `priceId` and `reference` (checkout session id, unique). `Space`, `Event`, `Group` each carry `createdBy`, a slug, a unique `url`, and an `updatedBy[]` audit array. `Token` holds short-lived verification/reset/login tokens with explicit `expiresAt`.

---

## 5. Main workflows

**Registration → verification → first login.** Register validates and hashes the password (bcrypt, pre-save hook, 10 rounds), stores the requesting user-agent, issues an HTTP-only JWT cookie (`sameSite: none`, `secure: true`, 1-day expiry; JWT itself 7-day). A verification token is hashed (SHA-256) before storage and emailed as a link. Login from a **new device** is refused: the system generates a six-digit code, encrypts it with Cryptr before persisting, stores it with a 60-minute expiry, and the user must complete `/sendLoginCode/:email` → `/loginWithCode/:email`.

**Profile gate.** `CheckIfProfileIsFilled` blocks every dashboard route until address fields are complete — a deliberate product decision (a UK community register needs postal addresses for dues and verification), implemented as a routing guard rather than scattered checks.

**Election lifecycle.** An `author` creates an Election (title, slug, start/end datetime) → creates one or more Polls under it (each inherits the election window) → adds candidates by selecting existing users. Candidates are exclusive across polls in the same election: adding a user already in a sibling poll moves them, and the response distinguishes *"Candidates Moved to this Poll"* from *"Candidates Added Successfully"*. Admins verify voters individually (`isManuallyVerified`). Voting is open while `now <= endDateTime`; a voter may change their vote, which removes it from the previous candidate and decrements that count. Ordinary `user`-role accounts cannot see `voteCount` until the poll closes; authors/admins always can.

**Payments.** Client requests a Checkout Session (`mode` = payment or subscription; subscriptions get `trial_period_days: 7`) → Stripe-hosted checkout → redirect to `/paymentsuccess/success/{CHECKOUT_SESSION_ID}` → client saves payment details, with the product name resolved server-side from the Stripe price/product API rather than trusted from the client. Stripe's webhook (raw body, signature-verified) independently fulfils `checkout.session.completed`, updating amount/mode/status and computing trial and 1-year expiry dates with moment.js. A Billing Portal session endpoint lets subscribers self-manage.

---

## 6. Technology stack

**Backend:** Node 20, Express 4 (ESM modules), MongoDB + Mongoose 8, JWT (`jsonwebtoken`), bcryptjs, Cryptr, `crypto` SHA-256, Passport + passport-twitter + passport-google-oauth20, google-auth-library, googleapis, Stripe SDK 14, gocardless-nodejs 3, Nodemailer + nodemailer-express-handlebars, @getbrevo/brevo, Handlebars, slugify, moment / moment-timezone, ua-parser-js, morgan, cors, cookie-parser, express-session, express-async-handler, dotenv.

**Frontend:** React 18, Vite 5, react-router-dom 6, Redux Toolkit 2 + react-redux 9, axios, Bootstrap 5 + react-bootstrap, Ant Design 5, react-icons, react-toastify, react-confirm-alert, react-paginate, react-country-region-selector, @stripe/react-stripe-js, @gocardless/react-dropin, @react-oauth/google, add-to-calendar-button-react, react-confetti, moment, lodash, ESLint.

**Infra:** Docker (node:20-alpine), Docker Compose (app + mongo, named volume), Render (public deploy), Cloudinary (client-side image upload preset), Google Cloud project for OAuth credentials.

---

## 7. Engineering complexity — what was actually hard, and why

**7.1 Cross-poll candidate exclusivity under concurrent writes.** The rule "a candidate belongs to exactly one poll within an election" cannot be enforced by a unique index because candidates live in an embedded array inside sibling documents. The implementation loads all polls for the election, scans each poll's `options[]` for the candidate, splices them out of whichever one holds them, saves that sibling, and only then pushes into the target poll — all inside a loop over a batch of candidate IDs. Difficulty: it is a multi-document mutation with no transaction, operating on embedded arrays, with per-candidate partial-success semantics.

**7.2 Mongoose `VersionError` retry.** Saving a document whose embedded array changed under you throws `VersionError`. The code catches it specifically, re-fetches the poll, decrements a retry budget of 3, and distinguishes it from other errors (which abort immediately). This is a real optimistic-concurrency control loop. *Honest caveat worth knowing before an interview:* the retry reassigns a `const`-bound variable, so in practice the retry path would throw a `TypeError`. The **design instinct is right and demonstrable**; the implementation has a bug. Mentioning it yourself, with the fix (`let`, plus re-applying the mutation after re-fetch, or moving to `$pull`/`$push` atomic operators or a transaction), is a *stronger* interview answer than hoping nobody reads it.

**7.3 Vote transfer semantics.** `castVote` finds whether the voter appears in *any* option's `vote[]`, filters them out, decrements that option's `voteCount`, then pushes to the new option and increments. Maintaining a denormalised counter alongside the authoritative array is a classic correctness trap (the two can diverge), and the code does keep them in step on the happy path.

**7.4 Device-fingerprint 2FA.** Parsing the user-agent, maintaining an allow-list per user, generating and *encrypting* (not just hashing) a six-digit code so it can be decrypted for comparison, and giving it an independent 60-minute expiry in a separate `Token` collection — a complete step-up-authentication flow built from primitives rather than pulled from a library.

**7.5 Webhook body-parser ordering.** Stripe signature verification requires the unparsed body. The webhook route is registered with `express.raw()` **before** `express.json()` is applied globally — a subtle ordering requirement that silently breaks signature verification if you get it wrong.

**7.6 Serving SPA and API from one origin.** The custom fallback middleware distinguishes asset extensions from client routes, sets anti-cache headers on `index.html`, and the Vite `build` script copies `dist/` into the server root. This removes the entire cross-origin cookie problem class in deployment while local dev still runs cross-origin (hence the CORS allow-list plus `sameSite: none; secure: true` cookies).

**7.7 Two payment providers with different mental models.** Stripe is session/webhook-driven and card-first; GoCardless is mandate/billing-request-driven and Direct-Debit-first, with a redirect flow of its own. Supporting both means two incompatible state machines feeding one `Payment` collection.

---

## 8. Architectural decisions and trade-offs

| Decision | Reasoning (evidence-based) | Trade-off | Status |
|---|---|---|---|
| **Embed candidates and votes inside `Poll.options[]`** rather than a separate `Vote` collection | Reads are poll-shaped: one query renders a whole ballot with counts | Document growth, no unique index on voter, multi-doc invariants must be hand-enforced, `VersionError` exposure | Workable at community scale; would not survive high-concurrency voting |
| **HTTP-only cookie JWT** rather than localStorage token | XSS cannot read the token; `secure` + `sameSite: none` set explicitly | Requires CORS credentials config and HTTPS; no refresh-token rotation; no server-side revocation | Sound choice, incompletely finished |
| **Denormalised `voteCount` alongside `vote[]`** | Avoids recomputing counts on every render | Two sources of truth that can diverge on a partial failure | Consistent on happy path |
| **Role-gating duplicated client and server** (`ConditionalRendering.jsx` ↔ `auth.middleware.js`) | Server is authoritative; client prevents dead-end UI | Two places to change a rule | Correct pattern |
| **Soft delete on Election/Poll** (`deleteStatus: "active"`), hard delete elsewhere | Elections are audit-sensitive | Inconsistent across the domain; `deletePoll` actually hard-deletes with the soft path commented out | Half-migrated — honest to call it so |
| **Gmail OAuth2 → Brevo** for transactional email | Mail failed on the deployed host (commit *"fix: email not sending"*, 21 Dec 2024) | Vendor lock-in to Brevo; dead Nodemailer code retained | Resolved by replacement |
| **Monolith serving API + SPA from one Node process** | Single deploy unit on Render; same-origin cookies | No independent scaling of static assets; build-time coupling | Right call for this scale |
| **Two payment providers** | Cards for one-off tickets, Direct Debit for recurring UK dues | Double integration surface; GoCardless never got past hard-coded demo values | Incomplete by design (abandoned before launch) |

---

## 9. Problems solved (interview-ready)

**9.1 — Email delivery failed in deployment.**
*Situation:* The auth system depends on email for verification, password reset and 2FA codes; the Gmail OAuth2 + Nodemailer implementation worked locally.
*Problem:* After deploying, email stopped sending (commit `fix: email not sending and added node serve react`, 21 December 2024 — the same commit that introduced same-origin serving, so the deploy environment is where it surfaced).
*Investigation:* Evidenced by the outcome rather than narrated in the repo — the Gmail path was kept intact (`sendEmaili`) while a parallel implementation was written.
*Solution:* Re-implemented email against the Brevo transactional API, pre-rendering Handlebars templates manually (read file → `Handlebars.compile` → inject context) instead of relying on the Nodemailer handlebars plugin, with ESM-safe `__dirname` resolution. The function signature is identical, so no call site changed.
*Result:* Email works through an API rather than SMTP — which also removes the SMTP-port restrictions typical of PaaS hosts.
*Evidence:* `helpers/email.helper.js` contains both implementations; commit message and date.
*Lesson to state:* keeping a provider-shaped function signature meant a provider swap touched one file. That is the reusable point.

**9.2 — Concurrent candidate assignment corrupting polls.**
*Situation:* Organisers assign candidates to polls, and the same candidate may be moved between polls.
*Problem:* Saving a poll whose embedded `options[]` had changed concurrently throws Mongoose `VersionError`; batch assignment made collisions likely.
*Solution:* A bounded retry loop (3 attempts) that re-fetches current state on `VersionError` and rethrows anything else, plus explicit cross-poll de-duplication before writing.
*Result:* Documented intent of the design is at-most-once assignment with a clear message distinguishing *moved* from *added*.
*Evidence:* `controllers/poll.controller.js`, `addCandidates`.
*Honest caveat:* the retry rebinds a `const`, so the retry branch would fail. Say this yourself; the fix is one keyword plus re-applying the mutation after re-fetch.

**9.3 — Stripe webhook signature verification vs. global JSON parsing.**
*Problem:* `express.json()` consumes the body; Stripe needs the raw bytes to verify the signature.
*Solution:* Register the webhook route with `express.raw({type: "application/json"})` above the global JSON parser in `index.js`.
*Result:* Signature verification succeeds and fulfilment runs independently of the browser redirect, so a user closing the tab does not lose the payment record.
*Evidence:* `index.js` middleware ordering; `webhook`/`fulfillCheckout` in `payment.controller.js`.

**9.4 — Role switching the author could not get right.**
*Evidence:* commit, 24 March 2024: *"After 7hours I couldn't figure out my desired switch role function"*, followed two days later by *"fixed sidebar null error bug"* and state-management commits for the dashboard sidebar.
*Why it matters:* this is a rare, genuinely honest artefact — a timestamped record of getting stuck on a role/state-propagation problem and resolving it by lifting role state into Redux rather than deriving it locally. Good material for a "tell me about something you struggled with" question, because it is documented rather than reconstructed.

---

## 10. Security, reliability and operational posture (candid)

Done well:
- Passwords hashed with bcrypt in a pre-save hook (never in a controller).
- Verification/reset tokens **hashed before storage**; login codes **encrypted** with an explicit expiry collection.
- JWT in an HTTP-only, `secure`, `sameSite: none` cookie — not localStorage.
- Server-side authorisation on every mutating route; client gating is additive, not load-bearing.
- Stripe webhook signature verification; product/price details resolved from Stripe, not trusted from the client.
- Suspended-account check inside `protect`, so suspension takes effect on the next request.
- Error handler suppresses stack traces outside development.
- Secrets externalised to `.env`, and `.env` is correctly git-ignored and **not tracked** (verified: `git ls-files | grep env` returns nothing).

Known weaknesses — record these for yourself, fix before showcasing, and do **not** publish:
1. **Hard-coded secrets in source:** the GoCardless sandbox access token and the Stripe webhook signing secret are literals in `payment.controller.js`; the `express-session` secret is a literal in `index.js`. They are sandbox/dev values, but the pattern is wrong and a reviewer will see it in ten seconds.
2. **GoCardless flow is hard-coded demo data** — the customer's name, email and Bradford address are the developer's own, baked into the handler. This is an unfinished spike, not a feature. It also means **the repository contains your own home address**; strip it before making anything public.
3. **`castVote` is behind `authorOnly`** in `routes/poll.route.js`, which contradicts the UI (members vote). A rules/routing mismatch worth fixing and worth understanding before an interviewer asks.
4. **No automated tests of any kind.** No test runner, no test files, no CI. The single biggest gap between this and a production-grade repo.
5. **No rate limiting** on login, login-code or password-reset endpoints; six-digit codes with a 60-minute window are brute-forceable without it.
6. **No vote-integrity constraint at the database level** — one vote per voter per poll is enforced only in application code.
7. **No logging/monitoring beyond `morgan` and `console.log`**; no structured logs, no error tracking, no health check.
8. **No refresh-token rotation or server-side session revocation**; logout clears the cookie but the JWT stays valid for its 7-day life.
9. **`express.static("public")` references a directory that does not exist** in the repo.

None of this is unusual for a solo pre-launch project. All of it is cheap to fix and each fix is itself a portfolio talking point.

---

## 11. Lessons learned (grounded in this project)

1. **Provider-shaped abstractions pay for themselves.** Swapping Gmail for Brevo touched exactly one file because the send function's signature was defined by the application's needs, not the vendor's SDK. The payment layer, by contrast, leaked Stripe concepts (`priceId`, `reference` = checkout session id) into the `Payment` model — so adding GoCardless meant a second, incompatible path rather than a second implementation of one interface. The same lesson, learned once and missed once, in the same codebase.
2. **Embedded arrays trade query simplicity for invariant pain.** Votes and candidates inside `Poll.options[]` made rendering trivial and correctness hard — no unique index on voter, hand-written cross-document exclusivity, and `VersionError` as a routine occurrence. A `Vote` collection with a compound unique index on `(pollId, voterId)` would have made the central correctness rule a database guarantee.
3. **Deployment surfaces a different class of bug than development.** Email, routing after authentication, and SPA/API origin handling all broke only once deployed (three December 2024 commits in a row). Deploying earlier and more often would have found them months earlier.
4. **Enforce rules on the server; mirror them on the client.** Done correctly here — and the one place the two disagree (`castVote` under `authorOnly`) is exactly where a bug lives.
5. **Tests are the thing that was skipped, and it shows.** Every bug identified in this audit — the `const` retry, the voting role mismatch, the half-migrated soft delete — is the kind a single integration test would have caught.
6. **Knowing when to stop is a real decision.** The pivot away from a community pitch, and the deliberate rebranding to a neutral demo, was judgement, not failure. It is defensible in an interview *as* judgement.

---

## 12. Verified career claim bank

Every claim below is traceable to the repository. Qualifications are mandatory, not optional.

1. **Designed and built a full-stack MERN community-management platform end to end as sole developer — 85 commits over 10 months (19 Feb – 22 Dec 2024), ~4,150 lines of Node/Express backend and ~10,650 lines of React frontend.**
   *Evidence level:* VERIFIED. *Evidence:* `git shortlog -sne --all` (single author), `git log` dates, `wc -l` over source directories.
   *Qualification:* Portfolio project; never launched to users.

2. **Implemented six authentication paths — email/password, email verification, password reset, one-time emailed login code, Google OAuth and Twitter/X OAuth — in a 1,409-line auth controller.**
   *Evidence level:* VERIFIED. *Evidence:* `controllers/auth.controller.js`, `routes/auth.route.js`.

3. **Built device-fingerprint step-up authentication: unknown user-agents are blocked and must complete an encrypted six-digit code with a 60-minute expiry before a session is issued.**
   *Evidence level:* VERIFIED. *Evidence:* `login`, `sendLoginCode`, `loginWithCode`; `ua-parser-js`; `Token` model.

4. **Designed a four-tier role-based access control model (`user`/`author`/`admin` + suspended) enforced server-side on every mutating endpoint across 59 route handlers and mirrored in client-side route and component guards.**
   *Evidence level:* VERIFIED. *Evidence:* `middleware/auth.middleware.js`, `routes/*.js`, `ConditionalRendering.jsx`, `ProtectPrivateRoute.jsx`.

5. **Built an online election system supporting multiple concurrent polls per election, cross-poll candidate exclusivity, changeable votes with count transfer, verified-voter gating, and results concealed from members until polls close.**
   *Evidence level:* VERIFIED. *Evidence:* `poll.controller.js`, `election.controller.js`, `components/Polls.jsx`.

6. **Handled optimistic-concurrency conflicts explicitly, catching Mongoose `VersionError` and retrying against freshly-loaded state with a bounded retry budget.**
   *Evidence level:* VERIFIED (as an implemented design). *Qualification:* the retry path contains a `const`-reassignment bug; present this as a design you can critique, not as proven-correct code.

7. **Integrated two payment providers behind one domain model: Stripe (PaymentIntents, hosted Checkout, subscriptions with a 7-day trial, Billing Portal, signature-verified webhooks with independent fulfilment) and GoCardless Direct Debit billing requests.**
   *Evidence level:* VERIFIED for the integration code. *Qualification:* **test/sandbox credentials only; no real payments ever processed.** The GoCardless path is a spike with hard-coded values.

8. **Migrated the transactional email layer from Gmail OAuth2 + Nodemailer to the Brevo API in response to a deployment-only delivery failure, without changing a single call site.**
   *Evidence level:* VERIFIED. *Evidence:* both implementations in `helpers/email.helper.js`; commit *"fix: email not sending"* (21 Dec 2024).

9. **Architected the frontend around Redux Toolkit with 16 reducers and a consistent per-domain service/slice/filter triad across 24 state files.**
   *Evidence level:* VERIFIED. *Evidence:* `redux/store.js`, `redux/features/*/`.

10. **Containerised the application (Docker, node:20-alpine; Docker Compose with a MongoDB service and named volume) and deployed it publicly to Render, serving the Vite-built SPA and the REST API from a single same-origin Node process.**
    *Evidence level:* VERIFIED. *Evidence:* `Dockerfile`, `docker-compose.yml`, SPA fallback middleware in `index.js`, Vite `build` script; `https://niuk-project.onrender.com` returned HTTP 200 on 17 Oct 2026.

11. **Modelled auditability into the schema: `createdBy` plus an append-only `updatedBy[]` array of `{userId, updatedAt}` on every editable entity.**
    *Evidence level:* VERIFIED. *Evidence:* Election, Poll, Event, Group models.

**Claims that are NOT available and must never be made:** user counts, votes cast, elections run, revenue, uptime, performance benchmarks, adoption, retention, time saved, or any before/after comparison. There is no data behind any of them.

---

## 13. CV achievement candidates

### High-confidence, quantified (all quantities are codebase/VCS measurements)

- Designed and built a full-stack community-management platform (React 18 · Redux Toolkit · Node/Express · MongoDB) as sole developer across 85 commits and 10 months, comprising ~15,000 lines of application code, 59 REST endpoints, 8 data models and 16 Redux reducers.
- Implemented a six-path authentication system — credentials, email verification, password reset, one-time login code, Google OAuth and Twitter/X OAuth — with bcrypt hashing, SHA-256 token hashing, encrypted time-limited login codes and HTTP-only JWT cookies.
- Built an online election module supporting multiple concurrent polls, cross-poll candidate exclusivity, vote transfer with count reconciliation and verified-voter gating, enforced through a four-tier RBAC model across 59 endpoints.
- Integrated Stripe (hosted Checkout, subscriptions with trial periods, Billing Portal, signature-verified webhooks) and GoCardless Direct Debit behind a single payment model, with server-side product resolution to prevent client-supplied pricing.
- Containerised and deployed the platform to Render using Docker (node:20-alpine) and Docker Compose, serving a Vite-built SPA and the REST API from one same-origin Node process.

### High-confidence, non-quantified

- Architected a MERN application end to end: schema design, REST API, authentication and authorisation, third-party integrations, state management, containerisation and deployment.
- Migrated the transactional email layer between providers (Gmail OAuth2/Nodemailer → Brevo API) in response to a production-environment delivery failure, with zero changes to calling code.
- Implemented optimistic-concurrency handling for concurrent writes to embedded document arrays, with bounded retry against re-read state.
- Designed audit trails into the data model, capturing creator and an append-only edit history on every mutable entity.

### Requires confirmation / not yet true (do not use until done)

- Anything about users, traffic, votes, payments, reliability or performance.
- Any "production" framing. Say *"deployed publicly"*, which is true, not *"in production"*, which is not.
- Test coverage, CI, monitoring — none exist today. Adding them converts three of the weaknesses in §10 into fresh, honest CV lines.

---

## 14. Portfolio case study material

**Headline:** *NIUK — a community-management and online-elections platform, built end to end.*

**Framing paragraph (public-safe, use verbatim if useful):**
> NIUK began as a product idea for a Nigerian diaspora community that had outgrown Twitter/X: membership was implicit, leadership elections were informal, Spaces and events disappeared into the timeline, and there was no way to collect dues. I designed and built the platform solo over ten months — authentication with six sign-in paths and device-based step-up verification, four-tier role-based access control, an elections module with multiple concurrent polls and verified voters, event and group management, and dual payment-provider integration — then deployed it as a containerised single-origin Node application. I chose not to pitch it to the community in the end, for reasons outside the code, and kept it as a demonstration of the engineering rather than a launched product.

**What to show:** the election flow (election → polls → candidates → verified voters → voting with concealed counts) is the most distinctive feature and the hardest to build; lead with it, not with auth.

**Structure to write up:** Context → Problem → My role (sole developer) → Architecture (use the §4 diagram) → Three deep-dives (election domain invariants; device-based 2FA; webhook-driven payment fulfilment) → What I'd do differently (§11) → Honest status (built, deployed, not launched).

**What I would improve next, in priority order:**
1. Move votes into their own collection with a compound unique index on `(pollId, voterId)` — turn the central business rule into a database guarantee.
2. Add integration tests for the election flow and the auth flow; wire up CI.
3. Remove the hard-coded secrets and personal address; finish or delete the GoCardless path.
4. Fix `castVote`'s route guard and the `const` retry bug.
5. Add rate limiting on all code/token endpoints.
6. Add structured logging and a health endpoint.

---

## 15. Interview story bank

**A. Domain invariants under concurrency** (complex problem solving)
*S:* Candidates had to belong to exactly one poll within an election, but candidates live in embedded arrays across sibling documents. *T:* Enforce exclusivity without a unique index and without transactions. *A:* Scan sibling polls, splice the candidate out, save, then push into the target; batch-safe, with distinct "moved" vs "added" responses; `VersionError` caught and retried against re-read state. *R:* Working exclusivity rule and a clear organiser-facing message. *E:* `poll.controller.js`.
*Strength of this story:* you can critique your own solution — the retry bug, and the fact that a `Vote` collection with a compound index was the better design. Self-critique reads as seniority.

**B. A failure that only appeared in deployment** (production debugging / ownership)
Gmail OAuth2 email worked locally and silently stopped on the deployed host; re-implemented against Brevo's API behind the same function signature, so no call site changed. Lesson: deploy early; shape abstractions around your application's needs, not the vendor's SDK. *E:* two implementations in `email.helper.js`, commit of 21 Dec 2024.

**C. Security design from primitives** (security)
Walk through the chain: bcrypt pre-save hook → SHA-256-hashed verification tokens → Cryptr-encrypted login codes with independent expiry → device allow-list triggering step-up auth → HTTP-only `secure` `sameSite:none` JWT cookie → server-side RBAC with a suspension check inside the guard. Then volunteer what's missing: rate limiting, token rotation, revocation. *E:* `auth.controller.js`, `auth.middleware.js`, `auth.helper.js`.

**D. Integrating two incompatible payment models** (integration)
Stripe is session- and webhook-driven; GoCardless is mandate- and billing-request-driven. Explain raw-body webhook ordering, why fulfilment runs server-side independently of the browser redirect, and why resolving product names from Stripe rather than the client matters. Be upfront that the GoCardless path stayed a spike.

**E. Deciding not to ship** (judgement / ambiguity)
The intended adopter community entered a leadership dispute. Rather than pitch into instability, you stopped, rebranded the work and repositioned it. Commit `1c6ff9c` *"removed niuk tags"* dates the decision. Shows you can separate sunk cost from forward value.

**F. Getting stuck and getting unstuck** (learning / resilience)
The commit *"After 7hours I couldn't figure out my desired switch role function"* is a timestamped record of a hard state-propagation problem, resolved two days later by lifting role state into Redux. Genuine, verifiable, and unusually honest.

---

## 16. Technical article and blog opportunities

1. **"One vote per voter: why I should have used a unique index instead of an embedded array."** Core problem: modelling votes in MongoDB. Why interesting: it is the clearest possible illustration of invariants belonging in the database, told against your own code. Must anonymise: nothing — this is pure technique. Do not publish: the hard-coded secrets, the GoCardless personal data.
2. **"VersionError: optimistic concurrency in Mongoose, and the retry loop I got wrong."** Explain the failure mode, show the retry pattern, show the `const` bug, show the `$pull`/`$push` and transaction alternatives. Publishing your own bug is credible and rare.
3. **"Stripe webhooks and Express body parsers: the ordering that silently breaks signature verification."** Short, high-utility, evergreen search traffic.
4. **"Swapping an email provider in one file: designing around your application, not the SDK."** Gmail OAuth2 → Brevo, with the signature-stability argument and the contrast against the payment layer, which leaked vendor concepts into the domain model.
5. **"Step-up authentication from primitives: device fingerprints, encrypted one-time codes and expiring tokens."** Include the honest caveat about rate limiting.
6. **"Serving a Vite SPA and an Express API from one origin — and the cookie problems it deletes."** `sameSite`, `secure`, CORS credentials, the asset-extension fallback, and anti-cache headers on `index.html`.

---

## 17. Professional content opportunities (LinkedIn / X)

Only ideas with a real story behind them:

- **"I built a voting system and put the most important rule in the wrong place."** The embedded-array-vs-unique-index lesson. Credible because it is self-critical and specific.
- **"I spent seven hours stuck on a role-switching function."** Quote your own commit message. Developers respond to documented struggle far more than to polished outcomes.
- **"Three bugs that only existed after I deployed."** Email, post-auth routing, SPA/API origin — all three in a single December week. Point: deploy early.
- **"I built a platform for a community, then decided not to pitch it."** The judgement story. Honest, unusual, and it pre-empts the "did anyone use it?" question by answering it first.
- **"Portfolio project ≠ toy project — here's the architecture."** Post the §4 diagram, list the counts (59 endpoints, 8 models, 16 reducers, 6 auth paths), state plainly that it has no users and that the engineering is the point.
- **"What I'd change if I rebuilt it today."** The §14 improvement list; shows current judgement, not just past work.

Avoid entirely: anything implying adoption, impact or scale.

---

## 18. Private vs public-safe

**PRIVATE — never publish:**
- Contents of `.env` and `client/.env` (Mongo URI, JWT and Cryptr secrets, Google/Twitter/Stripe/Brevo credentials, Cloudinary preset).
- The hard-coded GoCardless sandbox token, Stripe webhook signing secret and `express-session` secret present in source (§10.1) — and these should be **rotated and removed from the code** regardless of whether anything is published.
- **The developer's personal postal address and email, hard-coded in `payment.controller.js`.** Remove before the repository is shown to anyone.
- The Render dashboard deploy link and Google Cloud project URLs in the READMEs.
- The weakness list in §10, as a list. Individual items are fine to discuss in an interview; publishing a vulnerability inventory of a deployed app is not.
- The specific community's identity and its leadership dispute. Say "a diaspora community organisation"; do not name names or characterise anyone's conduct.

**PUBLIC-SAFE:**
- Architecture diagram, technology stack, feature list, code-size and endpoint counts.
- The engineering stories in §15 and §16, with the caveat that §15-E is told about *circumstances*, not *people*.
- All claims in §12 with their qualifications attached, and the bullets in §13's first two sections.
- The honest statement that the platform was deployed but never launched, and has no users.

---

## 19. Evidence gaps

| Missing | Why it would help | Where it might exist | Can it be closed? |
|---|---|---|---|
| Any usage data | Would enable scale/adoption claims | Nowhere — no production DB, no real users | **No.** Permanently unavailable, by the nature of the project. Stop looking. |
| Screenshots / demo video of the running app | Portfolio needs visuals; the Render deploy may be slow-starting or stale | Run locally via `docker-compose up` and capture the election flow | **Yes — highest-value next action.** |
| Confirmation the Render deploy still functions beyond returning 200 | "Deployed publicly" is stronger with a working link | `https://niuk-project.onrender.com` | **Yes** — open it, test a signup, and either keep it alive or stop linking it. |
| Seeded demo data | A reviewer clicking an empty dashboard sees nothing | Write a seed script with fictional users, an election and polls | **Yes** — a day's work, and it makes the demo tell the story. |
| Exact first-deploy date | Would date the "deployed" claim precisely | Render dashboard deploy history (link in `README.md`) | **Yes** — check the dashboard. |
| Design rationale at the time | The §8 trade-off table is reconstructed from code, not from recorded decisions | Only your memory | **Partly** — write a short ADR-style note now, dated, before the memory fades further. |
| Tests / CI | Converts §10's biggest weakness into a strength | Does not exist | **Yes** — add integration tests for election and auth flows. |

---

## 20. Appendix — evidence notes and reproducible measurements

All commands run from `/Users/abrahamakintola/Documents/Coding/portfolio-projects/niuk/demoapp-project`, 17 October 2026.

| Measurement | Value | Command |
|---|---|---|
| Commits | 85 | `git log --oneline \| wc -l` |
| Contributors | 1 (Akintola Abraham) | `git shortlog -sne --all` |
| First commit | 2024-02-19 04:19 +0000 | `git log --format='%ad' --date=iso \| tail -1` |
| Last commit | 2024-12-22 00:59 +0000 | `git log --format='%ad' --date=iso \| head -1` |
| Commits by month | Feb 17 · Mar 32 · Apr 6 · Sep 4 · Oct 13 · Dec 13 | `git log --format='%ad' --date=format:'%Y-%m' \| sort \| uniq -c` |
| Distinct files touched | 228 | `git log --format= --name-only \| sort -u \| wc -l` |
| Backend LOC | 4,153 | `wc -l controllers/*.js routes/*.js models/*.js middleware/*.js helpers/*.js index.js` |
| Frontend JS/JSX LOC | 10,666 | `find client/src -name '*.jsx' -o -name '*.js' \| xargs cat \| wc -l` |
| Frontend source files | 79 | `find client/src -name '*.jsx' -o -name '*.js' \| wc -l` |
| Route handlers | 59 (21 POST · 20 GET · 12 PATCH · 6 DELETE) + 1 raw webhook | `grep -hoE '^router\.(get\|post\|patch\|delete)' routes/*.js \| sort \| uniq -c` |
| Mongoose models | 8 | `ls models/` |
| Redux state files | 24 (16 reducers registered) | `ls client/src/redux/features/*/*.js`; `redux/store.js` |
| `.env` tracked in git? | **No** | `git ls-files \| grep -i env` → empty |
| Local DB reachable? | **No** | `mongosh mongodb://localhost:27017/demoapp-project` → `ECONNREFUSED` |
| Public deploy status | HTTP 200 | `curl -s -o /dev/null -w "%{http_code}" -L https://niuk-project.onrender.com/` |

**Relationship between the two sub-repositories:** `anonapp-project` holds 55 commits ending 16 April 2024 in a `client/` + `server/` layout; `demoapp-project` holds those same 55 plus 30 more, ending 22 December 2024, with the server flattened to the repository root. They are the same project at two points in time. All measurements above use `demoapp-project`.

**Key commits cited in this dossier:**
- `1c6ff9c` — *"removed niuk tags"*, 2 Apr 2024, 15 files — de-branding and pivot to portfolio artefact.
- *"After 7hours I couldn't figure out my desired switch role function"*, 24 Mar 2024 — documented difficulty.
- *"added election and voting for users"*, 4 Oct 2024 — election module.
- *"fix: email not sending and added node serve react"*, 21 Dec 2024 — provider migration and same-origin serving.
- *"fix: bad routing after authentication"*, 18 Dec 2024 — deployment-surfaced bug.
- *"feat: added docker"*, 21–22 Dec 2024 — containerisation.

**Methodology note:** no production database existed to query; no write of any kind was performed against any database; no credential value appears in this document or in the accompanying JSON. Every statistic above is reproducible from the repository by anyone with a copy of it.
