# petclinic-customers-service

The Customers Service manages owner and pet data in the Spring PetClinic microservices architecture. It provides REST endpoints for creating, reading, and updating owner and pet entities. Built with Spring Boot and Spring Data JPA, it integrates with Eureka for service discovery and exposes metrics for monitoring. The service runs on port 8081 by default and is part of a broader ecosystem that includes an API gateway, vets service, and visits service, with the `petclinic-api-gateway` routing requests to this service.

## Project Overview

The `petclinic-customers-service` is a Spring Boot microservice responsible for persisting and serving owner and pet data within the Spring PetClinic application. It handles the customer domain by providing data access and REST API layers for owner and pet entities.

## Features

### Owner Management

- **CRUD Operations**: Create, read, update, and list owner records via REST endpoints under `/owners`.
- **Input Validation**: Owner request DTO (`OwnerRequest`) with field-level validation using `@NotBlank` and `@Digits` annotations.
- **Entity Mapping**: Automatic mapping between request objects and JPA entities using the `OwnerEntityMapper`.

### Pet Management

- **CRUD Operations**: Create, update, and retrieve pet records via REST endpoints under `/owners/{ownerId}/pets`.
- **Pet Type Listing**: Retrieve a list of all available pet types via `/petTypes`.
- **Request/Response DTOs**: Separate DTOs (`PetRequest` for input, `PetDetails` for output) for clean API boundaries.

### Infrastructure and Monitoring

- **Service Discovery**: Integration with Netflix Eureka for service registration and discovery.
- **Distributed Tracing**: Zipkin integration for tracing requests across services.
- **Metrics and Monitoring**: Micrometer with Prometheus registry for metrics collection; custom common tags configured via `MetricConfig`.
- **Configuration Management**: Spring Cloud Config support for externalized configuration.
- **Health Management**: Spring Boot Actuator endpoints for health checks and service status.

### Technical Features

- **Spring Data JPA**: Simplified data access with repository interfaces for `Owner` and `Pet` entities.
- **Custom Exception Handling**: `ResourceNotFoundException` for returning HTTP 404 responses when resources are not found.
- **Testing Support**: Unit tests for pet endpoints with JUnit and AssertJ.

## Requirements

- **Java**: JDK 17 or higher
- **Maven**: 3.6+ for building the project
- **Database**: MySQL (production) or HSQLDB (local development/testing)
- **Spring Boot**: Parent project version 4.0.1 (Spring PetClinic Microservices)
- **External Services**:
  - Spring Cloud Config Server for configuration
  - Netflix Eureka Server for service discovery
  - Zipkin Server for distributed tracing

## Installation

To clone and build the project, follow these steps:

1. **Clone the repository**:

   ```bash
   git clone https://github.com/audoclyphia-evals/petclinic-customers-service.git
   cd petclinic-customers-service
   ```

2. **Build the project**:

   ```bash
   mvn clean install
   ```

3. **Build Docker image** (optional):

   ```bash
   mvn clean install -P buildDocker
   ```

## Quickstart

