---
id: 02c4c254-dcd4-44d0-b6f0-2d25e2a5ab09
blueprint: captains_mate_app_43
title: 'FCM Push Notifications'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774536757
---
# FCM Notifications

## FCM Notification Tool - Quick Start Guide

### Prerequisites

1. **Firebase Service Account Key** (JSON file)
2. **A device FCM token** to send to

### Step 1: Get the Service Account Key

To generate the Service Account Key:

1. Go to [Firebase Console](https://console.firebase.google.com/)
2. Select the relevant project
3. Go to **Project Settings** → **Service Accounts**
4. Click **Generate New Private Key**
5. Save the downloaded JSON file

### Step 2: Set Up the Tool

```bash
cd notifications

# Install dependencies
npm install

# Save the service account key file as:
# notifications/firebase-service-account.json
```

### Step 3: Get a Device FCM Token

Run the app in debug mode and add a breakpoint at line 97 in `lib/src/features/notifications/services/fcm_service.dart`:

```dart
messaging.getToken().then((String? token) async => await sendToken(token: token));
```

Copy the `token` value when the breakpoint hits.

### Step 4: Configure Your Token (Optional)

```bash
# Create .env file to avoid typing token each time
echo "TEST_FCM_TOKEN=paste_your_token_here" > .env
```

### Step 5: Send a Notification

```bash
# Basic send (uses token from .env)
npm run send

# Custom message
npm run send -- --title "Test Alert" --body "Hello from CLI"

# Specify token directly
npm run send -- --token "your_fcm_token" --title "Hello" --body "Test"

# Full example with all options
npm run send -- --title "New Location" --body "Check out this spot" --path "location" --resource-id "12345"
```

### Available Options

| Option | Description |
|--------|-------------|
| `--token` | Device FCM token |
| `--title` | Notification title |
| `--body` | Notification body |
| `--action` | Action type (default: "path") |
| `--path` | Navigation path (default: "location") |
| `--resource-id` | Resource ID to navigate to |
| `--immediate` | Show immediately (default: "true") |
| `--geofence` | WKT polygon for geofencing |

## Geofencing with --geofence

The `--geofence` option allows you to send notifications that only appear to users within a specific geographic area.

### WKT Polygon Format

WKT (Well-Known Text) is a standard format for geographic shapes. A polygon looks like:

```
POLYGON((lon1 lat1, lon2 lat2, lon3 lat3, lon1 lat1))
```

**Important:** Coordinates are **longitude first, then latitude** (opposite of the usual "lat, long" convention).

### Example

A rectangle around central London:
```
POLYGON((-0.15 51.48, -0.15 51.52, -0.08 51.52, -0.08 51.48, -0.15 51.48))
```

### How It Works

1. Notification is sent with the WKT polygon in the `geoarea` field
2. App receives it and parses the WKT string into coordinates
3. App checks the user's current location against the polygon
4. **If user is INSIDE the polygon** → notification is shown and stored
5. **If user is OUTSIDE** → notification is silently discarded

### Usage Example

Send a notification only to users physically in Southampton:

```bash
npm run send -- \
  --title "Southampton Event" \
  --body "Join us at the marina!" \
  --geofence "POLYGON((-1.42 50.89, -1.42 50.92, -1.38 50.92, -1.38 50.89, -1.42 50.89))"
```

Users outside that area won't see the notification.

## App Implementation Details

The geofencing logic is handled by the following files in the app:

**1. Notification Data Model** (`lib/src/features/notifications/models/notification_data.dart`)

The `geofence` field (mapped from `geoarea` in JSON) stores the WKT polygon string. The model provides a `geofencePolygon` getter that parses this into coordinates:

```dart
List<LatLng> get geofencePolygon {
  if (geofence == null || geofence!.isEmpty) return [];
  return getGeofencePolygon(geofence!);
}
```

**2. WKT Parser** (`lib/src/helpers/wkt_helper.dart`)

The `getGeofencePolygon()` function parses WKT POLYGON strings into a list of `LatLng` coordinates using regex:

```dart
final polygonRegex = RegExp(r'POLYGON\s*\(\s*\(\s*([^)]+)\s*\)\s*\)');
```

**3. Point-in-Polygon Check** (`lib/src/helpers/wkt_helper.dart`)

The `isPointInPolygon()` function uses the **Ray Casting Algorithm** to determine if the user's location is inside the polygon. It casts a horizontal ray from the point and counts intersections with polygon edges - an odd count means inside, even means outside.

**4. FCM Service** (`lib/src/features/notifications/services/fcm_service.dart`)

The `_checkGeofenceBounds()` method ties it all together:

```dart
Future<bool> _checkGeofenceBounds(NotificationData notificationData) async {
  List<LatLng> geofencePolygon = notificationData.geofencePolygon;

  // If no geofence, allow the notification
  if (geofencePolygon.isEmpty) return true;

  // Get user's current position
  LatLng? currentPosition = await PositionHelper.getCurrentPosition();

  // Check if user is inside the polygon
  return currentPosition != null &&
      isPointInPolygon(currentPosition, geofencePolygon);
}
```

This check runs when a notification is received. If it returns `false`, the notification is silently discarded and never shown to the user.

**5. Position Helper** (`lib/src/helpers/position.dart`)

The geofence check uses `flutter_background_geolocation` to get the user's current position:

```dart
var position = await bg.BackgroundGeolocation.getCurrentPosition(
  desiredAccuracy: bg.Config.DESIRED_ACCURACY_MEDIUM,
  timeout: Config.locationTimeout,
);
```

This is a **one-time position check** when the notification arrives, not continuous monitoring.

## Why Data Messages + flutter_local_notifications?

The Node.js tool sends **data-only messages** (not notification messages). This means FCM delivers the data silently to the app, and the app decides whether/how to show a notification using `flutter_local_notifications`.

### Benefits of Data Messages

| Aspect | Data Messages | Notification Messages |
|--------|---------------|----------------------|
| Auto-displays | No - app controls | Yes - system handles |
| Business logic | Full control | Limited |
| Filtering | Can check geofence, expiry, preferences | Shows immediately |
| Storage | Can store before showing | Harder to intercept |

### App Processing Flow

Before showing a notification, the app checks (`lib/src/features/notifications/services/fcm_service.dart`):

1. **User preferences** - are notifications enabled?
2. **Immediate flag** - should it show now?
3. **Expiry date** - has it expired?
4. **Geofence** - is user in the target area?

Only after all checks pass does it call `flutterLocalNotificationsPlugin.show()`.

## App Initialization (`main.dart`)

FCM is set up when the app starts:

```dart
await FcmService().initialiseFirebaseMessaging();

// Background messages (app closed/background)
FirebaseMessaging.onBackgroundMessage(onBackgroundMessage);

// Foreground messages (app open)
FirebaseMessaging.onMessage.listen((message) =>
    FcmService().handleForegroundNotification(message));

// Token refresh
FirebaseMessaging.instance.onTokenRefresh.listen((fcmToken) {
    FcmService().sendToken(token: fcmToken);
});
```

## Deep Linking / Navigation

When a notification is tapped, the app navigates based on `action` and `path` fields:

| Action | Path | Navigation |
|--------|------|------------|
| `path` | `location` | ViewLocationScreen (tab 1) |
| `path` | `report` | ViewLocationScreen (tab 2) |
| `path` | `boat` | ViewBoatScreen |
| `path` | `member` | ViewMemberScreen |
| `externalUrl` | URL | Opens external browser |

This is handled in `lib/src/features/notifications/screens/notifications_list_screen.dart`.

## Notification Storage

Notifications are stored locally in Hive (`lib/src/features/notifications/services/notification_repository.dart`):

- Stored when received (after passing all checks)
- Marked as read/unread
- Auto-deleted when expired (30 days retention or custom expiry date)

## User Notification Settings

Users can control notifications via settings stored in SharedPreferences (`lib/src/helpers/shared_preferences.dart`):

| Setting | Key | Default | Controls |
|---------|-----|---------|----------|
| Push Notifications | `notification_enabled` | `true` | FCM push notifications |
| Geofence Notifications | `geofence_notification_enabled` | `true` | Location-based notifications |

The FCM service checks `notificationEnabled` before showing any notification:

```dart
bool isNotificationEnabled = await SharedPreferencesHelper().notificationEnabled;
if (!isNotificationEnabled) return; // Skip showing notification
```

## Notification UI

A dedicated screen shows notifications (`lib/src/features/notifications/screens/notifications_list_screen.dart`):

- **Unread/Read tabs** - toggle between states
- **Search** - filter notifications
- **Auto-refresh** - polls every 3 seconds for new notifications
- **Pull-to-refresh** - manual refresh

### Unread Notification Modal

When a user logs in, if they have unread notifications, a modal popup appears (`lib/src/features/notifications/widgets/modals/unread_notification_modal.dart`):

```
Login → Check for unread notifications → Show modal → User taps "View" → Opens notification list
```

This is triggered in the authentication flow (`lib/src/screens/authentication_screen.dart`) and displayed on the home screen.

## Sending to Multiple Devices

The CLI tool sends to one device at a time. The `index.js` library supports broader sending:

```javascript
const { initializeFirebase, sendDataMessageToMultiple, sendDataMessageToTopic } = require('./index');

initializeFirebase();

// Multiple specific devices
await sendDataMessageToMultiple(
  ['token1', 'token2', 'token3'],
  { title: 'Hello', body: 'Message' }
);
```

## Token Management

- **On app start**: Token sent to backend via `sendToken()`
- **On token refresh**: Automatically re-sent to backend
- **On logout**: Token revoked via `FcmService().revokeToken()`

**Note:** The backend API endpoint for storing tokens is not yet implemented (`lib/src/features/notifications/api/notification_api_client.dart`).

## Related Documentation

- [Location Geofence Notifications](./LOCATION_GEOFENCE_NOTIFICATIONS.md) - A separate notification system triggered when users physically approach locations

## Troubleshooting

**"Failed to initialize Firebase"**
- Ensure `firebase-service-account.json` exists in the notifications folder
- Check the file contains valid JSON

**"Invalid registration token"**
- FCM tokens expire when the app is reinstalled
- Get a fresh token from the app

**"FCM token is required"**
- Set `TEST_FCM_TOKEN` in your `.env` file, or
- Use `--token` parameter when running the command