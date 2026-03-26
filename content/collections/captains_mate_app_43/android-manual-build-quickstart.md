---
id: 797e9758-83e5-4eae-8778-d3fb17613e94
blueprint: captains_mate_app_43
title: 'Android Manual Build Quickstart'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774536381
---
# Android Manual Build Quick-Start Guide

## Overview
This guide walks you through building and deploying the Android app manually without using the CI/CD pipeline. Use this when you don't have access to GitHub Actions but need to release to Google Play.

---

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] Computer with Linux, macOS, or Windows
- [ ] Java JDK 17 installed (Temurin distribution recommended)
- [ ] Android SDK installed (via Android Studio or command-line tools)
- [ ] Flutter SDK installed (version 3.32.6 or current stable)
- [ ] Ruby 3.2+ installed
- [ ] Access to Google Play Console
- [ ] Release keystore file (`android_keystore_release.keystore`)
- [ ] Keystore credentials (alias and passwords)
- [ ] All environment variables from handover (see Section 9 of [ANDROID_HANDOVER_CHECKLIST.md](ANDROID_HANDOVER_CHECKLIST.md))
- [ ] `google-services.json` file downloaded
- [ ] `google_key.json` file (for automated upload)

---

## Method 1: Using Fastlane (Recommended)

This method uses the existing Fastlane setup for automated building and uploading.

### Step 1: Set Up Environment Variables

Create a file to store your environment variables:

```bash
# Create env file in project root
cd /Users/emmabowerbank/Development/CA/Cruising-Association-tca-mobile-app-3c628ea7950f
nano .env.android.local
```

Paste this content (fill in the actual values):

```bash
# Android Configuration
export ANDROID_APP_IDENTIFIER="uk.org.cruising.captainsmate"
export ANDROID_KEYSTORE_ALIAS="YOUR_ALIAS"
export ANDROID_KEYSTORE_PASSWORD="YOUR_KEYSTORE_PASSWORD"
export ANDROID_KEYSTORE_KEY_PASSWORD="YOUR_KEY_PASSWORD"
export FLUTTER_FLAVOR="production"

# OAuth Configuration
export DEFINE_OAUTH_CLIENT_ID="YOUR_VALUE"
export DEFINE_OAUTH_SCOPES="YOUR_VALUE"
export DEFINE_OAUTH_AUTH_ENDPOINT="YOUR_VALUE"
export DEFINE_OAUTH_TOKEN_ENDPOINT="YOUR_VALUE"
export DEFINE_OAUTH_USER_ENDPOINT="YOUR_VALUE"

# API Endpoints
export DEFINE_API_BASE_URL="YOUR_VALUE"
export DEFINE_STAGING_API_BASE_URL="YOUR_VALUE"

# Error Tracking
export DEFINE_SENTRY_DSN="YOUR_VALUE"

# Location Services
export BACKGROUND_LOCATION_API_KEY="YOUR_VALUE"

# Firebase Android
export DEFINE_FIREBASE_ANDROID_APP_ID="YOUR_VALUE"
export DEFINE_FIREBASE_ANDROID_API_KEY="YOUR_VALUE"
export DEFINE_FIREBASE_SENDER_ID="YOUR_VALUE"
export DEFINE_FIREBASE_BUCKET_NAME="YOUR_VALUE"
export DEFINE_FIREBASE_PROJECT_ID="YOUR_VALUE"

# AWS (if using S3)
export AWS_ACCESS_KEY_ID="YOUR_VALUE"
export AWS_SECRET_ACCESS_KEY="YOUR_VALUE"
```

Save and exit (Ctrl+X, then Y, then Enter).

### Step 2: Load Environment Variables

```bash
# Source the environment file
source .env.android.local

# Verify variables are loaded
echo $ANDROID_APP_IDENTIFIER
echo $FLUTTER_FLAVOR
```

### Step 3: Set Up Required Files

```bash
# Place keystore in home directory
cp /path/to/android_keystore_release.keystore ~/keystore

# Place Google Play API key
cp /path/to/google_app_developer_api_key.json ~/google_key.json

# Place Google Services config
cp /path/to/google-services.json android/app/google-services.json

# Place Firebase config (if using Firebase App Distribution)
cp /path/to/firebase.json ~/firebase.json
```

