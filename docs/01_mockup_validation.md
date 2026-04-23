# Mockup Validation — `zero-debt mockups _standalone_.html`

Validates the 10 screens bundled in the mockup HTML against the product workflow, architecture contracts, and edge cases a developer must handle during implementation.

**Verdict:** Mockups cover the **P1 golden path end-to-end** at high fidelity. There are well-bounded, mostly peripheral gaps. Explicit guidance per gap is in §4.

---

## 1. Mockup Inventory

Screens extracted from the bundle (source: `screens_auth_repos.js`, `screens_repo_detail_phase.js`, `screens_runs.js`, `shared_chrome.js`):

| # | Screen ID | Component | Purpose |
|---|---|---|---|
| 1 | Login | `LoginScreen` | DB-auth email+password form; Argon2id / 8h session TTL callouts; "Reset" and "Request access" links (not themselves mocked) |
| 2 | Repos list | `ReposScreen` | Table of connected repos with org/name, branch, language, last run + phase tag, status, open PRs, issues; search + filter + "Connect repository" |
| 3 | Connect repo | `ConnectRepoScreen` | 3-step form (URL+branch, PAT w/ validation, phase enablement) |
| 4 | Repo detail / settings | `RepoDetailScreen` | Global repo settings + per-phase overrides (P1 expanded) + access panel + recent activity + disconnect |
| 5 | Phase picker (new run) | `PhasePickerScreen` | Repo+branch selector + 5-card capability grid (P4/P5 marked `preview`) |
| 6 | Sonar Fix configure | `SonarFixFormScreen` | Tabs for issue source (Upload / Live API / Paste), fix strategy, rule allowlist, delivery, "Enqueue run" CTA |
| 7 | Run progress (live) | `RunProgressScreen` | Stats tiles, LangGraph node timeline, streaming LLM call panel, SSE event log |
| 8 | Run summary | `RunSummaryScreen` | Stats tiles, per-issue disposition table, delivery card, top-rules chart, token/latency panel |
| 9 | Runs history | `RunsHistoryScreen` | Activity-strip stats + filterable run table |
| 10 | Run audit | `RunAuditScreen` | Checkpoints list, security/audit entries, LLM interactions archive table |

Shared chrome present: `TopBar`, `SideNav`, `AppShell`, `Icon`, `PHASES` metadata, `Stat`, `Row`, `statusChip`, design tokens (`--bg-0`, `--fg-*`, `--ok`, `--warn`, `--err`, `--accent`, phase color tags `p1..p5`).

---

## 2. Architecture Alignment Check

Evaluating against the five core workflows.

| Workflow | Alignment | Evidence in mockup | Status |
|---|---|---|---|
| **Auth** (ADR-0013) | Matches | Login copy "Argon2id hashing · sessions stored in Redis · 8h TTL"; footer `v0.4.2 · rocky-9-demo` signals DB-auth stage | ✅ |
| **Repo Management with overrides** | Matches | RepoDetail shows global settings + per-phase overrides merged `over config.phases.*` | ✅ |
| **Run Configuration** | Matches | PhasePicker → SonarFixForm → "Enqueue run"; queued onto Redis per copy | ✅ |
| **Live Execution (SSE + LangGraph)** | Matches strongly | Node-by-node graph timeline with exactly the PHASE-1-BLUEPRINT node names; SSE channel `zerodebt:run:r_7a2f8c1e` annotated; heartbeat 15s noted | ✅ |
| **Audit History** | Matches | RunAudit shows LG checkpoints, branch-write guards, default-branch check, `zerodebt_telemetry.llm_interactions` collection name | ✅ |
| **No default-branch writes** (ADR-0010) | Matches | SonarFixForm delivery card: "allow_default_branch=false — will refuse to write to main" | ✅ |
| **Subgraph contract** (ADR-0002) | Matches | RepoDetail "Phase overrides" treats each phase as a separately-toggleable block | ✅ |
| **Polyglot persistence** (ADR-0016) | Matches | Audit view references `mongo · zerodebt_telemetry.llm_interactions`; sidebar footer "1.3k Mongo evt/h" | ✅ |
| **LLM Gateway logical ids** (ADR-0004) | Matches | SonarFixForm uses `llm_model_role: code-smart`; progress shows `claude-sonnet-4-6` resolved | ✅ |

No architectural misalignments detected. Copy uses the right vocabulary (subgraph, checkpoint, phase_input, KEK, Arq worker, SSE, Mongo collection names) at the right places.

---

## 3. Gaps — Missing Screens and States

### 3.1 Missing screens

