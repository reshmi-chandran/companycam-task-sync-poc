## Build Plan (Grid & Ferry Research Tasks)

### Objectives
- Refactor and unify the existing Grid (4 levels) and Ferry (5 levels) tasks into a single, configurable web experience.
- Implement a minimal Python backend API to run Asymmetry and Order-of-Symmetry analytics, forward payloads to researchers’ callback endpoints, and supply redirect info.
- Deliver documentation, deterministic behavior, Qualtrics compatibility, and a ready-to-deploy package on a low-cost serverless/static host.

### Target Architecture
- **Frontend**: React + Vite SPA hosted on Vercel/Netlify. URL parameters (`task`, `level`, `pid`, `callbackUrl`, `returnUrl`, `seed`, flags) drive task selection and behavior. Client captures raw trial events, ensures deterministic randomization via seeded PRNG, and handles POST/redirect flow.
- **Backend**: FastAPI (Python 3.x) packaged as a serverless function. Endpoint `/process` validates submissions, computes Asymmetry & Order-of-Symmetry, posts consolidated JSON to `callbackUrl`, and returns redirect metadata to the client. No persistent storage; logs stored via hosting provider.
- **Integration**: Qualtrics launches participants through redirect; backend posts full results to researcher endpoint; frontend redirects participants back with `pid` and status query parameters.

### Implementation Schedule
| Phase | Focus |
|-----|-------|
| 1 | Kickoff & environment setup, audit existing Heroku tasks, finalize tech stack, scaffold React/Vite app and FastAPI function, define payload schemas & deterministic RNG strategy. |
| 2 | Port/refactor Grid & Ferry task logic into modular React components, implement seeded PRNG utilities, create level configuration files, and wire core gameplay loops with trial logging. |
| 3 | Build submission pipeline on frontend (state store, payload serializer, retry UX) and stub backend endpoint; ensure URL param validation, error handling, and redirect scaffolding. |
| 4 | Implement FastAPI logic: schema validation (Pydantic), Asymmetry & Order-of-Symmetry functions, webhook dispatcher with retries, redirect response contract; integrate with frontend. |
| 5 | End-to-end testing: automated unit tests (Jest/Pytest), browser smoke tests (Playwright/Cypress), manual cross-browser QA, deterministic seed verification, Qualtrics sandbox run. |
| 6 | Documentation pass: README, deployment guide, data dictionary (raw fields + symmetry metrics), Qualtrics integration instructions, troubleshooting tips, change log. |
| 7 | Deployment to chosen host (e.g., Vercel for both SPA and serverless function), environment configuration, final verification with researcher-provided callback, handoff package preparation. |

### Deliverables
- Unified React/Vite codebase (`/app`) with modular task engines, shared utilities, and automated build pipeline.
- FastAPI backend (`/api`) ready for serverless deployment, including symmetry analytics, webhook relay, and redirect responses.
- CI scripts for lint/test/build, plus deployment configuration for Vercel or Netlify.
- Documentation bundle: quick start, architecture overview, configuration guide, Qualtrics integration walkthrough, data dictionary, and versioning notes.
- Demo recording or screenshot walkthrough showing configuration, task execution, data submission, and redirect back to Qualtrics.

### Deployment & Configuration
- Single repository containing both frontend and backend, deployed via Vercel monorepo or Netlify app + functions.
- Environment variables: `API_BASE_URL`, `ALLOWED_ORIGINS`, optional `WEBHOOK_SECRET`, `RETURN_URL_DEFAULT`.
- Deterministic logging: each run records `taskVersion`, `buildCommit`, `seed`, and `timestamp` within payloads and console logs for traceability.

### Risks & Mitigations
- **Legacy logic mismatches**: Validate against original Heroku tasks using reference sessions and regression fixtures.
- **Webhook failures**: Backend implements retries/backoff and exposes status codes to frontend; include manual download fallback.
- **Qualtrics/CORS constraints**: Coordinate allowed origins early; provide guidance for researchers on hosting endpoints that accept POSTs.
- **Timeline compression**: Prioritize core flows (task execution, analytics, redirect) before polish; keep daily checkpoints with stakeholders for rapid feedback.

### Assumptions
- Access to original task assets/code and Python analytics scripts at project start.
- Researcher callback endpoints or mock servers available before integration testing begins.
- Hosting accounts (Vercel/Netlify) ready for deployment; SSL handled automatically by platform.

