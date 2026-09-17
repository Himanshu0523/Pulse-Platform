# 🏛️ Domain Boundaries & V1 Scope Specification

This document defines the **frozen product scope for V1** and establishes **strict domain boundaries** for the Pulse Chat Platform to ensure zero architectural degradation and ensure a clean, migration-friendly evolution into V2 microservices / modular monolith.

---

## 🎯 Phase 0.1 — Frozen Product Scope (V1)

The following core features constitute the complete and frozen feature set for the V1 release. No new major feature domains will be introduced during V1 implementation.

### 📋 V1 Feature Matrix

| Feature Area | Included Capabilities in V1 |
| :--- | :--- |
| **Authentication & Identity** | Email/password sign-up, login, JWT token rotation, 2FA (TOTP / Speakeasy), Email verification, Password reset |
| **User Profile & Contacts** | Avatar, bio, custom status, friend requests (send, accept, decline, block), presence status |
| **Workspaces & Channels** | Multi-workspace support, workspace creation/switching, member roles (Owner, Admin, Member, Guest), invitation tokens/links, public/private rooms (channels) |
| **Direct & Group Messaging** | 1:1 Direct Messages, multi-user Group DMs, rich text markdown, link previews |
| **Message Lifecycle** | Send, edit, soft-delete, pin, search, delivery & read receipts, scheduled messages |
| **Reactions & Threads** | Emoji reactions with live count, threaded conversation replies, thread participant tracking |
| **Real-Time Presence & State**| Heartbeat-based online/away/offline presence, typing indicators, live channel participant lists |
| **Notifications** | In-app notification center, unread badges, mention notifications (`@user`, `@channel`) |
| **File & Media Sharing** | Multi-file uploads, image/video/audio preview, Cloudinary & local storage adapters |
| **Voice & Video Calling** | WebRTC 1:1 calling, Mediasoup SFU multi-party video & audio conferencing, screen sharing |
| **Interactive Collaboration**| Collaborative real-time whiteboard canvas, channel polls & interactive voting |
| **Integrations Engine** | Inbound webhook ingestion, GitHub OAuth & webhook parser, Sentry alerts, Email bridge |
| **AI Assistant & Knowledge** | Conversation summaries, smart 3-option replies (Groq / Llama-3), meeting transcripts, action-item extraction, semantic search |
| **Security & Compliance** | AES-256 field encryption, End-to-End Encryption key exchange, GDPR Article 20 ZIP data export, 11-step cascade deletion, immutable audit logs |

---

## 🧩 Phase 0.2 — Domain Ownership & Boundary Model

Each domain owns its specific state, database models, socket events, and service logic. Cross-domain interactions must occur through defined internal interfaces/services rather than direct tight coupling.

```
┌────────────────────────────────────────────────────────────────────────────┐
│                             PULSE PLATFORM                                 │
└────────────────────────────────────────────────────────────────────────────┘
        │
        ├── 🔐 Auth Domain
        │    └── User Identity, Credentials, TOTP 2FA, JWT Tokens, Verification
        │
        ├── 🏢 Workspace Domain
        │    ├── Workspaces & Settings
        │    ├── Members & Role-Based Access Control (RBAC)
        │    └── Workspace Invites & Expiring Tokens
        │
        ├── 💬 Chat Domain
        │    ├── Conversations (Direct & Group)
        │    ├── Rooms (Workspace Channels)
        │    ├── Messages & Scheduled Messages
        │    ├── Message Threads & Participants
        │    ├── Polls & Voting
        │    └── Reactions & Pins
        │
        ├── 📡 Communication & Media Domain
        │    ├── Online / Offline Presence Lifecycle
        │    ├── Typing Indicators
        │    ├── WebRTC Signaling
        │    ├── Mediasoup SFU Audio / Video Transports & Routing
        │    └── Whiteboard Collaboration Stream
        │
        ├── 📁 Files & Storage Domain
        │    ├── Upload Pipelines (Local, Cloudinary, S3-ready)
        │    ├── Media Previews & Transcoding
        │    └── Attachment Metadata & Access Control
        │
        ├── 🔔 Notification Domain
        │    ├── Notification Lifecycle & Delivery
        │    └── Push, In-App Badges & Mention Alerts
        │
        ├── 🔌 Integrations Domain
        │    ├── Integration Catalog Marketplace
        │    ├── User / Workspace Credentials & OAuth
        │    ├── Inbound Webhook Dispatcher
        │    ├── Inbound Email Bridge
        │    └── Connectors: GitHub, Sentry, Generic Webhook
        │
        ├── 🧠 AI & Knowledge Domain
        │    ├── Knowledge Base Articles & Quotas
        │    ├── Vector Search Indexing (Pinecone / Local Fallback)
        │    ├── Conversation Summarization & Smart Replies (Groq)
        │    └── Call Transcripts & Action Item Extraction
        │
        └── 🛡️ Security & Compliance Domain
             ├── Audit Logging (AuditLog)
             ├── Compliance Logs (ComplianceLog)
             ├── GDPR 11-Step Cascade Deletion
             └── Portability Data Export (ZIP)
```

---

## 🔒 V2 Migration Readiness Rules

1. **No Circular Schema References**: Mongoose models in one domain must reference IDs rather than embedding schemas across domains.
2. **Event-Driven Decoupling**: Domain side-effects (e.g. "message sent" -> "notification created") are mediated through Event Handlers / Queues (Bull / Redis) to enable straightforward migration to an event bus (e.g., NATS / Kafka) in V2.
3. **Isolated Configuration**: Service secrets and configurations are domain-namespaced.
