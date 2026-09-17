# Pulse Codebase Description & Feature Guide

This document provides a comprehensive overview of the Pulse Real-Time Communication Platform codebase. It describes each feature, maps them to their respective client-side and server-side files, and details the role of each component, hook, store, controller, model, and service.

---

## 📌 Core Features Summary

1. **User Authentication & Session Management**: Secure register/login, dual JWT token rotation, and local profile configurations.
2. **Real-Time Private & Group Chatting**: Fast message transport via WebSockets with typing indicators, read receipts, and edit/delete support.
3. **High-Quality WebRTC Video & Audio Calls**: Selective Forwarding Unit (SFU) architecture using **Mediasoup** to enable scalable multi-user calls.
4. **Collaborative Shared Whiteboard**: Interactive canvas sync for users in active room calls with drawing tools (brush, undo/redo, clear).
5. **Secure File Sharing**: Multipart file upload via Multer, stored in MongoDB GridFS, with upload progress bars and inline download/preview options.
6. **Presence System**: Dynamic status tracker (Online, Away, Busy, Offline) using Redis caching and Socket.IO heartbeat signals.
7. **Friend & Block List Management**: Interactive system to send requests, accept/reject, delete friends, and block/unblock users.
8. **In-App Notification Center**: Pop-ups and persistent alerts for mentions, friend requests, incoming/missed calls, and new messages.

---

## 🏗️ Architecture & Folder Structure

```
Project-4/
├── client/                 # React Frontend
│   └── src/
│       ├── app/            # Global App setup, providers
│       ├── components/     # Shared UI components (Modals, Buttons, Inputs)
│       ├── config/         # System config (Axios & Socket clients)
│       ├── context/        # React Context providers (Theme, Auth)
│       ├── features/       # Feature-driven modular files 🚀
│       ├── hooks/          # Shared global React hooks
│       ├── layouts/        # Page layouts (Auth, Dashboard, Meeting)
│       ├── pages/          # Individual app pages / views
│       ├── routes/         # React Router 7 setup
│       ├── store/          # Zustand state stores (global state)
│       └── utils/          # Formatting and sound utilities
└── server/                 # Express Backend
    └── src/
        ├── config/         # Database and cache initializations
        ├── controllers/    # API controllers receiving REST requests
        ├── mediasoup/      # WebRTC SFU Workers & Router managers
        ├── middleware/     # Auth, logs, security, error middlewares
        ├── models/         # Mongoose DB schema definitions
        ├── routes/         # Express endpoint mappings
        ├── services/       # Core business logic handlers
        ├── socket/         # Socket.IO handlers for real-time traffic
        └── utils/          # Prometheus metrics, logging, token utilities
```

---

## 🛡️ Feature 1: User Authentication & Profile

Handles registering new users, managing access/refresh JWT tokens, profile details editing, settings changes, and custom avatar uploads.

### 💻 Client-Side Files
* **`client/src/features/auth/api/`**
  * `register.js` | `login.js` | `logout.js` | `refresh.js`: API request calls mapping to `/api/auth/` backend endpoints.
* **`client/src/features/auth/components/`**
  * `LoginForm.jsx` & `RegisterForm.jsx`: UI forms with validation.
* **`client/src/features/auth/hooks/`**
  * `useLogin.js` | `useRegister.js` | `useLogout.js`: React Query wrapper hooks updating state stores on success.
* **`client/src/features/auth/validation/`**
  * `loginSchema.js` & `registerSchema.js`: Input validation models.
* **`client/src/features/profile/`**
  * `api/profile.js` & `api/settings.js`: Routes to update name, bio, and settings.
  * `components/AvatarUpload.jsx`: Handles avatar crop, selection, and upload.
  * `components/EditProfileForm.jsx` & `SettingsForm.jsx`: Configuration UI interfaces.
  * `hooks/useProfile.js` & `useSettings.js`: React Query hooks for fetching and updating user configurations.
* **`client/src/store/authStore.js`**
  * Zustand store that persists user objects and temporary access tokens.

### ⚙️ Server-Side Files
* **`server/src/models/User.js`**
  * Database schema for user login info, settings configurations, and array of active refresh tokens.
