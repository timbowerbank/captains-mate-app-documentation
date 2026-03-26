---
id: dc6b6c8d-176a-4a4c-81c0-4dc768214616
blueprint: captains_mate_app_43
title: 'iOS Build Deployment'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774536490
---
# iOS Build, Signing, and Deployment Documentation

## Overview

This Flutter project uses **Fastlane** for iOS builds and deployment, **Match** for code signing and provisioning profile management, and **GitHub Actions** for CI/CD automation. The certificates and provisioning profiles are stored in an **AWS S3 bucket**, while sensitive credentials are managed through **GitHub Secrets** and **Variables**.

---

## Architecture Components

### 1. Fastlane Configuration
Located in `/ios/fastlane/`

### 2. Match (Code Signing)
Manages certificates and provisioning profiles via S3

### 3. GitHub Actions Workflow
Defined in `/.github/workflows/actions.yml`

### 4. Storage Locations
- **S3 Buckets**: `tca-secure-assets` (auth files) and `tca-app-certificates` (signing certificates)
- **GitHub**: Secrets and Variables for environment configuration

---

## Build Process Flow

### GitHub Actions iOS Job (`actions.yml:95-162`)

```yaml
Job: ios
Runs on: macos-latest
Environment: staging
```

#### Step-by-Step Process:

1. **Checkout Code** - Checks out the repository
2. **Setup Flutter** - Installs Flutter 3.32.6 (stable channel)
3. **Install Flutter Dependencies** - Runs `flutter pub get`
4. **Setup Ruby** - Installs Ruby 3.2 with bundler cache for fastlane
5. **Download Secure Files from S3** - Downloads Apple API key and Firebase config
6. **Build and Deploy** - Executes Fastlane lane `build_and_deploy`

---

## Fastlane Configuration

### Fastfile (`/ios/fastlane/Fastfile`)

#### Main Lanes:

**1. `bootstrap` Lane (lines 14-18)**
- Installs Flutter using the fastlane-plugin-flutter
- Sets Flutter channel to stable

**2. `build` Lane (lines 21-67)**
- Updates CocoaPods specs repository
- Runs **Match** to sync certificates and provisioning profiles
- Fetches latest TestFlight build number
- Generates code files using `dart run build_runner`
- Builds iOS IPA with Flutter using release mode and production flavor
- Exports IPA using Xcode
- Returns path to built IPA file

**3. `deploy` Lane (lines 70-75)**
- Uploads IPA to TestFlight
- Skips waiting for build processing

**4. `build_and_deploy` Lane (lines 78-86)**
- Runs bootstrap (if running in CI)
- Executes build lane
- Executes deploy lane with IPA path

#### App Store Connect API Configuration (lines 7-11)
```ruby
app_store_connect_api_key(
  key_id: ENV.fetch('APPLE_APP_DEVELOPER_KEY_ID'),
  issuer_id: ENV.fetch('APPLE_APP_DEVELOPER_ISSUER_ID'),
  key_filepath: '~/apple_key.p8'
)
```

---

## Match Configuration

### Matchfile (`/ios/fastlane/Matchfile`)

```ruby
s3_bucket("tca-app-certificates")        # S3 bucket storing certificates/profiles
s3_region("eu-west-2")                   # AWS region
storage_mode("s3")                       # Storage backend
type("development")                      # Default type
app_identifier([ENV.fetch('APPLE_APP_IDENTIFIER')])
username("appdeveloper@theca.org.uk")   # Apple Developer account
team_id("8W6EM53C5H")                   # Apple Team ID
```

#### Match Usage in Build Process:
In the build lane (line 25):
```ruby
match(type: 'appstore', app_identifier: ENV.fetch('APPLE_APP_IDENTIFIER'))
```

This downloads the App Store distribution certificate and provisioning profile from the S3 bucket `tca-app-certificates`.

---

## Appfile (`/ios/fastlane/Appfile`)

```ruby
app_identifier(ENV.fetch('APPLE_APP_IDENTIFIER'))  # Bundle ID from env var
apple_id("appdeveloper@theca.org.uk")              # Apple Developer email
```

---

## Export Options

### exportOptions.beta.plist & exportOptions.release.plist

