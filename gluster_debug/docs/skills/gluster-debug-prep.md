# Debug 公司自部署 Gluster Cluster - 准备阶段信息整理

## 1. 文档目标与范围

本文档用于整理公司自部署 Gluster Cluster 的当前已知信息，作为后续构建 debug skill 的输入材料。

当前阶段仅做“事实记录与样例收集”，不在本版本中提供完整排障决策树。

---

## 2. 环境与默认参数规则

### 2.1 部署环境

- Gluster 服务部署在 **tess cluster** 中。
- 服务部署在指定的 **cluster / namespace** 下。

### 2.2 默认 cluster / namespace 解析规则

当命令未显式指定 `--cluster` 或 `-n` 时，从本地上下文读取默认值：

```bash
cat ~/.tess_ns
# 示例输出
908:sdsnushare01-dev
```

据此解析默认参数为：

- cluster: `908`
- namespace: `sdsnushare01-dev`

随后可显式传参查询，例如：

```bash
tk get pod --cluster 908 -n sdsnushare01-dev
```

---

## 3. Gluster 部署拓扑

### 3.1 GlusterFS Deployment 规则

Gluster pod 通过 Deployment 部署，固定存在 3 个 GlusterFS Deployment。

原因：volume 为三副本（replica 3），每个副本所在 node 需要位于不同机架，以避免单机架重启造成所有副本同时受影响。

### 3.2 GlusterFS Deployment 查询示例

```bash
tk get deploy -l glusterfs=deployment

# 示例输出
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
glusterfs-generic-fd1   2/2     2            2           282d
glusterfs-generic-fd2   2/2     2            2           282d
glusterfs-generic-fd3   2/2     2            2           282d
```

### 3.3 Heketi 控制器

Gluster cluster 的控制器为 Heketi，查询示例：

```bash
tk get deploy -l heketi=deployment

# 示例输出
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
heketi-generic-908   1/1     1            1           282d
```

---

## 4. Pod 运行现状快照

以下为当前样例环境中的 pod 状态（节选自 `tkgo` / `tk get pod`）：

```text
glusterfs-generic-fd1-54797dd8b6-cndkj   5/5 Running 0                3d1h
glusterfs-generic-fd1-54797dd8b6-czl5c   5/5 Running 0                6d19h
glusterfs-generic-fd2-775fd7dd7f-j8hh2   5/5 Running 5 (3d ago)       5d23h
glusterfs-generic-fd2-775fd7dd7f-xdvhm   5/5 Running 6 (6d18h ago)    58d
glusterfs-generic-fd3-7bfc674bfc-gq7t6   5/5 Running 26 (12d ago)     44d
glusterfs-generic-fd3-7bfc674bfc-s6ng5   5/5 Running 21 (7d20h ago)   58d
heketi-generic-908-7c9c64bb85-bx9kx      1/1 Running 0                2d17h
```

已观察特征：

- Gluster 各 deployment 当前整体处于 Running/Ready。
- 不同 Pod 的重启次数差异较大（特别是 fd3 相关 pod），可作为后续排障输入信号。

---

## 5. 健康检查命令与样例结果

### 5.1 健康检查执行方式

登录任一 gluster pod 执行：

```bash
tk --cluster 908 -n sdsnushare01-dev exec -it glusterfs-generic-fd3-7bfc674bfc-gq7t6 -- health_check --debug
```

### 5.2 样例输出中的关键阶段

- check pool list
- collect all metrics
- get all bricks
- check glusterd service
- check brick status
- check heal-count

### 5.3 当前样例结论

样例末尾返回：

```text
health_check_is_failed
```

同时输出中可见多个 brick heal count 明显偏高（示例：769、886、872、320、275、258、196 等），也有低于阈值而被忽略的项（阈值示例为 20）。

---

## 6. 当前已观察到的风险信号

1. **heal-count 高值分布较多**
   - 说明存在较明显的 self-heal 压力或历史积压。
2. **部分 Gluster pod 重启次数偏高**
   - 需结合节点事件、容器日志、探针状态进一步确认是否存在周期性抖动。
3. **health_check 总体失败**
   - 尽管基础服务（glusterd/brick status）步骤显示 success，整体仍失败，表明至少有一个关键健康维度未满足门限。

---

## 7. 补充信息清单

> 以下内容待后续继续补充，用于完善 debug skill 的“准备阶段资料”。

