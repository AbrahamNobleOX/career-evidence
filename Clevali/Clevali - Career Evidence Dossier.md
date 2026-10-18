# Clevali (Expense Tracker) — Career Evidence Dossier

**Audit date:** 17 October 2026
**Repository audited:** `expense-tracker` — https://github.com/AbrahamNobleOX/expense-tracker-app
**Local path:** `~/Documents/Coding/personal-projects/tracking-reactnn`
**Branch audited:** `v1-golive` (the release branch; 2 commits ahead of `main`)

> **Evidence classification used throughout:** **VERIFIED** (directly evidenced in code, configuration or Git history) · **DERIVED** (calculated from verified evidence) · **USER-ATTESTED** (first-hand account from the author; not independently verifiable from the repository, but noted where code evidence is consistent with it) · **STRONGLY SUPPORTED** · **INFERRED** · **UNKNOWN / REQUIRES CONFIRMATION**.

---

## 1. Executive Summary

**Clevali** is a cross-platform personal expense tracker built with **React Native (Expo SDK 53) and TypeScript**, storing all data in an **on-device SQLite database**. It is a solo-authored portfolio and skills project, built in a concentrated burst between **5 June and 22 July 2025**, and — per the author — the **only one of his mobile applications to be published to both the Google Play Store and the Apple App Store**, where the Android release remained listed for roughly a year. *(Build and code evidence VERIFIED; store publication USER-ATTESTED.)*

**Why it exists.** Two stated reasons, both of which the artefacts support. First, to consolidate the author's understanding of cross-platform mobile development end to end — not just building a UI, but taking an app through native configuration, signing, store metadata, review and release. Second, it was built in preparation for a **UK Global Talent visa application**, as demonstrable evidence of breadth across web and mobile. *(USER-ATTESTED.)* The repository is consistent with both: the final five commits are pure release engineering (EAS build configuration, dependency pinning, hiding developer-only controls, and an iOS export-compliance declaration), on a branch literally named `v1-golive`.

**Role and contribution.** Authorship is unambiguous: **59 of 59 commits (100%) are the author's**, across every layer — data model, navigation, UI, notification scheduling, import/export, and store release configuration. There is no team attribution ambiguity in this project. *(VERIFIED — `git shortlog -sne --all`.)*

**What was actually built** — all verifiable in code:

- An **offline-first SQLite data layer** (5 tables) with an idempotent bootstrap and additive in-place schema evolution, plus a **mock-database shim** that lets the identical codebase run on web where `expo-sqlite` has no implementation.
- A **multi-currency model** — a normalised `currencies` / `user_currencies` pair with a single-default invariant enforced in the data layer — driving per-currency filtering, per-currency budgets and per-currency totals throughout the app.
- **Transactional JSON import/export as a deliberate substitute for cloud sync**: a full-database export shared through the OS share sheet, and an import that runs inside `BEGIN`/`COMMIT`/`ROLLBACK` with per-row validation, upsert-by-id, and skip-and-warn tolerance of malformed rows.
- A **reminder-notification scheduler** that works around a platform limitation: the OS scheduling API offers fixed calendar repeats, so daily reminders use a native `DAILY` trigger while every other cadence (2, 3, 7, 14, 30 days) is implemented as a computed one-shot date trigger re-armed from a `last_triggered` timestamp.
- A **9-route Expo Router application** with first-run gating, a hidden-route pattern for settings and debug screens, swipe-to-edit/delete gestures, month/year grouping with lazily loaded historical years, and budget progress visualisation.

**Production evidence: none exists, and that is an architectural consequence, not an omission.** The application has **no backend, no server, no remote database and no analytics or crash reporting**. Every record a user creates lives only in SQLite on their own device and is never transmitted anywhere. There is therefore no production database for this audit to query and no usage telemetry to report. **This dossier contains no usage, retention or adoption metrics, and none should be claimed** until store console figures are retrieved (see §19). *(VERIFIED by inspection of the full dependency list and source tree.)*

**The strongest career evidence here is not scale — it is shipping and trade-off articulation.** Two things make this project worth having on record. First, it is **proof of delivery through the full mobile release pipeline**: `versionCode 17` in the Android configuration establishes that at least seventeen distinct Android build artefacts were produced, and the last commit on the release branch is an iOS export-compliance declaration — the specific, unglamorous friction that only appears when you are actually submitting to App Store Connect. Second, it contains a **clean, defensible privacy-versus-measurability trade-off**: the author chose an architecture in which user financial data never leaves the device, and the direct cost of that choice is that he cannot tell you how the app is used. Being able to state that cost plainly, rather than inventing numbers, is the more valuable interview position.

---

## 2. Project Overview

| Attribute | Detail |
|---|---|
| Human-readable name | **Clevali — Expense Tracker** (`app.json` → `expo.name`) |
| Internal slug | `expense-tracker` (previously `tracking-react-native`, renamed 10 Jun 2025) |
| Project type | Cross-platform mobile application (iOS + Android), offline-first, no backend |
| Purpose | Personal/portfolio skills project; preparation for a UK Global Talent visa application *(USER-ATTESTED)* |
| Author | Akintola Abraham — sole contributor |
| Build window | 5 June 2025 → 22 July 2025 (**47 days**) |
| Total commits | **59**, all by one author |
| Application code | **5,540 lines** of TypeScript/TSX across 9 routes and 12 library modules |
| Net code churn | 7,615 insertions / 968 deletions (excluding `package-lock.json`) |
| Database | SQLite on device (`expenses.db`), **5 tables** |
| Bundle identifier | `com.techfric.expensetracker` (identical on iOS and Android) |
| Android `versionCode` | **17** |
| iOS `buildNumber` | 1 |
| EAS project | Configured (`appVersionSource: remote`, production `autoIncrement`) |
| Automated tests | **0** |
| Store status | Published to Google Play and Apple App Store; Play listing live ~1 year; Apple developer subscription believed lapsed *(USER-ATTESTED)* |

### What the application does

A user opens the app for the first time and is routed into a **currency setup** flow: they pick one or more currencies from a built-in catalogue of 20 and star one as their default. From then on, the app records expenses — each with a date, currency, category (one of eight), free-text description and amount — into a local SQLite database.

The **Expenses** screen presents that data as a two-level drill-down. The top level is a list of months grouped under year headers, newest first, with the current month pinned to the top and badged; each month card shows the expense count, the total for the selected currency, and — if a budget exists — a progress bar that turns red and reports the percentage overspend when the month exceeds budget. Scrolling to the bottom lazily loads the preceding year, back to a floor of 2000, generating placeholder month cards for years with no data. Tapping a month drills into a day-grouped `SectionList`, where each day carries its own header and subtotal, and individual expenses can be swiped left to reveal edit and delete actions.

Alongside this sit three configuration flows — **budgets** (one monthly limit per year per currency), **reminders** (an enable toggle, a time of day, and one of six frequencies), and **currency management** (add, remove, change default, with a guard preventing removal of the last remaining currency) — and two data flows: **export** of the entire database as JSON through the OS share sheet, and **import** of such a file back in.

### Who uses it

End users are individuals tracking personal spending, particularly those operating in more than one currency — the inclusion of **GBP and NGN** as the pair used throughout the author's own sample data strongly suggests the multi-currency design was driven by the real need of someone living between the UK and Nigeria. *(INFERRED from `lib/sampleData.ts` and commit `8ade42f`.)*

---

## 3. My Role and Contribution

### Verified personal contribution

