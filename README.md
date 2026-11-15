# Velera · Co-trainer Scoring

A scoring app for the co-trainers who play the roleplay calls in the Velera
leadership simulation. One co-trainer, one device, one team at a time: pick the
team you are roleplaying with, score the three rounds, and the trainer reads the
results on the same page.

It is a companion to the simulation itself (`velera-connect`), but it is a
**separate application with its own data store**: during a session the trainer
runs the simulation's `#board` view in one tab and this app's **Results** tab in
another.

---

## What's here

```
.
├── index.html     the whole app
├── api/scores.js  the shared score store (Vercel function)
└── README.md
```

The page is one self-contained file (CSS and JavaScript inlined), plus one
serverless function. No build step, no npm packages, no
package manager.

Preview locally:

```bash
open index.html      # macOS
xdg-open index.html  # Linux
start index.html     # Windows
```

Opened as a local file it works on that device only (the status line says *Not
connected*). Sharing between devices needs the Vercel deployment below.

---

## How it is used

Two tabs in the header.

**Score** — the working screen. Top to bottom:

1. The co-trainer's name, so the trainer knows whose judgement a score is.
2. **The team they are roleplaying with**, as tappable cards. Each shows how
   many of the three rounds that team has already had scored.
3. **All three rounds, laid out in full** — nothing hidden behind a selection.

Each round shows its character (Round 1 · Permission, N. Sabry, Head of Sales)
and, for each element, **all three scores written out as sentences**. The
co-trainer taps the description that matches what they saw rather than
interpreting a number:

> **Connection** — Did he ask about her?
> **0** Only talked about the rules and the money.
> **1** Asked about her, but defended himself first.
> **2** Found the real reason — she did not think he was ready.

Below the elements sits that round's **cap** in amber, then a field for *the
moment it turned, in their words*, then its own submit button. Rounds submit
independently, so three co-trainers can work three tables at once.

**Results** — a leaderboard, one line per team ranked by total (ties share a rank), with each round's score and ending. Below it, every note as its own line.
Project it in the debrief. **Copy results** puts the ranked lines and notes on the
clipboard as plain text.

---

## Scoring model

| Round | Elements | Out of | Endings |
|---|---|---|---|
| 1 · Permission | Connection, Attitude, Integrity | 6 | 5–6 Warm · 3–4 Truce · 0–2 Awkward |
| 2 · Production | Relationships, Vision, Motivation | 6 | 5–6 Warm · 3–4 Truce · 0–2 Awkward |
| 3 · People development | Mentor, Monitor | 4 | 3–4 Confident · 2 Dependent · 0–1 Fragile |

Sixteen in total. The header counter tracks the selected team's running total.

**Caps override the arithmetic**, which is why they are printed on screen rather
than left in the script. A participant who spends the first minute of Round 1
defending the deadline scores 0 on Connection whatever else happens; one who
never finds out why N. Sabry went around him cannot end warm.

---

## Editing

All of it is in `index.html`.

### Teams

```js
var TEAMS=['Team 1','Team 2','Team 3','Team 4','Team 5','Team 6'];
```

Rename or add freely. Scores are keyed by a slug of the team name, so renaming a
team after scoring orphans its existing records.

### The rounds, elements and descriptions

```js
var ROUNDS=[
  {n:'1', title:'Permission', who:'N. Sabry · Head of Sales',
   cap:'...',
   el:[ {k:'Connection', s:'Did he ask about her?', o:['...0...','...1...','...2...']}, ... ]},
  ...
];
```

`k` is the element name, `s` the question under it, `o` the three descriptions in
order 0, 1, 2. Add or remove elements freely — the maximum, the totals and the
headers all derive from `el.length`, so a four-element round needs no other
change. The endings thresholds are the one hard-coded part:

```js
function endingFor(n,total){
  if(n==='3') return total>=3?'Confident':(total>=2?'Dependent':'Fragile');
  return total>=5?'Warm':(total>=3?'Truce':'Awkward');
}
```

### Look

The palette is a handful of CSS custom properties on `:root` — Velera's green
and gold. Change `--gold` and `--bg` and the whole app follows.

---

## Where scores go

Scores are stored on the server, so every device that opens the page sees the
same leaderboard. The page checks for new scores every 4 seconds.

- `api/scores.js` is a Vercel serverless function: `GET /api/scores` returns
  all scores, `POST /api/scores` saves one (keyed `<team-slug>_r<n>`, so
  re-submitting a round replaces it).
- The data lives in an **Upstash Redis** database, in one hash called
  `velera:scores`.
- Each device also keeps a copy in `localStorage`. A score submitted while the
  connection is down is queued on that device and sent automatically once it is
  back.

The status line at the top is green (*Live*) when the server is reachable, and
amber when it is not.

Scores are editable. Reopening a team and round that has already been scored
brings the selections back, pre-filled, with the previous note and scorer named,
and the button reads *Update score*.

### Restoring scores

**Results → Restore scores** opens a panel with three options:

- **Restore from this device** — every device keeps its own permanent copy of
  every score it has seen (`localStorage` key `velera-archive-v1`), even after the
  shared list is cleared. This sends back any that are missing from the shared
  list, or newer than the shared copy. The panel says how many there are.
- **Restore from a backup file** — loads a file saved with *Download backup* and
  puts its scores back in the shared list, replacing the same team and round.
- **Download backup** — saves every score (shared plus this device's copies) to
  a `velera-scores-<date>.json` file. Worth doing at the end of each session.

### Clearing scores between sessions

Set an `ADMIN_KEY` environment variable in Vercel, then:

```bash
curl -X DELETE "https://<your-app>.vercel.app/api/scores?key=*" -H "x-admin-key: <ADMIN_KEY>"
```

`key=team-2_r1` deletes a single round instead of everything.

---

## Deploying on Vercel

1. In Vercel: **Add New → Project**, import this GitHub repository.
   Framework preset **Other**; leave the build command and output directory empty.
2. In the project: **Storage → Create Database → Upstash (Redis)**, free plan,
   and connect it to the project. This adds `KV_REST_API_URL` and
   `KV_REST_API_TOKEN` automatically. (`UPSTASH_REDIS_REST_URL` /
   `UPSTASH_REDIS_REST_TOKEN` from an Upstash account also work.)
3. **Redeploy** so the function picks up the new variables.
4. Open the site: the status line should read *Live*.

Anyone with the link can submit scores, so share it only with co-trainers.
