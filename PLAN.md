# Twrlzy native app: plan

Status: draft for Pat to approve. No native code is written until this is approved.
Date: 17 Nov 2026.

Labels used below:
- **[sure]** I'm confident, or it comes from Apple's own docs.
- **[likely]** Widely reported by developers, not promised by Apple.
- **[unsure]** My best guess. We check it before relying on it.

## 1. What we're building

The same Twrlzy the kids use today (who's watching, the game, pick 1 of 3, the dose, open the show), rebuilt as a native iPad app so it can do what a web page can't:

1. Unlock only the chosen streaming app, only for the dose, then lock it again.
2. Open each service reliably.
3. Warn "5 minutes left" while the kid is inside Netflix or Disney+.
4. Keep settings in sync between the two iPads.
5. Keep the TMDB key on a server, never on an iPad.
6. Haptics where the hardware has them (see section 7: most iPads don't).

The web app stays live and keeps working. The native app is a separate project. It can be a new repo or a `native/` folder; Pat decides (section 14).

## 2. Stack (same as Chrp)

- Expo (latest SDK at the time we start) + React Native + TypeScript, built with EAS Build and sent to TestFlight with EAS Submit.
- Firebase: Auth, Firestore, Cloud Functions, App Check.
- Expo modules: `expo-notifications` (5 minute alert), `expo-haptics`, `expo-linking` and `expo-web-browser` (opening services), `expo-secure-store` (PIN hash), `expo-localization` (EN/FR).
- Screen Time: a library that wraps Apple's Swift APIs and adds the extensions through an Expo config plugin. Two candidates **[likely]**:
  - `react-native-device-activity` (Kingstinct): most used, has an Expo config plugin, needs a custom dev build (not Expo Go).
  - `expo-app-blocker`: smaller, newer.
  - If neither fits, we write a small Expo native module in Swift ourselves. That adds about 3 days.
- Reuse: the `CORE` section of `index.html` (catalog, eligibility, dose, daily state, setup-link encode/decode) has no DOM in it. We copy it into `core.ts` and add types. `dev/core_test.js` becomes a Jest test so both apps give the same answers.

The Screen Time parts need a real iPad. The simulator can't do them. **[likely]**

## 3. Screen Time: how "unlock only the chosen app" works

Apple gives us three frameworks **[sure]**:

| Framework | What it does for us |
|---|---|
| FamilyControls | Asks permission, and shows Apple's app picker (`FamilyActivityPicker`). |
| ManagedSettings | Puts a "shield" (lock screen) over chosen apps, and takes it off. |
| DeviceActivity | Runs a small extension in the background when a time window starts, is about to end, ends, or when the kid has used an app for N minutes. |

### 3.1 Permission (once per iPad)

- The app asks for **child** authorization (`.child`). A parent approves with their Apple Account password on the kid's iPad. This only works if the kid's iPad is signed in to a child Apple Account in Pat's Family Sharing group. **[sure]**
- With child authorization the kid can't delete Twrlzy or turn the permission off without the parent. **[sure]**
- If the kids' iPads aren't child accounts, we use `.individual` authorization. That works, but anyone who knows the iPad passcode can remove it. **[sure]** Open question for Pat (section 14).

### 3.2 Telling the app which icon is Netflix (once per iPad)

Apple never tells an app which apps are installed. The picker gives back opaque "tokens" and we can't read "this is Netflix" from them. **[sure]**

So in the grown-up area, for each service that's on, there's a row: "Netflix: choose the Netflix app". The parent taps it, Apple's picker opens, they tick Netflix only, and we save that token under `netflix`. We repeat for Disney+, Prime Video, Crave, YouTube, Apple TV, Tubi.

- Tokens belong to that iPad. We do this setup on each iPad, and we don't sync tokens between iPads. **[unsure]** Assume so until we test it.
- Optional extra row: "Also keep these locked" (Safari, other video apps, games), so the kids can't get around it through YouTube in Safari.

### 3.3 Day to day

1. **Normal state:** every streaming app from 3.2 is shielded. Twrlzy itself is never shielded.
2. **Kid taps "Let's watch":**
   - Twrlzy removes the shield from that one app, for example Disney+.
   - It starts a DeviceActivity window from now to a backstop end (see below).
   - It adds a usage event: "Disney+ used for 42 minutes" (the dose).
   - It opens the show (section 4).
3. **5 minutes before the end:** the extension sends a local notification, "5 minutes left" in the kid's language (section 6).
4. **Dose reached:** the extension puts the shield back on Disney+. The kid sees our own shield screen (Nova, "All done! See you tomorrow"), designed in a Shield Configuration extension. **[sure]** that the shield screen can be customized. Its buttons can only close the shield. They can't open Twrlzy. **[likely]**
5. **Backstop:** the window has a hard end, for example the dose plus 30 minutes, or bedtime. When it ends, everything locks even if the usage event never fired.

### 3.4 Usage time or clock time?

- **Usage time (recommended):** pausing for a snack doesn't use up the dose. This relies on usage events, and developers report they sometimes fire early, even right away. **[likely]** We guard against that: we ignore an event that fires before 80% of the clock time has passed, and the backstop always applies.
- **Clock time:** simpler and very predictable, but a pause still counts.
- Pat chooses. We can also start with clock time and switch.

### 3.5 Known limits

- A DeviceActivity window must be at least 15 minutes long. **[likely]** (Apple's error text suggests the workaround.) For short doses (2 episodes of a 7 minute show = 14 minutes), we set a 15 minute window and use the warning callback as the real end.
- Unlocking Disney+ unlocks all of Disney+, not just Bluey. Apple has no "this show only" option. **[sure]** Parents set Disney+ profiles and ratings inside Disney+ as they do today.
- The background extension has a very small memory limit. It only reads the App Group and changes shields; it never calls the network. **[likely]**
- Data from the Screen Time APIs must stay on the device. We don't send it to Firebase. **[sure]** that Apple restricts this, see section 10.

### 3.6 iOS extensions we ship (each needs the entitlement, section 5)

1. The app itself.
2. Device Activity Monitor extension (re-locks, sends the 5 minute alert).
3. Shield Configuration extension (the Nova "all done" screen).
4. Shield Action extension (what the shield buttons do). Optional; we can skip it in v1.

They share data through an App Group (`group.<bundle ID>`).

## 4. Opening each service

On iOS a normal `https://` link opens the streaming app if it's installed and the service supports universal links. **[sure]** for the mechanism. Each service is **[likely]** or **[unsure]** until `linktest` passes on a real iPad. The native app gets its own "link test" screen in the grown-up area, like `linktest.html` today.

| Service | How native opens it | Notes |
|---|---|---|
| Netflix | `https://www.netflix.com/title/<id>` | Universal link, opens the app. **[likely]** |
| Disney+ | `https://www.disneyplus.com/...` | Universal link. Disney+ changes URL formats from time to time; fallback is the app home. **[likely]** |
| Prime Video | `https://app.primevideo.com/...` | Same rule as the web app (decision already made). **[likely]** |
| Crave | `https://www.crave.ca/...` | Whether it opens the app or Safari is **[unsure]**. Test it. |
| YouTube | `https://www.youtube.com/...` | Opens the YouTube app. **[sure]** Channel IDs in the catalog are still unverified. |
| Apple TV | `https://tv.apple.com/ca/...` | Opens the TV app. **[likely]** |
| Tubi | `https://tubitv.com/...` | **[unsure]** |
| **Télé-Québec** | Inside Twrlzy, in an in-app browser locked to `telequebec.tv` | See below. |
| TFO | Same as Télé-Québec, locked to `tfo.org` | |

**Télé-Québec and TFO.** Today they open in Safari. Natively, if we unlocked Safari for them, the kid would get the whole web. Two better options:

- **A (recommended):** play them inside Twrlzy in a web view that only allows `telequebec.tv` (or `tfo.org`) and the video player domains it needs. The timer is Twrlzy's own, so we don't need a shield, and we show our own "5 minutes left" and "all done" screens. **[likely]** works; Télé-Québec's video player may need extra domains, so we test it.
- **B:** Télé-Québec has its own iPad app (free, French). **[likely]** If it supports universal links, it works like Netflix with a shield. **[unsure]** whether it does. Télé-Québec is folding Squat into "Télé-Québec Jeunesse" and launched a new platform in 2026, so links may move. **[likely]**

The app checks that a service's app is installed before unlocking (`canOpenURL` with the services listed in `LSApplicationQueriesSchemes`). If it isn't installed, the kid sees "Ask a grown-up to install Disney+" instead of a dead link.

## 5. The Apple entitlement

- Name: **Family Controls**, key `com.apple.developer.family-controls`. **[sure]**
- **Development** version: Xcode/EAS can add it right away for development builds installed on registered iPads. **[sure]** This means we can build and test Screen Time on the kids' iPads before Apple approves anything.
- **Distribution** version (needed for TestFlight and the App Store): request it at developer.apple.com > Account > the Family Controls (Distribution) request form. Requested per bundle ID, so **one request for the app and one per extension** (3 or 4 requests). **[likely]**
- How long: no official timeline. Developers on Apple's forums report anywhere from a few days to 4 to 5 weeks, and some say the extension requests took longer than the app one. In May 2026 there was a reported backlog. **[likely]** Plan for **2 to 5 weeks**. That's why Pat requests it tonight and phase 1 doesn't need it.
- If it stalls past 3 weeks: open a Developer Technical Support request with the Team ID and the bundle IDs (Apple staff on the forums suggest this). **[likely]**

## 6. "5 minutes left"

- **Shows in a streaming app:** the Device Activity Monitor extension posts a local notification when the warning callback fires ("5 minutes left! Find a good place to stop." / "Plus que 5 minutes!"). It appears as a banner on top of Netflix. **[likely]** that extensions can post local notifications; it's a common pattern.
- **Clock-time fallback:** when the kid taps "Let's watch", the app also schedules a normal local notification at dose minus 5 minutes. If usage mode is on and the kid paused, we cancel and reschedule it when they come back to Twrlzy.
- **Télé-Québec / TFO (in-app):** our own in-app banner with Nova and a soft sound.
- Notification permission is asked during parent setup, not during kid flow.
- Tapping the banner opens Twrlzy on the "almost done" screen.

## 7. Haptics

- `expo-haptics` on: the wheel ticks, the reveal, "Let's watch", the 2 second Parents hold finishing.
- **Important:** iPads don't have the Taptic Engine that iPhones have, so taps don't vibrate on most or all iPads. **[likely]** On the kids' iPads, haptics will probably do nothing. They work on an iPhone. We still wire them in (it's cheap) but don't count on them; sound and motion carry the feedback on iPad.

## 8. Syncing the two iPads (Firebase)

What syncs: settings (kids, ages, games, services on/off, show list, hidden shows), today's state per kid (watched, spins, bonus) so both iPads agree, and the PIN **hash**. What doesn't sync: Screen Time tokens (per iPad, 3.2), the TMDB key (server only).

- **Firestore layout:** `families/{familyId}` (settings, members), `families/{familyId}/days/{date}_{kidId}` (daily state). Same shapes as today's `popcorn.settings` and `popcorn.state.*` so `core.ts` works unchanged.
- **Joining, no passwords for the kids:** each iPad signs in to Firebase anonymously. The first iPad creates the family. The second iPad joins by scanning a QR code or pasting a one-time code (like today's setup link), confirmed with the parent PIN. Firestore security rules only allow devices listed in the family's members.
- **Offline:** Firestore keeps working offline and catches up later. **[sure]**
- **Moving from the web app:** the native app accepts the existing setup link, so Pat's current settings come over in one paste. Same `hashPin` salt so the PIN keeps working.
- **Privacy:** kids' first names would be stored in Firestore. Option: parents use nicknames. Region: `northamerica-northeast1` (Montréal), which helps with Québec's Law 25. **[likely]** It's a reasonable choice; this isn't legal advice.
- Which Firebase SDK: whichever Chrp uses (React Native Firebase or the Firebase JS SDK), so Pat sees the same setup. **[unsure]** which one Chrp uses; Pat to tell me.

## 9. TMDB key on a server function

- A Cloud Function `showArt` takes a show ID and returns poster and backdrop URLs. The TMDB key is stored as a Firebase secret (Pat types it into Terminal; it never goes in chat or code).
- Results are cached in Firestore for 30 days, so TMDB is called a handful of times per show, not on every spin.
- App Check (App Attest on iOS) so only our app can call the function. Plus a simple per-family rate limit.
- Cloud Functions and secrets need Firebase's pay-as-you-go **Blaze** plan. **[sure]** Expected cost for two iPads: about $0 a month; set a budget alert at $5. **[likely]**
- TMDB terms: the free key is for non-commercial use, and the app must show TMDB attribution ("This product uses the TMDB API but is not endorsed or certified by TMDB") in the grown-up area. **[sure]** A free, family-only app is probably fine. A public App Store release is a grey area; if we go public, read the terms again or ask TMDB. **[unsure]**

## 10. App Store rules for kids' apps

- **Don't put Twrlzy in the Kids category.** Kids-category apps can't send kids to other apps or websites without a parental gate on every link, and can't use most third-party SDKs. **[sure]** Opening Netflix is our whole job, so the Kids category doesn't fit. Screen-time and parental-control apps usually ship in **Lifestyle** or **Utilities** with a 4+ age rating. **[likely]**
- Even outside the Kids category, a kid-facing app must follow the privacy and kids' data rules (COPPA in the US, Law 25 in Québec): no ads, no third-party analytics, a privacy policy URL, and accurate App Privacy answers in App Store Connect. **[sure]**
- Screen Time data: Apple's guidelines say apps using these APIs can't send that data off the device or use it for anything other than the feature. **[sure]** for the rule. **[unsure]** of the exact guideline number today; check before submission.
- Trademarks: text labels like "Netflix" are fine inside the app as they are on the web. Avoid service names in the app name, icon and screenshots. **[likely]**
- Show artwork in App Store screenshots: don't. Use our own drawings.
- **TestFlight only?** If Twrlzy stays family-only, we never need App Store review. TestFlight builds expire after 90 days, so we upload a fresh build at least every 90 days. **[sure]** TestFlight on a child Apple Account: **[unsure]**. The fallback is to install development builds directly on the registered iPads.

## 11. Android: what it can and can't do

- **Can:** open each service reliably (Android intents are good at this), the 5 minute notification, haptics on most phones and tablets, Firebase sync, the TMDB function. The Expo code is mostly shared.
- **Can't, easily:** Android has no public equivalent of Apple's shields for a normal app. Google's Family Link does it, but other apps can't control Family Link. **[sure]** The options are:
  - Usage Access + an Accessibility Service that sends the kid home when a locked app opens. Works, but Google Play requires a special declaration and review for both. **[likely]**
  - Device owner mode (`setPackagesSuspended`), which means wiping and setting up the tablet specially. Too heavy for a family app.
- **Recommendation:** no Android until the iPad app is done and only if someone needs it. If we do it, start with "picker + deep links + 5 minute notification" without locking.

## 12. Phases

Effort is in focused working days for me writing code, plus Pat's testing time on the iPads. It assumes Pat's accounts are ready.

### Phase 0: accounts and requests (tonight, Pat)
See TODO tonight in the chat. The entitlement request is the long pole, so it goes in first.

### Phase 1: smallest useful TestFlight build, no Screen Time (4 to 6 days)
- Expo project, `core.ts` from the web catalog with its tests, kid flow: who's watching, the 3 games, reveal, pick 1 of 3, dose, Let's watch.
- Grown-up area: 2 second hold with a filling ring, PIN, math question first time, settings tabs.
- Import from the existing setup link. Local storage only (no Firebase yet).
- Deep links + native link-test screen. Télé-Québec/TFO in a locked in-app browser.
- "5 minutes left" as a scheduled local notification (clock time).
- Haptics wired in (for iPhone).
- **Why first:** no special entitlement needed, so it goes to TestFlight right away and the kids can use it while Apple reviews the entitlement.
- **Done when:** both kids use it for a few days and the links pass on both iPads.

### Phase 2: Screen Time on development builds (5 to 8 days)
- Screen Time library + extensions, authorization, the per-service app picker, shield on/off, usage events, backstop window, Nova shield screen, 5 minute alert from the extension.
- Installed as a development build on the registered iPads (no approval needed yet).
- **Done when:** a full week of evenings works: unlock, warning, re-lock, no stuck-open or stuck-locked apps.

### Phase 3: Firebase (3 to 5 days)
- Anonymous auth, family create/join by QR or code, Firestore sync, security rules, offline.
- `showArt` function with the TMDB secret, cache, App Check.
- **Done when:** a change on one iPad shows on the other within a few seconds, and pictures load without any key on the iPads.

### Phase 4: Screen Time on TestFlight (1 to 2 days, after Apple approves)
- Switch the extensions to the distribution entitlement, build, TestFlight.

### Phase 5: polish and decide on the App Store (later)
- Port the Claude Design polish from the web app, accessibility pass, French copy review.
- Decision: stay family-only on TestFlight, or go public (privacy policy, App Privacy answers, TMDB terms, review).

**Total to a full TestFlight build: about 13 to 21 days of work,** spread around Apple's approval wait.

## 13. Risks

| Risk | Impact | What we do |
|---|---|---|
| Entitlement approval is slow (weeks) | Screen Time can't reach TestFlight | Request tonight; phases 1 to 3 don't need it; development builds in the meantime. |
| Usage events fire early or late **[likely]** | Kid gets locked out early or gets extra time | Sanity check on clock time, backstop window, parent "unlock 10 more minutes" button. |
| A streaming app's universal link stops opening the app | Kid lands in Safari | Link-test screen, fallback to app home, catalog fix in the next build. |
| Expo Screen Time library doesn't cover what we need | Delay | Fall back to our own small Swift module (+3 days). |
| Kids' iPads aren't child accounts in Family Sharing | Weaker lock (`.individual`) | Pat checks tonight; switch them to child accounts if possible. |
| Apps stay unlocked if the extension crashes | Free-for-all evening | Backstop window, re-lock on every Twrlzy launch, re-lock at a fixed bedtime. |
| TestFlight 90 day expiry | App stops opening | Calendar reminder; EAS build every month. |
| TMDB terms if we go public | Need a commercial agreement or no pictures | Stay family-only, or use our own drawings. |

## 14. Questions for Pat before phase 1

1. Are the kids' iPads signed in with child Apple Accounts in your Family Sharing group? Which iPad models and iPadOS version?
2. Usage time or clock time for the dose (3.4)?
3. Télé-Québec: in-app browser (recommended) or their app?
4. New repo for the native app, or a `native/` folder in this one?
5. Which Firebase SDK and bundle ID prefix does Chrp use? We'll match them.
6. Family-only on TestFlight for now, or plan for a public App Store release?
