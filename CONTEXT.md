# Gym Notes

A local-first mobile workout tracker: a deliberately-incomplete remake of FitNotes
plus live shared session recording between friends. This glossary fixes the
vocabulary the product, spec and code all use.

## Training

**Exercise**:
A reusable named movement, such as a lat pulldown. An exercise carries a name,
exactly one category, a measurement type, a default unit, free-text notes and an
optional machine brand. Exercises are referenced by sets, never owned by a session.
Two machines that read the same weight but feel different are two exercises: there
is deliberately no gym or equipment concept. There is no favourite flag either —
the split already answers what to train today.
_Avoid_: Workout, movement, lift

**Category**:
The single grouping an exercise belongs to, such as Back or Legs, carrying a name, a
colour and a user-chosen order. Every exercise belongs to exactly one, never many,
because the picker sits on the hot path and a two-level tree is materially faster to
navigate than a flat list of a hundred-odd exercises.
_Avoid_: Muscle group, tag, folder

**Measurement type**:
What an exercise's weight field means. There are exactly two: *weight and reps*,
where the weight is the load lifted, and *bodyweight and reps*, where it is the
weight added to the body, defaulting to none and permitted to go below zero for band
or machine assistance. The set row is identical under both, so the hot path never
forks. A measurement type cannot change once an exercise has sets — the weight field
would survive the change but its meaning would flip from load lifted to weight
added, silently corrupting every past set and every personal record.
_Avoid_: Exercise type, mode, unit type

**Session**:
One dated training instance, owned by exactly one user. A session is a date holding
ordered supersets — there is no planned-versus-actual concept, and nothing is ever
reconciled against anything. A date may hold more than one session; same-day
sessions are told apart by their start time. A session is live until it ends, either
because the owner finishes it or because it sits idle for an hour; an idle ending is
stamped at the last set's time, so the end time always means when training actually
stopped, while the one-hour reopen window runs from when the session actually closed.
_Avoid_: Workout, training day

**Agenda**:
A short, free-text description of what a session or a split day is about, such as
"Back, Core". It may name several body parts or goals, and it describes rather
than prescribes: nothing is ever reconciled against it. A session's agenda is a
**copy** taken when the session is created, never a live view of a split day, so
editing or replacing a split can never reach backwards and rewrite what a past
session says it was.
_Avoid_: Focus, topic, theme, label

**Split**:
A named, recurring cycle of **training days**, each carrying an agenda. It exists
so a user never has to decide what to train today. A split holds no dates and no
rest days: rest is simply the absence of a session, so a cycle of Push, Pull and
Legs is three days long, not seven. A user may keep several splits; exactly one is
active at a time, or none. A split prescribes no exercises and no sets, which is
what keeps it from being a routine.
_Avoid_: Routine, program, plan, schedule

**Split day**:
One entry in a split: an agenda and its position in the cycle. It carries nothing
else — no date, no exercises, no sets.
_Avoid_: Rotation day, phase, slot

**Active split**:
The one split currently supplying agendas to new sessions. Exactly one split is
active, or none, and the others sit inert in the user's list. Making a different
split active is not a migration: each split's position in its own cycle is read
from the sessions that used it, so switching away and back resumes where that
split left off.
_Avoid_: Current split, selected split, default split

**Superset**:
An ordered line-up of exercises trained in rotation, sitting directly under a
session. A superset is the general case and a lone exercise the degenerate one: a
superset whose line-up holds a single exercise, with one set per round. Modelling it
this way round is what stops supersets being expensive to add later.
_Avoid_: Group, circuit, block

**Line-up**:
The ordered exercises a superset rotates through. A line-up of one is an ordinary
single exercise, which is the common case.
_Avoid_: Members, contents, list

**Round**:
One pass through a superset's line-up, holding one set per exercise in it. Rounds
are explicitly ordered, never inferred from timestamps, because recorders and manual
reordering both make timestamp order wrong.
_Avoid_: Circuit, cycle, pass

