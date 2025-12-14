# Fastlane Configuration for Formulus iOS

This directory contains the Fastlane configuration for automating iOS builds and App Store submissions.

## Prerequisites

1. **App Store Connect API Key**: Create an API key in App Store Connect with App Manager or Admin access
2. **Apple Developer Account**: Active membership in the Apple Developer Program
3. **Bundle Identifier**: Ensure your app's bundle identifier is registered in App Store Connect

## Setup

### 1. Install Dependencies

```bash
cd formulus/ios
bundle install
```

### 2. Configure App Store Connect API Key

You have two options:

#### Option A: Use API Key (Recommended for CI/CD)

1. Create an API key in App Store Connect:
   - Go to Users and Access > Keys
   - Create a new key with App Manager or Admin role
   - Download the `.p8` key file

2. Set environment variables:
   ```bash
   export APP_STORE_CONNECT_API_KEY_ID="your-key-id"
   export APP_STORE_CONNECT_ISSUER_ID="your-issuer-id"
   export APP_STORE_CONNECT_KEY_CONTENT="$(cat /path/to/AuthKey_XXXXX.p8)"
   ```

#### Option B: Use App-Specific Password (Manual)

1. Generate an app-specific password in your Apple ID account
2. Set environment variable:
   ```bash
   export FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD="your-password"
   export FASTLANE_USER="your-apple-id@example.com"
   ```

### 3. Configure Match (Code Signing)

Match is used to manage certificates and provisioning profiles. Set up a private Git repository to store certificates:

```bash
# Set the match git URL
export MATCH_GIT_BASIC_AUTHORIZATION="$(echo -n 'username:token' | base64)"
export MATCH_GIT_BRANCH="main"
export MATCH_GIT_URL="https://github.com/your-org/certificates-repo.git"

# Run match setup (first time only)
bundle exec fastlane match appstore
```

For CI/CD, use readonly mode:
```bash
bundle exec fastlane match appstore readonly:true
```

## Usage

### Build the App

```bash
bundle exec fastlane build
```

### Upload to TestFlight

```bash
# Upload to TestFlight
bundle exec fastlane beta

# Upload and distribute to external testers
bundle exec fastlane beta distribute_external:true

# Upload without waiting for processing
bundle exec fastlane beta skip_waiting:true
```

### Upload to App Store

```bash
# Upload to App Store Connect
bundle exec fastlane release

# Upload and submit for review
bundle exec fastlane release submit_for_review:true

# Upload with automatic release
bundle exec fastlane release automatic_release:true
```

### Run Tests

```bash
bundle exec fastlane test
```

## Environment Variables

The following environment variables can be set:

- `APP_STORE_CONNECT_API_KEY_ID`: API Key ID from App Store Connect
- `APP_STORE_CONNECT_ISSUER_ID`: Issuer ID from App Store Connect
- `APP_STORE_CONNECT_KEY_CONTENT`: Content of the `.p8` key file
- `APPLE_TEAM_ID`: Your Apple Developer Team ID
- `BUNDLE_IDENTIFIER`: Your app's bundle identifier
- `PROVISIONING_PROFILE_NAME`: Name of the provisioning profile (if using manual profiles)
- `MATCH_GIT_URL`: URL of the Git repository for storing certificates
- `MATCH_GIT_BASIC_AUTHORIZATION`: Base64-encoded credentials for the certificates repo

## Versioning

Fastlane can automatically increment version numbers:

- `increment_version`: Increment the marketing version (e.g., 1.0.0 → 1.0.1)
- `increment_build`: Increment the build number (e.g., 1 → 2)

You can specify custom versions:
```bash
bundle exec fastlane release version_number:1.2.0 build_number:100
```

## Troubleshooting

### Certificate Issues

If you encounter certificate errors:
1. Ensure your certificates are valid in Apple Developer Portal
2. Run `bundle exec fastlane match appstore` to sync certificates
3. Check that your bundle identifier matches the provisioning profile

### Build Errors

1. Ensure all dependencies are installed: `bundle install && pod install`
2. Check Xcode version compatibility
3. Verify React Native and iOS deployment target versions

### Upload Errors

1. Verify App Store Connect API key has correct permissions
2. Check that the app exists in App Store Connect
3. Ensure version numbers are incremented correctly

## Additional Resources

- [Fastlane Documentation](https://docs.fastlane.tools)
- [App Store Connect API](https://developer.apple.com/app-store-connect/api/)
- [Match Documentation](https://docs.fastlane.tools/actions/match/)
