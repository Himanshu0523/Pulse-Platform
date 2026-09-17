# 📞 Mediasoup WebRTC Call Architecture & High-Scale Architecture Blueprint

This document details the **current WebRTC SFU calling architecture** implemented in the Pulse platform, highlights single-node scalability bottlenecks, and provides an end-to-end blueprint for scaling the system to support thousands of concurrent calls across multi-core CPU clusters and distributed server nodes.

---

## 📌 1. Current Call Architecture (Single SFU Worker)

Pulse utilizes a **Selective Forwarding Unit (SFU)** architecture powered by **Mediasoup**. Unlike Mesh (Peer-to-Peer) which suffers from $O(N^2)$ upload bandwidth overhead on clients, an SFU routes all media through a central server where each participant uploads **1 set of streams** (Audio/Video) and receives **$N-1$ streams** from other room members.

```
                          CURRENT CALL ARCHITECTURE (SFU)
                          
   ┌───────────────┐                                          ┌───────────────┐
   │  Client A     │                                          │  Client B     │
   │  (WebBrowser) │                                          │  (WebBrowser) │
   └───────┬───────┘                                          └───────┬───────┘
           │                                                          │
           │  1. Socket.IO Signaling (join-room, createTransport)    │
           ├──────────────────────────┐    ┌──────────────────────────┤
           │                          ▼    ▼                          │
           │                   ┌──────────────┐                       │
           │                   │  Socket.IO   │                       │
           │                   │  Signaling   │                       │
           │                   └──────┬───────┘                       │
           │                          │                               │
           │  2. Mediasoup IPC        ▼                               │
           │  ┌───────────────────────────────┐                       │
           │  │ Mediasoup Manager (Node.js)   │                       │
           │  └───────────────┬───────────────┘                       │
           │                  │                                       │
           │  3. C++ Worker   ▼                                       │
           │  ┌───────────────────────────────┐                       │
           │  │    Mediasoup C++ Worker       │                       │
           │  │  ┌─────────────────────────┐  │                       │
           │  │  │  Router (Room ID: R1)   │  │                       │
           │  │  │                         │  │                       │
           │  │  │  SendTransport (A) ────►│  │                       │
           │  │  │  RecvTransport (A) ◄────┤  │                       │
           │  │  │                         │  │                       │
           │  │  │  SendTransport (B) ────►│  │                       │
           │  │  │  RecvTransport (B) ◄────┤  │                       │
           │  │  └─────────────────────────┘  │                       │
           │  └───────────────────────────────┘                       │
           │                                                          │
   4. RTP  ▼ Direct WebRTC Transport (UDP/TCP Ports 40000-41000)      ▼ 4. RTP
   ═════════════════════════════════════════════════════════════════════════════
```

---

## 🛠️ 2. Detailed Media & Signaling Handshake Lifecycle

Below is the step-by-step connection flow executed when a user joins a call:

```
Client (useMediasoup.js)             Server (Socket.IO Handler)       Mediasoup Manager
        │                                       │                                │
        ├────── 1. join-room ──────────────────►│                                │
        │                                       ├────── getOrCreateRoom() ──────►│ (Creates Router)
        │                                       │◄───── routerRtpCapabilities ───┤
        │◄───── routerRtpCapabilities ──────────┤                                │
        │                                       │                                │
        ├────── 2. createWebRtcTransport ──────►│                                │
        │        (producing: true)              ├────── createWebRtcTransport()─►│ (Creates Transport A_send)
        │◄───── { id, iceParams, dtlsParams }───┤                                │
        │                                       │                                │
        ├────── 3. connectTransport ───────────►│                                │
        │        (transportId, dtlsParameters)  ├────── transport.connect() ────►│ (DTLS Handshake Complete)
        │                                       │                                │
        ├────── 4. produce ────────────────────►│                                │
        │        (kind, rtpParameters)          ├────── transport.produce() ────►│ (Creates Producer)
        │◄───── { producerId } ─────────────────┤                                │
        │                                       ├────── broadcast new-producer ─► All Other Room Members
        │                                       │                                │
        ├────── 5. consume ────────────────────►│                                │
        │        (remoteProducerId, rtpCaps)    ├────── router.canConsume() ────►│
        │                                       ├────── transport.consume() ────►│ (Creates Consumer)
        │◄───── { consumerId, rtpParameters }───┤                                │
        │                                       │                                │
        ├────── 6. resumeConsumer ─────────────►│                                │
        │                                       ├────── consumer.resume() ──────►│ (Media Begins Streaming)
```

