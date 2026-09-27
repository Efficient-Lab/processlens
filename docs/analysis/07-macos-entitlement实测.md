# ProcessLens macOS 采集路径实测记录

| 项目 | 内容 |
|------|------|
| 日期 | 2026-09-25 |
| 环境 | macOS 27.0 (26A428)，Xcode 27.0，Apple Development 签名证书（Team RHTT62U954） |
| 目的 | 验证需求稿 §7.1 技术路线的关键假设：ES 事件语义、entitlement 门槛、免审批降级路径 |
| 对应分析 | `02-desktop-app-engineer.md` TR-01/02/03、PM-R04；`06-senior-project-manager.md` PM-R02 |

---

## 实测结果汇总

| # | 验证项 | 方法 | 结果 | 结论 |
|---|--------|------|------|------|
| 1 | ES 客户端能否在无 Apple 批准 entitlement 的情况下本地运行 | hello world + `com.apple.developer.endpoint-security.client` entitlement，开发证书签名 | **exec 即被 SIGKILL**（exit 137，零输出）；同一二进制去掉 entitlement 后正常输出 | entitlement 约束在 exec 时强制，需 Apple 签发的 provisioning profile；加 `get-task-allow` 无效（**适用范围：SIP 开启环境**；SIP 关闭后可运行，见文末追加调查） |
| 2 | ES 事件语义（READ/WRITE/path/bytes） | — | **无法实测**：ES 客户端根本无法启动（见 #1） | TR-01/02 维持"待 entitlement 获批后实测"；Apple 官方文档口径不变 |
| 3 | 免 entitlement 的进程网络元数据 | `proc_pidfdinfo` 枚举目标进程 socket fd | **可用**：拿到 `10.0.0.16:49683 → 151.101.1.69:443`、TCP state、IPv4/6 | macOS 免审批可枚举每进程连接端点（快照式） |
| 4 | 免 entitlement 的每进程流量字节数 | `nettop -p <pid>`（nstat 数据源） | **可用**：bytes_in=21 / bytes_out=37，按连接分列 | macOS 免审批可拿到每进程/每连接流量统计 |
| 5 | 免 entitlement 的打开文件枚举 | `proc_pidfdinfo` + `PROC_PIDFDVNODEPATHINFO` | **可用**：枚举到 `/private/etc/hosts`、目标写入文件、`~/Library/Safari/Bookmarks.plist` 路径 | 只能枚举"当前打开"的 fd 快照；无 READ/WRITE 事件区分、无 CREATE/DELETE/RENAME 事件流 |
| 6 | 免 entitlement 的文件事件流 | FSEvents 路径（评估） | 目录级变化、无进程归属、无 READ 事件 | 不满足 FR-3.x 语义 |

---

## 关键结论

### 1. macOS 的 ES entitlement 是 exec 级硬门槛（验证 PM-R04/TR-03）

`endpoint-security.client` 二进制在 entitlement 获批前**连进程都启动不了**——不是 `es_new_client` 返回错误，而是内核在 exec 阶段 SIGKILL。这意味着：

- macOS 侧 G1 PoC 在 SIP 开启的机器上**必须等 entitlement 获批**，无开发签名降级路径；审批期间可改用关闭 SIP 的测试机联调（见文末 2026-09-27 追加调查）。
- G0 的排期预估不能假设"申请期间在 SIP 开启环境先做着"，但 SIP-off 测试机是官方认可的联调路径，macOS 采集层代码不必等到获批才动笔。
- "Windows beta 先行"仍是审批延误时的产品兜底；开发层面有 SIP-off 路径兜底，PM-R04 的"获批前可行性路径"已有答案。

### 2. 免 entitlement 的 macOS 降级路径只能做"快照"，验证企划书 §19 的否决正确

不用 entitlement 能拿到的最好组合是：

- 连接端点 + 每连接流量：`proc_pidfdinfo` + `nettop` 数据源（快照式，轮询）
- 打开文件路径：`proc_pidfdinfo` vnode 枚举（快照式）

**拿不到**：文件 READ/WRITE/CREATE/DELETE/RENAME 事件流、事件级进程归属的时间线。这恰好是企划书 §19 说的"前后快照工具"形态——本次实测证明免 entitlement 路线只能做到这个程度，维持"不退化"的 Go/No-Go 结论。

### 3. 免 entitlement 路径仍有用途：启动基线快照

`proc_pidfdinfo` 枚举（实测 #3/#5 可用）正是桌面工程师提出的 **GAP-02 Session 启动基线快照**的实现手段——Start 时对既有进程树做 socket/fd 快照，弥补"中途 attach 看不到既有连接"（TR-08）。这个能力无需任何审批，可作为正式方案的一部分保留。

### 4. TR-09（无 FDA 事件盲区）无法本轮验证

ES 客户端起不来，"无 FDA 时受保护目录事件是否下发"无法实测，维持待验证。但旁证：枚举路径能看到 `~/Library/Safari/`（TCC 保护目录）说明当前终端上下文有 FDA 或 Safari 路径本身可读；ES 事件投递是否依赖 FDA 仍需获批后实测。

