# ADR-004: JWT Token Rotation with Family Invalidation and Double-Submit CSRF

## Status
Accepted

## Context
Pulse requires stateless, horizontal-ready API authentication while protecting against common web vulnerabilities including token theft, Cross-Site Scripting (XSS), Cross-Site Request Forgery (CSRF), and token replay attacks.

## Decision
Implement a multi-layered defense-in-depth authentication model:
1. **Short-Lived Access Token:** Signed JWT with 15-minute expiration passed in the `Authorization: Bearer <token>` header (or socket handshake `auth.token`).
2. **Long-Lived Refresh Token in `httpOnly` Cookie:** Signed JWT with 7-day expiration stored in a `SameSite: Strict` / `Lax` (configurable for cross-origin), `Secure`, `httpOnly` cookie inaccessible to client-side JavaScript.
3. **Token Rotation & Token Families (`tokenFamily.service.js`):** Every refresh request invalidates the previous refresh token and issues a new pair. If an expired or already-used refresh token is presented, the entire "token family" is revoked immediately, neutralizing stolen credentials.
4. **Double-Submit CSRF Cookie:** State-changing HTTP endpoints (`POST`, `PUT`, `DELETE`, `PATCH`) require an `X-CSRF-Token` header that matches the value in the encrypted CSRF cookie (`csrfMiddleware.js`).
5. **Brute Force & Lockout:** Exponential backoff and temporary account locking (`accountLock.js`) after repeated failed credential attempts.

## Consequences
### Positive
- Even if an access token is compromised, the window of vulnerability is limited to 15 minutes.
- Refresh tokens cannot be stolen via simple XSS because they reside in `httpOnly` cookies.
- Replayed tokens trigger immediate family revocation, alerting the user and terminating all associated active sessions.
- Fully compliant with modern OWASP session management guidelines.

### Negative / Tradeoffs
- Requires synchronization logic on the frontend (`axios.js` response interceptor) to queue pending API calls while refreshing the token.
- Strict CSRF checking requires an extra round-trip to `/api/csrf-token` on unauthenticated client cold starts.
