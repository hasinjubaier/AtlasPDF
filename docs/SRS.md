# AtlasPDF: Universal PDF Conversion Platform
## Software Requirements Specification (SRS)
**A format-aware, privacy-first system for converting supported files and structured data into reliable PDF documents.**

---

### Document Control

| Item | Details |
| :--- | :--- |
| **Document ID** | `ATLASPDF-SRS-001` |
| **Version / Status** | `1.1 / Technical Baseline & Stack Realization` |
| **Prepared Date** | 29 October 2026 |
| **Primary Audience** | Product, engineering, QA, security, design, operations, and business stakeholders |
| **Working Name** | AtlasPDF (subject to trademark and domain review) |
| **Classification** | Confidential working document; not a legal or compliance certification |
| **Change Policy** | Changes to supported formats, retention targets, conversion engines, security controls, or service tiers require an impact review and versioned change log. |
| **Research Method** | Desk research of selected global products using their public official pages and documentation. No exhaustive census of every converter and no controlled performance benchmark was performed. |

#### Revision History
| Version | Date | Change | Owner |
| :--- | :--- | :--- | :--- |
| 0.1 | 29 Oct 2026 | Initial market scan and product requirements draft. | Product / Engineering |
| 1.0 | 29 Oct 2026 | Baseline SRS prepared for stakeholder review. Preserves approved serverless/SaaS architecture stack alignment. | Product / Engineering |
| 1.1 | 29 Oct 2026 | Architectural corrections: refined 60-minute cleanup target, multi-location storage lifecycle breakdown, ConvertAPI subprocessor disclosures, presigned PUT reuse caveats and upload abuse controls, 4-tier format status evidence, Node.js serverless runtime specification, and production startup guard for mock providers. | Engineering Leadership |

---

### Important Scope Note
> **No software can faithfully convert every possible file format into a meaningful PDF.** This specification defines an explicit, testable support catalogue, safe handling for unsupported files, and phased expansion for specialised formats.

---

