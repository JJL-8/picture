# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Yu-Picture (智能协同云图库) is a collaborative cloud image library platform built with Vue 3 + Spring Boot. It supports public image browsing, private spaces, team spaces with real-time collaborative editing via WebSocket, and AI-powered image processing.

The repo contains three modules:
- **yu-picture-frontend** — Vue 3 SPA (TypeScript)
- **yu-picture-backend** — Spring Boot 2.7.6 backend (Java 8, standard layered architecture)
- **yu-picture-backend-ddd** — Alternative backend using DDD (Domain-Driven Design) architecture

## Build & Run Commands

### Frontend (`yu-picture-frontend/`)
```bash
npm run dev          # Start Vite dev server
npm run build        # Type-check + production build
npm run pure-build   # Production build without type-check
npm run lint         # ESLint with auto-fix
npm run format       # Prettier format src/
npm run type-check   # vue-tsc type checking
npm run openapi      # Generate API client code from backend Swagger (requires backend running on :8123)
```

### Backend (`yu-picture-backend/`)
```bash
mvn spring-boot:run                    # Run the application
mvn package                            # Build JAR
mvn test                               # Run tests
mvn test -Dtest=YuPictureBackendApplicationTests  # Run a single test class
```
Main class: `com.yupi.yupicturebackend.YuPictureBackendApplication`

Backend runs on port **8123**. The API doc (Knife4j/Swagger) is available at `http://localhost:8123/api/doc.html`.

### Infrastructure
```bash
docker-compose up -d   # Start MySQL 8.0 (port 30306) and Redis 7 (port 30379)
```
Database: `yu_picture`, schema in `yu-picture-backend/sql/create_table.sql`

## Architecture

### Backend (Standard — `yu-picture-backend`)

Package: `com.yupi.yupicturebackend`

| Layer | Package | Purpose |
|-------|---------|---------|
| Controller | `controller` | REST endpoints (User, Picture, Space, SpaceUser, SpaceAnalyze, File) |
| Service | `service` / `service.impl` | Business logic |
| Mapper | `mapper` | MyBatis-Plus data access (XML in `resources/mapper/`) |
| Model | `model.entity` | DB entities: User, Picture, Space, SpaceUser |
| Model | `model.dto` | Request DTOs |
| Model | `model.vo` | View objects (response) |
| Model | `model.enums` | Enums: UserRole, SpaceType, SpaceLevel, SpaceRole, PictureReviewStatus |
| Manager | `manager` | Cross-cutting managers (COS, file upload, auth, WebSocket, sharding) |
| Config | `config` | CORS, COS client, MyBatis-Plus, JSON serialization |
| API | `api` | External API integrations (Aliyun AI out-painting, image search) |

