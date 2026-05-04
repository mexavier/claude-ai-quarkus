# REST API Patterns (RESTEasy Reactive)

## Full CRUD Resource

```java
@Path("/api/v1/users")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@Tag(name = "Users", description = "User management")
public class UserResource {

    private final UserService service;

    public UserResource(UserService service) {
        this.service = service;
    }

    @GET
    @Operation(summary = "List users with optional search and pagination")
    @APIResponse(responseCode = "200", description = "User list")
    public List<UserResponse> list(
            @RestQuery String search,
            @RestQuery @DefaultValue("0") int page,
            @RestQuery @DefaultValue("20") int size) {
        return service.list(search, page, size);
    }

    @GET
    @Path("/{id}")
    @APIResponse(responseCode = "200", description = "User found")
    @APIResponse(responseCode = "404", description = "User not found")
    public UserResponse getById(@PathParam("id") Long id) {
        return service.findById(id);
    }

    @POST
    @Operation(summary = "Create a new user")
    @APIResponse(responseCode = "201", description = "User created")
    @APIResponse(responseCode = "400", description = "Validation error")
    public Response create(@Valid CreateUserRequest request, @Context UriInfo uriInfo) {
        UserResponse created = service.create(request);
        URI location = uriInfo.getAbsolutePathBuilder()
            .path(String.valueOf(created.id())).build();
        return Response.created(location).entity(created).build();
    }

    @PUT
    @Path("/{id}")
    @APIResponse(responseCode = "200", description = "User updated")
    @APIResponse(responseCode = "404", description = "User not found")
    public UserResponse update(@PathParam("id") Long id,
                               @Valid UpdateUserRequest request) {
        return service.update(id, request);
    }

    @DELETE
    @Path("/{id}")
    @APIResponse(responseCode = "204", description = "User deleted")
    @APIResponse(responseCode = "404", description = "User not found")
    public Response delete(@PathParam("id") Long id) {
        service.delete(id);
        return Response.noContent().build();
    }
}
```

---

## Request DTOs with Validation

```java
// Java records with Bean Validation (Hibernate Validator)
public record CreateUserRequest(
    @NotBlank @Email String email,
    @NotBlank @Size(min = 8) String password,
    @NotBlank @Size(min = 2, max = 50) String username,
    @Min(18) @Max(120) int age
) {}

public record UpdateUserRequest(
    @Size(min = 2, max = 50) String username,
    @Min(18) @Max(120) Integer age
) {}
```

---

## Response DTOs

```java
public record UserResponse(Long id, String email, String username, int age) {

    public static UserResponse from(User user) {
        return new UserResponse(user.id, user.email, user.username, user.age);
    }
}
```

---

## Exception Mappers

```java
@Provider
public class NotFoundExceptionMapper implements ExceptionMapper<NotFoundException> {

    @Override
    public Response toResponse(NotFoundException ex) {
        return Response.status(Response.Status.NOT_FOUND)
            .entity(Map.of("error", ex.getMessage()))
            .build();
    }
}

@Provider
public class ConstraintViolationExceptionMapper
        implements ExceptionMapper<ConstraintViolationException> {

    @Override
    public Response toResponse(ConstraintViolationException ex) {
        Map<String, String> errors = ex.getConstraintViolations().stream()
            .collect(Collectors.toMap(
                v -> extractFieldName(v.getPropertyPath().toString()),
                v -> v.getMessage()
            ));
        return Response.status(Response.Status.BAD_REQUEST).entity(errors).build();
    }

    private String extractFieldName(String path) {
        String[] parts = path.split("\\.");
        return parts[parts.length - 1];
    }
}

// Domain exception base
public class DomainException extends RuntimeException {
    public DomainException(String message) { super(message); }
}

@Provider
public class DomainExceptionMapper implements ExceptionMapper<DomainException> {

    @Override
    public Response toResponse(DomainException ex) {
        return Response.status(Response.Status.UNPROCESSABLE_ENTITY)
            .entity(Map.of("error", ex.getMessage()))
            .build();
    }
}
```

---

## Custom Validation Annotation