* **`server/src/routes/auth.routes.js`** & **`user.routes.js`**
  * Express endpoints defining routes for register, login, refresh, profile, status, and settings updates.
* **`server/src/controllers/auth.controller.js`** & **`user.controller.js`**
  * Authenticates users (bcrypt comparison), signs access/refresh tokens, and updates database records.
* **`server/src/middleware/auth.middleware.js`**
  * Extracts Bearer tokens from incoming HTTP request headers and validates JWT.
* **`server/src/utils/jwt.js`**
  * Sign and verify JWT algorithms for access and refresh tokens.

---

## 💬 Feature 2: Chat & Messaging (Private & Groups)

Supports 1-on-1 private messaging, typing indicators, read receipts, message reactions, edits, deletion, and advanced group administration (add/remove members, role promotion).

### 💻 Client-Side Files
* **`client/src/features/chat/api/`**
  * `conversations.js` | `messages.js` | `groups.js`: API requests handling CRUD operations on messages and conversations.
* **`client/src/features/chat/components/`**
  * `MessageBubble.jsx`: Individual chat bubble rendering messages, reactions, replies, and edits.
  * `SendMessageForm.jsx`: Custom textarea with typing triggers, file attachments, and send events.
  * `TypingIndicator.jsx`: Renders real-time dot animation when another user is typing.
  * `AddMembersModal.jsx` | `EditGroupModal.jsx` | `MemberListModal.jsx`: Interfaces to manage group details and members.
* **`client/src/features/chat/hooks/`**
  * `useConversations.js` & `useMessages.js`: Manage fetching and updating state of active chats.
  * `useSendMessage.js` & `useTyping.js`: Hook wrappers triggered when sending characters or active text editing.
  * `useChatSocket.js`: Connects Socket.IO incoming events (like new messages or message edits) to refresh query caches.
* **`client/src/features/chat/store/chatStore.js`**
  * Zustand store maintaining state of selected conversation, message drafts, and typing user list.

### ⚙️ Server-Side Files
* **`server/src/models/Conversation.js`** & **`Message.js`**
  * Schemas defining conversations (group name, avatar, members, last message timestamp) and messages (sender, content, read status, reactions array, edit history, replies).
* **`server/src/routes/conversation.routes.js`** | `message.routes.js` | `group.routes.js`
  * Express routers for fetching conversations, paginated messages, creating groups, and role edits.
* **`server/src/controllers/conversation.controller.js`** | `message.controller.js` | `group.controller.js`
  * Express handlers managing chat creation, message deletion logs, reactions toggling, and membership updates.
* **`server/src/socket/chat.socket.js`**
  * Handles WebSocket connections joining room chambers, broadcasting message payloads, typing alerts, and read/delivered acknowledgements.

---

## 📞 Feature 3: WebRTC Video & Audio Calls (Mediasoup SFU)

Enables real-time 1-on-1 or multi-party video and audio conferencing. Media is routed through a high-performance Selective Forwarding Unit (SFU) using **Mediasoup**. For full architectural details and horizontal scaling blueprints, see [CALL_ARCHITECTURE.md](CALL_ARCHITECTURE.md).

### 📐 Media & Signaling Flow Architecture

```
Client (WebRTC Device)         Socket.IO Signaling            Mediasoup C++ Worker
        │                              │                               │
        ├──── 1. join-room ───────────►│                               │
        │                              ├──── getOrCreateRoom() ───────►│ (Spawns Router)
        ├──── 2. createTransport ─────►│                               │
        │                              ├──── createWebRtcTransport() ─►│ (Send/Recv Transport)
        ├──── 3. produce ─────────────►│                               │
        │    (Audio/Video Track)       ├──── transport.produce() ─────►│ (Creates Producer)
        │                              ├──── Broadcast new-producer ──► Other Clients
        ├──── 4. consume ─────────────►│                               │
        │                              ├──── transport.consume() ─────►│ (Creates Consumer)
        │◄─── Media Streaming ─────────┴───────────────────────────────┤ (RTP Direct UDP/TCP)
```

