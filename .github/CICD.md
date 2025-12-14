# CI/CD Pipeline Documentation

This document describes the CI/CD pipelines for the Open Data Ensemble (ODE) monorepo.

## Overview

The ODE monorepo uses GitHub Actions for continuous integration and deployment. Each project has its own pipeline that triggers only when relevant files change.

## Pipelines

### Synkronus Docker Build & Publish

**Workflow File**: `.github/workflows/synkronus-docker.yml`

#### Triggers

- **Push to `main`**: Builds and publishes release images
- **Push to `develop`**: Builds and publishes pre-release images
- **Push to feature branches**: Builds and publishes branch-specific images
- **Pull Requests**: Builds but does not publish (validation only)
- **Manual Dispatch**: Allows manual triggering with optional version tag

#### Path Filters

The workflow only runs when files in these paths change:
- `synkronus/**` - Any file in the Synkronus project
- `.github/workflows/synkronus-docker.yml` - The workflow itself

#### Image Registry

Images are published to **GitHub Container Registry (GHCR)**:
- Registry: `ghcr.io`
- Image: `ghcr.io/opendataensemble/synkronus`

#### Tagging Strategy

| Branch/Event | Tags Generated | Description |
|--------------|----------------|-------------|
| `main` | `latest`, `main-{sha}` | Latest stable release |
| `develop` | `develop`, `develop-{sha}` | Development pre-release |
| Feature branches | `{branch-name}`, `{branch-name}-{sha}` | Feature-specific builds |
| Pull Requests | `pr-{number}` | PR validation builds (not pushed) |
| Manual with version | `v{version}`, `v{major}.{minor}`, `latest` | Versioned release |

#### Build Features

- **Multi-platform**: Builds for `linux/amd64` and `linux/arm64`
- **Build Cache**: Uses GitHub Actions cache for faster builds
- **Attestation**: Generates build provenance for security
- **Metadata**: Includes OCI-compliant labels and annotations

#### Permissions Required

The workflow requires these permissions:
- `contents: read` - To checkout the repository
- `packages: write` - To publish to GHCR

#### Secrets Used

- `GITHUB_TOKEN` - Automatically provided by GitHub Actions

### Formulus Android Build

**Workflow File**: `.github/workflows/formulus-android.yml`

#### Triggers

- **Push to `main` or `develop`**: Builds release APK
- **Pull Requests**: Builds debug APK (validation only)
- **Release published**: Builds and uploads to GitHub Release

#### Path Filters

The workflow only runs when files in these paths change:
- `formulus/**` - Any file in the Formulus project
- `.github/workflows/formulus-android.yml` - The workflow itself

#### Build Features

- **Node.js**: Uses Node.js 18.x
- **Java**: Uses Java 17 (Temurin distribution)
- **Signing**: Supports release signing with keystore (non-PR builds)
- **Artifacts**: Uploads APK as GitHub Actions artifact

#### Secrets Used

- `FORMULUS_RELEASE_KEYSTORE_B64` - Base64-encoded keystore file
- `FORMULUS_RELEASE_STORE_PASSWORD` - Keystore password
- `FORMULUS_RELEASE_KEY_ALIAS` - Key alias
- `FORMULUS_RELEASE_KEY_PASSWORD` - Key password

### Formulus iOS Build and Release

**Workflow File**: `.github/workflows/formulus-ios.yml`

#### Triggers

- **Push to `main` or `dev`**: Builds iOS app and uploads to TestFlight
- **Pull Requests**: Builds iOS app for simulator (validation only)
- **Release published**: Builds and uploads to TestFlight
- **Manual Dispatch**: Allows manual triggering with release type selection

#### Path Filters

The workflow only runs when files in these paths change:
- `formulus/**` - Any file in the Formulus project
- `.github/workflows/formulus-ios.yml` - The workflow itself

#### Build Features

- **macOS Runner**: Uses `macos-14` for iOS builds
- **Node.js**: Uses Node.js 18.x
- **Ruby**: Uses Ruby 3.2 for Fastlane
- **Xcode**: Uses Xcode 15.4
- **Fastlane**: Automated build, signing, and upload
- **Code Signing**: Supports Match for certificate management
- **Artifacts**: Uploads IPA and dSYM files

#### Release Types

1. **TestFlight**: Uploads to TestFlight for beta testing
2. **App Store**: Uploads to App Store Connect for production release

#### Secrets Required

**App Store Connect API (Recommended)**:
- `APP_STORE_CONNECT_API_KEY_ID` - API Key ID from App Store Connect
- `APP_STORE_CONNECT_ISSUER_ID` - Issuer ID from App Store Connect
- `APP_STORE_CONNECT_KEY_CONTENT` - Content of the `.p8` key file

