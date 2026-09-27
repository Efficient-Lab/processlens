# ProcessLens 需求文档

> 单 App 文件与网络行为监控器 · 正式需求文档（PRD）
> See what an app reads, writes, and connects to.

| 项目 | 内容 |
|------|------|
| 文档类型 | 产品需求文档（PRD / SRS） |
| 文档状态 | Draft v1.0 |
| 平台 | macOS / Windows |
| 日期 | 2026-09-24 |
| 上游文档 | `docs/ProcessLens_单App行为监控器_项目企划书.txt` |

本文档将项目企划转化为可执行、可验证的需求条目。所有需求按模块编号，标注优先级（P0 = MVP 必须交付；P1 = 后续版本）。需求措辞遵循"只陈述观察事实"的产品原则。

---

## 1. 概述

### 1.1 产品定位

用户选择一个应用并开始记录，ProcessLens 只聚合该应用及其相关子进程的文件访问与网络连接，最终生成普通人能读懂的行为报告。回答的核心问题是："这个 App 在这段时间做了什么"。

### 1.2 目标用户

- 开发者（调试自己或第三方 App 的文件/网络行为）
- 隐私敏感用户与 power user
- 安全研究入门用户
- 经常试用 AI 客户端或小众工具的用户

### 1.3 产品边界（贯穿全部需求的硬约束）

- 只观察与报告：不做防火墙、不拦截、不注入、不做 HTTPS MITM、不抓取 payload。
- 不做安全结论：不给进程打"恶意/安全"评分，不把时序相关性写成因果结论。
- 全本地：Session 数据本地存储，无需账户，默认无遥测，导出由用户主动触发。

### 1.4 Go/No-Go 前提

macOS 精准文件事件归因依赖 Endpoint Security entitlement，需向 Apple 申请。**entitlement 获批是 macOS 正式版的 Go/No-Go 条件**，须在开发初期（G0）即提交申请，不作为发布前补办的细节。

---

## 2. 术语定义

| 术语 | 定义 |
|------|------|
| Responsible App | 用户选定的被监控目标应用（一次仅一个） |
| App Identity | 跨启动稳定识别目标应用的身份（Bundle ID / Signing ID / Windows 可执行文件身份） |
| Process Tree | 目标 App 的主进程及其 Helper、Renderer、XPC Service、Updater、spawned child 等全部归属进程 |
| Session | 一次 Start → Stop 的监控记录及其生成的报告 |
| FileEvent | 文件行为事件（READ / WRITE / CREATE / DELETE / RENAME） |
| NetFlow | 网络连接元数据（端点、协议、流量等），不含 payload |
| Sensitive Location | 可配置的敏感路径列表（如 `~/.ssh`、Documents、浏览器 profile），仅作中性标记 |

---

## 3. 用户场景与用例

| 编号 | 场景 | 用户问题 | 系统输出 |
|------|------|----------|----------|
| UC-1 | 试用新 AI 客户端 | 它有没有碰我的 Documents、SSH、Git 配置？连了哪些服务？ | 敏感路径访问 + 域名/IP + 时间线 |
| UC-2 | 开发调试 | 我的 App 为什么启动就写几千个文件或产生意外连接？ | 按目录/操作/进程聚合；高频路径与连接目标 |
| UC-3 | 隐私检查 | 这个小工具是不是在后台连统计/广告域名？ | 目标域名、端口、流量、首次/最后连接时间 |
| UC-4 | 卸载前检查 | 这个 App 的数据大概散在哪些位置？ | 写入/创建过的目录聚合，辅助理解残留范围 |
| UC-5 | 行为对比（P1） | 升级前后某 App 行为是否变化？ | 两个 Session 的差异报告 |

### 3.1 核心用户流程

1. 从已安装 App 列表中选择目标应用。
2. 点击 Start Monitoring；系统建立该应用身份与 process tree 追踪。
3. 用户正常使用目标 App。
4. 点击 Stop；系统停止收集并生成 Session Report。
5. 用户先看 Summary，再按 Files / Network / Timeline 下钻。
6. 可导出 JSON / CSV / Markdown 报告。

---

## 4. 功能需求

### FR-1 应用选择与身份归属

