# Architecture: app and CI

This repository is a CircleCI demo for native Android. The app is a fork of Google's
[Sunflower](https://github.com/android/sunflower) sample, a gardening app. It exists to show
how an Android project builds, tests and ships on CircleCI, and to reproduce customer issues.

## 1. The app

### Stack (November 2026)

| Area | Version |
|------|---------|
| Android Gradle Plugin | 9.5.0 (built-in Kotlin) |
| Gradle | 9.8.2 (wrapper) |
| Kotlin / KSP | 2.4.21 / 2.3.13 |
| SDK levels | compile 37, target 36, min 24 (Android 7.0) |
| JDK | 17 bytecode; builds on JDK 17 or newer (CI uses 21) |

All versions are kept in the version catalog, `gradle/libs.versions.toml`. The build scripts
are Kotlin DSL: `settings.gradle.kts`, `build.gradle.kts` and `app/build.gradle.kts`.

### Layers

It is a single-module app (`:app`, package `com.google.samples.apps.sunflower`) using MVVM:

```
UI (Fragments + Data Binding layouts)
  -> ViewModels (viewmodels/)          LiveData / Flow, Paging
    -> Repositories (data/*Repository)
      -> Room database (data/*Dao, AppDatabase)   local plants and garden
      -> Unsplash REST API (api/UnsplashService)  plant photos, via Retrofit + OkHttp
```

- **UI.** `GardenActivity` hosts a single Navigation graph (`res/navigation/nav_garden.xml`).
  - `HomeViewPagerFragment` holds two tabs: `GardenFragment` (my garden) and
    `PlantListFragment` (all plants).
  - From either tab you can open `PlantDetailFragment`, and from there `GalleryFragment`
    (Unsplash photos).
  - Layouts use Data Binding, with binding adapters in `adapters/`. Images load through Glide.
- **Dependency injection.** Hilt. `di/DatabaseModule` provides Room and `di/NetworkModule`
  provides the Unsplash service. Code generation runs through KSP.
- **Data.**
  - Room database `sunflower-db` with two entities, `Plant` and `GardenPlanting`.
  - On first launch, `workers/SeedDatabaseWorker` (WorkManager) fills the plant table from
    `assets/plants.json`.
  - The gallery pages Unsplash results with Paging 3 (`UnsplashPagingSource`).
- **Configuration.** The Unsplash key comes from the `unsplash_access_key` Gradle property and
  is exposed as `BuildConfig.UNSPLASH_ACCESS_KEY`. Without a key, the app works but the
  gallery is empty.

### Build-time code generation

| Tool | Used for |
|------|----------|
| KSP | Room DAOs, Hilt components |
| `com.android.legacy-kapt` | Data Binding only (it still requires kapt) |
| Safe Args | Typed navigation arguments |

### Tests

| Kind | Location | Runs on | Gradle task |
|------|----------|---------|-------------|
| Unit tests (JVM) | `app/src/test` (4 classes) | Docker | `testDebugUnitTest` |
| Instrumented / Espresso | `app/src/androidTest` | Emulator | `connectedDebugAndroidTest` |

The instrumented tests use a custom runner, `utilities/MainTestRunner`, which swaps in a
Hilt test application. Coroutine tests use `MainCoroutineRule` with `runTest`.

## 2. CI on CircleCI

Configuration lives in `.circleci/config.yml`, and the Smarter Testing suite in
`.circleci/test-suites.yml`.

### Building blocks

- **Orbs.**
  - `circleci/android@3.2.0`: Gradle cache commands and emulator start/run commands.
  - `circleci/ruby@4.0.0`: `install-deps` (bundle install with cache) for Fastlane.
- **Executors.**
  - `android_docker`: `cimg/android:2026.10.1` with resource class `xlarge`. Used for lint,
    unit tests and the release build. The image ships JDK 21, Android SDK platforms up to
    37, Gradle and Ruby.
  - `android_machine`: the `android:2026.10.1` machine image with resource class `xlarge`.
    A VM with hardware acceleration, so it can run the emulator.
- **Gradle cache.** A local `restore_gradle_cache` command wraps the orb command. The cache
  key is a checksum of the build scripts plus `libs.versions.toml` and
  `gradle-wrapper.properties`, so a dependency bump creates a new cache.
  `gradle.properties` turns on the Gradle build cache, parallel builds and the configuration
  cache.

### Workflow `main`

```
lint ──┬── unit_test ───────────┬── integration_test ── beta (main branch only, context "mobile")
       └── unit_test_parallel ──┘
```

| Job | Executor | What it does |
|-----|----------|--------------|
| `lint` | Docker | `./gradlew lintDebug` and stores `app/build/reports` as artifacts. |
| `unit_test` | Docker | `./gradlew testDebugUnitTest` and uploads JUnit results to Test Insights. |
| `unit_test_parallel` | Docker, parallelism 4 | The same tests through Smarter Testing (`circleci testsuite run "unit tests"`), split across 4 nodes. |
| `integration_test` | Machine | Creates an API 36 x86_64 emulator (`system-images;android-36;google_apis;x86_64`), waits for boot, then runs `connectedDebugAndroidTest` with retries and uploads the results. |
| `beta` | Docker | `bundle exec fastlane beta`: builds the debug APK, uploads it to Firebase App Distribution, and stores the APK as a job artifact. |

`unit_test` and `unit_test_parallel` run the same tests on purpose, to compare a single node
with a split run.

### Smarter Testing (`unit_test_parallel`)

`.circleci/test-suites.yml` defines the suite `unit tests`:

- `discover` lists the test classes as fully qualified names, for example
  `com.google.samples.apps.sunflower.data.PlantTest`. These are the test atoms.
- `run` passes the atoms assigned to the current node to Gradle as `--tests <class>`. It then
  copies Gradle's JUnit XML into `test-results/unit` and keeps Gradle's exit code.
- The atoms match the `classname` attribute in Gradle's JUnit XML, which is how CircleCI gets
  per-class timings.

Included for free: timing-based splitting across nodes, and "Rerun workflow from failed",
which reruns only the failed tests. The job's `store_test_results` path must stay equal to
`outputs.junit-directory`. Test impact analysis, dynamic splitting and auto-rerun are
credit-charged options and are not enabled. `circleci testsuite doctor "unit tests"` checks
the setup; it is available in the CircleCI CLI inside CI jobs.

### Release: Fastlane and Firebase App Distribution

- Ruby gems are in `Gemfile` / `Gemfile.lock`: fastlane 2.241 and
  `fastlane-plugin-firebase_app_distribution` 1.0. Both need Ruby 3.2 or newer. The lockfile
  includes the `x86_64-linux` platform for CI.
- The lane `beta` in `fastlane/Fastfile` runs `gradle(task: "assembleDebug")` and then
  `firebase_app_distribution` for app `1:1009063717109:android:caf084735ba70751883dbc`.
- **Secrets** come from the CircleCI context `mobile`, never from the repository. Set one of:
  - `FIREBASE_SERVICE_ACCOUNT_JSON`: the content of a Google service account key with the
    *Firebase App Distribution Admin* role. This is the recommended option.
  - `FIREBASE_TOKEN`: the legacy token from `firebase login:ci`. It is used only when the
    service account variable is unset, and it expires. An invalid or expired token is the
    usual cause of a failing `beta` job.
- `beta` runs only on `main` (`filters: pipeline.git.branch == "main"`), after every test job
  has passed.

## 3. Getting the app onto a phone

The `beta` job produces a debug APK, signed with the debug key. The phone needs Android 7.0
(API 24) or newer. There are two ways to install it.

### Option A: Firebase App Distribution (for testers)

1. **One-time Firebase setup.**
   - In the Firebase console, open the project for app
     `1:1009063717109:android:caf084735ba70751883dbc`, then go to **App Distribution**.
   - Under **Testers & Groups**, add your email (or create a group such as `internal`).
   - At the moment the Fastfile passes no `testers:` or `groups:`. So after each upload, open
     the release in the console and add testers. To automate this, add
     `testers: "you@example.com"` or `groups: "internal"` to `firebase_app_distribution` in
     the Fastfile.
2. **Trigger a build.** Push to `main`, or rerun the workflow on `main`, and wait for `beta`
   to go green. The job log shows the release link.
3. **On the phone.**
   1. Open the invitation email from Firebase App Distribution and tap **Get started**.
   2. Sign in with the invited Google account and accept the invitation.
   3. When asked, install **Firebase App Tester**, or use the web flow and add its home screen
      shortcut.
   4. In App Tester, open *Sunflower* and tap **Download**, then **Install**.
   5. When Android asks, allow *Install unknown apps* for App Tester or the browser.
      You'll find it under Settings, Apps, Special app access.
4. **New builds.** Each `beta` run creates a new release. Testers get notified, and App Tester
   shows an **Update** button.

### Option B: CircleCI artifact (quick, no Firebase needed)

1. Open the `beta` job in CircleCI, go to the **Artifacts** tab and download
   `app/build/outputs/apk/debug/app-debug.apk`.
2. Install it:
   - **With a cable (adb).** Enable *Developer options* and *USB debugging* on the phone,
     then run:
     ```
     adb install -r app-debug.apk
     ```
   - **Without a cable.** Open the artifact link in the phone's browser (sign in to CircleCI
     first), download the APK, tap it, and allow *Install unknown apps* for the browser.

### Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `beta` fails with an authentication error | The token is missing or expired in the `mobile` context. Add `FIREBASE_SERVICE_ACCOUNT_JSON` or regenerate `FIREBASE_TOKEN`. |
| Upload succeeds but no email arrives | No testers on the release. Add them in the console, or set `testers:` / `groups:` in the Fastfile. |
| `App not installed` / package conflict | A build with the same `applicationId` but a different signing key is already installed. Uninstall it first. |
| `There was a problem parsing the package` | The phone runs Android older than 7.0 (`minSdk` 24). |
| `beta` never runs | The workflow didn't run on `main`, or a test job failed. |
