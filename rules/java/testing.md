---
paths:
  - '**/*.java'
---

# Java Testing

> This file extends [common/testing.md](../common/testing.md) with Java and Quarkus specific content.

## Framework

- Use **JUnit 5** (Jupiter) as the primary testing framework.
- Use **RestAssured** for API/Endpoint testing.

## Quarkus Testing

- Use **@QuarkusTest** for integration tests that require the Quarkus container.
- Use **@InjectMock** (from `quarkus-junit5-mockito`) to mock CDI beans in integration tests.
- Use **@TestProfile** for environment-specific test configurations.

## Assertions

Use **AssertJ** for fluent, readable assertions:

```java
assertThat(user.getName()).isEqualTo("Mario");
assertThat(response.getStatusCode()).isEqualTo(200);
```

## Coverage

- Use **Jacoco** for code coverage reporting.
- Aim for 80%+ branch coverage.

## Reference

See skill: `springboot-tdd` (patterns apply to Quarkus) and Quarkus guide for specialized testing patterns.
