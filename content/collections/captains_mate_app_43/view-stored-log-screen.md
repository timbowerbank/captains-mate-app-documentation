---
id: f71d0049-11a5-43bd-9a42-935112b3a7ce
blueprint: captains_mate_app_43
title: 'View Stored Log Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774535779
---
# ViewStoredLogScreen

**File:** `lib/src/screens/settings/view_stored_log_screen.dart`
**Route:** `/view-stored-log-screen`

---

## Purpose

A developer-only diagnostic screen for viewing and uploading device logs. It is hidden behind a 5-tap developer mode toggle in `SettingsScreen`. The screen displays device info at the top, a log-level selector dropdown, and a live-streaming list of log entries (newest first). An "Upload Logs" button in the bottom navigation bar sends the logs to a remote endpoint. The log stream uses `TcaLog.whereCreatedAfter()` which emits log entries in real time.

---

## UI Simplified Structure

```
Scaffold
 ├── AppBar
 │    └── Text ("DEVICE LOGS")
 │
 ├── body: Column
 │    │
 │    ├── _LogHeader (ConsumerStatefulWidget)
 │    │    ├── DeviceInfoHeader
 │    │    │    └── Device model, OS, app version
 │    │    ├── InfoItem ("Log Level")
 │    │    │    └── TcaDropdownButton<LogLevel>
 │    │    │         ├── values: all LogLevel enum values
 │    │    │         └── onChanged → setState + sharedPreferences + TcaLog.setLogLevel
 │    │    ├── TcaSettingHeader ("Logs")
 │    │    └── Divider (1px border)
 │    │
 │    └── _LogBody (StatefulWidget)
 │         └── Expanded > Container (offWhite background)
 │              └── Scrollbar > ListView
 │                   └── ..._logs.reversed.map → Text (log entry, trimmed)
 │
 └── bottomNavigationBar: SafeArea > Consumer
      └── TcaButton ("Upload Logs")
           ├── isLoading: isUploading
           └── onTap → read logEnabledAt → TcaLog.upload() → snackbar
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `sharedPreferencesProvider` | `lib/src/data/providers/shared_preferences.dart` | Read in `_LogHeader` to get `currentLogLevel` on init. Written to via `setCurrentLogLevel()` when the user changes the dropdown. Read in the upload button to get `logEnabledAt` timestamp |

### Models / Enums

| Model / Enum | File | Role |
|---|---|---|
| `LogModel` | `lib/src/features/logs/models/log_model.dart` (via `logs/index.dart`) | Represents a single log entry. Has `id` (for deduplication) and `toString()` for display |
| `LogLevel` | `lib/src/enums/log_level.dart` | Enum for log verbosity levels (e.g. verbose, debug, info, warning, error). Has `asString` getter for display names. Stored as integer index in SharedPreferences |

### Widgets

| Widget | File | Role |
|---|---|---|
| `DeviceInfoHeader` | `lib/src/widgets/settings/device_info_header.dart` | Displays device metadata (model, OS version, app version) at the top |
| `TcaDropdownButton` | `lib/src/widgets/tca_dropdown_button.dart` | Generic dropdown used for the log level selector. Typed as `TcaDropdownButton<LogLevel>` |
| `TcaSettingHeader` | `lib/src/widgets/settings/setting_header.dart` | Section header text ("Logs") |
| `InfoItem` | `lib/src/widgets/settings/setting_info_item.dart` | Labelled info row wrapping the dropdown ("Log Level") |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | "Upload Logs" button with loading state in the bottom bar |

### Helpers

| Helper | File | Role |
|---|---|---|
| `TcaLog` | `lib/src/helpers/log.dart` | `whereCreatedAfter()` — returns a `Stream<List<LogModel>>` of log entries. `upload(logEnabledAt:)` — uploads logs to the remote endpoint. `setLogLevel()` — changes the active log verbosity level |
| `SnackbarHelper` | `lib/src/helpers/snackbar.dart` | Shows success/error messages after upload attempt |

---

## API / Backend Dependencies

| Call | Service | Description |
|---|---|---|
| `TcaLog.upload(logEnabledAt:)` | Internal logging service | Uploads stored logs to a remote endpoint (likely Sentry or a custom server). The `logEnabledAt` timestamp filters which logs to include. The exact endpoint is encapsulated within `TcaLog` |

No application REST API calls are made directly from this screen.

---

## Data Flow

### Log Level Initialisation

```
_LogHeader.initState
  │
  └─ Post-frame callback:
       └─ sharedPreferencesProvider.currentLogLevel
            └─ Parse int string → LogLevel.values[index]
                 └─ setState(logLevel = parsed value)