**Code Signing (Match)**:
- `MATCH_GIT_URL` - URL of Git repository for storing certificates
- `MATCH_GIT_BASIC_AUTHORIZATION` - Base64-encoded credentials for certificates repo
- `MATCH_GIT_BRANCH` - Branch name for certificates (default: `main`)

**App Configuration**:
- `APPLE_TEAM_ID` - Your Apple Developer Team ID
- `IOS_BUNDLE_IDENTIFIER` - Your app's bundle identifier (optional, defaults to `org.reactjs.native.example.Formulus`)

#### Manual Release Process

**Upload to TestFlight**:
1. Go to **Actions** → **Formulus iOS Build and Release**
2. Click **Run workflow**
3. Select branch (`main` or `dev`)
4. Choose `testflight` as release type
5. (Optional) Set `skip_waiting` to `true` to skip waiting for processing
6. Click **Run workflow**

**Upload to App Store**:
1. Go to **Actions** → **Formulus iOS Build and Release**
2. Click **Run workflow**
3. Select branch (`main`)
4. Choose `appstore` as release type
5. (Optional) Set `submit_for_review` to `true` to automatically submit for review
6. Click **Run workflow**

#### Versioning

- Build numbers are automatically incremented on each build
- Version numbers can be manually specified or auto-incremented
- Fastlane handles version management in Xcode project

#### Documentation

- [Fastlane Setup Guide](../formulus/ios/fastlane/README.md) - Detailed Fastlane configuration
- [iOS App Store Setup Guide](../formulus/IOS_APP_STORE_SETUP.md) - Complete setup instructions

## Using Published Images

### Pull Latest Release

```bash
docker pull ghcr.io/opendataensemble/synkronus:latest
```

### Pull Specific Version

```bash
docker pull ghcr.io/opendataensemble/synkronus:v1.0.0
```

### Pull Development Build

```bash
docker pull ghcr.io/opendataensemble/synkronus:develop
```

### Pull Feature Branch Build

```bash
docker pull ghcr.io/opendataensemble/synkronus:feature-xyz
```

## Manual Release Process

To create a versioned release:

1. Go to **Actions** → **Synkronus Docker Build & Publish**
2. Click **Run workflow**
3. Select the `main` branch
4. Enter version (e.g., `v1.0.0`)
5. Click **Run workflow**

This will create:
- `ghcr.io/opendataensemble/synkronus:latest`
- `ghcr.io/opendataensemble/synkronus:v1.0.0`
- `ghcr.io/opendataensemble/synkronus:v1.0`

## Image Visibility

By default, GHCR packages inherit the repository's visibility:
- **Public repositories** → Public images (no authentication needed)
- **Private repositories** → Private images (authentication required)

### Authenticating with GHCR

For private images:

```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
```

## Monitoring Builds

### View Workflow Runs

1. Go to the **Actions** tab in GitHub
2. Select **Synkronus Docker Build & Publish**
3. View recent runs and their status

### View Published Images

1. Go to the repository main page
2. Click **Packages** in the right sidebar
3. Select **synkronus**
4. View all published tags and their details

## Troubleshooting

### Build Fails on Push

1. Check the **Actions** tab for error logs
2. Common issues:
   - Dockerfile syntax errors
   - Missing dependencies in build context
   - Network issues during dependency download

### Image Not Published

1. Verify the branch name matches the workflow triggers
2. Check that the workflow has `packages: write` permission
3. Ensure the push event (not PR) triggered the workflow

### Cannot Pull Image

1. Verify the image tag exists in GHCR
2. For private repos, ensure you're authenticated
3. Check image name spelling: `ghcr.io/opendataensemble/synkronus`

## Best Practices

### For Developers

1. **Test locally first**: Build and test Docker images locally before pushing
2. **Use feature branches**: Create feature branches for experimental changes
3. **Review build logs**: Check Actions logs even for successful builds
4. **Tag releases properly**: Use semantic versioning for releases

### For Deployments

1. **Pin versions in production**: Use specific version tags, not `latest`
2. **Test pre-releases**: Use `develop` tag for staging environments
3. **Monitor image sizes**: Keep images lean for faster deployments
4. **Use health checks**: Always configure health checks in deployments

## Future Enhancements

Potential improvements to the CI/CD pipeline:

- [ ] Add automated testing before build
- [ ] Implement security scanning (Trivy, Snyk)
- [ ] Add deployment to staging environment
- [ ] Create release notes automation
- [ ] Add Slack/Discord notifications
- [ ] Implement rollback mechanisms
- [ ] Add performance benchmarking

## Related Documentation

- [Root README](../README.md) - Monorepo overview
- [Synkronus DOCKER.md](../synkronus/DOCKER.md) - Docker quick start
- [Synkronus DEPLOYMENT.md](../synkronus/DEPLOYMENT.md) - Comprehensive deployment guide
