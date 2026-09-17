# Pulse — System Overview

> **Status:** Living document. Updated as the system evolves.  
> **Last revised:** September 2026  
> **Scope:** Full-stack architecture for `private-pulse-platform`

---

## 1. What Pulse Is

Pulse is a real-time collaboration platform combining:

- **Persistent chat** — DMs, group channels, threads, file sharing, polls
- **Live video/audio rooms** — SFU-based conferencing via Mediasoup (WebRTC)
- **Peer rooms** — Mesh WebRTC signaling for small ad-hoc sessions
- **Workspaces** — Multi-tenant channel organization (Slack-model)
- **Knowledge base** — AI-searchable article system per workspace
- **Integrations** — GitHub, Sentry, Email Bridge, Generic Webhooks
- **AI layer** — Smart replies, message summarization, live transcription, action items

---

## 2. Deployment Topology

```
Browser (Vercel CDN)
       │
       │  HTTPS / WSS
       ▼
  Express + Socket.IO  (Render — single instance)
       │
       ├── MongoDB Atlas         (primary datastore)
       ├── Redis Cloud           (pub/sub, session, cache, BullMQ queues)
       ├── Cloudinary            (image/file CDN)
       ├── Cloudflare R2         (large file object store)
       ├── Groq API              (LLM — smart replies, summaries, transcription)
       └── Mediasoup Workers     (in-process C++ SFU, forked per CPU core)
```

| Layer | Technology | Hosted On |
|-------|-----------|-----------|
| Frontend | React 18 + Vite + Tailwind CSS | Vercel |
| Backend API | Express 5 + Node.js 22 | Render |
| Realtime | Socket.IO 4 (ws + polling fallback) | Render |
| WebRTC SFU | Mediasoup 3.20 | In-process on Render |
| Database | MongoDB 7 via Mongoose | MongoDB Atlas |
| Cache / Queue | Redis via ioredis + BullMQ | Redis Cloud |
| Media Storage | Cloudinary (images) + Cloudflare R2 (files) | CDN |
| AI | Groq SDK (Llama 3) | External API |
| Email | Nodemailer (SMTP) | External SMTP |

---

## 3. Frontend Map

```
client/src/
├── main.jsx                   Entry point — React Query + Router + Providers
├── app/                       QueryClient, router definition
├── layouts/
│   ├── AuthLayout.jsx         Unauthenticated shell (no sidebar)
│   ├── MainLayout.jsx         Authenticated shell (sidebar + socket)
│   └── DashboardLayout.jsx    Dashboard-specific shell
├── pages/                     Route-level components (lazy loaded)
│   ├── Home.jsx               Public landing page
│   ├── Login / Register       Auth pages → delegate to features/auth
│   ├── Dashboard.jsx          Activity feed
│   ├── Chat/                  DirectChat, ChatHub
│   ├── Rooms/                 RoomsHub, RoomsLobby
│   ├── Friends/               Friend list + requests
│   ├── Knowledge/             KnowledgeHub
│   ├── Notifications/         NotificationCenter
│   ├── Profile/               User profile editor
│   ├── Settings/              App settings
│   ├── Whiteboard.jsx         Collaborative canvas
│   └── Workspaces/            Workspace management
├── features/                  Domain logic, co-located by feature
│   ├── auth/                  Login, Register forms + auth API
│   ├── chat/                  Message API, chat store, E2E utils
│   ├── calls/                 Call API, Mediasoup hooks, call store
│   ├── rooms/                 Room API, room hooks
│   ├── friends/               Friends API + hooks
│   ├── groups/                Group API + hooks
│   ├── files/                 Upload API + components
│   ├── integrations/          Integration catalog + credential UI
│   ├── notifications/         Notification hooks
│   ├── profile/               Profile update API
│   ├── search/                Search API
│   ├── settings/              Settings API
│   ├── whiteboard/            Canvas sync logic
│   └── workspace/             Workspace CRUD + invite
├── context/
│   ├── SocketProvider.jsx     Single global Socket.IO connection (JWT auth)
│   ├── AuthContext.jsx        Auth state hydration
│   ├── EncryptionContext.jsx  E2E key management
│   ├── ThemeContext.jsx       Dark/light mode
│   ├── WorkspaceContext.jsx   Active workspace
│   └── ToastContext.jsx       Global toast notifications
├── store/                     Zustand stores (client state)
│   ├── authStore.js           JWT tokens, user object
│   ├── callStore.js           Active call state
│   ├── notificationStore.js   Unread counts, notification list
│   ├── presenceStore.js       Online presence map
│   ├── socketStore.js         Socket connection state
│   ├── workspaceStore.js      Active workspace + channels
│   └── themeStore.js          Theme preference
├── services/
│   ├── axios.js               Axios instance (baseURL, interceptors, CSRF, token refresh)
│   └── knowledgeService.js    Knowledge article API calls
└── hooks/                     Shared React hooks
```

