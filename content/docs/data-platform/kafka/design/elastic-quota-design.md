---
title: "Elastic Quota Design"
description: "Kafka 弹性配额的保底、权重、上限与动态流量控制设计。"
---

## 弹性配额（Elastic Quota）设计文档

**状态**: 设计中
**关联 Review**: （开发完成后填写）
**调研输入**: `plans/quota-flow-control-survey.md`

### 1. 概述

为应对线上热点事件（topic 流量突增数倍）和消费实例扩容（新消费者追历史数据）导致的 broker 带宽打满、集群不稳定、核心业务受损问题，在现有静态配额体系之上引入**动态流量限制**（仅治理非内部 topic 的客户端流量）：每个配额实体配置 **reservation（保底速率）/ weight（权重）/ limit（上限）**，实体的实际限速值（dynamicLimit）由 broker 上的控制环在 **[有效保底, 上限]** 区间内周期性调整（默认每秒重算）——broker 空闲时上浮（实体可使用空闲带宽），争用时下压（回收借用的带宽），但永不低于有效保底（保底以内永不限流）、永不高于上限。broker 级容量池配置定义可分配总量，防止带宽打满。生产与消费两个方向均为**双锚点**：业务身份稳定时按 client-id 配置（生产 = 业务线 client-id，消费 = group，依赖本仓库 Client ID == Group ID 一致性校验），否则配在 topic-partition 上（详见 §3.2）。控制环只对**实际正在运行**的实体做保底核算与分配（流量阈值三态判定，详见 §3.3.2 Step 0）。

### 2. 背景

#### 2.1 现有架构

现有配额链路（详见调研报告第 2 章）：

- `ClientQuotaManager`（`core/src/main/scala/kafka/server/ClientQuotaManager.scala`）按实体维护滑动窗口 `Rate`（默认 11 样本 × 1s），produce 在 `KafkaApis.scala:691`、fetch 在 `KafkaApis.scala:1039` 逐分区记账（4 参数重载已携带 topicPartition 字符串）；
- 超限后按 `QuotaUtils.throttleTime = (value - bound)/bound × windowSize` 计算节流时间，响应携带 `throttle_time_ms` 并 mute channel（`ThrottledChannel.scala`）；
- 配额值查找走 `DefaultQuotaCallback` 的 11 级优先链（`ClientQuotaManager.scala:624-695`），本仓库已扩展 topic-partition 实体（`plans/topic-partition-quota-design.md`）；
- 配额变更经 KRaft `ClientQuotaControlManager` / ZK `DynamicConfig` 下发，最终调用 `ClientQuotaManager.updateQuota` 与 `updateQuotaMetricConfigs` 更新 MetricConfig。

**局限性**：
1. 只有静态上限，无保底——broker 过载时无法优先保障核心业务；
2. 非工作保全——broker 空闲时客户端也不能超过自身配额，配额设小掐死热点、设大打爆集群，两难；
3. 无 broker 总容量概念——所有配额之和可远超物理带宽，无法防止带宽打满；
4. 限速值静态——只回答"你超没超自己的配额"，不回答"broker 还有没有空闲"。

#### 2.2 需求说明

**目标**：
1. 核心业务（指定的 topic / 消费组）配置保底速率，broker 争用时优先保障，极端情况下业务不失败（减速可接受，失败不可接受）；
2. 资源充足时任何实体可弹性使用空闲容量（热点突增尽量放行、catch-up 尽快追赶）；
3. broker 总带宽有容量池包络，防止带宽打满导致集群不稳定；
4. 单机保底总和超过容量时（leader 迁移/宕机导致），**按配置比例缩放各实体有效保底**，全员降级但全员存活，同时告警触发人工协调（迁 leader / 扩容）。

**约束条件**：
- 不修改客户端协议（Fetch RPC 不携带 group 字段，客户端异构）；消费组维度由 client-id 代理（依赖已有 Client ID == Group ID 一致性校验；未接入校验的客户端按 client-id 各自成回退实体兜底，见 §3.2）；
- 部分业务 group 随机（每实例一 group、名称与启动时间/IP 关联），无法按 client-id 预配置——消费方向支持以 topic-partition 为配置锚点（§3.2），沿用本仓库 TP 消费限速的既有思路；
- 新功能必须有开关且默认关闭，关闭时行为与现状完全一致（仓库审查规范 §3.3）；
- 弹性流控仅治理**非内部 topic**：开关开启时，内部 topic（`__` 前缀）分区的流量绕过弹性记账与限速（不计入容量池、不受 overlay / 新客限速约束），内部链路（AutoBalancer 指标上报等）天然不受影响。现有代码并不存在按 client-id 前缀豁免的机制（`INTERNAL_CLIENT_ID_PREFIX` 仅用于命名内部 producer），不引入；
- 容量配置为"可分配给客户端流量"的净值，复制流量不受本功能管控（运维设置容量时预留复制预算，必要时用 KIP-73 复制配额约束）。

**本期不支持的场景**：
- 跨 broker 集群级配额聚合（给业务方承诺集群级 SLA 数字）——留二期；带宽保护是 per-broker 资源，per-broker 视角已满足过载保护目标；
- `client-id + topic-partition` 组合粒度（现有实体模型禁止该组合，维持不变）；
- KIP-392 follower fetch（消费流量假定落在 leader）；
- `request_percentage`（CPU 时间配额）不纳入动态调整，仅管带宽（Produce/Fetch 两个方向）；
- catch-up 冷读的自动识别（按 LEO - fetchOffset 阈值分类）——Phase 2 实现，Phase 1 由运维对已知回溯类 client-id 配置低 `quota_weight` 替代。

### 3. 设计方案

#### 3.1 概念模型：动态限速

每个配额实体三参数：

| 参数 | 配置键 | 默认值 | 语义 |
|------|--------|--------|------|
| reservation | `producer_byte_reservation` / `consumer_byte_reservation` | 0 | 保底速率，动态限速值的下界。争用时优先满足 |
| weight | `quota_weight` | 1.0 | 分配空闲容量的权重（双向共用） |
| limit | `producer_byte_rate` / `consumer_byte_rate`（**现有键，语义不变**） | 无限 | 硬上限，动态限速值的上界 |

核心不变式：

```
effRes_e  ≤  dynamicLimit_e  ≤  limit_e
（有效保底）    （动态限速值）      （静态上限）
```

**单次请求的判定逻辑与现状完全相同**——记账后比较窗口均值与限速值，超限则计算节流时间并 mute。变化只有一点：限速值由静态配置变为控制环每周期重算的 `dynamicLimit`：

- broker 空闲 → `dynamicLimit` 上浮（最高至 limit）→ 实体可超出保底使用空闲带宽；
- broker 争用 → `dynamicLimit` 下压（最低至 effRes）→ 借用被回收；
- 用量 ≤ 有效保底的实体永远碰不到限速值 → 永不限流。

**"回收"的含义**：带宽无法收回已发送的字节；回收 = 调低限速值 → 该实体后续请求的节流时间变长（mute 间隔变长）→ 客户端实际速率在秒级内滑落。全程不返回错误：produce 延迟响应 + mute，fetch 返回空响应 + mute，客户端只减速不失败。