| 编号 | 需求 | 优先级 |
|------|------|--------|
| FR-1.1 | 系统应列出已安装应用供用户选择（macOS 按 Bundle，Windows 按可执行文件身份） | P0 |
| FR-1.2 | 系统应为选定应用建立 App Identity，保证跨启动稳定识别 | P0 |
| FR-1.3 | 系统应维护 `App Identity → Executables → Live Processes → Descendants` 四级归属映射 | P0 |
| FR-1.4 | 监控期间应持续跟踪目标 App 的 Helper / Renderer / XPC / Updater / spawned child process，其文件与网络事件统一归属到该 Session | P0 |
| FR-1.5 | 归属判断应基于签名 / Bundle / 父子进程等多信号综合判定，并在 Processes 页展示归属依据供用户核对 | P0 |
| FR-1.6 | 同一时刻只允许监控一个 Responsible App；不支持"选择单个 PID"作为产品模型 | P0 |

### FR-2 Session 生命周期

| 编号 | 需求 | 优先级 |
|------|------|--------|
| FR-2.1 | 用户可手动 Start / Stop 监控；Start 时记录 Session 元数据（目标应用、OS、版本、权限状态），Stop 时停止采集并生成报告 | P0 |
| FR-2.2 | Session 数据全量本地存储（SQLite），不落盘到应用目录以外的位置 | P0 |
| FR-2.3 | 目标 App 的进程退出 / 重启不应中断 Session；新启动的归属进程应继续被追踪 | P0 |
| FR-2.4 | 采集端崩溃或异常退出后，已采集的 Session 数据应可恢复，报告可基于已有数据生成 | P0（G4 完成） |
| FR-2.5 | 支持两个 Session 的对比差异报告 | P1 |

### FR-3 文件事件采集

| 编号 | 需求 | 优先级 |
|------|------|--------|
| FR-3.1 | 采集归属进程的文件事件：READ / WRITE / CREATE / DELETE / RENAME | P0 |
| FR-3.2 | 每条 FileEvent 记录：timestamp、process、operation、path、bytes（可选）、result；bytes 平台可得性不对称——Windows FileIo Read/Write 含 IoSize（已实测），macOS ES 文件事件无字节数（待 entitlement 获批后实测） | P0 |
| FR-3.3 | 不采集文件内容；报告中不出现文件正文 | P0 |
| FR-3.4 | 按路径、目录、进程三个维度聚合文件事件 | P0 |
| FR-3.5 | 对命中可配置敏感路径列表的访问做中性标记（Sensitive Location Access），不作恶意判定 | P0 |

### FR-4 网络事件采集

| 编号 | 需求 | 优先级 |
|------|------|--------|
| FR-4.1 | 采集归属进程的网络连接元数据：domain / IP、port、protocol、连接次数、首次/最后连接时间为 P0 必达；上行/下行流量为平台依赖——Windows 经 ETW Kernel-Network 已实测可用，macOS 需 NEFilterPacketProvider 逐包计数，列为 G1 验证项，成本不可接受则降级为连接数/时长并在 Summary 注明 | P0 |
| FR-4.2 | 每条 NetFlow 记录：timestamp、process、protocol、local/remote endpoint、hostname（可获得时，主要来源为 DNS 关联即 FR-4.4，直连 IP 场景无 hostname）、bytes up/down（平台依赖同 FR-4.1） | P0 |
| FR-4.3 | 不抓 payload、不 MITM、不解密 HTTPS；域名无法解析时诚实展示 IP 与可得字段，不伪造 URL | P0 |
| FR-4.4 | DNS 解析与 flow 关联（提升 hostname 覆盖率；不抓 payload）。不实现则 Network 页域名列大面积为空、Summary"主要连到哪里"卖点（UC-1/UC-3、AC-3）无法兑现。Windows 经 `Microsoft-Windows-DNS-Client` Operational 通道 + DNS 缓存实现（通道默认关闭，启用需管理员，已实测）；macOS 观察系统 DNS 解析路径 | P0 |

### FR-5 报告与信息架构

| 编号 | 需求 | 优先级 |
|------|------|--------|
| FR-5.1 | **Home**：选择 App、最近 Session 列表、权限状态、Start 入口 | P0 |
| FR-5.2 | **Summary**：监控时长、文件事件数、网络目标数、上传/下载量、敏感路径摘要；默认视图，风格为"行为账单"而非原始日志 | P0 |
| FR-5.3 | **Files**：Read / Write / Create / Delete / Rename 分类，按路径、目录、进程聚合 | P0 |
| FR-5.4 | **Network**：Domain/IP、Port、Protocol、连接次数、首末时间、流量统计 | P0 |
| FR-5.5 | **Timeline**：文件与网络事件按时间混排，支持过滤 | P0 |
| FR-5.6 | **Processes**：主进程、Helper、Renderer、XPC/子进程及归属关系 | P0 |
| FR-5.7 | **Export**：JSON / CSV / Markdown 导出；默认不包含不必要的文件内容 | P0 |
| FR-5.8 | Summary 应包含以下区块：Sensitive Access、Top Writes、New Files、Network Destinations、Timeline Correlation | P0 |
| FR-5.9 | 时序并排展示（如"READ file → 1s 后 outbound traffic"）必须标注为时间相关而非因果证明，UI 文案统一为 "Observed sequence, not proof of upload" | P0 |
| FR-5.10 | 导出前应提示报告可能包含用户名、路径、域名/IP 等敏感信息 | P0 |

