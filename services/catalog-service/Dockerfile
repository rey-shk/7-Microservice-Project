# Build stage
FROM golang:1.22-alpine AS builder

WORKDIR /build

# Pre-fetch Go modules for layer caching
COPY go.mod go.sum ./
RUN go mod download

# Copy source and compile statically
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o catalog-service .

# Runtime stage
FROM alpine:3.20

WORKDIR /app

RUN apk --no-cache add ca-certificates tzdata

COPY --from=builder /build/catalog-service /app/catalog-service

EXPOSE 8082

ENTRYPOINT ["/app/catalog-service"]
