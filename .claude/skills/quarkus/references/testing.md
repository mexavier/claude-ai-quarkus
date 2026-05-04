# Testing Strategies (Quarkus)

## Unit Testing (no framework dependency)

```java
// Plain JUnit 5 + Mockito — identical to any Java project
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserRepository userRepository;

    @InjectMocks
    UserService userService;

    @Test
    void create_newEmail_persistsUser() {
        // Arrange
        var request = new CreateUserRequest("alice@example.com", "P@ssword1", "alice", 25);
        when(userRepository.findByEmail("alice@example.com")).thenReturn(Optional.empty());

        // Act
        userService.create(request);

        // Assert
        verify(userRepository).persist(argThat(u -> "alice@example.com".equals(u.email)));
    }

    @Test
    void create_duplicateEmail_throwsDomainException() {
        var request = new CreateUserRequest("alice@example.com", "P@ssword1", "alice", 25);
        when(userRepository.findByEmail("alice@example.com"))
            .thenReturn(Optional.of(new User()));

        assertThrows(DomainException.class, () -> userService.create(request));
    }
}
```

---

## Integration Testing with `@QuarkusTest`

```java
@QuarkusTest
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class UserResourceTest {

    @Test
    @Order(1)
    void createUser_validRequest_returns201() {
        given()
            .contentType(ContentType.JSON)
            .body("""
                {
                  "email": "alice@example.com",
                  "password": "P@ssword1",
                  "username": "alice",
                  "age": 25
                }
                """)
        .when()
            .post("/api/v1/users")
        .then()
            .statusCode(201)
            .body("email", equalTo("alice@example.com"))
            .body("username", equalTo("alice"))
            .header("Location", containsString("/api/v1/users/"));
    }

    @Test
    @Order(2)
    void getUser_existingId_returns200() {
        // Assumes user created in test 1; use @TestTransaction to reset if needed
        given()
        .when()
            .get("/api/v1/users/1")
        .then()
            .statusCode(200)
            .body("email", equalTo("alice@example.com"));
    }

    @Test
    @Order(3)
    void updateUser_validRequest_returns200() {
        given()
            .contentType(ContentType.JSON)
            .body("""{"username":"alice_updated","age":26}""")
        .when()
            .put("/api/v1/users/1")
        .then()
            .statusCode(200)
            .body("username", equalTo("alice_updated"));
    }

    @Test
    @Order(4)
    void deleteUser_existingId_returns204() {
        given()
        .when()
            .delete("/api/v1/users/1")
        .then()
            .statusCode(204);
    }

    // --- Negative cases ---

    @Test
    void createUser_invalidEmail_returns400() {
        given()
            .contentType(ContentType.JSON)
            .body("""{"email":"not-an-email","password":"P@ssword1","username":"bob","age":20}""")
        .when()
            .post("/api/v1/users")
        .then()
            .statusCode(400)
            .body("email", notNullValue());
    }

    @Test
    void getUser_notFound_returns404() {
        given()
        .when()
            .get("/api/v1/users/99999")
        .then()
            .statusCode(404);
    }

    @Test
    void deleteUser_notFound_returns404() {
        given()
        .when()
            .delete("/api/v1/users/99999")
        .then()
            .statusCode(404);
    }
}
```

`@QuarkusTest` + Dev Services: a real PostgreSQL container starts automatically. No manual
`@DynamicPropertySource` or `@Container` needed.

---

## Resetting Database State Between Tests

```java
@QuarkusTest
class IsolatedUserTest {

    @Test
    @TestTransaction  // Rolls back after each test method
    void createUser_transactionRolledBack() {
        given()
            .contentType(ContentType.JSON)
            .body("""{"email":"temp@example.com","password":"P@ssword1","username":"temp","age":20}""")
        .when()
            .post("/api/v1/users")
        .then()
            .statusCode(201);
        // Database is rolled back after this test — no state leaks
    }
}
```

---

## Mocking CDI Beans with `@InjectMock`

