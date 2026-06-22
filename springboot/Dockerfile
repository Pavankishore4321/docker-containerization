# ─────────────────────────────────────────────────────────────────────────────
# Dockerfile: Spring Boot Backend — Multi-Stage Build
# Project:    AspTax, Nagpur Metro, Aragen, Kaveri Seeds (Incresol)
#
# Stage 1 (builder): Maven compiles source → produces .jar
# Stage 2 (runtime): Copies .jar into lightweight JRE image
#
# Build: docker build -t asptax-api:latest .
# Run:   docker run -d -p 8080:8080 \
#          -e SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/asptax \
#          -e SPRING_DATASOURCE_USERNAME=root \
#          -e SPRING_DATASOURCE_PASSWORD=secret \
#          asptax-api:latest
# ─────────────────────────────────────────────────────────────────────────────

# ── STAGE 1: BUILD ────────────────────────────────────────────────────────────
FROM maven:3.8.7-openjdk-8 AS builder

WORKDIR /app

# Copy pom.xml first — Docker cache optimization
# Maven dependencies only re-downloaded if pom.xml changes
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source code and build
COPY src ./src
RUN mvn clean package -DskipTests

# Verify JAR was created
RUN ls -lh target/*.jar

# ── STAGE 2: RUNTIME ──────────────────────────────────────────────────────────
# Use slim JRE — no Maven, no source code, much smaller image
FROM openjdk:8-jre-slim

WORKDIR /app

# Create non-root user for security (never run apps as root)
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Copy JAR from builder stage
COPY --from=builder /app/target/*.jar app.jar

# Set ownership
RUN chown appuser:appuser app.jar

# Switch to non-root user
USER appuser

# Expose Spring Boot default port
EXPOSE 8080

# Health check using Spring Boot Actuator
HEALTHCHECK --interval=30s --timeout=10s --retries=3 --start-period=40s \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# JVM tuning for containers
ENV JAVA_OPTS="-Xms256m -Xmx512m \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -Djava.security.egd=file:/dev/./urandom"

# Start Spring Boot app
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
