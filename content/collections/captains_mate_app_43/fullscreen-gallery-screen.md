---
id: b094bfd1-85ec-4bb6-93eb-6ef1518b85d1
blueprint: captains_mate_app_43
title: 'Fullscreen Gallery Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1777653423
parent: faa4011a-a306-467e-ac40-635e775f6e76
---
# FullscreenGalleryScreen

**File:** `lib/src/screens/fullscreen_gallery_screen.dart`
**Route:** `/fullscreen_gallery`

---

## Purpose

A fullscreen, zoomable image gallery for viewing location photos. Users can swipe through images with pinch-to-zoom support, view image metadata (title, author, caption), and delete their own images. The screen also respects a "boat show mode" that obscures author names for privacy.

---

## UI Simplified Structure

```
TcaSafeScaffold
 ├── AppBar (transparent, overlaid, light status bar icons)
 │
 └── body: Stack
      │
      ├── PhotoViewGallery.builder          ← Swipeable, zoomable image carousel
      │    └── PhotoViewGalleryPageOptions
      │         └── CachedNetworkImageProvider (per image)
      │
      └── Align (bottom)
           └── Container (semi-transparent black overlay)
                └── GalleryInfoWidget       ← Image metadata + delete action
                     ├── RichText (title + author name)
                     ├── TcaTextAccordion (expandable caption)
                     └── TcaButton ("Delete Image")  ← Only if image.isEditable
                          └── triggers TcaDialog (confirmation)
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `sharedPreferencesProvider` | `lib/src/data/providers/shared_preferences.dart` | Reads `boatShowMode` flag on init to determine whether to obscure author names |
| `locationsProvider` | `lib/src/data/providers/location.dart` | Read during image deletion to get the current locations list, then invalidated to refresh |
| `repositoryManagerProvider` | `lib/core/providers/repository_provider.dart` | Accesses `LocationRepository` to persist the updated location (with image removed) to local Hive storage |

### Models

| Model | File | Role |
|---|---|---|
| `Location` | `lib/src/features/locations/models/location.dart` | Freezed model; holds the `images` list and `id`. Passed in via route arguments. Uses `copyWith` to produce updated copies after deletion |
| `LocationImage` | `lib/src/features/locations/models/location_image.dart` | JSON-serialised model representing a single image (`url`, `thumbUrl`, `title`, `caption`, `authorName`, `isEditable`, `centerOfInterest`, `reportId`) |
| `FullscreenGalleryScreenArguments` | (defined in-file, line 20) | Simple args class carrying the `Location` and `initialIndex` to this screen |

### Widgets

| Widget | File | Role |
|---|---|---|
| `GalleryInfoWidget` | `lib/src/features/locations/widgets/gallery_info_widget.dart` | Bottom overlay showing title, author, expandable caption, and a conditional "Delete Image" button with confirmation dialog |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | Styled button used in the delete confirmation dialog |
| `TcaDialog` | `lib/src/widgets/tca_dialog.dart` | Styled `AlertDialog` wrapper used for error messages and delete confirmation |
| `TcaTextAccordion` | `lib/src/widgets/tca_accordian.dart` | Expandable/collapsible text widget used for long captions inside `GalleryInfoWidget` |

---

## API / Backend Dependencies

| Call | Client | Endpoint | Description |
|---|---|---|---|
| `ApiClient.location.deleteLocationImage(locationId, imageUrl)` | `LocationApiClient` (`lib/src/features/locations/api/location_api_client.dart`) | `DELETE /mLocation/{locationId}/images?imageurl={encodedUrl}` | Deletes a single image from a location on the server |

`ApiClient` is a static service-locator class (`lib/core/api/api_client.dart`) that resolves named Dio-based clients via Kiwi (dependency injection). The underlying HTTP layer uses Dio.

---

## Data Flow

```
┌──────────────┐    Navigator.pushNamed     ┌──────────────────────────┐
│  TcaGallery  │ ─────────────────────────> │ FullscreenGalleryScreen  │
│  (thumbnail  │   passes Location +        │                          │
│   carousel)  │   initialIndex as args     │  reads boatShowMode from │
└──────────────┘                            │  sharedPreferencesProvider│
                                            └────────────┬─────────────┘
                                                         │
                                              user taps "Delete Image"
                                                         │
                                                         v
                                            ┌─────────────────────────┐
                                            │ GalleryInfoWidget shows │
                                            │ TcaDialog confirmation  │
                                            └────────────┬────────────┘
                                                         │ confirmed
                                                         v
                                            ┌─────────────────────────┐
                                            │ _deleteImage()          │
                                            │  1. DELETE via API      │
                                            │  2. Read locationsProvider
                                            │  3. Remove image from   │
                                            │     location.images     │
                                            │  4. Save updated        │
                                            │     Location to Hive    │
                                            │     (LocationRepository)│
                                            │  5. Invalidate          │
                                            │     locationsProvider   │
                                            │  6. Show snackbar       │
                                            │  7. Pop if no images    │
                                            │     remain, else adjust │
                                            │     index               │
                                            └─────────────────────────┘
                                                   on error ↓
                                            ┌─────────────────────────┐
                                            │ TcaDialog with error    │
                                            │ message (DioException / │
                                            │ ApiNoConnectionException│
                                            └─────────────────────────┘
```

---

## Known Caveats / Tech-Debt Notes

1. **`_index` adjustment after delete may be incorrect** (line 199-202): After a successful deletion the code sets `_index = index == 0 ? _index! + 1 : _index! - 1`, but `index` here is the *list position of the location in `newLocations`*, not the image index. This variable name collision (`index` from `indexWhere` vs the image `_index`) is confusing and could produce wrong page-jump behaviour.

2. **Route arguments accessed via `ModalRoute` casting**: The screen relies on `ModalRoute.of(context)!.settings.arguments as FullscreenGalleryScreenArguments` with a force-unwrap (`!`). A malformed navigation call would crash at runtime.

3. **Location data passed by value, not by ref**: The `Location` object is passed in the route arguments and is not watched via a provider. After deleting an image, the screen manually patches the locations provider rather than reactively rebuilding from it. This means the gallery's own `location` reference is stale after deletion - the screen works around this by popping or adjusting the index.

4. **Hero tag collision**: Both `TcaGallery` and `FullscreenGalleryScreen` use a hardcoded `'gallery_hero_tag'` for hero animations. If multiple galleries existed on the same page, this would cause a Flutter hero animation error.

5. **Boat show mode loaded asynchronously in `initState`**: The `boatShowMode` flag is read via an async `SharedPreferences` call inside `loadProviders()`. The screen shows a loading spinner until this completes, adding a small delay to an otherwise instant screen transition.

6. **`LocationRepository` TODO**: The repository itself notes that location CRUD is partially migrated - "We will need to migrate all location calls across to this class at some point." The `save` call used here after image deletion is part of this in-progress migration.

7. **`elementAtOrNull` with force-unwrap on delete**: On line 150, `location.images.elementAtOrNull(_index!)` safely returns null, but the `.url` on line 150's `onDeleteImage` callback then calls `elementAtOrNull(...)!.url` with a force-unwrap, defeating the null-safety.

8. **No optimistic UI update**: The delete operation waits for the API response before updating the UI. There is no loading state shown on the delete button during the network call.