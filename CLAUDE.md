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
- **Android API Levels**: 34 and 35 (build tools and platforms)
- **Supported Platforms**: linux/amd64 and linux/arm64

### Key Environment Variables
- `ANDROID_SDK_ROOT=/sdk` - Android SDK location
- `ANDROID_NDK_HOME=/ndk` - Android NDK location
- `JAVA_HOME` - Must be set by users to select JDK version

### Multi-Architecture Support
The Dockerfile includes special handling for aarch64/arm64 architectures:
- Native x86_64 build tools don't work on ARM
- For Android 34 and 35, ARM-specific build tools are downloaded from https://github.com/lzhiyong/android-sdk-tools
- Detection happens at build time using `uname -m`

### SDK Package Management
SDK packages are defined in `pkg.txt` and installed via `sdkmanager`. Currently includes:
- Build tools 34.0.0 and 35.0.0
- Platforms android-34 and android-35
- Google Play Services, Android and Google m2repository
- Google APIs add-on for API 24

## Build and Development

### Building the Docker Image Locally
```bash
docker build -t gitlab-ci-android:local .
```

### Building for Multiple Platforms
```bash
docker buildx build --platform linux/amd64,linux/arm64 -t gitlab-ci-android:local .
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
1. Builds multi-platform images on every push
2. Publishes to Docker Hub with tags: `latest`, `35`, `34`
3. Requires `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets

## Modifying SDK Components

### Adding New SDK Packages
1. Edit `pkg.txt` to add SDK package identifiers
2. Find available packages: `sdkmanager --list --sdk_root=${ANDROID_SDK_ROOT}`
3. Rebuild the Docker image

### Adding New Android API Level
1. Add build-tools and platforms entries to `pkg.txt`
2. If targeting aarch64, update `ANDROID_XX_BUILD_TOOLS_X86_VERSION` and `ANDROID_XX_BUILD_TOOLS_AARCH64_VERSION` environment variables
3. Add corresponding installation logic in the Dockerfile's aarch64 section (lines 93-110)
4. Update Docker image tags in `.github/workflows/build.yml`

### Updating NDK Version
1. Change `NDK_VERSION` environment variable in Dockerfile
2. Verify the download URL is still valid for the new version

## Important Notes

- The image includes **all** JDK versions (8, 11, 17, 21). Users select which to use via `JAVA_HOME` environment variable in their CI configuration
- Android SDK command-line tools version is dynamically fetched from the latest available version on Google's website
- SDK licenses are pre-accepted in the image build process
- The `repo` tool (Google's repo tool for AOSP) is included and verified with SHA256 checksum
