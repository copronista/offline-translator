# AGENTS.md

## What this is
- Android on-device translator: text, documents, images, OCR, dictionary, TTS, browser reader mode.
- Kotlin + Jetpack Compose app with a public AIDL API and Rust native libraries.
- This clone tracks the `copronista/offline-translator` fork of `DavidVentura/offline-translator`;
  README badges/release links still point upstream.
- Application id / package is `dev.davidv.translator` (not the workspace directory name).

## Stack facts
- `minSdk 23`, `compileSdk/targetSdk 36`, Java/Kotlin target 11.
- Gradle 8.11.1, AGP 8.10.1, Kotlin 2.2.21, NDK `28.0.13004108`.
- Rust toolchain `1.88.0`, `cargo-ndk 4.1.2`.
- Runtime model inference is MNN-only. Do not expect `.onnx` to load at runtime;
  ONNX files go through the on-device migration/conversion path.
- Translation models come from Mozilla/Firefox; OCR is PP-OCR/PaddleOCR;
  TTS uses Piper/Coqui/Kokoro/MMS/Sherpa/Mimic3 voices; dictionaries come from Kaikki/Wiktionary.

## Build
- Canonical CI-equivalent build: `bash build.sh` (builds the `Dockerfile` image, then runs
  `./gradlew assembleRelease`).
- Direct Gradle builds require `ANDROID_SDK_ROOT` or `ANDROID_HOME`, the NDK, Rust, and `cargo-ndk`.
  `preBuild` builds `libbindings.so`, `libmodel_converter.so`, and generates UniFFI Kotlin.
- Default dev ABIs are `arm64-v8a` and `x86_64`; select one with `-PtargetAbi=arm64-v8a`.
- Release/CI builds only `arm64-v8a` and `armeabi-v7a`. A local `build.sh` does not produce the
  armv7 release APK.
- APKs land in `app/build/outputs/apk/`.
- `local.properties` (`sdk.dir=...`) is gitignored; create it or set `ANDROID_SDK_ROOT`/`ANDROID_HOME`.
- `app/src/main/jniLibs/`, `signed/`, `build/`, and `.kotlin/` are generated/ignored.

## Verification
- Unit tests: `./gradlew testDebugUnitTest`.
- Single test: `./gradlew testDebugUnitTest --tests 'dev.davidv.translator.InputSplitTest'`.
- Format: `./gradlew ktlintFormat`; check: `./gradlew ktlintCheck`.
- Detekt: `./gradlew detekt detektHardcodedStrings`. `detekt` is advisory (`maxIssues: -1`);
  `detektHardcodedStrings` is blocking.
- Lint suites: `./gradlew lintAll` (ktlint + detekt) or `make check` (detekt + hardcoded strings, no ktlint).
- NewApi-only lint (slow, release gate): `make newapi` ->
  `./gradlew :app:lintDebug -PtargetAbi=arm64-v8a -PlintNewApiOnly`.
- Instrumented tests need a device/emulator; `app/src/androidTest/assets` is ~85 MB.
  Use `./gradlew connectedAndroidTest`.
- `make screenshots` also needs a connected device and is only for the Weblate UI-export workflow.

## Lint/style rules
- No `println` under `app/src/main`; `make lint` greps for it.
- Hardcoded user-facing strings are rejected by the custom `NoHardcodedUiString` detekt rule
  (source in `detekt-rules/`). Use `res/values*/strings.xml`.
- Translations are managed in Weblate. `MissingTranslation` is intentionally disabled;
  partial locales are normal.
- Follow `.editorconfig`: 2-space indent, max line length 140, final newline.
- Write code, comments, and commit messages in English.

