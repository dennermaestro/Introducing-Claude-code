# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
EventHub is a full-stack event ticket booking platform built for QA training. Users register/log in, browse events, book tickets, manage bookings, and create their own events. Each user operates in an isolated sandbox (their own bookings and user-created events).

- **Frontend** (`frontend/`): Next.js 14 App Router, React 18, TypeScript + JSX, Tailwind, React Query v5, Axios — port 3000
- **Backend** (`backend/`): Express, Prisma 5, MySQL 8, JWT auth (7-day expiry, bcryptjs), Swagger UI at `/api/docs` — port 3001
- **Tests** (`tests/`): Playwright E2E, Chromium only

## Commands
Run from the repo root unless noted.

```bash
npm run setup        # npm install in backend/ and frontend/
npm run db:push      # push Prisma schema to MySQL (non-interactive)
npm run migrate      # prisma migrate dev (interactive — needs a real terminal)
npm run seed         # insert the 10 static seeded events
npm run dev          # backend (nodemon) + frontend (next dev) via concurrently
npm run build        # next build of the frontend
npm run lint         # next lint in frontend/

npm test                                              # all Playwright tests
npx playwright test tests/<file>.spec.js --reporter=line   # one file
npx playwright test -g "TC-001"                       # one test by title
npm run test:ui      # Playwright UI mode
npm run test:report  # open last HTML report
docker compose up --build   # run the Playwright suite in the official Playwright image
```

CI-equivalent checks (`.github/workflows/ci.yml`):
```bash
cd backend && npx prisma validate && npx prisma format   # CI fails if format changes the schema
cd frontend && npx tsc --noEmit && npm run build
```

Env files: `backend/.env` (`DATABASE_URL`, `PORT`, `NODE_ENV`, `CORS_ORIGIN`) — see `backend/.env.example`; `frontend/.env.local` (`NEXT_PUBLIC_API_URL=http://localhost:3001/api`).

## Tests run against the hosted app, not localhost
`playwright.config.ts` sets `baseURL` to `https://eventhub.rahulshettyacademy.com`, and specs also hard-code `BASE_URL` to the same host. Running the suite does **not** require the local servers, and local code changes are **not** exercised by it unless the URL is changed. Tests run serially (`fullyParallel: false`, no retries) because they share one test account's sandbox.

Test account: `rahulshetty1@gmail.com` / `Magiclife1!`

## Backend architecture
`server.js` (DB connect, graceful shutdown) → `app.js` (CORS, routes, Swagger) → `src/`, layered as:

Routes (`src/routes`, with Swagger JSDoc) → validators (`express-validator`) → Controllers (thin HTTP) → Services (business rules, Prisma transactions) → Repositories (pure Prisma) → MySQL.

- Auth: `authRoutes`/`authService` issue JWTs; `middleware/authMiddleware.js` attaches the user and scopes queries to `userId`.
- Errors: services throw domain errors from `src/utils/errors.js` (`NotFoundError`, `InsufficientSeatsError`, `ValidationError`, …); `middleware/errorHandler.js` maps them to HTTP responses. Throw these rather than writing responses in services.
- Data model (`prisma/schema.prisma`): `User`, `Event` (`userId` nullable — null means a seeded static event), `Booking` (belongs to a user and an event; cascades on delete).

## Frontend architecture
- `app/` pages: home, `events` (+ `[id]` detail/booking), `bookings` (+ `[id]`), `admin/events`, `admin/bookings`, `login`, `register`.
- Data flow: page → React Query hook in `lib/hooks/` (`useEvents`, `useBookings`, `useAuth`) → API module in `lib/api/` → Axios client with interceptors (JWT header). `lib/providers.jsx` wires React Query + Toast context.
- `lib/api/` contains both `.js` modules (`authApi`, `eventsApi`, `bookingsApi`, `client.js`) and `.ts` ones (`client.ts`, `events.ts`, `bookings.ts`, `index.ts`). The hooks import the `.js` modules, so edit those.
- Key UI elements carry `data-testid` attributes (list in README.md and `.claude/skills/eventhub-domain/ui-selectors.md`). Keep them stable — tests depend on them.

## Key business rules
- Max 6 user-created events and max 9 bookings per user; overflow prunes the oldest (FIFO).
- Booking ref format is `<T>-XXXXXX`, where `<T>` is the event title's first letter, uppercased (`randomRef` in `bookingService.js`). README's `EVT-XXXXXX` is outdated.
- Booking decrements available seats atomically; cancellation restores them.
- Refund eligibility (client-side): 1 ticket = eligible, >1 ticket = not eligible.
- Accessing another user's booking returns "Access Denied".
- Seeded static events are immutable.

## Project skills (`.claude/skills/`)
- `eventhub-domain` — business rules, API reference, user flows, UI selectors. Consult before writing tests or answering domain questions.
- `playwright-best-practices` — locator/assertion/structure standards for tests.
- `generate-tests`, `review-tests`, `create-scenarios`, `test-strategy` — QA workflows (write Playwright tests, review them, produce scenario docs in `docs/`, assign scenarios to pyramid layers).

## Test conventions
- Files: `tests/<feature-name>.spec.js`, plain JavaScript; test titles prefixed with scenario IDs (`TC-001: …`) from `docs/test-scenarios.md`.
- Locator priority: `data-testid` > role > label/placeholder > id > CSS class.
- No `page.waitForTimeout()`; wait with web-first assertions (`expect(...).toBeVisible()`).
- Each test is self-contained: login → action → assert.

## Other repo contents
- `Introducing-Claude-code/` is a separate nested git clone (the user's fork), not part of this app — don't edit it as part of EventHub changes.
- Remotes: `origin` = upstream course repo (rahulshettyacademy/eventhub), `mine` = the user's fork (dennermaestro/Introducing-Claude-code). Push to `mine`.
- `deploy.yml` deploys `main` to a production server over SSH; `playwright.yml` runs the E2E suite on push to `main`.
