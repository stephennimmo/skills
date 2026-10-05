---
name: quarkus
description: Quarkus project conventions. Use when creating or changing any Quarkus project, including extensions, application.yaml configuration and profiles, dev services, Panache entities, Flyway migrations and PostgreSQL schema, test data, tests, Protobuf/gRPC definitions, and Containerfiles.
---

Follow the `java` skill for general Java conventions (records, Optional, constructor injection, validation).
For REST endpoints and the Resource → Service → Repository layering, also follow the `quarkus-rest` skill.

## Project Basics

- API projects listen on port 8080.
- YAML files end in `.yaml`, not `.yml`.
- Use dev services whenever running locally. Don't hand-configure a local database or OIDC server.
- Absolutely no sensitive information (passwords, client secrets, tokens) in the git repository. Read them from environment variables in `%prod`.
- When code changes, update the README and any other documentation in the project.

## Extensions

Add extensions with Maven: `./mvnw quarkus:add-extension -Dextensions="<extension>"`.

| Extension                       | Purpose                                      |
| :---                            | :---                                         |
| `quarkus-config-yaml`           | `application.yaml` configuration             |
| `quarkus-hibernate-orm-panache` | Panache entities and repositories            |
| `quarkus-jdbc-postgresql`       | PostgreSQL driver and dev service            |
| `quarkus-flyway`                | Schema migrations                            |
| `quarkus-hibernate-validator`   | Bean Validation                              |
| `quarkus-oidc`                  | OIDC authentication and Keycloak dev service |

## Configuration

Configuration lives in `src/main/resources/application.yaml` and uses the `%dev`, `%test`, `%prod` profile structure.

- Turn off the banner.
- Leave datasource and OIDC server settings unset in `%dev` and `%test` so dev services start PostgreSQL and Keycloak automatically.
- Set real connection details only in `%prod`, read from environment variables.

```yaml
quarkus:
  banner:
    enabled: false
  flyway:
    migrate-at-start: true
    locations: db/migration

"%dev":
  quarkus:
    flyway:
      locations: db/migration,db/testdata

"%test":
  quarkus:
    flyway:
      locations: db/migration,db/testdata

"%prod":
  quarkus:
    datasource:
      jdbc:
        url: ${DB_URL}
      username: ${DB_USERNAME}
      password: ${DB_PASSWORD}
    oidc:
      auth-server-url: ${OIDC_AUTH_SERVER_URL}
      client-id: ${OIDC_CLIENT_ID}
```

## Panache Entities

Entities are accessed only through repositories (see Repository in the `quarkus-rest` skill).

- Suffix is `Entity`: `PersonEntity`, `CarEntity`.
- Always annotate the class with both `@Entity(name = "...")` and `@Table(name = "...")`. The entity name is the plain noun without the `Entity` suffix, and the table name is the exact table: `CustomerEntity` has `@Entity(name = "Customer")` and `@Table(name = "customer")`.
- The ID follows the entity name, never just `id`: `PersonEntity` has `personId`, mapped to the column `person_id`.
- Every field has `@Column` with `name` set, plus `nullable` matching the schema.
- Validate fields with Jakarta Validation annotations that reference messages in `ValidationMessages.properties`.
- Fields are public.
- Generate `equals`, `hashCode` and `toString`.

```java
@Entity(name = "Customer")
@Table(name = "customer")
public class CustomerEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "customer_id", nullable = false)
    public Integer customerId;

    @NotBlank(message = "{customer.name.required}")
    @Column(name = "name", nullable = false)
    public String name;

    // equals, hashCode, toString
}
```

## Database Schema (Flyway + PostgreSQL)

Use Flyway for all schema management. Migrations go in `src/main/resources/db/migration`.

- Table names are singular: `customer`, not `customers`. Exception: use `users` for a user table because `user` is a reserved word.
- The primary key is the entity root table name plus `_id`: `customer` has `customer_id`.
- Use `SERIAL` for primary keys and `INT` for foreign keys. Do not use UUID. Map these IDs to `Integer` in Java.
- Always restart the primary key sequence at `10000000` (8 digits).
- Use `TEXT` for alphanumeric data, not `VARCHAR`.
- Add `created_at` and `updated_at` where it makes sense.

