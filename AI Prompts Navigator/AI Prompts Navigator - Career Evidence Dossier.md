# AI Prompts Navigator — Career Evidence Dossier

**Audit date:** 17 October 2026
**Project type:** Self-initiated personal product — cross-browser extension (Manifest V3), published to public add-on stores
**Local path:** `~/Documents/Coding/portfolio-projects/ai-prompts-navigator`
**Artefacts audited:** `ai-prompts-navigator-chromium/`, `ai-prompts-navigator-mozilla/`, `landing-page/ai-prompt-navigator-landing/` (git repo), and three dated development archives (`.zip`) preserving earlier iterations
**Public listings verified (all three, 17 October 2026):**
- Chrome Web Store — https://chromewebstore.google.com/detail/ai-prompts-navigator/ajodgicjemafjmbgkjbhfogdpfejihjg (offered by *abramseyakid*)
- Microsoft Edge Add-ons — https://microsoftedge.microsoft.com/addons/detail/ai-prompts-navigator/abcopdbmckchogodlkknclocabccidie (publisher *Clevali*)
- Firefox Add-ons — https://addons.mozilla.org/en-US/firefox/addon/ai-prompts-navigator/ (publisher *Clevali*)

> **Revision note.** This dossier was first produced on 17 October 2026 with the Chrome and Edge listings recorded as *UNKNOWN / REQUIRES CONFIRMATION*, because neither could be located by public search. The project owner supplied both listing URLs the same day; all three stores were then measured directly and this document was revised. Every Chrome and Edge figure below is now **VERIFIED**.

---

## 1. Executive Summary

**AI Prompts Navigator** is a cross-browser extension that solves a small, real, universal annoyance: long AI chat conversations have no table of contents. The extension injects a collapsible side panel into **ChatGPT, Claude, Google Gemini, DeepSeek and Grok**, lists every prompt the user has sent in the current conversation as a numbered, clickable outline, and scrolls smoothly to any of them on click, with a hover preview of the full prompt text.

**Role: sole author.** This was a personal product idea, designed, built, packaged, branded, documented and published by one person. There is no second contributor anywhere in the extension artefacts. The only non-personal authorship in the whole project is in the *landing page* repository, where an AI site-builder (Lovable / `gpt-engineer-app[bot]`) produced the initial scaffold and the user then took over (see §3).

**Most technically significant aspect.** The extension is, in effect, a **resilience layer over five proprietary, obfuscated, frequently-changing single-page applications it does not control and cannot test against in CI.** Everything interesting in the codebase exists to survive that: a prioritised selector cascade with per-platform text-extraction fallbacks, per-platform conversation-ID parsing, SPA route-change detection without access to history hooks, layered retry ladders for interfaces that are still hydrating, and aggressive style isolation against hostile host-page CSS.

**Strongest verified production evidence.** The extension is **live on all three major extension stores and has been for approximately 18 months**, with a **combined store-reported user count of 64** measured on 17 October 2026: **24 active installs on Microsoft Edge**, **21 users on the Chrome Web Store** (rated **5.0 from 2 ratings**), and **19 average daily users on Firefox** (rated **4.0 from 1 review**). Each store counts users on its own definition, so 64 is a sum of three differently-defined figures rather than a single measured population (§8.1). It is a modest number, and it is real — three independent review processes passed, and the software has kept running in users' browsers for a year and a half without a single update.

**What this project does *not* evidence.** There is no database, no backend, no telemetry and no analytics — **by deliberate design** (see §9), which means there is no usage instrumentation of any kind: no funnel, no retention curve, no breakage alerting. The adoption figures above come from the stores, not the product. There is also no version-control history for the extension code itself, no automated tests, and no update since April 2025.

**Clearest career evidence.** This is strong, defensible evidence of **end-to-end independent product delivery**: identifying a problem, building a working browser extension against five third-party interfaces, porting it across two divergent extension platforms, narrowing its permission surface for store review, writing a privacy policy and a marketing site, and getting it through public store review to real installed users — all solo, in roughly one month of elapsed work.

---

## 2. Project Overview

| Attribute | Detail |
|---|---|
| Human-readable name | AI Prompts Navigator |
| Earlier names (evidenced) | *ChatGPT Quick Jump* → *AI Chat Quick Jump* → *AI Prompts Navigator* |
| Type | Browser extension, Manifest V3 |
| Version | 1.0.0 |
| Target browsers | Chrome / Chromium / Edge (Chromium build), Firefox ≥ 109 (Gecko build) |
| Supported host sites | `chatgpt.com`, `claude.ai`, `gemini.google.com`, `chat.deepseek.com`, `grok.com` |
| Core code size | `content.js` — 1,684 lines / 62,243 bytes (byte-identical across both builds) |
| Backend | **None.** No server, no database, no network requests |
| Packaged size (AMO listing) | 91.39 KB |
| Build window (file/archive + store evidence) | 15 March 2025 → 21 April 2025 |
| Public listings (all verified) | Chrome Web Store (`ajodgicjemafjmbgkjbhfogdpfejihjg`), Edge Add-ons (`abcopdbmckchogodlkknclocabccidie`), Firefox Add-ons (`ai-prompts-navigator`) |
| Author (manifest `author` field) | Abraham Akintola |
| Store publisher names | *Clevali* (Edge, Firefox); *abramseyakid* (Chrome) |
| Licence (AMO listing) | All Rights Reserved |

### What it does, end to end

1. The content script loads on any of the five supported origins at `document_end`.
2. It detects the platform from `window.location.hostname` and parses a **conversation ID** out of the URL using a per-platform pattern.
3. It injects a fixed-position side panel (off-screen at `right: -320px`) plus a draggable launcher button whose position persists in `localStorage`.
4. On open, it walks the live DOM with a **prioritised selector cascade** to find the user's own messages, renders each as a numbered button with truncated preview text, and wires click → `scrollIntoView({behavior:"smooth", block:"center"})` plus a temporary highlight on the target message.
5. A `MutationObserver` on the chat container watches for new user messages and incrementally refreshes the list; a second `MutationObserver` on `document.body` detects SPA URL changes and resets state for the new conversation while preserving whether the panel was open.
6. Hovering a list item populates a single shared tooltip element with the full prompt text, repositioned to avoid the viewport edges.

---

## 3. Business Problem, Context and My Role

### The problem

Every major AI chat product renders a conversation as an unbounded vertical scroll with no index. A research or debugging session of thirty or forty exchanges becomes unnavigable: finding "the question where I asked about X" means scrolling and skim-reading. None of the five platforms shipped a prompt outline at the time of build (early 2025). The problem is platform-agnostic, which is exactly why a browser extension — rather than a feature request to five separate vendors — is the right shape of solution.

### Role and contribution

**Verified personal contribution — extension (100%).** All extension artefacts are the user's work. Evidence: `manifest.json` `"author": "Abraham Akintola"`; `homepage_url` and AMO support URL both point to a gist under `AbrahamNobleOX`; AMO support email is the same address as the landing-page git commits (`abramseyakid@gmail.com`); there is no second contributor, no vendored library and no package manifest anywhere in either extension build. The extension has **zero runtime dependencies** — it is hand-written vanilla JavaScript against the WebExtensions and DOM APIs.

**Verified personal contribution — landing page (partial, and this must be stated honestly).** The landing-page repository (`AbrahamNobleOX/ai-prompt-navigator-landing`) contains 19 commits: **11 by `gpt-engineer-app[bot]`** (the Lovable AI site builder) between 19:32 and 20:14 on 3 April 2025, and **8 by Akintola Abraham** on 4 April 2025. The AI tool produced the Vite + React + TypeScript + Tailwind + shadcn/ui scaffold and a first pass at the sections. The user's own commits then did the work that made it a shipped site: rebranding it to AI Prompts Navigator, adding dependencies, restructuring the Header / Footer / Hero / About / Download / FAQ / Privacy components, rewriting the CSS variables and Tailwind theme, authoring the privacy-policy content, and adding a **GitHub Actions workflow that builds and deploys to GitHub Pages** on push to `main`. *Classification: the scaffold is AI-generated; the branding, content, component restructuring and deployment pipeline are personally authored.* The dossier makes no claim that the landing page was built from scratch by hand.

