---
title: "Containerization with Docker for .NET Applications"
date_created: 2023-04-02
date_updated: 2023-04-02
authors: ["Repository Maintainers"]
tags: ["docker", "containers", "dotnet", "microservices", "devops"]
difficulty: "intermediate"
---

# Containerization with Docker for .NET Applications

## Overview

Containerization has revolutionized how applications are built, deployed, and scaled. Docker provides a platform for packaging applications into standardized units called containers, which include the application code, runtime, system tools, libraries, and settings. This document covers Docker concepts, best practices, and patterns specifically for .NET developers, focusing on how to effectively containerize .NET applications.

## Docker Fundamentals

### Container vs. Virtual Machines

Containers differ from traditional virtual machines in several important ways:

| Aspect | Containers | Virtual Machines |
|--------|------------|-----------------|
| Virtualization | OS-level virtualization | Hardware virtualization |
| Size | Lightweight (MBs) | Heavy (GBs) |
| Boot Time | Seconds | Minutes |
| Isolation | Process isolation | Complete isolation |
| Resource Usage | Low overhead | Higher overhead |
| Operating System | Shares host OS kernel | Requires guest OS |

### Key Docker Components

- **Docker Engine**: Runtime environment for containers
- **Docker Images**: Read-only templates for creating containers
- **Docker Containers**: Running instances of Docker images
- **Dockerfile**: Script to build Docker images
- **Docker Compose**: Tool for defining multi-container applications
- **Docker Registry**: Repository for storing and sharing Docker images

## Containerizing .NET Applications

### .NET Container Images

Microsoft provides official Docker images for .NET:

- **SDK images** (`mcr.microsoft.com/dotnet/sdk`): For building applications
- **Runtime images** (`mcr.microsoft.com/dotnet/aspnet`): For running ASP.NET Core applications
- **Runtime-deps images** (`mcr.microsoft.com/dotnet/runtime-deps`): Minimal images for self-contained apps

### Basic Dockerfile

A simple Dockerfile for an ASP.NET Core application:

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src

# Copy csproj and restore dependencies
COPY ["MyApp.csproj", "./"]
RUN dotnet restore

# Copy the rest of the code and build
COPY . .
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

# Publish stage
FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish

# Final stage
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Building and Running

Basic Docker commands for .NET applications:

```bash
# Build the Docker image
docker build -t myapp:1.0 .

# Run the container
docker run -d -p 8080:80 --name myapp-container myapp:1.0

# View logs
docker logs myapp-container

# Stop the container
docker stop myapp-container

# Remove the container
docker rm myapp-container
```

## Docker Best Practices for .NET

### Multi-stage Builds

Multi-stage builds reduce image size and improve security:

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src

# Copy csproj files and restore as distinct layers
COPY ["MyApp.API/MyApp.API.csproj", "MyApp.API/"]
COPY ["MyApp.Core/MyApp.Core.csproj", "MyApp.Core/"]
COPY ["MyApp.Infrastructure/MyApp.Infrastructure.csproj", "MyApp.Infrastructure/"]
RUN dotnet restore "MyApp.API/MyApp.API.csproj"

# Copy everything else and build
COPY . .
RUN dotnet build "MyApp.API/MyApp.API.csproj" -c Release -o /app/build

# Test stage
FROM build AS test
WORKDIR /src
RUN dotnet test "MyApp.Tests/MyApp.Tests.csproj" --logger "console;verbosity=detailed"

# Publish stage
FROM build AS publish
RUN dotnet publish "MyApp.API/MyApp.API.csproj" -c Release -o /app/publish

# Final stage
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Create non-root user
RUN adduser --disabled-password --gecos "" appuser
USER appuser

EXPOSE 80
EXPOSE 443
ENTRYPOINT ["dotnet", "MyApp.API.dll"]
```

### Optimizing Docker Images

#### Layer Caching

Structure Dockerfiles to leverage caching:

```dockerfile
# Copy only csproj files first and restore
COPY ["MyApp.API/MyApp.API.csproj", "MyApp.API/"]
COPY ["MyApp.Core/MyApp.Core.csproj", "MyApp.Core/"]
RUN dotnet restore "MyApp.API/MyApp.API.csproj"

