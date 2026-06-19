# Single HTML + Supabase Migration Plan
**Project:** Notepad Calculator  
**Date:** 2026-06-19 (updated for hosting)  
**Goal:** Simplify from complex Next.js (localStorage only) to a **single self-contained index.html** while adding **real user-based login and persistent stored data** using Supabase (free tier viable for solo use).

**Hosting:** Free Vercel (preferred) or GitHub Pages for the static HTML file.

**Developed by:** Based on original https://github.com/SteveRidout/notepad-calculator + current local Next.js clone. Plan and implementation by Grok.

## 1. Decision & Rationale
- **Pure single HTML** for the app logic/UI/calculator (maximum simplicity, double-click or static-host friendly, no Node runtime needed for the app itself).
- **Supabase** (managed Postgres + Auth) for:
  - Real user accounts (email + password)
  - Per-user private data (notes)
  - Cross-device access
- Why Supabase over full custom Node (original style):
  - Zero server code to maintain.
  - Free tier is extremely generous for 1 user.
  - Direct browser client integration fits single-file model perfectly.
  - Row Level Security (RLS) gives strong per-user isolation without custom backend.
- Alternative considered: PocketBase (self-hosted single binary). We start with Supabase for easiest free start; migration path noted if needed.
- Current local version complexity (Next.js, .bat launcher, heavy node_modules) is removed.

**Volume note:** Solo/personal use → Free tier sufficient (500MB DB, 50k MAUs, unlimited API). Main Free caveat = projects pause after 1 week inactivity (easy workaround below).

## 2. Final Architecture
- **Frontend:** One file `index.html`
  - Self-contained UI (replicates current: header, sidebar with draggable notes, title+editor+live results pane, settings dropdown, delete modal).
  - Tailwind via CDN for rapid modern styling (matches current dark/gold theme).
  - All JS in one `<script>` block (no build step).
  - Supabase browser client loaded via CDN.
  - Custom favicon (embedded): Dark background with notepad + calculator icon.
  - Header: Notepad Calculator
  - Footer: Vibe code by Grok by Lawrence Lim
- **Backend/Data:** Supabase (required for user accounts + stored data)
  - Auth (email/password signup/login/logout, sessions persisted in browser).
  - Database tables with RLS.
  - Direct calls from the browser (no server/proxy code).
- **Hosting Split (Free)**
  - **Frontend (index.html + all UI/calculator logic)**: Hosted on **Vercel** (free tier). This is the part you asked about.
  - **Backend (users, login, saved notes)**: Hosted on **Supabase** (free tier). This cannot be moved to Vercel.
  - Vercel serves the static HTML. The HTML then securely talks to your Supabase project from the user's browser.
