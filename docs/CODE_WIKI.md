# xiaozhi-esp32-server-java 代码 Wiki

> 面向新人的代码导航文档。目标是：读完本文即可理解项目全貌、找到关键代码位置、开始参与开发。
>
> 阅读顺序建议：第 1~3 章建立全局认知 → 第 4 章按需深入模块 → 第 5~6 章理解核心机制 → 第 7 章动手扩展。

---

## 目录

1. [项目概览](#1-项目概览)
2. [快速上手](#2-快速上手)
3. [模块总览与依赖关系](#3-模块总览与依赖关系)
4. [各模块详解](#4-各模块详解)
5. [核心流程详解](#5-核心流程详解)
6. [关键设计决策与设计模式速查](#6-关键设计决策与设计模式速查)
7. [扩展开发指南](#7-扩展开发指南)
8. [工程规范](#8-工程规范)

---

## 1. 项目概览

### 1.1 这是什么项目

一个 ESP32 智能语音对话设备的 Java 服务端，实现「设备语音输入 → 语音识别 → LLM 对话 → 语音合成 → 设备语音播放」的完整闭环，并配套管理后台（设备/角色/模型配置/对话记录）。

### 1.2 技术栈

| 层 | 技术 |
|---|---|
| 语言/框架 | Java 21、Spring Boot 3.5（虚拟线程已开启） |
| 构建 | Maven 多模块（`xiaozhi-parent` 聚合） |
| 数据库 | MySQL 8.0 + MyBatis-Plus（Flyway 管理迁移，仅 server 进程执行） |
| 缓存/协调 | Redis 7（Lettuce 池 + Redisson 分布式锁） |
| AI 框架 | Spring AI 1.1.4（`spring-ai-bom` 统一版本） |
| 认证 | Sa-Token（JWT Simple 模式 + Redis 会话） |
| 实时通信 | WebSocket（设备主链路）、Redis Pub/Sub（跨实例） |
| 本地 AI | Silero VAD (ONNX)、Vosk (STT)、sherpa-onnx (TTS)、Opus 编解码（concentus 纯 Java） |
| 前端 | Vue 3 + TypeScript + Ant Design Vue + Pinia + Vite（`web/` 目录） |
| 测试 | 后端 JUnit/Maven；前端 Vitest + Playwright |

### 1.3 双进程架构（最重要的全局概念）

两个独立的 Spring Boot 应用共享 MySQL + Redis，各自打包、独立部署：

| 进程 | 模块 | 端口 | 角色 |
|---|---|---|---|
| 管理服务 | `xiaozhi-server` | 8091 | REST API、用户/设备/角色管理、OTA、Flyway 迁移、Web 聊天 |
| 对话服务 | `xiaozhi-dialogue` | 8092 | WebSocket 音频流、VAD/STT/LLM/TTS 对话管道 |

```
                 ┌─────────────┐         ┌──────────────────┐
   ESP32 设备 ───│ xiaozhi-    │  OTA    │  xiaozhi-server  │ ←── Web 前端 (Vue3)
      │          │ dialogue    │◄────────│      :8091       │
      └─WebSocket│    :8092    │  配置下发 └────────┬─────────┘
                 └──────┬──────┘                  │
                        │    共享 MySQL + Redis    │
                        └────────────┬────────────┘
```

关键机制：
- **dialogue 水平扩展**：每个 dialogue 实例启动时向 Redis 自注册（`DialogueServerRegistrar`），server 通过 OTA 配置把设备分配到不同实例。
- **跨实例事件**：server 修改配置后经 Redis Pub/Sub（`RedisBroadcast`）通知所有 dialogue 实例热更新，无需重启。
- **模块组装靠显式 `@ComponentScan`**：不是自动发现，新增包必须改启动类注解（见 4.5/4.6）。

---

## 2. 快速上手

### 2.1 环境准备

- JDK 21、Maven 3.8+、MySQL 8.0、Redis 7、Node.js 18+（前端）
- 模型文件与本地库**不在 git 内**，首次运行前执行（Git Bash/WSL）：

```bash
scripts/download_models.sh     # 全量：VAD + Vosk STT + sherpa-onnx TTS + native 库
# 或 scripts/download_base.sh  # 仅 VAD（最小可运行）
```

产物落在 `models/` 与 `lib/`（Vosk、sherpa-onnx 的 JNI 原生库）。

### 2.2 配置

- 主配置：各模块 `src/main/resources/application.yml`，环境差异在 `application-dev.yml` / `application-prod.yml`
- 数据库/Redis 连接等敏感信息走 `.env` 环境变量
- 修改数据库结构**只能新增 Flyway 迁移文件**（`xiaozhi-server/src/main/resources/db/migration/`），禁止手改 SQL

### 2.3 构建与运行

```bash
# 后端全量构建（项目根目录）
mvn clean install -DskipTests

# 启动管理服务（先启动它，Flyway 会自动建表）
java -Djava.library.path=lib -jar xiaozhi-server/target/xiaozhi-server-*.jar

# 启动对话服务
java -Djava.library.path=lib -jar xiaozhi-dialogue/target/xiaozhi-dialogue-*-exec.jar

# 或用脚本（Git Bash / WSL）
bin/all.sh start    # 编译并启动两个服务
bin/all.sh status   # 查看运行状态
```

注意：dialogue 打包为 `*-exec.jar`（classifier exec），其他模块是标准 jar。

### 2.4 前端

```bash
cd web
npm install
npm run dev          # Vite 开发服务器
npm run test:run     # Vitest 单测
npm run lint         # oxlint + eslint
npm run type-check   # TS 类型检查
```

### 2.5 验证

- 管理后台：浏览器访问 `http://localhost:8091`（Vue 前端由 server 托管）或 `npm run dev` 直连
- API 文档：`http://localhost:8091/swagger-ui.html`（Knife4j + SpringDoc）
- 设备连接：WebSocket `ws://<dialogue-ip>:8092/ws/xiaozhi/v1/`（路径常量 `ServerAddressProvider.WS_PATH`），握手时携带 `device-id` 头

---

## 3. 模块总览与依赖关系

### 3.1 依赖方向图（自下而上）

```
xiaozhi-common          ← 契约层：枚举/事件/DTO/跨进程接口，无业务实现
        ▲
xiaozhi-service         ← 业务领域层：DDD 领域 + MyBatis + 存储安全等基础设施
        ▲
xiaozhi-ai              ← AI 能力层：LLM/TTS/STT Provider + 记忆 + 工具调用
        ▲
┌───────┴────────┐
xiaozhi-dialogue  xiaozhi-server     ← 两个进程入口，各自组装上面的模块

web/                ← Vue3 前端（独立构建，调 server 的 REST 与 dialogue 的 WebSocket）
```

### 3.2 各模块一句话职责

| 模块 | 职责 |
|---|---|
| `xiaozhi-common` | 共享契约：实体/DTO、14 个领域事件、`DeviceState` 状态机、`ServerAddressProvider`、`DialogueServerRegistry`、`RedisBroadcast`、Opus 音频工具 |
| `xiaozhi-service` | 22 个领域包：device/role/config/template 完整 DDD；user/message 等 DAL+Service 简化结构；storage/token/security 基础设施 |
| `xiaozhi-ai` | Provider 模式的 LLM(7)/TTS(8)/STT(7) 提供商、记忆（窗口/摘要）、`XiaoZhiToolCallingManager`、MCP 工具注册 |
| `xiaozhi-dialogue` | WebSocket 接入、VAD→STT→LLM→TTS→Player 对话管道、会话管理、设备自注册 |
| `xiaozhi-server` | REST 控制器（按领域分包）、Flyway、Sa-Token 安全、OTA、Web 聊天 |
| `web/` | Vue3 管理界面 + Web 聊天页 |

---

## 4. 各模块详解

### 4.1 xiaozhi-common —— 契约层

两个进程共同依赖的最底层模块，**只放契约，不放实现**。

核心内容：

```
com.xiaozhi
├── communication/
│   ├── ServerAddressProvider.java        # 地址组装：有域名→wss://ws.{domain}；无→ws://{ip}:{port}
│   ├── registry/DialogueServerRegistry   # 接口：register/unregister/heartbeat/selectServer
│   └── common/RedisBroadcast             # Spring事件 → Redis Pub/Sub 广播（6个频道）
├── event/                                # 14 个领域事件（DeviceOnlineEvent、SpeechRecognizedEvent...）
├── enums/DeviceState                     # IDLE → LISTENING → THINKING → SPEAKING 状态机
├── common/
│   ├── Speech                            # TTS 播放对象（PCM/Opus + 文本 + mood），ofOpus() 免转码
│   ├── web/ApiResponse                   # 统一响应封装
│   └── port/                             # TokenResolver、ConfigLookup ← 依赖倒置端口（service 实现）
└── utils/                                # AudioUtils、OpusProcessor（concentus 纯 Java Opus 编解码）等 11 个
```

**依赖倒置设计**：common 定义 `port/TokenResolver`、`port/ConfigLookup` 端口接口，实现在 service 模块，供 ai 模块回调而不产生反向依赖。

### 4.2 xiaozhi-service —— 业务领域层

**DDD 三档完整度**（22 个顶层包）：

**A 档：完整 DDD（聚合根 + Repository）**——`device`、`role`、`config`、`template`

以 `device` 为例的标准结构（其他领域照此模板）：

```
com.xiaozhi.device/
├── dal/mysql/
│   ├── dataobject/DeviceDO.java          # MyBatis-Plus 实体（继承 common 的 BaseDO）
│   └── mapper/DeviceMapper.java
├── domain/
│   ├── Device.java                       # 聚合根：无 public setter，行为方法 + DomainSignal 收集
│   └── repository/DeviceRepository.java  # save() 时把 DomainSignal 转译为 Spring 事件
├── infrastructure/
│   ├── DeviceRepositoryImpl.java
│   └── convert/DeviceConverter.java      # DO ↔ Domain
├── service/
│   ├── DeviceService.java                # 接口（方法签名只允许 BO/Resp，禁用 *Req —— ArchUnit 强制）
│   └── impl/DeviceServiceImpl.java
└── convert/DeviceConvert.java            # MapStruct：DO ↔ BO/Resp
```

**B 档：DAL + Service 简化结构**——`user`、`userauth`、`authrole`、`permission`、`message`、`summary`、`agent`、`operationlog` 等

**C 档：基础设施**——`storage`（本地/阿里云 OSS/腾讯 COS，策略工厂）、`token`（第三方 Token 管理）、`security`（Sa-Token `StpInterfaceImpl` + `@CheckOwner` 属主校验 AOP）、`communication.registry`（`RedisDialogueServerRegistry` 实现 common 接口）

关键点：
- `map-underscore-to-camel-case: false`——MyBatis 列名直接映射，**写 SQL 时注意列名要和 DO 字段完全一致**
- 领域事件流：`Device` 聚合根收集 `DomainSignal` → `Repository.save()` 转译 Spring 事件 → `RedisBroadcast` 跨实例广播 → 所有 dialogue 实例热更新

### 4.3 xiaozhi-ai —— AI 能力层

六大子包：

```
com.xiaozhi.ai
├── llm/          # ChatModelProvider 接口 + 7 个提供商 + memory/ 记忆子包
├── tts/          # TtsService 接口 + 8 个提供商 + TtsServiceAdapter
├── stt/          # SttService 接口 + 7 个提供商
├── tool/         # XiaoZhiToolCallingManager（工具调用核心）
├── mcp/          # MCP 配置/注册器
└── utils/
```

**LLM 提供商**（`llm/factory/providers/`）：

| Provider | 名称 | 实现方式 |
|---|---|---|
| `OpenAiModelProvider` | openai | spring-ai-openai |
| `OllamaModelProvider` | ollama | spring-ai-ollama |
| `ZhiPuModelProvider` | zhipu | spring-ai-zhipuai |
| `DifyModelProvider` | dify | 自研 `DifyChatModel` |
| `CozeModelProvider` | coze | 自研 `CozeChatModel` |
| `XingHuoModelProvider` | xinghuo | 自研（OkHttp SSE） |
| `XingChenModelProvider` | xingchen | 自研 |

`ChatModelFactory` 路由链：**精确匹配 provider 名 → 回退 openai（兼容 OpenAI 协议端点）→ 抛异常**。

**记忆层**（`llm/memory/`）——自研 `ChatMemory` 接口（**有意放弃** Spring AI 的 ChatMemory）：

- `Conversation` 基类：消息容器（不管持久化），`ownerId+roleId+sessionId` 唯一
- `MessageWindowConversation`：滑动窗口，按**对话组**裁剪（工具组 4 条整体移动，避免孤儿 ToolResponse）
- `SummaryConversation`：超阈值时 `Thread.startVirtualThread()` 异步调 LLM 生成摘要，存 `sys_summary` 表后移除已摘要消息
- `DatabaseChatMemory`：`MessageBO` ↔ Spring AI Message 互转（含工具链重建）
- `ConversationFactory`：按 `role.getMemoryType()` 分派 `"summary"` / `"window"`

**STT**（`stt/`）：`VoskSttService`（本地，默认兜底）、FunASR、Aliyun、Aliyun-NLS、Tencent、Volcengine、Xfyun。`SttServiceFactory` 缓存键 `provider:configId`。

**TTS**（`tts/`）：`EdgeTtsService`（默认兜底）、`SherpaOnnxTtsService`（本地）、Aliyun、Aliyun-NLS、Tencent、Volcengine、Xfyun、MiniMax。`TtsServiceFactory` 缓存键含 voiceName/pitch/speed（5 段）。`TtsServiceAdapter` 是适配器，把 TtsService 桥接到 Spring AI `TextToSpeechModel`。

**工具调用核心**（`tool/XiaoZhiToolCallingManager`）——必读：

1. **流式 tool call 分片合并修复**：修复 Spring AI 已知 issue（流式续传 chunk id 为 `""` 导致 tool call 被拆散）
2. **幻觉工具防护**：模型调用未注册工具时返回错误信息让模型自行总结，不崩流
3. `ToolSession.addToolCallMessages()` 把工具链中间消息存入会话，供 Persona 注入 Conversation

**工具会话隔离**：每个会话独立 `ToolsSessionHolder`（`Map<String, ToolCallback>`），工具不跨会话共享。`ToolsGlobalRegistry` + `GlobalToolRedisRegistry`（Redis key `xiaozhi:system-global-tools`）解决全局工具跨进程可见性（server 进程前端读取展示）。

### 4.4 xiaozhi-dialogue —— 对话服务

```
com.xiaozhi
├── DialogueApplication                   # 启动类 (:8092)
├── dialogue/
│   ├── DialogueService                   # VAD→STT→分发 + abort 打断（详见 5.1）
│   ├── audio/                            # VadService / AecService / SileroVadModel
│   ├── llm/factory/PersonaFactory        # Persona 组装工厂
│   ├── playback/                         # Player / ScheduledPlayer / FileSynthesizer / OpusRecorder
│   └── runtime/                          # Persona / DialogueContext / DialogueTurn
└── communication/
    ├── server/websocket/WebSocketHandler # 设备接入入口
    ├── common/                           # SessionManager / ChatSession / MessageHandler
    ├── domain/                           # Message 密封类层级（消息协议）
    └── message/MessageSender             # 统一出站消息
```

**消息协议**（`communication/domain/`）：`Message` 是 `sealed abstract class`，Jackson 按 `type` 字段多态路由：

| type | 类 | 方向 | 说明 |
|---|---|---|---|
| hello | `HelloMessage` / `HelloMessageResp` | 双向 | 握手协商音频参数（opus/16kHz/单声道/60ms） |
| listen | `ListenMessage` | 设备→服务 | Start/Stop/Text/Detect 四种状态 |
| iot | `IotMessage` | 设备→服务 | 设备 IoT 能力与状态上报 |
| abort | `AbortMessage` | 设备→服务 | 设备主动打断 |
| goodbye | `GoodbyeMessage` | 设备→服务 | 会话结束 |
| mcp | `DeviceMcpMessage` | 双向 | 设备侧 MCP 工具 |

文本帧承载控制消息，二进制帧承载 Opus 音频。

### 4.5 xiaozhi-server —— 管理服务

启动类 `XiaozhiApplication`，`@ComponentScan` 显式列出：common 全部 + service 全部 + ai 全部 + 自身 `file/mcpserver/memory/music/server` 包；`@MapperScan` 14 个 mapper 包。**新增包必须同步改这两个注解。**

控制器**按领域分包**（不是集中 controller 包）：

| Controller | 路由 | 职责 |
|---|---|---|
| `UserController` | `/api/user` | 登录/注册/token 刷新/验证码 |
| `AuthRoleController` | `/api/auth-role` | 后台 RBAC 角色 |
| `DeviceController` | `/api/device` | 设备 CRUD + **OTA 端点（免鉴权）** |
| `RoleController` | `/api/role` | 对话角色（声音/提示词/模型绑定） |
| `TemplateController` | `/api/template` | 提示词模板 |
| `ConfigController` | `/api/config` | LLM/TTS/STT 模型配置 |
| `AgentController` | `/api/agent` | Coze/Dify 智能体 |
| `MessageController` | `/api/message` | 对话记录查询/导出 |
| `MemoryController` | `/api/memory` | 摘要记忆管理 |
| `McpToolController` | `/api/mcpTool` | 工具启用/禁用 |
| `WebChatController` | `/api/chat` | Web 聊天（open/stream/close） |
| `FileUploadController` / `MusicController` | `/api/file` | 上传 / 音乐服务 |

**Flyway 迁移**：`db/migration/V<版本号>__<下划线描述>.sql`，当前 V1~V10，仅 server 进程执行。

**安全纵深**（4 层）：
1. `SaTokenConfig`：`SaInterceptor` 拦截 `/api/**` 强制登录（JWT Simple 模式）
2. `@SaCheckPermission` 注解 → `StpInterfaceImpl` 查权限 key
3. `@CheckOwner` AOP → `OwnershipAspect` 数据属主校验（防越权）
4. `SecurityHeaderFilter` + `RateLimitInterceptor`

### 4.6 web/ —— 前端

```
web/src/
├── views/           # 页面（DashboardView、DeviceView、RoleView、config/ 四个配置页、ChatView...）
├── components/      # 通用组件（含 chat/ 子目录）
├── composables/     # use 前缀组合式函数（~28 个：useAuth、useWebSocket...）
├── services/        # API 层，与后端 Controller 1:1 映射（user.ts/device.ts/role.ts...）
├── store/           # Pinia Setup Store（useUserStore/useAppStore/useDeviceStore/useLoadingStore）
├── router/          # routes.ts + guards.ts（路由级权限校验）
├── directives/      # permission.ts 自定义指令（组件级权限）
└── constants/api.ts # 所有后端路径常量
```

权限双通道：路由级 `guards.ts` 校验 `meta.permission`，组件级 `v-permission` 指令按 `permissionKey` 控制显隐——与后端 `@SaCheckPermission` 的 key（如 `system:device`）三方对齐。

Web 聊天：`services/chat.ts` 调 `/api/chat/*`（HTTP），`services/websocket.ts` 直连 dialogue 的 WebSocket。

---

## 5. 核心流程详解

### 5.1 对话管道（核心数据流，必须理解）

```
ESP32 设备 → WebSocket → 二进制 Opus 帧
  → WebSocketHandler.handleBinaryMessage
  → DialogueService.processAudioData
      → VadService（Opus解码 → AEC → Silero ONNX 推理；预缓冲500ms，静音800ms判停）
      │
      ├─ SPEECH_START:  ① startStt（同步建流防竞态）
      │                 ② 若 persona.isActive() → abortDialogue（语音打断TTS）
      ├─ SPEECH_CONTINUE: session.sendAudioData → 喂 STT 流
      └─ SPEECH_END:     completeAudioStream + 转换 THINKING
  → STT 流式识别（虚拟线程）→ sendStt 回显设备 → 发布 SpeechRecognizedEvent
  → 存用户音频 WAV → handleText
      → 构建 UserMessage（情感挂 metadata，不拼文本前缀）
      → IntentService（EXIT 意图 → 播报告别语）
      → persona.chat(userMessage, true)
          → conversation.add → chatModel.stream(Prompt + ToolCallingChatOptions)
          → LLM 流式响应 → 过滤 thinking 内容
          → FileSynthesizer：SentenceHelper 分句 → 逐句 TTS 产文件 → Flux<Speech>
          → ScheduledPlayer.play：虚拟线程 sendFramesLoop（60ms 绝对时间调度）
          → sendOpusFrame → 设备
```

**关键类定位**：

| 类 | 位置 | 职责 |
|---|---|---|
| `WebSocketHandler` | `dialogue/communication/server/websocket/` | 接入、握手、分发 |
| `SessionManager` | `dialogue/communication/common/` | sessionId↔ChatSession 主表 + deviceId 反向索引 |
| `DialogueService` | `dialogue/dialogue/` | 音频接收/VAD/STT 管理/打断 |
| `Persona` | `dialogue/dialogue/runtime/` | **聚合 ChatModel+Synthesizer+Player+Conversation**，chat() 是对话循环 |
| `PersonaFactory` | `dialogue/dialogue/llm/factory/` | 按角色配置组装 Persona |
| `ScheduledPlayer` | `dialogue/dialogue/playback/` | 虚拟线程 + Burst 模式调度播放 |
| `FileSynthesizer` | `dialogue/dialogue/playback/` | LLM token 流 → 分句 → TTS 文件 |

### 5.2 设备连接生命周期

```
1. WebSocket 握手（device-id 头）→ afterConnectionEstablished
2. 注册会话（SessionManager）→ 幽灵会话清理（跨实例 Redis 通知旧实例关闭）
3. 查设备 → 绑定 deviceId→sessionId 索引 → 发布 DeviceOnlineEvent
4. 已绑定角色 → initializeBoundDevice：
     ToolsSessionHolder（工具隔离）→ PersonaFactory.buildPersona → AEC 初始化 → 状态 ONLINE
5. 设备发 hello JSON → 协商音频参数 → 回 HelloMessageResp（opus/16kHz/1ch/60ms）
   （features.mcp=true 时虚拟线程异步初始化设备侧 MCP 工具）
6. 二进制帧 → 进入对话管道（5.1）
7. 关闭 → 异步更新离线（含重连时序保护）→ closeSession → 清 VAD/AEC
```

未绑定设备的处理：`user_chat_` 前缀的 Web 虚拟设备自动创建绑定；未命名实体设备用默认 TTS 播报验证码（CAS 防并发重复生成）。

### 5.3 打断机制（abort，顺序敏感）

`DialogueService.abortDialogue()` **严格顺序**：

```
① closeAudioStream + 转换 LISTENING（VAD 打断时跳过——startStt 已建新流）
② persona.getSynthesizer().cancel()   # 先取消上游 Flux 订阅
③ player.stop()                        # 再停止播放、清空队列
④ sendTtsMessage(session, null, "stop")# 通知设备切回聆听
⑤ 处理 goodbye 回调（若有）
```

**为什么先 ② 后 ③**：若不先取消 Synthesizer，SentenceHelper 会继续分句并 `player.play(newFlux)`，导致新旧音频重叠。

触发源（统一经 `ChatAbortedEvent` 事件解耦）：设备 abort 消息、listen Text 输入、goodbye 清理、Redis 跨实例指令、VAD 检测到用户说话。

### 5.4 播放调度（ScheduledPlayer）

- 每个播放器独立虚拟线程 `sendFramesLoop`，支持海量并发
- **Burst 预缓冲**：`playPosition` 初始 -120ms（2 帧），前 2 帧立即发送，防首帧破音
- **绝对时间调度**：`targetSendTime = startTimestamp + playPosition`，60ms/帧，无累积误差
- 句子间隔 300ms（`SENTENCE_GAP_MARKER` 空帧标记）
- `hasContent()` 比单纯 `isPlaying` 判断更全面（含队列/订阅状态），防漏打断
- `OpusRecorder` 组合进 Player：下发帧同步写 OGG 留档

### 5.5 跨实例通信

| 机制 | 用途 |
|---|---|
| `DialogueServerRegistrar` | dialogue 实例启动自注册 Redis，心跳 TTL 续期 |
| `RedisBroadcast` | 6 个频道：`xiaozhi:role-changed`、`xiaozhi:config-changed`、`xiaozhi:close-session` 等 |
| `DeviceRegistry` + `InstanceIdHolder` | 实例内设备表，识别幽灵会话 |

---

## 6. 关键设计决策与设计模式速查

### 6.1 必须遵守的架构决策

1. **Persona 拥有对话循环**：LLM、TTS、Player、Conversation 全部聚合在 Persona 内，Session 持有 Persona 引用而非各组件
2. **设备状态机**：`DeviceState` 枚举（IDLE→LISTENING→THINKING→SPEAKING）替代分散布尔标志，`transitionTo()` 带日志
3. **工具会话隔离**：`ToolsSessionHolder` 每会话独立，ToolContext 只传 sessionId（避免序列化问题）
4. **模型文件不入 git**：通过 scripts 下载
5. **Flyway 独占 DDL**：永远只加迁移文件
6. **Service 层禁用 `*Req` DTO**：Controller 解包为 BO 再调 Service（ArchUnit 测试强制）

### 6.2 设计模式速查表

| 模式 | 落点 |
|---|---|
| 策略 + 工厂 | `ChatModelFactory` / `TtsServiceFactory` / `SttServiceFactory`（Provider 注册 + 名称路由 + 兜底回退） |
| 适配器 | `TtsServiceAdapter`（TtsService → Spring AI TextToSpeechModel） |
| 依赖倒置 | common `port/` 接口 ← service 实现 |
| 装饰/扩展 | `XiaoZhiToolCallingManager` 扩展 Spring AI `DefaultToolCallingManager` |
| 模板方法 | `Conversation` 基类，子类覆写窗口/摘要策略 |
| 观察者/事件 | 14 个领域事件 + `RedisBroadcast` 事件桥 |
| 组合 | `OpusRecorder` 组合进 `Player`（替代继承） |
| 密封类 | `Message` 协议层级（Jackson 多态） |
| AOP | `OwnershipAspect`（属主校验）、`AuditLogAspect`（审计日志） |

---

## 7. 扩展开发指南

### 7.1 新增 LLM Provider

1. 实现 `ChatModelProvider` 接口（`xiaozhi-ai/src/main/java/com/xiaozhi/ai/llm/factory/`）：

```java
@Component
public class XxxModelProvider implements ChatModelProvider {
    @Override
    public String getProviderName() { return "xxx"; }

    @Override
    public ChatModel createChatModel(ConfigBO config, RoleBO role) {
        // 构建 Spring AI ChatModel
    }
}
```

2. 放入 `llm/factory/providers/` 目录
3. 若 Spring AI 无现成集成 → 自研 ChatModel 实现，参考 `llm/providers/CozeChatModel.java`
4. 依赖加入根 `pom.xml` `<dependencyManagement>` + `xiaozhi-ai/pom.xml`
5. 无需改工厂——`ChatModelFactory` 构造注入 `List<ChatModelProvider>` 自动收集

### 7.2 新增 TTS / STT Provider

同模式：实现 `TtsService` 或 `SttService` 接口 → 放 `tts/providers/` 或 `stt/providers/` → **需要在 `TtsServiceFactory.createApiService()` 或 `SttServiceFactory.createApiService()` 的 switch 中加分支**（这两个工厂不是自动收集）。

### 7.3 新增业务领域（后端）

1. 在 `xiaozhi-service` 建领域包（对照 4.2 的 device 模板）
2. 简单领域可用 B 档结构（DAL + Service + Convert）
3. 在 `XiaozhiApplication` / `DialogueApplication` 的 `@ComponentScan` / `@MapperScan` 加对应包
4. 表结构 → 新增 Flyway 迁移文件 `V<下一个版本>__<描述>.sql`
5. 在 `xiaozhi-server` 对应领域包加 Controller + AppService

### 7.4 新增前端页面

1. `web/src/views/` 建视图组件
2. `web/src/router/routes.ts` 注册路由，`meta.permission` 填权限 key
3. `web/src/services/` 建 service 文件（路径常量进 `constants/api.ts`）
4. 权限 key 需在数据库 permission 表注册（或 Flyway 迁移）
5. 组件内用 `v-permission` 指令控制元素级显隐

### 7.5 新增设备控制消息类型

1. `communication/domain/` 新增 `XxxMessage extends Message`，并在 `Message` 的 `@JsonSubTypes` 注册 type 名
2. `MessageHandler.handleMessage()` switch 加分支
3. 下行消息统一走 `MessageSender`

### 7.6 调试技巧

- **打断相关**：先读 `DialogueService.abortDialogue()` 注释，理解 ②→③ 顺序
- **工具调用异常**：看 `XiaoZhiToolCallingManager.executeToolCalls()` 的 Observation 埋点
- **音频问题**：`AudioUtils` / `OpusProcessor`（纯 Java Opus）；录音留档在 `audio/{date}/{device-id}/`
- **跨实例不生效**：检查 `RedisBroadcast` 频道与 `RedisSubscriber` 是否匹配
- **新增 Bean 不生效**：九成是 `@ComponentScan` 没加包

---

## 8. 工程规范

### 8.1 代码风格

| 端 | 规则 |
|---|---|
| Java | 4 空格缩进，PascalCase 类名，camelCase 方法/变量，Lombok 减样板 |
| TS/Vue | 2 空格，**无分号**，**单引号**，composable 用 `use` 前缀，组件 PascalCase 文件名 |
| 提交 | Conventional Commits + 中文描述（`feat: 新增web聊天`），类型限 `feat/fix/refactor/update` |

### 8.2 测试

```bash
mvn test -pl xiaozhi-server -Dtest=DeviceControllerTest          # 单类
mvn test -pl xiaozhi-server -Dtest=XxxTest#testMethod            # 单方法
cd web && npm run test:run && npm run test:e2e                   # 前端单测 + E2E
```

后端有 **ArchUnit 架构测试**守护分层规则（如 Service 层禁依赖 `*Req`），改动分层前先跑测试。

### 8.3 提交前自查清单

- [ ] 新增包已同步 `@ComponentScan` / `@MapperScan`（两个启动类按需）
- [ ] 涉及表结构 → 新增 Flyway 迁移文件，未手改已执行过的 SQL
- [ ] 新增 REST 接口 → 权限 key 已注册，属主资源已加 `@CheckOwner`
- [ ] 新增依赖 → 根 pom `dependencyManagement` + 模块 pom
- [ ] PR 描述注明数据库迁移与配置变更

---

## 附：新人第一周路径建议

1. **跑起来**：按第 2 章完成本地启动，用 Web 聊天页发一条消息走通全链路
2. **读三个类**：`WebSocketHandler` → `DialogueService` → `Persona.chat()`，对照 5.1 流程图
3. **打断一次**：对话中说话触发 VAD 打断，断点跟一遍 `abortDialogue()`
4. **加一个 Provider**：找一个最简单的 TTS（如 Edge）仿写一个，走通 7.2 流程
5. **改一个领域**：给 device 加个字段，从 Flyway → DO → Domain → Service → Controller → 前端走一遍全栈
