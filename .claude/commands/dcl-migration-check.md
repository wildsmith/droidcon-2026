---
description: Check a module's readiness for Declarative Gradle (DCL) and Isolated Projects
---

Analyze Gradle build files for Declarative Gradle migration readiness and Isolated Projects compatibility.

**Usage:**
- `claude dcl-migration-check path/to/module` — scan a specific module
- `claude dcl-migration-check` — scan all modules in the current project
- `claude dcl-migration-check --deep` — deep scan: also analyze `.kt` build logic source files
- `claude dcl-migration-check path/to/module --deep` — deep scan a specific module
- `claude dcl-migration-check --json` — output machine-readable JSON (for CI pipelines)
- `claude dcl-migration-check --deep --json` — deep scan with JSON output

If `$ARGUMENTS` contains `--deep`, enable Deep mode. If `$ARGUMENTS` contains `--json`, produce JSON output (see Step 5c). The remaining argument, if any, is the target module path. If no module path is given, scan all modules.

### Scan Modes

**Standard** (default): Scan `build.gradle.kts`, `build.gradle`, and `settings.gradle.kts` files. Exclude `buildSrc/`, `build-logic/`, and `gradle/` directories from the migration report (they are build logic and stay as KTS).

**Deep** (`--deep`): In addition to Standard, also scan `.kt` source files in `buildSrc/src/`, `build-logic/*/src/`, `gradle/*/src/`, and any `*-plugin*/src/` directories. Many projects implement Gradle plugins, convention plugins, and custom tasks in regular `.kt` files. These contain the same anti-patterns (cross-project access, eager APIs, `afterEvaluate`) that block Isolated Projects. Flag them with the same rules, but note they are **build logic files** — they don't migrate to DCL themselves, but must be fixed for Isolated Projects compatibility.

## Available Software Types

Use these as the top-level block when generating `.gradle.dcl` files:

| Applied Plugin | Software Type Block | Ecosystem Plugin |
|---|---|---|
| `com.android.application` | `androidApplication { }` | `org.gradle.experimental.android-ecosystem` |
| `com.android.library` | `androidLibrary { }` | `org.gradle.experimental.android-ecosystem` |
| `java-library` | `javaLibrary { }` | `org.gradle.experimental.jvm-ecosystem` |
| `org.jetbrains.kotlin.jvm` | `kotlinJvmLibrary { }` | `org.gradle.experimental.jvm-ecosystem` |
| `java-gradle-plugin` | **Do NOT migrate** — stays as `.gradle.kts` | N/A |
| `org.jetbrains.kotlin.multiplatform` | `kotlinMultiplatform { }` (prototype) | unified-prototype only |

### DSL Schema by Software Type

**`androidApplication`** — available properties and nested blocks:

| Property / Block | Type | Required | Notes |
|---|---|---|---|
| `namespace` | `String` | ✅ | Android namespace |
| `applicationId` | `String` | ✅ | Application ID |
| `versionCode` | `Int` | | App version code |
| `versionName` | `String` | | App version name |
| `buildTypes { release { minify { enabled } } }` | Block | | ProGuard/R8 minification |
| `buildTypes { debug { applicationIdSuffix } }` | Block | | Debug variant suffix |
| `dependencies { }` | Block | | `api()`, `implementation()`, `compileOnly()`, `runtimeOnly()` |
| `testing { dependencies { } }` | Block | | Test deps (replaces `testImplementation`, `androidTestImplementation`) |

What moves to settings `defaults {}`: `compileSdk`, `minSdk`, `targetSdk`, `jdkVersion`
What the Software Type handles: plugin application (AGP, KGP), `compileOptions`, `kotlinOptions`

**`androidLibrary`** — same as `androidApplication` except no `applicationId`, `versionCode`, `versionName`, or `applicationIdSuffix`.

**`javaLibrary`** / **`kotlinJvmLibrary`** — `dependencies { }` and `testing { dependencies { } }` blocks only.

### Project Features

| Feature | DSL | Notes |
|---|---|---|
| Kotlin Serialization | `kotlinSerialization { version = libs.versions.x; jsonEnabled = true }` | Adds kotlinx-serialization plugin + runtime |
| Testing | `testing { dependencies { implementation(libs.junit) } }` | Test dependencies and configuration |
| Hilt | `hilt { }` | Hilt/Dagger injection (when available) |
| Compose | `compose { }` | Jetpack Compose (when available as a feature) |

