# Repository Guidelines

## Project Structure & Module Organization

This is a Java 21 / Spring Boot 3.5 multi-module Maven project with a dual-process architecture:

- `xiaozhi-common` — shared utilities, constants, and base classes
- `xiaozhi-service` — business logic and data access (MyBatis-Plus, Flyway migrations in `src/main/resources/db`)
- `xiaozhi-ai` — AI platform integrations (OpenAI, Ollama, Dify, Coze, etc.)
- `xiaozhi-dialogue` — dialogue service (:8092), handles WebSocket/MQTT real-time audio and AI pipelines
- `xiaozhi-server` — management backend (:8091), REST API, user/device/role management, OTA
- `web/` — Vue 3 + TypeScript frontend (Ant Design Vue, Vite), with `src/views`, `src/components`, `src/services`, `src/store`
- `scripts/` — model download scripts (`download_models.sh`, `download_stt.sh`, `download_tts.sh`)
- `bin/` — startup scripts (`all.sh`, `server.sh`, `dialogue.sh`)
- `lib/` — native libraries (onnxruntime, sherpa-onnx, vosk)
- `models/` — local STT/TTS model files

## Build, Test, and Development Commands

```bash
# Backend: full build (from project root)
mvn clean install -DskipTests

# Backend: run management server
java -Djava.library.path=lib -jar xiaozhi-server/target/xiaozhi-server-*.jar

# Backend: run dialogue service
java -Djava.library.path=lib -jar xiaozhi-dialogue/target/xiaozhi-dialogue-*-exec.jar

# Or use bin scripts (Git Bash / WSL)
bin/all.sh start        # compile and start both services
bin/all.sh status       # check status

# Frontend (from web/)
npm run dev             # start Vite dev server
npm run build           # production build with type-check
npm run test:run        # run Vitest unit tests
npm run test:e2e        # run Playwright E2E tests
npm run lint            # oxlint + eslint with auto-fix
npm run format          # Prettier formatting
```

## Coding Style & Naming Conventions

**Backend (Java):** 4-space indentation, standard Java conventions. Class names in `PascalCase`, methods and variables in `camelCase`. Use Lombok for boilerplate reduction. Package structure follows `com.xiaozhi.*`.

**Frontend (TypeScript/Vue):** 2-space indentation, LF line endings. No semicolons, single quotes (Prettier config). Composables prefixed with `use` (e.g., `useAuth.ts`, `useWebSocket.ts`). Components use `PascalCase` filenames. Services map 1:1 to backend API modules.

**Config:** Application config in `application.yml` / `application-dev.yml` / `application-prod.yml`. Environment variables in `.env`.

## Testing Guidelines

- **Frontend unit tests:** Vitest with `@vue/test-utils`, located in `src/__tests__/` and alongside components. Run with `npm run test:run`.
- **Frontend E2E tests:** Playwright, configured in `playwright.config.ts`. Run with `npm run test:e2e`.
- **Backend tests:** JUnit via Maven (`mvn test`). Place tests in each module's `src/test/java`.

## Commit & Pull Request Guidelines

Commit messages follow Conventional Commits with Chinese descriptions:

```
feat: 新增web聊天
fix: 修复打断后没有正确切换isPlaying状态
refactor: 重构toolcalls
update: 更新log采用注解
```

Types used: `feat`, `fix`, `refactor`, `update`. Pull requests should include a clear description of changes, link related issues, and note any database migration or config changes required.

## Architecture Notes

The system uses a **dual-process architecture** — `xiaozhi-server` (management) and `xiaozhi-dialogue` (real-time conversation) share MySQL 8.0 and Redis 7 but run independently. The dialogue service supports horizontal scaling; new instances auto-register with the server. Database schema is managed by Flyway — never edit SQL manually; add versioned migration files under `xiaozhi-server/src/main/resources/db`.
