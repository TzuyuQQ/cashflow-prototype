# findings.md — Project-Based Cash Flow Tool

## Architecture Decision
**Pivot (2026-06-30):** Moved from a global cash view to a project-scoped model.
Every CashEntry belongs to exactly one Project. All aggregate views (dashboard, AR, AP, forecast)
derive their numbers from filtering/reducing the shared `CASH_ENTRIES` array.

---

## Data Model

### Project
```js
{
  id:            'PROJ-001',          // primary key; used in ?id= URL param
  projectNo:     'GPMS260015',        // display code matching ERP/GPM format
  name:          'P-希音-CN-肇庆-电商-Ⅳ期',
  client:        '希音供应链（中国）有限公司',
  region:        'CN',                // CN | EU | NA | APAC | ME
  pm:            '张三',
  status:        'active',            // active | at-risk | on-hold | completed
  contractValue: 12000000,
  currency:      'CNY',
  startDate:     '2026-03-01',
  endDate:       '2026-12-31',
}
```

### CashEntry
```js
{
  id:          'CE-001',
  projectId:   'PROJ-001',
  date:        '2026-06-28',          // ISO date — when it happened / was issued
  type:        'invoice',             // invoice | receipt | expense
  subtype:     '进度款',
  direction:   'in',                  // in (收) | out (付)
  currency:    'CNY',
  amount:      2400000,
  amountCny:   2400000,               // pre-converted; use for all aggregations
  counterpart: '希音供应链（中国）有限公司',
  dueDate:     '2026-07-15',          // null for already-confirmed items
  status:      'confirmed',           // confirmed | pending | overdue
  notes:       '',
}
```

### ProjectSummary (computed, not stored)
Derived at render time by reducing CASH_ENTRIES for a given projectId:
```js
function summarize(projectId, entries) {
  const e = entries.filter(x => x.projectId === projectId);
  return {
    totalInvoiced:   sum(e, type='invoice',  direction='out'),  // billed to client
    totalCollected:  sum(e, type='receipt',  direction='in'),   // cash received
    totalExpenses:   sum(e, type='expense',  direction='out'),  // costs paid
    netCash:         totalCollected - totalExpenses,
    uncollected:     totalInvoiced  - totalCollected,
    overdueAR:       e.filter(x => x.status==='overdue' && x.direction==='in'),
    upcomingEvents:  e.filter(x => x.status==='pending' && daysUntil(x.dueDate) <= 30),
  };
}
```

---

## Enum Values

| Field | Values |
|---|---|
| Project.status | `active` 进行中 / `at-risk` 风险 / `on-hold` 暂停 / `completed` 已完成 |
| CashEntry.type | `invoice` 开票 / `receipt` 回款 / `expense` 支出 |
| CashEntry.subtype | 合同首款 / 进度款 / 验收款 / 尾款 / 采购付款 / 人工费 / 运营成本 / 运费 / 其他 |
| CashEntry.status | `confirmed` 已确认 / `pending` 待收/待付 / `overdue` 逾期 |
| CashEntry.direction | `in` 收入 / `out` 支出 |

---

## Health / Status Badge Rules

| Status | Badge color | When |
|---|---|---|
| 进行中 | blue | default active project |
| 风险 | red | netCash < 0 OR overdueAR.length > 0 |
| 暂停 | gray | manually flagged |
| 已完成 | green | endDate passed, all AR collected |

Net cash color rule (table cells):
- positive: `#52c41a` green
- negative: `#f56c6c` red
- zero: `#909399` gray

---

## Mock Data Set (10 projects, 90 cash entries)

### Projects
| ID | 项目编号 | Name | Client | Status | Contract (¥) |
|---|---|---|---|---|---|
| PROJ-001 | GPMS260015 | P-希音-CN-肇庆-电商-Ⅳ期 | 希音供应链（中国）有限公司 | active | 12,000,000 |
| PROJ-002 | GPMS260062 | P-UNILEVER-AR-仓库-Ⅰ期 | Unilever Argentina S.A. | at-risk | 8,500,000 |
| PROJ-003 | GPMS250445 | P-US ELOGISTICS-CA-仓库 | US Elogistics Inc. | active | 5,600,000 |
| PROJ-004 | GPMS260031 | P-CEVA-DE-物流中心-Ⅱ期 | CEVA Logistics GmbH | on-hold | 9,200,000 |
| PROJ-005 | GPMS250388 | P-LAZADA-TH-曼谷-仓库-Ⅲ期 | Lazada Thailand Co., Ltd | completed | 6,800,000 |
| PROJ-006 | GPMS260101 | P-京东-CN-上海-仓储-Ⅰ期 | 京东物流股份有限公司 | active | 9,500,000 |
| PROJ-007 | GPMS260118 | P-DHL-BR-圣保罗-物流中心 | DHL Supply Chain Brasil Ltda | active | 7,200,000 |
| PROJ-008 | GPMS260133 | P-AMAZON-US-西雅图-自动化仓储 | Amazon.com Services LLC | at-risk | 11,000,000 |
| PROJ-009 | GPMS260147 | P-MAERSK-NL-鹿特丹-港口物流 | Maersk Container Industry A/S | active | 8,800,000 |
| PROJ-010 | GPMS260159 | P-GRAB-SG-新加坡-仓库-Ⅰ期 | Grab Holdings Ltd | active | 5,400,000 |