**Ownership scope personally exercised:** product idea → UX design → implementation → cross-browser porting → icon and brand assets → privacy policy → marketing site → CI/CD for the site → store submission and review response → published product.

---

## 4. System Architecture

```
┌──────────────────────── Browser ────────────────────────┐
│                                                          │
│  ┌─ Toolbar action (icon) ─────────────────────────────┐ │
│  │  click → background.js                              │ │
│  └───────────────────────┬─────────────────────────────┘ │
│                          │                                │
│  ┌───────────────────────▼─────────────────────────────┐ │
│  │  BACKGROUND                                          │ │
│  │  Chromium: MV3 service_worker                        │ │
│  │  Firefox:  MV3 background.scripts (event page)       │ │
│  │                                                      │ │
│  │  • host allow-list check on tab.url                  │ │
│  │  • toggles panelVisible                              │ │
│  │  • swaps action icon (enabled / -disabled variants)  │ │
│  │  • Firefox only: persists state to chrome.storage    │ │
│  │  • off-platform feedback:                            │ │
│  │      Chromium → alert()                              │ │
│  │      Firefox  → chrome.notifications.create()        │ │
│  └───────────────────────┬─────────────────────────────┘ │
│            chrome.tabs.sendMessage / runtime.onMessage    │
│  ┌───────────────────────▼─────────────────────────────┐ │
│  │  CONTENT SCRIPT  (content.js, 1,684 lines)           │ │
│  │  injected at document_end into 5 host origins        │ │
│  │                                                      │ │
│  │  ┌ Platform detection (hostname)                    │ │
│  │  ┌ extractChatId()  ── per-platform URL patterns    │ │
│  │  ┌ findChatQuestions() ── selector cascade,         │ │
│  │  │     per-platform text extraction, DOM render     │ │
│  │  ┌ createPanel() / addDebugButton() ── injected UI  │ │
│  │  ┌ observeChat()     ── MutationObserver on chat    │ │
│  │  ┌ bodyObserver      ── MutationObserver for SPA    │ │
│  │  │                      route changes               │ │
│  │  └ retry ladders     ── 5× / 2s, plus 500/1000/     │ │
│  │                         1500 ms post-navigation     │ │
│  └───────────────────────┬─────────────────────────────┘ │
│                          │ reads + mutates                │
│  ┌───────────────────────▼─────────────────────────────┐ │
│  │  HOST PAGE DOM (ChatGPT / Claude / Gemini /          │ │
│  │                 DeepSeek / Grok)                     │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
│  LOCAL STATE ONLY:                                        │
│   • localStorage → launcher button x/y position           │
│   • chrome.storage.local (Firefox) → panel open/closed    │
│                                                           │
│  ✗ No server   ✗ No database   ✗ No network requests      │
│  ✗ No analytics   ✗ No third-party scripts                │
└───────────────────────────────────────────────────────────┘

        SEPARATE ARTEFACT ─ marketing site
        Vite + React 18 + TS + Tailwind + shadcn/ui
        → GitHub Actions → GitHub Pages
```

**Verification of the "no network" claim:** `grep` across `content.js` for `fetch(`, `XMLHttpRequest`, `.send(`, `analytics` and `googletagmanager` returns **zero matches**. The only persistence anywhere in the extension is two `localStorage` calls (button position) and two `chrome.storage.local` calls in the Firefox background script (panel state). This is a *verifiable* privacy posture, not a marketing claim — which matters because the product's input is the user's own prompt text.

---

## 5. Technology Stack

| Layer | Technology |
|---|---|
| Extension platform | WebExtensions / Manifest V3 (Chromium + Gecko) |
| Language | Vanilla JavaScript (ES2017+), zero dependencies |
| APIs used | `chrome.action`, `chrome.tabs`, `chrome.runtime` messaging, `chrome.storage.local`, `chrome.notifications`, `chrome.runtime.getURL` |
| DOM techniques | `MutationObserver` (×3 roles), `querySelectorAll` cascades, `scrollIntoView`, `getBoundingClientRect`, `requestAnimationFrame` |
| Styling | Inline style assignment (primary, for isolation) + `styles.css` (secondary) |
| Persistence | `localStorage`, `chrome.storage.local` |
| Packaging | Hand-assembled zip bundles per store |
| Marketing site | Vite 5, React 18, TypeScript 5, Tailwind 3, shadcn/ui, Radix UI, React Router |
| Site CI/CD | GitHub Actions (`actions/setup-node@v4` → `npm ci` → `npm run build` → `upload-pages-artifact@v3` → `deploy-pages@v4`) |
| Site hosting | GitHub Pages (`gh-pages` branch present on origin) |

---

## 6. Engineering Implementation — the parts that were actually hard

### 6.1 Selector cascade across five hostile, obfuscated DOMs

This is the heart of the product. There is no API for "the user's messages in this conversation"; there is only whatever markup each vendor happens to emit this week. The implementation tries five selectors in priority order and stops at the first that yields elements:

| Platform | Selector | What it tells you |
|---|---|---|
| ChatGPT | `[data-message-author-role="user"]` | Semantic, stable — the best case |
| Claude | `[data-testid="user-message"]` | A *test* hook, repurposed as a product hook — stable only by the vendor's convenience |
| Gemini | `user-query-content` | A **custom element tag name** — Angular web-component internals |
| DeepSeek | `.fbb737a4` | A **build-hashed CSS class** — guaranteed to change on the vendor's next deploy |
| Grok | `.whitespace-pre-wrap` | A **Tailwind utility class** — broad, non-semantic, high false-positive risk |

Each tier has its own text-extraction branch (`textContent` → `innerText` → nested `.whitespace-pre-wrap` lookup), because the markup shapes behind those selectors are not equivalent. The `.fbb737a4` and `.whitespace-pre-wrap` cases are the honest ones to talk about in an interview: they are **knowingly fragile** choices made because the vendor exposed nothing better, and the mitigation is architectural (fail soft, retry, show a Refresh button, never throw) rather than selectorial.

**Fail-soft behaviour is explicit:** when every selector returns nothing, the panel renders "No user messages found" plus a styled **Refresh** button wired back to `findChatQuestions`, rather than rendering an empty panel or an error. Every selector attempt is individually wrapped in `try/catch` so one malformed selector cannot abort the cascade.

### 6.2 Conversation identity across five different URL schemes

`extractChatId()` parses five different URL shapes and falls back to a named sentinel per platform rather than `null`:

| Platform | Pattern | Fallback |
|---|---|---|
| ChatGPT | `/c/<id>` | `chatgpt-home` |
| Claude | `/chat/<id>` | `claude-home` |
| Gemini | `/app/<id>` **plus `location.hash`** | `gemini-home` |
| DeepSeek | `/chat/s/<id>` | `deepseek-home` |
| Grok | `/chat/<id>` | `grok-home` |

The Gemini case is a genuine piece of observed-behaviour engineering: Gemini keeps the same path across some conversation switches and only changes the hash fragment, so the hash is concatenated into the identity key. Without it, switching conversations in Gemini would leave a stale prompt list on screen. The sentinel fallbacks (rather than `null`) make the "landed on the home page with no conversation" case a *distinct, comparable* state, so the change-detection logic treats home→conversation as a real transition.

