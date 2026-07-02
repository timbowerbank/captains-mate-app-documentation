---
id: 709568b8-b9d8-41e5-8b2f-e2243e235051
blueprint: captains_mate_app_43
title: Authentication
use_synced_content: false
parent: a1b7cec3-99c2-4a5d-9c8b-9790b8204982
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1781091338
---
# Authentication Overview

This document describes how authentication and user permissions work in the TCA Mobile App.

## Authentication Flow

The app uses **OAuth 2.0 with Authorization Code + PKCE** flow via the `flutter_appauth` package and the CA SSO server.

### Login Process

1. **Initiate OAuth** - User taps LOGIN — the app opens a browser (Chrome Custom Tab on Android, Safari on iOS) with the OAuth login dialog.
2. **User authenticates with the SSO provider** (enters credentials, or auto-completes if a browser session is cached)
3. SSO server redirects back to the app with a short-lived **authorization code**
4. **App exchanges authorization code** for access and refresh tokens

| Token | Lifetime | Purpose |
|-------|----------|---------|
| Access token | Short (exact lifetime unconfirmed) | Proves the user is authenticated on every API request |
| Refresh token | Longer (exact lifetime unconfirmed, app assumes 28 days) | Used to obtain new access tokens without re-authenticating |

5. **Persist credentials** - Tokens saved to secure storage with expiration dates
6. **Fetch user profile** from backend API; if user ID has changed (account switch), local repos are cleared and re-initialised
7. **Register FCM token** - Device FCM token sent to the API token endpoint if the user has notifications enabled
8. **Validate scopes** - App checks if required OAuth scopes were granted

**Key file:** `lib/src/features/auth/providers/auth_provider.dart`

### Token Management

| Token | Purpose | Storage |
|-------|---------|---------|
| Access Token | Bearer token for API requests | Secure storage |
| Refresh Token | Obtain new access tokens | Secure storage |
| Access Token Expiration | Track whether to refresh (checked once, at startup in `_attemptLogin`) | Secure storage |
| Refresh Token Expiration | Track if re-login needed (currently this is only used to determine the number of days before log-in needed for the offline mode warning)  | Secure storage |

**Automatic refresh:** If a request returns a 401 or 403, the interceptor attempts to refresh the access token, then retries the original request. This is handled reactively in lib/core/api/interceptors/refresh_interceptor.dart.

### Making API Requests

**API authentication:** Every request includes `Authorization: Bearer {token}` header via `lib/core/api/interceptors/auth_interceptor.dart`.

The server validates this on every request.

### When the Access Token Expires

When an access token expires the server returns a 401 or 403 in the API response. The `refresh_interceptor` detects this and:

1. Reads the refresh token from secure storage
2. Sends it directly to SSO server token endpoint (HTTP only — no browser involved)
3. If the token is valid, server returns a **new access token and a new refresh token**
4. The old refresh token is immediately invalidated (token rotation)
5. New tokens are saved to secure storage
6. The original failed request is retried with the new access token

### When the Refresh Token is Invalid or Expired

If the refresh token itself is invalid or expired, the server returns `invalid_grant`. The app cannot recover silently — the user must go through the full SSO flow again. `_redirectIfRetryFails()` pushes to the auth screen, the browser opens, and if the SSO browser session is still active it auto-completes without prompting for credentials.

### Key Properties

- **Access tokens are stateless** — validated by cryptographic signature, no server-side lookup
- **Refresh tokens are stateful** — SSO server tracks them server-side, allowing revocation, expiry, and rotation
- **Token rotation means each refresh token can only be used once** — two simultaneous uses of the same refresh token will always result in one receiving `invalid_grant`

### Auth State

The `AuthState` model (`lib/src/features/auth/domain/auth_state.dart`) tracks:

```dart
AuthState {
  accessToken: String?
  refreshToken: String?
  accessTokenExpiration: DateTime?
  refreshTokenExpiration: DateTime?
  currentUser: CurrentUser?
  scopes: List<String>?
}
```

**Computed properties:**

`isAuthenticated` - Returns `true` if both `accessToken` and `currentUser` exist

## User Permissions & Roles

### Role-Based Access Control

Roles are stored as strings in the `currentUser.roles` list, populated from the backend API response.

**Role checks** in `lib/src/features/user/models/current_user.dart`:

```dart
bool get isAdmin => roles.contains('admin');
bool get isDemonstrator => roles.contains('demonstrator');
```

### Role-Gated Features

Only two features in the app are gated by roles, both in `lib/src/screens/settings/settings_screen.dart`:

| Role | Feature | Purpose |
|------|---------|---------|
| `isDemonstrator` | "Boat Show Mode" toggle | Obscures sensitive/personal info during demos |
| `isAdmin` | "Use Staging API" toggle | Switch between staging and production API endpoints |

---

## OAuth Scopes

### Configuration

OAuth scopes are configured via build-time environment variables (`--dart-define`):

```dart
// lib/src/helpers/config.dart
static const oauthScopes = String.fromEnvironment('DEFINE_OAUTH_SCOPES');
```

**Configuration sources:**
- Local development: `.vscode/launch.json` (from `launch.json.example`)
- Debug or profile builds: `dart-defines.dev.json`
- Release builds: `dart-defines.prod.json`

