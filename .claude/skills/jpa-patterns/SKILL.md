---
name: jpa-patterns
description: JPA/Hibernate patterns and common pitfalls (N+1, lazy loading, transactions, queries). Use when user has JPA performance issues, LazyInitializationException, or asks about entity relationships and fetching strategies.
---

# JPA Patterns Skill

Best practices and common pitfalls for JPA/Hibernate in Spring applications.

## When to Use
- User mentions "N+1 problem" / "too many queries"
- LazyInitializationException errors
- Questions about fetch strategies (EAGER vs LAZY)
- Transaction management issues
- Entity relationship design
- Query optimization

---

## Quick Reference: Common Problems

| Problem | Symptom | Solution |
|---------|---------|----------|
| N+1 queries | Many SELECT statements | JOIN FETCH, @EntityGraph |
| LazyInitializationException | Error outside transaction | Open Session in View, DTO projection, JOIN FETCH |
| Slow queries | Performance issues | Pagination, projections, indexes |
| Dirty checking overhead | Slow updates | Read-only transactions, DTOs |
| Lost updates | Concurrent modifications | Optimistic locking (@Version) |

---

## N+1 Problem

> The #1 JPA performance killer

### The Problem

```java
// ❌ BAD: N+1 queries
@Entity
public class Author {
    @Id private Long id;
    private String name;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    private List<Book> books;
}

// This innocent code...
List<Author> authors = authorRepository.findAll();  // 1 query
for (Author author : authors) {
    System.out.println(author.getBooks().size());   // N queries!
}
// Result: 1 + N queries (if 100 authors = 101 queries)
```

### Solution 1: JOIN FETCH (JPQL)

```java
// ✅ GOOD: Single query with JOIN FETCH
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @Query("SELECT a FROM Author a JOIN FETCH a.books")
    List<Author> findAllWithBooks();
}

// Usage - single query
List<Author> authors = authorRepository.findAllWithBooks();
```

### Solution 2: @EntityGraph

```java
// ✅ GOOD: EntityGraph for declarative fetching
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @EntityGraph(attributePaths = {"books"})
    List<Author> findAll();

    // Or with named graph
    @EntityGraph(value = "Author.withBooks")
    List<Author> findAllWithBooks();
}

// Define named graph on entity
@Entity
@NamedEntityGraph(
    name = "Author.withBooks",
    attributeNodes = @NamedAttributeNode("books")
)
public class Author {
    // ...
}
```

### Solution 3: Batch Fetching

```java
// ✅ GOOD: Batch fetching (Hibernate-specific)
@Entity
public class Author {

    @OneToMany(mappedBy = "author")
    @BatchSize(size = 25)  // Fetch 25 at a time
    private List<Book> books;
}

// Or globally in application.properties
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

### Solution 4: Panache JOIN FETCH (Quarkus)

```java
// ✅ GOOD: JOIN FETCH in Panache Active Record
public static List<Author> findAllWithBooks() {
    return find("FROM Author a JOIN FETCH a.books").list();
}

// ✅ GOOD: JOIN FETCH in Panache Repository
@ApplicationScoped
public class AuthorRepository implements PanacheRepository<Author> {

    public List<Author> findAllWithBooks() {
        return find("FROM Author a JOIN FETCH a.books").list();
    }

    public Optional<Author> findByIdWithBooks(Long id) {
        return find("FROM Author a JOIN FETCH a.books WHERE a.id = ?1", id)
            .firstResultOptional();
    }
}
```

### Detecting N+1

```yaml
# Spring Boot — Enable SQL logging
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

```properties
# Quarkus — Enable SQL logging
quarkus.hibernate-orm.log.sql=true
quarkus.log.category."org.hibernate.SQL".level=DEBUG
quarkus.log.category."org.hibernate.type.descriptor.sql".level=TRACE
```

---

## Lazy Loading

### FetchType Basics

```java
@Entity
public class Order {

    // LAZY: Load only when accessed (default for collections)
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;

    // EAGER: Always load immediately (default for @ManyToOne, @OneToOne)
    @ManyToOne(fetch = FetchType.EAGER)  // ⚠️ Usually bad
    private Customer customer;
}
```

### Best Practice: Default to LAZY

```java
// ✅ GOOD: Always use LAZY, fetch when needed
@Entity
public class Order {

    @ManyToOne(fetch = FetchType.LAZY)  // Override EAGER default
    private Customer customer;

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;
}
```

### LazyInitializationException

```java
// ❌ BAD: Accessing lazy field outside transaction
@Service
public class OrderService {

    public Order getOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();
    }
}

// In controller (no transaction)
Order order = orderService.getOrder(1L);
order.getItems().size();  // 💥 LazyInitializationException!
```

### Solutions for LazyInitializationException

**Solution 1: JOIN FETCH in query**
```java
// ✅ Fetch needed associations in query
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);
```

**Solution 2: @Transactional on service method**
```java
// ✅ Keep transaction open while accessing
@Service
public class OrderService {

    @Transactional(readOnly = true)
    public OrderDTO getOrderWithItems(Long id) {
        Order order = orderRepository.findById(id).orElseThrow();
        // Access within transaction
        int itemCount = order.getItems().size();
```

> **Quarkus note:** Use `jakarta.transaction.Transactional`, not `org.springframework.transaction.annotation.Transactional`.
> Quarkus does not support Open-Session-in-View — all lazy collection access must occur inside a `@Transactional` service method.

---

## Panache-Specific Patterns

### Active Record vs Repository

```
Use Panache Active Record (extends PanacheEntity) when:
  • Simple CRUD with domain logic naturally on the entity
  • Small teams comfortable with active record style

Use Panache Repository (implements PanacheRepository<T>) when:
  • Strict separation between data access and domain model
  • Need to @InjectMock the repository in unit tests (easier)
  • Multiple datasources or complex query composition
```

### Optimistic Locking

```java
@Entity
public class Product extends PanacheEntity {
    public String name;
    public BigDecimal price;

    @Version
    public Long version;  // Auto-incremented on each UPDATE
}

// Concurrent update — catch OptimisticLockException
@Transactional
public ProductResponse update(Long id, UpdateProductRequest req) {
    try {
        Product product = Product.findById(id);
        product.price = req.price();
        return ProductResponse.from(product);
    } catch (OptimisticLockException e) {
        throw new DomainException("Product was modified concurrently — please retry");
    }
}
```

### Pagination (Panache vs Spring Data)

```java
// Spring Data — returns Page<T>
Page<User> page = userRepository.findAll(PageRequest.of(0, 20));

// Panache — uses PanacheQuery (zero-indexed pages)
PanacheQuery<User> query = User.find("active", true);
List<User> items  = query.page(0, 20).list();   // first page
long total        = query.count();
int totalPages    = (int) Math.ceil((double) total / 20);
```

### Batch Operations

```java
// Bulk update (bypasses dirty checking — more efficient for large sets)
@Transactional
public void deactivateOldUsers(LocalDate cutoff) {
    User.update("active = false WHERE createdAt < ?1", cutoff);
}

// Bulk delete
@Transactional
public long deleteInactive() {
    return User.delete("active", false);
}
```
