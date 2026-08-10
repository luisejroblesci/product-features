# Spec: Surface Smarter Testing on the Org Home Page

**Experiment:** `smarter-testing/surface-features-org-home`  
**Status:** Draft — for eng/PM discussion  
**Prototype:** _(TBD — see mockup.html)_

---

## Problem Statement

CircleCI's three Smarter Testing features — Test Impact Analysis (TIA), Dynamic Test Splitting (DTS), and Auto Rerun Failed Tests (ARFT) — are in Beta with low adoption. Users must independently discover and follow setup docs; there is no in-product signal.

Amplitude data shows the **Org Home page is the highest-traffic page** among the candidate surfaces (Org Home, Project Overview, Pipelines page). The Org Home project card grid is where users already check their project health daily. Adding a per-project eligibility signal here creates the earliest and most broadly visible discovery path.

> **See also:** [`surface-features-pipeline-page`](../surface-features-pipeline-page/readme.md) — a companion spec targeting the Pipelines page as a reinforcement surface. This Org Home spec is V1; the Pipelines page ships later.

---

## Use Cases

### UC-1: Eligible project, no features enabled

A user opens Org Home. Their `web-ui-consolidated` project uses Jest and runs with `parallelism: 4`, making it eligible for TIA and DTS.

**Expected experience:** A subtle Smarter Testing section appears inside the project card, below the Overview/Pipelines/Deploys nav links. It reads: `⚡ 2 optimizations available →`. Clicking anywhere on the hook opens a slide-out panel with one card per eligible feature, each showing an estimated monthly savings and CTAs to set up.

---

### UC-2: One feature active, more available

A user enabled TIA last week. DTS is still available.

**Expected experience:** The hook reads `✓ 1 active · ⚡ +1 more available →`. Clicking opens a panel with an active card for TIA (showing this-run metrics) and an available card for DTS (showing estimated savings and setup CTAs).

---

### UC-3: All eligible features active

All features the project qualifies for are enabled.

**Expected experience:** The hook reads `✓ Smarter Testing active →`. Clicking opens a panel showing all active feature cards with per-run actuals. A footer CTA links to Test Insights for deeper analysis.

---

### UC-4: Project not eligible

The project does not meet eligibility signals for any feature.

**Expected experience:** No Smarter Testing section appears. Card is unchanged.

---

### UC-5: User dismisses a feature suggestion

A user opens the panel for a project and clicks "Not interested" on the TIA card.

**Expected experience:** TIA is removed from that project's suggestion list. The hook updates to reflect the remaining available count (e.g., `⚡ 1 optimization available →`). A "Restore dismissed suggestions" link appears inside the panel. Dismissal is persisted in `localStorage` with a per-project, per-feature key.

---

## Card States

Eligibility is evaluated at the **project level** (not per-run). All four states apply to the Smarter Testing section within each project card.

### State A — Not eligible
No Smarter Testing section is added. The card looks identical to today.

---

### State B — Eligible, none active

A Smarter Testing section appears below the nav links:

```
┌──────────────────────────────────────────────┐
│ web-ui-consolidated                          │
│ Overview  Pipelines  Deploys                 │
├──────────────────────────────────────────────┤
│ ⚡ 3 optimizations available  →          [✕] │
├──────────────────────────────────────────────┤
│ LAST RUN    4 minutes ago  ✕                 │
│ LAST TRIGGERED BY  spike/stripe-billing...   │
└──────────────────────────────────────────────┘
```

- Icon: lightning bolt (⚡), color: amber
- Count: number of eligible features that haven't been dismissed
- Clicking anywhere on the hook row opens the slide-out panel
- `[✕]` dismiss button hides the section for this project

---

### State C — Partially active

```
┌──────────────────────────────────────────────┐
│ web-ui-consolidated                          │
│ Overview  Pipelines  Deploys                 │
├──────────────────────────────────────────────┤
│ ✓ 1 optimization active  ·  ⚡ +2 optimizations available  →  [✕] │
├──────────────────────────────────────────────┤
│ LAST RUN    4 minutes ago  ✕                 │
└──────────────────────────────────────────────┘
```

- Green `✓ N optimization(s) active` count for enabled features
- Amber `⚡ +N optimization(s) available` count for remaining eligible features
- Both click to open the panel

---

### State D — All eligible features active

