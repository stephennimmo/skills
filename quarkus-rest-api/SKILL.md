---
name: quarkus-rest-api
description: How to create and maintain a REST API with Quarkus. Use when creating or changing a Quarkus project, REST endpoint, Resource/Service/Repository class, request/response/domain record, Panache entity, Flyway migration, OIDC security, application.yaml configuration, Containerfile, or API tests.
---

Also follow the `java` skill for general Java conventions (records, Optional, constructor injection, validation).

## Project Basics

- API projects listen on port 8080.
- YAML files end in `.yaml`, not `.yml`.
- Use dev services whenever running locally. Don't hand-configure a local database or OIDC server.
- Absolutely no sensitive information (passwords, client secrets, tokens) in the git repository. Read them from environment variables in `%prod`.
- When code changes, update the README and any other documentation in the project.

## Layering: Resource → Service → Repository

Every REST feature is built in three layers. Dependencies point one way only: Resource → Service → Repository. No layer skips a level or calls upward.

### Packages

Package by domain subject, not by layer. All three layers for a subject live together in one package: `com.example.${project_name}.${subject}`. There are no `api`, `service` or `repository` packages.

For a `customerapi` project, the `com.example.customerapi.customer` package contains:

| Class                 | Layer      | Kind                                              | Public methods consume/produce |
| ---                   | ---        | ---                                               | ---                            |
| `CustomerResource`    | Resource   | Resource class                                    | Request and Response records   |
| `NewCustomerRequest`  | Resource   | Request record for creating (`POST`)              |                                |
| `EditCustomerRequest` | Resource   | Request record for updating (`PUT`)               |                                |
| `CustomerResponse`    | Resource   | Response record                                   |                                |
| `CustomerService`     | Service    | Service class                                     | Domain records only            |
| `Customer`            | Service    | Domain record (just the noun, no `Domain` suffix) |                                |
| `CustomerRepository`  | Repository | Repository class                                  | Entity classes only            |
| `CustomerEntity`      | Repository | Panache entity                                    |                                |

Because the layers share a package, package boundaries don't enforce the layering. The class roles and the dependency rules below do: a Resource never touches `CustomerRepository` or `CustomerEntity`, even though they are visible.

Generated IDs are named after the entity, never just `id`: `Customer` has `customerId`, `Bill` has `billId`. This applies to entities, domain records, request/response records and database columns (`customer_id`).

### Resource (`*Resource`)
Owns the HTTP contract and nothing else.
- Annotated with `@Path`, uses Quarkus REST (`quarkus-rest`, `quarkus-rest-jackson`).
- Version every endpoint in the URL: `/api/v1/customers`.
- Accepts Request records and returns Response records only. Never exposes domain records or entities in the API.
- Uses a separate Request record per operation: `NewCustomerRequest` for create, `EditCustomerRequest` for update. The ID comes from the path, never from the request body.
- Validates input with `@Valid` and Bean Validation annotations on the Request record.
- Converts Request → domain with `request.toDomain()` (`request.toDomain(customerId)` for edits) and domain → Response with `CustomerResponse.from(customer)`.
- Always returns `jakarta.ws.rs.core.Response` with the correct HTTP status code (201 + Location on create, 204 on delete, 404 when not found, 400 for bad requests).
- Documents every endpoint with OpenAPI (`quarkus-smallrye-openapi`). Use `@APIResponse` to declare each response code and its return type, including the error states.
- Puts `@RolesAllowed` on each method, never on the class.
- Injects Services only. MUST NOT inject a Repository or `EntityManager`.
- Contains no business logic, no `@Transactional`.