### Settings File (`settings.gradle.dcl`)

Generated settings file should include these sections:

```kotlin
pluginManagement {
    repositories {
        gradlePluginPortal()
        google()
        mavenCentral()
    }
}

plugins {
    id("org.gradle.experimental.android-ecosystem")
}

dependencyResolutionManagement {
    repositoriesMode = FAIL_ON_PROJECT_REPOS
    repositories {
        google()
        mavenCentral()
    }
}

defaults {
    androidLibrary {
        jdkVersion = 17
        compileSdk = 35
        minSdk = 26
    }
    androidApplication {
        jdkVersion = 17
        compileSdk = 35
        minSdk = 26
        targetSdk = 35
    }
}
```

## Pre-Scan Checks

### Gradle Version & DCL Availability

Check `gradle/wrapper/gradle-wrapper.properties`:
- **Gradle ≥ 8.13 nightly:** DCL ecosystem plugins available. Proceed with full migration.
- **Gradle 8.x stable:** Scan is still useful for **Isolated Projects readiness**, but warn that generated `.gradle.dcl` files are for preview only until upgrade.
- **Gradle < 8:** Recommend upgrading first.

Include in report header if needed:
> ⚠️ **Gradle X.Y detected.** DCL ecosystem plugins require Gradle ≥ 8.13 nightly. Generated `.gradle.dcl` files are for preview/planning only.

### Groovy Build Files

