# CODEBUDDY.md This file provides guidance to CodeBuddy when working with code in this repository.

## Common Commands

### Backend (mateclaw-server)

**Build the server (skip tests):**
```bash
cd mateclaw-server && mvn package -Dmaven.test.skip=true
```
Use `-Paliyun-first` to prioritize Aliyun Maven mirrors (faster inside mainland China).

**Install plugin-api first (required before building server):**
```bash
cd mateclaw-plugin-api && mvn install -Dmaven.test.skip=true
```
The server depends on `mateclaw-plugin-api` as a local SNAPSHOT. Always install it before the first server build.

**Run all tests:**
```bash
cd mateclaw-server && mvn test
```
Full suite (~50 min). 271 test classes using JUnit 5 + Mockito + ArchUnit. Uses H2 file-based DB at `./data/mateclaw`.

**Run a single test class:**
```bash
cd mateclaw-server && mvn test -Dtest=WikiProcessingServiceTest
```

**Run media-gen tests only:**
```bash
cd mateclaw-server && mvn test -P media-gen
```
Runs only tests tagged `@Tag("media-gen")` (image/video generation features).

**Start dev server (H2 profile):**
```bash
cd mateclaw-server && mvn spring-boot:run
```
Listens on port 18088. H2 database auto-created at `./data/mateclaw`. No MySQL needed for dev.

**Start dev server with MySQL:**
```bash
cd mateclaw-server && mvn spring-boot:run -Dspring-boot.run.profiles=mysql
```

**Docker build (full pipeline):**
```bash
docker compose build
```
Multi-stage build: frontend (Node 22 + pnpm 10) → backend (Maven) → runtime (Playwright image). Set `MAVEN_FLAGS=-Paliyun-first` in `.env` for faster Maven downloads in CN.

### Frontend (mateclaw-ui)

**Start dev server:**
```bash
cd mateclaw-ui && pnpm install && pnpm dev
```
Vite dev server on port 5173, proxies `/api` to `localhost:18088`.

**Build for production:**
```bash
cd mateclaw-ui && pnpm build
```
Runs `vue-tsc` type-check then Vite build. Output goes to `../mateclaw-server/src/main/resources/static` (embedded in JAR).

**Lint:**
```bash
cd mateclaw-ui && pnpm lint
```
ESLint on `.ts` and `.vue` files with auto-fix.

## Architecture

### Project Overview

MateClaw is a self-hosted personal AI assistant platform — Spring Boot 3.5 backend + Vue 3 frontend, licensed Apache 2.0. It provides reasoning, knowledge (LLM Wiki), memory, tool execution, and multi-channel entry points, all from a single JAR.

### Module Map

| Module | Role |
|---|---|
| `mateclaw-server` | Core Spring Boot application (port 18088). Contains all business logic, AI agent runtime, plugin host. |
| `mateclaw-ui` | Vue 3 + TypeScript admin SPA (Vite, Element Plus, Tailwind CSS). Built output lands in server's `static/` for single-JAR deployment. |
| `mateclaw-webchat` | Embeddable chat widget (UMD/ES bundles). Independent build from the admin UI. |
| `mateclaw-plugin-api` | Java SDK for third-party plugins. Defines SPI contracts: `MateClawPlugin`, `PluginContext`, `PluginManifest`, channel adapters, memory providers. |
| `mateclaw-plugin-sample` | Reference plugin implementation showing how to register tools via the SDK. |

### Server Package Architecture

The server is organized into domain packages under `vip.mate.*`, each a self-contained vertical slice:

- **`agent`** — Core AI agent runtime. Contains `AgentGraphBuilder` (125KB, the largest file — defines StateGraph nodes and transitions), `BaseAgent`, tool bindings, delegation logic, and a runtime console for admin observation. Supports ReAct (iterative reason-act-observe) and Plan-and-Execute patterns.
- **`llm`** — LLM provider management. Multi-vendor with health tracking and automatic failover (DashScope, OpenAI, Anthropic, Gemini, DeepSeek, Kimi, Ollama, 14+ total). Failed providers enter a cooldown window. Model config is admin-UI driven, keys stored in `mate_model_provider` table and hot-reloaded.
- **`wiki`** — LLM Wiki knowledge pipeline. `WikiProcessingService` (122KB) digests raw material (PDF, Markdown, HTML) into structured pages with `[[links]]`, citation tracking, chunking, and embedding. Injects wiki context into agent system prompts.
- **`memory`** — Memory lifecycle system. Post-conversation extraction, scheduled consolidation, Dream workflow (focus & archive), fact projection with half-life decay. See `MemoryEmergenceService`.
- **`skill`** — SKILL.md package system. Installs, runs, and sandboxes skill bundles. Supports LESSONS.md self-evolution, composition, templates, and ClawHub discovery.
- **`channel`** — Multi-channel adapters. Independent adapters for DingTalk, Feishu, WeChat Work, WeChat, Telegram, Discord, QQ, Slack, Web. Each adapter has isolated error classification and health monitoring. `ChannelManager` maintains an active-adapter registry with ReadWriteLock for hot-swap.
- **`tool`** — Tool registry and built-in tools: browser automation (Playwright), shell execution, file operations, document generation, MCP client management, daemon/security tools.
- **`plugin`** — Plugin lifecycle management. `PluginManager` scans workspace and user-global plugin JARs at startup, loads via isolated `URLClassLoader`, bridges registrations to Tool/Channel/Memory/Provider services. Extension points: TOOL, PROVIDER, CHANNEL, MEMORY.
- **`workflow`** — Declarative workflow engine with Pebble template-condition compilation.
- **`approval`** — Approval-gated sensitive operations. Workflow ensures human sign-off before execution.
- **`workspace`** — Multi-workspace support (conversations, files, documents, members, core CRUD).
- **`audit` / `activity`** — Full audit trail and activity feeds.
- **`auth`** — JWT authentication, personal access tokens, Spring Security integration.
- **`acp`** — Agent Communication Protocol bridge to external coding agents (Claude Code, Codex).
- **`cron`** — Distributed cron scheduling with ShedLock (JDBC-based) to prevent duplicate execution across instances.
- **`trigger`** — Event-driven trigger system for agent lifecycle, channel messages, workflow completions.
- **`dashboard` / `system` / `task`** — Monitoring, health checks, feature toggles, async task service.
- **`stt` / `tts`** — Speech-to-text and text-to-speech with multi-provider support.
- **`i18n`** — Internationalization (`messages.properties` / `messages_en.properties`).

