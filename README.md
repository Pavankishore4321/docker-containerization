# docker-containerization

Docker configurations built from real containerized deployments at
**Incresol Software Services**. Covers Angular frontend, Spring Boot backend,
MongoDB replica set, and a complete full-stack docker-compose setup.

---

## Repository Structure

```
docker-containerization/
├── angular-nginx/
│   ├── Dockerfile          # Multi-stage: ng build → Nginx (no Node.js in final image)
│   └── nginx.conf          # Nginx config — Angular routing, gzip, security headers
├── springboot/
│   └── Dockerfile          # Multi-stage: Maven build → lightweight JRE image
├── mongodb/
│   └── docker-compose-mongo-replica.yml   # 3-node replica set (Primary + Secondary + Arbiter)
├── docker-compose.yml      # Full stack: Angular + Spring Boot + MongoDB + Nginx proxy
├── nginx-proxy.conf        # Nginx routes /api → Spring Boot, / → Angular
└── README.md
```

---

## Angular Multi-Stage Dockerfile

**Project: AspTax, P-Collab**

The most important file in this repo. A multi-stage build that compiles
Angular inside Docker and serves the output with Nginx — no Node.js in
the final image.

```
Stage 1 (builder):   node:18-alpine
  → npm install
  → ng build --configuration=production
  → produces dist/asptax-web/

Stage 2 (runtime):   nginx:1.25-alpine
  → COPY dist/ into /usr/share/nginx/html
  → Final image: ~25MB (vs ~500MB with Node.js)
```

### Why this matters

| | With Node.js in image | Multi-stage (this) |
|---|---|---|
| Image size | ~500MB | ~25MB |
| Attack surface | Large (Node + npm) | Only Nginx |
| Build reproducible | Depends on host Node | Always same |
| Production ready | No | Yes |

### Build and run

```bash
cd angular-nginx

# Build the image
docker build -t asptax-web:latest .

# Run on port 80
docker run -d -p 80:80 --name asptax-web asptax-web:latest

# Verify
curl http://localhost/health
```

### nginx.conf features
- `try_files $uri /index.html` — Angular client-side routing works on refresh
- Static asset caching with 1-year expiry (safe — Angular uses hashed filenames)
- Gzip compression for JS, CSS, JSON
- Security headers: X-Frame-Options, X-XSS-Protection, X-Content-Type-Options

---

## Spring Boot Multi-Stage Dockerfile

**Project: AspTax, Nagpur Metro, Aragen, Kaveri Seeds, Daimler**

```
Stage 1 (builder):   maven:3.8.7-openjdk-8
  → mvn dependency:go-offline (cached if pom.xml unchanged)
  → mvn clean package -DskipTests
  → produces target/app.jar

Stage 2 (runtime):   openjdk:8-jre-slim
  → Copies only the JAR — no Maven, no source code
  → Runs as non-root user (security best practice)
```

### Build and run

```bash
cd springboot

# Build image
docker build -t asptax-api:latest .

# Run with environment variables
docker run -d -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e SPRING_DATASOURCE_URL=jdbc:mysql://db-host:3306/asptax \
  -e SPRING_DATASOURCE_USERNAME=root \
  -e SPRING_DATASOURCE_PASSWORD=secret \
  --name asptax-api \
  asptax-api:latest

# Check health
curl http://localhost:8080/actuator/health
```

---

## MongoDB 3-Node Replica Set

**Project: Node Application High-Availability**

```
mongo-primary   (port 27017)  ← all writes go here, priority: 2
mongo-secondary (port 27018)  ← replicates from primary, priority: 1
mongo-arbiter   (port 27019)  ← votes in elections, NO data stored
```

### Start replica set

```bash
# Start all 3 MongoDB nodes
docker-compose -f mongodb/docker-compose-mongo-replica.yml up -d

# Verify replica set status (after init)
docker exec -it mongo-primary mongosh --eval "rs.status()"
```

### Connection string (from your app)

```
mongodb://mongoadmin:changeme@mongo-primary:27017,mongo-secondary:27017/asptax?replicaSet=rs0&authSource=admin
```

### Why replica set?
- **Automatic failover** — if primary dies, secondary is elected new primary within ~10 seconds
- **Data redundancy** — secondary has a full copy of all data
- **Zero downtime** — application stays connected during failover

---

## Full Stack docker-compose

**Project: AspTax complete environment**

Runs the entire application stack with one command.

```bash
# Start everything
docker-compose up -d

# Check all services are healthy
docker-compose ps

# View logs
docker-compose logs -f

# Stop everything
docker-compose down
```

### Services and ports

| Service | Container | Port | Purpose |
|---|---|---|---|
| nginx | asptax-nginx | 80 | Reverse proxy — entry point |
| angular | asptax-angular | internal | Angular frontend |
| springboot | asptax-api | 8080 | Spring Boot REST API |
| mongodb | asptax-mongodb | 27017 | MongoDB database |

### Traffic flow

```
User → http://localhost
         │
       Nginx (port 80)
         ├── /api/*  → Spring Boot (port 8080)
         └── /*      → Angular (port 80)
                              ↕
                          MongoDB (port 27017)
```

---

## Real Projects These Dockerfiles Supported

| Project | Stack Containerized | Dockerfile Used |
|---|---|---|
| AspTax | Angular + Spring Boot + MySQL | angular-nginx/Dockerfile + springboot/Dockerfile |
| P-Collab | Angular + Azure App Service | angular-nginx/Dockerfile |
| Node App HA | Node.js + MongoDB Replica Set | mongodb/docker-compose-mongo-replica.yml |
| Multi-client deployments | Spring Boot + Tomcat → Docker | springboot/Dockerfile |

---

## Docker Best Practices Used

| Practice | Where applied |
|---|---|
| Multi-stage builds | Both Angular and Spring Boot Dockerfiles |
| Non-root user | Spring Boot container runs as `appuser` |
| Health checks | All services in docker-compose.yml |
| Layer caching | package.json and pom.xml copied before source |
| Minimal base images | Alpine variants used where possible |
| No secrets in image | Passwords passed via environment variables |
| `.dockerignore` | Add `node_modules/`, `dist/`, `target/` to avoid bloat |

---

## Author

**Pavan Kishore Nakka**
DevOps & Cloud Engineer | 3+ Years Experience
AWS Certified Solutions Architect – Associate | AWS Certified Cloud Practitioner

[LinkedIn](https://www.linkedin.com/in/nakka-pavan-kishore)
