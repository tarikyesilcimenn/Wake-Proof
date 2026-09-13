# WakeProof

An offline alarm for heavy sleepers. The alarm keeps ringing until you provide
**physical proof** that you actually got out of bed — a QR code taped to the
bathroom mirror, an NFC sticker on the kettle, real steps, math, or a shake.

Built with Flutter for **iOS and Android**.

> **No API. No backend. No account. No tracking.**
> WakeProof writes no networking code and ships no analytics SDK, crash
> reporter, ad identifier or AI service. Release builds go further and strip the
> `INTERNET` permission entirely, so the shipped app is physically incapable of
> opening a socket. Every alarm, station, QR payload, NFC tag id and statistic
> lives in this app's private storage and is deleted with the app.

## Support

For help with WakeProof, see [SUPPORT.md](SUPPORT.md) or email
tarikyesilcimenn@gmail.com.

## Policy

Privacy Policy, Terms of Use, and Subscription Policy are available in
[POLICY.md](POLICY.md).

---

## Quick start

```bash
flutter pub get
```

```bash
flutter run
```

Requires Flutter 3.44+ / Dart 3.12+. Verified with:

```bash
flutter analyze
```

```bash
flutter build apk --release
```

Android builds need `minSdk 24` and core library desugaring — both already
configured in `android/app/build.gradle.kts`.

### iOS: two manual Xcode steps

Windows/Linux checkouts cannot edit the Xcode project, so two things must be
done once on a Mac:

1. **Alarm tones** — drag `ios/Runner/Sounds/*.wav` into the Runner target
   (Build Phases → Copy Bundle Resources). Until then iOS falls back to the
   default notification sound; in-app playback works either way because the
   same files are also Flutter assets.
2. **NFC capability** — attach `ios/Runner/Runner.entitlements` to the Runner
   target (Signing & Capabilities → *Near Field Communication Tag Reading*) and
   enable the capability on the App ID. Without it, NFC stations report
   "unsupported" and the app degrades to QR gracefully.

---

## What is implemented

| Area | Status |
| --- | --- |
| Home with next alarm, proof chain, readiness, privacy promise | Done |
| Create/edit alarm: time, repeat days, label, sound, snooze mode, vibrate | Done |
| Mission chain builder with drag-to-reorder and per-mission config | Done |
| QR stations: local generation, naming, print / save / share as PDF | Done |
| NFC stations: tag registration by hardware id, graceful fallback | Done |
| Ringing screen: full-screen, looping tone, hold-to-emergency-stop | Done |
| Missions: QR (live camera), NFC, Steps, Math, Shake | Done |
| Wake Check: fires N minutes after dismissal, timed, breaks streak on fail | Done |
| Success screen: clear time, steps, streak, next alarm preview | Done |
| Insights: streak calendar, averages, missed alarms, emergency stops | Done |
| Alarm readiness: notifications, exact alarm, battery, camera, motion, NFC | Done |
| Local persistence for alarms, stations, attempts, settings | Done |
| Light + dark theme ported from the design mockup | Done |

### Deliberately not built

* **iOS AlarmKit** (iOS 26+) — see limitations below. The scheduler is behind an
  interface so it can be added without touching any UI code.
* **Gallery QR import** — omitted on purpose. A saved photo of your bathroom QR
  would defeat the entire product.
* **Cloud backup / sync** — there is nowhere to sync to, by design.

---

## Competitive position

The category leader is **Alarmy** (Delight Room, 243K iOS ratings, 4.8, ~#26
Lifestyle, marketed at 75M–100M users). Its App Store pitch is *"You'll wake up.
Always."*, illustrated by a stock Clock app with five stacked alarms next to one
Alarmy alarm, and *"Beat a mission to stop your alarm"* over a grid of missions:
math, memory grid, household item hunt, shake, photo, QR/barcode, typing, steps,
squats. It also sells sleep tracking, snore recording and sleep sounds on a
subscription. Newer entrants like **AlarmK** are building on iOS 26 **AlarmKit**,
which gives them a real system alarm — the exact thing this README documents
WakeProof cannot guarantee on iOS today.

