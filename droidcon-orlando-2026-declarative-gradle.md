<!-- .slide: data-visibility="hidden" -->

# Speaker Note Key (for presenter only)

| Tag | Meaning | Say out loud? |
|---|---|---|
| `[PACING: ...]` | Stage direction. How long to pause and what to do during silence. | No. Stay silent, scan audience. |
| `[TRANSITION: ...]` | Bridge sentence into the next slide. Say it, then advance. | Yes. |
| `[AUDIENCE]` | Interactive moment: show of hands, rhetorical question. | Yes. |

Everything between `[PACING]` and `[TRANSITION]` is your scripted talking point.

---

# The End of `build.gradle`
## Embracing Declarative Gradle and ~~Software~~ Project Types

Tom Belk | Android Platform Lead | <img src="https://upload.wikimedia.org/wikipedia/commons/9/98/Capital_One_logo.svg" class="logo-inline logo-white-bg" alt="Capital One" style="height:1.2em; vertical-align:middle;">

DroidCon Orlando | July 2026

> Speaker Notes: Welcome everyone. I'm Tom Belk, Android Platform Lead at Capital One. We build developer experience tooling, CI, IDE plugins, feature architecture, networking, and testing infrastructure used by 9 applications. Today we're going to talk about the future of how you configure your Android builds. [PACING: point at the strikethrough in the title.] And yes, that strikethrough in my title is not a design choice. Somewhere between building this talk and standing here, Gradle deleted the `@SoftwareType` annotation outright and renamed the whole concept from 'Software Type' to 'Project Type' which ate my original title and forced me to rewrite a chunk of these slides at the last minute. If a stray 'Software Type' sneaks through anywhere today, apologies. The good news: it's the same concept, just renamed and promoted out of Gradle's internal packages which is actually a great sign that this stuff is maturing fast.

---

# About Me

<img src="profile.JPG" class="profile-img" alt="Tom Belk">

- **Android Platform Lead** at Capital One
- DevEx tooling for **9 applications**
- CI, IDE, Feature Architecture, Networking, Testing
- Gradle plugin author and build infrastructure engineer
- 2,500+ module monorepo

> Speaker Notes: Quick intro. My team builds the platform layer that 9 Android applications depend on. We author custom Gradle plugins, manage a monorepo with thousands of modules, and care deeply about build performance and developer experience.

---

# Agenda

<div style="position:relative; padding-left:50px; margin-top:10px; font-size:0.82em;">
<div style="position:absolute; left:20px; top:0; bottom:0; width:0; border-left:3px dashed rgba(245,166,35,0.6); opacity:0.9;"></div>
<div style="position:relative; margin-bottom:12px;"><span style="position:absolute; left:-38px; top:3px; font-size:0.75em;">🌅</span> <strong style="color:#f5a623;">What is Declarative Gradle?</strong><br><span style="color:#ccc; font-size:0.85em;">and why should you care?</span></div>
<div style="position:relative; margin-bottom:12px;"><span style="position:absolute; left:-38px; top:3px; font-size:0.75em;">🐚</span> <strong style="color:#f5a623;">The Android experience today</strong><br><span style="color:#ccc; font-size:0.85em;">the problem we're solving</span></div>
<div style="position:relative; margin-bottom:12px;"><span style="position:absolute; left:-38px; top:3px; font-size:0.75em;">🌊</span> <strong style="color:#00c9db;">DCL syntax & real examples</strong><br><span style="color:#ccc; font-size:0.85em;">what it looks like in practice</span></div>
<div style="position:relative; margin-bottom:12px;"><span style="position:absolute; left:-38px; top:3px; font-size:0.75em;">🏄</span> <strong style="color:#00c9db;">Isolated Projects</strong><br><span style="color:#ccc; font-size:0.85em;">the complementary enforcement feature</span></div>
<div style="position:relative; margin-bottom:12px;"><span style="position:absolute; left:-38px; top:3px; font-size:0.75em;">🏝️</span> <strong style="color:#3DDC84;">Migration path & pitfalls</strong><br><span style="color:#ccc; font-size:0.85em;">how to get there</span></div>
<div style="position:relative; margin-bottom:12px;"><span style="position:absolute; left:-38px; top:3px; font-size:0.75em;">🐠</span> <strong style="color:#3DDC84;">Plugin authoring in DCL</strong><br><span style="color:#ccc; font-size:0.85em;">for build engineers</span></div>
<div style="position:relative; margin-bottom:12px;"><span style="position:absolute; left:-38px; top:3px; font-size:0.75em;">🤖</span> <strong style="color:#3DDC84;">AI-assisted migration</strong><br><span style="color:#ccc; font-size:0.85em;">tooling for scale</span></div>
<div style="position:relative; margin-bottom:0;"><span style="position:absolute; left:-38px; top:3px; font-size:0.75em;">🌴</span> <strong style="color:#fff;">Resources & Q&A</strong></div>
</div>

> Speaker Notes: Here's our roadmap for the next 30-35 minutes. We'll start with what Declarative Gradle actually is and why you should care. Then we'll look at the Android experience today - the problem we're solving. From there, real DCL syntax and examples, Isolated Projects, the migration path and common pitfalls, plugin authoring, AI-assisted migration tooling, and we'll wrap up with resources and Q&A. [AUDIENCE] Quick show of hands - how many of you have more than 10 modules in your Android project? More than 50? More than 500? Ok great - this talk is for all of you, but the pain scales with module count.

---

# The Problem Statement

<div class="horror-scroll-container">

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.serialization")
    id("com.google.dagger.hilt.android")
    id("androidx.navigation.safeargs.kotlin")
    id("com.google.firebase.crashlytics")
    id("com.google.gms.google-services")
}

android {
    compileSdk = 35
    namespace = "com.example.myapp"

    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = 26
        targetSdk = 35
        versionCode = 142
        versionName = "6.43.0"
        testInstrumentationRunner = "com.example.CustomTestRunner"
        multiDexEnabled = true
        vectorDrawables.useSupportLibrary = true
        resConfigs("en", "es", "fr", "de", "ja", "ko", "zh")
    }

    signingConfigs {
        getByName("debug") { storeFile = file("debug.keystore") }
        create("release") {
            storeFile = file(System.getenv("KEYSTORE_PATH") ?: "release.keystore")
            storePassword = System.getenv("KEYSTORE_PASSWORD") ?: ""
            keyAlias = System.getenv("KEY_ALIAS") ?: ""
            keyPassword = System.getenv("KEY_PASSWORD") ?: ""
        }
    }

    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            isMinifyEnabled = false
            isShrinkResources = false
        }
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            signingConfig = signingConfigs.getByName("release")
        }
        create("staging") {
            initWith(getByName("release"))
            applicationIdSuffix = ".staging"
            matchingFallbacks += listOf("release")
        }
    }

    flavorDimensions += listOf("environment", "store")
    productFlavors {
        create("production") { dimension = "environment" }
        create("developer") { dimension = "environment" }
        create("playStore") { dimension = "store" }
        create("galaxyStore") { dimension = "store" }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
        isCoreLibraryDesugaringEnabled = true
    }

    kotlinOptions { 
        jvmTarget = "17" 
    }
    buildFeatures { 
        compose = true
        buildConfig = true
        viewBinding = true 
    }
    composeOptions { kotlinCompilerExtensionVersion = "1.5.11" }

    packaging {
        resources { 
            excludes += "/META-INF/{AL2.0,LGPL2.1}" 
        }
    }

    lint {
        abortOnError = false
        checkDependencies = true 
    }
    testOptions {
        unitTests {
            isReturnDefaultValues = true 
        }
    }
}

dependencies {
    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.analytics)
    implementation(libs.firebase.crashlytics)
    implementation(libs.hilt.android)
    kapt(libs.hilt.compiler)
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.lifecycle.runtime)
    implementation(libs.androidx.activity.compose)
    implementation(platform(libs.compose.bom))
    implementation(libs.compose.ui)
    implementation(libs.compose.material3)
    implementation(libs.compose.navigation)
    implementation(libs.kotlinx.serialization.json)
    implementation(libs.retrofit)
    implementation(libs.okhttp)
    implementation(libs.okhttp.logging)
    implementation(libs.coil.compose)
    implementation(libs.timber)
    implementation(libs.datastore.preferences)
    testImplementation(libs.junit)
    testImplementation(libs.mockk)
    testImplementation(libs.turbine)
    testImplementation(libs.coroutines.test)
    androidTestImplementation(libs.espresso.core)
    androidTestImplementation(libs.compose.ui.test)
    coreLibraryDesugaring(libs.desugar.jdk)
}

afterEvaluate {
    tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile>().configureEach {
        kotlinOptions.freeCompilerArgs += listOf("-opt-in=kotlin.RequiresOptIn")
    }
}
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.serialization")
    id("com.google.dagger.hilt.android")
    id("androidx.navigation.safeargs.kotlin")
    id("com.google.firebase.crashlytics")
    id("com.google.gms.google-services")
}

android {
    compileSdk = 35
    namespace = "com.example.myapp"

    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = 26
        targetSdk = 35
        versionCode = 142
        versionName = "6.43.0"
        testInstrumentationRunner = "com.example.CustomTestRunner"
        multiDexEnabled = true
        vectorDrawables.useSupportLibrary = true
        resConfigs("en", "es", "fr", "de", "ja", "ko", "zh")
    }

    signingConfigs {
        getByName("debug") { storeFile = file("debug.keystore") }
        create("release") {
            storeFile = file(System.getenv("KEYSTORE_PATH") ?: "release.keystore")
            storePassword = System.getenv("KEYSTORE_PASSWORD") ?: ""
            keyAlias = System.getenv("KEY_ALIAS") ?: ""
            keyPassword = System.getenv("KEY_PASSWORD") ?: ""
        }
    }

    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            isMinifyEnabled = false
            isShrinkResources = false
        }
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            signingConfig = signingConfigs.getByName("release")
        }
    }
