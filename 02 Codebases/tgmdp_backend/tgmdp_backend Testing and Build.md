---
tags: [codebase, tgmdp-backend]
---
# tgmdp_backend Testing and Build

Up: [[tgmdp_backend]] · [[System Overview]]

## The test suite at a glance

`src/test/java` holds about **880** test files, mirroring the main package tree. Rough counts:

| Kind | Marker | Files | Needs DB / ESB? |
|---|---|---|---|
| Pure unit tests | `@ExtendWith(MockitoExtension.class)` | ~455 | no |
| Web-slice tests | `@WebMvcTest(XxxController.class)` | ~190 | no |
| Full-context tests | `@SpringBootTest` | ~48 | some |
| Integration | `@Tag("integration")` | few | yes (DB2/ESB) |
| Live ESB | `@Tag("esbIntegration")` | few | yes (ESB) |

This is a healthy [[Testing Pyramid]]: many fast unit tests at the bottom, fewer slice tests, and a thin top of tests that need real infrastructure. The default `test` task **excludes** the slow tags:

```groovy
// build.gradle
test {
    useJUnitPlatform { excludeTags 'integration', 'esbIntegration' }
    maxParallelForks = 4
    environment 'SECRETS_DIR', "${projectDir}/src/test/resources/jasypt/secrets/"
    finalizedBy jacocoTestReport
}
tasks.register('integrationTest', Test) { useJUnitPlatform { includeTags 'integration' } ... }
tasks.register('esbIntegrationTest', Test) { useJUnitPlatform { includeTags 'esbIntegration' } ... }
```

Why tag instead of separate folders? The test sits next to the code it covers, and one annotation moves it between "every commit" and "only when infra is available".

## Layer 1: service unit tests with Mockito

`src/test/java/com/taiwanmobile/boilerplate/service/tgmdpvendor/TgmdpvendorAuthServiceTest.java:320-332`

```java
@Test
void should_throwAccountLockedMes4_when_accountStatusIsLocked() {
    AcptAccountMst account = activeAccount();
    account.setStatus("1");
    when(captchaRepository.findValidCaptchaByToken("tok")).thenReturn(Optional.of(validCaptcha()));
    when(accountRepository.findByUserIdIgnoreCase("M001-001")).thenReturn(Optional.of(account));

    assertThatThrownBy(() -> authService.sendOtp(sendOtpRequest(), "10.0.0.1"))
            .isInstanceOf(TgmdpvendorAuthAccountLockedException.class)
            .hasMessage("帳號鎖定！");

    verify(loginLogRepository).logLoginAttempt(anyLong(), eq("M001-001"), eq("1"), eq("4"), eq("1"), anyString(), eq("WEB"));
}
```

What to learn from it:
- **Naming convention** `should_<outcome>_when_<condition>`. It is mandated by `tgmdp_backend/CLAUDE.md` and reads like a spec. See [[Clean Code Naming]].
- **Arrange / Act / Assert** with Mockito stubs (`when(...).thenReturn(...)`) and AssertJ assertions.
- **Verify side effects**, not just return values. The audit row with `mes=4` is part of the legacy contract, so the test checks it was written.
- **Encode decisions in tests.** The next test, `should_throwAccountLockedMes4_when_accountStatusIsDeleted_bugForBug` (`:334-347`), pins a deliberate legacy bug in place, with the legacy line numbers in a comment. A future "fix" would break the test and force a conversation. Tests are executable documentation of *rulings*.

Repositories are mocked at their boundary, so these tests never touch DB2. See [[Unit Testing]] and [[Mocking]].

## Layer 1b: repository tests that mock JDBC

`repository/tgmdpweb/projectAcceptance/PoCategoryBrowseRepositoryImplTest.java` mocks `NamedParameterJdbcTemplate`, then checks paging maths and that the right parameters are bound. It **cannot** prove the SQL is valid DB2. That gap is why DB2 bugs (`-302`, `-420`, `-440`) showed up only against a live database. The team's workaround is a tiny standalone Java program with the DB2 JCC jar that runs one query against the real DB, without booting Spring. Know what each test layer *can't* catch.

## Layer 2: controller slice tests with MockMvc

`controller/tgmdpweb/projectAcceptance/PoCategoryControllerTest.java`

```java
@ActiveProfiles("test")
@WebMvcTest(PoCategoryController.class)
class PoCategoryControllerTest {
    @Autowired private MockMvc mockMvc;
    @MockitoBean private PoCategoryService service;
    @MockitoBean private JwtUtility jwtUtility;          // security filter dependencies
    @MockitoBean private JwtService jwtService;
    @MockitoBean private BasicAuthService basicAuthService;
    ...
    mockMvc.perform(get("/api/v1/tgmdpweb/project-acceptance/po-category/init")
                    .with(user("testUser").authorities(new SimpleGrantedAuthority("TASK_TGMDPs011:main"))))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.code").value(200));
```

`@WebMvcTest` starts only the web layer: controllers, filters, advice, Jackson. It does **not** start JPA or real services. The service is a `@MockitoBean`. `spring-security-test`'s `user(...).authorities(...)` fakes an authenticated principal. The test proves URL mapping, argument binding, JSON shape and status codes.

Why all the security mocks? `AuthTokenFilter` is a `@Component` filter, so the slice picks it up, and its dependencies must exist. This is the price of filters with many collaborators.

