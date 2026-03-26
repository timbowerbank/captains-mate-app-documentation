---
id: 07e549a9-67a1-488c-a4fe-67a1c9876093
blueprint: captains_mate_app_43
title: 'Android Build Deployment'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774536252
---
# Android Build, Signing, and Deployment Documentation

## Overview

This Flutter project uses **Fastlane** for Android builds and deployment, **Gradle** for build configuration and signing, and **GitHub Actions** for CI/CD automation. The signing keystore and Google Play API credentials are stored in an **AWS S3 bucket**, while sensitive credentials are managed through **GitHub Secrets** and **Variables**.

---

## Architecture Components

### 1. Fastlane Configuration
Located in `/android/fastlane/`

### 2. Gradle Build System
- Root: `/android/build.gradle`
- App module: `/android/app/build.gradle`
- Properties: `/android/gradle.properties`

### 3. GitHub Actions Workflow
Defined in `/.github/workflows/actions.yml`

### 4. Storage Locations
- **S3 Bucket**: `tca-secure-assets` (keystore and API keys)
- **GitHub**: Secrets and Variables for environment configuration

---

## Build Process Flow

### GitHub Actions Android Job (`actions.yml:10-86`)

```yaml
Job: android
Runs on: ubuntu-latest
Environment: staging
```

#### Step-by-Step Process:

1. **Checkout Code** - Checks out the repository
2. **Setup Java** - Installs Java 17 (Temurin distribution)
3. **Setup Flutter** - Installs Flutter 3.32.6 (stable channel)
4. **Install Flutter Dependencies** - Runs `flutter pub get`
5. **Setup Android SDK** - Configures Android SDK tools
6. **Setup Ruby** - Installs Ruby 3.2 with bundler cache for Fastlane
7. **Download Secure Files from S3** - Downloads keystore, API keys, and Firebase configs
8. **Build and Deploy** - Executes Fastlane lane `build_and_deploy`

---

## Fastlane Configuration

### Fastfile (`/android/fastlane/Fastfile`)

#### Main Lanes:

**1. `bootstrap` Lane (lines 8-12)**
- Installs Flutter using the fastlane-plugin-flutter
- Sets Flutter channel to stable

**2. `build` Lane (lines 15-67)**
- Fetches latest build number from Google Play internal track
- Auto-increments build number (`last_build_number + 1`)
- Generates code files using `dart run build_runner`
- Builds Android App Bundle (AAB) with:
  - Release mode
  - Code shrinking enabled (`--shrink`)
  - Production flavor from `FLUTTER_FLAVOR` env var
  - All dart-define parameters for configuration
- Locates the built AAB file in the output directory
- Returns path to built AAB file

**3. `deploy` Lane (lines 70-75)**
- Uploads AAB to Google Play Internal Testing track
- Uses Google Play API credentials from `google_key.json`

**4. `build_and_deploy` Lane (lines 78-86)**
- Runs bootstrap (if running in CI)
- Executes build lane
- Executes deploy lane with AAB path

#### Build Configuration Details:
```ruby
build: 'appbundle'                    # Builds AAB (required for Play Store)
build_number: last_build_number + 1   # Auto-incremented from Play Console
build_args: [
  '--verbose',
  '--release',                        # Production build
  '--shrink',                         # Enable code shrinking
  '--flavor', ENV.fetch('FLUTTER_FLAVOR'),
  # All --dart-define parameters...
]
```

---

## Appfile (`/android/fastlane/Appfile`)

```ruby
json_key_file('~/google_key.json')              # Google Play API credentials
package_name(ENV.fetch('ANDROID_APP_IDENTIFIER')) # App package name
```

The `google_key.json` file is downloaded from S3 during the CI build process.

---

## Gradle Build Configuration

### Root build.gradle (`/android/build.gradle`)

**Key Settings:**
- **Kotlin Version**: 1.9.24
- **Compile SDK**: 36
- **Target SDK**: 35
- **Min SDK**: 24
- **Repositories**: Google Maven, Maven Central, custom plugin repos

### App build.gradle (`/android/app/build.gradle`)

#### Application Configuration (lines 36-61)

