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
  - **Frontend (index.html + all UI/calculator logic)**: Hosted on **Vercel** (free tier).
  - **Backend (users, login, saved notes)**: Must use an external service. This part **cannot** be hosted purely inside Vercel if you want to keep the project as a single static HTML file.

**Important Answer to your question:**

**Can you avoid Supabase and use only Vercel?**

**No, not easily** — if you want real user login + private stored data.

Vercel is excellent for **hosting** the HTML, but Vercel itself does **not** provide:
- User authentication (login, signup, password, sessions)
- A secure per-user database

Vercel has storage products (Postgres, KV), but using them securely for user accounts from a pure client-side HTML file requires adding serverless functions. This would break the "single file" simplicity we have now.

### Realistic Options

| Approach                  | Real User Login? | Private Data per User? | Stays Single HTML? | Recommendation |
|---------------------------|------------------|------------------------|--------------------|----------------|
| Supabase (current)        | Yes             | Yes                    | Yes                | Easiest       |
| Clerk                     | Yes             | Yes                    | Yes                | Best alternative |
| Firebase                  | Yes             | Yes                    | Yes                | Good          |
| Only localStorage         | Fake only       | No (per browser)       | Yes                | Simplest, but limited |
| Vercel KV + Functions     | Possible        | Possible               | No                 | More complex  |

**My advice right now:**
- If you want to drop Supabase, the cleanest replacement while staying single-file is **Clerk**.
- If you are okay with "no real login across devices", we can switch to pure localStorage (much simpler).

