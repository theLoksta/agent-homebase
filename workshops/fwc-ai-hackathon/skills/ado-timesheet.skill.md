---
id: skill.ado-timesheet
kind: skill
version: 1.0.0
applies_to: '**'
name: ado-timesheet
description: Generates a weekly timesheet summary from Azure DevOps commit, PR, and work item activity for one or more accounts. Groups output by work item, oldest-first, with inline PR references.
when_to_use: timesheet, weekly summary, ado activity, work summary, what did I work on, weekly report
user-invocable: true
inputs:
  type: object
  required:
    - week_ending
  properties:
    week_ending:
      type: string
      description: The Friday end-date of the week in question (e.g. "Fri 15 May 2026").
    accounts:
      type: array
      description: Override the default accounts to search. Defaults to {{ado.primary_account}} and {{ado.service_account}}.
outputs:
  - return_tier: 2
agent:
  tools:
    - mcp (ado-remote-mcp)
  agents: []
  model: null
  handoffs: []
---

# ADO Timesheet

> **Environment note:** The Azure DevOps remote MCP server (`{{ado.mcp_url}}`) currently requires VS Code + GitHub Copilot. Claude Code support is not yet available (pending Microsoft Entra OAuth registration). Run this skill in VS Code Agent mode.

You generate a weekly timesheet summary by querying Azure DevOps for commit, PR, and work item activity across the configured accounts and repos.

## Configuration tokens

| Token | Example value | Purpose |
| --- | --- | --- |
| `{{ado.org}}` | `fwcdev` | ADO organisation name |
| `{{ado.mcp_url}}` | `https://mcp.dev.azure.com/fwcdev` | Remote MCP server URL |
| `{{ado.primary_account}}` | `Lachlan.Grant@fwc.gov.au` | Primary work account |
| `{{ado.service_account}}` | `ITEX-LG@fwc.gov.au` | Service/automation account |
| `{{ado.project}}` | `Customer Services Platform` | Main ADO project |
| `{{ado.repos}}` | `JavaScript and Plugins, CustomerPortal` | Comma-separated repos to search |
| `{{ado.phase}}` | `Chambers Onboarding` | Current engagement phase — used in summary opening line |
| `{{ado.week_start_day}}` | `Saturday` | First day of the timesheet week |

---

## Week period

The timesheet week runs **{{ado.week_start_day}} to Friday** (7 days).
e.g. "week ending Fri 15 May" = Sat 9 May → Fri 15 May.

All ADO timestamps are UTC. Convert to local time before attributing activity to a day:
- **AEST (UTC+10):** approx. April–October
- **AEDT (UTC+11):** approx. October–April

---

## Step-by-step

### 1. Commits (both accounts)

Use `repo_search_commits` with `authorEmail` + `fromDate`/`toDate` for each repo in `{{ado.repos}}`.
- Set `includeWorkItems: true` to surface linked work item IDs.
- Run for both `{{ado.primary_account}}` and `{{ado.service_account}}`.

### 2. PRs created (both accounts)

Use `repo_pull_request` action `list` with `createdByUser` for each account, `status: All`.
Filter results where `creationDate` or `closedDate` falls within the week.

### 3. PRs reviewed (both accounts)

Use `repo_pull_request` action `list` with `userIsReviewer` for each account, `status: All`.
Filter results where `closedDate` or `creationDate` falls within the week.

### 4. Work items — recent activity

Use `wit_work_item` action `my` with `type: myactivity` (primary account only — this tool uses the authenticated user).
Returns IDs only — follow up with `get_batch` requesting these fields:
`["System.Id", "System.Title", "System.WorkItemType", "System.State", "System.ChangedDate", "System.ChangedBy", "System.AssignedTo"]`

**Important:** `ChangedBy` reflects the _last_ person to edit, not necessarily the configured user. Always dig deeper.

### 5. Work item revisions (where ChangedBy ≠ configured user)

Use `wit_work_item` action `list_revisions`, then filter with jq:

```bash
jq '[.[] | {rev, date: .fields["System.ChangedDate"], by: .fields["System.ChangedBy"].uniqueName}] | map(select(.date >= "YYYY-MM-DDTHH:00:00Z"))' file.json
```

UTC window for a full local week:
- AEST: Monday 00:00 AEST = Sunday 14:00 UTC
- AEDT: Monday 00:00 AEDT = Sunday 13:00 UTC

### 6. Work item comments (both accounts)

Use `wit_work_item` action `list_comments` for any item in the activity list.
Filter with jq on `createdDate` and `createdBy.uniqueName`.

### 7. Work items assigned to others

Do not skip items where `AssignedTo` isn't the configured user — they may have commented or updated fields on them.

---

## What ADO will NOT capture

- Local dev work with no commits
- Meetings, reading specs, planning
- Work done but not yet committed/commented

---

## Output format

- **Opening line:** `Continuing my {{ado.project}} responsibilities for their ***{{ado.phase}}*** phase:`
- Group by work item (not by day)
- Order: oldest-first based on first activity in the week
- No specific dates — just the week label
- High-level nested bullet points per item
- Work item headings: **`[US-XXXXX]`: Title** (bold, backtick-bracketed ID)
- PR references inline: **`[PR-XXXX]`**
- Assignee note as blockquote under heading: `>_(assigned to ...)_`
- Bold+italic (`***text***`) for key terms/names; italic (`_text_`) for secondary/parenthetical info
- Bulk sprint-planning, triage, or minor field-update items → separate **"Other activity"** section after a `---` rule (user decides whether to include when pasting)
- **Always output twice:** rendered/formatted version first, then raw Markdown in a fenced code block for copy-paste
- **Conversation/chat title:** `Weekly Timesheet Summary [{startDate} to {endDate}]`
  e.g. `Weekly Timesheet Summary [Sat 9 May to Fri 15 May 2026]`

---

## Verification

A reviewer can confirm this skill ran correctly when:

- [ ] Commits checked for both `{{ado.primary_account}}` and `{{ado.service_account}}`
- [ ] PRs checked as both author and reviewer
- [ ] Work item revisions checked for items where `ChangedBy` ≠ configured user
- [ ] Output rendered + raw Markdown both present
- [ ] Chat title follows the `Weekly Timesheet Summary [...]` format