# Then copy everything else
COPY . .
```

#### Minimizing Image Size

Techniques to reduce image size:

```dockerfile
# Use Alpine-based images
FROM mcr.microsoft.com/dotnet/aspnet:6.0-alpine AS final

# Remove unnecessary files
RUN apt-get update \
    && apt-get install -y --no-install-recommends some-package \
    && rm -rf /var/lib/apt/lists/*

# Use .dockerignore to exclude files
# .dockerignore contents:
# */bin
# */obj
# .git
# .vs
# .vscode
```

#### Self-contained Applications

Creating minimal images with self-contained applications:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src

COPY ["MyApp.csproj", "./"]
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release -r linux-musl-x64 --self-contained true /p:PublishTrimmed=true -o /app/publish

# Use runtime-deps for self-contained apps
FROM mcr.microsoft.com/dotnet/runtime-deps:6.0-alpine AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["./MyApp"]
```

### Environment Variables and Configuration

Managing configuration in Docker containers:

```dockerfile
# Set default environment variables in Dockerfile
ENV ASPNETCORE_ENVIRONMENT=Production
ENV ASPNETCORE_URLS=http://+:80

# Override at runtime
# docker run -e "ASPNETCORE_ENVIRONMENT=Development" -e "ConnectionStrings__DefaultConnection=..." myapp
```

ASP.NET Core code for using environment variables:

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureAppConfiguration((hostingContext, config) =>
        {
            // Add support for environment variables prefixed with "MYAPP_"
            config.AddEnvironmentVariables(prefix: "MYAPP_");
        })
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });
```

### Health Checks

Implementing health checks for container orchestration:

```csharp
// Add health checks in Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddHealthChecks()
        .AddDbContextCheck<ApplicationDbContext>()
        .AddCheck("HTTP Endpoint", new HttpHealthCheck("https://api.example.com"))
        .AddCheck<CustomHealthCheck>("Custom");
        
    // Register with liveness and readiness paths
    services.AddHealthChecksUI()
        .AddInMemoryStorage();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // ...
    
    // Configure health check endpoints
    app.UseHealthChecks("/health");
    
    app.UseHealthChecks("/health/ready", new HealthCheckOptions
    {
        Predicate = check => check.Tags.Contains("ready"),
        ResponseWriter = WriteResponse
    });
    
    app.UseHealthChecks("/health/live", new HealthCheckOptions
    {
        Predicate = check => check.Tags.Contains("live"),
        ResponseWriter = WriteResponse
    });
    
    app.UseHealthChecksUI(options =>
    {
        options.UIPath = "/health-ui";
    });
    
    // ...
}

private static Task WriteResponse(HttpContext context, HealthReport result)
{
    context.Response.ContentType = "application/json";
    
    var response = new
    {
        status = result.Status.ToString(),
        results = result.Entries.Select(e => new
        {
            component = e.Key,
            status = e.Value.Status.ToString(),
            description = e.Value.Description
        })
    };
    
    return context.Response.WriteAsync(JsonSerializer.Serialize(response));
}
```

Dockerfile and Docker Compose configuration for health checks:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost/health/live || exit 1
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  api:
    build: 
      context: .
      dockerfile: Dockerfile
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health/live"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s
```

### Graceful Shutdown

Handling application shutdown properly:

```csharp
// Program.cs
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        })
        .UseConsoleLifetime(options =>
        {
            // Handle shutdown better
            options.SuppressStatusMessages = true;
        })
        .ConfigureServices(services =>
        {
            // Register a hosted service for graceful shutdown
            services.AddHostedService<GracefulShutdownService>();
        });

// GracefulShutdownService.cs
public class GracefulShutdownService : IHostedService
{
    private readonly ILogger<GracefulShutdownService> _logger;
    private readonly IHostApplicationLifetime _appLifetime;
    
    public GracefulShutdownService(
        ILogger<GracefulShutdownService> logger,
        IHostApplicationLifetime appLifetime)
    {
        _logger = logger;
        _appLifetime = appLifetime;
    }
    
    public Task StartAsync(CancellationToken cancellationToken)
    {
        _appLifetime.ApplicationStarted.Register(OnStarted);
        _appLifetime.ApplicationStopping.Register(OnStopping);
        _appLifetime.ApplicationStopped.Register(OnStopped);
        
        return Task.CompletedTask;
    }
    
    public Task StopAsync(CancellationToken cancellationToken)
    {
        return Task.CompletedTask;
    }
    
    private void OnStarted()
    {
        _logger.LogInformation("Application started");
    }
    
    private void OnStopping()
    {
        _logger.LogInformation("Application is shutting down...");
        // Give time for in-flight requests to complete
        Thread.Sleep(5000);
    }
    
    private void OnStopped()
    {
        _logger.LogInformation("Application stopped");
    }
}
```

