# Architecture Overview

REST API layer for managing Owner and Pet resources within the Spring PetClinic microservices system.

The `petclinic-customers-service` is a Spring Boot microservice responsible for the customer domain — creating, reading, updating, and searching Owner records, as well as managing Pets associated with those owners. It exposes a REST API, integrates with Spring Cloud service discovery (Eureka), and provides Micrometer-based observability for production monitoring.

## Architecture

The service follows a standard layered Spring Boot architecture with clear separation between the web (controller) layer, the data access (repository) layer, and the domain model.

### Component Overview

| Layer | Components | Responsibility |
|-------|-----------|----------------|
| **Web** | `OwnerResource`, `PetResource` | HTTP endpoints for CRUD operations on Owners and Pets |
| **Mapping** | `OwnerEntityMapper`, `Mapper<T,R>` | Conversion between DTO request objects and JPA entities |
| **Model** | `Owner`, `Pet`, `PetType` | JPA entity definitions and domain logic |
| **Data Access** | `OwnerRepository`, `PetRepository` | Spring Data JPA interfaces for database operations |
| **Config** | `MetricConfig`, `CustomersServiceApplication` | Application bootstrap, service discovery, and Micrometer metrics setup |
| **Exception Handling** | `ResourceNotFoundException` | Standardized 404 responses for missing resources |

### Data Flow

REST requests enter through `OwnerResource` or `PetResource`, which delegate to the appropriate repository for persistence. Request payloads are mapped to entities via `OwnerEntityMapper` (implementing the generic `Mapper` interface). The `Owner` entity maintains a bidirectional one-to-many relationship with `Pet`, meaning pet data is eagerly loaded through its owning owner.

API request and response DTOs (`OwnerRequest`, `PetRequest`, `PetDetails`) decouple the external API contract from internal entity structure, allowing the persistence model to evolve independently.

### Key Design Decisions

- **Spring Data JPA repositories** — `OwnerRepository` and `PetRepository` extend `JpaRepository`, inheriting standard CRUD operations with zero boilerplate. Custom queries (e.g., `findByLastNameStartingWithIgnoreCase`) follow Spring Data naming conventions.
- **DTO separation** — `OwnerRequest` and `PetRequest` records serve as inbound DTOs, while `PetDetails` serves as an outbound view model for pet information. This prevents entity classes from leaking into the API contract.
- **Generic mapper interface** — The `Mapper<T,R>` interface with `OwnerEntityMapper` implementation provides a consistent pattern for request-to-entity conversion.
- **Metrics integration** — `MetricConfig` registers a common `"application"` tag on all metrics and enables `@Timed` aspect support, which the controllers use (e.g., `@Timed("petclinic.owner")`).
- **Service discovery** — `CustomersServiceApplication` enables a Eureka discovery client, allowing the service to register itself and be discovered by other microservices in the PetClinic system.

### System Architecture

<!-- diagram_key: architecture_system_architecture -->
```mermaid
flowchart TB
    subgraph External_Layer [External Layer]
        direction LR
        web_browser([Web Browser])
        mini_program([Mini Program])
    end

    subgraph Gateway_Layer [API Gateway Layer]
        direction LR
        duty_gateway{{Duty Gateway}}
        duty_auth{{Duty Auth}}
    end

    subgraph Business_Layer [Core Business Layer]
        direction LR
        subgraph Guest_MS [Guest Microservice]
            duty_callbacks[Callback Service]
            duty_portal[Portal Service]
            duty_auth_svc[Auth Service]
            youth_portal[Youth Portal]
        end

        subgraph Archive_MS [Archive Microservice]
            job_company[Job Company]
            job_youth[Job Youth]
        end
    end

    subgraph Core_Service [Core Service Layer]
        direction LR
        duty_core[Core Service]
        message_task[Message Task]
        duty_message[Message Management]
    end

    subgraph Technical_Framework [Technical Framework Layer]
        direction LR
        duty_architecture[Duty Architecture]
        duty_mappings[Duty Mappings]
        duty_jobs[Duty Jobs]
        duty_common[Duty Common]
    end

    subgraph Infrastructure [Infrastructure Layer]
        direction LR
        modulus_jwt[(Certificate Storage)]
        modulus_web[(Website Storage)]
        modulus_questionnaire[(Questionnaire Storage)]
        modulus_queue_job[(Rate Limit Queue)]
    end

    %% External to Gateway
    web_browser -->|"HTTP"| duty_gateway
    mini_program -->|"HTTP"| duty_gateway

    %% Gateway routing
    duty_gateway --> duty_portal
    duty_gateway --> duty_auth_svc
    duty_gateway --> youth_portal
    duty_gateway --> duty_callbacks
    duty_gateway --> job_company
    duty_gateway --> job_youth

    %% Business to Core
    duty_callbacks --> duty_core
    duty_portal --> duty_core
    duty_auth_svc --> duty_core
    youth_portal --> duty_core
    job_company --> duty_core
    job_youth --> duty_core
    duty_core --> message_task
    duty_core --> duty_message

    %% Core to Framework
    duty_core --> duty_architecture
    duty_core --> duty_common
    duty_architecture --> duty_mappings
    duty_architecture --> duty_jobs

    %% Framework to Infrastructure
    duty_mappings --> modulus_jwt
    duty_mappings --> modulus_web
    duty_mappings --> modulus_questionnaire
    duty_jobs --> modulus_queue_job
```

