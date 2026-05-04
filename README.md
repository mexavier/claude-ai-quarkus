# Claude Code Template for Quarkus Application

This template provides a structured starting point for Quarkus 3.x applications, optimized for Claude Code. It includes comprehensive agents, skills, and best practices to streamline enterprise Java development with Quarkus, MicroProfile, Panache, and the SmallRye ecosystem.

Clone this repository and use it with Claude Code to generate production-ready Quarkus applications.

```
.
├── .claude
│   ├── agents
│   │   ├── code-reviewer.md
│   │   ├── devops-engineer.md
│   │   ├── docker-expert.md
│   │   ├── kubernetes-specialist.md
│   │   ├── quarkus-engineer.md
│   │   └── security-engineer.md
│   ├── settings.local.json
│   └── skills
│       ├── README.md
│       ├── code-quality
│       │   └── SKILL.md
│       ├── design-patterns
│       │   └── SKILL.md
│       ├── jpa-patterns
│       │   └── SKILL.md
│       ├── logging-patterns
│       │   └── SKILL.md
│       └── quarkus
│           ├── SKILL.md
│           └── references
│               ├── cloud.md
│               ├── data.md
│               ├── security.md
│               ├── testing.md
│               └── web.md
├── CLAUDE.md
├── README.md
└── pom.xml
```

## Getting Started

```bash
# Start dev mode with live reload and Dev Services
./mvnw quarkus:dev

# Run tests (Dev Services starts PostgreSQL automatically)
./mvnw test

# Build an optimized JVM JAR
./mvnw package

# Build a native binary (requires GraalVM or Docker)
./mvnw package -Pnative

# Build native using Docker (no local GraalVM needed)
./mvnw package -Pnative -Dquarkus.native.container-build=true
```

### Dev UI

When running in dev mode, the Quarkus Dev UI is available at `http://localhost:8080/q/dev`.
It provides live access to configuration, health, metrics, and all registered extensions.

### Key Endpoints (after generating an application)

| Endpoint | Description |
|----------|-------------|
| `/q/health` | All health checks |
| `/q/health/live` | Liveness probe |
| `/q/health/ready` | Readiness probe |
| `/q/openapi` | OpenAPI specification |
| `/q/swagger-ui` | Swagger UI (dev mode) |
| `/q/metrics` | Prometheus metrics |
| `/q/dev` | Dev UI (dev mode) |

## What's Included

### Agents

| Agent | Purpose |
|-------|---------|
| `quarkus-engineer` | Enterprise Quarkus 3.x development — CDI, Panache, RESTEasy Reactive, SmallRye |
| `code-reviewer` | Code quality, security vulnerabilities, test coverage |
| `devops-engineer` | CI/CD pipelines, infrastructure automation |
| `docker-expert` | Multi-stage builds, container optimization, supply chain security |
| `kubernetes-specialist` | Cluster design, Helm, GitOps, RBAC |
| `security-engineer` | Threat modeling, DevSecOps, compliance automation |

### Skills

| Skill | Purpose |
|-------|---------|
| `quarkus` | Core Quarkus patterns — resources, services, Panache, testing |
| `quarkus/references/web` | RESTEasy Reactive, JAX-RS, OpenAPI, CORS |
| `quarkus/references/data` | Panache Active Record & Repository, Dev Services, Flyway |
| `quarkus/references/security` | SmallRye JWT, OIDC, `@RolesAllowed` |
| `quarkus/references/cloud` | MicroProfile Health/Config, Fault Tolerance, OpenTelemetry |
| `quarkus/references/testing` | `@QuarkusTest`, RestAssured, `@InjectMock`, native testing |
| `jpa-patterns` | N+1, lazy loading, Panache-specific patterns |
| `logging-patterns` | Structured JSON logging, MDC, SLF4J best practices |
| `code-quality` | Clean code, API contracts, null safety, performance |
| `design-patterns` | Builder, Factory, Strategy, Observer, Decorator |

## Technology Stack

- **Runtime**: Quarkus 3.33.1 (LTS)
- **Java**: 25 (LTS)
- **Jakarta EE**: 10
- **MicroProfile**: 6
- **Build**: Maven 3.9+
- **Data**: Hibernate ORM with Panache, Flyway
- **REST**: RESTEasy Reactive (JAX-RS 3.1)
- **Security**: SmallRye JWT / Quarkus OIDC
- **Testing**: JUnit 5, RestAssured, Dev Services (Testcontainers-based)
- **Native**: GraalVM / Mandrel

## Project Conventions (from CLAUDE.md)

- Group ID: `pl.piomin.services`
- Artifact ID: matches the parent directory name
- No Lombok — explicit Java records and constructors
- Constructor injection only (no `@Inject` on fields)
- `application.properties` for all configuration (no YAML unless explicitly added)
- `@Transactional` from `jakarta.transaction` (not Spring)
- Docker Compose file generated for all external dependencies
- CircleCI pipeline generated in `.circleci/`
- Semantic versioning — bump PATCH on each generated version