## Docker Compose for .NET Applications

### Basic Compose File

A simple Docker Compose file for a .NET application with a database:

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: src/MyApp.API/Dockerfile
    ports:
      - "8080:80"
      - "8081:443"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=https://+:443;http://+:80
      - ASPNETCORE_Kestrel__Certificates__Default__Password=password
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/https/aspnetapp.pfx
      - ConnectionStrings__DefaultConnection=Server=db;Database=MyAppDb;User=sa;Password=YourStrong!Password;
    depends_on:
      - db
    volumes:
      - ~/.aspnet/https:/https:ro
    networks:
      - myapp-network

  db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong!Password
    ports:
      - "1433:1433"
    volumes:
      - sqldata:/var/opt/mssql
    networks:
      - myapp-network

networks:
  myapp-network:
    driver: bridge

volumes:
  sqldata:
```

### Multi-service Application

A more complex Docker Compose for a microservices application:

```yaml
version: '3.8'

services:
  api-gateway:
    build:
      context: .
      dockerfile: src/ApiGateway/Dockerfile
    ports:
      - "8000:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    depends_on:
      - orders-api
      - catalog-api
      - identity-api
    networks:
      - frontend
      - backend

  orders-api:
    build:
      context: .
      dockerfile: src/OrdersApi/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__OrdersDb=Server=orders-db;Database=OrdersDb;User=sa;Password=YourStrong!Password;
      - MessageBroker__Host=rabbitmq
    depends_on:
      - orders-db
      - rabbitmq
    networks:
      - backend

  catalog-api:
    build:
      context: .
      dockerfile: src/CatalogApi/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__CatalogDb=Server=catalog-db;Database=CatalogDb;User=sa;Password=YourStrong!Password;
      - MessageBroker__Host=rabbitmq
    depends_on:
      - catalog-db
      - rabbitmq
    networks:
      - backend

  identity-api:
    build:
      context: .
      dockerfile: src/IdentityApi/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__IdentityDb=Server=identity-db;Database=IdentityDb;User=sa;Password=YourStrong!Password;
    depends_on:
      - identity-db
    networks:
      - backend

  orders-db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong!Password
    volumes:
      - orders-data:/var/opt/mssql
    networks:
      - backend

  catalog-db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong!Password
    volumes:
      - catalog-data:/var/opt/mssql
    networks:
      - backend

  identity-db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong!Password
    volumes:
      - identity-data:/var/opt/mssql
    networks:
      - backend

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"  # Management UI
    networks:
      - backend

  seq:
    image: datalust/seq:latest
    ports:
      - "5341:80"
    environment:
      - ACCEPT_EULA=Y
    networks:
      - backend

networks:
  frontend:
  backend:

volumes:
  orders-data:
  catalog-data:
  identity-data:
```

### Development vs. Production

Using Docker Compose for different environments:

#### Development Compose File (docker-compose.yml)

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: src/MyApp.API/Dockerfile
      target: development
    ports:
      - "8080:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=MyAppDb;User=sa;Password=YourStrong!Password;
    volumes:
      - ./src:/src
    networks:
      - myapp-network

  db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong!Password
    ports:
      - "1433:1433"
    networks:
      - myapp-network

networks:
  myapp-network:
```

#### Production Compose File (docker-compose.prod.yml)

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: src/MyApp.API/Dockerfile
      target: production
    ports:
      - "80:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=Server=db;Database=MyAppDb;User=sa;Password=${DB_PASSWORD}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
    networks:
      - myapp-network

  db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=${DB_PASSWORD}
    volumes:
      - sqldata:/var/opt/mssql
    deploy:
      placement:
        constraints:
          - node.role == manager
    networks:
      - myapp-network

