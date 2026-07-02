---
id: 37ceba22-f034-4ac4-b7a1-dcbe0bf0524b
blueprint: captains_mate_app_43
title: 'Create Edit Report Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1777653444
parent: faa4011a-a306-467e-ac40-635e775f6e76
---
# CreateEditReportScreen

**File:** `lib/src/screens/create_edit_report_screen.dart`
**Route:** `/create_edit_report`

---

## Purpose

A multi-purpose form screen for creating, editing, saving as draft, and deleting reports associated with a location. It operates in three distinct modes depending on the route arguments:

1. **Create mode** (`report == null`, `isCreatingLocation == false`) - Post a new report to an existing location, with an option to save as a local draft.
2. **Edit mode** (`report != null`) - Update or delete an existing report.
3. **Create-location mode** (`isCreatingLocation == true`) - Attach a report to a location that is currently being created (not yet persisted). The report is stored in the `createEditLocationProvider` state rather than sent to the API.

All modes support an optional single image attachment.

---

## UI Simplified Structure

```
TcaSafeScaffold
 ├── AppBar ("ADD REPORT" or "EDIT REPORT")
 │
 └── body: SingleChildScrollView > Form
      │
      ├── Text (location name)
      │
      ├── TcaFormField (Title - plain text, optional)
      │
      ├── TcaFormField (Report Text - HTML rich editor, required)
      │    └── HtmlEditor (html_editor_enhanced)
      │
      ├── Image Upload Section
      │    ├── TcaButton ("ADD IMAGE" - outlined, opens AddImageModal)
      │    └── _buildSelectedImageWidget()   ← inline preview with delete icon
      │         ├── Image.file (80x80 thumbnail)
      │         ├── Text (title, description, filename)
      │         └── IconButton (delete/remove)
      │
      └── Action Buttons (conditional on mode)
           │
           ├── [Create mode]
           │    ├── TcaButton ("POST REPORT")
           │    └── TcaButton ("SAVE AS DRAFT" - outlined)
           │
           ├── [Edit mode]
           │    ├── TcaButton ("SAVE CHANGES")
           │    └── TcaButton ("DELETE REPORT" - outlined, only if no comments)
           │
           └── [Create-location mode]
                └── TcaButton ("ADD REPORT")
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `reportListProvider` | `lib/src/features/reports/providers/report_list_provider.dart` | Riverpod `Notifier` (keepAlive) that exposes `createReport`, `updateReport`, `deleteReport`, `uploadReportImage`, and `deleteReportImage`. Orchestrates API calls and Hive persistence via `LocationRepository` |
| `authenticationProvider` | `lib/src/features/auth/providers/auth_provider.dart` | Reads `currentUser?.name` to stamp the author name on new reports |
| `repositoryManagerProvider` | `lib/core/providers/repository_provider.dart` | Accesses `DraftReportRepository` to save/delete local draft reports in Hive |
| `draftReportProvider` | `lib/src/data/providers/draft_report.dart` | `FutureProvider` that loads all draft reports from `DraftReportRepository`. Invalidated after saving or posting a draft |
| `createEditLocationProvider` | `lib/src/data/providers/create_edit_location.dart` | `StateProvider<CreateEditLocationRequest>` - in create-location mode, the report is written back into this provider's state via `copyWith(report: ...)` |
| `locationsProvider` | `lib/src/data/providers/location.dart` | Indirectly invalidated by `reportListProvider` after any create/update/delete to ensure the locations list is refreshed |

### Models

| Model | File | Role |
|---|---|---|
| `Report` | `lib/src/features/reports/models/report.dart` | Hive-persisted model with `id`, `title`, `text`, `authorName`, `comments`, `isEditable`, etc. Used in edit mode |
| `CreateReportRequest` | `lib/src/features/reports/api/requests/create_report_request.dart` | Freezed model for creating reports. Also doubles as the draft-report Hive model (has `draftId`, `lastEdited`, `images`) |
| `UpdateReportRequest` | `lib/src/features/reports/api/requests/update_report_request.dart` | Freezed model sent to the API when editing a report (`PATCH /report/{id}`) |
| `CreateLocationReportRequest` | `lib/src/features/locations/api/requests/create_location_report_request.dart` | Lightweight model for attaching a report to a location-in-progress (create-location flow). Holds `title`, `text`, and `images` |
| `ImageUploadRequest` | `lib/src/features/locations/api/requests/image_upload_request.dart` | Freezed model holding `filePath`, `title`, `description` for a pending image upload |
| `Location` | `lib/src/features/locations/models/location.dart` | Parent entity - reports are nested within locations |
| `CreateEditLocationRequest` | `lib/src/features/locations/models/create_edit_location_request.dart` | Draft location model used when creating a new location; the report is attached to its `report` field |
| `CreateEditReportScreenArguments` | (defined in-file, line 35) | Route arguments carrying `location`, `draftLocation`, `report`, `draftReport`, and `isCreatingLocation` |
| `SelectedLocationRoute` | `lib/src/helpers/route_argumnets/selected_location_route.dart` | Holds `id` and `tabIndex` for navigating to `ViewLocationScreen` after post/save |

### Widgets

| Widget | File | Role |
|---|---|---|
| `TcaFormField` | `lib/src/widgets/tca_form_field.dart` | Polymorphic form field supporting `text`, `textarea`, `html`, `file`, `select`, `multiselect`, and `number` types. Used here for the title (text) and report body (html) fields |
| `AddImageModal` | `lib/src/features/reports/widgets/modals/add_image_modal.dart` | Bottom sheet modal with title, file picker, and description fields. Returns an `ImageUploadRequest` via `onImageAdded` callback |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | Styled action button used for all form actions |
| `TcaDialog` | `lib/src/widgets/tca_dialog.dart` | Alert dialog wrapper used for error messages and delete confirmation |

---

## API / Backend Dependencies

All API calls are mediated through `reportListProvider.notifier` which delegates to `ReportsApiClient` and `LocationApiClient`.

| Action | API Client | Endpoint | Description |
|---|---|---|---|
| Create report | `ReportsApiClient` | `POST /report` | Creates a report, returns `EntityID` |
| Update report | `ReportsApiClient` | `PATCH /report/{reportId}` | Patches title/text, returns updated `LastMod` |
| Delete report | `ReportsApiClient` | `DELETE /report/{reportId}` | Removes the report |
| Upload report image | `ReportsApiClient` (via `ImageUploadMixin`) | `POST /report/{reportId}/images` | Multipart file upload |
| Delete report image | `LocationApiClient` | `DELETE /mLocation/{locationId}/images?imageurl={url}` | Images are stored at the location level, so deletion goes through the location API |
| Refresh location images | `LocationApiClient` | `GET /mLocation/{locationId}/images` | Called after upload/delete to sync the full image list back into local storage |

Draft reports are stored locally only via `DraftReportRepository` (Hive box: `draft_reports`) and never touch the API.

---

## Data Flow

### Create & Post Report

```
User fills form → taps "POST REPORT"
  │
  ├─ Validate form (_formKey)
  │
  ├─ reportListProvider.createReport()
  │    ├─ POST /report → returns Report with EntityID
  │    ├─ Save updated Location (with new report) to Hive
  │    └─ Invalidate locationsProvider
  │
  ├─ If image selected:
  │    ├─ Append report to _location.reports (local copy)
  │    └─ reportListProvider.uploadReportImage()
  │         ├─ POST /report/{id}/images (multipart)
  │         ├─ GET /mLocation/{id}/images (refresh images)
  │         ├─ Save updated Location to Hive
  │         └─ Invalidate locationsProvider
  │
  ├─ If was a draft: delete from DraftReportRepository, invalidate draftReportProvider
  │
  └─ Navigate: popUntil(first) → push ViewLocationScreen (tab: Reports)
       └─ Show snackbar "Report posted"