*Caveat on method: no iOS device or Mac was available, so this is based on App
Store listings, marketing screenshots, public reviews and the vendors' own help
docs — not a hands-on teardown of the running apps.*

What that analysis changed here:

| Their approach | What WakeProof does instead |
| --- | --- |
| Mission **variety** is the headline — more missions implies a better alarm | Missions are ranked by *what they prove*. A chain of math + shake is labelled **Weak** with "Every mission here can be done lying down. Loud, but not proof." |
| Missions presented as a flat menu of equals | The picker is grouped into "Proves you got up", "After the chain", and "Wakes your brain, not your body", and each row states how it can be cheated |
| Loudness as the differentiator ("End of the World", "Highway Emergency") | Tones are grouped by intensity (Gentle / Firm / Harsh) with a visible strength meter, so you can pick without auditioning all of them at 1am |
| Ads + subscription; reviewers report paying twice and ads that take several taps to dismiss | No purchases, no ads, no accounts |
| Sleep tracking and snore **recording** — a microphone running all night, tied to an account | No microphone permission at all, no account, no network |

The **proof strength meter** (`lib/missions/proof_strength.dart`) is the core of
this positioning. It scores a chain 0–100 by how hard it is to beat *without
leaving the bed*, and hard-caps any chain with no station and no steps at 24 —
because no quantity of arithmetic proves you stood up, and grading it higher
would be the same overclaim the category is built on. It also names the single
highest-value fix ("Add a QR or NFC station far from the bed"). Covered by
`test/proof_strength_test.dart`.

## Architecture

```
lib/
├── main.dart                     entry point, portrait lock
├── app.dart                      MaterialApp, theme, alarm event host
├── core/
│   ├── theme/                    WpPalette (design tokens), WpTheme, type ramp
│   └── utils/                    id generation, time/duration formatting
├── data/
│   ├── models/                   Alarm, Mission, ProofStation, AlarmAttempt,
│   │                             StreakStats, AppSettings
│   ├── storage/                  LocalStore (JSON file) + repositories
│   └── app_state.dart            single ChangeNotifier source of truth
├── missions/
│   ├── mission_catalog.dart      icons, copy and metadata per mission type
│   ├── mission_engine.dart       drives the chain from ringing to silence
│   └── math_problem.dart         local problem generator
├── services/
│   ├── alarm_scheduler.dart      platform-independent scheduling contract
│   ├── notification_alarm_scheduler.dart   the current implementation
│   ├── capability_service.dart   permission + hardware readiness checks
│   ├── alarm_audio.dart          looping alarm playback with alarm-stream audio
│   ├── alarm_sounds.dart         bundled tone catalogue
│   ├── step_counter.dart         pedometer with accelerometer fallback
│   ├── shake_detector.dart       peak-detection shake counter
│   ├── nfc_service.dart          Core NFC / Android NFC tag id reader
│   └── qr_export_service.dart    on-device PDF for print/save
└── ui/
    ├── widgets/                  WpCard, WpStepRow, WpTile, WpStat, WpHoldButton…
    └── screens/                  shell, home, editor, proof library, stations,
                                  ringing, missions/, success, wake check,
                                  insights, readiness
```

### Data model

```dart
Alarm         id, hour, minute, repeatDays(ISO 1–7), missions[], label,
              soundId, snoozeMode, vibrate, enabled
Mission       id, type(qr|nfc|steps|math|shake|wakeCheck), stationId,
              targetSteps, mathProblems, difficulty, shakeCount,
              wakeCheckDelayMinutes
ProofStation  id, kind(qr|nfc), name, secret, createdAt, note
AlarmAttempt  id, alarmId, firedAt, dismissedAt, emergencyStopped, snoozeCount,
              stepsCompleted, completedMissionIds[], wakeCheck, missed
StreakStats   derived: currentStreak, bestStreak, onTimeRate, averageWakeMinutes,
              averageProofDuration, missedAlarms, emergencyStops, completedDays
```

