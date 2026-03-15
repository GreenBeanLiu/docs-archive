# OpenClaw China Network Fix Skill - 实现计划

## 概述

创建一个名为 `china-network-fix` 的 OpenClaw skill，提供一键配置代理的解决方案，解决国内网络环境下 OpenClaw 和 NanoClaw 各服务的连接问题。

## 文件结构

```
/Users/lijie/.openclaw/workspace/skills/china-network-fix/
├── SKILL.md                    # 技能描述和使用文档（中英文）
└── scripts/
    └── setup_proxy.sh          # 自动化配置脚本
```

## 实现细节

### 1. SKILL.md 内容结构

#### Frontmatter
```yaml
---
name: china-network-fix
description: 一键配置 OpenClaw 和 NanoClaw 代理设置，解决国内网络环境下的连接问题。支持 Telegram、Discord、GitHub 等服务。Use when OpenClaw/NanoClaw services fail to connect in China, or when you need to configure proxy for Telegram/Discord/GitHub APIs.
---
```

#### 主要章节
1. **简介**: 说明 skill 的用途和适用场景
2. **快速开始**: 一键命令使用方式
3. **工作原理**: 解释配置了哪些内容
4. **详细配置**: 各服务的具体配置说明
   - OpenClaw LaunchAgent 环境变量
   - NanoClaw LaunchAgent 环境变量
   - Discord 代理配置
   - Telegram 代理配置
5. **自定义代理**: 如何使用非默认代理地址
6. **验证配置**: 如何检查配置是否生效
7. **故障排查**: 常见问题和解决方案
8. **技术细节**: 深入说明代理机制
9. **参考资料**: 相关文档链接

### 2. setup_proxy.sh 脚本实现

#### 功能模块

**A. 参数处理**
```bash
PROXY="${1:-http://127.0.0.1:7890}"
SOCKS_PORT="${2:-7890}"
SOCKS_PROXY="socks5://127.0.0.1:${SOCKS_PORT}"
OPENCLAW_PLIST="$HOME/Library/LaunchAgents/ai.openclaw.gateway.plist"
NANOCLAW_PLIST="$HOME/Library/LaunchAgents/com.nanoclaw.plist"
```

**B. 前置检查**
- 检查 OpenClaw plist 文件是否存在
- 检查 NanoClaw plist 文件是否存在
- 如果都不存在，提示至少需要安装一个服务
- 备份原 plist 文件（首次运行时）

**C. 配置 LaunchAgent 环境变量**
使用 Python plistlib 修改 plist：
```python
import plistlib

def configure_plist(plist_path, proxy_url, socks_proxy, is_openclaw=True):
    with open(plist_path, 'rb') as f:
        plist = plistlib.load(f)

    env = plist.get('EnvironmentVariables', {})
    env.update({
        'http_proxy': proxy_url,
        'https_proxy': proxy_url,
        'all_proxy': socks_proxy,
        'HTTP_PROXY': proxy_url,
        'HTTPS_PROXY': proxy_url,
        'ALL_PROXY': socks_proxy,
    })

    # OpenClaw 需要额外的 Node.js 配置
    if is_openclaw:
        env.update({
            'NODE_OPTIONS': '--experimental-fetch',
            'GLOBAL_AGENT_HTTP_PROXY': proxy_url
        })

    # NanoClaw 需要 no_proxy 配置
    if not is_openclaw:
        env['no_proxy'] = 'localhost,127.0.0.1,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.local'

    plist['EnvironmentVariables'] = env

    with open(plist_path, 'wb') as f:
        plistlib.dump(plist, f)
```

**D. 配置 Discord 代理**
```bash
openclaw config set channels.discord.proxy "$PROXY"
openclaw config set channels.discord.enabled true
```

**E. 重启服务**
```bash
# 重启 OpenClaw
if [ -f "$OPENCLAW_PLIST" ]; then
  launchctl unload "$OPENCLAW_PLIST" 2>/dev/null || true
  sleep 2
  launchctl load "$OPENCLAW_PLIST" 2>/dev/null || true
fi

# 重启 NanoClaw
if [ -f "$NANOCLAW_PLIST" ]; then
  launchctl unload "$NANOCLAW_PLIST" 2>/dev/null || true
  sleep 2
  launchctl load "$NANOCLAW_PLIST" 2>/dev/null || true
fi

sleep 3
```

