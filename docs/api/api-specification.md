# 🌐 REST API & Real-Time Protocol Specification (Phases 4, 5, 6, 7, 8)

This specification defines the standard API contract, standardized error architecture, workspace management, conversation lifecycle, cursor-based pagination, Socket.IO real-time event taxonomy, and Redis ephemeral state protocols for **Pulse Chat Platform**.

---

## 🏗️ Phase 4 — Backend Core Architecture

### Step 4.1 — Express Application Layer Separation
To facilitate eventual domain-by-domain migration (e.g. into Go microservices), the codebase strictly separates responsibilities:
```
server.js (HTTP / WebSockets / Mediasoup Process Lifecycle)
   └── app.js (Express Application Pipeline & Middleware)
        ├── routes/ (HTTP Route Path Definitions)
        ├── middleware/ (Auth, Workspace, RateLimit, Error Handling)
        ├── validators/ (Request Payload Schema Validation)
        ├── controllers/ (HTTP Request/Response Enveloping Only)
        ├── services/ (Pure Domain Business Logic)
        └── models/ (Database Entities & Data Access Layer)
```

### Step 4.2 — Standard API Request/Response Contract
Every API operation flows strictly through the following pipeline:
```
Client Request
      ↓
Authentication Middleware (JWT / 2FA verification)
      ↓
Validation Middleware (Body, Query, Params schema check)
      ↓
Authorization Middleware (RBAC & Workspace Tenant Isolation)
      ↓
Controller (Unwraps input, invokes Service)
      ↓
Service (Domain Business Logic execution)
      ↓
Repository / Mongoose Model (Database persistence)
      ↓
Standardized JSON Response Envelope
```

#### Standard Success Response Envelope
```json
{
  "success": true,
  "data": { ... },
  "message": "Operation completed successfully",
  "meta": {
    "requestId": "req_8f9a2b1c4d",
    "timestamp": "2026-09-12T11:30:00Z"
  }
}
```

#### Standard Paginated Response Envelope (Cursor-Based)
```json
{
  "success": true,
  "data": [ ... ],
  "pagination": {
    "limit": 50,
    "hasMore": true,
    "nextCursor": "64f1a2b3c4d5e6f7a8b9c0d1",
    "prevCursor": "64f1a2b3c4d5e6f7a8b9c0e5"
  },
  "meta": {
    "requestId": "req_8f9a2b1c4d",
    "timestamp": "2026-09-12T11:30:00Z"
  }
}
```

### Step 4.3 — Standardized Error Architecture