---

## 4. Backend Map

```
server/src/
├── server.js                  Process entry — HTTP + Socket.IO + graceful shutdown
├── app.js                     Express app — middleware stack + route mounting
├── config/
│   ├── db.js                  Mongoose connection (with retry)
│   ├── redis.js               ioredis pub/sub clients
│   └── cloudinary.js          Cloudinary SDK init
├── models/                    28 Mongoose schemas (see data-architecture.md)
├── routes/                    33 route files (mounted in app.js)
├── controllers/               Request handlers
├── services/                  41 domain service files
├── middleware/                 23 middleware files
├── socket/                    11 Socket.IO handler files
├── mediasoup/
│   ├── config.js              Codec, transport, simulcast config
│   └── manager.js             Worker pool — create, round-robin, respawn
├── integrations/
│   ├── core/                  Integration pipeline engine
│   ├── github/                GitHub webhook + OAuth adapter
│   ├── sentry/                Sentry alert adapter
│   ├── email/                 Email bridge adapter
│   └── generic/               Custom webhook handler
├── queues/
│   ├── integrationQueues.js   BullMQ queues for integration events
│   └── linkPreview.queue.js   BullMQ queue for URL preview generation
├── workers/
│   └── integrationWorker.js   BullMQ worker — processes integration events
├── jobs/
│   ├── messageQueue.job.js    Scheduled message delivery
│   ├── messageBuffer.job.js   Write-buffer flush to MongoDB
│   └── cleanupUnverifiedUsers.job.js  Cron cleanup of unverified accounts
└── utils/                     JWT, metrics (prom-client), request context
```

---

## 5. Data Flow Summary

```
User Action (Browser)
    │
    ├── HTTP/REST via axios.js → Express route → controller → service → MongoDB
    │
    └── Socket.IO event → socket handler → service → MongoDB
                │
                └── io.to(room).emit() → connected clients
```

---

## 6. Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Realtime transport | Socket.IO (polling → WS upgrade) | Automatic reconnection, state recovery |
| Video SFU | Mediasoup (in-process) | Low latency, no external SFU cost |
| Mesh fallback | WebRTC via `room.socket.js` signaling | Small rooms (≤4) without SFU overhead |
| State management | Zustand (client) | Lightweight, no boilerplate vs Redux |
| Server cache | Redis | Presence, rate limiting, session, queues |
| Auth | JWT (access + refresh) + CSRF double-submit | Stateless, cookie-safe |
| E2E Encryption | Client-side key exchange (EncryptionContext) | Server never sees plaintext |
| AI | Groq (Llama 3) | Low-latency inference, generous free tier |

---

## 7. External Dependencies

| Service | Used For | Failure Impact |
|---------|----------|----------------|
| MongoDB Atlas | All persistent data | **Critical** — server degrades |
| Redis Cloud | Pub/sub, cache, queues | Degraded — no presence/queues |
| Groq API | AI features | Graceful — AI features disabled |
| Cloudinary | Image uploads | Non-critical — uploads fail |
| Cloudflare R2 | File uploads | Non-critical — file uploads fail |
| SMTP | Email verification, digests | Non-critical — email features fail |