```gradle
android {
    namespace System.getenv('ANDROID_APP_IDENTIFIER')
    compileSdk 36

    defaultConfig {
        applicationId System.getenv('ANDROID_APP_IDENTIFIER')
        minSdkVersion 24
        targetSdkVersion 35
        multiDexEnabled true

        manifestPlaceholders += [
            'appAuthRedirectScheme': System.getenv('ANDROID_APP_IDENTIFIER'),
            'geolocationApiKey': System.getenv('BACKGROUND_LOCATION_API_KEY')
        ]
    }
}
```

#### Signing Configuration (lines 73-80)

The signing configuration supports two sources for credentials:

1. **Local `key.properties` file** (for local development)
2. **Environment variables** (for CI/CD)

```gradle
signingConfigs {
    release {
        keyAlias = keystoreProperties['keyAlias']
                   ?: System.getenv('ANDROID_KEYSTORE_ALIAS')

        keyPassword = keystoreProperties['keyPassword']
                      ?: System.getenv('ANDROID_KEYSTORE_KEY_PASSWORD')

        storeFile = keystoreProperties['storeFile']
                    ? file(keystoreProperties['storeFile'])
                    : file(System.getenv('HOME') + '/keystore')

        storePassword = keystoreProperties['storePassword']
                        ?: System.getenv('ANDROID_KEYSTORE_PASSWORD')
    }
}
```

**Signing Logic:**
- Tries to read from `key.properties` first (if file exists)
- Falls back to environment variables (used in CI)
- In CI, keystore file is downloaded to `~/keystore`

#### Build Types (lines 82-93)

```gradle
buildTypes {
    release {
        signingConfig = signingConfigs.release.storeFile.exists()
                        ? signingConfigs.release
                        : signingConfigs.debug
        shrinkResources false
        proguardFiles "${background_geolocation.projectDir}/proguard-rules.pro"
    }
}
```

**Note**: Uses release signing if keystore exists, otherwise falls back to debug signing.

#### Product Flavors (lines 95-105)

```gradle
flavorDimensions "flavor-type"

productFlavors {
    beta {
        dimension "flavor-type"
    }

    production {
        dimension "flavor-type"
    }
}
```

Creates build variants:
- `betaRelease`
- `productionRelease`

#### 16KB Page Size Support (lines 111-117)

```gradle
packagingOptions {
    jniLibs {
        useLegacyPackaging = false
    }
}
```

Ensures compatibility with Android 15+ devices (Google Play requirement from Nov 2025).

### gradle.properties (`/android/gradle.properties`)

```properties
org.gradle.jvmargs=-Xmx4G                          # Gradle memory allocation
android.useAndroidX=true                           # AndroidX support
android.enableJetifier=true                        # Jetifier for legacy deps
android.bundle.pageAlignTo16KB=true                # 16KB page alignment
```

---

## Variables and Secrets Inventory

### AWS Configuration

#### GitHub Variables (Public)
| Variable | Used In | Purpose |
|----------|---------|---------|
| `AWS_ACCESS_KEY_ID` | GitHub Actions | AWS authentication for S3 access |

#### GitHub Secrets (Encrypted)
| Secret | Used In | Purpose |
|--------|---------|---------|
| `AWS_SECRET_ACCESS_KEY` | GitHub Actions | AWS authentication for S3 access |

#### AWS Details
- **Region**: `eu-west-2` (London)
- **S3 Bucket**: `tca-secure-assets`

---

### Android Signing & Deployment

#### GitHub Variables
| Variable | Used In | Purpose |
|----------|---------|---------|
| `ANDROID_APP_IDENTIFIER` | Fastlane, Gradle | Package name (e.g., `uk.org.cruising.captainsmate`) |
| `ANDROID_KEYSTORE_ALIAS` | Gradle signing | Keystore key alias |

#### GitHub Secrets
| Secret | Used In | Purpose |
|--------|---------|---------|
| `ANDROID_KEYSTORE_KEY_PASSWORD` | Gradle signing | Password for the key in the keystore |
| `ANDROID_KEYSTORE_PASSWORD` | Gradle signing | Password for the keystore file itself |

---

### S3 Stored Files (Android)

Downloaded in GitHub Actions step "Download keystore and Google API file" (lines 50-56):

