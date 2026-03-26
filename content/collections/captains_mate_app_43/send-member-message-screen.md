---
id: 4249f462-f299-420a-9e6a-37ece384678f
blueprint: captains_mate_app_43
title: 'Send Member Message Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774534780
---
# SendMemberMessageScreen

## Purpose

`SendMemberMessageScreen` is a simple form screen that allows users to send an email message to another member. The message is sent via the API, which presumably forwards it as an email to the recipient member.

**Route:** `/member/send_message`

**Navigation:** This screen is only accessible from the member detail view when the member is emailable.

| Source | File | Context |
|--------|------|---------|
| Member detail | [view_member_screen.dart:267](lib/src/screens/members/view_member_screen.dart#L267) | "Email the Member" button (shown if `member.emailable` is true) |

---

## UI Structure

```
SendMemberMessageScreen (StatefulWidget)
├── Scaffold
│   ├── AppBar
│   │   └── Title: "NEW MESSAGE"
│   │
│   └── Body (Padding)
│       ├── Column
│       │   ├── Text: "Send a message via email to"
│       │   ├── Text: member.fullName
│       │   │
│       │   ├── Form
│       │   │   └── TcaFormField (textarea)
│       │   │       └── Label: "Message" (required)
│       │   │
│       │   ├── Spacer
│       │   │
│       │   └── TcaButton: "SEND"
```

---

## Associated Components

### Providers

This screen does not use any Riverpod providers - it's a simple `StatefulWidget` that makes a direct API call.

### Models

| Model | Source | Purpose |
|-------|--------|---------|
| `Member` | `features/members/models/member.dart` | Member data (used to display `fullName`) |
| `SendMemberMessageScreenArguments` | Defined in same file | Route arguments containing `Member` and member `id` |

### Widgets (Bespoke)

| Widget | Location | Purpose |
|--------|----------|---------|
| `TcaFormField` | `widgets/tca_form_field.dart` | Multi-purpose form field (used as textarea here) |
| `TcaButton` | `widgets/tca_button.dart` | Standard app button with loading state |

### Helpers

| Helper | Location | Purpose |
|--------|----------|---------|
| `SnackbarHelper` | `helpers/snackbar.dart` | Shows success/error snackbar messages |

### Enums

| Enum | Location | Purpose |
|------|----------|---------|
| `TcaFormFieldType` | `enums/form_field_type.dart` | Form field type (uses `textarea`) |

### Values / Constants

| Value | Location | Purpose |
|-------|----------|---------|
| `Dimens` | `values/dimens.dart` | Spacing constants |
| `Styles` | `values/styles.dart` | Text styles (`appBarTitle`, `sendMessageHelper`, `sendMessageName`) |

---

## External Dependencies

| Package | Usage |
|---------|-------|
| `form_field_validator` | `RequiredValidator` for message field validation |

---

## API / Backend Dependencies

| Endpoint | Method | Usage |
|----------|--------|-------|
| `ApiClient.members.sendMessage(id, message)` | POST | Sends email message to member |

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INITIALIZATION                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Route Arguments (SendMemberMessageScreenArguments)               │
│        │                                                             │
│        ├── member: Member (for display)                              │
│        └── id: String (for API call)                                 │
│                                                                      │
│  2. initState()                                                      │
│        │                                                             │
│        └── Create TextEditingController with listener                │
│              └── setState() on every text change (for button state)  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                       USER ACTIONS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Tap "SEND" button                                                   │
│        │                                                             │
│        ├── Validate form                                             │
│        │     └── If invalid → Show error on field                    │
│        │                                                             │
│        ├── Set _isLoading = false (BUG: should be true)              │
│        │                                                             │
│        ├── ApiClient.members.sendMessage(id, message)                │
│        │     │                                                       │
│        │     ├── Success → SnackbarHelper: "Message successfully     │
│        │     │             sent" → Navigator.pop()                   │
│        │     │                                                       │
│        │     └── Error → SnackbarHelper: error message               │
│        │                 → Set _isLoading = true (BUG: should be     │
│        │                   false)                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Known Caveats / Tech Debt

### 1. Loading State Logic is Inverted (BUG)
The `_isLoading` state is set incorrectly - the boolean values are swapped:

```dart
void _sendMessage(String id) async {
  // ...
  setState(() {
    _isLoading = false;  // BUG: Should be true (starting load)
  });

  try {
    await ApiClient.members.sendMessage(...);
  } catch (_) {
    // ...
    setState(() {
      _isLoading = true;  // BUG: Should be false (load finished)
    });
    return;
  }
  // ...
}
```

This means the button won't show a loading indicator during the API call, and will incorrectly show loading after an error.

### 2. Member Extracted in `build()`
The `_member` is extracted from route arguments inside `build()` rather than `didChangeDependencies()`:

```dart
@override
Widget build(BuildContext context) {
  var args = ModalRoute.of(context)!.settings.arguments as SendMemberMessageScreenArguments;
  _member = args.member;
  // ...
}
```

### 3. No Error Logging
Errors from the API call are caught but not logged - only a generic snackbar is shown:

```dart
} catch (_) {
  SnackbarHelper.showSnackBar(
    message: 'An error occurred while sending message, please try again later',
  );
  // No TcaLog.error() call
}
```

### 4. Redundant `id` in Arguments
The `SendMemberMessageScreenArguments` contains both `member` and `id`, but `member` already contains an `id`. The separate `id` parameter may be for a different purpose (perhaps the authenticated user's ID?), but this is unclear and potentially confusing.

### 5. TextEditingController Listener Triggers Unnecessary Rebuilds
The controller listener calls `setState(() {})` on every keystroke, which rebuilds the entire widget tree. This is likely intended to update button state but could be more efficient with a focused rebuild.

### 6. No Confirmation Before Sending
There's no confirmation dialog before sending the message - tapping "SEND" immediately initiates the API call.

---

## File References

- Screen: [send_member_message_screen.dart](lib/src/screens/send_member_message_screen.dart)
- Member Model: [member.dart](lib/src/features/members/models/member.dart)
- Member API Client: [member_api_client.dart](lib/src/features/members/api/member_api_client.dart)
- Form Field Widget: [tca_form_field.dart](lib/src/widgets/tca_form_field.dart)