### Table of Contents
1. [Introduction](#1-introduction)
2. [Product Vision, Goals and Scope](#2-product-vision-goals-and-scope)
3. [Market Research and Competitive Analysis](#3-market-research-and-competitive-analysis)
4. [Users, Personas and Operating Environment](#4-users-personas-and-operating-environment)
5. [Use Cases and User Journeys](#5-use-cases-and-user-journeys)
6. [Functional Requirements](#6-functional-requirements)
7. [Supported Format Catalogue and Conversion Rules](#7-supported-format-catalogue-and-conversion-rules)
8. [Non-Functional Requirements](#8-non-functional-requirements)
9. [System Architecture and Technical Design](#9-system-architecture-and-technical-design)
10. [Data Model and API Requirements](#10-data-model-and-api-requirements)
11. [User Experience Requirements](#11-user-experience-requirements)
12. [Error Handling and Observability](#12-error-handling-and-observability)
13. [Security, Privacy and Compliance Requirements](#13-security-privacy-and-compliance-requirements)
14. [Verification, Validation and Acceptance Criteria](#14-verification-validation-and-acceptance-criteria)
15. [Release Plan and Delivery Roadmap](#15-release-plan-and-delivery-roadmap)
16. [Risks, Dependencies and Open Decisions](#16-risks-dependencies-and-open-decisions)
17. [Success Metrics](#17-success-metrics)
- [Appendix A. Glossary](#appendix-a-glossary)
- [Appendix B. Research Sources](#appendix-b-research-sources)
- [Appendix C. Release Readiness Checklist](#appendix-c-release-readiness-checklist)

---

## 1. Introduction

### 1.1 Purpose
This SRS defines what AtlasPDF must do, how it should behave, how quality will be measured, and what engineering must verify before release. It is intended to be a shared contract for product, design, engineering, QA, security, and operations.

### 1.2 Product Summary
AtlasPDF is a browser-first conversion service that turns supported documents, spreadsheets, presentations, images, web pages, and structured text/data into readable PDFs. It identifies likely file types, offers relevant output settings, runs a format-specific conversion pipeline, validates the result, and provides a short-lived download.

### 1.3 Scope Boundary
**Core product promise:** Convert supported inputs into a useful PDF with predictable layout, clear status, practical error messages, and transparent file retention. Unknown or unsupported files must be declined safely rather than falsely advertised as convertible.

### 1.4 Language and Requirement Terms
| Term | Meaning |
| :--- | :--- |
| **MUST / SHALL** | Mandatory requirement for the stated release unless explicitly marked Future. |
| **SHOULD** | Recommended requirement; deviations must be documented and justified. |
| **MAY** | Optional capability. |
| **P1 / MVP** | Required for the first public release. |
| **P2 / Expansion** | Planned after the MVP is stable and demand is validated. |
| **P3 / Specialist** | Requires format-specific engines, licensing, or a separate feasibility gate. |
| **Conversion engine** | The isolated parser/rendering tool or service adapter responsible for a class of formats. |
| **Job** | A conversion request and its associated state, inputs, configuration, and output artifacts. |
| **Structured data**| CSV, JSON, XML, YAML, and similar text formats rendered as legible tables or formatted text. |

### 1.5 References and Research Date
Public product pages and vendor documentation were reviewed on 29 October 2026 where available. Findings represent publicly documented positioning, not an independent security audit, exhaustive feature audit, or laboratory test. Full source list appears in [Appendix B](#appendix-b-research-sources).

---

## 2. Product Vision, Goals and Scope

### 2.1 Vision
Make PDF creation from everyday work files and useful data formats feel like one reliable workflow, without forcing users to learn which separate converter they need.

### 2.2 Product Principles
- **Format-aware by design:** Detect the actual file type and route to the correct renderer; do not depend on filename extension alone.
- **Honest support:** Show supported extensions and known limitations before conversion. Never promise universal conversion for arbitrary binary or proprietary data.
- **Privacy by default:** No permanent file library in the MVP. Enforce a target of initiating file cleanup within 60 minutes of job termination, with durable retries and an infrastructure lifecycle safety net. Provide visible deletion controls and maintain minimal, sanitized operational logging.
- **Fidelity where practical:** Preserve layout, fonts, tables, page order, image orientation, and spreadsheet print settings as far as the selected engine allows.
- **Progressive disclosure:** Basic conversion should take one or two decisions; advanced settings appear only when relevant.
- **Graceful failure:** Explain the reason, suggest a fix, and preserve retry options when safe.

### 2.3 Business Goals
| Goal ID | Goal | Measure / Initial Target |
| :--- | :--- | :--- |
| **BG-01** | Make common files convert with little effort. | At least 90% task completion in moderated usability tests for common DOCX, XLSX, PPTX, and image workflows. |
| **BG-02** | Build user trust around files. | Retention policy visible before upload; 100% of terminal jobs target cleanup initiation at 60 minutes, backed by daily reconciliation checks and an automated R2 lifecycle safety net. |
| **BG-03** | Handle common formats accurately. | At least 98% successful conversion in the approved regression corpus for supported, valid test files; quality reviewed separately. |
| **BG-04** | Keep failures understandable. | Every terminal failure maps to a documented error category and user-safe message. |
| **BG-05** | Support a sustainable service. | Per-engine cost and CPU time measured; configurable quotas and plan limits available before paid growth. |

### 2.4 In Scope for MVP (P1)
- Responsive web application/PWA with guest conversion and optional account sign-in.
- Single-file and batch upload, drag-and-drop, file-type detection, support matrix, and conversion options.
- Office documents, spreadsheets, and presentations; common raster images; plain text, Markdown, CSV, JSON, XML, and source-code listings.
- ZIP archive intake for a bounded set of supported files, with entry selection and predictable output ordering.
- Truthful progress stage reporting, cancellation where safe, retry, output preview, download, and clear failure messages.
- Optional OCR for scanned images and image-only documents, initially with a small, tested language set.
- Secure temporary private storage in Cloudflare R2, malware and signature inspection, isolated subprocessor workers, targeted 60-minute cleanup initiation, and operational metrics.
- HTML upload conversion rendering with local resources only by default.

### 2.5 Out of Scope for MVP
- Arbitrary URL-to-PDF conversion (disabled by default until SSRF security review and network egress gating pass).
- Faithful conversion of every proprietary design, CAD, 3D, engineering, or scientific binary format.
- Playback-preserving audio/video conversion into a static PDF (future media features may create metadata sheets or transcripts only as explicit workflows).
- Conversion of an arbitrary executable, encrypted container, or unknown binary into a meaningful document.
- Full PDF editor, e-signature, PDF-to-Office reconstruction, permanent cloud drive, team workspace, and AI chat with files.
- Guaranteeing identical results to Microsoft Office for every complex document before the supported corpus proves it.
- Claiming legal, privacy, or security certifications before a formal scope-specific audit.

---

## 3. Market Research and Competitive Analysis

### 3.1 Method and Limitations
The market contains PDF suites, general file-conversion platforms, native office exporters, and offline utilities. This review compares representative products with broad international visibility based on official pages. Public claims can change, vary by region or plan, and do not establish equal output quality. We did not test every product, every plan, or every supported conversion pair.

### 3.2 Competitive Snapshot
| Product / Category | Publicly Documented Scope | Strengths / Observed Positioning | Product Lesson |
| :--- | :--- | :--- | :--- |
| **Adobe Acrobat Online / Desktop** | Common Office, image, and document formats; desktop has a wider format table. Online converter covers major everyday types [R01][R02]. | Known brand, mature PDF workflow, emphasis on layout; limitations linked to format, app, font, and feature support. | List supported formats by route and explain when advanced formatting may change. |
| **Smallpdf** | PDF suite with Office, image, HTML, text, OpenDocument, EPUB; OCR and PDF/A listed [R03][R04]. | Easy conversion journey and adjacent PDF tools. Public converter describes 20+ formats. | Keep default flow short, then offer useful PDF actions after conversion without making them prerequisites. |
| **iLovePDF** | Office-to-PDF plus broad PDF actions; web, mobile, desktop paid offerings. Basic tier has limited processing [R05]. | Freemium multi-device suite with clear plan limits. | Design quotas as explicit product configuration, not hard-coded scattered limits. |
| **CloudConvert** | Conversion API with 200+ formats, advanced options, async jobs, webhooks, storage integrations [R06]. | Useful for format diversity and custom workflows. Privacy page states deletion <= 24 hours [R07]. | Use engine-adapter architecture, job queue, webhooks, and configurable format options. Aim for shorter published retention. |
| **Convertio** | Advertises 300+ formats and 25,600+ pairs; input deleted after conversion, output retained up to 24h [R09][R10]. | Broad web conversion with custom options for many conversion types. | Show live support catalogue and state deletion window in plain language. |
| **Zamzar** | Advertises 1,100+/1,200+ formats across doc, image, video, audio, ebook, CAD [R11][R12]. | Large catalogue and API reach, including specialist categories. | Separate "render file as PDF" from "extract metadata/transcript". Do not imply every source type has an equivalent PDF. |
| **PDF24 Creator / Tools** | Free Windows offline app keeps files on PC. Online webpage-to-PDF processes in cloud, deletes output shortly [R13][R14]. | Offline processing is a strong privacy option; web tools offer quick no-install route. | Treat offline/local processing as a credible future differentiator; disclose cloud vs local processing per route. |
| **FreeConvert** | General conversion; privacy page states files stored on AWS Ireland, deleted after 8 hours [R15]. | Clear retention policy and broad conversion categories. | Make deletion controls easy to find and verify them with automated lifecycle tests. |
| **Microsoft Office / LibreOffice** | Native PDF export. MS notes spreadsheet internal links may be lost and font substitution can occur online [R16][R17]. | Often best starting point for fidelity in files created in the same suite; works locally. | Benchmark against native reference output and expose print area, page fit, font, and export warnings. |

### 3.3 What the Research Suggests
- Format count alone is an incomplete quality metric. A platform may support hundreds of formats while complex layouts still require specific engines.
- Privacy practices differ: published deletion windows range from immediate to 8 or 24 hours. AtlasPDF should set a short operational target for initiating cleanup (60 minutes past completion) backed by durable execution.
- Pre-upload expectations matter: scanned PDFs need OCR, animated GIFs need frame handling choices, and audio/video cannot be static pages without explicit extraction workflows.
- Conversion controls should adapt: spreadsheet print-area/fit for XLSX, image order/orientation for images, line-wrap for code.

### 3.4 Suggested Positioning
> **"One clear route from common files and useful data to a clean PDF, with visible support limits, quality checks, and short-lived file storage."** Differentiator is trust, workflow clarity, and measurable format quality.

### 3.5 Feature Opportunity Matrix
| Opportunity | User Value | Delivery |
| :--- | :--- | :--- |
| **Pre-upload support check** | Reduces failed uploads and surprises. | P1: Extension + MIME/signature detection and support catalogue. |
| **Format-aware options** | Improves output without overwhelming users. | P1: Show only relevant controls. |
| **Quality warnings** | Helps users catch layout or content loss. | P1: Per-engine warnings and preview; P2: Automated comparisons. |
| **Short retention + delete button** | Builds confidence with sensitive documents. | P1: 60-minute cleanup initiation target + user deletion + R2 lifecycle safety net. |
| **Offline / local conversion** | Useful for confidential files and low-bandwidth users. | P2: Desktop/local runtime; limited browser-local conversions. |
| **Developer API & webhooks** | Allows teams to automate routine conversion. | P2/P3: After core quality and operations are stable. |
| **Specialist formats** | Serves niche professional workflows. | P3: Subject to business demand, licensing, and proof of concept. |
| **Freemium tiers** | Balances free access with sustainable processing cost. | Configurable caps; test paid options for larger batches, OCR, priority. |

---

## 4. Users, Personas and Operating Environment

### 4.1 User Classes
| Persona | Primary Jobs | Needs / Constraints |
| :--- | :--- | :--- |
| **Student / applicant** | Convert assignments, scanned notes, slides, images. | Fast, low-cost, works on mobile, sensible layout, OCR. |
| **Office user** | Convert reports, spreadsheets, forms, presentations. | Formatting, sheet selection, print area, page numbering, confidentiality. |
| **Developer / analyst** | Turn Markdown, source code, CSV, JSON, XML, or logs into PDF evidence. | Monospace rendering, line wrap, page headers, long-file handling, deterministic output. |
| **Privacy-conscious user** | Convert contracts, ID scans, client files. | Clear retention targets, no public links, minimal logs, no AI training on files. |
| **Small-team administrator**| Set limits, inspect failure rates, manage billing/API later. | Usage visibility, stable queues, cost controls, access separation. |

### 4.2 Operating Environment
- **Client:** Modern desktop and mobile browsers, responsive web experience; PWA installability is desirable but not required.
- **Initial browser support:** Latest two stable versions of Chrome, Edge, Firefox, and Safari, plus current iOS Safari and Android Chrome.
- **Server:** Serverless Node.js runtime (`nodejs20.x` or `nodejs22.x`) on Vercel for the API control plane. **Edge runtime is NOT used** for database queries, S3 storage SDK, or buffer processing to avoid runtime socket and memory limitations.
- **Background Orchestration:** Inngest serverless durable functions for job lifecycle steps and delayed cleanup timers.
- **Third-Party Subprocessor:** ConvertAPI REST API for format conversions, accessed via HTTPS regional endpoints.
- **Network:** HTTPS only for all application and download traffic; Cloudflare R2 object storage is private; time-limited presigned URLs are used for uploads (15 min) and downloads (5 min).
- **Locale:** English at MVP, UTF-8 end-to-end, architecture ready for right-to-left and language-specific OCR expansion.

### 4.3 Assumptions and Dependencies
- Users have legal permission to process submitted files.
- An explicit format catalogue is maintained with extension, signature, parser, engine version, limitations, max size, timeout, and test corpus.
- Subprocessor (ConvertAPI) terms of service and data protection agreements apply; files transferred to ConvertAPI are processed in accordance with documented retention rules.
- Cloud region and storage location reviewed for privacy, latency, and legal obligations.
- Complex Office output fidelity verified with representative test files; file extension coverage does not prove fidelity.

---

## 5. Use Cases and User Journeys

### 5.1 UC-01: Convert a Single Document
1. User opens the converter and selects a file or drags it into the upload area.
2. Client and server inspect size, extension, MIME hints, and file signature. The system identifies a supported type or explains why it is unsupported.
3. System presents source-aware options (page size, orientation, spreadsheet print settings).
4. User starts conversion. A job ID is created and the UI displays a truthful state: `uploading`, `queued`, `processing`, `validating`, `ready`, or `failed`.
5. The conversion adapter sends the file to ConvertAPI in an isolated execution sandbox, receives the output PDF, validates it, and stores the result in private R2 storage.
6. User previews, downloads, deletes, or retries. The system targets initiating cleanup of inputs and outputs at 60 minutes after job termination.

### 5.2 UC-02: Batch Conversion
The user selects multiple files. The system reports which files are supported, allows per-file settings where needed, and offers either one PDF per input or a single merged PDF only when all selected inputs can be rendered. Output order follows the visible queue order. A single file failure must not silently corrupt successful outputs.

### 5.3 UC-03: Convert a ZIP Archive
The user uploads a ZIP. The system lists entries without extracting to a public path, calculates archive and expanded-size limits, rejects path traversal and excessive nesting, and lets the user select supported entries. The system does not execute archive contents. Configured policy prevents archive bombs and recursive extraction loops.

### 5.4 UC-04: Convert a Public Web Page
The user pastes a URL. The system validates the URL, blocks internal/private destinations and unsafe redirects, applies DNS/IP checks at connection time, restricts protocols and resource limits, and renders only within a network-isolated browser worker. Failures such as authentication walls, blocked resources, or very long pages are explained. **This feature is disabled until SSRF controls pass security review.**

### 5.5 UC-05: Create a Searchable PDF from a Scan
The user uploads an image or image-only document, selects OCR and an available language, then converts. The result is marked as OCR-processed. The UI warns that recognition may be imperfect, especially for low-resolution, handwritten, or skewed content. Source images remain available only until expiry.

### 5.6 UC-06: Unsupported or Failed File
The system names the reason category without exposing stack traces: unsupported format, damaged file, password-protected/encrypted input, detected malware, resource limit, timeout, or engine failure. It suggests a safe next action and supplies a support code that does not expose file content.

---

## 6. Functional Requirements

### 6.1 File Intake and Identification
| ID | Priority | Requirement | Verification |
| :--- | :--- | :--- | :--- |
| **FR-001** | P1 | The system SHALL accept files through file picker and drag-and-drop on supported browsers. | UI test on desktop and mobile. |
| **FR-002** | P1 | The system SHALL allow a configurable number of files per batch and display the batch limit before upload. | Limit displayed and enforced server-side. |
| **FR-003** | P1 | The system SHALL enforce per-file, per-batch, and expanded-archive size limits server-side. | Oversized and ZIP-bomb fixtures rejected. |
| **FR-004** | P1 | The system SHALL inspect file signature/content in addition to filename extension and client-supplied MIME. | Renamed/mismatched fixtures detected. |
| **FR-005** | P1 | The system SHALL maintain a support catalogue that maps formats to engines, options, limits, and known caveats. | Catalogue powers UI and dispatch. |
| **FR-006** | P1 | The system SHALL identify unsupported, corrupted, empty, and password-protected files before or during safe parsing, then show actionable errors. | Each condition has a stable error code. |
| **FR-007** | P1 | The system SHALL replace user filenames with generated storage identifiers and safely encode the display name. | No path traversal or overwrite from filename. |
| **FR-008** | P1 | The system SHALL allow users to remove a file from the queue before conversion begins. | Removed item is not processed. |
| **FR-009** | P2 | The system SHOULD support importing from approved cloud providers after OAuth scopes and security review. | Tokens least-privilege; import is explicit. |

### 6.2 Conversion and Job Lifecycle
| ID | Priority | Requirement |
| :--- | :--- | :--- |
| **FR-010** | P1 | The system SHALL create a unique job ID and track queued, processing, validating, ready, failed, cancelled, and expired states. |
| **FR-011** | P1 | The system SHALL dispatch work by detected format and selected conversion profile, not by extension alone. |
| **FR-012** | P1 | The system SHALL expose progress at meaningful stages; it SHALL NOT invent a precise percentage when the engine cannot provide one. |
| **FR-013** | P1 | The system SHALL allow cancellation when safe. A cancellation request SHALL stop worker execution or terminate the isolated job within a configured deadline. |
| **FR-014** | P1 | The system SHALL retry only transient failures using bounded backoff and idempotent job keys; deterministic parse failures SHALL NOT be retried indefinitely. |
| **FR-015** | P1 | The system SHALL support one PDF per input and a merged PDF option for compatible batches, with user-visible ordering. |
| **FR-016** | P1 | The system SHALL generate a unique output artifact per job and validate that it is a readable PDF with at least one page unless an approved exception applies. |
| **FR-017** | P1 | The system SHALL permit download only to the job owner/authorized session using short-lived, unguessable or signed access. |
| **FR-018** | P1 | The system SHALL provide a preview and download action after success, without requiring the user to create an account for basic guest conversion. |
| **FR-019** | P1 | The system SHALL provide a safe retry action that reuses valid settings and never duplicates billing/usage consumption for a single accepted job unless policy says otherwise. |
| **FR-020** | P2 | The system SHOULD support webhooks and API-based job creation after authentication, rate limits, and tenant isolation are implemented. |

### 6.3 PDF Output Controls
| ID | Priority | Requirement |
| :--- | :--- | :--- |
| **FR-021** | P1 | The system SHALL provide a sensible default profile and preserve the source layout where supported by the selected engine. |
| **FR-022** | P1 | The system SHALL expose page size (A4, Letter, source/auto where available), orientation, and margins when the engine supports these options. |
| **FR-023** | P1 | The system SHALL provide source-specific settings for spreadsheet sheets/print area/scaling and image ordering/orientation when relevant. |
| **FR-024** | P1 | The system SHALL allow users to select image fit mode (contain/cover or fit-to-page) and image resolution preset for raster-image conversion. |
| **FR-025** | P1 | The system SHALL support common page-range or slide-range controls when supported by the source format and engine. |
| **FR-026** | P1 | The system SHALL preserve Unicode text using embedded or compatible fonts where licensing and engine support permit; substitutions SHALL be disclosed when detectable. |
| **FR-027** | P1 | The system SHALL provide optional OCR for supported image inputs and scanned documents, with language selection and a clear OCR label. |
| **FR-028** | P1 | The system SHALL apply safe defaults for animated or multi-frame image inputs and explain whether frames are preserved as separate pages or a selected frame is used. |
| **FR-029** | P2 | The system SHOULD offer PDF/A output profiles after conformance testing and explain that archival profiles may disable certain features or change file size. |
| **FR-030** | P2 | The system SHOULD support accessible PDF output improvements, including document title/language and semantic tagging when the source and engine support them. |

### 6.4 Format-Specific Capabilities
| ID | Priority | Requirement |
| :--- | :--- | :--- |
| **FR-031** | P1 | Office documents and presentations SHALL render through a controlled document engine with explicit page and font handling. |
| **FR-032** | P1 | Spreadsheets SHALL support sheet selection and a fit/scaling choice; the UI SHALL warn when tables may become too small or span many pages. |
| **FR-033** | P1 | CSV, JSON, XML, and plain text SHALL be rendered as readable content, with line wrapping, page headers where selected, and a readable overflow strategy. |
| **FR-034** | P1 | Markdown and supported source-code files SHALL be rendered in a document layout; source-code mode SHALL use a legible monospaced font and preserve line order. |
| **FR-035** | P1 | HTML uploads SHALL render with local resources only by default; remote resources require a controlled policy and must not access internal networks. |
| **FR-036** | P1 | Public URL to PDF SHALL remain feature-flagged until URL parsing, DNS/IP enforcement, redirect handling, protocol restrictions, and sandbox egress tests pass. |
| **FR-037** | P1 | ZIP conversion SHALL list supported entries, preserve a deterministic sort order, reject unsafe paths, and enforce expanded size/depth/file-count limits. |
| **FR-038** | P2 | The system MAY add EPUB, email formats, vector formats, and additional image types after per-format conformance testing. |
| **FR-039** | P3 | Audio/video sources MAY produce a transcript, metadata sheet, or selected-frame contact sheet only as a distinct extraction workflow, not as an equivalent static conversion. |
| **FR-040** | P3 | CAD, 3D, and specialist proprietary formats SHALL require a separately approved engine and commercial/legal review before support is advertised. |

### 6.5 Privacy, Accounts and Administration
| ID | Priority | Requirement |
| :--- | :--- | :--- |
| **FR-041** | P1 | The system SHALL support guest conversion without mandatory account creation for the advertised guest limits. |
| **FR-042** | P1 | The system SHALL display file retention and processing location information before upload and offer an immediate delete action for each job. |
| **FR-043** | P1 | The system SHALL target initiating deletion of inputs, outputs, and temporary files within 60 minutes of terminal state, with durable retries and reconciliation. |
| **FR-044** | P1 | Operational logs SHALL exclude file contents, passwords, access tokens, and raw signed URLs. File names SHALL be omitted or pseudonymized in logs unless justified. |
| **FR-045** | P1 | The system SHALL provide configurable quotas for guest, registered, and API users, including file size, batch count, daily jobs, and OCR usage. |
| **FR-046** | P1 | Administrators SHALL be able to enable/disable a format or engine, adjust limits, and disable a vulnerable engine without a full application release. |
| **FR-047** | P1 | Administrators SHALL see job outcome, duration, engine version, resource use, and a safe error code, not uploaded file content. |
| **FR-048** | P2 | The system SHOULD support authenticated accounts, subscriptions, invoices, and usage history after privacy and billing decisions are approved. |
| **FR-049** | P2 | The system SHOULD support organisation-level access controls, audit logs, and regional processing options for business plans. |
| **FR-050** | P2 | The system MAY support user-selected persistent workspaces only with clear retention controls and a separate consent/plan model. |

### 6.6 Accessibility and Help
| ID | Priority | Requirement |
| :--- | :--- | :--- |
| **FR-051** | P1 | All primary actions SHALL be keyboard-accessible and expose semantic labels, focus states, and accessible status announcements. |
| **FR-052** | P1 | Error messages SHALL name the problem and a next action; they SHALL not expose internal paths, stack traces, or secrets. |
| **FR-053** | P1 | The UI SHALL describe format support and limitations in plain language, including known output changes. |
| **FR-054** | P1 | The application SHALL support responsive layouts and respect reduced-motion preferences. |
| **FR-055** | P2 | The product SHOULD provide translated UI and OCR language packs based on measured user demand. |

---

## 7. Supported Format Catalogue and Conversion Rules

### 7.1 Proposed MVP Format Matrix & Support Status
Every format is assigned an explicit four-tier support status:
1. **Planned:** In scope for design; engine parameters and schema drafted.
2. **Provider capability confirmed:** Subprocessor / conversion engine documentation verifies conversion capability.
3. **Integration tested:** Automated end-to-end integration test passes on representative samples.
4. **Approved for production:** Golden-file regression corpus passes; caveats documented; ready for public release.

| Category | Extensions | Status | PDF Behavior | Engine & Notes |
| :--- | :--- | :---: | :--- | :--- |
| **Word Processing** | DOCX, DOC, RTF, ODT | **Provider capability confirmed** | Convert to paginated PDF. | ConvertAPI Office Engine. Macros disabled. Font embedding enabled. |
| **Spreadsheets** | XLSX, XLS, ODS, CSV | **Provider capability confirmed** | PDF with sheet selection and fit-to-page options. CSV rendered as formatted table. | ConvertAPI Excel Engine. Document formula recalculation caveats. |
| **Presentations** | PPTX, PPT, ODP | **Provider capability confirmed** | Slides as pages; slide range selectable. | ConvertAPI PowerPoint Engine. Speaker notes excluded by default. |
| **Raster Images** | JPG, JPEG, PNG, TIFF, BMP, WebP | **Provider capability confirmed** | Paginated PDF; respect EXIF orientation; configurable fit. | ConvertAPI Image Engine. Multi-frame TIFF/GIF options. |
| **Text & Markup** | TXT, MD, HTML | **Provider capability confirmed** | Paginated document layout with line wrapping. | ConvertAPI / Headless engine. HTML limited to local resources. |
| **Structured Data** | JSON, XML | **Provider capability confirmed** | Prettified formatted text with line wrapping. | Text engine. Long lines wrapped to avoid clipping. |
| **Source Code & Logs**| LOG, and allowlisted code extensions | **Planned** | Monospaced font layout with line numbering. | Requires code syntax highlighting integration tests. |
| **ZIP Archive** | ZIP | **Planned** | Extracts allowlisted entries and merges or generates zip. | Safe stream parser; max 50 entries, 100 MB, 10:1 ratio. |
| **OCR Inputs** | Scanned JPG, PNG, single-page PDF | **Provider capability confirmed** | Searchable PDF with invisible text layer. | ConvertAPI OCR. Initial languages: eng, spa, fra, deu. |
| **Web Page URL** | HTTP/HTTPS URLs | **Restricted / Disabled** | Webpage-to-PDF capture. | **Disabled in MVP (FR-036).** Requires SSRF security gate. |
| **EPUB / Email** | EPUB, EML, MSG | **Planned (P2)** | Paginate ebook or email content. | Deferred to Phase 5 expansion. |
| **Audio / Video / CAD**| MP3, MP4, DWG, 3D | **Unsupported** | No static conversion. | Rejected with `UNSUPPORTED_FORMAT`. |

### 7.2 Format Registry Fields
Each format entry SHALL include:
- `format_id`: Canonical format identifier
- `status`: Four-tier status (`Planned`, `Provider capability confirmed`, `Integration tested`, `Approved for production`)
- `extensions`: Accepted file extensions
- `signature_rule`: Magic numbers / byte patterns for content detection
- `permitted_mimes`: Permitted MIME hints from HTTP headers
- `engine_id`: Owning conversion provider/engine identifier
- `version`: Engine and dependency version
- `options_schema`: Supported options and JSON schema
- `limits`: Per-file size, max page, expanded archive limits
- `expected_output`: Expected output behavior and layout rules
- `known_limitations`: Known format limitations and caveats
- `license_review_status`: Applicable license status
- `locales`: Supported locales and scripts
- `test_fixtures`: Test fixture set reference
- `feature_flag`: Enable/disable flag state

### 7.3 Format Admission Checklist
- A valid and representative test corpus is available, including boundary, malformed, large, and multilingual files.
- Output quality is compared with a suitable reference renderer or agreed visual/content expectations.
- Engine license, redistribution, font license, and transitive dependencies are reviewed.
- Parser can run with no unnecessary network access, restricted filesystem access, and resource ceilings.
- Failure cases, timeout behavior, telemetry, and user-facing caveats are implemented.
- Security testing covers malformed input, decompression bombs, parser crashes, and cross-job isolation.

### 7.4 Business Rules
| Rule | Definition |
| :--- | :--- |
| **BR-01** | An extension alone never proves that a file is safe or supported. |
| **BR-02** | Unknown binary files are never silently rendered as text; a hex dump is not a standard conversion. |
| **BR-03** | Label each route as native render, text/data presentation, OCR-assisted, or extraction workflow. |
| **BR-04** | Mark output ready only after PDF validation; batch order follows the visible order. |
| **BR-05** | Report batch failures individually; keep valid outputs until their normal expiry. |
| **BR-06** | Never log passwords. Disclose and obtain consent before using third-party processing or AI. |

---

## 8. Non-Functional Requirements

### 8.1 Performance and Capacity
| ID | Requirement / Proposed Baseline | Measurement Method |
| :--- | :--- | :--- |
| **NFR-PERF-01** | For files up to 20 MB and up to 20 pages in the approved common-office benchmark corpus, target p95 conversion completion within 30 seconds after upload at normal load. Complex spreadsheets and OCR are measured as separate classes. | Automated benchmarks by format/engine and load band. |
| **NFR-PERF-02** | API requests for metadata/status/options should target p95 under 500 ms, excluding file upload, conversion, and third-party identity/billing services. | APM distributed traces. |
| **NFR-PERF-03** | Initial pilot should sustain 100 concurrent conversion jobs without cross-tenant data leakage or unbounded queue growth; capacity must be re-baselined from observed cost and load tests. | Load, soak, and queue saturation tests. |
| **NFR-PERF-04** | Large uploads should support resumable or multipart upload where the storage provider and client permit it. | Network interruption test. |
| **NFR-PERF-05** | Worker CPU, memory, scratch disk, and wall-clock limits SHALL be enforced per job. | Container resource and timeout tests. |

### 8.2 Availability and Recovery
- **NFR-AVL-01 (P1):** Target 99.5% monthly availability for the conversion control plane during public beta, excluding published maintenance and external dependencies, measured with synthetic checks.
- **NFR-AVL-02 (P1):** Queue outage or worker failure must not lose accepted job state; jobs shall move to retryable or failed status without appearing complete.
- **NFR-AVL-03 (P1):** Deployments must support rollback to a known-good API and worker image. Engine upgrades are versioned and can be disabled through configuration.
- **NFR-AVL-04 (P1):** Since files are temporary, file contents are not backed up. Job metadata backup/restore policy is separate and must not reintroduce expired artifacts.
- **NFR-AVL-05 (P2):** 99.9% availability and multi-region recovery are target candidates only after usage, cost, and operational readiness justify the investment.

### 8.3 Security
- **NFR-SEC-01:** All production traffic must use HTTPS. Storage objects in Cloudflare R2 must remain private by default and be retrievable only through authorized, short-lived presigned URLs.
- **NFR-SEC-02:** Validate file type through signature/content inspection; enforce file count, size, expanded-size, page count, timeout, and memory limits. Presigned PUT URLs can be reused prior to expiration; therefore, the system must enforce unique object keys per attempt, short expiration windows (5-10 min), rate limits, post-upload HEAD validation, immediate deletion of invalid objects, and quota controls.
- **NFR-SEC-03:** Untrusted file conversions are delegated to ConvertAPI's isolated sandboxes with non-root execution and macro execution strictly disabled.
- **NFR-SEC-04:** Inspect file signatures (magic bytes) before job acceptance; fail closed on unknown or tampered file types.
- **NFR-SEC-05:** Use application-generated storage names; protect against path traversal, archive traversal, XXE, XML expansion bombs, command injection, and parser vulnerabilities.
- **NFR-SEC-06:** URL rendering is **disabled by default in MVP**. When enabled in a future release, it must enforce HTTP/HTTPS allow rules, connection-time DNS/IP validation, redirect revalidation, and block loopback/private/link-local/metadata ranges.
- **NFR-SEC-07:** Secrets reside in a managed secret store (Vercel Environment Variables); they must not be placed in source, logs, job metadata, or client code.
- **NFR-SEC-08:** Apply least privilege, CSRF protections where cookie-auth is used, rate limits, bot/abuse controls, dependency scanning, and regular patching.
- **NFR-SEC-09:** Security events and administrative changes are auditable without recording document content.
- **NFR-SEC-10:** Run an independent threat model and penetration test before public launch; critical/high findings must be resolved or explicitly risk-accepted by the accountable owner.

### 8.4 Privacy and Retention
| ID | Requirement |
| :--- | :--- |
| **NFR-PRV-01** | Default processing is temporary. AtlasPDF targets initiating automated deletion of original inputs and converted outputs in Cloudflare R2 at 60 minutes after job terminal state. Deletion operations are executed durably via Inngest with retries, monitored for lag, and audited via periodic reconciliation. An immutable Cloudflare R2 bucket lifecycle rule (24 hours) serves as a secondary infrastructure safety net. |
| **NFR-PRV-02** | A "Delete now" action SHALL immediately revoke download access in the control plane and dispatch an immediate deletion request for R2 objects and the ConvertAPI file cache (where a FileId exists). |
| **NFR-PRV-03** | Job artifact retention (60m target), application logs (7-14 days), account profile data, and billing records have separate documented schedules. Metadata is anonymized and purged after 30 days. |
| **NFR-PRV-04** | Do not use submitted files to train AI models. Subprocessor (ConvertAPI) terms of service confirm files are not used for AI model training and are kept ephemerally. |
| **NFR-PRV-05** | The privacy notice must describe processing regions (e.g. EU or US regional endpoints), subprocessors (ConvertAPI, Cloudflare, Neon, Vercel), file retention, metadata logs, account data, and user rights in plain language. |
| **NFR-PRV-06** | Privacy/legal review must map obligations for launch markets, including relevant data protection and consumer rules; no certification or blanket legal compliance claim is implied by this SRS. |

### 8.5 Usability and Accessibility
- **NFR-UX-01:** A first-time user should complete a common conversion without an account and without reading a tutorial.
- **NFR-UX-02:** The landing/upload screen should communicate supported formats, maximum limits, privacy policy, and the next step before the user uploads.
- **NFR-A11Y-01:** Target WCAG 2.2 AA for the web UI, using automated checks plus keyboard and screen-reader testing [R20].
- **NFR-A11Y-02:** Focus order, text contrast, visible labels, status announcements, error association, and reduced-motion support must be tested.
- **NFR-LOC-01:** All content must use UTF-8 and layout must tolerate long filenames, Unicode scripts, and translated labels.
- **NFR-UX-03:** Mobile layout must support file chooser access, queue management, and download without hover-only controls.

### 8.6 Maintainability and Portability
- **NFR-MNT-01:** Each conversion engine must be hidden behind a stable adapter contract (`IConversionProvider`) for detect, validate, convert, inspect output, and report errors.
- **NFR-MNT-02:** Conversion engine versions and parameters must be pinned and tested against the regression corpus before production.
- **NFR-MNT-03:** The application must separate product UI, job orchestration, file storage, and rendering workers so one engine can be disabled or replaced independently.
- **NFR-MNT-04:** CI/CD must include unit, integration, contract, security, and visual-regression tests; release artifacts must be traceable to a source revision.
- **NFR-MNT-05:** Mock conversion providers are strictly forbidden in production; an automated production startup guard must abort deployment if a mock engine is configured in production.
- **NFR-OBS-01:** Metrics and traces must correlate API request, job, and engine version using opaque IDs and must not include file content or sensitive filenames.

---

## 9. System Architecture and Technical Design

### 9.1 Baseline Architecture Realization
The approved system realization implements the modular architecture via a modern cloud-native stack:
- **Application & Control Plane:** Next.js App Router and TypeScript, hosted on **Vercel** using the **Node.js runtime (`nodejs20.x` or `nodejs22.x`)**. The Edge runtime is deliberately avoided for storage, database, and conversion operations to ensure full compatibility with AWS SDK v3, buffer manipulation, and TCP connection pooling.
- **Metadata Database:** **Neon Serverless PostgreSQL** managed via **Drizzle ORM** with connection pooling over WebSockets/HTTPS.
- **Private Artifact Storage:** **Cloudflare R2** object storage (S3-compatible, private bucket, presigned PUT/GET URLs, and a 24-hour bucket lifecycle rule safety net).
- **Background Orchestration:** **Inngest** serverless durable event-driven functions (handling retries, backoff, step execution, cancellation, and delayed cleanup timers).
- **Conversion Engine:** Pluggable conversion provider abstraction, initially backed by **ConvertAPI** with regional endpoints and pinned parameters.
- **Developer Constraints:** Zero requirement for local .NET SDK, Docker Desktop, or local PostgreSQL daemon; uses cloud-managed developer instances or local in-memory mocks.

| Layer | Components | Responsibilities |
| :--- | :--- | :--- |
| **Experience** | Next.js (App Router, React Server Components, Client UI) | Upload dropzone, options drawer, truthful job progress, PDF preview, download, immediate deletion. |
| **Control Plane API** | Next.js Route Handlers (`/api/v1/*`) on Node.js runtime | Session management, input pre-validation, presigned upload URLs, job state transitions, ownership authorization. |
| **Storage** | Cloudflare R2 Private Bucket | Private storage for uploads and converted PDFs under generated opaque keys (`inputs/`, `outputs/`). Automatic bucket lifecycle policy (24h). |
| **Orchestration** | Inngest Durable Execution Engine | Step-wise conversion workflows, bounded backoff, idempotent retries, cancellation signals, and delayed 60-minute cleanup triggers. |
| **Conversion Adapters**| `ConversionProvider` interface + `ConvertApiProvider` | Format translation, option mapping, engine timeout isolation, output streaming to R2. |
| **Platform Services** | Neon PostgreSQL, Structured Logger | Opaque metadata storage, audit logs, error aggregation, operational metrics. |

### 9.2 Storage & Retention Breakdown by Location
To ensure transparency and compliance, retention and deletion behaviors are explicitly documented for every storage location:

| Storage Location | Artifact Type | Retention Policy & Cleanup Mechanism | Deletion Guarantees & Caveats |
| :--- | :--- | :--- | :--- |
| **Cloudflare R2 (`inputs/*`)** | Original uploaded files | Target cleanup initiation at 60 minutes after job termination via Inngest `DeleteObjectCommand`. | Immediate deletion on "Delete Now". Inngest retries on transient storage errors. 24-hour R2 bucket lifecycle rule serves as permanent safety net. |
| **Cloudflare R2 (`outputs/*`)** | Generated PDF outputs | Target cleanup initiation at 60 minutes after job completion. Presigned download URLs expire in 5 minutes. | Immediate revocation and deletion on "Delete Now". Backed by Inngest retries and 24-hour R2 lifecycle rule. |
| **ConvertAPI File Server** | Temporary input/output caches | Ephemeral processing cache. ConvertAPI automatically purges files after a maximum of **3 hours** [R06][R07]. | When ConvertAPI returns a `FileId`, AtlasPDF issues a `DELETE https://{region}.convertapi.com/d/{FileId}` request upon job completion. Provider deletion is best-effort and subject to upstream network availability. |
| **Neon PostgreSQL Database** | Metadata records (`jobs`, `job_files`, `quota_ledgers`) | Retained for **30 days** for quota tracking, abuse monitoring, and operational metrics. | Contains sanitized display names and safe error codes only. Zero file content or presigned URLs are stored. Purged via daily automated cron. |
| **Application & Serverless Logs** | Operational JSON logs | Retained for **7 to 14 days** per Vercel log drain defaults. | Filenames, file contents, passwords, session tokens, and presigned URLs are strictly redacted by logging interceptors. |

---

## 10. Data Model and API Requirements

### 10.1 Data Entities
| Entity | Key Fields | Rules |
| :--- | :--- | :--- |
| **GuestSession / User** | `session_id`, `plan_id`, `created_at`, `status` | Guest session is an opaque, cryptographically random, rate-limited identifier. |
| **Job** | `job_id`, `owner_ref`, `input_key`, `output_key`, `input_type`, `output_type`, `engine_id`, `state`, `progress_stage`, `created_at`, `terminal_at`, `expires_at`, `error_code` | No raw file content or signed URLs stored. Generated storage keys only. |
| **JobFile** | `file_id`, `job_id`, `display_name_safe`, `detected_format`, `size_bytes`, `page_count`, `order_index`, `status` | Sanitized display name; filenames excluded from system logs. |
| **ConversionProfile** | `profile_id`, `format_id`, `options_schema`, `defaults`, `max_bytes`, `max_pages`, `timeout`, `version` | Settings validated strictly against server-owned schemas. |
| **FormatRegistry** | `format_id`, `status`, `extensions`, `signature_rule`, `engine_id`, `version`, `caveats` | Four-tier status field; dynamic configuration can disable formats immediately. |
| **QuotaLedger** | `owner_ref`, `period`, `jobs_used`, `bytes_used`, `ocr_units`, `reservations` | Idempotent reservations; failed jobs follow charge/refund policy. |
| **AuditEvent** | `actor_ref`, `action`, `object_id`, `timestamp`, `request_id`, `outcome` | Security and administrative audit trail; no file content or secrets. |

### 10.2 Proposed API Surface
| Method / Path | Purpose | Controls |
| :--- | :--- | :--- |
| `GET /api/v1/formats` | Return active supported formats, limits, and source-aware options. | Cacheable; only returns formats with `Approved for production` status. |
| `POST /api/v1/jobs` | Create job and reserve quota. | Session auth, schema validation, idempotency key, rate limit. |
| `POST /api/v1/jobs/{id}/upload-ticket` | Issue a private short-lived R2 presigned PUT URL. | Owner authorization, expected size/type, 10-minute expiration, unique object key. |
| `POST /api/v1/jobs/{id}/complete-upload`| Finalize upload and dispatch Inngest conversion event. | Recheck object existence & size in R2; inspect magic bytes; atomic transition. |
| `GET /api/v1/jobs/{id}` | Read state, progress stage, warnings, expiry, and safe error. | Owner authorization only; no private storage keys exposed. |
| `POST /api/v1/jobs/{id}/cancel` | Request cancellation of an active job. | Owner-only, idempotent cancellation signal. |
| `POST /api/v1/jobs/{id}/delete` | Revoke access immediately and enqueue physical R2 and ConvertAPI deletion. | Owner-only; immediate access revocation. |
| `GET /api/v1/jobs/{id}/download` | Return a short-lived signed R2 download redirect after validation. | Owner-only; short TTL (300s); no permanent public URL. |
| `GET /api/v1/admin/metrics` | Admin operational metrics summary. | Admin role authorization, audit log. |

---

## 11. User Experience Requirements

### 11.1 Required Screens and States
| Screen / State | Required Elements | Acceptance Notes |
| :--- | :--- | :--- |
| **Landing / Upload** | Product promise, supported format link, file picker, drop zone, limits, and privacy summary. | No misleading "any file" claim. |
| **File Queue** | File name, detected type, size, compatibility status, reorder/remove action. | Extension/signature mismatch clearly flagged. |
| **Settings** | Output page defaults and source-aware options only. | Suppress irrelevant controls based on detected type. |
| **Processing** | Job stage, truthful progress phase, cancel action, and safe help. | No fake percentages; accessible screen-reader live announcements. |
| **Success** | Embedded PDF preview, download button, "Delete files now" button, expiry timer, warnings. | Success rendered only after PDF validation passes. |
| **Failure** | Specific user-safe error message, next steps, retry button, support code. | Never display raw stack traces or internal paths. |
| **Format Details / Help** | Accepted formats, limits, known caveats, OCR notes, sample use cases. | Dynamically matches active format registry. |
| **Admin Console** | Job metrics, safe failure log, engine health, quotas, format feature flags. | Sensitive actions audited. |

### 11.2 Product Copy Guardrails
> **Never claim:** "Convert any file", "perfectly preserves everything", "100% secure", "military-grade", "guaranteed 60-minute deletion", or "fully GDPR compliant" without an independently verified basis.  
> **Always prefer:** "Supports the formats listed here", "formatting may vary for complex documents", and "files target automated deletion starting at 60 minutes after conversion".

---

## 12. Error Handling and Observability

### 12.1 Error Catalogue
| Error Code | User Message | System Action |
| :--- | :--- | :--- |
| `UNSUPPORTED_FORMAT` | This file type is not supported yet. See supported formats. | Reject; do not route to worker. |
| `TYPE_MISMATCH` | The file extension does not match its contents. Check the file and try again. | Reject; delete uploaded object; do not route by extension alone. |
| `FILE_CORRUPT` | The file could not be read. Try opening and saving it again in its original app. | Fail deterministically; no infinite retries. |
| `PASSWORD_PROTECTED` | This file is protected and cannot be processed in this workflow. | Do not request or log passwords in MVP. |
| `MALWARE_DETECTED` | This upload was blocked because it failed a safety check. | Reject upload and record security audit event. |
| `LIMIT_EXCEEDED` | The file exceeds the current size or complexity limit. | Reject upload; delete uploaded object promptly. |
| `TIMEOUT` | Conversion took too long. Try a smaller file or simplify the document. | Terminate worker; clean scratch; retry once if transient. |
| `ENGINE_UNAVAILABLE` | Conversion is temporarily unavailable. Try again later. | Circuit-break; alert operations; avoid duplicate charge. |
| `OUTPUT_INVALID` | We could not validate the generated PDF. No file was marked ready. | Quarantine output; log diagnostics; alert if rate spikes. |
| `URL_BLOCKED` | This address cannot be processed for safety reasons. | Reject URL fetch; log category only. |
| `OCR_LOW_CONFIDENCE` | The PDF was created, but some text may be inaccurate. | Deliver output with warning label and preview. |

---

## 13. Security, Privacy and Compliance Requirements

### 13.1 Threat Model Summary
| Threat | Example Risk | Required Mitigation |
| :--- | :--- | :--- |
| **Parser Exploit** | Malformed DOCX/PDF/image compromises worker. | ConvertAPI isolated sandbox, non-root, restricted egress, adversarial testing. |
| **ZIP Bomb** | Tiny archive expands to consume disk/memory. | Enforce expanded-size (max 100MB), max file count (50), max depth (3), and 10:1 ratio limits. |
| **Path Traversal** | Archive member writes outside scratch area. | Strip directory paths, reject `..` segments and symlinks, normalize target names. |
| **Presigned URL Reuse & Abuse** | Client reuses presigned PUT URL to upload oversized or malicious files. | Unique object keys per ticket (`inputs/{jobId}/{fileId}-{randomHex}`), 10-minute expiration, post-upload HEAD verification, immediate deletion of oversized objects, and IP rate limits. |
| **SSRF** | URL renderer accesses cloud metadata or private network. | **URL conversion disabled in MVP.** Reject requests with 403 `URL_BLOCKED`. |
| **Cross-Tenant Data Leak** | User guesses job ID or reuses a download link. | Strict ownership checks (`job.owner_ref == session.id`), cryptographically signed cookies, private R2 objects, 5-minute signed download URLs. |
| **Sensitive Data Persistence** | Documents remain in storage or logs. | Targeted 60-minute cleanup initiation, Inngest retries, daily reconciliation, 24-hour R2 lifecycle rule, "Delete Now" button, log sanitization. |
| **Supply-Chain / Licensing** | Dependency vulnerability or license incompatibility. | Automated dependency scanning, version pinning, lockfiles, SBOM generation. |

---

## 14. Verification, Validation and Acceptance Criteria

### 14.1 Acceptance Test Catalogue
| Test ID | Scenario | Pass Condition |
| :--- | :--- | :--- |
| **AT-01** | Valid DOCX with tables, headers, footer, and mixed fonts converts into a readable PDF. | PDF opens; expected page count/layout verified; warnings displayed for font substitutions. |
| **AT-02** | XLSX with multiple sheets is submitted. | User can select sheets; PDF page scaling matches selection; printed output readable. |
| **AT-03** | PPTX is submitted with a slide-range option. | Each selected slide appears in expected order; speaker notes excluded by default. |
| **AT-04** | Multiple JPG/PNG images are submitted. | Output page order matches UI queue; rotation/orientation correct; fit behavior matches selection. |
| **AT-05** | JSON or source code has a line longer than page width. | Output readable and deterministic; long line wrap/overflow follows documented rules. |
| **AT-06** | A file extension is renamed to a different type. | System identifies mismatch or rejects before conversion; no unsafe dispatch. |
| **AT-07** | A zero-byte, corrupted, or password-protected file is submitted. | Each yields a specific safe error code; no infinite retries. |
| **AT-08** | An archive contains `../` path entries or excessive expanded size. | Archive rejected or unsafe entries excluded per policy; no writes escape scratch area. |
| **AT-09** | A URL resolves to localhost/private/metadata IP or redirects there. | URL job is blocked; worker cannot access destination; security event logged safely. *(Route disabled in MVP).* |
| **AT-10** | Two different users try to access each other's job or download. | Access denied regardless of guessed job ID or URL tampering. |
| **AT-11** | A user deletes a completed job. | Download access revoked immediately, deletion queued, and final artifact cleanup confirmed. |
| **AT-12** | A failed or terminal job reaches retention target. | Cleanup initiation triggered at 60 minutes; input and output artifacts removed. |
| **AT-13** | Worker/provider fails mid-conversion. | Job reaches failed or retryable state; no false success reported; scratch artifacts cleaned. |
| **AT-14** | Malware scanner / security check is unavailable for a high-risk format. | Configured fail-safe policy followed; file is not silently processed. |
| **AT-15** | A screen-reader user uploads and converts a file. | Controls, progress, errors, and download actions are properly announced and operable. |
| **AT-16** | A batch contains a mix of supported and unsupported files. | Unsupported files identified; supported files can continue; order and outcomes visible. |
| **AT-17** | Expired signed download URL is reused. | Access denied; new authorized link issued only while job is still valid. |
| **AT-18** | Engine or adapter version is upgraded. | Full regression corpus passes; release blocked if quality or security thresholds fail. |

---

## 15. Release Plan and Delivery Roadmap

### 15.1 Phased Plan
- **Phase 1: Project Scaffold, Shared Types, Database Schema & Drizzle Setup**
- **Phase 2: Cloudflare R2 Integration & Secure Direct-to-Storage Upload Flow**
- **Phase 3: Inngest Background Pipeline & Conversion Provider Abstraction**
- **Phase 4: Core Format Support Pipeline & PDF Output Validation**
- **Phase 5: Responsive Web UI / PWA Frontend & PDF Preview**
- **Phase 6: Privacy Enforcement, 60-Minute Retention SLA & Immediate Deletion**
- **Phase 7: Observability, Safe Error Handling & Admin Diagnostic Summary**
- **Phase 8: Golden Fixtures, Security Hardening & Acceptance Sign-off**
- **Phase 9: Production Deployment, Neon Branching & Rollback Verification**

---

## 16. Risks, Dependencies and Open Decisions

### 16.1 Risk Register
| Risk | Likelihood / Impact | Mitigation / Owner |
| :--- | :--- | :--- |
| **False "universal converter" expectation** | High / High | Product: Publish explicit format catalogue; display limitations before upload. |
| **Complex Office fidelity varies** | High / High | Engineering/QA: Reference corpus tests; display warnings for known layout/font differences. |
| **Third-party conversion cost exceeds free limits** | Medium / High | Business: Enforce strict per-IP/per-session daily quotas and file size limits. |
| **Presigned URL abuse / replay** | Medium / Medium | Engineering: Unique keys per ticket, 10-minute expiration, post-upload verification, immediate purge of invalid objects. |
| **SSRF vulnerability in URL rendering** | Medium / Critical | Security: Keep URL conversion completely disabled for initial MVP. |
| **Retention cleanup failure under outage** | Low-Medium / Critical | Platform: Inngest durable retries, hourly reconciliation sweep, Cloudflare R2 24h bucket lifecycle rule. |
| **Cross-tenant data exposure** | Low / Critical | Security: Cryptographic guest session cookies + strict database owner checks. |

---

## 17. Success Metrics

| Metric | Definition | Target / Initial Operating Rule |
| :--- | :--- | :--- |
| **Conversion Success Rate** | Successful validated jobs / jobs accepted for conversion. | >=98% on approved valid regression corpus. |
| **Task Completion** | Users who obtain a valid PDF / users who initiate conversion. | >=90% in moderated usability tests. |
| **Output Quality Defects** | Cases with layout/content defects / total reviewed conversions. | Tracked per format family; zero undetected corruptions. |
| **Time to PDF** | Elapsed time from upload completion to ready output. | p95 <= 30 seconds for files up to 20 MB / 20 pages. |
| **Deletion SLA Adherence**| Artifacts targeted for cleanup at 60 minutes / total terminal artifacts. | 100% of jobs trigger cleanup at 60m; daily reconciliation reports zero orphaned objects > 24 hours. |
| **Error Explainability** | Terminal failures mapped to documented safe error codes. | 100% of terminal failures. |
| **Security Incidents** | Confirmed data leaks, cross-tenant access, or SSRF events. | Zero tolerance (0 confirmed incidents). |

---

## Appendix A. Glossary
- **Artifact:** An uploaded input file, intermediate temporary file, or generated PDF stored for a conversion job.
- **ConvertAPI:** Third-party cloud file conversion service used as the initial SaaS conversion provider.
- **Drizzle ORM:** TypeScript object-relational mapper used for database access and schema migrations.
- **Inngest:** Durable event-driven execution platform providing reliable background jobs, retries, and scheduled steps.
- **Neon:** Serverless PostgreSQL database platform with branching and connection pooling.
- **Presigned URL:** Time-limited cryptographic URL granting direct read or write access to Cloudflare R2 object storage.
- **R2:** Cloudflare's S3-compatible private object storage service with zero egress fees.
- **SSRF:** Server-Side Request Forgery; an exploit where a server is induced to make unintended network requests.

---

## Appendix B. Research Sources
- [R01] Adobe Acrobat Help: Supported file formats for PDF conversion.
- [R02] Adobe Acrobat Online: Online file converter handling and supported types.
- [R03] Smallpdf: PDF Converter and public format scope.
- [R04] Smallpdf Tools Catalog: OCR, PDF/A, HTML, and Office formats.
- [R05] iLovePDF: Pricing, plan limits, and web/mobile processing tiers.
- [R06] CloudConvert: File Conversion API, async jobs, and webhooks.
- [R07] CloudConvert Privacy: Data retention and deletion policy.
- [R08] CloudConvert Security: Container isolation and processing regions.
- [R09] Convertio: Public converter homepage and claims.
- [R10] Convertio Privacy: File handling and 24-hour deletion policy.
- [R11] Zamzar: Multi-format conversion catalog.
- [R12] Zamzar Developers: Conversion API architecture and categories.
- [R13] PDF24: Creator offline tool and local file processing.
- [R14] PDF24 Webpage-to-PDF: Temporary cloud processing and cleanup.
- [R15] FreeConvert: File retention and AWS storage policies.
- [R16] Microsoft Support: Office PDF export options and spreadsheet link handling.
- [R17] LibreOffice Help: PDF export settings and options.
- [R18] OWASP Cheat Sheet Series: File Upload Cheat Sheet.
- [R19] OWASP Cheat Sheet Series: Server-Side Request Forgery Prevention Cheat Sheet.
- [R20] W3C Web Accessibility Initiative: Web Content Accessibility Guidelines (WCAG) 2.2.
- [R21] OCRmyPDF Documentation: OCR pipeline and PDF/A trade-offs.

---

## Appendix C. Release Readiness Checklist
| Owner | Sign-off Item | Done |
| :--- | :--- | :---: |
| **Product** | Public copy only describes actively supported formats and tested limitations. | [ ] |
| **Product** | Guest limits, batch behavior, retry semantics, and billing/quotas are documented. | [ ] |
| **Engineering** | All P1 functional requirements are mapped to tests and release notes. | [ ] |
| **Engineering** | Every enabled engine is pinned, reproducible, monitored, and independently disableable. | [ ] |
| **Engineering** | Production startup guard strictly rejects `MockConversionProvider` configuration. | [ ] |
| **Security** | File upload and worker isolation threat model reviewed. | [ ] |
| **Security** | URL-to-PDF feature disabled unless SSRF acceptance tests and penetration review pass. | [ ] |
| **Privacy** | Retention policy (60-minute cleanup initiation target) published only after deletion tests pass. | [ ] |
| **Privacy** | Logs, metrics, and error reports contain no file content, secrets, or signed URLs. | [ ] |
| **QA** | Golden-file and visual regression corpus passes for every active format. | [ ] |
| **QA** | Malformed, large, archive-bomb, Unicode, and unsupported inputs tested. | [ ] |
| **Accessibility** | WCAG 2.2 AA target test plan complete; keyboard and screen-reader checks passed for core flows. | [ ] |
| **Operations** | Alerts, on-call owner, rollback path, incident runbooks, and service status communications prepared. | [ ] |
| **Legal** | Engine/font licenses, terms, privacy notice, subprocessor agreements, and launch-market obligations reviewed. | [ ] |
| **Business** | Cost per successful conversion and quota economics measured before final pricing. | [ ] |
