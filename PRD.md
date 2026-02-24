# Party QR — Product Requirements Document

## Overview

Build a Next.js application that manages a simple QR-code-based club entry system.
**PayloadCMS 3.x** is embedded directly into the Next.js app and serves as the admin backend — handling the database, user management, code imports, and the bouncer scanner view.

The app has three distinct surfaces:

1. **Guest page** (`/?code=<value>`) — public, unauthenticated. Guests activate their pass and receive a QR code.
2. **Payload Admin panel** (`/admin`) — authenticated. Admins manage codes, import CSVs, and view usage stats.
3. **Bouncer scanner view** (custom Payload admin view, e.g. `/admin/scanner`) — authenticated Payload user. Bouncers scan QR codes and confirm entry.

---

## Tech Stack

| Concern | Choice |
|---|---|
| Framework | Next.js 15 (App Router) |
| CMS / Admin | Payload CMS 3.x (embedded in Next.js via `@payloadcms/next`) |
| Styling / UI | shadcn/ui + Tailwind CSS — minimalist design |
| Database (production) | Neon (PostgreSQL) — `@payloadcms/db-postgres`, `DATABASE_URL` env var |
| Database (local dev) | SQLite — `@payloadcms/db-sqlite`, no extra infrastructure needed |
| Migrations | Payload handles schema generation and migrations automatically |
| QR code generation | `react-qr-code` |
| QR code scanning | `html5-qrcode` (camera-based, mobile-friendly) |
| Hosting | Vercel |

> **Note:** Drizzle ORM is not needed. Payload manages the database schema entirely. The `entry-codes` collection is defined in Payload config and Payload creates/migrates the table.

---

## Payload Collections & Config

### Collection: `users`

Built-in Payload `Users` collection. Used for admin staff and bouncers.

- Fields: `email`, `password` (default Payload auth fields)
- Roles are not needed for MVP — any authenticated user can access all admin views.

### Collection: `entry-codes`

| Field | Type | Notes |
|---|---|---|
| `code` | text, required, unique | Raw code string (e.g. `abc123qwe456`) |
| `activatedAt` | date (timestamp), optional | Set when guest clicks "Claim Entry Pass" |
| `usedAt` | date (timestamp), optional | Set when bouncer confirms entry |

Access control:
- Read / Update: authenticated users only (admin + bouncers).
- Create / Delete: authenticated users only.

Admin list view should display: `code`, `activatedAt`, `usedAt`, and a computed "Status" column showing one of: `Unused` / `Active` / `Expired` / `Used`.

### Custom Payload Admin Views

#### 1. CSV Import View (`/admin/import`)

A custom React component registered as a Payload admin view.

- Renders a file input accepting `.csv` files.
- CSV format: one code per row, single column, no header required (header row is ignored if non-numeric).
- On submit:
  - Reads the file client-side.
  - Calls a Next.js API route `POST /api/import` with the list of codes.
  - The API bulk-inserts codes into the `entry-codes` collection using Payload's local API (`payload.create()` in a loop, or a raw DB upsert).
  - Duplicates are silently ignored (`ON CONFLICT DO NOTHING`).
  - Returns `{ imported: N, skipped: M }`.
- Displays the result summary to the admin.

#### 2. Bouncer Scanner View (`/admin/scanner`)

