---
title: "Quota Flow Control Survey"
---

## Kafka 流控/配额方案调研报告

> 目的:为重新设计本仓库的客户端限速方案提供选型输入。目标模型:**每个消费者组/topic 拥有最低保障资源量(保底),极端争用下核心业务的生产/消费不失败;资源充足时可弹性使用空闲容量(类 YARN 动态资源池)**。
>
> 本报告为调研性质文档,不做最终选型;选型确认后另起 `plans/<功能名>-design.md` 设计文档。
>
> 调研日期:2026-07

---

### 0. 执行摘要

- Kafka 内置配额(含本仓库 topic-partition 扩展)全部是**静态上限 + 违规后延迟/mute channel** 的惩罚式模型:无保底、无弹性借用、非工作保全(broker 空闲时客户端也不能超过自身配额),且**不存在 broker 总带宽池的概念**。
- 跨领域调研显示,各成熟系统高度一致地收敛为**三参数模型:reservation(保底)+ weight(权重,分配空闲容量)+ limit(上限)**——Linux HTB(rate/ceil)、YARN Fair Scheduler(minResources/weight/maxResources)、mClock(reservation/weight/limit)、trTCM(CIR/PIR)、SQL Server Resource Governor(MIN/MAX/CAP)本质是同一模型的不同表述。
- Kafka 生态内最接近目标的先例:**Strimzi kafka-quotas-plugin**(基于官方 KIP-257 插件点实现集群共享预算动态分配,但无保底)、**Confluent Cloud**(专有的"租户额度切片 + auto-tuning",最接近完整目标)、**Pulsar ResourceGroups**(集群聚合配额 + broker 间用量交换收敛)、**AutoMQ**(分优先级令牌桶保护 produce 流量)。
- 推荐方向:以 **(reservation, weight, limit) 三参数**为配置模型,执行层借鉴 **HTB/mClock 的"先满足保底、再按权重分剩余、limit 封顶"**;落地路径建议分阶段——先基于 KIP-257 `ClientQuotaCallback` 插件验证动态分配策略(Strimzi 式,低侵入),再视效果改造执行层实现真正的调度式保底(详见第 6 章)。

---

### 1. 背景与目标

#### 1.1 现状痛点

本仓库当前的客户端限速由 `ClientQuotaManager`(`core/src/main/scala/kafka/server/ClientQuotaManager.scala`)实现,并已扩展出 topic-partition 粒度配额(11 级优先链,见 `plans/topic-partition-quota-design.md`)。该模型的问题:

1. **只有上限,没有保底**。配额是"不许超过 X",而不是"至少保证 X"。当 broker 整体过载时,没有任何机制优先保障核心业务的生产/消费成功,可能造成业务失败。
2. **非工作保全(non-work-conserving)**。broker 空闲时,客户端也不能超过自身配额使用空闲带宽,资源浪费。
3. **静态配置**。配额值人工设定,无法随负载/租户活跃度动态调整;为了防最坏情况只能把配额设得保守,进一步加剧浪费。
4. **惩罚式执行**。超配额后计算 throttle time 并 mute channel(延迟),而非调度式地在争用时按优先级/保底分配带宽。
5. **无总量概念**。配额之间互相独立,所有配额之和可以远超 broker 实际能力,也没有"broker 总容量池"可供弹性分配。

#### 1.2 目标模型

借鉴 YARN 动态资源池:

| 语义 | 说明 |
|------|------|
| 保底(reservation) | 每个消费者组/topic 分配最低资源量,争用时优先满足,保证极端情况下业务不失败 |
| 弹性共享(work-conserving) | 资源充足时,允许超出保底使用空闲容量 |
| 上限(limit,可选) | 防止单一租户无限占用 |
| 争用回收 | 借用的资源在其他租户需要其保底时能被收回(通过节流借用者实现) |

---

### 2. Kafka 内置流控机制盘点(结合本仓库代码)

#### 2.1 客户端配额(KIP-13 / KIP-124 / KIP-219)

**机制**(KIP-13,0.9 引入):broker 端按实体(user / client-id 组合)维护滑动窗口 `Rate` 指标(默认 11 个样本 × 1 秒窗口,`QuotaConfigs.java:44,54`),每次 produce/fetch 记账后检查是否超过配额;超过则抛出 `QuotaViolationException`,计算节流时间并延迟响应。选择"延迟"而非"拒绝"的原因:拒绝会引发客户端重试,反而放大负载。

**本仓库关键代码路径**:

- 管理器组装:`core/src/main/scala/kafka/server/QuotaFactory.scala:28-92`。`QuotaType` 枚举含 Fetch / Produce / Request / ControllerMutation / LeaderReplication / FollowerReplication / AlterLogDirsReplication / RLMCopy / RLMFetch;`QuotaManagers` 持有各管理器实例。
- produce 记账:`KafkaApis.scala:691`,逐分区调用 `quotas.produce.recordAndGetThrottleTimeMs(session, clientId, topicPartition.toString, sizeInBytes, timeMs)`,取各分区节流时间最大值。
- fetch 记账:`KafkaApis.scala:1039`;违规时通过 `unrecordQuotaSensor`(`KafkaApis.scala:1050`)把已记账的字节数"反记账",避免返回空响应却重复计费。
- 节流公式:`core/src/main/scala/kafka/utils/QuotaUtils.scala:40-53`,`throttleTime = (value - bound) / bound × windowSize`,即"等待多久平均速率才能回到配额内"。带宽配额不封顶;Request 配额用 `boundedThrottleTime` 封顶一个窗口(`ClientRequestQuotaManager.scala:47,84`)。
- 执行机制:`ThrottledChannel.scala:36-61` 构造即调用 `startThrottling`(经 `RequestHandlerHelper.scala:65-75` 到 SocketServer 将连接 mute),放入 DelayQueue,到期后 unmute。KIP-219(2.0)改进为**先返回带 `throttle_time_ms` 的响应、再 mute**,避免客户端请求超时引发重试风暴。

**KIP-124**(0.11):增加 `request_percentage`(请求处理线程时间百分比)配额,防止某客户端占满网络/IO 线程 CPU。

**语义定性**:静态硬上限,窗口平均。一次大突发会把整个窗口的平均值顶高,导致节流时间偏长(这正是后来 KIP-599 改用令牌桶的动因)。

#### 2.2 副本复制配额(KIP-73)

`core/src/main/scala/kafka/server/ReplicationQuotaManager.scala`:对**指定 replicas**(topic 级 `leader/follower.replication.throttled.replicas`)的复制流量按 broker 级速率(`leader/follower.replication.throttled.rate`,默认 `Long.MAX_VALUE`)限速。leader 侧从 fetch 响应中剔除超速的 throttled 分区,follower 侧从 fetch 请求中剔除。`isQuotaExceeded`(:84)+ `isThrottled(tp)`(:102)。本质仍是静态上限,只是作用对象是副本迁移/追赶流量。值得注意:它的执行方式是**"从本轮响应中剔除"而非 mute 连接**——一种更接近调度的执行形态。

#### 2.3 Controller Mutation 配额(KIP-599)

`core/src/main/scala/kafka/server/ControllerMutationQuotaManager.scala`:对创建/删除 topic、创建分区的"分区变更数"限速,使用 **`TokenBucket`**(`org.apache.kafka.common.metrics.stats.TokenBucket`)而非 `Rate`:

- **为何弃用 Rate**:创建一个大 topic 是单次不可分割的大突发,用窗口平均会导致平均值长期超标、节流远超必要时长;令牌桶允许桶被单次大操作透支为负,之后按速率恢复,`throttleTime = -credit / rate`(`ControllerMutationQuotaManager.scala:144-151`)。
- 分 Strict(新版客户端直接返回 `THROTTLING_QUOTA_EXCEEDED` 错误)与 Permissive(只记账并 mute)两种模式。
- KAFKA-10364(2.7)已把 TokenBucket 作为通用 `MeasurableStat` 进入 metrics 库,**带宽配额切换到令牌桶语义的基础设施已经存在**。

#### 2.4 分层存储配额(KIP-956)与配额管理 API(KIP-546)

- KIP-956(3.9):`remote.log.manager.copy/fetch.max.bytes.per.second`,RLM 上传/拉取的 **broker 级总量**静态上限——Kafka 中少见的"全局池上限"先例,但无租户内分配。
- KIP-546(2.6):`DescribeClientQuotas` / `AlterClientQuotas` Admin API 与 `ClientQuotaEntity` 模型,本仓库 topic-partition 实体即扩展于此(`ClientQuotaEntity.java:36`)。

#### 2.5 可插拔扩展点:KIP-257 ClientQuotaCallback

配置 `client.quota.callback.class`(`QuotaConfigs.java:56-60`),在 `QuotaFactory.scala:79-80` 实例化并注入所有客户端配额管理器。