Storage is one JSON document in the app's documents directory, written
atomically (temp file + rename). A corrupt file is discarded rather than
allowed to block the alarm.

### The proof flow

1. The scheduler fires (OS notification, or the in-app due-check poll when the
   app is open).
2. `AppState.beginAttempt` opens an `AlarmAttempt` and the full-screen
   `RingingScreen` takes over. Audio starts on the alarm audio stream and the
   screen is kept awake.
3. `MissionEngine` hands out one mission at a time. Each mission screen owns its
   sensor and is the only thing that can mark that mission complete. Back
   navigation is blocked (`PopScope(canPop: false)`).
4. When the chain is empty the attempt is closed, the tone stops, and — if the
   alarm has a Wake Check — a second alarm is armed for N minutes later.
5. **Emergency stop** is always on screen but needs a three second hold. It ends
   the alarm immediately, is recorded, and breaks the streak.

### Anti-cheat choices

* QR payloads are `wakeproof://station/<id>#<128-bit local secret>` — a random
  QR cannot satisfy a mission, and the code cannot be reproduced by another
  install.
* Only live camera frames are accepted; there is no gallery path.
* NFC compares the tag's hardware id, so a copied NDEF record is not enough.
* Steps prefer the hardware pedometer; the accelerometer fallback only counts
  proper peak-to-trough cycles at a walking cadence.
* Shake and Math are labelled in-app as brain/backup missions, not proof —
  because they can be done from bed and the app says so.

---

## Platform behaviour and known limitations

### Android — reliable

