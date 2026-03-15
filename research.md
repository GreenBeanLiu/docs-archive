# OpenClaw Skill 研究报告

## 目标
创建一个通用的 OpenClaw skill，用于解决国内网络环境下的代理配置问题。

## 现有 Skills 分析

### 1. fix-gateway-proxy
- **目的**: 修复 OpenClaw gateway 的 Telegram 代理配置
- **结构**:
  - `SKILL.md`: 技能描述和使用说明
  - `scripts/fix_proxy.sh`: 自动化脚本
- **核心功能**:
  - 修改 `~/Library/LaunchAgents/ai.openclaw.gateway.plist`
  - 注入代理环境变量（http_proxy, https_proxy, all_proxy）
  - 重新加载 LaunchAgent 服务
- **默认代理**: `http://127.0.0.1:7890`

### 2. fix-discord-china
- **目的**: 修复 Discord 在中国大陆的连接问题
- **结构**: 仅包含 `SKILL.md`（纯文档型 skill）
- **核心内容**:
  - Discord 代理配置命令
  - LaunchAgent 环境变量配置
  - 详细的故障排查指南
  - 中文文档

## OpenClaw 代理机制分析

### LaunchAgent 环境变量
OpenClaw gateway 作为 macOS LaunchAgent 运行，不继承终端的环境变量，必须在 plist 文件中显式配置：

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>http_proxy</key>
  <string>http://127.0.0.1:7890</string>
  <key>https_proxy</key>
  <string>http://127.0.0.1:7890</string>
  <key>all_proxy</key>
  <string>socks5://127.0.0.1:7890</string>
  <key>HTTPS_PROXY</key>
  <string>http://127.0.0.1:7890</string>
  <key>HTTP_PROXY</key>
  <string>http://127.0.0.1:7890</string>
  <key>ALL_PROXY</key>
  <string>socks5://127.0.0.1:7890</string>
</dict>
```

### 特定服务代理配置
某些服务（如 Discord）还需要通过 `openclaw config` 命令配置：

```bash
openclaw config set channels.discord.proxy "http://127.0.0.1:7890"
```

### Node.js 特殊配置
对于 Node.js 22+ 的原生 fetch，需要额外配置：

```xml
<key>NODE_OPTIONS</key>
<string>--experimental-fetch</string>
<key>GLOBAL_AGENT_HTTP_PROXY</key>
<string>http://127.0.0.1:7890</string>
```

## 国内常见网络问题

1. **Telegram API 被屏蔽** (`api.telegram.org`)
2. **Discord API/Gateway 被屏蔽**
3. **GitHub API 访问缓慢**
4. **OpenAI API 被屏蔽**
5. **其他国际服务连接问题**

## 通用解决方案设计

### 方案 1: 通用代理配置工具
创建一个统一的脚本，自动配置所有必要的代理设置：
- LaunchAgent 环境变量
- OpenClaw 配置文件
- 支持自定义代理地址
- 一键重启服务

### 方案 2: 诊断 + 修复工具
不仅配置代理，还包括：
- 检测当前网络环境
- 诊断哪些服务需要代理
- 自动应用相应配置
- 验证配置是否生效

### 方案 3: 扩展现有 fix-gateway-proxy
在现有脚本基础上扩展：
- 支持更多服务（Discord, GitHub, OpenAI）
- 添加配置验证
- 提供中英文文档

## 推荐方案

**方案 1 + 部分方案 2**：创建一个通用的代理配置 skill，包含：

1. **自动化脚本** (`scripts/setup_proxy.sh`):
   - 配置 LaunchAgent 环境变量
   - 配置 Discord 代理
   - 可选配置其他服务
   - 验证配置

2. **详细文档** (`SKILL.md`):
   - 中英文双语
   - 常见问题排查
   - 各服务特定配置说明

3. **特性**:
   - 默认代理: `http://127.0.0.1:7890`
   - 支持自定义代理地址
   - 幂等操作（可重复执行）
   - 自动备份原配置

## 技术要点

1. **plist 文件操作**: 使用 Python plistlib 确保 XML 格式正确
2. **服务重启**: `launchctl unload/load` 或 `openclaw gateway restart`
3. **配置验证**: 检查代理连通性，验证服务状态
4. **错误处理**: 检查文件存在性，提供清晰的错误信息

## 文件结构

```
china-network-fix/
├── SKILL.md           # 技能描述和使用文档
└── scripts/
    └── setup_proxy.sh # 自动化配置脚本
```

## 下一步

编写详细的实现计划（plan.md），包括：
1. SKILL.md 的完整内容结构
2. setup_proxy.sh 的详细实现逻辑
3. 需要处理的边界情况
4. 测试验证方案

---

# OpenClaw Discord 稳定性问题研究

## 当前版本状态

