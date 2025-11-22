## FAQ: Version Catalogs (.toml) + Lock Files for Android

**Q1: Do I really need a `libs.versions.toml` file? What problem does it solve?**
**A:** Yep, it’s worth it. The TOML “version catalog” centralizes all your dependency and plugin versions. No more hunting through 12 modules to bump Retrofit. It also makes diffs cleaner and PRs smaller. Pair it with lock files to freeze what actually resolves on CI so “it worked on my machine” stops being a thing.

---

**Q2: Creating a `.toml` from scratch looks tedious. Any faster way?**
**A:** Totally. A few options:

* **IDE help:** Android Studio/IntelliJ have plugins that generate and edit a version catalog.
* **Script it:** Quick grep/sed (or small Kotlin script) to scan `build.gradle(.kts)` files and emit TOML stubs.
* **Ask AI:** Point an AI at your repo and let it draft the initial `libs.versions.toml` for you. You then tidy names and group things.

---

**Q3: Where does the file live and how do I wire it?**
**A:** Put it at `gradle/libs.versions.toml`. In `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories { google(); mavenCentral() }
    versionCatalogs {
        create("libs") {
            from(files("gradle/libs.versions.toml"))
        }
    }
}
```

Then in modules:

```kotlin
dependencies {
    implementation(libs.retrofit)
    testImplementation(libs.junit)
}
```

---

**Q4: What goes inside the TOML?**
**A:**

```toml
[versions]
kotlin = "2.0.0"
retrofit = "2.11.0"

[libraries]
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
okhttp   = { module = "com.squareup.okhttp3:okhttp", version = "4.12.0" }

[bundles]
networking = ["retrofit", "okhttp"]

[plugins]
android-app = { id = "com.android.application", version = "8.7.2" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
```

Use `libs.bundles.networking` when you want a group.

---

**Q5: How do lock files fit into this?**
**A:** Version catalogs pin the versions you *ask* for. Lock files pin the versions you *actually get* after Gradle resolves transitives. That means repeatable builds on every machine/CI.

---

**Q6: How do I enable dependency locking?**
**A:**

1. In the root build (or all modules):

```kotlin
dependencyLocking {
    lockAllConfigurations()
}
```

2. Write lock files once:

```bash
./gradlew --write-locks
```

This creates files under `gradle/dependency-locks/`. Commit them.

---

**Q7: How do I update a single dependency without blowing up the lock files?**
**A:**

* Bump it in `libs.versions.toml`, then:

```bash
./gradlew --update-locks group:name
# e.g. ./gradlew --update-locks com.squareup.retrofit2:retrofit
```

* Or just rerun:

```bash
./gradlew --write-locks
```

and commit the changed lockfiles.

---

**Q8: Can I keep using dynamic versions like `1.+` or `latest.release`?**
**A:** Please don’t. They defeat reproducibility. If you *must*, locking will freeze whatever Gradle picked *today*, which can surprise you later. Prefer exact versions in the catalog.

---

**Q9: What about BOMs (platforms) like Firebase or OkHttp BOM?**
**A:** Declare the BOM in the catalog and use `platform()`:

```toml
[libraries]
okhttp-bom = { module = "com.squareup.okhttp3:okhttp-bom", version = "4.12.0" }
okhttp     = { module = "com.squareup.okhttp3:okhttp" } # no version here
```

```kotlin
dependencies {
    implementation(platform(libs.okhttp.bom))
    implementation(libs.okhttp)
}
```

---

**Q10: How do I handle Gradle plugins with the catalog?**
**A:** Put them under `[plugins]` in TOML and apply in `build.gradle.kts`:

```kotlin
plugins {
    alias(libs.plugins.android.app)
    alias(libs.plugins.kotlin.android)
}
```

Pinning plugin versions here keeps them consistent across modules.

---

**Q11: Multi-module project—one catalog or many?**
**A:** Usually one global `libs` is perfect. If you’re in a mono-repo with very different stacks, you can create multiple catalogs and import multiple TOML files in `settings.gradle.kts`.

---

**Q12: How do I migrate an existing project quickly?**
**A:**

1. Create `gradle/libs.versions.toml`.
2. Move shared versions there; give readable aliases (`coil`, `room.ktx`, `androidx-core-ktx`).
3. Replace hardcoded versions in module `build.gradle(.kts)` with `libs.*` aliases.
4. Enable locking and run `./gradlew --write-locks`.
5. Commit the TOML + lock files.

Do it in small PRs (e.g., “network stack”, “androidx”). Easier to review.

---

**Q13: How do I name aliases without future regret?**
**A:** Use product-ish names, not artifact IDs:

* ✅ `retrofit`, `okhttp.logging`, `room.runtime`
* ❌ `com-squareup-retrofit2-retrofit`

Make bundles for “feature sets” (`androidx-ui`, `testing`).

---

**Q14: Will lock files bloat my repo?**
**A:** They’re text, pretty small, and super diff-friendly. The stability you get is worth the handful of files under `gradle/dependency-locks/`.

---

**Q15: CI tips—how do I stop “it changed on CI”?**
**A:**

* Fail the build if lock files aren’t up to date:

  * Run `./gradlew --write-locks` in a CI check and ensure no diffs.
* Only allow lock changes in dedicated “dependency update” PRs.

---

**Q16: What’s the difference between lock files and dependency verification?**
**A:** Lock files freeze versions. **Dependency verification** freezes checksums to prevent tampered artifacts. You can use both: first lock versions, then write verification metadata for security.

---

**Q17: Can Renovate/Dependabot work with version catalogs?**
**A:** Yep. They can read and bump `libs.versions.toml`. Combine with a job that runs `--update-locks` so PRs include updated lockfiles.

---

**Q18: I updated a library in TOML but Gradle still resolves the old version. Why?**
**A:** You’re probably still on old locks. Run `./gradlew --update-locks group:name` or `--write-locks`. Also check for another alias or a BOM overriding it.

---

**Q19: Can I have comments in TOML?**
**A:** Yes—use `#` comments. Great for “why” notes next to tricky versions.

---

**Q20: Any quality-of-life flags I should know?**
**A:**

* **Typesafe accessors** for `libs.*` are generated automatically; use them for IDE autocomplete.
* Use bundles to cut boilerplate:

  ```toml
  [bundles]
  test = ["junit", "mockk", "truth"]
  ```

  ```kotlin
  testImplementation(libs.bundles.test)
  ```

---

**Q21: What if a transitive dependency breaks me but I don’t directly declare it?**
**A:** Add an explicit alias for it in TOML and force the version in your module, then update locks:

```toml
[libraries]
androidx-annotation = { module = "androidx.annotation:annotation", version = "1.9.0" }
```

```kotlin
implementation(libs.androidx.annotation)
```

```bash
./gradlew --write-locks
```

---

**Q22: How do I roll back a bad update fast?**
**A:** Revert the PR that changed TOML and lock files. Builds go back to the previously locked graph. That’s the beauty of locking.
