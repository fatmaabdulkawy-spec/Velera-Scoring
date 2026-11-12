# Velera · Co-trainer Scoring

A scoring app for the co-trainers who play the roleplay calls in the Velera
leadership simulation. One co-trainer, one device, one team at a time: pick the
team you are roleplaying with, score the three rounds, and the trainer reads the
results on the same page.

It is a companion to the simulation itself (`velera-connect`), but it is a
**separate application with its own data store** — see *Why it is separate*
below.

---

## What's here

```
.
├── index.html     the whole app
└── README.md
```

One self-contained file. CSS and JavaScript inlined, no build step, no
dependencies, no package manager.

```bash
open index.html      # macOS
xdg-open index.html  # Linux
start index.html     # Windows
```

To host, drop it on any static host. The only external request is the Inter
webfont from Google Fonts, which degrades to Arial offline.

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

**Results** — every team with their three rounds, scores, endings and notes.
Project it in the debrief. A **Copy results** button puts the whole thing on the
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

Every submit writes **twice**:

1. **To the device**, into `localStorage` under `velera-scores-v1`. This always
   succeeds, so the Results tab and the team counters work regardless.
2. **To the shared store**, one document per team and round at
   `scores/<team-slug>_r<n>`, via the `db` capability. Everyone with the page
   open sees it live.

A status line at the top of the page says which is happening — green when
sharing is live, amber when it is not, with the reason. Each round's own message
confirms *Saved and shared* or *Saved on this device*.

Sharing only works for people inside the owner's organization with at least
Contributor access. Outside viewers on a public link read but never write. Worth
opening the page once on each co-trainer's device before a session and checking
the line is green; if it is amber, that co-trainer uses **Copy results** instead.

Scores are editable. Reopening a team and round that has already been scored
brings the selections back, pre-filled, with the previous note and scorer named,
and the button reads *Update score*.

---

## Why it is separate

A published page can only read **its own** data store. The simulation
(`velera-connect`) writes participants' entries to its store; this app writes
co-trainer scores to its own. Neither can see the other's.

So during a session the trainer runs two tabs:

- the simulation's `#board` view — what the teams wrote,
- this app's **Results** tab — what the co-trainers scored.

Merging them onto one live screen would mean moving scoring back inside the
simulation. That was a deliberate design decision, not an oversight.

---

## Known behaviour worth keeping

The snapshot callback uses `snap.docs.forEach(...)`, not `snap.forEach(...)`.
The runtime returns a `QuerySnapshot` with a `docs` array and no `forEach`; the
wrong form throws inside the listener and leaves the results silently empty.
The same bug still exists in the simulation's `#board` view.

---

## Pushing it

```bash
git init
git add .
git commit -m "Velera co-trainer scoring"
gh repo create velera-scoring --private --source=. --push
```