| File | S3 Path | Downloaded To | Purpose |
|------|---------|---------------|---------|
| Google API Key | `s3://tca-secure-assets/google_app_developer_api_key.json` | `~/google_key.json` | Google Play Console API authentication |
| Android Keystore | `s3://tca-secure-assets/android_keystore_release.keystore` | `~/keystore` | Release APK/AAB signing |
| Firebase Config | `s3://tca-secure-assets/firebase.json` | `~/firebase.json` | Firebase app distribution config |
| Google Services | `s3://tca-secure-assets/google-services.json` | `android/app/google-services.json` | Android Firebase SDK configuration |

---

### Application Configuration

#### GitHub Variables
| Variable | Purpose | Used In |
|----------|---------|---------|
| `DEFINE_OAUTH_CLIENT_ID` | OAuth client ID | Flutter build (dart-define) |
| `DEFINE_OAUTH_SCOPES` | OAuth scopes | Flutter build (dart-define) |
| `DEFINE_OAUTH_AUTH_ENDPOINT` | OAuth authorization URL | Flutter build (dart-define) |
| `DEFINE_OAUTH_TOKEN_ENDPOINT` | OAuth token URL | Flutter build (dart-define) |
| `DEFINE_OAUTH_USER_ENDPOINT` | OAuth user info URL | Flutter build (dart-define) |
| `DEFINE_API_BASE_URL` | Production API base URL | Flutter build (dart-define) |
| `DEFINE_STAGING_API_BASE_URL` | Staging API base URL | Flutter build (dart-define) |

#### GitHub Secrets
| Secret | Purpose | Used In |
|--------|---------|---------|
| `DEFINE_SENTRY_DSN` | Sentry error tracking DSN | Flutter build (dart-define) |
| `BACKGROUND_LOCATION_API_KEY` | Background geolocation API key | Flutter build (dart-define), Gradle manifest |

---

### Firebase Configuration (Android)

#### GitHub Secrets
| Secret | Purpose |
|--------|---------|
| `DEFINE_FIREBASE_ANDROID_APP_ID` | Firebase Android app ID |
| `DEFINE_FIREBASE_ANDROID_API_KEY` | Firebase Android API key |
| `DEFINE_FIREBASE_SENDER_ID` | Firebase Cloud Messaging sender ID (shared) |
| `DEFINE_FIREBASE_BUCKET_NAME` | Firebase Storage bucket (shared) |
| `DEFINE_FIREBASE_PROJECT_ID` | Firebase project ID (shared) |

---

## Build and Signing Flow

### 1. Preparation Phase (GitHub Actions)
1. **Download Keystore**: Retrieved from `s3://tca-secure-assets/android_keystore_release.keystore` to `~/keystore`
2. **Download API Key**: Retrieved from S3 to `~/google_key.json`
3. **Download Firebase Configs**: Both `firebase.json` and `google-services.json`
4. **Set Environment Variables**: All secrets and variables made available

### 2. Build Phase (Fastlane)
1. **Version Code**: Fetches latest from Google Play internal track and increments
2. **Code Generation**: Runs `build_runner` to generate code files
3. **Flutter Build**: Builds App Bundle with:
   - Production flavor
   - Release mode
   - Code shrinking enabled
   - All configuration via `--dart-define`

### 3. Signing Phase (Gradle)
During the Flutter build, Gradle automatically signs the AAB:

1. **Reads Signing Config**: From environment variables:
   - `ANDROID_KEYSTORE_ALIAS`
   - `ANDROID_KEYSTORE_KEY_PASSWORD`
   - `ANDROID_KEYSTORE_PASSWORD`
2. **Locates Keystore**: At `~/keystore` (downloaded from S3)
3. **Signs AAB**: Using the release signing configuration
4. **Output**: Signed AAB at `build/app/outputs/bundle/productionRelease/*.aab`

### 4. Deployment Phase (Fastlane)
1. **Upload**: Uses `upload_to_play_store` action
2. **Authentication**: Via `~/google_key.json` (Google Play API service account)
3. **Track**: Uploads to `internal` testing track
4. **Package Name**: From `ANDROID_APP_IDENTIFIER` environment variable

---

## Build Output Locations

### AAB Location
The build process looks for the AAB file in:
```
build/app/outputs/bundle/{flavor}Release/*.aab
```

For production flavor:
```
build/app/outputs/bundle/productionRelease/app-production-release.aab
```

For beta flavor:
```
build/app/outputs/bundle/betaRelease/app-beta-release.aab
```

---

## Deployment Targets

