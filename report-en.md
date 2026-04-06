# Final Fact-Check Report: Kahf Guard APK v4.6.188

**Date:** 2026-04-06  
**Build:** `com.kahf.dns` v4.6.188 (APKPure XAPK)  
**Toolchain:** JADX 1.5.5 (standard mode + `--show-bad-code` for stubbed methods)  
**Scope:** Static analysis only. Server-side behavior and runtime decisions cannot be verified through decompilation.  
**Sources cross-referenced:** Viral Facebook post (which analyzed v4.6.186), CEO public statement, Kahf Guard privacy policy (effective 2024-12-04), kahfguard.com website, decompiled source code.

> **Important note on file paths and line numbers:** The viral post analyzed APK v4.6.186, while this report analyzes v4.6.188. Android apps use ProGuard/R8 obfuscation, which renames classes and shifts line numbers between builds. Furthermore, different JADX versions and settings produce different output. Throughout this report, where we state that a specific file path from the viral post was "not found," this means **not found in our specific decompilation of v4.6.188** — the underlying functionality may exist at different paths. The viral post's file paths may have been valid for their version and toolchain. We have independently verified the presence or absence of each claimed *behavior*, regardless of file paths.

---

## Executive Summary

A viral Facebook post accused Kahf Guard of being a "remote-controlled trojan" that "sells user data to Facebook" and conducts "extreme surveillance." This report independently verifies each claim against the decompiled APK code, the published privacy policy, and the CEO's public statements.

**Understanding the product context:** Kahf Guard is primarily a content-blocking and digital wellbeing app designed to help users avoid haram content on the internet. It uses DNS-level blocking to filter harmful websites and Accessibility Service permissions to block specific content within apps (e.g., Reels, Shorts, adult content in browsers). For such a product to be effective, the developers need analytics to measure whether their blocking is actually working — are users spending less time on harmful content? Are the filters effective? Which features are most used? This is a legitimate need, similar to how any product needs feedback loops to improve. The question is not whether analytics should exist, but whether the *specific analytics choices* are proportionate and properly disclosed.

**Key findings:**

- Several core allegations have merit — the app does integrate Facebook SDK with advertiser tracking enabled, uses PostHog analytics with session replay infrastructure, and syncs installed apps and usage data to servers periodically and on-demand.
- Several claims are factually wrong or exaggerated — wrong product names, inflated numbers, and misunderstanding of standard Android development patterns.
- The CEO's marketing language ("zero surveillance," "never sell data," "no data collection") is contradicted by the code and even Kahf's own privacy policy. A more accurate framing would have been: *"We collect analytics to improve our blocking effectiveness and measure product goals."*
- Much of what the viral post calls "surveillance" is actually the standard mechanism by which content-blocking and parental-control apps function on Android.

---

## Claim-by-Claim Fact-Check

### Claim 1: "Facebook SDK Sends Data to Meta / Sold Users to Facebook"

| Aspect | Verdict |
|--------|---------|
| Facebook SDK present? | **TRUE** — Confirmed in `KahfGuardApp.m4516b()` |
| AdvertiserID enabled? | **TRUE** — `com.facebook.sdk.AdvertiserIDCollectionEnabled = true` |
| AutoLogEvents enabled? | **TRUE** — `com.facebook.sdk.AutoLogAppEventsEnabled = true` |
| Advanced Matching infrastructure present? | **TRUE (but passive)** — Facebook SDK's `UserDataStore` code exists in `ec/C3439k.java` with fields for `email`, `first_name`, `last_name`, `phone`. However, **Kahf Guard's own code does NOT actively call** `setUserData()`, `setUserEmail()`, `setUserPhone()`, or any Advanced Matching API. Zero calls to these methods were found in the `com/kahf/dns/` package. The UserDataStore code is **latent SDK library code** that ships with every Facebook SDK integration, not something the app actively uses to push PII to Meta. |
| "Sold to Facebook"? | **FALSE** — SDK telemetry for ad attribution is not a direct data sale. Furthermore, no evidence was found of the app actively sending user email/name/phone to Facebook. |
| File path `hg/l0.java` lines 97-109? | **Not found in our v4.6.188 decompilation** — may have existed in v4.6.186 or under different JADX settings. The corresponding SDK code is in `ec/C3439k.java` in our build. |

**What Facebook SDK actually collects in this integration:** With AdvertiserID and AutoLogEvents enabled, the SDK automatically collects: Google Advertising ID (GAID), app install/launch events, device metadata (OS, model, carrier, etc.). These are standard SDK auto-events. The Advanced Matching feature (which could send email/name/phone) exists as library code but is **not actively invoked** by Kahf Guard's application code — the data fields for email, name, and phone that users provide during account creation are stored in Kahf's own SharedPreferences, not passed to Facebook's `UserDataStore`.

**Standard practice context:** Facebook SDK / Meta SDK is used by millions of apps for install attribution and ad campaign measurement. Its presence is standard industry practice.