---

## 对需求稿的回写建议

| 条目 | 建议 |
|------|------|
| §1.4 Go/No-Go | 补一句实测结论："entitlement 未获批时 ES 客户端无法启动（exec 即被终止），macOS 采集层零降级路径" |
| §10 G0 | 输出物增加："确认 NE content-filter entitlement 是否需审批"（本次未验证，Apple 文档显示为受限值）；"确认获批前是否有开发联调路径（本次实测：无）" |
| §10 G1 | macOS 侧 PoC 明确依赖 G0 完成；Windows 侧可独立先行 |
| FR-2.x / GAP-02 | 新增"Session 启动基线快照"：Start 时用 `proc_pidfdinfo`（macOS）/ `GetExtendedTcpTable`（Windows）枚举既有连接与打开文件，免 entitlement 即可实现 |
| §11 风险表 | "entitlement 未获批"行的控制措施补实测证据：不是"审批慢则晚点发"，而是"未获批期间 macOS 一行采集代码都跑不了" |

## 附：测试产物

临时代码与日志已按规则清理（`$TMPDIR/pltest/`）。关键命令：

```bash
# ES entitlement 二进制 exec 即被杀的复现
clang -o hello hello.c
codesign -s "Apple Development: ..." --entitlements es_test.entitlements -f hello
./hello   # → SIGKILL (exit 137)，零输出

# 免 entitlement 的进程 socket 枚举（可用）
proc_pidfdinfo(pid, fd, PROC_PIDFDSOCKETINFO, ...)

# 免 entitlement 的每进程流量（可用）
nettop -p <pid> -l 1 -x -J bytes_in,bytes_out
```

---

## 追加调查：NE entitlement 与 SIP-off 联调路径（2026-09-27）

基于 Apple 官方文档联网核实，补充/更正三条结论。

### A. NE content-filter entitlement 无需 Apple 审批（解决 TR-03 / PM-R02）

`com.apple.developer.networking.networkextension` 自 2016 年起为自助 capability：付费开发者账号在 Certificates, Identifiers & Profiles 给 App ID 启用 "Network Extensions" 即可，不走 managed entitlement 申请表。此前分析文档（`02-desktop-app-engineer.md` TR-03、`06-senior-project-manager.md` PM-R02、`00-综合分析报告.md`）假设的"NE 同为受限审批、Go/No-Go 双门槛"**不成立**——需要 Apple 审批的只有 ES entitlement 一项。

Developer ID 分发侧的注意点：

- entitlement 值必须用 `-systemextension` 后缀变体：`content-filter-provider-systemextension`；Xcode Signing & Capabilities UI 只写入不带后缀的值，需手动改 `.entitlements`
- Xcode 自动签名与 Organizer 的 Developer ID 导出对该后缀支持不可靠，需手动下载 Developer ID provisioning profile 并手动签名（可写脚本固化）
- `com.apple.developer.system-extension.install`（宿主 App 安装系统扩展）为自助 entitlement，无需审批
- ES entitlement 仅能用于 Developer ID 分发，Mac App Store 不支持；ES 审批常先只批 development，Developer ID 分发授权可能需单独跟进，申请时应注明分发意图
- Full Disk Access 与 System Extension 激活是 TCC 用户授权，不是 entitlement 申请项

### B. SIP 关闭后 ES client 可运行：获批前存在官方认可的联调路径

Apple System Extensions 页面明示：entitlement 审批期间可临时禁用 SIP 进行测试。`csrutil disable` 后 entitlement 强制检查不再执行，无 entitlement 的 ES client 可正常启动。即：

- 实测 #1/#2 的结论只在 SIP 开启环境成立；SIP-off 测试机上 TR-01/02 事件语义验证可提前进行
- G1 macOS PoC 不必干等审批，备一台 SIP-off 测试机/测试分区即可先行联调；正式分发仍须获批，Go/No-Go 结论不变
- Apple Silicon 关闭 SIP 需在 recoveryOS 选"降低安全性"；建议专用测试环境，不在日常开发机上操作

### C. 回写状态

上述结论已回写需求文档 v1.1（§1.4、FR-2.6、§7.1、§10 G0、§11 风险表、§12 参考链接）。前文"回写建议"表中 §10 G0 行的两个待确认项（NE 是否需审批、获批前是否有联调路径）由此节定论。

### 来源（抓取日期 2026-09-27）

- [System Extensions — Apple](https://developer.apple.com/system-extensions/)：entitlement 申请入口 + SIP-off 测试说明
- [Network Extensions Entitlement — Apple](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.networking.networkextension)：`-systemextension` 后缀值
- [TN3134: Network Extension provider deployment](https://developer.apple.com/documentation/technotes/tn3134-network-extension-provider-deployment)：content filter 部署形态
- [Endpoint Security Entitlement — Apple](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.endpoint-security.client)：Developer ID 分发限定