### 6.3 SPA navigation detection without history hooks

A content script cannot cleanly hook a host SPA's router. The solution: a `MutationObserver` on `document.body` with `{childList:true, subtree:true}` that compares `location.href` to a cached value on every mutation batch — using DOM churn as a proxy clock for route changes. On a detected change the code captures `wasPanelVisible` *before* mutating any state and restores it afterwards, so a user who switches conversations with the panel open does not have to reopen it.

**Trade-off, stated honestly:** observing the entire body subtree for mutations on a page that streams tokens character-by-character is not free. The callback is cheap (a string comparison and an early return) which is what makes it viable, but this is the design decision in the codebase most open to a reviewer's challenge, and the defensible answer is "it was the only mechanism available that works across all five platforms without per-vendor router knowledge."

### 6.4 Retry ladders for interfaces that are still hydrating

Three independent retry mechanisms exist because the failure happens at three different times:

- `initializeWithRetry(attempt, maxAttempts=5)` — 5 attempts, 2 s apart, gated on a loading-indicator probe (`.loading, .spiner, [role="progressbar"]`), for first load.
- `startObserving(attempt, maxAttempts=5)` — same cadence, for the case where the chat *container* has not yet rendered so there is nothing to observe.
- A hand-unrolled **500 ms / 1,000 ms / 1,500 ms** triple retry after a conversation change, each conditional on the previous attempt having found zero prompts.

Plus four redundant initialisation triggers: immediate call, `setTimeout(1500)`, `window.onload`, and `DOMContentLoaded` — guarded by an `isInitialized` flag so only one wins. This is defensive to the point of being inelegant, and it is the correct instinct for an environment where you cannot reproduce the user's timing.

### 6.5 Style isolation in a hostile host page

The panel must render correctly inside five apps with aggressive global CSS, each with its own light/dark theming. The strategy is deliberate and visible in the code:

- **Inline styles as the primary mechanism** — `styles.css` exists (157 lines) but the live panel, tooltip, buttons and list items are all styled by direct `element.style.*` assignment, which outranks any host-page class rule without needing `!important` everywhere.
- `z-index: 9999999` and `position: fixed` to escape host stacking contexts entirely.
- A **full font stack declared on every injected element** (`'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', …`) rather than inheriting, so the panel looks identical on all five platforms instead of adopting each vendor's typography.
- Absolute-positioned scroll container with explicit `top: 60px; bottom: 0` and matching `height: calc(100% - 60px)` — a belt-and-braces approach to a scrolling bug that the code comments record as having been fought ("ULTRA SIMPLE STRUCTURE — no flex layouts, fixed positioning only", "ABSOLUTELY SIMPLEST APPROACH POSSIBLE FOR MAXIMUM RELIABILITY"). Those comments are themselves evidence of a debugging episode that ended in deliberately choosing the dumbest layout that could not break.

### 6.6 One shared tooltip instead of N

Rather than attaching a tooltip element per list item, a **single** `#ai-prompts-navigator-tooltip` node is created once and repopulated on hover. It carries a sticky header with a "Question *n*" counter and a close button, a scroll-overflow indicator that self-dismisses after 3 s, and post-render **viewport collision handling**: after it becomes visible, its `getBoundingClientRect()` is re-measured and it flips to the opposite side of the panel if it would overflow the right edge, or is pushed up if it would overflow the bottom. There is also a 50 ms grace delay on `mouseout` that checks whether the cursor has moved *onto* the tooltip, so a scrollable tooltip can actually be scrolled — a detail that only appears after someone has used the thing properly.

### 6.7 Cross-browser port: MV3 is not one platform

The Chromium and Firefox builds share a **byte-identical `content.js` and `styles.css`**; all divergence is isolated to `manifest.json` and `background.js`. The port was not cosmetic:

| Concern | Chromium build | Firefox build |
|---|---|---|
| Background | `background.service_worker` | `background.scripts` (Gecko MV3 event page) |
| State across background restarts | module-level `let panelVisible` | `chrome.storage.local`, read on startup, written on each toggle |
| Off-platform user feedback | `alert()` | `chrome.notifications.create()` — because `alert()` is unavailable in a worker/event-page context |
| Async handling | callback style | `async`/`await` with `try/catch` around `setIcon` + `storage.set` |
| Extra permissions | — | `scripting`, `storage`, `notifications` |
| Gecko metadata | — | `browser_specific_settings.gecko` with explicit ID and `strict_min_version: "109.0"` |

The state-persistence change is the substantive one: an MV3 background context is **ephemeral**, so any state held in a module-level variable silently resets when it is torn down. The Chromium build still carries that latent bug; the Firefox build fixed it. That is a real, demonstrable diagnosis of a platform-lifecycle issue, and an honest answer to "what would you fix first?"

### 6.8 Permission narrowing for store review

Comparing the 19 March 2025 archive manifest with the shipped Chromium manifest shows a deliberate reduction of the requested privilege surface:

| | March 2025 (dev) | Shipped Chromium |
|---|---|---|
| Permissions | `activeTab`, `scripting`, `tabs` | `activeTab`, `tabs` |
| Host permissions | 6 origins (incl. legacy `chat.openai.com`) | 5 origins |
| `web_accessible_resources` | **10 JavaScript module files** exposed to the page | `icons/*` only |
| Content scripts | 11 files | 1 file |

Exposing ten script files as web-accessible resources is a genuine (if minor) attack-surface issue and a predictable store-review flag; the shipped build exposes only icons. Dropping `scripting` matters too — it is the permission that produces the scariest install-time warning, and it was unnecessary once the content script was declared statically. This is concrete evidence of **security-aware packaging for public distribution**, not just "it worked locally."

**An important qualification from the store data.** The Edge Add-ons listing reports its published package's permissions as `activeTab`, `scripting`, `tabs` — i.e. the **earlier** set. The narrowed manifest (local mtime 17 April 2025) is therefore *later than the Edge submission*, and the Chrome listing — last updated **21 April 2025**, four days after that manifest was finalised — is the one most likely to carry it. The Chrome listing page does not expose its permission list, so this cannot be confirmed from outside. **Defensible phrasing:** the narrowing is verified in the final packaged Chromium build and the consolidation to a single content script is verified in both shipped builds; *which* store received *which* permission set is partly inferred. Do not claim all three stores shipped the minimum-privilege manifest.

---

## 7. Architectural Decisions and Trade-offs

**Decision 1 — Modular split, then deliberate re-consolidation into one file.**
*Problem:* the March build had grown to ~50 KB of content script.
*Decision:* it was split into **ten modules** (`ui-utils.js`, `panel.js`, `scroll-helper.js`, `questions-ui.js`, `debug-button.js`, `url-observer.js`, `chat-observer.js`, `chat-helper.js`, `questions-helper.js`, `panel-toggle.js`) on 19 March 2025 — then **reverted to a single `content.js`** by 5 April 2025.
*Reasoning (inferred from the manifest diff, not from a written record):* MV3 content scripts are not ES modules; sharing state across ten plain scripts means ten globals and a load-order contract in the manifest, and every module had to be listed in `web_accessible_resources`. The single file removed the ordering fragility and the exposure in one move.
*Trade-off:* a 1,684-line file with no module boundaries, which is the project's main maintainability cost.
*Result:* a simpler, smaller, safer package. *Classification: decision VERIFIED by manifest comparison; reasoning INFERRED.*

**Decision 2 — Inline styles over a stylesheet.** Chosen for guaranteed specificity inside uncontrolled host pages. Trade-off: styling is scattered through logic, there is no theming layer, and `styles.css` became partly vestigial. A Shadow DOM root would be the modern answer and would give isolation *and* a clean stylesheet; it was not used.

