# OpenClaw 架构研究报告

> 版本: 2026.1.29 | 运行时: Node 22+ / TypeScript ESM | 许可证: MIT

---

## 一、系统定位与设计目标

OpenClaw 是一个**自托管的个人 AI 助手平台**。核心设计目标:

1. **多通道统一接入** — 对接 WhatsApp、Telegram、Slack、Discord、Signal、iMessage 等 18+ 消息通道
2. **多模型动态调度** — 支持 Claude、ChatGPT、Gemini、Copilot、Qwen、本地模型等, 具备故障转移能力
3. **插件化扩展** — 30+ 插件包, 通道/记忆/认证/工具均可独立开发和加载
4. **单用户常驻** — 面向个人设备, 以 Gateway 进程常驻运行, CLI/移动端作为客户端

---

## 二、全局架构总览

```
+------------------------------------------------------------------+
|                        用户交互层                                  |
|  +----------+  +----------+  +---------+  +--------+  +--------+ |
|  | WhatsApp |  | Telegram |  | Discord |  | Slack  |  | Signal | |
|  +----+-----+  +----+-----+  +----+----+  +---+----+  +---+----+ |
|       |              |             |           |            |      |
+-------+--------------+-------------+-----------+------------+-----+
        |              |             |           |            |
+-------v--------------v-------------v-----------v------------v-----+
|                     Channel 通道层                                 |
|  +------------------+  +----------------+  +-------------------+  |
|  | Channel Plugin   |  | Channel Dock   |  | Channel Registry  |  |
|  | (适配器集合)      |  | (轻量元数据)   |  | (UI元数据+排序)   |  |
|  +--------+---------+  +-------+--------+  +---------+---------+  |
|           |                    |                      |           |
+-----------+--------------------+----------------------+-----------+
            |                    |
+-----------v--------------------v----------------------------------+
|                     Gateway 网关层                                 |
|  +-------------+  +------------+  +-------------+  +----------+  |
|  | WS Server   |  | RPC Method |  | Broadcast   |  | Auth     |  |
|  | (连接管理)   |  | Handler    |  | (事件分发)  |  | (认证鉴权)|  |
|  +------+------+  +-----+------+  +------+------+  +-----+----+  |
|         |                |                |               |       |
|  +------v------+  +------v------+  +------v------+               |
|  | Chat Run    |  | Channel Mgr |  | Node        |               |
|  | State       |  | (生命周期)   |  | Registry    |               |
|  +------+------+  +------+------+  +------+------+               |
+---------+----------------+----------------+-----------------------+
          |                |                |
+---------v----------------v----------------v-----------------------+
|                     Routing 路由层                                 |
|  +------------------+  +-----------------+  +------------------+  |
|  | Binding 解析     |  | Session Key 构建 |  | Agent Route 级联 |  |
|  +--------+---------+  +--------+--------+  +--------+---------+  |
+-----------+-----------------+-------------------+-----------------+
            |                 |                   |
+-----------v-----------------v-------------------v-----------------+
|                     Agent 代理层                                   |
|  +----------------+  +-----------------+  +-------------------+   |
|  | Agent Scope    |  | Model Selection |  | Model Fallback    |   |
|  | (配置解析)     |  | (别名/候选)     |  | (故障转移循环)    |   |
|  +-------+--------+  +--------+--------+  +--------+----------+   |
|          |                    |                     |              |
|  +-------v--------------------v---------------------v----------+  |
|  |              Pi Embedded Runner (执行引擎)                   |  |
|  |  模型解析 -> 认证profile -> 上下文窗口校验 -> 尝试循环      |  |
|  +------+------------------------------------------------------+  |
+---------+---------------------------------------------------------+
          |
+---------v---------------------------------------------------------+
|                     Provider 提供者层                              |
|  +----------+  +--------+  +--------+  +-------+  +-----------+  |
|  | Anthropic|  | OpenAI |  | Google |  | Qwen  |  | Copilot   |  |
|  +----------+  +--------+  +--------+  +-------+  +-----------+  |
|  +----------+  +--------+  +--------+  +-------+  +-----------+  |
|  | Minimax  |  | Xiaomi |  | Ollama |  | Bedrock| | Custom    |  |
|  +----------+  +--------+  +--------+  +-------+  +-----------+  |
+-------------------------------------------------------------------+
          |
+---------v---------------------------------------------------------+
|                     Plugin 插件层                                  |
|  +-------------+  +-----------+  +-------------+  +------------+ |
|  | Discovery   |  | Loader    |  | Registry    |  | Hook 系统  | |
|  | (多源发现)  |  | (jiti加载)|  | (注册中心)  |  | (生命周期) | |
|  +-------------+  +-----------+  +-------------+  +------------+ |
+-------------------------------------------------------------------+
          |
+---------v---------------------------------------------------------+
|                     Infrastructure 基础设施层                      |
|  +--------+  +----------+  +--------+  +---------+  +----------+ |
|  | Config |  | Session  |  | Media  |  | Memory  |  | Security | |
|  | (配置)  |  | (会话)   |  | (媒体) |  | (记忆)  |  | (安全)   | |
|  +--------+  +----------+  +--------+  +---------+  +----------+ |
|  +--------+  +----------+  +--------+  +---------+               |
|  | Logger |  | Device ID|  | mDNS   |  | Dotenv  |               |
|  | (日志)  |  | (设备)   |  | (发现) |  | (环境)  |               |
|  +--------+  +----------+  +--------+  +---------+               |
+-------------------------------------------------------------------+
```