If `.gradle` (Groovy) files are found instead of `.gradle.kts`:
- Flag as a two-step migration: Groovy → KTS first, then KTS → DCL.
- Still scan for blocker patterns. Recommend [Groovy to KTS migration](https://docs.gradle.org/current/userguide/migrating_from_groovy_to_kotlin_dsl.html) as prerequisite.

### Composite Builds (`includeBuild()`)

If `settings.gradle.kts` contains `includeBuild()`:
- **Included builds stay as KTS** — they are build logic and do not migrate to DCL.
- In Deep mode, scan included build source files for Isolated Projects violations.
- Composite builds for plugin development is a *good sign* — build logic is already separated from build scripts.

## Analysis Steps

### 1. Read the file(s) and classify the module

**Version catalog detection:** Check for `gradle/libs.versions.toml` (or any `.toml` file referenced in `settings.gradle.kts` via `versionCatalogs {}`). If found, parse it and build a lookup map:
- **Libraries:** `[libraries]` entries → maps GAV coordinates to catalog accessors (e.g., `com.squareup.okhttp3:okhttp` → `libs.okhttp`)
- **Plugins:** `[plugins]` entries → maps plugin IDs to catalog accessors
- **Versions:** `[versions]` entries → maps version keys to catalog version accessors

This map is used in Step 4 to preserve or introduce version catalog references in generated DCL.

**Type-safe project dependencies (`projects` catalog):** Check the Gradle wrapper version (`gradle/wrapper/gradle-wrapper.properties`) and `settings.gradle.kts`:
- **Gradle ≥ 9:** `projects` catalog is available by default. Use `projects.core.network` instead of `project(":core:network")`.
- **Gradle < 9:** Look for `enableFeaturePreview("TYPESAFE_PROJECT_ACCESSORS")` in `settings.gradle.kts`. If present, use `projects`. Otherwise, keep `project(":...")` syntax.

The `projects` accessor converts path segments to camelCase:
- `:core:network` → `projects.core.network`
- `:core:legacy-network:ktor` → `projects.core.legacyNetwork.ktor`
- `:feature:sign-in` → `projects.feature.signIn`
- `:ui:card-details-ui` → `projects.ui.cardDetailsUi`

Determine the DCL Software Type from applied plugins (see table above). Gradle plugin projects (`java-gradle-plugin`) should NOT be migrated — they are build logic.

### 2. Scan for DCL blockers

Check for these patterns and classify each finding.

**Violation source attribution:** For every finding, tag with `source: "project"` or `source: "third-party:<plugin-id>"`.
- **Project code** — patterns in the project's build files or build logic (fixable by the team)
- **Third-party plugin** — patterns from external plugins (e.g., `com.google.dagger.hilt.android` calling `afterEvaluate {}` in its `apply()`). Identified by the violation living in a classpath JAR or generated by a plugin's apply method.
- When ambiguous (convention plugin wrapping a third-party plugin), attribute to `"project"` since the team controls the wrapper.

**❌ Hard Blockers** (prevent DCL migration):
- Conditional logic: `if`, `when`, `for`, `while`
- Function calls: `file()`, `rootProject.file()`, `System.getenv()`, `exec {}`
- `afterEvaluate {}` / `beforeEvaluate {}`
- Cross-project access: `project(":other").`, `rootProject.`, `parent.`
- `allprojects {}` / `subprojects {}`
- Custom task registration: `tasks.register {}`, `tasks.create {}`, `task(`
- `extra[` / `extra.set(` / `ext[` / `ext.`
- `buildscript {}`
- Local variables (`val`/`var`) at top level or inside android {}
- `apply(plugin = ...)` / `apply(from = ...)`
- String interpolation in property assignments: `"${...}"`

**⚠️ Warnings** (should move to settings defaults):
- `compileSdk`, `minSdk`, `targetSdk` assignments
- `jvmTarget`, `sourceCompatibility`, `targetCompatibility`
- `composeOptions { kotlinCompilerExtensionVersion = ... }`
- `kotlinOptions { freeCompilerArgs += ... }`

**🔶 Isolated Projects Violations** (block parallel configuration):
- `allprojects {}` / `subprojects {}`
- `project(":path")` config access
- `rootProject.` / `childProjects` / `parent.` property access
- Eager task APIs: `tasks.create(`, `tasks.getByName(`, `tasks.getByPath(`, `tasks.findByPath(`, `tasks.all {`, `tasks.matching {`, `tasks.withType<T> {` without `.configureEach`, `whenTaskAdded {`
- `dependsOn()` with task references
- `project` inside `@TaskAction` / `doFirst {}` / `doLast {}`
- `extensions["name"] as Type` (use `extensions.getByType<Type>()`)
- `@PathSensitive(ABSOLUTE)` (use `NONE` or `RELATIVE`)
- `findByName()` on containers (use `named()`)
- `configurations.getByName()` (use `configurations.named()`)
- `tasks.replace()` (use `tasks.register()`)

### 3. Assess readiness

Classify each module into one of these categories:

| Category | Criteria | Action |
|---|---|---|
| ✅ **Ready** | Zero hard blockers. May have warnings. | Generate DCL, migrate now. |
| ⚠️ **Nearly Ready** | Zero hard blockers but has mechanically fixable Isolated Projects violations. | Auto-fix violations, then generate DCL. |
| 🔶 **Has Blockers** | 1–3 hard blockers with clear fixes. | List fixes with priority. Offer to fix mechanical ones. |
| ❌ **Not Ready** | 4+ hard blockers or architectural redesign needed. | Migration roadmap only. |

Categories communicate actionability better than arbitrary scores.

### 4. Generate equivalent `.gradle.dcl` (if ✅ Ready or ⚠️ Nearly Ready)

Only generate DCL for `build.gradle.kts` / `build.gradle` files (not `.kt` build logic files — those stay as Kotlin).

**Version catalog handling:** Preserve and improve the dependency format:
- `libs.x` catalog accessors → keep as-is (fully supported in DCL)
- Raw GAV strings → replace with `libs.x` if a catalog match exists, else keep raw
- `platform()` / BOM declarations → preserve

**Project dependency handling:**
- If `projects` catalog is available (Gradle ≥ 9, or `TYPESAFE_PROJECT_ACCESSORS` enabled) → convert `project(":...")` to `projects.x.y` with camelCase segments (e.g., `project(":core:legacy-network")` → `projects.core.legacyNetwork`)
- Otherwise → keep legacy `project(":...")` syntax

**Dependency ordering:** Sort dependencies alphabetically within each configuration group:
1. Group by configuration: `api` → `compileOnly` → `implementation` → `runtimeOnly`
2. Within each group: `platform()`/BOM first, then `libs.x` alphabetically, then `projects.x` alphabetically
3. Apply same ordering inside `testing { dependencies { } }`

| KTS Pattern | DCL Equivalent |
|---|---|
| `plugins { id("com.android.library") }` | Top-level `androidLibrary { }` |
| `plugins { id("com.android.application") }` | Top-level `androidApplication { }` |
| `plugins { id("java-library") }` | Top-level `javaLibrary { }` |
| `plugins { id("org.jetbrains.kotlin.jvm") }` | Top-level `kotlinJvmLibrary { }` |
| `android { namespace = "..." }` | `namespace = "..."` (direct child of Software Type) |
| `android { defaultConfig { applicationId = "..." } }` | `applicationId = "..."` (direct child) |
| `android { defaultConfig { versionCode = N } }` | `versionCode = N` (direct child) |
| `android { defaultConfig { versionName = "..." } }` | `versionName = "..."` (direct child) |
| `android { buildTypes { release { isMinifyEnabled = true } } }` | `buildTypes { release { minify { enabled = true } } }` |
| `dependencies { implementation(libs.x) }` | `dependencies { implementation(libs.x) }` (preserve, sort alphabetically) |
| `dependencies { implementation("group:artifact:version") }` | `dependencies { implementation(libs.x) }` if catalog match exists, else keep raw |
| `dependencies { api(libs.x) }` | `dependencies { api(libs.x) }` |
| `dependencies { implementation(project(":path")) }` | `dependencies { implementation(projects.path) }` if `projects` catalog available |
| `dependencies { testImplementation(libs.x) }` | `testing { dependencies { implementation(libs.x) } }` |
| `dependencies { androidTestImplementation(libs.x) }` | `testing { dependencies { implementation(libs.x) } }` |
| `plugins { id("kotlinx-serialization") }` | `kotlinSerialization { version = libs.versions.x }` (use catalog version if available) |
| `compileSdk`, `minSdk`, `targetSdk` | Remove — move to settings `defaults {}` |
| `compileOptions`, `kotlinOptions`, `jvmTarget` | Remove — derived from `jdkVersion` in settings defaults |
| `composeOptions { kotlinCompilerExtensionVersion = ... }` | Remove — derived automatically |
| `plugins {}` block | Remove entirely — Software Type handles it |

### 5. Output structured report

```markdown
## DCL Migration Report: :<module-path>

📊 **Readiness: [✅ Ready | ⚠️ Nearly Ready | 🔶 Has Blockers | ❌ Not Ready]**

**Software Type:** `androidLibrary` | `androidApplication` | `javaLibrary` | etc.

### ✅ Compatible
- Plugin: com.android.library
- Dependencies: N (all using version catalog)
- Namespace: com.example.mylib
- Version catalog: gradle/libs.versions.toml detected
- Projects catalog: available (Gradle 9+)

### ❌ Hard Blockers (N found)
1. **Line XX**: `pattern` — Description
   → Recommended fix

### ⚠️ Warnings (N found)
1. **Line XX**: `pattern` — Recommendation

### 🔶 Isolated Projects Violations (N found)
1. **Line XX**: `pattern` — Description
   → Recommended fix

### Generated DCL (if Ready or Nearly Ready)
\```kotlin
androidLibrary {
    namespace = "com.example.mylib"
    dependencies {
        implementation(libs.okhttp)
        implementation(projects.core.common)
    }
    testing {
        dependencies {
            implementation(libs.junit)
        }
    }
}
\```

### Settings Defaults Recommendations
- `compileSdk = 35` → move to settings defaults
- `minSdk = 26` → move to settings defaults
- `jdkVersion = 17` → move to settings defaults
```

### 5b. Project-Wide Summary (when scanning multiple modules)

```markdown
## Project-Wide DCL Migration Summary

| Metric | Count |
|---|---|
| Total modules scanned | N |
| ✅ Ready | N |
| ⚠️ Nearly Ready | N |
| 🔶 Has Blockers | N |
| ❌ Not Ready | N |

### Top Blockers Across All Modules
1. `afterEvaluate {}` — found in N modules (project: N, third-party: N)
2. `allprojects {}` — found in N modules (project: N, third-party: N)
3. Custom task registration — found in N modules (project: N, third-party: N)

### Third-Party Plugin Blockers
| Plugin ID | Violation | Affected Modules |
|---|---|---|
| `com.google.dagger.hilt.android` | `afterEvaluate {}` in plugin apply | N |
| `com.google.firebase.crashlytics` | Config-time file manipulation | N |

### Recommended Migration Order
1. Start with ✅ Ready leaf library modules
2. Address settings-level issues (allprojects → dependencyResolutionManagement)
3. Fix ⚠️ Nearly Ready modules (mechanical Isolated Projects fixes)
4. Tackle 🔶 Has Blockers modules one blocker at a time
5. App modules last (most complex, likely ❌ Not Ready)
```

### 6. Offer to perform the migration

After presenting the report, ask: **"Would you like me to migrate the ready modules now?"**

If yes, for each module classified as ✅ Ready or ⚠️ Nearly Ready:

1. **Create** `build.gradle.dcl` alongside the existing build file (do NOT delete the original)
2. **Generate** `settings.gradle.dcl` if it doesn't exist (with `pluginManagement`, `plugins`, `dependencyResolutionManagement`, and `defaults` blocks)
3. **Fix mechanical Isolated Projects violations** in `.kt`/`.kts` build logic files:
   - `tasks.create("name")` → `tasks.register("name")`
   - `tasks.getByName("name")` → `tasks.named("name")`
   - `configurations.getByName("name")` → `configurations.named("name")`
   - `tasks.all { }` → `tasks.configureEach { }`
   - `tasks.withType<T> { }` → `tasks.withType<T>().configureEach { }`
   - `extensions["name"] as Type` → `extensions.getByType<Type>()`
   - `findByName("name")` → `named("name")` or `names.contains("name")`
4. **Do NOT auto-fix** patterns requiring architectural decisions (allprojects, afterEvaluate, cross-project access, custom tasks, conditional logic)
5. **Output summary** of files created, files modified, and remaining manual steps

**Important:** Always create new files rather than overwriting. Let the user diff, validate, and remove originals themselves.

### 7. Validate the Migration

After creating files, suggest validation commands:

```bash
# Verify generated DCL files parse correctly (requires DCL-compatible Gradle)
./gradlew --dry-run :module:help

# Check for Isolated Projects violations across the whole build
./gradlew help --isolated-projects

# Run a full build to verify nothing broke
./gradlew :module:assemble
```

If the project is not on a DCL-compatible Gradle version, note DCL files are for preview and suggest running only the `--isolated-projects` check.

**Important:** Present these commands to the user with context on what each verifies. Do not run them automatically.

### 5c. JSON Output (when `--json` is requested)

Write a `dcl-migration-report.json` file instead of printing Markdown:

```json
{
  "gradleVersion": "8.14-nightly",
  "dclAvailable": true,
  "versionCatalog": "gradle/libs.versions.toml",
  "projectsCatalogAvailable": true,
  "groovyFilesDetected": false,
  "compositeBuilds": ["build-logic"],
  "modules": [
    {
      "path": ":core:network",
      "softwareType": "androidLibrary",
      "readiness": "ready",
      "hardBlockers": [],
      "warnings": [
        {
          "line": 12,
          "pattern": "compileSdk = 35",
          "category": "settings-default",
          "recommendation": "Move to defaults {} in settings.gradle.dcl",
          "source": "project"
        }
      ],
      "isolatedProjectsViolations": [],
      "generatedDcl": "androidLibrary {\n    namespace = \"com.example.core.network\"\n}",
      "settingsDefaults": ["compileSdk", "minSdk", "jdkVersion"]
    }
  ],
  "summary": {
    "totalModules": 1,
    "ready": 1,
    "nearlyReady": 0,
    "hasBlockers": 0,
    "notReady": 0,
    "topBlockers": [],
    "thirdPartyBlockers": []
  }
}
```

**Schema notes:**
- `readiness`: `"ready"`, `"nearly-ready"`, `"has-blockers"`, `"not-ready"`
- Every finding has `source`: `"project"` or `"third-party:<plugin-id>"`
- `generatedDcl` is `null` for non-Ready modules
- `summary.thirdPartyBlockers` lists external plugins blocking migration

Print: `Wrote dcl-migration-report.json (N modules scanned)`.
