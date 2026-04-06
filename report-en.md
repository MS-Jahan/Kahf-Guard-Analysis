# Kahf Guard: Fact-Checking the Viral Allegations — What the Code Actually Shows

Recently, a viral Facebook post accused Kahf Guard of being a "remote-controlled trojan" that "sells user data to Facebook" and conducts "extreme surveillance."

As an analyst, I decompiled Kahf Guard's APK (v4.6.188) using JADX 1.5.5 and performed a static analysis. I also checked what each analytics SDK actually collects according to its official documentation. The viral post gets several specifics wrong, but not everything is baseless — some concerns are legitimate.

Here's a balanced, evidence-based breakdown.

---

## Part 1: The Analytics SDKs — What They Actually Collect

The viral claim: "The app contains Facebook SDK, PostHog, and Google Analytics, which means they sell your data to Meta."

### What the code shows:

**Three analytics tools are confirmed** in the decompiled code:
- **Facebook SDK** (`KahfGuardApp.m4516b`): initialized with `AdvertiserID: enabled` and `AutoLogEvents: enabled`
- **PostHog** (`KahfGuardApp.m4518d`): self-hosted at `https://p.kahf.co.uk` with session replay capability
- **Firebase Analytics + Crashlytics** (`KahfGuardApp.m4524j`): standard crash and usage reporting

