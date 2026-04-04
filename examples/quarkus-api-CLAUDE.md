# Quarkus API Project Configuration (ECC)

This is a **localized rule set** for a Quarkus-native API, designed to be used with Google Antigravity or Claude Code.

## Development Commands

- **Initialize**: `mvn io.quarkus.platform:quarkus-maven-plugin:3.15.1:create`
- **Dev Mode**: `mvn quarkus:dev` (Hot reload enabled)
- **Compile**: `mvn compile`
- **Test**: `mvn test`
- **Verify**: `mvn verify` (includes integration tests)
- **Native Image**: `mvn verify -Pnative` (requires GraalVM)
- **Add Extension**: `mvn quarkus:add-extension -Dextensions="..."`

## Project Architecture

- **Resource (Controller)**: JAX-RS annotations (`@Path`, `@GET`, `@POST`)
- **Service**: Business logic with `@ApplicationScoped` and `@Transactional`
- **Repository**: Panache Patterns (`PanacheRepository<T>`)
- **Entity**: Panache Patterns (`PanacheEntity` or manual `@Entity`)
- **Integration**: Reactive with **Mutiny** (`Uni<T>`, `Multi<T>`)

## Design Patterns & Rules

### I. Reactive First
- **Blocking Calls**: NEVER use blocking calls on the event loop. Use `Uni` or `Multi`.
- **Worker Pools**: Use `@Blocking` for unavoidable synchronous calls.

### II. CDI & Injection
- **Constructor Injection**: MUST use constructor injection for all beans.
- **Scopes**: Prefer `@ApplicationScoped` for services and `@RequestScoped` for resources.

### III. Data Access (Panache)
- **Write Operations**: MUST have `@Transactional`.
- **Query Optimization**: Use `with` parameters for type-safe queries.
- **Sorting**: Use `Sort.by(...)` to prevent string-based vulnerabilities.

### IV. Observability & Health
- **Health Checks**: Implement `HealthCheck` for Readiness and Liveness.
- **Metrics**: Use `@Counted` or `@Timed` from MicroProfile Metrics.

### V. Security (OIDC/JWT)
- **Annotations**: Use `@RolesAllowed` on all sensitive resources.
- **Headers**: Ensure `quarkus-security` is configured with OIDC.

## Build Triggers

> [!TIP]
> After modifying `pom.xml`, run `mvn dependency:analyze` to verify dependency health.
> After modifying `application.properties`, run `mvn quarkus:dev` to reload context.

## References
- Agent: `quarkus-reviewer`
- Skills: `quarkus-patterns`, `quarkus-security`, `quarkus-tdd`.
