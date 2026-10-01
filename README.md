# Prestige Dealer Alliance Timelogs

Single-file time-clock app (front end) — the file Netlify serves as `index.html`.

Originally deployed as **Shift Clock** on `sundial-logs.netlify.app`; this copy adds:

## Big centre title
`Prestige Dealer Alliance` with a spaced `TIMELOGS` subtitle and accent rule at the top of
every screen (sign-in and app).

## Admin edit function
- Editing a shift writes an audit record: entry id, date, old → new times, timestamp and the
  **editor's username**.
- Each edited row is tagged `✎ edited by @username`.
- A **Change history** panel sits at the bottom of the timesheet listing every change with
  `by @username · date time`, plus a closing `Last change on this sheet by @username` line.
- The admin can **add**, **edit** and **delete** shifts on somebody else's behalf (the sheet
  has an *Add shift* button, every row has *Edit*, and the edit box has *Delete shift*).
- Deleting a shift and approving or declining a staff correction request are logged the same
  way, so the admin who did it is always named at the bottom of the sheet.
- The timesheet CSV gained an **Edited by** column; the punch log CSV now includes the
  edit/delete events alongside the punches.
- The record is written to both the shift object (`entry.audit`) and the shared settings log
  (`settings.audit`), so it survives if the backend only keeps one of them.

## Everything is approved by an admin
- Staff never change a logged shift directly: their rows only offer **Request edit**, which
  files a correction request with a note.
- The admin reviews requests under **Team → Edit requests**, can adjust the times being
  approved, and chooses **Approve** or **Decline**. The outcome and the admin's username are
  stored on the request (`resolvedBy`) and written into the change history at the bottom of
  that person's sheet.
- Schedules work the same way: the clock screen has **Request schedule change**, the week is
  set out in the same day-by-day editor, and the admin sees the schedule on file beside the
  one requested before approving it. Approving saves it and logs the admin's username.
- The server enforces this, not just the page: a person may only ever *add* to their own
  shifts (clock in, break, clock out), may not delete a shift, may not write the change
  history, and may not approve or decline their own request. Only an admin can change
  accounts, passcodes or schedules.

## What a staff account cannot do
- **Change a passcode** — the button is admin-only and the server refuses; an admin resets
  it from **Team → Reset passcode**.
- **Export** a timesheet or a punch log — the export buttons are admin-only.
- Change somebody else's record, turn sign-ups on or off, or switch off the last admin.

## Eastern time, 12-hour
- Every clock, time, schedule and CSV value is **US Eastern (America/New_York, EST/EDT)**
  shown in **12-hour** form (`9:05 AM`, `6:00 PM`) — the big clock included.
- Schedule and correction fields are written the same way (`9:00 AM`); the CSV import and
  `parseTime` still accept `09:00-18:00` as well.

## More flexible schedules
- Per-day schedule editor: **Rest day / Fixed hours / Split shift / Flexible hours** with
  12-hour time fields, plus quick actions *Copy Monday to weekdays*, *Copy Monday to every
  day*, *All days off*.
- Split shifts keep both windows (`8:00 AM–12:00 PM + 1:00 PM–5:00 PM`) instead of being
  flattened into one span; overnight shifts still show `(next day)`.
- Flexible days are never counted as late and the clock reads
  “Flexible hours today — clock in whenever”.
- CSV import keeps both windows of a split shift and accepts `Flexible` per day.
- Saving a schedule goes to the server straight away and is written into the change history
  like any other edit.

## Files

| File | Purpose |
| --- | --- |
| `index.html` | The site — the page Netlify serves. |
| `netlify/functions/*.js` | The backend: `status`, `salt`, `signup`, `login`, `logout`, `state`, `sync`, `migrate`, `health`. |
| `lib/store.js` | Shared storage and permission code the functions are bundled with. |
| `package.json` | Pulls in `@netlify/blobs` when Netlify builds from git. |
| `netlify.toml`, `_redirects` | Publish the repository root, bundle the functions, and map `/api/*` onto them. |
| `index.test.html` + `_mock-api.html` | QA copy only: the page with an in-page mock of `/api/*`. **Not for production.** |
| `backend.test.html` | QA only: runs the real function files against an in-memory Blobs stand-in (64 checks). **Not for production.** |
| `serve.ps1` | Tiny local static server (`http://localhost:8787/`) for the QA copies. |
| `push-to-github.ps1` | Old helper that uploaded `index.html` and `README.md` through the GitHub Contents API (this machine has no git binary). |

## The backend

Everything the page asks of `/api/*` now lives in this repository, so one deploy ships the
page and the API together.

| Endpoint | Method | Needs a sign-in | What it does |
| --- | --- | --- | --- |
| `/api/status` | GET | no | `{ needsSetup, signup }` for the sign-in screen. |
| `/api/salt?u=name` | GET | no | The salt to hash the passcode with; unknown names get a stable one of their own. |
| `/api/signup` | POST | no | Creates an account — the first one becomes the admin; returns `{ token, userId }`. |
| `/api/login` | POST | no | Checks the hash, returns `{ token, userId }`; 401 keeps the message the page shows. |
| `/api/logout` | POST | yes | Drops this device's token. |
| `/api/state` | GET | yes | The whole timesheet; your own passcode hash, never anyone else's. |
| `/api/sync` | POST | yes | Folds this device's changes in, replies with the full state. |
| `/api/migrate` | POST | no | One-time move of data in — only while the server is empty, otherwise **409**. |
| `/api/health` | GET | no | `{ node, read, write, conditional, ok }` storage self-test. |

Storage is **Netlify Blobs** at site scope — the same service the original backend used.

**Existing data.** The store name and key the original code used are not known from the
outside, so on first use the backend lists every store on the site and reads the keys inside
until it finds data shaped like the app state (`users` / `entries` / `requests` /
`settings`), whether that is one key holding everything or one key per collection. It then
reads and writes in that same place, so existing accounts, passcodes and timesheets carry on
unchanged. If it finds nothing, it starts blank on a store named `sundial-timelogs`, and
`/api/migrate` can be used to seed it (see below).

**Permissions** are enforced server-side: an admin may change anything — including adding,
editing and deleting shifts for other people. Anyone else may only clock their own punches
and file requests; the server refuses a staff passcode change, a rewrite of an already
recorded shift, a deleted shift, a change to somebody else's record, and approving their own
request — and nobody can switch off the last admin.

## Deploying

1. Push this repository to Netlify: *Add new site → Import an existing project* and pick
   `sundial-timelogs`. Netlify installs `package.json` (which is where `@netlify/blobs`
   comes from), bundles `lib/store.js` into every function with esbuild, publishes the
   repository root and wires `/api/*` through `netlify.toml`.
2. Confirm `/api/status` and `/api/health` answer on the new site, then sign in.

If the deploy is uploaded by drag-and-drop instead, dependencies are not installed and the
functions fail — build from git.

**Moving the data over.** On a site whose storage is still empty, sign in to the old
deployment, take a copy of its state (`GET /api/state` with the sign-in token), and post it
to `/api/migrate` on the new site together with `sessionUserId`. After that the accounts,
timesheets and change history are live on the new site, and people sign in with their usual
passcodes.

After deploying, confirm once that the change history survives a reload on another device —
that proves the backend stores the extra `audit` fields.