```

### Save as Draft

```
User fills form → taps "SAVE AS DRAFT"
  │
  ├─ Build CreateReportRequest (new draftId via Uuid, or update existing)
  │
  ├─ DraftReportRepository.save(draft, draftId)
  │
  ├─ Invalidate draftReportProvider
  │
  └─ Navigate: popUntil(first) → push ViewLocationScreen (tab: Reports)
       └─ Show snackbar "Report saved as draft"
```

### Edit & Save Changes

```
User edits form → taps "SAVE CHANGES"
  │
  ├─ Validate form
  │
  ├─ reportListProvider.updateReport()
  │    ├─ PATCH /report/{id} → returns UpdateReportResponse (lastMod)
  │    ├─ Save updated Location to Hive
  │    └─ Invalidate locationsProvider
  │
  ├─ Image handling:
  │    ├─ If new image selected → uploadReportImage()
  │    └─ Else if previous image removed → deleteReportImage()
  │
  └─ Navigator.pop → Show snackbar "Report changes saved"
```

### Delete Report

```
User taps "DELETE REPORT" → TcaDialog confirmation
  │
  ├─ reportListProvider.deleteReport()
  │    ├─ DELETE /report/{id}
  │    ├─ Save updated Location (report removed) to Hive
  │    └─ Invalidate locationsProvider
  │
  └─ Navigator.pop → Show snackbar "Report deleted"
