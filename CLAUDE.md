# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run Commands

### Backend (Maven, Java 21)

```bash
# Build all modules (skip tests)
mvn clean install -DskipTests

# Build a single module and its dependencies
mvn clean install -DskipTests -pl xiaozhi-server --also-make

# Run all tests
mvn test

# Run a single test class
mvn test -pl xiaozhi-server -Dtest=DeviceControllerTest

# Run a single test method
mvn test -pl xiaozhi-server -Dtest=DeviceControllerTest#testMethodName

# Start/stop services via bin scripts
bin/all.sh start          # Compile + start both server (8091) + dialogue (8092)
bin/all.sh stop           # Stop both
bin/all.sh status         # Check running status
bin/server.sh start       # Start xiaozhi-server only
bin/dialogue.sh start     # Start xiaozhi-dialogue only

# Run a single jar directly (after build)
java -Djava.library.path=lib -jar xiaozhi-server/target/xiaozhi-server-*.jar
```

The dialogue module uses a classifier `exec` for its jar (`*-exec.jar`); other modules use the standard jar.

### Frontend (Vue 3 + Vite, in `web/`)

```bash
cd web
npm run dev              # Dev server
npm run build            # Production build
npm run test:run         # Vitest unit tests
npm run test:e2e         # Playwright E2E tests
npm run lint             # Lint (oxlint + eslint)
npm run type-check       # TypeScript type checking
```

## High-Level Architecture

### Dual-Process Design

The system runs as two independent Spring Boot applications sharing MySQL + Redis:

| Process | Port | Role |
|---------|------|------|
| `xiaozhi-server` | 8091 | Admin backend: REST API, user/device/role CRUD, OTA upgrades, Flyway migrations |
| `xiaozhi-dialogue` | 8092 | Dialogue service: WebSocket audio streaming, AI conversation pipeline |

`dialogue` supports horizontal scaling — new instances self-register in Redis, and the server distributes devices via OTA configuration. Flyway migrations run only on `xiaozhi-server`.

### Maven Module Map

```
xiaozhi-parent (pom)
├── xiaozhi-common       # Enums, events, shared interfaces (ServerAddressProvider, DialogueServerRegistry, RedisBroadcast)
├── xiaozhi-service      # Domain services, MyBatis mappers, DDD repositories, DTOs, MapStruct converters
├── xiaozhi-ai           # LLM/TTS/STT providers, MCP tool protocol, memory/conversation, tool calling
├── xiaozhi-server       # Admin REST controllers, Flyway migrations, web-chat, server-specific config
└── xiaozhi-dialogue     # WebSocket handler, DialogueService, Persona factory, audio pipeline (VAD, playback)
```

Module composition is done via explicit `@ComponentScan` basePackages in each `*Application.java`, not auto-discovery. When adding a new package, update the relevant `@ComponentScan` and `@MapperScan` annotations.

### Dialogue Pipeline (the core data flow)

```
ESP32 Device → WebSocket → Binary audio (Opus)
  → VAD (Silero VAD, ONNX) — detects speech start/continue/end
  → STT (Vosk/FunASR/Aliyun/Tencent/Xunfei) — speech to text
  → [Emotion detection] → UserMessage with metadata
  → LLM (OpenAI/Zhipu/Ollama/Dify/Coze/Xinghuo) — with tools + memory
  → TTS (sherpa-onnx/Edge/Aliyun/Tencent/Volcengine/Xunfei) — text to Opus audio
  → Player (scheduled playback with interrupt support)
  → WebSocket → ESP32 Device
```

Key classes in this pipeline:
- `WebSocketHandler` — accepts connections, parses hello/binary/text messages
- `SessionManager` — manages `ChatSession` instances with deviceId→sessionId index
- `DialogueService` — orchestrates VAD → STT → text handling, handles abort/wake-word
- `Persona` — aggregates ChatModel + Synthesizer + Player + Conversation for a role; the core dialogue loop lives here (`chat()` method)
- `PersonaFactory` — creates Persona per session based on the device's assigned role config
- `Player` — scheduled Opus frame playback; supports interrupt (abort) and post-chat callbacks
- `SynthesizerFactory` — selects TTS provider per role config

### AI Module (`xiaozhi-ai`) Structure

