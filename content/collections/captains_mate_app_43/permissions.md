---
id: 48184433-7137-42ff-bb65-88f071fdec5e
blueprint: captains_mate_app_43
title: Permissions
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780581674
---
# Permission Handling

## Overview

The app requests three categories of runtime permissions: **location**, **notifications**, and **photo library**. Location is by far the most significant, as it underpins the core friend-tracking and location-sharing features.

---

## Permissions Collected

### Location (Fine / Background)

| Platform | Declaration |
|----------|-------------|
| Android  | `ACCESS_FINE_LOCATION`, `ACCESS_BACKGROUND_LOCATION`, `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_TYPE_LOCATION`, `WAKE_LOCK`, `RECEIVE_BOOT_COMPLETED` |
| iOS      | `NSLocationWhenInUseUsageDescription`, `NSLocationAlwaysUsageDescription`, `NSLocationAlwaysAndWhenInUseUsageDescription` |

The iOS permission reason shown to the user is:
> "This app uses your location to show your current position on the map and to share with friends."

Background modes `location` and `remote-notification` are declared in `UIBackgroundModes` on iOS.

**Package used:** `geolocator` (permission checks/requests) + `flutter_background_geolocation` (background tracking)

---

### Notifications (Push + Local)

| Platform | Declaration |
|----------|-------------|
| Android  | `POST_NOTIFICATIONS` |
| iOS      | Handled by Firebase Messaging (system dialog) |

**Packages used:** `firebase_messaging`, `flutter_local_notifications`

---

### Photo Library

| Platform | Declaration |
|----------|-------------|
| iOS      | `NSPhotoLibraryUsageDescription` |
| Android  | No explicit declaration required for image picker |

The iOS permission reason shown to the user is:
> "This app uses the photo library to allow uploading new location images."

**Package used:** `image_picker`

---

### Motion / Activity Recognition

| Platform | Declaration |
|----------|-------------|
| iOS      | `NSMotionUsageDescription` |
| Android  | Requested at runtime via `permission_handler` on Android 10+ (SDK ≥ 29); no manifest declaration required |

The iOS permission reason shown to the user is:
> "This app uses your motion to help show your current position on the map to share with friends."

On iOS, this is implicitly requested by `flutter_background_geolocation`. On Android 10+, the app explicitly requests `activityRecognition` via `permission_handler` before calling `BackgroundGeolocation.start()` — this prevents the plugin from opening its own permission dialog and causing a lifecycle loop. See [lib/src/helpers/position.dart](lib/src/helpers/position.dart) for details.

---

## When Permissions Are Requested

### Location

Location permission is requested at multiple points, always gated through `PositionHelper.checkPermissions()` in [lib/src/helpers/position.dart](lib/src/helpers/position.dart).

| Trigger | Detail |
|---------|--------|
| **App startup** | `PositionHelper.init()` is called during initialisation. If sharing was previously enabled, the app checks permission status before resuming broadcasting. |
| **Enabling location sharing** (Settings screen toggle) | When the user turns on location sharing, `checkPermissions()` is called before starting the broadcast. |
| **Switching to Friends tab** | Location sharing is a prerequisite for the Friends feature. The app checks sharing is active before allowing tab access. |
| **Set Location / Map screen** | When the map screen loads, it checks for permission before attempting to fetch the user's current position. |

For background location sharing, the `locationAuthorizationRequest` is set to `'Always'`; for foreground-only use it is `'WhenInUse'`. The `flutter_background_geolocation` plugin shows its own system-level rationale dialog when upgrading to background access:

> "Allow {applicationName} to access to this device's location in the background? So your position can continue to be updated while you are not using the app, please enable {backgroundPermissionOptionLabel} location permission."

---

### Notifications

Requested once during app initialisation inside `FcmService.initialiseFirebaseMessaging()` in [lib/src/features/notifications/services/fcm_service.dart](lib/src/features/notifications/services/fcm_service.dart). The system permission dialog is shown the first time the app launches. The request asks for `alert`, `badge`, and `sound`.

---

### Photo Library

Requested implicitly by the `image_picker` plugin when the user taps a file input field (e.g. when uploading a location image). The system permission dialog is shown the first time the gallery is accessed.

---

## What Happens if a Permission is Refused

### Location – Denied (one-time refusal)

`checkPermissions()` calls `Geolocator.requestPermission()`. If the result is anything other than `whileInUse` or `always`, the method returns `false`. The calling code then:

- Prevents broadcasting from starting / continuing.
- Shows the **location permission denied dialog** (see below).

### Location – Denied Forever (permanently denied on Android / "Don't Ask Again")

