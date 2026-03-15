# OpenClaw Discord 在中国不稳定问题分析

## 问题现象

OpenClaw 的 Discord 集成在中国大陆环境下频繁断连，表现为：
- WebSocket 连接每 30-60 秒断开一次
- 错误码：`1006 Abnormal Closure`
- 自动重连但立即再次断开
- 循环往复，无法保持稳定连接

## 根本原因

### 1. GFW 主动探测和干扰

Discord 的所有域名和 IP 在中国大陆被 GFW 深度封锁：

```
被封锁的域名：
- discord.com
- discord.gg
- discordapp.com
- discordapp.net
- gateway.discord.gg (WebSocket Gateway)
```

**GFW 干扰机制**：
- **DNS 污染**：返回错误 IP 地址
- **IP 黑名单**：直连 Discord IP 被 RST
- **深度包检测（DPI）**：识别 Discord 流量特征
- **WebSocket 探测**：主动探测长连接并中断

### 2. WebSocket 长连接的脆弱性

Discord Gateway 使用 WebSocket 协议维持长连接：

```javascript
// OpenClaw 使用的 Discord Gateway 连接
// 位置: node_modules/@buape/carbon/src/plugins/gateway/GatewayPlugin.ts

class SafeGatewayPlugin {
  async connect() {
    this.ws = new WebSocket(gatewayUrl, {
      agent: proxyAgent  // 通过代理连接
    })

    this.ws.on('close', (code) => {
      if (code === 1006) {
        // 异常关闭 - 没有发送关闭帧
        this.handleReconnectionAttempt()
      }
    })
  }
}
```

**WebSocket 1006 错误含义**：
- 连接在没有正常关闭握手的情况下断开
- 通常由网络中断、代理失败、防火墙干扰导致
- 无法从错误本身判断具体原因

### 3. 代理协议对 WebSocket 的支持问题

#### HTTP 代理的局限性

OpenClaw 默认配置使用 HTTP CONNECT 代理：

```bash
# 当前配置
channels.discord.proxy = "http://127.0.0.1:7890"
```

HTTP 代理处理 WebSocket 的流程：
```
1. 客户端 -> 代理: CONNECT gateway.discord.gg:443
2. 代理 -> Discord: 建立 TCP 隧道
3. 客户端 <-> Discord: TLS 握手
4. 客户端 <-> Discord: WebSocket Upgrade
5. 客户端 <-> Discord: WebSocket 数据流
```

**问题点**：
- HTTP 代理只负责建立隧道，不处理 WebSocket 协议
- 长连接容易被代理服务器超时断开
- 代理节点的 QoS 策略可能限制长连接

#### VLESS Reality 协议的特性

测试环境使用的代理协议：
```yaml
# ClashX Meta 配置
type: vless
flow: xtls-rprx-vision
tls: true
reality-opts: { ... }
```

**VLESS Reality 特点**：
- 伪装成正常 HTTPS 流量
- 对 WebSocket 长连接支持有限
- 可能被 QoS 策略识别并限速

### 4. OpenClaw 重连策略的问题

#### 当前实现

```javascript
// node_modules/@buape/carbon/src/plugins/gateway/GatewayPlugin.ts:400

handleReconnectionAttempt() {
  if (this.reconnectAttempts >= this.maxReconnectAttempts) {
    throw new Error(`Max reconnect attempts (${this.maxReconnectAttempts}) reached after code 1006`)
  }

  // 固定 1000ms 退避
  setTimeout(() => {
    this.reconnectAttempts++
    this.connect()
  }, 1000)
}
```

**问题**：
1. **固定退避时间**：始终等待 1000ms，没有指数退避
2. **重连次数限制**：`maxReconnectAttempts` 可能设置为 0
3. **缺少连接质量检测**：不判断连接是否真正稳定

#### 理想的重连策略

```javascript
// 应该使用指数退避
handleReconnectionAttempt() {
  const backoff = Math.min(
    1000 * Math.pow(2, this.reconnectAttempts),
    30000  // 最大 30 秒
  )

  // 添加随机抖动避免雷鸣群效应
  const jitter = Math.random() * 1000

  setTimeout(() => {
    this.reconnectAttempts++
    this.connect()
  }, backoff + jitter)
}
```

### 5. Discord Gateway 心跳机制

Discord Gateway 要求客户端定期发送心跳：

```javascript
// Discord Gateway 心跳协议
{
  "op": 1,  // Heartbeat
  "d": null
}

// 服务器响应
{
  "op": 11,  // Heartbeat ACK
  "d": null
}
```

**在不稳定网络下的问题**：
- 心跳包可能被 GFW 丢弃
- 服务器未收到心跳，主动断开连接
- 客户端未收到 ACK，认为连接失效

## 代码级别的问题定位

### 1. 代理配置代码