## Release flow
- Bump `versionName` and `versionCode` in `app/build.gradle.kts`.
- Per-ABI versionCode is `versionCode * 10 + offset`: armv7=1, arm64=2, x86=3, x86_64=4.
- Add changelogs at `fastlane/metadata/android/en-US/changelogs/${versionCode*10+1}.txt` and `+2`.
- `./fetch-release-apks.sh` downloads CI release artifacts into `app/build/outputs/apk/ci-release/`.
- `./sign-apk.sh <keystore> <storepass> <keypass> <alias>` signs only the fetched CI APKs and writes
  `signed/translator-{arm64,armv7}-<version>.apk`. It needs `ANDROID_SDK_ROOT` with build-tools 34.0.0.
- Do not point `sign-apk.sh` at local `app/build/outputs/apk/release` outputs; fetch CI artifacts first.
- Tag as `v${versionName}` and upload both arm64 and armv7 APKs.

## Models, catalogs, offline packs
- Bundled catalog: `app/src/main/assets/index_v7.json`. Do not hand-edit; regenerate it.
- Internal catalog: `python3 generate_index.py` (default mode, writes `catalog_sources/source_catalog.json`).
- Public catalog: `python3 generate_index.py --mode public --base-url https://offline-translator.davidv.dev`
  (writes `app/src/main/assets/index_v7.json`).
- `regen_index.sh` runs internal + bucket mirror + public and expects an external `../bucket/`.
- `download_offline.py` installs built packs into a `dev.davidv.translator/` directory for a device;
  `download_bucket.py` is a maintainer mirroring tool, not an installer.
- On-device files live under `Documents/dev.davidv.translator/`; see `OFFLINE_SETUP.md`.
- `new_lang.md` is the guide for adding languages, voices, and dictionaries. It references
  `translator-rs`, which is a separate repo and is not vendored here.

## Native/codegen layout
- `app/src/main/bindings` -> `libbindings.so` (slimt, MNN OCR/TTS, adblock, UniFFI bindings).
- `app/src/main/model-converter` -> `libmodel_converter.so` (ONNX->MNN; separate cdylib because its
  protobuf dependency conflicts with slimt).
- `app/bindings-bindgen` -> host `uniffi-bindgen` binary.
- Generated Kotlin bindings: `app/build/generated/source/uniffi/kotlin`. Never edit generated files.
- Local dev can use `path =` dependencies into a sibling `translator-rs` checkout; CI uses git dependencies.

## Architecture/entrypoints
- `TranslatorApplication` wires settings, catalog, downloads, translation, speech, detection, and adblock.
- `MainActivity` is the Compose UI entrypoint. `ProcessTextActivity` handles `PROCESS_TEXT`/`TRANSLATE`;
  `BrowserActivity` handles `VIEW`/reader mode.
- Services:
  - `AidlTranslationService` (AIDL API in `app/src/main/aidl/`).
  - `TranslatorTtsService` (system TTS engine).
  - `TranslationApiService` (LibreTranslate-compatible HTTP server).
  - `DownloadService`, `DocumentTranslationService`, `ScreenTranslateService`.
- Feature packages: `ui`, `browser`, `screenTranslate`, `accessibilityOverlay`, `assistantOverlay`,
  `overlayChrome`, `adblock`.
- Catalog/download/pack logic: `CatalogModels`, `CatalogRepository`, `DownloadManager`, `PackInstaller`,
  `LanguageMetadataManager`.
- Unit tests: `app/src/test`; instrumented and UI-export tests: `app/src/androidTest`.

## Known fork gotchas
- `scripts/` does not exist in this fork, but `Makefile` and `.pre-commit-config.yaml` reference
  `scripts/check_extra_translations.py` and `scripts/render_svg_sections.py`. `make lint` and
  `make screenshots` fail at those steps; run the Gradle targets directly.
- The `extra-translations` pre-commit hook fails for the same missing-script reason.
- `OFFLINE_SETUP.md` still says `index_v6.json`; the executable `download_offline.py` defaults to
  `index_v7.json`.
- `fetch-release-apks.sh` defaults to the latest tag via `git describe --tags`, which resolves the local
  `v0.8.8` tag; that tag points at an unpushed commit with no CI run. Pass a pushed commit/ref explicitly.
