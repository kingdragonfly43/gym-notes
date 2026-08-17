# Gym Notes

A local-first mobile workout tracker: a deliberately-incomplete remake of FitNotes
plus live shared session recording between friends. This glossary fixes the
vocabulary the product, spec and code all use.

## Training

**Exercise**:
A reusable named movement, such as a lat pulldown. Exercises are referenced by
sets, never owned by a session.
_Avoid_: Workout, movement, lift

**Session**:
One dated training instance, owned by exactly one user. A session is a date
holding ordered exercises — there is no planned-versus-actual concept. A date may
hold more than one session; same-day sessions are told apart by their start time.
_Avoid_: Workout, training day

**Agenda**:
A short, free-text description of what a session or a split day is about, such as
"Back, Core". It may name several body parts or goals, and it describes rather
than prescribes: nothing is ever reconciled against it.
_Avoid_: Focus, topic, theme, label

**Split**:
A recurring cycle of a fixed number of days, each day carrying an agenda. It
exists so a user never has to decide what to train today. A session's agenda
defaults to the agenda of its day in the split and stays editable; a session is
never required to match it.
_Avoid_: Routine, program, plan

**Set**:
A single logged row inside a session, recording a weight and a rep count against
one exercise.
_Avoid_: Entry, rep, row

**User data**:
The complete collection of a user's sessions. The unit that gets backed up,
exported, adopted at sign-in, and discarded on collision. Deliberately excludes
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
