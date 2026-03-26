---
id: 92bf91f9-392f-45dc-8b01-1a5ce8c62b26
blueprint: captains_mate_app_43
title: 'iOS Manual Build Quickstart'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774536593
---
# iOS Manual Build Quick-Start Guide

## Overview
This guide walks you through building and deploying the iOS app manually without using the CI/CD pipeline. Use this when you don't have access to GitHub Actions but need to release to TestFlight.

---

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] macOS machine with Xcode installed
- [ ] Flutter SDK installed (version 3.32.6 or current stable)
- [ ] Ruby 3.2+ installed
- [ ] Access to Apple Developer Portal (`appdeveloper@theca.org.uk`)
- [ ] Access to App Store Connect
- [ ] All environment variables from handover (see Section 9 of [IOS_HANDOVER_CHECKLIST.md](IOS_HANDOVER_CHECKLIST.md))
- [ ] `GoogleService-Info.plist` file downloaded
- [ ] Code signing certificates and provisioning profiles

---

## Method 1: Using Fastlane (Recommended if you have Match access)

This method uses the existing Fastlane setup but requires S3/Match access.

### Step 1: Set Up Environment Variables

Create a file to store your environment variables:

```bash
# Create env file in project root
cd /Users/emmabowerbank/Development/CA/Cruising-Association-tca-mobile-app-3c628ea7950f
nano .env.ios.local
```

Paste this content (fill in the actual values):

```bash
# Apple Configuration
export APPLE_APP_IDENTIFIER="uk.org.cruising.captainsmate"
export APPLE_APP_DEVELOPER_KEY_ID="YOUR_KEY_ID"
export APPLE_APP_DEVELOPER_ISSUER_ID="YOUR_ISSUER_ID"

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

# Firebase iOS
export DEFINE_FIREBASE_IOS_APP_ID="YOUR_VALUE"
export DEFINE_FIREBASE_IOS_API_KEY="YOUR_VALUE"
export DEFINE_FIREBASE_IOS_BUNDLE_ID="YOUR_VALUE"
export DEFINE_FIREBASE_SENDER_ID="YOUR_VALUE"
export DEFINE_FIREBASE_BUCKET_NAME="YOUR_VALUE"
export DEFINE_FIREBASE_PROJECT_ID="YOUR_VALUE"

# AWS & Match (if using)
export AWS_ACCESS_KEY_ID="YOUR_VALUE"
export AWS_SECRET_ACCESS_KEY="YOUR_VALUE"
export MATCH_PASSWORD="YOUR_VALUE"
```

Save and exit (Ctrl+X, then Y, then Enter).

### Step 2: Load Environment Variables

```bash
# Source the environment file
source .env.ios.local

# Verify variables are loaded
echo $APPLE_APP_IDENTIFIER
```

### Step 3: Set Up Required Files

```bash
# Place Apple API key in home directory
cp /path/to/apple_app_developer_api_key.p8 ~/apple_key.p8

# Place GoogleService-Info.plist in iOS Runner
cp /path/to/GoogleService-Info.plist ios/Runner/GoogleService-Info.plist

# Place Firebase config (if using Firebase App Distribution)
cp /path/to/firebase.json ~/firebase.json
```

### Step 4: Install Dependencies

```bash
# Install Flutter dependencies
flutter pub get

# Install Ruby gems (including Fastlane)
cd ios
bundle install
cd ..
```

### Step 5: Build with Fastlane

```bash
# Navigate to iOS directory
cd ios

# Run the full build and deploy process
bundle exec fastlane build_and_deploy

# OR run steps separately:
# Build only (creates IPA)
bundle exec fastlane build

# Deploy only (requires IPA path from build output)
bundle exec fastlane deploy path:/path/to/app.ipa
```

### Step 6: Monitor Upload

Watch the terminal output. If successful, you'll see:
```
✓ Successfully uploaded package to TestFlight
```

Check TestFlight in App Store Connect for processing status.

---

## Method 2: Manual Build Without Match (No S3 Access)

Use this method if you don't have access to S3/Match but have certificates manually installed.

### Step 1: Install Code Signing Materials Manually

#### Option A: Import from .p12 file
```bash
# Double-click the .p12 certificate file
# OR import via command line:
security import /path/to/certificate.p12 -k ~/Library/Keychains/login.keychain-db
```

