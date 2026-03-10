---
paths:
  - '**/*.java'
---

# Java Coding Style

> This file extends [common/coding-style.md](../common/coding-style.md) with Java and Quarkus specific content.

## Standards

- Follow **Google Java Style Guide** or project-specific conventions.
- Use **Java 21+** features (Records, Sealed Classes, Switch Expressions).
- Use **Lombok** only if explicitly required by the project; prefer native Records.

## Quarkus & CDI

- Use **ArC (Quarkus CDI)** for dependency injection.
- Prefer constructor injection over field injection (`@Inject`).
- Use scope annotations correctly (`@ApplicationScoped`, `@RequestScoped`, `@Singleton`).

## Immutability

Prefer immutable data structures and native Records:

```java
public record User(String name, String email) {}

public record Point(double x, double y) {}
```

## Formatting

- Use **Spotless** or **Google Java Format** for consistent styling.
- Ensure proper import ordering (static imports first, then grouped by package).

## Reference

See skill: `java-coding-standards` for comprehensive Java idioms and Quarkus-specific ArC patterns.
