# Private Pulse Platform — Chat Architecture Specification

## Executive Overview

The **Private Pulse Platform Chat Subsystem** is an enterprise-grade, real-time, end-to-end encrypted (E2EE) messaging engine built for high concurrency, zero packet loss, and multi-tenant workspace isolation.

This document outlines the complete system architecture, data models, real-time socket protocols, cryptographic pipelines, asynchronous background worker jobs, and dependent feature integrations.

---

## 1. High-Level System Architecture

```
                    +-------------------------------------------------+
                    |               React Single Page App             |
                    |   (Zustand Stores, SocketContext, E2EE Engine)  |
                    +------------------------+------------------------+
                                             |
                                 REST API & WebSockets
                                (HTTPS / WSS + Auth JWT)
                                             |
                                             v
                    +-------------------------------------------------+
                    |             Node.js / Express Gateway           |
                    |    (Socket.IO v4 + Fast-Path Token Middleware)  |
                    +----+-------------------+-------------------+----+
                         |                   |                   |
                         v                   v                   v
            +--------------------+   +---------------+   +--------------------+
            | Redis Pub/Sub      |   | MongoDB       |   | BullMQ Worker      |
            | (@socket.io/adapter|   | (Messages,    |   | (Link Previews,    |
            | & Session Buffer)  |   | Conversations)|   | Scheduled Messages)|
            +--------------------+   +---------------+   +--------------------+
```

---

## 2. Core Real-Time Socket Gateway & Connection Recovery

### 2.1 Engine Configuration
- **Socket.IO v4 Engine** with `@socket.io/redis-adapter` for multi-node horizontal scaling.
- **Connection State Recovery (3-Minute Buffer)**:
  ```javascript
  connectionStateRecovery: {
    maxDisconnectionDuration: 3 * 60 * 1000, // 3 minutes in-memory buffer
    skipMiddlewares: true                   // Fast-path authentication recovery
  }
  ```
- **Fast-Path Authentication**:
  Upon connection, the gateway validates the JWT access token and assigns `socket.userId` directly from the token claims, performing non-blocking async populates for user details to prevent handshake delays.

### 2.2 Reconnection Lifecycle & Zero Packet Loss
1. **Network Disruption**: Client UI actions are gracefully frozen; `socketStore` sets `syncStatus = 'reconnecting'`. Amber pulsing recovery banner appears.
2. **Instant Reconnection (`socket.recovered === true`)**: Socket.IO automatically replays missed in-memory room packets. UI unfreezes with zero packet loss.
3. **State Resynchronization (`socket.recovered === false`)**: If the disconnection exceeded 3 minutes or the backend node restarted, TanStack Query invalidates active cached queries (`queryClient.invalidateQueries()`) to fetch fresh state from MongoDB.

---

## 3. End-to-End Encryption (E2EE) Architecture

The platform operates under a **Zero-Knowledge Architecture** where plaintext messages are never transmitted over the network or stored on backend databases.

### 3.1 Encryption Flow
1. **Key Generation**: Web Crypto API (`ECDH` / `AES-GCM-256`) generates local key pairs stored securely in browser IndexedDB/LocalStorage.
2. **Message Payload Encryption**:
   - Message text is encrypted using `AES-GCM-256`.
   - The AES payload key is wrapped using the recipient's public key (`ECDH`).
3. **Database Representation (`Message` model)**:
   ```json
   {
     "isEncrypted": true,
     "ciphertext": "...",
     "encryptedKey": "...",
     "iv": "...",
     "keyIv": "...",
     "algorithm": "AES-GCM",
     "senderPublicKeyJwk": { ... }
   }
   ```
4. **Client Decryption**: The recipient decrypts `encryptedKey` using their private key, then decrypts `ciphertext` using the restored `AES-GCM` key and `iv`.

---

## 4. Subsystems & Data Pipeline