| Measure | Value |
|---|---|
| Commits authored | **59 / 59 (100%)** |
| Other contributors | 0 |
| Period | 5 Jun – 22 Jul 2025 |

**Evidence level: VERIFIED** (`git shortlog -sne --all` returns a single author).

Every component of this system — schema, data access, navigation, all nine screens, the notification scheduler, the import/export subsystem, the build configuration and the store release — is the author's own work. Unlike a client or employer project, there is no portion of this codebase for which individual authorship is in question.

### Contribution by area

| Area | Evidence |
|---|---|
| **Data modelling** | 5-table SQLite schema designed across commits `9e4b432` (expenses), `51975dd` (currency), `69d2cc8` (budgets), `02703c1` (reminders), `4422e31` (user currencies) |
| **Data access layer** | 12 modules in `lib/`, ~1,800 lines; all SQL parameterised |
| **Navigation & app shell** | Expo Router migration (`2d6ba76`), tab navigator with hidden routes, first-run redirect gating (`app/_layout.tsx`) |
| **UI** | 9 screens, 3,700+ lines; NativeWind/Tailwind styling with a StyleSheet fallback on two screens |
| **Gestures & interaction** | `react-native-gesture-handler` swipe-to-action (`8d34744`), pull-to-refresh, lazy year loading (`70e1715`) |
| **Notifications** | Permission handling, category/action registration, dual-mode scheduling, foreground and response listeners (`lib/notifications.ts`, `02703c1`, `dc5d753`) |
| **Data portability** | Transactional JSON import, full-table export, raw DB export, transactional wipe (`a41366a`, `ac49f3c`, `d574753`, `da16d52`) |
| **Release engineering** | EAS configuration, dependency pinning, dev-control removal, iOS export-compliance declaration (`2fef9f6`, `18e8ff1`, `b7f8549`, `94d80ea`) |
| **Documentation** | `README.md`, `BUDGET_FEATURE.md`, `DATA_PORTABILITY.md` — feature specs written alongside the code |

### Development pattern (DERIVED)

| Period | Commits | Character |
|---|---|---|
| 5–10 Jun 2025 | **54 (92%)** | Core build — entire feature set delivered in six days |
| 13 Jul 2025 | 4 | Build configuration, dependency updates, dev-control removal |
| 22 Jul 2025 | 1 | iOS export-compliance declaration |

Commit-message convention is disciplined and consistent: 49 `feat`, 5 `refactor`, 2 `fix`, 2 `chore`. The shape of this history — a dense build sprint, then a month's gap, then a short cluster of release-only commits — is itself the evidence that the project went through an actual store submission cycle rather than stopping at "it runs on my simulator".

---

## 4. System Architecture

### Component view

```
┌──────────────────────────────────────────────────────────────┐
│  DEVICE (iOS / Android)                                      │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  PRESENTATION — Expo Router (file-based routing)       │  │
│  │                                                        │  │
│  │  Tab bar:   index  ·  add-expense  ·  expenses         │  │
│  │  Hidden:    set-budget · notification-settings ·       │  │
│  │             currency-settings · currency-setup · debug │  │
│  │                                                        │  │
│  │  Styling: NativeWind (Tailwind) + StyleSheet           │  │
│  │  Gestures: react-native-gesture-handler (Swipeable)    │  │
│  └────────────────────────┬───────────────────────────────┘  │
│                           │                                  │
│  ┌────────────────────────▼───────────────────────────────┐  │
│  │  DOMAIN / DATA ACCESS — lib/                           │  │
│  │                                                        │  │
│  │  db.ts ──────────── schema bootstrap + expense CRUD    │  │
│  │  currency.ts ────── currency catalogue + user set      │  │
│  │  budget.ts ──────── yearly monthly budgets             │  │
│  │  reminderSettings.ts ─ reminder persistence            │  │
│  │  notifications.ts ─ scheduling + permissions           │  │
│  │  dbJsonExport.ts ── full-DB export                     │  │
│  │  dbJsonImport.ts ── transactional import               │  │
│  │  dbWipe.ts ──────── transactional reset                │  │
│  │  utils.ts ───────── Intl formatting + date keys        │  │
│  └──────┬───────────────────────────┬─────────────────────┘  │
│         │                           │                        │
│  ┌──────▼──────────────┐  ┌─────────▼──────────────────────┐ │
│  │  SQLite             │  │  AsyncStorage                  │ │
│  │  (expo-sqlite)      │  │  (first-run / setup flags)     │ │
│  │                     │  └────────────────────────────────┘ │
│  │  expenses           │                                     │
│  │  yearly_monthly_    │  ┌────────────────────────────────┐ │
│  │    budgets          │  │  OS Notification Scheduler     │ │
│  │  reminder_settings  │  │  (expo-notifications)          │ │
│  │  currencies         │  └────────────────────────────────┘ │
│  │  user_currencies    │                                     │
│  └─────────────────────┘  ┌────────────────────────────────┐ │
│                           │  Filesystem + Share sheet      │ │
│                           │  (expo-file-system / -sharing  │ │
│                           │   / -document-picker)          │ │
│                           └────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘

        ✗ No backend    ✗ No network calls    ✗ No analytics
        ✗ No auth       ✗ No remote storage   ✗ No crash reporting
```

**The absence of the right-hand side of a conventional architecture diagram is the architecture.** There is no API client, no authentication, no sync engine and no telemetry anywhere in the dependency tree or source. The only data that ever leaves the device does so when the user explicitly taps Export and chooses a destination in the OS share sheet. *(VERIFIED.)*

### Database schema

| Table | Columns | Notes |
|---|---|---|
| `expenses` | `id` PK, `category`, `description`, `amount` REAL, `currency`, `date`, `updated` | Currency stored denormalised as a code string; `updated` added by later migration |
| `yearly_monthly_budgets` | `id` PK, `year`, `currency`, `monthly_limit` REAL, **UNIQUE(year, currency)** | One uniform monthly limit per year per currency |
| `reminder_settings` | `id` PK, `frequency` INTEGER (days), `start_time`, `last_triggered`, `enabled` | Single-row table by convention |
| `currencies` | `id` PK, `code` UNIQUE, `name`, `symbol` | Populated from a 20-currency catalogue on selection |
| `user_currencies` | `id` PK, `currency_id` FK → `currencies(id)` ON DELETE CASCADE, `is_default` | Exactly one row carries `is_default = 1`, enforced in code |

### Technology stack

| Layer | Technology | Version |
|---|---|---|
| Framework | React Native | 0.79.3 |
| Platform | Expo SDK | ~53.0.10 (New Architecture enabled) |
| Language | TypeScript | ~5.8.3, `strict: true` |
| UI runtime | React | 19.0.0 |
| Routing | Expo Router | ~5.0.7 |
| Styling | NativeWind + Tailwind CSS | 4.1.23 / 3.4.17 |
| Database | expo-sqlite | ~15.2.11 |
| Key–value store | @react-native-async-storage/async-storage | ^2.2.0 |
| Notifications | expo-notifications | ^0.31.3 |
| Gestures / animation | react-native-gesture-handler, react-native-reanimated | ^2.25.0 / ^3.6.1 |
| File & share | expo-file-system, expo-sharing, expo-document-picker | ~18.1.10 / ~13.1.5 / ^13.1.5 |
| Date input | @react-native-community/datetimepicker | ^8.4.1 |
| Icons | @expo/vector-icons | ^14.1.0 |
| Build & release | EAS Build / EAS Submit | CLI ≥ 16.15.0 |

TypeScript is configured in **strict mode** across the whole project. *(VERIFIED — `tsconfig.json`.)*

