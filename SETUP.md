# Maintenance Expenses Tracker — Setup & Deployment

A single-file HTML expense tracker with Supabase (PostgreSQL) backend and free Vercel hosting.

## Architecture
- **Frontend**: Single `index.html` (vanilla JS, no build step)
- **Database**: Supabase (PostgreSQL free tier)
- **Hosting**: Vercel (free tier)

---

## Step 1 — Create the Supabase project

1. Go to https://supabase.com → sign up → **New project**.
2. Region: choose one close to you (e.g. `Singapore`).
3. Generate a strong database password — save it somewhere safe.
4. Wait ~1 minute for provisioning.

### Step 1a — Create tables

Go to **SQL Editor** in the Supabase dashboard, paste this, and click **Run**:

```sql
-- CAPEX items (assets)
CREATE TABLE capex_items (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  code TEXT DEFAULT '',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Expenses
CREATE TABLE expenses (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  date DATE NOT NULL,
  category TEXT NOT NULL,
  amount NUMERIC(12,2) NOT NULL DEFAULT 0,
  remarks TEXT DEFAULT '',
  capex_item_id UUID REFERENCES capex_items(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Monthly targets (one row per month, keyed by YYYY-MM)
CREATE TABLE targets (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  month_key TEXT NOT NULL UNIQUE,
  total NUMERIC(12,2) DEFAULT 0,
  per_category JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_expenses_date ON expenses(date);
CREATE INDEX idx_expenses_category ON expenses(category);
CREATE INDEX idx_targets_month_key ON targets(month_key);
```

### Step 1b — Disable Row Level Security (for personal/internal use)

This app has no user log-in, so RLS is off. Run in SQL Editor:

```sql
ALTER TABLE capex_items DISABLE ROW LEVEL SECURITY;
ALTER TABLE expenses DISABLE ROW LEVEL SECURITY;
ALTER TABLE targets DISABLE ROW LEVEL SECURITY;
```

### Step 1c — Get your API keys

Go to **Settings → API**:
- Copy **Project URL** (looks like `https://abcdefg.supabase.co`)
- Copy **anon public** key (long JWT string starting with `eyJ...`)

---

## Step 2 — Configure `index.html`

Open `index.html` and find this section near the top of the `<script>` block:

```js
const SUPABASE_URL = 'https://YOUR-PROJECT.supabase.co';
const SUPABASE_ANON_KEY = 'YOUR-ANON-PUBLIC-KEY';
```

Replace both placeholders with the values from Step 1c. The badge in the header changes to "OFFLINE" red if these aren't set or are wrong.

---

## Step 3 — Push to GitHub

1. Create a new (empty) repo on GitHub, e.g. `maintenance-expenses`.
2. From your project folder:
```bash
git init
git add index.html vercel.json .gitignore
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/<you>/maintenance-expenses.git
git push -u origin main
```
(Don't commit local `.xlsx` exports — `.gitignore` excludes them.)

---

## Step 4 — Deploy to Vercel

1. Go to https://vercel.com → sign in with GitHub.
2. **Add New Project → Import** your `maintenance-expenses` repo.
3. Framework preset: leave as **Other** (no build).
4. **Deploy** — Vercel will publish to `https://maintenance-expenses-<hash>.vercel.app`.

Optional: **Settings → Domains** to add a custom domain.

---

## Step 5 — Verify

Open your live URL and:
1. Add an expense → it should persist after refresh.
2. Add CAPEX items → they should appear across browsers.
3. Set monthly targets → they should persist.
4. Run the Excel export → the `.xlsx` should download with the correct format.

---

## Migrating existing localStorage data

If you previously used the old localStorage-only version, the app will **automatically detect** old data on first load and migrate it to Supabase (only if the relevant table is empty). The localStorage entry is cleared after a successful migration.

To re-trigger migration manually, open the browser console:
```js
localStorage.setItem('maint_expenses', JSON.stringify({...your data...}));
location.reload();
```

---

## Local development

Open `index.html` in any browser — no server required for basic testing. As long as the Supabase URL/key are filled in, all features work.

Note: For testing without a network, the app shows an "OFFLINE" badge and silently disables DB calls.

---

## File layout

```
.
├── index.html       # The entire app (~1900 lines)
├── vercel.json      # Routing + security headers config
├── .gitignore       # Excludes .env, generated xlsx files
└── SETUP.md         # You're reading it
```

---

## Security notes

- The Supabase **anon key** is public — that's by design for client-side SDKs. Supabase isolates data via Row Level Security. Since this app disables RLS, anyone with the URL can read/write — so the URL is effectively a shared credential.
- For production-grade auth (multi-tenant, multi-user), enable Supabase Auth and add per-user RLS policies. The current setup is targeted at a single-tenant internal company tool.
- `vercel.json` adds a few security headers (X-Frame-Options DENY, etc.).

---

## Troubleshooting

- **Header badge says OFFLINE** → `SUPABASE_URL` or `SUPABASE_ANON_KEY` is wrong/missing. Check console for the error.
- **`permission denied for table expenses`** → RLS isn't disabled. Re-run the `ALTER TABLE ... DISABLE ROW LEVEL SECURITY` statements.
- **CAPEX dropdown empty even though you added items** → browser console likely shows a fetch error. Confirm network access and that the tables exist.
- **Migration didn't run** → open console: `supabase` should be defined and `localStorage.maint_expenses` should hold your old data.

---

## Free-tier limits (as of 2026)

- **Supabase free**: 500 MB database, 50k monthly active users (way more than needed). Project pauses after 7 days of inactivity — set up a free Supabase + GitHub workflow to ping it if you care, or upgrade later.
- **Vercel free**: 100 GB bandwidth/month, unlimited static sites.
- Plenty for many years of a single-tenant expense tracker.