networks:
  myapp-network:

volumes:
  sqldata:
```

Running with different configurations:

```bash
# Development
docker-compose up -d

# Production
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## Advanced Docker Patterns

### Sidecar Pattern

Using containers to extend functionality:

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: src/MyApp.API/Dockerfile
    volumes:
      - api-logs:/app/logs
    networks:
      - myapp-network

  log-exporter:
    image: log-exporter:latest
    volumes:
      - api-logs:/logs:ro
    environment:
      - TARGET_STORAGE=azure
      - STORAGE_ACCOUNT=mylogstorage
      - STORAGE_KEY=${STORAGE_KEY}
    networks:
      - myapp-network

volumes:
  api-logs:
```

### Ambassador Pattern

Using a proxy container for service access:

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: src/MyApp.API/Dockerfile
    environment:
      - DB_HOST=db-ambassador
      - DB_PORT=5432
    networks:
      - myapp-network

  db-ambassador:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    networks:
      - myapp-network
      - db-network

  db:
    image: postgres:13
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    networks:
      - db-network

networks:
  myapp-network:
  db-network:
```

### Init Container Pattern

Using init containers for setup tasks:

```yaml
# docker-compose.yml
version: '3.8'

services:
  db-init:
    build:
      context: ./db-scripts
      dockerfile: Dockerfile
    environment:
      - DB_HOST=db
      - DB_PORT=1433
      - DB_USER=sa
      - DB_PASSWORD=YourStrong!Password
    depends_on:
      - db
    networks:
      - myapp-network

  api:
    build:
      context: .
      dockerfile: src/MyApp.API/Dockerfile
    depends_on:
      - db-init
    networks:
      - myapp-network

  db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong!Password
    networks:
      - myapp-network

networks:
  myapp-network:
```

## CI/CD for Containerized .NET Applications

### Azure DevOps Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
    - main
    - feature/*

variables:
  dockerRegistry: 'myregistry.azurecr.io'
  imageName: 'myapp'
  tag: '$(Build.BuildNumber)'

stages:
- stage: Build
  jobs:
  - job: BuildAndPush
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: Docker@2
      displayName: 'Build and push Docker image'
      inputs:
        containerRegistry: 'MyDockerRegistry'
        repository: '$(dockerRegistry)/$(imageName)'
        command: 'buildAndPush'
        Dockerfile: '**/Dockerfile'
        buildContext: '$(Build.SourcesDirectory)'
        tags: |
          $(tag)
          latest

- stage: Deploy
  dependsOn: Build
  jobs:
  - deployment: DeployToAKS
    environment: 'Production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: 'Deploy to Kubernetes'
            inputs:
              action: 'deploy'
              kubernetesServiceConnection: 'MyAKSCluster'
              namespace: 'production'
              manifests: |
                $(Pipeline.Workspace)/manifests/deployment.yml
                $(Pipeline.Workspace)/manifests/service.yml
              containers: '$(dockerRegistry)/$(imageName):$(tag)'
```

### GitHub Actions Workflow

```yaml
# .github/workflows/docker-publish.yml
name: Docker Build and Publish

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
      
    - name: Login to Container Registry
      uses: docker/login-action@v2
      with:
        registry: myregistry.azurecr.io
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
        
    - name: Build and push Docker image
      uses: docker/build-push-action@v3
      with:
        context: .
        push: true
        tags: myregistry.azurecr.io/myapp:latest,myregistry.azurecr.io/myapp:${{ github.sha }}
        cache-from: type=registry,ref=myregistry.azurecr.io/myapp:latest
        cache-to: type=inline
        
    - name: Deploy to Azure App Service
      uses: azure/webapps-deploy@v2
      with:
        app-name: 'my-container-app'
        slot-name: 'production'
        images: 'myregistry.azurecr.io/myapp:${{ github.sha }}'
