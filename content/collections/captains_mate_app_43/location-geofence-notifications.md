---
id: a5eac0eb-2a0c-4a15-a253-df93e1eb3c86
blueprint: captains_mate_app_43
title: 'Location Geofence Notifications'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774536835
---
# Location Geofence Notifications

This document describes the location-based notification system that triggers notifications when users physically approach specific locations.

## Overview

This is a **separate system** from FCM Push Notifications. Instead of being pushed from a server, these notifications are:

1. **Attached to Location objects** via the API
2. **Stored locally** on the device
3. **Triggered automatically** when the user enters a geographic radius around the location

## How It Works

```
Location syncs from API → App creates geofence → User enters radius → Local notification shown
        ↓                        ↓                      ↓                      ↓
   Has notification?      Based on lat/long      Device detects         Uses FCM service
   Has radius?            and radius             GPS boundary           to display
```

## Data Model

### LocationNotification (`lib/src/features/locations/models/location_notification.dart`)

```dart
factory LocationNotification({
  @JsonKey(name: 'text') required String text,      // Message to display
  @JsonKey(name: 'radius') required double? radius, // Geofence radius in meters
  @Default(false) bool triggered,                   // Has notification been shown?
})
```

### Attached to Location (`lib/src/features/locations/models/location.dart`)

```dart
@JsonKey(name: 'notification')
@HiveField(20)
LocationNotification? notification,
```

**Note:** `LocationNotification` is only attached to the `Location` model - no other models (boats, members, reports, etc.) have this capability.

## flutter_background_geolocation Package

This system uses the `flutter_background_geolocation` package for:

- **Creating geofences** around location coordinates
- **Monitoring geofence entry** events
- **Background operation** when app is closed

```dart
import 'package:flutter_background_geolocation/flutter_background_geolocation.dart' as bg;

// Add geofences
await bg.BackgroundGeolocation.addGeofences(geofences);

// Listen for entry events
bg.BackgroundGeolocation.onGeofence(onGeofence);
```

This is **continuous monitoring** - the device tracks the user's location and triggers when they enter a geofence radius, even when the app is closed.

## Geofence Service (`lib/src/features/locations/services/location_geofence_service.dart`)

### Initialization

On app start, the service:

1. Gets all locations with untriggered notifications
2. Creates geofences for each
3. Registers a listener for geofence events

```dart
Future<void> initialise() async {
  List<Location> locations = (await LocationRepository().getAll())
      .where((l) => l.notification != null && !l.notification!.triggered)
      .toList();

  await syncGeofences(locations);

  if (!await SharedPreferencesHelper().geofenceNotificationEnabled) {
    await stop();
    return;
  }

  await start();
}
```

### Geofence Creation

For each location with a notification, a geofence is created:

```dart
bg.Geofence(
  identifier: location.id,
  radius: location.notification!.radius!,
  latitude: location.position.value!.latitude,
  longitude: location.position.value!.longitude,
  notifyOnEntry: true,
  extras: {
    'location_id': location.id,
    'type': 'location',
  },
)
```

### When User Enters Geofence

The `onGeofence()` function is triggered:

```dart
Future<void> onGeofence(bg.GeofenceEvent event) async {
  // Check if geofence notifications are enabled
  if (!await SharedPreferencesHelper().geofenceNotificationEnabled) return;

  // Get the location from offline storage
  Location? location = await LocationRepository().get(locationId);

  // Check if already triggered
  if (location.notification!.triggered == true) return;

  // Show notification using FCM service
  FcmService().handleBackgroundNotification(RemoteMessage(
    data: NotificationData(
      title: notification.text,
      path: 'location',
      action: NotificationAction.path,
      resourceId: location.id,
      immediate: true,
    ).toJson(),
  ));

  // Mark as triggered so it won't show again
  await LocationRepository().save(
    location.copyWith(
      notification: notification.copyWith(triggered: true),
    ),
    location.id,
  );
}
```

## User Notification Settings

Users can control notifications via settings stored in SharedPreferences (`lib/src/helpers/shared_preferences.dart`):

| Setting | Key | Default | Controls |
|---------|-----|---------|----------|
| Push Notifications | `notification_enabled` | `true` | FCM push notifications |
| Geofence Notifications | `geofence_notification_enabled` | `true` | Location-based notifications |

The geofence service checks `geofenceNotificationEnabled` before processing:

```dart
if (!await SharedPreferencesHelper().geofenceNotificationEnabled) return;
```

When geofence notifications are disabled:
- Geofence listener is stopped
- Location tracking config is adjusted (unless position sharing is enabled)

## Sync Behavior (`lib/src/features/sync/services/sync_service.dart`)

When locations sync from the API:

1. If the notification **text changed**, the `triggered` flag is reset (so the new notification can show)
2. If the notification **text is the same**, the existing `triggered` state is preserved

```dart
bool notificationChanged = existingLocation.notification?.text !=
    location.notification?.text;

notification: LocationNotification(
  text: location.notification?.text ?? '',
  radius: location.notification?.radius,
  triggered: notificationChanged
      ? false
      : existingLocation.notification?.triggered ?? false,
)
```

## Comparison with FCM Push Notifications

| Aspect | Location Geofence | FCM Push |
|--------|-------------------|----------|
| **Trigger** | User enters geographic area | Server sends message |
| **Source** | Attached to Location via API | Sent via Firebase |
| **Geofence shape** | Circle (radius from point) | Polygon (WKT format) |
| **One-time vs repeating** | One-time (marked as triggered) | Each message is independent |
| **Requires internet** | No (works offline) | Yes (to receive) |
| **Control** | Backend sets on location | Server decides when to send |

## Key Files

| File | Purpose |
|------|---------|
| `lib/src/features/locations/models/location_notification.dart` | Data model |
| `lib/src/features/locations/models/location.dart` | Location model with notification field |
| `lib/src/features/locations/services/location_geofence_service.dart` | Geofence management and triggering |
| `lib/src/features/sync/services/sync_service.dart` | Handles sync and triggered state |
| `lib/main.dart` | Headless task for background geofence events |

## Background Handling (`main.dart`)

Geofence events can trigger even when the app is closed:

```dart
@pragma('vm:entry-point')
void headlessTask(bg.HeadlessEvent headlessEvent) async {
  switch (headlessEvent.name) {
    case bg.Event.GEOFENCE:
      bg.BackgroundGeolocation.startBackgroundTask().then((int taskId) async {
        bg.GeofenceEvent geofenceEvent = headlessEvent.event;
        await onGeofence(geofenceEvent);
        bg.BackgroundGeolocation.stopBackgroundTask(taskId);
      });
      break;
  }
}
```

## Related Documentation

- [FCM Push Notifications](./FCM_PUSH_NOTIFICATIONS.md) - Server-pushed notifications via Firebase