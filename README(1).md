# Clinic Patient Management System — Starter

A working full-stack scaffold: **Node.js/Express + TypeScript + PostgreSQL (Prisma) + JWT/RBAC auth**
on the backend, **React + TypeScript + Vite + Tailwind** on the frontend.

This is a foundation, not a finished product — it covers auth, patients, appointments, and
billing end-to-end (API + basic UI) so you have a real pattern to extend into the rest of the
plan (notifications, reporting, compliance hardening, etc).

## What's implemented

- **Auth**: JWT login, RBAC (`ADMIN`, `DOCTOR`, `NURSE`, `RECEPTIONIST`), password hashing (bcrypt),
  admin-only staff creation, audit log on login.
- **Patients**: register, search, update, soft-delete, medical record notes.
- **Appointments**: create/update/cancel, provider double-booking prevention.
- **Billing**: invoice generation with line items, payment recording, auto status transitions.
- **Security basics**: helmet, CORS, rate limiting, centralized error handling, input validation (zod).
- **Frontend**: login page, protected routes, patients list/registration, appointments list —
  wired to the real API via axios.

## What's NOT implemented yet (from your original plan)

Notifications/reminders (email/SMS), reports & exports, insurance claim integrations, payment
gateway integration, data migration tooling, full audit-log UI, localization, and everything
under Testing/Deployment/Marketing/Business sections. The architecture (modular services +
RBAC middleware) is built to make adding these straightforward.

## Project structure

```
clinic-system/
  backend/
    prisma/schema.prisma      # data model
    prisma/seed.ts            # creates initial admin user
    src/
      config/                 # env, prisma client
      middleware/              # auth, RBAC, error handling
      modules/
        auth/                 # login, registration, password change
        patients/              # patient CRUD + medical records
        appointments/          # scheduling + overlap prevention
        billing/                # invoices + payments
      index.ts                 # app entry point
  frontend/
    src/
      api/client.ts            # axios instance w/ JWT interceptor
      context/AuthContext.tsx  # auth state
      pages/                   # Login, Patients, Appointments
      components/              # Layout, ProtectedRoute
```

## Setup

### 1. Database
You need a PostgreSQL instance (local, Docker, or hosted e.g. Supabase/Neon/RDS).

```bash
docker run --name clinic-db -e POSTGRES_PASSWORD=password -e POSTGRES_DB=clinic_db -p 5432:5432 -d postgres:16
```

### 2. Backend

```bash
cd backend
cp .env.example .env        # edit DATABASE_URL and JWT_SECRET
npm install
npx prisma migrate dev --name init   # creates tables
npm run seed                          # creates admin@clinic.local / ChangeMe123!
npm run dev                           # starts API on :4000
```

> **Note on this scaffold's build environment**: I built and type-checked this code in a
> sandboxed container that couldn't reach `binaries.prisma.sh`, so I could not run
> `prisma generate` / `prisma migrate` here to get a 100% end-to-end confirmation. The code is
> syntactically correct and type-checks cleanly against a normal Prisma client — but please run
> `npx prisma migrate dev` yourself as the first real test, and open an issue in your own tracking
> if anything doesn't line up (most likely candidate: minor Prisma API differences if you change
> the version pin in `package.json`).

### 3. Frontend

```bash
cd frontend
cp .env.example .env         # points at http://localhost:4000/api by default
npm install
npm run dev                  # starts on :5173
```

Log in with `admin@clinic.local` / `ChangeMe123!` — **change this password immediately**
(`POST /api/auth/change-password`), and create real staff accounts via
`POST /api/auth/register` (admin-only).

## Deploying

**Netlify hosts the frontend only.** It's a static/serverless platform — it can't run the
Express backend or hold a database connection. Deploy the two halves separately:

### Frontend → Netlify
This repo includes `netlify.toml` (build settings) and `frontend/public/_redirects`
(SPA fallback, fixes the "Page not found" 404 you get when React Router paths are
requested directly). In Netlify:
1. New site from Git → pick this repo.
2. Build settings are read automatically from `netlify.toml` (base: `frontend`, command:
   `npm run build`, publish: `frontend/dist`). If deploying without Git (drag-and-drop),
   build locally first (`cd frontend && npm run build`) and drag the `dist` folder in —
   but make sure `_redirects` ends up inside `dist` (Vite copies anything in `frontend/public`
   into `dist` automatically on build).
3. Set the environment variable `VITE_API_URL` in Netlify's site settings to your deployed
   backend's URL (e.g. `https://your-api.onrender.com/api`) — this needs to be set *before*
   the build, since Vite bakes `import.meta.env` values in at build time.

### Backend → Render / Railway / Fly.io / a VPS
Any of these work: point them at `backend/`, set `DATABASE_URL` and `JWT_SECRET` as
environment variables, run `npx prisma migrate deploy` (production-safe migration command,
unlike `migrate dev`) as part of the build/release step, then `npm run build && npm start`.
Also add your Netlify domain to `CORS_ORIGIN` in the backend's env so the browser is allowed
to call it.

## Security notes before production

- Change `JWT_SECRET` to a long random value; never commit `.env`.
- Put this behind HTTPS/TLS — never run it over plain HTTP outside local dev.
- The current rate limiter is global; add a stricter one specifically on `/api/auth/login`
  to slow brute-force attempts.
- For real HIPAA-relevant deployments: encrypt PHI at rest (Postgres column-level or disk
  encryption), enforce audit logging on all reads/writes to `Patient` and `MedicalRecord`
  (only login is logged right now), and get a formal security/compliance review — this
  scaffold is a starting point, not a compliance certification.