> **Critical Clarification: SDK Tracking Scope vs. Accessibility Monitoring**
>
> A common source of confusion is whether these SDKs can track what you do in OTHER apps (Chrome, WhatsApp, Instagram, etc.). **They cannot.** Here's why:
>
> **Android's Application Sandbox** ([Android Open Source Project](https://source.android.com/docs/security/app-sandbox)) assigns a unique user ID (UID) to each app and runs it in its own process with kernel-level isolation. An SDK embedded in Kahf Guard can only access Kahf Guard's own process, views, and screens. It **cannot** see your Chrome tabs, WhatsApp chats, or Instagram feeds.
>
> - **PostHog Session Replay** captures the **host app's own view hierarchy** — i.e., Kahf Guard's settings screens, onboarding flows, and dashboard. It uses native Android APIs to "grab the view hierarchy state when the screen is drawn" ([PostHog Mobile Docs](https://posthog.com/docs/session-replay/mobile)) — referring to Kahf Guard's own screens only.
> - **Facebook SDK** logs events that happen **inside Kahf Guard** (app launch, screen views within the app). It cannot see activity in other apps.
> - **Firebase Analytics** tracks **Kahf Guard's own screens and lifecycle** — when the user opens Kahf Guard, which settings page they view, etc.
>
> **The cross-app monitoring** (reading URLs from browsers, monitoring WhatsApp/Telegram/Instagram screens) is done by Kahf Guard's **own Accessibility Service code** (`KahfBlockerService`), NOT by any SDK. The Accessibility Service is a special Android API designed to cross app boundaries for accessibility features — which content blockers also use. This is Kahf Guard's custom blocking code, not third-party analytics.
>
> **Summary:**
>
> | What does the tracking | Can see Kahf Guard's own screens | Can see other apps' screens |
> |---|---|---|
> | Facebook SDK / PostHog / Firebase | **Yes** | **No** — blocked by Android sandbox |
> | KahfBlockerService (Accessibility) | Yes | **Yes** — by design, for blocking |

### What each SDK actually collects (per official docs + decompiled code):

**All data below is scoped to Kahf Guard's OWN app screens only** (settings, onboarding, dashboard, etc.), not your activity in other apps:

#### Facebook SDK (Meta App Events)

The Facebook SDK is primarily used for ad attribution (tracking if a Facebook ad led to an app install) and logging user actions to build better ad profiles. When integrated, it communicates directly with Meta's servers.

**Collected automatically** ([Meta Developer Docs](https://developers.facebook.com/docs/app-events/getting-started-app-events-android/)):
- **Google Advertising ID (GAID)** — a persistent, device-level identifier that enables cross-app tracking on Android and allows Meta to link app activity to the user's overarching Facebook profile
- **Auto-logged events**: app install, app launch, and standard events like "Add to Cart" or "Purchase"
- **Device metadata** sent with every event: OS version, hardware model, mobile carrier, IP address, timezone, remaining storage space, screen resolution, language
- **Advanced Matching** ([Meta Docs](https://developers.facebook.com/docs/app-events/advanced-matching)): the code includes `UserDataStore` integration (`p130fc.RunnableC4208c`) which can pass **hashed versions** of email address, phone number, first name, and gender directly to Meta to improve ad targeting

**What this means in Kahf Guard's context:** The code initializes the SDK with `AdvertiserID: enabled` and `AutoLogEvents: enabled` (`KahfGuardApp.m4516b`). This is not a data "sale" — but data IS sent to Meta's servers for ad targeting and attribution. For users who installed this app specifically to escape Meta's tracking ecosystem, this is deeply ironic.

---

#### Google Analytics for Firebase

Firebase Analytics is the industry standard for tracking how users interact with a mobile app.

**Collected automatically** ([Google Analytics Help](https://support.google.com/analytics/answer/9234069), [Firebase Docs](https://support.google.com/firebase/answer/6318039)):
- **Events**: `first_open`, `session_start`, `user_engagement`, `screen_view`, `app_update`, `app_remove`, `app_exception` (crashes), and more — all without any additional code
- **Parameters sent with every event**: `app_version`, `firebase_screen_class`, `firebase_screen_id`
- **User properties** (auto-collected): device model, OS version, language, geographic info (country-level derived from IP)
- **User engagement**: session count, session duration, time spent on each screen
- Developers can also configure up to 25 custom user properties and 500 custom event types

---

#### Firebase Crashlytics

Crashlytics is a lightweight real-time crash reporter that helps developers track, prioritize, and fix stability issues.

**Collected automatically** ([Firebase Crashlytics Docs](https://firebase.google.com/docs/crashlytics/data-collection)):
- **Crash state**: Stack traces — the exact lines of code that caused the app to crash
- **Device state at crash**: free RAM, free disk space, device orientation, whether app was in foreground or background
- **Device info**: OS version, device model, jailbreak/root status
- **Installation UUID**: a unique identifier for the specific app installation, to track if the same user is experiencing crashes repeatedly
- **Custom logs/breadcrumbs**: developers can leave traces like "User opened settings," "User tapped save" alongside crash reports

---

#### Google Play Services

This is not just an SDK — it's a core part of the Android OS. The app uses it to hook into Google's APIs.

**Collected automatically** ([Google Play Services](https://support.google.com/googleplay/answer/9023446)):
- **Device information**: hardware model, OS version, device identifiers (including the Google Advertising ID), mobile network info
- **Diagnostic & usage data**: app crash logs, performance metrics, battery usage statistics
- **Security telemetry**: device security state (rooted/compromised check via Play Integrity API / SafetyNet)

---

#### PostHog (Self-Hosted at `p.kahf.co.uk`)

PostHog is an open-source product analytics suite. Unlike Google Analytics, which heavily relies on aggregate data, PostHog allows for granular, user-level tracking. Kahf Guard runs a self-hosted instance.

**Collected by default** ([PostHog Docs](https://posthog.com/docs/privacy/data-collected)):
- **Product analytics**: screen views (`captureScreenViews = true` by default), screen transitions, clicks, and custom events
- **Application lifecycle events**: `captureApplicationLifecycleEvents = true` — app open, close, background, foreground
- **Device properties**: OS, screen size, app version

**Session Replay** ([PostHog Session Replay Privacy](https://posthog.com/docs/session-replay/privacy)) — infrastructure present in code (`PostHogSessionReplayHandler`):
- Records **Kahf Guard's own app screens only** (settings, onboarding, etc.) — cannot see other apps due to Android sandbox isolation
- In wireframe mode (default): captures the **host app's view hierarchy** as a JSON wireframe — layout, positions, sizes, and text content of Kahf Guard's own UI elements ([PostHog Mobile Docs](https://posthog.com/docs/session-replay/mobile))
- In screenshot mode (if enabled): takes actual screenshots of Kahf Guard's own screens
- Can capture: touch/click coordinates within Kahf Guard, keyboard input state, screen transitions between Kahf Guard's pages
- **Masking defaults**:
  - Text inputs are masked by default (`maskAllTextInputs = true`)
  - Images are masked by default (`maskAllImages = true`)
  - Passwords are always masked regardless of config
- Whether session replay is actively recording for all users depends on **server-side PostHog configuration** — this cannot be confirmed through static analysis
- Even with masking enabled, the SDK still captures Kahf Guard's own screen structure, touch coordinates within the app, and navigation patterns

### Why some analytics are legitimate for this product:

Kahf Guard is a content-blocking and digital accountability app. For such a product to be effective, the developers need to know:
- **Is the blocking working?** — How many sites were blocked, which categories are most common, are users bypassing blocks?
- **Is the app reducing harmful usage?** — Is time spent on social media actually going down after installation? Are Reels/Shorts blocks effective?
- **Are users staying with the app?** — Retention rates, uninstall reasons, feature adoption
- **Is the app technically stable?** — Crashes, ANRs, device compatibility issues

Without this data, the developers cannot improve the blocking effectiveness or fix bugs. Firebase Analytics, Crashlytics, and even a self-hosted tool like PostHog are reasonable choices for answering these questions. The **quantity and type** of analytics is the debate — not whether analytics should exist at all.

### What the viral post gets wrong:
- "Selling data" implies a direct commercial transaction. The Facebook SDK sends telemetry to Meta for ad attribution — standard SDK behavior, not a data sale
- OpenReplay is NOT present — it's PostHog. The domain `openreplay.kahf.co.uk` does not exist in this build

### What the viral post gets right:
- Facebook SDK IS present in an app marketed as privacy-focused — this is a legitimate concern
- The app DOES send data to third-party analytics ecosystems, contradicting the CEO's "zero surveillance" and "no data collection" claims

---

## Part 2: Browser URL Reading — How the Blocking Works

**The viral claim:** The app "steals URLs from 17 browsers."

**What the code shows:** `KahfBlockerService` monitors **18 browser packages** (not 17) via Android's Accessibility Service — Chrome, Firefox, Brave, Tor, DuckDuckGo, Edge, Opera, Vivaldi, and others.

**Why this is necessary:** Kahf Guard is a content blocker. DNS-level blocking cannot see what happens inside HTTPS connections. To block specific pages (like Instagram Reels while allowing DMs), the app MUST read the URL from the browser's accessibility nodes. This is how all Android content blockers that operate at the app level work.

**Important scope note:** This cross-app URL reading is done by Kahf Guard's **own Accessibility Service code** (`KahfBlockerService`), NOT by Facebook SDK, PostHog, or Firebase. The URLs are used locally for the blocking decision. The code does not show URLs being logged or uploaded to analytics services. However, the capability to read all URLs from 18 browsers exists — users should be aware of it.

---

## Part 3: ALLOWLISTED_SIG and evaluateJavascript

### ALLOWLISTED_SIG

**The viral claim:** "Anyone can push malware through this signature."

**What the code shows:** `SignatureCheck` contains one hardcoded secondary signature. This is a standard dual-signing pattern used by apps distributed through multiple channels (Play Store, direct APK, alternative stores). It's also used by PairIP's anti-piracy verification.

**The claim is false.** Only someone with the corresponding private key can produce a valid signature. Without that key, nobody can push a malicious update.

### evaluateJavascript

**The viral claim:** "The server can run remote JavaScript — it's a backdoor."

**What the code shows:** `C5181p` injects JavaScript into WebViews for content filtering (e.g., removing Reels from Instagram Web). The JS strings come from the app's internal state machine, not directly from a remote server.

**The claim is unproven.** The code shows local WebView manipulation for blocking, not arbitrary remote code injection. However, the app does load some remote content in WebViews (like a YouTube proxy page), which is a property of all WebViews — not a Kahf-specific backdoor.

---

## Part 4: Social App Monitoring — Necessary but Broad

**The viral claim:** "Terrifying surveillance of WhatsApp, Telegram, Instagram."

**What the code shows:** The app monitors two sets of social apps:
- **Core set** (`KahfBlockerService.f8269P`): YouTube, Instagram, Facebook, Telegram, WhatsApp, WhatsApp Business
- **Extended set** (`KahfBlockerService.f8271R`): Twitter, TikTok, Snapchat, Reddit, Threads, Pinterest, Discord, Tumblr

**This IS how blocking works.** To block Instagram Reels but allow DMs, the app must distinguish between these screens — which requires reading accessibility nodes. To detect "risky words" in Telegram, it must read text from the interface.

**Important scope note:** Again, this cross-app monitoring is done by Kahf Guard's **own Accessibility Service**, not by any analytics SDK. The SDKs (Facebook, PostHog, Firebase) cannot see what you do in these apps.

**The legitimate concern:** The depth of monitoring goes beyond what DNS-only blocking would need. And the `GetChildDevicesQuery` GraphQL query sends the **full list of installed apps** (including system apps) to the server — this inventory capability exists for ALL users, not just parental control scenarios.

---

## Part 5: Silent FCM Push — Parental Control or Privacy Risk?

**The viral claim:** "The server silently steals user data via push notifications."

**What the code shows:** In a full JADX pass, `onMessageReceived` can sometimes be recovered with `--show-bad-code` (the default decompilation in this repo previously left the method as an oversized stub). Treat the table below as **behavior implied by recovered code strings**, not something you can eyeball in a single line number—re-run JADX with `--show-bad-code` on your copy of the APK to confirm.

The handler distinguishes **at least these FCM-driven actions** (names from message/type routing):

| FCM / payload role (as decompiled) | What It Triggers |
|----------|-----------------|
| `restriction_update` | Syncs parental control rules |
| `permission_sync` | Syncs device permission status |
| `metadata_update` | Syncs device metadata |
| DNS-related sync | Syncs DNS blocking rules |
| `installed_apps_sync` | **Triggers installed-apps upload path** |
| `app_usage_sync` | **Triggers app-usage upload path** |
| `supervision_update` | Handles parent-child relationship changes |

The code explicitly logs: `"Silent sync message, no notification displayed"` for data-only FCM messages.

**What the viral post gets wrong:** This is NOT a "trojan." These FCM triggers are part of the parental control infrastructure — parents can remotely check their child's installed apps and usage.

**The legitimate concern:** The same mechanism works for ALL users, not just children. There is no code-level distinction between "this is a parent asking" vs. "this is our server requesting data." The server CAN silently request installed apps and usage data from any user's device without notification. Whether it actually does depends on server-side logic, which cannot be verified through static analysis.

---

## Part 6: Periodic Data Sync Workers

Beyond FCM triggers, the app has two **periodic sync workers** that run every 6 hours:
- `InstalledAppsSyncWorker`: uploads the complete list of installed apps
- `AppUsageSyncWorker`: uploads per-app usage duration

These are scheduled via WorkManager and run regardless of whether parental controls are active.

**Mission-effectiveness angle (why this is not *only* “evil telemetry”):**  
DNS filtering alone cannot prove that a user’s *overall* digital habits improved. If the product goal is “less time on harmful apps” or “parent sees child device honestly,” then **aggregate or per-app usage** and **inventory** can be inputs to that question—similar to how parental dashboards work. The fair critique is **proportionality and consent language**: the same streams that help measure effectiveness for **supervised/child** setups look heavy for a **general adult** user if they were never clearly framed as “we upload your full app list and hourly usage to our servers,” independent of parental mode.

---

## Part 7: Privacy Policy vs. Reality

**What the privacy policy (last updated March 2024) correctly discloses:**
- Facebook Analytics, Google Analytics for Firebase, Firebase Crashlytics, Google Play Services under Third Party Access
- Collection of: device info, app version, OS version, feature usage, child device app lists, screen time, public IP, country-level location

**What the privacy policy does NOT disclose:**
1. **PostHog analytics** — the self-hosted instance at `p.kahf.co.uk` is never mentioned
2. **Session replay infrastructure** — no mention of UI recording capability
3. **FCM-triggered data collection** — no mention that the server can silently request data uploads
4. **Periodic sync mechanisms** — 6-hour installed apps and usage sync workers are not described
5. The policy claims "Only aggregated, anonymized data is periodically transmitted" — but advertising identifiers (GAID), per-app usage, and installed app lists are identifiable, not anonymous
6. The policy is 13+ months old and does not reflect current integrations

**CEO's public statements vs. code:**
- "We never sell your data" — true in the narrow legal sense (no direct sale), but data IS shared with Meta and PostHog
- "Zero-surveillance, privacy-first" — objectively false. The app monitors 18 browsers, reads accessibility nodes from 14+ social apps, syncs installed apps and usage data, and has session replay infrastructure
- "No data collection" — contradicted by Facebook SDK, PostHog, Firebase, and sync workers

---

## Part 8: Surveillance vs. Analytics — The Important Distinction

A key question is: **Is this surveillance or just analytics?** The answer is: **it's both, through different mechanisms.**

### Analytics (scoped to Kahf Guard's own screens):

The SDKs (Facebook, PostHog, Firebase) track user interaction **within Kahf Guard itself** — settings screens, onboarding flows, dashboard views. This is standard product analytics:
- Which settings the user toggles
- How long they spend on each screen within Kahf Guard
- App installs, launches, crashes
- Device model, OS version, language

This is the same kind of analytics used by millions of apps and is **not surveillance** in the conventional sense.

**However, some of this analytics is genuinely necessary for the product's purpose.** Kahf Guard exists to help users avoid harmful content. To know if it's actually working, the developers need data like: how many sites are being blocked, whether Reels/Shorts blocks are reducing social media usage, retention rates, and crash reports. Without this feedback loop, they cannot improve the blocking effectiveness. Firebase and PostHog (self-hosted) are reasonable tools for answering these questions. The debate is about **proportionality** — not whether analytics should exist at all.

### Cross-app monitoring (Kahf Guard's own code, for blocking):

The Accessibility Service (`KahfBlockerService`) reads URLs from browsers and monitors social apps. This is **necessary for the blocking functionality** and is Kahf Guard's own code — not any analytics SDK. The URLs are used locally for blocking decisions. However, the **capability** to read all URLs and social app screens exists.

For a product whose core promise is "block haram content," knowing which browsers and social apps users spend time on — and how effectively blocks are reducing that time — is directly relevant to measuring product success. The accessibility permissions are the mechanism; the question is whether the data collected through them is proportional to the blocking need.

### Server-side data collection (goes beyond blocking analytics):

Beyond blocking and standard analytics, Kahf Guard has mechanisms that go further:
- **`InstalledAppsSyncWorker`**: uploads the complete list of all installed apps (including system apps) every 6 hours
- **`AppUsageSyncWorker`**: uploads per-app usage durations every 6 hours
- **FCM-triggered sync**: server can silently request both of these uploads immediately, without user notification
- This applies to **ALL users**, not just parental control scenarios

For **parental control** use cases, knowing what apps a child has installed and how much time they spend on each is directly useful. For general users who installed the app for personal accountability, this data collection is harder to justify as necessary for blocking. It becomes device inventory and usage telemetry — data that could be useful for product development but goes beyond what blocking requires.

The viral post conflates these three categories, which leads to confusion. The SDKs are not doing the cross-app monitoring; Kahf Guard's own code is. And some of this data collection is understandable for a blocking/accountability product. But the breadth (all installed apps, all usage durations, all users — not just children) and the secrecy (undisclosed in privacy policy, silent FCM triggers) push parts of it beyond reasonable product analytics.

---

## Part 9: Developer Logging — What's Exposed and Who Can See It

A ripgrep count on this repo’s JADX output (`com.kahf.dns`, `*.java`) finds on the order of **~165 calls** to `Log.d` / `Log.e` / `Log.w` / `Log.i` across **~20** first-party files (APK v4.6.188). Additional diagnostics may go through wrappers (e.g. internal logging helpers) that this simple pattern does not capture—so treat the number as a **lower bound on noisy debug output**, not an exact audit.

### Can analytics SDKs read these logs?

**PostHog `captureLogcat`**: The PostHog Android SDK has an optional `captureLogcat` feature in session replay that could capture device logs. **In Kahf Guard's decompiled code, `captureLogcat` is NOT explicitly enabled** (the PostHog config in `KahfGuardApp.m4518d` sets `sessionReplay`, `captureScreenViews`, and lifecycle events, but does not set `captureLogcat`). The default value for this option is `false`. So these logs are NOT being sent to PostHog.

**Firebase Crashlytics**: Only captures custom logs (`Crashlytics.log()`) alongside crash reports. It does NOT automatically read `Log.d`/`Log.e` statements from the app's logcat.

**Facebook SDK**: Does NOT have any logcat capture feature. It only logs its own app events.

**Bottom line: The developer logs stay on the device.** They are not captured or sent by any analytics SDK in this build.

### What do the logs expose?

**Low sensitivity (operational/debug):**
- `AccessibilityMonitor`: logs when accessibility service enables/disables, SafetyNet check results
- `BlockOverlay`: logs overlay UI creation errors, animation errors
- `BlockPauseManager`: logs pause/unpause actions for blocked apps
- `BootRecoveryWorker`: logs service restart scheduling after boot

**Medium sensitivity (configuration data):**
- `KahfGuardApp`: logs "PostHog SDK initialized successfully", "Firebase App initialized", "Crashlytics configured"
- `MainActivity`: logs PostHog analytics initialization
- Notification service: logs notification channel creation, permission checks

**Higher sensitivity (but partially mitigated):**
- **FCM token**: `Log.d("KG_SYNC", "[FCM_TOKEN] New token generated: " + AbstractC10230n.m14287O0(20, token) + "...")` — logs the **first 20 characters** of the Firebase Cloud Messaging token. This is truncated (not the full token), but still exposes a portion.
- **FCM message data**: `Log.d("SZ_NOTI", "Message received: data - " + remoteMessage.getData())` — logs the **full data payload** of every FCM message, including restriction IDs, device IDs, and sync types.
- **Deep link URLs**: `Log.d("SZ_NOTI", "Opening URL - " + str7)` — logs full click URLs from notifications.
- **Authentication tokens from deep links**: logged in some code paths.

### Risk assessment:

These are clearly **developer debug logs left in production** — a common but sloppy practice. The risks are:
- **On Android 4.1+**: logcat is app-private, so other apps cannot read these logs
- **Via ADB**: if the device is connected to a computer with USB debugging, all logs are readable
- **The FCM token truncation is good practice** but still logs 20 chars of a sensitive value
- **The full FCM data payload logging** is the most concerning — it could include device IDs, restriction IDs, and other server-originated data

These logs are NOT sent to any analytics service, but they represent **developer negligence** that should be addressed in a production app.

---

## Balanced Conclusion

### What the viral post gets RIGHT:
1. Facebook SDK IS present with advertiser tracking enabled
2. Session replay infrastructure (PostHog, not OpenReplay) exists in the code
3. Installed apps and usage data ARE synced to servers periodically and on-demand
4. The CEO's "zero surveillance" claim is contradicted by the code
5. The privacy policy is outdated and incomplete

### What the viral post gets WRONG:
1. It's not a "trojan" — it's a legitimate blocking app with legitimate monitoring needs
2. "Sold to Facebook" is the wrong framing — it's SDK telemetry for ad attribution, not a data sale
3. OpenReplay doesn't exist in this build — it's PostHog
4. "11 IP lookup services" is unsubstantiated — only 1 HTTPS endpoint found
5. "Anyone can sign malware" — false, requires a private key
6. ALLOWLISTED_SIG is standard dual-signing, not a backdoor
7. The SDKs (Facebook, PostHog, Firebase) can only track Kahf Guard's **own screens** — they cannot see your activity in other apps. The cross-app monitoring is done by Kahf Guard's Accessibility Service for blocking purposes

### The real issue is proportionality and transparency:
- A content-blocking app **needs** some analytics to measure blocking effectiveness — this is reasonable
- A content-blocking app **needs** accessibility permissions to block inside browsers and social apps — this is necessary
- But it doesn't **need** Facebook's advertising SDK — GAID collection and Meta ad attribution are not required for blocking
- Session replay capability (even if limited to Kahf Guard's own screens and masked by default) goes beyond what blocking requires
- Uploading installed apps and usage data for ALL users (not just parental control) goes beyond what's needed to measure blocking effectiveness
- The privacy policy should disclose ALL third-party integrations
- The CEO should stop using absolute language ("zero," "never") when the code says otherwise

Both the viral post and Kahf's marketing use extremes. The truth is in the middle: Kahf Guard is a legitimate product with legitimate monitoring needs, but it integrates analytics tools that go beyond what's necessary, fails to fully disclose them, and markets itself with language its own code contradicts.

---

*This analysis is based on static code analysis of APK v4.6.188 using JADX 1.5.5. Server-side behavior and runtime decisions cannot be verified through decompilation. Not legal advice.*

**Sources:**
- Project reference: [SDK_Data_Collection_Guide.md](SDK_Data_Collection_Guide.md) (third-party SDK collection summary used to cross-check wording here)
- [Meta App Events - Automatic Event Logging](https://developers.facebook.com/docs/app-events/getting-started-app-events-android/)
- [Meta App Events - Advanced Matching](https://developers.facebook.com/docs/app-events/advanced-matching)
- [Google Analytics - Automatically Collected Events](https://support.google.com/analytics/answer/9234069)
- [Firebase Analytics - Automatically Collected User Properties](https://support.google.com/firebase/answer/6317486)
- [Firebase Crashlytics - Data Collection](https://firebase.google.com/docs/crashlytics/data-collection)
- [Google Play Services - Data Collection](https://support.google.com/googleplay/answer/9023446)
- [PostHog Android SDK](https://posthog.com/docs/libraries/android)
- [PostHog Mobile Session Replay](https://posthog.com/docs/session-replay/mobile)
- [PostHog - Data Collected](https://posthog.com/docs/privacy/data-collected)
- [PostHog Session Replay - Privacy Controls](https://posthog.com/docs/session-replay/privacy)
- [Android Application Sandbox](https://source.android.com/docs/security/app-sandbox)
- [Kahf Guard Privacy Policy](https://kahfguard.com/privacy-policy/)
- [Privacy International - How Apps on Android Share Data with Facebook](https://privacyinternational.org/report/2647/how-apps-android-share-data-facebook-report)