- **OpenClaw 版本**: 2026.3.13 (最新)
- **Discord 依赖**:
  - `@discordjs/voice`: ^0.19.1 (已是最新)
  - `discord-api-types`: ^0.38.42 (已是最新)

## 关键发现

OpenClaw **不直接依赖** `discord.js` 包，而是使用：
1. `@discordjs/voice` - Discord 语音功能库
2. `discord-api-types` - Discord API 类型定义

这两个包都已经是 npm 上的最新版本。

## Discord 实现架构

OpenClaw 使用自定义的 Discord 集成：
- **实现位置**: `/usr/local/lib/node_modules/openclaw/dist/plugin-sdk/discord.js`
- **架构模式**: Gateway + Plugin SDK
- **核心组件**:
  - `gateway-plugin.d.ts` - Gateway 插件接口
  - `gateway-error-guard.d.ts` - 错误处理
  - `gateway-registry.d.ts` - Gateway 注册表
  - `monitor.gateway.d.ts` - 监控系统

## 不稳定问题可能原因

### 1. 网络层面
- Discord Gateway 在国内被墙，连接不稳定
- 代理配置不完整或失效
- WebSocket 连接频繁断开

### 2. 配置层面
- `channels.discord.proxy` 未正确配置
- LaunchAgent 环境变量缺失
- Gateway 重连策略配置不当

### 3. 实现层面
- Gateway 错误处理机制
- 重连逻辑
- 心跳检测

## 诊断步骤

1. **检查日志**:
   ```bash
   tail -100 /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep -i discord
   ```

2. **检查代理配置**:
   ```bash
   openclaw config get channels.discord.proxy
   plutil -p ~/Library/LaunchAgents/ai.openclaw.gateway.plist | grep -A 20 EnvironmentVariables
   ```

3. **检查 Gateway 状态**:
   ```bash
   openclaw gateway status
   ps aux | grep openclaw-gateway
   ```

4. **测试 Discord 连接**:
   ```bash
   curl -x http://127.0.0.1:7890 https://discord.com/api/v10/gateway
   ```

## 解决方案

### 短期方案（无需升级）
1. 确保代理配置完整
2. 调整 Gateway 重连参数
3. 增加错误日志级别

### 长期方案
1. 等待 OpenClaw 更新（如果有 Discord 相关修复）
2. 监控 `@discordjs/voice` 和 `discord-api-types` 的更新
3. 考虑使用更稳定的代理方案

## 结论

**升级 OpenClaw 无法解决问题**，因为：
1. 当前已是最新版本
2. Discord 依赖包已是最新
3. 问题更可能是配置或网络原因

需要提供具体的错误日志和不稳定表现，才能进一步诊断。

---

# Discord 不稳定问题诊断结果

## 日志分析

从日志中发现核心问题：

```
discord gateway: WebSocket connection closed with code 1006
discord gateway: Attempting resume with backoff: 1000ms after code 1006
```

**WebSocket 错误码 1006** 含义：
- 异常关闭（Abnormal Closure）
- 连接在没有发送关闭帧的情况下断开
- 通常由网络问题、代理问题或防火墙导致

## 问题根源

### 1. 代理配置问题
- ✅ LaunchAgent 已配置代理环境变量
- ✅ ClashX Meta 代理正在运行（端口 7890）
- ❌ **Discord 专用代理配置缺失**：`channels.discord.proxy` 未设置

### 2. 代理连接测试
```bash
curl -x http://127.0.0.1:7890 https://discord.com/api/v10/gateway
```
- 代理隧道建立成功（HTTP/1.1 200 Connection established）
- TLS 握手开始但未完成
- 可能是代理规则或 Discord Gateway WebSocket 连接问题

### 3. 重连策略
- 当前使用固定 1000ms backoff
- 频繁断开重连（每 30-60 秒一次）
- 重连策略可能不够智能（应该使用指数退避）

## 解决方案

### 立即修复

1. **配置 Discord 专用代理**：
```bash
openclaw config set channels.discord.proxy "http://127.0.0.1:7890"
openclaw gateway restart
```

2. **验证配置**：
```bash
openclaw config get channels.discord
openclaw gateway status
```

### 进阶优化

1. **检查 ClashX 规则**：
   - 确保 Discord 域名（`*.discord.com`, `gateway.discord.gg`）走代理
   - 检查是否有 WebSocket 相关的规则限制

2. **调整重连策略**（需要修改 OpenClaw 配置）：
   - 使用指数退避而非固定 1000ms
   - 增加最大重连间隔
   - 添加连接健康检查

3. **监控日志**：
```bash
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep discord
```

## 预期效果

配置 `channels.discord.proxy` 后：
- Discord Gateway 将使用专用代理配置
- WebSocket 连接应该更稳定
- 1006 错误频率应该显著降低

如果问题仍然存在，可能需要：
- 更换代理节点
- 调整 ClashX 的 WebSocket 相关配置
- 检查网络质量和延迟
