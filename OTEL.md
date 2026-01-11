# Running Spring Pet Clinic with OpenTelemetry

This guide explains how to run the Spring Pet Clinic application with the OpenTelemetry Java agent for automatic instrumentation.

## Prerequisites

- Java 25+
- Docker

## Start the Observability Backend

Start LGTM (Grafana, Loki, Tempo, Prometheus) with Docker:

```bash
docker run -d --name lgtm -p 3000:3000 -p 4317:4317 -p 4318:4318 grafana/otel-lgtm:latest
```

This provides:
- **Grafana** at http://localhost:3000 (admin/admin)
- **OTLP endpoint** at `localhost:4317` (gRPC) and `localhost:4318` (HTTP)

## Download the OpenTelemetry Java Agent

```bash
mkdir -p local
curl -L -o local/opentelemetry-javaagent.jar \
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
```

## Build the Application

```bash
./gradlew bootJar
```

## Run the Application

```bash
java -javaagent:local/opentelemetry-javaagent.jar \
     -Dotel.javaagent.configuration-file=otel.properties \
     -jar build/libs/spring-petclinic-4.0.0-SNAPSHOT.jar
```

## Configuration

The `otel.properties` file contains the agent configuration:

```properties
otel.service.name=spring-petclinic
otel.exporter.otlp.endpoint=http://localhost:4317
otel.exporter.otlp.protocol=grpc
otel.resource.attributes=deployment.environment.name=development
otel.resource.disabled.keys=process.command_args,process.runtime.description,process.executable.path
```

## Manual Instrumentation

This project includes manual instrumentation to add business context to traces.

### Dependencies

The OpenTelemetry API is added to `build.gradle`:

```groovy
implementation platform("io.opentelemetry:opentelemetry-bom:${otelVersion}")
implementation 'io.opentelemetry:opentelemetry-api'
```

### Example: Adding Owner Name to Spans

In `OwnerController.showOwner()`, the owner's name is added to the current span:

```java
Span.current().setAttribute("owner.name", owner.getFirstName() + " " + owner.getLastName());
```

When the Java agent is attached, `Span.current()` returns the active HTTP server span.
The `owner.name` attribute will appear on traces for `/owners/{ownerId}` requests.

## What Gets Instrumented

The Java agent automatically instruments:

- HTTP server requests (Spring MVC)
- Database operations (JDBC/JPA)
- HTTP client calls
- Cache operations
- Spring scheduling

## Verify Telemetry

1. Access the application at http://localhost:8080
2. Navigate through some pages (e.g., find owners, view owner details)
3. Open Grafana at http://localhost:3000
4. Go to **Explore** → Select **Tempo** → Search for traces

## Cleanup

```bash
docker stop lgtm && docker rm lgtm
```
