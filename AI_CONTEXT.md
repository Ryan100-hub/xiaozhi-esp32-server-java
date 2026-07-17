# AI 编程工具上下文文档

> 本文档面向 AI 编程工具（Cursor/Copilot/Claude 等），提供项目全貌以跳过检索、减少 Token 消耗。
> 最后更新：2026-07-17

---

## 1. 项目概览

**项目名称**: xiaozhi-esp32-server-java（小智 ESP32 服务端 Java 版）
**技术栈**: Java 21 + Spring Boot 3.5.8 + Spring AI 1.1.4 + Vue 3 + TypeScript + Vite 7
**数据库**: MySQL 8.0 + Redis 7（Lettuce + Redisson）
**ORM**: MyBatis-Plus 3.5.16（`map-underscore-to-camel-case: false`，列名直接映射）
**认证**: Sa-Token 1.39.0（JWT + Redis）
**API 文档**: Knife4j 4.5.0 + SpringDoc 2.8.0
**构建**: Maven（后端）+ npm（前端）
**架构**: 双进程架构 — xiaozhi-server(8091) + xiaozhi-dialogue(8092) 共享 MySQL/Redis

---

## 2. Maven 模块结构

```
xiaozhi-parent (pom, v5.0.0)
├── xiaozhi-common       # 共享：枚举、事件、BO/Req/Resp、工具类、Redis通信端口
├── xiaozhi-service      # 业务层：DDD领域模型、Repository、MyBatis Mapper、Service
├── xiaozhi-ai           # AI集成：LLM/TTS/STT Provider、记忆管理、工具调用、MCP
├── xiaozhi-server       # 管理后台(:8091)：REST Controller、Flyway迁移、安全配置
└── xiaozhi-dialogue     # 对话服务(:8092)：WebSocket、VAD/STT/LLM/TTS管道、音频播放
```

**模块依赖链**: `common ← service ← ai ← server / dialogue`

**启动类**:
- `xiaozhi-server`: `com.xiaozhi.XiaozhiApplication`
- `xiaozhi-dialogue`: `com.xiaozhi.DialogueApplication`（jar 后缀 `-exec.jar`）

**@ComponentScan 注意**: 模块通过显式 `basePackages` 控制扫描，新增包需更新对应启动类注解。

---

## 3. 核心依赖版本

| 依赖 | 版本 | 用途 |
|------|------|------|
| Spring Boot | 3.5.8 | 基础框架 |
| Spring AI | 1.1.4 | AI 抽象层（ChatModel/TtsService） |
| Spring Data BOM | 2024.1.2 | 锁定到 3.4.x（规避 3.5.x Redis bug） |
| MyBatis-Plus | 3.5.16 | ORM |
| Sa-Token | 1.39.0 | 认证授权 |
| MapStruct | 1.6.3 | 对象映射 |
| Redisson | 3.25.0 | 分布式锁 |
| ONNX Runtime | 1.23.2 | VAD 模型推理 |
| sherpa-onnx | 1.12.21 | 本地 TTS |
| Vosk | 0.3.45 | 本地 STT |
| Concentus | 1.0.2 | Opus 编解码 |
| WebRTC Java | 0.14.0 | AEC 回声消除 |
| PageHelper | 2.1.0 | 分页 |
| Knife4j | 4.5.0 | API 文档 |

---

## 3.1 常用命令速查

```bash
# === 后端 ===
mvn clean install -DskipTests                    # 全量编译
mvn clean install -DskipTests -pl xiaozhi-server --also-make  # 单模块编译
mvn test                                          # 运行全部测试
mvn test -pl xiaozhi-server -Dtest=DeviceControllerTest  # 单个测试类
java -Djava.library.path=lib -jar xiaozhi-server/target/xiaozhi-server-*.jar   # 启动server
java -Djava.library.path=lib -jar xiaozhi-dialogue/target/xiaozhi-dialogue-*-exec.jar  # 启动dialogue
bin/all.sh start|stop|restart|status              # 脚本管理全部服务

# === 前端（web/目录） ===
npm run dev          # Vite开发服务器(8084端口，代理/api→localhost:8091)
npm run build        # 生产构建（含类型检查）
npm run test:run     # Vitest单元测试
npm run test:e2e     # Playwright E2E测试
npm run lint         # Oxlint + ESLint
npm run format       # Prettier格式化
npm run type-check   # TypeScript类型检查
```

---

## 4. 后端包结构与关键类

### 4.1 xiaozhi-common（公共层）

```
com.xiaozhi
├── common/
│   ├── annotation/        # @AuditLog, @CheckOwner, @CheckOwners
│   ├── config/            # RedisCacheConfig, RuntimePathConfig
│   ├── domain/            # DomainEvent接口, AbstractDomainEvent
│   ├── exception/         # OperationFailedException, ResourceNotFoundException, UnauthorizedException 等
│   ├── model/
│   │   ├── bo/            # 12个BO: UserBO, DeviceBO, RoleBO, ConfigBO, TemplateBO, MessageBO, MessageMetadataBO, SummaryBO, VerifyCodeBO, OperationLogBO, AgentBO, UserAuthBO
│   │   ├── dataobject/    # BaseDO（createTime/updateTime 自动填充）
│   │   ├── req/           # 34个请求DTO（BasePageReq为分页基类）
│   │   └── resp/          # 14个响应VO（PageResp<T>为通用分页）
│   ├── port/              # ConfigLookup（配置查询端口）, TokenResolver（第三方Token解析端口）
│   ├── web/               # ApiResponse<T>（统一响应）, ResultStatus（状态码常量）
│   ├── CacheHelper.java   # Redis缓存辅助
│   └── Speech.java        # 音频播放对象（PCM/Opus + text + emotion）
├── communication/
│   ├── common/RedisBroadcast.java         # Redis Pub/Sub 广播（6个频道）
│   ├── registry/DialogueServerRegistry.java # 对话服务注册表接口
│   └── ServerAddressProvider.java         # 服务地址组装器
├── enums/
│   ├── DeviceState.java   # IDLE → LISTENING → THINKING → SPEAKING
│   ├── ListenMode.java    # Auto / Manual / RealTime
│   └── ListenState.java   # Start / Stop / Text / Detect
├── event/                 # 14个Spring事件（见第10节）
└── utils/                 # AudioUtils, JsonUtil, OpusProcessor, CommonUtils 等
```