Both files configure the IPA export with:
```xml
<key>method</key>
<string>app-store</string>              <!-- Distribution method -->

<key>teamID</key>
<string>8W6EM53C5H</string>             <!-- Apple Team ID -->

<key>provisioningProfiles</key>
<dict>
    <key>uk.org.cruising.captainsmate</key>
    <string>match AppStore uk.org.cruising.captainsmate</string>
</dict>
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
- **S3 Buckets**:
  - `tca-secure-assets` - Stores API keys and config files
  - `tca-app-certificates` - Stores Match certificates and provisioning profiles

---

### Apple Developer & App Store Connect

#### GitHub Variables
| Variable | Used In | Purpose |
|----------|---------|---------|
| `APPLE_APP_IDENTIFIER` | Fastlane | Bundle identifier (e.g., `uk.org.cruising.captainsmate`) |
| `APPLE_APP_DEVELOPER_ISSUER_ID` | Fastlane | App Store Connect API issuer ID |
| `APPLE_APP_DEVELOPER_KEY_ID` | Fastlane | App Store Connect API key ID |

#### GitHub Secrets
| Secret | Used In | Purpose |
|--------|---------|---------|
| `MATCH_PASSWORD` | Match/Fastlane | Decrypts Match certificates stored in S3 |

#### Hardcoded Values
| Value | Location | Description |
|-------|----------|-------------|
| `appdeveloper@theca.org.uk` | Appfile, Matchfile | Apple Developer account email |
| `8W6EM53C5H` | Matchfile, exportOptions plists | Apple Team ID |
| `uk.org.cruising.captainsmate` | exportOptions plists | Hardcoded bundle ID (should use env var) |

---

### S3 Stored Files (iOS)

Downloaded in GitHub Actions step "Download keystore and Google API file" (lines 119-128):

| File | S3 Path | Downloaded To | Purpose |
|------|---------|---------------|---------|
| Apple API Key | `s3://tca-secure-assets/apple_app_developer_api_key.p8` | `~/apple_key.p8` | App Store Connect API authentication |
| Firebase Config | `s3://tca-secure-assets/firebase.json` | `~/firebase.json` | Firebase app distribution config |
| GoogleService Info | `s3://tca-secure-assets/GoogleService-Info.plist` | `ios/Runner/GoogleService-Info.plist` | iOS Firebase SDK configuration |

**Note**: Match certificates and provisioning profiles are automatically synced from `s3://tca-app-certificates` by Match.

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

---

### Firebase Configuration (iOS)

#### GitHub Secrets
| Secret | Purpose |
|--------|---------|
| `DEFINE_FIREBASE_IOS_APP_ID` | Firebase iOS app ID |
| `DEFINE_FIREBASE_IOS_API_KEY` | Firebase iOS API key |
| `DEFINE_FIREBASE_IOS_BUNDLE_ID` | Firebase iOS bundle identifier |
| `DEFINE_FIREBASE_SENDER_ID` | Firebase Cloud Messaging sender ID (shared) |
| `DEFINE_FIREBASE_BUCKET_NAME` | Firebase Storage bucket (shared) |
| `DEFINE_FIREBASE_PROJECT_ID` | Firebase project ID (shared) |

---

## Code Signing & Provisioning Flow

### 1. Match Synchronization
When `match(type: 'appstore', ...)` is called in the build lane:

1. **Connects to S3**: Uses AWS credentials to access `tca-app-certificates` bucket
2. **Downloads Encrypted Files**: Retrieves certificate and provisioning profile
3. **Decrypts**: Uses `MATCH_PASSWORD` to decrypt files
4. **Installs**: Installs certificate to macOS keychain and provisioning profile to Xcode

### 2. Build Process
Flutter builds the iOS app with:
- Bundle ID from `APPLE_APP_IDENTIFIER`
- Signing certificate and provisioning profile installed by Match
- Build number auto-incremented from latest TestFlight build

### 3. Export
Xcode exports the IPA using:
- App Store distribution method
- Team ID: `8W6EM53C5H`
- Provisioning profile: `match AppStore uk.org.cruising.captainsmate`

