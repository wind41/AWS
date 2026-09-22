# AWS Route 53 SME - Exam Bank & Mock Tests

> Question banks, difficulty guides, and mock exam findings.

---

# FILE: exam/codex-review-findings-gap.md
<!-- SOURCE FILE: exam/codex-review-findings-gap.md -->

# Codex 校验 findings — 缺口补充域 21/22/23（2026-09-20）

## topic 21 Cloud Map / 服务发现
核心正确：Public DNS namespace=public HZ+DNS/API；Private DNS=PHZ+DNS/API；HTTP namespace 不建 HZ 只 API discovery。修正：
1. WEIGHTED 补 fail-open（无健康实例时按全部实例处理）；Alias 必须 WEIGHTED；CNAME 必须 WEIGHTED 且不能配 HealthCheckConfig。
2. "标准 R53 HC 仅适用 public namespace" 错误 → HealthCheckConfig 支持 Public DNS 和 HTTP namespace，不支持 Private DNS；custom health 不是"仅限 VPC 资源"；二者不能同时配。
3. "awsvpc→A" 过度简化 → ECS service discovery awsvpc 支持 A 或 SRV；bridge/host 才只能 SRV。
4. "Service Connect 用 HTTP namespace 并自动增删 DNS 记录" 错误 → 新建时创 HTTP 类型但可引用任何现有 namespace；Service Connect 用 Cloud Map API 注册表 + 托管代理，不靠 DNS 记录增删。

## topic 22 迁移与批量变更
核心正确：ChangeResourceRecordSets 整批原子成功/失败；INSYNC = 全部 R53 权威已更新，不代表递归缓存过期。修正：
1. "Zone import 仅适合空 zone" 过绝对 → 控制台原生导入最多 1000 条；非空 zone 可导入，但文件含已存在记录则整次导入失败。
2. **"单请求最多 1000 个变更" 错误 → 上限是 1000 个 ResourceRecord 元素（Alias 也计入）+ 所有 Value 共 32000 字符**，非按 Changes[] 数量。
3. UPSERT 双层双倍 → 请求大小计算时每个 RR 元素及 Value 字符计两次；change-throughput 每个 UPSERT 消耗 2 token，桶 burst 1500/持续 100 changes/s。
4. NS 切换 → 旧服务与新 zone 先降 NS TTL 至 60-900s 等旧 TTL 过期，复制验证后改注册商委派；旧权威至少保留 48h，父区缓存仍可能持续两天。
5. "每批必须等 INSYNC 才提交下一批" 非硬要求 → 是安全串行策略；并发未处理完会收 PriorRequestNotComplete。

## topic 23 企业跨区跨账号架构
（Codex 校验被截断。修正 subagent 请按官方核对：hub-spoke 是 RAM 共享 Resolver **rule** 非 endpoint 本身、消费账号关联自己 VPC；跨账号 PHZ 用 CreateVPCAssociationAuthorization + AssociateVPCWithHostedZone；Profiles vs 逐 VPC vs Resolver rule 选型边界；混合云 outbound endpoint 条件转发。有不准处一并改。）


================================================================================

# FILE: exam/codex-review-findings-p1.md
<!-- SOURCE FILE: exam/codex-review-findings-p1.md -->

# Codex 校验 findings — P1 新域 00a/15/16（2026-09-20）

## topic 00a DNS 协议基础
核心正确：glue/in-domain、NXDOMAIN vs NODATA、负缓存公式、AA≠AD。修正：
1. **SOA MINIMUM 默认 86400 秒（不是 900）**；SOA 记录 TTL 默认 900；有效负缓存 = min(SOA MINIMUM, SOA TTL) = min(86400,900)=900 秒。改正文里"SOA MINIMUM 常见 900"的说法。
2. "stub 必发 RD=1 / recursive 对上游必发 RD=0"过绝对 → RD 只表达是否请求递归，不标识角色；forwarder 转发可带 RD=1，迭代查询通常 RD=0。
3. RFC 9471 措辞 → 要求返回所有 available in-domain glue，放不下则 TC=1；不新增"registry/zone 必须预先配置全部 glue"的要求。
4. in-domain vs sibling → RFC 9471 正式区分二者；sibling 不等同任意域外 NS；bailiwick 取决查询上下文。
5. TC=1 后 → 改用"另一种 transport"（经典客户端必须支持 TCP）。
6. EDNS0 声明值 = requestor 可接收的最大 payload，不是硬截断阈值（还受服务端上限/路径 MTU/分片影响）。
7. CD=1 = 请求递归器不验证，但仍受实现/本地策略/BAD cache 影响，非"保证返回原始数据"。
8. 实验命令：普通 EDNS 查询 flags 不等于 DO；要显式 `+dnssec` 才置 DO 位。改实验里 `flags: do` 的示例。

## topic 15 可观测性
核心正确：HealthCheckStatus/HealthCheckPercentageHealthy/ConnectionTime/TimeToFirstByte/SSLHandshakeTime 真实存在（AWS/Route53）；百分比仅 endpoint HC；InboundQueryVolume/OutboundQueryVolume/OutboundQueryAggregateVolume 存在；query logging 可投 CloudWatch Logs/S3/Firehose 且同 Region。修正：
1. **拼写：`EndpointUnHealthyENICount` → `EndpointUnhealthyENICount`**。
2. 容量判断优先看 `ResolverEndpointCapacityStatus`（ENI Count 指标只表 OPERATIONAL/AUTO_RECOVERING 状态）。
3. query logging 粒度不固定到实例/ENI → 含 srcaddr，srcids 可含 instance/resolver endpoint/RNI，取决查询路径；缓存命中不记录。

## topic 16 成本模型
（Codex 校验被截断，未列出新错误；后续如需再单独校验。核心数字 hosted zone 前 25 个 $0.50/月、health check AWS 内免费外收费、resolver endpoint 按 ENI 小时 方向正确。）


================================================================================

# FILE: exam/codex-review-findings-p2.md
<!-- SOURCE FILE: exam/codex-review-findings-p2.md -->

# Codex 校验 findings — P2 新域 18/19/20（2026-09-20）

## topic 18 Traffic Flow
核心正确：policy=版本化配置树、policy record=API policy instance、仅 public zone、隐藏底层记录、未关联 policy 免费、instance $50/月按部分月折算、默认 50 policies/账号 + 1000 versions/policy + 5 instances/账号（policies/instances 可提额）。修正：
1. "复杂 alias 树"过窄 → Traffic Flow 编排的是同一 DNS 类型的路由记录树，节点可为规则或 endpoint，不要求都是 Alias。
2. "秒回滚"易误导 → UpdateTrafficPolicyInstance 有控制面处理时间，应轮询 instance state；递归缓存仍使客户端看到旧答案至 TTL 到期。
3. "Alias 指向根 policy record 只付一份 $50"不完整 → 只产生一条 policy-record 月费，但这些 Alias/CNAME 查询仍按 DNS query 计费。
4. Geoproximity：2024-01 起可直接在 public/PHZ 创建；不要说交互地图是 Traffic Flow "唯一剩余价值"（还有版本化/嵌套/跨 zone 复用）。

## topic 19 IAM 治理
核心正确：route53:* / route53resolver:* / route53profiles:* action 命名空间不可混写；DNSSEC KMS key policy 授权 dnssec-route53.amazonaws.com 执行 DescribeKey/GetPublicKey/Sign + 带 kms:GrantIsForAWSResource=true 的 CreateGrant，可加 SourceAccount/SourceArn；RAM 可共享 Resolver rule/Route53 Profile/DNS Firewall rule group（共享配置对象非 endpoint 本身，消费账号仍需关联自己 VPC）。修正：
1. "List/Get 都只能 Resource:*" 错误 → 逐 action 判断；ListHostedZones 用 *，但 GetHostedZone/ListResourceRecordSets/GetDNSSEC 等支持 hosted-zone ARN。

## topic 20 Resolver 高级
（Codex 校验被截断未列新错误。修正 subagent 请按官方现行文档核对以下点的准确性：Resolver delegation rules 2024 新增、autodefined/system rules、DoH/DoH-FIPS、IPv6 dual-stack endpoint、DNS64 需搭 NAT Gateway、Resolver on Outposts、outbound endpoint 每 ENI 约 10K QPS 且建议 ≥2 AZ。有不准处一并改。）


================================================================================

# FILE: exam/codex-review-findings-topic14.md
<!-- SOURCE FILE: exam/codex-review-findings-topic14.md -->

# Codex 审校 findings — topic 14 技术原理深挖（2026-09-20）

核心确认正确：stub/recursive/authoritative 分层、Route53 anycast+四 stripe、VPC Resolver 不发 ECS、18% 聚合规则、DS→DNSKEY→RRSIG 信任链方向正确。以下需修正：

### 递归解析路径
1. "TLD 委派必须附带四个 awsdns glue，否则死循环"错误 → glue 只对 in-bailiwick（委派域内）NS 必需；awsdns-* 是委派域外名称，可独立解析，不依赖该 child delegation 的 glue（RFC 9471）。
2. "AA=1 表示应答可信"易误导 → AA 只表示该服务器对此答案有权威性，不提供密码学真实性；只有 DNSSEC 验证成功 + AD 标志才表示 authenticated data（RFC 4035）。
3. "只有递归解析器缓存，stub 不缓存"过绝对 → OS/本地代理/应用都可能缓存；VPC Resolver(Nitro) 也维护本地短期缓存，上游故障时甚至可能提供超 TTL 缓存答案。
4. "Route53 权威侧改动秒级生效/几乎总已更新"过强 → 变更通常 60 秒内传播到全部 R53 NS，应以 GetChange=INSYNC 确认，不能只归因递归 TTL。
5. "Public/Private hosted zone 都直接对应权威 NS"需区分 → PHZ 数据经 VPC Resolver 私有连接访问；直接查其公开 awsdns 地址不会返回 PHZ 数据。

### Anycast 与四组 NS
6. "全球 50+ edge / 约 80% resolver 选最低 RTT" 是 2014 架构博客测量 → 只能作设计背景，不能当权威现行数字断言。

（其余 VPC Resolver/health checker/DNSSEC 密码学部分 Codex 未列出新错误，视为方向正确。）


================================================================================

# FILE: exam/codex-review-findings.md
<!-- SOURCE FILE: exam/codex-review-findings.md -->

# Codex 技术审校 findings（cx ask 咨询式，2026-09-20）

审了 topic 03 路由策略 + topic 07 DNSSEC。以下为需修正项，每条已带正确说法。其余核心表述 Codex 确认成立。

## Topic 03 — 路由策略（11 项）
1. **Simple 不是"最多返回 8 个值"**：8 值上限属 Multivalue。Simple 返回 RRset 全部值并随机排序；单 RRset 配额 400 条。删掉正文表/速记/Mermaid 里的"Simple ≤8"。
2. **Latency 目标不是"必须在 AWS Region"**：可指向 AWS 外资源，但记录仍须指定一个 AWS Region；延迟依据是用户到 AWS 数据中心，故对外部资源可能不准。
3. **Failover Primary 不是"无健康检查就无法创建"**：无 HC 的记录被视为始终健康，故 Primary 不会切到 Secondary。Alias Primary 可用 EvaluateTargetHealth。改为"要有效故障转移，Primary 需 HC 或 ETH"。
4. **Failover fail-open 条件不全**："两者均不健康返回 Primary"仅当 Primary 和 Secondary 都真正参与健康评估；Secondary 无 HC 时始终健康，不存在"两者均不健康"。
5. **Weighted 0 权重过度简化**：仅当所有非零权重记录不健康时才考虑零权重记录；零权重记录本身仍受 HC 影响；全部不健康时 fail-open 后仍按权重选，零权重不自动成唯一兜底。
6. **Geoproximity 不再"必须用 Traffic Flow"**：2024 起支持普通记录/API/CLI/SDK 创建；仅地图可视化仍限 Traffic Flow。
7. **Bias 范围应含 0**：有效 -99~99；0=无偏移，正值扩大区域，负值缩小。
8. **Multivalue 不是"只返回健康记录"**：正常最多返回 8 条健康记录；全部不健康时 fail-open 返回最多 8 条不健康记录。
9. **IP-based 掩码**：普通条目 IPv4 /1-/24、IPv6 /1-/48；默认位置 * 可表示 /0（::/0）。
10. **ECS 检测域名占位符写错**：应为正确的 knowledge-center route-53 ECS 支持链接（占位符 `<domain>` 需替换真实域名）。
11. **实验里"必须直查权威 NS、并发约 10K 次"是编造**：非 AWS 要求；weighted 是概率分布，样本量按统计目标定；shell 顺序循环≠并发。删除此绝对化表述。

## Topic 07 — DNSSEC（核心正确，8 项精度修正）
核心确认正确：KSK/ZSK 分工、DS 在父区、禁用前先删父区 DS、KSK 的 KMS 密钥须 asymmetric ECC_NIST_P256、island of trust。
1. "KSK 签署 ZSK"→ KSK 签署包含 KSK/ZSK 的整个 DNSKEY RRset；ZSK 签署其余 RRset。
2. "DS 包含 KSK 公钥"→ DS 是子区 DNSKEY 的摘要+算法元数据，不含完整公钥。
3. "DNSSEC 加密 DNS 流量"→ 只提供来源认证与完整性，不加密（无机密性）。
4. "权威 DNS 执行 DNSSEC 验证"→ 权威服务器提供签名数据，验证通常由递归解析器完成。
5. "RRSIG 对单条记录签名"→ RRSIG 对同名同类型的整个 RRset 签名。
6. "签名有效期等同 TTL"→ TTL 控缓存；RRSIG inception/expiration 独立控签名有效期。
7. "NSEC3 彻底阻止区域枚举"→ NSEC3 仅提高枚举成本，不能保证阻止离线破解。
8. **补充关键考点**：Route 53 DNSSEC KSK 所用的 KMS 密钥**必须位于 us-east-1**（且 asymmetric ECC_NIST_P256）。

## 第二批抽查（topics 02/04/05，2026-09-20）

### Topic 02 — records-and-alias
1. **删除"Simple RRSET 每次最多返回 8 值"**（1.2 节）：Simple 返回全部值随机排序；8 值上限是 Multivalue；单 RRset 配额 400。→ 这是从 research 文件传导来的同一错误，需连根改。
2. **Alias 可以指向另一条 Alias（Alias 链受支持）**：不要写"target 不能是另一条 Alias"。同 zone 目标通常要求同类型；zone apex 不能最终指向 CNAME。整条 Alias 链若末端是合格 AWS 资源仍免费。
3. **"只有 Alias 能建在 apex"表述纠正**：apex 可有普通 A/AAAA/MX/TXT；准确说法=**CNAME 不能位于 apex，Alias 可以**。
4. **不要泛化"所有 AWS 目标都用 A/AAAA-Alias"**：类型取决于目标（CloudFront 启用 IPv6 才加 AAAA；API Gateway/接口端点通常 A）。
5. **Alias 目标列表不完整**：现还含 VPC Lattice 等；表注明"示例"或补全。

### Topic 04 — health-checks
1. **间隔/失败阈值补全**：Endpoint HC 间隔仅 10s/30s，默认 30s 且创建后不可改；FailureThreshold 连续 1–10 次，默认 3。
2. **18% 规则限定范围**：>18% checker 判健康才算健康，仅适用于全球 checker 的 Endpoint HC，不适用于 Calculated/CloudWatch。
3. **响应时间限定**：HTTP(S) 4s 建连 + 连接后 2s 内收 2xx/3xx；TCP 10s 建连；字符串检查还须状态码后 2s 内收 body。
4. **字符串匹配/SNI**：搜索串区分大小写，须完整位于 body 前 5,120 bytes；HTTPS 开 SNI 把 FQDN 放进 TLS ClientHello，首次证书名不匹配会重试且省略 SNI。
5. **"只有三类 HC"已过时**：还有 ARC RECOVERY_CONTROL；应分述 Endpoint / Calculated / CloudWatch / Recovery Control。
6. **Calculated/CloudWatch 修正**：Calculated 按健康子检查数与 HealthThreshold 比较；CloudWatch 读 alarm 数据流而非强制 alarm state，字段是 InsufficientDataHealthStatus，不支持跨账号 alarm。

### Topic 05 — resolver（沿用重点核对项，按官方文档修正转发规则优先于 PHZ、.2/VPC+2、inbound/outbound、split-horizon、autodefined rules 的任何不准确表述）

## 第三批抽查（topics 08-13，2026-09-20）

### Topic 08 — subdomain-takeover
- 核心正确：Route 53 仅保护 Scenario 1；正确退役顺序=先删父区 NS→等 TTL→再删 child zone。
- CNAME 向量不要把 ELB 列为可抢注目标；当前典型可利用目标=S3、Elastic Beanstalk、AWS 重分配域名时的 CloudFront。
- StopZoneSniping 内部实现（DaasPurgeDelegations/周期重评估/backfill）不是公开契约，删掉；公开结论只到 Scenario1 阻止重叠 NS 分配、删父区委派后 hold 移除、Scenario2-5 不保护。
- "SOA serial=1 是修复指纹"不可靠（默认示例值1，不自动递增、可改）；改为核对父区委派与目标 NS 是否真权威。
- "DNSSEC 启用后 TTL 强制一周"过宽 → 一周是签名 zone 的最大有效 TTL；原 TTL<一周的记录不受影响。
- Lab A 是破坏性实验（真造 dangling delegation）→ 标注仅在严格隔离、完全掌控的测试域执行并立即修复。

### Topic 09 — domain-registration
- 核心正确：注册与 DNS 托管生命周期独立；transfer lock/auth code/恢复受 TLD 规则约束。
- "gTLD 多 Amazon Registrar、ccTLD 多 Gandi"过度概括 → 按 GetDomainDetail/TLD 官方表确认实际 registrar。
- "转移必需解锁+一次性 auth code+60 天"过绝对 → 部分 TLD 不支持锁/不要求 auth code；60 天按 TLD/注册商规则确认。
- "90 天后域名永久删除、RDAP 正逐步取代 WHOIS"错误/过时 → 90 天是 AWS 账号重开期限，域名处置依 registrar；RDAP 自 2025-01-28 已是 gTLD 权威来源。

### Topic 10 — Profiles
- 核心正确：每 VPC 最多一个 Profile，可 RAM 跨账号共享。
- "只打包四类资源"不完整 → 还支持 Interface VPC endpoints，及 DNSSEC validation/反向 DNS/DNS Firewall failure mode 等设置。
- "local 永远优先于 Profile"易误导 → 先按最具体域名匹配；仅相同域名冲突时 local 优先。
- 补边界：Profile/资源/VPC 必须同 Region；Organizations 共享可自动接受，RAM 权限不一定只读。

### Topic 11 — ARC
- 核心正确：routing control 是人工/API 开关；cluster 五个区域数据面端点，≥3/5 quorum。
- "每 routing control 自动对应 health check"错误 → 必须显式创建 RECOVERY_CONTROL health check 并关联 RoutingControlArn。
- "客户端需访问三个端点形成 quorum"易误导 → 客户端只调用一个数据面端点；3/5 是 ARC 后端复制一致性；控制平面在 us-west-2。
- "确定性切换/安全规则绝对阻止"过强 → 状态切换确定，但实际流量受 DNS TTL/缓存影响；紧急可 SafetyRulesToOverride 覆盖。

### Topic 12 — 集成与配额
- 核心正确：默认 500 hosted zones/账号、10000 records/zone 可提；合格 AWS Alias 查询免费。
- **"Route 53 API 统一 5 RPS"已过时** → 公共 Route 53 API 账号桶=持续 10 RPS/突发 50；DNS changes=持续 100/s/突发 1500；5 RPS 主要是 Resolver API。
- Alias 目标列表不完整 + ETH 实验错误 → 还含 Elastic Beanstalk/App Runner/AppSync/OpenSearch/Lightsail/VPC Lattice；单条 simple Alias 即使目标不健康仍可能 fail-open，需多记录组验证 ETH。
- "GA 永远两个 IP/Resolver 约 10K QPS/ENI"不精确 → 按官方现行数字核对。

### Topic 13 — 排查纪律
- 方法论 topic，核对其中引用的具体数字/响应码语义与官方一致即可（尤其 VPC .2/+2、SERVFAIL/REFUSED/NXDOMAIN 含义）。


================================================================================

# FILE: exam/exam-bank-01-07.md
<!-- SOURCE FILE: exam/exam-bank-01-07.md -->

# Route 53 SME 模拟题库 — Topics 01–07（上半）

> 覆盖：01 Hosted Zones / 02 记录与 Alias / 03 路由策略 / 04 健康检查 / 05 Resolver 与混合 DNS / 06 DNS Firewall / 07 DNSSEC。
> 题型：场景单选 / 多选（真实 SME 风格，给客户场景问最佳方案）为主，少量问答题。每题含【答案】【解析】。
> 总题数：52 题（Ch01 8 · Ch02 7 · Ch03 8 · Ch04 8 · Ch05 8 · Ch06 7 · Ch07 6）。

---

## 第 1 章 · Hosted Zones（8 题）

### 1-1（单选）
客户在同一账号下同时持有 public zone `habitat-energy.au`（父）与 `prod.habitat-energy.au`（子），父 zone 里有一条指向子 zone 的 NS 委派记录。客户想把子域记录合并进父 zone、随后删除子 zone，要求切换零停机。最安全的操作顺序是？

- A. 先删父 zone 里的 NS 委派记录，再在父 zone 建全子域记录
- B. 先在父 zone 建全所有子域记录并验证，再删父 zone 里的 NS 委派记录，最后删子 zone
- C. 直接删除子 zone，Route 53 会自动把记录并入父 zone
- D. 同时删 NS 委派记录和子 zone，再补建父 zone 记录
- E. 把子 zone 的 NS 记录复制到父 zone 即可

【答案】B

【解析】只要父 zone 里的 NS 委派记录还在，Route 53 就始终把 `*.prod.habitat-energy.au` 的查询引向子 zone，父 zone 内同名记录在删委派前不会被查询——这正好允许"先建全记录再切换"的无缝迁移。A/D 顺序颠倒会在记录未建全前造成解析失败；C 错在合并不是自动的；E 把委派 NS 复制到父 zone 并不改变委派优先行为。（Topic 01 §1.6 / case 178118102100384）

### 1-2（多选，选两项）
关于 Private Hosted Zone（PHZ）的启用与解析边界，正确的有哪些？

- A. 使用 PHZ 要求关联 VPC 的 `enableDnsHostnames` 与 `enableDnsSupport` 都为 true
- B. PHZ 记录可被公共递归 DNS（如 8.8.8.8）解析
- C. VPC 若用 DHCP Option Set 指向自定义 DNS，PHZ 记录会返回 NXDOMAIN
- D. PHZ 支持 DNSSEC signing
- E. PHZ 支持 IP-based 路由策略

【答案】A、C

【解析】PHZ 只能经 VPC Resolver（VPC+2/.2）解析，公共 DNS 解析不到（B 错，正是 case 178239640400861 的 resolv.conf fallback 根因）；DHCP Option Set 指向自定义 DNS 会绕过 VPC Resolver 导致 NXDOMAIN（C 对）。PHZ 不支持 DNSSEC（仅 public zone，D 错），也不支持 IP-based（E 错）。启用前提是两个 DNS 属性都为 true（A 对）。（Topic 01 §1.2 / §1.4）

### 1-3（单选）
客户裸域 `example.com`（zone apex）要指向一个 ALB。应如何配置？

- A. 建一条 apex 的 CNAME 指向 ALB DNS 名
- B. 建一条 apex 的 A (Alias) 指向 ALB
- C. 建一条 apex 的 NS 记录指向 ALB
- D. 建一条 apex 的 TXT 记录写入 ALB 名
- E. apex 无法指向 ALB，必须换用 www 子域

【答案】B

【解析】zone apex 已强制带 NS+SOA 记录，而 DNS RFC 规定 CNAME 不能与同名其他类型共存，所以 apex 不能建 CNAME（A/排除）。指向 ALB 这类 AWS 资源用 A-Alias 即可（B 对），Alias 免费、少一跳、可开 Evaluate Target Health。（Topic 01 §1.3 / Topic 02）

### 1-4（单选）
一个 VPC 内查询 `seattle.accounting.example.com`，该 VPC 同时关联了 `accounting.example.com` 和 `example.com` 两个 PHZ。Route 53 会用哪个 PHZ 应答？

- A. `example.com`（更早创建的）
- B. `accounting.example.com`（最具体匹配）
- C. 随机选一个
- D. 两个都查，合并结果
- E. 返回 NXDOMAIN，因为命名空间重叠冲突

【答案】B

【解析】重叠命名空间取最长/最具体匹配（most-specific-match）：`accounting.example.com` 是请求域名更具体的父级，胜过 `example.com`。不是按创建时间或随机（A/C 错）。（Topic 01 §1.5）

### 1-5（单选）
客户跨账号把一个 PHZ 关联到另一账号的 VPC，希望免去逐个授权的繁琐。以下哪种做法可省去 `VpcAssociationAuthorization` 步骤？

- A. 用 VPC Peering 替代
- B. 使用 Route 53 Profiles
- C. 打开 `enableDnsHostnames`
- D. 把 PHZ 改成 public zone
- E. 用 RAM 共享该 PHZ

【答案】B

【解析】跨账号关联 PHZ 到别账号 VPC 通常需先做 `VpcAssociationAuthorization`；改用 Route 53 Profiles 可免此步，且适合 >300 VPC-PHZ 关联的大规模治理。（Topic 01 §1.2 / E27）

### 1-6（单选）
客户删除一个已启用 DNSSEC signing 的 hosted zone 时报 `HostedZoneNotEmpty`。最可能的原因与正确处理是？

- A. zone 内还有普通记录，删掉即可
- B. 需先 deactivate/delete 该 zone 的 KSK
- C. 该 zone 被 RAM 共享，需先取消共享
- D. NS/SOA 记录不能删，导致 zone 非空
- E. 需先关闭 VPC 关联

【答案】B

【解析】带 KSK 的 zone 删除会报 `HostedZoneNotEmpty`，需先停用/删除 KSK 再删 zone。NS/SOA 是每个 zone 自带的、不计入"非空"判断（D 错）。（Topic 01 §4 QA-325）

### 1-7（多选，选两项）
关于 hosted zone 的计费与迁移行为，正确的有哪些？

- A. 每个 hosted zone 前 25 个按 $0.50/月 计费
- B. 同一 zone 在 12 小时内删除会重复收费
- C. 迁移记录到新 zone 时，apex 的 NS 与 SOA 需要一并复制过去
- D. 新建 zone 的 apex NS+SOA 由 Route 53 自动生成
- E. 每个 PHZ 默认可关联无限个 VPC

【答案】A、D

【解析】hosted zone $0.50/月（前 25 个，A 对）；同 zone 12 小时内删除不重复收费（B 错）；迁移不复制 NS 与 SOA，新 zone apex 的 NS+SOA 由 Route 53 自动生成（C 错、D 对）；单 PHZ 默认关联上限 300 VPC（E 错）。（Topic 01 §1.6 / §4）

### 1-8（问答）
客户用 Kubernetes ExternalDNS 管理 `prod.example.com` 子 zone，现在想把子域合并进父 zone `example.com` 并删除旧子 zone。除了 DNS 记录迁移本身，还有哪两个真实的坑必须处理？

【答案】(1) ACM 证书验证用的 CNAME 记录必须精确复制到父 zone，否则证书续期失败；(2) 删旧 zone 前必须把 ExternalDNS 的 `--zone-id-filter`/`--domain-filter` 改指新 zone ID，否则它会继续往已删除的旧 zone 写记录。

【解析】这两点是 case 178118102100384 reply3 明确的合并真实坑：ExternalDNS 的 TXT 记录含 `heritage=external-dns`，若不改 filter，它会对着已删除的旧 zone 反复写入，造成记录漂移/失败。（Topic 01 §2 核心案例补充）

---

