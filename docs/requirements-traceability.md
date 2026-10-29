# AtlasPDF: Requirements Traceability Matrix (RTM)

**Document Version:** 1.1  
**Date:** 29 October 2026  
**Status:** Approved Baseline (Revision 1.1)  
**Source Document:** `ATLASPDF-SRS-001` (Version 1.1)  

---

## 1. Overview and Traceability Methodology

This Requirements Traceability Matrix (RTM) establishes a bidirectional mapping between the normative requirements specified in `ATLASPDF-SRS-001`, the approved technical architecture, the codebase modules, and the verification methods.

### Priority Legend
- **P1 (MVP):** Mandatory for initial public release. Must map to an implemented component and passing automated test.
- **P2 (Expansion):** Planned post-MVP expansion following initial stabilization and demand validation.
- **P3 (Specialist):** Requires specialized format engines, licensing reviews, or dedicated feasibility gates.
- **Gated / Deferred:** Intentionally disabled or feature-flagged off for safety (e.g. URL-to-PDF).

---

## 2. Functional Requirements Traceability

### 2.1 File Intake and Identification (FR-001 to FR-009)

| Req ID | Pri | Requirement Description | Architectural Component | Implementation in Approved Stack | Verification / Test Method | Status |
| :--- | :---: | :--- | :--- | :--- | :--- | :---: |
| **FR-001** | P1 | Accept files through file picker and drag-and-drop on supported browsers. | `src/components/upload/Dropzone.tsx` | HTML5 Drag and Drop API + `<input type="file">`, mobile file-chooser compatible. | Playwright desktop & mobile UI tests. | P1 (MVP) |
| **FR-002** | P1 | Allow configurable number of files per batch; display batch limit before upload. | `src/lib/config/limits.ts`, `src/components/upload/FileQueue.tsx` | Maximum 5 files per batch in guest tier; displayed in landing banner and validated client/server. | Server route unit test + UI validation test. | P1 (MVP) |
| **FR-003** | P1 | Enforce per-file, per-batch, and expanded-archive size limits server-side. | `src/app/api/v1/jobs/route.ts`, R2 Presigner | Enforces 20 MB/file, 100 MB/batch; R2 presigned PUT specifies content-length bounds. Post-upload HEAD check verifies actual arrival size. | Integration test with oversized mock payload and ZIP bomb fixtures. | P1 (MVP) |
| **FR-004** | P1 | Inspect file signature/content in addition to extension and client MIME. | `src/lib/format/signature.ts`, `complete-upload` route | Reads initial 4096 bytes via R2 GetObject; verifies magic byte signatures before queuing. | Unit test with renamed/spoofed file fixtures (AT-06). | P1 (MVP) |
| **FR-005** | P1 | Maintain a support catalogue mapping formats to engines, options, limits, caveats. | `src/lib/format/registry.ts`, `GET /api/v1/formats` | Central TypeScript Format Registry with 4-tier evidence status; only returns `Approved for production` formats. | Schema validation test + registry integrity test. | P1 (MVP) |
| **FR-006** | P1 | Identify unsupported, corrupted, empty, password-protected files with safe errors. | `src/lib/format/detector.ts`, Inngest workflow | Safe error codes (`UNSUPPORTED_FORMAT`, `FILE_CORRUPT`, `PASSWORD_PROTECTED`). Immediate deletion of invalid uploaded objects. | Automated adversarial test corpus (AT-07). | P1 (MVP) |
| **FR-007** | P1 | Replace user filenames with generated storage IDs; safely encode display names. | `src/app/api/v1/jobs/route.ts`, R2 storage keys | Random UUID storage keys (`inputs/{jobId}/{fileId}-{hex}`); display names HTML-entity sanitized in DB. | Path traversal test; security audit of R2 keys. | P1 (MVP) |
| **FR-008** | P1 | Allow users to remove a file from queue before conversion begins. | `src/components/upload/FileQueue.tsx` | Client state removes file; cancelled job enqueues R2 cleanup. | Interactive UI test + mock queue test. | P1 (MVP) |
| **FR-009** | P2 | Support importing from approved cloud providers (Google Drive, Dropbox, OneDrive). | `src/lib/cloud-import/*` | Cloud provider OAuth2 integrations with least-privilege tokens. | Integration contract tests. | P2 (Deferred) |

