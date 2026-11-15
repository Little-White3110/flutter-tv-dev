---
name: flutter-tv-dev
description: Build Flutter apps for Android TV / Google TV / Fire TV — the TV manifest (leanback feature, 320x180 banner, LEANBACK_LAUNCHER), D-pad focus with the `dpad` package, 10-foot layout, TV AVD and ADB remote simulation, and Play TV release requirements. Use when the task involves Flutter on Android TV / TV box / Fire TV, D-pad or remote navigation, focus traversal or focus memory, TV overscan margins, or publishing a Flutter app as a TV app.
description_zh: "Flutter Android TV 应用开发规范：TV Manifest、D-pad 焦点、10-foot 布局、ADB 调试与上架"
description_en: "Flutter Android TV development guide (manifest, D-pad focus, 10-foot UI, ADB, release)"
license: MIT
agent_created: true
metadata:
  version: "1.0.0"
  category: mobile
  tech_stack:
    - Flutter >= 3.24
    - dpad ^3.0.0 (fluttercandies)
  sources:
    - developer.android.com/training/tv (Android TV developer docs)
    - github.com/fluttercandies/dpad (v3 README, MIT)
    - Google Play TV app quality guidelines
---

# Flutter Android TV app development

Flutter has **no official TV support**. A Flutter app becomes a TV app through three things working
together: a **TV manifest** (so the launcher recognizes it), a **D-pad focus model** (because there
is no touch), and **10-foot layout** (because the user sits three meters away). Miss any one and the
app breaks in a way that looks like a Flutter bug but isn't.

## 1. Read the right reference first

| Situation | Open |
|---|---|
| Writing focus/navigation code | `references/dpad-focus.md` |
| Manifest, AVD, ADB, package name entry, release | `references/tv-build-debug-release.md` |

## 2. The TV manifest (mandatory; nothing else matters if this is wrong)

File: `android/app/src/main/AndroidManifest.xml`

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
  <!-- required: declares this app targets TV -->
  <uses-feature android:name="android.software.leanback" android:required="true" />
  <!-- required: TV has no touchscreen; omitting this hides the app from TV users on Play -->
  <uses-feature android:name="android.hardware.touchscreen" android:required="false" />

  <application android:banner="@drawable/tv_banner">          <!-- 320x180 px, no alpha -->
    <activity
        android:name=".MainActivity"
        android:exported="true"
        android:screenOrientation="landscape"
        android:configChanges="orientation|screenSize|screenLayout|keyboard|keyboardHidden">
      <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
      </intent-filter>
    </activity>
  </application>
</manifest>
```

Banner: `android/app/src/main/res/drawable/tv_banner.png`, exactly **320x180** (16:9), no
transparency. Without `LEANBACK_LAUNCHER` the app does not appear on the TV home screen at all.

> Do **not** add a second `<category android:name="android.intent.category.LAUNCHER" />` expecting
> one APK for both: ship phone and TV as separate flavors instead. See the build reference.

## 3. Focus is the product

Flutter's default directional focus picks the **geometrically closest** widget. On real TV layouts
that means focus jumps lanes between sidebar and content, skips rows, escapes carousels, and forgets
positions. Use `fluttercandies/dpad`, which replaces the engine with the model TV platforms use
(Android `FocusFinder` / Leanback): **regions with memory** and **beam-based traversal**.

```yaml
# pubspec.yaml
dependencies:
  dpad: ^3.0.0
```

```dart
import 'package:dpad/dpad.dart';

