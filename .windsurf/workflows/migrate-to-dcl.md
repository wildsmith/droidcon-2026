---
description: Analyze build files for Declarative Gradle (DCL) migration readiness and Isolated Projects compatibility
---

# DCL Migration Readiness Audit

Analyze Gradle build files for migration readiness to Declarative Gradle (`.gradle.dcl`) and compatibility with Gradle's Isolated Projects feature. This skill scans a target module (or entire project), classifies blockers, generates equivalent DCL where possible, and outputs a structured migration report.

## Context

Declarative Gradle (DCL) replaces imperative `.gradle.kts` build scripts with a restricted, statically-analyzable subset of Kotlin (`.gradle.dcl`). DCL files cannot contain arbitrary code — only typed property assignments and nested configuration blocks. Isolated Projects is a complementary Gradle feature that forbids cross-project configuration access, enabling parallel project configuration.

**Key constraints of DCL files:**
- No `if`/`when`/`for`/`while` — no conditional or iterative logic
- No function calls (except dependency declarations and nested block accessors)
- No `val`/`var` declarations — no local variables
- No `ext`/`extra` properties
- No `plugins {}` block — plugin application is handled by the Software Type
- No `afterEvaluate {}`, `beforeEvaluate {}`, or lifecycle hooks
- No `allprojects {}`, `subprojects {}`, or cross-project access
- No custom task registration (`tasks.register {}`, `tasks.create {}`)
- No `import` statements
- One Software Type per project (`androidApplication`, `androidLibrary`, `javaLibrary`, `kotlinJvm`, etc.)

## Options

Before starting, ask the user which scan depth they prefer:

| Option | What gets scanned | When to use |
|---|---|---|
| **Standard** (default) | `build.gradle.kts`, `build.gradle`, and `settings.gradle.kts` files | Quick audit of build scripts only |
| **Deep** | Standard + all `.kt` files under `buildSrc/`, `build-logic/`, `gradle/`, and any `*-plugin*/src/` directories | When build logic lives in Kotlin source files (convention plugins, custom tasks, plugin implementations) |

If the user says "deep" or "include Kotlin", use Deep mode. Otherwise, default to Standard.

### Output Format

| Option | Description |
|---|---|
| **Markdown** (default) | Human-readable report with emoji categories, code fences, and recommendations |
| **JSON** (`--json`) | Machine-readable JSON for CI pipelines, dashboards, or downstream tooling |

If the user says "json", "--json", or "CI output", produce JSON output (see Step 7b).

## Pre-Scan Checks

### Gradle Version & DCL Availability

Before scanning, check the Gradle wrapper version in `gradle/wrapper/gradle-wrapper.properties`:
- **Gradle ≥ 8.13 with nightly:** DCL ecosystem plugins are available. Proceed with full migration.
- **Gradle 8.x (stable):** DCL files will not be recognized. The scan is still useful for **Isolated Projects readiness**, but warn the user that generated `.gradle.dcl` files cannot be used until they upgrade to a nightly or DCL-supporting Gradle version.
- **Gradle < 8:** Recommend upgrading Gradle first. Many Isolated Projects APIs (`tasks.named`, `configurations.named`) require 7.x+.

Include this in the report header:
> ⚠️ **Gradle X.Y detected.** DCL ecosystem plugins require Gradle ≥ 8.13 nightly. Generated `.gradle.dcl` files are for preview/planning only until you upgrade.

(Skip this warning if the project is already on a compatible version.)

### Groovy Build Files

