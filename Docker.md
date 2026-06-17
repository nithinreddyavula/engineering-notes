# Docker — Images, Containers, Compose

## Core Concepts
- **Image**: read-only blueprint (OS, Java runtime, JAR, config) — like a Java class.
- **Container**: running instance of an image, alive and handling requests — like a Java object. One image → many containers.
- **Dockerfile**: instructions to build an image.

## BookMyShow Dockerfile
```dockerfile
FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
EXPOSE 8080
COPY target/bookmyshow-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```
- `FROM`: Alpine-based Java 17 image (~5MB vs Ubuntu's ~200MB) → faster pulls during incidents, smaller attack surface (fewer pre-installed packages = fewer CVEs), lower EC2 storage cost at scale.
- `WORKDIR`: sets working directory inside container.
- `COPY`: copies the Maven-built fat JAR into the image.
- `EXPOSE`: documents the listening port (8080) — does not actually map it.
- `ENTRYPOINT`: command run on container start.

## Layer Caching
Each instruction creates a cached layer, hashed by instruction + inputs. On rebuild, unchanged-hash layers are pulled from cache; a changed layer forces it and every layer below it to rebuild. Stable instructions (FROM, WORKDIR, EXPOSE) should sit above frequently-changing ones (COPY) to maximize cache reuse and minimize rebuild time — important for fast EC2 hotfix deployments.

## Why Dockerfile Alone Isn't Enough (Multi-Container Apps)
Three unsolved problems with separate `docker run` commands per service:
1. **Orchestration** — Spring Boot starting before MySQL is ready throws a connection-refused exception and crashes.
2. **Networking** — isolated containers can't reach each other via `localhost`.
3. **Configuration** — env vars (DB creds, Redis host) scattered across multiple manual commands.

## Docker Compose Fix
Single `docker-compose.yml` defines all services:
```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: bookmyshow
    volumes:
      - mysql-data:/var/lib/mysql
  redis:
    image: redis:7-alpine
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/bookmyshow
      SPRING_REDIS_HOST: redis
    depends_on: [mysql, redis]
volumes:
  mysql-data:
```
- Creates a shared private network — containers reach each other by **service name** (`mysql`, `redis`), not `localhost`, via Docker's internal DNS.
- `depends_on` controls startup order.
- `volumes` persists MySQL data across container restarts.
- `docker-compose up -d` builds the app image, pulls mysql/redis images, creates the network, and starts everything with one command.
