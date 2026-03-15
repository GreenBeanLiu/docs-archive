# macOS 系统优化建议报告

**生成时间**: 2026-03-15 17:36
**系统**: macOS 14.6.1 (23G93)
**运行时长**: 32 天 20 小时

---

## 系统状态概览

### 硬件配置
- **CPU**: 6 核心 / 12 线程
- **内存**: 16 GB
- **磁盘**: 466 GB (已用 32%)

### 当前状态
- **负载**: 1.78, 1.81, 1.88 (正常)
- **内存使用**: 10.3 GB / 16 GB (64%)
- **磁盘使用**: 139 GB / 466 GB (32%)

---

## 优化建议

### 🔴 高优先级（立即优化）

#### 1. iTerm2 CPU 占用过高 (89.3%)

**问题**:
```
PID 36926: iTerm2 占用 89.3% CPU
运行时长: 5 小时 22 分钟
```

**原因**:
- 可能有大量输出的进程在后台运行
- 或者有死循环的脚本

**解决方案**:
```bash
# 查看 iTerm2 中运行的进程
ps aux | grep -E "PID|36926" -A 5

# 建议重启 iTerm2
# 或关闭不需要的标签页/窗口
```

#### 2. Claude CLI 占用较高 (57.3% CPU, 496 MB 内存)

**问题**:
```
PID 54754: claude --allow-dangerously-skip-permissions
CPU: 57.3%
内存: 496 MB
```

**建议**:
- 当前会话结束后退出
- 避免长时间保持 Claude CLI 运行

#### 3. 虚拟机占用大量内存 (2.67 GB)

**问题**:
```
PID 75842: Virtualization.VirtualMachine
CPU: 16.4%
内存: 2.67 GB (16% 总内存)
```

**建议**:
- 检查是否有不需要的虚拟机在运行
- 考虑降低虚拟机内存分配

---

### 🟡 中优先级（建议优化）

#### 4. 清理缓存文件 (16 GB)

**缓存占用**:
```
Yarn:           6.3 GB
JetBrains:      4.8 GB
Google Chrome:  1.9 GB
微信:           1.7 GB
微信开发者工具:  1.1 GB
```

**清理命令**:
```bash
# 清理 Yarn 缓存
yarn cache clean

# 清理 JetBrains 缓存（如果不再使用）
rm -rf ~/Library/Caches/JetBrains

# 清理浏览器缓存
rm -rf ~/Library/Caches/Google/Chrome

# 清理微信缓存（谨慎，可能删除聊天记录）
# rm -rf ~/Library/Caches/com.tencent.xinWeChat
```

**预计释放**: 10-15 GB

#### 5. 清理旧日志文件

**日志状态**:
```
超过 30 天的日志: 204 个
日志总大小: 65 MB
```

**清理命令**:
```bash
# 清理超过 30 天的日志
find ~/Library/Logs -name "*.log" -mtime +30 -type f -delete

# 清理系统日志（需要管理员权限）
sudo rm -rf /private/var/log/*.log.*
```

**预计释放**: 50-100 MB

#### 6. 优化 OpenClaw Gateway 内存占用

**问题**:
```
PID 55494: openclaw-gateway
内存: 457 MB
```

**建议**:
- 检查是否有内存泄漏
- 定期重启服务（每周一次）

```bash
# 重启 OpenClaw Gateway
openclaw gateway restart
```

---

### 🟢 低优先级（可选优化）

#### 7. 清理不用的 Homebrew 包

**当前状态**: 51 个 Homebrew 包

**清理命令**:
```bash
# 查看未使用的依赖
brew autoremove

# 清理旧版本
brew cleanup
```

#### 8. 优化 Docker 资源分配

**当前状态**:
```
newapi 容器: 49.76 MB / 5.535 GB (仅用 0.9%)
```

**建议**:
- Docker 分配了 5.5 GB 内存但只用了 50 MB
- 可以降低 Docker 内存限制到 2 GB

**操作**:
1. 打开 Docker Desktop
2. Settings → Resources → Memory
3. 调整为 2 GB

**预计释放**: 3.5 GB 内存

#### 9. 关闭不必要的后台进程

**发现的后台进程**:
```bash
# 多个 tail -f 进程在后台运行
PID 75661: tail -f logs/nanoclaw.log
PID 62975: tail -f ~/.openclaw/logs/gateway.log
```