---

## 三、启动流程

从 CLI 入口到 Agent 执行, 经历以下阶段:

```
entry.ts                              run-main.ts
  |                                      |
  | 1. process.title="openclaw"          | 5. 加载 .env
  | 2. 抑制 Node 实验性警告              | 6. assertSupportedRuntime()
  | 3. 必要时 respawn 子进程             | 7. tryRouteCli() 快速路由
  | 4. 解析 --profile/--dev             | 8. buildProgram()
  |                                      | 9. 懒加载子命令 + 插件
  +----> dynamic import run-main.ts ---->+
                                         |
                                    program.parseAsync()
                                         |
                                    preaction hooks
                                    (banner, config校验, 插件加载)
                                         |
                                    Commander dispatch
                                         |
                              +----------+----------+
                              |                     |
                        gateway start          agent command
                              |                     |
                     startGatewayServer()    agentCliCommand()
                              |                     |
                     WebSocket + HTTP         gateway优先
                     通道启动                  本地 fallback
                     插件注册
                     Cron 服务
```

**关键设计**: 快速路由(Route-First)机制允许 `health`、`status`、`sessions` 等简单命令绕过完整的 Commander 解析和插件加载, 降低延迟。

---

## 四、Gateway 网关层详解

Gateway 是系统的**控制平面**, 以 WebSocket 服务器常驻运行。

### 4.1 连接生命周期

```
客户端 connect                  Gateway
   |                              |
   |-------- TCP/TLS ------------>|
   |                              | 生成 connId (UUID)
   |<-- connect.challenge(nonce)--|
   |                              |
   |--- connect(auth, device) --->|
   |                              | 1. 协议版本协商
   |                              | 2. 角色校验 (operator/node)
   |                              | 3. 设备签名验证 (ED25519)
   |                              | 4. Gateway 认证 (token/password/Tailscale)
   |                              | 5. Scope 分配
   |<---- connect.ok ------------|
   |                              | 注册 Presence, 加入 clients Set
   |                              |
   |--- request(method, params)-->| authorizeGatewayMethod()
   |                              | handler = coreGatewayHandlers[method]
   |<---- response ---------------|
   |                              |
   |<---- event(broadcast) ------| broadcast() 带背压控制
   |                              |
   |--- close ------------------->| 清理 Presence, Node, Pending
```

### 4.2 RPC 方法注册表

