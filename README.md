# Moov Coach Offline Patch

Patched version of Moov Coach (v5.2.5492) that works **completely offline** without needing Moov's servers (which are permanently down).

## What's the problem?

Moov Coach was a fitness app that worked with the Moov Now / Moov HR wristbands. The company shut down and their servers (`muscle.moov.cc`) went offline, making the app unusable — it requires a login on every launch and can't reach the server.

This patch removes that dependency so you can keep using your Moov hardware.

## What does the patch do?

| Patch | File | Description |
|-------|------|-------------|
| **OfflineHelper** | `smali/cc/moov/patch/OfflineHelper.smali` | New class. Creates a User + UserProfile directly in memory using public setters, then calls `setCurrentUser()` which persists to SharedPreferences. Uses numeric userId (`"1000001"`) as required by `User.triggerLogin()`. Only creates if `sCurrentUser` is null. |
| **Skip login** | `smali/cc/moov/main/LaunchActivity$1.smali` | `run()` always navigates to `MainTabbedActivity`, bypassing the `getCurrentUser()` / `isComplete()` check that would redirect to the login screen. |
| **Null user handler** | `smali/cc/moov/main/MainTabbedActivity.smali` | When `getCurrentUser()` returns null in `onCreate()`, calls `OfflineHelper.ensureOfflineUser()` instead of restarting the app via `gotoTheFirstPlace()` (which caused an infinite restart loop). |
| **Field visibility** | `smali/cc/moov/sharedlib/onboarding/User.smali` | `sCurrentUser` and `mUserProfile` changed from `protected` to `public`. ART (Android Runtime) enforces access checks across packages, unlike desktop JVMs. |
| **Server reachability** | `smali/cc/moov/common/network/Reachability.smali` | `isOurServerReachable()` always returns `true`, preventing server-down error dialogs. |
| **Profile sync** | `smali/cc/moov/sharedlib/onboarding/UserProfileApiHelper.smali` | `uploadToServer()` and `fetchDictFromServer()` are no-ops. `uploadToServer()` passes the real `UserProfile` object to the `onFinish` callback (not null) to prevent `NullPointerException` when `SettingsActivity` calls `.save()`. |

### What's NOT patched

- **Bluetooth / sensor logic**: Untouched. All workout processing (HR, cadence, pace, swimming laps, boxing punches) runs locally in `libbridge.so` and was never server-dependent.
- **Local storage**: Untouched. Workouts are saved to the local SQLite database (ActiveAndroid) as they always were.
- **Settings/profile UI**: Untouched. You can change height, weight, gender, name, etc. from the app's settings — changes persist to SharedPreferences locally.
- **Other network calls**: The `UploadQueue` and other sync mechanisms will silently fail (no server to talk to) but won't crash the app. Workout data stays on-device.

## Installation

### Prerequisites

- Android device (API 19+ / Android 4.4+)
- The original Moov Coach app **must be uninstalled first** (the signature is different)

### Steps

1. **Uninstall** the original Moov Coach from your device
2. **Enable** installation from unknown sources:
   - Android 8+: Settings → Apps → Special access → Install unknown apps → (your file manager) → Allow
   - Android 7 and below: Settings → Security → Unknown sources → Enable
3. **Copy** `moov-offline.apk` to your phone (USB, email, cloud, etc.)
4. **Open** the APK file to install it
5. **Launch** the app — it should go straight to the main screen, no login required
6. **Go to Settings** inside the app to set your real height, weight, gender, and birthday

### First launch

On the very first launch, the app creates a default profile:
- Name: "Moov User"
- Height: 175 cm
- Weight: 70 kg
- Gender: Male
- Birthday: 1990-01-01
- Unit: Metric

**Change these immediately** in the app's settings to match your real data — the workout algorithms (calorie calculation, heart rate zones, etc.) depend on them.

## Building from source

If you want to apply the patches yourself or modify them:

```bash
# Requirements
sudo apt install apktool default-jdk zipalign

# Decompile the original APK
apktool d original-moov-coach.apk -o moov-patched

# Apply patches (copy the modified files from this repo)
cp -r patches/smali/cc/moov/patch moov-patched/smali/cc/moov/patch
# ... apply other patches (see below)

# Recompile
apktool b moov-patched -o moov-offline.apk --use-aapt2

# Sign (required by Android)
keytool -genkey -v -keystore release.keystore -alias moov -keyalg RSA \
    -keysize 2048 -validity 36500 -storepass yourpassword -keypass yourpassword \
    -dname "CN=MoovOffline, OU=Patch, O=Local, L=Local, ST=Local, C=ES"

jarsigner -sigalg SHA256withRSA -digestalg SHA-256 \
    -keystore release.keystore -storepass yourpassword -keypass yourpassword \
    moov-offline.apk moov

# Optimize
zipalign -v 4 moov-offline.apk moov-offline-aligned.apk
mv moov-offline-aligned.apk moov-offline.apk
```