```java
// 1. Define the annotation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = ValidEmailDomainValidator.class)
public @interface ValidEmailDomain {
    String message() default "Email domain not allowed";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    String[] allowedDomains() default {"example.com"};
}

// 2. Implement the validator (plain CDI bean — no Spring needed)
@ApplicationScoped
public class ValidEmailDomainValidator
        implements ConstraintValidator<ValidEmailDomain, String> {

    private String[] allowedDomains;

    @Override
    public void initialize(ValidEmailDomain annotation) {
        this.allowedDomains = annotation.allowedDomains();
    }

    @Override
    public boolean isValid(String email, ConstraintValidatorContext ctx) {
        if (email == null) return true;
        return Arrays.stream(allowedDomains)
            .anyMatch(d -> email.endsWith("@" + d));
    }
}

// 3. Use it
public record CreateUserRequest(
    @ValidEmailDomain(allowedDomains = {"company.com"}) String email
) {}
```

---

## REST Client Reactive

```java
// Define the client interface
@RegisterRestClient(configKey = "payment-api")
@Path("/payments")
public interface PaymentClient {

    @POST
    @Produces(MediaType.APPLICATION_JSON)
    @Consumes(MediaType.APPLICATION_JSON)
    PaymentResponse charge(ChargeRequest request);

    @GET
    @Path("/{id}/status")
    PaymentStatus getStatus(@PathParam("id") String paymentId);
}

// Inject and use
@ApplicationScoped
public class PaymentService {

    @RestClient
    PaymentClient paymentClient;

    public PaymentResponse charge(ChargeRequest request) {
        return paymentClient.charge(request);
    }
}
```

```properties
# application.properties
quarkus.rest-client.payment-api.url=https://api.payments.example.com
quarkus.rest-client.payment-api.scope=jakarta.inject.Singleton
quarkus.rest-client.payment-api.connect-timeout=5000
quarkus.rest-client.payment-api.read-timeout=10000
```

---

## CORS Configuration

```properties
# application.properties
quarkus.http.cors=true
quarkus.http.cors.origins=http://localhost:3000,https://app.example.com
quarkus.http.cors.methods=GET,POST,PUT,DELETE,OPTIONS,PATCH
quarkus.http.cors.headers=accept,authorization,content-type
quarkus.http.cors.exposed-headers=location
quarkus.http.cors.access-control-max-age=24H
```

---

## OpenAPI / SmallRye

SmallRye OpenAPI is auto-configured when the extension is on the classpath.

```java
// Resource-level annotation
@Tag(name = "Products", description = "Product catalog operations")
@Path("/api/v1/products")
public class ProductResource { ... }

// Method-level annotations
@Operation(summary = "Create product", description = "Creates a new product in the catalog")
@APIResponse(responseCode = "201", description = "Product created",
    content = @Content(schema = @Schema(implementation = ProductResponse.class)))
@APIResponse(responseCode = "400", description = "Validation failed")
@POST
public Response create(@Valid CreateProductRequest request, @Context UriInfo uriInfo) { ... }
```

```properties
# application.properties
quarkus.smallrye-openapi.info-title=My API
quarkus.smallrye-openapi.info-version=1.0.0
quarkus.smallrye-openapi.info-description=Enterprise API built with Quarkus

# Dev-only Swagger UI
%dev.quarkus.swagger-ui.always-include=true
```

Endpoints: `/q/openapi` (spec), `/q/swagger-ui` (UI in dev).

---

## Quick Reference

| Annotation | Purpose |
|-----------|---------|
| `@Path("/api/v1/x")` | Resource root path |
| `@GET`, `@POST`, `@PUT`, `@DELETE`, `@PATCH` | HTTP method |
| `@PathParam("id")` | Path variable |
| `@RestQuery` | Query param (RESTEasy Reactive shorthand) |
| `@DefaultValue("0")` | Default query param value |
| `@Valid` | Trigger Bean Validation |
| `@Produces(APPLICATION_JSON)` | Response content type |
| `@Consumes(APPLICATION_JSON)` | Request content type |
| `@Context UriInfo` | Build Location URIs |
| `@Provider` | Register JAX-RS provider (ExceptionMapper, etc.) |
| `@RegisterRestClient` | Declare REST client interface |
| `@RestClient` | Inject REST client |
| `@Tag`, `@Operation`, `@APIResponse` | SmallRye OpenAPI docs |
