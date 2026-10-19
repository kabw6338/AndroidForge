# AndroidForge

> A small "Android build service inside GitHub" — drop a ZIP, run a workflow, get an APK.

AndroidForge is a GitHub-native system that automatically builds an Android APK from an uploaded Android source-code ZIP. It performs real project detection (Gradle, Flutter, multi-module, native C/C++, Kotlin vs Java), chooses a compatible toolchain (JDK / Gradle / AGP / NDK / Flutter versions) on the fly, runs the build, picks the relevant APK, and exposes it as a downloadable GitHub Actions artifact.

## 🌐 Use the web UI (recommended)

A drag-and-drop web interface is hosted on GitHub Pages:

➡️ **https://professional-x.github.io/AndroidForge/**

You will need a GitHub Personal Access Token (classic PAT with `repo` + `workflow` scopes, or a fine-grained PAT with `Contents: Read/Write`, `Actions: Read/Write`, `Releases: Read/Write`). The token is stored only in your browser's `localStorage` and sent directly to `api.github.com` — it never touches any other server.

### Web UI flow

1. Open the URL above.
2. Enter your fork's owner / repo name / PAT.
3. Click **Save connection**.
4. Drag your `.zip` into the upload card (or click to pick).
5. Choose a build variant (auto / debug / release / bundle).
6. Click **⚒️ Build APK**.
7. Watch live status, step progress, and elapsed time.
8. When the build finishes, click the run link to download your APK from the GitHub Actions UI.

### How the web UI works under the hood

- Creates a GitHub Release on tag `web-build-<timestamp>` on your fork.
- Uploads your `.zip` as a release asset via the GitHub REST API.
- Triggers `workflow_dispatch` on `.github/workflows/build-android.yml` with `zip_source=release-asset` and `release_tag=<tag>`.
- Polls the run status every 8 s and renders a step-by-step progress UI.
- When the run completes, lists the produced artifacts with download links.

---

## 🐳 Prebuilt Docker image (build optimization)

To eliminate repeated installation of JDK / Android SDK / Gradle / Flutter on every APK build, this repo ships a **prebuilt Docker image** that contains the entire Android build environment.

### Image contents
| Layer | Version(s) |
|---|---|
| Base | `ubuntu:22.04` |
| JDK | 8, 11, 17 (all three preinstalled; `JAVA_HOME` defaults to 17) |
| Android SDK | Command-line tools + platform-tools |
| Build-tools | 28.0.3, 29.0.3, 30.0.3, 32.0.0, 33.0.2, 34.0.0 |
| Android platforms | android-23 → android-34 |
| CMake | 3.18.1, 3.22.1 (for native C/C++ code) |
| Gradle | 8.9 system-wide (used only when a project has no usable wrapper) |
| Flutter | 3.27.0 stable (with desktop platforms enabled) |
| Linux desktop libs | libgtk-3-dev, libmpv-dev, libayatana-appindicator3-dev, libsecret-1-dev |
| Utilities | git, curl, wget, unzip, zip, python3, PyYAML |

### Image location & lifecycle

- **Image**: `ghcr.io/<owner>/androidforge-build-env:latest` (also tagged with short SHA + date)
- **Build workflow**: `.github/workflows/build-android-image.yml` — builds and pushes the image to GHCR only when `Dockerfile.android` (or the workflow itself) changes. It does NOT run on every APK build.
- **Dockerfile**: `Dockerfile.android` at the repo root.
- **Consumer**: `.github/workflows/build-android.yml` declares `container: ghcr.io/<owner>/androidforge-build-env:latest` on its build job, so the workflow starts inside the image instead of installing tools from scratch.

### One-time initial setup

After forking the repo, you must publish the image once before the APK workflow can use it:

1. Go to the repo's **Settings → Actions → General → Workflow permissions** and enable **"Read and write permissions"** (this lets the GITHUB_TOKEN push packages to GHCR).
2. Go to **Settings → Packages** and make sure new packages are visible to your account (default is "private" — the APK workflow needs `packages: read` to pull the image, which works even if private).
3. Trigger the **"Build & Publish Android Image"** workflow from the Actions tab (or push a change to `Dockerfile.android`). First build takes ~15 min and produces a ~3 GB image.
4. After the image is published, the APK workflow will pull it automatically on its next run.

### Maintenance: how to update the image

When you want to bump a tool version (e.g. Flutter 3.27 → 3.30, add a new Android platform, switch Gradle 8.9 → 8.10):

