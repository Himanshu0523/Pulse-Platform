# 🚀 MERN Backend – Full API & Socket Reference for Frontend Developers

This document describes every feature, API endpoint, WebSocket event, and data model of the production‑ready backend. **All features are already implemented and tested** – you only need to build the frontend.

---

## 🧩 System Overview

- **Authentication**: JWT access tokens (short‑lived) + `httpOnly` refresh tokens (rotated on each use with lineage family tracking). Passkeys/WebAuthn FIDO2 support. Argon2id password hashing + HIBP breach check.
- **Enterprise Integrations**: Pluggable adapter framework (GitHub PRs, Issues, Pushes, Reviews, Deployments), HMAC SHA-256 webhook ingress, BullMQ async queues, AES-256-GCM encrypted credentials, repository-to-channel routing rules.
- **Real‑time Communication**: Socket.IO with authentication (send the access token in the handshake).
- **File Storage**: MongoDB GridFS for large files, with streaming uploads and progress feedback.
- **Media**: mediasoup SFU for multi‑party audio/video calls (scalable, low‑latency).
- **Caching & Presence**: Redis used for BullMQ queues, rate limiting, presence (online/offline), and performance caching.
- **Monitoring**: Structured logging (Winston), health checks, Prometheus metrics, and request IDs.

**Base URL**: `https://private-pulse-platform-backend.onrender.com/api` (or `http://localhost:5000/api` locally)
**WebSocket URL**: `wss://private-pulse-platform-backend.onrender.com` (or `ws://localhost:5000`)

---

## 🔐 Authentication Flow

1. **Register** `POST /auth/register`
   Body: `{ email, password, displayName }`
   Response: `{ success, accessToken, user: { id, email, displayName } }`
   → Checked against HIBP pwned database and hashed using Argon2id. A refresh token is set as an `httpOnly` cookie.
2. **Login** `POST /auth/login`
   Body: `{ email, password }`
   Response: same as register. Evaluates device fingerprint anomaly score.
3. **Refresh Token** `POST /auth/refresh-token` (uses cookie)
   Response: `{ success, accessToken }`
   → Old refresh token is rotated, belonging to a token family lineage. Token reuse automatically purges session family.
4. **Passkeys / WebAuthn**:
   - `GET /auth/passkeys/register-options` & `POST /auth/passkeys/register-verify`
   - `GET /auth/passkeys/login-options` & `POST /auth/passkeys/login-verify`
5. **Signout Everywhere** `POST /auth/signout-everywhere`
   Revokes all token families for current user.
6. **Get Current User**
   `GET /users/me` (requires Bearer token)

All subsequent requests must include the **access token** in the `Authorization` header:
`Authorization: Bearer <accessToken>`

---

## 🔌 Enterprise Integration Platform (`/integrations` & `/webhooks`)

| Method | Endpoint | Description | Notes |
|---|---|---|---|
| `GET` | `/integrations/catalog` | List catalog integrations | GitHub, Sentry, etc. |
| `GET` | `/integrations/catalog/:key` | Get integration specification | Integration metadata & capabilities |
| `POST` | `/integrations/workspace` | Connect integration to workspace | `{ workspaceId, integrationKey, config }` |
| `GET` | `/integrations/workspace` | List active workspace integrations | Filter by workspaceId |
| `DELETE` | `/integrations/workspace/:id` | Disconnect workspace integration | - |
| `POST` | `/integrations/credentials` | Save encrypted token/key | AES-256-GCM cipher encryption |
| `GET` | `/integrations/credentials` | List saved user credentials | Masked credentials |
| `POST` | `/integrations/routing` | Create routing rule | `{ repository, eventType, targetChannelId }` |
| `GET` | `/integrations/routing` | List workspace routing rules | - |
| `PUT` | `/integrations/routing/:id` | Update routing rule | - |
| `DELETE` | `/integrations/routing/:id` | Delete routing rule | - |
| `GET` | `/integrations/events` | Query event audit log | Query by workspaceId, status, provider |
| `POST` | `/webhooks/github` | Ingest GitHub webhook | HMAC SHA-256 `X-Hub-Signature-256` verified |


### 👥 Friend System (`/friends`)