**Decision 3 — No backend, no telemetry, by design.** The privacy policy states that conversations, browsing history, PII and cookies are not collected, and the code corroborates it (zero network calls). Trade-off, and it is a real one: **the product has no idea how it is used.** There is no funnel, no retention curve, no selector-breakage alerting. Every adoption figure in this dossier comes from the three stores, not from the product — which is why they are all point-in-time counts with no trend behind them. This dossier's thin metrics section is a *direct consequence of a defensible privacy decision* — a genuinely interesting trade-off to be able to discuss.

**Decision 4 — Hard-coded vendor selectors with no remote config.** Selectors live in the shipped bundle, so a vendor DOM change can only be fixed by a new store release and a review cycle. A remotely-fetched selector manifest would fix same-day — at the cost of a server, a network request on every page, and the end of the "no network calls" privacy claim. The trade-off was resolved in favour of privacy. *Classification: trade-off VERIFIED from code; the reasoning is INFERRED.*

**Decision 5 — Toolbar action toggles rather than `chrome.sidePanel`.** A self-injected fixed-position panel works identically on Chromium and Firefox; `chrome.sidePanel` is Chromium-only and would have forked the UI layer. The shared, byte-identical `content.js` across both builds is the pay-off.

---

## 8. Production Evidence and Statistical Analysis

> **Scope note.** The audit prompt's production-database safety protocol was not applicable: **this project has no `.env` file, no credentials, no connection string, no database and no backend of any kind.** No queries were run because there is nothing to query. All metrics below come from public store listings.

### 8.1 Verified metrics — all three stores, measured 17 October 2026

| Store | Users | Metric definition | Rating | Version | Size | Last updated | Identifier |
|---|---|---|---|---|---|---|---|
| Microsoft Edge Add-ons | **24** | `activeInstallCount` — active installations | none (0 ratings) | 1.0.0 | — | 6 Apr 2025 (per listing API) | `abcopdbmckchogodlkknclocabccidie` |
| Chrome Web Store | **21** | "users" as displayed on the listing | **5.0 from 2 ratings** | 1.0.0 | 86.73 KiB | 21 Apr 2025 | `ajodgicjemafjmbgkjbhfogdpfejihjg` |
| Firefox Add-ons | **19** | `average_daily_users` — rolling daily actives | **4.0 from 1 review** | 1.0.0 | 91.39 KB | 16 Apr 2025 | `ai-prompts-navigator` |
| **Combined** | **64** | **Sum of three different definitions — see caveat** | 3 ratings total, avg 4.67 | — | — | — | — |

**The caveat matters and should always travel with the number.** Edge reports *active installs*, Chrome reports a *user* count on its own undisclosed basis, and Firefox reports *average daily users*. These are three different measurements, so **64 is a sum of three store-reported figures, not a measured population of 64 distinct people.** It is also a point-in-time reading, not a cumulative install total — none of the three stores publishes lifetime installs to a non-owner.

**Defensible phrasing:** *"Published to all three major browser extension stores; store-reported user counts totalled 64 across Chrome, Edge and Firefox when measured in October 2026, roughly 18 months after release."* Not defensible: "64 active users", "64 people use it daily", any cumulative install figure, or any growth/retention rate.

**Ratings across all stores: 3 ratings total** (Chrome 5.0×2, Firefox 4.0×1, Edge none) — a mean of 4.67. With n=3 this is directionally positive and statistically meaningless; always state the sample size.

### 8.1.1 Firefox detail

Measured 17 October 2026 via the public AMO API (`/api/v5/addons/addon/ai-prompts-navigator/`) and cross-checked against the rendered listing page.

| Metric | Value | What it means | Limitation |
|---|---|---|---|
| Average daily users | **19** | AMO's rolling count of installs active per day | A point-in-time figure at audit date; AMO does not expose the historical series publicly, so peak and cumulative installs are unknown |
| Weekly downloads | **1** | New installs in the trailing week | The product has not been marketed or updated since April 2025 |
| Rating | **4.0 from 1 review** | One user left 4 stars with written text | n=1; statistically meaningless, directionally positive |
| Listing created | **13 Apr 2025, 09:48 UTC** | Date submitted/published | — |
| Last updated | **16 Apr 2025** | Last version change | — |
| Listed version | **1.0.0** | Never superseded | — |
| Package size | **91.39 KB** | Shipped bundle | — |
| Listing lifespan | **~18 months** (13 Apr 2025 → 17 Oct 2026) | Continuously publicly available; passed review and has not been taken down | **DERIVED** from created date vs audit date |

### 8.2 What the Chrome and Edge listings add beyond the user counts

Three findings from the store data are worth more than the raw numbers:

1. **Three independent review processes were passed.** Chrome, Edge and Mozilla each operate their own add-on review, and each examines permissions, packaging and privacy claims separately. Three approvals is meaningfully stronger evidence of competent, compliant packaging than one, and it is external third-party validation rather than self-assessment.
2. **The build-and-publish window extends to 21 April 2025.** The Chrome listing's last-updated date is four days later than the final local `manifest.json` mtime, pushing the total elapsed project window from 33 to **37 days** (§8.3).
3. **Edge shipped the earlier permission set.** The Edge listing reports `activeTab`, `scripting`, `tabs` — the pre-narrowing manifest. This is a genuine constraint on the permission-narrowing claim and is handled in §6.8; it also means the three stores do not all carry identical packages, which is itself a small, honest detail about managing parallel store submissions.

**One anomaly, recorded rather than explained away.** The Edge listing API reports a last-update date of **6 April 2025**, which is *earlier* than the Firefox listing's creation (13 April) and earlier than the final Chromium manifest (17 April). That date matches the mtimes of `background.js`, `styles.css` and the icon set in the local Chromium build, so it most likely reflects the **packaged build date** rather than a listing-update date — but this is **INFERRED**, not verified. It does not affect any user-count figure. If the exact Edge publication date matters later, Partner Center will show it.

### 8.2.1 Firefox-specific phrasing

*"Published a browser extension to the Firefox Add-ons store that has been continuously available for ~18 months and reports 19 average daily users."* Not defensible for any single store: a cumulative install total, a growth rate, or a retention figure — none of the three stores publishes that data to a non-owner, and the extension collects none itself.

### 8.3 Development-effort evidence (DERIVED from file and archive timestamps)

| Date | Evidence | Milestone |
|---|---|---|
| 15 Mar 2025 | `README.md` inside archive | Project starts as **ChatGPT Quick Jump**, ChatGPT-only |
| 16 Mar 2025 | `styles.css` | Panel styling |
| 18 Mar 2025 | `content.js.original` (53 KB), `content.js.bak` (50 KB) | Substantial single-file implementation already exists |
| 19 Mar 2025 | Manifest listing 10 JS modules; renamed **AI Chat Quick Jump**; 6 origins incl. `chat.openai.com` | Multi-platform support + modular refactor |
| 22 Mar 2025 | `js/panel-toggle.js` | Modular era continues |
| 30 Mar 2025 | `README.md` (2,307 B), `enhanceUrlTracking.js` | Renamed **AI Prompts Navigator**; SPA URL-tracking work |
| 3 Apr 2025 | 11 Lovable bot commits; privacy policy "Last Updated" | Landing page scaffolded; privacy policy written |
| 4 Apr 2025 | 8 personal commits incl. GitHub Pages workflow | Landing page rebranded, restructured, CI/CD added |
| 5–7 Apr 2025 | Single-file `content.js` (62 KB), icon set, Chromium package | Re-consolidation; Chromium build finalised |
| 13 Apr 2025 | Firefox `background.js` + `manifest.json` rewritten; `ai-prompts-navigator.zip` built; **AMO listing created 09:48 UTC** | Firefox port and submission, same day |
| 16 Apr 2025 | AMO `last_updated` | Post-submission revision |
| 17 Apr 2025 | Chromium `manifest.json` last modified | Minimum-privilege Chromium manifest finalised |
| 21 Apr 2025 | **Chrome Web Store listing last updated** (verified from the live listing) | Final store submission — the last recorded activity on the project |

