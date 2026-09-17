# ADR-002: Socket.IO Transport with Redis Pub/Sub Adapter for Horizontal Scaling

## Status
Accepted

## Context
Pulse requires low-latency real-time bidirectional communication for messaging, typing indicators, presence, call signaling, and canvas sync.
The solution must handle intermittent mobile disconnections gracefully and allow multiple backend Node.js processes to share socket rooms without sticky session coupling.

## Decision
1. **Transport:** Use Socket.IO (v4) with transport fallback configured as `['polling', 'websocket']`. Polling establishes the initial handshake reliably behind CDNs and firewalls, followed by WebSocket upgrade.
2. **State Recovery:** Enable native Socket.IO Connection State Recovery (`maxDisconnectionDuration: 180000ms`) with fast-path re-authentication to replay missed events during momentary network blips.
3. **Cluster Scalability:** Integrate `@socket.io/redis-adapter` backed by `ioredis`. When broadcasting to `user:${userId}` or `conversation:${id}`, events publish to Redis channels so clients connected across any backend instance receive them.

## Consequences
### Positive
- Transparent recovery on transient network drops; avoids massive database queries on client reconnect.
- Horizontal scaling capability is built-in; new backend nodes can be spun up without changing client connection logic.
- Reliable fallback mechanism for environments blocking raw WebSockets.

### Negative / Tradeoffs
- Requires a persistent Redis instance for pub/sub (handled via Redis Cloud).
- Slightly higher memory footprint than bare WebSocket (`ws`) library.