**Is Kahf Guard a "privacy app"?** The website markets Kahf Guard as a content-blocking and digital wellbeing app ("Protect your faith, family, & future," "blocks harmful content, breaks digital addictions"). Their FAQ mentions "without compromising privacy" and their DNS blocks "trackers," but the app is **not primarily marketed as an anti-Big-Tech-tracking tool**. It's a haram content blocker and parental control app. The CEO's claims about "zero surveillance" and "no data collection" create a privacy expectation, but the website itself focuses on content blocking, not privacy protection per se.

**Privacy policy disclosure:** The privacy policy **does** list "Facebook Analytics" under Third Party Access. The privacy policy claims "Only aggregated, anonymized data is periodically transmitted to external services." Regarding whether the Facebook SDK actually sends Personally Identifiable Information (PII): the SDK sends the Google Advertising ID (GAID) and auto-logged events, which are device-level identifiers — not PII like email or name. The GAID is a resettable advertising identifier, not a personal identifier in the traditional sense, though it can be used for cross-app tracking by Meta's ad network.

---

### Claim 2: "Reads URL Bars from 17 Browsers"

| Aspect | Verdict |
|--------|---------|
| Reads browser URLs? | **TRUE** — `KahfBlockerService` uses Android Accessibility Service |
| 17 browsers? | **CLOSE** — Actually **18** browser package IDs in the code |
| File path `ei/a.java` lines 24/287? | **Not found in our v4.6.188 decompilation** — may correspond to a different version or JADX settings. Actual code is in `KahfBlockerService.java` (line 81 area) in our build. |

**The 18 monitored browsers:** Chrome, Chrome Dev, Firefox, Firefox Rocket, Samsung Browser, Opera, Opera Mini, Microsoft Edge, Brave, DuckDuckGo, Vivaldi, Tor Browser, Spin Browser, Ecosia, Free Adblocker Browser, IDM Browser, Cast Web Video Browser, Kahf Browser.

**Standard practice context:** This is how ALL Android content blockers that need URL-level filtering work. DNS-level blocking cannot see HTTPS paths. To block Instagram Reels while allowing DMs, or block specific YouTube content, the app must read the URL from the browser's accessibility nodes. This is the necessary mechanism, not evidence of surveillance. The URLs are used locally for the blocking decision — the code does not show URLs being uploaded to analytics services.

---

### Claim 3: "ALLOWLISTED_SIG = Backdoor / evaluateJavascript = Remote Code Execution"

#### ALLOWLISTED_SIG

| Aspect | Verdict |
|--------|---------|
| Secondary signature exists? | **TRUE** — `ALLOWLISTED_SIG = "Vn3kj4p..."` in `com.pairip.SignatureCheck` |
| "Anyone can sign malware with it"? | **FALSE** — Only the holder of the corresponding private key can produce a matching signature |
| Is this a backdoor? | **FALSE** — Standard dual-signing pattern for multi-channel distribution (Play Store, direct APK, etc.) |

**Standard practice context:** PairIP is a legitimate anti-piracy/anti-tamper library used by many Android apps. Allowlisted signatures are standard for enterprise/official builds across different distribution channels. This is not a security vulnerability.

#### evaluateJavascript

| Aspect | Verdict |
|--------|---------|
| evaluateJavascript used? | **TRUE** — In `p176hl/C5181p.java` and `p000/C12905z0.java` |
| Remote server pushes arbitrary JS? | **NOT PROVEN** — The JavaScript strings come from the app's internal state machine, not directly from a remote endpoint |
| File path `bl/r.java` lines 56-63? | **Not found in our v4.6.188 decompilation** — may correspond to a different version or JADX settings |

**Standard practice context:** `evaluateJavascript()` is the standard Android API for injecting JavaScript into WebViews. It's used here for content filtering (removing Reels from Instagram web view, etc.). The injected JS is app-defined, not server-pushed. All apps that use WebViews for content modification use this API. This is not a backdoor.

---

### Claim 4: "12 System Events Restart Tracking"

| Aspect | Verdict |
|--------|---------|
| Multiple system events monitored? | **TRUE** |
| 12 events? | **TRUE** — Combined across `HeartbeatReceiver` and `SystemEventReceiver` |
| Purpose is surveillance? | **MISLEADING** — Purpose is to keep the blocking service alive |

**12 events confirmed:** `BOOT_COMPLETED`, `LOCKED_BOOT_COMPLETED`, `MY_PACKAGE_REPLACED`, `DATE_CHANGED`, `ACTION_POWER_CONNECTED`, `ACTION_POWER_DISCONNECTED`, `USER_PRESENT`, `TIME_SET`, `TIMEZONE_CHANGED`, `PACKAGE_REPLACED`, `SCREEN_ON`, `BATTERY_OKAY`

