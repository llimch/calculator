# Notepad Calculator — Single File Version

**Single self-contained `index.html` + Supabase for real user accounts and data.**

**Current version: v12 (2026-06-19)** — see Changelog below.

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
- `index.html` — the complete application (current **v12**)
- `VersionControl/index.html.vNN.YYYYMMDD.HHMM` — historical snapshots (do not run)
- `SINGLE_HTML_SUPABASE_PLAN.md` — full plan, schema, hosting guide, change log, and versioning procedure

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

## Current Version

**v12** (2026-06-19)

The app version is shown in:
- Main footer (e.g. `2026 • v12`)
- Login screen footer
- Settings dropdown (bottom)

## Versioning Policy

All changes to the live application follow the project's strict version control procedure (see `SINGLE_HTML_SUPABASE_PLAN.md` → "Version Control Procedure"):

1. Before significant edits, the current `index.html` is snapshotted to `VersionControl/index.html.vNN.YYYYMMDD.HHMM`.
2. Live file always remains `index.html` in the project root.
3. `VersionControl/` is **history only** (do not open/run files from it).
4. A new `APP_VERSION` constant + UI labels are added/updated for the release.

**Latest snapshot:** `VersionControl/index.html.v12.20260619.2337`

**Policy introduced:** 2026-06-19

## Changelog

### v12 (2026-06-19) — Header compaction + keyboard resilience + results cleanup
- Removed the logo (NC) and "Notepad Calculator" main heading text to save vertical space.
- Moved the note dropdown (pull-down menu) into the primary header (replacing the removed title location).
- **Special keys / operator toolbar** (ABC, + − × ÷ = ⌨︎) moved from the bottom of the screen to a compact row in the **top-right of the header**. This prevents the toolbar from being blanked out/covered when the iPhone virtual keyboard appears.
- Removed the separate "note title bar" (the extra dark bar containing the select + Delete). All controls integrated into a single slim header for maximum editor space.
- **"+ New Note"** removed from the dropdown selector (hard to reach when the list of notes is long).
- Added a prominent **green "+" button** directly next to the pencil (✎) edit icon in the header for instant new note creation.
- Removed all leading `=` characters from the **results panel** (left column). Values now display cleanly (e.g. `63295.15` instead of `= 63295.15`).
- Added first-class **versioning**:
  - `const APP_VERSION = 'v12';` defined in the source.
  - Version rendered in footers and Settings panel.
- Minor: tightened header padding and button sizes for the compact layout; snapshot procedure followed.

### v11 (2026-06-19)
- iOS autofill suppression, numeric keyboard defaults, operator toolbar (previous bottom version), title modal, font/results width settings, login-only flow, dynamic footer, etc. (see previous entries and `SINGLE_HTML_SUPABASE_PLAN.md` for details).

### v10 and earlier
See `SINGLE_HTML_SUPABASE_PLAN.md` (Change Log + Version Control sections) for the complete prior history and all snapshots.

**Developed following the original SteveRidout notepad-calculator concept.**

## Version Control (summary)
All edits follow the procedure in `SINGLE_HTML_SUPABASE_PLAN.md`.

- Current live file: `index.html`
- History: `VersionControl/` (v06 … v12)
- Git commits should include the new snapshot when bumping versions.
