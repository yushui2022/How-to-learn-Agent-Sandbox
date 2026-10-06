# 第十二章　AutoPause 与运营账：闲置沙箱的成本治理

<!-- chapter-nav -->

[← 上一章](第十一章-内存账本-5MB增量与并发供给的物理基础.md) · [章节目录](../../README.md) · [下一章 →](第十三章-可观测性与执行审计-Agent的行车记录仪.md)
<!-- /chapter-nav -->



> 《从 0 到 1 学 Agent 沙箱》系列 · 进阶篇
>
> **源码基线**：腾讯 CubeSandbox `v0.7.2`，commit `f1aaa737fb3862202b1731e0e0d844c28779f930`。

## 引子：等待三小时，资源却一直在线

一个 Agent 早上创建沙箱，读取数据并生成初稿；随后等待用户确认，三小时后才继续运行测试。如果沙箱一直处于 Running，CPU 可能几乎为零，内存、cgroup、网络状态和节点容量却仍被占用。把“Agent 工作流有很长等待段”当成通用统计并不严谨；不同产品、用户和任务的等待比例差异很大，必须从事件日志测量。本章用一个可调参数 `q` 表示等待占比，而不把 80% 当作事实。

AutoPause 的目标是把等待期间的驻留内存换成磁盘快照或远端对象存储，唤醒时恢复原状态。它不是免费休眠：挂起需要冻结、保存、清理运行实例，恢复需要读盘、重建网络和重新接入代理。只有当省下的驻留成本大于挂起/恢复成本，并且额外延迟可接受时，策略才值得启用。

## 一、先做盈亏平衡

设：

- `C_m`：运行沙箱每秒的内存/节点机会成本；
- `C_s`：保存快照的存储成本每秒；
- `T_idle`：预计等待时长；
- `T_pause`：执行挂起耗时；
- `T_resume`：唤醒耗时；
- `C_io`：挂起和恢复产生的 I/O、网络与 CPU 成本；
- `V_latency`：产品为额外延迟付出的价值。

不挂起的成本约为 `C_run = C_m × T_idle`。挂起的直接成本约为：

```text
C_pause = C_m × T_pause + C_s × T_idle + C_io + V_latency × (T_pause + T_resume)
```

当 `C_run > C_pause` 才有经济收益。若只看存储和内存，最小有利等待时长可粗略写成：

```text
T_break_even ≈ (C_m × T_resume + C_io) / (C_m − C_s)
```

这是估算式，忽略了并发 I/O、失败重试和用户流失。它的价值在于提醒我们：10 秒交互暂停和 3 小时人工确认不能使用同一个阈值。

## 二、CubeSandbox 的真实生命周期语义