Gateway 通过 `coreGatewayHandlers` 字典注册 25+ 类 RPC 方法:

```
coreGatewayHandlers
  |
  +-- connect*        连接/认证
  +-- chat.*          发送/中止/历史
  +-- agent           Agent 执行
  +-- agents.*        Agent 管理
  +-- channels.*      通道启停/状态
  +-- config.*        配置 CRUD
  +-- models.*        模型目录
  +-- sessions.*      会话查询
  +-- health          健康检查
  +-- status          系统状态
  +-- cron.*          定时任务
  +-- device.*        设备配对
  +-- node.*          远程节点
  +-- exec.*          执行审批
  +-- browser.*       浏览器控制
  +-- tts.*           文本转语音
  +-- skills.*        技能管理
  +-- wizard.*        引导向导
  +-- update.*        版本更新
  +-- send.*          消息发送
  +-- usage.*         用量指标
```

### 4.3 广播与背压

```
broadcast(event, payload, opts)
  |
  for each client in clients:
    |
    +-- hasEventScope(client, event)?  // Scope 守卫
    |     NO --> skip
    |
    +-- bufferedAmount > MAX_BUFFERED_BYTES?
    |     YES + dropIfSlow --> skip (丢弃)
    |     YES + !dropIfSlow --> close(1008, "slow consumer")
    |
    +-- socket.send(frame)  // 正常发送
```

Scope 守卫确保敏感事件(设备配对、执行审批)仅发送给有权限的客户端。

### 4.4 Chat 会话状态

```
ChatRunState
  |
  +-- registry: Map<sessionId, Queue<ChatRunEntry>>
  |     支持每会话多个待执行 run, 队列式消费
  |
  +-- buffers: Map<runId, string>
  |     流式文本累积
  |
  +-- deltaSentAt: Map<runId, number>
  |     150ms 节流控制
  |
  +-- abortedRuns: Map<runId, timestamp>
       中止标记
```

---

## 五、Agent 代理层详解

### 5.1 执行引擎架构

```
agentCliCommand(opts)
  |
  +-- opts.local?
  |     YES --> agentCommand() 直接本地执行
  |     NO  --> agentViaGatewayCommand()
  |               |
  |               +-- callGateway("agent", params)
  |               |     成功 --> 返回结果
  |               |     失败 --> fallback agentCommand()
  |
agentCommand() 本地执行流程:
  |
  1. loadConfig() + resolveAgentId()
  2. resolveSession()            // 会话解析
  3. ensureAgentWorkspace()      // 工作区初始化
  4. resolveConfiguredModelRef() // 模型选择
  5. buildSkillsSnapshot()       // 技能快照
  |
  6. runWithModelFallback()      // 带故障转移的执行
  |     |
  |     for each candidate in [primary, ...fallbacks]:
  |       |
  |       +-- cooldown 检查 (profile 冷却期)
  |       +-- runEmbeddedPiAgent() 或 runCliAgent()
  |       |     |
  |       |     成功 --> return {result, provider, model}
  |       |     FailoverError --> 记录, 尝试下一个
  |       |     其他错误 --> 立即抛出
  |
  7. updateSessionStoreAfterAgentRun()
  8. deliverAgentCommandResult()  // 通过 deps 发送到目标通道
```

### 5.2 模型故障转移

```
runWithModelFallback()
  |
  候选列表: [primary] + [configured fallbacks] - [not in allowlist]
  |
  for (candidate of candidates):
    |
    +-- 检查 provider profile cooldown
    |     所有 profile 冷却中 --> skip
    |
    +-- 尝试执行
    |
    +-- 成功 --> return
    |
    +-- FailoverError?
    |     reason: billing(402) | rate_limit(429) | auth(401/403)
    |             | timeout(408/ETIMEDOUT) | format(400)
    |     YES --> 记录 attempt, continue
    |     NO  --> throw (不可恢复)
    |
  所有候选失败 --> throw 最后一个错误
```

