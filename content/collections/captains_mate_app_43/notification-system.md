---
id: 02c4c254-dcd4-44d0-b6f0-2d25e2a5ab09
blueprint: captains_mate_app_43
title: 'Notification System'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780407202
parent: 632ff8b5-8ee1-442d-ba83-4c7378ffd528
---
# Notification System

## Overview

The app receives notifications from three sources. All are stored as `NotificationData` objects in an isolated Hive box and displayed in the notification list screen. The key linking field is `tcaNotificationId` (mapped to `notification_id` in JSON), the server's notification table ID.

---

## Notification Sources

### 1. FCM Push Notifications

Delivered by Firebase Cloud Messaging. The server sends **data-only messages** (not notification messages), so FCM delivers the payload silently and the app decides whether and how to show a notification.

- **Foreground** — received via `FirebaseMessaging.onMessage` listener → `handleMessageViaFCMListener()`
- **Background / terminated** — received via `onBackgroundMessage` top-level handler → `handleNotificationViaHeadless()`

**Flow:**
1. Parse `NotificationData` from FCM payload (`id` = Firebase message GUID, `created_at` = now)
2. Validate → Save → Display check → Show (see [Shared Pipeline](#shared-pipeline))
3. The `immediate` value is always taken from the **incoming FCM payload**, not the saved record. Synced records may have `immediate` defaulting to `false`; using the stored value would suppress display incorrectly.

### 2. Location (Geofence) Notifications

Triggered when the device enters a circular geofence boundary associated with a `Location`. These are local-only — no server push needed.

- Stored as `LocationNotification` inside the `Location` model, not as standalone records
- No `notification_id` — uses a generated UUID as the stored `NotificationData.id`
- The `triggered` flag is stored in Hive but excluded from JSON serialisation so a server sync cannot reset it

**Flow:**
1. `onGeofence` event fires (via `flutter_background_geolocation`)
2. Checks `geofenceNotificationEnabled` in user preferences
3. Looks up the associated `Location` from local storage
4. If `LocationNotification.triggered == false`, proceeds
5. Calls `FcmService().handleNotificationViaHeadless()` with a constructed `RemoteMessage` — runs the same validate → save → show pipeline as FCM
6. Saves the `Location` back with `triggered = true` to prevent re-firing

### 3. Pull Notifications (Sync)

Fetched during the delta sync (`MembersBatch`) via `GET /notifications/retrieve` with a `CAAPI-tfrom` header. Only runs on delta syncs — skipped on a full sync (`tfrom == null`).

**Flow:**
1. Fetch notifications for the delta period
2. Load all locally stored notifications into memory (keyed by `tcaNotificationId`)
3. For each server notification: validate → save (same shared logic as FCM)
4. No system notification is shown — newly saved notifications appear silently in the unread list

---

## Notification Data Model

`NotificationData` (`lib/src/features/notifications/models/notification_data.dart`) — Freezed + Hive serialisable:

| Field | JSON key | Notes |
|---|---|---|
| `id` | `id` | Firebase GUID (FCM) or generated UUID (sync/geofence) — Hive storage key |
| `action` | `action` | `path` or `external_url` (`NotificationAction` enum) |
| `title` | `title` | |
| `body` | `body` | |
| `path` | `path` | Route for navigation |
| `resourceId` | `resource_id` | ID of the resource to navigate to |
| `immediate` | `immediate` | Boolean stored as string; defaults to `true` |
| `geofence` | `geoarea` | WKT polygon string |
| `expiresAt` | `expiry_date` | |
| `readAt` | `seen_at` | |
| `createdAt` | `created_at` | |
| `tcaNotificationId` | `notification_id` | Server notification table ID; used for cross-source deduplication |

Three factory constructors:
- `NotificationData.fromJsonWithId(id, json)` — used for FCM; auto-sets `created_at` to now
- `NotificationData.fromSyncJson(id, json)` — used for sync; preserves server `created_at`
- `NotificationData.fromJson(json)` — used for tap payload decoding (id already embedded)

---

## Shared Pipeline

### Validation (`NotificationHelper.validateNotification`)

Called before saving, for all three sources:

1. **Geofence** — if `geoarea` is set and `PositionHelper.rawPosition` (cached) is not within the polygon, discard
2. **Expiry** — discard if past `expiry_date` or older than `Config.notificationDataRetentionDays` (30 days) based on `created_at`

### Save & Merge (`NotificationHelper.save`)

Handles deduplication across all three sources:

| Scenario | Outcome |
|---|---|
| No `tcaNotificationId` (geofence/local) | Always save as new |
| `tcaNotificationId` not in local storage | Save as new |
| Exists locally, existing has `readAt` set | No save — return existing (already read) |
| Exists locally, incoming has `readAt`, existing does not | Update existing `readAt` |
| Exists locally, neither has `readAt` | No save — return existing |

### Marking as Read (`NotificationHelper.markAsRead`)

Called when a notification is tapped in the list:
1. Sets `readAt` on the local record in Hive
2. If `tcaNotificationId` is set, calls `POST /notification/{tcaNotificationId}/seen`
3. Server failure is logged but does not block the local update (geofence notifications with no `tcaNotificationId` are marked locally only)

---

## Display Pipeline

All notification display goes through `FcmService` (`lib/src/features/notifications/services/fcm_service.dart`):

```
receive → _handleRemoteMessage()
            ↓
         _parseAndSaveNotificationData()
            ├── NotificationHelper.validateNotification()  [geofence, expiry]
            ├── NotificationHelper.save()                  [dedup write to Hive]
            └── return null if already read (readAt set)
            ↓
         _shouldDisplay()
            ├── notificationEnabled (SharedPreferences)
            └── immediate == true
            ↓
         _showNotification()                               [flutter_local_notifications]
```

### Why Data Messages?

FCM delivers **data-only messages** (not notification messages), giving the app full control:

| Aspect | Data Messages | Notification Messages |
|--------|---------------|----------------------|
| Auto-displays | No — app controls | Yes — system handles |
| Business logic | Full control | Limited |
| Filtering | Can check geofence, expiry, preferences | Shows immediately |
| Storage | Can store before showing | Harder to intercept |

### Display Conditions Summary

| Condition | Blocks saving? | Blocks display? |
|---|---|---|
| Outside geofence | Yes | No (not re-checked at display time) |
| Expired | Yes | No (not re-checked at display time) |
| Already read (`readAt` set) | No | Yes |
| Notifications disabled in settings | No | Yes |
| `immediate` is false | No | Yes |

A notification with `immediate: false` is saved and appears in the unread list but never produces an alert.

---

## App Initialization

**At startup** (`main.dart`):
```dart
await FcmService().initialiseFirebase();
FirebaseMessaging.onBackgroundMessage(onBackgroundMessage);
```

**After login** (`lib/src/data/providers/app_initialisation_provider.dart`):
```dart
await FcmService().initialiseFirebaseMessaging();
```

`initialiseFirebaseMessaging()` sets up:
- **Foreground messages** — `FirebaseMessaging.onMessage.listen(...)` → `handleMessageViaFCMListener()`
- **Token refresh** — `_messaging.onTokenRefresh.listen(...)` → sends new token to server if a refresh token exists in secure storage

---

## Geofence Implementation

### FCM Geofence (WKT Polygon)

FCM notifications can carry a `geoarea` WKT polygon. The check runs synchronously during validation using the cached device position:

```dart
static bool isWithinGeofence(NotificationData notificationData) {
  final geofencePolygon = notificationData.geofencePolygon;
  if (geofencePolygon.isEmpty) return true;
  final currentPosition = PositionHelper.rawPosition;
  return currentPosition != null &&
      isPointInPolygon(currentPosition, geofencePolygon);
}
```

`PositionHelper.rawPosition` is the last position recorded by the background geolocation service — a synchronous cached lookup, not a fresh GPS request.

The WKT parser (`lib/src/helpers/wkt_helper.dart`) converts a `POLYGON((lon lat, ...))` string to `List<LatLng>` using regex. `isPointInPolygon` uses the Ray Casting Algorithm (odd intersection count = inside).

**Important:** WKT coordinates are **longitude first, then latitude** (opposite of the usual "lat, lon" convention).

### Location Geofence (Radius)

Circular geofences are created by `LocationGeofenceService` (`lib/src/features/locations/services/location_geofence_service.dart`) for all locations that have a `LocationNotification` with `triggered == false`:

```dart
bg.Geofence(
  identifier: location.id,
  radius: location.notification!.radius!,
  latitude: location.position.value!.latitude,
  longitude: location.position.value!.longitude,
  notifyOnEntry: true,
)
```

This is **continuous monitoring** — `flutter_background_geolocation` tracks the user and fires even when the app is closed.

**Sync behaviour:** If the notification text changes on the server, `triggered` is reset to `false` so the new notification can fire. If unchanged, `triggered` is preserved. The flag is excluded from JSON serialisation so a sync can't accidentally reset it.

---

## Notification UI

### List Screen (`lib/src/features/notifications/screens/notifications_list_screen.dart`)

- Unread/Read tabs
- Search / filter
- Auto-refresh — polls Hive every 10 seconds for new notifications
- Pull-to-refresh

### Deep Linking / Navigation

When a notification is tapped, the app navigates based on `action` and `path`:

| Action | Path | Navigation |
|--------|------|------------|
| `path` | `location` | ViewLocationScreen (tab 1) |
| `path` | `report` | ViewLocationScreen (tab 2) |
| `path` | `boat` | ViewBoatScreen |
| `path` | `member` | ViewMemberScreen |
| `externalUrl` | URL | Opens external browser |

Navigation only occurs if both `path` and `resource_id` are present. If either is missing, the tap is a no-op.

### Unread Notification Modal

On login, if unread notifications exist, a modal appears once per session (guarded by `hasSeenNotificationModal` provider):

```
Login → Check for unread notifications → Show modal (once per session) → User taps "View" → Opens notification list
```

---

## Tap Handling

### Entry Points by App State

| App State | Entry Point | How |
|---|---|---|
| Foreground | `onDidReceiveNotificationResponse` | Fires immediately when notification tapped |
| Background | `onDidReceiveBackgroundNotificationResponse` | Fires when user taps notification in system tray |
| Terminated | `getNotificationAppLaunchDetails()` | Checked at startup; sets `_launchNotificationHandled = true` to prevent Android double-fire |

All three ultimately route through `handleNotificationTap()` (top-level in `main.dart`, annotated `@pragma('vm:entry-point')`).

### App Terminated (Cold Start)

1. `_checkForPendingNotification()` runs at startup
2. `getNotificationAppLaunchDetails()` detects the app was launched via a tap
3. Sets `pendingNotificationProvider` directly in the `ProviderContainer`
4. Sets `_launchNotificationHandled = true` to skip the duplicate Android callback

### App Foregrounded (Tapped from Background)

`handleNotificationTap()` → `FcmService.onNotificationTap(details, _appContainer)`:
- Saves to Hive (safety net — may not have been saved on iOS terminated state)
- If `_appContainer` available: sets `pendingNotificationProvider`
- If `_appContainer` is null (Android background isolate — app backgrounded but not terminated): saves notification ID to `SharedPreferences` via `setPendingTappedNotificationId`

### HomeScreen Navigation (`_handleNotifications`)

Called on first mount and on every `resumed` lifecycle event:
1. Check `pendingNotificationProvider` — if set, navigate immediately
2. Check `SharedPreferencesHelper.pendingTappedNotificationId` — fallback for Android background isolate path; clears the stored ID after reading
3. If neither is set, check for unread notifications → show modal (once per session, guarded by `hasSeenNotificationModal`)

`pendingNotificationProvider` is also listened to in `build` (via `ref.listen`) to handle notifications that arrive while the home screen is already active, gated by `_startupReady` to avoid firing during initial startup.

> **iOS note:** Notification data handling from a terminated state is limited. This is why pull notifications have been introduced to plug the gap.

---

## Notification Storage

Stored in Hive via `NotificationRepository` (`lib/src/features/notifications/services/notification_repository.dart`):
- Written on receipt, after passing validate + save checks
- Marked read/unread via `markAsRead`
- On repository initialisation, records older than 30 days (`Config.notificationDataRetentionDays`) or past `expiresAt` are deleted automatically

---

## User Notification Settings

### In-App Toggles

| Setting | SharedPreferences key | Default | Controls |
|---|---|---|---|
| Background notifications | `notification_enabled` | `true` | All FCM push notifications |
| Geographic notifications | `geofence_notification_enabled` | `true` | Location geofence notifications |

Both toggles are disabled (greyed out) without device-level notification permission. Geographic notifications additionally require "Always On" location permission.

### Two-Layer Permission Model

| Layer | What it controls | Where it lives |
|---|---|---|
| Device permissions | Whether toggles are enabled in the UI | OS (via Firebase Messaging / Geolocator) |
| In-app preferences | Whether notifications are actually shown | SharedPreferences |

A user's in-app preference is preserved if device permission is temporarily revoked. When they re-grant it, the toggle reflects the saved preference immediately.

### Enforcement

- **FCM** — `FcmService._shouldDisplay()` checks `notificationEnabled`
- **Geofence** — `LocationGeofenceService` checks `geofenceNotificationEnabled` before processing; skips starting the service if the preference is off
- **FCM token** — only registered with the server on login if `notificationEnabled` is `true`; revoked on logout regardless

---

## Token Management

| Event | Action |
|---|---|
| App start / login | Token sent to CA backend via `sendToken()`, along with `deviceId` (identifies multiple devices per user) |
| Token refresh | `onTokenRefresh` listener re-sends if user is logged in (refresh token present in secure storage) |
| Logout | Token revoked via `revokeToken()` |

If a refresh send fails (e.g. network error), the failure is logged but not retried. The server holds a stale token until the next login.

---

## Platform-Specific Setup

### iOS

| Config | Purpose |
|---|---|
| `remote-notification` background mode (`Info.plist`) | Receive FCM messages while backgrounded |
| `location` background mode | Continue geofence monitoring in background |
| `fetch` background mode | Background fetch tasks via `transistorsoft` geolocation library |
| `aps-environment` entitlement (`Runner.entitlements`) | `development` for debug builds, `production` for release |
| `AppDelegate.swift` | Sets up `FlutterLocalNotificationsPlugin` action isolate registrant; configures `UNUserNotificationCenter` as delegate (required for iOS 10+ foreground handling) |
| `GoogleService-Info.plist` | Two files — production and dev; correct one must be in place at build time |

### Android

| Config | Purpose |
|---|---|
| `POST_NOTIFICATIONS` permission | Required on Android 13+ |
| `FOREGROUND_SERVICE` + `FOREGROUND_SERVICE_TYPE_LOCATION` | Background location tracking |
| `ACCESS_FINE_LOCATION` + `ACCESS_BACKGROUND_LOCATION` | Geofence monitoring |
| `RECEIVE_BOOT_COMPLETED` | Restart background services after device reboot |
| Notification channel (`high_importance_channel`, importance: Max) | Created at runtime by `flutter_local_notifications` in `main.dart` |
| `google-services.json` | Two files — production and dev |

---

## Key Files

| File | Purpose |
|---|---|
| `lib/src/features/notifications/models/notification_data.dart` | Core data model |
| `lib/src/features/notifications/services/fcm_service.dart` | FCM initialisation, message processing, display |
| `lib/src/features/notifications/services/notification_helper.dart` | Shared validate, save, markAsRead logic |
| `lib/src/features/notifications/services/notification_repository.dart` | Hive persistence |
| `lib/src/features/notifications/enums/notification_action.dart` | `path` / `external_url` enum |
| `lib/src/features/locations/services/location_geofence_service.dart` | Circular geofence monitoring |
| `lib/src/features/sync/services/sync_endpoint_jobs/notifications_job.dart` | Pull/sync notification fetch |
| `lib/src/helpers/wkt_helper.dart` | WKT parser + point-in-polygon check |
| `lib/src/helpers/position.dart` | `rawPosition` cached device position |
| `lib/main.dart` | App entry point; background handlers; tap handling |

## Related Documentation

- [FCM Dev Tool](./FCM_DEV_TOOL.md) - Local development tool for sending test notifications