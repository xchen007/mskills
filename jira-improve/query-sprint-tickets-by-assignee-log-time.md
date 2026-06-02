# Query Sprint Tickets by Assignee with Total Log Time（探索记录）

> Goal: Get all tickets in one sprint, group by assignee, and show each person’s total logged time.
>
> **快速分组查询（不含 log time）见 skill：`jira-sprint-query`（REST API，~1.2s）**
>
> 本文档记录通过 `jira` CLI 实现含 log time 统计的完整工作流。

## Known Issues with jira CLI Approach

- **`jira issue list` 不支持 `ORDER BY`**：JQL 中加入 `ORDER BY` 会报 HTTP 400 错误，需改用 REST API 实现排序（见 `jira-sprint-query` skill）。
- **`jira issue view` 不支持 `--no-cache` flag**：该 flag 仅在 `jira issue list` 中可用。
- **串行查询慢**：每个 ticket 单独调用 `jira issue view`，10个 ticket 约需 10s+；REST API 批量查询约 1.2s。

---

## jira CLI Workflow (含 log time 统计)

### Step 1: Fetch sprint issues (keys only)

Use `jira issue list --raw` only to get sprint issue keys.

```bash
jira issue list -q "sprint = \"SDS-CP-Sprint11-2026\"" --raw > sprint.json
jq -r '.[].key' sprint.json > keys.txt
```

### Step 2: Fetch each issue detail (time fields)

Use `jira issue view --raw` per key to get reliable `timespent`/`timetracking`.

```bash
while IFS= read -r k; do
  jira issue view "$k" --raw \
  | jq -r '[.key,
            (.fields.assignee.displayName // "Unassigned"),
            (.fields.summary // ""),
            (.fields.status.name // ""),
            ((.fields.timespent // 0)|tostring),
            (.fields.timetracking.timeSpent // "0m")] | @tsv'
done < keys.txt > sprint_tickets_with_time.tsv
```

### Step 3: Aggregate by assignee

Count tickets + statuses + total logged seconds.

```bash
awk -F '\t' 'NF>=6{
  ass=$2; status=$4; sec=$5+0;
  total[ass]+=sec; count[ass]++; sc[ass,status]++
}
END{
  for (a in count){
    printf "%s\t%d\tClosed:%d\tResolved:%d\tIn Progress:%d\tOpen:%d\t%d\n",
      a, count[a], sc[a,"Closed"]+0, sc[a,"Resolved"]+0, sc[a,"In Progress"]+0, sc[a,"Open"]+0, total[a]
  }
}' sprint_tickets_with_time.tsv
```

---

## Common Pitfalls to Skip

1. **Do not rely on `jira issue list --raw` for log time fields**
   It often lacks `timespent`/`timetracking` in returned fields.

2. **Do not use `--no-cache` with `jira issue view`**
   `jira issue view` does not support that flag.

3. **Do not batch all keys into one command argument**
   Newlines can break URL parsing. Always process keys line-by-line (`while read`).

4. **Do not aggregate before filtering malformed lines**
   Keep `NF>=6` and valid key checks to avoid bad totals.

5. **Do not mix units when summing**
   Sum with `timespent` (seconds), then convert to hours/days for display.

---

## Why This Pattern Is Stable

- `list --raw` is fast and good for key discovery.
- `view --raw` is reliable for per-issue detail.
- `jq -> TSV -> awk/python` is simple to debug and easy to reuse.