**Derived:** approximately **37 days** from first artefact to final store update, for a solo evenings-and-weekends personal project — covering three product renames, a multi-platform expansion from one AI service to five, a modular refactor and its reversal, a cross-browser port, a marketing site with CI/CD, a privacy policy, and **three separate public store submissions**. *Limitation: filesystem mtimes and archive entry dates are weaker evidence than commits, and they bound the work window rather than measuring hours spent. The 13 Apr (Firefox) and 21 Apr (Chrome) endpoints are store-verified and authoritative.*

---

## 9. Business Impact

This is a free personal product, so impact is **qualitative and adoption-based, not commercial**. Stated honestly:

- **A real, publicly shipped product with real installed users, on every major browser.** 64 store-reported users across Chrome, Edge and Firefox, ~18 months after the last update, without the author doing anything. Modest in absolute terms; categorically different from a portfolio project that was never published.
- **Passed three independent public store reviews.** Chrome, Edge and Mozilla each gate on permissions, packaging and privacy claims separately. Three approvals is external third-party validation of the packaging work in §6.8 — and evidence of the ability to navigate three different submission processes, which are not interchangeable.
- **Positively received, at a sample size too small to generalise.** 3 ratings across all stores, mean 4.67 (Chrome 5.0×2, Firefox 4.0×1). Worth stating *with* the n.
- **Privacy-preserving by construction, not by promise.** A tool whose input is the user's own prompt text is a high-trust position. The code makes zero network requests and stores nothing but a button's x/y coordinates. For a product in this category that is a meaningful differentiator, and it is auditable by anyone who unzips the package.
- **Zero running cost, zero operational burden.** No server, no database, no on-call. The product has been live for 18 months at a marginal cost of £0 and zero maintenance hours — a direct consequence of the architecture.
- **User-facing time saved: NOT MEASURED.** Any claim like "saves users X minutes per session" would be fabricated. The product collects no usage data and there is no before/after study.

---

## 10. Honest Weaknesses Found in the Audit

Including these makes every other claim in this dossier more credible, and they are the material for the best interview answers.

