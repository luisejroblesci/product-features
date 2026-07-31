# Spec: Surface Smarter Testing on the Pipelines Page

**Experiment:** `smarter-testing/surface-features-pipeline-page`  
**Status:** Draft — for eng/PM discussion  
**Prototype:** https://claude.ai/code/artifact/198f6dab-0804-4c39-ae98-775ca200b774

---

## Problem Statement

CircleCI's three Smarter Testing features — Test Impact Analysis (TIA), Dynamic Test Splitting (DTS), and Auto Rerun Failed Tests (ARFT) — are in Beta. Adoption is low because users must independently discover and follow setup docs. There is no in-product signal at the moment users are reviewing their CI results.

The Pipelines page is CircleCI's highest-traffic page ([Amplitude dashboard](https://app.amplitude.com/analytics/circleci/dashboard/fz92hq7s)). Every row represents a pipeline run the user is already evaluating. Surfacing eligibility indicators and savings estimates directly in those rows creates a discovery path with zero additional navigation.

---

## Use Cases

### UC-1: Eligible project, no features enabled
A user opens the Pipelines page. Their project uses Jest and runs with `parallelism: 4`, making it eligible for TIA and DTS. They have never heard of Smarter Testing.

**Expected experience:** An amber badge `⚡ Pipeline optimizations · 2 available` appears on the row. Hovering shows a tooltip with feature names and estimated monthly savings. Clicking opens a slide-out panel with one card per feature, each showing a monthly savings estimate and a CTA to set up.

---

### UC-2: One feature active, more available
A user enabled TIA last week. DTS and ARFT are still available for their project.

**Expected experience:** Two badges appear: a green `✓ TIA active` badge and an amber `⚡ +2 available` badge. The SAVINGS column shows per-run actuals for TIA (`83% fewer tests`) alongside monthly estimates for the remaining features. The panel shows a mixed view: an active card with this-run metrics for TIA, and eligible cards for DTS and ARFT.

---

### UC-3: All eligible features active
All features the project qualifies for are enabled.

**Expected experience:** A single green `✓ Smarter Testing active` badge replaces the amber badge. SAVINGS column shows combined per-run savings. The panel shows all-active cards with per-run actuals. A footer CTA links to Smarter Testing Insights for deeper analysis.

---

### UC-4: Project not eligible
The project does not meet eligibility signals for any feature (e.g. uses an unsupported framework, `parallelism: 1`, no JUnit output).

**Expected experience:** No badge appears. Row is unchanged.

---

## Eligibility Signals (project-level, not per-run)

| Feature | Proxy signal |
|---|---|
| Test Impact Analysis | Supported framework detected (Jest, pytest, Go, Vitest, RSpec); single-process test execution; **not** E2E-heavy (see TIA Disqualifiers below) |
| Dynamic Test Splitting | `parallelism > 1` in `.circleci/config.yml` for any test job |
| Auto Rerun Failed Tests | `store_test_results` configured (JUnit XML); flaky test history present in Test Insights |

> **Open with eng:** Confirm which signals are queryable at pipeline-run data layer. These are proxy signals.

### TIA Disqualifiers

Projects that are E2E-heavy should be excluded from TIA recommendations. Specifically, if Cypress or Playwright is detected and tests appear to call a running application (separate service), TIA will not work — coverage-based test selection requires single-process execution.

### Test Runner Detection Caveat

Test runners may be hidden behind task commands (e.g. a Makefile target or a custom wrapper script that calls `jest` or `pytest` internally). Parsing `.circleci/config.yml` alone won't always surface the runner directly. The detection pipeline needs to account for this indirection.

### Feature Adoption Logic

If **any single job** in a project is already using one of the three ST features, that feature is considered adopted for the project. Full coverage across all jobs is not required to count the feature as active.

---

## Savings Display Logic

**Inactive features → monthly aggregate estimates**  
Show estimates based on project activity over the past 30 days (test count, run frequency, flaky failure rate). Do not show per-run estimates for features the user has not enabled — the per-run framing implies something happened this run, which is false.

**Active features → per-run actuals**  
Compare current run against the rolling 30-day average prior to feature activation. Surface as: `83% fewer tests` or `4m 54s saved this run`.

---

## Decisions Made

