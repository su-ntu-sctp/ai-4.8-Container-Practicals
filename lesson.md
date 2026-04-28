# Lesson 4.8: Containerizing Simple-CRM-Lite (Saturday Coaching)

## Lesson Overview

In this lesson you will apply the Docker and containerization concepts you learned in previous lessons to a Spring Boot application - the **simple-crm-lite** project.

## Why Simple-CRM-Lite?

Today's focus is learning Docker, not debugging complex applications. Your full simple-crm (with Product, Interaction entities, relationships) can introduce issues unrelated to Docker - like JPA lazy loading in containers or relationship serialization errors. Using a simplified baseline (Customer entity only) ensures everyone learns Docker concepts successfully.

---

## Learning Objectives

By the end of this session, you will be able to:

1. **Apply** Docker containerization to a multi-layered Spring Boot REST API application
2. **Create** a Dockerfile for a Spring Boot application with PostgreSQL dependencies
3. **Configure** Docker Compose to orchestrate multiple containers (application + database)
4. **Test** and verify that containerized endpoints work correctly

---

## Prerequisites

Before starting this lesson, ensure you have:

- Completed Lessons 4.4 (Local Containerization) and 4.6 (Docker Compose)
- Docker Desktop installed and running
- Your **simple-crm-lite** Spring Boot project ready and working locally
- PostgreSQL running locally with the `simplecrmlite` database created

---

## Part 1: Verify Your Setup

Before containerizing, ensure your simple-crm-lite application is working correctly locally. Run it with Maven and confirm you can hit the `/customers` endpoint before proceeding.

---

## Part 2: Understanding What We'll Build

### Current Architecture (Local)

```
┌─────────────────────────────────────┐
│   Your Computer                     │
│                                     │
│  ┌──────────────────────────────┐  │
│  │  simple-crm-lite (Port 8080) │  │
│  │  - Running with mvn          │  │
│  │  - Uses Java 21              │  │
│  └──────────────────────────────┘  │
│              ↓                      │
│  ┌──────────────────────────────┐  │
│  │  PostgreSQL (Port 5432)      │  │
│  │  - Running on your machine   │  │
│  │  - Database: simplecrmlite   │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
```

### Target Architecture (Containerized)

```
┌─────────────────────────────────────────────────┐
│   Your Computer                                 │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │  Docker Network: simple-crm-lite_default │  │
│  │                                          │  │
│  │  ┌──────────────────────────────────┐   │  │
│  │  │  simple-crm-lite-app Container   │   │  │
│  │  │  - Spring Boot                   │   │  │
│  │  │  - Java 21                       │   │  │
│  │  │  - Port 8080                     │   │  │
│  │  └──────────────────────────────────┘   │  │
│  │              ↓                           │  │
│  │  ┌──────────────────────────────────┐   │  │
│  │  │  simple-crm-lite-db Container    │   │  │
│  │  │  - PostgreSQL 16                 │   │  │
│  │  │  - Port 5432                     │   │  │
│  │  │  - Data in volume                │   │  │
│  │  └──────────────────────────────────┘   │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### Key Differences

| Aspect | Local | Containerized |
|--------|-------|---------------|
| **Application** | Runs with `mvn spring-boot:run` | Runs in Docker container |
| **Database** | Your local PostgreSQL | Separate PostgreSQL container |
| **Isolation** | Shares your computer's environment | Each container is isolated |
| **Portability** | Only works on your machine | Works on any machine with Docker |
| **Cleanup** | Must uninstall manually | Delete containers = clean slate |

### Why Containerize?

**Benefits you'll experience today:**
1. **Consistency** - Works the same on any machine
2. **Isolation** - App dependencies don't affect your computer
3. **Easy cleanup** - Remove containers, everything's gone
4. **Professional practice** - This is how production apps run

---

## Part 3: Create the Dockerfile

A Dockerfile is a recipe that tells Docker how to build an image of your application.

### Step 1: Understand the Dockerfile Structure

We'll create a **multi-stage Dockerfile** (from Lesson 4.4):

```
Stage 1: BUILD
- Use Maven image
- Copy source code
- Build JAR file

Stage 2: RUN
- Use Java 21 runtime image
- Copy JAR from Stage 1
- Run the application
```

**Why multi-stage?**
- Smaller final image (no Maven, no source code)
- More secure (only runtime dependencies)
- Faster to deploy

### Step 2: Create the Dockerfile

In your `simple-crm-lite` project root directory, create a file named `Dockerfile` (no extension):

```dockerfile
# ============================================
# STAGE 1: BUILD
# ============================================
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /app