### Data Layer

- **ORM**: MyBatis Plus 3.5.16 (NOT JPA). 63 Mapper interfaces in `**.repository` packages, scanned by `@MapperScan("vip.mate.**.repository")`. 60 Entity classes. No JPA repositories — use MyBatis Plus `BaseMapper<T>` pattern exclusively.
- **Schema migration**: Flyway with two separate migration paths — `classpath:db/migration/h2/` for dev and `classpath:db/migration/mysql/` for production. Active profile selects the path. Current migration level: V110+.
- **Dev database**: H2 file-based (`jdbc:h2:file:./data/mateclaw;MODE=MySQL`) — MySQL compatibility mode enabled.
- **Production database**: MySQL 8.0 with `utf8mb4_unicode_ci` collation, configured via `application-mysql.yml` and environment variables.
- **Seed data**: `DatabaseBootstrapRunner` handles initial setup — supports desktop mode (waits for language selection) and web mode (auto-initializes with zh-CN).
- **Pagination**: `PaginationInnerInterceptor` without hardcoded `DbType` — auto-detects from JDBC connection at runtime.

### Key Architectural Patterns

1. **Spring MVC (Servlet), NOT WebFlux**. WebFlux dependencies are explicitly excluded from POMs. SSE streaming for chat responses, WebSocket for Talk Mode.
2. **Virtual threads** enabled (`spring.threads.virtual.enabled: true`).
3. **MCP client lifecycle** is self-managed (`McpClientManager`), NOT Spring Boot auto-configuration. All MCP auto-config classes are excluded in `MateClawApplication`.
4. **Plugin classloader isolation**: Each plugin JAR gets its own `URLClassLoader`. `PluginManager` handles load → enable → disable lifecycle with rollback support.
5. **Provider health tracking**: LLM providers are monitored for consecutive failures. After a threshold, they enter a cooldown window and are skipped on subsequent requests.
6. **Channel isolation**: Each IM channel adapter runs independently. Failure of one channel does not affect others.
7. **Pagination plugin**: `PaginationInnerInterceptor` without hardcoded database type — detects dialect from the JDBC connection at runtime (critical: hardcoding H2 caused MySQL pagination to silently return 0).
8. **Docker runtime uses Playwright image**: `mcr.microsoft.com/playwright:v1.52.0-noble` — Chromium, Firefox, WebKit pre-installed. `PLAYWRIGHT_BROWSERS_PATH=/ms-playwright` tells the Java driver where browsers live. The Playwright version in `pom.xml` MUST match the Docker image tag.

### Plugin Development

Third-party plugins extend MateClaw through four extension points defined in `mateclaw-plugin-api`:
- **TOOL** — Register Spring AI `ToolCallback` beans exposed to agents
- **PROVIDER** — Register custom `ChatModel` implementations
- **CHANNEL** — Register new IM channel adapters
- **MEMORY** — Register external memory providers (only one active at a time)

Plugins implement `MateClawPlugin` interface. A `mateclaw-plugin.json` manifest declares name, version, type, entry point class, and config schema. `PluginManager` discovers JARs from `{workspace}/plugins/` and `~/.mateclaw/plugins/`.

### Testing

- 271 test classes in `mateclaw-server/src/test`
- JUnit 5 (`@Test`, `@Tag`), Mockito (inline mock maker), ArchUnit (architecture invariants)
- No separate test `application.yml` — tests use the default dev profile with H2
- ByteBuddy agent is statically attached via Surefire `-javaagent` arg (required for Mockito on JDK 21+)
- `media-gen` Maven profile runs only `@Tag("media-gen")` tests for faster iteration on image/video features