1. “如果一个 volume ...” 场景的完整描述与判定条件（你已开始输入，待续）。
2. heal-count 异常时的分级标准（告警阈值、持续时长、是否按 volume 粒度判定）。
3. 常见失败模式到排障动作的映射（例如：网络分区、磁盘故障、node 重启、brick offline）。
4. 推荐的最小排障命令与执行顺序（只读命令优先）。
5. Heketi 侧需要同步检查的项目（拓扑、device 状态、pending operation）。

---

## 8. Skill 化前必须补齐的数据字典与诊断规则

本章节用于把后续需要的信息结构化，便于直接转化为 debug skill 的输入参数、规则引擎和执行步骤。

### 8.1 Volume 维度映射（volume -> brick -> pod/node/rack）

目标：当某个 volume 出现异常时，可在 1~2 步内定位到对应副本与故障域。

建议字段模板：

| 字段 | 示例 | 说明 |
|---|---|---|
| volume_name | vol-xxx | volume 名称 |
| volume_id | xxxxxxxx | volume 唯一标识 |
| replica_count | 3 | 副本数 |
| brick_path_list | /var/lib/heketi/... | 该 volume 的全部 brick 路径 |
| brick_to_pod | brickA -> glusterfs-generic-fd1-xxx | brick 与 pod 映射 |
| pod_to_node | podA -> tess-node-xxx | pod 与 node 映射 |
| node_to_rack | nodeA -> rack-1 | node 与机架映射 |
| is_rack_isolated | true/false | 三副本是否满足跨机架 |

---

### 8.2 健康判定标准（阈值 + 持续时间 + 分级）

目标：把“看到异常”转为“可自动判定等级”。

建议定义模板：

| 指标 | Warning | Critical | 持续时间要求 | 备注 |
|---|---|---|---|---|
| heal_count_per_brick | >= W1 | >= C1 | 连续 T 分钟 | 建议按 volume 聚合 |
| pod_restart_delta | >= W2 | >= C2 | 近 N 小时 | 关注短期突增 |
| offline_brick_count | >= 1 | >= 2 | 实时 | 可直接触发高优先级 |
| health_check_result | failed | failed + 关键项失败 | 实时 | 结合具体失败项解释 |

> 注：W1/C1/W2/C2/T/N 由你后续补具体值。

---

### 8.3 故障模式 -> 诊断路径映射

目标：把“症状”映射成固定排障路线，减少临场判断波动。

建议模板：

| 症状/触发信号 | 初步判断 | 第一检查项 | 第二检查项 | 第三检查项 | 结论输出 |
|---|---|---|---|---|---|
| heal-count 持续高 | self-heal 积压或链路抖动 | volume heal info | brick online 状态 | node/rack 事件 | 根因候选 + 下一步 |
| pod 重启异常 | 节点/探针/容器异常 | pod describe/events | 容器日志 | 节点状态 | 是否需要节点侧介入 |
| brick offline | 存储或网络异常 | volume status | 对应节点健康 | heketi 设备状态 | 是否触发修复流程 |

---

### 8.4 最小化排障命令清单（只读优先）

目标：给 skill 一个安全、稳定、可复现的命令执行顺序。

建议格式：

| 顺序 | 命令 | 预期输出要点 | 异常特征 | 风险等级 |
|---|---|---|---|---|
| 1 | tk get pod ... | gluster/heketi 全部 Running | CrashLoopBackOff/NotReady | 低 |
| 2 | tk get deploy ... | ready 副本满足期望 | ready 不足 | 低 |
| 3 | health_check --debug | 各检查阶段 success | health_check_is_failed | 低 |
| 4 | volume 相关只读查询 | heal/offline 信息明确 | heal 激增或 brick offline | 低 |

---

### 8.5 Heketi 拓扑与容量检查基线

目标：确保从控制器视角验证存储拓扑与容量健康。

建议补齐项：

1. Heketi 拓扑查询命令与字段解释（cluster/node/device）。
2. 设备可用容量、已用容量、阈值告警规则。
3. pending operation 的识别与处理建议。
4. heketi pod 与 gluster pod 关联检查（同 node 风险、通信风险）。

---

### 8.6 Node/Rack/Zone 约束验证规则

目标：自动判断三副本是否满足“跨故障域部署”。

建议规则：

1. 单 volume 三副本必须分布在不同 rack。
2. 若发现同 rack 多副本，标记为高风险结构性问题。
3. node 重启时应评估同 rack 其他副本可用性，避免并发影响。