# Copy pom.xml first (for better caching)
COPY pom.xml .

# Download dependencies (cached if pom.xml doesn't change)
RUN mvn dependency:go-offline -B

# Copy source code
COPY src ./src

# Build the application
RUN mvn clean package -DskipTests

# ============================================
# STAGE 2: RUN
# ============================================
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# Copy JAR from build stage
COPY --from=build /app/target/simple-crm-lite-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Step 3: Understanding Each Line

#### Stage 1 - Build Stage

**`FROM maven:3.9-eclipse-temurin-21 AS build`**
- Uses official Maven image with Java 21
- Names this stage "build" (referenced in Stage 2)

**`WORKDIR /app`**
- Sets working directory inside container

**`COPY pom.xml .`**
- Copied first for Docker layer caching
- If pom.xml doesn't change, dependency layer is reused

**`RUN mvn dependency:go-offline -B`**
- Downloads all Maven dependencies
- Cached if pom.xml hasn't changed

**`COPY src ./src`**
- Copies source code after dependencies for better caching

**`RUN mvn clean package -DskipTests`**
- Builds the JAR file: `simple-crm-lite-0.0.1-SNAPSHOT.jar`

#### Stage 2 - Runtime Stage

**`FROM eclipse-temurin:21-jre-alpine`**
- Lightweight Java 21 JRE image (no JDK, no Maven)
- Alpine Linux = very small size

**`COPY --from=build /app/target/simple-crm-lite-0.0.1-SNAPSHOT.jar app.jar`**
- Copies JAR from build stage only
- Build stage is discarded after this

**`ENTRYPOINT ["java", "-jar", "app.jar"]`**
- Command to run when container starts

### Step 4: Build Your Docker Image

```bash
cd simple-crm-lite

docker build -t simple-crm-lite:latest .
```

**Expected output:**
```
[+] Building 45.2s (15/15) FINISHED
 => [build 1/6] FROM maven:3.9-eclipse-temurin-21
 => [build 2/6] WORKDIR /app
 => [build 3/6] COPY pom.xml .
 => [build 4/6] RUN mvn dependency:go-offline -B
 => [build 5/6] COPY src ./src
 => [build 6/6] RUN mvn clean package -DskipTests
 => [stage-1 1/3] FROM eclipse-temurin:21-jre-alpine
 => [stage-1 2/3] WORKDIR /app
 => [stage-1 3/3] COPY --from=build ...app.jar
 => exporting to image
Successfully built and tagged simple-crm-lite:latest
```

**First build takes time** (2-5 minutes) — downloads base images and dependencies.
**Second build is faster** (~30 seconds) — layers are cached.

### Step 5: Verify Image Was Created

```bash
docker images | grep simple-crm-lite
```

**Expected output:**
```
simple-crm-lite   latest   abc123def456   2 minutes ago   350MB
```

**✅ Success!** Docker image created.

---

## Part 4: Run Your Containerized Application

### Step 1: Understanding the Challenge

