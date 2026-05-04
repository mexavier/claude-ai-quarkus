# Cloud-Native Patterns (MicroProfile & SmallRye)

## MicroProfile Config

### Single Property

```java
@ApplicationScoped
public class PaymentService {

    @ConfigProperty(name = "payment.api.url")
    String paymentApiUrl;

    @ConfigProperty(name = "payment.timeout.seconds", defaultValue = "30")
    long timeoutSeconds;

    @ConfigProperty(name = "payment.enabled", defaultValue = "true")
    boolean enabled;
}
```

### Type-Safe Config Mapping (Preferred for Groups)

```java
@ConfigMapping(prefix = "app")
public interface AppConfig {
    PaymentConfig payment();
    DatabaseConfig database();
    FeatureFlags features();

    interface PaymentConfig {
        String apiUrl();
        long timeoutSeconds();
    }

    interface DatabaseConfig {
        int maxPoolSize();
    }

    interface FeatureFlags {
        boolean newCheckoutEnabled();
    }
}

// Inject the mapping
@ApplicationScoped
public class CheckoutService {
    private final AppConfig config;

    public CheckoutService(AppConfig config) {
        this.config = config;
    }

    public boolean isNewCheckout() {
        return config.features().newCheckoutEnabled();
    }
}
```

```properties
# application.properties
app.payment.api-url=https://api.payments.example.com
app.payment.timeout-seconds=30
app.database.max-pool-size=20
app.features.new-checkout-enabled=true

# Profile-based overrides
%dev.app.payment.api-url=http://localhost:8081
%test.app.features.new-checkout-enabled=false
```

---

## MicroProfile Health

```java
// Liveness: is the app running and not deadlocked?
@Liveness
@ApplicationScoped
public class AppLivenessCheck implements HealthCheck {

    @Override
    public HealthCheckResponse call() {
        return HealthCheckResponse.up("app");
    }
}

// Readiness: can the app serve traffic? (dependencies reachable)
@Readiness
@ApplicationScoped
public class DatabaseReadinessCheck implements HealthCheck {

    @Inject
    AgroalDataSource dataSource;

    @Override
    public HealthCheckResponse call() {
        try (Connection conn = dataSource.getConnection()) {
            conn.isValid(1);
            return HealthCheckResponse.up("database");
        } catch (Exception e) {
            return HealthCheckResponse.down("database");
        }
    }
}

// Startup: one-time initialization check
@Startup
@ApplicationScoped
public class MigrationStartupCheck implements HealthCheck {

    @Override
    public HealthCheckResponse call() {
        // Quarkus Flyway runs before health checks by default
        return HealthCheckResponse.up("migrations");
    }
}
```

Endpoints (available without authentication):
- `/q/health` — all checks
- `/q/health/live` — liveness only
- `/q/health/ready` — readiness only
- `/q/health/started` — startup only

```properties
# Expose health over management port (optional)
quarkus.management.enabled=true
quarkus.management.port=9000
```

---

## SmallRye Fault Tolerance

```java
@ApplicationScoped
public class ExternalApiService {

    @Inject
    @RestClient
    ExternalApiClient client;

    @Retry(maxRetries = 3, delay = 500, delayUnit = ChronoUnit.MILLIS,
           retryOn = {IOException.class, TimeoutException.class})
    @CircuitBreaker(requestVolumeThreshold = 10, failureRatio = 0.5,
                    delay = 5, delayUnit = ChronoUnit.SECONDS,
                    successThreshold = 2)
    @Timeout(value = 3, unit = ChronoUnit.SECONDS)
    @Fallback(fallbackMethod = "getDefaultData")
    public ExternalData getData(String id) {
        return client.fetch(id);
    }

    // Fallback method — same signature as the guarded method
    private ExternalData getDefaultData(String id) {
        return new ExternalData(id, "default", Instant.now());
    }

    // Bulkhead: limit concurrent calls
    @Bulkhead(value = 10, waitingTaskQueue = 5)
    public void processAsync(String payload) { ... }

    // Rate limit
    @RateLimit(value = 100, window = 1, windowUnit = ChronoUnit.MINUTES)
    public void rateLimitedOperation() { ... }
}
```

---

