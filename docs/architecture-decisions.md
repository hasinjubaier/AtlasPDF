# AtlasPDF: Architecture Decision Records (ADRs)

**Project:** AtlasPDF (Universal File-to-PDF Conversion Platform)  
**Document Status:** Approved Baseline (Revision 1.1)  
**Date:** 29 October 2026  

---

## Index of Architecture Decision Records

- [ADR-001: Adoption of Next.js App Router and TypeScript for Unified Experience and Control Plane](#adr-001-adoption-of-nextjs-app-router-and-typescript-for-unified-experience-and-control-plane)
- [ADR-002: Neon Serverless PostgreSQL and Drizzle ORM for Metadata and State Persistence](#adr-002-neon-serverless-postgresql-and-drizzle-orm-for-metadata-and-state-persistence)
- [ADR-003: Cloudflare R2 Private Object Storage, Presigned PUT URL Reuse Caveats, and Abuse Controls](#adr-003-cloudflare-r2-private-object-storage-presigned-put-url-reuse-caveats-and-abuse-controls)
- [ADR-004: Inngest for Serverless Durable Background Execution, Bounded Retries, and Retention Reconciliation](#adr-004-inngest-for-serverless-durable-background-execution-bounded-retries-and-retention-reconciliation)
- [ADR-005: Conversion Provider Abstraction and ConvertAPI as Initial Conversion Subprocessor](#adr-005-conversion-provider-abstraction-and-convertapi-as-initial-conversion-subprocessor)
- [ADR-006: Elimination of Local .NET SDK, Docker Desktop, and Local PostgreSQL Server](#adr-006-elimination-of-local-net-sdk-docker-desktop-and-local-postgresql-server)
- [ADR-007: Cryptographic Guest Session Model and Anti-IDOR Authorization Enforcement](#adr-007-cryptographic-guest-session-model-and-anti-idor-authorization-enforcement)
- [ADR-008: Multi-Location Storage Retention Breakdown and 60-Minute Cleanup Initiation Target](#adr-008-multi-location-storage-retention-breakdown-and-60-minute-cleanup-initiation-target)
- [ADR-009: URL-to-PDF Conversion Feature Gating and SSRF Prevention](#adr-009-url-to-pdf-conversion-feature-gating-and-ssrf-prevention)
- [ADR-010: Database Migration Strategy and Zero-Downtime Schema Evolutions](#adr-010-database-migration-strategy-and-zero-downtime-schema-evolutions)
- [ADR-011: Serverless Node.js Runtime Specification and Prohibition of Edge Runtime for Data Operations](#adr-011-serverless-nodejs-runtime-specification-and-prohibition-of-edge-runtime-for-data-operations)
- [ADR-012: Four-Tier Format Support Evidence Lifecycle](#adr-012-four-tier-format-support-evidence-lifecycle)
- [ADR-013: Production Startup Guard for Mock Conversion Provider Safety](#adr-013-production-startup-guard-for-mock-conversion-provider-safety)

---

## ADR-001: Adoption of Next.js App Router and TypeScript for Unified Experience and Control Plane

### Status
**Accepted**

### Context & Problem Statement
The preliminary SRS contemplated an ASP.NET Core backend coupled with a separate Next.js web application. Maintaining two distinct runtimes (.NET CLR and Node.js/V8) introduces cognitive load, dual build pipelines, disparate deployment topologies, and heavy local development prerequisites (such as the .NET SDK). AtlasPDF requires a fast, responsive UI, high-performance edge routing, type-safe API endpoints, and rapid iteration.

### Decision Drivers
- Developer ergonomics and unified TypeScript codebase across frontend and API.
- Zero requirement for local .NET SDK or C# compilers.
- Seamless deployment to Vercel with API route handlers and server actions.
- First-class support for React 19, server components, and streaming UI.

### Decision Outcome
Chosen option: **Unified Next.js App Router with TypeScript**.
The API routes (`/api/v1/*`) serve as the control plane for job creation, upload authorization, progress queries, and download dispatch. The frontend leverages React Server Components and client-side hooks for drag-and-drop queues, truthful progress stages, and PDF previews.

---

## ADR-002: Neon Serverless PostgreSQL and Drizzle ORM for Metadata and State Persistence

### Status
**Accepted**

### Context & Problem Statement
AtlasPDF requires a relational database to store job metadata, file attributes, quota ledgers, and audit logs. The system must support serverless scale-to-zero during quiet periods, connection pooling under high concurrency, fast ephemeral branches for testing, and zero requirement for running a local PostgreSQL Docker container or Windows service.

### Decision Drivers
- Serverless architecture compatible with Vercel serverless execution.
- No local database daemon installation required for developers.
- Type-safe query building and automated lightweight migrations.
- Ephemeral database branching for CI/CD pipelines.

### Decision Outcome
Chosen option: **Neon Serverless PostgreSQL with Drizzle ORM**.
Drizzle ORM defines tables, relations, and enums in TypeScript (`src/db/schema.ts`). Migrations are generated into versioned SQL files and applied via `drizzle-kit migrate`.

---

## ADR-003: Cloudflare R2 Private Object Storage, Presigned PUT URL Reuse Caveats, and Abuse Controls

### Status
**Accepted**

### Context & Problem Statement
Direct-to-storage presigned uploads avoid memory bottlenecks and bypass Vercel serverless function body size limitations (4.5 MB). However, standard S3/Cloudflare R2 presigned PUT URLs remain cryptographically valid until expiration and **cannot be treated as one-time tickets**. Furthermore, server-side `HeadObject` size verification only detects oversized files **after** they have reached storage. Robust abuse defenses are required to prevent storage exhaustion and unauthorized data replays.

### Decision Drivers
- Support uploads up to 20 MB without hitting Vercel payload limits.
- Mitigate replay and reuse risks inherent to presigned PUT URLs.
- Prevent storage exhaustion caused by oversized uploads or quota circumvention.
- Zero egress fees for cost predictability (BG-05).

### Decision Outcome
Chosen option: **Cloudflare R2 with Presigned PUT URLs and Multi-Layer Abuse Controls**.
1. **Unique Object Keys per Ticket:** Every upload ticket generates a fresh, random key:
   `inputs/{jobId}/{fileId}-{randomHex16}.{sanitizedExt}`.
2. **Short Expiration Window:** Presigned PUT URLs expire in **10 minutes**.
3. **Pre-Allocation Quota Checks:** Daily quotas and batch limits are reserved in Neon before issuing tickets.
4. **Post-Upload HEAD Verification & Immediate Purge:** The control plane issues a `HeadObject` command on R2 during `complete-upload`. Because this detects oversized files after storage arrival, any object exceeding the 20 MB limit is immediately deleted via `DeleteObjectCommand`, the job is marked `LIMIT_EXCEEDED`, and an abuse penalty counter is recorded.
5. **Additional Abuse Controls:**
   - IP-based rate limiting on upload ticket requests (max 10/min).
   - Content-Type header matching the requested format family.
   - Cloudflare WAF protection against volumetric request flooding.

---

## ADR-004: Inngest for Serverless Durable Background Execution, Bounded Retries, and Retention Reconciliation

### Status
**Accepted**

### Context & Problem Statement
Conversion jobs are asynchronous and require durable orchestration: step execution, bounded retries on transient errors, graceful cancellation, and reliable file cleanup. Running container-based message queues requires dedicated infrastructure and continuous maintenance.

### Decision Drivers
- Serverless event-driven architecture with zero long-running worker servers.
- Durable step functions with automatic state persistence and bounded retry/backoff.
- Built-in durable delayed sleep (`step.sleepUntil`) to trigger the 60-minute cleanup target.
- Reliable hourly reconciliation cron to guarantee zero orphaned storage objects.

### Decision Outcome
Chosen option: **Inngest**.
- The Next.js API dispatches `atlaspdf/job.created` to Inngest.
- Inngest invokes `processConversionJob`, orchestrating input inspection, conversion dispatch, output validation, and database updates.
- **Retry Policy:**
  - Deterministic failures (unsupported formats, corrupt files, password protection) fail immediately with zero retries.
  - Transient failures (network drops, upstream 502/503) retry up to 3 times with exponential backoff (2s, 6s, 18s).
- **Cleanup & Reconciliation:**
  - Inngest schedules a delayed timer (`step.sleepUntil`) executing at `terminalAt + 60 minutes`.
  - If storage deletion encounters a transient error, Inngest retries up to 5 times.
  - An Inngest scheduled cron function (`reconcileStorageRetention`) runs hourly to detect and purge any orphaned artifacts older than 75 minutes.

---

## ADR-005: Conversion Provider Abstraction and ConvertAPI as Initial Conversion Subprocessor

### Status
**Accepted**

### Context & Problem Statement
Self-hosting headless LibreOffice, unpaper, and Tesseract OCR inside custom Docker containers requires substantial operational overhead and violates the project constraint of avoiding Docker Desktop. An external SaaS subprocessor (ConvertAPI) provides immediate, high-fidelity conversion capabilities but requires explicit privacy disclosures, regional routing, and lifecycle controls.

### Decision Drivers
- Rapid delivery of MVP with high-fidelity Office, image, and data conversion.
- Compliance with requirement of no local Docker Desktop.
- Explicit compliance with privacy and data protection principles.
- Decoupled provider interface to allow future alternative engines.

### Decision Outcome
Chosen option: **`IConversionProvider` abstraction with `ConvertApiProvider`**.
- Core orchestrator interacts only with `IConversionProvider` interface (`supports`, `convert`, `deleteProviderFile`, `getHealth`).
- **ConvertAPI Subprocessor Disclosures & Controls:**
  - Files transferred to ConvertAPI are processed under strict enterprise agreements (SOC 2 Type 2, ISO 27001).
  - ConvertAPI terms guarantee that user files are never used to train AI models.
  - ConvertAPI caches files ephemerally for a maximum of **3 hours**.
  - Where ConvertAPI returns a `FileId`, AtlasPDF issues a `DELETE https://{region}.convertapi.com/d/{FileId}` request upon job completion (best-effort deletion).
  - Regional endpoints (`eu-v2.convertapi.com`, `us-v2.convertapi.com`) are configured to keep processing within approved jurisdictions.
  - Macro execution is strictly disabled on all Microsoft Office formats.
- A `MockConversionProvider` is provided for offline testing, protected by a production startup guard.

---

## ADR-006: Elimination of Local .NET SDK, Docker Desktop, and Local PostgreSQL Server

### Status
**Accepted**

### Context & Problem Statement
Engineering guidelines explicitly forbid mandatory local installations of the .NET SDK, Docker Desktop, or a local PostgreSQL database server. All development and testing workflows must be executable in a clean, lightweight Node.js/TypeScript environment.

### Decision Drivers
- Developer onboarding simplicity (clone repository, `npm install`, `npm run dev`).
- Low resource consumption on developer workstations.
- Uniformity across developer OS environments (Windows, macOS, Linux).

### Decision Outcome
- **No .NET:** Unified Next.js App Router and TypeScript.
- **No Docker Desktop:** All local testing relies on pure Node.js in-memory mocks (`aws-sdk-client-mock`, Inngest test SDK, Mock Conversion Provider) and cloud-based dev branches.
- **No Local PostgreSQL Daemon:** Database development connects securely to Neon serverless cloud branches using connection pooling over HTTPS/WebSockets, or in-memory SQLite/pg-mem for isolated unit tests.

---

## ADR-007: Cryptographic Guest Session Model and Anti-IDOR Authorization Enforcement

### Status
**Accepted**

### Context & Problem Statement
The SRS mandates guest conversion without mandatory account registration (FR-041), while simultaneously requiring strict multi-tenant data isolation, download permission checks (FR-017), and prevention of cross-tenant data leaks (NFR-SEC-01, AT-10). Unguessable UUIDs alone are explicitly prohibited as a substitute for authorization.

### Decision Outcome
Chosen option: **Cryptographically Signed Guest Session Cookies**.
- On first visit, the server issues an HTTP-only, SameSite=Lax, Secure cookie containing an HMAC-SHA256 signed `session_id`.
- When a job is initialized, `owner_ref = session_id` is recorded in Neon.
- Every API endpoint modifying or reading job state (`/upload-ticket`, `/complete-upload`, `/download`, `/cancel`, `/delete`) compares the request's verified `session_id` with `job.owner_ref`.
- If mismatched, the server returns `404 Not Found` (to avoid leaking job existence).

---

## ADR-008: Multi-Location Storage Retention Breakdown and 60-Minute Cleanup Initiation Target

### Status
**Accepted**

### Context & Problem Statement
Requirement NFR-PRV-01 establishes a privacy-first principle of temporary processing. However, claiming an unqualified guarantee that "all files are deleted within exactly 60 minutes" is technically misleading because network failures, third-party provider caches, and asynchronous queue lags can affect absolute millisecond-level guarantees. Transparent engineering requires defining a precise operational cleanup target, documenting retention across all four storage environments, and providing multiple layers of deletion defense.

### Decision Drivers
- Truthful product communication and legal compliance.
- Transparent distinction between primary R2 storage, subprocessor caches, and metadata logs.
- Defense-in-depth against orphaned storage objects.

### Decision Outcome
Chosen option: **60-Minute Cleanup Initiation Target with Multi-Location Retention Lifecycle**.
1. **Cloudflare R2 Original Uploads (`inputs/*`):** Targeted for deletion initiation at 60 minutes after job termination. Executed via Inngest with retries.
2. **Cloudflare R2 Converted Outputs (`outputs/*`):** Targeted for deletion initiation at 60 minutes. Presigned download URLs expire in 5 minutes.
3. **ConvertAPI File Server Cache:** Ephemeral subprocessor cache purged automatically by ConvertAPI after max 3 hours; AtlasPDF executes best-effort deletion via `DELETE /d/{FileId}` where supported.
4. **Neon Database Metadata & Logs:** Metadata retained for 30 days for quota/abuse tracking (sanitized display names only; zero file contents); purged via daily cron. Logs retained for 7–14 days with strict redaction.
5. **Immediate User Deletion ("Delete Now"):** Immediately revokes access in Neon and enqueues immediate R2 and ConvertAPI deletion calls.
6. **Secondary Safety Net:** Cloudflare R2 bucket lifecycle rule permanently expires and shreds all objects older than 24 hours.

---

## ADR-009: URL-to-PDF Conversion Feature Gating and SSRF Prevention

### Status
**Accepted**

### Context & Problem Statement
Requirement FR-036 states: *"Public URL to PDF SHALL remain feature-flagged until URL parsing, DNS/IP enforcement, redirect handling, protocol restrictions and sandbox egress tests pass."* Server-Side Request Forgery (SSRF) represents a critical security risk where arbitrary URL fetching could access cloud metadata endpoints or private VPC assets.

### Decision Outcome
Chosen option: **URL-to-PDF is explicitly DISABLED by default for initial MVP release**.
- The public web application hides and disables any URL input fields.
- API Route handlers reject any conversion request specifying `inputType: "url"` with `403 Forbidden` (`code: URL_BLOCKED`).
- Future activation requires connection-time DNS socket re-validation, private IP range blocks, protocol allowlisting, and independent penetration testing.

---

## ADR-010: Database Migration Strategy and Zero-Downtime Schema Evolutions

### Status
**Accepted**

### Context & Problem Statement
Production database changes must occur without application downtime, schema locking, or deployment failures. The migration pipeline must be fully automated, testable, and capable of rolling back code without breaking the database.

### Decision Outcome
Chosen option: **Drizzle Kit SQL Migrations with Additive-Only Evolution Rules**.
- Schema changes are defined in `src/db/schema.ts` and compiled to pure SQL files in `drizzle/migrations/` using `npx drizzle-kit generate`.
- Migrations are applied in CI/CD via `npx drizzle-kit migrate` prior to deploying new Vercel serverless functions.
- All new columns must be `nullable` or provide default values. No destructive column renames or drops occur in the same release.

---

## ADR-011: Serverless Node.js Runtime Specification and Prohibition of Edge Runtime for Data Operations

### Status
**Accepted**

### Context & Problem Statement
Vercel supports both the Edge Runtime and the Node.js Serverless Runtime. While Edge functions offer low startup latency, they enforce severe execution constraints: lack of standard Node.js `Buffer` and `Stream` implementations, incompatible cryptographic APIs for certain AWS SDK v3 signing procedures, absence of native TCP socket pooling for Neon PostgreSQL, and restricted maximum execution timeouts (typically 25–30 seconds).

### Decision Drivers
- Complete compatibility with AWS SDK v3 S3 client (`@aws-sdk/client-s3`).
- Reliable buffer manipulation for chunked magic byte signature verification.
- Native connection pooling with Neon PostgreSQL.
- Avoidance of obscure runtime incompatibilities and silent failure modes.

### Decision Outcome
Chosen option: **Enforce Node.js Runtime (`nodejs20.x` or `nodejs22.x`) on All Data and Storage Route Handlers**.
- All route handlers under `/api/v1/*` and the Inngest handler `/api/inngest` must declare:
  `export const runtime = 'nodejs';`
- The Edge runtime is strictly prohibited for any route handler that connects to Neon, signs R2 URLs, or handles binary data.

---

## ADR-012: Four-Tier Format Support Evidence Lifecycle

### Status
**Accepted**

### Context & Problem Statement
To eliminate misleading "universal converter" marketing claims (BG-03, BR-01), the system must enforce a verifiable admission pipeline before any format is advertised to users or enabled in the UI.

### Decision Outcome
Chosen option: **Four-Tier Format Support Lifecycle**.
Every format in the Format Registry must progress sequentially through four gates:
1. `Planned`: Schema and options defined in code.
2. `Provider capability confirmed`: Vendor official documentation verifies format support.
3. `Integration tested`: Automated end-to-end integration tests pass on representative fixtures.
4. `Approved for production`: Golden-file regression corpus passes visual and structural quality checks.

Public-facing API endpoints (`GET /api/v1/formats`) only return formats whose status is `Approved for production`.

---

## ADR-013: Production Startup Guard for Mock Conversion Provider Safety

### Status
**Accepted**

### Context & Problem Statement
A `MockConversionProvider` is required for offline testing and CI workflows. However, accidentally deploying a mock provider to production would result in generating synthetic fake PDFs for real users, violating data integrity and creating massive user confusion.

### Decision Outcome
Chosen option: **Enforce a Production Startup Assertion Guard**.
During application bootstrapping (`src/lib/conversion/factory.ts`):
```typescript
if (process.env.NODE_ENV === "production" && process.env.CONVERSION_PROVIDER === "mock") {
  throw new Error("FATAL: MockConversionProvider is strictly prohibited in production environments.");
}
```
If misconfigured, the deployment fails fast during container/function cold start, preventing any mock-generated outputs from reaching production users.
