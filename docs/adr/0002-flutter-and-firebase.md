# Flutter and Firebase for the client and backend

Gym Notes needs a single cross-platform client shipping to iOS and Android from
day one, plus a backend that keeps the app fully usable offline, syncs a
recorder's sets onto the session owner's phone within seconds, and preserves a
user's training log when an anonymous first run later signs in. The stack and
the backend constrain each other on exactly these points, so they were decided
together rather than in sequence.

We chose **Flutter** for the client and **Firebase**, specifically **Cloud
Firestore** in the **`us-central1`** region, for the backend.

## Considered options

**React Native / Expo** was the leading candidate for most of this effort's
research, on the strength of EAS's cloud iOS builds and installable device
loop — the only stack that treated "no Mac" as a supported configuration. That
constraint is gone: [Will this project have access to a
Mac?](https://github.com/kingdragonfly43/gym-notes/issues/13) confirmed a Mac
mini M4 and a physical iPhone SE are in hand, and Apple Developer enrollment
is accepted. With a real local iOS loop available either way, Flutter's
first-party coverage of every feature this app needs — Firebase (FlutterFire),
`in_app_purchase`, Google-maintained `google_mobile_ads` with UMP and ATT
documented by Google itself, and `mobile_scanner` for QR — becomes the
deciding edge, on top of an official Flutter MCP server and agent skills. RN's
`react-native-google-mobile-ads` and `expo-iap` are solid but
community-maintained, and Expo's own docs warn that "AI models and LLMs
frequently provide outdated information about Expo" because of the
ecosystem's churn.

**Kotlin Multiplatform + Compose Multiplatform** was ruled out. It has no
Google AdMob SDK at all — not even community — no first-party Firebase SDK,
and the worst iOS build story of the three even with a Mac available. For an
app whose entire free-tier revenue depends on a correctly rendered, legally
compliant banner ad, hand-writing and maintaining that integration twice,
natively, was not worth KMP's edge on local persistence.

**Supabase** (optionally paired with PowerSync) was the leading backend
alternative. Its relational model fits a workout log more naturally than
documents, and self-hosting is a genuinely open exit path. It loses on the
app's single hardest requirement: Supabase ships no first-party offline
persistence, so the local-first, fully-offline-capable requirement would need
a second vendor (PowerSync, +$49/month on top of Supabase's own $25/month
step) bolted on. Supabase's cost curve is also a hard step to $25/month the
moment any free-tier limit is crossed or a project sits idle a week, against
Firebase's linear creep from $0 — a material difference for an app whose
entire v1 revenue is a banner ad and a one-time unlock.

**Peer-to-peer sync** (Multipeer Connectivity, Nearby Connections, Wi-Fi
Direct/Aware, BLE) was assessed seriously and found structurally additive,
not alternative: free cloud backup is already committed, so a cloud sync path
must exist regardless, making P2P a second path to build and support on top
of the one being built anyway. Its fatal cases are also common rather than
exotic — client/AP isolation is a default on the gym guest wifi this app will
actually be used on, and iOS backgrounding kills the "within seconds" promise
unless the phone stays unlocked and on-screen. Parked as a possible v2 latency
optimization, never a v1 candidate.

**Realtime Database** was considered over Firestore within Firebase. It's
faster (~10ms vs ~30ms claimed) and has a cleaner `onDisconnect()` primitive,
but that speed is imperceptible at this app's logging cadence, and it only
offers 3 locations worldwide — Belgium is the only European one, against
Firestore's roughly 13 single European regions. Firestore's real cost,
its ~1 write/sec-per-document ceiling, is avoided for free by a domain model
that already treats each `Set` as its own document rather than an array
field.

## Consequences

- **The Mac mini is now this project's iOS build and debug machine**, and the
  standing rule from #13 carries forward: anything touching ads, billing,
  camera, or live-session behaviour is exercised on both platforms in the week
  it is written.
- **Every `Set` must be its own Firestore document under its session**, never
  a field in an array, to stay under the ~1 write/sec-per-document ceiling
  during a fast-logging burst (e.g. a superset).
- **Transactions cannot be used on the offline-capable hot path** — Firestore
  transactions fail outright while offline. Batched writes and atomic field
  transforms (`increment`, `arrayUnion`, `serverTimestamp`) are the offline-safe
  primitives instead.
- **Firestore's free quotas are confirmed daily**, not monthly: 50,000
  reads, 20,000 writes, 20,000 deletes per day, resetting around midnight
  Pacific ([firebase.google.com/docs/firestore/quotas](https://firebase.google.com/docs/firestore/quotas),
  [firebase.google.com/pricing](https://firebase.google.com/pricing)). Size
  any future capacity planning against the daily figure, not the larger
  monthly misreading that no longer appears in current docs.
- **The Firestore project region is `us-central1` (Iowa), single-region**,
  chosen for US-based users at roughly half the per-operation cost of a
  multi-region location. This is a one-way door — Firestore's location cannot
  be changed after project creation — so a pivot to EU users later means a new
  project and a data migration, not a settings change.
- **Cloud Functions and Cloud Storage both require the Blaze (pay-as-you-go)
  plan just to provision**, even if usage stays inside the free allotment.
  Not needed for v1, but the moment either is introduced, a billing budget
  alert must be attached at the same time — an unbounded card on Blaze is the
  standard footgun.
- The anonymous-to-signed-in upgrade uses `linkWithCredential()`, which
  preserves the Firebase UID and therefore the user's whole log — this is the
  mechanism the "local-first with optional sign-in, never lose two years of
  lifts" decision on the map depends on.
