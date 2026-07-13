# Declarative Gradle: DroidCon Orlando 2026

**"The End of build.gradle: Embracing Declarative Gradle and Project Types"**

A presentation for DroidCon Orlando 2026 covering Declarative Gradle (DCL), Project Types, Isolated Projects, migration strategies, and AI-assisted migration tooling.

## Repository Contents

| Path | Description |
|---|---|
| `index.html` | Reveal.js presentation (serves the slides) |
| `droidcon-orlando-2026-declarative-gradle.md` | Slide content in Markdown (with speaker notes, AI skill demos reflect latest capabilities) |
| `The End of build.gradle...pdf` | Pre-exported PDF of the presentation |
| `profile.JPG` | Speaker profile images |
| `DroidCon_icon_rotatet.avif` | DroidCon icon used in the slide theme |
| `.windsurf/workflows/migrate-to-dcl.md` | Windsurf AI skill for DCL migration auditing |
| `.claude/commands/dcl-migration-check.md` | Claude Code command for DCL migration auditing |

## Viewing the Presentation

### Local Server

```bash
python3 -m http.server 3000
```

Then open `http://localhost:3000` in your browser.

### Speaker Notes

Press `S` while viewing the presentation to open the speaker notes window.

### Keyboard Shortcuts

| Key | Action |
|---|---|
| `→` / `Space` | Next slide |
| `←` | Previous slide |
| `S` | Speaker notes |
| `F` | Fullscreen |
| `Esc` / `O` | Slide overview |

## Exporting as PDF

Reveal.js supports PDF export via the browser print dialog:

1. **Open the print-friendly URL**:
   ```
   http://localhost:3000/?print-pdf
   ```
2. **Wait** for all slides to render (they will display stacked vertically)
3. **Open Print dialog**: `Cmd+P` (macOS) or `Ctrl+P` (Windows/Linux)
4. **Configure settings**:
   - Destination: **Save as PDF**
   - Layout: **Landscape**
   - Margins: **None**
   - ✅ Enable **Background graphics**
5. **Save**

> **Tip**: Chrome/Chromium gives the best PDF export results. Firefox may have rendering differences.

## AI Migration Skills

This repository includes AI coding assistant skills that audit Gradle builds for DCL readiness **and perform the migration**. Both skills:

- **Pre-scan checks:** Verify Gradle version/DCL availability, flag Groovy→KTS prerequisite, detect composite builds (`includeBuild()`)
- Classify each module's Project Type (`androidApplication`, `androidLibrary`, `javaLibrary`, `kotlinJvmLibrary`)
- Parse `gradle/libs.versions.toml` and **inline** version catalog references (`libs.x`) to GAV coordinates — fully declarative `.dcl` files don't support catalog accessors yet
- Emit raw `project(":path")` dependencies in generated `.dcl` (as used by the official DCL samples); `projects.x` type-safe accessors are used only in `.kts` contexts
- Check for DCL blockers (imperative logic, cross-project access, eager APIs, etc.)
- Check for Isolated Projects violations (allprojects, subprojects, eager task APIs, project access in task actions)
- **Assess readiness** using actionable categories (✅ Ready, ⚠️ Nearly Ready, 🔶 Has Blockers, ❌ Not Ready) instead of arbitrary scores
- Generate equivalent `.gradle.dcl` with dependencies sorted alphabetically by configuration group
- Generate `settings.gradle.dcl` with `dependencyResolutionManagement`, `defaults`, and ecosystem plugin
- Auto-fix mechanical Isolated Projects violations (eager → lazy API replacements)
- Offer to create migration files alongside originals for safe diffing
- **Suggest validation commands** (`./gradlew --isolated-projects`, `--dry-run`) after migration
- **Violation source attribution:** Tag every finding as `project` (fixable by the team) or `third-party:<plugin-id>` (requires plugin author update)
- **JSON output mode** (`--json`): Machine-readable `dcl-migration-report.json` for CI pipelines, dashboards, and downstream tooling

### Scan Modes

Both skills support two scan modes:

| Mode | Files Scanned | Use When |
|---|---|---|
| **Standard** (default) | `build.gradle.kts`, `build.gradle`, `settings.gradle.kts` | Quick audit of build scripts |
| **Deep** | Standard + `.kt` source files in `buildSrc/`, `build-logic/`, plugin modules | Your build logic lives in Kotlin source files (convention plugins, custom tasks, plugin implementations) |

### Output Formats

| Format | Flag | Description |
|---|---|---|
| **Markdown** (default) | — | Human-readable report with emoji categories and recommendations |
| **JSON** | `--json` | Machine-readable `dcl-migration-report.json` for CI integration |

Deep mode is recommended for projects that implement Gradle plugins or convention plugins in `.kt` files, since those files contain the same anti-patterns (cross-project access, eager APIs, `afterEvaluate`) that block Isolated Projects and DCL adoption.

### Using with Windsurf (Cascade)

1. Copy `.windsurf/workflows/migrate-to-dcl.md` into your project's `.windsurf/workflows/` directory
2. Invoke with `/migrate-to-dcl` in Windsurf chat
3. Cascade will ask whether you want Standard or Deep scan mode
4. Optionally provide a specific module path, or let it scan the entire project

**Examples:**
- `/migrate-to-dcl` — scan all modules (Standard mode, Cascade will ask)
- Tell Cascade: "run /migrate-to-dcl on the :core:network module with deep scan"

### Using with Claude Code

1. Copy `.claude/commands/dcl-migration-check.md` into your project's `.claude/commands/` directory
2. Invoke from the command line:

```bash
# Scan a specific module (Standard mode)
claude dcl-migration-check path/to/module

# Scan all modules (Standard mode)
claude dcl-migration-check

# Deep scan — also analyze .kt build logic source files
claude dcl-migration-check --deep

# Deep scan a specific module
claude dcl-migration-check path/to/module --deep

# JSON output for CI pipelines
claude dcl-migration-check --json

# Deep scan with JSON output
claude dcl-migration-check --deep --json
```

## Resources

- [Declarative Gradle Homepage](https://declarative.gradle.org/)
- [GitHub: gradle/declarative-gradle](https://github.com/gradle/declarative-gradle)
- [Isolated Projects Docs](https://docs.gradle.org/current/userguide/isolated_projects.html)
- [DCL Migration Guide](https://github.com/gradle/declarative-gradle/blob/main/docs/reference/migration-guide.md)
- [Gradle Newsletter](https://newsletter.gradle.org/)

## License

Presentation content and AI skills are provided under the Apache 2.0 License. Code samples derived from [gradle/declarative-gradle](https://github.com/gradle/declarative-gradle) (Apache 2.0).
