# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
A custom, personal digital advent calendar web application built as a gift. It displays 24 clickable doors that unlock sequentially in December, revealing private photos, notes, media, and gift card codes.

## Tech Stack & Architecture
- **Frontend & API:** Next.js (App Router) with React and TypeScript, hosted on Vercel.
- **Database:** Amazon DynamoDB (single-table design storing 24 door items with static content and dynamic `isOpened` state).
- **Media Storage:** Private Amazon S3 bucket accessed exclusively via short-lived server-generated S3 Presigned URLs.
- **Authentication:** Lightweight passphrase gate validated via server-side API and stored in an `HttpOnly` session cookie.
- **CI/CD:** Vercel handles frontend/API deployments automatically on push to `main`. GitHub Actions (`.github/workflows/deploy.yml`) runs linting/typechecking and syncs `./calendar-assets` to S3 using AWS OIDC keyless auth.

## Directory Structure
```text
├── app/
│   ├── api/
│   │   ├── auth/route.ts         # Validates passphrase and sets HttpOnly cookie
│   │   ├── doors/route.ts        # GET all 24 doors (sanitized: strips locked content)
│   │   └── doors/[day]/route.ts  # POST/GET to unlock a door, mark isOpened=true, return presigned S3 URL
│   ├── page.tsx                  # Main 24-door responsive calendar grid
│   └── layout.tsx                # Root layout & global styles
├── components/
│   ├── DoorCard.tsx              # Individual door UI (locked, unlocked/unopened, opened states)
│   ├── SurpriseModal.tsx         # Modal displaying revealed note, S3 media, or gift code
│   └── PassphraseGate.tsx        # Simple login screen if unauthenticated
├── lib/
│   ├── aws.ts                    # Singleton AWS SDK v3 clients (DynamoDBDocumentClient, S3Client)
│   └── date.ts                   # Server-side timezone-aware date validation logic
├── scripts/
│   └── seed.ts                   # Local TypeScript script to populate 24 DynamoDB items
└── calendar-assets/              # Local media files synced to S3 (keep repo private)
```

## DynamoDB Schema (`AdventCalendarDoors`)
- **Partition Key:** `day` (Number, `1` through `24`)
- **Static Attributes (seeded ahead of time):**
  - `unlockDate` (String, ISO format e.g. `"2026-12-01"`)
  - `title` (String)
  - `message` (String)
  - `mediaType` (String: `"image" | "video" | "audio" | "none"`)
  - `s3Key` (String, optional path inside S3 bucket e.g. `"day-01/photo.jpg"`)
  - `giftCode` (String, optional secret string)
- **Dynamic Attributes (updated at runtime):**
  - `isOpened` (Boolean, defaults to `false`)
  - `openedAt` (String, ISO timestamp, set when first opened)

## Security & Business Logic Rules
1. **Never trust client dates:** All date unlock checks MUST happen server-side in `/api/doors` and `/api/doors/[day]` using a fixed target timezone (e.g., `America/New_York` or `America/Los_Angeles`).
2. **Never leak locked surprises:** `GET /api/doors` must strip `message`, `s3Key`, `giftCode`, and `title` from any item whose `unlockDate` is in the future before returning JSON to the client.
3. **Private S3 bucket only:** Never make the S3 bucket public. Generate temporary `GetObjectCommand` presigned URLs (15-minute expiry) on demand when an unlocked door is opened.
4. **Keep it simple (V1 philosophy):** Avoid unnecessary abstractions, ORMs, or multi-service complexity. Use `@aws-sdk/client-dynamodb`, `@aws-sdk/lib-dynamodb`, `@aws-sdk/client-s3`, and `@aws-sdk/s3-request-presigner`.

## Environment Variables
Required in `.env.local` and Vercel Project Settings:
- `AWS_REGION`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `DYNAMODB_TABLE_NAME`
- `S3_BUCKET_NAME`
- `CALENDAR_PASSPHRASE`
- `CALENDAR_TIMEZONE` (e.g., `"America/New_York"`)

## Common Commands
- `npm run dev` — Start local Next.js development server
- `npm run lint` — Run ESLint and TypeScript checks (must pass before pushing to `main`)
- `npx tsx scripts/seed.ts` — Seed/update the 24 static door items in DynamoDB
