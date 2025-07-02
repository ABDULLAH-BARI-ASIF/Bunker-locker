# Android Keystore Build Issue Fix

## Problem
The Android build was failing with the following error:
```
> A failure occurred while executing com.android.build.gradle.tasks.PackageAndroidArtifact$IncrementalSplitterRunnable.
> com.android.ide.common.signing.KeytoolException: Failed to read key from store "/home/runner/work/Bunker-locker/Bunker-locker/sign_key.jks": Tag number over 30 is not supported.
```

## Root Cause
The error "Tag number over 30 is not supported" occurs when a JKS (Java KeyStore) file contains format tags that are incompatible with the keytool version being used. This typically happens when:

1. **JKS Format Incompatibility**: The keystore was created with JDK 8+ which introduced newer cryptographic features and tags
2. **Version Mismatch**: The Android build tools or keytool version doesn't support the newer JKS format features
3. **Format Evolution**: JKS format has evolved over time, and newer versions use tags > 30 that older tools can't handle

## Solution Implemented

### Changes Made
Modified both GitHub Actions workflow files:
- `.github/workflows/release.yml`
- `.github/workflows/main.yml`

### Fix Details
1. **Keystore Format Conversion**: Added a step to convert the JKS keystore to PKCS12 format during the build process
2. **Temporary File Handling**: The original JKS file is decoded to a temporary file, converted, then removed
3. **Updated Configuration**: Changed the keystore file reference from `sign_key.jks` to `sign_key.p12`

### Technical Implementation
```bash
# Decode the base64 keystore to a temporary JKS file
echo '${{ secrets.KEY_STORE }}' | base64 --decode > temp_sign_key.jks

# Convert JKS to PKCS12 format using keytool
keytool -importkeystore \
  -srckeystore temp_sign_key.jks \
  -srcstoretype JKS \
  -srcstorepass '${{ secrets.KEY_STORE_PASSWORD }}' \
  -destkeystore sign_key.p12 \
  -deststoretype PKCS12 \
  -deststorepass '${{ secrets.KEY_STORE_PASSWORD }}' \
  -srcalias '${{ secrets.ALIAS }}' \
  -destalias '${{ secrets.ALIAS }}' \
  -srckeypass '${{ secrets.KEY_PASSWORD }}' \
  -destkeypass '${{ secrets.KEY_PASSWORD }}' \
  -noprompt

# Clean up temporary file
rm temp_sign_key.jks
```

## Why PKCS12?
1. **Modern Standard**: PKCS12 is the recommended keystore format for modern Android development
2. **Better Compatibility**: More widely supported across different Java versions and Android build tools
3. **Future-Proof**: Less likely to encounter compatibility issues with newer Android Gradle Plugin versions

## Expected Results
- ✅ Build process should complete successfully
- ✅ APK signing should work correctly
- ✅ No more "Tag number over 30" errors
- ✅ Maintained compatibility with existing secrets and configuration

## Alternative Solutions (Not Implemented)
1. **Recreate Original Keystore**: Generate a new JKS keystore with older format (not recommended for security)
2. **Downgrade JDK**: Use an older JDK version (not recommended for security and feature support)
3. **Update Android Gradle Plugin**: Sometimes newer versions handle JKS better (current version 8.11.0 is already latest)

The implemented solution provides the best balance of compatibility, security, and maintainability.