## OpenTelemetry (Distributed Tracing)

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-opentelemetry</artifactId>
</dependency>
```

```properties
quarkus.otel.exporter.otlp.endpoint=http://localhost:4317
quarkus.otel.service.name=user-service
quarkus.otel.resource.attributes=deployment.environment=production
```

```java
// Custom spans
@ApplicationScoped
public class PaymentService {

    @Inject
    Tracer tracer;

    @WithSpan("process-payment")
    public PaymentResult processPayment(PaymentRequest request) {
        Span span = tracer.spanBuilder("validate-card")
            .setAttribute("card.last4", request.last4())
            .startSpan();
        try (Scope scope = span.makeCurrent()) {
            return doProcess(request);
        } finally {
            span.end();
        }
    }
}
```

---

## Stork Service Discovery

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-stork</artifactId>
</dependency>
<dependency>
    <groupId>io.smallrye.stork</groupId>
    <artifactId>stork-service-discovery-consul</artifactId>
</dependency>
```

```properties
quarkus.stork.payment-service.service-discovery.type=consul
quarkus.stork.payment-service.service-discovery.consul-host=consul
quarkus.stork.payment-service.service-discovery.consul-port=8500
quarkus.stork.payment-service.load-balancer.type=round-robin
```

```java
@RegisterRestClient(baseUri = "stork://payment-service")
@Path("/payments")
public interface PaymentClient {
    @POST
    PaymentResponse charge(ChargeRequest request);
}
```

---

## Kubernetes Extension

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-kubernetes</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-container-image-docker</artifactId>
</dependency>
```

```properties
# application.properties
quarkus.kubernetes.deployment-target=kubernetes
quarkus.kubernetes.replicas=3
quarkus.kubernetes.liveness-probe.http-action-path=/q/health/live
quarkus.kubernetes.readiness-probe.http-action-path=/q/health/ready
quarkus.kubernetes.liveness-probe.initial-delay=10S
quarkus.kubernetes.readiness-probe.initial-delay=5S
quarkus.kubernetes.resources.requests.memory=256Mi
quarkus.kubernetes.resources.requests.cpu=250m
quarkus.kubernetes.resources.limits.memory=512Mi
quarkus.kubernetes.resources.limits.cpu=500m
quarkus.kubernetes.env.secrets=app-secrets
quarkus.kubernetes.labels.app=user-service
quarkus.kubernetes.labels.version=1.0.0

# Container image
quarkus.container-image.registry=registry.example.com
quarkus.container-image.group=pl.piomin.services
quarkus.container-image.name=user-service
quarkus.container-image.tag=1.0.0

# Build and push during mvn package
quarkus.container-image.build=true
quarkus.container-image.push=false
```

`./mvnw package` generates `target/kubernetes/kubernetes.yml` with Deployment, Service, and ConfigMap.

---

## Native Build

```bash
# Requires GraalVM installed locally
./mvnw package -Pnative

# No local GraalVM needed — uses Docker
./mvnw package -Pnative -Dquarkus.native.container-build=true

# Native + push container image
./mvnw package -Pnative \
  -Dquarkus.native.container-build=true \
  -Dquarkus.container-image.build=true
```

Quarkus generates `src/main/docker/Dockerfile.native` and `Dockerfile.jvm` automatically.

For classes accessed via reflection at native runtime:

```java
@RegisterForReflection
public class ExternalApiResponse { ... }
```

---

## Quick Reference

| Component | Extension / API |
|-----------|----------------|
| Config | `@ConfigProperty`, `@ConfigMapping` (built-in) |
| Health | `quarkus-smallrye-health`, `HealthCheck`, `@Liveness`/`@Readiness` |
| Fault Tolerance | `quarkus-smallrye-fault-tolerance`, `@Retry`, `@CircuitBreaker`, `@Fallback` |
| Tracing | `quarkus-opentelemetry`, `@WithSpan`, `Tracer` |
| Metrics | `quarkus-micrometer-registry-prometheus`, `MeterRegistry` |
| Service Discovery | `quarkus-smallrye-stork` + provider extension |
| Kubernetes Manifests | `quarkus-kubernetes` |
| Container Image | `quarkus-container-image-docker` |
| Native | `-Pnative`, `@RegisterForReflection` |
