---
id: c9459837-7d4a-4d5c-902d-76c1a1a9a923
blueprint: captains_mate_app_43
title: 'Android Handover Checklist'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774536323
---
# Android Build & Deployment Handover Checklist

## Overview
This checklist ensures the new developer has everything needed to build and release the Android app manually without CI/CD pipeline access.

---

## 1. Google Play Console Access

### Required Access
- [ ] **Google Play Console** login credentials
  - Provide account email and password/2FA method
  - Ensure access includes app management and release permissions
  - Access to internal testing track

### App Information
- [ ] Package Name: `uk.org.cruising.captainsmate`
- [ ] Current version code and version name in Play Console
- [ ] Any pending releases or reviews

### Google Play Developer API
- [ ] **Service Account Email**: `<provide email from google_key.json>`
- [ ] Verify service account has proper permissions:
  - Release Manager or Admin role
  - API access enabled in Play Console

---

## 2. Android Signing Keystore

### Keystore File (Option 1: Via S3)
- [ ] **AWS S3 Access**
  - AWS Access Key ID: `<provide value>`
  - AWS Secret Access Key: `<provide value>`
  - Bucket: `tca-secure-assets`
  - Region: `eu-west-2`
  - File path: `s3://tca-secure-assets/android_keystore_release.keystore`

### Keystore File (Option 2: Direct Export)
If S3 access not available:
- [ ] **Release Keystore** (`android_keystore_release.keystore`)
  - Export from current developer's machine
  - Provide file directly (store securely!)
  - Destination: `~/keystore` (home directory)

### Keystore Credentials
- [ ] **Keystore Password**: `<provide ANDROID_KEYSTORE_PASSWORD>`
  - This is the password to open the keystore file itself

- [ ] **Key Alias**: `<provide ANDROID_KEYSTORE_ALIAS>`
  - The alias name of the signing key inside the keystore

- [ ] **Key Password**: `<provide ANDROID_KEYSTORE_KEY_PASSWORD>`
  - The password for the specific key (may be same as keystore password)

### Keystore Information
- [ ] Keystore algorithm: `<e.g., RSA>`
- [ ] Key validity period: `<expiry date>`
- [ ] Certificate fingerprint (SHA-256): `<provide for verification>`
- [ ] Set calendar reminder for keystore expiry

**⚠️ CRITICAL**: The Android keystore is irreplaceable. If lost, you cannot update the app in Play Store. Store multiple secure backups.

---

## 3. Google Play API Key

Required for automated uploads to Play Store:

- [ ] **API Key File** (`google_key.json`)
  - Download from S3: `s3://tca-secure-assets/google_app_developer_api_key.json`
  - OR generate new one from Google Play Console:
    - Settings → API access → Service accounts
    - Grant permissions (Release Manager role minimum)
  - Destination: `~/google_key.json`

- [ ] **Service Account Details**
  - Service account email: `<provide value>`
  - Key ID: `<provide value>`
  - Creation date: `<date>`

---

## 4. Firebase Configuration

### Required Files

- [ ] **google-services.json**
  - Download from S3: `s3://tca-secure-assets/google-services.json`
  - OR download from Firebase Console → Project Settings → Android app
  - Destination: `android/app/google-services.json`

- [ ] **firebase.json** (for Firebase App Distribution, if used)
  - Download from S3: `s3://tca-secure-assets/firebase.json`
  - Destination: `~/firebase.json`

### Firebase Credentials (dart-define values)

- [ ] `DEFINE_FIREBASE_ANDROID_APP_ID` = `<provide value>`
- [ ] `DEFINE_FIREBASE_ANDROID_API_KEY` = `<provide value>`
- [ ] `DEFINE_FIREBASE_SENDER_ID` = `<provide value>`
- [ ] `DEFINE_FIREBASE_BUCKET_NAME` = `<provide value>`
- [ ] `DEFINE_FIREBASE_PROJECT_ID` = `<provide value>`

---

## 5. Application Configuration Variables

### OAuth Configuration
- [ ] `DEFINE_OAUTH_CLIENT_ID` = `<provide value>`
- [ ] `DEFINE_OAUTH_SCOPES` = `<provide value>`
- [ ] `DEFINE_OAUTH_AUTH_ENDPOINT` = `<provide value>`
- [ ] `DEFINE_OAUTH_TOKEN_ENDPOINT` = `<provide value>`
- [ ] `DEFINE_OAUTH_USER_ENDPOINT` = `<provide value>`