| Method     | Endpoint              | Description                    | Request Body        |
| ---------- | --------------------- | ------------------------------ | ------------------- |
| `POST`   | `/friends/request`  | Send a friend request          | `{ recipientId }` |
| `POST`   | `/friends/accept`   | Accept a pending request       | `{ requestId }`   |
| `POST`   | `/friends/reject`   | Reject a request               | `{ requestId }`   |
| `POST`   | `/friends/cancel`   | Cancel a request you sent      | `{ requestId }`   |
| `DELETE` | `/friends/remove`   | Remove a friend                | `{ friendId }`    |
| `POST`   | `/friends/block`    | Block a user                   | `{ userId }`      |
| `GET`    | `/friends`          | List all accepted friends      | –                  |
| `GET`    | `/friends/requests` | List incoming pending requests | –                  |

---

### 💬 Conversations & Messages

#### Conversations (`/conversations`)

| Method   | Endpoint           | Description                                | Request Body   | Response              |
| -------- | ------------------ | ------------------------------------------ | -------------- | --------------------- |
| `GET`  | `/conversations` | List all conversations (with unread count) | –             | `{ conversations }` |
| `POST` | `/conversations` | Get or create a 1‑to‑1 conversation      | `{ userId }` | `{ conversation }`  |

#### Messages (`/messages`)

| Method     | Endpoint                                   | Description                    | Request Body / Query                      | Response                     |
| ---------- | ------------------------------------------ | ------------------------------ | ----------------------------------------- | ---------------------------- |
| `GET`    | `/messages/:conversationId?page=&limit=` | Get paginated messages         | query params                              | `{ messages, pagination }` |
| `POST`   | `/messages`                              | Send a message (HTTP fallback) | `{ conversationId, content, replyTo? }` | `{ message }`              |
| `PUT`    | `/messages/edit`                         | Edit a message                 | `{ messageId, content }`                | `{ message }`              |
| `DELETE` | `/messages/:messageId`                   | Soft‑delete a message         | –                                        | `{ success }`              |
| `PATCH`  | `/messages/reaction`                     | Toggle a reaction              | `{ messageId, emoji }`                  | `{ reactions }`            |

---

### 👥 Group Chats (`/groups`)

| Method     | Endpoint                                    | Description                     | Request Body                              | Response             |
| ---------- | ------------------------------------------- | ------------------------------- | ----------------------------------------- | -------------------- |
| `POST`   | `/groups`                                 | Create a group                  | `{ groupName, participants: [userId] }` | `{ conversation }` |
| `PUT`    | `/groups/:conversationId/rename`          | Rename group (admin only)       | `{ groupName }`                         | `{ groupName }`    |
| `PUT`    | `/groups/:conversationId/avatar`          | Upload group avatar (multipart) | `avatar` file                           | `{ groupAvatar }`  |
| `POST`   | `/groups/:conversationId/invite`          | Add a member (admin only)       | `{ userId }`                            | `{ success }`      |
| `DELETE` | `/groups/:conversationId/members/:userId` | Kick member (admin only)        | –                                        | `{ success }`      |
| `POST`   | `/groups/:conversationId/leave`           | Leave the group (self)          | –                                        | `{ success }`      |
| `PUT`    | `/groups/:conversationId/role/:userId`    | Promote/demote admin            | `{ action: 'promote'\|'demote' }`        | `{ success }`      |
| `GET`    | `/groups/:conversationId/members`         | List members with admin flags   | –                                        | `{ members }`      |

---

### 📎 File Sharing (`/files`)

| Method     | Endpoint                                    | Description                      | Notes                                    |
| ---------- | ------------------------------------------- | -------------------------------- | ---------------------------------------- |
| `POST`   | `/files/upload?conversationId=&socketId=` | Upload a file (multipart)        | Progress emits via socket to`socketId` |
| `GET`    | `/files/:fileId/download`                 | Download the file                | Sets`Content‑Disposition: attachment` |
| `GET`    | `/files/:fileId/preview`                  | Preview the file (inline)        | For images, PDFs, etc.                   |
| `DELETE` | `/files/:fileId`                          | Delete file (only uploader)      | –                                       |
| `GET`    | `/files/conversation/:conversationId`     | List all files in a conversation | –                                       |

---

### 📞 Call History (`/calls`)

| Method  | Endpoint                | Description                            |
| ------- | ----------------------- | -------------------------------------- |
| `GET` | `/calls?page=&limit=` | List call history for the current user |
| `GET` | `/calls/:callId`      | Get details of a specific call         |

---

### 🔔 Notifications (`/notifications`)

