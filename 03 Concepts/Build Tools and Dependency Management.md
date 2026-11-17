---
tags: [concept]
---
# Build Tools and Dependency Management

A build tool turns source into something runnable: it resolves dependencies, compiles, runs tests and packages the result. Examples are Gradle or Maven (and the older Ant) for Java, and npm with Vite or Metro for JavaScript. A lockfile or BOM pins exact dependency versions so every machine builds the same thing.

Transitive dependencies and version conflicts are the usual pain points.

## In this codebase
- **Gradle multi-project with a BOM.** `tgmdp_backend/build.gradle:1-10` applies the Spring Boot plugin (`4.1.1`) plus `io.spring.dependency-management`, JaCoCo, SonarQube, Checkstyle, PMD and SpotBugs; Java 17 toolchain (`:68`). `settings.gradle` includes `sharedModules`, which `build.gradle:215` consumes as `implementation project(':sharedModules')`. The comment at `:49-55` explains a real version-conflict lesson: the subproject must track the root's Boot version or it compiles against older Spring/Spring Data than runtime resolves.
- **Comments record why a pin exists.** `build.gradle` has notes that re-pinning `commons-lang3`, `httpclient5`, `log4j2` would downgrade what Boot 4's BOM already ships, and a netty and Tomcat patch overrides in an `ext` block, each with its CVE reason (`:27-46`). Reading them is faster than diffing versions.
- **Duplicate declarations.** `com.ibm.db2:jcc` is declared as `runtimeOnly 11.5.7.0` (`:231`) and `implementation 12.1.0.0` (`:244`); Gradle's conflict resolution picks the higher version. Easy to miss when upgrading the driver.
- **Tests as a build concern.** The `test` task excludes `integration`/`esbIntegration` tags, runs `maxParallelForks = 4`, sets a test `SECRETS_DIR`, and finalizes with `jacocoTestReport` (`:94-115`); separate tasks run the slow tags. `-Pfast` disables static analysis (`:13-18`).
- **tgmdp_jobs** is Gradle too, but also pulls vendored jars from a folder: `implementation fileTree(dir: "lib/", include: '*.jar', exclude: 'ojdbc8.jar')` (`tgmdp_jobs/build.gradle:89`) and `files('lib/MDPWebService.jar')` (`:215`); a comment says `ojdbc8.jar` was moved to a real versioned dependency. Jars in a folder have no declared version or transitive resolution.
- **Legacy: no dependency management at all.** `legacy/ACPT_ADS/build.xml:21-34` is an Ant file with `init`/`compile` targets and `source/target 1.8`; the Eclipse `legacy/TGMDP_legacy_new/.classpath:6-19` lists jars by absolute workstation path (`D:/TGMDP_JAR/commons-lang-2.6.jar`), and `legacy/TGMDP/TGMDP/WEB-INF/lib` holds ~81 checked-in jars. Reproducing a legacy build means finding those jars first.

## Why it matters
A reproducible build is the base for everything else. The legacy-to-Gradle jump is a large part of what "modernisation" meant here: versions became explicit, security patches became a one-line change.

## Check yourself
Why does `sharedModules` need `apply plugin: 'io.spring.dependency-management'` in `subprojects { }`?

## Related
- [[Configuration and Environments]]
- [[Linting and Formatting]]
- [[Test Coverage Gates]]
- [[Git Branching]]
- [[tgmdp_backend Testing and Build]]
- [[Vendor Portal Build Lint and Testing]]
- [[RN Shell Build and Environments]]
- [[tgmdp_rn_app]]
- [[tgmdp_fake_esb]]
- [[Concepts MOC]]
- [[System Overview]]
