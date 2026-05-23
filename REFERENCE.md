# AI Generated

# Motorola MyKey (AI Key / Red Key) — Reference

Decompile source: `com.motorola.mykey_2.0.0.059-200000059_minAPI35(nodpi)_apkmirror.com.apk`
Decompile dir: `MotoMyKey/`
Package: `com.motorola.mykey`
Service process: `com.motorola.mykey.trigger`

---

## 1. System settings read & written by MyKey

All persisted state lives in Android Settings DBs. The trigger service watches these via a `ContentObserver` (`MyKeyTriggerService$SettingsObserver`) and re-reads them on every change.

### Settings.System keys

| Key | Type | Purpose |
|---|---|---|
| `my_key_tap_app` | string (intent URI) | Resolved intent for default/single press |
| `my_key_long_press_app` | string (intent URI) | Resolved intent for long press |
| `single_press_enable` | int (0/1, default 1) | Enable single press gesture |
| `double_press_enable` | int (0/1, default 1) | Enable double press gesture |
| `is_block_trigger` | int (0/1) | Suppress triggers in landscape orientation |
| `ptt_enable` | int (0/1) | Push-to-Talk enabled |
| `work_profile_ptt_enable` | int (0/1) | PTT in work profile |
| `mdm_disable_red_key_reminder` | int (0/1) | Show MDM-disabled reminder dialog |
| `my_key_ai_key_guide_dialog` | int (0/1) | Show AI Key intro guide |
| `my_key_ai_key_guide_dialog_other` | int (0/1) | Alt intro guide |
| `tap_app_main_press_hold` | string (intent URI) | **Main (power) button — long press** |
| `tap_app_quick_single` | string (intent URI) | **AI/Red Key — single press** |
| `tap_app_quick_double` | string (intent URI) | **AI/Red Key — double press** |
| `tap_app_quick_press_hold` | string (intent URI) | **AI/Red Key — long press** |

### Settings.Global keys

| Key | Purpose |
|---|---|
| `moto_spaces_enabled` | If 1, MyKey suppresses triggers while Moto Spaces is active |
| `quick_button_single_press_guide` | Single-press intro overlay |

### Settings.Secure keys

| Key | Purpose |
|---|---|
| `block_trigger_when_landscape` | Mirror of `is_block_trigger`, surfaced in `MyKeyOptionSettingActivity` |

### SharedPreferences

File: `system_gesture_le_key_settings_preferences`

Stores the **user-visible selection** (name + serialized intent) so the picker UI can re-render the choice. The system-settings keys above are the authoritative source the trigger service reads.

| Pref key prefix | Holds |
|---|---|
| `selected_app_name` | Display name of the picked action |
| `selected_app_name_main_double` / `_main_press` | Main button gesture labels |
| `selected_app_name_quick_single` / `_quick_double` / `_quick_press` | Red Key gesture labels |
| `selected_launch_app_intent*` | Serialized intent for launchable apps |

### Action-string dispatch (passed in the `EXPERIENCE` intent)

The system input layer fires an `EXPERIENCE` Intent with one of these `action` extras. `MyKeyTriggerService.onStartCommand` sparse-switches on the string:

| Action string | Handler | Used for |
|---|---|---|
| `action_single_press` | `doPressAction$2` | Single tap |
| `action_double_press` | `doPressAction$5` | Double tap |
| `action_long_press` | `doPressAction$10` | Press & hold |
| `action_key_press` | PTT broadcast (down) | PTT down |
| `action_key_release` | PTT broadcast (up) | PTT up |
| `action_enterprise` | `doPressAction$10` | MDM-forced action |

### MDM restrictions (consumed via `motoEnterpriseSdk`)

When set, these force the corresponding `tap_app_*` value to an enterprise action regardless of user choice:

- `single_press_red_key` → `action_enterprise`
- `double_press_red_key` → `action_enterprise`
- `long_press_red_key` → `action_enterprise`
- `long_press_power_key` → `"6"` (long press main button)
- `double_press_power_key` → `action_enterprise`

---

## 2. RedKeyActivity — what it exposes

File: `MotoMyKey/smali/com/motorola/mykey/activity/RedKeyActivity.smali`
Layout: `res/layout/activity_action_red_key.xml`
ViewModel: `com.motorola.mykey.viewmodel.HomePageViewModel`

A `ViewPager2 + TabLayout` with three tabs, one per gesture:

| Tab | Persists to |
|---|---|
| Single press | `tap_app_quick_single` |
| Double press | `tap_app_quick_double` |
| Press & hold | `tap_app_quick_press_hold` |

### Candidate actions (assignable to a gesture)

Each option in a tab is a `*CandidateInfo` bean under `MotoMyKey/smali/com/motorola/mykey/bean/`:

