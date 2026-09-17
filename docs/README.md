# 📚 Private Pulse Platform — Master Documentation Hub

Welcome to the central documentation index for the **Private Pulse Platform** (`Pulse-chat-app`), a production-grade enterprise collaboration platform featuring real-time messaging, WebRTC audio/video SFU conferencing, collaborative whiteboards, pluggable enterprise integrations, and end-to-end security.

---

## 🧭 Documentation Sitemap & Navigation

```
docs/
├── README.md                          # Master Documentation Hub (You are here)
│
├── architecture/                      # System Architecture & Technical Specifications
│   ├── overview.md                    # Core architecture deep dive, domain ownership, and design foundations
│   ├── system-overview.md             # Component topology, layer responsibilities, C4 context
│   ├── backend-architecture.md        # Node.js, Express, middleware pipeline & service contracts
│   ├── realtime-architecture.md       # Socket.IO event buses, Redis pub/sub adapter, heartbeats
│   ├── webrtc-architecture.md         # Mediasoup SFU topologies, simulcast, RTP transport pipelines
│   ├── chat-architecture.md           # E2EE messaging, thread replies, buffer queues & offline sync
│   ├── call-architecture.md           # Multi-party audio/video routing & SFU worker core balancing
│   ├── data-architecture.md           # MongoDB schemas, compound indices, GridFS storage, Redis keys
│   ├── database-design.md             # Entity Relationship Diagrams (ERD) & relational data maps
│   ├── domain-boundaries.md           # Frozen V1/V2 domain matrices and module contracts
│   ├── integration-architecture.md    # Pluggable adapter framework, HMAC webhooks, BullMQ queues
│   └── integrations-deep-dive.md      # Detailed event schema & execution pipeline specifications
│
├── api/                               # API, WebSocket & Protocol Specifications
│   ├── api-specification.md           # REST endpoint schemas, headers, status codes & envelopes
│   ├── frontend-integration-guide.md  # Comprehensive guide for frontend devs with socket events
│   └── authentication-flow.md         # Dual JWT token families, rotation, WebAuthn & Argon2id
│
├── security/                          # Security Engineering & Compliance Blueprints
│   ├── authorization-matrix.md        # Granular RBAC permissions matrix & multi-tenant isolation rules
│   └── testing-checklist.md           # 73-point enterprise security & penetration testing checklist
│
├── adr/                               # Architecture Decision Records (ADRs)
│   ├── ADR-001-mediasoup-sfu-vs-mesh.md         # Decision: Mediasoup SFU over P2P Mesh
│   ├── ADR-002-socketio-with-redis-adapter.md   # Decision: Socket.IO with Redis adapter for horizontal scale
│   ├── ADR-003-hybrid-cloud-storage.md          # Decision: GridFS vs Cloudinary / S3 storage tiers
│   ├── ADR-004-jwt-token-rotation-and-csrf.md   # Decision: Token-family rotation and double-submit CSRF
│   └── ADR-005-message-buffer-write-pipeline.md # Decision: Redis buffer write pipeline for high-volume chat
│
├── operations/                        # Deployment, Infrastructure & CI/CD
│   ├── deployment-guide.md            # Production deployment runbook (Render, Vercel, Docker)
│   └── git-workflow.md                # Branching conventions, pull requests, automated checks
│
├── guides/                            # Developer & Contributor Guides
│   ├── codebase-walkthrough.md        # Comprehensive component, controller, hook, and service catalog
│   ├── folder-structure.md            # Complete file tree, naming conventions, and file roles
│   └── sfu-collaboration-blueprint.md # Original design blueprint for real-time collaboration
│
└── roadmap/                           # Product Roadmap, Planning & Milestones
    ├── enterprise-roadmap.md          # Multi-tenant workspace backlog & enterprise feature plans
    ├── migration-v1-to-v2.md          # V1 to V2 migration roadmap (microservices & event buses)
    ├── competition-analysis.md        # In-depth competitive analysis vs Slack, Teams, and Discord
    ├── execution-tracker.md           # Phase milestone execution tracker and feature completion status
    └── implementation-plan.md         # System rollout strategy and deployment schedule
```

---

## 🎯 Role-Based Reading Paths

| Role | Recommended Reading Path |
| :--- | :--- |
| **New Frontend Developer** | 1. [`guides/codebase-walkthrough.md`](./guides/codebase-walkthrough.md)<br>2. [`api/frontend-integration-guide.md`](./api/frontend-integration-guide.md)<br>3. [`api/api-specification.md`](./api/api-specification.md)<br>4. [`architecture/realtime-architecture.md`](./architecture/realtime-architecture.md) |
| **Backend & Core Engineer** | 1. [`architecture/overview.md`](./architecture/overview.md)<br>2. [`architecture/backend-architecture.md`](./architecture/backend-architecture.md)<br>3. [`architecture/database-design.md`](./architecture/database-design.md)<br>4. [`architecture/webrtc-architecture.md`](./architecture/webrtc-architecture.md) |
| **DevOps & Infrastructure** | 1. [`operations/deployment-guide.md`](./operations/deployment-guide.md)<br>2. [`operations/git-workflow.md`](./operations/git-workflow.md)<br>3. [`adr/`](./adr/) |
| **Security & Compliance Auditor** | 1. [`security/authorization-matrix.md`](./security/authorization-matrix.md)<br>2. [`security/testing-checklist.md`](./security/testing-checklist.md)<br>3. [`api/authentication-flow.md`](./api/authentication-flow.md)<br>4. [`adr/ADR-004-jwt-token-rotation-and-csrf.md`](./adr/ADR-004-jwt-token-rotation-and-csrf.md) |
| **Product Manager & Architect** | 1. [`architecture/domain-boundaries.md`](./architecture/domain-boundaries.md)<br>2. [`roadmap/competition-analysis.md`](./roadmap/competition-analysis.md)<br>3. [`roadmap/enterprise-roadmap.md`](./roadmap/enterprise-roadmap.md)<br>4. [`roadmap/migration-v1-to-v2.md`](./roadmap/migration-v1-to-v2.md) |

