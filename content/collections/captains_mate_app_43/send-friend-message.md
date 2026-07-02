---
id: 9904057e-24ed-4cfc-bc41-23360a06d3db
blueprint: captains_mate_app_43
title: 'Send Friend Message'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780407917
parent: faa4011a-a306-467e-ac40-635e775f6e76
---
# SendMessageScreen (Send Friend Message)

## Purpose

`SendMessageScreen` is a form screen that allows users to send an email message to a friend. The message is sent via the API, which forwards it as an email to the friend. This screen includes "boat show mode" support to obscure the friend's name for privacy.

**Route:** `/friend/send_message`

**Navigation:** This screen is only accessible from the friend modal on the home screen when the friend is emailable.

| Source | File | Context |
|--------|------|---------|
| Friend modal | [tca_home_screen_friend_modal.dart:67](lib/src/widgets/home_screen/tca_home_screen_friend_modal.dart#L67) | "Send an email" button (shown if `friend.emailable` is true) |

---

## UI Structure

```
SendMessageScreen (ConsumerStatefulWidget)
├── TcaSafeScaffold
│   ├── AppBar
│   │   └── Title: "NEW MESSAGE"
│   │
│   └── Body (Padding)
│       ├── Column
│       │   ├── Text: "Send a message via email to"
│       │   ├── Text: friend.displayName (obscured if boat show mode)
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

| Provider | Type | Purpose |
|----------|------|---------|
| `sharedPreferencesProvider` | Provider | Reads `boatShowMode` setting to obscure friend name |

### Models

| Model | Source | Purpose |
|-------|--------|---------|
| `Friend` | `features/friends/models/friend.dart` | Friend data (used to display `displayName` and get `id`) |
| `SendMessageScreenArguments` | Defined in same file | Route arguments containing `Friend` |

### Widgets (Bespoke)

| Widget | Location | Purpose |
|--------|----------|---------|
| `TcaFormField` | `widgets/tca_form_field.dart` | Multi-purpose form field (used as textarea here) |
| `TcaButton` | `widgets/tca_button.dart` | Standard app button with loading state |

### Helpers

| Helper | Location | Purpose |
|--------|----------|---------|
| `SnackbarHelper` | `helpers/snackbar.dart` | Shows success/error snackbar messages |

### Extensions

| Extension | Location | Purpose |
|-----------|----------|---------|
| `String.toObscured()` | `extensions/string.dart` | Obscures friend name in boat show mode |

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
| `flutter_riverpod` | State management - `ConsumerStatefulWidget`, `ref.read()` for shared preferences |
| `form_field_validator` | `RequiredValidator` for message field validation |

---

## API / Backend Dependencies

| Endpoint | Method | Usage |
|----------|--------|-------|
| `ApiClient.friends.sendMessage(friendId, message)` | POST | Sends email message to friend |

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INITIALIZATION                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Route Arguments (SendMessageScreenArguments)                     │
│        │                                                             │
│        └── friend: Friend                                            │
│                                                                      │
│  2. initState()                                                      │
│        │                                                             │
│        ├── Create TextEditingController with listener                │
│        │     └── setState() on every text change                     │
│        │                                                             │
│        └── Post-frame callback                                       │
│              └── sharedPreferencesProvider.boatShowMode              │
│                    └── setState() → _boatShowMode                    │
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
│        ├── ApiClient.friends.sendMessage(friendId, message)          │
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
Same bug as `SendMemberMessageScreen` - the `_isLoading` state is set incorrectly:

```dart
void _sendMessage() async {
  // ...
  setState(() {
    _isLoading = false;  // BUG: Should be true (starting load)
  });

  try {
    await ApiClient.friends.sendMessage(...);
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

This means the button won't show a loading indicator during the API call.

### 2. Friend Extracted in `build()`
The `_friend` is extracted from route arguments inside `build()`:

```dart
@override
Widget build(BuildContext context) {
  var args = ModalRoute.of(context)!.settings.arguments as SendMessageScreenArguments;
  _friend = args.friend;
  // ...
}
```

### 3. No Error Logging
Errors from the API call are caught but not logged:

```dart
} catch (_) {
  SnackbarHelper.showSnackBar(
    message: 'An error occurred while sending message, please try again later',
  );
  // No TcaLog.error() call
}
```

### 4. Force Unwrap on `friend.id`
The code assumes `friend.id` is not null:

```dart
await ApiClient.friends.sendMessage(
  friendId: _friend.id!,  // Force unwrap - could crash if id is null
  message: _message.text,
);
```

### 5. TextEditingController Listener Triggers Unnecessary Rebuilds
Same issue as `SendMemberMessageScreen` - calls `setState(() {})` on every keystroke.

### 6. Class Name Mismatch
The class is named `SendMessageScreen` but the file is `send_friend_message_screen.dart`. This could cause confusion with the member messaging screen. Consider renaming to `SendFriendMessageScreen` for clarity.

### 7. Inconsistent with SendMemberMessageScreen
These two screens are nearly identical but have subtle differences:
- This one uses `ConsumerStatefulWidget`, member version uses `StatefulWidget`
- This one has boat show mode support, member version doesn't
- Different argument class names (`SendMessageScreenArguments` vs `SendMemberMessageScreenArguments`)
- Different route patterns (`/friend/send_message` vs `/member/send_message`)

Consider extracting a shared base widget or utility.

---

## File References

- Screen: [send_friend_message_screen.dart](lib/src/screens/send_friend_message_screen.dart)
- Friend Model: [friend.dart](lib/src/features/friends/models/friend.dart)
- Friends API Client: [friends_api_client.dart](lib/src/features/friends/api/friends_api_client.dart)
- Friend Modal: [tca_home_screen_friend_modal.dart](lib/src/widgets/home_screen/tca_home_screen_friend_modal.dart)
- Form Field Widget: [tca_form_field.dart](lib/src/widgets/tca_form_field.dart)