## 第 2 章 · 记录类型与 Alias（7 题）

### 2-1（单选）
客户要把子域 `www.example.com` 指向一个**第三方（非 AWS 托管）**的域名 `cdn.partner.net`。应使用哪种记录？

- A. A-Alias
- B. AAAA-Alias
- C. CNAME
- D. A 记录直接填 partner 域名
- E. Alias 指向同 zone 记录

【答案】C

【解析】Alias 的目标只能是选定 AWS 资源或同 zone 同类型记录，指不了外部/非 R53 托管域名；要指向任意外部 DNS 名只能用 CNAME。（Topic 02 §1.3 A4）

### 2-2（多选，选三项）
关于 Alias 相对 CNAME 的特性，正确的有哪些？

- A. Alias 可以建在 zone apex，CNAME 不行
- B. Alias 指向 AWS 资源的查询免费
- C. Alias 指向 AWS 资源时可以单独设置 TTL
- D. Alias 支持 Evaluate Target Health，CNAME 不支持
- E. Alias 的目标可以再是另一条 Alias 或 CNAME

【答案】A、B、D

【解析】Alias 能建在 apex（A 对）、指向 AWS 资源查询免费（B 对）、有 ETH（D 对）。Alias 指向 AWS 资源不能设 TTL，用资源默认 TTL（C 错）；Alias 的 target 不能再是 Alias/CNAME（E 错）。（Topic 02 §1.3 / A1–A5）

### 2-3（单选）
客户做 failover，主备记录都指向 ELB（可建 Alias 的资源）。以下哪种做法最规范？

- A. 给每条 alias 记录另外建一个独立 endpoint 健康检查
- B. alias 记录设 Evaluate Target Health = Yes，不再单独建健康检查
- C. 把 alias 改成 CNAME 再挂健康检查
- D. 用 simple 路由 + 健康检查
- E. 关闭健康检查，靠 TTL 过期切换

【答案】B

【解析】指向可建 alias 的 AWS 资源做 failover 时，用 ETH=Yes 让 Alias 自动感知目标健康，不要再单独建 HC（否则多余且可能冲突）。CNAME 无 ETH（C 错），simple 不能关联 HC（D 错）。（Topic 02 §1.3 A5 / Topic 04 C14）

### 2-4（单选）
客户用 simple 路由在一条记录里配了 12 个 A 值，观察到每次 dig 只返回一部分且顺序变化。关于这一行为，正确的解释是？

- A. 配置错误，simple 只能配 1 个值
- B. Simple 的一条 RRset 会返回**全部**值并随机排序；每次 dig 只看到一部分是 UDP 应答体积/客户端截断所致，不是 Route 53 的"8 值上限"
- C. 健康检查剔除了不健康的值
- D. TTL 过期导致部分值丢失
- E. Route 53 在做真正的负载均衡

【答案】B

【解析】Simple 路由的一条 RRset 内可含多个值（单 RRset 配额 400 条），权威 NS **返回该 RRset 的全部值并随机排序**——这是粗粒度分散，不是负载均衡（E 错），也不涉及健康检查（simple 不能关联 HC）。**"每次最多返回 8 个值"是 Multivalue Answer 路由的特性，不是 Simple**；simple 下看到"只回一部分"通常是 UDP 应答大小限制/解析器或客户端截断的表现，而非 Route 53 主动截到 8 条。（Topic 02 §1.2 / Topic 03 §1.1）

### 2-5（单选）
关于 TTL，以下哪项正确？

- A. Route 53 记录有 300 秒的默认 TTL
- B. Alias 指向 AWS 资源时可自设 TTL
- C. 记录 TTL 无默认值，需自行设定；SOA 缩短 TTL 无副作用
- D. TTL 只能设整分钟
- E. CNAME 不能设 TTL

【答案】C

【解析】Route 53 不为记录设默认 TTL，需自设（建议 60–172800 秒）；SOA 也可缩短 TTL 无副作用。Alias 指向 AWS 资源不能设 TTL（B 错），CNAME 可设 TTL（E 错）。（Topic 02 §1.2 / §4）

### 2-6（单选）
客户把 apex 子域配成 A (Alias) 指向 Kong NLB，配置本身 100% 正确，但 VPC 内 dig 却返回旧的 Apigee ELB IP。最可能的根因层次是？

- A. Alias 记录类型选错了
- B. apex 不该用 Alias
- C. VPC 内解析被 Resolver Forward Rule 拦截，查询没到达公网 zone
- D. NLB 未开启健康检查
- E. ELB canonical hosted zone id 填错

【答案】C

【解析】记录/Alias 配置层完全正确，"解析结果不对"的真实根因在解析优先级层：VPC 内的 Forward Rule 优先于 PHZ 优先于公网，查询被转发到企业 DNS（返回旧记录），根本没到达公网 zone。这正是 case 178602594200640——A-Alias→NLB 是本 topic 正面范例，失效根因归 Resolver（Topic 05）。（Topic 02 §2）

### 2-7（问答）
简述"Alias 指向 AWS 资源"相对"CNAME"在**性能**与**计费**上的两点优势。

【答案】(1) 性能：Alias 由 Route 53 在权威侧直接返回目标 IP，少一次 DNS 往返；CNAME 需 resolver 额外一次 lookup 去解析别名目标。(2) 计费：Alias 指向 AWS 资源的查询免费；CNAME 按查询正常计费。

【解析】能用 Alias 指 AWS 资源就别用 CNAME——成本 + 性能双优。（Topic 02 §1.3 A3 / §4）

---

## 第 3 章 · 路由策略（8 题）

### 3-1（单选）
客户在多个 AWS Region 部署了相同应用，希望终端用户被路由到"网络延迟最低"的 Region。应选哪种路由策略？

- A. Weighted
- B. Latency
- C. Geolocation
- D. Geoproximity
- E. Simple

【答案】B

【解析】Latency 路由按 AWS 长期实测的网络延迟把用户导向延迟最低 Region 的资源，目标须在 AWS Region。Weighted 是按你设的比例分（灰度/负载），不是延迟（A 错）；geolocation/geoproximity 按地理位置而非网络延迟。注意 latency 数据非实时、会漂移。（Topic 03 §1.1 / B6）

### 3-2（单选）
客户用 geolocation 路由，为 CN、US 各配了记录，但没有配 default。一个来自无法映射到任何地理位置的 IP 的查询会得到什么？

- A. 随机返回 CN 或 US 记录
- B. 返回离得最近的记录
- C. "no answer"
- D. NXDOMAIN
- E. 返回全部记录

【答案】C

【解析】geolocation 对未映射/未覆盖的来源，若没有 default 记录（`CountryCode:"*"`）就返回 "no answer"，不会兜底挑一个。必须显式建 default。（Topic 03 §1.1 / B8 / case 178246722300140）

### 3-3（多选，选两项）
关于 geolocation 与 geoproximity 的区别，正确的有哪些？

- A. geolocation 只看用户位置；geoproximity 看资源+用户位置
- B. geoproximity 可用 bias（±1..±99）扩张/收缩某资源的吸引范围
- C. geolocation 也能用 bias 调整吸引范围
- D. geoproximity 不需要 Traffic Flow
- E. geolocation 需要 Traffic Flow

【答案】A、B

【解析】geolocation 仅按用户位置（大洲/国家/州）、重叠取最小区、可设 default；geoproximity 看资源+用户位置、用 bias 移动边界、通常需 Traffic Flow、同名同类型上限仅 30。bias 是 geoproximity 独有（C 错）。（Topic 03 §1.1 / B7）

### 3-4（单选）
一条 failover 记录，Primary 和 Secondary 的健康检查同时都是 unhealthy。Route 53 会返回什么？

- A. 返回 Secondary
- B. 返回 Primary（fail open）
- C. 返回 "no answer"
- D. NXDOMAIN
- E. 随机返回一个

【答案】B

【解析】failover 的 fail-open 行为：Primary+Secondary 都不健康时返回 Primary，此行为不可配置。（Topic 03 §1.1 / Topic 04 §1 / QA-337）

### 3-5（单选）
客户在一组 weighted 记录里把某条记录的 weight 设为 0，其余记录权重非 0。什么情况下这条 0 权重记录会被返回？

- A. 永远不会被返回
- B. 每次都会被返回一小部分
- C. 仅当组内所有非 0 权重记录都不健康时
- D. 当它自身的健康检查通过时
- E. 当组内权重之和为 0 时

【答案】C

【解析】单条设 weight 0 = 停止该记录流量；但仅当组内所有非 0 权重记录都不健康时才会启用 0 权重记录（组内全 0 时才对 0 权重记录平均分）。所以"永远不返回"是错的（A）。（Topic 03 §1.1 / B12）

### 3-6（单选）
客户在一个 Private Hosted Zone 里想按客户端源 IP 段做 IP-based 路由。会发生什么？

- A. 正常生效
- B. PHZ 不支持 IP-based 路由策略
- C. 需要先开 Traffic Flow
- D. 需要 outbound endpoint
- E. 仅 IPv6 支持

【答案】B

【解析】PHZ 只支持 simple/failover/multivalue/weighted/latency/geolocation/geoproximity；IP-based 只能在 public zone。（Topic 03 §1.1 / B11 / Topic 01 §1.4）

### 3-7（多选，选两项）
关于 EDNS0 / edns-client-subnet (ECS) 对地理/延迟类路由准确性的影响，正确的有哪些？

- A. resolver 支持 ECS 时，Route 53 能用用户 IP 的截断前缀更准确定位
- B. VPC 的 .2 resolver 支持 EDNS0 但不支持 ECS
- C. PHZ 使用 EDNS0 来做路由决策
- D. 不支持 ECS 时，用 resolver 自身源 IP 近似定位，可能偏差
- E. ECS 会加密 DNS 查询内容

【答案】A、D（B 也正确，但本题按"两项最直接描述定位机制"取 A、D；若允许三项则 A、B、D）

【解析】ECS 让权威 NS 拿到用户 IP 截断前缀，定位更准（A）；不支持时退回 resolver 源 IP 近似（D）。VPC .2 resolver 支持 EDNS0 但不支持 ECS（B 也是正确知识点）；PHZ 不使用 EDNS0，改用所在 Region 的 VPC Resolver 数据（C 错）；ECS 与加密无关（E 错）。（Topic 03 §1.2）

> 说明：本题若判分按三选，正确项为 A、B、D。

### 3-8（问答）
客户咨询 Route 53 geolocation 用的是什么 IP 地理库、多久更新一次。作为 SME，你应如何答复？

【答案】答复要点：使用业界领先的第三方商业 IP 地理定位数据库，因商业协议限制不便披露具体供应商名；该类库会定期更新，AWS 在其发布后自动摄取并持续跟进，但 AWS 对具体传播/更新周期不作公开承诺。同时提醒客户务必配置 Default 记录兜底无法映射的来源（否则返回 "no answer"），国家级映射准确率约 99.8%（粒度越细准确率越低）。

【解析】内部指引是 MaxMind（约每周更新），但"不对客户披露供应商名"、"不给具体周期"。case 178246722300140 的三大 SME 落点：GeoIP 库不披露、必配 default 否则 no answer、EDNS0/ECS 决定精度。（Topic 03 §2 / B8）

---

## 第 4 章 · 健康检查（8 题）

### 4-1（单选）
客户要监控一个**内部 ALB**（私有、不可路由 IP）的健康。哪种健康检查类型最合适？

- A. Endpoint HTTP 检查，直接填内部 ALB 私有 IP
- B. Endpoint TCP 检查内部 IP
- C. CloudWatch alarm-based（或 Calculated）健康检查
- D. Simple 路由自带的健康检查
- E. 无法监控内部资源

【答案】C

【解析】不能对 private/不可路由/multicast IP 建 endpoint 健康检查（A/B 排除）；内部 ALB 应改用 CloudWatch alarm-based 或 calculated HC（EC2 场景可配 EIP 固定公网 IP）。simple 路由不能关联 HC（D 错）。（Topic 04 §1 / C19）

### 4-2（单选）
客户的 HTTPS endpoint 健康检查突然变红，同时发现站点证书刚过期。证书过期是否是健康检查失败的原因？

- A. 是，HTTPS 健康检查会校验证书
- B. 否，HTTPS 健康检查不校验证书，证书过期不会导致失败
- C. 是，但只在开启 string matching 时
- D. 否，但会触发 INSUFFICIENT_DATA
- E. 是，需要更新根 CA

【答案】B

【解析】HTTPS 健康检查不校验证书，证书过期/无效不会导致检查失败——这是经典陷阱题。健康检查变红的原因要另找（连接超时、状态码非 2xx/3xx、string 未匹配等）。（Topic 04 §1 / C17）

### 4-3（单选）
关于 Route 53 健康检查的聚合判定（多地 checker），正确的是？

- A. 需超过 50% 的 checker 报健康才判健康
- B. 需超过 18% 的 checker 报健康才判健康
- C. 全部 checker 都健康才判健康
- D. 任一 checker 健康即判健康
- E. 由客户配置阈值百分比

【答案】B

【解析】聚合规则：> 18% 的 checker 报健康即判健康，≤ 18% 判不健康。18% 门槛用于防止某处网络隔离造成的误判，不是多数决（A 错）。（Topic 04 §1 / C15）

### 4-4（多选，选两项）
关于健康检查的响应时间阈值，正确的有哪些？

- A. HTTP/HTTPS：4s 内建 TCP 连接 + 连接后 2s 内回 2xx/3xx
- B. TCP：10s 内建连
- C. string matching 的匹配串必须落在 body 的前 5,120 字节内
- D. HTTP 需 10s 内回 2xx
- E. TCP 需 2s 内建连

【答案】A、B（C 也是正确知识点）

【解析】HTTP/HTTPS 是 4s 建连 + 2s 回 2xx/3xx（A 对）；TCP 是 10s 建连（B 对）；string match 匹配串须在 body 前 5120 字节内（C 也对）。D/E 把阈值记混。（Topic 04 §1 / C16）

> 说明：若判分允许三项，正确项为 A、B、C。

### 4-5（单选）
一个 CloudWatch alarm-based 健康检查，其 alarm 进入 `INSUFFICIENT_DATA` 状态。健康检查会如何判定？

- A. 一律判健康
- B. 按 HC 的 `InsufficientDataHealthState` 配置（默认 Unhealthy）
- C. 一律判不健康，不可改
- D. 保持上次状态，不可改
- E. 触发 fail open

【答案】B

【解析】CloudWatch HC 监控 alarm 的数据流；INSUFFICIENT_DATA 走 `InsufficientDataHealthState`，三态为 Unhealthy（默认）/ Healthy / LastKnownStatus。case P460958645 的缓解就是改成 LastKnownStatus。（Topic 04 §1 / §2 / C18）

### 4-6（单选）
一个 CloudWatch alarm-based HC 关联的 alarm 已恢复 OK，但 HC 长时间（数十分钟）仍停留 Unhealthy。作为紧急恢复手段，哪种操作能触发 HC 重新评估？

- A. 删除并重建 hosted zone
- B. 对该 HC 执行 `UpdateHealthCheck`（如改一个无害字段）
- C. 重启关联的 EC2 实例
- D. 删除 CloudWatch alarm
- E. 只能等待，无法干预

【答案】B

【解析】`UpdateHealthCheck` 会触发 HC 重新评估，可作紧急恢复手段（case P460958645 中客户正是手动 UpdateHealthCheck 后恢复）。正常 R53 对 alarm 状态变化应在 1–2 分钟内响应，数十分钟属异常。（Topic 04 §1 / §2）

### 4-7（单选）
受害账户在 ALB access log 里发现持续的探测，User-Agent 为 `Amazon-Route53-Health-Check-Service (ref 28e27f8c-...)`，但自己并未创建该健康检查。这属于什么问题、如何处理？

- A. 正常流量，无需处理
- B. Unwanted HC abuse——由 Support Ops 走标准流程（确认 HC → 联系 offending 账户 → 等 7 天 → Mechanic 禁用，需 2PR）
- C. DDoS 攻击，走 Shield 流程
- D. 直接在自己账户里删除该 HC
- E. 修改 SG 屏蔽即可，无需上报

【答案】B

【解析】任意账户都能建指向不属于自己 IP 的 endpoint HC 造成滋扰。User-Agent 与 `ref=<HC ID>` 是识别关键。处理是 Support Ops 专属：确认 HC → 联系 offending → 等 7 天无回复 → 用 Mechanic `controlapi disable-health-check` 禁用（需 2PR），禁用非删除（`Disabled=true`）。受害方无法删别人账户里的 HC（D 错）。（Topic 04 §1 / §2 / case V2254930641）

### 4-8（问答）
简述 failover 的两种"fail-open"表现。

【答案】(1) 单条 failover 记录视角：Primary + Secondary 的健康检查都 unhealthy 时，Route 53 返回 Primary（fail open 到 primary），不可配置。(2) 整个 Hosted Zone 视角：该 zone 所有健康检查都 unhealthy 时，Route 53 fail open（当作全部通过来应答）。

【解析】fail-open 的设计意图是"宁可返回一个可能不健康的答案，也不返回空"，避免因健康检查系统性故障导致整站不可解析。（Topic 04 §1）

---

## 第 5 章 · Resolver 与混合 DNS（8 题）

### 5-1（单选）
VPC 的主 CIDR 是 `10.201.244.0/22`。VPC Resolver（.2 resolver）的地址是？

- A. 10.201.244.1
- B. 10.201.244.2
- C. 10.201.247.253
- D. 169.254.169.253
- E. 10.201.244.253

【答案】B

【解析】VPC Resolver 接入地址 = VPC 主 CIDR 基地址 +2，即 10.201.244.2（经典 .2 / VPC+2）。注意 169.254.169.253 是 link-local 别名，但按题目"VPC+2"算法答案是 10.201.244.2。（Topic 05 §1.1）

### 5-2（单选）
EC2 无法用 `nslookup ... 10.x.x.2`（VPC DNS）解析。工程师第一反应去检查 EC2 到 VPC DNS 之间的 Security Group / NACL / 路由表。这个排查方向对吗？

- A. 对，SG/NACL 常常拦住到 .2 的流量
- B. 不对，EC2 到 VPC DNS(.2) 的流量不经过 SG/NACL/路由表
- C. 对，但只需检查 NACL
- D. 不对，应改用公共 DNS
- E. 对，需要放行 UDP 53

【答案】B

【解析】EC2 → VPC DNS(.2) 的流量不经过 SG/NACL/路由表，这是排障最大的方向性坑（case 178767531400698 中 Genie 初稿即犯此错）。真正的阻断点在 Outbound Endpoint 的 ENI 子网 NACL。（Topic 05 §1.1 / D20）

### 5-3（单选）
某域名在 VPC 内既有一个 PHZ 记录、又有一条 Resolver Forward Rule 指向企业 DNS。VPC 内查询该域名时会怎样？

- A. 用 PHZ 记录应答
- B. 查询被 Forward Rule 转发到企业 DNS（Forward Rule 优先于 PHZ）
- C. 报冲突错误
- D. 随机选一个
- E. 先查 PHZ，未命中再转发

【答案】B

【解析】VPC 内解析优先级链：Forward Rule > PHZ > System Rule > 公网递归。同域名冲突时 Forward Rule 胜，查询被转发出去而不用 PHZ 记录。（Topic 05 §1.4 / D22 / case 178602594200640）

### 5-4（单选）
关于 Inbound / Outbound Resolver Endpoint 的方向，正确的是？

- A. Inbound = AWS→on-prem 查询；Outbound = on-prem→AWS 查询
- B. Inbound = on-prem→AWS 查询；Outbound = AWS→on-prem 查询
- C. 两者方向相同，只是冗余
- D. Inbound 用于公网，Outbound 用于私网
- E. Outbound endpoint 使用公网 IP

【答案】B

【解析】Inbound 让本地/其他网络进来查 AWS 内部名/PHZ；Outbound 让 VPC 的查询出去转发到本地 DNS。记忆：In=别人进来查我，Out=我出去查别人。Outbound endpoint 是私有 IP，出公网需 NAT Gateway（E 错）。（Topic 05 §1.2 / D21）

### 5-5（单选）
客户想把 VPC 内**所有**域名查询都转发到企业 DNS，让 VPC 内查询永远不到达公网 Route 53。应如何配置 Resolver Rule？

- A. 为每个域名各建一条 FORWARD 规则
- B. 建一条域名为 `.`（dot）的 FORWARD 规则，关联 outbound endpoint
- C. 删除所有 System Rule
- D. 建一条 SYSTEM 规则指向企业 DNS
- E. 关闭 VPC 的 enableDnsSupport

【答案】B

【解析】域名为 `.` 的 FORWARD 规则覆盖除 PHZ/AWS 内部名以外的所有域名，把 VPC 内查询全部转出（case 178602594200640 里的 `RULE-INFOBLX`）。Rule 必须关联 outbound endpoint 才生效。（Topic 05 §1.3 / §1.4）

### 5-6（多选，选两项）
关于 Resolver 的 QPS / 吞吐边界，正确的有哪些？

- A. 每个 Outbound Endpoint 最多 6 个 ENI，聚合可达约 60,000 请求/秒
- B. 每个 ENI 约 10K QPS，但经 NLB 或 SG 因 connection tracking 会降到约 1.5–1.7K QPS
- C. 每 ENI 固定 1024 QPS 不可调
- D. Resolver 是全球服务，非 Regional
- E. Rule 不关联 outbound endpoint 也能生效

【答案】A、B

【解析】每 Outbound Endpoint ≤6 ENI、聚合 ~60K req/s（A）；每 ENI ~10K QPS，经 NLB/SG 因强制 connection tracking 降到 ~1.5–1.7K（约 6 倍，B）。1024 是实例侧 VPC+2 link-local 的 PPS 限制（不是 endpoint QPS，C 混淆）；Resolver 是 Regional（D 错）；Rule 必须关联 outbound endpoint（E 错）。（Topic 05 §1.5）

### 5-7（单选）
Outbound Endpoint 有两个 ENI，分属不同子网/NACL。工程师排查发现其中一个 ENI 子网的 NACL 阻断了 DNS。为让转发可靠，NACL 上应双向放行哪些？

- A. 只放行入站 TCP 53
- B. 出站 UDP/TCP 53 + 入站 UDP/TCP ephemeral(1024-65535)，且每个 ENI 子网都要一致
- C. 只放行出站 UDP 53
- D. 放行所有 ICMP
- E. 只在一个 ENI 子网放行即可

【答案】B

【解析】Outbound Endpoint 每个 ENI 子网都要在 NACL 双向放行 DNS：出站 UDP/TCP 53 + 入站 UDP/TCP ephemeral。多 ENI 跨不同子网/NACL 时配置必须一致，否则部分查询超时、表现为间歇故障；且单条 ENI 通不代表全通。（Topic 05 §1.4 / §2 case 二 / D 速记）

### 5-8（问答）
客户跨账号用 AWS RAM 共享一条 Resolver Forward Rule。关于共享范围、outbound endpoint 与成员账号权限，有哪三点关键结论？

【答案】(1) 共享限**同一 Region**，无需额外 VPC peering；(2) 父账号的 outbound endpoint 会随 rule 一起共享，成员账号不需另建 endpoint；(3) 成员账号只能使用共享 rule，不能修改或删除它。

【解析】这是考点 D24。常见错误是以为跨账号共享 rule 还要单独建 outbound endpoint。（Topic 05 §1.3 / D24）

---

## 第 6 章 · DNS Firewall / Global Resolver（7 题）

### 6-1（单选）
DNS Firewall 的核心用途和过滤维度是？

- A. 加密所有 DNS 流量
- B. 对出站 DNS 查询做域名层过滤，主要防 DNS 数据渗漏（exfiltration）
- C. 按 IP/端口做网络层过滤
- D. 校验 DNSSEC 签名
- E. 对入站 DNS 查询做应用层过滤

【答案】B

【解析】DNS Firewall 对经 VPC Resolver 的出站 DNS 查询做域名字符串层过滤，在解析成 IP 之前拦截命中恶意/黑名单域名的查询，主要防 exfiltration。它不加密、不看 IP/端口/应用层协议。（Topic 06 §1 / E25）

### 6-2（单选）
一个 VPC 关联了多个 rule group，rule group 内又有多条 rule。它们的处理顺序是？

- A. priority 数字越大越先处理
- B. priority 数字越小越先处理（rule group 与 rule 都是）
- C. 按创建时间倒序
- D. 随机
- E. 只处理 priority 最大的一条

【答案】B

【解析】rule group 之间按关联 priority 数字从小到大处理，rule group 内部每条 rule 也按组内 priority 从小到大处理（lowest numeric priority first）。（Topic 06 §1 / E25）

### 6-3（多选，选两项）
关于 DNS Firewall 规则的动作（Action），正确的有哪些？

- A. 含 domain list 的规则可选 ALLOW / BLOCK / ALERT
- B. 不含 domain list 的规则（Advanced 保护）只能 BLOCK / ALERT
- C. 任何规则都能设 ALLOW
- D. BLOCK 只能返回 NXDOMAIN
- E. Advanced 保护可以设 ALLOW

【答案】A、B

【解析】含 domain list 才能 ALLOW（A）；Advanced（DGA/tunneling 等）只能 BLOCK/ALERT，不能 ALLOW（B 对、C/E 错）。BLOCK 响应有三种：NXDOMAIN / NODATA / OVERRIDE(CNAME 重定向)（D 错）。（Topic 06 §1 / E25）

### 6-4（单选）
客户用 allowlist（只放行白名单）方式，把入口域名 `svc.example.com` 加入了 ALLOW 列表，但它的 CNAME target 指向未列入白名单的 `backend.other.net`，结果查询被 BLOCK。根因是？

- A. ALLOW 规则失效
- B. 重定向链上的后续域名未显式加入 domain list（默认检查整条 CNAME 链，trust 仅在单次查询事务内有效）
- C. domain list 不支持 CNAME
- D. 白名单必须用通配符
- E. AWS 托管列表拦截了它

【答案】B

【解析】默认会检查整条重定向链，被 ALLOW 域名的 CNAME target 若未列入 domain list，其 A+AAAA 相当于未被 ALLOW 命中而被 BLOCK。allowlist 要把整条 CNAME 链域名都覆盖，或正确配置 Trust Redirection Domains。（Topic 06 §1 / §2 内部 QA-497）

### 6-5（单选）
客户配了一条 ALERT 规则和一条 DGA Advanced 规则，想确认某次查询是否命中了 DGA 检测。仅凭 dig 返回 NXDOMAIN 可以判定吗？应看哪里？

- A. 可以，NXDOMAIN 就代表 DGA 命中
- B. 不能，必须查 OCSF 日志的 `firewall_rule_id`；ALERT 命中流量的 `action_name` 仍是 `Allowed`
- C. 可以，看 dig 的 flags 即可
- D. 不能，只能看 CloudTrail
- E. 可以，看响应的 TTL

【答案】B

【解析】ALERT 命中的流量被放行，OCSF 日志里 `action_name` 仍是 `Allowed`；DGA 是否命中不能凭 dig 的 NXDOMAIN 判断，必须查日志的 `firewall_rule_id`（OCSF 动作值是 Allowed/Denied，不是 ALLOW/BLOCK）。这是"用户可见形态 ≠ 中间判据"的典型。（Topic 06 §2 / §4）

### 6-6（多选，选两项）
关于 Global Resolver 与 VPC Resolver 上 DNS Firewall 的区别，正确的有哪些？

- A. Global Resolver 的 Firewall 规则绑定到 DNS View，而非 VPC
- B. Global Resolver 控制面 API 固定在 us-east-2（Ohio）
- C. Global Resolver 只能保护 VPC 内部工作负载
- D. VPC Resolver 的 Firewall 规则绑定 DNS View
- E. Global Resolver 需要 VPN 才能被客户端访问

【答案】A、B

