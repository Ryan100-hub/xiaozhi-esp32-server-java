# (1) 解释Dialogue水平扩展概念

> **Workspace:** d:\devFile\ai\code\github\xiaozhi-esp32-server-java

---

**User**

解释一下-  ‘dialogue 水平扩展’

**User**

`d:\devFile\ai\code\github\xiaozhi-esp32-server-java/docs/CODE_WIKI.md` 解释一下 什么是  dialogue 水平扩展

**User**

`d:\devFile\ai\code\github\xiaozhi-esp32-server-java/docs/CODE_WIKI.md` 解释一下 什么是  dialogue 水平扩展

**Assistant**

这是"双进程架构"里 dialogue 服务能横向扩容的核心机制。先看整体数据流。

## 一、什么是"水平扩展"

| 维度 | 垂直扩展（Scale Up） | 水平扩展（Scale Out） |
|---|---|---|
| 做法 | 单机加 CPU/内存/带宽 | 增加**实例数量**，多台机器跑同一份程序 |
| 上限 | 受单机硬件上限 | 近似线性，可弹性伸缩 |
| 代价 | 昂贵、有天花板 | 需要解决**服务发现 + 请求分配 + 状态共享** |

dialogue 服务承载的是**设备 WebSocket 长连接 + 实时音频推理**（VAD/STT/TTS），属于典型的"长连接 + 计算密集"负载：单实例的连接数和 CPU 都有硬上限。所以它必须能水平扩展。

但水平扩展的前提是：**新实例加进来，别人怎么知道它存在？设备又怎么找到它？** 这就是"dialogue 水平扩展"这句话要解决的问题。

## 二、本项目的三步实现

### 1. 自注册（实例上线）

[dialogueServerRegistrar.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-dialogue/src/main/java/com/xiaozhi/communication/DialogueServerRegistrar.java) 是每个 dialogue 实例内的一个 Spring 组件：

- `@PostConstruct register()` — 启动即向 Redis 注册自己
- `scheduleAtFixedRate(..., 30, 30, SECONDS)` — 每 30 秒续一次心跳
- `@PreDestroy unregister()` — 优雅关闭时主动注销

注册信息由 [ServerAddressProvider.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-common/src/main/java/com/xiaozhi/communication/ServerAddressProvider.java) 组装（websocket / udp / ota / mcp 四类地址），所以每个实例上报的是**自己可达的地址**，而不是写死的配置。

### 2. 注册中心与健康检查

[RedisDialogueServerRegistry.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-service/src/main/java/com/xiaozhi/communication/registry/RedisDialogueServerRegistry.java) 用两个 Redis 结构实现"服务发现"：

```
xiaozhi:dialogue:servers        (Hash)  instanceId → JSON(地址信息)
xiaozhi:dialogue:heartbeat:<id> (String) TTL = 60s
```

- **心跳 TTL 60s > 上报间隔 30s**：容忍一次心跳丢失，不会误判实例下线。
- `getAvailableServers()` 用 `multiGet` 批量校验心跳，**心跳过期的条目会被顺手清理**（僵尸实例自动摘除），因此不需要额外的清理定时任务。

### 3. 设备分配（OTA 下发）

