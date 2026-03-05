# ${{ values.name }}

${{ values.description }}

## Prerequisites

- Java 21
- Gradle 8.13+ (or use the Gradle wrapper)

## Getting Started

Generate the Gradle wrapper (first time only):

```bash
gradle wrapper
```

Build the project:

```bash
./gradlew build
```

Run the application:

```bash
./gradlew bootRun
```

The service starts on port 8080.

## Endpoints

- Health check: `GET /health`
- Actuator health: `GET /actuator/health`

## Docker

Build the image:

```bash
docker build -t ${{ values.name }} .
```

Run the container:

```bash
docker run -p 8080:8080 ${{ values.name }}
```

## CI/CD

This project includes a GitHub Actions workflow that runs on every push:

- **Build** - Compiles the project with Gradle
- **Test** - Runs JUnit tests
- **Deploy** - Builds and pushes a Docker image to GHCR (main branch only)
