# macOS 能力审批口径（2026-09-27 核实）

ProcessLens macOS 侧能力分三类：

## 需 Apple 审批（managed entitlement，仅一项）

- `com.apple.developer.endpoint-security.client`：经 <https://developer.apple.com/contact/request/system-extension/> 申请（2026-09-25 已提交）。
- 审批常先只批 development，Developer ID 分发授权可能需单独跟进；申请时已注明分发意图。
- ES 仅支持 Developer ID 分发，Mac App Store 不可用。
- **审批期间联调路径**：Apple 明示可临时禁用 SIP 测试；SIP-off 环境下无 entitlement 的 ES client 可运行。SIP 开启时 exec 即被 SIGKILL（2026-09-25 实测）。

## 自助开启（无需审批）

- `com.apple.developer.networking.networkextension`：2016 年起自助。Developer ID 分发必须用 `-systemextension` 后缀值（如 `content-filter-provider-systemextension`），Xcode UI 只写不带后缀的值 → 需手动改 `.entitlements` + 手动签名（自动签名/Organizer 导出不可靠）。
- `com.apple.developer.system-extension.install`：宿主 App 安装系统扩展用，自助。

## 用户授权（非 entitlement）

- Full Disk Access（TCC）、System Extension 激活（系统设置批准）、公证（Developer ID + notarytool）。

## 历史口径更正

早期分析文档（`analysis/00`/`02`/`06`）曾假设 NE content-filter 也需 Apple 审批、"Go/No-Go 双门槛"——已证伪，以本文与 `analysis/07` 追加调查、PRD v1.1 为准。

来源：`docs/analysis/07-macos-entitlement实测.md` 追加调查节（含官方链接与抓取日期）。