```

</div>
<p style="text-align:center; margin-top:12px; font-size:0.75em; color:#f5a623;">↕ ...it just keeps going</p>

> Speaker Notes: [PACING: Let the auto-scroll run for 5-6 seconds. The audience will start laughing when they realize it doesn't stop. Then:] This is a real-world build.gradle.kts. 120+ lines. Signing configs, flavor dimensions, compiler arguments, afterEvaluate hacks. It's Kotlin code masquerading as configuration. For a new engineer, what's configuration and what's code? The line is blurry. And this is ONE module. Imagine 2,500 of these.

---

<div style="text-align:center; padding:40px 20px; position:relative;">
<div style="background:#1e1e2e; border:1px solid rgba(0,201,219,0.3); border-radius:12px; padding:30px 40px; display:inline-block; max-width:700px; position:relative;">
<p style="font-size:1.4em; margin:0 0 20px 0;">Stages of reading someone else's <code>build.gradle.kts</code>:</p>
<p style="font-size:1.1em; text-align:left; margin:8px 0;">1. "Ok, plugins block, I get it"</p>
<p style="font-size:1.1em; text-align:left; margin:8px 0;">2. "What does this <code>afterEvaluate</code> do..."</p>
<p style="font-size:1.1em; text-align:left; margin:8px 0;">3. "Why is there a <code>for</code> loop in a config file"</p>
<p style="font-size:1.1em; text-align:left; margin:8px 0;">4. <strong style="color:#f5a623;">"I'll just copy it and hope it works"</strong></p>
<span style="position:absolute; bottom:-10px; right:-20px; font-size:2.5em;">😕</span>
</div>
</div>

> Speaker Notes: [PACING: Let the list build tension. Pause after item 4 for the laugh.] We've all been there. The mental model mismatch is real: it looks like configuration, but it's actually a full Kotlin program. That's what Declarative Gradle fixes. [TRANSITION: So what IS Declarative Gradle? Let me define it clearly.]

---

# What IS Declarative Gradle?

- A **new configuration language** (DCL) for Gradle

  File extension: `.gradle.dcl`

- A **strict subset of Kotlin** syntax - familiar, but constrained

- Express **what** you want, not **how** to achieve it


- Separates **software definition** from **build logic**


- Actively evolving, check the official docs for current status

- Backed by Gradle Inc. - ongoing work now landing in core Gradle releases

> Speaker Notes: It's Gradle's initiative to create a clear separation between what your project is versus how it gets built. The DCL language is a restricted subset of Kotlin - no arbitrary code, no imperative logic, just pure configuration. This initiative's status changes frequently so check the Gradle newsletter or declarative.gradle.org for the latest details.

---

# Key Principles (from Gradle)

1. **Ease of use** for regular software developers
    - Define software without understanding build system internals

2. **Complete flexibility** for build engineers
    - Power users retain full automation capabilities

3. **Excellent IDE integration**
    - Fast, reliable imports; tooling can safely modify definitions


> Speaker Notes: These three principles drive the project; ease of use, complete flexibility, and IDE integration. The first is critical - most app developers shouldn't need to understand how Gradle works internally just to add a dependency or change a compile SDK version.

---

# Why Should You Migrate?

| Pain Point Today | DCL Solves It |
|---|---|
| Build scripts are code - hard to parse tooling-wise | DCL is statically analyzable |
| IDE import is slow and fragile | Parallel configuration, faster sync |
| New devs confused by build logic | Pure declaration, no imperative code |
| `allprojects`/`subprojects` cause coupling | Project Types + defaults in settings |
| Plugin conflicts and ordering issues | Controlled, typed plugin application |

> Speaker Notes: [PACING: Give 3 seconds for the audience to scan the table, then anchor on row 2.] Why should you migrate? Quick note on row 2 - IDE import is slow and fragile. How many of you have waited 3+ minutes for a Gradle sync after touching a build file? That's because Gradle has to evaluate every build script sequentially just to understand your project structure. DCL eliminates that by being statically analyzable. The IDE can understand your project without executing anything. The other rows follow the same pattern: today's pain point on the left, DCL's structural fix on the right. [TRANSITION: The enterprise version of this story is even more compelling.]

---

# Why for Enterprise?

- **Configuration time scales linearly** with module count today
- Cross-project coupling via `allprojects {}` blocks prevents parallelism
- Declarative files enable **parallel configuration** of projects
- Build engineers define **Project Types** once, developers reuse them
- Tooling can **auto-modify** DCL files safely (mutations API)

> Speaker Notes: [PACING: Let the audience read for 2-3 seconds, then anchor on the first bullet.] Unfortunately, configuration time scales linearly with module count. At Capital One, we have over 2,500 modules. A full project configuration takes over 4 minutes. DCL files can be configured in parallel because they're guaranteed to be isolated. No code that reaches into other projects. The rest of these items follow from that one change. [TRANSITION: There's also a security angle that most people don't think about.]

---

# Security & Auditability

<div style="text-align:center; margin:10px 0;">
<div style="background:#1e1e2e; border:1px solid rgba(61,220,132,0.4); border-radius:12px; padding:20px 30px; display:inline-block;">
<p style="font-size:1.2em; margin:0;">No one can sneak a <code>Runtime.exec()</code> into a <code>.dcl</code> file.</p>
</div>
</div>

| Concern | `.gradle.kts` | `.gradle.dcl` |
|---|---|---|
| Arbitrary code execution | Possible | Impossible |
| Network calls in config | Possible | Impossible |
| File system manipulation | Possible | Impossible |
| Static analysis by security tools | Difficult | Trivial |
| Auditable by compliance teams | Requires Kotlin expertise | Plain data, anyone can review |

> Speaker Notes: [PACING: Let the callout box land for 2 seconds, then deliver.] Read that snippet at the top. No one can sneak a Runtime.exec() into a .dcl file. Today, a build.gradle.kts file can do anything. Download files, execute shell commands, exfiltrate environment variables. Look at the table: every single concern in the left column is "Possible" for KTS and "Impossible" for DCL. That last row is the one that resonates with leadership: auditable by compliance teams without requiring Kotlin expertise. Your AppSec team can review a DCL file. They cannot meaningfully review a KTS file. [TRANSITION: Now let's talk about what this actually means for build speed.]

---

# The Configuration Time Problem

| Metric | Sequential (Today) | Parallel (DCL) | Speedup |
|---|---|---|---|
| 100 modules | ~12s config | ~2s config | **6x** |
| 500 modules | ~55s config | ~4s config | **14x** |
| 2,500 modules | ~4.5min config | ~8s config | **34x** |

<small>*Theoretical maximums based on sequential behavior and perfect parallelism. Real-world gains will be lower, but even 5-10x at enterprise scale changes the game.*</small>

**Why?** Today, projects can access each other's configuration → forces sequential evaluation.

DCL files are isolated by design → Gradle can configure all projects **in parallel**.

> Speaker Notes: [PACING: Let the table land for 3 seconds. The numbers speak for themselves.] The configuration time problem. Focus on that bottom row. 2,500 modules. 4.5 minutes of configuration time today, just to figure out what to build. With parallel configuration enabled by DCL: 8 seconds. That's a 34x improvement. Now, important caveat, I have it in the footnote: these are theoretical maximums. Real-world numbers will be lower. But even a 5-10x improvement at this scale changes how your team works. [TRANSITION: But it's worth being clear about what DCL does NOT speed up.]

---

# Performance: What DCL Speeds Up (and Doesn't)

<div class="pipeline-bar">
<div class="pipeline-segment unchanged" style="flex:1;">Startup</div>
<div class="pipeline-segment unchanged" style="flex:1;">Settings</div>
<div class="pipeline-segment fast" style="flex:3; flex-direction:column;">⚡ Configuration<br><span style="font-size:0.75em; font-weight:400; letter-spacing:0.05em;">PARALLEL</span></div>
<div class="pipeline-segment unchanged" style="flex:1.2;">Task<br>Graph</div>
<div class="pipeline-segment unchanged" style="flex:3; flex-direction:column;">Execution<br><span style="font-size:0.75em; opacity:0.6; font-weight:400;">(already parallel)</span></div>
</div>

<div style="text-align:center; margin-top:30px;">
<span style="color:rgba(255,255,255,0.4); font-size:0.7em;">░░░ Unchanged</span>&nbsp;&nbsp;&nbsp;
<span style="color:#3DDC84; font-size:0.7em;">███ DCL accelerates this</span>
</div>

DCL only affects the **configuration phase**. Compilation, testing, APK packaging - all unchanged.

> Speaker Notes: [PACING: The green pulsing segment draws the eye immediately. Give 2 seconds, then explain.] Important to set expectations here. Look at the green segment - that's the ONLY part DCL speeds up. Configuration. The phase where Gradle reads your build files and figures out what to do. It does NOT make your Kotlin compiler faster, your tests run quicker, or your APK build smaller. For small projects where config is 2 seconds, you won't notice. For enterprise with 2500 modules where config is 4+ minutes, that's a significant improvement. The execution phase is already parallelized by Gradle - DCL brings that same parallelism to configuration.

---

# Android Application in DCL

```kotlin
// build.gradle.dcl
androidApplication {
    versionCode = 8
    versionName = "0.1.2"
    applicationId = "org.gradle.experimental.android.application"
    namespace = "org.gradle.experimental.android.application"

    dependencies {
        implementation("com.google.guava:guava:32.1.3-jre")
        implementation(project(":android-util"))
    }

    buildTypes {
        release {
            minify {
                enabled = true
            }
        }
        debug {
            applicationIdSuffix = ".debug"
        }
    }
}
```


> Speaker Notes: This is a real example from Gradle's declarative gradle repository. Notice, no plugins block, no android block wrapping everything. The top level block is the Project Type: androidApplication. Everything inside is pure configuration. [TRANSITION: That's an app module - libraries are even leaner.]

---

# Android Library in DCL

```kotlin
// build.gradle.dcl
androidLibrary {
    namespace = "org.gradle.experimental.android.library"
    dependencies {
        api("com.google.guava:guava:32.1.3-jre")
        implementation(project(":android-util"))
    }
    kotlinSerialization {
        version = "1.6.3"
        jsonEnabled = true
    }
    testing {
        dependencies {
            implementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
        }
    }
}
```

> Speaker Notes: Here's a library module. Notice kotlinSerialization is a nested feature - it's a "project feature" that adds capabilities. No plugin ID application, no figuring out the right plugin to apply. Just declare what you need. [TRANSITION: And the smallest modules collapse to almost nothing.]

---

# Minimal Module: 2 Lines

<div style="font-size:1.3em;">

```kotlin
// android-util/build.gradle.dcl
androidLibrary {
    namespace = "org.gradle.experimental.android.library.util"
}
```

</div>

<div style="text-align:center; margin-top:20px;">
<div style="background:#1e1e2e; border:1px solid rgba(61,220,132,0.3); border-radius:12px; padding:20px 30px; display:inline-block;">
<p style="font-size:1.1em; margin:0;">Typical <code>.gradle.kts</code>: <strong style="color:#f5a623;">35-80 lines</strong> → DCL with defaults: <strong style="color:#3DDC84;">2 lines</strong> - <span style="color:#00c9db;">95% reduction</span></p>
</div>
</div>

> Speaker Notes: [PACING: Let the 2-line code block and the callout box land for 3 seconds.] This is the dream. A utility module that inherits all its defaults from settings. Two lines. No boilerplate. Compare this to the 40+ lines we typically see in a build.gradle.kts for a simple library module. Now you might be thinking "wait, convention plugins already solve this." You're right, partially. Convention plugins centralize logic, but the consuming build file still has to apply them, still uses full Kotlin, and still allows arbitrary code alongside. DCL makes the consuming file pure data. The convention plugin becomes a Project Type, and the consuming file becomes two lines of configuration that tools can statically analyze. [TRANSITION: Let me show you what a Project Type actually is.]

---

# Project Types Explained

<div style="display:flex; gap:16px; margin-bottom:8px;">
<div style="flex:1; background:rgba(245,166,35,0.08); border:1px solid rgba(245,166,35,0.4); border-radius:8px; padding:10px 14px;">
<p style="margin:0 0 2px 0; font-weight:600; color:#f5a623; font-size:0.9em;">Convention Plugin</p>
<p style="margin:0; font-size:0.72em;">Kotlin code that applies plugins & sets defaults. You write these today in <code>buildSrc</code> or an included build.</p>
</div>
<div style="display:flex; align-items:center; padding:0 4px; font-size:1.4em; color:#00c9db;">→</div>
<div style="flex:1; background:rgba(61,220,132,0.08); border:1px solid rgba(61,220,132,0.4); border-radius:8px; padding:10px 14px;">
<p style="margin:0 0 2px 0; font-weight:600; color:#3DDC84; font-size:0.9em;">Project Type</p>
<p style="margin:0; font-size:0.72em;">A convention plugin that <em>also</em> exposes a typed, declarative API. Consumers write <code>.gradle.dcl</code> instead of Kotlin.</p>
</div>
</div>

<p style="text-align:center; font-size:0.65em; color:#888; margin:0 0 6px 0;">Convention plugins don't go away. They become the <strong style="color:#3DDC84;">implementation</strong> inside a Project Type. One per project. <strong style="color:#00c9db;">Project Features</strong> nest inside.</p>

```kotlin
// The definition = the typed surface devs configure in .gradle.dcl
@Restricted
interface AndroidLibraryDefinition : Definition<AndroidLibraryModel> {
    val namespace: Property<String>
    val dependencies: DependencyCollector
}

