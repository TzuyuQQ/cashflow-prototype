# CLAUDE.md

Project context for Claude Code sessions in this repo.

## Goal

This is a **demo prototype** (no backend, not shipping to production as-is) for a GCF — Global Cash
Flow — management tool. The pitch: give project managers and finance a project-scoped view of cash
health (invoiced/collected/spent/net per project) instead of a single global cash number, so risk shows
up at the project level where it's actionable.

The current focus is the **Payment Plan feature**: vendor payment forecasting used to read
`CashEntry.dueDate` (contractual/ERP due dates), which don't reflect what Finance actually intends to
pay or when. The fix makes PMs own an explicit payment plan (commitment → phases) as the sole source of
*future* vendor outflow, fully decoupled from *actual* confirmed payments. The user is actively shaping
this feature to demo to stakeholders — expect requests like "make up realistic data for every project"
and prioritize the seed data actually rendering correctly over further architecture work, unless asked.

## Repo layout

- **`cashflow-prototype/`** — the active project: a GCF (Global Cash Flow) management tool prototype. This is where all real work happens.
- **`gpm-prototype/`** — a separate, unrelated prototype. Not part of GCF; don't touch unless explicitly asked.
- Root `index.html` — just a redirect to `cashflow-prototype/index.html`.
- `Meeting_Presentation_EN.html`, `Speaker_Notes_CN.html`, `User_Guide_CN.html` — standalone docs, not app code.

Everything below describes `cashflow-prototype/`.

## What this is

A no-backend, no-build prototype for project-scoped cash flow management (发起/回款/支付 tracking across
projects), built for demoing to stakeholders. Seven pages under `GCF/` plus a portfolio dashboard
(`index.html`):

- `index.html` — 全局总览 (portfolio dashboard): KPIs, monthly chart, 项目净现金流排名 table, alerts
- `GCF/项目列表/项目列表.html` — project list with AR health bar
- `GCF/项目详情/项目详情.html` — single-project drill-down: KPIs, chart, entries, Payment Plan card
- `GCF/收支明细/收支明细.html` — cross-project transaction ledger
- `GCF/月度预测/月度预测.html` — rolling monthly forecast matrix (T to T+5)
- `GCF/应收账款/应收账款.html` — AR ledger
- `GCF/应付账款/应付账款.html` — AP ledger

For full session-by-session history and detailed design rationale, see `cashflow-prototype/progress.md`
(chronological log) and `cashflow-prototype/findings.md` (data model + business rules reference).

## Tech stack

Vue 3 + Element Plus + ECharts, all via CDN `<script>` tags. No build step, no bundler, no backend —
every page is a standalone HTML file with its own inline `PROJECTS`/`CASH_ENTRIES` (and, on 3 pages,
Payment Plan) mock data arrays. No JS/Python runtime is available in this environment for running the
app or tests — verify logic by careful manual/static tracing, and recommend the user open pages in a
real browser to confirm interactively.

## Data model

- **Project**: `id` (internal FK, e.g. `PROJ-001`) vs `projectNo` (user-facing code, e.g. `GPMS260015`).
  **Always display/refer to projects by `projectNo` — `PROJ-XXX` is an internal-only key, never shown in
  the UI.**
- **CashEntry**: belongs to exactly one project (`projectId`); `type` (invoice/receipt/expense) ×
  `direction` (in/out) × `status` (confirmed/pending/overdue). All aggregates derive from filtering/
  reducing the shared `CASH_ENTRIES` array — nothing is pre-aggregated/stored.
- **Payment Plan** (added this session, in `index.html`, `GCF/项目详情/项目详情.html`,
  `GCF/月度预测/月度预测.html` only — NOT `收支明细.html` or `项目列表.html`):
  - `VendorPaymentCommitment` (`VPC-xxx`): one vendor "deal" on a project, `totalExpectedCny`. A project
    can have **zero, one, or many** commitments — dashboard/portfolio figures must sum across all of a
    project's commitments, never read a single one.
  - `PaymentPlanPhase` (`PPP-xxx`): planned installment against a commitment, `status`
    planned/replanned/completed/cancelled. Only `planned`/`replanned` phases count as future outflow
    anywhere (`ppIsActivePhase`).
  - Actual payments are plain `CashEntry` rows tagged with an optional `commitmentId` — the **only**
    link between actual payments and a commitment; never matched by vendor name.
  - Full business rules (the 4 `ppReviewReasons` triggers, monthly-forecast tiering, etc.) are in
    `findings.md`.

## Critical gotchas learned this session

1. **`PP_SCHEMA_VERSION` must be bumped whenever the `DEFAULT_VENDOR_PAYMENT_COMMITMENTS` /
   `DEFAULT_PAYMENT_PLAN_PHASES` seed arrays change.** `loadPaymentPlanState()` prefers `localStorage`
   over the seed defaults whenever the cached schema version matches — so editing seed data in the file
   silently does nothing for any browser that already loaded the page. Forgetting this bump was the
   cause of a real bug (dashboard looked "mostly empty" despite seed data covering all 10 projects).
2. **The Payment Plan helper block must stay byte-identical** across `index.html`, `GCF/项目详情/项目详情.html`,
   `GCF/月度预测/月度预测.html` (constants, `pp*` helper functions, load/save/reset functions, seed data).
3. **`GCF/项目列表/项目列表.html` is out of scope for the Payment Plan feature** — it keeps its own
   separate `CASH_ENTRIES` copy with no `commitmentId` field, and shows AR/回款 health, not vendor
   payment plans. Don't add Payment Plan code there.
4. Serve pages over `http://` (not `file://`) when testing `localStorage` sharing across pages —
   `file://` origins are often partitioned per-page by modern browsers.

## Current status

Payment Plan feature is implemented end-to-end (commitment/phase CRUD, review-reason alerts, dashboard
付款计划 column + 制定计划/付款计划 action button, monthly forecast tiering) and seeded across all 10
projects — including 3 projects with a second commitment (multi-commitment aggregation) and 1 project
(`GPMS260031` / `PROJ-004`) deliberately left with zero commitments to demo the empty-state/CTA. The
`PP_SCHEMA_VERSION` cache-invalidation bug (see gotcha #1 above) is fixed as of version `2`. Ready for
the user to demo and review; **awaiting explicit user approval before anything gets committed.**

Full history: `cashflow-prototype/progress.md`. Data model + business rules: `cashflow-prototype/findings.md`.
Original phased plan (now including the Payment Plan milestone): `cashflow-prototype/task_plan.md`.


## Standing constraints

- **Do not `git commit` feature work without explicit user approval** — the user reviews changes first.
- Discuss significant UI/UX changes before implementing when the user asks to "discuss before making
  changes."
