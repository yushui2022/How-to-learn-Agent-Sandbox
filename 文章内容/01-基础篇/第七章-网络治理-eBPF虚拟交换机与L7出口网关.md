# 第七章　网络治理：eBPF 虚拟交换机与 L7 出口网关

<!-- chapter-nav -->

[← 上一章](第六章-虚拟化底座-KVM-RustVMM与containerd-Shim-v2的三角关系.md) · [章节目录](../../README.md) · [下一章 →](第八章-基础篇综合推演-给一个企业场景做架构设计.md)
<!-- /chapter-nav -->



> 《从 0 到 1 学 Agent 沙箱》系列 ｜ 基础篇 · 第七章  
> 主案例：腾讯 CubeSandbox  
> 基线：CubeSandbox v0.7.2，commit `f1aaa737fb3862202b1731e0e0d844c28779f930`。本文把仓库中的 CubeVS/CubeEgress 实现和公开设计文档分开描述；不同版本的 map 名称和策略字段可能变化。  
> 前置：第六章（KVM、RustVMM、Shim v2）；后续：第十章（凭据安全）、第十三章（执行审计）。

## 引子：恶意代码开始扫描内网

一个数据分析 Agent 需要访问公开包仓库。它在沙箱里执行了几行代码，却把目标从 `pypi.org` 换成了 `10.0.0.12:6379`，开始扫描同一集群里的 Redis。这个请求如果只经过 Docker 的默认 NAT，可能会到达宿主机能够到达的网络；如果只在 L7 代理里写域名规则，非 HTTP 流量又可能绕开代理；如果只把所有数据包送进用户态代理，吞吐和延迟会受到影响。

网络治理真正要解决的是三件事：**沙箱能不能抵达某个地址，允许的请求是否应该经过更细的 L7 检查，平台能不能解释这次访问发生了什么。** CubeSandbox 把这三件事分成两层：CubeVS 在内核态处理 TAP、NAT、连接跟踪和 L3/L4 策略，CubeEgress 在用户态处理域名、SNI、Host、方法、路径、凭据和审计。

这不是“eBPF 取代代理”，而是“快速路径和深检查路径各司其职”。本章沿着一个数据包的旅程走一遍。

## 一、网络边界先画出来

每个沙箱拥有自己的 TAP 设备和内部地址。TAP 是宿主机看到的虚拟网卡端点，guest 看到的是自己的网络设备；两者之间没有 Linux Bridge 广播域，而是由 CubeVS 把数据包接到宿主机的 eBPF 数据路径。

CubeVS 的公开架构文档把三个程序放在三个边界：

| 程序 | 挂载位置 | 方向 | 主要职责 |
|---|---|---|---|
| `from_cube` | 每个 TAP 的 TC ingress | 沙箱 → 宿主机 | 策略判断、DNS 观察、SNAT、会话创建、ARP 代理 |
| `from_world` | 宿主机物理网卡的 TC ingress | 外部 → 宿主机 | 反向 NAT、端口映射、回包转发 |
| `from_envoy` | `cube-dev` 的 TC egress | 宿主机 → 沙箱 | CubeEgress 回包和宿主探针的 DNAT |