A custom React component registered as a Payload admin view. Requires Payload authentication (automatically enforced by Payload's admin wrapper).

- Full-width camera view powered by `html5-qrcode`.
- Continuously reads from the camera. On QR code detection:
  - Calls `POST /api/verify` with the scanned code value.
  - Displays one of the result states below.
  - After showing the result, resets to camera view (either automatically after a delay or when the bouncer taps "Scan again").

##### Scan Result States

| State | Condition | UI |
|---|---|---|
| **Valid** | Code exists, `activatedAt` set, < 15 min elapsed, `usedAt` is null | Full-screen green flash + **"Confirm Entry"** button |
| **Expired** | `activatedAt` set, ≥ 15 min elapsed | Red/orange alert: "This pass has expired." |
| **Already Used** | `usedAt` is not null | Red alert: "This pass has already been used." |
| **Not Activated** | Code exists, `activatedAt` is null | Yellow warning: "This pass has not been activated yet." |
| **Invalid** | Code not in database | Red alert: "Unknown QR code." |

- When the bouncer presses **"Confirm Entry"**:
  - Calls `POST /api/use` with the code.
  - API sets `usedAt = NOW()` atomically (only if `usedAt` is still null).
  - Shows brief success feedback, then resets to camera view.

---

## Guest Page (`/?code=<value>`)

Standard Next.js page — **not** inside the Payload admin, no authentication required.

### Page Load Behavior

- Read `code` from the URL query string.
- Fetch code status from `GET /api/code?code=<value>` (or use a Server Component to query Payload local API directly).
- If code **not found** → render Next.js 404 page.
- If code **found and not yet activated** → show welcome screen with **"Claim Entry Pass"** button.
- If code **already activated** → skip the button, go directly to QR code display (supports page refresh).

### Claiming the Pass

- Guest clicks **"Claim Entry Pass"**.
- Client calls `POST /api/activate` with the code.
- API sets `activatedAt = NOW()` using an atomic update (only if `activatedAt` is still null — prevents double-activation from double-clicks or concurrent requests).
- On success, the page transitions to the QR code display.

### QR Code Display

- Renders a QR code whose **encoded value is exactly the raw code string** (e.g. `abc123qwe456`).
- Shows the text: "Show this QR code to the bouncer at the entrance."
- Shows a note: "Valid for 15 minutes from the moment you claimed it."
- Design must be mobile-friendly; QR code should be large (min 256×256px).

---

## API Routes

All routes return JSON. Use appropriate HTTP status codes.

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/api/activate` | none | Sets `activatedAt` for a code (atomic, idempotent) |
| `POST` | `/api/verify` | none | Returns status of a code: `valid` / `expired` / `used` / `not-activated` / `invalid` |
| `POST` | `/api/use` | none | Sets `usedAt` atomically (only if not already used) |
| `POST` | `/api/import` | Payload session cookie | Bulk-inserts codes from a parsed CSV payload |

> `/api/verify` and `/api/use` are called from the bouncer scanner which is inside the authenticated Payload admin, but the routes themselves don't need to re-validate the session for MVP. If security is a concern later, add Payload session verification middleware.

---

## Environment Variables

```bash
# Production — Neon PostgreSQL
DATABASE_URL=postgresql://user:pass@host/dbname?sslmode=require

# Payload
PAYLOAD_SECRET=some-long-random-secret

# App
NEXT_PUBLIC_APP_URL=https://your-app.vercel.app
```

For **local development**, set `DATABASE_URL` to a relative SQLite path (e.g. `file:./dev.db`) and use `@payloadcms/db-sqlite`. Payload config should switch adapters based on the `DATABASE_URL` scheme:

```ts
// payload.config.ts
const db = process.env.DATABASE_URL?.startsWith('file:')
  ? sqliteAdapter({ client: { url: process.env.DATABASE_URL } })
  : postgresAdapter({ pool: { connectionString: process.env.DATABASE_URL } });
```

---

## Project Structure (guidance)

```
/
├── app/
│   ├── (payload)/          # Payload admin routes (auto-generated by @payloadcms/next)
│   │   └── admin/
│   │       ├── [[...segments]]/
│   │       ├── import/     # Custom CSV import view
│   │       └── scanner/    # Custom bouncer scanner view
│   ├── api/
│   │   ├── activate/route.ts
│   │   ├── verify/route.ts
│   │   ├── use/route.ts
│   │   └── import/route.ts
│   └── page.tsx            # Guest entry pass page (/?code=...)
├── payload.config.ts
├── collections/
│   ├── Users.ts
│   └── EntryCodes.ts
└── components/
    ├── QRCodeDisplay.tsx
    ├── ScannerView.tsx
    └── CsvImportForm.tsx
```

---

## Non-Goals (MVP)

- Role-based access control within Payload (admin vs. bouncer roles).
- Sending emails or generating invite links (handled by an external system).
- Multi-event support (single flat collection of codes).
- Real-time push updates on the scanner (polling on each scan is sufficient).
- Rate limiting on public API routes.