```

### Log Level Change

```
User selects new level from dropdown
  │
  ├─ setState(logLevel = newValue)
  ├─ sharedPreferencesProvider.setCurrentLogLevel(newValue)
  └─ TcaLog.setLogLevel(newValue)
```

### Live Log Stream

```
_LogBody.initState
  │
  └─ TcaLog.whereCreatedAfter().listen((List<LogModel> event)
       │
       └─ For each log in event:
            ├─ Check if log.id already in _logs (deduplication)
            └─ If new: setState → _logs.add(log)

  Display: _logs.reversed → newest entries at top
```

### Upload Logs

```
User taps "Upload Logs"
  │
  ├─ setState(isUploading = true)
  │
  ├─ sharedPreferencesProvider.logEnabledAt → DateTime?
  │
  ├─ TcaLog.upload(logEnabledAt: logEnabledAt)
  │    └─ Uploads to remote endpoint
  │
  ├─ .then → SnackbarHelper("Logs uploaded successfully")
  │
  ├─ .catchError → SnackbarHelper("An error occurred when uploading logs...")
  │
  └─ .whenComplete → setState(isUploading = false)
```

---

## Navigation Callers

| Source | File | Context |
|---|---|---|
| SettingsScreen | `lib/src/screens/settings/settings_screen.dart` | "View Logs" item, only visible when developer mode is enabled (hidden 5-tap toggle) |

---

## Known Caveats / Tech-Debt Notes

1. **`_LogBody.dispose()` calls `super.dispose()` before cancelling subscription** (lines 163-167): The `_subscription?.cancel()` is called after `super.dispose()`. The conventional order is to clean up resources first, then call super. If the stream emits during disposal, `setState` could be called on a disposed widget.

2. **Log deduplication is O(n) per event** (lines 155-158): Each incoming log is checked against the entire `_logs` list via `_logs.map((l) => l.id).contains(element.id)`. As the log list grows, this becomes increasingly expensive. A `Set<String>` of seen IDs would be more efficient.

3. **`LogLevel` parsed from string index** (line 95): `currentLogLevel` returns a string that is parsed as `int` and used as an index into `LogLevel.values`. If the stored value is corrupted or out of range, `int.parse` or the index access could throw. There is no error handling for this.

4. **`RouteAware` mixin declared but never used** (line 18): `ViewStoredLogScreen` mixes in `RouteAware` (`with RouteAware`) but never subscribes to a `RouteObserver` or overrides any `RouteAware` methods (`didPush`, `didPop`, etc.). This is dead code.

5. **No pagination or virtual scrolling**: All log entries are held in memory in the `_logs` list and rendered in a `ListView` (not `ListView.builder`). For long-running sessions with verbose logging, this could consume significant memory and cause jank as the list grows.

6. **Stream subscription not paused when screen is not visible**: The `_subscription` continues to receive and process log events even if the user navigates away (since `_LogBody` remains in the widget tree as part of the `Column`). This is wasteful but harmless since the screen is only reachable via developer mode.

7. **Upload does not show what was uploaded**: After a successful upload, there is no indication of how many logs were sent or what time range was covered. The user has no feedback beyond the generic success snackbar.

8. **`_LogHeader` and `_LogBody` are separate widgets with no shared state**: The log level dropdown in `_LogHeader` changes the log level via `TcaLog.setLogLevel()`, but `_LogBody`'s stream (`whereCreatedAfter()`) was started in `initState` with the previous level. Changing the log level does not restart the stream or filter existing entries — it only affects future log calls. Already-displayed entries at the old level remain visible.

9. **Three-widget split across the file**: The screen is split into `ViewStoredLogScreen` (main scaffold + upload button), `_LogHeader` (ConsumerStatefulWidget), and `_LogBody` (StatefulWidget). The upload state (`isUploading`) lives on the main screen, the log level lives on the header, and the log entries live on the body. This split makes it difficult to coordinate state (e.g. disabling upload while the log level is changing).