| Method    | Endpoint                                | Description                       |
| --------- | --------------------------------------- | --------------------------------- |
| `GET`   | `/notifications?page=&limit=`         | Get all notifications             |
| `PATCH` | `/notifications/:notificationId/read` | Mark one as read                  |
| `PATCH` | `/notifications/mark-all-read`        | Mark all as read                  |
| `GET`   | `/notifications/unread-count`         | Get count of unread notifications |

---

### 📍 Presence (`/presence`)

| Method   | Endpoint              | Description                    | Request Body             |
| -------- | --------------------- | ------------------------------ | ------------------------ |
| `GET`  | `/presence/:userId` | Get one user’s presence       | –                       |
| `POST` | `/presence/friends` | Get presence of multiple users | `{ friendIds: [...] }` |

---

## 🖥️ Socket.IO Events (Real‑time)

Connect using:

```javascript
const socket = io(URL, { auth: { token: accessToken } });
```

After connection, you are automatically joined to a personal room (`user:${userId}`) so you can receive notifications and calls.

### 📨 Chat & Messaging

| Client → Server     | Payload                                   | Description                                    |
| -------------------- | ----------------------------------------- | ---------------------------------------------- |
| `join-chat`        | `{ conversationId }`                    | Join a conversation room (to receive messages) |
| `leave-chat`       | `{ conversationId }`                    | Leave the room                                 |
| `typing`           | `{ conversationId }`                    | User is typing                                 |
| `stopTyping`       | `{ conversationId }`                    | User stopped typing                            |
| `sendMessage`      | `{ conversationId, content, replyTo? }` | Send a new message                             |
| `messageDelivered` | `{ messageId }`                         | Acknowledge delivery (recipient received)      |
| `messageSeen`      | `{ messageId }`                         | Acknowledge read (recipient opened)            |

| Server → Client    | Payload                              | Description                       |
| ------------------- | ------------------------------------ | --------------------------------- |
| `receiveMessage`  | `message` object                   | New message broadcast to the room |
| `messageSent`     | `{ messageId, status }`            | Confirmation to the sender        |
| `messageEdited`   | `{ messageId, content, editedAt }` | Message was edited                |
| `messageDeleted`  | `{ messageId }`                    | Message was deleted               |
| `reactionUpdated` | `{ messageId, reactions }`         | Reactions changed                 |
| `typing`          | `{ userId, conversationId }`       | Remote user is typing             |
| `stopTyping`      | `{ userId, conversationId }`       | Remote user stopped               |

### 👥 Group Chat Events (server → client)

| Event                  | Payload                                    | Description        |
| ---------------------- | ------------------------------------------ | ------------------ |
| `groupCreated`       | `{ conversation }`                       | New group created  |
| `groupRenamed`       | `{ conversationId, groupName, oldName }` | Group renamed      |
| `groupAvatarUpdated` | `{ conversationId, groupAvatar }`        | Avatar updated     |
| `memberAdded`        | `{ conversationId, member }`             | New member added   |
| `memberKicked`       | `{ conversationId, userId }`             | Member removed     |
| `memberLeft`         | `{ conversationId, userId }`             | Member left        |
| `adminRoleUpdated`   | `{ conversationId, userId, isAdmin }`    | Admin role changed |
| `mention`            | `{ conversationId, message, sender }`    | User was mentioned |

### 📎 File Sharing (server → client)

| Event                    | Payload                 | Description                                          |
| ------------------------ | ----------------------- | ---------------------------------------------------- |
| `file-upload-progress` | `{ uploaded, done? }` | Emitted to the socketId provided during upload       |
| `new-file`             | `{ file }`            | Broadcast to conversation room when upload completes |
| `file-deleted`         | `{ fileId }`          | Broadcast when a file is deleted                     |

### 📍 Presence (client → server)

| Event               | Payload                   | Description                                                |
| ------------------- | ------------------------- | ---------------------------------------------------------- |
| `presence:update` | `{ status, metadata? }` | Manually set status (`online`, `away`, `busy`, etc.) |
| `presence:typing` | `{ conversationId }`    | Toggle typing status (auto‑reverts after 3s)              |

### 📞 WebRTC / mediasoup Signaling

All call media is handled via **mediasoup** (SFU). The signaling flow:

