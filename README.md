# Patient Management System

A microservices-based patient management system built with Java Spring Boot, gRPC, and Docker.

## Architecture

- **Patient Service**: REST API for patient CRUD operations, communicates with Billing Service via gRPC.
- **Billing Service**: gRPC service for billing operations.
- **Database**: PostgreSQL for patient data.

## Technologies

- Java 21
- Spring Boot
- gRPC
- PostgreSQL
- Docker & Docker Compose
- Maven

## Getting Started

1. Clone the repository.
2. Copy `.env` and update variables if needed.
3. Run `docker-compose up --build` to start the services.

## API Endpoints

- Patient Service: http://localhost:4000
- Billing Service: gRPC on port 9001

## Development

- Build services: `mvn clean package` in each service directory.
- Run locally: Update application.properties for local DB.

## Contributing

Please read the [Backend Architecture Guide](BACKEND_ARCHITECTURE_GUIDE.md) for detailed information.