#### Option B: Download from Apple Developer Portal
1. Go to https://developer.apple.com
2. Navigate to Certificates, Identifiers & Profiles
3. Download "Apple Distribution" certificate
4. Download Provisioning Profile for `uk.org.cruising.captainsmate`
5. Double-click both files to install

### Step 2: Set Up Environment Variables

Same as Method 1, Step 1 (but AWS/MATCH variables not needed).

```bash
source .env.ios.local
```

### Step 3: Place Required Files

```bash
# Place GoogleService-Info.plist
cp /path/to/GoogleService-Info.plist ios/Runner/GoogleService-Info.plist
```

### Step 4: Install Dependencies

```bash
flutter pub get

cd ios
bundle install
pod install
cd ..
```

### Step 5: Generate Code Files

```bash
# Run build_runner to generate required code
dart run build_runner build --delete-conflicting-outputs
```

### Step 6: Build IPA with Flutter

```bash
# Build for production flavor
flutter build ipa \
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
  --dart-define=FIREBASE_IOS_APP_ID="$DEFINE_FIREBASE_IOS_APP_ID" \
  --dart-define=FIREBASE_IOS_API_KEY="$DEFINE_FIREBASE_IOS_API_KEY" \
  --dart-define=FIREBASE_IOS_BUNDLE_ID="$DEFINE_FIREBASE_IOS_BUNDLE_ID" \
  --dart-define=FIREBASE_SENDER_ID="$DEFINE_FIREBASE_SENDER_ID" \
  --dart-define=FIREBASE_BUCKET_NAME="$DEFINE_FIREBASE_BUCKET_NAME" \
  --dart-define=FIREBASE_PROJECT_ID="$DEFINE_FIREBASE_PROJECT_ID"
```

**Note:** If this fails with code signing errors, proceed to Method 3 (Xcode).

### Step 7: Upload to TestFlight

The IPA will be located at: `build/ios/ipa/cruising_association.ipa` (or similar name).

#### Option A: Using Transporter App
1. Download "Transporter" from Mac App Store
2. Open Transporter
3. Drag and drop the IPA file
4. Click "Deliver"

#### Option B: Using Fastlane (if you have API key)
```bash
# Place API key
cp /path/to/apple_app_developer_api_key.p8 ~/apple_key.p8

# Upload with Fastlane
cd ios
bundle exec fastlane deliver \
  --ipa ../build/ios/ipa/cruising_association.ipa \
  --skip_screenshots \
  --skip_metadata
```

---

## Method 3: Build with Xcode (Most Manual, No Fastlane)

Use this when all automated methods fail or you prefer full manual control.

### Step 1: Open Project in Xcode

```bash
cd ios
open Runner.xcworkspace
```

### Step 2: Configure Signing in Xcode

1. Select "Runner" project in left sidebar
2. Select "Runner" target
3. Go to "Signing & Capabilities" tab
4. **Uncheck** "Automatically manage signing"
5. Select your Team: `8W6EM53C5H`
6. For "Release" configuration:
   - Provisioning Profile: Select the one for `uk.org.cruising.captainsmate`
   - Signing Certificate: "Apple Distribution"

### Step 3: Set Up Environment in Xcode (Optional)

If you need to test run in Xcode:
1. Product → Scheme → Edit Scheme
2. Run → Arguments → Environment Variables
3. Add each `DEFINE_*` variable

**Note:** For release builds, you may skip this and use flutter build instead.

### Step 4: Build Archive

1. In Xcode, select device: **Any iOS Device (arm64)**
2. Menu: Product → Archive
3. Wait for build to complete (5-15 minutes)
4. Archive Organizer window will open automatically

### Step 5: Distribute Archive

1. In Organizer window, select your archive
2. Click "Distribute App"
3. Choose "App Store Connect"
4. Click "Next"
5. Choose "Upload"
6. Click "Next" through options
7. Review and click "Upload"
8. Wait for upload to complete

### Step 6: Verify in App Store Connect

1. Go to https://appstoreconnect.apple.com
2. Navigate to your app
3. Go to TestFlight tab
4. Wait for build to appear (can take 5-15 minutes)
5. Wait for processing to complete (can take 30+ minutes)

---

## Troubleshooting Common Issues