```
┌──────────────────────────────────────────────┐
│ web-ui-consolidated                          │
│ Overview  Pipelines  Deploys                 │
├──────────────────────────────────────────────┤
│ ✓ Smarter Testing active  →                 │
├──────────────────────────────────────────────┤
│ LAST RUN    4 minutes ago  ✓                 │
└──────────────────────────────────────────────┘
```

- Single green line; no dismiss button (nothing left to set up)
- Clicking opens the panel showing all active feature cards + Insights CTA

---

## Slide-Out Panel

The panel is identical in structure and content to the Pipelines page spec. See [`surface-features-pipeline-page/user-flows.md`](../surface-features-pipeline-page/user-flows.md) for full panel content specification.

**Summary:**
- **Available feature card:** feature name, description, estimated monthly savings (amber), tech-stack tags, "Install" button (creates branch + PR), "View docs" link, "Not interested" dismiss option
- **Active feature card:** feature name, per-run actuals (green), "View run details" link
- **State D footer:** "Open Insights" CTA linking to Test Insights for the project
- **After dismissal:** "Restore dismissed suggestions" link appears in panel body

Panel opens from the right side of the viewport. Main content shifts left by the panel width (400px). Panel can be closed with the `✕` button in the panel header.

---

## Eligibility Signals (project-level, not per-run)

Same signals as the Pipelines page spec. Reproduced here for reference:

| Feature | Proxy signal |
|---|---|
| Test Impact Analysis | Supported framework detected (Jest, pytest, Go, Vitest, RSpec); single-process test execution; not E2E-heavy |
| Dynamic Test Splitting | `parallelism > 1` in `.circleci/config.yml` for any test job |
| Auto Rerun Failed Tests | `store_test_results` configured; flaky test history present in Test Insights |

**TIA disqualifiers:** Cypress/Playwright projects testing against a running application (separate service) are excluded. TIA requires single-process coverage.

**Feature adoption logic:** If any single job in a project uses a feature, that feature counts as active. Full coverage across all jobs is not required.

**Detection caveat:** Test runners hidden behind Makefile targets or wrapper scripts won't be surfaced by parsing `.circleci/config.yml` alone. The detection pipeline needs to account for this indirection.

---

## Savings Display Logic

Identical to the Pipelines page spec:

- **Inactive features → monthly aggregate estimates.** Based on project activity over the past 30 days. Do not show per-run framing for inactive features.
- **Active features → per-run actuals.** Compare current run against the rolling 30-day average prior to feature activation. e.g., `83% fewer tests` or `4m 54s saved this run`.

---

## Dismissal UX

- The `[✕]` button on the hook row dismisses the **entire Smarter Testing section** for that project.
- Inside the panel, "Not interested" on a specific feature card dismisses that feature from the project's suggestions.
- Dismissal is stored in `localStorage` under a key per project + feature (e.g., `st-dismissed:web-ui-consolidated:TIA`).
- A "Restore dismissed suggestions" link appears inside the panel whenever any feature has been dismissed.
- There is no org-level or global dismiss — dismissal is per-user, per-browser.

---

## Decisions Made

| Decision | Rationale |
|---|---|
| Org Home first, Pipelines page later | Amplitude data shows Org Home is the highest-traffic surface. Shipping here first maximizes discovery reach. |
| Inline section within the card (not badge, not separate section) | Badges tested as too subtle for the card-grid layout. A separate bottom section keeps the card's nav links and last-run info intact while giving ST its own space. |
| Single-line hook (minimalist) | Keeps cards compact. Full detail lives in the panel — the hook's job is curiosity, not information density. |
| Same slide-out panel as Pipelines page | Reuses the design system pattern and avoids a second spec for panel content. Consistency across surfaces. |
| Per-project dismiss (not per-feature from the card) | Card-level dismiss is one-click; feature-level dismiss is available inside the panel for finer control. |
| All eligible projects shown (no cap) | Org Home already limits visible projects via the pinned/recent list. Capping within that would hide relevant signals. |
| No savings number in the hook | The single-line hook stays scannable. Savings details live in the panel to reward curiosity. |
| "Install" button over Claude Code deeplink | CircleCI already has the infrastructure to onboard a project to each feature. An "Install" button that creates a branch + PR removes the need for any local tooling (Claude Code CLI, terminal access) and puts the implementation directly in a PR the customer can review and merge at their own pace. |

---

## Out of Scope (V1)