#### 3.2 实体与 metric tags 规则

- **生产方向**：**双锚点**，按现有 11 级优先链决定归属——业务线 producer 的 client-id 稳定时按 client-id 配置；否则配在 topic-partition 上（复用现有扩展；produce 记账已携带 tp 字符串，分区随 leader 落点天然完成 per-broker 切分）；
- **消费方向**：**双锚点**，按现有 11 级优先链决定归属：
  - **client-id（= group）**：group 名稳定的业务，按组配置 reservation/weight/limit；
  - **topic-partition**：group 随机的业务（每实例一 group、group 名含启动时间/IP 等，无法按 client-id 预配置——即本仓库原有 TP 消费限速要解决的场景），弹性参数配在其消费的 topic 分区上。语义：该分区上**所有未被更高优先级配置命中的消费流量之和**共享这一个实体的额度（同一分区的多个随机 group 落入同一个桶，桶内 FCFS，不做组间公平——同一业务的实例间可接受；不同业务共用 topic 时不适用，应拆 topic 或使用稳定 group）；
  - 都未配置 → 按 client-id **各自成回退实体**参与分配（default 实体只是**参数模板**——为回退实体提供默认 w/L 取值，不是共享流量桶；在 default 实体上配置 `quota_weight` 即统一压低所有回退实体的单体权重），并始终受容量池总闸约束——**即使全部为随机 group 且零实体配置，容量池仍保证 broker 不被打满**（相对现状的本质改进：现状 default 配额是"每组各得一份"，组数无界则总承诺无界）；
- 静态配额的 11 级优先链**保持不变**：若流量命中更高优先级静态配置（如 user+client-id），仍按原规则取 tags 与 limit；user 系实体（第 1-4、6-9 级）**不注册弹性配额键**（§3.6），命中它们的流量按默认弹性参数参与分配（见下表）；
- **三参数原子解析**：reservation / weight / limit 以 `quotaMetricTags` 决定的实体为**唯一取值点**——一次解析同时取出该实体上的三个参数，某参数未配置时用该实体类型的 default 模板值、再无则用全局默认（res=0 / w=1.0 / L=∞）。**禁止**三参数各自独立走优先链（否则会出现 res 取自 TP 配置、limit 取自 client-id 配置的跨级混配）。跨来源组合（如实体显式配 L、res 来自模板）仍可能违反同实体校验，解析后统一钳制 `res_e = min(res_e, L_e)`；
- **内部 topic 豁免**：记账入口逐分区判定（produce/fetch 本就逐分区记账），`__` 前缀 topic 的分区字节不做配额记账、不参与弹性分配。现网内部 topic 无静态配额配置，豁免不产生行为差异；开关关闭时记账路径完全不变；
- 弹性配额启用后，新增 `QuotaTypes.ElasticQuotaEnabled` 位，`quotaMetricTags` 的**最终回退规则**调整为：produce manager 回退到 `("", "", topicPartition)`，fetch manager 回退到 `("", clientId, "")`（现状最终回退是 `("", clientId, "")`，produce 方向变化仅在弹性开关开启时生效）。produce 与 fetch 是两个独立的 `ClientQuotaManager` 实例，回退规则互不影响。

**归属规则（记账时由优先链一次性决定，控制环不识别客户端）**：

活跃检测的单位 = 记账的单位 = 限速的单位 = **sensor（配额实体）**。每一笔生产/消费流量恰好记入**一个**实体（不重不漏——容量池的 Σu/Σgreen 汇总因此成立，不会重复计费）；"活跃客户端"实为"活跃实体"，TP 实体的活跃判定只回答"该分区是否有流量"，不关心背后有几个客户端：

| 配置情况（按优先链顺序判定） | 该笔流量归属 |
|---------|-------------|
| 命中 user 精确系配置（第 1-4 级：user+client-id / user+TP / user+client-default / user，**优先于 TP**） | 该 user 系实体——**不支持弹性参数**（§3.6），按默认弹性参数（res=0、w=1.0、L=其静态限额）参与分配。目标场景不以 SASL user 作业务身份；若集群已有 user 系配额，注意其捕获的流量不受 TP/client-id 上的弹性配置治理 |
| 所在分区有 TP 配置（第 5 级） | TP 实体——**捕获该分区其余全部流量，包括有纯 client-id 配置的客户端**（纯 client-id 为第 10 级，本设计不调整链序：改序会改变存量 TP 配额语义，并触碰 quotaLimit/quotaMetricTags 并行同步审查项） |
| 命中 user-default 系配置（第 6-9 级） | 同 user 精确系：不支持弹性参数，按默认弹性参数参与分配 |
| 命中纯 client-id 配置（第 10 级） | 该 client-id 实体 |
| 都未配置 | 回退实体：消费按 client-id、生产按 topic-partition **各自成实体**（默认参数 res=0、w=模板值（TP 无模板则 1.0）、L=∞；default 实体是参数模板而非共享桶，每个回退实体独立记账、独立受限） |

示例——TP 参数配在 `T1-0`、client-id 参数配在 `group-A`：group-A 消费 T1-0 → TP:T1-0 桶；group-A 消费 T2-0（无 TP 配置）→ group-A 桶；随机组 rnd-x 消费 T1-0 → TP:T1-0 桶；rnd-x 消费 T2-0 → rnd-x 回退桶。控制环看到的活跃实体集合即 {TP:T1-0, group-A, rnd-x}。

**运营约束**：TP 配置按分区聚合整个分区的流量，桶内不区分客户端个体（排障用 BrokerTopicPartitionMetrics 等现有指标）；稳定组业务与随机组业务尽量不共用 topic，共用时接受该 topic 按 TP 粒度整体治理。

> **开发注意**：`DefaultQuotaCallback.quotaLimit` 与 `quotaMetricTags` 是并行维护的两套分支逻辑，本次修改回退规则与注入 overlay 时必须逐行同步（仓库审查必查项，`docs/review/review-guide.md` §2.1）。

#### 3.3 Phase 1：动态限速控制环（本期实现）

**原则：不改热路径判定语义**。记账/超限/mute 全部沿用现状，唯一变化是限速值由控制环动态计算并注入。新增 `ElasticQuotaController`（每 broker 一个后台线程，produce/fetch 两个方向独立计算）。

##### 3.3.1 符号与输入