建议输出模板：

| volume | replica 节点 | rack 分布 | 是否合规 | 风险等级 |
|---|---|---|---|---|
| vol-xxx | nodeA/nodeB/nodeC | rack1/rack2/rack3 | yes | low |

---

### 8.7 关键日志入口与关键词字典

目标：统一日志定位路径与关键词解释，方便 skill 自动给出 next step。

建议补齐：

1. glusterfs / glusterd / heketi 的日志入口（容器名、路径、命令）。
2. 高频关键词与含义（timeout、transport、peer、split-brain、I/O error 等）。
3. 关键词 -> 建议下一步检查项映射。

---

### 8.8 处置动作分级（观察 / 干预 / 高风险）

目标：为 skill 增加“安全护栏”，避免直接执行高风险动作。

建议分级：

| 级别 | 类型 | 说明 | 示例 |
|---|---|---|---|
| L1 | 观察类 | 纯查询，无状态变更 | get/describe/log/health_check |
| L2 | 低风险干预 | 小范围可回滚动作 | 单 pod 重启（需条件） |
| L3 | 高风险干预 | 可能影响业务数据或可用性 | volume/heal 强制操作、拓扑变更 |

规则建议：

1. skill 默认只执行 L1。
2. 涉及 L2/L3 动作时必须输出确认提示与风险说明。
3. L3 需要明确变更窗口与人工审批条件。

---

## 9. Volume 写入失败最小排障流程（L1 只读版）

> 目标：在不做状态变更的前提下，快速判断“写入失败”更可能发生在客户端、网络路径、Gluster 副本状态、还是容量层。

### 9.1 输入信息（必须先收集）

1. 失败业务 Pod 名称、所在 node、失败时间窗口。
2. 报错原文（例如 `Read-only file system`、`Input/output error`、`Connection timed out`）。
3. 目标 PVC/PV/volume 名称（至少一个可追踪标识）。

### 9.2 执行步骤（建议顺序，优先使用 nst）

#### Step 1: 确认上下文 cluster/ns

```bash
cat ~/.tess_ns
# 若业务指定 cluster/ns，则以显式参数优先
```

#### Step 2: 检查 Gluster/Heketi 基础存活

```bash
# Gluster 存储池与基础状态
nst gfs pool list -c <cluster> -n <namespace>

# Heketi 健康与砖块汇总
nst heketi health -c <cluster> -n <namespace>
nst heketi brick summary -c <cluster> -n <namespace>
```

判定要点：
- 若 heketi health 异常、brick summary 显示异常聚集，优先归类为“平台侧异常”。

#### Step 3: 获取目标 PVC/PV/Volume 关联

```bash
# 从业务侧先拿 PVC 详情
nst tess pvc info -c <cluster> -n <biz-namespace> <pvc>

# 可选：列出 PV 辅助定位
nst tess pv list -c <cluster>

# 若已知关键字，可反查 volume
nst gfs find volume -c <cluster> -n <namespace> <keyword_or_filter>
```

判定要点：
- 必须形成至少一条可追踪链路：`业务 Pod -> PVC -> PV -> Gluster Volume`。

#### Step 4: 检查目标 volume 详情与诊断

```bash
nst gfs volume info -c <cluster> -n <namespace> <volume>
nst gfs volume diag -c <cluster> -n <namespace> <volume>
nst gfs volume find-lock -c <cluster> -n <namespace> <volume>
```

判定要点：
- 若存在 lock 异常、brick 异常或 volume 诊断异常，直接提升优先级。

#### Step 5: 绑定目标 volume 到副本位置

```bash
# 结合 volume info/diag + heketi brick summary
# 补齐 volume -> brick -> pod/node/rack 映射（对应第 8.1 数据字典）
```

判定要点：
- 若目标 volume 存在 brick offline / 同 rack 多副本 / 副本节点同时抖动，直接提升优先级。

#### Step 6: 检查业务 Pod 侧证据（客户端视角）

```bash
# 若现场暂无 nst 对应命令，使用 tk 读取业务 pod 事件与日志
tk describe pod <biz-pod> --cluster <cluster> -n <biz-namespace>
tk logs <biz-pod> --cluster <cluster> -n <biz-namespace> --since=30m
```