【解析】Global Resolver 的 Firewall 规则绑 DNS View（A），控制面固定 us-east-2（B）；Global Resolver 保护 VPC 外部客户端、VPC Resolver 保护内部（C 错），VPC Resolver 的 Firewall 绑 VPC（D 错）；Global Resolver 提供全球 Anycast IP，无需 VPN（E 错）。（Topic 06 §1）

### 6-7（问答）
简述 DNS Firewall 与 Network Firewall 在"看得到什么流量"上的区别，以及二者关系。

【答案】DNS Firewall 只看经 VPC Resolver 的出站 DNS 查询（域名层）；Network Firewall 过滤网络/应用层流量，但看不到 Resolver 发起的 DNS 查询。二者是互补关系，不是替代——Network Firewall 拦不住经 Resolver 的 DNS exfiltration，需 DNS Firewall 补位。

【解析】考点 E26 的核心：不要以为 Network Firewall 能拦住 DNS 查询，也不要以为二者二选一。（Topic 06 §1 / E26）

---

## 第 7 章 · DNSSEC（6 题）

### 7-1（单选）
客户要禁用某 public zone 的 DNSSEC signing，直接在 Route 53 点禁用时报 `Please remove DS records in the parent zone first`。正确的禁用顺序是？

- A. 直接强制禁用 signing，再删 DS
- B. 先删父区/注册商的 DS 记录、等传播完成，再关闭 signing
- C. 先删子区 DNSKEY，再关 signing
- D. 先删 KSK，再删 DS
- E. 同时删 DS 和 DNSKEY

【答案】B

【解析】禁用必须先解除信任链：先删父区 DS → 等 DS TTL 过期/传播 → 再关 signing，Route 53 才允许。顺序不能颠倒。（Topic 07 §一/§四 考点 1）

### 7-2（单选）
关于 DS 记录与 DNSKEY 的放置位置，正确的是？

- A. DS 在子区，DNSKEY 在父区
- B. DS 在父区（子区 KSK 对应 DNSKEY 的摘要 + 算法元数据，不含完整公钥），DNSKEY 在子区
- C. 两者都在子区
- D. 两者都在父区
- E. DS 在根区，DNSKEY 在 TLD

【答案】B

【解析】DS（Delegation Signer）放在父区，内容是**子区 KSK 对应 DNSKEY 的摘要（digest）加上算法元数据**（Key Tag / Algorithm / Digest Type / Digest），**不包含完整的 KSK 公钥**，是"父指子"的信任指针；DNSKEY（KSK flag 257 / ZSK flag 256）在子区自身。方向别搞反。（Topic 07 §一 考点 4）

### 7-3（单选）
一个 zone 已经 signing（有 DNSKEY、记录带 RRSIG），但父区 `.com` 里查不到对应 DS 记录。这是什么状态、对解析有何影响？

- A. 信任链完整，解析器会验证
- B. island of trust（信任孤岛）：有签名无信任链，解析器不验证、按普通 DNS 正常返回，不影响可用性
- C. zone 不可解析，返回 SERVFAIL
- D. 配置错误，必须立刻修复否则宕机
- E. DNSSEC 未启用

【答案】B

【解析】zone 自己 signing 但父区无 DS = island of trust，支持 DNSSEC 的解析器不会验证（因为父区没告诉它该 zone 签名了），解析仍正常、不影响可用性。常见于刚启用 signing 还没建 DS，或禁用过程先删了 DS 还没关 signing。（Topic 07 §一/§四 考点 3 / case 178970323600938）

### 7-4（单选）
客户启用 DNSSEC 时报 `<key ARN> could not be used by Route 53 DNSSEC`。KSK 绑定的 KMS CMK 必须满足哪些要求？

- A. 对称密钥、任意规格即可
- B. 位于 us-east-1、asymmetric（非对称）、ECC_NIST_P256，且 key policy 授权 Route 53 DNSSEC 服务
- C. RSA_2048、对称用途
- D. 只要在 us-east-1 即可
- E. 必须是多区域密钥

【答案】B

【解析】KSK 的 CMK 必须**位于 us-east-1**、是 **asymmetric**、规格 **ECC_NIST_P256**（SIGN_VERIFY 用途），且 key policy 授权 `dnssec-route53.amazonaws.com`；任一不满足即报 "could not be used by Route 53 DNSSEC"。D 错在"只要在 us-east-1"——区域正确只是必要条件之一，非对称/规格/授权也都要满足。（Topic 07 §一/§四 考点 2 / case 卡点 2）

### 7-5（单选）
客户 `dig pd-market.com @<recursive>` 得到 SERVFAIL，第一反应怀疑 DNSSEC 验证失败。用什么命令能快速区分"是不是 DNSSEC 验证问题"？

- A. `dig pd-market.com +short`
- B. `dig pd-market.com @<recursive> +cd`——变 NOERROR 则是 DNSSEC 验证失败；仍 SERVFAIL 则与 DNSSEC 无关
- C. `dig DNSKEY pd-market.com`
- D. `dig +trace`
- E. `nslookup pd-market.com`

【答案】B

【解析】`+cd`（Checking Disabled）关闭解析器的 DNSSEC 验证：加了后变 NOERROR，说明原 SERVFAIL 是 DNSSEC 验证失败；仍 SERVFAIL 则与 DNSSEC 无关（case 178970323600938 真正根因是 stale NS 委派）。（Topic 07 §三/§四 考点 5）

### 7-6（问答）
简述轮换 KSK 时为何要走"DS 双记录过渡"，不这样做会有什么风险？

【答案】轮换 KSK 时新旧 DS 需并存一段时间：因为父区 DS、注册商侧变更有传播延迟，且旧 DS 可能仍在解析器缓存中。若直接撤旧 DS/换新，缓存里旧 DS 与 zone 新签名不匹配，中途会出现 DNSSEC 验证失败（SERVFAIL）。等旧 DS 在解析器缓存中过期后再撤旧，可避免验证中断。

【解析】DS/注册商变更受父区 TTL 与注册商处理影响，双记录过渡是平滑轮换的标准做法。（Topic 07 §一 传播与轮换）

---

## 附：答案速查表

| 题号 | 答案 | 题号 | 答案 | 题号 | 答案 |
|---|---|---|---|---|---|
| 1-1 | B | 3-1 | B | 5-1 | B |
| 1-2 | A,C | 3-2 | C | 5-2 | B |
| 1-3 | B | 3-3 | A,B | 5-3 | B |
| 1-4 | B | 3-4 | B | 5-4 | B |
| 1-5 | B | 3-5 | C | 5-5 | B |
| 1-6 | B | 3-6 | B | 5-6 | A,B |
| 1-7 | A,D | 3-7 | A,D(,B) | 5-7 | B |
| 1-8 | 问答 | 3-8 | 问答 | 5-8 | 问答 |
| 2-1 | C | 4-1 | C | 6-1 | B |
| 2-2 | A,B,D | 4-2 | B | 6-2 | B |
| 2-3 | B | 4-3 | B | 6-3 | A,B |
| 2-4 | B | 4-4 | A,B(,C) | 6-4 | B |
| 2-5 | C | 4-5 | B | 6-5 | B |
| 2-6 | C | 4-6 | B | 6-6 | A,B |
| 2-7 | 问答 | 4-7 | B | 6-7 | 问答 |
|  |  | 4-8 | 问答 | 7-1 | B |
|  |  |  |  | 7-2 | B |
|  |  |  |  | 7-3 | B |
|  |  |  |  | 7-4 | B |
|  |  |  |  | 7-5 | B |
|  |  |  |  | 7-6 | 问答 |


================================================================================

# FILE: exam/exam-bank-08-13.md
<!-- SOURCE FILE: exam/exam-bank-08-13.md -->

# Route 53 SME 模拟题库 · Wave4 下半（Topics 08–13）

> 覆盖：08 子域接管/悬空委派、09 域名注册与生命周期、10 Profiles、11 ARC Routing Controls、12 服务集成+配额限流、13 通用排查纪律。
> 题型：场景单选（单选）/ 多选 / 少量问答。每题含题干 + 选项 A–E + 【答案】 + 【解析】。
> 合计 46 题（08:8 · 09:8 · 10:7 · 11:8 · 12:8 · 13:7）。

---

## 第 08 章 · 子域接管 / dangling delegation / StopZoneSniping（8 题）

### 08-1（单选）
某客户发现子域 `sub.example.com` 被他人接管。调查发现：父域仍保留该子域的 4 个 awsdns NS 委派记录，但对应的 child hosted zone 早已被删除。这属于哪个 Scenario，Route 53 的 StopZoneSniping 是否**设计上**会防护？

- A. Scenario 1，会防护
- B. Scenario 5，会防护
- C. Scenario 1，不防护
- D. Scenario 5，不防护
- E. Scenario 3，会防护

【答案】A
【解析】"child zone 曾存在→被删→父域仍留委派" 正是 Scenario 1，是 5 个场景中 Route 53 **唯一**提供防护（对被删 zone 的 NS 组加 hold）的场景。Scenario 2–5 一律不防护。注意题目问的是"设计上是否防护"，与"这次实际是否被拦住"是两回事（后者取决于 hold 当时是否在效）。

### 08-2（单选）
关于 subdomain takeover 的成功条件，下列哪项**最准确**？

- A. 攻击者必须让新建 zone 的全部 4 个 NS 与悬空 NS 完全重叠
- B. 攻击者只需让新建 zone 的 NS 组与悬空 NS 组重叠 ≥1 个即可，重叠越多接管越稳定
- C. 攻击者必须在删除该 zone 的同一账号内重建
- D. 只要父域存在任意 NS 记录即可接管，无需重叠
- E. 必须先拿到 EPP auth code 才能接管

【答案】B
【解析】接管只需命中 ≥1 个重叠 NS（递归解析器在多 NS 间选择/重试，命中攻击者掌控的那个即拿到伪造权威应答）；重叠越多越稳定。"必须整组重叠"是错误认知。接管可在**任意**账号进行（跨账号），与 EPP code 无关（那是域名转移概念）。

### 08-3（多选）
关于 StopZoneSniping 保护特性，以下哪些描述正确？（多选）

- A. 保护是跨账号全局的，阻止任何账号被分配到重叠 NS
- B. 保护仅在删除该 zone 的账号内部生效
- C. 只要新 zone 重叠 1 个或多个 NS 即被阻止（per-NS）
- D. hold 是永久的，删过 zone 的 NS 组永远被锁定
- E. hold 靠"父域仍被观测到委派"续命，委派从父域消失后会被 purge、NS 回池

【答案】A、C、E
【解析】保护是跨账号全局（A 对，B 错）；per-NS 生效，重叠任一即拦（C 对）；hold 非永久（D 错），靠周期性 DNS 重评估续命，父域不再委派则 purge 回池（E 对）。此外保护 backfill 之前就已 dangling 的老委派可能不覆盖。

### 08-4（单选）
退役一个子域时，消除悬空委派的**正确删除顺序**是？

- A. 先删 child hosted zone，再删父域的 NS 委派记录
- B. 先删父域的 NS 委派记录 → 等 TTL 过期 → 再删 child hosted zone
- C. 同时删除父域 NS 记录和 child zone
- D. 只删 child zone，父域委派保留以便日后复用
- E. 先启用 DNSSEC，再任意顺序删除

【答案】B
【解析】文档原则：先删父域 NS 记录并等 TTL 过期（确保 resolver 缓存清空），再删 child zone，"This ensures that no one can hijack the child hosted zone"。反过来（先删 zone 留委派）正是制造 Scenario 1 悬空的错误操作。

### 08-5（单选）
运维做只读巡检时，判断一个子域委派**悬空**的典型 dig 签名是？

- A. `dig sub NS +short` 返回空，且 `dig SOA @NS` 返回 aa
- B. `dig sub NS +short` 仍返回那组 NS，但 `dig SOA @那组NS` 返回 REFUSED/SERVFAIL
- C. `dig sub A @8.8.8.8` 返回 NOERROR + 有 A 记录
- D. `dig sub SOA` 返回 serial=1
- E. `dig sub NS` 返回 NXDOMAIN

【答案】B
【解析】悬空签名 = 父域仍返回 NS 委派（委派还在），但那组 NS 对该域已不再权威（直查 SOA 得 REFUSED/SERVFAIL，NS 上无权威 zone）。选项 D 的 serial=1 是**修复指纹**（新建占位空 zone），不是悬空签名。

### 08-6（单选）
Thales case（`kycshowcase.d1.thalescloud.io`）首轮据"报告时点 SERVFAIL、无权威 zone"推断为 Scenario 5，客户澄清后翻案为 Scenario 1。这个反转给 SME 的**排查纪律**教训是？

- A. dig 的 SERVFAIL 永远意味着 Scenario 5
- B. Scenario 归类必须以父域/删除历史的第一手信息为准，不能只凭报告时点的 dig 签名反推
- C. 只要看到 SERVFAIL 就可对客下 defect 定性
- D. 客户澄清不可信，应以 dig 为准
- E. Scenario 1 和 5 在防护上等价，归类无所谓

【答案】B
【解析】报告时点的 dig 签名只反映"此刻无权威 zone"，无法区分"从未建 zone（Scenario 5）"还是"曾建后删（Scenario 1）"。必须以删除/委派历史的第一手信息定性。归类直接决定"设计上是否本应防护"，绝非无所谓。

### 08-7（多选）
关于 subdomain takeover 的责任归属与处置流程，正确的有？（多选）

- A. 它利用的是父域侧的悬空 DNS 记录，属共享责任模型的客户侧，不是 AWS 服务漏洞
- B. AWS 可以直接读取并删除接管者第三方账号里的 hosted zone
- C. 研究员报告此类漏洞应走 AWS abuse form（report-abuse），而非普通 support case 直接触发对接管者账号的动作
- D. 若服务团队确认属保护缺陷，对外披露前应走 MAPS 并同步 SecOps
- E. NS 向量的修复只能由父域持有方在父域侧完成

【答案】A、C、D、E
【解析】takeover 属客户侧（A 对）；AWS 无法读/删第三方账号 zone（B 错）；研究员漏洞走 abuse form/T&S（C 对）；确认缺陷走 MAPS+SecOps（D 对）；NS 向量修复只能父域持有方做（E 对）。

### 08-8（单选）
关于 DNSSEC 作为对 dangling delegation 风险的防护，下列哪项正确？

- A. DNSSEC 可以替代"正确的删除顺序"，启用后无需再关心删除顺序
- B. DNSSEC signing 让应答需权威源签名，伪造 zone 无法通过验真，是纵深防御；但受 TTL 强制 1 周、PHZ 不支持、需父域 DS 等约束
- C. PHZ（私有区）也完全支持 DNSSEC signing
- D. 启用 DNSSEC 后 TTL 可任意设置
- E. DNSSEC 能阻止攻击者建立重叠 NS 的 zone

【答案】B
【解析】DNSSEC 是密码学纵深防御——伪造 zone 无正确签名/DS 信任链断裂，验证型 resolver 会拒绝；但它不替代正确删除顺序（A 错），且 PHZ 不支持（C 错）、启用后 TTL 强制 1 周（D 错）、需父域支持 DS。它不阻止别人建 zone（E 错），只让伪造应答无法验真。

---

## 第 09 章 · 域名注册 / 生命周期 / TLD 差异（8 题）

### 09-1（单选）
客户 `gateio.jp` 反映：Route 53 控制台显示到期 2027/6/29，而 WHOIS（whois.jprs.jp）显示 2026/06/30，且状态码显示 "-"，怀疑异常。SME 的正确判读是？

- A. WHOIS 到期日权威，客户域名即将过期需立即续期
- B. 两个到期日都对——R53/Gandi 的 2027/6/29 是有效到期日，JPRS WHOIS 按"到期月月末"呈现旧年份且更新滞后；状态 "-" 因 .jp 不支持 transfer lock，均属正常
- C. 状态 "-" 说明域名被锁定，需解锁
- D. 需立即向 JPRS 提工单纠正 WHOIS bug
- E. R53 到期日错误，应以 JPRS 为准修正

【答案】B
【解析】.jp（JPRS）三特性：① WHOIS 到期日恒为到期月月末（6/29→显示 6/30）；② 旧到期月过完才更新新年份（约 7/1 更新）；③ 不支持 transfer lock，无锁状态显示 "-"。有效期以 Gandi/Route 53 记录为准，客户无需操作（保持 auto-renew）。

### 09-2（多选）
关于 Route 53 的"注册商侧"与"DNS 托管侧"的边界，正确的有？（多选）

- A. 删除 hosted zone 会自动注销域名注册
- B. 注销域名注册不会自动删除 hosted zone
- C. 域名被 suspend（clientHold）时，hosted zone 里的记录仍在，但公网查不到，因为注册局停止委派
- D. 改了 hosted zone 的 NS 后，必须在 Registered domains 侧同步更新注册局 NS 委派，否则解析走旧 NS
- E. 注册（registration）与解析（resolution）是同一套生命周期

【答案】B、C、D
【解析】注册 ≠ 解析，两套独立生命周期（E 错）。删 hosted zone 不注销域名（A 错）；注销域名不删 hosted zone（B 对）；suspend/clientHold 断的是注册局委派、记录仍在（C 对）；改 hosted zone NS 必须两侧同步（D 对）。

### 09-3（单选）
关于 .jp 域名的续期与转移特性，下列哪项**错误**？

- A. .jp 续期窗口很窄，仅到期前 30 天到 6 天（D-30 ~ D-6）
- B. .jp 不支持 late renewal
- C. .jp 不支持 transfer lock，状态码显示 "-" 属正常
- D. .jp 由 Amazon Registrar（AMAZON_KS）直接注册
- E. .jp 由 Gandi 代理注册

【答案】D
【解析】.jp 是 ccTLD，由 Gandi（注册商 ID 81）代理（E 对，D 错）。续期窗 D-30~D-6（A 对）、无 late renewal（B 对）、无 transfer lock 显示 "-"（C 对）。Amazon Registrar 主要负责通用 gTLD（.com/.net/.org 等）。

### 09-4（单选）
客户在域名转移**完成前**关闭了源账号，三个 Amazon Registrar 域名被 suspend 进入 clientHold、公网 DNS 中断。要恢复并完成转移，正确路径是？

- A. 直接从目标账号发起转移即可
- B. 重开源账号 → AES 事件驱动自动 unsuspend → 再走标准跨账号转移；转移必须由源账号发起
- C. 联系 JPRS 解除 clientHold
- D. 等域名进入 redemption 后从池中重新注册
- E. 修改目标账号的 NS 委派即可恢复解析

【答案】B
【解析】账号一关，自助转移路径被切断（转移必须由源账号发起）。恢复路径：重开源账号 → AES 事件驱动自动 unsuspend（文档恢复窗口最长 24h，非合同 SLA）→ 标准跨账号转移。这些是 Amazon Registrar 域名，与 JPRS 无关。

### 09-5（单选）
关于账号关闭触发的域名生命周期时限，下列哪项正确？

- A. 关闭后立即删除域名，无任何窗口
- B. 关闭 → 每日 WILL_SUSPEND 通知 5 天 → SUSPEND_DOMAIN(clientHold) → suspend 约 30 天后进入删除流程 → 90 天 post-closure 永久关闭
- C. suspend 后 7 天即永久删除，不可恢复
- D. suspend 等同于 delete，域名立即回到公开池
- E. 90 天内 hosted zone 会自动恢复解析

【答案】B
【解析】标准时限：每日通知 5 天 → suspend(clientHold) → +30 天进入删除 → 90 天 post-closure 永久。suspend ≠ delete（D 错），clientHold 后仍有约 30 天窗口。域名 unsuspend 后 hosted zone 可能仍处 isolation（NS REFUSED），DNS 恢复是另一条独立恢复链（E 错）。

### 09-6（多选）
关于域名转移（transfer out）的前提条件，正确的有？（多选）

- A. 域名注册满 60 天（新注册/刚转入有 60 天锁）
- B. 解除 transfer lock（若该 TLD 支持）
- C. 取得 EPP auth code（转移授权码）
- D. admin/registrant 邮箱可达（接收转移确认邮件）
- E. 必须先删除 hosted zone

【答案】A、B、C、D
【解析】转移前提：60 天锁（A）、解锁（B，部分 TLD 如 .jp 本就不支持锁）、EPP auth code（C）、联系人邮箱可达（D）。删除 hosted zone 与转移域名无关（E 错，且注册≠解析）。变更 registrant 也常触发 60 天锁。

### 09-7（单选）
关于赎回期（Redemption）与删除，下列哪项正确？

- A. 域名过期后随时可免费找回
- B. 进入 redemption 后 restore 通常收费且不保证成功；进入 pending delete 后不可赎回
- C. redemption 期内域名仍正常解析
- D. pending delete 阶段仍可免费赎回
- E. released 阶段域名自动回到原账号

【答案】B
【解析】赎回期 restore 收费且不保证成功；pending delete 后不可赎回（D 错）；released 后回到公开可注册池，需重新抢注（E 错）。redemption 阶段域名已暂停、不正常解析（C 错）。"过期随时免费找回"是常见错误认知（A 错）。

### 09-8（问答）
简述在排查"客户改了 NS 却发现解析仍走旧值"时，SME 应首先确认的关键点，以及 WHOIS 与 RDAP 的关系。

【答案】
- **关键点**：`update-domain-nameservers` 改的是**注册局侧委派**；只改 hosted zone 的 NS 而不在 Registered domains 侧同步更新注册局 NS 委派，注册局仍指向旧 NS，解析走旧 NS——这是"改了 NS 却不生效"的根因。传播还受父区(TLD)NS 记录 TTL 影响（数分钟到数十分钟）。
- **WHOIS vs RDAP**：WHOIS 是传统文本协议、各注册局格式各异（如 .jp 走 whois.jprs.jp 返回日文字段）；RDAP 是其现代 JSON/HTTP 结构化替代，支持权限分级，ICANN 正逐步以 RDAP 取代 gTLD 的 WHOIS。
【解析】呼应 §1.7 注册侧↔解析侧接缝与 §1.6 目录服务。SME 排查转移/续期/解析类问题还应先查联系人邮箱可达性与续期是否成功。

---

## 第 10 章 · Route 53 Profiles（7 题）

### 10-1（单选）
关于 Route 53 Profile 的本质，下列哪项**最准确**？

- A. Profile 是一种全新的 DNS 资源类型，用来取代 PHZ
- B. Profile 是把 PHZ、Resolver rules、DNS Firewall rule groups、Resolver query logging 配置成组打包、批量关联到多个 VPC 的分发机制
- C. Profile 只能打包 PHZ，不含其它资源
- D. 用了 Profile 之后就不再需要创建 PHZ
- E. Profile 是一种 Resolver endpoint 的高可用版本

【答案】B
【解析】Profile 是**打包分发机制**，不是新资源类型、不取代 PHZ（A、D 错）。它能装四类资源：PHZ、Resolver rules、DNS Firewall rule groups、Resolver query logging 配置（C 错，不止 PHZ）。仍需先单独创建这些资源再放进 Profile。

### 10-2（单选）
一个 VPC 需要下发多组 DNS 治理策略，下列做法正确的是？

- A. 给该 VPC 同时关联 3 个 Profile 分层下发
- B. 把这些策略合并进同一个 Profile，因为一个 VPC 同时只能关联 1 个 Profile
- C. 一个 VPC 最多可关联 5 个 Profile
- D. VPC 无法关联 Profile，只能逐条配置
- E. 一个 Profile 只能关联到 1 个 VPC

【答案】B
【解析】每个 VPC 同时只能关联 1 个 Profile，要下发多组策略必须合并进同一个 Profile（不能叠加）。反向：1 个 Profile 可关联到很多 VPC（E 错，这正是它规模化的意义）。

### 10-3（单选）
某 VPC 本地直接关联了一个 PHZ，同时又通过 Profile 下发了一个同命名空间的 PHZ。查询该命名空间时，Route 53 采用哪个？

- A. Profile 下发的 PHZ 优先
- B. VPC 本地（local）关联的 PHZ 优先于 Profile 下发的同名配置
- C. 随机选择
- D. 报冲突错误，两者都不生效
- E. 合并两者的记录一起返回

【答案】B
【解析】优先级铁律：**local > Profile**。本地是"就近覆盖"，Profile 是"集中默认"。在都属 local 或都属 Profile 的范围内仍遵循最具体匹配。

### 10-4（单选）
跨账号用 Profile 做集中式 DNS 治理，标准分发路径是？

- A. 对每个 PHZ 逐个做 VpcAssociationAuthorization 跨账号授权
- B. 中心账号创建 Profile → 通过 AWS RAM 共享给成员账号/Organization → 成员账号接受后关联到自身 VPC
- C. 把 Profile 复制到每个成员账号
- D. 通过 CloudFormation StackSets 手工同步配置
- E. 跨账号无法使用 Profile

【答案】B
【解析】Profile 通过 AWS RAM 共享给成员账号或整个 Organization，成员账号接受后关联到自身 VPC。这**免去了逐 PHZ 的 VpcAssociationAuthorization** 跨账号授权（A 是被替代的旧繁琐做法）。

### 10-5（多选）
以下哪些资源可以被打包进一个 Route 53 Profile？（多选）

- A. Private Hosted Zones（PHZ）
- B. Resolver rules（转发/系统规则）
- C. DNS Firewall rule groups（含优先级与 fail-open/closed 行为）
- D. Resolver query logging 配置
- E. EC2 安全组

【答案】A、B、C、D
【解析】Profile 可打包四类 DNS 配置：PHZ、Resolver rules、DNS Firewall rule groups、Resolver query logging 配置。EC2 安全组不是 DNS 治理资源，不在其列（E 错）。

### 10-6（单选）
关于 Profile 变更的影响范围，下列哪项正确？

- A. 改 Profile 内配置只影响中心账号自己的 VPC
- B. 改 Profile 内配置会作用于所有关联该 Profile 的 VPC，集中一致但一次误改的爆炸半径覆盖全部关联 VPC
- C. 改 Profile 后需逐个 VPC 手工重新关联才生效
- D. Profile 一旦创建即不可修改
- E. 改 Profile 只影响新关联的 VPC，已关联的不受影响

【答案】B
【解析】集中定义→自动传播到所有关联 VPC 是 Profile 的优点，但也意味着误改的爆炸半径覆盖全部关联 VPC，变更需谨慎评审。无需逐个重关联（C 错），Profile 可修改（D 错）。

### 10-7（单选）
何时应把逐 VPC 手工配置切换为 Route 53 Profiles？

- A. 只有单个 VPC 时
- B. 当 PHZ-VPC 关联逼近每 PHZ 300 VPC 上限、或几十上百 VPC 逐个配置不可维护时
- C. 仅当需要 DNSSEC 时
- D. 仅当使用公网 hosted zone 时
- E. Profiles 只适用于跨 Region 场景

【答案】B
【解析】Profiles 定位于治理规模化：当每 PHZ 300 VPC 上限逼近、或几十上百 VPC 逐个配置运维爆炸时，是标准治理方案。单 VPC（A）用不上；与 DNSSEC/公网 zone/跨 Region 无必然绑定。

---

## 第 11 章 · Application Recovery Controller (ARC) Routing Controls（8 题）

### 11-1（单选）
ARC Routing Control 与普通 Route 53 Health Check 的**根本区别**是？

- A. Routing control 主动探测端点，探测失败自动切流
- B. Routing control 是人工/编程拨动的确定性开关，背后改一个 R53 health check 的状态来切流，不依赖对应用的探测
- C. 两者完全等价，只是 ARC 更贵
- D. Routing control 只能用于单 Region
- E. 普通 health check 无法用于 failover 记录

【答案】B
【解析】ARC routing control 是"开关"不是"探针"——由人/自动化确定性拨动，背后驱动一个 health check 状态翻转，进而切 failover 记录。规避了灰色故障下探测不准或探测本身故障导致无法可靠切流的问题。

### 11-2（单选）
灾难发生时，切换 routing control 状态应使用哪条路径？

