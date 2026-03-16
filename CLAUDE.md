# ClaudeUsageBar

macOS 菜单栏应用，用于监控 Claude.ai 用量。支持双账号同时监控。

## 构建

```bash
cd app && ./build.sh
```

构建产物：`app/build/ClaudeUsageBar.app`（Universal Binary: arm64 + x86_64）

构建需要 macOS + `swiftc`。脚本会分别编译 arm64 和 x86_64，用 `lipo` 合并为 Universal Binary 并签名。

## 项目结构

```
app/                        # macOS 应用
  ClaudeUsageBar.swift      # 单文件应用：AppDelegate, UsageManager, UsageView
  Info.plist                # Bundle 配置 (com.claude.usagebar, macOS 12.0+, LSUIElement)
  build.sh                  # 构建脚本 → Universal Binary (arm64 + x86_64)
  create_dmg.sh             # DMG 安装包生成
website/                    # 落地页 & SEO 博客（纯静态 HTML/CSS/JS）
  index.html                # 主页
  blog/                     # SEO 博客文章
```

## 架构

**MVVM + ObservableObject 模式**，所有逻辑在 `app/ClaudeUsageBar.swift` 中：

### 核心类

- **AppDelegate** — 菜单栏图标、弹窗管理、全局快捷键 (⌘U)，图标颜色编码（绿 <70%、黄 70-90%、红 >90%）
- **AccountData** — 单个账号的数据模型（ObservableObject），持有 session/weekly/sonnet 用量
- **UsageManager** — 全局数据管理器，持有 account1 + account2，负责网络请求和通知（每 5 分钟自动刷新）
- **AccountUsageView** — 单账号用量显示子视图（进度条 + 重置时间）
- **CookieInputSection** — 单账号 Cookie 输入子视图
- **UsageView** — 主弹窗 SwiftUI 视图，组合两个账号的用量和设置
- **CustomTextField / PasteableTextField** — NSViewRepresentable 桥接，解决弹窗中粘贴板支持问题

### 框架

- SwiftUI — 弹窗 UI
- AppKit — NSStatusItem、NSPopover、NSUserNotification
- Carbon — 全局快捷键注册
- Combine — objectWillChange 转发（嵌套 ObservableObject）

### 数据存储（UserDefaults）

| Key | 说明 |
|-----|------|
| `claude_session_cookie_1` / `_2` | 账号 1/2 的完整 Cookie |
| `last_notified_threshold_1` / `_2` | 各账号上次通知的阈值 |
| `notifications_enabled` | 全局通知开关 |
| `open_at_login` | 登录时启动 |
| `shortcut_enabled` | ⌘U 快捷键开关 |

旧版单账号 key `claude_session_cookie` 和 `last_notified_threshold` 会在首次启动时自动迁移到 `_1`。

### API 端点

- `https://claude.ai/api/bootstrap` — 获取 organizationId
- `https://claude.ai/api/organizations/{orgId}/usage` — 获取用量数据

### 响应格式

```json
{
  "five_hour": { "utilization": 45.5, "resets_at": "..." },
  "seven_day": { "utilization": 62.3, "resets_at": "..." },
  "seven_day_sonnet": { "utilization": 38.1, "resets_at": "..." }
}
```

`seven_day_sonnet` 仅 Pro 套餐存在，用于自动检测 Pro/Free。

## 关键技术细节

- 认证：使用 claude.ai 的 Session Cookie，从 Cookie 中 `lastActiveOrg` 提取 org ID
- 存储：仅 UserDefaults（Cookie、设置、通知状态）
- 网络：URLSession → `https://claude.ai/api/organizations/{orgId}/usage`
- 通知：NSUserNotification，可配置阈值
- ATS：Info.plist 中允许 anthropic.com 域名
- 无 Dock 图标：LSUIElement = true

## 约定

- 所有代码集中在单文件 `app/ClaudeUsageBar.swift`
- 无外部依赖，无 Xcode 项目，直接用 `swiftc` 编译
- 目标平台 macOS 12.0+ (Monterey)
- Bundle ID: `com.claude.usagebar`
- 使用 `NSUserNotification`（已废弃但免签名可用）
- 通知阈值：25%、50%、75%、90%
- 网站为纯静态 HTML + 内联 CSS，无构建步骤
- MIT 许可证