| Candidate class | Behavior |
|---|---|
| `NoneCandidateInfo` | Do nothing (`REAL_NONE_VALUE = ""`) |
| `LaunchAppCandidateInfo` | Launch any installed app (uses `LaunchAppActivity` picker) |
| `ShortcutCandidateInfo` | Launch a launcher pinned-shortcut |
| `AppCandidateInfo` | Base/legacy app candidate |
| `MusicControlCandidate` | Play/Pause (sends media key broadcast) |
| `ReadyForCandidateInfo` | Launch "Smart Connect" (Ready For) |
| `PTTCandidateInfo` | Push-to-Talk (gated by `ptt_enable`) |
| `EnterpriseCandidateInfo` | MDM-defined action |
| `GlobalSerachCandidateInfo` | Global Search (note the typo in the class name) |
| `SmartEyeCandidateInfo` | Smart Eye / Moto vision feature |
| `MagicCanvasCandidateInfo` | Magic Canvas |
| `RecorderCandidateInfo` | Voice recorder |
| `ImageStudioCandidateInfo` | Image Studio |
| `ServiceCandidateInfo` | Launch a system service action |
| `SpecialCandidateInfo` | Reserved / device-specific |
| `OverLockscreenCandidateInfo` | Variant that runs over the lockscreen |

**No `ScreenshotCandidateInfo` exists** — taking a screenshot is not a first-class candidate. The closest is the bundled Moto AI "Catch Up" launch, encoded as:

```
intent:#Intent;launchFlags=0x00040000;package=com.motorola.uxcore;\
action=com.motorola.uxcore.action.QUICK_LAUNCH;\
B.MOTO_LAUNCH_WITH_SCREENSHOT=true;end
```

(found in `MyKeyTriggerService$initValue$1.smali:683`, `MainButtonActivity.smali:917`, `HomePageViewModel.smali:3684`, `DoublePressPowerInjection.smali:90`). This snaps a screenshot and passes it to Moto AI for summarization — not a plain save-to-gallery.

### Regional / variant candidate lists

`HomePageViewModel` builds the candidate list per region/SKU:

- `getCandidatesRedKey()` — default Red Key list
- `getCandidatesPressAndHold()` — long-press-specific
- `getCandidatesQuickOnlyFuji()` — Fuji SKU (Razr 60 Ultra)
- `getCandidatesQuickOnlyJapanCustom()` — Japan-locked SKU

`RedKeyActivity$RedKeyObserver` watches `ptt_enable` and re-runs `updateSettings()` so the PTT row appears/disappears live.

---

## 3. Related activities

| Activity | Layout | Purpose |
|---|---|---|
| `SettingRouterActivity` | (transparent) | Manifest entry from system Settings; routes `MY_KEY_SETTINGS` |
| `QuickRouterActivity` | (transparent) | Routes `MY_KEY_QUICK_BUTTON_SETTINGS` and `…_PRESS_HOLD` |
| `MyKeyOptionSettingActivity` | `activity_mykey_option_setting.xml` | Top-level options screen — only switch is `block_trigger_when_landscape` |
| `SettingsActivity` | (host) | Parent of the per-gesture pickers |
| `MainButtonActivity` | `activity_action_main_button.xml` | Power button — 2 tabs (double, long) |
| `QuickButtonActivity` | `activity_action_quick_button.xml` | "Quick Button" SKU variant — 3 tabs |
| `RedKeyActivity` | `activity_action_red_key.xml` | **AI Key picker — 3 tabs** |
| `LaunchAppActivity` | (search-style) | App picker shown when user chooses "Launch app" |
| `TutorialDialogActivity` | (dialog) | First-run tutorial |
| `NoneReminderDialogActivity` | (dialog) | "No action assigned" / missing-app warning |
| `DisableKeyEventDialog` | (dialog) | MDM-disabled-key notice |
| `LicenseDetailActivity` | — | OSS licenses |
| `GoogleLensRouterActivity` | — | Routes a press to Google Lens |
| `VenusCliTipsActivity` | — | Venus SKU tip overlay |

---

## 4. Trigger service architecture

File: `MotoMyKey/smali/com/motorola/mykey/service/MyKeyTriggerService.smali`

```
System input layer (privileged)
        │  Intent { action=com.motorola.intent.action.EXPERIENCE,
        │           extras: action=<one of action_*> }
        ▼
MyKeyTriggerService (process com.motorola.mykey.trigger,
                    permission BIND_EXPERIENCE)
        │
        ├─ onStartCommand  → guards (MDM disabled?, Moto Spaces?, landscape?)
        │                 → doPressAction(action, fromKeyEvent=false)
        │
        ├─ doPressAction(action)
        │     sparse-switch on hashCode → coroutine $2/$5/$10 or PTT broadcast
        │     │
        │     └─ reads tap_app_quick_* from Settings.System
        │        → parses intent URI → startActivity
        │
        ├─ SettingsObserver (15 URIs)  → updateSettings() on any change
        │
        ├─ mDisplaySwitchReceiver      → lid state (folding phone)
        ├─ mScreenOffOnReceiver        → register/unregister sensor manager
        └─ mEnterpriseReceiver         → apply MDM policy
```

