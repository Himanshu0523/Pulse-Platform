# ADR-005: In-Memory Message Write Buffer Pipeline with Periodic Flush

## Status
Accepted

## Context
In high-frequency real-time messaging, persisting every incoming chat message synchronously to MongoDB on receipt creates high I/O latency, exhausts database connection pools, and degrades Socket.IO broadcast speeds for active conversation channels.

## Decision
Implement a decoupled write pipeline:
1. **Immediate In-Memory Fan-Out:** When a `message:send` socket event is validated, the message is immediately broadcast to the target Socket.IO room (`conversation:${id}`) so participants perceive zero-latency communication.
2. **Buffering (`messageBuffer.service.js`):** The validated message payload is queued into an in-memory batch buffer.
3. **Scheduled Batch Persistence (`messageBuffer.job.js`):** Every 3 seconds, a background job extracts all queued messages and executes a single bulk write (`Message.insertMany(...)`) to MongoDB Atlas.
4. **Graceful Shutdown Flush (`server.js`):** On receiving `SIGTERM` / `SIGINT`, the shutdown hook forces a complete write-flush of all pending buffered messages before closing the database connection and exiting the process.

## Consequences
### Positive
- Sub-millisecond broadcast responsiveness for active chat channels.
- Drastically reduces database write operations by batching dozens of simultaneous messages into single multi-document inserts.
- Protects MongoDB from connection pool starvation during activity spikes.

### Negative / Tradeoffs
- Potential data loss window of up to 3 seconds if the Node.js process experiences an abrupt, unhandled fatal crash (e.g. `SIGKILL` or out-of-memory kernel termination). Graceful shutdowns mitigate this for normal deployments and restarts.