```

### Create-Location Mode

```
User fills form → taps "ADD REPORT"
  │
  ├─ Validate form
  │
  ├─ Build CreateLocationReportRequest (title, text, images)
  │
  ├─ Write to createEditLocationProvider.state via copyWith(report: draft)
  │
  └─ Navigator.pop (returns to create-location flow)
```

---

## Navigation Callers

The screen is navigated to from four places:

| Source | File | Mode |
|---|---|---|
| "ADD FURTHER REPORT" button | `lib/src/widgets/tca_location_reports.dart` | Create (existing location) |
| Report "Edit" action | `lib/src/widgets/tca_location_reports.dart` | Edit (existing report) |
| Draft report tap | `lib/src/screens/my_drafts_screen.dart` | Create with pre-filled draft |
| Report step in create-location | `lib/src/screens/create_edit_location/progress_screen.dart` | Create-location mode |

---

## Known Caveats / Tech-Debt Notes

1. **Route arguments reassigned in `build()`** (lines 188-195): `_location`, `_report`, `_draftLocation`, `_draftReport`, and `_isCreatingLocation` are re-extracted from `ModalRoute` arguments on every rebuild and assigned to mutable instance fields. This means any in-method mutations to `_location` (e.g. line 393, 509) are overwritten on the next build cycle. The code works around this because the mutations happen within async methods that complete before the next build, but this pattern is fragile.

2. **Image not persisted in drafts**: The `ImageUploadRequest.images` field is annotated with `@JsonKey(includeFromJson: false, includeToJson: false)` on `CreateReportRequest`, so saved drafts lose their attached images when the app restarts. The `CreateLocationReportRequest` has the same exclusion for images.

3. **`_isDeleting` flag not reset on error** (line 568): In `_deleteReport`, on catch the code resets `_isLoading` instead of `_isDeleting`, so the delete button stays in its loading state permanently after a failed delete.

4. **Image upload failure is non-blocking on create**: If `uploadReportImage` fails after a successful `createReport`, the report is still posted but the image is silently lost (error dialog shown but no rollback). The user may not realise the image was not attached.

5. **`selectedImage!.filePath` force-unwrap in `reportListProvider`**: `uploadReportImage` in the notifier does `File(selectedImage!.filePath)` with a force-unwrap on `selectedImage`, even though the parameter type is `ImageUploadRequest?`.

6. **Delete only allowed when no comments**: The "DELETE REPORT" button is conditionally shown only when `_report!.comments.isEmpty`. This is a business rule (reports with community replies cannot be deleted) but is not communicated to the user - the button simply disappears.

7. **Navigation after post uses `popUntil(first)` then `pushNamed`**: This clears the entire navigation stack back to root and then pushes `ViewLocationScreen`. If the user navigated through a deep stack (e.g. from the map), they lose their navigation history.

8. **`_title` controller conditionally initialised with `??=`** (line 197): The controller is only created on the first build. If route arguments change (unlikely but possible with key reuse), the title won't update.

9. **HTML editor content retrieved async**: Both `_postReport` and `_saveDraftReport` call `await _html.getText()` which communicates with the webview. If the webview is in a bad state, this can hang or return empty content without warning.