# ProcessLens 安全架构综合分析

> 视角：Security Architect | 日期：2026-09-24
> 主依据：`docs/ProcessLens_需求文档.md`（PRD Draft v1.0），上游：`docs/ProcessLens_单App行为监控器_项目企划书.txt`
> 严重度分级：Critical / High / Medium / Low / Informational

---

## 总体结论

PRD 在**产品层隐私边界**上做得很扎实：只观察不拦截（§1.3、§8）、不抓 payload（FR-4.3、NFR-2.1）、默认无遥测（NFR-2.2）、不篡时序为因果（FR-5.9）、导出前敏感信息提示（FR-5.10）。这些是产品定位的强项，应予保留。

但 **ProcessLens 自身是本机上权限最高的组件之一**（macOS：Endpoint Security + Full Disk Access + Network Extension 系统扩展；Windows：SYSTEM 级特权服务 + WFP/ETW），PRD 对"采集器自身被攻击/被滥用"这一面几乎没有设防：

1. **IPC 信任边界完全未定义**。FR-6.4 只写了"GUI 通过 IPC 获取筛选后的 Session 数据"，没有认证、没有接口面限制、没有输入校验。Windows 上这意味着任何本地进程若能连上服务管道，就拿到了一个 SYSTEM 级数据出口甚至控制能力——这是特权服务最经典的提权漏洞类别。
2. **"不抓 payload"目前是承诺而非架构保证**。FR-6.2 选定的 Network Extension Content Filter 机制在技术上**有能力**接收 flow 数据；只有强制约定"永不返回 filterDataVerdict、永不订阅 socket data"，承诺才成立。同理，ES 扩展持有 FDA，技术上可以读文件内容——需要把"不读"变成可验证的结构性约束（最小接口、不开内容读取路径、采集器开源可审计）。
3. **Session 数据本身是高价值目标**。SQLite 里存的是敏感路径访问记录与域名/IP 元数据（FR-2.2、§6），泄露即等于泄露"用户在监控谁、被监控 App 的行为画像"，但 PRD 没有文件权限、加密、保留/删除策略的任何要求。
4. **攻击者可控字符串贯穿渲染与导出链路**。被监控 App 是潜在对手：它可以构造带 HTML/JS 的文件名、注册恶意域名、写入 `=cmd|...` 式路径。这些数据会进入 Tauri WebView（存储型 XSS → 调用 Tauri command）和 CSV/Markdown 导出（公式注入）。PRD 全篇没有"外部数据一律不可信"的处理要求。
5. **没有更新与漏洞响应机制**。G4 只覆盖签名/公证/安装/卸载。一个内核级组件如果没有签名自动更新通道和漏洞披露流程，任何发布后发现的漏洞都意味着长期不可修复的特权驻留——这与其"安全工具"的品牌定位直接冲突。

**结论：PRD 可进入下一阶段，但必须新增一组"采集器自身安全"需求（IPC 认证、存储保护、渲染/导出消毒、观察边界可验证、更新与响应机制），否则 T-01/T-02/T-04/T-06 任一条落地都会击穿产品的核心信任卖点。**

---

## 威胁与风险清单