### FR-6 权限与安装

| 编号 | 需求 | 优先级 |
|------|------|--------|
| FR-6.1 | macOS：通过 Endpoint Security System Extension 采集进程与文件事件；引导用户激活 System Extension 并授予 Full Disk Access | P0 |
| FR-6.2 | macOS：通过 Network Extension Content Filter System Extension 采集网络流元数据 | P0 |
| FR-6.3 | Windows：通过 ETW `Microsoft-Windows-Kernel-File` 采集文件事件（含 FileObject→path 映射管线），通过 ETW `Microsoft-Windows-Kernel-Network` 采集连接元数据与流量（无内核驱动），WFP `appId` 作 App 级归属辅助，DNS-Client Operational 通道做 hostname 关联；P0 只观察不阻断。全部采集面已在 Windows 11 26200 实测可用（`docs/analysis/08-windows-采集路径实测.md`） | P0 |
| FR-6.4 | 系统级采集由特权服务/系统扩展完成，GUI 通过 IPC 获取筛选后的 Session 数据 | P0 |
| FR-6.5 | 权限未授予或 System Extension 未激活时，Home 页应明确展示权限状态与引导，不允许静默降级采集 | P0 |
| FR-6.6 | 提供干净的卸载流程：移除系统扩展 / 服务与本地数据由用户可选 | P0（G4 完成） |

---

## 5. 非功能需求

### NFR-1 性能

| 编号 | 需求 | 优先级 |
|------|------|--------|
| NFR-1.1 | 监控不应明显拖慢被监控 App；高事件量场景下 UI 不卡死 | P0 |
| NFR-1.2 | 按平台能力尽早过滤（macOS ES 进程级 mute 待获批后实测；Windows ETW FileIo 为全系统流、无 PID 内核过滤，用户态过滤成本计入性能预算）、用户层批处理，使用 ring buffer；只持久化目标 App 的事件；非归属事件不持久化、不进日志、不进崩溃转储 | P0 |
| NFR-1.3 | 10–30 分钟 Session 持续录制数据稳定、无丢失性故障（G2 验收） | P0 |

### NFR-2 隐私与安全

| 编号 | 需求 | 优先级 |
|------|------|--------|
| NFR-2.1 | 默认不采集文件内容、不读取网络 payload，只记录行为元数据 | P0 |
| NFR-2.2 | Session 默认不上传；应用无账户体系；可选遥测必须完全独立且默认关闭 | P0 |
| NFR-2.3 | 敏感路径访问只做中性标记，不标恶意；不声称能证明某文件被上传 | P0 |

### NFR-3 合规与分发

| 编号 | 需求 | 优先级 |
|------|------|--------|
| NFR-3.1 | macOS 正式版必须取得 Endpoint Security entitlement，Network Extension 使用对应 Developer ID entitlement 配置 | P0（G0 启动） |
| NFR-3.2 | macOS 分发采用 Developer ID 签名 + 公证；不走 Mac App Store 仍需正确签名公证 | P0 |
| NFR-3.3 | Windows 采集服务按最小必要权限运行，GUI 与采集层分离 | P0 |

### NFR-4 可用性

| 编号 | 需求 | 优先级 |
|------|------|--------|
| NFR-4.1 | Summary 必须让非安全专业用户不读 raw event 即可回答"它碰了哪些重要位置、主要连到哪里"（G3 验收） | P0 |
| NFR-4.2 | 报告优先于原始日志：默认 Summary，高级用户再展开 Timeline 与 raw event | P0 |

### NFR-5 兼容性

| 编号 | 需求 | 优先级 |
|------|------|--------|
| NFR-5.1 | Windows：建议最低支持 Windows 10 1809+（Kernel-File/Kernel-Network manifest 架构自 RS4 稳定，待多版本验证）；已实测基线 Windows 11 build 26200 + SDK 10.0.26100 | P0 |
| NFR-5.2 | macOS：最低支持版本以系统扩展与 ES API 行为为准（建议 ≥13，待 entitlement 获批后实测） | P0 |

