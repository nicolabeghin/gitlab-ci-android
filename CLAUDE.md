# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository builds a Docker image (`inovex/gitlab-ci-android`) designed for building Android applications in CI/CD environments, particularly GitLab CI. The image contains the Android SDK, Android NDK, and necessary build tools for Android development.

**Docker Hub**: https://hub.docker.com/r/inovex/gitlab-ci-android/

## Architecture

### Base Configuration
- **Base Image**: Ubuntu 20.04
- **Android NDK**: r25c
- **Supported JDK Versions**: 8, 11, 17, and 21 (all included in the image)
- **Default Android API Level**: 35 (configurable via `ANDROID_LEVEL` build argument)
- **Supported Platforms**: linux/amd64 and linux/arm64

### Key Environment Variables
- `ANDROID_SDK_ROOT=/sdk` - Android SDK location
- `ANDROID_NDK_HOME=/ndk` - Android NDK location
- `JAVA_HOME` - Must be set by users to select JDK version

### Multi-Architecture Support
The Dockerfile includes special handling for aarch64/arm64 architectures:
- Native x86_64 build tools don't work on ARM
- ARM-specific build tools are automatically downloaded from https://github.com/lzhiyong/android-sdk-tools
- The correct version is determined by querying the GitHub releases API for releases matching the specified Android level
- Detection happens at build time using `uname -m`
- If no matching aarch64 build tools are found for the specified Android level, the build will fail with a clear error message

### SDK Package Management
SDK packages are dynamically generated based on the `ANDROID_LEVEL` build argument. The generated package list includes:
- Build tools for the specified Android level (e.g., `build-tools;35.0.0`)
- Platform for the specified Android level (e.g., `platforms;android-35`)
- Google Play Services
- Android and Google m2repository
- Google APIs add-on for API 24

**Note**: The `pkg.txt` file is no longer used - packages are generated dynamically during the Docker build.

## Build and Development

### Building the Docker Image Locally
```bash
# Build with default Android level (35)
docker build -t gitlab-ci-android:local .

# Build with specific Android level
docker build --build-arg ANDROID_LEVEL=34 -t gitlab-ci-android:34 .
docker build --build-arg ANDROID_LEVEL=33 -t gitlab-ci-android:33 .
```

### Building for Multiple Platforms
```bash
# Build for multiple platforms with default Android level
docker buildx build --platform linux/amd64,linux/arm64 -t gitlab-ci-android:local .

# Build for multiple platforms with specific Android level
docker buildx build --build-arg ANDROID_LEVEL=34 --platform linux/amd64,linux/arm64 -t gitlab-ci-android:34 .
```

### Testing the Image
```bash
# Run interactive shell in the image
docker run -it gitlab-ci-android:local /bin/bash

# Verify SDK installation
docker run gitlab-ci-android:local ${ANDROID_SDK_ROOT}/cmdline-tools/bin/sdkmanager --list --sdk_root=${ANDROID_SDK_ROOT}

# Verify NDK installation
docker run gitlab-ci-android:local ls -la ${ANDROID_NDK_HOME}
```

### CI/CD Pipeline
GitHub Actions workflow (`.github/workflows/build.yml`) automatically:
1. Builds images for Android levels 33, 34, and 35 on every push using a matrix strategy
2. Each build creates multi-platform images (linux/amd64 and linux/arm64)
3. Publishes to Docker Hub with level-specific tags (`:33`, `:34`, `:35`)
4. Tags Android level 35 as `:latest`
5. Requires `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets

To add support for a new Android level, update the matrix in `.github/workflows/build.yml`:
```yaml
strategy:
  matrix:
    android_level: [33, 34, 35, 36]  # Add new level here
```

## Modifying SDK Components

### Adding New SDK Packages
SDK packages are dynamically generated in the Dockerfile. To add new packages:
1. Edit the Dockerfile's package generation section (around line 68-74)
2. Add new `echo` statements for additional packages
3. Find available packages: `sdkmanager --list --sdk_root=${ANDROID_SDK_ROOT}`
4. Rebuild the Docker image

Example:
```dockerfile
echo "extras;android;m2repository" >> /sdk/pkg.txt && \
echo "ndk;25.2.9519653" >> /sdk/pkg.txt  # Add new package
```

### Building for a New Android API Level
The Dockerfile now supports any Android API level via the `ANDROID_LEVEL` build argument:

```bash
docker build --build-arg ANDROID_LEVEL=36 -t gitlab-ci-android:36 .
```

**Requirements for aarch64 support**:
- A matching release must exist at https://github.com/lzhiyong/android-sdk-tools/releases
- The release tag must follow the pattern `v{ANDROID_LEVEL}.X.Y` (e.g., `v36.0.0`)
- If no matching release exists, the build will fail with an error message during aarch64 build tools installation

**To add a new level to CI/CD**:
1. Verify aarch64 build tools are available on GitHub
2. Add the level to the matrix in `.github/workflows/build.yml`

### Updating NDK Version
1. Change `NDK_VERSION` environment variable in Dockerfile
2. Verify the download URL is still valid for the new version at https://developer.android.com/ndk/downloads

## Important Notes

- The image includes **all** JDK versions (8, 11, 17, 21). Users select which to use via `JAVA_HOME` environment variable in their CI configuration
- Android SDK command-line tools version is dynamically fetched from the latest available version on Google's website
- SDK licenses are pre-accepted in the image build process
- The `repo` tool (Google's repo tool for AOSP) is included and verified with SHA256 checksum
- **pkg.txt is now dynamically generated** during the Docker build based on the `ANDROID_LEVEL` build argument - the static file is no longer used
- For aarch64/arm64 platforms, the build will query GitHub's API at build time to find the correct build tools version for the specified Android level
- The `ANDROID_LEVEL` build argument defaults to 35 if not specified
