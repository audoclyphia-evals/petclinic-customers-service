# petclinic-customers-service

A Spring Boot microservice managing customer and pet data for the PetClinic application.

![Tests](https://img.shields.io/badge/tests-passing-brightgreen.svg)

This service provides REST API endpoints for managing owners and their pets, backed by JPA repositories and a relational database. It integrates with Netflix Eureka for service discovery and Spring Cloud Config for centralized configuration.

## Project Overview

This service is one component of the PetClinic microservices architecture, responsible for the customer domain — owners and their pets. It exposes RESTful endpoints consumed by the API gateway (`petclinic-api-gateway`).

The service is organized into three primary layers:

| Layer | Package | Responsibility |
|-------|---------|----------------|
| **Configuration** | `config` | Metrics and monitoring setup via Micrometer |
| **Domain Model** | `model` | JPA entities (`Owner`, `Pet`, `PetType`) and repository interfaces |
| **Web / API** | `web` | REST controllers, DTOs, mappers, and exception handling |

### Key Components

- **`CustomersServiceApplication`** — Spring Boot entry point with Eureka discovery client enabled
- **`OwnerResource`** — REST controller for Owner CRUD operations
- **`PetResource`** — REST controller for Pet management and pet type retrieval
- **`MetricConfig`** — Micrometer metrics configuration with common tags and timed aspects
- **`ResourceNotFoundException`** — Custom runtime exception returning HTTP 404

```mermaid
flowchart TB
    %% MAIN APPLICATION ENTRY POINT
    app([CustomersServiceApplication]):::entryPoint
    
    %% CONFIGURATION LAYER
    subgraph Config_Tier [Configuration]
        metricConfig[MetricConfig]:::config
    end
    
    %% WEB/CONTROLLER LAYER
    subgraph Web_Tier [Web Layer]
        ownerResource[OwnerResource<br/>REST Controller]:::controller
        petResource[PetResource<br/>REST Controller]:::controller
        
        ownerRequest[OwnerRequest<br/>DTO Record]:::dto
        petRequest[PetRequest<br/>DTO Record]:::dto
        petDetails[PetDetails<br/>Response DTO]:::dto
        
        mapperInterface[Mapper Interface]:::mapper
        ownerMapper[OwnerEntityMapper<br/>Implements Mapper]:::mapper
        
        resourceNotFound[ResourceNotFoundException]:::exception
    end
    
    %% DATA ACCESS LAYER
    subgraph Data_Tier [Data Access Layer]
        ownerEntity[Owner Entity]:::entity
        petEntity[Pet Entity]:::entity
        petType[PetType Entity]:::entity
        
        ownerRepo[OwnerRepository Interface<br/>extends JpaRepository]:::repository
        petRepo[PetRepository Interface<br/>extends JpaRepository]:::repository
    end
    
    %% SERVICE DISCOVERY INTEGRATION
    serviceDiscovery([Service Discovery]):::external
    
    %% RELATIONSHIPS
    app -->|Starts| ownerResource
    app -->|Starts| petResource
    app -->|Configures| metricConfig
    
    ownerResource -->|Uses| ownerRepo
    ownerResource -->|Uses| ownerMapper
    petResource -->|Uses| petRepo
    
    ownerMapper -.->|Implements| mapperInterface
    ownerMapper -->|Converts| ownerRequest
    ownerMapper -->|Creates| ownerEntity
    
    ownerEntity -->|Contains many| petEntity
    petEntity -->|Belongs to| petType
    
    ownerRepo -->|Manages| ownerEntity
    petRepo -->|Manages| petEntity
    
    app -.->|Registers with| serviceDiscovery
    
    %% DEFINITIONS
    classDef entryPoint fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef config fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef controller fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef dto fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef mapper fill:#fce4ec,stroke:#b71c1c,stroke-width:2px
    classDef exception fill:#ffebee,stroke:#c62828,stroke-width:2px
    classDef entity fill:#e0f7fa,stroke:#006064,stroke-width:2px
    classDef repository fill:#f1f8e9,stroke:#33691e,stroke-width:2px
    classDef external fill:#e8eaf6,stroke:#283593,stroke-width:2px
    
    %% NOTES
    %% Layering: Entry Point → Configuration → Web Layer → Data Access → External
    %% All components grounded in context: CustomersServiceApplication (cluster_3), MetricConfig (cluster_5),
    %% OwnerResource/OwnerRequest/OwnerMapper (cluster_9), PetResource/PetRequest/PetDetails (cluster_1),
    %% Owner/OwnerRepository (cluster_4), Pet/PetRepository/PetType (cluster_2),
    %% Mapper interface (cluster_9), ResourceNotFoundException (cluster_0),
    %% Service discovery from context: "discovery client enabled" in cluster_3
```

## Features

The service provides the following core capabilities:

- **Owner Management** — Create, read, and update owner records via REST endpoints at `/owners`
- **Pet Management** — Create, retrieve, and update pets associated with owners via `/owners/{ownerId}/pets`
- **Pet Type Catalog** — Retrieve available pet types via `/petTypes`
- **Request Validation** — Input validation using Jakarta Bean Validation annotations (`@NotBlank`, `@Digits`, `@Min`)
- **DTO Mapping** — Dedicated `OwnerEntityMapper` and generic `Mapper` interface for converting between request DTOs and entity objects
- **Standardized Error Handling** — `ResourceNotFoundException` ensures consistent HTTP 404 responses for missing resources
- **Metrics and Monitoring** — Micrometer integration with Prometheus registry, common application tags, and `@Timed` aspects on controllers
- **Service Discovery** — Eureka client registration for dynamic service lookup within the microservices cluster
- **Distributed Tracing** — Spring Boot Zipkin starter for request tracing across services

## Requirements

To run and develop this service, you will need:

- **Java 17** or higher (required by Jakarta EE namespace)
- **Apache Maven 3.9+** for building the project
- **Database** — MySQL (production) or HSQLDB (included as a runtime dependency for development)
- **Network access** to a Eureka service registry instance (for service discovery)
- **Network access** to a Spring Cloud Config server (for centralized configuration)

## Installation

### Prerequisites

Verify Java and Maven are installed:

```bash
java -version   # Should show Java 17+
mvn -version    # Should show Maven 3.9+
```

### Build the Service

Clone the repository and build:

```bash
git clone https://github.com/audoclyphia-evals/petclinic-customers-service.git
cd petclinic-customers-service
mvn clean install
```

### Build the Docker Image

A `buildDocker` Maven profile is available:

```bash
mvn package -PbuildDocker
```

The service exposes port **8081** by default (configured via `docker.image.exposed.port` in `pom.xml`).

## Quick Start

Once the service is built, you can run it quickly using the default in-memory database.

1. **Start the service:**

```bash
mvn spring-boot:run
```

2. **Verify the service is running:**

```bash
curl http://localhost:8081/owners
```

Expected output: an empty JSON array `[]` if no owners exist yet.

3. **Create an owner:**

```bash
curl -X POST http://localhost:8081/owners \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Jane",
    "lastName": "Doe",
    "address": "123 Main St",
    "city": "Springfield",
    "telephone": "5551234567"
  }'
```

Expected output: the created owner object with an assigned `id`.

## Usage

With the service running, you can interact with the REST API endpoints. The following sections detail the available operations for owners and pets.

### Owner Endpoints

The `OwnerResource` controller (defined in the Project Overview) is mapped to `/owners` and provides the following operations:

| Method | Path | Description | Status |
|--------|------|-------------|--------|
| `POST` | `/owners` | Create a new owner | `201 Created` |
| `GET` | `/owners` | List all owners | `200 OK` |
| `GET` | `/owners/{ownerId}` | Retrieve a single owner by ID | `200 OK` |
| `PUT` | `/owners/{ownerId}` | Update an existing owner | `204 No Content` |

#### Create an Owner

```bash
curl -X POST http://localhost:8081/owners \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "John",
    "lastName": "Smith",
    "address": "456 Oak Ave",
    "city": "Portland",
    "telephone": "5559876543"
  }'
```

#### Retrieve All Owners

```bash
curl http://localhost:8081/owners
```

#### Retrieve a Specific Owner

```bash
curl http://localhost:8081/owners/1
```

#### Update an Owner

```bash
curl -X PUT http://localhost:8081/owners/1 \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "John",
    "lastName": "Smith",
    "address": "789 Pine Rd",
    "city": "Portland",
    "telephone": "5551112222"
  }'
```

If the owner is not found, the endpoint throws `ResourceNotFoundException` (defined in the Project Overview), which returns an HTTP 404 response.

### Pet Endpoints

The `PetResource` controller (defined in the Project Overview) provides pet management and pet type retrieval:

| Method | Path | Description | Status |
|--------|------|-------------|--------|
| `GET` | `/petTypes` | List all available pet types | `200 OK` |
| `POST` | `/owners/{ownerId}/pets` | Create a pet for an owner | `201 Created` |
| `GET` | `/owners/*/pets/{petId}` | Retrieve a specific pet's details | `200 OK` |
| `PUT` | `/owners/*/pets/{petId}` | Update a pet | `204 No Content` |

#### Retrieve Pet Types

```bash
curl http://localhost:8081/petTypes
```

#### Create a Pet for an Owner

```bash
curl -X POST http://localhost:8081/owners/1/pets \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Buddy",
    "birthDate": "2023-06-15",
    "type": { "id": 1 }
  }'
```

If the specified owner does not exist, the endpoint throws `ResourceNotFoundException` (defined in the Project Overview) with an HTTP 404 response.

#### Retrieve Pet Details

```bash
curl http://localhost:8081/owners/*/pets/1
```

The response is a `PetDetails` record (defined in the Project Overview) containing the pet's information.

#### Update a Pet

```bash
curl -X PUT http://localhost:8081/owners/*/pets/1 \
  -H "Content-Type: application/json" \
  -d '{
    "id": 1,
    "name": "Buddy",
    "birthDate": "2023-06-15",
    "type": { "id": 2 }
  }'
```

The following sequence diagrams illustrate the flow of these operations:

```mermaid
sequenceDiagram
    %% Pet Management Flow through PetResource controller
    %% Entities: PetResource, PetRepository, PetRequest, PetDetails, ResourceNotFoundException

    actor Client as REST Client
    participant PetResource as PetResource
    participant PetRepository as PetRepository
    participant PetRequest as PetRequest
    participant PetDetails as PetDetails
    participant ResourceNotFoundException as ResourceNotFoundException

    %% Create Pet operation
    rect rgb(240, 255, 240)
        note over Client, PetDetails: Create Pet Request Flow
        Client->>Client: Prepare PetRequest
        Client->>PetResource: POST /pets (PetRequest body)
        PetResource->>PetRequest: Validate PetRequest
        PetResource->>PetRepository: save(pet)
        PetRepository-->>PetResource: savedPet
        PetResource->>PetDetails: Transform to PetDetails
        PetDetails-->>PetResource: petDetails
        PetResource-->>Client: Return PetDetails (201 Created)
    end

    %% Retrieve Pet operation with error handling
    rect rgb(255, 240, 240)
        note over Client, PetDetails: Retrieve Pet Request Flow
        Client->>PetResource: GET /pets/{petId}
        
        alt Pet Not Found
            PetResource->>PetRepository: findById(petId)
            PetRepository-->>PetResource: Optional.empty()
            PetResource->>ResourceNotFoundException: throw new ResourceNotFoundException
            PetResource-->>Client: 404 Not Found
        else Pet Found
            PetResource->>PetRepository: findById(petId)
            PetRepository-->>PetResource: Optional.of(pet)
            PetResource->>PetDetails: Transform to PetDetails
            PetDetails-->>PetResource: petDetails
            PetResource-->>Client: Return PetDetails (200 OK)
        end
    end

    %% Update Pet operation
    rect rgb(240, 240, 255)
        note over Client, PetDetails: Update Pet Request Flow
        Client->>PetResource: PUT /pets/{petId} (PetRequest body)
        
        alt Pet Not Found
            PetResource->>PetRepository: findById(petId)
            PetRepository-->>PetResource: Optional.empty()
            PetResource->>ResourceNotFoundException: throw new ResourceNotFoundException
            PetResource-->>Client: 404 Not Found
        else Pet Found
            PetResource->>PetRepository: findById(petId)
            PetRepository-->>PetResource: Optional.of(pet)
            PetResource->>PetRepository: save(updatedPet)
            PetRepository-->>PetResource: savedPet
            PetResource->>PetDetails: Transform to PetDetails
            PetDetails-->>PetResource: petDetails
            PetResource-->>Client: Return PetDetails (200 OK)
        end
    end
```

```mermaid
sequenceDiagram
    autonumber
    participant Client as "REST Client"
    participant OwnerResource as "OwnerResource"
    participant OwnerMapper as "OwnerEntityMapper"
    participant OwnerRepo as "OwnerRepository"
    participant OwnerDB as "Owner Entity"
    participant PetDB as "Pet Entity"
    
    Note over Client, PetDB: Owner Management Flow - CRUD Operations
    
    %% Create Owner
    Client->>+OwnerResource: POST /owners (OwnerRequest)
    OwnerResource->>+OwnerMapper: map(new Owner(), request)
    OwnerMapper-->>-OwnerResource: Owner entity
    OwnerResource->>+OwnerRepo: save(owner)
    OwnerRepo->>+OwnerDB: persist(owner)
    OwnerDB-->>-OwnerRepo: persisted owner
    OwnerRepo-->>-OwnerResource: saved owner
    OwnerResource-->>-Client: 201 CREATED
    
    %% Update Owner
    Client->>+OwnerResource: PUT /owners/{id} (OwnerRequest)
    OwnerResource->>+OwnerRepo: findById(id)
    OwnerRepo->>+OwnerDB: find(id)
    OwnerDB-->>-OwnerRepo: existing owner
    OwnerRepo-->>-OwnerResource: Optional<Owner>
    OwnerResource->>+OwnerMapper: map(existingOwner, request)
    OwnerMapper-->>-OwnerResource: updated owner
    OwnerResource->>+OwnerRepo: save(updatedOwner)
    OwnerRepo->>+OwnerDB: persist(updatedOwner)
    OwnerDB-->>-OwnerRepo: persisted owner
    OwnerRepo-->>-OwnerResource: saved owner
    OwnerResource-->>-Client: 204 NO_CONTENT
    
    %% Get Owner by ID
    Client->>+OwnerResource: GET /owners/{id}
    OwnerResource->>+OwnerRepo: findById(id)
    OwnerRepo->>+OwnerDB: find(id)
    OwnerDB-->>-OwnerRepo: owner with pets
    OwnerRepo-->>-OwnerResource: Optional<Owner>
    OwnerResource-->>-Client: 200 OK (Owner with Pets)
    
    %% Get All Owners
    Client->>+OwnerResource: GET /owners
    OwnerResource->>+OwnerRepo: findAll()
    OwnerRepo->>+OwnerDB: findAll
    OwnerDB-->>-OwnerRepo: list of owners
    OwnerRepo-->>-OwnerResource: List<Owner>
    OwnerResource-->>-Client: 200 OK (List of Owners)
    
    Note right of OwnerDB: Owner entity has OneToMany relationship with Pet entity (cascade=ALL#59; EAGER fetch)
```