`v0.7.2` 文档定义了 `running`、`pausing`、`paused`、`resuming`、`terminated` 五种对外状态；`pausing` 和 `resuming` 是瞬时态，表示正在执行动作，不代表一个可长期查询的稳定状态。`paused` 保存 VM 状态并释放 CPU/内存驻留，`terminated` 不可恢复。参考：[沙箱生命周期文档](https://github.com/TencentCloud/CubeSandbox/blob/f1aaa737fb3862202b1731e0e0d844c28779f930/docs/zh/guide/lifecycle.md)。

自动动作由 `timeout` 与 `on_timeout` 驱动：`on_timeout="kill"` 到期销毁，`on_timeout="pause"` 到期挂起；负值 `NEVER_TIMEOUT` 表示不因空闲自动回收。文档特别强调，暂停不会自动取消有限的空闲截止时间；如果仍然超过 TTL，暂停沙箱可以被销毁。也就是说，AutoPause 和保留时长是两件事：前者节省驻留资源，后者决定状态最多保存多久。

生命周期管理器的 sweeper 以 `LastActiveMs` 和 `CreatedAt` 的较新者为基线，计算 idle 时长；启动时有 warmup，避免刚恢复的管理器因尚未补齐活动时间而误暂停。源码见 [sweeper.go](https://github.com/TencentCloud/CubeSandbox/blob/f1aaa737fb3862202b1731e0e0d844c28779f930/cube-lifecycle-manager/internal/sweeper/sweeper.go)。这是一项设计上的安全阀：空闲判定不是“没有 CPU 就挂起”，而是“在可观测活动时间之后没有请求达到阈值”。

## 三、触发机制：时间、事件与租户策略

### 3.1 时间阈值

最简单的策略是 `idle_for >= timeout`。适合批处理和明确的人工等待，但阈值过短会在 Agent 两次工具调用之间频繁 pause/resume，产生抖动；阈值过长则错过节省机会。可按任务画像设置：

| 场景 | 典型等待模式 | 策略方向 |
|---|---|---|
| 交互式 Code Interpreter | 用户很快继续提问 | 阈值较长，优先低唤醒延迟 |
| 人工审批/长轮询 | 等待分钟到小时 | 阈值较短，暂停后保留状态 |
| 批量评测 | 队列分段运行 | 按批次窗口暂停，避免每步抖动 |
| 长编译/训练 | CPU/IO 持续忙 | 不以 CPU 短时低谷判定闲置 |

### 3.2 事件唤醒

恢复事件通常来自 API 请求、SDK `connect()`、任务队列或用户操作。事件到达后，控制面需要先把状态从 `paused` 置为 `resuming`，再恢复快照、网络和代理；同一沙箱的并发请求必须合并，不能让十个请求各启动一次恢复。

唤醒后的第一个请求可能比普通请求慢。客户端应区分“正在恢复”和“任务执行失败”，使用幂等 request ID 重试。恢复失败时不能简单标记为 Running：需要保留失败原因、快照引用和下一步可执行动作（重试、销毁或人工介入）。

### 3.3 租户级策略

不同租户对延迟和成本的权重不同。可把策略写成：

```text
policy = (idle_threshold, max_paused_age, storage_class,
          resume_deadline, on_resume_failure)
```

企业合规租户可能要求暂停快照加密、限定保留区域、超过 24 小时销毁；内部评测租户可能追求低成本，允许更长恢复时间。平台默认值只能作为兜底，不能覆盖租户的最大保留和数据删除要求。

## 四、挂起的技术账：快照不是“把 RAM 搬到磁盘”这么简单

CubeSandbox 的暂停流程由 CubeMaster 创建快照绑定，Cubelet 执行 `PauseToSnapshot`，成功后 Master 标记绑定 READY；超时或明确失败的状态处理不同。源码在 [sandbox_resume_pause.go](https://github.com/TencentCloud/CubeSandbox/blob/f1aaa737fb3862202b1731e0e0d844c28779f930/CubeMaster/pkg/service/sandbox/sandbox_resume_pause.go)。

至少有五类状态需要记账：

1. **VM 内存页**：已写入页的大小与压缩比。
2. **设备与 VMM 状态**：virtio 队列、寄存器、vsock 连接等。
3. **文件系统 COW**：暂停前写入的私有层和临时文件。
4. **网络状态**：TCP 连接是否可恢复；很多长连接需要重建。
5. **元数据**：快照、卷、节点绑定、引用计数和 TTL。

远端对象存储可降低本地盘压力，却增加上传和恢复延迟；本地 NVMe 低延迟但受节点故障和容量约束。快照加密、校验和、垃圾回收、重复快照引用都属于运营成本。第九章讨论快照机制时强调的 COW/clone 语义，在 AutoPause 中直接变成存储账。

## 五、三种工作流的延迟—成本推演

### 交互式工作流

假设每次用户思考平均 45 秒，唤醒 250ms，内存机会成本为每沙箱每秒 0.00002 元，快照存储和 I/O 摊销每次 0.01 元。等待 45 秒的运行成本是 0.0009 元，低于一次挂起 I/O 的 0.01 元，频繁 AutoPause 反而更贵。策略应提高阈值，或使用预热池。

### 人工审批工作流

假设等待 2 小时，运行成本为 `7200×0.00002=0.144` 元；挂起/恢复与存储合计 0.03 元，额外延迟 0.5 秒。此时挂起有明显收益，前提是业务允许恢复延迟且快照不包含不应长期保留的数据。

### 批量评测

评测任务可能每轮运行 20 秒、等待队列 5 分钟。若一次评测有 1000 个沙箱，全部维持驻留会把内存峰值放大；按批次分组暂停可降低峰值。但批次同时恢复会造成 I/O 风暴，必须限制恢复并发并预留磁盘吞吐。上述数字均为示例假设，不代表 CubeSandbox 性能实测。

## 六、横向参照：E2B、Serverless 与 Kubernetes

E2B 的 SDK 语义强调沙箱可连接、暂停和恢复，适合把工作目录作为会话状态；使用托管服务时，实际暂停阈值、单次运行上限和存储计费要以合同与官方文档为准，不能从 API 名称推断。参考：[E2B Sandbox 文档](https://e2b.dev/docs/sandbox)。

Serverless 函数通常在调用结束后回收执行环境，冷启动换取成本；Agent 沙箱的差异是状态和交互更长，不能简单套用“一次调用一次销毁”。Kubernetes 的 Pod eviction、Job TTL 和 `terminationGracePeriodSeconds` 提供了调度与退出语义，但 Pod 本身没有 VM 快照级的工作状态恢复；Agent Sandbox 控制器若增加 pause/resume，需要明确快照后网络、PVC 和身份的语义。

## 企业视角：万级平台的月度 TCO

以 10,000 个并发会话为例，设平均驻留内存 512MiB、节点可用内存 48GiB，则内存维度至少需要约 `10000×0.5/48≈105` 台节点，未计 CPU、网络和冗余。若测得会话等待占比 `q=0.6`，AutoPause 能释放其中 60% 的驻留内存，理论节点数可降至约 42 台；实际还要扣除唤醒批次、快照存储、故障域冗余和冷却余量。若 q 只有 0.1，省下的节点可能被快照与恢复成本抵消。

运营看板应同时显示：运行/暂停数量、平均与 P95 idle、暂停成功率、恢复成功率、恢复延迟 P50/P95、快照字节数、存储费用、因恢复失败而销毁的比例。只看“暂停数量”会奖励误暂停；只看“节点节省”会掩盖用户延迟。

## 本章小结

AutoPause 是一个带状态和延迟的成本优化器。闲置比例必须从业务轨迹测量，80% 只能作为待验证假设；触发条件要区分人工等待、交互思考、批量队列和长任务。CubeSandbox 的 `pause/resume` 具备明确的快照绑定、TTL 和失败语义，但仍需为快照存储、网络重建、并发恢复和数据保留建立独立账本。

## 思考题

1. 用 `q` 表示等待比例，分别取 0.2、0.6、0.9，计算 10,000 个会话的可释放内存；再加入 20% 故障域冗余，判断 AutoPause 是否仍值得。
2. 设计一个防抖策略：连续空闲 5 分钟才暂停，恢复后 2 分钟内不再暂停。它对交互式 Agent 的延迟和存储有什么影响？
3. 如果暂停成功但 Master 元数据写入失败，或者 RPC 超时而 Cubelet 已经 PAUSED，平台应如何决定 Resume、重试和销毁？请画出状态转换和幂等键。

## 下章预告

挂起让资源账变便宜，但也让“发生过什么”变得更难重建。下一章从执行轨迹、文件操作和网络访问三类事件出发，设计 Agent 的行车记录仪，并说明日志采集点、留存成本和取证边界。

## 参考与源码

- [CubeSandbox 生命周期文档（v0.7.2）](https://github.com/TencentCloud/CubeSandbox/blob/f1aaa737fb3862202b1731e0e0d844c28779f930/docs/zh/guide/lifecycle.md)
- [CubeMaster pause/resume 实现](https://github.com/TencentCloud/CubeSandbox/blob/f1aaa737fb3862202b1731e0e0d844c28779f930/CubeMaster/pkg/service/sandbox/sandbox_resume_pause.go)
- [生命周期 sweeper](https://github.com/TencentCloud/CubeSandbox/blob/f1aaa737fb3862202b1731e0e0d844c28779f930/cube-lifecycle-manager/internal/sweeper/sweeper.go)
- [E2B Sandbox 文档](https://e2b.dev/docs/sandbox)
- [Kubernetes Pod 生命周期](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
