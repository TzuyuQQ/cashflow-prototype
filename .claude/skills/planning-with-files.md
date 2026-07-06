# Planning with Files — Cash Flow Tool

Use when starting any complex task involving the dashboard, data pipeline, or API routes.

## On Session Start
Always read task_plan.md and progress.md before doing anything.
If they don't exist, create them now.

## File Responsibilities

### task_plan.md
The master plan. Contains:
- Project goal (one sentence)
- Current milestone (e.g. "MVP Dashboard", "Invoice Import", "Forecast Module")
- Numbered implementation steps with status: [ ] todo, [x] done, [~] in progress
- Acceptance criteria for current milestone

### findings.md
Running notes. Contains:
- Data model decisions (schema for projects, invoices, expenses, cash entries)
- API design choices and why
- Bugs found and their root cause
- External dependencies (libraries, APIs)

### progress.md
Session log. After each step, append:
- Date/time
- What was completed
- What's next
- Any blockers

## On Session End
Always update progress.md with a summary before stopping.
Never leave task_plan.md with stale [~] items — resolve or re-plan.