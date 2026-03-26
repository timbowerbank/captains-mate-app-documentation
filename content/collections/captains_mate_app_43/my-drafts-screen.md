---
id: 0da2891d-8c32-47b9-b2f9-56b288c9fc26
blueprint: captains_mate_app_43
title: 'My Drafts Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774533879
---
# MyDraftsScreen

## Purpose

`MyDraftsScreen` displays the user's locally saved draft locations and draft reports. Users can resume editing drafts or delete them. Drafts are stored locally and allow users to save work in progress when creating new locations or reports.

**Route:** `/my_drafts`

**Navigation:** This screen is accessible from the app drawer.

| Source | File | Context |
|--------|------|---------|
| App drawer | [tca_drawer.dart:74](lib/src/widgets/tca_drawer.dart#L74) | "My Drafts" menu item with edit icon |

---

## UI Structure

```
MyDraftsScreen (ConsumerStatefulWidget)
├── Scaffold
│   ├── AppBar
│   │   └── Title: "DRAFTS"
│   │
│   └── Body (Padding)
│       └── Column
│           │
│           ├── [If no drafts]
│           │   └── Text: "No drafts saved" (centered)
│           │
│           └── [If drafts exist]
│               ├── "Draft Locations" section
│               │   └── List of TcaItemCard
│               │       ├── Title: draft.name
│               │       ├── Subtitle: "Last edited X ago"
│               │       ├── Tap → Navigate to ProgressScreen
│               │       └── Delete icon → Confirmation dialog
│               │
│               └── "Draft Reports" section
│                   └── List of TcaItemCard
│                       ├── Title: draft.title or "Untitled report"
│                       ├── Subtitle: location.name + "Last edited X ago"
│                       ├── Tap → Navigate to CreateEditReportScreen
│                       └── Delete icon → Confirmation dialog
```

---

## Associated Components

### Providers

| Provider | Type | Purpose |
|----------|------|---------|
| `locationsProvider` | `FutureProvider<List<Location>>` | Fetches locations to display location name for draft reports |
| `draftLocationProvider` | `FutureProvider<List<CreateEditLocationRequest>>` | Fetches locally saved draft locations |
| `draftReportProvider` | `FutureProvider.autoDispose<List<CreateReportRequest>>` | Fetches locally saved draft reports |
| `createEditLocationProvider` | `StateProvider` | Set when resuming a draft location edit |
| `repositoryManagerProvider` | `Provider` | Access to repositories for deleting drafts |

### Models

| Model | Source | Purpose |
|-------|--------|---------|
| `Location` | `features/locations/models/location.dart` | Location data for displaying report's parent location name |
| `CreateEditLocationRequest` | `features/locations/models/` | Draft location data with `name`, `lastEdited`, `draftId` |
| `CreateReportRequest` | `features/reports/index.dart` | Draft report data with `title`, `locationId`, `lastEdited`, `draftId` |

### Widgets (Bespoke)

| Widget | Location | Purpose |
|--------|----------|---------|
| `TcaItemCard` | `widgets/tca_item_card.dart` | Card wrapper for each draft item |
| `TcaDialog` | `widgets/tca_dialog.dart` | Confirmation dialog for delete actions |
| `TcaButton` | `widgets/tca_button.dart` | Dialog action buttons |

### Helpers

| Helper | Location | Purpose |
|--------|----------|---------|
| `SnackbarHelper` | `helpers/snackbar.dart` | Shows confirmation snackbar after deletion |

### Repositories

| Repository | Location | Purpose |
|------------|----------|---------|
| `DraftLocationRepository` | `features/locations/services/` | Local storage for draft locations |
| `DraftReportRepository` | `features/reports/services/` | Local storage for draft reports |

### Enums

| Enum | Location | Purpose |
|------|----------|---------|
| `TcaButtonVariant` | `enums/button_variant.dart` | Button styling for dialog actions |

### Values / Constants

| Value | Location | Purpose |
|-------|----------|---------|
| `AppColors` | `values/colors.dart` | Delete icon colour |
| `Dimens` | `values/dimens.dart` | Spacing constants |
| `Styles` | `values/styles.dart` | Text styles (`appBarTitle`, `draftSectionTitle`, `draftTitle`, `draftSubtitle`, `draftSubtitleLight`, `dialogText`) |

---

## External Dependencies

| Package | Usage |
|---------|-------|
| `flutter_riverpod` | State management - `ConsumerStatefulWidget`, providers |
| `timeago` | Formats `lastEdited` as relative time (e.g., "5 minutes ago") |

---

## API / Backend Dependencies

None - this screen works entirely with local storage (drafts are stored on device, not synced to server).

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INITIALIZATION                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  initState() → addPostFrameCallback                                  │
│        │                                                             │
│        └── _loadProviders()                                          │
│              │                                                       │
│              ├── locationsProvider.future → locations                │
│              ├── draftReportProvider.future → draftReports           │
│              └── draftLocationProvider.future → draftLocations       │
│                    │                                                 │
│                    └── setState() → Update local state               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                       USER ACTIONS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Tap Draft Location                                                  │
│        │                                                             │
│        ├── createEditLocationProvider.state = draft                  │
│        └── Navigator.pushNamed(ProgressScreen)                       │
│                                                                      │
│  Tap Draft Report                                                    │
│        │                                                             │
│        └── Navigator.pushNamed(CreateEditReportScreen)               │
│              └── arguments: { location, draftReport }                │
│                                                                      │
│  Delete Draft Location                                               │
│        │                                                             │
│        ├── Show TcaDialog confirmation                               │
│        │     └── If confirmed:                                       │
│        │           ├── DraftLocationRepository.delete(draftId)       │
│        │           ├── ref.invalidate(draftLocationProvider)         │
│        │           └── SnackbarHelper: "Draft location deleted"      │
│                                                                      │
│  Delete Draft Report                                                 │
│        │                                                             │
│        ├── Show TcaDialog confirmation                               │
│        │     └── If confirmed:                                       │
│        │           ├── DraftReportRepository.delete(draftId)         │
│        │           ├── ref.invalidate(draftReportProvider)           │
│        │           └── SnackbarHelper: "Draft report deleted"        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Known Caveats / Tech Debt

### 1. List Not Updated After Deletion
After deleting a draft, `ref.invalidate()` is called to refresh the provider, but the local state (`draftLocations`, `draftReports`) is not updated. The UI won't reflect the deletion until the user navigates away and back.

```dart
if (shouldDelete) {
  await ref.read(repositoryManagerProvider)
      .getRepository<DraftReportRepository>()
      .delete(draft.draftId!);

  ref.invalidate(draftReportProvider);  // Provider invalidated
  // But local `draftReports` list not updated - UI stale

  SnackbarHelper.showSnackBar(message: 'Draft report deleted');
}
```

**Fix:** Call `_loadProviders()` after deletion or use `ref.watch()` in build.

### 2. Force Unwrap on `draft.name` and `draft.draftId`
Several force unwraps that could crash:

```dart
title: draft.name!,  // Could be null
// ...
.delete(draft.draftId!);  // Could be null
```

### 3. No Error Handling for Missing Location
When building report items, `firstWhere` is used without `orElse`, which throws if no location matches:

```dart
var location = locations.firstWhere((l) => l.id == draft.locationId);
// Throws StateError if location not found
```

Should use `firstWhereOrNull` and handle the null case.

### 4. No Loading State
The screen shows empty state immediately while providers are loading. Should show a loading indicator until `_loadProviders()` completes.

### 5. Data Loaded in `initState` Instead of Using `ref.watch`
The screen loads data once in `initState` via `addPostFrameCallback` and stores it in local state. This is an anti-pattern with Riverpod - should use `ref.watch()` in `build()` for reactive updates.

### 6. No Pull-to-Refresh
Users cannot refresh the drafts list manually.

### 7. Delete Confirmation Not Null-Checked
The delete dialog result is used without null check:

```dart
var shouldDelete = await showDialog(...);

if (shouldDelete) {  // Could be null if dialog dismissed
  // ...
}
```

This works because `null` is falsy, but explicit `shouldDelete == true` would be clearer.

---

## File References

- Screen: [my_drafts_screen.dart](lib/src/screens/my_drafts_screen.dart)
- Draft Location Provider: [draft_location.dart](lib/src/data/providers/draft_location.dart)
- Draft Report Provider: [draft_report.dart](lib/src/data/providers/draft_report.dart)
- Draft Location Repository: [draft_location_repository.dart](lib/src/features/locations/services/draft_location_repository.dart)
- Draft Report Repository: [draft_report_repository.dart](lib/src/features/reports/services/draft_report_repository.dart)
- Drawer: [tca_drawer.dart](lib/src/widgets/tca_drawer.dart)