| 符号 | 含义 | 来源 |
|------|------|------|
| `C_raw` | 该方向容量配置 | `elastic.quota.{produce\|fetch}.capacity.bytes.per.second` |
| `s` | 安全系数 | `elastic.quota.safety.ratio`，默认 0.9 |
| `C` | 可分配容量 = `C_raw × s` | 计算值 |
| `E` | 本周期 Active 实体集合 | 三态判定见 Step 0：只有实际正在运行的实体参与保底核算与分配（业务线/group 可申请很多、同时运行少是常态） |
| `res_e` | 实体配置的保底 | 新增配额键；与 tags 实体**原子解析**（§3.2）：实体值 → 同类型 default 模板 → 0（TP 无模板，沿用现有禁止），解析后钳制 ≤ L_e |
| `w_e` | 实体权重 | 新增配额键；同上原子解析：实体值 → default 模板 → 1.0 |
| `L_e` | 实体静态上限 | 同一次解析中取该实体的 `*_byte_rate`，未配置为 ∞ |
| `u_e` | 实体观测速率 | 实体现有 `byte-rate` 配额指标（窗口均值），控制环只读 metrics registry，**不新增热路径记账** |
| `throttled_e` | 实体上一窗口是否发生节流 | 实体现有 `throttle-time` 配额指标（窗口均值 > 0 即视为受限），控制环只读 |
| `g` | 最低授予 | `elastic.quota.min.grant.bytes.per.second`，默认 1MB |
| `ε` | 活跃判定阈值 | `elastic.quota.active.threshold.bytes.per.second`，默认 1024（1KB/s） |
| `K` | 活跃判定记忆周期数 | `elastic.quota.active.window.intervals`，默认 10（记忆时长 = K × 周期 ≈ 10s，与 byte-rate 滑动窗口同量级；迟滞，防间歇业务抖动） |
| `H` | 新客/待命限速基准 = `C_raw × (1 - s)` | 派生值（容量安全余量），无独立配置 |

##### 3.3.2 每周期计算步骤

每 `refresh.interval`（默认 1s）对每个方向执行：

**Step 0 实体状态判定（三态分类）**

依据实体现有 `byte-rate` 指标（窗口均值，停发后约一个窗口 ~11s 内归零）分类：

| 状态 | 判定 | 本周期待遇 |
|------|------|-----------|
| Active | 近 K 个周期内任一周期 u_e > ε | 全额参与 Step 1-7 分配 |
| Standby | 有 sensor 但近 K 个周期 u_e ≤ ε | 不参与分配（保底**不计入 R**、不占 spare）；overlay 写入待命限速 `clamp(max(res_e, H), 0, L_e)`——唤醒瞬间最多用到自己的保底或安全余量，下周期转 Active 拿全额 |
| Unknown | 无 sensor（新客户端 / sensor 已过期） | 首次请求创建 sensor；overlay 未命中时按 `max(res_e, H)` 约束（Step 7 未命中规则，与 Standby 待遇一致），下周期进入分类 |

设计意图：业务线/group 可以申请很多，但同时运行的通常只有几个——**只有 Active 实体参与保底核算**，否则闲置配置会无谓压低运行中实体的 effRes。H（容量安全余量）的本职就是兜住"控制环还不认识的流量"；未知/待命实体若配有保底则待遇取 `max(res_e, H)`——保底实体在冷启动/重启/sensor 过期后**立即**受保底保护，不必等控制环认知；K 周期记忆是迟滞，防间歇型业务在两态间抖动。可选优化：sensor 创建事件触发一次去抖的立即重算，把新客认知延迟压到亚秒（实现阶段决定）。

**Step 1 有效保底（超卖等比缩放）**

```
R = Σ_{e∈Active} res_e            // 只统计 Active 实体（Standby 的保底不预留，见 Step 0/3）
scale = min(1, C / R)
effRes_e = res_e × scale
```

缩放**立即生效**（容量保护优先，不做迟滞）；"超卖状态"的告警指标需连续 `overcommit.confirm.intervals` 个周期确认后才切换（迟滞只作用于告警，不影响计算），防 leader 抖动导致告警翻转。

**Step 2 需求估计（每周期全量重算，无历史状态）**

```
demand_e = L_e            若 throttled_e    // 真实需求被限速截断、不可观测：按上限申报，实拿多少由 Step 4 注水裁决
demand_e = max(u_e, g)    否则              // 未受限实体的观测速率即其需求
```

受限实体申报 `L_e`（未配置时为 ∞）不会击穿容量：claim 无界只意味着注水时不主动出局，实际获配始终是加权份额、总量恒等于 spare。`g` 保证低流量/新实体的起步额度（软性：只抬高需求申报，不是硬保证）。需求每周期重算、**无爬升上限**——受限实体最快一个周期（1s）即拿到加权公平份额；代价是"申报过冲 → 回落"的轻微振荡（见 3.3.4），1s 周期下表现为秒级速率纹波，评审裁定可接受，不引入平滑。

**Step 3 保底占用与可分空闲**

```
green_e = min(effRes_e, demand_e)   // 实体预计落在保底内的用量
spare   = C - Σ_{e∈E} green_e       // 其余全部可分
```

关键点：保底**不做永久预留**。实体没用满的保底（`demand_e < effRes_e`）和闲置实体的保底自动留在 `spare` 里分给别人——这就是工作保全。代价是"闲置保底实体突然唤醒"存在一个周期的瞬态（见 3.3.4）。

**Step 4 加权注水分配空闲**

```
claim_e = max(0, min(demand_e, L_e) - green_e)   // 实体超出保底的需求（封顶静态上限）
S = { e | claim_e > 0 }，剩余 = spare
循环直至 剩余 = 0 或 S = ∅:
    对每个 e∈S: 份额_e = 剩余 × w_e / Σ_{S} w
    若 份额_e ≥ claim_e: extra_e = claim_e，e 移出 S（吃不完的份额留给下一轮）
    否则: extra_e = 份额_e，全部分完，循环结束
```

实现上按 `claim_e / w_e` 升序排序后单遍扫描即可（O(n log n)），claim 无界（受限申报且 L=∞）的实体排在最后、按权重分摊剩余，不需要真的迭代。

**Step 5 剩余空闲摊作突发余量**

若注水后仍有剩余（所有 claim 均已满足、总需求 < C），按权重摊给全部活跃实体（封顶 `L_e`）记作 `headroom_e`——让需求增长在两个周期之间就能兑现，不必等控制环。

**Step 6 定值与钳制**

```
dynamicLimit_e = clamp(green_e + extra_e + headroom_e,  下界 effRes_e,  上界 L_e)
```

下界钳制保证"保底内永不限流"：即使实体当前需求低于保底，限速值也不低于 `effRes_e`，随时可以无阻涨回保底。不做周期间平滑——限速值直接取当期计算结果（评审裁定：平滑系数非必要参数，1s 周期下直接重算即快速收敛）。

**Step 7 注入生效**

写入 `dynamicLimits` overlay（`ConcurrentHashMap[metricTags → limit]`）：Active 实体写分配值，Standby 实体写待命限速。`DefaultQuotaCallback.quotaLimit` 末端规则：overlay 命中 → 取 `min(静态链结果, overlay 值)`；未命中且该方向启用 → 取 `min(静态链结果, max(res_e, H))`，其中 res_e 按 §3.2 与静态链同实体原子解析——配了保底的实体在 sensor 过期/broker 重启后的冷启动期即受保底保护而非被压到 H，该式与 Standby 待命限速一致（Unknown 与 Standby 待遇统一）；方向未启用 → 仅静态链。并复用现有 `updateQuotaMetricConfigs` 机制刷新已创建 sensor 的 MetricConfig。overlay 读路径为无锁查询，弹性开关关闭时恒为空、直接短路。

##### 3.3.3 数值示例

**示例 1（常态分配，消费方向）**：`C = 900`（1000 × 0.9）。三个消费组，A 上一窗口已被节流（throttled），B/Cbf 未受限：

