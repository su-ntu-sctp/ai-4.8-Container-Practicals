# Lesson 4.8: Containerizing Simple-CRM-Lite (Saturday Coaching)

## Lesson Overview

In this lesson you will apply the Docker and containerization concepts you learned in previous lessons to a Spring Boot application - the **simple-crm-lite** project.

You will run **both** the application and its PostgreSQL database in containers with Docker Compose. There is no local PostgreSQL to install and no `mvn spring-boot:run` - you go straight from writing the code to running it in containers. Finally, you will prove that your data survives the containers being removed and recreated.

## Why Simple-CRM-Lite?

Today's focus is learning Docker, not debugging complex applications. Your full simple-crm (with Product, Interaction entities, relationships) can introduce issues unrelated to Docker - like JPA lazy loading in containers or relationship serialization errors. Using a simplified baseline (Customer entity only) ensures everyone learns Docker concepts successfully.

---

## Learning Objectives

By the end of this session, you will be able to:

1. **Scaffold** a simplified Spring Boot REST API project (simple-crm-lite) with a single entity
2. **Create** a multi-stage Dockerfile for a Spring Boot application with PostgreSQL dependencies
3. **Configure** Docker Compose to run the application and a PostgreSQL database together
4. **Test** containerized CRUD endpoints
5. **Verify** that data persists in a Docker volume after `docker compose down` and `docker compose up`

---

## Prerequisites

Before starting this lesson, ensure you have:

- Completed Lessons 4.4 (Local Containerization) and 4.6 (Docker Compose)
- Docker Desktop installed and running
- VS Code (Java 21 and Maven are optional - the application is built inside Docker)
- **Port 5432 free.** The database container uses port 5432. If you installed PostgreSQL on your computer, stop it first, as in Lesson 4.6:

```bash
# Windows (WSL)
sudo service postgresql stop

# macOS (Homebrew) - use your installed version
brew services stop postgresql@16
```

**Note:** You do NOT need PostgreSQL installed on your computer. Both the application and the database run in containers.

---

## Part 0: Create Simple-CRM-Lite

Let's write the simplified project this lesson uses. You will not run it with Maven - in Parts 2 and 3 it runs in containers.

### Step 1: Generate the Project with Spring Initializr

