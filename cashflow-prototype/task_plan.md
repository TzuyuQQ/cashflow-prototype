# task_plan.md — Project-Based Cash Flow Tool

## Project Goal
A project-management cash flow prototype where every dollar is tied to a project.
Users navigate from the global portfolio dashboard → project list → project detail,
with cross-project AR, AP, and monthly forecast views as supporting pages.

## Current Milestone
**Milestone 2 — Project-Based MVP (7 pages)**

## Acceptance Criteria
- All pages open standalone in a browser with no build step
- All pages share the same `PROJECTS` and `CASH_ENTRIES` mock-data arrays (inline in each file or extracted to a linked `mock-data.js`)
- `index.html` global dashboard aggregates across all projects from that shared data
- Project detail page driven by `?id=PROJ-XXX` URL param, falls back to first project
- Monetary values formatted consistently (¥ with commas, 万 shorthand for cards)
- Status dots + health badges consistent across all pages
- Sidebar navigation identical on all pages (active link changes per page)

---

## Data Structures

### Entity 1 — Project
```js
{
  id:            'PROJ-001',          // internal key, used in URL param
  projectNo:     'GPMS260015',        // display ID (matches template)
  name:          'P-希音-CN-肇庆-电商-Ⅳ期',
  client:        '希音供应链（中国）有限公司',
  region:        'CN',                // CN | EU | NA | APAC | ME
  pm:            '张三',
  status:        'active',            // active | at-risk | on-hold | completed
  contractValue: 12000000,            // CNY
  currency:      'CNY',
  startDate:     '2026-03-01',
  endDate:       '2026-12-31',
}
```

### Entity 2 — CashEntry
```js
{
  id:          'CE-001',
  projectId:   'PROJ-001',
  date:        '2026-06-28',
  type:        'invoice',             // invoice | receipt | expense
  subtype:     '进度款',              // see enum table below
  direction:   'in',                  // in | out
  currency:    'CNY',
  amount:      2400000,               // original currency
  amountCny:   2400000,               // CNY equivalent
  counterpart: '希音供应链（中国）有限公司',
  dueDate:     '2026-07-15',          // null for confirmed items
  status:      'confirmed',           // confirmed | pending | overdue
  notes:       '',
}
```

### Entity 3 — ProjectSummary (derived, computed at runtime)
```js
{
  projectId:       'PROJ-001',
  contractValue:   12000000,
  totalInvoiced:   7200000,    // invoices sent to client (direction: out → type: invoice)
  totalCollected:  4800000,    // cash received (type: receipt, direction: in)
  totalExpenses:   3200000,    // costs paid (type: expense, direction: out)
  netCash:         1600000,    // totalCollected - totalExpenses
  uncollected:     2400000,    // totalInvoiced - totalCollected
  overdueAR:       [...],      // CashEntry[] where status=overdue AND direction=in
  upcomingEvents:  [...],      // CashEntry[] where status=pending AND dueDate within 30 days
}
```

### Enum Values
| Field       | Values |
|---|---|
| Project status | `active` (进行中) / `at-risk` (风险) / `on-hold` (暂停) / `completed` (已完成) |
| CashEntry type | `invoice` (开票) / `receipt` (回款) / `expense` (支出) |
| CashEntry subtype | 合同首款 / 进度款 / 验收款 / 尾款 / 采购付款 / 人工费 / 运营成本 / 运费 / 其他 |
| CashEntry status | `confirmed` (已确认) / `pending` (待收/待付) / `overdue` (逾期) |
| CashEntry direction | `in` (收入) / `out` (支出) |

---

## Page Inventory

