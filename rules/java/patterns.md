---
paths:
  - '**/*.java'
---

# Java Patterns

> This file extends [common/patterns.md](../common/patterns.md) with Java and Quarkus specific content.

## Quarkus Panache (Data Access)

- Prefer **Panache Entity** (Active Record) for simple CRUD.
- Use **Panache Repository** for complex domain logic or when separation is required.

```java
// Active Record
@Entity
public class Person extends PanacheEntity {
    public String name;
    public LocalDate birth;
}

// Usage
List<Person> allPeople = Person.listAll();
```

## Reactive Programming (Mutiny)

- Use **SmallRye Mutiny** for reactive APIs.
- Prefer `Uni` for single results and `Multi` for streams.
- Ensure non-blocking execution in reactive routes.

## DTOs and Mapping

- Use **Records** for Data Transfer Objects.
- Use **MapStruct** for efficient, type-safe mapping between entities and DTOs.

## Reference

See skills: `jpa-patterns` and `springboot-patterns` for broader architectural patterns applicable to Java projects.