**能控制什么**:
- `quotaMetricTags(quotaType, principal, clientId)`:决定请求归入哪个配额桶(**即"谁和谁共享一个配额"的分组规则**);
- `quotaLimit(quotaType, metricTags)`:该桶的配额值,可随时变化(配合 `quotaResetRequired` 触发全量刷新);
- `updateClusterMetadata(cluster)`:感知分区 leader 分布变化,是实现"按 leader 分布切分集群配额"的关键钩子。

**不能控制什么**:节流时间公式、mute 执行机制、sensor 生命周期、记账时机——执行层完全固化在 `ClientQuotaManager` / `QuotaUtils` 中。

**结论**:KIP-257 足以实现"动态计算每个租户在本 broker 的配额值"(Strimzi、Confluent 均基于此),但**无法实现调度式的严格保底**(它只能改"上限值",不能改变"超限即延迟"的执行语义)。

#### 2.6 本仓库扩展:topic-partition 配额与 AutoBalancer 流量上报

**topic-partition 配额**(`plans/topic-partition-quota-design.md`):
- 新增 `TOPIC_PARTITION` 配额实体(`ClientQuotaEntity.java:36`),实体名 `<topic>-<partition>`;可单独配置或与 user 组合,禁止与 client-id 组合、禁止 default(`ClientQuotaControlManager.java:229-252`);仅支持 `producer_byte_rate` / `consumer_byte_rate`(`DynamicConfig.scala:104-132`、`QuotaConfigs.java:184-191`)。
- 未新建独立管理器,复用 `ClientQuotaManager`,通过 `DefaultQuotaCallback` 的 **11 级优先链**(`ClientQuotaManager.scala:624-695`)在 user/client-id 层级间插入 topic-partition 层级。
- 无 MetadataVersion 门控,配置存在即生效(`QuotaTypes` 位掩码,`ClientQuotaManager.scala:46-54`)。
- **对新方案的意义**:证明了在现有实体模型上新增配额维度(未来的 consumer group 维度)的完整改造路径(entity type → 校验 → 优先链 → 记账参数)。

**AutoBalancer 分区流量上报**(AutoMQ 注入):
- `BrokerTopicPartitionMetrics.java`:分区级 MessagesIn/BytesIn/BytesOut Yammer 指标(懒注册);
- `AutoBalancerMetricsReporter.java`:筛选后每 10s 写入内部 topic `__auto_balancer_metrics`(producer client-id 带 `__automq_client_` 前缀,豁免配额);开关 `auto.balancer.reporter.enable`(默认 false)。
- **对新方案的意义**:集群级"每分区实际流量"数据流已经存在,是动态配额决策(集群聚合视角)的现成数据源。

#### 2.7 能力边界小结

| 能力 | 现状 |
|------|------|
| 静态上限(user/client-id/topic-partition) | 有 |
| 保底 | **无** |
| 弹性借用 / 工作保全 | **无** |
| broker 总容量池 | **无**(最接近的只有复制节流速率与 RLM 配额;`allTopicsStats` 可观测总流量但非配置上限) |
| 动态调整入口 | 有(KIP-257 callback,可动态改 limit 与分组) |
| 令牌桶基础设施 | 有(KAFKA-10364) |
| consumer group 配额实体 | **无**(需仿照 topic-partition 扩展新增) |
| 上游社区保底/弹性类 KIP | **无**(检索未发现 mainstream 提案) |

---

### 3. Kafka 生态其他系统的流控方案

#### 3.1 Strimzi kafka-quotas-plugin ★最接近的开源先例

- **机制**:实现 KIP-257 的 `StaticQuotaCallback`。配置**集群/broker 级总吞吐预算**(如 produce 总量 40 MB/s),插件把总预算**动态分配给当前活跃的客户端**——A 只用 10 MB/s 时 B 可用到 30 MB/s,即工作保全的共享池。另支持按磁盘水位(soft/hard limit)渐进收紧配额直至停写。
- **语义**:共享总量上限 + 动态再分配;**无保底、无权重、无层级**。
- **启示**:证明了"不动 Kafka 核心、纯插件实现动态共享池"的可行性;其"总量预算"概念正是本仓库缺失的 broker 容量池。
- 来源:https://github.com/strimzi/kafka-quotas-plugin

#### 3.2 Confluent Cloud 多租户动态配额(专有)

