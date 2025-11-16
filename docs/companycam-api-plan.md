## CompanyCam Proof-of-Concept API Plan

### Objectives
- Deliver a backend-focused proof that the PM system can push tasks into CompanyCam, capture 100+ offline photos per task, sync them back with immutable metadata, allow admin photo selection, and export PDFs with only chosen images.
- Validate offline-first reliability, CompanyCam interoperability (albums, photo metadata, webhooks), and selective export accuracy within a compressed 48-hour window.

### Proof Targets (48-Hour Chain)
1. **PM Task → CompanyCam Album**: API accepts a task payload, provisions a mapped CompanyCam album, and stores the association in Postgres.
2. **Offline 100+ Photo Burst**: Mobile client (or scripted harness) captures >100 photos offline, queues them with hashes, and submits when back online.
3. **Sync Back with Immutable Metadata**: API uploads missing files to CompanyCam, receives webhook confirmations, and persists timestamp/user/GPS metadata in read-only form for that task.
4. **Admin Selects Photos**: Dashboard consumes `GET /tasks/:id/photos`, renders thumbnails with metadata, and allows checkbox selection with ordering preserved server-side.
5. **Export PDF with Only Selected Images**: `POST /tasks/:id/pdf` builds a report containing task header and just the selected photos (ordered), each annotated with immutable metadata.

### 48-Hour Test Plan
- **Environment Prep**: Configure CompanyCam sandbox, Postgres schema (`tasks`, `photos`, `selections`, `exports`), S3 bucket for temporary uploads, and webhook tunnel (ngrok) for local validation.
- **Test 1 – Task→Album Mapping**: Use REST client to create PM task; assert album ID returned and stored. Cross-check via CompanyCam dashboard that the album exists with matching metadata.
- **Test 2 – Offline Burst Intake**: Run a scripted harness that simulates 120 photos with mixed quality, duplicates, and varied timestamps. Force offline mode by disabling network, queue locally, then re-enable to trigger batch POST. Validate API deduplicates via hash and enqueues uploads without loss.
- **Test 3 – Metadata Integrity**: After CompanyCam uploads complete, inspect webhook logs to ensure each photo entry in Postgres has immutable fields (timestamp, user, GPS, photoId). Attempt to mutate metadata via API (should be rejected).
- **Test 4 – Admin Selection Workflow**: Hit `GET /tasks/:id/photos` and verify payload includes thumbnails + metadata. Perform selection POST with ordered IDs; refresh and confirm persisted state matches order and checkbox values.
- **Test 5 – Selective PDF Export**: Trigger `POST /tasks/:id/pdf` using a subset of photos. Confirm job completes, PDF stored in S3, and download includes only selected photos in order with metadata captions. Spot-check that non-selected photos are absent.
- **Test 6 – End-to-End Demo**: Record a walkthrough covering the entire chain: create task → verify album → simulate offline burst → sync → review metadata → select subset → download PDF. Capture logs, API responses, and resulting artifacts for stakeholder review.

