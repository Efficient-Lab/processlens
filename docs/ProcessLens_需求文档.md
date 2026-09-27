# ProcessLens 需求文档

> 单 App 文件与网络行为监控器 · 正式需求文档（PRD）
> See what an app reads, writes, and connects to.

| 项目 | 内容 |
|------|------|
| 文档类型 | 产品需求文档（PRD / SRS） |
| 文档状态 | Draft v1.1（修正 NE entitlement 审批口径与 SIP-off 联调路径，见 §7.1/§10） |
| 平台 | macOS / Windows |
| 日期 | 2026-09-27 |
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

macOS 精准文件事件归因依赖 Endpoint Security entitlement，需向 Apple 申请。**entitlement 获批是 macOS 正式版的 Go/No-Go 条件**，须在开发初期（G0）即提交申请，不作为发布前补办的细节。它是 macOS 侧唯一需要 Apple 审批的能力：Network Extension 相关 capability 为自助开启（见 §7.1）。未获批期间 SIP 开启环境下 ES 客户端 exec 即被终止、无法运行（`docs/analysis/07` 实测）；官方认可的联调路径为审批期间临时禁用 SIP。

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
| FR-2.6 | **Session 启动基线快照**：Start 时枚举既有进程的连接与打开文件作为基线（macOS 用 `proc_pidfdinfo`，Windows 用 `GetExtendedTcpTable`），补偿中途 attach 看不到既有连接的问题；免 entitlement 即可实现（GAP-02） | P0 |

### FR-3 文件事件采集

| 编号 | 需求 | 优先级 |
|------|------|--------|
| FR-3.1 | 采集归属进程的文件事件：READ / WRITE / CREATE / DELETE / RENAME | P0 |
| FR-3.2 | 每条 FileEvent 记录：timestamp、process、operation、path、bytes（可选）、result | P0 |
| FR-3.3 | 不采集文件内容；报告中不出现文件正文 | P0 |
| FR-3.4 | 按路径、目录、进程三个维度聚合文件事件 | P0 |
| FR-3.5 | 对命中可配置敏感路径列表的访问做中性标记（Sensitive Location Access），不作恶意判定 | P0 |

### FR-4 网络事件采集

| 编号 | 需求 | 优先级 |
|------|------|--------|
| FR-4.1 | 采集归属进程的网络连接元数据：domain / IP、port、protocol、连接次数、首次/最后连接时间、上行/下行流量 | P0 |
| FR-4.2 | 每条 NetFlow 记录：timestamp、process、protocol、local/remote endpoint、hostname（可获得时）、bytes up/down | P0 |
| FR-4.3 | 不抓 payload、不 MITM、不解密 HTTPS；域名无法解析时诚实展示 IP 与可得字段，不伪造 URL | P0 |
| FR-4.4 | DNS 解析与 flow 关联增强（提升 hostname 覆盖率） | P1 |

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
| FR-6.3 | Windows：通过 ETW FileIo 采集文件事件，通过 WFP ALE 层采集网络元数据；P0 只观察不阻断 | P0 |
| FR-6.4 | 系统级采集由特权服务/系统扩展完成，GUI 通过 IPC 获取筛选后的 Session 数据 | P0 |
| FR-6.5 | 权限未授予或 System Extension 未激活时，Home 页应明确展示权限状态与引导，不允许静默降级采集 | P0 |
| FR-6.6 | 提供干净的卸载流程：移除系统扩展 / 服务与本地数据由用户可选 | P0（G4 完成） |

---

## 5. 非功能需求

### NFR-1 性能

| 编号 | 需求 | 优先级 |
|------|------|--------|
| NFR-1.1 | 监控不应明显拖慢被监控 App；高事件量场景下 UI 不卡死 | P0 |
| NFR-1.2 | 事件在内核层尽早过滤、用户层批处理，使用 ring buffer；只持久化目标 App 的事件 | P0 |
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
| 进程 + 文件事件 | Endpoint Security System Extension | entitlement 需向 Apple 申请（macOS 侧唯一审批项）；审批期间可在 SIP 关闭的测试环境联调；用户需激活 System Extension 并授予 Full Disk Access（用户授权，非 entitlement） |
| 网络流 | Network Extension Content Filter System Extension | entitlement 为自助开启，无需 Apple 审批；Developer ID 分发必须用 `-systemextension` 后缀值（`content-filter-provider-systemextension`）+ 手动签名/provisioning profile；宿主 App 需 `system-extension.install`（自助） |
| UI / 数据层 | Tauri + Rust + SQLite | 系统扩展用原生 Swift/Obj-C/C 封装，经 IPC 与主 App 通信 |
| 签名分发 | Developer ID + notarization | — |

