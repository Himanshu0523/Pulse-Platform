# Integration Architecture

> **Scope:** Webhook receivers, provider adapters, credential management, event bus/queues, and routing rules.

---

## 1. Overview

Pulse provides an enterprise-grade integration framework allowing external systems (GitHub, Sentry, Email, Custom Webhooks) to interact with workspaces, channels, and conversations.

The integration pipeline is built around **idempotent event ingestion**, **secure credential storage (AES-256)**, **rule-based message formatting & routing**, and **background worker execution via BullMQ**.

```
Inbound Webhook / Email / Event
              │
              ▼
  Express Ingress Controller (`/api/webhooks/*`)
              │
              ├── 1. Signature Verification (HMAC / Secret / Token)
              ├── 2. Idempotency Check (`IntegrationEvent.externalEventId`)
              │
              ▼
     Integration Pipeline Engine (`integrations/core/`)
              │
              ├── Normalize Payload into Canonical Event
              ├── Match `IntegrationRoutingRule` (Filter, Channel mapping)
              │
              ▼
     BullMQ Integration Queue (`integrationQueues.js`)
              │
              ▼
     Background Worker (`integrationWorker.js`)
              │
              ├── Format Message / Notification
              ├── Dispatch to Channel via `chat.socket.js` / `Message` store
              └── Record `IntegrationEvent.status = 'processed'`
```

---

## 2. Integration Catalog & Schemas

The system defines 6 core models supporting integrations:

| Model | Purpose | Key Fields |
|-------|---------|------------|
| `IntegrationCatalog` | Global registry of supported integrations | `slug`, `name`, `authType` (oauth2/webhook/api_key), `capabilities` |
| `WorkspaceIntegration` | Instance of an integration configured in a workspace | `workspaceId`, `catalogId`, `enabled`, `config`, `secretToken` |
| `IntegrationRoutingRule` | Filters & actions routing events to channels | `workspaceIntegrationId`, `eventType`, `filters`, `targetChannelId` |
| `IntegrationEvent` | Audit trail of incoming events (idempotency key) | `externalEventId`, `provider`, `payload`, `status`, `receivedAt` |
| `UserIntegrationCredential` | Per-user OAuth credentials (encrypted) | `userId`, `provider`, `accessToken` (AES-256-GCM), `refreshToken` |
| `ChannelEmailAddress` | Unique incoming email addresses mapped to channels | `channelId`, `emailToken`, `active` |

---

## 3. Supported Integrations

### 1. GitHub (`integrations/github/`)
- **Features:** Webhook event handling (commits, PRs, issues, releases), OAuth2 linking for user accounts.
- **Verification:** HMAC SHA-256 signature verification via `X-Hub-Signature-256`.
- **Actions:** Automated rich-embed messages posted to designated channels, direct issue creation via chat action buttons.

### 2. Sentry (`integrations/sentry/`)
- **Features:** Real-time error alerts and issue threshold notifications.
- **Verification:** Sentry webhook secret signature verification.
- **Actions:** Posts formatted alert cards with stack trace summaries, error frequency, and direct deep links.

### 3. Email Bridge (`integrations/email/`)
- **Features:** Inbound email-to-channel routing via dedicated channel aliases (e.g. `channel+token@mail.pulse.domain`).
- **Processing:** Parses MIME headers, extracts body/attachments, maps to `Message` model with attachment uploads to Cloudflare R2 / Cloudinary.

### 4. Generic Custom Webhooks (`integrations/generic/`)
- **Features:** User-definable JSON webhook ingestion.
- **Processing:** Configurable mapping rules and template strings translating arbitrary incoming JSON schemas into rich markdown chat messages.

---

## 4. Security & Isolation

1. **Credential Encryption:** All OAuth access tokens, refresh tokens, and webhook secrets stored in `UserIntegrationCredential` or `WorkspaceIntegration` are encrypted at rest using **AES-256-GCM** with IV and authentication tags.
2. **Idempotency:** Ingress controllers check `IntegrationEvent.findOne({ externalEventId, provider })`. Duplicate webhooks are acknowledged (`200 OK`) and ignored to prevent replay loops.
3. **Workspace Isolation:** All routing rules enforce workspace tenancy (`workspaceId`). Events cannot target channels outside their originating workspace.
4. **Rate Limiting:** Webhook endpoints carry isolated rate limits in `rateLimit.js` to protect against upstream flooding or DoS attacks.
