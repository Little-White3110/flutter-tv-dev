# Android TV build, debug and release reference (Flutter)

## 1. Manifest

File `android/app/src/main/AndroidManifest.xml`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
  <uses-feature android:name="android.software.leanback" android:required="true" />
  <uses-feature android:name="android.hardware.touchscreen" android:required="false" />

  <application android:banner="@drawable/tv_banner">
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

- Banner: `android/app/src/main/res/drawable/tv_banner.png`, exactly **320x180** px, 16:9, no alpha.
- No `LEANBACK_LAUNCHER` -> app invisible on TV home. `touchscreen` not `required="false"` -> Play
  reports the device as incompatible.

### Separate phone and TV from one codebase

Use flavors with two entrypoints rather than one activity declaring both launcher categories:

```
lib/main.dart      -> phone
lib/main_tv.dart   -> TV
flutter run -t lib/main_tv.dart
flutter build apk -t lib/main_tv.dart --flavor tv
```

Per-flavor manifest could also live in `android/app/src/tv/AndroidManifest.xml`. Keep the TV activity
in the TV variant only.

## 2. Gradle basics

`android/app/build.gradle.kts`: `minSdk` must satisfy the Flutter/dpad floor (TV devices are
typically API 24+; verify against your plugins). No TV-specific plugin is required.

## 3. TV emulator

```bash
sdkmanager --list | grep "system-images;.*android-tv"
sdkmanager "system-images;android-36;android-tv;arm64-v8a"
avdmanager create avd -n TV_API36 -k "system-images;android-36;android-tv;arm64-v8a" -d tv_1080p
avdmanager list device            # tv_1080p, tv_4k, tv_720p
emulator -avd TV_API36 &
flutter devices
flutter run -d <emulator-id>
```

TV emulators are slow to boot; keep one warm. Desktop targets are faster for focus work.

## 4. Real device / box

```bash
adb connect <tv-ip>:5555          # TV: Settings > Device Preferences > About > Build, tap 7x for dev options
adb devices
adb install -r build/app/outputs/flutter-apk/app-debug.apk
adb shell am start -n <package>/<MainActivity>
adb logcat | grep -E "flutter|AndroidRuntime"
```

Failing screen mirroring is not a substitute — always verify on the actual remote.

## 5. Simulating the remote

```bash
adb shell input keyevent 19   # DPAD up
adb shell input keyevent 20   # DPAD down
adb shell input keyevent 21   # DPAD left
adb shell input keyevent 22   # DPAD right
adb shell input keyevent 23   # DPAD_CENTER (select)
adb shell input keyevent 4    # BACK
adb shell input keyevent 82   # MENU
adb shell input text "search+term"
adb exec-out screencap -p > tv.png     # screenshot
```

Media keys matter for playback apps: `85` play/pause, `87` next, `88` previous.

## 6. Media playback notes

Flutter TV apps commonly use `media_kit` or `video_player` (platform views on Android TV can be
janky on low-end boxes — measure early). Pair playback with `audio_service`/`MediaSession` so system
transport controls and "now playing" work. Handle HDMI/CEC standby and keep a wake lock during
playback only.

## 7. Release checklist (Google Play TV)

- [ ] Banner 320x180 with no transparency
- [ ] TV screenshots (1920x1080) uploaded in the TV track
- [ ] `android.software.leanback` + `touchscreen required="false"` declared
- [ ] Aggressive physical-table-style checklist from the TV app quality guidelines: D-pad navigation
      complete, no touch-only affordances, landscape only, legible at 10 feet, no unsupported
      hardware assumptions (no camera/telephony/GPS), proper back behavior
- [ ] No dependency on Google Play Services features unavailable on some boxes
- [ ] Fire TV / other stores: check their own banner and manifest rules; Fire OS accepts most Android
      TV APKs, Vega OS (Kepler) does **not** (different runtime, needs a separate JS-based build)
- [ ] Test on one low-end box, not just the emulator: focus + playback performance differ sharply

## 8. Reference projects worth reading

| Repo | What to copy from it |
|---|---|
| `GhostenEditor/Ghosten-Player` | large-scale Flutter TV UI structure |
| `HTWMedia/HTV` | IPTV/EPG TV layout patterns |
| `varunsalian/debrify` | Flutter across Android TV + desktop, single codebase |
| `leanbitlab-org/LtvLauncher` | Flutter TV launcher, leanback interop |
| `DevelopersCoffee/airo` | separate `lib/main_tv.dart` entrypoint + ADB run script |
