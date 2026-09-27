# ProcessLens 桌面端资深工程师视角分析

> 分析对象：`docs/ProcessLens_需求文档.md`（PRD Draft v1.0，主依据）
> 参照文档：`docs/ProcessLens_单App行为监控器_项目企划书.txt`（上游企划）
> 视角：Desktop App Engineering —— 关注系统扩展/特权服务的真实能力边界、IPC 与进程边界、签名分发链路、性能预算可验证性。
> 日期：2026-09-24
>
> 约定：凡涉及 macOS ES / Network Extension、Windows ETW / WFP 等 API 能力边界的事实判断，一律标注"待实测验证"，以 G1 PoC 实测结果为准，本文不作断言。

---

## 总体结论

**架构方向成立，但 §7 平台技术约束有四处实质性乐观/错误表述，且数据模型中两个字段在选定遥测源上不可得。**

1. **macOS ES 没有"READ"事件**。ES 文件事件集为 open/close/create/write/unlink/rename/clone/exchangedata 等，"READ"只能由 `OPEN(FREAD)` 或 `CLOSE(modified)` 近似（待实测验证具体事件语义与可用版本下限）。这意味着 FR-3.1 的"READ"实际语义是"以读方式打开"，缓存命中、已持有 fd 的重复读、部分 mmap 场景不产生新事件（待实测验证）。此外 ES 事件携带 path，但**不携带字节数**（待实测验证 `WRITE` 事件字段集），FR-3.2 的 `bytes` 在 macOS 上大概率为空。
2. **macOS NE Content Filter 的 flow 元数据不含流量字节数**。`NEFilterDataProvider` 提供 flow 级元数据（endpoint、方向、sourceApp/auditToken），上行/下行字节需引入 `NEFilterPacketProvider` 逐包计数（不读 payload，但全量包经过扩展，有吞吐成本，待实测验证）。FR-4.1/FR-4.2 的"bytes up/down"在 macOS 上要么加包级通道，要么降级为连接数/时长。
3. **Windows ETW FileIo 不含路径**。FileIo 事件携带 FileObject 指针，路径需由 `FileIo_Name`/`FileCreate` 类事件构建 FileObject→path 映射（待实测验证映射完整性与丢事件下的错配率）。这是 §7.2 完全没有提及的核心工程量；且 FileIo 为全系统事件流、无法按 PID 在内核侧过滤（待实测验证），NFR-1.2"内核层尽早过滤"在 Windows 上不成立。
4. **macOS 双 entitlement 门槛被低估**。`com.apple.developer.networking.networkextension`（content-filter-provider 值）与 ES entitlement 同属受限授权、均需向 Apple 申请（待实测验证当前审批流程）；文档把 NE 写成"确认配置"，把 Go/No-Go 单挂在 ES 上。G0 应同时提交两项申请。另外，获批前的开发联调可能需要 provisioning profile 或降级 SIP 测试环境（待实测验证），影响 G1 排期。

> 〔勘误〕2026-09-27 核实 Apple 官方文档：NE capability 自 2016 年起自助开启、无需审批；"双门槛/G0 双申请"不成立，Go/No-Go 单挂 ES 的口径正确。需申请的仅 `endpoint-security.client`；NE 的 Developer ID 注意点为 `-systemextension` 后缀值 + 手动签名。SIP 降级联调路径猜测成立：SIP-off 环境无 entitlement 也可运行 ES client（Apple 明示审批期间可临时禁用 SIP 测试）。详见 `07-macos-entitlement实测.md` 追加调查。