`checkPermissions()` detects `LocationPermission.deniedForever` and returns `false` immediately without requesting again. The app:

- Shows the location permission denied dialog with an **"UPDATE PERMISSIONS"** button that calls `Geolocator.openLocationSettings()` to take the user directly to the system settings page.
- Disables location sharing in `SharedPreferences`.
- Stops any active background broadcast.

**Dialog text:**
> "Unable to read your current location, you may have denied the permission."

**Dialog actions:**
- **UPDATE PERMISSIONS** – Opens system location settings.
- **RETURN TO LOCATIONS** – Dismisses the dialog.

### Location – Permission Revoked While App Is Running

The home screen monitors `Permission.locationAlways.status` on each resume (via app lifecycle changes). If the user revokes permission in system settings while the app is backgrounded:

- `PositionBroadcastHelper.stopBroadcasting(unpublicise: true)` is called.
- The location sharing preference is set to `false`.
- The user's position is no longer broadcast to friends.

### Location – Required for Friends Feature

Switching to the Friends tab triggers several checks in sequence:

**1. No friend name set**

If the user has no friend name or boat name configured:

**Dialog text:**
> "Unable to share your location, you don't have a name set. You can specify one in settings."

**Dialog actions:**
- **SETTINGS** – Opens the Settings screen.
- **RETURN TO LOCATIONS** – Dismisses and stays on the Locations tab.

**2. Sharing not yet enabled — consent prompt**

If the user has not enabled location sharing, a consent dialog is shown before enabling it:

**Dialog text:**
> "Share your location to appear on the map and find members who are cruising nearby."

If background updates are enabled, an additional line is shown:
> "Your updated location will continue to be shared periodically even when the App is closed. You can change this behaviour in Settings or by tapping the arrow above."

**Dialog actions:**
- **SHARE MY LOCATION** – Enables sharing and proceeds to the Friends tab.
- **MAYBE LATER** – Shows the disabled-friends dialog below, then returns to Locations.

**3. Sharing disabled or permission denied**

If sharing could not be enabled (permission denied or user declined):

**Dialog text:**
> "Sorry, but you can only view and message members if you share your location."

**Dialog action:**
- **RETURN TO LOCATIONS** – Dismisses the dialog and stays on the Locations tab.

### Location – Map Screen (Set Location)

If permission is unavailable when the map screen loads, it falls back to a default position (CA House, London) rather than the user's real location. No error dialog is shown in this context.

### Notifications – Denied

The app does not show a custom explanation or re-request after denial. However, the user can toggle notifications independently within the app's settings via a `SharedPreferences` flag (`notificationEnabled`). If this flag is `false`, all incoming FCM notifications are silently suppressed regardless of the system permission state.

### Photo Library – Denied

If the user denies photo library access, the `image_picker` call fails silently. No custom dialog is shown by the app; the system handles the denial message.

---

## Summary Table

| Permission | When Requested | Denied (once) | Denied Forever / Revoked |
|------------|---------------|--------------|--------------------------|
| Fine Location | Startup, sharing toggle, map screen | Feature disabled, no dialog | Custom dialog + link to system settings |
| Background Location | When enabling background sharing | Background sharing disabled | Background sharing disabled |
| Notifications | App startup (once) | No re-request; in-app toggle still works | Notifications suppressed silently |
| Photo Library | On file input tap | System dialog handles it | Silent failure, no custom UI |
| Motion / Activity Recognition | iOS: implicitly by background geolocation. Android 10+: explicitly via `permission_handler` at startup | No explicit handling | No explicit handling |

---

## Key Files

| File | Role |
|------|------|
| [lib/src/helpers/position.dart](lib/src/helpers/position.dart) | Core `checkPermissions()` logic and background geolocation config |
| [lib/src/helpers/enable_localation_sharing_helper.dart](lib/src/helpers/enable_localation_sharing_helper.dart) | Permission denied dialogs, Friends feature gating |
| [lib/src/screens/home_screen.dart](lib/src/screens/home_screen.dart) | Lifecycle-aware permission status checks |
| [lib/src/screens/settings/settings_screen.dart](lib/src/screens/settings/settings_screen.dart) | Sharing toggle with permission check |
| [lib/src/features/notifications/services/fcm_service.dart](lib/src/features/notifications/services/fcm_service.dart) | Notification permission request and display guard |
| [android/app/src/main/AndroidManifest.xml](android/app/src/main/AndroidManifest.xml) | Android permission declarations |
| [ios/Runner/Info-Release.plist](ios/Runner/Info-Release.plist) | iOS permission usage descriptions |