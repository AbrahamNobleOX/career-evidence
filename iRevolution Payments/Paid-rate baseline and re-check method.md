# Paid-rate baseline and re-check method
### Private working note — not part of the Career Evidence Dossier

**Purpose.** Hold the prepared baseline and a repeatable method so the effect of the July 2026 release on payment conversion can be assessed cleanly **once enough post-release data exists**. Prepared 10 October 2026.

**Suggested re-check date: March 2027** (≈8 months of post-release data), or any time after ~6 months.

---

## 1. Why this is not in the dossier yet

At the time of preparation the release had only ~3 months of post-deployment data. At those sample sizes the smallest difference distinguishable from normal month-to-month variation was about **±2 percentage points** — so anything smaller than that, in either direction, simply could not be seen. Measuring again later, with a larger post-release window, gives a genuine answer rather than a premature one.

## 2. The comparison that must NOT be used

Post-release 78.9% against the **lifetime** average of 76.3% looks like a ~2.6 pp improvement. **It is an artefact.** The lifetime figure is dragged down by a dip during 2025 that had already recovered *before* the release — May and June 2026 both recorded 80.1%, higher than any month since. Any claim built on the lifetime average will not survive scrutiny. Use the windowed method below instead.

## 3. Key dates

| Event | Date |
|---|---|
| Worldpay MD5 signing fix (commit `af467f9`) | 2026-07-05 |
| Full modernisation release to production | 2026-07-17 |
| Baseline prepared | 2026-10-10 |

Payment links are valid for 7 days, so conversion matures quickly. **Always exclude payments created in the last 14 days** of any analysis window, or recent immature requests will depress the "after" figure.

## 4. Baseline as measured on 2026-10-10

### Adjacent equal-length windows (84 days either side of the fix)

| Window | Requests | Paid | Paid rate | Mean amount | Users |
|---|---|---|---|---|---|
| Before: 2026-04-12 → 2026-07-04 | 3,146 | 2,483 | 78.93% | £784.90 | 61 |
| After: 2026-07-05 → 2026-09-26 | 3,229 | 2,534 | 78.48% | £722.41 | 67 |

Difference −0.45 pp; 95% CI [−2.46, +1.56]; two-proportion z = −0.438; p = 0.661. Minimum detectable difference ±2.01 pp. **Inconclusive at this duration — not evidence of absence of an effect.**

### Year-over-year, same calendar window (5 Jul – 26 Sep)

| Year | Requests | Paid rate |
|---|---|---|
| 2023 | 5,366 | 79.76% |
| 2024 | 4,491 | 77.00% |
| 2025 | 4,430 | 69.35% |
| 2026 | 3,229 | 78.48% |

### Stratified by payment size (adjacent windows)

| Band | Before | After |
|---|---|---|
| < £100 | 86.14% | 85.73% |
| £100–250 | 79.28% | 78.31% |
| £250–500 | 78.03% | 77.80% |
| £500–1,000 | 75.05% | 76.37% |
| £1,000–2,000 | 77.91% | 74.69% |
| £2,000+ | 72.50% | 72.28% |

### Monthly reference series (paid rate)

2025: Jul 63.4 · Aug 70.9 · Sep 72.3 · Oct 73.7 · Nov 72.0 · Dec 72.2
2026: Jan 74.9 · Feb 72.6 · Mar 74.5 · Apr 77.3 · **May 80.1 · Jun 80.1** (pre-fix) · Jul 76.2 · Aug 79.8 · Sep 79.2 · Oct 78.8

## 5. How to re-run it

Read-only against the production database. Three controls, all required:

**(a) Adjacent equal-length windows.** Keep the "before" window fixed at 2026-04-12 → 2026-07-04 (3,146 requests, 78.93%). Set the "after" window to run from 2026-07-05 to 14 days before the re-check date. Compare paid rates; the longer the after-window, the smaller the effect that becomes detectable.

**(b) Year-over-year seasonal control.** Compare the same calendar window across 2023, 2024, 2025 and the current year, to separate a release effect from seasonality.

**(c) Size stratification.** Break both windows into the amount bands above, since a shift in mean payment size can otherwise mask or manufacture an apparent effect.

**Statistics.** Two-proportion z-test plus a 95% confidence interval on the difference. Also report the minimum detectable difference, so an inconclusive result is not mistaken for a null result.

**Maturity rule.** Exclude the final 14 days before the re-check date.

## 6. Interpreting whatever comes back

- **A rise beyond the detection floor** — report it as an association, not proof of causation. Several things changed at once in July 2026 (signing fix, branded emails, UI and KPI work, access control), so no single cause can be isolated from this data alone.
- **Still inconclusive** — say so plainly; make no conversion claim in either direction.
- **A fall** — check mix and seasonality first (controls b and c) before drawing any conclusion.

**Regardless of outcome:** the MD5 signing fix should be claimed as closing a security and accounting-integrity defect. Its justification does not depend on conversion — a payment system in which the customer can choose what they pay is unacceptable irrespective of what the conversion metric does.

## 7. Related open question

The 2025 movement in paid rate (dip to 63.4% in July 2025, recovery to ~80% by May 2026) predates this work and has no established cause. Worth asking internally, since it is the reason the lifetime average is an invalid baseline.