### 4.2 xiaozhi-service（业务层）

**DDD 分层模式**（每个领域包统一结构）:
```
<domain>/
├── dal/mysql/dataobject/<Domain>DO.java    # MyBatis-Plus 实体
├── dal/mysql/mapper/<Domain>Mapper.java    # extends BaseMapper<T>
├── domain/<Domain>.java                    # 聚合根（无public setter，行为方法变更状态）
├── domain/repository/<Domain>Repository.java # Repository接口
├── domain/vo/<ValueObject>.java            # 值对象（Java Record）
├── infrastructure/<Domain>RepositoryImpl.java # Repository实现（发布领域事件）
├── infrastructure/convert/<Domain>Converter.java # DO↔Domain手动转换(@Component)
├── service/<Domain>Service.java            # 服务接口
├── service/impl/<Domain>ServiceImpl.java   # 服务实现
└── convert/<Domain>Convert.java            # MapStruct转换器
```

**领域清单**:

| 领域 | 聚合根 | 值对象 | DDD完整 |
|------|--------|--------|---------|
| agent | - | - | 否（仅Service） |
| authrole | - | - | 否 |
| authrolepermission | - | - | 否 |
| config | AiConfig | - | 是 |
| device | Device | VerifyCode | 是 |
| mcptoolexclude | - | - | 否 |
| message | - | - | 否 |
| operationlog | - | - | 否（含AOP切面） |
| permission | - | - | 否 |
| role | Role | LlmConfig, VoiceConfig, AudioConfig, MemoryStrategy | 是 |
| security | - | - | OwnershipAspect + OwnershipChecker |
| storage | - | - | StorageServiceFactory（Local/Aliyun/Tencent） |
| summary | - | - | 否 |
| template | Template | - | 是 |
| token | - | - | TokenProvider（Aliyun/Coze） |
| user | - | - | 否（含WxLoginService） |
| userauth | - | - | 否 |

**领域信号机制**: 聚合根通过 `pullSignals()` 返回变更信号（如 `UPDATED`, `DEFAULT_CHANGED`），Repository.save() 根据信号发布对应 Spring ApplicationEvent。

### 4.3 xiaozhi-ai（AI 集成层）

#### LLM Provider 体系
```
ChatModelProvider（接口）
├── getProviderName() / createChatModel() / supports()
│
ChatModelFactory（工厂，Spring自动注入所有Provider）
├── 按provider名称路由，fallback到OpenAI兼容协议
│
providers/
├── OpenAiModelProvider     # OpenAI + 兼容协议（默认fallback）
├── OllamaModelProvider     # 本地Ollama
├── ZhiPuModelProvider      # 智谱GLM
├── DifyModelProvider       # Dify平台（自定义DifyChatModel）
├── CozeModelProvider       # Coze平台（自定义CozeChatModel）
├── XingHuoModelProvider    # 讯飞星火（自定义XingHuoChatModel）
└── XingChenModelProvider   # 星尘（自定义XingChenChatModel + SDK封装）
```

#### STT Provider 体系
```
SttService（接口）→ getProviderName() / stream(Flux<byte[]>) → SttResult
SttServiceFactory（工厂，默认Vosk，带实例缓存）
├── VoskSttService          # 本地离线（默认）
├── FunASRSttService        # 阿里FunASR
├── AliyunSttService        # 阿里云语音
├── AliyunNlsSttService     # 阿里云NLS
├── TencentSttService       # 腾讯云
├── XfyunSttService         # 讯飞
└── VolcengineSttService    # 火山引擎
```

#### TTS Provider 体系
```
TtsService（接口）→ textToSpeech() / getOptions() / getVoiceName()
TtsServiceAdapter → 适配为Spring AI TextToSpeechModel
TtsServiceFactory（工厂，默认Edge TTS，缓存key: provider:configId:voiceName:pitch:speed）
├── EdgeTtsService          # 微软Edge TTS（默认）
├── SherpaOnnxTtsService    # 本地离线
├── AliyunTtsService        # 阿里云
├── AliyunNlsTtsService     # 阿里云NLS
├── TencentTtsService       # 腾讯云
├── VolcengineTtsService    # 火山引擎
├── XfyunTtsService         # 讯飞
└── MiniMaxTtsService       # MiniMax
```

#### 记忆/对话管理
```
Conversation（实体，管理消息列表 + SystemMessage组装）
├── MessageWindowConversation   # 滑动窗口（固定条数截断）
└── SummaryConversation         # 摘要（LLM总结历史）

ConversationFactory（接口）
├── DefaultConversationFactory  # 创建MessageWindowConversation
└── SummaryConversationFactory  # 创建SummaryConversation

ChatMemory（接口）→ DatabaseChatMemory（数据库持久化）
UserMessageAssembler → 注入时间/情绪前缀
ToolCallMessageCodec → 工具调用消息编解码
```

#### 工具调用体系
```
XiaoZhiToolCallingManager → 自定义工具调用管理器（修复Spring AI流式chunk合并bug）
ToolRegistrar（接口）
├── DeviceMcpToolRegistrar  # 设备端MCP工具
├── SystemToolRegistrar     # 系统内置工具
└── ToolsGlobalRegistry     # 全局工具注册表（Redis发布）
ToolsSessionHolder → 会话级工具隔离（每个会话独立工具集）
```

#### MCP 集成
```
ai/mcp/
├── config/CustomMcpSyncClientCustomizer  # MCP客户端定制
├── registrar/DeviceMcpToolRegistrar      # 设备MCP工具注册
├── registrar/SystemToolRegistrar         # 系统工具注册
└── server/McpToolQueryServiceImpl        # MCP工具查询
```

