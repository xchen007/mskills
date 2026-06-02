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
  --data-urlencode "fields=summary,assignee,status,updated,issuetype,priority" | \
  python3 << 'EOF'
import sys, json
from collections import defaultdict
data   = json.load(sys.stdin)
issues = data.get('issues', [])
groups = defaultdict(list)
for i in issues:
    f        = i['fields']
    assignee = (f.get('assignee') or {}).get('displayName', 'Unassigned')
    groups[assignee].append({
        'key':     i['key'],
        'summary': f.get('summary','')[:60],
        'status':  (f.get('status')    or {}).get('name','?'),
        'type':    (f.get('issuetype') or {}).get('name','?'),
        'updated': f.get('updated','')[:10],
    })
for assignee, tickets in groups.items():
    print(f'── {assignee} ({len(tickets)}) ──')
    print(f"  {'KEY':<15} {'STATUS':<14} {'TYPE':<18} {'UPDATED':<12} SUMMARY")
    print('  ' + '-'*95)
    for t in tickets:
        print(f"  {t['key']:<15} {t['status']:<14} {t['type']:<18} {t['updated']:<12} {t['summary']}")
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
- **`currentUser()` requires valid auth token** — ensure `$JIRA_API_TOKEN` is not expired.
