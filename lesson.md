# Lesson 4.8: Containerizing Simple-CRM-Lite (Saturday Coaching)

## Lesson Overview

 In this lesson you will apply the Docker and containerization concepts you learned in previous lessons to a Spring Boot application - the **simple-crm-lite** project. 

## Why Simple-CRM-Lite?

Today's focus is learning Docker, not debugging complex applications. Your full simple-crm (with Product, Interaction entities, relationships) can introduce issues unrelated to Docker - like JPA lazy loading in containers or relationship serialization errors. Using a simplified baseline (Customer entity only) ensures everyone learns Docker concepts successfully in our 3-hour session.



## Learning Objectives

By the end of this session, you will be able to:

1. **Apply** Docker containerization to a multi-layered Spring Boot REST API application
2. **Create** a Dockerfile for a Spring Boot application with PostgreSQL dependencies
3. **Configure** Docker Compose to orchestrate multiple containers (application + database)
4. **Test** and verify that containerized endpoints work correctly
5. **Troubleshoot** common containerization issues with instructor guidance


---

## Prerequisites


---

## Part 1: Verify Your Setup (15 minutes)

Before containerizing, let's ensure your simple-crm-lite application is working perfectly.

## Part 2: Understanding What We'll Build (10 minutes)

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

We'll create a **multi-stage Dockerfile** (remember from Lesson 4.5?):

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
# Use Maven image with Java 21 to build the application
FROM maven:3.9-eclipse-temurin-21 AS build

# Set working directory inside container
WORKDIR /app

# Copy pom.xml first (for better caching)
# Docker caches layers - if pom.xml doesn't change, dependencies are cached
COPY pom.xml .

# Download dependencies (this layer is cached if pom.xml doesn't change)
RUN mvn dependency:go-offline -B

# Copy the entire source code
COPY src ./src

# Build the application (skip tests for faster build)
# Creates JAR file in target/ directory
RUN mvn clean package -DskipTests

# ============================================
# STAGE 2: RUN
# ============================================
# Use lightweight Java 21 runtime image (no Maven needed)
FROM eclipse-temurin:21-jre-alpine

# Set working directory
WORKDIR /app

# Copy the JAR file from build stage
# Build stage created: target/simple-crm-lite-0.0.1-SNAPSHOT.jar
COPY --from=build /app/target/simple-crm-lite-0.0.1-SNAPSHOT.jar app.jar

# Expose port 8080 (informational - tells users which port to use)
EXPOSE 8080

# Run the application
# java -jar runs the Spring Boot JAR file
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Step 3: Understanding Each Line

Let's break down the key parts:

#### Stage 1 - Build Stage

**`FROM maven:3.9-eclipse-temurin-21 AS build`**
- Uses official Maven image with Java 21
- Names this stage "build" (we reference it later)
- This image has Maven and Java 21 installed

**`WORKDIR /app`**
- Sets working directory to /app inside container
- All subsequent commands run from this directory

**`COPY pom.xml .`**
- Copies pom.xml to container
- Done first for Docker layer caching
- If pom.xml doesn't change, dependencies layer is reused

**`RUN mvn dependency:go-offline -B`**
- Downloads all Maven dependencies
- Cached if pom.xml hasn't changed
- Speeds up subsequent builds

**`COPY src ./src`**
- Copies source code to container
- Done after dependencies for better caching

**`RUN mvn clean package -DskipTests`**
- Builds the application
- Creates JAR file: `simple-crm-lite-0.0.1-SNAPSHOT.jar`
- Skips tests for faster build (tests already passed locally)

#### Stage 2 - Runtime Stage

**`FROM eclipse-temurin:21-jre-alpine`**
- Uses lightweight Java 21 JRE image (no JDK, no Maven)
- Alpine Linux = very small size
- Only has what's needed to RUN Java applications

**`COPY --from=build /app/target/simple-crm-lite-0.0.1-SNAPSHOT.jar app.jar`**
- Copies JAR file from build stage
- Renames it to app.jar
- Build stage is discarded after this