WakeProof schedules through `AlarmManager.setAlarmClock()` (via
`flutter_local_notifications`' `AndroidScheduleMode.alarmClock`), the same
mechanism the system clock app uses. It is exempt from Doze, shows the system's
next-alarm indicator, and carries:

* a **full-screen intent** so the alarm UI appears over the lock screen,
* `FLAG_INSISTENT` so the tone repeats until handled,
* a per-tone notification channel with `USAGE_ALARM` audio attributes,
* `showWhenLocked` / `turnScreenOn` on the activity.

Two OS settings still decide whether it works, and both are surfaced on the
**Setup → Alarm readiness** screen with one-tap fixes:

* **Exact alarm permission** (`SCHEDULE_EXACT_ALARM`, Android 12+). Without it
  Android delays alarms by minutes.
* **Battery optimisation exemption.** Some vendor skins (Xiaomi, Huawei, Oppo,
  Samsung's aggressive modes) will still kill background apps; the app warns
  rather than pretending otherwise.

### iOS — honestly limited

**A third-party iOS app cannot register a true system alarm before iOS 26 /
AlarmKit.** This is an Apple restriction, not something a library can work
around. What WakeProof actually does on iOS:

* schedules a **time-sensitive local notification** with a bundled sound;
* rings on the lock screen, but iOS caps notification sound length (~30 s) and
  will not force the app to the foreground;
* plays the looping tone once the user opens the notification, using the
  `playback` audio category so silent mode does not mute it.

Consequences to be aware of:

* If the app is **force-quit**, iOS may not deliver the notification reliably.
* **Focus / Do Not Disturb** must allow WakeProof (Time Sensitive notifications
  help but are not a guarantee). Critical alerts would fix this but require a
  special entitlement from Apple, which this project does not claim to have.
* The Setup screen states all of this in the app, not just in this README.

**AlarmKit path.** `AlarmScheduler` is an abstract interface
(`lib/services/alarm_scheduler.dart`). Adding iOS 26+ support means writing a
second implementation backed by AlarmKit and choosing it at construction time in
`AppState`; no screen, model or mission code changes.

### Sensors

| Feature | Android | iOS |
| --- | --- | --- |
| QR camera scan | CameraX via `mobile_scanner` | AVFoundation via `mobile_scanner` |
| NFC | Android NFC (tag id) | Core NFC (needs entitlement) |
| Steps | `TYPE_STEP_COUNTER`, needs `ACTIVITY_RECOGNITION` | Core Motion pedometer |
| Steps fallback | Accelerometer peak detection | Accelerometer peak detection |
| Shake | Accelerometer | Accelerometer |

If a device has no step sensor or the permission is refused, the step mission
silently switches to the accelerometer counter and says so on screen. If a
device has no NFC, NFC missions are skipped rather than trapping the user awake,
and the UI recommends a QR station instead.

---

## Privacy guarantee

* **No network code.** There is no `http`, `dio`, `web_socket_channel` or
  similar dependency, and no line in `lib/` opens a connection. Grep for it.
* **`INTERNET` is stripped from release builds.** This one deserves the full
  story rather than a slogan:
  * The app's own manifest never declares it.
  * But `mobile_scanner` depends on ML Kit's barcode scanner, which transitively
    pulls in `com.google.android.datatransport`, and *that* library's manifest
    contributes `INTERNET` and `ACCESS_NETWORK_STATE` so it can upload usage
    telemetry to Google.
  * `android/app/src/release/AndroidManifest.xml` therefore removes both with
    `tools:node="remove"`. The release APK's merged manifest contains neither,
    which means no bundled library — present or future — can reach the network,
    by accident or otherwise.
  * QR scanning is unaffected: mobile_scanner uses the **bundled** on-device ML
    Kit model, not the Play Services variant that downloads one.
  * Debug and profile builds keep the permission so `flutter run`, hot reload and
    DevTools can reach the Dart VM service. Verify a release build yourself with
    `aapt dump permissions build/app/outputs/flutter-apk/app-release.apk`.
* **No analytics, no crash reporting, no advertising id, no push tokens.**
* **No AI or model inference.** QR decoding is a deterministic algorithm; NFC is
  an id comparison. Nothing is inferred about you.
* **No cloud storage.** Statistics are computed from a local JSON file.
* **Erase everything** from Setup → Privacy. There is no server copy.

Permissions requested, and why:

| Permission | Used for |
| --- | --- |
| Notifications | showing and sounding the alarm |
| Exact alarm (Android) | firing at the right second |
| Battery optimisation exemption (Android, optional) | surviving power saving |
| Camera | scanning your own QR station, live only |
| NFC | reading the id of your own tag |
| Activity recognition / Motion | counting steps for the step mission |
| Vibrate, wake lock | alarm feedback and keeping the ringing screen up |

---

## Design

**One brand, two platform grammars.** The colour identity is the provided
mockup's on both platforms — warm neutral background (`#F5F1E8`), deep green
primary (`#203F3A`), tabular numerals, controls sized for someone barely awake.
The *structure* around those colours follows whichever platform the app is
running on. Tokens live in `lib/core/theme/wp_palette.dart` as a Flutter
`ThemeExtension`, with a matching dark palette.

| | iOS (Apple HIG) | Android (the mockup) |
| --- | --- | --- |
| Navigation | `CupertinoSliverNavigationBar` — large title that collapses to an inline blurred bar on scroll, `‹ Back` chevron with label | Inline header row with brand mark and bordered icon buttons |
| Page transitions | `CupertinoPageRoute` + interactive swipe-back | Material zoom transition |
| Lists | Inset grouped sections: one rounded container, hairline separators inset past the leading glyph, Settings-style coloured glyph tiles, chevrons on navigable rows | Individually spaced cards |
| Type ramp | HIG styles — body 17, headline 17 semibold, subhead 15, footnote 13 | The mockup's 12–15pt ramp |
| Toggles | `CupertinoSwitch` | Material `Switch` |
| Multi-choice | `CupertinoSlidingSegmentedControl` | Bordered chip row |
| Time picker | `CupertinoDatePicker` wheel in a sheet with Cancel/Done | Material clock dial |
| Sheets | `showCupertinoModalPopup` with grabber, 14pt top radius | Material bottom sheet |
| Alerts | `CupertinoAlertDialog`; destructive choices use `CupertinoActionSheet` | `AlertDialog` |
| Transient message | Top banner (iOS has no snackbar) | `SnackBar` |
| Tab bar | Flat 49pt bar, hairline top border, tinted glyphs | Floating pill bar |
| Press feedback | Opacity dim | Ink ripple |
| Section headers | UPPERCASE footnote in secondary grey | Sentence case, ink coloured |
| Icons | SF Symbols-style `CupertinoIcons` | Material `Icons` |

Two icons — NFC and walking — keep the Material glyph on both platforms:
`CupertinoIcons` has no contactless or pedestrian symbol and its nearest matches
(wifi, person) read as the wrong thing. A misleading icon is worse than a
cross-platform one.

**How it works.** `lib/core/platform/wp_adaptive.dart` holds the branching
primitives (`isCupertino`, `wpRoute`, `wpSheet`, `wpConfirm`, `wpPickTime`,
`WpSwitch`, `WpSegmented`, `WpTextField`…) and the `Wp*` widgets in
`lib/ui/widgets/` branch internally. Screens are written once against those
widgets and never test the platform themselves, so there is a single screen tree
rather than two. `isCupertino` reads `Theme.of(context).platform`, so the whole
iOS design can be previewed on an Android device by setting
`debugDefaultTargetPlatformOverride = TargetPlatform.iOS` in `main()`.

Alarm tones (`assets/sounds/*.wav`) are synthesised waveforms generated for this
project — no licensed or downloaded audio is shipped.

### The energy ramp

An alarm app that feels like paperwork loses. But a bedtime screen that shouts
is worse. So the energy is placed where it helps you wake up and nowhere else:

| Moment | Feel | How |
| --- | --- | --- |
| Home, at night | Calm | Content settles in with a light stagger; the countdown switches to a live `in 12:04` in the final hour so the screen reads as armed |
| Ringing | Urgent, alive | The clock breathes, the primary action glows on a slow pulse, banked progress shows as chain dots, the mission card springs in each time the chain advances |
| Mid-mission | Momentum | Step and shake counters are big glowing rings whose numbers pop on every increment, with a haptic per unit and a stronger one at each quarter; copy keeps pace ("Past halfway", "Almost there!") |
| Mission cleared | Banked | Full-screen green wash, heavy haptic, chain dot expands |
| Chain cleared | Celebration | Hand-rolled particle burst, streak counts up from zero, stats land one after another, rising three-beat haptic, "personal best" when earned |
| Rejected | Felt, not read | The card shakes horizontally and the phone knocks twice |

**Haptics are a language, not decoration.** `WpHaptics` exposes six semantic
levels (`tap`, `tick`, `milestone`, `missionComplete`, `reject`, `celebrate`)
whose intensities are deliberately distinguishable, so a half-awake user can
tell "that key registered" from "that mission is done" without focusing their
eyes. The whole vocabulary is behind the `hapticsEnabled` setting, which now has
a toggle in Setup. Notably, the emergency stop fires `reject`, not a
celebration: bailing out is a loss and the phone says so.

**Reduce Motion is honoured everywhere.** `reduceMotion(context)` reads the
platform accessibility flag; when it is on, every animation collapses to an
instant state change and the app behaves identically. This is not optional
polish — the app runs at the moment someone is most disoriented, and large-scale
motion is genuinely unpleasant with a vestibular disorder. Verified on device by
setting `animator_duration_scale 0`.

Two deliberate non-decisions: shake progress **never decays** (punishing someone
who pauses while half-asleep is cruel, not motivating), and nothing celebrates a
missed alarm or an emergency stop.

---

## Project decisions worth knowing

* **`permission_handler` is pinned below 13.** Version 14 of its Android
  implementation compiles against SDK 37, which the current Android SDK only
  publishes as `android-37.0`; AGP cannot resolve that hash. 12.x exposes the
  same API.
* **Wake Check is post-dismissal by design.** It is not part of the chain that
  silences the alarm; it fires afterwards, on a timer, and failing it breaks the
  streak. That is the only way to catch someone who cleared the proof and went
  straight back to bed.
* **Snooze modes.** `off` removes the button entirely; `strict` allows one three
  minute snooze *and resets the whole proof chain*; `allowed` gives three nine
  minute snoozes.
* **Attempt log is capped** at 120 days so the local file cannot grow forever.