// A Plugin<Project> binds that definition to its build logic:
@BindsProjectType(AndroidLibraryBinding::class)
abstract class AndroidLibraryPlugin : Plugin<Project>
// The binding names the "androidLibrary" block and points to an
// apply action that runs the old convention-plugin logic (AGP, SDKs).
```

> Speaker Notes: [PACING: Give 3-4 seconds for the two definition boxes, then walk through the code.] This is how I like to think about Project Types and convention plugins. A convention plugin is imperative Kotlin that runs at configuration time - applies plugins, sets properties, wires tasks. You write these today. A Project Type wraps that convention plugin and also exposes a typed, declarative API on top. The convention plugin doesn't disappear - it becomes the implementation layer. What changes? Consumers write pure declarative config instead of Kotlin; there's exactly one Project Type per project with no overlap; and tools can safely modify a .dcl file because the schema is known. One question you might have is - can you compose Project Types; aka wrap AGP's androidLibrary inside your own? Each project gets exactly one Project Type, so you don't nest them. But your implementation layer is still full Kotlin and can call AGP directly. For modular add-ons, that's what Project Features are for. [TRANSITION: Let's look at what the settings file looks like in DCL.]

---

# `settings.gradle.dcl`: Defaults and Ecosystem Plugins

<div class="horror-scroll-container" style="height:420px;">

```kotlin
// settings.gradle.dcl
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

// ⬇ THE KEY BLOCK: replaces allprojects/subprojects
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

</div>

> Speaker Notes: The settings file is where build engineers set up the ecosystem. Three things to notice: First, dependencyResolutionManagement centralizes your repositories - no more allprojects { repositories { } } blocks scattered across modules. Second, the defaults block at the bottom - set compileSdk once, apply everywhere. No more convention plugins that fight each other. Every androidLibrary and androidApplication in your project inherits these values unless explicitly overridden.

---

# Before vs. After Comparison

| Aspect | `.gradle.kts` (Today) | `.gradle.dcl` (Future) |
|---|---|---|
| File extension | `.gradle.kts` | `.gradle.dcl` |
| Language | Full Kotlin | Restricted Kotlin subset |
| Arbitrary code | ✅ Allowed | ❌ Not allowed |
| Plugin application | `plugins { id("...") }` | Via Project Type |
| Cross-project access | Possible (dangerous) | Blocked by design |
| IDE analysis | Requires full evaluation | Static analysis possible |
| Tooling modification | Fragile | Safe mutations API |

> Speaker Notes: [PACING: Give 4 seconds for a 7-row table.] This is the whole shift summarized in one slide. I want to anchor on the "Arbitrary code" row. Today, allowed; tomorrow, not allowed. That sounds limiting, but the flexibility doesn't disappear. It moves to where it belongs: the plugin implementation layer. Developers get a clean, safe config surface. Build engineers keep full Kotlin power in included builds. The last two rows are what tooling authors care about. DCL files can be analyzed statically and modified safely via a mutations API. [TRANSITION: Let me show you what this actually looks like from the IDE's perspective.]

---

# What Your IDE Sees

<div style="display:flex; gap:20px; align-items:stretch;">
<div style="flex:1; min-width:0; display:flex; flex-direction:column;">

<p style="text-align:center; font-weight:600; color:#ef4444; margin-bottom:8px;"><code>.gradle.kts</code> : Full eval required</p>

<div class="ide-model-box ide-model-kts" style="flex:1;">
<span style="color:#808080;">// IDE must execute to understand:</span><br>
<span style="color:#ef4444;">❓</span> plugins { ... } <span style="color:#808080;">  &larr; which ones?</span><br>
<span style="color:#ef4444;">❓</span> android { ... } <span style="color:#808080;">  &larr; what schema?</span><br>
<span style="color:#ef4444;">❓</span> if (isCI) { ... } <span style="color:#808080;">&larr; can't predict</span><br>
<span style="color:#ef4444;">❓</span> afterEvaluate { } <span style="color:#808080;">&larr; side effects?</span><br>
<span style="color:#ef4444;">❓</span> project(":other") <span style="color:#808080;">&larr; needs other project</span><br><br>
<span style="color:#f5a623;">⚠️ Result: full Kotlin evaluation,</span><br>
<span style="color:#f5a623;">&nbsp;&nbsp; sequential, slow, fragile</span>
</div>

</div>
<div style="flex:1; min-width:0; display:flex; flex-direction:column;">

<p style="text-align:center; font-weight:600; color:#3DDC84; margin-bottom:8px;"><code>.gradle.dcl</code> : Static analysis only</p>

<div class="ide-model-box ide-model-dcl" style="flex:1;">
<span style="color:#808080;">// IDE knows the schema upfront:</span><br>
<span style="color:#3DDC84;">✅</span> androidLibrary <span style="color:#808080;">    &larr; typed block</span><br>
<span style="color:#3DDC84;">✅</span> &nbsp;&nbsp;namespace = "..." <span style="color:#808080;">&larr; Property&lt;String&gt;</span><br>
<span style="color:#3DDC84;">✅</span> &nbsp;&nbsp;dependencies { } <span style="color:#808080;"> &larr; known structure</span><br>
<span style="color:#3DDC84;">✅</span> &nbsp;&nbsp;testing { } <span style="color:#808080;">      &larr; opt-in feature</span><br>
<span style="color:#3DDC84;">✅</span> &nbsp;&nbsp;<span style="color:#808080;">                  &larr; nothing else possible</span><br><br>
<span style="color:#3DDC84;">✨ Result: instant parsing, complete</span><br>
<span style="color:#3DDC84;">&nbsp;&nbsp; type info, safe refactoring</span>
</div>

</div>
</div>

> Speaker Notes: [PACING: 3 seconds for the visual contrast to land.] What your IDE sees - this is why IDE support matters. On the Left side: to understand a KTS file, your IDE must fully evaluate it. Execute the Kotlin, resolve plugins dynamically, figure out what schema the android block even has. It can't predict conditional logic or afterEvaluate side effects. It needs to evaluate other projects if there's cross-project access. That's why Gradle sync is slow. On the Right side: DCL. The schema is known statically from the Project Type definition. The IDE never needs to execute anything. It knows every valid property, every valid nested block, instantly. That's why completion is perfect and sync is fast. [TRANSITION: And this isn't hypothetical. IDE support exists today.]

---

# IDE Support: It's Real

| IDE | Status | Features |
|---|---|---|
| **Android Studio** | ✅ Available | Syntax highlighting, completion, semantic analysis |
| **IntelliJ IDEA** | ✅ Available | Same as Android Studio |
| **Visual Studio Code** | ✅ Extension available | Highlighting, completion, mutations |
| **Eclipse IDE** | ✅ Plugin available | Highlighting, completion |

- Code completion shows **only valid properties** in current scope
- No noise - DCL strictness means precise suggestions
- Mutations (refactorings) available directly in editor


> Speaker Notes: [PACING: 3 seconds for the table, then reassure.] I want to emphasize: this is NOT vaporware. Four IDEs have support today. Android Studio Nightly, IntelliJ EAP, VS Code, and Eclipse. The upside of DCL's strictness is that code completion is perfect. It can only suggest properties that are actually valid in the current scope. No more autocomplete noise. No more guessing. Have you ever typed inside an android block and gotten 200 suggestions you'd never use? That goes away. [TRANSITION: Let me show you how to enable it.]

---

# Enabling DCL in Android Studio

1. Install **Android Studio Nightly**
2. Enable internal mode: `idea.properties` → `idea.is.internal=true`
3. Open **Tools → Internal Actions → Registry**
4. Enable flags:
    - `gradle.declarative.studio.support`
    - `gradle.declarative.ide.support`
5. Restart

> Speaker Notes: If you want to try this today, here's how. It's a nightly build requirement for now, but the flags are there and it works with the sample projects. [PACING: Pause 5-6 seconds here.] I'll leave this up for a moment, the slides are available online if anyone needs to reference these flags later. [TRANSITION: Now let's talk about Isolated Projects, a separate but complementary feature.]

---

# The Gradle Client (Two-Way Tooling)

<!-- .slide: data-visibility="hidden" -->