**`EXPOSE 8080`**
- Documents that application uses port 8080
- Doesn't actually open the port (done with docker run -p)

**`ENTRYPOINT ["java", "-jar", "app.jar"]`**
- Command to run when container starts
- Starts the Spring Boot application

### Step 4: Build Your Docker Image

Now let's build the image from your Dockerfile:

```bash
# Make sure you're in simple-crm-lite directory
cd simple-crm-lite

# Build the image
docker build -t simple-crm-lite:latest .
```

**Understanding the command:**
- `docker build` - Docker build command
- `-t simple-crm-lite:latest` - Name and tag the image
  - `simple-crm-lite` = image name
  - `latest` = version tag
- `.` - Build context (current directory)

**Expected output:**
```
[+] Building 45.2s (15/15) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load .dockerignore
 => [build 1/6] FROM maven:3.9-eclipse-temurin-21
 => [build 2/6] WORKDIR /app
 => [build 3/6] COPY pom.xml .
 => [build 4/6] RUN mvn dependency:go-offline -B
 => [build 5/6] COPY src ./src
 => [build 6/6] RUN mvn clean package -DskipTests
 => [stage-1 1/3] FROM eclipse-temurin:21-jre-alpine
 => [stage-1 2/3] WORKDIR /app
 => [stage-1 3/3] COPY --from=build /app/target/simple-crm-lite-0.0.1-SNAPSHOT.jar app.jar
 => exporting to image
Successfully built and tagged simple-crm-lite:latest
```

**First build takes time** (2-5 minutes):
- Downloads base images
- Downloads Maven dependencies
- Compiles code

**Second build is faster** (30 seconds):
- Layers are cached
- Only rebuilds what changed

### Step 5: Verify Image Was Created

```bash
docker images | grep simple-crm-lite
```

**Expected output:**
```
simple-crm-lite   latest   abc123def456   2 minutes ago   350MB
```

**✅ Success!** You've created a Docker image of your simple-crm-lite application!

**Instructor Checkpoint:** Everyone should have an image created. Raise hand if you see errors.

---

## Part 4: Run Your Containerized Application 

Now let's run your application in a container!

### Step 1: Understanding the Challenge

**Problem:** Your containerized app needs to connect to PostgreSQL, but:
- Container is isolated
- Can't access `localhost` PostgreSQL (that's YOUR computer, not inside container)
- Need to tell container where database is

**Solution for now:** Connect to host's PostgreSQL from container

### Step 2: Update application.properties

We need to change database connection for containerized environment.

**Create a new file:** `src/main/resources/application-docker.properties`

```properties
# Database Configuration for Docker
# Use host.docker.internal to connect to host machine's PostgreSQL
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

**What's different?**
- `localhost` → `host.docker.internal`
- `host.docker.internal` is Docker's way to access host machine from container
- Database name is `simplecrmlite` (same as we created earlier)
- Everything else stays the same

### Step 3: Rebuild Image with New Configuration

Since we added application-docker.properties, rebuild:

```bash
docker build -t simple-crm-lite:latest .
```

This should be faster (cached layers).

### Step 4: Run the Container

```bash
docker run -d \
  --name simple-crm-lite-app \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=docker \
  simple-crm-lite:latest