`id` is the internal foreign key used everywhere in code; `项目编号`/`projectNo` (GPMS-format) is the
only one shown in the UI or referenced when talking to the user — see the Payment Plan section's
`use-projectno-not-internal-id` note below.

90 `CashEntry` rows (`CE-001`…`CE-090`) spread across all 10 projects, Jan 2025–Jul 2026, mixing
invoice/receipt/expense and confirmed/pending/overdue statuses.

---

## Shared Mock Data Strategy
Each HTML page defines `const PROJECTS = [...]` and `const CASH_ENTRIES = [...]` inline.
All pages use the same identical arrays — copy-pasted across files for the prototype.
(A real implementation would load from an API.)

---

## Tech Stack
- **Vue 3** global CDN: `https://unpkg.com/vue@3/dist/vue.global.js`
- **Element Plus** CDN: `https://unpkg.com/element-plus/dist/index.full.js` + CSS
- **ECharts** CDN: `https://unpkg.com/echarts/dist/echarts.min.js` (all pages with charts)
- No build step — all pages are standalone HTML files

---

## Layout Reuse (unchanged from Session 1)
- `.top-nav`, `.sidebar`, `.main-wrapper`, `.breadcrumb-bar`, `.content-area`, `.card` — verbatim from template-procurement
- Colors: primary `#1677ff`, orange `#ff7a00`, bg `#f0f2f5`
- Status dots: `.dot-green/.dot-yellow/.dot-red/.dot-gray`
- El-table header style: `{ background: '#f5f7fa', color: '#606266', fontWeight: '600', fontSize: '13px' }`

---

## Payment Plan Feature (2026-07 session — Vendor Payment Commitment / Payment Plan Phases)

### Problem being fixed
All "upcoming payment" forecasting previously read `CashEntry.dueDate` / PO / invoice due dates for
expense rows. Those dates represent contractual or ERP-side due dates, not what Finance actually intends
to pay and when — so 现金流总览's 30-day/7-day panels and 月度预测's forecast columns overstated or
mis-timed vendor outflow. The fix introduces an explicit PM-owned payment plan as the single source of
truth for *future* outflow, decoupled from *actual* confirmed payments.

### New data model
```js
// VendorPaymentCommitment — one per vendor "deal" on a project (the total amount owed)
{
  id:               'VPC-001',
  projectId:        'PROJ-001',
  vendor:           'TTT工程安装（上海）有限公司',
  totalExpectedCny: 1000000,
  currency:         'CNY',
  notes:            '仓储自动化设备安装工程款，分期支付',
  createdAt:        '2026-06-01',
}

// PaymentPlanPhase — one or more planned installments against a commitment
{
  id:               'PPP-001',
  commitmentId:     'VPC-001',          // ← sole link back to the commitment
  projectId:        'PROJ-001',         // denormalized for cheap scope filtering
  vendor:           'TTT工程安装（上海）有限公司', // denormalized for display
  phaseName:        '首期款（20%）',
  plannedDate:      '2026-08-01',
  plannedAmountCny: 200000,
  plannedPct:       20,
  status:           'planned',          // planned | replanned | completed | cancelled
  notes:            '',
  updatedAt:        '2026-06-01',
}
```
`CashEntry` (`CASH_ENTRIES`/`ALL_ENTRIES`, all 4 files) gained one new optional field:
`commitmentId` — set only on confirmed expense rows that represent an actual payment against a
commitment (worked example: `CE-090`). It is the *only* linkage between actual payments and a
commitment; matching by vendor name/counterpart text is explicitly never used for money math.

### Business rules encoded in shared helpers (identical in `index.html`, `月度预测.html`, `项目详情.html`)
- `ppIsActivePhase(phase)` → `status === 'planned' || 'replanned'`. Only active phases count as future
  predicted outflow anywhere in the app (dashboard KPIs, alerts, monthly chart, forecast matrix).
  `completed`/`cancelled` phases are excluded from all forward-looking totals.