1. Edit `Dockerfile.android` — bump the relevant version.
2. Commit and push to `main`. The `build-android-image.yml` workflow auto-triggers (because its `paths:` filter includes `Dockerfile.android`).
3. Wait ~10–15 min for the new image to publish. The `:latest` tag is moved automatically; old SHA-tagged images are kept for rollbacks.
4. The next APK build picks up the new image automatically.

### How the APK workflow uses the image

The `build-android.yml` workflow now does this on every run:

1. Starts the job **inside** the prebuilt Docker container (no JDK/SDK/Gradle install).
2. Checks out the repo, installs Python deps (PyYAML, ~1 MB).
3. Acquires the source ZIP via the existing input/URL/release mechanism.
4. Runs `detect_project.py` to identify Gradle vs Flutter vs multi-module, etc.
5. Runs `setup_environment.py` to pick a compatible JDK / Gradle / Flutter version.
6. **Switches `JAVA_HOME`** to the detected JDK (all three are preinstalled — no `actions/setup-java` needed).
7. **Installs only missing SDK components** via `sdkmanager` (if the project's `compileSdk` isn't already in the image, e.g. a project that needs android-22).
8. Restores the **per-project Gradle cache** (`actions/cache@v4` keyed on gradle version + hash of build.gradle/gradle-wrapper.properties/libs.versions.toml).
9. Restores the **per-project Flutter pub cache** for Flutter projects.
10. Runs `build_project.py` → `assembleDebug` / `assembleRelease` / `bundleDebug`.
11. Runs `collect_apk.py` → picks the primary APK.
12. Uploads the APK as a workflow artifact + publishes it to a release tagged `web-build-output-<run_id>`.
13. On failure, runs `diagnose_failure.py` to surface the most likely cause.

**Time saved:** the old workflow spent ~3–5 min per build installing JDK 17 (~250 MB), Android SDK platforms + build-tools (~1 GB), Gradle (~150 MB), and Flutter (~1.5 GB). The new workflow spends ~30 s pulling the cached Docker layer + zero time installing these tools.

---

## CLI / direct workflow use

If you prefer not to use the web UI, you can also trigger the build directly from GitHub:

```
Android Project ZIP
        │
        ▼
   Project Detection
        │
        ▼
   Toolchain Detection
        │
        ▼
   Environment Setup
        │
        ▼
    Android Build
        │
        ▼
    APK Detection
        │
        ▼
   GitHub Artifact
        │
        ▼
    Download APK
```

You do **not** need to manually configure Android Studio, Gradle, Java, the Android SDK, the NDK, or Flutter. AndroidForge inspects the project and picks everything automatically.

---

## Table of contents

