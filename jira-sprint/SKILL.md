---
name: jira-sprint
description: Use when querying Jira sprint tickets — discovering the active sprint by name prefix and/or querying tickets by assignee with sorted table output. Prefer over jira-server MCP for sorted/grouped queries (MCP does not support ORDER BY).
---

# Jira Sprint

## Overview

Two independent steps — run either or both:

- **Step 1** — Discover active Sprint ID from a name prefix (skip if you already know the ID)
- **Step 2** — Query and display tickets in table format (skip if you only need Sprint info)

**Pre-requisites:** `$JIRA_API_TOKEN` and `$JIRA_API_BASE_URL` must be set.

---

## Step 1: Discover Active Sprint ID

> **Skip this step** if you already know the Sprint ID — just set `SPRINT_ID=<id>` manually.

```bash
PROJECT=SDSTOR
SPRINT_PREFIX="SDS-CP-Sprint"

TMP_SPRINT=$(mktemp)
curl -s -H "Authorization: Bearer $JIRA_API_TOKEN" \
  "$JIRA_API_BASE_URL/rest/api/2/search" \
  -G --data-urlencode "jql=project = ${PROJECT} AND updated >= -15d ORDER BY updated DESC" \
  --data-urlencode "maxResults=200" \
  --data-urlencode "fields=customfield_12501" > "$TMP_SPRINT"

SPRINT_ID=$(TMP_FILE="$TMP_SPRINT" SPRINT_PREFIX="$SPRINT_PREFIX" python3 << 'EOF'
import json, re, os
prefix = os.environ.get('SPRINT_PREFIX', 'SDS-CP-Sprint')
seen = {}
for i in json.load(open(os.environ['TMP_FILE'])).get('issues', []):
    for s in (i['fields'].get('customfield_12501') or []):
        raw = str(s)
        m   = re.search(r'name=([^,\]]+)', raw)
        st  = re.search(r'state=([^,\]]+)', raw)
        sid = re.search(r'id=(\d+)', raw)
        if m and m.group(1).startswith(prefix) and st and st.group(1) == 'ACTIVE':
            seen[m.group(1)] = sid.group(1) if sid else '?'
for name, sid in sorted(seen.items()):
    print(sid)
    break  # first ACTIVE sprint only
EOF
)
rm "$TMP_SPRINT"

echo "Sprint ID: $SPRINT_ID"
```

**To list all historical sprints** (not just ACTIVE), replace `-15d` with `-365d`, remove the `state == 'ACTIVE'` filter, and print name + id + state instead of just id.

---

## Step 2: Query Tickets

> **Skip this step** if you only need Sprint info.
> Requires `$SPRINT_ID` from Step 1, or set manually: `SPRINT_ID=125626`

```bash
# Default: my tickets, sorted by updated desc
JQL="sprint = ${SPRINT_ID} AND assignee = currentUser() ORDER BY updated DESC"

TMP_TICKETS=$(mktemp)
curl -s -H "Authorization: Bearer $JIRA_API_TOKEN" \
  "$JIRA_API_BASE_URL/rest/api/2/search" \
  -G --data-urlencode "jql=${JQL}" \
  --data-urlencode "maxResults=100" \
  --data-urlencode "fields=summary,assignee,status,issuetype,priority,timespent,timeoriginalestimate,resolutiondate,project,customfield_10016" > "$TMP_TICKETS"

TMP_FILE="$TMP_TICKETS" python3 << 'EOF'
import json, os
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

issues = json.load(open(os.environ['TMP_FILE'])).get('issues', [])
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
rm "$TMP_TICKETS"
```

---

## Common JQL Filters

| Goal | JQL fragment |
|------|-------------|
| My tickets only | `AND assignee = currentUser()` |
| Exclude done | `AND status NOT IN (Done, Closed, Resolved)` |
| All assignees | remove `AND assignee = ...` |
| Stories only | `AND issuetype = Story` |
| High priority | `AND priority = High` |

---

## Common Mistakes

- **`jira issue list` with ORDER BY → HTTP 400** — always use REST API.
- **Sprint name instead of ID** — `sprint = 125626` (no quotes) is faster; `sprint = 'SDS-CP-Sprint11-2026'` also works.
- **`status != Done` misses Closed/Resolved** — use `status NOT IN (Done, Closed, Resolved)`.
- **`currentUser()` fails** — ensure `$JIRA_API_TOKEN` is not expired.
- **PTS all `-`** — `customfield_10016` varies by instance. Find correct field: add `&expand=names` to the API call and search for "Story Points".
- **`curl ... | python3 << 'EOF'` SyntaxError** — heredoc takes over stdin; `sys.stdin` is empty. Always save curl output to a temp file (`TMP=$(mktemp); curl ... > "$TMP"`) and read via `json.load(open(os.environ['TMP_FILE']))`.

## Known Non-Working Approaches (Step 1)

| Approach | Error |
|----------|-------|
| `sprint = "SDS-CP-Sprint*"` | WAF blocks — "The request is blocked" |
| `sprint ~ "SDS-CP-Sprint"` | `~` operator not supported for sprint field |
| Sprint Autocomplete API | 15-result cap; misses ACTIVE Sprint |
| `sprint in openSprints()` | Conditionally works if user has Board association; cannot extract Sprint ID |
| Board name filter (`?name=SDS`) | Misses Boards with generic names (e.g. "Control Plane") |