---

## 5. Engineering Implementation — the parts that were actually difficult

### 5.1 Running one codebase on a platform where the database does not exist

`expo-sqlite` has no web implementation. Rather than fork the data layer or block web entirely, `initDatabase()` detects `Platform.OS === "web"` and installs a **structurally-typed mock** satisfying the `SQLiteDatabase` interface — `getAllAsync` returns `[]`, `runAsync` returns a zero-row result, and so on. Every consumer of the data layer is written against one interface and needs no platform awareness.

**Why this is worth noting:** it is a small, cheap application of dependency inversion at exactly the point where a cross-platform codebase usually fractures. The honest limitation, which should be stated rather than hidden, is that web therefore renders a functional but permanently empty app — this is a *graceful degradation* decision, not web support.

### 5.2 Schema evolution without a migration framework

The bootstrap is idempotent: every `CREATE TABLE IF NOT EXISTS` runs on each launch. Columns added after the first release (`currency`, then `updated`) are applied as `ALTER TABLE … ADD COLUMN` wrapped in `try/catch`, treating the error from an already-existing column as success. The `updated` migration also backfills: `UPDATE expenses SET updated = date WHERE updated IS NULL`.

**The trade-off, stated honestly:** this works, and for an app with no server-side coordination and a handful of additive changes it is pragmatic. But it uses exception control flow in place of state, has no schema version table, and gives no way to express a non-additive change. **This is the single clearest "what I would do differently" in the project** — see §9 and §12.

### 5.3 Fighting the OS notification scheduler

`expo-notifications` offers calendar-based repeating triggers — daily, weekly — but not "every N days". The implementation splits on this:

- **Daily** uses the native `DAILY` trigger with hour and minute, and is genuinely fire-and-forget.
- **Every other cadence** computes the next occurrence manually: days elapsed since `last_triggered`, days remaining until due, a same-day check that rolls to tomorrow if the configured time has already passed today, then schedules a **one-shot `DATE` trigger**. A `last_triggered` timestamp is written back when the notification arrives — handled both for taps (`addNotificationResponseReceivedListener`) and for foreground delivery (`addNotificationReceivedListener`) — and the next one is armed from there.

All scheduled notifications are cancelled before rescheduling, which keeps the system idempotent and prevents duplicate-reminder accumulation across settings changes.

**Known limitation to be honest about:** because each non-daily reminder is a single scheduled date, the chain depends on the app being opened or the notification being received to re-arm. An uninstall-reinstall or a long silent period can break the cadence. A production app would reconcile this on every cold start.

### 5.4 Form state leaking across a tab navigator

This is the problem the Git history shows the author genuinely wrestling with, across six commits over two days (`b105e1c`, `e7ea599`, `6155d3c`, `921b072`, `7ac750f`, `de49a46`).

**Situation.** `add-expense` is both the "new expense" tab and the "edit expense" destination reached by swiping a row. In a tab navigator the screen stays mounted, so tapping the tab after editing re-presented the previous expense's values.

**Solution.** Three mechanisms in combination: the tab's `href` carries `{ reset: "true", ts: Date.now() }` so each tap produces a distinct parameter set, a `useEffect` keyed on `[id, reset, ts]` branches between reset / load-for-edit / clear-for-new, and `useFocusEffect` refreshes currency data on every focus so a currency added elsewhere appears immediately.

**Lesson, in the author's own words to reach for in an interview:** in a stack navigator, a screen's lifecycle does your state management for you; in a tab navigator it does not, and you have to make the identity of "which thing am I editing" explicit rather than relying on mount and unmount.

### 5.5 Transactional import with partial-failure tolerance

`importDatabaseFromJson` opens `BEGIN TRANSACTION`, then walks five collections — expenses, budgets, reminder settings, currencies, user currencies — validating each row's required fields, upserting by primary key, counting what it applied, and `console.warn`-ing and skipping anything malformed rather than aborting. It commits at the end and rolls back inside a `catch`, with the rollback itself wrapped defensively. The caller receives per-collection counts and surfaces them to the user.

The design decision underneath is that **a partially valid backup file should restore the valid part**, which for a personal data file is almost certainly the right call. The counter-argument — that silently skipping rows can hide corruption from the user — is real, and the mitigation would be surfacing skipped-row counts in the UI rather than only in the console.

### 5.6 Destructive-action safety

Wiping all data requires **two sequential confirmations** with escalating wording, then runs inside a single transaction that counts each table before deleting, honours foreign-key ordering (`user_currencies` before `currencies`), clears the three AsyncStorage setup flags so the app correctly re-enters first-run setup, and cancels all scheduled notifications so reminders do not fire against an empty database.

That last detail is the mark of someone thinking about **state that lives outside the database** — OS-level scheduled notifications are not rolled back by a SQL transaction, and forgetting them would leave the app nagging a user about expenses that no longer exist.

### 5.7 Multi-currency as a first-class concern

Currency is not a display formatting concern in this app; it partitions the data. Expenses are filtered by selected currency, budgets are keyed by `(year, currency)`, month totals are per-currency, and the currency selector is present on the add, list and budget screens. The single-default invariant is maintained by clearing all defaults before setting one (`UPDATE user_currencies SET is_default = 0`), and removing the current default promotes another currency automatically.

Formatting uses the `Intl.NumberFormat` API with a locale and currency code, plus a live thousands-separator formatter applied to the amount field as the user types, with a matching unformatter before persistence.

---

## 6. Main Workflows

| Workflow | Path |
|---|---|
| **First run** | Launch → `initDatabase()` → check two AsyncStorage flags → if incomplete, `router.replace("/currency-setup")` → select currencies, star a default → persist → mark both flags → home |
| **Record an expense** | Add tab (carries reset params) → date picker (future dates blocked) → currency → category → description → amount (live-formatted) → validate → `INSERT` → clear form → navigate to list |
| **Edit an expense** | Expenses → drill into month → swipe row left → pencil → `add-expense?id=N&ts=…` → load → `UPDATE` with new `updated` timestamp |
| **Review spending** | Expenses → currency filter → months grouped under year headers, current month pinned → budget progress per month → tap month → day-grouped sections with subtotals |
| **Set a budget** | Home or banner → Set Budget → currency + year (current + 4 future) + amount → upsert on `(year, currency)` → list of existing budgets refreshes |
| **Configure reminders** | Home → Reminders → toggle, frequency (6 options), time → request permission → cancel existing → schedule daily or computed one-shot |
| **Back up / restore** | Export JSON → all tables serialised → share sheet. Import JSON → document picker → confirm → transactional upsert → navigate to expenses |
| **Reset** | Home → Wipe All Data → two confirmations → transactional delete of 5 tables + AsyncStorage flags + notification cancellation |

---

## 7. Production Evidence and Statistical Analysis

### Why there is no production data

This section exists to state a negative precisely, because the alternative is for the gap to be filled later with a guess.

The application has **no server component of any kind**. There is no API, no remote database, no authentication, no sync, and no analytics or crash-reporting SDK — verified by inspecting the complete dependency list in `package.json` and searching the source for network calls. Every expense, budget, currency preference and reminder setting a user creates is written to a SQLite file inside that app's private storage on that user's own device, and is readable only by that app on that device.

**The consequence, stated plainly: the author has no way to know how many expenses users recorded, how often they opened the app, or whether they kept using it.** That is not an oversight in the audit; it is the direct and intended cost of the privacy decision described in §8.1.

### Repository metrics (VERIFIED)

