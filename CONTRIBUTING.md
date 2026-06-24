# Contributing Guidelines

Guidelines for contributing to the `petclinic-customers-service` module, a Spring Boot microservice providing REST APIs for managing Owner and Pet resources in the Spring PetClinic system.

## Overview

The `petclinic-customers-service` is a Java-based Spring Boot microservice that exposes REST endpoints for CRUD operations on Owner and Pet domain entities. It serves as the customer and pet management layer within the broader Spring PetClinic system, providing data transfer objects, domain models, exception handling, and observability metrics.

Contributions to this module should align with its existing architectural patterns: Spring Boot conventions, RESTful API design, DTO-to-entity mapping, and Micrometer-based metrics. The service uses Maven as its package manager, and all code must conform to the established Java style and test coverage expectations.

## Development

### Prerequisites

- Java (JDK) installed
- Maven (package manager) for dependency management and build commands
- An IDE that supports Java development (e.g., IntelliJ IDEA or Eclipse)

### Setting Up the Development Environment

Clone the repository and navigate to the project root:

```bash
git clone <repository-url>
cd petclinic-customers-service
```

Build the project using Maven:

```bash
mvn clean install
```

### Project Structure

```
├── pom.xml
└── src/
    ├── main/
    │   └── java/
    │       └── org/
    │           └── springframework/
    │               └── samples/
    │                   └── petclinic/
    │                       └── customers/
    │                           ├── config/
    │                           ├── model/
    │                           └── web/
    │                               └── mapper/
    └── test/
        └── java/
            └── org/
                └── springframework/
                    └── samples/
                        └── petclinic/
                            └── customers/
                                └── web/
```

- **Source code** is located under `src/main/java/org/springframework/samples/petclinic/customers/`.
- **Test code** is located under `src/test/java/org/springframework/samples/petclinic/customers/web/`.
- The module is built with **Maven**; no additional scripts or make targets are available.

### Key Architectural Components

| Component | Description |
|---|---|
| **OwnerResource** | REST controller exposing CRUD endpoints for Owner entities under `/owners`. |
| **PetResource** | REST controller handling pet-related HTTP endpoints for CRUD operations. |
| **PetType / Pet** | Entity classes defining pet domain models. |
| **OwnerRequest / PetRequest** | DTO records for incoming request validation. |
| **PetDetails** | View model record for API response representation. |
| **OwnerRepository / PetRepository** | Spring Data JPA repository interfaces for data access. |
| **ResourceNotFoundException** | Custom runtime exception class annotated to return HTTP 404 status. |
| **MetricConfig** | Configuration class for setting up Micrometer metrics with common tags and timed aspects. |

### Architecture Diagram

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

### Code Style

- Follow standard Java coding conventions.
- Use Lombok or records where the existing codebase applies them (e.g., `OwnerRequest`, `PetRequest`, `PetDetails` are defined as records).
- Ensure REST controllers follow consistent naming and response patterns as established in `OwnerResource` and `PetResource`.

## Testing

### Running Tests

Execute all tests with Maven:

```bash
mvn test
```

### Test Location and Structure

- Tests reside under `src/test/java/org/springframework/samples/petclinic/customers/web/`.
- Test classes follow standard Java naming conventions with a `Test` suffix (e.g., `PetResourceTest`).
- Unit tests are used to validate REST endpoint behavior for resources such as Pets and Owners.

### Writing New Tests

- Place new test classes in the corresponding package under `src/test/java/org/springframework/samples/petclinic/customers/web/`.
- Follow existing test patterns: focus on endpoint validation and resource response correctness.
- Ensure that any new controller methods, exception handlers, or mapper logic are accompanied by unit tests.
- Tests should cover both happy-path and error scenarios (e.g., `ResourceNotFoundException` for missing resources).

### Code Quality

- No pre-commit hooks or linting tools are configured in this repository.
- Run `mvn test` before submitting changes to verify all tests pass.

## Contributing

Contributions are accepted via pull requests to the `petclinic-customers-service` repository.

### Workflow

1. **Fork** the repository.
2. **Create a branch** for your changes:

   ```bash
   git checkout -b feature/your-change
   ```

3. **Make your changes**, ensuring they follow the existing code style and include appropriate tests.
4. **Run all tests** to verify nothing is broken:

   ```bash
   mvn test
   ```

5. **Commit** your changes with clear, descriptive messages following the Conventional Commits format:

   ```
   feat: add endpoint for finding owners by last name
   fix: correct pet type serialization in response
   docs: update contributing guidelines
   ```

6. **Push** your branch and **open a pull request** against the main branch.

### Guidelines

- Keep pull requests focused on a single concern.
- Update or add tests alongside any new functionality or bug fixes.
- Ensure code compiles and passes all existing tests before submitting.
- When adding REST endpoints, follow the existing controller patterns in `OwnerResource` and `PetResource`.
- When modifying domain models (e.g., `Owner`, `Pet`, `PetType`), verify that all DTOs and mappers remain consistent.

### Reporting Issues

Issues should be reported via the repository's issue tracker. When reporting, include:

- A clear description of the problem or feature request.
- Steps to reproduce (for bugs).
- Relevant log output or error messages.