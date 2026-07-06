# Create Plans — Cash Flow Tool Project Roadmap

Use when starting a new milestone or planning the full project from a brief.

## Cash Flow Tool: Project Structure

### Core Modules to Plan Around
1. **Data Layer** — projects, invoices, expenses, cash entries (schema + DB)
2. **API Layer** — CRUD routes for cash flow data (FastAPI or Next.js API routes)
3. **Dashboard** — charts (inflow vs outflow, burn rate, forecast), KPI cards
4. **Alerts** — negative cash flow warnings, overdue invoices
5. **Export** — CSV/PDF report generation

## How to Generate a Plan

When asked to create a plan for a feature or milestone:
1. Identify which module(s) it touches
2. Break into ≤10 steps, each completable in one session
3. Flag dependencies (e.g. "needs data layer before dashboard")
4. Estimate complexity: Simple / Medium / Complex
5. Write final plan to task_plan.md with acceptance criteria

## Current Prototype Priorities (MVP)
- [ ] Project list with basic cash flow summary per project
- [ ] Dashboard with inflow/outflow bar chart and net cash KPI card
- [ ] Manual entry form for income and expenses
- [ ] Simple forecast (linear trend based on last 30 days)
- [ ] Export to CSV

Do not build beyond MVP scope without updating this file first.