- A. Control plane（route53-recovery-control-config）
- B. Cluster 的 data plane endpoint（route53-recovery-cluster，5 个 Region endpoint，轮询直到成功）
- C. 直接改 Route 53 记录
- D. 修改 hosted zone 的 NS 委派
- E. 通过 Service Quotas 控制台

【答案】B
【解析】考点铁律：平时用 control plane 配置，**救灾切流必须用 cluster 的 5 个 data plane endpoint**（轮询直到一个成功）。control plane 在大区域性灾难中可能不可达。

### 11-3（单选）
ARC Cluster 的高可用设计是？

- A. 单 Region 部署
- B. 由 5 个 AWS Region 组成，读写 routing control 状态需至少 3 个（3/5 quorum）一致，即使 2 个 Region 不可用仍可切换
- C. 由 3 个 Region 组成，需全部在线
- D. 由 5 个 Region 组成，需 5 个全在线才能切
- E. 由 2 个可用区组成

【答案】B
【解析】Cluster 跨 5 个 Region，3/5 quorum；即使 2 个 Region 挂掉仍可靠切换。这保证"救灾工具本身不和被救系统同挂"。需全部在线（C、D）是错误认知。

### 11-4（多选）
关于 Safety Rules，正确的有？（多选）

- A. 核心目的是防止一次误操作把所有 Region 同时关掉（防"全关"）
- B. Assertion rule 约束一组 routing control 必须满足某条件才允许改变（如至少 1 个为 On）
- C. Gating rule 用一个"门"control 允许/禁止对另一组 control 的更改
- D. Safety rule 会主动探测应用健康
- E. 有了 safety rule 就不再需要 data plane endpoint

【答案】A、B、C
【解析】Safety rule 是护栏，防"全关"与防误触（A、B、C 对）。它不探测应用（D 错，那是普通 health check 的事）；切换仍必须走 data plane endpoint（E 错）。

### 11-5（单选）
Routing control 到 DNS 切流的绑定链条，正确顺序是？

- A. routing control → failover 记录 → health check → DNS
- B. routing control → 驱动 R53 health check(Type=RECOVERY_CONTROL) → 关联的 failover 记录(Primary/Secondary) → DNS 应答切换
- C. health check → routing control → cluster → DNS
- D. failover 记录 → cluster → routing control → DNS
- E. routing control 直接改 A 记录的 IP

【答案】B
【解析】链条：拨 routing control → 改其驱动的 health check(Type=RECOVERY_CONTROL)状态 → 关联的 failover(Primary/Secondary)记录按 health check 判定切换 → DNS 完成切流。routing control 不直接改记录 IP（E 错）。

### 11-6（单选）
在 Active/Standby 双 Region DR 场景中（rc-primary 对 us-east-1、rc-secondary 对 us-west-2），下列哪个 safety rule 能防止运维一次误操作导致全站无端点？

- A. 允许 rc-primary 与 rc-secondary 同时为 Off
- B. Assertion rule：约束 [rc-primary, rc-secondary] 中为 On 的数量必须 ≥1（ATLEAST 1），禁止同时全 Off
- C. 把两个 control 合并成一个
- D. 关闭 safety rule 以加快切换
- E. 只对 rc-primary 设 gating rule

【答案】B
【解析】ATLEAST 1 On 的 assertion rule 会在试图把最后一个 On 也关掉时拒绝切换，防止"两边都没流量"的自陷式全局中断。这正是 Lab D 演示的护栏。

### 11-7（单选）
为 routing control 建的 Route 53 health check，其 Type 与关键属性应是？

- A. Type=HTTP，带探测 URL
- B. Type=RECOVERY_CONTROL，用 RoutingControlArn 关联（由 routing control 驱动，非探测型）
- C. Type=CALCULATED，聚合多个子 health check
- D. Type=CLOUDWATCH_METRIC，绑定告警
- E. Type=TCP，探测端口

【答案】B
【解析】ARC 的 health check 是 `Type=RECOVERY_CONTROL`，通过 `RoutingControlArn` 关联，状态由 routing control 直接驱动而非主动探测。其它类型都是探测/计算型 health check。

### 11-8（问答）
在 DR 演练中，团队通过一个 data plane endpoint 把 rc-secondary 切为 On 并观察到一次 `dig` 成功返回 us-west-2 端点，就宣布"切换路径完全健康"。请指出这个结论的纪律缺陷。

【答案】
一次查询成功不足以证明整条切换路径健康：① cluster 有 5 个 data plane endpoint，仅验证了其中一个可达，未证明灾难时其余 endpoint 的可用性（生产脚本应轮询 5 个直到成功）；② 一次 `dig` 成功只反映当前解析结果，不能证明 failover 记录/health check 在各种故障组合下都会正确切换；③ 应结合 safety rule 验证（如尝试全关被拒）以确认护栏在效。宁可说"已验证经该 endpoint 可切换"，不下"整条路径完全健康"的充分性结论。
【解析】呼应"单点成功 ≠ 全局健康"与"必要非充分"纪律（与 topic 13 一致）。

---

## 第 12 章 · 服务集成 + 配额与限流（8 题）

### 12-1（单选）
客户问"为什么在根域 `example.com` 上配 CNAME 报错"，SME 的正确回答是？

- A. 根域 CNAME 需要额外付费才能启用
- B. 标准 DNS 禁止在 zone apex 放 CNAME，应改用 Alias 记录（Alias 可用于 apex）
- C. 根域只能用 TXT 记录
- D. 需要先启用 DNSSEC
- E. CNAME 在 apex 需要设置 TTL=0

【答案】B
【解析】标准 DNS 禁止 apex 放 CNAME；Route 53 的 Alias 记录没有此限制，可用于 apex，且指向 AWS 资源不额外收费、AWS 自动维护目标 IP。

### 12-2（多选）
关于 Alias 到 S3 静态网站的硬约束，正确的有？（多选）

- A. 目标必须是"网站托管"endpoint（s3-website-<region>），不是普通 REST endpoint
- B. bucket 名必须与记录名完全一致
- C. region 要选对
- D. Alias 到 S3 网站支持 Evaluate Target Health
- E. 可以直接 Alias 到普通 S3 REST endpoint

【答案】A、B、C
【解析】S3 网站 Alias 三硬约束：网站 endpoint（A）、bucket 名 == 记录名（B）、region 选对（C）。S3 网站 Alias **不支持** Evaluate Target Health（D 错）；不能用普通 REST endpoint（E 错）。

### 12-3（单选）
关于 Evaluate Target Health 的支持面，下列哪项正确？

- A. CloudFront、S3 网站、ELB 都支持
- B. ELB 支持；CloudFront 与 S3 网站不支持
- C. 只有 CloudFront 支持
- D. 所有 Alias 目标都强制开启
- E. 只有 API Gateway 支持

【答案】B
【解析】ELB 支持 Evaluate Target Health（Route 53 看目标组是否有健康目标）；CloudFront（全球边缘无"目标"概念）与 S3 网站不支持。别承诺 CloudFront Alias 能做目标健康评估。

### 12-4（单选）
下列关于 Route 53 数据面 vs 控制面限流的说法，哪项正确？

- A. 公网权威 DNS 查询有严格的每账号 QPS 配额
- B. 公网权威查询几乎无限、不计账号配额；被限流的是控制面 API（约 5 请求/秒/账号）和 Resolver endpoint（per-ENI ~10,000 QPS）
- C. 控制面 API 无任何限流
- D. Resolver endpoint 无 QPS 上限
- E. 所有查询都计入统一的账号级 QPS 限流

【答案】B
【解析】公网权威查询由 AWS anycast 车队承载，属数据面、不计账号配额；能被限流的是控制面 API（~5 req/s，如 ChangeResourceRecordSets）与 Resolver endpoint（per-ENI ~10k QPS，加 IP 线性扩）。混答是高频扣分点。

### 12-5（单选）
需要一次性修改一个 hosted zone 里的上百条记录，正确做法是？

- A. 写脚本循环调用 change-resource-record-sets，每条一次，每秒 >5 次
- B. 用单次 ChangeResourceRecordSets 请求打包多个变更，避免触发 Throttling/PriorRequestNotComplete
- C. 先提额把 hosted zone 上限调到 10 万
- D. 分账号并发调用绕过限流
- E. 逐条调用但加 sleep 到每秒恰好 5 次

【答案】B
【解析】控制面 ~5 req/s，循环单条调用会秒级触发 `Throttling`/`PriorRequestNotComplete`。正解是用单次 ChangeResourceRecordSets **打包多条 Changes**。

### 12-6（多选）
关于 `ip-ranges.json` 中 Route 53 相关 service 的三分层（呼应 NBC 案例），对应正确的有？（多选）

- A. ROUTE53 = 公网权威 NS 的 IP 范围
- B. ROUTE53_RESOLVER = VPC 内递归解析器（有 per-ENI QPS 限流的那层）
- C. ROUTE53_HEALTHCHECKS = 健康检查探测车队
- D. ROUTE53 = VPC 内递归解析器
- E. 三者对应同一条数据路径

【答案】A、B、C
【解析】三分层分别对应权威/递归/探测三条完全不同的数据路径：ROUTE53=权威 NS、ROUTE53_RESOLVER=递归（有 QPS 限流）、ROUTE53_HEALTHCHECKS=探测。D、E 混淆了路径。

### 12-7（单选）
NBC（National Bank of Canada）case 中，客户想在防火墙放行 AWS 公共 DNS 服务器 IP 以实现 zone delegation。SME 的关键判读**不含**下列哪项？

- A. 要放行的是 ROUTE53（权威 NS）的 IP，不是 ROUTE53_RESOLVER
- B. 公网权威查询由 anycast 承载、不计账号限流，客户无需为 QPS 提额
- C. NS IP 是静态的但仍可能新增前缀，应全量收录并订阅 SNS 变更监控
- D. NS delegation 本身需要在防火墙放行入站 53，且必须逐个 region 过滤 IP
- E. 需放行的是客户递归解析器 outbound 到权威 NS（目的端口 53）的查询流量

【答案】D
【解析】题问"不含"（即错误项）。NS delegation 本身不需要防火墙改动；需放行的是客户递归 outbound→权威 NS（UDP/TCP 目的 53）。放行应**全量收录、不按 region 过滤**（实测含 8 种 region 字段）并订阅 SNS（AmazonIpSpaceChanged）兜底。D 两处都错。A/B/C/E 均为正确判读。

### 12-8（单选）
关于 Global Accelerator (GA) 与 Route 53 选路的分工，下列哪项最准确？

- A. GA 和 Route 53 二选一，不能叠加
- B. GA 提供两个静态 anycast IP，在网络层做最优接入（客户端到入口的路径）；Route 53 在 DNS 层决定把哪个 IP 返给客户端；二者可叠加
- C. GA 在 DNS 层选路，Route 53 在网络层选路
- D. GA 是 Route 53 延迟路由的替代品，功能完全等价
- E. GA 只能用普通 A 记录、不能用 Alias 指向

【答案】B
【解析】GA 解决"客户端到 AWS 入口的网络路径"（静态 anycast IP、走骨干网），Route 53 解决"把哪个 IP 返给客户端"（DNS 层选路），分工不同、可叠加。Route 53 可用 A 记录指向两个静态 IP，或用 Alias 指向 accelerator DNS 名（E 错）。

---

## 第 13 章 · 通用故障排查纪律（横切，7 题）

### 13-1（单选）
一台 EC2 解析某域名失败。判断故障范围时，SME 首先应做的是？

- A. 立即断定 Route 53 服务整体故障
- B. 先界定 blast radius——是所有 client 还是这一台？所有域名还是这一个？所有查询还是偶发？再归因
- C. 直接建议客户提额
- D. 假定是记录被删并重建记录
- E. 直接对客下单一根因结论

【答案】B
【解析】单点失败 ≠ 服务故障，一次成功 ≠ 路径整体健康。纪律：永远先界定故障范围再归因。确定性全失败指向"所有路径共有的一环"，概率性失败指向"多路径中某一条"。

### 13-2（单选）
客户报告 EC2 `nslookup jcrew.com 10.201.244.2` 三次全部 timeout（VPC CIDR 为 10.201.244.0/22）。AI 初稿建议"检查 EC2 到 .2 的 SG/NACL/路由"。SME 的正确纠正是？

- A. 初稿正确，按建议检查 SG/NACL
- B. 10.201.244.2 = VPC CIDR base + 2 = AmazonProvidedDNS，EC2→.2 这段不经 SG/NACL/路由；真正阻断点在下游（Resolver Outbound Endpoint 的 ENI 子网 NACL、转发目标路由）
- C. timeout 说明记录不存在，应检查记录配置
- D. 应先重启 EC2
- E. 直接判定为限流，建议提额

【答案】B
【解析】AmazonProvidedDNS = VPC CIDR base+2，EC2→该地址不过 SG/NACL/路由（AWS 底层实现）。方向应指向下游 Resolver Endpoint 的 ENI 子网 NACL/转发目标路由。且 timeout 是网络层信号，不是"记录没配"（C 错），也不能直接当限流（E 错）。

### 13-3（单选）
排查中确认了一个 NACL 拦截了 DNS 流量。关于是否可承诺"修这个 NACL 就恢复"，正确的纪律是？

- A. 可以承诺，找到一个阻断点即可结案
- B. 修它是恢复的必要条件但不一定充分——可能同时存在第二个问题（如 TGW 路由不可达、目标 DNS 不响应）；除非已证明是唯一阻断点，否则不承诺"修完即恢复"
- C. 必须先修完所有可能问题再回复
- D. 阻断点越多越应该承诺一次修好
- E. 只要是 DIRECTLY_OBSERVED 的阻断就等于充分条件

【答案】B
【解析】必要非充分：多 ENI/多路径下常有第二处问题。措辞用"这是一处需修复的阻断；同时请确认 A/B/C"，不做充分性承诺。DIRECTLY_OBSERVED 证明"这处确实阻断"，不等于"唯一阻断"（E 错）。

### 13-4（多选）
关于 DNS 响应码/无响应的排查方向，正确的对应有？（多选）

- A. timeout（无响应）→ 网络层：SG/NACL/路由阻断、目标 DNS 宕、UDP 分片丢失
- B. NXDOMAIN → 权威明确回答"此名不存在"，是数据问题（记录缺失/拼写/未创建），不是网络问题
- C. SERVFAIL → 递归/权威处理失败：DNSSEC 验证失败、转发目标不响应、上游超时
- D. REFUSED → 服务器拒绝应答：权限/ACL、非授权区、递归被关
- E. NXDOMAIN → 一定是网络阻断导致

【答案】A、B、C、D
【解析】timeout=网络层（A）、NXDOMAIN=数据问题（B，E 错）、SERVFAIL=处理失败（C）、REFUSED=拒绝应答（D）。把 timeout 当"记录没配"、把 NXDOMAIN 当网络问题都是走错方向。

### 13-5（单选）
客户三个域名"昨天正常、今早全挂"。SME 的首要归因方向与标准取证序列是？

- A. 逐条记录单独排查，先看第一个域名的某条 A 记录
- B. 多资源同时失效 → 找公共因子（同账户/同 Hosted Zone/注册商 NS 被改）；标准序列：whois → dig <domain> NS → 控制台查 Hosted Zone/NS → CloudTrail 查 24-48h 变更事件
- C. 直接判定为 DDoS 攻击
- D. 建议客户重新注册这三个域名
- E. 先启用 DNSSEC 再观察

【答案】B
【解析】三域名同时失效指向"三者共有的一环"，而非单条记录。标准"注册/委派链"取证序列：whois→dig NS→控制台→CloudTrail（ChangeResourceRecordSets/DeleteHostedZone/UpdateDomainNameservers）。逐一给验证命令、不预先断言单一根因。

### 13-6（单选）
客户改完记录后说"还是不通"。SME 应先做什么？

- A. 立即判定记录改错了并回滚
- B. 先问"等了多久？用了哪个递归器？"，再直查权威（dig @权威NS）确认改动已生效——递归失败但权威成功 = 负缓存/TTL 滞后，不是配置错
- C. 直接重建整个 hosted zone
- D. 判定为限流并提额
- E. 关闭 DNSSEC

【答案】B
【解析】负缓存（SOA minimum/negative TTL）会缓存 NXDOMAIN，TTL 也会缓存旧值。递归失败但直查权威成功 = 缓存滞后而非配置错。用 SOA minimum 估算等待时间。

### 13-7（问答）
简述 SME 排查中的"证据分级（四级）"及其在回复措辞上的纪律。

【答案】
四级证据强度：
- **DIRECTLY_OBSERVED**：从客户真实导出/日志/工具输出直接读到 → 措辞"已确认/observed"。
- **DOCUMENTED_FACT**：官方文档明确规定的行为（如 VPC DNS = CIDR base+2）→ 措辞"根据文档"。
- **INFERENCE**：由观测+文档推出、存在反例可能 → 措辞"suggests/很可能"，不用"is/确认"。
- **UNKNOWN**：未验证、需客户提供 → 列入索取清单，绝不当事实写进结论或承诺。

纪律：回复中每个断言都要能对应到一个级别；证据不足时不为了给"确定答案"而把 INFERENCE 说成根因（不做唱因归因）。宁可回复"这是一处确认的阻断，另有 A/B 需确认"，也不唱一个未经证实的单一根因。
【解析】呼应 §1.2 证据分级与 §1.8 不做唱因归因，是横切所有 R53 case 的核心方法论。

---

> 题库结束。合计 46 题（08:8 · 09:8 · 10:7 · 11:8 · 12:8 · 13:7）。


================================================================================

# FILE: exam/exam-bank-advanced-tt-principles.md
<!-- SOURCE FILE: exam/exam-bank-advanced-tt-principles.md -->

# Route 53 SME 进阶练习题库 —— 真实故障模式诊断 + 底层技术原理

> **素材来源**：`research/internal/r53-tt-failure-patterns.md`（内部 TT/case/COE 故障模式）、`research/internal/r53-sage-qa-supplement.md`（Sage/answers 专家口径）、`topics/14-technical-deep-principles.md`（底层机制深挖）。
> **题型**：场景单选（A-E）/ 多选。每题含 题干 + 选项 + 【答案】+ 【解析】（引回原理/故障模式）。
> **难度定位**：SME 级——侧重"给现象问根因/下一步排查"与"底层为什么这么设计"。
> **共 48 题**，按主题分章。多选题在题干标注"（多选）"。

---

## 第一章 真实故障模式诊断 —— 解析优先级 / PHZ / 转发规则（10 题）

### 1. VPC 内 dig 一条公网记录返回 NXDOMAIN，但指定 8.8.8.8 能正常解析。最可能的根因是？
- A. VPC 的 enableDnsSupport 被关闭
- B. 该域名存在一个覆盖它的 Private Hosted Zone，PHZ 内无此记录且不 fallback 公网
- C. Resolver Endpoint 的 QPS 超限
- D. DNSSEC 验签失败返回了 NXDOMAIN
- E. 公网权威服务器故障

【答案】B
【解析】重叠命名空间经典陷阱。为某域建 PHZ 后会生成 autodefined 规则，查询名落入 PHZ 即在 PHZ 内权威解析；PHZ 里没有该记录时**直接返回 NXDOMAIN，不 fallback 到公网**。8.8.8.8 不受 VPC PHZ 影响故能解析。正解是 split-view DNS。（enableDnsSupport 关了会整体解析失败，不是选择性 NXDOMAIN；DNSSEC 验签失败返回的是 SERVFAIL 而非 NXDOMAIN。）

### 2. 某 VPC 同时关联了 DNS Firewall、一条自定义 forward rule、一个 PHZ。一个查询进来，`.2` Resolver 的评估顺序是？
- A. PHZ → forward rule → DNS Firewall → 公网
- B. forward rule → PHZ → DNS Firewall → 公网
- C. DNS Firewall → Resolver 规则(forward rule) → Private Hosted Zone → 公网
- D. 公网 → PHZ → forward rule → DNS Firewall
- E. 按记录创建时间先后评估

【答案】C
【解析】`.2` Resolver 对每个查询按固定优先级评估：**DNS Firewall → Resolver 规则 → PHZ → 公网**，且 longest match 优先。Firewall BLOCK 的查询不再往下走。

### 3.（多选）关于"同一域名同时存在 PHZ 与自定义 forward rule"，下列正确的是？
- A. custom forward rule 默认压过 autodefined system rule
- B. 更长前缀的子域 PHZ 会赢过父域的 forward rule（longest match 优先于 rule 类型优先）
- C. 等长时 PHZ 永远胜出
- D. 给 example.com 加 outbound forward rule 想 fallback 公网是反模式，会让整域走公网、PHZ 失效
- E. forward rule 与 PHZ 冲突时随机选择

【答案】A、B、D
【解析】custom forward rule 显式定义、优先级最高（压过 autodefined）；但 longest match 优先于 rule 类型——子域 PHZ 前缀更长会赢过父域 forward rule。用 forward rule 给整域做公网 fallback 是反模式（forward rule 压过 autodefined，整域走公网、私区失效、白花 endpoint 费）。C 错在"等长时看类型（custom forward 胜）"，E 错在并非随机。

### 4. ECS 服务报 UnknownHostException，删掉某个 PHZ 后立即恢复。根因与两条修复路径是？
- A. 根因是 QPS 超限；修复是加 ENI 或降 TTL
- B. 根因是一个父域 PHZ 把子服务查询"吸走"返回空（私区最长匹配 + 不 fallback）；修复是删掉该 PHZ，或在该 PHZ 内补齐指向 LB 的镜像 A 记录
- C. 根因是 DNSSEC 信任链断裂；修复是撤父区 DS
- D. 根因是 conntrack；修复是去掉限制性 SG
- E. 根因是委派跳级；修复是逐级委派

【答案】B
【解析】私区与公区不能共享同一根域重叠，私区优先且从最靠近根域的区开始解析。多余的父域 PHZ 把查询吸走返回空。修复两条路：删区，或在该 PHZ 内补镜像记录。

### 5. PrivateLink 端点解析不出私有 IP，CloudAuth token prefetch connect timed out。首先应检查哪两个 VPC 属性？
- A. enableDnsSupport 和 enableDnsHostnames
- B. mapPublicIpOnLaunch 和 assignIpv6
- C. DNS Firewall 状态和 Resolver rule
- D. Flow Logs 和 VPC Peering
- E. NACL 和安全组

【答案】A
【解析】PHZ 解析要求 VPC 同时打开 enableDnsSupport 和 enableDnsHostnames，两者都为 true 才生效。老 VPC / 手工建的 VPC 常默认没开 hostnames。VPCE 私有 DNS、Amazon 提供的 DNS 名解析都依赖它。

### 6. `dev.api.example.com` 间歇性 ERR_NAME_NOT_RESOLVED，偶尔又能解析。最可能的委派配置错误是？
- A. NS 记录 TTL 设得太长
- B. 把 dev.api.example.com 直接从 example.com 委派，而非从直接父区 api.example.com 逐级委派
- C. DNSSEC 未启用
- D. PHZ 未关联 VPC
- E. Resolver endpoint 不足

【答案】B
【解析】多级子域委派必须从直接父区委派，不能跳级。跳级委派（从 example.com 直接委派 dev.api.example.com）会造成间歇失败。核对链 example.com → api.example.com → dev.api.example.com，每级 NS 建在直接父区。

### 7. 客户想在 R53 私有区里用 NS 记录把子区委派给 on-prem BIND，操作失败。原因是？
- A. 私有区 NS 记录需要额外付费
- B. R53 私有区不支持 delegation/NS 记录；纯 R53 场景用"多 PHZ + 重叠命名空间 + Resolver 规则"等效替代
- C. on-prem BIND 版本过旧
- D. 需要先启用 DNSSEC
- E. 私有区 NS 记录 TTL 必须大于 172800

【答案】B
【解析】R53 私有区不支持 NS 委派。纯 R53 一般不需委派（多 PHZ + overlapping namespace + resolver rule 即可等效）；只有子区要委派到非 R53 权威（on-prem）时才撞功能缺口，需用 DNS forwarding 变通。

### 8. 跨账号把 PHZ 关联到另一账号的 VPC，在控制台找不到入口。正确做法是？
- A. 控制台开启跨账号共享开关即可
- B. 用 CLI/SDK/API：先在 PHZ 所有者账号 create-vpc-association-authorization，再在 VPC 所有者账号 associate-vpc-with-hosted-zone；或用 Route 53 Profiles 免授权步骤
- C. 必须先合并两个账号到同一 Organization
- D. 只能通过 Support case 手工关联
- E. 先删除 PHZ 再在目标账号重建

【答案】B
【解析】跨账号关联 PHZ 无法在控制台做，必须 CLI/SDK/API 两步授权；Route 53 Profiles 可免此授权步骤。

### 9. 为什么"按客户端 IP/ISP 做 split-horizon 路由"不可靠？
- A. R53 不支持 A 记录
- B. 无法保证终端用户用的是其 ISP 的 DNS（可能用 8.8.8.8），源 IP 判断会错、返回错记录；此类需求应转为访问控制问题
- C. split-horizon 需要企业级支持合同
- D. 私网 IP 不能放进 public zone
- E. R53 会随机丢弃一半查询

【答案】B
【解析】控制不了终端用户用哪个 resolver，据源 IP 判断会错。专家建议转为访问控制（Lambda@Edge 判 IP 段重定向 + 边缘封锁），而非靠 DNS。（私网 IP 其实可以放进 public zone。）

### 10.（多选）关于 dangling delegation（悬挂委派）的安全风险与正确操作顺序，下列正确的是？
- A. 悬挂委派属安全 Sev2，攻击者可能接管子域
- B. 删除子 zone 的正确顺序：先删父域 NS 委派、再删子 zone
- C. 迁移子 zone 时先更新父域 NS 记录
- D. R53 有防护栏自动阻止悬挂委派
- E. 悬挂委派只影响解析延迟，不涉及安全

【答案】A、B、C
【解析】父域 NS 委派指向的子 zone 被删/迁到别 tenant 后，委派 NS 指向已无权威的 nameserver，攻击者可接管。几乎没有防护栏自动阻止（D 错）。删/迁子 zone 要先处理父域 NS 委派。E 错——这是安全问题。

---

## 第二章 真实故障模式诊断 —— Resolver Endpoint 限流 / QPS / 连接跟踪（8 题）

### 11. 客户问"1024 PPS 和 10,000 QPS 是不是一回事"。正确区分是？
- A. 是同一个限，两个说法而已
- B. `.2` Resolver 每 ENI 硬限 1024 PPS（不可提升，缓存命中/IMDS 查询也算）；Resolver Endpoint 每 ENI/IP 约 10,000 QPS（超 50% 应加 ENI），是两个不同的限
- C. 1024 PPS 可提升，10,000 QPS 不可提升
- D. 两者都可通过 Support 提额
- E. 只有公网权威查询才有这两个限

【答案】B
【解析】两个不同硬限：`.2` 每 ENI 1024 PPS 不可增（缓存命中、IMDS 都吃额度）；Resolver Endpoint 每 ENI 约 10K QPS 可加 ENI。多数 1 查询=1 UDP 包 PPS≈QPS，EDNS0 大包/切 TCP 时不等。

### 12. 某出站 Resolver Endpoint ENI 明明没到 10K QPS 却大量丢包超时，并发布了 `conntrack_allowance_exceeded_delta` 指标。根因是？
- A. 客户超过了 10K QPS 硬限
- B. ENI 使用了限制性安全组规则、或查询经过 NLB，触发 connection tracking，把 UDP QPS 压到低至约 1,500 QPS
- C. 目标 on-prem 名服务器不可达
- D. DNSSEC 验签占用了带宽
- E. IMDS 偷走了额度

【答案】B
【解析】NX 底层用 conntrack。限制性 SG 或经 NLB 强制 conntrack 时，UDP QPS 被压到约 1,500 QPS（约 6 倍降幅）。conntrack 丢包发布 `conntrack_allowance_exceeded_delta`；纯 10K 超限发布 `udp_throttled_count`。修复：去掉限制性 SG / 不走 NLB。

