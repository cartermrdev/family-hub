# iOS phone options: PWA vs. Expo vs. SwiftUI

Research for issue #5. Informs the later decision "Phone experience: PWA or native?" (#8). Researched 2026-10-01 against primary sources; current shipping iOS is 27 (Safari 27).

## Answer

A Home Screen web app (PWA) meets the v1 bar: Web Push works on iOS/iPadOS 16.4+ for web apps added to the Home Screen, needs **no** Apple Developer Program membership, and shares nearly all code with the web Wall Display. What it cannot do is widgets or Lock Screen widgets, and push breaks if the web app is not installed or if the service worker fails to show a notification (Declarative Web Push on iOS 18.4+ removes the second risk). **Recommendation (not a decision):** start with a PWA for v1. Move to Expo if widgets, Lock Screen widgets or Live Activities become must-haves; that costs $99/yr plus TestFlight or ad hoc builds. SwiftUI only makes sense if the developer wants a native-only phone app and accepts maintaining two codebases.

## Comparison

| | PWA (Home Screen web app) | Expo / React Native | SwiftUI native |
|---|---|---|---|
| Push for Messages / due Chores | Yes, iOS 16.4+, only once added to the Home Screen; permission must come from a tap [1][2] | Yes, through APNs (Expo push service is free, or APNs directly) [6][7] | Yes, through APNs (needs paid membership) [11] |
| Push reliability caveats | Every push must show a visible notification or Safari revokes permission [2]; Declarative Web Push (18.4+) removes that penalty [3] | Expo: "at-least-once" to APNs, "rare" losses; then Apple's rules apply [7] | Plain APNs; Apple's delivery rules apply |
| App icon badge | Yes, Badging API [1][2] | Yes | Yes |
| Home Screen / Lock Screen widgets, Live Activities | **No.** No web API exists for these (no Apple docs describe one) | Yes, using `expo-widgets` (widget UI written with `@expo/ui/swift-ui`, runs in a limited runtime) or `expo-apple-targets` with Swift [9][10] | Yes, WidgetKit is SwiftUI-native [8] |
| Offline | Service worker + IndexedDB; Home Screen apps keep their own 7-day ITP counter and are likely to be granted persistent storage [4][5] | Full native storage | Full native storage |
| Apple Developer Program ($99/yr) | **Not needed** [2] | Needed for push and for installing on Members' phones [6][11][12] | Needed for push and for installing on Members' phones [11][12] |
| Getting it onto 4 iPhones | Open a URL, then Share → Add to Home Screen [13] | TestFlight internal testing (up to 100, no review, builds expire after 90 days) or ad hoc via EAS (registered UDIDs, 100 iPhones/yr) [14][15][16] | Same as Expo, using Xcode |
| Code shared with the web Wall Display | ~100% (same app, separate layouts) | High if the Wall Display is also Expo web (React Native for Web) [17]; less if it's a plain React/DOM app | ~0% for UI; only the backend/API is shared |
| Single-dev effort | Lowest. One codebase, deploy updates instantly | Medium. Native builds, credentials, EAS (free tier: 15 iOS builds/month, low priority) [18], refresh TestFlight every 90 days | Highest. Swift + a separate web app for the Wall Display |
| Needs a Mac | No | No (EAS builds in the cloud) | Yes (Xcode) |

## Details

### Push notifications

**PWA.** Apple: "Add web push to Home Screen web apps in iOS 16.4 or later", using the standard Push API, Notifications API, Badging API and service workers. "You don't need to join the Apple Developer Program to send web push notifications." [2] Rules that matter:
- The user has to add the site to the Home Screen first. Permission must be requested from a user gesture (a button tap) [1][2]. Onboarding therefore needs a step-by-step "Add to Home Screen, then tap Enable notifications" flow.
- "Safari doesn't support invisible push notifications… If you don't [present them], Safari revokes the push notification permission for your site." [2] WebKit puts it more bluntly: "if an event handler doesn't show the user visible notification for any reason we revoke its push subscription." [3] A bug, or a network failure inside the service worker, can silently unsubscribe a Member.
- **Declarative Web Push** (iOS/iPadOS 18.4+, Home Screen web apps) sends the notification content in the payload. A service worker can still change it, but "there is no penalty for service workers failing to display a notification" [3]. Using it largely removes the revocation risk. It is backward compatible with standard Web Push [3].
- The server sends standard RFC 8030 + VAPID requests to `*.push.apple.com`. TTL can be up to 30 days and the push service stores only a limited number while the device is offline. `Urgency: high` asks for immediate delivery. Payload limit is 4 KB [2]. Libraries such as `web-push` handle this.
- Notifications obey Focus modes, and Focus settings sync across devices through the manifest `id` [1].
- Since iOS 26, every site added to the Home Screen opens as a web app by default, manifest or not [19]. This makes "install" simpler. Push still needs the Home Screen web app, not a Safari tab.
- The WWDC26 "What's new in WebKit for Safari 27" session announces no changes to Web Push or Home Screen web apps [20]. Treat the iOS 16.4/18.4 behavior above as current, but test it on iOS 27 devices (**version-dependent**).