MaterialApp(
  builder: Dpad.wrap(
    debugOverlay: kDebugMode,          // non-negotiable while developing: shows focused node live
    theme: const DpadThemeData(effects: [DpadGlowEffect()], scrollPadding: 64),
    onBack: () {
      if (navigatorKey.currentState!.canPop()) { navigatorKey.currentState!.pop(); return true; }
      showExitDialog(); return true;
    },
  ),
  home: const HomePage(),
)
```

Seven rules that keep a complex TV app predictable (full detail in `references/dpad-focus.md`):

1. **One `autofocus: true` per screen** — give it to the primary action (Play, first card).
2. **One `DpadRegion` per visual section** — sidebar, each poster row, each grid.
3. **`memoryKey` on every region inside a section switcher/tab** — plain state dies with the subtree,
   producing the classic "tabs forget where I was" bug.
4. **Padding goes inside the scroll view** so scale/glow effects are not clipped: `height: cardHeight + 32`,
   and `ListView` padding `EdgeInsets.symmetric(horizontal: 36, vertical: 16)`.
5. **`stop` edges on anchored panels, `wrap` on carousels.** A sidebar must never let focus fall off screen.
6. **Wire actions through `DpadFocusable.onSelect`, not the child's `onPressed`.** `DpadFocusable` is
   the single focus stop; a wrapped Material button is just visuals (`excludeChildFocus` defaults true).
7. **Leave `TextField`s bare** (never wrapped in `DpadFocusable`). Arrows then move the caret mid-string
   and navigate away at the edges automatically — a remote-only user is never trapped in search.

If adding `dpad` is not an option, the same guarantees must be rebuilt by hand with
`Focus` + `FocusScope` + `FocusTraversalGroup` + `Shortcuts`/`Actions`; expect to re-implement focus
memory and edge behavior. Prefer the package.

## 4. 10-foot layout numbers

| Item | Value |
|---|---|
| Design canvas | 1920x1080 (design at 1080p, provide 4K assets only for hero art) |
| Overscan-safe margin | ~5% of width per side (>= 48dp); keep focus effects inside it |
| Body text | >= 18sp; minimum legible 12sp is **not** acceptable on TV |
| Focus target | >= 48dp, with clear focused state (scale ~1.05-1.1 / border / glow) |
| Orientation | landscape only, never portrait |
| Contrast | high; assume bright room and cheap panel |
| Focus feedback | mandatory — there is no cursor or hover fallback |

Focus must always be **visible and predictable**. A remote pointing at nothing is a dead TV.

## 5. TV-specific pitfalls (each maps to a Flutter fix)

| Pitfall | Fix |
|---|---|
| App installs but not visible on TV home | missing `LEANBACK_LAUNCHER` / `android.software.leanback` |
| Play shows "not compatible with your TV" | `touchscreen` not marked `required="false"` |
| Leaving a row and returning lands on item 0 | missing `DpadRegion` / `memoryKey` |
| Focus scales but gets clipped at row edge | padding is outside the scroll view; move it inside |
| Two D-pad stops per card | `DpadFocusable` wrapping a focusable child; keep default `excludeChildFocus` |
| Focus lands nowhere after dialog close / route pop | no initial focus target; use one `autofocus` + `Dpad.wrap` scope |
| Dynamic content steals focus | guard list refreshes behind `Skeletonizer`-style placeholders; never move focus on async data arrival |
| Portrait-looking layout / relayout on TV | lock `screenOrientation="landscape"` |
| Network images pop in jankily while traversing | always provide placeholder + `fadeIn` |
| Assuming touch gestures | every gesture needs a focusable equivalent reachable by D-pad |

## 6. Debug loop

```bash
# TV emulator (once)
sdkmanager "system-images;android-36;android-tv;arm64-v8a"
avdmanager create avd -n TV_API36 -k "system-images;android-36;android-tv;arm64-v8a" -d tv_1080p
emulator -avd TV_API36 &

# Real box/TV over network
adb connect <tv-ip>:5555
adb devices

flutter run -d <tv-ip>:5555          # or the emulator id
flutter build apk --debug
adb install -r build/app/outputs/flutter-apk/app-debug.apk
adb shell am start -n com.example.tv/.MainActivity

# Simulate the remote: 19/20/21/22 = DPAD up/down/left/right, 23 = DPAD_CENTER, 4 = BACK
adb shell input keyevent 19 20 21 22
adb shell input text "query"          # search boxes
adb logcat | grep flutter
```

Desktop (`flutter run -d windows/macos/linux`) drives the same focus tree with arrow keys — the
fastest inner loop. Ship verification on a real device or TV AVD.

## 7. Verification checklist (run before "done")

- [ ] App icon appears on the TV home screen (manifest + banner correct)
- [ ] Launch focus target is the primary action; nothing is unfocusable on boot
- [ ] Every primary flow is completable with D-pad only, no touch
- [ ] Rows/carousels remember position after leaving and returning
- [ ] Overlays (dialog, drawer, menu) trap focus and restore it on dismiss
- [ ] Route changes restore a sensible focus target
- [ ] Dynamic/lazy content does not steal or drop focus
- [ ] Text fields: arrows edit caret, then navigate away at edges; no trap
- [ ] Layout survives overscan: no content clipped at any edge
- [ ] Back button: pops routes, confirms exit at root
- [ ] Verified with `debugOverlay: true` at least once

Release specifics (banner assets, TV screenshots, Play TV quality form) are in
`references/tv-build-debug-release.md`.