---

### 2.2 Conversion and Job Lifecycle (FR-010 to FR-020)

| Req ID | Pri | Requirement Description | Architectural Component | Implementation in Approved Stack | Verification / Test Method | Status |
| :--- | :---: | :--- | :--- | :--- | :--- | :---: |
| **FR-010** | P1 | Unique job ID; track queued, processing, validating, ready, failed, cancelled, expired. | `src/db/schema.ts` (`jobs`), Inngest workflow | RFC 4122 v4 UUID; PostgreSQL enum `job_state` driven by atomic state transitions. | State machine transition unit & integration tests. | P1 (MVP) |
| **FR-011** | P1 | Dispatch work by detected format and selected profile, not by extension alone. | `src/inngest/functions/conversion.ts` | Dispatches conversion based on verified `job.inputType` (from signature) and format registry. | Route dispatch test with mismatched extension fixtures. | P1 (MVP) |
| **FR-012** | P1 | Expose progress at meaningful stages; SHALL NOT invent precise fake percentage. | `src/lib/types/job.ts`, UI Progress Stage | Stages: `STG_INITIALIZED`, `STG_UPLOADED`, `STG_CONVERTING`, `STG_VALIDATING`, `STG_READY`. | UI announcement test; verify no arbitrary timer percentages. | P1 (MVP) |
| **FR-013** | P1 | Allow cancellation when safe; stop worker execution or terminate isolated job. | `POST /api/v1/jobs/{id}/cancel`, Inngest cancel event | Atomically sets `state = cancelled`; Inngest aborts downstream steps and purges scratch. | Cancellation race condition test (AT-13). | P1 (MVP) |
| **FR-014** | P1 | Retry only transient failures with bounded backoff; deterministic failures no retry. | Inngest function retry configuration | Exponential backoff for HTTP 502/503/network; immediate failure on parse/format errors. | Chaos injection test with mocked transient/deterministic errors. | P1 (MVP) |
| **FR-015** | P1 | Support one PDF per input and merged PDF option for compatible batches. | Inngest batch workflow, `pdf-lib` merger | Per-file PDF output default; optional sequential batch merge via `pdf-lib` preserving UI order. | Batch conversion integration test (AT-04, AT-16). | P1 (MVP) |
| **FR-016** | P1 | Generate unique output artifact; validate readable PDF with >= 1 page. | `src/lib/pdf/validator.ts` | Validates `%PDF-` header, xref trailer, and page count >= 1 using `pdf-lib` structural check. | Output validation test with corrupt PDF fixtures. | P1 (MVP) |
| **FR-017** | P1 | Permit download only to job owner/authorized session using short-lived signed access. | `GET /api/v1/jobs/{id}/download`, R2 Presigner | Validates `session_id == job.owner_ref`; issues 5-minute presigned GET URL. | Cross-session IDOR penetration test (AT-10, AT-17). | P1 (MVP) |
| **FR-018** | P1 | Provide preview and download after success without requiring account creation. | `src/components/preview/PdfViewer.tsx` | In-browser PDF preview via secure iframe/PDF.js and download button for guest sessions. | End-to-end guest conversion test. | P1 (MVP) |
| **FR-019** | P1 | Safe retry action reuses valid settings and never duplicates quota consumption. | `POST /api/v1/jobs/{id}/retry` | Re-runs conversion step without deducting additional conversion quota. | Quota ledger idempotent retry test. | P1 (MVP) |
| **FR-020** | P2 | Webhooks and API-based job creation after authentication and tenant isolation. | `src/app/api/v1/developer/*` | Signed HMAC webhooks and API keys for registered users. | Webhook replay & signature verification test. | P2 (Deferred) |

