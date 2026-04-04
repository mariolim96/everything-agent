# Spring Boot API Project Configuration (ECC)

This is a **localized rule set** for a Spring Boot 3+ API, designed to be used with Google Antigravity or Claude Code.

## Development Commands

- **Initialize**: `spring start -d web,data-jpa,security,actuator`
- **Run**: `./mvnw spring-boot:run` or `./gradlew bootRun`
- **Compile**: `./mvnw compile`
- **Test**: `./mvnw test`
- **Verify**: `./mvnw verify`
- **Build**: `./mvnw package`
- **Native Image**: `./mvnw native:compile -Pnative`

## Project Architecture

- **Controller**: `@RestController`, `@RequestMapping`
- **Service**: Business logic with `@Service` and `@Transactional`
- **Repository**: Spring Data JPA (`JpaRepository<T, ID>`)
- **Entity**: Standard JPA annotations (`@Entity`, `@Table`)
- **Messaging**: Spring Cloud Stream / Kafka (optional)

## Design Patterns & Rules

### I. Layered Architecture
- **Business Logic**: MUST reside in the Service layer. Controllers should only handle requests and responses.
- **DTOs**: Use **Records** for DTOs to ensure immutability.

### II. Dependency Injection
- **Constructor Injection**: MUST use constructor injection for all beans.
- **Bean Scopes**: Prefer default `@Service` (Singleton) for logic and `@RequestScope` for user-specific context.

### III. Data Access (JPA)
- **Transactions**: MUST have `@Transactional` on service methods that modify state.
- **Efficiency**: Use `@Query` with `JOIN FETCH` to prevent N+1 query problems.

### IV. Observability (Actuator)
- **Endpoints**: Enable `/actuator/health` and `/actuator/metrics`.
- **Custom Metrics**: Use `MeterRegistry` from Micrometer for custom business metrics.

### V. Security (OAuth2/JWT)
- **Web Security Configuration**: Define a `SecurityFilterChain` bean.
- **Annotations**: Use `@PreAuthorize` for method-level security.

## Build Triggers

> [!TIP]
> After modifying `pom.xml`, run `./mvnw dependency:tree` to check for conflicts.
> After modifying `application.properties` or `application.yml`, run `./mvnw spring-boot:run` to reload context.

## References
- Agent: `java-reviewer`
- Skills: `springboot-patterns`, `springboot-security`, `springboot-tdd`.