| 实体 | res | w | L | 观测 u | 受限 | demand | green | claim |
|------|-----|---|---|--------|------|--------|-------|-------|
| B 核心组 | 300 | 1 | ∞ | 200 | 否 | 200 | 200 | 0 |
| A 热点组 | 0 | 1 | 800 | 600 | 是 | 800（=L） | 0 | 800 |
| Cbf 回溯组 | 0 | 0.1 | ∞ | 100 | 否 | 100 | 0 | 100 |

`spare = 900 - 200 = 700`，按权重 1 : 0.1 注水：A 份额 700 × 1/1.1 ≈ 636 < claim、Cbf ≈ 64 < claim，一轮分完。结果：

```
dynamicLimit: B = max(200, 300) = 300   // 下界钳到保底
              A = 636                    // 借走大部分空闲（仍受限 → 下周期继续按 L 申报）
              Cbf = 64                   // 低权重被压到最低
预计总用量 = 200 + 636 + 64 = 900 = C ✓
```

**示例 2（争用回收与收敛）**：接示例 1，下周期 B 流量涨到 300 触顶被节流 → 按 L = ∞ 申报，green = 300（吃满保底），`spare = 600`；A 仍受限（claim 800）、Cbf 未受限（demand 100，claim 100），权重 1 : 1 : 0.1。B 的 claim 无界不出局，一轮分完：B ≈ 286、A ≈ 286、Cbf ≈ 29。结果：

```
dynamicLimit: B = 300 + 286 = 586,  A = 636→286,  Cbf = 64→29
```

B 需求上升后，A/Cbf 的借用**在一个周期内被自动收回**——回收就是限速值重算，没有任何"收回已发字节"的动作。再下一周期，若 B 实际只用到 350（未再触顶）→ 按 u = 350 申报：green 300、claim 50，B 份额 ≥ 50 先出局，剩余按 1 : 0.1 分 → B = 350、A ≈ 500、Cbf ≈ 50——受限申报的"过冲"在一个周期内自动回落，全程约 2 个周期（~2s）收敛（申报-回落纹波见 §3.3.4）。

**示例 3（超卖等比缩放，生产方向）**：`C = 1000`（示例取整，忽略安全系数），topic 分区 A、B 保底各 600，因 leader 迁移同落一台：`R = 1200 > 1000` → `scale = 0.833` → 有效保底各 500。两者需求都 ≥ 500 时：green 各 500，`spare = 0`，`dynamicLimit` 各 500——等比降级、全员存活，触发超卖告警，等待人工协调或 AutoBalancer 重摊。

**示例 4（保底虚高：纸面超卖但实际空闲）**：`C = 1000`。X（res 800，实际 u 100）、Y（res 600，u 50）——声明远高于实际，均未受限；Z（res 0，w 1，受限 → 按 L = ∞ 申报）想借空闲：

- Step 1：R = 1400 > 1000 → scale ≈ 0.71 → effRes X ≈ 571、Y ≈ 429，超卖告警触发（纸面比 1.4）；
- Step 2/3：X/Y 未受限按观测申报（demand 100 / 50），green = min(effRes, demand) → X 100、Y 50；`spare = 1000 − 150 = 850`；
- Step 4：唯一 claimant Z 分得 850。

结果：`dynamicLimit` X = 571（钳到有效保底，是"地板"而非"占用"）、Y = 429、Z = **850**——纸面超卖 40% 的节点上 Z 仍可用 85% 容量。**未使用的保底不占池子**（Step 3 按需求计 green），超卖状态只是承诺风险信号，不影响空闲分配。若 X 随后猛增：当周期内可瞬时冲到 571（其地板），总量短暂超 C；下周期 X 受限按上限申报、green 吃满 571，spare 收缩至 379、X 与 Z 按权重均分 → Z 被回收（850 → ~190）。

##### 3.3.4 边界情况与瞬态

| 场景 | 行为 | 说明 |
|------|------|------|
| 新客户端首次出现（Unknown） | 至下一周期受 `max(res_e, H)` 约束；多个新客同时出现时瞬态 ≈ Σmax(res, H) | H 即安全余量，兜底与限幅一体；配保底的实体冷启动即恢复保底待遇；可选 sensor 创建事件触发即时重算 |
| Standby 保底实体唤醒 | 保底未被预留（工作保全），唤醒当周期最多用到 max(res, H)，总量可能短暂超 C（幅度 ≤ 该保底值） | 安全系数吸收 + 下周期转 Active、回收他人借用；要求恒预留可调低 `safety.ratio` 等效 |
| 间歇型业务（低频突发） | u 在窗口间归零，可能在 Active/Standby 间来回切换 | K 周期迟滞记忆；仍抖动则调大 K 或将 ε 置 0 |
| 保底虚高（声明 ≫ 实际用量） | 不阻塞他人借用（Step 3 按需求计 green，未用承诺不占池，见示例 4）；代价：①声明者可瞬时冲到有效保底，Σ(effRes − green) 构成潜在瞬态敞口；②真争用时承诺仍全额兑现，挤压无保底业务 | `reservation-utilization` 指标暴露虚高 + 运营定期修剪（治理问题，算法不自动打折） |
| 周期内需求突增 | 实体受当前 `dynamicLimit` 约束，超出部分被现有机制节流；下周期按受限申报参与注水 | 突发兑现延迟 ≤ 1-2 周期（1s 级），Step 5 突发余量进一步缓解 |
| 申报-回落振荡（受限实体） | 受限按 L 申报 → 获配可能高于真实需求 → 下周期转未受限、按 u 申报 → 配额回落，表现为秒级速率纹波 | 幅度受注水权重份额约束，1-2 周期收敛（示例 2）；评审裁定删除平滑换取算法简单，实测振荡过大再评估迟滞 |
| 多实体请求连坐节流 | produce/fetch 单请求跨多分区/多实体时取 max throttle，fetch 超限还会整响应置空——一个受限实体压制整个请求 | 现状机制固有（非弹性引入），动态下压使触发更频繁；1s 控制环把误压制窗口压到秒级；运营建议高优业务的 producer/订阅不与受限实体混在同一请求流 |
| 大量实体 × minGrant > spare | `g` 是软性的（只影响需求估计），注水时份额自然缩小 | 不会击穿容量 |
| leader 频繁切换 | 缩放立即生效保护容量；告警迟滞确认防状态翻转 | 限速值每周期全量重算，切换后 1-2 个周期收敛；迟滞只作用于告警、不作用于计算 |
| 大量随机 group（每实例一组） | 每个实体默认 w=1，N 个实体合计 N 份权重，稀释核心实体份额 | 广播型业务锚定 TP（实体数 = 分区数，有界）；给 default client-id 配小 `quota_weight`（如 0.1）统一压低回退实体单体权重。生产方向回退实体为 TP 粒度（数量 ≤ 本机 leader 分区数、天然有界；TP 无 default 模板，回退权重固定 1.0，需压低时对分区显式配置） |
| 随机 group 追历史与稳态消费同分区 | 两者共享同一 TP 实体额度，Phase 1 无法区分（配额层拿不到 lag）；好处：追历史撑不破该分区预算，伤不到其他实体 | Phase 2 lag 分类划入低权重；Phase 1 应急手段为临时调低该分区 limit |
| overlay 条目生命周期 | 与 sensor 过期对齐（1h 不活跃清理） | 防泄漏 |

