# CallBeacon — Technical Documentation

**Automatic Missed-Call SMS Responder for Android**
*"Unavailable, not misunderstood."*

> A native Android application (Java) that detects a missed call in real time and automatically sends the caller a personalized SMS explaining the user's current status — driving, sleeping, in a meeting, or otherwise unreachable — with zero manual effort and zero cloud dependency.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Technology Stack](#2-technology-stack)
3. [Architecture](#3-architecture)
4. [Feature Breakdown](#4-feature-breakdown)
5. [Permissions & Justification](#5-permissions--justification)
6. [Application Workflow](#6-application-workflow)
7. [Engineering Challenges & Solutions](#7-engineering-challenges--solutions)
8. [Key Design Decisions](#8-key-design-decisions)
9. [Security & Privacy](#9-security--privacy)
10. [Project Structure](#10-project-structure)
11. [FAQ](#11-faq)
12. [Future Improvements](#12-future-improvements)

---

## 1. Project Overview

### 1.1 Why CallBeacon Was Built

Android has never had a built-in way to auto-reply to a **missed phone call** — the way WhatsApp lets a user set an "away" status for chats. If someone calls you while you're driving, sleeping, or in a meeting and you can't answer, the caller is simply left guessing, and you're left with a mental backlog of people to text back once you're free.

CallBeacon closes that gap: set a status once, and the app takes care of replying to every missed call on your behalf, automatically, in the background, without the app even being open.

### 1.2 Problem Statement

| Problem | Consequence |
|---|---|
| A missed call gives the caller no context | Anxiety, repeated call-backs, miscommunication |
| Manually texting back every missed caller | Tedious, easy to forget, doesn't scale past a few calls a day |
| No native Android "away status" for calls | Users have no built-in tool to solve this — third-party automation is the only option |

### 1.3 Target Users

- Professionals who are frequently in meetings or unable to answer calls during work hours
- Drivers who cannot safely respond to calls or texts while on the road
- Anyone who wants a lightweight, private, offline way to manage missed-call communication without relying on a cloud service or a third-party server holding their call data

### 1.4 Elevator Pitch

> CallBeacon watches the device's call log in the background and automatically sends a personalized, multi-language SMS status update to anyone whose call was missed — with support for manual status, time-based schedules, and automatic status detection (low battery, flight mode, driving) — while keeping all user data encrypted and fully on-device.

---

## 2. Technology Stack

CallBeacon is a **hybrid-style native app**: everything that touches the Android OS (permissions, background services, telephony, storage) is 100% native Java. Everything the user sees and taps is HTML/CSS/JavaScript rendered inside a native `WebView`. No cross-platform framework (Cordova, Ionic, React Native, Flutter) is involved — the WebView bridge is written entirely by hand.

### 2.1 Languages

| Layer | Language | Location |
|---|---|---|
| App logic / OS integration | Java 17 | `app/src/main/java/**` |
| User Interface | HTML5, CSS3, Vanilla JavaScript (ES6) | `app/src/main/assets/www/**` |
| Build configuration | Groovy (Gradle DSL) | `build.gradle` |

### 2.2 Platform & Build

| Tool | Version | Purpose |
|---|---|---|
| compileSdk / targetSdk | 34 (Android 14) | Compiled and tested against the latest Android APIs |
| minSdk | 26 (Android 8.0 Oreo) | Minimum supported Android version |
| JDK | 17 | Compiler toolchain |
| Build system | Gradle (Groovy DSL) | Dependency management & build variants |

### 2.3 Key Dependencies

| Library | Purpose |
|---|---|
| `androidx.appcompat`, `material`, `constraintlayout` | Base UI components and theming |
| `androidx.activity` | Modern `ActivityResultContracts` permission API |
| `androidx.lifecycle` | Lifecycle-aware Activities/Services |
| `androidx.work:work-runtime` | WorkManager — guaranteed scheduled background jobs |
| `androidx.security:security-crypto` | `EncryptedSharedPreferences` — AES-256 on-device encryption |
| `com.google.code.gson` | Serializes Java objects (call logs, stats, settings) to/from JSON |
| JUnit 4 / Espresso | Unit and instrumented UI testing |

### 2.4 Data & Storage

- **`EncryptedSharedPreferences`** (AES-256-GCM for values, AES-256-SIV for keys) is the single source of truth for all app data.
- **No SQLite, no Room, no backend server** — the app is fully local and offline by design (see [Section 8](#8-key-design-decisions) for reasoning).
- `android:allowBackup="false"` — sensitive data is explicitly excluded from Android's automatic cloud backup.

### 2.5 Localization

12+ languages (English, Hindi, Marathi, Tamil, Bengali, Telugu, Gujarati, Punjabi, Urdu, Kannada, Malayalam, French, Arabic), each with 4 SMS tones (Formal, Friendly, Brief, Hinglish-mix) and a "no time mentioned" variant of each.

---

## 3. Architecture

### 3.1 Two-Layer Design

```
┌─────────────────────────────────────────────────────────┐
│                      MainActivity                        │
│  ┌───────────────────────┐      ┌──────────────────────┐ │
│  │   Native Android Layer │◄────►│   WebView Frontend    │ │
│  │   (Java)                │      │  (HTML / CSS / JS)   │ │
│  │                          │      │                       │ │
│  │ • Services               │      │ • 11 in-app screens  │ │
│  │ • Broadcast Receivers    │      │ • State management   │ │
│  │ • Permissions            │      │ • Rendering/routing  │ │
│  │ • SMS / Call Log APIs    │      │ • i18n (12 languages)│ │
│  │ • Encrypted storage      │      │                       │ │
│  └───────────┬───────────────┘      └──────────┬───────────┘ │
│              │        WebAppInterface.java                  │
│              │     (@JavascriptInterface bridge)             │
│              └───────────────window.Android──────────────────┘
└─────────────────────────────────────────────────────────┘
```

- **Native Android Layer (Java):** owns everything the OS controls — permissions, background services, reading the call log, sending SMS, listening to system broadcasts (boot, battery, Bluetooth), job scheduling, and encrypted storage.
- **WebView Frontend (HTML/CSS/JS):** owns everything the user sees and taps — onboarding, home screen, status picker, settings, call log, statistics, language picker, etc.
- **The Bridge — `WebAppInterface.java`:** a Java class annotated with `@JavascriptInterface`, injected into the WebView as the global JavaScript object `window.Android`. It is the *only* communication channel between the two layers.

### 3.2 Why a Hybrid (WebView) UI Instead of Pure Native XML?

- The app has 11 distinct screens with a custom design system, dark mode, and animations — iterating on that in HTML/CSS is significantly faster than building and maintaining 11 separate Android Activities/Fragments with XML layouts.
- A single codebase renders every screen inside one `WebView`, instead of duplicating navigation and lifecycle logic across many Activities.
- Because every OS-level task (permissions, services, SMS, encryption) still runs as genuine native Java code, there is **no performance trade-off for background work** — the WebView is responsible only for the visible UI.

### 3.3 High-Level Data Flow — "A Missed Call Happens"

```
1. Caller calls the phone; the user cannot pick up.
2. The call ends → Android writes a new "missed" row into the system Call Log.
3. MissedCallService's ContentObserver on CallLog.Calls.CONTENT_URI fires instantly.
4. After a short buffer delay, CallLogHelper queries the newest missed call
   (number + timestamp) and resolves it to a saved contact name.
5. The service checks: Is a status active? Is auto-reply enabled?
   Does the "who gets an SMS" rule (Everyone / Contacts Only / No One) allow this caller?
6. SmsHelper builds the message from the correct language + tone template
   in SmsTemplates.java, filling in {name}, {status}, {time}.
7. SmsHelper calls Android's SmsManager to send the SMS via the device's own SIM,
   registering PendingIntents to detect success or failure.
8. The event is saved to encrypted storage, a notification is shown, and a
   broadcast tells the open WebView UI to refresh the Calls tab live.
```

---

## 4. Feature Breakdown

| Feature | What It Does |
|---|---|
| **Automatic Missed-Call SMS Reply** | Detects a missed call in real time via a `ContentObserver` on the system Call Log and sends a status SMS automatically — no user action required. |
| **Manual Status Setting** | User picks a status (Driving, Sleeping, In a Meeting, etc.) with an icon and an optional "available after" time. |
| **Custom SMS Templates** | 4 tones per language (Formal, Friendly, Brief, Hinglish-mix), plus a fully custom, user-written template. |
| **Time-Based Scheduling** | Statuses can be scheduled to auto-activate and auto-clear at specific times, with repeat rules (once, daily, weekdays, weekends), powered by `WorkManager`. |
| **Auto-Detect Status** | Automatically applies a status based on real device conditions — battery below 5%, airplane mode on, or Bluetooth car-kit connected — without the app being opened. |
| **Do Not Disturb (DND) Hours** | A configurable time window (e.g., 10 PM–7 AM) during which a default "Sleeping" status auto-applies, even without a manual status set. |
| **Who Gets an SMS** | Controls whether auto-replies go to Everyone, Contacts Only, or No One. |
| **Call Log & History** | Shows every missed call handled, whether the SMS succeeded, and the exact message sent — stored locally and encrypted. |
| **Callback Requests** | Flags specific missed calls as "needs a callback," tracked and cleared from a dedicated tab. |
| **Usage Statistics** | Tracks total missed calls handled, total SMS sent, a weekly activity chart, and a daily usage streak. |
| **Multi-Language UI + SMS** | 12 supported languages for both the app interface and the outgoing SMS text. |
| **Dark Mode** | Full dark theme applied instantly across the WebView UI via CSS variables. |
| **Privacy Controls** | Phone number masking in the UI/log, auto-delete of old entries, and a one-tap "Reset App" that wipes all encrypted data. |
| **Background Service** | A persistent foreground service (`MissedCallService`) keeps the call-log watcher alive even when the app is closed, and restarts itself after a device reboot. |
| **Backup & Restore Behavior** | `android:allowBackup="false"` is set intentionally — sensitive call/SMS data is deliberately excluded from Android's automatic cloud backup rather than silently synced off-device (see [Section 9](#9-security--privacy)). |
| **Deep Link Support** | A registered `callbeacon://open` custom URI scheme allows the app to be opened directly from other apps or links. |

---

## 5. Permissions & Justification

| Permission | Required? | Why It's Needed |
|---|---|---|
| `READ_CALL_LOG` | Required | Detects when a call was missed by reading the system call log. This is a Google Play *sensitive permission* — publishing to the Play Store requires a Permissions Declaration Form justifying its use. |
| `SEND_SMS` | Required | Sends the automatic status SMS to the caller. |
| `READ_CONTACTS` | Required | Displays the caller's saved name instead of a raw number, and enables the "Contacts Only" reply filter. |
| `POST_NOTIFICATIONS` (Android 13+) | Required | Shows the persistent foreground-service notification and delivery-status alerts. |
| `FOREGROUND_SERVICE` / `FOREGROUND_SERVICE_DATA_SYNC` | Required | Legally runs a long-lived background service that keeps monitoring the call log. |
| `RECEIVE_BOOT_COMPLETED` | Required | Restarts the monitoring service automatically after the phone reboots. |
| `READ_CALENDAR` | Optional | Auto-detects an "In a Meeting" status from calendar events. |
| `BLUETOOTH` / `BLUETOOTH_CONNECT` | Optional | Auto-detects "Driving" when the phone connects to a car's Bluetooth. |
| `VIBRATE` | Optional | Notification feedback. |
| `WAKE_LOCK` | Optional | Lets the service act immediately even if the CPU was asleep. |
| `SCHEDULE_EXACT_ALARM` / `USE_EXACT_ALARM` | Optional | Precise time-based status scheduling. |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | Optional | Lets the user exempt the app from battery optimization so the background service stays alive reliably. |
| `INTERNET` / `ACCESS_NETWORK_STATE` | Reserved | Declared for a possible future web-status-sync feature; **not used for any network call today**. |

---

## 6. Application Workflow

1. **Onboarding** — the user grants permissions via `ActivityResultContracts.RequestMultiplePermissions()`, and is asked to exempt the app from battery optimization (with a clear explanation dialog).
2. **Set a Status** — the user picks a status and, optionally, an "available after" time, or lets a schedule/auto-detect rule set it automatically.
3. **Background Monitoring** — `MissedCallService` runs as a foreground service, holding a `ContentObserver` on the Call Log at all times, even with the app closed.
4. **Detection & Reply** — a missed call is detected, the message is built from the correct language/tone template, and `SmsManager` sends it through the device's own SIM.
5. **Feedback Loop** — the result (sent/failed) is written to encrypted storage, a notification is shown, and — if the app happens to be open — the WebView UI refreshes live via a broadcast.
6. **Resilience** — `BootReceiver` restarts the service after a reboot; `MainActivity.onResume()` always re-reads the latest status from encrypted storage, so a scheduled change made while the app process was killed is never missed.

---

## 7. Engineering Challenges & Solutions

These are real bugs found and fixed during development — each one reflects a genuine Android platform quirk rather than a textbook example.

### 7.1 Scheduled Status Never Activated
**Problem:** Users could add a schedule and see it in the list, but the status never actually activated at the scheduled time.
**Root Cause:** The `WorkManager` `Worker` class had mistakenly been declared as a `<service>` tag in the manifest — which does nothing for WorkManager, since Workers are not Android Services and require no manifest declaration.
**Fix:** Built `ScheduleManager.java`, which correctly computes the delay and calls `WorkManager.enqueueUniqueWork(...)` to schedule the job for real.

### 7.2 App Stuck on "Requesting…" After Reset
**Problem:** After a user reset the app's data and re-granted permissions, the UI would get stuck showing "Requesting…" forever.
**Root Cause:** Android's permission-result callback only includes entries for permissions requested in that specific call. Already-granted permissions are simply missing from the results map, and checking `Boolean.TRUE.equals(null)` silently evaluates to `false` — making an already-granted permission look denied.
**Fix:** Permission state is now checked directly against the live OS state with `ContextCompat.checkSelfPermission()` instead of trusting the results map alone.

### 7.3 Invisible SMS Failures
**Problem:** If an SMS failed to send (no signal, radio off, carrier rejection), the app assumed success simply because `sendTextMessage()` didn't throw an exception.
**Fix:** Added a "sent" `PendingIntent` pointing to `SmsDeliveryReceiver`, which reads the actual `SmsManager` result code and broadcasts a real failure event the UI can display.

### 7.4 Home Screen Not Updating After a Scheduled Change
**Problem:** If a `WorkManager` job fired a status change while the app's process was already killed in the background, the broadcast telling the UI to refresh was lost.
**Fix:** `MainActivity.onResume()` now unconditionally re-reads the status from encrypted storage and re-injects it into the WebView every time the app comes to the foreground, instead of relying solely on the broadcast.

### 7.5 WebView Viewport Height Collapsing
**Problem:** CSS's `100dvh` unit sometimes collapsed to the content's height instead of the real screen height inside Android's WebView.
**Fix:** A small JS utility measures `window.innerHeight` directly and stores it in a CSS variable, re-measuring on resize, orientation change, and shortly after page load.

### 7.6 Background Execution Restrictions (the biggest overall challenge)
Since Android 8+, and especially on manufacturer skins (Xiaomi, Oppo, Vivo), the OS aggressively kills background services to save battery — which would silently break missed-call detection with no visible error. This was solved in layers:
- `MissedCallService` runs as a **foreground service** (`startForeground()`) so the OS treats it as higher priority.
- It returns `START_STICKY` so Android restarts it if killed.
- `BootReceiver` restarts it after a device reboot.
- The app explicitly asks the user to exempt it from battery optimization, with a dialog explaining why.

---

## 8. Key Design Decisions

**Why Java?**
Java is the long-established, fully-supported language for native Android development with mature tooling, first-class access to every Android system API (Services, ContentObservers, SmsManager, WorkManager), and the most predictable behavior for long-running background components — which is the core requirement of this app.

**Why no SQLite / Room database?**
The data involved is small and simple: a handful of single values (current status, settings) and short JSON lists (call log capped at 200 entries, schedules, callback requests). `EncryptedSharedPreferences` with Gson-serialized JSON was simpler to implement, sufficiently fast at this scale, and came with built-in AES-256 encryption out of the box — avoiding the overhead of building and maintaining a schema, migrations, and a custom encryption layer for data that doesn't need relational querying. A migration to Room is a planned improvement once call history volume grows (see [Section 12](#12-future-improvements)).

**Why local processing, and why no backend?**
Missed-call and SMS content is inherently sensitive personal data. Keeping detection, message-building, and sending 100% on-device removes an entire category of risk — there is no server to breach, no data in transit, and no dependency on network availability for a feature that must work reliably in low-connectivity situations like driving. The `INTERNET` permission is declared only in reserve for a possible future opt-in web-status-sync feature; it is not used by any current feature.

---

## 9. Security & Privacy

CallBeacon is built around an **offline-first, on-device-only** data model:

- **Offline:** The UI is loaded locally from `file:///android_asset/www/index.html`, bundled inside the APK — no server is contacted to render or operate the app.
- **No cloud:** All data — user name, status, call history, settings, statistics — is stored only on the device using `EncryptedSharedPreferences` (AES-256-GCM for values, AES-256-SIV for keys), backed by a `MasterKey` stored in the Android Keystore. If encryption setup fails (e.g., on certain emulators), the app gracefully falls back to plain `SharedPreferences` rather than crashing.
- **No analytics:** There is no analytics or telemetry SDK integrated into the app.
- **No SMS upload:** Outgoing message content is built and sent entirely on-device; message text is never transmitted anywhere except directly to the caller via the SIM's own SMS channel.
- **No call upload:** Call log data read from the system provider never leaves the device — it is read, processed, and stored locally only.

Additional measures:
- `android:allowBackup="false"` excludes sensitive data from Android's automatic cloud backup.
- Phone numbers are masked in debug logs (e.g., `98765***10`), with an optional setting to mask numbers in the visible call-log UI as well.
- A one-tap **"Reset App"** action wipes all encrypted data completely (`AppPrefs.clearAll()`).
- `WebView` debugging is explicitly disabled in production builds.

---

## 10. Project Structure

```
com.callbeacon.app/
├── activities/
│   ├── MainActivity.java        — hosts the WebView, permission flow, bridge registration
│   └── SplashActivity.java      — brief splash screen before MainActivity launches
├── services/
│   ├── MissedCallService.java   — foreground service; ContentObserver on the Call Log
│   ├── SmsSenderService.java    — sends a one-off SMS on demand
│   ├── AutoDetectService.java   — applies status based on battery/airplane/Bluetooth events
│   ├── ScheduleWorker.java      — activates a scheduled status (WorkManager Worker)
│   └── ScheduleClearWorker.java — clears a scheduled status at its end time
├── receivers/
│   ├── BootReceiver.java        — restarts the service after reboot
│   ├── MissedCallReceiver.java  — detects the RINGING→IDLE-without-OFFHOOK pattern
│   ├── SmsDeliveryReceiver.java — reads SMS sent/delivered result codes
│   └── SystemStateReceiver.java — forwards battery/airplane/Bluetooth events
├── utils/
│   ├── AppPrefs.java            — single access point for encrypted storage
│   ├── CallLogHelper.java       — queries the Call Log, resolves contact names
│   ├── SmsHelper.java           — builds and sends the SMS text
│   ├── SmsTemplates.java        — source of truth for all SMS text (12 languages × 4 tones)
│   ├── ScheduleManager.java     — computes delays and enqueues WorkManager jobs
│   ├── NotificationHelper.java  — builds all notification types
│   └── WebAppInterface.java     — the JS ↔ Java bridge (@JavascriptInterface)
└── assets/www/
    ├── index.html                — 11 in-app screens as HTML sections
    ├── css/style.css             — custom design system, dark mode support
    ├── js/app.js                 — state management, routing, rendering
    └── js/i18n.js                — translation dictionary for 12 languages
```

---

## 11. FAQ

**Is this app fully functional, or a prototype?**
It's a complete, working Android app with a release build configuration. The core pipeline — call-log detection, SMS sending, permissions, background service, and scheduling — is fully implemented and testable on a real device (API 26+). Reading the call log and sending SMS require a real modem/SIM, so full end-to-end testing isn't possible on an emulator alone; a "Simulate Missed Call" mode is built into the UI for safe demoing without a real incoming call.

**Is there a backend server or API?**
No. CallBeacon is completely local. There is no database server and no network dependency for its core functionality.

**Does it use React Native, Flutter, or Cordova?**
No. It's a hand-built hybrid app: a native Android WebView with a custom JavaScript bridge, and plain vanilla JavaScript (no framework) on the frontend.

**Isn't exposing a JavaScript interface a security risk?**
It can be, if a WebView loads untrusted remote content — any script on that page would gain access to the exposed native methods. CallBeacon's WebView only ever loads a fixed local HTML file bundled inside the APK; it never loads a remote URL into the bridge-enabled page, and WebView debugging is disabled in production.

**Can this scale to many users?**
As a fully local, on-device app with no backend, there are no server-scaling concerns — each user's data lives only on their own phone. The practical considerations at scale are Play Store's sensitive-permission review for `READ_CALL_LOG`, device fragmentation across manufacturers' battery-management behavior, and eventually migrating from JSON-in-`SharedPreferences` to Room as call history grows.

**What are the current limitations?**
It's Android-only (iOS does not permit this kind of call-log/SMS background access). It depends on the device's own SIM for sending SMS, so it's subject to carrier-level restrictions on automated messages. It also relies on the user manually exempting it from battery optimization for fully reliable background operation on stricter Android skins.

---

## 12. Future Improvements

- **Multi-SIM support** — status-based routing per SIM slot on dual-SIM devices.
- **AI-generated reply suggestions** — smarter, context-aware SMS text beyond fixed templates.
- **Wear OS integration** — set or view status directly from a paired smartwatch.
- **Cloud backup (opt-in)** — an optional, user-controlled encrypted backup/sync, kept separate from the current fully-local default.
- **Calendar-based auto-detect** — using the already-declared `READ_CALENDAR` permission to automatically apply an "In a Meeting" status.
- **Migration to Room** — moving call history and structured data from JSON-in-`EncryptedSharedPreferences` to a proper local database as data volume grows.
- **Web status-sync page** — an optional, opt-in shareable link showing a user's current status, using the already-reserved `INTERNET` permission.

---