### API Endpoints
- [ ] `DEFINE_API_BASE_URL` = `<provide value>` (Production)
- [ ] `DEFINE_STAGING_API_BASE_URL` = `<provide value>` (Staging)

### Error Tracking
- [ ] `DEFINE_SENTRY_DSN` = `<provide value>`

### Location Services
- [ ] `BACKGROUND_LOCATION_API_KEY` = `<provide value>`

### Package Identifier
- [ ] `ANDROID_APP_IDENTIFIER` = `uk.org.cruising.captainsmate`

### Build Flavor
- [ ] `FLUTTER_FLAVOR` = `production` (or `beta` for testing)

---

## 6. GitHub Access (Optional - for CI/CD understanding)

- [ ] **GitHub Repository Access**
  - Repository URL: `<provide URL>`
  - Read access to view workflows and secrets list

- [ ] **GitHub Actions Access** (if continuing to use CI/CD)
  - Ability to trigger manual workflow runs
  - Access to view workflow run logs

---

## 7. AWS S3 Access Summary

If providing S3 access for all secure assets:

### Bucket: `tca-secure-assets`
- [ ] `android_keystore_release.keystore`
- [ ] `google_app_developer_api_key.json`
- [ ] `google-services.json`
- [ ] `firebase.json`

### AWS Credentials
- [ ] AWS Access Key ID: `<provide value>`
- [ ] AWS Secret Access Key: `<provide value>`
- [ ] Region: `eu-west-2`
- [ ] IAM permissions: Read access to bucket

---

## 8. Development Environment Setup

### Software Requirements
- [ ] Operating System: Linux, macOS, or Windows
- [ ] Java JDK: Version 17 (Temurin distribution recommended)
- [ ] Android SDK: Platform tools and build tools installed
- [ ] Flutter SDK: 3.32.6 (stable channel) or current version
- [ ] Ruby: 3.2 or later
- [ ] Fastlane installed (`bundle install` in android/ directory)
- [ ] Android Studio (optional but recommended for SDK management)

### Android SDK Components
- [ ] Android SDK Platform 36 (compile SDK)
- [ ] Android SDK Platform 35 (target SDK)
- [ ] Android SDK Platform 24 (minimum SDK)
- [ ] Android SDK Build-Tools (latest version)
- [ ] Android SDK Command-line Tools

### Verification Commands
```bash
# Verify installations
java -version        # Should show Java 17
flutter --version    # Should show Flutter 3.32.6 or current
ruby --version       # Should show Ruby 3.2+
bundle --version     # Should show Bundler installed
```

---

## 9. Local Build Instructions

### Create Environment Setup Script

Provide a `.env` file or script with all environment variables:

```bash
# Android Configuration
export ANDROID_APP_IDENTIFIER="uk.org.cruising.captainsmate"
export ANDROID_KEYSTORE_ALIAS="<value>"
export ANDROID_KEYSTORE_PASSWORD="<value>"
export ANDROID_KEYSTORE_KEY_PASSWORD="<value>"
export FLUTTER_FLAVOR="production"

# OAuth Configuration
export DEFINE_OAUTH_CLIENT_ID="<value>"
export DEFINE_OAUTH_SCOPES="<value>"
export DEFINE_OAUTH_AUTH_ENDPOINT="<value>"
export DEFINE_OAUTH_TOKEN_ENDPOINT="<value>"
export DEFINE_OAUTH_USER_ENDPOINT="<value>"

# API Endpoints
export DEFINE_API_BASE_URL="<value>"
export DEFINE_STAGING_API_BASE_URL="<value>"

# Error Tracking
export DEFINE_SENTRY_DSN="<value>"

# Location Services
export BACKGROUND_LOCATION_API_KEY="<value>"

# Firebase Android
export DEFINE_FIREBASE_ANDROID_APP_ID="<value>"
export DEFINE_FIREBASE_ANDROID_API_KEY="<value>"
export DEFINE_FIREBASE_SENDER_ID="<value>"
export DEFINE_FIREBASE_BUCKET_NAME="<value>"
export DEFINE_FIREBASE_PROJECT_ID="<value>"

# AWS (if using S3)
export AWS_ACCESS_KEY_ID="<value>"
export AWS_SECRET_ACCESS_KEY="<value>"
```

