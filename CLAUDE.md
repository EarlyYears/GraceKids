# GraceKids — Early Years Ministry Check-in System

Child check-in/check-out system for a church children's ministry. In active
Sunday use. Treat it as production software: a bad deploy means volunteers
cannot check children in or out on a Sunday morning.

## Shape of the project

Single-file app. Everything is `index.html` (~203KB, ~655 lines of very long
lines). There is no build step, no bundler, no `package.json`, no Node.
`tests.html` sits beside it — see Tests below.

- Lines 1–8: head
- Lines 9–172: `<style>`
- Lines 174–280: HTML body / markup
- Line 280: Chart.js 4.4.1 from cdnjs
- Lines 281–655: `<script type="module">` — the entire application

Because lines are extremely long, **navigate by `grep -n` for a function or
string, not by reading line ranges.** Reading the whole file will flood context.

## Stack

- Firebase 10.12.0 modular SDK (`firebase-app`, `firebase-auth`,
  `firebase-database`) loaded from gstatic CDN
- Firebase **Realtime Database**, project `early-years-grace-kids-reg`
- Firebase **Email/Password auth** (replaced an earlier hardcoded password)
- Chart.js for dashboard charts
- Hosted on **GitHub Pages** from `main` at
  https://earlyyears.github.io/GraceKids/

## Data model (Realtime Database)

    data/children      — child registry (names, birthdays, medical, family links)
    data/checkins/     — check-in records per session
    data/familyCodes/  — family codes used for check-in/check-out
    data/teachers      — per-class teacher counts
    data/snWorkers     — special-needs / One-to-One workers
    data/deskNotes/    — notes left at the desk, written per-note
    data/messageLog/   — one record per text sent, pruned after 60 days

## Tabs

`checkin`, `dashboard`, `registration`, `registry`, `medical`, `incidents`,
`reports`, `desknotes`

## Behaviour worth knowing before changing anything

- **Class assignment is birthday-driven.** Birthdays must be parsed as *local*
  dates — a past bug moved children up a class a day early due to timezone
  drift. Do not reintroduce UTC parsing.
- **One-to-One (1:1) care** requires 2 workers per child and has its own room
  placement, shown on tags, pop-ups and history. 1:1 children with no room must
  not silently fall back to their age class.
- **Per-record writes, not whole-collection writes.** Desk notes and children
  are written individually (`syncOneChild`, `syncOneDeskNote`). Writing the whole
  collection caused concurrent edits from different desks to wipe each other.
  Keep it that way — multiple desks run simultaneously on a Sunday.
- **Offline handling** is real and load-bearing. There is a `dirtyWrites` queue
  with retry, an offline banner driven by *two* signals (Firebase
  `.info/connected` and the browser offline event), and a sync status indicator.
- **Auto-reload on new build** — desks poll `index.html` and reload when a new
  build deploys, but only when idle. Uses a ranged GET (HEAD returned
  intermittent 503s from Pages).
- **Focus management matters.** After check-in, check-out, cancel and printing,
  focus returns to the search box so volunteers can keep typing. `afterprint` is
  used because the print dialog steals focus.
- **Printing** produces child tags; there is a `testPrint` path.
- **Identify children by name, never by list position.** Rows and dialogs used
  to embed `children.indexOf(c)` in handlers, but the sync listener rebuilds and
  re-sorts `children` on every change, so those positions go stale. That
  released the wrong children, printed the wrong tags and overwrote the wrong
  registration records. Handlers take a name and resolve it when clicked.
- **Dates must be worked out locally, never from `toISOString()`.** That returns
  UTC, which is a day ahead here from 20:00 in summer and 19:00 in winter. It
  caused every child to be auto-checked-out mid evening service. Use `localDay()`.
- **Escape anything a person typed** with `escHtml` before putting it on screen.
  A medical note reading "Give <half a tablet" is otherwise swallowed whole.
- **Every check-out goes through the same confirmation pop-up**, one child or
  five, and that path is what records `coTime`/`coVia`/`coCode`. Do not add a
  shortcut that skips it — the last one silently stopped recording check-outs.

## Deploying

Pushing to `main` deploys to GitHub Pages. There is no staging environment and
no CI. Deploys occasionally fail transiently — the history has several
"Re-trigger Pages deploy" commits. After pushing, confirm the live site actually
updated.

    git push origin main

## Safety rails

- **Never deploy untested changes on a Saturday or Sunday.** Volunteers depend
  on this working at service time.
- Tag before risky work. Existing convention:
  `known-good-YYYY-MM-DD-<reason>` and `checkpoint-YYYY-MM-DD-<reason>`.
- The app has a `downloadBackup` function — take a backup before schema or
  data-shape changes.
- This database holds **children's personal data**: names, birthdays, medical
  and allergy information, parent contact details and incident reports. Do not
  paste real records into commits, issues, logs or chat. Do not loosen the
  Firebase database rules — unauthenticated reads are currently correctly
  denied. The Firebase web API key in `index.html` is public by design; the
  database rules are what protect the data.

## Testing locally

    python3 -m http.server 8000

Then open http://localhost:8000. Note this connects to the **live** Firebase
database — there is no separate dev database, so any data you create or change
while testing is real.

## Tests

`tests.html` reads the real functions out of `index.html` and runs them against
known inputs, using invented children. Open it in a browser:

    python3 -m http.server 8000     # then http://localhost:8000/tests.html

It also deploys with the app, so https://earlyyears.github.io/GraceKids/tests.html
checks whatever version is actually live.

**Run it before every deploy.** It covers the things that have actually broken:
the birthday/class boundary, local dates, the evening-service auto-reset,
cancelling a family check-in, code matching, check-out recording, escaping,
phone normalisation and who may bump at capacity. Add a test whenever you fix a
bug — that is what stops it coming back.

## Commit style

Existing history uses short, specific, sentence-case subjects describing the
user-visible effect, e.g.
"Fix class assignment: parse birthday as local date so a child is not moved up
the day before their birthday (timezone bug)". Match it.
