# 🌿 Git Branching Strategy & Workflow Guide

To maintain a clean, reproducible, and scalable codebase across all development phases, Pulse Chat Platform enforces the following branch lifecycle:

---

## 🔀 Branch Taxonomy

```
main (Production)
 │
 └── develop (Staging / Active Integration)
      │
      ├── feature/xyz-realtime-voice
      ├── feature/jwt-rotation
      ├── fix/webrtc-candidate-leak
      ├── release/v1.0.0
      └── hotfix/v1.0.1
```

### Branch Roles & Naming Conventions

| Branch Pattern | Base Branch | Merge Target | Description |
| :--- | :--- | :--- | :--- |
| `main` | - | - | Production-ready, stable, deployed state. Tagged with version releases (e.g. `v1.0.0`). |
| `develop` | `main` | `main` | Main development integration branch. Nightly builds and staging deployments. |
| `feature/*` | `develop` | `develop` | New features (e.g. `feature/channel-email-bridge`, `feature/ai-action-items`). |
| `fix/*` | `develop` | `develop` | Bug fixes for issues identified in `develop`. |
| `release/*` | `develop` | `main` & `develop` | Release stabilization branches (e.g. `release/v1.0.0`). Only metadata & bug fixes. |
| `hotfix/*` | `main` | `main` & `develop` | Critical production fixes directly branched off `main`. |

---

## 🚀 Development Lifecycle Rules

1. **Never commit directly to `main` or `develop`**. All code changes must go through a Pull Request (PR) with passing CI tests.
2. **Atomic Commits**: Use Conventional Commits formatting:
   - `feat(chat): add thread participant tracking`
   - `fix(webrtc): prevent producer renegotiation race condition`
   - `refactor(auth): isolate 2FA verification to auth domain`
   - `docs(api): update OpenAPI spec for message scheduling`
3. **Branch Protection**:
   - Required status checks (linter, unit tests, env verification).
   - Minimum 1 code review approval for merge into `develop` or `main`.