### 13. 内部 runbook 判定：`conntrack_allowance_exceeded_delta` 和 `udp_throttled_count` 两个指标都 NOT OK。如何判定主因？
- A. 一定是 conntrack
- B. 一定是 throttling
- C. 看 ENI 迁移时机——迁移到大实例前已在大实例上则主因是 throttling；迁移中才升到 CONN_TRACK_FLEXI_FLEET 则主因是 conntrack
- D. 两个指标不可能同时 NOT OK
- E. 无法判定，直接加 ENI

【答案】C
【解析】两者都中时看 capacity_level 迁移时机：迁移前已在大实例 → 主因 throttling；迁移中才升到 CONN_TRACK_FLEXI_FLEET → 主因 conntrack。修复：throttling 全 ENI 中招→加 ENI，部分中招→均衡分流；conntrack→去 SG 限制/不走 NLB。

### 14. 出站解析大面积超时/SERVFAIL，但 conntrack 与 throttling 指标都正常。下一步最应怀疑什么、怎么定位？
- A. 客户 SG 配错；抓 SG 规则
- B. 目标 on-prem 名服务器无响应/不可达；用 CloudWatch Insights 按分钟算 timeout%，再对 Athena 表按 destip 聚合 respcode='TIMEOUT' 统计每个目标 NS 的超时数
- C. DNSSEC 信任链断；检查 DS
- D. PHZ 重叠；列出关联 PHZ
- E. VPC DNS 属性未开；检查 enableDnsHostnames

【答案】B
【解析】内部 runbook 明确：出站 endpoint 支持工单最常见根因是目标 NS 不可达。限流指标正常时优先怀疑目标 NS，用 timeout% + 按 destip 聚合超时定位。

### 15. on-prem 发查询到 Inbound Resolver Endpoint，收不到任何响应（连错误都没有），排查极难。底层原因是？
- A. Inbound endpoint 只接受递归查询（RD=1），收到迭代查询会立即静默丢弃、无错误响应
- B. Inbound endpoint 返回了 REFUSED 但被防火墙拦截
- C. 安全组把响应包丢了
- D. 查询触发了 DNS Firewall BLOCK
- E. QPS 超限被静默丢弃

【答案】A
【解析】R53 Resolver endpoint 是递归 resolver，只接受递归查询。发迭代查询给 Inbound endpoint 会被立即丢弃、无错误响应（不是 REFUSED，是无响应），这是 forwarding 的硬性要求。

### 16. 2026-02-04 TRE（税务引擎）DNS 大面积超时约 1 小时，同区 Datapath/OPF 无恙。COE 根因是？
- A. R53 权威 anycast edge 故障
- B. TRE 用共享出站 resolver endpoint，一次 Alexa 变更重启 Redis 使其 VPC 产生 2.76 亿+ 查询把共享 endpoint 打到峰值 821K QPS；Datapath/OPF 因有专用 endpoint 未受影响
- C. DNSSEC 轮换失败
- D. 控制面 API 限流
- E. NLB fail-open

【答案】B
【解析】共享出站 endpoint 的"吵闹邻居"事件。多租户共享 endpoint 无隔离，一个租户的查询洪峰把共享 endpoint 打满，其他共享租户受牵连；专用 endpoint 因隔离未受影响。SME 考点：专用 vs 共享 endpoint 的容量与隔离权衡。

### 17. ChangeResourceRecordSets 报 HTTP 400 `Throttling / Rate exceeded`。正确理解是？
- A. DNS 查询 QPS 超限，需加 ENI
- B. 这是控制面 API 限流（每账户 5 请求/秒），与数据面 DNS 查询限流无关；公网权威 DNS 查询 QPS 无限制
- C. 需要给 hosted zone 提额
- D. `.2` Resolver 的 1024 PPS 超限
- E. conntrack 触发

【答案】B
【解析】Route 53 API 每账户 5 rps（控制面）；DNS 查询是数据面不受此限。不要把 API throttling 与 resolver QPS 混为一谈。

### 18. 为什么"出站 endpoint 的 QPS"与"收到的入站 QPS"不相等？
- A. 因为出站会压缩查询
- B. R53 Resolver 给每个入站请求生成冗余出站查询（并行发到多个 target/cell），故出站 QPS 与入站不等
- C. 出站只处理未缓存的一半
- D. 出站丢弃了迭代查询
- E. NLB 复制了流量

【答案】B
【解析】R53 Resolver 为每个入站请求生成冗余出站查询，所以出站 ENI 的 QPS 与收到的 QPS 不相等（这也是高可用的一部分）。

---

## 第三章 真实故障模式诊断 —— Health Check / Failover（8 题）

### 19. 内部 ALB 后端不健康，但 R53 failover 不切换。根因是？
- A. R53 health check 从全球公网 endpoint 发起，无法探测私有 IP；内部资源要用 CloudWatch alarm 型 HC，或对 ELB 用 Evaluate Target Health(ETH)
- B. ALB 不支持 R53 健康检查
- C. failover 记录 TTL 太长
- D. 缺少 DNSSEC
- E. ETH 只对公网 ELB 有效

【答案】A
【解析】R53 HC 从公网发起，探不了私有 IP。内部资源用 CW-alarm HC 或 ELB 的 ETH。所有 ELB（含内部）自带 R53 health check，指向 ELB 的 Alias 勾选 ETH 即可免费用。

### 20.（多选）关于 Evaluate Target Health (ETH) 的语义，下列正确的是？
- A. ETH 只对 Alias 记录有效；非 Alias 记录用关联的 Health Check
- B. Simple routing 下 ETH 无效（单记录永远按值应答）
- C. Alias + ETH 与显式 HC 同时开时，两者都必须通过才算健康（AND），best practice 是只开一个
- D. ELB（含 internal/私网）已被 R53 健康检查，勾 ETH 即可，无需自建 endpoint HC
- E. ETH 对普通 A 记录 + Weighted 路由同样直接生效

【答案】A、B、C、D
【解析】ETH 仅 Alias；Simple routing 下无效；ETH+HC 同开是 AND 且不推荐；ELB 已被 R53 检查，勾 ETH 即可。E 错——非 Alias 记录用的是"关联的 Health Check"，不是 ETH。

### 21. 某区 NLB 后端 100% 不健康，R53 仍往该区发流量、跨区 failover 不触发。根因是？
- A. ETH 未开
- B. NLB fail-open：目标全部不健康时 NLB 仍放行，于是 ETH 认为"健康"，跨区 failover 不触发
- C. health check 间隔太长
- D. 记录 TTL 过短
- E. DNSSEC 阻断

【答案】B
【解析】NLB fail-open 是"全不健康仍收流量"的根因——ETH 依赖 LB 上报，LB fail-open 就骗过 ETH。方案：改用 failover routing 配合能反映真实后端的 HC（CW 复合指标/应用级 HC）。

### 22. 新建的 HTTPS health check 立刻 unhealthy。哪些是合理根因？（多选）
- A. 探测路径需要鉴权，应用返回 403（403 视为不健康）
- B. HTTPS 探测未开 SNI 或证书不覆盖被探 FQDN
- C. 探测路径返回 5xx
- D. 证书已过期
- E. 记录 TTL 设为 0

【答案】A、B、C
【解析】health check 语义：2xx/3xx 才算健康，403/5xx 算失败；HTTPS 需正确 SNI + 有效证书覆盖被探 FQDN。注意 D 是陷阱——**HTTPS 健康检查不校验证书**，证书过期不会让 HC 失败（见第六章原理题）。E 与 HC 健康判定无关。

### 23. Failover 双记录返回规则，下列哪个是正确的兜底行为？
- A. 主备都不健康 → 不返回任何记录
- B. 主备都不健康 → 返回主（fail-open 到 primary，不可配置）
- C. 主备都不健康 → 返回备
- D. 主备都不健康 → 随机返回
- E. 主备都不健康 → 返回 SERVFAIL

【答案】B
【解析】主健康→只返回主；主不健康备健康→返回备；主备都不健康→返回主（last resort fail-open，不可配置）。

### 24. 基于 CloudWatch alarm 的跨区 failover 配置里，主/备记录的 ETH 应如何设置？
- A. 主备 ETH 都设 True
- B. 主备 ETH 都设 False（健康信号来自 CW alarm 而非 target 本身），否则 failover 不触发
- C. 主 True、备 False
- D. 主 False、备 True
- E. ETH 与 CW alarm 互斥，不能同时用

【答案】B
【解析】CW-alarm 跨区 failover 的 HC 指向目标区自定义 CW 指标/alarm，主备 ETH 都要设 False，否则 failover 不触发（真实 case 卡点）。

### 25. ETH=True 对 NLB 的判定（cross-zone 关闭），下列描述正确的是？
- A. 只看该 AZ 是否有 NLB 节点在监听端口，不看后端 target
- B. 既看该 AZ 是否有健康的 NLB 节点，也看后端 target：某 AZ"至少一个 target group 有健康 target"则该 AZ 的 NLB IP 才进 DNS；>1 个 TG 且任一 UNHEALTHY → FAIL
- C. 只看后端 target，不看 NLB 节点
- D. NLB 跨区开关会直接改变 ETH 判定结果
- E. ETH 对 NLB 永远返回健康

【答案】B
【解析】ETH 对 NLB 两层判断：NLB 侧（cross-zone 关时该 AZ 节点只探同 AZ 目标，至少一个 TG 有健康目标该 AZ IP 才进 DNS）+ target 侧（>1 TG 且有一个 UNHEALTHY 则 FAIL）。NLB 跨区开/关不影响 ETH 判定。

### 26. 单 VPC 内某 service 的 VPC Endpoint 一般是否需要开 ETH？专家口径是？
- A. 必须关闭 ETH，否则会误摘除
- B. 通常不需要（单 VPC 一 service 只能建一个 endpoint，无第二个可切）；且 Alias 到 VPCE 时无论 ETH 设什么，AZ 故障时不健康 IP 都会被自动摘除，建议一律设 True（"要么正确要么无害"）
- C. ETH 对 VPCE 完全无效
- D. 需要额外付费才能对 VPCE 用 ETH
- E. VPCE 必须自建 endpoint HC

【答案】B
【解析】VPCE 通常没有第二个 endpoint 可切；Alias 到 VPCE 时 AZ 故障不健康 IP 会被自动摘除，gavinmc 建议一律设 True。

---

## 第四章 真实故障模式诊断 —— 路由策略 / DNSSEC 运营 / 注册（8 题）

### 27. 同 DC 内，经 NAT 的服务器解析到 us-east-1（预期），带公网 IP 的 resolver 却解析到 us-west-1（意外）。latency-based routing 的判定依据是？
- A. 最终客户端的 IP
- B. 发起解析的 resolver 的源 IP（不同 resolver 公网 IP 来自不同 ISP 池 → 判到不同区）；若 resolver 开了 EDNS0/ECS 则用客户端子网判定
- C. VPC 所在区域
- D. 记录 TTL
- E. 权威服务器最近的 edge

【答案】B
【解析】latency/geo 基于发起解析的 resolver IP，非客户端 IP。开 ECS 时用客户端子网判定；不开则用 resolver IP。注意 VPC Resolver 不支持 ECS。

### 28. 一组 weighted 记录部分权重为 0、部分非 0，都健康。R53 如何选择？
- A. 按比例把 0 权重也纳入
- B. 先只考虑非 0 权重记录；只有当所有非 0 权重记录都不健康时，才考虑 0 权重记录
- C. 0 权重记录永远不返回
- D. 随机返回
- E. 0 权重记录优先返回

【答案】B
【解析】R53 先只考虑非 0 权重记录；仅当所有非 0 权重都不健康才考虑 0 权重记录。可用"权重 100/0"做手动 failover。

### 29. 启用 DNSSEC 签名后，验证型 resolver 却不做验签 / 链中断导致 SERVFAIL。根因是？
- A. 算法选错
- B. 只启用签名不够，必须逐级建立 chain of trust：把每级 KSK 的 DS 注册到直接父区（example.com 的 DS 注册到 .com，子域 DS 注册到 example.com）；缺任一级 DS 传播则验签不发生
- C. ZSK 未轮换
- D. resolver 不支持 EDNS0
- E. 记录 TTL 太长

【答案】B
【解析】启签≠验签生效，必须逐级 DS 注册建信任链。缺任一级 DS 则验签不发生（子区签名也白搭）。R53 Resolver 验签失败返回 SERVFAIL。

### 30. 想在 R53 hosted zone 上禁用 DNSSEC，报 `KeySigningKeyInParentDSRecord 400`。正确处理是？
- A. 重试几次即可
- B. 父区仍有指向该 KSK 的 DS 记录，必须先从父区/注册商移除 DS（先解除信任链）才能安全关签；回滚要考虑 resolver 缓存 TTL，分步进行
- C. 先删除所有 A 记录
- D. 联系 Support 强制关闭
- E. 先轮换 KSK

【答案】B
【解析】关 DNSSEC 前必须先撤父区 DS。Slack 事件教训：回滚未考虑 resolver 缓存 TTL 把小故障放大，应分步进行。

### 31.（多选）关于 R53 Resolver（`.2` / AmazonProvidedDNS）的 DNSSEC 校验语义，下列正确的是？
- A. 只有当 resolver 自己在做递归解析时才做 DNSSEC 校验，转发给其他 resolver 的查询由那个 resolver 校验
- B. `.2` 校验失败返回 SERVFAIL，不放行伪造响应
- C. `.2` 忽略 DO/CD 位、不返回 RRSIG、不置 AD 位 → 客户端侧（如 delv）无法自行校验，要客户端级 DNSSEC 须自建递归 DNS
- D. `.2` 对所有转发区也全链路验签
- E. `.2` 会把 RRSIG 原样返回给客户端

【答案】A、B、C
【解析】Resolver 验签范围 = 它自己递归解析的名字，转发区不覆盖；校验失败 SERVFAIL；`.2` 不返回 DNSSEC 记录/AD 位，客户端验签不支持。D、E 与实际相反。

### 32. 4 个 PHZ 复用同一 KMS CMK，客户困惑为何 DS 记录值各不相同。正确解释是？
- A. 是 bug，应各建独立 CMK
- B. 复用 CMK → KSK 相同（同公钥）；但 DS 用 ECDSA P-256 + SHA-256（algo 13），同一消息多次签名值不同，故不同 HZ 的 DS 值可不同，任取一个上报 RIR 即可；建议别跨 fault domain 复用 CMK
- C. DS 值不同说明 KSK 不同
- D. 必须每个 zone 用不同 CMK 才能启用 DNSSEC
- E. DS 值随机生成与密钥无关

【答案】B
【解析】复用 CMK → KSK 同、DS 值可不同（algo 13 签名不确定性）；每 HZ 仍需各自启签；建议 prod/non-prod 等 fault domain 分开。

### 33. 从其他注册商转入 R53，状态卡在 `serverTransferProhibited`。正确排查方向是？
- A. 该状态由注册商设置，客户自己在 R53 控制台解除
- B. 该状态由注册局(Registry, 如 Verisign/PIR)设置，比 clientTransferProhibited 更难解除；对注册局做 WHOIS 看真实状态，多为账单/法务问题，需原注册商转呈注册局
- C. 直接重新发起转入即可
- D. 提交 Support case 由 AWS 强制转入
- E. 等待 60 天自动解除

【答案】B
【解析】server* 状态由注册局设置，比 client* 难解除。对注册局 WHOIS（.com/.net 查 Verisign，.org 查 PIR）看真实状态；多为账单问题，注册商要转呈注册局。

### 34. 安全审计报"Missing EPP configuration，可能被劫持"。正确澄清是？
- A. 需要立即配置 EPP
- B. EPP 是注册商↔注册局的协议，EPP 状态码只是域名状态、不是发变更的通道；R53 Domains 防未授权变更的真正机制是 IAM 角色/策略
- C. R53 Domains 有独立 EPP 配置项需开启
- D. 必须启用 DNSSEC 才能防劫持
- E. EPP 缺失会导致 DNS 解析失败

【答案】B
【解析】R53 Domains 无独立 EPP"配置"，clientTransferProhibited 有助防转移但可被有权限者移除；访问控制正解是 IAM。

---

## 第五章 底层技术原理 —— 递归 vs 权威 / anycast / 委派选择（7 题）

### 35. Route 53 托管 hosted zone 那一侧扮演什么 DNS 角色？
- A. 递归解析器，会替客户端迭代查询根/TLD
- B. 权威 name server，只对发到它、它负责的 zone 作答，从不迭代、从不缓存别人的数据
- C. stub resolver
- D. 转发解析器
- E. 既是权威又是递归

【答案】B
【解析】R53 托管 zone = 权威（authoritative），只按权威身份作答，收到 RD=1 也不递归。VPC Resolver 才是递归层。这是 SME 最常被问混的分界。

### 36. 客户"改了记录为什么没生效"。机制层最准确的解释与首要排查是？
- A. R53 权威侧改动要几小时才生效
- B. R53 权威侧改动是秒级生效的；卡的是下游递归解析器手里旧记录的剩余 TTL。先问客户在哪测、经过哪个递归解析器、旧记录 TTL 多少
- C. 一定是负缓存
- D. 一定是 DNSSEC 未重签
- E. 需要手工刷新 R53 缓存

【答案】B
【解析】只有递归解析器缓存，权威(R53)不缓存。R53 侧秒级生效，感知延迟取决于下游递归缓存的 TTL 剩余。

### 37.（多选）关于 R53 权威的四组委派 NS（stripe）与 shuffle sharding，下列正确的是？
- A. 每个 hosted zone 的 4 条委派 NS 从 .com/.net/.org/.co.uk 四个 stripe 各取一条
- B. 多 TLD 意义在于 TLD 层容灾（某 TLD 自身故障时还有其他 stripe 可用）
- C. shuffle sharding 强制"任意两个 hosted zone 的 4 条 NS 重叠不超过 2 条"，切碎故障爆炸半径
- D. 四个 stripe 全由 Verisign 运营
- E. 每个 zone 的 4 条 NS 都指向同一个 edge

【答案】A、B、C
【解析】4 条 NS 横跨四个 TLD stripe（各取一条）；多 TLD 为 TLD 层容灾；shuffle sharding 保证任两 zone NS 重叠≤2。D 错——.com/.net 是 Verisign，.org/.co.uk 是不同运营方。E 错——见下题。

### 38. 为什么 R53 选择"每个 edge 一般只宣告一个 stripe"，而非在每个 edge 宣告全部四条 NS？
- A. 节省 anycast 地址
- B. 若每 edge 宣告全部四条，解析器无论选哪条都落到同一最近 edge，该 edge 一旦不可达则四条 NS 全军覆没无处可退；每 edge 一个 stripe 使 4 条 NS 分布在 4 个不同 edge，任一 edge/路径故障仍有 3 条可退 + 获得 Internet 路径多样性
- C. 因为 anycast 不支持多 stripe
- D. 为了降低成本
- E. 因为解析器不支持多 NS

【答案】B
【解析】延迟 vs 可用性取舍：牺牲少许延迟换 edge/路径多样性容灾。代价是某些 NS 从较远 edge 应答，但约 80% 解析器靠 SRTT 自动收敛到最快 NS，对最小 RTT 影响很小。

### 39. 客户抱怨"控制台显示的 4 条 NS 和我 dig NS 拿到的不一致"。最可能原因与正确处置？
- A. R53 出 bug，应重建 zone
- B. 多半是父区(注册商/上级 zone)委派没同步、或存在旧 delegation set 残留；不要动 R53 侧那条自动生成的 NS 记录
- C. 应手工修改 R53 自动生成的 NS 记录使其匹配
- D. DNSSEC 未启用导致
- E. anycast 导致每次 dig 结果本就不同

【答案】B
【解析】NS 不一致多为父区委派未同步或旧 delegation set 残留。"Except in rare circumstances, don't change it"——别动 R53 自动生成的 NS 记录。

### 40. 递归解析路径中，stub→recursive 与 recursive→权威 两段查询的 RD（Recursion Desired）标志分别是？
- A. 都是 RD=1
- B. stub→recursive 带 RD=1（请替我跑全程）；recursive→权威 带 RD=0（权威不递归），R53 收到 RD=1 也不递归、只按权威作答
- C. 都是 RD=0
- D. stub→recursive RD=0，recursive→权威 RD=1
- E. RD 标志由 TTL 决定

【答案】B
【解析】stub→recursive RD=1，recursive→权威 RD=0；R53 权威即使收到 RD=1 也不递归。委派靠 NS 记录 + glue（避免"要解析 NS 又要先解析 NS"死循环），权威应答置 AA=1。

### 41. 若某上游被 `.` rule 指到 Cisco Umbrella，而该上游全部挂掉，R53 Resolver 会怎样？
- A. 自动回落到公网根做递归解析（forward-first）
- B. 不会自动回落到公网根——R53 不提供 BIND 的 forward-first 语义；要回落只能删除该 dot rule（控制面 API 操作）
- C. 返回负缓存
- D. 切到备用上游
- E. 直接返回 8.8.8.8 的结果

【答案】B
【解析】R53 无 BIND forward-first。所有查询转发到某上游，上游全挂不会自动回落公网根。gavinmc 明确"这通常是好事，forward-first 结果不可预期"。要回落只能删 dot rule。（另：Resolver rule 无内建 failover，多 target IP 时逐个试、顺序≈随机。）

---

## 第六章 底层技术原理 —— 缓存/负缓存 / health checker 18% / DNSSEC 信任链（7 题）

### 42. 客户"新建了记录，但一直 NXDOMAIN"。若之前查过这个当时不存在的名字，最可能的机制是？
- A. 正缓存过久
- B. 负缓存：递归解析器缓存了 NXDOMAIN，时长 = min(SOA.MINIMUM, SOA.TTL)（R53 默认 MINIMUM 约 900s），新建后仍要等负缓存过期
- C. DNSSEC 验签失败
- D. R53 权威未生效
- E. anycast 路由错误

【答案】B
【解析】负缓存（RFC 2308/9077）：缓存 NXDOMAIN/NODATA 否定应答，时长取 min(SOA.MINIMUM, SOA.TTL)。"记录明明建了却还 NXDOMAIN"十有八九是负缓存。

### 43. 关于 SOA 记录的 MINIMUM 字段，下列哪个是它在现代 DNS 里的真正用途？
- A. 记录的默认 TTL
- B. 负缓存 TTL 的上限（负缓存时长 = min(SOA.MINIMUM, SOA.TTL)，RFC 9077）
- C. zone 传输间隔
- D. SOA 记录本身的 TTL
- E. DNSSEC 签名有效期

【答案】B
【解析】常见误解是"MINIMUM = 默认记录 TTL"，实际它是负缓存 TTL 上限。RFC 9077 修订：负缓存 TTL 取 min(SOA.MINIMUM, SOA.TTL)，避免 SOA TTL 小而 MINIMUM 大时负缓存过久。

### 44. R53 health checker 判定端点"健康"的全球共识阈值是？其设计意图是？
- A. 50%；简单多数
- B. 可用 checker 中报告健康的比例 > 18% → 判健康，≤18% → 不健康；低阈值意在防止端点仅因被少数检查点的网络路径隔离就被误判为不健康（抗局部隔离误报，偏 fail-toward-available），且不可调
- C. 100%；所有 checker 必须一致
- D. 18%；但客户可在控制台调整
- E. 取决于 HC 类型

【答案】B
【解析】>18% checker 报健康即判健康，抗局部网络隔离误报，不可调（文档明示可能未来变更但用户不能设）。这是"能被多少比例全球检查点看到"的共识，不是端点自身业务健康。

### 45.（多选）关于 R53 health check 的底层机制，下列正确的是？
- A. 各检查点互不协调，即使配 30s 间隔端点也会在某几秒收到多次、又有几秒完全没有，平均约每 2 秒一次
- B. HTTPS 健康检查不校验证书——证书过期/无效不会让 HC 失败
- C. HTTP/HTTPS：4s 内建 TCP 连接 + 连上后 2s 内返回 2xx/3xx；带字符串匹配时需在响应体前 5120 字节内出现指定字符串
- D. TCP 健康检查需 10s 内建立 TCP 连接
- E. 健康结论是同步瞬时传播到所有 DNS 数据面的

【答案】A、B、C、D
【解析】checker 独立全球数据面、互不协调；HTTPS 不校验证书；各类型阈值如题；TCP 10s 建连。E 错——健康结论跨数据面是异步、秒级但非瞬时。

### 46. 客户问"故障发生后为什么没立刻切"。failover 感知总耗时由哪几段串联构成？
- A. 只有 HC 检测一段
- B. ① HC 达到失败阈值（间隔×连续失败，最快约 10s）+ ② 健康结论跨数据面异步传播（秒级非瞬时）+ ③ 递归解析器缓存中旧记录 TTL 过期（取决于 record TTL），三段串联
- C. 只有 TTL 过期一段
- D. HC 检测 + DNSSEC 重签
- E. 控制面 API 传播 + anycast 收敛

【答案】B
【解析】切换总耗时 = 检测(≥10s) + 跨数据面异步传播(秒级) + 递归缓存 TTL 过期。别简化成"HC 一失败就切"。failover 记录 TTL 要设小（如 60s）就是压第三段。

### 47. DNSSEC 验证链四步的正确顺序是？
- A. ZSK 验业务记录 → KSK 验 ZSK → 父区 DS 验 KSK → 根信任锚
- B. 父区 DS 按 Digest Type 哈希子区 KSK 并比对（信任 KSK）→ KSK 验签 DNSKEY RRset（信任其中的 ZSK）→ 信任 ZSK → ZSK 的 RRSIG 验业务记录（全过则置 AD 位）
- C. 根 → ZSK → KSK → DS → 业务记录
- D. DNSKEY 验 DS → DS 验 RRSIG → RRSIG 验记录
- E. KSK 直接签业务记录

【答案】B
【解析】四步：父区 DS→子 KSK（按 Digest Type，2=SHA-256 主流）；KSK 签 DNSKEY RRset→信任 ZSK；ZSK 的 RRSIG 验业务记录。DS 在父区、DNSKEY 在子区，方向别反；DS 只是子 KSK 的摘要指纹不含完整公钥；R53 用 algo 13(ECDSAP256SHA256)。

### 48. 支持验证的递归解析器返回 SERVFAIL，下列哪些是可能断点？（多选）
- A. 换过 KSK 但父区 DS 未更新，DS digest 与子 KSK 对不上
- B. RRSIG 已过期（即使 TTL 未到也失败，多见于手工/外部签名 zone）
- C. 中间盒子裁掉大 DNSSEC 应答或不支持 EDNS0 大报文，DNSKEY 缺失
- D. 父区无 DS（这是 island of trust 半启用态，不验证做普通解析，不 SERVFAIL）
- E. DS Digest Type 填错

【答案】A、B、C、E
【解析】DS 与 KSK 对不上、RRSIG 过期（RRSIG 有效期≠TTL，过期即使 TTL 未到也 SERVFAIL）、DNSKEY 被裁/EDNS0 不支持、Digest Type 填错都会 SERVFAIL。D 是陷阱——父区无 DS 是 island of trust，不验证走普通解析，不 SERVFAIL。

---

*（题库结束，共 48 题）*


================================================================================

# FILE: exam/exam-difficulty-guide.md
<!-- SOURCE FILE: exam/exam-difficulty-guide.md -->

# Route 53 SME 题库难度分层指南 + 干扰项工程

> **用途**：为 route53-sme 题库建立统一的难度分层标准与干扰项设计规范，指出现有题库各层占比与真实 SME 能力画像的差距，并给出可直接复用的 15 道 L3 示范题。
> **配套材料**：`exam-bank-01-07.md`（52 题）、`exam-bank-08-13.md`（46 题）、`exam-bank-advanced-tt-principles.md`（48 题）。
> **适用对象**：出题人、题库维护者、SME 考核设计者。

---

## 第一部分 · 难度三层模型（L1 / L2 / L3）