```sql
CREATE TABLE customer (
    customer_id SERIAL PRIMARY KEY,
    name        TEXT      NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

ALTER SEQUENCE customer_customer_id_seq RESTART 10000000;
```

### Test data
- Put test data in `src/main/resources/db/testdata/V999__testdata.sql`.
- Use primary key values below the sequence start (for example `1`, `2`, `3`) so test rows are easy to find and delete.
- Only `%dev` and `%test` load `db/testdata`. `%prod` never does (see Configuration).

## Testing

Extensive tests are required, not optional.
- Use `@QuarkusTest`, backed by dev services and the `db/testdata` rows. Don't mock the database.
- For REST endpoint coverage, see Testing in the `quarkus-rest` skill.

## Protobuf and gRPC

- Use proto3.
- Every enum has `UNSPECIFIED` as its `0` value.
- Prefix every enum value with the enum name in upper snake case:
```protobuf
enum ConnectionType {
  CONNECTION_TYPE_UNSPECIFIED = 0;
  CONNECTION_TYPE_INITIATOR = 1;
  CONNECTION_TYPE_ACCEPTOR = 2;
}
```

## Containerfiles (Red Hat Hardened Images)

Assume Podman. Name the files `Containerfile`, never `Dockerfile`. Source all images from Red Hat registries (`registry.access.redhat.com/hi/...`, `registry.redhat.io`, `quay.io`).

- Put Containerfiles at the **project root**. Delete the generated files under `src/main/docker/`; the root Containerfiles are the source of truth.
- Red Hat Hardened Images are **distroless**: no shell, no package manager, no `run-java.sh`.
- The default runtime user is **UID 65532**. Always `COPY --chown=65532:65532` and end with `USER 65532`.
- Use exec-form `ENTRYPOINT` only.

### JVM: `Containerfile`
- Base: `registry.access.redhat.com/hi/openjdk:latest-runtime`
- Prerequisite: `./mvnw package` (fast-jar under `target/quarkus-app/`)
- Copy Quarkus layers separately so library layers stay cacheable:
```dockerfile
COPY --chown=65532:65532 target/quarkus-app/lib/ /deployments/lib/
COPY --chown=65532:65532 target/quarkus-app/*.jar /deployments/
COPY --chown=65532:65532 target/quarkus-app/app/ /deployments/app/
COPY --chown=65532:65532 target/quarkus-app/quarkus/ /deployments/quarkus/
```
- Start with `java -jar` (this image has no `run-java.sh`):
```dockerfile
ENV JAVA_TOOL_OPTIONS="-Dquarkus.http.host=0.0.0.0 -Djava.util.logging.manager=org.jboss.logmanager.LogManager -XX:MaxRAMPercentage=75.0 -XX:+ExitOnOutOfMemoryError"
WORKDIR /deployments
EXPOSE 8080
USER 65532
ENTRYPOINT ["java", "-jar", "/deployments/quarkus-run.jar"]
```
- Default build (no `-f`): `podman build -t <image> .`

### Native: `Containerfile.native`
- Base: `registry.access.redhat.com/hi/core-runtime:latest` (includes **glibc**; a standard dynamically linked Quarkus native binary runs here)
- Do **not** use `registry.access.redhat.com/hi/static:latest` unless the binary is fully statically linked (`--static`). `hi/static` has no glibc.
- Prerequisite: `./mvnw package -Dnative` (or add `-Dquarkus.native.container-build=true` if GraalVM is not local)
- Copy the runner as a single executable:
```dockerfile
COPY --chown=65532:65532 --chmod=0755 target/*-runner /application
EXPOSE 8080
USER 65532
ENTRYPOINT ["/application", "-Dquarkus.http.host=0.0.0.0"]
```
- Build with an explicit file: `podman build -f Containerfile.native -t <image> .`

### `.containerignore`
Name the file `.containerignore`, not `.dockerignore`. The context must ignore the rest of the tree but **keep nested** Quarkus outputs (a single `!target/quarkus-app/*` is not enough):
```gitignore
*
!target/*-runner
!target/*-runner.jar
!target/lib/*
!target/quarkus-app/
!target/quarkus-app/**
```

### Podman Compose
- Don't put a `version` key in Podman Compose files.

### Documenting the images
In the project README, document both flows: JVM `Containerfile` vs native `Containerfile.native`, matching image names on `podman build` and `podman run`.
