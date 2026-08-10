# User Flows: Surface Smarter Testing on Org Home

**Experiment:** `smarter-testing/surface-features-org-home`  
**Status:** Draft — for eng/PM discussion  
**Target UI:** Org Home page (Organization Home)  
**Features:** Test Impact Analysis · Dynamic Test Splitting · Auto Rerun Failed Tests

---

## Background

Amplitude data shows the Org Home page is the highest-traffic surface in CircleCI — more visited than the Pipelines page or Project Overview. The Org Home project card grid is where users check their most important projects daily. Surfacing Smarter Testing eligibility here creates a discovery path at the top of the user's daily workflow.

Eligibility is evaluated at the **project level** (not per-run) and cached. The same hook appears on a project's card until the user sets up a feature, dismisses it, or eligibility changes.

---

## Features Overview

| Feature | What it does | Key eligibility signal |
|---|---|---|
| **Test Impact Analysis (TIA)** | Runs only tests affected by code changes | Supported framework (Jest, pytest, Go, Vitest, RSpec); single-process tests; not E2E-heavy |
| **Dynamic Test Splitting (DTS)** | Distributes tests evenly across parallel nodes | `parallelism > 1` in config for any test job |
| **Auto Rerun Failed Tests (ARFT)** | Retries failed test atoms automatically | `store_test_results` configured; flaky test history present |

---

## Card States

### State A — Not eligible
No Smarter Testing section. Card is unchanged. Project does not meet eligibility signals for any feature.

---

### State B — Eligible, none active

An amber hook row appears below the nav links on the project card:

```
⚡ N optimizations available  →  [✕]
```

- N = number of undismissed eligible features
- Amber color signals opportunity (not error)
- `[✕]` button on the right dismisses the section for this project
- Clicking anywhere on the hook row (except ✕) opens the slide-out panel

If only 1 feature eligible: `⚡ 1 optimization available  →  [✕]`

---

### State C — Partially active

```
✓ N optimization(s) active  ·  ⚡ +M optimization(s) available  →  [✕]
```

- Green `✓ N optimization(s) active` for enabled features
- Amber `⚡ +M optimization(s) available` for remaining unenabled features
- Clicking opens the panel showing a mix of active cards and available cards
- `[✕]` dismisses the available suggestions for this project (active status remains visible)

---

### State D — All eligible features active

```
✓ Smarter Testing active  →
```

- Single green line; no ✕ button
- Clicking opens the panel showing all active feature cards with per-run metrics
- Panel footer has "Open Insights" CTA for the project

---

## Primary Flows

---

### Flow 1: Discovery → Install

**Trigger:** User visits Org Home. Their project is eligible for one or more features (State B or C).

```
1. User visits Org Home
   └─ Project card for "web-ui-consolidated" shows:
      "⚡ 2 optimizations available  →  [✕]"

2. User clicks the hook row
   └─ Slide-out panel opens from the right
   └─ Main content shifts left (400px)
   └─ Panel header: "Smarter Testing  ·  web-ui-consolidated"
   └─ Panel shows one card per eligible feature:
      ┌─────────────────────────────────────────┐
      │ ⚡ Test Impact Analysis  [Available]     │
      │ Run only tests affected by your changes  │
      │                                          │
      │ Est. ~15m/month                          │
      │ Based on 145 tests/day, ~60% skip rate   │
      │                                          │
      │ [Install]   View docs ↗                  │
      │                       Not interested     │
      └─────────────────────────────────────────┘

3. User clicks "Install"
   └─ Button transitions to "Creating PR…" (spinner, disabled)
   └─ CircleCI backend:
        a. Creates branch: circleci/setup-test-impact-analysis
        b. Commits .circleci/test-suites.yml with TIA config pre-filled
           (framework: jest, detected from project config)
        c. Opens PR against the project's default branch

4. On success:
   └─ Button transitions to: "✓ PR created  View PR →"
   └─ "View PR →" opens the PR on GitHub/GitLab/Bitbucket in a new tab

5. Customer reviews and merges the PR
   └─ On next pipeline run, TIA is active
   └─ On next Org Home load, card shows State C or D

[On error:]
   └─ Button transitions to: "Failed — try again"
   └─ Secondary link: "View docs ↗" as a manual fallback
```