三个程序共享 pinned BPF maps。设备注册表把 sandbox IP、TAP ifindex 和元数据对应起来；会话表保存双向 NAT 状态；LPM trie 保存每个沙箱的 CIDR 策略；DNS map 保存域名策略和待匹配的 DNS 查询。Go 侧的 `cubevs` 包负责加载、固定、更新和回收这些 map，C/eBPF 程序负责每个数据包的快速路径。[CubeVS 网络模型](https://github.com/TencentCloud/CubeSandbox/blob/v0.7.2/docs/architecture/network.md)

把控制面和数据面分开后，策略更新不需要重新编译 BPF 程序。添加一个沙箱、放行一个 CIDR、创建一个端口映射，本质上是更新 map 和设备元数据；程序本身继续运行。

## 二、出站数据包：从 TAP 到外网

### 2.1 第一步：策略先于 NAT

沙箱发出的数据包以内部地址为源，进入自己的 TAP。`from_cube` 首先识别它属于哪个 ifindex，然后按以下顺序处理：

1. 访问沙箱默认网关的内部流量直接放行到宿主侧路径；
2. 检查 `allow_out`（当前源码中可见 `allow_out_v3` 等版本化 map）是否有匹配；
3. 如果没有允许项，再检查 `deny_out`；
4. 都没有命中时，根据默认策略决定放行或丢弃；
5. 只有通过策略的数据包才进入 DNS 观察、L7 引流或 SNAT。

当前文档描述的策略优先级是“allow 优先于 deny，未命中时默认放行”，并且总有一组私有、回环、链路本地网段被默认拒绝，用来降低访问宿主机、集群内网和其他沙箱的风险。要实现严格出网，通常需要安装全拒绝的 `0.0.0.0/0`，再用 allow 列表打孔。默认值不是安全结论，产品必须把 `allow_internet_access`、CIDR 和域名策略组合后再解释。

### 2.2 第二步：DNS 只负责把域名变成可判断的 IP

网络包最终要送到 IP，Agent 配置却通常是域名。CubeVS 的域名策略采用“观察 DNS 查询、学习响应地址”的方式：

- 沙箱发起 DNS 查询时，`from_cube` 检查域名是否在允许列表，并记录查询标识与源端口；
- DNS 响应回来后，`from_world` 找到对应查询，读取 A 记录；
- 每个解析到的 IP 以 DNS TTL 为生命周期写入该沙箱的允许 map；
- TTL 到期后，reaper 删除旧地址，避免 CDN 或动态服务的旧 IP 永久放行。

这层策略解决的是“这个域名解析出的地址能否进入 L3 快速路径”，不是完整的 HTTP 身份认证。它依赖真实的 DNS 查询；如果程序使用硬编码 hosts、DoH 或自带解析器，系统未必能看到域名，必须把这种情况当成策略盲区。域名策略只能放在 allow 侧，是因为系统需要把允许的域名转成短期 IP 许可，而不是用 deny 侧表达一个不稳定的地址集合。

### 2.3 第三步：SNAT 与会话表

通过策略的数据包还使用沙箱的内部源地址，不能直接在宿主机外部路由。CubeVS 在 `from_cube` 中为它分配 SNAT IP 和源端口，同时写入两张会话 map：

- `egress_sessions` 以沙箱侧原始五元组为主键，保存 SNAT 地址、端口、TAP ifindex、时间戳和 TCP 状态；
- `ingress_sessions` 以外部侧五元组为主键，保存恢复原始五元组所需的信息。

两张表不是重复存同一份状态，而是让出站和回包都能 O(1) 找到方向。TCP 会按连接状态设置不同超时；UDP 和 ICMP 使用更简单的活动/回包模型。reaper 定期清除过期项，避免长时间运行的 Agent 把 NAT 表耗尽。

源端口分配需要处理并发和冲突。实现中通过 per-IP 的水位和 BPF spin lock 保护更新，再检查新五元组是否已经在反向表中占用。这个设计把锁范围放在端口分配的短临界区，而不是给每个数据包加全局锁。它仍然有容量边界：可用 SNAT 地址、源端口和 BPF map 大小共同决定单节点能承受多少并发连接。

### 2.4 回包：`from_world` 只做它知道的反向工作

外部回复到达宿主机物理网卡后，`from_world` 先查 `ingress_sessions`。匹配成功，它把目的地址和端口还原为沙箱内部五元组，再重定向到对应 TAP。没有会话项时，如果目标端口命中了 `remote_port_mapping`，它把流量送入暴露该端口的沙箱；否则丢弃或交由宿主机正常处理。

这条路径说明了为什么“端口映射”和“出站 NAT”不能混成一个概念：出站连接需要状态化的双向会话，入站服务端口可以用静态映射把宿主端口送到指定 TAP。企业做审计时也要分开统计“沙箱主动连接”和“外部访问沙箱服务”。

## 三、为什么不用一长串 iptables 规则

iptables 并不是“不安全”，它的问题是当规则数和沙箱数一起增长时，管理与匹配成本容易失控。假设每个沙箱都有若干 NAT、端口、允许/拒绝和状态规则，规则链会随着租户增长；更新一条租户策略还可能触发全局规则重排或锁竞争。

CubeVS 的选择是：让每个沙箱拥有自己的 TAP 和 map 内层表，数据包在 eBPF 中通过 ifindex 定位沙箱，再在 LPM trie、哈希表和会话表中完成有限次查找。它的收益不是一个脱离硬件和 workload 的“固定倍数”，而是让策略状态按沙箱分片、让 fast path 留在内核，并避免共享 bridge 的广播和多跳。

代价也应说清楚：eBPF 验证器有指令和复杂度约束，map 的大小需要预留，错误的程序更新可能影响节点上的所有沙箱，调试比一条 iptables 规则困难。生产环境需要程序版本、map schema、加载失败回滚和节点级故障隔离，而不是只把“iptables 换成 eBPF”当成性能优化。

## 四、L7 出口：什么时候应该把包送到用户态

### 4.1 L3/L4 不知道请求意图

内核态可以快速判断目标 IP、端口和连接状态，却不知道 HTTPS 请求的 Host、TLS SNI、HTTP 方法和路径。对 Agent 而言，允许 `api.example.com` 的 `POST /v1/*`，和允许同一个 IP 上的任意端口、任意路径，是两个完全不同的授权。

CubeEgress 的设计是按需引流。规则标记某个域名、端口和 scheme 需要 L7 处理后，CubeVS 在连接路径上打 mark，把流量导向 `cube-dev`；宿主侧 TPROXY 将连接交给 OpenResty，代理保留原始目的信息并完成上游连接。没有命中 L7 规则的流量继续走普通 L3/L4 快速路径，不必为所有数据包付代理成本。

### 4.2 规则不是域名黑名单，而是有顺序的策略程序

CubeEgress 的规则可以按 SNI、Host、method、path、scheme 和端口匹配，动作可以是 allow、deny、audit 或 inject。文档当前采用 first-match-wins：第一条匹配规则决定结果，未命中时默认拒绝。规则顺序本身就是策略的一部分：更具体的允许例外要放在更宽的拒绝规则之前。

一个安全的企业策略通常分三层：

1. CubeVS 先挡住私有、回环和不允许的网段；
2. DNS 学习把允许域名的 IP 限定到短期地址集合；
3. CubeEgress 再按 HTTPS、SNI、Host、路径和方法做细粒度判断。

这三层并不是互相替代。L3 允许不等于 L7 允许；L7 代理允许也不等于可以访问所有非 HTTP 协议。策略文档必须写清楚“哪一层做最终拒绝”。

### 4.3 HTTPS 透明代理的信任前提

要读取 HTTPS 的 Host、路径和请求头，CubeEgress 需要在宿主侧终止 TLS，再与上游建立连接。CubeSandbox 的做法是为沙箱模板预置受信任的根证书，由代理现场签发叶证书。这样，沙箱内的客户端会把代理视为受信任的 TLS 终点。

这不是凭空增加安全，而是把信任从“任意外网服务”改成“平台代理 + 上游证书验证”。部署时要明确：

- 模板内的根 CA 是平台级信任材料，必须控制其分发和轮换；
- 上游证书验证仍然必须开启，否则代理可能把凭证发给伪造的地址；
- 不支持或不应解密的协议要走显式旁路策略；
- 审计记录不应把 Authorization、Cookie 和 Token 原文写入日志。

### 4.4 凭证注入：让密钥留在沙箱外

凭证注入是 CubeEgress 和普通代理最有价值的差异之一。Agent 请求 `api.example.com` 时，沙箱内不放 API Key；代理在完成 HTTPS、SNI、Host 和规则检查后，把 Authorization header 加到转发请求上。模型看不到环境变量里的 Key，也不能通过 `printenv`、读取文件或把 Key 写进 commit 的方式直接拿到它。

但“注入式方案免疫泄露”是错误的说法。它主要阻断“密钥进入 guest”这条路径，仍需要防止：

- 规则把凭证注入到了错误的域名或明文 HTTP；
- DNS 被控制，域名解析到攻击者地址；
- 上游服务本身返回或反射了秘密；
- 代理日志、配置中心或运维人员泄露了密钥；
- Agent 获得了一个允许调用高权限 API 的合法响应，并用它做了不该做的事。

因此 CubeEgress 的凭证能力应与 Vault/KMS 分工：Vault/KMS 负责保存、轮换和授权，代理负责在受限请求上短暂使用，沙箱负责永远不持有明文副本。

## 五、网络治理的三个边界

### 5.1 沙箱与宿主机

第一道边界是 TAP、内部地址、默认拒绝网段和 NAT。目标是让沙箱不能把宿主机或同节点其他沙箱当作普通内网服务。这个边界必须覆盖 IPv4、IPv6、回环、链路本地、DNS、元数据服务以及宿主机暴露的管理端口。只写一张 IPv4 deny 表是不够的。

### 5.2 沙箱与外网

第二道边界是出站策略。企业需要决定是默认允许后按风险阻断，还是默认拒绝后按业务域名放行。对于 Coding Agent，拉包、拉代码、访问模型 API 和上传结果通常需要不同的域名、方法和凭证，不能只给一个“允许互联网”的布尔值。

### 5.3 平台与证据

第三道边界是审计。每次放行、拒绝、注入、TLS 握手失败、DNS 学习和端口映射都应该能关联到 sandbox_id、tenant_id、policy_version 和时间。网络审计不是抓包越多越好：应记录足以回答“谁访问了什么、被哪条策略允许、有没有注入凭证、结果如何”的字段，并对敏感头脱敏。

## 六、横向参照

| 方案 | 网络策略特点 | 适合的位置 | 主要限制 |
|---|---|---|---|
| Kubernetes NetworkPolicy | 面向 Pod/Namespace 的 L3/L4 选择器 | 已有 K8s 的基础隔离 | 对单请求的 SNI、路径、凭证注入表达力有限 |
| Cilium/eBPF | 内核态策略、连接跟踪、可观测性和服务网络 | 大规模 K8s 网络数据面 | 仍需另配 L7 代理和 Agent 生命周期治理 |
| E2B | 平台侧网络和域名策略，SDK 直接使用 | 托管 Agent 沙箱 | 具体底层和企业自定义程度取决于部署形态 |
| CubeVS + CubeEgress | 每沙箱 TAP、eBPF L3/L4、按需 L7、凭证注入 | CubeSandbox 的节点级数据面 | 需要维护 BPF、代理、证书和策略版本 |

CubeVS 并不是要替代 Cilium 的完整集群网络，CubeEgress 也不是通用 API 网关。它们围绕“一个不可信 Agent 的出口”做了专门取舍。

## 七、企业视角：把“能联网”改写成可审计合同

企业网络需求不应写成“允许沙箱上网”，而应写成一份合同：

```text
任务：tenant-a / sandbox-123
允许：pypi.org:443（GET/HEAD，普通出站）
允许：api.example.com:443（POST /v1/*，代理注入 token-a）
拒绝：10.0.0.0/8、169.254.0.0/16、节点管理端口
记录：DNS、策略命中、上游地址、状态码、字节数、延迟
凭证：只允许 HTTPS，密钥由 Vault 提供，代理日志脱敏
```

这份合同可以同时服务安全、开发和财务：安全团队得到最小出口和取证字段，开发团队知道依赖下载与 API 调用为什么失败，财务团队可以按租户统计出站流量和代理成本。

对于多租户平台，策略还要有版本和回滚。策略更新应是沙箱级、原子可见的；一条租户规则错误，不能把节点上的所有 allow map 清空。L7 代理升级应支持灰度和旁路，BPF 程序升级应有加载失败保护。否则网络层会从“安全边界”变成新的全局单点。

## 本章小结

| 关键认知 | 一句话 |
|---|---|
| 三段 eBPF | `from_cube` 处理沙箱出站，`from_world` 处理外部入站，`from_envoy` 处理宿主侧回沙箱流量 |
| NAT 状态 | SNAT/DNAT 依赖双向会话表，端口和 map 容量是并发连接上限的一部分 |
| L3 与 L7 | eBPF 负责快速的 IP/端口/连接策略，CubeEgress 负责域名、SNI、Host、路径、凭证和审计 |
| DNS 学习 | 将允许域名的解析结果按 TTL 写入短期 IP 许可，但不等于 HTTPS 身份认证 |
| 凭证注入 | 阻止密钥进入 guest，不等于对错误规则、恶意上游和合法权限滥用免疫 |
| 企业治理 | “能联网”必须变成带租户、版本、证据和回滚的出口合同 |

## 思考题

1. 一个沙箱需要访问 `api.example.com:443`，但该域名每分钟切换 CDN IP。请说明 DNS 学习、LPM trie、L7 SNI 匹配各自解决什么问题，哪个环节过期会造成误放行或误拒绝。
2. 评估两种设计：所有出站流量都送到 L7 代理；只有命中规则的流量按需引流。请按每包 CPU、策略表达力、审计完整性和故障半径比较。
3. 设计一个凭证注入规则，要求只允许 `POST /v1/run`，禁止明文 HTTP，并且日志不能记录 Authorization。列出至少五个失败路径和对应的拒绝证据。
4. 一个节点有 10,000 个沙箱，每个沙箱 20 个并发 TCP 连接。你需要估算哪些 map、SNAT 端口、reaper 周期和日志存储容量？不要只用“连接数 × 规则数”一个数字回答。

## 下章预告

基础篇最后一章不再介绍组件，而是把前七章拼成一个企业架构。我们会分别推演多租户代码执行 SaaS、数据分析 Agent 和 Coding Agent 平台：同样的 CubeSandbox，为什么模板、网络、凭证、生命周期和容量答案完全不同。

---

> **参考与延伸**
> - CubeVS 网络模型：<https://github.com/TencentCloud/CubeSandbox/blob/v0.7.2/docs/architecture/network.md>
> - CubeEgress 安全代理：<https://github.com/TencentCloud/CubeSandbox/blob/v0.7.2/docs/zh/guide/security-proxy.md>
> - CubeSandbox 网络深潜：<https://github.com/TencentCloud/CubeSandbox/blob/v0.7.2/docs/zh/blog/posts/2026-06-23-cubesandbox-network-deep-dive.md>
> - Cilium eBPF 网络架构：<https://docs.cilium.io/en/stable/concepts/ebpf/intro/>
> - Kubernetes NetworkPolicy：<https://kubernetes.io/docs/concepts/services-networking/network-policies/>
> - E2B Runtime 网络与出口说明：<https://github.com/e2b-dev/runtime>
