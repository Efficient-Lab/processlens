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
| 1 | ES 客户端能否在无 Apple 批准 entitlement 的情况下本地运行 | hello world + `com.apple.developer.endpoint-security.client` entitlement，开发证书签名 | **exec 即被 SIGKILL**（exit 137，零输出）；同一二进制去掉 entitlement 后正常输出 | entitlement 约束在 exec 时强制，需 Apple 签发的 provisioning profile；加 `get-task-allow` 无效 |
| 2 | ES 事件语义（READ/WRITE/path/bytes） | — | **无法实测**：ES 客户端根本无法启动（见 #1） | TR-01/02 维持"待 entitlement 获批后实测"；Apple 官方文档口径不变 |
| 3 | 免 entitlement 的进程网络元数据 | `proc_pidfdinfo` 枚举目标进程 socket fd | **可用**：拿到 `10.0.0.16:49683 → 151.101.1.69:443`、TCP state、IPv4/6 | macOS 免审批可枚举每进程连接端点（快照式） |
| 4 | 免 entitlement 的每进程流量字节数 | `nettop -p <pid>`（nstat 数据源） | **可用**：bytes_in=21 / bytes_out=37，按连接分列 | macOS 免审批可拿到每进程/每连接流量统计 |
| 5 | 免 entitlement 的打开文件枚举 | `proc_pidfdinfo` + `PROC_PIDFDVNODEPATHINFO` | **可用**：枚举到 `/private/etc/hosts`、目标写入文件、`~/Library/Safari/Bookmarks.plist` 路径 | 只能枚举"当前打开"的 fd 快照；无 READ/WRITE 事件区分、无 CREATE/DELETE/RENAME 事件流 |
| 6 | 免 entitlement 的文件事件流 | FSEvents 路径（评估） | 目录级变化、无进程归属、无 READ 事件 | 不满足 FR-3.x 语义 |

---

## 关键结论

### 1. macOS 的 ES entitlement 是 exec 级硬门槛（验证 PM-R04/TR-03）

`endpoint-security.client` 二进制在 entitlement 获批前**连进程都启动不了**——不是 `es_new_client` 返回错误，而是内核在 exec 阶段 SIGKILL。这意味着：

- macOS 侧 G1 PoC **必须等 entitlement 获批**，或在批准前用完全不同的技术路径（无）。无开发签名降级路径。
- G0 的排期预估不能假设"申请期间先做着"，macOS 采集层代码只能在获批后编写联调（或先在虚拟机/另一台获批机器上）。
- "Windows beta 先行"从兜底方案变成**macOS 获批前的唯一可执行路径**，PM-R03 的优先级上调。

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