Do you want me to:
A) Switch the project to **Clerk** (still single HTML + Vercel)?
B) Remove external service and use **localStorage only**?
C) Keep Supabase but make the setup instructions clearer?
- **Recommended Deployment**
  - Use your GitHub repo (https://github.com/llimch/calculator.git) as the source.
  - Connect Vercel to that repo — Vercel will build/deploy automatically on every push.
- **Local dev:** Just open index.html in browser (or simple http server for CORS if needed during dev).
- **Cleanup:** All Next.js / old source moved to `bak/`.

## 3. Supabase Project Setup – Step by Step (Recommended for your project)

### Step 1: Create Account & Project
1. Go to https://supabase.com
2. Click **Sign up** (you can use GitHub, Google, or email – no credit card needed)
3. After logging in, click **New project**
4. Fill in:
   - **Name**: `notepad-calculator` (or anything you like)
   - **Database Password**: Create a strong password and save it somewhere safe
   - **Region**: Choose the closest to you (Singapore if available)
5. Click **Create new project** and wait ~1-2 minutes

### Step 2: Get Your Supabase Keys (Important)
1. Once the project is ready, go to the left sidebar → **Settings** → **API**
2. Copy these two values (you will need them):
   - **Project URL** → looks like `https://abc123.supabase.co`
   - **anon public** key → starts with `eyJhbGciOi...`

### Step 3: Enable Email Login
1. In the left sidebar, click **Authentication**
2. Go to the **Providers** tab
3. Make sure **Email** is enabled
4. (Recommended for testing) Turn **OFF** "Confirm email" so you can sign up and log in immediately without email verification

### Step 4: Create the Database Tables + Security
1. In the left sidebar, click **SQL Editor**
2. Click **New query**
3. Copy and paste the entire code below, then click **Run**

```sql
-- 1. Notes table
create table if not exists public.notes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade not null,
  title text not null default 'New Note',
  content text not null default '',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- 2. Note ordering table (for drag & drop order)
create table if not exists public.note_order (
  user_id uuid primary key references auth.users(id) on delete cascade,
  note_ids uuid[] not null default '{}'
);

-- 3. Enable Row Level Security (RLS)
alter table public.notes enable row level security;
alter table public.note_order enable row level security;

-- 4. Security policies – users can only access their own data
create policy "Users can view own notes"
  on public.notes for select using (auth.uid() = user_id);

create policy "Users can insert own notes"
  on public.notes for insert with check (auth.uid() = user_id);

create policy "Users can update own notes"
  on public.notes for update using (auth.uid() = user_id);

create policy "Users can delete own notes"
  on public.notes for delete using (auth.uid() = user_id);

create policy "Users can view own note order"
  on public.note_order for select using (auth.uid() = user_id);

create policy "Users can manage own note order"
  on public.note_order for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

-- 5. Auto-update timestamp trigger
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

### Step 5: Put the Keys into Your Project
1. Open your local `index.html`
2. Near the top of the file, replace the placeholder lines:
   ```js
   const SUPABASE_URL = 'https://YOUR-PROJECT-ID.supabase.co';
   const SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...YOUR-ANON-KEY-HERE...';
   ```
3. Paste your real **Project URL** and **anon key** from Step 2

### Step 6: Test & Deploy
1. Open `index.html` directly in your browser and test signup/login + creating notes
2. When it works locally:
   ```powershell
   git add index.html
   git commit -m "Add Supabase keys"
   git push
   ```
3. Vercel will automatically redeploy (check your Vercel dashboard)
4. Visit your live Vercel URL and test again

**Tip**: On the free plan, Supabase may pause your project after ~1 week of inactivity. You can prevent this with a simple GitHub Action (I can give you the code if needed).

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

## Latest Adjustments (from latest user feedback)

- Removed left sidebar note list (A) — redundant now that pulldown select (D) exists for note selection.
- Removed duplicate title display (B) in the main editor area.
- Moved the "Last: ..." date/time info to the **bottom of the footer**, left-aligned (C).
- Pulldown box (D) made narrower (200px width). The title text displayed in the pulldown is now **red**.
- Added a small pencil (✎) icon next to the pulldown for editing the current note's title.
- The <select> pulldown now serves as the primary note chooser and title display.

These changes keep the UI clean, especially for mobile views, while preserving core functionality (Supabase login, calculations, note switching via pulldown, drag-reorder was in removed sidebar but list can be re-added if needed).

**Status:** All changes applied to index.html. Test locally or push to GitHub for Vercel.

---

## v10 Changes (2026-06-19) — iPhone Numeric Keyboard + Operator Toolbar

**Version snapshot:** `VersionControl/index.html.v10.20260619.2308`

### Changes Made
- Removed the duplicate yellow "+" button from the top header (it duplicated the "+ New Note" option already present at the bottom of the note `<select>` pulldown).
- Added `inputmode="decimal"` to the main editor `<textarea id="note-content">`. iPhone now defaults to the numeric/decimal keyboard when tapping to edit calculations or notes.
- Title editing (`✎` pencil) was upgraded from `prompt()` to a proper centered modal. The title input also defaults to `inputmode="decimal"` for numeric preference on both sections.
- Added a custom **operator toolbar** that appears (fixed at bottom) whenever the editor is focused:
  - **ABC / 123** toggle button: switches `inputMode` between "decimal" (numeric) and "text" (standard alpha keyboard) and forces iOS to refresh the keyboard.
  - Operator buttons: `+`  `−`  `×`  `÷`  `=` (inserts correct ASCII characters so the built-in calculator continues to parse expressions).
  - Dismiss (⌨︎) button to hide toolbar and blur the field.
- Toolbar provides the missing `+` and `-` (and other operators) without forcing users back to the full alpha keyboard permanently.
- Follows user's explicit request: "defaulted to numeric as entry but allow toggle back to standard keyboard".
- Applied to the primary content editor. Title modal gets numeric default via its input (toolbar kept off the modal to avoid UI overlap).

All edits followed the Version Control Procedure (pre-edit snapshot captured as v10).

## v11 (2026-06-19) — Remove unwanted iOS autofill bar
- Added `autocomplete="off"`, `autocorrect="off"`, `autocapitalize="off"`, `spellcheck="false"` to both the main editor textarea and the title edit input (statically + reinforced in JS on every focus and keyboard toggle).
- This eliminates the iOS system autofill suggestion bar that was showing password (key), credit card, and location icons above the keyboard.
- Result: Clean numeric keyboard by default (or standard full keyboard via the ABC/123 toggle) with only our operator toolbar, no extraneous system icons.
- New snapshot: `VersionControl/index.html.v11.20260619.2317`

---

## Change Log – Implemented Features & UI Adjustments

All changes were made to the single `index.html` while following the Version Control Procedure (snapshots saved in `VersionControl/`).

### Core Architecture
- Converted from complex Next.js + localStorage to a **single self-contained static HTML file**.
- Integrated **Supabase** for real user authentication (email/password) and per-user private note storage with Row Level Security (RLS).
- Removed public sign-up. New users must be added manually by the administrator directly in Supabase.
- Added `vercel.json` for clean Vercel deployment.
- Enter key on login form now submits the "Log in" button.
- Full support for local testing (`python -m http.server` or `npx serve`).

### Hosting & Deployment
- Primary hosting: **Vercel** (free static hosting) connected to GitHub repo `https://github.com/llimch/calculator.git`.
- Local run supported without Vercel.

### UI & Layout Improvements
- **New Note button**: Replaced large yellow "+ New Note" button with a small yellow "+" icon placed beside the gear/settings button.
- **Sidebar**: Narrowed to `w-48`. Removed redundant "Notes" label. On mobile, notes list is collapsed by default with arrow toggle to expand/collapse.
- **Note selection**: Added a pulldown `<select>` in the title bar area. Lists all notes + "+ New Note" option. Selecting a note loads it into the editor. Current note title is displayed in the select.
- **Note title bar**:
  - Note title input styled in **light blue**.
  - Added **last updated** display in **light green**.
  - Format: `Last: 19/Jun/2026 (Fri) 10:23 pm` (UTC stored in DB, displayed in user's local timezone with date + time).
- **Results column**: Width now configurable in settings (down to 50px). Default ~180px. Supports numbers up to 8,888,888,888.00.
- **Font size**: +/- controls added in settings (10px–24px range) + Reset. Persists in browser.
- **Header on mobile**: "Notepad Calculator" title uses smaller font. User email moved below the Logout button for better space usage.
- **Footer**: Updated to "Vibe Code Using SuperGrok by Lawrence Lim YYYY" (year is dynamic via JavaScript and updates automatically). Same footer added to login screen.
- **Status bar**: Removed the "Connected to Supabase" bar to save vertical space.
- **Settings panel (gear)**: Added Font Size and Results Width controls.
- **Login screen**: Public sign-up completely removed. Only login form + message that new users are added manually by administrator. Footer updated.

### Version Control
- Strict procedure followed: Before major edits, current `index.html` is copied to `VersionControl/index.html.vNN.YYYYMMDD.HHMM`.
- Multiple snapshots created during development (v01 through v11).
- v10 (20260619.2308): iPhone numeric keyboard default + custom operator toolbar with toggle.
- v11 (20260619.2317): Suppressed iOS password/credit-card/location autofill suggestions for clean keyboard.

### Other
- Mobile-first responsive adjustments for iPhone (narrow results, collapsible notes, pulldown selector, stacked header elements).
- Supabase keys remain in `index.html` (standard for client-side static apps; security relies on RLS policies).

**Current recommended files to push to GitHub:**
- `index.html`
- `vercel.json`
- `README.md`
- `SINGLE_HTML_SUPABASE_PLAN.md`
- `.gitignore`
- `VersionControl/` (history)

---

**Status:** All requested changes implemented and documented. Ready for local testing and Vercel deployment.

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

---

## Next Steps After Vercel Deployment (Current Situation)

The deployment succeeded, but you see **"Supabase not configured"**.

This is normal — the keys in `index.html` are still the placeholder values.

### Do this now (follow the version control rule):

1. **Create snapshot first** (mandatory):
   ```powershell
   # Get current time in HHMM format
   $time = Get-Date -Format "HHmm"
   Copy-Item index.html "VersionControl/index.html.v06.20260619.$time"
   ```

2. **Edit your local `index.html`**:
   - Open the file
   - Replace these two lines near the top with your real Supabase values:
     ```js
     const SUPABASE_URL = 'https://YOUR-PROJECT-ID.supabase.co';
     const SUPABASE_ANON_KEY = 'eyJhbGciOi...';
     ```

3. **Push the update**:
   ```powershell
   git add index.html
   git commit -m "Configure real Supabase keys"
   git push
   ```

4. Wait ~30-60 seconds for Vercel to redeploy.

5. Refresh your live Vercel URL.

6. On the Vercel congratulations page, you can click the yellow **"Reload after editing"** button.

Once the keys are correct, the login screen will appear and you can create accounts / save notes.
- `node_modules/`
- Any old Next.js files

Only the files needed for Vercel + the VersionControl history.