- **机制**:每个租户(逻辑集群)有 ingress/egress 总额度;通过自研 `ClientQuotaCallback` 把租户额度**按分区 leader 分布切片到各 broker**,leader 迁移/扩缩容时自动重算。当 broker 总负载逼近其容量时,**auto-tuning** 按各租户原始额度比例收缩活跃租户配额、把闲置租户的份额让给活跃租户;负载缓解或集群扩容后恢复。
- **语义**:租户额度≈软保底 + 弹性共享 + broker 容量保护,是**目标模型最完整的生产级实现**,但闭源。
- **启示**:① 集群额度→per-broker 切片的架构;② "按比例收缩"的争用回收策略;③ 全部构建在 KIP-257 之上。
- 来源:https://www.confluent.io/blog/cloud-native-multi-tenant-kafka-with-confluent-cloud/

#### 3.3 AutoMQ 分优先级令牌桶

- **机制**:`AsyncNetworkBandwidthLimiter` = 令牌桶 + 优先级队列。`s3.network.baseline.bandwidth` 定义 broker 总带宽池;流量分层——Tier-0(produce)**不节流直接放行**,catch-up 读、compaction 等低优先级流量在令牌不足时排队等待。冷读走对象存储,与热路径物理隔离。
- **语义**:总量池 + 优先级抢占式共享;"保底"以"最高优先级不节流"的形式体现,而非显式速率值。
- **启示**:优先级分层是实现"极端情况下核心业务不失败"的另一条路径——不给核心业务设保底数值,而是给它更高的调度优先级;实现比按租户精细分配简单。
- 来源:https://www.automq.com/blog/deep-dive-into-the-challenges-of-building-kafka-on-top-of-s3

#### 3.4 Apache Pulsar

- **publish/dispatch rate**:broker / namespace / topic / subscription 多级静态限速,PIP-322 用统一令牌桶重构了实现(解决旧 precise limiter 的锁竞争)。
- **ResourceGroups(PIP-82)**:定义租户/namespace 级**集群聚合配额**;各 broker 周期性交换本地用量,分布式地调整自己的本地份额,使集群总量向配额值收敛。**有意设计为软限制**(容忍短期超调,反馈收敛)。
- **publish buffer 背压**:`maxMessagePublishBufferSizeInMB` 超限后 broker 停止读取 producer 连接,借 TCP 背压传导——真正的负载自适应机制。
- **启示**:PIP-82 的"用量交换 + 各自收敛"是无中心协调者的集群聚合配额参考实现;其"软限制"取舍(接受精度换取无锁与可用性)值得借鉴。
- 来源:https://github.com/apache/pulsar/blob/master/pip/pip-82.md 、https://pulsar.apache.org/docs/4.0.x/concepts-throttling/

#### 3.5 RocketMQ 与 Redpanda(简述)

- **RocketMQ**:无配额体系,纯反应式过载保护——PageCache busy(CommitLog 锁持有超时)、发送队列等待超时即 fast-fail 返回 SYSTEM_BUSY(客户端默认不重试该错误)。作为对比设计点:**快速失败 + 不重试**也是一种"极端情况下保护系统"的策略,但牺牲的恰是用户想要的"业务不失败"。
- **Redpanda**:内部用 Seastar 调度组(shares)做 CPU/IO 比例调度,但**租户面仍是 Kafka 兼容的静态配额**(`rpk cluster quotas`)。说明:即便引擎具备强调度能力,把它暴露成租户级保底/弹性语义仍需要显式设计。
- 来源:https://docs.redpanda.com/current/manage/cluster-maintenance/manage-throughput/

---

### 4. 其他领域经典流控/资源共享方案

#### 4.1 调度器类

##### YARN Fair Scheduler / Capacity Scheduler(用户所指"动态资源池"的原型)

- **Fair Scheduler**:每队列 `minResources`(保底)/ `maxResources`(上限)/ `weight`(分剩余的权重)。调度顺序:低于 minShare 的队列**最优先**,其余按 `used/fairShare` 排序;空闲资源按权重分配。争用回收靠**抢占**:低于保底超过 `minSharePreemptionTimeout` 后,杀掉超额队列最新启动的容器。Cloudera "动态资源池"即其管理界面封装(父子池层级、未声明子池继承默认值)。
- **Capacity Scheduler**:`capacity`(保底百分比,兄弟队列之和 = 100,**从制度上保证保底不超卖**)/ `maximum-capacity`(弹性上限)/ `user-limit-factor`(队列内单用户弹性倍数)。
- **启示**:① 保底之和 ≤ 总容量的约束必须在配置校验层强制;② 抢占(对应到 Kafka:节流借用者)是保底成立的必要条件;③ 父子层级(broker → topic → 消费者组)与"未声明实体继承默认值"直接可搬。
- 来源:https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/FairScheduler.html 、https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/CapacityScheduler.html