```

## Monitoring Containerized .NET Applications

### Application Insights

Setting up Application Insights in a Docker container:

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    // Add Application Insights telemetry
    services.AddApplicationInsightsTelemetry();
    
    // Configure the TelemetryConfiguration
    services.AddSingleton<ITelemetryInitializer, CloudRoleNameInitializer>();
}

// CloudRoleNameInitializer.cs
public class CloudRoleNameInitializer : ITelemetryInitializer
{
    private readonly string _roleName;
    
    public CloudRoleNameInitializer(IConfiguration configuration)
    {
        _roleName = configuration["ApplicationInsights:RoleName"] ?? "MyApp";
    }
    
    public void Initialize(ITelemetry telemetry)
    {
        telemetry.Context.Cloud.RoleName = _roleName;
        telemetry.Context.Cloud.RoleInstance = Environment.MachineName;
    }
}
```

Dockerfile with Application Insights:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .

ENV APPLICATIONINSIGHTS_CONNECTION_STRING="InstrumentationKey=your-key;IngestionEndpoint=https://regionname.in.applicationinsights.azure.com/"
ENV ApplicationInsights__RoleName="MyDockerApp"

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Prometheus and Grafana

Monitoring .NET applications with Prometheus and Grafana:

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddHealthChecks()
        .AddDbContextCheck<ApplicationDbContext>();
        
    // Add Prometheus metrics
    services.AddOpenTelemetryMetrics(builder =>
    {
        builder.AddAspNetCoreInstrumentation();
        builder.AddHttpClientInstrumentation();
        builder.AddPrometheusExporter();
    });
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // ... other middleware
    
    // Expose Prometheus metrics endpoint
    app.UseOpenTelemetryPrometheusScrapingEndpoint();
    
    // ... other middleware
}
```

Docker Compose configuration:

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:80"
    networks:
      - monitoring-network

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    networks:
      - monitoring-network

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - monitoring-network

networks:
  monitoring-network:

volumes:
  prometheus-data:
  grafana-data:
```

Prometheus configuration file (prometheus.yml):

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'netcore-app'
    metrics_path: /metrics
    static_configs:
      - targets: ['api:80']
```

## Security Best Practices

### Secure Docker Images

Securing Docker images for .NET applications:

```dockerfile
# Start with the latest security updates
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS final

# Set the working directory
WORKDIR /app

# Copy application files
COPY --from=publish /app/publish .

# Create non-root user
RUN adduser --disabled-password --gecos "" appuser
USER appuser

# Define health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost/health || exit 1

# Define environment variables
ENV ASPNETCORE_URLS=http://+:80
ENV DOTNET_RUNNING_IN_CONTAINER=true

# Limit container resources
# docker run --cpus=1.5 --memory=2g myapp

# Expose port
EXPOSE 80

# Run the application
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Scanning Docker Images

Tools and commands for security scanning:

```bash
# Use Trivy for vulnerability scanning
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image myapp:latest

# Use Docker Scan (built into Docker)
docker scan myapp:latest

# Use Snyk
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock snyk/snyk:docker snyk test --docker myapp:latest
```

### Secret Management

Secure handling of secrets in Docker:

```yaml
# docker-compose.yml with Docker secrets
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
    secrets:
      - db_connection
      - api_key
    networks:
      - myapp-network

secrets:
  db_connection:
    file: ./secrets/db_connection.txt
  api_key:
    file: ./secrets/api_key.txt

networks:
  myapp-network:
```

Using secrets in ASP.NET Core:

```csharp
// Program.cs
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureAppConfiguration((hostingContext, config) =>
        {
            // Add Docker secrets
            if (Directory.Exists("/run/secrets"))
            {
                foreach (var file in Directory.GetFiles("/run/secrets"))
                {
                    string name = Path.GetFileName(file);
                    
                    if (name.Contains("__"))
                    {
                        // Handle hierarchical configuration
                        string[] parts = name.Split("__");
                        string key = string.Join(":", parts);
                        
                        config.AddKeyPerFile(
                            directoryPath: "/run/secrets", 
                            optional: true,
                            fileNameToKeyMapping: f => f == name ? key : f);
                    }
                    else
                    {
                        config.AddKeyPerFile("/run/secrets", optional: true);
                    }
                }
            }
        })
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });
```

### Running Containers with Least Privilege

Creating minimal permissions for containers:

```bash
# Run container with restricted capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp:latest

# Run container with read-only file system
docker run --read-only --tmpfs /tmp myapp:latest
```

## Common Pitfalls

