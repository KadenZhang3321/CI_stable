# 开源社区多集群基础设施主动监控体系设计

**文档版本：** v3.1

**适用范围：** 多个 Kubernetes 集群环境下的 CI/CD 基础设施健康监控与告警

**目标：** 构建跨集群的统一可观测性平台，主动发现组件、网络、存储、云资源故障，按严重级别生成结构化邮件通知，交由办公软件机器人解析并分发给对应责任人。

---

## 一、设计目标与原则

- **集中式指标采集与告警计算：** 所有集群的健康指标汇聚到一个中心 Prometheus，由中心统一计算告警，避免在每个集群维护独立的告警规则和通知渠道。
- **轻量级集群侧部署：** 每个业务集群仅运行采集器（kube-state-metrics、node-exporter、Pushgateway、CronJob），不独立部署 Prometheus 和 Alertmanager，降低维护成本。
- **统一邮件出口：** 所有告警仅通过 Alertmanager 发送到唯一的企业邮箱地址，不在系统中配置多种通知渠道。告警的分发由外部办公软件机器人完成。
- **结构化邮件标题：** 邮件标题包含严重级别、集群标识、告警名称等字段，固定格式，保证机器人可准确解析并私聊到对应运维人员。
- **主动巡检：** 对已知问题实施周期性拨测，在用户可感知之前主动发现异常，将运维模式从被动响应转变为主动探知。

---

## 二、总体架构

```
                                  ┌──────────────────────────┐
                                  │         中心集群          │
                                  │                          │
                                  │  ┌──────────┐┌──────────┐│
                                  │  │ Prom-1   ││ Prom-2   ││
                                  │  │ (HA 对)  ││          ││
                                  │  └────┬─────┘└────┬─────┘│
                                  │       └─────┬──────┘      │
                                  │             │             │
                                  │   ┌─────────▼──────────┐  │
                                  │   │ Alertmanager       │  │
                                  │   │ (Gossip 集群 ×2)    │  │
                                  │   └─────────┬──────────┘  │
                                  │             │             │
                                  │   ┌─────────▼──────────┐  │
                                  │   │ Grafana (单实例)   │  │
                                  │   └────────────────────┘  │
                                  │                          │
                                  │   ┌────────────────────┐  │
                                  │   │ Pushgateway (可选) │◄─┼──┐
                                  │   └────────────────────┘  │  │
                                  └─────────────┬────────────┘  │
                                                │               │
          ┌─────────────────────────────────────┼───────────────┼──┐
          │                                     │               │  │
  ┌───────▼────────┐              ┌────────────▼──┐            │  │
  │    集群 A       │              │    集群 B       │     ...   │  │
  │                │              │                │            │  │
  │ ┌────────────┐ │              │ ┌────────────┐ │            │  │
  │ │Pushgateway │◄├──┐           │ │Pushgateway │◄├──┐         │  │
  │ └────────────┘ │  │           │ └────────────┘ │  │         │  │
  │                │  │           │                │  │         │  │
  │ ┌────────────┐ │  │           │ ┌────────────┐ │  │         │  │
  │ │kube-state- │ │  │           │ │kube-state- │ │  │         │  │
  │ │  metrics   │ │  │           │ │  metrics   │ │  │         │  │
  │ └────────────┘ │  │           │ └────────────┘ │  │         │  │
  │                │  │           │                │  │         │  │
  │ ┌────────────┐ │  │           │ ┌────────────┐ │  │         │  │
  │ │   node-    │ │  │           │ │   node-    │ │  │         │  │
  │ │  exporter  │ │  │           │ │  exporter  │ │  │         │  │
  │ └────────────┘ │  │           │ └────────────┘ │  │         │  │
  │                │  │           │                │  │         │  │
  │ ┌────────────┐ │  │           │ ┌────────────┐ │  │         │  │
  │ │CronJob     ├──┘  │           │ │CronJob     ├──┘  │         │  │
  │ │ 巡检集     ├───┐ │           │ │ 巡检集     ├───┐ │         │  │
  │ └────────────┘   │ │           │ └────────────┘   │ │         │  │
  └───────────────────┼─┘           └──────────────────┼─┘         │  │
                      └────────────────────────────────┘            │  │
                       CronJob 关键巡检 ─ ─ ─虚线: 直推中心 Pushgateway  │  │
                       (绕过集群侧，避免单点)                            │  │

  中心 Prometheus ──远程拉取──► 各集群 Pushgateway / kube-state-metrics / node-exporter

  Alertmanager ──SMTP──► infra-alert@community.org ──机器人轮询──► 办公软件机器人 ──解析标题私聊──► 运维人员
```

