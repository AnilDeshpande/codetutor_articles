# 🧩 Adding Detekt to an Android Project for Static Code Analysis

Static Code Analysis is an essential part of maintaining **clean**, **consistent**, and **bug-free** Android projects.
In Kotlin-based projects, **[Detekt](https://detekt.dev/)** is one of the most popular tools used to perform this analysis automatically.

This guide explains:

* What Detekt is
* How to add and configure it in your Android project
* How to run it locally and in CI/CD
* What a typical workflow looks like

---

## 📘 Table of Contents

1. [What is Detekt](#-what-is-detekt)
2. [Step-by-Step Setup](#-step-by-step-setup)

   * [Step 1: Add Plugin](#step-1-add-plugin)
   * [Step 2: Generate Config File](#step-2-generate-config-file)
   * [Step 3: Run Detekt Locally](#step-3-run-detekt-locally)
   * [Step 4: Integrate Git Hooks (Optional)](#step-4-integrate-git-hooks-optional)
   * [Step 5: Run in CI/CD (GitHub Actions Example)](#step-5-run-in-cicd-github-actions-example)
3. [What Detekt Checks For](#-what-detekt-checks-for)
4. [Customizing Rules](#-customizing-rules)
5. [Combining with Other Tools](#-combining-with-other-tools)
6. [Typical Quality Gate Flow](#-typical-quality-gate-flow)
7. [Integration with SonarQube (Optional)](#-integration-with-sonarqube-optional)
8. [Summary and Typical Workflow](#-summary-and-typical-workflow)
9. [Date & References](#-date--references)

---

## 🧠 What is Detekt

**Detekt** is a **static code analysis tool for Kotlin** that scans your codebase for:

* Code smells (long methods, complex classes, unused parameters)
* Maintainability issues (duplicate conditions, deep nesting)
* Style violations (naming, braces, empty blocks)
* Potential bugs and anti-patterns

It helps you **enforce consistent code quality** across your project automatically — just like how Android Lint works for resources and XML.

---

## ⚙️ Step-by-Step Setup

### ✅ Step 1: Add Plugin

In your **root** `build.gradle.kts`:

```kotlin
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.1" apply false
}
```

Then in your **app module** (`app/build.gradle.kts`):

```kotlin
plugins {
    id("io.gitlab.arturbosch.detekt")
}

detekt {
    config = files("$rootDir/config/detekt/detekt.yml")
    buildUponDefaultConfig = true
    reports {
        html.required.set(true)
        xml.required.set(true)
        sarif.required.set(true)
    }
}
```

📁 Recommended project structure:

```
project-root/
 ├── app/
 ├── config/
 │    └── detekt/
 │         └── detekt.yml
 └── build.gradle.kts
```

---

### ✅ Step 2: Generate Config File

Run:

```bash
./gradlew detektGenerateConfig
```

This creates a `detekt.yml` file at your root directory.
Move it to `config/detekt/detekt.yml` and adjust rules as needed.

---

### ✅ Step 3: Run Detekt Locally

Run analysis manually with:

```bash
./gradlew detekt
```

Find reports here:

```
app/build/reports/detekt/detekt.html
```

---

### ✅ Step 4: Integrate Git Hooks (Optional)

To ensure code is checked before commits, create a pre-commit hook:

```bash
echo "./gradlew detekt || exit 1" > .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
```

---

### ✅ Step 5: Run in CI/CD (GitHub Actions Example)

Add this workflow file:
**`.github/workflows/code-quality.yml`**

```yaml
name: Code Quality Checks

on:
  pull_request:
  push:
    branches: [ main, develop ]

jobs:
  detekt:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17

      - name: Run Detekt
        run: ./gradlew detekt

      - name: Upload Detekt Report
        uses: actions/upload-artifact@v4
        with:
          name: detekt-report
          path: app/build/reports/detekt/detekt.html
```

This ensures every pull request is automatically analyzed —
the build fails if new issues appear.

---

## 🔍 What Detekt Checks For

| Category           | Description              | Example Rules                      |
| ------------------ | ------------------------ | ---------------------------------- |
| **style**          | Formatting and naming    | Variable naming, spacing, braces   |
| **complexity**     | Measures maintainability | Long functions, deep nesting       |
| **performance**    | Inefficient constructs   | Unnecessary object creation        |
| **potential-bugs** | Likely bugs              | Duplicate conditions, unsafe casts |
| **empty-blocks**   | Missing logic            | Empty catch or when branch         |
| **exceptions**     | Error handling practices | Swallowed exceptions               |
| **comments**       | Documentation quality    | TODOs, missing KDoc                |

---

## 🧰 Customizing Rules

Open `config/detekt/detekt.yml` and tweak to your liking:

```yaml
style:
  MagicNumber:
    active: true
    ignoreNumbers: [0, 1, -1]
  MaxLineLength:
    active: true
    maxLineLength: 120

complexity:
  LongMethod:
    threshold: 40
```

---

## ⚙️ Combining with Other Tools

It’s common to combine **Detekt** with **ktlint** or **Spotless** for full code quality checks.

Example Gradle task:

```bash
./gradlew spotlessCheck detekt
```

| Tool                  | Role                                                |
| --------------------- | --------------------------------------------------- |
| **ktlint / Spotless** | Enforce formatting and style                        |
| **Detekt**            | Enforce logic quality and structure                 |
| **Android Lint**      | Android-specific XML, resource, and security issues |

---

## 🧱 Typical Quality Gate Flow

In a robust Android CI pipeline:

```bash
./gradlew spotlessCheck detekt lintDebug testDebugUnitTest
```

| Step                          | Tool            | Purpose                          |
| ----------------------------- | --------------- | -------------------------------- |
| **1. spotlessCheck / ktlint** | Formatter       | Enforce style consistency        |
| **2. detekt**                 | Static analysis | Catch code smells and complexity |
| **3. lintDebug**              | Android Lint    | Platform-level checks            |
| **4. testDebugUnitTest**      | Tests           | Verify logic correctness         |

This creates a **multi-layered safety net** for your codebase.

---

## 🌐 Integration with SonarQube (Optional)

To include Detekt reports in SonarQube dashboards:

```bash
./gradlew detekt sonar
```

Configure SonarQube to read Detekt’s XML report for unified project metrics (bugs, maintainability, code smells).

---

## ✅ Summary and Typical Workflow

| Step                | Command                                 | Purpose          |
| ------------------- | --------------------------------------- | ---------------- |
| **Install plugin**  | Add `id("io.gitlab.arturbosch.detekt")` | Enable Detekt    |
| **Generate config** | `./gradlew detektGenerateConfig`        | Create rule file |
| **Analyze locally** | `./gradlew detekt`                      | Run checks       |
| **Integrate CI**    | Add GitHub Action                       | Automate checks  |
| **Customize rules** | Edit `detekt.yml`                       | Tune behavior    |

---

### 🧭 Typical Developer Workflow

1. 🧑‍💻 Developer writes code and runs Detekt locally.
2. 🤖 CI/CD runs Detekt automatically on PR.
3. 🛑 Build fails if new issues are found.
4. ✅ Developer fixes issues and pushes again.
5. 📊 Reports stored as artifacts or integrated into SonarQube.

This ensures **consistent, high-quality Kotlin code** across the team.

---

## 🗓️ Date & References

**Compiled:** November 8, 2025

**References:**

* [Detekt Official Documentation](https://detekt.dev/)
* [Android Developers: Code Quality Tools](https://developer.android.com/studio/write/lint)
* [SonarQube Documentation](https://docs.sonarsource.com/sonarqube/)
* [GitHub Actions Documentation](https://docs.github.com/actions)
* [Gradle Plugin Portal: Detekt](https://plugins.gradle.org/plugin/io.gitlab.arturbosch.detekt)

---

✅ **In short:**

> **Detekt** is your Kotlin code reviewer — it automates static analysis, ensures maintainable code, and integrates seamlessly into Android CI/CD pipelines.

---

📄 **Suggested file name:**
`android-detekt-static-code-analysis-setup.md`