- `ppActualPaidCny(commitmentId, cashEntries)` sums confirmed `type='expense',direction='out'` entries
  whose `commitmentId` matches — never vendor-name matching.
  `ppRemainingUnpaidCny`, `ppProgressPct` are derived from it.
- `ppReviewReasons(commitment, phases, cashEntries, todayS)` — the only place "needs review" is
  decided; it is never a stored/manually-set field. Reasons (kept verbatim in English per the
  finalized spec — see Assumptions below), with `PP_TOLERANCE_CNY = 1` (¥1) tolerance on the two
  amount comparisons:
  1. `futureTotal - remaining > 1` → "Future planned amount exceeds remaining unpaid amount"
  2. `remaining - futureTotal > 1` → "Future planned amount is below remaining unpaid amount"
  3. any active phase with `plannedDate < today` → "A planned payment date has passed"
  4. latest confirmed actual payment date > latest phase `updatedAt` → "Actual payment occurred after
     the plan was last updated"
- `ppFuturePlannedOutflowInRange(phases, projectIds, fromS, toS)` — used by the dashboard's 30-day KPI
  and 月度预测's T+3 KPI; sums active-phase `plannedAmountCny` whose `plannedDate` falls in the range.
- Phase CRUD (`savePhase`/`deletePhase` in 项目详情.html) never reads `CASH_ENTRIES` and never
  auto-changes `status` — "completed" is only ever written by the PM via the phase form's status field.

### Persistence
`localStorage` keys (identical across all 3 pages that read/write them):
`gcf_pp_schema_version` / `gcf_pp_commitments` / `gcf_pp_phases`, `PP_SCHEMA_VERSION = 2` (bumped from
`1` after the seed data below was expanded — see "Stale-cache gotcha").
`loadPaymentPlanState()` seeds from `DEFAULT_VENDOR_PAYMENT_COMMITMENTS`/`DEFAULT_PAYMENT_PLAN_PHASES`
on first load or schema mismatch; `resetPaymentPlanState()` (wired to 项目详情.html's "重置演示数据"
button) clears the three keys and re-seeds. `收支明细.html` intentionally does not read/write this
state — it only carries the `commitmentId` data field for realism, per the instruction to leave its
logic untouched.

### Stale-cache gotcha (discovered 2026-07-14)
`loadPaymentPlanState()` only reseeds from `DEFAULT_*` when the cached `gcf_pp_schema_version` in
`localStorage` does *not* match `PP_SCHEMA_VERSION`. This means editing the `DEFAULT_*` seed arrays in
the file has **no visible effect** in any browser that already loaded the page under the old version —
it keeps serving the stale cached commitments/phases. This actually happened: seed data was expanded
from 1 commitment to 10, then to 13 with multi-commitment cases, but the dashboard still looked "mostly
empty" until `PP_SCHEMA_VERSION` was bumped `1 → 2` to force every browser to discard its cache and
reseed. **Any future edit to the seed arrays must bump `PP_SCHEMA_VERSION` again.**

