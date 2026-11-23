# Daily Checklist — Design

## Intent

A personal checklist for today that its owner opens every morning and uses
throughout the day on several devices (laptop and phone). Unfinished items carry
over to the next day so nothing gets lost; finished items drop off so each
morning starts clean.

## Goals

- Show **today's tasks** on open: everything unfinished (including carried-over
  items) plus anything added today.
- Add a task, tick it done, untick it, edit its text, delete it.
- **Unfinished tasks carry over** automatically when the day changes.
- **Finished tasks clear** once their day has passed (they stay visible, ticked,
  for the rest of the day they were finished).
- Carried-over tasks show a small label with the day they were first added
  ("from yesterday", "from Fri", "from Nov 12"), so slipping tasks are visible.
- **Same list on every device**, updating within a couple of seconds.
- Usable on a phone browser and installable to the home screen.

## Non-goals

- Multiple users, sharing, or collaboration — one owner only.
- Due dates, reminders, notifications, priorities, tags, projects, subtasks.
- History or stats of past days (finished tasks are kept in the database, so
  this could be added later, but there's no UI for it).
- Full offline editing with conflict resolution.

## Anti-goals (would make it a failure even if it "works")

- Losing a task, or a task silently disappearing before it's done.
- The phone and laptop showing different lists.
- Slow or fiddly to add a task in the morning (adding must be one field + Enter).
- Needing to sign in every visit.

## Constraints

- React front end (owner asked for React).
- Free to run: Supabase free tier for data and sign-in; free static hosting
  (Vercel or Netlify).
- No server of our own.
- Lives in `daily-checklist/` in this repository for now (this repo is the
  Superpowers plugin; the app is self-contained so it can move to its own repo).

## Approaches considered

1. **React + Supabase (chosen).** Hosted Postgres with built-in email sign-in
   and realtime updates. Real app on every device, no server to maintain.
   Costs: a free Supabase account, a free host, ~15 minutes of setup, and a
   dependency on a third-party service.
2. **React + own small server.** Full control, but hosting, uptime and auth are
   on the owner — too much upkeep for a personal checklist.
3. **Page hosted on claude.ai with shared storage.** Almost no setup, but only
   reachable through Claude and not an app the owner controls.

The owner picked option 1.

## Decisions made with the owner

- Today's checklist, opened in the morning.
- Unfinished items carry over to the next day.
- Works across several devices.
- React + hosted database (Supabase).

## Design

### Data

One table, `tasks`:

| column | type | notes |
|---|---|---|
| `id` | uuid, pk, default `gen_random_uuid()` | |
| `user_id` | uuid, references `auth.users`, default `auth.uid()` | |
| `text` | text, not null | |
| `created_on` | date, not null | local date the task was added; drives the "from …" label |
| `done_on` | date, null | local date it was ticked; null = unfinished |
| `done_at` | timestamptz, null | when it was ticked; orders finished tasks *(builder's call)* |
| `position` | double precision, not null | display order |
| `created_at` | timestamptz, default `now()` | |

Row-level security: a user can select/insert/update/delete only rows where
`user_id = auth.uid()`.

### "Today" and carry-over

No nightly job is needed — carry-over falls out of the query:

> Show tasks where `done_on IS NULL` **or** `done_on = today`.

Where `today` is the **device's local date** (`YYYY-MM-DD`, computed in the
browser). Consequences:

- An unfinished task from any earlier day still appears → carried over.
- A task finished today stays visible, ticked, until midnight.
- A task finished on an earlier day no longer appears → cleared.
- Unticking a task sets `done_on` back to null.

Each device uses its own local date *(designer's call)*. If the phone and
laptop are in different time zones they can briefly disagree around midnight;
that's accepted for a single owner who's normally in one place. A shared
owner time zone can be added later if it matters.

The app recomputes `today` when the tab regains focus and at local midnight, so
a tab left open overnight rolls over correctly.

The "from …" label shows when `created_on < today`: "from yesterday" for one
day back, the weekday name within the last 6 days, otherwise a short date.

### Order

Unfinished tasks first, in `position` order; finished tasks below them, in the
order they were finished. New tasks are added at the bottom of the unfinished
group (`position` = max + 1). No drag-to-reorder in the first version.

### Sync

Supabase realtime subscription on `tasks` for the signed-in user. Changes are
applied optimistically in the UI and reconciled from the server; on realtime
events, update local state. On tab focus, refetch the list as a safety net.

If a write fails (e.g. offline), keep the change visible, show a small
"Not saved — retrying" notice, and retry; never drop the user's input.

*(Designer's call, from builder check.)* Unsaved changes go into a small
pending-changes queue in `localStorage`, replayed on the next open or when the
connection returns, so a task typed offline survives closing the tab or the
phone killing the app. The last list fetched is also kept in `localStorage`;
opened with no connection, the app shows that list with an "Offline — showing
last saved list" notice rather than an empty list. This is not full offline
editing: no conflict resolution beyond "last write wins".

### Sign-in

Supabase email sign-in with a **6-digit code typed into the app** *(designer's
call, from builder check)*. A magic link opens in the phone's browser, and on
iPhone a home-screen app keeps separate storage from Safari, so the link would
leave the installed app signed out. A code works the same everywhere. The
session persists, so each device signs in once. Sign-up is restricted to the owner: disable public sign-ups in
Supabase after the owner's account exists (documented in the setup guide).

### Screen

```
 Monday, Nov 23
 ─────────────────────────────
 [ ] Call dentist        from Fri
 [ ] Draft report
 [x] Reply to Sam
 ─────────────────────────────
 + Add a task...
```

- Header: today's date.
- Add field always visible; Enter adds; field stays focused for the next task.
- Tap checkbox to toggle. Tap text to edit inline; Enter/blur saves; empty text
  deletes after confirm. A delete control on each row.
- Empty state: "Nothing on today's list."
- Mobile-first layout; works with light and dark system themes.

### Installable

A web app manifest and icons so it can be added to the home screen. A service
worker that caches the app shell so it opens fast; data still comes from
Supabase (no offline editing).

### Stack

- Vite + React + TypeScript.
- `@supabase/supabase-js`.
- `vite-plugin-pwa` for manifest/service worker.
- Plain CSS (no UI library).
- Vitest + React Testing Library for tests; the date logic ("is this visible
  today", "from …" label) is pure functions with unit tests.

### Setup deliverables

- `supabase/schema.sql`: table, defaults, RLS policies.
- `.env.example` with `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY`.
- `README.md`: step-by-step for creating the Supabase project, running the SQL,
  enabling email sign-in, setting redirect URLs, deploying to Vercel/Netlify,
  disabling public sign-ups, and adding to the phone's home screen.

## Left to the builder

- Component structure and state management (plain React state/hooks expected).
- Exact styling.
- Retry/backoff details for failed writes.
- Which of Vercel or Netlify the README leads with.
