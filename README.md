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
- Deleting a shift and approving a staff correction request are logged the same way.
- The timesheet CSV gained an **Edited by** column; the punch log CSV now includes the
  edit/delete events alongside the punches.
- The record is written to both the shift object (`entry.audit`) and the shared settings log
  (`settings.audit`), so it survives if the backend only keeps one of them.

## More flexible schedules
- Per-day schedule editor: **Rest day / Fixed hours / Split shift / Flexible hours** with time
  pickers, plus quick actions *Copy Monday to weekdays*, *Copy Monday to every day*,
  *All days off*.
- Split shifts keep both windows (`8:00 AM–12:00 PM + 1:00 PM–5:00 PM`) instead of being
  flattened into one span; overnight shifts still show `(next day)`.
- Flexible days are never counted as late and the clock reads
  “Flexible hours today — clock in whenever”.
- CSV import keeps both windows of a split shift and accepts `Flexible` per day.

## Files

| File | Purpose |
| --- | --- |
| `index.html` | The site — deploy this. |
| `index.test.html` + `_mock-api.html` | QA copy only: same page with an in-page mock of `/api/*`, so the flows can be exercised without a deploy. **Not for production.** |
| `serve.ps1` | Tiny local static server (`http://localhost:8787/`) for the QA copy. |
| `push-to-github.ps1` | Uploads `index.html` and `README.md` via the GitHub Contents API (this machine has no git binary). |

## Deploying

Drop `index.html` onto the existing Netlify site (or drag the folder onto
[netlify.new](https://netlify.new)). The Netlify functions that back `/api/*`
(`status`, `state`, `sync`, `login`, `signup`, `salt`, `logout`, `migrate`) live in the
site's `netlify/functions` folder and are **not** part of this repository.

After deploying, confirm once that the change history survives a reload on another device —
that proves the backend stores the extra `audit` fields.