### Note on `$`-prefixed resource files

The original APK contains drawable files with `$` in their names (e.g., `$avd_hide_password__0.xml`) which `aapt2` rejects. These are renamed to `_dollar_avd_*` during the build. This only affects the password visibility toggle animation and has no functional impact.

## Troubleshooting

### App crashes on launch
Check logcat for clues:
```bash
adb logcat | grep -iE "moov|offline|fatal|crash"
```
The `OfflineHelper` logs to logcat with tag `OfflineHelper` — look for "Creating offline user..." and "Offline user created OK".

### App shows "no connection" warnings
These are cosmetic — the app still functions. The `isOurServerReachable` patch prevents blocking dialogs, but some UI elements may still show connectivity warnings.

### App crashes when saving settings
If you see `NullPointerException` on `UserProfile.save()`, the `uploadToServer` patch may need updating. Check that it passes the real `UserProfile` to the `onFinish` callback.

### Workouts don't save
Workout data is processed and saved locally in SQLite by `libbridge.so`. If workouts aren't saving, it's likely a Bluetooth connection issue with the wristband, not related to this patch.

### Want to reset to the default offline user
Clear the app's data: Settings → Apps → Moov Coach → Storage → Clear Data. On next launch, the OfflineHelper will create a fresh default profile.

## Technical details

### Architecture overview
```
┌─────────────────────────────┐
│     UI Layer (Java)         │  Activities, Fragments
├─────────────────────────────┤
│     Bridge Layer (JNI)      │  HttpClientBridge, UrlManager, WorkoutModelBridge
├─────────────────────────────┤
│     Native Layer (C++)      │  libbridge.so — sensor processing, workout
│                             │  algorithms, reports, coaching logic
├─────────────────────────────┤
│     Storage                 │  SharedPreferences (user/profile)
│                             │  SQLite/ActiveAndroid (workouts)
│                             │  Protobuf (sensor data serialization)
└─────────────────────────────┘
```

### Key discovery: the app is offline-capable by design
The heavy lifting (sensor data processing, real-time coaching, workout reports) all runs locally in `libbridge.so`. The server was only used for:
- Authentication (login/register) → **patched out**
- Profile sync → **patched out (no-op with real callback)**
- Workout cloud backup → **fails silently, data stays local**
- Social features (Firebase) → **fails silently**
- Firmware updates → **no longer relevant**

### Lessons learned during patching
- **ART is stricter than JVM**: `protected` fields are not accessible cross-package at runtime, even though smali/bytecode allows it at compile time. Fields accessed from `cc.moov.patch` must be `public` in `cc.moov.sharedlib.onboarding`.
- **userId must be numeric**: `User.triggerLogin()` calls `Long.valueOf(userId)` — string IDs cause `NumberFormatException`.
- **Context timing matters**: `ApplicationContextReference.getContext()` returns null during `MoovApplication.onCreate()` before `setContext()` is called. The OfflineHelper must run later (e.g., in `MainTabbedActivity.onCreate()`).
- **Callback parameters matter**: `uploadToServer`'s `onFinish(boolean, UserProfile, int)` must receive the real `UserProfile`, not null — otherwise `SettingsActivity.onPause()` crashes when calling `.save()` on the returned profile.

### Original API endpoints (for reference)
These are the endpoints the original app called on `https://muscle.moov.cc`:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/account/login` | POST | Email/password login |
| `/api/v1/account/register` | POST | Create account |
| `/api/v1/account/facebook_login` | POST | Facebook login |
| `/api/v1/my_access_token` | GET | Refresh access token |
| `/api/v1/me/logout` | POST | Logout |
| `/api/v1/profile/basic/{userId}` | GET/PUT | Read/update profile |
| `/api/v1/profile/avatar/{userId}` | PUT | Upload avatar |
| `/api/v1/uploaded/contract/{id}` | POST | Firmware contracts |
| `/api/v1/workout/photo/{id}/{name}` | GET | Workout photos |

Authentication used the `X-Moov-Api-Session` header with a session ID, plus `MOOV_SESSIONID` cookies.

## License

This patch is provided for personal use to restore functionality of hardware you own. The original Moov Coach app is property of Moov Inc. (defunct).

Generated by Opus4.6