---

## ⚠️ 3. Current Scalability Limitations (Bottlenecks)

While the single-worker Mediasoup setup works seamlessly for smaller deployments, it has structural bottlenecks when scaling up:

1. **Single-Thread C++ Worker Limit**:
   - `mediasoup.createWorker()` currently spawns **1 worker C++ process**.
   - Node.js/C++ single thread caps out at ~300–500 active audio/video tracks per core. Multi-core CPUs (e.g., 16 or 32 cores) are currently underutilized.
2. **Port Exhaustion**:
   - WebRTC UDP/TCP port range is statically set to `40000-41000` (1000 ports).
   - In a multi-party call with 50 users consuming video + audio, total active ports can rapidly deplete.
3. **Single Machine Memory & Bandwidth Ceiling**:
   - High-definition video streams require 1.5–2.5 Mbps per participant.
   - A single server node with 1 Gbps NIC saturates at ~400 concurrent video subscribers.
4. **Lack of Distributed State Sync**:
   - If user A connects to Signaling Node 1 and user B connects to Signaling Node 2, signaling events cannot cross server boundaries without Redis Pub/Sub signaling.

---

## 🚀 4. Blueprint for High-Scale Distributed Calling Architecture

To scale Pulse calls to support **10,000+ concurrent users** and **hundreds of large video rooms**, we implement a 4-tier horizontal scaling strategy:

```
                            HIGH-SCALE DISTRIBUTED SFU CLUSTER
                            
                             ┌────────────────────────┐
                             │    Load Balancer       │
                             │  (Nginx / AWS ALB)     │
                             └───────────┬────────────┘
                                         │
                 ┌───────────────────────┴───────────────────────┐
                 ▼                                               ▼
     ┌───────────────────────┐                       ┌───────────────────────┐
     │  Signaling Server 1   │                       │  Signaling Server 2   │
     │  (Express + Socket)   │                       │  (Express + Socket)   │
     └───────────┬───────────┘                       └───────────┬───────────┘
                 │                                               │
                 └───────────────────────┬───────────────────────┘
                                         │
                                         ▼
                         ┌───────────────────────────────┐
                         │   Redis Cluster / Pub-Sub     │
                         │  (@socket.io/redis-adapter)   │
                         └───────────────┬───────────────┘
                                         │
             ┌───────────────────────────┴───────────────────────────┐
             ▼                                                       ▼
  ┌──────────────────────────────────┐               ┌──────────────────────────────────┐
  │         SFU NODE 1               │               │         SFU NODE 2               │
  │ ┌──────────────────────────────┐ │               │ ┌──────────────────────────────┐ │
  │ │  Worker Pool (CPU Core 0-N)  │ │   Pipe        │ │  Worker Pool (CPU Core 0-N)  │ │
  │ │  ┌────────┐      ┌────────┐  │ │   Transport   │ │  ┌────────┐      ┌────────┐  │ │
  │ │  │Worker 1│      │Worker 2│  │ │ ◄───────────► │ │  │Worker 1│      │Worker 2│  │ │
  │ │  └────┬───┘      └────┬───┘  │ │  (Inter-Node) │ │  └────┬───┘      └────┬───┘  │ │
  │ └───────┼───────────────┼──────┘ │               │ └───────┼───────────────┼──────┘ │
  └─────────┼───────────────┼────────┘               └─────────┼───────────────┼────────┘
            ▼               ▼                                  ▼               ▼
     WebRTC Transports (Ports 40000-50000)             WebRTC Transports (Ports 40000-50000)
```

