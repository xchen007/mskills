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

> ⚠️ **Output rule — strictly enforced:**
> - Print the bash output **verbatim as a code block**. The terminal table IS the final answer.
> - **NEVER render a Markdown table after the bash output.** Do not reformat, summarize, or recreate the data in any other format.
> - If you feel the urge to add a table or list after the bash output — **don't**. Just print the raw output.

```bash
# Default: my tickets, sorted by updated desc
# Adjust JQL filter as needed — always re-run the query, never manually filter the previous output
# Examples:
#   Exclude done:   AND status NOT IN (Done, Closed, Resolved)
#   All assignees:  remove "AND assignee = currentUser()"
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

RESET  = '\033[0m';  BOLD   = '\033[1m';  DIM    = '\033[2m'
CYAN   = '\033[96m'; GREEN  = '\033[92m'; YELLOW = '\033[93m'
RED    = '\033[91m'; BLUE   = '\033[94m'; MAGENTA= '\033[95m'

MAX_SUMMARY = 60

def c_key(s):      return CYAN + BOLD + s + RESET
def c_status(s):
    if s in ('Resolved','Done','Closed'):      return GREEN + s + RESET
    if s in ('In Progress','In Review'):       return YELLOW + s + RESET
    if s == 'Blocked':                         return RED + BOLD + s + RESET
    return DIM + s + RESET
def c_pri(s):
    if s in ('P1','Highest','Critical'):       return RED + BOLD + s + RESET
    if s in ('P2','High'):                     return RED + s + RESET
    if s in ('P3','Medium'):                   return YELLOW + s + RESET
    return DIM + s + RESET

def fmt_time(sec):
    if not sec: return '-'
    h, m = divmod(sec // 60, 60)
    return f'{h}h {m}m' if m else f'{h}h'

def fmt_done(ds):
    if not ds: return '-'
    try:
        dt   = datetime.fromisoformat(ds.replace('Z','+00:00'))
        days = (datetime.now(timezone.utc) - dt).days
        if days == 0: return 'today'
        if days == 1: return 'yesterday'
        if days <  7: return f'{days}d ago'
        if days < 14: return 'last week'
        if days < 30: return f'{days//7}w ago'
        return dt.strftime('%Y-%m-%d')
    except: return ds[:10]

def trunc(s, n): return s[:n-1] + '…' if len(s) > n else s
def pad(colored, raw_len, width): return colored + ' ' * max(0, width - raw_len)

issues = json.load(open(os.environ['TMP_FILE'])).get('issues', [])
raw_rows = []
col_rows = []
for i in issues:
    f   = i['fields']
    key = i['key']
    sts = (f.get('status')   or {}).get('name', '?')
    smr = trunc(f.get('summary') or '', MAX_SUMMARY)
    pri = (f.get('priority') or {}).get('name', '-')
    pts = str(f.get('customfield_10016') or '-')
    est = fmt_time(f.get('timeoriginalestimate'))
    log = fmt_time(f.get('timespent'))
    asn = (f.get('assignee') or {}).get('displayName', 'Unassigned')
    don = fmt_done(f.get('resolutiondate'))
    typ = (f.get('issuetype')or {}).get('name', '?')
    prj = (f.get('project')  or {}).get('name', '?')
    raw_rows.append([key, sts, smr, pri, pts, est, log, asn, don, typ, prj])
    col_rows.append([c_key(key), c_status(sts), smr, c_pri(pri), pts, est, log, asn, don, typ, prj])

hdrs   = ['KEY','STATUS','SUMMARY','PRI','PTS','EST.','LOG','ASSIGNEE','DONE','TYPE','PROJECT']
widths = [max(len(h), max((len(r[i]) for r in raw_rows), default=0)) for i,h in enumerate(hdrs)]
sep    = '  '.join('-'*w for w in widths)
hdr_ln = '  '.join(BOLD + h.ljust(widths[i]) + RESET for i,h in enumerate(hdrs))
print(hdr_ln)
print(DIM + sep + RESET)
for ri, row in enumerate(col_rows):
    cells = [pad(row[i], len(raw_rows[ri][i]), widths[i]) for i in range(len(hdrs))]
    print('  '.join(cells))
print(f'\n{BOLD}Total: {len(issues)} tickets{RESET}')
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
