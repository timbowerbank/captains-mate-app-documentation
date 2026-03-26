---
id: b0d7a655-7343-401f-aee5-6b975bca828f
blueprint: captains_mate_app_43
title: 'Authentication and Permissions'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774535955
---
# Authentication and Permissions Overview

This document describes how authentication and user permissions work in the TCA Mobile App.

## Authentication Flow

The app uses **OAuth 2.0 with Authorization Code + PKCE** flow via the `flutter_appauth` package.

### Login Process

1. **Initiate OAuth** - User taps LOGIN, system OAuth dialog appears
2. **Exchange authorization code** for access and refresh tokens
3. **Fetch user profile** from backend API
4. **Validate scopes** - Ensures all required OAuth scopes were granted
5. **Persist credentials** - Tokens saved to secure storage

**Key file:** `lib/src/features/auth/providers/auth_provider.dart`

### Token Management

| Token | Purpose | Storage |
|-------|---------|---------|
| Access Token | Bearer token for API requests | Secure storage |
| Refresh Token | Obtain new access tokens | Secure storage |
| Access Token Expiration | Track when to refresh | Secure storage |
| Refresh Token Expiration | Track when re-login needed | Secure storage |

**Automatic refresh:** Tokens are refreshed when the access token expires within 5 minutes, handled by `lib/core/api/interceptors/refresh_interceptor.dart`.

**API authentication:** Every request includes `Authorization: Bearer {token}` header via `lib/core/api/interceptors/auth_interceptor.dart`.

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
- `isAuthenticated` - Returns `true` if both `accessToken` and `currentUser` exist
- `needsTokenRefresh` - Returns `true` if access token expires within 5 minutes
- `isRefreshTokenExpired` - Returns `true` if refresh token is past expiration

---

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

**Code example:**

```dart
// Demonstrator-only section (line 355)
if (currentUser.isDemonstrator) ...[
  TcaSettingHeader('Admin'),
  TcaSettingSwitch(
    value: _boatShowMode,
    title: 'Boat Show Mode',
    subtitle: 'Obscures any potentially personal or sensitive information',
    // ...
  ),
],

// Admin-only toggle (line 468)
if (currentUser.isAdmin)
  TcaSettingSwitch(
    value: _isUsingStaging,
    title: 'Use Staging API',
    // ...
  ),
```

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
- CI/CD: GitHub Actions variables (`.github/workflows/actions.yml`)
- Release builds: Fastlane (`android/fastlane/Fastfile`, `ios/fastlane/Fastfile`)

### Example Scopes

From the example launch config:

```
openid ca_ci_edit_sub ca_owndetails_basic ca_member_functionality
```

| Scope | Purpose |
|-------|---------|
| `openid` | Standard OIDC scope for user identity |
| `ca_ci_edit_sub` | Edit subscription/membership data |
| `ca_owndetails_basic` | Access to user's own basic details |
| `ca_member_functionality` | Access to member features |

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
| `InvalidTokenException` | Token is invalid or revoked | Forces re-login |
| `SkipSyncException` | Sync can be skipped (offline mode) | Proceeds to home screen |