### Step 4: Install Dependencies

```bash
# Install Flutter dependencies
flutter pub get

# Install Ruby gems (including Fastlane)
cd android
bundle install
cd ..
```

### Step 5: Build with Fastlane

```bash
# Navigate to Android directory
cd android

# Run the full build and deploy process
bundle exec fastlane build_and_deploy

# OR run steps separately:
# Build only (creates AAB)
bundle exec fastlane build

# Deploy only (requires AAB path from build output)
bundle exec fastlane deploy path:/path/to/app.aab
```

### Step 6: Monitor Upload

Watch the terminal output. If successful, you'll see:
```
Successfully uploaded app bundle to Google Play
```

Check Google Play Console Internal Testing track for the new build.

---

## Method 2: Using Gradle Directly (No Fastlane)

Use this method if you prefer building with Gradle directly or Fastlane has issues.

### Step 1: Set Up Environment Variables

Same as Method 1, Step 1 & 2.

```bash
source .env.android.local
```

### Step 2: Set Up Required Files

```bash
# Place keystore in home directory
cp /path/to/android_keystore_release.keystore ~/keystore

# Place Google Services config
cp /path/to/google-services.json android/app/google-services.json
```

### Step 3: Install Dependencies

```bash
flutter pub get
```

### Step 4: Generate Code Files

```bash
# Run build_runner to generate required code
dart run build_runner build --delete-conflicting-outputs
```

### Step 5: Build AAB with Gradle

```bash
cd android

# Build production release AAB
./gradlew bundleProductionRelease

# OR for beta builds
./gradlew bundleBetaRelease
```

**Note**: Gradle will automatically use the signing configuration from environment variables.

### Step 6: Locate the AAB

The AAB will be located at:
```
build/app/outputs/bundle/productionRelease/app-production-release.aab
```

### Step 7: Upload to Google Play

#### Option A: Using Google Play Console Web Interface
1. Go to https://play.google.com/console
2. Select your app
3. Navigate to Release → Internal testing
4. Click "Create new release"
5. Upload the AAB file
6. Add release notes
7. Review and roll out

#### Option B: Using Fastlane (if you have google_key.json)
```bash
# Place Google Play API key
cp /path/to/google_app_developer_api_key.json ~/google_key.json

# Upload with Fastlane
cd android
bundle exec fastlane supply \
  --aab ../build/app/outputs/bundle/productionRelease/app-production-release.aab \
  --track internal \
  --json_key ~/google_key.json \
  --package_name uk.org.cruising.captainsmate
```

---

## Method 3: Using Flutter Build Command

Use this for maximum control over the build process.

### Step 1: Set Up Environment Variables

Same as Method 1, Step 1 & 2.

```bash
source .env.android.local
```

### Step 2: Set Up Required Files

```bash
# Place keystore in home directory
cp /path/to/android_keystore_release.keystore ~/keystore

# Place Google Services config
cp /path/to/google-services.json android/app/google-services.json
```

### Step 3: Install Dependencies

```bash
flutter pub get
```

### Step 4: Generate Code Files

```bash
dart run build_runner build --delete-conflicting-outputs
```

### Step 5: Build AAB with Flutter