1. **A dead code path.** `updateQuestionsList()` (line 778) looks up `document.getElementById("questions-list")` — an element that `createPanel()` never creates (it creates `#ai-prompts-navigator-questions`). The function therefore returns early every time. `extractQuestions()` and the `observeChat()` pipeline that call it are consequently **vestigial**: all live rendering happens through `findChatQuestions()`, which queries the correct ID. The feature works because a second, parallel implementation does the job — a classic artefact of iterative rewriting without deleting the predecessor.
2. **A duplicated function.** `observeUrlChanges()` is defined twice (lines 537 and 1598) with identical bodies; the second definition silently wins.
3. **State lost on background teardown in the Chromium build.** `panelVisible` lives in a module-level variable in an MV3 service worker. Diagnosed and fixed for Firefox (`chrome.storage.local`); never back-ported.
4. **57 `console.*` calls shipped to production.** Useful during development against five unreproducible third-party UIs; noise in users' consoles.
5. **No automated tests at all.** Defensible in part — the system under test is five proprietary SPAs that cannot be fixture-captured reliably — but the pure functions (`escapeHtml`, `extractChatId`'s URL parsing) are trivially testable and were not tested.
6. **No version control for the extension code.** History survives only as three dated `.zip` archives and filesystem timestamps. This audit was materially harder as a result, and it is the single highest-value process change for future projects.
7. **Unmaintained since April 2025.** Given §6.1, at least the DeepSeek (`.fbb737a4`) and Grok (`.whitespace-pre-wrap`) selectors are likely to have broken by now. **Current functional status on all five platforms is unverified.**
8. **A latent `event` reference.** The `mouseout` handler reads `event.clientX`/`event.clientY` from the implicit global rather than the handler's own parameter — works in practice, fragile by design.

---

## 11. Problems Solved (STAR-ready)

**P1 — One extension, five incompatible DOMs.**
*Situation:* the same product had to work on ChatGPT, Claude, Gemini, DeepSeek and Grok, none of which offers an API for "the user's messages" and none of which shares a markup convention.
*Problem:* Gemini renders prompts inside an Angular custom element; DeepSeek's only distinguishing hook is a build-hashed class (`.fbb737a4`); Grok's is a generic Tailwind utility class.
*Investigation:* manually inspected each platform's live DOM to find the most specific durable hook available on each, grading them by expected stability.
*Solution:* a prioritised selector cascade with per-selector `try/catch`, per-platform text-extraction branches, and explicit fail-soft UI (an informative empty state plus a Refresh button) instead of an exception.
*Result:* one byte-identical content script serves all five platforms and degrades visibly rather than silently when a vendor changes its markup.
*Evidence:* `content.js` lines 940–1090; identical cascades in `observeChat`, `extractQuestions` and `findChatQuestions`.

**P2 — Switching conversations left a stale prompt list.**
*Situation:* all five platforms are SPAs; a content script sees no route event.
*Problem:* after switching conversations, the panel continued to show the previous conversation's prompts, and reopening the panel manually was needed.
*Investigation:* established that each platform encodes conversation identity differently — and that Gemini sometimes changes only the URL *hash*, so path-only parsing missed the switch entirely.
*Solution:* `extractChatId()` with five URL patterns (including Gemini's hash) and named per-platform fallbacks; a `document.body` `MutationObserver` comparing `location.href`; capture of `wasPanelVisible` before state reset and restoration after; and a 500/1000/1500 ms retry ladder for the newly-loading conversation.
*Result:* conversation switches refresh the list and preserve the panel's open state.
*Evidence:* `content.js` lines 1503–1596 and 1634–1684.

**P3 — The Firefox port lost its panel state.**
*Situation:* porting the Chromium build to Firefox MV3.
*Problem:* `panelVisible` in a module-level variable does not survive background-context teardown; worse, `alert()` — used to warn users on unsupported sites — does not exist in a worker/event-page context at all, so that feedback path was simply dead.
*Investigation:* traced both symptoms to the MV3 background lifecycle rather than to Firefox-vs-Chrome API differences.
*Solution:* moved panel state into `chrome.storage.local` (read on startup, written on every toggle); replaced `alert()` with `chrome.notifications.create()`; converted the handler to `async`/`await` with `try/catch` around the icon and storage writes; declared `browser_specific_settings.gecko` with an explicit add-on ID and `strict_min_version: "109.0"`.
*Result:* a Firefox build that passed AMO review and has been live since 13 April 2025.
*Evidence:* diff of `ai-prompts-navigator-mozilla/background.js` (92 lines) against `ai-prompts-navigator-chromium/background.js` (75 lines).

**P4 — The panel would not scroll reliably inside host pages.**
*Situation:* long conversations produce long lists; the list must scroll inside the injected panel.
*Problem:* host-page CSS and nested flex layouts broke the scroll container intermittently and differently per platform.
*Investigation:* recorded in the code's own comments — successive simplifications of the layout.
*Solution:* abandoned flex entirely for the panel; absolute-positioned scroll container with explicit `top`/`bottom` plus matching `height: calc(100% - 60px)`, `overflowY` forced to `scroll` on open, and a post-open verification pass re-run at 0/1,000/2,000 ms that forcibly re-asserts visibility on the containers and items.
*Result:* reliable scrolling across all five platforms.
*Evidence:* `content.js` lines 276–447 and 448–536; comments "ULTRA SIMPLE STRUCTURE — no flex layouts" and "ABSOLUTELY SIMPLEST APPROACH POSSIBLE FOR MAXIMUM RELIABILITY".

**P5 — Packaging an over-privileged dev build for public review.**
*Situation:* the development build requested `scripting`, six host origins, and exposed ten JS files as web-accessible resources.
*Problem:* unnecessary privilege produces scarier install warnings, a larger attack surface, and avoidable store-review friction.
*Solution:* consolidated to one content script, dropped `scripting` from the Chromium build, removed the legacy `chat.openai.com` origin, and reduced `web_accessible_resources` to `icons/*`.
*Result:* a minimal-privilege Chromium package and a Firefox package that passed AMO review.
*Evidence:* manifest comparison, March archive vs shipped builds (§6.8).

---

## 12. Lessons Learned

- **Use version control even for a weekend project.** The absence of git for the extension code is the project's most consequential process failure. Three `.zip` snapshots and mtimes are a poor substitute, and reconstructing the decision history for this dossier required archaeology that a `git log` would have answered in a second.
- **Delete the old implementation when you write the new one.** The `updateQuestionsList` / `findChatQuestions` duplication (§10.1) is the direct cost of iterating by addition. The product works, but half the message-extraction pipeline is dead weight that a future maintainer would have to read before realising it never runs.
- **Platform lifecycle beats platform API differences.** The hardest part of the Firefox port was not `chrome.*` vs `browser.*` — it was understanding that an MV3 background context is ephemeral. The same lesson applies back to the Chromium build, which still has the bug.
- **Modularisation that the platform fights is not worth it.** Splitting into ten non-module scripts added a manifest-ordering contract and ten web-accessible files, and solved nothing MV3 didn't re-break. Reverting was the right call — but Shadow DOM plus a bundler would have given both isolation and structure, and is what today's version would use.
- **Privacy by construction has a cost, and it should be a conscious one.** Collecting nothing is the right default for a tool that reads the user's prompts — but it means the author has *no* visibility when a vendor DOM change silently breaks the product. A privacy-preserving middle ground (a local-only, explicitly opt-in "selector failed" counter, or simply a visible in-panel error state the user can report) would preserve the trust position while making breakage discoverable.
- **Shipping is a skill of its own.** Icons, a privacy policy, a landing page, store copy, permission justification and a review cycle were roughly as much elapsed work as the panel itself, and none of it is visible in the code.

---

## 13. Verified Career Claim Bank

| # | Claim | Level | Evidence | Period | Qualification |
|---|---|---|---|---|---|
| 1 | Designed, built and published a cross-browser Manifest V3 extension to **all three major extension stores** — Chrome Web Store, Microsoft Edge Add-ons and Firefox Add-ons — as sole author | **VERIFIED** | Three live listings (IDs in §8.1); `manifest.json` author field; no other contributor in any artefact | Mar–Apr 2025; live since Apr 2025 | Publisher shows as *Clevali* on Edge/Firefox and *abramseyakid* on Chrome; manifest author field names Abraham Akintola |
| 2 | Extension has been continuously publicly available for ~18 months with a combined store-reported user count of 64 (Edge 24, Chrome 21, Firefox 19) | **VERIFIED** (counts) / **DERIVED** (sum, duration) | All three listings read 17 Oct 2026; AMO `created: 2025-04-13` | Apr 2025 – 17 Oct 2026 | **Three different metric definitions summed** — active installs vs users vs daily actives. Point-in-time, not cumulative installs. Never state as "64 daily users" |
| 3 | Rated 5.0 on Chrome (2 ratings) and 4.0 on Firefox (1 review); 3 ratings across all stores | **VERIFIED** | Chrome listing page; AMO `ratings.average: 4.0, count: 1`; Edge has 0 ratings | At audit date | **n = 3 in total.** Always state the sample size |
| 4 | Built a DOM-integration layer supporting five independent AI chat platforms (ChatGPT, Claude, Gemini, DeepSeek, Grok) from a single codebase | **VERIFIED** | `manifest.json` host permissions; five-tier selector cascade and five URL-parsing branches in `content.js` | Mar–Apr 2025 | Current functional status on each platform is unverified (unmaintained since Apr 2025) |
| 5 | Ported the extension across two divergent MV3 implementations, diagnosing and fixing a background-lifecycle state-persistence bug and an unavailable-API path | **VERIFIED** | `background.js` diff: `chrome.storage.local` persistence; `alert()` → `chrome.notifications` | Apr 2025 | Fix applied to the Firefox build only |
| 6 | Reduced the extension's requested privilege surface before public release — removed the `scripting` permission, a legacy host origin, and ten web-accessible JavaScript files | **VERIFIED** (in the final packaged Chromium build) | March archive manifest vs final Chromium manifest | Mar–Apr 2025 | **The Edge listing reports the earlier set (`activeTab`, `scripting`, `tabs`)**, so this did not reach all three stores. Firefox retains `scripting`/`storage`/`notifications` for platform reasons. Claim the narrowing, not its universal rollout (§6.8) |
| 7 | Built the product with zero backend, zero network requests and zero third-party dependencies; all state is browser-local | **VERIFIED** | `grep` for `fetch`/`XHR`/analytics returns zero matches; only `localStorage` + `chrome.storage.local` | Mar–Apr 2025 | — |
| 8 | Wrote a hand-rolled resilience layer — prioritised selector cascade, per-platform text extraction, layered retry ladders and fail-soft UI — for five third-party SPAs with no stable integration contract | **VERIFIED** | `content.js` lines 829–1503; three independent retry mechanisms | Mar–Apr 2025 | Some selectors are knowingly fragile; this is a stated design trade-off |
| 9 | Built and deployed a React/TypeScript/Tailwind marketing site with an automated GitHub Actions → GitHub Pages pipeline | **VERIFIED** | `.github/workflows/deploy.yml`; 8 personal commits; `gh-pages` branch on origin | 4 Apr 2025 | **Initial scaffold was AI-generated (Lovable, 11 bot commits).** Branding, restructuring, content and CI are personal work — say so |
| 10 | Delivered the complete product lifecycle solo — idea, UX, implementation, icons, privacy policy, landing page, packaging, and three separate store submissions and reviews — in roughly five weeks of elapsed part-time work | **STRONGLY SUPPORTED** | Artefact timeline 15 Mar – 21 Apr 2025 (§8.3); endpoints store-verified | Mar–Apr 2025 | Elapsed window, not hours worked; interior dates derived from mtimes and archive dates |
| 11 | Passed three independent third-party add-on review processes (Google, Microsoft, Mozilla) with the same product | **VERIFIED** | Three live listings at audit date | Apr 2025 | Review *outcomes* are evidenced by the listings being live; the review correspondence itself was not inspected |

---

## 14. CV Achievement Candidates

### High-confidence, quantified

- Designed, built and published **AI Prompts Navigator**, a zero-dependency Manifest V3 browser extension that adds a clickable prompt outline to five AI chat platforms (ChatGPT, Claude, Gemini, DeepSeek, Grok); **live on all three major extension stores** — Chrome, Edge and Firefox — with **64 store-reported users 18 months after release**, rated 5.0 on Chrome, and running at **zero infrastructure cost with no backend and no telemetry**.
- Took a solo product from idea to **three approved public store listings in ~5 weeks**, passing **three independent third-party add-on reviews** (Google, Microsoft, Mozilla) — including packaging, icon assets, privacy policy and permission justification for each.
- Shipped a **cross-browser extension from a single 1,684-line content script**, byte-identical across Chromium and Gecko builds, with all platform divergence isolated to the manifest and background layer — including a diagnosed fix for **MV3 background-context state loss** using `chrome.storage.local`.
- Cut the extension's requested privilege surface before release — **removed one permission, one host origin and ten web-accessible script files** — producing a minimum-privilege Chromium package.

### High-confidence, non-quantified

- Engineered a **fail-soft DOM integration layer** against five proprietary single-page applications with no public API and no stable markup contract, combining a prioritised five-tier selector cascade, per-platform text-extraction fallbacks, three independent retry ladders and an explicit recoverable empty state.
- Solved SPA conversation-change detection without router access by **parsing five distinct URL schemes** (including Google Gemini's hash-only transitions) and using a `MutationObserver` on the document body as a route-change proxy, preserving UI state across navigation.
- Built the product **privacy-first by construction**: zero network requests, zero analytics, zero third-party code, with all state confined to browser-local storage — a verifiable property of the shipped bundle, not a policy promise.
- Delivered an entire consumer product solo — UX, implementation, icon set, privacy policy, React/TypeScript marketing site with automated GitHub Pages deployment, store packaging and review response across three stores.

### Phrasing warnings (not blockers)

These bullets are usable, but only in the forms given — the shortcuts are not defensible:

- Write **"64 store-reported users across three stores"**, never "64 daily users" or "64 active users": the three stores count differently (§8.1), and no cumulative install total is available from any of them.
- Write **"rated 5.0 on Chrome (2 ratings)"** or "4.67 average across 3 ratings", never a bare "5-star rated" — the sample is three.
- Claim the **permission narrowing**, not that it shipped everywhere: the Edge listing carries the earlier permission set (§6.8).

---

## 15. Portfolio Case Study Material

**One-line pitch.** A browser extension that gives long AI conversations the table of contents they should have shipped with — across five platforms, with no server and no data collection.

**Challenge.** AI chat products render conversations as an infinite scroll with no index. Building an outline for them means reading the DOM of five separate proprietary SPAs that expose no API, change their markup without notice, and in two cases offer no better hook than a build-hashed class or a Tailwind utility.

**Role.** Sole designer, engineer and publisher. (Landing-page scaffold was AI-generated; branding, restructuring, content and the deployment pipeline were not.)

**Solution.** A single zero-dependency content script injected into five origins. Platform detection by hostname, conversation identity by per-platform URL parsing, prompt discovery by a prioritised selector cascade with per-platform text extraction, live updates via `MutationObserver`, and a self-injected fixed-position panel styled inline to survive hostile host CSS. State lives in the browser; nothing leaves it.

**Interesting problems to feature:** the five-selector cascade and what each selector choice reveals about the vendor behind it; Gemini's hash-only conversation switches; the MV3 ephemeral-background bug found during the Firefox port; the modularise-then-re-consolidate decision and why the platform made modularity expensive; the permission-narrowing pass before store review.

**Results.** Live on Chrome, Edge and Firefox since April 2025. 64 store-reported users at 18 months (Edge 24, Chrome 21, Firefox 19 — three different metric definitions). Rated 5.0 on Chrome (2 ratings) and 4.0 on Firefox (1). Three independent store reviews passed. Zero maintenance, zero infrastructure, zero cost since release.

**What I'd do differently.** Shadow DOM + a bundler for isolation *and* structure; git from commit one; delete superseded code paths; back-port the storage fix to Chromium; a privacy-preserving breakage signal so vendor DOM changes surface without telemetry; strip production logging.

---

## 16. Interview Story Bank

| Theme | Story | Where |
|---|---|---|
| **Integrating with systems you don't control** | The five-selector cascade — and specifically why `.fbb737a4` (DeepSeek) and `.whitespace-pre-wrap` (Grok) were knowingly fragile choices, and how the mitigation was architectural (fail soft + retry + visible Refresh) rather than trying to find a better selector that didn't exist | §6.1, P1 |
| **Debugging a platform, not your code** | MV3 ephemeral background contexts: the panel-state bug and the dead `alert()` path, both traced to lifecycle rather than to Chrome-vs-Firefox API differences | §6.7, P3 |
| **Architecture decision + reversal** | Splitting into ten modules, then consolidating back to one file — and being able to explain *why* the platform made modularity expensive (no ES modules in content scripts, manifest load-order contract, ten web-accessible files) rather than "it felt simpler" | §7 D1 |
| **A trade-off with no right answer** | No telemetry means no breakage detection. Defend the privacy decision, then describe the middle ground you'd build today | §7 D3, §12 |
| **Security awareness** | Finding your own dev build over-privileged and cutting it back before review: the `scripting` permission, the legacy origin, ten web-accessible scripts | §6.8, P5 |
| **Honest self-critique** | The dead `updateQuestionsList` path: a feature that works because a second implementation quietly took over, and the duplicated `observeUrlChanges`. Strong answer to "tell me about technical debt you created" | §10.1–10.2 |
| **Ownership and shipping** | Taking a personal idea through icons, privacy policy, landing page, CI/CD, packaging and third-party store review in ~33 elapsed days | §8.3 |
| **Working with AI tooling honestly** | The landing page was scaffolded by Lovable and then substantially reworked by hand — being precise about what the tool produced and what you did is itself a credibility signal | §3 |

---

## 17. Technical Article and Blog Opportunities

1. **"Five AI chat apps, five different ways to find a user's message."** Walk through `[data-message-author-role]`, `[data-testid]`, a custom element, a build-hashed class and a Tailwind utility — what each reveals about the app behind it, and how to design a cascade that degrades instead of throwing. *Nothing to anonymise; all selectors are publicly observable.*
2. **"Your MV3 background script is not a server."** Ephemeral background contexts, state that silently vanishes, and APIs (`alert`) that don't exist there — using the Firefox port as the worked example, with the before/after `background.js`.
3. **"Detecting navigation in an SPA you don't own."** Why you can't hook the router from a content script, why a body `MutationObserver` + `location.href` comparison works, what it costs on a token-streaming page, and the Gemini hash-only case.
4. **"I shipped an extension with no analytics — here's what that cost me."** The privacy-by-construction trade-off, honestly: what you gain in trust, what you lose in breakage detection, and the opt-in middle ground. *The most distinctive piece available from this project.*
5. **"Trimming an extension's permissions before store review."** A concrete before/after of the manifest, why `scripting` and web-accessible scripts matter, and how minimum privilege reads to a reviewer.
6. **"I modularised, then un-modularised."** A short honest piece on why MV3 punishes multi-file content scripts, and what Shadow DOM + a bundler would do instead.

*Do not publish:* nothing here is commercially sensitive — this is a personal project with no proprietary code or customer data. The only care needed is to avoid presenting selectors as currently working (they may have broken) and to be precise about the landing page's AI-generated scaffold.

---

## 18. Professional Content Opportunities (LinkedIn / X)

Each of these is grounded in something that actually happened; none is a motivational post.

- **"A browser extension I shipped 18 months ago still runs in 64 people's browsers, and I haven't touched it since."** Zero servers, zero cost, zero maintenance — a short post on what architecture buys you. *Evidence: three live store listings, Apr 2025 → today.*
- **"I shipped the same extension to three stores. They disagree about what a 'user' is."** Edge counts active installs, Chrome counts "users", Firefox counts average daily users — a genuinely useful short post for anyone about to publish an extension and wonder why their numbers don't reconcile. *Evidence: §8.1.*
- **"Five AI chat apps. One of them identifies your message with a CSS class called `.fbb737a4`."** A post about integrating with interfaces that were never meant to be integrated with. *Evidence: the selector cascade.*
- **"I shipped a tool that reads your AI prompts and sends them nowhere."** Privacy by construction and what it costs — the most interesting trade-off in the project. *Evidence: zero network calls in the bundle, verifiable by anyone.*
- **"Auditing my own two-year-old code, I found a function that has never once done anything."** `updateQuestionsList` and the dead pipeline — a credible, specific technical-debt post. *Evidence: §10.1.*
- **"Porting to Firefox taught me more about Chrome than Chrome did."** MV3 background lifecycle. *Evidence: the `background.js` diff.*
- **"The part of shipping nobody counts."** Icons, privacy policy, landing page, store copy, review — roughly half the elapsed time on a one-month project. *Evidence: §8.3 timeline.*
- **Retrospective: "What I'd change about a project I shipped in 2025."** Shadow DOM, git from day one, deleting superseded code, back-porting the storage fix. *Evidence: §12.*

---

## 19. Private vs Public-Safe Information

**Public-safe (portfolio, LinkedIn, articles, CV):** everything about the architecture, the selector strategy, the MV3 porting work, the permission narrowing, the privacy posture, all three stores' public metrics (they are on public listing pages anyone can read), the three listing URLs, the development timeline, the technology stack, and all the self-critical findings in §10. There is no customer data, no credential, no proprietary code and no commercial confidence anywhere in this project. Three caveats when publishing: always state the **sample size** alongside any rating, always state the **metric-definition caveat** alongside the combined user count, and always state that the **landing-page scaffold was AI-generated**.

**Private / for your own records only:** the possibility that selectors are currently broken on one or more platforms (do not demo without checking first — §20.6); the Edge-vs-Chromium permission discrepancy (§6.8) if you would rather not volunteer it unprompted, though it is a *good* answer if asked; personal email and GitHub handle appear in listing metadata and are already public but need not be amplified; any future revenue or ad plans referenced in the privacy policy.

---

## 20. Evidence Gaps and Further Metrics Worth Collecting

| # | Missing evidence | Why it matters | Where it lives | Action |
|---|---|---|---|---|
| 1 | **Historical user curves and peak/cumulative installs for all three stores** | Turns three point-in-time numbers into a trend, and a cumulative install total is a legitimately bigger (and still honest) figure than a point-in-time count | Chrome **Developer Dashboard**, Microsoft **Partner Center**, AMO **Developer Hub** → Statistics (all owner-only) | Sign in to each, export the install/user series since Apr 2025. **Highest-value remaining item** |
| 2 | **Exact Edge publication date** | The Edge API's 6 Apr 2025 date predates the Firefox listing and is probably the package build date, not the listing date (§8.2) | Microsoft **Partner Center** → submission history | Record the actual "published" date to close the one dated anomaly in this dossier |
| 3 | **Whether the Chrome listing carries the narrowed permission set** | Would settle §6.8 — the Edge listing shipped `scripting`, the final local manifest does not | Chrome **Developer Dashboard** → the published package, or `chrome://extensions` with the store build installed | Install the Chrome Web Store build and read its permissions |
| 4 | **Any review feedback or rejection rounds from the three stores** | "Passed three reviews" is already verified; *how many rounds it took* is an interview story that cannot be reconstructed from the listings | Review-decision emails in the submission account's inbox | Search mail for "Chrome Web Store", "Partner Center" and "addons.mozilla" around 6–25 Apr 2025 |
| 5 | **The text of the three ratings** (Chrome ×2, Firefox ×1) | A real user sentence is worth more than a star rating in a portfolio | Public listing reviews tabs on both stores | Read and quote them |
| 6 | **Current functional status on each of the five platforms** | Everything in §6.1 implies some selectors have decayed; you should know before anyone demos it | Your own browser, 15 minutes | Install the build, open one conversation on each platform, record pass/fail per platform and date it |
| 7 | **Any ad/monetisation outcome** | The privacy policy anticipates ads and the panel reserves an ad-banner container (commented out) | Personal records | Confirm none was ever enabled — currently the code says it was prepared but never activated |
| 8 | **Landing-page traffic (GitHub Pages / any analytics)** | Would evidence reach beyond installs | GitHub Pages has no built-in analytics; check if any was added | Likely unavailable — record as permanently unknown if so |
| 9 | **Whether the gh-pages site is still live** | A working demo link materially strengthens a portfolio entry | `origin/gh-pages` exists; the obvious Pages URL returned 404 at audit date | Check repository Settings → Pages for the configured URL, or redeploy |

---

## Appendix — Evidence Notes and Methodology

**What was inspected.** Both packaged extension builds in full (`manifest.json`, `background.js`, `content.js`, `styles.css`, icon sets); three dated development archives extracted and compared against the shipped builds; the landing-page repository including full `git log`, `shortlog -sne`, per-commit file statistics, branch list and the GitHub Actions workflow; filesystem modification times across every artefact; and the public Firefox Add-ons listing via both the AMO v5 API and the rendered page.

**Production-data protocol.** Not applicable. There is no `.env`, no credential, no connection string, no database and no backend in this project. No query of any kind was executed against any system. No credential, token or secret appears in any output file, because none exists in the project.

**Store-metric methodology.** All three stores were read on **17 October 2026**, the same day, so the figures are directly contemporaneous even though their definitions differ.

- *Firefox:* `https://addons.mozilla.org/api/v5/addons/addon/ai-prompts-navigator/`, cross-checked against the rendered listing page; both agreed on 19 users, 4.0/1 review, v1.0.0, created 13 Apr 2025. `average_daily_users` is a rolling point-in-time measure, **not** a cumulative total.
- *Edge:* `https://microsoftedge.microsoft.com/addons/getproductdetailsbycrxid/abcopdbmckchogodlkknclocabccidie`, returning `activeInstallCount: 24`, `ratingCount: 0`, v1.0.0, and the permission list quoted in §6.8.
- *Chrome:* the public listing page for `ajodgicjemafjmbgkjbhfogdpfejihjg`, reporting 21 users, 5.0 from 2 ratings, v1.0.0, 86.73 KiB, last updated 21 Apr 2025. The Chrome listing page does not expose a permission list.

The combined figure of 64 is an **arithmetic sum of three differently-defined store metrics**, flagged as such everywhere it appears, and is never presented as a count of distinct individuals. No cumulative install total is claimed, because none of the three stores publishes one to a non-owner.

**Audit history.** The Chrome and Edge listings could not be located by public search during the original audit (searched by exact product name, by publisher name *Clevali*, and by domain-restricted search against `chromewebstore.google.com`, `chrome.google.com` and `microsoftedge.microsoft.com`) and were recorded as UNKNOWN rather than assumed in either direction. The project owner supplied both URLs the same day and both were then measured directly. The original negative result is retained here because it explains the revision note at the top of this document, and because "the listing exists but is not discoverable by name search" is itself worth knowing about the product's distribution.

**Date methodology.** The extension has no version-control history, so its timeline is reconstructed from filesystem mtimes and zip-archive entry dates. These bound the work window and are reliable for ordering, but they are weaker than commits and cannot measure effort. Landing-page dates come from git commit timestamps and are reliable. AMO dates come from the API and are authoritative.

**Attribution methodology.** Extension authorship is attributed wholly to the project owner on the basis of the manifest author field, matching contact details across the AMO listing and the git commits, the absence of any other contributor or vendored dependency, and the owner's own account. Landing-page authorship is split explicitly by commit author, with the AI-generated portion named rather than absorbed.

**Line references** are to `ai-prompts-navigator-chromium/content.js` (1,684 lines), which is byte-identical to the Mozilla build's copy.

**Confidence statement.** Every quantified figure in this dossier traces to either a public store API response read on the audit date or a direct file/commit inspection. No percentage, time saving, revenue figure, user-count estimate or performance improvement has been invented. The one place where a larger number is available but unproven — **cumulative lifetime installs across the three stores**, which would almost certainly exceed the 64 current users — is recorded as a gap (§20.1) rather than estimated, and the one place where a cleaner story was available but contradicted by evidence — the permission narrowing reaching all three stores — is recorded as a qualification (§6.8) rather than smoothed over.