**Problem:** Your containerized app needs to connect to PostgreSQL, but:
- Container is isolated
- Cannot access `localhost` PostgreSQL (that's your computer, not inside the container)
- Need to tell the container where the database is

**Solution for now:** Connect to host's PostgreSQL from container using `host.docker.internal`

### Step 2: Update application.properties

Create a new file: `src/main/resources/application-docker.properties`

```properties
# Database Configuration for Docker
spring.datasource.url=jdbc:postgresql://host.docker.internal:5432/simplecrmlite
spring.datasource.username=postgres
spring.datasource.password=password

# JPA Configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# Server Configuration
server.port=8080
```

**Key difference:** `localhost` is replaced with `host.docker.internal` — Docker's way to reach the host machine from inside a container.

### Step 3: Rebuild Image

```bash
docker build -t simple-crm-lite:latest .
```

### Step 4: Run the Container

```bash
docker run -d \
  --name simple-crm-lite-app \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=docker \
  simple-crm-lite:latest
```

**Flags explained:**
- `-d` — detached mode (runs in background)
- `--name simple-crm-lite-app` — names the container
- `-p 8080:8080` — maps host port 8080 to container port 8080
- `-e SPRING_PROFILES_ACTIVE=docker` — tells Spring Boot to use `application-docker.properties`

### Step 5: Check Container is Running

```bash
docker ps
```

**Expected output:**
```
CONTAINER ID   IMAGE                    COMMAND              STATUS         PORTS
a1b2c3d4e5f6   simple-crm-lite:latest   "java -jar app.jar"  Up 9 seconds   0.0.0.0:8080->8080/tcp
```

### Step 6: Check Application Logs

```bash
docker logs simple-crm-lite-app
```

**Look for:**
```
Started SimpleCrmLiteApplication in X.XXX seconds
✅ Sample data loaded: 3 customers added
```

### Step 7: Test the Containerized Application

```bash
curl http://localhost:8080/customers
```

**Expected:** JSON array with 3 customers

### Step 8: Test All CRUD Operations

#### Create (POST)
```bash
curl -X POST http://localhost:8080/customers \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Tony",
    "lastName": "Stark",
    "email": "tony@starkindustries.com",
    "contactNo": "55566677",
    "jobTitle": "CEO",
    "yearOfBirth": 1980
  }'
```

**Expected:** 201 CREATED with new customer (id: 4)

#### Read (GET)
```bash
curl http://localhost:8080/customers/4
```

**Expected:** 200 OK with Tony Stark's data

#### Update (PUT)
```bash
curl -X PUT http://localhost:8080/customers/4 \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Tony",
    "lastName": "Stark",
    "email": "ironman@starkindustries.com",
    "contactNo": "55566677",
    "jobTitle": "Superhero CEO",
    "yearOfBirth": 1980
  }'
```

**Expected:** 200 OK with updated data

#### Delete (DELETE)
```bash
curl -X DELETE http://localhost:8080/customers/4
```

**Expected:** 204 NO CONTENT

Verify deletion:
```bash
curl http://localhost:8080/customers/4
```

**Expected:** 404 NOT FOUND

**✅ Success!** Your containerized application works perfectly!

### Step 9: Stop and Remove Container

```bash
docker stop simple-crm-lite-app
docker rm simple-crm-lite-app
```

We'll use Docker Compose in the next part instead.

---

## Part 5: Add PostgreSQL Container with Docker Compose

### Step 1: Why Docker Compose?

**Problem with current setup:**
- Application in container ✅
- Database on your computer ❌
- Not fully portable

**Solution: Docker Compose**
- Application container + Database container
- Both start with one command
- Fully portable — only needs Docker

### Step 2: Create docker-compose.yml

In your `simple-crm-lite` project root, create `docker-compose.yml`:

```yaml
services:
  # ========================================
  # PostgreSQL Database Container
  # ========================================
  db:
    image: postgres:16-alpine
    container_name: simple-crm-lite-db
    environment:
      POSTGRES_DB: simplecrmlite
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5433:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ========================================
  # Spring Boot Application Container
  # ========================================
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: simple-crm-lite-app
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/simplecrmlite
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: password
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

# ========================================
# Volumes (Persistent Storage)
# ========================================
volumes:
  postgres-data:
```

### Step 3: Understanding docker-compose.yml

#### Database Service (db)

**`image: postgres:16-alpine`** — official lightweight PostgreSQL 16 image

**`ports: "5433:5432"`** — maps host port 5433 to container port 5432, avoiding conflict with your local PostgreSQL on 5432

**`volumes: postgres-data:/var/lib/postgresql/data`** — persists database data; survives container removal

**`healthcheck`** — checks if database is ready before app starts

#### Application Service (app)

**`build:`** — builds image from Dockerfile in current directory

**`SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/simplecrmlite`** — uses `db` as hostname (Docker Compose service name), not `localhost`

**`depends_on: condition: service_healthy`** — app waits for database to pass health check before starting

**`restart: unless-stopped`** — automatically restarts if the container crashes

### Step 4: Remove application-docker.properties

Docker Compose uses environment variables directly, so this file is no longer needed:

```bash
rm src/main/resources/application-docker.properties
```

### Step 5: Start Everything with Docker Compose

```bash
cd simple-crm-lite

docker compose up -d
```

**Expected output:**
```
[+] Running 3/3
 ✔ Network simple-crm-lite_default       Created
 ✔ Container simple-crm-lite-db          Started
 ✔ Container simple-crm-lite-app         Started
```

### Step 6: Watch the Logs

```bash
docker compose logs -f
```

**Look for:**
```
simple-crm-lite-db   | database system is ready to accept connections
simple-crm-lite-app  | Started SimpleCrmLiteApplication in X.XXX seconds
simple-crm-lite-app  | ✅ Sample data loaded: 3 customers added
```

Press `Ctrl+C` to exit log viewing — containers keep running.

### Step 7: Verify Both Containers Running

```bash
docker compose ps
```

**Expected output:**
```
NAME                  IMAGE                  STATUS                   PORTS
simple-crm-lite-app   simple-crm-lite        Up 30 seconds (healthy)  0.0.0.0:8080->8080/tcp
simple-crm-lite-db    postgres:16-alpine     Up 45 seconds (healthy)  0.0.0.0:5433->5432/tcp
```

Both should show `Up` and `(healthy)`.

---

## Part 6: Test the Complete Containerized Stack

### Step 1: Basic Endpoint Test

```bash
curl http://localhost:8080/customers
```

**Expected:** JSON array with 3 sample customers

### Step 2: Test Full CRUD Cycle

**Create:**
```bash
curl -X POST http://localhost:8080/customers \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Natasha",
    "lastName": "Romanoff",
    "email": "natasha@shield.gov",
    "contactNo": "44455566",
    "jobTitle": "Agent",
    "yearOfBirth": 1985
  }'
```

**Read:**
```bash
curl http://localhost:8080/customers
```

Should now show 4 customers (3 sample + Natasha)

**Update:**
```bash
curl -X PUT http://localhost:8080/customers/4 \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Natasha",
    "lastName": "Romanoff",
    "email": "blackwidow@avengers.com",
    "contactNo": "44455566",
    "jobTitle": "Avenger",
    "yearOfBirth": 1985
  }'
```

**Delete:**
```bash
curl -X DELETE http://localhost:8080/customers/4
```

**Verify deletion:**
```bash
curl http://localhost:8080/customers
```

Should be back to 3 customers.

### Step 3: Test Data Persistence

Data should survive container restart:

```bash
# Stop all containers
docker compose down

# Start again
docker compose up -d

# Wait 10 seconds for startup
sleep 10

# Check data
curl http://localhost:8080/customers
```

**Expected:** Still see 3 sample customers — volume persists data even when containers are removed.

### Step 4: Access Database Directly (Optional)

```bash
docker exec -it simple-crm-lite-db psql -U postgres -d simplecrmlite
```

**Inside PostgreSQL:**
```sql
SELECT * FROM customer;
\q
```

**✅ Success!** Complete containerized stack working perfectly!

---

## Useful Docker Compose Commands

```bash
# Start all services in background
docker compose up -d

# Start and rebuild images
docker compose up -d --build

# Stop all services (containers remain)
docker compose stop

# Start stopped services
docker compose start

# Stop and remove all containers and networks
docker compose down

# Stop and remove everything including volumes (data lost!)
docker compose down -v

# View logs from all services
docker compose logs

# View logs from specific service
docker compose logs app

# Follow logs in real-time
docker compose logs -f

# View running services
docker compose ps

# Restart specific service
docker compose restart app

# Execute command in service container
docker compose exec app /bin/sh

# View resource usage
docker stats
```

---

## Troubleshooting Common Issues

### Issue 1: Port Already in Use

**Error:** `Bind for 0.0.0.0:8080 failed: port is already allocated`

**Solution:**
```bash
# Find what's using port 8080
lsof -i :8080

# Kill the process if needed
kill -9 <PID>

# Or change port in docker-compose.yml
ports:
  - "8081:8080"
```

---

### Issue 2: Database Connection Failed

**Error in logs:** `Connection refused` or `Unknown host`

**Solution 1 — Wait longer:**
```bash
docker compose down
docker compose up -d
sleep 20
docker compose logs app
```

**Solution 2 — Check health:**
```bash
docker compose ps
# DB should show "(healthy)"
```

**Solution 3 — Verify network:**
```bash
docker network ls
docker network inspect simple-crm-lite_default
```

---

### Issue 3: Changes Not Reflected

**Solution:** Rebuild the image
```bash
docker compose down
docker compose up -d --build
```

---

### Issue 4: "No space left on device"

```bash
docker container prune -f
docker image prune -a -f
docker volume prune -f
```

---

### Issue 5: Sample Data Not Loading

```bash
docker exec -it simple-crm-lite-db psql -U postgres -d simplecrmlite

# Check if customers exist
SELECT * FROM customer;
\q

# If empty, restart app
docker compose restart app
docker compose logs app
```