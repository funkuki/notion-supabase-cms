# notion-supabase-cms
Notion's file images — covers, inline blocks — use signed URLs that expire in roughly one hour. If you're building a Next.js site with Notion as your CMS, those images will silently break on any cached or statically rendered page.

This toolkit solves that problem by treating Supabase Storage as the stable serving layer. When you publish a post, a sync script downloads the cover from Notion and uploads it to your Supabase bucket, generating a permanent public URL. Inline content images are handled lazily at render time: each image block is checked against Supabase first, and only downloaded from Notion if it hasn't been synced yet. Pages always receive a Supabase URL — never an expiring Notion signed URL.

What's included:

lib/notion.ts — typed query helpers for Notion databases, always reads image URLs from Supabase via getPublicUrl()
lib/notion-image-sync.ts — per-block image upload with HEAD-check deduplication
lib/sync-notion-covers.ts — batch cover sync script across multiple databases
app/api/cron/sync-covers — Vercel Cron endpoint, secured with CRON_SECRET
app/api/sync/covers — manual trigger endpoint for immediate sync after publishing
ISR revalidation wired into both sync routes
Stack: Next.js App Router · Notion API · Supabase Storage · Vercel

Setup takes about 20 minutes. Configure your Notion database schema, create Supabase buckets, add environment variables, and drop the library files into your project. A Claude Code skill (/notion-supabase-cms) is included to scaffold the full setup automatically.