### Google Play Internal Testing Track
- **Track**: `internal`
- **Purpose**: Internal testing before wider release
- **Version Code**: Auto-incremented from this track
- **Upload Method**: Google Play Developer API v3

### Future Tracks (Not Currently Used)
The app can be promoted to:
- `alpha` - Alpha testing
- `beta` - Beta testing
- `production` - Production release

---

## Deployment Triggers

### Automatic Deployment
- **Push to `master` branch**: Triggers both Android and iOS builds

### Manual Deployment
- **`workflow_dispatch`**: Can be manually triggered from GitHub Actions UI

---

## Security Considerations

### Encrypted in GitHub Secrets
- AWS secret access key
- Keystore passwords (2 passwords: store and key)
- Sentry DSN
- All Firebase credentials
- Background location API key

### Stored in S3 (Encrypted at Rest)
- Release keystore file
- Google Play API service account JSON
- Firebase configuration files

### Public GitHub Variables
- AWS access key ID (public but non-sensitive)
- Package identifier
- API endpoints
- OAuth client IDs (public by nature)

### Security Best Practices in Use
✅ Keystore stored remotely (not in repository)
✅ Passwords in GitHub Secrets (encrypted)
✅ Google Play API key stored securely (not hardcoded)
✅ Firebase configs downloaded dynamically (not committed)
✅ Uses service account for Play Store (not personal account)

---

## Dependencies

### Ruby Gems (Gemfile)
- `fastlane` - Build automation

### Fastlane Plugins (Pluginfile)
- `fastlane-plugin-flutter` - Flutter build integration

### Gradle Plugins (build.gradle)
- `com.android.application` - Android app plugin
- `com.google.gms.google-services` - Google Services plugin
- `kotlin-android` - Kotlin support
- `dev.flutter.flutter-gradle-plugin` - Flutter Gradle plugin

### Key Libraries (app/build.gradle dependencies)
- `com.android.support:multidex` - MultiDex support
- `androidx.window` - Window management
- `desugar_jdk_libs` - Java 8+ API support for older Android versions

---

## Flavors and Build Variants

### Available Flavors
1. **beta** - Beta builds (for testing)
2. **production** - Production builds (for release)

### Build Types
1. **debug** - Development builds
2. **release** - Release builds (signed with release keystore)

### Build Variants Matrix
The combination creates these variants:
- `betaDebug`
- `betaRelease`
- `productionDebug`
- `productionRelease`

**CI/CD uses**: `productionRelease` (set via `FLUTTER_FLAVOR: "production"`)

---

## Key Configuration Sources

### Environment Variables in Build
The following environment variables are injected during build:

**From Gradle:**
- `ANDROID_APP_IDENTIFIER` - Package name and namespace
- `BACKGROUND_LOCATION_API_KEY` - Injected into manifest

**From Fastlane (via flutter_build):**
- All `--dart-define` parameters for app configuration

### Manifest Placeholders
Defined in `app/build.gradle` (lines 57-60):
```gradle
manifestPlaceholders += [
    'appAuthRedirectScheme': System.getenv('ANDROID_APP_IDENTIFIER'),
    'geolocationApiKey': System.getenv('BACKGROUND_LOCATION_API_KEY')
]
```

These are replaced in `AndroidManifest.xml` during build.

---

## Android 15+ Compatibility

### 16KB Page Size Support
The project includes configuration for Android 15+ devices with 16KB page sizes:

1. **gradle.properties** (line 6):
   ```properties
   android.bundle.pageAlignTo16KB=true
   ```

2. **app/build.gradle** (lines 113-117):
   ```gradle
   packagingOptions {
       jniLibs {
           useLegacyPackaging = false
       }
   }
   ```

This ensures:
- Native libraries are uncompressed
- Proper alignment for 16KB page sizes
- Compliance with Google Play requirements from November 2025

---

## Troubleshooting Reference

### Common Issues

1. **Keystore Not Found**
   - Verify S3 bucket `tca-secure-assets` is accessible
   - Check AWS credentials (`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`)
   - Ensure keystore downloaded to `~/keystore`

2. **Signing Failures**
   - Verify `ANDROID_KEYSTORE_ALIAS` matches the alias in the keystore
   - Check both passwords are correct:
     - `ANDROID_KEYSTORE_PASSWORD` (for the store)
     - `ANDROID_KEYSTORE_KEY_PASSWORD` (for the key)
   - Ensure keystore file is not corrupted

