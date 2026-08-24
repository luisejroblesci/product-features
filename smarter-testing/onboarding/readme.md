# Spec: Smarter Testing Onboarding Wizard

**Experiment:** `smarter-testing/onboarding`  
**Status:** Draft — for eng/PM/design discussion  
**Prototype:** [mockup.html](./mockup.html)  
**Local flow diagram:** [local-flow.html](./local-flow.html)

---

## Problem Statement

The Org Home and Pipelines page now surface Smarter Testing eligibility signals. The previous CTA — "Set up with Claude Code" via a `claude://` deeplink — required local CLI installation and had no defined fallback for users without it. The onboarding wizard replaces this with a hosted, guided setup flow that works for all users regardless of local tooling.

**Goal:** Take a user from "eligible, not installed" to "PR merged, analysis running" with minimal friction.

---

## Wizard Steps

A three-step accordion modal. Steps unlock sequentially; completed steps remain expandable for review.

### Step 1 — GitHub App installation

Installs the CircleCI GitHub App on the user's GitHub organization. This is a prerequisite for the wizard to create and push configuration files.

- **Why write access?** — Explicitly surfaced in a permissions box:
  - Creates `.circleci/test-suites.yml` — configures which features to enable and for which job
  - Modifies `.circleci/config.yml` — adds `store_test_results` hook required by TIA and DTS
- CTA: "Install CircleCI GitHub App ↗" — opens GitHub App install page in new tab, then "Continue →" appears
- Skip path: "I've already installed it →" — marks step as skipped, advances to step 2
- If a setup branch already exists: warning banner shown with a "View PR →" link

### Step 2 — Select a test job

Selects which job in the user's pipeline will have Smarter Testing enabled.

- Dropdown pre-populated with the recommended job (ranked by `store_test_results` + `parallelism > 1` signals)
- Hint text explains ranking criteria
- CTA: "Continue →"
- Step summary when complete: shows the selected job name

### Step 3 — Choose setup path

After selecting a job, the user picks one of two installation paths. This decision determines how the config files are created and committed.

---

## Local Path

**User mental model:** "I'll run a prompt in my AI assistant and it handles everything."

**Prerequisites shown before the copy block:**
1. Terminal open at the repo root
2. AI assistant ready (Claude, Cursor, or similar)

**What's copied:** A single complete prompt containing:
- Exact content for `.circleci/test-suites.yml` (pre-filled with the selected job)
- Instructions to add `store_test_results` to `config.yml` if missing
- Branch name to create: `circleci/smarter-testing-setup`
- Commit message and PR title

**After copying:** Auto-advances to the waiting state. The wizard shows:
- Branch chip with copy button
- Pulsing "Waiting for your PR…" indicator
- Description: "We'll detect when your PR is ready. You can close this window — the banner at the top of the page will track progress."
- Fallback CTA: "I've merged the PR →" for users who don't need detection

**Persistent banner:** While in the waiting state, an amber banner appears above the app shell with the branch name and a "View progress →" link. Survives the modal being closed.

---

## UI Path

**User mental model:** "CircleCI does it for me, I just merge the PR."

**What CircleCI does automatically:**
1. Creates branch `circleci/smarter-testing-setup`
2. Commits `.circleci/test-suites.yml` and any `config.yml` modifications
3. Opens a PR on GitHub

**What the user sees:**
- 2-second spinner: "CircleCI is creating the setup files and opening a PR…"
- Success state: branch chip + "View PR on GitHub →" link
- CTA: "I've merged the PR →"

This mirrors the Install button behavior on the Org Home surface.

---

## Duplicate Install Detection

| Condition | Handling |
|---|---|
| GH App already installed | Step 1 is skipped; wizard opens at step 2 |
| `circleci/smarter-testing-setup` branch already exists | Warning banner in step 1 body with "View PR →" link; user can continue setup or navigate to the PR |
| PR already merged (feature active) | Out of scope V1 — entry point should not offer setup to already-active projects |

---

## "You're all set!" Screen

Shown after the user confirms PR merge on either path.

**Copy:**
> We're running analysis to build data to intelligently run the right tests. Check back after a few pipeline runs to start seeing your savings.

**CTAs:** "View Test Insights →" (green) · "Close"

---

## Persistent Banner

Shown on the background page (above the app shell) when the user enters the Local path's waiting state.

