# US consumer-health-data laws: research

Research for [#25](https://github.com/kingdragonfly43/gym-notes/issues/25). It builds on
[Privacy policy and store data disclosures](https://github.com/kingdragonfly43/gym-notes/blob/research/privacy-disclosures/docs/research/privacy-disclosures.md)
(#19) and does not repeat its store or GDPR findings. This document gathers findings and
the decisions they force. It is not a privacy policy and makes no product decisions.
Where a finding implies a decision, the decision is flagged for the human under
**Decisions this forces**.

Every finding is graded:

- **Confirmed**: primary source (statute text on a legislature site, an enrolled bill,
  attorney-general guidance or report, a court docket), URL cited.
- **Community**: credible secondary source (law-firm or trade-body analysis, a
  company's own published policy used as evidence of industry practice), URL cited.
- **Inferred**: reasoning from the sources above applied to this app, no URL of its own.

Research date: **2026-10-05**. None of this is legal advice, and the crux question (§2)
has no court ruling or AG guidance that answers it directly.

### Facts this research relies on (from earlier tickets, not re-litigated)

- The developer is a **US-based individual** with no EU or UK establishment, on an Apple
  **Individual** account and a Play **Personal** account
  ([#18](https://github.com/kingdragonfly43/gym-notes/issues/18)). Nothing says a
  company or LLC has been formed.
- **User data** is a user's sessions, splits and exercise library, plus the global unit
  preference ([CONTEXT.md](../../CONTEXT.md)). A **step** holds a weight, a rep count, a
  completion ratio and a unit. An **exercise** carries free-text notes; a session and a
  split day carry a free-text **agenda**. **Personal records** are derived on read and
  never stored.
- **No body mass is stored anywhere.** Under *bodyweight and reps*, the weight is the
  weight added to the body (#19, CONTEXT.md).
- An **anonymous account** exists from first launch. A user who never signs in has no
  cloud copy: their user data stays on the installation. Signing in uploads the user data
  to Cloud Firestore (`us-central1`, Spark, no server) with Google as processor
  ([ADR-0002](../adr/0002-flutter-and-firebase.md), #19).
- Ads: one AdMob interstitial at the start of rest, after a usage threshold, announced by
  one card, with ATT after that card; one non-consumable **Remove Ads** IAP; revenue was
  removed as a design criterion ([#11](https://github.com/kingdragonfly43/gym-notes/issues/11)).
- AdMob collects and shares IP-derived approximate location, device and app identifiers,
  app interactions and diagnostics (#19, §1). **Apple 5.1.3 already forbids any training
  data reaching an ad request** (#19, §3.1). This document takes that rule as given.

---

## Bottom line

1. **Only Washington's My Health My Data Act (MHMDA) is in force, has no size threshold,
   and plausibly reaches the user data itself.** Its definition turns on whether data
   "identifies" health status, not on how the business uses it, and it lists
   "measurements". Nevada's SB 370 and Connecticut's consumer-health-data definition
   cover only data a business *uses* to identify health status, which Gym Notes never
   does. — **Confirmed** (texts) / **Inferred** (application).
2. **Whether sessions, sets and steps are "consumer health data" under MHMDA is genuinely
   unresolved.** No court, AG FAQ or legislative report addresses strength training. The
   AG's closest example ("an app that tracks someone's digestion or perspiration is
   collecting consumer health data") points towards coverage. Major fitness apps publish
   Washington consumer-health-data policies, and MyFitnessPal lists "fitness activity,
   fitness goals, and fitness level" as consumer health data. **Free-text notes and
   agendas are the surer hook:** a user who types "left knee rehab" has handed the app a
   health condition and its treatment, and the app "receives" and "retains" it. —
   **Confirmed** (AG FAQ, statute) / **Community** (industry practice) / **Inferred**
   (verdict).
3. **If it is consumer health data, MHMDA's cost for the core app is modest.** Collecting
   it without consent is allowed "to the extent necessary to provide a product or service
   that the consumer … has requested". Recording a session and backing it up after sign-in
   are requested services. What remains: a **separate consumer health data privacy
   policy**, linked from the store listing and from inside the app; access, deletion and
   consent-withdrawal rights within **45 days**, with an **appeal** route ending at the AG;
   deletion reaching backups within six months; a binding processor contract (Firebase's
   terms); and reasonable security. — **Confirmed** (text) / **Inferred** (application).
4. **AdMob is the expensive edge, and only under a broad reading.** No training data
   reaches an ad request (Apple 5.1.3). But an ad request tells Google that *this
   identifier uses this fitness app*. MHMDA counts "data that identifies a consumer
   seeking health care services", and defines health care services as any service to
   "assess, measure, improve, or learn about" physical health. On that reading every ad
   request **shares** consumer health data with a third party, which needs separate
   opt-in consent, and ad revenue in exchange might even be a **sale**, which needs a
   signed, expiring authorisation. The AG's FAQs read the Act more narrowly, and Google
   says it never uses health information to personalise ads. Risk: low, not zero. —
   **Confirmed** (text, Google statement) / **Inferred** (application).
5. **A never-signed-in user is probably outside MHMDA's practical reach, but not by any
   definition.** Unlike Apple's "collect", MHMDA's covers anything that "access[es],
   retain[s] … or otherwise process[es]" data "in any manner". It has no on-installation
   carve-out. The developer, though, never receives, holds or shares that data, so the
   rights and sharing duties have nothing to act on. AdMob's ad requests go out for these
   users too, so point 4 applies to them regardless. — **Confirmed** (text) /
   **Inferred** (application).
6. **Washington has a private right of action, through the Consumer Protection Act**:
   injunction, actual damages, attorney's fees, discretionary trebling capped at $25,000,
   and class actions. The plaintiff must show injury to "business or property". The AG can
   seek $7,500 per violation. MHMDA class actions so far target ad-SDK data flows
   (Amazon). Nevada, Connecticut, New York's bill and Vermont have AG enforcement only. —
   **Confirmed** (statutes, docket) / **Community** (litigation pattern).
7. **Connecticut became a second live risk on 1 July 2026, through a side door.** The
   CTDPA now applies, with no volume threshold, to anyone who "control[s] or process[es]
   consumers' sensitive data". Sensitive data includes "data revealing … a mental or
   physical health condition, diagnosis, disability or treatment". It also includes data
   from anyone the controller "has actual knowledge, or wilfully disregards, is a child".
   So one Connecticut user's injury note, or one store age signal saying "under 13", may
   pull the whole comprehensive act in: privacy-notice contents, opt-in consent for
   sensitive data, consumer rights, a targeted-advertising opt-out, and a flat ban on
   targeted ads to known 13–17-year-olds. — **Confirmed** (text, AG report) /
   **Inferred** (application).
8. **New York's Health Information Privacy Act passed both houses again (S9269, June
   2026) and has not been delivered to the governor.** Its definition is the broadest:
   information "collected or processed in connection with" health status. It forbids
   selling such information outright and requires a strict authorisation for advertising
   uses. It would take effect six months after signature, plausibly mid-2027. The
   governor vetoed the 2025 version. — **Confirmed** (bill text, Assembly record).
9. **Vermont (Act 145, signed 16 June 2026) adds no-threshold consumer-health-data rules
   from 1 January 2028**, using Connecticut's use-based definition. Maryland, Texas and
   California are threshold-gated and do not reach a solo app on their health-data
   provisions. — **Confirmed**.
10. **No geofencing rule touches Gym Notes as designed.** All four laws ban geofences
    around health facilities. MHMDA's ban applies to "any person", not only regulated
    entities. The app has no location feature. — **Confirmed** (text) / **Inferred**.

---

## 1. Which laws exist, and their status

| Law | Status on 2026-10-05 | Size threshold | Definition style | Enforcement | Grade / source |
|---|---|---|---|---|---|
| **Washington MHMDA**, RCW 19.373 | In force: geofence rule since 23 Jul 2023; everything else since 31 Mar 2024 (30 Jun 2024 for small businesses). Not amended since enactment ("2023 c 191" on every section). | **None.** "Small business" (under 100,000 consumers a year) only got the later start date. | **Content-based**: data that "identifies" health status, plus an inference clause. | AG **and private action** via the Consumer Protection Act. | **Confirmed**, [RCW 19.373](https://app.leg.wa.gov/RCW/default.aspx?cite=19.373&full=true), [WA AG FAQ](https://www.atg.wa.gov/protecting-washingtonians-personal-health-data-and-privacy) |
| **Nevada SB 370**, NRS 603A.400–.550 | In force since 31 Mar 2024. Unamended in the NRS as revised to 15 Apr 2026. | None. | **Use-based**: data a regulated entity "uses to identify" health status. | AG only: "Do not create a private right of action". | **Confirmed**, [NRS 603A](https://www.leg.state.nv.us/NRS/NRS-603A.html), [SB 370 enrolled §36](https://archive.leg.state.nv.us/Session/82nd2023/Bills/SB/SB370_EN.pdf) |
| **Connecticut**, CTDPA §42-526 (consumer health data) | In force since 1 Oct 2023. | None for §42-526. | **Use-based**: data a controller "uses to identify" health condition, diagnosis or status. | AG only. | **Confirmed**, [Conn. Gen. Stat. ch. 743jj](https://www.cga.ct.gov/current/pub/chap_743jj.htm) |
| **Connecticut**, CTDPA as a whole, as amended by **PA 25-113** (1 Jul 2026) and **PA 26-64** (1 Oct 2026) | In force. | **None for anyone who processes sensitive data**; otherwise 35,000 consumers. | Sensitive data includes data **"revealing"** a health condition or treatment (content-based), plus consumer health data and children's data. | AG only. | **Confirmed**, [2026 Supplement](https://www.cga.ct.gov/2026/sup/chap_743jj.htm), [PA 26-64](https://www.cga.ct.gov/2026/ACT/PA/PDF/2026PA-00064-R00SB-00004-PA.PDF) |
| **New York HIPA**, S9269/A10357 (2026) | **Passed** Senate 3 Jun 2026 and Assembly 4 Jun 2026; returned to Senate; not yet delivered to the governor. The 2025 version (S929) was **vetoed** 19 Dec 2025. | None. | **Broadest**: "collected or processed in connection with" health status. | AG only; up to $15,000 per violation. | **Confirmed**, [Assembly record](https://nyassembly.gov/leg/?default_fld=&leg_video=&bn=S09269&term=2025&Summary=Y&Actions=Y&Text=Y), [S929](https://www.nysenate.gov/legislation/bills/2025/S929) |
| **Vermont Act 145** (S.71) | Signed 16 Jun 2026; **effective 1 Jan 2028**. | None for consumer-health-data rules (§2415k); otherwise 35,000 consumers or 3,000 sensitive-data consumers. | Use-based (Connecticut's wording). | AG only; the legislature may add a private right if the AG is not funded. | **Confirmed**, [Act 145 as enacted](https://legislature.vermont.gov/Documents/2026/Docs/ACTS/ACT145/ACT145%20As%20Enacted.pdf) |
| Maryland MODPA | In force. | 35,000 consumers (or 10,000 plus 20% revenue from sales). | Use-based consumer health data. | AG. | **Confirmed**, [Md. Com. Law §14-4702](https://mgaleg.maryland.gov/mgawebsite/Laws/StatuteText?article=gcl&section=14-4702&enactments=false) |
| Texas TDPSA | In force since 1 Jul 2024. | SBA small businesses are exempt except that they must get consent before **selling** sensitive data ("mental or physical health diagnosis"). | — | AG. | **Confirmed**, [HB 4 enrolled, §541.002](https://capitol.texas.gov/tlodocs/88R/billtext/html/HB00004F.htm) |
| California CCPA | In force. | $26.625M revenue, 100,000 consumers, **or 50% of revenue from selling or sharing** personal information. | Health is "sensitive personal information". | CPPA, AG; limited private action for breaches. | **Confirmed**, [CPPA FAQ](https://cppa.ca.gov/faq.html) |

Other states: Michigan and Massachusetts have MHMDA-style bills that are not enacted, and
Maine's LD 1822 died on 13 April 2026. — **Community**,
[Troutman state tracker](https://www.troutmanprivacy.com/2026/03/proposed-state-privacy-law-update-march-30-2026/),
[privacylawmap on LD 1822](https://privacylawmap.com/blog/maine-privacy-law-ld-1822).

**What this means for the map (Inferred):** for v1, the binding laws are Washington
(content-based, private action) and Connecticut (use-based for consumer health data, but
content-based through "sensitive data"). Nevada is almost certainly out. New York is the
one to watch before launch.

---

## 2. The crux: is user data "consumer health data"?

### 2.1 What the definitions say

**Washington** (RCW 19.373.010(8)) — **Confirmed**,
[RCW 19.373.010](https://app.leg.wa.gov/RCW/default.aspx?cite=19.373.010):

> "Consumer health data" means personal information that is linked or reasonably
> linkable to a consumer and that identifies the consumer's past, present, or future
> physical or mental health status.

"Physical or mental health status includes, but is not limited to":

- "(i) Individual health conditions, treatment, diseases, or diagnosis;"
- "(ii) Social, psychological, behavioral, and medical interventions;"
- "(v) Bodily functions, vital signs, symptoms, or measurements of the information
  described in this subsection (8)(b);"
- "(xii) Data that identifies a consumer seeking health care services; or"
- "(xiii) Any information that a regulated entity … processes to associate or identify a
  consumer with the data described in (b)(i) through (xii) … that is derived or
  extrapolated from nonhealth information (such as proxy, derivative, inferred, or
  emergent data …)."

"Health care services" (RCW 19.373.010(15)) means "any service provided to a person to
assess, measure, improve, or learn about a person's mental or physical health", including
"(e) Bodily functions, vital signs, symptoms, or measurements of the information described
in this subsection". "Personal information" includes "data associated with a persistent
unique identifier, such as a cookie ID, an IP address, a device identifier". —
**Confirmed**, same page.

**Washington AG FAQ** — **Confirmed**,
[WA AG](https://www.atg.wa.gov/protecting-washingtonians-personal-health-data-and-privacy)
(undated; "may be periodically updated"):

- FAQ 5: "Information that does not identify a consumer's past, present, or future
  physical or mental health status does not fall within the Act's definition … while
  information about the purchase of toilet paper or deodorant is not consumer health data,
  **an app that tracks someone's digestion or perspiration is collecting consumer health
  data**."
- FAQ 6: "nonhealth data that a regulated entity collects but does not process to
  identify or associate a consumer with a physical or mental health status is not consumer
  health data."

**Nevada** (NRS 603A.430): "personally identifiable information … that a regulated entity
**uses to identify** the past, present or future health status of the consumer". It
includes "(5) Bodily functions, vital signs or symptoms" and excludes information used to
"(b) Identify the shopping habits or interests of a consumer, if that information is not
used to identify the specific past, present or future health status". — **Confirmed**,
[NRS 603A](https://www.leg.state.nv.us/NRS/NRS-603A.html). FPF: Nevada's law covers data
a business "uses to identify", where Washington's covers data that could identify. —
**Community**, [FPF, "Health Data Is What Health Data Does in Nevada"](https://fpf.org/blog/health-data-is-what-health-data-does-in-nevada/).

**Connecticut** (§42-515(9), from 1 Jul 2026): "any personal data that a controller **uses
to identify** a consumer's physical or mental health condition, diagnosis or status". But
**sensitive data** (§42-515(40) after PA 26-64) includes "(A) data **revealing** … (iii) a
mental or physical health condition, diagnosis, disability or treatment", "(B) consumer
health data" and "(D) personal data collected from an individual the controller has actual
knowledge, or wilfully disregards, is a child". — **Confirmed**,
[2026 Supplement](https://www.cga.ct.gov/2026/sup/chap_743jj.htm),
[PA 26-64 §12](https://www.cga.ct.gov/2026/ACT/PA/PDF/2026PA-00064-R00SB-00004-PA.PDF).

**New York S9269** (pending): "Regulated health information" is information reasonably
linkable to an individual (an IP address or device identifier is enough) that "is
**collected or processed in connection with**" past, present or future physical or mental
health status. Status includes "(v) bodily functions, vital signs, symptoms, or
measurements of the information" and "(xii) data that identifies an individual seeking
health care services". — **Confirmed**,
[S9269 text](https://nyassembly.gov/leg/?default_fld=&leg_video=&bn=S09269&term=2025&Summary=Y&Actions=Y&Text=Y).

### 2.2 What industry practice shows

- **MyFitnessPal**'s Washington Consumer Health Data Privacy Policy (effective 13 Dec 2024)
  lists "Fitness activity, fitness goals, and fitness level" and "Information about the
  types of physical activities you engage in, and the duration of your physical activity"
  as consumer health data. — **Community**,
  [MyFitnessPal](https://www.myfitnesspal.com/washington-health-data-privacy-policy)
- **Strava**'s Consumer Health Data Policy (effective 1 Jan 2026) is narrower: "a limited
  amount of the Activity Data we collect may be considered to be Consumer Health Data …
  information that identifies vital signs or health-related measurements". — **Community**,
  [Strava](https://www.strava.com/legal/consumer-health-data-policy)
- The NAI (ad-tech trade body) says the AG's FAQs indicate that readings reaching
  "interest in fitness products or purchase of general toiletries" do "not reflect the
  intent of the law". — **Community**,
  [NAI, 2 Apr 2024](https://thenai.org/washington-my-health-my-data-act/)

These are companies' own risk choices, not law. They show that the conservative reading
is mainstream for fitness apps, and that the line between "activity" and "health
measurement" is drawn differently by different companies. — **Inferred**.

### 2.3 Grading each part of user data (all Inferred unless stated)

| Part of user data | Washington MHMDA | Nevada | Connecticut | New York S9269 (if enacted) |
|---|---|---|---|---|
| **Sessions, sets and steps**: exercise, weight, reps, completion ratio, dates, over years | **Arguable, unresolved.** For: clause (v) "measurements", the AG's "tracks … perspiration" example, and the fact that a load and rep count measure what a body can do. Against: (v) covers measurements "of the information described" (conditions, bodily functions), and a lift is a performance, not a bodily function or symptom; the AG reads "identifies health status" with restraint (toiletries FAQ). Gym Notes stores no body mass, heart rate, sleep or diet, which are what fitness apps' policies actually list. | **Out.** Gym Notes never uses it to identify health status. | **Out as consumer health data** (use-based). Out as sensitive data unless it "reveals" a condition, which loads and reps alone do not. | **Likely in.** "In connection with" physical health status is satisfied by a strength-training app on its plainest reading. |
| **Free-text exercise notes, exercise names and agendas** | **In, for any user who writes health content** ("left knee rehab", "post-surgery", a medication). That is clause (i) conditions and treatment, and "collect" includes "receive" and "retain". The app does not solicit it, but the statute has no unsolicited-data exception. | Out unless used. | **Sensitive data** where it reveals a condition or treatment, which **triggers the whole CTDPA** (§4). | In. |
| **Completion ratio** (half reps; assisted reps modelled, not surfaced) | Same as sets. It could support an inference of reduced capacity, but only clause (xiii) inference counts, and only if the app processes it to associate the user with health status, which it never does (AG FAQ 6). | Out. | Out. | Same as sets. |
| **Personal records** (derived on read, never stored) | Derived, but derived about performance, not health. Clause (xiii) needs processing "to associate or identify a consumer with" health data. **Out**, provided the app never draws health conclusions. | Out. | Out. | Arguably in, as "derivation" of in-scope data. |
| **The fact of being a user**: app identity plus an IP address, device ID or account | **Arguable.** Clause (xii) covers "data that identifies a consumer seeking health care services". Whether a self-entry strength tracker is a "service … to assess, measure, improve … physical health" is the open question (§3.3). | Out. | Out. | Arguable on the same reasoning. |

**Verdict (Inferred):** treat the cloud copy of user data as **consumer health data under
MHMDA**. The sets alone are a coin-flip, but the free-text fields make it near-certain that
*some* users' cloud copies contain it. The app cannot tell which, so it has to treat all of
them alike. Under Nevada and Connecticut's consumer-health-data clause it is **not**
consumer health data, provided one design rule holds:

> **The app never infers, scores or labels health status from user data**, for example
> "possible injury" from a halt in sessions or a drop in loads, or "recovery" streaks.

That rule keeps clause (xiii) of MHMDA and every use-based definition out of reach. It
should go into the spec next to "no training data in ad requests, ever".

**What would settle it:** a WA AG FAQ or a court ruling on fitness data. Neither exists as
of this research. — **Confirmed** (by absence from the AG FAQ and from the dockets checked).

**What would change it:** adding body mass or body measurements (clause (v), and listed by
MyFitnessPal), a HealthKit or Health Connect import (also the FTC Health Breach
Notification Rule's "multiple sources" test, #19 §3.4), heart rate, or any health-framed
feature. Each moves the user data from arguable to plainly covered, in Nevada and
Connecticut too if the feature *uses* the data to identify health status. — **Inferred**.

---

## 3. Washington MHMDA in detail

### 3.1 Does it reach this developer?

- "Regulated entity" means "any legal entity that: (a) Conducts business in Washington, or
  produces or provides products or services that are targeted to consumers in Washington;
  and (b) alone or jointly with others, determines the purpose and means of collecting,
  processing, sharing, or selling of consumer health data." — **Confirmed**,
  RCW 19.373.010(23).
- AG FAQ 3: "Generally, all persons and businesses that conduct business in Washington (or
  provide services or products to Washington), and that collect, process, share, or sell
  consumer health data are impacted by the Act." — **Confirmed**, WA AG FAQ.
- "Small business" means a regulated entity that "(a) Collects, processes, sells, or shares
  consumer health data of fewer than 100,000 consumers during a calendar year", among
  others. Small businesses got a **later start date (30 June 2024), not an exemption**. —
  **Confirmed**, RCW 19.373.010(28) and each section's subsection (2).
- "Consumer" means "(a) a natural person who is a Washington resident; or (b) a natural
  person whose consumer health data is collected in Washington". — **Confirmed**,
  RCW 19.373.010(7).

**Application (Inferred):**

- Selling Remove Ads to Washington residents through the US storefronts, in English and
  USD, is very likely "conducting business in Washington". The app is in scope.
- **"Legal entity" and a sole proprietor.** Sections 2–8 bind a "regulated entity", defined
  as a "legal entity". Sections 9 (sale) and 10 (geofence) bind any "person", which
  expressly includes natural persons. An unincorporated individual could argue he is not a
  "legal entity". It is an untested argument that a court applying a statute framed as
  closing a privacy gap is unlikely to accept. Do not plan around it.
- "Collected in Washington" reaches a user from another state who trains while visiting
  Washington. **The app cannot know who is a Washington consumer**, so in practice it
  applies the Act to every US user or to none.

### 3.2 What it requires, if user data is consumer health data

| Obligation | Text | Grade | What it means for Gym Notes (Inferred) |
|---|---|---|---|
| **Separate consumer health data privacy policy** | Must disclose categories collected and purposes, sources, categories shared, "a list of the categories of third parties and specific affiliates" shared with, and how to exercise rights. "A regulated entity and a small business shall prominently publish a link to its consumer health data privacy policy on its homepage." (RCW 19.373.020) | **Confirmed** | A second document, not a section of the main policy. |
| **"Separate and distinct"** | AG FAQ 4: the policy "must be a separate and distinct link on the regulated entity's homepage and may not contain additional information not required under the My Health My Data Act." | **Confirmed**, WA AG FAQ | It cannot be folded into the general policy page as a heading. |
| **Where "homepage" is, for an app** | "the application's platform page or download page, and a link within the application, such as from the application configuration, 'about,' 'information,' or settings page." (RCW 19.373.010(16)) | **Confirmed** | Two placements: the **store listing** and **in-app settings/about**. Both stores have one privacy-policy URL field. Pointing it at a page that carries two distinct links (general policy, consumer health data policy) is the common practice. Whether that satisfies "platform page" is untested. |
| **Consent to collect** | Not required "to the extent necessary to provide a product or service that the consumer … has requested". Otherwise opt-in consent "for a specified purpose", obtained before collection. (RCW 19.373.030(1)(a)) | **Confirmed** | **No consent screen is needed for the core app.** Recording sessions is the requested product; cloud backup and sync are requested at sign-in. Any *other* purpose, such as analytics or crash reporting that touched user data, would need opt-in consent. |
| **Consent to share** | "separate and distinct from the consent obtained to collect", unless necessary for the requested product. Consent must disclose categories, purpose, "categories of entities with whom the consumer health data is shared", and how to withdraw. Consent cannot come from accepting general terms, closing content, or "deceptive designs". (RCW 19.373.030(1)(b)–(c), 19.373.010(6)) | **Confirmed** | Not needed for Firebase (a processor) or for friends (requested by the owner). Needed for **AdMob only if** an ad request carries consumer health data (§3.3). |
| **"Share" excludes processors** | Disclosure "to a processor when such sharing is to provide goods or services in a manner consistent with the purpose for which the consumer health data was collected and disclosed to the consumer" is not sharing. (RCW 19.373.010(27)(b)(i)) | **Confirmed** | Firestore and Firebase Auth are processors under Firebase's Data Processing and Security Terms (#19, §5.3). |
| **Processor contract** | A processor may process only "pursuant to a binding contract … that sets forth the processing instructions and limit[s] the actions the processor may take". (RCW 19.373.060) Contracting with a processor inconsistently with the policy is itself a violation (19.373.020(1)(e)). | **Confirmed** | Accept Firebase's terms in the console and keep a record. Whether Google's standard terms meet "processing instructions" is for a lawyer, but this is the industry's normal reliance. |
| **Right to access** | Confirm collection and "access such data, including a list of all third parties and affiliates with whom" it was shared or sold, with an "active email address or other online mechanism" for each. (19.373.040(1)(a)) | **Confirmed** | The export file covers the user data itself. The third-party list is empty or short if only processors receive it. |
| **Right to withdraw consent** | (19.373.040(1)(b)) | **Confirmed** | Only bites if consent was relied on (e.g., for ads, §3.3). |
| **Right to delete** | Delete "from all parts of the … network, including archived or backup systems", and notify processors and third parties. Backup deletion may be delayed, but "such delay may not exceed six months". (19.373.040(1)(c)) | **Confirmed** | Feeds #24. Firebase Auth clears backups "within 180 days" (#19), roughly six months. Firestore's own backup retention on Spark was not established here. |
| **Timing and appeals** | Within **45 days**, extendable once by 45 with notice. Free up to twice a year. An appeal process, answered in writing within 45 days, and on denial a way to complain to the AG. (19.373.040(1)(f)–(h)) | **Confirmed** | GDPR's one month (#19) is stricter, so meeting it meets this. **The appeal route and AG pointer are new.** |
| **Authentication** | May require use of "an existing account", may not require creating one. (19.373.040(1)(d)–(e)) | **Confirmed** | The signed-in session authenticates in-app requests. The Play web deletion path (#19) can match the provider email. |
| **Security and access limits** | Restrict access to those who need it; security meeting "reasonable standard of care within the … industry". (19.373.050) | **Confirmed** | Firestore security rules scoped per account. |
| **Sale** | Unlawful without a signed "valid authorization": named purchaser, purpose, one-year expiry, signature, six-year retention. (19.373.070) | **Confirmed** | Never sell. See §3.3 for the AdMob edge case. |
| **Geofencing** | Unlawful for "any person" to geofence (2,000 ft or less) "around an entity that provides in-person health care services" to track, collect from, or message consumers. (19.373.080, .010(14)) | **Confirmed** | Not implicated. A future "gym check-in" feature must not draw boundaries; a physical-therapy clinic is in-person health care, and gyms often host one. |
| **Non-discrimination** | "may not unlawfully discriminate against a consumer for exercising any rights" (19.373.030(1)(d)) | **Confirmed** | Declining ad-sharing consent (if ever asked) cannot cost the user features. |

### 3.3 Does AdMob's collection count as collecting or sharing consumer health data?

**The facts (Confirmed, from #19):** the Google Mobile Ads SDK sends IP address
(approximate location), device and app identifiers, app interactions and diagnostics to
Google, for advertising, analytics and fraud prevention, and Google shares it. Apple 5.1.3
forbids training data in ad requests, and nothing from user data is ever put into one.

**What reaches Google, then (Inferred):** an identifier, an approximate location, and the
fact that the request comes from Gym Notes. An AdMob ad unit belongs to one app, so the
request identifies the app.

**Under MHMDA (Inferred):**

1. IP-derived approximate location is **not "precise location information"**, which needs
   accuracy "within a radius of 1,750 feet" (RCW 19.373.010(19)). Clause (xi) is not
   triggered. — **Confirmed** (definition) / **Inferred**.
2. The identifiers are "personal information" (RCW 19.373.010(18)). They become consumer
   health data only if they "identify the consumer's … health status". The only route is
   clause (xii), "data that identifies a consumer seeking health care services", and only
   if Gym Notes is a "health care service" ("any service provided to a person to assess,
   measure, improve, or learn about a person's … physical health").
3. **Narrow reading (more likely):** a self-entry training notebook is not a health care
   service. The AG's FAQs read "health status" with restraint, and the NAI reads them as
   rejecting "interest in fitness products" as health data. On this reading AdMob receives
   no consumer health data, and nothing changes.
4. **Broad reading (plaintiff's):** helping someone "improve" physical health is exactly
   what a training app does. Every ad request then **shares** consumer health data with a
   third party (Google is not a processor when it uses the data for its own advertising).
   That needs **separate opt-in consent** first, with the RCW 19.373.030(1)(c) disclosures.
   Because Google pays for the impressions, a plaintiff could also argue the exchange is a
   **sale**, needing a signed, one-year, named-purchaser authorisation. That would make
   ads in Washington impractical.
5. **Google's own statement** cuts towards the narrow reading but does not settle it:
   "we never use sensitive information like health, race, religion, or sexual orientation
   to personalize ads". — **Confirmed**,
   [Google, restricted data processing](https://business.safety.google/rdp/) (updated July 2023).
   That speaks to Google's *use*. MHMDA's "share" turns on *disclosure*.
6. **Non-personalised or restricted processing helps, partly.** With restricted data
   processing, Google says "we will act as your service provider (or processor)" and limits
   use to ad delivery, measurement, security and similar purposes. — **Confirmed**, same
   page. A disclosure to a processor "consistent with the purpose for which the consumer
   health data was collected" is not sharing. But serving ads is not a purpose the data was
   collected for, so this is a partial argument at best. — **Inferred**. AdMob removed its
   account-level "US State Regulations" setting on 16 June 2025; non-personalised ads for US
   states are now configured through the US-states message in Privacy & messaging. —
   **Community** (Google SDK forum advisor reply, 8 May 2025,
   [thread](https://groups.google.com/g/google-admob-ads-sdk/c/atami0BKI00)).
7. **The litigation pattern points here.** The first MHMDA class action, *Maxwell v.
   Amazon.com* (W.D. Wash. No. 2:25-cv-00261, filed 10 Feb 2025), targets an advertising
   SDK embedded in third-party apps. It was consolidated as *In re Amazon Ads SDK
   Litigation*, No. 2:25-cv-00252. — **Confirmed**,
   [CourtListener docket](https://www.courtlistener.com/docket/69628048/maxwell-v-amazoncom-inc/),
   [consolidation order](https://docs.justia.com/cases/federal/district-courts/washington/wawdce/2:2025cv00261/344509/16).
   That case turns on *precise* location revealing clinic visits, which Gym Notes never
   sends. Its later status could not be confirmed from primary sources. Secondary sources
   conflict on whether it was dismissed. — **Community**,
   [WilmerHale](https://www.wilmerhale.com/en/insights/blogs/wilmerhale-privacy-and-cybersecurity-law/20250220-first-lawsuit-filed-under-washingtons-my-health-my-data-act).

**Under Nevada and Connecticut (Inferred):** both use-based. Neither the app nor Google
uses the ad request to identify health status (Google's statement above). **Not consumer
health data.** Connecticut's *targeted advertising* rules may still apply for other
reasons (§4).

**Under New York S9269, if enacted (Inferred):** "processing" without authorisation is
lawful only if "strictly necessary" for the requested product or for internal operations,
"which exclude any activities related to marketing, advertising". If the broad reading
makes the app's identifiers regulated health information, **every ad request needs a
separate, plain-language, 12-point authorisation, renewed at least yearly**. Selling it to
a third party is unlawful outright, with no consent route. — **Confirmed** (text) /
**Inferred** (application).

**The options this leaves (Inferred; decision for the human):**

| Option | MHMDA position | Cost |
|---|---|---|
| A. Ads as planned, rely on the narrow reading | Defensible; untested. | None now. Litigation risk stays (§6). |
| B. Non-personalised / restricted processing for all US users | Narrow reading plus a processor argument. Also removes CTDPA "targeted advertising" duties (§4) and aligns with #23's `TEEN` option. | eCPM only, already ruled out as a criterion by #11. |
| C. Ask a separate MHMDA-style **sharing consent** on the announcement card, before any ad; no ads for anyone who declines | Satisfies even the broad reading's *share* consent (not a *sale*). | One more choice on a screen that already exists; lost impressions from decliners. Cannot be merged with ATT or with general terms. |
| D. No ads where the law is unclear (WA), geolocated by IP | Satisfies everything. | IP location is unreliable and "collected in Washington" includes visitors, so it leaks both ways. |

B and C stack: B shrinks what is disclosed, and C covers the remainder.

---

## 4. Connecticut: the sensitive-data trigger

- From **1 July 2026** the CTDPA applies to persons that "(1) … controlled or processed
  the personal data of not fewer than thirty-five thousand consumers …; (2) control or
  process consumers' sensitive data …; or (3) offer consumers' personal data for sale in
  trade or commerce." — **Confirmed**, §42-516 as amended by PA 25-113 §6,
  [2026 Supplement](https://www.cga.ct.gov/2026/sup/chap_743jj.htm).
- The AG: "expansion of the CTDPA to all processing of sensitive data and all sales of
  such data is unique to Connecticut". The §42-526 consumer-health-data duties "apply to
  all consumer health data controllers who do business in Connecticut, regardless of their
  size". The AG is investigating a fertility-tracker app, testing "their app on both iOS
  and Android devices to review data flows". It is also investigating a data broker "that
  offers SDKs to app developers". Companies "may not willfully blind themselves to users'
  age". — **Confirmed**,
  [CT AG 2025 CTDPA enforcement report, Feb 2026](https://portal.ct.gov/-/media/ag/press_releases/2026/annual-report-final-2.pdf)
- PA 26-64 (effective 1 Oct 2026) left the consumer-health-data and sensitive-data
  definitions as PA 25-113 wrote them. — **Confirmed**,
  [PA 26-64 §12](https://www.cga.ct.gov/2026/ACT/PA/PDF/2026PA-00064-R00SB-00004-PA.PDF)

**How Gym Notes could fall in (Inferred):**

1. **A Connecticut user's note that reveals an injury or treatment** is "data revealing … a
   physical health condition … or treatment". Once it is in the cloud copy, the developer
   "controls or processes" sensitive data, and the whole act applies to that user.
2. **A store age signal saying "under 13"** gives "actual knowledge" that a user is a child
   (CTDPA's "child" is COPPA's). Personal data from that user is sensitive data. #19 found
   the app must consume these signals for Texas, Utah and Louisiana. The same signal may
   switch on the CTDPA for a Connecticut child.
3. Loads and reps alone do not "reveal a condition". Without notes or age signals, the
   CTDPA turns on the 35,000-consumer threshold, which a solo app will not reach.

**What the CTDPA then requires (Confirmed, §§42-518, 42-520, 42-522 as amended; Inferred
where marked):**

- **Consent before processing sensitive data**, and only where "reasonably necessary"
  (§42-520(a)(1)(D)). There is no "requested service" exception like Washington's in that
  clause. §42-524's "provide a product or service specifically requested by a consumer"
  construction rule may cover the core function, but that is untested. — **Confirmed**
  (text) / **Inferred** (interplay).
- **No targeted advertising or sale** for a consumer the controller knows, or wilfully
  disregards, "is at least thirteen years of age but younger than eighteen" — with no
  consent route (§42-520(a)(1)(I)). Store age signals supply that knowledge. —
  **Confirmed** / **Inferred**.
- A **privacy notice** with categories, purposes, rights and appeal, a "clear and
  conspicuous disclosure" of targeted advertising, an email or online contact, a statement
  on **large-language-model training**, and the **month and year last updated**. It must be
  linked with the word "privacy" on the **store page** and in the **app's settings menu**.
  A single generally applicable notice is allowed (§42-520(b)).
- Rights to access, correct, delete, portability, and to **opt out of targeted advertising
  and sale**, including through **opt-out preference signals** such as GPC, answered within
  45 days with an appeal (§§42-518, 42-520(c)).
- A **data protection assessment** for processing sensitive data or for targeted
  advertising (§42-522).
- "Targeted advertising" excludes ads "based on the context of a consumer's … visit to an
  … online application" (§42-515). **Non-personalised AdMob ads fall outside it.** —
  **Confirmed** (definition) / **Inferred** (fit).
- Enforcement is the AG's alone, as an unfair trade practice. Cure is at the AG's
  discretion since 2025. "Nothing … shall be construed as providing the basis for … a
  private right of action". — **Confirmed**, §42-525.

---

## 5. Is a never-signed-in user outside scope?

**The definitions (Confirmed):**

- MHMDA: "'Collect' means to buy, rent, access, retain, receive, acquire, infer, derive, or
  otherwise process consumer health data in any manner"; "'Process' … means any operation
  or set of operations performed on consumer health data". — RCW 19.373.010(5), (20).
- Nevada: the same "collect" wording. — NRS 603A.420.
- Connecticut: "process" includes "collection, use, storage, disclosure, analysis,
  deletion or modification", and a controller "determines the purpose and means of
  processing". — §42-515.
- New York S9269: "processing" includes "retention, creation, generation, derivation,
  recording … storage", and a regulated entity "controls the processing". — S9269 §1120.
- Contrast Apple: "Data that is processed only on device is not 'collected'" (#19, §1.1).
  None of the state laws draws that line.

**Application (Inferred):**

1. **No statute exempts processing that never leaves the user's phone.** Nobody has tested
   whether a developer whose code stores data on the user's own installation "processes"
   it, or "determines the means". The text is broad enough to argue either way.
2. **In practice the obligations have nothing to bite on.** Collection for the requested
   product needs no consent (MHMDA). The developer holds nothing to access or delete. On
   the installation, the user deletes by uninstalling, and the export file already gives
   access. Nothing is shared. The privacy policy only has to say that local-only user data
   never reaches the developer. The residual exposure is close to nil.
3. **The never-signed-in user is not outside AdMob's flow.** Ad requests go out for anonymous
   accounts too, after the usage threshold. Whatever §3.3 concludes applies to every user
   who sees ads, signed in or not.
4. **Sign-in is the moment that matters.** It is when user data first reaches the
   developer's processor. It is the natural place for the disclosures, and for consents
   if any are needed. Connecticut's sensitive-data consent may be one, and #19's
   conservative GDPR reading is another.

---

## 6. Private right of action, and what it means for a solo developer

- MHMDA §090: a violation "is an unfair or deceptive act in trade or commerce and an
  unfair method of competition for the purpose of applying the consumer protection act",
  and the practices are "matters vitally affecting the public interest". — **Confirmed**,
  [RCW 19.373.090](https://app.leg.wa.gov/RCW/default.aspx?cite=19.373&full=true)
- CPA private action: "Any person who is injured in his or her business or property"
  may sue to enjoin and recover "actual damages sustained by him or her, or both, together
  with the costs of the suit, including a reasonable attorney's fee". Damages may be
  trebled, but the increase "may not exceed twenty-five thousand dollars". —
  **Confirmed**, [RCW 19.86.090](https://app.leg.wa.gov/RCW/default.aspx?cite=19.86.090)
- AG civil penalty: "not more than $7,500 for each violation". — **Confirmed**,
  [RCW 19.86.140](https://app.leg.wa.gov/RCW/default.aspx?cite=19.86.140)
- AG FAQ 2: enforced "by the Attorney General as well as through private action". —
  **Confirmed**, WA AG FAQ.
- Practitioners read §090 as satisfying the CPA's unfair-act and public-interest elements
  automatically. That leaves injury to business or property, and causation, as the
  plaintiff's real hurdles. Emotional harm does not count. — **Community**,
  [Byte Back, analysing the private right of action](https://www.bytebacklaw.com/2023/05/analyzing-the-washington-my-health-my-data-acts-private-right-of-action/)
- No WA AG enforcement action under MHMDA was found in the AG's FAQ, its 2026 Data Privacy
  Report, or news coverage. — **Confirmed** (by absence:
  [WA AG Data Privacy Report 2026](https://agportal-s3bucket.s3.us-west-2.amazonaws.com/Data%20Privacy/DPR_2026_final.pdf)
  lists MHMDA among existing laws without enforcement detail).
- Nevada (NRS 603A.550), Connecticut (§42-525(d)), New York S9269 (§1127, AG only) and
  Vermont (Act 145 §2) all exclude private actions. Vermont's legislature says it "may
  consider adding a private right of action" if the AG is not funded. — **Confirmed**.

**What it implies (Inferred):**

1. **The exposure is personal.** With no company formed, a judgment or settlement falls on
   the developer as an individual. Forming an LLC and buying liability insurance are the
   usual answers. Both are decisions for the human, with a lawyer, and neither is a
   compliance step.
2. **The economics of a claim are attorney's fees, not damages.** Trebling is capped at
   $25,000, but fee-shifting and class aggregation are what make a small defendant worth a
   demand letter. Defence costs would dwarf any plausible damages.
3. **Plaintiffs' firms so far chase ad-SDK data flows at large companies.** A solo app with
   few Washington users is a poor class target, but a template demand letter costs little
   to send.
4. **The best protection is cheap: give a demand letter nothing to point at.** That means
   the separate consumer health data policy linked in both places, a working rights flow
   with an appeal, no consumer health data in ad requests (already a store rule), and a
   decided position on §3.3 recorded with its reasons. Each item is mostly documentation.
5. **Injury is the defence's strongest ground.** The app is free, and nothing is sold. A
   plaintiff would have to show injury to "business or property", for example the value of
   data taken or money spent on Remove Ads under a misleading policy. That is weaker than in
   the SDK cases, which allege location data was monetised.

---

## 7. Nevada and New York, briefly

**Nevada (Confirmed, NRS 603A):** a consumer must be someone "who has requested a product or
service from a regulated entity" and resides in Nevada or whose data is collected there.
The privacy policy must also state "Whether a third party may collect consumer health data
over time and across different Internet websites or online services" and the policy's
effective date (603A.495). Deletion within **30 days** after authentication, with backup
delay up to **2 years** (603A.515). Geofence ban within 1,750 ft of medical facilities
(603A.540). Willful violations carry civil penalties up to $15,000 each (NRS 598.0999).
**Application (Inferred):** nothing in Gym Notes is used to identify health status, so
Nevada does not reach it. It becomes relevant only if a health-inference feature is added.

**New York S9269 (Confirmed, bill text):** applies to any entity controlling regulated
health information of a New York resident or of anyone "physically present in New York".
Selling regulated health information is unlawful outright. Other processing needs either
strict necessity (with a notice) or a separate authorisation: plain language, at least
12-point font, stating that declining will not affect the service, expiring within a year,
with a revoke control in account settings, and no re-ask for nine months after a refusal.
An account deletion request "shall be treated as a request to delete", within **30 days**.
Disposal follows "a publicly available retention schedule", within 60 days of no longer
being needed. AG penalties up to $15,000 per violation. Effective "6 months after it shall
have become a law".

**Application (Inferred):** the core app fits "strictly necessary … providing … a specific
product, feature, or service requested", which needs a clear notice, not an authorisation.
Advertising is the problem (§3.3). The 2025 bill passed in January and was not delivered
to the governor until 8 December 2025 ([S929](https://www.nysenate.gov/legislation/bills/2025/S929)). S9269 is likely to be decided in late 2026 and, if signed, to bite
around mid-2027.

---

## Decisions this forces (flagged for the human, not taken here)

1. **Health-data stance under MHMDA.** Treat the cloud copy of user data as consumer health
   data (the recommended conservative reading, given free-text notes), or rely on the
   narrow reading. Everything below assumes the conservative one.
2. **A separate consumer health data privacy policy.** One document covering Washington
   (and Nevada for good measure), linked distinctly from the store listings' privacy URL
   target and from in-app settings/about. It must not carry content beyond what MHMDA
   requires. This feeds #27.
3. **AdMob under the broad reading.** Choose among §3.3's options: (A) ads as planned; (B)
   non-personalised / restricted processing for all US users; (C) a separate sharing
   consent on the announcement card, with no ads for those who decline; (D) geo-excluding
   Washington. B and C combine. Coordinate with #23, where blanket `TEEN` treatment is
   already on the table.
4. **The "no health inference" design rule.** Adopt as a spec rule: the app never infers,
   scores or labels health status from user data. It is what keeps Nevada, Connecticut's
   consumer-health-data clause and MHMDA's inference clause out of reach.
5. **Free-text fields.** Accept that notes and agendas can hold health information (and
   therefore Connecticut sensitive data), or add a one-line hint against entering medical
   details. A hint reduces the risk but does not remove it. Removing the fields was not
   considered, since it is a product question.
6. **Consent at sign-in.** Whether the sign-in screen carries an explicit consent to cloud
   storage of user data "which may include health information". It would satisfy
   Connecticut's sensitive-data consent, #19's conservative GDPR Art. 9 reading if EEA/UK
   ships (#22), and New York's notice duty. It must be separate from terms acceptance.
7. **Rights handling.** Add to whatever #24 and #27 settle: a 45-day clock (30 days if New
   York's bill is signed), an **appeal step** with the Washington AG's complaint link, and
   processor notification. "Delete the cloud copy" must also reach backups within six
   months.
8. **Business form and insurance.** Whether to form an LLC before launch, and whether to
   insure. Raised by the Washington private right of action; a decision for the human with
   counsel.
9. **Location features, ever.** Record that the app has no geofence. Any future location
   feature (gym check-in, nearby friends) needs review against the four geofence bans.
10. **Watch list before launch.** New York S9269's signature or veto; any Washington AG FAQ
    on fitness data; *In re Amazon Ads SDK Litigation*; Vermont's January 2028 start.

## What feeds other tickets

**[#27 The privacy policy: contents, host and public identity](https://github.com/kingdragonfly43/gym-notes/issues/27)**

- A **second, separate** consumer health data policy (Washington: distinct link, no extra
  content; Nevada: also state whether third parties track across sites and services, plus
  an effective date).
- **Placement**: the store listing ("platform page or download page") **and** in-app
  settings/about, for both the Washington and the Connecticut notices. Connecticut wants
  the word "privacy" in the link text.
- **New content items from Connecticut (if its act applies):** a large-language-model
  training statement (say "no"), the month and year last updated, a targeted-advertising
  disclosure and opt-out (unless ads are non-personalised), and an appeal route.
- **The appeal process** with the Washington AG's complaint mechanism, and Nevada's AG
  contact on a refused appeal.
- **Retention statement** that reconciles Firebase's 180-day backup clearance with
  Washington's six-month backup limit.
- The policy's **change process** (already a Texas item, #19) gains a Connecticut duty: a
  retroactive material change requires notifying affected consumers and offering
  withdrawal (§42-520(b)(3)).

**[#23 Store age signals, minors and ad treatment](https://github.com/kingdragonfly43/gym-notes/issues/23)**

- **A store "under 13" signal may switch on the whole CTDPA** for a Connecticut user
  (sensitive data, no threshold), on top of COPPA's actual-knowledge rule.
- **A store 13–17 signal bans targeted advertising and sale for that Connecticut user,
  with no consent route** (§42-520(a)(1)(I), "younger than eighteen" since 1 Jul 2026).
  The Connecticut AG says companies "may not willfully blind themselves to users' age". So
  consuming the signal and then ignoring it is the worst position.
- Non-personalised ads (option B in §3.3) settle the Connecticut minors problem and the
  MHMDA-sharing question at once. It is the same lever as blanket `TEEN` treatment.

**[#24 Account deletion under the daily delete quota](https://github.com/kingdragonfly43/gym-notes/issues/24)**

- Deadlines: Washington **45 days** (+45 with notice), Nevada **30 days** after
  authentication, Connecticut 45 days, New York **30 days** (pending). A multi-day
  client-driven deletion fits inside all of them.
- Washington requires deletion **from backups within six months**, and notice to
  processors. Nevada allows 2 years for backups.
- New York (pending) would treat any **account deletion request as a health-data
  deletion request**.

**[#26 Crash reporting and analytics in v1](https://github.com/kingdragonfly43/gym-notes/issues/26)**

- Under MHMDA, collection beyond the requested service needs opt-in consent, and sharing
  with a non-processor needs a second, separate consent. Any crash or analytics payload
  that could carry user data (exercise names, notes, agendas in breadcrumbs or logs)
  would need both. Keep user data out of telemetry entirely.

## New questions that look like their own tickets

- **CCPA through ad revenue.** The CCPA reaches any business that "derive[s] 50% or more
  of their annual revenue from selling or sharing California residents' personal
  information". If AdMob *personalised* ads are most of the app's revenue, that prong may
  apply regardless of size. It is not a health-data question, but it is a size-independent
  trigger in the largest US market, and option B (non-personalised ads) likely defuses it.
  — **Confirmed** (threshold, [CPPA FAQ](https://cppa.ca.gov/faq.html)) / **Inferred**
  (application).
- **New York S9269 launch gate.** If it is signed this winter, it takes effect around
  mid-2027 with an advertising-authorisation regime unlike any other state's. Worth a
  standing check before v1 ships, not a ticket yet.
- **Body measurement tracking (in fog).** If it is ever added, user data becomes plainly
  consumer health data in Washington and New York. The Apple and Play declarations may
  move from Fitness towards Health (#19), and the sign-in consent question becomes a must.

---

## Sources

**Washington**
- RCW 19.373 (full chapter): https://app.leg.wa.gov/RCW/default.aspx?cite=19.373&full=true
- RCW 19.373.010 (definitions): https://app.leg.wa.gov/RCW/default.aspx?cite=19.373.010
- WA AG, My Health My Data FAQ: https://www.atg.wa.gov/protecting-washingtonians-personal-health-data-and-privacy
- WA AG Data Privacy Report 2026: https://agportal-s3bucket.s3.us-west-2.amazonaws.com/Data%20Privacy/DPR_2026_final.pdf
- ESHB 1155 final bill report: https://lawfilesext.leg.wa.gov/biennium/2023-24/Pdf/Bill%20Reports/House/1155-S.E%20HBR%20FBR%2023.pdf
- RCW 19.86.090 (private action): https://app.leg.wa.gov/RCW/default.aspx?cite=19.86.090
- RCW 19.86.140 (civil penalties): https://app.leg.wa.gov/RCW/default.aspx?cite=19.86.140
- *Maxwell v. Amazon.com*, No. 2:25-cv-00261 (W.D. Wash.): https://www.courtlistener.com/docket/69628048/maxwell-v-amazoncom-inc/
- Consolidation as *In re Amazon Ads SDK Litigation*: https://docs.justia.com/cases/federal/district-courts/washington/wawdce/2:2025cv00261/344509/16

**Nevada**
- NRS 603A (rev. 15 Apr 2026): https://www.leg.state.nv.us/NRS/NRS-603A.html
- SB 370 (2023) enrolled: https://archive.leg.state.nv.us/Session/82nd2023/Bills/SB/SB370_EN.pdf
- NRS 598 (penalties): https://www.leg.state.nv.us/NRS/NRS-598.html

**Connecticut**
- Chapter 743jj (current): https://www.cga.ct.gov/current/pub/chap_743jj.htm
- 2026 Supplement (PA 25-113 amendments, effective 1 Jul 2026): https://www.cga.ct.gov/2026/sup/chap_743jj.htm
- Public Act 26-64 (SB 4, effective 1 Oct 2026): https://www.cga.ct.gov/2026/ACT/PA/PDF/2026PA-00064-R00SB-00004-PA.PDF
- CT AG 2025 CTDPA enforcement report (Feb 2026): https://portal.ct.gov/-/media/ag/press_releases/2026/annual-report-final-2.pdf

**New York**
- S9269 actions and text (Assembly site): https://nyassembly.gov/leg/?default_fld=&leg_video=&bn=S09269&term=2025&Summary=Y&Actions=Y&Text=Y
- S9269 (Senate site): https://www.nysenate.gov/legislation/bills/2025/S9269
- S929 (2025, vetoed): https://www.nysenate.gov/legislation/bills/2025/S929

**Vermont**
- Act 145 as enacted: https://legislature.vermont.gov/Documents/2026/Docs/ACTS/ACT145/ACT145%20As%20Enacted.pdf
- Bill status S.71: https://legislature.vermont.gov/bill/status/2026/S.71

**Other states**
- Maryland Com. Law §14-4701: https://mgaleg.maryland.gov/mgawebsite/Laws/StatuteText?article=gcl&section=14-4701&enactments=false
- Maryland Com. Law §14-4702: https://mgaleg.maryland.gov/mgawebsite/Laws/StatuteText?article=gcl&section=14-4702&enactments=false
- Texas HB 4 (TDPSA) enrolled: https://capitol.texas.gov/tlodocs/88R/billtext/html/HB00004F.htm
- CPPA FAQ (CCPA thresholds): https://cppa.ca.gov/faq.html

**Google**
- Restricted data processing: https://business.safety.google/rdp/
- AdMob, US states privacy laws: https://support.google.com/admob/answer/9561022
- Google Mobile Ads SDK forum, US State Regulations setting removal (8 May 2025): https://groups.google.com/g/google-admob-ads-sdk/c/atami0BKI00

**Community**
- FPF, "Health Data Is What Health Data Does in Nevada": https://fpf.org/blog/health-data-is-what-health-data-does-in-nevada/
- NAI on MHMDA (2 Apr 2024): https://thenai.org/washington-my-health-my-data-act/
- Byte Back on MHMDA's private right of action: https://www.bytebacklaw.com/2023/05/analyzing-the-washington-my-health-my-data-acts-private-right-of-action/
- WilmerHale on the first MHMDA lawsuit: https://www.wilmerhale.com/en/insights/blogs/wilmerhale-privacy-and-cybersecurity-law/20250220-first-lawsuit-filed-under-washingtons-my-health-my-data-act
- MyFitnessPal Washington Consumer Health Data Privacy Policy: https://www.myfitnesspal.com/washington-health-data-privacy-policy
- Strava Consumer Health Data Policy: https://www.strava.com/legal/consumer-health-data-policy
- Troutman state privacy tracker (30 Mar 2026): https://www.troutmanprivacy.com/2026/03/proposed-state-privacy-law-update-march-30-2026/
- privacylawmap on Maine LD 1822: https://privacylawmap.com/blog/maine-privacy-law-ld-1822
