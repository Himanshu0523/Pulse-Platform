# Security & Authorization Matrix

> **Scope:** Multi-tenant access controls, role hierarchy, resource boundaries, and isolation policies.

---

## 1. Role-Based Access Control (RBAC) Hierarchy

Pulse enforces a strict multi-tenant boundary model. Users belong to workspaces via `WorkspaceMember` records. Roles are scoped to specific workspaces.

```
       [Owner]  (Level 5) — Full tenancy destruction & billing control
          │
       [Admin]  (Level 4) — Workspace settings, member roles, integrations, audits
          │
     [Moderator](Level 3) — Content moderation, article deprecation, room management
          │
       [Member] (Level 2) — Read/write messages, file uploads, initiate calls
          │
        [Guest] (Level 1) — Channel-specific limited guest participation
          │
   [Non-Member User] (Level 0) — Zero tenant access (Isolated)
```

---

## 2. Definitive Authorization Matrix

| Resource | Non-Member User | Member | Moderator | Admin | Owner |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Workspace** | ❌ (Denied) | Read Info | Read Info | Update Settings, Invites | Full Control, Delete Workspace |
| **Chat / DM** | Only if Participant | Full Participation | Full Participation | Full Participation | Full Participation |
| **Channel** | ❌ (Denied) | Join / Read / Send | Create / Update Channel | Delete Channel, Security Mode | Full Control |
| **Message** | ❌ (Denied) | Create, Edit Own | Delete Any Message | Delete Any Message | Full Control |
| **File** | ❌ (Denied) | Upload, Read Allowed | Delete Any In Channel | Delete Any, Quotas | Full Control |
| **Integration** | ❌ (Denied) | View Enabled | View Enabled | Connect, Configure, Route | Full Control |
| **Webhook** | HMAC Required | HMAC Required | HMAC Required | Manage Keys, Disable | Full Control |
| **Search** | Scoped to User DMs | Scoped to Tenant/Convos | Scoped to Tenant/Convos | Scoped to Tenant/Convos | Scoped to Tenant/Convos |
| **AI (Summaries / Replies)** | ❌ (Denied) | In Convos Joined | In Convos Joined | Across Workspace | Full Control |
| **Compliance & Audit** | ❌ (Denied) | ❌ (Denied) | ❌ (Denied) | View Dashboards, Exports | Full Retention Control |

---

## 3. Multi-Tenant Cross-Boundary Isolation ("Tenant A vs Tenant B")

```
   Tenant A User (User A)
             │
             ├── REST API Request ──────────▶ [403 Forbidden]
             ├── Socket.IO (join-chat) ────▶ [chat:error "Not authorized"]
             ├── Socket.IO (workspace) ────▶ [workspace:error "Not a member"]
             ├── WebRTC Room Action ────────▶ [Denied: Host verification failed]
             ├── Integration Rule Query ───▶ [403 Forbidden: Not a member]
             ├── Search Query (messages) ──▶ [403 Forbidden / Restricted to Convos]
             ├── File Download / List ─────▶ [403 Forbidden / Not authorized]
             ├── AI Chat Summarization ────▶ [403 Forbidden / Not a participant]
             └── Admin Dashboard ──────────▶ [403 Forbidden / Admin privileges req]
             │
             ▼
      Tenant B Resource
```

### Layered Enforcement Points

1. **REST API Boundary:**
   - Workspace routes enforce `workspaceMiddleware` (`WorkspaceMember.findOne({ workspace, user })`).
   - Conversation routes enforce `'participants.user': req.user._id`.
   - File downloads verify the requesting user is in the conversation where the file was uploaded.
   - Knowledge Base endpoints require active workspace membership for reads and moderator/admin roles for deprecation or deletion.
   - AI endpoints (`summarizeChat`, `getSmartReplies`, `smartSearch`) strictly assert conversation participant status before processing prompts.

2. **Socket.IO Real-Time Layer:**
   - `workspace:join` queries `WorkspaceMember.findOne` before allowing the client to join `workspace:${id}` broadcast rooms.
   - `join-chat` queries `Conversation.findById` and validates participant lists before socket room membership is granted.
   - `sendMessage` enforces conversation authorization, message deduplication, and E2EE mode verification.

3. **WebRTC Signaling Layer:**
   - Mesh actions (`room:mute`, `room:remove-participant`) require verification of `room.host === userId`.
   - SFU routers isolate RTP traffic per `roomId`.

4. **Integration Layer:**
   - Inbound webhooks require valid cryptographic HMAC signatures (`X-Hub-Signature-256`, Sentry auth headers, or integration tokens).
   - E2EE-only channels prohibit integration ingestion via `validateChannelSecurity.js`.
