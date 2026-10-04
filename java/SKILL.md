---
name: java
description: Java coding conventions. Use when writing, generating or reviewing any Java code, including choosing the JDK version and build tool, designing DTOs and return types, wiring dependencies, and adding Bean Validation.
---

## General

- Use the latest LTS release for all new code.
- Maven is the preferred build tool.
- Use records as DTOs everywhere possible.
- Return `Optional` for any nullable return type.
- Always use constructor injection, never field injection with `@Inject`:
```java
private final CompanyRepository companyRepository;

public CompanyService(CompanyRepository companyRepository) {
    this.companyRepository = companyRepository;
}
```

## Validation

- Use Hibernate Validator. Put `@Valid` on all appropriate service layer and repository layer method signatures.
- Store validation messages in `src/main/resources/ValidationMessages.properties`, and reference them from the constraint annotations instead of hard-coding message text.
