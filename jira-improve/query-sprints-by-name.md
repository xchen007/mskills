# 按名称前缀查询 Sprint（探索记录）

> 场景：查询所有以 `SDS-CP-Sprint` 开头的 Sprint，无需维护 Board ID 列表
>
> **可执行命令见 skill：`jira-sprint-discovery`**

---

## 推荐方案：从项目 Issue 的 Sprint 字段反查

**原理**：Sprint 数据存储在 issue 的 `customfield_12501` 字段中（序列化字符串格式，需正则提取），通过查询项目近期 issue 即可提取所有关联 Sprint，无需知道 Board ID。

### 时间窗口选择

| 时间窗口 | 耗时 | 适用场景 |
|---------|------|---------|
| `-15d`  | ~2.5s | 快速找当前 ACTIVE Sprint |
| `-30d`  | ~2.9s | 当前 + 最近结束的 Sprint |
| `-365d` | ~3.5s | 全量历史（一年内） |

> **注意**：时间窗口仅影响 issue 更新时间过滤。若某 Sprint 内所有 issue 在时间窗口外无更新，则该 Sprint 不会出现（如 Sprint06-2026 缺失即因此）。

---

## 实际查询结果（2026-06-02，时间窗口 -365d）

| Sprint 名称 | ID | 状态 | 开始日期 | 结束日期 |
|---|---|---|---|---|
| SDS-CP-Sprint17-2025 | 113378 | CLOSED | 2025-08-25 | 2025-09-08 |
| SDS-CP-Sprint18-2025 | 113379 | CLOSED | 2025-09-08 | 2025-09-22 |
| SDS-CP-Sprint19-2025 | 113380 | CLOSED | 2025-09-22 | 2025-10-06 |
| SDS-CP-Sprint20-2025 | 113381 | CLOSED | 2025-10-06 | 2025-10-20 |
| SDS-CP-Sprint21-2025 | 113382 | CLOSED | 2025-10-20 | 2025-11-03 |
| SDS-CP-Sprint22-2025 | 113383 | CLOSED | 2025-11-03 | 2025-11-17 |
| SDS-CP-Sprint23-2025 | 113384 | CLOSED | 2025-11-17 | 2025-12-01 |
| SDS-CP-Sprint24-2025 | 113385 | CLOSED | 2025-12-01 | 2025-12-15 |
| SDS-CP-Sprint25-2025 | 113386 | CLOSED | 2025-12-15 | 2025-12-29 |
| SDS-CP-Sprint01-2026 | 120526 | CLOSED | 2026-01-05 | 2026-01-19 |
| SDS-CP-Sprint02-2026 | 120527 | CLOSED | 2026-01-19 | 2026-02-02 |
| SDS-CP-Sprint03-2026 | 120528 | CLOSED | 2026-02-02 | 2026-02-16 |
| SDS-CP-Sprint04-2026 | 122284 | CLOSED | 2026-02-16 | 2026-03-02 |
| SDS-CP-Sprint05-2026 | 122285 | CLOSED | 2026-03-02 | 2026-03-16 |
| SDS-CP-Sprint07-2026 | 122854 | CLOSED | 2026-03-30 | 2026-04-13 |
| SDS-CP-Sprint08-2026 | 124304 | CLOSED | 2026-04-13 | 2026-04-27 |
| SDS-CP-Sprint09-2026 | 124906 | CLOSED | 2026-04-27 | 2026-05-11 |
| SDS-CP-Sprint10-2026 | 125513 | CLOSED | 2026-05-11 | 2026-05-25 |
| SDS-CP-Sprint11-2026 | 125626 | **ACTIVE** | 2026-05-25 | 2026-06-08 |
| SDS-CP-Sprint12-2026 | 125627 | FUTURE | 2026-06-08 | 2026-06-22 |

---

## 已验证失败的查询方案

### ❌ 方案一：按 Board 名称过滤（有漏查风险）

```bash
# 只能找到名称含 "SDS" 的 Board，漏掉名称为 "Control Plane" 的 Board 28465
curl -s -H "Authorization: Bearer $JIRA_API_TOKEN" \
  "$JIRA_API_BASE_URL/rest/agile/1.0/board?name=SDS&maxResults=50"
```

**失败原因**：Sprint 所属 Board 的名称不一定包含团队名称前缀，维护 Board 名称映射不可靠，全量 Board 有 4326 个无法全量扫描。

---

### ❌ 方案二：JQL Sprint 通配符查询

```bash
# WAF 拦截，返回 "The request is blocked"
jql: sprint = "SDS-CP-Sprint*"

# Jira 不支持 ~ 操作符用于 sprint 字段，报错：
# "The operator '~' is not supported by the 'sprint' field."
jql: sprint ~ "SDS-CP-Sprint"
```

**失败原因**：Jira JQL 的 `sprint` 字段仅支持精确匹配（`=`），不支持模糊查询；通配符写法被 WAF 拦截。

---

### ❌ 方案三：Sprint Autocomplete API（结果不完整）

```bash
curl -s -H "Authorization: Bearer $JIRA_API_TOKEN" \
  "$JIRA_API_BASE_URL/rest/api/2/jql/autocompletedata/suggestions?fieldName=sprint&fieldValue=SDS-CP-Sprint"
```

**失败原因**：返回上限为 15 条，且会漏掉当前 ACTIVE 的 Sprint（如 SDS-CP-Sprint11-2026 未出现在返回结果中），结果不可靠。

---

### ❌ 方案四：通过 label 查询（依赖人工标注）

```bash
curl -s -H "Authorization: Bearer $JIRA_API_TOKEN" \
  "$JIRA_API_BASE_URL/rest/api/2/jql/autocompletedata/suggestions?fieldName=labels&fieldValue=SDS-CP-Sprint"
```

**失败原因**：label 需要人工维护，不能保证每个 Sprint 的 issue 都打了对应 label，准确性无法保证。

---

### ❌ 方案五：JQL openSprints() 过滤

```bash
jql: sprint in openSprints()
```

**失败原因**：`openSprints()` 仅返回当前用户有权限关联的 Board 上的 Sprint，若用户未关联对应 Board，返回结果为空。