判定要点：
- 若仅单 pod/单 node 复现，优先怀疑客户端节点侧或路径问题。
- 若多 pod 跨 node 同时失败，优先怀疑 volume/平台侧问题。

#### Step 7: 容量与 inode 基线检查

```bash
# 优先通过 heketi 视角观察容量
nst heketi volume list -c <cluster> -n <namespace>

# 必要时在对应 gluster 节点或容器内执行只读容量检查（按现场工具）
# df -h
# df -i
```

判定要点：
- 容量或 inode 耗尽可直接导致写入失败。

---

### 9.3 快速分流结论模板

| 现象组合 | 初步结论 | 下一步 |
|---|---|---|
| 业务报错 + 平台健康正常 + 单 node 复现 | 客户端节点/路径问题概率高 | 查该 node 网络、挂载、内核日志 |
| 多业务 pod 同时失败 + health_check failed | Gluster 集群健康问题概率高 | 查目标 volume 的 brick/heal/offline |
| 报错为只读或 I/O 错误 + brick 异常 | volume 副本异常概率高 | 查副本状态与故障域分布 |
| 容量/inode 紧张 | 资源耗尽导致失败 | 走容量处置流程 |

---

### 9.4 输出规范（供未来 skill 使用）

每次排障至少输出：

1. `scope`: cluster/ns + volume + 业务 pod。
2. `symptom`: 报错原文 + 发生时间。
3. `health_baseline`: deploy/pod/health_check 摘要。
4. `volume_mapping`: volume->brick->pod/node/rack（若缺失，标记阻塞）。
5. `provisional_conclusion`: 当前最可能方向（1~2 个）。
6. `next_action`: 仅 L1 只读动作；若需 L2/L3，明确“需人工确认”。

### 9.5 典型场景：服务端 brick 在线，但客户端对单个 subvol 三副本全部 ENOTCONN

场景特征：

1. volume 为 replica 3，包含多个 subvol（例如 4 组，共 12 bricks）。
2. `gluster v status <volume>` 显示 brick 进程在线。
3. 客户端日志出现同一组 `client-x`（例如 client-6/7/8）同时报 `Transport endpoint is not connected (errno=107)`。
4. 用户侧写入失败可稳定复现（如 `touch` 返回 ENOTCONN）。

判定说明：

- 这通常是“客户端到某个 subvol 的传输层连接全部失效”，而不是“brick 进程不存在”。
- replica 3 的单个 subvol 若 0/3 可连通，该 hash 范围内写请求会失败；其他 subvol 可能仍正常，因此表现为“部分路径失败/间歇失败”。

建议排查顺序：

1. **先定位失败 subvol 与三副本端点**
   - 从客户端 mount 日志中的 `client-x` 反查到对应 brick IP:port。
   - 用 `nst gfs volume info/diag <volume>` 固化 `subvol -> brick` 映射。
2. **在客户端节点验证到三个 brick 端口连通性**
   - 检查到目标 `IP:PORT` 的 TCP 状态（是否持续重连、超时、RST）。
   - 对照失败时间窗口确认是否同时失联。
3. **检查服务端 brick 日志与 glusterd 日志（同时间窗口）**
   - 关注 transport 断连、rpc 超时、认证失败、端口变更相关日志。
4. **检查网层与节点状态**
   - 节点网络抖动、丢包、conntrack 压力、策略变更（安全组/网络策略/iptables）。
   - 特别关注“单 client 节点到特定三副本节点”的定向问题。
5. **校验是否为客户端挂载会话陈旧**
   - 若 brick 重启后恢复，需重点怀疑客户端 transport 会话卡死或 graph 未及时刷新。

处置建议（分级）：

- **L1（默认）**：收集日志、端口连通性、subvol 映射、时间线证据。
- **L2（需确认）**：单客户端侧重新挂载或重启对应 client 进程验证。
- **L3（高风险）**：brick-op/cluster-op 或批量重启，必须人工审批。

### 9.6 证据采集清单（按时间窗口执行）

> 目标：在一次排查中拿到可闭环证据，避免“看起来恢复了但根因不明”。

先定义变量（示例）：

```bash
# 现场按实际替换
CLUSTER=908
GFS_NS=sdsnushare01-dev
BIZ_NS=<biz-namespace>
VOLUME=vol_6e3fb78ceb67b9e2fc2c178a2f2a823e
PVC=<pvc-name>
BIZ_POD=<biz-pod>
WINDOW=60m
```