### 💻 Client-Side Files
* **`client/src/features/calls/`**
  * `api/calls.js`: REST calls to log initiate/end timestamps in DB call history.
  * `components/VideoGrid.jsx`: Dynamically renders local and remote WebRTC video elements.
  * `components/CallControls.jsx`: Mutes mic, disables video, toggles screen sharing, and triggers disconnection.
  * `components/ActiveCallBanner.jsx` & `IncomingCallOverlay.jsx`: Overlay HUD notifications indicating active call details or incoming call ringers.
  * `hooks/useMediasoup.js`: Connects to Mediasoup client API. Configures local WebRTC Device, registers Send/Receive transports, produces local camera streams, and consumes remote streams.
  * `hooks/useCall.js` & `useCallListener.js`: Interfaces calling states with user events (place call, answer, hangup, ring triggers).
  * `store/callStore.js`: Zustand store storing the active call status, connection states, mute options, and participant objects.
* **`client/src/hooks/useWebRTC.js`**
  * Low-level WebRTC fallback helpers.

### ⚙️ Server-Side Files
* **`server/src/mediasoup/`**
  * `config.js`: Setup file for worker pools, routers, WebRTC transports, ports, IP configurations, and media codecs.
  * `manager.js`: Controls creating workers, assigning rooms to routers, setting up send/receive transports, and handling stream production and consumption.
* **`server/src/socket/mediasoup.handlers.js`**
  * Translates client transport requests (createTransport, connectTransport, produce, consume, resume) into backend Mediasoup commands.
* **`server/src/socket/call.socket.js`**
  * Manage call signaling protocols (`call:initiate`, `call:accept`, `call:reject`, `call:end`) to trigger device vibration, sound effects, and DB call logs.
* **`server/src/models/Call.js`**
  * Mongoose model logging call type (audio/video), participants, duration, initiator, and end status.

### 🚀 Scalable Architecture Blueprint Overview (Detailed in CALL_ARCHITECTURE.md)
* **Multi-Worker CPU Core Pooling**: Spawning $N$ C++ Mediasoup Workers matching total server CPU cores (`os.cpus().length`).
* **Inter-Worker Media Piping (`pipeToRouter`)**: Dynamic media routing between Routers across worker threads and remote server nodes.
* **Redis Pub/Sub Signaling Adapter**: Decoupled Socket.IO signaling across load-balanced application servers using `@socket.io/redis-adapter`.
* **Dynamic Simulcast & Quality Adaptation**: 3 spatial resolution layers (High 720p, Medium 360p, Low 180p) automatically adapted per subscriber by the SFU.

---

## 🎨 Feature 4: Collaborative Shared Whiteboard

Allows participants in a room to draw synchronously on a collaborative digital canvas.

### 💻 Client-Side Files
* **`client/src/features/whiteboard/`**
  * `components/WhiteboardCanvas.jsx`: Core canvas rendering logic using HTML5 Canvas APIs, trackpad, and mouse event listeners.
  * `components/WhiteboardToolbar.jsx`: User buttons panel selecting pencil size, brush color, eraser, undo/redo actions, and canvas clearing.
  * `hooks/useWhiteboard.js` & `useWhiteboardSocket.js`: Connects drawing paths and mouse coordinates to Socket.IO events to distribute actions in real-time.
  * `store/whiteboardStore.js`: Zustand store tracking brush settings, drawing history arrays, and undo/redo stacks.

### ⚙️ Server-Side Files
* **`server/src/whiteboard/manager.js`**
  * Ephemeral storage managing the drawing state history per meeting room. Replays the stroke history to new participants joining late.
* **`server/src/socket/whiteboard.handlers.js`**
  * Socket listener routing drawing actions (`draw`, `clear`, `undo`, `redo`) to all participants in a room and caching the state in the manager.

---

## 📎 Feature 5: Secure File Sharing

Supports uploading, downloading, and previewing files/images directly inside chat interfaces.

