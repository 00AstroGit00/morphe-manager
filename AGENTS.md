# AGENTS.md

Morphe Manager — Android app (Kotlin, Jetpack Compose, AGP 9.4.1, Gradle 9.8.2, `minSdk` 26, `compileSdk` 37) that patches YouTube / YouTube Music / Reddit APKs at the bytecode level. No root required. `namespace`/`applicationId` `app.morphe.manager` (debug suffix `.debug`).

## Branch model

- `main` — release branch. Pushes trigger semantic-release + APK publish, website deploy, optional FCM.
- `dev` — development branch. PRs target `dev`; a workflow auto-opens a draft PR `dev` → `main`.
- Conventional Commits only (`feat`, `fix`, `perf`, `revert`); release notes are generated from them.

## Modules & entry points

- `:app` — Android application. `app.morphe.manager.ManagerApplication` (Koin DI; short-circuits for the `:patcher` process), `MainActivity` (singleTask; handles `.mpp` bundles, share intents, `BATCH_PATCH`, `morphe.software/add-source` deep links). UI is Compose; DI modules in `app/.../di/` (Koin). Data: Room `AppDatabase` (`app/schemas/app.morphe.manager.data.room.AppDatabase`, currently schema v11) + migrations in `data/room/Migrations.kt`; Ktor networking; WorkManager (initializer removed from manifest; app wires it).
- `:patcher-core` — pure JVM library (no Android): the patcher host that runs inside the embedded JVM. Same code runs in-app and in the embedded runtime. Key flow: `Session`/`PatchPipeline`/`HostMain` ↔ `HostCommand`/`HostEvent`.
- `../morphe-patcher` — optional composite build. If the sibling checkout exists, `settings.gradle.kts` substitutes `app.morphe:morphe-patcher` with it. Local edits there take effect without publishing.

## Patching process architecture (high impact)

- Patching runs in a separate service/process: `app.morphe.manager.patcher.runtime.PatcherService` (`android:process=":patcher"`). `PatcherProcesses` manages it. `ManagerApplication.onCreate` returns early in `:patcher` (Firebase/DI must not run there). Manifest sets `allowNativeHeapPointerTagging="false"` and `largeHeap="true"` (HotSpot/bionic constraint for the embedded JVM).
- Native launcher: `app/src/main/cpp/jvm_launcher.cpp` (CMake). AIDL in `app/src/main/aidl/.../patcher/runtime/`. Root/mount assets in `app/src/main/assets/root/` (LF endings enforced by `.gitattributes`).

## Build & test commands

```bash
./gradlew :app:testDebugUnitTest          # JVM unit tests for :app
./gradlew :patcher-core:test              # JVM unit tests for patcher-core
./gradlew :app:lint                        # the only lint gate (lintVital is disabled in-app)
./gradlew assembleDebug                   # unsigned build (fork PRs / local)
./gradlew assembleRelease                 # signed release (needs keystore env + JVM toolchains + Android SDK)
```

CI (`.github/workflows/build_pull_request.yml`) runs: `:app:testDebugUnitTest` → `:app:lint` → `assembleDebug` (forks) / `assembleRelease` (trusted).

- Unit tests only; no instrumentation tests. `testOptions.unitTests.isReturnDefaultValues = true` (Android SDK stubs). `app/src/test/.../TranslationFormatTest` validates Crowdin placeholders against source strings (crash prevention).
- Room schemas: `app/schemas/app.morphe.manager.data.room.AppDatabase/` (current v11). Bump schema + add migration in `data/room/Migrations.kt` when changing entities.
- `gradle.properties`: JVM args, toolchain env (`JAVA_HOME_17_X64`, `JAVA_HOME_25_X64`), caching on, configuration-cache **off**.

## Toolchains & the embedded JVM (the tricky part)