- **Content:** "⏳ Smarter Testing setup in progress — `circleci/smarter-testing-setup` · [View progress →] [×]"
- **Survives:** Modal being closed; page navigation (requires infrastructure support — see open questions)
- **Dismissed by:** Clicking × or completing setup (confirmation triggers `hidePersistentBanner()`)

---

## Decisions Made

| Decision | Rationale |
|---|---|
| Replace `claude://` deeplink with wizard | Deeplink requires local CLI install with undefined fallback. Wizard works for all users. Confirmed in 2026-08-24 PM/design session. |
| Three steps (GH App → job → path) | Minimum viable sequence: permission gate, context capture, execution path. Steps 3-5 of the original wizard (guided YAML editing) are replaced by the two paths. |
| Explicit permission explanation in step 1 | Users distrust "write access to your org" without explanation. Naming the exact files reduces abandonment at this gate. |
| Local path copies a complete prompt, not step-by-step instructions | Reduces cognitive load; single paste action. The AI assistant handles the decomposition. |
| UI path auto-creates branch + PR (no guided manual steps) | Consistent with Org Home Install button. Reduces wizard steps from 5 to 3. |
| Copy → auto-advance to waiting state | Removes a redundant "I've copied it, now what?" confirmation step. User can still manually confirm merge as a fallback. |
| Skip Claude as a separate path | Local path is model-agnostic (Claude, Cursor, etc.). A dedicated Claude path would duplicate the Local path with minor copy differences. |

---

## Open Questions

1. **GH App write scope** — Does the CircleCI GitHub App already have `contents: write` for all connected repos? If some orgs need to re-authorize, step 1 UX needs to account for a re-authorization state (not just "already installed").

2. **Webhook / branch detection** — Is it technically feasible to detect when `circleci/smarter-testing-setup` is pushed so the wizard auto-advances from "waiting" to "PR detected"? What is the expected latency? This determines whether the detection is real or always requires manual "I've merged the PR" confirmation.

3. **UI path file creation** — Does CircleCI create the branch and PR via the same API used by the Org Home Install button? What data needs to be available at path-selection time (job name, framework, existing `config.yml` content)?

4. **Copy block format** — Is the monolithic prompt format preferred over structured step-by-step instructions? Are there prompt injection concerns from user-controlled values (job name, repo name) being interpolated into the prompt?

5. **VCS parameterization** — When do GitLab and Bitbucket need to be supported? The wizard currently uses "GitHub App" language throughout. Parameterizing a `vcs` variable now (`'GitHub' | 'GitLab' | 'Bitbucket'`) would prevent rework later. Connects to the branch-exists detection mechanism (GitLab/Bitbucket use different APIs).

6. **Persistent banner infrastructure** — Is there product infrastructure for a cross-page persistent banner that survives React Router navigation? Or does this need to be implemented via `localStorage` and a re-render hook on the project/pipelines page?

7. **Entry point** — Does the wizard open as an inline modal on the Pipelines page (preferred for context continuity) or as a separate route? This affects how the wizard receives project context (repo, selected feature, job list) at open time.

8. **Multiple users, same repo** — What happens if two users start the wizard for the same repo simultaneously? The branch-exists check in step 1 is the current gate, but it's not a lock. Discuss idempotency with eng.

---

## Out of Scope (V1)

- GitLab / Bitbucket support (GitHub only)
- Enabling Smarter Testing on multiple jobs in one wizard flow (single job selection)
- Post-merge activation confirmation (wizard ends at "I've merged the PR"; activation is inferred)
- In-wizard rollback / undo
- Notification when analysis is complete and ST is active
- Mobile/responsive layout

---

## Changelog

| Date | Who | Change |
|---|---|---|
| 2026-08-24 | Luis Jiménez | Created spec based on Figma design review and PM discussion. Confirmed: 3-step wizard, Local + UI paths, skip Claude path. |

---

## References

- [Prototype (interactive mockup)](./mockup.html)
- [User flows](./user-flows.md)
- [Figma design](https://www.figma.com/design/SN2k8iNUEmKojGQBH1mgzX/Smarter-Testing?node-id=3050-27998)
- [Org Home spec](../surface-features-org-home/readme.md) — Install button pattern (UI path predecessor)
- [Pipelines page spec](../surface-features-pipeline-page/readme.md)
- [Getting started with Smarter Testing](https://circleci.com/docs/guides/test/getting-started-with-smarter-testing/)