### 5.3 认证 Profile 轮转

```
resolveApiKeyForProvider(providerId)
  |
  优先级:
  1. 显式 profileId 指定
  2. 环境变量 (ANTHROPIC_API_KEY, OPENAI_API_KEY, ...)
  3. AuthProfileStore 中按 order 排序的 profiles
  4. 配置文件中的 apiKey 字段
  |
  返回: { apiKey, profileId, source, mode }
         mode: "api-key" | "oauth" | "token" | "aws-sdk"

AuthProfileStore:
  {
    profiles:  { "profile-1": ApiKeyCredential, "profile-2": OAuthCredential, ... }
    order:     { "agent-main": ["profile-2", "profile-1"] }  // 每 agent 顺序
    lastGood:  { "anthropic": "profile-1" }                  // 上次成功
    usageStats: { "profile-1": { uses, lastUsedAt, ... } }   // 轮转统计
  }
```

---

## 六、Channel 通道层详解

### 6.1 三层抽象

```
+------------------------------------------------------------------+
|  Channel Registry (注册表)                                        |
|  - UI 元数据: 标签, 图标, 文档路径                                |
|  - 排序: CHAT_CHANNEL_ORDER 定义规范顺序                          |
+------------------------------------------------------------------+
         |
+--------v---------------------------------------------------------+
|  Channel Dock (码头)                                              |
|  - 轻量元数据, 不导入重量级实现                                    |
|  - 能力声明: chatTypes, reactions, threads, streaming              |
|  - 配置解析器, 文本限制, 流式合并参数                              |
+------------------------------------------------------------------+
         |
+--------v---------------------------------------------------------+
|  Channel Plugin (插件)                                            |
|  - 完整适配器集合 (15+ 可选适配器)                                |
|  - gateway.startAccount() / stopAccount()                        |
|  - outbound / messaging / groups / mentions / security / ...     |
+------------------------------------------------------------------+
```

### 6.2 通道插件适配器矩阵

```
ChannelPlugin
  |
  +-- config          配置解析 (必选)
  +-- gateway         Gateway 启停 (startAccount/stopAccount)
  +-- outbound        出站消息发送
  +-- messaging       消息格式化/解析
  +-- groups          群组操作
  +-- mentions        @提及解析
  +-- threading       线程/回复支持
  +-- streaming       流式输出控制
  +-- commands        斜杠命令注册
  +-- onboarding      引导流程
  +-- setup           初始化配置
  +-- pairing         设备配对
  +-- security        安全策略
  +-- status          状态检查
  +-- auth            认证流程
  +-- directory       联系人目录
  +-- resolver        ID 解析
  +-- actions         消息操作 (编辑/删除/反应)
  +-- heartbeat       心跳探活
  +-- agentTools      通道特有的 Agent 工具
```

### 6.3 通道生命周期管理

```
createChannelManager()
  |
  startChannel(channelId, accountId?)
    |
    plugin = getChannelPlugin(channelId)
    |
    for each accountId:
      |
      abort = new AbortController()
      store.aborts.set(id, abort)
      |
      task = plugin.gateway.startAccount({
        cfg, accountId, account, runtime,
        abortSignal: abort.signal,      // <-- 优雅停机信号
        log, getStatus, setStatus
      })
      |
      store.tasks.set(id, tracked(task))
  |
  stopChannel(channelId, accountId?)
    |
    abort.abort()                       // 发出停机信号
    plugin.gateway.stopAccount(...)     // 插件清理
    await task                          // 等待任务完成
```

---

## 七、消息路由层详解

### 7.1 路由解析级联

消息从通道到达后, 通过级联匹配确定目标 Agent:

```
resolveAgentRoute(channel, peer, cfg)
  |
  1. 过滤 bindings: 匹配 channel + accountId
  |
  2. peer 匹配       --> 为特定联系人绑定专属 Agent
  |    (未命中)
  3. guild 匹配      --> 为 Discord 服务器绑定 Agent
  |    (未命中)
  4. team 匹配       --> 为 Teams 团队绑定 Agent
  |    (未命中)
  5. account 匹配    --> 为特定账号绑定默认 Agent
  |    (未命中)
  6. 通配符账号匹配  --> 通道级别默认 Agent
  |    (未命中)
  7. 全局默认 Agent
  |
  返回: { agentId, channel, accountId, sessionKey, matchedBy }
```

### 7.2 Session Key 构建

Session Key 决定了会话的**隔离粒度**:

```
dmScope 模式:
  |
  "main"
  |  所有 DM 共享一个会话
  |  key: agent:{agentId}:main
  |
  "per-peer"
  |  每个联系人独立会话, 跨通道共享
  |  key: agent:{agentId}:dm:{peerId}
  |
  "per-channel-peer"
  |  每个通道的每个联系人独立
  |  key: agent:{agentId}:{channel}:dm:{peerId}
  |
  "per-account-channel-peer"
     完全隔离
     key: agent:{agentId}:{channel}:{accountId}:dm:{peerId}

群组/线程:
  key: agent:{agentId}:{channel}:{peerKind}:{peerId}

身份链接 (resolveLinkedPeerId):
  config 可将多个 ID 映射到规范身份
  例: WhatsApp +1234... + Telegram @alice --> "alice"
```

---

## 八、插件系统详解

### 8.1 发现 -> 加载 -> 注册

```
discoverOpenClawPlugins()
  |
  四个来源 (优先级从高到低, 同名首次发现胜出):
  |
  1. config 指定路径     cfg.plugins.paths[]
  2. workspace 扩展      .openclaw/extensions/
  3. 全局扩展            ~/.openclaw/extensions/
  4. 捆绑扩展            {install_dir}/extensions/
  |
  返回: PluginCandidate[] + diagnostics

loadOpenClawPlugins(candidates)
  |
  1. 配置 jiti 模块加载器, 注册 "openclaw/plugin-sdk" 别名
  |
  2. for each candidate:
  |     |
  |     +-- 检查 enable/disable 状态
  |     +-- jiti(candidate.source)  // 直接加载 .ts 文件
  |     +-- 解析 default export
  |     +-- 校验 configSchema (JSON Schema)
  |     +-- 校验 memory slot (仅允许一个 memory 插件)
  |     +-- 调用 plugin.register(api)
  |
  3. 缓存 registry (按 workspace + config hash)

createPluginRegistry()
  |
  注册方法:
  +-- registerTool(factory, opts)        Agent 工具
  +-- registerHook(event, handler, opts) 生命周期钩子
  +-- registerGatewayMethod(method, fn)  WS RPC 扩展
  +-- registerChannel(plugin)            通道插件
  +-- registerProvider(provider)         模型提供者
  +-- registerHttpRoute(route)           HTTP 端点
  +-- registerCliCommand(register)       CLI 命令
```

### 8.2 Hook 系统

```
Hook 类型:
  |
  +-- Void Hook (并行触发, 无返回值合并)
  |     message_received, agent_end, session_start,
  |     gateway_start, message_sent, after_tool_call
  |
  +-- Modifying Hook (按优先级串行, 结果向下传递)
  |     before_agent_start, message_sending,
  |     before_tool_call, before_compaction
  |
  执行顺序: priority 降序 (数值越高越先执行)

before_agent_start hook 示意:
  |
  plugin-A (priority=100): 注入 systemPrompt 片段
       |
       v
  plugin-B (priority=50):  注入 prependContext
       |
       v
  plugin-C (priority=10):  修改 tools 列表
       |
       v
  最终合并结果传给 Agent 执行引擎
```

---

## 九、基础设施层

### 9.1 配置管理