```javascript
// OpenClaw Discord 插件代理配置
// 位置: dist/plugin-sdk/discord.js

const proxyUrl = config.channels?.discord?.proxy
if (proxyUrl) {
  const proxyAgent = new HttpsProxyAgent(proxyUrl)

  // 传递给 Discord.js WebSocket
  const client = new Client({
    ws: {
      agent: proxyAgent
    }
  })
}
```

**问题**：
- 只支持 HTTP/HTTPS 代理
- 没有针对 WebSocket 的特殊处理
- 没有连接超时和重试配置

### 2. WebSocket 连接代码

```javascript
// @buape/carbon Gateway 插件
// node_modules/@buape/carbon/src/plugins/gateway/GatewayPlugin.ts

class SafeGatewayPlugin {
  constructor(options) {
    this.maxReconnectAttempts = options.maxReconnectAttempts ?? 0  // 默认为 0！
    this.reconnectAttempts = 0
  }

  handleClose(code: number) {
    if (code === 1006) {
      console.log(`WebSocket connection closed with code 1006`)
      console.log(`Attempting resume with backoff: 1000ms after code 1006`)
      this.handleReconnectionAttempt()
    }
  }
}
```

**问题**：
- `maxReconnectAttempts` 默认为 0，导致重连失败
- 固定 1000ms 退避，不适应网络波动
- 没有区分临时故障和永久故障

### 3. 日志输出

```javascript
// OpenClaw 日志系统
// dist/subsystem-BDbeCphF.js:1119

logToFile({
  subsystem: "gateway/channels/discord",
  message: "discord gateway: WebSocket connection closed with code 1006"
})

logToFile({
  subsystem: "gateway/channels/discord",
  message: "discord gateway: Attempting resume with backoff: 1000ms after code 1006"
})
```

**观察到的日志模式**：
```
22:13:32 - WebSocket connection closed with code 1006
22:13:32 - Attempting resume with backoff: 1000ms
22:14:08 - WebSocket connection closed with code 1006  (36秒后)
22:14:08 - Attempting resume with backoff: 1000ms
22:14:59 - WebSocket connection closed with code 1006  (51秒后)
22:14:59 - Attempting resume with backoff: 1000ms
```

**规律**：
- 连接持续时间：30-60 秒
- 断开后立即重连
- 重连成功但很快再次断开

## 技术层面的解决方案

### 方案 1：优化代理配置（已尝试）

```bash
# 配置 Discord 专用代理
openclaw config set channels.discord.proxy "http://127.0.0.1:7890"

# 或使用 SOCKS5
openclaw config set channels.discord.proxy "socks5://127.0.0.1:7890"
```

**效果**：✅ 代理生效，但仍然不稳定

### 方案 2：更换代理节点（已尝试）

测试结果：
- HK-A-YXVM-1.0倍率: 45ms - 仍然断连
- HK-A-YXVM-2.0倍率: 46ms - 仍然断连
- JP-A-xTom-1.0倍率: 50ms - 仍然断连

**效果**：❌ 节点延迟低但问题依旧

### 方案 3：修改 OpenClaw 代码（需要 fork）

#### 3.1 增加重连次数限制

```javascript
// 修改 @buape/carbon 或 OpenClaw 配置
const client = new Client({
  ws: {
    agent: proxyAgent,
    // 增加重连配置
    maxReconnectAttempts: Infinity,  // 无限重连
    reconnectInterval: 5000,         // 初始 5 秒
    reconnectDecay: 1.5,             // 指数增长
    reconnectMaxInterval: 60000      // 最大 60 秒
  }
})
```

#### 3.2 实现智能重连

```javascript
class ImprovedGatewayPlugin {
  constructor() {
    this.reconnectAttempts = 0
    this.lastSuccessfulConnection = null
    this.connectionDurations = []
  }

  calculateBackoff() {
    // 如果连接持续时间很短，说明网络不稳定
    const avgDuration = this.getAverageConnectionDuration()

    if (avgDuration < 60000) {  // 少于 1 分钟
      // 使用更长的退避时间
      return Math.min(5000 * Math.pow(2, this.reconnectAttempts), 120000)
    } else {
      // 网络相对稳定，快速重连
      return Math.min(1000 * Math.pow(1.5, this.reconnectAttempts), 30000)
    }
  }

  handleClose(code) {
    if (code === 1006) {
      const duration = Date.now() - this.lastSuccessfulConnection
      this.connectionDurations.push(duration)

      const backoff = this.calculateBackoff()
      setTimeout(() => this.connect(), backoff)
    }
  }
}
```

#### 3.3 添加心跳监控