| Metric | Value | Source |
|---|---|---|
| Total commits | 59 | `git rev-list --count HEAD` |
| Contributors | 1 (100% author) | `git shortlog -sne --all` |
| Build window | 5 Jun – 22 Jul 2025 (47 days) | first and last commit dates |
| Core build sprint | 54 commits in 6 days (92%) | commits-per-day aggregation |
| Application code | 5,540 lines TS/TSX | `wc -l app/*.tsx lib/*.ts` |
| Code churn (excl. lockfile) | +7,615 / −968 | `git log --numstat` |
| Routes | 9 | `app/` directory |
| Library modules | 12 | `lib/` directory |
| Database tables | 5 | schema inspection |
| Currency catalogue | 20 | `WORLD_CURRENCIES` |
| Expense categories | 8 | `add-expense.tsx` |
| Reminder frequencies | 6 | `FREQUENCY_OPTIONS` |
| Automated tests | 0 | no test files, no test runner in dependencies |
| Most-revised file | `app/expenses.tsx` (22 commits) | `git log --name-only` |

### Release metrics

| Metric | Value | Evidence level | Qualification |
|---|---|---|---|
| Android `versionCode` | **17** | VERIFIED (`app.json`) | Establishes that at least 17 distinct Android build artefacts were produced under this configuration. It is **not** a count of public Play Store releases — internal, preview and failed builds also consume version codes. State it as builds, not releases. |
| iOS `buildNumber` | 1 | VERIFIED (`app.json`) | With `appVersionSource: "remote"`, EAS manages the authoritative number server-side, so the local value does not reflect iOS build count. |
| Release branch | `v1-golive`, 2 commits ahead of `main` | VERIFIED | Contains the dev-control removal and the iOS encryption declaration — i.e. the store-submission work was deliberately isolated. |
| App Store / Play Store publication | Published to both | **USER-ATTESTED** | Consistent with, but not proven by, the EAS configuration, dual bundle identifiers and iOS export-compliance declaration present in the repository. |
| Play Store listing duration | ~1 year | **USER-ATTESTED** | Not independently verifiable from the repository. |
| Downloads / installs / active users | **UNKNOWN** | — | Retrievable from the Play Console and App Store Connect — see §19. Do not estimate. |

### A note on the sample data file

`exported-data.json` in the repository root contains 94 expenses, 4 budgets, 3 currencies and 1 reminder setting. **This is developer sample data generated by `lib/sampleData.ts` for testing, not user data.** It must never be cited as usage evidence. It is noted here only so that a future reader does not mistake it for one.

---

## 8. Architectural Decisions and Trade-offs

### 8.1 Local-only storage, no cloud sync

| | |
|---|---|
| **Problem** | An expense tracker holds a detailed picture of someone's financial life: what they buy, where, how much, how often. Any server-side storage makes the developer the custodian of that. |
| **Decision** | Store everything in on-device SQLite. No backend, no account, no network transmission at all. |
| **Reasoning** | Privacy by architecture rather than by policy: data that is never collected cannot be breached, subpoenaed, mishandled or monetised. It also removes authentication, hosting cost, GDPR data-controller obligations and an entire class of security surface from a solo-maintained app. *(Privacy intent USER-ATTESTED; the architecture itself VERIFIED.)* |
| **Trade-offs accepted** | No multi-device sync. No server-side backup — device loss means data loss unless the user exported. **No analytics, therefore no product insight and no usage metrics at all.** |
| **Result** | A genuinely private app, and a measurable cost the author can name precisely: he shipped to two app stores and cannot tell you how the app was used, because he chose not to be able to. |

### 8.2 Manual JSON export/import instead of sync

| | |
|---|---|
| **Problem** | §8.1 leaves users with no way to move to a new phone. |
| **Decision** | Full-database JSON export through the OS share sheet, and transactional import from a picked file. |
| **Reasoning** | Preserves the no-server guarantee while giving users genuine data ownership — a human-readable file they control, can inspect, and can store wherever they choose. |
| **Trade-offs** | Manual rather than automatic; no conflict resolution; the user must remember to do it. |
| **Result** | Working device-to-device portability, documented for users in `DATA_PORTABILITY.md`. **Documentation drift to fix:** that document describes a "Merge" versus "Replace All" choice at import; the shipped implementation offers a single upsert-by-id import path. One or the other should be corrected. *(VERIFIED by comparing `DATA_PORTABILITY.md` against `app/index.tsx` and `lib/dbJsonImport.ts`.)* |

### 8.3 Uniform monthly budget per year, rather than per month

| | |
|---|---|
| **Problem** | Budgets could be modelled per month (flexible, 12 rows per year per currency) or as one figure applied to all months (simple, 1 row). |
| **Decision** | One `monthly_limit` per `(year, currency)`, enforced by a UNIQUE constraint. |
| **Reasoning** | Explicitly documented as a simplicity choice in `BUDGET_FEATURE.md`: most people think in terms of a steady monthly target, and the simpler model makes the UI a single field instead of a twelve-row editor. |
| **Trade-offs** | Cannot express a higher December budget or a mid-year change. |
| **Known defect this introduced** | `loadMonthlyBudget()` loads only the **current year's** budget and the month list then compares **every displayed month, including months from earlier years, against that single figure**. Historical months are therefore scored against a budget that may not have applied to them. The fix is to resolve the budget per month's own year. *(VERIFIED — `app/expenses.tsx`.)* |

### 8.4 Denormalised currency code on expenses

Expenses store a currency **code string** rather than a foreign key to `currencies`. This keeps queries and the JSON export flat and self-describing, and means removing a currency from the user's selection cannot orphan expense rows. The cost is no referential integrity between expenses and the currency table, and no cascade if a currency's metadata changes. For an append-mostly local ledger where currency codes are an ISO standard that does not change, this is a defensible call — but it should be presented as a deliberate choice, with the cost named.

### 8.5 Expo managed workflow with EAS

Choosing Expo over bare React Native traded some native-module freedom for a dramatically shorter path from code to signed store binary — credentials, build infrastructure and submission handled by EAS rather than by Xcode and Gradle configuration. **For a solo developer whose explicit goal was to reach both stores, this is the decision that made the goal achievable**, and `eas.json` with its development / preview / production profiles and `autoIncrement` is the evidence it was used properly.

### 8.6 NativeWind for styling

Tailwind utility classes via NativeWind on seven screens, with two screens (`currency-setup`, `currency-settings`) and parts of `_layout` using `StyleSheet.create` instead. The reasoning is familiar from web work — rapid iteration without context-switching to a separate style object — and the author's existing Tailwind fluency transferred directly. **The inconsistency between the two approaches within one codebase is a real, if minor, cohesion weakness** and should be acknowledged rather than defended if it comes up.

---

## 9. Known Weaknesses (recorded deliberately)

A career record is more useful, and far more defensible in an interview, when it names its own problems. These are all verified by inspection.