### Architecture Summary
- **Backend Service**: Node.js + Fastify (TypeScript) deployed on AWS Fargate or Lambda, with Postgres (RDS) for task/photo metadata and job queues.
- **CompanyCam Adapter**: OAuth2 client storing refresh tokens, thin wrapper for album and photo endpoints, plus webhook receiver for photo events and metadata confirmation ([CompanyCam API docs](https://help.companycam.com/en/articles/6828353-api-and-custom-integrations)).
- **Storage & Queueing**: S3 temporary bucket for offline uploads, AWS SQS for deduplicated photo jobs to respect CompanyCam rate limits.
- **Admin Tools**: REST endpoints serving photo grids, selection state, and PDF export jobs consumed by the PM dashboard (React web).
- **PDF Pipeline**: Puppeteer template service that renders task header, ordered photos, and per-photo metadata; outputs to S3 with signed download URLs.

### Integration Flow
1. **Task Provisioning**: `POST /tasks` receives PM task payload, creates or reuses a CompanyCam album, persists mapping (`taskId`, `albumId`, version, seed) in Postgres.
2. **Mobile Sync Tokens**: API issues signed upload URLs/tokens so field crews push offline batches when connectivity returns.
3. **Offline Upload Queue**: Mobile client posts batches (`photoId`, hash, metadata, binary). API stores batch in S3, enqueues dedup job, and marks provisional entries in Postgres.
4. **CompanyCam Upload & Webhook**: Worker uploads missing files to CompanyCam, records resulting photo IDs. Webhook receiver confirms final metadata (timestamp, user, GPS) and marks photo immutable.
5. **Admin Selection**: Dashboard calls `GET /tasks/:id/photos` for grid-ready payload. Admin toggles checkboxes; selections stored via `POST /tasks/:id/photos/selection` with order preserved.
6. **PDF Export**: Admin triggers `POST /tasks/:id/pdf`. Service pulls selected photos, requests high-res URLs, renders Puppeteer template, stores PDF, and notifies dashboard.

### Core API Surface
- `POST /tasks` – Create task + album mapping, return IDs and upload credentials.
- `POST /tasks/:id/photos` – Accept offline batch metadata + binary (multipart or signed URL references); respond with dedupe status.
- `GET /tasks/:id/photos` – Paginated photo list (thumbnail URL, checkbox state, timestamp, user, GPS, hash).
- `POST /tasks/:id/photos/selection` – Save ordered selection, capturing admin ID and timestamp for audit.
- `POST /tasks/:id/pdf` – Initiate export job; returns job ID plus eventual signed PDF URL.
- Webhooks: `/webhooks/companycam` – Validate signature, persist photo metadata, and reconcile status flags.

### Offline & Deduplication Strategy
- Client computes SHA-256 hash per photo; server uses hash + task ID to detect duplicates before re-upload.
- Batch uploads stored in S3 with metadata manifest; worker retries failed CompanyCam uploads with exponential backoff.
- Postgres tables track states: `pending`, `uploaded`, `confirmed`. Only `confirmed` entries surface in admin UI.
- Conflict handling: if webhook arrives before local batch confirmation, merge by hash and keep earliest metadata.

### Selective Photo Export
- Admin grid UI consumes API payload with thumbnail CDN URLs (CompanyCam or cached proxy), metadata badges, and checkbox bindings.
- Selection changes persisted immediately to support collaborative workflows.
- PDF builder respects stored order, injecting captions: photo index, timestamp, user, GPS coordinates, CompanyCam photo ID.
- Export audit log records requester, version, and list of photo IDs to guarantee immutability.

### 48-Hour Proof-of-Concept Test Plan
- **Setup (0–6h)**: Deploy Fastify skeleton, Postgres schema (`tasks`, `photos`, `selections`, `exports`), and mock CompanyCam adapter for local runs.
- **Integration (6–18h)**: Connect to CompanyCam sandbox: album creation, photo upload, webhook receiver. Seed sample task, verify API maps album IDs correctly.
- **Offline Burst (18–30h)**: Script 120-photo offline batch with duplicates and forced network toggles. Validate queue processing, dedupe accuracy, and webhook reconciliation.
- **Selection & PDF (30–42h)**: Build selection endpoints plus Puppeteer template with placeholder branding. Ensure metadata renders per photo.
- **Validation (42–48h)**: Run end-to-end demo: task → album → offline upload → admin selection → PDF download. Capture logs, metrics, and video walkthrough for stakeholders.

### Deliverables
- Sync architecture document detailing flow charts, sequence diagrams, and data contracts.
- Node.js API repo with CompanyCam integration, offline queue handling, selection state, and PDF export endpoint.
- Test artifacts: scripts/logs proving 100-photo offline burst, dedupe, webhook reconciliation, and selective PDF export.
- Demo assets: short video showing dashboard grid, selection workflow, and final PDF; README with setup instructions and environment variable documentation.