### Ignoring Container Lifecycle

**Problem**: Not handling container startups and shutdowns properly.

**Solution**: Implement proper initialization and graceful shutdown handling:

```csharp
// Program.cs
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        })
        .ConfigureServices(services =>
        {
            // Add startup and shutdown coordination
            services.AddHostedService<AppInitializationService>();
        });

// AppInitializationService.cs
public class AppInitializationService : IHostedService
{
    private readonly ILogger<AppInitializationService> _logger;
    private readonly IHostApplicationLifetime _appLifetime;
    private readonly ApplicationDbContext _dbContext;
    
    public AppInitializationService(
        ILogger<AppInitializationService> logger,
        IHostApplicationLifetime appLifetime,
        ApplicationDbContext dbContext)
    {
        _logger = logger;
        _appLifetime = appLifetime;
        _dbContext = dbContext;
    }
    
    public async Task StartAsync(CancellationToken cancellationToken)
    {
        try
        {
            // Perform database migrations
            _logger.LogInformation("Applying database migrations...");
            await _dbContext.Database.MigrateAsync(cancellationToken);
            
            // Register shutdown handlers
            _appLifetime.ApplicationStopping.Register(OnStopping);
            
            _logger.LogInformation("Application initialized successfully");
        }
        catch (Exception ex)
        {
            _logger.LogCritical(ex, "Failed to initialize application");
            _appLifetime.StopApplication();
        }
    }
    
    public Task StopAsync(CancellationToken cancellationToken)
    {
        return Task.CompletedTask;
    }
    
    private void OnStopping()
    {
        _logger.LogInformation("Application is shutting down...");
        // Perform cleanup operations
        Thread.Sleep(5000); // Give time for in-flight requests
    }
}
```

### Ignoring Container Storage Limitations

**Problem**: Treating containers as persistent storage.

**Solution**: Use proper storage solutions for container data:

```yaml
# docker-compose.yml with proper volume management
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - tempdata:/app/temp
    networks:
      - myapp-network

  db:
    image: postgres:latest
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - myapp-network

volumes:
  tempdata:
    driver: local
  pgdata:
    driver: local

networks:
  myapp-network:
```

### Excessive Memory Usage

**Problem**: .NET applications using too much memory in containers.

**Solution**: Configure .NET runtime for containers:

```dockerfile
# Set memory limits in Dockerfile
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=true
ENV DOTNET_GCHeapHardLimit=800000000
ENV COMPlus_GCHeapHardLimit=800000000
```

ASP.NET Core configuration:

```csharp
// Program.cs
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
            webBuilder.ConfigureKestrel(options =>
            {
                // Limit concurrent connections
                options.Limits.MaxConcurrentConnections = 100;
                options.Limits.MaxRequestBodySize = 10 * 1024 * 1024; // 10 MB
                options.Limits.MaxRequestHeadersTotalSize = 32 * 1024; // 32 KB
            });
        });
```

### Overcomplicating Multi-stage Builds

**Problem**: Creating overly complex multi-stage builds.

**Solution**: Keep Dockerfiles simple and maintainable:

```dockerfile
# Simple yet effective multi-stage build
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src

# Copy csproj and restore dependencies
COPY *.csproj ./
RUN dotnet restore

# Copy everything and build
COPY . .
RUN dotnet publish -c Release -o /app/publish

# Create final image
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

## Further Reading

- [.NET Docker Documentation](https://docs.microsoft.com/en-us/dotnet/core/docker/introduction)
- [Docker for .NET Developers](https://docs.docker.com/language/dotnet/)
- [Microservices with .NET and Docker](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/container-docker-introduction/)
- [Docker Security Best Practices](https://docs.docker.com/develop/security-best-practices/)
- [Optimizing ASP.NET Core for Docker](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/docker/building-net-docker-images)

## Related Topics

- [Kubernetes for .NET Applications](../kubernetes/kubernetes-basics.md)
- [Microservices Architecture](../../architecture/microservices/introduction.md)
- [CI/CD Pipelines for Containerized Applications](../../devops/cicd-pipelines.md)
- [Azure Container Apps](../../cloud/azure/container-apps.md)
- [AWS Elastic Container Service (ECS)](../../cloud/aws/ecs.md)