# Security Patterns (SmallRye JWT & OIDC)

## SmallRye JWT Setup

### Dependencies

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-jwt</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-jwt-build</artifactId>
</dependency>
```

### Configuration

```properties
# application.properties
mp.jwt.verify.publickey.location=META-INF/resources/publicKey.pem
mp.jwt.verify.issuer=https://auth.example.com

# Token signing (for token generation endpoint)
smallrye.jwt.sign.key.location=META-INF/resources/privateKey.pem

# HTTPS enforcement in production
%prod.quarkus.http.ssl.certificate.files=/certs/tls.crt
%prod.quarkus.http.ssl.certificate.key-files=/certs/tls.key
```

---

## Securing Resources

```java
// Require authentication for all methods in class
@Path("/api/v1/users")
@Authenticated
public class UserResource {

    @GET
    @RolesAllowed({"USER", "ADMIN"})
    public List<UserResponse> list() { ... }

    @GET
    @Path("/{id}")
    @RolesAllowed({"USER", "ADMIN"})
    public UserResponse getById(@PathParam("id") Long id) { ... }

    @POST
    @RolesAllowed("ADMIN")
    public Response create(@Valid CreateUserRequest request, @Context UriInfo uriInfo) { ... }

    @DELETE
    @Path("/{id}")
    @RolesAllowed("ADMIN")
    public Response delete(@PathParam("id") Long id) { ... }

    // Allow anonymous access to a specific method
    @GET
    @Path("/public/count")
    @PermitAll
    public long publicCount() { ... }
}
```

---

## Accessing JWT Claims

```java
@Path("/api/v1/profile")
@Authenticated
public class ProfileResource {

    @Inject
    JsonWebToken jwt;

    @Inject
    SecurityIdentity identity;

    @GET
    public ProfileResponse getProfile() {
        String subject    = jwt.getSubject();
        String email      = jwt.getClaim("email");
        Set<String> roles = identity.getRoles();
        String name       = identity.getPrincipal().getName();
        return new ProfileResponse(subject, email, name, roles);
    }
}
```

---

## Token Generation

```java
@ApplicationScoped
public class TokenService {

    public String generateToken(User user) {
        return Jwt.issuer("https://auth.example.com")
            .subject(user.id.toString())
            .claim("email", user.email)
            .claim("username", user.username)
            .groups(user.roles.stream().map(r -> r.name).collect(Collectors.toSet()))
            .expiresIn(Duration.ofHours(1))
            .sign();
    }
}

@Path("/api/v1/auth")
public class AuthResource {

    private final UserRepository repo;
    private final TokenService tokenService;

    public AuthResource(UserRepository repo, TokenService tokenService) {
        this.repo = repo;
        this.tokenService = tokenService;
    }

    @POST
    @Path("/login")
    @PermitAll
    @Produces(MediaType.TEXT_PLAIN)
    public Response login(@Valid LoginRequest request) {
        User user = repo.findByEmail(request.email())
            .filter(u -> BcryptUtil.matches(request.password(), u.passwordHash))
            .orElseThrow(() -> new NotAuthorizedException("Invalid credentials"));
        return Response.ok(tokenService.generateToken(user)).build();
    }
}
```

---

## Password Hashing

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-elytron-security-common</artifactId>
</dependency>
```

```java
import io.quarkus.elytron.security.common.BcryptUtil;

// Hash a password (e.g., at registration)
String hash = BcryptUtil.bcryptHash(plainTextPassword);

// Verify
boolean matches = BcryptUtil.matches(plainTextPassword, hash);
```

---

## Programmatic Authorization

Use when `@RolesAllowed` isn't expressive enough.

```java
@ApplicationScoped
public class UserService {

    @Inject
    SecurityIdentity identity;

    public UserResponse findById(Long id) {
        User user = repo.findByIdOptional(id)
            .orElseThrow(() -> new NotFoundException("User not found"));

        // Only ADMIN or the user themselves can view the profile
        String currentUserId = identity.getPrincipal().getName();
        if (!identity.hasRole("ADMIN") && !currentUserId.equals(id.toString())) {
            throw new ForbiddenException("Access denied");
        }
        return UserResponse.from(user);
    }
}
```

---

## OIDC (OpenID Connect)

For production SSO with Keycloak, Auth0, etc.

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-oidc</artifactId>
</dependency>
```

```properties
quarkus.oidc.auth-server-url=https://keycloak.example.com/realms/myrealm
quarkus.oidc.client-id=my-service
quarkus.oidc.application-type=service

# Dev Services starts Keycloak automatically in dev/test mode
%dev.quarkus.keycloak.devservices.realm-path=keycloak-realm.json
```

Annotation usage is identical: `@Authenticated`, `@RolesAllowed`.

---

## Security Best Practices

- Store secrets in environment variables, never in `application.properties` committed to git
- Use `%prod.*` prefix for production-only sensitive config
- Rotate JWT signing keys periodically
- Set `mp.jwt.verify.issuer` to prevent token reuse across services
- Prefer OIDC over hand-rolled JWT for production systems
- Stateless JWT APIs do not need CSRF protection (no session cookies)
- Set `quarkus.http.cors.origins` to explicit allowed origins in production (never `*` with credentials)
- Use `@Min`/`@Max` and `@Size` to prevent oversized inputs at the boundary

---

## Quick Reference

| Annotation / API | Purpose |
|-----------------|---------|
| `@Authenticated` | Require any valid JWT |
| `@RolesAllowed("ROLE")` | Require specific role claim |
| `@PermitAll` | Allow anonymous access |
| `@DenyAll` | Deny all (useful as class-level default) |
| `@Inject JsonWebToken jwt` | Access raw JWT claims |
| `@Inject SecurityIdentity identity` | Access principal, roles |
| `Jwt.issuer(...).sign()` | Build and sign a JWT |
| `BcryptUtil.bcryptHash(pw)` | Hash password |
| `BcryptUtil.matches(pw, hash)` | Verify password |