### 4.1 Message Threading & Reply Engine
- Messages support nested discussions without polluting the main conversation stream.
- **Root Message Fields**: `replyCount`, `lastReplyAt`, `isThreadReply: false`.
- **Thread Reply Fields**: `threadRoot: <ObjectId>`, `parentMessage: <ObjectId>`, `isThreadReply: true`.
- **DB Indexing**:
  `{ conversation: 1, isThreadReply: 1, lastReplyAt: -1 }` for instant active thread lookups.

### 4.2 Asynchronous Link Preview Worker
1. When a message containing a URL is posted, the backend initializes `preview.status = 'PENDING'`.
2. A BullMQ job is dispatched to `linkPreview.worker.js`.
3. The worker fetches OpenGraph metadata (title, description, image, favicon).
4. On success, `preview.status` updates to `'READY'`, and a `message_updated` socket event broadcasts the rendered preview card to room participants.

### 4.3 Scheduled Messages Subsystem
1. Users can compose messages set for future delivery (`scheduledAt`).
2. Message metadata is pushed to the BullMQ scheduled queue (`scheduledMessage.worker.js`).
3. Upon timer expiry, the worker creates the database record and emits `new_message` to the socket channel.

### 4.4 Disappearing Messages (Self-Destruct)
1. Conversations allow configuring `disappearingMessagesDuration` (`86400` = 24h, `604800` = 7d, `7776000` = 90d).
2. Background cron workers purge expired messages based on `createdAt + disappearingMessagesDuration < Date.now()`.

---

## 5. Dependent Feature Integrations

```
+-----------------------------------------------------------------------+
|                      DEPENDENT FEATURES MAP                           |
+-----------------------------------------------------------------------+
|                                                                       |
|  [ Workspace Isolation ] ---> Scopes Conversations & Channels by ID   |
|                                                                       |
|  [ Voice & Video Calls ] ---> WebRTC Signaling over Chat Socket Rooms |
|                                                                       |
|  [ Notification Hub  ] ---> @Mentions, Pinned Msgs, System Alerts     |
|                                                                       |
|  [ Rich Media Uploads] ---> Multer + S3/Cloud Storage Middleware      |
|                                                                       |
|  [ User Presence     ] ---> Online / Offline / Typing Status Sockets  |
+-----------------------------------------------------------------------+
```

### 5.1 Workspace Isolation & Multi-Tenancy
- Every conversation and message has an explicit `workspace: ObjectId` reference.
- Sockets automatically scope room joining (`workspace:<workspaceId>:conversation:<conversationId>`) to prevent cross-tenant data leaks.

### 5.2 Voice & Video Calling Integrations
- WebRTC Peer-to-Peer and SFU call offer/answer/ICE-candidate signaling flows directly through the Chat Socket Gateway.
- Components (`IncomingCallOverlay`, `OutgoingCallOverlay`, `ActiveCallBanner`) listen to chat socket rooms to present seamless call overlays.

### 5.3 Notification Hub & Direct Mentions
- Message text parsing detects `@username` tags and populates `mentions: [UserId]`.
- Triggers push notifications and updates the `NotificationHub` unread badge counter in real time.

---

## 6. Socket Protocol Event Map

| Event Name | Direction | Payload Description |
| :--- | :--- | :--- |
| `join_conversation` | Client -> Server | `{ conversationId }` |
| `send_message` | Client -> Server | Encrypted payload, attachments, replyTo, threadRoot |
| `new_message` | Server -> Client | Full message object or E2EE payload |
| `typing_start` / `typing_stop` | Bidirectional | `{ conversationId, userId }` |
| `message_read` | Server -> Client | Read receipt update for direct & group conversations |
| `message_updated` | Server -> Client | Edits, link preview ready, reactions, pins |
| `call_signal` | Bidirectional | WebRTC offer, answer, ICE candidates |

---

## 7. Performance & Scalability Guarantees

1. **Sub-50ms Message Delivery Latency**: Powered by Redis Pub/Sub adapter and WebSocket binary/JSON frame transport.
2. **Horizontal Scale-Out**: Stateless Express socket nodes behind an NGINX load balancer with sticky sessions.
3. **Database Query Efficiency**: Compound indexes on `{ conversation: 1, createdAt: -1 }` for paginated infinite scrolling.
