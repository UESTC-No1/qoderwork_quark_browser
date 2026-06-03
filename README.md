# QoderWork Browser Connector — 夸克浏览器适配项目

**版本：扩展 v1.4.0 / MCP Server v1.3.0** | [更新日志](CHANGELOG.md)

将 QoderWork Browser Connector 从 Chrome 迁移到夸克浏览器 PC 版，包含适配后的浏览器扩展和配套的 MCP Server。

## 项目结构

```
qoderwork_quark/
├── extension/              # 夸克版浏览器扩展（侧载安装）
│   ├── manifest.json       #   Manifest V3（已移除 Chrome Web Store 专属字段）
│   ├── background.js       #   Service Worker（quark:// 兼容 + V2 重连优化）
│   ├── popup.html / .js    #   弹出界面（去 Chrome 文案 + attachedTabs 空值保护）
│   ├── content-scripts/    #   页面注入脚本（标准 DOM API，无需改动）
│   ├── icons/              #   扩展图标
│   └── _locales/           #   国际化（已标注"夸克版"）
│
├── mcp-server/             # MCP Server（TypeScript）
│   ├── src/
│   │   ├── index.ts        #   MCP 入口（stdio 传输）
│   │   ├── relay-bridge.ts #   WebSocket 桥接（V2 + CDP + V1 三级协议自动回退）
│   │   ├── v1-cdp-tools.ts #   CDP 命令实现（16 个工具，V1 和 CDP 模式共用）
│   │   └── tools/index.ts  #   工具定义与注册（自动适配 V1/CDP/V2 模式）
│   ├── package.json
│   └── README.md
│
├── CHANGELOG.md            # 更新日志
├── MIGRATION_GUIDE.md      # 完整迁移指南
└── README.md               # 本文件
```

## 与 Chrome 版的关键区别

为防止 Chrome 版和夸克版串台，以下差异点需要特别注意：

| 维度 | Chrome 版 | 夸克版 |
|------|----------|--------|
| 插件名称 | QoderWork Browser Connector | QoderWork Browser Connector (Quark) |
| 分发方式 | Chrome Web Store 自动安装 | 开发者模式手动侧载 |
| manifest.json `key` | 有（Web Store 注入） | 无 |
| manifest.json `update_url` | `clients2.google.com` | 无 |
| `tabGroups` 权限 | `permissions`（硬编码） | `optional_permissions`（可选） |
| `nativeMessaging` | `permissions`（硬编码） | `optional_permissions`（可选） |
| 受限 URL 列表 | `chrome://`, `edge://` 等 | 额外增加 `quark://` |
| `tabGroups` 事件监听 | 直接注册 | 存在性检查后注册 |
| 冲突检测错误消息 | 仅检查 `chrome-extension://` | 同时检查 `chrome-extension://` 和 `quark-extension://` |
| 错误提示文案 | "Chrome 安全策略..." | "浏览器安全策略..." |
| browserType 上报 | `"chrome"` | `"quark"`（自动检测） |
| popup attachedTabs | 直接使用 | 空值安全保护（默认空数组） |
| V2 重连策略 | 5 次快速重试 | 1 次重试 + 5 分钟静默冷却 |
| MCP 协议模式 | V2 only（需 Relay 支持） | V2 优先 → CDP 回退 → V1 三级自动回退 |

## 快速开始

### 1. 安装夸克版扩展

1. 打开夸克浏览器，访问 `quark://extensions/`
2. 开启"开发者模式"
3. 点击"加载已解压的扩展程序"，选择 `extension/` 文件夹

### 2. 构建并配置 MCP Server

```bash
cd mcp-server
npm install
npm run build
```

在 MCP 客户端配置中添加：

```json
{
  "mcpServers": {
    "quark-browser": {
      "command": "node",
      "args": ["<绝对路径>/qoderwork_quark/mcp-server/dist/index.js"]
    }
  }
}
```

## 架构

```
MCP Client ──stdio──▶ MCP Server ──WebSocket──▶ Relay ──WebSocket──▶ 夸克扩展
(Claude /             (本项目)                  (QoderWork          (本项目
 Cursor /             V2: tools/invoke          Desktop App)        extension/)
 QoderWork)           CDP: 标准 CDP 协议
                      V1: forwardCDPCommand
```

MCP Server 支持三级协议自动回退：
- **V2 协议**（优先）: 连接 `/extension/v2`，使用 `tools/discover` + `tools/invoke`，扩展内部执行工具逻辑
- **CDP 协议**（推荐回退）: 连接 `/cdp?token=<hex>`，使用标准 Chrome DevTools Protocol，与 QoderWork 内置浏览器连接器共用端点，需要 token 认证（自动从 `/json/version` 获取）
- **V1 协议**（最后手段）: 连接 `/extension`，使用 `forwardCDPCommand` 发送 CDP 命令，可能与浏览器扩展争夺通道

## 防串台注意事项

开发和维护过程中，请遵循以下规则避免 Chrome 版和夸克版混淆：

1. **不要将 Chrome 版的 `manifest.json` 直接复制到夸克版** — `key` 和 `update_url` 字段会导致侧载失败
2. **修改 background.js 时务必同步受限 URL 列表** — Chrome 版不需要 `quark://`，夸克版必须有
3. **国际化文案保持独立** — 夸克版使用"夸克版"标注，Chrome 版保持原名
4. **MCP Server 的包名和描述已标注 Quark** — 不要与 Chrome 版 MCP Server 混用
5. **错误消息使用通用措辞** — 夸克版使用"浏览器安全策略"而非"Chrome 安全策略"
6. **`chrome.*` API 调用无需修改** — 这是所有 Chromium 浏览器的标准 API，不属于 Chrome 特定引用