##### mClock / dmClock(存储 QoS,Ceph 生产验证)★调度器形态的最佳模板

- **机制**(VMware OSDI'10):每客户端三标签 **(reservation, limit, weight)**。两阶段调度:先服务 reservation 标签到期的请求(**强制保底**),再把剩余容量按 weight 标签比例分配给未达 limit 的客户端。标签为 O(1) 递推计算(`R_i = max(R_{i-1} + 1/r, now)`),无全局锁;dmClock 扩展到分布式多服务端(Ceph OSD 生产使用),闲置客户端回归时标签重同步、不因历史闲置受罚。
- **启示**:**单一调度器统一三参数**的最简洁形式;标签递推的实现方式对高频请求路径(如 broker 每次 produce/fetch)开销极小;dmClock 证明了该模型可分布式化。
- 来源:https://cs.uwaterloo.ca/~brecht/courses/854-Emerging-2014/readings/misc/mclock-osdi-2010.pdf 、https://docs.ceph.com/en/latest/rados/configuration/mclock-config-ref/

##### SQL Server Resource Governor / Oracle DBRM / Kubernetes

- **SQL Server RG**:资源池 `MIN_CPU_PERCENT`(争用时保底)/ `MAX_CPU_PERCENT`(**软顶:仅争用时生效,空闲可超**)/ `CAP_CPU_PERCENT`(硬顶:任何时候不可超);MIN 之和 ≤ 100。**软顶/硬顶的区分**是重要设计词汇:多数租户想要"能蹭空闲"的软顶 + "防失控"的硬顶。
- **Oracle DBRM**:多级百分比计划(Level 1 未用完的容量下放 Level 2)+ `SHARES` 比例 + `UTILIZATION_LIMIT` 硬顶;适合租户内部再分层(如同一租户的实时/批量消费组)。
- **Kubernetes/cgroups**:`requests` → cpu.shares(争用时按比例保障)+ `limits` → cfs_quota(硬顶);QoS 三级 **Guaranteed(requests=limits,无弹性)/ Burstable(保底+弹性)/ BestEffort(无保底)** 是清晰的租户分级词汇——目标模型即"Burstable"。
- 来源:https://learn.microsoft.com/en-us/sql/t-sql/statements/create-resource-pool-transact-sql 、https://docs.oracle.com/en/database/oracle/oracle-database/21/admin/managing-resources-with-oracle-database-resource-manager.html 、https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/

#### 4.2 网络类

##### Linux tc HTB(Hierarchical Token Bucket)★结构形态的最佳模板

- **机制**:类树结构,每类 `rate`(保底,任何时候可发)/ `ceil`(借用上限)/ `prio`(借用优先级)/ `quantum`(同级公平粒度)。子类流量超过 rate 后可**向父类借用空闲容量**直至 ceil;三态机:can-send(rate 内)→ may-borrow(父有空闲且未到 ceil)→ can't-send。无抢占,纯机会式借用——父类没空闲时自动跌回 rate。
- **启示**:"保底 + 向上借用 + 封顶"的经典结构;层级天然映射 broker(根)→ topic/租户(内部类)→ 消费者组(叶子);机会式借用**不需要抢占机制**(带宽是瞬时资源,下一时刻重新分配即可),比 YARN 的容器抢占简单得多——这一点对 Kafka 极为有利,因为流量记账本来就是逐请求进行的。
- 来源:https://man7.org/linux/man-pages/man8/tc-htb.8.html

##### trTCM(RFC 2698)双速率三色标记 ★契约语义的最佳模板

- **机制**:CIR/CBS(承诺速率/突发)+ PIR/PBS(峰值速率/突发)双令牌桶。包着色:≤CIR 为**绿**(承诺内,必须保障)、CIR~PIR 为**黄**(弹性带,尽力而为/可降级)、>PIR 为**红**(丢弃/拒绝)。srTCM(RFC 2697)为单速率变体。
- **启示**:与目标契约语义**一字不差**——CIR=保底、PIR=硬顶、黄色带=弹性区间。可直接借用其"颜色"概念定义 Kafka 请求的处理策略:绿=不节流,黄=broker 有空闲则放行、争用则节流,红=节流(甚至快速失败)。
- 来源:https://datatracker.ietf.org/doc/html/rfc2698

##### WFQ / DRR / max-min fairness

- **max-min fairness**:注水算法定义的公平——无法在不损害更小分配者的前提下增加任何流的分配。**加权 max-min** 正是"按权重分剩余容量"的形式化定义。
- **DRR(Deficit Round Robin)**:每队列 quantum(∝权重)+ 赤字计数器,O(1) 每包成本(WFQ 为 O(log N)),是 HTB quantum 机制的内部实现。适合做"保底满足后剩余带宽按权重轮转分配"的低开销原语。
- 来源:https://web.stanford.edu/class/ee384x/EE384X/papers/DRR.pdf

#### 4.3 算法类

- **令牌桶 vs 漏桶**:令牌桶容忍突发(令牌可积攒)、漏桶强制匀速。消息流量天然突发,令牌桶是默认正确选择;Kafka 自身也已从 Rate 窗口平均演进出 TokenBucket(KIP-599,§2.3)。
- **Guava RateLimiter**:SmoothBursty(默认存 1 秒突发额度)/ SmoothWarmingUp(冷启动爬坡,coldFactor=3,防止空闲后突然满速打爆下游)。其"**预支**"设计(本次超额放行、把等待成本转嫁给下一请求)与 KIP-599 令牌桶透支异曲同工,适合处理"单条大消息不可分割"的记账问题。
- **窗口算法**:固定窗口有边界 2× 突发问题;滑动窗口计数器(当前+上一窗口加权)是低成本高精度的实用选择。Kafka 现有 Rate 即多样本滑动窗口。
- 来源:https://www.alibabacloud.com/blog/594820

#### 4.4 自适应/反馈类

- **TCP AIMD**:加性增、乘性减,Chiu-Jain 证明可从纯局部反馈收敛到公平——说明"公平共享"未必需要中心配置。
- **BBR**:用观测到的 BtlBw × RTprop 估计最优操作点,主动探测容量。对应到 broker:可用观测吞吐/延迟为每租户估计"可达容量",作为动态上限。
- **Sentinel**:系统自适应保护(load1 + `maxQps × minRt` 的 BBR 式容量估计,超过即限流)+ WarmUp 冷启动模式。
- **Envoy adaptive concurrency**:gradient = minRTT/sampleRTT 动态调节并发上限,延迟健康则扩、恶化则缩。
- **启示**:自适应机制不提供"保底"承诺,但可以作为**总容量池的动态估计器**(替代人工配置 broker 容量)以及保底之上的**动态软顶**;与静态三参数模型是互补关系。
- 来源:https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/adaptive_concurrency_filter

---

### 5. 横向对比矩阵

| 系统/方案 | 保底 | 弹性借用(工作保全) | 上限 | 层级结构 | 争用回收 | 实现复杂度 | 对本需求的适配 |
|---|---|---|---|---|---|---|---|
| Kafka 内置配额(含本仓库 TP 扩展) | 无 | 无 | 硬(静态) | user>client-id 优先链 | n/a | 已有 | 现状基线 |
| Strimzi quotas-plugin | 无 | 有(共享总预算) | 集群总量 | 无 | 活跃者均分 | 低 | KIP-257 即插即用先例 |
| Confluent Cloud | 隐式(租户额度) | 有(auto-tune) | broker 容量 | 租户 | 按额度比例收缩 | 高 | 架构参考(闭源) |
| AutoMQ 限流器 | Tier-0 优先级保护 | 有(优先级令牌桶) | baseline bandwidth | 优先级层 | 低优排队 | 中 | 优先级模式可移植 |
| Pulsar ResourceGroups | 无(仅集群聚合) | 有(broker 份额动态) | 组总量(软) | 租户/NS | 反馈收敛 | 中 | 集群聚合参考 |
| YARN Fair/Capacity | minResources / capacity% | weight 分剩余 | maxResources | 队列树 | 抢占 | 高 | 概念模板 |
| Linux HTB | rate | 向父类借至 ceil | ceil | 类树 | 机会式(无抢占) | 中 | **结构模板** |
| mClock/dmClock | reservation | weight 分剩余 | limit | 平面(可组合) | reservation 优先调度 | 中 | **调度器模板** |
| trTCM(RFC 2698) | CIR | 黄色带 | PIR | 无 | 颜色降级 | 低 | **契约语义模板** |
| SQL Server RG | MIN% | 空闲可超软顶 MAX% | CAP%(硬) | 池 | 调度回收 | — | 软顶/硬顶概念 |
| K8s requests/limits | shares 比例保障 | 争用外可超 | cfs 硬顶 | QoS 三级 | 驱逐/节流 | — | 租户分级词汇 |
| 自适应(BBR/Sentinel/Envoy) | 无 | 天然 | 动态 | 无 | 反馈收缩 | 中 | 容量估计/动态顶补充 |

**关键观察**:HTB、mClock、YARN、trTCM、SQL Server RG 是同一个三参数模型在不同领域的实例;差异只在**执行形态**(逐包调度 / 容器抢占 / 标记降级)与**层级深度**。Kafka 生态内尚无开源的完整实现,但 KIP-257 + 令牌桶 + AutoBalancer 指标流已备齐大部分零件。

---

### 6. 设计启示与推荐方向

#### 6.1 推荐配置模型:三参数 + 颜色语义

为每个配额实体(消费者组 / topic,可分别或组合)定义:

```
reservation  保底速率(争用时必须满足;所有实体 reservation 之和 ≤ broker 容量,配置层强校验)
weight       权重(分配"容量 - Σreservation"的剩余/空闲带宽)
limit        上限(可选,硬顶;缺省 = 不封顶,纯靠容量约束)
```

请求处理时按 trTCM 颜色决策:

- **绿**(实体用量 ≤ reservation):直接放行,永不节流 → 实现"极端情况下业务不失败";
- **黄**(reservation < 用量 ≤ limit):broker 容量有空闲则放行(工作保全),争用时按权重比例节流回收 → 实现"资源充足时尽可能用";
- **红**(用量 > limit):按现有机制节流。

该模型向后兼容:现有配额等价于 `reservation=0, limit=旧配额值`;未配置保底的实体即 K8s 语义的 BestEffort。

#### 6.2 五个关键难点(设计阶段必须解答)

1. **配额是 per-broker 的,而消费者组/topic 流量跨 broker**。保底以什么口径定义?两条路:(a) 按分区 leader 分布把集群级保底切片到 broker(Confluent 做法,`ClientQuotaCallback.updateClusterMetadata` 即为此设计);(b) broker 间交换用量、各自收敛(Pulsar PIP-82)。本仓库的 `__auto_balancer_metrics` 分区级流量流(§2.6)是现成的集群聚合数据源,倾向 (a) 起步(仅依赖元数据,无需用量交换)。
2. **broker 总容量池不存在,需新增配置**。如 `broker.produce.capacity.bytes.per.second` / `broker.fetch.capacity.bytes.per.second`(参考 AutoMQ `s3.network.baseline.bandwidth`、KIP-956 的 broker 级总量先例)。进阶:用自适应估计(§4.4)替代人工配置。
3. **保底语义要求"绿色流量不节流",现有执行层是"超限即节流"**。KIP-257 插件只能动态改 limit 值,做不到按颜色区分执行;真正的调度式保底需要改造 `ClientQuotaManager` 记账/判定路径(在 `recordAndGetThrottleTimeMs` 处引入容量池与颜色判定)。
4. **配额实体无 consumer group 概念**。需新增 entity type,完整改造路径已有本仓库 topic-partition 扩展先例(§2.6);注意本仓库审查规范:新实体需考虑 MetadataVersion 门控(TP 扩展未做,是已知欠账)、新功能默认关闭开关。
5. **组内再分配与噪声隔离**。同一保底实体内多个客户端如何分享额度(参考 Capacity Scheduler `user-limit-factor`、DRR 轮转);保底实体的突发是否允许预支(参考 KIP-599 透支、Guava 预支)。

#### 6.3 落地路径初评

| 路径 | 做法 | 优点 | 局限 |
|------|------|------|------|
| **A. KIP-257 插件路线**(Strimzi 式) | 自研 `ClientQuotaCallback`:维护容量池与各实体 reservation/weight,**周期性重算**各实体的动态 limit(空闲多则调高、争用则按"保底 + 权重份额"收缩) | 不动核心代码,风险最低,可独立部署/回滚;足以验证分配策略与参数 | 保底是"近似软保底"(重算周期内可能被瞬时突发挤占);颜色级执行语义做不到;仍受窗口平均节流公式制约 |
| **B. 改造执行层路线** | 修改 `ClientQuotaManager`(或新建 Manager):记账时对照容量池做颜色判定,绿放行、黄视空闲、红节流;底层换令牌桶(基础设施已有,KAFKA-10364) | 真正的调度式保底,语义精确 | 侵入核心热路径(produce/fetch 每请求),需处理锁与性能、混合升级兼容,工作量大 |
| **C. 分阶段(推荐)** | 先 A 验证策略有效性与参数手感(容量值、权重、重算周期),指标充分后再做 B 落地精确语义 | 风险递进、每阶段独立可用;A 阶段产出(容量池配置、实体模型、分配算法)可被 B 复用 | 周期较长 |

**推荐**:路径 C。第一阶段基于 KIP-257 + 新增 broker 容量配置 + AutoBalancer 指标流,实现"软保底 + 弹性共享"插件;第二阶段视效果决定是否改造执行层引入颜色语义与令牌桶。最终选型与详细设计另起 `plans/<功能名>-design.md`。

---

### 7. 参考资料

#### Kafka KIP 与 issue

- KIP-13 Quotas:https://cwiki.apache.org/confluence/display/KAFKA/KIP-13+-+Quotas
- KIP-73 Replication Quotas:https://cwiki.apache.org/confluence/display/KAFKA/KIP-73+Replication+Quotas
- KIP-124 Request rate quotas:https://cwiki.apache.org/confluence/display/KAFKA/KIP-124+-+Request+rate+quotas
- KIP-219 Improve quota communication(KAFKA-6028):https://issues.apache.org/jira/browse/KAFKA-6028
- KIP-257 Configurable Quota Management:https://cwiki.apache.org/confluence/display/KAFKA/KIP-257+-+Configurable+Quota+Management
- KIP-546 Client Quota APIs:https://cwiki.apache.org/confluence/display/KAFKA/KIP-546%3A+Add+Client+Quota+APIs+to+the+Admin+Client
- KIP-599 Throttle topic operations:https://cwiki.apache.org/confluence/display/KAFKA/KIP-599%3A+Throttle+Create+Topic%2C+Create+Partition+and+Delete+Topic+Operations
- KIP-956 Tiered Storage Quotas:https://cwiki.apache.org/confluence/display/KAFKA/KIP-956+Tiered+Storage+Quotas
- KAFKA-10364 通用 TokenBucket:https://issues.apache.org/jira/browse/KAFKA-10364

#### Kafka 生态

- Strimzi kafka-quotas-plugin:https://github.com/strimzi/kafka-quotas-plugin
- Confluent Cloud 多租户设计:https://www.confluent.io/blog/cloud-native-multi-tenant-kafka-with-confluent-cloud/
- Confluent Kafka Summit 2020 "Sharing is Caring":https://www.confluent.io/resources/kafka-summit-2020/sharing-is-caring-toward-creating-self-tuning-multi-tenant-kafka/
- AutoMQ 限流设计:https://www.automq.com/blog/deep-dive-into-the-challenges-of-building-kafka-on-top-of-s3
- Pulsar PIP-82 Resource Groups:https://github.com/apache/pulsar/blob/master/pip/pip-82.md
- Pulsar throttling 概念:https://pulsar.apache.org/docs/4.0.x/concepts-throttling/
- Redpanda 吞吐管理:https://docs.redpanda.com/current/manage/cluster-maintenance/manage-throughput/

#### 其他领域

- YARN Fair Scheduler:https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/FairScheduler.html
- YARN Capacity Scheduler:https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/CapacityScheduler.html
- Linux tc-htb(8):https://man7.org/linux/man-pages/man8/tc-htb.8.html
- RFC 2697 srTCM:https://datatracker.ietf.org/doc/html/rfc2697
- RFC 2698 trTCM:https://datatracker.ietf.org/doc/html/rfc2698
- mClock(OSDI 2010):https://cs.uwaterloo.ca/~brecht/courses/854-Emerging-2014/readings/misc/mclock-osdi-2010.pdf
- Ceph mClock 配置:https://docs.ceph.com/en/latest/rados/configuration/mclock-config-ref/
- Kubernetes Pod QoS:https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/
- SQL Server Resource Governor:https://learn.microsoft.com/en-us/sql/t-sql/statements/create-resource-pool-transact-sql
- Oracle Database Resource Manager:https://docs.oracle.com/en/database/oracle/oracle-database/21/admin/managing-resources-with-oracle-database-resource-manager.html
- DRR 论文:https://web.stanford.edu/class/ee384x/EE384X/papers/DRR.pdf
- Guava RateLimiter 解析:https://www.alibabacloud.com/blog/594820
- Envoy adaptive concurrency:https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/adaptive_concurrency_filter
