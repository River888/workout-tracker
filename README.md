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
  }
}
```

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

## Backup

Programme → Export downloads a JSON file containing `routines`,
`activeRoutine`, and `history`. Import merges by id/date without duplicates,
and still accepts old history-only (array) backups.