1. Go to [https://start.spring.io](https://start.spring.io)
2. Fill in the project details:
   - **Project:** Maven
   - **Language:** Java
   - **Spring Boot:** leave the default version
   - **Group:** `com.example`
   - **Artifact:** `simple-crm-lite`
   - **Name:** `simple-crm-lite`
   - **Package name:** `com.example.simplecrmlite`
   - **Packaging:** Jar
   - **Java:** 21
3. Under **Dependencies**, add:
   - **Spring Web**
   - **Spring Data JPA**
   - **PostgreSQL Driver**
4. Click **Generate**, extract the `.zip`, and open the project in VS Code

**Important:** Keep the **Artifact** exactly as `simple-crm-lite` and leave the default version as `0.0.1-SNAPSHOT`. The Dockerfile later in this lesson references the exact filename this produces (`simple-crm-lite-0.0.1-SNAPSHOT.jar`) — matching it now avoids a mismatch later.

### Step 2: Configure application.properties

Open `src/main/resources/application.properties` and replace its contents with:

```properties
# Application name
spring.application.name=simple-crm-lite

# Database connection - values come from docker-compose.yml (Part 3)
spring.datasource.url=${SPRING_DATASOURCE_URL}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD}

# JPA configuration - create or update the customer table automatically
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

# Server configuration
server.port=8080
```

**What are the `${...}` values?**
- They are placeholders filled in from **environment variables** when the application starts - the same pattern as Lesson 4.6
- Docker Compose sets `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME` and `SPRING_DATASOURCE_PASSWORD` for the app container in Part 3
- This keeps database settings out of your code: the same image works with any database, only the environment changes
- Because these values only exist inside Docker Compose, the application is meant to run with `docker compose`, not `mvn spring-boot:run`

### Step 3: Create the Customer Entity

Create `src/main/java/com/example/simplecrmlite/model/Customer.java`:

```java
package com.example.simplecrmlite.model;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;

@Entity
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String firstName;
    private String lastName;
    private String email;
    private String contactNo;
    private String jobTitle;
    private Integer yearOfBirth;

    public Customer() {
    }

    public Customer(String firstName, String lastName, String email,
                     String contactNo, String jobTitle, Integer yearOfBirth) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.email = email;
        this.contactNo = contactNo;
        this.jobTitle = jobTitle;
        this.yearOfBirth = yearOfBirth;
    }

    // Getters and setters

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getFirstName() {
        return firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

    public String getLastName() {
        return lastName;
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getContactNo() {
        return contactNo;
    }

    public void setContactNo(String contactNo) {
        this.contactNo = contactNo;
    }

    public String getJobTitle() {
        return jobTitle;
    }

    public void setJobTitle(String jobTitle) {
        this.jobTitle = jobTitle;
    }

    public Integer getYearOfBirth() {
        return yearOfBirth;
    }

    public void setYearOfBirth(Integer yearOfBirth) {
        this.yearOfBirth = yearOfBirth;
    }
}
```

### Step 4: Create the Repository

Create `src/main/java/com/example/simplecrmlite/repository/CustomerRepository.java`:

```java
package com.example.simplecrmlite.repository;

import com.example.simplecrmlite.model.Customer;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {
}
```

### Step 5: Create the REST Controller

Create `src/main/java/com/example/simplecrmlite/controller/CustomerController.java`:

```java
package com.example.simplecrmlite.controller;

import com.example.simplecrmlite.model.Customer;
import com.example.simplecrmlite.repository.CustomerRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/customers")
public class CustomerController {

    @Autowired
    private CustomerRepository customerRepository;

    @GetMapping
    public List<Customer> getAllCustomers() {
        return customerRepository.findAll();
    }

    @GetMapping("/{id}")
    public ResponseEntity<Customer> getCustomerById(@PathVariable Long id) {
        return customerRepository.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<Customer> createCustomer(@RequestBody Customer customer) {
        Customer saved = customerRepository.save(customer);
        return ResponseEntity.status(HttpStatus.CREATED).body(saved);
    }

    @PutMapping("/{id}")
    public ResponseEntity<Customer> updateCustomer(@PathVariable Long id,
                                                     @RequestBody Customer updatedCustomer) {
        return customerRepository.findById(id)
                .map(existing -> {
                    existing.setFirstName(updatedCustomer.getFirstName());
                    existing.setLastName(updatedCustomer.getLastName());
                    existing.setEmail(updatedCustomer.getEmail());
                    existing.setContactNo(updatedCustomer.getContactNo());
                    existing.setJobTitle(updatedCustomer.getJobTitle());
                    existing.setYearOfBirth(updatedCustomer.getYearOfBirth());
                    Customer saved = customerRepository.save(existing);
                    return ResponseEntity.ok(saved);
                })
                .orElse(ResponseEntity.notFound().build());
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteCustomer(@PathVariable Long id) {
        if (!customerRepository.existsById(id)) {
            return ResponseEntity.notFound().build();
        }
        customerRepository.deleteById(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Step 6: Seed Sample Data on Startup

Create `src/main/java/com/example/simplecrmlite/SimpleCrmLiteApplication.java` (or update the main class Spring Initializr generated):

```java
package com.example.simplecrmlite;

import com.example.simplecrmlite.model.Customer;
import com.example.simplecrmlite.repository.CustomerRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class SimpleCrmLiteApplication {

    public static void main(String[] args) {
        SpringApplication.run(SimpleCrmLiteApplication.class, args);
    }

    @Bean
    CommandLineRunner loadSampleData(CustomerRepository customerRepository) {
        return args -> {
            if (customerRepository.count() == 0) {
                customerRepository.save(new Customer("Bruce", "Wayne",
                        "bruce@wayneenterprises.com", "11122233", "CEO", 1975));
                customerRepository.save(new Customer("Diana", "Prince",
                        "diana@themyscira.gov", "22233344", "Ambassador", 1980));
                customerRepository.save(new Customer("Clark", "Kent",
                        "clark@dailyplanet.com", "33344455", "Reporter", 1978));
                System.out.println("✅ Sample data loaded: 3 customers added");
            }
        };
    }
}
```

**Note:** The `if (customerRepository.count() == 0)` check means the 3 sample customers are only added when the `customer` table is **empty**. You will use this in Part 5 to prove that your data persists.

---

## Part 1: What We'll Build

### Target Architecture

```
┌──────────────────────────────────────────────────────┐
│   Your Computer                                      │
│                                                      │
│   localhost:8080              localhost:5432         │
│        │                            │                │
│  ┌─────┼────────────────────────────┼─────────────┐  │
│  │     │  Docker Network: simple-crm-lite_default │  │
│  │     ▼                            ▼             │  │
│  │  ┌────────────────────┐   ┌──────────────────┐ │  │
│  │  │ app                │   │ db               │ │  │
│  │  │ simple-crm-lite    │──▶│ PostgreSQL 16    │ │  │
│  │  │ Spring Boot :8080  │   │ :5432            │ │  │
│  │  └────────────────────┘   └────────┬─────────┘ │  │
│  └────────────────────────────────────┼───────────┘  │
│                                       ▼              │
│                         Volume: postgres-data        │
│                         (the data lives here)        │
└──────────────────────────────────────────────────────┘
```

### Key Ideas

| Idea | What it means in this lesson |
|------|------------------------------|
| **Service name as hostname** | The app connects to `jdbc:postgresql://db:5432/simplecrmlite`. Inside the app container, `localhost` means the app container itself, so it uses the service name `db` instead |
| **Configuration from the environment** | Docker Compose sets `SPRING_DATASOURCE_*`; `application.properties` reads them |
| **Start order** | The database has a healthcheck; the app waits until the database is healthy |
| **Data in a named volume** | PostgreSQL stores its files in the `postgres-data` volume, not inside the container, so the data survives when containers are removed |

### Why Containerize?

**Benefits you'll experience today:**
1. **Consistency** - Works the same on any machine
2. **Isolation** - App dependencies don't affect your computer
3. **Easy cleanup** - Remove containers, everything's gone (except data you chose to keep in a volume)
4. **Professional practice** - This is how production apps run

---

## Part 2: Create the Dockerfile

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

**Note — filename alternative:** The `COPY` line above hardcodes the exact JAR filename (`simple-crm-lite-0.0.1-SNAPSHOT.jar`), which matches the Artifact/version you set in Part 0, Step 1. If you ever rename your project or bump the version and this line stops matching, you can swap it for the wildcard pattern used in Lessons 4.4/4.6 instead, which works regardless of the exact filename:

```dockerfile
COPY --from=build /app/target/*.jar app.jar
```

Either approach works — the hardcoded version above is used here since Part 0 fixes the exact project name and version in advance.

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

**Don't run this image on its own with `docker run`** — the app needs a database and the `SPRING_DATASOURCE_*` environment variables. Docker Compose provides both in Part 3, and builds the image for you with `--build`.

---

## Part 3: Run the App and Database with Docker Compose

### Step 1: Why Docker Compose?

The application needs a database. Docker Compose starts both containers, connects them on a shared network and creates the volume - with one command.

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
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d simplecrmlite"]
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

**Note — Postgres version:** This lesson uses `postgres:16-alpine`, while Lesson 4.6 used `postgres:15`. Any recent PostgreSQL version works for these exercises.

### Step 3: Understanding docker-compose.yml

#### Database Service (db)

**`image: postgres:16-alpine`** — official lightweight PostgreSQL 16 image

**`environment`** — creates the `simplecrmlite` database and the `postgres` user the first time the container starts

**`ports: "5432:5432"`** — publishes PostgreSQL on `localhost:5432` so tools on your computer (for example a database GUI) can connect. The app does not need this - it connects over the Compose network. Port 5432 on your computer must be free (see Prerequisites)

**`volumes: postgres-data:/var/lib/postgresql/data`** — PostgreSQL keeps its data files in the named volume `postgres-data`. The volume is separate from the container, so it survives `docker compose down`

**`healthcheck`** — `pg_isready` reports when the database is ready to accept connections

#### Application Service (app)

**`build:`** — builds the image from the Dockerfile in the current directory

**`SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/simplecrmlite`** — fills in the `${SPRING_DATASOURCE_URL}` placeholder in `application.properties`. The host is `db` (the service name), not `localhost`

**`depends_on: condition: service_healthy`** — the app starts only after the database passes its healthcheck

**`restart: unless-stopped`** — automatically restarts the app container if it crashes

### Step 4: Start Everything with Docker Compose

```bash
cd simple-crm-lite

docker compose up -d --build
```

`--build` builds the application image from your Dockerfile before starting the containers.

**Expected output (after the build):**
```
[+] Running 4/4
 ✔ Network simple-crm-lite_default          Created
 ✔ Volume "simple-crm-lite_postgres-data"   Created
 ✔ Container simple-crm-lite-db             Healthy
 ✔ Container simple-crm-lite-app            Started
```

### Step 5: Watch the Application Logs

```bash
docker compose logs -f app
```

**Look for:**
```
simple-crm-lite-app  | Started SimpleCrmLiteApplication in X.XXX seconds
simple-crm-lite-app  | ✅ Sample data loaded: 3 customers added
```

Press `Ctrl+C` to stop viewing the logs — the containers keep running.

### Step 6: Verify Both Containers Are Running

```bash
docker compose ps
```

**Expected output:**
```
NAME                  IMAGE                 SERVICE   STATUS                    PORTS
simple-crm-lite-app   simple-crm-lite-app   app       Up 30 seconds             0.0.0.0:8080->8080/tcp
simple-crm-lite-db    postgres:16-alpine    db        Up 45 seconds (healthy)   0.0.0.0:5432->5432/tcp
```

The database shows `(healthy)` because it has a healthcheck. The app shows `Up`.

---

## Part 4: Test the CRUD API

### Step 1: Read All Customers

```bash
curl http://localhost:8080/customers
```

**Expected:** JSON array with the 3 sample customers (Bruce Wayne, Diana Prince, Clark Kent)

### Step 2: Create (POST)

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

**Expected:** 201 CREATED with the new customer. Note the `id` in the response (usually `4`) and use it in the next steps.

### Step 3: Read One (GET)

```bash
curl http://localhost:8080/customers/4
```

**Expected:** 200 OK with Tony Stark's data

### Step 4: Update (PUT)

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

**Expected:** 200 OK with the updated data

### Step 5: Delete (DELETE)

```bash
curl -X DELETE http://localhost:8080/customers/4
```

**Expected:** 204 NO CONTENT

Verify the deletion:
```bash
curl -i http://localhost:8080/customers/4
```

**Expected:** `HTTP/1.1 404` — the customer is gone

**✅ Success!** Your containerized application reads and writes to the containerized database.

---

## Part 5: Prove the Data Persists

Containers are disposable: `docker compose down` **removes** them. Your data must survive this, because PostgreSQL stores it in the `postgres-data` volume, not in the container.

You will write a customer, remove all the containers, start new ones, and check the customer is still there.

### Step 1: Write a Customer You Will Keep

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

**Note the `id`** in the response (usually `5`, because Tony used `4`). This time, do NOT delete it.

### Step 2: Check the Customer Is in the Database

Ask PostgreSQL directly, not the app:

```bash
docker compose exec db psql -U postgres -d simplecrmlite \
  -c "SELECT id, first_name, last_name, email FROM customer ORDER BY id;"
```

**Expected:** 4 rows — the 3 sample customers and Natasha Romanoff

### Step 3: Remove the Containers

```bash
docker compose down
```

**Expected output:**
```
[+] Running 3/3
 ✔ Container simple-crm-lite-app        Removed
 ✔ Container simple-crm-lite-db         Removed
 ✔ Network simple-crm-lite_default      Removed
```

The containers and the network are gone. Notice the **volume is not in this list**. Check:

```bash
docker compose ps -a      # no containers listed
docker volume ls          # the volume is still there
```

**Expected from `docker volume ls`:**
```
DRIVER    VOLUME NAME
local     simple-crm-lite_postgres-data
```

### Step 4: Start New Containers

```bash
docker compose up -d
docker compose logs -f app
```

Wait for `Started SimpleCrmLiteApplication`, then press `Ctrl+C`. These are brand-new containers, attached to the same volume.

### Step 5: Check the Evidence

**1. Natasha is still there** (use the `id` from Step 1):
```bash
curl http://localhost:8080/customers/5
```

**Expected:** 200 OK with Natasha Romanoff's data

**2. The database still has 4 rows:**
```bash
docker compose exec db psql -U postgres -d simplecrmlite \
  -c "SELECT id, first_name, last_name FROM customer ORDER BY id;"
```

**3. The sample data was NOT loaded again:**
```bash
docker compose logs app | grep "Sample data loaded"
```

**Expected:** no output. The new app container found customers already in the table, so `count() == 0` was false and the seeding was skipped.

| Evidence | What you see | Why |
|----------|--------------|-----|
| `curl /customers/5` | Natasha is returned | The row was read from the volume |
| `psql SELECT` | 4 rows, same ids | The database files survived `down` |
| `grep "Sample data loaded"` | No output | The table was not empty on startup |

**✅ Data persisted** — the containers were replaced, the data was not.

### Step 6: Contrast — Delete the Volume

Now see what happens without the volume:

```bash
docker compose down -v      # -v also removes the named volume
docker volume ls            # simple-crm-lite_postgres-data is gone
docker compose up -d
docker compose logs -f app  # wait for "Started", then Ctrl+C
```

Check again:
```bash
curl -i http://localhost:8080/customers/5             # 404 - Natasha is gone
docker compose logs app | grep "Sample data loaded"   # ✅ Sample data loaded: 3 customers added
```

PostgreSQL started with an empty volume, so the table was empty and the 3 sample customers were loaded again. **The data lived in the volume, not in the container.**

> **Remember:** `docker compose down` keeps volumes. `docker compose down -v` deletes them — and your data with them. Never use `-v` on data you need.

---

## Useful Docker Compose Commands

```bash
# Build images and start all services in the background
docker compose up -d --build

# Start all services in the background
docker compose up -d

# Stop all services (containers remain)
docker compose stop

# Start stopped services
docker compose start

# Stop and remove containers and network (volumes are kept)
docker compose down

# Stop and remove everything including volumes (data lost!)
docker compose down -v

# View logs from all services
docker compose logs

# View logs from one service
docker compose logs app

# Follow logs in real time
docker compose logs -f

# View services and their status
docker compose ps

# Restart one service
docker compose restart app

# Open a PostgreSQL shell in the db container
docker compose exec db psql -U postgres -d simplecrmlite

# Open a shell in the app container
docker compose exec app /bin/sh

# List volumes
docker volume ls

# View resource usage
docker stats
```

---

## Troubleshooting Common Issues

### Issue 1: Port Already in Use

**Error:** `Bind for 0.0.0.0:5432 failed: port is already allocated` (or `0.0.0.0:8080`)

**Port 5432:** PostgreSQL is probably running on your computer. Stop it:
```bash
# Windows (WSL)
sudo service postgresql stop

# macOS (Homebrew)
brew services stop postgresql@16
```

**Port 8080:** Find and stop whatever is using it:
```bash
lsof -i :8080
kill -9 <PID>
```

Then run `docker compose up -d` again.

---

### Issue 2: Database Connection Failed

**Error in logs:** `Connection refused` or `UnknownHostException: db`

**Check the database is healthy:**
```bash
docker compose ps
# db should show "(healthy)"
```

**Check the connection settings** in `docker-compose.yml`: the URL must be `jdbc:postgresql://db:5432/simplecrmlite` — `db`, not `localhost`.

**Check the network:**
```bash
docker network inspect simple-crm-lite_default
```

---

### Issue 3: `Could not resolve placeholder 'SPRING_DATASOURCE_URL'`

The app started without the environment variables from `docker-compose.yml`. This happens if you run it with `mvn spring-boot:run`, or if the `environment:` section of the `app` service is missing or misspelled. Run the app with `docker compose up -d --build`.

---

### Issue 4: Changes Not Reflected

You changed the code but the app behaves the same. Rebuild the image:
```bash
docker compose up -d --build
```

---

### Issue 5: My Data Disappeared

- **You ran `docker compose down -v`** — the `-v` deletes the volume and all its data
- **You ran Compose from a different folder** — the volume name starts with the project folder name (`simple-crm-lite_postgres-data`). A renamed or copied folder (for example `simple-crm-lite (1)`) gets a new, empty volume. Check with `docker volume ls`

---

### Issue 6: Sample Data Not Loading

The sample data is only added when the `customer` table is **empty**. If you already have customers, that is persistence working — not a bug.

```bash
docker compose exec db psql -U postgres -d simplecrmlite -c "SELECT * FROM customer;"
```

To start completely fresh:
```bash
docker compose down -v
docker compose up -d
```

---

### Issue 7: "No space left on device"

```bash
docker container prune -f
docker image prune -a -f
docker volume prune -f      # careful: removes unused volumes and their data
```
