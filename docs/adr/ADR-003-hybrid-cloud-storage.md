# ADR-003: Hybrid Storage Strategy (Cloudinary for Media, Cloudflare R2 for Files)

## Status
Accepted

## Context
A real-time chat application handles two primary categories of uploaded content:
1. **User Media (Avatars, UI images, photo attachments):** Demands instant thumbnail generation, face-centering, responsive breakpoints, and automated format transcoding (WebP/AVIF).
2. **General File Attachments (Documents, archives, binaries, videos):** Demands high storage volume, fast upload/download throughput, and zero bandwidth egress penalties.

Single-provider solutions present trade-offs: Cloudinary bandwidth and raw storage costs become prohibitively expensive for large binary files; while AWS S3 / Cloudflare R2 lack native real-time image transformation and smart cropping pipelines.

## Decision
Adopt a **hybrid storage tiering model**:
- **Cloudinary:** Used exclusively for user avatars, workspace icons, and image attachments requiring real-time transforms and CDN optimization.
- **Cloudflare R2 (S3 API):** Used for large attachments, PDFs, documents, audio recordings, and raw files, taking advantage of Cloudflare's $0 egress fee policy.

## Consequences
### Positive
- Optimal cost efficiency: zero bandwidth egress fees for heavy user file downloads.
- Superior media UX: images automatically delivered in optimal next-gen formats (WebP/AVIF) sized for the client's screen.
- Clear separation of concerns in file processing pipelines.

### Negative / Tradeoffs
- Requires maintaining two storage SDKs/credentials (`cloudinary.js` and S3 client for R2).
- The `File` model must track the active storage provider (`provider: 'cloudinary' | 'cloudflare-r2'`).