All error responses return a uniform error envelope with appropriate HTTP status codes:
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": [
      { "field": "email", "issue": "Invalid email address format" }
    ]
  },
  "meta": {
    "requestId": "req_8f9a2b1c4d",
    "timestamp": "2026-09-12T11:30:00Z"
  }
}
```

| HTTP Status | Error Code | Description |
| :--- | :--- | :--- |
| **`400 Bad Request`** | `VALIDATION_ERROR` | Schema validation failure on body, query, or params |
| **`401 Unauthorized`** | `AUTHENTICATION_ERROR` | Missing, invalid, or expired JWT / session |
| **`403 Forbidden`** | `AUTHORIZATION_ERROR` | Insufficient permissions or workspace mismatch |
| **`404 Not Found`** | `RESOURCE_NOT_FOUND` | Target entity (workspace, room, message) does not exist |
| **`409 Conflict`** | `RESOURCE_CONFLICT` | Duplicate unique key (email, workspace slug, room name) |
| **`422 Unprocessable`** | `BUSINESS_RULE_ERROR` | Request syntax is valid but violates domain invariant |
| **`429 Too Many Req`** | `RATE_LIMIT_EXCEEDED` | Exceeded rate limit thresholds in Redis |
| **`500 Internal Error`** | `INTERNAL_SERVER_ERROR` | Unhandled runtime exception |

### Step 4.4 — Comprehensive Validation Matrix

| Payload Target | Validation Rules |
| :--- | :--- |
| **Request Body** | Joi/Express-validator schema checks (types, lengths, regex, required fields) |
| **Query Parameters** | Sanitization and numeric boundaries on `limit`, `cursor`, filters |
| **Route Params** | MongoDB `ObjectId` regex check (`/^[0-9a-fA-F]{24}$/`) |
| **HTTP Headers** | `x-workspace-id`, CSRF tokens, HMAC webhook signatures |
| **Uploaded Files** | MIME type whitelisting, magic byte checks, file size cap enforcement (10MB) |
| **WebSocket Events** | Payload structure & type validation before processing |
| **Webhook Payloads** | Inbound provider verification (GitHub SHA-256 HMAC, Sentry auth) |
| **AI Prompt Requests**| Content length caps & prompt injection character sanitization |

---

## 🏢 Phase 5 — Workspace System API Reference

### Step 5.1 — Workspace CRUD
- `POST /api/workspaces`: Create new workspace (creator becomes `owner`).
- `GET /api/workspaces`: Retrieve user's joined workspaces.
- `GET /api/workspaces/:workspaceId`: Retrieve workspace details and metadata.
- `PUT /api/workspaces/:workspaceId`: Update workspace name, logo, settings (`admin`+).
- `POST /api/workspaces/:workspaceId/archive`: Archive workspace (read-only mode).
- `DELETE /api/workspaces/:workspaceId`: Permanently delete workspace (`owner` only).

### Step 5.2 — Member Management
- `POST /api/workspaces/:workspaceId/invites`: Generate expiring invite link or send email invite.
- `POST /api/workspaces/join/:token`: Accept invitation and join workspace.
- `POST /api/workspaces/invites/:token/reject`: Decline invite.
- `GET /api/workspaces/:workspaceId/members`: List workspace members with roles & statuses.
- `PUT /api/workspaces/:workspaceId/members/:userId/role`: Change member role (`owner` | `admin` | `moderator` | `member` | `guest`).
- `DELETE /api/workspaces/:workspaceId/members/:userId`: Remove member from workspace.

### Step 5.3 — Rooms & Channels
- `POST /api/workspaces/:workspaceId/rooms`: Create channel (text/audio/video, public/private).
- `GET /api/workspaces/:workspaceId/rooms`: List available channels for member.
- `PUT /api/workspaces/:workspaceId/rooms/:roomId`: Rename channel or update topic/description.
- `POST /api/workspaces/:workspaceId/rooms/:roomId/archive`: Archive channel.
- `DELETE /api/workspaces/:workspaceId/rooms/:roomId`: Delete channel.
- `POST /api/workspaces/:workspaceId/rooms/:roomId/members`: Manage private channel member access.

---

## 💬 Phase 6 — Chat System API & Pagination

### Step 6.1 — Conversation Lifecycle
- `POST /api/conversations`: Initiate a new 1:1 Direct Message or Group DM.
- `GET /api/conversations`: Fetch active DM/Group DM conversations.
- `POST /api/conversations/:id/participants`: Add user to Group DM.
- `DELETE /api/conversations/:id/participants/:userId`: Remove user / Leave conversation.
- `POST /api/conversations/:id/archive`: Archive conversation.
- `POST /api/conversations/:id/restore`: Restore archived conversation.

### Step 6.2 & 6.3 — Message Lifecycle & Metadata
- `POST /api/messages`: Send message (supports text, attachments, mentions, replyTo, E2EE ciphertext).
- `GET /api/messages/:targetId`: Fetch message stream (channel or conversation).
- `PUT /api/messages/:id`: Edit message content (marks `isEdited: true`, records `editedAt`).
- `DELETE /api/messages/:id`: Soft delete message (`isDeleted: true`, preserves thread integrity).
- `POST /api/messages/:id/reactions`: Add/remove emoji reaction.
- `POST /api/messages/:id/forward`: Forward message to another room/conversation.
- `GET /api/messages/:id/thread`: Fetch thread replies for a root message.

### Step 6.4 — Cursor-Based Pagination
Pulse Chat uses cursor-based pagination for high performance and zero duplicate/missed messages during live stream updates:
- **`limit`**: Number of records to return (Default: `50`, Max: `100`).
- **`before`**: Fetch messages older than `_id` cursor (`_id < before`).
- **`after`**: Fetch messages newer than `_id` cursor (`_id > after`).
- **Query Optimization**: Leverages compound index `{ roomId: 1, _id: -1 }`.

---

## 📡 Phase 7 — Real-Time Socket.IO Protocol

### Step 7.1 — Standardized `noun.action` Event Taxonomy

All real-time events strictly adhere to the `noun.action` naming convention:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        SOCKET.IO EVENT TAXONOMY                        │
├─────────────────────┬──────────────────────────────────────────────────┤
│ Connection          │ connection, disconnect, reconnect, auth.error   │
│ Presence            │ user.online, user.offline, presence.updated      │
│ Chat Messages       │ message.created, message.updated, message.deleted│
│ Typing Indicators   │ typing.started, typing.stopped                  │
│ Reactions           │ reaction.added, reaction.removed                 │
│ Threads             │ thread.reply.created, thread.updated             │
│ Rooms / Channels    │ room.created, room.updated, room.deleted         │
│ Calls & Media       │ call.initiated, call.answered, call.ended        │
│ Integrations        │ integration.event.received                       │
└─────────────────────┴──────────────────────────────────────────────────┘
```

