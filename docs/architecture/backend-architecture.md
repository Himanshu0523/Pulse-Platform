# Backend Architecture

> **Scope:** Express server, middleware pipeline, route organization, services, and jobs.

---

## 1. Process Entry (`server.js`)

The server boots in this order:

1. `dotenv` loads env vars
2. `app.js` builds the Express application
3. `http.createServer(app)` wraps it for Socket.IO
4. `connectDB()` opens the Mongoose connection (with retry)
5. `initRedis()` opens pub/sub ioredis clients
6. Redis adapter is attached to Socket.IO for horizontal scale readiness
7. `mediasoupManager.init()` spawns C++ worker processes (graceful — server starts even if this fails)
8. `server.listen(PORT)` opens the HTTP port
9. Background jobs start: `messageBufferWorker`, `cleanupUnverifiedUsersJob`

### Graceful Shutdown

On `SIGTERM` / `SIGINT`:
1. Stop message buffer worker, flush pending writes to MongoDB
2. Stop HTTP server (drain in-flight requests)
3. Close Mediasoup workers
4. Close Mongoose connection
5. Quit Redis pub/sub clients
6. Force-exit after 10 s if connections hang

---

## 2. Express Middleware Stack (`app.js`)

Middleware runs in this exact order on every request:

```
requestIdMiddleware       → X-Request-Id header (unique per request)
requestContext            → AsyncLocalStorage for request tracing
perfTimingMiddleware      → records start time for duration logging
loggingMiddleware         → structured request/response logging (winston)
metricsMiddleware         → prom-client counter/histogram increment
helmet()                  → security headers (CSP, HSTS, etc.)
compression()             → gzip/brotli response compression
rawBody capture           → only on /api/webhooks (HMAC verification)
express.json({ 50mb })    → JSON body parsing
express.urlencoded        → form body parsing
cookieParser              → cookie parsing (refresh token cookie)
generateCsrfToken         → CSRF token generation (double-submit cookie)
verifyCsrfToken           → CSRF token validation (skips GET/OPTIONS)
morgan("dev")             → dev request logging
─── Routes ───
authMiddleware            → JWT verification (injected at /api boundary)
auditMiddleware           → writes AuditLog records for state-changing ops
errorHandler              → centralized error shaping + response
```

---

## 3. Route Organization

Routes are grouped into **modules** (`src/modules/`) and mounted in `app.js`:

| Mount Path | Module / File | Auth Required |
|-----------|---------------|---------------|
| `/` + `/api` | `health.routes.js` | No |
| `/metrics` | prom-client handler | No |
| `/api/csrf-token` | inline handler | No |
| `/api/auth` | `authModule.routes` | No |
| `/api/webhooks` | `webhooksModule.routes` | HMAC sig |
| `/api/integrations` (catalog) | `integrationCatalog.routes` | No |
| `/api/docs` | `docs.routes` | No |
| **— authMiddleware + auditMiddleware injected here —** | | |
| `/api/rooms` | `callsModule.roomRoutes` | ✅ |
| `/api/users` | `authModule.userRoutes` | ✅ |
| `/api/friends` | `friendsModule.routes` | ✅ |
| `/api/calls` | `callsModule.callRoutes` | ✅ |
| `/api/conversations` | `chatModule.conversationRoutes` | ✅ |
| `/api/groups` | `chatModule.groupRoutes` | ✅ |
| `/api/webrtc` | `callsModule.webrtcRoutes` | ✅ |
| `/api/files` | `filesModule.routes` | ✅ |
| `/api/notifications` | `notificationsModule.notificationRoutes` | ✅ |
| `/api/presence` | `notificationsModule.presenceRoutes` | ✅ |
| `/api/keys` | `key.routes` | ✅ |
| `/api/ai` | `aiModule.routes` | ✅ |
| `/api/workspaces` | `workspaceModule.routes` | ✅ |
| `/api/knowledge` | `knowledgeModule.routes` | ✅ |
| `/api/messages/:id/thread` | `chatModule.threadRoutes` | ✅ |
| `/api/messages` | `chatModule.scheduledMessageRoutes` + `chatModule.messageRoutes` | ✅ |
| `/api/polls` | `chatModule.pollRoutes` | ✅ |
| `/api/compliance` | `compliance.routes` | ✅ |
| `/api/search` | `search.routes` | ✅ |
| `/api/integrations` (extended) | routing, events, GitHub, actions, custom webhooks, email bridge | ✅ |

