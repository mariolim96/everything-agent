---
paths:
  - '**/*.java'
---

# Java Security

> This file extends [common/security.md](../common/security.md) with Java and Quarkus specific content.

## Secret Management

- Use **Quarkus Config** with environment variables (`QUARKUS_DATASOURCE_PASSWORD`).
- Use **SmallRye Config** profiles for environment-specific secrets.
- Never hardcode credentials in `application.properties`.

## Security Framework

- Use **Quarkus Security** for OIDC, JWT, or Basic Auth.
- Use `@RolesAllowed`, `@Authenticated`, or `@PermitAll` to secure endpoints.

## Input Validation

- Use **Hibernate Validator** (Bean Validation API) for declarative validation:

```java
public record CreateUserRequest(
    @NotBlank String name,
    @Email String email
) {}
```

## Reference

See skill: `springboot-security` for common Java security pitfalls and mitigation strategies.
