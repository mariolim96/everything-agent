---
name: quarkus-verification
description: "Verification loop for Quarkus projects: build, static analysis, tests with coverage, security scans, and native image checks before release or PR."
origin: ECC
---

# Quarkus Verification Loop

Run before PRs, after major changes, and pre-deploy. Includes native image build verification.

## When to Activate

- Before opening a pull request for a Quarkus service
- After major refactoring or dependency upgrades
- Pre-deployment verification for staging or production
- Running full build → lint → test → security scan → native compile pipeline
- Validating test coverage meets thresholds

## Phase 1: Build

Maven:
```bash
./mvnw -T 4 clean verify -DskipTests
```

Gradle:
```bash
./gradlew clean build -x test
```

If build fails, stop and fix before proceeding.

## Phase 2: Static Analysis

Maven plugins (add to pom.xml):
```bash
./mvnw -T 4 spotbugs:check pmd:check checkstyle:check
```

Gradle (if configured):
```bash
./gradlew checkstyleMain pmdMain spotbugsMain
```

FindBugs example configuration in pom.xml:
```xml
<plugin>
  <groupId>com.github.spotbugs</groupId>
  <artifactId>spotbugs-maven-plugin</artifactId>
  <version>4.7.3</version>
  <executions>
    <execution>
      <goals><goal>check</goal></goals>
    </execution>
  </executions>
</plugin>
```

## Phase 3: Tests + Coverage

Unit and integration tests:
```bash
./mvnw -T 4 test
./mvnw jacoco:report
```

Or with Gradle:
```bash
./gradlew test jacocoTestReport
```

Verify coverage meets 80%+ threshold:
```bash
# Check JaCoCo report
cat target/site/jacoco/index.html | grep -o "Total.*[0-9]\+%"
```

### Unit Tests with Quarkus Testing

```java
@QuarkusTest
class UserServiceTest {

  @Inject UserRepository userRepository;
  @Inject UserService userService;

  @Test
  void createUser_validInput_returnsUser() {
    var request = new CreateUserRequest("Alice", "alice@example.com");
    var result = userService.create(request);

    assertThat(result.getName()).isEqualTo("Alice");
    assertThat(result.getEmail()).isEqualTo("alice@example.com");
  }

  @Test
  void createUser_duplicateEmail_throwsException() {
    userRepository.save(new User("alice@example.com", "password"));

    var request = new CreateUserRequest("Bob", "alice@example.com");
    assertThatThrownBy(() -> userService.create(request))
        .isInstanceOf(DuplicateEmailException.class);
  }
}
```

### Integration Tests with Testcontainers

```java
@QuarkusTest
@QuarkusTestResource(PostgresTestResource.class)
class UserRepositoryIntegrationTest {

  @Inject UserRepository userRepository;

  @Test
  void savesAndFinds() {
    var user = new User("Alice", "alice@example.com");
    userRepository.persist(user);

    var found = userRepository.find("email = ?1", "alice@example.com").firstResultOptional();

    assertThat(found).isPresent();
    assertThat(found.get().getName()).isEqualTo("Alice");
  }
}
```

TestResource configuration:
```java
public class PostgresTestResource implements QuarkusTestResourceLifecycleManager {

  private PostgreSQLContainer<?> postgres;

  @Override
  public Map<String, String> start() {
    postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    postgres.start();

    return Map.of(
        "quarkus.datasource.jdbc.url", postgres.getJdbcUrl(),
        "quarkus.datasource.username", postgres.getUsername(),
        "quarkus.datasource.password", postgres.getPassword());
  }

  @Override
  public void stop() {
    if (postgres != null) {
      postgres.stop();
    }
  }
}
```

### API Tests with REST Assured

Test REST endpoints:
```java
@QuarkusTest
class UserResourceTest {

  @Test
  void createUser_validInput_returns201() {
    given()
        .contentType(ContentType.JSON)
        .body(new CreateUserRequest("Alice", "alice@example.com"))
      .when()
        .post("/api/users")
      .then()
        .statusCode(201)
        .body("name", equalTo("Alice"))
        .body("email", equalTo("alice@example.com"));
  }

  @Test
  void createUser_invalidEmail_returns400() {
    given()
        .contentType(ContentType.JSON)
        .body(new CreateUserRequest("Alice", "not-an-email"))
      .when()
        .post("/api/users")
      .then()
        .statusCode(400);
  }

  @Test
  void getUser_existingUser_returns200() {
    User created = createUser("Alice", "alice@example.com");

    given()
      .when()
        .get("/api/users/{id}", created.getId())
      .then()
        .statusCode(200)
        .body("name", equalTo("Alice"));
  }
}
```

## Phase 4: Security Scan

Dependency CVEs:
```bash
./mvnw org.owasp:dependency-check-maven:check
# or
./gradlew dependencyCheckAnalyze
```

Search for hardcoded secrets:
```bash
grep -rn "password\s*=\s*\"" src/ --include="*.java" --include="*.yml" --include="*.properties"
grep -rn "sk-\|api_key\|secret" src/ --include="*.java" --include="*.yml"
```

Check for common security issues:
```bash
# System.out.println instead of logging
grep -rn "System\.out\.print" src/main/ --include="*.java"

# Raw exception messages
grep -rn "\.getMessage()" src/main/ --include="*.java"

# Wildcard CORS
grep -rn "allowedOrigins.*\*" src/main/ --include="*.java"
```

## Phase 5: Native Image Build (Optional but Recommended)

Build native executable:
```bash
./mvnw clean package -Pnative -DskipTests
# Takes ~2-5 minutes on first build
```

Verify native binary:
```bash
./target/quarkus-app-runner --version
./target/quarkus-app-runner  # starts the app
```

Check size and startup time:
```bash
ls -lh target/quarkus-app-runner
# Compare JVM vs native startup time
time ./target/quarkus-app-runner
```

## Phase 6: Dev Mode Live Testing (Optional)

Hot reload with tests:
```bash
./mvnw quarkus:dev
# In another terminal:
./mvnw test -pl . -am
```

## Phase 7: Lint/Format (Optional Gate)

Apply code formatting:
```bash
./mvnw spotless:apply
# or
./gradlew spotlessApply
```

## Phase 8: Diff Review

Review all changes:
```bash
git diff --stat
git diff
```

Checklist:
- No debugging logs left (`System.out`, `log.debug` without guards)
- Meaningful error messages and HTTP statuses
- Transactions present where needed
- Validation applied to all inputs
- Config changes documented
- Dependencies updated consistent with version pins

## Output Template

```
VERIFICATION REPORT
===================
Build:       [PASS/FAIL]
Static:      [PASS/FAIL] (spotbugs/pmd/checkstyle)
Tests:       [PASS/FAIL] (X/Y passed, Z% coverage)
Security:    [PASS/FAIL] (CVE findings: N)
Native:      [PASS/FAIL] (size: XXMb, startup: XXXms)
Diff:        [X files changed]

Overall:     [READY / NOT READY]

Issues to Fix:
1. ...
2. ...
```

## Continuous Mode

For long sessions:
- Run `./mvnw -T 4 test` every 30 minutes
- Rebuild native image before deployment
- Watch for startup time regressions

Keep the gate strict: treat warnings as defects in production systems.