### 4.4 xiaozhi-server（管理后台 :8091）

#### REST API 全览

| Controller | 基路径 | 端点 | 权限注解 |
|-----------|--------|------|---------|
| UserController | `/api/user` | `GET /check-token`, `POST /refresh-token`, `POST /login`, `POST /tel-login`, `POST /wx-login`, `POST`(注册), `GET`(分页), `PUT /{userId}`, `POST /resetPassword`, `POST /sendEmailCaptcha`, `POST /sendSmsCaptcha`, `GET /checkUser` | `@SaIgnore`(login/register/captcha) |
| DeviceController | `/api/device` | `GET`(分页), `POST /batchUpdate`, `POST`(创建), `PUT /{deviceId}`, `DELETE /{deviceId}`, `GET\|POST /ota`, `POST /ota/activate` | `@SaCheckPermission("system:device")` |
| RoleController | `/api/role` | `GET`(分页), `PUT /{roleId}`, `POST`(创建), `DELETE /{roleId}`, `GET /sherpaVoices`, `GET /testVoice` | `@SaCheckPermission("system:role")` |
| ConfigController | `/api/config` | `GET`(分页), `PUT /{configId}`, `POST`(创建), `DELETE /{configId}` | `@SaCheckPermission("system:config")` |
| TemplateController | `/api/template` | `GET`(分页), `POST`(创建), `PUT /{templateId}`, `DELETE /{templateId}` | `@SaCheckPermission("system:prompt-template")` |
| MessageController | `/api/message` | `GET`(分页), `GET /conversations`, `DELETE /{messageId}`, `DELETE`(批量) | `@SaCheckPermission` |
| AuthRoleController | `/api/auth-role` | `GET`(分页), `GET /{authRoleId}/permissions`, `PUT /{authRoleId}/permissions` | `@SaCheckPermission("system:auth-role")` |
| AgentController | `/api/agent` | `GET`(分页) | `@SaCheckPermission("system:config:agent")` |
| McpToolController | `/api/mcpTool` | `PATCH /role/{roleId}/tools`, `POST /role/{roleId}/exclude-tools`, `PATCH /global/tools`, `GET /role/{roleId}/disabled-tools`, `GET /system-global` | - |
| MemoryController | `/api/memory` | `GET /summary/{roleId}/{deviceId}`, `DELETE /summary/{roleId}/{deviceId}` | - |
| FileUploadController | `/api/file` | `POST /upload` | - |
| MusicController | `/api/file` | `POST /music` | - |
| WebChatController | `/api/chat` | `POST /open`, `GET /stream`(SSE), `POST /close` | `@SaCheckPermission("system:chat")` |

**统一响应**: `ApiResponse<T>` → `{ code: 200|500|..., message: "...", data: T }`
**分页响应**: `PageResp<T>` → `{ list, total, pageNo, pageSize }`

#### AppService 层
每个Controller对应一个AppService负责业务编排：`UserAppService`, `DeviceAppService`, `RoleAppService`, `ConfigAppService`, `TemplateAppService`, `MessageAppService`, `AuthRoleAppService`, `AgentAppService`, `WebChatService`

#### 服务器配置类
| 类 | 职责 |
|----|------|
| SaTokenConfig | JWT集成，拦截`/api/**`，OPTIONS放行 |
| WebMvcConfig | CORS全开放，日志/限流拦截器，静态资源映射，虚拟线程异步超时120s |
| SwaggerConfig | Knife4j/OpenAPI，Bearer Token安全方案 |
| ThreadConfig | 虚拟线程执行器 |
| GlobalExceptionHandler | 全局异常处理 |
| RateLimitInterceptor | 登录/注册/验证码限流 |
| SecurityHeaderFilter | 安全响应头 |
| PageFilter | 分页过滤器 |
| BaseController | 控制器基类 |

### 4.5 xiaozhi-dialogue（对话服务 :8092）

#### WebSocket 通信
```
WebSocketConfig → 端点路径: /ws/xiaozhi/v1/
WebSocketHandler（核心）
├── afterConnectionEstablished() → 提取device-id + Authorization，创建会话
├── handleTextMessage() → JSON消息分发（hello/listen/abort/goodbye/iot/mcp）
├── handleBinaryMessage() → Opus音频帧处理
└── handleHelloMessage() → 握手，回复音频参数，异步初始化MCP
```

#### 消息协议（密封类体系）
| 消息类 | type | 说明 |
|--------|------|------|
| HelloMessage | hello | 设备握手（AudioParams + Features） |
| HelloMessageResp | - | 响应（transport/sessionId/audioParams） |
| ListenMessage | listen | 聆听控制（mode: auto/manual/realtime） |
| AbortMessage | abort | 中止对话 |
| GoodbyeMessage | goodbye | 告别 |
| IotMessage | iot | IoT设备控制 |
| DeviceMcpMessage | mcp | 设备MCP协议 |
| UnknownMessage | unknown | 未知消息 |

**AudioParams**: Opus 16kHz/1ch/60ms

#### 会话管理
```
ChatSession → 持有 Device/Persona/Player/AudioStream/ToolsSessionHolder
SessionManager → 管理所有ChatSession，deviceId→sessionId反向索引
MessageHandler → 分发文本消息
MessageSender → 向设备发送消息（STT/TTS/Binary/Emotion）
InactiveSessionChecker → 超时断连
RedisSubscriber → 监听跨实例Redis广播
```