---

### 2.3 PDF Output Controls (FR-021 to FR-030)

| Req ID | Pri | Requirement Description | Architectural Component | Implementation in Approved Stack | Verification / Test Method | Status |
| :--- | :---: | :--- | :--- | :--- | :--- | :---: |
| **FR-021** | P1 | Sensible default profile preserving source layout where supported. | `src/lib/conversion/profiles.ts` | Default profiles tuned for standard A4/Letter margins and layout preservation. | Golden-file visual regression tests. | P1 (MVP) |
| **FR-022** | P1 | Expose page size (A4, Letter, auto), orientation, margins when supported. | `src/components/settings/OptionsDrawer.tsx` | Options drawer exposes page size, orientation (portrait/landscape), margins. | Form validation test + ConvertAPI parameter test. | P1 (MVP) |
| **FR-023** | P1 | Spreadsheet settings (sheets, print area, scaling) and image ordering/orientation. | `src/components/settings/SpreadsheetOptions.tsx` | Parameters mapped to ConvertAPI Excel options and image sequencing pipeline. | XLSX sheet selection test (AT-02). | P1 (MVP) |
| **FR-024** | P1 | Image fit mode (contain/cover/fit-to-page) and resolution preset for raster images. | `src/components/settings/ImageOptions.tsx` | Options mapped to ConvertAPI image-to-PDF pipeline. | Image fit and resolution test (AT-04). | P1 (MVP) |
| **FR-025** | P1 | Common page-range or slide-range controls when supported. | `src/components/settings/OptionsDrawer.tsx` | Parses page range string (e.g. `1-5, 8`) and passes to conversion adapter. | Slide range test for PPTX (AT-03). | P1 (MVP) |
| **FR-026** | P1 | Preserve Unicode text using embedded fonts; disclose detectable substitutions. | ConvertAPI adapter font configuration | Full UTF-8 support; font embedding enabled; warnings emitted if fallback font used. | Multilingual Unicode fixture test (Arabic, CJK, Cyrillic). | P1 (MVP) |
| **FR-027** | P1 | Optional OCR for supported image inputs and scans with language selection. | ConvertAPI OCR adapter | Exposes OCR toggle, language selection (eng, deu, fra, spa); outputs searchable text layer. | OCR searchable text layer test. | P1 (MVP) |
| **FR-028** | P1 | Safe defaults for animated/multi-frame images; explain frame handling. | Image adapter frame extractor | GIF/TIFF defaults to all frames as separate pages or first frame with user notice. | Multi-frame TIFF and GIF fixture test. | P1 (MVP) |
| **FR-029** | P2 | Offer PDF/A archival output profiles after conformance testing. | PDF/A conversion profile | Validates PDF/A compliance via VeraPDF or engine conformance. | PDF/A conformance check. | P2 (Deferred) |
| **FR-030** | P2 | Accessible PDF output improvements (document title, language, semantic tags). | Tagged PDF pipeline | Generates tagged structural trees in PDF. | PDF accessibility inspection test. | P2 (Deferred) |

---

### 2.4 Format-Specific Capabilities (FR-031 to FR-040)

