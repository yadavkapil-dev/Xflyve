# XFlyve

A logistics operations tool for small trucking/transport companies — the kind of business running a handful of trucks and drivers off spreadsheets, group chats, and paper proof-of-delivery slips. XFlyve gives an admin one place to assign jobs to drivers and trucks, track proof-of-delivery documents through an approval step, keep NHVR-relevant compliance records (work diaries, work logs), see what's ready to invoice, and get real-time visibility into what's happening across the fleet — while drivers get a simple mobile-friendly view of their own jobs and paperwork.

It is not a TMS/ERP replacement. It doesn't do route optimization, GPS tracking, payroll, or actual invoice generation — see [Known Limitations](#known-limitations--honest-future-improvements) for the deliberate scope boundary.

This is a full-stack portfolio/production-readiness project: real auth, real role-based access control, a real CI/CD pipeline with automated tests, and an audited security pass — built to demonstrate how a small operational tool gets built and hardened end-to-end, not to claim adoption or scale it hasn't been tested at.

## Live demo

- **App (frontend):** https://xflyve.vercel.app
- **API (backend):** https://xflyve.onrender.com
- **API docs (Swagger):** https://xflyve.onrender.com/api-docs

### Try It Out

Log in with the demo accounts below to see both portals without registering — one is an admin, one is a driver:

Sign in as the admin to create and assign a job, review PODs, and see the invoicing and activity views; sign in as the driver to start and complete that job and upload paperwork.

| Role | Email | Password |
|---|---|---|
| Admin | `admin@example.com` | `admin123` |
| Driver | `marcus.chen@example.com` | `Interview123!` |

The backend runs on Render's free tier, so it spins down after a period of inactivity. The first request after it's been idle can take up to a minute while the service wakes back up — a slow initial load (or a login that seems to hang the first time) is expected, not a broken deploy. Requests after that are fast.

## Architecture overview

