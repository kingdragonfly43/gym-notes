# Backend and live-sync platform comparison

Research for [#5](https://github.com/kingdragonfly43/gym-notes/issues/5). This gathers
facts and lays out trade-offs. **The decision belongs to [#8](https://github.com/kingdragonfly43/gym-notes/issues/8)**
and is deliberately not made here.

All pricing and limits observed **2026-08-15**. Every non-obvious claim carries a source
link. Claims that could not be pinned to a primary source are marked **[unverified]**.

---

## 1. What the app actually asks of a backend

From the map ([#2](https://github.com/kingdragonfly43/gym-notes/issues/2)), the committed
decisions that constrain this choice:

| Requirement | Consequence for the backend |
| --- | --- |
| Fully usable with **no account**, no sign-in wall | Needs anonymous/no-identity local operation, and a way to later attach an identity **without losing data** |
| Sign-in **only** for the friends feature, via Google or Apple | Needs both providers; Apple's rules make Google-alone illegal on iOS (see §2) |
| **Live sync, not merge-at-end** — recorder's entries visible on owner's phone within seconds | Needs server-push (WebSocket/long-lived stream), not polling |
| **Cloud backup is free** for signed-in users | Rules out any P2P-only design; there must be a cloud store, and its storage cost is borne by the developer |
| Fully offline-capable local log | Needs a local database and a write queue that survives app restart |
| **Near-zero cost at small scale** | Free tier must cover a few hundred users; the step to the first paid tier matters more than the per-unit rate |
| Ads + third-party sign-in + body-related data | Store privacy disclosures and a DPA are mandatory regardless of choice (see §7) |
| Symmetric peers, two people recording for each other simultaneously | Two writers into one session document — conflict semantics are load-bearing |

Note the shape of the load: this is a **tiny fan-out** problem. One session has at most a
handful of subscribers. None of the scaling ceilings that dominate backend marketing
material (thousands of concurrent subscribers, millions of messages) are anywhere near
binding here. The differentiators are the free tier, the offline story, and SDK coverage.

---

## 2. The auth constraint is fixed before the backend choice

**Apple App Store Review Guideline 4.8** makes this non-negotiable on iOS: an app that uses
"a third-party or social login service (such as ... Google Sign-In ...) to set up or
authenticate the user's primary account **must also offer as an equivalent option another
login service**" that limits collection to name and email, lets the user keep their email
private, and does not collect in-app interactions for advertising without consent
([Apple App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)).
In practice that means Sign in with Apple. So "Google and Apple" is not a preference — it
is the minimum legal set once Google is offered.

Two more guideline facts that shape the design, both already aligned with the map's
decisions:

- **5.1.1(v)**: "If your app doesn't include significant account-based features, let people
  use it without a login." The zero-friction first run is not just a differentiator, it is
  the compliant path.
- **5.1.1(v)**: "If your app supports account creation, you must also offer account
  deletion within the app." Google Play matches this — account deletion must be reachable
  in-app *and* via an external web URL declared in Play Console, and must delete all
  associated data, not merely freeze the account
  ([Play User Data policy](https://support.google.com/googleplay/android-developer/answer/10144311)).

Sign in with Apple works on Android and web via a Services ID and the web/OAuth flow, but
requires an active **Apple Developer Program membership ($99/yr)** to configure
([Apple docs](https://developer.apple.com/documentation/sign_in_with_apple)). That fee is
already an input to the project's economics regardless of backend.

**The anonymous-to-identified upgrade is the single most important auth question here**,
because the app must work with no account and then attach one later without losing two
years of lifts. Every candidate below is judged on whether that upgrade preserves the
existing user's data.

---

## 3. Firebase

### Auth
- Google Sign-In and Sign in with Apple are both first-party, documented per platform
  ([iOS Google](https://firebase.google.com/docs/auth/ios/google-signin),
  [iOS Apple](https://firebase.google.com/docs/auth/ios/apple),
  [Android Apple](https://firebase.google.com/docs/auth/android/apple) — Android uses a web
  flow since there is no native Apple SDK there).
- **Anonymous → permanent upgrade preserves the UID and all associated data** via
  `linkWithCredential()`
  ([anonymous auth docs](https://firebase.google.com/docs/auth/flutter/anonymous-auth)).
  This is precisely the pattern the map requires, and it is the cleanest of any candidate.
  - Caveat: since 2023-09-15, email-enumeration protection is on by default for new
    projects and **blocks linking an anonymous account to an email/password credential**.
    Federated (Google/Apple) linking is unaffected
    ([Firebase blog](https://firebase.blog/posts/2023/07/best-practices-for-anonymous-authentication/)).
    Since this app only needs Google/Apple, this does not bite — but do not add
    email/password later without re-checking.
- "Firebase Authentication with Identity Platform" is a free feature-unlock, not a paid
  migration, but it changes the free-tier shape: Spark becomes a **daily** active user cap
  (3,000 DAU for Tier 1 providers), Blaze bills monthly with a 50,000 MAU no-cost tier
  ([auth limits](https://firebase.google.com/docs/auth/limits)). Irrelevant at this scale.

### Realtime
Two products, and the choice between them matters more than the choice of Firebase itself.

| | Cloud Firestore | Realtime Database |
| --- | --- | --- |
| Stated typical response | ≤30 ms | ≤10 ms, "ideal for frequent state-syncing" |
| Write ceiling | **~1 write/sec sustained to a single document** | 1,000 writes/sec per database instance; 64 MB/min sustained |
| Free-tier connections | n/a (per-op billing) | **100 simultaneous connections on Spark**; 200,000 on Blaze |
| Locations | ~13 European regions + `eur3` multi-region | **Only 3 worldwide**: Iowa, Belgium, Singapore |

Sources: [Firestore vs RTDB](https://firebase.google.com/docs/firestore/rtdb-vs-firestore),
[RTDB limits](https://firebase.google.com/docs/database/usage/limits),
[Firestore locations](https://firebase.google.com/docs/firestore/locations),
[RTDB locations](https://firebase.google.com/docs/database/locations).
The latency figures are from a product comparison page, **not an SLA** — no formal latency
guarantee was found for either product. **[unverified: any documented cap on concurrent
snapshot listeners per client]**

The ~1 write/sec-per-document Firestore limit is the one number that actually touches this
design: if a whole session is one document and two people are hammering sets in during a
superset, that is a real ceiling. It is avoided by modelling each `Set` as its own document
under the session rather than as an array field — which is the natural model anyway.

Firestore snapshot listeners apply **latency compensation**: a local write fires the
listener immediately, before server acknowledgement, and `snapshot.metadata.fromCache`
tells you which state you are looking at
([listen docs](https://firebase.google.com/docs/firestore/query-data/listen)). For a live
shared session this is exactly the right primitive — the recorder sees their own entry
instantly and the owner sees it on the next round trip.

### Offline and conflict
This is Firebase's strongest suit and the reason it is hard to beat for this app.

- **Firestore persistence is enabled by default on iOS and Android** (disabled on web),
  default cache 100 MB, configurable to unlimited
  ([offline docs](https://firebase.google.com/docs/firestore/manage-data/enable-offline)).
  No extra library, no extra vendor.
- On reconnect the client synchronizes local changes, with **last-write-wins** for
  conflicting writes to the same document (same source).
- **Transactions fail when the client is offline** — stated explicitly in the docs
  ([transactions](https://firebase.google.com/docs/firestore/manage-data/transactions)).
  **Batched writes do queue and execute offline.** This is a genuine design constraint: an
  offline-capable app cannot use transactions on its hot path, and must reach for batched
  writes and atomic field transforms (`increment`, `arrayUnion`, `serverTimestamp`) instead.
  **[unverified: the exact offline-queue semantics of field transforms issued while
  offline — the docs cover transform semantics and transaction failure but not this
  specific interaction]**
- RTDB queues writes offline, and with persistence enabled the queue survives app restart;
  `onDisconnect()` schedules server-side writes that fire on ungraceful disconnect, which is
  a neat fit for "recorder left the session"
  ([RTDB offline](https://firebase.google.com/docs/database/android/offline-capabilities)).

Last-write-wins at document granularity is acceptable here **because of the domain model**:
a `Set` is owned by whoever logged it, and two people do not normally edit the same set row
simultaneously. Conflicts are rare by construction, not by cleverness.

### Cost
Spark (free) is still available to new projects
([pricing](https://firebase.google.com/pricing)). Free quotas:

- **Firestore**: 50,000 reads/day, 20,000 writes/day, 20,000 deletes/day, 1 GiB stored,
  10 GiB/month egress ([usage](https://firebase.google.com/docs/firestore/usage)).
- **RTDB**: 1 GB stored, 10 GB/month downloaded, **100 simultaneous connections**.
- **Auth**: free for standard providers well past any plausible scale here.

Beyond free, Firestore bills per operation. Rates depend on location — the official billing
example uses the North America multi-region `nam5`:
**$0.06/100K reads, $0.18/100K writes, $0.02/100K deletes, $0.18/GiB-month storage,
$0.12/GiB egress** ([billing example](https://firebase.google.com/docs/firestore/billing-example)).
Single-region locations are roughly half that (~$0.03/100K reads, ~$0.09/100K writes per
[Cloud pricing](https://cloud.google.com/firestore/pricing)) — **worth confirming for the
specific region chosen, as multi-region vs regional is a ~2x difference on every operation**.
RTDB on Blaze is $5/GB stored/month and $1/GB downloaded.

> **Discrepancy to be aware of**: Firebase's Firestore usage page states the free quotas
> as **per day**, while the Blaze billing example describes the same figures as **monthly**.
> These cannot both be true. Assume the conservative (monthly) reading when sizing, and
> confirm against live billing before relying on headroom. **[unverified]**

**Cost curve shape: smooth and linear, with no step.** This is Firebase's decisive
commercial advantage for this project. There is no $25/month cliff — a hobby-scale app
simply stays at $0 and creeps up per-operation as it grows. Two caveats:

1. **Cloud Functions require the Blaze plan** to deploy at all (a billing instrument must
   be attached, even if usage stays inside the free allotment). Blaze includes 2M
   invocations/month free.
2. **Cloud Storage for Firebase now requires Blaze** for any project provisioning a default
   bucket for the first time, as of 2026-02-03
   ([storage changes FAQ](https://firebase.google.com/docs/storage/faqs-storage-changes-announced-sept-2024)).
   Irrelevant for v1 (no progress photos), relevant the moment images appear.

Attaching a card to Blaze with no budget cap is the classic Firebase footgun. A billing
budget alert is mandatory operational hygiene, not optional.

### Data residency
**Firestore's location is fixed at creation and cannot be changed** — "once you provision a
database instance, you cannot change its location setting"
([locations](https://firebase.google.com/docs/firestore/locations)). This is a
one-way door that must be decided in #8, not deferred. Firestore offers `eur3`
(Belgium + Netherlands) plus ~13 single European regions. **RTDB offers only three
locations worldwide, of which Belgium is the only European one** — so choosing RTDB is
also choosing Belgium if EU residency matters.

Google acts as processor under the
[Firebase DPA](https://firebase.google.com/terms/data-processing-terms) (last modified
2024-08-21), incorporating Standard Contractual Clauses; ISO 27001 and SOC 1/2/3 are
maintained. Sub-processors are published with 30-day advance notice and a 90-day objection
window ([sub-processors](https://firebase.google.com/terms/subprocessors)).

### SDKs
| Stack | Status |
| --- | --- |
| **Flutter** | **First-party** — FlutterFire, hosted on firebase.google.com, official `flutterfire_cli` ([setup](https://firebase.google.com/docs/flutter/setup)) |
| **React Native / Expo** | Two options, both with caveats. `react-native-firebase` wraps the native SDKs but **cannot run in Expo Go** — it needs a development build. The Firebase **JS SDK** runs in Expo Go but "does not support all services for mobile," explicitly excluding Analytics, Dynamic Links and Crashlytics, and needs explicit AsyncStorage wiring or auth state does not survive restart ([Expo's Firebase guide](https://docs.expo.dev/guides/using-firebase/)) |
| **Kotlin Multiplatform** | **No first-party SDK exists.** Community only: [GitLive firebase-kotlin-sdk](https://github.com/GitLiveApp/firebase-kotlin-sdk), actively maintained (release 2026-04-29), explicitly "not an official Google Firebase product" |

---

## 4. Supabase

### Auth
- Google and Apple both supported, and on native the documented best practice is the
  **native ID-token flow** (`signInWithIdToken()`) rather than an OAuth browser redirect —
  which is the better UX
  ([Google](https://supabase.com/docs/guides/auth/social-login/auth-google),
  [Apple](https://supabase.com/docs/guides/auth/social-login/auth-apple)).
- Two sharp edges worth knowing before committing:
  - Apple returns the user's **full name only on first authorization** — capture it then or
    it is gone.
  - The Apple OAuth path (needed on Android/web) requires a `.p8` signing key that
    **Apple requires regenerating every 6 months** — a recurring maintenance chore for a
    solo developer, and a silent-breakage risk if missed.
- **Anonymous sign-ins** exist and can be upgraded via `linkIdentity()` with Google/Apple
  ([anonymous auth](https://supabase.com/docs/guides/auth/auth-anonymous)), but must be
  **manually enabled** — not on by default. Rate-limited to 30 requests/hour per IP by
  default, with CAPTCHA recommended against account farming.
  **[unverified: whether anonymous users count toward the MAU quota. The MAU billing docs
  define MAU generically with no anonymous carve-out. Assume they count.]** This matters:
  a design that creates an anonymous Supabase user for every first-run install converts
  "installs" into "MAUs", which is a billing dimension Firebase does not have.

### Realtime
Three mechanisms ([overview](https://supabase.com/docs/guides/realtime)):

- **Postgres Changes** — subscribe to INSERT/UPDATE/DELETE. Simplest, but **authorizes
  every event against each subscriber**, so throughput scales with subscriber count, not
  write rate, and processing is single-threaded to preserve order. Supabase's own guidance:
  use Broadcast instead beyond ~3,000 concurrent subscribers on the same changes
  ([postgres-changes](https://supabase.com/docs/guides/realtime/postgres-changes)).
- **Broadcast** — pub/sub, fans out once regardless of subscriber count. "Broadcast from
  Database" writes to `realtime.messages`; messages are retained ~72 hours to 4 days
  ([broadcast](https://supabase.com/docs/guides/realtime/broadcast)).
- **Presence** — for "who is in this session right now". Explicitly not for high-frequency
  updates ([presence](https://supabase.com/docs/guides/realtime/presence)).

Measured latency: on a Micro instance with 500 clients on Postgres Changes, **p95 ≈ 228 ms**
with RLS on. **[unverified: comparable p95 figures for Broadcast — only qualitative
"low-latency" claims found.]** At two-phones-per-session scale, Postgres Changes is
comfortably sufficient and simpler; Presence is a natural fit for showing the recorder is
attached.

### Offline and conflict
**Supabase ships no first-party offline persistence.** This is the central trade-off
against Firebase, and it is not a small one — the app's hard requirement is full offline
operation, so this gap must be filled by a second component:

| Option | What it gives | Cost |
| --- | --- | --- |
| [PowerSync](https://docs.powersync.com/integrations/supabase/guide) | Streams a filtered subset into local SQLite; reads/writes queue offline; **customizable conflict resolution**; Flutter, RN, **Kotlin**, Swift, Web | A second vendor with its own free tier and its own $49/mo step |
| [ElectricSQL](https://electric-sql.com/docs/integrations/supabase) | Read-path sync only — states outright that "offline is out of scope ... it does not concern itself with client-side persistence" | You build local persistence yourself |
| [WatermelonDB](https://supabase.com/blog/react-native-offline-first-watermelon-db) | Community pattern via Postgres RPC + WatermelonDB's pull/push protocol | You own the sync logic |
| DIY | Local SQLite + your own outbox queue | You own everything |

Conflict behaviour: Postgres is authoritative, plain writes are last-write-wins at row or
column level, and **there is no first-party CRDT or merge logic**. Whatever strategy you
want, you design it.

### Cost
[Pricing](https://supabase.com/pricing), observed 2026-08-15:

- **Free**: 500 MB database, 50,000 MAU, 5 GB egress, 200 concurrent realtime connections,
  2M realtime messages/month, 1 GB file storage. **Limit of 2 active projects.**
- **Free projects are paused after 1 week of inactivity.** For a personal app with a slow
  start this is a genuine operational hazard, not a footnote.
- **Pro: $25/month** — 8 GB database, 100,000 MAU, 250 GB egress, 500 realtime connections,
  5M messages. Overages: DB $0.125/GB, MAU $0.00325/user, egress $0.09/GB, realtime
  connections $10/1,000, messages $2.50/million.

**Cost curve shape: a hard step, not a gradient.** $0 → $25/month the instant you need a
third project, want to escape the weekly auto-pause, or exceed any free quota. There is no
intermediate tier. Against the map's "near-zero cost at small scale" requirement, this is
the material difference from Firebase: Firebase creeps, Supabase steps. $300/year is not a
lot in absolute terms, but for an app whose entire revenue model is a banner ad and a
one-time unlock, it is plausibly more than the app earns in year one.

### Data residency
Region is pinned at creation from a wide European set — Ireland, London, Paris, Frankfurt,
Zurich, Stockholm ([regions](https://supabase.com/docs/guides/platform/regions)). The docs
themselves caution that "region selection is a data-location control, not proof of
regulatory compliance." **Region cannot be changed after creation** — the documented
migration path is to create a new project and migrate
([troubleshooting](https://supabase.com/docs/troubleshooting/change-project-region-eWJo5Z)).
Same one-way door as Firebase.

SOC 2 Type 2, ISO 27001, HIPAA with a BAA; a [DPA](https://supabase.com/legal/dpa) is
available, with sub-processor changes carrying 30-day notice
([security](https://supabase.com/security)). **[unverified: the current contents of the
sub-processor list page itself.]** Self-hosting is the documented escape hatch for stronger
residency guarantees, at the cost of owning backups, hardening and DR.

### SDKs
| Stack | Status |
| --- | --- |
| **Flutter** | **First-party** — [`supabase_flutter`](https://github.com/supabase/supabase-flutter) under the `supabase` org ([quickstart](https://supabase.com/docs/guides/getting-started/quickstarts/flutter)) |
| **React Native / Expo** | `supabase-js` is first-party but needs documented plumbing: `react-native-url-polyfill`, a storage adapter (AsyncStorage, or `expo-sqlite` localStorage on Expo), `detectSessionInUrl: false`, and an `AppState` listener to pause token refresh when backgrounded ([RN quickstart](https://supabase.com/docs/guides/auth/quickstarts/react-native)) |
| **Kotlin Multiplatform** | [`supabase-kt`](https://github.com/supabase-community/supabase-kt) — **community**, under `supabase-community` not `supabase`. Notably, Supabase does host a full [Kotlin API reference](https://supabase.com/docs/reference/kotlin/installing) on its own docs site, which is a stronger endorsement than Firebase gives GitLive, but it is still not an in-house SDK |

---

## 5. Other managed backends

Assessed and **ruled out quickly**, with the reason:

- **CloudKit** — Apple-only, no Android access. Fatal against "iOS and Android from day one".
- **Convex** — excellent realtime DX and reactive queries, but **no first-party offline
  sync**, confirmed on its own [sync page](https://www.convex.dev/sync) ("Convex doesn't
  currently provide a full offline sync mechanism"). Its anonymous auth provider is flagged
  in Convex's own docs as "an advanced feature ... not recommended to newcomers" — exactly
  the guest-first pattern this app leans on. Free tier 1M function calls and 0.5 GB storage;
  Pro **$25/developer/month**. Disqualified on offline.
- **AWS Amplify / AppSync + DataStore** — Amplify Gen 1, which DataStore belongs to, is in
  maintenance mode with **EOL 2027-05-01**; Amplify Flutter v1 was already deprecated
  2025-04-30, and Gen 2 does not carry DataStore's offline conflict model forward
  ([Amplify docs](https://docs.amplify.aws/gen1/flutter/prev/build-a-backend/more-features/datastore/)).
  Building a new offline-sync app on a deprecating primitive is not defensible.
- **Ably / PubNub** — transport only, no database, no auth, no backup. Would need pairing
  with a datastore, doubling the vendor count. (Data point: Ably free is 6M messages/month
  and 200 concurrent connections; PubNub free is 200 MAU, and its Starter tier is
  **$98/month**, which is disqualifying on its own.)
- **Turso / libSQL** — good edge-SQLite story, but sync uses **last-push-wins at row level**,
  and there is no built-in auth or live subscription channel
  ([concurrent writes](https://docs.turso.tech/tursodb/concurrent-writes)). Would need auth
  and push bolted on.
- **Jazz** — a genuine local-first CRDT database with good Expo support and a self-hostable
  sync server. **[unverified: Jazz Cloud pricing — the site describes "$9–$79/month" and
  "flexible pricing for scale-to-zero units" with no concrete free-tier table.]** Interesting,
  too immature to bet a solo project's data layer on.

Worth a genuine look:

### Appwrite Cloud
The most complete single-vendor alternative. Google and Apple among 30+ OAuth2 providers,
plus **native anonymous sessions convertible to permanent accounts** by attaching an OAuth2
session to the same account ([OAuth2](https://appwrite.io/docs/products/auth/oauth2)).
Realtime over a single persistent WebSocket. Free tier is generous — 75,000 MAU, 2 GB
storage, 5 GB bandwidth, 250 concurrent realtime connections, 2M messages/month — with
**Pro at $25/month** ([pricing](https://appwrite.io/pricing)). Frankfurt is among the
available regions ([regions](https://appwrite.io/docs/products/network/regions)), and
self-hosting on a single VPS is supported.

**The disqualifier is the same as Supabase's, but without Supabase's mitigations**: no
documented offline persistence or conflict-resolution model — Appwrite is server-authoritative
REST + realtime, not a local-first sync engine. And there is **no official KMP SDK**
(only community wrappers; [issue #9446](https://github.com/appwrite/appwrite/issues/9446)
remains open). Free tier also pauses after 1 week idle.

### InstantDB
The closest thing to "Firebase, but relational and genuinely local-first". Native Google
OAuth via `signInWithIdToken`, native Sign in with Apple, and a documented Guest Auth mode
([auth docs](https://www.instantdb.com/docs/auth)). Local replica with all operations
executing locally when offline. Free tier is 1 GB database with **no pause on inactivity**
and commercial use allowed; Pro is **$30/month**
([pricing](https://www.instantdb.com/pricing)).

Concerns: **Flutter support is community-only** (`instantdb_flutter`; official-SDK feature
requests #19/#171/#308 remain open), there is no KMP SDK, and
**[unverified: any region-selection or EU-hosting option — none found in primary sources,
suggesting single-region US hosting]**, which is a live GDPR question given the data
classes involved. **[unverified: the exact conflict-resolution algorithm — characterised as
CRDT-with-server-authority in secondary sources only, not in InstantDB's own docs.]**
Also: the whole company is a young startup, and this app's committed promise is *free
backup of two years of lifts*. Vendor longevity is a real criterion here, not FUD.

### PowerSync + a managed Postgres
Not a backend but a **sync layer**, and it deserves attention because it has the best
documented offline story of anything surveyed: local SQLite, offline write queue, and
explicit **last-write-wins per field with a documented custom conflict-resolution hook**
([custom conflict resolution](https://docs.powersync.com/handling-writes/custom-conflict-resolution)).
It is also the **only candidate with an official Kotlin Multiplatform SDK**, alongside
Flutter, RN, Web, Swift and .NET.

Free tier: 2 GB synced/month, 500 MB hosted, 50 peak concurrent connections, deactivated
after 1 week idle. **Pro from $49/month** ([pricing](https://www.powersync.com/pricing)).
No auth of its own — bring any JWT provider.

The catch is vendor count: PowerSync + Postgres host + auth provider = three vendors and
three free tiers to keep alive, for a solo developer. Its natural home in this decision is
**as a rescue for Supabase's offline gap** (§4), not as a standalone answer. Note that
choosing it stacks two step-functions: $25 (Supabase Pro) + $49 (PowerSync Pro) = $74/month.

### PocketBase
Single Go binary, embedded SQLite, native Google and Apple OAuth2, realtime via SSE
([auth docs](https://pocketbase.io/docs/authentication/)). Official Dart/Flutter and JS
SDKs; no first-party RN or KMP. Cost curve is **flat** — a VPS at cents per day, no
usage metering, which is genuinely attractive against the near-zero-cost requirement.

The cost is ops: you run, patch, back up and monitor a server, with no vendor SLA, and you
personally own the durability of the "free cloud backup" promise. For a solo developer whose
scarce resource is attention, this trades a small money cost for a recurring attention cost.

---

## 6. Serverless peer-to-peer over the local network

This deserves a real assessment. Two workout buddies genuinely are standing in the same room
on the same wifi, and the latency of a direct link beats any round trip to Belgium.

### What the transports actually are

| Transport | Reach | Notes |
| --- | --- | --- |
| **Multipeer Connectivity** | iOS/macOS/tvOS **only** | Uses infrastructure wifi, peer-to-peer wifi and Bluetooth ([docs](https://developer.apple.com/documentation/multipeerconnectivity)). Needs `NSLocalNetworkUsageDescription` and `NSBonjourServices` in Info.plist |
| **Nearby Connections** | Android + iOS, **asymmetrically** | Android gets Bluetooth Classic, BLE and Wi-Fi Direct (no AP needed); iOS reliably reaches only Wi-Fi LAN, BLE still being built out. **There is no offline (no-AP) iOS↔Android path** ([google/nearby #2447](https://github.com/google/nearby/discussions/2447)). Strategies: `P2P_CLUSTER`/`P2P_STAR`/`P2P_POINT_TO_POINT` ([reference](https://developers.google.com/android/reference/com/google/android/gms/nearby/connection/Strategy)) |
| **Wi-Fi Direct / Wi-Fi Aware** | Android 8+ ([Aware](https://developer.android.com/develop/connectivity/wifi/wifi-aware), [Direct](https://developer.android.com/develop/connectivity/wifi/wifip2p)) | No AP required — survives client isolation |
| **Wi-Fi Aware on iOS** | **iOS 26 only** | Genuinely new: framework `WiFiAware`, entitlement `com.apple.developer.wifi-aware` ([docs](https://developer.apple.com/documentation/WiFiAware)). Excludes every device not on iOS 26. **[unverified: real-world adoption share as of Aug 2026]** |
| **BLE** | Both | Realistic throughput 220–800 Kbps on 4.2, ~1.1 Mbps on 5.0 ([Punch Through](https://punchthrough.com/ble-throughput-part-4/)) — far more than enough for "set logged" events. Android 12+ needs `BLUETOOTH_SCAN`/`CONNECT`/`ADVERTISE` ([docs](https://developer.android.com/develop/connectivity/bluetooth/bt-permissions)); iOS needs `NSBluetoothAlwaysUsageDescription` |
| **mDNS/DNS-SD + sockets** | Both | [RFC 6762](https://datatracker.ietf.org/doc/html/rfc6762) / [RFC 6763](https://www.rfc-editor.org/info/rfc6763/) — genuinely cross-platform, low-level |
| **WebRTC data channels** | Both | **Still needs a signalling channel** to exchange SDP, even for a purely local connection. WebRTC specifies no signalling transport. A QR code could carry it — the app already has QR scanning for friending — but this is real work |

### Failure modes, and which are fatal

| Failure mode | Verdict |
| --- | --- |
| **Captive portal** (gym/hotel guest wifi login page) | **Not fatal.** Local L2 traffic is not gated by portal auth, so MPC/BLE/mDNS/Wi-Fi Direct keep working. Only designs needing a STUN/TURN server are blocked. Ironically, P2P *wins* here where a cloud backend loses |
| **Client/AP isolation on guest wifi** | **Fatal to every wifi-based transport.** Layer-2 isolation blocks client-to-client traffic including mDNS on UDP 5353 ([Meraki docs](https://documentation.meraki.com/Wireless/Operate_and_Maintain/How_Tos/Firewall_and_Traffic_Shaping/Wireless_Client_Isolation)). Survivable only via Bluetooth or Wi-Fi Direct/Aware, which bypass the AP entirely. **This is the single most important fact in this section**: it is a default configuration on exactly the guest networks gyms deploy |
| **Devices on different networks** (one cellular, one wifi) | **Fatal to every local transport** except Bluetooth. No local design bridges two networks |
| **iOS Local Network permission denied** | **Fatal until re-granted in Settings.** There is no official API to check the status without triggering the prompt; even Google's Cast SDK docs tell developers to "degrade gracefully" because there is no clean in-app detect-and-recover ([Cast docs](https://developers.google.com/cast/docs/ios_sender/permissions_and_discovery)) |
| **iOS backgrounding / screen lock** | **Fatal to the "within seconds" promise** unless the phone stays awake and foregrounded. MPC sessions disconnect on backgrounding by design; background BLE is throttled with a ~10-second wake budget and will not relaunch a force-quit app ([Core Bluetooth background docs](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html)). In practice the owner's phone must be unlocked and on-screen for the whole session — a heavy UX constraint in a gym |
| **Discovery reliability / pairing UX** | **Annoying, not fatal.** Mitigable with QR-assisted pairing, which this app already has |

### Library reality

Nothing here is first-party or turnkey for any of the three candidate stacks:

| Library | Platforms | Status |
| --- | --- | --- |
| [`flutter_nearby_connections`](https://pub.dev/packages/flutter_nearby_connections) | iOS + Android | v1.1.2, **2 years stale** |
| [`nearby_connections`](https://pub.dev/packages/nearby_connections) | Android only | v4.3.0, 18 months old |
| [`flutter_p2p_connection`](https://pub.dev/packages/flutter_p2p_connection) | Android only (iOS "planned") | v3.0.3 |
| [`bonsoir`](https://pub.dev/packages/bonsoir) (mDNS) | Android, iOS, desktop | **Actively maintained** — but a discovery primitive, not a session manager |
| [`flutter_webrtc`](https://pub.dev/packages/flutter_webrtc) | All | **Actively maintained** — again a primitive |
| [`react-native-multipeer`](https://github.com/lwansbrough/react-native-multipeer) | iOS only | **Inactive**, no releases in 12 months |
| [`expo-nearby-connections`](https://github.com/puguhsudarma/expo-nearby-connections) | iOS + Android, **no iOS↔Android interop** | Lightly maintained, ~30 stars |
| Kotlin Multiplatform | — | **Nothing found.** Hand-written `expect`/`actual` against each native SDK |

The maintained options (`bonsoir`, `flutter_webrtc`) are building blocks. Expect to write
real platform-native glue, twice, and to own it forever.

### Merge model

A full CRDT is over-engineered for this. The workload is short-lived, two-writer and
append-only, and the domain model already assigns every `Set` an owner. An **append-only
event log keyed by (device-id, monotonic counter)** merges trivially and files cleanly into
the authoritative cloud log afterwards. Automerge (~300 KB+) or Yjs (~65 KB) would buy
conflict semantics the domain does not generate.

### Shipping precedent

The most instructive finding is a negative one. **Jackbox Party Pack** — the canonical
"everyone in one room, phones as controllers" product — is **not** local P2P: devices go to
`jackbox.tv` and depend on internet connectivity to Jackbox's servers, using the local wifi
only incidentally ([Jackbox](https://www.jackboxgames.com/blog/how-to-play-jackbox-games-with-friends-and-family-remotely)).
A company whose entire product is same-room multi-device sync chose a cloud relay. AirDrop
is the strongest positive precedent but is an OS service using undocumented AWDL, not
replicable by third parties. **[unverified: no primary-source example was found of a
shipping consumer app doing genuine serverless P2P sync of *structured app data* across
iOS and Android.]**

### Honest verdict on P2P

It is **technically feasible and genuinely appealing in the median case**, and it is the
only option that works when the gym wifi has a captive portal. But:

1. **It cannot be the answer on its own**, because free cloud backup is already a committed
   decision. A cloud store must exist regardless. P2P is therefore strictly *additional*
   code, not *alternative* code — it adds a second sync path to build, test and support,
   on top of the one you are building anyway.
2. **Its fatal cases are common, not exotic.** AP isolation is a default on gym guest wifi;
   cellular-only is routine; iOS backgrounding kills the "within seconds" promise unless the
   phone stays lit.
3. **The library ecosystem will not carry it.** Stale or Android-only packages across all
   three candidate stacks, and nothing at all for KMP.

The defensible framing is: **P2P is a possible v2 latency optimisation layered over a cloud
sync path, never a v1 substitute for one.** If it is wanted at all, the cheapest version is
a same-wifi fast path over mDNS + sockets that silently defers to the cloud path whenever
discovery fails — and even that is meaningful work for a benefit (sub-second instead of
~250 ms) that users will not perceive.

---

## 7. Privacy, residency and store disclosures — applies to every option

These obligations arise from the *app*, not from the backend, so they are not a
differentiator — but they must be budgeted for regardless of the choice in #8.

- **Apple privacy labels**: the app must declare **Health & Fitness** (fitness/exercise
  data), **Identifiers** (user ID, device advertising ID), **Purchases**, and **Usage Data**.
  Critically, "you need to identify all of the data you **or your third-party partners**
  collect" — the developer is responsible for the ad network's and any SDK's collection
  ([App Privacy Details](https://developer.apple.com/app-store/app-privacy-details/)).
  A banner ad network almost certainly pushes this into the **Tracking** bucket, which
  triggers App Tracking Transparency.
- **Play Data safety form**: must disclose Health and fitness ("fitness info: exercise,
  physical activity"), Device or other IDs, User IDs, whether data is shared with third
  parties, whether it is encrypted in transit, and whether users can request deletion
  ([Play data safety](https://support.google.com/googleplay/android-developer/answer/10787469)).
- **Play User Data policy** explicitly classes **health data as personal and sensitive**,
  requiring a privacy policy both in Play Console and in-app, plus prominent in-app
  disclosure before the relevant permission request
  ([policy](https://support.google.com/googleplay/android-developer/answer/10144311)).
- **Ads consent**: serving personalised ads in the EEA/UK/Switzerland requires a
  TCF-certified CMP — Google's UMP SDK is the first-party route and also handles the ATT
  prompt ([AdMob UMP](https://developers.google.com/admob/android/privacy)).
- **Transfers**: for EU users on a US-hosted backend, the EU–US Data Privacy Framework
  remains in force — the General Court **dismissed** the annulment action against the
  adequacy decision ([Curia](https://curia.europa.eu/site/upload/docs/application/pdf/2025-09/cp250106en.pdf)).
  Both Firebase and Supabase additionally offer SCC-backed DPAs, so either is defensible.
  **Choosing an EU region is still the lower-friction path**, and it is a one-way door on
  both platforms.

**The residency implication for #8**: because both Firebase and Supabase fix the region
permanently at project creation, the region decision must be made *at the same time* as the
backend decision, not deferred to build time. If EU residency is wanted and RTDB is chosen,
Belgium is the only option.

---

## 8. Side-by-side

| | Firebase (Firestore) | Supabase | Appwrite | InstantDB | Supabase + PowerSync | P2P only |
| --- | --- | --- | --- | --- | --- | --- |
| Google + Apple sign-in | First-party | First-party, native ID-token | First-party | First-party | Via auth provider | n/a |
| Anonymous → upgrade, data preserved | **Yes, UID preserved** | Yes, `linkIdentity()`, opt-in | Yes | Guest Auth **[unverified upgrade path]** | Depends on provider | n/a |
| Realtime push | Snapshot listeners, ~30 ms claimed | Postgres Changes p95 ~228 ms | WebSocket | Built-in | Via sync stream | Sub-second, when it connects |
| Offline persistence | **On by default, mobile** | **None first-party** | **None** | Local replica | SQLite, best-in-class | Local only |
| Conflict model | LWW per document + atomic transforms | LWW, DIY | Server-authoritative | CRDT-ish **[unverified]** | **LWW per field, customizable** | Append-only event log |
| Free tier fits a few hundred users | **Yes** | Yes, but **pauses after 1 week idle** | Yes, pauses idle | Yes, no pause | Yes, pauses idle | n/a |
| Cost curve | **Linear creep from $0** | **Step to $25/mo** | Step to $25/mo | Step to $30/mo | Step to $74/mo combined | $0 |
| EU region | ~13 regions (RTDB: Belgium only), **immutable** | 6+ regions, **immutable** | Frankfurt | **[unverified — likely US only]** | Inherits Postgres host | n/a |
| Flutter SDK | **First-party** | **First-party** | First-party | Community | First-party | Stale/partial |
| RN/Expo SDK | Native needs dev build; JS SDK limited | First-party + plumbing | First-party | **First-party** | First-party | Inactive/partial |
| KMP SDK | Community (GitLive) | Community (supabase-kt) | Community | None | **Official** | None |
| Provides free cloud backup | Yes | Yes | Yes | Yes | Yes | **No — structurally cannot** |

---

## 9. What the evidence points at (not a decision)

Presented as evidence for #8 to weigh, not as a foreclosure.

**The strongest case for Firebase** rests on three facts that map unusually tightly onto
this project's committed decisions: offline persistence works out of the box on both mobile
platforms with no second vendor; the anonymous-to-Google/Apple upgrade preserves the UID and
therefore the user's whole log, which is exactly the map's "local-first with optional
sign-in" requirement; and the cost curve creeps linearly from zero rather than stepping,
which is what "near-zero cost at small scale" actually demands. The costs are a real
vendor-lock-in concern, a document data model that must be designed around a
~1-write/sec-per-document ceiling, no transactions while offline, and no first-party KMP SDK.

**The strongest case for Supabase** is a relational model that fits a workout log more
naturally than documents do, a genuinely open exit path via self-hosting, and better SQL
ergonomics for the analytics deferred to post-v1. It is weakened here by the offline gap —
the app's hardest requirement is the one thing Supabase does not provide first-party — and
by the $25/month step arriving before the app plausibly earns $25/month.

**P2P should be scoped as a possible v2 optimisation**, not a v1 candidate, because free
cloud backup is already committed and therefore a cloud sync path must be built regardless.

**Two questions this research could not close** and which #8 should resolve directly:

1. Whether Supabase anonymous users count toward the MAU quota — this determines whether a
   first-run-creates-anonymous-user design is free or billable.
2. Whether the Firestore free quotas are daily or monthly — the Firebase docs contradict
   themselves, and the answer is a 30x difference in headroom.

**Also to be decided at the same time as #8, because it is a one-way door on every managed
candidate: the database region.**