**F. 验证配置**
```bash
# OpenClaw 验证
if command -v openclaw &> /dev/null; then
  echo "OpenClaw 状态:"
  openclaw gateway status 2>&1 | head -5
  openclaw config get channels.discord.proxy 2>&1
  echo "查看 Telegram 日志: openclaw channels logs --lines 10"
fi

# NanoClaw 验证
if [ -f "$NANOCLAW_PLIST" ]; then
  echo "NanoClaw 状态:"
  launchctl list | grep nanoclaw
  echo "查看日志: tail -f ~/Works/nanoclaw/logs/nanoclaw.log"
fi
```

#### 错误处理

1. **plist 文件不存在**:
   ```bash
   if [ ! -f "$OPENCLAW_PLIST" ] && [ ! -f "$NANOCLAW_PLIST" ]; then
     echo "错误: 未找到 OpenClaw 或 NanoClaw 的 plist 文件"
     echo "OpenClaw: 请先运行 'openclaw gateway install'"
     echo "NanoClaw: 请先运行 'npm run setup' 在 ~/Works/nanoclaw 目录"
     exit 1
   fi
   ```

2. **Python 不可用**:
   ```bash
   if ! command -v python3 &> /dev/null; then
     echo "错误: 需要 Python 3"
     exit 1
   fi
   ```

3. **openclaw 命令不可用**:
   ```bash
   if ! command -v openclaw &> /dev/null; then
     echo "错误: 未找到 openclaw 命令"
     exit 1
   fi
   ```

4. **备份失败**:
   ```bash
   if ! cp "$PLIST" "$PLIST.backup.$(date +%Y%m%d_%H%M%S)"; then
     echo "警告: 无法创建备份文件"
     read -p "是否继续? (y/N) " -n 1 -r
     [[ ! $REPLY =~ ^[Yy]$ ]] && exit 1
   fi
   ```

#### 幂等性保证

- 每次运行前先移除旧的代理配置
- 使用 Python plistlib 确保 XML 格式正确
- 配置前检查是否已存在相同配置

### 3. 代码片段

#### setup_proxy.sh 完整框架

```bash
#!/bin/bash
set -e

# 颜色输出
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# 参数
PROXY="${1:-http://127.0.0.1:7890}"
SOCKS_PORT="${2:-7890}"
SOCKS_PROXY="socks5://127.0.0.1:${SOCKS_PORT}"
OPENCLAW_PLIST="$HOME/Library/LaunchAgents/ai.openclaw.gateway.plist"
NANOCLAW_PLIST="$HOME/Library/LaunchAgents/com.nanoclaw.plist"

# 函数: 打印信息
info() { echo -e "${GREEN}[INFO]${NC} $1"; }
warn() { echo -e "${YELLOW}[WARN]${NC} $1"; }
error() { echo -e "${RED}[ERROR]${NC} $1"; }

# 前置检查
check_prerequisites() {
  # 检查 OpenClaw plist
  # 检查 NanoClaw plist
  # 检查 Python
  # 至少需要一个服务
}

# 备份配置
backup_plist() {
  # 创建备份（OpenClaw 和 NanoClaw）
}

# 配置 LaunchAgent
configure_launchagent() {
  # 使用 Python 修改 plist（OpenClaw 和 NanoClaw）
}

# 配置 Discord
configure_discord() {
  # openclaw config set（仅 OpenClaw）
}

# 重启服务
restart_services() {
  # launchctl unload/load（OpenClaw 和 NanoClaw）
}

# 验证配置
verify_config() {
  # 检查状态（OpenClaw 和 NanoClaw）
  # 显示配置
}

# 主流程
main() {
  info "开始配置代理..."
  info "代理地址: $PROXY"

  check_prerequisites
  backup_plist
  configure_launchagent
  configure_discord
  restart_services
  verify_config

  info "配置完成！"
}

main
```

### 4. 涉及的文件路径

- **创建**: `/Users/lijie/.openclaw/workspace/skills/china-network-fix/SKILL.md`
- **创建**: `/Users/lijie/.openclaw/workspace/skills/china-network-fix/scripts/setup_proxy.sh`
- **修改**: `~/Library/LaunchAgents/ai.openclaw.gateway.plist`
- **修改**: `~/Library/LaunchAgents/com.nanoclaw.plist`
- **备份**: `~/Library/LaunchAgents/ai.openclaw.gateway.plist.backup.YYYYMMDD_HHMMSS`
- **备份**: `~/Library/LaunchAgents/com.nanoclaw.plist.backup.YYYYMMDD_HHMMSS`