### Configured Scopes

All environments (`dart-defines.prod.json`, `dart-defines.dev.json`, `launch.json`) currently configure a single scope:

```
openid
```

| Scope | Purpose |
|-------|---------|
| `openid` | Standard OIDC scope for user identity |

### Scope Validation

After login, the app validates that all requested scopes were granted:

```dart
// lib/src/features/auth/providers/auth_provider.dart
void _validateScopes(List<String> requiredScopes, List<String>? grantedScopes) {
  for (final scope in requiredScopes) {
    if (!(grantedScopes ?? []).contains(scope)) {
      throw MissingScopeException(requiredScopes, grantedScopes ?? []);
    }
  }
}
```

If scopes are missing, a `MissingScopeException` is thrown and the user sees a warning dialog:
> "Some permissions may not have been granted"

### Scopes vs Roles

**Important distinction:**
- **OAuth Scopes** - Used for backend API authorization only. The app validates they were granted but does not use them for in-app feature gating.
- **Roles** - Used for in-app feature gating (`isAdmin`, `isDemonstrator`). These come from the user profile, not OAuth.

---

## Key Files Reference

| Purpose | File |
|---------|------|
| Auth provider & logic | `lib/src/features/auth/providers/auth_provider.dart` |
| Auth state model | `lib/src/features/auth/domain/auth_state.dart` |
| User model with roles | `lib/src/features/user/models/current_user.dart` |
| OAuth config | `lib/src/helpers/config.dart` |
| OAuth helper utilities | `lib/src/helpers/auth.dart` |
| Secure token storage | `lib/src/helpers/secure_storage.dart` |
| Token refresh interceptor | `lib/core/api/interceptors/refresh_interceptor.dart` |
| Auth header interceptor | `lib/core/api/interceptors/auth_interceptor.dart` |
| Login UI & flow | `lib/src/screens/authentication_screen.dart` |
| Role-based settings | `lib/src/screens/settings/settings_screen.dart` |
| Missing scope exception | `lib/src/data/exceptions/MissingScopeException.dart` |

---

## Exception Types

| Exception | Trigger | Handling |
|-----------|---------|----------|
| `MissingScopeException` | User didn't grant required OAuth scopes | Warning dialog, login continues |
| `InvalidTokenException` | Token is invalid or revoked | Loading stops and the login button is shown — the user must manually tap it to initiate the OAuth flow; login is not triggered automatically |


## Known Caveats / Tech-Debt Notes

1. **Logout doesn't work**: Cookies are cached in the device browser.

2. **Refresh interceptor bypasses Riverpod auth state**: `refresh_interceptor.dart` reads the refresh token directly from `FlutterSecureStorage` and writes new tokens via `AppAuth.saveAuth()` — it never touches `authenticationProvider`. After an interceptor-triggered refresh, `ref.read(authenticationProvider).accessToken` is stale until the app reinitialises. The auth provider acknowledges this: `updateCurrentUser` deliberately avoids calling `_save()` to prevent overwriting the interceptor's freshly-saved tokens with the stale state values.

3. **FCM token registration on login is fire-and-forget**: `unawaited(FcmService().sendToken())` in `_handleUserAuthentication()`. If it fails, the server never receives the device token and push notifications break silently for that session. The failure is logged to Sentry but not surfaced to the user and not retried.

4. **`refreshTokenExpiration` calculated differently across code paths**: The refresh interceptor subtracts an extra day (`.subtract(Duration(days: 1))`) when saving the refresh token expiration; the auth provider's own login and refresh paths do not. The displayed "next login due" time can differ by a day depending on which code path last refreshed the token.

5. **Refresh failure redirect may be too aggressive**: `_redirectIfRetryFails()` in `refresh_interceptor.dart` fires for any refresh failure, including transient network errors. It should only redirect to auth for definitive server rejections (`invalid_grant`). Currently any momentary network drop during a refresh attempt will push the user to the auth screen unnecessarily.

6. **Token expiry checks are dead code**: `AuthState` has two getters — `needsTokenRefresh` and `isRefreshTokenExpired` — coded by CoreBlue but never wired up. Completing this work would allow the app to detect an expired refresh token before firing a network refresh call, and redirect to auth cleanly without an unnecessary round-trip to SSO server.

7. **Concurrent refresh race condition**: If multiple API requests fire simultaneously with an expired access token, each independently triggers `_refreshAndRetry`. With SSO server's refresh token rotation, the first successful refresh invalidates the token — any concurrent refresh attempt using the same token will get `invalid_grant`. There is no serialisation in place to prevent this.

8. **Refresh token failure in headless mode not handled**: If the refresh attempt fails due to the race condition mentioned above or the refresh token has expired, the app is not in the foreground so it is unable to log the user in. So in headless mode the app will keep attempting refreshes indefinitely — every time a location update fires, it'll hit 401, try to refresh, get invalid_grant, fail silently, and repeat. There's no mechanism to give up cleanly in headless based on token expiry. It just keeps hitting the SSO server with invalid refresh token attempts until the user opens the app. If the access token is expired and can't be refreshed, the background task should be abandoned entirely - background tasks are best-effort by nature.