#### A. 基线信息（volume 与平台）

```bash
nst gfs volume info -c ${CLUSTER} -n ${GFS_NS} ${VOLUME}
nst gfs volume diag -c ${CLUSTER} -n ${GFS_NS} ${VOLUME}
nst gfs volume find-lock -c ${CLUSTER} -n ${GFS_NS} ${VOLUME}
nst heketi health -c ${CLUSTER} -n ${GFS_NS}
nst heketi brick summary -c ${CLUSTER} -n ${GFS_NS}
```

采集目标：
- 固化 `subvol -> brick(IP:path)` 对应关系。
- 固化当时的 heketi/gluster 健康状态。

#### B. 客户端侧证据（失败现场）

```bash
# 业务 pod 基线
 tk describe pod ${BIZ_POD} --cluster ${CLUSTER} -n ${BIZ_NS}
 tk logs ${BIZ_POD} --cluster ${CLUSTER} -n ${BIZ_NS} --since=${WINDOW}

# 客户端挂载日志（在客户端节点）
# 重点 grep: Transport endpoint is not connected / client-x / reconnect / disconnected
```

采集目标：
- 记录报错原文与时间戳。
- 记录失败时对应的 client-x 编号（如 client-6/7/8）。

#### C. client-x 到 brick 端点映射

```bash
# 从 mount log 中提取 client-x -> brick endpoint
# 再与 nst gfs volume info/diag 输出对照
```

采集目标：
- 明确“失败的是哪一组 subvol（三副本）”。

#### D. 网络/传输层证据（客户端到三副本端口）

```bash
# 在客户端节点检查到 3 个 brick IP:port 的连接状态（按现场工具）
# 例如：ss/netstat/tcpdump（只读采集）
```

采集目标：
- 是否同时超时、RST、反复重连。
- 是否仅该客户端节点异常，还是全局异常。

#### E. 服务端日志证据（对应三副本）

```bash
# 在三个 brick 所在 pod/节点采集同时间窗口日志
# 重点 grep: transport/rpc/disconnect/auth/port
```

采集目标：
- 与客户端错误时间戳对齐，确认是服务端拒绝、网络中断还是会话问题。

#### F. 输出结论模板（单次排障）

```text
[Scope]
cluster/ns:
volume:
biz pod/node:

[Symptom]
first_seen:
error:

[Subvol Mapping]
failed_subvol:
brick_endpoints:

[Client Evidence]
client-log-summary:
transport-state-summary:

[Server Evidence]
brick-log-summary:
heketi-summary:

[Provisional Root Cause]
1)
2)

[Next Action]
L1/L2/L3:
reason:
```

---

## 附录 A：已记录的关键命令（原样保留）

```bash
# 查询 gluster deployment
tk get deploy -l glusterfs=deployment

# 查询 heketi deployment
tk get deploy -l heketi=deployment

# 查询 pod（默认上下文）
tkgo

# 查询 pod（显式 cluster/ns）
tk get pod --cluster 908 -n sdsnushare01-dev

# 查看默认上下文
cat ~/.tess_ns

# 执行健康检查
tk --cluster 908 -n sdsnushare01-dev exec -it glusterfs-generic-fd3-7bfc674bfc-gq7t6 -- health_check --debug
```

---

## 附录 B：Gluster 项目排查工具（nst）命令整理

> 本附录为你提供的命令文档整理版，后续可直接映射到 skill 的命令层。

### B.1 通用用法

#### 查看帮助

```bash
nst help
nst help <subcommand>
nst <subcommand> -h
```

#### 前缀缩写

支持命令前缀匹配，例如：

```bash
nst t p i ...
# 等价于 nst tess pvc info ...
```

#### 全局参数

常见可用：

```bash
nst --debug <subcommand> ...
```

#### 资源定位格式（重点）

很多命令支持 FQDN 风格资源名：

```text
cluster/namespace/name
```

例如 PVC、Volume 相关命令。

---

### B.2 tess 命令组

#### tess pvc info

查看 PVC 的状态、引用、应用关联等信息。

```bash
nst tess pvc info [options] <pvc>
```

常用参数：
- `-c, --cluster`
- `-n, --namespace`

#### tess pvc add-ref

给 PVC 添加引用。

```bash
nst tess pvc add-ref [options] <pvc> <ref>
```

#### tess pvc rm-ref

移除 PVC 引用。