| # | Issue | Where | Severity |
|---|---|---|---|
| 1 | **Zero automated tests.** No test runner, no test files. The date-key, currency-formatting and budget-percentage logic in `lib/utils.ts` is pure and trivially unit-testable; its absence is a gap, not a constraint. | project-wide | High |
| 2 | **Budget compared across the wrong year** — see §8.3. Historical months are scored against the current year's budget. | `app/expenses.tsx` | Medium (correctness) |
| 3 | **`useRef` called inside `renderExpenseItem`**, a render *function* rather than a component. It works because the function is invoked in a component's render path, but it violates the Rules of Hooks and is fragile under list virtualisation. Extracting a real `ExpenseItem` component fixes it. | `app/expenses.tsx` | Medium |
| 4 | **Table name interpolated into SQL** in the export: `SELECT * FROM ${table.name}`. Not exploitable here — names come from `sqlite_master` — but it is the pattern that becomes an injection the moment the source changes. | `lib/dbJsonExport.ts` | Low (pattern) |
| 5 | **Exception-driven migrations** with no schema version table — see §5.2. | `lib/db.ts` | Medium |
| 6 | **Developer surfaces shipped in the release build**: the debug screen remains routable (hidden from the tab bar via `href: null`, not removed), and raw-DB-export plumbing stays in the bundle. The sample-data and raw-export buttons were commented out before submission (`b7f8549`) rather than removed behind a `__DEV__` guard. | `app/debug.tsx`, `app/index.tsx` | Low–Medium |
| 7 | **`initializeDefaultCurrencies()` is effectively dead** — it creates tables, checks for existing user currencies, and returns either way. | `lib/currency.ts` | Low |
| 8 | **Documentation drift** between `DATA_PORTABILITY.md` and the shipped import flow — see §8.2. | docs | Low |
| 9 | **Artificial 500 ms delay** in `loadMoreYears`, commented as simulating database latency. | `app/expenses.tsx` | Low |
| 10 | **Non-daily reminder chains can break** if the app is not opened for a long period — see §5.3. | `lib/notifications.ts` | Medium |
| 11 | **Unused import** of the deprecated `ProgressBarAndroid`. | `app/expenses.tsx` | Trivial |

---

## 10. Business Impact

There is no business here. Clevali is a personal project with no revenue model, no client and no commercial stakeholder, and **no business impact should ever be claimed for it**.

Its value is **career capital**, and that value is real:

- **Demonstrated delivery.** A complete product taken from empty repository to published store listing, solo, in under two months. For a portfolio, "shipped and reviewed by Apple and Google" is a materially different claim from "built".
- **Demonstrated breadth.** The author's other career-evidence projects are web and backend systems. This one establishes cross-platform mobile, native release engineering and offline-first data design — the versatility argument he set out to make. *(USER-ATTESTED motivation; the artefact supports it.)*
- **Demonstrated judgement.** The privacy architecture and the data-portability answer to the sync problem are coherent, defensible decisions with costs the author can state.
- **Evidence for an immigration application.** Built explicitly as supporting material for a UK Global Talent application. *(USER-ATTESTED.)*

---

## 11. Problems Solved

### Problem 1 — Reminder cadences the platform cannot express

- **Situation.** Users should be able to be reminded every 2, 3, 7, 14 or 30 days.
- **Problem.** `expo-notifications` repeating triggers cover fixed calendar units only. There is no "every N days".
- **Investigation.** The trigger-type API was read against the requirement; `DAILY` fits one case and nothing fits the rest.
- **Solution.** Hybrid scheduling: a native `DAILY` trigger for daily; for everything else a computed one-shot `DATE` trigger derived from `last_triggered`, frequency and configured time, including a same-day roll-forward when the time has passed, with the timestamp written back by both the tap listener and the foreground-received listener, and a blanket cancel before every reschedule.
- **Result.** All six cadences work. The remaining weakness — chain breakage after long inactivity — is known and documented.
- **Evidence.** `lib/notifications.ts`; commits `02703c1`, `dc5d753`.

### Problem 2 — Edit form showing the previous expense

Covered in full at §5.4. Six commits across two days; solved with explicit route-parameter identity plus `useFocusEffect`.

### Problem 3 — Adding currency support to a schema already in use

- **Situation.** The first version was single-currency; multi-currency support landed on 6 June, after the expenses table existed.
- **Problem.** Existing rows had no currency, and a `NOT NULL` column cannot simply be added to a populated table without a default.
- **Solution.** `ALTER TABLE expenses ADD COLUMN currency TEXT NOT NULL DEFAULT 'GBP'` wrapped in `try/catch`, so existing rows adopt a sensible default and re-running on an already-migrated database is a no-op. The same pattern, plus a backfill `UPDATE`, was used for `updated`.
- **Result.** Backward-compatible evolution with no data loss. The pattern's limits are recorded at §5.2 and §9.5.
- **Evidence.** `lib/db.ts`; commits `51975dd`, `39711de`.

### Problem 4 — Browsing history that does not exist yet

- **Situation.** A new user has no expenses; a long-term user has years of them. A month list must be useful in both cases without loading everything.
- **Solution.** Year-grouped month list starting at the current year, lazily appending the preceding year on `onEndReached` down to a floor of 2000, synthesising twelve placeholder month cards (non-interactive, visually muted) for years with no data, with the current month pinned to the top and badged.
- **Result.** A calendar that feels complete from day one and stays navigable at scale.
- **Evidence.** `app/expenses.tsx`; commits `1416830`, `70e1715`.

### Problem 5 — Apple's export-compliance gate

- **Situation.** Submission to App Store Connect.
- **Problem.** Every binary must declare whether it uses non-exempt encryption; without the declaration, each submission stalls on a manual question.
- **Solution.** `ITSAppUsesNonExemptEncryption: false` added to the iOS `infoPlist` — accurate here, since the app performs no cryptography and makes no network calls.
- **Result.** Submissions proceed without manual intervention. **This is the final commit in the repository**, and it is the clearest single artefact proving the project reached real store submission.
- **Evidence.** `app.json`; commit `94d80ea`, 22 July 2025.

---

## 12. Lessons Learned

1. **Shipping is a separate skill from building.** 92% of the code existed by 10 June. The app was not submitted until late July. The gap was not features — it was icons, splash screens, adaptive icons, bundle identifiers, version codes, signing, store metadata, review, and a one-line encryption declaration. Build time does not predict ship time.

2. **Privacy by architecture has a bill, and it comes due at review time.** Choosing never to collect data is the right call for a personal finance app and the author would make it again. The bill is that this dossier cannot contain a single usage metric. Knowing in advance what a decision costs you — and accepting it anyway — is different from discovering it afterwards.

3. **Tab navigators do not manage your state for you.** The most time lost to a single class of bug came from assuming a screen's lifecycle would reset it. Making "which record am I editing" explicit in the route is the durable fix.

4. **`try/catch` around `ALTER TABLE` is where a migration strategy goes to hide.** It worked for two additive columns. It would not survive a rename, a type change or a split table. A version table and ordered migrations are a day's work at the start and a rewrite later.

5. **Writing the feature document alongside the feature caught design gaps.** `BUDGET_FEATURE.md` and `DATA_PORTABILITY.md` were written as the features landed and are the reason the budget model's constraints are articulable today. The corollary: documentation that is not maintained drifts, and §8.2 is the proof.

6. **Skipping tests on a solo project is a loan, not a saving.** The pure utility functions would have cost an hour to cover. The budget-year defect at §9.2 is exactly the kind of thing a single test over `getMonthKey` plus budget resolution would have caught.

7. **Cross-platform means platform differences surface, not disappear.** iOS cannot exit programmatically; Android can. iOS wants a spinner date picker; Android wants the default. Web has no SQLite. The framework removes duplication; it does not remove the platforms.

---

## 13. Verified Career Claim Bank

> Every claim below carries its evidence level. Do not promote a level to make a claim sound better.

**C1.** Designed, built and published a cross-platform React Native (Expo) expense-tracking application to the Google Play Store and the Apple App Store as sole developer.
*Evidence: VERIFIED for design/build/configuration (59/59 commits; EAS configuration; dual-platform bundle identifiers; iOS export-compliance declaration). **USER-ATTESTED** for the publication itself.*
*Period: June–July 2025.*
*Qualification: Until Play Console or App Store Connect records are retrieved, say "published", not "published to N users".*

