---
name: quarkus
description: Quarkus 3.x development - RESTEasy Reactive, Panache, SmallRye, Testing, and Cloud-native patterns. Use for building enterprise Java applications with Quarkus.
metadata:
  version: "1.0.0"
  domain: backend
  triggers: Quarkus, RESTEasy Reactive, Panache, SmallRye, MicroProfile, CDI, Quarkus native, Jakarta EE
  role: specialist
  scope: implementation
  output-format: code
---

# Quarkus Skill

Enterprise Quarkus 3.x development with RESTEasy Reactive, Panache, MicroProfile, and SmallRye.

## Core Workflow

1. **Analyze** — Requirements, CDI scopes, Panache strategy (Active Record vs Repository), security model
2. **Design** — Extensions list, architecture layers, API contract
3. **Implement** — JAX-RS resources, CDI services, Panache repositories, ExceptionMappers
4. **Secure** — SmallRye JWT or OIDC, `@RolesAllowed`, input validation
5. **Test** — `@QuarkusTest` + RestAssured, run `./mvnw test` (green required)
6. **Deploy** — `/q/health` returns UP, Docker Compose, CircleCI pipeline

---

## Architecture

```
src/main/java/pl/piomin/services/
├── resource/     # JAX-RS @Path endpoints
├── service/      # @ApplicationScoped business logic
├── repository/   # PanacheRepository implementations
├── entity/       # @Entity (Panache) classes
├── dto/          # Java record Request/Response DTOs
├── config/       # @ConfigMapping interfaces
└── exception/    # Custom exceptions + ExceptionMappers
```

---

## Quick Start Templates

### Entity (Panache Active Record)

```java
package pl.piomin.services.entity;

import io.quarkus.hibernate.orm.panache.PanacheEntity;
import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

@Entity
@Table(name = "products", indexes = @Index(columnList = "name"))
public class Product extends PanacheEntity {

    @NotBlank
    @Column(nullable = false)
    public String name;

    @DecimalMin("0.0")
    @Column(nullable = false)
    public BigDecimal price;

    @Version
    public Long version;

    public static Optional<Product> findByName(String name) {
        return find("name", name).firstResultOptional();
    }

    public static List<Product> findByPriceLessThan(BigDecimal maxPrice) {
        return list("price < ?1", maxPrice);
    }
}
```

### Repository (Alternative Pattern)

```java
package pl.piomin.services.repository;

import io.quarkus.hibernate.orm.panache.PanacheRepository;
import jakarta.enterprise.context.ApplicationScoped;
import pl.piomin.services.entity.Product;
import java.math.BigDecimal;
import java.util.List;

@ApplicationScoped
public class ProductRepository implements PanacheRepository<Product> {

    public List<Product> findCheaperThan(BigDecimal maxPrice) {
        return list("price < ?1", maxPrice);
    }

    public List<Product> findAllWithCategory() {
        return find("FROM Product p LEFT JOIN FETCH p.category").list();
    }
}
```

### Service

```java
package pl.piomin.services.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.NotFoundException;
import pl.piomin.services.dto.CreateProductRequest;
import pl.piomin.services.dto.ProductResponse;
import pl.piomin.services.entity.Product;
import pl.piomin.services.repository.ProductRepository;
import java.util.List;

@ApplicationScoped
public class ProductService {

    private final ProductRepository repo;

    public ProductService(ProductRepository repo) {
        this.repo = repo;
    }

    public List<ProductResponse> list() {
        return repo.listAll().stream().map(ProductResponse::from).toList();
    }

    public ProductResponse findById(Long id) {
        return repo.findByIdOptional(id)
            .map(ProductResponse::from)
            .orElseThrow(() -> new NotFoundException("Product " + id + " not found"));
    }

    @Transactional
    public ProductResponse create(CreateProductRequest request) {
        Product product = new Product();
        product.name = request.name();
        product.price = request.price();
        repo.persist(product);
        return ProductResponse.from(product);
    }

    @Transactional
    public ProductResponse update(Long id, CreateProductRequest request) {
        Product product = repo.findByIdOptional(id)
            .orElseThrow(() -> new NotFoundException("Product " + id + " not found"));
        product.name = request.name();
        product.price = request.price();
        return ProductResponse.from(product);
    }

    @Transactional
    public void delete(Long id) {
        if (!repo.deleteById(id)) {
            throw new NotFoundException("Product " + id + " not found");
        }
    }
}
```

### REST Resource

```java
package pl.piomin.services.resource;

import jakarta.validation.Valid;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.*;
import org.jboss.resteasy.reactive.RestQuery;
import pl.piomin.services.dto.CreateProductRequest;
import pl.piomin.services.dto.ProductResponse;
import pl.piomin.services.service.ProductService;
import java.net.URI;
import java.util.List;

@Path("/api/v1/products")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class ProductResource {

    private final ProductService service;

    public ProductResource(ProductService service) {
        this.service = service;
    }

    @GET
    public List<ProductResponse> list() {
        return service.list();
    }

    @GET
    @Path("/{id}")
    public ProductResponse getById(@PathParam("id") Long id) {
        return service.findById(id);
    }

    @POST
    public Response create(@Valid CreateProductRequest request, @Context UriInfo uriInfo) {
        ProductResponse created = service.create(request);
        URI location = uriInfo.getAbsolutePathBuilder()
            .path(String.valueOf(created.id())).build();
        return Response.created(location).entity(created).build();
    }

    @PUT
    @Path("/{id}")
    public ProductResponse update(@PathParam("id") Long id,
                                  @Valid CreateProductRequest request) {
        return service.update(id, request);
    }

    @DELETE
    @Path("/{id}")
    @jakarta.ws.rs.core.Response.Status
    public Response delete(@PathParam("id") Long id) {
        service.delete(id);
        return Response.noContent().build();
    }
}
```

