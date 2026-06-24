# Contributing Guidelines

Guidelines for contributing to the PetClinic Customers Service microservice.

The `petclinic-customers-service` is a Spring Boot microservice responsible for managing owners and pets within the Spring PetClinic application. It exposes REST APIs for owner and pet CRUD operations, persists data via JPA, and registers with Eureka for service discovery. Contributors should understand the domain model, REST layer, and testing conventions outlined below before submitting changes.

```mermaid
flowchart TB
    %% System Architecture for petclinic-customers-service microservice
    %% Evidence-based diagram using code context

    %% External Systems Tier
    subgraph External_Systems [External Systems]
        SD[Service Discovery]
        MB[Metrics Backend]
    end

    %% Application Entry Point
    subgraph Application_Tier [Application Entry Point]
        CSA[CustomersServiceApplication]
    end

    %% Web Layer Tier
    subgraph Web_Layer [Web Layer]
        OR[OwnerResource]
        PR[PetResource]
        RNF[ResourceNotFoundException]
        OEM[OwnerEntityMapper]
        MAPPER[Mapper]
    end

    %% Model Layer Tier
    subgraph Model_Layer [Model Layer]
        OWN[Owner]
        PET[Pet]
        PT[PetType]
        ORR[OwnerRepository]
        PRP[PetRepository]
    end

    %% Configuration Tier
    subgraph Config_Layer [Configuration Layer]
        MC[MetricConfig]
    end

    %% DTO/View Model Tier
    subgraph DTO_Tier [DTOs and View Models]
        OREQ[OwnerRequest]
        PREQ[PetRequest]
        PDET[PetDetails]
    end

    %% Relationships
    CSA -->|enables discovery| SD
    CSA -->|configures| MC
    MC -->|exports metrics| MB

    OR -->|uses| ORR
    OR -->|uses| OEM
    PR -->|uses| PRP
    PR -->|uses| ORR
    RNF -.->|throws| OR
    RNF -.->|throws| PR

    OEM -->|implements| MAPPER
    OEM -->|maps to| OWN
    OR -->|accepts| OREQ
    PR -->|accepts| PREQ
    PR -->|returns| PDET

    ORR -->|queries| OWN
    PRP -->|queries| PET
    PRP -->|queries| PT
    OWN -->|has many| PET
    PET -->|belongs to| OWN
    PET -->|is of type| PT

    %% External connections
    SD -.->|registers| CSA
    MB -.->|receives metrics from| CSA
```

## Development

### Prerequisites

- **Java 17+** (Spring Boot parent POM determines the exact version)
- **Maven 3.x** for building and dependency management
- Access to a running **MySQL** instance (production) or **HSQLDB** (default for local/test)

### Project Structure

The source code is organized under the `org.springframework.samples.petclinic.customers` package:

```
src/main/java/org/springframework/samples/petclinic/customers/
├── CustomersServiceApplication.java   # Application entry point
├── config/
│   └── MetricConfig.java              # Micrometer metrics configuration
├── model/
│   ├── Owner.java                     # Owner JPA entity
│   ├── OwnerRepository.java           # Owner data access interface
│   ├── Pet.java                       # Pet JPA entity
│   ├── PetRepository.java             # Pet data access interface
│   └── PetType.java                   # PetType JPA entity
└── web/
    ├── OwnerResource.java             # Owner REST controller
    ├── OwnerRequest.java              # Owner request DTO
    ├── PetResource.java               # Pet REST controller
    ├── PetRequest.java                # Pet request DTO
    ├── PetDetails.java                # Pet API response record
    ├── ResourceNotFoundException.java  # Custom 404 exception
    └── mapper/
        ├── Mapper.java                # Generic mapper interface
        └── OwnerEntityMapper.java     # Owner request-to-entity mapper
```

### Building

```bash
# Build the project (skip tests)
mvn clean package -DskipTests

# Build with tests
mvn clean package
```

### Running Locally

```bash
# Run the application (uses HSQLDB by default for local dev)
mvn spring-boot:run
```

The service exposes its REST API on a port configured in `application.properties` (default is **8081** for Docker). When running locally without infrastructure, HSQLDB provides an in-memory database.

### Key Dependencies

