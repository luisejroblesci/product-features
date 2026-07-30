# User Flows: Surface Smarter Testing on Pipelines Page

**Experiment:** `smarter-testing/surface-features-pipeline-page`  
**Status:** Draft — for eng/PM discussion  
**Target UI:** New Pipelines page (Updated view toggle ON)  
**Features:** Test Impact Analysis · Dynamic Test Splitting · Auto Rerun Failed Tests

---

## Background

The Pipelines page is CircleCI's highest-traffic page (Amplitude: [dashboard](https://app.amplitude.com/analytics/circleci/dashboard/fz92hq7s)). The three Smarter Testing features go undiscovered because adoption requires users to find and follow setup docs independently. Surfacing eligibility indicators directly in the pipeline row creates a low-friction discovery path at the moment users are already looking at their CI data.

Each row on the Pipelines page represents a triggered pipeline run. Eligibility is evaluated at the **project level** (not per-run), so the same indicator appears for every run belonging to an eligible project.

---

## Features Overview

| Feature | What it does | Key eligibility signal |
|---|---|---|
| **Test Impact Analysis (TIA)** | Runs only tests affected by code changes | Supported framework detected (Jest, pytest, Go, Vitest, RSpec); single-process tests |
| **Dynamic Test Splitting (DTS)** | Distributes tests evenly across parallel nodes | `parallelism > 1` in config |
| **Auto Rerun Failed Tests (ARFT)** | Retries failed test atoms automatically | `store_test_results` configured; flaky test history present |

---

## Row States

### State A — Not eligible
No change to the row. The project does not meet the minimum eligibility signals for any feature.

---

### State B — Eligible, none enabled

**Trigger:** Project is eligible for 1–3 features, none currently active.

**Row addition:**  
A small pill badge appears in the pipeline row, inline with the run metadata:

```
⚡ Optimize tests  (2 available)
```

- Icon: lightning bolt
- Color: amber/yellow — signals opportunity, not warning or error
- Count: number of eligible features
- If only 1 feature: `⚡ Optimize tests  (1 available)`

**Hover state (tooltip):**  
Lists each eligible feature with a one-line estimated saving:

```
Smarter Testing available
─────────────────────────────
⚡ Test Impact Analysis
   Skip tests unaffected by your changes
   Est. 30–60% fewer tests per PR

⚡ Dynamic Test Splitting
   Balance load across your 4 parallel nodes
   Est. ~1m 20s saved per run

→ Click to set up
```

**Click behavior:**  
Opens a slide-out panel (right side) or inline expansion below the row.

**Panel content:**  
One card per eligible feature, stacked vertically.

```
┌─────────────────────────────────────────────────────┐
│ ⚡ Test Impact Analysis                              │
│ Run only the tests affected by your code changes.   │
│                                                     │
│ Est. savings: ~40% fewer tests per PR               │
│ Works with: Jest, pytest, Go, Vitest, RSpec         │
│                                                     │
│  [Set up with Claude Code]   View docs ↗            │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ ⚡ Dynamic Test Splitting                            │
│ Distribute tests evenly so all 4 nodes finish       │
│ at the same time.                                   │
│                                                     │
│ Est. savings: ~1m 20s per run                       │
│ Requires: parallelism > 1 (already configured ✓)   │
│                                                     │
│  [Set up with Claude Code]   View docs ↗            │
└─────────────────────────────────────────────────────┘
```

