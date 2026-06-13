# REST API Reference and Data Models

![Tests](https://img.shields.io/badge/tests-passing-brightgreen.svg)

REST API endpoints and domain data models for the PetClinic Customers Service — a Spring Boot microservice responsible for managing owners, pets, and pet type information.

## Overview

The PetClinic Customers Service provides the customer-facing data layer of the Pet Clinic application. It exposes RESTful endpoints for CRUD operations on **Owner** and **Pet** resources, backed by JPA entities persisted to a relational database. The service handles owner registration, pet enrollment, and pet type lookups.

The codebase is organized into three primary packages:

- **`model`** — JPA entities (`Owner`, `Pet`, `PetType`) and Spring Data repositories (`OwnerRepository`, `PetRepository`)
- **`web`** — REST controllers (`OwnerResource`, `PetResource`), request DTOs (`OwnerRequest`, `PetRequest`), response DTOs (`PetDetails`), and exception handling (`ResourceNotFoundException`)
- **`web/mapper`** — Object mapping between request DTOs and entity objects (`Mapper` interface, `OwnerEntityMapper`)
- **`config`** — Application configuration including Micrometer metrics (`MetricConfig`)

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

- **Owner management** — Create, retrieve, update, and list owner records through dedicated REST endpoints under `/owners`
- **Pet management** — Create and update pets associated with an owner, and retrieve individual pet details via endpoints under `/owners/{ownerId}/pets`
- **Pet type lookup** — Retrieve the list of available pet types from the `/petTypes` endpoint
- **Request validation** — Input DTOs (`OwnerRequest`, `PetRequest`) are validated with Jakarta Bean Validation constraints (e.g., `@NotBlank`, `@Min`, `@Digits`)
- **Standardized error handling** — `ResourceNotFoundException` provides consistent HTTP 404 responses when requested resources do not exist
- **Metrics and observability** — Micrometer integration with Prometheus registry, common tags, and `@Timed` aspect support for controller-level metrics
- **Service discovery** — Eureka client registration via Spring Cloud for integration with the broader microservices ecosystem
- **Distributed tracing** — Spring Cloud Sleuth / Zipkin support for request tracing across services

## Requirements

- **Java** 17 or higher
- **Maven** 3.6+
- A relational database (MySQL for production; HSQLDB available for local development)
- Network access to a Eureka discovery server (for service registration in a microservices deployment)

## Installation

```bash
# Clone the repository
git clone https://github.com/audoclyphia-evals/petclinic-customers-service.git
cd petclinic-customers-service

# Build the project
mvn clean package

# Run the service (default port 8081)
mvn spring-boot:run
```

To build without running tests:

```bash
mvn clean package -DskipTests
```

To build the Docker image (requires the `buildDocker` profile):

```bash
mvn clean package -PbuildDocker
```

## Quick Start

1. Ensure Java 17+ and Maven are installed.
2. Build and start the service:

```bash
mvn spring-boot:run
```

3. Verify the service is running by requesting all owners:

```bash
curl http://localhost:8081/owners
```

Expected output (empty list if no data exists):

```json
[]
```

4. Create a new owner:

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

Expected output (returns the created owner with an assigned ID):

```json
{
  "id": 1,
  "firstName": "Jane",
  "lastName": "Doe",
  "address": "123 Main St",
  "city": "Springfield",
  "telephone": "5551234567",
  "pets": []
}
```

## Usage

### Owner Endpoints

The `OwnerResource` controller manages all owner operations under the `/owners` base path. The controller class is annotated with `@Timed("petclinic.owner")` for Micrometer metrics collection.

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

#### Create an Owner

```
POST /owners
```

Creates a new owner record. Returns the persisted owner with an auto-generated ID.

**Request body:** `OwnerRequest` (validated)

**Response:** `201 Created` — `Owner`

```bash
curl -X POST http://localhost:8081/owners \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "John",
    "lastName": "Smith",
    "address": "456 Oak Ave",
    "city": "Madison",
    "telephone": "5559876543"
  }'
```

#### Retrieve All Owners

```
GET /owners
```

Returns a list of all owner records.

**Response:** `200 OK` — `List<Owner>`

```bash
curl http://localhost:8081/owners
```

#### Retrieve a Single Owner

```
GET /owners/{ownerId}
```

Returns a single owner by ID. The `ownerId` path variable must be greater than or equal to 1.

**Response:** `200 OK` — `Owner` | `404 Not Found`

```bash
curl http://localhost:8081/owners/1
```

#### Update an Owner

```
PUT /owners/{ownerId}
```

Updates an existing owner. If the owner is not found, a `ResourceNotFoundException` is thrown, resulting in a 404 response.

**Request body:** `OwnerRequest` (validated)

**Response:** `204 No Content` | `404 Not Found`

```bash
curl -X PUT http://localhost:8081/owners/1 \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "John",
    "lastName": "Smith-Jones",
    "address": "789 Elm Blvd",
    "city": "Madison",
    "telephone": "5551112222"
  }'
```

### Pet Endpoints

The `PetResource` controller handles pet-related operations. It manages pet creation and updates within the context of an owner, and provides pet type lookups.

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

#### Retrieve Pet Types

```
GET /petTypes
```

Returns the list of all available pet types (e.g., cat, dog, lizard).

**Response:** `200 OK` — `List<PetType>`

```bash
curl http://localhost:8081/petTypes
```

#### Create a Pet for an Owner

```
POST /owners/{ownerId}/pets
```

Creates a new pet and associates it with the specified owner. If the owner does not exist, a `ResourceNotFoundException` is returned.

**Request body:** `PetRequest` (validated)

**Response:** `201 Created` — `Pet` | `404 Not Found`

```bash
curl -X POST http://localhost:8081/owners/1/pets \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Buddy",
    "birthDate": "2023-06-15",
    "type": { "id": 1 }
  }'
```

#### Retrieve Pet Details

```
GET /owners/*/pets/{petId}
```

Returns the detailed view of a pet. The `*` wildcard indicates that the owner ID is not used for lookup — the pet is retrieved directly by its own ID. The response uses the `PetDetails` record, which presents a formatted view of the pet entity.

**Response:** `200 OK` — `PetDetails`

```bash
curl http://localhost:8081/owners/1/pets/1
```

#### Update a Pet

```
PUT /owners/*/pets/{petId}
```

Updates an existing pet's information. The pet ID is derived from the request body.

**Request body:** `PetRequest` (validated)

**Response:** `204 No Content`

```bash
curl -X PUT http://localhost:8081/owners/1/pets/1 \
  -H "Content-Type: application/json" \
  -d '{
    "id": 1,
    "name": "Buddy Jr.",
    "birthDate": "2023-06-15",
    "type": { "id": 2 }
  }'
```

### Error Handling

The `ResourceNotFoundException` class is a `RuntimeException` annotated to produce HTTP 404 responses. It is thrown by both `OwnerResource` and `PetResource` when a requested owner or pet cannot be found.

```java
// Thrown when an owner is not found
throw new ResourceNotFoundException("Owner " + ownerId + " not found");
```

### Data Models

| Entity | Table | Key Fields |
|--------|-------|------------|
| `Owner` | `owners` | `id`, `firstName`, `lastName`, `address`, `city`, `telephone` |
| `Pet` | (mapped by JPA) | `id`, `name`, `birthDate`, `owner` (relationship) |
| `PetType` | (mapped by JPA) | `id`, `name` |

The `Owner` entity maintains a `@OneToMany` relationship with `Pet`, eagerly fetched and cascade-enabled. Pets are sorted alphabetically by name when accessed via `Owner.getPets()`.