| Dependency | Purpose |
|---|---|
| `spring-boot-starter-data-jpa` | JPA/Hibernate data access |
| `spring-boot-starter-webmvc` | REST API layer |
| `spring-boot-starter-actuator` | Health checks and metrics endpoints |
| `spring-cloud-starter-netflix-eureka-client` | Service discovery registration |
| `spring-cloud-starter-config` | Externalized configuration |
| `spring-boot-starter-zipkin` | Distributed tracing |
| `mysql-connector-j` | MySQL JDBC driver (runtime) |
| `hsqldb` | In-memory database for dev/test |
| `micrometer-registry-prometheus` | Prometheus metrics export |

### Domain Model

The service manages three core entities:

- **`Owner`** — Represents a pet owner with fields for `firstName`, `lastName`, `address`, `city`, and `telephone`. Maintains a one-to-many relationship with `Pet` via the `pets` set. The `addPet()` method manages the bidirectional relationship between owners and pets.
- **`Pet`** — Represents an individual pet with a name, birth date, type, and owner reference.
- **`PetType`** — A lookup entity for pet categories (e.g., cat, dog).

Data access is provided by `OwnerRepository` and `PetRepository`, both extending `JpaRepository`. The `OwnerRepository` supports searching owners by last name prefix via `findByLastNameStartingWithIgnoreCase`. When a requested owner or pet is not found, the `ResourceNotFoundException` is thrown, returning an HTTP 404 status.

```mermaid
sequenceDiagram
    autonumber
    participant Client as "HTTP Client"
    participant OR as "OwnerResource"
    participant ORQ as "OwnerRequest"
    participant OEM as "OwnerEntityMapper"
    participant Repo as "OwnerRepository"
    participant O as "Owner"
    participant RNF as "ResourceNotFoundException"

    Note over Client,RNF: Owner Management CRUD Flow

    %% Create Owner Flow
    Client->>OR: POST /owners (OwnerRequest)
    activate OR
    OR->>ORQ: Validate OwnerRequest
    activate ORQ
    ORQ-->>OR: Validated OwnerRequest
    deactivate ORQ
    OR->>OEM: map(new Owner(), ownerRequest)
    activate OEM
    OEM->>O: setFirstName/LastName/Address/City/Telephone
    activate O
    O-->>OEM: Owner
    deactivate O
    OEM-->>OR: Owner
    deactivate OEM
    OR->>Repo: save(owner)
    activate Repo
    Repo-->>OR: Saved Owner
    deactivate Repo
    OR-->>Client: 201 Created (Owner)
    deactivate OR

    %% Find Single Owner Flow
    Client->>OR: GET /owners/{ownerId}
    activate OR
    OR->>Repo: findById(ownerId)
    activate Repo
    Repo-->>OR: Optional Owner
    deactivate Repo
    alt Owner found
        OR-->>Client: Owner
    else Owner not found
        OR->>RNF: throw ResourceNotFoundException
        activate RNF
        RNF-->>Client: 404 Not Found
        deactivate RNF
    end
    deactivate OR

    %% Search Owners Flow
    Client->>OR: GET /owners/search?lastName=*
    activate OR
    OR->>Repo: findByLastNameStartingWithIgnoreCase(lastName)
    activate Repo
    Repo-->>OR: List of Owners
    deactivate Repo
    OR-->>Client: List of Owners
    deactivate OR

    %% Update Owner Flow
    Client->>OR: PUT /owners/{ownerId} (OwnerRequest)
    activate OR
    OR->>ORQ: Validate OwnerRequest
    activate ORQ
    ORQ-->>OR: Validated OwnerRequest
    deactivate ORQ
    OR->>Repo: findById(ownerId)
    activate Repo
    Repo-->>OR: Optional Owner
    deactivate Repo
    alt Owner found
        OR->>OEM: map(existingOwner, ownerRequest)
        activate OEM
        OEM->>O: Update fields
        activate O
        O-->>OEM: Updated Owner
        deactivate O
        OEM-->>OR: Owner
        deactivate OEM
        OR->>Repo: save(owner)
        activate Repo
        Repo-->>OR: Saved Owner
        deactivate Repo
        OR-->>Client: 204 No Content
    else Owner not found
        OR->>RNF: throw ResourceNotFoundException
        activate RNF
        RNF-->>Client: 404 Not Found
        deactivate RNF
    end
    deactivate OR
```

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client
    participant PR as PetResource
    participant Pet as Pet
    participant PRepo as PetRepository
    participant ORepo as OwnerRepository
    participant PT as PetType

    rect rgb(230, 245, 255)
    note over Client, PT: Create Pet Flow
    Client->>+PR: POST /owners/{ownerId}/pets
    PR->>ORepo: Find owner by ID
    ORepo-->>PR: Return owner
    PR->>Pet: Create pet from request
    PR->>PRepo: Save pet
    PRepo-->>PR: Return saved pet
    PR-->>-Client: 201 Created
    end

    rect rgb(255, 245, 230)
    note over Client, PT: Update Pet Flow
    Client->>+PR: PUT /owners/*/pets/{petId}
    PR->>PRepo: Find pet by ID
    PRepo-->>PR: Return existing pet
    PR->>Pet: Update pet fields
    PR->>PRepo: Save updated pet
    PRepo-->>PR: Return updated pet
    PR-->>-Client: 204 No Content
    end

    rect rgb(230, 255, 230)
    note over Client, PT: Find Pet Flow
    Client->>+PR: GET owners/*/pets/{petId}
    PR->>PRepo: Find pet by ID
    PRepo-->>PR: Return pet
    PR-->>-Client: 200 OK (PetDetails)
    end

    rect rgb(255, 230, 245)
    note over Client, PT: Get PetTypes Flow
    Client->>+PR: GET /petTypes
    PR->>PRepo: findPetTypes()
    PRepo->>PT: Query pet types
    PT-->>PRepo: Return pet type list
    PRepo-->>PR: Return pet type list
    PR-->>-Client: 200 OK (List of PetType)
    end
