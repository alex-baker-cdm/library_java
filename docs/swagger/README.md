# Library Management System API Documentation

This directory contains the OpenAPI/Swagger documentation for the Library Management System API.

## Files

- `openapi.yaml` - OpenAPI 3.0.3 specification describing all available API endpoints

## API Overview

The Library Management System provides a REST API for managing patron profiles, book holds, and checkouts. The API follows HATEOAS principles and returns hypermedia-enriched responses.

### Patron Profile Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/profiles/{patronId}` | Get patron profile with links to holds and checkouts |
| GET | `/profiles/{patronId}/holds/` | Get all current holds for a patron |
| POST | `/profiles/{patronId}/holds` | Place a hold on a book |
| GET | `/profiles/{patronId}/holds/{bookId}` | Get details of a specific hold |
| DELETE | `/profiles/{patronId}/holds/{bookId}` | Cancel a hold |
| GET | `/profiles/{patronId}/checkouts/` | Get all current checkouts for a patron |
| GET | `/profiles/{patronId}/checkouts/{bookId}` | Get details of a specific checkout |

### Actuator Endpoints

The application exposes Spring Boot Actuator endpoints for monitoring:

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/actuator/health` | Health check endpoint |
| GET | `/actuator/info` | Application information |
| GET | `/actuator/metrics` | Available metrics |
| GET | `/actuator/prometheus` | Prometheus-format metrics |

## Viewing the Documentation

### Using Swagger UI

You can view this documentation interactively using Swagger UI:

1. Visit [Swagger Editor](https://editor.swagger.io/)
2. Import the `openapi.yaml` file
3. Explore and test the API endpoints

### Using Redoc

Alternatively, you can use Redoc for a clean documentation view:

1. Visit [Redoc Demo](https://redocly.github.io/redoc/)
2. Enter the URL to the raw `openapi.yaml` file

### Local Development

To integrate Swagger UI into the application, you can add the springdoc-openapi dependency:

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
    <version>1.6.14</version>
</dependency>
```

Then access Swagger UI at `http://localhost:8080/swagger-ui.html`

## Data Models

### Key Entities

- **PatronId**: UUID identifying a library patron
- **BookId**: UUID identifying a book in the catalog
- **LibraryBranchId**: UUID identifying a library branch
- **Hold**: A reservation placed on a book by a patron
- **Checkout**: A book currently borrowed by a patron

### HATEOAS Links

All responses include `_links` objects following the HAL specification, providing navigation to related resources and available actions.
