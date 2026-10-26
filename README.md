# Aegis Platform — Repository Research Handoff (NotebookLM + Paper Prep)

This README is a **repository-grounded handoff** describing what is actually implemented in `Ilyankhan/aegis-platform` at the time of analysis.

---

## Table of Contents

1. [Scope and Evidence Policy](#scope-and-evidence-policy)
2. [Repository Identity and Purpose](#repository-identity-and-purpose)
3. [Monorepo Structure](#monorepo-structure)
4. [Implemented System Architecture](#implemented-system-architecture)
5. [End-to-End Data Flows](#end-to-end-data-flows)
6. [Backend (FastAPI + Celery) Deep Dive](#backend-fastapi--celery-deep-dive)
7. [Frontend (Next.js 15 / React 19) Deep Dive](#frontend-nextjs-15--react-19-deep-dive)
8. [CLI Integration State (What Exists vs Missing)](#cli-integration-state-what-exists-vs-missing)
9. [Data Models and Persistence](#data-models-and-persistence)
10. [Cryptography and Secret Handling](#cryptography-and-secret-handling)
11. [Authentication, Sessions, and Authorization](#authentication-sessions-and-authorization)
12. [Webhooks and Scan Orchestration](#webhooks-and-scan-orchestration)
13. [Notifications and External Integrations](#notifications-and-external-integrations)
14. [Deployment, Runtime, and Configuration](#deployment-runtime-and-configuration)
15. [Tests, CI, and Operational Scripts](#tests-ci-and-operational-scripts)
16. [Operational Commands Grounded in Repo Files](#operational-commands-grounded-in-repo-files)
17. [File-by-File Inventory](#file-by-file-inventory)
18. [Implementation Limitations and Repository Discrepancies](#implementation-limitations-and-repository-discrepancies)
19. [Research-Ready Summary for NotebookLM / Paper Drafting](#research-ready-summary-for-notebooklm--paper-drafting)

---

## Scope and Evidence Policy

- This document uses only repository evidence from files under:
  - `/home/runner/work/aegis-platform/aegis-platform/README.md` (previous state)
  - `/home/runner/work/aegis-platform/aegis-platform/PRD.md`
  - `/home/runner/work/aegis-platform/aegis-platform/AEGIS_CLI_SPEC.md`
  - `/home/runner/work/aegis-platform/aegis-platform/apps/**`
  - `/home/runner/work/aegis-platform/aegis-platform/deploy/**`
  - `/home/runner/work/aegis-platform/aegis-platform/.github/workflows/**`
  - `/home/runner/work/aegis-platform/aegis-platform/scripts/**`
- Where docs/specs claim behavior not directly evidenced in implementation files, this README marks that explicitly.

---

## Repository Identity and Purpose

- Repository: `Ilyankhan/aegis-platform`
- Root `package.json` describes it as:
  - “Centralized cloud-native control plane and orchestration fabric for the Aegis security ecosystem”
- Implemented stack from code/config:
  - Backend: FastAPI + SQLAlchemy async + Alembic + Celery + Redis (`apps/api`)
  - Frontend: Next.js App Router + React 19 (`apps/web`)
  - Data stores expected: PostgreSQL + Redis
- Core implemented use case:
  - Ingest scan findings (from webhook-driven repo scans and CLI sync payloads)
  - Persist incidents and scan runs
  - Track triage lifecycle and audit chain
  - Serve telemetry + dashboard APIs
  - Send Slack alerts

---

## Monorepo Structure

Top-level evidence:

- `apps/api`: backend API + worker code
- `apps/web`: frontend dashboard
- `deploy`: local compose + deployment manifests
- `.github/workflows`: CI and keepalive automation
- `scripts`: keepalive utility script
- Root install/start/deploy metadata: `Dockerfile`, `render.yaml`, `install.sh`, `install.ps1`, `start.sh`, `requirements.txt`, `package.json`

---

## Implemented System Architecture

### Backend entrypoint and API surface

- FastAPI app defined in `apps/api/app/main.py` as `app = FastAPI(...)`
- API v1 router mounted with prefix `/api/v1`
- Route groups from `apps/api/app/api/v1/router.py`:
  - `/auth`
  - `/webhooks`
  - `/repositories`
  - `/incidents`
  - `/scans`
  - `/telemetry`
  - `/organizations`
- Health endpoints:
  - `/health`
  - `/health/ready` and `/ready`

### Worker/queue subsystem

- Celery app in `apps/api/app/core/celery_app.py`
- Task entrypoint: `process_scan_event` in `apps/api/app/workers/tasks.py`
- Queue broker/backend: `settings.REDIS_URL`

### Frontend

- Next.js app under `apps/web/app`
- Main dashboard page component: `apps/web/app/page.tsx`
- API contracts and client helpers: `apps/web/lib/api.ts`

---

## End-to-End Data Flows

### 1) GitHub webhook → scan queue → worker processing

1. `POST /api/v1/webhooks/github` (`apps/api/app/api/v1/endpoints/webhooks.py`)
2. Validates payload and repository context
3. Verifies HMAC signature via `verify_github_signature(...)` (`apps/api/app/services/webhook_auth.py`)
4. Creates `ScanRun(status="QUEUED")`
5. Dispatches Celery task `aegis.tasks.process_scan_event`
6. Worker clones repo shallowly, runs CLI, parses JSON findings, updates incidents/audits, sends optional Slack summary

### 2) CLI sync payload → incident upsert

1. `POST /api/v1/scans/cli` (`apps/api/app/api/v1/endpoints/scans.py`)
2. Accepts `CliScanPayload` (`apps/api/app/schemas/scan_run.py`)
3. Creates `ScanRun(trigger_source="cli", status="COMPLETED")`
4. Upserts incident rows using deterministic fingerprint logic
5. Writes tamper-evident audit entries
6. Sends optional Slack summary for high/active findings

### 3) Dashboard load and interactions

- `apps/web/app/page.tsx` concurrently requests telemetry/repositories/incidents/organizations
- Calls come from `apps/web/lib/api.ts` helpers:
  - `fetchTelemetry`, `fetchRepositories`, `fetchIncidents`, etc.
- Triage actions call backend status APIs and optimistically update UI

---

## Backend (FastAPI + Celery) Deep Dive

### Core modules (`apps/api/app/core`)

- `config.py`: Pydantic settings model; env loading from `apps/api/.env`
- `database.py`: async SQLAlchemy engine/session + `get_db`
- `security.py`: password hashing, manual HS256 JWT encode/decode, session revocation manager
- `crypto.py`: masking, blind index, fingerprint generation, AES-GCM encryption/decryption helpers
- `celery_app.py`: Celery configuration
- `rate_limiter.py`: Redis-backed and in-memory fallback rate limits + account lockouts
- `status_page.py`: HTML status rendering for root route

### Endpoint modules (`apps/api/app/api/v1/endpoints`)

- `auth.py`: register/login/refresh/logout, OAuth callback paths, profile ops, CLI token generation
- `incidents.py`: incident listing, triage updates, audits, exports, bulk operations
- `organizations.py`: workspace settings including Slack webhook and test-alert endpoint
- `repositories.py`: repository onboarding, scans, webhook install config, scan-all
- `scans.py`: CLI payload ingest
- `telemetry.py`: aggregate dashboard counters + MTTR calculation
- `webhooks.py`: GitHub webhook ingestion + async dispatch

### Worker (`apps/api/app/workers/tasks.py`)

Key implemented behavior:

- Delivery idempotency lock in Redis key `lock:delivery:{guid}` with `ex=86400, nx=True`
- Input hardening for clone URL and branch patterns before `git clone`
- Shallow clone command: `git clone --depth=1 --branch <branch> -- <url> <temp_dir>`
- Optional GitHub token injection for private clone URLs
- Scanner command:
  - `aegis scan <temp_dir> --format=json --verify --no-color`
- Accepts scanner exit codes `0` and `1`; non-zero/one treated as failure
- Computes token identity blind index + incident fingerprint
- Incident lifecycle logic:
  - create new incident on first sighting
  - reopen resolved/dismissed as `REGRESSION` on reappearance
  - auto-resolve open/regression incidents missing from current fingerprint set
- Writes audit events via `create_tamper_evident_audit`
- Sends aggregated Slack summary when findings/regressions exist
- Cleans `/tmp` clone directory in `finally`

---

## Frontend (Next.js 15 / React 19) Deep Dive

### Routing and middleware

- Middleware at `apps/web/middleware.ts`:
  - protects non-public routes via `aegis_auth_token` cookie
  - redirects non-local `vercel.app` host traffic to canonical domain
  - rewrites `aegis.ilyankhan.tech` install path
  - injects security headers (`X-Frame-Options`, `X-Content-Type-Options`, etc.)

### State and context

- `apps/web/context/AuthContext.tsx`:
  - session bootstrapping via `fetchCurrentUser`
  - login/register/logout wrappers
  - cross-tab auth token sync via `storage` event
- `apps/web/context/ToastContext.tsx`:
  - global toast UI event bus

### API contract layer

- `apps/web/lib/api.ts` defines TS interfaces matching backend concepts:
  - `Incident`, `Repository`, `ScanRun`, `TelemetryData`, auth/org/webhook types
- Handles token attach, refresh retry on 401, and endpoint wrappers for major flows

### Major UI components (implemented)

- Incident management: `IncidentTable`, `IncidentDetailModal`, `IncidentToolbar`, `components/incident/*`
- Repository/scan views: `RepositoriesView`, `ScansView`
- Navigation/UX: `Navbar`, `CommandMenu`, `LandingView`, skeleton components
- Settings/modals: account, alert, webhook, keyboard shortcuts
- Telemetry cards: `TelemetryCards`

---

## CLI Integration State (What Exists vs Missing)

### Implemented integration points

- Worker invokes external `aegis` executable (`apps/api/app/workers/tasks.py`)
- CLI scan ingest endpoint: `POST /api/v1/scans/cli`
- CLI token endpoint: `GET /api/v1/auth/cli-token` (30-day JWT)
- Frontend exposes CLI auth/token helpers in `apps/web/lib/api.ts`

### Not bundled in this repository

- No `apps/cli` source tree is present
- No `apps/api/bin/aegis` file is present in current tree
- Worker therefore depends on externally available CLI binary (`AEGIS_BIN_PATH`, system path, or `/usr/local/bin/aegis`)

---

## Data Models and Persistence

SQLAlchemy model files:

- `Organization` (`apps/api/app/models/organization.py`)
- `Repository` (`apps/api/app/models/repository.py`)
- `ScanRun` (`apps/api/app/models/scan_run.py`)
- `Incident` (`apps/api/app/models/incident.py`)
- `IncidentAudit` (`apps/api/app/models/audit.py`)
- `User` (`apps/api/app/models/user.py`)

Important constraints/indexes evidenced in code:

- Incident uniqueness: `UniqueConstraint("repository_id", "fingerprint")`
- Incident operational indexes on status/severity/last_seen/verification status
- FK relationships for cascade behavior (e.g., repos under org, incidents under repo)

Database initialization and migration evidence:

- `deploy/init.sql`: enables `uuid-ossp` and `pgcrypto`
- Alembic config and migrations under `apps/api/alembic/**`

---

## Cryptography and Secret Handling

From `apps/api/app/core/crypto.py`:

- `mask_secret(secret)` preserves first 4 chars and masks the rest
- `compute_blind_index(secret)` uses HMAC-SHA256 with `AEGIS_BLIND_PEPPER`
- `compute_incident_fingerprint(repository_id, rule_id, file_path, secret_hash)` uses SHA256 deterministic key
- `encrypt_secret(raw_secret)` uses AES-GCM with 12-byte nonce prepended
- `decrypt_secret(...)` reverses nonce+ciphertext format
- Token helpers:
  - `encrypt_token_b64`
  - `decrypt_token_b64` (with plaintext fallback path)

Note: Worker currently encrypts `masked_val` when writing `encrypted_secret_blob` (not raw plaintext secret), because worker receives only masked output from CLI JSON.

---

## Authentication, Sessions, and Authorization

From `apps/api/app/core/security.py` and `apps/api/app/api/v1/endpoints/auth.py`:

- Password hashing: PBKDF2-HMAC-SHA256 (100,000 iterations)
- JWT implementation: custom HS256 signing/verification logic
- Access token default TTL: 15 minutes (`create_access_token`)
- Refresh token default TTL: 7 days (`create_refresh_token`) with `jti`
- Refresh rotation and reuse detection via `SessionTokenManager`
- Revocation persistence attempts Redis, falls back to in-memory sets/maps
- Route guards use `get_current_user` dependency

Rate limiting and lockout:

- `RateLimiter` supports Redis-first with in-memory fallback
- Login lockout policy implemented in `AccountLockoutManager`

---

## Webhooks and Scan Orchestration

From `apps/api/app/api/v1/endpoints/webhooks.py`:

- Supports `push` and selected `pull_request` actions
- Handles `ping` handshake
- Auto-provisions repository (and default org if absent)
- Requires/validates `X-Hub-Signature-256` under enforced conditions
- Handles branch delete/zero-SHA events by acknowledging without scan

From `apps/api/app/services/github_service.py`:

- Can auto-install repository webhook via GitHub API (`events: push, pull_request`)
- Webhook URL generated from backend base URL with localhost fallback to Render URL

---

## Notifications and External Integrations

### Slack

From `apps/api/app/services/notifications.py`:

- `send_slack_incident_alert(...)` exists (single incident card)
- `send_slack_scan_summary_alert(...)` used by worker and CLI sync path
- `send_test_alert(...)` used by org settings endpoint

### Email (Resend)

From `apps/api/app/services/email.py`:

- `EmailService` builds HTML templates for:
  - OTP verification
  - password reset
  - welcome email
  - breach alert
- `send_email(...)` calls `https://api.resend.com/emails`
- If key missing/mock, logs and returns success for dev/test path

### SIEM formatter utilities

From `apps/api/app/services/siem.py`:

- Provides CEF/syslog formatting and dispatch helpers
- No direct endpoint/worker invocation found in reviewed API/worker paths

---

## Deployment, Runtime, and Configuration

### Local compose

- `deploy/docker-compose.yml` runs:
  - `postgres:16-alpine`
  - `redis:7-alpine`
  - `api` (build from `apps/api/Dockerfile`)
  - `web` (build from `apps/web/Dockerfile`)

### API startup

- `apps/api/start.sh`:
  - attempts Alembic migration
  - optionally starts Celery worker if `REDIS_URL` and `ENABLE_CELERY_WORKER=true`
  - starts Uvicorn

### Dockerfiles

- `apps/api/Dockerfile`: installs git/curl, runs CLI installer URL, installs Python deps, starts `start.sh`
- Root `Dockerfile`: similar but copies `apps/api/` from root context
- `apps/web/Dockerfile`: multi-stage Next.js build and runtime

### Render + Vercel manifests

- `deploy/render.yaml` includes API web service and optional dedicated worker service
- Root `render.yaml` defines API service only with somewhat different env var set
- `deploy/vercel.json` and `apps/web/vercel.json` are both present and equivalent schema/build/install entries

### Environment examples

- `apps/api/.env.example` documents backend vars
- `apps/web/.env.example` documents `NEXT_PUBLIC_API_URL`

---

## Tests, CI, and Operational Scripts

### Backend tests (`apps/api/tests`)

- `test_auth.py`
- `test_compliance_and_audit.py`
- `test_crypto.py`
- `test_health_and_security.py`
- `test_incidents.py`
- `test_notifications.py`
- `test_rate_limiter.py`
- `test_repositories.py`
- `test_scans_and_integrations.py`
- `test_webhooks.py`

### CI workflows

- `.github/workflows/ci.yml`
  - backend pytest job
  - frontend lint + build job
- `.github/workflows/keepalive.yml`
  - cron ping every 10 minutes to `/health`

### Ops script

- `scripts/keepalive.sh`: manual keepalive and readiness probe helper

---

## Operational Commands Grounded in Repo Files

### From root `package.json`

- `npm run dev` → runs `apps/web` dev server
- `npm run build` → builds `apps/web`
- `npm run start` → starts `apps/web`
- `npm run lint` → lint web app

### Local infrastructure

```bash
docker compose -f deploy/docker-compose.yml up -d
```

### API runtime

```bash
cd apps/api
pip install -r requirements.txt
./start.sh
```

### Frontend runtime

```bash
cd apps/web
npm install
npm run dev
```

### Tests and CI-equivalent checks

```bash
pytest apps/api/tests/ -v
cd apps/web && npm run lint && npm run build
```

---

## File-by-File Inventory

### Root

- `README.md` (this document)
- `PRD.md` (product requirements narrative/spec claims)
- `AEGIS_CLI_SPEC.md` (CLI contract/spec claims)
- `Dockerfile` (root API image build path)
- `render.yaml` (root Render config)
- `requirements.txt` (`-r apps/api/requirements.txt`)
- `package.json`, `package-lock.json` (workspace + web scripts)
- `install.sh`, `install.ps1` (CLI installers)
- `start.sh` (root delegator to `apps/api/start.sh`)
- `.aegisignore`, `.gitignore`, `.python-version`, `LICENSE`

### Deployment folder (`deploy/`)

- `docker-compose.yml`
- `init.sql`
- `render.yaml`
- `vercel.json`

### GitHub workflows (`.github/workflows/`)

- `ci.yml`
- `keepalive.yml`

### Scripts (`scripts/`)

- `keepalive.sh`

### Backend package (`apps/api/`)

- Runtime/config: `Dockerfile`, `start.sh`, `requirements.txt`, `alembic.ini`, `.env.example`
- Alembic: `alembic/env.py`, migrations in `alembic/versions/*.py`
- Core source tree (`app/`):
  - `main.py`
  - `api/v1/router.py`
  - `api/v1/endpoints/{auth,incidents,organizations,repositories,scans,telemetry,webhooks}.py`
  - `core/{celery_app,config,crypto,database,rate_limiter,security,status_page}.py`
  - `models/{organization,repository,scan_run,incident,audit,user}.py`
  - `schemas/{auth,incident,repository,scan_run,telemetry,webhook,organization,audit}.py`
  - `services/{audit,auth_tokens,email,github_service,notifications,siem,webhook_auth}.py`
  - `workers/tasks.py`
- Tests: `tests/*.py` listed above

### Frontend package (`apps/web/`)

- Runtime/config: `Dockerfile`, `package.json`, `next.config.ts`, `tsconfig.json`, `tailwind.config.ts`, `eslint.config.mjs`, `postcss.config.mjs`, `.env.example`, `vercel.json`
- App routes in `app/` including:
  - `page.tsx` (main dashboard)
  - auth-related pages (`login`, `signup`, `verify-email`, `reset-password`, callback)
  - `landing/page.tsx`, `settings/page.tsx`, `cli/page.tsx` redirect
- Components in `components/` including incident table/detail/toolbar, repository/scan views, command menu, nav, onboarding, settings/alert modals, skeletons
- Context providers: `context/AuthContext.tsx`, `context/ToastContext.tsx`
- Client utilities/contracts: `lib/api.ts`, plus helper modules
- Middleware: `middleware.ts`
- Public assets: `public/*`

---

## Implementation Limitations and Repository Discrepancies

This section distinguishes implemented code from documentation/spec claims.

1. **Repository owner typo in clone/install references**
   - Existing docs/scripts reference `Ilyan321/aegis-platform` (missing `k`) in:
     - prior README clone command
     - `install.sh` (`REPO="Ilyan321/aegis-platform"`)
     - `install.ps1` (`$Repo = "Ilyan321/aegis-platform"`)

2. **CLI source path referenced but absent**
   - Installers reference `github.com/<repo>/apps/cli/cmd/aegis@latest`
   - Current repo has no `apps/cli/` directory

3. **Bundled CLI binary claim not evidenced in current tree**
   - Prior README mentions `apps/api/bin/aegis`
   - Current repository listing does not include `apps/api/bin/`
   - Worker has fallback path logic for that location but must usually use external binary install

4. **Duplicate Render manifests with divergent definitions**
   - `render.yaml` (root) and `deploy/render.yaml` both exist
   - They differ in env vars and service layout (root includes `SECRET_KEY`/Google vars; deploy includes optional worker service)

5. **Duplicate Vercel manifests**
   - Both `deploy/vercel.json` and `apps/web/vercel.json` exist with same framework/build/install shape

6. **Spec/document features vs current implementation**
   - `PRD.md` and `AEGIS_CLI_SPEC.md` contain broad architectural goals and future/contractual claims
   - Some claims are not directly verifiable in current repository code (e.g., local CLI internals and rule engine behavior), because CLI source is not included here

7. **SIEM utilities are present but not wired in primary request flows**
   - `apps/api/app/services/siem.py` exists
   - Reviewed endpoint/worker paths do not show direct SIEM dispatch invocation

8. **Potentially environment-sensitive defaults in config**
   - `settings` in `apps/api/app/core/config.py` contain development defaults for secrets/URLs; production safety depends on deployment env overrides

---

## Research-Ready Summary for NotebookLM / Paper Drafting

For a paper grounded in this repository, the strongest evidence-backed themes are:

1. **Asynchronous webhook ingestion architecture**
   - Fast API acceptance path + Celery offload + Redis idempotency locks
2. **Incident lifecycle state machine**
   - OPEN → RESOLVED / DISMISSED with REGRESSION reopening
3. **Tamper-evident audit design**
   - Hash-chained audit entries (`previous_hash` / `entry_hash`)
4. **Masked-token-based incident identity strategy**
   - Blind index and deterministic fingerprinting over normalized identifiers
5. **Hybrid auth/session controls**
   - PBKDF2 passwords + custom JWTs + refresh rotation + revocation checks
6. **Operational duality**
   - Dashboard/API control plane plus external CLI dependency model
7. **Practical deployment reality**
   - Multiple manifests and documentation drift that should be normalized for production clarity

This handoff is intended to be self-contained for NotebookLM ingestion without relying on unstated external context.