| Req ID | Pri | Requirement Description | Architectural Component | Implementation in Approved Stack | Verification / Test Method | Status |
| :--- | :---: | :--- | :--- | :--- | :--- | :---: |
| **FR-031** | P1 | Office documents & presentations render through controlled engine with font handling. | `ConvertApiProvider` (Office Engine) | Office engine with disabled macros; pinned engine versions and standard fonts. | Golden-file tests for DOCX, PPTX (AT-01, AT-03). | P1 (MVP) |
| **FR-032** | P1 | Spreadsheets support sheet selection, fit/scaling; warn on table overflow. | `ConvertApiProvider` (Excel Engine) | Sheet selection parameter, fit-to-width; UI warning for wide sheets. | Multi-sheet XLSX test (AT-02). | P1 (MVP) |
| **FR-033** | P1 | CSV, JSON, XML, TXT rendered as readable content with line wrapping, headers. | Text & Data Converter Pipeline | Formatted table rendering for CSV; formatted monospaced view with line wrapping for JSON/XML. | Long-line overflow test (AT-05). | P1 (MVP) |
| **FR-034** | P1 | Markdown and source code rendered in document layout with monospaced font. | Code/Markdown Converter Pipeline | Syntax-highlighted HTML rendered to PDF; monospaced font, line numbers optional. | Code listing formatting test. | P1 (MVP) |
| **FR-035** | P1 | HTML uploads render with local resources only by default; no internal network access. | HTML Converter Pipeline | Content Security Policy restricting remote network fetches during HTML rendering. | HTML SSRF test with external/internal links. | P1 (MVP) |
| **FR-036** | P1 | Public URL to PDF SHALL remain feature-flagged until SSRF security review passes. | System Feature Flag (`ENABLE_URL_CONVERSION`) | **Disabled by default in MVP.** UI hides URL inputs; API returns 403 `URL_BLOCKED`. | Verification that URL conversion requests are rejected (AT-09). | Gated (OFF) |
| **FR-037** | P1 | ZIP conversion lists supported entries, deterministic sort, rejects unsafe paths. | `src/lib/archive/zip.ts` | Zip streaming parser rejects `..`, enforces max 50 entries, 100 MB expanded, 10:1 ratio. | Path traversal & zip bomb test (AT-08). | P1 (MVP) |
| **FR-038** | P2 | EPUB, email formats (EML/MSG), vector formats (SVG), and HEIC images. | Specialized format adapters | Defer to Phase 5 expansion after licensing and conformance tests. | Specialized format tests. | P2 (Deferred) |
| **FR-039** | P3 | Audio/video sources produce transcript or metadata sheet only (not static conversion). | Media extraction pipeline | Defer to Phase 5 specialist pipeline. | Extraction workflow test. | P3 (Deferred) |
| **FR-040** | P3 | CAD, 3D, and proprietary formats require separately approved commercial engine. | Specialist engine bridge | Defer to Phase 5 specialist pipeline. | Commercial engine audit. | P3 (Deferred) |

---

### 2.5 Privacy, Accounts and Administration (FR-041 to FR-050)

| Req ID | Pri | Requirement Description | Architectural Component | Implementation in Approved Stack | Verification / Test Method | Status |
| :--- | :---: | :--- | :--- | :--- | :--- | :---: |
| **FR-041** | P1 | Support guest conversion without mandatory account creation. | `src/lib/auth/session.ts` | Cryptographic guest session cookies; no registration required for core conversion. | End-to-end guest journey test. | P1 (MVP) |
| **FR-042** | P1 | Display file retention & location before upload; offer immediate delete action. | Landing UI & `POST /api/v1/jobs/{id}/delete` | Pre-upload privacy badge ("60m cleanup target"); "Delete files now" button in UI. | Immediate deletion test (AT-11). | P1 (MVP) |
| **FR-043** | P1 | Target initiating deletion of inputs and outputs within 60 minutes of terminal state. | Inngest function `cleanupExpiredJob` | Durable timer triggers R2 object deletion and updates DB `state = expired` at 60m. Daily reconciliation audits storage. | Retention automated test (AT-12). | P1 (MVP) |
| **FR-044** | P1 | Operational logs exclude file contents, passwords, tokens, raw signed URLs. | Structured Logger (`src/lib/logger.ts`) | Strict redaction interceptor; filenames omitted/sanitized in telemetry. | Log inspection test against security audit requirements. | P1 (MVP) |
| **FR-045** | P1 | Configurable quotas for guest, registered, API users (size, batch, daily, OCR). | `src/lib/config/limits.ts`, `quota_ledgers` | Central quota configuration validated in job creation middleware. | Quota exhaustion test. | P1 (MVP) |
| **FR-046** | P1 | Admin ability to enable/disable format or engine without full app release. | Neon `format_registry` / Feature Flags | Dynamic database-backed format registry flags (`isEnabled: boolean`). | Admin flag toggle test. | P1 (MVP) |
| **FR-047** | P1 | Administrators see job outcome, duration, engine version, safe error code (no file content). | Admin API (`GET /api/v1/admin/metrics`) | Metrics aggregation query displaying anonymized job stats without file data. | Admin view privacy audit. | P1 (MVP) |
| **FR-048** | P2 | Authenticated accounts, subscriptions, invoices, and usage history. | Auth0 / Supabase Auth / Stripe | Deferred to post-MVP monetization phase. | Billing integration test. | P2 (Deferred) |
| **FR-049** | P2 | Organization-level access controls, audit logs, and regional processing. | Enterprise tenant management | Deferred to enterprise tier. | RBAC permission test. | P2 (Deferred) |
| **FR-050** | P2 | User-selected persistent workspaces with clear retention controls. | Persistent workspace module | Deferred to post-MVP phase. | Workspace retention test. | P2 (Deferred) |