```bash
nst tess pvc rm-ref [options] <pvc> <ref>
```

#### tess pvc add-app

给 PVC 绑定应用标识。

```bash
nst tess pvc add-app [options] <pvc> <app>
```

#### tess pvc rm-app

移除 PVC 绑定的应用。

```bash
nst tess pvc rm-app [options] <pvc> <app>
```

#### tess pv list

列出 PV（支持筛选）。

```bash
nst tess pv list [options]
```

#### tess node list

列出节点信息（支持字段筛选/展示）。

```bash
nst tess node list [options]
```

---

### B.3 gfs 命令组

#### gfs cmd

在 gfs 相关容器/环境执行命令。

```bash
nst gfs cmd [options] -- <raw command>
```

#### gfs pool list

列出存储池信息。

```bash
nst gfs pool list [options]
```

#### gfs volume info

查看 volume 详情。

```bash
nst gfs volume info [options] <volume>
```

常用参数：
- `-c, --cluster`
- `-n, --namespace`

#### gfs volume connect

查看/生成 volume 连接信息。

```bash
nst gfs volume connect [options] <volume>
```

#### gfs volume diag

volume 诊断信息。

```bash
nst gfs volume diag [options] <volume>
```

#### gfs volume find-lock

查找 volume 锁信息。

```bash
nst gfs volume find-lock [options] <volume>
```

#### gfs find volume

按条件查找 volume。

```bash
nst gfs find volume [options] <keyword_or_filter>
```

#### gfs brick-op

对 brick 执行运维动作（风险较高，通常用于故障修复）。

```bash
nst gfs brick-op [options] <volume> <operation>
```

常见附加参数：
- `-s, --subvol`
- `-i, --index`

#### gfs cluster-op

集群级运维动作。

```bash
nst gfs cluster-op [options] <operation>
```

#### gfs tess-op

tess 集成场景下的 gfs 运维动作。

```bash
nst gfs tess-op [options] <operation>
```

#### gfs client-op

客户端侧操作。

```bash
nst gfs client-op [options] <operation>
```

#### gfs drop-cache

执行缓存清理相关操作。

```bash
nst gfs drop-cache [options] <target>
```

---

### B.4 heketi 命令组

#### heketi raw

透传 heketi 原生命令。

```bash
nst heketi raw [options] -- <heketi raw args>
```

#### heketi volume list

列出 heketi volume。

```bash
nst heketi volume list [options]
```

#### heketi brick summary

查看 brick 汇总信息。

```bash
nst heketi brick summary [options]
```

#### heketi cap add

增加容量/扩容相关操作。

```bash
nst heketi cap add [options] <args>
```

#### heketi health

健康检查。

```bash
nst heketi health [options]
```

---

### B.5 csi 命令组

#### csi purge-gfs-log

清理 gfs 日志（CSI 场景）。

```bash
nst csi purge-gfs-log [options]
```

常见参数：
- `--cluster`（该命令无短参 `-c`）
- `-n, --namespace`
- `-p, --pod`
- `-s, --size`
- `-y, --yes`

---

### B.6 命令风险分层建议（供 skill 编排使用）

- **L1 只读类（默认）**：`info/list/diag/find-lock/find volume/health/summary`
- **L2 低风险变更类（需确认）**：`pvc add-ref/rm-ref/add-app/rm-app`（依赖业务流程）
- **L3 高风险运维类（必须人工确认）**：`gfs brick-op/cluster-op/tess-op/client-op/drop-cache`、`heketi cap add`、`csi purge-gfs-log`

skill 默认应优先 L1，涉及 L2/L3 时提示风险并等待人工确认。

---

## 附录 C：ENOTCONN 场景决策树（推荐执行模型）

### C.1 方案选择

- **方案 A（推荐）**：按“证据优先”决策树执行（先定界、再分流、最后处置）。
- 方案 B（备选）：按组件分组检查（client/server/network 各自查完再汇总）。

> 选择理由：方案 A 在故障窗口有限时更快收敛，并且更适合后续 skill 自动化编排。

### C.2 决策树

