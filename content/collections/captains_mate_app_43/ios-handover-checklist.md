---
id: 579c4f14-29dd-425e-b1cc-a5ab846a5314
blueprint: captains_mate_app_43
title: 'iOS Handover Checklist'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774536542
---
# iOS Build & Deployment Handover Checklist

## Overview
This checklist ensures the new developer has everything needed to build and release the iOS app manually without CI/CD pipeline access.

---

## 1. Apple Developer Account Access

### Required Access
- [ ] **Apple Developer Portal** login credentials
  - Account: `appdeveloper@theca.org.uk`
  - Provide password/2FA method
  - Team ID: `8W6EM53C5H`

- [ ] **App Store Connect** login credentials
  - Same account as above
  - Ensure access to TestFlight and App Store management
  - Verify permissions include app upload rights

### App Information
- [ ] Bundle Identifier: `uk.org.cruising.captainsmate`
- [ ] Current app version and build number in TestFlight
- [ ] Any pending app reviews or submissions

---

## 2. Code Signing Materials

### Certificates & Profiles (Option 1: Via Match/S3)
- [ ] **AWS S3 Access**
  - AWS Access Key ID: `<provide value>`
  - AWS Secret Access Key: `<provide value>`
  - Bucket: `tca-app-certificates`
  - Region: `eu-west-2`

- [ ] **Match Password**
  - Password to decrypt Match certificates: `<provide value>`

### Certificates & Profiles (Option 2: Manual Export)
If S3 access not available, export from current machine:
- [ ] **Distribution Certificate** (.p12 file)
  - Export from Keychain Access
  - Provide export password
  - Certificate name: "Apple Distribution: ..."

- [ ] **Provisioning Profile**
  - Export .mobileprovision file
  - Profile name: `match AppStore uk.org.cruising.captainsmate`
  - Or download fresh from Apple Developer Portal

---

## 3. App Store Connect API Key

Required for automated TestFlight uploads:

- [ ] **API Key File** (`apple_key.p8`)
  - Download from S3: `s3://tca-secure-assets/apple_app_developer_api_key.p8`
  - OR download fresh from App Store Connect → Users and Access → Keys

- [ ] **API Key Details**
  - Key ID: `<provide APPLE_APP_DEVELOPER_KEY_ID>`
  - Issuer ID: `<provide APPLE_APP_DEVELOPER_ISSUER_ID>`
  - Key file location: Store as `~/apple_key.p8`

---

## 4. Firebase Configuration

### Required Files

- [ ] **GoogleService-Info.plist**
  - Download from S3: `s3://tca-secure-assets/GoogleService-Info.plist`
  - OR download from Firebase Console → Project Settings → iOS app
  - Destination: `ios/Runner/GoogleService-Info.plist`

- [ ] **firebase.json** (for Firebase App Distribution, if used)
  - Download from S3: `s3://tca-secure-assets/firebase.json`
  - Destination: `~/firebase.json`

### Firebase Credentials (dart-define values)

- [ ] `DEFINE_FIREBASE_IOS_APP_ID` = `<provide value>`
- [ ] `DEFINE_FIREBASE_IOS_API_KEY` = `<provide value>`
- [ ] `DEFINE_FIREBASE_IOS_BUNDLE_ID` = `<provide value>`
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
- [ ] `apple_app_developer_api_key.p8`
- [ ] `firebase.json`
- [ ] `GoogleService-Info.plist`

### Bucket: `tca-app-certificates`
- [ ] Match certificates and provisioning profiles (encrypted)
- [ ] Requires `MATCH_PASSWORD` to decrypt

### AWS Credentials
- [ ] AWS Access Key ID: `<provide value>`
- [ ] AWS Secret Access Key: `<provide value>`
- [ ] Region: `eu-west-2`
- [ ] IAM permissions: Read access to both buckets

---

## 8. Development Environment Setup

### Software Requirements
- [ ] macOS machine (for iOS builds)
- [ ] Xcode version: `<current version>` or later
- [ ] Command Line Tools installed
- [ ] Flutter SDK: 3.32.6 (stable channel) or current version
- [ ] Ruby: 3.2 or later
- [ ] CocoaPods installed
- [ ] Fastlane installed (`bundle install` in ios/ directory)