##### 3.3.5 性能分析与预算

**热路径（每请求）：零新增开销。**
- 判定逻辑不变（窗口均值 vs bound）；动态限速值存放在 sensor 的 MetricConfig 中，与现状一致；
- overlay **不在每请求路径上**——仅在 sensor 首次创建与控制环刷新两个低频时机被查询；
- `quotaResetRequired` 保持返回 false（控制环主动推送更新），不产生每请求回调；
- 开关关闭时全部短路，与基线逐字节一致（验证方案场景④对比）。

**控制环（每 1s，单后台线程，每方向）：**

| 步骤 | 复杂度 | n=1,000 实体估算 | n=10,000 实体估算 |
|------|--------|-----------------|------------------|
| 指标快照 | O(n)（按实体索引点读，不扫 registry） | ~0.1ms | ~1ms |
| 三态分类 + 缩放 + 注水 | O(n log n)（一次排序 + 单遍扫描，纯算术） | <0.5ms | ~5ms |
| MetricConfig 刷新 | O(变更数)（仅刷新变化超阈值的条目） | 稳态趋近 0 | 同左 |

1 万活跃实体规模下每周期约几毫秒，1s 周期下单核占用 <1%（单次成本与在跑的 AutoBalancerMetricsReporter 单轮上报同量级，频率更高）。实体数有天然上界：活跃 TP 实体 ≤ 本机 leader 分区数，活跃 client-id ≤ 同时运行的组数。

**两个真实风险与对策：**
1. **禁止全量扫 metrics registry**（broker 指标总数可达数万至数十万）：控制环维护自己的活跃实体索引（sensor 创建/过期时增删），快照按索引点读，成本只与配额实体数相关；
2. **写锁持有时间**：现有 `updateQuotaMetricConfigs` 在 ClientQuotaManager 写锁内更新 MetricConfig，热路径取 sensor 走读锁，每周期刷新数千条会阻塞请求线程；1s 周期使刷新频率较 5s 方案放大 5 倍，**变更阈值过滤因此是必要项而非优化项**。对策：①仅刷新变化超过阈值（默认 5%，是否暴露为配置实现阶段定）的条目——稳态下动态限速几乎不变，变更数趋近 0；②分批刷新、批间释放锁；③必要时改为无锁更新 + 最终一致（sensor 以略旧 bound 创建，下周期校正）。实现阶段以基准测试定案。

**验收基准（硬性）：**
- 控制环单周期耗时：10,000 实体 < 50ms（微基准）；
- 开启 vs 关闭：produce/fetch p99 延迟与吞吐差异在测量噪声内（集成测试场景④）；
- 单次写锁持有 < 1ms。

#### 3.4 超卖处理与人工协调闭环

- 超卖判定：本机活跃实体 `Σreservation > C`（leader 迁移/宕机导致）；
- 处理：Step 1 等比缩放（配置值不变，只缩"本机有效承诺"）；缩放的是承诺不是分配，实体用不满的保底经 Step 3 自动流给他人，不浪费；
- 超卖仅意味着**承诺总和**超过容量，不等于实压：只有当保底实体实际用满其有效保底时 `spare` 才趋于 0（借用停摆）；声明高但实压低时，空闲仍可被他人借用（示例 4）。告警须结合 `green-usage-ratio` 分级：**纸面超卖 + 低实压 = 治理级**（修剪虚高保底），**超卖 + 高实压 = 立即协调**（迁 leader / 扩容）；
- **人工协调触发器**：超卖比指标 + 进入/退出日志（见 3.7），运维据此迁 leader / 扩容；AutoBalancer 重摊 leader 后自动退出超卖态。

#### 3.5 Phase 2：逐请求精确执行（后续阶段，此处仅立框架）

**动机**：Phase 1 的保底是"软保底"——控制环周期内的瞬时突发可能短暂挤占，回收粒度为周期（秒级）。若实测不满足，Phase 2 把判定下沉到单请求。

**机制**：实体保底桶（速率 = `effRes_e`）+ 每方向一个共享容量桶（速率 = C），底层复用 KAFKA-10364 的 `TokenBucket`。单请求判定：

```
输入: 实体 e, 本次字节数 B
1. e 的保底桶扣减 B 成功（含 burst 透支）
     → 放行不限流。容量桶同步记账（允许扣成负数——保底优先于一切）
2. e 已达静态上限 L_e
     → 按现有公式节流
3. 否则为借用: 从共享容量桶扣 B:
     有令牌 → 放行（借用空闲）
     桶空   → 节流, throttle ≈ 桶赤字/C（KIP-599 透支式公式）
```

借用回收的微观过程：保底流量上升 → 共享容量桶消耗加快 → 借用请求扣不到令牌 → 借用方 throttle 变长 → 其速率滑向自身保底，秒级自动均衡。逐请求抢容量桶为 FCFS（近似按需分配），**权重语义仍由控制环承担**：Phase 2 = 控制环（权重塑形软份额）+ 保底桶（硬保底）+ 容量桶（硬总闸）三层叠加，控制环不废弃。

其余规划：catch-up 冷读自动分类（fetch 时以 `LEO - fetchOffset > 阈值` 标记，归入低权重类）；热路径锁与性能分析、混合升级方案届时单独评审。Phase 1 的配置模型、容量池、控制环产物全部复用。

**produce/fetch 精度不对称**：fetch 的判定在读数据之前，不放行 = 返回空响应（复用 `KafkaApis.scala:1050` 的 unrecord 机制），出向字节真实未发出，管控精确；produce 判定时字节已进入 broker（入向已消耗），单次请求物理上总是放行，判定结果只决定 mute 时长（压制后续速率），入向保护天生滞后一拍（KIP-13 选延迟不选拒绝的同因）。开发与验收时不应期望 produce 逐字节精确拦截。

#### 3.6 配置清单

**broker 级（新增，`KafkaConfig` 注册，支持动态更新）**：

| 配置键 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `elastic.quota.enable` | boolean | **false** | 总开关，关闭时所有新逻辑不生效 |
| `elastic.quota.produce.capacity.bytes.per.second` | long | -1 | 生产方向容量池，-1 = 该方向不启用 |
| `elastic.quota.fetch.capacity.bytes.per.second` | long | -1 | 消费方向容量池，-1 = 该方向不启用 |
| `elastic.quota.safety.ratio` | double | 0.9 | 容量安全系数，对冲控制环周期内突发 |
| `elastic.quota.refresh.interval.ms` | long | 1000 | 控制环周期；连坐节流误压制窗口与周期同阶，默认 1s 压到秒级 |
| `elastic.quota.min.grant.bytes.per.second` | long | 1048576 | 新/低流量实体最低授予 g（软性） |
| `elastic.quota.active.threshold.bytes.per.second` | long | 1024 | 活跃判定阈值 ε（u_e 超过即视为运行中） |
| `elastic.quota.active.window.intervals` | int | 10 | 活跃判定记忆周期数 K（迟滞）；记忆时长 = K × 周期 ≈ 10s |
| `elastic.quota.overcommit.confirm.intervals` | int | 3 | 超卖**告警状态**进入/退出的连续确认周期数（不影响缩放计算） |