**CTA: "Set up with Claude Code"**  
Deeplink that opens Claude Code with a pre-populated prompt for this feature.  
See [Open Questions — CTA Deeplink](#open-questions) for protocol details.

---

### State C — Partially active (some features on, some eligible)

**Trigger:** Project has 1+ features active, and is eligible for at least one more.

**Row:**  
Two badges appear side by side:

```
✓ TIA active    ⚡ Optimize tests  (1 available)
```

- Green `✓` badge for active features (one per active feature, or consolidated if 2–3)
- Amber `⚡` badge for remaining eligible, unenabled features

**Click: Panel shows mixed cards**

Active feature card:
```
┌─────────────────────────────────────────────────────┐
│ ✓ Test Impact Analysis  — Active                    │
│                                                     │
│ This run:   47 of 312 tests selected  (-85%)        │
│ Time saved: 3m 12s vs. 30-day avg                   │
│                                                     │
│  [View run details ↗]                               │
└─────────────────────────────────────────────────────┘
```

Unenabled eligible card: same as State B card.

---

### State D — All eligible features active

**Trigger:** All features the project is eligible for are currently enabled.

**Row:**
```
✓ Smarter Testing active
```

Single green badge. Replaces the amber badge entirely.

**Hover tooltip:**
```
Smarter Testing active
─────────────────────────────
✓ Test Impact Analysis   — 3m 12s saved this run
✓ Dynamic Test Splitting — 1m 08s saved this run
✓ Auto Rerun Failed Tests — 2 flakes recovered
```

**Click: Panel shows all active cards** with actual savings data (same card format as State C active card).

---

## Eligibility Logic Detail

Eligibility is evaluated at the **project/repo level** and cached — not recalculated per run.

### Test Impact Analysis
- Supported test framework detected in repo (Jest, pytest, Go test, Vitest, RSpec)
- Tests run in a single process (not across separate containers/services)
- `test-suites.yml` not yet configured with `test-impact-analysis: true`

### Dynamic Test Splitting
- `parallelism > 1` found in `.circleci/config.yml` for any test job
- `test-suites.yml` not yet configured with `dynamic-test-splitting: true`
- *(Note: project may use legacy `circleci tests split`. Distinguish from new DTS — see Edge Cases)*

### Auto Rerun Failed Tests
- `store_test_results` present in config (JUnit XML output confirmed)
- Flaky test history detected via Test Insights data (tests that passed and failed on the same commit within a 14-day window)
- `test-suites.yml` not yet configured with `max-auto-rerun`

> **TBD with eng:** Confirm which of these signals are queryable from pipeline run data via the CCI API. These are proxy signals; actual implementation may differ.

---

## Savings Calculation

### Inactive features (estimated)

| Feature | Basis for estimate |
|---|---|
| Test Impact Analysis | Industry benchmarks or internal pilot data (e.g. "30–60% fewer tests per PR for projects with >200 tests") |
| Dynamic Test Splitting | `(avg run duration) × (imbalance factor based on parallelism count)` — TBD formula |
| Auto Rerun Failed Tests | Flaky test frequency from Test Insights × avg rerun time |

### Active features (actual)

- Rolling **30-day average** run duration for the project, measured from the date before feature activation
- Delta = `baseline_avg - current_run_duration`
- Surface as: "Xm Ys saved this run" or "Xm Ys avg saved"

> **TBD with data/analytics:** Confirm data availability for pre-activation baseline. Amplitude dashboard is one reference, but run-level data needs to be accessible in the product backend.

---

## CTA: Claude Code Deeplink {#open-questions}

### Proposed approach

Each "Set up with Claude Code" button constructs a deeplink:

```
claude://open?prompt=<url-encoded-prompt>
```

The prompt is pre-populated with:
- The specific feature to set up
- The user's project context (repo name, detected language, existing config snippets)
- Instructions to follow the Smarter Testing setup guide

**Example prompt (Test Impact Analysis, Jest project):**
```
I want to set up Test Impact Analysis for my CircleCI project "my-org/my-repo".
It's a JavaScript project using Jest. Please follow the steps at 
https://circleci.com/docs/guides/test/set-up-test-impact-analysis/ 
and help me configure .circleci/test-suites.yml with test-impact-analysis: true.
```

### Happy path (Claude Code installed)
1. User clicks "Set up with Claude Code"
2. Browser opens `claude://` URI → Claude Code launches (or surfaces from tray)
3. Claude Code opens with the pre-populated prompt
4. User follows the guided setup in their terminal

### Fallback (Claude Code not installed)
1. Browser attempts `claude://` URI → nothing happens / OS shows error
2. After 2–3 seconds with no response, the CTA falls back:
   - Show an install prompt: "Claude Code not detected. [Install Claude Code] or [Open docs instead]"
   - "Install Claude Code" → `https://claude.ai/code` (or CLI install page)
   - "Open docs instead" → feature-specific docs page

### Open questions for eng
1. Does a `claude://` URI scheme exist? If not, does it need to be registered as part of the Claude Code CLI installer?
2. Is there a web-based alternative (claude.ai/code or similar) that doesn't require local install?
3. How is fallback detection handled cross-browser? (The 2–3s timeout heuristic is imprecise)
4. Should the prompt include project context (repo, language, existing YAML snippets)? If yes, what data is available in the Pipelines page at time of click?
5. Should there be analytics on CTA clicks to measure conversion? (Amplitude event: `smarter_testing_cta_clicked`, properties: feature, source: pipelines_page)

---

## Edge Cases

| Case | Handling |
|---|---|
| Same project appears multiple times in the list (multiple runs) | Indicator shows on every row. Option to discuss: deduplicate so only the most recent run for a project shows the badge (reduces noise). |
| Run is in progress (Running status) | Indicator still shows — eligibility is project-level, not tied to run completion. |
| User wants to dismiss the indicator | No dismiss in V1. Flag for PM discussion: do we want a "not interested" option that suppresses for 30 days? |
| Project uses legacy `circleci tests split` (not Smarter Testing) | DTS indicator should note "Upgrade from legacy test splitting" rather than treating it as a fresh setup. Differentiate in eligibility check. |
| Org on unsupported auth (`circleci/<UID>`) | Test Insights flaky data unavailable. ARFT eligibility falls back to JUnit signal only (no flaky history). Show indicator with reduced confidence signal. |
| Smarter Testing in beta | All indicators should include a subtle "Beta" label to set expectations. |
| Project has `test-suites.yml` but feature flag is off | Already configured but disabled — show as "Paused" rather than "Set up". CTA changes to "Re-enable". (TBD — requires detecting partial configuration state.) |

---

## What We're Not Solving in V1

- Eligibility for projects without JUnit XML output (non-standard test runners)
- In-depth savings analytics page (link to existing Test Insights for now)
- Old pipelines UI (legacy view with the "Updated view" toggle OFF)
- Mobile/responsive layout
- Team-level or org-level rollout controls

---

## Related Links

- [Smarter Testing getting started docs](https://circleci.com/docs/guides/test/getting-started-with-smarter-testing/)
- [Test Impact Analysis setup](https://circleci.com/docs/guides/test/set-up-test-impact-analysis/)
- [Dynamic Test Splitting setup](https://circleci.com/docs/guides/test/use-dynamic-test-splitting/)
- [Auto Rerun Failed Tests setup](https://circleci.com/docs/guides/test/auto-rerun-failed-tests/)
- [Amplitude dashboard](https://app.amplitude.com/analytics/circleci/dashboard/fz92hq7s)
- [Ideas/notes doc](https://docs.google.com/document/d/1ozecsP6-9RcR6cIoCK0D1R2CKnKCUt8cwGzr2GH9BI4/edit?tab=t.m2hbu0vq0o3j)
- Slack: `#smarter-testing-private` · `#smarter-testing`
