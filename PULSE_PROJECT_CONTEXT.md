# JAGGAER PULSE — Project Context & Handoff

## What Is This

PULSE (Performance, Unified Leadership & Success Engine) is a two-portal web application built for JAGGAER's Global Customer Care operations team. It is hosted on GitHub Pages with a Supabase backend. There is no build step — the entire app is two single-file HTML applications.

---

## Live URLs

| Environment | URL |
|---|---|
| Production | https://jaggaer-gcc.github.io/gcc-portfolio |
| Dev | https://jaggaer-gcc.github.io/gcc-portfolio-dev |

---

## Local Repositories

| Repo | Local Path | GitHub Org |
|---|---|---|
| Production | Documents > gcc-portfolio | jaggaer-gcc |
| Dev | Documents > gcc-portfolio-dev | jaggaer-gcc |

Deploy by dropping files into the folder and pushing via GitHub Desktop. No build step required.

---

## Supabase Projects

| Environment | URL | Project |
|---|---|---|
| Production | https://tuubqxnphoifqjbumgxa.supabase.co | gcc-portfolio |
| Dev | https://yhjsxfmyhzrrleqgaclz.supabase.co | gcc-portfolio-dev |

### Credentials

**Production**
- URL: `https://tuubqxnphoifqjbumgxa.supabase.co`
- Anon key: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InR1dWJxeG5waG9pZnFqYnVtZ3hhIiwicm9sZSI6ImFub24iLCJpYXQiOjE3Nzg1NDQ2MjAsImV4cCI6MjA5NDEyMDYyMH0.U0Tdp6GUBZMF16z00WvxmMTRKtWaJQEX954oy_w70Q0`

**Dev**
- URL: `https://yhjsxfmyhzrrleqgaclz.supabase.co`
- Anon key: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InloanN4Zm15aHpycmxlcWdhY2x6Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3Nzg2MTc3OTYsImV4cCI6MjA5NDE5Mzc5Nn0.ym8ns4Tk2fwRQAWUKpJdBoZr1_rfR6OpYOekKnw9V8w`

---

## File Structure

```
gcc-portfolio/
  index.html        — Portfolio Tool (single file, ~95KB)
  elt.html          — CS ELT Weekly Updates (single file, ~69KB)
  pulse-logo.png    — Transparent PNG logo (PULSE by JAGGAER)
```

The ELT source master uses `YOUR_SUPABASE_URL` and `YOUR_SUPABASE_ANON_KEY` as placeholders. Always run the credential swap before outputting dev or prod files.

---

## Database Schema

### profiles
```sql
id uuid, email text (unique), name text, display_name text,
role text, initials text, region text, department text,
can_edit bool, is_admin bool, portal_access text,
must_change_password bool
```

`portal_access` values: `portfolio` | `elt` | `both`

### projects
```sql
id uuid, name text, type text, status text, rag text,
owner text, region text, priority text, sponsor text,
start date, due date, progress int, description text,
kpis jsonb, impact text, milestones jsonb, tasks jsonb,
blockers jsonb, acceptance text, created_by uuid, created_at timestamptz
```

### kpi_values
```sql
id text, sla numeric, fcr numeric, csat numeric,
deflect numeric, ttr numeric, esc numeric
```

### elt_submissions
```sql
id uuid, submitter text, user_id uuid, user_email text,
user_role text, user_region text, user_department text,
user_display_name text, week_date text, submitted_by text,
answers jsonb, created_at timestamptz
```

### elt_files
```sql
id uuid, week_date text, filename text, storage_path text,
uploaded_by text, uploaded_at timestamptz, file_size int, description text
```

Storage bucket: `elt-files` (public bucket, anon upload policy)
RLS: disabled on all tables

---

## Architecture

### Auth
- Supabase Auth with email/password
- Shared storageKey: `gcc-suite-auth` across both portals
- Email confirmation: disabled
- `must_change_password` flag triggers password set modal on first login

### Portal Routing
- `portal_access = portfolio` → straight to Portfolio
- `portal_access = elt` → straight to ELT
- `portal_access = both` → portal selector screen on SIGNED_IN
- Portal choice stored in `localStorage` key `gcc-portal-choice`
- Cleared on sign out

### DB Access Pattern
All DB operations use direct REST fetch with anon key (not Supabase JS client — corporate network JWT issue).

```javascript
async function dbFetch(path, options={}) {
  const headers = {
    'apikey': SUPABASE_ANON_KEY,
    'Authorization': 'Bearer ' + SUPABASE_ANON_KEY,
    'Content-Type': 'application/json',
  };
  const res = await fetch(SUPABASE_URL + '/rest/v1/' + path, {
    method: options.method || 'GET', headers, body: options.body
  });
  ...
}
```

Storage operations use `sb.storage.from('elt-files')` via Supabase JS client.

---

## Permission Model

| Role | portal_access | can_edit | is_admin |
|---|---|---|---|
| Superadmin | both | true | true |
| Portal Admin | portfolio or elt | true | true |
| Editor | portfolio or elt | true | false |
| Viewer | portfolio or elt | false | false |

### Excluded from ELT submitters list
`jarmour@jaggaer.com` is excluded from the ELT dashboard submitters list and has the Submit nav hidden. Managed via `ELT_EXCLUDED` array in elt.html.

---

## Color System (PULSE Brand)

```css
--bg: #F4F5F7               /* page background */
--surface: #FFFFFF           /* cards */
--surface2: #F9FAFB          /* table heads, inputs */
--border: #E5E7EB            /* card borders */
--text: #111827              /* primary text */
--text2: #4B5563             /* secondary text */
--text3: #6B7280             /* muted text */
--blue: #C8161D              /* PULSE red — primary action color */
--sidebar-bg: #1E1E2E        /* dark sidebar */
```

Sidebar uses hardcoded hex values (`#1E1E2E`, `#3A3A50`, `#E8EAF0`) not CSS vars, to prevent theme bleed.