**Expo.** Needs a physical device, a **paid Apple Developer account** and a development build. Push does not work in Expo Go on iOS. EAS CLI generates the APNs key [6]. The Expo push service is free, capped at 600 notifications/sec per project, and aims for "at-least-once" delivery to Apple. Native device tokens are available if you want to call APNs directly [7].

**SwiftUI.** Standard APNs with UserNotifications. The push capability needs Certificates, Identifiers & Profiles, which only paid members get [11].

There is no Apple documentation comparing delivery reliability of Web Push and native APNs. Both use APNs (web push endpoints are on `push.apple.com` [2]). Any claim that one is "more reliable" in practice is **undocumented**. The documented difference is the revocation rule for web push [2][3].

### Widgets and Lock Screen

WidgetKit covers Home Screen, Lock Screen and StandBy widgets plus Live Activities (via ActivityKit). Widgets are SwiftUI app extensions [8]. A Home Screen web app has no way to provide them; WebKit and Apple docs describe no web widget API (**absence of documentation, not an explicit statement**). The PWA gets an app icon badge [1] and notifications, nothing more.

Expo: the official `expo-widgets` supports Home Screen sizes, Lock Screen accessory families and Live Activities. Its widget code "runs in an isolated runtime and can only use `@expo/ui/swift-ui` components, with no React hooks, app state, or asynchronous work", and it does not work in Expo Go [9]. The community plugin `expo-apple-targets` lets you write the widget in Swift inside an Expo project [10].

### Offline

PWA: service worker caching + IndexedDB. Safari's 7-day deletion of script-writable storage counts days of use, and "Web applications added to the home screen are not part of Safari and thus have their own counter of days of use" [5], so an app opened regularly keeps its data. Home Screen web apps get the same quota as the browser. WebKit grants persistent storage "based on heuristics like whether the website is opened as a Home Screen Web App" [4], a heuristic and not a guarantee. For lists and Chores, an offline read cache with queued writes is enough. Native apps have no storage limits worth mentioning.

### Distribution (native options)

- **Free Apple Account**: on-device testing from Xcode only [11]. Profiles expire after 7 days, max 3 devices, and no push (push needs CI&P) [11][21]. Not workable for a family.
- **$99/yr Apple Developer Program** [12]: gives TestFlight, ad hoc and the App Store [11].
- **TestFlight internal testing**: up to 100 internal testers. They must be App Store Connect users with a role (Account Holder, Admin, App Manager, Developer or Marketing), so each family member needs an Apple Account added to the team. No beta review. Builds work for 90 days [14][15]. The Teens' Apple Accounts must be able to accept the App Store Connect invite (**not verified for child/managed accounts**; the Apple doc says Managed Apple Accounts in reserved domains cannot test).
- **TestFlight external testing**: up to 10,000 testers, but the first build needs Beta App Review [15].
- **Ad hoc**: up to 100 iPhones per membership year, with UDIDs registered. No App Store Connect and no review [16]. EAS "internal distribution" automates this (`eas device:create`), and you rebuild whenever a device is added [22].
- **Unlisted App Store distribution**: possible, but needs a full App Review submission and an approval request [23]. Overkill for 4 people.
- There is no "family-only" channel. In practice a family uses TestFlight internal testing (renew every 90 days) or ad hoc builds (valid about a year, as long as the profile is).

### Code sharing with the Wall Display

The Wall Display is likely a web app in a kiosk browser. A PWA reuses the whole codebase (data layer, auth, components) with a phone layout. With Expo, sharing is high only if the Wall Display is also an Expo web app (React Native for Web, "universal components") [17], which affects the Wall Display stack decision. If the Wall Display is plain React DOM, you share the API client, types and logic, not the UI. With SwiftUI, only the backend is shared.

### Single-developer effort (judgment, not sourced)

