# Data Access Patterns (Panache)

## Choosing Between Active Record and Repository

| | Active Record | Repository |
|--|--------------|-----------|
| **Pattern** | `extends PanacheEntity` | `implements PanacheRepository<T>` |
| **Query location** | Static methods on entity | Methods on repository bean |
| **Unit test mocking** | Hard (static methods) | Easy (`@InjectMock ProductRepository`) |
| **Use when** | Simple CRUD, domain logic on entity | Strict separation, multiple data sources |

---

## Panache Entity (Active Record)

```java
@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_users_email", columnList = "email"),
    @Index(name = "idx_users_username", columnList = "username")
})
@NamedQuery(name = "User.findActiveByRole",
    query = "SELECT u FROM User u JOIN u.roles r WHERE u.active = true AND r = :role")
public class User extends PanacheEntity {

    @NotBlank
    @Email
    @Column(nullable = false, unique = true, length = 255)
    public String email;

    @NotBlank
    @Column(nullable = false, length = 100)
    public String username;

    @Column(nullable = false)
    public boolean active = true;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(name = "user_roles")
    public List<Role> roles = new ArrayList<>();

    @Version
    public Long version;

    @Column(updatable = false)
    public Instant createdAt;

    public Instant updatedAt;

    @PrePersist
    void onCreate() { this.createdAt = Instant.now(); }

    @PreUpdate
    void onUpdate() { this.updatedAt = Instant.now(); }

    // Static finder methods — queries live on the entity
    public static Optional<User> findByEmail(String email) {
        return find("email", email).firstResultOptional();
    }

    public static List<User> findActiveUsers() {
        return list("active", true);
    }

    public static List<User> findByRole(String role) {
        return find("#User.findActiveByRole", Parameters.with("role", role)).list();
    }

    // Panache provides: findById, findAll, list, stream, count, deleteById, etc.
}
```

> **Public fields**: Panache Active Record uses public fields. Hibernate uses bytecode enhancement
> to intercept field access, so lazy loading and dirty tracking still work correctly.

---

## Panache Repository

```java
@ApplicationScoped
public class UserRepository implements PanacheRepository<User> {

    // Custom finders
    public Optional<User> findByEmail(String email) {
        return find("email", email).firstResultOptional();
    }

    public List<User> findActiveUsers(int page, int size) {
        return find("active", true).page(page, size).list();
    }

    // JOIN FETCH to avoid N+1
    public List<User> findAllWithRoles() {
        return find("FROM User u LEFT JOIN FETCH u.roles").list();
    }

    public Optional<User> findByIdWithRoles(Long id) {
        return find("FROM User u LEFT JOIN FETCH u.roles WHERE u.id = ?1", id)
            .firstResultOptional();
    }

    // Count and pagination
    public long countActive() {
        return count("active", true);
    }
}
```

---

## Service with Transactions

```java
@ApplicationScoped
public class UserService {

    private final UserRepository repo;

    public UserService(UserRepository repo) {
        this.repo = repo;
    }

    // Read-only: no @Transactional needed unless lazy collections accessed
    public UserResponse findById(Long id) {
        return repo.findByIdOptional(id)
            .map(UserResponse::from)
            .orElseThrow(() -> new NotFoundException("User " + id + " not found"));
    }

    // Write: always @Transactional
    @Transactional
    public UserResponse create(CreateUserRequest req) {
        if (repo.findByEmail(req.email()).isPresent()) {
            throw new DomainException("Email already registered: " + req.email());
        }
        User user = new User();
        user.email = req.email();
        user.username = req.username();
        repo.persist(user);
        return UserResponse.from(user);
    }

    @Transactional
    public UserResponse update(Long id, UpdateUserRequest req) {
        User user = repo.findByIdOptional(id)
            .orElseThrow(() -> new NotFoundException("User " + id + " not found"));
        if (req.username() != null) user.username = req.username();
        // No explicit save — Hibernate dirty-checks within transaction
        return UserResponse.from(user);
    }

    @Transactional
    public void delete(Long id) {
        if (!repo.deleteById(id)) {
            throw new NotFoundException("User " + id + " not found");
        }
    }
}
```

