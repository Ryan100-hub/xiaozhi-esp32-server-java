# GitHub Copilot Instructions — Xiaozhi ESP32 Server Java

## 项目简介
Xiaozhi ESP32 Server Java — 为 Xiaozhi ESP32 智能硬件设备提供企业级 Java 后端和 Vue 3 前端管理平台。

## 技术栈速查
- Java 21 + Spring Boot 3.5.8 + Spring AI 1.1.4 + Maven 多模块
- Vue 3.5 + TypeScript 5.9 + Vite 7 + Ant Design Vue 4 + Pinia 3
- MySQL 8.0 (Flyway) + Redis 7 (Lettuce + Redisson)
- Sa-Token JWT 认证, MyBatis-Plus ORM, Knife4j API 文档

## 架构要点
- **双进程**: xiaozhi-server(:8091 管理后台) + xiaozhi-dialogue(:8092 对话服务)
- **对话管道**: ESP32→WebSocket→VAD→STT→LLM→TTS→Player→WebSocket→ESP32
- **DDD 分层**: dal(mysql) → domain → infrastructure → service → convert
- **Provider 模式**: LLM/STT/TTS 均通过 Factory + Provider 接口支持多厂商

## 代码规范
### Java
- 4 空格缩进, Lombok 注解, `com.xiaozhi.*` 包结构
- Service 层不依赖 `*Req` DTO, Controller 负责解包
- MyBatis 列名直接映射 (不自动驼峰)
- 新增包需更新 `@ComponentScan` 和 `@MapperScan`

### TypeScript/Vue
- 2 空格缩进, 无分号, 单引号
- Composable 以 `use` 前缀命名
- Service 与后端 API 1:1 映射

### 提交信息
Conventional Commits + 中文: `feat:`, `fix:`, `refactor:`, `update:`

## 关键约束
- Flyway 迁移文件在 `xiaozhi-server/src/main/resources/db/migration/`, 只增不改
- dialogue 模块 jar 使用 `exec` 分类器 (`*-exec.jar`)
- sherpa-onnx 托管在项目 `repo/` 目录
- 虚拟线程已启用, 异步任务使用 `Thread.startVirtualThread()`
- 模型文件不入 Git, 需通过 `scripts/download_models.sh` 下载

## 构建命令
```bash
mvn clean install -DskipTests                    # 后端全量构建
cd web && npm run dev                            # 前端开发服务器
cd web && npm run build                          # 前端生产构建
```