### Request/Response DTOs

```java
package pl.piomin.services.dto;

import jakarta.validation.constraints.*;
import java.math.BigDecimal;

public record CreateProductRequest(
    @NotBlank String name,
    @NotNull @DecimalMin("0.0") BigDecimal price
) {}
```

```java
package pl.piomin.services.dto;

import pl.piomin.services.entity.Product;
import java.math.BigDecimal;

public record ProductResponse(Long id, String name, BigDecimal price) {
    public static ProductResponse from(Product p) {
        return new ProductResponse(p.id, p.name, p.price);
    }
}
```

### Exception Mapper

```java
package pl.piomin.services.exception;

import jakarta.validation.ConstraintViolationException;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.ExceptionMapper;
import jakarta.ws.rs.ext.Provider;
import java.util.Map;
import java.util.stream.Collectors;

@Provider
public class ConstraintViolationExceptionMapper
        implements ExceptionMapper<ConstraintViolationException> {

    @Override
    public Response toResponse(ConstraintViolationException ex) {
        Map<String, String> errors = ex.getConstraintViolations().stream()
            .collect(Collectors.toMap(
                v -> v.getPropertyPath().toString(),
                v -> v.getMessage()
            ));
        return Response.status(Response.Status.BAD_REQUEST).entity(errors).build();
    }
}
```

### Test

```java
package pl.piomin.services.resource;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class ProductResourceTest {

    @Test
    void createProduct_validRequest_returns201() {
        given()
            .contentType(ContentType.JSON)
            .body("""{"name":"Widget","price":10.99}""")
        .when()
            .post("/api/v1/products")
        .then()
            .statusCode(201)
            .body("name", equalTo("Widget"))
            .body("price", equalTo(10.99f))
            .header("Location", notNullValue());
    }

    @Test
    void createProduct_missingName_returns400() {
        given()
            .contentType(ContentType.JSON)
            .body("""{"price":10.99}""")
        .when()
            .post("/api/v1/products")
        .then()
            .statusCode(400);
    }

    @Test
    void getProduct_notFound_returns404() {
        given()
        .when()
            .get("/api/v1/products/99999")
        .then()
            .statusCode(404);
    }
}
```

---

## Constraints

### MUST DO
- CDI constructor injection (no `@Inject` on fields)
- `@Valid` on all request body parameters
- `@Transactional` (from `jakarta.transaction`) for all write operations
- `@ConfigProperty` / `@ConfigMapping` for all config — no hardcoded values
- `ExceptionMapper` (`@Provider`) for all custom domain exceptions
- `application.properties` only — no YAML unless `quarkus-config-yaml` is added
- Use Panache Active Record or Repository — never raw `EntityManager` in resource layer
- `./mvnw test` must pass before marking task complete

### MUST NOT DO
- Spring annotations: `@RestController`, `@Service`, `@Autowired`, `@Component`, `@SpringBootApplication`
- `spring-boot-starter-*` dependencies
- Field injection with `@Inject` on non-constructor sites
- `@Singleton` from `javax.inject` (use `@ApplicationScoped`)
- Skip `@Transactional` on multi-step write operations
- Mix blocking Panache calls with Mutiny reactive without vertical separation
- `org.springframework.transaction.annotation.Transactional` (use jakarta)

---

## Reference Guide

| Topic | Reference File |
|-------|---------------|
| REST endpoints, validation, OpenAPI | `references/web.md` |
| Panache entities, repositories, transactions | `references/data.md` |
| SmallRye JWT, OIDC, `@RolesAllowed` | `references/security.md` |
| MicroProfile Health, Fault Tolerance, Stork | `references/cloud.md` |
| `@QuarkusTest`, RestAssured, Dev Services | `references/testing.md` |

---

## Common Annotations

| Annotation | Purpose |
|-----------|---------|
| `@Path("/api/v1/users")` | JAX-RS resource root path |
| `@GET`, `@POST`, `@PUT`, `@DELETE`, `@PATCH` | HTTP method mapping |
| `@PathParam("id")` | URI path variable |
| `@RestQuery` | Query string parameter (RESTEasy Reactive shorthand) |
| `@ApplicationScoped` | CDI singleton (one per application) |
| `@RequestScoped` | CDI one-per-request scope |
| `@Transactional` | JTA transaction boundary |
| `@ConfigProperty` | Single config value injection |
| `@ConfigMapping` | Type-safe grouped config |
| `@RolesAllowed` | Method-level RBAC |
| `@Authenticated` | Require any authenticated user |
| `@QuarkusTest` | Quarkus integration test |
| `@InjectMock` | CDI bean mock in tests |
| `@TestSecurity` | Simulate authenticated user in tests |

---

## Knowledge Base

Quarkus 3.33.1 (LTS), Java 25, Jakarta EE 10, MicroProfile 6, RESTEasy Reactive, Hibernate ORM with Panache, SmallRye JWT, SmallRye Fault Tolerance, SmallRye Health, SmallRye OpenAPI, Quarkus Flyway, OpenTelemetry, ArC CDI, RestAssured, JUnit 5, Dev Services, GraalVM native, Maven 3.9+.