- Pipelines page surface (tracked in [`surface-features-pipeline-page`](../surface-features-pipeline-page/readme.md))
- Legacy Org Home (if any)
- Mobile/responsive layout
- Org-level or team-level rollout controls for the ST section
- Projects without JUnit XML (non-standard test runners)
- In-depth savings analytics page (link to Test Insights instead)
- Server-side persistence of dismissal state (localStorage only in V1)

---

## Install Flow

When a user clicks "Install" on a feature card in the panel, CircleCI's backend:

1. Creates a new branch on the project's VCS repo (e.g. `circleci/setup-test-impact-analysis`)
2. Commits the necessary config changes (`.circleci/test-suites.yml` with the feature enabled and the detected settings pre-filled)
3. Opens a PR against the project's default branch with a description explaining the change

The button transitions through states:
- **Default:** `Install`
- **In-progress:** `Creating PR…` (spinner, button disabled)
- **Success:** `✓ PR created  View PR →` (link to the PR on GitHub/GitLab/Bitbucket)
- **Error:** `Failed — try again` (with a "View docs" fallback link)

The "Install" button is idempotent: if a PR already exists for this feature on this project, clicking again links to the existing PR instead of creating a duplicate.

---

## Open Questions

1. **Eligibility data API** — Which project-level signals (framework, parallelism, JUnit, flaky history) are queryable from the Org Home backend? Confirm with eng.
2. **VCS write permissions** — Does CircleCI already have write access (branch + PR creation) scoped to every connected repo, or does this require re-requesting OAuth scopes for some users? Confirm with the VCS integrations team.
3. **PR content generation** — How does the backend know what to put in `test-suites.yml`? It needs the detected framework, parallelism value, and any existing config. Confirm what data is available at install-click time.
4. **Branch naming and PR template** — Proposed branch: `circleci/setup-<feature-slug>`. PR description: short explanation of what the change does + link to docs. Define the template.
5. **Idempotency** — If the user clicks "Install" a second time (or the PR was closed without merging), what happens? Create a new PR, reopen the old one, or link to the closed PR?
6. **Post-merge activation** — After the customer merges the PR, does the feature become active immediately on the next pipeline run, or is there additional setup required?
7. **Baseline for savings actuals** — Is the pre-activation 30-day average run duration accessible in the product backend for showing per-run savings on active-feature cards?
8. **Analytics** — Should clicking "Install" fire an Amplitude event? Proposed: `smarter_testing_install_clicked` with properties `feature`, `source: org_home`, `project`. Also track PR-created and PR-merged events for funnel analysis.
9. **Org Home pagination** — Eligible projects outside the initial viewport: confirm lazy-loading behavior with eng.

---

## Next Steps (as of 2026-08-10)

1. **Eng alignment** — Share this spec with the Org Home eng team; confirm which eligibility signals are available in the Org Home data layer.
2. **Data team** — Identify eligible projects per feature using `config.yml` signals, Insights API data, and test results. This powers both this surface and the Pipelines page spec.
3. **Manual outreach** — Leverage TSMs and Field Engineers to inform customers ahead of the in-product experience.
4. **Phase the build** — Ship per feature rather than waiting to surface all three at once.
5. **Feedback loop** — Explore an inline feedback mechanism on the suggestions so customers can signal faster.

---

## Changelog

| Date | Who | Change |
|---|---|---|
| 2026-08-10 | Luis Jiménez | Created spec and prototype for Org Home surface. Amplitude data points to Org Home as highest-traffic page. |
| 2026-08-10 | Luis Jiménez | Replaced "Set up with Claude Code" CTA with "Install" button (creates branch + PR). Removes CLI dependency; uses existing CircleCI onboarding infrastructure. |

---

## References

- [Prototype (mockup.html)](./mockup.html)
- [User flows](./user-flows.md)
- [Pipelines page spec (companion)](../surface-features-pipeline-page/readme.md)
- [Getting started with Smarter Testing](https://circleci.com/docs/guides/test/getting-started-with-smarter-testing/)
- [Amplitude dashboard](https://app.amplitude.com/analytics/circleci/dashboard/fz92hq7s)
- [Ideas/notes doc](https://docs.google.com/document/d/1ozecsP6-9RcR6cIoCK0D1R2CKnKCUt8cwGzr2GH9BI4/edit?tab=t.m2hbu0vq0o3j)
- Slack: `#smarter-testing-private` (C0ACNPLRZ7A) · `#smarter-testing`
