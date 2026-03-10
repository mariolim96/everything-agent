---
paths:
  - '**/*.java'
---

# Java Hooks

> This file extends [common/hooks.md](../common/hooks.md) with Java specific content.

## PostToolUse Hooks

- **Spotless/Google Java Format**: Auto-format `.java` files after edit if configured.
- **Checkstyle**: Run static analysis after editing Java files.

## Warnings

- Warn about field injection (`@Inject` on fields) — suggest constructor injection.
- Warn about missing `@Valid` on controller/resource parameters.
- Warn about usage of `System.out.println` — suggest using `org.jboss.logging.Logger` or SLF4J.