- **Recommended Deployment**
  - Use your GitHub repo (https://github.com/llimch/calculator.git) as the source.
  - Connect Vercel to that repo — Vercel will build/deploy automatically on every push.
- **Local dev:** Just open index.html in browser (or simple http server for CORS if needed during dev).
- **Cleanup:** All Next.js / old source moved to `bak/`.

## 3. Supabase Project Setup (Do this first)
1. Go to https://supabase.com → Sign up (GitHub/Google/email, free, no card).
2. Create new project (name e.g. "notepad-calculator-personal", password for DB).
3. In project Dashboard → **Authentication** → Providers → Enable **Email** (password sign-in). Disable "Confirm email" for simplicity (or keep for production).
4. Go to **SQL Editor** and run the following to create tables + RLS (copy-paste exactly):

```sql
-- Enable extensions if needed (usually already on)
-- 1. Notes table
create table if not exists public.notes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade not null,
  title text not null default 'New Note',
  content text not null default '',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- 2. Note ordering (for drag-reorder persistence)
create table if not exists public.note_order (
  user_id uuid primary key references auth.users(id) on delete cascade,
  note_ids uuid[] not null default '{}'
);

-- 3. Enable RLS
alter table public.notes enable row level security;
alter table public.note_order enable row level security;

-- 4. RLS Policies (only owner can access)
create policy "Users can view own notes"
  on public.notes for select
  using (auth.uid() = user_id);

create policy "Users can insert own notes"
  on public.notes for insert
  with check (auth.uid() = user_id);

create policy "Users can update own notes"
  on public.notes for update
  using (auth.uid() = user_id);

create policy "Users can delete own notes"
  on public.notes for delete
  using (auth.uid() = user_id);

-- Order table policies
create policy "Users can view own note order"
  on public.note_order for select
  using (auth.uid() = user_id);

create policy "Users can manage own note order"
  on public.note_order for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);

-- 5. Optional: Updated_at trigger
create or replace function public.handle_updated_at()
returns trigger language plpgsql as $$
begin
  new.updated_at = now();
  return new;
end;
$$;

create trigger notes_updated_at
  before update on public.notes
  for each row execute function public.handle_updated_at();
```

5. Go to **Project Settings → API**:
   - Copy `Project URL` and `anon public` key.
   - In the final `index.html`, replace the placeholders:
     ```js
     const SUPABASE_URL = 'https://YOUR-PROJECT.supabase.co';
     const SUPABASE_ANON_KEY = 'eyJhbGc...';
     ```

6. (Optional) Create a test user via Dashboard → Authentication → Users.

**Free tier maintenance:** To prevent auto-pause, add a free GitHub Action that pings a table once/week (documented in hosting section below).

## 4. Data Model (in Supabase)
- `auth.users` (managed by Supabase)
- `notes` (as above)
- `note_order` (user_id + note_ids array) — replicates current drag-reorder exactly.
- No localStorage primary storage. Optional local cache for offline feel later.

Notes `content` stores the raw multi-line text exactly as in current editor.

## 5. Single HTML Implementation Strategy
- **Styling:** `<script src="https://cdn.tailwindcss.com"></script>` + runtime `tailwind.config` to match current theme (#121212 bg, #FFD700 accents, monospace editor, etc.).
- **Layout:** Flex header + sidebar (w-56) + main editor.
- **Editor:** Two synced panes (textarea + results div). Live `input` + `scroll` sync. Results computed on every change.
- **Calculator:** Direct port of current `src/utils/calculator.ts` (evaluateLines, preprocessLine, Parser, formatNumber, smart %, variables, ans, line continuation, $ commas, comments via : or ').
  - Pure JS functions, no dependencies.
- **State:** In-memory + Supabase sync.
- **Features to replicate exactly:**
  - New Note button
  - Draggable sidebar list (HTML5 drag/drop + persist order)
  - Title editable
  - Delete with confirm modal
  - Settings: Dark/Light, text color swatches (8 colors), result position (left/right)
  - Welcome note on first use per user
  - Proper previous answer & variable scoping
- **Auth UI:** Centered login/signup card when not authenticated (email + password). Switch tabs. After login load user's notes.
- **Header:** Logo + user email + Settings gear + Logout.
- **Persistence:** On login load notes + order from Supabase. Autosave on changes (debounced or on blur/key).
- **Extras:** Export all as zip (optional v1 or later), simple loading states.

**No external files** except two CDNs (Tailwind + Supabase JS).

## 6. Porting Notes from Current Code
- `src/utils/calculator.ts` → inline JS object/functions.
- `src/utils/storage.ts` → replaced by Supabase client calls + helper functions (`loadNotes`, `saveNote`, `deleteNote`, `loadOrder`, `saveOrder`).
- `src/types/index.ts` → simple JS objects.
- Components:
  - `Sidebar.tsx` → divs + event listeners for drag.
  - `NoteEditor.tsx` → split-pane + sync logic.
  - `SettingsPanel.tsx` + colors → dropdown + buttons.
  - `DeleteConfirmModal.tsx` → simple overlay div.
- Main `page.tsx` logic → top-level state + useEffect equivalents (event listeners + async awaits).
- Theme handling, effective colors (white→dark-blue in light).

Keep behavior 100% identical for calculations and UX where possible.

## 7. File Structure After Changes
```
notepadcalculator/
├── index.html                 ← THE APP (single file)
├── SINGLE_HTML_SUPABASE_PLAN.md
├── bak/                       ← everything old moved here
│   ├── src/
│   ├── next.config.mjs
│   ├── ... (all other old files)
│   └── node_modules/ (optional, large)
└── (maybe a minimal README.md note)
```

## 8. Cleanup Steps (Executed Automatically)
- Create `bak/` directory.
- Move:
  - `src/`
  - `terminals/`
  - `next.config.mjs`, `postcss.config.mjs`, `tailwind.config.ts`
  - `tsconfig.json`, `next-env.d.ts`
  - `package.json`, `package-lock.json`
  - `settings.json`
  - Any other build-related (keep plan doc and new html)
- node_modules may be left or moved last (very large).
- After build, user can safely `rm -rf bak` or delete node_modules manually.

## 9. Hosting & Running (Vercel Recommended)
**Important clarification**: 
- The single `index.html` (UI + calculator + everything visual) can be **fully hosted on Vercel** for free.
- User accounts and saved notes **must stay on Supabase**. This part cannot move to Vercel.

| Part                    | Hosted On     | Why |
|-------------------------|---------------|-----|
| index.html + UI + Calculator | **Vercel**   | Static frontend |
| Login / User accounts   | **Supabase** | Requires real auth + database |
| Saved notes (per user)  | **Supabase** | Requires secure per-user storage + RLS |

### Recommended way (using your GitHub repo)
Your repo: https://github.com/llimch/calculator.git

1. Make sure your local `index.html` (with correct Supabase URL + anon key) + `vercel.json` is committed and pushed to the repo.
2. Go to https://vercel.com → Sign in with GitHub.
3. Click **"Add New Project"** → Import the `llimch/calculator` repository.
4. Vercel will auto-detect the static HTML. Click Deploy.
5. Your app will be live at `https://your-project.vercel.app` (you can add a custom domain later for free).

A `vercel.json` is included in the project for clean routing and basic security headers.

### Alternative quick deploy (no Git)
- On Vercel dashboard, choose "Upload" and drag the single `index.html` file.

### Local testing
```bash
# Any of these work
npx serve .
# or
python -m http.server 8080
```

4. (Optional) Keep Supabase project alive with this GitHub Action (add to your repo):
   ```yaml
   # .github/workflows/keep-supabase-alive.yml
   name: Keep Supabase alive
   on:
     schedule:
       - cron: '0 0 * * 1'   # Monday
   jobs:
     ping:
       runs-on: ubuntu-latest
       steps:
         - run: |
             curl -X GET "https://YOUR.supabase.co/rest/v1/notes?select=id&limit=1" \
               -H "apikey: YOUR_ANON_KEY" \
               -H "Authorization: Bearer YOUR_ANON_KEY"
   ```

## 10. Risks / Limitations & Mitigations
- Free Supabase pause: Use the cron above (or log in weekly).
- No offline: Notes require internet after login (can add local fallback in future).
- CDN dependency (Tailwind/Supabase): Reliable CDNs; for ultra-offline could inline but increases file size.
- Email auth only (for now). Username support can be added via user metadata.
- Single file size will be ~50-80KB (acceptable).

## 11. Implementation Order (This Build)
1. Documentation (this file) ✅
2. Project cleanup (move to bak/)
3. Create index.html skeleton + Tailwind
4. Port calculator engine
5. Build auth UI + Supabase init + login/signup flows
6. Build notes loading/saving + editor + live results
7. Sidebar + drag reorder + note order sync
8. Settings panel + theme
9. Delete modal + other polish (welcome note, loading states)
10. Final test + minimal docs

## 12. Post-Build
- Test with real Supabase project.
- Add a small `README.md` or update this plan with "done" markers.
- Future enhancements (if wanted): offline cache, zip export, dark/light per user settings stored in Supabase.

**This document preserves all memory.** Do not delete without copying key sections.

---

**Status:** Ready to execute. All following actions (cleanup + build) will follow this plan exactly.

## Version Control Procedure (index.html only)
Primary source file: `index.html`

**Mandatory steps BEFORE every rework or edit to index.html**:
1. Identify the next version number (v01, v02, ... — always increment by 1 from the highest existing in VersionControl/).
2. Copy the **current (pre-rework) version** of the file using this exact name format:
   `index.html.vNN.YYYYMMDD.HHMM`
   - NN = zero-padded increment (v01, v02, v10, ...)
   - YYYYMMDD.HHMM = local date+time in 24-hour format (e.g. 20260619.1245)
   - Full example: `index.html.v02.20260620.0930`
3. Target directory: `VersionControl/`
4. After the copy is done, perform the edit/rework on the working copy of `index.html` in the project root.
5. Document the change in README.md (under a new "Changes in vXX" note or in Key Fixes).
6. Test by opening `index.html` directly in a browser.

VersionControl/ is **history only** — do not execute files from it or modify them. The live file is always the root `index.html`.

This policy was introduced 2026-06-19 per requirements. This is a single-file HTML project only. All future edits to `index.html` must follow it exactly.

## Git Commands for GitHub (llimch/calculator.git)

Run these commands **in PowerShell** from inside the project folder:

```powershell
# 1. Initialize git (skip if you already have a .git folder)
git init

# 2. Add the correct files (include VersionControl history)
git add index.html vercel.json README.md SINGLE_HTML_SUPABASE_PLAN.md .gitignore
git add VersionControl/

# 3. Commit
git commit -m "Single HTML + Supabase version with VersionControl history"

# 4. Connect to your GitHub repo
git remote remove origin 2>$null
git remote add origin https://github.com/llimch/calculator.git

# 5. Push
git branch -M main
git push -u origin main
```

**After pushing:**
- Go to https://vercel.com
- Import the repository `llimch/calculator`
- Deploy (it will be free)

**Do NOT push these:**
- `bak/`
- `node_modules/`
- Any old Next.js files

Only the files needed for Vercel + the VersionControl history.