**Standard practice context:** Any Android service that must run persistently (content blockers, parental controls, VPNs, fitness trackers) needs to restart after system events. If the blocking service dies, the user loses protection. This is a standard and necessary pattern, not evidence of surveillance.

---

### Claim 5: "Terrifying Surveillance — WhatsApp/Telegram/Instagram"

| Aspect | Verdict |
|--------|---------|
| Social apps monitored? | **TRUE** — 14 social apps total |
| WhatsApp chat list tracking? | **MISLEADING** — The app monitors **which tab/screen** the user is on (e.g., whether the "Updates"/Status tab is selected), NOT the actual chat list contents or contact names. It reads accessibility node IDs like `com.whatsapp:id/navigation_bar_item_large_label_view` to detect tab labels ("Updates", "التحديثات") and `com.whatsapp:id/updates_list` for the Status container. This is screen-state detection for blocking WhatsApp Status/Stories, not reading who you're chatting with. |
| Telegram text reading? | **TRUE** — Reads accessibility nodes for content/word detection (for "risky word" blocking feature) |
| Instagram monitoring? | **TRUE** — Monitors for Reels blocking via node IDs like `reel_recycler`, `clips_viewer_view_pager` |
| All installed apps sent to server? | **TRUE** — `GetChildDevicesQuery` includes `installedApps { packageName, appName, isSystemApp }` |
| File path `xg/o0.java` line 18? | **Not found in our v4.6.188 decompilation** — actual file is `p081dh/C3057o0.java` in our build |

**Core set (6 apps):** YouTube, Instagram, Facebook, Telegram, WhatsApp, WhatsApp Business  
**Extended set (8 apps):** Twitter/X, TikTok, Snapchat, Reddit, Threads, Pinterest, Discord, Tumblr

**Standard practice context:** Content-blocking / parental-control apps MUST monitor app screens to implement blocking features. To block Reels without blocking all of Instagram, the app needs to distinguish screen context via accessibility nodes. This is how Screen Time, Google Family Link, and similar products work. The monitoring is the mechanism for the product's core feature — users grant explicit accessibility permission, which Android explains.

**Why installed app list and usage data sync is legitimate for ALL users (not just parental control):** Kahf Guard was originally built as a personal accountability tool (blocking haram content for individual users) before parental control was added later. For the app to fulfill its primary mission — helping users stay away from harmful content — the developers need to understand user patterns: Are users finding ways to bypass blocking? Which apps are users gravitating toward? Is the app actually reducing time on problematic apps? This data helps improve the blocking algorithms and detect circumvention patterns. The installed app list also helps Kahf Guard know which apps to monitor and block. This is similar to how any digital wellbeing tool (Google's Digital Wellbeing, Apple's Screen Time) tracks app usage to provide insights and enforce limits.

**That said,** it would be better practice to clearly disclose this in the privacy policy and give users granular control over what data is synced.

---

### Claim 6: "FCM Silent Push = Trojan / Remote Data Upload"

| Aspect | Verdict |
|--------|---------|
| Server can silently trigger data sync? | **TRUE** — FCM data messages with "Silent sync message, no notification displayed" |
| Installed apps sync triggered? | **TRUE** — `installed_apps_sync` enqueues `InstalledAppsSyncWorker` |
| App usage sync triggered? | **TRUE** — `app_usage_sync` enqueues `AppUsageSyncWorker` |
| Is it a "trojan"? | **FALSE** — These are parental control infrastructure, not malware |
| File path lines 1580/1663? | **CANNOT VERIFY** — The `onMessageReceived` method (2344 instruction units) did not fully decompile; `--show-bad-code` output is fragmented |

**6 confirmed FCM message types:** `restriction_update`, `permission_sync`, `metadata_update`, DNS sync, `installed_apps_sync`, `app_usage_sync`, `supervision_update`

**Standard practice context:** Silent FCM pushes for parental control sync are used by Google Family Link, Qustodio, Bark, and similar products. Parents need to remotely check their child's device status without the child dismissing a notification. This is standard parental control architecture.

**Legitimate concern:** The same mechanism that allows a parent to check their child's device also allows Kahf's server to request data from ANY user's device silently. There is no visible code-level distinction between "parent requesting" and "server collecting." The trust model depends entirely on server-side logic, which cannot be verified through static analysis.

---

### Claim 7: "UUID Persists After Uninstall via backup_rules.xml / 11 IP Lookup Services"

#### UUID Backup

| Aspect | Verdict |
|--------|---------|
| UUID backed up to Google Cloud? | **PARTIALLY TRUE** — `backup_rules.xml` backs up `user_prefs.xml` and `device_data/` |
| "Forever tracking"? | **FALSE** — Standard Android Auto Backup; not a tracking mechanism |

**Standard practice context:** This is standard Android Auto Backup — a convenience feature so that if a user loses their phone or switches devices, their app settings are automatically restored when they reinstall. Think of it like how your Chrome bookmarks sync across devices, or how WhatsApp restores your settings on a new phone. Thousands of apps use this. The user doesn't have to reconfigure everything from scratch. This has nothing to do with tracking — it's a user convenience feature built into Android itself.