[xiaozhi-server 的 OTA 接口](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-server/src/main/java/com/xiaozhi/device/DeviceAppService.java#L240-L253) 是分配发生的地方：

```java
selectedServer = dialogueServerRegistry.selectServer();   // 随机负载均衡
String websocketAddress = selectedServer != null
        ? selectedServer.getWebsocketAddress()
        : serverAddressProvider.getWebsocketAddress();     // 降级回退
```

`selectServer()` 内部用 `ThreadLocalRandom` 随机挑一个可用实例。设备每次开机走 OTA，就拿到一个 `websocket.url`，然后连到对应实例。

## 三、几个容易误解的点

1. **分配是"每次 OTA"，不是"每句话"**。设备一旦连上某实例，整段会话都粘在该实例上（会话状态在实例内存里：`ChatSession` / `Persona`）。重新分配只发生在设备重启重新走 OTA 时。
2. **负载均衡策略是随机，不是最少连接**。对长连接场景，随机策略实现简单且足够均匀；但已连接的设备不会因为实例负载升高而迁移。
3. **降级路径很重要**：Redis 不可用时 `selectServer()` 返回 null，OTA 回退到 `ServerAddressProvider` 的默认地址——服务不至于完全不可用。
4. **实例故障 = 该实例上的设备断开重连**。设备重连会重新走 OTA，从而被分配到存活实例。这正是水平扩展能容忍实例宕机的原理。
5. **有状态的部分必须外置**。所以 [checkStorageConfig()](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-dialogue/src/main/java/com/xiaozhi/communication/DialogueServerRegistrar.java#L72-L81) 会在启动时警告：如果 `StorageService` 是 `local` 模式，音频文件只在本机磁盘，集群下跨实例不可用，必须换 COS/OSS。
6. **跨实例通知走 Redis Pub/Sub**。管理端改配置后，通过 `RedisBroadcast` 广播给所有 dialogue 实例热更新，实例之间不直接通信。

## 四、一句话总结

> **dialogue 水平扩展 = 实例自注册到 Redis（`DialogueServerRegistrar`）+ 心跳 TTL 做健康检查（`RedisDialogueServerRegistry`）+ server 在 OTA 时把设备随机分配到某个实例。**

新增一个 dialogue 实例的完整操作就是：部署、启动、配好地址 —— 不需要改任何 server 端配置，也不需要重启其他

---

**User**

如果 Redis 宕机，服务如何降级？

**Assistant**

Redis 在本项目里承担了 6 种职责，降级行为**不一致**——有的 fail-open（放行/回退），有的 fail-closed（直接不可用）。逐个说。

## 一、有降级（fail-open）

| # | 用途 | 降级行为 | 代码位置 |
|---|---|---|---|
| 1 | **OTA 服务发现** | 回退到配置里的默认地址 | [DeviceAppService.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-server/src/main/java/com/xiaozhi/device/DeviceAppService.java#L241-L248) |
| 2 | **实例心跳注册** | 静默失败，30s 后自动重试 | [DialogueServerRegistrar.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-dialogue/src/main/java/com/xiaozhi/communication/DialogueServerRegistrar.java#L38-L58) |
| 3 | **登录/验证码限流** | 异常时**放行**（return true） | [RateLimitInterceptor.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-server/src/main/java/com/xiaozhi/server/config/RateLimitInterceptor.java#L69-L72) |
| 4 | **分布式锁（防缓存击穿）** | 拿不到锁就直查 DB | [CacheHelper.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-common/src/main/java/com/xiaozhi/common/CacheHelper.java#L66-L81) |
| 5 | **跨实例 Pub/Sub 广播** | 静默失败，仅 error 日志 | [RedisBroadcast.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-common/src/main/java/com/xiaozhi/communication/common/RedisBroadcast.java#L101-L108) |

### 1. OTA 分配 —— 最关键的一条

`getAvailableServers()` 内部整段 `try/catch`，异常时**返回空 List**，于是：

```
getAvailableServers() → []  →  selectServer() 返回 null
                            ↓
DeviceAppService: selectedServer == null
                            ↓
回退 serverAddressProvider.getWebsocketAddress()
```

后果：所有设备被分配到**同一个默认地址**，水平扩展能力丧失、负载集中，但服务可用。

> 注意一个反直觉的点：如果生产环境配了 `xiaozhi.server.domain`，`websocketAddress` 是 `wss://ws.<domain>/ws/xiaozhi/v1/`（走网关），此时 Redis 挂了**反而几乎无感**——因为本来就没按实例分配。

### 2. 心跳 —— 自愈

`register()` 首次失败只 `log.warn`，不阻止 dialogue 启动；心跳任务每 30s 重试。**Redis 恢复后实例自动重新注册，无需重启**。这是设计得最好的一处。

### 3. 限流 —— 安全上的缺口

```java
} catch (Exception e) {
    // Redis 异常时放行，不影响正常使用
    log.error("限流检查异常，已放行: {}", e.getMessage());
}
return true;
```

可用性优先，但 Redis 挂掉期间**登录接口失去暴力破解防护**。属于"用安全换可用"的显式取舍。

### 4. 分布式锁 —— 压力转移到 MySQL

`tryLock` 超时、被中断、或抛 `RedisConnectionException` 都被 `catch (Exception)` 兜住，走 `dbGetter.get()`。后果是**缓存击穿保护失效**，叠加下一条会更严重。

### 5. 广播 —— 需要人工介入

`publish()` 只记 error 日志。后果：管理端改了角色/配置/清对话历史，**其他 dialogue 实例收不到通知**，配置热更新失效，必须重启实例才生效。

## 二、无降级（fail-closed）

### 6. 设备 WebSocket 建连 —— 最严重的隐患

这条链路**没有任何 try/catch**：

```
WebSocketHandler.afterConnectionEstablished()   ← 无 try/catch
  └─ MessageHandler.afterConnection()
       ├─ deviceRegistry.getInstance(deviceId)   ← 直接抛异常
       └─ sessionManager.registerDevice()
            └─ deviceRegistry.bind(deviceId)     ← 直接抛异常
```

[DeviceRegistry.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-dialogue/src/main/java/com/xiaozhi/communication/common/DeviceRegistry.java#L38-L62) 的 `bind/getInstance/refresh/unbind` 全部裸调 Redis，而调用方 [WebSocketHandler.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-dialogue/src/main/java/com/xiaozhi/communication/server/websocket/WebSocketHandler.java#L52) 和 [MessageHandler.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-dialogue/src/main/java/com/xiaozhi/communication/common/MessageHandler.java#L114-L124) 都没有兜底。

**结论：Redis 宕机 → 新设备连不上 dialogue，已建立的会话不受影响。**

对比一下第 3 条（限流用 `catch` 放行）和第 6 条（建连裸奔），这是同一份代码库里降级策略不一致的地方，也是最该补的短板。

### 7. Sa-Token 鉴权 —— 管理后台不可用

依赖 `sa-token-redis-jackson`（[pom.xml](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/pom.xml#L141-L145)），session 存 Redis。`sa-token-jwt` 虽让 token 本身是 JWT，但登录态校验仍需 Redis。dialogue 的配置注释也明说了：

```yaml
# Sa-Token（仅用于 @SaIgnore 和无状态接口校验，token 验证依赖 Redis）
```

Redis 挂 → Web 前端登录、所有鉴权接口 401/500。

### 8. 缓存层整体失效

dialogue 配了 `spring.cache.type: redis`，Redis 挂掉等于**缓存层全空**，所有 `@Cacheable` 请求穿透到 MySQL。叠加第 4 条（分布式锁也失效），MySQL 会承受全部压力。

## 三、整体降级能力评估

```
                  Redis 宕机
                       │
        ┌──────────────┴──────────────┐
        │                             │
   对话主链路可用                  管理面不可用
   （已连接设备正常说话）          （后台登录/配置全挂）
        │
        ├─ 新设备无法连接   ← 隐患（第 6 条）
        ├─ 全部流量挤到默认地址 ← 可接受（第 1 条）
        └─ 配置热更新失效   ← 需人工重启（第 5 条）
```

**一句话总结：**
> Redis 宕机时，**已建立的对话不受影响**（会话状态在实例内存），但**新设备连不上**、**管理后台不可用**、**配置热更新失效**，同时**登录限流和缓存保护双双失守**。系统整体是"半降级"状态，而非优雅降级。

## 四、如果要补齐，优先级建议

1. **高**：给 `DeviceRegistry` 的 `bind/getInstance/refresh` 加 try/catch 并降级为"本地内存映射 + 单机模式"，让新设备在 Redis 故障时仍能连接（单体部署下这些映射本就无意义）。
2. **中**：`RedisBroadcast.publish` 失败时落一条 DB/本地待办，Redis 恢复后补发，避免配置热更新静默丢失。
3. **低**：限流加本地 `Caffeine` 兜底计数器，Redis 不可用时用本地计数代替"直接放行"。

需要我实现其中某一项吗

---

**User**

解释 STT和 TTS  

**Assistant**

STT 和 TTS 是语音对话闭环的两端。先看它们在整个管道里的位置。

## 一、概念：一个"耳朵"，一个"嘴巴"

| | STT | TTS |
|---|---|---|
| 全称 | **S**peech-**T**o-**T**ext | **T**ext-**T**o-**S**peech |
| 中文 | 语音识别 / ASR | 语音合成 |
| 方向 | 音频 → 文本 | 文本 → 音频 |
| 类比 | 系统的耳朵 | 系统的嘴巴 |
| 本项目接口 | [SttService.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-ai/src/main/java/com/xiaozhi/ai/stt/SttService.java) | [TtsService.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-ai/src/main/java/com/xiaozhi/ai/tts/TtsService.java) |

它们夹在 LLM 两侧：**STT 把"人说的话"变成 LLM 能读的文本，TTS 把 LLM 写的文本变回人能听的语音**。LLM 本身只处理文本，不接触音频——这就是为什么必须有这一对。

## 二、STT 在本项目里怎么设计的

接口只有一个方法：

```java
SttResult stream(Flux<byte[]> audioSink);
```

两个设计点值得注意：

**1. 入参是 `Flux<byte[]>` 而不是 `byte[]`**

这是**流式识别**的信号——音频边采集边送进来，而不是"等用户说完、凑齐一整段再识别"。因为 `Flux` 是响应式流，上游 VAD 检测到语音就持续推数据，STT 一边收一边出中间结果。这是低延迟的关键：用户说完最后一个字，文本几乎同时就绪，不用再多等一轮网络往返。

**2. 返回值是 `SttResult` record，不只是 String**

看 [SttResult.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-ai/src/main/java/com/xiaozhi/ai/stt/SttResult.java#L16-L22)：

```java
public record SttResult(
    String text,
    String emotion,            // happy / neutral / angry / sad
    Double emotionScore,
    String emotionDegree,      // weak / moderate / strong
    Double emotionDegreeScore
) { }
```

除了文本，还携带**情感标签 + 置信度**。这意味着部分厂商（阿里云 paraformer、火山引擎）在识别的同时能做情感判断，后续可以喂给 LLM 让它"共情"。不支持情感的模型这些字段为 `null`（`SttResult.textOnly()`）。

**已接入的厂商**：Vosk、FunASR（本地）、阿里云 NLS、阿里云、腾讯、讯飞、火山引擎。

## 三、TTS 在本项目里怎么设计的

```java
Path textToSpeech(String text) throws Exception;
```

对比 STT 一眼能看出差异：**TTS 返回的是文件路径 `Path`，不是音频流**。

原因是要发给 ESP32 的音频必须是 **Opus 编码 + 固定帧长**，而各厂商 TTS 输出的格式五花八门（wav/mp3/pcm，采样率不一）。所以流程是：

```
厂商原始音频 → 落地为文件 → 重采样 → Opus 编码 → Player 按帧调度播放 → WebSocket 发给 ESP32
```

TTS 接口还带了一组**音色控制参数**（`getVoiceName()` / `getSpeed()` / `getPitch()` / `audioFormat()`），对应角色的"声音人设"。

另外有个适配层 [TtsServiceAdapter.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-ai/src/main/java/com/xiaozhi/ai/tts/TtsServiceAdapter.java)：把自研的 `TtsService` 桥接成 Spring AI 标准的 `TextToSpeechModel`，这样既保留厂商特有能力，又能复用 Spring AI 生态。

**已接入的厂商**：sherpa-onnx（本地）、Edge TTS（免费）、阿里云、腾讯、火山引擎、讯飞、MiniMax。

## 四、一个常见混淆点：VAD ≠ STT

两者经常被搞混，但职责完全不同：

| | VAD | STT |
|---|---|---|
| 全称 | Voice Activity Detection | Speech-to-Text |
| 回答的问题 | **"有没有人在说话？说完了吗？"** | **"说的是什么？"** |
| 输出 | 布尔/状态（speech start / continue / end） | 文本 + 情感 |
| 实现 | Silero VAD（ONNX，本地，极轻量） | 各厂商 ASR 服务 |
| 作用 | **切句**——决定何时开始送音频、何时收尾 | **转写**——把音频变成文字 |

先由 VAD 切出"一句完整的话"，再把这段音频交给 STT。没有 VAD，STT 就不知道该在哪断开，只能一直等超时。

## 五、本地 vs 云端：真实的取舍

这是选型时最实际的决策点：

| | 本地（Vosk / FunASR / sherpa-onnx） | 云端（阿里/腾讯/讯飞/火山） |
|---|---|---|
| 费用 | 免费 | 按量计费 |
| 延迟 | 极低（无网络往返） | 取决于网络，通常 100ms~1s |
| 隐私 | 音频不出机器 | 音频上传第三方 |
| 精度 | 一般，方言/噪声场景差 | 高，且支持情感、热词 |
| 部署 | 需下载模型（`scripts/download_models.sh`） | 只需配 API Key |
| 离线可用 | 是 | 否 |

本项目的做法是**两者都支持，按角色配置选**——这也解释了为什么 `lib/` 和 `models/` 目录要单独下载模型文件，而它们不在 git 里。

## 六、架构模式：Provider + Factory

STT 和 TTS 与 LLM 用的是**同一套模式**：

```
SttService（接口）
   ├── VoskSttService
   ├── FunASRSttService
   ├── AliyunSttService
   └── ... 
        ↑
SttServiceFactory  ← 按角色的 configType 路由
```

所以：

- 一个设备（角色）可以配 Vosk 做 STT、火山做 TTS，自由组合；
- 新增厂商只需 3 步：实现接口 → 在 `providers/` 下建类 → 注册到对应 Factory；
- 配置变更时通过 Redis Pub/Sub 广播，各实例调用 `sttServiceFactory.removeCache(config)` 清缓存热生效（[RedisSubscriber.java](file:///d:/devFile/ai/code/github/xiaozhi-esp32-server-java/xiaozhi-dialogue/src/main/java/com/xiaozhi/communication/common/RedisSubscriber.java#L160-L180)）。

## 七、一句话总结

> **STT = 把声音变成字（耳朵），TTS = 把字变成声音（嘴巴）**。二者是 LLM 与外界的翻译层——LLM 只懂文本，靠它们才能"听说"。本项目里 STT 用流式 `Flux` 入参换低延迟、用 `SttResult` 顺带传情感；TTS 落文件 + Opus 编码后交给 Player 分帧播放；两者都用 Provider + Factory 模式支持本地/云端自由

---