### Domain Model and DTOs

The entity model centers on `Owner` and `Pet`, with `PetType` as a reference entity. The DTO layer provides clean request/response boundaries.

<!-- diagram_key: class_domain_model_and_dtos -->
```mermaid
classDiagram
    direction TB

    %% Entity classes in model package
    namespace model {
        class Owner {
            <<Entity>>
            -id: Integer
            -firstName: String
            -lastName: String
            -address: String
            -city: String
            -telephone: String
            -pets: Set~Pet~
            +getId(): Integer
            +getFirstName(): String
            +getLastName(): String
            +getAddress(): String
            +getCity(): String
            +getTelephone(): String
            +setFirstName(firstName: String): void
            +setLastName(lastName: String): void
            +setAddress(address: String): void
            +setCity(city: String): void
            +setTelephone(telephone: String): void
            +getPets(): List~Pet~
            +addPet(pet: Pet): void
        }

        class Pet {
            <<Entity>>
            -id: Integer
            -name: String
            -birthDate: Date
            -type: PetType
            -owner: Owner
            +getId(): Integer
            +getName(): String
            +getBirthDate(): Date
            +getType(): PetType
            +getOwner(): Owner
            +setId(id: Integer): void
            +setName(name: String): void
            +setBirthDate(birthDate: Date): void
            +setType(type: PetType): void
            +setOwner(owner: Owner): void
        }

        class PetType {
            <<Entity>>
            -id: Integer
            -name: String
            +getId(): Integer
            +getName(): String
            +setId(id: Integer): void
            +setName(name: String): void
        }

        class OwnerRepository {
            <<Interface>>
            +findByLastNameStartingWithIgnoreCase(lastName: String): List~Owner~
        }

        class PetRepository {
            <<Interface>>
            +findPetTypes(): List~PetType~
            +findPetTypeById(typeId: int): Optional~PetType~
        }
    }

    %% DTOs in web package
    namespace web {
        class OwnerRequest {
            <<DTO>>
        }
        class PetRequest {
            <<DTO>>
        }
        class PetDetails {
            <<DTO>>
        }
    }

    %% Mapper classes in web.mapper package
    namespace web_mapper {
        class Mapper {
            <<Interface>>
            +map(response: E, request: R): E
        }

        class OwnerEntityMapper {
            <<Class>>
            +map(owner: Owner, request: OwnerRequest): Owner
        }
    }

    %% Relationships
    Owner "1" -- "0..*" Pet : owns
    Pet "0..*" -- "1" PetType : is type of

    OwnerRepository ..> Owner : returns
    PetRepository ..> Pet : returns
    PetRepository ..> PetType : returns

    OwnerEntityMapper ..|> Mapper : implements
    OwnerEntityMapper ..> OwnerRequest : uses
    OwnerEntityMapper ..> Owner : maps to
```

### Request Lifecycle

The following sequence diagram illustrates the complete CRUD flows for both Owner and Pet resources, showing the interaction between controllers, mappers, repositories, and the database.

