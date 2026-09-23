# Build stage
FROM maven:3.9.9-eclipse-temurin-21-alpine AS builder

WORKDIR /build

# Pre-fetch dependencies for layer caching
COPY pom.xml .
RUN mvn dependency:go-offline -B || true

# Copy source code and build package
COPY src ./src
RUN mvn clean package -DskipTests

# Runtime stage
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY --from=builder /build/target/auth-service-*.jar app.jar

EXPOSE 8081

ENTRYPOINT ["java", "-jar", "app.jar"]
