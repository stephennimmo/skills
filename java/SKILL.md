---
name: java
description: Java coding conventions. Use when writing, generating or reviewing any Java code, including choosing the JDK version and build tool, designing DTOs and return types, wiring dependencies, and adding Bean Validation.
---

## General

- Use the latest LTS release for all new code
- Maven is the preferred build tool
- Records should be used as DTOs everywhere possible.
- Optional should be used for any nullable return types
- Don't use `@Inject`, but instead always use constructor injection. Example:
```java
private final CompanyRepository companyRepository;

public CompanyService(CompanyRepository companyRepository) {
    this.companyRepository = companyRepository;
}
```

## Validation

- Use Hibernate Validator. Put `@Valid` on all appropriate service layer and repository layer method signatures.
- Store validation messages in `src/main/resources/ValidationMessages.properties`, and reference them from the constraint annotations instead of hard-coding message text.
