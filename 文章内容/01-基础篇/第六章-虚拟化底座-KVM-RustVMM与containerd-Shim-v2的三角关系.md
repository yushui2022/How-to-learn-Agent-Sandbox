# 第六章　虚拟化底座：KVM、RustVMM 与 containerd Shim v2 的三角关系

<!-- chapter-nav -->

[← 上一章](第五章-控制面与数据面-一次沙箱创建的十步旅程.md) · [章节目录](../../README.md) · [下一章 →](第七章-网络治理-eBPF虚拟交换机与L7出口网关.md)
<!-- /chapter-nav -->



> 《从 0 到 1 学 Agent 沙箱》系列 ｜ 基础篇 · 第六章  
> 主案例：腾讯 CubeSandbox  
> 基线：CubeSandbox v0.7.2，commit `f1aaa737fb3862202b1731e0e0d844c28779f930`。代码与文档会继续演进，本文把“源码事实”和“架构解释”分开。  
> 前置：第五章（控制面、数据面和一次创建的十步旅程）。

## 引子：它到底是容器，还是虚拟机？

第一次接触 CubeSandbox，最容易产生一个分类错误：看到 `containerd`、OCI 镜像和 Shim，就把它当成一种更快的容器；看到 KVM、独立内核和 MicroVM，又把它当成一台缩小版虚拟机。两个判断各对了一半。

它的隔离边界来自 KVM 和 guest 内核，使用体验却借用了容器生态。上层可以继续谈镜像、任务、标准输入输出和 `exec`；下层真正运行的是一个拥有独立内核的客户机。**“既是 VM，又有容器体验”不是宣传语，而是三个组件共同完成的接口转换：KVM 画墙，RustVMM 造机器，Shim v2 把机器翻译成容器运行时听得懂的对象。**

本章只回答一个硬问题：这三个层次如何接在一起。我们先解释硬件虚拟化究竟隔离了什么，再看 RustVMM 的精简设备模型，最后钻进 CubeShim，说明 containerd 为什么可以管理一台并不共享宿主内核的 MicroVM。网络策略、快照算法和并发调度只在需要定位边界时提到，不在本章展开。

## 一、KVM 画出的那道硬件墙

### 1.1 Guest 不是一个被“限制的进程”

普通进程运行在宿主机内核建立的地址空间里。即使进程有独立的 namespace，它执行的系统调用仍然进入同一个宿主内核。KVM 的模型不同：宿主机内核通过 `/dev/kvm` 创建虚拟机对象和 vCPU，guest 看到的是一组虚拟 CPU、虚拟内存和虚拟设备。guest 内核在自己的地址空间中运行，宿主机不能把它简单地当成一个共享内核下的进程组。

在 x86 上，可以把 CPU 的执行分成两个权限世界来理解：宿主机在 VMX root 模式运行 VMM 和 KVM，guest 在 VMX non-root 模式运行。guest 执行普通指令时，处理器可以直接运行；当它执行需要宿主机介入的操作，例如访问尚未处理的虚拟设备、修改虚拟化控制状态或触发某些异常，处理器才发生 VM exit，把控制权交回 KVM/VMM。**普通 guest 内存访问并不会因为“访问了内存”就每次 VM exit。**

这一区别很重要。把 VM exit 解释成“guest 每走一步都回到宿主机”，会错误地推导出 MicroVM 必然很慢；把它解释成“guest 永远碰不到宿主机”，又会掩盖 VMM、KVM、硬件和侧信道的真实风险。虚拟化的性能来自尽量让 guest 在硬件上直接运行，虚拟化的安全来自让敏感操作只能通过受控出口离开 guest。

### 1.2 EPT/NPT：两次地址翻译，而不是一堵魔法墙

guest 程序首先使用 guest 虚拟地址。guest 内核的页表把它翻译成 guest physical address（GPA）；硬件扩展页表 EPT（Intel）或 NPT/RVI（AMD）再把 GPA 翻译成 host physical address（HPA）。VMM 通过 KVM API 安装这些 guest 内存区域，控制哪些 GPA 真的对应到宿主机的哪些页。