---

### 2.6 Accessibility and Help (FR-051 to FR-055)

| Req ID | Pri | Requirement Description | Architectural Component | Implementation in Approved Stack | Verification / Test Method | Status |
| :--- | :---: | :--- | :--- | :--- | :--- | :---: |
| **FR-051** | P1 | Keyboard-accessible actions, semantic labels, focus states, accessible status announcements. | UI Components, `aria-live` regions | Accessible Radix UI primitives, visible focus rings, live status updates. | axe-core automated audit + keyboard navigation test (AT-15). | P1 (MVP) |
| **FR-052** | P1 | Error messages name the problem and next action; no internal paths or stack traces. | Safe Error Mapper (`src/lib/errors.ts`) | Maps all internal errors to safe public error messages and support codes. | Error message audit against OWASP disclosure rules. | P1 (MVP) |
| **FR-053** | P1 | UI describes format support and limitations in plain language. | `src/app/supported-formats/page.tsx` | Dedicated formats page powered directly by active format registry data. | Content verification test against format registry. | P1 (MVP) |
| **FR-054** | P1 | Responsive layouts, respect reduced-motion preferences. | Tailwind CSS (`motion-reduce:*`) | Mobile-first layout; disables animations when `prefers-reduced-motion` is active. | Viewport resizing & accessibility media query test. | P1 (MVP) |
| **FR-055** | P2 | Translated UI and OCR language packs based on user demand. | i18n localization module | Deferred to expansion phase. | Multi-language translation test. | P2 (Deferred) |

---

## 3. Non-Functional Requirements Traceability