### Seed data coverage (as of the `PP_SCHEMA_VERSION = 2` seed)
All 10 projects have Payment Plan data except one, deliberately:
- **9 projects** have exactly one `VendorPaymentCommitment` (`VPC-002`…`VPC-010`, matching `PROJ-002`…
  `PROJ-010`, skipping `PROJ-001`'s original slot — see below) with phases spanning all 4 statuses.
- **`PROJ-001` (GPMS260015)** has two commitments — `VPC-001` (¥1,000,000, the original worked example,
  needs-review because future planned exceeds remaining) plus `VPC-011` (¥600,000, added for realism,
  review-clean) — to exercise multi-commitment aggregation.
- **`PROJ-006` (GPMS260101)** and **`PROJ-010` (GPMS260159)** likewise each gained a second, review-clean
  commitment (`VPC-012` ¥450,000 and `VPC-013` ¥380,000 respectively) for the same reason.
- **`PROJ-004` (GPMS260031)** has its commitment (`VPC-004`, and phases `PPP-009`/`PPP-010`) removed
  entirely, and its one tagged `CashEntry` (`CE-026`) untagged — this project has **zero** commitments
  on purpose, to demo the "未设置" / "制定计划" empty-state CTA on the dashboard. This was an explicit,
  user-approved design choice (not a data gap) — do not "fix" it by adding a commitment without asking.

Dashboard aggregation (`paymentPlanFor(projectId)` in `index.html`) sums `totalExpectedCny`/
`actualPaidCny` across *every* commitment belonging to a project — confirmed hand-traced results:
PROJ-001 → ¥1,600,000 total / ¥100,000 paid / ¥1,500,000 remaining / 6.3% / ⚠ needs review; PROJ-006 →
¥2,450,000 / ¥1,500,000 / ¥950,000 / 61.2% / clean; PROJ-010 → ¥1,980,000 / ¥980,000 / ¥1,000,000 /
49.5% / clean; PROJ-004 → `hasCommitments:false`.

### Dashboard surfacing (`index.html`, added 2026-07-14)
The 全局总览 dashboard's 项目净现金流排名 table gained a 付款计划 column: a compact progress bar +
⚠ badge with an `<el-tooltip>` showing 预计总额/实付/未付余额 (computed by `paymentPlanFor`, see above),
grey "未设置" text for zero-commitment projects. The 操作 column gained a second button — "付款计划" for
projects with data, "制定计划" (warning-styled) for projects without — that navigates to
`项目详情.html?id={id}#payment-plan`. `项目详情.html`'s Payment Plan card is a plain `<div id="payment-plan">`
(not tab-gated), so the anchor plus a `mounted()` `scrollIntoView` when `location.hash === '#payment-plan'`
is enough to land the PM directly on it.

### Monthly Forecast (月度预测.html) tiering
New `monthTierFor(monthKey, todayS)` classifies each matrix column as `past` / `current` / `future`
by comparing `monthKey` to `todayS.slice(0,7)`. New `outflowBreakdownFor(monthKey, projectId, viewMode)`
implements the rule directly:
- `past` → confirmed actual only (`outflowForecast` forced to 0).
- `current` → confirmed actual filtered to `date <= today`, plus active phases filtered to
  `plannedDate > today`.
- `future` → active phases for the whole month; confirmed-actual naturally stays 0 in the mock data
  (a "confirmed" entry dated in the future is not a realistic state).

### Worked example (used to hand-verify the logic; today = 2026-07-14)
`VPC-001` / TTT工程安装, `totalExpectedCny = 1,000,000`, phases `PPP-001/002/003` = ¥200k (2026-08-01) /
¥200k (2026-09-01) / ¥600k (2026-10-01), all `status:'planned'`, `updatedAt:'2026-06-01'`. One actual
payment `CE-090` = ¥100,000 on 2026-07-08, `commitmentId:'VPC-001'`.
Hand-traced results: `actualPaid=100,000`, `remaining=900,000`, `progress=10%`, `futureTotal=1,000,000`
(all 3 phases are still in the future) → triggers reasons #1 (1,000,000 − 900,000 = 100,000 > ¥1) and
#4 (2026-07-08 > 2026-06-01); reasons #2/#3 do not fire. Dashboard 30-day KPI (window 2026-07-14 →
2026-08-13) picks up only `PPP-001` (¥200,000); 7-day alert panel picks up none (earliest phase is
Aug 1, outside the 2026-07-14→2026-07-21 window).

### Assumptions / design decisions made without stopping to ask
1. **Review reasons kept as literal English strings** (matching the spec text exactly) rather than
   translated to Chinese, even though the rest of each page's UI is in Chinese. Applied identically in
   all 3 places `ppReviewReasons` is rendered. If Chinese labels are wanted, this is a single find/replace
   per reason string, done consistently everywhere since the strings are computed, not stored.
2. **`月度预测.html`'s `rowTotal`/`grandTotal` refactored to sum from `cellData`/`colTotal`** (instead of
   re-deriving independently, as the original code did) so per-row/column totals can never mathematically
   disagree with the tiered/phase-sourced cell figures. Mirrors the pre-existing pattern where
   `runningBalance` already summed via `this.colTotal()`.
3. **`colTotal()`'s pre-existing quirk of ignoring `viewMode`** (always effectively "all") was deliberately
   preserved, not "fixed" — out of scope for this feature.
4. **Shared date helper naming standardized to `isoDate(d)` / `todayStr()`** (todayStr calling isoDate)
   across all 3 files that carry the Payment Plan block, to avoid an initial shadowing collision with the
   pre-existing `daysOverdue(dateStr)`/`daysFromNow(dateStr)` parameter names in `index.html`.
5. **No JS/Python runtime was available in this sandboxed environment**, so all calculation logic above
   was verified by careful manual/hand tracing and static code review rather than executing tests. The
   user should open the pages in a browser to confirm interactively, especially the localStorage
   read/write/reset cycle across pages (which additionally requires serving the files over `http://`
   rather than `file://`, since browsers partition `localStorage` per-origin and `file://` origins are
   frequently treated as opaque/unique per page by modern browsers).