To run the Customers Service locally, ensure that the required external services are running first. See the [Requirements](#requirements) section for details.

1. **Ensure required services are running** (Config Server, Eureka Server).

2. **Run the application**:

   ```bash
   mvn spring-boot:run
   ```

   The service will start on port `8081`.

3. **Test the endpoints**:

   ```bash
   # Retrieve all owners
   curl http://localhost:8081/owners

   # Create a new owner
   curl -X POST http://localhost:8081/owners \
     -H "Content-Type: application/json" \
     -d '{
       "firstName": "John",
       "lastName": "Doe",
       "address": "123 Main St",
       "city": "Springfield",
       "telephone": "1234567890"
     }'
   ```

## Usage

The following tables summarize the available REST endpoints. For detailed request/response specifications, including status codes and request body schemas, see the [API Reference](#api-reference) section.

### Owner Endpoints

| Method | URL                 | Description                     |
| ------ | ------------------- | ------------------------------- |
| GET    | `/owners`           | Retrieve all owners             |
| GET    | `/owners/{ownerId}` | Retrieve an owner by ID         |
| POST   | `/owners`           | Create a new owner              |
| PUT    | `/owners/{ownerId}` | Update an existing owner        |

### Pet Endpoints

| Method | URL                              | Description              |
| ------ | -------------------------------- | ------------------------ |
| GET    | `/petTypes`                      | Retrieve all pet types   |
| POST   | `/owners/{ownerId}/pets`         | Create a new pet         |
| PUT    | `/owners/*/pets/{petId}`         | Update an existing pet   |
| GET    | `/owners/*/pets/{petId}`         | Retrieve a pet by ID     |

### Data Models

#### Owner Fields

- `firstName` (String, required)
- `lastName` (String, required)
- `address` (String, required)
- `city` (String, required)
- `telephone` (String, required, max 12 digits)

#### Pet Fields

- `name` (String)
- `birthDate` (Date)
- `type` (PetType reference)
- Owner relationship (many-to-one with `Owner` entity)

## API Reference

The following OpenAPI specification provides detailed endpoint descriptions, request parameters, and response schemas. This spec corresponds to the endpoints listed in the [Usage](#usage) section above.

```yaml
openapi: 3.0.3
info:
  title: petclinic-customers-service API
  description: REST API documentation for the Customers Service
  version: 1.0.0
paths:
  /owners:
    get:
      summary: Retrieve all owners
      description: Returns a list of all owners in the system.
      operationId: getAllOwners
      tags:
      - Owner
      responses:
        '200':
          description: Successfully retrieved the list of owners
        '401':
          description: Unauthorized access
        '500':
          description: Internal server error
    post:
      summary: Create a new owner
      description: Creates a new owner record. The request body must contain valid owner data.
      operationId: createOwner
      tags:
      - Owner
      responses:
        '201':
          description: Owner created successfully
        '400':
          description: Invalid request body
        '500':
          description: Internal server error
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              description: Request body for creating an owner.
  /owners/{ownerId}:
    get:
      summary: Retrieve an owner by ID
      description: Returns the owner with the specified identifier.
      operationId: getOwnerById
      tags:
      - Owner
      responses:
        '200':
          description: Successfully retrieved the owner
        '400':
          description: Bad request, e.g., invalid ownerId
        '404':
          description: Owner not found
        '500':
          description: Internal server error
      parameters:
      - name: ownerId
        in: path
        required: true
        schema:
          type: integer
          minimum: 1
        description: The unique numeric identifier of the owner
    put:
      summary: Update an existing owner by ID
      description: Updates the owner with the specified identifier. Requires owner ID in path and owner data in request body.
      operationId: updateOwner
      tags:
      - Owner
      responses:
        '204':
          description: Owner updated successfully, no content returned
        '400':
          description: Bad request due to invalid input data or validation failure
        '401':
          description: Unauthorized – authentication required
        '404':
          description: Owner not found with given ID
        '500':
          description: Internal server error
      parameters:
      - name: ownerId
        in: path
        required: true
        schema:
          type: integer
          minimum: 1
        description: Unique identifier for the owner to update
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              description: Owner data to update
  /owners/{ownerId}/pets:
    post:
      summary: Create a new pet for an owner
      description: Creates a new pet associated with the specified owner.
      operationId: createPetForOwner
      tags:
      - Pet
      responses:
        '201':
          description: Pet created successfully
        '400':
          description: Bad request
        '404':
          description: Owner not found
        '500':
          description: Internal server error
      parameters:
      - name: ownerId
        in: path
        required: true
        schema:
          type: integer
        description: Unique identifier of the owner
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              description: Pet request data
  /owners/*/pets/{petId}:
    get:
      summary: Retrieve a pet by ID
      description: Retrieves the details of a specific pet by its unique identifier.
      operationId: findPet
      tags:
      - Pet
      responses:
        '200':
          description: Successfully retrieved the pet details
        '400':
          description: Invalid pet ID format
        '401':
          description: Unauthorized access
        '500':
          description: Internal server error
      parameters:
      - name: petId
        in: path
        required: true
        schema:
          type: integer
        description: The unique identifier of the pet to retrieve
  /petTypes:
    get:
      summary: Retrieve all pet types
      description: Retrieves a list of all available pet types from the system.
      operationId: getPetTypes
      tags:
      - Pet
      responses:
        '200':
          description: Successfully retrieved pet types
        '400':
          description: Bad request
        '401':
          description: Unauthorized
        '500':
          description: Internal server error
tags:
- name: Owner
- name: Pet
```

## Additional Documentation

### Project Structure

```
src/main/java/org/springframework/samples/petclinic/customers/
├── CustomersServiceApplication.java           # Application entry point
├── config/
│   └── MetricConfig.java                      # Metrics configuration
├── model/
│   ├── Owner.java                             # Owner JPA entity
│   ├── OwnerRepository.java                   # Owner repository interface
│   ├── Pet.java                               # Pet JPA entity
│   ├── PetRepository.java                     # Pet repository interface
│   └── PetType.java                           # Pet type JPA entity
└── web/
    ├── OwnerResource.java                     # Owner REST controller
    ├── OwnerRequest.java                      # Owner request DTO
    ├── PetResource.java                       # Pet REST controller
    ├── PetRequest.java                        # Pet request DTO
    ├── PetDetails.java                        # Pet response DTO
    ├── ResourceNotFoundException.java         # Custom 404 exception
    └── mapper/
        ├── Mapper.java                        # Generic mapper interface
        └── OwnerEntityMapper.java             # Owner entity mapper implementation
```

### Key Classes

- **`CustomersServiceApplication`**: Spring Boot main class that starts the service with service discovery enabled via `@EnableDiscoveryClient`.
- **`OwnerResource`**: REST controller class exposing endpoints for owner CRUD operations under `/owners`, annotated with `@Timed("petclinic.owner")` for metrics.
- **`PetResource`**: REST controller class for pet-related operations including pet creation, update, retrieval, and pet type listing.
- **`MetricConfig`**: Configuration class for metrics that sets up common tags and a timed aspect.

### Related Services

With the Customers Service providing owner and pet data, the following companion services complete the Spring PetClinic microservices ecosystem:

- `petclinic-api-gateway` — Routes requests to this service
- `petclinic-vets-service` — Manages veterinarians
- `petclinic-visits-service` — Manages pet visits