```bash
flutter build appbundle \
  --release \
  --flavor production \
  --dart-define=OAUTH_CLIENT_ID="$DEFINE_OAUTH_CLIENT_ID" \
  --dart-define=OAUTH_SCOPES="$DEFINE_OAUTH_SCOPES" \
  --dart-define=OAUTH_AUTH_ENDPOINT="$DEFINE_OAUTH_AUTH_ENDPOINT" \
  --dart-define=OAUTH_TOKEN_ENDPOINT="$DEFINE_OAUTH_TOKEN_ENDPOINT" \
  --dart-define=OAUTH_USER_ENDPOINT="$DEFINE_OAUTH_USER_ENDPOINT" \
  --dart-define=API_BASE_URL="$DEFINE_API_BASE_URL" \
  --dart-define=STAGING_API_BASE_URL="$DEFINE_STAGING_API_BASE_URL" \
  --dart-define=SENTRY_DSN="$DEFINE_SENTRY_DSN" \
  --dart-define=FIREBASE_ANDROID_APP_ID="$DEFINE_FIREBASE_ANDROID_APP_ID" \
  --dart-define=FIREBASE_ANDROID_API_KEY="$DEFINE_FIREBASE_ANDROID_API_KEY" \
  --dart-define=FIREBASE_SENDER_ID="$DEFINE_FIREBASE_SENDER_ID" \
  --dart-define=FIREBASE_BUCKET_NAME="$DEFINE_FIREBASE_BUCKET_NAME" \
  --dart-define=FIREBASE_PROJECT_ID="$DEFINE_FIREBASE_PROJECT_ID" \
  --dart-define=BACKGROUND_LOCATION_API_KEY="$BACKGROUND_LOCATION_API_KEY"
```

### Step 6: Upload to Google Play

Follow Step 7 from Method 2 (upload via web console or Fastlane).

---

## Method 4: Using key.properties File (Local Development)

For repeated local builds, using a `key.properties` file is more convenient than environment variables.

### Step 1: Create key.properties File

```bash
nano android/key.properties
```

Paste this content (fill in actual values):

```properties
storeFile=/Users/yourusername/keystore
storePassword=YOUR_KEYSTORE_PASSWORD
keyAlias=YOUR_KEY_ALIAS
keyPassword=YOUR_KEY_PASSWORD
```

Save and exit.

**⚠️ SECURITY**: Never commit this file to git! It's already in `.gitignore`.

### Step 2: Verify .gitignore

```bash
# Ensure key.properties is in .gitignore
grep "key.properties" android/.gitignore
```

### Step 3: Set Up Other Files

```bash
# Place Google Services config
cp /path/to/google-services.json android/app/google-services.json
```

### Step 4: Set Required Environment Variables

You still need these for the build:

```bash
export ANDROID_APP_IDENTIFIER="uk.org.cruising.captainsmate"
export FLUTTER_FLAVOR="production"
export BACKGROUND_LOCATION_API_KEY="YOUR_VALUE"

# All DEFINE_* variables
export DEFINE_OAUTH_CLIENT_ID="YOUR_VALUE"
# ... (all the others)
```

### Step 5: Build with Gradle

```bash
cd android
./gradlew bundleProductionRelease
```

Gradle will automatically read signing credentials from `key.properties`.

### Step 6: Upload to Google Play

Follow upload instructions from Method 2.

---

## Troubleshooting Common Issues

### Issue: "Keystore file not found"

**Solution:**
```bash
# Verify keystore location
ls -la ~/keystore

# If using key.properties, check path
cat android/key.properties | grep storeFile
```

### Issue: "Wrong password for keystore"

**Solution:**
Double-check your keystore password and key password. They might be different.

```bash
# Verify environment variables
echo $ANDROID_KEYSTORE_PASSWORD
echo $ANDROID_KEYSTORE_KEY_PASSWORD

# OR check key.properties
cat android/key.properties
```

### Issue: "google-services.json not found"

**Solution:**
Ensure the file is in the correct location:
```bash
ls -la android/app/google-services.json
```

### Issue: "Version code XYZ already exists"

**Solution:**
You need to increment the version code. Edit `pubspec.yaml`:

```yaml
version: 1.2.3+45  # Change the number after + (version code)
```

Then rebuild.

### Issue: "Google Play API authentication failed"

**Solution:**
1. Verify `google_key.json` exists: `ls ~/google_key.json`
2. Check service account has permissions in Play Console
3. Ensure service account email matches the one in `google_key.json`

### Issue: "Signing config missing"

**Solution:**
Ensure either:
- Environment variables are set (check with `echo $ANDROID_KEYSTORE_ALIAS`)
- OR `key.properties` file exists (check with `ls android/key.properties`)

### Issue: "Build fails with 'ANDROID_APP_IDENTIFIER not set'"

**Solution:**
```bash
# Set the environment variable
export ANDROID_APP_IDENTIFIER="uk.org.cruising.captainsmate"

# Verify it's set
echo $ANDROID_APP_IDENTIFIER
```

