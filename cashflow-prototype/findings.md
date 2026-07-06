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

## Mock Data Set (5 projects, ~40 cash entries)

### Projects
| ID | Name | Client | Status | Contract (¥) |
|---|---|---|---|---|
| PROJ-001 | P-希音-CN-肇庆-电商-Ⅳ期 | 希音供应链 | active | 12,000,000 |
| PROJ-002 | P-UNILEVER-AR-仓库-Ⅰ期 | UNILEVER Argentina | at-risk | 8,500,000 |
| PROJ-003 | P-US ELOGISTICS-CA-仓库 | US Elogistics Inc. | active | 5,600,000 |
| PROJ-004 | P-CEVA-DE-物流中心-Ⅱ期 | CEVA Logistics GmbH | on-hold | 9,200,000 |
| PROJ-005 | P-LAZADA-TH-曼谷-Ⅲ期 | Lazada Thailand | completed | 6,800,000 |

Each project has ~8 CashEntries spanning Jan–Jul 2026 (mix of invoice, receipt, expense;
some confirmed, some pending, some overdue).

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
