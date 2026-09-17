# WebRTC Architecture

> **Scope:** Two WebRTC modes — SFU via Mediasoup and Mesh signaling via `room.socket.js`.

---

## 1. Two WebRTC Modes

Pulse implements **two parallel WebRTC subsystems** for different use cases:

| Mode | Use Case | Max Participants | Implementation |
|------|----------|-----------------|----------------|
| **SFU (Selective Forwarding Unit)** | Video calls, group meetings | Many (server-routed) | Mediasoup 3.20 |
| **Mesh** | Peer-to-peer ad-hoc rooms | ~4 (browser-limited) | WebRTC + `room.socket.js` signaling |

---

## 2. SFU Mode — Mediasoup

### Architecture

```
Browser A                  Mediasoup Server               Browser B
    │                           │                              │
    │── ms:create-transport ───▶│                              │
    │◀─ transportParams ────────│                              │
    │── ms:connect-transport ──▶│                              │
    │── ms:produce (video) ────▶│── ms:new-producer ──────────▶│
    │                           │◀─ ms:consume (video) ────────│
    │                           │── consumerParams ───────────▶│
    │                           │◀─ ms:resume-consumer ────────│
    │◀──────────── RTP/RTCP (UDP) ─────────────────────────────│
```

The server **never decodes media** — it only routes RTP packets between producers and consumers.

### Worker Pool (`mediasoup/manager.js`)

```
numWorkers = process.env.MEDIASOUP_NUM_WORKERS || 1

For each worker i:
  rtcMinPort = 40000 + i * portRangePerWorker
  rtcMaxPort = rtcMinPort + portRangePerWorker - 1
  worker = mediasoup.createWorker({ rtcMinPort, rtcMaxPort })
```

- Workers are **C++ processes** forked from the Node.js process
- Each worker gets a dedicated **RTP port range** to prevent binding conflicts
- Worker selection: **least-loaded round-robin** (`getNextWorker()`)
- Worker crash → automatic respawn after 2-second cooldown

### Codec Configuration (`mediasoup/config.js`)

| Codec | Type | Notes |
|-------|------|-------|
| Opus | Audio | 48kHz, stereo |
| VP8 | Video | Default, wide browser support |
| VP9 | Video | Profile 2, better compression |
| H.264 | Video | Profile 4d0032, hardware acceleration |

### Simulcast

Three spatial layers for adaptive bitrate:

| Layer | Max Bitrate | Scale | FPS |
|-------|------------|-------|-----|
| `r0` (low) | 150 Kbps | 4× downscale | 15 |
| `r1` (medium) | 500 Kbps | 2× downscale | 24 |
| `r2` (high) | 1.5 Mbps | native | 30 |

### Socket Events (`mediasoup.handlers.js`)

| Event (Client → Server) | Action |
|------------------------|--------|
| `ms:join-room` | Create or get router for roomId; return RTP capabilities |
| `ms:create-transport` | Create WebRTC transport; return `transportId`, ICE params, DTLS params |
| `ms:connect-transport` | Complete DTLS handshake with client's fingerprint |
| `ms:produce` | Create Producer (one per media track); notify other participants |
| `ms:consume` | Create Consumer for a remote producer; return consumer params |
| `ms:resume-consumer` | Un-pause consumer (required after create) |
| `ms:leave-room` | Remove participant; close transports; notify others |

| Event (Server → Client) | Meaning |
|------------------------|---------|
| `ms:new-producer` | A new participant started producing |
| `ms:producer-closed` | A producer was closed (participant left/muted) |

### Graceful Degradation

Mediasoup initialization is wrapped in a try/catch in `server.js`. If the worker binary fails (binary incompatibility, missing deps), the server starts normally with all other features operational. Video/audio calls will fail at the socket handler level with an error event to the client.

---

## 3. Mesh Mode — Room Signaling

### Architecture

```
Browser A                  server (room.socket.js)           Browser B
    │── room:join ────────────▶│◀─── room:join ──────────────│
    │                          │──── room:peer-joined ───────▶│
    │◀─── room:peer-joined ────│                              │
    │                          │                              │
    │── room:signal ───────────▶────── room:signal ──────────▶│
    │ (SDP offer/answer)        │   (forwarded verbatim)      │
    │── room:ice-candidate ────▶│──── room:ice-candidate ────▶│
    │                           │                             │
    │◀═══════════ Direct P2P WebRTC connection ═══════════════│
```

The server acts as a **pure signaling relay** — SDP and ICE candidates are forwarded between peers. Media flows **directly browser-to-browser** (no server-side media processing).

### When Mesh Is Used

- Ad-hoc peer rooms (`/room/:roomCode`) for small groups
- No server media processing cost
- Degrades with >4 participants due to upload bandwidth requirements

### Room State (`Room` model)

```js
{
  roomCode: String,        // unique 8-char code
  title: String,
  host: ObjectId,          // User ref
  settings: {
    passcode: String,      // optional entry passcode
    waitingRoom: Boolean,
    muteOnEntry: Boolean,
    videoOffOnEntry: Boolean,
    allowScreenShare: Boolean,
    allowWhiteboard: Boolean,
    allowChat: Boolean,
  },
  participants: [ObjectId],
  isActive: Boolean,
  type: 'instant' | 'personal' | 'scheduled',
}
```

### REST API (`room.routes.js`)

| Method | Path | Action |
|--------|------|--------|
| `POST` | `/api/rooms/instant` | Create instant room (with wizard settings) |
| `GET/POST` | `/api/rooms/personal` | Get or create personal room |
| `POST` | `/api/rooms/join` | Join room by code + optional passcode |
| `GET` | `/api/rooms/:roomCode` | Get room details |
| `PATCH` | `/api/rooms/:roomCode/settings` | Update room settings |
| `DELETE` | `/api/rooms/:roomCode` | End/close room |

---

## 4. Call History (`call.socket.js` + `call.service.js`)

Separate from the media layer — tracks call **signaling events** for history:

| Event | Description |
|-------|-------------|
| `call:initiate` | Caller starts call → save `Call` doc, ring recipient |
| `call:accept` | Recipient accepts → update `Call.status = 'accepted'` |
| `call:reject` | Recipient rejects → update `Call.status = 'rejected'` |
| `call:end` | Either party ends → record duration |
| `call:missed` | Timeout → update status |

The `Call` model records: caller, receiver, type (audio/video/room), status, duration, timestamps.

---

## 5. Known Limitations

| Limitation | Impact | Mitigation |
|-----------|--------|-----------|
| Mediasoup workers fail on Render free tier kernel | Video calling broken | `MEDIASOUP_SKIP_WORKER_PREBUILT_DOWNLOAD=true` + `npm rebuild mediasoup` from source |
| No TURN server configured | Mesh connections fail behind symmetric NAT | Add Twilio TURN or coturn in production |
| Single Mediasoup worker on Render | CPU bottleneck at scale | Scale up Render instance; set `MEDIASOUP_NUM_WORKERS` |
| `ANNOUNCED_IP` defaults to 127.0.0.1 | WebRTC transport unreachable in production | Set `ANNOUNCED_IP` env var to Render public IP |
