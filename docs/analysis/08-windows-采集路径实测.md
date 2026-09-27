# ProcessLens Windows 采集路径实测记录

| 项目 | 内容 |
|------|------|
| 日期 | 2026-09-27 |
| 环境 | Windows 11（10.0.26200），Windows SDK 10.0.26100.0，WPT/xperf，Rust 1.95.0，Node 24，WebView2 已装；采集测试经 UAC 提权执行 |
| 目的 | 验证需求稿 §7.2 Windows 技术路线：ETW FileIo 文件事件、WFP 网络元数据、字节数与 hostname 来源、权限模型 |
| 对应分析 | `02-desktop-app-engineer.md` TR-04/05/06/07/14、GAP-02；`01-product-manager.md` PM-09/PM-11 |

---

## 实测结果汇总

| # | 验证项 | 方法 | 结果 | 结论 |
|---|--------|------|------|------|
| 1 | ETW Kernel-File 会话能否启动采集 | `logman create trace -p Microsoft-Windows-Kernel-File`（提权） | **可用**：2 秒窗口捕获 46,180 条全系统文件事件，`EventsLost=0` | 采集通道成立；非管理员启动被拒（"拒绝访问"），证实特权服务是硬依赖 |
| 2 | FileIo 事件是否含路径 | 写/读/删探针文件，tracerpt 解 ETL | **部分含路径**：Create/Name/Rename 类事件直接携带 `\Device\HarddiskVolume4\...` 全路径；Read/Write 类（占比最大，23,058 条）只有 FileObject/FileKey 指针 + `IOSize`/`ByteOffset`，**无路径** | 文档论断属实：FileObject→path 映射管线是文件侧主体工程 |
| 3 | FileIo 事件归属字段 | 同一 ETL | **`Execution.ProcessID` 携带进程 PID**，探针事件正确归属发起进程 | PID 级归属可行 |
| 4 | FileIo 操作覆盖面 | EventID/Task 分布统计 | Create、Read、Write、Delete(SetInfo/Cleanup 族）、Rename、Close 全覆盖 | FR-3.1 操作集可满足，语义映射表需按 EventID 建 |
| 5 | WFP net events 内容 | `netsh wfp show netevents` + SDK `fwpmtypes.h` 头文件核对 | 事件头含端点四元组 + `appId`（`\device\harddiskvolume4\...\svchost.exe` 可执行路径）+ `userId`；**无 PID、无 bytes**；导出 300 条均为 DROP/CAPABILITY 类 | TR-14 可关闭：无 processId；`appId` 正好是 FR-1.x 需要的 App Identity |
| 6 | 用户态 `FwpmNetEventSubscribe*` 能订到什么 | `fwpmtypes.h`/`fwpmu.h` 枚举核对 | 订阅面只暴露 CLASSIFY_DROP/CAPABILITY 等事件；ALE 授权成功事件（ALE_AUTH_*）只走内核 callout，用户态订阅拿不到 | **"WFP ALE 采集连接元数据"在用户态不成立**；成功连接流要靠 ETW Kernel-Network |
| 7 | ETW Kernel-Network 内容 | 提权采集 + TCP socket 探针 | 每事件携带 **`PID + saddr/sport/daddr/dport + size + connid`**，Connect/Accept/Disconnect/Reconnect 生命周期齐全 | FR-4.1/4.2 的端点、PID 归属、bytes 三需求**单一数据源即满足，无需内核驱动** |
| 8 | 既有连接基线快照 | `Get-NetTCPConnection`（GetExtendedTcpTable 等价物，免管理员） | 返回 Established 连接 + `OwningProcess` PID | GAP-02 基线快照可行（TR-08 对应能力） |
| 9 | hostname 关联 | `Get-DnsClientCache` + DNS-Client 通道 | 缓存免管理员可查；`Microsoft-Windows-DNS-Client/Operational` 默认关闭，启用需管理员 | FR-4.4 升 P0 的建议成立，实现路径明确 |
| 10 | App Identity 信号 | `Get-AuthenticodeSignature` 抽样 | signed/unsigned 正确区分（系统程序 Valid，cargo.exe NotSigned） | TR-13 分层 fallback（签名>路径+哈希）可行 |
| 11 | Rust 生态与构建链 | cargo search / 本地 registry | `ferrisetw`/`ferrisetw2`、`windows` crate 存在；crates.io 直连可用 | Tauri + Rust 技术栈在 Windows 无阻塞 |

---

## 关键结论

### 1. Windows 文件侧：可行，但"路径解析"是必须正视的主体工程（验证 TR-04/TR-05）

Kernel-File 事件流在本机实测的真实结构是：

- **Create/Name/Rename 族** → 直接带全路径（内核路径，`\Device\HarddiskVolumeN\` 形式，需规范化层转盘符，对应 GAP-05）
- **Read/Write 族（占绝对大头）** → 只有 FileObject 指针 + IOSize，无路径

所以 §7.2 一句"ETW FileIo events ... 结合 process/thread 信息做归属"掩盖了两件事：FileObject→path 映射管线是必须实现的核心组件；全系统流量（2 秒 4.6 万条）只能用户态过滤，NFR-1.2 的"内核层尽早过滤"在 Windows 不成立——两处文档修改建议全部成立。

### 2. Windows 网络侧：主数据源应改为 ETW Kernel-Network，WFP 退居辅助

文档 §7.2 写"通过 WFP ALE 层采集网络元数据"，实测发现该路径有一个文档（含分析稿）未指出的结构性限制：**用户态 `FwpmNetEventSubscribe` 拿不到成功建连事件**（ALE_AUTH_* 只在内核 callout 侧产生），订阅面全是 DROP/CAPABILITY 类。如果坚持走 WFP 拿连接流，就得写 callout 驱动——引入驱动签名/attestation 成本，违背"P0 无内核驱动"的既定策略。

而 ETW Kernel-Network（实测 #7）一次给到 `PID + 端点四元组 + size + 连接生命周期`，是 P0 连接元数据的正确主力源。修正后的 Windows 网络采集架构：

- **连接流/端点/bytes/PID 归属**：ETW Kernel-Network（无驱动，提权服务内订阅）
- **App 级身份校验**：WFP `appId`（net events 里的可执行路径）辅助
- **hostname**：DNS-Client Operational 通道（需管理员启用）+ 反向查询兜底，FR-4.4 维持升 P0 结论

### 3. "Windows beta 先行"在本机获得正向证据

对照 `07-macos-entitlement实测.md`：macOS 侧 ES 客户端未获批 exec 即被杀、零降级路径；Windows 侧本次实测**无一项硬阻塞**——文件事件、网络连接元数据、PID/App 归属、流量字节、基线快照、工具链全部可用，唯一门槛是"采集服务需管理员权限"（本属 FR-6.4/NFR-3.3 架构假设，且 UAC 提权路径已实测可用）。**Windows 先行 beta 是条件齐全的正式路径**，同时支持 PM-R03 把 Windows 独立发布路径写成正式需求。

### 4. 工程量的真实分布

Windows 侧剩余不确定性集中在两处（G1 应设的实测项）：
- **FileObject→path 映射准确率**：丢事件、对象复用、rename 链下的错配率未量化（本轮只验证了通道存在，未做长时序压力测试）
- **用户态过滤成本**：4.6 万事件/2 秒的体量下，按 App 归属过滤的 CPU/内存预算需要 PoC 实测

---

## 对需求稿的回写建议

| 条目 | 建议 |
|------|------|
| §7.2 文件 I/O 行 | 补两点实测事实：①Create/Name/Rename 族带内核路径，Read/Write 只有 FileObject 指针，FileObject→path 映射管线是必需组件；②全系统流量无 PID 内核过滤，用户态过滤计入性能预算 |
| §7.2 网络行 | **改写**：主采集源为 ETW Kernel-Network（连接生命周期 + 端点 + PID + bytes，无驱动）；WFP `appId` 作为 App 级归属辅助；用户态 WFP 订阅拿不到成功建连事件（ALE_AUTH_* 仅内核 callout）这一限制显式标注 |
| §7.2 权限服务行 | 补实测证据：非管理员 `logman` 启动 ETW 会话被拒，特权服务是硬依赖而非可选项 |
| FR-4.4 | 维持升 P0；补充实现路径：DNS-Client Operational 通道（默认关闭，启用需管理员） |
| TR-14 | 关闭：SDK 头文件与 netsh 导出双向确认 net event 无 processId；归属靠 Kernel-Network 的 PID 字段 |
| TR-06 | 更新为实测结论：Kernel-Network 确认含 PID + size，bytes 走通无需 callout 驱动 |
| §10 G1 | Windows 侧验证项收敛为两个：FileObject→path 映射准确率、用户态过滤性能成本（原三项中"字段可得性"本轮已确认） |
| NFR/兼容矩阵 | 新增一行实测基线：Windows 11 build 26200 + SDK 10.0.26100 全部能力验证通过；建议最低支持 Windows 10 1809+（manifest 架构自 RS4 稳定）——待标"待多版本验证" |

## 附：测试产物

临时 ETL/XML/脚本已按规则清理（`$TEMP\pl_*`）。关键命令：

```powershell
# 文件事件采集（需管理员）
logman create trace pl_kfile -p "Microsoft-Windows-Kernel-File" 0xFFFFFFFF 5 -o out.etl -ets
# ... 触发文件 IO ...
logman stop pl_kfile -ets
tracerpt out.etl -o out.xml -of XML   # 解析：Create/Name/Rename 带 FileName；Read/Write 只有 FileObject+IOSize

# 网络连接元数据（需管理员）
logman create trace pl_knet -p "Microsoft-Windows-Kernel-Network" 0xFFFFFFFF 5 -o net.etl -ets
# 事件携带 PID/saddr/sport/daddr/dport/size/connid + Connect/Accept/Disconnect

# WFP 事件导出（需管理员）
netsh wfp show netevents file=wfp.xml   # 端点+appId+userId；无 PID/bytes；无成功建连事件

# 免管理员基线能力
Get-NetTCPConnection -State Established   # 既有连接 + OwningProcess PID
Get-DnsClientCache                        # hostname 关联
Get-AuthenticodeSignature <exe>           # App Identity 签名信号
```