### Step 7.2 & 7.3 — Socket Authorization & Secure Room Joining

1. **Connection Auth Handshake**:
   - Client sends JWT access token in `auth: { token }`.
   - Socket middleware verifies JWT, loads `socket.userId`, and checks user status.
2. **Room Joining Authorization**:
   - Before a socket can join `workspace:<id>`, `room:<id>`, or `conversation:<id>`, the server queries `WorkspaceMember` or `Conversation` to ensure the user is an active, non-blocked participant.
   - Unauthorized join attempts are rejected with `auth.error`.

---

## ⚡ Phase 8 — Ephemeral State & Redis Architecture

High-frequency real-time operations bypass MongoDB to maintain sub-millisecond responsiveness:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        REDIS DATA ARCHITECTURE                         │
├──────────────────────┬─────────────────────────────────────────────────┤
│ User Presence        │ Hash: `presence:<userId>` (status, last_seen)   │
│ Ephemeral Typing     │ Key: `typing:<roomId>:<userId>` (TTL: 4s)       │
│ Token Families       │ Key: `refresh:<familyId>` (TTL: 7d)             │
│ Rate Limit Buckets   │ Key: `ratelimit:<action>:<ip/userId>` (Sliding) │
│ Distributed Locks    │ Key: `lock:<resourceId>` (Redlock TTL: 5s)      │
│ Socket.io Adapter    │ Pub/Sub: Multi-node event broadcasting         │
└──────────────────────┴─────────────────────────────────────────────────┘
```

### Step 8.1 — Presence Tracking
- Statuses: `online`, `offline`, `away`, `busy`.
- Heartbeat interval: Every 30 seconds.
- Disconnect: Automatic 60-second grace period before broadcasting `user.offline`.

### Step 8.2 — Typing Indicators
- Ephemeral Redis key with a 4-second TTL.
- Broadcast via Redis Pub/Sub directly to room sockets.
- Zero persistence in MongoDB.

### Step 8.3 — Rate Limiting Matrix (Redis Sliding Window)

| Action | Limit Threshold | Window |
| :--- | :--- | :--- |
| **User Login** | 5 attempts | 15 Minutes |
| **Registration** | 3 accounts | 1 Hour |
| **Password Reset** | 3 requests | 1 Hour |
| **Message Sending** | 30 messages | 10 Seconds |
| **File Uploads** | 10 files | 1 Minute |
| **Search Queries** | 20 queries | 1 Minute |
| **AI Assistant** | 10 requests | 1 Minute |
| **Integration Webhooks** | 120 requests | 1 Minute |
