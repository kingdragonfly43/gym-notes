# The split is dateless: a queue of training days, not a calendar

A **split** answers one question — what do I train today? — and the obvious way to
build it is a calendar: pick a cycle length, anchor it to a start date, and colour
in the days. We did not do that. A split is a cycle of **training days only**, with
no dates and no rest days, and it advances when a session is completed rather than
when a day passes. There is no calendar surface in the app.

## Considered options

**A dated cycle** was the shape the work started from: N days including rest days,
anchored to a start date, advancing with the calendar. It is what every routine app
does, and it buys a genuine forward calendar you can scroll. It also drifts out of
sync with reality the first time life intervenes — a missed Tuesday leaves the app
insisting you are on Legs for the rest of the cycle — and the fix is a "realign"
affordance that is both fiddly to design and an admission that the model was wrong.

**A hybrid** — rest days advancing with the calendar, training days advancing on
completed sessions — was considered and rejected as the worst of both. It puts two
different advance rules inside one cycle, so a skipped training day stalls forever
while rest days tick past around it.

**The training-day queue** was chosen. Rest is not modelled at all; it is simply
the absence of a session. A Push / Pull / Legs split is three days long, not seven.
Because nothing in the split refers to a date, there is nothing for it to drift
against: the drift problem is dissolved rather than solved. "What do I train today?"
has the same answer whether you last trained yesterday or eight days ago.

**Where the cycle position lives** was a second, related choice. Storing a
`nextDay` pointer on the split makes it mutable state with several writers — it has
to be corrected when a session is deleted, when the split is edited, and it would be
written by a recorder's device during a live shared session. Instead each **session**
records the split day it was trained as, an immutable fact written once, and the
split's next day is read back from those facts. This follows the precedent already
set by personal records, which are derived on read and never stored.

## Consequences

- **A user who trains on a fixed weekly schedule is not modelled.** Mon/Wed/Fri is
  a schedule, not a split, and supporting it would mean the app knowing which days
  you intended to train — and therefore being in a position to nag you about a day
  you missed. This is accepted deliberately.
- **There is no calendar surface.** The split lives in a settings-level screen
  listing the user's splits, and on one tappable line in the new-session flow
  reading "Day 2 · Pull". The past is already served by the session ledger; a
  calendar would be a second view of the same data.
- **Training out of order costs nothing.** The next session offers the derived next
  day, and the user may pick any other day of the active split instead, or type a
  free-text agenda belonging to no day at all. A free-text session references no
  split day and so is invisible to the cycle, leaving the next session still
  offering what was due.
- **Deleting a session rewinds the cycle**, correctly and for free, because the
  position is read from sessions rather than stored alongside them.
- **Several splits cost nothing extra.** Each split's position is read from the
  sessions that used it, so switching away and back resumes where that split left
  off with no bookkeeping.
- **A session's split day reference may dangle.** It is a weak reference used only
  for this derivation; the agenda itself is stored as an immutable copy on the
  session, so a deleted or edited split day never changes what history says. If the
  most recent session's reference no longer resolves, the derivation walks back to
  the most recent one that does, and falls to day 1 if none do.
- **Retrofitting dates later would be expensive**, touching the split, the session
  and the surface together. That is the cost of this decision and the reason it is
  recorded here.
