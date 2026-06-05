# How to Use

## Step 1 — Install the skill

Run this in your project root:

```bash
mkdir -p .claude/skills/notion-supabase-cms && curl -o .claude/skills/notion-supabase-cms/SKILL.md \
  https://raw.githubusercontent.com/funkuki/notion-supabase-cms/main/SKILL.md
```

This adds `SKILL.md` to `.claude/skills/notion-supabase-cms/` in your project.

## Step 2 — Run the skill in Claude Code

Open Claude Code in your project and type:

```
/notion-supabase-cms
```

Claude will read the skill and guide you through the full setup step by step.

## Step 3 — Follow the setup steps

Claude will walk you through:

1. **Notion database schema** — which properties to add (`Slug`, `Cover URL`, `Status`, etc.)
2. **Supabase Storage buckets** — which buckets to create and how to set them to public
3. **Environment variables** — all required keys for `.env.local` and Vercel
4. **Library files** — `lib/notion.ts`, `lib/supabase.ts`, `lib/notion-image-sync.ts`, `lib/sync-notion-covers.ts`
5. **API routes** — cron endpoint and manual trigger endpoint
6. **`vercel.json`** — cron job schedule configuration

## Step 4 — Sync covers after publishing

Every time you publish a new post in Notion, trigger a cover sync:

```bash
curl -X POST "https://your-domain.com/api/sync/covers?secret=<REVALIDATE_SECRET>"
```

The daily cron job handles this automatically overnight — use the manual trigger for immediate results.