> `@Transactional` must be `jakarta.transaction.Transactional` — not the Spring one.

---

## Dev Services (PostgreSQL)

With `quarkus-jdbc-postgresql` on the classpath, Dev Services automatically starts a
PostgreSQL container in dev and test modes. No `application.properties` datasource config needed.

```properties
# application.properties — minimal config, Dev Services handles the rest in dev/test
quarkus.datasource.db-kind=postgresql
quarkus.hibernate-orm.database.generation=validate

# Flyway runs migrations automatically
quarkus.flyway.migrate-at-start=true
quarkus.flyway.locations=classpath:db/migration

# Override for production
%prod.quarkus.datasource.jdbc.url=jdbc:postgresql://db:5432/myapp
%prod.quarkus.datasource.username=${DB_USER}
%prod.quarkus.datasource.password=${DB_PASSWORD}
```

Dev Services logs on startup:
```
Dev Services for default datasource (postgresql) started - 127.0.0.1:49152
```

---

## Flyway Migrations

```sql
-- src/main/resources/db/migration/V1__create_users_table.sql
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    email      VARCHAR(255) NOT NULL UNIQUE,
    username   VARCHAR(100) NOT NULL,
    active     BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ,
    version    BIGINT
);

CREATE INDEX idx_users_email    ON users (email);
CREATE INDEX idx_users_username ON users (username);
```

---

## Pagination with PanacheQuery

```java
@ApplicationScoped
public class ProductService {

    public PageResult<ProductResponse> listPaginated(int page, int size) {
        PanacheQuery<Product> query = Product.findAll();
        long total = query.count();
        List<ProductResponse> items = query.page(page, size).list()
            .stream().map(ProductResponse::from).toList();
        return new PageResult<>(items, total, page, size);
    }
}

// DTO for paginated response
public record PageResult<T>(List<T> items, long total, int page, int size) {
    public int totalPages() {
        return (int) Math.ceil((double) total / size);
    }
}
```

> Panache pagination is **zero-indexed** — page 0 is the first page.

---

## Optimistic Locking

```java
@Entity
public class Product extends PanacheEntity {
    public String name;

    @Version
    public Long version;  // Incremented on each UPDATE; conflicts throw OptimisticLockException
}

// Handle in service
@Transactional
public ProductResponse update(Long id, UpdateProductRequest req) {
    try {
        Product product = Product.findById(id);
        product.name = req.name();
        return ProductResponse.from(product);
    } catch (OptimisticLockException e) {
        throw new DomainException("Product was modified concurrently — please retry");
    }
}
```

---

## Auditing with Lifecycle Callbacks

```java
// Reusable base class (not a Panache entity itself)
@MappedSuperclass
public abstract class AuditedEntity extends PanacheEntity {

    @Column(updatable = false)
    public Instant createdAt;

    public Instant updatedAt;

    @PrePersist
    void onCreate() { createdAt = Instant.now(); }

    @PreUpdate
    void onUpdate() { updatedAt = Instant.now(); }
}

@Entity
public class Order extends AuditedEntity {
    public String reference;
}
```

---

## Quick Reference

| Method | Description |
|--------|-------------|
| `Product.findById(id)` | Find by primary key |
| `Product.findByIdOptional(id)` | Find by PK as Optional |
| `Product.findAll()` | Returns PanacheQuery (lazy) |
| `Product.listAll()` | Returns all as List |
| `Product.list("name", value)` | Simple equality filter |
| `Product.find("price < ?1", max)` | JPQL filter |
| `Product.count("active", true)` | Count with filter |
| `repo.persist(entity)` | INSERT |
| `repo.deleteById(id)` | DELETE by PK, returns boolean |
| `query.page(page, size).list()` | Paginated results (zero-indexed) |
| `query.firstResultOptional()` | First result as Optional |
