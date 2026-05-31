# Wiring the Motorola AI Key to take a screenshot

A pure-ADB recipe to make the Motorola AI Key (a.k.a. "Red Key") trigger a real system screenshot — shutter sound, thumbnail UI, save to gallery — using one open-source helper app and no root. Can be extended to launch any other shortcut too.

Tested on: Motorola devices running Android 16 with `com.motorola.mykey` v2.0.0.059 + ScreenshotTile (NoRoot).

> Requires ScreenshotTile **v2.21.0 or later** (the version that introduced the `TAKE_SCREENSHOT` activity intent). Older builds only expose the broadcast trigger or the static shortcut, neither of which fit the AI Key cleanly — see [Why this approach](#why-this-approach).

---

## TL;DR

1. Install [ScreenshotTile](https://github.com/cvzi/ScreenshotTile/) (F-Droid, Play Store, or GitHub releases).

2. Open the app once:
   - turn ON **"Native method"** + **"Use system defaults"**
   - set the **broadcast/intent password** in app settings (any string — you'll reuse it in step 3)

3. Wire the AI Key double-press to trigger a screenshot (replace `YOUR_PASSWORD`):
    ```bash
    adb shell <<'EOF'
    settings put system tap_app_quick_double 'intent:#Intent;action=com.github.cvzi.screenshottile.TAKE_SCREENSHOT;package=com.github.cvzi.screenshottile;launchFlags=0x10000000;S.secret=YOUR_PASSWORD;end'
    EOF
    ```

4. Double-press the AI Key. Screenshot taken — of the foreground app, no flash to home, app context preserved.

The same recipe works for `tap_app_quick_single` (single press) and `tap_app_quick_press_hold` (long press) — change the key name in step 3.

---

## Prerequisites

- A Motorola device with the AI Key (Razr 60 / Edge series shipping with `com.motorola.mykey`)
- `adb` set up with USB debugging
- The phone unlocked and ADB authorized
- About 5 minutes

You do **not** need root, Magisk, Shizuku, a custom recovery, or any APK patching.

---

## Step-by-step

### 1. Install ScreenshotTile (NoRoot)

Open source, GPL v3, by [cvzi](https://github.com/cvzi). Available on [F-Droid](https://f-droid.org/packages/com.github.cvzi.screenshottile/), [Play Store](https://play.google.com/store/apps/details?id=com.github.cvzi.screenshottile), or [GitHub releases](https://github.com/cvzi/ScreenshotTile/releases).

```bash
adb install ScreenshotTile.apk
```

### 2. Enable its accessibility service

Either via the system UI: `Settings → Accessibility → ScreenshotTile → On`, or via ADB:

```bash
adb shell settings put secure enabled_accessibility_services \
  com.github.cvzi.screenshottile/com.github.cvzi.screenshottile.services.ScreenshotAccessibilityService
adb shell settings put secure accessibility_enabled 1
```

On Android 13+ you may need to unlock **"Allow restricted settings"** for the app first (Settings → Apps → ScreenshotTile → ⋮ → Allow restricted settings). See the [ScreenshotTile README](https://github.com/cvzi/ScreenshotTile#restricted-settings-only-native-method).

### 3. Configure ScreenshotTile

Open the app once. Then:

- Turn on **Native method** + **Use system defaults** — accessibility-service screenshot path, no MediaProjection prompts.
- Set the **broadcast/intent password** in app settings to any string. This same password is used by both the broadcast intent and the activity intent (the one used here).

### 4. Back up current MyKey settings (optional)

```bash
{
  echo "#!/bin/sh"
  for key in tap_app_quick_single tap_app_quick_double tap_app_quick_press_hold \
             single_press_enable double_press_enable; do
    val=$(adb shell settings get system "$key" | tr -d '\r')
    if [ "$val" = "null" ] || [ -z "$val" ]; then
      echo "adb shell settings delete system $key"
    else
      printf "adb shell settings put system %s '%s'\n" "$key" "$val"
    fi
  done
} > mykey_backup.sh && chmod +x mykey_backup.sh
```

Revert any time with `./mykey_backup.sh`.

### 5. Wire the AI Key

Pick the gesture:

| Setting key | Gesture |
|---|---|
| `tap_app_quick_single` | Single press |
| `tap_app_quick_double` | Double press |
| `tap_app_quick_press_hold` | Long press |

Then (replace `YOUR_PASSWORD` with the password from step 3):

```bash
adb shell <<'EOF'
settings put system tap_app_quick_double 'intent:#Intent;action=com.github.cvzi.screenshottile.TAKE_SCREENSHOT;package=com.github.cvzi.screenshottile;launchFlags=0x10000000;S.secret=YOUR_PASSWORD;end'
EOF
```

Add `;B.partial=true` before `;end` if you want the area selector instead of a full-screen screenshot.

> **Why the heredoc?** The intent URI contains semicolons, which the device shell would otherwise treat as command separators. A heredoc (`<<'EOF'`) sends each line verbatim.

### 6. Verify

```bash
adb shell settings get system tap_app_quick_double
adb shell pm dump com.github.cvzi.screenshottile | grep -A2 ScreenshotTriggerActivity
```

The first command should echo back your intent URI exactly. The second should show the activity's intent filter for `com.github.cvzi.screenshottile.TAKE_SCREENSHOT`.

### 7. Test

Open any app and double-press the AI Key. Expected: shutter sound, standard system screenshot thumbnail, foreground app stays visible throughout, saved screenshot is of the foreground app.

Optionally watch logcat:

```bash
adb logcat -c && adb logcat -s MyKeyTriggerService:* ScreenshotTriggerAct:*
```

Expected:

```
MyKeyTriggerService: doPressAction: action_double_press
MyKeyTriggerService: launchSpecialApp - specialApp : intent:#Intent;action=com.github.cvzi.screenshottile.TAKE_SCREENSHOT;...
```

---

## Why this approach

When the AI Key is pressed, the Motorola framework binds to `com.motorola.mykey/.service.MyKeyTriggerService` and fires an Intent with extra `action=action_single_press` (or `_double_` / `_long_`). The service reads the matching `tap_app_quick_*` value from `Settings.System`, calls `Intent.parseUri()`, and dispatches by intent action:

```
parsedIntent.action == ?
  "com.motorola.mykey.action.SHORTCUT"          → startShortcutByGaKey()
  "com.android.systemui.screenrecord.START"     → startScreenRecordByGaKey()
  "com.motorola.mykey.action.MUSIC"             → playOrPauseMusic()
  (anything else)                               → startSpecialActivityByGaKey()
                                                  → plain startActivity()
```

The intent we use targets `ScreenshotTriggerActivity` — an exported activity in ScreenshotTile that:

- Validates the same `secret` extra used by the existing broadcast trigger
- Calls `App.screenshot()` / `App.screenshotPartial()` directly — same code path as the Quick Settings tile
- Uses `Theme.NoDisplay`, `taskAffinity=""`, `noHistory` so it never disturbs the foreground app's task stack

Result: clean capture of the originating app with the standard screenshot UI, no flash to home.

### What didn't work

| Approach | Outcome |
|---|---|
| `com.github.cvzi.screenshottile.SCREENSHOT` broadcast | Reachable in principle, but MyKey only does `startActivity`, never `sendBroadcast`. |
| Direct bind to `com.android.systemui.screenshot.ScreenshotInputService` | The right primitive, but MyKey can't `bindService` from config — only patching MyKey itself reaches it. |
| Make ScreenshotTile the device Voice Assistant (`MyVoiceInteractionService`) | Works perfectly, but takes over long-press-home, the power-button assist option, and every other system assist gesture. Too invasive. |
| Remap the AI Key keycode at the kernel layer | Needs root — Motorola's framework consumes the press before it reaches app-level dispatch. |

---

## Reverting

```bash
./mykey_backup.sh
```

Or undo just one gesture:

```bash
# revert to original (whatever it was before)
adb shell settings put system tap_app_quick_double 'YOUR_ORIGINAL_VALUE'

# or set to None
adb shell settings put system tap_app_quick_double '0'

# or remove the setting entirely
adb shell settings delete system tap_app_quick_double
```

`settings delete` and `settings put ... '0'` are functionally equivalent for these keys — both leave the gesture inert.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Press does nothing, no log output | `double_press_enable` is `0` | `adb shell settings put system double_press_enable 1` |
| Log shows `Wrong secret` | `S.secret=...` in the intent URI doesn't match the password set in the app | Update one to match the other |
| Log shows `Extra 'secret' is required.` | Forgot the `S.secret=` part of the URI | Re-run step 5 with the secret in place |
| Log shows `Secret was not set in the app settings.` | Never set the broadcast/intent password in ScreenshotTile, or it's still the default `(unset)` | Open ScreenshotTile → set a password |
| `ActivityNotFoundException` | ScreenshotTile too old; doesn't include `ScreenshotTriggerActivity` | Update ScreenshotTile to v2.21.0 or later |
| Log shows `In MDM model, doEnterprise...` | Device enrolled in MDM that restricts the gesture | Out of scope — managed devices block this |
| `/system/bin/sh: ...inaccessible or not found` when running the wire-up | Semicolons interpreted as command separators | Use the heredoc form, not single-line nested quotes |
| Settings UI shows "None" for this gesture but it still works | Cosmetic — MyKey's UI doesn't recognise the custom intent action | Ignore; do not tap the tab or it overwrites your value |

---

## Generalizing

This dispatch pattern works for **any app on the device that exposes an exported activity for a custom intent action**. For screenshots specifically, ScreenshotTile's `TAKE_SCREENSHOT` activity intent is the canonical recipe.

For other gestures, you can also point MyKey at any launcher shortcut on the device:

```bash
adb shell <<EOF
settings put system tap_app_quick_double 'intent:#Intent;action=com.motorola.mykey.action.SHORTCUT;launchFlags=0x10000000;S.shortcut_package_name=PACKAGE;S.shortcut_id=ID;end'
EOF
```

Enumerate available shortcuts on a device:

```bash
adb shell dumpsys shortcut | grep -B1 "ShortcutInfo "
```

Useful examples on a typical Moto device:

| App | Shortcut id | What it does |
|---|---|---|
| `com.google.android.apps.photos` | `manifest_view_screenshots` | Open the Screenshots album |
| `com.motorola.camera5` | (DEPTH_CAPTURE) | Camera in depth mode |
| `com.google.android.apps.docs` | `SCAN_DOCUMENT_FROM_SHORTCUT` | Drive document scanner |

You can split gestures across different targets — e.g. single press opens camera, double press takes a screenshot, long press opens the screenshots album.

---

## References

| Resource | Where |
|---|---|
| Upstream activity-intent feature | [cvzi/ScreenshotTile#763](https://github.com/cvzi/ScreenshotTile/pull/763) |
| ScreenshotTile manifest | [AndroidManifest.xml](https://github.com/cvzi/ScreenshotTile/blob/master/app/src/main/AndroidManifest.xml) |
| ScreenshotTile README — activity intents | [`#automatic-screenshots-with-activity-intents`](https://github.com/cvzi/ScreenshotTile#automatic-screenshots-with-activity-intents) |
| Full MyKey settings catalog | [REFERENCE.md](./REFERENCE.md) |

### Smali sources

For anyone decompiling their own MyKey APK to verify the dispatch logic:

| Concern | File |
|---|---|
| AI Key trigger service | `smali/com/motorola/mykey/service/MyKeyTriggerService.smali` |
| Intent-action dispatch switch | `MyKeyTriggerService.smali:2537–2601` |
| `launchSpecialApp` (no validation, just parseUri + startActivity) | `MyKeyTriggerService.smali:2359` |
| `startShortcutByGaKey` (the SHORTCUT path) | `MyKeyTriggerService.smali:3603` |
| Settings the trigger service observes | `MyKeyTriggerService$SettingsObserver.smali` |