```java
package com.example.customerapi.customer;

@Path("/api/v1/customers")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class CustomerResource {

    private final CustomerService customerService;

    public CustomerResource(CustomerService customerService) {
        this.customerService = customerService;
    }

    @GET
    @Path("/{customerId}")
    @RolesAllowed(Roles.CRM_READ)
    @APIResponse(responseCode = "200", content = @Content(schema = @Schema(implementation = CustomerResponse.class)))
    @APIResponse(responseCode = "404", description = "Customer not found")
    public Response get(@PathParam("customerId") Integer customerId) {
        return customerService.findById(customerId)
                .map(customer -> Response.ok(CustomerResponse.from(customer)).build())
                .orElseGet(() -> Response.status(Response.Status.NOT_FOUND).build());
    }

    @POST
    @RolesAllowed(Roles.CRM_WRITE)
    @APIResponse(responseCode = "201", content = @Content(schema = @Schema(implementation = CustomerResponse.class)))
    @APIResponse(responseCode = "400", description = "Invalid request")
    public Response create(@Valid NewCustomerRequest request, @Context UriInfo uriInfo) {
        Customer customer = customerService.create(request.toDomain());
        URI location = uriInfo.getAbsolutePathBuilder().path(customer.customerId().toString()).build();
        return Response.created(location).entity(CustomerResponse.from(customer)).build();
    }

    @PUT
    @Path("/{customerId}")
    @RolesAllowed(Roles.CRM_WRITE)
    @APIResponse(responseCode = "200", content = @Content(schema = @Schema(implementation = CustomerResponse.class)))
    @APIResponse(responseCode = "400", description = "Invalid request")
    @APIResponse(responseCode = "404", description = "Customer not found")
    public Response update(@PathParam("customerId") Integer customerId, @Valid EditCustomerRequest request) {
        return customerService.update(request.toDomain(customerId))
                .map(customer -> Response.ok(CustomerResponse.from(customer)).build())
                .orElseGet(() -> Response.status(Response.Status.NOT_FOUND).build());
    }
}
```

### Service (`*Service`)
Owns business logic and transactions.
- `@ApplicationScoped`. Put `@Transactional` on service methods that write.
- Public methods consume and return Domain records only. Never return entities.
- Converts domain ↔ entity internally.
- Injects Repositories only. The service layer is the only way to reach data access.

### Repository (`*Repository`)
Owns data access.
- Use the Panache repository pattern: `@ApplicationScoped` classes implementing `PanacheRepositoryBase<CustomerEntity, Integer>`.
- Public methods consume and return Entity classes only.

## Security

- Authentication and authorization use OIDC with JWT bearer tokens (`quarkus-oidc`).
- Keycloak is the reference provider, but stay provider-agnostic: only standard OIDC configuration, no Keycloak-specific APIs or extensions, so any OIDC provider can be swapped in.
- Define roles as `String` constants in an interface named `Roles`, and use those constants in `@RolesAllowed` instead of string literals.
- `Roles` is shared by every subject, so it lives in the `security` package (`com.example.${project_name}.security`), not in a subject package:
```java
package com.example.customerapi.security;

public interface Roles {
    String CRM_READ = "crm-read";
    String CRM_WRITE = "crm-write";
    String CRM_ADMIN = "crm-admin";
}
```

## Panache Entities

- Suffix is `Entity`: `PersonEntity`, `CarEntity`.
- Always annotate the class with both `@Entity(name = "...")` and `@Table(name = "...")`. The entity name is the plain noun without the `Entity` suffix, and the table name is the exact table: `CustomerEntity` has `@Entity(name = "Customer")` and `@Table(name = "customer")`.
- The ID follows the entity name: `PersonEntity` has `personId`, mapped to the column `person_id`.
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

Use Flyway (`quarkus-flyway`) for all schema management. Migrations go in `src/main/resources/db/migration`.

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

## Configuration

Configuration lives in `src/main/resources/application.yaml` (requires `quarkus-config-yaml`) and uses the `%dev`, `%test`, `%prod` profile structure.

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

## Testing

Extensive tests for every API endpoint are required, not optional.
- Use `@QuarkusTest` with REST Assured against the real HTTP endpoints, backed by dev services.
- Cover the success path and every error state declared with `@APIResponse` (400, 404, and so on).
- Cover security: unauthenticated requests and requests without the required role.

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

### `.dockerignore`
The context must ignore the rest of the tree but **keep nested** Quarkus outputs (a single `!target/quarkus-app/*` is not enough):
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

### README
Document both flows: JVM `Containerfile` vs native `Containerfile.native`, matching image names on `podman build` and `podman run`.
