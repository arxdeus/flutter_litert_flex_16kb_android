# TFLite Flex Delegate for Android (16KB Page Size)

CI repository for building `libtensorflowlite_flex.so` with 16KB page size support for Android.

## Why

Starting with Android 15, arm64 devices may use 16KB page size instead of 4KB.
Google Play [requires](https://developer.android.com/guide/practices/page-sizes) 16KB compatibility for new apps.

Pre-built `.so` files from TensorFlow do not support 16KB page alignment.
This repository builds the flex delegate with the correct linker flags:
- `-Wl,-z,max-page-size=16384`
- `-Wl,-z,common-page-size=16384`

## Architectures

| ABI | Description |
|-----|-------------|
| `arm64-v8a` | 64-bit ARM (modern devices) |
| `armeabi-v7a` | 32-bit ARM (legacy devices) |

## Running the Build

**Manual:** Actions → *Build TFLite Flex Delegate for Android* → Run workflow

You can specify a TensorFlow git ref (branch, tag, or SHA). Defaults to `master`.

**Automatic:** triggers on changes to `patches/` or the workflow file.

## Artifacts & Releases

Each build creates a **GitHub Release** tagged `tf-<commit-hash>` with two `.so` files attached:
- `libtensorflowlite_flex-arm64-v8a.so`
- `libtensorflowlite_flex-armeabi-v7a.so`

## Replacing `.so` in a project using `flutter_litert_flex`

The [`flutter_litert_flex`](https://github.com/hugocornellier/flutter_litert_flex) package
pulls in `org.tensorflow:tensorflow-lite-select-tf-ops` via Maven, which bundles its own
`libtensorflowlite_flex.so` **without** 16KB page alignment.

There are two ways to replace it with our 16KB-aligned build:

---

### Option A — Automatic download via `build.gradle.kts` (recommended)

Add the following to your **app-level** `android/app/build.gradle.kts`.
Gradle will download the `.so` files from GitHub Releases on first build
and cache them in the build directory.

```kotlin
import java.net.URI

// --- Flex Delegate 16KB config ---
val flexDelegateRepo = "YOUR_USERNAME/flex_delegate_android_16kb"  // ← your repo
val flexDelegateTag = "latest"  // or a specific tag like "tf-73ef2dec449a"
val flexDelegateAbis = listOf("arm64-v8a", "armeabi-v7a")
val flexDelegateCacheDir = layout.buildDirectory.dir("flex-delegate")

val downloadFlexDelegate by tasks.registering {
    val cacheDir = flexDelegateCacheDir.get().asFile
    outputs.dir(cacheDir)

    onlyIf {
        flexDelegateAbis.any { abi ->
            !File(cacheDir, "$abi/libtensorflowlite_flex.so").exists()
        }
    }

    doLast {
        val baseUrl = if (flexDelegateTag == "latest") {
            "https://github.com/$flexDelegateRepo/releases/latest/download"
        } else {
            "https://github.com/$flexDelegateRepo/releases/download/$flexDelegateTag"
        }

        flexDelegateAbis.forEach { abi ->
            val target = File(cacheDir, "$abi/libtensorflowlite_flex.so")
            if (!target.exists()) {
                val url = "$baseUrl/libtensorflowlite_flex-$abi.so"
                logger.lifecycle("⬇ Downloading flex delegate ($abi)...")
                target.parentFile.mkdirs()
                URI(url).toURL().openStream().use { input ->
                    target.outputStream().use { output -> input.copyTo(output) }
                }
                logger.lifecycle("  ✅ ${target.length() / 1_048_576} MB → $target")
            }
        }
    }
}

android {
    // ... your existing config ...

    defaultConfig {
        ndk {
            abiFilters += flexDelegateAbis
        }
    }

    sourceSets.getByName("main") {
        jniLibs.srcDir(flexDelegateCacheDir)
    }

    packaging {
        jniLibs {
            pickFirsts += flexDelegateAbis.map { "lib/$it/libtensorflowlite_flex.so" }
        }
    }
}

tasks.configureEach {
    if (name.startsWith("merge") && name.endsWith("NativeLibs")) {
        dependsOn(downloadFlexDelegate)
    }
}
```

**How it works:**
- On first `flutter build apk`, Gradle downloads both `.so` files (~110 MB each) into `build/flex-delegate/`
- Subsequent builds reuse the cache (files survive until `flutter clean`)
- `sourceSets.jniLibs.srcDir` adds the cache dir as a native library source
- `pickFirsts` resolves the conflict with the Maven-bundled `.so`
- Nothing is committed to git — the build directory is already in `.gitignore`

To force re-download: `cd android && ./gradlew clean`

---

### Option B — Manual download

#### Step 1 — Add the custom `.so` files

Download the `.so` files from the [latest release](../../releases/latest) and place them:

```
your_app/android/app/src/main/jniLibs/
├── arm64-v8a/libtensorflowlite_flex.so
└── armeabi-v7a/libtensorflowlite_flex.so
```

> Remember to rename the files — drop the `-arm64-v8a` / `-armeabi-v7a` suffix.

#### Step 2 — Configure Gradle

In `android/app/build.gradle.kts`:

```kotlin
android {
    defaultConfig {
        ndk {
            abiFilters += listOf("arm64-v8a", "armeabi-v7a")
        }
    }

    packaging {
        jniLibs {
            pickFirsts += listOf(
                "lib/arm64-v8a/libtensorflowlite_flex.so",
                "lib/armeabi-v7a/libtensorflowlite_flex.so",
            )
        }
    }
}
```

<details>
<summary>Groovy version (build.gradle)</summary>

```groovy
android {
    defaultConfig {
        ndk {
            abiFilters 'arm64-v8a', 'armeabi-v7a'
        }
    }

    packagingOptions {
        pickFirst 'lib/arm64-v8a/libtensorflowlite_flex.so'
        pickFirst 'lib/armeabi-v7a/libtensorflowlite_flex.so'
    }
}
```

</details>

#### Step 3 — Add to `.gitignore` (optional)

The `.so` files are ~110 MB each. You may want to gitignore them:

```gitignore
android/app/src/main/jniLibs/**/libtensorflowlite_flex.so
```

---

### Verifying the replacement

Build the APK and confirm the correct `.so` is bundled:

```bash
flutter build apk

# Check the .so inside the APK
unzip -l build/app/outputs/flutter-apk/app-release.apk | grep libtensorflowlite_flex

# Verify 16KB alignment
unzip -o build/app/outputs/flutter-apk/app-release.apk \
  lib/arm64-v8a/libtensorflowlite_flex.so -d /tmp
readelf -l /tmp/lib/arm64-v8a/libtensorflowlite_flex.so | grep -i align
```

The LOAD segment alignment should show `0x4000` (16384).

### How it works

The `flutter_litert_flex` plugin declares a Maven dependency on
`org.tensorflow:tensorflow-lite-select-tf-ops`, which provides:
- **Java class** `org.tensorflow.lite.flex.FlexDelegate` — still needed, kept as-is
- **Native library** `libtensorflowlite_flex.so` — replaced by our 16KB-aligned build

The `pickFirsts` directive resolves the duplicate: Gradle picks our copy
and discards the one from the AAR.

## Patches

### `fix-android-flex-delegate-build.patch`

Fixes **duplicate symbol** linker errors when building the flex delegate for Android.

**Root cause:** `portable_tensorflow_lib_lite` unconditionally depends on `onednn_env_vars`
(Intel MKL — useless on ARM), which pulls in `//tensorflow/core:framework` (the desktop
framework library). On Android, `framework_internal_impl` gets linked via `if_static()`,
duplicating symbols already present in `portable_tensorflow_lib_lite`
(`graph_def_util.cc`, `kernel_def_builder.cc`, `graph_to_functiondef.cc`, etc.).

**Fix:**
- Guard `onednn_env_vars` behind `if_not_mobile()`
- Add explicit `protobuf` and `executor` deps that were previously transitive through `framework_internal_impl`

## Versions

| Component | Version |
|-----------|---------|
| Android NDK | r28c (28.2.13676358) |
| Android SDK | API 36 |
| Build Tools | 36.1.0 |
| Bazel | 7.7.0 (from TF `.bazelversion`) |
| Python | 3.12 (hermetic) |

## Repository Structure

```
├── .github/workflows/
│   └── build-flex-delegate-android.yml   # GitHub Actions workflow
├── patches/
│   └── fix-android-flex-delegate-build.patch
└── README.md
```
