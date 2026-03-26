---
id: 25e46320-b4ea-4936-a7be-c7ac52fb6379
blueprint: captains_mate_app_43
title: 'Build Troubleshooting Guide'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774536438
---
# Build & Deployment Troubleshooting Guide

## Overview
This comprehensive troubleshooting guide covers common build and deployment issues, including critical scenarios where you're missing access to essential resources like S3, Match passwords, or keystores.

---

## Table of Contents
1. [Missing Critical Resources](#missing-critical-resources)
2. [iOS Build Issues](#ios-build-issues)
3. [Android Build Issues](#android-build-issues)
4. [Firebase Configuration Issues](#firebase-configuration-issues)
5. [Environment Variable Issues](#environment-variable-issues)
6. [Fastlane Issues](#fastlane-issues)
7. [Flutter Build Issues](#flutter-build-issues)
8. [Deployment Issues](#deployment-issues)
9. [Gradle Issues](#gradle-issues)
10. [Recovery Procedures](#recovery-procedures)

---

## Missing Critical Resources

### 🔴 CRITICAL: No Access to S3 Bucket

**Impact**: Cannot download signing materials, API keys, or Firebase configs.

#### Scenario A: Temporary S3 Access Loss

**Workaround:**
1. Contact previous developer or infrastructure admin immediately
2. Request manual export of all required files:
   - **iOS**: `apple_app_developer_api_key.p8`, `GoogleService-Info.plist`, Match certificates
   - **Android**: `android_keystore_release.keystore`, `google_app_developer_api_key.json`, `google-services.json`

3. Store files locally in secure locations
4. Follow manual build procedures from quick-start guides

**Files to Request:**
```
iOS Files:
- apple_app_developer_api_key.p8
- GoogleService-Info.plist
- firebase.json (if using Firebase App Distribution)

Android Files:
- android_keystore_release.keystore
- google_app_developer_api_key.json
- google-services.json
- firebase.json (if using Firebase App Distribution)
```

#### Scenario B: Permanent S3 Access Loss / No Recovery Possible

**If you cannot get files from S3 or previous developer:**

##### For iOS:
1. **Generate new certificates via Apple Developer Portal** (ONLY if you have Apple Developer account access)
2. **Will NOT work if using Match** - Match certificates stored in S3 cannot be recreated identically
3. **You'll need to set up new signing** - see [Recovery: iOS Signing from Scratch](#recovery-ios-signing-from-scratch)

##### For Android:
1. **If keystore cannot be recovered** - see [Recovery: Lost Android Keystore](#recovery-lost-android-keystore)
2. **This is CRITICAL** - Without keystore, you cannot update the existing app

#### Scenario C: Can Access S3 But Don't Have AWS Credentials

**Solutions:**

**Option 1: Request AWS Credentials**
Contact infrastructure admin for:
- AWS Access Key ID
- AWS Secret Access Key
- Region: `eu-west-2`
- Bucket permissions: Read access to `tca-secure-assets` and `tca-app-certificates`

**Option 2: Use AWS CLI to Download Files Manually**

If someone can give you temporary access:
```bash
# Configure AWS CLI
aws configure
# Enter: Access Key ID, Secret Access Key, Region (eu-west-2)

# Download iOS files
aws s3 cp s3://tca-secure-assets/apple_app_developer_api_key.p8 ~/apple_key.p8
aws s3 cp s3://tca-secure-assets/GoogleService-Info.plist ios/Runner/GoogleService-Info.plist
aws s3 cp s3://tca-secure-assets/firebase.json ~/firebase.json

# Download Android files
aws s3 cp s3://tca-secure-assets/android_keystore_release.keystore ~/keystore
aws s3 cp s3://tca-secure-assets/google_app_developer_api_key.json ~/google_key.json
aws s3 cp s3://tca-secure-assets/google-services.json android/app/google-services.json
```

**Option 3: Use Web Console**
1. Log into AWS Console: https://console.aws.amazon.com
2. Navigate to S3
3. Select bucket `tca-secure-assets`
4. Download required files manually
5. Place in correct locations per quick-start guides

---

### 🔴 CRITICAL: No Match Password (iOS)

**Impact**: Cannot decrypt iOS certificates stored in S3 via Match.

#### Scenario A: Match Password Lost/Unknown

**Symptoms:**
```
[!] Error decrypting repo
[!] The passphrase you entered is wrong
```

**Solutions in Order of Preference:**

**Option 1: Recover Password from Previous Developer**
- Check password managers (1Password, LastPass, etc.)
- Check secure notes or documentation
- Check GitHub Secrets (if they have access)

**Option 2: Recover Password from GitHub Secrets**
If you have GitHub admin access:
1. Go to repository Settings → Secrets and Variables → Actions
2. Look for `MATCH_PASSWORD`
3. **Note**: You can only see if it exists, not the value itself
4. If someone has access, they can view and share it

**Option 3: Reset Match and Generate New Certificates**

⚠️ **WARNING**: This invalidates all existing certificates.

```bash
# Backup current certificates first (if you can download them)
cd ios

# Nuke existing Match setup (DESTRUCTIVE)
bundle exec fastlane match nuke development
bundle exec fastlane match nuke distribution
bundle exec fastlane match nuke appstore

# Generate new certificates with new password
# You'll need Apple Developer account access
export MATCH_PASSWORD="your_new_strong_password"
bundle exec fastlane match appstore --force_for_new_devices

# Update GitHub Secret with new password
# Settings → Secrets → Update MATCH_PASSWORD
```

**What This Breaks:**
- All devices with old provisioning profiles need new ones
- All CI/CD builds need new MATCH_PASSWORD
- All other developers need to run match again

#### Scenario B: Cannot Access Match at All (No S3 + No Password)

**Solution**: Set up manual signing - see [Recovery: iOS Signing from Scratch](#recovery-ios-signing-from-scratch)

---

### 🔴 CRITICAL: Lost Android Keystore

**Impact**: **CANNOT UPDATE APP** - This is the worst-case scenario for Android.

#### Scenario A: Keystore File Lost/Corrupted

**Symptoms:**
```
Keystore file not found
Keystore was tampered with, or password was incorrect
```

**Recovery Options:**

**Option 1: Recover from Backups**
Check all possible backup locations:
- Previous developer's machine
- S3 bucket backup
- Company backup systems
- CI/CD server
- Password manager file attachments
- Email archives (if keystore was ever shared via email - not recommended but check)

**Verification Command:**
```bash
# Verify keystore integrity
keytool -list -v -keystore ~/keystore -storepass YOUR_PASSWORD
```

**Option 2: Check if Play App Signing is Enabled**

This is your ONLY hope if keystore is truly lost:

1. Log into Google Play Console
2. Go to Release → Setup → App signing
3. Check if "Google Play App Signing" is enabled

**If Play App Signing is ENABLED:**
✅ **You're safe!** Google manages the actual signing key.
- Your upload keystore is replaceable
- You can generate a new upload keystore
- Contact Google Play Support to reset upload key

**Steps to Generate New Upload Key:**
```bash
# Generate new keystore
keytool -genkey -v \
  -keystore upload-keystore.jks \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -alias upload \
  -storetype JKS

# Generate PEM certificate for Google
keytool -export -rfc \
  -keystore upload-keystore.jks \
  -alias upload \
  -file upload_certificate.pem

# Submit to Google Play Console
# Release → Setup → App signing → Upload new key
```

**If Play App Signing is NOT ENABLED:**
❌ **You cannot update the app.**

See [Recovery: Lost Android Keystore - No Play App Signing](#recovery-lost-android-keystore---no-play-app-signing)

#### Scenario B: Have Keystore But Lost Passwords

**Symptoms:**
```
keystore password was incorrect
Cannot recover key
```

**Recovery Options:**

**Option 1: Password Recovery Attempts**
Check these locations:
- Password managers
- GitHub Secrets: `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEYSTORE_KEY_PASSWORD`
- Previous developer's notes
- `android/key.properties` file (if it exists and wasn't gitignored properly)

**Option 2: Brute Force (If you have partial knowledge)**
If you know password patterns:
```bash
# Use a tool like John the Ripper (ONLY for your own keystore)
# This is time-consuming and may not work
```

**Option 3: Check Keystore Alias**
Maybe just the alias is wrong:
```bash
# List all aliases in keystore (requires store password)
keytool -list -v -keystore ~/keystore
```

**If All Fails:**
Same as lost keystore - check if Play App Signing is enabled.

---

### 🔴 No Access to Apple Developer Portal

**Impact**: Cannot create/renew certificates, provision profiles, or upload to TestFlight.

#### Scenario A: Awaiting Account Access

**Temporary Workaround:**
- Request previous developer to export certificates
- Use exported `.p12` certificate file
- Use exported `.mobileprovision` file
- Can build locally but cannot upload to TestFlight without API key

**What You Can Still Do:**
```bash
# Build IPA locally
flutter build ipa --release --flavor production

# Archive in Xcode
# Open ios/Runner.xcworkspace in Xcode
# Product → Archive
```

**What You Cannot Do:**
- Upload to TestFlight
- Create new certificates
- Renew expired provisioning profiles

#### Scenario B: Account Exists But No Admin Access

**Request these permissions:**
- App Manager or Admin role
- Access to Certificates, Identifiers & Profiles
- Access to App Store Connect

**Temporary Solution:**
Ask someone with access to:
1. Export current certificates
2. Generate App Store Connect API key for you
3. Add you as internal tester for builds

---

### 🔴 No Access to Google Play Console

**Impact**: Cannot upload builds, manage releases, or view app details.

#### Scenario A: Awaiting Account Access

**Request these permissions:**
- Release Manager role (minimum)
- Admin role (preferred)
- Access to app: `uk.org.cruising.captainsmate`

**What You Can Still Do:**
```bash
# Build AAB locally
flutter build appbundle --release --flavor production
```

**What You Cannot Do:**
- Upload to Play Store
- View current version codes
- Manage testers
- Submit for review

#### Scenario B: Have Service Account But Not Personal Account

**If you have `google_key.json`:**
```bash
# You can upload via Fastlane/CLI
cd android
bundle exec fastlane deploy path:/path/to/app.aab
```

**But you cannot:**
- Use Play Console web interface
- Manually manage releases
- View analytics
- Respond to reviews

**Solution:** Request personal account access for full management.

---

## iOS Build Issues

### Issue: "No profiles for 'uk.org.cruising.captainsmate' were found"

**Cause**: Provisioning profile not installed or expired.

**Solutions:**

**Solution 1: Download via Match**
```bash
cd ios
bundle exec fastlane match appstore --readonly
```

**Solution 2: Download from Apple Developer Portal**
1. Go to https://developer.apple.com
2. Certificates, Identifiers & Profiles → Profiles
3. Download App Store profile for your bundle ID
4. Double-click to install

**Solution 3: Create New Profile**
```bash
cd ios
bundle exec fastlane match appstore --force_for_new_devices
```

---

### Issue: "Code signing certificate not valid"

**Cause**: Certificate expired or not installed.

**Check Certificate:**
```bash
# List installed certificates
security find-identity -v -p codesigning
```

**Solutions:**

**Solution 1: Install from Match**
```bash
cd ios
bundle exec fastlane match appstore
```

**Solution 2: Import Certificate**
```bash
# Import .p12 file
security import certificate.p12 -k ~/Library/Keychains/login.keychain-db
```

**Solution 3: Download from Apple Developer Portal**
1. Go to Certificates section
2. Download Distribution certificate
3. Double-click to install

---

### Issue: "Apple API authentication failed"

**Cause**: Missing or invalid API key.

**Check API Key:**
```bash
ls ~/apple_key.p8
```

**Verify Environment Variables:**
```bash
echo $APPLE_APP_DEVELOPER_KEY_ID
echo $APPLE_APP_DEVELOPER_ISSUER_ID
```

**Solutions:**

**Solution 1: Download API Key from S3**
```bash
aws s3 cp s3://tca-secure-assets/apple_app_developer_api_key.p8 ~/apple_key.p8
```

**Solution 2: Generate New API Key**
1. Go to App Store Connect
2. Users and Access → Keys
3. Generate new key with App Manager access
4. Download `.p8` file
5. Note Key ID and Issuer ID
6. Update environment variables

---

### Issue: "Build number already exists in TestFlight"

**Cause**: Version code conflict.

**Solution 1: Let Fastlane Auto-Increment**
```bash
cd ios
bundle exec fastlane build
# Fastlane fetches latest build number and increments
```

**Solution 2: Manual Increment in Xcode**
1. Open `ios/Runner.xcworkspace`
2. Select Runner target
3. General tab → Build field
4. Increment number
5. Rebuild

---

### Issue: "Keychain access denied"

**Cause**: Certificates locked in keychain.

**Solution:**
```bash
# Unlock keychain
security unlock-keychain ~/Library/Keychains/login.keychain-db

# OR set keychain to not lock
security set-keychain-settings -lut 21600 ~/Library/Keychains/login.keychain-db
```

---

### Issue: "Pod install fails"

**Cause**: CocoaPods cache or outdated specs.

**Solutions:**
```bash
cd ios

# Update CocoaPods
sudo gem install cocoapods

# Update specs
pod repo update

# Clean and reinstall
rm -rf Pods Podfile.lock
pod install --repo-update
```

---

## Android Build Issues

### Issue: "Keystore not found"

**Cause**: Keystore file missing or wrong path.

**Check Keystore Location:**
```bash
ls -la ~/keystore
```

**Solutions:**

**Solution 1: Download from S3**
```bash
aws s3 cp s3://tca-secure-assets/android_keystore_release.keystore ~/keystore
```

**Solution 2: Check key.properties Path**
```bash
cat android/key.properties | grep storeFile
# Ensure path is correct
```

**Solution 3: Set Environment Variable**
```bash
export HOME=/Users/yourusername  # Adjust to your home path
# Gradle looks for $HOME/keystore
```

---

### Issue: "Wrong password for keystore"

**Cause**: Incorrect keystore or key password.

**Verify Passwords:**
```bash
# Test keystore password
keytool -list -v -keystore ~/keystore -storepass YOUR_STORE_PASSWORD

# If this works, store password is correct
# Then test key password with actual alias
```

**Solutions:**

**Solution 1: Check GitHub Secrets**
- `ANDROID_KEYSTORE_PASSWORD` (store password)
- `ANDROID_KEYSTORE_KEY_PASSWORD` (key password)

**Solution 2: Try Same Password for Both**
Sometimes they're the same:
```bash
export ANDROID_KEYSTORE_PASSWORD="password123"
export ANDROID_KEYSTORE_KEY_PASSWORD="password123"
```

**Solution 3: Check key.properties**
```bash
cat android/key.properties
# Compare values
```

---

### Issue: "Version code XYZ already exists"

**Cause**: Version code conflict in Play Console.

**Check Current Version Code:**
```bash
# In pubspec.yaml
cat pubspec.yaml | grep version
# Shows: version: 1.2.3+45 (45 is version code)
```

**Solutions:**

**Solution 1: Let Fastlane Auto-Increment**
```bash
cd android
bundle exec fastlane build
# Fastlane fetches latest from Play Store and increments
```

**Solution 2: Manual Increment**
Edit `pubspec.yaml`:
```yaml
version: 1.2.3+46  # Increment number after +
```

**Solution 3: Check All Tracks**
Log into Play Console and check version codes in:
- Internal testing
- Closed testing
- Open testing
- Production

Version code must be higher than all tracks.

---

### Issue: "google-services.json not found"

**Cause**: Firebase config file missing.

**Check File:**
```bash
ls -la android/app/google-services.json
```

**Solutions:**

**Solution 1: Download from S3**
```bash
aws s3 cp s3://tca-secure-assets/google-services.json android/app/google-services.json
```

**Solution 2: Download from Firebase Console**
1. Go to https://console.firebase.google.com
2. Select project
3. Project Settings → General
4. Your apps → Android app
5. Download `google-services.json`
6. Place in `android/app/`

---

### Issue: "Google Play API authentication failed"

**Cause**: Missing or invalid service account key.

**Check API Key:**
```bash
ls ~/google_key.json
cat ~/google_key.json | grep client_email
```

**Solutions:**

**Solution 1: Download from S3**
```bash
aws s3 cp s3://tca-secure-assets/google_app_developer_api_key.json ~/google_key.json
```

**Solution 2: Generate New Service Account Key**
1. Go to Google Cloud Console
2. Select Firebase project
3. IAM & Admin → Service Accounts
4. Find service account or create new one
5. Create new key (JSON)
6. Download and save as `~/google_key.json`

**Solution 3: Check Permissions**
In Google Play Console:
1. Setup → API access
2. Find service account
3. Ensure it has "Release Manager" or "Admin" role

---

### Issue: "Execution failed for task ':app:signReleaseBundle'"

**Cause**: Signing configuration error.

**Check Signing Config:**
```bash
cd android
./gradlew signingReport
```

**Solutions:**

**Solution 1: Verify Environment Variables**
```bash
echo $ANDROID_KEYSTORE_ALIAS
echo $ANDROID_KEYSTORE_PASSWORD
echo $ANDROID_KEYSTORE_KEY_PASSWORD
ls ~/keystore
```

**Solution 2: Check app/build.gradle**
Verify signing config section reads environment variables:
```gradle
signingConfigs {
    release {
        keyAlias = System.getenv('ANDROID_KEYSTORE_ALIAS')
        keyPassword = System.getenv('ANDROID_KEYSTORE_KEY_PASSWORD')
        storeFile = file(System.getenv('HOME') + '/keystore')
        storePassword = System.getenv('ANDROID_KEYSTORE_PASSWORD')
    }
}
```

---

### Issue: "Multidex configuration error"

**Cause**: App exceeds 65K method limit without proper multidex config.

**Already Configured:**
The project already has multidex enabled in `android/app/build.gradle`:
```gradle
multiDexEnabled true
```

**If Still Failing:**
```bash
# Clean and rebuild
cd android
./gradlew clean
./gradlew bundleProductionRelease
```

---

## Firebase Configuration Issues

### Issue: "Firebase not initialized"

**Cause**: Firebase configuration files missing or incorrect.

**Check Files:**
```bash
# iOS
ls -la ios/Runner/GoogleService-Info.plist

# Android
ls -la android/app/google-services.json
```

**Solutions:**

**Solution 1: Download from S3**
```bash
# iOS
aws s3 cp s3://tca-secure-assets/GoogleService-Info.plist ios/Runner/GoogleService-Info.plist

# Android
aws s3 cp s3://tca-secure-assets/google-services.json android/app/google-services.json
```

**Solution 2: Download from Firebase Console**
See [Firebase Configuration Files Missing](#firebase-configuration-files-missing)

---

### Issue: Firebase configuration files missing

**Download from Firebase Console:**

**For iOS:**
1. Go to https://console.firebase.google.com
2. Select your project
3. Project Settings → General
4. Your apps → iOS app (`uk.org.cruising.captainsmate`)
5. Download `GoogleService-Info.plist`
6. Place in `ios/Runner/GoogleService-Info.plist`

**For Android:**
1. Same console
2. Your apps → Android app (`uk.org.cruising.captainsmate`)
3. Download `google-services.json`
4. Place in `android/app/google-services.json`

---

### Issue: "Firebase dart-define values incorrect"

**Verify Required Variables:**

**iOS Firebase Variables:**
```bash
echo $DEFINE_FIREBASE_IOS_APP_ID
echo $DEFINE_FIREBASE_IOS_API_KEY
echo $DEFINE_FIREBASE_IOS_BUNDLE_ID
echo $DEFINE_FIREBASE_SENDER_ID
echo $DEFINE_FIREBASE_BUCKET_NAME
echo $DEFINE_FIREBASE_PROJECT_ID
```

**Android Firebase Variables:**
```bash
echo $DEFINE_FIREBASE_ANDROID_APP_ID
echo $DEFINE_FIREBASE_ANDROID_API_KEY
echo $DEFINE_FIREBASE_SENDER_ID
echo $DEFINE_FIREBASE_BUCKET_NAME
echo $DEFINE_FIREBASE_PROJECT_ID
```

**Get Values from Config Files:**

**From GoogleService-Info.plist (iOS):**
```bash
# On macOS
/usr/libexec/PlistBuddy -c "Print :GOOGLE_APP_ID" ios/Runner/GoogleService-Info.plist
```

**From google-services.json (Android):**
```bash
cat android/app/google-services.json | grep client_id
cat android/app/google-services.json | grep api_key
```

---

## Environment Variable Issues

### Issue: "Environment variable not set"

**Symptoms:**
```
Error: ANDROID_APP_IDENTIFIER not set
Error: APPLE_APP_IDENTIFIER not set
```

**Check Variables:**
```bash
# List all environment variables
env | grep DEFINE_
env | grep ANDROID_
env | grep APPLE_
```

**Solutions:**

**Solution 1: Source Environment File**
```bash
source .env.ios.local
# OR
source .env.android.local
```

**Solution 2: Export Variables**
```bash
export ANDROID_APP_IDENTIFIER="uk.org.cruising.captainsmate"
export APPLE_APP_IDENTIFIER="uk.org.cruising.captainsmate"
# ... export all others
```

**Solution 3: Create Environment File**
Follow templates in quick-start guides.

---

### Issue: "dart-define values not working"

**Cause**: Variables not passed to Flutter build command.

**Verify in Build Command:**
```bash
flutter build appbundle --release --flavor production \
  --dart-define=OAUTH_CLIENT_ID="$DEFINE_OAUTH_CLIENT_ID" \
  --dart-define=API_BASE_URL="$DEFINE_API_BASE_URL"
  # ... (ensure ALL variables are passed)
```

**Check Value Passed:**
```bash
# Verify variable has value
echo $DEFINE_OAUTH_CLIENT_ID
# Should show actual value, not empty
```

---

## Fastlane Issues

### Issue: "Fastlane command not found"

**Cause**: Ruby gems not installed.

**Solutions:**
```bash
# Install bundler
gem install bundler

# Install fastlane via bundler
cd ios  # or android
bundle install

# Run with bundle exec
bundle exec fastlane build
```

---

### Issue: "Fastlane plugin flutter not found"

**Cause**: Flutter plugin not installed.

**Solution:**
```bash
cd ios  # or android

# Check Pluginfile
cat fastlane/Pluginfile
# Should show: plugin 'fastlane-plugin-flutter'

# Install plugins
bundle exec fastlane install_plugins

# OR manually
bundle exec fastlane add_plugin flutter
```

---

### Issue: "Fastlane match failed"

**Symptoms:**
```
[!] Error decrypting repo
[!] Could not decrypt the repo
```

**Cause**: Wrong Match password or S3 access issues.

**Solutions:**

**Solution 1: Verify Match Password**
```bash
echo $MATCH_PASSWORD
# Should show password, not empty
```

**Solution 2: Verify S3 Access**
```bash
echo $AWS_ACCESS_KEY_ID
echo $AWS_SECRET_ACCESS_KEY

# Test S3 access
aws s3 ls s3://tca-app-certificates/
```

**Solution 3: Use Match in Readonly Mode**
```bash
cd ios
bundle exec fastlane match appstore --readonly
# Only downloads, doesn't create new certificates
```

---

## Flutter Build Issues

### Issue: "Flutter command not found"

**Cause**: Flutter not in PATH.

**Solutions:**
```bash
# Add to PATH (adjust path to your Flutter installation)
export PATH="$PATH:/path/to/flutter/bin"

# Verify
flutter --version

# Make permanent (add to ~/.zshrc or ~/.bashrc)
echo 'export PATH="$PATH:/path/to/flutter/bin"' >> ~/.zshrc
source ~/.zshrc
```

---

### Issue: "Build runner fails"

**Symptoms:**
```
[SEVERE] Error running build_runner
```

**Solutions:**
```bash
# Clean build runner cache
flutter clean

# Run with delete conflicting outputs
dart run build_runner build --delete-conflicting-outputs

# If still failing, run with verbose
dart run build_runner build --delete-conflicting-outputs --verbose
```

---

### Issue: "Flutter dependencies conflict"

**Symptoms:**
```
version solving failed
```

**Solutions:**
```bash
# Update dependencies
flutter pub upgrade

# Get dependencies fresh
rm pubspec.lock
flutter pub get

# Clean everything
flutter clean
flutter pub get
```

---

### Issue: "Flavor not found"

**Symptoms:**
```
Flavor 'production' not found
```

**Verify Flavor Configuration:**

**Android** - Check `android/app/build.gradle`:
```gradle
productFlavors {
    production {
        dimension "flavor-type"
    }
    beta {
        dimension "flavor-type"
    }
}
```

**iOS** - Check if schemes exist in Xcode:
1. Open `ios/Runner.xcworkspace`
2. Product → Scheme → Manage Schemes
3. Verify production scheme exists

**Solution:**
```bash
# Use correct flavor name (case-sensitive)
export FLUTTER_FLAVOR="production"  # Not "Production"
```

---

## Deployment Issues

### Issue: "Upload to TestFlight fails"

**Symptoms:**
```
[!] Could not upload to TestFlight
```

**Solutions:**

**Solution 1: Check API Key**
```bash
ls ~/apple_key.p8
echo $APPLE_APP_DEVELOPER_KEY_ID
echo $APPLE_APP_DEVELOPER_ISSUER_ID
```

**Solution 2: Use Xcode Organizer**
1. Open Xcode
2. Window → Organizer
3. Select archive
4. Click "Distribute App"
5. Follow prompts

**Solution 3: Use Transporter App**
1. Download Transporter from Mac App Store
2. Drag IPA file into Transporter
3. Click "Deliver"

---

### Issue: "Upload to Play Store fails"

**Symptoms:**
```
Google Play API error
```

**Solutions:**

**Solution 1: Check Service Account**
```bash
cat ~/google_key.json | grep client_email
```

**Solution 2: Verify Permissions**
1. Log into Play Console
2. Setup → API access
3. Verify service account has Release Manager role

**Solution 3: Check Package Name**
```bash
echo $ANDROID_APP_IDENTIFIER
# Must match: uk.org.cruising.captainsmate
```

**Solution 4: Upload via Web Console**
1. Go to Play Console
2. Release → Testing → Internal testing
3. Create new release
4. Upload AAB manually

---

### Issue: "Build processing stuck"

**Symptoms:**
- TestFlight: Build stuck in "Processing"
- Play Store: Build stuck in "Publishing"

**TestFlight:**
- Can take 5-60 minutes
- Check App Store Connect → TestFlight → Build Activity
- If stuck >2 hours, contact Apple Support

**Play Store:**
- Can take 30 minutes to several hours
- Check Play Console → Release dashboard
- Status: "Being reviewed" is normal
- If stuck >24 hours, check for email from Google

---

## Gradle Issues

### Issue: "Gradle daemon fails"

**Solutions:**
```bash
cd android

# Stop daemon
./gradlew --stop

# Clean
./gradlew clean

# Rebuild
./gradlew bundleProductionRelease
```

---

### Issue: "Gradle build very slow"

**Solutions:**

**Solution 1: Increase Memory**
Edit `android/gradle.properties`:
```properties
org.gradle.jvmargs=-Xmx4G
org.gradle.parallel=true
org.gradle.caching=true
```

**Solution 2: Clean Cache**
```bash
cd android
./gradlew clean cleanBuildCache
```

---

### Issue: "Gradle sync failed"

**Cause**: Dependencies or plugin version conflicts.

**Solutions:**
```bash
# Update Gradle wrapper
cd android
./gradlew wrapper --gradle-version=8.0

# Invalidate caches (if using Android Studio)
# File → Invalidate Caches → Invalidate and Restart
```

---

## Recovery Procedures

### Recovery: iOS Signing from Scratch

**When to use**: Lost Match password, no S3 access, starting fresh.

**Prerequisites:**
- Access to Apple Developer Portal
- Apple Team ID: `8W6EM53C5H`
- Bundle ID: `uk.org.cruising.captainsmate`

**Steps:**

**1. Create Distribution Certificate**
```bash
# Generate CSR
openssl req -new -newkey rsa:2048 -nodes \
  -keyout ios_distribution.key \
  -out ios_distribution.csr

# Or use Keychain Access → Certificate Assistant → Request from CA
```

**2. Upload CSR to Apple Developer Portal**
1. Go to https://developer.apple.com
2. Certificates, Identifiers & Profiles → Certificates
3. Click "+" to create new
4. Select "Apple Distribution"
5. Upload CSR file
6. Download certificate
7. Double-click to install

**3. Create App Store Provisioning Profile**
1. Still in developer.apple.com
2. Profiles section
3. Click "+" to create new
4. Select "App Store"
5. Select App ID: `uk.org.cruising.captainsmate`
6. Select certificate you just created
7. Generate and download
8. Double-click to install

**4. Update Xcode Signing**
1. Open `ios/Runner.xcworkspace`
2. Select Runner project → Runner target
3. Signing & Capabilities tab
4. Uncheck "Automatically manage signing"
5. Select Team: `8W6EM53C5H`
6. Select Provisioning Profile you created

**5. Build Manually in Xcode**
1. Product → Archive
2. Distribute via Xcode Organizer

**6. (Optional) Set Up New Match**
```bash
cd ios

# Initialize new Match repo (use S3 or git)
bundle exec fastlane match init

# Generate new certificates
export MATCH_PASSWORD="new_secure_password"
bundle exec fastlane match appstore

# Update GitHub Secrets with new MATCH_PASSWORD
```

---

### Recovery: Lost Android Keystore - No Play App Signing

**When to use**: Keystore completely lost AND Play App Signing is NOT enabled.

**⚠️ CRITICAL WARNING**: You cannot update the existing app. You must publish a new app.

**Impact:**
- All existing users will NOT automatically get updates
- New app means new package name
- Lose all reviews, ratings, and statistics
- Lose all existing installs

**Steps:**

**1. Confirm Keystore is Unrecoverable**
- Checked all backups
- Contacted previous developer
- Checked S3, password managers, email archives
- Confirmed Play App Signing is NOT enabled

**2. Decision Point**

**Option A: Publish New App** (Required if Play App Signing disabled)
1. Create new keystore with NEW package name
2. Submit new app to Play Store
3. Communicate with existing users
4. Consider migration strategy

**Option B: Contact Google Play Support** (Slim chance)
1. Go to Play Console Help
2. Explain situation
3. Google MAY allow keystore reset in rare cases
4. Provide proof of ownership
5. Usually denied if Play App Signing not enabled

**3. Create New Keystore (If proceeding with new app)**
```bash
# Generate new keystore with NEW package name
keytool -genkey -v \
  -keystore new_keystore.jks \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -alias new-key-alias \
  -storetype JKS

# STORE SECURELY - multiple backups!
# DOCUMENT all passwords in password manager
```

**4. Update App Configuration**
```bash
# Change package name in pubspec.yaml
# Update ANDROID_APP_IDENTIFIER
# Update all references to old package name
```

**5. Submit New App**
- Create new Play Store listing
- New package name
- New app entry
- Cannot reuse old package name

**6. User Migration Plan**
- In-app notification in old app (if you can still build it)
- Email to users (if you have list)
- Website announcement
- Social media announcement

**Prevention for Future:**
1. **ENABLE PLAY APP SIGNING** immediately for new app
2. Store keystore in multiple secure locations
3. Document passwords in secure password manager
4. Set calendar reminders to verify backups
5. Add keystore to S3 backup
6. Keep encrypted backup with different provider

---

### Recovery: Lost All Access (S3, Passwords, Everything)

**Nuclear scenario**: Previous developer unavailable, no handover completed.

**Assessment:**

**What You Have Access To:**
- [ ] Apple Developer Portal?
- [ ] Google Play Console?
- [ ] GitHub repository?
- [ ] Firebase Console?
- [ ] AWS Console?
- [ ] Codebase only?

**Scenario A: Have Apple/Google Console Access**

**iOS:**
1. Generate new certificates via Developer Portal (see [Recovery: iOS Signing from Scratch](#recovery-ios-signing-from-scratch))
2. Create new App Store Connect API key
3. Download Firebase configs from Firebase Console
4. Can deploy, but need to rebuild signing infrastructure

**Android:**
1. Check if Play App Signing enabled (crucial!)
2. If yes: Can request new upload key from Google
3. If no: Cannot update app (see [Recovery: Lost Android Keystore - No Play App Signing](#recovery-lost-android-keystore---no-play-app-signing))
4. Download Firebase configs from Firebase Console

**Scenario B: Have AWS Console Access**
1. Download all files from S3 buckets
2. Still need passwords (keystore passwords, Match password)
3. May need to reset some credentials

**Scenario C: Have GitHub Repo Admin Access**
1. Can view GitHub Secrets list (but not values)
2. Can see workflow definitions
3. Know what variables are needed
4. Must obtain actual values elsewhere

**Scenario D: Only Have Codebase**
**This is worst case:**

**Immediate Actions:**
1. Request emergency access from company stakeholders
2. Contact previous developer through all channels
3. Check if company has password manager with shared vault
4. Check company documentation/wiki
5. Check email archives for any shared credentials

**If Cannot Recover:**

**iOS:**
- Generate new signing infrastructure
- Will work, but takes time to set up

**Android:**
- Check Play App Signing status
- If disabled: Cannot update existing app
- If enabled: Can request new upload key

**Both Platforms:**
- Can regenerate Firebase configs
- Can regenerate API keys
- Can rebuild OAuth configuration
- Main issue: Signing materials

---

### Prevention Checklist

To avoid these scenarios in the future:

**Backup Strategy:**
- [ ] Android keystore stored in 3+ locations
- [ ] iOS certificates exported and backed up
- [ ] All passwords in team password manager
- [ ] S3 bucket regularly backed up
- [ ] AWS credentials documented and secured
- [ ] Multiple team members have access

**Documentation:**
- [ ] Handover checklist completed
- [ ] All credentials documented
- [ ] Recovery procedures written
- [ ] Emergency contacts listed
- [ ] Access permissions documented

**Regular Verification:**
- [ ] Quarterly access verification
- [ ] Annual credential rotation
- [ ] Test backup restoration
- [ ] Verify CI/CD still works
- [ ] Update documentation

**Team Knowledge:**
- [ ] At least 2 people can build iOS
- [ ] At least 2 people can build Android
- [ ] Team knows where docs are
- [ ] Emergency procedures practiced
- [ ] Stakeholders aware of risks

---

## Emergency Contacts & Resources

### When You're Completely Stuck

**Apple Support:**
- Developer Support: https://developer.apple.com/support/
- App Store Connect: https://developer.apple.com/contact/app-store/

**Google Support:**
- Play Console Help: https://support.google.com/googleplay/android-developer/
- API Support: https://developers.google.com/android-publisher

**Community Resources:**
- Stack Overflow: https://stackoverflow.com/
- Flutter Discord: https://discord.com/invite/flutter
- Fastlane Community: https://github.com/fastlane/fastlane/discussions

**Escalation Path:**
1. Check this troubleshooting guide
2. Search error message online
3. Check official documentation
4. Contact previous developer
5. Contact infrastructure admin
6. Contact platform support (Apple/Google)
7. Consider hiring consultant if critical

---

*Guide Created: 2026-01-07*
*Last Updated: 2026-01-07*