真实 SME 面对的不是"背知识点"，而是"从一堆真假难辨的现象里判根因、下一步、并把握措辞纪律"。据此把题分三层，每层考的认知动作不同：

### L1 — 记忆 / 事实回取（Recall）
- **认知动作**：直接回取一个孤立事实、定义、配额、阈值、方向。
- **题干特征**：单一知识点，无场景或场景只是包装，问"是什么/多少/哪个方向"。
- **正确项特征**：一个客观事实，与其他选项无需权衡。
- **典型例**：VPC Resolver 地址 = CIDR base+2（5-1）；health checker 18% 阈值（4-3）；DS 在父区 / DNSKEY 在子区（7-2）；API 控制面 5 req/s（12-4）。
- **判据**：SME 候选人只要读过文档就能答对，不需要任何推理或排除。

### L2 — 应用 / 单跳因果（Apply）
- **认知动作**：把一条规则套到一个具体场景，做一次因果映射即可得解。
- **题干特征**：给一个明确场景 + 一个明确现象，问"该怎么配 / 为什么会这样"，因果链只有一跳。
- **正确项特征**：需要"知识点 → 场景"的一次转换，但不需要在多个可行方案间权衡。
- **典型例**：apex 指 ALB 用 A-Alias（1-3）；内部 ALB 用 CloudWatch alarm HC（4-1）；`.` FORWARD rule 全转发（5-5）；weighted 权重 0 何时启用（3-5）。
- **判据**：知道规则就能一步推到答案；干扰项多为"记混的相邻知识点"。

### L3 — 多跳分析 / 诊断纪律（Analyze）
- **认知动作**：现象与根因不在同一层——必须穿过"表层现象 → 排除误导方向 → 定位真实根因层 → 判断必要/充分 → 措辞纪律"这条多跳链。
- **题干特征**：给一个**误导性现象**或**看似合理但错误的初判**，问真实根因、下一步排查、或结论的纪律缺陷。常含"AI 初稿建议…"、"客户第一反应…"、"仅凭 X 能否判定…"。
- **正确项特征**：
  - 现象层与根因层**分离**（配置全对但结果错 → 根因在解析优先级/缓存/下游）。
  - 需要在**多个可行方案**里选最佳，或识别"必要非充分"。
  - 常要求对照**证据分级**判断能不能下结论。
- **典型例**：A-Alias→NLB 配置 100% 正确但 dig 返回旧 IP，根因在 Resolver Forward Rule（2-6）；EC2→.2 timeout，初稿查 SG/NACL 是方向性错误（13-2）；conntrack vs throttling 双指标 NOT OK 看 ENI 迁移时机判主因（13/进阶 13）；一次 dig 成功宣布"整条路径健康"的纪律缺陷（11-8）。
- **判据**：即使背熟所有文档，若不具备"分离现象与根因 + 排除误导 + 必要非充分 + 证据分级"的诊断纪律，仍会答错。这是区分"通过认证"与"能独立扛 case"的分水岭。

### 一句话区分
| 层 | 问的是 | 失分的人缺什么 |
|---|---|---|
| L1 | 事实是什么 | 没读过文档 |
| L2 | 这个场景该用哪条规则 | 规则记混 / 不会套场景 |
| L3 | 这个误导现象背后真正的根因/下一步/能不能下结论 | 缺诊断纪律，被表层现象带偏 |

---

## 第二部分 · 现有题库各层占比与真实 SME 差距

### 现有分层估算

按上述模型逐题归类（含单选/多选/问答），三份题库合计约 146 题的分层占比估算如下：

| 题库 | 总数 | L1 记忆 | L2 应用 | L3 多跳分析 |
|---|---|---|---|---|
| exam-bank-01-07 | 52 | ~22 (42%) | ~22 (42%) | ~8 (16%) |
| exam-bank-08-13 | 46 | ~14 (30%) | ~19 (41%) | ~13 (28%) |
| exam-bank-advanced-tt-principles | 48 | ~10 (21%) | ~20 (42%) | ~18 (38%) |
| **合计** | **146** | **~46 (31%)** | **~61 (42%)** | **~39 (27%)** |

> 说明：这是按"认知动作"而非"题目自称难度"的归类。进阶题库整体更偏 L3，01-07 章基础题偏 L1/L2。归类边界题（如某些多选）就高不就低。

### 真实 SME 能力画像（目标分布）

真实 case 里 SME 的时间几乎不花在"背事实"上——事实可查文档；真正吃经验的是 L3 的诊断纪律。理想考核应向 L3 倾斜：

| 层 | 现状 | 真实 SME 目标 | 差距 |
|---|---|---|---|
| L1 记忆 | 31% | **15–20%** | **超配** ~11–16pp |
| L2 应用 | 42% | 35–40% | 基本匹配 |
| L3 多跳分析 | 27% | **40–50%** | **欠配** ~13–23pp |

### 三条核心差距

1. **L3 欠配、L1 超配**：现有题库近 1/3 是纯记忆题，而真实 SME 的核心价值在 L3 诊断。基础章（01-07）尤其明显——大量"哪个方向/多少阈值"型单点题，应压缩或升级为场景诊断题。

2. **"误导现象 → 真根因"型题偏少且集中**：这是 L3 的主力题型（如 2-6、13-2），但主要集中在进阶题库，基础章几乎没有。真实 case 最常见的正是"配置全对但结果不对"——现有基础题很少训练候选人"越过配置层往解析优先级/缓存/下游找根因"。

3. **"必要非充分 + 证据分级"纪律题稀缺**：13-3（必要非充分）、13-7（四级证据）、11-8（单点成功≠全局健康）是全库仅有的几道纪律题，却是 SME 区别于初级支持的关键。这类题应成体系，而非零星点缀。

### 改进方向
- 把基础章的部分 L1 单点题**升级**为 L3：保留知识点，但用"误导性初判 + 问真根因"的壳重写（示范见第四部分）。
- 为每个 topic 至少配 1–2 道"配置全对但结果错"的 L3 诊断题。
- 建立"纪律题"小专题：必要非充分、证据分级、blast radius 界定、缓存滞后 vs 配置错，横切所有 topic。

---

## 第三部分 · 干扰项工程原则（Distractor Engineering）

干扰项质量直接决定题目区分度。真实 SME 考核的干扰项必须"像真的"——反映候选人真实会犯的错，而非明显荒谬的凑数项。以下是四条硬性原则。

### 原则 1 · 反"最长最全就是对"（Anti–longest-answer bias）
- **问题**：候选人有"选最长/最全/最谨慎那项"的应试直觉。若正确项恒为最详尽的选项，题目考的是应试技巧而非知识。
- **规则**：
  - 正确项长度应与干扰项相当，**不刻意最长**；有时让最短选项为正确项。
  - 把"看起来最全面/最稳妥"的表述做成**干扰项**（如"必须先修完所有可能问题再回复"——过度稳妥恰是错的，见 13-3 选项 C）。
  - "同时做 A 和 B 和 C 更安全"型堆砌选项，往往是要排除的过度设计。

### 原则 2 · 真实可信干扰项（Plausible, mistake-derived distractors）
- **问题**：荒谬干扰项（"删除 hosted zone 会自动注销域名"给没读过的人也能一眼排除）无区分度。
- **规则**：
  - 每个干扰项都应对应一个**真实会犯的错**：AI 初稿的方向性错误（EC2→.2 查 SG/NACL）、记混的相邻阈值（HTTP 4s+2s vs TCP 10s）、混淆的近义机制（conntrack vs throttling、latency 用 client IP vs resolver IP）。
  - 优先取材于真实 case/TT 里客户或初判者实际走错的方向。
  - 干扰项应"半对"——在某个相邻场景下成立，但在本题场景下错（如"HTTPS 健康检查会校验证书"——听起来天经地义，恰是经典陷阱）。

### 原则 3 · 多个可行选最佳（Best-of-plausible）
- **问题**：只有一个"能用"的选项时，题目退化为 L1。
- **规则**：
  - L3 题应让 **2–3 个选项都"技术上可行"**，但只有一个是**最佳/最规范/最安全**。
  - 让候选人在"能做"与"该做"之间权衡：如内部资源监控，"endpoint HC 填私有 IP"技术上像可行（实则不允许），"CloudWatch alarm HC"才是正解；如批量改记录，"逐条调用加 sleep 到每秒 5 次"能跑，"单次打包多个 Changes"才规范。
  - 在解析里必须说明**为什么次优项被否**，而不只说正确项对。

### 原则 4 · 条件不足则不下结论（Insufficient-evidence option）
- **问题**：候选人倾向于总从选项里挑一个"根因"，即使证据不足。
- **规则**：
  - 在合适的题里提供"**信息不足，需先确认 X**/这是必要非充分/不能仅凭此判定"型选项，且让它成为**正确项**。
  - 训练"必要非充分"：找到一个阻断点 ≠ 修完即恢复（可能有第二处问题）。
  - 训练"单点≠全局"：一次 dig 成功 ≠ 整条切换路径健康；一个 checker ≠ 聚合判定。
  - 训练"证据分级措辞"：把"直接下单一根因结论"做成要排除的干扰项，把"这是一处确认阻断，另有 A/B 需确认"做成正解。

### 附：一条 L3 干扰项自检清单
出一道 L3 题后，逐条核对：
- [ ] 正确项**不是**最长/最全的那个？
- [ ] 每个干扰项都能对应一个真实会犯的错，而非凑数？
- [ ] 至少有 2 个选项"技术上可行"，靠"最佳"而非"唯一可行"区分？
- [ ] 若场景证据不足，是否提供并奖励"不下结论/必要非充分"选项？
- [ ] 解析是否既说明正确项为何对，也说明**每个次优/陷阱项为何错**？

---

## 第四部分 · 15 道 L3 示范题（带解析）

> 以下 15 题严格按第一部分 L3 判据与第三部分四原则设计：现象与根因分离、含真实可信干扰、多个可行选最佳、并在合适处奖励"不下结论"。可直接并入题库或作为重写模板。每题标注考点来源。

### L3-1（单选）· 现象与根因分离
客户的 apex 记录 `app.example.com` 配成 A (Alias) 指向 NLB，工程师反复核对：记录类型对、Alias target 对、ELB canonical hosted zone id 对、ETH 已开。但 VPC 内 `dig` 始终返回一个陌生的旧 IP。下一步最应该做什么？

- A. 把 A-Alias 改成 CNAME 重试
- B. 重建该 hosted zone
- C. 检查该 VPC 的 Resolver 规则与关联 PHZ——查询很可能被 Forward Rule 或同名 PHZ 在到达公网 zone 之前就截走了
- D. 提高 NLB 的健康检查频率
- E. 联系 Support 报 Alias 解析 bug

【答案】C

【解析】配置层（记录/Alias/ETH）已被逐项证明正确，"结果不对"的根因必然在**更上游的解析优先级层**：VPC 内 Forward Rule > PHZ > 公网，查询可能根本没到达你改的那个公网 zone。这是"配置全对但结果错"的标志性 L3 场景。A/B（改类型、重建）是在已排除的配置层里空转；D 与解析结果无关；E 把自己的解析路径问题误报成服务 bug。（来源：2-6 / 5-3）

### L3-2（单选）· 反最全项 + 必要非充分
排查中你已 DIRECTLY_OBSERVED 一条 NACL 拦截了到目标 DNS 的流量。客户催问"修这条 NACL 是不是就恢复了？"最符合 SME 纪律的回复是？

- A. 是，找到阻断点即可承诺修完恢复
- B. 先把所有可能的问题（NACL、路由、SG、目标 NS、TTL）全部修一遍再回复，确保万无一失
- C. 这是一处必须修复的阻断，但它是恢复的必要非充分条件；请同时确认目标 NS 可达、转发路由无第二处阻断，我不承诺"修完即恢复"
- D. 判定为限流，建议提额
- E. 只要是直接观测到的阻断，就等于唯一阻断，可以承诺

【答案】C

【解析】必要非充分是 L3 纪律核心：多 ENI/多路径下常存在第二处问题，DIRECTLY_OBSERVED 只证明"这处确实阻断"，不证明"唯一阻断"（E 错）。A 过度自信；B 是"最全最稳妥"陷阱项（原则 1）——实际上盲修一切既拖延又可能引入新问题，不是 SME 做法；D 是无依据改方向。正解在"能做"之上追求"该怎么措辞"。（来源：13-3）

### L3-3（单选）· 误导性初判
EC2 执行 `nslookup jcrew.com 10.201.244.2` 三次全 timeout（VPC CIDR 10.201.244.0/22）。一份 AI 初稿建议"检查 EC2 到 10.201.244.2 的 Security Group / NACL / 路由表"。作为 SME 你的纠正是？

- A. 初稿方向正确，按建议查 SG/NACL/路由
- B. 10.201.244.2 = CIDR base+2 = AmazonProvidedDNS，EC2→.2 这段流量不经过 SG/NACL/路由；真正阻断点在下游（Outbound Endpoint 的 ENI 子网 NACL、转发目标路由/可达性）
- C. timeout 说明记录不存在，应检查记录配置
- D. 先重启 EC2 再看
- E. 直接判定限流并提额

【答案】B

【解析】方向性错误是 L3 高频陷阱：EC2→AmazonProvidedDNS(.2) 由底层实现承载，不过 SG/NACL/路由表，所以往这里查是空耗（A 错）。timeout 是**网络层无响应信号**，不是"记录没配"（C 把 timeout 错读成 NXDOMAIN 类数据问题）。真正应指向下游 Resolver Outbound Endpoint 的 ENI 子网 NACL 与转发目标可达性。（来源：13-2 / 5-2）

### L3-4（单选）· 近义机制混淆 + 多个可行选最佳
一个出站 Resolver Endpoint 的某 ENI 远未到 10K QPS 就大量丢包超时，并发布了 `conntrack_allowance_exceeded_delta` 指标。最可能根因与修复是？

- A. 超过 10K QPS 硬限，应加 ENI
- B. 该 ENI 挂了限制性安全组、或查询经过 NLB，触发 connection tracking，把 UDP QPS 压到约 1,500；修复是去掉限制性 SG / 不走 NLB
- C. 目标 on-prem NS 不可达，检查转发目标
- D. DNSSEC 验签占带宽
- E. IMDS 偷走了额度

【答案】B

【解析】区分 conntrack 与 throttling 是 L3 近义机制题：`conntrack_allowance_exceeded_delta` 明确指向连接跟踪被打满（限制性 SG 或 NLB 强制 conntrack，QPS 降约 6 倍），而非 QPS 超限（那会发布 `udp_throttled_count`，对应 A 的修复）。C 是"限流指标都正常时"才优先怀疑的方向，本题指标已明确指向 conntrack，故 C 是相邻场景的错位。加 ENI（A）修错了病。（来源：进阶 12 / 进阶 14）

### L3-5（单选）· 双阳性指标的判别纪律
内部 runbook 判定某出站 endpoint 的 `conntrack_allowance_exceeded_delta` 与 `udp_throttled_count` **两个指标都 NOT OK**。如何判定主因，而不是盲目两个都修？

- A. 一定是 conntrack，去掉 SG 限制
- B. 一定是 throttling，加 ENI
- C. 看 ENI 的 capacity_level 迁移时机：迁移到大实例前就已在大实例上 → 主因 throttling；迁移过程中才升到 CONN_TRACK_FLEXI_FLEET → 主因 conntrack
- D. 两个指标不可能同时 NOT OK，数据有误
- E. 无法判定，直接加 ENI 兜底

【答案】C

【解析】双阳性时不能各修一半或猜一个（A/B 都是过早收敛），要用 capacity_level 迁移时机做判别——这是"多信号冲突时如何定主因"的 L3 分析。D 否认了真实会发生的双中招；E 放弃分析直接堆资源，是初级做法。正解要求候选人知道两指标背后的容量迁移机制。（来源：进阶 13）

### L3-6（单选）· 限流指标正常时的下一跳
出站解析大面积超时/SERVFAIL，但 conntrack 与 throttling 指标**都正常**。下一步最应怀疑什么、怎么定位？

- A. 客户 SG 配错，抓 SG 规则
- B. 目标 on-prem NS 无响应/不可达；用 CloudWatch Insights 按分钟算 timeout%，再对 Athena 表按 destip 聚合 `respcode='TIMEOUT'` 统计每个目标 NS 的超时数
- C. DNSSEC 信任链断，检查 DS
- D. PHZ 重叠，列关联 PHZ
- E. VPC DNS 属性未开，检查 enableDnsHostnames

【答案】B

【解析】"限流指标正常"这个条件把 conntrack/throttling 排除后，L3 要求候选人跳到**下一个最可能层**——目标 NS 不可达（出站 endpoint 支持工单最常见根因），并给出可执行的按 destip 聚合定位法。C/D/E 都是没有本题现象支撑的其他方向，属"不看条件套模板"。正解体现"排除法 + 可执行下一步"。（来源：进阶 14）

### L3-7（单选）· NLB fail-open 的隐蔽根因
某区 NLB 后端 target 100% 不健康，但 Route 53 仍把流量发往该区、跨区 failover 不触发。ETH 已开。根因是？

- A. ETH 没生效，重开一次
- B. NLB fail-open——目标全部不健康时 NLB 仍放行流量，于是 ETH 从 NLB 得到的信号是"健康"，跨区 failover 不触发；应改用能反映真实后端的 HC（应用级/CW 复合指标）配 failover routing
- C. health check 间隔太长
- D. 记录 TTL 过短
- E. DNSSEC 阻断了切换

【答案】B

【解析】这是"信号源本身撒谎"的 L3 陷阱：ETH 依赖 LB 上报，而 NLB fail-open 在后端全挂时仍上报可用，直接骗过 ETH。表面看 ETH/HC 都"正常工作"，根因在 NLB 的 fail-open 语义与 ETH 的信任链错配。A 把问题当成配置没生效；C/D/E 都在无关层。（来源：进阶 21）

### L3-8（单选）· 半启用态的正确判读
一个 zone 已 signing（有 DNSKEY、记录带 RRSIG），但父区 `.com` 里查不到对应 DS。一份初判写"DNSSEC 配置错误，必须立刻修复否则宕机"。正确判读是？

- A. 初判正确，立即抢修
- B. 这是 island of trust（信任孤岛）：有签名、无信任链，支持 DNSSEC 的解析器不会验证、按普通 DNS 正常返回，**不影响可用性**；常见于刚启签未建 DS，或禁用过程先删 DS 未关签
- C. zone 已不可解析，会返回 SERVFAIL
- D. 必须先撤 DNSKEY
- E. 说明 DNSSEC 从未启用

【答案】B

【解析】L3 要求区分"半启用态"与"故障态"：有签名无 DS = island of trust，解析仍正常，不是宕机风险（A/C 把半启用态误判为故障，是最典型的过度反应错误）。E 与"已有 DNSKEY/RRSIG"矛盾。正解奖励"先判断这个中间态到底影不影响可用性"的冷静分析。（来源：7-3 / 进阶 30 相关）

### L3-9（单选）· SERVFAIL 的快速二分
客户 `dig pd-market.com @<recursive>` 得 SERVFAIL，第一反应怀疑 DNSSEC 验证失败。用哪个命令能一步区分"是不是 DNSSEC 验证问题"，且如何判读结果？

- A. `dig pd-market.com +short`：有输出即正常
- B. `dig pd-market.com @<recursive> +cd`：加 `+cd`（关闭验证）后变 NOERROR → 原 SERVFAIL 是 DNSSEC 验证失败；仍 SERVFAIL → 与 DNSSEC 无关（另找根因，如 stale NS 委派）
- C. `dig DNSKEY pd-market.com`：查得到就没问题
- D. `dig +trace`：能跑完就排除 DNSSEC
- E. `nslookup pd-market.com`

【答案】B

【解析】L3 诊断纪律：先用一个**判别性实验**把假设二分，而不是直接认领"DNSSEC 失败"这个第一反应。`+cd` 是关键——它精准隔离验证这一环。A/C/D/E 都不能干净地把 DNSSEC 验证从其他 SERVFAIL 成因里分离出来。正解奖励"用最小实验证伪假设"而非顺着直觉下结论。（来源：7-5）

### L3-10（单选）· ECS 缺失导致的 latency 误路由
同 DC 内，经 NAT 出去的服务器解析到 us-east-1（预期），而带独立公网 IP 的另一台 resolver 却解析到 us-west-1（意外）。latency-based routing 的判定依据、以及为何两台结果不同？

- A. 依据最终客户端 IP；两台客户端不同故不同
- B. 依据发起解析的 **resolver 源 IP**（不同 resolver 的公网 IP 来自不同 ISP 池，被判到不同区）；若 resolver 开了 EDNS0/ECS 才用客户端子网判定——注意 VPC Resolver 不支持 ECS
- C. 依据 VPC 所在区域
- D. 依据记录 TTL
- E. 依据权威服务器最近的 edge

【答案】B

【解析】L3 要穿过"我以为按客户端定位"的直觉，落到"按 resolver 源 IP"这个真实机制，并解释两台差异来自 resolver 出口 IP 的 ISP 归属不同。A 是最常见误解（把 client 当判定源）；正解还需带出 ECS 的边界条件（VPC Resolver 不支持）。区分度就在候选人是否知道判定源到底是谁。（来源：进阶 27）

### L3-11（单选）· 多资源同挂的公共因子取证
客户三个域名"昨天正常、今早全挂"。以下哪个是首要归因方向与标准取证序列？

- A. 逐条记录单独排查，先看第一个域名的某条 A 记录
- B. 多资源同时失效 → 先找公共因子（同账户 / 同 hosted zone / 注册商 NS 被改）；标准序列 whois → `dig <domain> NS` → 控制台查 Hosted Zone/NS → CloudTrail 查 24–48h 变更事件（ChangeResourceRecordSets / DeleteHostedZone / UpdateDomainNameservers）
- C. 直接判定为 DDoS
- D. 建议客户重新注册三个域名
- E. 先启用 DNSSEC 再观察

【答案】B

【解析】blast radius 界定是 L3 的起手式：三域名**同时**失效指向三者共有的一环，而非某条单独记录（A 是把大范围故障当单点排，方向就错了）。正解给出完整的"注册/委派链"取证序列。C/D/E 都是没界定范围就跳结论。（来源：13-5）

### L3-12（单选）· 缓存滞后 vs 配置错
客户改完记录后说"还是不通"。SME 应先做什么？

- A. 立即判定记录改错并回滚
- B. 先问"等了多久？用哪个递归解析器测的？旧记录 TTL 多少？"，再直查权威 `dig @<权威NS>` 确认改动已生效——若权威成功但递归失败，是缓存/负缓存/TTL 滞后，不是配置错
- C. 直接重建整个 hosted zone
- D. 判定为限流并提额
- E. 关闭 DNSSEC 观察

【答案】B

【解析】L3 要求先区分"权威侧是否已生效"与"递归侧缓存是否滞后"：R53 权威改动秒级生效，感知延迟来自下游递归缓存的剩余 TTL 或负缓存（NXDOMAIN 被缓存 min(SOA.MINIMUM, SOA.TTL)）。A/C 在没验证权威侧前就动配置，可能把对的改错。正解的关键是"直查权威"这个隔离动作。（来源：13-6 / 进阶 36 / 进阶 42）

### L3-13（单选）· 场景归类必须以第一手信息为准
一个子域被接管，报告时点 `dig` 显示无权威 zone、SERVFAIL。有人据此直接归为 Scenario 5（不防护）。客户随后澄清该 child zone 曾存在、后被删除。正确的归类与纪律教训是？

- A. dig 的 SERVFAIL 永远意味着 Scenario 5，维持原判
- B. 应翻案为 Scenario 1（曾建后删 → 父域留委派），且教训是：Scenario 归类必须以父域/删除历史的第一手信息为准，不能只凭报告时点的 dig 签名反推
- C. 只要看到 SERVFAIL 就可对客下 defect 定性
- D. 客户澄清不可信，以 dig 为准
- E. Scenario 1 和 5 防护上等价，归类无所谓

【答案】B

【解析】L3 纪律：报告时点的 dig 只反映"此刻无权威 zone"，无法区分"从未建（S5）"还是"曾建后删（S1）"——而这个区分直接决定"设计上是否本应防护"。必须以删除/委派历史第一手信息定性（Thales case 教训）。A/C 凭表层签名硬判；D 否定客户第一手信息；E 无视归类后果。（来源：08-6 / 08-1）

### L3-14（单选）· 批量变更的限流规避（多个可行选最佳）
需要一次性修改一个 hosted zone 里的上百条记录。以下哪种做法最规范？

- A. 写脚本循环调用 change-resource-record-sets，每条一次，每秒发 >5 次
- B. 逐条调用但加 sleep 精确卡到每秒恰好 5 次
- C. 用**单次** ChangeResourceRecordSets 请求打包多个 Changes，避免触发 `Throttling` / `PriorRequestNotComplete`
- D. 先提额把 hosted zone 记录上限调到 10 万
- E. 分多个账号并发调用绕过限流

【答案】C

【解析】多个"看似可行"里选最佳的 L3 题：A 秒级触发限流（明显错）；B 技术上"能跑"且卡在 5 req/s 边界内，是最有迷惑性的次优项——但它仍是逐条串行、慢且脆（一次 PriorRequestNotComplete 就打乱节奏），不是规范做法；D 答非所问（上限不是限流）；E 是绕过而非解决。正解是控制面本就支持的单请求打包多变更。解析必须说明 B 为何被否。（来源：12-5）

### L3-15（单选）· 单点成功≠全局健康
DR 演练中，团队通过 ARC 的一个 data plane endpoint 把 rc-secondary 切为 On，并观察到一次 `dig` 成功返回 us-west-2 端点，就宣布"整条切换路径完全健康"。这个结论的纪律缺陷是？

- A. 没有缺陷，一次成功即可结论
- B. 单点成功≠全局健康：① cluster 有 5 个 data plane endpoint，只验证了一个可达，未证明灾难时其余可用（生产脚本应轮询 5 个直到成功）；② 一次 dig 只反映当前解析，未证明各种故障组合下 failover 都会正确切换；③ 还应验证 safety rule 在效（如试全关被拒）。宜说"已验证经该 endpoint 可切换"，不下"整条路径完全健康"的充分性结论
- C. 缺陷在于没同时关掉 primary
- D. 缺陷在于 dig 应该用 +trace
- E. 缺陷在于没启用 DNSSEC

【答案】B

【解析】"必要非充分"在切换路径上的延伸：验证了一个 endpoint/一次解析，不能推广到"整条路径在所有灾难组合下都健康"。L3 奖励候选人识别**充分性缺口**并给出更严谨的措辞与补充验证（轮询 5 endpoint、验 safety rule）。C/D/E 都是无关的操作细节，没击中"单点 vs 全局"的纪律要害。（来源：11-8 / 11-2 / 11-3）

---

## 附 · 15 题答案速查

| 题号 | 答案 | 考点 | 题号 | 答案 | 考点 |
|---|---|---|---|---|---|
| L3-1 | C | 现象/根因分离 | L3-9 | B | 最小实验二分 |
| L3-2 | C | 必要非充分 | L3-10 | B | ECS/resolver 判定源 |
| L3-3 | B | 误导性初判 | L3-11 | B | blast radius 取证 |
| L3-4 | B | conntrack vs QPS | L3-12 | B | 缓存滞后 vs 配置错 |
| L3-5 | C | 双阳性指标判别 | L3-13 | B | 第一手信息归类 |
| L3-6 | B | 排除后下一跳 | L3-14 | C | 批量变更选最佳 |
| L3-7 | B | NLB fail-open | L3-15 | B | 单点≠全局 |
| L3-8 | B | island of trust | | | |

> 指南结束。三层模型 + 现状差距 + 四条干扰项原则 + 15 道 L3 示范题，可直接用于题库升级与出题评审。


================================================================================

# FILE: exam/mock-exam-scored.md
<!-- SOURCE FILE: exam/mock-exam-scored.md -->

# Route 53 SME 计分模拟卷（Mock Exam · Scored）

