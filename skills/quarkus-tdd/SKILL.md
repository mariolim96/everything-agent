---
name: quarkus-tdd
description: Test-driven development for Quarkus using JUnit 5, Mockito, REST Assured, Testcontainers, and JaCoCo. Use when adding features, fixing bugs, or refactoring.
origin: ECC
---

# Quarkus TDD Workflow

TDD guidance for Quarkus services with 80%+ coverage (unit + integration).

## When to Use

- New features or REST endpoints
- Bug fixes or refactors
- Data access logic or security rules

## Workflow

1. Write test first (it should FAIL)
2. Implement minimal code to pass (GREEN)
3. Refactor with tests still green (IMPROVE)
4. Enforce coverage threshold (JaCoCo 80%+)

## Unit Tests with Mocking

```java
@QuarkusTest
class MarketServiceTest {

  @InjectMock MarketRepository repo;
  @Inject MarketService service;

  @Test
  void createsMarket() {
    var request = new CreateMarketRequest("Test", "Description", Instant.now(), List.of("tech"));
    var market = new Market();
    market.id = 1L;
    market.name = request.name();

    Mockito.when(repo.persistAndFlush(any())).thenReturn(market);

    var result = service.create(request);

    assertThat(result.id()).isEqualTo(1L);
    assertThat(result.name()).isEqualTo("Test");
    verify(repo).persistAndFlush(any());
  }

  @Test
  void findNonExistent_returnsEmpty() {
    Mockito.when(repo.findByIdOptional(999L)).thenReturn(Optional.empty());

    var result = service.findById(999L);

    assertThat(result).isEmpty();
  }

  @ParameterizedTest
  @ValueSource(strings = {"", "  ", "a".repeat(201)})
  void createMarket_invalidName_throwsException(String name) {
    var request = new CreateMarketRequest(name, "Description", Instant.now(), List.of("tech"));

    assertThatThrownBy(() -> service.create(request))
        .isInstanceOf(ConstraintViolationException.class);
  }
}
```

Patterns:
- Arrange-Act-Assert (AAA)
- Mock external dependencies only
- Use `@InjectMock` for injected mocks
- Avoid partial mocks; prefer explicit stubbing
- Use `@ParameterizedTest` for variants

## Web Layer Tests with REST Assured

```java
@QuarkusTest
class MarketResourceTest {

  @Test
  void listMarkets_noParams_returns200() {
    given()
      .when()
        .get("/api/markets")
      .then()
        .statusCode(200)
        .body("size()", greaterThanOrEqualTo(0));
  }

  @Test
  void createMarket_validRequest_returns201() {
    given()
        .contentType(ContentType.JSON)
        .body("""
          {
            "name": "Tech Market",
            "description": "Tech discussion",
            "endDate": "2030-01-01T00:00:00Z",
            "categories": ["technology"]
          }
        """)
      .when()
        .post("/api/markets")
      .then()
        .statusCode(201)
        .body("name", equalTo("Tech Market"));
  }

  @Test
  void createMarket_missingFields_returns400() {
    given()
        .contentType(ContentType.JSON)
        .body("""
          {
            "name": "Incomplete"
          }
        """)
      .when()
        .post("/api/markets")
      .then()
        .statusCode(400);
  }

  @Test
  void getMarket_byId_returns200() {
    // Create a market first
    Long marketId = createTestMarket(1L, "Test");

    given()
      .when()
        .get("/api/markets/{id}", marketId)
      .then()
        .statusCode(200)
        .body("name", equalTo("Test"));
  }

  @Test
  void deleteMarket_authorized_returns204() {
    Long marketId = createTestMarket(1L, "To Delete");

    given()
        .header("Authorization", "Bearer " + createToken("admin"))
      .when()
        .delete("/api/markets/{id}", marketId)
      .then()
        .statusCode(204);
  }

  @Test
  void deleteMarket_unauthorized_returns403() {
    Long marketId = createTestMarket(1L, "Protected");

    given()
      .when()
        .delete("/api/markets/{id}", marketId)
      .then()
        .statusCode(403);
  }
}
```

## Integration Tests with Database

