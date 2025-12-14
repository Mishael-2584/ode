# iOS App Store Publishing Setup Guide

This guide walks you through setting up the automated iOS App Store publishing pipeline for the Formulus app.

## Overview

The iOS App Store publishing pipeline uses:
- **Fastlane**: For automating builds, code signing, and App Store submissions
- **GitHub Actions**: For CI/CD automation
- **App Store Connect API**: For secure authentication (recommended)
- **Match**: For managing certificates and provisioning profiles (optional)

## Prerequisites

1. **Apple Developer Account**: Active membership in the Apple Developer Program ($99/year)
2. **App Store Connect Access**: Admin or App Manager role
3. **App Registered**: Your app must be registered in App Store Connect with a bundle identifier
4. **GitHub Repository**: With Actions enabled

## Step 1: Create App Store Connect API Key

The App Store Connect API key is the recommended method for CI/CD authentication.

### 1.1 Create the API Key

1. Go to [App Store Connect](https://appstoreconnect.apple.com/)
2. Navigate to **Users and Access** → **Keys** → **App Store Connect API**
3. Click the **+** button to create a new key
4. Enter a name (e.g., "Formulus CI/CD")
5. Select **App Manager** or **Admin** role
6. Click **Generate**
7. **Download the `.p8` key file** (you can only download it once!)
8. Note the **Key ID** and **Issuer ID** displayed on the page

### 1.2 Store the API Key

Save the `.p8` file securely. You'll need to add its content to GitHub Secrets.

## Step 2: Configure GitHub Secrets

Add the following secrets to your GitHub repository:

1. Go to your repository → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret** for each secret below:

### Required Secrets

| Secret Name | Description | How to Get |
|------------|--------------|------------|
| `APP_STORE_CONNECT_API_KEY_ID` | The Key ID from App Store Connect | From Step 1.1 (e.g., `ABC123XYZ`) |
| `APP_STORE_CONNECT_ISSUER_ID` | The Issuer ID from App Store Connect | From Step 1.1 (UUID format) |
| `APP_STORE_CONNECT_KEY_CONTENT` | Content of the `.p8` file | `cat AuthKey_XXXXX.p8` (entire file content) |
| `APPLE_TEAM_ID` | Your Apple Developer Team ID | From [Apple Developer Portal](https://developer.apple.com/account) → Membership |

### Optional Secrets

| Secret Name | Description | Default |
|------------|--------------|---------|
| `IOS_BUNDLE_IDENTIFIER` | Your app's bundle identifier | `org.reactjs.native.example.Formulus` |
| `MATCH_GIT_URL` | Git repo URL for certificates (if using Match) | Not used if not set |
| `MATCH_GIT_BASIC_AUTHORIZATION` | Base64 auth for certificates repo | Not used if not set |
| `MATCH_GIT_BRANCH` | Branch for certificates repo | `main` |

### Example: Adding APP_STORE_CONNECT_KEY_CONTENT

```bash
# On macOS/Linux
cat AuthKey_XXXXX.p8 | pbcopy  # macOS
cat AuthKey_XXXXX.p8 | xclip   # Linux

# Then paste into GitHub Secrets
```

Or manually copy the entire content of the `.p8` file (including `-----BEGIN PRIVATE KEY-----` and `-----END PRIVATE KEY-----`).

## Step 3: Configure Code Signing

You have two options for code signing:

### Option A: Use Match (Recommended for Teams)

Match manages certificates and provisioning profiles in a Git repository.

#### 3.1 Create a Private Git Repository

Create a private repository to store certificates (e.g., `your-org/certificates`).

#### 3.2 Set Up Match Locally

```bash
cd formulus/ios
bundle install
bundle exec fastlane match appstore
```

This will:
- Create certificates and provisioning profiles
- Store them in your Git repository
- Configure your Xcode project

#### 3.3 Configure GitHub Secrets

Add these secrets:
- `MATCH_GIT_URL`: URL of your certificates repository
- `MATCH_GIT_BASIC_AUTHORIZATION`: Base64-encoded `username:token` for Git access
- `MATCH_GIT_BRANCH`: Branch name (default: `main`)

To generate `MATCH_GIT_BASIC_AUTHORIZATION`:
```bash
echo -n "username:personal-access-token" | base64
```

### Option B: Manual Certificates (Simpler for Solo Developers)

If you prefer to manage certificates manually:

1. Create certificates in [Apple Developer Portal](https://developer.apple.com/account/resources/certificates/list)
2. Download and install certificates on your Mac
3. Create provisioning profiles in the portal
4. Download and install provisioning profiles
5. The workflow will use the certificates from the macOS runner's keychain

**Note**: Manual certificates require the workflow to have access to the keychain, which is available on GitHub's macOS runners.

## Step 4: Update Bundle Identifier

Ensure your app's bundle identifier matches what's registered in App Store Connect:

1. Open `formulus/ios/Formulus.xcodeproj` in Xcode
2. Select the **Formulus** target
3. Go to **Signing & Capabilities**
4. Update **Bundle Identifier** to match your App Store Connect app
5. Or set the `IOS_BUNDLE_IDENTIFIER` secret in GitHub

## Step 5: Verify Workflow Configuration

The workflow is already configured at `.github/workflows/formulus-ios.yml`. Verify:

1. The workflow triggers on the correct branches
2. The Xcode version matches your requirements
3. The Node.js and Ruby versions are compatible

## Step 6: Test the Pipeline

### 6.1 Test Build (Pull Request)

1. Create a pull request that modifies files in `formulus/`
2. The workflow will build the app for the iOS Simulator
3. Check the **Actions** tab to verify the build succeeds

### 6.2 Test TestFlight Upload

1. Push to `main` or `dev` branch
2. The workflow will automatically:
   - Build the iOS app
   - Upload to TestFlight
3. Check App Store Connect → TestFlight to see the build

### 6.3 Manual TestFlight Upload

1. Go to **Actions** → **Formulus iOS Build and Release**
2. Click **Run workflow**
3. Select branch: `main`
4. Release type: `testflight`
5. Click **Run workflow**

## Step 7: Release to App Store

### 7.1 Upload to App Store Connect

1. Go to **Actions** → **Formulus iOS Build and Release**
2. Click **Run workflow**
3. Select branch: `main`
4. Release type: `appstore`
5. (Optional) Check `submit_for_review` to automatically submit
6. Click **Run workflow**

### 7.2 Complete App Store Listing

After upload, complete your App Store listing in App Store Connect:

1. Go to **App Store Connect** → **My Apps** → **Formulus**
2. Select the new build
3. Complete required metadata:
   - App description
   - Screenshots
   - Privacy policy URL
   - App category
   - Age rating
4. Submit for review

## Versioning

### Automatic Versioning

The workflow automatically increments build numbers. To manage versions:

- **Marketing Version** (e.g., 1.0.0): Update in Xcode or via Fastlane
- **Build Number**: Automatically incremented on each build

### Manual Version Control

You can specify versions in the Fastfile or update them in Xcode:

```bash
# Update version in Xcode project
agvtool new-marketing-version 1.2.0
agvtool next-version -all
```

## Troubleshooting

### Build Fails: "No signing certificate found"

**Solution**: Ensure certificates are set up:
- If using Match: Verify `MATCH_GIT_URL` and related secrets are set
- If using manual: Ensure certificates are installed on the macOS runner (they should be available by default)

### Upload Fails: "Authentication failed"

**Solution**: Verify App Store Connect API credentials:
1. Check `APP_STORE_CONNECT_API_KEY_ID` matches the Key ID
2. Verify `APP_STORE_CONNECT_ISSUER_ID` is correct
3. Ensure `APP_STORE_CONNECT_KEY_CONTENT` includes the full `.p8` file content (including headers)

### Upload Fails: "Invalid bundle identifier"

**Solution**: 
1. Verify the bundle identifier in Xcode matches App Store Connect
2. Check that the app exists in App Store Connect with this bundle ID
3. Ensure `IOS_BUNDLE_IDENTIFIER` secret matches (if set)

### Build Fails: "Pod install failed"

**Solution**:
1. Check CocoaPods version compatibility
2. Verify `Podfile` is correct
3. Check network connectivity during pod install

### TestFlight Upload Succeeds but Build Not Visible

**Solution**:
1. Processing can take 10-30 minutes
2. Check App Store Connect → TestFlight → Builds
3. Verify the build is not expired or removed
4. Check email notifications from Apple

## Best Practices

### Security

1. **Never commit** `.p8` files or certificates to the repository
2. **Rotate API keys** periodically (every 6-12 months)
3. **Use Match** for team environments to centralize certificate management
4. **Limit API key permissions** to minimum required (App Manager, not Admin)

### Versioning

1. **Use semantic versioning**: `MAJOR.MINOR.PATCH` (e.g., 1.2.3)
2. **Increment build numbers** for each TestFlight upload
3. **Tag releases** in Git for traceability
4. **Document version changes** in release notes

### Testing

1. **Test in TestFlight** before App Store submission
2. **Use internal testing** for quick iterations
3. **Use external testing** for broader beta testing
4. **Monitor crash reports** in App Store Connect

### Workflow

1. **Use feature branches** for development
2. **Merge to `dev`** for TestFlight testing
3. **Merge to `main`** for App Store releases
4. **Create GitHub releases** to trigger App Store uploads

## Additional Resources

- [Fastlane Documentation](https://docs.fastlane.tools)
- [App Store Connect API Documentation](https://developer.apple.com/documentation/appstoreconnectapi)
- [Match Documentation](https://docs.fastlane.tools/actions/match/)
- [Apple Developer Portal](https://developer.apple.com/account)
- [App Store Connect](https://appstoreconnect.apple.com/)

## Support

If you encounter issues:

1. Check the **Actions** tab for detailed error logs
2. Review Fastlane logs in the workflow output
3. Check App Store Connect for build status
4. Consult the troubleshooting section above
5. Open an issue in the repository with error details