这道二级地址翻译带来三层工程效果：

1. guest 页表可以在 guest 内部变化，宿主机不必为每次普通进程访问模拟一遍；
2. 一个 VM 的 GPA 空间和另一个 VM 不直接相连，越界访问通常会在 guest 或二级页表边界处失败；
3. VMM 可以按内存区域管理权限、快照和写时复制，而不是把整台物理机交给 guest。

但“独立 GPA 空间”不等于“无风险”。KVM、VMM 的设备模拟、CPU 微架构侧信道、宿主机内核以及 guest 自身漏洞，仍然在威胁模型中。MicroVM 做的是缩小暴露面和限制故障半径，不是把安全问题从世界上删除。

源码入口可以从 CubeHypervisor 的 VM 配置与内存管理模块开始看：仓库中的 `hypervisor/vmm/src/vm.rs` 负责 VM 状态、启动、暂停和恢复，`hypervisor/vmm/src/memory_manager.rs` 负责 guest 内存区域和快照相关数据结构。[v0.7.2 VM 实现](https://github.com/TencentCloud/CubeSandbox/tree/v0.7.2/hypervisor/vmm/src)

## 二、RustVMM：不是一台 VMM，而是一套造 VMM 的零件

### 2.1 Firecracker、Cloud Hypervisor 与 CubeHypervisor 的血缘

RustVMM 经常被误解成某个具体的虚拟机产品。更准确的说法是：它是一组用 Rust 编写的虚拟化组件和共同设计方法，帮助项目组装 CPU、内存、virtio 设备、KVM 通道、快照和安全策略。

Firecracker 以 Serverless 为目标，把设备集合压到很小，用少数 virtio 设备和专用控制面换取启动速度、密度与攻击面收敛。Cloud Hypervisor 在同一生态中提供更完整、更可配置的云工作负载能力，设备、架构和迁移能力的选择更多。CubeHypervisor 则在 Cloud Hypervisor 基础上面向 CubeSandbox 的模板恢复、快照、网络和资源模型继续定制。这里的“血缘”表示代码和设计思想的传承，不表示三个项目的配置、性能或安全结论可以直接互换。

一个有用的比较方式是看“谁承担复杂度”：

| 路线 | 把复杂度放在哪里 | 适合的工程目标 |
|---|---|---|
| Firecracker | 通过严格的设备最小集限制功能面 | Serverless、强约束、可预测的 VM |
| Cloud Hypervisor | 提供更完整的设备和架构选择 | 云主机、通用虚拟化、可扩展配置 |
| CubeHypervisor | 在上游能力上加入 Cube 的快照、恢复与 Agent 语义 | 会话级、状态化、批量派生的 Agent 沙箱 |
| QEMU/KVM | 用成熟、完整的设备模型换取兼容性 | 通用 VM、旧操作系统、设备兼容优先 |

表格不是产品排名。一个设备模型越完整，能跑的 guest 越多，但初始化、测试和安全审计的范围也越大。对 Agent 沙箱来说，“少一个不需要的设备”通常比“多支持一种旧系统”更有价值。

### 2.2 精简设备模型的两次收益

MicroVM 的精简不是为了把 VM 做成“功能残缺的玩具”，而是把执行者真正需要的接口保留下来。典型设备包括 virtio block、virtio net、vsock、串口，以及平台启动所需的最小控制设备。一个 Agent 通常需要读写文件、访问网络、和宿主侧 agent 通信，却不需要模拟一套完整 PC 主板、声卡、USB 控制器和各种历史设备。

每砍掉一个设备，至少减少两种成本：

- **启动成本**：设备初始化和 guest 驱动探测变少；
- **审计成本**：设备后端、状态机和异常路径变少，安全审计的边界更清楚。

这不是说 virtio 天生安全。virtio 后端仍是 VMM 的攻击面，设备配置和宿主侧文件、网络后端仍然需要参数校验、权限限制和 seccomp。正确的结论是：**最小设备模型让“需要被证明安全的东西”变少了。**

### 2.3 PVH 直启：减少固件路径，但不是免费安全

传统 VM 通常经历固件、引导加载器、内核的启动链。CubeHypervisor 支持直接装载 Linux 内核，x86 路径可以使用 PVH 启动协议。这样做可以跳过一部分固件初始化，让启动路径更短，也避免把完整 BIOS/UEFI 设备模型引入每个沙箱。

这里有两个容易被写过头的句子。第一，PVH 是一种启动路径，不代表所有 guest、所有架构和所有配置都必须使用它。第二，去掉固件可以减少固件代码面，但不能推出“没有启动安全问题”：内核、initramfs、命令行、VMM 和 guest agent 仍然决定启动后的边界。Firecracker 的官方文档也把 PVH 描述为可用的直接启动模式，而不是唯一启动协议。[Firecracker PVH 文档](https://github.com/firecracker-microvm/firecracker/blob/main/docs/pvh.md)

## 三、Shim v2：把 MicroVM 翻译成容器任务

### 3.1 containerd 为什么不需要知道所有 VM 细节

containerd Runtime v2 的关键思想是：containerd 不把每个容器的全部执行逻辑塞进自己进程，而是为运行时启动一个独立的 shim。shim 负责保存任务生命周期、连接标准输入输出、转发信号，并通过 ttrpc 等协议与更底层的运行时通信。这样，containerd 的上层对象仍然是 task，底层可以是 runc，也可以是 Kata、Firecracker 或 CubeSandbox 的 MicroVM。

CubeShim 的位置可以画成这样：

```text
containerd
    │  Shim v2 API：Create / Start / Exec / Kill / Delete
    ▼
containerd-shim-cube-rs  —— CubeShim
    │  ttrpc / vsock
    ▼
cube-agent（guest 内）
    │
    ▼
进程、文件系统与应用服务
```

CubeShim 向上实现 containerd Shim v2 接口，向下负责准备 rootfs、内存文件和内核，驱动 CubeHypervisor 创建或恢复 VM，再通过 guest 内的 `cube-agent` 执行任务。containerd 因此不需要理解 KVM 的 `KVM_RUN`、virtio 设备或快照文件；它只看到一个可以启动、执行、发送信号、读取输出并删除的运行时任务。

CubeShim 自己的 README 明确描述了这条桥：它用 Rust 实现 Shim v2，向上接 containerd，向下通过 ttrpc 与 VM 内的 agent 通信，并把 `Create / Start / Exec / Kill / Delete` 等生命周期调用转发进去。[CubeShim README](https://github.com/TencentCloud/CubeSandbox/blob/v0.7.2/CubeShim/README.md)

### 3.2 这座桥解决了什么问题

**第一，复用生态。** 企业已有 containerd、OCI 镜像、镜像仓库、日志和运行时接口，不必为每一种隔离底座再造一套完全不同的 API。

**第二，隔离进程。** VMM 不需要和 containerd 共用一个长生命周期进程。CubeShim 崩溃时，containerd 可以按运行时语义感知任务失败；VMM 又有自己的 seccomp 和生命周期边界。

**第三，统一 Agent 的操作面。** Agent 应用通常不关心自己在 runc 容器、gVisor 沙箱还是 MicroVM 里。它需要的是文件、进程、端口、标准输入输出和结果回传。Shim 把这些稳定动作映射到底层实现。

**第四，保留 VM 的状态能力。** 传统 OCI 运行时擅长创建和销毁进程组；CubeShim 还要处理 VM 暂停、恢复、快照和 guest 状态。它让容器接口成为入口，而不是把 MicroVM 强行简化成一次性进程。

### 3.3 桥接也会引入新的失败面

兼容接口不是没有代价。一个 `Kill` 到底是结束 guest 内的主进程，还是关掉整台 VM？一个 `Delete` 失败后，磁盘、快照和网络设备谁负责清理？标准接口通常只规定语义，不规定跨越 host、VMM、guest agent 的补偿顺序。

因此 CubeShim 的难点不在于“把函数名翻译一下”，而在于保持两个状态机的一致：containerd 认为 task 已经 `RUNNING` 时，guest agent 必须可用；containerd 请求删除时，VMM、TAP、rootfs 和快照资源必须有可重复执行的清理路径；网络抖动时，重试不能把一个 VM 启动两次，也不能把仍在运行的任务误判为孤儿。

这解释了为什么 Shim v2 是“桥”而不是“胶水”。它承接了上层生态的稳定性承诺，同时承担了底层虚拟机状态机的一部分复杂度。

## 四、rootfs 从哪里来：从 OCI 镜像到可恢复模板

一个可用的 Agent 沙箱至少需要三类输入：guest 内核、rootfs 和运行时状态。CubeSandbox 的文档把模板流程概括为：OCI 镜像经过构建引擎生成 rootfs，再准备冷启动或内存快照，最后登记为可派生的模板。

可以把它拆成六步：

1. **选择 OCI 输入**：确定基础发行版、Python/Node/编译器、guest agent 和证书策略；
2. **构建文件系统**：BuildKit 等构建链把镜像层转换为沙箱需要的 rootfs；
3. **写入 Agent 运行时**：guest 内必须有能接收命令、处理文件和报告状态的 agent；
4. **配置启动参数**：VMM 需要内核、命令行、rootfs、网络和 vsock 等配置；
5. **准备模板状态**：对需要快速派生的模板，可以预先启动服务并保存内存/设备状态；
6. **节点分发与本地派生**：运行节点拿到可读的模板副本后，在本地创建 rootfs 和内存的 CoW 版本。

第六步是性能和运营的交叉点。模板如果只放在远端共享存储，创建沙箱会把网络 IO 加进关键路径；如果每个节点保存一份副本，启动更稳定，但节点要承担本地磁盘和模板同步成本。CubeSandbox 的公开架构文档明确把模板 rootfs、内存快照、CubeCoW 和节点本地数据面放在一起讨论。[架构总览](https://github.com/TencentCloud/CubeSandbox/blob/v0.7.2/docs/architecture/overview.md)

模板也不是“冻结后的万能镜像”。内核版本、guest agent、证书、网络策略和应用依赖都可能变化。企业要为模板建立版本、兼容性和回滚关系：应用层认为“模板升级成功”，不等于旧快照可以继续被新 VMM 恢复。

## 五、横向参照：三种方式把 VM 带进容器生态

Firecracker 选择了极简 VMM 和专用 API；Kata Containers 选择把轻量 VM 纳入 Kubernetes/containerd 运行时；CubeSandbox 选择在 RustVMM/KVM 和 E2B 兼容接口之上，增加模板、快照、网络和 Agent 生命周期语义。

三者的差别可以用一句话概括：

- Firecracker 主要回答“怎样安全且快速地启动大量 MicroVM”；
- Kata 主要回答“怎样让 Kubernetes 的 Pod 运行在 VM 隔离边界内”；
- CubeSandbox 主要回答“怎样把一个有状态 Agent 的完整工作环境按需创建、复用、暂停、恢复和回滚”。

这也是为什么不能只比较“谁的启动更快”。如果企业只需要 K8s 中的硬件隔离，Kata 的生态集成可能更重要；如果要从 SDK 创建一个有状态、可快照的 Agent 工作空间，CubeSandbox 的控制面和状态语义才是关键；如果是 Serverless 函数且设备模型高度固定，Firecracker 的极简路径可能更合适。

## 六、企业视角：复用容器生态意味着什么

对企业而言，Shim v2 的价值不是让产品看起来像容器，而是把存量组织能力复用到更强的隔离边界上：镜像构建团队继续维护 OCI 镜像，平台团队继续使用 containerd 和节点监控，安全团队可以沿用镜像扫描、运行时审计和权限审查，Kubernetes 团队也能理解任务和节点的生命周期。

但复用接口不等于复用全部假设。容器默认的“进程很快结束”在 Coding Agent 中可能变成长时间会话；Pod 的重建语义不等于 VM 状态的回滚语义；共享内核的资源计量也不能直接代替 guest 内存和 VMM 开销的计量。企业落地时至少要重新定义：

- task 的成功是进程退出，还是 Agent 环境可用；
- delete 是销毁文件，还是同时删除快照、卷、网络和审计证据；
- 资源配额按容器请求、guest RAM、VMM overhead，还是三者之和计算；
- K8s 的调度、审计和升级哪些可以复用，哪些必须进入 CubeSandbox 的控制面。

## 本章小结

| 关键认知 | 一句话 |
|---|---|
| KVM | 通过硬件辅助虚拟化、二级地址翻译和受控 VM exit，把 guest 从共享内核进程组变成独立客户机 |
| RustVMM | 是一套组装 VMM 的组件和方法，不等于单一产品；设备模型越小，启动和审计边界通常越可控 |
| PVH | 是减少固件路径的启动方式，不等于绝对安全，也不是所有配置的唯一启动方式 |
| Shim v2 | 向上呈现 containerd task，向下驱动 MicroVM 与 guest agent，是容器体验和 VM 隔离的桥 |
| 模板 | OCI 镜像只是输入，真正可快速派生的模板还包含内核、rootfs、agent 和预热状态 |
| 企业复用 | 复用容器接口能降低生态迁移成本，但不能照搬共享内核和短进程的运营假设 |

## 思考题

1. 如果 guest 在执行普通内存访问时每次都 VM exit，单个 Agent 的系统调用和文件 IO会发生什么变化？哪些操作应该留在 guest 内完成，哪些操作必须交给 VMM？
2. 你要同时支持一个需要旧内核设备的企业应用和一个只运行 Python 的不可信 Agent，应该使用完整 QEMU、Cloud Hypervisor、Firecracker 还是 CubeSandbox？请把“兼容性、攻击面、启动成本、状态能力”分别打分，并写出不采用另外三种的理由。
3. containerd 把 task 标记为 `RUNNING`，但 guest agent 尚未完成健康检查。你的 Shim 应该延迟状态上报、报告运行但不可用，还是直接失败？不同选择对重试、审计和用户体验有什么影响？
4. 一个模板在节点 A 可从内存快照恢复，在节点 B 只有 rootfs 没有快照。控制面应把它们视为同一个模板，还是两个能力等级不同的模板版本？

## 下章预告

KVM 解决了“墙是什么”，Shim 解决了“怎么接进生态”，但沙箱一旦联网，真正的出口又在哪里？下一章进入 CubeVS 和 CubeEgress：三段 eBPF 程序如何在内核态完成 TAP、NAT、连接跟踪和策略匹配，L7 网关又怎样把域名过滤、凭据注入与审计接起来。

---

> **参考与延伸**
> - CubeSandbox v0.7.2 架构总览：<https://github.com/TencentCloud/CubeSandbox/blob/v0.7.2/docs/architecture/overview.md>
> - CubeShim v0.7.2 README：<https://github.com/TencentCloud/CubeSandbox/blob/v0.7.2/CubeShim/README.md>
> - CubeHypervisor VM 实现：<https://github.com/TencentCloud/CubeSandbox/tree/v0.7.2/hypervisor/vmm/src>
> - Firecracker PVH 文档：<https://github.com/firecracker-microvm/firecracker/blob/main/docs/pvh.md>
> - containerd Runtime v2 设计：<https://github.com/containerd/containerd/tree/main/runtime/v2>
> - Kata Containers 架构文档：<https://github.com/kata-containers/kata-containers/tree/main/docs>