### Test Build Commands
- [ ] Provide working example of local build command
- [ ] Document any custom build flags or configurations
- [ ] Explain flavor system (production vs beta)

---

## 10. Key.properties File (Alternative to Environment Variables)

For local development, provide a `key.properties` file template:

```properties
storeFile=/path/to/keystore
storePassword=<KEYSTORE_PASSWORD>
keyAlias=<KEY_ALIAS>
keyPassword=<KEY_PASSWORD>
```

- [ ] **File Location**: `android/key.properties`
- [ ] **Security Note**: NEVER commit this file to git (already in .gitignore)
- [ ] Provide filled-in template securely (password manager, encrypted file, etc.)

---

## 11. Knowledge Transfer Sessions

### Required Walkthroughs
- [ ] **Keystore Management**
  - Explain keystore importance and backup strategy
  - Show how to verify keystore integrity
  - Demonstrate keystore password usage
  - Explain consequences of losing keystore

- [ ] **Build Process**
  - Walk through Fastlane lanes
  - Explain build versioning strategy (version codes)
  - Show how version codes are auto-incremented
  - Explain flavors (production vs beta)

- [ ] **Gradle Configuration**
  - Review signing configuration in app/build.gradle
  - Explain manifest placeholders
  - Show product flavors setup
  - Discuss 16KB page size compatibility

- [ ] **Deployment Process**
  - Google Play Internal Testing track upload
  - Promoting builds to other tracks (alpha, beta, production)
  - Release notes and version management
  - Staged rollout strategy

- [ ] **Troubleshooting Common Issues**
  - Signing failures
  - Version code conflicts
  - Firebase configuration issues
  - Google Play API authentication problems

---

## 12. Documentation & Resources

- [ ] **Access to Internal Documentation**
  - Wiki or confluence pages
  - Architecture diagrams
  - API documentation

- [ ] **Contact Information**
  - Who manages Google Play Console
  - Who manages AWS infrastructure
  - Firebase admin contacts
  - Emergency contacts for production issues

- [ ] **Reference Links**
  - Google Play Console: https://play.google.com/console
  - Firebase Console: https://console.firebase.google.com
  - AWS Console: https://console.aws.amazon.com
  - Fastlane Documentation: https://docs.fastlane.tools/

---

## 13. Security & Access Management

- [ ] **Password Manager Access**
  - If using 1Password/LastPass/etc for credentials
  - Shared vault or collection name

- [ ] **2FA Backup Codes**
  - Google Play Console account 2FA backup codes
  - AWS MFA backup (if enabled)

- [ ] **Keystore Backups**
  - Primary backup location: `<location>`
  - Secondary backup location: `<location>`
  - Last verified date: `<date>`
  - Backup verification schedule