---

## Key Behaviors

### ELT Tool
- Week always starts on Monday (local date, not UTC)
- One submission per user per week — editing replaces, not duplicates
- Submission locked at noon ET Saturday
- Superadmins can edit any submission at any time
- File uploads: superadmin only, stored in `elt-files` Supabase Storage bucket
- CSV export includes: week, display name, full name, email, department, role, region, all 7 sections

### ELT Sections (7)
1. KPI / SPO Updates
2. Celebrations & Wins
3. Coaching Moments
4. Risks & Escalations
5. Weekly Execution
6. This Week's Priorities
7. Dependencies & Help Needed

### Portfolio Tool
- KPI definitions stored in `KPI_DEFS` array (in-memory, loaded from `kpi_values` table)
- Milestones and tasks stored as JSONB arrays on the project record
- All modals locked — no outside-click dismiss, X or Cancel only
- Project detail modal has sticky header with X button
- Budget fields removed entirely

---

## Dropdown Options

### Departments
GCC, CSM, PS, CS Ops, GCC Ops, GPT, Pre-Sales, Sales, Marketing, Account Manager, Partner Support, Global Advisory

### Regions
Global, US, EU, EMEA, APAC, LATAM

---

## Dev Workflow Rules

1. **Always build and test in dev first**
2. Generate prod files separately only after dev is confirmed working
3. Dev files: swap to dev Supabase credentials + add amber DEV ENVIRONMENT banner in sidebar
4. Never use `YOUR_SUPABASE_URL` placeholder in output files — always swap credentials before presenting files

### Dev file generation pattern
```python
dev = content.replace(PROD_URL, DEV_URL).replace(PROD_KEY, DEV_KEY)
dev = dev.replace('<div class="logo-sub">Operations Hub</div>',
                  '<div class="logo-sub" style="color:var(--amber)">DEV ENVIRONMENT</div>')
```

### ELT file generation pattern (master uses placeholders)
```python
prod_e = content.replace('YOUR_SUPABASE_URL', PROD_URL).replace('YOUR_SUPABASE_ANON_KEY', PROD_KEY)
dev_e = content.replace('YOUR_SUPABASE_URL', DEV_URL).replace('YOUR_SUPABASE_ANON_KEY', DEV_KEY)
dev_e = dev_e.replace('<div class="logo-sub">Weekly Readout</div>',
                      '<div class="logo-sub" style="color:var(--amber)">DEV ENVIRONMENT</div>')
```

---

## Working File Locations (Claude session)

| File | Path |
|---|---|
| Portfolio master | `/tmp/index_v15.html` |
| ELT master (placeholders) | `/home/claude/gcc-v2/elt_v2.html` |
| ELT master (prod creds) | `/home/claude/gcc-v2/elt.html` |

When starting a new Claude session, upload `index.html` and `elt.html` from your local **gcc-portfolio** folder. These become the working masters. Save them to `/tmp/index_v15.html` and `/home/claude/gcc-v2/elt_v2.html` (after stripping prod credentials for the ELT master).

---

## Known Issues / Watch Points

- Supabase JS client cannot be used for DB queries on corporate network (JWT parsing error). Use direct REST fetch only.
- Storage operations (upload/download/delete) use `sb.storage.from()` — this works fine.
- `profiles` table has no `created_at` column.
- ELT master file uses `YOUR_SUPABASE_URL` placeholders — always swap before output.
- `must_change_password` is guarded by `sessionStorage` key `pw-prompt-shown` to prevent repeated prompts.