**其余判断**：归属模型（FR-1.3/1.5）方向正确——macOS 上 XPC/Helper 的 ppid 是 launchd，父子链必然断裂，签名/Bundle 多信号归属是必选项而非增强项；Windows P0 用 user-mode `FwpmNetEvents` + ETW 即可覆盖连接元数据，**无需内核驱动**，签名门槛低，Windows 先行 beta 的策略可行。"无丢失性故障"（NFR-1.3）与"不明显拖慢"（NFR-1.1）按现措辞**无法验收**，必须量化（见 NFR 修改建议）。跨平台统一数据模型可行，但 `bytes`、`hostname`、`READ` 语义三项存在平台不对称，需要在模型层显式处理。

结论：**Conditional Go 不变，但建议把 §7 按本文修正重写，并把"macOS 流量字节数"与"Windows 路径解析"列为 G1 的两个独立验证项——它们是当前 MVP 范围里唯二可能推翻 P0 承诺的点。**

---

## 技术风险清单

| 编号 | 严重度 | 涉及条目 | 风险 | 建议/替代方案 |
|------|--------|----------|------|----------------|
| TR-01 | 高 | FR-3.1、§7.1 | ES 无 READ 事件；"读"只能由 OPEN(FREAD)/CLOSE(modified) 近似，语义为"打开"而非"实际读取"；缓存读、长持 fd 重读不可见（均待实测验证） | 数据模型 operation 按平台事件语义重定义；Summary/Files 文案区分"打开读取"与"写入"；G1 实测事件语义矩阵（含 macOS 版本下限） |
| TR-02 | 高 | FR-4.1/4.2、§7.1、FR-5.2 | NE flow 元数据无 bytes；流量统计需 NEFilterPacketProvider 逐包经过扩展（不读 payload，但全量流量过通道），吞吐/延迟成本未知（待实测验证） | G1 设专项：包级通道吞吐基准 + 目标 App 网速回归测试；若成本不可接受，macOS 降级为连接数/时长/端点，bytes 标"可得时"（Windows 侧有替代源，见 TR-06） |
| TR-03 | 高 | §1.4、§7.1、§10 G0 | NE content-filter entitlement 同为受限授权需 Apple 审批（待实测验证流程）〔勘误：NE 无需审批、自助开启，2026-09-27 核实〕；获批前开发联调可能依赖 profile 或 SIP 降级环境〔已核实：SIP-off 可运行〕 | G0 同时提交 ES + NE 两项申请〔勘误：仅 ES 需申请〕；Go/No-Go 改为双门槛〔勘误：仍为 ES 单门槛〕；G1 排期预留审批等待期的降级验证路径 |
| TR-04 | 高 | §7.2、FR-6.3、FR-3.x | ETW FileIo 不含路径；FileObject→path 映射需消费 Name/FileCreate 流并在丢事件、对象复用、rename 链下保持正确（待实测验证准确率） | G1 专项验证映射管线；定义映射缺失时的降级展示（file object id + 卷路径）；备选 minifilter 驱动（路径准确但需驱动签名，分发门槛上升，列为 Plan B） |
| TR-05 | 高 | NFR-1.2、§7.2 | FileIo 为全系统事件流，无法按 PID 内核过滤（待实测验证 provider 过滤能力）；高负载下用户态过滤成本与 ETW 会话缓冲溢出丢事件是真实风险 | NFR-1.2 改为"按平台能力尽早过滤"；ETW 会话缓冲、批大小、丢弃计数纳入性能预算；Session 元数据落 `lost_events` 计数并在报告可见 |
| TR-06 | 中 | FR-4.1/4.2、§7.2 | WFP ALE net events 提供端点/appId/userId，无 bytes（待实测验证是否含 PID） | Windows 流量字节用 ETW Kernel-Network（含 PID + size，无驱动，待实测验证字段）或 TCPIP ESTATS 轮询补足；无需为 bytes 上 callout 驱动 |
| TR-07 | 中 | FR-4.2/4.4、FR-5.4、UC-1/3 | hostname 覆盖率两平台都低：NE remoteEndpoint 在 macOS 上通常只有 IP（待实测验证），WFP 侧本就无 hostname；不做 DNS 关联则"域名"列大面积为空，Summary 核心卖点落空 | FR-4.4 DNS/flow 关联升为 P0（观察 DNS 流量或系统解析缓存，不抓 payload）；诚实展示 IP + 可得字段 |
| TR-08 | 中 | FR-1.4、FR-2.3 | 中途 attach：NE filter 只见激活后新建的 flow，既有长连接不可见（待实测验证）；既有进程的 FileObject→path 映射同样缺失 | 补"Session 启动基线快照"需求：枚举目标进程树 + 既有连接快照（macOS proc_pidinfo 类接口 / Windows GetExtendedTcpTable，待实测验证） |
| TR-09 | 中 | FR-6.1、§7.1 | FDA 对 ES 事件投递范围的影响未实测：无 FDA 时受保护目录（Safari、邮件、TCC 保护位置）的事件/路径是否完整下发（待实测验证） | G1 实测"无 FDA 事件盲区矩阵"；若存在盲区，把 FDA 从"建议授予"升级为采集正确性硬依赖并在权限引导中说明原因 |
| TR-10 | 中 | §7.1、FR-6.6 | ES 客户端打包形态未定案（系统扩展内嵌 vs 特权 daemon/SMAppService，两种形态的安装激活、升级、卸载路径不同，待实测验证当前 macOS 版本支持矩阵） | G1 前定案；卸载流程（FR-6.6）按选定形态单独设计激活解除顺序 |
| TR-11 | 中 | NFR-1.1、§7.1 | NEFilterDataProvider 是内联通道：每个新 flow 需返回 verdict，扩展延迟/崩溃直接影响目标 App 建连（待实测验证失败模式与回退行为） | verdict 路径零分配/常数时间；扩展崩溃时的系统回退行为实测；将"扩展导致的建连延迟"纳入 NFR-1 指标 |
| TR-12 | 低 | FR-3.1、§6 | DELETE/RENAME 语义平台差异：Windows delete 多为 delete-on-close/SetInfo disposition，macOS 有 unlink/rename/exchangedata/clone 多种表达（待实测验证映射完备性） | 建平台操作映射表为需求附件；聚合层以归一化 operation 为准，原始操作保留备查 |
| TR-13 | 低 | FR-1.2、§7.2 | Windows 未签名应用常见、macOS ad-hoc 签名存在；签名身份缺失时 App Identity 跨启动稳定性无保障 | App Identity 定义分层 fallback：签名 > Bundle/路径 > 文件哈希 + 路径；Processes 页展示所用信号（与 FR-1.5 一致） |
| TR-14 | 低 | §7.2、FR-1.4 | WFP ALE net event 归属字段为 appId（设备路径）/userId，是否含 PID 待实测验证；若不含，进程级归属需额外关联 | 归属以 appId（App 级）为准更符合产品模型；PID 级展示由 ES/ETW 进程事件补齐 |