---

## 6. 数据模型需求

| 对象 | 关键字段 |
|------|----------|
| Session | target_app, start/end, OS, version, permissions |
| Process | pid, ppid, executable, signature/bundle identity, start/end |
| FileEvent | timestamp, process, operation, path, bytes(optional), result |
| NetFlow | timestamp, process, protocol, local/remote endpoint, hostname(if available), bytes up/down |
| Aggregate | folder/domain/process counts, totals, first/last seen |

要求：数据模型跨 macOS / Windows 统一，两平台共用 Summary、Timeline、Export 逻辑（UI/Core 层为 Tauri + Rust + SQLite）。

---

## 7. 平台技术约束

### 7.1 macOS

| 能力 | 技术 | 约束 |
|------|------|------|
| 进程 + 文件事件 | Endpoint Security System Extension | entitlement 需向 Apple 申请；用户需激活 System Extension 并授予 Full Disk Access |
| 网络流 | Network Extension Content Filter System Extension | Developer ID 直接分发需对应 entitlement/value 与系统扩展配置 |
| UI / 数据层 | Tauri + Rust + SQLite | 系统扩展用原生 Swift/Obj-C/C 封装，经 IPC 与主 App 通信 |
| 签名分发 | Developer ID + notarization | — |

### 7.2 Windows