**Set**:
One exercise's work within a round, naming the exercise and holding ordered steps.
A straight set has a single step; a drop set has several.
_Avoid_: Entry, rep, row

**Step**:
A weight, a rep count and a completion ratio inside a set, recorded with the unit it
was entered in — the innermost record of what was actually lifted. Steps are their
own level rather than being inferred, so a drop set is never guessed at from
adjacent matching exercises.
_Avoid_: Segment, part, sub-set

**Completion ratio**:
How much of each rep in a step was actually completed, as a fraction of a whole rep.
A full rep is one and a half rep is a half, because only half the range was moved.
It bears on volume only and never on the rep count shown: eight half reps display as
eight reps marked half, never as four. Only the half-rep marking ships in v1;
assisted reps are modelled but not yet surfaced. It is named for the rep rather than
the person — an *effort* ratio would read as a verdict on a lifter who is always
trying their hardest.
_Avoid_: Effort ratio, quality, partial

**Unit**:
The unit a weight is expressed in. It lives in three places with three jobs: a step
stores the unit it was entered in, because a step is an immutable record of what was
lifted; an exercise carries a default that pre-fills new entry; and a global
preference seeds that default when an exercise is created. Changing an exercise's
unit therefore affects only future work and never rewrites history. The global
preference is **pounds**. The increment is global at **5 lb / 2.5 kg**, the smallest
usable plate pair in each system, with no per-exercise override.
_Avoid_: Measure, weight unit

**Personal record**:
The best a user has done on one exercise, derived on read and never stored. For a
weight-and-reps exercise it is the highest volume reached in a single set, counting
each step's weight against its reps and their completion ratio; for a
bodyweight-and-reps exercise it is the highest rep count, always, so that adding
weight to a movement later never silently changes what its record means. Records are
per set and never per session, and they ship as a badge on the set row only.
_Avoid_: PB, best, record

**User data**:
The complete collection of a user's sessions and splits. The unit that gets backed
up, exported, adopted at sign-in, and discarded on collision. Deliberately excludes
the account, friendships and friend requests.
_Avoid_: Log, history, records

`Log` is reserved for its ordinary programming meaning and never names a domain
concept here.

## Identity and sharing

**Account**:
The identity a user's data belongs to. Exists from first launch as an anonymous account,
and becomes a signed-in account when linked to a Google or Apple credential.
_Avoid_: User, profile, login

**Recorder**:
A friend attached to a live session by that session's owner, able to write into it
while holding that session's pen. A recorder is provenance, never ownership — the
session and its sets belong to the owner regardless of who entered them.
_Avoid_: Trainer, coach, editor

**Pen**:
The exclusive right to write into a session. Exactly one person holds a session's
pen at any moment, which is why two people never write to the same session at
once. The owner is the sole grantor: attaching a recorder hands them the pen, and
the owner may take it back or pass it on at any time. Deleting is the one act the
pen does not convey — it stays with the owner.
_Avoid_: Lock, token, writer, clipboard

**Friend**:
A durable, symmetric contact between two signed-in accounts, established by QR
scan and confirmed by acceptance. Friendship is separate from session access:
being a friend does not grant recording rights, it only makes attachment possible.
_Avoid_: Contact, connection, follower

**Friend request**:
A pending, one-directional ask created when one account scans another's QR code.
It never expires, and it becomes a friendship only on acceptance.
_Avoid_: Invite, invitation

**Alias**:
A private, local name one user gives a friend. It replaces that friend's display
name throughout the setter's own app and is never visible to anyone else.
_Avoid_: Nickname, label

**Display name**:
The account-level name a friend sees, seeded from the sign-in provider where
available and always editable in-app.
_Avoid_: Username, handle

**Split grant**:
Permission for one chosen friend to see your active split — its days, which day
you last trained and which comes next, and nothing else. It is one-directional
and per-friend: granting yours does not get you theirs. It is revoked silently,
and removing the friend revokes it. It conveys no access to sessions, to session
history or to dates, which is what keeps it from quietly becoming a general view
onto someone's training record.
_Avoid_: Share, subscription, follow
