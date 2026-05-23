
# Wiring the Motorola AI Key to take a screenshot

A pure-ADB recipe to make the Motorola AI Key (a.k.a. "Red Key") trigger a screenshot without root.
Can be extended to launch any other shourtcuts too. 

Tested on: Motorola devices running Android 16 with
`com.motorola.mykey` v2.0.0.059 + `ScreenshotTile (NoRoot)` 35.x.

---

## TL;DR

1. Install [ScreenshotTile](https://github.com/cvzi/ScreenshotTile/)

2. Open the app once; turn ON "Native method" + "Use system defaults"

3. Wire AI Key double-press to the screenshot shortcut
    ```bash
    adb shell <<'EOF'
    settings put system tap_app_quick_double 'intent:#Intent;action=com.motorola.mykey.action.SHORTCUT;launchFlags=0x10000000;S.shortcut_package_name=com.github.cvzi.screenshottile;S.shortcut_id=takeScreenshot;end'
    EOF
    ```

4. Double-press the AI Key. Screenshot taken.

The same recipe works for `tap_app_quick_single` (single press) and `tap_app_quick_press_hold` (long press) — just change the key name in step 4.

---

## Prerequisites

- A Motorola device with the AI Key (Razr 60/Edge series shipping with `com.motorola.mykey`)
- `adb` set up with USB debugging
- The phone unlocked and ADB authorized
- About 5 minutes

You do **not** need root, Magisk, Shizuku, a custom recovery, or any
APK patching.

---

## Step-by-step

### 1. Install ScreenshotTile (NoRoot)

Open source, GPL v3, by cvzi. Available on
[F-Droid](https://f-droid.org/packages/com.github.cvzi.screenshottile/),
[Play Store](https://play.google.com/store/apps/details?id=com.github.cvzi.screenshottile),
or download the APK from
[GitHub releases](https://github.com/cvzi/ScreenshotTile/releases).

```bash
adb install ScreenshotTile.apk
```

### 2. Enable its accessibility service

Either via the system UI: `Settings → Accessibility → ScreenshotTile → On`,
or via ADB in one shot:

```bash
adb shell settings put secure enabled_accessibility_services \
  com.github.cvzi.screenshottile/com.github.cvzi.screenshottile.services.ScreenshotAccessibilityService
adb shell settings put secure accessibility_enabled 1
```

On Android 13+ you may need to unlock **"Allow restricted settings"** for
the app first (Settings → Apps → ScreenshotTile → ⋮ → Allow restricted
settings) before the accessibility toggle becomes available. See the
[ScreenshotTile README](https://github.com/cvzi/ScreenshotTile#restricted-settings-only-native-method).

### 3. Configure ScreenshotTile

Open the app once. Turn on:

- ✅ **Native method** (uses the accessibility-service screenshot — same
  pipeline as the system screenshot key chord)
- ✅ **Use system defaults** (so you get the standard thumbnail + share UI)

This avoids the MediaProjection permission prompt that the Legacy method
otherwise triggers per shot.

### 4. Back up your current MyKey settings (optional but recommended)

```bash
{
  echo "#!/bin/sh"
  for key in tap_app_quick_single tap_app_quick_double tap_app_quick_press_hold \
             single_press_enable double_press_enable \
             my_key_tap_app my_key_long_press_app
  do
    val=$(adb shell settings get system "$key" | tr -d '\r')
    if [ "$val" = "null" ] || [ -z "$val" ]; then
      echo "adb shell settings delete system $key"
    else
      printf "adb shell settings put system %s '%s'\n" "$key" "$val"
    fi
  done
} > mykey_backup.sh && chmod +x mykey_backup.sh
```

To revert later: `./mykey_backup.sh`

### 5. Wire the AI Key

Pick the gesture you want:

| Setting key | Gesture |
|---|---|
| `tap_app_quick_single` | Single press |
| `tap_app_quick_double` | Double press |
| `tap_app_quick_press_hold` | Long press |

Then:

```bash
adb shell <<'EOF'
settings put system tap_app_quick_double 'intent:#Intent;action=com.motorola.mykey.action.SHORTCUT;launchFlags=0x10000000;S.shortcut_package_name=com.github.cvzi.screenshottile;S.shortcut_id=takeScreenshot;end'
EOF
```

> **Why the heredoc?** The intent URI contains semicolons, which the device
> shell treats as command separators. A heredoc (`<<'EOF'`) sends each line
> verbatim without any host-side or device-side expansion.

### 6. Verify

```bash
adb shell settings get system tap_app_quick_double
```

Expected:

```
intent:#Intent;action=com.motorola.mykey.action.SHORTCUT;launchFlags=0x10000000;S.shortcut_package_name=com.github.cvzi.screenshottile;S.shortcut_id=takeScreenshot;end
```

And check the shortcut is enabled:

```bash
adb shell dumpsys shortcut | grep -A1 takeScreenshot
```

Look for `flags=0x...` with no `Dis` (FLAG_DISABLED 0x40) bit.

### 7. Test

Double-press the AI Key. You should hear the shutter sound and see the
standard screenshot thumbnail.

Optionally watch logcat:

```bash
adb logcat -c && adb logcat -s MyKeyTriggerService:* ScreenshotAccessibilityService:*
```

Expected:

```
MyKeyTriggerService: doPressAction: action_double_press
MyKeyTriggerService: Ga key start shortcut: intent:...shortcut_id=takeScreenshot...
ScreenshotAccessibilityService: <screenshot taken>
```

---

## How it works

A small smali archaeology trip explains why this is possible.

### MyKey reads the setting at trigger time

When the AI Key is pressed, the Motorola framework binds to
`com.motorola.mykey/.service.MyKeyTriggerService` and fires an Intent with
extra `action=action_single_press` (or `_double_` / `_long_`). The service
reads the matching `tap_app_quick_*` value from `Settings.System`, calls
`Intent.parseUri()`, and dispatches by **intent action**:

```
parsedIntent.action == ?
  "com.motorola.mykey.action.SHORTCUT"          → startShortcutByGaKey()
  "com.android.systemui.screenrecord.START"     → startScreenRecordByGaKey()
  "com.motorola.mykey.action.MUSIC"             → playOrPauseMusic()
  (anything else)                               → startSpecialActivityByGaKey()
                                                  → plain startActivity()
```

Source: `MyKeyTriggerService.smali:2537–2601`.

### The shortcut branch is the key

`startShortcutByGaKey` reads two extras from the intent and calls
`LauncherApps.startShortcut()`:

```
shortcut_package_name = "com.github.cvzi.screenshottile"
shortcut_id           = "takeScreenshot"
```

Source: `MyKeyTriggerService.smali:3603`. MyKey holds
`android.permission.ACCESS_SHORTCUTS`, so the cross-package lookup works.

The Android framework grants `startShortcut()` callers a **temporary
permission** to launch the target activity even if it's `exported="false"`.
ScreenshotTile's `NoDisplayActivity` is not exported — but the static
shortcut at `shortcuts.xml` makes it reachable via `LauncherApps`, and
from there it can call the accessibility service to take the screenshot.

### Important findings

A bunch of things you might assume don't work — and one that does.

| Approach | Works? | Why |
|---|---|---|
| Direct `startActivity` to a SystemUI screenshot Activity | ❌ | No exported launchable Activity in SystemUI takes a screenshot |
| `bindService` to `com.android.systemui.screenshot.ScreenshotInputService` (the Moto Actions recipe) | ❌ from MyKey config | MyKey only does `startActivity`, never `bindService` |
| Broadcast to `SYSTEM_ACTION_TAKE_SCREENSHOT` | ❌ | `com.android.systemui.permission.SELF` is signature-only |
| Launch `AppClipsTrampolineActivity` | ❌ | `LAUNCH_CAPTURE_CONTENT_ACTIVITY_FOR_NOTE` permission is role-protected |
| Launch ScreenshotTile's `NoDisplayActivity` directly | ❌ | `exported="false"` — `SecurityException` |
| Piggyback on Moto Actions' `QuickScreenshotService` or `TakeScreenshotService` | ❌ | Both `exported="false"`, no external trigger surface |
| Re-route `EXPERIENCE` intent to a custom app | ❌ | `BIND_EXPERIENCE` is signature-only |
| Remap the AI Key to a different keycode via ADB | ❌ | Motorola framework consumes the key before app-level dispatch |
| **`LauncherApps.startShortcut` via the `com.motorola.mykey.action.SHORTCUT` dispatch branch** | ✅ | What this guide uses |

A few subtler facts that matter:

- **The MyKey UI hides "Launch app" for double/long press**, but the runtime
  has no allowlist. Any intent URI you write to `tap_app_quick_double` or
  `tap_app_quick_press_hold` via ADB will be parsed and dispatched, including
  this shortcut intent.
  Source: `MyKeyTriggerService.smali:2359` (`launchSpecialApp`) — only checks
  for empty / `"0"` (the None sentinel), then immediately calls
  `Intent.parseUri()` and dispatches.

- **The settings UI overwrites your custom intent.** If you open
  Settings → Red Key → Double press and pick anything from the candidate
  list, your ADB-set value is replaced. Avoid that tab after wiring it, or
  re-run the wire-up command to restore.

- **A `tap_app_quick_*` value of `"0"` means "no action / None"**, not a
  numeric zero. Don't confuse it with the boolean `single_press_enable` /
  `double_press_enable` toggles, which are also stored as `"0"`/`"1"` but
  control whether the gesture is even processed.

- **`Settings.System` stores intent URIs as flat strings.** The semicolons
  are part of the intent URI format, not shell separators — but both your
  host shell and the device shell will see them as separators unless you
  use proper nesting (heredoc, or outer-double / inner-single quotes).

- **3-finger swipe screenshot lives in Moto Actions**, but its dispatch
  chain is fully internal:
  `framework → QuickScreenshotService → TakeScreenshotService → SystemUI`
  — none of those have an external trigger. The only way to invoke it would
  be to fake the 3-finger gesture at the input layer, which requires
  signature permissions.

- **`com.motorola.uxcore` was the only stock no-install screenshot path.**
  If you have that app installed, the intent
  `intent:#Intent;launchFlags=0x00040000;package=com.motorola.uxcore;action=com.motorola.uxcore.action.QUICK_LAUNCH;B.MOTO_LAUNCH_WITH_SCREENSHOT=true;end`
  takes a screenshot — but also opens the Moto AI overlay. If `uxcore` is
  uninstalled (or your SKU never had it), this guide is your alternative.

---

## Reverting

```bash
./mykey_backup.sh
```

Or to undo just this one gesture:

```bash
# if your original value was something:
adb shell settings put system tap_app_quick_double 'YOUR_ORIGINAL_VALUE'

# if it was "0" / None:
adb shell settings put system tap_app_quick_double '0'

# if there was no value set at all:
adb shell settings delete system tap_app_quick_double
```

`settings delete` and `settings put ... '0'` are functionally equivalent
for these keys — both result in the gesture doing nothing.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Press does nothing, no log output | `double_press_enable` is `0` | `adb shell settings put system double_press_enable 1` |
| Log shows `Ga key start shortcut` but no screenshot taken | Accessibility service not actually running | Open ScreenshotTile, toggle accessibility off + on |
| Log shows `Invalid intent GA can't launch app` | `tap_app_quick_double` value didn't store correctly | Re-run the heredoc command; verify with `adb shell settings get system tap_app_quick_double` |
| Log shows `In MDM model, doEnterprise...` | Device enrolled with MDM that restricts the gesture | This guide can't help on managed devices |
| `/system/bin/sh: ...inaccessible or not found` when running the wire-up | Semicolons interpreted as command separators | Use the heredoc form, not single-line nested quotes |
| Settings UI shows "None" for double press but it still works | Cosmetic — the UI doesn't recognize the shortcut intent | Ignore. Don't tap the tab or your intent gets overwritten |

---

## Generalizing

The same pattern works for **any app on the device that publishes a static
or dynamic shortcut**. List candidates:

```bash
adb shell dumpsys shortcut > /tmp/shortcuts.txt
grep -B1 "ShortcutInfo " /tmp/shortcuts.txt
```

For each, identify the `packageName` and `id`. Then:

```bash
adb shell <<EOF
settings put system tap_app_quick_double 'intent:#Intent;action=com.motorola.mykey.action.SHORTCUT;launchFlags=0x10000000;S.shortcut_package_name=PACKAGE;S.shortcut_id=ID;end'
EOF
```

Useful examples found on a typical Moto device:

| App | Shortcut id | What it does |
|---|---|---|
| `com.github.cvzi.screenshottile` | `takeScreenshot` | Take a screenshot (this guide) |
| `com.github.cvzi.screenshottile` | `toggleFloatingButton` | Toggle the floating screenshot button |
| `com.google.android.apps.photos` | `manifest_view_screenshots` | Open the screenshots album |
| `com.motorola.camera5` | (DEPTH_CAPTURE shortcut) | Open camera in depth mode |
| `com.google.android.apps.docs` | `SCAN_DOCUMENT_FROM_SHORTCUT` | Drive document scanner |

You can even split gestures across different shortcuts — e.g. single press
opens camera, double press takes a screenshot, long press opens
screenshots album. All three are pure ADB.

---

## References

Decompiled smali paths (for anyone wanting to verify the dispatch logic):

| Concern | File |
|---|---|
| AI Key trigger service | `MotoMyKey/smali/com/motorola/mykey/service/MyKeyTriggerService.smali` |
| Intent-action dispatch switch | `MyKeyTriggerService.smali:2537–2601` |
| `launchSpecialApp` (no validation, just parseUri + startActivity) | `MyKeyTriggerService.smali:2359` |
| `startShortcutByGaKey` (the path this guide uses) | `MyKeyTriggerService.smali:3603` |
| All `Settings.System` keys MyKey reads | `MyKeyTriggerService$SettingsObserver.smali` |
| ScreenshotTile shortcut declaration | `https://github.com/cvzi/ScreenshotTile/blob/master/app/src/main/res/xml/shortcuts.xml` |
| ScreenshotTile manifest | `https://github.com/cvzi/ScreenshotTile/blob/master/app/src/main/AndroidManifest.xml` |
| Moto Actions screenshot pipeline (for reference, not used here) | `MotoActions/smali/com/motorola/actions/features/quickscreenshot/service/TakeScreenshotService.smali` |
| Moto Actions → SystemUI bind recipe | `MotoActions/smali/e6.1/e.smali:62–118` |

Companion reference doc with full MyKey settings catalog:
[REFERENCE.md](./REFERENCE.md)