**实体级（新增配额键，仅注册到 client-id / topic-partition ConfigDef；user 系实体不支持弹性参数，见 §3.2 归属规则）**：

| 配置键 | 类型 | 适用实体 | 说明 |
|--------|------|---------|------|
| `producer_byte_reservation` | long | client-id、topic-partition | 生产保底；业务线 client-id 稳定时配组上，否则配分区上 |
| `consumer_byte_reservation` | long | client-id、topic-partition | 消费保底；随机 group 业务配在 TP 上 |
| `quota_weight` | double | 上述实体及 default client-id | 权重，默认 1.0；default 实体是回退实体的参数模板，在其上配置可统一压低随机长尾 |

配置校验：同一实体上 `reservation ≤ limit`（若两者均配置）；`quota_weight > 0`；`safety.ratio ∈ (0, 1]`；capacity 为 -1（该方向禁用）或正数。运行期兜底：三参数原子解析后统一钳制 `res_e = min(res_e, L_e)`（跨来源组合场景，§3.2）。

配置示例：

```bash
# 核心 topic 分区生产保底 50MB/s
kafka-configs.sh --alter --add-config 'producer_byte_reservation=52428800' \
  --entity-type topic-partitions --entity-name core-topic-0

# 核心消费组（client-id=group）消费保底 100MB/s，上限 300MB/s
kafka-configs.sh --alter --add-config 'consumer_byte_reservation=104857600,consumer_byte_rate=314572800' \
  --entity-type clients --entity-name core-group

# 回溯类消费组低权重（争用时优先被回收）
kafka-configs.sh --alter --add-config 'quota_weight=0.1' \
  --entity-type clients --entity-name backfill-group

# 随机 group 业务（每实例一组的广播消费）：在其消费的 topic 分区上配保底与上限，
# 对该分区所有未命中更高优先级配置的消费流量整体生效，无需知道 group 名
kafka-configs.sh --alter --add-config 'consumer_byte_reservation=52428800,consumer_byte_rate=209715200' \
  --entity-type topic-partitions --entity-name broadcast-topic-0

# 压低随机长尾 group 的集体权重
kafka-configs.sh --alter --add-config 'quota_weight=0.1' \
  --entity-type clients --entity-default
```

#### 3.7 可观测性

| 指标（每方向） | 类型 | 说明 |
|------|------|------|
| `elastic-quota-capacity` | Gauge | C = 配置容量 × safety.ratio |
| `elastic-quota-usage` | Gauge | 活跃实体观测速率之和 |
| `reservation-overcommit-ratio` | Gauge | Σ(活跃实体 reservation) / C，>1 即超卖，**告警项**（驱动缩放的口径） |
| `reservation-promised-ratio` | Gauge | 生产方向：本机 leader 分区**已配置**保底总和 / C（含闲置实体，暴露潜在超卖风险） |
| `elastic-quota-overcommit-state` | Gauge | 0/1，超卖告警状态（经迟滞确认） |
| `green-usage-ratio` | Gauge | Σgreen / C，保底**实压**水位；与 overcommit-ratio 组合区分"纸面超卖但空闲"（治理级）与"超卖且实压"（立即协调） |
| `reservation-utilization` | Gauge（实体 tags，仅配置了 res 的实体） | u_e / res_e，保底利用率；长期偏低即保底虚高，作为运营修剪依据 |
| `dynamic-limit` | Gauge（实体 tags） | 实体当前动态限速值，随 sensor 生命周期创建/过期 |
| `elastic-quota-refresh-age-ms` | Gauge | 距控制环上次成功完成刷新的毫秒数；> 3 × 周期即告警（控制环卡死/线程退出监测，见 §3.9） |

日志：超卖进入/退出 INFO（含缩放系数与受影响实体数）；每轮分配摘要 DEBUG。

#### 3.8 兼容性

- `elastic.quota.enable=false`（默认）：控制环不启动、overlay 恒空、tags 回退规则不变，行为与现状逐字节一致；
- 仅新增配置键，无 RPC schema / 元数据 record 变更，不做 MetadataVersion 门控（与 topic-partition 配额先例一致）；**升级顺序要求文档化**：集群全部升级到本版本后才允许配置新键（旧 controller 会拒绝未知键，fail-safe）；
- 现有静态配额语义不变：`producer/consumer_byte_rate` 仍是硬上限，动态限速值恒 ≤ 静态上限；
- **监控口径变化（仅弹性开启时）**：produce 方向未命中任何静态配置的流量，其 tags 由 `("", clientId, "")` 变为 `("", "", topic-partition)`，对应 byte-rate / throttle-time 配额指标的实体身份随之改变，依赖旧口径聚合的监控看板需同步调整；内部 topic 分区流量不再做配额记账（现网内部 topic 无配额配置，无实际行为差异）；
- ZK 与 KRaft 两种模式均支持（改动点对齐 TP 配额先例）。

#### 3.9 风险与缓解

| 风险 | 影响 | 缓解措施 |
|-----|------|---------|
| 控制环滞后，周期内突发瞬时打满 | 短暂过载 | safety.ratio 余量 + Step 5 突发余量 + 1s 短周期（必要时可再缩短） |
| 闲置保底实体唤醒瞬态 | 单周期超 C（幅度 ≤ 该保底） | 安全系数吸收，下周期回收；可调低 safety.ratio 等效恒预留 |
| leader 频繁切换导致分配震荡 | 限速抖动 | 缩放立即生效 + 告警迟滞；限速值每周期全量重算，1-2 周期收敛 |
| 控制环线程异常退出或卡死 | overlay 冻结在最后一次计算值（下界仍为 effRes，但不再跟随负载变化） | 线程设 uncaughtExceptionHandler 告警并自动重建；`elastic-quota-refresh-age-ms` 指标 + 告警兜底（§3.7） |
| overlay/配置表条目泄漏 | 内存增长 | 条目生命周期与 sensor 过期对齐（1h 不活跃清理） |
| dynamic-limit 指标基数过大 | metrics 压力 | 仅活跃实体存在指标，随 sensor 过期；沿用分区级指标懒注册经验 |
| 控制环读取指标与写 MetricConfig 的并发 | 写锁持有过长阻塞热路径读锁 | 变更阈值过滤 + 分批放锁，必要时无锁更新 + 最终一致（§3.3.5）；验收基准：单次写锁持有 < 1ms |
| 控制环遍历 metrics registry | 周期成本随全局指标数膨胀 | 维护活跃实体索引、按索引点读（§3.3.5），成本只与配额实体数相关 |
| 容量配置失真（网卡异构/复制占用波动） | 保护失效或过度限流 | 容量与用量指标可视化对照；二期考虑自适应估计 |

### 4. 关键技术决策