Key subsystems in `manager/`:
- **auth/** — Sa-Token based RBAC permission system. Roles (admin/editor/viewer) and permissions loaded from `resources/biz/spaceUserAuthConfig.json`. `SpaceUserAuthManager` resolves permissions based on space type (private/team/public).
- **upload/** — Template method pattern for picture uploads (`PictureUploadTemplate` base, `FilePictureUpload` and `UrlPictureUpload` implementations).
- **websocket/** — Real-time collaborative picture editing using Spring WebSocket + Disruptor lock-free queue for event processing.
- **sharding/** — ShardingSphere dynamic sharding for the picture table by spaceId.

### Backend (DDD — `yu-picture-backend-ddd`)

Package: `com.yupi.yupicture`

| Layer | Package | Purpose |
|-------|---------|---------|
| interfaces | `interfaces.controller` / `interfaces.dto` / `interfaces.vo` / `interfaces.assembler` | API layer with DTO↔Entity mapping |
| application | `application.service` | Application services (orchestration) |
| domain | `domain.{picture,space,user}` | Domain entities, value objects, repositories (interfaces), domain services |
| infrastructure | `infrastructure.*` | Persistence (MyBatis-Plus), external APIs, configs, managers |
| shared | `shared.auth` / `shared.websocket` / `shared.sharding` | Cross-domain concerns |

### Frontend (`yu-picture-frontend/src/`)

- **Vue 3** (Composition API) + **Pinia** state management + **Vue Router**
- **Ant Design Vue 4** component library
- **Axios** HTTP client with interceptors in `request.ts` — base URL is `/api`, auto-redirects to login on 401 (code 40100)
- **OpenAPI code generation** — API client types and functions auto-generated into `src/api/` from backend Swagger via `@umijs/openapi` (`openapi.config.js`)
- **Route-level access control** in `access.ts` — admin routes (`/admin/*`) require `userRole === 'admin'`
- **State** — Single Pinia store `useLoginUserStore` manages auth state
- Path alias: `@` → `src/`

Core entities mapped from backend: User, Picture, Space, SpaceUser.

Pages: HomePage, picture CRUD, space management, space analysis (ECharts), admin pages, collaborative editing.

## Key Technical Details

- **Auth**: Sa-Token for backend session management, stored in Redis. Frontend uses `withCredentials: true` for cookie-based sessions.
- **File Storage**: Tencent Cloud COS (configured via `CosClientConfig`).
- **Caching**: Redis (distributed) + Caffeine (local) multi-level cache.
- **Database**: MySQL 8.0 with MyBatis-Plus. Logical delete (`isDelete` field) on all entities.
- **API Response Format**: Wrapped in `BaseResponse<T>` with `code`, `data`, `message` fields. Use `ResultUtils` to construct responses.
- **Error Handling**: `BusinessException` with `ErrorCode` enum, handled globally by `GlobalExceptionHandler`.
- **HTTP Test Files**: IntelliJ HTTP client test files in `yu-picture-backend/httpTest/`.

## Deployment

### Server Information

| Item | Value |
|------|-------|
| Deploy Path | `/root/yu-picture/backend` |
| JAR Name | `yu-picture-backend-0.0.1-SNAPSHOT.jar` |
| Port | 8123 |
| Profile | prod |
| JVM Args | `-Xms512m -Xmx1024m` |

### Deploy Commands

```bash
# 1. Build JAR locally
mvn clean package -DskipTests

# 2. Upload to server
scp yu-picture-backend/target/yu-picture-backend-0.0.1-SNAPSHOT.jar root@<SERVER_IP>:/root/yu-picture/backend/

# 3. SSH to server and restart
ssh root@<SERVER_IP>
cd /root/yu-picture/backend

# Check existing process
ps -ef | grep yu-picture

# Stop old process (replace <PID> with actual process ID)
kill <PID>

# Start new service
nohup java -jar -Xms512m -Xmx1024m -Dspring.profiles.active=prod yu-picture-backend-0.0.1-SNAPSHOT.jar > app.log 2>&1 &

# Verify startup
tail -f app.log
```

### Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| Multiple Java processes running | Previous `kill` failed, started multiple times | Use `kill -9 <PID>` to force kill all, then start one |
| CPU 100% after deploy | Multiple processes competing for resources | `ps -ef \| grep yu-picture` to check, kill duplicates |
| SSH connection lost | Shell stuck from foreground process | Open new SSH connection, kill stuck process |
| Service won't start | Port occupied (8123) | `netstat -tunlp \| grep 8123` to find process |
| High CPU during startup | Normal JVM warmup | Wait 30-60 seconds, CPU should drop |

### Important Notes

1. **Always verify no old processes before starting** — Multiple processes will cause resource competition
2. **Use `kill` (SIGTERM) first, `kill -9` (SIGKILL) only if needed** — Graceful shutdown allows cleanup
3. **Check `app.log` for startup errors** — Look for "Started YuPictureBackendApplication" to confirm success
4. **If CPU stays high for >2 minutes** — Likely a code issue, rollback to previous version