---

## 🏛️ Section Summaries

### 1. Architecture (`docs/architecture/`)
* **[`overview.md`](./architecture/overview.md)** — Architectural principles, domain boundaries, service layers, and core dependencies.
* **[`system-overview.md`](./architecture/system-overview.md)** — Macro-level system diagram, gateway architecture, and data flow topologies.
* **[`backend-architecture.md`](./architecture/backend-architecture.md)** — Express 5 server configuration, middleware chain, rate limiters, and error handling.
* **[`realtime-architecture.md`](./architecture/realtime-architecture.md)** — Socket.IO protocol, namespace design, room management, and presence tracking.
* **[`webrtc-architecture.md`](./architecture/webrtc-architecture.md)** — Mediasoup worker pool, Routers, Transports, Producers, Consumers, and simulcast bitrates.
* **[`chat-architecture.md`](./architecture/chat-architecture.md)** — E2EE Signal-protocol-style messaging, thread replies, reactions, and search indexing.
* **[`call-architecture.md`](./architecture/call-architecture.md)** — Multi-party voice/video SFU scaling, screen sharing, and audio level detection.
* **[`data-architecture.md`](./architecture/data-architecture.md)** — Primary data models, Mongoose schema constraints, GridFS storage, and caching strategies.
* **[`database-design.md`](./architecture/database-design.md)** — Relational structure, compound index strategies, and shard key recommendations.
* **[`domain-boundaries.md`](./architecture/domain-boundaries.md)** — Strict separation of concerns between Auth, Workspaces, Chat, Calls, Files, and Integrations.
* **[`integration-architecture.md`](./architecture/integration-architecture.md)** — Inbound webhook verification (HMAC SHA-256), BullMQ queues, and credential encryption (AES-256-GCM).
* **[`integrations-deep-dive.md`](./architecture/integrations-deep-dive.md)** — Event transformation pipeline, rate limiting, and third-party routing rules.

### 2. API & Protocols (`docs/api/`)
* **[`api-specification.md`](./api/api-specification.md)** — Full REST API catalog covering Auth, Users, Workspaces, Channels, Messages, Calls, and Integrations.
* **[`frontend-integration-guide.md`](./api/frontend-integration-guide.md)** — Practical developer guide containing sample request/response envelopes and client-side Socket.IO listeners.
* **[`authentication-flow.md`](./api/authentication-flow.md)** — Deep dive into Argon2id password hashing, HIBP breach defense, WebAuthn/Passkeys, and dual-token refresh family rotation.

### 3. Security & Compliance (`docs/security/`)
* **[`authorization-matrix.md`](./security/authorization-matrix.md)** — Permission matrix across Owner, Admin, Moderator, Member, and Guest across all system resources.
* **[`testing-checklist.md`](./security/testing-checklist.md)** — Comprehensive 73-point verification plan covering OWASP Top 10, multi-tenant isolation, SSRF, and cryptographic audits.

### 4. Architecture Decision Records (`docs/adr/`)
* Contains formal records for major system design decisions, including the SFU vs Mesh evaluation, Redis adapter selection, token security, and storage architecture.

### 5. Operations & CI/CD (`docs/operations/`)
* **[`deployment-guide.md`](./operations/deployment-guide.md)** — Deployment guides for Render (backend + mediasoup native C++ compilation), Vercel (frontend SPA), and Docker Compose.
* **[`git-workflow.md`](./operations/git-workflow.md)** — Branch naming taxonomy, commit guidelines, and automated GitHub Actions CI pipeline requirements.

### 6. Developer Guides (`docs/guides/`)
* **[`codebase-walkthrough.md`](./guides/codebase-walkthrough.md)** — Complete file-by-file walkthrough of frontend React pages/components and backend services.
* **[`folder-structure.md`](./guides/folder-structure.md)** — Standardized file and directory organization structure.
* **[`sfu-collaboration-blueprint.md`](./guides/sfu-collaboration-blueprint.md)** — Architectural design blueprint for real-time collaboration platforms.

### 7. Roadmap & Product Evolution (`docs/roadmap/`)
* **[`enterprise-roadmap.md`](./roadmap/enterprise-roadmap.md)** — Workspaces, enterprise audit logs, fine-grained notification rules, and custom integrations.
* **[`migration-v1-to-v2.md`](./roadmap/migration-v1-to-v2.md)** — Scalability roadmap transitioning from monolith to distributed microservices.
* **[`competition-analysis.md`](./roadmap/competition-analysis.md)** — Feature comparison against Slack, Discord, Microsoft Teams, and Zoom.
* **[`execution-tracker.md`](./roadmap/execution-tracker.md)** — Detailed milestone progress logs and deliverables status.
* **[`implementation-plan.md`](./roadmap/implementation-plan.md)** — Phased rollout timeline.