### Issue: "Gradle daemon fails to start"

**Solution:**
```bash
cd android
./gradlew --stop
./gradlew clean
./gradlew bundleProductionRelease
```

### Issue: "MultiDex issues"

**Solution:**
Already configured in the project. If you see MultiDex errors, ensure you're targeting minSdkVersion 24 or higher.

### Issue: "Flutter command not found"

**Solution:**
```bash
# Add Flutter to PATH (adjust path to your Flutter installation)
export PATH="$PATH:/path/to/flutter/bin"

# Verify
flutter --version
```

---

## Building for Different Tracks

### Internal Testing (Default)
```bash
cd android
bundle exec fastlane build_and_deploy
```

Uploads to internal testing track automatically.

### Beta Testing
To upload to beta track instead:

```bash
cd android

# Build
bundle exec fastlane build

# Deploy to beta track
bundle exec fastlane supply \
  --aab /path/to/app.aab \
  --track beta \
  --json_key ~/google_key.json \
  --package_name uk.org.cruising.captainsmate
```

### Production Release
```bash
cd android

# Build
bundle exec fastlane build

# Deploy to production track
bundle exec fastlane supply \
  --aab /path/to/app.aab \
  --track production \
  --json_key ~/google_key.json \
  --package_name uk.org.cruising.captainsmate
```

**⚠️ WARNING**: Always test in internal/beta tracks before production!

---

## Quick Reference: Build Commands

### Full automated build with Fastlane:
```bash
source .env.android.local
cd android
bundle exec fastlane build_and_deploy
```

### Build only (no deploy):
```bash
source .env.android.local
cd android
bundle exec fastlane build
```

### Gradle build directly:
```bash
source .env.android.local
cd android
./gradlew bundleProductionRelease
```

### Flutter build directly:
```bash
source .env.android.local
dart run build_runner build --delete-conflicting-outputs
flutter build appbundle --release --flavor production \
  --dart-define=OAUTH_CLIENT_ID="$DEFINE_OAUTH_CLIENT_ID" \
  # ... (add all other dart-defines)
```

### Upload existing AAB:
```bash
cd android
bundle exec fastlane supply \
  --aab /path/to/app.aab \
  --track internal \
  --json_key ~/google_key.json \
  --package_name uk.org.cruising.captainsmate
```

### Clean build:
```bash
cd android
./gradlew clean
flutter clean
flutter pub get
```

---

## Building Beta vs Production Flavors

### Production Flavor (default):
```bash
export FLUTTER_FLAVOR="production"
flutter build appbundle --release --flavor production
```

AAB location: `build/app/outputs/bundle/productionRelease/app-production-release.aab`

### Beta Flavor:
```bash
export FLUTTER_FLAVOR="beta"
flutter build appbundle --release --flavor beta
```

AAB location: `build/app/outputs/bundle/betaRelease/app-beta-release.aab`

---

## Post-Build Verification

After successful build and upload:

### 1. Check Google Play Console
- Log into https://play.google.com/console
- Navigate to your app
- Go to Release → Testing → Internal testing
- Verify new build appears
- Check build status (processing/available for testing)

### 2. Check Version Information
- Version name: from `pubspec.yaml`
- Version code: auto-incremented or manually set
- Confirm they're correct in Play Console

### 3. Test the Build
- Add yourself as internal tester (if not already)
- Install via Play Store on device
- Verify app launches
- Test core functionality
- Check for crashes or errors

### 4. Review Build Details
- Check APK/AAB size
- Review supported devices
- Verify minimum API level
- Check for any warnings

### 5. Notify Team
- Inform team new build is available
- Provide build number and release notes
- Request testing feedback

---

## Version Management

### Version Name
Located in `pubspec.yaml`:
```yaml
version: 1.2.3+45
         ^^^^^  ^^
         name   code
```

- **Version Name** (1.2.3): User-facing version
- **Version Code** (+45): Internal build number (must always increment)

### Auto-Increment Version Code
Fastlane automatically increments version code based on Play Console internal track:

```bash
cd android
bundle exec fastlane build_and_deploy
# Automatically gets latest version code from Play Store and increments
```