```java
@QuarkusTest
@QuarkusTestResource(PostgresTestResource.class)
@ActiveProfiles("test")
class MarketIntegrationTest {

  @Inject MarketRepository marketRepository;
  @Inject MarketService marketService;

  @BeforeEach
  void setUp() {
    // Clear data between tests
    Market.deleteAll();
  }

  @Test
  void createAndFindMarket() {
    var request = new CreateMarketRequest(
        "Integration Test", "Testing", Instant.now(), List.of("test"));

    var created = marketService.create(request);
    var found = marketService.findById(created.id());

    assertThat(found).isPresent();
    assertThat(found.get().name()).isEqualTo("Integration Test");
  }

  @Test
  void persistsMultipleMarkets() {
    marketService.create(new CreateMarketRequest("Market 1", "Desc", Instant.now(), List.of("a")));
    marketService.create(new CreateMarketRequest("Market 2", "Desc", Instant.now(), List.of("b")));

    var all = Market.listAll();

    assertThat(all).hasSize(2);
  }

  @Test
  void updateMarket_changesData() {
    var created = marketService.create(
        new CreateMarketRequest("Original", "Desc", Instant.now(), List.of("tag")));

    var market = marketRepository.findByIdOptional(created.id()).get();
    market.name = "Updated";
    market.persist();

    var updated = marketRepository.findByIdOptional(created.id()).get();
    assertThat(updated.name).isEqualTo("Updated");
  }
}
```

## Persistence Tests with Panache

```java
@QuarkusTest
@QuarkusTestResource(PostgresTestResource.class)
class MarketRepositoryTest {

  @Inject MarketRepository repo;

  @BeforeEach
  void setUp() {
    Market.deleteAll();
  }

  @Test
  void savesAndFinds() {
    var market = new Market();
    market.name = "Test";
    market.persist();

    var found = Market.find("name", "Test").firstResultOptional();

    assertThat(found).isPresent();
    assertThat(found.get().name).isEqualTo("Test");
  }

  @Test
  void findByStatusQuery() {
    createMarket("Active", MarketStatus.ACTIVE);
    createMarket("Closed", MarketStatus.CLOSED);

    var active = Market.find("status = ?1", MarketStatus.ACTIVE).list();

    assertThat(active).hasSize(1);
    assertThat(active.get(0).name).isEqualTo("Active");
  }

  @Test
  void pagination() {
    for (int i = 0; i < 25; i++) {
      createMarket("Market " + i, MarketStatus.ACTIVE);
    }

    var page1 = Market.find("status", MarketStatus.ACTIVE)
        .page(Page.of(0, 10)).list();
    var page2 = Market.find("status", MarketStatus.ACTIVE)
        .page(Page.of(1, 10)).list();

    assertThat(page1).hasSize(10);
    assertThat(page2).hasSize(10);
  }

  private void createMarket(String name, MarketStatus status) {
    var market = new Market();
    market.name = name;
    market.status = status;
    market.persist();
  }
}
```

## Coverage (JaCoCo)

Maven configuration in pom.xml:
```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.14</version>
  <executions>
    <execution>
      <id>prepare-agent</id>
      <goals><goal>prepare-agent</goal></goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>test</phase>
      <goals><goal>report</goal></goals>
    </execution>
    <execution>
      <id>jacoco-check</id>
      <phase>test</phase>
      <goals><goal>check</goal></goals>
      <configuration>
        <rules>
          <rule>
            <element>PACKAGE</element>
            <includes>
              <include>com.example.*</include>
            </includes>
            <limits>
              <limit>
                <counter>LINE</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.80</minimum>
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

Run coverage:
```bash
./mvnw clean test jacoco:report
# View: target/site/jacoco/index.html
```

## Assertions

Use AssertJ for readable assertions:
```java
assertThat(result)
    .isNotNull()
    .hasFieldOrPropertyWithValue("name", "Test");

assertThat(markets)
    .hasSize(5)
    .extracting(Market::getName)
    .contains("Tech", "Finance");

assertThatThrownBy(() -> service.create(null))
    .isInstanceOf(ConstraintViolationException.class)
    .hasMessageContaining("must not be null");
```

## Test Resource Management

```java
public class PostgresTestResource implements QuarkusTestResourceLifecycleManager {
  private PostgreSQLContainer<?> postgres;

  @Override
  public Map<String, String> start() {
    postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("postgres")
        .withPassword("postgres");
    postgres.start();

    return Map.ofEntries(
        Map.entry("quarkus.datasource.jdbc.url", postgres.getJdbcUrl()),
        Map.entry("quarkus.datasource.username", postgres.getUsername()),
        Map.entry("quarkus.datasource.password", postgres.getPassword()),
        Map.entry("quarkus.datasource.db-kind", "postgresql"));
  }

  @Override
  public void stop() {
    if (postgres != null) {
      postgres.stop();
    }
  }
}
```

## CI Commands

Maven:
```bash
./mvnw clean test
./mvnw verify  # includes integration tests
./mvnw clean verify -DskipITs  # skip integration tests
```

Gradle:
```bash
./gradlew test
./gradlew integrationTest
./gradlew jacocoTestReport
```

GitHub Actions example:
```yaml
- name: Run tests
  run: ./mvnw clean verify

- name: Upload coverage
  uses: codecov/codecov-action@v3
  with:
    files: ./target/site/jacoco/jacoco.xml
```

**Remember**: Keep tests fast (mock external calls), isolated (no shared state), and deterministic. Focus on behavior, not implementation details.