```

## Testing

### Running Tests

```bash
# Run all tests
mvn test

# Run a specific test class
mvn test -Dtest=PetResourceTest

# Run with coverage (requires JaCoCo plugin configuration)
mvn test jacoco:report
```

### Test Structure

Tests reside under `src/test/java/org/springframework/samples/petclinic/customers/`, mirroring the main source layout.

| Test File | Coverage |
|---|---|
| `web/PetResourceTest` | `PetResource` controller endpoints — verifies pet type retrieval, pet creation, and pet update operations using `@WebMvcTest` with mocked dependencies |

### Writing New Tests

- Tests use **JUnit 5** (`junit-jupiter-api` and `junit-jupiter-engine`).
- Assertions are written with **AssertJ** (`assertj-core`).
- Controller tests use **`spring-boot-starter-webmvc-test`** for mock MVC testing.
- Test classes are named `{ClassName}Test` and placed in the same package as the class under test.
- Dependencies such as `OwnerRepository`, `PetRepository`, and `OwnerEntityMapper` should be mocked using `@MockBean`.

```java
// Example test pattern for a REST controller
@WebMvcTest(PetResource.class)
class PetResourceTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldReturnPetTypes() throws Exception {
        mockMvc.perform(get("/petTypes"))
            .andExpect(status().isOk());
    }
}
```

When adding new endpoints or modifying existing ones (such as the `OwnerResource` or `PetResource` controllers), ensure corresponding test cases are added or updated.

## Contributing

### Workflow

1. Fork the repository and create a feature branch from `main`.
2. Make changes following the project conventions outlined below.
3. Run `mvn test` to verify all tests pass.
4. Submit a pull request with a clear description of the changes.

### Code Style

- Follow standard Java coding conventions.
- Use **4 spaces** for indentation (no tabs).
- Entity classes use JPA annotations at the class and field level.
- REST controllers are placed in the `web` package; request/response DTOs live alongside them.
- Entity-to-DTO mapping uses the `Mapper<TReq, TEntity>` interface pattern, with concrete implementations in the `web/mapper` package.

### Commit Messages

- Use clear, imperative commit messages (e.g., "Add pet type lookup endpoint").
- Reference related issues where applicable.
- Keep commits focused — one logical change per commit.

### Pull Request Guidelines

- PRs should target the `main` branch.
- Include context on **what** changed and **why** in the PR description.
- Ensure the build passes (`mvn clean package`) before requesting review.
- Add or update tests for any new functionality or bug fixes.

### Reporting Issues

Issues should be reported in the project's issue tracker with:
- A descriptive title
- Steps to reproduce (for bugs)
- Expected vs. actual behavior
- Environment details (Java version, OS, database)