```java
@QuarkusTest
class PaymentResourceTest {

    @InjectMock
    PaymentService paymentService;  // CDI bean replaced by Mockito mock

    @Test
    void chargePayment_serviceReturnsSuccess_returns200() {
        when(paymentService.charge(any()))
            .thenReturn(new PaymentResult("tx-123", PaymentStatus.SUCCESS));

        given()
            .contentType(ContentType.JSON)
            .body("""{"amount":99.99,"currency":"USD","cardToken":"tok_abc"}""")
        .when()
            .post("/api/v1/payments/charge")
        .then()
            .statusCode(200)
            .body("transactionId", equalTo("tx-123"))
            .body("status", equalTo("SUCCESS"));
    }

    @Test
    void chargePayment_serviceThrows_returns500() {
        when(paymentService.charge(any())).thenThrow(new RuntimeException("Gateway down"));

        given()
            .contentType(ContentType.JSON)
            .body("""{"amount":99.99,"currency":"USD","cardToken":"tok_abc"}""")
        .when()
            .post("/api/v1/payments/charge")
        .then()
            .statusCode(500);
    }
}
```

---

## External Service Mocking with `@QuarkusTestResource`

```java
// 1. Implement lifecycle manager
public class WireMockResource implements QuarkusTestResourceLifecycleManager {

    private WireMockServer server;

    @Override
    public Map<String, String> start() {
        server = new WireMockServer(options().dynamicPort());
        server.start();

        // Stub external endpoint
        server.stubFor(get(urlEqualTo("/external/data/123"))
            .willReturn(aResponse()
                .withHeader("Content-Type", "application/json")
                .withBody("""{"id":"123","value":"test"}""")));

        // Override the REST client URL to point to WireMock
        return Map.of("quarkus.rest-client.external-api.url", server.baseUrl());
    }

    @Override
    public void stop() { server.stop(); }
}

// 2. Use in test
@QuarkusTest
@QuarkusTestResource(WireMockResource.class)
class ExternalApiResourceTest {

    @Test
    void fetchData_existingId_returns200() {
        given()
        .when()
            .get("/api/v1/data/123")
        .then()
            .statusCode(200)
            .body("value", equalTo("test"));
    }
}
```

---

## Security Testing

```java
@QuarkusTest
class SecuredResourceTest {

    @Test
    @TestSecurity(user = "alice", roles = "USER")
    void getProfile_authenticatedUser_returns200() {
        given()
        .when()
            .get("/api/v1/profile")
        .then()
            .statusCode(200);
    }

    @Test
    @TestSecurity(user = "bob", roles = "ADMIN")
    void deleteUser_adminRole_returns204() {
        given()
        .when()
            .delete("/api/v1/users/1")
        .then()
            .statusCode(204);
    }

    @Test
    void getProfile_noToken_returns401() {
        given()
        .when()
            .get("/api/v1/profile")
        .then()
            .statusCode(401);
    }

    @Test
    @TestSecurity(user = "charlie", roles = "USER")
    void deleteUser_insufficientRole_returns403() {
        given()
        .when()
            .delete("/api/v1/users/1")
        .then()
            .statusCode(403);
    }
}
```

---

## Test Profile (`application.properties` in `src/test/resources`)

```properties
# src/test/resources/application.properties
# Dev Services handles PostgreSQL automatically — override only when needed
quarkus.datasource.db-kind=postgresql
quarkus.hibernate-orm.database.generation=drop-and-create

# Faster startup in tests
quarkus.log.level=WARN
quarkus.log.category."pl.piomin.services".level=DEBUG

# Disable features not needed in tests
%test.quarkus.otel.sdk.disabled=true
```

---

## Native Integration Testing

```java
// Runs the same test suite against the native binary
// Must build native first: ./mvnw package -Pnative
@QuarkusIntegrationTest
class NativeUserResourceIT extends UserResourceTest {
    // No additional code needed — inherits all tests
}
```

---

## Quick Reference

| Annotation | Purpose |
|-----------|---------|
| `@QuarkusTest` | Boot full Quarkus app (with Dev Services) |
| `@TestTransaction` | Roll back DB after each test method |
| `@InjectMock` | Replace CDI bean with Mockito mock |
| `@QuarkusTestResource` | Start/stop external resources (WireMock, custom) |
| `@TestSecurity` | Simulate an authenticated user |
| `@QuarkusIntegrationTest` | Run tests against native binary |
| `given().when().then()` | RestAssured BDD-style HTTP test |
| `@TestMethodOrder` + `@Order` | Control test execution order |
