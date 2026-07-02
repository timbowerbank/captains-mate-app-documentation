---
id: d7be6b2e-4dcf-4ed7-a56d-79bf9ad9deac
blueprint: captains_mate_app_43
title: 'Offline Data'
use_synced_content: false
parent: a1b7cec3-99c2-4a5d-9c8b-9790b8204982
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780585063
---
# Sync Architecture

This document describes the complete data synchronisation flow, from authentication through app lifecycle into the batched sync engine.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Authentication Flow](#authentication-flow)
3. [Sync State Machine](#sync-state-machine)
4. [Batch Execution](#batch-execution)
5. [Queue Runner & Persistence](#queue-runner--persistence)
6. [Retry & Error Handling](#retry--error-handling)
7. [Settings Integration](#settings-integration)
8. [App Lifecycle Integration](#app-lifecycle-integration)
9. [Sync Trigger Points](#sync-trigger-points)
10. [Full State Flow Diagram](#full-state-flow-diagram)
11. [File Reference](#file-reference)

---

## Architecture Overview

The app uses a **batched, resumable synchronisation system** with five sequential batches, persistent queue state, per-batch retry logic, and graceful error handling. Sync is tightly coupled to authentication (token validity gates all API calls) and app lifecycle (pause/resume triggers re-checks).

Key design properties:
- Sync queue state is persisted to `SharedPreferences` so an interrupted sync can resume from the correct batch on the next run.
- An in-memory location cache is shared between batches 1–3 to avoid repeated Hive reads across 9,000+ locations.
- Delta sync uses a `lastSyncTfrom` timestamp set to the *batch start time* (not completion time), ensuring entities created during a long sync aren't missed on the next run.
- Error categories determine whether a failure retries or fails immediately — auth and bad-request errors stop the queue immediately, network errors retry with a configurable delay.

**State management:** Riverpod `StateNotifier` (`keepAlive: true`) for both auth and sync.  
**Persistence:** SharedPreferences (sync metadata), Hive (entity data), Flutter Secure Storage (tokens).

---

## Authentication Flow

**Files:**
- `lib/src/features/auth/providers/auth_provider.dart` — `Authentication` StateNotifier
- `lib/src/features/auth/domain/auth_state.dart` — `AuthState` (Freezed)
- `lib/src/helpers/auth.dart` — `AppAuth` secure storage helpers
- `lib/core/api/interceptors/auth_interceptor.dart` — DIO interceptor (header injection only)
- `lib/core/api/interceptors/refresh_interceptor.dart` — DIO interceptor (401/403 token refresh + retry)

### AuthState

```
accessToken
refreshToken
accessTokenExpiration
refreshTokenExpiration
currentUser
scopes

computed:
  isAuthenticated
```

### Login sequence

1. OAuth2 flow via `FlutterAppAuth` → access + refresh tokens acquired
2. Tokens saved to Keychain/Keystore via `SecureStorageHelper`
3. Current user fetched from API (skipped if fetched within the last 2 minutes — hardcoded in `auth_provider.dart`)
4. If user changed since last session: all local data cleared, repositories reinitialized
5. FCM token registered (if notifications permitted)
6. Sync triggered — `runSync` derives background mode from `lastSuccessfulSync` internally; foreground when null, background otherwise

### Token refresh

Two separate DIO interceptors handle auth:

**`auth_interceptor.dart`** — runs on every outgoing request:
- Reads the access token from `FlutterSecureStorage` and attaches `Authorization: Bearer {accessToken}`
- Attaches device/version headers (`CAAPI-ClientVersion`, `CAAPI-AppVersion`, `CAAPI-OSVersion`)

**`refresh_interceptor.dart`** — runs on every response:
- On 401/403: reads the refresh token directly from `FlutterSecureStorage`, calls `AppAuth.refreshToken()` (from `auth.dart`), saves the new tokens via `AppAuth.saveAuth()`, then retries the original request
- This bypasses the Riverpod `Authentication` notifier entirely — the notifier state holds stale tokens after an interceptor-triggered refresh (see tech debt in [authentication](/v4.3/overview/authentication)
- On refresh failure: calls `_redirectIfRetryFails()` which pushes to `AuthenticationScreen` via `OneContext` (guarded against redirecting if already on that screen)

Note: the sync's own `authError` category (`SyncState.error`) is a separate path — it fires when a batch API call returns 401/403 and the interceptor's refresh itself also fails, meaning the interceptor rethrows and the batch error propagates to `SyncQueueRunner`.

### First-run iOS behaviour

On first launch, the iOS Keychain is cleared before reading. This prevents stale tokens from a previous install being silently accepted.

---

## App Initialisation

`SyncNotifier.reset()` is called early in `lib/src/data/providers/app_initialisation_provider.dart` before authentication runs, ensuring the notifier is in a clean `idle` state (and `_cancelled = true`) regardless of any previous session state.

---

## Sync State Machine

**Files:**
- `lib/src/features/sync/providers/sync_provider.dart` — `SyncNotifier`
- `lib/src/features/sync/providers/sync_state.dart` — `SyncState` (Freezed)

### States

```
idle()
loading(step, isPreSync, retryAt, showBackgroundSyncButton)
success()
error(message, stack, isBackground, isFullSync)
```

`isPreSync` is `true` during the connectivity check that runs before batches begin.  
`retryAt` is set during the wait between retry attempts (used by the UI to show a countdown).

### How each state is reached

| State | Set when |
|-------|----------|
| `idle()` | App initialisation (`reset()`), app paused (`reset()` called on pause), or no sync required (`checkForSync` returned false with `force: false`) |
| `loading(isPreSync: true)` | `runBatchedSync` starts — synchronously before any batch runs; also during the connectivity check on non-startup syncs |
| `loading(isPreSync: false)` | Each batch progress step emitted by `onProgress`; also during the retry wait (`retryAt` populated, `showBackgroundSyncButton` shown after first failure) |
| `success()` | All batches completed, geofences synced, providers invalidated; also emitted immediately if a `SkipSyncException` is thrown |
| `error(isBackground: false)` | A batch fails beyond max retries (or fails immediately) during a foreground sync (`isBackgroundSync == false`) — rendered inline by `OfflineDataSync` on the auth screen |
| `error(isBackground: true)` | Same failure condition but during a background sync (`isBackgroundSync == true`) — picked up by the home screen `ref.listen` to show `SyncFailedModal` |

### Key notifier flags

| Flag | Meaning |
|------|---------|
| `_cancelled` | Set to `true` by `reset()` when the app is paused mid-sync; polled by the queue runner before each batch and during retry waits so the in-flight sync stops cleanly |
| `_isBackgroundSync` | True when `lastSuccessfulSync` is not null (prior sync exists, local data available) or the user tapped "Sync in background" during a foreground sync — controls error surfacing (modal vs inline) |
| `_syncFirstFailureSeen` | Tracks whether the first batch failure has been seen — used with `_isFullSync` to compute `showBackgroundSyncButton` |
| `_isFullSync` | True when `tfrom` is null (no `lastSyncTfrom` — first run or after a data reset). Used in two places: (1) controls the timestamp persistence strategy in `_runBatchedSyncQueueRunner` (when to write `lastSyncTfrom`); (2) passed into `SyncState.error` so the home screen modal shows `fullSyncFailedText` vs `partialSyncFailedText` |

Additional computed properties on `SyncState`:

| Property | Meaning |
|----------|---------|
| `showBackgroundSyncButton` | **Delta sync** (`_isFullSync = false`): true as soon as batches start running. **Full sync** (`_isFullSync = true`): true only after the first retry fails (`_syncFirstFailureSeen`). Always false during background errors (user is already on the home screen) |
| `isBackgroundError` | True for errors that occur during a background sync — triggers the home screen error modal |

---

## Batch Execution

**Files:**
- `lib/src/features/sync/models/sync_batch.dart` — batch enum
- `lib/src/features/sync/services/sync_service.dart` — dispatcher
- `lib/src/features/sync/services/batches/` — individual batch implementations
- `lib/src/features/sync/services/sync_endpoint_jobs/` — per-endpoint fetch logic

Batches run sequentially. Each batch is retried independently before the queue fails.

### Batch 0 — Meta

File: `lib/src/features/sync/services/batches/meta_batch.dart`

Endpoints: `metaLocationTypes`, `metaAttributes`, `overviewSections`  
Always a full fetch — `tfrom` is ignored for meta data.  
Also: updates current user (`skipIfFresh: true` — skipped if `_currentUserFetchedAt` is within the last 2 minutes, hardcoded in `auth_provider.dart`); configures log level from the user response.  
Progress steps: 1 → 3 of 12. The `[0/12]` step ("Updating account...") is only emitted when the user call actually runs — omitted on a cache hit to avoid the counter going backwards on retries.

### Batch 1 — Locations

File: `lib/src/features/sync/services/batches/locations_batch.dart`

Endpoints: `locations`, `localities`  
Also: populates `syncLocationCacheProvider` (in-memory `List<Location>`), deletes removed locations.  
**Delta sync only:** sets `lastSyncTfrom = batchStartTime` on completion. On a full sync this is withheld until all batches complete, so a failure in a later batch doesn't leave `tfrom` set for batches that never ran.  
Progress steps: 4 → 5 of 12

### Batch 2 — Location Content

File: `lib/src/features/sync/services/batches/content_batch.dart`

Endpoints: `locationAttributes`, `locationOverviews`  
Reads from the in-memory location cache (populated in batch 1) to avoid repeated Hive reads.  
Updates the cache before handing off to batch 3.  
Progress steps: 6 → 7 of 12

### Batch 3 — Location Related

File: `lib/src/features/sync/services/batches/related_batch.dart`

Endpoints: `reports`, `discounts`, `hlrs`  
Reads from the in-memory location cache.  
Progress steps: 8 → 10 of 12

### Batch 4 — Members

File: `lib/src/features/sync/services/batches/members_batch.dart`

Endpoints: `boats`, `members`  
Handles boat deletions.  
**Delta sync only:** also fetches notifications (`loadNotifications`) using `tfrom` — skipped on full syncs (`tfrom == null`).  
On completion: sets `lastSuccessfulSync = now()`. **Full sync only:** also sets `lastSyncTfrom = batchStartTime` here (withheld from batch 1 to ensure all batches completed before `tfrom` is committed).  
Progress steps: 11 → 12 of 12 (plus an unnumbered "Checking for notifications..." step on delta syncs)

### Post-batch

After all batches complete:
1. Sync geofences (`_syncGeoAreas`)
2. Clear in-memory location cache
3. Invalidate location providers
4. Set `SyncState.success()`

---

## Queue Runner & Persistence

**Files:**
- `lib/src/features/sync/services/sync_queue_runner.dart` — `SyncQueueRunner`
- `lib/src/features/sync/models/sync_queue_state.dart` — `SyncQueueState`
- `lib/src/helpers/shared_preferences.dart` — `SharedPreferencesHelper`

### Persisted queue state

`SyncQueueState` is written to SharedPreferences as JSON after every batch:

| Field | Type | Description |
|-------|------|-------------|
| `currentBatchIndex` | int | Index of the next batch to run |
| `syncStartedAt` | DateTime | When this queue was created |
| `batchStatuses` | `List<SyncBatchStatus>` | Per-batch success/failure record |

`SyncNotifier.runBatchedSync` checks for a persisted queue before handing off to `SyncQueueRunner`, which simply runs whatever `initialState` it receives:
- **Found, incomplete, not expired:** pass it to `SyncQueueRunner` as `initialState` — runner resumes from `currentBatchIndex`
- **Found, expired** (older than `Config.syncQueueExpiryDays`) **or complete:** discard, create `SyncQueueState.initial()`, run all batches from batch 0 (Meta)
- **Not found:** same — create `SyncQueueState.initial()` and run all batches from batch 0 (Meta)

### SyncBatchStatus transitions

Each batch slot in `batchStatuses` holds a `SyncBatchStatus` enum value:

| Status | Meaning | How it is set |
|--------|---------|---------------|
| `pending` | Not yet attempted | Initial state for all batches in `SyncQueueState.initial()` |
| `complete` | Ran and succeeded | `markBatchComplete(batchIndex)` — also advances `currentBatchIndex` to the next batch |
| `failed` | Ran and failed (will retry) | `markBatchFailed(batchIndex)` — does **not** advance `currentBatchIndex`, so the same batch is retried |

**`isComplete`** — computed property on `SyncQueueState`; true when every slot is `complete`. Used on launch to decide whether to resume or restart: a completed queue has nothing left to do, so it is treated the same as "not found" and a fresh queue is created from batch 0.

**`isExpired`** — computed property on `SyncQueueState`; true when `DateTime.now() - syncStartedAt >= Config.syncQueueExpiryDays`. Used on launch to guard against resuming a very old queue: if the app was closed for several days, the server data will have moved on and resuming mid-queue could leave the device with a partially-stale dataset. An expired queue is discarded and restarted from batch 0 regardless of how many batches had already completed.

### SharedPreferences sync keys

| Key | Value |
|-----|-------|
| `lastSuccessfulSync` | Timestamp of last fully-completed sync |
| `lastSyncTfrom` | Start time of last successful locations batch (delta marker sent as `?tfrom=` to the API) |
| `syncQueueState` | JSON-serialised `SyncQueueState` |
| `refreshFrequency` | User's chosen sync interval |

### Delta sync timestamp strategy

`lastSyncTfrom` is set to the **batch start time** (`SyncQueueState.syncStartedAt`), not the completion time. This is intentional: entities created on the server between when the locations batch fetched data and when later batches ran will still be picked up on the next delta sync, because the next sync's `?tfrom=` will pre-date those entities.

**When `lastSyncTfrom` is set:**

| Condition | When set | Value |
|-----------|----------|-------|
| Delta sync (`lastSyncTfrom` was non-null) | After locations batch (batch 1) completes | `batchStartTime` |
| Full sync (`lastSyncTfrom` was null) | After the final batch (Members) completes | `batchStartTime` |
| Any batch fails | Never — not written on failure | — |

The full sync case deliberately withholds `lastSyncTfrom` until all batches complete. If it were written after the locations batch (as in a delta sync), a failure in a later batch would leave `lastSyncTfrom` set — meaning the next sync would be a delta, skipping entity types (e.g. Members) that never completed. By waiting until the final batch, a failed full sync always retries as a full sync.

---

## Retry & Error Handling

**File:** `lib/src/features/sync/models/sync_error_category.dart`

### Error categories

| Category | Cause | Behaviour |
|----------|-------|-----------|
| `offline` | No connection | Retry with delay |
| `timeout` | Connection too slow | Retry with delay |
| `serverError` | 5xx response | Retry with delay |
| `authError` | 401/403 or `InvalidTokenException` from a batch | Fail immediately — sets `SyncState.error`, no automatic redirect. Surfaces inline (foreground) or via `SyncFailedModal` (background) like any other error |
| `badRequest` | 4xx response | Fail immediately |
| `invalidData` | Parse/type error | Fail immediately |
| `unknown` | Everything else | Retry with delay |

### Retry flow

```
Batch throws exception
  ↓
SyncErrorCategory.fromError() classifies it
  ↓
shouldFailImmediately?
  YES → Sentry.captureException → rethrow → SyncState.error() → queue stops
  NO  → consecutiveFailures++
         if < maxRetries:
           SyncState.loading(retryAt: now + retryDelay)
           Sentry.addBreadcrumb (retry attempt logged, not a full exception)
           wait retryDelay seconds (polling _cancelled)
           retry same batch
         if ≥ maxRetries:
           Sentry.captureException → rethrow → SyncState.error() → queue stops
```

**Retry delay:** `Config.retryDelay` (30s) in production; `Config.retryDelayDebugMode` (10s) in debug mode — selected via `kDebugMode` in `SyncQueueRunner`.  
**Max retries:** `Config.syncMaxRetries`.

**Note:** To force a retry during development set `const _debugForceSyncFailure = false;` to `true` in `sync_service.dart`.

### UI feedback during retries

- A countdown timer shows `retryAt` to the user.
- `showBackgroundSyncButton` is computed as `!_isFullSync || _syncFirstFailureSeen`:
  - **Delta sync** (`_isFullSync = false`, e.g. manual sync from settings): button appears immediately when batches start — `!false` is always true.
  - **Full sync** (`_isFullSync = true`, e.g. first launch, clear-and-redownload): button is hidden until the first retry also fails — `!true || false` = false on the first failure; `!true || true` = true from the second failure onward.
  - The `SyncLocationsDialog` widget simply checks `syncState.showBackgroundSyncButton` — all logic is internal to the notifier.

---

## Settings Integration

**File:** `lib/src/screens/settings/settings_screen.dart`

### Refresh frequency

The `refreshFrequency` preference controls automatic sync scheduling:

| Value | Behaviour |
|-------|-----------|
| `auto` | `toDuration()` returns `Duration.zero` — `nextRequiredRefresh == lastSync`, which is always in the past, so sync triggers on every startup. On resume (non-startup) a 30-minute buffer is added to `nextRequiredRefresh` specifically for `auto`, so it only syncs if 30+ minutes have passed since the last sync (preventing an immediate re-sync if the user just backgrounded and foregrounded the app) |
| `twoHours` | Sync if last sync > 2h ago |
| `sixHours` | Sync if last sync > 6h ago |
| `twelveHours` | Sync if last sync > 12h ago |
| `oneDay` | Sync if last sync > 24h ago |
| `twoDays` | Sync if last sync > 48h ago |
| `manual` | `toDuration()` returns `null` — hits the early-exit in `checkForSync`, never auto-syncs. Only syncs via the manual trigger in settings |

For `auto` frequency on resume (non-startup), a 30-minute buffer is added to `nextRequiredRefresh` so the app doesn't re-sync immediately if the user backgrounds and foregrounds within 30 minutes of the last sync. This buffer does not apply to other frequencies — their intervals are already long enough to be self-governing.

### Manual sync

The "Update your App with the latest data now" button calls `_showSyncLocationsDialog()`, which opens `SyncLocationsDialog` and then calls `runSync(force: true, startAsBackground: false)`. `force: true` bypasses the frequency check. `startAsBackground: false` explicitly forces foreground mode — without this, `_isBackgroundSync` would be auto-derived as `true` for returning users (since `lastSuccessfulSync != null`), which would cause `SyncLocationsDialog` to never dismiss itself on error (see tech debt in `settings_screen.md`). On error, `SyncLocationsDialog` reads `syncState.isFullSync` directly to pick the correct failure copy, so no `fullSync` flag needs to be threaded through the call.

### Full data reset

The "Clear and redownload all locations" button is disabled while a sync is in progress (`isDisabled: syncState.isLoading`). A defensive guard in `_showClearLocationsDialog` also checks this and redirects to `SyncLocationsDialog` if somehow reached mid-sync. When tapped while idle, the sequence is:

1. Shows a confirmation dialog ("Are you sure you want to clear all your offline data?") — user must confirm to proceed
2. Resets `lastSuccessfulSync`, `lastSyncTfrom`, and `syncQueueState` to null
3. Calls `repositoryManager.deleteSyncData()` (deletes all Hive entity data)
4. Calls `_showSyncLocationsDialog()` — the same shared helper used by manual sync, which opens `SyncLocationsDialog` and then calls `runSync(force: true)`

The pre-steps are the only thing that makes it distinct from a manual sync. By clearing `lastSuccessfulSync` and `lastSyncTfrom` they guarantee the next sync is a full fetch (`_isFullSync = true`), so the "Sync in background" button in `SyncLocationsDialog` only appears after the first retry fails — matching first-launch behaviour. For a regular manual sync (`lastSyncTfrom` is set, delta sync), the button appears immediately when batches start.

### Staging API (admins only)

Toggling the staging API switch logs the user out, clears all local data, and navigates back to the auth screen — triggering a fresh full sync against the staging environment. All subsequent sync API calls go to the staging base URL until toggled back.

---

## App Lifecycle Integration

**Files:**
- `lib/src/screens/authentication_screen.dart`
- `lib/src/screens/home_screen.dart`

### Authentication screen — startup sequence

```
1. Is refresh token expired?
     YES → show login button (stop here)
     NO  → continue

2. Can we reach the API?
     NO  → connection check took < 1s: show offline modal on auth screen
             user taps "Open in Offline Mode" → navigate to home screen using locally cached data (no sync runs)
             (if no refresh token: show login button instead)
           connection check took > 1s (slow/timeout): automatically navigate to home screen using locally cached data (no sync runs)
     YES → continue

3. Is access token still valid?
     NO  → refresh token
     YES → continue

4. await runSync(force=true)   ← noSuccessfulSync: background mode derived as false, user waits
   runSync(force=true)         ← at least one successful sync exists: background mode derived as true, navigates home immediately
```

The user sees the sync progress on the auth screen whenever `noSuccessfulSync` is true (`lastSuccessfulSync == null`) — covering the first-ever run, any subsequent launch where every previous sync failed or was cancelled, and any launch after a settings "clear and redownload" reset. Once at least one sync has completed, `runSync` is called fire-and-forget (no `await`) and navigation to the home screen happens immediately. The sync continues executing on the same event loop — the home screen observes it via `ref.listen` on `syncNotifierProvider`.

### Home screen — pause and resume

The home screen pause and resume handlers are both protected by an `_isHandlingLifecycle` re-entry guard and ignore the `inactive` state, so that permission dialogs (which cause `inactive` → `resumed` transitions) don't trigger spurious sync checks.

### Home screen — pause

Triggered on `paused` or `detached` (treated identically):
1. If sync is currently in progress, skip — resetting mid-sync would clear `isProcessing`, allowing the resume handler to immediately start a new sync that races the still-running one
2. Call `syncNotifier.reset()`: sets `_cancelled = true` and state to `idle`. The `_cancelled` flag is what actually stops the queue runner — it polls the flag before each batch and during retry waits, so the in-flight sync stops cleanly at the next opportunity

### Home screen — resume

Triggered when the app returns to the foreground:
1. If sync is currently in progress, skip entirely — this guards against a background sync (started after the user tapped "Sync in background") being interrupted by a spurious `resumed` event from a permission dialog, which would otherwise `pushReplacementNamed` the user back to the auth screen mid-sync
2. Call `checkForSync(isStartUp: false)` — if a sync is due (see [checkForSync](#checkforsync)), `pushReplacementNamed` to the auth screen, which re-runs the full auth → sync sequence

### Background sync error handling

`home_screen.dart` listens to `syncNotifierProvider`. When `isBackgroundError` transitions from false to true, it shows a `SyncFailedModal` dialog. The user can dismiss it and continue using local data. 

### Sync in-progress indicators

Three UI elements indicate that a background sync is in progress (`syncNotifierProvider.isLoading`):

| Location | Widget | Tappable | Behaviour on tap |
|----------|--------|----------|-----------------|
| Home screen — map | Floating action button with `CircularProgressIndicator` | Yes | Opens `SyncLocationsDialog` so the user can monitor progress or move it to background |
| Home screen — location sheet | Small `CircularProgressIndicator` (8×8px) next to "Locations" label | No | Purely informational |
| Settings screen — "Update your App" row | Spinner within `TcaSettingItem` (via `isLoading`) | Yes | Opens `SyncLocationsDialog` (partial sync mode) |

**Settings "Clear and redownload" guard:** if the user taps "Clear and redownload" while a sync is already in progress, `_showClearLocationsDialog` detects `isLoading` and opens `SyncLocationsDialog` instead of the clear confirmation dialog, preventing a concurrent reset.

**Background Sync Success:** On background sync success, a SnackBar confirms the data is up to date.

---

## Sync Trigger Points

| Trigger | Mode | Notes |
|---------|------|-------|
| App startup — `noSuccessfulSync` | Foreground | User waits on auth screen (`runSync` awaited) until either sync completes successfully or sync fails; repeats on every launch until a sync completes and `lastSuccessfulSync` is set in SharedPreferences |
| App startup — at least one sync completed and `lastSuccessfulSync` is not null | Background | User sent to home screen immediately |
| App resume | Background | Home screen calls `checkForSync(isStartUp: false)` — see below |
| Settings: "Update your App" button | Foreground | `force: true` — bypasses `checkForSync` entirely |
| Settings: "Clear and redownload" | Foreground | Full sync after data reset; `force: true` |
| Successful login | Background | After token acquired and user fetched |

### checkForSync

`checkForSync` is called from the home screen resume handler to decide whether to redirect to the auth screen. It is also called inside `runBatchedSync` when `force: false`, though in practice all current callers pass `force: true` which bypasses it. The same underlying conditions govern both startup and resume: whether there is unfinished work or whether enough time has elapsed since the last successful sync.

Returns `true` (sync required) when any of the following hold:

| Condition | Detail |
|-----------|--------|
| Incomplete, non-expired queue | A `SyncQueueState` exists in SharedPreferences that is neither complete nor expired — a previous sync was interrupted and can be resumed from the batch it left off at |
| No `lastSuccessfulSync` | No sync has ever completed (first run, or after a clear-and-redownload that cleared the timestamp) |
| Frequency interval elapsed | `now > lastSuccessfulSync + interval`. For `auto` on a non-startup check, a 30-minute buffer (5 min via kDebugMode) is added to prevent an immediate re-sync if the user briefly backgrounds the app |

Returns `false` (no sync needed) when:
- Frequency is `manual` — early exit, frequency interval is never evaluated
- Frequency interval has not yet elapsed (after applying the `auto` buffer if applicable)

---

## Full State Flow Diagram

```
AUTH SCREEN
│
├─ refresh token expired (or checkAuthStatus: false passed explicitly)? → YES → show login button [END]
│
├─ API reachable? → NO → fast failure (<1s): offline modal shown on auth screen
│                           └─ user taps "Open in Offline Mode" → home screen on cached data, no sync [END]
│                              (no refresh token → show login button [END])
│                         slow failure (>1s): auto-navigate to home screen on cached data, no sync [END]
│
├─ access token valid?
│   YES → _handleApiCall(_syncDataAndDownloadMaps())
│   NO  → _handleApiCall(refreshToken(), callback: _syncDataAndDownloadMaps)
│             └─ InvalidTokenException (account revoked) → show login button [END]
│                other exception → show offline modal [END]
│
├─ noSuccessfulSync (lastSuccessfulSync == null)?
│   YES → await runSync(force: true)   ← user waits on auth screen
│   NO  → runSync(force: true)         ← fire-and-forget; _handleApiCall immediately calls handleHomeScreenRoute()
│
└─ runBatchedSync
   │
   ├─ _isBackgroundSync derived from lastSuccessfulSync (true if not null)
   │
   ├─ persisted SyncQueueState?
   │   ├─ incomplete + not expired → resume from currentBatchIndex
   │   └─ null / expired / complete → SyncQueueState.initial() (run all batches)
   │
   ├─ [non-startup only] connectivity check → SyncState.loading(isPreSync: true)
   │
   └─ _runBatchedSyncQueueRunner
      │
      ├─ FOR each batch (currentBatchIndex → 4):
      │   │
      │   ├─ batch.run() → emit progress steps
      │   │   │
      │   │   ├─ SUCCESS → markBatchComplete, persist queue, advance index
      │   │   │
      │   │   ├─ FAILURE → classify SyncErrorCategory
      │   │   │   ├─ shouldFailImmediately (auth/bad/invalid)
      │   │   │   │   └─ rethrow → SyncState.error() [END]
      │   │   │   └─ retriable (offline/timeout/server/unknown)
      │   │   │       ├─ < maxRetries → SyncState.loading(retryAt) → wait → retry
      │   │   │       └─ ≥ maxRetries → rethrow → SyncState.error() [END]
      │   │   │
      │   │   └─ _cancelled=true → cache cleared, returns early [END]
      │   │
      │   └─ [repeat for next batch]
      │
      ├─ syncGeoAreas
      ├─ clear in-memory location cache
      ├─ invalidate location providers
      └─ SyncState.success()
         │
         ├─ noSuccessfulSync was true  → user was waiting on AUTH SCREEN (OfflineDataSync inline widget)
         │   ├─ success → OfflineDataSync.onComplete() navigates home; _handleApiCall's handleHomeScreenRoute() is a no-op (guard fires)
         │   ├─ retrying → countdown shown; after first retry fails, "Sync in background" button appears (full sync)
         │   │             user taps → continueInBackground(), navigate home; further errors via SyncFailedModal
         │   └─ terminal error → runSync returns (catches internally); _handleApiCall checks isError → returns early
         │                       OfflineDataSync renders inline error + "Continue to app" button
         │                       user taps → continueInBackground(), handleHomeScreenRoute() → navigates home
         └─ noSuccessfulSync was false → user already on HOME SCREEN (handleHomeScreenRoute() called immediately)
                                         success → SnackBar
                                         error → SyncFailedModal (home screen ref.listen)

HOME SCREEN
│
├─ listen(syncNotifierProvider)
│   ├─ isBackgroundError → show SyncFailedModal
│   └─ isSuccess → success SnackBar
│
└─ App lifecycle
   ├─ pause
   │   ├─ isProcessing? → skip (prevents race with still-running sync)
   │   └─ syncNotifier.reset() → _cancelled=true, state=idle
   │
   └─ resume
       ├─ isProcessing? → skip (guards against spurious resumed from permission dialog)
       └─ checkForSync(isStartUp: false)
           ├─ incomplete non-expired queue → true
           ├─ lastSuccessfulSync==null    → true
           ├─ frequency elapsed (auto: +30min buffer on resume) → true
           ├─ manual frequency           → false
           └─ true → pushReplacementNamed to AUTH SCREEN

SETTINGS SCREEN
├─ "Update your App..." → _showSyncLocationsDialog()
│   └─ opens SyncLocationsDialog → runSync(force: true, startAsBackground: false)
└─ "Clear and redownload..." → confirmation dialog
    └─ confirmed → reset SharedPrefs + delete Hive data
        └─ _showSyncLocationsDialog()
            └─ opens SyncLocationsDialog → runSync(force: true, startAsBackground: false)
```

---

## File Reference

| File | Purpose |
|------|---------|
| `lib/src/features/sync/providers/sync_provider.dart` | `SyncNotifier` — orchestrates the full sync lifecycle |
| `lib/src/features/sync/providers/sync_state.dart` | `SyncState` Freezed union |
| `lib/src/features/sync/services/sync_queue_runner.dart` | Batch executor with retry logic |
| `lib/src/features/sync/services/sync_service.dart` | Batch dispatcher |
| `lib/src/features/sync/models/sync_batch.dart` | `SyncBatch` enum |
| `lib/src/features/sync/models/sync_error_category.dart` | Error classification |
| `lib/src/features/sync/models/sync_queue_state.dart` | Persisted queue state |
| `lib/src/features/sync/services/batches/meta_batch.dart` | Batch 0: types, attributes, sections |
| `lib/src/features/sync/services/batches/locations_batch.dart` | Batch 1: locations + cache |
| `lib/src/features/sync/services/batches/content_batch.dart` | Batch 2: attributes, overviews |
| `lib/src/features/sync/services/batches/related_batch.dart` | Batch 3: reports, discounts, HLRs |
| `lib/src/features/sync/services/batches/members_batch.dart` | Batch 4: boats, members |
| `lib/src/features/sync/services/sync_endpoint_jobs/` | Per-endpoint fetch + merge logic |
| `lib/src/features/sync/providers/sync_location_cache_provider.dart` | In-memory location list shared across batches |
| `lib/src/features/auth/providers/auth_provider.dart` | `Authentication` — OAuth, token refresh, user |
| `lib/src/features/auth/domain/auth_state.dart` | `AuthState` Freezed union |
| `lib/src/helpers/auth.dart` | `AppAuth` secure storage read/write helpers |
| `lib/core/api/interceptors/auth_interceptor.dart` | DIO interceptor — token injection + 401 refresh |
| `lib/src/data/providers/app_initialisation_provider.dart` | Startup initialisation sequence |
| `lib/src/data/providers/friend_setting.dart` | Friend/sharing settings state notifier |
| `lib/src/helpers/shared_preferences.dart` | Provider wrapper for `SharedPreferencesHelper` |
| `lib/src/screens/authentication_screen.dart` | Auth screen — startup sync sequence |
| `lib/src/screens/home_screen.dart` | Home screen — lifecycle + background sync monitoring |
| `lib/src/screens/settings/settings_screen.dart` | Settings — manual sync triggers + frequency |
| `lib/src/features/sync/widgets/offline_data_sync.dart` | Auth screen sync progress UI |
| `lib/src/features/sync/widgets/sync_locations_dialog.dart` | Settings manual sync modal |