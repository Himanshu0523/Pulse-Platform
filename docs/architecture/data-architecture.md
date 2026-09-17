# Data Architecture

> **Scope:** MongoDB collections, entity relationships, caching strategies (Redis), file storage, and data retention.

---

## 1. Primary Datastore: MongoDB

Pulse uses MongoDB 7 (via Mongoose) as its primary datastore. The schemas are organized into 6 functional domains across 28 models.

```
┌────────────────────────────────────────────────────────────────────────┐
│                          Core Entities & Domains                       │
├───────────────────┬───────────────────┬────────────────────────────────┤
│ Identity & Auth   │ Messaging & Chat  │ Workspaces & Channels          │
│ • User            │ • Conversation    │ • Workspace                    │
│ • Friend          │ • Message         │ • WorkspaceMember              │
│                   │ • ThreadPartic.   │ • WorkspaceInvite              │
│                   │ • ScheduledMsg    │                                │
│                   │ • Poll            │                                │
├───────────────────┼───────────────────┼────────────────────────────────┤
│ Realtime & Calls  │ Integrations      │ AI & Knowledge & Governance    │
│ • Room            │ • Inegr.Catalog   │ • KnowledgeArticle             │
│ • Call            │ • WorkspaceIntegr.│ • KnowledgeQuota               │
│ • File            │ • Integr.Event    │ • SearchQuery                  │
│ • Notification    │ • Integr.Routing  │ • ActionItem / Transcript      │
│                   │ • UserIntegrCred. │ • AuditLog / ComplianceLog     │
│                   │ • ChannelEmail    │                                │
└───────────────────┴───────────────────┴────────────────────────────────┘
```

---

## 2. Entity Relationship Overview

```
User (1) ──── (N) WorkspaceMember (N) ──── (1) Workspace
 │                    │
 │                    └─ (1) Role (owner | admin | moderator | member | guest)
 │
 ├──── (N) Conversation ──── (N) Message ──── (N) Reaction (embedded)
 │                                │
 │                                ├──── (N) ThreadMessage
 │                                └──── (1) Poll
 │
 ├──── (N) Call (callerId / receiverId / roomId)
 ├──── (N) Room (hostId / participantIds)
 └──── (N) Notification (userId / actorId)
```

---

## 3. Detailed Schema Domains

### 1. Identity & Access
- **`User`**: Account identity, password hash (`argon2id`), MFA secrets (`speakeasy`), biometric WebAuthn credentials, online status, preferences, account lockout counters.
- **`Friend`**: Bi-directional friend relationships with status (`pending`, `accepted`, `blocked`).

### 2. Messaging & Conversations
- **`Conversation`**: Supports `DM`, `GROUP_DM`, and workspace `CHANNEL`. Stores participant references, last message snapshot, unread counters.
- **`Message`**: Core message document. Supports text, attachments, E2E encrypted payloads, link preview metadata, soft-delete flags (`isDeleted`), and embedded reactions (`emoji`, `userIds[]`).
- **`ThreadParticipant`**: Tracks read state and notification preferences for comment threads.
- **`ScheduledMessage`**: Outbox for future message deliveries evaluated by `messageQueue.job.js`.
- **`Poll`**: Interactive polls embedded in chat with voting options, user vote records, and expiration timestamps.

### 3. Workspaces & Tenancy
- **`Workspace`**: Multi-tenant container for channels and team data. Custom slug, branding, member limits.
- **`WorkspaceMember`**: Junction table mapping `User` to `Workspace` with explicit roles (`owner`, `admin`, `moderator`, `member`, `guest`) and permissions.
- **`WorkspaceInvite`**: Cryptographic tokens for workspace onboarding with expiration and single/multi-use limits.

### 4. Calls & Media
- **`Room`**: Ad-hoc meeting rooms and personal meeting rooms. Stores room codes, host ID, access passcodes, waiting room flags, and audio/video default toggles.
- **`Call`**: Call logs recording 1:1 or group calls, duration, start/end timestamps, and terminal state (`accepted`, `rejected`, `missed`, `ended`).
- **`File`**: File metadata including original name, mime type, size, storage provider (`cloudinary` vs `cloudflare-r2`), public URL, and uploader ID.

### 5. Integrations & Automation
- **`IntegrationCatalog`**, **`WorkspaceIntegration`**, **`IntegrationRoutingRule`**, **`IntegrationEvent`**, **`UserIntegrationCredential`**, **`ChannelEmailAddress`** (see `integration-architecture.md`).

### 6. AI, Search & Compliance
- **`KnowledgeArticle`**: Workspace documentation and articles with markdown body, tags, view counts, and vector embeddings for semantic search.
- **`KnowledgeQuota`**: Workspace-level usage tracking for AI embeddings and LLM prompts.
- **`ActionItem` & `Transcript`**: Generated during video/audio call recordings via Groq LLM processing.
- **`AuditLog`**: Immutable audit logs capturing administrative actions, workspace permission changes, and security events.
- **`ComplianceLog`**: Regulatory export logs and retention events.

---

## 4. Indexing & Query Optimization

| Collection | Key Indexes | Purpose |
|------------|-------------|---------|
| `messages` | `{ conversationId: 1, createdAt: -1 }` | Fast chat timeline pagination |
| `messages` | `{ scheduledFor: 1, isSent: 1 }` | Scheduled message worker polling |
| `conversations` | `{ participants: 1, updatedAt: -1 }` | User inbox list queries |
| `workspaceMembers` | `{ workspaceId: 1, userId: 1 }` (unique) | Fast tenancy & RBAC membership lookups |
| `rooms` | `{ roomCode: 1 }` (unique) | Instant room lookups |
| `integrationEvents` | `{ provider: 1, externalEventId: 1 }` (unique) | Idempotent webhook deduplication |
| `users` | `{ email: 1 }`, `{ username: 1 }` (unique) | Authentication queries |

---

## 5. Caching & Volatile State (Redis)

Redis is used strictly for low-latency, non-persistent, or volatile state:

1. **User Presence:** `user:presence:{userId}` string key with 60-second TTL refreshed via socket heartbeats.
2. **Socket Adapter:** Redis Pub/Sub channels (`socket.io#*`) enabling cross-process socket messaging.
3. **Write Buffer:** In-memory / Redis list buffering chat messages prior to bulk MongoDB insertion.
4. **BullMQ Queues:** Background job queue persistence for integrations, link previews, and scheduled tasks.
5. **Rate Limiting:** Token bucket tracking for sensitive endpoints (login, register, webhook ingresses).

---

## 6. Storage Strategy (Hybrid CDN)

- **Images & Avatars:** Processed and served via **Cloudinary** (automatic responsive cropping, WebP conversion).
- **Files & Attachments:** Stored in **Cloudflare R2** via S3-compatible API (zero egress fees, encrypted object storage).