<!-- diagram_key: sequence_owner_and_pet_crud_flows -->
```mermaid
sequenceDiagram
    %% Owner/Pet CRUD Flows - Sequence Diagram
    %% Participants derived from repository context
    actor Client as "Client (Browser/API)"
    participant OwnerRes as OwnerResource
    participant PetRes as PetResource
    participant OwnerRepo as OwnerRepository
    participant PetRepo as PetRepository
    participant OwnerMapper as OwnerEntityMapper

    autonumber

    Client->>OwnerRes: POST /owners
    activate OwnerRes
    OwnerRes->>OwnerMapper: map(OwnerRequest)
    OwnerMapper-->>OwnerRes: Owner entity
    OwnerRes->>OwnerRepo: save(Owner)
    OwnerRepo-->>OwnerRes: saved Owner
    OwnerRes-->>Client: 201 Created
    deactivate OwnerRes

    Client->>OwnerRes: GET /owners/{ownerId}
    activate OwnerRes
    OwnerRes->>OwnerRepo: findById(ownerId)
    alt Owner found
        OwnerRepo-->>OwnerRes: Owner
        OwnerRes-->>Client: 200 Owner
    else Owner not found
        OwnerRepo-->>OwnerRes: empty
        OwnerRes-->>Client: 404 Not Found
    end
    deactivate OwnerRes

    Client->>OwnerRes: GET /owners or /owners/search
    activate OwnerRes
    OwnerRes->>OwnerRepo: findAll or findByLastName
    OwnerRepo-->>OwnerRes: List of Owners
    OwnerRes-->>Client: 200 Owners
    deactivate OwnerRes

    Client->>OwnerRes: PUT /owners/{ownerId}
    activate OwnerRes
    OwnerRes->>OwnerMapper: map Owner with OwnerRequest
    OwnerMapper-->>OwnerRes: updated Owner
    OwnerRes->>OwnerRepo: save(Owner)
    OwnerRepo-->>OwnerRes: updated Owner
    OwnerRes-->>Client: 204 No Content
    deactivate OwnerRes

    Client->>PetRes: POST /owners/{ownerId}/pets
    activate PetRes
    PetRes->>OwnerRepo: findById(ownerId)
    alt Owner exists
        OwnerRepo-->>PetRes: Owner
        PetRes->>PetRepo: save(Pet)
        PetRepo-->>PetRes: saved Pet
        PetRes-->>Client: 201 Created
    else Owner not found
        OwnerRepo-->>PetRes: empty
        PetRes-->>Client: 404 Not Found
    end
    deactivate PetRes

    Client->>PetRes: PUT /owners/*/pets/{petId}
    activate PetRes
    PetRes->>PetRepo: save(Pet)
    PetRepo-->>PetRes: updated Pet
    PetRes-->>Client: 204 No Content
    deactivate PetRes

    Client->>PetRes: GET /owners/*/pets/{petId}
    activate PetRes
    PetRes->>PetRepo: findById(petId)
    alt Pet found
        PetRepo-->>PetRes: Pet
        PetRes-->>Client: 200 PetDetails
    else Pet not found
        PetRepo-->>PetRes: empty
        PetRes-->>Client: 404 Not Found
    end
    deactivate PetRes

    Client->>PetRes: GET /petTypes
    activate PetRes
    PetRes->>PetRepo: findPetTypes
    PetRepo-->>PetRes: List of PetType
    PetRes-->>Client: 200 PetTypes
    deactivate PetRes
```

## Project Structure

```
src/
├── main/
│   └── java/
│       └── org/springframework/samples/petclinic/customers/
│           ├── CustomersServiceApplication.java   # Spring Boot entry point with Eureka discovery
│           ├── config/
│           │   └── MetricConfig.java              # Micrometer metrics configuration
│           ├── model/
│           │   ├── Owner.java                     # Owner entity (JPA)
│           │   ├── OwnerRepository.java           # Spring Data repository for Owner
│           │   ├── Pet.java                       # Pet entity (JPA)
│           │   ├── PetRepository.java             # Spring Data repository for Pet
│           │   └── PetType.java                   # PetType reference entity
│           └── web/
│               ├── OwnerResource.java             # REST controller for /owners endpoints
│               ├── PetResource.java               # REST controller for /pets endpoints
│               ├── OwnerRequest.java              # Inbound DTO for Owner create/update
│               ├── PetRequest.java                # Inbound DTO for Pet create/update
│               ├── PetDetails.java                # Outbound view model for Pet responses
│               ├── ResourceNotFoundException.java # 404 exception handling
│               └── mapper/
│                   ├── Mapper.java                # Generic mapper interface
│                   └── OwnerEntityMapper.java     # OwnerRequest → Owner mapper
└── test/
    └── java/
        └── org/springframework/samples/petclinic/customers/
            └── web/
                └── PetResourceTest.java           # Unit tests for PetResource
```

### Key Packages

- **`config`** — Application configuration classes. `MetricConfig` sets up Micrometer common tags and the `TimedAspect` bean for method-level timing.
- **`model`** — JPA entities and Spring Data repository interfaces. `Owner` manages the one-to-many relationship with `Pet` via `addPet()` and sorted `getPets()`.
- **`web`** — REST controllers, DTOs, and the mapper sub-package. Controllers are annotated with `@RestController` and `@Timed` for metrics. The `ResourceNotFoundException` is annotated to produce HTTP 404 responses.
- **`web/mapper`** — The `Mapper<T,R>` interface defines a generic `map(target, source)` contract, implemented by `OwnerEntityMapper` for Owner DTO conversion.