```

**Understanding the command:**
- `docker run` - Run a container
- `-d` - Detached mode (runs in background)
- `--name simple-crm-lite-app` - Name the container
- `-p 8080:8080` - Port mapping (host:container)
  - First 8080 = your computer's port
  - Second 8080 = container's port
- `-e SPRING_PROFILES_ACTIVE=docker` - Environment variable
  - Tells Spring Boot to use application-docker.properties
- `simple-crm-lite:latest` - Image to run

**Expected output:**
```
a1b2c3d4e5f6... (container ID)
```

### Step 5: Check Container is Running

```bash
docker ps
```

**Expected output:**
```
CONTAINER ID   IMAGE                    COMMAND              CREATED         STATUS         PORTS                    NAMES
a1b2c3d4e5f6   simple-crm-lite:latest   "java -jar app.jar"  10 seconds ago  Up 9 seconds   0.0.0.0:8080->8080/tcp   simple-crm-lite-app
```

**Key information:**
- CONTAINER ID: Unique identifier
- IMAGE: simple-crm-lite:latest
- STATUS: Up X seconds (should be running)
- PORTS: 0.0.0.0:8080->8080/tcp (accessible on port 8080)
- NAMES: simple-crm-lite-app

### Step 6: Check Application Logs

```bash
docker logs simple-crm-lite-app
```

**Look for:**
```
Started SimpleCrmLiteApplication in X.XXX seconds
✅ Sample data loaded: 3 customers added
```

**If you see errors instead:**
- Raise hand for instructor help
- Common issue: Database connection (we'll troubleshoot together)

### Step 7: Test the Containerized Application

**Test GET all customers:**
```bash
curl http://localhost:8080/customers
```

**Expected:** JSON array with 3 customers

**Or use Postman:**
- GET `http://localhost:8080/customers`
- Should return 200 OK with customer data

### Step 8: Test All CRUD Operations

Let's verify all endpoints work:

#### Create a new customer (POST)
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

**Expected:** 201 CREATED with new customer (should have id: 4)

#### Get one customer (GET)
```bash
curl http://localhost:8080/customers/4
```

**Expected:** 200 OK with Tony Stark's data

#### Update customer (PUT)
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

**Expected:** 200 OK with updated data (email and jobTitle changed)

#### Delete customer (DELETE)
```bash
curl -X DELETE http://localhost:8080/customers/4
```

**Expected:** 204 NO CONTENT (empty response)

Verify deletion:
```bash
curl http://localhost:8080/customers/4
```

**Expected:** 404 NOT FOUND with error message

**✅ Success!** Your containerized application works perfectly!

### Step 9: Stop and Remove Container

When you're done testing:

```bash
# Stop the container
docker stop simple-crm-lite-app

# Remove the container
docker rm simple-crm-lite-app
```

**Why remove?**
- We're about to use Docker Compose instead
- Compose will create new containers for us

---

## Part 5: Add PostgreSQL Container with Docker Compose (30 minutes)

Currently, we're still using your local PostgreSQL. Let's containerize that too!

### Step 1: Why Docker Compose?

**Problem with current setup:**
- Application in container ✅
- Database on your computer ❌
- Not fully portable (needs PostgreSQL installed)
- Complex to share with team

**Solution: Docker Compose**
- Define multi-container applications
- Application container + Database container
- Both start with one command
- Fully portable (only needs Docker)

### Step 2: Create docker-compose.yml

In your `simple-crm-lite` project root, create `docker-compose.yml`:

```yaml
# Docker Compose file for Simple-CRM-Lite
# Defines application and database containers

version: '3.8'  # Docker Compose file format version

services:
  # ========================================
  # PostgreSQL Database Container
  # ========================================
  db:
    # Use official PostgreSQL 16 image
    image: postgres:16-alpine
    
    # Container name
    container_name: simple-crm-lite-db
    
    # Environment variables for PostgreSQL
    environment:
      # Database name to create
      POSTGRES_DB: simplecrmlite
      # Superuser username
      POSTGRES_USER: postgres
      # Superuser password
      POSTGRES_PASSWORD: password
    
    # Port mapping (host:container)
    # Access database on localhost:5433 (avoiding conflict with local PostgreSQL on 5432)
    ports:
      - "5433:5432"
    
    # Volume for persistent data
    # Data survives even if container is removed
    volumes:
      - postgres-data:/var/lib/postgresql/data
    
    # Health check - ensures database is ready before app starts
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ========================================
  # Spring Boot Application Container
  # ========================================
  app:
    # Build image from Dockerfile in current directory
    build:
      context: .
      dockerfile: Dockerfile
    
    # Container name
    container_name: simple-crm-lite-app
    
    # Port mapping (host:container)
    ports:
      - "8080:8080"
    
    # Environment variables for Spring Boot
    environment:
      # Database connection string
      # Uses service name 'db' as hostname (Docker networking)
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/simplecrmlite
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: password
    
    # Dependencies - app starts AFTER database is healthy
    depends_on:
      db:
        condition: service_healthy
    
    # Restart policy - automatically restart if crashed
    restart: unless-stopped

# ========================================
# Volumes (Persistent Storage)
# ========================================
volumes:
  # PostgreSQL data volume
  # Data persists even when containers are removed
  postgres-data:
```