### 💻 Client-Side Files
* **`client/src/features/files/`**
  * `api/files.js`: Backend routes to upload multipart data or download file hashes.
  * `components/FileUpload.jsx`: Input drop zone emitting upload progress updates via socket.
  * `components/FilePreviewModal.jsx` & `FileList.jsx`: Renders preview panels for images, audio/video, or document links.
  * `hooks/useUploadFile.js` & `useFiles.js`: Wraps React Query file fetching and Axios post uploads.

### ⚙️ Server-Side Files
* **`server/src/models/File.js`**
  * Schema storing metadata of uploaded files (size, mimetype, GridFS file reference, uploader, conversation).
* **`server/src/utils/gridfs.js`**
  * Connects GridFS storage engine to MongoDB. Handles reading streams, writing streams, and file deletion.
* **`server/src/middleware/upload.middleware.js`**
  * Multer storage engine routing files to GridFS bucket.
* **`server/src/controllers/file.controller.js`** & `routes/file.routes.js`
  * Rest API streams files directly to clients from GridFS, checking user access parameters.

---

## 📍 Feature 6: Presence Management

Keeps track of user connection states (Online, Away, Busy, Offline) and updates this information dynamically.

### 💻 Client-Side Files
* **`client/src/store/socketStore.js`**
  * Connects Socket client, handles authorization tokens, updates online lists.
* **`client/src/hooks/useSocket.js`**
  * Custom hook wrapping socket connect/disconnect lifecycles.

### ⚙️ Server-Side Files
* **`server/src/config/redis.js`** & **`utils/redis.js`**
  * Configuration to read/write key values in Redis cache to track user online tokens.
* **`server/src/services/presence.service.js`**
  * Business logic updating status settings, tracking user heartbeat logs, and computing offline duration.
* **`server/src/socket/presence.socket.js`**
  * Socket handlers detecting active disconnects, publishing presence updates to the Redis Pub/Sub network, and updating MongoDB's `lastSeen` values.

---

## 👥 Feature 7: Friend System

Enables sending, accepting, rejecting, or blocking user relationships.

### 💻 Client-Side Files
* **`client/src/features/friends/`**
  * `api/friends.js` & `search.js`: API endpoints for friend requests and user search.
  * `hooks/useFriends.js` | `useFriendRequests.js` | `useSearchUsers.js`: React Query hooks mapping database records to UI components like the Friends Hub.

### ⚙️ Server-Side Files
* **`server/src/models/Friend.js`**
  * Schema tracking relationships: requester, recipient, and status (`pending`, `accepted`, `rejected`, `blocked`).
* **`server/src/controllers/friend.controller.js`** & `routes/friend.routes.js`
  * Handles logic for friending, unfriending, and blocking users, emitting real-time updates via Socket.IO.

---

## 🔔 Feature 8: Notification Center

Pops up live alert banners and lists persistent notifications for user activity.

### 💻 Client-Side Files
* **`client/src/features/notifications/`**
  * `api/notifications.js`: Fetches persistent notifications and marks them as read.
  * `components/NotificationBell.jsx`: Trigger button showing total unread count.
  * `components/NotificationItem.jsx`: Individual item layout for various notification types.
  * `hooks/useNotificationSocket.js` & `useNotifications.js`: Listens to incoming real-time socket events to push browser banners.
  * `store/notificationStore.js`: Zustand store holding current notification list.

### ⚙️ Server-Side Files
* **`server/src/models/Notification.js`**
  * Schema capturing alert context: recipient, sender, reference event ID, type, and read status.
* **`server/src/services/notification.service.js`**
  * Processes notification triggers, saves them to MongoDB, and pushes live WebSocket alerts.

---

## 🔗 Feature 9: URL Extraction & Rich Link Preview Engine

Extracts hyperlinks from incoming chat messages, processes them asynchronously through a background queue (BullMQ), scrapes metadata (OpenGraph/OEmbed), caches results in Redis, and broadcasts live previews to client UI components.

### 📐 Pipeline Architecture & Flowchart

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
                                       └─────────────────────┘