> **Investigate this:** the `@PreAuthorize` on `/init` requires `TASK_TGMDPs011:main` **and** `SYS_工程驗收`, but the test grants only the first and expects 200. Either method security isn't active in this slice, or something else grants the authority. Run the test and find out. Whichever it is, it tells you what this test does *not* cover. (I did not run it.)

## Layer 3: integration tests

`integration/BaseIntegrationTest.java`

```java
@Tag("esbIntegration")
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles({"local", "integration"})
public abstract class BaseIntegrationTest { @LocalServerPort protected int appPort; ... }
```

A real app on a random port, called over HTTP with REST Assured, against a real ESB. Other tools in `build.gradle`: Testcontainers (containerised dependencies), CXF local transport (in-memory SOAP endpoints), Awaitility (poll until an async callback lands), H2 (in-memory DB for an `e2e` profile).

SOAP services have one notable trick, from `CLAUDE.md`: tests mock `SoapClientUtility` and use Mockito's `MockedConstruction` to intercept `new XxxPortTypeService()`. If you change which `SoapClientUtility` method a service calls, update the stubs, or the test NPEs with "port is null".

## Build tooling: Gradle

```bash
./gradlew clean build -x test     # what CI's build job runs
./gradlew test                    # unit tests + JaCoCo report
./gradlew -Pfast test             # skip Checkstyle/PMD/SpotBugs (pre-commit)
./gradlew integrationTest         # loads .env, needs DB/ESB
./gradlew wsdl2java               # regenerate SOAP clients
```

Notable build pieces in `build.gradle`:

- **Multi-project:** `settings.gradle` includes `sharedModules`, a git submodule used as a Gradle subproject. The `Dockerfile` must `COPY sharedModules` before resolving dependencies; its comment explains the "No variants exist" error you get otherwise.
- **Code generation:** the `wsdl2java` task runs before `compileJava` (see [[tgmdp_backend ESB Integration]]).
- **`-Pfast` switch** disables the static-analysis tasks with a property. This is a build-time feature flag.
- **Static analysis:** Checkstyle (`config/checkstyle/checkstyle.xml`, `ignoreFailures = false`), PMD (`config/pmd/ruleset.xml`, `ignoreFailures = false`), SpotBugs (`ignoreFailures = true`, report only), SonarQube (token from env, read through `safeGetenv`). See [[Linting and Formatting]].
- **Coverage:** JaCoCo runs after `test` and produces a report. I found **no** `jacocoTestCoverageVerification` rule, so coverage is reported but not enforced. Compare the frontends' gates in [[Test Coverage Gates]].
- **Dependency hygiene:** the `ext {}` block and `resolutionStrategy.force` pin versions for named CVEs. The comments explain a subtle trap: for libraries managed by the Spring Boot BOM, `force` is silently overridden, so you must set the BOM property instead (e.g. `set('tomcat.version', '11.0.27')`). See [[Build Tools and Dependency Management]].
- **Jasypt encrypt task:** `jasyptEncrypt` reads plaintext from stdin, so it never lands in shell history.

## CI/CD and local gates

`.gitlab-ci.yml` stages: `build → test → deploy → lint → quality`. Build jobs are `when: manual` per environment (`-Penv=${ENV_NAME}`) and archive the jar for a week. `tgmdp_backend/CLAUDE.md` adds that a separate Jenkins pipeline builds for UAT, and that the jar is never committed. Note one oddity: the CI test template starts a `postgres` service, which the DB2-based app doesn't use. It looks like a leftover from the boilerplate.

`.githooks/pre-commit` runs on the developer's machine:
1. `git diff --cached --check` (whitespace, conflict markers),
2. a regex **secret scan** over staged lines (`api_key|secret|password|token|jwt` followed by 8+ characters),
3. `./gradlew -Pfast test --quiet` when Java/Gradle/config files are staged.

The secret scan is heuristic and has false positives (`this.password = password;`). `CLAUDE.md` asks you to rename locals rather than bypass with `--no-verify`. A gate that people habitually skip protects nothing. See [[Secrets Management]] and [[Code Review]].

Also available: `Makefile` (`make run PROFILE=local`, `make debug` on port 5005, docker targets), `docker-compose.yml`, and a multi-stage `Dockerfile` on `eclipse-temurin:17`.

## Try it yourself

1. Run `./gradlew -Pfast test --tests '*PoCategoryControllerTest'` and answer the "Investigate this" question above.
2. Write a new test `should_lockAccount_when_thirdWrongOtpSubmitted` for `TgmdpvendorAuthService.verifyOtp`. Which repository call proves the lock happened?
3. Add a `jacocoTestCoverageVerification` rule with a minimum of 0.6 in a scratch branch. Does the build still pass?

## Questions to check understanding

- What does `@WebMvcTest` load, and what does it leave out? Why does that make it faster than `@SpringBootTest`?
- Why can a green repository test still hide a DB2 `-302` bug?
- Why exclude `integration` tests from the default `test` task instead of deleting them from CI?

## Related

- [[tgmdp_backend]] · [[System Overview]] · [[tgmdp_backend Persistence and DB2]] · [[tgmdp_backend ESB Integration]] · [[tgmdp_backend Logging and Configuration]]
- [[Testing Pyramid]] · [[Unit Testing]] · [[Mocking]] · [[Test Coverage Gates]] · [[Linting and Formatting]] · [[Code Review]]
- [[Build Tools and Dependency Management]] · [[Git Branching]] · [[Secrets Management]] · [[Clean Code Naming]] · [[Monorepo vs Polyrepo]] · [[Feature Flags and Mock Toggles]]