- Embeds OpenJDK Android builds (AngelAuraMC) for `arm64-v8a` (JDK 25), `armeabi-v7a` (JDK 17), `x86_64` (JDK 25), pinned by SHA-256 in `app/build.gradle.kts` (`embeddedJvms`, `embeddedJvmModules/Files`).
- `buildSrc` tasks: `PrepareEmbeddedJvmTask` (download/link → `jvm/<abi>.zip.xz`), `EmbeddedJvmAssetsTask`, `ImageDelta`/`XzArchive`. Changing the JDK or adding an ABI: edit `embeddedJvms`; `EMBEDDED_JVM_ID` changes automatically and the app re-unpacks.
- `:patcher-core` produces a shrunk host jar (`ShrinkJarTask`, `patcher-core/host-rules.pro`). **A new library a patch calls into must be kept in `patcher-core/host-rules.pro` and in `app/proguard-rules.pro`/R8 rules.**
- JDK 17 and 25 are required toolchains (17 also shrinks the host). JDK 21 runs Gradle. CI installs all three via `setup-java`. In this Termux environment only JDK 17/21 are present and the Android SDK is not configured, so full Android builds/lint are not locally reproducible here.

## Release flow (don't improvise)

1. Bump `version` in `app/gradle.properties`.
2. `.github/scripts/prepare-release.sh <version>` — updates the version, runs `assembleRelease`, renames the APK to `morphe-manager-<version>.apk`, GPG-signs it.
3. semantic-release (`.releaserc`) publishes: updates `CHANGELOG.md` + `app-release.json`, uploads the APK + `.asc` to GitHub, backmerges `main` → `dev`, and triggers the `morphe-website` deploy.
4. Release signing comes from secrets: `KEYSTORE_B64` → `app/keystore.jks`, plus `KEYSTORE_PASSWORD` / `KEYSTORE_ENTRY_ALIAS` / `KEYSTORE_ENTRY_PASSWORD`.

## Dependencies & licensing

- `morphe-patcher` resolves from GitHub Packages (`maven.pkg.github.com/MorpheApp/registry`) — rejects anonymous reads. Local builds need `gpr.user`/`gpr.key` in `local.properties` or Gradle props, or `GITHUB_ACTOR`/`GITHUB_TOKEN` in env (see `settings.gradle.kts`).
- **Version pinning (verified 2026-11-08 via GitHub releases/docs)**: `gradle/libs.versions.toml` pins `morphe-patcher = 2.3.0` (matches latest stable `v2.3.0`; `v2.3.1-dev.1` pre-release exists upstream). Kotlin `2.4.21` is current on the 2.4 line; KSP (`dev-tools-gradle-plugin`) is `2.3.12` (Google KSP shows `2.3.13` available — a minor bump; validate against Kotlin/AGP/Room before changing). Manager is at `v2.5.0` (`app/gradle.properties`, `app-release.json`). Dependency bumps should be CI-validated (AGP/Kotlin/KSP/Compose/embedded-JVM toolchain are tightly coupled).
- Licensee gates the build: only `0BSD`, `Apache-2.0`, `BSD-3-Clause`, or explicitly-allowed deps ship. JitPack deps (`ARSCLib`, `libsu`) are whitelisted by hand in `app/licenses/licenses.json`. `app/google-services.json` is committed (FCM config); Firebase Analytics is disabled in the manifest.
- `package.json` holds only semantic-release tooling; `npm ci` is part of the release workflow.

## Translations & release assets

- Crowdin (`crowdin.yml`): sources `app/src/main/res/values/{strings,plurals}.xml` → `values-%android_code%`. Push workflow on `dev` path changes; `TranslationFormatTest` fails if a translation's placeholders would crash at runtime.
- Release assets tracked/generated: `CHANGELOG.md`, `app-release.json`, `app/gradle.properties` (version). PR template requires linking an open issue (`Closes #`).

## Gotchas

- `local.properties` is gitignored and not committed; missing it breaks `morphe-patcher` resolution locally.
- `lintVital` is disabled — changing lint rules has no effect on release builds; `:app:lint` is the real gate.
- Fork PRs build unsigned `assembleDebug`; trusted PRs build signed `assembleRelease` (secrets: `KEYSTORE_*`, `GITHUB_TOKEN` for package resolution).
- `:patcher` process behavior is special (see architecture) — app init, Firebase, and most singletons must not assume the main process.
- `.gitattributes` enforces LF for `gradlew` and `app/src/main/assets/root/*.sh`; keep that intact.