---
name: notion-supabase-cms
description: Set up Notion as a headless CMS with Supabase Storage for image hosting in a Next.js project. Solves Notion's expiring signed image URLs by syncing covers and inline images to Supabase. Includes cron sync, manual trigger API route, and ISR revalidation.
---

# Notion + Supabase CMS Setup

Set up Notion as a headless CMS with Supabase Storage for stable image hosting. This architecture prevents Notion's ~1-hour expiring image URLs from breaking production pages.

## Why this architecture

Notion's file-type images (covers, inline blocks) use signed URLs that expire after ~1 hour. Fetching `page.cover` or block image URLs directly will result in broken images on any cached/static page. This skill wires up Supabase Storage as the serving layer so all image URLs are permanent public URLs.

## Prerequisites

- Next.js project (App Router)
- Supabase project with Storage enabled
- Notion workspace with at least one database
- Vercel deployment (for cron support)

---

## Step 1 — Notion database schema

Each content database needs these properties at minimum:

| Property | Type | Notes |
|---|---|---|
| Title | Title | Post title |
| Slug | Rich text | URL-safe identifier, unique per post |
| Status | Select | Values: `Draft`, `Published` |
| Published At | Date | |
| Cover URL | Files & media | Upload cover image here |
| Tags | Multi-select | |
| Category | Select | |
| Summary | Rich text | |

> **Cover URL must be a "Files & media" property**, not the built-in Notion page cover. The sync script reads from this property, not `page.cover`.

---

## Step 2 — Supabase Storage buckets

Create one bucket per content type. Set each bucket to **Public**.

```
work
blog
assets
```

Inside each bucket, the sync script creates these paths automatically:
- `covers/<slug>` — cover images
- `icons/<slug>` — icons (work bucket only)
- `images/<blockId>` — inline content images

---

## Step 3 — Environment variables

Add to `.env.local` and Vercel project settings:

```env
NOTION_TOKEN=
NOTION_WORK_DB_ID=
NOTION_BLOG_DB_ID=
NOTION_ASSETS_DB_ID=

NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

CRON_SECRET=          # random string, used to authenticate Vercel cron calls
REVALIDATE_SECRET=    # random string, used to authenticate manual sync trigger
```

> `SUPABASE_SERVICE_ROLE_KEY` is required to upload files to Storage (bypasses RLS). Never expose this on the client side.

---

## Step 4 — Core library files

### `lib/supabase.ts`

```ts
export type Bucket = 'work' | 'blog' | 'assets'

export function getPublicUrl(bucket: Bucket, path: string): string {
  const base = process.env.NEXT_PUBLIC_SUPABASE_URL!
  return `${base}/storage/v1/object/public/${bucket}/${path}`
}
```

### `lib/notion.ts` (key pattern)

Always use `getPublicUrl()` for image fields — never read `page.cover` or Notion file URLs directly:

```ts
function toPost(page: PageObjectResponse): Post {
  const slug = text(page, 'Slug')
  return {
    coverUrl: getPublicUrl('blog', `covers/${slug}`),
    // ...
  }
}
```

For page blocks (inline images), pass the bucket to `getPageBlocks` so images are synced on render:

```ts
export async function getPageBlocks(pageId: string, bucket?: Bucket) {
  const response = await notion.blocks.children.list({ block_id: pageId, page_size: 100 })
  let blocks = response.results as any[]
  if (!bucket) return blocks
  const { syncBlockImages } = await import('./notion-image-sync')
  return syncBlockImages(blocks, bucket)
}
```

### `lib/notion-image-sync.ts` (key pattern)

`uploadImageToSupabase` always returns the Supabase URL — never falls back to an expiring Notion URL:

```ts
export async function uploadImageToSupabase(
  imageUrl: string,
  blockId: string,
  bucket: Bucket
): Promise<string> {
  const storagePath = `images/${blockId}`
  const publicUrl = getPublicUrl(bucket, storagePath)

  try {
    const check = await fetch(publicUrl, { method: 'HEAD' })
    if (check.ok) return publicUrl  // already synced

    const res = await fetch(imageUrl)
    const contentType = res.headers.get('content-type') ?? 'image/jpeg'
    const buffer = Buffer.from(await res.arrayBuffer())

    await supabase.storage.from(bucket).upload(storagePath, buffer, {
      contentType: contentType.split(';')[0].trim(),
      upsert: true,
    })
  } catch (error) {
    console.error(`Error syncing image ${blockId}:`, error)
    // Return Supabase URL regardless — a missing image recovers on next ISR;
    // a baked-in expired Notion URL does not.
  }

  return publicUrl
}
```

---

## Step 5 — Sync scripts

### `scripts/sync-notion-covers.ts`

Script that queries all databases and uploads cover images to Supabase. Run manually or as part of `npm run build`.

Add to `package.json`:

```json
"scripts": {
  "build": "tsx scripts/sync-notion-covers.ts && next build",
  "sync:covers": "tsx scripts/sync-notion-covers.ts",
  "sync:images": "tsx scripts/sync-notion-images.ts"
}
```

---

## Step 6 — API routes

### Cron endpoint: `app/api/cron/sync-covers/route.ts`

```ts
import { NextRequest, NextResponse } from 'next/server'
import { revalidatePath } from 'next/cache'
import { syncAllCovers } from '@/lib/sync-notion-covers'

export const maxDuration = 300

export async function GET(request: NextRequest) {
  if (request.headers.get('authorization') !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }
  const result = await syncAllCovers()
  revalidatePath('/', 'layout')
  return NextResponse.json({ success: true, ...result })
}
```

### Manual trigger: `app/api/sync/covers/route.ts`

```ts
import { NextRequest, NextResponse } from 'next/server'
import { revalidatePath } from 'next/cache'
import { syncAllCovers } from '@/lib/sync-notion-covers'

export async function POST(request: NextRequest) {
  const { searchParams } = new URL(request.url)
  if (searchParams.get('secret') !== process.env.REVALIDATE_SECRET) {
    return NextResponse.json({ error: 'Invalid secret' }, { status: 401 })
  }
  const result = await syncAllCovers()
  revalidatePath('/', 'layout')
  return NextResponse.json({ success: true, ...result })
}
```

---

## Step 7 — Vercel cron configuration

Add to `vercel.json`:

```json
{
  "crons": [
    {
      "path": "/api/cron/sync-covers",
      "schedule": "0 2 * * *"
    }
  ]
}
```

> Vercel Hobby plan supports maximum 2 cron jobs with daily minimum interval. For hourly, upgrade to Pro or use an external cron service (e.g. cron-job.org) to POST to `/api/sync/covers?secret=<REVALIDATE_SECRET>`.

---

## Usage after setup

**After publishing a new post in Notion**, trigger a sync manually:

```bash
curl -X POST "https://your-domain.com/api/sync/covers?secret=<REVALIDATE_SECRET>"
```

The cron job handles nightly cleanup automatically.

---

## Known limitations

### Nested block images are NOT synced

`getPageBlocks` only fetches top-level blocks. Images inside these Notion block types still use expiring Notion URLs:

- **Column** (`column_list → column → image`)
- **Toggle**
- **Callout**
- **Quote**

**Fix when needed:** Recursively fetch children for `column_list`, `toggle`, `callout`, `quote` in `getPageBlocks`, and update `syncBlockImages` to recurse into nested blocks.

### New post covers require a sync run

Cover images are not synced on-demand at render time. A new post will show a broken cover until the sync runs. The cron handles this automatically overnight; use the manual trigger for immediate results.