| # | File path | Menu? | Role |
|---|---|---|---|
| 1 | `index.html` | ✓ 全局总览 | Portfolio dashboard — aggregates across all projects |
| 2 | `GCF/项目列表/项目列表.html` | ✓ 项目列表 | Primary project list with health summary per row |
| 3 | `GCF/项目详情/项目详情.html` | ✗ non-menu | Project drill-down (KPIs + chart + entries); driven by `?id=` |
| 4 | `GCF/收支明细/收支明细.html` | ✓ 收支明细 | All entries across all projects, filterable by project |
| 5 | `GCF/月度预测/月度预测.html` | ✓ 月度预测 | Rolling T→T+5 month net cash per project (row per project) |
| 6 | `GCF/应收账款/应收账款.html` | ✓ 应收账款 | AR ledger across all projects — invoices + receipts |
| 7 | `GCF/应付账款/应付账款.html` | ✓ 应付账款 | AP ledger across all projects — expenses + upcoming payments |

---

## Implementation Steps

### Phase 0 — Already Done
- [x] Folder scaffold (`cashflow-prototype/`)
- [x] `index.html` — global dashboard (KPI cards + ECharts bar chart + recent txns)
  - **Needs update:** reframe KPIs as portfolio aggregates; add Top-5 worst-net table; add alert panel

### Phase 1 — Core Project Pages ✅ DONE
- [x] **Step 1 (update):** `index.html` — portfolio dashboard
  - KPIs: Total Portfolio Net Cash, Total Outstanding AR, Total AP Due, # At-Risk Projects
  - Top-5 worst-net-cash projects table with status badges and 查看详情 links
  - Alert panel: overdue AR items + payments due in ≤7 days
  - Monthly ECharts chart (grouped diverging bars + net line, toggle to net view)

- [x] **Step 2:** `GCF/项目列表/项目列表.html`
  - Filter bar: 状态, 大区, PM
  - Table: 项目编号, 项目名称, 客户, PM, 状态badge, 合同总额, 净现金流 (±color), 回款进度 health bar, 近期事件 tags
  - Row click → `../项目详情/项目详情.html?id={project.id}`

- [x] **Step 3:** `GCF/项目详情/项目详情.html`
  - Reads `?id=` from URL; falls back gracefully to PROJ-001
  - Breadcrumb: 现金流管理 / 项目列表 / {project.name}
  - Project header card with status badge
  - 5 KPI cards: 合同总额, 已开票, 已回款, 总支出, 净现金流
  - ECharts: monthly inflow/outflow bars + net line; toggle to net-only bar view
  - el-tabs: 全部 | 开票记录 | 回款记录 | 支出记录
  - Table: date, type badge, subtype, direction, amount CNY, original currency, counterpart, dueDate, status

### Phase 2 — Supporting Views ✅ DONE
- [x] **Step 4:** `GCF/收支明细/收支明细.html`
  - Filter: 项目, 类型, 方向, 日期范围, 状态 — all combinable
  - Summary strip: 条数, 合计收入, 合计支出, 净现金流
  - Cross-project table with project link, pagination, sort by date

- [x] **Step 5:** `GCF/月度预测/月度预测.html`
  - T to T+5 matrix: Jun–Nov 2026, one row per project + 合计 row
  - Each cell shows ▲ inflow, ▼ outflow, net with color coding; 预 tag for pending items
  - View mode toggle: 全部 / 已确认 / 待确认
  - KPI cards: T+3 expected inflow/outflow/net, current month confirmed net

- [x] **Step 6:** `GCF/应收账款/应收账款.html`
  - KPI cards: 已开票, 已回款, 逾期未收, 待收30天
  - Aging labels (逾期N天 / 还有N天 / 已收) with color badges
  - Filter: 项目, 类型 (发票/回款), 状态

- [x] **Step 7:** `GCF/应付账款/应付账款.html`
  - KPI cards: 已支出, 待付, 7天内到期
  - Due date urgency badges (到期提示)
  - 确认付款 action button on pending rows
  - Filter: 项目, 支出类型, 状态

### Phase 3 — Polish
- [ ] Update sidebar HTML on all 7 pages so nav links are consistent
- [ ] Confirm `?id=` routing works on project detail across browsers
- [ ] Verify monetary formatting is consistent (¥X,XXX,XXX and ¥X,XXX万 cards)
