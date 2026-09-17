# 📁 Pulse Chat Platform — Complete Folder Structure & Architecture

A comprehensive directory breakdown and module reference for the **Pulse Chat Platform** fullstack application.

---

## 🏗️ High-Level Project Overview

```
Pulse-chat-app/
├── 📄 package.json                  # Root workspace package configuration
├── 📄 docker-compose.yml            # Container orchestration (Redis, MongoDB, Services)
├── 📄 Dockerfile                    # Default root backend container image definition
├── 📄 README.md                     # Main repository overview & quickstart guide
├── 📂 docs/                         # Centralized documentation hub
│   ├── 📄 README.md                 # Master Documentation Sitemap & Navigation
│   ├── 📂 architecture/             # System blueprints, data design & topologies
│   ├── 📂 api/                      # REST specifications, socket events & auth flows
│   ├── 📂 security/                 # RBAC matrix & 73-point security checklist
│   ├── 📂 adr/                      # Architecture Decision Records
│   ├── 📂 operations/               # Deployment runbooks & Git workflow
│   ├── 📂 guides/                   # Developer walkthroughs & folder structure
│   └── 📂 roadmap/                  # Enterprise backlog, migration & market analysis
├── 📂 client/                       # React frontend application (Vite + TailwindCSS)
├── 📂 server/                       # Node.js backend application (Express + Socket.io + Mediasoup)
├── 📂 scripts/                      # Operational & database migration utilities
└── 📂 tests/                        # Fullstack integration & load testing suites
```

---

## 🖥️ 1. Client Application (`/client`)

The frontend is built with **React 18**, **Vite**, **Tailwind CSS**, **Socket.io Client**, and **Mediasoup Client**. It follows a **Feature-Driven Modular Architecture**.

```
client/
├── 📄 index.html                    # Root HTML document
├── 📄 vite.config.js                # Vite configuration and server proxies
├── 📄 tailwind.config.js            # Tailwind theme tokens and custom styling
├── 📄 vercel.json                   # Vercel SPA routing and headers configuration
├── 📄 package.json                  # Frontend dependencies and scripts
└── 📂 src/
    ├── 📄 main.jsx                  # Application entry point & root context providers
    ├── 📄 index.css                 # Global CSS styles and Tailwind directives
    │
    ├── 📂 app/                      # Application initialization and root router wrappers
    │
    ├── 📂 asserts/ / 📂 assets/     # Static assets (images, icons, audio chimes)
    │
    ├── 📂 config/                   # API base URLs, WebSocket endpoints, feature flags
    │
    ├── 📂 constants/                # App-wide constants, enum definitions, action types
    │
    ├── 📂 context/                  # React Context providers for cross-cutting state
    │   ├── 📄 AuthContext.jsx       # Authentication state, login/logout, tokens
    │   ├── 📄 EncryptionContext.jsx # End-to-End Encryption (E2EE) key management
    │   ├── 📄 SocketContext.jsx     # WebSocket instance hook & access
    │   ├── 📄 SocketProvider.jsx    # Real-time Socket.io lifecycle & listener setup
    │   ├── 📄 ThemeContext.jsx      # Light / Dark mode state & theme switcher
    │   ├── 📄 ToastContext.jsx      # Global toast / notification dispatch system
    │   └── 📄 WorkspaceContext.jsx  # Active workspace selection & metadata
    │
    ├── 📂 features/                 # Modular Domain-Driven Feature Packages
    │   ├── 📂 auth/                 # Authentication, 2FA, OTP verification, Password recovery
    │   ├── 📂 calls/                # 1:1 and group audio/video calling (Mediasoup SFU & WebRTC)
    │   │   ├── 📂 api/              # Call session REST API endpoints
    │   │   ├── 📂 components/       # VideoGrid, CallControls, ScreenShare, DeviceSelector
    │   │   ├── 📂 hooks/            # useCall, useMediasoup, useScreenShare
    │   │   └── 📂 store/            # Call state management
    │   ├── 📂 chat/                 # Direct messaging, channels, threads, reactions, polls
    │   │   ├── 📂 api/              # Message and conversation REST API clients
    │   │   ├── 📂 components/       # MessageList, MessageInput, ThreadView, Reactions, Polls
    │   │   ├── 📂 hooks/            # useChat, useMessages, useTypingIndicator
    │   │   ├── 📂 store/            # Chat message caching & conversation state
    │   │   └── 📂 utils/            # Mention parsers, markdown renderers, date formatters
    │   ├── 📂 whiteboard/           # Collaborative real-time canvas drawing feature
    │   ├── 📂 integrations/         # Webhook connectors, GitHub, Sentry, & Email bridge
    │   │   ├── 📂 api/              # Integration APIs
    │   │   ├── 📂 components/       # IntegrationOverview, GitHubEventCard, EmailCard, Modals
    │   │   └── 📂 hooks/            # useIntegrations, useWebhookFeed
    │   ├── 📂 workspace/            # Workspaces, Channels, Roles & Member management
    │   ├── 📂 rooms/                # Audio/Video conference rooms and voice lounges
    │   ├── 📂 files/                # File uploads, cloud media preview, attachment browser
    │   ├── 📂 friends/              # Friend lists, direct requests, incoming/outgoing invites
    │   ├── 📂 notifications/        # User notification bell, read/unread management
    │   ├── 📂 profile/              # User profile, custom status, avatar, security preferences
    │   ├── 📂 search/               # Global search across messages, files, and members
    │   ├── 📂 settings/             # System preferences, audio/video device options
    │   └── 📂 dashboard/            # Workspace dashboard & activity analytics
    │
    ├── 📂 components/               # Shared Reusable UI Components
    │   ├── 📂 ui/                   # Buttons, Modals, Dropdowns, Tooltips, Badges, Inputs
    │   ├── 📂 layout/               # Sidebar, Header, NavigationBar, AppLayout
    │   ├── 📂 common/               # Loading spinners, Empty states, Error boundaries
    │   ├── 📂 feedback/             # Alert banners, Confirm dialogs, Skeletons
    │   └── 📂 Meetings/             # Meeting lobby & quick join components
    │
    ├── 📂 hooks/                    # Reusable custom React hooks (e.g. useDebounce, useMediaQuery)
    ├── 📂 layouts/                  # Base Layout wrappers (AuthLayout, MainAppLayout)
    ├── 📂 lib/                      # Third-party SDK wrappers (Axios client, Mediasoup helpers)
    ├── 📂 pages/                    # High-level route pages (Home, Chat, Dashboard, Auth, etc.)
    ├── 📂 routes/                   # React Router route definitions & ProtectedRoute guards
    ├── 📂 services/                 # Shared REST API service classes
    ├── 📂 store/                    # Global Redux / Zustand stores
    ├── 📂 styles/                   # Modular style sheets & animation keyframes
    ├── 📂 types/                    # Type definitions and data contracts
    └── 📂 utils/                    # Common helper functions (crypto, formatting, time, sanitize)
```