```

### 💻 Client-Side Files
* **`client/src/features/chat/components/URLPreview/`**
  * `PreviewCard.jsx`: Container delegating extracted URLs to platform-specific rich preview components.
  * `GitHubPreview.jsx`: Renders GitHub repository details, stars, forks, and issues.
  * `YouTubePreview.jsx`: Renders inline video preview cards with embedded players.
  * `FigmaPreview.jsx`: Renders embedded design frame cards for Figma links.
  * `JiraPreview.jsx`: Displays issue status, priority, and assignee details for Jira ticket links.
  * `GenericPreview.jsx`: Fallback OpenGraph card displaying thumbnail image, title, and description.
  * `PreviewError.jsx`: Handles unavailable or restricted URL states gracefully.
* **`client/src/features/chat/hooks/useMessagePreview.js`**
  * Subscribes to real-time `message:preview` socket events and merges link metadata into message store items.

### ⚙️ Server-Side Files
* **`server/src/services/preview.service.js`**
  * Parses message strings using regex, detects valid URLs, normalizes domains, and fetches OpenGraph / metadata.
* **`server/src/worker/linkPreview.worker.js`**
  * BullMQ worker executing link parsing, scraping, and platform detection in background threads.
* **`server/src/socket/chat.socket.js`**
  * Emits `message:preview` payloads to active room participants once link metadata processing completes.

---

## 🔐 Feature 10: Client-Side End-to-End Encryption (E2EE)

Provides client-side cryptographic security using Web Crypto API standards (AES-256-GCM symmetric session keys + ECDH key exchange & RSA key pairs). Raw message text is encrypted on the sender's browser and decrypted only on recipient devices.

### 💻 Client-Side Files
* **`client/src/context/EncryptionContext.jsx`**
  * React Context managing key derivation, session key caches, and device key generation.
* **`client/src/hooks/useEncryption.js` & `useSetupEncryption.js`**
  * Custom hooks for key initialization, public key publication, and message encryption/decryption routines.
* **`client/src/features/chat/hooks/useDecryptedMessage.js`**
  * Decrypts incoming encrypted message envelopes asynchronously before rendering in the chat bubble.
* **`client/src/components/common/E2eeUnlockModal.jsx` & `E2eeStatusHeader.jsx`**
  * UI modals for master passphrase entry and active encryption status indicators.

### ⚙️ Server-Side Files
* **`server/src/models/Message.js`**
  * Supports encrypted envelope payloads (`isEncrypted`, `ciphertext`, `nonce`, `keyId`).
* **`server/src/controllers/message.controller.js`**
  * Stores and routes ciphertext without reading or modifying message contents.

---




add - 
AI Workspace Memory & Semantic Search
Contextual Workspace Memory: Stores and indexes historical discussions, meeting transcripts, uploaded files, and whiteboards to answer queries about past decisions.

Vector-Based Semantic Search: Locates information based on intent and meaning rather than strict keyword matching (e.g., finding discussions about "authentication changes" without using those exact words).

Multi-Modal Retrieval: Unified search across text messages, audio transcripts, shared documents, and whiteboard notes.

2. Automated Knowledge & Decision Tracking
Automated Decision Extraction: AI detects and logs key organizational decisions made during chats or meetings into a centralized Decision Register.

Decision Traceability: Links every decision directly back to the original message thread, meeting recording, or document where it was discussed.

Workspace Knowledge Graph: Visually maps relationships between team members, projects, decisions, code repositories, and documentation.

3. Intelligent Workspace Management
AI Daily Briefings: Generates personalized morning summaries highlighting key decisions, critical project updates, unread blockers, and upcoming tasks.

Automated Task Synthesis: Analyzes meeting transcripts and chat threads to automatically draft actionable tasks, assign owners, and set deadlines.

Workspace Timeline: Displays a chronological event stream combining meetings, key file uploads, pull requests, and major decisions in one timeline.

4. Advanced Enterprise & Security Features
Granular Role-Based Access Control (RBAC): Customizable permission tiers for enterprise administrators, department leads, guests, and external clients.

Automated Compliance Logging: Audit trails tracking user access, file downloads, permission changes, and security configurations.

Zero-Trust Encryption Layers: End-to-end encryption (E2EE) with client-side key management for sensitive channels and private documents.