---

### Flow 2: Already active → Insights

**Trigger:** User visits Org Home. Their project has all eligible features active (State D).

```
1. User visits Org Home
   └─ Project card shows:
      "✓ Smarter Testing active  →"

2. User clicks the hook row
   └─ Slide-out panel opens
   └─ Panel shows all active feature cards with per-run actuals:
      ┌─────────────────────────────────────────┐
      │ ✓ Test Impact Analysis  [Active]         │
      │ Running. Selecting only affected tests.  │
      │                                          │
      │ This run:  83% fewer tests ran           │
      │ 52 of 312 tests · 3m 12s saved           │
      │                                          │
      │ [View run details ↗]                     │
      └─────────────────────────────────────────┘

3. Panel footer shows:
   "⌁ Open Insights — view flaky tests, slowest tests, and savings history"
   [Open Insights →]

4. User clicks "Open Insights"
   └─ Navigates to Test Insights for that project
      (e.g. /insights/web-ui-consolidated/tests)
```

---

### Flow 3: Dismiss

**Trigger:** User sees the ST hook on a card and wants to hide it.

#### Sub-flow 3a: Card-level dismiss (entire section)

```
1. User clicks [✕] on the hook row
   └─ Smarter Testing section animates out (fade + collapse)
   └─ Card returns to its default State A appearance
   └─ Dismissed state stored in localStorage:
      Key: "st-card-dismissed:web-ui-consolidated"

   [No immediate "restore" link on the card — avoids noise]
   [Restore available inside the panel if user reopens it]
```

#### Sub-flow 3b: Per-feature dismiss (inside panel)

```
1. User opens panel for a project (State B or C)
2. User clicks "Not interested" under a specific feature card (e.g. DTS)
   └─ DTS card fades out
   └─ Remaining available count in panel header updates
   └─ Hook row on the card updates:
      Was: "⚡ 2 optimizations available"
      Now: "⚡ 1 optimization available"
      (State C would read: "✓ 1 optimization active · ⚡ +1 optimization available")
   └─ Dismissal stored in localStorage:
      Key: "st-dismissed:web-ui-consolidated:DTS"
   └─ "↩ Restore dismissed suggestions" link appears in panel body

3. If all available features are dismissed:
   └─ Panel body shows: "All suggestions dismissed for this project."
   └─ Hook row on card hides (card appears as State A)
   └─ "↩ Restore dismissed suggestions" still in panel (accessible by clicking the card… wait)
   
   [Note for eng: if section is hidden after all dismissed, how does user
    access "restore"? Options: keep a muted "..." or gear icon on the card,
    or only expose restore through a settings/notifications area. TBD.]
```

---

## Eligibility Logic Detail

Eligibility is evaluated at the **project/repo level** and cached — not recalculated per pageview.

### Test Impact Analysis
- Supported test framework detected in repo (Jest, pytest, Go test, Vitest, RSpec)
- Tests run in a single process (not across separate containers/services)
- `test-suites.yml` not yet configured with `test-impact-analysis: true`
- **Disqualifier:** Cypress or Playwright detected with tests calling a running application → excluded

### Dynamic Test Splitting
- `parallelism > 1` found in `.circleci/config.yml` for any test job
- `test-suites.yml` not yet configured with `dynamic-test-splitting: true`
- _(Note: distinguish from legacy `circleci tests split` — show "Upgrade from legacy" rather than fresh setup)_

### Auto Rerun Failed Tests
- `store_test_results` present in config (JUnit XML output confirmed)
- Flaky test history detected via Test Insights (tests passing and failing on the same commit within a 14-day window)
- `test-suites.yml` not yet configured with `max-auto-rerun`

---

## Savings Calculation

### Inactive features (estimated monthly)
Show an estimate based on project activity over the past 30 days:

| Feature | Basis |
|---|---|
| TIA | Internal benchmarks or pilot data (e.g. "30–60% fewer tests per PR for projects with >200 tests") |
| DTS | `avg run duration × imbalance factor based on parallelism count` (formula TBD with data team) |
| ARFT | Flaky failure frequency from Test Insights × avg rerun time |

### Active features (per-run actuals)
- Rolling **30-day average** run duration, measured from the day before feature activation
- Surface as: `Xm Ys saved this run` or `X% fewer tests ran`

---

## CTA: Install Button

The "Install" button on each available-feature card triggers CircleCI's backend to create a branch and PR on the project's VCS repo, with the feature fully configured and ready for the customer to review.

### What CircleCI creates

**Branch name:** `circleci/setup-<feature-slug>`
- TIA: `circleci/setup-test-impact-analysis`
- DTS: `circleci/setup-dynamic-test-splitting`
- ARFT: `circleci/setup-auto-rerun-failed-tests`

**Committed file:** `.circleci/test-suites.yml` — created or updated with the feature block pre-filled using detected project signals (framework, parallelism value, etc.).

**PR description:** Short plain-English explanation of the change + link to the Smarter Testing docs for the feature.

### Button states

| State | Label | Behavior |
|---|---|---|
| Default | `Install` | Clickable |
| In-progress | `Creating PR…` | Spinner; button disabled |
| Success | `✓ PR created  View PR →` | Opens VCS PR in a new tab |
| Already exists | `View existing PR →` | Links to the open or merged PR |
| Error | `Failed — try again` | Retryable; "View docs ↗" shown as fallback |

### Idempotency
If a PR for this feature already exists on the project (open or merged), "Install" links to the existing PR rather than creating a duplicate.

---

## Edge Cases

| Case | Handling |
|---|---|
| Project eligible for only 1 feature | Hook reads `⚡ 1 optimization available  →`. Singular form. |
| User dismisses all features for a project | Hook section hides (State A appearance). Access to "Restore" TBD — see Flow 3b note. |
| All projects on Org Home are State A | Page looks identical to today. No ST surface at all. |
| Project uses legacy `circleci tests split` | DTS suggestion reads "Upgrade from legacy test splitting" rather than treating it as a fresh setup. |
| Org on unsupported auth (no Test Insights access) | ARFT eligibility falls back to JUnit signal only (no flaky history). Show indicator with reduced confidence. |
| `test-suites.yml` configured but feature flag off | Show as "Paused" rather than "Available". Install button changes to "Re-enable". (Requires detecting partial config state — TBD.) |
| User has 20+ projects on Org Home | All eligible projects show ST section. No cap. Eligible projects in view get the signal; off-screen projects get it when scrolled into view. |
| Project mid-run when user loads Org Home | Eligibility is project-level, not run-level. ST section shows regardless of run status. |

---

## What We're Not Solving in V1

- Pipelines page surface (separate spec in `surface-features-pipeline-page/`)
- Per-user server-side persistence of dismissal state (localStorage only)
- In-depth savings analytics page (link to Test Insights instead)
- Mobile/responsive layout
- Org-level or team-level rollout controls for the ST section
- Legacy Org Home (if any)

---

## Related Links

- [Spec (readme.md)](./readme.md)
- [Prototype (mockup.html)](./mockup.html)
- [Pipelines page spec (companion)](../surface-features-pipeline-page/readme.md)
- [Smarter Testing getting started docs](https://circleci.com/docs/guides/test/getting-started-with-smarter-testing/)
- [Test Impact Analysis setup](https://circleci.com/docs/guides/test/set-up-test-impact-analysis/)
- [Dynamic Test Splitting setup](https://circleci.com/docs/guides/test/use-dynamic-test-splitting/)
- [Auto Rerun Failed Tests setup](https://circleci.com/docs/guides/test/auto-rerun-failed-tests/)
- [Amplitude dashboard](https://app.amplitude.com/analytics/circleci/dashboard/fz92hq7s)
- Slack: `#smarter-testing-private` · `#smarter-testing`