#### 对话管道（核心数据流）
```
ESP32设备 → WebSocket → Opus二进制帧
  → VadService（Silero VAD ONNX）
  │   状态机: NO_SPEECH → SPEECH_START → SPEECH_CONTINUE → SPEECH_END
  │   配置: speechThreshold=0.4, silenceThreshold=0.3, energyThreshold=0.001, silenceTimeoutMs=800
  │   支持: pre-buffer(500ms), tail-keep(300ms), AEC回声消除
  │
  → SttService.stream(audioFlux) → SttResult（text + emotion）
  │   [虚拟线程中执行]
  │
  → DialogueService.handleText()
  │   ├── 情感标签处理
  │   ├── IntentService.detect() → EXIT等意图
  │   └── Persona.chat(userMessage)
  │       ├── chatStream() → ChatModel.stream() → MessageAggregator聚合
  │       ├── convert() → ChatToken流（分离thinking/content）
  │       └── Synthesizer.synthesize()
  │           ├── SentenceHelper分句
  │           ├── TtsService.textToSpeech()
  │           └── Player.play(Speech流)
  │               └── ScheduledPlayer → 精确60ms间隔调度 → Opus编码 → WebSocket
  │
  → DialogueListener.onDialogueTurn() → 持久化到数据库

打断流程: abortDialogue() → Synthesizer.cancel() → Player.stop() → sendTtsStop()
```

#### 关键运行时类
| 类 | 职责 |
|----|------|
| Persona | 核心对话循环聚合体（ChatModel + Synthesizer + Player + Conversation + SttService + ToolCallbacks） |
| PersonaFactory | 构建Persona（幂等），组装各组件 |
| PersonaListener / DialogueListener | 对话事件回调，消息持久化 |
| DialogueService | 编排VAD→STT→文本处理→中止 |
| ScheduledPlayer | 虚拟线程调度播放器，burst模式预缓冲2帧 |
| FileSynthesizer | LLM token流→分句→逐句TTS→PCM→Player |
| SynthesizerFactory | 合成器工厂 |
| VadService | Silero VAD ONNX语音活动检测 |
| AecService | WebRTC AEC3回声消除 |

#### 系统内置工具（Function Calling）
| 工具类 | 功能 |
|--------|------|
| ChangeRoleFunction | 切换角色 |
| LocalMusicPlayer | 本地音乐播放 |
| NewChatFunction | 新对话 |
| PlayHuiBenFunction | 绘本播放 |
| PlayListGetter | 播放列表获取 |
| PlayMusicFunction | 音乐播放 |
| Quote0Function | 引用功能 |
| SessionExitFunction | 退出会话 |
| IotService | IoT设备控制 |
| DeviceMcpService | 设备MCP工具 |

---

## 5. 数据库表结构

### 4.6 关键枚举与常量值

#### ConfigBO 字段值
- **configType**: `llm`(大语言模型), `stt`(语音识别), `tts`(语音合成), `agent`(智能体), `oss`(对象存储)
- **modelType**（内部枚举 `ConfigBO.ModelType`）: `chat`, `vision`, `intent`, `embedding`
- **state**: `"1"`=启用, `"0"`=禁用
- **isDefault**: `"1"`=默认, `"0"`=非默认

#### PermissionDO.permissionType（MySQL ENUM）
- `menu` — 菜单
- `button` — 按钮
- `api` — 接口

#### BasePageReq 默认值
- `pageNo`: 默认 `1`，`@Min(1)`
- `pageSize`: 默认 `10`，`@Min(1) @Max(1000)`

#### ResultStatus 全部状态码

| 常量名 | 值 | 含义 |
|--------|----|------|
| SUCCESS | 200 | 操作成功 |
| CREATED | 201 | 对象创建成功 |
| ACCEPTED | 202 | 请求已接受 |
| NO_CONTENT | 204 | 执行但无返回数据 |
| MOVED_PERM | 301 | 资源已移除 |
| SEE_OTHER | 303 | 重定向 |
| NOT_MODIFIED | 304 | 未修改 |
| BAD_REQUEST | 400 | 参数错误 |
| UNAUTHORIZED | 401 | 未授权 |
| FORBIDDEN | 403 | 访问受限/授权过期 |
| NOT_FOUND | 404 | 资源未找到 |
| BAD_METHOD | 405 | 不允许的HTTP方法 |
| CONFLICT | 409 | 资源冲突/被锁 |
| UNSUPPORTED_TYPE | 415 | 不支持的媒体类型 |
| ERROR | 500 | 系统内部错误 |
| NOT_IMPLEMENTED | 501 | 接口未实现 |

> 注意：项目无自定义业务错误码，全部使用标准 HTTP 状态码。

### 4.7 Redis 缓存与频道

#### 缓存名称与 TTL

| 缓存名 | 基础TTL | 随机偏移上限 | 防雪崩策略 |
|--------|---------|-------------|-----------|
| `XiaoZhi:Device` | 1天 | +10%（最多+3600s） | 基础时长+随机偏移 |
| `XiaoZhi:Permission` | 7天 | +10%（最多+3600s） | 同上 |
| `XiaoZhi:User` | 1天 | +10%（最多+3600s） | 同上 |
| `XiaoZhi:SysConfig` | 7天 | +10%（最多+3600s） | 同上 |
| `XiaoZhi:McpToolExclude` | 7天 | +10%（最多+3600s） | 同上 |
| 默认（其他） | 1天 | +10%（最多+3600s） | 同上 |

- 序列化：Key=`StringRedisSerializer`，Value=`GenericJackson2JsonRedisSerializer`
- CacheHelper 提供 `getWithLock()` 带分布式锁缓存查询（锁key前缀 `"lock:"`，等待3s，自动释放10s）

#### Redis Pub/Sub 频道

| 常量名 | 频道名 | 消息内容 | 触发事件 |
|--------|--------|---------|---------|
| CHANNEL_CLEAR_CONVERSATION | `xiaozhi:clear-conversation` | deviceId | ConversationHistoryClearedEvent |
| CHANNEL_ROLE_CHANGED | `xiaozhi:role-changed` | deviceId | DeviceRoleChangedEvent |
| CHANNEL_CONFIG_CHANGED | `xiaozhi:config-changed` | JSON `{configType, configId}` | AiConfigChangedEvent |
| CHANNEL_CLOSE_SESSION | `xiaozhi:close-session` | deviceId | DeviceSessionClosedEvent |
| CHANNEL_ROLE_UPDATED | `xiaozhi:role-updated` | roleId | RoleUpdatedEvent |
| CHANNEL_DEVICE_UPDATED | `xiaozhi:device-updated` | deviceId | DeviceUpdatedEvent |