If the project uses `.gradle` (Groovy) files instead of `.gradle.kts` (Kotlin DSL):
- **Flag this early in the report.** Groovy → DCL is a two-step migration: Groovy → KTS first, then KTS → DCL.
- Still scan for blocker patterns (the same anti-patterns exist in Groovy), but note that the generated DCL will need manual validation since the syntax translation is less direct.
- Recommend converting to KTS as a prerequisite. Gradle provides a [Groovy to KTS migration guide](https://docs.gradle.org/current/userguide/migrating_from_groovy_to_kotlin_dsl.html).

### Composite Builds (`includeBuild()`)

If `settings.gradle.kts` contains `includeBuild()` directives:
- **Included builds stay as KTS.** They are build logic (convention plugins, custom tasks) and do not migrate to DCL.
- In Deep mode, scan included build source files for Isolated Projects violations.
- Note any dependency substitution patterns (`substitution.substitute()`) — these are compatible with Isolated Projects but should be documented in the report.
- If the project uses composite builds for plugin development, this is a *good sign* — it means build logic is already separated from build scripts, which is the architecture DCL assumes.

## Available Software Types

These are the Software Types available in the Gradle Android/JVM ecosystem plugins (as of EAP3). Use them as the top-level block in generated `.gradle.dcl` files.

### Software Type Reference

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
| `testing { dependencies { } }` | Block | | Test dependencies (replaces `testImplementation`, `androidTestImplementation`) |

What moves to settings `defaults {}`: `compileSdk`, `minSdk`, `targetSdk`, `jdkVersion`
What the Software Type handles: plugin application (AGP, KGP), `compileOptions`, `kotlinOptions`

**`androidLibrary`** — same as `androidApplication` except no `applicationId`, `versionCode`, `versionName`, or `applicationIdSuffix`. Minimal form (when all config is in settings defaults):

```kotlin
androidLibrary {
    namespace = "com.example.core.utils"
}
```

**`javaLibrary`** / **`kotlinJvmLibrary`** — available properties:

| Property / Block | Notes |
|---|---|
| `dependencies { }` | `api()`, `implementation()`, `compileOnly()`, `runtimeOnly()` |
| `testing { dependencies { } }` | Test dependencies |

### Project Features

Project Features are opt-in capabilities that nest inside any Software Type block. They add functionality without requiring explicit plugin application.

| Feature | DSL | Notes |
|---|---|---|
| Kotlin Serialization | `kotlinSerialization { version = libs.versions.x; jsonEnabled = true }` | Adds kotlinx-serialization plugin + runtime |
| Testing | `testing { dependencies { implementation(libs.junit) } }` | Test dependencies and configuration |
| Hilt | `hilt { }` | Hilt/Dagger injection (when available) |
| Compose | `compose { }` | Jetpack Compose (when available as a feature) |

Features not yet available can be noted as "pending" in the migration report.

### Settings File (`settings.gradle.dcl`)

The settings file registers ecosystem plugins and defines defaults shared across all projects. See Step 6 for the full template.

Key sections: `pluginManagement {}`, `plugins {}`, `dependencyResolutionManagement {}`, `defaults {}`.

## Steps

### 1. Identify Target Scope

If the user provides a specific module path, scan only that module's `build.gradle.kts` (or `build.gradle`). Otherwise, scan all Gradle build files in the project.

**Standard mode:** Scan `build.gradle.kts`, `build.gradle`, and `settings.gradle.kts` files. Exclude `buildSrc/`, `build-logic/`, and `gradle/` directories from the migration report (they are build logic and stay as KTS).

**Version catalog detection:** Check for `gradle/libs.versions.toml` (or any `.toml` file referenced in `settings.gradle.kts` via `versionCatalogs {}`). If found, parse it and build a lookup map of:
- **Libraries:** `[libraries]` entries → maps GAV coordinates to catalog accessors (e.g., `com.squareup.okhttp3:okhttp` → `libs.okhttp`)
- **Plugins:** `[plugins]` entries → maps plugin IDs to catalog accessors (e.g., `org.jetbrains.kotlin.android` → `libs.plugins.kotlin.android`)
- **Versions:** `[versions]` entries → maps version keys to catalog version accessors (e.g., `kotlinxSerialization = "1.6.3"` → `libs.versions.kotlinx.serialization`)

This map is used in Step 5 to preserve version catalog references in generated DCL files. DCL fully supports version catalog accessors — they are not imperative code.

**Type-safe project dependencies (`projects` catalog):** Starting with Gradle 9, the `projects` accessor is enabled by default, providing type-safe project dependency references. Check the Gradle wrapper version (in `gradle/wrapper/gradle-wrapper.properties`) and `settings.gradle.kts` for:
- **Gradle ≥ 9:** `projects` catalog is available by default. Use `projects.core.network` instead of `project(":core:network")`.
- **Gradle < 9 with opt-in:** Look for `enableFeaturePreview("TYPESAFE_PROJECT_ACCESSORS")` in `settings.gradle.kts`. If present, `projects` is available.
- **Otherwise:** Fall back to the legacy `project(":path")` syntax.

The `projects` accessor maps project paths to dot-separated, camelCased accessors. Kebab-case and snake_case path segments are converted to camelCase:
- `:core:network` → `projects.core.network`
- `:core:legacy-network:ktor` → `projects.core.legacyNetwork.ktor`
- `:feature:sign-in` → `projects.feature.signIn`
- `:ui:card-details-ui` → `projects.ui.cardDetailsUi`

**Deep mode:** In addition to Standard, also scan `.kt` source files in:
- `buildSrc/src/`
- `build-logic/*/src/`
- `gradle/*/src/`
- Any directory matching `*-plugin*/src/` or `*-plugins/src/`
- Convention plugin files (typically `*.gradle.kts` in `buildSrc/src/main/kotlin/`)

Deep mode is important because many projects implement Gradle plugins, convention plugins, and custom tasks in regular `.kt` files. These contain the same patterns (cross-project access, eager APIs, `afterEvaluate`) that block Isolated Projects and DCL adoption. Flag these with the same Isolated Projects Violations rules, but note they are **build logic files** — they don't migrate to DCL themselves, but they must be fixed for Isolated Projects compatibility.

Also scan `settings.gradle.kts` for project-wide issues (e.g., `allprojects {}` in the root build file).

### 2. Classify Each Module

Determine the Software Type based on applied plugins:

| Applied Plugin | Software Type |
|---|---|
| `com.android.application` | `androidApplication` |
| `com.android.library` | `androidLibrary` |
| `java-library` / `org.gradle.java-library` | `javaLibrary` |
| `org.jetbrains.kotlin.jvm` | `kotlinJvm` |
| `java-gradle-plugin` | `gradlePlugin` (stays KTS — build logic) |
| `org.jetbrains.kotlin.multiplatform` | `kotlinMultiplatform` (prototype only) |

**Important:** Gradle plugin projects (`java-gradle-plugin`) should NOT be migrated to DCL. They are build logic and remain as `.gradle.kts` in an included build.

### 3. Check for DCL Blockers

Scan each build file (and `.kt` build logic files in Deep mode) for patterns incompatible with DCL. Classify each finding.

**Violation source attribution:** For every blocker or violation found, determine whether it originates from:
- **Project code** — patterns written directly in the project's build files or build logic (fixable by the project team)
- **Third-party plugin** — patterns introduced by an applied plugin from an external dependency (e.g., `com.google.dagger.hilt.android`, `com.google.firebase.crashlytics`, `androidx.navigation.safeargs`). These are identified by:
  - The violation occurring inside a plugin's applied configuration (e.g., inside a `plugins.withType<T>` callback from a classpath dependency)
  - The offending code living in a JAR on the build classpath rather than in project source
  - The pattern being generated by a plugin's `apply()` method (e.g., a plugin that calls `project.afterEvaluate {}`)

Tag each finding with `source: "project"` or `source: "third-party:<plugin-id>"`. This distinction is critical because:
- **Project code violations** → the team can fix these directly
- **Third-party plugin violations** → the team must wait for the plugin author to update, find an alternative plugin, or work around it

When attribution is ambiguous (e.g., the violation is in a convention plugin that wraps a third-party plugin), attribute to `"project"` since the project team controls the convention plugin.

#### ❌ Hard Blockers (cannot migrate until resolved)

| Pattern | Detection | Fix |
|---|---|---|
| **Conditional logic** | `if (`, `when (`, `when {`, `for (`, `while (` | Move to convention plugin / Software Type implementation |
| **Function calls in config** | `file(`, `rootProject.file(`, `System.getenv(`, `Runtime.exec(`, `exec {` | Move to task execution time or Software Type |
| **`afterEvaluate {}`** | Literal string match | Replace with lazy APIs: `provider {}`, `configureEach {}`, `tasks.named {}` |
| **`beforeEvaluate {}`** | Literal string match | Use `plugins.withType<T> {}` or settings plugins |
| **Cross-project access** | `project(":other").`, `rootProject.`, `parent.` (config reads) | Use proper dependency declarations or settings defaults |
| **`allprojects {}`** | Literal string match | Move to `dependencyResolutionManagement {}` in settings or convention plugins |
| **`subprojects {}`** | Literal string match | Move to convention plugins applied per-project |
| **Custom task registration** | `tasks.register {`, `tasks.create {`, `task(` | Move to included build plugin |
| **`extra` / `ext` properties** | `extra[`, `extra.set(`, `ext[`, `ext.` | Replace with typed Software Type properties |
| **`buildscript {}`** | Literal block | Migrate to `plugins {}` block first, then to DCL Software Type |
| **Local variables** | `val ` / `var ` at top level or inside `android {}` | Inline the value or move to settings defaults |
| **String interpolation in config** | `"${...}"` in property assignments | Use literal values; dynamic computation belongs in Software Type |
| **`apply(plugin = ...)` / `apply(from = ...)`** | Literal match | Use `plugins {}` block (pre-DCL step) or Software Type |

#### ⚠️ Warnings (migrate to settings defaults or Software Type)

| Pattern | Detection | Recommendation |
|---|---|---|
| **`compileSdk` / `minSdk` / `targetSdk`** | Explicit assignment in `android {}` | Move to `defaults {}` block in `settings.gradle.dcl` |
| **`jvmTarget` / `sourceCompatibility` / `targetCompatibility`** | Explicit assignment | Derive from `jdkVersion` in settings defaults |
| **`composeOptions { kotlinCompilerExtensionVersion = ... }`** | Explicit assignment | Derived automatically from KGP version in DCL |
| **`kotlinOptions { freeCompilerArgs += ... }`** | Explicit assignment | Move to Software Type or settings defaults |

#### 🔶 Isolated Projects Violations (blocks parallel configuration)

These patterns are compatible with KTS but block Isolated Projects (and therefore block full DCL benefits):

| Pattern | Detection | Fix |
|---|---|---|
| **`allprojects {}`** | Function call with lambda | Move to settings `dependencyResolutionManagement {}` or convention plugins |
| **`subprojects {}`** | Function call with lambda | Convention plugins applied per-project |
| **`project(":path")` config access** | `project(":` followed by `.` property access | Use dependency declarations instead |
| **`rootProject.` property access** | Dot-qualified on `rootProject` | Wire via settings or included build |
| **`childProjects` property** | Property access | Convention plugins |
| **`parent.` property access** | Dot-qualified on `parent` | Convention plugins or settings defaults |
| **Eager task APIs** | `tasks.create(`, `tasks.getByName(`, `tasks.getByPath(`, `tasks.findByPath(`, `tasks.all {`, `tasks.matching {`, `tasks.withType<T> {` (without `.configureEach`), `whenTaskAdded {` | Replace with `tasks.register()`, `tasks.named()`, `tasks.withType<T>().configureEach {}` |
| **`dependsOn()` with task refs** | `dependsOn(tasks.named(`, `dependsOn(tasks.getByName(` | Wire task outputs to inputs via `Property<T>` |
| **`project` access in task actions** | `project.` inside `@TaskAction` or `doFirst {}`/`doLast {}` | Wire via `@Input`, `Property<T>`, `Provider<T>` |
| **`extensions["name"] as Type`** | Array access on `extensions` with cast | Use `extensions.getByType<Type>()` |
| **`@PathSensitive(ABSOLUTE)`** | Annotation match | Use `NONE` for files, `RELATIVE` for directories |
| **`findByName()` on containers** | Method call | Use `named()` or `names.contains()` |
| **`configurations.getByName()`** | Method call | Use `configurations.named()` |
| **`tasks.replace()`** | Method call | Use `tasks.register()` with proper wiring |

### 4. Assess Readiness

Classify each module into one of these categories based on the findings from Step 3:

| Category | Criteria | Action |
|---|---|---|
| ✅ **Ready** | Zero hard blockers. May have warnings (settings defaults). | Generate DCL, migrate now. |
| ⚠️ **Nearly Ready** | Zero hard blockers but has Isolated Projects violations (all mechanically fixable). | Auto-fix violations, then generate DCL. |
| 🔶 **Has Blockers** | 1–3 hard blockers, all with clear fixes. | List fixes with priority order. Offer to fix mechanical ones. |
| ❌ **Not Ready** | 4+ hard blockers, or blockers requiring architectural redesign (e.g., heavy conditional logic, cross-project task wiring). | Provide a migration roadmap but do not generate DCL. |

**Why categories instead of scores:** A module with a single `afterEvaluate {}` and 20 clean dependencies is "🔶 Has Blockers" (one fix away), not "70%". A module with `compileSdk` set locally but no blockers is "✅ Ready" (trivial settings move), not "97%". Categories communicate actionability better than arbitrary numbers.

### 5. Generate Equivalent DCL

For modules classified as **✅ Ready** or **⚠️ Nearly Ready**, generate the equivalent `.gradle.dcl` file using the Software Type references above.

**Version catalog handling:** When generating DCL, preserve and improve the dependency format from the source build file:
- If the source uses `libs.x` catalog accessors (e.g., `implementation(libs.okhttp)`), keep them as-is. Version catalog accessors are fully supported in DCL.
- If the source uses raw GAV strings (e.g., `implementation("com.squareup.okhttp3:okhttp:4.12.0")`), check the version catalog lookup map from Step 1. If a matching entry exists, replace the raw string with the catalog accessor. If no match, keep the raw string.
- If the source uses `platform()` / BOM declarations with catalog refs (e.g., `implementation(platform(libs.compose.bom))`), preserve them.

**Project dependency handling:**
- If the `projects` catalog is available (Gradle ≥ 9, or `TYPESAFE_PROJECT_ACCESSORS` enabled), convert `project(":...")` to `projects.x.y` using camelCase for kebab-case/snake_case segments (e.g., `project(":core:legacy-network")` → `projects.core.legacyNetwork`).
- If not available, keep the legacy `project(":...")` syntax.

**Dependency ordering:** Sort all dependencies alphabetically within each configuration group (`api` before `implementation`, then alphabetical within each). This applies to both the `dependencies { }` block and `testing { dependencies { } }` block.

Ordering rules:
1. Group by configuration: `api` → `compileOnly` → `implementation` → `runtimeOnly`
2. Within each group: `platform()`/BOM entries first, then `libs.x` entries alphabetically, then `projects.x` entries alphabetically
3. Apply the same ordering inside `testing { dependencies { } }`

**Mapping rules:**

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
| `android { buildTypes { debug { applicationIdSuffix = "..." } } }` | `buildTypes { debug { applicationIdSuffix = "..." } }` |
| `dependencies { implementation(libs.x) }` | `dependencies { implementation(libs.x) }` (preserve, sort alphabetically) |
| `dependencies { implementation("group:artifact:version") }` | `dependencies { implementation(libs.x) }` if catalog match exists, else keep raw |
| `dependencies { api(libs.x) }` | `dependencies { api(libs.x) }` |
| `dependencies { implementation(project(":path")) }` | `dependencies { implementation(projects.path) }` if `projects` catalog available |
| `dependencies { testImplementation(libs.x) }` | `testing { dependencies { implementation(libs.x) } }` |
| `dependencies { androidTestImplementation(libs.x) }` | `testing { dependencies { implementation(libs.x) } }` |
| `plugins { id("kotlinx-serialization") }` | `kotlinSerialization { version = libs.versions.x }` (Project Feature, use catalog version if available) |
| `compileSdk`, `minSdk`, `targetSdk` | Remove — move to settings `defaults {}` |
| `compileOptions`, `kotlinOptions`, `jvmTarget` | Remove — derived from `jdkVersion` in settings defaults |
| `composeOptions { kotlinCompilerExtensionVersion = ... }` | Remove — derived automatically in DCL |
| `plugins {}` block | Remove entirely — Software Type handles plugin application |

### 6. Generate Settings Defaults Recommendations

Based on common values across scanned modules, suggest a complete `settings.gradle.dcl` file:

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

Note: `dependencyResolutionManagement` with `FAIL_ON_PROJECT_REPOS` centralizes repository declarations. This is a Gradle best practice and prerequisite for removing `repositories {}` blocks from individual build files.

### 7. Output Migration Report

For each module, output a structured report:

```markdown
## DCL Migration Report: :<module-path>

📊 **Readiness: [✅ Ready | ⚠️ Nearly Ready | � Has Blockers | ❌ Not Ready]**

**Software Type:** `androidLibrary` | `androidApplication` | `javaLibrary` | etc.

### ❌ Hard Blockers (N found)
1. **Line XX**: `if (isCI) { ... }` — Conditional logic
   → Move to convention plugin or Software Type implementation
2. **Line XX**: `afterEvaluate { ... }` — Lifecycle hook
   → Replace with lazy APIs (provider {}, configureEach {})

### ⚠️ Warnings (N found)
1. **Line XX**: `compileSdk = 35` — Should be in settings defaults
2. **Line XX**: `jvmTarget = "17"` — Derive from jdkVersion

### 🔶 Isolated Projects Violations (N found)
1. **Line XX**: `allprojects { ... }` — Cross-project configuration
   → Move to dependencyResolutionManagement {} in settings
2. **Line XX**: `tasks.create("myTask") { ... }` — Eager task creation
   → Use tasks.register("myTask") { ... }

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

### 8. Project-Wide Summary (when scanning multiple modules)

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
These violations originate from external plugins and cannot be fixed by the project team:
| Plugin ID | Violation | Affected Modules |
|---|---|---|
| `com.google.dagger.hilt.android` | `afterEvaluate {}` in plugin apply | N |
| `com.google.firebase.crashlytics` | Config-time file manipulation | N |

### Recommended Migration Order
1. Start with ✅ Ready leaf library modules
2. Address settings-level issues (allprojects → dependencyResolutionManagement)
3. Fix ⚠️ Nearly Ready modules (mechanical Isolated Projects fixes)
4. Tackle � Has Blockers modules one blocker at a time
5. App modules last (most complex, likely ❌ Not Ready)
```

### 9. Perform the Migration (when the user confirms)

After presenting the report, ask the user: **"Would you like me to migrate the ready modules now?"**

If yes, for each module classified as ✅ Ready or ⚠️ Nearly Ready:

1. **Create the `.gradle.dcl` file** — write the generated DCL to `build.gradle.dcl` alongside the existing build file
2. **Do NOT delete the original** `build.gradle.kts` / `build.gradle` — Gradle supports mixed builds; the user can validate before removing it
3. **Generate `settings.gradle.dcl`** if it doesn't exist — include the `pluginManagement`, `plugins`, `dependencyResolutionManagement`, and `defaults` blocks
4. **Fix Isolated Projects violations in-place** when the fix is mechanical:
   - `tasks.create("name")` → `tasks.register("name")`
   - `tasks.getByName("name")` → `tasks.named("name")`
   - `configurations.getByName("name")` → `configurations.named("name")`
   - `tasks.all { }` → `tasks.configureEach { }`
   - `tasks.withType<T> { }` → `tasks.withType<T>().configureEach { }`
   - `extensions["name"] as Type` → `extensions.getByType<Type>()`
   - `findByName("name")` → `named("name")` or `names.contains("name")`
5. **Flag but do NOT auto-fix** patterns that require architectural decisions:
   - `allprojects {}` / `subprojects {}` — requires choosing between convention plugins or settings defaults
   - `afterEvaluate {}` — requires understanding the deferred logic
   - Cross-project access — requires dependency architecture changes
   - Custom task registration — requires moving to included build
   - Conditional logic — requires Software Type design decisions
6. **Output a summary** of files created, files modified, and remaining manual steps

**Important:** Always create new files rather than overwriting. Let the user diff, validate, and remove the originals themselves.

### 10. Validate the Migration

After creating files, suggest validation commands the user can run:

```bash
# Verify the generated DCL files parse correctly (requires DCL-compatible Gradle)
./gradlew --dry-run :module:help

# Check for Isolated Projects violations across the whole build
./gradlew help --isolated-projects

# Run a full build to verify nothing broke
./gradlew :module:assemble
```

If the project is not yet on a DCL-compatible Gradle version, note that the DCL files are for preview/planning and suggest running only the `--isolated-projects` check against the existing build files.

**Important:** The AI cannot run these commands itself. Present them to the user with context on what each verifies.

### 7b. JSON Output (when `--json` is requested)

When the user requests JSON output, write the report as a single JSON file (e.g., `dcl-migration-report.json`) instead of printing Markdown. The schema:

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
      "generatedDcl": "androidLibrary {\n    namespace = \"com.example.core.network\"\n    dependencies {\n        implementation(libs.okhttp)\n    }\n}",
      "settingsDefaults": ["compileSdk", "minSdk", "jdkVersion"]
    },
    {
      "path": ":app",
      "softwareType": "androidApplication",
      "readiness": "not-ready",
      "hardBlockers": [
        {
          "line": 15,
          "pattern": "if (isCI) { ... }",
          "category": "conditional-logic",
          "recommendation": "Move to convention plugin or Software Type",
          "source": "project"
        },
        {
          "line": 88,
          "pattern": "afterEvaluate { ... }",
          "category": "lifecycle-hook",
          "recommendation": "Replace with lazy APIs",
          "source": "third-party:com.google.dagger.hilt.android"
        }
      ],
      "warnings": [],
      "isolatedProjectsViolations": [],
      "generatedDcl": null,
      "settingsDefaults": []
    }
  ],
  "summary": {
    "totalModules": 2,
    "ready": 1,
    "nearlyReady": 0,
    "hasBlockers": 0,
    "notReady": 1,
    "topBlockers": [
      { "pattern": "conditional-logic", "count": 1, "projectCount": 1, "thirdPartyCount": 0 },
      { "pattern": "lifecycle-hook", "count": 1, "projectCount": 0, "thirdPartyCount": 1 }
    ],
    "thirdPartyBlockers": [
      { "pluginId": "com.google.dagger.hilt.android", "violation": "afterEvaluate {}", "affectedModules": 1 }
    ]
  }
}
```

**Key design decisions:**
- `readiness` uses lowercase kebab-case: `"ready"`, `"nearly-ready"`, `"has-blockers"`, `"not-ready"`
- Every finding has a `source` field: `"project"` or `"third-party:<plugin-id>"`
- `generatedDcl` is `null` for modules that are not Ready or Nearly Ready
- `summary.topBlockers` splits counts by `projectCount` and `thirdPartyCount`
- `summary.thirdPartyBlockers` lists external plugins blocking migration

Write the JSON file to the project root (or the path the user specifies). Print a one-line confirmation: `Wrote dcl-migration-report.json (N modules scanned)`.

The JSON output is designed for:
- **CI pipelines** — fail a build if any module regresses from Ready to Has Blockers
- **Dashboards** — track migration progress over time
- **Downstream tooling** — feed into Jira ticket creation, Slack notifications, etc.