1. **Join a call room**`joinMediasoupRoom` → server responds with `routerRtpCapabilities`.
2. **Create a `Device`** on the client using `mediasoup-client` and load the capabilities.
3. **Create WebRTC transports**`createWebRtcTransport` → returns `{ id, iceParameters, iceCandidates, dtlsParameters }`.Create both send and receive transports.
4. **Connect transports**`connectWebRtcTransport` with `{ transportId, dtlsParameters }`.
5. **Produce media**`produce` with `{ transportId, kind, rtpParameters }` → server returns `producerId`.
6. **Consume remote producers** when you receive `newProducer` event.
   `consume` with `{ producerId, transportId }` → returns consumer parameters.

| Client → Server           | Payload                                  | Description               |
| -------------------------- | ---------------------------------------- | ------------------------- |
| `joinMediasoupRoom`      | `{ roomId }`                           | Join a call room          |
| `createWebRtcTransport`  | `{}`                                   | Create a transport        |
| `connectWebRtcTransport` | `{ transportId, dtlsParameters }`      | Connect DTLS              |
| `produce`                | `{ transportId, kind, rtpParameters }` | Produce a track           |
| `consume`                | `{ producerId, transportId }`          | Consume a remote producer |
| `resumeConsumer`         | `{ consumerId }`                       | Resume consumption        |
| `closeProducer`          | `{ producerId }`                       | Stop producing            |

| Server → Client    | Payload                            | Description                            |
| ------------------- | ---------------------------------- | -------------------------------------- |
| `newProducer`     | `{ producerId, kind, socketId }` | A remote participant started producing |
| `participantLeft` | `{ socketId }`                   | A participant left the call            |

### 📞 Call Management (legacy HTTP/Socket for history)

These events manage the call metadata (history, notifications) alongside mediasoup.

| Client → Server | Payload                                     | Description             |
| ---------------- | ------------------------------------------- | ----------------------- |
| `call:start`   | `{ conversationId, type, targetUserId? }` | Initiate a call         |
| `call:accept`  | `{ callId }`                              | Accept an incoming call |
| `call:reject`  | `{ callId }`                              | Reject a call           |
| `call:end`     | `{ callId }`                              | End an active call      |

| Server → Client  | Payload                                    | Description                |
| ----------------- | ------------------------------------------ | -------------------------- |
| `call:created`  | `{ callId }`                             | Sent to initiator          |
| `call:incoming` | `{ callId, from, conversationId, type }` | Sent to callee(s)          |
| `call:accepted` | `{ callId, answeredBy }`                 | Sent to all participants   |
| `call:rejected` | `{ callId, rejectedBy }`                 | Sent to initiator          |
| `call:missed`   | `{ callId }`                             | Sent after timeout         |
| `call:ended`    | `{ callId, duration }`                   | Sent to all when call ends |

### ✏️ Whiteboard (server ↔ client)

| Event (client → server)        | Payload                | Description                                            |
| ------------------------------- | ---------------------- | ------------------------------------------------------ |
| `join-whiteboard`             | `{ roomId }`         | Join a whiteboard; server responds with`{ history }` |
| `draw`                        | `{ roomId, action }` | Send a drawing action                                  |
| `undo` / `redo` / `clear` | `{ roomId }`         | Sync operations                                        |

| Event (server → client)        | Payload               | Description              |
| ------------------------------- | --------------------- | ------------------------ |
| `draw`                        | `{ action, index }` | Broadcast drawing action |
| `undo` / `redo` / `clear` | `{}`                | Sync state               |

### 🔔 Notifications (server → client)

| Event                | Payload                 | Description                            |
| -------------------- | ----------------------- | -------------------------------------- |
| `new-notification` | `notification` object | Sent to the recipient’s personal room |

---

## 🗃️ Data Models (simplified)

**User**
`{ _id, email, displayName, avatar, bio, status, lastSeen, settings: { notifications, privacy, theme } }`

**Conversation**
`{ _id, participants: [User], isGroup, groupName, groupAvatar, lastMessage, lastMessageAt, unreadCount }`

**Message**
`{ _id, sender: User, content, type, status, createdAt, reactions, isDeleted, editHistory, replyTo }`

**Friend**
`{ _id, requester, recipient, status: 'pending'|'accepted'|'rejected'|'blocked' }`

**Notification**
`{ _id, recipient, type, from: User, referenceId, conversation, payload, read, createdAt }`

**Call**
`{ _id, participants: [User], type: 'audio'|'video', status: 'pending'|'accepted'|'rejected'|'missed'|'ended', startedAt, endedAt, duration, initiator, conversation }`

---

## 🚀 Frontend Implementation Checklist