### Manual Version Code Update
Edit `pubspec.yaml`:
```yaml
version: 1.2.3+46  # Increment the number after +
```

---

## Tips for Success

1. **First Build**: Expect it to take 10-20 minutes. Gradle downloads dependencies on first run.

2. **Keep Environment File Secure**: Never commit `.env.android.local` to git:
   ```bash
   echo ".env.android.local" >> .gitignore
   ```

3. **Backup Your Keystore**: The keystore is irreplaceable. Store secure backups:
   ```bash
   # Create encrypted backup
   cp ~/keystore ~/keystore.backup
   # Store in multiple secure locations
   ```

4. **Test Incrementally**: Try each method in order. Start with Fastlane if possible.

5. **Build Regularly**: Don't wait until emergency. Practice builds help identify issues early.

6. **Monitor Build Logs**: Save successful build logs for reference when troubleshooting.

7. **Verify Signing**: After first successful build, verify the signing certificate:
   ```bash
   cd android
   ./gradlew signingReport
   ```

8. **Clean Builds**: If you encounter weird issues, try a clean build:
   ```bash
   flutter clean
   cd android
   ./gradlew clean
   cd ..
   flutter pub get
   ```

---

## Build Performance Tips

### Speed Up Gradle Builds

Edit `android/gradle.properties` (already configured):
```properties
org.gradle.jvmargs=-Xmx4G
org.gradle.parallel=true
org.gradle.caching=true
```

### Speed Up Flutter Builds
```bash
# Use build cache
flutter build appbundle --release --flavor production --build-shared-library
```

### Parallel Builds
Gradle automatically uses parallel builds when possible. Monitor with:
```bash
./gradlew bundleProductionRelease --info
```

---

## Emergency Contact Information

If you get completely stuck:

- **Google Play Support**: https://support.google.com/googleplay/android-developer/
- **Fastlane Documentation**: https://docs.fastlane.tools/
- **Flutter Android Deployment**: https://docs.flutter.dev/deployment/android
- **Gradle Documentation**: https://docs.gradle.org/

---

## Checklist: First Build

Use this checklist for your first manual build:

- [ ] Environment variables file created and sourced
- [ ] Keystore file copied to `~/keystore`
- [ ] Keystore credentials verified
- [ ] `google-services.json` in `android/app/`
- [ ] `google_key.json` in `~/ ` (for upload)
- [ ] Flutter dependencies installed (`flutter pub get`)
- [ ] Ruby gems installed (`cd android && bundle install`)
- [ ] Build command executed
- [ ] Build completed successfully
- [ ] AAB file located in output directory
- [ ] AAB uploaded to Play Console
- [ ] Build appears in Internal Testing track
- [ ] Build installed on test device
- [ ] App launches successfully
- [ ] Core functionality tested

---

## Next Steps After First Successful Build

- [ ] Document any issues you encountered
- [ ] Update this guide with your findings
- [ ] Create backup of keystore (if not already done)
- [ ] Set up regular build testing schedule
- [ ] Consider setting up CI/CD access for easier future releases
- [ ] Review and optimize build time
- [ ] Set up release notes template
- [ ] Configure staged rollout strategy

---

## Advanced: Building APK for Testing

For direct device installation (not Play Store):

```bash
# Build production APK
flutter build apk --release --flavor production \
  --dart-define=OAUTH_CLIENT_ID="$DEFINE_OAUTH_CLIENT_ID" \
  # ... (add all other dart-defines)

# OR build beta APK
flutter build apk --release --flavor beta \
  --dart-define=OAUTH_CLIENT_ID="$DEFINE_OAUTH_CLIENT_ID" \
  # ... (add all other dart-defines)
```

APK location: `build/app/outputs/flutter-apk/app-production-release.apk`

Install on device:
```bash
# Via ADB
adb install build/app/outputs/flutter-apk/app-production-release.apk

# OR transfer file to device and install manually
```

**Note**: Play Store requires AAB (Android App Bundle), not APK. APKs are only for direct testing.

---

*Guide Created: 2026-01-07*
*Last Updated: 2026-01-07*