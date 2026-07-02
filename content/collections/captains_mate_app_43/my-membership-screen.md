---
id: fc5ce6c0-5e81-44ef-81ed-0ecf22cde55b
blueprint: captains_mate_app_43
title: 'My Membership Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780405926
parent: faa4011a-a306-467e-ac40-635e775f6e76
---
# MyMembershipScreen

## Purpose

`MyMembershipScreen` is a simple display screen that shows the current user's membership card. It decodes a base64-encoded card image and displays it along with the member's name and membership number. This serves as a digital membership card that users can show for identification.

**Route:** `/my_membership`

**Navigation:** This screen is accessible from the app drawer.

| Source | File | Context |
|--------|------|---------|
| App drawer | [tca_drawer.dart:79](lib/src/widgets/tca_drawer.dart#L79) | "My Membership" menu item with card icon |

---

## UI Structure

```
MyMembershipScreen (StatelessWidget)
├── TcaSafeScaffold
│   ├── AppBar
│   │   └── Title: "MY MEMBERSHIP"
│   │
│   └── Body (Center)
│       └── Consumer (Riverpod)
│           └── Column (centered)
│               ├── Image.memory (base64-decoded card image)
│               ├── Text: currentUser.name
│               └── Text: currentUser.membership.memberNumber
```

---

## Associated Components

### Providers

| Provider | Type | Purpose |
|----------|------|---------|
| `authenticationProvider` | `Riverpod` | Provides `currentUser` with membership data |

### Models

| Model | Source | Purpose |
|-------|--------|---------|
| `CurrentUser` | `features/user/models/current_user.dart` | User data including `name` and `membership` |
| `Membership` (nested) | Part of `CurrentUser` | Contains `cardImg` (base64) and `memberNumber` |

### Widgets (Bespoke)

None - this screen only uses standard Flutter widgets.

### Values / Constants

| Value | Location | Purpose |
|-------|----------|---------|
| `Dimens` | `values/dimens.dart` | Margin constants |
| `Styles` | `values/styles.dart` | Text styles (`appBarTitle`, `dialogTextBold`) |

---

## External Dependencies

| Package | Usage |
|---------|-------|
| `flutter_riverpod` | `Consumer` widget to watch `authenticationProvider` |
| `dart:convert` | `base64Decode` for decoding the card image |

---

## API / Backend Dependencies

None - this screen displays data already loaded in the `authenticationProvider` state. The membership data is fetched during authentication/login.

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                           DATA FLOW                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  authenticationProvider.currentUser                                  │
│        │                                                             │
│        ├── currentUser.membership.cardImg (base64 string)            │
│        │     │                                                       │
│        │     └── base64Decode() → Image.memory()                     │
│        │                                                             │
│        ├── currentUser.name → Text widget                            │
│        │                                                             │
│        └── currentUser.membership.memberNumber → Text widget         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Known Caveats / Tech Debt

### 1. Force Unwraps Without Null Checks (CRASH RISK)
Multiple force unwraps that could crash if data is missing:

```dart
var currentUser = ref.watch(authenticationProvider).currentUser!;  // Force unwrap
// ...
Image.memory(
  base64Decode(currentUser.membership!.cardImg!),  // Double force unwrap
),
// ...
currentUser.membership!.memberNumber ?? '',  // Force unwrap, but null-coalesces memberNumber
```

If `currentUser`, `membership`, or `cardImg` is null, the app will crash. Should add null checks or a loading/error state.

### 2. No Loading State
The screen assumes data is always available. If the authentication state is still loading or the membership data hasn't been fetched, the force unwraps will crash.

### 3. No Error Handling for Invalid Base64
If `cardImg` contains invalid base64 data, `base64Decode()` will throw a `FormatException`. No try-catch wrapper exists.

### 4. StatelessWidget with Consumer
The screen uses `StatelessWidget` with a `Consumer` inside. This is valid but could be simplified to `ConsumerWidget`:

```dart
// Current approach
class MyMembershipScreen extends StatelessWidget {
  Widget build(BuildContext context) {
    return Scaffold(
      body: Consumer(
        builder: (context, ref, child) {
          // ...
        },
      ),
    );
  }
}

// Cleaner approach
class MyMembershipScreen extends ConsumerWidget {
  Widget build(BuildContext context, WidgetRef ref) {
    // ...
  }
}
```

### 5. No Placeholder for Missing Card Image
If the card image is null or fails to decode, there's no fallback placeholder image.

### 6. Text Overflow on Name
The name text uses `overflow: TextOverflow.clip` which will cut off long names without indication. Consider `TextOverflow.ellipsis` or wrapping.

---

## File References

- Screen: [my_membership_screen.dart](lib/src/screens/my_membership_screen.dart)
- Auth Provider: [auth_provider.dart](lib/src/features/auth/providers/auth_provider.dart)
- Current User Model: [current_user.dart](lib/src/features/user/models/current_user.dart)
- Drawer: [tca_drawer.dart](lib/src/widgets/tca_drawer.dart)