```javascript
class HeartbeatMonitor {
  constructor(ws) {
    this.ws = ws
    this.lastHeartbeatAck = Date.now()
    this.missedHeartbeats = 0
  }

  startMonitoring() {
    this.interval = setInterval(() => {
      const timeSinceLastAck = Date.now() - this.lastHeartbeatAck

      if (timeSinceLastAck > 45000) {  // 45 秒没有 ACK
        this.missedHeartbeats++

        if (this.missedHeartbeats >= 2) {
          // 主动断开并重连
          this.ws.close(4000, 'Heartbeat timeout')
        }
      }
    }, 15000)  // 每 15 秒检查一次
  }

  onHeartbeatAck() {
    this.lastHeartbeatAck = Date.now()
    this.missedHeartbeats = 0
  }
}
```

### 方案 4：使用更稳定的代理协议

#### Shadowsocks + simple-obfs

```yaml
# 更适合 WebSocket 长连接
proxies:
  - name: "SS-Obfs"
    type: ss
    server: example.com
    port: 8388
    cipher: chacha20-ietf-poly1305
    password: "password"
    plugin: obfs
    plugin-opts:
      mode: tls
      host: cloudflare.com
```

#### Trojan

```yaml
# Trojan 对 WebSocket 支持更好
proxies:
  - name: "Trojan"
    type: trojan
    server: example.com
    port: 443
    password: "password"
    sni: example.com
```

### 方案 5：使用 WebSocket 代理中继

```javascript
// 在本地运行 WebSocket 代理服务器
// 处理重连逻辑，对 OpenClaw 透明

const WebSocket = require('ws')
const HttpsProxyAgent = require('https-proxy-agent')

const wss = new WebSocket.Server({ port: 8080 })

wss.on('connection', (clientWs) => {
  const proxyAgent = new HttpsProxyAgent('http://127.0.0.1:7890')

  const discordWs = new WebSocket('wss://gateway.discord.gg', {
    agent: proxyAgent
  })

  // 实现智能重连和心跳监控
  const reconnector = new SmartReconnector(discordWs, proxyAgent)

  // 转发消息
  clientWs.on('message', (data) => discordWs.send(data))
  discordWs.on('message', (data) => clientWs.send(data))
})
```

## 为什么升级 OpenClaw 无法解决问题

### 1. 问题不在 OpenClaw 版本

```bash
# 当前版本已是最新
openclaw --version
# OpenClaw 2026.3.13 (61d171a)

# Discord 依赖也是最新
@discordjs/voice: 0.19.1
discord-api-types: 0.38.42
```

### 2. 问题在于网络环境

OpenClaw 的 Discord 实现本身没有问题，在国外环境下运行稳定。问题在于：
- GFW 的主动干扰
- WebSocket 长连接的脆弱性
- 代理协议的局限性

### 3. 这是系统性问题

即使 OpenClaw 更新到最新版本，只要：
- Discord 仍然被 GFW 封锁
- 使用的代理协议不变
- 重连策略不改进

问题就会持续存在。

## 实际可行的解决方案

### 短期方案：接受现状

**现实情况**：
- Discord 在中国就是不稳定
- 虽然频繁断连，但能自动重连
- 功能基本可用，只是体验差

**建议**：
- 保持当前配置（代理已启用）
- 使用延迟最低的节点
- 接受偶尔的消息延迟

### 中期方案：优化代理配置

1. **尝试不同代理协议**：
   - Shadowsocks
   - Trojan
   - VMess

2. **使用专门的 Discord 代理节点**：
   - 选择针对 Discord 优化的节点
   - 避免使用高倍率节点

3. **配置代理规则**：
```yaml
# ClashX Meta 规则
rules:
  - DOMAIN-SUFFIX,discord.com,专用节点
  - DOMAIN-SUFFIX,discord.gg,专用节点
  - DOMAIN-KEYWORD,discord,专用节点
```

### 长期方案：贡献代码改进

如果想从根本上解决问题，可以：

1. **Fork OpenClaw**：
   ```bash
   git clone https://github.com/openclaw/openclaw.git
   cd openclaw
   ```

2. **修改重连逻辑**：
   - 实现指数退避
   - 增加连接质量监控
   - 优化心跳处理

3. **提交 Pull Request**：
   - 改进对中国网络环境的支持
   - 让更多用户受益

## 总结

OpenClaw Discord 在中国不稳定的根本原因是：

1. **GFW 封锁**：Discord 被深度封锁，WebSocket 连接被主动干扰
2. **协议局限**：HTTP/VLESS 代理对 WebSocket 长连接支持有限
3. **重连策略**：固定 1000ms 退避，不适应不稳定网络
4. **心跳机制**：Discord 心跳包容易被丢弃，导致连接断开

**升级 OpenClaw 无法解决问题**，因为这是网络环境和协议层面的系统性问题，而非软件版本问题。

**实际可行的方案**：
- 接受现状，功能基本可用
- 优化代理配置和节点选择
- 或者贡献代码改进重连逻辑
