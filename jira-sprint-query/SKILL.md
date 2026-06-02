---
name: jira-sprint-query
description: Use when querying Jira sprint tickets with assignee filter, grouping, or sorted output. Prefer this over jira-server MCP tools — MCP jira-server does not support ORDER BY (HTTP 400) and cannot group by assignee. Use jira-sprint-discovery first if Sprint ID is unknown.
---

# Jira Sprint Query

## Overview

Query tickets in a sprint using the REST API. All queries use Sprint ID (not name) for speed (~1.2s). Use **jira-sprint-discovery** first if you don't know the Sprint ID.

**Pre-requisites:** `$JIRA_API_TOKEN` and `$JIRA_API_BASE_URL` must be set.

> **Do NOT use `jira issue list` for these queries** — its JQL does not support `ORDER BY` and returns HTTP 400.

## Query: My Tickets in a Sprint (grouped by assignee, sorted by updated)

```bash
SPRINT_ID=125626   # get from jira-sprint-discovery

curl -s -H "Authorization: Bearer $JIRA_API_TOKEN" \
  "$JIRA_API_BASE_URL/rest/api/2/search" \
  -G --data-urlencode "jql=sprint = ${SPRINT_ID} AND assignee = currentUser() ORDER BY updated DESC" \
  --data-urlencode "maxResults=100" \
  --data-urlencode "fields=summary,assignee,status,issuetype,priority,timespent,timeoriginalestimate,resolutiondate,project,customfield_10016" | \
  python3 << 'EOF'
import sys, json
from datetime import datetime, timezone

def fmt_time(seconds):
    if not seconds: return '-'
    h, m = divmod(seconds // 60, 60)
    return f'{h}h {m}m' if m else f'{h}h'

def fmt_done(date_str):
    if not date_str: return '-'
    try:
        dt = datetime.fromisoformat(date_str.replace('Z', '+00:00'))
        days = (datetime.now(timezone.utc) - dt).days
        if days == 0: return 'today'
        if days == 1: return 'yesterday'
        if days < 7:  return f'{days}d ago'
        if days < 14: return 'last week'
        if days < 30: return f'{days//7}w ago'
        return dt.strftime('%Y-%m-%d')
    except: return date_str[:10]

data   = json.load(sys.stdin)
issues = data.get('issues', [])
rows   = []
for i in issues:
    f = i['fields']
    rows.append([
        i['key'],
        (f.get('status')    or {}).get('name', '?'),
        (f.get('summary')   or '')[:52],
        (f.get('priority')  or {}).get('name', '-'),
        str(f.get('customfield_10016') or '-'),
        fmt_time(f.get('timeoriginalestimate')),
        fmt_time(f.get('timespent')),
        (f.get('assignee')  or {}).get('displayName', 'Unassigned'),
        fmt_done(f.get('resolutiondate')),
        (f.get('issuetype') or {}).get('name', '?'),
        (f.get('project')   or {}).get('name', '?'),
    ])

hdrs   = ['KEY', 'STATUS', 'SUMMARY', 'PRI', 'PTS', 'EST.', 'LOG', 'ASSIGNEE', 'DONE', 'TYPE', 'PROJECT']
widths = [max(len(h), max((len(r[i]) for r in rows), default=0)) for i, h in enumerate(hdrs)]
fmt    = '  '.join(f'{{:<{w}}}' for w in widths)
sep    = '  '.join('-' * w for w in widths)
print(fmt.format(*hdrs))
print(sep)
for r in rows:
    print(fmt.format(*r))
print(f'\nTotal: {len(issues)} tickets')
EOF
```

## Query: All Tickets in a Sprint (all assignees)

Change the JQL to remove the assignee filter:

```bash
jql=sprint = ${SPRINT_ID} ORDER BY assignee ASC, updated DESC
```

## Common JQL Filters

| Goal | JQL fragment to add |
|------|---------------------|
| My tickets only | `AND assignee = currentUser()` |
| Exclude done | `AND status NOT IN (Done, Closed, Resolved)` |
| Stories only | `AND issuetype = Story` |
| High priority | `AND priority = High` |
| Sort by updated | `ORDER BY updated DESC` |
| Sort by assignee then updated | `ORDER BY assignee ASC, updated DESC` |

## Common Mistakes

- **`jira issue list` with ORDER BY → HTTP 400** — always use REST API for sorted queries.
- **Using Sprint name instead of ID** — `sprint = 125626` (ID, no quotes) is faster and unambiguous; `sprint = 'SDS-CP-Sprint11-2026'` also works but requires exact name.
- **`status != Done` misses Closed/Resolved** — use `status NOT IN (Done, Closed, Resolved)`.
- **Story Points field varies by instance** — `customfield_10016` is common but may differ. If PTS shows `-` for all tickets, find the correct field by adding `&expand=names` to the API call and searching the response for "Story Points".