- PWA: one app, instant updates, no Apple account, no Mac. Extra work: Home Screen onboarding, the push subscription lifecycle, re-subscribing after revocation.
- Expo: TypeScript stays, but you add native build pipelines, credentials, EAS quota (15 iOS builds/month on the free tier, may queue 90+ minutes [18]), TestFlight refresh every 90 days, and an upgrade with each Expo SDK release.
- SwiftUI: best native quality, but a second language and UI stack, a Mac and Xcode, and nothing shared with the Wall Display.

## Risks

1. **Silent loss of push subscriptions (PWA).** Standard Web Push revokes the subscription if a push doesn't show a notification [2][3]. Mitigation: use Declarative Web Push (iOS 18.4+), always call `showNotification`, track subscriptions on the server, and show an in-app "notifications are off" banner.
2. **Install friction (PWA).** Push only works after Add to Home Screen. Four motivated users can get past this, but each phone needs a one-time setup.
3. **No widgets (PWA).** If "glance at today's Chores on the Lock Screen" turns out to matter, the PWA can't do it. That would force Expo or SwiftUI later. The backend (Web Push vs. APNs tokens) would need a second sender, which is an argument for building the notification sender behind an interface from day one.
4. **Platform policy (PWA).** Apple briefly removed Home Screen web apps in the EU in the iOS 17.4 betas, then backed down [24]. Not relevant to a US household today, but it shows Apple can change PWA support.
5. **Apple-side expiry (native).** TestFlight builds expire after 90 days [14]. Ad hoc profiles and the membership are yearly, and a lapsed membership stops push and distribution.
6. **Teen accounts (native/TestFlight).** Whether a child Apple Account can be an App Store Connect user is **unverified**. Ad hoc avoids the question.
7. **Version drift.** This is all documented for iOS 16.4 / 18.4 / 26. The iOS 27 behavior was not announced as changed [20] but has not been tested here.

## Sources

1. WebKit, "Web Push for Web Apps on iOS and iPadOS" — https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
2. Apple, "Sending web push notifications in web apps and browsers" — https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers
3. WebKit, "Meet Declarative Web Push" — https://webkit.org/blog/16535/meet-declarative-web-push/
4. WebKit, "Updates to Storage Policy" — https://webkit.org/blog/14403/storage-policy-updates/
5. WebKit, "Full Third-Party Cookie Blocking and More" (7-day cap, Home Screen exception) — https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/
6. Expo, "Expo push notifications setup" — https://docs.expo.dev/push-notifications/push-notifications-setup/
7. Expo, "Push notifications FAQ" — https://docs.expo.dev/push-notifications/faq/
8. Apple, WidgetKit — https://developer.apple.com/documentation/widgetkit
9. Expo, `expo-widgets` — https://docs.expo.dev/versions/latest/sdk/widgets/
10. `expo-apple-targets` (community, Evan Bacon) — https://github.com/EvanBacon/expo-apple-targets
11. Apple, "Choosing a Membership" (compare memberships) — https://developer.apple.com/support/compare-memberships/
12. Apple, "Program enrollment" (fees) — https://developer.apple.com/help/account/membership/program-enrollment/
13. WebKit, Safari 26 beta (Home Screen web apps) — https://webkit.org/blog/16993/news-from-wwdc25-web-technology-coming-this-fall-in-safari-26-beta/
14. Apple, "Add internal testers" — https://developer.apple.com/help/app-store-connect/test-a-beta-version/add-internal-testers
15. Apple, TestFlight — https://developer.apple.com/testflight/
16. Apple, "Devices overview" (100 devices per product family per membership year; ad hoc) — https://developer.apple.com/help/account/devices/devices-overview
17. Expo, "Develop websites with Expo" — https://docs.expo.dev/workflow/web/
18. Expo pricing — https://expo.dev/pricing
19. WebKit, Safari 26 beta: "By default, every website added to the Home Screen opens as a web app" — https://webkit.org/blog/16993/news-from-wwdc25-web-technology-coming-this-fall-in-safari-26-beta/
20. Apple, WWDC26 "What's new in WebKit for Safari 27" — https://developer.apple.com/videos/play/wwdc2026/204/
21. Apple Developer Forums, free provisioning limits (7-day profiles, 3 devices) — https://developer.apple.com/forums/thread/761325 (forum thread, secondary; limits **not in formal docs**)
22. Expo, "Internal distribution" — https://docs.expo.dev/build/internal-distribution/
23. Apple, "Unlisted app distribution" — https://developer.apple.com/support/unlisted-app-distribution/
24. EU Home Screen web app reversal (secondary press; Apple's DMA update page is the primary) — https://www.macworld.com/article/2238869/ios-17-4-home-screen-web-apps-digital-markets-act-eu.html