```text
Start
  |
  |-- S1: 是否可稳定复现 ENOTCONN?
  |      |-- 否 -> 进入“间歇故障采样模式”（拉长 WINDOW + 多点采样）
  |      `-- 是 -> S2
  |
  |-- S2: 是否定位到失败 subvol（三副本端点）?
  |      |-- 否 -> 先做 client-x 与 brick 端点映射
  |      `-- 是 -> S3
  |
  |-- S3: 客户端到三端点连接是否同时异常?
  |      |-- 是 -> S4
  |      `-- 否 -> 优先客户端局部问题（挂载会话/节点网络）
  |
  |-- S4: 服务端日志同窗是否有 transport/rpc 异常?
  |      |-- 是 -> 优先服务端/网络路径问题
  |      `-- 否 -> 优先客户端会话陈旧或中间网络策略问题
  |
  `-- S5: 处置分级
         |-- L1 收集完成 -> 输出 provisional root cause
         |-- L2 验证动作 -> 小范围、可回滚、需确认
         `-- L3 高风险动作 -> 必须审批
```

---

## 附录 D：命令模板（可直接复制，L1 默认）

### D.1 统一变量块（推荐）

- **方案 A（推荐）**：统一变量块，所有命令复用变量。
- 方案 B（备选）：每条命令手工替换参数。

```bash
CLUSTER=908
GFS_NS=sdsnushare01-dev
BIZ_NS=<biz-namespace>
BIZ_POD=<biz-pod>
VOLUME=vol_6e3fb78ceb67b9e2fc2c178a2f2a823e
WINDOW=60m
BRICK1_IP=<ip1>
BRICK1_PORT=<port1>
BRICK2_IP=<ip2>
BRICK2_PORT=<port2>
BRICK3_IP=<ip3>
BRICK3_PORT=<port3>
```

### D.2 平台与 volume 基线

```bash
nst gfs volume info -c ${CLUSTER} -n ${GFS_NS} ${VOLUME}
nst gfs volume diag -c ${CLUSTER} -n ${GFS_NS} ${VOLUME}
nst gfs volume find-lock -c ${CLUSTER} -n ${GFS_NS} ${VOLUME}
nst heketi health -c ${CLUSTER} -n ${GFS_NS}
nst heketi brick summary -c ${CLUSTER} -n ${GFS_NS}
```

### D.3 客户端证据采集

```bash
tk describe pod ${BIZ_POD} --cluster ${CLUSTER} -n ${BIZ_NS}
tk logs ${BIZ_POD} --cluster ${CLUSTER} -n ${BIZ_NS} --since=${WINDOW}
```

### D.4 客户端到三副本端口连通性（只读）

- **方案 A（推荐）**：`ss` 快速看连接状态。
- 方案 B（备选）：`tcpdump` 抓包确认握手/RST/重传。

```bash
# 方案 A：快速状态
ss -tanp | grep -E "${BRICK1_IP}:${BRICK1_PORT}|${BRICK2_IP}:${BRICK2_PORT}|${BRICK3_IP}:${BRICK3_PORT}"

# 方案 B：采样抓包（按现场权限）
# tcpdump -i any host ${BRICK1_IP} and port ${BRICK1_PORT} -nn -tttt
# tcpdump -i any host ${BRICK2_IP} and port ${BRICK2_PORT} -nn -tttt
# tcpdump -i any host ${BRICK3_IP} and port ${BRICK3_PORT} -nn -tttt
```

### D.5 服务端日志同窗采集（只读）

```bash
# 在三副本对应 gluster pod/节点执行（按现场工具）
# 关键词建议：transport|rpc|disconnect|timeout|auth|port
```

---

## 附录 E：Skill 规则化草案（可直接转 skill）

### E.1 输入契约

必填：
- cluster
- gluster namespace
- business namespace
- volume
- business pod
- time window

可选：
- pvc/pv
- known failed client-x
- suspected brick endpoints

### E.2 输出契约

- health baseline
- failed subvol mapping
- client evidence summary
- server evidence summary
- provisional root cause (top 1-2)
- next actions by level (L1/L2/L3)

### E.3 自动执行边界

- **方案 A（推荐）**：默认仅自动执行 L1；L2/L3 仅生成建议，不自动执行。
- 方案 B（备选）：允许自动执行部分 L2（需要预授权白名单）。

> 推荐 A，原因：可降低误操作风险，更适合多租户与生产环境。

### E.4 完成判定（Definition of Done）

一次排障任务仅在满足以下条件时标记完成：

1. 已定位失败 subvol 与三副本端点。
2. 已收集客户端与服务端同时间窗证据。
3. 已输出至少一个可验证的根因方向。
4. 若建议 L2/L3，已附风险说明和确认要求。