### 4.8 Persona 关键方法签名

```java
// 聚合字段（@Builder构建）
String sessionId;
PersonaListener listener;
SttService sttService;
ChatModel chatModel;
Synthesizer synthesizer;
Player player;
Conversation conversation;
List<ToolCallback> toolCallbacks;  // 默认空列表
GoodbyeMessageSupplier goodbyeMessages;

// 核心方法
void chat(String userMessage)                              // 默认启用工具调用
void chat(String userMessage, boolean useFunctionCall)     // 无metadata的便利方法
void chat(UserMessage userMessage, boolean useFunctionCall) // **主入口**：带元数据对话
boolean isActive()                                         // Synthesizer工作中 或 Player有内容
void sendGoodbyeMessage()                                  // 告别→播放→关闭会话→清理资源

// 内部方法
Flux<ChatResponse> chatStream(Instant, UserMessage, boolean) // 核心流式LLM调用
Flux<ChatToken> convert(Flux<ChatResponse>)                  // ChatResponse→ChatToken（thinking/content分离）
```

### 4.9 存储服务路由（StorageServiceFactory）

```
configService.getDefaultBO("oss") → 读取sys_config中configType="oss"的默认配置
├── null 或 provider="local"  → LocalStorageService
├── provider="tencent"        → TencentCosStorageService
├── provider="aliyun"         → AliyunOssStorageService
└── 其他                      → 降级 LocalStorageService
```
- 云端客户端通过DCL缓存复用，provider变更时自动shutdown旧客户端
- `@PreDestroy` 释放资源

---

### 5.1 表清单

| 表名 | 主键 | 关键字段 | 说明 |
|------|------|---------|------|
| sys_user | userId(Int,AUTO) | username, password, wxOpenId, wxUnionId, name, avatar, state, isAdmin, authRoleId, tel, email, loginIp, loginTime | 用户表 |
| sys_user_auth | id(Long,AUTO) | userId, openId, unionId, platform, profile | 第三方认证 |
| sys_device | deviceId(String,INPUT) | deviceName, roleId, mcpList, ip, location, wifiName, chipModelName, type, version, state, userId | 设备表 |
| sys_role | roleId(Int,AUTO) | userId, avatar, roleName, roleDesc, voiceName, ttsPitch, ttsSpeed, state, ttsId, modelId, sttId, temperature, topP, vadEnergyTh, vadSpeechTh, vadSilenceTh, vadSilenceMs, isDefault, memoryType | 角色表 |
| sys_config | configId(Int,AUTO) | userId, configName, configDesc, configType, modelType, provider, appId, apiKey, apiSecret, ak, sk, apiUrl, state, isDefault, enableThinking | AI配置表 |
| sys_template | templateId(Int,AUTO) | userId, templateName, templateDesc, templateContent, category, isDefault, state | 提示词模板 |
| sys_message | messageId(Long,AUTO) | userId, deviceId, sender, message, metadata(JSON), statDate, audioPath, state, messageType, toolCalls, sessionId, source, roleId | 消息表 |
| sys_summary | 无主键注解 | deviceId, roleId, lastMessageTimestamp, summary, promptTokens, completionTokens, createTime | 对话摘要 |
| sys_permission | permissionId(Int,AUTO) | parentId, name, permissionKey, permissionType, path, component, icon, sort, visible, status | 权限表（菜单/按钮/API三级） |
| sys_auth_role | authRoleId(Int,AUTO) | authRoleName, roleKey, description, status | 认证角色 |
| sys_auth_role_permission | id(Int,AUTO) | authRoleId, permissionId, createTime | 角色权限关联 |
| sys_mcp_tool_exclude | id(Long,AUTO) | excludeType, bindType, bindCode, bindKey, excludeTools | MCP工具排除 |
| sys_operation_log | id(Long,AUTO) | userId, ip, module, operation, method, url, handler, params, success, errorMsg, costMs, createTime | 操作日志 |

### 5.2 表间关系

```
sys_user (1)──(N) sys_device          (userId)
sys_user (1)──(N) sys_role            (userId)
sys_user (1)──(N) sys_config          (userId)
sys_user (1)──(N) sys_template        (userId)
sys_user (1)──(N) sys_message         (userId)
sys_user (1)──(N) sys_user_auth       (userId)
sys_user (N)──(1) sys_auth_role       (authRoleId)
sys_device (N)──(1) sys_role          (roleId)
sys_role   (N)──(1) sys_config [LLM]  (modelId)
sys_role   (N)──(1) sys_config [TTS]  (ttsId)
sys_role   (N)──(1) sys_config [STT]  (sttId)
sys_auth_role (1)──(N) sys_auth_role_permission (authRoleId)
sys_permission (1)──(N) sys_auth_role_permission (permissionId)
```

### 5.3 Flyway 迁移历史

| 版本 | 说明 |
|------|------|
| V1 | 初始化13张表 + 默认管理员 + 6个默认模板 + 权限数据 + 角色 |
| V2 | 默认本地OSS配置 |
| V3 | sys_message.sender新增`tool`枚举值 |
| V4 | 消息指标列抽离（tokens/sttDuration/ttsDuration等） |
| V5 | Web聊天权限 |
| V6 | sys_message.source字段（区分web/device来源） |
| V7 | sys_device.mcpList字段 |
| V8 | sys_message.metadata JSON列 |
| V9 | sys_config.enableThinking字段 |
| V10 | sys_role的ttsPitch/ttsSpeed改DOUBLE |

**迁移文件路径**: `xiaozhi-server/src/main/resources/db/migration/`
**命名规范**: `V{N}__{description}.sql`（注意双下划线）

---

## 6. 前端结构（web/）

### 6.1 技术栈
- Vue 3.5 + TypeScript 5.9 + Vite 7
- Ant Design Vue 4.2 + Pinia 3 + Vue Router 4
- Axios + vue-i18n + dayjs + jsencrypt + wavesurfer.js