| 编号 | 严重度 | 威胁场景 | 当前需求覆盖情况 | 建议新增需求 |
|------|--------|----------|------------------|--------------|
| T-01 | **Critical** | IPC 未认证：本地恶意进程连接 Windows 特权服务管道 / macOS sysex XPC，窃取 Session 数据、篡改监控目标（把 ProcessLens 变成针对第三方 App 的本地监视工具），甚至利用消息处理缺陷在 SYSTEM/root 上下文执行代码（LPE） | FR-6.4 仅声明 IPC 存在，无认证/鉴权/接口面要求 | SEC-FR-1/2/3（IPC 双向认证、最小接口面、消息校验） |
| T-02 | **High** | 被监控 App 构造恶意文件名/域名（含 `<script>`、事件属性等），经 SQLite 进入 Tauri WebView 渲染 → 存储型 XSS → 调用 Tauri command（fs/shell scope）执行任意操作；若 GUI 进程也持有 TCC 授权则危害放大 | 无。FR-5 全章未提渲染安全 | SEC-FR-6（不可信数据处理 + CSP + allowlist 最小化）+ SEC-FR-8（FDA 不授予 GUI） |
| T-03 | **High** | "不抓 payload"无架构保证：NE Content Filter 机制可接收 flow 数据；ES 扩展持 FDA 可读任意文件。承诺依赖开发者自觉，一旦实现偏差或代码变更即静默破坏产品底线 | FR-4.3/NFR-2.1 是行为声明，无可验证约束 | SEC-FR-7（观察边界结构性约束 + 可审计性） |
| T-04 | **High** | Session SQLite 明文落盘（路径含用户名、`~/.ssh` 访问记录、域名画像），同用户任意进程可读；多用户机器上其他账户也可能读取；无删除/保留策略 | FR-2.2 仅"本地存储"；FR-6.6 卸载时可选删除 | SEC-FR-4（0600 权限、可选加密、删除/保留策略） |
| T-05 | **High** | 无更新机制：系统扩展/特权服务的漏洞无法触达存量用户；entitlement 组件驻留期长，攻击窗口随时间扩大 | G4 仅签名/公证/安装/卸载 | SEC-FR-10（签名自动更新 + 回滚保护） |
| T-06 | **High** | Windows 服务经典攻击面叠加：命名管道 ACL 过宽、未加引号服务路径、DLL 劫持、WFP 若走 callout driver 则需内核驱动签名/attestation（计划外成本） | NFR-3.3 仅"最小必要权限"，无具体攻击面要求 | SEC-FR-8（服务加固清单；优先用户态 WFP 事件订阅规避内核驱动） |
| T-07 | **Medium** | 事件完整性静默降级：被监控 App 恶意泛洪事件打爆 ring buffer（NFR-1.2），报告"看起来完整"实则丢数据，用户据此下错误结论 | NFR-1.2/1.3 从性能角度覆盖，未要求披露丢失 | SEC-FR-9（丢失计数必须进报告） |
| T-08 | **Medium** | 归属欺骗：恶意目标 spawn 间接子进程/利用 daemon 代执行逃逸归属（漏报）；或其他进程伪装签名/Bundle 混入 process tree（错报，嫁祸） | FR-1.5 多信号归属是功能设计，无对抗性场景要求 | 补 FR-1.5 子条目：定义归属置信度与对抗性测试用例 |
| T-09 | **Medium** | 导出报告被当作"证据"传播：JSON/CSV 可手工伪造后冠以 ProcessLens 名义指控第三方 App；FR-5.9 的措辞约束无法防止文件被篡改 | FR-5.9/5.10 部分覆盖 | SEC-FR-5 附加：导出文件嵌入"未经签名、可被修改"声明；P1 可评估报告签名 |
| T-10 | **Medium** | 供应链：Rust/Tauri/Swift 依赖 CVE、GitHub Release 被替换或 typosquat——分发内核级工具的项目是供应链攻击高价值目标 | NFR-3.2 签名公证覆盖安装包，无依赖审计/发布验证 | SEC-NFR-1（SBOM + cargo audit CI gate + Release 校验和与验证文档） |
| T-11 | **Medium** | 无漏洞响应流程：无 SECURITY.md/披露渠道/修复 SLA；安全研究者发现问题无处可报，漏洞流向公开 issue 或黑市 | 完全缺失 | SEC-FR-11（漏洞披露政策 + 响应流程） |
| T-12 | **Low** | 采集器自身被攻陷的 blast radius：ES 扩展可见全系统事件，攻击者拿到它即拿到全机文件/网络观测能力。纵深手段（组件分离、最小 entitlement、开源审计）均未要求 | NFR-1.2"只持久化目标事件"是性能条款而非安全条款 | SEC-FR-7/8 + 将企划书 §14"开源采集器"提升为正式安全需求 |
| T-13 | **Informational** | 日志/崩溃报告泄漏：调试日志或 crash dump 若含 Session 数据且被第三方工具收集，绕过"无遥测"承诺 | NFR-2.2 覆盖遥测，未覆盖崩溃报告 | 纳入 SEC-FR-4/11：崩溃上报 opt-in 且脱敏 |

---

## 安全需求补充建议

以下条目可直接并入 PRD（建议新增 **FR-7 安全与信任边界** 模块，或拆分进现有 FR/NFR）。

### FR-7 安全与信任边界（新增模块）

| 编号 | 需求 | 优先级 |
|------|------|--------|
| SEC-FR-1 | **IPC 双向认证**：macOS sysex XPC 必须以 audit token + code signing requirement 校验客户端为同一 Developer ID 签名的 GUI；Windows 服务管道/ALPC 必须设置 ACL 限定本机用户并校验对端进程签名身份。未认证连接一律拒绝 | P0 |
| SEC-FR-2 | **IPC 消息校验**：协议版本化；所有消息经 schema 校验，字段长度/数量设上限；默认拒绝未知消息类型；协议层纳入 fuzz 测试 | P0 |
| SEC-FR-3 | **IPC 最小接口面**：采集端仅暴露「白名单控制指令 + 筛选后 Session 数据」两类接口；不得通过 IPC 暴露系统级原始事件流或非目标进程数据 | P0 |
| SEC-FR-4 | **Session 存储保护**：SQLite 文件权限 0600（仅当前用户）；提供单 Session 删除与"退出时清理"选项；P1 评估 SQLCipher/系统 Keychain 托管密钥的静态加密；崩溃上报须 opt-in 且不含路径/域名明细 | P0 |
| SEC-FR-5 | **导出消毒**：CSV 导出对以 `=` `+` `-` `@` 开头的单元格强制前缀转义（防公式注入）；Markdown/JSON 导出对路径、域名等外部来源字符串做转义；导出文件名做安全化处理；导出文件头部注明"本文件未经签名，内容可被修改" | P0 |
| SEC-FR-6 | **渲染安全**：路径、域名、进程名等全部外部来源字符串按不可信数据处理（强制转义/文本节点渲染）；WebView 配置严格 CSP、禁止加载远程内容；Tauri allowlist 最小化，fs/shell 等 scope 默认关闭或收窄到应用目录 | P0 |
| SEC-FR-7 | **"只观察"结构性约束**：(a) NE Content Filter 永不返回 `filterDataVerdict`、不订阅 socket/packet 数据，使 payload 在机制上不可达；(b) 采集层不调用任何文件内容读取 API，代码评审与 CI 静态检查强制执行；(c) 非目标进程事件在扩展/内核层尽早 mute，不进入缓冲与持久化；(d) 采集器/系统扩展源码开源或提供可审计构建（将企划书 §14 建议落地为需求） | P0 |
| SEC-FR-8 | **权限最小化**：Full Disk Access 仅授予系统扩展，GUI 进程不得持有；entitlement 仅申请功能必需值；Windows 服务以最小权限账户运行、服务路径加引号、安装目录 ACL 防篡改；优先用户态 WFP ALE 事件订阅，若确需 callout driver 则必须规划驱动签名/attestation | P0 |
| SEC-FR-9 | **完整性披露**：ring buffer 溢出、采集异常导致的事件丢失必须计数并在报告中显式标注（如"本 Session 丢弃 N 条事件"），不得静默 | P0 |
| SEC-FR-10 | **安全更新机制**：提供签名的自动更新通道（macOS Sparkle+EdDSA / Windows 签名更新包），仅经 HTTPS 拉取，防版本回滚；系统扩展升级路径（版本迁移、用户重新授权提示）纳入测试 | P1 |
| SEC-FR-11 | **漏洞响应**：仓库提供 SECURITY.md 与安全联系渠道；定义严重漏洞修复 SLA 与紧急停用/最低版本提示机制 | P1 |