### 5. 权衡考量

#### 方案 A: 纯文档型 skill（类似 fix-discord-china）
**优点**:
- 简单，无需维护脚本
- 用户可以理解每一步操作
- 灵活性高

**缺点**:
- 用户需要手动执行多个命令
- 容易出错
- 不够自动化

#### 方案 B: 脚本型 skill（类似 fix-gateway-proxy）
**优点**:
- 一键执行，用户体验好
- 减少人为错误
- 可以添加验证和错误处理

**缺点**:
- 需要维护脚本
- 可能遇到环境差异问题
- 调试相对复杂

#### 方案 C: 混合型（推荐）
**优点**:
- 提供脚本快速配置
- 文档详细说明每一步
- 用户可以选择自动或手动

**缺点**:
- 文档和脚本需要保持同步

**选择**: 方案 C（混合型）

### 6. 测试验证方案

#### 测试场景

1. **全新安装 OpenClaw**:
   - 运行 `openclaw gateway install`
   - 运行 `setup_proxy.sh`
   - 验证配置生效

2. **全新安装 NanoClaw**:
   - 在 `~/Works/nanoclaw` 运行 `npm run setup`
   - 运行 `setup_proxy.sh`
   - 验证配置生效

3. **同时安装 OpenClaw 和 NanoClaw**:
   - 运行 `setup_proxy.sh`
   - 验证两个服务都配置成功

4. **已有配置**:
   - 已配置过代理
   - 再次运行 `setup_proxy.sh`
   - 验证配置被正确更新

5. **自定义代理**:
   - 运行 `setup_proxy.sh http://127.0.0.1:1080 1080`
   - 验证使用自定义代理

6. **配置验证**:
   - 检查 plist 文件内容
   - 运行 `openclaw config get channels.discord.proxy`
   - 查看服务日志

#### 验证命令

```bash
# OpenClaw 验证
# 1. 检查 OpenClaw plist 环境变量
plutil -p ~/Library/LaunchAgents/ai.openclaw.gateway.plist | grep -A 20 EnvironmentVariables

# 2. 检查 Discord 配置
openclaw config get channels.discord.proxy

# 3. 检查 gateway 状态
openclaw gateway status

# 4. 测试 Telegram 连接
openclaw channels logs --lines 10 | grep telegram

# 5. 测试 Discord 连接
tail -f ~/.openclaw/logs/gateway.log | grep discord

# NanoClaw 验证
# 6. 检查 NanoClaw plist 环境变量
plutil -p ~/Library/LaunchAgents/com.nanoclaw.plist | grep -A 20 EnvironmentVariables

# 7. 检查 NanoClaw 服务状态
launchctl list | grep nanoclaw

# 8. 查看 NanoClaw 日志
tail -f ~/Works/nanoclaw/logs/nanoclaw.log

# 9. 测试 NanoClaw Telegram 连接
tail -f ~/Works/nanoclaw/logs/nanoclaw.log | grep -i telegram
```

## 实现步骤

1. ✅ 研究现有 skills 结构
2. ✅ 编写实现计划
3. ✅ 等待用户审核和批注
4. ✅ 根据批注修改计划
5. ✅ 实现 SKILL.md
6. ✅ 实现 setup_proxy.sh
7. ✅ 测试验证
8. ✅ 完善文档

## 注意事项

1. **安全性**: 备份原配置文件，避免数据丢失
2. **兼容性**: 确保脚本在不同 macOS 版本上运行
3. **用户体验**: 提供清晰的输出信息和错误提示
4. **文档质量**: 中英文双语，详细的故障排查指南
5. **幂等性**: 脚本可以重复执行而不产生副作用

## 预期效果

用户只需运行一条命令：
```bash
bash ~/.openclaw/workspace/skills/china-network-fix/scripts/setup_proxy.sh
```

即可完成 OpenClaw 和 NanoClaw 的所有代理配置，解决国内网络环境下的连接问题。

脚本会自动检测已安装的服务：
- 如果安装了 OpenClaw，配置 OpenClaw gateway 和 Discord 代理
- 如果安装了 NanoClaw，配置 NanoClaw LaunchAgent 代理
- 如果两者都安装，同时配置两个服务