- **Frontend** — React 19 (Vite), Material UI v7, React Router v7. Talks to the backend over Axios (REST) and Socket.IO (real-time). Deployed to Vercel.
- **Backend** — Node.js + Express 5 + Mongoose 8, exposing a REST API. Deployed to Render.
- **Database** — MongoDB. Local dev runs it via Docker (`docker-compose.yml`, `mongo:6.0`); production points at a real MongoDB deployment via `MONGO_URI`.
- **File storage** — Cloudinary holds uploaded PDFs (PODs, work diary pages). Uploads go through Multer (in-memory, no disk writes) with a magic-byte check on the actual file bytes (not just the client-supplied MIME type) before ever reaching Cloudinary.
- **Real-time layer** — Socket.IO, authenticated with the same JWT verification as the REST API. Each connection joins a private room keyed to the server-verified user ID (never client-supplied), so notifications can only ever reach their intended recipient.
- **AI Assistant** — Backed by a real external LLM via [OpenRouter](https://openrouter.ai) (a free-tier model, called through a plain `fetch` wrapper — no vendor SDK), not a hardcoded/rule-based chatbot. It runs a standard tool-calling loop against six read-only tools that wrap the app's own controllers; the model never sees raw database access, and every tool call is re-validated against the requesting user's real role server-side even if the model tries something it shouldn't.
- **Email** — [Resend](https://resend.com), used for a handful of transactional emails (password reset, job assigned, POD rejected). Entirely optional and fire-and-forget — the app functions normally with no key configured, it just skips sending.
- **Observability** — Winston for structured backend logging, Sentry (optional, both frontend and backend) for error monitoring.

## Core features

**Jobs lifecycle** — Admin creates a job (title, pickup/delivery, driver, truck, date, local/interstate type), assigns it to a driver and a truck for that day (with atomic double-booking protection — a truck can't be assigned to two jobs on the same day, enforced at the database level via a unique index, not just application logic). The driver starts and completes the job from their own view. Every transition (created, assigned, started, completed) is logged to a per-job activity trail and triggers an in-app notification.

**Proof of Delivery (PODs)** — Driver uploads a PDF POD tied to a job. Admin reviews and approves or rejects it. Approved PODs feed directly into invoice readiness. Rejected ones notify the driver (in-app and by email) so they know to redo it.

**Work Diary** — Interstate-jobs-only PDF logbook pages (NHVR fatigue/compliance records), uploaded by the driver, filtered and downloadable by an admin by date range and/or driver. There is no approval workflow for these — a submitted diary is just a record on file, not something that gets approved or rejected. Date filtering is scoped to the actual trip date, not the (possibly later) upload date, so a diary uploaded a few days late for an earlier trip still shows up correctly in a compliance-date lookup.

**Work Logs** — A structured daily entry (hours/km for local jobs, start/end odometer readings for interstate jobs) the driver fills in themselves, tied to a specific job. Like Work Diary, there is no approval workflow — this was a deliberate simplification; a submitted log is just a record.

**Invoicing readiness** — A job is "ready to invoice" once it's marked completed and has at least one approved POD (work diaries/logs don't gate this — only the POD does). The Invoicing page lists these jobs with a single "Mark as Invoiced" action per job: a confirmation dialog, then a `PUT /api/jobs/:jobId { invoiceStatus: "invoiced" }` call that removes it from the ready list. This is a status flag, not an invoicing feature — it does not calculate amounts, generate a document, or send anything to anyone.

**Real-time notifications** — Job assignment/updates, POD submission/approval/rejection, work diary/log submission, and job start/completion all push a live in-app notification over Socket.IO to the right recipient (drivers see their own; admin-relevant events fan out to every active admin). No email digest or push notifications — in-app only, plus the few transactional emails noted above.

**AI Assistant** — A chat widget (both landing page and inside the app) backed by a real LLM with six role-gated, read-only tools: today's jobs (driver), available trucks, pending POD approvals (admin), rejected documents (admin — PODs only, since diaries/logs have no rejection concept to query), invoice-ready jobs (admin), and a daily operations summary (admin). No conversation memory across requests, no streaming, no write/mutation tools — it can look things up, not change anything.

**Activity audit trail** — Every job-related action (created, assigned, started, completed, POD submitted/approved/rejected, diary/log submitted) is logged to an append-only, write-once collection and shown as a per-job timeline on the admin Jobs page. Admin-only; there's no cross-job activity feed, just the one job you're looking at.

**Security** — JWT bearer-token auth (7-day tokens, no refresh rotation — see limitations), bcrypt password hashing (cost factor 12), role-based access control (`admin`/`driver`, enforced both at the route-middleware level and again inside controllers for per-resource ownership), a magic-byte file-signature check on uploads (not just trusting the client's declared MIME type), Helmet security headers, a configurable CORS whitelist, and layered rate limiting — a general per-IP limit, a stricter one on login/password-reset routes, and a per-user limit on the AI chat endpoint (protecting the shared OpenRouter quota, not the IP).

**CI/CD** — GitHub Actions runs on every push/PR to `main` and `production-readiness`: backend unit + integration tests (against an isolated in-memory MongoDB, never a real database), frontend lint + unit tests + production build, one full Playwright end-to-end workflow test (admin creates a job → driver starts/completes it → uploads a POD → admin approves it → invoice-ready), then a Docker build for both services. On an actual push to `main`, it triggers real deploys (Render for the backend, Vercel for the frontend) and polls both until they're confirmed healthy before finishing.

## Setup / running locally

**Requirements:** Node.js ≥18, a MongoDB instance (local via Docker, or your own).

```bash
git clone https://github.com/yadavkapil-dev/Xflyve.git
cd Xflyve

# Backend
cd backend
npm install
cp .env.example .env   # then fill in the values below
npm run dev            # starts on http://localhost:3001

# Frontend (separate terminal)
cd xflyve-frontend
npm install
cp .env.example .env   # VITE_API_URL should already point at the backend above
npm run dev             # starts on http://localhost:5173
```

Optional: `docker-compose up` at the repo root spins up MongoDB + both services in containers instead.

### Environment variables

**Backend (`backend/.env`):**

| Variable | Required? | Notes |
|---|---|---|
| `MONGO_URI` | Always | App won't start without it |
| `JWT_SECRET` | Always | App won't start without it |
| `CLOUDINARY_CLOUD_NAME` / `CLOUDINARY_API_KEY` / `CLOUDINARY_API_SECRET` | Production only | Optional for local dev — POD/diary uploads just won't work without them |
| `CORS_WHITELIST` and/or `FRONTEND_URL` | Production only | At least one required so the deployed frontend can call the API |
| `OPENROUTER_API_KEY` | **Effectively required for the AI Assistant** | **Not currently listed in `.env.example` — this is a known gap.** Without it, every AI Assistant request fails (gracefully — no crash, just no useful reply); everything else in the app works fine. |
| `RESEND_API_KEY` | Optional | Also not in `.env.example`. Without it, transactional emails are silently skipped — the app degrades gracefully. |
| `PORT` | Optional | Defaults to `3001` |
| `NODE_ENV` | Optional | Defaults to `development` |
| `LOG_LEVEL` | Optional | Defaults to `info` |
| `SENTRY_DSN` / `SENTRY_TRACES_SAMPLE_RATE` | Optional | Sentry fully disabled if unset |

**Frontend (`xflyve-frontend/.env`):**

| Variable | Required? | Notes |
|---|---|---|
| `VITE_API_URL` | Yes | Backend API base URL |
| `VITE_SENTRY_DSN` / `VITE_SENTRY_TRACES_SAMPLE_RATE` | Optional | Sentry fully disabled if unset |

### Running tests

```bash
cd backend && npm test              # unit + integration, with coverage
cd xflyve-frontend && npm test      # vitest
cd e2e && npm test                  # Playwright, boots its own backend+frontend+DB
```

## Known limitations / honest future improvements

**Security**
- **No refresh-token rotation** — a single 7-day JWT, no refresh flow, no session management, no MFA/SSO. This was a deliberate scope decision during the security-hardening pass, not an oversight, but it's a real gap for anything beyond a small trusted team.
- **`exportDriversExcel` (admin Excel export) and `backend/scripts/exportUsersCsv.js` both export every account regardless of role** — since driver and admin accounts share one Mongoose collection, both currently include admin accounts (and, for the Excel export, archived drivers) in what's meant to be a drivers-only export. The equivalent list endpoint (`getAllDrivers`) already filters correctly; these two didn't get the same fix. Confirmed still present as of this write-up.
- Backend has no ESLint configured (frontend does). Known, not yet addressed.

**Pagination gaps** — a few endpoints return their entire result set with no paging, which is fine at small scale but won't stay fine indefinitely:
- A driver's own jobs and own work logs (`getMyJobs`, `getAssignedJobs`, `getMyLogs`, `getLogsByDriver`)
- Truck assignment history (`getAllAssignments`)
- The in-app notification list is capped at the newest 20 per session client-side (the backend supports real pagination; the frontend just doesn't call for more)

**By design, not gaps:**
- **No invoicing or payroll calculation.** "Mark as Invoiced" is a status flag an admin sets manually — XFlyve doesn't compute amounts, generate an invoice document, or integrate with any accounting system.
- **No multi-tenant support.** This is a single-company tool — there's no concept of separate organizations sharing one deployment. Every admin sees every driver, job, and truck in the database.
- **No GPS/live location tracking, no route planning/optimization.**

**Other confirmed-current gaps:**
- `Job.podUrl` is a dead schema field — fully superseded by the `Job.podIds` → `JobPod` relationship, but never removed.
- `Truck.assignedDriver` is backend-wired (readable/writable via the API) but the frontend never uses it — real driver-to-truck assignment happens entirely through the separate daily `TruckAssignment` collection instead.
- The Activity audit trail has no cross-job/admin-wide view — only a per-job timeline.

## Tech stack

**Backend:** Node.js, Express 5, Mongoose 8 (MongoDB), JWT (`jsonwebtoken`), `bcryptjs`, Helmet, `express-rate-limit`, `express-validator`, Multer, Cloudinary SDK, Socket.IO, Winston, `@sentry/node`, Resend, ExcelJS, `archiver`, Swagger (`swagger-ui-express`) for API docs. AI Assistant integration is a hand-written `fetch` wrapper around OpenRouter's chat-completions API (no vendor SDK). Tested with Jest, Supertest, and `mongodb-memory-server`.

**Frontend:** React 19, Vite 7, React Router 7, Material UI 7 (+ Emotion, `@mui/x-charts`, `@mui/x-date-pickers`), Axios, Socket.IO client, `dayjs`, `@sentry/react`. Tested with Vitest and React Testing Library; linted with ESLint 9 (flat config) including `eslint-plugin-jsx-a11y`.

**End-to-end:** Playwright, run against a real (isolated, in-memory) backend and a real Vite dev server.

**Infra:** Docker (multi-stage builds, non-root containers, healthchecks) for both services; `docker-compose` for local dev. Render (backend) + Vercel (frontend) in production, deployed via GitHub Actions on push to `main`, gated behind the full test suite passing first.

## What I Learned

Building and hardening XFlyve reinforced:

- Designing role-based access control that holds up under scrutiny — not just gating routes by role, but checking resource ownership inside controllers, and actually finding (and fixing) the one admin route that had been missed.
- The difference between "the feature works" and "the feature is safe" — validating uploaded files by their actual bytes instead of a client-supplied MIME type, and scoping rate limits to what's genuinely worth protecting (a shared AI provider quota, not just an IP address).
- Real-time features done properly — Socket.IO rooms keyed to server-verified identity rather than anything a client could spoof.
- Integrating an LLM without making it a trust boundary — tool-calling restricted to read-only, role-gated functions that re-check the caller's real permissions server-side on every call.
- Writing tests that catch real regressions instead of just passing — tracing an end-to-end test failure back to a legitimate, correct backend change elsewhere, rather than assuming the fix was the bug.
- Keeping documentation honest over time — auditing this README against the actual current code before rewriting it, instead of letting stale claims sit uncorrected.

## Author

**Kapil Yadav**

- Portfolio: https://kapilyadav.dev
- LinkedIn: https://linkedin.com/in/yadav-kapil
- GitHub: https://github.com/yadavkapil-dev
