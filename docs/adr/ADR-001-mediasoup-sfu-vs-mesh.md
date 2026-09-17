# ADR-001: Selection of Mediasoup SFU with Mesh WebRTC Fallback

## Status
Accepted

## Context
Pulse requires real-time audio/video conferencing supporting two distinct usage patterns:
1. **Multi-participant video calls (meetings & channels):** Groups ranging from 5 to 30+ simultaneous streams where peer-to-peer mesh architecture fails due to $O(N^2)$ client upload and bandwidth limits.
2. **Ad-hoc quick P2P peer rooms:** Lightweight ad-hoc collaboration between 2-4 users requiring zero server media processing costs.

## Decision
We adopted a **dual-architecture WebRTC strategy**:
- **Mediasoup (v3) as in-process SFU:** An in-process C++ worker pool handles multi-party RTP packet routing without transcoding, supporting simulcast (low/med/high bitrate tiers) and adaptive bandwidth management.
- **Mesh WebRTC Signaling Relay (`room.socket.js`):** Lightweight SDP offer/answer relay for small peer-to-peer rooms (`/room/:code`), avoiding SFU server resource consumption.

## Consequences
### Positive
- SFU enables predictable client bandwidth ($O(N)$ download, $O(1)$ upload per producer).
- Zero third-party video billing (e.g. Agora, LiveKit cloud costs).
- Simulcast provides smooth degradation on poor network connections.
- Peer mesh fallback ensures small rooms continue functioning even if SFU workers fail or encounter kernel constraints.

### Negative / Tradeoffs
- Mediasoup C++ native workers introduce binary dependency requirements (e.g. Python, GCC/Clang during build) which require specific build configurations on platforms like Render (`MEDIASOUP_SKIP_WORKER_PREBUILT_DOWNLOAD=true`).
- UDP port ranges (`40000-49999`) must be accessible and correctly configured with `ANNOUNCED_IP`.