---

## ⚙️ 2. Server Application (`/server`)

The backend is built with **Node.js**, **Express**, **MongoDB (Mongoose)**, **Redis**, **Socket.io**, and **Mediasoup SFU**.

```
server/
├── 📄 package.json                  # Backend dependencies, engines, and start scripts
├── 📄 patch-mediasoup.js            # Mediasoup Windows/Linux build patch
├── 📂 logs/                         # Winston/Morgan application logs
├── 📂 uploads/                      # Uploaded files & temporary file processing
├── 📂 worker/                       # Background task runners (Queues, Cron processors)
└── 📂 src/
    ├── 📄 server.js                 # HTTP server, Mediasoup SFU worker & Socket.io initialization
    ├── 📄 app.js                    # Express configuration, middleware pipeline & route mounts
    │
    ├── 📂 config/                   # Environment variables, DB connection, Redis & Mediasoup configs
    ├── 📂 constants/                # Global enums, socket events, error codes, HTTP status codes
    │
    ├── 📂 models/                   # Mongoose Database Schemas (28 Models)
    │   ├── 📄 User.js               # User accounts, credentials, 2FA, online status
    │   ├── 📄 Workspace.js          # Workspace metadata, owner, settings
    │   ├── 📄 WorkspaceMember.js    # Member roles, channel permissions
    │   ├── 📄 WorkspaceInvite.js    # Workspace invitation tokens & expiring links
    │   ├── 📄 Conversation.js       # 1:1 and Group conversations
    │   ├── 📄 Room.js               # Channel rooms (Text, Audio, Video)
    │   ├── 📄 Message.js            # Message content, attachments, reactions, threads
    │   ├── 📄 MessageSummary.js     # AI-generated message & thread summaries
    │   ├── 📄 ScheduledMessage.js   # Queued messages for future delivery
    │   ├── 📄 ThreadParticipant.js  # Thread subscriber tracking
    │   ├── 📄 File.js               # Uploaded files metadata, mime types, URLs
    │   ├── 📄 Friend.js             # Friend relationships & pending requests
    │   ├── 📄 Call.js               # Call session logs & participant history
    │   ├── 📄 Transcript.js         # Audio/Video meeting speech transcripts
    │   ├── 📄 Poll.js               # Channel/Direct interactive polls & votes
    │   ├── 📄 Notification.js       # System and user notification records
    │   ├── 📄 ChannelEmailAddress.js# Inbound email addresses linked to channels
    │   ├── 📄 IntegrationCatalog.js # Marketplace of available integrations
    │   ├── 📄 WorkspaceIntegration.js # Active integrations per workspace
    │   ├── 📄 IntegrationEvent.js   # Stored incoming webhook events
    │   ├── 📄 IntegrationRoutingRule.js # Event filter & channel routing rules
    │   ├── 📄 UserIntegrationCredential.js # OAuth tokens & API keys
    │   ├── 📄 KnowledgeArticle.js   # Workspace knowledge base documents
    │   ├── 📄 KnowledgeQuota.js     # AI/Knowledge base usage limits
    │   ├── 📄 SearchQuery.js        # Search indexing and query cache
    │   ├── 📄 ActionItem.js         # AI-extracted action items from discussions
    │   ├── 📄 AuditLog.js           # Security & admin action audit trails
    │   └── 📄 ComplianceLog.js      # Data retention & GDPR/HIPAA compliance logs
    │
    ├── 📂 controllers/              # Business Logic Handlers (33 Controllers)
    │   ├── 📄 auth.controller.js            # Registration, login, JWT token issuance
    │   ├── 📄 twoFactor.controller.js       # TOTP 2FA setup & verification
    │   ├── 📄 githubAuth.controller.js      # GitHub OAuth authentication
    │   ├── 📄 message.controller.js         # Send, edit, delete, pin messages
    │   ├── 📄 conversation.controller.js    # Create/fetch direct & group chats
    │   ├── 📄 room.controller.js            # Channel creation, topic, settings
    │   ├── 📄 call.controller.js            # Start/stop calls, get active calls
    │   ├── 📄 file.controller.js            # Multi-part upload, streaming, S3/Local
    │   ├── 📄 friend.controller.js          # Send/accept friend requests, block users
    │   ├── 📄 user.controller.js            # Profile updates, avatar, user lookup
    │   ├── 📄 workspace.controller.js       # Create workspace, invite members
    │   ├── 📄 workspaceIntegration.controller.js # Configure workspace integrations
    │   ├── 📄 webhook.controller.js         # Inbound webhook ingestion & dispatch
    │   ├── 📄 customWebhook.controller.js   # User-defined webhooks management
    │   ├── 📄 emailWebhook.controller.js    # SendGrid/Mailgun inbound email bridge
    │   ├── 📄 integrationActions.controller.js # Trigger third-party actions
    │   ├── 📄 integrationCatalog.controller.js # Browse integration directory
    │   ├── 📄 integrationEvents.controller.js  # Query past webhook payloads
    │   ├── 📄 integrationRouting.controller.js # Configure routing rules to channels
    │   ├── 📄 ai.controller.js              # AI summaries, smart replies, action extraction
    │   ├── 📄 knowledge.controller.js       # Knowledge base CRUD & semantic search
    │   ├── 📄 scheduledMessage.controller.js # Schedule messages
    │   ├── 📄 poll.controller.js            # Create polls and record votes
    │   ├── 📄 search.controller.js          # Elastic / Mongo full-text search
    │   ├── 📄 presence.controller.js        # Online/offline/away status
    │   ├── 📄 thread.controller.js          # Reply threads & participant tracking
    │   ├── 📄 notification.controller.js    # Mark read/fetch notifications
    │   ├── 📄 dataExport.controller.js      # GDPR compliance user data export
    │   ├── 📄 userDeletion.controller.js    # Account deletion & anonymization
    │   ├── 📄 health.controller.js          # Server & DB health check
    │   └── 📄 webrtc.controller.js          # WebRTC ICE servers / credentials
    │
    ├── 📂 routes/                   # Express REST API Routes (32 Route Modules)
    │   ├── 📄 auth.routes.js                # /api/auth
    │   ├── 📄 message.routes.js             # /api/messages
    │   ├── 📄 conversation.routes.js        # /api/conversations
    │   ├── 📄 room.routes.js                # /api/rooms
    │   ├── 📄 workspace.routes.js           # /api/workspaces
    │   ├── 📄 file.routes.js                # /api/files
    │   ├── 📄 friend.routes.js              # /api/friends
    │   ├── 📄 user.routes.js                # /api/users
    │   ├── 📄 call.routes.js                # /api/calls
    │   ├── 📄 webrtc.routes.js              # /api/webrtc
    │   ├── 📄 notification.routes.js        # /api/notifications
    │   ├── 📄 search.routes.js              # /api/search
    │   ├── 📄 ai.routes.js                  # /api/ai
    │   ├── 📄 knowledge.routes.js           # /api/knowledge
    │   ├── 📄 poll.routes.js                # /api/polls
    │   ├── 📄 thread.routes.js              # /api/threads
    │   ├── 📄 scheduledMessage.routes.js    # /api/scheduled-messages
    │   ├── 📄 webhook.routes.js             # /api/webhooks
    │   ├── 📄 customWebhook.routes.js       # /api/custom-webhooks
    │   ├── 📄 emailBridge.routes.js         # /api/email-bridge
    │   ├── 📄 integrationCatalog.routes.js  # /api/integrations/catalog
    │   ├── 📄 workspaceIntegration.routes.js# /api/integrations/workspace
    │   ├── 📄 integrationEvents.routes.js   # /api/integrations/events
    │   ├── 📄 integrationRouting.routes.js  # /api/integrations/routing
    │   ├── 📄 integrationActions.routes.js  # /api/integrations/actions
    │   ├── 📄 compliance.routes.js          # /api/compliance
    │   └── 📄 health.routes.js              # /api/health
    │
    ├── 📂 socket/                   # Real-Time WebSocket Handlers (Socket.io)
    │   ├── 📄 index.js              # Socket server initialization & auth middleware
    │   ├── 📄 chat.socket.js        # Live messaging, typing indicators, reactions
    │   ├── 📄 room.socket.js        # Live channel join/leave & channel updates
    │   ├── 📄 thread.socket.js      # Real-time thread replies & subscriptions
    │   ├── 📄 presence.socket.js    # Heartbeats, online/offline status broadcasting
    │   ├── 📄 call.socket.js        # WebRTC call signaling & ringing
    │   ├── 📄 mediasoup.handlers.js # Mediasoup SFU (Router, Transports, Producers, Consumers)
    │   ├── 📄 whiteboard.handlers.js# Real-time whiteboard drawing sync & strokes
    │   ├── 📄 transcript.socket.js  # Live speech-to-text transcript streaming
    │   └── 📄 workspace.socket.js   # Workspace level events & member updates
    │
    ├── 📂 integrations/             # External Service Connector Implementations
    │   ├── 📂 core/                 # Base integration engine & event dispatcher
    │   ├── 📂 github/               # GitHub webhook parser (PRs, Issues, Commits)
    │   ├── 📂 sentry/               # Sentry error event parser & alerts
    │   ├── 📂 email/                # Inbound/Outbound email bridge service
    │   ├── 📂 generic/              # Custom JSON payload webhook parser
    │   └── 📄 ADDING_A_NEW_INTEGRATION.md # Developer guide for adding connectors
    │
    ├── 📂 mediasoup/                # Mediasoup SFU Engine (Workers, Routers, Audio/Video configs)
    ├── 📂 whiteboard/               # Whiteboard state manager & snapshot cache
    ├── 📂 middleware/               # Auth (JWT), Rate limiters, Role validator, Error handler
    ├── 📂 services/                 # Email (Nodemailer), AI (Gemini/OpenAI), S3/Storage, Token
    ├── 📂 queues/                   # Bull / Redis background job queues (Email, Summary, Webhooks)
    ├── 📂 workers/                  # Queue worker processors
    ├── 📂 validators/               # Request payload validation schemas
    ├── 📂 utils/                    # Crypto helpers, formatters, response helpers, loggers
    ├── 📂 seeds/                    # Database seed scripts for testing & staging
    └── 📂 scripts/                  # Migration & maintenance scripts
```

