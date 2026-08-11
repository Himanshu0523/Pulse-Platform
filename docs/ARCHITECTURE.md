# Private Pulse Platform — Deep Dive System Architecture

This document provides a technical analysis of the system architecture, design decisions, and engineering trade-offs made in the building of **Private Pulse Platform**.

---

## 1. System Overview & Core Objectives

Private Pulse Platform is designed as an ultra-low latency, scalable real-time collaboration ecosystem. The primary design goals are:
- **Sub-100ms Media Latency**: Utilizing Selective Forwarding Units (SFU) via MediaSoup instead of Mesh topologies for group video.
- **Horizontal Gateway Scalability**: Socket.IO instances backed by Redis Pub/Sub adapters.
- **Resilient Security**: End-to-end encryption principles for sensitive data, JWT rotation, and context-isolated request pipelines.

---

## 2. End-to-End System Architecture

```mermaid
flowchart TD
    subgraph Client ["Client Tier (Vercel Global Edge)"]
        ReactApp["Vite + React 18 SPA"]
        SocketEngine["Socket.IO Client Engine"]
        MediaDevice["MediaSoup RTP Device"]
    end

    subgraph API Gateway ["API Gateway Tier (Render)"]
        CORS["CORS & Request Context Isolation"]
        ExpressRouter["Express.js REST Engine"]
        AuthGuard["JWT & RBAC Middleware"]
    end

    subgraph RealTime Engine ["Real-Time & Media Tier"]
        SocketGateway["Socket.IO Server Cluster"]
        MediaSoupSFU["MediaSoup Worker (C++ Native SFU)"]
    end

    subgraph Persistence ["Persistence & Data Tier"]
        MongoDB[("MongoDB Atlas Primary Store")]
        RedisCache[("Upstash Redis Cache & PubSub")]
    end

    ReactApp -->|"REST HTTPS"| CORS
    CORS --> ExpressRouter --> AuthGuard --> MongoDB
    SocketEngine <-->|"WSS Handshake + Events"| SocketGateway
    MediaDevice <-->|"WebRTC Transport (RTP/RTCP)"| MediaSoupSFU
    SocketGateway <-->|"Redis Adapter Pub/Sub Sync"| RedisCache
```

---

## 3. WebRTC Media Architecture: Mesh vs. SFU Trade-off

### The Challenge
Pure Peer-to-Peer (Mesh) WebRTC requires $N \times (N-1)$ connections. In a group call with 6 participants, each client must encode and transmit 5 video streams, saturating client uplink bandwidth and CPU.

### The Solution: Selective Forwarding Unit (MediaSoup)
We implemented **MediaSoup SFU** at the server layer:
- **Producers**: Each client sends exactly **1 video stream** and **1 audio stream** to the SFU worker.
- **Consumers**: The SFU forwards incoming RTP streams to all other connected participants without re-encoding, reducing CPU overhead to near zero.

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client Device
    participant Gateway as Express / Socket.IO
    participant SFU as MediaSoup SFU Worker

    Client->>Gateway: Request WebRTC Transport Options
    Gateway->>SFU: Create WebRtcTransport (Producer)
    SFU-->>Gateway: Return DTLS Parameters & Transport ID
    Gateway-->>Client: Transport Created
    Client->>SFU: Connect Transport & Produce Audio/Video Track
    SFU-->>Client: Return Producer ID
    Note over Client,SFU: Real-Time WebRTC Media Flow Activated
```

---

## 4. Real-Time Gateway & Scaling Strategy

```mermaid
flowchart LR
    NodeA["Socket.IO Node A"] <-->|"Pub/Sub"| Redis[("Redis Adapter")]
    NodeB["Socket.IO Node B"] <-->|"Pub/Sub"| Redis
    NodeC["Socket.IO Node C"] <-->|"Pub/Sub"| Redis

    User1["User 1 (Node A)"] --> NodeA
    User2["User 2 (Node C)"] --> NodeC
```

By leveraging `@socket.io/redis-adapter`:
- State is decoupled from single Node.js processes.
- Messages sent in `user:{userId}` or `room:{roomId}` on Node A broadcast across Redis and deliver instantly to User 2 connected to Node C.

---

## 5. Security & Engineering Best Practices

1. **Anti-Flicker UX & Micro-Animations**: Loading indicators utilize a **200ms anti-flicker delay rule** and Conic Gradient rendering with full ARIA screen reader compliance.
2. **Context Isolation**: Every HTTP request is tagged with a unique `x-request-id` tracked using Node's `AsyncLocalStorage` for distributed log correlation.
3. **CORS Hygiene**: Strict preflight handling for production domains (`https://private-pulse-platform.vercel.app`) positioned at the root of the middleware pipeline.