### Issue: "No provisioning profiles found"

**Solution:**
```bash
# Download profiles from Apple
# OR if using Match:
cd ios
bundle exec fastlane match appstore --readonly
```

### Issue: "Certificate not trusted"

**Solution:**
1. Open Keychain Access
2. Find your certificate
3. Double-click → Trust → Always Trust
4. Enter your password

### Issue: "Unable to find App ID"

**Solution:**
Verify `APPLE_APP_IDENTIFIER` is set correctly:
```bash
echo $APPLE_APP_IDENTIFIER
# Should output: uk.org.cruising.captainsmate
```

### Issue: Flutter build fails with "No Firebase options"

**Solution:**
Ensure `GoogleService-Info.plist` is in the correct location:
```bash
ls -la ios/Runner/GoogleService-Info.plist
```

### Issue: "Build number already exists" in TestFlight

**Solution:**
Increment build number manually in Xcode:
1. Open `Runner.xcworkspace` in Xcode
2. Select Runner project → Runner target
3. General tab → Build field
4. Increment by 1

OR let Fastlane auto-increment:
```bash
cd ios
bundle exec fastlane run increment_build_number
```

### Issue: Match password incorrect

**Solution:**
Double-check the `MATCH_PASSWORD` value. If lost:
- Contact previous developer or infrastructure admin
- OR regenerate certificates (will invalidate old ones)

### Issue: API key authentication failed

**Solution:**
1. Verify API key file exists: `ls ~/apple_key.p8`
2. Check key hasn't expired in App Store Connect
3. Verify `APPLE_APP_DEVELOPER_KEY_ID` and `APPLE_APP_DEVELOPER_ISSUER_ID`

### Issue: Pod install fails

**Solution:**
```bash
cd ios
pod repo update
pod install --repo-update
cd ..
```

---

## Quick Reference: Build Commands

### Full automated build (with Match):
```bash
source .env.ios.local
cd ios
bundle exec fastlane build_and_deploy
```

### Build without deploy:
```bash
source .env.ios.local
cd ios
bundle exec fastlane build
```

### Flutter build directly:
```bash
source .env.ios.local
dart run build_runner build --delete-conflicting-outputs
flutter build ipa --release --flavor production \
  --dart-define=OAUTH_CLIENT_ID="$DEFINE_OAUTH_CLIENT_ID" \
  # ... (add all other dart-defines)
```

### Upload existing IPA:
```bash
# Via Fastlane
cd ios
bundle exec fastlane deliver --ipa /path/to/app.ipa

# Via Transporter (GUI)
# Open Transporter app and drag IPA file
```

---

## Post-Build Verification

After successful build and upload:

1. **Check TestFlight**
   - Log into App Store Connect
   - Navigate to TestFlight
   - Verify new build appears
   - Check build status (processing/ready to test)

2. **Test the Build**
   - Add yourself as internal tester
   - Install via TestFlight app on device
   - Verify app launches and core functionality works

3. **Notify Team**
   - Inform team new build is available
   - Provide build number and release notes
   - Request testing feedback

---

## Tips for Success

1. **First Build**: Expect it to take longer. Xcode may need to download components.

2. **Keep Environment File Secure**: Never commit `.env.ios.local` to git. Add to `.gitignore`:
   ```bash
   echo ".env.ios.local" >> .gitignore
   ```

3. **Test Incrementally**: Try each method in order. Start with Fastlane if possible.

4. **Document Deviations**: If you modify the process, document what you changed and why.

5. **Save Success Recipe**: Once you successfully build, save the exact commands you used.

6. **Build Regularly**: Don't wait until emergency. Practice builds help identify issues early.

---

## Emergency Contact Information

If you get completely stuck:

- **Apple Developer Support**: https://developer.apple.com/support/
- **Fastlane Documentation**: https://docs.fastlane.tools/
- **Flutter iOS Deployment**: https://docs.flutter.dev/deployment/ios

---

## Next Steps After First Successful Build

- [ ] Document any issues you encountered
- [ ] Update this guide with your findings
- [ ] Set up calendar reminders for certificate renewals
- [ ] Consider setting up CI/CD access for easier future releases
- [ ] Create a build checklist for your specific setup

---

*Guide Created: 2026-01-07*
*Last Updated: 2026-01-07*