| 能力 | 技术 | 说明 |
|------|------|------|
| 文件 I/O | ETW `Microsoft-Windows-Kernel-File` | Create/Name/Rename 族事件直接携带内核路径（`\Device\HarddiskVolumeN\`，需规范化层转盘符）；Read/Write 族仅携带 FileObject 指针 + `IOSize`，**路径需 FileObject→path 映射管线**（核心工程量，丢事件下可能错配）。事件经 `Execution.ProcessID` 携带 PID 归属；为全系统事件流、无 PID 内核过滤，用户态过滤成本计入性能预算（已实测，见 `docs/analysis/08-windows-采集路径实测.md`） |
| 网络连接元数据 | ETW `Microsoft-Windows-Kernel-Network` | **主采集源**：每事件携带 PID + 端点四元组 + size + connid + 连接生命周期（Connect/Accept/Disconnect/Reconnect），覆盖 FR-4.1/4.2 的端点与 bytes，无需内核驱动（已实测） |
| 网络归属辅助 | WFP net events（`appId`/`userId`） | `appId` 为可执行文件设备路径，作 App 级身份校验。限制：net events 无 PID/bytes；用户态 `FwpmNetEventSubscribe*` 只能订阅 DROP/CAPABILITY 类事件，**成功建连（ALE_AUTH_*）仅内核 callout 可得，P0 不走该路径**（已实测） |
| hostname 关联 | `Microsoft-Windows-DNS-Client` Operational 通道 | 默认关闭，启用需管理员；配合 DNS 缓存查询提升 hostname 覆盖率（FR-4.4） |
| 权限服务 | Privileged Windows service | **硬依赖**：非管理员启动 ETW 会话被系统拒绝（已实测）；系统级采集在特权服务内完成，GUI 经 IPC 取筛选后数据 |
| 基线快照 | `GetExtendedTcpTable` / `Get-DnsClientCache` / `Get-AuthenticodeSignature` | Session Start 时枚举既有连接（带 PID）、DNS 缓存、签名身份，均免管理员可用（已实测） |
| UI / Core | Tauri + Rust | 与 macOS 共用数据模型与报告逻辑；`ferrisetw`/`windows` crate 提供 ETW/WFP 绑定 |

---

## 8. 明确不做（Scope Exclusion）

| 范围 | 说明 |
|------|------|
| 全系统长期监控 / EDR | 只做单 App session recording |
| 恶意/安全评分 | 不对未知进程下结论 |
| 文件内容抓取 | 只记行为元数据 |
| HTTPS MITM / URL path 解密 / payload 抓包 | 只记连接元数据 |
| 实时阻断网络或文件访问 | 观察与报告定位 |
| 云端账户 / 社区风险数据库 | 全本地 |
| 前后快照对比降级方案 | 若 entitlement 长期不获批，不退化为快照工具（企划书 Go/No-Go 结论） |

---

## 9. 验收标准（MVP）

| 编号 | 验收项 |
|------|--------|
| AC-1 | 选择一个 App 后，主进程及主要 helper/child 的事件能被统一归属 |
| AC-2 | Files 稳定展示 READ / WRITE / CREATE / DELETE / RENAME；Network 展示目标地址、端口、连接次数和基础流量统计 |
| AC-3 | Summary 让用户不阅读 raw event 即可回答"它碰了哪些重要位置、主要连到哪里" |
| AC-4 | 监控不明显拖慢被监控 App；高事件量场景 UI 不卡死 |
| AC-5 | 报告中不出现将时间相关性误写为数据泄露结论的表述 |
| AC-6 | macOS 正式版在 entitlement、签名、公证和用户权限流程上完全合规 |

---

## 10. 里程碑与 Gate

| 阶段 | 工作 | Gate |
|------|------|------|
| G0 权限申请 | 提交 Apple Endpoint Security entitlement + Network Extension content-filter entitlement（双申请，后者同为受限审批项） | 申请已提交；明确审批路径 |
| G1 PoC | 拆为三项：①macOS 抓取目标 App 文件+网络事件并关联 process tree（依赖 G0 获批，无降级路径——未获批时 ES 客户端 exec 即终止）；②macOS PacketProvider 包级流量通道吞吐成本验证（决定 bytes 是否保留）；③Windows 抓取目标 App 文件+网络事件（采集面已实测可行，剩 FileObject→path 映射准确率与用户态过滤性能成本两个验证项） | 单 Session 数据可信 |
| G2 Core | 统一事件模型、SQLite、聚合、Timeline、Export | 10–30 分钟 Session 稳定 |
| G3 UX | App Picker、Summary、敏感路径、域名聚合、权限 onboarding | 非安全专业用户可看懂 |
| G4 Beta | 签名、公证、安装/卸载、崩溃恢复、性能 | 真实用户环境可持续使用 |

---

## 11. 风险与依赖

| 风险 | 影响 | 控制措施 |
|------|------|----------|
| Apple entitlement 未获批/审批慢 | macOS 核心功能无法正式分发；且已实测未获批时 ES 客户端 exec 即终止、零降级路径 | G0 第一天提交双申请（ES + NE）；Windows beta 先行作为正式路径 |
| Helper/XPC 归属错误 | 漏掉或错算行为 | 签名/Bundle/父子进程多信号归属（FR-1.5）；Windows 侧以 Kernel-Network PID 字段 + WFP appId 多信号归属（已实测）；Processes 页展示归属依据供核对 |
| 网络域名无法完整解析 | 只能看到 IP 或 hostname 不完整 | 诚实展示可得字段（FR-4.3）；DNS/flow 关联已升 P0（FR-4.4，Windows 实现路径已实测） |
| Windows FileObject→path 映射错配 | 文件路径错误或缺失 | G1 专项验证映射准确率；缺失时降级展示 file object id + 卷路径；Plan B 为 minifilter 驱动（分发成本上升） |
| 事件量大影响性能 | 目标 App 或系统变慢 | 按平台能力尽早过滤、用户层批处理、ring buffer（NFR-1.2）；Windows 实测 2 秒约 4.6 万条全系统事件，用户态过滤成本列入 G1 验证 |
| 用户把时序当因果 | 误判隐私行为 | 统一 UI 文案标注（FR-5.9） |
| 免费项目维护成本 | 拖累收费产品开发 | 严格 P0 边界；不做 blocking / IDS / malware DB / HTTPS decrypt |

### 外部依赖

- Apple Endpoint Security entitlement 审批（macOS 硬门槛）+ Network Extension content-filter entitlement（同为受限审批项）
- Apple Developer ID 签名与公证服务
- Windows 代码签名证书（OV/EV）与 SmartScreen 信誉积累（Windows beta 先行时为分发硬门槛）
- 开源传播渠道：GitHub / HN / Reddit（品牌目标，非功能依赖）

---

## 12. 附录：官方技术参考

- Apple — Monitoring System Events with Endpoint Security：<https://developer.apple.com/documentation/endpointsecurity/monitoring-system-events-with-endpoint-security>
- Apple — System Extensions：<https://developer.apple.com/documentation/systemextensions>
- Apple — Network Extension Entitlement：<https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.networking.networkextension>
- Microsoft — ETW FileIo：<https://learn.microsoft.com/windows/win32/etw/fileio>
- Microsoft — ETW Kernel-Network：<https://learn.microsoft.com/windows/win32/etw/ms-windowskernelnetwork>
- Microsoft — Windows Filtering Platform：<https://learn.microsoft.com/windows/win32/fwp/about-windows-filtering-platform>

Windows 侧采集能力实测记录见 `docs/analysis/08-windows-采集路径实测.md`（2026-09-27，Windows 11 26200）。