### Notable framework dependencies (`<uses-library>`)

- `moto-settings` (required) — extended Settings provider keys
- `moto-core_services` — exposes `MotoScreenHelperManager` / `ScreenHelperLite` (incl. `captureDisplay`, `injectKeyEvents`, `injectMotionEvents`). **MyKey itself does not call these**, but they're loaded into its classpath.
- `moto-enterprise-internal` (required) — MDM policy SDK

### Notable permissions (from `AndroidManifest.xml`)

- `com.motorola.permission.TAKE_SCREENSHOT` — present but unused by MyKey itself
- `com.motorola.permission.BIND_EXPERIENCE` — required for the trigger service
- `com.motorola.permission.MONITOR_INPUT` — receive raw input events
- `com.motorola.permission.READ_MOTO_DEVICE_POLICY_STATE` / `POLICY_STATE` — MDM
- `android.permission.WRITE_SECURE_SETTINGS` / `com.motorola.permission.WRITE_SECURE_SETTINGS` — write the `tap_app_*` keys
- **`android.permission.ACCESS_SHORTCUTS`** — lets MyKey query and launch shortcuts from any package via `LauncherApps` (critical for the SHORTCUT dispatch branch)

---

## 5. Runtime dispatch logic (intent-action based switch)

After `Intent.parseUri(rawString, URI_INTENT_SCHEME)` resolves the stored
`tap_app_quick_*` value, MyKey routes by **the parsed Intent's action
string** — not by candidate type or any allowlist.

Source: `MyKeyTriggerService.smali:2537–2601` (inside `launchSpecialApp`).

| Parsed `intent.action` | Handler | Behavior |
|---|---|---|
| `com.android.systemui.screenrecord.START` | `startScreenRecordByGaKey` | Start screen recording via SystemUI |
| `com.android.systemui.screenrecord.REAL.START` | `startScreenRecordByGaKey` | Same |
| `com.motorola.mykey.action.SHORTCUT` | `startShortcutByGaKey` | Launch a registered shortcut via `LauncherApps.startShortcut()` |
| `com.motorola.cn.mykey.action.SHORTCUT` | `startShortcutByGaKey` | China variant of same |
| `com.motorola.mykey.action.MUSIC` | `playOrPauseMusic` | Play/pause via media-key broadcast |
| _(anything else)_ | `startSpecialActivityByGaKey` | Plain `startActivity(intent)` |

### `launchSpecialApp` — no validation, just dispatch

Source: `MyKeyTriggerService.smali:2359`.

```
launchSpecialApp(rawString, settingsKey):
   if (rawString == "" || rawString == "0") return    // "none" sentinel
   if (!isUserSetupComplete()) return                  // first-boot guard
   intent = Intent.parseUri(rawString, URI_INTENT_SCHEME)
   ... ActivityOptions setup based on lid state ...
   dispatch by intent.action (table above)
```

**There is no candidate-type allowlist.** The MyKey UI shows different
candidates per gesture tab (e.g. "Launch app" only appears for single
press) but that's a UI-side filter only. The trigger service will faithfully
dispatch any valid intent URI you write to any of the three slots via ADB.

### `startShortcutByGaKey` — the SHORTCUT branch

Source: `MyKeyTriggerService.smali:3603`.

```
startShortcutByGaKey(intent, bundle):
   pkg = intent.extras.getString("shortcut_package_name")
   id  = intent.extras.getString("shortcut_id")
   if pkg or id empty: return false
   query = LauncherApps.ShortcutQuery()
              .setPackage(pkg)
              .setShortcutIds([id])
              .setQueryFlags(0x419)              // dynamic|manifest|cached|...
   shortcuts = LauncherApps.getShortcuts(query, Process.myUserHandle())
   if shortcuts non-empty and first is enabled:
      LauncherApps.startShortcut(shortcuts[0], null, bundle)
```

Why this matters: `LauncherApps.startShortcut()` gets a **temporary
framework permission** to launch the shortcut's target activity, even if
that activity is `exported="false"`. This is the only ADB-controllable
path to invoke a non-exported activity in another package.

### Other intent-action sub-cases worth knowing

- `com.motorola.mykey.action.MUSIC` → play/pause without launching any app
- `com.android.systemui.screenrecord.REAL.START` → start a screen recording
  via SystemUI's exported service. Subject to the user-grant overlay if
  not previously authorized