3. **Google Play API Failures**
   - Verify `google_key.json` downloaded successfully
   - Check service account has necessary permissions in Google Play Console
   - Ensure `ANDROID_APP_IDENTIFIER` matches the app in Play Console

4. **Version Code Conflicts**
   - Fastlane auto-increments from internal track
   - If conflict occurs, may need to manually increment
   - Check version codes across all tracks in Play Console

5. **Build Variant Issues**
   - Verify `FLUTTER_FLAVOR` is set correctly (e.g., "production")
   - Flavor name must match exactly (case-sensitive)
   - AAB output path depends on flavor name (lowercase)

6. **Firebase Configuration Missing**
   - Ensure `google-services.json` downloaded to `android/app/`
   - Verify `com.google.gms.google-services` plugin is applied
   - Check Firebase credentials in dart-defines

---

## Recommendations

### Security Improvements
1. Implement keystore rotation policy
2. Rotate AWS credentials periodically
3. Audit S3 bucket permissions regularly
4. Consider separating staging and production keystores
5. Enable AWS CloudTrail for S3 access logging

### Process Improvements
1. Add separate workflow for beta vs production
2. Implement staged rollout percentages
3. Add automated testing before deployment
4. Add deployment approval gates for production track
5. Configure release notes automation
6. Add APK generation for testing (in addition to AAB)

### Monitoring
1. Set up Google Play Console API alerts
2. Monitor crash reports via Sentry
3. Track deployment success/failure metrics
4. Set up Firebase Crashlytics for detailed crash reports

---

## Quick Reference Commands

### Local Development

#### Setup
```bash
cd android
bundle install
```

#### Build AAB Locally (requires env vars)
```bash
# Set required environment variables first
export ANDROID_APP_IDENTIFIER="uk.org.cruising.captainsmate"
export FLUTTER_FLAVOR="production"
# ... set all other required env vars

# Run build
bundle exec fastlane build
```

#### Deploy to Play Store
```bash
bundle exec fastlane deploy path:/path/to/app.aab
```

#### Full Build and Deploy
```bash
bundle exec fastlane build_and_deploy
```

### Gradle Commands

#### Build Production Release
```bash
cd android
./gradlew bundleProductionRelease
```

#### Build Beta Release
```bash
./gradlew bundleBetaRelease
```

#### Clean Build
```bash
./gradlew clean
```

#### Check Signing Config
```bash
./gradlew signingReport
```

---

## Integration with Firebase

### Firebase Plugins Used
1. **Google Services Plugin**: Processes `google-services.json`
2. **Firebase SDK**: Configured via downloaded `GoogleService-Info.plist`

### Firebase Configuration Flow
1. `google-services.json` downloaded from S3 to `android/app/`
2. Google Services plugin processes it during build
3. Firebase credentials passed via `--dart-define` for runtime config
4. App initializes Firebase with provided configuration

---

## File Reference

| File | Purpose |
|------|---------|
| `/.github/workflows/actions.yml` | CI/CD workflow definition |
| `/android/fastlane/Fastfile` | Build and deployment lanes |
| `/android/fastlane/Appfile` | Package and API configuration |
| `/android/fastlane/Pluginfile` | Fastlane plugin dependencies |
| `/android/Gemfile` | Ruby dependencies |
| `/android/build.gradle` | Root Gradle configuration |
| `/android/app/build.gradle` | App module Gradle configuration |
| `/android/gradle.properties` | Gradle properties and flags |
| `/android/settings.gradle` | Gradle project settings |

---

## API & Service Accounts

### Google Play Console
- **Service Account Key**: `google_app_developer_api_key.json` (in S3)
- **Package Name**: From `ANDROID_APP_IDENTIFIER`
- **Upload Track**: `internal`

### AWS S3
- **Bucket**: `tca-secure-assets`
- **Region**: `eu-west-2`
- **Access**: Via IAM credentials

---

## Version Management

### Version Code
- **Source**: Google Play internal track
- **Method**: Auto-increment by Fastlane
- **Lane**: `google_play_track_version_codes(track: 'internal')`

### Version Name
- **Source**: `pubspec.yaml` (Flutter project)
- **Method**: Read by Flutter during build
- **Format**: Semantic versioning (e.g., 1.2.3)

---

*Last Updated: 2026-01-07*