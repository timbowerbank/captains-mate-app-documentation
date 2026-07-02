---
id: eac78dd9-50b4-4794-bc21-0bb2e3fa9877
blueprint: captains_mate_app_43
title: 'Add Gallery Image Screen'
use_synced_content: false
parent: faa4011a-a306-467e-ac40-635e775f6e76
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1777653374
---
# AddGalleryImageScreen
**File:** `lib/src/screens/add_gallery_image_screen.dart`
**Route:** `/add_gallery_image`

## Purpose

Allows a user to upload a new image to a location's gallery. The user provides an optional title, a required image file, and an optional description, and must accept the Terms and Conditions before uploading. On successful upload, the location's image metadata (URLs, titles) is refreshed from the API and the updated Location record is saved back to local Hive storage.

Route name: `/add_gallery_image`

---

## UI Structure (Simplified Tree)

```
TcaSafeScaffold
├── AppBar ("ADD GALLERY IMAGE")
│
└── body: LayoutBuilder
     └── SingleChildScrollView
          └── ConstrainedBox (minHeight: maxHeight)
               └── IntrinsicHeight
                    └── Padding
                         └── Form
                              └── Column
                                   ├── Text (location name)
                                   ├── TcaFormField ("Title" — optional, text)
                                   ├── TcaFormField ("Image" — required, file picker)
                                   ├── TcaFormField ("Description" — optional, textarea)
                                   ├── RichText
                                   │    └── "Terms and Conditions" link → AboutScreen
                                   ├── Spacer
                                   └── TcaButton ("UPLOAD IMAGE", isLoading: _isUploading)
```

---

## Associated Providers

| Provider | Type | Purpose |
|---|---|---|
| `locationsProvider` | `FutureProvider<List<Location>>` | Read after upload to find the updated location and apply the refreshed image list |
| `repositoryManagerProvider` | `Provider` | Accessed to save the updated `Location` (with new images) back to `LocationRepository` in Hive |

---

## Associated Models

| Model | Key Fields | Source |
|---|---|---|
| `Location` | `id`, `name`, `images` | Passed via route arguments. `id` used for API calls; `name` displayed at top of form |
| `AddGalleryImageScreenArguments` | `location` | Route argument wrapper — carries the `Location` to which the image is being added |

---

## Associated Widgets (Bespoke)

| Widget | File | Role |
|---|---|---|
| `TcaFormField` | `lib/src/widgets/tca_form_field.dart` | Used three times: text input for title, file picker for image, textarea for description |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | "UPLOAD IMAGE" action button, shows loading state during upload |
| `TcaDialog` | `lib/src/widgets/tca_dialog.dart` | Error dialog shown if the upload API call fails |

---

## API / Backend Dependencies

| Action | Method | Description |
|---|---|---|
| Upload image | `ApiClient.location.uploadLocationImage()` | POSTs the image file with optional title and description to the location's gallery |
| Refresh images | `ApiClient.location.getLocationImages()` | GETs the updated image list after a successful upload, used to update local storage |

---

## Data Flow

### Upload

The `location` is passed in via `AddGalleryImageScreenArguments` and extracted from route arguments in `build()`.

When the user taps "UPLOAD IMAGE":

1. Form validates — if the image field is empty, validation fails and the upload is aborted
2. `_isUploading` is set to `true`, showing a loading indicator on the button
3. `ApiClient.location.uploadLocationImage()` is called with the file and any optional title/description

If the upload fails, a `TcaDialog` is shown with the error message (using `ApiNoConnectionException.message` for connectivity errors, or `DioException.message` for API errors) and `_isUploading` returns to `false`.

### After Successful Upload

4. `ApiClient.location.getLocationImages()` fetches the updated image metadata for the location
5. The matching `Location` is found in `locationsProvider` and updated via `copyWith(images: newImages)` — this updates the image references (URLs, titles etc.) on the location record, not the image files themselves
6. The updated `Location` is saved back to `LocationRepository` in Hive
7. `locationsProvider` is invalidated to force a refresh
8. The screen pops and a success snackbar is shown

If step 4–7 fails, only a `TcaLog.error()` is produced. The user sees the success snackbar regardless and the local cache will remain stale until the next full sync.


## Known Caveats / Tech Debt

1. **`_location` extracted in `build()`** — Route arguments are read and assigned to `_location` inside `build()` (line 58–60) rather than in `initState()` or `didChangeDependencies()`. This is the same pattern noted in `SendMemberMessageScreen`.

2. **`_file` force-unwrapped on upload** — Line 150: `file: _file!`. The form validator marks the image field as required, but if validation somehow passes with `_file == null` this will crash.

3. **`firstWhere` without `orElse`** — Line 182: `locations.firstWhere((l) => l.id == _location.id)` throws a `StateError` if the location is not found. This could occur if the location was deleted remotely between the screen opening and the upload completing.

4. **Image refresh failure is silent to the user** — The `getLocationImages` call after upload (lines 178–199) is wrapped in its own try/catch. If it fails, only a log error is produced — the user sees a success snackbar and the screen pops, but the local cache won't reflect the new image until the next full sync.

5. **Controllers and gesture recognizer never disposed** — `_title`, `_description` (`TextEditingController`), and `_termsLink` (`TapGestureRecognizer`) are created in the state class but there is no `dispose()` override to release them, which leaks resources.