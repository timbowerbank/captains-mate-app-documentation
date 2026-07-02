---
id: a5eac0eb-2a0c-4a15-a253-df93e1eb3c86
blueprint: captains_mate_app_43
title: 'FCM Development Tool'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780395258
parent: 632ff8b5-8ee1-442d-ba83-4c7378ffd528
---
# FCM Notification Tool — Local Development

## Prerequisites

1. **Firebase Service Account Key** (JSON file)
2. **A device FCM token** to send to

## Setup

```bash
cd notifications

# Install dependencies
npm install

# Save the service account key file as:
# notifications/firebase-service-account.json
```

## Getting a Device FCM Token

Run the app in debug mode and add a breakpoint in the `getToken()` method in `lib/src/features/notifications/services/fcm_service.dart`:

```dart
Future<String?> getToken() async {
  return await _messaging.getToken().then((token) {
    TcaLog.info('FCM token retrieved: $token');
    return token;
  });
}
```

Copy the `token` value when the breakpoint hits.

Optionally save it to avoid retyping:
```bash
echo "TEST_FCM_TOKEN=paste_your_token_here" > .env
```

## Sending a Notification

```bash
# Basic send (uses token from .env)
npm run send

# Custom message
npm run send -- --title "Test Alert" --body "Hello from CLI"

# Specify token directly
npm run send -- --token "your_fcm_token" --title "Hello" --body "Test"

# Full example with all options
npm run send -- --title "New Location" --body "Check out this spot" --path "location" --resource-id "12345" --immediate true
```

## Available Options

| Option | Description |
|--------|-------------|
| `--token` | Device FCM token |
| `--title` | Notification title |
| `--body` | Notification body |
| `--action` | Action type (default: `path`) |
| `--path` | Navigation path (default: `location`) |
| `--resource-id` | Resource ID to navigate to |
| `--immediate` | Show immediately (default: `true`) |
| `--geofence` | WKT polygon for geofencing |

**Important:** `--action` must be `external_url` or `path`. Any other value will throw an error — the app's `NotificationAction` enum only accepts these two values.

## Geofencing with --geofence

Sends a notification that only appears to users inside a specific geographic area.

### WKT Polygon Format

```
POLYGON((lon1 lat1, lon2 lat2, lon3 lat3, lon1 lat1))
```

**Important:** Coordinates are **longitude first, then latitude** (opposite of the usual "lat, lon" convention).

A rectangle around central London:
```
POLYGON((-0.15 51.48, -0.15 51.52, -0.08 51.52, -0.08 51.48, -0.15 51.48))
```

### How the App Handles It

1. Receives the notification with the WKT polygon in the `geoarea` field
2. Checks the device's cached position (`PositionHelper.rawPosition`) against the polygon using the Ray Casting Algorithm
3. Inside the polygon → notification is stored and shown
4. Outside → silently discarded and never saved

```bash
npm run send -- \
  --title "Southampton Event" \
  --body "Join us at the marina!" \
  --geofence "POLYGON((-1.42 50.89, -1.42 50.92, -1.38 50.92, -1.38 50.89, -1.42 50.89))"
```

## Sending to Multiple Devices

The CLI tool sends to one device at a time. The `index.js` library supports broader sending:

```javascript
const { initializeFirebase, sendDataMessageToMultiple } = require('./index');

initializeFirebase();

await sendDataMessageToMultiple(
  ['token1', 'token2', 'token3'],
  { title: 'Hello', body: 'Message' }
);
```

## Troubleshooting

**"Failed to initialize Firebase"**
- Ensure `firebase-service-account.json` exists in the notifications folder
- Check the file contains valid JSON

**"Invalid registration token"**
- FCM tokens expire when the app is reinstalled — get a fresh token from the app

**"FCM token is required"**
- Set `TEST_FCM_TOKEN` in your `.env` file, or use `--token` directly

## Related Documentation

- [Notification System](./NOTIFICATIONS.md) - Full architecture overview