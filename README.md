# AtlasPDF: Universal File-to-PDF Conversion Platform

> **A format-aware, privacy-first system for converting supported files and structured data into reliable PDF documents.**

---

## 1. Project Overview

AtlasPDF is a cloud-native document conversion platform built to turn common office files, spreadsheets, presentations, raster images, and structured data into well-formatted, validated PDF documents. 

AtlasPDF is architected on **honest format boundaries**:
- It never makes deceptive "converts any file" or "universal binary converter" claims.
- It operates with an explicit, four-tier support catalog.
- It enforces privacy-by-design with temporary storage and a targeted 60-minute cleanup lifecycle.
- Arbitrary URL-to-PDF conversion is **strictly disabled** for the initial MVP release to eliminate Server-Side Request Forgery (SSRF) risks.

---

## 2. Approved Technology Stack

The platform is designed to run completely serverless, eliminating developer prerequisites for local .NET SDKs, Docker Desktop, or local PostgreSQL instances:

| Component | Selected Technology | Architecture Role |
| :--- | :--- | :--- |
| **Application & Control Plane** | [Next.js](https://nextjs.org/) (App Router) + TypeScript | Responsive web app, UI components, and API Route Handlers running on **Node.js runtime (`nodejs20.x`/`nodejs22.x`)**. |
| **Relational Database** | [Neon](https://neon.tech/) Serverless PostgreSQL | Metadata, job states, quota ledgers, and audit logs with connection pooling and database branching. |
| **ORM & Migrations** | [Drizzle ORM](https://orm.drizzle.team/) | TypeScript-native data modeling and versioned SQL schema migrations. |
| **Private Object Storage** | [Cloudflare R2](https://www.cloudflare.com/developer-platform/r2/) | S3-compatible private storage with zero egress fees, direct presigned uploads, and 24-hour lifecycle safety nets. |
| **Background Orchestration** | [Inngest](https://www.inngest.com/) | Durable event-driven job execution, bounded exponential retries, durable timers, and scheduled reconciliation. |
| **Conversion Provider** | [ConvertAPI](https://www.convertapi.com/) | Pluggable SaaS subprocessor for high-fidelity Office, image, and text rendering (macros disabled, regional endpoints). |
| **Hosting Platform** | [Vercel](https://vercel.com/) | Serverless deployments with atomic releases, instant rollbacks, and preview branch environments. |

---

## 3. Architecture Documentation

Detailed architectural specifications, decisions, and planning documents reside in the [`docs/`](./docs) directory:

- [**`docs/SRS.md`**](./docs/SRS.md): Complete Software Requirements Specification (`ATLASPDF-SRS-001` v1.1) preserving all original requirement IDs (`FR-001` to `FR-055`), non-functional requirements, business rules, and acceptance criteria.
- [**`docs/architecture.md`**](./docs/architecture.md): Deep-dive system architecture, Mermaid topology diagrams, direct presigned upload security, lifecycle state machine, and multi-location retention breakdown.
- [**`docs/architecture-decisions.md`**](./docs/architecture-decisions.md): Architecture Decision Records (`ADR-001` to `ADR-013`) detailing runtime choices, security tradeoffs, and operational guardrails.
- [**`docs/requirements-traceability.md`**](./docs/requirements-traceability.md): Bidirectional Requirements Traceability Matrix (RTM) linking all requirements and acceptance test scenarios (`AT-01` to `AT-18`) to code components.
- [**`docs/implementation-roadmap.md`**](./docs/implementation-roadmap.md): 9-phase implementation delivery plan with clear entry and exit criteria.

---

## 4. Key Architectural & Security Guardrails

### 4.1 Multi-Location Retention Lifecycle
AtlasPDF defines a clear operational target for initiating cleanup:
1. **Cloudflare R2 Inputs (`inputs/*`):** Targeted for deletion initiation at 60 minutes after job termination via Inngest durable workflows.
2. **Cloudflare R2 Outputs (`outputs/*`):** Targeted for deletion initiation at 60 minutes. Presigned download URLs expire in 5 minutes.
3. **ConvertAPI File Server:** Files cached during conversion are purged automatically by ConvertAPI after a maximum of 3 hours. AtlasPDF executes best-effort deletion via `DELETE https://{region}.convertapi.com/d/{FileId}` where supported.
4. **Neon Database Metadata & Logs:** Metadata retained for 30 days for quota/abuse tracking (sanitized display names only; zero file contents); purged via daily cron. Logs retained for 7–14 days with strict redaction.
5. **Immediate User Purge:** The "Delete now" action immediately revokes download permissions and enqueues physical object deletion.
6. **Infrastructure Safety Net:** An immutable Cloudflare R2 bucket lifecycle rule hard-deletes any object older than 24 hours.

### 4.2 Presigned PUT Upload Security & Abuse Controls
Standard S3/R2 presigned PUT URLs remain cryptographically valid until expiration and cannot be treated as one-time tickets. To eliminate replay and abuse risks:
- Every upload ticket generates a unique, non-deterministic object key (`inputs/{jobId}/{fileId}-{randomHex16}.{ext}`).
- Short 10-minute maximum expiration.
- Server-side `HeadObject` verification detects oversized files after storage arrival; any file exceeding 20 MB is immediately deleted via `DeleteObjectCommand` and flagged as `LIMIT_EXCEEDED`.
- Pre-upload quotas and IP velocity limits prevent bucket exhaustion.

### 4.3 Anti-IDOR Authorization
Guest conversion requires no mandatory account registration, but files are isolated using cryptographically signed (HMAC-SHA256) session cookies. Every API endpoint strictly asserts that `request.session_id === job.owner_ref`. Unguessable UUIDs are never treated as a substitute for authorization.

### 4.4 Four-Tier Format Support Lifecycle
Formats must progress through four sequential verification gates:
1. `Planned`
2. `Provider capability confirmed`
3. `Integration tested`
4. `Approved for production` (Only formats in this state are exposed via the public API).

### 4.5 Mock Provider Safety Guard
`MockConversionProvider` is strictly confined to automated test suites and explicitly flagged local environments. A server startup assertion aborts application startup if a mock provider is configured when `NODE_ENV === 'production'`.

---

## 5. Development Guidelines

- **No Local .NET or Docker:** Developers use remote Neon serverless branches and Node.js in-memory test mocks.
- **Node.js Runtime Enforced:** All data and storage route handlers run on `nodejs20.x` or `nodejs22.x`; Vercel Edge Runtime is avoided for database and storage handlers.
- **Additive Migrations:** All schema changes must be backwards-compatible to permit instant code rollbacks.

---

## 6. Project Status

- **Current State:** Architecture documentation complete and approved baseline established.
- **Repository Status:** Fresh directory (Git repository not yet initialized; pending stakeholder authorization).
- **Next Step:** Awaiting final review of corrections before initiating **Phase 1: Project Scaffold, Shared Types, Database Schema & Drizzle Setup**.