---

## 6. Runtime gotchas

These behaviors are easy to trip over when configuring via ADB.

| Gotcha | Cause | Workaround |
|---|---|---|
| **The settings UI overwrites your ADB-set intent** if you open Red Key → \<gesture\> tab and pick anything | The UI calls `setCurrentSelectionToDb()` which writes to both `Settings.System` and the SharedPreferences mirror | After ADB-wiring, don't tap that tab. If you do, re-run the ADB command |
| **`'0'` is the "none" sentinel, not numeric zero** | `launchSpecialApp` early-returns when the string is `""` or `"0"` | To "no-op" a gesture, set value to `'0'` or `delete` the key |
| **Empty string and `null` both behave as "none"** | Same early-return path; `settings get` returns `"null"` for unset keys | `settings delete system <key>` is the cleanest way to reset |
| **`single_press_enable` / `double_press_enable` are independent toggles** | They gate whether the gesture is even processed, before `launchSpecialApp` is called | If your intent looks correct but nothing happens, check these are `1` |
| **Semicolons in intent URIs break shell parsing** | Both host and device shells treat `;` as command separator | Use heredoc (`<<'EOF'`) or outer-double / inner-single quote nesting |
| **`settings put system <key> null` stores the literal string `"null"`** | The `settings` command doesn't interpret `null` | Use `settings delete system <key>` instead |
| **MDM `single_press_red_key` / `double_press_red_key` / `long_press_red_key` restrictions force `action_enterprise`** | `motoEnterpriseSdk.hasUserRestriction()` checked in `doPressAction` | Not applicable to personal devices |
| **`isUserSetupComplete()` guard** | First-boot suppression | Resolves on its own after device setup |

### ADB quoting recipes

Values without shell metacharacters (no `;`, `$`, `&`, etc.) — simple:

```bash
adb shell settings put system double_press_enable 1
```

Values with semicolons (intent URIs) — heredoc is most robust:

```bash
adb shell <<'EOF'
settings put system tap_app_quick_double 'intent:#Intent;action=...;end'
EOF
```

Or nested quotes on one line:

```bash
adb shell "settings put system tap_app_quick_double 'intent:#Intent;action=...;end'"
```

---

## 7. Controlling MyKey from outside the app

Writing the `tap_app_quick_*` keys requires `WRITE_SECURE_SETTINGS` —
works over ADB without root.

### Plain activity-launch intents

```bash
# Single press → open Camera
adb shell settings put system tap_app_quick_single \
  'intent:#Intent;component=com.motorola.camera5/com.motorola.camera.Camera;launchFlags=0x10000000;end'

# Single press → open any app's main activity
adb shell settings put system tap_app_quick_single \
  'intent:#Intent;action=android.intent.action.MAIN;category=android.intent.category.LAUNCHER;package=com.example;launchFlags=0x10000000;end'
```

### Shortcut-launch intents (reaches non-exported activities)

```bash
adb shell <<'EOF'
settings put system tap_app_quick_double 'intent:#Intent;action=com.motorola.mykey.action.SHORTCUT;launchFlags=0x10000000;S.shortcut_package_name=<target.package>;S.shortcut_id=<shortcutId>;end'
EOF
```

Enumerate available shortcuts on the device:

```bash
adb shell dumpsys shortcut | grep -B1 "ShortcutInfo "
```

### Special intent actions

```bash
# Play/pause music (no app launched)
adb shell settings put system tap_app_quick_single \
  'intent:#Intent;action=com.motorola.mykey.action.MUSIC;end'

# Start screen recording via SystemUI
adb shell settings put system tap_app_quick_press_hold \
  'intent:#Intent;action=com.android.systemui.screenrecord.START;end'

# Moto AI "Catch Up" (only works if com.motorola.uxcore is installed)
adb shell settings put system tap_app_quick_press_hold \
  'intent:#Intent;launchFlags=0x00040000;package=com.motorola.uxcore;action=com.motorola.uxcore.action.QUICK_LAUNCH;B.MOTO_LAUNCH_WITH_SCREENSHOT=true;end'
```

### Gesture enable/disable + block-in-landscape

```bash
adb shell settings put system single_press_enable 1   # 0 to disable
adb shell settings put system double_press_enable 1
adb shell settings put system is_block_trigger 1      # block when landscape
```

### Backing up & reverting

