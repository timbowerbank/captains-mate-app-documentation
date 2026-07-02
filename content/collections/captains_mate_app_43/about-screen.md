---
id: 9ba7561f-a5fa-45d6-a89c-27266856622b
blueprint: captains_mate_app_43
title: 'About Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1777651809
parent: faa4011a-a306-467e-ac40-635e775f6e76
---
# AboutScreen

**File:** `lib/src/screens/about_screen.dart`
**Route:** `/about`

---

## Purpose

A static informational screen displaying legal disclaimers, terms and conditions for image uploads, copyright notices, and attribution credits. It links out to external websites for CoreBlue (developer), Lighthouse of London (designer), and OpenStreetMap/OpenSeaMap copyright details. The app version number is shown at the bottom.

---

## UI Simplified Structure

```
TcaSafeScaffold
 ├── AppBar
 │    └── Text ("ABOUT")
 │
 └── body: SingleChildScrollView > Padding > Column
      │
      ├── Text (legal disclaimer paragraph)
      ├── Text (usage terms paragraph)
      ├── Text ("By uploading images to this site, you:")
      ├── Padding > Column (numbered list of 5 image upload T&Cs)
      │
      ├── Text ("© Cruising Association 2012-{currentYear}")
      │
      ├── RichText ("Map data © OpenStreetMap...")
      │    └── TextSpan ("copyright details")  ← tappable, opens Constants.copyrightUrl
      │
      ├── RichText ("Design by Lighthouse of London")
      │    └── TextSpan ("Lighthouse of London")  ← tappable, opens Constants.lighthouseUrl
      │
      ├── RichText ("Software by CoreBlue")
      │    └── TextSpan ("CoreBlue")  ← tappable, opens Constants.coreblueUrl
      │
      └── TcaVersionNumber
```

---

## Associated

### Providers

None — this is a purely static screen with no Riverpod providers.

### Models

None.

### Widgets

| Widget | File | Role |
|---|---|---|
| `TcaVersionNumber` | `lib/src/widgets/tca_version_number.dart` | Displays the app version and build number at the bottom of the screen |

### Helpers

| Helper | File | Role |
|---|---|---|
| `LauncherHelper` | `lib/src/helpers/launcher.dart` | Opens external URLs in the system browser. Used for CoreBlue, Lighthouse, and copyright links |

### Constants

| Constant | File | Role |
|---|---|---|
| `Constants.coreblueUrl` | `lib/src/values/constants.dart` | URL for the CoreBlue developer website |
| `Constants.lighthouseUrl` | `lib/src/values/constants.dart` | URL for the Lighthouse of London designer website |
| `Constants.copyrightUrl` | `lib/src/values/constants.dart` | URL for OpenStreetMap/OpenSeaMap copyright details |

---

## API / Backend Dependencies

None — this screen is entirely static with no network calls.

---

## Navigation Callers

| Source | File | Context |
|---|---|---|
| TcaDrawer | `lib/src/widgets/tca_drawer.dart` | "About" menu item in the app drawer |
| AddGalleryImageScreen | `lib/src/screens/add_gallery_image_screen.dart` | "Terms and Conditions" link navigates here |

---

## Known Caveats / Tech-Debt Notes

1. **`TapGestureRecognizer` instances are never disposed** (lines 12-25): Three `TapGestureRecognizer` objects are created as instance fields on a `StatelessWidget`. Since `StatelessWidget` has no `dispose` lifecycle, these recognizers are never cleaned up. In practice the leak is negligible since the screen is short-lived, but it is technically incorrect — a `StatefulWidget` with proper disposal would be more correct.

2. **Dynamic copyright year**: The copyright text uses `DateTime.now().year` (line 92), so it updates automatically. This is fine but means the text is not truly "static" — it rebuilds with the current year on each widget build.

3. **No scroll-to-bottom indicator**: The screen contains a large amount of legal text. There is no visual hint that the content scrolls, which may cause users to miss the attribution and version info at the bottom.