### 6.2 目录结构
```
web/src/
├── components/        # 13个可复用组件（AudioPlayer, FloatingChat, RobotAvatar, SettingsDrawer等）
├── composables/       # 28个组合式函数（useAuth, useWebSocket, useChatSession, useTable等）
├── config/            # llm_factories.json, providerConfig.ts
├── constants/         # api.ts, enums.ts, index.ts, storage.ts
├── directives/        # permission.ts（权限指令）
├── layouts/           # MainLayout, AppHeader, AppSidebar, AppFooter
├── locales/           # zh-CN.ts, en-US.ts
├── router/            # index.ts, routes.ts, guards.ts
├── services/          # 15个API服务文件
├── store/             # 4个Pinia Store
├── types/             # 14个TypeScript类型定义
├── utils/             # date.ts, format.ts, i18n.ts, jsencrypt.ts, logger.ts等
├── views/             # 22个页面视图
├── App.vue
└── main.ts
```

### 6.3 路由表

| 路径 | 组件 | 权限 | 说明 |
|------|------|------|------|
| /login | LoginView | - | 登录 |
| /register | RegisterView | - | 注册 |
| /forget | ForgetView | - | 找回密码 |
| /dashboard | DashboardView | system:dashboard | 仪表盘 |
| /user | UserView | system:user | 用户管理 |
| /device | DeviceView | system:device | 设备管理 |
| /role | RoleView | system:role | 角色配置 |
| /template | TemplateView | system:prompt-template | 提示词模板 |
| /memory/chat | MemoryManagementView | system:role | 短期记忆 |
| /memory/summary | MemoryManagementView | system:role | 摘要记忆 |
| /config/model | ModelConfigView | system:config | 模型配置 |
| /config/agent | AgentView | system:config:agent | 智能体 |
| /config/stt | SttConfigView | system:config | STT配置 |
| /config/tts | TtsConfigView | system:config | TTS配置 |
| /config/oss | OssConfigView | system:config | OSS存储 |
| /chat | ChatView | system:chat | Web聊天 |
| /auth-role | AuthRoleView | system:auth-role | 权限角色 |
| /setting/account | AccountView | system:setting | 个人中心 |

### 6.4 Pinia Store

| Store | 状态 | 说明 |
|-------|------|------|
| useUserStore | userInfo, permissions, authRole, token, refreshToken | 用户+权限+Token（localStorage持久化） |
| useAppStore | sidebarCollapsed, isMobile, navigationStyle, pageTitle | 布局状态 |
| useDeviceStore | devices, currentDevice | 设备列表+在线状态 |
| useLoadingStore | loading计数器 | 全局loading（防闪烁） |

### 6.5 API 服务层

| 文件 | 方法 |
|------|------|
| user.ts | login, telLogin, register, resetPassword, sendEmailCaptcha, sendSmsCaptcha, queryUsers, updateUser, addUser |
| device.ts | queryDevices, addDevice, updateDevice, deleteDevice, clearDeviceMemory |
| role.ts | queryRoles, addRole, updateRole, deleteRole, testVoice, querySherpaVoices, getSystemGlobalTools |
| template.ts | queryTemplates, addTemplate, updateTemplate, deleteTemplate, setDefaultTemplate |
| config.ts | queryConfigs, addConfig, updateConfig, deleteConfig, queryPlatformConfig |
| agent.ts | queryAgents |
| message.ts | queryMessages, deleteMessage, batchDeleteMessages, exportMessages, queryConversations |
| memory.ts | querySummaryMemory, queryChatMemory, deleteSummaryMemory |
| authRole.ts | queryAuthRoles, getAuthRolePermissionConfig, updateAuthRolePermissions |
| chat.ts | openChatSession, closeChatSession, chatStream(SSE) |
| upload.ts | uploadFile(XHR, 进度回调) |
| audio.ts | initAudio, loadOpusLibrary, initOpusDecoder, handleBinaryAudioMessage(WebAssembly) |
| websocket.ts | connectToServer, disconnectFromServer, sendTextMessage, startDirectRecording, registerMessageHandler |

### 6.6 Vite 开发代理配置

```ts
// vite.config.ts
server: {
  port: 8084,
  host: '0.0.0.0',
  proxy: {
    '/api': {
      target: process.env.API_URL || env.VITE_BACKEND_URL || 'http://localhost:8091',
      changeOrigin: true,
      secure: true,
      // path 不做重写，原样转发到后端
    }
  }
}
```

### 6.7 环境配置

| 文件 | 变量 |
|------|------|
| .env.development | VITE_API_BASE_URL=/api, VITE_WS_URL=ws://127.0.0.1:8091/ws/xiaozhi/v1 |
| .env.production | VITE_API_BASE_URL=/api, VITE_WS_URL=ws://connectai.chat/ws/xiaozhi/v1 |

### 6.8 前端 TypeScript 类型定义（types/）

| 文件 | 导出的关键类型 |
|------|---------------|
| api.ts | `ApiResponse<T>`, `PageData<T>`, `PageResponse<T>`, `ListResponse<T>`, `EmptyResponse`, `DataResponse<T>`, `BaseQueryParams`, `PageQueryParams` |
| user.ts | `User`, `UserQueryParams`, `UpdateUserParams` |
| role.ts | `ModelType('llm'\|'agent')`, `VoiceProvider`(8种), `MemoryType('window'\|'summary')`, `Role`, `RoleQueryParams`, `VoiceOption`, `ModelOption`, `SttOption`, `PromptTemplate`, `RoleFormData`, `TestVoiceParams` |
| device.ts | `Device`, `DeviceQueryParams`, `Role`(简化版) |
| config.ts | `ConfigType('llm'\|'stt'\|'tts'\|'agent'\|'oss')`, `ModelType('chat'\|'vision'\|'intent'\|'embedding')`, `Config`, `ConfigQueryParams`, `ConfigField`, `ConfigTypeInfo`, `LLMModel`, `LLMFactory` |
| message.ts | `MessageSender('user'\|'assistant'\|'system')`, `Message`, `Conversation`, `MessageQueryParams`, `ConversationQueryParams` |
| chat.ts | `ChatToken`(type: `'thinking'\|'content'`), `ChatMessage` |
| menu.ts | `MenuMeta`, `MenuItem` |
| memory.ts | `SummaryMemory`, `ChatMemory`, `MemoryQueryParams`, `MemoryManagementState` |
| mcpTool.ts | `SystemGlobalToolSummary`, `McpToolItem`, `McpToolSchemaProperty` |
| authRole.ts | `AuthRole`, `AuthRoleQueryParams`, `PermissionTreeNode`(permissionType: `'menu'\|'button'\|'api'`), `AuthRolePermissionConfig` |
| template.ts | `PromptTemplate`, `TemplateQuery`, `TemplateFormData`, `CategoryOption` |
| agent.ts | `AgentQueryParams`, `Agent`, `PlatformConfig`, `ProviderOption`, `FormItem`, `PlatformFormItems` |
| jsencrypt.d.ts | `JSEncrypt` 模块声明 |