- **Built by Gradle Inc.** - open-source at [gradle/gradle-client](https://github.com/gradle/gradle-client)
- Standalone Compose Desktop app for exploring DCL models
- **Two-way highlighting**: DCL files ↔ configured model
- **Mutations API**: programmatic refactoring of DCL files
- Works even on DCL files with errors

```kotlin
// Mutations API example: adding a dependency programmatically
val mutation = AddDependency(
    projectPath = ":core:network",
    configuration = "implementation",
    dependency = "com.squareup.okhttp3:okhttp:4.12.0"
)
// Applies directly to the .gradle.dcl file on disk
mutationEngine.apply(mutation)
```

> Speaker Notes: [PACING: 2 seconds, then clarify provenance.] This is built by Gradle themselves. It's an open-source Compose Desktop application at github.com/gradle/gradle-client. It's a reference implementation showing what's possible when build files are statically analyzable. The Mutations API is the key concept here: because DCL files are structured data, tooling can safely modify them. Look at the code example. Adding a dependency is a typed operation, not a regex find-and-replace in a Kotlin file. This same API powers IDE refactorings and could power CI bots that auto-update dependencies. The mutations work even on DCL files that have syntax errors, because the parser understands the structure independently of validity. Note: this is currently a standalone app, not baked into Gradle itself, but the mutations API will likely be integrated into IDE plugins and Gradle's tooling API over time.

---

# What About Isolated Projects?

- **Separate Gradle feature** (not DCL, but complementary)
- Enforces that projects **cannot access each other's configuration**
- Enables **parallel project configuration**
- Violations reported at build time

```
// Enable in gradle.properties
org.gradle.isolated-projects=true
```


> Speaker Notes: What about Isolated Projects? It's the enforcement mechanism that prevents cross-project access. DCL builds are inherently isolated because you can't write imperative code that reaches into other projects. [AUDIENCE] Has anyone here actually tried running --isolated-projects on their build? If you have, you know what's coming next.

---

# Isolated Projects: What It Blocks

<div style="font-size:1.2em;">

```kotlin
// ❌ VIOLATION: Cross-project access
allprojects {
    repositories { mavenCentral() }
}

// ❌ VIOLATION: Accessing another project's config
project(":app").android.compileSdk

// ❌ VIOLATION: subprojects block
subprojects {
    apply(plugin = "kotlin-android")
}
```

</div>

**These patterns are incompatible with both Isolated Projects and DCL.**

> Speaker Notes: These are the bread and butter of many enterprise builds. allprojects, subprojects, and direct project access are all violations. If your build uses these patterns - and most do - you have migration work ahead. One subtler behavior change to flag: under Isolated Projects a child project no longer implicitly inherits properties or methods from its parent project - the docs call this out specifically, and it bites enterprise builds that lean on inherited ext properties.

---

# Isolated Projects: Violation Reporting

- Run with `--isolated-projects` flag or property
- Gradle reports violations as **build warnings/errors**
- Each violation includes:
  - The **offending project**
  - The **accessed project**
  - The **type of access** (cross-project, project-to-build, etc.)
- Can temporarily bypass with `org.gradle.isolated-projects.dangerously-ignore-problems=true` (use sparingly)

> Speaker Notes: Gradle gives you actionable diagnostics. You can run your existing build with isolated projects enabled and get a report of everything that needs to change. It's like lint for your build configuration. The docs also call out a dedicated diagnostics mode - org.gradle.isolated-projects.diagnostics - that surfaces violations without failing the build, which is the safest way to survey a large build before you commit to fixing anything.

---

# How Gradle Reports Violations

<div style="font-size:1.2em;">

```
$ ./gradlew :app:assembleDebug --isolated-projects

> PROBLEM: Build file 'build.gradle.kts': line 12
  Project ':app' cannot access 'Project.name' 
  functionality on project ':core:network'
  
  Reason: Isolated projects prevents cross-project 
  configuration access.
  
  Possible solution: Use a project dependency instead 
  of direct project access.

BUILD SUCCESSFUL with 3 isolation violations
```

</div>

> Speaker Notes: This is what the output looks like. Gradle tells you exactly which file, which line, what the violation is, and often suggests a fix. You can run this on your existing build today.

---

# Why Most Android Projects Are Blocked Today

1. **`allprojects {}` / `subprojects {}`** - Used in nearly every multi-module project
2. **Convention plugins accessing other projects** - Common pattern for shared config
3. **AGP cross-project dependencies in configuration** - Variant-aware resolution
4. **buildSrc with project-level logic** - Anything that evaluates at config time
5. **Third-party plugins using `project.rootProject`** - Very common

> Speaker Notes: Most non-trivial Android projects use at least one of these patterns. The migration isn't going to happen overnight, and that's okay. Gradle is providing a path forward. [AUDIENCE] Raise your hand if your project uses allprojects or subprojects.

---

# Common Plugin Compatibility Challenges

| Plugin | Status | Notes |
|---|---|---|
| AGP (Android Gradle Plugin) | 🟡 In progress | Core DCL support via experimental ecosystem plugin |
| KGP (Kotlin Gradle Plugin) | 🟡 In progress | Kotlin support available in prototypes |
| KMP (Kotlin Multiplatform) | 🟡 Prototype | `kotlinMultiplatform` Project Type exists in unified-prototype |
| Hilt / Dagger | ❓ Unknown | Symbol processing plugins need adaptation |
| Navigation SafeArgs | ❓ Unknown | Code generation at config time is problematic |
| Room / KSP | 🟡 Partial | KSP itself needs to be DCL-aware |
| Firebase / Crashlytics | ❓ Unknown | Config-time file manipulation |

**Takeaway**: Any plugin that runs imperative logic during configuration is potentially incompatible.

> Speaker Notes: [PACING: This is a 7-row table. Give 4-5 seconds, then anchor.] These are some of the Plugin compatibility challenges. Look at the status column. Lots of yellow and question marks. AGP and KGP are actively being worked on by Gradle and Google. But Hilt? SafeArgs? Firebase? Those are question marks. And the key takeaway is at the bottom in bold: any plugin that runs imperative logic during configuration is potentially incompatible. If your build uses Hilt or Room, you'll need to wait for those ecosystems to adapt. [TRANSITION: If you're a plugin author yourself, here's what to do.]

---

# If You're a Gradle Plugin Author

**Things to watch for:**

<div style="padding-left:30px; font-size:0.85em;">

- <span style="color:#ef4444;">Don't</span> use `project.allprojects` or `project.subprojects` in your plugin
- <span style="color:#ef4444;">Don't</span> access `project.rootProject` configuration at config time
- <span style="color:#ef4444;">Don't</span> register tasks in other projects from your plugin
- <span style="color:#3DDC84;">Do</span> use lazy configuration APIs: `Property<T>`, `Provider<T>`, `TaskProvider`
- <span style="color:#3DDC84;">Do</span> consider creating a Project Type if your plugin defines a project archetype
- <span style="color:#3DDC84;">Do</span> test with `--isolated-projects` to validate compatibility

</div>

```kotlin
// ❌ Eager, cross-project
project.rootProject.allprojects { 
    apply(plugin = "my-plugin") 
}

// ✅ Scoped to current project
project.plugins.apply("my-plugin")
```

> Speaker Notes: If you maintain a Gradle plugin - internal or open source - start testing with isolated projects today. Any violation your plugin causes will block your users from adopting DCL. Again, stop using subprojects or allprojects, stop using rootProject at configuration time, and don't reach into other projects from your Plugin.

---

# Plugins & Project Features in DCL

<div style="display:flex; gap:20px; align-items:flex-start;">
<div style="flex:1; min-width:0;">
<div style="font-size:0.8em;">

**Today** (imperative):
</div>
<div style="font-size:1.1em;">

```kotlin
// build.gradle.kts
plugins {
    id("com.android.library")
    id("org.jetbrains.kotlin.android")
    id("kotlinx-serialization")
}
```

</div>
</div>
<div>
<div style="flex:1; min-width:0; font-size:0.8em;">

**DCL** (implicit via Project Type + Features):
</div>
<div style="font-size:1.0em;">

```kotlin
// build.gradle.dcl
androidLibrary {
    namespace = "com.example.mylib"

    // Project Feature: opt-in capability
    kotlinSerialization {
        version = "1.6.3"
        jsonEnabled = true
    }

    testing {
        dependencies {
            implementation("junit:junit:4.13.2")
        }
    }
}
```

</div>
</div>

</div>
</div>

- No `plugins {}` block in DCL - Project Type handles plugin application
- **Project Features** add opt-in capabilities (registered in settings)
- If not referenced, zero cost

> Speaker Notes: In DCL, you don't apply plugins in build files. The ecosystem plugin in settings registers Project Types, and Project Features add capabilities. Plugin application is handled by the framework, not the developer. Project Features are the extension mechanism - they're like plugins but scoped and typed. kotlinSerialization, testing, hilt - each is a feature you opt into per project. You only pay for what you use, and the IDE knows exactly what's available in the current scope. [TRANSITION: So how do you get from your KTS build to this?]

---

# Migration Journey Overview

<div style="display:flex; align-items:center; justify-content:center; gap:6px; flex-wrap:wrap; margin:20px 0;">
<span style="background:#8B0000; color:#fff; padding:8px 14px; border-radius:6px; font-size:0.7em; font-weight:bold;">Your Build Today</span><span style="color:#f5a623; font-size:1.2em;">→</span>
<span style="background:#f5a623; color:#000; padding:8px 14px; border-radius:6px; font-size:0.7em; font-weight:bold;">Run --isolated-projects</span><span style="color:#f5a623; font-size:1.2em;">→</span>
<span style="background:#f5a623; color:#000; padding:8px 14px; border-radius:6px; font-size:0.7em; font-weight:bold;">Fix Violations</span><span style="color:#00c9db; font-size:1.2em;">→</span>
<span style="background:#00c9db; color:#000; padding:8px 14px; border-radius:6px; font-size:0.7em; font-weight:bold;">Convert Settings</span><span style="color:#00c9db; font-size:1.2em;">→</span>
<span style="background:#00c9db; color:#000; padding:8px 14px; border-radius:6px; font-size:0.7em; font-weight:bold;">Migrate Leaf Modules</span><span style="color:#3DDC84; font-size:1.2em;">→</span>
<span style="background:#3DDC84; color:#000; padding:8px 14px; border-radius:6px; font-size:0.7em; font-weight:bold;">Custom Project Types</span><span style="color:#3DDC84; font-size:1.2em;">→</span>
<span style="background:#3DDC84; color:#000; padding:8px 14px; border-radius:6px; font-size:0.7em; font-weight:bold;">Full DCL Build</span>
</div>

You can stop at any point. Each step improves your build independently.

> Speaker Notes: Migration journey overview. Every step provides standalone value. Fixing isolated projects violations makes your build faster today. Converting settings to DCL centralizes your defaults today. You don't need to go all the way to get benefits. [TRANSITION: So where do you start? It depends on your scale.]

---

# Where to Start: Module Selection Strategy

| Project Size | Strategy | First Target |
|---|---|---|
| **Small** (1-10 modules) | Migrate all at once | Start with settings.gradle.dcl |
| **Medium** (10-50 modules) | Bottom-up, leaf modules first | Utility/core libraries |
| **Large** (50-500 modules) | Batch by team/feature | One team pilots, others follow |
| **Enterprise** (500+ modules) | Automated analysis + phased | AI-assisted triage by readiness category |

## What makes a good first candidate?

- No custom tasks or `afterEvaluate`
- No cross-project references
- Pure library (no app module)
- Minimal plugin applications

> Speaker Notes: For small projects, you can often convert everything in a day. For large projects, you need a strategy. Leaf modules - the ones at the bottom of your dependency graph with no downstream consumers of their build logic - are always the safest first step. They tend to be simple utility libraries with no custom configuration. [TRANSITION: Once you've picked targets, here are the rules of the road.]

---

# Migration Best Practices

1. **Run `--isolated-projects` first** - get your violation baseline
2. **Start with leaf modules** - no downstream build-logic consumers
3. **Keep `.kts` and `.dcl` side-by-side** - Gradle supports mixed builds
4. **Centralize before converting** - move shared config to settings defaults *before* migrating individual modules
5. **Batch by readiness** - group modules by category (✅ Ready first, then ⚠️ Nearly Ready)
6. **Validate incrementally** - CI checks after each batch
7. **Don't migrate app modules first** - they have the most custom logic

```kotlin
// Tip: Check a module's readiness in one command
./gradlew :my-module:dependencies --isolated-projects
// Zero violations? Ready to migrate.
```

> Speaker Notes: The most common mistake is trying to migrate your app module first - it always has the most custom logic, signing configs, build types, flavor dimensions. Start with the boring utility modules. Get confidence. Build muscle memory. Then tackle the complex ones.

---

# Migration Example: Before → After

<div style="display:flex; gap:20px; align-items:flex-start;">
<div style="flex:1; min-width:0; font-size:0.9em;">

```kotlin
// build.gradle.kts (35+ lines)
plugins {
    id("com.android.library")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.example.core.network"
    compileSdk = 35

    defaultConfig {
        minSdk = 26
        testInstrumentationRunner =
            "androidx.test.runner.AndroidJUnitRunner"
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }
}

dependencies {
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
    testImplementation("junit:junit:4.13.2")
}
```

</div>
<div style="flex:1; min-width:0; font-size:1.0em;">

```kotlin
// build.gradle.dcl (14 lines)
androidLibrary {
    namespace = "com.example.core.network"

    dependencies {
        implementation("com.squareup.okhttp3:okhttp:4.12.0")
        implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
    }

    testing {
        dependencies {
            implementation("junit:junit:4.13.2")
        }
    }
}
```

<div style="font-size:0.6em;">

**Where did everything else go?**
- `compileSdk`, `minSdk`, `jdkVersion` → **settings defaults**
- Plugin application → **Project Type**
- `compileOptions`/`kotlinOptions` → **derived from jdkVersion**

</div>

</div>
</div>

> Speaker Notes: [PACING: Give 4 seconds for the side-by-side comparison to land.] Let's take a look at a before and after kts -> dcl migration. On the left side is a typical library module today; 35+ lines, most of which is identical boilerplate across every module. On the right side is the same module in DCL. 14 lines. Where did everything go? compileSdk, minSdk, jdkVersion all move to settings defaults, defined once for the entire project. Plugin application is handled by the Project Type. compileOptions and kotlinOptions are derived automatically from jdkVersion. What's left is only what's unique to this module: its namespace and its dependencies. [TRANSITION: Let's look at a more complex migration involving a convention plugin.]

---

# Migration: Convention Plugin to Project Type

<div style="display:flex; gap:20px; align-items:flex-start;">
<div style="flex:1; min-width:0; font-size:1.0em;">

**Before** (convention plugin):
```kotlin
// android-library-convention.gradle.kts
plugins {
    id("com.android.library")
    id("org.jetbrains.kotlin.android")
}
android {
    compileSdk = 35
    defaultConfig { minSdk = 26 }
}
```

</div>
<div style="flex:1; min-width:0; font-size:0.9em;">

**After** (custom Project Type):
```kotlin
// build-logic/ - Gradle project-features binding API

// 1. The Plugin<Project> is just a marker that carries the binding:
@BindsProjectType(MyAndroidLibraryBinding::class)
abstract class MyAndroidLibraryPlugin : Plugin<Project>

// 2. The binding names the .dcl block + points at an apply action:
class MyAndroidLibraryBinding : ProjectTypeBinding {
    override fun bind(builder: ProjectTypeBindingBuilder) {
        builder.bindProjectType(
            "myAndroidLibrary",
            MyAndroidLibraryApplyAction::class,
        )
    }
}

// 3. Your convention-plugin code moves into the apply action:
abstract class MyAndroidLibraryApplyAction :
    ProjectTypeApplyAction<MyAndroidLibraryDefinition, BuildModel.None> {
    override fun apply(
        context: ProjectFeatureApplicationContext,
        definition: MyAndroidLibraryDefinition,
        buildModel: BuildModel.None,
    ) {
        // the logic from your old convention plugin
    }
}
```

</div>
</div>

> Speaker Notes: [PACING: 3 seconds for the side-by-side.] For enterprises with convention plugins, the migration path is pretty direct. On the left is your convention plugin today. On the right is the same logic formalized as a Project Type using Gradle's project-features binding API - three pieces. First, the Plugin<Project> is now just a marker: it carries a @BindsProjectType annotation and has no logic of its own. Second, a small binding class names the 'myAndroidLibrary' DCL block and points at an apply action. Third, the imperative code from your convention plugin, applying AGP and setting compileSdk, relocates into the ApplyAction's apply method. BuildModel.None just means this Project Type carries no separate build-model object; the typed definition interface from the earlier slide is enough. One gotcha worth naming: reaching the Android extension needs a Project, which isn't a default apply-action service, so you inject Project and mark the binding withUnsafeApplyAction() when you add that logic. If you already have convention plugins, you're 80% of the way there - the binding is just ceremony around logic you already wrote.

---

# Migration: Convention Plugin to Project Type

<p style="font-size:0.72em; margin:0 0 6px 0;">Inside the apply action, <code>context</code> only offers <code>getBuildModel()</code>. The real power comes from <strong style="color:#3DDC84;">injected, isolation safe services</strong>.</p>

</p>

<div style="display:flex; gap:16px; align-items:flex-start;">
<div style="flex:1; min-width:0;">
<p style="font-size:0.78em; margin:0 0 4px 0;"><strong style="color:#3DDC84;">Safe path</strong>, no <code>Project</code>:</p>
<div style="font-size:0.9em;">

```kotlin
abstract class MyAndroidLibraryApplyAction : /* ... */ {

    // Injected the same way a task gets services:
    @get:Inject abstract val tasks: TaskRegistrar
    @get:Inject abstract val configurations: ConfigurationRegistrar
    @get:Inject abstract val layout: ProjectFeatureLayout

    override fun apply(/* context, definition, buildModel */) {
        configurations.dependencyScope("moduleApi")

        val out = layout.contextBuildDirectory.map { it.dir("info") }
        tasks.register("generateInfo", GenerateInfoTask::class.java) { t ->
            t.namespace.set(definition.namespace)   // lazy Provider wiring
            t.outputDirectory.set(out)
        }
    }
}
```

</div>
</div>
<div style="flex:1; min-width:0;">
<p style="font-size:0.78em; margin:0 0 4px 0;"><strong style="color:#f5a623;">Escape hatch</strong>, when you need <code>Project</code>:</p>
<div style="font-size:1.0em;">

```kotlin
// 1. The binding must opt in:
builder.bindProjectType(
    "myAndroidLibrary",
    MyAndroidLibraryApplyAction::class,
).withUnsafeApplyAction()

// 2. Only now can the action inject the real Project:
abstract class MyAndroidLibraryApplyAction : /* ... */ {

    @get:Inject abstract val project: Project

    override fun apply(/* context, definition, buildModel */) {
        project.plugins.apply("com.android.library")
        project.extensions.configure<LibraryExtension>("android") {
            namespace = definition.namespace.get()
        }
    }
}
```

</div>
</div>
</div>

> Speaker Notes: [PACING: Dense code slide - give 6-8 seconds, then walk each side.] So what actually goes inside that apply action since the context object you're handed has exactly one method, getBuildModel. Where does the configuration power come from? Injection. Gradle instantiates your apply action as a managed type, the same way it does a task, so you declare services with @Inject. On the left is the safe path: TaskRegistrar to register tasks, ConfigurationRegistrar for role-locked configurations, ProjectFeatureLayout for output directories. Everything is lazy - definition.namespace is a Provider wired straight into the task, nothing is resolved at configuration time, and nothing reaches outside this project. That's what makes parallel configuration safe. On the right is the escape hatch. Applying AGP needs the real Project and the safe API can't express that yet. So you call withUnsafeApplyAction() on the binding and only then are you allowed to inject Project. [TRANSITION: With the mechanics covered, let's talk about the pitfalls you'll hit.]

---

# Common Pitfalls of DCL Migration

1. **Can't use `if`/`when` in DCL** - no conditional logic in build files
2. **Can't call functions** - no `file()`, `rootProject.file()`, etc.
3. **No version catalogs in `.dcl` yet** - convert `libs.x` to GAV coords (`org:lib:1.4`)
4. **One Project Type per project** - can't mix `androidLibrary` + `javaLibrary`
5. **Can't use variables** - everything is a direct assignment
6. **Feature ordering doesn't matter** - but may trip up imperative thinkers
7. **No `ext` properties in DCL files** - use typed Project Type properties instead

> Speaker Notes: These are the gotchas. If you have conditional configuration - like different behavior per build type that's computed - that logic moves into the Project Type implementation. The DCL file stays pure. The version catalog one surprises people: today, fully declarative .dcl files can't reference libs.x accessors, so you convert them back to plain GAV coordinates like org:lib:1.4. Version catalogs still work in the .kts files during the intermediate migration, and Gradle has said catalog support in DCL is coming in a future EAP - but as of today it's a real footgun. On the ext properties point: this only applies to .dcl files. Your Project Type implementation code (in the included build) can still use extra properties, full Kotlin, whatever it needs. If you relied on ext properties to pass data between build scripts, the DCL answer is to define typed properties on the Project Type instead.

---

# Common Pitfalls of Isolated Projects

1. **`allprojects` in convention plugins** - refactor to per-project
2. **Shared `buildscript` classpath** - each project configures independently
3. **Version catalog access** - some older patterns violate isolation
4. **Test fixtures across projects** - careful dependency declaration
5. **Cross-project task wiring** - use artifact dependencies

<div style="font-size:1.2em;">

```kotlin
// ❌ allprojects {  repositories { mavenCentral() }  }

// ✅ settings.gradle.kts
dependencyResolutionManagement {
    repositories { mavenCentral() }
}
```

</div>

> Speaker Notes: The most common Isolated Projects violation is repository declaration via allprojects. The fix is simple: move to dependencyResolutionManagement in settings. But not all fixes are this straightforward. Shared buildscript classpath means each project must configure independently. Some older version catalog patterns violate isolation. Test fixtures across projects require careful dependency declaration. Custom task wiring between projects must use proper artifact dependencies instead of direct project access.

---

# Your Convention Plugin, Evolved

| Your Convention Plugin Today | In DCL |
|---|---|
| applies AGP/KGP | → implicit (via Project Type) |
| sets compileSdk, minSdk, targetSdk | → settings defaults |
| configures signing | → project feature |
| wires test dependencies | → project feature |
| configures lint rules | → project feature |
| sets Java/Kotlin version | → settings defaults |
| applies DexGuard/R8 | → project feature |

If your build already consolidates logic into a centralized plugin...
**you're already thinking like DCL.**

The migration is: formalize what you have into the Project Type API.

> Speaker Notes: If your enterprise has invested in a build-configuration plugin that centralizes decisions - congratulations, you've been building toward DCL without knowing it. The conceptual model is the same. The API formalization is what's new.

---

# Where Does Your Build Logic Go?

<div class="mermaid">
graph TD
    A[Today: root build.gradle.kts] -->|moves to| B[Included Build]
    C[Today: buildSrc] -->|becomes| B
    D[Today: convention plugins] -->|formalize as| E[Project Type Plugin]
    E -->|registered in| F[settings.gradle.dcl]
    B -->|contains| E
    F -->|applies to| G[Project .gradle.dcl files]
</div>

| What you have today | Where it goes in DCL |
|---|---|
| `buildSrc/` | Included build (`build-logic/`) |
| Root `build.gradle.kts` tasks | Included build or lifecycle plugin |
| Convention plugins | Project Type definitions |
| `allprojects {}` / `subprojects {}` | `defaults {}` block in settings |
| Custom task registration | Included build plugin |

> Speaker Notes: This is the question everyone asks after seeing DCL examples - "ok but where does all my custom stuff go?" The answer is included builds. Your buildSrc becomes a proper included build, typically named build-logic. Your convention plugins become Project Type definitions inside that included build. Your root build.gradle.kts tasks move into plugins registered from the included build. Build logic doesn't disappear. It just moves to the right layer. Developers see clean DCL files. Build engineers maintain the included build with full Kotlin power.

---

# CI/CD: Nothing Changes

```bash
# Before DCL
./gradlew :app:assembleDebug
./gradlew test
./gradlew lint

# After DCL - IDENTICAL commands
./gradlew :app:assembleDebug
./gradlew test
./gradlew lint
```

**Your CI pipeline doesn't know or care about DCL.**

- Task names: unchanged
- Gradle invocation: unchanged
- `gradle.properties` flags: unchanged
- Build cache: unchanged
- Artifact output paths: unchanged

> Speaker Notes: If you manage CI infrastructure, your Jenkins, GitHub Actions, or Bitrise pipelines don't need a single change. DCL only affects how projects are configured, not how tasks are invoked or executed. The build outputs are identical. The cache keys are identical. The only thing that changes is the file extension of your build scripts. If you have scripts that parse build.gradle.kts files directly, those need updating - but that's rare.

---

# DCL Status and Compatibility Matrix

| Component | Status |
|---|---|
| DCL file format (`.gradle.dcl`) | ✅ Experimental |
| Android ecosystem plugin | ✅ Prototype available |
| Kotlin ecosystem plugin | ✅ Prototype available |
| Java ecosystem plugin | ✅ Prototype available |
| `gradle init` with DCL | ✅ Available (since Gradle 8.12) |
| Android Studio support | ✅ Nightly builds |
| IntelliJ IDEA support | ✅ Available |
| VS Code extension | ✅ Available |
| <span style="background:rgba(239,68,68,0.15); padding:2px 6px; border-radius:4px; border-left:3px solid #ef4444;">Production readiness</span> | <span style="background:rgba(239,68,68,0.15); padding:2px 6px; border-radius:4px;">❌ <strong style="color:#ef4444;">Not yet</strong></span> |
| <span style="background:rgba(239,68,68,0.15); padding:2px 6px; border-radius:4px; border-left:3px solid #ef4444;">Plugin author adoption</span> | <span style="background:rgba(239,68,68,0.15); padding:2px 6px; border-radius:4px;">❌ <strong style="color:#ef4444;">Not yet</strong></span> |

<div style="margin-top:8px; padding:8px 16px; border-left:3px solid #f5a623; background:rgba(245,166,35,0.06); border-radius:0 6px 6px 0;">
<p style="font-size:0.72em; font-style:italic; margin:0;">"Declarative Gradle is <strong>ready</strong> for trying out our provided sample projects. Declarative Gradle is <strong style='color:#ef4444;'>not ready</strong> for adoption by plugin authors, build engineers or software engineers."<span style="color:#f5a623;"> - Official docs</span></p>
</div>

> Speaker Notes: [PACING: 4 seconds for a 10-row table. Anchor on the last two rows.] This is a DCL status and compat matrix. Look at the bottom two rows. Red X's. Production readiness: not yet. Plugin author adoption: not yet. Everything above is green and working, but those last two rows are where we really stand. Gradle's official docs state: "Declarative Gradle is ready for trying out our provided sample projects. Declarative Gradle is not ready for adoption by plugin authors, build engineers or software engineers." I'm showing you this because I want you to be excited about the direction without deploying it Monday morning. This is for experimentation and preparation, not production migration today. One nuance on that timestamp: the standalone EAP dates to April 2025, but a lot of the recent movement - like the Project Types rename - has been shipping in core Gradle releases. Watch the mainline distribution, not just the EAP repo, to see where it's really going.

---

# Roadmap (from Gradle)

<div style="position:relative; padding-left:40px; margin-top:6px; font-size:0.82em;">
<div style="position:absolute; left:14px; top:6px; bottom:6px; width:3px; background:linear-gradient(to bottom, #3DDC84, #00c9db, #f5a623); border-radius:2px;"></div>
<div style="position:relative; margin-bottom:9px;"><span style="position:absolute; left:-33px; top:4px; width:12px; height:12px; background:#3DDC84; border-radius:50%; border:2px solid #fff;"></span><strong style="color:#3DDC84;">EAP1</strong> <span style="color:#888;">(July 2024)</span><br><span style="font-size:0.82em;">Initial samples, IDE prototypes</span></div>
<div style="position:relative; margin-bottom:9px;"><span style="position:absolute; left:-33px; top:4px; width:12px; height:12px; background:#3DDC84; border-radius:50%; border:2px solid #fff;"></span><strong style="color:#3DDC84;">EAP2</strong> <span style="color:#888;">(November 2024)</span><br><span style="font-size:0.82em;">Enhanced IDE support, more samples</span></div>
<div style="position:relative; margin-bottom:9px;"><span style="position:absolute; left:-33px; top:4px; width:12px; height:12px; background:#3DDC84; border-radius:50%; border:2px solid #fff;"></span><strong style="color:#3DDC84;">EAP3</strong> <span style="color:#888;">(April 2025)</span> <span style="color:#888; font-size:0.8em;">latest standalone EAP</span><br><span style="font-size:0.82em;">Eclipse support, refinements, Now in Android sample</span></div>
<div style="position:relative; margin-bottom:9px;"><span style="position:absolute; left:-33px; top:4px; width:12px; height:12px; background:#00c9db; border-radius:50%; border:2px solid #fff; box-shadow:0 0 8px rgba(0,201,219,0.6);"></span><strong style="color:#00c9db;">Landing in core Gradle</strong> <span style="color:#888;">(2026)</span> <span style="color:#f5a623; font-size:0.8em;">← We are here</span><br><span style="font-size:0.82em;">DCL groundwork shipping in mainline Gradle</span></div>
<div style="position:relative; margin-bottom:9px;"><span style="position:absolute; left:-33px; top:4px; width:12px; height:12px; background:#555; border-radius:50%; border:2px solid #888;"></span><strong style="color:#ccc;">Next standalone EAP</strong> <span style="color:#888;">(TBD)</span><br><span style="font-size:0.82em;">Samples catch up to core</span></div>
<div style="position:relative; margin-bottom:9px;"><span style="position:absolute; left:-33px; top:4px; width:12px; height:12px; background:#555; border-radius:50%; border:2px solid #888;"></span><strong style="color:#ccc;">Incubating</strong> <span style="color:#888;">(Future)</span><br><span style="font-size:0.82em;">Project Types and DCL stabilizing</span></div>
<div style="position:relative; margin-bottom:0;"><span style="position:absolute; left:-33px; top:4px; width:12px; height:12px; background:#f5a623; border-radius:50%; border:2px solid #fff;"></span><strong style="color:#f5a623;">Stable</strong> <span style="color:#888;">(Future)</span><br><span style="font-size:0.82em;">Production-ready Project Types and DCL</span></div>
</div>


> Speaker Notes: The roadmap shows steady progress. The three green dots are the EAPs. But notice where the "you are here" marker actually sits: it's not on an EAP anymore, it's on core Gradle. That's the important shift. The "recent" movement, the Project Types rename, the binding API, has been landing directly in mainline Gradle releases not in a new standalone EAP. After that: incubating, then stable, both still on the horizon with no committed date. If you want to track this watch the core Gradle release notes not just the declarative-gradle EAP repo.

---

# How AI Can Help

<div style="text-align:center; margin:20px 0;">
<div style="background:#1e1e2e; border:2px solid rgba(61,220,132,0.5); border-radius:12px; padding:24px 36px; display:inline-block; max-width:750px;">
<p style="font-size:1.3em; margin:0 0 12px 0;">🤖 + 🐘 = ❤️</p>
<p style="font-size:1.1em; margin:0 0 16px 0;">There is an <strong style="color:#3DDC84;">open-source</strong> AI migration skill included in this presentation's repo.</p>
<p style="font-size:0.85em; margin:0; color:#ccc;">Drop it into your project and go.</p>
</div>
</div>

| File | Works With |
|---|---|
| `.windsurf/workflows/migrate-to-dcl.md` | Windsurf (Cascade) |
| `.claude/commands/dcl-migration-check.md` | Claude Code |

<p style="text-align:center; margin-top:16px; font-size:0.85em;"><strong style="color:#00c9db;">github.com/wildsmith/droidcon-2026</strong></p>

> Speaker Notes: Before I walk through the details, I want to highlight that there's an open-source AI migration skill in the same GitHub repo as these slides. Two versions, one for Windsurf and one for Claude Code. You can clone the repo, drop the file into your project, and run it against your build files. If you're on a different AI coding assistant, the markdown is self-explanatory enough to adapt.

---

# AI-Assisted Migration: The Opportunity

**Problem**: Migrating hundreds or thousands of modules manually is not feasible.

**Solution**: AI tooling that can:

<div style="padding-left:40px;">

1. <strong style="color:#00c9db;">Analyze</strong> your current build files for DCL readiness
2. <strong style="color:#00c9db;">Identify</strong> Isolated Projects violations
3. <strong style="color:#00c9db;">Generate</strong> equivalent DCL configurations
4. <strong style="color:#00c9db;">Create</strong> custom Project Type scaffolding
5. <strong style="color:#00c9db;">Detect</strong> incompatible plugin patterns

</div>

> Speaker Notes: At Capital One, we have thousands of modules. Manual migration is not realistic. So what can AI tooling do? A few things. Analyze your current build files for DCL readiness - are they simple enough to convert? Identify Isolated Projects violations - the cross-project patterns that block you. Generate equivalent DCL configurations from your existing KTS files. Scaffold custom Project Types based on what your convention plugins already do. And detect incompatible plugin patterns like afterEvaluate or rootProject access.

---

# AI Skill: Example Output

<div style="font-size:1.15em;">

```markdown
## DCL Migration Report: :core:network

📊 **Readiness: ✅ Ready**

**Project Type:** `androidLibrary`

### Generated DCL:
androidLibrary {
    namespace = "com.example.core.network"
    dependencies {
        // libs.okhttp converted to GAV (no catalogs in .dcl yet):
        implementation("com.squareup.okhttp3:okhttp:4.12.0")
    }
    testing {
        dependencies {
            implementation("junit:junit:4.13.2")
        }
    }
}

### ❌ Hard Blockers: 0
### ⚠️ Warnings:
- `compileSdk = 35` → move to settings defaults
- `jvmTarget = "17"` → derived from jdkVersion
```

</div>

> Speaker Notes: Here's what the skill outputs. A migration report per module - what can move, what the DCL looks like, and any blockers. For simple modules, it's fully automated. For complex ones, it gives you a roadmap.

---

# AI Skill: Detecting Blockers

<div style="font-size:1.15em;">

```markdown
## DCL Migration Report: :app

📊 **Readiness: ❌ Not Ready**

**Project Type:** `androidApplication`

### ❌ Hard Blockers (4 found):
1. **Line 15**: `if (isCI) { ... }` (Conditional logic)
   → Move to convention plugin or Project Type implementation
2. **Line 32**: `project(":core").android.compileSdk` (Cross-project access)
   → Use proper dependency declarations
3. **Line 45**: `tasks.register("customTask") { ... }` (Custom task)
   → Move to included build plugin
4. **Line 58**: `extra["buildTimestamp"] = ...` (Extra properties)
   → Replace with typed Project Type properties
```

</div>

> Speaker Notes: For blocked modules, the skill doesn't just say "can't do it", it tells you exactly why and suggests the fix pattern.

---

# AI Migration Skill: In Action

<div style="font-size:0.9em;">

```bash
$ claude dcl-migration-check core/network/build.gradle.kts

## DCL Migration Report: :core:network

📊 **Readiness: ✅ Ready**

### ✅ Compatible:
- Plugin: com.android.library
- Dependencies: 8 (catalog refs converted to GAV for .dcl)
- Namespace: com.example.core.network
- Version catalog: gradle/libs.versions.toml detected (inlined to GAV)

### ⚠️ Warnings:
- `kotlinOptions { jvmTarget = "17" }` 
  → Derived from jdkVersion in settings defaults

### ❌ Hard Blockers:
- None found!

### Generated DCL:
androidLibrary {
    namespace = "com.example.core.network"
    dependencies {
        implementation("com.squareup.okhttp3:okhttp:4.12.0")
        implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
    }
    kotlinSerialization {
        version = "1.6.3"
        jsonEnabled = true
    }
    testing {
        dependencies {
            implementation("io.mockk:mockk:1.13.8")
        }
    }
}
```

</div>

> Speaker Notes: This is what the output looks like in practice. The module is classified as Ready - zero hard blockers. The only action item is a Warning: kotlinOptions should be derived from jdkVersion in settings defaults. Notice the generated DCL inlines your version catalog references - libs.okhttp becomes the full GAV coordinate - because fully declarative .dcl files can't use catalog accessors yet. The skill does that conversion for you from your libs.versions.toml. Most leaf modules in well-maintained projects will be Ready or Nearly Ready.

---

# Try It: `gradle init` with DCL

<div style="font-size:1.2em;">

```bash
# Generate a new Android project with DCL (Gradle 8.12+)
gradle init \
  -Dorg.gradle.buildinit.specs=org.gradle.experimental.android-ecosystem-init:0.1.44

# You will be asked:
# "Additional project types were loaded. 
#  Do you want to generate a project using a 
#  contributed project specification?"
# Answer: yes
```

</div>

> Speaker Notes: The fastest way to try DCL today is gradle init with the experimental spec. It generates a working Android project with DCL files, the right Gradle version, and IDE support ready to go.

---

# Try It: Existing Samples

| Sample | Repository |
|---|---|
| Android App | [`gradle/declarative-samples-android-app`](https://github.com/gradle/declarative-samples-android-app) |
| Java App | [`gradle/declarative-samples-java-app`](https://github.com/gradle/declarative-samples-java-app) |
| Kotlin App | [`gradle/declarative-samples-kotlin-app`](https://github.com/gradle/declarative-samples-kotlin-app) |
| Now In Android | [`gradle/nowinandroid` (dcl branch)](https://github.com/gradle/nowinandroid/tree/main-declarative) |
| Bleeding Edge | [`gradle/declarative-gradle` unified-prototype](https://github.com/gradle/declarative-gradle/tree/main/unified-prototype) |

```bash
git clone https://github.com/gradle/declarative-samples-android-app
cd declarative-samples-android-app
./gradlew build
```

> Speaker Notes: Clone any of these, open in Android Studio Nightly with the flags enabled, and you'll see DCL in action. The Now In Android sample is particularly interesting because it's a real-world app.

---

# What to Do Today

1. **Try the samples** - get hands-on experience
2. **Run `--isolated-projects`** on your build - see your violations
3. **Move `allprojects`/`subprojects`** to settings or convention plugins
4. **Audit your custom plugins** for cross-project access
5. **Subscribe to [Gradle newsletter](https://newsletter.gradle.org/)** for DCL updates
6. **Provide feedback** - [github.com/gradle/declarative-gradle](https://github.com/gradle/declarative-gradle/blob/main/docs/feedback.md)

> Speaker Notes: What to do today. You don't have to wait for DCL to be stable to start preparing. Running isolated projects analysis today will surface problems. Removing allprojects/subprojects is good hygiene regardless. Start with the easy wins.

---

# FAQ: General

<div style="font-size:0.8em;">

| Question | Short Answer |
|---|---|
| Are Isolated Projects required for DCL? | No. They are separate but complementary. |
| What happens to `settings.gradle.kts`? | Becomes `settings.gradle.dcl` |
| What happens to root `build.gradle.kts`? | Eliminated; logic moves to included builds |
| Do `gradle.properties` change? | No. They remain as-is. |
| **How long until stability?** | **Isolated Projects has been maturing across the Gradle 9.x line; DCL itself is still experimental - no committed date.** |
| **Does DCL replace Kotlin DSL?** | **For project config, yes. Build logic stays KTS.** |
| What about `buildscript {}` blocks? | Legacy/discouraged. Migrate to `plugins {}` first, then DCL. |
| Do build scans still work? | Yes. Completely unchanged. |
| **How does DCL interact with configuration cache?** | **Complementary. Config cache caches the task graph; Isolated Projects builds on it to configure projects in parallel, and DCL speeds the config phase.** |
| **Is DCL still actively moving forward?** | **Yes - the groundwork has been landing directly in core Gradle (rename, binding API out of `internal`). Dedicated team, Google partnership.** |

</div>

> Speaker Notes: Before we get into the Q&A I've noted some FAQs responses: (1) Isolated Projects and DCL are complementary but independent. DCL files are inherently isolated because they cannot contain imperative code. You can adopt either without the other. (2) settings.gradle.kts becomes settings.gradle.dcl which declares plugins, ecosystem configuration, and shared defaults. (3) Root build.gradle.kts is typically eliminated; root-level tasks or logic moves into included build plugins. (4) gradle.properties files remain unchanged - JVM args, feature flags, credentials stay there. (5) Isolated Projects has been maturing across the Gradle 9.x line; DCL itself has no committed stability date yet. Subscribe to the Gradle newsletter. (6) Long-term, DCL is the intended default for project configuration. Kotlin DSL remains for build logic, included builds, and custom plugins. (7) If you still have buildscript { classpath(...) } blocks, step one is migrating to the plugins {} block. That's a prerequisite before DCL. The buildscript pattern is already deprecated in modern Gradle.

---

# FAQ: Migration & Compatibility

| Question | Short Answer |
|---|---|
| Can I mix `.gradle.dcl` and `.gradle.kts`? | Yes. Designed for incremental migration. |
| Do version catalogs (`libs.versions.toml`) still work? | In `.kts`, yes. In fully declarative `.dcl` files, **not yet** - inline to GAV coords. |
| What about Groovy DSL projects? | KTS migration recommended first. Groovy → KTS → DCL is the safest path. |
| Does `./gradlew assembleDebug` still work? | Yes. Task invocation is unchanged. CI scripts don't change. |
| Is there a KMP Project Type? | Prototype exists (`kotlinMultiplatform`). Not stable yet. |

> Speaker Notes: These are some migration-specific questions. (1) Gradle explicitly supports mixed migration with some projects on DCL and others on KTS in the same build. (2) Version catalogs are completely separate from build scripts - they define dependency coordinates and versions, not project configuration. libs.versions.toml stays exactly as-is. (3) If you're on Groovy, KTS migration first is the safest path. Groovy to KTS to DCL. The syntax translation from Groovy to DCL directly is less well-tested. (4) Task invocation, CI commands, and the entire execution model are unchanged. Only configuration syntax changes. (5) There's a prototype kotlinMultiplatform Project Type in the unified-prototype repo. If you're doing KMP, keep an eye on it.

---

# FAQ: Plugin Authors

| Question | Short Answer |
|---|---|
| What should plugin authors do now? | Test with `--isolated-projects` |
| **Can my convention plugin become a Project Type?** | **Yes, conceptually the same thing. Why rewrite? You get typed IDE support + tooling mutations.** |
| What about config-time code generation? | Move to task execution time with `Provider<T>` |

**Key takeaway:** If you already use lazy APIs (`Property<T>`, `Provider<T>`, `TaskProvider`) and avoid cross-project access, you are in good shape.

> Speaker Notes: Plugin authors have the most work ahead. Start by testing with --isolated-projects. Consider whether your plugin should define a Project Type or register Project Features. The DCL docs note that plugin author adoption is "not yet ready" but getting ready now saves you time later. Convention plugins that set compileSdk, minSdk, apply Kotlin, and wire dependencies are essentially Project Types already. The formal API is still experimental. For config-time code gen (annotation processors, code generators, anything inspecting the project graph during configuration), the pattern is to move generation to task execution time using lazy Provider APIs.

---

# Summary

- **DCL is the future** of Gradle build configuration
- **DCL work is landing in core Gradle**, beyond the April 2025 standalone EAP - still experimental, not production-ready
- **Start preparing** by eliminating `allprojects`/`subprojects`
- **Run `--isolated-projects`** to see your violation count
- **Your CI doesn't change** - same commands, same outputs
- **Security win** - DCL files are auditable, non-executable data

> Speaker Notes: To summarize - DCL is coming. It's not here yet for production, but the direction is clear. The best thing you can do today is prepare: eliminate cross-project patterns, test with isolated projects, and keep an eye on both the EAP samples and the core Gradle releases, which is where the recent DCL work has been landing.

---

# Summary (cont.)

- **IDE support** exists in Android Studio, IntelliJ, VS Code, Eclipse
- **Version catalogs** - work in `.kts`, not yet in `.dcl` (inline to GAV); **Gradle properties** unchanged
- **AI tooling** can accelerate migration when DCL stabilizes
- **Plugin authors**: test your plugins with Isolated Projects now

> Speaker Notes: And remember - this is NOT a big-bang migration. Your CI doesn't change, and you can mix DCL and KTS in the same build during the transition. One caveat to be precise about: version catalogs keep working in your .kts files, but fully declarative .dcl files can't reference libs.x accessors yet, so those get inlined to GAV coordinates - Gradle says catalog support in DCL is coming. IDE support is real today in nightly builds. AI tooling can help you triage thousands of modules when the time comes.

---

# Resources

| Question | Short Answer |
|---|---|
| Declarative Gradle Homepage | [declarative.gradle.org](https://declarative.gradle.org/) |
| GitHub Repository | [github.com/gradle/declarative-gradle](https://github.com/gradle/declarative-gradle) |
| Isolated Projects Docs | [docs.gradle.org/.../isolated_projects.html](https://docs.gradle.org/current/userguide/isolated_projects.html) |
| Migration Guide | [DCL Migration Guide](https://github.com/gradle/declarative-gradle/blob/main/docs/reference/migration-guide.md) |
| Migration Case Study | [Gradle-Client Migration](https://github.com/gradle/declarative-gradle/blob/main/docs/reference/migration-case-study.md) |
| VS Code Extension | [gradle.github.io/declarative-vscode-extension](https://gradle.github.io/declarative-vscode-extension/) |
| Droidcon NYC 2024 Talk | [Declarative Gradle on Android](https://www.droidcon.com/2024/10/17/declarative-gradle-on-android/) |
| KotlinConf 2024 Talk | Developer-first Gradle builds - Sterling Greene & Paul Merlin |
| Gradle Slack | `#declarative_gradle` channel |
| Gradle Newsletter | [newsletter.gradle.org](https://newsletter.gradle.org/) |

> Speaker Notes: [PACING: This is a reference slide. Don't read it row by row. Give 5 seconds then guide.] These links are all in the slides, which are available in the GitHub repo. If you only bookmark two things: the migration guide and the Gradle Slack channel. The migration guide walks through a real project conversion step by step. The Slack channel, #declarative_gradle, is where the Gradle team actively answers questions. The DroidCon NYC 2024 talk on Declarative Gradle on Android is also excellent if you want a deeper technical dive from the Gradle team's perspective. [TRANSITION: And with that, let's open it up.]

---

# Thank You!

<img src="profile.JPG" class="profile-img" alt="Tom Belk">

**Tom Belk** | Android Platform Lead | Capital One

<div style="margin:20px 0; padding:16px 24px; border-left:4px solid #3DDC84; background:rgba(61,220,132,0.05); border-radius:0 8px 8px 0;">
<p style="font-size:1.1em; font-style:italic; margin:0; color:#3DDC84;">Configuration is data, not code. Start treating it that way.</p>
</div>

*Questions?*

> Speaker Notes: I want to thank folks for attending this presentation and would like to open the floor for Q&A. All resource links are available in the presentation slides and the GitHub repo. [REFERENCE - don't read these, but have answers ready: "When will this be production ready?" - no firm date. "Does this replace Kotlin DSL?" - eventually, but KTS remains supported. "What about Groovy?" - DCL is separate from both Groovy and KTS.]

---

<!-- .slide: data-visibility="hidden" -->

# Appendix: Slide Design Suggestions

## Visual Theme
- **Dark background** (dark navy/charcoal - `#1a1a2e` or `#0f0f23`) with bright syntax-highlighted code
- **Accent colors**: Gradle teal (`#02303A`), Gradle green (`#1BA8A0`), Android green (`#3DDC84`)
- Use Gradle's elephant logo and brand colors (teal/green accents)
- DroidCon branding in footer (small, bottom-right)
- **Font**: JetBrains Mono for code, Inter or Montserrat for body text
- **Code blocks**: Use a dark IDE theme (Dracula or One Dark) with syntax highlighting
- **Slide transitions**: Simple fade or none - fast-moving slides don't need fancy transitions

## Per-Slide Background Suggestions
| Slides | Background / Visual |
|---|---|
| 1 (Title) | Dark gradient (#0f0f23 → #1a1a2e), Gradle elephant + Android robot silhouette, your name/company logo bottom |
| 2 (About Me) | Subtle circuit board pattern overlay, professional headshot in circle |
| 3 (Agenda) | Clean numbered list on dark background, green accent markers |
| 4 (Problem) | Screenshot of a massive build.gradle.kts (wall of text), slightly blurred edges - convey overwhelm |
| 5-6 (What/Principles) | Clean diagram of "What vs How" separation, white/teal boxes on dark |
| 7-8 (Why) | Side-by-side comparison graphics with red/green indicators |
| 9-12 (Code) | Full-bleed code snippets with syntax highlighting (Dracula theme), dark background |
| 13 (Settings) | Architecture diagram: settings.gradle.dcl → project files flow, connecting lines in teal |
| 14 (Comparison) | Split screen: left (old/dim), right (new/bright green glow) |
| 15-16 (IDE) | Screenshots from the official Declarative Gradle YouTube demos |
| 17 (Gradle Client) | Screenshot of Gradle Client app with arrows highlighting mutations |
| 18-20 (Isolated Projects) | Red-bordered code boxes (violations) vs green-bordered (valid) |
| 21-22 (Blockers) | Danger-tape overlay effect (diagonal yellow/black stripes at edges) |
| 23-25 (Plugin Auth) | Dark background with plugin puzzle-piece graphic |
| 26-31 (Migration) | Step-indicator dots (like a progress bar) across the top |
| 32 (Violations) | Terminal screenshot (green text on black) |
| 33-34 (Status/Roadmap) | Timeline graphic with glowing nodes at each EAP |
| 35-38 (AI orig) | Terminal/IDE screenshots with AI assistant overlay |
| 39-40 (Try It) | Terminal screenshot of gradle init with arrow annotations |
| 43-44 (FAQ) | Q&A graphic - speech bubble icons, dark background with teal accent bubbles |
| 45-47 (Enterprise) | Enterprise building icon or monorepo graph visualization (nodes+edges) |
| 48-50 (AI Skills) | Split: left = code/yaml definition, right = terminal output |
| 51 (Summary) | Key points as floating cards with subtle depth shadows |
| 52 (Q&A) | Your contact info, large QR code linking to slides/repo |

## Meme Suggestions
- **Slide 4** (Problem): "This is fine" dog surrounded by build.gradle.kts flames - caption: *"My 300-line build file is totally maintainable"*
- **Slide 11** (Minimal Module): Thanos "I used the build file to destroy the build file" - showing 2-line DCL
- **Slide 19** (Isolated Projects blocks): Drake meme - rejecting `allprojects`, approving `defaults`
- **Slide 21** (Why blocked): "You guys are getting parallel configuration?" (We're the Millers meme)
- **Slide 30** (Pitfalls): Astronaut meme - "Wait, you can't write code in build files?" / "Never could" 🔫
- **Slide 33** (Status): "Are we there yet?" Shrek meme - for production readiness
- **Slide 43** (FAQ): Galaxy brain meme progression: `allprojects` → convention plugin → Project Type → DCL defaults
- **Slide 45** (Enterprise): "It's the same picture" (corporate needs you to find the difference) - convention plugin vs Project Type
- **Slide 47** (Insight): Spider-Man pointing at Spider-Man - convention plugin pointing at Project Type

## Slide Density Guidelines
- **Code slides**: Max 15-20 lines of code per slide, generous font size (18pt+)
- **Text slides**: Max 5-6 bullet points, no full paragraphs
- **Comparison slides**: Use columns or tables, never walls of text
- **FAQ slides**: 3-4 Q&A pairs per slide max - if more, split across slides
- **Meme slides**: Full-bleed image with minimal overlaid text