### 7.2 Windows

| 能力 | 技术 | 说明 |
|------|------|------|
| 文件 I/O | ETW FileIo events | Create / Read / Write / Delete / Rename，结合 process/thread 信息做归属 |
| 网络 | WFP / ALE 层 | 按 application / user / connection 过滤；P0 只观察元数据 |
| 权限服务 | Privileged Windows service | 系统级采集，GUI 经 IPC 取数 |
| UI / Core | Tauri + Rust | 与 macOS 共用数据模型与报告逻辑 |

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
| G0 权限申请 | 提交 Apple Endpoint Security entitlement（注明 Developer ID 分发意图）；验证 NE Developer ID 配置：`-systemextension` 后缀 entitlement 值 + 手动签名 + provisioning profile | ES 申请已提交；NE 无需审批、自助开启，仅需打通 Developer ID 签名链 |
| G1 PoC | macOS/Windows 各抓取一个目标 App 的文件事件和网络连接，能关联 process tree | 单 Session 数据可信 |
| G2 Core | 统一事件模型、SQLite、聚合、Timeline、Export | 10–30 分钟 Session 稳定 |
| G3 UX | App Picker、Summary、敏感路径、域名聚合、权限 onboarding | 非安全专业用户可看懂 |
| G4 Beta | 签名、公证、安装/卸载、崩溃恢复、性能 | 真实用户环境可持续使用 |

---

## 11. 风险与依赖

| 风险 | 影响 | 控制措施 |
|------|------|----------|
| Apple entitlement 未获批/审批慢 | macOS 核心功能无法正式分发 | G0 第一天提交申请；审批期间用 SIP-off 测试机联调 PoC（2026-09-25 实测：SIP 开启下 ES 客户端 exec 即被终止，无其他降级路径）；延迟则 Windows beta 先行 |
| Helper/XPC 归属错误 | 漏掉或错算行为 | 签名/Bundle/父子进程多信号归属（FR-1.5）；Processes 页展示归属依据供核对 |
| 网络域名无法完整解析 | 只能看到 IP 或 hostname 不完整 | 诚实展示可得字段（FR-4.3）；DNS/flow 关联增强列为 P1（FR-4.4） |
| 事件量大影响性能 | 目标 App 或系统变慢 | 内核层尽早过滤、用户层批处理、ring buffer（NFR-1.2） |
| 用户把时序当因果 | 误判隐私行为 | 统一 UI 文案标注（FR-5.9） |
| 免费项目维护成本 | 拖累收费产品开发 | 严格 P0 边界；不做 blocking / IDS / malware DB / HTTPS decrypt |

### 外部依赖

- Apple Endpoint Security entitlement 审批（macOS 硬门槛）
- Apple Developer ID 签名与公证服务
- 开源传播渠道：GitHub / HN / Reddit（品牌目标，非功能依赖）

---

## 12. 附录：官方技术参考

- Apple — Monitoring System Events with Endpoint Security：<https://developer.apple.com/documentation/endpointsecurity/monitoring-system-events-with-endpoint-security>
- Apple — System Extensions：<https://developer.apple.com/documentation/systemextensions>
- Apple — Network Extension Entitlement：<https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.networking.networkextension>
- Apple — TN3134 Network Extension provider deployment：<https://developer.apple.com/documentation/technotes/tn3134-network-extension-provider-deployment>
- Apple — System Extensions（entitlement 申请入口与 SIP-off 测试说明）：<https://developer.apple.com/system-extensions/>
- Microsoft — ETW FileIo：<https://learn.microsoft.com/windows/win32/etw/fileio>
- Microsoft — Windows Filtering Platform：<https://learn.microsoft.com/windows/win32/fwp/about-windows-filtering-platform>