### Verification Commands
```bash
# Verify installations
xcode-select --version
flutter --version
ruby --version
pod --version
bundle --version
```

---

## 9. Local Build Instructions

### Create Environment Setup Script

Provide a `.env` file or script with all environment variables:

```bash
# Apple Configuration
export APPLE_APP_IDENTIFIER="uk.org.cruising.captainsmate"
export APPLE_APP_DEVELOPER_KEY_ID="<value>"
export APPLE_APP_DEVELOPER_ISSUER_ID="<value>"

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

# Firebase iOS
export DEFINE_FIREBASE_IOS_APP_ID="<value>"
export DEFINE_FIREBASE_IOS_API_KEY="<value>"
export DEFINE_FIREBASE_IOS_BUNDLE_ID="<value>"
export DEFINE_FIREBASE_SENDER_ID="<value>"
export DEFINE_FIREBASE_BUCKET_NAME="<value>"
export DEFINE_FIREBASE_PROJECT_ID="<value>"

# AWS (if using Match/S3)
export AWS_ACCESS_KEY_ID="<value>"
export AWS_SECRET_ACCESS_KEY="<value>"
export MATCH_PASSWORD="<value>"
```

### Test Build Commands
- [ ] Provide working example of local build command
- [ ] Document any custom build flags or configurations
- [ ] Explain flavor system (production vs staging)

---

## 10. Knowledge Transfer Sessions

### Required Walkthroughs
- [ ] **Code Signing Process**
  - Explain how Match works
  - Show how to manually manage certificates if Match unavailable
  - Demonstrate certificate renewal process

- [ ] **Build Process**
  - Walk through Fastlane lanes
  - Explain build versioning strategy
  - Show how build numbers are incremented

- [ ] **Deployment Process**
  - TestFlight upload procedures
  - App Store submission process
  - Release notes and version management

- [ ] **Troubleshooting Common Issues**
  - Certificate expiration
  - Provisioning profile mismatches
  - Build failures and debugging
  - TestFlight processing issues

---

## 11. Documentation & Resources

- [ ] **Access to Internal Documentation**
  - Wiki or confluence pages
  - Architecture diagrams
  - API documentation

- [ ] **Contact Information**
  - Who to contact for Apple Developer account issues
  - Who manages AWS infrastructure
  - Firebase admin contacts
  - Emergency contacts for production issues

- [ ] **Reference Links**
  - Apple Developer Portal: https://developer.apple.com
  - App Store Connect: https://appstoreconnect.apple.com
  - Firebase Console: https://console.firebase.google.com
  - AWS Console: https://console.aws.amazon.com

---

## 12. Security & Access Management

- [ ] **Password Manager Access**
  - If using 1Password/LastPass/etc for credentials
  - Shared vault or collection name

- [ ] **2FA Backup Codes**
  - Apple account 2FA backup codes
  - AWS MFA backup (if enabled)

- [ ] **Certificate Expiration Dates**
  - Distribution certificate expiry: `<date>`
  - Provisioning profile expiry: `<date>`
  - API key expiry: `<date or note if doesn't expire>`
  - Set calendar reminders for renewals

---

## 13. Current State & Pending Work

- [ ] **Last Successful Build**
  - Date: `<date>`
  - Version: `<version>`
  - Build number: `<build>`
  - Commit hash: `<hash>`

- [ ] **Known Issues**
  - Any ongoing build problems
  - Deprecated dependencies
  - Pending certificate renewals

- [ ] **Pending Releases**
  - Any versions awaiting App Store review
  - Planned upcoming releases
  - Features in development

---

## 14. Emergency Procedures

- [ ] **If Build Fails**
  - Rollback procedures
  - Where to find previous working builds
  - Who to notify

- [ ] **If Certificate Expires**
  - Renewal process
  - Timeline for renewal
  - Impact on users

- [ ] **Production Incident Response**
  - On-call contacts
  - Incident escalation process
  - Rollback capabilities

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
- [ ] New developer completed test build and upload to TestFlight
- [ ] All environment variables verified and working
- [ ] All questions answered

---

## Notes & Additional Information

_Use this space for any additional context, quirks, or important information:_

```
<Add any additional notes here>
```

---

*Document Created: 2026-01-07*
*Last Updated: 2026-01-07*