### NFR 补充

| 编号 | 需求 | 优先级 |
|------|------|--------|
| SEC-NFR-1 | 供应链安全：CI 集成 cargo audit / 依赖漏洞扫描与 secrets 扫描并设为门禁；生成 SBOM；Release 提供签名校验和与验证指引 | P0 |
| SEC-NFR-2 | 安全测试纳入验收：IPC fuzz、WebView 注入用例、CSV 公式注入用例、权限最小化核查进入 G4 Gate | P0 |

---

## 需要修改的条目

| 条目 | 现状 | 修改建议 |
|------|------|----------|
| FR-6.4 | "GUI 通过 IPC 获取筛选后的 Session 数据" | 追加双向认证与最小接口面要求（指向 SEC-FR-1/2/3） |
| FR-6.1 | 引导授予 FDA | 明确 FDA 授予对象为系统扩展而非 GUI 主进程（SEC-FR-8） |
| FR-2.2 | "全量本地存储（SQLite）" | 追加存储保护：文件权限、删除策略、可选加密（SEC-FR-4） |
| FR-4.3 / NFR-2.1 | "不抓 payload"为行为声明 | 改写为结构性约束：filter verdict 配置 + 无内容读取路径（SEC-FR-7），使其可测试 |
| FR-5.7 / FR-5.10 | 导出格式与提示 | 追加导出消毒与"文件可修改"声明（SEC-FR-5） |
| NFR-1.2 / NFR-1.3 | ring buffer 性能条款 | 追加事件丢失披露义务（SEC-FR-9），性能条款同时是完整性条款 |
| FR-6.6 | 干净卸载 | 追加：卸载须覆盖系统扩展停用/移除的完整路径与残留审计 |
| §8 明确不做 | 未涉及自身接口 | 追加一行："不向任何第三方进程开放 IPC 接口 / 不提供系统级原始事件流导出" |
| §11 风险表 | 无安全自身风险行 | 新增行：IPC 未认证本地攻击（SEC-FR-1）、采集器被攻陷 blast radius（SEC-FR-7/8）、无更新机制导致漏洞驻留（SEC-FR-10） |
| §9 验收标准 | 无安全验收 | 新增 AC："构造恶意文件名/域名的测试 App 不能破坏报告渲染与导出安全"；"非授权进程无法连接采集端 IPC"；"采集层无文件内容读取调用（静态检查）" |
| §10 G4 Gate | 签名/公证/安装/卸载/崩溃恢复/性能 | 追加：自动更新通道联调 + SEC-NFR-2 安全测试清单通过 |

---

## 证据清单

### 已读文件
- `docs/ProcessLens_需求文档.md`：PRD 全文，逐条核对 FR-1~FR-6、NFR-1~4、§8 排除项、§9~11 验收与风险
- `docs/ProcessLens_单App行为监控器_项目企划书.txt`：上游企划，§9/§10 技术方案、§13 隐私边界、§14 开源建议

### 关键定位
- `ProcessLens_需求文档.md` FR-6.4：IPC 仅一句功能描述，无安全要求 → T-01
- `ProcessLens_需求文档.md` FR-6.2 + §7.1：Content Filter 机制可触达 payload → T-03
- `ProcessLens_需求文档.md` FR-2.2 + §6：SQLite 明文存敏感路径/域名 → T-04
- `ProcessLens_需求文档.md` §10 G4：无更新机制 → T-05
- `ProcessLens_企划书.txt` §14：建议开源采集器（应提升为需求）→ SEC-FR-7(d)