| # | Screen | Referenced by | Severity |
|---|---|---|---|
| G-01 | **Signup / Request access** | Login footer "Request access" | Medium — needed for M1 |
| G-02 | **Password reset** | Login "Reset" link | Medium — needed for M1 |
| G-03 | **User profile / account settings** | Top-right avatar, session mgmt | Medium — needed for M1 |
| G-04 | **Users management (admin)** | SideNav item `users` | Low — stub acceptable until admin work |
| G-05 | **Global audit log** | SideNav item `audit` (different from per-run audit) | Low — stub acceptable |
| G-06 | **Metrics dashboard** | SideNav item `metrics` | Low — stub acceptable |
| G-07 | **App-level settings** | SideNav item `settings` | Low — stub acceptable |
| G-08 | **Phase configure forms for P2–P5** | PhasePicker cards | Low for P1 GA; required when each phase is built |
| G-09 | **GitHub OAuth connect flow** | ConnectRepo copy "PAT or OAuth" | Low — PAT suffices for v1 |

### 3.2 Missing states (per existing screen)

| # | Screen | Missing state | Severity |
|---|---|---|---|
| S-01 | Login | Invalid credentials, locked account, session-expired redirect | High — ship in M1 |
| S-02 | Repos list | Empty (zero repos connected — onboarding hero), loading skeleton | Medium |
| S-03 | Connect repo | PAT invalid / wrong scopes / repo 404 / private-repo auth failure | **High** — token errors are common |
| S-04 | Connect repo | Duplicate repo already connected | Medium |
| S-05 | Repo detail | Newly-connected repo (no runs yet) → "Recent activity" empty state | Medium |
| S-06 | Repo detail | Repo disconnected by another user / PAT expired | Medium |
| S-07 | Repo detail | **Disconnect confirmation modal** — destructive action | **High** — ship before M7 |
| S-08 | Repo detail | **Rotate PAT modal** — destructive action | High |
| S-09 | Phase picker | No repos connected yet | Medium |
| S-10 | Sonar Fix form | Upload failure / malformed JSON / schema mismatch | High |
| S-11 | Sonar Fix form | Repo currently has an in-flight run (per-repo cap exceeded) | Medium |
| S-12 | Sonar Fix form | User concurrency cap exceeded (2 active runs) | Medium |
| S-13 | Run progress | **Queued / pending** (before worker picks up) | **High** — Arq dequeue delay is real |
| S-14 | Run progress | **Paused / awaiting checkpoint resume** after worker restart | High |
| S-15 | Run progress | **Failed** with error detail and "Retry" affordance | High |
| S-16 | Run progress | SSE disconnected / reconnecting banner | Medium |
| S-17 | Run progress | **Cancel confirmation modal** | High |
| S-18 | Run summary | Partial-success state (status=PARTIAL) | Medium |
| S-19 | Run summary | Failed-run summary shape | High |
| S-20 | Runs history | Empty (no runs) | Medium |
| S-21 | Runs history | Filtered-to-zero | Low |
| S-22 | Run audit | Export NDJSON download state / progress | Low |
| S-23 | Global | 401 / session expired toast + modal | **High** — cross-cutting |
| S-24 | Global | 403 / permission denied page | Medium |
| S-25 | Global | 404 / not found | Low |
| S-26 | Global | 500 / server error page | Low |
| S-27 | Global | Network offline / API unreachable banner | Medium |

### 3.3 Interaction patterns not yet specified

- **Keyboard shortcuts** — PhasePicker hints `↵ Continue` / `Esc Cancel`, but no global keyboard map is defined.
- **Responsive breakpoints** — all mockups target ~1280px+ desktop. Tablet/mobile scaling rules absent. (`zero-debt` is a desktop tool — acceptable, but declare it.)
- **Accessibility** — focus styles present in tokens; full a11y pass (ARIA labels, form error association, color-contrast audit on phase tags) is not demonstrated.
- **SSE reconnect UX** — event stream panel shows `last event 0.2s ago`; no reconnect indicator.
- **Toast notification system** — no toast primitive shown; ship one to cover errors + success messages.

---

## 4. Recommendations — Designer vs Developer

### 4.1 **Designer-owned** (block merge, request updated mocks)

These affect destructive or trust-critical flows and warrant explicit UX decisions.

