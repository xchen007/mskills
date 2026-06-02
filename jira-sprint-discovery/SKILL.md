---
name: jira-sprint-discovery
description: Use when you know a sprint name prefix (e.g. "SDS-CP-Sprint") but not the exact sprint name or ID, and need to discover the current ACTIVE sprint or list all historical sprints for a team
---

# Jira Sprint Discovery

## Overview

Discover Sprint names and IDs by scanning project issues — no Board ID required. Sprint data lives in `customfield_12501` on each issue; querying recent issues extracts all associated Sprints.

**Pre-requisites:** `$JIRA_API_TOKEN` and `$JIRA_API_BASE_URL` must be set.

## Query: Find Current ACTIVE Sprint (~2.5s)

```bash
PROJECT=SDSTOR
SPRINT_PREFIX="SDS-CP-Sprint"
TIME_WINDOW="-15d"

curl -s -H "Authorization: Bearer $JIRA_API_TOKEN" \
  "$JIRA_API_BASE_URL/rest/api/2/search" \
  -G --data-urlencode "jql=project = ${PROJECT} AND updated >= ${TIME_WINDOW} ORDER BY updated DESC" \
  --data-urlencode "maxResults=200" \
  --data-urlencode "fields=customfield_12501" | \
  SPRINT_PREFIX="$SPRINT_PREFIX" python3 << 'EOF'
import sys, json, re, os
prefix = os.environ.get('SPRINT_PREFIX', 'SDS-CP-Sprint')
seen = {}
for i in json.load(sys.stdin).get('issues', []):
    for s in (i['fields'].get('customfield_12501') or []):
        raw = str(s)
        m   = re.search(r'name=([^,\]]+)', raw)
        st  = re.search(r'state=([^,\]]+)', raw)
        sid = re.search(r'id=(\d+)', raw)
        sd  = re.search(r'startDate=([^,\]]+)', raw)
        ed  = re.search(r'endDate=([^,\]]+)', raw)
        if m and m.group(1).startswith(prefix) and st and st.group(1) == 'ACTIVE' and m.group(1) not in seen:
            seen[m.group(1)] = {
                'id':    sid.group(1) if sid else '?',
                'start': sd.group(1)[:10] if sd and sd.group(1) not in ('<null>','null') else '-',
                'end':   ed.group(1)[:10] if ed and ed.group(1) not in ('<null>','null') else '-',
            }
for name, v in sorted(seen.items()):
    print(f"{name}  ID:{v['id']}  {v['start']} -> {v['end']}")
EOF
```

## Query: List All Historical Sprints (~3.5s)

Change `TIME_WINDOW="-365d"` and `maxResults=500`, remove the `state == 'ACTIVE'` filter:

```bash
PROJECT=SDSTOR
SPRINT_PREFIX="SDS-CP-Sprint"
TIME_WINDOW="-365d"

curl -s -H "Authorization: Bearer $JIRA_API_TOKEN" \
  "$JIRA_API_BASE_URL/rest/api/2/search" \
  -G --data-urlencode "jql=project = ${PROJECT} AND updated >= ${TIME_WINDOW} ORDER BY updated DESC" \
  --data-urlencode "maxResults=500" \
  --data-urlencode "fields=customfield_12501" | \
  SPRINT_PREFIX="$SPRINT_PREFIX" python3 << 'EOF'
import sys, json, re, os
prefix = os.environ.get('SPRINT_PREFIX', 'SDS-CP-Sprint')
seen = {}
for i in json.load(sys.stdin).get('issues', []):
    for s in (i['fields'].get('customfield_12501') or []):
        raw = str(s)
        m   = re.search(r'name=([^,\]]+)', raw)
        st  = re.search(r'state=([^,\]]+)', raw)
        sid = re.search(r'id=(\d+)', raw)
        sd  = re.search(r'startDate=([^,\]]+)', raw)
        ed  = re.search(r'endDate=([^,\]]+)', raw)
        if m and m.group(1).startswith(prefix) and m.group(1) not in seen:
            seen[m.group(1)] = {
                'id':    sid.group(1) if sid else '?',
                'state': st.group(1)  if st  else '?',
                'start': sd.group(1)[:10] if sd and sd.group(1) not in ('<null>','null') else '-',
                'end':   ed.group(1)[:10] if ed and ed.group(1) not in ('<null>','null') else '-',
            }
print(f"{'Sprint Name':<35} {'ID':<8} {'State':<8} {'Start':<12} End")
print('-'*75)
for name, v in sorted(seen.items()):
    marker = ' ◀ ACTIVE' if v['state'] == 'ACTIVE' else (' ◀ FUTURE' if v['state'] == 'FUTURE' else '')
    print(f"{name:<35} {v['id']:<8} {v['state']:<8} {v['start']:<12} {v['end']}{marker}")
print(f'\n{len(seen)} sprints found')
EOF
```

## Time Window Reference

| Window | Latency | Use case |
|--------|---------|----------|
| `-15d` | ~2.5s | Find current ACTIVE Sprint |
| `-30d` | ~2.9s | Current + most recently closed |
| `-365d` | ~3.5s | Full year history |

> **Caveat:** If all issues in a Sprint had zero updates within the window, that Sprint won't appear. (e.g. a Sprint with no active work may be missing from `-15d` results.)

## Known Non-Working Approaches

| Approach | Error |
|----------|-------|
| `sprint = "SDS-CP-Sprint*"` | WAF blocks — "The request is blocked" |
| `sprint ~ "SDS-CP-Sprint"` | "The operator '~' is not supported by the 'sprint' field" |
| Sprint Autocomplete API | 15-result cap; misses the current ACTIVE Sprint |
| `sprint in openSprints()` | Conditionally works if user has Board association; returns empty without it. Cannot extract Sprint ID or date range. |
| Board name filter (`?name=SDS`) | Misses Boards with generic names (e.g. "Control Plane") |
| `python3 -c "...multiline..."` | Agent collapses newlines to literal `\n` → `SyntaxError: unexpected character after line continuation character`. Use heredoc (`python3 << 'EOF'`) instead. |

## Next Step

Once you have the Sprint ID, use **jira-sprint-query** to query tickets in that sprint.
