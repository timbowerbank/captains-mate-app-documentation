---
id: fcfe1331-c567-4581-984f-21278a6cfd3a
blueprint: captains_mate_app_43
title: 'Help Support Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774533777
---
# HelpSupportScreen

## Purpose

`HelpSupportScreen` is a static informational screen that provides users with:

- Help documentation explaining how the app works
- Link to online help resources (CA website)
- Support contact information (email)
- Glossary of terms used in the app (location types)

This is a read-only screen with no user input beyond tapping external links.

**Route:** `/help_support`

**Navigation:** This screen is accessible from the app drawer.

| Source | File | Context |
|--------|------|---------|
| App drawer | [tca_drawer.dart:90](lib/src/widgets/tca_drawer.dart#L90) | "Help / Support" menu item with support icon |

---

## UI Structure

```
HelpSupportScreen (StatelessWidget)
├── Scaffold
│   ├── AppBar
│   │   └── Title: "HELP / SUPPORT"
│   │
│   └── Body (SingleChildScrollView)
│       └── Padding
│           └── Column
│               ├── "Help" section
│               │   ├── Heading: "Help"
│               │   ├── RichText: App description (Locations, Overview, Info)
│               │   ├── RichText: Reports and Images explanation
│               │   ├── Text: Online help prompt
│               │   └── TcaButton: "Help (Online)" → Opens CA website
│               │
│               ├── "Support" section
│               │   ├── Heading: "Support"
│               │   ├── Text: Issue reporting prompt
│               │   ├── TcaButton: "Contact / Support" → Opens email
│               │   └── Text: Email address display
│               │
│               └── "Terms Used in this App" section
│                   ├── Heading: "Terms Used in this App"
│                   ├── RichText: Location types intro
│                   └── Bulleted list:
│                       ├── Anchor/Buoy
│                       ├── Local quay
│                       ├── Marina
│                       ├── Boatyard
│                       ├── Port
│                       ├── Other
│                       └── Omnibus
```

---

## Associated Components

### Providers

None - this is a static `StatelessWidget` with no state management.

### Models

None - displays hardcoded content only.

### Widgets (Bespoke)

| Widget | Location | Purpose |
|--------|----------|---------|
| `TcaButton` | `widgets/tca_button.dart` | Action buttons for external links |

### Helpers

| Helper | Location | Purpose |
|--------|----------|---------|
| `LauncherHelper` | `helpers/launcher.dart` | Opens external URLs (website, email) |
| `SnackbarHelper` | `helpers/snackbar.dart` | Shows error message if email app unavailable |

### Values / Constants

| Value | Location | Purpose |
|-------|----------|---------|
| `Dimens` | `values/dimens.dart` | Spacing constants |
| `Styles` | `values/styles.dart` | Text styles (`appBarTitle`, `helpTextHeading`, `helpText`, `helpTextBold`) |

---

## External Dependencies

| Package | Usage |
|---------|-------|
| `url_launcher` | `canLaunchUrl` to check if email app is available |
| `intersperse` | Adds spacing between bullet point widgets |

---

## API / Backend Dependencies

None - this is a static content screen.

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                       USER ACTIONS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Tap "Help (Online)" button                                          │
│        │                                                             │
│        └── LauncherHelper.launch(                                    │
│              'https://www.theca.org.uk/help/captainsmate'            │
│            )                                                         │
│              └── Opens external browser                              │
│                                                                      │
│  Tap "Contact / Support" button                                      │
│        │                                                             │
│        ├── canLaunchUrl('mailto:...')                                │
│        │     └── If false → SnackbarHelper: "Unable to open..."      │
│        │                                                             │
│        └── LauncherHelper.launch('mailto:captains.mate@theca.org.uk')│
│              └── Opens email client                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Known Caveats / Tech Debt

### 1. Email Launch Logic Issue
The email button checks `canLaunchUrl` but still calls `launch` even if the check fails:

```dart
onTap: () async {
  var url = 'mailto:captains.mate@theca.org.uk';
  if (!(await canLaunchUrl(Uri.parse(url)))) {
    SnackbarHelper.showSnackBar(
        message: 'Unable to open email application');
  }
  LauncherHelper.launch(url);  // Called regardless of canLaunchUrl result
},
```

Should have an `else` or `return` to prevent launching when unavailable.

### 2. Hardcoded Content
All help text is hardcoded in the widget. If content needs to change frequently, consider:
- Moving to a configuration file
- Fetching from a CMS/API
- Using localisation strings

### 3. Hardcoded URLs
The help URL and email address are hardcoded:
- `https://www.theca.org.uk/help/captainsmate`
- `captains.mate@theca.org.uk`

Should be moved to a constants file or configuration.

### 4. Unused Import
The `intersperse` package is imported but could be replaced with standard Flutter patterns (e.g., `ListView.separated` or manual spacing).

### 5. No Loading State for URL Launch
The buttons don't show any loading indicator while checking/launching URLs. Users might tap multiple times if there's a delay.

### 6. Missing Error Handling for LauncherHelper
If `LauncherHelper.launch()` fails, there's no error handling or user feedback.

---

## File References

- Screen: [help_support_screen.dart](lib/src/screens/help_support_screen.dart)
- Launcher Helper: [launcher.dart](lib/src/helpers/launcher.dart)
- Drawer: [tca_drawer.dart](lib/src/widgets/tca_drawer.dart)