| ID | Item | Why designer |
|---|---|---|
| S-03 | PAT validation error states on Connect repo | Wrong UX here = user blames zero-debt, not their token |
| S-07 | Disconnect repository confirmation | Destructive — irreversible from UI's POV |
| S-08 | Rotate PAT modal | Handles live secret input — needs proper shape |
| S-13 | Queued state on run progress | First thing a user sees after clicking "Enqueue run" — worth getting right |
| S-15, S-19 | Failed run detail + summary | Recovery affordance placement is not obvious |
| S-17 | Cancel run confirmation | Destructive; also confusing if click is accidental |
| S-23 | Global session-expired UX | Cross-cutting; inconsistency = confusion |
| G-01, G-02, G-03 | Signup / reset / profile | Pre-onboarding is the first impression — critical path |

### 4.2 **Developer-derivable** (no designer round-trip needed)

These are covered by existing design primitives (`zd-card`, `zd-chip`, `zd-ph`, `zd-phase-tag`, status color tokens) and common patterns.

| ID | Item | Guidance |
|---|---|---|
| S-01 | Login invalid-credentials | Inline `--err` text under input; reuse `.zd-chip.err` pattern |
| S-02, S-05, S-20 | Empty-state layouts | Reuse `zd-ph` placeholder primitive + CTA button from chrome |
| S-04, S-11, S-12 | Client-side validation errors | Inline messages under the offending field |
| S-10 | Malformed JSON upload | Inline error in `zd-ph` with retry CTA |
| S-14 | Paused / resume-from-checkpoint | Reuse running node chip with different color (`warn` tone) + inline copy |
| S-16 | SSE reconnect banner | Top-of-content banner reusing `zd-chip.warn` styling |
| S-18 | Partial-success summary | Reuse summary layout; `status=partial` already has a color token |
| S-21 | Filtered-to-zero | Inline "no results · clear filters" row |
| S-22 | Export progress | Button disabled + spinner `zd-spin` until ready |
| S-24–S-27 | Error pages | Single centered `zd-ph` card + status code + CTA |
| Responsive | Breakpoint rules | Declare "desktop-first, min-width 1024px"; collapse sidenav below that — dev call |
| Toast system | Primitive | Ship a lightweight toast service reusing chip colors; no designer needed |

### 4.3 **Deferred** (stub for M10 phase freeze, full design later)

Out-of-critical-path for P1 GA.

| ID | Item | Deferred scope |
|---|---|---|
| G-04 | Users admin screen | Stub with list + "designed later" banner |
| G-05 | Global audit log | Stub; reuse RunAudit's LLM interactions table pattern |
| G-06 | Metrics dashboard | Stub; reuse runs-history stat strip pattern |
| G-07 | App-level settings | Stub; reuse repo-settings card pattern |
| G-08 | P2–P5 configure forms | Design per-phase when that phase enters its implementation milestone |
| G-09 | GitHub OAuth flow | Design in DE-1 (deployment evolution) |

---

## 5. Binary Decision

**Do the mockups need an updated pass from a designer before P1 UI work starts?**

**Partially.** Start development immediately using the existing mockups for the P1 golden path (screens #1–#10). Block the following before M7 (UI for P1) ships:

- **Must-have designer pass (list exactly):** S-03, S-07, S-08, S-13, S-15, S-17, S-19, S-23, G-01, G-02, G-03.
- Everything else in §3 is **dev-derivable**; build using the token set and primitives already present.
- §4.3 items are **stubbed placeholders** until their own milestones arrive.

**Recommended designer work order:**

1. Auth completion pack: Login errors (S-01) + Signup (G-01) + Reset password (G-02) + Session-expired UX (S-23) + User profile (G-03). Single deliverable.
2. Destructive-action modal pack: Disconnect repo (S-07), Cancel run (S-17), Rotate PAT (S-08).
3. Recovery states pack: Queued run (S-13), Failed run + retry (S-15, S-19), PAT validation errors on Connect (S-03).

Once those three packs land, dev has everything needed through P1 GA without further designer turns.

---

## 6. Open Questions for Product / Design

Before locking the UI scope:

1. **Signup flow:** self-serve signup, invite-only, or admin-provisioned? Login copy says "Request access" — implies invite-only or admin-approval. Confirm.
2. **User roles:** only "user" and "admin"? RBAC beyond that (e.g., "viewer" on specific repos)? Currently unspecified.
3. **Concurrency cap enforcement UX:** when a user hits their cap, does "Enqueue" fail or queue-behind-cap? Current mocks don't show.
4. **Patch preview before enqueue:** should users review proposed patches before PR is opened, or is PR-then-review (current design) acceptable? Affects whether a file-diff view is required.
5. **Mobile / tablet support:** declared unsupported, limited, or responsive? Affects component breakpoints.

Resolve these before the designer starts the three packs above.