---

## 需求缺口清单

需求稿未写、但实现必需的项目：

- **GAP-01 IPC 协议规范**：FR-6.4 只有"GUI 通过 IPC 获取筛选后的 Session 数据"一句。必需定义：通道形态（macOS 扩展↔App、Windows 服务↔GUI 各自选型）、对端鉴权（仅自家 GUI 可连，本机也不可信默认）、消息 schema 与版本协商、推流/拉取模式与背压策略、断线重连与事件重放语义。这是特权边界上唯一的攻击面，建议按"窄动词 + 输入校验"原则先写契约再写 UI。
- **GAP-02 Session 启动基线快照**：Start 时目标 App 可能已在运行且已有长连接/已打开文件。需枚举既有进程树、既有网络连接、进行中 FileObject→path 状态（配合 TR-08），否则 Session 前半段系统性漏报。
- **GAP-03 事件丢失计数与上报**：ETW 缓冲溢出、ring buffer 溢出、ES 背压丢弃均真实存在（待实测验证各源丢弃语义）。Session 元数据需 `lost_events` 计数字段，报告端诚实展示"本 Session 约丢失 N 条事件"——这既是 NFR-1.3 的验收依据，也是"只陈述观察事实"原则的延伸。
- **GAP-04 事件去重与时间线排序**：文件源与网络源是两个时钟/队列，Timeline 混排（FR-5.5）需要跨源时间对齐与排序规则；重复 open、同路径高频 IO 需要合并窗口定义，否则 UC-2"启动写几千个文件"会撑爆 UI 与存储。
- **GAP-05 路径规范化层**：macOS firmlink（`/private/var` ↔ `/var`、`/System/Volumes/Data`）、符号链接、大小写不敏感 APFS；Windows `\Device\HarddiskVolumeN\` → 盘符、UNC 路径、8.3 短名（均待实测验证覆盖）。敏感路径匹配（FR-3.5）与目录聚合（FR-3.4）的正确性全部依赖该层，应单列为核心模块需求。
- **GAP-06 归属映射的崩溃重建**：FR-2.4 要求崩溃后可恢复，但 PID→identity 归属映射在采集进程内存中；重启后需从进程快照 + ES exec 历史重建映射，否则恢复后事件无法归属。需写进 FR-2.4 的实现约束。
- **GAP-07 敏感路径列表管理界面**：FR-3.5 说"可配置敏感路径列表"，但全部 FR-5 页面需求中没有设置入口。需补 FR（设置页：默认列表 + 增删改）。
- **GAP-08 SQLite schema 版本与迁移**：FR-2.2 定了本地存储，未定 schema 版本策略。多版本共存（用户升级 App 后旧 Session 仍可读）需要 user_version + 迁移脚本需求。
- **GAP-09 运行中权限撤销检测**：FR-6.5 只规定启动前/Home 页权限状态。监控中用户撤销 FDA、停用扩展、退出目标 App 的运行时检测与 Session 状态迁移（标记 degraded 而非静默漏采）需补需求。
- **GAP-10 PID 复用防护**：数据模型 Process 有 pid + start/end，但需求未显式规定事件键控为 (pid, start_time)；Windows PID 复用快，错归属风险需在 FR-1.4 或数据模型注释中显式排除。
- **GAP-11 卸载/停用顺序**：FR-6.6 只说"移除系统扩展/服务与本地数据"。监控进行中卸载、系统扩展停用需宿主 App 在场、Windows 服务删除顺序等生命周期细节需展开。
- **GAP-12 签名/构建流水线需求**：NFR-3.2 只写结果。实际需：App + 两个系统扩展各自的 entitlements/provisioning profile 管理、公证流程覆盖内嵌扩展、CI 产物矩阵（macOS arm64/x86_64、Windows x64）、G4 前必须有能跑通的 walking-skeleton 签名构建。
- **GAP-13 目标 App 升级导致的 identity 漂移**：被监控 App 自动更新后签名 ID 不变但版本/哈希变化，"跨启动稳定识别"（FR-1.2）需明确对版本变化的容忍策略（identity 不含版本，版本作为 Session 元数据记录）。

---

## 需要修改的需求条目

| 条目 | 原文 | 建议改法 |
|------|------|----------|
| FR-3.1 | "采集归属进程的文件事件：READ / WRITE / CREATE / DELETE / RENAME" | 改为按平台事件语义定义并附映射表：macOS 侧 READ ≈ OPEN(FREAD)/CLOSE(modified)（"读"=以读方式打开，非逐次读取；缓存读与长持 fd 不可见，待实测验证）；Windows 侧按 FileIo 事件族映射。报告文案同步使用"打开读取"口径 |
| FR-3.2 | "每条 FileEvent 记录：timestamp、process、operation、path、bytes（可选）、result" | 保留 optional，但加注平台可得性：macOS ES 文件事件无字节数（待实测验证 WRITE 事件字段集）→ macOS 恒为空；Windows FileIo Read/Write 有 IoSize 可用。Summary 的"估算字节数"类区块需在 macOS 上有降级展示 |
| FR-4.1 | "……连接次数、首次/最后连接时间、上行/下行流量" | 拆为两级：连接元数据（端点/协议/次数/首末时间）为 P0 必达；流量字节数标注"平台依赖"——Windows 经 ETW Kernel-Network 或 ESTATS（P0，无驱动）；macOS 需 NEFilterPacketProvider，列为 G1 验证项，成本不可接受则降级为连接数/时长并在 Summary 注明 |
| FR-4.2 | "……hostname（可获得时）、bytes up/down" | hostname 补注"主要来源为 DNS/flow 关联（见 FR-4.4），直连 IP 场景无 hostname"；bytes 同 FR-4.1 平台差异标注 |
| FR-4.4 | "DNS 解析与 flow 关联增强（提升 hostname 覆盖率）｜P1" | 升 P0。理由：macOS NE remoteEndpoint 多为 IP（待实测验证）、Windows 本就无 hostname；不做关联则 Network 页"域名"列与 Summary"主要连到哪里"的核心卖点（UC-1/UC-3、AC-3）无法兑现 |
| §7.1 ES 行 | "用户需激活 System Extension 并授予 Full Disk Access" | 补两点：①ES 客户端打包形态（系统扩展 vs 特权 daemon）待 G1 定案；②FDA 与事件投递范围的关系待实测验证——若受保护目录事件依赖 FDA，则 FDA 是采集正确性硬依赖而非可选权限 |
| §7.1 NE 行 | "Developer ID 直接分发需对应 entitlement/value 与系统扩展配置" | 明确为受限授权：`com.apple.developer.networking.networkextension`（content-filter-provider）需向 Apple 申请审批（待实测验证流程），与 ES 并列进入 §1.4 Go/No-Go；G0 双申请。〔勘误〕前提不成立：NE 自助开启无需审批（2026-09-27 核实）；正确要求是 Developer ID 用 `-systemextension` 后缀值 + 手动签名，PRD v1.1 §7.1 已按此回写 |
| §7.1 ES 行（隐含） | — | 增补：获批前开发联调可能需 provisioning profile 或 SIP 降级环境（待实测验证），G1 排期预留审批等待期。〔已核实〕SIP-off 环境 ES client 可运行（Apple 明示）；SIP 开启下 dev-signed exec 即被终止（2026-09-25 实测） |
| §7.2 文件 I/O 行 | "Create / Read / Write / Delete / Rename，结合 process/thread 信息做归属" | 补关键事实：FileIo 事件不含路径，需 FileObject→path 映射管线（工程量主体，丢事件下可能错配）；FileIo 无法按 PID 内核过滤，全系统流量需用户态过滤并计入性能预算（均待实测验证） |
| §7.2 网络行 | "WFP / ALE 层 按 application / user / connection 过滤；P0 只观察元数据" | 明确 P0 实现为 user-mode `FwpmNetEvents` 订阅（无内核驱动、签名门槛低）；归属字段为 appId/userId，是否含 PID 待实测验证；bytes 由 ETW Kernel-Network 补足而非 WFP |
| NFR-1.1 | "监控不应明显拖慢被监控 App；高事件量场景下 UI 不卡死" | 量化为可验收指标草案：①目标 App 基准操作（冷启动/典型任务）开启监控后耗时增量 ≤10%；②采集进程常驻 CPU ≤5% 单核、RSS ≤200MB（草案值，G2 校准）；③定义"高事件量"负载（如 ≥10k 事件/秒持续 60s）下 GUI p95 帧间隔 ≤16ms、事件到 UI 可见延迟 ≤2s；④macOS 侧另测扩展对目标 App 建连延迟的影响 |
| NFR-1.2 | "事件在内核层尽早过滤、用户层批处理，使用 ring buffer" | 改为"按平台能力尽早过滤"：macOS ES 进程级 mute 可用（待实测验证 mute 粒度）；Windows ETW FileIo 无 PID 内核过滤，用户态过滤成本计入预算；保留批处理 + ring buffer 架构要求 |
| NFR-1.3 | "10–30 分钟 Session 持续录制数据稳定、无丢失性故障（G2 验收）" | 定义验收口径：规定负载曲线下，事件丢失率 ≤0.1%（草案）且 `lost_events` 计数在报告中可见；"无丢失性故障"改为"丢失可计量、可上报"——要求零丢失不可证伪也不可实现 |
| FR-2.3 | "目标 App 的进程退出 / 重启不应中断 Session；新启动的归属进程应继续被追踪" | 补中途 attach 基线要求：Session Start 时对既有进程树与既有连接做快照（对应 GAP-02），否则"新启动进程被追踪"成立而"已在运行进程"漏采 |
| FR-2.4 | "采集端崩溃或异常退出后，已采集的 Session 数据应可恢复" | 补归属映射重建约束：崩溃恢复后需从进程快照重建 PID→identity 映射（GAP-06）；恢复中断窗口内的事件标记为缺失而非静默丢弃 |
| FR-6.5 | "权限未授予或 System Extension 未激活时，Home 页应明确展示权限状态与引导" | 扩为运行时监测：监控中权限被撤销/扩展被停用时，Session 标记 degraded 并提示用户，禁止静默降级（与 FR-6.5 现有"不允许静默降级"原则一致，但覆盖运行期） |
| FR-1.2 | "建立 App Identity，保证跨启动稳定识别" | 补分层 fallback：签名身份 > Bundle/安装路径 > 文件哈希+路径（Windows 未签名与 macOS ad-hoc 场景）；目标 App 升级导致的哈希/版本变化不破坏 identity（版本记入 Session 元数据，GAP-13） |
| FR-5.5 | "文件与网络事件按时间混排" | 补跨源时间对齐说明：两采集源时间戳需统一单调基准，混排为近似有序（对应 GAP-04）；FR-5.9 的文案原则此处同样适用 |
| §9 AC-4 | "监控不明显拖慢被监控 App；高事件量场景 UI 不卡死" | 替换为 NFR-1.1 修订后的量化指标引用（同上），使 AC 可判定 |
| §10 G1 | "macOS/Windows 各抓取一个目标 App 的文件事件和网络连接" | G1 拆为三个并行验证项：①事件归属链路（现有）；②macOS 包级流量通道成本（TR-02）；③Windows FileObject→path 映射准确率（TR-04）。Gate 条件相应改写 |

---

## 附：跨平台字段可得性矩阵（供 §6 数据模型修订参考）

| 字段 | macOS | Windows | 备注 |
|------|-------|---------|------|
| FileEvent.path | ✓（es_file_t） | ✓（需 FileObject→path 映射管线，TR-04） | 均待实测验证边界场景 |
| FileEvent.bytes | ✗（待实测验证 WRITE 事件字段） | ✓（IoSize） | 平台不对称，保留 optional |
| FileEvent 语义 READ | "以读打开"近似 | 真实 Read IRP | 同名字段不同语义 |
| NetFlow endpoint/port/proto | ✓ | ✓ | — |
| NetFlow hostname | 多为 IP，需 DNS 关联 | 无，需 DNS 关联 | FR-4.4 升 P0 的依据 |
| NetFlow bytes | 需 PacketProvider（成本待实测） | ETW Kernel-Network（待实测验证） | 不同遥测源 |
| 进程归属信号 | 签名/Bundle/auditToken 丰富 | 签名缺失常见，需 fallback | TR-13 |
| 内核侧按目标过滤 | ES mute（进程级，待实测验证） | FileIo 无 PID 过滤 | NFR-1.2 措辞冲突根源 |