> 65 题 · 覆盖 13 个知识域 · 按考试比重加权（路由策略 / 健康检查 / Resolver / Alias 加重）
> 题源：exam-bank-01-07.md（52 题）+ exam-bank-08-13.md（46 题），已 Codex 校正。

---

## 考试须知（Instructions）

- **建议用时**：90 分钟
- **总分**：100 分
- **及格线**：≥ 70 分（Pass）
- **题量**：65 题
- **题型与分值**：
  - 单选题 51 题，每题 **1.4 分**（合计 71.4 分）
  - 多选题 11 题，每题 **2.0 分**（合计 22.0 分，**多选须选全且不多选才得分，不倒扣、无部分分**）
  - 问答题 3 题，每题 **2.2 分**（合计 6.6 分，按要点覆盖度给分）
  - 合计 71.4 + 22.0 + 6.6 = **100 分**
- **域分布（按考试比重加权）**：

  | 域 | 主题 | 题数 |
  |---|---|---|
  | 01 | Hosted Zones | 4 |
  | 02 | 记录与 Alias ⭐ | 6 |
  | 03 | 路由策略 ⭐ | 8 |
  | 04 | 健康检查 ⭐ | 7 |
  | 05 | Resolver 与混合 DNS ⭐ | 8 |
  | 06 | DNS Firewall / Global Resolver | 5 |
  | 07 | DNSSEC | 5 |
  | 08 | 子域接管 / dangling delegation | 4 |
  | 09 | 域名注册 / 生命周期 | 4 |
  | 10 | Route 53 Profiles | 3 |
  | 11 | ARC Routing Controls | 4 |
  | 12 | 服务集成 + 配额限流 | 4 |
  | 13 | 通用故障排查纪律 | 3 |
  | | **合计** | **65** |

  ⭐ = 考试重点加权域。

> 答题时把每题答案填入**第二部分答题卡**；交卷后对照**第三部分**评分。

---

# 第一部分 · 题卷（Questions）

> 只含题干与选项，不含答案。多选题已在题干标注"多选"。

---

## 域 01 · Hosted Zones（4 题）

### Q1（单选，1.4 分）
客户在同一账号下同时持有 public zone `habitat-energy.au`（父）与 `prod.habitat-energy.au`（子），父 zone 里有一条指向子 zone 的 NS 委派记录。客户想把子域记录合并进父 zone、随后删除子 zone，要求切换零停机。最安全的操作顺序是？

- A. 先删父 zone 里的 NS 委派记录，再在父 zone 建全子域记录
- B. 先在父 zone 建全所有子域记录并验证，再删父 zone 里的 NS 委派记录，最后删子 zone
- C. 直接删除子 zone，Route 53 会自动把记录并入父 zone
- D. 同时删 NS 委派记录和子 zone，再补建父 zone 记录
- E. 把子 zone 的 NS 记录复制到父 zone 即可

### Q2（多选，2.0 分，选两项）
关于 Private Hosted Zone（PHZ）的启用与解析边界，正确的有哪些？

- A. 使用 PHZ 要求关联 VPC 的 `enableDnsHostnames` 与 `enableDnsSupport` 都为 true
- B. PHZ 记录可被公共递归 DNS（如 8.8.8.8）解析
- C. VPC 若用 DHCP Option Set 指向自定义 DNS，PHZ 记录会返回 NXDOMAIN
- D. PHZ 支持 DNSSEC signing
- E. PHZ 支持 IP-based 路由策略

### Q3（单选，1.4 分）
一个 VPC 内查询 `seattle.accounting.example.com`，该 VPC 同时关联了 `accounting.example.com` 和 `example.com` 两个 PHZ。Route 53 会用哪个 PHZ 应答？

- A. `example.com`（更早创建的）
- B. `accounting.example.com`（最具体匹配）
- C. 随机选一个
- D. 两个都查，合并结果
- E. 返回 NXDOMAIN，因为命名空间重叠冲突

### Q4（单选，1.4 分）
客户删除一个已启用 DNSSEC signing 的 hosted zone 时报 `HostedZoneNotEmpty`。最可能的原因与正确处理是？

- A. zone 内还有普通记录，删掉即可
- B. 需先 deactivate/delete 该 zone 的 KSK
- C. 该 zone 被 RAM 共享，需先取消共享
- D. NS/SOA 记录不能删，导致 zone 非空
- E. 需先关闭 VPC 关联

---

## 域 02 · 记录类型与 Alias ⭐（6 题）

### Q5（单选，1.4 分）
客户裸域 `example.com`（zone apex）要指向一个 ALB。应如何配置？

- A. 建一条 apex 的 CNAME 指向 ALB DNS 名
- B. 建一条 apex 的 A (Alias) 指向 ALB
- C. 建一条 apex 的 NS 记录指向 ALB
- D. 建一条 apex 的 TXT 记录写入 ALB 名
- E. apex 无法指向 ALB，必须换用 www 子域

### Q6（单选，1.4 分）
客户要把子域 `www.example.com` 指向一个**第三方（非 AWS 托管）**的域名 `cdn.partner.net`。应使用哪种记录？

- A. A-Alias
- B. AAAA-Alias
- C. CNAME
- D. A 记录直接填 partner 域名
- E. Alias 指向同 zone 记录

### Q7（多选，2.0 分，选三项）
关于 Alias 相对 CNAME 的特性，正确的有哪些？

- A. Alias 可以建在 zone apex，CNAME 不行
- B. Alias 指向 AWS 资源的查询免费
- C. Alias 指向 AWS 资源时可以单独设置 TTL
- D. Alias 支持 Evaluate Target Health，CNAME 不支持
- E. Alias 的目标可以再是另一条 Alias 或 CNAME

### Q8（单选，1.4 分）
客户做 failover，主备记录都指向 ELB（可建 Alias 的资源）。以下哪种做法最规范？

- A. 给每条 alias 记录另外建一个独立 endpoint 健康检查
- B. alias 记录设 Evaluate Target Health = Yes，不再单独建健康检查
- C. 把 alias 改成 CNAME 再挂健康检查
- D. 用 simple 路由 + 健康检查
- E. 关闭健康检查，靠 TTL 过期切换

### Q9（单选，1.4 分）
客户用 simple 路由在一条记录里配了 12 个 A 值，观察到每次 dig 只返回一部分且顺序变化。关于这一行为，正确的解释是？

- A. 配置错误，simple 只能配 1 个值
- B. Simple 的一条 RRset 会返回**全部**值并随机排序；每次 dig 只看到一部分是 UDP 应答体积 / 客户端截断所致，不是 Route 53 的"8 值上限"
- C. 健康检查剔除了不健康的值
- D. TTL 过期导致部分值丢失
- E. Route 53 在做真正的负载均衡

### Q10（单选，1.4 分）
客户把 apex 子域配成 A (Alias) 指向 Kong NLB，配置本身 100% 正确，但 VPC 内 dig 却返回旧的 Apigee ELB IP。最可能的根因层次是？

- A. Alias 记录类型选错了
- B. apex 不该用 Alias
- C. VPC 内解析被 Resolver Forward Rule 拦截，查询没到达公网 zone
- D. NLB 未开启健康检查
- E. ELB canonical hosted zone id 填错

---

## 域 03 · 路由策略 ⭐（8 题）

### Q11（单选，1.4 分）
客户在多个 AWS Region 部署了相同应用，希望终端用户被路由到"网络延迟最低"的 Region。应选哪种路由策略？

- A. Weighted
- B. Latency
- C. Geolocation
- D. Geoproximity
- E. Simple

### Q12（单选，1.4 分）
客户用 geolocation 路由，为 CN、US 各配了记录，但没有配 default。一个来自无法映射到任何地理位置的 IP 的查询会得到什么？

- A. 随机返回 CN 或 US 记录
- B. 返回离得最近的记录
- C. "no answer"
- D. NXDOMAIN
- E. 返回全部记录

### Q13（多选，2.0 分，选两项）
关于 geolocation 与 geoproximity 的区别，正确的有哪些？

- A. geolocation 只看用户位置；geoproximity 看资源+用户位置
- B. geoproximity 可用 bias（±1..±99）扩张/收缩某资源的吸引范围
- C. geolocation 也能用 bias 调整吸引范围
- D. geoproximity 不需要 Traffic Flow
- E. geolocation 需要 Traffic Flow

### Q14（单选，1.4 分）
一条 failover 记录，Primary 和 Secondary 的健康检查同时都是 unhealthy。Route 53 会返回什么？

- A. 返回 Secondary
- B. 返回 Primary（fail open）
- C. 返回 "no answer"
- D. NXDOMAIN
- E. 随机返回一个

### Q15（单选，1.4 分）
客户在一组 weighted 记录里把某条记录的 weight 设为 0，其余记录权重非 0。什么情况下这条 0 权重记录会被返回？

- A. 永远不会被返回
- B. 每次都会被返回一小部分
- C. 仅当组内所有非 0 权重记录都不健康时
- D. 当它自身的健康检查通过时
- E. 当组内权重之和为 0 时

### Q16（单选，1.4 分）
客户在一个 Private Hosted Zone 里想按客户端源 IP 段做 IP-based 路由。会发生什么？

- A. 正常生效
- B. PHZ 不支持 IP-based 路由策略
- C. 需要先开 Traffic Flow
- D. 需要 outbound endpoint
- E. 仅 IPv6 支持

### Q17（多选，2.0 分，选两项）
关于 EDNS0 / edns-client-subnet (ECS) 对地理 / 延迟类路由准确性的影响，选出**最直接描述定位机制**的两项？

- A. resolver 支持 ECS 时，Route 53 能用用户 IP 的截断前缀更准确定位
- B. VPC 的 .2 resolver 支持 EDNS0 但不支持 ECS
- C. PHZ 使用 EDNS0 来做路由决策
- D. 不支持 ECS 时，用 resolver 自身源 IP 近似定位，可能偏差
- E. ECS 会加密 DNS 查询内容

### Q18（单选，1.4 分）
Route 53 的 latency 路由所依据的延迟数据，下列描述正确的是？

- A. 是客户端到资源的实时 ping 延迟
- B. 是 AWS 长期实测的网络延迟，非实时、会漂移，目标须在 AWS Region
- C. 按客户端地理国家静态映射
- D. 由客户在控制台手工填写延迟值
- E. 等同于 geoproximity 的 bias 计算

---

## 域 04 · 健康检查 ⭐（7 题）

### Q19（单选，1.4 分）
客户要监控一个**内部 ALB**（私有、不可路由 IP）的健康。哪种健康检查类型最合适？

- A. Endpoint HTTP 检查，直接填内部 ALB 私有 IP
- B. Endpoint TCP 检查内部 IP
- C. CloudWatch alarm-based（或 Calculated）健康检查
- D. Simple 路由自带的健康检查
- E. 无法监控内部资源

### Q20（单选，1.4 分）
客户的 HTTPS endpoint 健康检查突然变红，同时发现站点证书刚过期。证书过期是否是健康检查失败的原因？

- A. 是，HTTPS 健康检查会校验证书
- B. 否，HTTPS 健康检查不校验证书，证书过期不会导致失败
- C. 是，但只在开启 string matching 时
- D. 否，但会触发 INSUFFICIENT_DATA
- E. 是，需要更新根 CA

### Q21（单选，1.4 分）
关于 Route 53 健康检查的聚合判定（多地 checker），正确的是？

- A. 需超过 50% 的 checker 报健康才判健康
- B. 需超过 18% 的 checker 报健康才判健康
- C. 全部 checker 都健康才判健康
- D. 任一 checker 健康即判健康
- E. 由客户配置阈值百分比

### Q22（多选，2.0 分，选三项）
关于健康检查的响应时间与匹配阈值，正确的有哪些？

- A. HTTP/HTTPS：4s 内建 TCP 连接 + 连接后 2s 内回 2xx/3xx
- B. TCP：10s 内建连
- C. string matching 的匹配串必须落在 body 的前 5,120 字节内
- D. HTTP 需 10s 内回 2xx
- E. TCP 需 2s 内建连

### Q23（单选，1.4 分）
一个 CloudWatch alarm-based 健康检查，其 alarm 进入 `INSUFFICIENT_DATA` 状态。健康检查会如何判定？

- A. 一律判健康
- B. 按 HC 的 `InsufficientDataHealthState` 配置（默认 Unhealthy）
- C. 一律判不健康，不可改
- D. 保持上次状态，不可改
- E. 触发 fail open

### Q24（单选，1.4 分）
一个 CloudWatch alarm-based HC 关联的 alarm 已恢复 OK，但 HC 长时间（数十分钟）仍停留 Unhealthy。作为紧急恢复手段，哪种操作能触发 HC 重新评估？

- A. 删除并重建 hosted zone
- B. 对该 HC 执行 `UpdateHealthCheck`（如改一个无害字段）
- C. 重启关联的 EC2 实例
- D. 删除 CloudWatch alarm
- E. 只能等待，无法干预

### Q25（单选，1.4 分）
受害账户在 ALB access log 里发现持续的探测，User-Agent 为 `Amazon-Route53-Health-Check-Service (ref 28e27f8c-...)`，但自己并未创建该健康检查。这属于什么问题、如何处理？

- A. 正常流量，无需处理
- B. Unwanted HC abuse——由 Support Ops 走标准流程（确认 HC → 联系 offending 账户 → 等 7 天 → Mechanic 禁用，需 2PR）
- C. DDoS 攻击，走 Shield 流程
- D. 直接在自己账户里删除该 HC
- E. 修改 SG 屏蔽即可，无需上报

---

## 域 05 · Resolver 与混合 DNS ⭐（8 题）

### Q26（单选，1.4 分）
VPC 的主 CIDR 是 `10.201.244.0/22`。VPC Resolver（.2 resolver）的地址是？

- A. 10.201.244.1
- B. 10.201.244.2
- C. 10.201.247.253
- D. 169.254.169.253
- E. 10.201.244.253

### Q27（单选，1.4 分）
EC2 无法用 `nslookup ... 10.x.x.2`（VPC DNS）解析。工程师第一反应去检查 EC2 到 VPC DNS 之间的 Security Group / NACL / 路由表。这个排查方向对吗？

- A. 对，SG/NACL 常常拦住到 .2 的流量
- B. 不对，EC2 到 VPC DNS(.2) 的流量不经过 SG/NACL/路由表
- C. 对，但只需检查 NACL
- D. 不对，应改用公共 DNS
- E. 对，需要放行 UDP 53

### Q28（单选，1.4 分）
某域名在 VPC 内既有一个 PHZ 记录、又有一条 Resolver Forward Rule 指向企业 DNS。VPC 内查询该域名时会怎样？

- A. 用 PHZ 记录应答
- B. 查询被 Forward Rule 转发到企业 DNS（Forward Rule 优先于 PHZ）
- C. 报冲突错误
- D. 随机选一个
- E. 先查 PHZ，未命中再转发

### Q29（单选，1.4 分）
关于 Inbound / Outbound Resolver Endpoint 的方向，正确的是？

- A. Inbound = AWS→on-prem 查询；Outbound = on-prem→AWS 查询
- B. Inbound = on-prem→AWS 查询；Outbound = AWS→on-prem 查询
- C. 两者方向相同，只是冗余
- D. Inbound 用于公网，Outbound 用于私网
- E. Outbound endpoint 使用公网 IP

### Q30（单选，1.4 分）
客户想把 VPC 内**所有**域名查询都转发到企业 DNS，让 VPC 内查询永远不到达公网 Route 53。应如何配置 Resolver Rule？

- A. 为每个域名各建一条 FORWARD 规则
- B. 建一条域名为 `.`（dot）的 FORWARD 规则，关联 outbound endpoint
- C. 删除所有 System Rule
- D. 建一条 SYSTEM 规则指向企业 DNS
- E. 关闭 VPC 的 enableDnsSupport

### Q31（多选，2.0 分，选两项）
关于 Resolver 的 QPS / 吞吐边界，正确的有哪些？

- A. 每个 Outbound Endpoint 最多 6 个 ENI，聚合可达约 60,000 请求/秒
- B. 每个 ENI 约 10K QPS，但经 NLB 或 SG 因 connection tracking 会降到约 1.5–1.7K QPS
- C. 每 ENI 固定 1024 QPS 不可调
- D. Resolver 是全球服务，非 Regional
- E. Rule 不关联 outbound endpoint 也能生效

### Q32（单选，1.4 分）
Outbound Endpoint 有两个 ENI，分属不同子网/NACL。工程师排查发现其中一个 ENI 子网的 NACL 阻断了 DNS。为让转发可靠，NACL 上应双向放行哪些？

- A. 只放行入站 TCP 53
- B. 出站 UDP/TCP 53 + 入站 UDP/TCP ephemeral(1024-65535)，且每个 ENI 子网都要一致
- C. 只放行出站 UDP 53
- D. 放行所有 ICMP
- E. 只在一个 ENI 子网放行即可

### Q33（问答，2.2 分）
客户跨账号用 AWS RAM 共享一条 Resolver Forward Rule。关于共享范围、outbound endpoint 与成员账号权限，请写出三点关键结论。

---

## 域 06 · DNS Firewall / Global Resolver（5 题）

### Q34（单选，1.4 分）
DNS Firewall 的核心用途和过滤维度是？

- A. 加密所有 DNS 流量
- B. 对出站 DNS 查询做域名层过滤，主要防 DNS 数据渗漏（exfiltration）
- C. 按 IP/端口做网络层过滤
- D. 校验 DNSSEC 签名
- E. 对入站 DNS 查询做应用层过滤

### Q35（单选，1.4 分）
一个 VPC 关联了多个 rule group，rule group 内又有多条 rule。它们的处理顺序是？

- A. priority 数字越大越先处理
- B. priority 数字越小越先处理（rule group 与 rule 都是）
- C. 按创建时间倒序
- D. 随机
- E. 只处理 priority 最大的一条

### Q36（多选，2.0 分，选两项）
关于 DNS Firewall 规则的动作（Action），正确的有哪些？

- A. 含 domain list 的规则可选 ALLOW / BLOCK / ALERT
- B. 不含 domain list 的规则（Advanced 保护）只能 BLOCK / ALERT
- C. 任何规则都能设 ALLOW
- D. BLOCK 只能返回 NXDOMAIN
- E. Advanced 保护可以设 ALLOW

### Q37（单选，1.4 分）
客户用 allowlist（只放行白名单）方式，把入口域名 `svc.example.com` 加入了 ALLOW 列表，但它的 CNAME target 指向未列入白名单的 `backend.other.net`，结果查询被 BLOCK。根因是？

- A. ALLOW 规则失效
- B. 重定向链上的后续域名未显式加入 domain list（默认检查整条 CNAME 链，trust 仅在单次查询事务内有效）
- C. domain list 不支持 CNAME
- D. 白名单必须用通配符
- E. AWS 托管列表拦截了它

### Q38（单选，1.4 分）
客户配了一条 ALERT 规则和一条 DGA Advanced 规则，想确认某次查询是否命中了 DGA 检测。仅凭 dig 返回 NXDOMAIN 可以判定吗？应看哪里？

- A. 可以，NXDOMAIN 就代表 DGA 命中
- B. 不能，必须查 OCSF 日志的 `firewall_rule_id`；ALERT 命中流量的 `action_name` 仍是 `Allowed`
- C. 可以，看 dig 的 flags 即可
- D. 不能，只能看 CloudTrail
- E. 可以，看响应的 TTL

---

## 域 07 · DNSSEC（5 题）

### Q39（单选，1.4 分）
客户要禁用某 public zone 的 DNSSEC signing，直接在 Route 53 点禁用时报 `Please remove DS records in the parent zone first`。正确的禁用顺序是？

- A. 直接强制禁用 signing，再删 DS
- B. 先删父区/注册商的 DS 记录、等传播完成，再关闭 signing
- C. 先删子区 DNSKEY，再关 signing
- D. 先删 KSK，再删 DS
- E. 同时删 DS 和 DNSKEY

### Q40（单选，1.4 分）
关于 DS 记录与 DNSKEY 的放置位置，正确的是？

- A. DS 在子区，DNSKEY 在父区
- B. DS 在父区（子区 KSK 对应 DNSKEY 的摘要 + 算法元数据，不含完整公钥），DNSKEY 在子区
- C. 两者都在子区
- D. 两者都在父区
- E. DS 在根区，DNSKEY 在 TLD

### Q41（单选，1.4 分）
一个 zone 已经 signing（有 DNSKEY、记录带 RRSIG），但父区 `.com` 里查不到对应 DS 记录。这是什么状态、对解析有何影响？

- A. 信任链完整，解析器会验证
- B. island of trust（信任孤岛）：有签名无信任链，解析器不验证、按普通 DNS 正常返回，不影响可用性
- C. zone 不可解析，返回 SERVFAIL
- D. 配置错误，必须立刻修复否则宕机
- E. DNSSEC 未启用

### Q42（单选，1.4 分）
客户启用 DNSSEC 时报 `<key ARN> could not be used by Route 53 DNSSEC`。KSK 绑定的 KMS CMK 必须满足哪些要求？

- A. 对称密钥、任意规格即可
- B. 位于 us-east-1、asymmetric（非对称）、ECC_NIST_P256，且 key policy 授权 Route 53 DNSSEC 服务
- C. RSA_2048、对称用途
- D. 只要在 us-east-1 即可
- E. 必须是多区域密钥

### Q43（单选，1.4 分）
客户 `dig pd-market.com @<recursive>` 得到 SERVFAIL，第一反应怀疑 DNSSEC 验证失败。用什么命令能快速区分"是不是 DNSSEC 验证问题"？

- A. `dig pd-market.com +short`
- B. `dig pd-market.com @<recursive> +cd`——变 NOERROR 则是 DNSSEC 验证失败；仍 SERVFAIL 则与 DNSSEC 无关
- C. `dig DNSKEY pd-market.com`
- D. `dig +trace`
- E. `nslookup pd-market.com`

---

## 域 08 · 子域接管 / dangling delegation（4 题）

### Q44（单选，1.4 分）
某客户发现子域 `sub.example.com` 被他人接管。调查发现：父域仍保留该子域的 4 个 awsdns NS 委派记录，但对应的 child hosted zone 早已被删除。这属于哪个 Scenario，Route 53 的 StopZoneSniping 是否**设计上**会防护？

- A. Scenario 1，会防护
- B. Scenario 5，会防护
- C. Scenario 1，不防护
- D. Scenario 5，不防护
- E. Scenario 3，会防护

### Q45（单选，1.4 分）
关于 subdomain takeover 的成功条件，下列哪项**最准确**？

- A. 攻击者必须让新建 zone 的全部 4 个 NS 与悬空 NS 完全重叠
- B. 攻击者只需让新建 zone 的 NS 组与悬空 NS 组重叠 ≥1 个即可，重叠越多接管越稳定
- C. 攻击者必须在删除该 zone 的同一账号内重建
- D. 只要父域存在任意 NS 记录即可接管，无需重叠
- E. 必须先拿到 EPP auth code 才能接管

### Q46（单选，1.4 分）
退役一个子域时，消除悬空委派的**正确删除顺序**是？

- A. 先删 child hosted zone，再删父域的 NS 委派记录
- B. 先删父域的 NS 委派记录 → 等 TTL 过期 → 再删 child hosted zone
- C. 同时删除父域 NS 记录和 child zone
- D. 只删 child zone，父域委派保留以便日后复用
- E. 先启用 DNSSEC，再任意顺序删除

### Q47（多选，2.0 分，选三项）
关于 StopZoneSniping 保护特性，以下哪些描述正确？

- A. 保护是跨账号全局的，阻止任何账号被分配到重叠 NS
- B. 保护仅在删除该 zone 的账号内部生效
- C. 只要新 zone 重叠 1 个或多个 NS 即被阻止（per-NS）
- D. hold 是永久的，删过 zone 的 NS 组永远被锁定
- E. hold 靠"父域仍被观测到委派"续命，委派从父域消失后会被 purge、NS 回池

---

## 域 09 · 域名注册 / 生命周期（4 题）

### Q48（单选，1.4 分）
客户 `gateio.jp` 反映：Route 53 控制台显示到期 2027/6/29，而 WHOIS（whois.jprs.jp）显示 2026/06/30，且状态码显示 "-"，怀疑异常。SME 的正确判读是？

- A. WHOIS 到期日权威，客户域名即将过期需立即续期
- B. 两个到期日都对——R53/Gandi 的 2027/6/29 是有效到期日，JPRS WHOIS 按"到期月月末"呈现旧年份且更新滞后；状态 "-" 因 .jp 不支持 transfer lock，均属正常
- C. 状态 "-" 说明域名被锁定，需解锁
- D. 需立即向 JPRS 提工单纠正 WHOIS bug
- E. R53 到期日错误，应以 JPRS 为准修正

### Q49（多选，2.0 分，选三项）
关于 Route 53 的"注册商侧"与"DNS 托管侧"的边界，正确的有哪些？

- A. 删除 hosted zone 会自动注销域名注册
- B. 注销域名注册不会自动删除 hosted zone
- C. 域名被 suspend（clientHold）时，hosted zone 里的记录仍在，但公网查不到，因为注册局停止委派
- D. 改了 hosted zone 的 NS 后，必须在 Registered domains 侧同步更新注册局 NS 委派，否则解析走旧 NS
- E. 注册（registration）与解析（resolution）是同一套生命周期

### Q50（单选，1.4 分）
客户在域名转移**完成前**关闭了源账号，三个 Amazon Registrar 域名被 suspend 进入 clientHold、公网 DNS 中断。要恢复并完成转移，正确路径是？

- A. 直接从目标账号发起转移即可
- B. 重开源账号 → AES 事件驱动自动 unsuspend → 再走标准跨账号转移；转移必须由源账号发起
- C. 联系 JPRS 解除 clientHold
- D. 等域名进入 redemption 后从池中重新注册
- E. 修改目标账号的 NS 委派即可恢复解析

### Q51（单选，1.4 分）
关于赎回期（Redemption）与删除，下列哪项正确？

- A. 域名过期后随时可免费找回
- B. 进入 redemption 后 restore 通常收费且不保证成功；进入 pending delete 后不可赎回
- C. redemption 期内域名仍正常解析
- D. pending delete 阶段仍可免费赎回
- E. released 阶段域名自动回到原账号

---

## 域 10 · Route 53 Profiles（3 题）

### Q52（单选，1.4 分）
关于 Route 53 Profile 的本质，下列哪项**最准确**？

- A. Profile 是一种全新的 DNS 资源类型，用来取代 PHZ
- B. Profile 是把 PHZ、Resolver rules、DNS Firewall rule groups、Resolver query logging 配置成组打包、批量关联到多个 VPC 的分发机制
- C. Profile 只能打包 PHZ，不含其它资源
- D. 用了 Profile 之后就不再需要创建 PHZ
- E. Profile 是一种 Resolver endpoint 的高可用版本

### Q53（单选，1.4 分）
某 VPC 本地直接关联了一个 PHZ，同时又通过 Profile 下发了一个同命名空间的 PHZ。查询该命名空间时，Route 53 采用哪个？

- A. Profile 下发的 PHZ 优先
- B. VPC 本地（local）关联的 PHZ 优先于 Profile 下发的同名配置
- C. 随机选择
- D. 报冲突错误，两者都不生效
- E. 合并两者的记录一起返回

### Q54（多选，2.0 分，选四项）
以下哪些资源可以被打包进一个 Route 53 Profile？

- A. Private Hosted Zones（PHZ）
- B. Resolver rules（转发/系统规则）
- C. DNS Firewall rule groups（含优先级与 fail-open/closed 行为）
- D. Resolver query logging 配置
- E. EC2 安全组

---

## 域 11 · ARC Routing Controls（4 题）

### Q55（单选，1.4 分）
ARC Routing Control 与普通 Route 53 Health Check 的**根本区别**是？