| Decision | Rationale |
|---|---|
| Monthly estimates for inactive features | Surfacing per-run estimates for inactive features implies something occurred in the run; monthly framing sets correct expectations and makes the value proposition more concrete. Raised by Luis Jiménez (2026-07-30). |
| No inline sub-row inside the pipeline run row | The sub-row pattern (left-bordered narrative below the row, matching `rerunFailedTestsJobMetrics.tsx`) breaks the design system when applied to project-level signals that appear on every run. Savings info lives in the SAVINGS column and the slide-out panel instead. Raised by Liam Clarke (2026-07-30); decision by Luis Jiménez (2026-07-30). |
| SAVINGS column as the primary data surface | Added a dedicated `SAVINGS (Beta)` column to the Pipelines table, visible alongside `PIPELINE RUNS`, `TRIGGER`, `START/FINISH`, and `ACTIONS`. Keeps the savings signal always visible without requiring a click. |
| Slide-out panel (not inline expansion) | Avoids layout disruption on the table; consistent with other CCI side panels; allows richer card content per feature. |
| Single "View docs" CTA → Getting Started guide | All "View docs" links go to `https://circleci.com/docs/guides/test/getting-started-with-smarter-testing/` rather than individual feature pages. Reduces friction for new users who need orientation before jumping into per-feature setup. |
| "Set up with Claude Code" CTA → `claude://` deeplink | Hypothesis: a pre-populated Claude Code prompt lowers setup friction for users who already have the CLI. Open question: does the URI scheme exist? What is the fallback if not installed? |
| Eligibility is project-level, not per-run | Badges appear on every run for the same project. No per-run eligibility recalculation. |
| Dismissal UX required | Users who open the slide-out panel and decide they're not interested need a way to dismiss a feature recommendation. Once dismissed, that feature should not reappear as a suggestion for that project. (2026-07-31 planning session) |
| E2E projects disqualified from TIA | Cypress/Playwright projects that test against a running app rely on separate services and are incompatible with single-process coverage — TIA should not be recommended for them. (2026-07-31 planning session) |

---

## Out of Scope (V1)

- Old pipelines UI (legacy view, "Updated view" toggle OFF)
- Eligibility for projects without JUnit XML (non-standard test runners)
- Org-level or team-level rollout controls
- In-depth savings analytics page (link to existing Test Insights instead)
- Mobile/responsive layout
- Projects with `test-suites.yml` configured but feature flag off (partial config state)

> **Note:** Dismiss / snooze was previously listed as out of scope but has been confirmed as needed — see Decisions Made above.

---

## Next Steps (as of 2026-07-31)

1. **Data reach-out** — Connect with the data team to identify eligible projects per feature using `config.yml` signals, Insights API data, and test results.
2. **Manual outreach** — Leverage TSMs and Field Engineers to proactively inform customers about features they're eligible for, ahead of the in-product experience.
3. **Phase the build** — Break the artifact into phases to ship faster. Ship suggestions per feature rather than waiting to surface all three at once.
4. **Feedback loop** — Luis to explore an inline feedback mechanism on recommendations so customers can give signal faster through the UI.
5. **Money team sync** — Luis to sync with the money team on new recommendations insights to inform surfacing and prioritization logic.

---

## Open Questions

1. **`claude://` URI scheme** — Does it exist? Needs to be registered by the Claude Code CLI installer. Alternative: web-based fallback (`claude.ai/code`). Confirm with `#project-code-factory`.
2. **Eligibility data availability** — Which project-level signals (framework, parallelism, JUnit, flaky history) are queryable from the Pipelines page backend?
3. **Baseline for savings actuals** — Is the pre-activation 30-day average run duration accessible in the product backend?
4. **Badge deduplication** — If the same project has 10 runs in view, the badge appears 10 times. Discuss: show only on the most recent run for each project?
5. **Analytics** — Should clicking the CTA fire an Amplitude event? Proposed: `smarter_testing_cta_clicked` with properties `feature`, `source: pipelines_page`.

---

## Changelog

| Date | Who | Change |
|---|---|---|
| 2026-07-30 | Luis Jiménez | Created prototype and this spec. Shared in `#smarter-testing-private` for review. |
| 2026-07-30 | Liam Clarke | Feedback: align savings display with existing `rerunFailedTestsJobMetrics` time-saved pattern. Shared component reference. |
| 2026-07-30 | Luis Jiménez | Decided against sub-row (breaks design system); savings surface moved to SAVINGS column and panel only. Switched inactive feature metrics from per-run to monthly aggregates. |
| 2026-07-31 | Luis Jiménez | Planning session: added TIA disqualifiers (E2E/Cypress/Playwright), test runner detection caveat, feature adoption logic (single job = adopted), dismissal UX confirmed as required (moved from out of scope), next steps added. |

---

## References

- [Prototype (interactive mockup)](https://claude.ai/code/artifact/198f6dab-0804-4c39-ae98-775ca200b774)
- [User flows](./user-flows.md)
- [Getting started with Smarter Testing](https://circleci.com/docs/guides/test/getting-started-with-smarter-testing/)
- [Amplitude dashboard](https://app.amplitude.com/analytics/circleci/dashboard/fz92hq7s)
- [Ideas/notes doc](https://docs.google.com/document/d/1ozecsP6-9RcR6cIoCK0D1R2CKnKCUt8cwGzr2GH9BI4/edit?tab=t.m2hbu0vq0o3j)
- Slack: `#smarter-testing-private` (C0ACNPLRZ7A) · `#smarter-testing`
- `rerunFailedTestsJobMetrics.tsx` — [GitHub](https://github.com/circleci/web-ui-consolidated/blob/4905c0a813fb57f21c9a921d09f00a66aa5f7dc0/web-ui/src/components/rerunFailedTestsJobMetrics.tsx)