1. [What this project does](#1-what-this-project-does)
2. [How to upload / provide a ZIP](#2-how-to-upload--provide-a-zip)
3. [How to start the workflow](#3-how-to-start-the-workflow)
4. [How project detection works](#4-how-project-detection-works)
5. [How the APK is obtained](#5-how-the-apk-is-obtained)
6. [Supported project types](#6-supported-project-types)
7. [Legacy-project strategy](#7-legacy-project-strategy)
8. [Limitations](#8-limitations)
9. [Common build failures](#9-common-build-failures)
10. [How to extend toolchain compatibility](#10-how-to-extend-toolchain-compatibility)
11. [GitHub Actions usage requirements](#11-github-actions-usage-requirements)
12. [Repository structure](#12-repository-structure)
13. [Security](#13-security)

---

## 1. What this project does

AndroidForge turns a GitHub Actions runner into an on-demand Android build service. Instead of forcing every contributor to install a fully-configured Android toolchain on their laptop, the build is delegated to a runner that:

- **Auto-detects** the kind of Android project you uploaded (Gradle vs Flutter, Java vs Kotlin, multi-module vs single-module, native C/C++ or not).
- **Picks a compatible toolchain** by consulting `config/toolchain-rules.yaml` — a single, modular file that maps every known Android Gradle Plugin (AGP) version to the Gradle version and JDK version it requires.
- **Installs the right toolchain versions** on the runner (JDK, Gradle, Android SDK, NDK, Flutter) with caching so repeated builds become faster.
- **Builds the project** using the project's own `gradlew` whenever possible, falling back to a system-installed Gradle when the wrapper is too old, missing, or corrupt.
- **Collects the resulting APK(s)** from the build output tree, picks the most relevant one (release > debug > bundle), renames it clearly, and uploads it as a GitHub Actions artifact you can download.
- **Diagnoses failures** — instead of just printing "BUILD FAILED", it scans the log for known signatures and reports the most likely cause (Java incompatibility, missing SDK, dependency resolution, manifest errors, etc.) directly in the workflow Summary.

The end-user experience is as close as GitHub allows to: upload ZIP → run workflow → wait → download APK.

---

## 2. How to upload / provide a ZIP

AndroidForge supports three ZIP sources. Pick whichever fits your workflow best.

### Option A — `input/` directory (simplest, recommended for first use)

1. Clone this repository locally.
2. Drop your Android source ZIP into `input/` (e.g. `input/myproject.zip`).
3. Commit and push (or push to a feature branch and open a PR — the workflow will trigger automatically when the ZIP lands on `main` because of the `push.paths: input/**.zip` trigger in `.github/workflows/build-android.yml`).

> ℹ️ Committing ZIPs into git is not ideal for huge binaries but is fine for source archives. For very large ZIPs use Option B or C.

### Option B — GitHub Release asset (recommended for large ZIPs)

1. Go to the repository → **Releases** → **Draft a new release**.
2. Pick a tag (e.g. `build-input/v1`).
3. Attach your ZIP as a release asset.
4. Publish the release.
5. Run the workflow manually (see [section 3](#3-how-to-start-the-workflow)) and select `release-asset` as the source, providing the tag you just created.

AndroidForge uses the GitHub REST API + the runner's built-in `GITHUB_TOKEN` to download the asset — **no PAT required**, no extra auth setup.

### Option C — Public download URL

1. Host the ZIP anywhere publicly downloadable (your own server, another GitHub release, a Google Drive direct link, etc.).
2. Run the workflow manually and select `download-url` as the source, providing the URL.

---

## 3. How to start the workflow

1. Open the repository on GitHub.
2. Go to the **Actions** tab.
3. In the left sidebar, click **Build Android APK**.
4. Click **Run workflow** (top-right).
5. Configure the inputs:

   | Input | Purpose |
   |---|---|
   | `zip_source` | Where to get the ZIP (`input-directory` / `download-url` / `release-asset`) |
   | `zip_url` | Required only if `download-url` is selected |
   | `release_tag` | Required only if `release-asset` is selected |
   | `build_variant` | `auto` (try debug, then release, then bundle) or pick a specific variant |
   | `enable_ndk` | Force-install the NDK even if no native code is detected |

6. Click **Run workflow**. The build typically takes 5–15 minutes (much faster on repeat runs thanks to caching).

> 💡 There is also a **Diagnostics** workflow that runs only the detection + toolchain-config steps (no actual build). Useful for troubleshooting what AndroidForge thinks of a ZIP before committing to a full build.

---

## 4. How project detection works

`scripts/detect_project.py` is the entry point. It:

1. **Finds the real project root** inside the extracted ZIP — ZIPs often nest everything under `MyProject-1.2.3/`, so the script walks up to 3 levels deep looking for `settings.gradle`, `pubspec.yaml`, or `build.gradle`.
2. **Detects indicator files** at the project root and one level deep: `settings.gradle(.kts)`, `build.gradle(.kts)`, `gradlew`, `gradle-wrapper.properties`, `AndroidManifest.xml`, `pubspec.yaml`, `local.properties`, `gradle.properties`, `CMakeLists.txt`, `Android.mk`, `gradle/libs.versions.toml`.
3. **Classifies the project**:
   - `flutter` — `pubspec.yaml` present at root.
   - `gradle` — `settings.gradle(.kts)` or `build.gradle(.kts)` present, then sub-typed into `kotlin` if the kotlin-android plugin or any `.kt` file is found, otherwise `java`.
4. **Parses the Gradle wrapper** — reads `gradle/wrapper/gradle-wrapper.properties` to extract the distribution URL and Gradle version, and flags whether it uses `http://` (legacy, will be auto-migrated to `https://`).
5. **Discovers modules** — both from `include ':name'` lines in `settings.gradle` and by scanning for sub-directories containing `build.gradle(.kts)`.
6. **Detects native code** — looks for `CMakeLists.txt` / `Android.mk` and counts `.cpp` / `.c` / `.cc` / `.cxx` files anywhere in the tree.
7. **Extracts versions** from all `build.gradle(.kts)` files and `gradle/libs.versions.toml`:
   - AGP version (multiple patterns supported: `classpath 'com.android.tools.build:gradle:X.Y.Z'`, version catalog `agp = "..."`, plugins block `version "..."`, etc.)
   - Kotlin version
   - NDK version (`ndkVersion = "..."`)
   - Java version (`sourceCompatibility = JavaVersion.VERSION_11`)
   - `compileSdk` / `targetSdk` / `minSdk`
8. **Assesses legacy status** — flags the project as legacy if any of: Gradle wrapper version < 7, AGP major < 7, Java ≤ 11, `compileSdk` < 30, or no wrapper at all.
9. **Emits a JSON summary** to stdout and to `$GITHUB_OUTPUT` for downstream workflow steps.

The detection step never modifies the project tree — it only reads.

---

## 5. How the APK is obtained

After the build step, `scripts/collect_apk.py`:

1. Recursively searches the project tree for `*.apk` and `*.aab` files.
2. **Skips intermediate outputs** — anything inside `build/intermediates/` is ignored. Only outputs in `build/outputs/apk/` or `build/outputs/bundle/` are kept.
3. **Picks a primary APK** using this priority: `*-release.apk` > `*-debug.apk` > other `.apk` > `.aab`.
4. **Renames each APK** to the form `AndroidForge-<module>-<variant>.apk` (e.g. `AndroidForge-app-release.apk`) and copies it into the `output/` directory.
5. **Writes the output paths** to `$GITHUB_OUTPUT` so the workflow can upload them.
6. The workflow then uses `actions/upload-artifact@v4` to publish the APK as a downloadable artifact named `AndroidForge-APKs`.

To download the APK:

1. Open the completed workflow run.
2. Scroll to the bottom of the run summary.
3. Click the **AndroidForge-APKs** artifact to download a ZIP containing your APK(s).
4. If the build failed, also grab the **AndroidForge-logs** artifact for diagnosis.

---

## 6. Supported project types

| Type | Detected via | Build command |
|---|---|---|
| Gradle (Java) | `settings.gradle` + no Kotlin plugin + `build.gradle` present | `./gradlew assembleDebug` / `assembleRelease` |
| Gradle (Kotlin) | `settings.gradle` + Kotlin plugin or `.kt` files | `./gradlew assembleDebug` / `assembleRelease` |
| Multi-module Gradle | `include ':...'` lines in `settings.gradle` | Same as above — Gradle handles module resolution |
| Old Gradle wrapper | `gradle-wrapper.properties` with version < 7 | Wrapper used; legacy fixes applied |
| Missing/corrupt wrapper | `gradle-wrapper.jar` absent | `gradle wrapper --gradle-version <X>` regenerates it |
| Native C/C++ (CMake) | `CMakeLists.txt` present | Same Gradle build — NDK + CMake installed by workflow |
| Native C/C++ (ndk-build) | `Android.mk` present | Same Gradle build — NDK installed |
| Flutter | `pubspec.yaml` at root | `flutter pub get` → `flutter build apk --debug` / `--release` |
| Version-catalog projects | `gradle/libs.versions.toml` present | Versions read from the catalog, fallback to inline declarations |
| Legacy AGP (3.x / 4.x) | AGP version string in `build.gradle` | Compatible Gradle + JDK chosen from `toolchain-rules.yaml` |

If your project type isn't on this list, AndroidForge will still try to build it — it falls back to a generic `./gradlew assembleDebug` → `assembleRelease` → `bundleDebug` sequence. Detection only gets *better* as the rules in `toolchain-rules.yaml` are extended.

---

## 7. Legacy-project strategy

Old Android projects are a first-class concern, not an afterthought. The strategy is:

1. **Prefer the project's own Gradle wrapper** — it pins the exact Gradle version the project was authored against. If the wrapper works on the runner, use it.
2. **Auto-fix common wrapper issues**:
   - Wrapper distribution URL using `http://` → migrated to `https://` in-place (in the workspace, never in the source ZIP).
   - `gradle-wrapper.jar` missing or corrupt → regenerated by running `gradle wrapper --gradle-version <X>` using a system-installed Gradle.
3. **Fall back to a system-installed Gradle** of a compatible version when the wrapper can't be used. The version is picked from the `agp_to_gradle` mapping in `toolchain-rules.yaml`.
4. **Pick a compatible JDK** using both the AGP version → JDK and Gradle version → JDK mappings. For example, AGP 3.x → JDK 8, AGP 4.x/7.x → JDK 11, AGP 8.x → JDK 17.
5. **Install old Android SDK platforms** — `platforms;android-23` through `platforms;android-34` are pre-installed by the workflow so old `compileSdk` values resolve cleanly.
6. **Disable the Gradle daemon** for very old Gradle versions (< 4) where daemon mode is unstable.
7. **Auto-inject `local.properties`** with `sdk.dir` if the project doesn't have one.
8. **Never modify the source ZIP destructively** — every fix is applied in the build workspace, which is discarded after the workflow completes.

---

## 8. Limitations

- **No signing by default.** Release builds are unsigned (or use the project's own debug/release config if present). To produce a signed release APK you must add a keystore as a GitHub Actions secret and reference it in your project's `build.gradle` `signingConfigs`. AndroidForge deliberately does not auto-sign because doing so safely requires project-specific keystore configuration.
- **No emulator / instrumentation tests.** AndroidForge only produces an APK. It does not run instrumentation tests on an emulator. Use a separate workflow for that.
- **NDK version must be installable from the SDK Manager.** The NDK version declared in `build.gradle` must be one that `sdkmanager` knows about. If you use an obscure NDK version, update `config/toolchain-rules.yaml → ndk.legacy_versions`.
- **Network access required.** Dependency resolution requires internet access to `maven.google.com`, `repo1.maven.org`, `plugins.gradle.org`. If your project depends on a private repository, the build will fail unless you make that repo accessible from a public GitHub runner.
- **Source ZIP is untrusted.** AndroidForge treats the uploaded ZIP as potentially malicious: no repository secrets are passed into the build environment, and the build runs on a standard ephemeral GitHub runner. Don't upload a project that executes arbitrary scripts at configuration time unless you've audited it first.
- **Build duration.** First builds take ~10–15 minutes (Gradle distribution + dependency download). Cached builds usually finish in 3–6 minutes.
- **Workflow timeout.** Hard limit of 60 minutes per build. If your project exceeds this, split modules or move to a self-hosted runner.

---

## 9. Common build failures

The diagnosis step (`scripts/diagnose_failure.py`) automatically scans the build log and ranks the most likely cause. Here are the most common ones and what to do:

| Diagnosis | Likely cause | Fix |
|---|---|---|
| **Java incompatibility** | JDK on the runner is older than the bytecode the project requires | Bump `agp_to_jdk` / `gradle_to_jdk` in `config/toolchain-rules.yaml` |
| **Gradle incompatibility** | AGP requires a newer Gradle than the one being used | Check `agp_to_gradle` mapping; the wrapper version may need regenerating |
| **Missing Android SDK platform** | `compileSdk` value not installed | Add the missing `platforms;android-N` package in the workflow's `setup-android` step |
| **Missing NDK** | `ndkVersion` declared in `build.gradle` not installed | Update `config/toolchain-rules.yaml → ndk.legacy_versions` |
| **Dependency resolution failure** | Repo URL missing or network issue | Add `google()` / `mavenCentral()` to `settings.gradle.kts`; check `dependency_fallbacks` |
| **Kotlin compilation failure** | Kotlin/Java version mismatch | Update `kotlin_to_jdk` mapping or bump Kotlin version |
| **Manifest merger failure** | Duplicate attributes across libraries | Inspect `app/build/outputs/logs/manifest-merger-*.txt` (uploaded as part of the logs artifact) |
| **Corrupt or missing Gradle wrapper** | `gradle-wrapper.jar` not in the ZIP | AndroidForge should auto-regenerate; if it fails, check the `regenerate-wrapper.log` artifact |
| **Unsupported project structure** | Module path declared but not present on disk | Verify the project root was detected correctly (run the **Diagnostics** workflow) |
| **Out of memory** | Default heap too small | Add `org.gradle.jvmargs=-Xmx4g` to `gradle.properties` |

When a build fails, always check two things:

1. The **Diagnosis** section in the workflow summary (rendered by `scripts/diagnose_failure.py`).
2. The **AndroidForge-logs** artifact — contains the full build log plus all Gradle reports.

---

## 10. How to extend toolchain compatibility

The entire compatibility matrix lives in **one file**: `config/toolchain-rules.yaml`. To support new AGP / Gradle / Kotlin / JDK versions, edit that file and commit. No script changes required.

### Adding a new AGP version

```yaml
agp_to_gradle:
  "8.8": "8.10"   # add new entry here

agp_to_jdk:
  "8.x": "17"     # already covers 8.8 — no change needed
```

### Adding a new default NDK

```yaml
ndk:
  default_version: "27.0.12077473"   # bump to a newer NDK
  legacy_versions:
    - "26.1.10909125"                # keep the old default as a fallback
    - "25.2.9519653"
```

### Adding a new Flutter version

```yaml
flutter:
  default_version: "3.27.0"
  channel: "stable"
  jdk: "17"
```

### Adding a new diagnosis pattern

If you encounter a build failure that the diagnosis step doesn't recognize, add the failing log signature to `scripts/diagnose_failure.py → DIAGNOSTICS`:

```python
{
    "label": "New error class",
    "patterns": [r"some-regex-from-the-log"],
    "hint": "What the user should do",
},
```

### Adding a new project type

If you need to support a project type AndroidForge doesn't know yet (e.g. React Native, Capacitor, Cordova), you need to:

1. Extend `scripts/detect_project.py` to recognize it (add an indicator file + classification logic).
2. Extend `scripts/setup_environment.py` to pick the right toolchain.
3. Extend `scripts/build_project.py` to run the right build command.

Detection / setup / build are intentionally **separate modules** — change one without touching the others.

---

## 11. GitHub Actions usage requirements

- **Runner**: `ubuntu-latest` (Ubuntu 22.04 / 24.04). Standard GitHub-hosted runner; no self-hosted runner required.
- **Actions used** (all official or widely-trusted community actions):
  - `actions/checkout@v4`
  - `actions/setup-python@v5`
  - `actions/setup-java@v4`
  - `actions/cache@v4`
  - `actions/upload-artifact@v4`
  - `android-actions/setup-android@v3`
  - `subosito/flutter-action@v2`
- **Permissions**: `contents: read`, `actions: write`. No secrets are required for the default flow. The release-asset source uses the runner's auto-provided `GITHUB_TOKEN` (read-only on the same repo).
- **Caching**: JDK (via `setup-java` cache), Gradle caches + wrapper, Python deps, and Flutter (via `flutter-action` cache). Cache keys incorporate the chosen Gradle version + a hash of all Gradle build files.
- **Timeout**: 60 minutes per build (configurable in `.github/workflows/build-android.yml`).
- **Concurrency**: One build per ref at a time — newer pushes do not cancel in-flight builds, so you always get a result for each input.

To run AndroidForge in your own organization, fork or clone this repository, push it to a new GitHub repo, and you're done. No additional setup is required.

---

## 12. Repository structure

```
AndroidForge/
├── .github/
│   └── workflows/
│       ├── build-android.yml   # Main workflow: ZIP → APK
│       └── diagnostics.yml      # Diagnostics-only workflow (detection + setup, no build)
├── scripts/
│   ├── acquire_zip.py           # Source ZIP acquisition (input-dir / URL / release-asset)
│   ├── extract_zip.py            # Extract ZIP into a clean workspace
│   ├── detect_project.py         # Project detection (Gradle / Flutter / multi-module / native)
│   ├── setup_environment.py     # Toolchain choice (JDK / Gradle / AGP / NDK / Flutter)
│   ├── build_project.py          # Build orchestration (runs gradlew / flutter)
│   ├── collect_apk.py            # Locate, rename, copy generated APK(s)
│   ├── diagnose_failure.py       # Scan build log for known error signatures
│   ├── build_summary.py         # Render Markdown build summary
│   └── requirements.txt          # PyYAML
├── config/
│   └── toolchain-rules.yaml     # Single source of truth for version mappings
├── input/
│   └── .gitkeep                  # Drop a ZIP here for the simplest workflow
├── output/
│   └── .gitkeep                  # Generated APKs land here before being uploaded
├── README.md
├── LICENSE                       # MIT
└── .gitignore
```

The system is intentionally modular: detection, environment setup, building, and artifact collection are four separate Python scripts. The workflow YAML orchestrates them. To change one part (e.g. add a new project type), you only touch the relevant script + `toolchain-rules.yaml` — the rest keeps working.

---

## 13. Security

- The uploaded ZIP is treated as **untrusted source code**.
- **No repository secrets** are passed into the build environment. The `permissions:` block in `build-android.yml` only grants `contents: read` and `actions: write` (for uploading artifacts).
- The auto-provided `GITHUB_TOKEN` is used **only** to download a release asset from the same repository via the GitHub REST API — it is never printed or logged, and is not exposed to the build commands themselves.
- Build commands (`./gradlew …`, `flutter …`) run in the standard ephemeral GitHub-hosted runner sandbox.
- If you must include signing keys or other secrets for a release build, declare them as GitHub Actions **repository secrets** and reference them via `${{ secrets.XXX }}` in the workflow — never commit them to the repo.

---

## License

MIT — see [LICENSE](LICENSE).