- A. Routing control 主动探测端点，探测失败自动切流
- B. Routing control 是人工/编程拨动的确定性开关，背后改一个 R53 health check 的状态来切流，不依赖对应用的探测
- C. 两者完全等价，只是 ARC 更贵
- D. Routing control 只能用于单 Region
- E. 普通 health check 无法用于 failover 记录

### Q56（单选，1.4 分）
灾难发生时，切换 routing control 状态应使用哪条路径？

- A. Control plane（route53-recovery-control-config）
- B. Cluster 的 data plane endpoint（route53-recovery-cluster，5 个 Region endpoint，轮询直到成功）
- C. 直接改 Route 53 记录
- D. 修改 hosted zone 的 NS 委派
- E. 通过 Service Quotas 控制台

### Q57（单选，1.4 分）
ARC Cluster 的高可用设计是？

- A. 单 Region 部署
- B. 由 5 个 AWS Region 组成，读写 routing control 状态需至少 3 个（3/5 quorum）一致，即使 2 个 Region 不可用仍可切换
- C. 由 3 个 Region 组成，需全部在线
- D. 由 5 个 Region 组成，需 5 个全在线才能切
- E. 由 2 个可用区组成

### Q58（多选，2.0 分，选三项）
关于 Safety Rules，正确的有哪些？

- A. 核心目的是防止一次误操作把所有 Region 同时关掉（防"全关"）
- B. Assertion rule 约束一组 routing control 必须满足某条件才允许改变（如至少 1 个为 On）
- C. Gating rule 用一个"门"control 允许/禁止对另一组 control 的更改
- D. Safety rule 会主动探测应用健康
- E. 有了 safety rule 就不再需要 data plane endpoint

---

## 域 12 · 服务集成 + 配额与限流（4 题）

### Q59（单选，1.4 分）
客户问"为什么在根域 `example.com` 上配 CNAME 报错"，SME 的正确回答是？

- A. 根域 CNAME 需要额外付费才能启用
- B. 标准 DNS 禁止在 zone apex 放 CNAME，应改用 Alias 记录（Alias 可用于 apex）
- C. 根域只能用 TXT 记录
- D. 需要先启用 DNSSEC
- E. CNAME 在 apex 需要设置 TTL=0

### Q60（单选，1.4 分）
下列关于 Route 53 数据面 vs 控制面限流的说法，哪项正确？

- A. 公网权威 DNS 查询有严格的每账号 QPS 配额
- B. 公网权威查询几乎无限、不计账号配额；被限流的是控制面 API（约 5 请求/秒/账号）和 Resolver endpoint（per-ENI ~10,000 QPS）
- C. 控制面 API 无任何限流
- D. Resolver endpoint 无 QPS 上限
- E. 所有查询都计入统一的账号级 QPS 限流

### Q61（单选，1.4 分）
需要一次性修改一个 hosted zone 里的上百条记录，正确做法是？

- A. 写脚本循环调用 change-resource-record-sets，每条一次，每秒 >5 次
- B. 用单次 ChangeResourceRecordSets 请求打包多个变更，避免触发 Throttling/PriorRequestNotComplete
- C. 先提额把 hosted zone 上限调到 10 万
- D. 分账号并发调用绕过限流
- E. 逐条调用但加 sleep 到每秒恰好 5 次

### Q62（多选，2.0 分，选三项）
关于 `ip-ranges.json` 中 Route 53 相关 service 的三分层（呼应 NBC 案例），对应正确的有哪些？

- A. ROUTE53 = 公网权威 NS 的 IP 范围
- B. ROUTE53_RESOLVER = VPC 内递归解析器（有 per-ENI QPS 限流的那层）
- C. ROUTE53_HEALTHCHECKS = 健康检查探测车队
- D. ROUTE53 = VPC 内递归解析器
- E. 三者对应同一条数据路径

---

## 域 13 · 通用故障排查纪律（3 题）

### Q63（单选，1.4 分）
一台 EC2 解析某域名失败。判断故障范围时，SME 首先应做的是？

- A. 立即断定 Route 53 服务整体故障
- B. 先界定 blast radius——是所有 client 还是这一台？所有域名还是这一个？所有查询还是偶发？再归因
- C. 直接建议客户提额
- D. 假定是记录被删并重建记录
- E. 直接对客下单一根因结论

### Q64（多选，2.0 分，选四项）
关于 DNS 响应码/无响应的排查方向，正确的对应有哪些？

- A. timeout（无响应）→ 网络层：SG/NACL/路由阻断、目标 DNS 宕、UDP 分片丢失
- B. NXDOMAIN → 权威明确回答"此名不存在"，是数据问题（记录缺失/拼写/未创建），不是网络问题
- C. SERVFAIL → 递归/权威处理失败：DNSSEC 验证失败、转发目标不响应、上游超时
- D. REFUSED → 服务器拒绝应答：权限/ACL、非授权区、递归被关
- E. NXDOMAIN → 一定是网络阻断导致

### Q65（问答，2.2 分）
简述 SME 排查中的"证据分级（四级）"及其在回复措辞上的纪律。

---

> 题卷结束。请把答案填入下方**第二部分答题卡**。

---

# 第二部分 · 答题卡（Answer Sheet）

> 单选填一个字母（A–E）；多选填多个字母（如 `A,C`）；问答简述要点。

| 题号 | 类型 | 你的答案 | 题号 | 类型 | 你的答案 |
|---|---|---|---|---|---|
| Q1 | 单选 | ______ | Q34 | 单选 | ______ |
| Q2 | 多选(2) | ______ | Q35 | 单选 | ______ |
| Q3 | 单选 | ______ | Q36 | 多选(2) | ______ |
| Q4 | 单选 | ______ | Q37 | 单选 | ______ |
| Q5 | 单选 | ______ | Q38 | 单选 | ______ |
| Q6 | 单选 | ______ | Q39 | 单选 | ______ |
| Q7 | 多选(3) | ______ | Q40 | 单选 | ______ |
| Q8 | 单选 | ______ | Q41 | 单选 | ______ |
| Q9 | 单选 | ______ | Q42 | 单选 | ______ |
| Q10 | 单选 | ______ | Q43 | 单选 | ______ |
| Q11 | 单选 | ______ | Q44 | 单选 | ______ |
| Q12 | 单选 | ______ | Q45 | 单选 | ______ |
| Q13 | 多选(2) | ______ | Q46 | 单选 | ______ |
| Q14 | 单选 | ______ | Q47 | 多选(3) | ______ |
| Q15 | 单选 | ______ | Q48 | 单选 | ______ |
| Q16 | 单选 | ______ | Q49 | 多选(3) | ______ |
| Q17 | 多选(2) | ______ | Q50 | 单选 | ______ |
| Q18 | 单选 | ______ | Q51 | 单选 | ______ |
| Q19 | 单选 | ______ | Q52 | 单选 | ______ |
| Q20 | 单选 | ______ | Q53 | 单选 | ______ |
| Q21 | 单选 | ______ | Q54 | 多选(4) | ______ |
| Q22 | 多选(3) | ______ | Q55 | 单选 | ______ |
| Q23 | 单选 | ______ | Q56 | 单选 | ______ |
| Q24 | 单选 | ______ | Q57 | 单选 | ______ |
| Q25 | 单选 | ______ | Q58 | 多选(3) | ______ |
| Q26 | 单选 | ______ | Q59 | 单选 | ______ |
| Q27 | 单选 | ______ | Q60 | 单选 | ______ |
| Q28 | 单选 | ______ | Q61 | 单选 | ______ |
| Q29 | 单选 | ______ | Q62 | 多选(3) | ______ |
| Q30 | 单选 | ______ | Q63 | 单选 | ______ |
| Q31 | 多选(2) | ______ | Q64 | 多选(4) | ______ |
| Q32 | 单选 | ______ | Q65 | 问答 | ______ |
| Q33 | 问答 | ______ | | | |

---

# 第三部分 · 答案 + 解析 + 评分（Answer Key）

> 逐题：答案 / 分值 / 解析。多选须选全且不多选才得该题满分（无部分分、不倒扣）。

---

## 域 01 · Hosted Zones

**Q1（1.4 分）答案：B**
只要父 zone 的 NS 委派记录还在，查询就始终引向子 zone；先建全记录再删委派可无缝迁移。A/D 顺序颠倒会造成解析失败；C 合并非自动；E 复制委派 NS 不改变委派优先。

**Q2（2.0 分）答案：A、C**
PHZ 只能经 VPC Resolver 解析，公共 DNS 解析不到（B 错）；DHCP Option Set 指向自定义 DNS 绕过 VPC Resolver 导致 NXDOMAIN（C 对）；PHZ 不支持 DNSSEC（D 错）、不支持 IP-based（E 错）；启用前提两个 DNS 属性都为 true（A 对）。

**Q3（1.4 分）答案：B**
重叠命名空间取最长/最具体匹配，`accounting.example.com` 胜过 `example.com`，非按创建时间或随机。

**Q4（1.4 分）答案：B**
带 KSK 的 zone 删除报 `HostedZoneNotEmpty`，需先停用/删除 KSK 再删 zone。NS/SOA 是自带记录，不计入"非空"。

## 域 02 · 记录类型与 Alias ⭐

**Q5（1.4 分）答案：B**
apex 已强制带 NS+SOA，CNAME 不能与同名其他类型共存，故 apex 不能用 CNAME；指向 AWS 资源用 A-Alias，免费、少一跳、可开 ETH。

**Q6（1.4 分）答案：C**
Alias 目标只能是选定 AWS 资源或同 zone 同类型记录，指不了外部/非 R53 托管域名；要指向任意外部 DNS 名只能用 CNAME。

**Q7（2.0 分）答案：A、B、D**
Alias 能建在 apex（A）、指 AWS 资源查询免费（B）、有 ETH（D）；Alias 指 AWS 资源不能设 TTL（C 错）；target 不能再是 Alias/CNAME（E 错）。

**Q8（1.4 分）答案：B**
指向可建 alias 的 AWS 资源做 failover，用 ETH=Yes 让 Alias 自动感知目标健康，不再单独建 HC。CNAME 无 ETH（C 错），simple 不能关联 HC（D 错）。

**Q9（1.4 分）答案：B**
Simple 一条 RRset 可含多值，权威 NS 返回全部值并随机排序——粗粒度分散非负载均衡（E 错），不涉及健康检查。"每次最多 8 值"是 Multivalue Answer 的特性，不是 Simple；simple 看到只回一部分通常是 UDP 应答大小/客户端截断。

**Q10（1.4 分）答案：C**
记录/Alias 配置层正确，"解析结果不对"的根因在解析优先级层：VPC 内 Forward Rule 优先于 PHZ 优先于公网，查询被转发到企业 DNS（返回旧记录），没到达公网 zone。

## 域 03 · 路由策略 ⭐

**Q11（1.4 分）答案：B**
Latency 按 AWS 长期实测网络延迟导向延迟最低 Region，目标须在 AWS Region。Weighted 是按比例分（A 错）；geo 类按地理而非网络延迟。

**Q12（1.4 分）答案：C**
geolocation 对未映射来源，若无 default 记录（`CountryCode:"*"`）返回 "no answer"，不会兜底挑一个。必须显式建 default。

**Q13（2.0 分）答案：A、B**
geolocation 仅按用户位置、可设 default；geoproximity 看资源+用户位置、用 bias 移边界、通常需 Traffic Flow。bias 是 geoproximity 独有（C 错）。

**Q14（1.4 分）答案：B**
failover 的 fail-open：Primary+Secondary 都不健康时返回 Primary，不可配置。

**Q15（1.4 分）答案：C**
单条 weight 0 = 停止该记录流量；仅当组内所有非 0 权重记录都不健康时才启用 0 权重记录。"永远不返回"是错的。

**Q16（1.4 分）答案：B**
PHZ 只支持 simple/failover/multivalue/weighted/latency/geolocation/geoproximity；IP-based 只能在 public zone。

**Q17（2.0 分）答案：A、D**
ECS 让权威 NS 拿到用户 IP 截断前缀，定位更准（A）；不支持时退回 resolver 源 IP 近似（D）。（B 也是正确知识点，但本题取最直接描述定位机制的 A、D；C/E 错。）

**Q18（1.4 分）答案：B**
latency 依据 AWS 长期实测网络延迟，非实时、会漂移，目标须在 AWS Region；不是实时 ping、非静态国家映射、非手工填写。

## 域 04 · 健康检查 ⭐

**Q19（1.4 分）答案：C**
不能对 private/不可路由 IP 建 endpoint HC（A/B 排除）；内部 ALB 用 CloudWatch alarm-based 或 calculated HC。simple 路由不能关联 HC（D 错）。

**Q20（1.4 分）答案：B**
HTTPS 健康检查不校验证书，证书过期/无效不会导致失败——经典陷阱。变红原因另找（超时、状态码非 2xx/3xx、string 未匹配）。

**Q21（1.4 分）答案：B**
聚合规则：> 18% 的 checker 报健康即判健康，≤ 18% 判不健康。不是多数决（A 错）。

**Q22（2.0 分）答案：A、B、C**
HTTP/HTTPS 4s 建连 + 2s 回 2xx/3xx（A）；TCP 10s 建连（B）；string match 须在 body 前 5120 字节内（C）。D/E 把阈值记混。

**Q23（1.4 分）答案：B**
INSUFFICIENT_DATA 走 `InsufficientDataHealthState`，三态 Unhealthy（默认）/ Healthy / LastKnownStatus。

**Q24（1.4 分）答案：B**
`UpdateHealthCheck` 触发 HC 重新评估，可作紧急恢复手段。正常 R53 对 alarm 状态变化应在 1–2 分钟内响应，数十分钟属异常。

**Q25（1.4 分）答案：B**
任意账户能建指向不属于自己 IP 的 endpoint HC 造成滋扰。User-Agent 与 `ref=<HC ID>` 是识别关键。处理是 Support Ops 专属流程；受害方无法删别人账户里的 HC（D 错）。

## 域 05 · Resolver 与混合 DNS ⭐

**Q26（1.4 分）答案：B**
VPC Resolver 接入地址 = VPC 主 CIDR 基地址 +2 = 10.201.244.2。169.254.169.253 是 link-local 别名，但按 "VPC+2" 算法答案是 10.201.244.2。

**Q27（1.4 分）答案：B**
EC2 → VPC DNS(.2) 的流量不经过 SG/NACL/路由表——最大的方向性坑。真正阻断点在 Outbound Endpoint 的 ENI 子网 NACL。

**Q28（1.4 分）答案：B**
VPC 内解析优先级：Forward Rule > PHZ > System Rule > 公网递归。同域名冲突时 Forward Rule 胜，查询被转发出去。

**Q29（1.4 分）答案：B**
In = 别人进来查我（on-prem→AWS）；Out = 我出去查别人（AWS→on-prem）。Outbound endpoint 是私有 IP，出公网需 NAT（E 错）。

**Q30（1.4 分）答案：B**
域名为 `.`（dot）的 FORWARD 规则覆盖除 PHZ/AWS 内部名以外的所有域名，把 VPC 内查询全部转出。Rule 必须关联 outbound endpoint 才生效。

**Q31（2.0 分）答案：A、B**
每 Outbound Endpoint ≤6 ENI、聚合 ~60K req/s（A）；每 ENI ~10K QPS，经 NLB/SG 因 connection tracking 降到 ~1.5–1.7K（B）。1024 是实例侧 link-local PPS 限制（C 混淆）；Resolver 是 Regional（D 错）；Rule 必须关联 endpoint（E 错）。

**Q32（1.4 分）答案：B**
每个 ENI 子网都要在 NACL 双向放行：出站 UDP/TCP 53 + 入站 UDP/TCP ephemeral，且多 ENI 跨子网配置须一致，否则间歇故障。

**Q33（问答，2.2 分）答案要点（三点全中给满分，缺一点按比例）**
(1) 共享限**同一 Region**，无需额外 VPC peering；
(2) 父账号的 outbound endpoint 会随 rule 一起共享，成员账号**不需另建 endpoint**；
(3) 成员账号只能**使用**共享 rule，不能修改或删除它。
（常见错误：以为跨账号共享 rule 还要单独建 outbound endpoint。）

## 域 06 · DNS Firewall / Global Resolver

**Q34（1.4 分）答案：B**
对经 VPC Resolver 的出站 DNS 查询做域名字符串层过滤，解析成 IP 之前拦截命中黑名单的查询，主防 exfiltration；不加密、不看 IP/端口/应用层。

**Q35（1.4 分）答案：B**
rule group 之间与组内每条 rule 都按 priority 数字从小到大处理（lowest numeric priority first）。

**Q36（2.0 分）答案：A、B**
含 domain list 才能 ALLOW（A）；Advanced 只能 BLOCK/ALERT，不能 ALLOW（B 对，C/E 错）。BLOCK 响应有 NXDOMAIN/NODATA/OVERRIDE 三种（D 错）。

**Q37（1.4 分）答案：B**
默认检查整条重定向链，被 ALLOW 域名的 CNAME target 若未列入 domain list，其 A+AAAA 相当于未被 ALLOW 命中而被 BLOCK。allowlist 要覆盖整条 CNAME 链或配 Trust Redirection Domains。

**Q38（1.4 分）答案：B**
ALERT 命中流量被放行，OCSF 日志里 `action_name` 仍是 `Allowed`；DGA 是否命中不能凭 dig 的 NXDOMAIN 判断，必须查日志 `firewall_rule_id`。

## 域 07 · DNSSEC

**Q39（1.4 分）答案：B**
禁用必须先解除信任链：先删父区 DS → 等传播 → 再关 signing。顺序不能颠倒。

**Q40（1.4 分）答案：B**
DS（父区）= 子区 KSK 对应 DNSKEY 的摘要 + 算法元数据，不含完整公钥；DNSKEY（KSK 257 / ZSK 256）在子区。方向别反。

**Q41（1.4 分）答案：B**
zone 自己 signing 但父区无 DS = island of trust，验证型解析器不验证、解析仍正常、不影响可用性。常见于刚启用未建 DS 或禁用过程已删 DS 未关 signing。

**Q42（1.4 分）答案：B**
KSK 的 CMK 必须位于 us-east-1、asymmetric、ECC_NIST_P256（SIGN_VERIFY），且 key policy 授权 `dnssec-route53.amazonaws.com`。D 错在"只要在 us-east-1"是必要非充分。

**Q43（1.4 分）答案：B**
`+cd`（Checking Disabled）关闭解析器 DNSSEC 验证：加后变 NOERROR = DNSSEC 验证失败；仍 SERVFAIL 则与 DNSSEC 无关。

## 域 08 · 子域接管 / dangling delegation

**Q44（1.4 分）答案：A**
"child zone 曾存在→被删→父域仍留委派" 是 Scenario 1，是 5 个场景中 Route 53 **唯一**提供防护的场景。题问"设计上是否防护"，与"这次实际是否被拦"两回事。

**Q45（1.4 分）答案：B**
接管只需命中 ≥1 个重叠 NS，重叠越多越稳定；可跨账号进行，与 EPP code 无关。"必须整组重叠"是错误认知。

**Q46（1.4 分）答案：B**
先删父域 NS 记录并等 TTL 过期（清空 resolver 缓存），再删 child zone。反过来正是制造 Scenario 1 悬空的错误操作。

**Q47（2.0 分）答案：A、C、E**
保护跨账号全局（A，B 错）；per-NS 生效，重叠任一即拦（C）；hold 非永久（D 错），靠父域仍被观测到委派续命，委派消失则 purge 回池（E）。

## 域 09 · 域名注册 / 生命周期

**Q48（1.4 分）答案：B**
.jp 三特性：WHOIS 到期日恒为到期月月末、旧到期月过完才更新新年份、不支持 transfer lock 状态显示 "-"。有效期以 Gandi/Route 53 为准，保持 auto-renew。

**Q49（2.0 分）答案：B、C、D**
注册 ≠ 解析（E 错）。删 hosted zone 不注销域名（A 错）；注销域名不删 hosted zone（B 对）；suspend/clientHold 断的是注册局委派、记录仍在（C 对）；改 hosted zone NS 必须两侧同步（D 对）。

**Q50（1.4 分）答案：B**
账号一关自助转移被切断（转移必须由源账号发起）。恢复：重开源账号 → AES 事件驱动自动 unsuspend → 标准跨账号转移。这些是 Amazon Registrar 域名，与 JPRS 无关。

**Q51（1.4 分）答案：B**
赎回期 restore 收费且不保证成功；pending delete 后不可赎回（D 错）；released 后回公开池需重新抢注（E 错）；redemption 阶段已暂停、不正常解析（C 错）；"过期随时免费找回"错（A）。

## 域 10 · Route 53 Profiles

**Q52（1.4 分）答案：B**
Profile 是打包分发机制，不是新资源类型、不取代 PHZ；能装 PHZ、Resolver rules、DNS Firewall rule groups、Resolver query logging 四类。

**Q53（1.4 分）答案：B**
优先级铁律：**local > Profile**。本地就近覆盖，Profile 集中默认。

**Q54（2.0 分）答案：A、B、C、D**
Profile 可打包四类 DNS 配置：PHZ、Resolver rules、DNS Firewall rule groups、Resolver query logging。EC2 安全组不在其列（E 错）。

## 域 11 · ARC Routing Controls

**Q55（1.4 分）答案：B**
ARC routing control 是"开关"不是"探针"——人/自动化确定性拨动，背后驱动 health check 状态翻转切 failover 记录，规避灰色故障下探测不准。

**Q56（1.4 分）答案：B**
平时用 control plane 配置；救灾切流必须用 cluster 的 5 个 data plane endpoint（轮询直到成功）。control plane 在大区域灾难中可能不可达。

**Q57（1.4 分）答案：B**
Cluster 跨 5 个 Region，3/5 quorum；即使 2 个 Region 挂掉仍可靠切换。需全部在线（C、D）错。

**Q58（2.0 分）答案：A、B、C**
Safety rule 是护栏，防"全关"与防误触（A、B、C）。它不探测应用（D 错）；切换仍必须走 data plane endpoint（E 错）。

## 域 12 · 服务集成 + 配额与限流

**Q59（1.4 分）答案：B**
标准 DNS 禁止 apex 放 CNAME；Route 53 Alias 无此限制，可用于 apex，指向 AWS 资源不额外收费、AWS 自动维护目标 IP。

**Q60（1.4 分）答案：B**
公网权威查询由 anycast 车队承载、不计账号配额；能被限流的是控制面 API（~5 req/s）与 Resolver endpoint（per-ENI ~10k QPS）。

**Q61（1.4 分）答案：B**
控制面 ~5 req/s，循环单条调用会秒级触发 `Throttling`/`PriorRequestNotComplete`。正解是用单次 ChangeResourceRecordSets 打包多条 Changes。

**Q62（2.0 分）答案：A、B、C**
三分层对应三条不同数据路径：ROUTE53=权威 NS、ROUTE53_RESOLVER=递归（有 QPS 限流）、ROUTE53_HEALTHCHECKS=探测。D、E 混淆路径。

## 域 13 · 通用故障排查纪律

**Q63（1.4 分）答案：B**
单点失败 ≠ 服务故障，一次成功 ≠ 路径整体健康。永远先界定 blast radius 再归因：确定性全失败指向公共环节，概率性失败指向多路径中某一条。

**Q64（2.0 分）答案：A、B、C、D**
timeout=网络层（A）、NXDOMAIN=数据问题（B，E 错）、SERVFAIL=处理失败（C）、REFUSED=拒绝应答（D）。把 timeout 当"记录没配"、把 NXDOMAIN 当网络问题都是走错方向。

**Q65（问答，2.2 分）答案要点（四级 + 措辞纪律，全中给满分）**
四级证据强度：
- **DIRECTLY_OBSERVED**：从客户真实导出/日志/工具输出直接读到 → 措辞"已确认/observed"。
- **DOCUMENTED_FACT**：官方文档明确规定的行为（如 VPC DNS = CIDR base+2）→ 措辞"根据文档"。
- **INFERENCE**：由观测+文档推出、存在反例可能 → 措辞"suggests/很可能"，不用"is/确认"。
- **UNKNOWN**：未验证、需客户提供 → 列入索取清单，绝不当事实写进结论或承诺。
纪律：每个断言对应一个级别；证据不足时不把 INFERENCE 说成根因（不做唱因归因），宁可回"这是一处确认的阻断，另有 A/B 需确认"，也不唱一个未经证实的单一根因。

---

## 评分与段位对照表（Scoring & Weakness Map）

### 一、总分段位

| 分数段 | 段位 | 说明 |
|---|---|---|
| 90–100 | **SME 精通（Expert）** | 可独立主导 R53 复杂 case，覆盖全域 |
| 80–89 | **熟练（Proficient）** | 重点域扎实，个别边缘知识点需补 |
| 70–79 | **及格（Pass）** | 达到 SME 基线，建议对错题域专项复习 |
| 60–69 | **待提升（Below Bar）** | 未达线，重点域存在系统性缺口 |
| < 60 | **需重训（Retrain）** | 建议通读题库 + 手册后重考 |

### 二、各知识域得分占比 → 薄弱域提示

> 算法：某域得分率 = 该域实得分 ÷ 该域满分。**得分率 < 70% 即列为薄弱域**，按下表定位补强方向。

| 域 | 题号范围 | 该域满分 | 薄弱时优先复习 |
|---|---|---|---|
| 01 Hosted Zones | Q1–Q4 | 5.8 | zone 合并顺序、PHZ 解析边界、最具体匹配、KSK 删除 |
| 02 记录与 Alias ⭐ | Q5–Q10 | 9.0 | apex 用 Alias、Alias vs CNAME、simple RRset、解析优先级 |
| 03 路由策略 ⭐ | Q11–Q18 | 12.2 | latency vs geo、default 兜底、weight=0、fail-open、ECS |
| 04 健康检查 ⭐ | Q19–Q25 | 11.0 | 内部资源用 CW HC、HTTPS 不校验证书、18% 聚合、阈值、abuse 流程 |
| 05 Resolver ⭐ | Q26–Q33 | 13.0 | VPC+2、.2 不过 SG/NACL、Forward>PHZ、In/Out 方向、`.`规则、QPS |
| 06 DNS Firewall | Q34–Q38 | 8.2 | 域名层过滤、priority 小优先、Action 面、CNAME 链、ALERT 日志 |
| 07 DNSSEC | Q39–Q43 | 7.0 | 禁用顺序、DS/DNSKEY 位置、island of trust、KMS CMK、+cd 判别 |
| 08 子域接管 | Q44–Q47 | 6.2 | Scenario 1 防护、≥1 NS 重叠、删除顺序、hold 续命 |
| 09 域名注册 | Q48–Q51 | 6.2 | .jp 特性、注册≠解析、账号关闭恢复、赎回期 |
| 10 Profiles | Q52–Q54 | 4.8 | 打包分发本质、local>Profile、四类可打包资源 |
| 11 ARC | Q55–Q58 | 6.2 | 开关非探针、data plane 5 endpoint、3/5 quorum、safety rule |
| 12 服务集成/配额 | Q59–Q62 | 6.2 | apex CNAME、数据面 vs 控制面限流、批量打包、ip-ranges 三分层 |
| 13 排查纪律 | Q63–Q65 | 6.6 | blast radius、响应码方向、四级证据分级 |
| | **合计** | **100** | |

### 三、薄弱域行动建议

- **重点加权域（02/03/04/05）任一 < 70%**：这些是考试主力域，优先回读对应章节的题库 + 手册对应 Topic，并把错题对应的真实 case 通读一遍。
- **DNSSEC（07）/ ARC（11）< 70%**：多为"顺序/位置/机制"记忆点，建议做流程图（如 DNSSEC 启用/禁用顺序、ARC 切流链条）强化。
- **排查纪律（13）< 70%**：这是横切方法论，薄弱会影响所有域答题措辞——重点掌握"blast radius 先界定""响应码方向""四级证据分级 + 不唱因归因"。
- **多选题失分集中**：注意多选须选全且不多选，复习时对每个选项逐一判断对错，而非只记正确项。


================================================================================

