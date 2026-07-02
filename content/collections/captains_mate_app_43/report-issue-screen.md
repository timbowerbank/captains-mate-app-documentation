---
id: bcf7fc0b-9c77-43df-adeb-461c0549ad3d
blueprint: captains_mate_app_43
title: 'Report Issue Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780407866
parent: faa4011a-a306-467e-ac40-635e775f6e76
---
# ReportIssueScreen

**File:** `lib/src/screens/settings/report_issue_screen.dart`
**Route:** `/settings/report-issue`

---

## Purpose

A simple form screen for users to report issues or bugs to the app developers. The user enters a text description, which is submitted to Sentry as a user-feedback event associated with a captured message. Device information is shown at the top via a `DeviceInfoHeader` widget. On successful submission the screen pops and shows a success snackbar. A privacy notice explains that the description and any stored log file will be sent to the app's developers via Sentry.

---

## UI Simplified Structure

```
TcaSafeScaffold
 ├── AppBar
 │    └── Text ("REPORT AN ISSUE")
 │
 └── body: Form > SafeArea > Stack
      │
      ├── SingleChildScrollView > Column
      │    │
      │    ├── DeviceInfoHeader
      │    │    └── Displays device/OS/app version info
      │    │
      │    ├── TcaFormField (textarea)
      │    │    ├── label: "Describe the issue"
      │    │    ├── isRequired: true
      │    │    └── validator: RequiredValidator ("The description is required")
      │    │
      │    └── Text (privacy notice)
      │         └── "When you submit a report, the information you have
      │              entered along with any log file..."
      │
      └── Align (bottomCenter)
           └── TcaButton ("SUBMIT")
                ├── isLoading: _isSubmitting
                └── onTap → validate → submit to Sentry → pop
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `authenticationProvider` | `lib/src/features/auth/providers/auth_provider.dart` | Reads `currentUser` to get the user's name and email for the Sentry feedback payload |

### Models

| Model | File | Role |
|---|---|---|
| `CurrentUser` | `lib/src/features/user/models/current_user.dart` | Provides `name` and `email` fields passed to `SentryFeedback` |

### Widgets

| Widget | File | Role |
|---|---|---|
| `DeviceInfoHeader` | `lib/src/widgets/settings/device_info_header.dart` | Displays device metadata (model, OS version, app version) at the top of the form for context |
| `TcaFormField` | `lib/src/widgets/tca_form_field.dart` | Textarea input for the issue description, with required validation |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | "SUBMIT" button with loading state support |

### Helpers

| Helper | File | Role |
|---|---|---|
| `TcaLog` | `lib/src/helpers/log.dart` | `TcaLog.info()` — logs the report text locally before submitting to Sentry |
| `SnackbarHelper` | `lib/src/helpers/snackbar.dart` | Shows "The issue has been reported successfully" after submission |

### Third-Party

| Package | Usage |
|---|---|
| `sentry_flutter` | `Sentry.captureMessage()` — creates a Sentry event with level `info`. `Sentry.captureFeedback()` — attaches user feedback (name, email, message) to the event |
| `form_field_validator` | `RequiredValidator` — validates the description is not empty |

---

## API / Backend Dependencies

| Call | Service | Description |
|---|---|---|
| `Sentry.captureMessage('User reported issue', level: SentryLevel.info)` | Sentry SDK | Creates a new Sentry event and returns a `SentryId`. This is the anchor event for the feedback |
| `Sentry.captureFeedback(SentryFeedback(...))` | Sentry SDK | Attaches user feedback (name, email, description) to the event. Only called if `sentryId != SentryId.empty()` (i.e. the capture succeeded) |

No application REST API calls are made. The submission goes directly to Sentry's ingestion endpoint via the SDK.

---

## Data Flow

### Submit Flow

```
User taps "SUBMIT"
  │
  ├─ Validate form → return if invalid
  │
  ├─ setState(_isSubmitting = true)
  │
  ├─ Read currentUser from authenticationProvider
  │
  ├─ TcaLog.info("Reporting issue to CA: \n {text}")
  │    └─ Writes to local log (may be uploaded separately)
  │
  ├─ Sentry.captureMessage('User reported issue', level: info)
  │    └─ Returns SentryId
  │
  ├─ if sentryId != SentryId.empty():
  │    └─ Sentry.captureFeedback(SentryFeedback(
  │         associatedEventId: sentryId,
  │         name: currentUser.name,
  │         contactEmail: currentUser.email,
  │         message: _reportController.text.trim()
  │       ))
  │
  ├─ SnackbarHelper.showSnackBar("The issue has been reported successfully")
  │
  └─ Navigator.pop(context)
```

---

## Navigation Callers

| Source | File | Context |
|---|---|---|
| SettingsScreen | `lib/src/screens/settings/settings_screen.dart` | "Report an issue" item under the "Account" section |

---

## Known Caveats / Tech-Debt Notes

1. **`_isSubmitting` never reset to `false`** (line 90): The flag is set to `true` before submission but never set back to `false` — the screen pops on success. If the Sentry calls throw (e.g. no network), the error is unhandled and the button remains in a permanent loading state with no way to retry or dismiss. There is no try-catch around the Sentry calls.

2. **No error handling on Sentry submission**: Neither `Sentry.captureMessage` nor `Sentry.captureFeedback` is wrapped in try-catch. If the device is offline or Sentry is unreachable, the `await` on `captureMessage` could throw, leaving the screen in a loading state with no user feedback.

3. **Feedback only sent if `captureMessage` succeeds** (lines 103-109): If `Sentry.captureMessage` returns `SentryId.empty()` (e.g. event was dropped by rate limiting or sampling), the user's feedback is silently discarded. The success snackbar and navigation still proceed, giving the user the impression their report was submitted.

4. **`_reportController.text.trim()` called but original text logged** (lines 95, 108): The `TcaLog.info` call logs the untrimmed text (with leading/trailing whitespace), while the Sentry feedback sends the trimmed text. This inconsistency is minor but could cause confusion when comparing logs to Sentry events.

5. **Privacy notice is hardcoded text** (lines 73-76): The privacy notice mentions "Sentry" by name. If the error reporting provider changes, this text needs manual updating. It also mentions "log file from the app" being sent, but the log file upload is actually a separate action (from `ViewStoredLogScreen`), not automatically included with this report.

6. **No character limit on description**: The textarea has no maximum length constraint. A very long description could exceed Sentry's feedback message size limit, potentially causing silent truncation or rejection.