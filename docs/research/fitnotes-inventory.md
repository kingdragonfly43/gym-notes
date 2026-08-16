# FitNotes (Android) feature inventory

Research for [#3](https://github.com/kingdragonfly43/gym-notes/issues/3) — "What exactly is being
remade from FitNotes". Parent map: [#2](https://github.com/kingdragonfly43/gym-notes/issues/2).

**Subject**: FitNotes — Gym Workout Log, by James Gay. Android only. Package
`com.github.jamesgay.fitnotes`. Latest documented major version at time of writing: **Version 25**.

## How to read this document

Every feature carries a v1 marking:

- **in v1** — required by a decision already recorded on the map, or so load-bearing to the hot path
  that omitting it makes the app non-functional.
- **out of v1** — explicitly ruled out on the map, or explicitly deferred to the post-v1 backlog.
- **undecided** — the map has not ruled on it. These are the open questions this document surfaces;
  see [Decisions this inventory forces](#decisions-this-inventory-forces).

Confidence is marked per claim:

- **Confirmed** — stated on a first-party source (fitnotesapp.com official docs, the Google Play
  listing).
- **Community** — consistently reported by independent third-party tooling that reads FitNotes data.
  Treated as strong but not first-party.
- **Inferred** — my reading, not stated anywhere. Called out explicitly every time.

### Source trust

| Source | Trust | Notes |
| --- | --- | --- |
| <https://www.fitnotesapp.com/> and its 22 documentation pages | First-party | Official help site by the developer. Sitemap last-modified 2024-05-27. The primary source for almost everything here. |
| Google Play listing (`com.github.jamesgay.fitnotes`) | First-party | Play's own page truncates under automated fetch; description text was recovered via an APK mirror that reproduces the Play copy verbatim. |
| Third-party converters/readers (`alanjonesit/FitNotes2Hevy`, `makkoncept/fitnotes_mcp`, `sakerchr/FitNotes`) | Community | Used only for the export/backup file formats, which the official docs describe by name but not by structure. |

**FitNotes is not open source.** Despite the `com.github.jamesgay.*` package name, there is no
public FitNotes source repository. A GitHub repository search for `fitnotes in:name` returns only
third-party clones, converters and unrelated projects; there is no first-party repo under the
developer's name. So no claim below is grounded in source code, and the internal data model is
knowable only through the documentation, the CSV export and the SQLite backup.

---

## 1. User-visible data model

### 1.1 The shape

Confirmed structure, from the exercises, home screen and workout tracking docs:

```
Category (muscle group)
  └─ Exercise  ─ name, notes, category, exercise type, weight unit
       └─ Set  ─ some subset of {weight, reps, distance, time}, per exercise type
            └─ Comment (0..1 per set)

Workout (a dated day)
  ├─ ordered Exercises, each with its ordered Sets
  ├─ Workout comment (0..1 per day)
  └─ optional start/end time and duration
```

The unit of the day is a **workout** — a date. FitNotes does not name a session object beyond
the date. (Gym Notes' fixed vocabulary calls this a `Session`; the map's `Session`/`Exercise`/`Set`
triple maps cleanly onto FitNotes' day/exercise/set, with FitNotes' `Category` as a fourth level
above `Exercise`.)

| Element | Details | Confidence | v1 |
| --- | --- | --- | --- |
| **Category** | Muscle-group grouping above exercises. Ships with defaults; users can add, edit, delete and reorder them, including a custom (non-alphabetical) order. Categories carry a colour, used as dots on the calendar and optionally on the home screen. Deleting a category permanently deletes its data. | Confirmed | **in v1** — the exercise picker is part of the hot path and FitNotes' picker is category-first. |
| **Exercise** | Fields: name, notes (form tips, equipment, machine settings), category, exercise type, weight unit. Editable after creation. Deletion is irreversible: "all data belonging to the exercise…will be permanently deleted." | Confirmed | **in v1** |
| **Exercise type** (measurement type) | Two default types: **Weight + Reps** and **Distance + Time**. Eight advanced types behind the paid Supporter app: Weight + Distance, Weight + Time, Reps + Distance, Reps + Time, Weight only, Reps only, Distance only, Time only. | Confirmed | **partly out** — see 1.2. |
| **Type change is lossy** | "If you change an exercise's type then any fields present in the new type will be retained, but fields not present in the new type will be deleted from the exercise's training history." | Confirmed | Behaviour worth copying or deliberately improving on; **undecided**. |
| **Per-exercise weight unit** | Units are settable per exercise (Supporter feature, added in Version 24), for gyms mixing kg and lb plates. Changing the unit offers either converting the values (100 kg → 220.46 lb) or relabelling without conversion. | Confirmed | **undecided** — the map fixes no unit policy. |
| **Set** | A row under an exercise on a date, holding the fields its exercise type defines. Ordered within the exercise. | Confirmed | **in v1** |
| **Set comment** | Zero or one free-text comment per set ("rest duration or spotter assistance"). The comment icon turns blue when a comment exists. Deleting a set deletes its comment. | Confirmed | **in v1** — cheap, and it is the pressure valve that keeps the set row itself narrow. |
| **Workout comment** | A separate free-text comment against the whole day, via the home screen overflow menu. | Confirmed | **undecided** |
| **Workout timing** | Optional start and end time per workout, either by running a timer ("Start Timer") or entering times manually; duration is computed. Version 24 added auto-stop when all sets are marked complete. | Confirmed | **undecided** — note it interacts with live shared sessions, which have an inherent notion of "in progress". |
| **Exercise notes / per-exercise settings** | Per exercise: weight increment, rest time, default graph, and (Supporter) barbell configuration for the plate calculator. | Confirmed | **undecided** — per-exercise weight increment is the highest-value one for the hot path. |
| **Favourite exercises** | Version 24 added marking exercises as favourites, surfaced as a Favourites category — described as "a simpler alternative to workout routines". | Confirmed | **undecided** — cheap, and a plausible v1 substitute for the routines feature that is deferred. |

### 1.2 Measurement types against the v1 line

The map rules **cardio out of v1**. That removes the Distance + Time default type and, with it,
distance and time as first-class set fields.

- **in v1**: Weight + Reps.
- **out of v1**: Distance + Time (cardio), and all six advanced types that involve distance or time.
- **undecided**: the weight-free / rep-free advanced types — **Reps only** (bodyweight: pull-ups,
  dips, push-ups) and **Weight only**. Reps-only is *not* cardio and is not on the map's out-of-scope
  list, but it is also not in v1's stated scope. **Inference, flagged**: a resistance-training app
  without a reps-only type forces users to log bodyweight movements as `0 kg × n`, which FitNotes
  users do in practice but grumble about. This is the single most likely accidental v1 gap.

FitNotes charges for the advanced types. Gym Notes' monetization is fixed as banner-plus-remove-ads,
so the advanced types cannot be the paywall here; each is simply in or out.

---

## 2. The set-entry flow (the hot path)

Step by step, as documented in Quick Start and Workout Tracking. **Confirmed** throughout.

1. **Home screen** opens on today's workout. If empty, it shows **Start New Workout** or **Copy
   Previous Workout**.
2. **Tap the add button** → the Exercise List opens.
3. **Choose an exercise** — browse by category (muscle group), or search. Custom exercises can be
   created from here with the add button. (Version 24 added a Favourites pseudo-category.)
4. **The Training screen opens**, containing three regions: the **set fields**, the **set list** of
   sets already recorded, and the **action buttons**.
5. **The set fields are pre-populated** with "the values of the first set from the last workout" for
   that exercise. This is the load-bearing detail of the whole flow: the common case is that the
   user changes nothing, or taps `+` once.
6. **Adjust** — tap a field to type on the keyboard, or use increment/decrement buttons. The
   increment step is the default weight increment, overridable per exercise.
7. **Save** — the set is appended to the set list. Buttons read **Save** and **Clear**.
8. **Optionally comment** — tap the comment icon next to a set in the list.
9. **Repeat** for subsequent sets. The rest timer can auto-start on set creation.
10. **Edit**: tapping a set in the list selects it, highlights it, and loads its values into the
    fields; **Save/Clear** become **Update/Delete**. Adjust, then Update.
11. **Delete**: select the set, tap Delete. Its comment goes with it.
12. **Back to the exercise list** to add the next exercise. Nothing is explicitly "finished" — the
    app saves as you go and the day simply accumulates.

Supporting behaviours on the same screen:

| Behaviour | Detail | Confidence | v1 |
| --- | --- | --- | --- |
| Pre-population from last workout | First set of the previous session for that exercise | Confirmed | **in v1** — this is the feature that makes FitNotes fast. |
| Increment/decrement buttons | Step = default weight increment, per-exercise overridable | Confirmed | **in v1** |
| Rest timer | Countdown between sets; remembers previous durations; per-exercise rest time; vibration, sound, volume; auto-start on set creation | Confirmed | **undecided** — universally used, but it is a whole subsystem (notifications, background, audio). |
| Select-to-edit (same fields do create and edit) | Confirmed | **in v1** — it is why the screen stays small. |
| Mark sets complete | Optional checkboxes for working through a pre-planned session | Confirmed | **out of v1** — it exists to serve routines, which are deferred. |
| Auto-select next set | Auto-advances during routine-based or copied workouts | Confirmed | **out of v1** — same reason. |
| Supersets | Group exercises so the app auto-advances to the next exercise in the group after a set | Confirmed | **undecided** |
| Keep screen on | Prevents sleep while training | Confirmed | **undecided** — trivial, and it matters more for a shared live session, not less. |
| Drop-sets, half-reps | *Not FitNotes features.* FitNotes has no drop-set or half-rep concept documented. | Inferred (absence of evidence across the full doc set) | **out of v1** by map decision — note the decision rules out something FitNotes does not appear to ship either, so it costs no fidelity. |

---

## 3. History, graphs and personal records

All **Confirmed** from the Progress Tracking, Calendar and Settings docs.

### 3.1 Graphs

Selected per exercise; each exercise has a configurable **default graph**.

Weight + Reps exercises:

- Estimated One Rep Maximum — highest calculated 1RM per workout
- Max Weight — heaviest single-set weight
- Workout Volume — total weight × reps
- Total Reps
- Max Reps — highest reps in a single set
- Weight and Reps — max weight for a specific rep-count
- Rep Maxes — progression of personal records over time

Distance + Time exercises: Max Distance, Max Time, Max Speed, Max Pace, Total Distance, Total Time.

Time ranges: Workout, Week, Month, Year, All, or Custom (custom start/end dates added in Version 24).
Version 24 also added **yearly progress graphs** comparing total workouts, volume and sets year over year.

**v1**: all cardio graphs are **out of v1** with cardio. The strength graphs are **undecided** — the
map defers "analytics" to the post-v1 backlog but never says whether a single progress graph counts
as analytics. **Inference, flagged**: a workout tracker with zero progress view is a hard sell; if
one graph survives into v1, Estimated 1RM or Max Weight is the one FitNotes users cite.

### 3.2 Estimated 1RM

FitNotes uses the **Brzycki formula**, and its own glossary hedges it: "reasonably accurate for
max-effort sets of 2-10 reps but can become increasingly inaccurate as the number of reps increases
and so should be used as a guide only." **Undecided** for v1; if any estimate ships, the formula
choice should be an explicit decision rather than a copied default.

### 3.3 Personal records

- Both **Estimated** and **Actual** rep maxes are viewable.
- Precedence rule, quoted: "A Personal Record with a higher rep-count and an equal or larger weight
  will supersede those at a lower rep-count."
- A **Track Personal Records** setting turns on notifications for new strength records and puts an
  achievement icon beside qualifying sets.
- Settings includes a manual **recalculate personal records** action — implying PRs are stored
  derived state that can drift, not computed on read. (**Inferred** from the existence of the action.)

**v1**: **undecided**. The PR badge appearing on the set row is a hot-path-adjacent reward moment
and is cheap; the full estimated/actual rep-max table is not.

### 3.4 History and stats

- **History tab** per exercise: previous workouts with set detail. Tapping a workout date gives
  stats (total volume, total reps) plus actions to view the full workout, edit sets, or **copy sets
  into the current workout**.
- **Stats tab** with the period filter listed above.

**v1**: per-exercise history is **in v1** — it is the same data the pre-population reads, and users
check "what did I do last time" mid-set. Stats aggregates are **undecided**.

### 3.5 Calendar

- **Month view**: grid, with coloured dots under each day, one colour per muscle-group category
  trained. A **Category Dots** toggle switches between category colours and a single blue dot.
- **List view**: reverse-chronological summary of every logged workout.
- Tapping a date opens the workout; a Workout Panel supports swiping and prev/next between workouts,
  showing the total workout count.
- **Filtering**, applying to both views: by category (multi-select, match-all or match-any), and by
  exercise with parameters — the doc's example is "workouts where you performed 'Flat Barbell Bench
  Press' and lifted at least 100 kg for 5+ reps".

**v1**: month view with training-day marks is **in v1** (history navigation is core). Category-colour
dots, list view and the two filter systems are **undecided** — the exercise filter in particular is a
substantial query surface.

### 3.6 Home screen navigation

Home shows one day. Swipe left/right or use arrows to change day; tap the date to jump back to today.
Version 24 added **skip empty dates** when navigating. A configurable **set limit** (1–10) controls
how many sets show per exercise on the day view, with a first-or-last choice. Long-press enters a
selection mode for collapsing, reordering, replacing, supersetting, or deleting exercises, and for
copying or moving selected exercises to another date via the calendar.

**v1**: day view with date navigation and exercise reorder/delete is **in v1**. Copy/move to another
date and the display-tuning settings are **undecided**.

---

## 4. Routines

**Confirmed** from the Routines doc.

Structure:

```
Routine ─ name, notes
  └─ Day / section (e.g. "Monday" or "Push")
       └─ Exercise
            └─ Predefined Set ─ weight/reps/distance/time, or blank
       └─ Group (superset/circuit, shown as a coloured bar)
```

- A routine is "a means of storing and logging your frequently used workouts" — a **template**, not a
  schedule.
- Days may be named by weekday or by muscle group. There is **no scheduling or activation**: nothing
  assigns a routine day to a calendar date. The user picks the routine from the exercise-list
  dropdown when they want it.
- Predefined sets can hold fixed values, or be left blank to "automatically copy between workouts" —
  blank ones show as zero the first time and populate from the previous workout thereafter.
- **Log All** next to a day adds that day's exercises (with predefined sets) into the current workout.
- Routines are created from the Exercise List via "Create New Routine"; edit mode supports
  adding/removing exercises, drag reordering, renaming days, and grouping exercises into supersets.
  The overflow menu offers edit details, copy, delete, and reorder days.
- **No routine sharing or import** is documented.

**Relationship to a logged day**: strictly one-way and one-shot. Logging a routine day copies
exercises and predefined sets into the workout; after that the workout is an ordinary workout with no
link back to the routine. This is worth noting for Gym Notes' domain model — it means FitNotes has no
"planned vs actual" concept at all, which is a large simplification and probably the right one.

**v1**: **out of v1** — the map lists routines/programs in the post-v1 backlog. Two dependent
features (Mark Sets Complete, Auto-Select Next Set) go out with them. **Copy Previous Workout** and
**Favourite Exercises** are the cheap partial substitutes and are **undecided**.

---

## 5. Backup, export and import

| Mechanism | Detail | Confidence | v1 |
| --- | --- | --- | --- |
| **Manual backup** | Writes a `FitNotes_Backup.fitnotes` file to device storage or a cloud folder; optional timestamped filenames. Version 24 moved to the modern file-access APIs (folder picker, custom filename, no external-storage permission). | Confirmed | See below. |
| **`.fitnotes` file format** | A **standard SQLite database** containing the whole log. Not first-party documented, but independently confirmed by multiple third-party tools that open the file directly (an MCP server describes it as "workout data as a standard SQLite database with a decent schema out of the box"). | Community | — |
| **Restore** | Restores from a chosen `.fitnotes` file. Full replace. | Confirmed | — |
| **Automatic backup (legacy)** | Google Drive integration, fires roughly an hour after a workout, keeps the 5 most recent backups in the app's own Drive folder. | Confirmed | — |
| **Android/Google One backup** | Version 25 added support for the platform's own backup service — data rides along with the device's Google account backup and restores on a new device. The docs say the legacy custom Drive backup "may be phased out". | Confirmed | — |
| **Spreadsheet export** | CSV, for workout data and (separately) body tracker data; "compatible with Excel and Google Sheets". | Confirmed | — |
| **CSV columns** | `Date, Exercise, Category, Weight, Weight Unit, Reps, Distance, Distance Unit, Time` — one row per set, no workout-level wrapper. Taken from a converter's input validation, not from FitNotes' own docs. Whether a comment column is present is **unverified**. | Community | — |
| **Import** | There is **no CSV import and no import of any third-party format**. The only inbound path is restoring a `.fitnotes` backup that FitNotes itself wrote. | Confirmed (by absence across the full doc set) + Inferred | — |

**v1 marking for this section:**

- **out of v1**: importing FitNotes backups (map decision). Also, by extension, `.fitnotes` as a
  format Gym Notes reads.
- **in v1**: cloud backup for signed-in users (free, map decision) and manual export/import for
  not-signed-in users (map decision).
- **undecided**: the *format* of that manual export/import. The one non-obvious lesson here is that
  FitNotes' CSV export is **lossy and one-way** — flat per-set rows, no comments confirmed, no
  routines, no per-exercise settings — while its round-trippable format is the whole SQLite file.
  The map requires manual export/import to be a genuine escape hatch for users with no account,
  which means Gym Notes needs one format that actually round-trips, and CSV-as-FitNotes-does-it
  would not.
- **undecided**: whether the human-readable **Share Workout** text output (see 6) also counts.

---

## 6. Other surfaces

| Feature | Detail | Confidence | v1 |
| --- | --- | --- | --- |
| **Share Workout** | Generates "a readable, text-based representation of your workout" to share anywhere. | Confirmed | **undecided** — worth noting it is the closest thing FitNotes has to the sharing that is Gym Notes' whole differentiator, and it is a one-way text dump. That is the bar being cleared. |
| **Body Tracker** | Separate from workouts. Body Weight and Body Fat enabled by default; other common measurements available but off. Custom measurements with name, optional unit, optional goal. Units: kg, lb, cm, in, %, plus custom. Goals are Increase / Decrease / Specific Value, driving green/red arrows and a goal line on the graph. Graph and History screens, with deltas, filterable. Its own CSV export. | Confirmed | **undecided** — a whole second data model, absent from the map entirely. |
| **1RM Calculator** | Enter weight and reps, get estimated 1RM plus 2RM…15RM "useful in planning the weight to use in your subsequent sets". | Confirmed | **undecided** |
| **Set Calculator** | Percentage-of-1RM set generation for programs like 5/3/1: pick a max from personal records, pick percentages, round to barbell increments, add the calculated sets straight into the workout. | Confirmed | **out of v1** — inference, flagged: it is a programming tool and belongs with routines. |
| **Plate Calculator** | Shows the plates needed per side for a weight. Configurable available plates and bar weights, with per-exercise barbell settings; defaults differ for metric and imperial. | Confirmed | **undecided** |
| **Theme** | Light or dark. Dark theme arrived only in Version 25, eleven years after launch. | Confirmed | **undecided** |
| **Unit system** | Global Metric (kg) / Imperial (lb), with per-exercise override behind Supporter. | Confirmed | **in v1** — a global unit choice is unavoidable; the per-exercise override is **undecided**. |
| **Calendar week start** | Monday, Saturday or Sunday, defaulting from locale. | Confirmed | **undecided** |
| **Delete workout history** | Bulk delete by date range or by exercise, preserving routines and custom configuration. | Confirmed | **undecided** |
| **Recalculate personal records** | Manual repair action for the derived PR data. | Confirmed | **out of v1** if PRs are out; otherwise a symptom to design away rather than copy. |

---

## 7. Monetization and ads

**This is the sharpest divergence between FitNotes and Gym Notes v1, and it is worth stating
plainly.**

Confirmed facts:

- FitNotes is **free and carries no advertising at all**. The Play listing's own copy: "A clean,
  simple, powerful workout tracker. Free to use, and no ads - ever." The site repeats it: "Free to
  use and no ads - ever!"
- There are **no in-app purchases inside FitNotes itself**, and no subscription.
- Monetization is a **separate paid companion app**, **FitNotes Supporter**
  (`com.fitnotesapp.fitnotessupporter`), bought on Google Play. Installing it unlocks features in the
  main app: the eight advanced exercise types, and per-exercise weight units.
- The FAQ frames it as a donation first and an unlock second — it exists "to help fund development",
  and the stated alternative for users who do not want to pay is simply to rate the app.
- **Price**: reported as **$6.99** by app-directory sources. **Not confirmed first-party** — the Play
  page did not yield a price under automated fetch, and the FAQ does not state one. Treat the figure
  as indicative; verify on-device before using it as a pricing anchor.
- There is **no ad placement to inventory**, because there are no ads. Nothing in FitNotes'
  layout reserves space for a banner.

**v1 marking**: the entire FitNotes monetization model is **out of v1** — superseded by the map's
decision (static bottom banner plus a one-time remove-ads IAP).

Two consequences that the "remake of FitNotes" framing hides, both worth surfacing:

1. **Gym Notes v1 is a strictly worse deal than FitNotes on the ads axis**, for the free user. The
   comparison users will make is not against a neutral baseline; it is against an app whose store
   listing brags "no ads - ever". The differentiator (live shared sessions) has to carry that.
2. **The banner has nowhere obvious to go.** FitNotes' home screen and training screen are dense and
   bottom-anchored — the action buttons (Save/Clear/Update/Delete) sit at the bottom of the training
   screen, which is exactly where a bottom banner lives. A remade layout has to solve this, and
   "mid-set mis-tap on an ad" is the specific failure the map's no-interstitials decision was
   protecting against. **Inference, flagged** — screen layout is from documentation screenshots
   described in prose, not from hands-on use.

---

## 8. Summary tables

### In v1

| Feature | Why |
| --- | --- |
| Categories, exercises, sets, per-set comments | Core data model |
| Weight + Reps exercise type | The only surviving measurement type after cardio is cut |
| Custom exercises and custom categories, reorderable | Confirmed FitNotes behaviour, unavoidable |
| Set-entry flow: pick exercise → training screen → pre-populated fields → +/− → Save | The hot path |
| Pre-population from the last workout's first set | The single feature that makes FitNotes fast |
| Select-a-set-to-edit, with Save/Clear ↔ Update/Delete | Keeps the screen small |
| Home screen day view with date navigation | Core |
| Exercise reorder and delete within a day | Core |
| Month calendar with training days marked | History navigation |
| Per-exercise history ("what did I do last time") | Same data the pre-population reads |
| Global metric/imperial unit choice | Unavoidable |
| Free cloud backup for signed-in users; manual export/import otherwise | Map decision |
| Banner ad plus one-time remove-ads IAP | Map decision (diverges from FitNotes) |

### Out of v1

| Feature | Why |
| --- | --- |
| Cardio: Distance + Time type, and all six distance/time advanced types | Map decision |
| All cardio graphs (max/total distance, time, speed, pace) | Follows cardio |
| Drop-sets, half-reps | Map decision — and FitNotes appears not to ship them either |
| Reading FitNotes `.fitnotes` backups or CSV | Map decision |
| Routines: days, predefined sets, Log All, routine groups | Post-v1 backlog |
| Mark Sets Complete; Auto-Select Next Set | Exist only to serve routines |
| Set Calculator (percentage-of-1RM programming) | Inference, flagged — belongs with routines |
| Paid-companion-app monetization; paywalled exercise types | Superseded by the map's monetization decision |

### Undecided — and why each is a real question

| Feature | The question it poses |
| --- | --- |
| **Reps-only / Weight-only exercise types** | Bodyweight work is resistance training, not cardio. Without reps-only, pull-ups get logged as `0 kg × n`. Most likely accidental gap in v1. |
| **Rest timer** | Near-universally used, but a whole subsystem (notifications, background execution, audio) — and it competes for the same screen real estate as the ad banner. |
| **Any progress graph at all** | The map defers "analytics" without saying whether one graph counts. Zero progress view is a hard sell. |
| **Personal record badges** | The badge on the set row is cheap and is the app's only reward moment. The full estimated/actual rep-max table is not cheap. |
| **Estimated 1RM and its formula** | If any estimate ships, Brzycki should be a decision, not a copied default. |
| **Workout-level comment** | Distinct from set comments; cheap. |
| **Workout start/end time and duration** | Interacts directly with live shared sessions, which have an inherent "in progress" state. |
| **Per-exercise settings** (weight increment, rest time, default graph) | Per-exercise weight increment is the highest-value one for the hot path. |
| **Per-exercise weight units** | Real for mixed-plate gyms; FitNotes charges for it. |
| **Lossy exercise-type change** | Copy FitNotes' destructive behaviour, or do better? |
| **Favourite exercises** | Cheap, and the natural v1 stand-in for deferred routines. |
| **Copy Previous Workout** | The other cheap stand-in for routines. |
| **Copy/move exercises to another date** | Repair path for logging on the wrong day. |
| **Supersets** | Affects both the data model (grouping) and the set-entry flow (auto-advance). Decide before the model is fixed. |
| **Calendar category-colour dots; list view; category and exercise filters** | The exercise filter is a substantial query surface; the rest are cheap. |
| **Body Tracker** | An entire second data model, absent from the map. |
| **Plate Calculator, 1RM Calculator** | Self-contained, no data-model cost, real user value. |
| **Share Workout (text)** | FitNotes' only sharing. Overlaps Gym Notes' differentiator. |
| **Keep screen on** | Trivial, and it matters *more* during a shared live session. |
| **Dark theme** | FitNotes took eleven years. Users noticed. |
| **Delete workout history in bulk** | Data-hygiene escape hatch. |
| **Manual export/import format** | The map requires it; FitNotes' own CSV is lossy and one-way, so it is not the model to copy. |

---

## 9. Decisions this inventory forces

Ordered by how much they constrain the data model, which is being designed next:

1. **Reps-only exercises in or out.** Changes the exercise-type enum and every set-row layout.
2. **Supersets in or out.** Adds a grouping level between exercise and workout if in.
3. **Per-exercise settings and per-exercise units in or out.** Changes the exercise record.
4. **Personal records in or out**, and if in, stored-derived (FitNotes, which then needs a repair
   action) or computed-on-read.
5. **Workout-level fields**: comment, start/end time, duration — and how they reconcile with a live
   shared session's own lifecycle.
6. **Body Tracker in or out.** Wholly separate model; cheapest decision to defer, most expensive to
   retrofit into a backup/sync schema later.
7. **The manual export format**, given that it must round-trip and FitNotes' CSV does not.
8. **Where the banner goes**, given that FitNotes' bottom-anchored action buttons occupy that space.

## 10. Honest gaps

Things I could not confirm and did not guess:

- **No source code exists to check.** FitNotes is closed-source; the internal schema, exact field
  types, and constraints are unverified.
- **The `.fitnotes` SQLite schema itself** — confirmed to be SQLite, but I did not open one, so table
  and column names here are absent rather than approximate.
- **CSV export columns** come from a third-party converter's validation list, not from FitNotes'
  documentation. Whether comments are exported is **unknown**.
- **The Supporter app's price** ($6.99) is from app directories, not first-party.
- **The Play listing's structured metadata** — install count, "Contains ads" / "In-app purchases"
  flags, Data Safety section — could not be fetched; Play's page truncates under automated retrieval.
  The no-ads and no-IAP claims here rest on the developer's own description copy on both Play and
  the official site, which is first-party but is marketing text.
- **Nothing here is from hands-on use of the app.** Every flow description is the documentation's
  account of itself. Where the docs are silent — exact tap counts, keyboard behaviour, how the
  training screen actually feels at set five — this document is silent too. If the set-entry hot path
  is going to be matched or beaten, someone should install FitNotes and count the taps.

---

## Sources

First-party:

- [FitNotes home](https://www.fitnotesapp.com/)
- [Quick Start](https://www.fitnotesapp.com/quick_start/)
- [Workout Tracking](https://www.fitnotesapp.com/workout_tracking/)
- [Exercises](https://www.fitnotesapp.com/exercises/)
- [Home Screen](https://www.fitnotesapp.com/home_screen/)
- [Calendar](https://www.fitnotesapp.com/calendar/)
- [Progress Tracking](https://www.fitnotesapp.com/progress_tracking/)
- [Routines](https://www.fitnotesapp.com/routines/)
- [Workout Tools](https://www.fitnotesapp.com/workout_tools/)
- [Body Tracker](https://www.fitnotesapp.com/body_tracker/)
- [Settings](https://www.fitnotesapp.com/settings/)
- [FAQ](https://www.fitnotesapp.com/faq/)
- [Glossary](https://www.fitnotesapp.com/glossary/)
- [Release notes — Version 24](https://www.fitnotesapp.com/release_24/), [Version 25](https://www.fitnotesapp.com/release_25/)
- [Google Play — FitNotes](https://play.google.com/store/apps/details?id=com.github.jamesgay.fitnotes)
- [Google Play — FitNotes Supporter](https://play.google.com/store/apps/details?id=com.fitnotesapp.fitnotessupporter)

Community (used only for file formats and the Supporter price):

- [alanjonesit/FitNotes2Hevy](https://github.com/alanjonesit/FitNotes2Hevy) — CSV export columns
- [makkoncept/fitnotes_mcp](https://github.com/makkoncept/fitnotes_mcp) — `.fitnotes` is SQLite
- [sakerchr/FitNotes](https://github.com/sakerchr/FitNotes) — corroborates the SQLite backup format
- [apkcombo listing](https://apkcombo.com/fitnotes/com.github.jamesgay.fitnotes/) — reproduces the Play description
