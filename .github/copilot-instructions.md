# Habit Tracker — Copilot Instructions

## Project overview

Build a personal daily habit tracker as a single-page web app (vanilla HTML/CSS/JS, no
frameworks) hosted on GitHub Pages, with Supabase as the backend for authentication, data
storage, and dynamic habit management.

Key design principle: habits are NOT hardcoded. They live in a `habits` table in Supabase.
Users can add new habits, archive old ones, and reorder them — all from within the app.
Historical log data is always preserved even when a habit is archived.

A habit is "active on a date" if:
  `start_date <= that_date  AND  (end_date IS NULL  OR  end_date > that_date)`

Same UI throughout: green `#1D9E75` accent, dark mode support, mobile-first layout.

---

## Supabase setup (one-time, done in the Supabase dashboard)

### Step 1 — Create a Supabase project
- Go to https://supabase.com → New project (free, no credit card)
- Copy your **Project URL** and **anon public key** from Settings → API

### Step 2 — Enable Email auth
- Authentication → Providers → Email → Enable
- Disable "Confirm email" (it's a personal app — just you)

### Step 3 — Run the full SQL below in Database → SQL editor → New query

```sql
-- ─── habits table ─────────────────────────────────────────────────────────────
-- One row per habit per user.
-- end_date = NULL  →  habit is active now
-- end_date is set  →  habit is archived (hidden from daily log, history kept)

create table habits (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid references auth.users(id) on delete cascade not null,
  name        text not null,
  section     text not null default 'General',
  type        text not null default 'bool' check (type in ('bool','both')),
  sort_order  integer not null default 0,
  start_date  date not null default current_date,
  end_date    date,
  created_at  timestamptz default now()
);

alter table habits enable row level security;

create policy "Users manage own habits"
  on habits for all
  using  (auth.uid() = user_id)
  with check (auth.uid() = user_id);

create index habits_user_active on habits (user_id, end_date);


-- ─── habits_log table ──────────────────────────────────────────────────────────
-- One row per user per habit per day.

create table habits_log (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid references auth.users(id) on delete cascade not null,
  habit_id    uuid references habits(id) on delete cascade not null,
  log_date    date not null,
  done        boolean not null default false,
  log_time    time,
  note        text,
  created_at  timestamptz default now(),
  updated_at  timestamptz default now(),
  unique (user_id, habit_id, log_date)
);

alter table habits_log enable row level security;

create policy "Users manage own logs"
  on habits_log for all
  using  (auth.uid() = user_id)
  with check (auth.uid() = user_id);

create index habits_log_date  on habits_log (user_id, log_date);
create index habits_log_habit on habits_log (habit_id, log_date);


-- ─── Auto-update updated_at ────────────────────────────────────────────────────

create or replace function update_updated_at()
returns trigger language plpgsql as $$
begin new.updated_at = now(); return new; end;
$$;

create trigger habits_log_updated_at
  before update on habits_log
  for each row execute function update_updated_at();


-- ─── Seed: insert the 21 starting habits ──────────────────────────────────────
-- IMPORTANT: Sign up in the app first so auth.uid() resolves to your account.
-- Then come back here and run this INSERT block once.

insert into habits (user_id, name, section, type, sort_order, start_date) values
  (auth.uid(), 'Oil pulling',                    'Morning routine', 'bool', 1,  current_date),
  (auth.uid(), 'Stretch ups',                    'Morning routine', 'bool', 2,  current_date),
  (auth.uid(), 'Yoga / Pranayama',               'Morning routine', 'both', 3,  current_date),
  (auth.uid(), 'Acupressure',                    'Morning routine', 'bool', 4,  current_date),
  (auth.uid(), 'Water in the morning',           'Morning routine', 'bool', 5,  current_date),
  (auth.uid(), 'Dry fruits',                     'Morning routine', 'bool', 6,  current_date),
  (auth.uid(), 'Long walk / exercise',           'Movement',        'both', 7,  current_date),
  (auth.uid(), 'Dips / body building',           'Movement',        'bool', 8,  current_date),
  (auth.uid(), 'Sports',                         'Movement',        'bool', 9,  current_date),
  (auth.uid(), 'No grains – fruits until 2 pm',  'Nutrition',       'bool', 10, current_date),
  (auth.uid(), 'Dinner window: 2 pm till dark',  'Nutrition',       'bool', 11, current_date),
  (auth.uid(), 'Fasting after dark (12–18 hrs)', 'Nutrition',       'both', 12, current_date),
  (auth.uid(), 'No distractions',                'Mind',            'bool', 13, current_date),
  (auth.uid(), 'Writing',                        'Mind',            'both', 14, current_date),
  (auth.uid(), 'Positive affirmations',          'Mind',            'bool', 15, current_date),
  (auth.uid(), 'Fake it till you become it',     'Mind',            'bool', 16, current_date),
  (auth.uid(), 'Reading (1 book/month)',          'Mind',            'bool', 17, current_date),
  (auth.uid(), 'Listen to music',                'Mind',            'bool', 18, current_date),
  (auth.uid(), 'Coding – NeetCode',              'Learning',        'both', 19, current_date),
  (auth.uid(), 'Systems design',                 'Learning',        'both', 20, current_date),
  (auth.uid(), 'Floss + night brush',            'Evening',         'bool', 21, current_date);
```

### Step 4 — Keep-alive (prevents free-tier 7-day pause)
- Sign up free at https://uptimerobot.com
- Add an HTTP monitor pointing at your Supabase project URL
- Interval: every 3 days — keeps the project alive with zero effort

---

## File structure

```
/
├── index.html                      ← entire app: HTML + CSS + JS in one file
├── .github/
│   └── copilot-instructions.md    ← this file
└── README.md
```

No build tools. No npm. No bundler. No separate CSS or JS files.
Supabase JS via CDN only:
`https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.min.js`

---

## index.html — complete spec

### Config block (top of `<script>`, user fills these two lines in)

```js
const SUPABASE_URL      = 'PASTE_YOUR_PROJECT_URL_HERE';
const SUPABASE_ANON_KEY = 'PASTE_YOUR_ANON_KEY_HERE';
const supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
```

---

### App-level state variables

```js
let currentUser   = null;    // supabase User object, null when logged out
let currentDate   = today(); // 'YYYY-MM-DD' string — the date being viewed
let activeHabits  = [];      // rows from habits table active on currentDate
let dayLog        = {};      // { [habit_id]: { done, log_time, note } }
let activeTab     = 'log';   // 'log' | 'week' | 'settings'
let expandedHabit = null;    // habit_id whose time/note fields are visible, or null
```

---

### Date utility functions

```js
function today()              // → 'YYYY-MM-DD' for right now
function addDays(dateStr, n)  // → 'YYYY-MM-DD' offset by n days (negative = past)
function fmtDisplay(dateStr)  // → 'Today', 'Yesterday', or 'Tue 10 Jun'
function weekDates()          // → array of 7 'YYYY-MM-DD' strings for Sun–Sat of current week
```

---

### Data layer — habits table (dynamic habit definitions)

```js
// Fetch habits active on dateStr for current user.
// Active = start_date <= dateStr AND (end_date IS NULL OR end_date > dateStr)
// Order: section ASC, sort_order ASC
async function fetchActiveHabits(dateStr) { ... }

// Fetch ALL habits (active + archived) for the Settings manage view.
// Order: section ASC, sort_order ASC, then archived ones by end_date DESC
async function fetchAllHabits() { ... }

// Add a new habit. start_date = today(), end_date = null.
// Before inserting, query MAX(sort_order) for that section and use +1.
async function addHabit(name, section, type) { ... }

// Archive: set end_date = today(). Does NOT delete. All log history preserved.
async function archiveHabit(habitId) {
  await supabase.from('habits')
    .update({ end_date: today() })
    .eq('id', habitId).eq('user_id', currentUser.id);
}

// Restore: clear end_date, set start_date = today() so it appears from now.
async function restoreHabit(habitId) {
  await supabase.from('habits')
    .update({ end_date: null, start_date: today() })
    .eq('id', habitId).eq('user_id', currentUser.id);
}

// Edit a habit's name, section, or type (does not affect dates or history).
async function updateHabit(habitId, fields) {
  // fields is an object with any of { name, section, type }
  await supabase.from('habits').update(fields)
    .eq('id', habitId).eq('user_id', currentUser.id);
}

// Move a habit up or down within its section by swapping sort_order with neighbour.
async function reorderHabit(habitId, direction) { ... }
// direction: 'up' | 'down'
// Only swap within the same section. Clamp at boundaries (no wrapping).
```

---

### Data layer — habits_log table (daily tracking)

```js
// Load all log entries for currentUser on dateStr.
// Returns: { [habit_id]: { done, log_time, note } }
// Missing habits default to { done: false, log_time: null, note: null }
async function loadDay(dateStr) {
  const { data } = await supabase
    .from('habits_log')
    .select('habit_id, done, log_time, note')
    .eq('user_id', currentUser.id)
    .eq('log_date', dateStr);
  return Object.fromEntries((data ?? []).map(r => [r.habit_id, r]));
}

// Upsert a single habit log row.
async function saveHabit(dateStr, habitId, done, logTime = null, note = null) {
  await supabase.from('habits_log').upsert({
    user_id:  currentUser.id,
    habit_id: habitId,
    log_date: dateStr,
    done,
    log_time: logTime || null,
    note:     note    || null,
  }, { onConflict: 'user_id,habit_id,log_date' });
}

// Load log entries for an array of date strings (week view).
// Returns: { [dateStr]: { [habit_id]: { done, log_time, note } } }
async function loadWeekDays(dates) { ... }

// Calculate streak: how many consecutive past days this habit was done=true.
// Query habits_log for this habit ordered by log_date DESC (up to 90 rows).
// Count from yesterday backwards until first done=false or missing row.
async function getStreak(habitId) { ... }
```

---

### Auth flow

On page load:
1. `supabase.auth.getSession()` → if valid session, set `currentUser` and show app
2. If no session → show login screen
3. `supabase.auth.onAuthStateChange((event, session) => { ... })` → handles session
   expiry (SIGNED_OUT event → clear state, show login)

Login screen layout:
```
        🌿 Daily Habits
   ┌──────────────────────────┐
   │ Email                    │
   └──────────────────────────┘
   ┌──────────────────────────┐
   │ Password                 │
   └──────────────────────────┘
   [      Log in      ]  [Sign up]
   (error message in red if auth fails)
```

When logged in, header shows truncated email + [Sign out] button.

---

### Screen layout

```
┌─ Header (sticky) ──────────────────────────────────┐
│ 🌿 Daily Habits         user@email.com  [Sign out] │
│ [‹]  [  Today — Mon 9 Jun  ]  [›]                  │
│ [ Log ]   [ Week ]   [ Settings ]                  │
└────────────────────────────────────────────────────┘
┌─ Content (scrollable) ─────────────────────────────┐
│  (tab content rendered here)                        │
└────────────────────────────────────────────────────┘
```

Max width 600px, centred. Full height. Sticky header. Scrollable content below.

---

### Log tab — render flow

1. Show spinner
2. `activeHabits = await fetchActiveHabits(currentDate)`
3. `dayLog = await loadDay(currentDate)`
4. Compute streaks lazily (load per-habit when needed, cache in a Map)
5. Remove spinner, render:

```
┌─ Stats (3 cards) ──────────────────────────────────┐
│  Done     Progress    Left                          │
│  5/21       24%        16                           │
└────────────────────────────────────────────────────┘
[████░░░░░░░░░░░░░░░░░░░░░░░░░░░]  24%

MORNING ROUTINE
┌─────────────────────────────────────────────────┐
│ Oil pulling                          [  toggle ] │
│ Stretch ups               🔥 3       [● toggle ] │ ← done=true, green
│ Yoga / Pranayama                     [  toggle ] │
└─────────────────────────────────────────────────┘
  If done=true and type='both', show below the row:
  ┌── Time [07:30]  Note [free text field ──────] ─┘

MOVEMENT
...etc
```

**Toggle behaviour (optimistic UI):**
1. Flip the toggle immediately in `dayLog` and re-render the row
2. If toggling ON and `habit.type === 'both'`: set `expandedHabit = habit.id`
3. If toggling OFF: set `expandedHabit = null`, clear time/note fields
4. Call `saveHabit()` in background
5. On error: revert `dayLog`, re-render, show toast "Save failed — check connection"

Never show a loading spinner on individual habit toggles — stay optimistic.

---

### Week tab — render flow

1. `dates = weekDates()` (7 strings, Sun–Sat)
2. `weekData = await loadWeekDays(dates)`
3. Collect habits that were active on at least one day in `dates`
4. Render:

**Top — bar chart (7 columns):**
- Bar height: proportional to `done count / total active habits` that day × 60px max
- Today column: label in green, others in `--text3`
- Percentage label below bar
- No data for a day → bar height 0, show "–"

**Bottom — habit grid, grouped by section:**
```
MORNING ROUTINE
  Oil pulling      [●][●][○][●][●][○][●]
  Stretch ups      [●][○][○][●][○][○][●]
  ...
MOVEMENT
  ...
```
- `●` green filled = done
- `○` grey outlined = not done / habit existed that day
- hidden = habit was not active on that day (don't render a dot)

---

### Settings tab — three stacked cards

**Card 1 — Account**
```
Email: user@example.com
[Sign out]
```

**Card 2 — Manage habits**

Active habits, grouped by section with section header above each group:
```
MORNING ROUTINE
  Oil pulling           [↑][↓]  [Edit]  [Archive]
  Stretch ups           [↑][↓]  [Edit]  [Archive]
  ...
```

[+ Add new habit] expands an inline form at the bottom of the active list:
```
Name     [_______________________________]
Section  [Morning routine             ▾]   ← populated from distinct sections in habits table
                                            plus "＋ New section…" option at the bottom
Type     (●) Yes/No only   ( ) Yes/No + time & note
                                     [Add habit]  [Cancel]
```
When "＋ New section…" is selected, show a text input for the section name inline.

Edit mode: clicking [Edit] replaces that row with the same fields pre-filled. Save with [Save] / cancel with [×].

Archive: show a confirmation dialog/message "Archive this habit? All past data is kept." → [Archive] / [Cancel]. On confirm, call `archiveHabit()`, refresh the list, show toast "Archived. Past data kept ✓".

Archived habits — collapsed by default:
```
▶ Archived habits (3)
```
Expanded:
```
▼ Archived habits (3)
  Old habit name           [Restore]
  Another old one          [Restore]
```
[Restore] calls `restoreHabit()`, shows toast "Habit restored from today ✓", refreshes list.

**Card 3 — Export data**
```
[↓ Download JSON]
```
On click:
1. Query all `habits_log` rows for this user (no date limit), join with `habits.name` and `habits.section`
2. Build array of `{ date, habit_name, section, done, log_time, note }`
3. Trigger download of `habits-export-YYYY-MM-DD.json`
4. Show toast "Downloaded ✓"

---

### Section ordering rules

Sections are derived from the `section` column — no separate table.

Default section display order (when all present):
```
Morning routine → Movement → Nutrition → Mind → Learning → Evening
```
Any user-created sections appear alphabetically after "Evening".

Section dropdown in Add/Edit is populated by:
```sql
SELECT DISTINCT section FROM habits WHERE user_id = auth.uid() ORDER BY section
```
plus "＋ New section…" appended at the bottom.

A section disappears from the log automatically when all its habits are archived.

---

### Design tokens — replicate exactly

```css
:root {
  --green:       #1D9E75;
  --green-dark:  #0F6E56;
  --green-light: #E1F5EE;
  --green-text:  #0F6E56;
  --radius:      10px;
  --radius-sm:   7px;
}

/* Light mode defaults */
--bg:      #ffffff;
--bg2:     #f5f5f3;
--bg3:     #eeede9;
--text:    #1a1a18;
--text2:   #5f5e5a;
--text3:   #888780;
--border:  rgba(0,0,0,0.12);
--border2: rgba(0,0,0,0.22);

/* Dark mode overrides (@media prefers-color-scheme: dark) */
--bg:          #1c1c1a;
--bg2:         #252523;
--bg3:         #2e2e2b;
--text:        #f0efe9;
--text2:       #a8a79f;
--text3:       #6b6a64;
--border:      rgba(255,255,255,0.10);
--border2:     rgba(255,255,255,0.20);
--green-light: #04342C;
--green-text:  #5DCAA5;
```

Font stack: `-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif`

Component sizes:
| Element           | Spec                                              |
|-------------------|---------------------------------------------------|
| Toggle pill       | 38×23px, thumb 17px, border-radius 12px           |
| Toggle ON colour  | background `--green`, border `--green-dark`       |
| Toggle OFF colour | background `--bg3`, border `--border2`            |
| Streak badge      | 11px, bg `--green-light`, text `--green-text`, padding 2px 7px, border-radius 10px |
| Section label     | 10px uppercase, letter-spacing 0.08em, `--text3`  |
| Habit row         | padding 10px 12px, bg `--bg`, border 0.5px `--border`, border-radius `--radius` |
| Stats card        | bg `--bg2`, no border, `--radius`, padding 10px 12px |
| Progress bar      | 7px height, `--bg3` track, `--green` fill, border-radius 4px |

---

### Toast spec

```css
position: fixed;
bottom: 24px;
left: 50%;
transform: translateX(-50%);
background: #2c2c2a;
color: white;
padding: 9px 18px;
border-radius: 20px;
font-size: 13px;
z-index: 100;
pointer-events: none;
transition: opacity 0.25s, transform 0.25s;
```

Auto-dismiss after 2.5s. Show for errors, archive, restore, add, export. Do NOT show on every toggle.

---

### Spinner spec

```html
<div style="
  width: 32px; height: 32px;
  border: 2.5px solid var(--border2);
  border-top-color: var(--green);
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  margin: 40px auto;
"></div>
```

Show while: checking auth, loading habits+log on date change. Hide before rendering content.
Never show spinner on individual toggles (optimistic UI instead).

---

### Error handling rules

- All `supabase.*` calls in `try/catch`
- Auth expired → `onAuthStateChange` SIGNED_OUT event → clear state, show login
- Empty day (no log rows) = valid, render all habits unchecked — not an error
- Failed habit load → toast "Could not load habits", render empty list
- Failed toggle save → revert optimistic update, toast "Save failed — check connection"
- Failed add/archive/restore → toast with specific message, refresh list from DB

---

## README.md — generate this file alongside index.html

```markdown
# 🌿 Daily Habit Tracker

A personal daily habit tracker with a real backend. Add habits, archive old ones,
restore them later — history is always preserved. Works on mobile and desktop.

## Setup (10 minutes, everything free)

### 1. Supabase (the backend)
1. Go to [supabase.com](https://supabase.com) → New project (no credit card)
2. Open **Database → SQL editor → New query**
3. Copy the full SQL block from `.github/copilot-instructions.md` and run it
4. Go to **Settings → API** → copy your Project URL and anon key

### 2. Configure the app
Open `index.html`, find these two lines near the top of the `<script>` tag and paste your values:
```js
const SUPABASE_URL      = 'PASTE_YOUR_PROJECT_URL_HERE';
const SUPABASE_ANON_KEY = 'PASTE_YOUR_ANON_KEY_HERE';
```

### 3. GitHub Pages (the host)
1. Push `index.html` to your GitHub repo
2. Go to **Settings → Pages → Source: main branch → Save**
3. Open your GitHub Pages URL (e.g. `https://yourusername.github.io/habit-tracker`)
4. Sign up with your email
5. Go back to the Supabase SQL editor and run the seed `INSERT` block — this loads your 21 starting habits

### 4. Keep Supabase alive (optional but recommended)
Free Supabase projects pause after 7 days of inactivity.
Add a free HTTP monitor at [uptimerobot.com](https://uptimerobot.com) pointing at your
Supabase project URL, set to ping every 3 days.

## Managing habits

Open the app → **Settings → Manage habits**

| Action  | What it does |
|---------|-------------|
| Add     | New habit appears in daily log from today |
| Archive | Removed from daily log — all past data kept |
| Restore | Reappears in daily log from today |
| ↑ ↓     | Reorder within section |
| Edit    | Rename or change section/type |

## Tech stack

- Frontend: vanilla HTML / CSS / JS — no frameworks, no build step
- Backend: [Supabase](https://supabase.com) — Postgres + Auth + Row Level Security
- Hosting: GitHub Pages — free, always on, no server to manage
```

---

## What Copilot must NOT do

- Do not add React, Vue, Svelte, or any JS framework
- Do not add webpack, vite, parcel, or any bundler
- Do not add TypeScript
- Do not add a separate CSS or JS file — everything stays inside `index.html`
- Do not add a Node.js / Express backend — Supabase is the entire backend
- Do not hardcode the habits list — habits always come from the `habits` table
- Do not delete or truncate `habits` rows on app load — only read and upsert
- Do not use `localStorage` for habit data — session token storage is handled automatically by the Supabase JS client
- Do not add Google / GitHub OAuth unless explicitly asked
- Do not add a service worker or PWA manifest unless explicitly asked