| 决策点 | 选择方案 | 备选方案 | 理由 |
|--------|---------|---------|------|
| 语义表述 | 直接用"动态限速"（限速值在 [保底, 上限] 区间滑动） | 引入 trTCM 三色等类比术语 | 评审意见：机制本质就是静态限速动态化，直说，不引入额外理解成本 |
| 保底执行形态 | Phase 1 控制环动态限速（软保底），Phase 2 再做逐请求判定（保底桶+容量桶） | 直接改热路径 | 先低风险验证策略与参数；不动热路径语义，可独立回滚 |
| 消费组维度 | client-id 代理 group | 改协议携带 group / 新增 group 实体 | Fetch 协议无 group 且客户端异构不可改；本仓库已有 Client ID == Group ID 一致性校验，client-id 是可靠代理 |
| 随机 group 应对 | 消费方向双锚点：弹性参数可配在 topic-partition 实体上，随机组业务锚定其消费的分区 | client-id 前缀/模式匹配（Redpanda 式）；仅靠 default 桶 | group 名随机（每实例一组/含时间戳 IP）无法预配置；topic 是稳定的业务身份，复用既有 TP 扩展与优先链、零新机制；前缀匹配需把实体模型改为非精确匹配（动优先链与配置存储），侵入大，列为后续可选 |
| 生产维度 | 双锚点：client-id（业务线）/ topic-partition | 仅 topic-partition | 生产侧业务线通常以 client-id 区分；优先链天然支持双锚点；TP 仍承接按 topic 管控与随机 client 场景，分区随 leader 落点天然 per-broker 切分 |
| 活跃实体判定 | 流量阈值 ε + K 周期迟滞（复用 byte-rate 指标，三态：Active/Standby/Unknown） | 按连接存在判定；查询 group 协调器状态；沿用 sensor 1h 过期 | "申请多、同时运行少"是常态，只算运行中实体才不压低 effRes；连接空闲 ≠ 在用且映射不到 TP 实体；协调器在其他 broker、跨机依赖且不覆盖生产侧；sensor 1h 过期粒度太粗（停机半小时仍占核算） |
| 作用范围 | per-broker（Phase 1） | 跨 broker 集群聚合 | 带宽是 per-broker 资源，过载保护不需要集群视角；集群 SLA 数字留二期 |
| 超卖策略 | 按配置比例缩放有效保底 + 告警人工协调 | 保底实体再分优先级 | 用户拍板：不引入新配置维度，降级确定可解释，全员存活；配合运维闭环 |
| 超卖缩放时机 | 缩放立即生效，迟滞只作用于告警状态 | 缩放本身做迟滞 | 容量保护优先；迟滞缩放会造成迟滞窗口内的真实过载 |
| 闲置保底处理 | 不预留，回到可分空闲（工作保全） | 恒预留全部已配置保底 | 预留浪费空闲带宽；唤醒瞬态幅度有界（≤ 该保底值），由安全系数吸收 |
| 争用信号 | 静态容量配置 × 安全系数 | 自适应容量估计（BBR/Sentinel 式） | 简单可预期；自适应留二期增强 |
| 动态限速注入 | quotaLimit 末端 min(静态, overlay) | 自定义 ClientQuotaCallback 整体替换 | 保留 11 级优先链与 KIP-257 插件接口；DefaultQuotaCallback 为私有内部类，替换需复制大量逻辑 |
| 权重粒度 | 单一 `quota_weight` 双向共用 | 分方向两个权重键 | 减少配置面；实测有需求再拆 |
| catch-up 识别 | Phase 1 人工配低权重；Phase 2 按 LEO-offset lag 自动分类 | Phase 1 即自动识别 | 配额层拿不到逐请求 lag，自动识别必须改执行层，归入 Phase 2 |
| MetadataVersion 门控 | 不加，文档化升级顺序 | 新增 MV gate | 仅新增配置键，无 schema/协议变更；旧 controller 拒绝未知键，天然 fail-safe；与 TP 配额先例一致 |
| 保底虚高处理 | 算法不打折声明值：green 按需求计已保证空闲不浪费；治理靠 `reservation-utilization` 可观测 + 运营修剪 | 按历史用量自动折扣有效保底 | 保底的意义就是为未来突发做的承诺（核心业务平时 100 也要保得住事故时的 800）；按历史打折会侵蚀保底语义，把治理问题变成语义问题 |
| 需求估计 | 每周期全量重算：未受限按观测 u_e、受限按 L_e 申报，由注水裁决实拿 | 爬升系数 h 放大观测值 + 平滑系数 α 渐进逼近 | 评审裁定：h/α 非必要，删除以简化算法与配置面（h+α 组合下 100→800 需 ~18 周期；重算 1-2 周期收敛）；"申报过冲→回落"纹波可接受 |
| 控制环周期 | 1s | 5s | 多实体连坐节流（max-throttle + fetch 空响应）的误压制窗口与周期同阶，1s 压到秒级；每周期毫秒级成本，1Hz 下 CPU <1%；写锁压力靠变更阈值过滤（§3.3.5） |
| user 维度弹性参数 | 不支持：弹性键仅注册到 client-id / TP 实体，命中 user 系静态配置的流量按默认弹性参数参与分配 | user 系全组合支持弹性键 | user 系与 TP 链序交错（精确系 1-4 级优先于 TP、default 系 6-9 级介于 TP 与纯 client-id 之间），支持它会显著复杂化双锚点归属规则；目标场景不以 SASL user 为业务身份 |
| 内部流量豁免 | 按 topic 维度：`__` 前缀分区绕过弹性记账与限速 | 按 client-id 前缀豁免 | client-id 前缀豁免机制在现有代码中不存在（`INTERNAL_CLIENT_ID_PREFIX` 仅用于命名）；内部流量的稳定标识是 topic，逐分区记账入口天然支持按分区判定 |

### 5. 实现计划