| Req ID | Category | Requirement Description | Stack Implementation | Verification Method | Status |
| :--- | :--- | :--- | :--- | :--- | :---: |
| **NFR-PERF-01** | Performance | p95 conversion <= 30s for common-office files <= 20 MB / 20 pages. | ConvertAPI optimized SaaS connection + Cloudflare R2 direct transfer. | Automated benchmark suite on golden corpus. | P1 (MVP) |
| **NFR-PERF-02** | Performance | API metadata/status requests p95 < 500 ms. | Vercel Node.js serverless functions + Neon connection pooling + indexing. | Synthetic distributed tracing & load test. | P1 (MVP) |
| **NFR-PERF-03** | Capacity | Sustain 100 concurrent conversion jobs without leakage or queue stall. | Inngest queue concurrency limits (bounded to 50 parallel steps). | Load & queue saturation stress test. | P1 (MVP) |
| **NFR-PERF-04** | Capacity | Large uploads support resumable or multipart upload where supported. | Direct R2 presigned PUT (expandable to S3 multipart upload in Phase 2). | Interrupted network upload test. | P1 (MVP) |
| **NFR-PERF-05** | Capacity | Worker CPU, memory, scratch disk, and wall-clock limits enforced. | Inngest step timeouts (90s) + ConvertAPI sandbox resource ceilings. | Resource boundary testing. | P1 (MVP) |
| **NFR-AVL-01** | Availability | Target 99.5% control plane monthly availability during public beta. | Vercel multi-zone serverless Node.js + Neon high availability architecture. | Synthetic uptime monitoring. | P1 (MVP) |
| **NFR-AVL-02** | Recovery | Queue outage or worker failure must not lose accepted job state. | Inngest durable execution logs state; uncompleted jobs retry or fail cleanly. | Chaos injection test (kill worker mid-job) (AT-13). | P1 (MVP) |
| **NFR-AVL-03** | Recovery | Deployments support rollback to known-good API; versioned engine configs. | Vercel 1-click rollback + additive Drizzle migrations + pinned provider versions. | Production rollback drill. | P1 (MVP) |
| **NFR-AVL-04** | Recovery | Files are temporary (no file backups); metadata backup must not restore expired files. | Neon automated backups with metadata filtering; R2 lifecycle handles purge. | Restore test verifying no expired files reactivated. | P1 (MVP) |
| **NFR-AVL-05** | Availability | 99.9% availability & multi-region recovery candidate after scale. | Multi-region DB replication and dual-provider setup. | Architecture review post-MVP. | P2 (Expansion) |
| **NFR-SEC-01** | Security | HTTPS only; private storage objects retrievable only via short-lived signed URLs. | Cloudflare R2 private bucket; TLS 1.3 enforced; 5-minute presigned GET URLs. | SSL Labs test + private bucket penetration test. | P1 (MVP) |
| **NFR-SEC-02** | Security | Validate file signature; enforce presigned PUT reuse defenses & abuse controls. | Unique keys per ticket (`inputs/{jobId}/{fileId}-{hex}`), 10m TTL, post-upload HEAD verification, immediate deletion of oversized objects, and IP velocity caps. | Presigned URL replay & oversized upload penetration test. | P1 (MVP) |
| **NFR-SEC-03** | Security | Parse files in isolated sandbox (non-root, read-only FS, restricted egress). | ConvertAPI isolated SaaS container sandbox; no local parser execution. | Sandboxing audit of provider infrastructure. | P1 (MVP) |
| **NFR-SEC-04** | Security | Inspect file signatures (magic bytes); explicit fail-closed policy. | Magic byte validation; immediate deletion and rejection of mismatched uploads. | Malware test fixture simulation. | P1 (MVP) |
| **NFR-SEC-05** | Security | Generated storage names; mitigate path traversal, archive bombs, XXE. | Random UUID keys; path normalization; zip streaming limits; disable XML external entities. | OWASP Top 10 automated security scanner. | P1 (MVP) |
| **NFR-SEC-06** | Security | Strict SSRF defense for URL rendering; connection-time DNS check. | **URL conversion disabled in MVP.** Feature gate strictly returns 403. | Security penetration test (AT-09). | Gated (OFF) |
| **NFR-SEC-07** | Security | Managed secret store; no secrets in code, logs, client, or job metadata. | Vercel Secrets / Environment Variables; strict logger redaction filters. | Git secret scanner (TruffleHog) + log audit. | P1 (MVP) |
| **NFR-SEC-08** | Security | Least privilege, CSRF protections, rate limits, bot abuse controls. | SameSite cookies, Cloudflare bot protection, IP rate limiting. | Security review & penetration test. | P1 (MVP) |
| **NFR-SEC-09** | Security | Security events and admin changes auditable without recording file content. | `audit_events` table in Neon; structured JSON logs with sanitized actor/action. | Audit log inspection. | P1 (MVP) |
| **NFR-SEC-10** | Security | Independent threat model & penetration test before public launch. | Formal architectural threat model (STRIDE) + penetration test review. | Pen-test sign-off record. | P1 (MVP) |
| **NFR-PRV-01** | Privacy | Target initiating deletion within 60 minutes of terminal state; durable retries. | Inngest delayed cleanup step + hourly reconciliation cron + R2 24h bucket lifecycle rule. | Automated 60-minute retention test (AT-12). | P1 (MVP) |
| **NFR-PRV-02** | Privacy | "Delete now" action immediately revokes access and enqueues physical purge. | `POST /api/v1/jobs/{id}/delete` sets state `expired`, purges R2 objects. | Immediate deletion test (AT-11). | P1 (MVP) |
| **NFR-PRV-03** | Privacy | Separate documented retention schedules for artifacts, logs, and accounts. | Artifacts: 60m target; Logs: 7–14 days anonymized; Metadata: 30 days anonymized. | Documentation and configuration audit. | P1 (MVP) |
| **NFR-PRV-04** | Privacy | Do not use submitted files to train AI models; subprocessor terms verified. | ConvertAPI terms guarantee zero AI model training; zero file persistence beyond 3h. | Provider contract and privacy audit. | P1 (MVP) |
| **NFR-PRV-05** | Privacy | Privacy notice describes region, subprocessors, retention, user rights. | Transparent `/privacy` page detailing Neon, Cloudflare, ConvertAPI, Vercel. | Legal and privacy team review. | P1 (MVP) |
| **NFR-PRV-06** | Privacy | Legal review maps obligations for launch markets; no false certifications. | Honest privacy copy adhering to Section 11.2 guardrails. | Legal compliance sign-off. | P1 (MVP) |
| **NFR-UX-01** | Usability | First-time user completes conversion without account or tutorial. | Streamlined 2-step flow: Drop file -> Click Convert -> Preview/Download. | Moderated usability testing (>=90% target). | P1 (MVP) |
| **NFR-UX-02** | Usability | Landing screen communicates supported formats, limits, privacy before upload. | Prominent format links, 20 MB size limit note, 60-minute cleanup target badge. | Usability visual inspection. | P1 (MVP) |
| **NFR-UX-03** | Usability | Mobile layout supports file chooser, queue, download without hover controls. | Touch-friendly controls, visible tap targets (min 44x44px), drawer UI. | Mobile device testing (iOS Safari, Android Chrome). | P1 (MVP) |
| **NFR-A11Y-01**| Accessibility | Target WCAG 2.2 AA for web UI using automated + screen-reader tests. | Semantic HTML, accessible color contrast (4.5:1), Radix UI accessible dialogs. | axe-core automated audit + manual NVDA test. | P1 (MVP) |
| **NFR-A11Y-02**| Accessibility | Focus order, visible labels, status announcements, reduced-motion. | `prefers-reduced-motion` media queries; `aria-live` status regions. | Manual keyboard walkthrough & accessibility audit. | P1 (MVP) |
| **NFR-LOC-01** | Localization | UTF-8 end-to-end; layout tolerates long filenames and Unicode scripts. | Database, API, and UI configured for UTF-8; responsive text truncation. | Internationalization test with long foreign names. | P1 (MVP) |
| **NFR-MNT-01** | Maintainability| Engines hidden behind stable adapter contract (`IConversionProvider`). | TypeScript provider interface decoupling conversion logic from orchestration. | Provider substitution unit test. | P1 (MVP) |
| **NFR-MNT-02** | Maintainability| Conversion engine versions pinned, scanned, and regression tested. | Pinned ConvertAPI engine options; golden corpus visual regression checks. | Engine upgrade test (AT-18). | P1 (MVP) |
| **NFR-MNT-03** | Maintainability| Separation of UI, orchestration, file storage, and rendering workers. | Decoupled Next.js UI, Neon DB, Inngest jobs, R2 storage, ConvertAPI adapter. | Architectural modularity review. | P1 (MVP) |
| **NFR-MNT-04** | Maintainability| CI/CD includes unit, integration, contract, security, regression tests. | GitHub Actions CI matrix running Vitest, lint, typecheck, Playwright. | CI build verification on PRs. | P1 (MVP) |
| **NFR-MNT-05** | Maintainability| Production startup guard rejects mock conversion provider in production. | Server startup assertion in `src/lib/conversion/factory.ts`. | Cold-start startup guard unit test. | P1 (MVP) |
| **NFR-OBS-01** | Observability | Metrics and traces correlate API request and job via opaque IDs; no file data. | Structured JSON logger with opaque UUIDs and sanitized tags. | Telemetry audit. | P1 (MVP) |

