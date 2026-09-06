# Workout Tracker

A single self-contained `index.html` workout tracker. No build step, no
dependencies — open the file in a browser. Data persists in `localStorage`
(falls back to in-memory if storage is blocked, e.g. sandboxed previews).

## Running

Open `index.html` in any modern browser, or serve the folder:

```
python -m http.server 8000   # then visit http://localhost:8000
```

> **localStorage is per-origin.** Opening the file from a different path
> (e.g. `Downloads/` vs this folder) is a different origin, so old data
> won't appear automatically. Use **Programme → Backup → Export** in the
> old copy and **Import** in the new one to carry history + routines across.

## Data model (localStorage keys)

| Key              | Shape | Purpose |
|------------------|-------|---------|
| `routines`       | `Routine[]` | All workout routines you've defined |
| `activeRoutine`  | `string` (routine id) | Which routine drives the Today tab |
| `workoutHistory` | `Session[]` | Every logged session |
| `activeSession`  | `ActiveSession \| null` | The workout currently in progress (see below) |

```jsonc
// Routine
{
  "id": "seed-ppc",
  "name": "Fasted Push / Pull / Conditioning",
  "days": [
    {
      "code": "A", "focus": "Push", "color": "#E8590C",
      "muscles": "Chest · Shoulders · Triceps · Quads", "duration": "~50 min",
      "exercises": [
        { "name": "Dumbbell Bench Press", "sets": 4, "reps": "8–10",
          "rest": 90, "equipment": "...", "cues": "...", "why": "...",
          "superset": "Lateral Raise" }
      ]
    }
  ]
}

// Session (one entry per completed workout)
{
  "date": "2026-06-13T08:30:00.000Z",
  "day": "A",                 // day code at time of logging
  "focus": "Push",            // snapshot — so old sessions render correctly
  "color": "#E8590C",         //   even after the routine is changed/deleted
  "routine": "Fasted PPL",    // routine name snapshot
  "routineId": "seed-ppc",
  "exercises": {              // keyed by EXERCISE NAME (see below)
    "Dumbbell Bench Press": [ { "w": 30, "reps": 9 }, { "w": 30, "reps": 8 } ]
  },
  "notes": {                  // optional, one note per exercise
    "Dumbbell Bench Press": "Left shoulder tight — drop a notch next time"
  }
}
```

## In-progress workouts

Starting a day creates an `activeSession` in storage, rewritten on **every**
logged/edited set, note and toggle — so a refresh, a phone lock or an accidental
back-swipe never loses progress. Leaving the day (back button) deliberately does
*not* finalise it; the Today tab shows a **Resume** card instead.

A workout is written to `workoutHistory` only by the **Finish workout** button at
the bottom of the day. Reopening that day later the same calendar day loads the
saved session back for editing and finishing again *overwrites the same history
entry* (matched on `date`) rather than logging a second workout — so
`ActiveSession.resumeDate` is the date of the entry being continued, and
`startedAt` is when the workout began (it becomes the entry's `date`).

A session left open from a previous day is committed to history on next launch
rather than being discarded.

While a workout is in progress, any already-logged set stays editable in place —
the weight/reps pickers are live, and changing one updates the stored set.

## Exercise notes

Each exercise takes one free-text note per workout (not per set), via **Add note**
below *Add set*. The most recent note for that exercise from any earlier session
appears as a pressable **📝 Last note** pill next to the Last/PR line, and notes
are shown read-only in the History detail view.

## Key design decision: history is keyed by exercise *name*

Per-exercise progress (charts + PR) is computed by scanning **all** history
for a given exercise name, independent of which routine it belonged to. This
means:

- **Changing routines never affects history.** Routines and history are
  separate stores.
- **Reused exercises keep one continuous progress line.** If two routines
  both contain "Dumbbell Bench Press", they share the same chart and PR.
- Each session snapshots its day's `focus`/`color`/`routine` so historical
  workouts still display correctly after the routine is edited or deleted.

To keep names consistent across routines, the exercise-name field in the
routine editor offers autocomplete from every name you've ever used.

## Changing your routine

Programme tab → **Manage routines** (intentionally low-prominence — this is
a rare action). From there you can switch the active routine, create / edit /
duplicate / delete routines, and within a routine add/edit/reorder days and
exercises.

## Sharing routines

Manage routines → **Export routines** downloads just your routines (no history)
as `{ "type": "workout-routines", "version": 1, "routines": [...] }`. **Import
routines** adds routines from such a file without touching history or existing
routines (ids are regenerated so the same file can be imported repeatedly; a
clashing name gets an `(imported)` suffix).

[`routine-template.json`](routine-template.json) is a hand-editable starting
point with every field documented inline — edit it and import it.

## Weight entry

Weight and reps are native `<select>` pickers (Android shows its scroll wheel),
with −/+ steppers on weight. Dumbbell lifts snap to the user's adjustable
dumbbell stack (`2.5, 3.5, 4.5, 5.5, 6.5, 8, 9, 10, 11.5, 13.5, 16, 18, 20.5,
22.5, 24` kg, per dumbbell). Pick **Other…** in any picker to type an arbitrary
number. The **Dumbbell stack snapping** toggle (Programme → Settings) sets the
default; each exercise also has a per-session **Dumbbell / Other** switch and a
**1× / 2×** count. You log the weight of one dumbbell — at **2×** the tracked
load (PR, progress, volume) is doubled.

## Tracking types

Each exercise has a tracking type (set in the routine editor):

- **Weight + reps** (default) — the weight/reps pickers above.
- **Reps only** (e.g. Burpees) — logs reps, no weight; PR = best reps.
- **Timed** (e.g. Plank) — a **Start** button counts down the set duration, then
  auto-transitions into the rest timer. PR = longest hold.
- **Interval / boxing** — hands-free: one **Start** auto-cycles work → rest across
  all rounds (e.g. 3 × 3 min work / 1 min rest), beeping between phases so you
  never touch the screen.

## Programme insights

The **Weekly Volume Balance** is computed live: each exercise is classified to a
muscle group by name, then shown as *planned* sets (the active routine's main
days) against *logged* sets from the **last 7 days**.

## Backup

Programme → **Export all** downloads a JSON file containing `routines`,
`activeRoutine`, and `history`. Import merges by id/date without duplicates,
and still accepts old history-only (array) backups. **Clear all history**
(Programme → Backup) wipes logged workouts but keeps routines.