### 4. Upload
Fastlane uploads to TestFlight using App Store Connect API:
- Key ID: `APPLE_APP_DEVELOPER_KEY_ID`
- Issuer ID: `APPLE_APP_DEVELOPER_ISSUER_ID`
- Key file: `~/apple_key.p8`

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
- Match password
- Sentry DSN
- All Firebase credentials
- Keystore passwords (Android)

### Stored in S3 (Encrypted at Rest)
- Apple API key (`.p8` file)
- Firebase configuration files
- Match certificates and provisioning profiles (encrypted with `MATCH_PASSWORD`)

### Public GitHub Variables
- AWS access key ID (public but non-sensitive)
- App identifiers
- API endpoints
- OAuth client IDs (public by nature)

### Hardcoded (Consider Moving to Variables)
- Apple Developer email
- Apple Team ID
- Bundle identifier in export options (should reference `APPLE_APP_IDENTIFIER`)

---

## Dependencies

### Ruby Gems (Gemfile)
- `fastlane` - Build automation
- `cocoapods` - iOS dependency management
- `openssl ~> 3.3.1` - Encryption support

### Fastlane Plugins (Pluginfile)
- `fastlane-plugin-flutter` - Flutter build integration

---

## Build Artifacts

### Generated Files
- **IPA file**: Located via `find_ipa` helper (likely in `build/ios/ipa/`)
- **Build logs**: Available in GitHub Actions
- **dSYM files**: Generated during build for crash reporting

---

## Troubleshooting Reference

### Common Issues

1. **Certificate/Profile Mismatch**
   - Check `MATCH_PASSWORD` is correct
   - Verify S3 bucket `tca-app-certificates` is accessible
   - Ensure `APPLE_APP_IDENTIFIER` matches provisioning profile

2. **API Authentication Failures**
   - Verify `apple_key.p8` downloaded successfully from S3
   - Check `APPLE_APP_DEVELOPER_KEY_ID` and `APPLE_APP_DEVELOPER_ISSUER_ID`
   - Ensure API key hasn't expired in App Store Connect

3. **Build Number Conflicts**
   - Fastlane auto-increments from latest TestFlight build
   - If sync fails, may need manual increment

4. **Pod Repository Out of Date**
   - Build lane includes `pod repo update` to prevent this

---

## Recommendations

### Security Improvements
1. Move hardcoded Apple Team ID to GitHub Variable
2. Use environment variable in exportOptions plist files for bundle ID
3. Rotate AWS keys periodically
4. Audit S3 bucket permissions regularly

### Process Improvements
1. Consider separate workflows for staging vs production
2. Add post-deployment notifications (Slack, email)
3. Implement automated testing before deployment
4. Add deployment approval gates for production

---

## Quick Reference Commands

### Local Development
```bash
# Install dependencies
cd ios
bundle install

# Run build lane locally (requires env vars)
bundle exec fastlane build

# Deploy to TestFlight
bundle exec fastlane deploy path:/path/to/app.ipa

# Full build and deploy
bundle exec fastlane build_and_deploy
```

### Match Commands
```bash
# Sync certificates (download from S3)
bundle exec fastlane match appstore

# Force refresh certificates
bundle exec fastlane match appstore --force

# List certificates
bundle exec fastlane match appstore --readonly
```

---

## File Reference

| File | Purpose |
|------|---------|
| `/.github/workflows/actions.yml` | CI/CD workflow definition |
| `/ios/fastlane/Fastfile` | Build and deployment lanes |
| `/ios/fastlane/Appfile` | App and Apple ID configuration |
| `/ios/fastlane/Matchfile` | Code signing configuration |
| `/ios/fastlane/Pluginfile` | Fastlane plugin dependencies |
| `/ios/Gemfile` | Ruby dependencies |
| `/ios/exportOptions.beta.plist` | IPA export configuration (beta) |
| `/ios/exportOptions.release.plist` | IPA export configuration (release) |
| `/ios/Podfile` | CocoaPods dependencies |

---

## Contact Information

**Apple Developer Account**: appdeveloper@theca.org.uk
**Apple Team ID**: 8W6EM53C5H
**AWS Region**: eu-west-2 (London)

---

*Last Updated: 2026-01-07*