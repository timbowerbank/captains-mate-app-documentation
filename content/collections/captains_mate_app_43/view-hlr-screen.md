---
id: 18079840-a557-4a54-9dda-b78d0a74803d
blueprint: captains_mate_app_43
title: 'View HLR Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780411700
parent: faa4011a-a306-467e-ac40-635e775f6e76
---
# ViewHlrScreen

## Purpose

`ViewHlrScreen` is the detail view for displaying a single HLR (Harbour/Local Representative). It presents HLR contact information and related marine locations. The screen supports:

- Viewing HLR details including name, contact details, and URLs
- Displaying the HLR header image
- Navigating to related marine locations
- Real-time updates when HLR data changes elsewhere in the app
- Handling deleted/updated resources with modal notifications

**Route:** `/hlr`

**Navigation:** This screen is only accessible from the map popup in `TcaMap`. When a user taps a location marker, if:
1. The location has `primaryHlrs` (at least one HLR associated)
2. The user has the `internal_hlrs` attribute filter active

...a "VIEW HLR" button appears in the popup, which navigates to this screen.

**Entry point:** [tca_map.dart:444](lib/src/widgets/tca_map.dart#L444)

---

## UI Structure

```
ViewHlrScreen (ConsumerStatefulWidget)
├── TcaSafeScaffold
│   ├── AppBar (PreferredSize)
│   │   ├── BackButton
│   │   ├── HLR Name (Text)
│   │   └── "HLR" subtitle (Text)
│   │
│   └── NestedScrollView
│       ├── SliverAppBar (collapsible header)
│       │   ├── FlexibleSpaceBar
│       │   │   └── CachedNetworkImage / Placeholder Image
│       │   │
│       │   └── Tab-like header ("OVERVIEW" - static, not interactive)
│       │
│       └── SingleChildScrollView (body content)
│           ├── _buildHlrs
│           │   └── TcaHlrCard (contact details, emails, phones, URLs)
│           │
│           └── _buildNearby
│               └── TcaAsyncDataWrapper
│                   └── List of TcaItemCard (related locations)
│
└── Modals (shown conditionally)
    ├── TcaResourceUpdatedModal
    └── TcaResourceDeletedModal
```

---

## Associated Components

### Providers

| Provider | Type | Purpose |
|----------|------|---------|
| `hlrProvider` | `FutureProvider<List<Hlr>>` | Fetches all HLRs from repository (filtered to those with centroid or primaryLocationId) |
| `locationsProvider` | `FutureProvider<List<Location>>` | Fetches all locations for "Related Marine Locations" section |

### Models

| Model | Source | Purpose |
|-------|--------|---------|
| `Hlr` | `features/hlrs/models/hlr.dart` | Core HLR data model |
| `HlrAddress` | `features/hlrs/models/hlr_address.dart` | HLR address details |
| `HlrContactDetails` | `features/hlrs/models/hlr_contact_details.dart` | Contact information container |
| `HlrEmail` | `features/hlrs/models/hlr_email.dart` | Email with optional label |
| `HlrPhoneNumber` | `features/hlrs/models/hlr_phone_number.dart` | Phone number with optional label |
| `HlrUrl` | `features/hlrs/models/hlr_url.dart` | URL with optional label |
| `Location` | `features/locations/models/location.dart` | Used for related locations display |
| `SelectedLocationRoute` | `helpers/route_arguments/` | Route arguments containing HLR ID |

### Widgets (Bespoke)

| Widget | Location | Purpose |
|--------|----------|---------|
| `TcaHlrCard` | `widgets/tca_hlr_card.dart` | Displays HLR contact details (name, emails, phones, URLs) with boat show mode support |
| `TcaItemCard` | `widgets/tca_item_card.dart` | Card wrapper for related location items |
| `TcaAsyncDataWrapper` | `widgets/tca_async_data_wrapper.dart` | Handles async data loading states for locations |
| `TcaResourceUpdatedModal` | `widgets/modals/` | Modal shown when HLR data has been updated |
| `TcaResourceDeletedModal` | `widgets/modals/` | Modal shown when HLR has been deleted |
| `TcaTappableLink` | `widgets/tca_tappable_link.dart` | Tappable email/phone/URL links (used within TcaHlrCard) |

### Helpers

| Helper | Location | Purpose |
|--------|----------|---------|
| `TcaLog` | `helpers/log.dart` | Centralised error logging (used in image fetch catch block) |

### Values / Constants

| Value | Location | Purpose |
|-------|----------|---------|
| `AppColors` | `values/colors.dart` | App colour palette |
| `Dimens` | `values/dimens.dart` | Spacing and sizing constants |
| `Styles` | `values/styles.dart` | Text styles (`locationTitle`, `locationSubtitle`, `overviewSection`, etc.) |
| `Assets` | `values/assets.dart` | Asset paths (`placeholderHlrImage`, `locationMarker`, `locationMarkerPrimary`) |

---

## External Dependencies

| Package | Usage |
|---------|-------|
| `flutter_riverpod` | State management - `ConsumerStatefulWidget`, `ref.read()`, `ref.listen()` |
| `cached_network_image` | Efficient image loading with caching for HLR header image |
| `collection` | `firstWhereOrNull` - returns `null` instead of throwing when HLR/location not found |
| `flutter_svg` | Renders SVG location markers in related locations list |

---

## API / Backend Dependencies

| Endpoint | Method | Usage |
|----------|--------|-------|
| `ApiClient.hlrs.getHlrImage(hlrId)` | GET | Fetches HLR image separately after initial load |
| `HlrRepository.getAll()` | GET | Fetches all HLRs (via `hlrProvider`) |
| `LocationRepository.getAll()` | GET | Fetches all locations for related locations (via `locationsProvider`) |

**Note:** Similar to `ViewLocationScreen`, the HLR image is fetched separately after initial load to avoid blocking the render.

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INITIALIZATION                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Route Arguments (SelectedLocationRoute)                          │
│        │                                                             │
│        ▼                                                             │
│  2. initState() - extracts _hlrId                                    │
│        │                                                             │
│        ▼                                                             │
│  3. _loadHlr()                                                       │
│        │                                                             │
│        ├──► hlrProvider ──► Find HLR by ID                           │
│        │         │                                                   │
│        │         ▼                                                   │
│        │    HLR not found? ──► TcaResourceDeletedModal ──► Pop       │
│        │         │                                                   │
│        │         ▼                                                   │
│        └──► setState() ──► _originalHlr, _hlr                        │
│                                                                      │
│  4. ApiClient.hlrs.getHlrImage()                                     │
│        │                                                             │
│        ▼                                                             │
│     setState() ──► Update _hlr.images                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      REACTIVE UPDATES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ref.listen(hlrProvider) ──► _loadHlr()                              │
│                                    │                                 │
│                                    ▼                                 │
│                      _originalHlr.isUpdated(_hlr)?                   │
│                                    │                                 │
│                              Yes   │   No                            │
│                              ▼     │                                 │
│                   TcaResourceUpdatedModal                            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                       USER ACTIONS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Tap Related Location ──► Navigator.pushReplacementNamed             │
│         │                      (ViewLocationScreen)                  │
│         │                                                            │
│         └──► arguments: SelectedLocationRoute(id: locationId)        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Known Caveats / Tech Debt

### 1. Image Loading Strategy
Similar to `ViewLocationScreen`, the HLR image is fetched separately after initial load, which can cause a visual "pop" when the image appears:

```dart
var image = await ApiClient.hlrs.getHlrImage(_hlrId);
// ...
_hlr = hlr.copyWith(images: image != null ? [image] : null);
```

### 2. Location ID Extracted in `initState()`
Unlike `ViewLocationScreen` (which extracts in `build()`), this screen extracts `_hlrId` in `initState()`:

```dart
@override
void initState() {
  super.initState();
  _hlrId = (ModalRoute.of(context)!.settings.arguments as SelectedLocationRoute).id;
  // ...
}
```

**Note:** Using `ModalRoute.of(context)` in `initState()` can be problematic as the widget may not be fully mounted. This works but is fragile.

### 3. Modal Shown via Build Method Side-Effect
Same pattern as `ViewLocationScreen` - the `TcaResourceUpdatedModal` is triggered during `build()`:

```dart
bool isHlrUpdated = _originalHlr != null && _hlr != null && _originalHlr!.isUpdated(_hlr!);
if (isHlrUpdated) _showHlrUpdatedModal(context);
```

### 4. Reuses `SelectedLocationRoute` for HLR
The screen reuses `SelectedLocationRoute` for route arguments despite being an HLR screen. Consider creating a dedicated `SelectedHlrRoute` for clarity.

### 5. ScrollController Created in `build()`
A new `ScrollController` is created on every build:

```dart
final ScrollController _scrollController = ScrollController(); // In build()
```

### 6. Static "OVERVIEW" Tab
The `_buildTabs()` method renders a single "OVERVIEW" tab that isn't functional (no `TabController`). This appears to be for visual consistency with `ViewLocationScreen` but adds no functionality.

### 7. Comment Refers to "Location"
In `_loadHlr()`, a comment incorrectly references "location":

```dart
// Check if the location still exists (else show deleted modal)
if (_tempHlr == null) { ... }
```

### 8. Inconsistent isUpdated Parameter Order
The `isUpdated` check has parameters in opposite order compared to `ViewLocationScreen`:

```dart
// ViewHlrScreen
_originalHlr!.isUpdated(_hlr!)

// ViewLocationScreen
location!.isUpdated(originalLocation!)
```

This may indicate inconsistent method signatures on the models.

---

## File References

- Screen: [view_hlr_screen.dart](lib/src/screens/view_hlr_screen.dart)
- HLR Card: [tca_hlr_card.dart](lib/src/widgets/tca_hlr_card.dart)
- HLR Provider: [hlr.dart](lib/src/data/providers/hlr.dart)
- HLR Model: [hlr.dart](lib/src/features/hlrs/models/hlr.dart)
- HLR API Client: [hlrs_api_client.dart](lib/src/features/hlrs/api/hlrs_api_client.dart)