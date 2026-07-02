---
id: 982a84e4-4dc3-435b-b7fe-2205e48311b4
blueprint: captains_mate_app_43
title: Setup
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780582148
---
# Setup

### 1. Install the pinned Flutter version

```bash
fvm install
```

This reads the version from `.fvmrc` and installs it.

### 2. Install dependencies

```bash
fvm flutter pub get
```
Ensure you use 'fvm' in the command to use the pinned version of Flutter

### 3. Run code generation

Generated files (`lib/generated/`, `.g.dart`, `.freezed.dart`) are **gitignored** and must be produced locally before the app will build.

```bash
# One-off generation
dart run build_runner build --delete-conflicting-outputs

# Watch mode (recommended during development)
dart run build_runner watch --delete-conflicting-outputs
```

### 4. Configure VS Code launch and dart-defines files

Copy the example launch file:

```bash
cp .vscode/launch.json.example .vscode/launch.json
```

The launch configurations load defines from JSON files rather than inline args. Both files are gitignored and must be created locally:

| File | Used by |
|------|---------|
| `dart-defines.dev.json` | **TCA debug** configuration |
| `dart-defines.prod.json` | **TCA release** configuration |

Each file is a flat JSON object containing the following keys:

| Key | Description |
|-----|-------------|
| `DEFINE_OAUTH_CLIENT_ID` | OAuth client ID |
| `DEFINE_OAUTH_SCOPES` | OAuth scopes |
| `DEFINE_OAUTH_AUTH_ENDPOINT` | OIDC auth endpoint |
| `DEFINE_OAUTH_TOKEN_ENDPOINT` | OIDC token endpoint |
| `DEFINE_OAUTH_USER_ENDPOINT` | OIDC user-info endpoint |
| `DEFINE_API_BASE_URL` | Production API base URL |
| `DEFINE_STAGING_API_BASE_URL` | Staging API base URL |
| `DEFINE_SENTRY_DSN` | Sentry DSN (leave blank for local dev) |
| `DEFINE_FIREBASE_IOS_API_KEY` | Firebase iOS API key |
| `DEFINE_FIREBASE_SENDER_ID` | Firebase sender ID |
| `DEFINE_FIREBASE_PROJECT_ID` | Firebase project ID |
| `DEFINE_FIREBASE_BUCKET_NAME` | Firebase storage bucket |
| `DEFINE_FIREBASE_IOS_BUNDLE_ID` | Firebase iOS bundle ID |
| `DEFINE_FIREBASE_IOS_ANDROID_ID` | Firebase Android app ID |
| `DEFINE_FIREBASE_IOS_APP_ID` | Firebase iOS app ID |
| `DEFINE_FIREBASE_ANDROID_API_KEY` | Firebase Android API key |

The `BACKGROUND_LOCATION_API_KEY` is set in the `env` block of `launch.json` directly (not in the dart-defines files) and isn't needed for local development.

### 5. Configure Firebase

The following come from the GoogleServices.json (Android) and GoogleService-Info.plist (iOS)

| dart-defines key | `firebase_options.dart` field |
|------------------|-------------------------------|
| `DEFINE_FIREBASE_ANDROID_API_KEY` | `android.apiKey` |
| `DEFINE_FIREBASE_IOS_ANDROID_ID` | `android.appId` |
| `DEFINE_FIREBASE_SENDER_ID` | `android.messagingSenderId` |
| `DEFINE_FIREBASE_PROJECT_ID` | `android.projectId` |
| `DEFINE_FIREBASE_BUCKET_NAME` | `android.storageBucket` |
| `DEFINE_FIREBASE_IOS_BUNDLE_ID` | `ios.iosBundleId` |
| `DEFINE_FIREBASE_IOS_APP_ID` | `ios.appId` |
| `DEFINE_FIREBASE_IOS_API_KEY` | `ios.apiKey` |

### 6. Run the app

Because of the environment variables needed on launch, the simplest approach is to use VSCode's Run and Debug.

Alternatively use:

```

export BACKGROUND_LOCATION_API_KEY=""
export APPLE_APP_IDENTIFIER="uk.org.cruising.captainsmate"
export ANDROID_APP_IDENTIFIER="com.cruisingassociation.captainsmate"

fvm flutter run \
  --debug \
  --flavor beta \
  --dart-define-from-file=dart-defines.dev.json 
  
```