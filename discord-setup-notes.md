# Discord 鲍伯配置记录

**配置时间**: 2026-03-12

## 当前工作配置

### OpenClaw 配置
- **频道**: bob-only (专属频道)
- **服务器**: GG's Claw (ID: 971392610017103912)
- **频道 ID**: 1481332368685137970
- **Bot ID**: 1480565645438484662
- **requireMention**: false (不需要 @mention)
- **代理**: http://127.0.0.1:7890

### Discord Bot 权限设置
在 https://discord.com/developers/applications 中配置：

#### Privileged Gateway Intents (必须启用)
- ✅ PRESENCE INTENT
- ✅ SERVER MEMBERS INTENT
- ✅ MESSAGE CONTENT INTENT (最关键)

#### Bot Permissions
- ✅ View Channel (查看频道)
- ✅ Send Messages (发送消息)
- ✅ Read Message History (读取消息历史)
- ✅ Embed Links (嵌入链接)
- ✅ Add Reactions (添加反应)

### 配置文件位置
- OpenClaw 配置: `~/.openclaw/openclaw.json`
- Discord 配置段:
```json
"discord": {
  "enabled": true,
  "proxy": "http://127.0.0.1:7890",
  "groupPolicy": "open",
  "guilds": {
    "971392610017103912": {
      "requireMention": false,
      "channels": {
        "bob-only": {
          "requireMention": false
        }
      }
    }
  }
}
```

### LLM 配置
- **Provider**: yunyi-claude
- **Base URL**: https://xchai.xyz
- **Model**: claude-opus-4-6
- **API Key**: sk-WqxSo5GlroQybzNC0UaXJNdhv54PeMWl0IB2V3QxYzJfhAqt

## 重要提示

### 如果 Discord 不工作
1. 检查 Message Content Intent 是否启用
2. 如果刚启用，需要重新邀请 bot 到服务器
3. 重启 OpenClaw: `openclaw gateway restart`

### 如果需要更换频道
编辑配置文件，修改 channels 部分：
```bash
openclaw config set channels.discord.guilds.971392610017103912.channels '{"新频道名":{"requireMention":false}}'
openclaw gateway restart
```

### 健康检查
运行健康检查 skill：
```bash
bash ~/.openclaw/workspace/skills/openclaw-health-check/scripts/health_check.sh
```

## 备份信息
- 备份位置: `~/openclaw-backups/`
- 最新备份: `openclaw-2026-03-12_0114.tar.gz` (176M)
- 自动保留最近 7 个备份

## 故障排查

### Discord 显示"输入中"但无输出
1. 检查 Message Content Intent 是否启用
2. 重新邀请 bot（生成新的 OAuth2 URL）
3. 确认 bot 在频道中有发送消息权限
4. 检查代理是否正常运行（端口 7890）

### 代理问题
```bash
# 测试代理
curl -I --proxy http://127.0.0.1:7890 https://discord.com

# 检查代理进程
lsof -i :7890
```

### 查看日志
```bash
# 实时日志
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# Discord 相关日志
tail -100 /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep -i discord
```

## 相关 Skills
- `openclaw-health-check`: 健康检查
- `openclaw-backup`: 备份配置
- `china-network-fix`: 代理配置
- `fix-discord-china`: Discord 代理指南

## 成功标志
- ✅ OpenClaw gateway 运行正常
- ✅ Discord 已登录（鲍伯）
- ✅ bob-only 频道已解析
- ✅ 代理已启用（REST + Gateway）
- ✅ 可以接收和发送消息
- ✅ 不需要 @mention 即可响应