- **LLM**: Provider pattern — `ChatModelProvider` interface with implementations in `ai/llm/factory/providers/`. Each provider (OpenAI, Zhipu, Ollama, Dify, Coze, Xinghuo, Xingchen) creates a Spring AI `ChatModel`. `ChatModelFactory` routes to the correct provider based on config.
- **Memory**: `Conversation` abstraction with `MessageWindowConversation` (sliding window) and `SummaryConversation` (summarization-based). `ConversationFactory` and `DatabaseChatMemory` handle persistence.
- **STT**: `SttService` interface, `SttServiceFactory` selects provider (Vosk, FunASR, Aliyun NLS, Tencent, Xunfei, Volcengine).
- **TTS**: `TtsService` interface, `TtsServiceFactory` selects provider (sherpa-onnx local, Edge TTS, Aliyun, Tencent, Volcengine, Xunfei, MiniMax). `TtsServiceAdapter` bridges to Spring AI's Speech API.
- **Tools**: `XiaoZhiToolCallingManager` — custom tool calling manager extending Spring AI's default, with a fix for streaming tool call chunk merging (known Spring AI issue). `ToolRegistrar` pattern for registering MCP and system tools per session.

### Communication Layer

- **WebSocket** (`xiaozhi-dialogue/communication/server/websocket/`): Primary device transport. Devices connect with `device-id` header, send a JSON `hello` message to negotiate audio params, then stream binary Opus audio.
- **Redis Pub/Sub**: Cross-instance communication — `RedisSubscriber` listens on Redis channels, `RedisBroadcast` publishes. Used for dialogue server discovery and cross-instance device messaging.
- **Message protocol**: JSON text messages for control (hello, goodbye, listen, abort, IoT, MCP), binary for Opus audio frames. Message types defined in `communication/domain/`.

### Service Layer DDD Pattern

Each domain in `xiaozhi-service` follows this structure:
```
<domain>/
├── dal/mysql/
│   ├── dataobject/<Domain>DO.java   # MyBatis-Plus entity
│   └── mapper/<Domain>Mapper.java   # MyBatis-Plus mapper
├── domain/
│   ├── <Domain>.java                # Domain model
│   ├── repository/<Domain>Repository.java
│   └── vo/*.java                    # Value objects
├── infrastructure/
│   ├── <Domain>RepositoryImpl.java
│   └── convert/<Domain>Converter.java
├── service/
│   ├── <Domain>Service.java         # Interface
│   └── impl/<Domain>ServiceImpl.java
└── convert/<Domain>Convert.java     # MapStruct converter (DO ↔ VO/BO)
```

Key rules enforced by ArchUnit tests:
- Service layer must NOT depend on `*Req` DTOs (controllers should unwrap them)
- `map-underscore-to-camel-case: false` — MyBatis maps column names directly (no auto camelCase)

### Configuration & Infrastructure

- **DB**: MySQL 8.0, Flyway migrations in `xiaozhi-server/src/main/resources/db/migration/`, HikariCP connection pool
- **Redis**: Spring Data Redis (Lettuce with connection pooling) + Redisson for distributed locks/coordination
- **Auth**: Sa-Token with JWT + Redis, supports cookie and header tokens with Bearer prefix
- **API Docs**: Knife4j + SpringDoc OpenAPI at `/swagger-ui.html` and `/v3/api-docs`
- **Native libs**: `lib/` directory for JNI native libraries (Vosk, sherpa-onnx); models in `models/` directory
- **Virtual threads**: Enabled via `spring.threads.virtual.enabled: true` — STT processing and MCP initialization use `Thread.startVirtualThread()`
- **Frontend** (`web/`): Vue 3 + TypeScript + Ant Design Vue + Pinia + Vue Router. Vite for build, Vitest + Playwright for testing

### Key Architectural Decisions

1. **Persona owns the dialogue loop**: LLM, TTS, Player, and Conversation are all aggregated inside Persona. Sessions hold a reference to Persona, not the individual components.
2. **Device state machine**: `DeviceState` enum (IDLE → LISTENING → THINKING → SPEAKING) replaces scattered boolean flags.
3. **Abort/interrupt**: VAD-detected speech during TTS playback triggers abort. `DialogueService.abortDialogue()` cancels the Synthesizer flux upstream, then stops the Player downstream, in that order to prevent audio overlap.
4. **Tool session isolation**: Each dialogue session gets its own set of tool callbacks via `ToolsSessionHolder` — tools are not shared across sessions.
5. **Model files are NOT in git**: Vosk models, sherpa-onnx TTS models, and native libraries must be downloaded via `scripts/download_models.sh` (or `download_base.sh` for VAD-only) before first run.

### Adding a New LLM/TTS/STT Provider

1. Implement the provider interface (`ChatModelProvider`, `TtsService`, `SttService`)
2. Add the provider class under `xiaozhi-ai/src/main/java/com/xiaozhi/ai/<layer>/providers/`
3. Register it in the corresponding factory (`ChatModelFactory`, `TtsServiceFactory`, `SttServiceFactory`)
4. Add any new dependencies to the root `pom.xml` `<dependencyManagement>` and to the relevant module's `pom.xml`