**建议**:
```bash
# 查找所有 tail 进程
ps aux | grep "tail -f" | grep -v grep

# 关闭不需要的
kill <PID>
```

#### 10. 定期重启系统

**当前运行时长**: 32 天 20 小时

**建议**:
- macOS 建议每 1-2 周重启一次
- 清理内存碎片和临时文件
- 释放被占用的资源

---

## 优化脚本

### 一键清理脚本

```bash
#!/bin/bash
# macOS 系统清理脚本

echo "开始清理系统..."

# 1. 清理 Yarn 缓存
echo "清理 Yarn 缓存..."
yarn cache clean 2>/dev/null

# 2. 清理 Homebrew
echo "清理 Homebrew..."
brew autoremove
brew cleanup

# 3. 清理旧日志
echo "清理旧日志..."
find ~/Library/Logs -name "*.log" -mtime +30 -type f -delete

# 4. 清理 npm 缓存
echo "清理 npm 缓存..."
npm cache clean --force

# 5. 清理 pip 缓存
echo "清理 pip 缓存..."
pip cache purge 2>/dev/null

# 6. 清理系统缓存
echo "清理系统缓存..."
sudo rm -rf /Library/Caches/*
sudo rm -rf /System/Library/Caches/*

# 7. 清理下载文件夹（超过 30 天）
echo "清理下载文件夹..."
find ~/Downloads -mtime +30 -type f -delete

# 8. 清理废纸篓
echo "清理废纸篓..."
rm -rf ~/.Trash/*

echo "清理完成！"
df -h /
```

### 内存优化脚本

```bash
#!/bin/bash
# 内存优化脚本

echo "优化内存..."

# 1. 清理内存缓存
sudo purge

# 2. 重启 OpenClaw Gateway
echo "重启 OpenClaw Gateway..."
openclaw gateway restart

# 3. 重启 NanoClaw
echo "重启 NanoClaw..."
launchctl kickstart -k gui/$(id -u)/com.nanoclaw

# 4. 清理 Docker 未使用的资源
echo "清理 Docker..."
docker system prune -f

echo "内存优化完成！"
vm_stat | head -5
```

---

## 监控建议

### 定期检查命令

```bash
# 每周执行一次
alias sys-check='
  echo "=== 系统负载 ===" &&
  uptime &&
  echo "" &&
  echo "=== 内存使用 ===" &&
  vm_stat | head -5 &&
  echo "" &&
  echo "=== 磁盘使用 ===" &&
  df -h / &&
  echo "" &&
  echo "=== CPU 占用 TOP 5 ===" &&
  ps aux | sort -rn -k 3 | head -6 &&
  echo "" &&
  echo "=== 内存占用 TOP 5 ===" &&
  ps aux | sort -rn -k 4 | head -6
'
```

### 设置定时任务

```bash
# 添加到 crontab
crontab -e

# 每周日凌晨 3 点清理
0 3 * * 0 /path/to/cleanup-script.sh

# 每天凌晨 2 点优化内存
0 2 * * * /path/to/memory-optimize.sh
```

---

## 预期效果

执行所有优化后：

| 项目 | 优化前 | 优化后 | 释放 |
|------|--------|--------|------|
| **内存使用** | 10.3 GB | ~7 GB | 3.3 GB |
| **磁盘空间** | 139 GB | ~125 GB | 14 GB |
| **CPU 负载** | 1.8 | ~1.0 | 降低 44% |
| **后台进程** | 多个 | 精简 | - |

---

## 总结

### 立即执行（5 分钟）
1. 重启 iTerm2（释放 89% CPU）
2. 退出 Claude CLI（释放 57% CPU + 496 MB 内存）
3. 清理 Yarn 缓存（释放 6.3 GB）

### 本周执行（30 分钟）
1. 清理所有缓存（释放 10-15 GB）
2. 清理旧日志（释放 50-100 MB）
3. 优化 Docker 内存分配（释放 3.5 GB）
4. 重启系统

### 长期维护
1. 每周运行清理脚本
2. 每月重启系统一次
3. 定期检查后台进程
4. 监控内存和 CPU 使用

**预计总释放**: 内存 3-4 GB，磁盘 15-20 GB
