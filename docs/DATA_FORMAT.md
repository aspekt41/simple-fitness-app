# Data format

Status: **draft for review (Phase 1)**. Format version: `schema_version: 1`.

This document is the contract for everything stored in `DATA_DIR`. The files
are the source of truth. The app is just one program that reads and writes
them. Anything in this document should still make sense to someone holding
only a copy of the data folder and a text editor, years from now.

Contents:

1. [Summary of decisions](#1-summary-of-decisions)
2. [Why JSON](#2-why-json)
3. [Data directory layout](#3-data-directory-layout)
4. [File naming](#4-file-naming)
5. [Workout identity](#5-workout-identity)
6. [Workout file reference](#6-workout-file-reference)
7. [Sets are a list](#7-sets-are-a-list)
8. [Units](#8-units)
9. [Exercise identity and `exercises.json`](#9-exercise-identity-and-exercisesjson)
10. [Versioning, compatibility and migration](#10-versioning-compatibility-and-migration)
11. [Validation: errors and warnings](#11-validation-errors-and-warnings)
12. [How the app writes files](#12-how-the-app-writes-files)
13. [Derived data and the index](#13-derived-data-and-the-index)
14. [What analysis will use (Phase 4)](#14-what-analysis-will-use-phase-4)
15. [Using the data without the app](#15-using-the-data-without-the-app)
16. [Examples and JSON Schema](#16-examples-and-json-schema)
17. [Open questions for review](#17-open-questions-for-review)

---

## 1. Summary of decisions

| Topic | Decision |
|---|---|
| Encoding | JSON, UTF-8, one file per workout |
| Filename | Start time in UTC, ISO 8601 basic format: `20260922T140512Z.json`. Collisions get `_2`, `_3`, and so on. |
| Timestamp in file | RFC 3339 with the offset as entered: `"started_at": "2026-09-22T07:05:12-07:00"` |
| Workout ID | The filename stem (`20260922T140512Z`). No UUID in the file. |
| Versioning | Integer `schema_version`, bumped only for breaking changes. Old files are never rewritten unless you edit that workout or explicitly run a migration. |
| Unknown fields | Allowed at every level, and **preserved** when the app re-saves a file |
| Sets | A list of set objects, one per set actually performed |
| Minimum set | At least one of `reps`, `duration_s` or `distance` (see [open question 2](#17-open-questions-for-review)) |
| Load | Optional `weight: {value, unit}`. The unit is always explicit. |
| Units | Stored as entered, never normalized on disk. Converted at read time for display and analysis. |
| Exercise identity | `name` is always stored. An optional `exercise_id` points into an optional `exercises.json` library with aliases, merged IDs and muscle groups. |
| Derived data | Never stored. Volume, PRs, e1RM and indexes are recomputed from the files. |

## 2. Why JSON

I agree with JSON and see no reason to push back. It is universally parsable,
has exact serde support, diffs well line by line, and `jq` makes it scriptable.
Its two real weaknesses matter here and are handled:

- **No comments.** Every level has a `notes` field, and you can add your own
  fields (section 10.3), so there is somewhere to put anything you would have
  written as a comment.
- **Unforgiving syntax** (a trailing comma breaks the file). The app reports
  the file name, line and column and keeps showing every other workout.
  Section 11 covers this.

Rejected alternatives:

| Format | Why not |
|---|---|
| XML | No concrete advantage here. It is verbose and harder to hand-edit, and the attribute/element choice adds ambiguity. XSD validation is not better than JSON Schema for this use. |
| YAML | Implicit typing (`no` → false, `1e3` → number, `08` → octal in some parsers), indentation-sensitive edits, and a large, inconsistent spec. Easy to read, but easy to corrupt by hand. |
| TOML | Deeply nested arrays of tables (`[[exercises.sets]]`) are awkward to read and edit. |
| JSON5 / JSONC | Would allow comments and trailing commas, but `jq`, most languages' standard libraries and most tools can't read it. It gives up portability for convenience. |
| CSV | Can't represent the workout → exercise → set nesting without repeating data or splitting across files. It's a good *export* format for later. |

## 3. Data directory layout

```text
DATA_DIR/
├── workouts/
│   ├── 20260922T140512Z.json
│   ├── 20260924T223000Z.json
│   └── ...
├── exercises.json          # optional exercise library
└── trash/                  # deleted workouts (see open question 4)
```

- The app reads only `workouts/*.json` files whose names match the pattern in
  section 4, plus `exercises.json`.
- Any other files are left alone, so a `.git/` directory, a README or notes can
  live here. A `*.json` file in `workouts/` that doesn't match the pattern is
  **reported** on the workout list as ignored. It is never silently skipped.
- Temporary files from atomic writes are dot-prefixed (`.20260922T140512Z.json.tmp-…`)
  and ignored by the reader. Leftovers from a crash hold no data that was ever
  committed, and are removed at startup.
- Workouts sit flat in one directory. 300 workouts a year is 3,000 files in ten
  years, which is trivial for any filesystem and keeps `ls`, `grep` and `git`
  simple. Year subdirectories were rejected as unnecessary. They can be added
  later without changing the file format.
- Backup means copying the folder. Version history means `git init` inside it.

## 4. File naming

**Format:** `YYYYMMDDTHHMMSSZ.json`, the workout's `started_at` converted to
UTC, truncated to whole seconds, in ISO 8601 **basic** format.

Example: `"started_at": "2026-09-22T07:05:12-07:00"` → `20260922T140512Z.json`.

The loader uses the regex `^[0-9]{8}T[0-9]{6}Z(_[0-9]+)?\.json$`.

### Basic format versus extended format with substitution

| Option | Example | Verdict |
|---|---|---|
| ISO 8601 basic | `20260922T140512Z` | **Chosen.** A real, conforming ISO 8601 representation that any ISO 8601 parser (and jiff/chrono in Rust) understands. It contains only `[0-9TZ]`, so it's safe on Windows, SMB, FAT/exFAT, Syncthing, Dropbox and in URLs, with nothing to escape. Names sort in chronological order. |
| Extended, `:` → `-` | `2026-09-22T14-05-12Z` | Easier to read, but it isn't ISO 8601 anymore, so no tool parses it as a time. `14-05-12` could also be misread as a time with a negative offset. It's a private dialect. |
| Extended, `:` → `.` or `_` | `2026-09-22T14.05.12Z` | Same problems. `.` also reads like a fractional time in some conventions. |
| Mixed (`2026-09-22T140512Z`) | | Mixes basic and extended forms in one representation, which ISO 8601 doesn't permit. |
| Extended with `:` | `2026-09-22T14:05:12Z` | Breaks Windows, SMB and some sync tools. You already ruled this out. |

The readability cost of the basic format is small. The date part is still easy
to scan (`20260922`), and the full human-readable time with offset is on
line 4 of every file.

### Why UTC in the filename, but the local offset inside the file

- **Filename in UTC:** sorting by name matches real chronological order even
  across DST changes and travel. With local times, two workouts during the
  repeated 01:00–02:00 hour on the DST fall-back night would collide or sort
  wrong. A local time with an offset in the name (`20260922T070512-0700`)
  doesn't sort correctly across offsets.
- **File content keeps the local offset:** "07:05 in the morning" is what you
  experienced, and it's what weekly and time-of-day analysis should use.
  Converting to UTC would throw that away.

### Collisions

Two workouts can map to the same second, for example when times are entered
to the minute or when importing. The app picks the first free name in this
order: `20260922T140512Z.json`, then `20260922T140512Z_2.json`,
`20260922T140512Z_3.json`, and so on. `_` sorts after `.`, so the base name
sorts first. The app always sorts by the parsed `started_at` and then by
filename, so it doesn't depend on lexical order. The final rename must never
overwrite an existing file (section 12).

### When the filename and `started_at` disagree

This happens when you edit `started_at` in a text editor. **`started_at` is
authoritative for all data purposes.** The file keeps loading under its current
name, and the app shows a warning ("filename doesn't match started_at"). When
you next save that workout from the UI, the app moves it to its canonical name.
The app never renames files on its own initiative.

## 5. Workout identity

**Decision: the filename stem is the workout's ID** (`20260922T140512Z`,
`20260922T140512Z_2`). There is no separate `id` field.

Reasons:

- **The filesystem guarantees uniqueness.** An ID inside the file does not. The
  most natural way to hand-create a workout is to copy an existing file, and
  that silently duplicates any embedded UUID. A copied file needs a new name
  anyway.
- **One source of identity.** A UUID in the body plus a timestamp filename
  would be two identities that can disagree and would need reconciling.
- **Nothing references workouts yet.** v1 has no plans, programs or links
  between workouts.

Cost: if you change a workout's start time, its ID and URL change (and git sees
a rename). The UI redirects to the new URL after saving, so this is acceptable.

The door stays open: if something later needs to reference workouts stably, an
optional `id` field can be added in a backward-compatible way (section 10).

Security: IDs from requests are never joined into a path. The app looks them
up in the in-memory index of files it actually found, and the ID pattern
(`[0-9]{8}T[0-9]{6}Z(_[0-9]+)?`) can't express `/`, `\` or `..`.

## 6. Workout file reference

A minimal valid workout:

```json
{
  "schema_version": 1,
  "name": "Push day",
  "started_at": "2026-09-29T18:00:00+01:00",
  "exercises": [
    {
      "name": "Push-up",
      "sets": [
        { "reps": 20 },
        { "reps": 15 }
      ]
    }
  ]
}
```

Conventions for every object:

- An optional field that is absent means "not recorded". The app never writes
  `null`. On read, `null` is treated as absent, so `"rpe": null` is fine in a
  hand edit.
- Unknown fields are allowed and preserved (section 10.3).
- Arrays are ordered, and that order is the order performed.

### 6.1 Workout (top level)

| Field | Type | Req. | Meaning |
|---|---|---|---|
| `schema_version` | integer | **yes** | Format version. Currently `1`. |
| `name` | string, not blank | **yes** | For example `"Upper A"`, `"Long run"`. Used to suggest the next workout. |
| `started_at` | RFC 3339 timestamp | **yes** | Local wall-clock time **with an offset** (`Z` or `±HH:MM`). Timestamps without an offset are rejected, because "07:00" without a zone is ambiguous forever. |
| `ended_at` | RFC 3339 timestamp | no | End time. Used to show the workout's duration. |
| `bodyweight` | weight | no | Your body weight that day. Enables bodyweight-inclusive volume (section 6.5). |
| `notes` | string | no | Free text. |
| `exercises` | array of exercise | **yes** | May be empty, for example for a workout logged in advance and filled in later. |

### 6.2 Exercise

| Field | Type | Req. | Meaning |
|---|---|---|---|
| `name` | string, not blank | **yes** | Display name, as logged. Always present so the file is self-describing without the library. |
| `exercise_id` | slug | no | A stable key into `exercises.json` (section 9), for example `"bench-press"`. It must match `^[a-z0-9]+(-[a-z0-9]+)*$`, max 64 characters. |
| `notes` | string | no | |
| `sets` | array of set | **yes** | May be empty, for example for an exercise that was planned but skipped. |

### 6.3 Set

| Field | Type | Meaning |
|---|---|---|
| `warmup` | boolean | Warm-up set. Excluded from PRs, working volume and progression. Absent means `false`. |
| `reps` | integer ≥ 0 | Completed reps. `0` records a failed attempt, which is meaningful for heavy singles. |
| `target_reps` | integer ≥ 1 | Planned reps. Lets Phase 4 tell "hit 8 of 8" from "got 6 of 8" without guessing. |
| `weight` | weight | **External** load: the bar, dumbbell, machine stack or added belt weight. Absent means none recorded. |
| `duration_s` | number ≥ 0 | Seconds of work: plank hold, interval time, run time. Decimals allowed (`105.4`). |
| `distance` | distance | Distance covered. |
| `rpe` | number 1–10 | Rate of perceived exertion (halves are conventional: `7.5`). RIR isn't stored separately. RIR ≈ 10 − RPE. |
| `rest_s` | number ≥ 0 | Rest taken **after** this set, before the next. |
| `notes` | string | |

**Rule:** every set must have at least one of `reps`, `duration_s` or
`distance`. Every other field is optional and can combine freely.

When writing, the app orders set fields as listed above. `warmup` comes first,
so warm-up lines stand out at a glance.

### 6.4 Quantities

Any quantity whose unit is a user choice is stored as an object that keeps the
value and unit together. You can't write a weight without a unit.

```json
{ "value": 102.5, "unit": "kg" }
```

| Kind | Units (as written by the app) |
|---|---|
| weight | `kg`, `lb` |
| distance | `m`, `km`, `mi`, `yd` |

Time is always seconds, and the unit is in the field name (`duration_s`,
`rest_s`), because nobody logs time in a unit that changes its meaning. The UI
shows it as `mm:ss`.

Rejected representations:

- `"weight": "102.5 kg"`: easy to type, but it's a mini-language that every
  consumer, including `jq`, has to parse.
- `"weight": 102.5, "weight_unit": "kg"`: two keys that can separate. If
  someone deletes or forgets one line, the unit becomes implied.
- `"weight_kg": 102.5` / `"weight_lb": 225`: explicit, but that means two
  mutually exclusive keys per quantity, which multiplies for distance.
- ISO 8601 durations (`"PT25M"`): standard, but not greppable or summable in
  `jq`, and overkill for "seconds".

### 6.5 How different kinds of exercise fit

The same set object covers all of these. There is no per-exercise "type" field
to keep in sync with the data:

| Exercise | Set looks like |
|---|---|
| Barbell / machine | `{ "reps": 8, "weight": { "value": 80, "unit": "kg" } }` |
| Bodyweight (pull-up, push-up) | `{ "reps": 9 }` |
| Weighted bodyweight (belt dip) | `{ "reps": 8, "weight": { "value": 10, "unit": "kg" } }`, where the weight is the *added* load |
| Assisted (band / machine pull-up) | `{ "reps": 8, "weight": { "value": -20, "unit": "kg" } }`, where negative weight means assistance (see open question 6) |
| Timed hold (plank) | `{ "duration_s": 60 }` |
| Cardio | `{ "distance": { "value": 5, "unit": "km" }, "duration_s": 1587 }` |
| Intervals | One set per interval, with `rest_s` |
| Loaded carry | `{ "weight": …, "distance": …, "duration_s": 31 }` |

Whether the lifter's body weight counts as part of the load is a property of
the *exercise*, not the set, so it lives in the library
(`"includes_bodyweight": true`, section 9). The workout file stays readable
without the library. Analysis simply can't add body weight to volume unless it
knows both that flag and the workout's `bodyweight`.

**Conventions to follow (documented, not enforced):**

- **Dumbbells:** record the weight of *one* dumbbell, as you would say it
  ("30s"). Volume stays consistent within an exercise, which is what the
  analysis compares.
- **Unilateral work:** `reps` counts reps *per side*.

## 7. Sets are a list

```json
"sets": [
  { "reps": 8, "weight": { "value": 80, "unit": "kg" } },
  { "reps": 8, "weight": { "value": 80, "unit": "kg" } },
  { "reps": 6, "weight": { "value": 80, "unit": "kg" } }
]
```

rather than `{ "sets": 3, "reps": 8, "weight": 80 }`.

**Why:** the compact form can't represent what actually happened in most real
sessions:

- Reps that drop across sets (8, 8, 6). With `3 × 8` the missed reps are simply
  lost, and those are exactly the signal progression rules need.
- Weight that changes across sets: warm-ups, pyramids, drop sets, back-off sets.
- Per-set RPE, rest, notes and failed attempts.
- Sets that aren't reps at all: holds and intervals.

**Cost:** straight sets are verbose, with three near-identical entries for
3 × 8. Two mitigations:

- The one-line-per-set file layout (section 12.2) keeps a set to one line.
- The UI can pre-fill or copy the previous set so you don't retype it.

Rejected alternatives:

- **Both forms allowed** (`sets` as a list *or* a `sets × reps` shorthand):
  every consumer, every `jq` one-liner and every future migration would have to
  handle two shapes forever.
- **Parallel arrays** (`"reps": [8, 8, 6], "weights": [80, 80, 80]`): compact,
  but a hand edit that adds a rep count without a weight silently shifts every
  value after it.

## 8. Units

**Store as entered. Never normalize on disk.**

- **Lossless.** A 45 lb plate is exactly 45 lb. Storing it as 20.411656 kg and
  converting back means every value becomes a rounding question.
- **Honest.** The file records what happened in the gym, which is what a
  future you (or another program) should see.
- **Explicit.** Every weight or distance carries its unit. A missing unit is an
  error, not a default.

Conversion happens when the app reads data, using exact definitions:
1 lb = 0.45359237 kg, 1 mi = 1609.344 m, 1 yd = 0.9144 m.

- **Analysis** converts everything to kg and metres internally, so a workout
  logged in pounds at a hotel gym (example 2) still counts toward your squat
  history.
- **Display** uses your preferred unit (open question 5). Converted values are
  rounded for display, and the value as entered is shown next to them, for
  example `102.1 kg (225 lb)`.
- **Editing** shows and saves the value in the unit it was entered in. Opening
  a lb workout while your preference is kg never converts it on save.
- **Recommendations** (Phase 4) are expressed in the unit you last used for
  that exercise, so you won't get "+2.5 kg" for a lb-plate lift.

Reader leniency: `lbs` is accepted as `lb` and `kgs` as `kg`, case-insensitively,
because those are what people type. A unit the app doesn't recognize (say
`"stone"`) doesn't fail the file. The value is displayed as written, excluded
from unit-dependent analysis, and flagged with a warning. The JSON Schema lists
only the canonical spellings, so an editor with schema support will nudge you
toward them.

## 9. Exercise identity and `exercises.json`

**Problem:** "Bench Press", "bench press" and "Barbell Bench" must count as one
exercise, or PRs and trends fragment.

**Design:** two layers, and the second is optional.

1. Every exercise in a workout stores its display **`name`**. A workout file is
   always fully readable on its own.
2. An optional **`exercises.json`** library gives exercises stable slug IDs,
   aliases and metadata. Workouts can reference it through an optional
   **`exercise_id`**.

```json
{
  "schema_version": 1,
  "exercises": [
    {
      "id": "pull-up",
      "name": "Pull-up",
      "aliases": ["pullup", "pull ups", "pullups"],
      "previous_ids": ["pullups"],
      "includes_bodyweight": true,
      "primary_muscles": ["lats"],
      "secondary_muscles": ["biceps", "upper-back"]
    }
  ]
}
```

| Field | Req. | Meaning |
|---|---|---|
| `id` | **yes** | Stable slug. Never rename or reuse it. |
| `name` | **yes** | Canonical display name. |
| `aliases` | no | Other names that mean this exercise. |
| `previous_ids` | no | Retired IDs that now resolve here. This lets you **merge duplicates without editing any workout file**. |
| `includes_bodyweight` | no | Body weight is part of the load (section 6.5). |
| `primary_muscles`, `secondary_muscles` | no | Used to suggest what to train next. |
| `notes` | no | |

### Resolution: how the app groups exercises for analysis

Names are **normalized** before any comparison: trim, lowercase, and collapse
runs of whitespace to one space. So `"  Bench   press"` → `"bench press"`.

1. If `exercise_id` is present and matches a library `id` (or an entry's
   `previous_ids`), use that entry.
2. Otherwise, if the normalized `name` matches a library entry's normalized
   `name` or alias, use that entry.
3. Otherwise the exercise is **unlinked**. It's grouped with other unlinked
   exercises by normalized name, and the most recent spelling is displayed.
   An `exercise_id` that isn't in the library is kept as the grouping key and
   triggers a warning.

Library consistency is checked at load time. Duplicate `id`s, or a normalized
name/alias claimed by two entries, are reported as errors. Analysis then falls
back to names, and **workouts are never hidden because the library is broken.**

Suggested muscle vocabulary (not enforced, and unknown values are treated as
their own group): `chest`, `shoulders`, `triceps`, `biceps`, `forearms`,
`lats`, `upper-back`, `lower-back`, `abs`, `glutes`, `quads`, `hamstrings`,
`adductors`, `calves`, `cardio`.

When you pick an exercise from the library in the UI, the app writes both
`name` (the canonical name) and `exercise_id`. Resolution is then fixed at log
time and doesn't change if aliases change later. How new, unknown names get
into the library (automatically or through an "exercises" page) is a Phase 3
decision.

Rejected alternatives:

- **Library-only references** (`"exercise": "bench-press"` with no name): the
  workout becomes unreadable without the library, which breaks a core goal.
- **Normalized name only, no library:** handles case and whitespace but not
  "Barbell Bench" vs "Bench Press", and has nowhere to put muscle groups.
- **UUIDs for exercise IDs:** unreadable in files and diffs, and you can't type
  them by hand. Slugs are stable enough, since they're never renamed and
  `previous_ids` handles merges.
- **Fuzzy matching** (edit distance): surprising and not explainable. Explicit
  aliases are boring and predictable.

## 10. Versioning, compatibility and migration

### 10.1 What bumps `schema_version`

- **No bump (additive change):** a new *optional* field whose absence keeps the
  old meaning. Old app versions ignore it and preserve it (10.3).
- **Bump (breaking change):** removing or renaming a field, changing a field's
  type or meaning, making a field required, or adding a field that *changes
  how existing fields must be interpreted*. A hypothetical
  `"weight_is_per_side": true` would be an example, because old code would
  silently compute the wrong volume.

`exercises.json` has its own independent `schema_version`.

### 10.2 Reading old and new files

- The app can read **every version up to the one it knows**. Each old version
  has frozen structs and an in-memory upgrade step (v1 → v2 → … → current).
  Files are upgraded in memory only.
- **Old files are never rewritten** as a side effect of reading, upgrading the
  app or starting up. A file is written in the current version only when:
  1. you edit and save *that workout* in the UI (the edit page will say "saving
     will upgrade this file from v1 to v2"), or
  2. you explicitly run a bulk migration command, which will be opt-in and will
     recommend committing or backing up first.
- A file with a **newer** `schema_version` than the app knows is listed with
  the message "written by a newer version of the app (schema 2; this app
  supports up to 1)". It is **read-only**: the app won't parse it, edit it or
  delete it, so an old binary can never downgrade and lose data.
- The example files in `docs/examples/` become permanent test fixtures. When v2
  exists, the v1 files stay as they are and the tests keep proving that v1
  still loads.

### 10.3 Unknown fields are preserved

Tolerating unknown fields isn't enough on its own. If an older app loads a file
containing fields it doesn't know and then saves it, those fields must survive,
or a round trip through an old app quietly deletes data. So every object
(workout, exercise, set, quantity, library entry) keeps its unknown fields and
writes them back unchanged when saved. This will have a dedicated round-trip
test in Phase 2.

This also means **you can add your own fields**, for example `"x_location":
"office"` (example 4) or `"x_tempo": "3-1-1"` on a set. The suggested (not
enforced) convention is an `x_` prefix, so a personal field can never clash
with an official field added later.

## 11. Validation: errors and warnings

The app never crashes and never hides other workouts because of one bad file.
Each file is either **valid**, **valid with warnings**, or **invalid**. Invalid
files appear in a "problems" section of the workout list with the message, and
the other workouts display normally.

Messages give the file, the JSON path and, for syntax errors, the line and
column. For example:

```text
20260924T223000Z.json: exercises[0].sets[2].reps: expected an integer, found string "8"
20260924T223000Z.json: syntax error at line 14, column 5: trailing comma
```

**Errors (the file is not loaded):**

- not UTF-8, or not valid JSON (a UTF-8 BOM is tolerated and stripped)
- `schema_version` missing, not an integer, or newer than supported
- a required field is missing, blank or of the wrong type
- `started_at` / `ended_at` isn't RFC 3339, or has no offset
- a set has none of `reps`, `duration_s` or `distance`
- out-of-range values: negative reps or duration, non-positive distance, RPE
  outside 1–10
- a quantity without a `value` or `unit`
- a duplicate key in one object (for example two `"reps"` in a set)

**Warnings (the file loads and the warning is shown):**

- filename doesn't match `started_at` (section 4)
- `ended_at` is before `started_at`
- unrecognized unit (section 8)
- `exercise_id` not found in the library (only if a library exists)

The JSON Schema describes what the app *writes*. The app *reads* a slightly
larger set of inputs: `null` treated as absent, unit aliases, and a BOM. All
of these are listed above.

## 12. How the app writes files

### 12.1 Atomic, non-clobbering writes

1. Write to a dot-prefixed temp file **in the same directory** (so the rename
   stays on one filesystem).
2. `fsync` the temp file.
3. Rename it to the target name. For a new workout, and when a changed start
   time moves a workout, the target must not already exist. That is checked
   under the app's single-writer lock, so a collision picks the next `_N` name
   instead of overwriting.
4. `fsync` the directory. This is best-effort, because some network
   filesystems don't support it.

Moving a workout to a new name writes the new file first and removes the old
one second. A crash in between leaves two copies, which is visible and easy to
fix. It never leaves zero.

Edits made in a text editor while the app is running: the edit form will carry
a hash of the file it was based on, and saving is refused if the file has
changed since ("this workout was modified on disk; reload"). Your hand edit
can't be overwritten by a stale form. This will be implemented in Phase 3.

### 12.2 Layout on disk

- UTF-8, no BOM, `\n` line endings, trailing newline, 2-space indent.
- Fixed key order: `schema_version`, `name`, `started_at`, `ended_at`,
  `bodyweight`, `notes`, `exercises`, then unknown fields. This way `head`
  shows the useful part.
- **Each set, and each `{value, unit}` quantity, is written on one line.**
  Arrays of plain strings are written on one line.
- Optional fields that are absent aren't written. `null` is never written.

The output is deterministic: saving an unchanged workout produces a
byte-identical file, so git shows no diff. The one-line-per-set layout needs a
small custom JSON writer (about 60 lines, no extra dependency). Plain
`serde_json` pretty-printing would spread each set over 5–8 lines. See open
question 3. Readers accept any valid JSON formatting, so this is a writing
convention, not part of the format.

## 13. Derived data and the index

- Nothing derived is ever stored in a workout file: no totals, no volume, no
  e1RM, no resolved muscle groups. Derived values go stale, and a stale total
  in a hand-edited file would be a lie.
- The app keeps an **in-memory index only**, built by scanning `workouts/` at
  startup and updated after each of its own writes. At a few thousand small
  files a full scan takes well under a second, so there's no on-disk cache in
  v1. If one is ever added, it will live in `DATA_DIR/.cache/`, and deleting it
  will always be safe.
- How hand edits made while the app is running get picked up (a rescan when
  file modification times change, or a "rescan" button) is a Phase 2 decision.

## 14. What analysis will use (Phase 4)

Listed here so each field's reason for existing is visible. The exact rules
come in Phase 4, for your approval.

| Analysis | Uses |
|---|---|
| Volume per workout, week and exercise | Σ `reps` × `weight` (in kg) over non-`warmup` sets. Bodyweight exercises optionally add the workout's `bodyweight`. |
| Weekly buckets | ISO weeks by the **local** date of `started_at`, using its own offset |
| Personal records | Per resolved exercise: heaviest weight, best e1RM, most reps at a weight, longest `duration_s`, best pace (`duration_s` / `distance`) |
| e1RM trend | A standard formula (for example Epley), only for sets with a weight and a limited rep range. Phase 4 will propose the details. |
| Frequency | Count of `started_at` per week |
| Progression suggestion | `target_reps` vs `reps`, `rpe`, and the unit last used |
| What to train next | Recency by `primary_muscles` (library) or by workout `name` if there's no library |

## 15. Using the data without the app

Everything is plain JSON, so standard tools work. These were run against
`docs/examples/data/`:

```sh
# List workouts: start time and name
jq -r '[.started_at, .name] | @tsv' workouts/*.json

# Every working set of bench press, ever
jq -c '.exercises[] | select(.exercise_id == "bench-press")
       | .sets[] | select(.warmup != true)' workouts/*.json

# Total working reps per workout
jq -r '[.name, ([.exercises[].sets[] | select(.warmup != true) | .reps // 0] | add)] | @tsv' workouts/*.json

# Which workouts were logged in pounds?
grep -l '"lb"' workouts/*.json
```

Backup: copy the directory. History: `git init` in `DATA_DIR`. Each workout is
its own file, so commits from different devices rarely conflict.

## 16. Examples and JSON Schema

`docs/examples/data/` is a complete, valid `DATA_DIR`:

| File | Shows |
|---|---|
| `workouts/20260922T140512Z.json` | Strength session in kg: warm-ups, 8/8/6 reps against targets of 8, RPE, rest, bodyweight pull-ups, a weighted dip, workout `bodyweight` and `ended_at` |
| `workouts/20260924T223000Z.json` | Logged in **lb** while travelling (offset `-04:00`, versus `-07:00` at home), a missed rep, a timed plank with no reps |
| `workouts/20260926T150000Z.json` | Distance and duration: a run, rowing intervals with fractional seconds and rest, a loaded carry (weight + distance + time) |
| `workouts/20260928T191500Z.json` | Minimal hand-written file: no `exercise_id`, a lowercase alias name (`"push ups"` resolves to `push-up`), an unlinked exercise (`"Band curls"`), and a custom `x_location` field |
| `exercises.json` | Library with aliases, `previous_ids`, `includes_bodyweight` and muscle groups |

JSON Schemas are in `docs/schema/`:

- `workout.v1.schema.json`
- `exercises.v1.schema.json`

They're worth having because they give editor autocomplete and validation when
hand-editing, and a machine-readable description that outlives this code. The
Rust code remains the authoritative validator. For this phase, the examples
were validated against the schemas, and ten deliberately broken variants (no
offset, `"reps": "8"`, `"unit": "lbs"`, an empty set, `"exercise_id": "../etc"`,
and others) were confirmed to be rejected. Phase 2 will propose how to keep
that check in the test suite.

They use JSON Schema **draft 2020-12**. The official specification page still
lists it as the current version, and it has the broadest tool support. A newer
"v1/2026" release exists in the spec repository, but tool support for it is
still catching up. Moving to it later is a documentation-only change.

VS Code: map the schema to your data folder in settings (adjust the path to
where the schema file lives relative to your workspace):

```json
"json.schemas": [
  { "fileMatch": ["**/workouts/*.json"], "url": "./docs/schema/workout.v1.schema.json" }
]
```

A `$schema` key isn't put inside workout files. It would be noise in every file
and would tie the data to a URL.

## 17. Open questions for review

I made a choice on each of these and wrote the doc and examples accordingly.
Tell me which ones to change.

1. **Workout ID = filename stem, no UUID** (section 5). The alternative is an
   embedded UUIDv7, which gives stable URLs across start-time edits but risks
   duplicates when you copy a file.
2. **Relaxing your minimum "each set has a rep count"** to "at least one of
   `reps`, `duration_s` or `distance`" (section 6.3). Forcing `"reps": 1` on a
   plank or a 5 km run would pollute rep-based analysis. Pure strength logging
   is unaffected.
3. **One line per set on disk** with a small custom writer (section 12.2),
   versus plain `serde_json` pretty output (no custom code, but 3–5× more lines
   and harder to hand-edit).
4. **Delete = move to `DATA_DIR/trash/`** rather than an immediate unlink.
   Recovery is `mv trash/X.json workouts/`. Emptying the trash would be a manual
   step. Alternatively, rely on git or backups and delete for real.
5. **Preferred display unit** via an env var (`WEIGHT_UNIT=kg|lb`,
   `DISTANCE_UNIT=km|mi`) in v1, versus a `settings.json` in `DATA_DIR`. The
   env var keeps v1 to two file formats. A settings file would travel with the
   data.
6. **Negative weight = assistance** for assisted pull-ups and dips (section
   6.5), versus a separate `assistance` quantity. It's simpler, but slightly
   less obvious when hand-editing.
7. **Strict filename pattern**: non-matching `*.json` files in `workouts/` are
   reported as ignored, not loaded under whatever name they have (section 3).

Next, Phase 2: the storage library. Timestamp handling will use `jiff`
(0.2.x, actively maintained, with first-class RFC 3339 and RFC 9557 support)
or `chrono`. I'll confirm crate choices and versions
with you at the start of that phase.