- [ ] **Auth**: Store access token in memory/localStorage; include in `Authorization` header. Handle token refresh on 401.
- [ ] **Socket**: Connect with `auth: { token }`. Handle reconnections and token updates.
- [ ] **State management**: Use Redux/Context to keep users, conversations, messages, notifications.
- [ ] **File upload**: Use `FormData`; provide socket ID to receive progress.
- [ ] **Calls**: Use `mediasoup-client`; follow the signaling flow described above.
- [ ] **Whiteboard**: Implement drawing tools and sync via socket.
- [ ] **Presence**: Fetch initial status via REST; update via socket events (you can emit `presence:update` when user changes status).
- [ ] **Notifications**: Show in‑app alerts and update badge count.
- [ ] **Pagination**: Use `page` and `limit` on all list endpoints.

---

## 🌐 Deployment Notes

- The backend uses environment variables (see `.env.example`).
- CORS is configured to allow your frontend origin.
- All static files (avatars, group avatars, uploaded files) are served under `/uploads`.
- Health checks: `/health`, `/liveness`, `/readiness`.
- Metrics: `/metrics` (Prometheus) and `/status` (monitoring dashboard).

---

## 🔗 URL Extraction & Link Preview Engine Pipeline

The backend includes an asynchronous link preview parsing system powered by BullMQ background workers and Redis caching.

```
                          USER SENDS MESSAGE
                                  │
                                  ▼
                    ┌───────────────────────────┐
                    │ Message Validation Layer  │
                    └─────────────┬─────────────┘
                                  │
                                  ▼
                    ┌───────────────────────────┐
                    │  URL Extraction Engine    │
                    │ (Regex / Domain Parser)   │
                    └─────────────┬─────────────┘
                                  │
                  ┌───────────────┴───────────────┐
                  ▼                               ▼
          [ Standard Message ]           [ Async Job Pushed ]
          Pushed to Database              to BullMQ Queue
                  │                               │
                  ▼                               ▼
          Socket Broadcast             ┌─────────────────────┐
         `message:created`             │   Preview Worker    │
                  │                    └──────────┬──────────┘
                  ▼                               │
          Rendered instantly                      ▼
          in Chat Timeline             ┌─────────────────────┐
                                       │ Platform Detector   │
                                       │ (GitHub/YouTube/etc)│
                                       └──────────┬──────────┘
                                                  │
                                                  ▼
                                       ┌─────────────────────┐
                                       │ OpenGraph Scraper / │
                                       │   Redis Cache       │
                                       └──────────┬──────────┘
                                                  │
                                                  ▼
                                       ┌─────────────────────┐
                                       │ Socket Broadcast    │
                                       │ `message:preview`   │
                                       └──────────┬──────────┘
                                                  │
                                                  ▼
                                       ┌─────────────────────┐
                                       │ React LinkPreview   │
                                       │    Component        │
                                       └──────────┬──────────┘
                                                  │
                                                  ▼
                                       ┌─────────────────────┐
                                       │ Rich Preview Card   │
                                       │ Appears in Message  │
                                       └──────────┬──────────┘
```

### Components & Flow
1. **Message Validation Layer**: Validates user payload schema and authentication token.
2. **URL Extraction Engine**: Scans incoming text using domain regex patterns to detect valid Web/HTTPS links.
3. **BullMQ Queue & Preview Worker**: Offloads HTTP fetching & scraping off the main request thread into `linkPreview.worker.js`.
4. **Platform Detector**: Identifies specialized platform links (GitHub API, YouTube OEmbed, Figma OEmbed, Jira APIs) and falls back to OpenGraph scrapers (`cheerio` / `html-parser`).
5. **Redis Cache**: Caches URL metadata (`link:preview:<md5_url>`) with TTL (24h) to eliminate redundant external requests.
6. **Real-time Socket Event (`message:preview`)**: Pushes updated preview metadata directly to conversation subscribers.

---

## 🔐 End-to-End Encryption (E2EE) Server Integration

- **Payload Envelope**: The backend stores and forwards ciphertext without inspecting or modifying payload contents:
  ```json
  {
    "isEncrypted": true,
    "ciphertext": "base64_encoded_string...",
    "nonce": "base64_nonce...",
    "keyId": "device_key_fingerprint"
  }
  ```
- **Zero-Knowledge Architecture**: The server only acts as a transport and relay layer; private keys never leave user client devices.

---