---

## 📐 5. Key Architecture Enhancements for Scalability

### A. Multi-Worker Engine (`WorkerPool`)
Instead of a single worker, spawn a pool of Mediasoup workers equal to `os.cpus().length`:

```javascript
const os = require('os');
const mediasoup = require('mediasoup');

class WorkerPool {
  constructor() {
    this.workers = [];
    this.nextWorkerIdx = 0;
  }

  async init() {
    const numWorkers = os.cpus().length;
    for (let i = 0; i < numWorkers; i++) {
      const worker = await mediasoup.createWorker({
        rtcMinPort: 40000 + (i * 1000),
        rtcMaxPort: 40999 + (i * 1000),
        logLevel: 'warn',
      });
      this.workers.push(worker);
    }
  }

  // Least-loaded or Round-Robin worker selection
  getWorker() {
    const worker = this.workers[this.nextWorkerIdx];
    this.nextWorkerIdx = (this.nextWorkerIdx + 1) % this.workers.length;
    return worker;
  }
}
```

---

### B. Inter-Worker & Inter-Router Media Piping (`pipeToRouter`)
When room participants span across different CPU workers or server nodes, use Mediasoup's `router.pipeToRouter()` mechanism to route media seamlessly between workers:

```javascript
// Route media stream produced on Worker 1 (Router 1) to Worker 2 (Router 2)
async function connectRouters(router1, router2, producerId) {
  const { pipeProducer, pipeConsumer } = await router1.pipeToRouter({
    producerId,
    router: router2
  });
  return { pipeProducer, pipeConsumer };
}
```

---

### C. Simulcast & VP8/VP9 Dynamic Quality Adaptation
Enable **Simulcast** on client video tracks to broadcast 3 quality spatial layers simultaneously (High 720p, Medium 360p, Low 180p). The SFU dynamically adjusts which layer to forward to each participant based on grid layout size and network bandwidth:

```javascript
// Client-side SendTransport produce call with Simulcast enabled
const producer = await sendTransport.produce({
  track: videoTrack,
  encodings: [
    { maxBitrate: 100000, scaleResolutionDownBy: 4.0 }, // Low: 180p
    { maxBitrate: 300000, scaleResolutionDownBy: 2.0 }, // Medium: 360p
    { maxBitrate: 900000, scaleResolutionDownBy: 1.0 }, // High: 720p
  ],
  codecOptions: { videoGoogleStartBitrate: 1000 }
});
```

---

### D. Scalable Signaling via Redis Adapter
Decouple Socket.IO across multiple server instances using `@socket.io/redis-adapter`. This ensures signaling messages (call invitations, candidate exchanges, producer alerts) reach participants regardless of which server node they are connected to:

```javascript
const { createClient } = require('redis');
const { createAdapter } = require('@socket.io/redis-adapter');

const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();

Promise.all([pubClient.connect(), subClient.connect()]).then(() => {
  io.adapter(createAdapter(pubClient, subClient));
});
```

---

## 📊 Summary of Architectural Upgrades

| Component | Current State | Scalable Enterprise Blueprint |
| :--- | :--- | :--- |
| **Worker Threads** | Single C++ process (1 CPU Core) | Worker Pool across all available CPU cores |
| **Port Range** | 40000–41000 (1,000 ports) | Dynamic range allocated per worker (40000–59999) |
| **Multi-Server Routing** | Local Map in single memory | `router.pipeToRouter()` + Redis Node Registry |
| **Video Quality** | Single Bitrate stream | 3-Layer Simulcast (Low, Med, High) |
| **Signaling Node** | Monolithic single Socket instance | Clustered Socket.IO nodes with Redis Pub/Sub |
| **Max Room Capacity** | ~20–30 active video streams | 500+ participants per call room |

---
