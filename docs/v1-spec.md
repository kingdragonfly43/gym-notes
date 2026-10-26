# Gym Notes v1 specification

The buildable specification for Gym Notes v1, assembled from the decisions on the
[Gym Notes v1 spec](https://github.com/kingdragonfly43/gym-notes/issues/2) map. Build
sessions work from this document without reopening product questions.

**How to read it.** This document states *what* v1 does. The *why* lives where each
decision was made: the linked ticket, an ADR under [`docs/adr/`](adr/), or the
glossary in [`CONTEXT.md`](../CONTEXT.md). Where this document and a ticket disagree,
the ticket's latest resolution wins and this document is the bug. Vocabulary follows
`CONTEXT.md` exactly: an **exercise** is never a "workout", and `log` never names a
domain concept.

**Two areas are still open** and marked **⚠ Open** where they appear. Each is a
ticket on the map; build work that touches them waits for it:

- [The recorder's screen during a live shared session](https://github.com/kingdragonfly43/gym-notes/issues/32)
- [Where a never-signed-in user's data lives](https://github.com/kingdragonfly43/gym-notes/issues/31)

---

## 1. What Gym Notes is

A local-first mobile tracker for resistance training: a **deliberately incomplete
remake of FitNotes**, plus **live shared sessions**, in which a friend at the gym
enters your sets on their own phone while you lift, and they show up on yours within
seconds.

**Who it is for.** People who lift with a training partner and want the fastest
possible way to enter a set, with no account, no sign-in wall and no onboarding.

**What makes it different.**

- **Zero-friction first run.** Fully usable with no account, no sign-in wall and no
  onboarding questionnaire. Anything that compromises this has to justify itself
  loudly. The one exception is a legally required store age check, and only in the
  three states whose laws require it (§7.6).
- **A tight set-entry path.** Two taps to a saved set inside a session, three from
  cold open (§4).
- **Symmetric peers, no roles.** Anyone can record for anyone; two people can record
  for each other at once, each owning their own session. There are no trainer or
  trainee roles.

**What it deliberately is not.** It is not a cardio tracker, a routine or program
builder, an analytics tool, a body-measurement tracker or a social network. It has
no feed, no messaging and no gym or equipment concept. The full list is in §13.

**Distribution.** iOS and Android from day one: shared sessions are cross-device by
nature, so a single-platform v1 would fail half its pairings. **Sold in the United
States only, priced in dollars, English only**
([EEA and UK at launch](https://github.com/kingdragonfly43/gym-notes/issues/22)).
Designed for **phones, in portrait**
([Device classes](https://github.com/kingdragonfly43/gym-notes/issues/21)).

---

## 2. Standing rules

These rules cut across every feature. Build and review against them.

| Rule | Source |
|---|---|
| No account, sign-in wall or questionnaire on first run. The store age check in regulated states is the only exception. | Map; [Age signals](https://github.com/kingdragonfly43/gym-notes/issues/23) |
| One writer per session: the **pen**. Concurrent writes to a session are impossible by design. | [ADR-0001](adr/0001-one-writer-per-session.md) |
| **No training data in an ad request, ever.** (Apple 5.1.3: a store rule, not a choice.) | [Privacy disclosures](https://github.com/kingdragonfly43/gym-notes/issues/19), [Privacy policy](https://github.com/kingdragonfly43/gym-notes/issues/27) |
| **The app never infers, scores or labels health status from user data.** Statements about the lift are allowed ("try 190 lb next set"). Statements about the body are not ("you may be injured"). | [Health data](https://github.com/kingdragonfly43/gym-notes/issues/28) |
| **No geofencing, ever, without review.** The app never reads location. | [Health-data laws](https://github.com/kingdragonfly43/gym-notes/issues/25), [Privacy policy](https://github.com/kingdragonfly43/gym-notes/issues/27) |
| The app never sells user data and never uses it to train AI models. | [Privacy policy](https://github.com/kingdragonfly43/gym-notes/issues/27) |
| **No user data in a crash report**: no exercise names, notes, agendas, weights, reps, split names, friend names or account IDs. Error messages never quote text the user typed. | [Crash reporting](https://github.com/kingdragonfly43/gym-notes/issues/26) |
| **Write changed fields only, never a whole-document `set()`.** Without this, expand-only schema evolution silently strips fields. | [ADR-0004](adr/0004-force-update-over-schema-migration.md) |
| Stored shapes **only ever gain optional fields**. Readers tolerate missing fields and fill defaults. A breaking change ships as a forced update. | [ADR-0004](adr/0004-force-update-over-schema-migration.md) |
| **Every `Set` is its own Firestore document** under its session, never an array element. | [ADR-0002](adr/0002-flutter-and-firebase.md) |
| **No Firestore transactions on the offline-capable path.** Use batched writes and atomic field transforms. | [ADR-0002](adr/0002-flutter-and-firebase.md) |
| **Anything stored about an account outside its own data tree must be listed in the account-deletion function.** | [ADR-0005](adr/0005-blaze-for-server-side-account-deletion.md) |
| Personal records and a split's cycle position are **derived on read, never stored**. | [Domain model](https://github.com/kingdragonfly43/gym-notes/issues/9), [ADR-0003](adr/0003-dateless-split.md) |
| Anything touching **ads, billing, camera, or live-session / background behaviour** is exercised on **both platforms in the week it is written**. | [Mac access](https://github.com/kingdragonfly43/gym-notes/issues/13), [ADR-0002](adr/0002-flutter-and-firebase.md) |
| The age signal is installation state: never stored in user data, the cloud, an export file, crash reports or analytics. | [Age signals](https://github.com/kingdragonfly43/gym-notes/issues/23) |

---

## 3. Domain model

The glossary in [`CONTEXT.md`](../CONTEXT.md) is the authority on every term below.
This section gives the shape and the rules a build needs. Source:
[Core logging domain model](https://github.com/kingdragonfly43/gym-notes/issues/9).

### 3.1 Shape

```
User data
├── Global unit preference            (lb; seeds new exercises)
├── Exercise library
│   └── Category                      name, colour, user-chosen order
│       └── Exercise                  name, measurement type, default unit,
│                                     notes, optional machine brand
├── Splits                            named; exactly one active, or none
│   └── Split day                     agenda + position in the cycle
└── Sessions                          dated; owned by exactly one account
    ├── start time, end time (nullable while live)
    ├── agenda                        a copy of the text, never a live view
    ├── split day reference           weak; may dangle
    └── Superset                      ordered; carries its line-up
        └── Round                     ordered; one pass through the line-up
            └── Set                   ordered; names one exercise
                └── Step              ordered; weight + unit, reps, completion ratio
```

- **A superset is the general case.** A lone exercise is a superset whose line-up
  holds one exercise, with one set per round.
- **Steps are their own level**, so a drop set is never inferred from adjacent
  matching rows.
- **Ordering is explicit at every level**, never derived from timestamps: recorders
  and manual reordering both make timestamp order wrong.
- **No planned-versus-actual concept.** Nothing is ever reconciled against anything.
- **User data** excludes the account, friendships, friend requests, split grants and
  all installation state (sign-in nudge suppression, ad usage counter, age signal,
  crash-report switch). This is the unit that is backed up, exported, adopted at
  sign-in and discarded on collision.

### 3.2 Exercise and category

- An exercise carries a **name**, exactly **one category**, a **measurement type**, a
  **default unit**, free-text **notes** and an optional **machine brand**. There is
  no favourite flag and no photo.
- **Two measurement types:**
  - *weight and reps*: the weight is the load lifted.
  - *bodyweight and reps*: the weight is the load **added** to the body. It defaults
    to none and may go below zero for band or machine assistance.

  The set row is identical under both, so set entry never forks.
- **Measurement type cannot change once an exercise has sets.** The app blocks the
  change and offers "create a new exercise" instead.
- Two machines that read the same weight but feel different are **two exercises**.
  There is no gym or equipment concept.
- **A starter library ships**, seeded on first launch. Every row, categories
  included, is as editable and deletable as a user-created one. There are no
  protected system rows. Its content reconstructs a library comparable to FitNotes',
  minus cardio (build work; see §14).
- **A recorder-created exercise is an ordinary exercise** in the owner's library,
  immediately and permanently.
- **Delete is a mark-delete** for sets and exercises, so a deleted exercise never
  orphans the sets that name it.

### 3.3 Units

- A **step stores the unit it was entered in**, so history is never rewritten.
- An **exercise carries a default unit** that pre-fills new entry. Changing it
  affects only future work.
- A **global preference seeds** that default when an exercise is created. It is
  **pounds**.
- The **increment is global at 5 lb / 2.5 kg**, with no per-exercise override.
  Never use 2.5 as a pound increment.

### 3.4 Completion ratio

A step's completion ratio is the fraction of each rep actually completed: 1 for a
full rep, 0.5 for a half rep. It affects **volume only**, never the displayed rep
count: eight half reps display as "8, marked half", never as 4. **Only the half-rep
marking ships in v1.** Assisted reps are modelled but not surfaced.

### 3.5 Personal records

Derived on read, never stored. **One record per exercise, per set:**

- *weight and reps*: the highest **volume** in a single set, counting each step's
  weight × reps × completion ratio.
- *bodyweight and reps*: the highest **rep count**, always.

Records ship **only as a badge on the set row**. There is no estimated 1RM, rep-max
table or session-level record.

### 3.6 Session lifecycle

One lifecycle for solo and shared sessions alike.

- A session is **live** until it ends. The owner either finishes it, or it **sits
  idle for an hour**.
- An idle ending is **stamped at the last set's time**, so the end time always means
  when training actually stopped.
- A session can be **reopened within one hour of when it actually closed**, not of
  its back-dated stamp. Reopening is also how a recorder is attached after the fact.
- **A date may hold more than one session.** Same-day sessions are told apart by
  start time and agenda.

### 3.7 The split

Source: [The split and the training calendar](https://github.com/kingdragonfly43/gym-notes/issues/14),
[ADR-0003](adr/0003-dateless-split.md).

- A split is a **named, recurring cycle of training days only**. It has no dates and
  no rest days; rest is the absence of a session. It prescribes no exercises and no
  sets.
- A user keeps **several splits, with exactly one active, or none**. **No split by
  default**, and nothing is blocked by its absence.
- **The cycle position is derived:** each session records the split day it used,
  and the next day is read back from those records. Deleting a session rewinds the
  cycle, and switching splits resumes each where it left off. If the latest
  session's reference dangles, the derivation walks back to the latest one that
  resolves, and falls to day 1 if none does.
- **Off-cycle training is free.** The new-session flow offers the derived next day.
  The user may pick any other day of the active split, or type a free-text agenda
  that belongs to no day and is invisible to the cycle.
- **Surfaces:**
  - a settings-level screen listing splits: pick the active one, rename, reorder
    days, edit agendas;
  - **one tappable line in the new-session flow**, e.g. "Day 2 · Pull".

  **There is no calendar surface for the split.** History's month view (§4.4) shows
  past sessions only, never the split or a future day. The empty split screen is a
  single card that explains splits, with one button.

---

## 4. Set entry: the hot path

Source: [Set-entry hot path](https://github.com/kingdragonfly43/gym-notes/issues/10).
The prototypes are on the
[`prototype/set-entry-hot-path`](https://github.com/kingdragonfly43/gym-notes/tree/prototype/set-entry-hot-path/prototypes/set-entry-hot-path)
branch (`screens.html` is the settled round). Design target: **iPhone SE 3 at
375×667**, the smallest and weakest device in the user base.

**Inside a session, 2 taps to a saved set, no typing**: tap the exercise, save.
**From cold open with a live session it is 3**: Continue on the Today tab, tap the
exercise, save. Cold open lands on Today rather than in the session (§4.4).

### 4.1 Screens

The shape is **screen per exercise**, as in FitNotes. Inside a session:

1. **Session index**: a chronological ledger of every superset and its sets, in one
   top-to-bottom scroll, in the order performed.
2. **Exercise picker**: a two-level tree. The categories form a **single-column
   list** of full-width rows, each with a colour accent, an exercise count and a
   chevron. Search is available. There are no recents.
3. **Entry screen**: **large ± steppers** at 5 lb and 1 rep, with no typing, sized to
   be hard to fat-finger mid-set. It also shows:
   - a **rest timer counting up** from the last saved set, as a strip in the header;
   - the previous-performance line;
   - this session's sets so far;
   - a derived PR badge.

   There is no target, no goal and no alarm. The logged-sets list has no "Today"
   header.
4. **New exercise form**: one full single-column form with every field visible. It
   is reached from the picker and drops the user straight into entry for the new
   exercise, without leaving the session.

### 4.2 Behaviour

- **Prefill** comes from the previous set of the same exercise: this session first,
  else the last session that had it, else an empty-bar guess. It always takes the
  **top step** of that set, never the last, or a drop set would start every
  following set at the drop weight.
- **Saving a set auto-advances** to the next exercise in the superset's line-up.
  For a lone exercise this has no visible effect.
- **Drop set and half rep are two buttons beside save.** Drop set appends a step;
  half rep marks the pending step. A set row renders as `185 lb × 8 > 135 lb × 6`,
  with a `HALF` pill where marked.
- **No ad ever sits beside the hot path's controls.** The only ad surface is the
  interstitial fired on save (§10).
- Exact spacing, copy and animation are build-time judgement calls.

### 4.3 Starting a session

The new-session flow takes the derived split line ("Day 2 · Pull", tappable) and an
editable agenda copied from that day. With no active split, the agenda starts empty.
The agenda is stored on the session as a copy. The flow lives on the Today tab
(§4.4).

### 4.4 The app shell

Source: [The app shell: top-level navigation and history](https://github.com/kingdragonfly43/gym-notes/issues/30).
The prototype is on the
[`prototype/app-shell`](https://github.com/kingdragonfly43/gym-notes/tree/prototype/app-shell/prototypes/app-shell)
branch. The settled shell is that prototype's shell A with shell B's Today screen and
shell B's per-exercise history.

- **A persistent bottom tab bar with four tabs: Today, History, Exercises,
  Friends.** It is hidden inside a session's flow (session index, picker, entry
  screen, new exercise form). Settings is a gear at the top of Today.
- **Cold open always lands on Today**, live session or not.
- **Today shows today only**: no day arrows and no calendar, because past days
  belong to History.
  - Each of today's sessions appears as its ledger. A date may hold more than one
    session.
  - A **live session is read-only on Today**, with two buttons: **Continue**, which
    opens the session index, and **Finish**, which ends the session.
  - With no live session, the **Start session** block sits here (§4.3).
  - First run shows only the Start session block, with no onboarding.
- **History** lists past sessions newest first under month headers. It has a
  **List | Month** switch. Month is a calendar grid of past training days; tapping a
  day opens its session, or lists them if there are several. A session opens as its
  read-only ledger, with **Reopen** while the one-hour window (§3.6) is open. This
  month view of the past is **not a split calendar**: it never shows the split or a
  future day (ADR-0003).
- **Exercises** is the library as the picker's category tree. An exercise opens on
  **History | Graph** tabs.
- **Per-exercise history** also sits on the entry screen as **Track | History |
  Graph** tabs, where Track is the entry screen of §4.1.
  - *History*: one row per past session that had the exercise, with its date, agenda
    and sets, and PR badges.
  - *Graph*: **one progress line per exercise** with one point per past session. For
    *weight and reps* the point is the top-step weight of that session's
    highest-volume set; for *bodyweight and reps* it is that session's highest rep
    count. This single line is the only graph in v1. Anything beyond it is analytics,
    which is post-v1 (§13).
- **Friends** holds the friend list, a Requests row with a count badge (also shown on
  the tab), **My QR code** and **Add friend (scan)**. Signed out, each of these swaps
  in place to the sign-in prompt (§6.5).
- **Settings** holds Units, Splits (§3.7), Export / Import (§5), Remove Ads and
  Restore Purchases (§10.2), Account (sign in, sign out, delete account) and the
  Privacy Policy.

---

## 5. Export and import

Source: [The export and import format](https://github.com/kingdragonfly43/gym-notes/issues/15),
plus the cautions in
[The privacy policy](https://github.com/kingdragonfly43/gym-notes/issues/27).

- **Format:** one plaintext UTF-8 **JSON** file carrying the whole **user data**:
  sessions, splits, the exercise library and the global unit preference. It carries
  **nothing about people**: no friends, no names, no account IDs, and no record of
  who entered a set. Installation state is excluded, so an import cannot reset the
  ad threshold.
- **Version:** the file is stamped with the app's `SCHEMA_VERSION` code constant,
  the single version line. An **older file is read forward** under expand-only
  rules. A **newer file is refused outright** ("this export was made by a newer
  version of Gym Notes"), never partially imported.
- **Import replaces, never merges.** A non-empty destination forces an explicit
  choice and **offers an export of what is about to be discarded first**. When the
  user is signed in, the confirm **names the cloud out loud**: *"this replaces your
  training record on this phone and everywhere it is backed up"*.
- **All-or-nothing validation.** Validate the whole file, and destroy the old record
  only once the new one is known to be good.
  - A dangling split-day reference is expected: preserved, never "repaired".
  - A dangling exercise reference is a hard failure.
  - Unknown fields within a known version are tolerated and ignored, because
    hand-editing is a legitimate rescue path.
- **Delivery:** the file is written to app-private storage and handed over through
  the **OS share sheet**, never written to Downloads. Filename:
  `gym-notes-YYYY-MM-DD-HHMM.json`. The export flow carries one plain line saying
  the file is **not encrypted and contains the whole training record**. There is no
  password and no encryption.
- **Naming:** "Export" / "Import" and "export file", never "back up" or "restore".
  "Backup" is reserved for cloud backup.
- **Entry points, all using one control:** Settings; before the sign-in collision
  discard; before account deletion; and the **fire exit** on any launch gate (§11.4).
  A destructive confirm is **never gated on the export succeeding**, because the
  share sheet cannot tell a save from a dismissal.
- **No export reminder.** Settings shows "Last export: never" (or a date) as a plain
  fact, with no badge, colour or action attached.
- A data-access request by email is answered with the export file **plus the list of
  recipients**: Google/Firebase, AdMob and Crashlytics (§12).

---

## 6. Identity, sign-in and installations

Sources: [Friend graph, identity and sign-in flow](https://github.com/kingdragonfly43/gym-notes/issues/6),
as amended by
[Enforcing one installation per account](https://github.com/kingdragonfly43/gym-notes/issues/20),
[Account deletion under the daily delete quota](https://github.com/kingdragonfly43/gym-notes/issues/24),
[Store age signals](https://github.com/kingdragonfly43/gym-notes/issues/23) and
[How the app treats possible health data](https://github.com/kingdragonfly43/gym-notes/issues/28).

### 6.1 Accounts

- **An anonymous account exists from first launch.** Signing in **links** a Google
  or Apple credential to it with `linkWithCredential()`, which preserves the UID.
- **Sign-in is required only** to open Add friend, show your own QR code, view
  friend requests, and attach or be attached as a recorder. Every other surface works
  signed out.
- **Signed-in users get free cloud backup** of all their user data, restorable on a
  new phone.

> **⚠ Open: where a never-signed-in user's data lives.** The privacy policy promises
> that a never-signed-in user's data never leaves the phone, but an anonymous
> Firebase account syncing through Firestore would upload it. Creating an anonymous
> account also needs a connection, which a first launch in a basement gym may not
> have. See [Where a never-signed-in user's data lives](https://github.com/kingdragonfly43/gym-notes/issues/31).

### 6.2 The sign-in sheet

Every sign-in route ends on one sheet. Above the Google and Apple buttons it carries:

> *Signing in stores your training record in the cloud, including your notes and
> agendas, which may describe health conditions. Friends you attach to a session or
> share your split with can see it. [Privacy]*

The sentence sits beside an **unticked "I agree" box**, and the sign-in buttons stay
disabled until it is ticked.

- The box appears on **every** sign-in, because an earlier consent cannot be known
  until after sign-in.
- The account stores **when consent was given and which wording version was shown**.
  That record is not user data, and it is deleted with the account.
- **[Privacy]** opens the Privacy Policy.
- Declining is simply not signing in. Withdrawing consent is *delete my account and
  keep this phone's copy* (§6.6).
- For a user treated as under 13, the sheet states in one line why sign-in is
  unavailable and points to the store (§7).

### 6.3 Signing in with existing local data

- **Empty cloud account:** the local user data is uploaded silently.
- **Non-empty cloud account:** the user **picks one side and the other is
  discarded**. This prompt shows **every time, even when signing back into the same
  account**.
  - Each side shows its session count, date range and last session date, e.g.
    "247 sessions, Mar 2024 – today" against "12 sessions, Jan – Feb 2024".
  - **An export of the side being discarded is offered first.**
  - The discard includes the exercise library, since it is part of user data.
- A **display name** is seeded from the provider where available and is always
  editable. Apple may return no name, so the app prompts for one if the provider
  gave none. An optional avatar may be set. Email is never shown to friends.

### 6.4 Sign-out

The local user data stays on the phone, untouched. Friends and requests disappear
from view. The app warns once: *"Your data stays on this phone. New sessions won't
be backed up until you sign in again."*

### 6.5 One installation per account

- An **installation** is one install of the app, identified by a random ID generated
  on first launch. A reinstall is a new installation.
- **A signed-in account is on at most one installation at a time, and the most
  recent sign-in wins**, never the most recent connection. Signing in records the
  current installation on the account. The new installation says nothing.
- **The replaced installation signs itself out** the next time it reads that record
  and finds it is no longer current.
  - It **keeps its local data**.
  - It **never navigates away**. Signed-in-only screens swap to the sign-in prompt
    in place.
  - A banner stays across the top until dismissed: *"You signed in on another phone,
    so this one is now signed out. Anything you log here stays on this phone only."*
- **Queued offline writes from the replaced installation race the sign-out, by
  design.** They may land once, and nothing is lost either way. **This rule keeps a
  user's own installations consistent. It is not a security boundary**, and nobody
  should "fix" it with Security Rules.
- The rule applies only to signed-in accounts. Recording for a friend is never a
  second-installation case.

### 6.6 Account deletion

Source: [ADR-0005](adr/0005-blaze-for-server-side-account-deletion.md).

- **Choices:**
  - Signed in: **"Delete my account and keep this phone's copy"** or **"Delete my
    account and erase this phone too"**.
  - Never signed in: **"Erase all data on this phone"**, which is not account
    deletion.
- **Flow:** offer an export → **sign in again with the same provider** → final
  "Delete" confirmation. Deletion needs a connection.
- **Refused during a live session** (holding a friend's pen, or a friend holding
  yours): *"Finish or end your live session first."*
- **Before the user confirms, the screen states:**
  - the Remove Ads purchase survives through Restore Purchases;
  - sets they entered as a recorder stay in friends' sessions;
  - copies on other phones are not erased;
  - the sign-in record lingers in Firebase's own backups for up to 180 days.
- **Sign in with Apple is revoked on the phone**, using the authorization code from
  the sign-in-again step, **before** the server function is called.
- **The server function** "delete my account":
  1. checks for a fresh sign-in;
  2. deletes the user's own data tree recursively;
  3. deletes everything about them stored with other accounts: friendships on both
     sides, friend requests sent and received, split grants given and received,
     and the entitlement record. The account's installation record and consent
     record go too, whether inside its tree or listed explicitly;
  4. deletes the Firebase Auth user;
  5. only then replies.

  It is **idempotent**, and the app confirms completion on the spot (about a minute).
- **A daily cleanup** repeats the sweep for the previous day's deletions, since a
  replaced installation's ID token can recreate documents for up to an hour.
- **Afterwards:**
  - "Keep this phone's copy" leaves the installation **exactly as a sign-out does**,
    under a new anonymous account.
  - "Erase this phone too" leaves it like a fresh install, except that installation
    state survives.
- **A replaced installation holding a deleted account** signs itself out, keeps its
  copy, and shows: *"This account was deleted on another phone. Anything on this
  phone stays on this phone only."* The signal is the account's installation record
  being gone, not the Auth "user not found" error.
- **Off-app deletion** (required by Play): a published privacy email address with
  **reply-to-confirm** from the account's own address. The developer runs the same
  deletion code from the Mac through the Admin SDK. This path cannot revoke Sign in
  with Apple, so the confirmation reply tells Apple users how to stop using it
  themselves (Settings → their name → Sign in with Apple → Gym Notes → Stop Using).

### 6.7 The sign-in nudge

- Fires **on session completion**. It shows only if the user is not signed in and no
  suppression period is running.
- The surface is a normal sign-in screen: a message that leads with the benefit
  (free backup, nothing else changes), the social sign-in buttons, and a dismiss
  cross.
- **Dismissal always resolves to an explicit choice.** Pressing the cross offers
  **"Remind me in 7 days"** or **"Don't remind me again"**, and the nudge closes only
  once one is picked.
- It is visibly distinct from the solicited "Sign in to add friends" gate.
- Its copy names the risk of losing the record plainly. Suppression is installation
  state.

---

## 7. Age signals and minors

Source: [Store age signals, minors and ad treatment](https://github.com/kingdragonfly43/gym-notes/issues/23).
Age changes exactly one thing: **whether a user may sign in**.

- **When the app asks:** only where state law requires (Texas, Utah, Louisiana),
  once at first launch and re-checked on later launches.
  - **iOS:** Declared Age Range, only when Apple reports that the account needs age
    assurance (`isEligibleForAgeFeatures` / `requiredRegulatoryFeatures`), behind
    iOS 26 availability guards.
  - **Android:** Play Age Signals, which shows no prompt in the US law states.
- **Under 13, as reported by the store:**
  - The full app works on their own phone: sessions, splits, export and import, and
    Remove Ads (through Ask to Buy).
  - They **cannot sign in**, so they have no cloud copy, no friends and no recording
    either way.
  - Ads carry `CHILD` treatment.
- **Unknown age inside a regulated state** (Play `VERIFICATION_REQUIRED`, or Apple
  unable to answer) is treated as **under 13 until the store gives an age**. It is
  never a lockout. Outside regulated states, unknown is treated like everyone else.
- **A signed-in installation that learns its user is under 13** signs itself out and
  keeps its data, by the same path as a replaced installation.
- **13–17:** the full app, including friends and recording.
- Store parental consent is **not** treated as COPPA verifiable parental consent.
- **Consent revocation:**
  - iOS: Apple prevents the launch, and the data stays on the phone.
  - Android: the launch check shows the **export fire exit**.
  - No server endpoint exists for it. The cloud copy goes only through ordinary
    account deletion.
- **Significant changes** (Texas's list: new data types collected, a new monetisation
  feature, or an age-rating change) need renewed parental consent. Until a parent
  approves, a regulated-state minor gets the **export fire exit only**. v1 itself
  ships no significant change. Classifying each release is a release-checklist item
  (§12.4).

---

## 8. Friends

Source: [Friend graph, identity and sign-in flow](https://github.com/kingdragonfly43/gym-notes/issues/6).

- **QR scan is the only way to add a friend in v1.** There is no share link and no
  typed code.
- **The QR code is permanent and static**, encoding the account ID. A leaked code
  only lets someone send a request that can be declined.
- **A scan is a search, not a friendship.** The scanner sees a confirm screen with the
  target's display name and avatar, then sends a **friend request**. The friendship
  exists once the recipient accepts.
- **Requests never expire** and are created on the server, so the recipient being
  offline at scan time does not matter.
  - Re-scanning someone you already have a pending request to is a no-op, with a
    "Request already sent" toast.
  - **Scanning someone whose request to you is pending auto-accepts it.**
- **Aliases** are private, local names that replace a friend's display name in your
  own app. The real display name stays visible on the friend's detail screen.
- **Removing a friend is symmetric and silent.** Both sides lose the friendship, and
  no notification is sent. Removal also revokes any split grant between them. Sets
  they entered stay, unconditionally. **There is no separate block.**
- **Split grant:**
  - **per friend and one-directional**: granting yours does not get you theirs;
  - **revoked silently**, and dies with the friendship;
  - the friend sees **the active split's days, which day you last trained and which
    comes next**, and **no dates**, ever;
  - it conveys no access to sessions or history.
- The QR camera flow is **device-only for testing**, because the iOS Simulator has
  no camera.

---

## 9. Live shared sessions

Sources: [Live shared session semantics](https://github.com/kingdragonfly43/gym-notes/issues/7),
[ADR-0001](adr/0001-one-writer-per-session.md).

### 9.1 The pen

- **Exactly one person holds a session's pen** at any moment, and only the holder
  can write.
- The pen conveys **every write** to the session and to the owner's exercise
  library: appending sets, editing them, reordering, creating an exercise.
- **It does not convey delete.** Deleting is **owner-only and also requires the pen**.
- **The owner is the sole grantor.** Attaching a recorder hands them the pen by
  default. The owner may take it back or pass it to another attached recorder at any
  time. A recorder can neither hand it back nor pass it on.
- **An owner cannot write while a recorder holds the pen.** They take it back first,
  with one tap. This is intended, not a bug.
- **Mutual recording:** A holds B's pen and B holds A's, so neither writes into their
  own session. **The recorder's screen is the primary surface** during a shared
  session.

### 9.2 Attachment

- The owner picks a friend from the friend list. Attachment is a **notification, not
  a permission request**.
- Attachment is to a **live session only**. Reopening a finished session (§3.6) is
  the way to attach someone after the fact.
- **Multiple recorders** may be attached, but there is still one pen.
- **Both apps must be at the same `SCHEMA_VERSION`**, in either direction. A mismatch
  is refused at attach time, never mid-session.
- **Live sync:** a recorder's sets appear on the owner's phone **within seconds**.

### 9.3 What a recorder sees

Everything useful to someone coaching you: the current session, all past sessions,
statistics such as personal records, and the exercise library. **Access dies with the
attachment**, when the session ends or the recorder is detached.

### 9.4 Failure and lifecycle

- **An offline pen holder's writes fail visibly and are never queued locally.** The
  holder sees a failed-to-record error telling them to ask the owner to take the pen
  back. The owner sees that the pen has gone dark and can take it back. **There is no
  automatic transfer.**
- **Detaching a recorder who holds the pen** returns it to the owner. Their sets stay.
- **Ending a session** detaches every recorder and discards anything not yet saved as
  a set. The recorder is told the session ended, rather than having the screen vanish.
- **Nothing records which sets a recorder entered.** A set is the owner's record.
- Recorders and split grants see notes and agendas. That is the owner's choice, and
  the sign-in consent sentence is its only disclosure: there are no per-attach
  warnings.

> **⚠ Open: the recorder's screen.** The questions the shared-session ticket handed to
> the set-entry prototype were never prototyped:
> - how a recorder switches between their own session and a friend's;
> - how the app makes it unmistakable whose session you are entering into;
> - how holding, gaining and losing the pen appears;
> - how offline write failures look;
> - which of the owner's history surfaces during entry;
> - which phone shows the on-save interstitial in a shared session.
>
> See [The recorder's screen during a live shared session](https://github.com/kingdragonfly43/gym-notes/issues/32).

---

## 10. Monetization

Source: [Monetization surface](https://github.com/kingdragonfly43/gym-notes/issues/11),
as amended by [Age signals](https://github.com/kingdragonfly43/gym-notes/issues/23) and
[Health data](https://github.com/kingdragonfly43/gym-notes/issues/28). Monetization is for
**principle, not income**: revenue is not a design criterion.

### 10.1 Ads

- **One format: a skippable interstitial at the start of rest, fired on saving a
  set.** There is no banner anywhere, and no video, rewarded or app-open ads.
- **Frequency:** AdMob's own cap of no more than one interstitial per two user
  actions, roughly three a session.
- **A failed load means no ad.** No layout space is reserved.
- **Network:** AdMob via `google_mobile_ads`.
- **Content:** max rating PG, with the weight-loss and supplement categories blocked
  (configured in the console).
- **Treatment on every request:**
  - AdMob's age-treatment setting `TEEN` (non-personalised) for everyone, and `CHILD`
    for store-reported or presumed under-13s. Use the new age-treatment setting, not
    the deprecated child-directed tags;
  - **restricted data processing** (`rdp=1`).
- **There is no ATT prompt and no UMP consent tool.** The app does no tracking and
  reads no IDFA.
- **Ads start only after a usage threshold**, counted on **sessions the user owns**,
  never on sessions recorded for a friend. The threshold is a deferred parameter;
  the working figure is 10. The counter is installation state.
- **One announcement card** at that moment says ads are starting, why, and offers the
  one-time purchase, with a dismiss. **It is the only ask the app ever makes.** There
  is no recurring offer.

### 10.2 Remove Ads

- **One non-consumable product, "Remove Ads".** There is no second tier, and **money
  gates no feature in v1**.
- Price: a deferred parameter, working figure **$4.99**.
- **Entitlement is local first, and attaches to the account when signed in**, so it
  follows the account across installations. Buying works fully signed out. Sign-in is
  never required to buy.
- **It does not cross platforms**, and that is accepted.
- **A visible Restore Purchases control** ships.
- **Shared sessions:** if **any participant** in a live session holds the
  entitlement, **nobody in that session sees ads**. Otherwise everyone does.
- Product IDs: Apple `com.kingdragonfly43.gymnotes.removeads`, created. The Play
  product is not yet created (§14.2).

---

## 11. Platform, stack and backend

### 11.1 Stack

**Flutter** for the client; **Firebase** for the backend: Cloud Firestore in
**`us-central1`** (single region, a one-way door), Firebase Auth, Cloud Functions,
Crashlytics. The reasoning and rejected alternatives are in
[ADR-0002](adr/0002-flutter-and-firebase.md). Key packages: FlutterFire,
`google_mobile_ads`, `in_app_purchase`, `sign_in_with_apple`, and `mobile_scanner`
for QR.

**Plan: Blaze**, with a **billing budget alert attached in the same act**. Never use
a spend-cap that pauses Cloud Functions, because it would switch off account
deletion ([ADR-0005](adr/0005-blaze-for-server-side-account-deletion.md)). **Firestore
keeps no backups**: point-in-time recovery is off and no backups are scheduled.
Turning either on must revisit deletion and the retention disclosure.

### 11.2 Firestore constraints

- One document per `Set` (~1 write/sec/document ceiling).
- No transactions on the offline path.
- Changed-field writes only.
- Free quotas are **daily** (50k reads / 20k writes / 20k deletes), and Blaze bills
  beyond them. A heavy five-year user is about 26,000 set documents.

### 11.3 OS floors and devices

Source: [Minimum supported OS versions](https://github.com/kingdragonfly43/gym-notes/issues/17),
[Device classes](https://github.com/kingdragonfly43/gym-notes/issues/21).

- **iOS 18 and Android API 28.** These are absolute numbers, moved only when a
  dependency, in practice Flutter, will not compile below them. There is no rolling
  policy and no in-app OS check, because store filtering makes one unreachable.
- `targetSdk` follows Play's requirement (API 36 now; API 37 expected during 2027).
- **Phones, portrait only.**
  - iOS ships an iPhone-only target. iPads may install it and must work in the
    phone-shaped window.
  - Android has no device exclusions.
  - **Wide Android screens show a phone-width column, centred**, enforced by one rule
    at the app root. It must work in both orientations.
  - Not listed on Mac or Vision Pro: both boxes are unticked in App Store Connect.
- **Testing:**
  - Daily work uses the iOS Simulator and an Android emulator on the Mac mini.
  - The **iPhone SE 3 tracks current iOS** and is used for camera/QR, StoreKit
    sandbox and real-hardware performance.
  - The OS floor is tested in the Simulator.
  - Each release gets one pass on an iPad simulator and one on a foldable emulator.

### 11.4 Force update and the fire exit

Source: [ADR-0004](adr/0004-force-update-over-schema-migration.md).

- **There is no stored schema version.** `SCHEMA_VERSION` is a code constant, used
  for the attach check and the export stamp.
- **A Firestore force-update document** holds:
  - the **minimum supported app version**;
  - the **current released build's OS floor per platform**.

  It is read at launch and cached. A device offline with a stale cached value
  proceeds normally.
- **Below the minimum, the app does not run.** The launch gate offers updating and
  **one other action: export**. There is no browsing, no logging and no statistics.
  - If the device's OS is below the released build's floor, the gate says plainly
    that this phone can no longer run Gym Notes, and offers the export.
  - Otherwise: *"Update to continue. If the update isn't offered to you, you can
    export your data here."*
- **Downgrade is unsupported** and presents as the update gate, never as corruption.
- **Release rule:** do not raise the minimum until **both stores are serving the new
  build**.
- The same export-only screen serves revoked parental consent (Android) and an
  unapproved significant change (§7).

### 11.5 Crash reporting

Source: [Crash reporting and analytics in v1](https://github.com/kingdragonfly43/gym-notes/issues/26).

- **Firebase Crashlytics ships; product analytics does not.**
- It runs from first launch for everyone, with no consent prompt and the same
  treatment at every age.
- A **"Send crash reports" switch** in Settings is on by default and stored on the
  installation.
- **Never linked to an account**: no `setUserIdentifier` and no UID.
- **It reports failures only**: crashes, plus handled failures the user is shown (a
  pen holder's offline write, a rejected import, a failed deletion call, a sync
  error). It never reports successes.
- **Allowed context:** screen name, app version, whether the user is signed in, and
  their live-session role (owner or recorder).

---

## 12. Privacy and store surface

Sources: [Privacy policy and store data disclosures](https://github.com/kingdragonfly43/gym-notes/issues/19)
(research on branch `research/privacy-disclosures`),
[US consumer-health-data laws](https://github.com/kingdragonfly43/gym-notes/issues/25)
(research on branch `research/us-health-data-laws`),
[How the app treats possible health data](https://github.com/kingdragonfly43/gym-notes/issues/28),
[The privacy policy: contents, host and public identity](https://github.com/kingdragonfly43/gym-notes/issues/27).

### 12.1 Stance

- **Every signed-in user's cloud copy is treated as possible consumer health data**
  under Washington's My Health My Data Act, and as Connecticut sensitive data where a
  note reveals a condition.
- Never-signed-in, local-only data is outside that stance (but see §6.1 ⚠).
- **Free-text fields stay as they are**, with no hint and no filtering.
- **The app stores no body mass anywhere.**

### 12.2 Public pages

All four pages are on a **domain the developer owns**, hosted on **GitHub Pages**
from this repo:

1. Home;
2. **Privacy Policy**;
3. **Consumer Health Data Privacy Policy** (a separate page with nothing else on it);
4. **Account deletion** (Play's off-app path).

**Where they are linked:**

- Each store's privacy-policy field points at the Privacy Policy, whose opening
  lines link to the health-data policy.
- In the app, **Settings → About** has two separate links with exactly those names.
- The sign-in consent sentence's **[Privacy]** link opens the Privacy Policy.

The **policy outline** (ten sections) is in
[the privacy-policy ticket's resolution](https://github.com/kingdragonfly43/gym-notes/issues/27). Writing
the text is build work. **Rights requests** (see, delete, withdraw consent) are
answered within **45 days**, with an appeal route ending at the **Washington Attorney
General's** complaint channel.

### 12.3 Public identity

- The developer is named **as an individual**, exactly as on the store listings.
- **One dedicated privacy address on the domain** handles policy contact, deletion by
  email, data requests and appeals. It is registered with Apple's private email
  relay. The personal Gmail appears nowhere.
- **Apple trader status: "not a trader"** (no EU distribution).
- **A mailbox street address** goes in Play's payments profile **before Remove Ads
  goes on sale on Play**.
- The policy carries no postal address.

### 12.4 Store declarations

- **Category:** Health & Fitness on both stores.
  - Apple's regulated-medical-device question: "No".
  - Play's Health apps declaration: Activity and Fitness.
- **Play target audience: 13+**.
- **Apple age rating** answered honestly; the expected result is **4+**. QR-only
  friends are not social media.
- **Data labels:**
  - **AdMob drives most of the declarations.**
  - The training record is **Fitness**.
  - Name, email, user ID and purchase history are declared on both stores. On
    Apple, the friend graph counts as **Contacts**.
  - Crashlytics adds crash and diagnostic data that is not linked to the user.
  - Verify against AdMob's privacy manifest whether "tracking" can be left
    undeclared.
- **Privacy manifests** are required for the Firebase SDKs, GoogleSignIn and Flutter.

### 12.5 Release checklist items

- **Significant policy change** means any change to what is collected (including any
  new third-party SDK), who receives it, why, how long it is kept, how rights are
  exercised, or the contact address. The steps:
  1. **notify each store before** the change takes effect (Texas SB 2420 §121.053);
  2. publish the new version with a new "last updated" date;
  3. show a one-time in-app notice on the next launch.

  If the change reaches data already collected, the notice also offers *delete
  account, keep this phone* **before** the change applies.
- **Significant change under the age-signal laws** (§7): new data types, a new
  monetisation feature, or an age-rating change. Adding analytics, body data, a new
  purchase or AI suggestions after launch all qualify.
- **Force-update minimum** is raised only when both stores serve the new build.

---

## 13. Out of scope for v1

From the map's Out-of-scope section. None of these graduate; revisiting one means a
new effort.

- **Cardio logging**, along with any measurement type beyond the two.
- **FitNotes backup import.**
- **Trainer/client roles** and any trainer-tier subscription.
- **Banner, video, rewarded and app-open ads.**
- **Feature gating and paid tiers.** Money only removes ads.
- **Machine photos** on an exercise.
- **A gym or equipment concept**, and training in more than one gym.
- **A human-readable or spreadsheet export.** If one ships later, it needs a different
  name than "export".
- **Tablet layouts, landscape, and Mac and Vision Pro listings.**
- **Selling outside the United States, and languages other than English.**
- **Body measurement tracking** in any form.
- **Product analytics.**
- **Shipping v1 to the stores**, which is beyond this spec's destination.

**Post-v1 backlog**, kept so the v1 model does not paint them into a corner:
routines/programs, analytics (beyond the one progress line per exercise in §4.4),
the **standing view grant** (a friend's durable read-only window onto your
sessions), and a **subscription wedge**, whose leading
candidate is AI-assisted training suggestions. Any suggestion feature is bounded by
the no-health-conclusions rule and the no-AI-training promise.

---

## 14. Shipping order

Solo-developer work lands in sequence. Each step leaves a runnable app that is more
useful than the last, and the riskiest cross-platform areas get both platforms in the
week they are written (§2).

| # | Step | Contents | Waits on |
|---|---|---|---|
| 1 | **Foundations** | Flutter project; Firebase on Blaze with budget alert; Firestore offline persistence; installation ID; Crashlytics with its switch; the local-data arrangement for never-signed-in users | ⚠ [Where a never-signed-in user's data lives](https://github.com/kingdragonfly43/gym-notes/issues/31) |
| 2 | **Solo set entry** | Categories, exercises, starter library; sessions → supersets → rounds → sets → steps; session index, picker, entry screen, new exercise form; prefill, steppers, auto-advance, drop set, half rep, rest timer, PR badge; session lifecycle, reopen, mark-delete; units | — |
| 3 | **App shell and history** | Tab bar; Today with Continue/Finish and the Start session block; History list and month view; session detail and reopen; Exercises library; per-exercise History and Graph tabs; Friends tab shell; Settings | 2 |
| 4 | **Splits** | Splits list, active split, derived next day, "Day 2 · Pull" line, agenda copy | — |
| 5 | **Export, import and the launch gate** | Export file, share sheet, import replace with validation; force-update document, launch gate and export fire exit | — |
| 6 | **Sign-in and cloud backup** | Sign-in sheet with consent; linking; choose-one prompt; sign-out; one installation per account; sign-in nudge; age signals and the under-13 rule; **account deletion** (server function, daily cleanup, Apple revocation) | 1 |
| 7 | **Friends** | Display name/avatar, QR code and scan, requests, aliases, removal, split grant | 6 |
| 8 | **Live shared sessions** | Attach, pen, schema-version check, live sync, recorder access window, offline failure, end/reopen, mutual recording, the recorder's screen | 7, ⚠ [The recorder's screen](https://github.com/kingdragonfly43/gym-notes/issues/32) |
| 9 | **Monetization** | Remove Ads with restore and account attachment; interstitial on save with threshold, cap, `TEEN`/`CHILD`, `rdp=1`; announcement card; the shared-session entitlement rule | 8 |
| 10 | **Launch readiness** | Domain, four public pages, privacy address and Apple relay; Play mailbox address; Play Remove Ads product; store forms, ratings, listings (US only, phones, not Mac or Vision Pro) | 9 |

**Why this order.**

- **Set entry first:** it is the product's central claim, and everything else
  decorates it.
- **Export before sign-in:** export is the escape hatch offered before both
  destructive sign-in-era actions, so it must exist before them. The launch gate
  needs it as its fire exit.
- **Account deletion ships with sign-in, not after it:** it is a store submission
  gate for any app that creates accounts.
- **Live shared sessions need friends, which need sign-in.**
- **Monetization comes last:** its shared-session rule spans every participant's
  entitlement, and its threshold counts owned sessions.

### 14.1 Build-time content and parameters

- **Starter library content**: reconstruct a FitNotes-comparable list, minus cardio,
  with categories, colours and measurement types.
- **Deferred parameters:** Remove Ads price (working $4.99) and the ad threshold
  (working 10 owned sessions).

### 14.2 Provisioned, and still to provision

Recorded in [Store and developer account provisioning](https://github.com/kingdragonfly43/gym-notes/issues/18).
Values were captured by the gitignored `.wizards/gym-notes-provisioning.sh`, and
config files live in a gitignored folder in the repo.

| Item | Value |
|---|---|
| Bundle ID / package name | `com.kingdragonfly43.gymnotes` (both platforms) |
| Firebase project | `gym-notes-4ee33`, Firestore `us-central1` |
| Apple | Individual enrolment; app "Gym Notes: Set Tracker", SKU `gymnotes-v1`, App Store ID `6816548868` |
| Play | Personal account with payments profile; app "Gym Notes" |
| AdMob app IDs | iOS `ca-app-pub-1790351629336093~8916527448`; Android `ca-app-pub-1790351629336093~6805369421` |
| AdMob interstitial units | iOS `ca-app-pub-1790351629336093/4770408036`; Android `ca-app-pub-1790351629336093/5029428338` |

**Still to do:**

- **Move the Firebase project to Blaze** and attach the budget alert (step 1).
- Move `GoogleService-Info.plist` and `google-services.json` from `~/Downloads` to
  the gitignored config folder (step 1).
- **Create the Play "Remove Ads" one-time product** once a build is uploaded to any
  track: suggested ID `remove_ads`, non-consumable, $4.99 (step 9 or 10).
- Buy the domain; set up the privacy address and register it with Apple's relay; set
  up the mailbox address (step 10).

---

## 15. Open questions carried into the build

- **The two ⚠ Open tickets** above. Each one's resolution amends this document.
- **Watch list:** New York S9269 (health privacy bill, pending); Washington AG
  guidance on fitness data; the *In re Amazon Ads SDK* litigation; Play's API 37
  target date; the Utah and Louisiana effective-date changes.
- **Outside the spec:** whether to form an LLC and carry insurance, which is a
  question for a lawyer.
