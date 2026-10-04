---
name: quarkus-rest
description: Quarkus REST API conventions. Use when adding or changing a REST endpoint or feature in a Quarkus project, including Resource/Service/Repository classes, package layout, request/response/domain records, OpenAPI annotations, @RolesAllowed and OIDC security, and endpoint tests.
---

Follow the `java` skill for general Java conventions (records, Optional, constructor injection, validation).
Follow the `quarkus` skill for project conventions (configuration, dev services, Panache entities, Flyway schema, Containerfiles).

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

Generated IDs are named after the entity, never just `id`: `Customer` has `customerId`, `Bill` has `billId`. Use the same name in domain, Request and Response records. (Entity and column naming is in the `quarkus` skill.)

### Resource (`*Resource`)
Owns the HTTP contract and nothing else.
- Annotated with `@Path`, uses Quarkus REST (extensions `quarkus-rest`, `quarkus-rest-jackson`).
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
- Use the Panache repository pattern, not active record: `@ApplicationScoped` classes implementing `PanacheRepositoryBase<CustomerEntity, Integer>`.
- Public methods consume and return Entity classes only.
- Entity and schema rules are in the `quarkus` skill.

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

## Testing

Extensive tests for every API endpoint are required, not optional.
- Use `@QuarkusTest` with REST Assured against the real HTTP endpoints, backed by dev services.
- Cover the success path and every error state declared with `@APIResponse` (400, 404, and so on).
- Cover security: unauthenticated requests and requests without the required role.
