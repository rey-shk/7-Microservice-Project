# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS builder

WORKDIR /src

# Restore project dependencies
COPY payment-service.csproj .
RUN dotnet restore

# Copy source and publish release build
COPY . .
RUN dotnet publish -c Release -o /app/publish --no-restore

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0

WORKDIR /app

COPY --from=builder /app/publish .

EXPOSE 8085

ENTRYPOINT ["dotnet", "payment-service.dll"]