---

## 4. Acceptance Test Scenarios Traceability (AT-01 to AT-18)

| Test ID | Scenario Description | Required Pass Condition | Implementation & Test Location |
| :--- | :--- | :--- | :--- |
| **AT-01** | Valid DOCX with tables, headers, footer, mixed fonts converts to PDF. | PDF opens; expected page count/layout verified; warnings for font substitution. | `tests/golden/docx-conversion.test.ts` |
| **AT-02** | XLSX with multiple sheets submitted. | User can select sheets; PDF page scaling matches selection; printed output readable. | `tests/golden/xlsx-conversion.test.ts` |
| **AT-03** | PPTX submitted with slide-range option. | Each selected slide appears in expected order; speaker notes excluded by default. | `tests/golden/pptx-conversion.test.ts` |
| **AT-04** | Multiple JPG/PNG images submitted. | Output page order matches UI queue; rotation/orientation correct; fit matches selection. | `tests/golden/image-conversion.test.ts` |
| **AT-05** | JSON/source code with lines longer than page width. | Output readable and deterministic; long line wrap/overflow follows rules. | `tests/golden/code-conversion.test.ts` |
| **AT-06** | File extension renamed to a different type. | System detects signature mismatch or rejects before conversion; no unsafe dispatch. | `tests/security/type-mismatch.test.ts` |
| **AT-07** | Zero-byte, corrupted, or password-protected file submitted. | Each yields a specific safe error code; no infinite retries; invalid object purged. | `tests/functional/corrupt-files.test.ts` |
| **AT-08** | Archive contains `../` path entries or excessive expanded size. | Archive rejected or unsafe entries excluded per policy; no writes escape scratch. | `tests/security/archive-traversal.test.ts` |
| **AT-09** | URL resolves to localhost/private/metadata IP or redirects there. | URL job is blocked; worker cannot access destination; security event logged safely. | `tests/security/ssrf-rejection.test.ts` |
| **AT-10** | Two different users try to access each other's job/download. | Access denied regardless of guessed job ID or URL tampering (Anti-IDOR). | `tests/security/idor-authorization.test.ts`|
| **AT-11** | User deletes a completed job via "Delete Now". | Download access revoked immediately, deletion queued, final artifact cleanup confirmed. | `tests/privacy/immediate-delete.test.ts` |
| **AT-12** | Failed job reaches retention deadline (60 min target). | Cleanup initiation triggered at 60 minutes; input and output artifacts removed. | `tests/privacy/retention-cleanup.test.ts` |
| **AT-13** | Worker crashes or network drops mid-conversion. | Job reaches failed or retryable state; no false success; scratch artifacts cleaned. | `tests/resilience/worker-failure.test.ts` |
| **AT-14** | Security check or scanner unavailable for high-risk format. | Configured fail-safe policy followed; file is not silently processed. | `tests/security/scanner-failsafe.test.ts` |
| **AT-15** | Screen-reader user uploads and converts a file. | Controls, progress, errors, and download actions announced and operable. | `tests/a11y/screen-reader-flow.test.ts` |
| **AT-16** | Batch contains a mix of supported and unsupported files. | Unsupported files identified; supported files can continue; order and outcomes visible. | `tests/functional/batch-mixed.test.ts` |
| **AT-17** | Expired signed download URL is reused. | Access denied; new authorized link can only be issued while job is still valid. | `tests/security/expired-url.test.ts` |
| **AT-18** | Engine or adapter version upgraded. | Full regression corpus runs; release blocked if quality/security thresholds fail. | `tests/regression/engine-upgrade.test.ts` |