| 任务 | 涉及文件 | 状态 |
|------|---------|------|
| 新增实体级配额键（reservation/weight）与 ConfigDef、配置校验（reservation ≤ limit、weight > 0；仅注册 client-id / TP 实体） | `server-common/src/main/java/org/apache/kafka/server/config/QuotaConfigs.java` | 待开发 |
| broker 级弹性配额配置注册 | `core/src/main/scala/kafka/server/KafkaConfig.scala`（或 server-common 新增 ElasticQuotaConfig） | 待开发 |
| KRaft 配置校验接受新键 | `metadata/src/main/java/org/apache/kafka/controller/ClientQuotaControlManager.java` | 待开发 |
| KRaft 配置下发到 manager | `core/src/main/scala/kafka/server/metadata/ClientQuotaMetadataManager.scala` | 待开发 |
| ZK 模式配置支持 | `core/src/main/scala/kafka/server/DynamicConfig.scala`、`core/src/main/scala/kafka/server/ZkAdminManager.scala` | 待开发 |
| ClientQuotaManager：dynamicLimits overlay、quotaLimit 注入（含 max(res,H) 未命中路径）、tags 回退规则、三参数原子解析、updateReservation/Weight 入口、QuotaTypes 新位 | `core/src/main/scala/kafka/server/ClientQuotaManager.scala` | 待开发 |
| 记账入口内部 topic 豁免（逐分区判定，`__` 前缀分区绕过弹性记账） | `core/src/main/scala/kafka/server/KafkaApis.scala` | 待开发 |
| 新增 ElasticQuotaController（控制环：指标采集、三态分类、等比缩放、受限申报、加权注水、突发余量、指标暴露，§3.3.2 Step 0-7） | `core/src/main/scala/kafka/server/ElasticQuotaController.scala`（新文件） | 待开发 |
| controller 装配与生命周期 | `core/src/main/scala/kafka/server/QuotaFactory.scala`、`BrokerServer.scala`/`KafkaServer.scala` | 待开发 |
| 单元测试：分配算法（注水/缩放/受限申报/minGrant/示例 1-4 数值断言） | `core/src/test/scala/unit/kafka/server/ElasticQuotaControllerTest.scala`（新文件） | 待开发 |
| 单元测试：overlay 注入与开关关闭零影响 | `core/src/test/scala/unit/kafka/server/ClientQuotaManagerTest.scala` | 待开发 |
| 集成测试：多实体争用、保底保障、超卖缩放、leader 迁移重算 | `core/src/test/scala/integration/kafka/api/ElasticQuotaIntegrationTest.scala`（新文件） | 待开发 |
| 性能基准：控制环单周期耗时（1k/10k 实体）、写锁持有时间、开关开/关热路径对比（§3.3.5 验收基准） | 微基准 + 集成测试场景④ | 待开发 |
| 文档更新 | `docs/extensions.md` | 待开发 |

**验证方案（开发完成后执行）**：
1. 单测/集测按上表覆盖，其中分配算法单测直接使用 §3.3.3 示例 1-4 做数值断言；`./gradlew core:test --tests "*ElasticQuota*"`；
2. 本地 3-broker 集群实测四场景：①热点突增在空闲集群全放行；②争用时核心实体保底不受挤压、借用按权重收缩（对照示例 2）；③kill 一台 broker 触发超卖，验证等比缩放与告警指标，AutoBalancer 重摊后自动恢复；④开关关闭时与基线版本行为/性能对比无差异；
3. 对照 `docs/review/review-guide.md` 高频必查项自查后提交 review。

### 6. 变更记录

- 2026-07-03：初版。基于调研报告与设计讨论确定：三参数模型、client-id 代理 group、per-broker 范围、超卖比例缩放（不采用优先级分级）、Phase 1 控制环 / Phase 2 执行层分阶段。
- 2026-07-03：补充"回收 = 调整后续请求准入而非收回字节"、Phase 1/2 的单请求判定流程、权重由控制环承担、produce/fetch 精度不对称。
- 2026-07-03：**评审修订**——①去除 trTCM/三色类比术语，统一为"动态限速"表述（限速值在 [保底, 上限] 区间内动态调整）；②§3.3 细化控制环算法：符号定义、八步计算公式、三个数值示例（常态/回收/超卖）、边界情况表；③明确超卖缩放立即生效、迟滞仅作用于告警状态；④新增 `demand.headroom.factor` 配置与 `reservation-promised-ratio` 指标。
- 2026-07-03：**随机 group 场景修订**——消费方向明确为双锚点：client-id（稳定组）与 topic-partition（随机组业务，弹性参数配在分区上，分区内所有未命中更高优先级配置的消费流量共享实体额度，桶内不做组间公平）；default client-id 支持 `quota_weight` 压低随机长尾集体权重；边界情况新增"实体数稀释权重"与"追历史与稳态同 TP 桶（Phase 1 不可区分，Phase 2 lag 分类）"；client-id 前缀匹配列为备选未采纳。
- 2026-07-03：**活跃实体判定修订**——"申请多、同时运行少"为常态：新增 Step 0 三态分类（Active / Standby / Unknown，流量阈值 ε + K 周期迟滞，复用 byte-rate 指标，替代"存在 sensor 即活跃"的 1h 粗粒度定义）；Σreservation 只计 Active；Standby 受待命限速 max(res, H)、新客受新客限速 H（= 容量安全余量）约束；生产方向同步改为双锚点（client-id 业务线 / topic-partition）。
- 2026-07-03：**性能评审补充**——新增 §3.3.5 性能分析与预算：热路径零新增开销（overlay 不在每请求路径）、控制环成本量化（10k 实体每周期毫秒级）、两个真实风险（禁止全量扫 registry → 活跃实体索引；写锁持有 → 变更阈值过滤/分批/无锁最终一致）、三条硬性验收基准；实现计划增加性能基准任务。
- 2026-07-03：**归属规则澄清**——§3.2 新增归属规则表与示例：活跃检测/记账/限速的单位统一为 sensor（实体），每笔流量恰好记入一个实体（不重不漏），控制环不识别客户端；只配 TP 时该分区全部流量聚合为一个实体（活跃判定 = 分区是否有流量）；显式标注既定语义"TP（第 5 级）捕获含 client-id 配置（第 10 级）客户端的流量"，不调整链序，运营上建议稳定组与随机组业务不共用 topic。
- 2026-07-07：**保底虚高评审**——新增示例 4（纸面超卖但实际空闲：未用保底不占池、借用不受影响，超卖只是承诺风险信号）；**修正 §3.4 过强表述**（"超卖期间 spare ≈ 0"仅在实压满时成立）；新增 `green-usage-ratio` 与实体级 `reservation-utilization` 指标，超卖告警按"纸面/实压"分级；边界情况与决策表补充"保底虚高"条目（算法不按历史用量折扣保底，虚高靠利用率观测 + 运营修剪治理）。
- 2026-07-07：**评审修订（第二轮）**——①删除错误的"内部 client-id 前缀豁免"描述（该机制在代码中不存在），改为按 topic 维度豁免：弹性只治理非内部 topic，`__` 前缀分区流量绕过弹性记账与限速；②明确 reservation/weight/limit **三参数原子解析**（同链同实体一次取值，禁止各自独立走链），跨来源组合运行期钳制 res ≤ L；③**算法简化**：删除需求爬升系数 h 与平滑系数 α（组合爬升 100→800 需 ~18 周期，太慢且非必要），需求每周期全量重算——未受限按 u_e、受限按 L_e 申报由注水裁决，原 Step 7 平滑删除（Step 8 → Step 7）；④控制环周期默认 5s → **1s**（缓解多实体连坐节流放大，误压制窗口降至秒级），K 默认 3 → 10 保持记忆时长量级，写锁变更阈值过滤升为必要项；⑤overlay 未命中路径由 H 改为 **max(res_e, H)**（保底实体冷启动/重启即受保护，与 Standby 待遇统一）；⑥重写回退实体表述：default 实体是**参数模板**而非共享流量桶，未配置流量各自成回退实体；⑦弹性参数**不支持 user 系实体**（链序交错：user 精确系 1-4 级优先于 TP），归属规则表按实际链序修正；⑧新增控制环失活监测（`refresh-age-ms` 指标 + 线程守护）、produce 回退 tags 变化的监控口径说明、"申报-回落振荡"与"连坐节流"边界条目；数值示例 1/2/4 按新算法重算。per-broker 语义维持单机视角（集群级保障明确不在本期范围）。