---

## 📚 3. Documentation & System Specifications (`/docs`)

```
docs/
├── 📄 FOLDER_STRUCTURE.md           # This document
├── 📄 ARCHITECTURE.md               # High-level system architecture overview
├── 📄 API_SPECIFICATION.md          # REST API endpoints & payload schemas
├── 📄 AUTHENTICATION_FLOW.md        # JWT, 2FA, OAuth & refresh token lifecycle
├── 📄 DATABASE_DESIGN.md            # MongoDB schema diagrams & entity relationships
├── 📄 DEPLOYMENT_GUIDE.md           # Production deployment instructions (Docker/Cloud)
└── 📄 INTEGRATIONS_ARCHITECTURE.md  # Webhook ingestion, transformation & routing architecture
```

---

## 🎯 Architecture Summary Table

| Layer / Subsystem | Technology Stack | Core Responsibility |
| :--- | :--- | :--- |
| **Frontend UI** | React 18, Vite, Tailwind CSS | High-performance SPA with modular feature slices |
| **Real-Time Engine** | Socket.io (WebSockets) | Live chat, typing indicators, presence, whiteboard, notifications |
| **Media SFU** | Mediasoup + WebRTC | Multi-party HD audio/video calls, screen sharing |
| **REST API Server** | Node.js, Express | Modular controllers & routes for resources and business logic |
| **Primary Database** | MongoDB (Mongoose) | 28 Schemas for users, messages, channels, integrations, logs |
| **Cache & Queue** | Redis + Bull | Real-time presence cache, pub/sub, background asynchronous workers |
| **Integrations** | Webhook Engine & Bridge | External event ingestion (GitHub, Sentry, Inbound Email) |
| **AI Features** | Gemini / OpenAI Services | Message summaries, smart action-item extraction, semantic search |