**C2.** Sole author of the entire codebase — 59 of 59 commits, 5,540 lines of strict-mode TypeScript across 9 screens and 12 modules.
*Evidence: VERIFIED (`git shortlog`, `wc -l`, `tsconfig.json`).*

**C3.** Designed and implemented a 5-table offline-first SQLite data layer with idempotent bootstrap, additive in-place schema evolution and backward-compatible data backfill.
*Evidence: VERIFIED (`lib/db.ts`, `lib/currency.ts`; commits `51975dd`, `39711de`).*

**C4.** Implemented a privacy-first architecture in which all financial data remains on the user's device — no backend, no authentication, no network transmission, no analytics.
*Evidence: VERIFIED by inspection of the complete dependency list and source tree. Privacy *intent* is USER-ATTESTED; the architecture is fact.*

**C5.** Built a transactional JSON import/export subsystem across five tables with per-row validation, upsert-by-id semantics, skip-and-warn tolerance of malformed rows, and `BEGIN`/`COMMIT`/`ROLLBACK` integrity — providing device-to-device portability without a server.
*Evidence: VERIFIED (`lib/dbJsonImport.ts`, `lib/dbJsonExport.ts`).*

**C6.** Engineered a hybrid local notification scheduler to deliver six reminder cadences on a platform API supporting only fixed calendar repeats, combining native daily triggers with computed one-shot date triggers re-armed from persisted state.
*Evidence: VERIFIED (`lib/notifications.ts`).*
*Qualification: the non-daily chain depends on app activity to re-arm; say so if pressed.*

**C7.** Produced at least 17 distinct Android build artefacts through an EAS Build pipeline configured with remote version sourcing and production auto-increment.
*Evidence: VERIFIED / DERIVED (`versionCode: 17`, `eas.json`).*
*Qualification: build artefacts, **not** public releases — internal and failed builds also consume version codes.*

**C8.** Implemented a multi-currency domain model (20-currency catalogue, normalised user selection, enforced single-default invariant) with per-currency filtering, budgets and aggregation throughout the application.
*Evidence: VERIFIED (`lib/currency.ts`, `lib/budget.ts`, `app/expenses.tsx`).*

**C9.** Built a performance-conscious transaction browser: year-grouped month navigation with lazily loaded historical years, day-grouped `SectionList` detail, gesture-driven edit/delete, pull-to-refresh and budget progress visualisation.
*Evidence: VERIFIED (`app/expenses.tsx`, 861 lines, 22 commits).*

**C10.** Solved cross-platform compatibility by implementing a structurally-typed mock database adapter, allowing a single codebase to run on web where the native SQLite module is unavailable.
*Evidence: VERIFIED (`lib/db.ts`).*
*Qualification: this is graceful degradation — the web build renders empty — not web feature parity.*

**C11.** Delivered a complete feature set in a 6-day build sprint (54 of 59 commits), then carried the project through a distinct release-engineering phase on a dedicated branch.
*Evidence: VERIFIED / DERIVED (commit date aggregation; `v1-golive` branch).*

**C12.** Authored user- and developer-facing documentation alongside implementation, including a feature specification for the budget subsystem and a data-portability guide.
*Evidence: VERIFIED (`README.md`, `BUDGET_FEATURE.md`, `DATA_PORTABILITY.md`).*

---

## 14. CV Achievement Candidates

### High-confidence quantified bullets

- Designed, built and shipped **Clevali**, a cross-platform expense-tracking app (React Native / Expo / TypeScript) to the **Apple App Store and Google Play** as sole developer — 5,540 lines of strict-mode TypeScript across 9 screens and a 5-table on-device SQLite data layer.
- Delivered a complete multi-currency expense tracker — 20-currency catalogue, per-currency budgets, 8 categories, 6 reminder cadences — in a **6-day build sprint (54 of 59 commits)**, then carried it through native release engineering to store submission.
- Built an **offline-first mobile architecture with no backend**: 5-table SQLite schema with idempotent bootstrap and backward-compatible migrations, plus transactional JSON import/export across all tables for device-to-device portability.
- Produced **17+ Android build artefacts** through an EAS Build pipeline with remote version sourcing and auto-increment, resolving iOS export-compliance and native configuration requirements to reach App Store submission.

### High-confidence non-quantified bullets

- Implemented a **privacy-by-architecture** design in which all personal financial data remains on-device — no backend, no authentication, no network transmission, no analytics — and solved the resulting portability problem with a transactional JSON export/import subsystem rather than a sync service.
- Engineered a **hybrid local notification scheduler** combining native daily triggers with computed one-shot date triggers to deliver reminder cadences the platform API does not natively support.
- Solved cross-platform fragmentation with a **structurally-typed mock database adapter**, keeping a single codebase runnable where the native SQLite module is unavailable.
- Built a **gesture-driven transaction browser** with year/month/day hierarchical grouping, lazily loaded historical years, swipe-to-edit/delete and live budget progress visualisation.

### Bullets requiring confirmation before use

- *"Published to the App Store and Google Play, with the Android release live for over a year"* — currently USER-ATTESTED. Strong, and worth using, but settle the dates before putting them in writing.
- *Anything citing downloads, installs, active users or ratings* — retrieve from the Play Console and App Store Connect first (§19). **Do not estimate.**

### Do not write

- Any percentage, time-saving, revenue or adoption figure. None exists.
- "Production system serving N users." It is a published app with unknown usage.
- "Full-stack mobile application." There is no backend, deliberately — and the deliberateness is the better story.

### Where to put it on your CV

Three options, in order of preference:

**1. A `Personal & Portfolio Projects` section (recommended).** A short section after your professional experience, two or three entries, each three or four lines. This is where Clevali belongs: it signals clearly that it is your own work rather than a client engagement, so nobody feels misled, while still getting the shipping claim onto page one or two. Format it like a role — name, stack, dates, then two bullets:

> **Clevali — Expense Tracker** · React Native, Expo, TypeScript, SQLite · Jun–Jul 2025
> Solo-built cross-platform mobile app, published to the Apple App Store and Google Play. Offline-first architecture with a 5-table on-device SQLite database and no backend — all financial data stays on the user's device.
> Implemented multi-currency budgeting, a hybrid local-notification scheduler, and transactional JSON import/export for device-to-device data portability.

**2. Folded into a `Technical Skills` or `Key Technologies` line as evidence.** Weaker — it reduces a shipped product to a keyword. Only do this if space is genuinely at a premium, and in that case cite it as "React Native (published iOS/Android app)" so the shipping claim survives.

**3. In your portfolio site and LinkedIn Projects section, with the CV pointing at it.** Do this *in addition* to option 1, not instead. LinkedIn's Projects section is the natural home for the longer story and screenshots; the CV bullet is the hook.

**What not to do:** don't list it under professional experience alongside paid roles. Reviewers notice, and the inference — that you padded — costs you more than the bullet gains. The project is strong enough to stand honestly as a personal project.

**One sentence worth having ready**, because interviewers ask why a tracking app has no cloud sync: *"It holds a complete picture of someone's spending, so I decided the data should never leave their phone. The cost is that I have no analytics — I genuinely cannot tell you how people used it, and I accepted that when I chose the architecture."* That answer does more for you than a download count would.

---

## 15. Portfolio Case Study Material