```bash
# Save current values to a runnable script
{
  echo "#!/bin/sh"
  for key in tap_app_quick_single tap_app_quick_double tap_app_quick_press_hold \
             single_press_enable double_press_enable is_block_trigger
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

Restore with `./mykey_backup.sh`.

---

## 8. Screenshot-trigger surfaces on Moto/Android — full taxonomy

A reference of every path that can take a screenshot on a Motorola
device, with reachability from `tap_app_quick_*` configuration.

### Summary

| Surface | Component type | Exported? | Gate | Reachable from MyKey config? |
|---|---|---|---|---|
| **ScreenshotTile (NoRoot) shortcut** | App shortcut | Manifest-published shortcut | None | ✅ via `com.motorola.mykey.action.SHORTCUT` |
| `com.motorola.uxcore` `QUICK_LAUNCH` w/ `MOTO_LAUNCH_WITH_SCREENSHOT=true` | Activity | Yes | None | ✅ plain `startActivity` (but opens Moto AI overlay) |
| `com.android.systemui.screenshot.ScreenshotInputService` | Service | Yes | `com.motorola.permission.TAKE_SCREENSHOT` (signature) | ❌ MyKey doesn't `bindService` from config |
| `com.android.systemui.screenshot.appclips.AppClipsTrampolineActivity` | Activity | Yes | `LAUNCH_CAPTURE_CONTENT_ACTIVITY_FOR_NOTE` (role) | ❌ MyKey doesn't hold the role permission |
| `com.android.systemui.screenshot.appclips.AppClipsService` | Service | Yes | Same role permission | ❌ Same |
| `SYSTEM_ACTION_TAKE_SCREENSHOT` broadcast | Receiver (dynamic) | Yes | `com.android.systemui.permission.SELF` (signature) | ❌ Signature-locked |
| `com.android.systemui.screenshot.TakeScreenshotService` | Service | No | `com.android.systemui.permission.SELF` | ❌ Internal only |
| `com.motorola.systemui.screenshot.MotoTakeScreenshotService` | Service | No | `…SELF` | ❌ Internal only |
| `com.motorola.systemui.screenshot.LensRouterActivity` | Activity | No | — | ❌ Not exported |
| `LaunchNoteTaskActivity` | Activity | Yes | None | ⚠️ Reachable but only `WIDGET_PICKER_SHORTCUT` entry — doesn't capture |
| QS `ScreenshotTile` (system) | Tile | N/A | — | ❌ Not launchable via Intent |
| `com.motorola.actions/.QuickScreenshotService` (3-finger gesture handler) | Service | No | — | ❌ Internal only |
| `com.motorola.actions/.TakeScreenshotService` | Service | No | — | ❌ Internal only |
| Moto Actions long-press-volume-down screenshot (`F7/a.run()` → `N5.c.T()`) | Internal Runnable | N/A | — | ❌ Internal only |
| Hardware key chord (power + volume-down) | Kernel/HAL | N/A | — | ❌ Not invokable from userspace |
| `KEYCODE_SYSRQ` via `adb shell input keyevent 120` | Input injection | N/A | — | ⚠️ Works from ADB but not from MyKey dispatch |
| `AccessibilityService.GLOBAL_ACTION_TAKE_SCREENSHOT` (Android 9+) | API call | N/A | Enabled accessibility service required | ✅ via helper app + shortcut (ScreenshotTile uses this) |
| `AccessibilityService.takeScreenshot(displayId, ...)` (Android 11+) | API call | N/A | Enabled accessibility service required | ✅ via helper app |
| `MediaProjection` | API | N/A | Per-session user prompt | ✅ via helper app (annoying UX) |
| Hidden `ScreenshotHelper.takeScreenshot()` (`com.android.internal.util`) | Class | N/A | Hidden API + signature | ❌ Not callable from non-platform apps |

### Why the SHORTCUT branch is the practical answer

- MyKey only does `startActivity` / `startActivityAsUser` from its
  `launchSpecialApp` dispatch — never `bindService`, never `sendBroadcast`
  (except for PTT). That immediately eliminates every screenshot surface
  that's a Service or Receiver.
- Every screenshot-taking Activity reachable from the manifest (whether
  AOSP or Moto-extended) is either non-exported, signature-permission
  protected, or role-protected. None are launchable from a user app.
- The shortcut path inverts the model: the user app publishes a static
  shortcut whose intent targets a non-exported activity in its own
  package. `LauncherApps.startShortcut()` grants the temporary launch
  permission. MyKey can call it because it holds `ACCESS_SHORTCUTS`.

### Apps known to publish useful shortcuts on Moto

Enumerate via `adb shell dumpsys shortcut`. On a typical Moto device:

| Package | Shortcut id | Behavior |
|---|---|---|
| `com.github.cvzi.screenshottile` | `takeScreenshot` | Take a screenshot (via accessibility service) |
| `com.github.cvzi.screenshottile` | `toggleFloatingButton` | Toggle the floating screenshot button |
| `com.github.cvzi.screenshottile` | `openSettings` | Open ScreenshotTile settings |
| `com.github.cvzi.screenshottile` | `openPhotoEditor` | Open photo editor with no image |
| `com.google.android.apps.photos` | `manifest_view_screenshots` | Open the Photos "Screenshots" album |
| `com.google.android.apps.docs` | (Drive scanner shortcut) | Document scanner (camera-based) |
| `com.motorola.camera5` | (DEPTH_CAPTURE shortcut) | Camera in depth mode |

### What does **not** work (verified)

- **Pure ADB with no helper app**, after `com.motorola.uxcore` is
  uninstalled: there is no stock launchable Activity that takes a
  screenshot and finishes. The shortcut path needs *some* app on the
  device to have published a screenshot-taking shortcut, and no Moto/AOSP
  pre-installed app does.
- **Piggybacking on Moto Actions' 3-finger pipeline**: every external
  surface (settings activities, content providers, the `TianxiAppActionService`
  app-action) either just opens settings UI or handles unrelated features
  (`quick_capture` = camera twist gesture, not screenshot).
- **Remapping the AI Key keycode**: the Motorola framework consumes the
  key event before it reaches WindowManager/Accessibility dispatch. Apps
  cannot observe or rewrite it. The only handler is whatever is bound
  via `BIND_EXPERIENCE` (signature-only).
- **Replacing MyKey with a custom EXPERIENCE handler**: `BIND_EXPERIENCE`
  is signature-protected; no ADB-installable app can register.

---

# Appendix — How Moto Actions takes a screenshot (3-finger gesture)

Decompile source: `Moto Actions & Gestures_08.17.21.apk`
Decompile dir: `MotoActions/`
Package: `com.motorola.actions`

## Gesture detection — done by the Moto framework, not the APK

The 3-finger swipe-down detection is provided by `com.motorola.aui.ThreeFingerGestureDetector` — a class loaded from the Moto framework via `<uses-library>`, **not** code in this APK. `MotoActions/smali/c6.1/d.smali:63` just probes for its presence:

```
Class.forName("com.motorola.aui.ThreeFingerGestureDetector")
```

When the framework reports a 3-finger swipe-down, it dispatches it to the `QuickScreenshotService` (registered in `AndroidManifest.xml` and gated by the `screenshot_trigger_gesture` SharedPreference).

A secondary `InputManager_proxy.monitorGestureInput(ctx, "moto_actions_input_channel", …)` call (`MotoActions/smali/A4/d.smali:277`) feeds raw MotionEvents into `c6/c.smali` purely to detect **5 quick 3-finger taps within 400 ms** → open `QuickScreenshotConflictActivity`. This path is *not* what fires a screenshot; it only surfaces the "the gesture didn't work, here's how to fix it" dialog.

## Screenshot capture — bind to SystemUI's ScreenshotInputService

The actual capture lives in `TakeScreenshotService.smali` (an `IntentService`). Triggered by an intent with action `"TAKE_SCREENSHOT_ACTION"`:

```
ComponentName cn = new ComponentName(
    "com.android.systemui",
    "com.android.systemui.screenshot.ScreenshotInputService");