- [ ] **Certificate Information**
  - Keystore creation date: `<date>`
  - Keystore expiry date: `<date>`
  - Set renewal reminders (keystore doesn't expire but good to track)

---

## 14. Current State & Pending Work

- [ ] **Last Successful Build**
  - Date: `<date>`
  - Version name: `<e.g., 1.2.3>`
  - Version code: `<e.g., 45>`
  - Commit hash: `<hash>`
  - Track: `<internal/beta/production>`

- [ ] **Known Issues**
  - Any ongoing build problems
  - Deprecated dependencies
  - Pending library updates

- [ ] **Pending Releases**
  - Any versions awaiting Play Store review
  - Planned upcoming releases
  - Features in development

---

## 15. Google Play Console Configuration

### App Details
- [ ] **App Name**: `<provide>`
- [ ] **Package Name**: `uk.org.cruising.captainsmate`
- [ ] **Developer Account**: `<email>`

### Release Tracks
- [ ] **Internal Testing**: Current setup (auto-upload destination)
- [ ] **Alpha**: `<if configured>`
- [ ] **Beta**: `<if configured>`
- [ ] **Production**: Current live version

### Testers
- [ ] Internal testing list emails: `<provide>`
- [ ] Closed testing list (if any): `<provide>`
- [ ] Open testing configuration (if any)

### App Signing
- [ ] **Play App Signing**: Enabled/Disabled
- [ ] **App signing key certificate**: `<SHA-256 fingerprint>`
- [ ] **Upload key certificate**: `<SHA-256 fingerprint>`
- [ ] Explain difference if Play App Signing is enabled

---

## 16. Emergency Procedures

- [ ] **If Build Fails**
  - Rollback procedures
  - Where to find previous working builds
  - Who to notify

- [ ] **If Keystore is Lost/Compromised**
  - Immediate actions
  - Contact Google Play support
  - Impact assessment (app cannot be updated!)
  - Recovery options (none - new app required)

- [ ] **Production Incident Response**
  - On-call contacts
  - Incident escalation process
  - Emergency rollback via Play Console
  - Staged rollout halt procedures

---

## 17. Build Variants & Flavors

### Flavors Configured
- [ ] **Production**: Production release builds
  - Package ID suffix: none
  - Configuration: Production API endpoints

- [ ] **Beta**: Beta testing builds
  - Package ID suffix: `<if any>`
  - Configuration: Staging or beta endpoints

### Build Commands by Variant
- [ ] Production release: `./gradlew bundleProductionRelease`
- [ ] Beta release: `./gradlew bundleBetaRelease`
- [ ] Explain which variant is used for which purpose

---

## 18. Gradle Configuration Details

### Key Settings
- [ ] Compile SDK: 36
- [ ] Target SDK: 35
- [ ] Min SDK: 24
- [ ] Kotlin version: 1.9.24
- [ ] MultiDex enabled: Yes
- [ ] 16KB page alignment: Enabled (Android 15+ compatibility)

### Important Gradle Files
- [ ] Walk through `android/build.gradle` (root)
- [ ] Walk through `android/app/build.gradle` (app module)
- [ ] Explain `gradle.properties` settings
- [ ] Review signing configuration section

---

## 19. Continuous Integration (Optional)

If new developer will maintain CI/CD:

- [ ] **GitHub Actions Workflow**
  - Location: `.github/workflows/actions.yml`
  - Android job: lines 10-86
  - Explain workflow triggers (push to master, manual dispatch)

- [ ] **GitHub Secrets Configuration**
  - List all required secrets
  - Explain how to update secrets
  - Document secret rotation policy

- [ ] **GitHub Variables Configuration**
  - List all required variables
  - Explain public vs secret values

---

## 20. Testing Strategy

- [ ] **Pre-Release Testing**
  - Internal testing track process
  - QA checklist or test plan
  - Regression testing requirements

- [ ] **Device Testing**
  - Minimum test devices/configurations
  - Known device-specific issues

- [ ] **Automated Testing**
  - Unit tests: `flutter test`
  - Integration tests: `<if configured>`
  - Where test coverage reports are stored

---

## Handover Completion

### Sign-off

**Previous Developer:**
- Name: ___________________________
- Date: ___________________________
- Signature: ___________________________

**New Developer:**
- Name: ___________________________
- Date: ___________________________
- Signature: ___________________________

### Post-Handover Verification

Within first week:
- [ ] New developer successfully cloned repository
- [ ] New developer can build project locally
- [ ] New developer can access all required accounts
- [ ] New developer completed test build locally
- [ ] New developer successfully uploaded to internal testing
- [ ] All environment variables verified and working
- [ ] All questions answered

---

## Notes & Additional Information

_Use this space for any additional context, quirks, or important information:_

```
<Add any additional notes here>

Important Gradle quirks:
-

Known issues:
-

Recent changes:
-

Upcoming planned changes:
-
```

---

## Critical Reminders

### 🔴 NEVER LOSE THE KEYSTORE
- The Android keystore is **irreplaceable**
- If lost, you **cannot update the app** in Play Store
- Maintain **multiple secure backups**
- Verify backups regularly

### 🔴 NEVER COMMIT SECRETS
- Never commit `key.properties`
- Never commit `keystore` files
- Never commit `google_key.json`
- Never commit `google-services.json`
- All sensitive files are in `.gitignore`

### 🔴 VERSION CODE MANAGEMENT
- Version codes must always increment
- Cannot reuse a version code once uploaded
- Fastlane auto-increments from internal track
- Manual builds require manual version code tracking

---

*Document Created: 2026-01-07*
*Last Updated: 2026-01-07*