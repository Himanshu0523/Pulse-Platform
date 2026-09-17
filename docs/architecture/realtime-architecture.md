# Real-Time Architecture

> **Scope:** Socket.IO setup, connection lifecycle, event namespacing, socket handlers, and state recovery.

---

## 1. Transport Layer

Pulse uses a **single global Socket.IO server** attached to the same HTTP server as Express.

```
Client (Browser)
    │
    │  1. HTTP long-polling (initial handshake)
    │  2. WebSocket upgrade (if supported)
    ▼
Socket.IO Server (port 5000)
    │
    ├── Redis Pub/Sub Adapter  ← horizontal scaling readiness
    └── Event handlers (see §4)
```

**Transport order:** `['polling', 'websocket']`  
Polling-first ensures the WebSocket upgrade handshake succeeds even behind restrictive proxies (e.g. Render's HTTP/2 gateway).

---

## 2. Connection Lifecycle

### Client Side (`SocketProvider.jsx`)

```
accessToken present?
    │ No → disconnect + set socket = null
    │ Yes
    ▼
io(SOCKET_URL, { auth: { token }, transports: ['polling', 'websocket'] })
    │
    ├── on('connect')
    │     ├── socket.recovered? → skip query invalidation (fast-path)
    │     └── !recovered → queryClient.invalidateQueries() (re-sync all data)
    │
    ├── on('disconnect') → mark isReconnecting = true
    ├── on('connect_error') → if auth error → redirect to /login
    └── on('reconnect_attempt') → update socketStore
```

The `SocketProvider` mounts once inside `MainLayout` (authenticated routes only). A single socket instance is shared across all components via `SocketContext`.

### Server Side (`server.js`)

```
io.use(authMiddleware)        ← JWT verification
    │
    ▼
io.on('connection', socket)
    ├── socket.join(`user:${userId}`)   ← personal room
    ├── setupPresenceSocket(socket, io)
    ├── mediasoupHandlers(socket, io)
    ├── setupChatSocket(io, socket)
    ├── startMessageQueueWorker(io)
    ├── registerThreadHandlers(io, socket)
    ├── whiteboardHandlers(socket, io)
    ├── callSocket(socket, io)
    ├── room.socket(socket, io)
    ├── registerTranscriptHandlers(io, socket)
    └── workspaceSocket(socket, io)
```

---

## 3. State Recovery

Socket.IO's built-in **Connection State Recovery** is enabled:

```js
connectionStateRecovery: {
  maxDisconnectionDuration: 3 * 60 * 1000,  // 3-minute buffer
  skipMiddlewares: true,                      // fast-path re-auth
}
```

**If recovered (`socket.recovered = true`):**
- Server replays missed events from in-memory buffer
- Client skips `queryClient.invalidateQueries()` — no re-fetch needed
- User sees seamless reconnection

**If NOT recovered (server restarted, >3-min gap):**
- Client calls `queryClient.invalidateQueries()` — all React Query caches invalidated
- All data re-fetched from REST API
- `syncStatus` transitions: `reconnecting` → `resyncing` → `synced`

---

## 4. Socket Handler Inventory

| File | Namespace / Events | Depends On |
|------|--------------------|-----------|
| `presence.socket.js` | `user:online`, `user:offline`, `typing:start`, `typing:stop`, heartbeat | `presence.service`, Redis |
| `chat.socket.js` | `message:send`, `message:edit`, `message:delete`, `message:react`, `read:mark` | `messageBuffer.service`, `notification.service` |
| `thread.socket.js` | `thread:message:send`, `thread:message:edit`, `thread:message:delete` | `thread.service` |
| `mediasoup.handlers.js` | `ms:join-room`, `ms:leave-room`, `ms:create-transport`, `ms:connect-transport`, `ms:produce`, `ms:consume`, `ms:resume-consumer` | `mediasoup/manager.js` |
| `room.socket.js` | `room:join`, `room:leave`, `room:signal`, `room:ice-candidate` | Signaling only (mesh WebRTC) |
| `call.socket.js` | `call:initiate`, `call:accept`, `call:reject`, `call:end` | `call.service`, `notification.service` |
| `whiteboard.handlers.js` | `whiteboard:draw`, `whiteboard:clear`, `whiteboard:state` | In-memory canvas state |
| `transcript.socket.js` | `transcript:chunk`, `transcript:complete`, `transcript:action-items` | `ai.service` (Groq) |
| `workspace.socket.js` | `workspace:join`, `workspace:leave`, `workspace:channel:update` | `workspaceStore` |

---

## 5. Room Topology

Socket.IO **rooms** (not to be confused with WebRTC rooms) are used for targeted event delivery:

| Room Name | Members | Used For |
|-----------|---------|---------|
| `user:{userId}` | Single user (all tabs) | DM notifications, call ringing, personal events |
| `conversation:{id}` | All conversation participants | Chat messages, typing indicators |
| `group:{id}` | All group members | Group messages, member changes |
| `workspace:{id}` | All workspace members | Channel updates, member events |
| `room:{roomCode}` | Active video room participants | WebRTC signaling, room events |
| `whiteboard:{roomId}` | Active whiteboard session | Canvas draw events |

---

## 6. Message Write Pipeline

Chat messages go through a **write buffer** to reduce MongoDB write pressure:

```
socket 'message:send'
    │
    ├── Immediately: broadcast to conversation room (optimistic)
    │
    └── messageBuffer.service → in-memory queue
              │
              │ every 3 seconds (messageBuffer.job)
              ▼
         MongoDB Message collection (batch insert)
              │
              └── notification.service → io.to(`user:${recipient}`) → push notification
```

**Failure mode:** If the server crashes before a 3-second flush, up to 3 seconds of messages can be lost. The graceful shutdown handler flushes the buffer before exit.

---

## 7. Redis Pub/Sub Adapter

The Socket.IO Redis adapter is initialized when `DISABLE_REDIS !== "true"`:

```js
io.adapter(createAdapter(getPubClient(), getSubClient()))
```

This makes `io.to(room).emit()` work across multiple server instances. Currently single-instance on Render, but the infrastructure is scale-ready.

---

## 8. Presence System

Presence is maintained via Redis with TTL-based expiry:

```
socket connect  → presence.service.setOnline(userId) → Redis SET user:presence:{id} TTL=60s
heartbeat ping  → presence.service.refreshTTL(userId)
socket disconnect → presence.service.setOffline(userId) → Redis DEL + broadcast
```

The `lastSeen` middleware updates the MongoDB `User.lastSeen` field on each authenticated HTTP request.