### 6.9 前端常量定义

#### enums.ts 关键枚举

| 枚举名 | 值 |
|--------|-----|
| UserState | DISABLED=0, NORMAL=1 |
| UserType | NORMAL=0, ADMIN=1 |
| DeviceState | OFFLINE=0, ONLINE=1 |
| MessageType | TEXT='text', AUDIO='audio', IMAGE='image', SYSTEM='system' |
| SenderType | USER='user', ASSISTANT='assistant', SYSTEM='system' |
| WebSocketState | CONNECTING=0, OPEN=1, CLOSING=2, CLOSED=3 |
| ThemeMode | LIGHT='light', DARK='dark', AUTO='auto' |
| Locale | ZH_CN='zh-CN', EN_US='en-US' |
| NavigationStyle | SIDEBAR='sidebar', TABS='tabs' |
| ConfigType | LLM='llm', STT='stt', TTS='tts', AGENT='agent' |

#### api.ts 关键常量

| 常量 | 值 |
|------|-----|
| REQUEST_TIMEOUT | 30000 (30s) |
| DEFAULT_PAGE_SIZE | 10 |
| PAGE_SIZE_OPTIONS | [10, 30, 50, 100, 1000] |
| MAX_FILE_SIZE | 10MB |
| ALLOWED_IMAGE_TYPES | jpeg, png, gif, webp |
| ALLOWED_AUDIO_TYPES | mp3, wav, mpeg |
| DEBOUNCE_DELAY | 500ms |
| WS_RECONNECT_DELAY | 3000ms |
| WS_MAX_RECONNECT_TIMES | 5 |
| PASSWORD_MIN_LENGTH / MAX_LENGTH | 8 / 32 |
| USERNAME_MIN_LENGTH / MAX_LENGTH | 4 / 20 |
| DEVICE_NAME_MAX_LENGTH | 50 |
| ROLE_NAME_MAX_LENGTH | 50 |
| DESCRIPTION_MAX_LENGTH | 500 |

#### 校验正则

| 名称 | 正则 |
|------|------|
| EMAIL_PATTERN | `/^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/` |
| PHONE_PATTERN | `/^1[3-9]\d{9}$/` |
| PASSWORD_PATTERN | `/^(?=.*[A-Za-z])(?=.*\d)[A-Za-z@$!%*#?&]{8,}$/` |
| USERNAME_PATTERN | `/^[a-zA-Z0-9_]{4,20}$/` |

### 6.10 LLM Provider 配置表（providerConfig.ts）

`configTypeMap` 定义了四大类配置的服务商信息：

**LLM（38+ provider，通过 typeFields 按名称定义表单字段）**:
OpenAI, Tongyi-Qianwen, XunFei Spark, ZHIPU-AI, DeepSeek, VolcEngine, MiniMax, Tencent Hunyuan, BaiChuan, Moonshot, SILICONFLOW, BaiduYiyan, Ollama, LM-Studio, Azure-OpenAI, xAI, Mistral, Gemini, Groq, OpenRouter, StepFun, NVIDIA, 01.AI, Anthropic, Voyage AI, GiteeAI, DeepInfra, LocalAI, VLLM, Xinference, HuggingFace, Cohere, TogetherAI, Replicate, 302.AI, Fish Audio, PPIO, NovitaAI, GPUStack, Upstage, LeptonAI, PerfXCloud, Google Cloud, Bedrock, CometAPI, DeerAPI

**STT（6 provider，typeOptions 下拉）**: tencent, aliyun(DashScope), aliyun-nls(NLS标准版), xfyun, funasr, volcengine(doubao)

**TTS（7 provider，typeOptions 下拉）**: tencent, aliyun, aliyun-nls, volcengine(doubao), xfyun, minimax, sherpa-onnx(本地)

**OSS（3 provider，typeOptions 下拉）**: local(本地存储), tencent(腾讯云COS), aliyun(阿里云OSS)

---

## 7. 事件体系

### 7.1 领域事件（14个）

| 事件 | 载荷 | 触发场景 |
|------|------|---------|
| DeviceUpdatedEvent | DeviceBO | 设备信息变更 |
| DeviceOnlineEvent | deviceId | 设备上线 |
| DeviceRoleChangedEvent | deviceId | 角色变更 |
| DeviceSessionClosedEvent | deviceId | 设备删除/重新激活 |
| RoleUpdatedEvent | roleId | 角色配置变更 |
| AiConfigChangedEvent | configType, configId | AI配置变更 |
| ChatSessionOpenedEvent | sessionId, deviceId | WebSocket连接 |
| ChatSessionClosedEvent | sessionId, deviceId | Session关闭 |
| ChatAudioOpenedEvent | sessionId, deviceId | 音频握手完成 |
| ChatAbortedEvent | sessionId, deviceId, reason | 设备打断 |
| SpeechRecognizedEvent | sessionId, text, emotion | STT完成 |
| TtsPlaybackCompletedEvent | sessionId | TTS播放结束 |
| ToolCallCompletedEvent | sessionId, toolName, arguments, result | 工具调用完成 |
| ConversationHistoryClearedEvent | deviceId | 对话历史清除 |

