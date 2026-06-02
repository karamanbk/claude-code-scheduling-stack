# 02 — Project Setup

This guide bootstraps the project: Next.js, Prisma, the database connection, Tailwind, and environment variables. By the end you should have an app that boots and connects to Postgres.

## 1. Create the Next.js app

```bash
npx create-next-app@latest scheduling-platform \
  --typescript --app --tailwind --eslint --src-dir
cd scheduling-platform
```

Choose the App Router. Keep the `src/` directory — every guide assumes `src/app/...`.

## 2. Set up route groups

Create the empty route groups described in the architecture guide:

```
src/app/
  (dashboard)/
  (booking)/
  (auth)/
  api/
```

Route groups (parenthesized folders) let you share a layout without affecting the URL. The dashboard group gets an auth-protected layout; the booking group stays public.

## 3. Add Prisma

```bash
npm install prisma @prisma/client
npx prisma init
```

This creates `prisma/schema.prisma`. Point the datasource at PostgreSQL. With recent Prisma versions, connection URLs live in a `prisma.config.ts` rather than inline in the schema; follow the version's docs. The key idea:

- `DATABASE_URL` — pooled connection used by the app at runtime
- `DIRECT_URL` — direct connection used for migrations

Create a singleton Prisma client so dev hot-reload doesn't open new connections each time:

```ts
// src/lib/prisma.ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient };

export const prisma = globalForPrisma.prisma ?? new PrismaClient();

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

## 4. Environment variables

Create `.env.local`. Use placeholders — never commit real secrets.

```bash
# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXTAUTH_SECRET=generate-a-long-random-string
NEXTAUTH_URL=http://localhost:3000

# Database
DATABASE_URL=postgresql://USER:PASSWORD@HOST:5432/DB
DIRECT_URL=postgresql://USER:PASSWORD@HOST:5432/DB

# OAuth (fill in as you wire each provider)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
MICROSOFT_CLIENT_ID=
MICROSOFT_CLIENT_SECRET=
MICROSOFT_TENANT_ID=common

# LLM (AI routing, smart forms, enrichment)
LLM_API_KEY=

# Email (SMTP)
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
SMTP_FROM_EMAIL=notifications@example.com
SMTP_FROM_NAME=Scheduling Platform

# Background jobs
CRON_SECRET=generate-another-random-string
```

> **Tip:** Add a short script or check that fails loudly if a required env var is missing in production. A missing `NEXTAUTH_SECRET` or `DATABASE_URL` causes confusing downstream errors.

## 5. Tailwind and base styles

`create-next-app` wires Tailwind. Define your design tokens (colors, fonts, radii) in your global CSS so the booking pages and dashboard share one look. Keep a small set of UI primitives (`Button`, `Input`, `Card`, `Badge`, `Select`) under `src/components/ui/` — you'll reuse them everywhere.

## 6. First migration & boot

Add an initial `Settings` model (placeholder for now) and run a migration, then start the app:

```bash
npx prisma migrate dev --name init
npm run dev
```

Add a trivial health endpoint to confirm the database connection:

```ts
// src/app/api/health/route.ts
import { NextResponse } from "next/server";
import { prisma } from "@/lib/prisma";

export async function GET() {
  await prisma.$queryRaw`SELECT 1`;
  return NextResponse.json({ ok: true });
}
```

Hit `http://localhost:3000/api/health` — a `{ ok: true }` means you're ready for the data model.

## Working with Claude Code

Ask Claude Code to:
1. Scaffold the route groups and the Prisma singleton.
2. Add the health endpoint and confirm it returns `ok`.
3. Stop there. Don't let it build the whole schema yet — the next guide does that deliberately.

Commit this skeleton before moving on.