**Overview.** Clevali is a privacy-first personal expense tracker for iOS and Android, built solo with React Native and Expo. Every expense a user records stays in a SQLite database on their own device.

**Challenge.** Expense trackers ask for the most sensitive data most people have: where their money goes, in what amounts, how often. Most solve this with an account and a server, which makes the developer the custodian of that picture. The challenge was to build something genuinely useful — multi-currency, budgeted, with reminders and the ability to move to a new phone — without ever taking custody of a single user's data.

**Role.** Sole developer: product definition, data modelling, architecture, UI, native configuration, build pipeline and store release.

**Solution.** An offline-first architecture with a five-table SQLite schema, a multi-currency domain model, uniform annual budgets with per-month progress visualisation, a hybrid local-notification scheduler, and a transactional JSON export/import subsystem that answers the device-migration problem without a sync service.

**Interesting engineering problems** (each a section of its own in a written case study):
1. Scheduling "every N days" reminders on an API that only does fixed calendar repeats (§5.3).
2. Running one codebase where the database does not exist (§5.1).
3. Evolving a shipped schema without a migration framework — what worked and what I would change (§5.2).
4. Form identity in a tab navigator (§5.4).
5. The privacy decision and its measurement cost (§8.1).

**Results.** Published to both stores; 17+ Android build artefacts; complete feature set in a six-day sprint; zero user data ever transmitted.

**What I would improve.** Unit tests over the pure date/currency/budget logic; a schema version table and ordered migrations; per-year budget resolution in the month list (a real defect, §9.2); extracting the swipeable row into a proper component; and `__DEV__` guards around developer surfaces rather than commented-out buttons.

**What to show.** Screenshots of the month list with budget progress, the day-grouped detail view, the swipe actions, and the currency setup flow. The architecture diagram from §4 — with the crossed-out "no backend / no analytics" block kept in, because the absence is the point.

---

## 16. Interview Story Bank

### Story A — The decision that cost me my metrics (trade-offs, judgement)

- **Situation.** Building a personal expense tracker to publish publicly.
- **Task.** Decide where user financial data lives.
- **Action.** Chose on-device SQLite with no backend, no account and no analytics, then solved the resulting device-migration problem with a transactional JSON export/import rather than adding a sync service.
- **Result.** Shipped to both stores with zero user data ever transmitted — and no ability to measure usage.
- **Evidence.** No network or analytics dependency anywhere in the source; `lib/dbJsonImport.ts`.
- **Why it lands.** It is a decision with a cost the candidate names unprompted. Most candidates describe trade-offs where they only got the upside.

### Story B — Working around a platform API (problem-solving)

- **Situation.** Six reminder cadences required; the OS scheduler supports fixed calendar repeats.
- **Task.** Deliver "every 2 / 3 / 14 / 30 days" without a server to push from.
- **Action.** Hybrid scheduler — native daily trigger where it fits; elsewhere a computed one-shot date trigger derived from `last_triggered`, frequency and configured time, with same-day roll-forward, written back by both tap and foreground listeners, with a blanket cancel before each reschedule.
- **Result.** All six cadences work; the known failure mode (long inactivity breaking the chain) is documented rather than hidden.
- **Evidence.** `lib/notifications.ts`.

### Story C — Shipping is its own discipline (delivery, ownership)

- **Situation.** Feature-complete on 10 June; submitted in late July.
- **Task.** Get a working app through two store review processes alone.
- **Action.** EAS build profiles, dependency pinning, version-code management, adaptive icons and splash assets, isolating release work on a `v1-golive` branch, removing developer controls, and declaring iOS export compliance.
- **Result.** Both stores. The final commit in the repository is a one-line encryption declaration — six weeks after the last feature.
- **Evidence.** Commit dates; `eas.json`; `94d80ea`.
- **Why it lands.** It answers "have you actually shipped something?" with specifics only someone who did it would know.

### Story D — Evolving a schema already in users' hands (data, backward compatibility)

- **Situation.** Single-currency app needed multi-currency support after the schema existed.
- **Action.** Additive `ALTER TABLE` with a sensible default for existing rows, idempotent on re-run, plus a backfill `UPDATE` for the later `updated` column.
- **Result.** No data loss, no forced reinstall.
- **Honest follow-up, which should be volunteered.** "It used exception control flow as a migration strategy and had no version table. It held for two additive columns and would not have held for a rename. I would start with an ordered migration runner next time."
- **Evidence.** `lib/db.ts`; commits `51975dd`, `39711de`.

### Story E — A bug I would catch today (self-awareness, testing)

- **Situation.** Budgets are stored per year; the month list shows several years.
- **Problem.** The screen loads only the current year's budget and compares every displayed month against it, so historical months are scored against the wrong figure.
- **Action/Result.** Found during a structured review of the codebase; the fix is to resolve the budget per month's own year.
- **Lesson.** One unit test over budget resolution would have caught it. The project has none — a decision taken for speed on a solo build that this defect is the direct bill for.
- **Why it lands.** Candidates who can point at a specific defect in their own shipped code and explain the process gap that let it through are rare, and far more convincing than candidates whose projects were all flawless.

### Story F — Why I learn by shipping (motivation, growth)

- **Situation.** Not the author's first React Native app — but the first taken all the way to public release.
- **Action.** Deliberately set the goal as "it must reach the stores", not "it must work".
- **Result.** The last 10% — signing, compliance, assets, review — turned out to contain most of what he did not previously know.
- **Evidence.** USER-ATTESTED motivation; the release-phase commits corroborate it.

---

## 17. Technical Article and Blog Opportunities

**1. "I built an expense tracker with no backend — and I have no idea how many people use it."**
*Core problem:* privacy-by-architecture and its measurement cost. *Why interesting:* almost every write-up of a local-first app sells the upside; this one prices the downside honestly. *What to cover:* the decision, the JSON portability answer, and what you give up. *Anonymise:* nothing — no user data exists.

**2. "Scheduling 'every N days' notifications when the OS only offers 'daily'."**
*Core problem:* a genuine platform-API gap. *Why interesting:* highly searchable, concretely useful, and the `last_triggered` re-arming pattern generalises. *Cover:* trigger types, the computation, the same-day edge case, the chain-breakage failure mode and how you would reconcile on cold start.

**3. "Migrating a SQLite schema in a shipped mobile app without a migration framework."**
*Core problem:* evolving a database that lives on strangers' phones. *Why interesting:* most RN tutorials stop at `CREATE TABLE`. *Cover:* the idempotent bootstrap, `try/catch` `ALTER TABLE`, the backfill — and an unflinching section on why you would use a version table next time.

**4. "One codebase, three platforms, and a database that doesn't exist on one of them."**
*Core problem:* the mock-adapter pattern for platform gaps. *Cover:* structural typing against the real interface, where graceful degradation is honest and where it is a lie.

**5. "The last 10% of shipping a mobile app is 90% of what you don't know yet."**
*Core problem:* the gap between feature-complete and submitted. *Cover:* the six-day build versus the six-week release tail, version codes, adaptive icons, `ITSAppUsesNonExemptEncryption`, EAS profiles. *Why interesting:* the single most useful article for a developer about to attempt their first store submission.

**Must not publish:** nothing in this project is commercially sensitive, and no customer data exists. The only judgement call is personal: decide how much of the Global Talent visa context you want public.

---

## 18. Professional Content Opportunities (LinkedIn / X)

Each of these has a real story behind it — none is generic.