### Step 3: Understanding docker-compose.yml

Let's break down the key sections:

#### Services Section

**Two services defined:**
1. `db` - PostgreSQL database
2. `app` - Spring Boot application (simple-crm-lite)

#### Database Service (db)

**`image: postgres:16-alpine`**
- Uses official PostgreSQL 16 image
- Alpine = lightweight version

**`container_name: simple-crm-lite-db`**
- Names container to match our project
- No conflicts with any existing containers

**`environment:`**
- Sets PostgreSQL configuration
- Creates database, user, password on startup
- Database name: `simplecrmlite` (separate from your Module 3 database)

**`ports: "5433:5432"`**
- Maps host port 5433 to container port 5432
- Using 5433 to avoid conflict with local PostgreSQL (5432)
- You can access DB at `localhost:5433`

**`volumes: postgres-data:/var/lib/postgresql/data`**
- Persists database data
- Data survives container removal
- Without this, data is lost when container stops

**`healthcheck:`**
- Checks if database is ready
- App waits for healthy database

#### Application Service (app)

**`build:`**
- Builds image from Dockerfile
- Context = current directory
- Uses the Dockerfile we created earlier

**`container_name: simple-crm-lite-app`**
- Names container to match our project
- Unique name, no conflicts

**`environment:`**
- **Key change:** `SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/simplecrmlite`
- Uses `db` as hostname (not `localhost` or `host.docker.internal`)
- Docker Compose creates network where services find each other by name
- Database name: `simplecrmlite` matches what we created

**`depends_on:`**
- Ensures database starts and is healthy before app
- App won't start until database passes health check

**`restart: unless-stopped`**
- Automatically restarts if crashes
- Production-ready configuration

#### Volumes Section

**`postgres-data:`**
- Named volume for database storage
- Managed by Docker
- Can be backed up, restored

### Step 4: Remove application-docker.properties

Since Docker Compose uses environment variables, we don't need this file:

```bash
rm src/main/resources/application-docker.properties
```

**Why remove it?**
- docker-compose.yml sets environment variables
- Cleaner approach
- One place to configure everything

### Step 5: Start Everything with Docker Compose

**One command starts both containers:**

```bash
# Make sure you're in simple-crm-lite directory
cd simple-crm-lite

# Start all services
docker-compose up -d
```

**Understanding the command:**
- `docker-compose up` - Start all services
- `-d` - Detached mode (background)

**Expected output:**
```
[+] Running 3/3
 ✔ Network simple-crm-lite_default       Created
 ✔ Container simple-crm-lite-db          Started
 ✔ Container simple-crm-lite-app         Started
```

**What just happened?**
1. Docker created a network: `simple-crm-lite_default`
2. Started PostgreSQL container: `simple-crm-lite-db`
3. Waited for database to be healthy
4. Started application container: `simple-crm-lite-app`
5. Application connected to database

### Step 6: Watch the Logs

```bash
# Watch logs from both containers
docker-compose logs -f
```

**Look for:**
```
simple-crm-lite-db   | database system is ready to accept connections
simple-crm-lite-app  | Started SimpleCrmLiteApplication in X.XXX seconds
simple-crm-lite-app  | ✅ Sample data loaded: 3 customers added
```

**Press Ctrl+C to exit log viewing** (containers keep running)

### Step 7: Verify Both Containers Running

```bash
docker-compose ps
```

