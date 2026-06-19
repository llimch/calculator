# Notepad Calculator — Single File Version

**Single self-contained `index.html` + Supabase for real user accounts and data.**

## Quick Start

1. Open [SINGLE_HTML_SUPABASE_PLAN.md](SINGLE_HTML_SUPABASE_PLAN.md) and follow the **Supabase setup** section (5 minutes).
2. Edit `index.html` at the top of the `<script>` and replace:
   - `SUPABASE_URL`
   - `SUPABASE_ANON_KEY`
3. Double-click `index.html` or serve it:
   ```bash
   npx serve .
   # or
   python -m http.server 8080
   ```
4. Create an account or log in. All notes are private and synced via Supabase.

## Features
- Full original calculator (variables, ans, %, line continuation, power, comments, currency formatting)
- Multiple notes with drag-to-reorder (persisted)
- Live calculation results
- Dark/light + color + result position settings
- Real email + password accounts (Supabase Auth)
- Data stored safely in your Supabase Postgres project

## Files
- `index.html` — the complete application
- `SINGLE_HTML_SUPABASE_PLAN.md` — full plan, schema, hosting guide, and rationale

## Cleanup
Old Next.js files have been moved to the `bak/` folder.

## Hosting (Everything Frontend on Vercel)

**Yes — the entire frontend (`index.html`) can be hosted on free Vercel.**

Your GitHub repo: https://github.com/llimch/calculator.git

### Recommended Deployment (Vercel + your GitHub)
1. Push the latest `index.html` + `vercel.json` (with your Supabase keys) to https://github.com/llimch/calculator.git
2. Go to [vercel.com](https://vercel.com), sign in with GitHub.
3. Import the `llimch/calculator` repository.
4. Deploy — done. Live at `https://your-project.vercel.app`.

A `vercel.json` is included for clean fallback and security headers. Vercel is free for this use case.

**Note on "everything"**:
- The UI, calculator, and interface → hosted on Vercel.
- User login + saved notes → powered by Supabase (separate free service that the HTML talks to).

## Quick Local Test
```bash
npx serve .
# or open index.html directly
```

See the plan document for complete details and GitHub Action to prevent free-tier pausing.

## Version Control
All edits to `index.html` now follow the strict Version Control Procedure documented in `SINGLE_HTML_SUPABASE_PLAN.md`.

Version history snapshots are kept in `VersionControl/`.

**Policy introduced:** 2026-06-19

Current live file: `index.html` (in root)
History-only directory: `VersionControl/` (do not run files from here)

**Developed following the original SteveRidout concept + local version simplification.**