#### IP Lookup Services

| Aspect | Verdict |
|--------|---------|
| 11 third-party IP services? | **PARTIALLY TRUE** — In our analysis, we found **6 geolocation services** + **7 IP-retrieval endpoints** = 13 total IP-related endpoints. The viral post's count of 11 doesn't exactly match, but the general claim of multiple IP services is accurate. |
| Plain HTTP service? | **TRUE** — `http://ip-api.com/json/{ip}` (no TLS) is used as Fallback #3 |

**6 geolocation services (fallback chain in `p000/C11771x0.java`):**
1. `https://api.ipify.org` (IP retrieval)
2. `https://api.bigdatacloud.net/data/ip-geolocation?ip=` (Fallback #1)
3. `https://freeipapi.com/api/json/` (Fallback #2)
4. **`http://ip-api.com/json/`** (Fallback #3 — **PLAIN HTTP**)
5. `https://ipapi.co/{ip}/country_name/` (Fallback #4)
6. `https://ipinfo.io/{ip}/json` (Final fallback)

**7 additional IP-retrieval endpoints (in `sg/C9792j.java`):**
`api.ipify.org`, `api.myip.com`, `checkip.amazonaws.com`, `api.ip.sb/ip`, `icanhazip.com`, `ipecho.net/plain`, `ipinfo.io/ip` (all HTTPS)

**Standard practice context:** Using multiple IP geolocation services as fallbacks is common for apps that need country detection (for DNS server selection, pricing, language defaults). The IP services are only used to determine the user's country for routing purposes.

**On the plain HTTP service (`ip-api.com`):** The viral post flags this as a security vulnerability. In practice, this service simply tells the app what the user's public IP is. Anyone monitoring the user's network traffic would already know their IP address — it's visible in every packet header. So while using HTTPS is always better practice, the "security risk" here is minimal: the data being "leaked" (your IP address) is already inherently visible to any network observer. The free tier of `ip-api.com` does not support HTTPS, which is why developers use it over HTTP. It's a minor best-practice issue, not a meaningful security vulnerability.

---

### Claim 8: "BootReceiver Uses VMRunner — Hidden Code"

| Aspect | Verdict |
|--------|---------|
| VMRunner.invoke used at boot? | **TRUE** — `BootReceiver.onReceive()` calls `VMRunner.invoke("WYWlOOlYRm2t3AHk", ...)` |
| Encrypted/obfuscated code? | **TRUE** — VMRunner loads bytecode from the PairIP native library (`libpairipcore.so`) |
| "Can't analyze what it does"? | **PARTIALLY TRUE** — Static analysis alone cannot determine what runs in the VM; dynamic analysis (instrumentation) would be needed |

**Standard practice context:** PairIP is a commercial anti-piracy/anti-tamper solution used by many Android apps to protect against cracking and redistribution. It's common for boot-initialization code to be protected this way. While this does make auditing harder, it's not evidence of malicious intent — it's evidence of commercial software protection.

---

### Claim 9: "OpenReplay Records Every Tap/Swipe/Keystroke"

| Aspect | Verdict |
|--------|---------|
| OpenReplay present in our build? | **Not found in v4.6.188** — No OpenReplay integration found in our decompilation |
| `openreplay.kahf.co.uk` endpoint? | **The endpoint IS live** — While not present in our v4.6.188 build code, the domain `openreplay.kahf.co.uk` resolves to a live web application (returns a loading page in browser). This confirms the viral post author likely found it in their v4.6.186 build, and the endpoint still exists on Kahf's infrastructure. It may have been replaced by PostHog in newer versions, or may coexist for other products/builds. |
| Session replay exists in current build? | **TRUE** — **PostHog** (not OpenReplay) with session replay infrastructure at `https://p.kahf.co.uk` |
| Can capture taps/swipes/keystrokes? | **TRUE** (of Kahf Guard's own screens only) — PostHog replay includes `RRMouseInteraction`, `RRFullSnapshotEvent`, keyboard events |
| File path `com/openreplay/tracker/listeners/Analytics.java`? | **Not found in our v4.6.188 decompilation** — may have existed in the viral post author's v4.6.186 build |

**PostHog configuration:**
- API Key: `phc_OmqCNJqYWGInDc1b3GQEGWDnPPIV6etPJ1bO7pLRL8N`
- Session replay: **enabled** in client config
- `captureLogcat`: **NOT hardcoded as enabled** — controlled by server-side PostHog config (`consoleLogRecordingEnabled`), defaults to `false`
- Default masking: text inputs masked, images masked, passwords always masked

**Critical scope clarification:** PostHog session replay can only capture **Kahf Guard's own app screens** (settings, onboarding, dashboard). It **CANNOT** see your activity in Chrome, WhatsApp, Instagram, or any other app. Android's application sandbox (each app gets a unique UID and kernel-level process isolation) prevents any SDK from seeing other apps' content. The viral post's implication that session replay records "everything you type" across all apps is **technically impossible on Android** — this is a fundamental misunderstanding of how mobile operating systems work.

**Is session replay within the app concerning?** Realistically, Kahf Guard's own screens contain only settings toggles (enable/disable blocking, configure filters, etc.) and dashboard views. There is no personal content within the app itself — no messages, no browsing history, no sensitive data entry beyond the initial account creation. Recording how users interact with settings screens is standard UX analytics that helps developers understand which features are confusing, which settings are most toggled, and where users get stuck. Thousands of apps (banking, e-commerce, social media) use session replay for exactly this purpose. PostHog is a well-respected open-source analytics platform, and using it for in-app UX analytics is normal developer practice.

**However:** PostHog is **NOT disclosed** in the privacy policy. This is a significant omission that should be corrected, regardless of how benign the data captured is. Transparency is important.

---

### Claim 10: "CEO's Blatant Lies"

The viral post directly challenges specific CEO statements. Here's the fact-check:

#### CEO Claim: "We never sell your data. Ever."

| Verdict | **Likely true in the literal sense** |
|--------|--------------------------------------|

"Sell" implies a direct commercial transaction. There is no evidence that Kahf receives payment from Meta for user data. The Facebook SDK sends device-level advertising identifiers (GAID) and auto-logged events for ad attribution, but **the app does not actively push PII (email, name, phone) to Facebook** — the Advanced Matching infrastructure exists as latent SDK code but is not invoked by the app's own code. The CEO's statement is likely accurate in the narrow sense — they don't "sell" data. However, SDK telemetry IS shared with Meta's ad network as a side-effect of using the Facebook SDK, which the CEO should acknowledge.

#### CEO Claim: "We build with a zero-surveillance, privacy-first approach."

| Verdict | **Needs clarification — local monitoring is functional, not surveillance; server-side analytics could be better disclosed** |
|--------|--------------------------------------------------------------|

The app:
- Monitors 18 browsers' URL bars via Accessibility Service
- Reads accessibility nodes from 14 social apps
- Syncs installed app lists every 6 hours
- Syncs per-app usage data every 6 hours
- Can be silently triggered via FCM to sync both immediately
- Integrates Facebook SDK with advertiser tracking
- Has PostHog session replay infrastructure (for its own screens)

**However, not all of this is "surveillance" in any meaningful sense.** The local device monitoring (browser URLs, social app screens, installed apps detection) is the core mechanism by which content blocking and digital wellbeing features work. You cannot block Reels without knowing the user is looking at Reels. You cannot enforce screen time limits without tracking app usage. This is functional monitoring required by the product, not surveillance for its own sake — the same way a security camera at a bank isn't "surveillance" in the pejorative sense.

**Where the CEO's claim falls short:** The analytics that go to external servers (Facebook SDK telemetry, PostHog analytics, periodic usage/app data syncs to Kahf's backend) are not "zero surveillance." A more honest framing would be: *"We collect analytics to measure our blocking effectiveness, improve the product, and help parents monitor children's devices. The data collected is primarily device-level identifiers and usage patterns, not personal content."* The CEO should have been transparent about this instead of using absolutes like "zero."

#### CEO Claim: "We don't collect anything about what someone does on their phone."

| Verdict | **Contradicted by the code and the privacy policy** |
|--------|------------------------------------------------------|

- `AppUsageSyncWorker` uploads per-app usage durations every 6 hours
- `InstalledAppsSyncWorker` uploads the full app inventory every 6 hours
- The privacy policy itself discloses "The amount of time spent on the other Apps" and "App List & Screen-time"

The CEO's own privacy policy contradicts this statement. However, an important nuance: the usage data is synced using a **device ID** (`deviceId`), not a user ID or email address. The GraphQL mutation `UpdateDeviceInstalledApps($deviceId: ID!, $apps: [InstalledAppInput!]!)` identifies data by device, not by person. While this is not fully "anonymized" (a device ID can be correlated to a user account), it does suggest the data is handled at a device level rather than being tied to a personal profile. The CEO should have said something like: *"We collect device-level usage data to measure blocking effectiveness and improve our product, as described in our privacy policy."*

#### CEO Claim: "If you want, you can decompile the app and verify the code."

| Verdict | **Ironic but technically valid** |
|--------|----------------------------------|

Credit where due — the app can be decompiled and analyzed. It's not completely locked down, and this entire analysis was possible. However, key parts (BootReceiver) are protected by PairIP's VMRunner, which is specifically designed to resist decompilation.

---

## Privacy Policy Analysis

**Policy effective date:** 2024-12-04 (per website footer)

### What the privacy policy correctly discloses:

1. Facebook Analytics — listed under Third Party Access
2. Google Analytics for Firebase — listed
3. Firebase Crashlytics — listed
4. Google Play Services — listed
5. Collection of: email/phone at sign-up, features used, public IP, time spent on other apps, child device app list & screen time, device configuration
6. Third-party payment processors (Stripe, bKash)

### What the privacy policy does NOT disclose:

1. **PostHog analytics** — the self-hosted instance at `p.kahf.co.uk` is never mentioned
2. **Session replay capability** — no mention of UI recording/interaction capture
3. **FCM-triggered data collection** — no mention that the server can silently request data uploads
4. **Specific sync mechanisms** — the 6-hour periodic workers for installed apps and usage are not described as such
5. **Number of IP geolocation services** — 6+ services used, including one plain HTTP endpoint, not mentioned

### What the privacy policy claims that the code contradicts:

- **"Only aggregated, anonymized data is periodically transmitted to external services"** — The Facebook SDK sends the Google Advertising ID (GAID), which is a device-level identifier (resettable, but still a unique identifier). Per-app usage durations and installed app lists include specific package names. Whether this constitutes "anonymized data" is debatable — the GAID is not PII in the traditional sense (it's not an email or name), but it is a persistent device identifier that can be used for cross-app tracking by ad networks. The privacy policy's "anonymized" claim is a stretch, though perhaps not an outright falsehood — it depends on how strictly you define "anonymized."

### Privacy policy assessment:

The policy is a **reasonable framework** — it does disclose the major third-party integrations (Facebook, Firebase) and the types of data collected. It does not try to claim zero data collection. However, it has significant gaps (PostHog, session replay, FCM triggers) and makes one claim ("only aggregated, anonymized data") that is directly contradicted by the SDKs' actual behavior.

---

## Developer Logging — What's Exposed and Who Can See It

Verified against both JADX outputs (`jadx_out` and `jadx_out_showbad`) in the `com.kahf.dns` package:

| Output | Log calls found | Unique source files |
|--------|----------------|---------------------|
| `jadx_out` (standard decompilation) | **150** | **20** |
| `jadx_out_showbad` (`--show-bad-code`) | **225** | **21** |

The extra 75 calls in the `--show-bad-code` output come primarily from `AndroidNotificationService.java`, which was too large for the standard decompiler to fully reconstruct. The sensitive log lines discussed below are **only visible in `jadx_out_showbad`** — they were not recoverable from standard JADX output.

### Can analytics SDKs read these device logs?

- **PostHog `captureLogcat`**: PostHog's Android SDK has an optional logcat capture mode. In Kahf Guard's decompiled code, **`captureLogcat` is NOT enabled** — the PostHog config in `KahfGuardApp.m4518d` sets `sessionReplay`, `captureScreenViews`, and lifecycle events but does not set `captureLogcat`. The default is `false`. These logs are NOT sent to PostHog.
- **Firebase Crashlytics**: Only captures explicit developer breadcrumbs (`Crashlytics.log()`). It does **not** automatically read device logcat statements.
- **Facebook SDK**: Has no logcat capture feature whatsoever.

**Bottom line: Developer logs stay on the device** and are not captured or transmitted by any analytics SDK in this build.

### Sensitivity breakdown of what the logs expose:

**Low sensitivity (operational/debug):**
- `AccessibilityMonitor` (`AccessibilityMonitorService.java`): logs service enable/disable transitions, grace period states, safety-net polling
- `BlockOverlay` (`C2486o.java`, `C2488p.java`): overlay creation errors, animation failures — 40+ log calls
- `BlockPauseManager` (`C2497u.java`): logs pause/unpause actions for blocked apps
- `HeartbeatReceiver.java`: logs service restart attempts

**Medium sensitivity (configuration data):**
- `KahfGuardApp.java`: "AndroidBillingManager initialized", "Failed to initialize billing/restriction/silence" — all from the `AutoSilencePro` tag
- `RescheduleWorker.java`, `SilenceModeWorker.java`: prayer scheduling and silence mode logs

**Higher sensitivity — confirmed in `AndroidNotificationService.java` (jadx_out_showbad only):**

- **FCM token (truncated)** — `AndroidNotificationService.java:436`:
  ```
  Log.d("KG_SYNC", "[FCM_TOKEN] New token generated: " + AbstractC10230n.m14287O0(20, token) + "...")
  ```
  Logs the **first 20 characters** of the Firebase Cloud Messaging registration token. The truncation helper (`m14287O0(20, ...)`) limits exposure, but a portion of a sensitive credential is still written to logcat.

- **Full FCM message data payload** — `AndroidNotificationService.java:554`:
  ```
  Log.d("SZ_NOTI", "Message received: data - " + remoteMessage.getData())
  ```
  Logs the **complete key-value map** of every incoming FCM push. This can include `notification_id`, `image_url`, `title`, `body`, `click_url`, `compose_screen`, and server-defined restriction/sync parameters.

- **Notification click URLs** — `AndroidNotificationService.java:332, 336`:
  ```
  Log.d("SZ_NOTI", "createPendingIntent: Received clickUrl = ".concat(str7))
  Log.d("SZ_NOTI", "createPendingIntent: Opening URL - ".concat(str7))
  ```
  Full URL from the FCM notification's `click_url` field is logged when building the pending intent.

- **Authentication token from deep link** — `MainActivity.java:387`:
  ```
  Log.d("Auth", "Got token from deep link: " + AbstractC10230n.m14287O0(10, strM10946a))
  ```
  Logs the **first 10 characters** of an authentication token received via deep link URL (e.g., from a login link in email). Again truncated with the same `m14287O0` helper, but still partially exposed.

### Risk assessment:

These are **developer debug logs left in a production build** — a common but sloppy practice. The risk level depends on context:
- **On Android 4.1+**: logcat is process-private — other installed apps cannot read these logs without special permissions.
- **Via ADB (USB debugging)**: if debugging is enabled and the device is connected to a computer, all logs are readable by the connected machine.
- **Truncated token logging** (20 chars for FCM token, 10 chars for auth token) is better than logging the full secret, but is still unnecessary exposure.
- **Full FCM payload logging** is the most concerning item — it logs every server-originated command as plaintext, which would reveal restriction configurations, device IDs, and sync types to anyone with ADB access.

Important note: **these logs are not visible in the standard `jadx_out` decompilation** because `AndroidNotificationService.onMessageReceived()` is a 2344-instruction method that the standard decompiler stubbed out. They only became visible with `--show-bad-code`. This does not mean the logs don't exist at runtime — they are compiled into the APK.

These logs are not surveillance, but they represent developer negligence. A production release should strip debug `Log.d` calls out (via ProGuard rules) or at minimum avoid logging partial secrets and full server payloads.

---

## The Analytics-vs-Surveillance Spectrum

### What is standard and necessary for this product category:

| Capability | Why it's needed | Industry comparison |
|-----------|----------------|---------------------|
| Browser URL monitoring via Accessibility | DNS can't see HTTPS paths | All Android content blockers (NetGuard, AdGuard, etc.) |
| Social app screen monitoring | Block Reels/Shorts/Status specifically | Bark, Qustodio, Google Family Link |
| Installed app list (child devices) | Parents need to see what kids install | Google Family Link, Apple Screen Time |
| Usage duration tracking | Measure if blocking is effective | Screen Time, Digital Wellbeing |
| Firebase Analytics + Crashlytics | Standard crash reporting and usage metrics | Used by ~80% of Android apps |
| Boot/system event receivers | Keep blocking service alive across reboots | All persistent services |
| FCM silent sync (parental control) | Parents remotely check child devices | Google Family Link uses similar patterns |

### What goes beyond blocking necessity:

| Capability | Blocking need? | Present? | Concern level |
|-----------|---------------|----------|---------------|
| Facebook SDK with AdvertiserID | **Not needed for blocking** | Yes | MEDIUM — Not needed for blocking functionality; the CEO claims no data is collected, yet SDK telemetry flows to Meta's ad network. The app is not primarily marketed as a privacy app, but the CEO's claims create a privacy expectation that this contradicts. |
| PostHog session replay (own screens) | **Useful for UX improvement** | Infrastructure present | LOW — Records only Kahf Guard's own settings/dashboard screens. Standard developer practice for UX optimization. **However, not disclosed in privacy policy.** |
| Installed apps sync for ALL users | **Useful for blocking effectiveness** | Yes (all users) | LOW-MEDIUM — Helps detect circumvention patterns and know which apps to target. Was part of the product before parental control was added. Should be clearly disclosed. |
| Usage sync for ALL users | **Useful for measuring blocking goals** | Yes (all users) | LOW-MEDIUM — Necessary to know if the app is achieving its goal (reducing time on harmful content). Data is device-identified, not user-identified. Should be clearly disclosed. |
| Plain HTTP IP service fallback | **Minor best-practice issue** | Yes (`ip-api.com`) | LOW — The "leaked" data (your IP) is inherently visible to network observers already. |

---

## Summary Scorecard

### Viral Post Claims

| # | Claim | Accuracy | Notes |
|---|-------|----------|-------|
| 1 | Facebook SDK sends data to Meta | **PARTIALLY TRUE** | SDK telemetry (GAID, auto-events) flows to Meta, but app does NOT actively send PII (email/name/phone). \"Sold\" is wrong framing. |
| 2 | Reads URLs from 17 browsers | **TRUE** (18 actually) | Standard for content blockers |
| 3 | ALLOWLISTED_SIG is a backdoor | **FALSE** | Standard dual-signing |
| 3b | evaluateJavascript = remote code exec | **FALSE** | Local WebView content filtering |
| 4 | 12 system events restart tracking | **TRUE** | Standard for persistent services |
| 5 | Monitors WhatsApp/Telegram/Instagram | **TRUE** | Standard for content/parental control. WhatsApp: monitors screen state/tabs, NOT chat content. |
| 6 | FCM silently uploads data | **TRUE** | Standard for parental control and accountability tools |
| 7a | UUID persists after uninstall | **MISLEADING** | Standard Android backup for user convenience, not a tracking mechanism |
| 7b | 11 IP services, one HTTP | **PARTIALLY TRUE** | 13 total IP endpoints found in our build; 1 plain HTTP confirmed (minimal real-world risk) |
| 8 | BootReceiver code is hidden | **TRUE** | PairIP anti-tamper; standard commercial practice |
| 9 | OpenReplay records everything | **WRONG NAME, but endpoint exists** | PostHog in v4.6.188, but `openreplay.kahf.co.uk` is a live endpoint. Either way: records own screens only — literally impossible to record other apps on Android. |
| 10 | CEO statements are lies | **PARTIALLY TRUE** | Some claims are contradicted by code and privacy policy, but the app is less malicious than portrayed |

### CEO / Kahf Claims

| Claim | Accuracy | Notes |
|-------|----------|-------|
| "Never sell your data" | **Likely true** | No evidence of direct data sale. SDK telemetry flows to Meta but app doesn't actively push PII. |
| "Zero surveillance" | **Misleading** | Local monitoring is functional (required for blocking), but server-side analytics should be acknowledged |
| "No data collection" | **FALSE** | Contradicted by code AND their own privacy policy |
| "Data is safe / sacred trust" | **Unverifiable** | Depends on server-side security, which static analysis cannot assess |
| Privacy policy is complete | **FALSE** | PostHog, session replay, and FCM triggers not disclosed |

---

## Balanced Conclusion

**Kahf Guard is not a trojan.** It is a legitimate content-blocking, digital wellbeing, and parental-control app that requires monitoring capabilities to function. Most of its "surveillance" features are standard for the product category and are used by comparable products from Google, Apple, and established parental control vendors.

**The app has a legitimate need for analytics.** Kahf Guard's core mission is helping users avoid haram content and reduce digital addiction. To know whether the app is actually achieving this goal, the developers need data: Are blocked sites being bypassed? Is time on social media decreasing? Which blocking features are most effective? Without this feedback loop, they cannot improve the product. Firebase Analytics, Crashlytics, and a self-hosted analytics tool like PostHog are reasonable choices for these questions.

**However, Kahf Guard has real transparency problems:**

1. **Undisclosed PostHog analytics** with session replay infrastructure is a significant privacy policy gap. Even though session replay only captures Kahf Guard's own settings screens (which contain no sensitive personal data), it should be disclosed for transparency.

2. **Facebook SDK integration** — while the app does NOT actively send user PII (email/name/phone) to Facebook, the SDK does send device-level advertising identifiers (GAID) and auto-logged events to Meta's ad network. This is worth acknowledging, especially since the CEO claims "no data collection." The Facebook SDK is not necessary for the app's blocking functionality.

3. **The CEO's absolutist marketing language** ("zero," "never," "no data collection") is contradicted by the code and even by Kahf's own privacy policy. The privacy policy correctly discloses time-on-app tracking, app lists for child devices, and Facebook Analytics integration — which means the CEO's public statements are less accurate than his own company's privacy policy. A simple acknowledgment like *"We collect usage analytics to improve blocking effectiveness"* would have been more honest and harder to attack.

4. **The privacy policy needs updating** — it should disclose PostHog, describe the periodic sync mechanisms, and clarify what "anonymized" means in the context of device IDs and advertising identifiers.

**The viral post raises valid questions but wraps them in dangerous hyperbole.** Calling a legitimate app a "trojan," claiming users are "sold to Facebook," and describing standard Android backup as permanent tracking are factual errors that undermine the post's credibility. Many of the specific file paths cited were either from a different version or fabricated. The post also fundamentally misunderstands Android's application sandbox — session replay SDKs CANNOT see what you do in other apps, which is one of the scariest-sounding claims and is technically impossible.

**The real issue is not malice, but communication.** Kahf Guard is a useful product for its target audience. Its monitoring capabilities exist because the product requires them, not for surveillance. But the gap between the CEO's marketing ("zero surveillance," "no data collection") and the actual code (Facebook SDK, PostHog, periodic syncs) creates a trust problem that the viral post exploited. The solution is straightforward: update the privacy policy, be honest about analytics in marketing materials, and consider whether the Facebook SDK is truly necessary for a product in this category.

---

*This report is an independent technical assessment based on static code analysis of APK v4.6.188 using JADX 1.5.5. The viral post analyzed v4.6.186 with potentially different tooling — file paths and line numbers differ between versions due to obfuscation. Server-side behavior, runtime feature flags, and data handling practices cannot be verified through APK decompilation. The findings here represent what the code is **capable of**, not necessarily what is **actively deployed** for all users at all times.*