1. **The architecture screenshot post.** The §4 diagram with the crossed-out "no backend / no analytics / no auth" block. One paragraph: an expense tracker holds a complete picture of someone's spending, so I decided it should never leave their phone. *Credible because the dependency list proves it.*

2. **"I shipped an app to both stores and I can't tell you how many people use it."** The honest trade-off post. Ends on the question: how do you measure product success when you deliberately chose not to measure anything?

3. **"6 days to build, 6 weeks to ship."** Commit-date graph. What lived in the gap: signing, icons, version codes, export compliance. *Credible because the Git history is the evidence.*

4. **The `ITSAppUsesNonExemptEncryption` post.** A short, genuinely useful one: here is the one-line declaration that stops App Store submissions stalling, and why an app with no cryptography still has to declare it.

5. **"A bug in my own shipped code."** The budget-year defect (§9.2): what it was, why zero tests let it through, what it cost. *Engineers trust people who publish their own defects.*

6. **"Not my first React Native app — my first one that shipped."** The reflection post on why finishing is the skill. *Only works because it is true.*

**Do not post:** anything implying scale, traction or user numbers. There are none, and inventing them would undermine every other post on this list.

---

## 19. Evidence Still Worth Collecting

| # | Missing evidence | Why it matters | Where to find it | What to ask / do |
|---|---|---|---|---|
| 1 | **Play Store install count, active installs, ratings, reviews, listing dates** | The one genuine production metric available. Even a modest install count turns C1 from attested to verified, and a listing date range proves the "~1 year live" claim. | **Google Play Console** → *Statistics* and *Release overview*, for `com.techfric.expensetracker` under the developer account. Retention and acquisition data is kept even after a listing is removed. | Sign in and export the statistics report. Capture: total installs, peak active installs, first publish date, last available date, ratings. |
| 2 | **App Store Connect units, impressions, publish dates** | Same, for iOS; also settles whether the iOS listing is still live. | **App Store Connect** → *App Analytics* / *Sales and Trends*. Accessible even if the developer subscription has lapsed, though the app will be removed from sale. | Check whether the Apple Developer Program membership can be viewed in a lapsed state; if the data is gone, record that explicitly rather than guessing. |
| 3 | **Exact store publication dates (both platforms)** | Several CV bullets depend on a defensible date range. The repository only proves the build window (Jun–Jul 2025). | Play Console release history; App Store Connect version history; store confirmation emails in the author's inbox. | Search email for "Google Play" and "App Store Connect" around July–August 2025. |
| 4 | **EAS build history** | Would convert `versionCode: 17` from an inference about build count into a verified record of builds and submissions, with dates. | `eas build:list` / `eas submit:list` against EAS project `2877daaf-…`, Expo account `noblexeyii`. | Run the CLI commands while signed in and save the output. |
| 5 | **Screenshots / store listing assets** | Needed for the portfolio case study; also the only visual record if the listings are gone. | Store consoles; the author's device; local asset files. | Retrieve before the listings lapse entirely, if they have not already. |
| 6 | **Any user feedback** | Even three reviews would be the only external signal this project has. | Play Console reviews; App Store ratings; messages from anyone the author shared it with. | Check reviews before the listing is removed. |
| 7 | **Global Talent application context** | If the app was cited as evidence in the application, the outcome is itself career evidence. | The author's own records. | Note privately whether it was submitted as supporting material and what the outcome was. |

**Until items 1–3 are retrieved, every store-related claim in this dossier must stay labelled USER-ATTESTED.**

---

## 20. Private vs Public-Safe Information

### Private career evidence (own records, interviews, CV decisions)

- The Global Talent visa motivation, unless the author chooses to make it public.
- The known-weaknesses list (§9) — excellent interview material when volunteered by the candidate, poor portfolio material when published unprompted.
- Any store console figures once retrieved, until the author decides which are worth publishing.
- The author's assessment of the app's low download volume.
- The Expo account identifier, EAS project ID and developer-account details.

### Public-safe material

- Everything in §4 (architecture), §5 (engineering implementation), §8 (decisions and trade-offs), §11 (problems solved), §12 (lessons) and §17–18 (content ideas).
- All repository metrics in §7 — commits, lines, tables, dates. All verifiable, none sensitive.
- The fact of publication to both stores, phrased as attested until §19 items 1–3 land.
- The source code itself — the repository is already public on GitHub, contains no secrets, no `.env` file, and no user data.
- The sample data in `exported-data.json` is synthetic developer data, not personal information.

**No credentials, API keys, tokens, connection strings or personal data appear anywhere in this repository or in this dossier.** The project has no `.env` file because it has no services to authenticate against. *(VERIFIED.)*

---

## 21. Audit Limitations

1. **No production data was available or queryable**, because none exists by design (§7). This audit therefore evaluates the artefact and its history, not its usage.
2. **Store publication, listing duration and the visa motivation are USER-ATTESTED** and were not independently verified. §19 sets out how to verify them.
3. **The audit was performed by static inspection.** The application was not built or run during this audit; behavioural claims derive from reading the source, and the defects at §9 were identified by inspection rather than by execution.
4. **`node_modules` and `package-lock.json` were excluded** from code metrics.
5. **The `v1-golive` branch was audited.** `main` lacks the final two release commits.
6. **The working tree had uncommitted modifications** to `app.json` and two asset files at audit time; these appear to be icon and configuration adjustments and were not analysed as history.

---

## Appendix A — Evidence Commands

```bash
git shortlog -sne --all                    # authorship: 59 commits, 1 author
git rev-list --count HEAD                  # 59
git log --format='%ad' --date=short | sort | uniq -c   # commits per day
git log --numstat --format='' -- '*.ts' '*.tsx' '*.js' '*.json' ':!package-lock.json'
wc -l app/*.tsx lib/*.ts                   # 5,540 lines
git log --name-only --format='' | sort | uniq -c | sort -rn   # churn by file
git log --oneline main..v1-golive          # 2 release commits
```

## Appendix B — Commit Timeline (selected)

| Date | Commit | Significance |
|---|---|---|
| 2025-06-05 | `6fa1c2d` | Initial commit |
| 2025-06-05 | `2d6ba76` | Restructured to Expo Router — the routing decision, made on day one |
| 2025-06-05 | `9e4b432` | Database initialisation and expense management |
| 2025-06-06 | `f373b4c` | Tab navigation and add-expense form with validation |
| 2025-06-06 | `51975dd` | **Multi-currency support added to an existing schema** |
| 2025-06-06 | `8d34744` | Gesture handler — swipeable expense rows |
| 2025-06-06 | `39711de` | `updated` column added with backfill of existing rows |
| 2025-06-07 | `a41366a` | JSON import |
| 2025-06-07 | `ac49f3c` | Transactional data wipe with confirmation |
| 2025-06-07 | `69d2cc8` | **Uniform monthly budget feature** |
| 2025-06-07 | `70e1715` | Lazily loaded historical years with placeholder months |
| 2025-06-08 | `02703c1` | **Reminder settings and notification scheduling** |
| 2025-06-08 | `4422e31` | Currency setup screen and user currency management |
| 2025-06-10 | `5c8aeb4` | Renamed `tracking-react-native` → `expense-tracker` |
| 2025-07-13 | `2fef9f6` | **EAS build configuration** — release phase begins |
| 2025-07-13 | `b7f8549` | Developer-only controls hidden before submission |
| 2025-07-22 | `94d80ea` | **iOS export-compliance declaration** — final commit |

---

*End of dossier. Prepared 17 October 2026 from static analysis of the `v1-golive` branch at commit `94d80ea`, plus first-hand context supplied by the author.*