### 7.2 事件监听器

| 监听类 | 事件 | 处理 |
|--------|------|------|
| RedisBroadcast | 6个变更事件 | Redis Pub/Sub跨实例广播 |
| SessionManager | DeviceUpdated | 同步设备信息到会话 |
| VadService | TtsPlaybackCompleted | 重置VAD模型状态 |
| AecService | TtsPlaybackCompleted | 重置AEC状态 |
| PersonaCleanup | ChatSessionClosed | 清理Persona资源 |
| ToolLogger | ToolCallCompleted | 记录工具调用日志 |
| ToolsGlobalRegistry | ApplicationReadyEvent | 发布全局工具到Redis |

---

## 8. 安全体系

- **认证**: Sa-Token JWT模式，拦截`/api/**`，`@SaIgnore`标注公开端点
- **权限**: `@SaCheckPermission` + `StpInterfaceImpl`提供权限列表
- **资源归属**: `@CheckOwner`注解 + `OwnershipAspect` AOP + 6个`OwnershipChecker`实现
- **审计**: `@AuditLog`注解 + AOP异步写入sys_operation_log
- **限流**: `RateLimitInterceptor`对登录/注册/验证码限流
- **密码**: `AuthenticationService`接口（encryptPassword/isPasswordValid）

---

## 9. 配置体系

### 9.1 应用配置

| 文件 | 端口 | 说明 |
|------|------|------|
| xiaozhi-server/application.yml | 8091 | 主配置（MyBatis/Redis/Flyway/Sa-Token/MCP/虚拟线程） |
| xiaozhi-server/application-dev.yml | - | MySQL localhost:3306/xiaozhi |
| xiaozhi-server/application-prod.yml | - | 域名connectai.chat |
| xiaozhi-dialogue/application.yml | 8092 | Flyway禁用，共享同一DB |
| xiaozhi-dialogue/application-dev.yml | - | MySQL同server |

### 9.2 关键配置项

```yaml
# MyBatis（重要！）
mybatis.configuration.map-underscore-to-camel-case: false  # 列名直接映射，不自动驼峰

# 虚拟线程
spring.threads.virtual.enabled: true

# Sa-Token
sa-token.token-name: Authorization
sa-token.token-prefix: Bearer
sa-token.timeout: 2592000  # 30天

# 运行时路径
xiaozhi.runtime.lib-dir: lib
xiaozhi.runtime.vosk-model-dir: models/vosk-model
xiaozhi.runtime.tts-model-dir: models/tts
xiaozhi.runtime.vad-model: models/silero_vad.onnx
```

---

## 10. 编码规范

### 10.1 Java 后端
- 4空格缩进，标准Java命名
- Lombok消除样板代码
- MapStruct用于DO↔BO/VO转换
- 领域模型无public setter，通过行为方法变更状态
- Repository实现负责发布领域事件
- Service层不依赖Req DTO（Controller负责解包）
- 虚拟线程用于STT处理和MCP初始化

### 10.2 TypeScript 前端
- 2空格缩进，LF换行
- 无分号，单引号（Prettier配置）
- 组合式函数前缀`use`（如useAuth.ts）
- 组件PascalCase文件名
- Service与后端API模块1:1对应

### 10.3 Git 提交规范
Conventional Commits + 中文描述：
```
feat: 新增web聊天
fix: 修复打断后没有正确切换isPlaying状态
refactor: 重构toolcalls
update: 更新log采用注解
```

---

## 11. 部署与基础设施

### 11.1 Docker

| 文件 | 服务 |
|------|------|
| docker-compose.yml | 全栈：mysql + node(前端) + server(8091) + dialogue(8092) + redis |
| docker-compose-infra.yml | 仅基础设施：mysql + redis |
| docker-compose-db.yml | 仅数据库：mysql + redis（开发用） |
| Dockerfile-server | 多阶段构建，双target(server/dialogue) |
| Dockerfile-node | 前端构建 + Nginx(8084) + SPA路由 + /api/反代 |
| Dockerfile-mysql | MySQL 8.0 + 初始化数据库 |

### 11.2 启动脚本（bin/）

| 脚本 | 说明 |
|------|------|
| all.sh | 编译+启动server(8091)+dialogue(8092) |
| server.sh | 仅server |
| dialogue.sh | 仅dialogue |
| _common.sh | 公共函数库（build/find_jar/start_service/stop_service） |

所有脚本支持 `start|stop|restart|status`。

### 11.3 原生依赖
- `lib/` — JNI原生库（Vosk, sherpa-onnx, WebRTC AEC）
- `models/` — STT/TTS模型文件（不在git中，需下载）
- 下载脚本: `scripts/download_models.sh`, `scripts/download_stt.sh`, `scripts/download_tts.sh`

---

## 12. 关键设计决策

1. **Persona 拥有对话循环**: LLM/TTS/Player/Conversation聚合在Persona中，Session持有Persona引用
2. **DeviceState 状态机**: `IDLE→LISTENING→THINKING→SPEAKING`，替代分散布尔标志
3. **打断顺序**: `abortDialogue()` → 先取消Synthesizer Flux上游 → 再停止Player下游（防止音频重叠）
4. **工具会话隔离**: 每个对话会话独立工具集（ToolsSessionHolder），不跨会话共享
5. **六边形端口**: `ConfigLookup`和`TokenResolver`避免AI模块直接依赖Service层
6. **密封类消息协议**: 编译安全的消息类型匹配
7. **Redis Pub/Sub 跨实例通信**: 6个频道用于配置变更/角色变更/会话清除等通知
8. **唯一默认不变式**: Repository.save()检测到DEFAULT_CHANGED信号时先resetDefault再保存
9. **ScheduledPlayer 精确调度**: 纳秒级时间控制，burst模式预缓冲2帧避免首帧破音
10. **Spring AI 流式chunk修复**: XiaoZhiToolCallingManager修复Spring AI流式工具调用chunk合并问题