```
配置加载流程:
  resolveConfigPath()
    |  检查 OPENCLAW_STATE_DIR / OPENCLAW_CONFIG_PATH
    |  回退到 ~/.openclaw/openclaw.json
    v
  readConfigFileSnapshot()
    |  JSON5 解析 (支持注释、尾逗号)
    |  环境变量替换
    v
  validateConfigObjectWithPlugins()
    |  Zod schema 校验
    |  插件 configSchema 校验
    v
  OpenClawConfig (已校验的类型安全对象)

配置持久化:
  - 原子写入: 写临时文件 -> rename
  - 备份轮转: 最近 5 个备份
  - Hash 变更检测: SHA256
  - 版本戳: lastTouchedVersion + lastTouchedAt
```

### 9.2 设备身份

```
ED25519 密钥对管理:
  |
  ~/.openclaw/identity/device.json (mode 0o600)
  {
    version: 1,
    deviceId: SHA256(publicKeyRawBytes),
    publicKeyPem,
    privateKeyPem,
    createdAtMs
  }
  |
  signDevicePayload(payload, privateKey) --> base64url 签名
  verifyDeviceSignature(payload, signature, publicKey) --> boolean
```

### 9.3 媒体处理管道

```
消息文本
  |
  parseMediaTokens()           提取 "MEDIA: <url>" 标记
  |                            跳过 fenced code block 内的标记
  v
  fetchRemoteMedia()           下载远程媒体 (5MB 限制)
  |                            Content-Disposition 解析
  v
  sanitizeFilename()           Unicode 安全的文件名清洗
  |
  v
  applyMediaUnderstanding()    多模态理解
  |
  +-- Image: OpenAI/Anthropic/Google/Minimax vision 模型
  +-- Audio: OpenAI/Groq/Deepgram/Google 语音转写
  +-- Video: Google Gemini 视频理解
  |
  v
  格式化结果注入 Agent 上下文
```

### 9.4 记忆系统

```
Memory Manager (SQLite + sqlite-vec)
  |
  索引结构:
  +-- chunks_vec   向量嵌入 (Float32 blob)
  +-- chunks_fts   全文搜索索引 (FTS5)
  +-- embedding_cache  嵌入缓存
  |
  搜索流程:
  query --> [向量搜索] + [BM25 关键词搜索]
                  |                |
                  +--- 混合排序 ---+
                         |
                    MemorySearchResult[]
  |
  嵌入提供者:
  +-- OpenAI (text-embedding-3-small)
  +-- Google Gemini (embedding-001)
  +-- 本地 (node-llama-cpp + embeddinggemma-300M)
  |
  批处理: 8000 tokens/批, 3 次重试, 指数退避
```

### 9.5 安全审计

```
安全审计框架:
  |
  +-- collectAttackSurfaceSummaryFindings()  攻击面分析
  +-- collectFileSystemFindings()            文件权限检查
  +-- collectSecretsInConfigFindings()       配置中的明文凭证
  +-- collectPluginsTrustFindings()          插件信任评估
  +-- collectExposureMatrixFindings()        通道暴露矩阵
  |
  输出: { severity: "info"|"warn"|"critical", remediation }
  |
  自动修复: safeChmod() 原子权限修正
```

---

## 十、模块依赖关系

```
                entry.ts
                   |
              run-main.ts
              /         \
         route.ts    build-program.ts
                          |
                   command-registry.ts
                   /      |       \
          register.*   preaction   deps.ts
              |            |         |
         commands/*    config/*   channels/*
              |            |         |
              +-----+------+---------+
                    |
              gateway/server.impl.ts
              /    |     |      \
        ws-*   methods  chat   channels
         |       |       |        |
        auth   agent   broadcast  plugin
         |       |       |        |
         +-------+---+---+--------+
                     |
               agents/agent.ts
              /      |       \
       model-*   pi-runner   auth-profiles
         |          |            |
    providers/*  pi-tools    failover
         |          |            |
         +----+-----+-----+-----+
              |            |
         media/*      memory/*
              |            |
        media-understanding/
              |
        link-understanding/
```

模块间的核心依赖原则:

1. **上层依赖下层, 下层不感知上层** — Gateway 依赖 Agent, Agent 不感知 Gateway
2. **插件通过 SDK 接口交互** — 插件不直接 import 核心模块, 而是通过 `openclaw/plugin-sdk`
3. **配置贯穿所有层** — `OpenClawConfig` 作为只读数据对象在各层间传递
4. **通道通过适配器解耦** — 核心不依赖具体通道实现, 通过 `ChannelPlugin` 接口交互

---

## 十一、核心设计模式总结

| 模式 | 应用位置 | 解决的问题 |
|------|----------|------------|
| **Route-First** | CLI 启动 | 简单命令绕过完整初始化, 降低延迟 |
| **Lazy Loading** | 子命令/插件注册 | 仅加载需要的模块, 减少启动内存 |
| **Factory + DI** | CliDeps, ChannelManager | 解耦创建与使用, 便于测试 |
| **级联匹配** | 消息路由 | peer->guild->team->account->default 灵活路由 |
| **故障转移循环** | 模型执行 | 多候选模型+多认证profile, 最大化可用性 |
| **并行/串行 Hook** | 插件系统 | void hook 并行提高吞吐; modifying hook 串行保证顺序 |
| **背压控制** | WebSocket 广播 | 慢客户端丢弃或断开, 防止内存溢出 |
| **轻量 Dock** | 通道元数据 | 避免导入重量级通道实现, 加速初始化 |
| **原子写入** | 配置/会话持久化 | 防止写入中断导致数据损坏 |
| **Scope 守卫** | Gateway 事件分发 | 敏感事件仅发送给授权客户端 |
| **Profile 轮转** | 认证管理 | 多 API key 自动轮转, 规避单点限流 |
| **Session Key 分级** | 会话隔离 | 4 级隔离粒度, 适配不同场景需求 |

---

## 十二、对 AI Agent 架构设计的启示

### 12.1 分层解耦

OpenClaw 的分层结构提供了一个经过验证的参考:

```
接入层 (Channel)     — 协议适配, 消息标准化
网关层 (Gateway)     — 连接管理, 认证鉴权, 事件分发
路由层 (Routing)     — 消息分发, 会话隔离
代理层 (Agent)       — 模型调度, 工具执行, 上下文管理
提供者层 (Provider)  — 模型抽象, 认证封装
基础设施层 (Infra)   — 配置, 持久化, 安全, 日志
```

每层通过接口/类型契约交互, 不直接依赖具体实现。

### 12.2 关键架构决策

1. **Gateway 作为控制平面** — 所有通道消息经 Gateway 汇聚, 而非各通道直连 Agent。这简化了状态管理, 代价是增加一跳延迟。

2. **模型调度与执行分离** — `model-selection` 负责选择, `model-fallback` 负责容错, `pi-runner` 负责执行。三者职责分明, 故障转移逻辑不侵入执行引擎。

3. **插件 SDK 边界** — 插件通过 `openclaw/plugin-sdk` 导入类型和工具, 核心通过 `PluginRegistry` 提供注册接口。这种双向隔离使插件生态可以独立演化。

4. **会话隔离分级** — 4 级 Session Key 粒度(main/per-peer/per-channel-peer/per-account-channel-peer)允许在共享上下文和隔离之间灵活选择, 无需修改核心逻辑。

5. **认证 Profile 池** — 多个 API key 组成池, 按使用统计轮转, 单个 key 限流时自动切换。这是生产环境中应对 rate limit 的务实方案。

### 12.3 值得借鉴的工程实践

- **jiti 加载 .ts 插件** — 无需预编译, 开发体验好
- **JSON5 配置格式** — 支持注释, 对用户友好
- **原子文件操作** — 所有持久化写入使用 temp+rename 模式
- **敏感信息脱敏** — 日志中自动检测并遮蔽 API key、token 等
- **ED25519 设备身份** — 零配置的设备级认证, 无需用户管理证书

---

*报告基于 OpenClaw 2026.1.29 源码分析生成。*