> **注：** 上图为逻辑架构。Prometheus HA、Alertmanager Gossip 集群、中心 Pushgateway 的物理部署方案详见 [2.1 节](#21-中心集群高可用) 和 [4.4 节](#44-跨集群网络与安全)。

### 架构要点说明

| 组件 | 说明 |
|------|------|
| **中心集群** | 承担全部告警计算与分发，部署 Prometheus、Alertmanager、Grafana，是监控大脑。 |
| **业务集群** | 仅部署轻量级采集器，无状态，可快速复制到新集群。 |
| **通知链路** | Alertmanager → SMTP → 企业邮箱 → 机器人轮询解析 → 私聊通知。该设计将告警路由逻辑从监控系统剥离，交给办公软件侧灵活管理。 |

### 2.1 中心集群高可用

中心集群是监控大脑，宕机会导致所有集群告警中断。最低限度应保障：

- **Prometheus HA：** 部署两个相同配置的 Prometheus 实例独立抓取，互为备份。后续可引入 Thanos Sidecar + 对象存储实现长期保留与全局查询去重。
- **Alertmanager 集群：** 至少部署 2 个实例组成 Gossip 集群，共享告警状态，避免单实例宕机时丢失告警或重复发送。
- **Grafana：** 可降级为单实例（故障时不影响告警通路），使用 PVC 持久化仪表盘配置。

---

## 三、邮件通知设计

### 3.1 唯一收件地址

所有告警均发送至：`infra-alert@community.org`（示例地址，具体由社区确定）。

该邮箱由办公软件机器人专用，不与其他邮箱混合使用。

### 3.2 邮件标题格式

标题采用固定结构，方便机器人解析：

```
[级别][集群]-告警名称[-附加标签]
```

| 字段 | 说明 |
|------|------|
| **级别** | `CRITICAL`、`WARNING`、`INFO` |
| **集群** | 业务集群的唯一标识，如 `prod-sh-1`、`staging` 等 |
| **告警名称** | Prometheus 告警规则中定义的 `alertname` |
| **附加标签**（可选） | 如涉及特定实例、命名空间时，追加用 `-` 连接的键值对 |

**示例：**

- `[CRITICAL][prod-sh-1] GithubUnreachable`
- `[WARNING][prod-sh-1] SharedDiskSpaceHigh-instance-shared`
- `[CRITICAL][staging] RunnerDown-deployment-runner`
- `[INFO][prod-sh-1] RecentOOMKilled-namespace-ci`

### 3.3 邮件正文

正文包含：

- 告警摘要（annotations 中的 `summary`）
- 详细描述（annotations 中的 `description`）
- 发生时间、当前值
- 附加标签列表，便于机器人二次路由

### 3.4 机器人路由逻辑（由办公软件侧实现）

机器人从邮箱拉取未读邮件，解析标题中的：

| 解析字段 | 用途 |
|---------|------|
| **级别** | 决定私聊的紧急程度和是否追加提醒 |
| **集群** | 决定通知哪一组运维人员（不同集群可能由不同人负责） |
| **告警名称** | 决定更精细的指派策略（例如存储类告警通知存储管理员） |

> 该部分逻辑不在本文档范围内，由办公软件机器人开发团队依据标题格式实现。

---

## 四、多集群指标采集设计

### 4.1 集群标识

每个业务集群必须拥有全局唯一的 `cluster` 标签，并注入到所有指标中：

- 采集组件（kube-state-metrics、node-exporter）通过外部标签注入。
- CronJob 在推送到 Pushgateway 时显式附加 `cluster` 标签。
- 中心 Prometheus 在抓取配置中为各集群 job 添加静态 `cluster` 标签，作为兜底。

### 4.2 采集端组件

每个业务集群需部署：

| 组件 | 用途 |
|------|------|
| **kube-state-metrics** | K8s 资源对象状态指标 |
| **node-exporter** | 节点系统指标（CPU、内存、磁盘、网络） |
| **Pushgateway** | 接收 CronJob 推送的临时指标 |
| **CronJob 巡检集** | 周期性执行拨测脚本，结果推送至本地 Pushgateway |

所有组件在同一 `monitoring` 命名空间下运行，统一管理。

### 4.3 中心 Prometheus 抓取

中心 Prometheus 通过添加 scrape job 拉取各集群端点。需保证网络可达（专线、VPN、LoadBalancer 暴露等）。在 job 配置中附加 `cluster` 标签，用于告警路由。

### 4.4 跨集群网络与安全

跨集群抓取是整个方案的关键依赖，需从可靠性和安全性两个维度保障。

**可靠性：**

- **抓取间隔：** 跨集群场景下推荐 30s–60s，避免频繁的跨网请求放大瞬时故障。
- **`for` 持续时间：** 告警规则中的 `for` 应至少为抓取间隔的 2–3 倍，防止瞬时网络波动触发误报。
- **Pushgateway 高可用：** Pushgateway 不支持集群/复制模式，同一集群部署多实例会导致指标分裂，不是真正的 HA。正确做法：关键巡检 CronJob 直接推送到中心集群的 Pushgateway，集群侧不部署 Pushgateway 也能工作；如果仍需集群侧 Pushgateway，则 CronJob 同时向集群侧和中心侧各推一份作为冗余。

**网络安全：**

- 跨集群抓取链路应启用 TLS（Prometheus `scheme: https`）或 mTLS，防止指标数据在公网/专线中被窃听。
- Exporter 端点应配置 HTTP Basic Auth 或 Bearer Token 认证，避免未授权方拉取集群指标。
- 网络策略（NetworkPolicy）限制 Prometheus 抓取来源 IP，仅允许中心集群的抓取端 IP 通过。

### 4.5 Pushgateway 指标时效性

Pushgateway 中的指标在主动删除前会一直保留。如果某个 CronJob 停止推送（Pod 被驱逐、镜像拉取失败等），旧值会持续存在，导致告警永远触发或永远静默。需在告警规则中加入时效性校验：

```promql
# 示例：只告警最近 5 分钟内推过的新鲜值
(github_probe_success == 0)
and
(time() - push_time_seconds{job="cronjob"}) < 300
```

同时应在中心侧部署定时任务，定期清理 Pushgateway 中超过阈值的陈旧指标。

### 4.6 可扩展性与服务发现

随着集群数量增长，中心 Prometheus 的资源压力和被管理目标的复杂性同步上升，需在架构层面预留扩展路径：

- **水平扩展：** Prometheus 原生不支持集群，当单实例内存/IO 触及瓶颈时，可引入 Thanos 或 VictoriaMetrics，将数据持久化与查询聚合分离到外部存储层，Prometheus 自身仅保留短窗口热数据。
- **服务发现：** 初期可按静态文件方式管理各集群端点（`file_sd_configs`），目标列表用 Git 管理。集群数量增多后可迁移到 Consul 或 Kubernetes API 实现动态发现，避免手动维护 scrape target 列表。
- **基数控制：** 高基数指标（如 Pod 级别、容器级别）是 Prometheus 内存的最大消耗来源。应在 scrape 阶段通过 `metric_relabel_configs` 丢弃不必要的标签，仅保留 `namespace`、`deployment` 等聚合维度标签用于告警。示例配置：

```yaml
metric_relabel_configs:
  # 丢弃 Pod 维度的高基数标签
  - action: labeldrop
    regex: "pod|container|container_id|uid"
  # 仅保留 deployment 级别的聚合维度
  - action: labelkeep
    regex: "__name__|cluster|namespace|deployment|job"
```

---

## 五、巡检项与告警规则设计

沿用前述 9 个问题对应的巡检项，所有告警规则统一写在中心 Prometheus 的规则文件中，通过 `cluster` 标签区分来源。

| 编号 | 问题 | 巡检方式 | 关键指标 | 告警触发条件 | 严重级别 | 建议频率 |
|------|------|---------|---------|-------------|---------|---------|
| 1 | GitHub 不可达 | 各集群 CronJob 拨测 | `github_probe_success` | `== 0` 持续 2m | **CRITICAL** | 1m |
| 2 | ServiceAccount 权限过大 | 每日权限合规检查 | `sa_privilege_excess` | `> 0` 持续 1h | **WARNING** | 日 |
| 3 | 云账号欠费/冻结 | CronJob 查询云 BSS | `cloud_account_status` | `== 0` 持续 5m | **CRITICAL** | 10m |
| 4 | 共享盘使用率过高 | CronJob df 或云监控 | `shared_disk_usage_percent` | `> 85` 持续 5m (WARNING)<br>`> 95` 持续 1m (CRITICAL) | **WARNING / CRITICAL** | 5m |
| 5 | 代理证书过期/代理宕机 | CronJob 证书检查 + 连通性拨测 | `proxy_cert_days_left`<br>`proxy_reachable` | `< 30` 天 (WARNING)<br>`== 0` 持续 2m (CRITICAL) | **WARNING / CRITICAL** | 证书日检<br>连通性 10m |
| 6 | 代码仓镜像同步失败 | CronJob 检查同步状态 | `mirror_sync_success` | `== 0` 持续 1h | **WARNING** | 日检/同步后触发 |
| 7 | Pod OOM 用户无感知 | kube-state-metrics 提供 OOM 计数；CI 模板自动评论 | `kube_pod_container_status_terminated_reason` | 10m 内出现 OOM 增量 `> 0` | **INFO** | 实时 |
| 8 | Runner 副本不可用 | kube-state-metrics 监控 Deployment 副本 | `kube_deployment_status_replicas_available` | `== 0` 持续 2m | **CRITICAL** | 实时 |
| 9 | 调度积压 | kube-state-metrics 统计 Pending Pod | `kube_pod_status_phase` | Pending Pod `> 5` 持续 5m | **WARNING** | 实时 |

> **注：** 问题 7 的用户通知仍然建议在 CI 流程内完成（自动 PR 评论），而 INFO 级别邮件仅作为运维侧统计参考，机器人可按标题规则静默处理或转发至内部群。
>
> **PromQL 实现提示：** 问题 7 使用 `kube_pod_container_status_terminated_reason{reason="OOMKilled"}` 做增量检测时，该指标为 counter 类型，需使用 `increase()` 函数并注意 counter 重置场景。表达式示例：
> ```promql
> increase(kube_pod_container_status_terminated_reason{reason="OOMKilled"}[10m]) > 0
> ```
>
> **数据持久化说明：** 本文档未包含长期存储方案。中心 Prometheus 默认保留 15 天数据，如需更长保留期或跨集群长期存储，可在后续引入 Thanos 或 VictoriaMetrics 作为附加层，但不影响当前架构的核心逻辑。

---

## 六、扩展新集群

加入监控体系仅需：

1. 在新集群部署标准化采集组件包（kube-state-metrics, node-exporter, Pushgateway, CronJob 集）。
2. 确保中心 Prometheus 能访问新集群的 Pushgateway 及 exporter 端点。
3. 在中心 Prometheus scrape 配置中新增对应 job，附带 `cluster` 标签。

告警规则自动覆盖新集群，邮件标题自动携带集群标识。

---

## 七、运维配套

### 7.1 维护窗口告警静默

计划内维护（集群升级、节点重启等）会触发预期内的告警，需提前配置 Alertmanager Silence 或 Prometheus 抑制规则，避免告警风暴。Silence 可通过 Alertmanager API/UI 按 `cluster` + `alertname` 匹配创建，设定起止时间，维护结束后自动失效。

### 7.2 告警去重与分组

Alertmanager 的 Gossip 集群（见 2.1 节）本身保证同一告警不会重复发送。此外应配置 `group_by`（按 `cluster` + `alertname` 分组）和 `group_interval`，将同一分组的多个 firing 实例合并到一封邮件中，避免邮件刷屏。

---

## 八、总结

本设计围绕**多集群统一监控、统一邮件出口、外部机器人分发**三大原则，以轻量采集、集中告警、结构化标题的方式，实现了：

- 基础架构团队从"被动响应"转向"主动发现"的运维模式转变。
- 监控系统自身不依赖复杂的通知渠道配置，保持简单稳定。
- 与办公软件机器人的解耦，让告警路由可自由定制且易于管理。

实施团队可按阶段落地：**先搭建中心监控集群，再逐个接入业务集群，最后对接机器人完成闭环通知。**