**Expected output:**
```
NAME                  IMAGE                    STATUS                   PORTS
simple-crm-lite-app   simple-crm-lite          Up 30 seconds (healthy)  0.0.0.0:8080->8080/tcp
simple-crm-lite-db    postgres:16-alpine       Up 45 seconds (healthy)  0.0.0.0:5433->5432/tcp
```

Both should show "Up" and "(healthy)" status.

---

## Part 6: Test the Complete Containerized Stack 

Now everything is containerized! Let's test it thoroughly.

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

**Important test:** Data should survive container restart.

```bash
# Stop all containers
docker-compose down

# Start again
docker-compose up -d

# Wait 10 seconds for startup
sleep 10

# Check data
curl http://localhost:8080/customers
```

**Expected:** Still see 3 sample customers

**Why?** Volume persists data even when containers are removed!

### Step 4: Access Database Directly (Optional)

Connect to containerized PostgreSQL:

```bash
# Access PostgreSQL shell in container
docker exec -it simple-crm-lite-db psql -U postgres -d simplecrmlite
```

**Inside PostgreSQL:**
```sql
-- View customers table
SELECT * FROM customer;

-- Exit
\q
```

**✅ Success!** Complete containerized stack working perfectly!

---


## Troubleshooting Common Issues

### Issue 1: Port Already in Use

**Error:** `Bind for 0.0.0.0:8080 failed: port is already allocated`

**Cause:** Something else using port 8080 (maybe your Module 3 simple-crm still running)

**Solution:**
```bash
# Find what's using port 8080
lsof -i :8080

# Kill the process (if needed)
kill -9 <PID>

# Or change port in docker-compose.yml
ports:
  - "8081:8080"  # Use 8081 on host instead
```

### Issue 2: Database Connection Failed

**Error in logs:** `Connection refused` or `Unknown host`

**Cause:** App trying to connect before database is ready

**Solution 1 - Wait longer:**
```bash
# Stop everything
docker-compose down

# Start and wait
docker-compose up -d
sleep 20  # Wait 20 seconds
docker-compose logs app
```

**Solution 2 - Check healthcheck:**
```bash
docker-compose ps
# DB should show "(healthy)"
```

**Solution 3 - Verify network:**
```bash
docker network ls
docker network inspect simple-crm-lite_default
# Both containers should be in same network
```

### Issue 3: Changes Not Reflected

**Problem:** Made code changes but container still runs old code

**Cause:** Need to rebuild image

**Solution:**
```bash
# Stop containers
docker-compose down

# Rebuild and start
docker-compose up -d --build
```

**The `--build` flag forces rebuild**

### Issue 4: "No space left on device"

**Error:** Docker build fails with disk space error

**Cause:** Old images/containers filling disk

**Solution:**
```bash
# Remove stopped containers
docker container prune -f

# Remove unused images
docker image prune -a -f

# Remove unused volumes
docker volume prune -f

# Nuclear option - remove everything
docker system prune -a --volumes -f
```

### Issue 5: Sample Data Not Loading

**Problem:** GET /customers returns empty array

**Cause:** DataLoader not running or database already has data

**Solution:**

```bash
# Access database
docker exec -it simple-crm-lite-db psql -U postgres -d simplecrmlite

# Check if customers exist
SELECT * FROM customer;

# If table is empty, restart app container
docker-compose restart app

# Check logs
docker-compose logs app
# Should see "Sample data loaded"
```

---

## Useful Docker Compose Commands

```bash
# Start all services in background
docker-compose up -d

# Start and rebuild images
docker-compose up -d --build

# Stop all services (containers remain)
docker-compose stop

# Start stopped services
docker-compose start

# Stop and remove all containers, networks
docker-compose down

# Stop and remove everything including volumes (data lost!)
docker-compose down -v

# View logs from all services
docker-compose logs

# View logs from specific service
docker-compose logs app

# Follow logs in real-time
docker-compose logs -f

# View running services
docker-compose ps

# Restart specific service
docker-compose restart app

# Execute command in service container
docker-compose exec app /bin/sh

# View resource usage
docker-compose stats
```

---

