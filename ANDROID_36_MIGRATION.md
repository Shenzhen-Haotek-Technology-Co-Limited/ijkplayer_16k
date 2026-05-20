# Android 36 Compatibility Migration

## Summary
Successfully migrated ijkplayer to target Android 36 (API level 36) with arm64-only support.

## Changes Made

### 1. SDK Configuration
- **compileSdkVersion**: 36
- **targetSdkVersion**: 36
- **buildToolsVersion**: 36.0.0
- **minSdkVersion**: 21 (required for arm64)

### 2. Architecture Support
- Removed all architectures except **arm64-v8a**
- Removed product flavors (all32, all64)
- Added `ndk.abiFilters "arm64-v8a"` to ensure only arm64 libraries are packaged

### 3. AndroidX Migration
- Migrated from deprecated support libraries to AndroidX
- Updated dependencies:
  - `com.android.support:appcompat-v7:23.0.1` → `androidx.appcompat:appcompat:1.6.1`
  - `com.android.support:preference-v7:23.0.1` → `androidx.preference:preference:1.2.1`
  - `com.android.support:support-annotations:23.0.1` → `androidx.annotation:annotation:1.7.1`
- Enabled AndroidX in gradle.properties:
  - `android.useAndroidX=true`
  - `android.enableJetifier=true`

### 4. Permissions Updates (AndroidManifest.xml)
- Limited `WRITE_EXTERNAL_STORAGE` to API 28 and below: `android:maxSdkVersion="28"`
- Added modern media permissions for Android 13+:
  - `READ_MEDIA_VIDEO`
  - `READ_MEDIA_AUDIO`
- Added `android:enableOnBackInvokedCallback="true"` for predictive back gesture (Android 33+)
- Added `android:requestLegacyExternalStorage="true"` for compatibility

### 5. Source Code Migration
- Migrated 22 Java files from support library imports to AndroidX
- Key import replacements:
  - `android.support.v4.app.Fragment` → `androidx.fragment.app.Fragment`
  - `android.support.v4.content.Loader` → `androidx.loader.content.Loader`
  - `android.support.v7.app.*` → `androidx.appcompat.app.*`
  - `android.support.annotation.*` → `androidx.annotation.*`

### 6. Build Optimization
- Enabled parallel builds: `org.gradle.parallel=true`
- Enabled build caching: `org.gradle.caching=true`
- Optimized JVM args: `org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8`

## Build Instructions

### Build Debug APK
```bash
cd android/ijkplayer
./gradlew :ijkplayer-example:assembleDebug
```

### Install to Device
```bash
./gradlew :ijkplayer-example:installDebug
```

### Output Location
```
android/ijkplayer/ijkplayer-example/build/outputs/apk/debug/ijkplayer-example-debug.apk
```

## Modules Updated
- `ijkplayer-example` - Main example app
- `ijkplayer-java` - Core Java player interface
- `ijkplayer-exo` - ExoPlayer integration
- `ijkplayer-arm64` - Native arm64 libraries

## Testing Checklist
- [ ] App launches without crashes
- [ ] Native libraries load correctly (libijkffmpeg.so, libijksdl.so, libijkplayer.so)
- [ ] Video playback works
- [ ] Media permissions are requested properly on Android 13+
- [ ] Back gesture works correctly
- [ ] File access works with scoped storage

## Known Issues
- ExoPlayer version r1.5.11 is very old and may need updating for full Android 36 compatibility
- Legacy external storage flag is a temporary compatibility measure

## Next Steps
Consider updating:
1. ExoPlayer to latest version (2.x or 3.x)
2. Gradle plugin to latest stable version
3. Remove legacy storage compatibility once scoped storage is fully implemented