---

## 4. Authentication & Authorization

### HTTP (REST)
- **Access token:** JWT (15-min expiry) — sent as `Authorization: Bearer <token>`
- **Refresh token:** JWT (7-day expiry) — stored in `httpOnly` Secure cookie
- **CSRF:** Double-submit cookie pattern — CSRF token generated server-side, validated on every state-changing request
- **Token families:** `tokenFamily.service.js` detects refresh token reuse (token rotation attack mitigation)
- **Account lockout:** `accountLock.js` middleware — exponential backoff after failed login attempts
- **2FA:** TOTP via `speakeasy` (`2fa.service.js`)
- **WebAuthn:** Passkey support (`webauthn.service.js`)
- **GitHub OAuth:** `githubAuth.middleware.js` + `githubAuth.routes.js`

### Socket.IO
- JWT extracted from `socket.handshake.auth.token` or `Authorization` header
- `socket.userId` set immediately from JWT (`fast-path`)
- DB user fetch is non-blocking; socket continues even if DB is slow
- Invalid token → `next(new Error("Invalid token"))` → client redirected to `/login`

---

## 5. Service Layer

Key services and their responsibilities:

| Service | Owns | Modifies | Fails? |
|---------|------|----------|--------|
| `messageBuffer.service` | In-memory write buffer | `Message` (batch write) | Messages delayed, not lost |
| `notification.service` | Push to user socket rooms | `Notification` | Silent — no notification delivered |
| `presence.service` | Online status in Redis | Redis presence keys | Users appear offline |
| `knowledge.service` | Article CRUD + AI search | `KnowledgeArticle`, `KnowledgeQuota` | Knowledge features unavailable |
| `ai.service` | Groq SDK calls | None (stateless) | AI features return empty |
| `email.service` | SMTP sends | None (stateless) | Email notifications fail silently |
| `credential.service` | Encrypted OAuth tokens | `UserIntegrationCredential` | Integration actions fail |
| `scheduledMessage.service` | Future message delivery | `ScheduledMessage`, `Message` | Messages not sent on time |
| `user-deletion.service` | Full account wipe | All user-owned data | Manual cleanup required |
| `embedding.service` | Vector embeddings for search | Redis / external vector store | Semantic search degrades |
| `cache.service` | Redis get/set/del wrapper | Redis | Falls back to DB queries |

---

## 6. Background Jobs

| Job | Trigger | What It Does |
|-----|---------|--------------|
| `messageBuffer.job` | Every 3 s + shutdown flush | Batches socket-received messages into MongoDB |
| `messageQueue.job` | On socket connection | Delivers scheduled messages past their send time |
| `cleanupUnverifiedUsers.job` | node-cron (periodic) | Deletes accounts with expired email verification |
| `linkPreview.queue` | BullMQ queue | Fetches URL Open Graph data for chat link previews |
| `integrationWorker` | BullMQ queue | Processes inbound webhook events through integration pipeline |

---

## 7. Observability

| Signal | Implementation |
|--------|---------------|
| Metrics | `prom-client` — request count, duration histograms at `/metrics` |
| Logging | `winston` structured logging via `loggingMiddleware` |
| Request tracing | `requestId` middleware → `X-Request-Id` header on all responses |
| Performance | `perfTimingMiddleware` — records per-request duration |
| Audit | `auditMiddleware` → writes `AuditLog` documents for state-changing ops |
| Anomaly detection | `anomalyDetection.service` — unusual access pattern alerting |
| Compliance | `ComplianceLog` model + `compliance.routes` — data retention reporting |