bindService(new Intent().setComponent(cn), conn, 0x41);  // BIND_AUTO_CREATE | BIND_IMPORTANT
```

(`MotoActions/smali/com/motorola/actions/features/quickscreenshot/service/TakeScreenshotService.smali:150-178`)

On `onServiceConnected`, it wraps the `IBinder` in a `Messenger` and sends a single message:

```
Messenger remote = new Messenger(binder);
Message msg = Message.obtain(null, /*what=*/ 1);
Bundle data = new Bundle();
data.putBoolean("com.motorola.actions.IS_QUICK_SCREENSHOT_TYPE", true);
msg.setData(data);
msg.replyTo = new Messenger(localHandler);
remote.send(msg);
```

(`MotoActions/smali/e6.1/e.smali:62-118`)

SystemUI's `ScreenshotInputService` handles `SurfaceControl.screenshot()`, saves to `MediaStore.Images`, posts the standard screenshot notification + thumbnail UI, and replies via `msg.replyTo`. **The APK doesn't touch the bitmap directly.**

## Required permission

The bind succeeds only if the caller holds `com.motorola.permission.TAKE_SCREENSHOT` (`signatureOrSystem`). Moto Actions has it (manifest line declaring `<uses-permission android:name="com.motorola.permission.TAKE_SCREENSHOT"/>`). **MyKey also already declares this permission**, so the same recipe will work from inside MyKey without any additional permission grant.

## Settings keys (Moto Actions)

| Key | Storage | Purpose |
|---|---|---|
| `screenshot_trigger_gesture` | SharedPref `com.motorola.actions_preferences` | Master enable for the 3-finger gesture |
| `quick_screenshot_do_not_show_conflict_again` | SharedPref | Suppresses conflict dialog after 5 rapid 3-finger taps |

## What this means for "AI Key → screenshot"

Both MyKey and Moto Actions are Motorola-signed apps with `com.motorola.permission.TAKE_SCREENSHOT`. To wire the AI Key to a screenshot we can **port the same SystemUI-bind recipe** into MyKey's `MyKeyTriggerService`:

1. In one of `doPressAction$2/$5/$10`, branch on a sentinel intent (or add a `ScreenshotCandidateInfo`).
2. `bindService(ComponentName("com.android.systemui", "com.android.systemui.screenshot.ScreenshotInputService"), conn, 0x41)`.
3. In `onServiceConnected`, send `Message(what=1)` with `Bundle("com.motorola.actions.IS_QUICK_SCREENSHOT_TYPE", true)`.
4. SystemUI does the rest — capture, save, notify.

This is the minimum-surface modification because we reuse the same system service Moto Actions uses, with the same permission MyKey already declares. No `SurfaceControl` reflection, no `MediaProjection` prompt, no accessibility service needed.

Open question worth verifying on-device: does SystemUI's `ScreenshotInputService` care about the **caller's package name**? If it whitelists `com.motorola.actions` specifically, the call from `com.motorola.mykey` could be rejected. Quick test: a tiny PoC APK signed with the Motorola system certificate (Magisk root + ReVanced-style remount) or `adb shell am startservice -n com.motorola.actions/...quickscreenshot.service.TakeScreenshotService -a TAKE_SCREENSHOT_ACTION` and watching logcat for "BindServiceFailed".

---

# Appendix — Smali file:line index

For verifying any claim in this document. All paths relative to project root.

## MyKey trigger service

| Concern | Path |
|---|---|
| Main service class | `MotoMyKey/smali/com/motorola/mykey/service/MyKeyTriggerService.smali` |
| `onStartCommand` (entry point) | `MyKeyTriggerService.smali:~4897` |
| `onBind` (Messenger IPC) | `MyKeyTriggerService.smali:~4544` |
| `doPressAction` sparse-switch | `MyKeyTriggerService.smali:~718` |
| `launchSpecialApp` (parse + dispatch) | `MyKeyTriggerService.smali:2359` |
| Action-based dispatch switch | `MyKeyTriggerService.smali:2537–2601` |
| `startShortcutByGaKey` | `MyKeyTriggerService.smali:3603` |
| `startSpecialActivityByGaKey` (plain startActivity path) | `MyKeyTriggerService.smali:~2585–2592` |
| `startScreenRecordByGaKey` | `MyKeyTriggerService.smali:~2612` |
| `playOrPauseMusic` | `MyKeyTriggerService.smali:~2579` |
| Long-press field read | `MyKeyTriggerService.smali:755, 773, 947, 1011` |
| Double-press field read | `MyKeyTriggerService.smali:1460, 1505, 1641, 1665, 1682, 1746` |
| Single-press field read | `MyKeyTriggerService.smali:1050, 1112, 1243` |
| MDM `single_press_red_key` restriction | `MyKeyTriggerService.smali:1132` |
| MDM `double_press_power_key` restriction | `MyKeyTriggerService.smali:1145` |
| MDM `long_press_red_key` restriction | `MyKeyTriggerService.smali:792` |
| PTT broadcast paths | `MyKeyTriggerService.smali:3358, 3455` |
| `SettingsObserver` (URI list) | `smali/com/motorola/mykey/service/MyKeyTriggerService$SettingsObserver.smali:72–375` |
| `mDisplaySwitchReceiver` | `smali/com/motorola/mykey/service/MyKeyTriggerService$mDisplaySwitchReceiver$1.smali:81` |
| `mScreenOffOnReceiver` | `smali/com/motorola/mykey/service/MyKeyTriggerService$mScreenOffOnReceiver$1.smali:71, 105` |
| `mEnterpriseReceiver` | `smali/com/motorola/mykey/service/MyKeyTriggerService$mEnterpriseReceiver$1.smali:81–329` |

## MyKey UI / ViewModels

| Concern | Path |
|---|---|
| RedKeyActivity (3 tabs) | `smali/com/motorola/mykey/activity/RedKeyActivity.smali` |
| MainButtonActivity (power) | `smali/com/motorola/mykey/activity/MainButtonActivity.smali` |
| QuickButtonActivity | `smali/com/motorola/mykey/activity/QuickButtonActivity.smali` |
| MyKeyOptionSettingActivity (block-landscape switch) | `smali/com/motorola/mykey/activity/MyKeyOptionSettingActivity.smali` |
| HomePageViewModel (candidate lists) | `smali/com/motorola/mykey/viewmodel/HomePageViewModel.smali` |
| `queryShortcutInfo` (cross-app shortcut listing) | `smali/com/motorola/mykey/viewmodel/MyKeySettingsViewModel.smali:2518–2660` |
| Candidate bean classes | `smali/com/motorola/mykey/bean/*CandidateInfo.smali` |

## Moto Actions (3-finger screenshot + volume-down screenshot)

| Concern | Path |
|---|---|
| QuickScreenshotService (gesture orchestrator) | `MotoActions/smali/com/motorola/actions/features/quickscreenshot/service/QuickScreenshotService.smali` |
| TakeScreenshotService (worker, binds SystemUI) | `MotoActions/smali/com/motorola/actions/features/quickscreenshot/service/TakeScreenshotService.smali` |
| Bind to SystemUI's `ScreenshotInputService` | `TakeScreenshotService.smali:150–178` |
| Messenger send w/ `IS_QUICK_SCREENSHOT_TYPE` | `MotoActions/smali/e6.1/e.smali:62–118` |
| 3-finger conflict-dialog detector (NOT the trigger) | `MotoActions/smali/c6.1/c.smali` |
| `ThreeFingerGestureDetector` probe | `MotoActions/smali/c6.1/d.smali:63` |
| InputManager gesture-monitor channel registration | `MotoActions/smali/A4/d.smali:277` |
| `screenshot_trigger_gesture` pref check | `MotoActions/smali/c6.1/h.smali:125` |
| Long-press volume-down → screenshot fallback (`F7/a.run()`) | `MotoActions/smali/F7/a.smali:37–47` |
| `N5.c.T()` internal trigger | `MotoActions/smali/N5/c.smali:3627–3657` |
| MotoService (BIND_EXPERIENCE alt handler — game-mode IPC, not screenshot) | `MotoActions/smali/com/motorola/actions/core/motoservice/MotoService.smali` |
| TianxiAppActionService (Assistant app-action — `quick_capture` only) | `MotoActions/smali/com/motorola/actions/features/quickcapture/tianxi/TianxiAppActionService.smali` |
| App-action definitions | `MotoActions/res/raw/app_action.json` |

## SystemUI (com.android.systemui)

| Concern | Path |
|---|---|
| `ScreenshotInputService` (exported, `TAKE_SCREENSHOT` perm) | `MotoSystemUI/smali_classes2/com/android/systemui/screenshot/ScreenshotInputService.smali` |
| `MotoGlobalScreenshot` (Moto-extended capture) | `MotoSystemUI/smali_classes2/com/android/systemui/screenshot/MotoGlobalScreenshot.smali` |
| `ScreenshotTile` (QS tile) | `MotoSystemUI/smali_classes2/com/android/systemui/qs/tiles/ScreenshotTile$1.smali:108` |
| `SystemActions` accessibility receiver | `MotoSystemUI/smali/com/android/systemui/accessibility/SystemActions.smali` |
| `SYSTEM_ACTION_TAKE_SCREENSHOT` handler | `MotoSystemUI/smali/com/android/systemui/accessibility/SystemActions$SystemActionsBroadcastReceiver.smali:790` |
| Bug-report screenshot path | `MotoSystemUI/smali/com/android/systemui/globalactions/GlobalActionsDialogLite$BugReportAction$1.smali` |
| LauncherProxy screenshot path | `MotoSystemUI/smali_classes2/com/android/systemui/recents/LauncherProxyService$1.smali` |
| `NoteTaskController` | `MotoSystemUI/smali_classes2/com/android/systemui/notetask/NoteTaskController.smali` |
| `LaunchNoteTaskActivity` (exported but `WIDGET_PICKER_SHORTCUT` only — no screenshot) | `MotoSystemUI/smali_classes2/com/android/systemui/notetask/shortcut/LaunchNoteTaskActivity.smali` |

## ScreenshotTile (com.github.cvzi.screenshottile)

| Concern | Path |
|---|---|
| Manifest | `https://github.com/cvzi/ScreenshotTile/blob/master/app/src/main/AndroidManifest.xml` |
| Static shortcuts (`takeScreenshot` etc.) | `https://github.com/cvzi/ScreenshotTile/blob/master/app/src/main/res/xml/shortcuts.xml` |
| `NoDisplayActivity` (target of `takeScreenshot` shortcut) | `com.github.cvzi.screenshottile.activities.NoDisplayActivity` — `exported="false"` |
| `ScreenshotAccessibilityService` | `com.github.cvzi.screenshottile.services.ScreenshotAccessibilityService` |
| `IntentHandler` (broadcast receiver for `…SCREENSHOT` action) | `com.github.cvzi.screenshottile.IntentHandler` |
