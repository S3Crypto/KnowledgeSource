---
title: "Microservices Architecture Principles and Patterns"
date_created: 2023-04-02
date_updated: 2023-04-02
authors: ["Repository Maintainers"]
tags: ["microservices", "architecture", "distributed-systems", "patterns", "principles"]
difficulty: "advanced"
---

# Microservices Architecture Principles and Patterns

## Overview

Microservices architecture is an approach to building software systems as a collection of small, independently deployable services, each responsible for specific business capabilities. This document explores the core principles, patterns, and practices that comprise microservices architecture, with a focus on implementation in .NET environments. Understanding these principles is crucial for designing scalable, resilient, and maintainable microservice-based systems.

## Core Principles

### Service Autonomy

The foundation of microservices architecture is autonomous services that can be developed, deployed, and scaled independently:

- **Independent Codebase**: Each service has its own source code repository
- **Separate Deployability**: Services can be deployed without affecting others
- **Technology Independence**: Services can use different technology stacks
- **Team Ownership**: Services are typically owned by specific teams

```csharp
// Each service has its own Program.cs entry point
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

### Bounded Contexts

Inspired by Domain-Driven Design (DDD), microservices should align with bounded contexts:

- **Business Domain Alignment**: Services are organized around business capabilities
- **Clear Boundaries**: Well-defined interfaces between services
- **Domain Language**: Each service uses the language of its domain
- **Encapsulated Data**: Services own their data and expose it only through APIs

```csharp
// Example of a service aligned with the "Order" bounded context
namespace OrderService
{
    public class Order
    {
        public int Id { get; set; }
        public string CustomerReference { get; set; } // Reference to a Customer in another bounded context
        public List<OrderLine> OrderLines { get; set; }
        public OrderStatus Status { get; set; }
        public DateTime CreatedAt { get; set; }
        
        // Business logic specific to the Order bounded context
        public decimal CalculateTotal()
        {
            return OrderLines.Sum(line => line.Quantity * line.UnitPrice);
        }
        
        public bool CanBeCancelled()
        {
            return Status == OrderStatus.Created || Status == OrderStatus.Processing;
        }
    }
}
```

### Single Responsibility

Each microservice should have a single responsibility:

- **Focused Functionality**: Service handles one business capability
- **High Cohesion**: Related functionality grouped together
- **Low Coupling**: Minimal dependencies between services
- **Right-Sizing**: Neither too large (monolith) nor too small (nanoservice)

```csharp
// Order service focuses only on order management
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderRepository _repository;
    private readonly IEventBus _eventBus;
    
    // Constructor with injected dependencies
    
    [HttpPost]
    public async Task<ActionResult<Order>> CreateOrder(OrderCreateDto orderDto)
    {
        var order = new Order
        {
            CustomerReference = orderDto.CustomerId,
            OrderLines = orderDto.Items.Select(i => new OrderLine {...}).ToList(),
            Status = OrderStatus.Created,
            CreatedAt = DateTime.UtcNow
        };
        
        await _repository.AddAsync(order);
        
        // Publish event for other services to react to
        await _eventBus.PublishAsync(new OrderCreatedEvent(order.Id, order.CustomerReference));
        
        return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
    }
    
    // Other order-specific endpoints
}
```

### Resilience and Fault Isolation

Microservices should be designed to handle failures gracefully:

- **Fault Isolation**: Failures in one service don't cascade to others
- **Circuit Breaking**: Prevent cascading failures by failing fast
- **Retry Logic**: Automatically retry transient failures
- **Fallback Mechanisms**: Provide alternatives when dependencies fail
- **Health Monitoring**: Active monitoring of service health

```csharp
// Example using Polly for resilience patterns
public class CustomerApiClient
{
    private readonly HttpClient _httpClient;
    private readonly IAsyncPolicy<HttpResponseMessage> _retryPolicy;
    private readonly IAsyncPolicy<HttpResponseMessage> _circuitBreakerPolicy;
    private readonly ILogger<CustomerApiClient> _logger;
    
    public CustomerApiClient(HttpClient httpClient, ILogger<CustomerApiClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
        
        // Define retry policy
        _retryPolicy = Policy
            .HandleTransientHttpError()
            .WaitAndRetryAsync(3, retryAttempt => 
                TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                onRetry: (outcome, timespan, retryCount, context) =>
                {
                    _logger.LogWarning("Retrying request after {Delay}ms, attempt {RetryCount}", 
                        timespan.TotalMilliseconds, retryCount);
                });
                
        // Define circuit breaker policy
        _circuitBreakerPolicy = Policy
            .HandleTransientHttpError()
            .CircuitBreakerAsync(5, TimeSpan.FromMinutes(1),
                onBreak: (outcome, timespan) =>
                {
                    _logger.LogWarning("Circuit broken for {DurationSeconds}s", timespan.TotalSeconds);
                },
                onReset: () =>
                {
                    _logger.LogInformation("Circuit reset");
                });
    }
    
    public async Task<CustomerDto> GetCustomerAsync(string customerId)
    {
        try
        {
            // Combine policies and execute
            var response = await _retryPolicy
                .WrapAsync(_circuitBreakerPolicy)
                .ExecuteAsync(() => _httpClient.GetAsync($"api/customers/{customerId}"));
                
            response.EnsureSuccessStatusCode();
            
            return await JsonSerializer.DeserializeAsync<CustomerDto>(
                await response.Content.ReadAsStreamAsync());
        }
        catch (BrokenCircuitException)
        {
            _logger.LogError("Circuit broken, returning fallback customer info");
            return GetFallbackCustomer(customerId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error retrieving customer {CustomerId}", customerId);
            throw;
        }
    }
    
    private CustomerDto GetFallbackCustomer(string customerId)
    {
        // Return minimal customer information
        return new CustomerDto
        {
            Id = customerId,
            Name = "Unknown Customer",
            IsFallback = true
        };
    }
}
```

### Decentralized Data Management

Each service manages its own data:

- **Private Databases**: Services have their own database
- **Data Duplication**: Accept some data duplication across services
- **Eventual Consistency**: Maintain consistency through events
- **Polyglot Persistence**: Use appropriate database technology for each service

```csharp
// Each service has its own DbContext
public class OrderDbContext : DbContext
{
    public OrderDbContext(DbContextOptions<OrderDbContext> options)
        : base(options)
    { }
    
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderLine> OrderLines { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>()
            .HasMany(o => o.OrderLines)
            .WithOne()
            .HasForeignKey(ol => ol.OrderId);
            
        // No references to Customer table - that belongs to another service
        // CustomerReference is just a string identifier
        modelBuilder.Entity<Order>()
            .Property(o => o.CustomerReference)
            .IsRequired();
    }
}

// Customer service has a completely separate database context
public class CustomerDbContext : DbContext
{
    public CustomerDbContext(DbContextOptions<CustomerDbContext> options)
        : base(options)
    { }
    
    public DbSet<Customer> Customers { get; set; }
    public DbSet<CustomerAddress> Addresses { get; set; }
    
    // No references to Order
}
```

### API-First Design

Microservices communicate through well-defined APIs:

- **Contract-First Development**: Define API contract before implementation
- **API Versioning**: Support multiple API versions
- **Backward Compatibility**: Maintain compatibility with clients
- **Clear Documentation**: Document APIs for consumers

```csharp
// API contract definition with OpenAPI/Swagger annotations
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
[ApiVersion("1.0")]
public class ProductsController : ControllerBase
{
    /// <summary>
    /// Gets a specific product by ID
    /// </summary>
    /// <param name="id">The product identifier</param>
    /// <returns>The requested product</returns>
    /// <response code="200">Returns the product</response>
    /// <response code="404">If product not found</response>
    [HttpGet("{id}")]
    [ProducesResponseType(StatusCodes.Status200OK, Type = typeof(ProductDto))]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductDto>> GetProduct(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        
        if (product == null)
            return NotFound();
            
        return Ok(product);
    }
    
    // Other API endpoints
}
```

### Design for Failure

Microservices must anticipate and handle failures:

- **Fallback Strategies**: Define behavior when dependencies fail
- **Timeouts**: Don't wait indefinitely for responses
- **Circuit Breakers**: Stop calling failing services
- **Bulkheads**: Isolate resources to prevent resource exhaustion
- **Graceful Degradation**: Provide reduced functionality when dependencies fail

```csharp
// Service registration with HTTP client using Polly for resilience
public void ConfigureServices(IServiceCollection services)
{
    // Register resilient HTTP client
    services.AddHttpClient<IInventoryApiClient, InventoryApiClient>(client =>
    {
        client.BaseAddress = new Uri(Configuration["Services:Inventory"]);
        client.Timeout = TimeSpan.FromSeconds(5); // Set timeout
    })
    .AddPolicyHandler(GetRetryPolicy())
    .AddPolicyHandler(GetCircuitBreakerPolicy())
    .AddPolicyHandler(GetBulkheadPolicy());
}

private IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(3, retryAttempt => 
            TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
}

private IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(5, TimeSpan.FromMinutes(1));
}

private IAsyncPolicy<HttpResponseMessage> GetBulkheadPolicy()
{
    return Policy.BulkheadAsync<HttpResponseMessage>(10, 100);
}
```

## Communication Patterns

### Synchronous Communication

Direct service-to-service communication:

- **REST APIs**: HTTP-based APIs for synchronous requests
- **gRPC**: Efficient binary protocol for service-to-service communication
- **GraphQL**: Query language for flexible data fetching
- **Direct Calls**: Immediate responses to requests

```csharp
// Synchronous REST communication between services
public class CheckoutService
{
    private readonly HttpClient _httpClient;
    
    public CheckoutService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }
    
    public async Task<OrderDto> CreateOrderAsync(OrderRequest request)
    {
        // Call Order service synchronously
        var response = await _httpClient.PostAsJsonAsync(
            "api/orders", request);
            
        response.EnsureSuccessStatusCode();
        
        return await response.Content.ReadFromJsonAsync<OrderDto>();
    }
}

// gRPC service definition
syntax = "proto3";

service OrderService {
    rpc CreateOrder(OrderRequest) returns (OrderResponse);
    rpc GetOrder(GetOrderRequest) returns (OrderResponse);
}

message OrderRequest {
    string customer_id = 1;
    repeated OrderItem items = 2;
}

message OrderItem {
    string product_id = 1;
    int32 quantity = 2;
}

message OrderResponse {
    string order_id = 1;
    string status = 2;
    double total_amount = 3;
}

message GetOrderRequest {
    string order_id = 1;
}
```

### Asynchronous Communication

Event-based communication between services:

- **Message Queues**: Reliable delivery of commands/events
- **Event Sourcing**: Store all changes as events
- **Event-Driven Architecture**: React to events from other services
- **Publish-Subscribe**: Services publish and subscribe to events

```csharp
// Defining events
public class OrderCreatedEvent : IntegrationEvent
{
    public int OrderId { get; }
    public string CustomerId { get; }
    public decimal TotalAmount { get; }
    public DateTime CreatedAt { get; }
    
    public OrderCreatedEvent(int orderId, string customerId, decimal totalAmount)
    {
        OrderId = orderId;
        CustomerId = customerId;
        TotalAmount = totalAmount;
        CreatedAt = DateTime.UtcNow;
    }
}

// Publishing events with MassTransit and RabbitMQ
public class OrderService
{
    private readonly IPublishEndpoint _publishEndpoint;
    private readonly IOrderRepository _orderRepository;
    
    public OrderService(
        IPublishEndpoint publishEndpoint,
        IOrderRepository orderRepository)
    {
        _publishEndpoint = publishEndpoint;
        _orderRepository = orderRepository;
    }
    
    public async Task<Order> CreateOrderAsync(OrderCreateDto dto)
    {
        var order = new Order
        {
            CustomerId = dto.CustomerId,
            Items = dto.Items.Select(i => new OrderItem {...}).ToList(),
            Status = OrderStatus.Created,
            CreatedAt = DateTime.UtcNow
        };
        
        await _orderRepository.AddAsync(order);
        
        // Publish event asynchronously
        await _publishEndpoint.Publish(new OrderCreatedEvent(
            order.Id, 
            order.CustomerId, 
            order.CalculateTotal()));
            
        return order;
    }
}

// Consuming events
public class OrderCreatedConsumer : IConsumer<OrderCreatedEvent>
{
    private readonly ILogger<OrderCreatedConsumer> _logger;
    private readonly INotificationService _notificationService;
    
    public OrderCreatedConsumer(
        ILogger<OrderCreatedConsumer> logger,
        INotificationService notificationService)
    {
        _logger = logger;
        _notificationService = notificationService;
    }
    
    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        var @event = context.Message;
        
        _logger.LogInformation(
            "Received OrderCreatedEvent for Order {OrderId}", @event.OrderId);
            
        await _notificationService.SendOrderConfirmationAsync(
            @event.CustomerId, 
            @event.OrderId,
            @event.TotalAmount);
    }
}

// Configure MassTransit in Startup
public void ConfigureServices(IServiceCollection services)
{
    services.AddMassTransit(x =>
    {
        // Register consumers
        x.AddConsumer<OrderCreatedConsumer>();
        
        x.UsingRabbitMq((context, cfg) =>
        {
            cfg.Host(Configuration["RabbitMQ:Host"], "/", h =>
            {
                h.Username(Configuration["RabbitMQ:Username"]);
                h.Password(Configuration["RabbitMQ:Password"]);
            });
            
            // Configure endpoints
            cfg.ConfigureEndpoints(context);
        });
    });
}
```

### API Gateway Pattern

Providing a unified API for clients:

- **Single Entry Point**: Clients communicate through gateway
- **Request Routing**: Route requests to appropriate services
- **Aggregation**: Combine responses from multiple services
- **Protocol Translation**: Convert between different protocols
- **Auth Handling**: Centralized authentication/authorization

```csharp
// Using Ocelot as API Gateway
// ocelot.json configuration
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/products/{id}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "product-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/v1/products/{id}",
      "UpstreamHttpMethod": [ "Get" ]
    },
    {
      "DownstreamPathTemplate": "/api/orders",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "order-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/v1/orders",
      "UpstreamHttpMethod": [ "Post" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer",
        "AllowedScopes": [ "orders.write" ]
      }
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://api.mycompany.com"
  }
}

// Program.cs in API Gateway
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            })
            .ConfigureAppConfiguration((hostingContext, config) =>
            {
                config
                    .SetBasePath(hostingContext.HostingEnvironment.ContentRootPath)
                    .AddJsonFile("appsettings.json", true, true)
                    .AddJsonFile($"appsettings.{hostingContext.HostingEnvironment.EnvironmentName}.json", true, true)
                    .AddJsonFile("ocelot.json")
                    .AddEnvironmentVariables();
            });
}

// Startup.cs in API Gateway
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options =>
            {
                // JWT configuration
            });
            
        services.AddOcelot();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
        
        app.UseOcelot().Wait();
    }
}
```

### Backends for Frontends (BFF)

Creating specialized gateways for different clients:

- **Client-Specific APIs**: Tailor APIs to client needs
- **Aggregation**: Combine data for specific UI requirements
- **Team Ownership**: Frontend teams own their BFF
- **Client Optimization**: Optimize responses for client types

```csharp
// Mobile BFF controller
[ApiController]
[Route("api/mobile/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly ICategoryService _categoryService;
    
    public ProductsController(
        IProductService productService,
        ICategoryService categoryService)
    {
        _productService = productService;
        _categoryService = categoryService;
    }
    
    // Optimized endpoint for mobile - simplified response with essential data
    [HttpGet("featured")]
    public async Task<ActionResult<List<MobileProductDto>>> GetFeaturedProducts()
    {
        var products = await _productService.GetFeaturedProductsAsync();
        
        // Return only data needed for mobile UI
        var results = products.Select(p => new MobileProductDto
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price,
            ThumbnailUrl = p.Images.FirstOrDefault()?.ThumbnailUrl,
            Rating = p.Reviews.Any() ? p.Reviews.Average(r => r.Rating) : null
        }).ToList();
        
        return results;
    }
}

// Web BFF controller
[ApiController]
[Route("api/web/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly ICategoryService _categoryService;
    private readonly IReviewService _reviewService;
    
    public ProductsController(
        IProductService productService,
        ICategoryService categoryService,
        IReviewService reviewService)
    {
        _productService = productService;
        _categoryService = categoryService;
        _reviewService = reviewService;
    }
    
    // Comprehensive endpoint for web - detailed response with more data
    [HttpGet("featured")]
    public async Task<ActionResult<WebProductsViewModel>> GetFeaturedProducts()
    {
        // Parallel requests to multiple services
        var productsTask = _productService.GetFeaturedProductsAsync();
        var categoriesTask = _categoryService.GetCategoriesAsync();
        
        await Task.WhenAll(productsTask, categoriesTask);
        
        var products = await productsTask;
        var categories = await categoriesTask;
        
        // Combine data for web UI
        return new WebProductsViewModel
        {
            Products = products.Select(p => new WebProductDto
            {
                Id = p.Id,
                Name = p.Name,
                Description = p.Description,
                Price = p.Price,
                Discount = p.Discount,
                Images = p.Images.Select(i => i.Url).ToList(),
                CategoryId = p.CategoryId,
                CategoryName = categories.FirstOrDefault(c => c.Id == p.CategoryId)?.Name,
                Specifications = p.Specifications
            }).ToList(),
            Categories = categories.Select(c => new CategoryDto
            {
                Id = c.Id,
                Name = c.Name,
                ProductCount = products.Count(p => p.CategoryId == c.Id)
            }).ToList()
        };
    }
}
```

## Implementation Patterns

### Database Per Service

Each service has its own dedicated database:

- **Data Autonomy**: Services control their data schema
- **Technology Choice**: Select appropriate database for service needs
- **Independent Scaling**: Scale database with service needs
- **Security Isolation**: Data access restricted to owning service

```csharp
// Product service using SQL Server
public class ProductDbContext : DbContext
{
    public ProductDbContext(DbContextOptions<ProductDbContext> options)
        : base(options)
    { }
    
    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }
    public DbSet<ProductImage> ProductImages { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Product-specific schema configuration
    }
}

// Configuration in Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<ProductDbContext>(options =>
        options.UseSqlServer(Configuration.GetConnectionString("ProductDatabase")));
}

// Order service using MongoDB
public class OrderContext
{
    private readonly IMongoDatabase _database;
    
    public OrderContext(IOptions<OrderDatabaseSettings> settings)
    {
        var client = new MongoClient(settings.Value.ConnectionString);
        _database = client.GetDatabase(settings.Value.DatabaseName);
    }
    
    public IMongoCollection<Order> Orders => 
        _database.GetCollection<Order>("Orders");
        
    public IMongoCollection<OrderItem> OrderItems => 
        _database.GetCollection<OrderItem>("OrderItems");
}

// Configuration in Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.Configure<OrderDatabaseSettings>(
        Configuration.GetSection(nameof(OrderDatabaseSettings)));
        
    services.AddSingleton<OrderContext>();
}
```

### CQRS (Command Query Responsibility Segregation)

Separating read and write operations:

- **Commands**: Change state (write operations)
- **Queries**: Return data (read operations)
- **Separation of Concerns**: Different models for reads and writes
- **Optimized Models**: Tailor data models to use cases
- **Different Stores**: Use separate data stores for reads and writes

```csharp
// Command handler for creating a product
public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, int>
{
    private readonly IProductWriteRepository _repository;
    private readonly IEventBus _eventBus;
    
    public CreateProductCommandHandler(
        IProductWriteRepository repository,
        IEventBus eventBus)
    {
        _repository = repository;
        _eventBus = eventBus;
    }
    
    public async Task<int> Handle(CreateProductCommand command, CancellationToken cancellationToken)
    {
        var product = new Product
        {
            Name = command.Name,
            Description = command.Description,
            Price = command.Price,
            CategoryId = command.CategoryId,
            CreatedAt = DateTime.UtcNow
        };
        
        // Write to write database
        var productId = await _repository.AddAsync(product, cancellationToken);
        
        // Publish event to update read models
        await _eventBus.PublishAsync(new ProductCreatedEvent
        {
            Id = productId,
            Name = product.Name,
            Price = product.Price,
            CategoryId = product.CategoryId
        });
        
        return productId;
    }
}

// Query handler for retrieving products
public class GetProductsQueryHandler : IRequestHandler<GetProductsQuery, List<ProductReadDto>>
{
    private readonly IProductReadRepository _repository;
    
    public GetProductsQueryHandler(IProductReadRepository repository)
    {
        _repository = repository;
    }
    
    public async Task<List<ProductReadDto>> Handle(GetProductsQuery query, CancellationToken cancellationToken)
    {
        // Read from optimized read database/model
        var products = await _repository.GetProductsAsync(
            query.CategoryId,
            query.MinPrice,
            query.MaxPrice,
            query.SortBy,
            query.SortDirection,
            query.PageNumber,
            query.PageSize,
            cancellationToken);
            
        return products;
    }
}

// Controller using CQRS pattern
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;
    
    public ProductsController(IMediator mediator)
    {
        _mediator = mediator;
    }
    
    [HttpGet]
    public async Task<ActionResult<List<ProductReadDto>>> GetProducts(
        [FromQuery] int? categoryId,
        [FromQuery] decimal? minPrice,
        [FromQuery] decimal? maxPrice,
        [FromQuery] string sortBy = "Name",
        [FromQuery] string sortDirection = "asc",
        [FromQuery] int pageNumber = 1,
        [FromQuery] int pageSize = 10)
    {
        var query = new GetProductsQuery
        {
            CategoryId = categoryId,
            MinPrice = minPrice,
            MaxPrice = maxPrice,
            SortBy = sortBy,
            SortDirection = sortDirection,
            PageNumber = pageNumber,
            PageSize = pageSize
        };
        
        var result = await _mediator.Send(query);
        return Ok(result);
    }
    
    [HttpPost]
    public async Task<ActionResult> CreateProduct(CreateProductCommand command)
    {
        var productId = await _mediator.Send(command);
        return CreatedAtAction(nameof(GetProduct), new { id = productId }, null);
    }
    
    [HttpGet("{id}")]
    public async Task<ActionResult<ProductReadDto>> GetProduct(int id)
    {
        var query = new GetProductByIdQuery { Id = id };
        var result = await _mediator.Send(query);
        
        if (result == null)
            return NotFound();
            
        return Ok(result);
    }
}
```

### Event Sourcing

Storing state changes as a sequence of events:

- **Event Log**: Primary source of truth is the event log
- **State Reconstruction**: Build current state from events
- **Audit Trail**: Complete history of all changes
- **Temporal Queries**: Query system state at any point in time
- **Event Replay**: Rebuild projections or fix bugs by replaying events

```csharp
// Domain events
public abstract class DomainEvent
{
    public Guid Id { get; } = Guid.NewGuid();
    public DateTime Timestamp { get; } = DateTime.UtcNow;
}

public class OrderCreatedEvent : DomainEvent
{
    public Guid OrderId { get; set; }
    public string CustomerId { get; set; }
    public List<OrderItemDto> Items { get; set; }
}

public class OrderItemAddedEvent : DomainEvent
{
    public Guid OrderId { get; set; }
    public Guid OrderItemId { get; set; }
    public string ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

public class OrderStatusChangedEvent : DomainEvent
{
    public Guid OrderId { get; set; }
    public string OldStatus { get; set; }
    public string NewStatus { get; set; }
    public string Reason { get; set; }
}

// Event sourced entity
public class Order : EventSourcedAggregate
{
    public Guid Id { get; private set; }
    public string CustomerId { get; private set; }
    public List<OrderItem> Items { get; private set; } = new List<OrderItem>();
    public string Status { get; private set; }
    public decimal TotalAmount => Items.Sum(i => i.Quantity * i.UnitPrice);

    // Create a new order
    public static Order Create(Guid orderId, string customerId, List<OrderItemDto> items)
    {
        var order = new Order();
        order.ApplyChange(new OrderCreatedEvent
        {
            OrderId = orderId,
            CustomerId = customerId,
            Items = items
        });
        
        return order;
    }
    
    // Add item to order
    public void AddItem(string productId, int quantity, decimal unitPrice)
    {
        if (Status != "Created")
            throw new InvalidOperationException("Cannot add items to order in status: " + Status);
            
        var orderItemId = Guid.NewGuid();
        
        ApplyChange(new OrderItemAddedEvent
        {
            OrderId = Id,
            OrderItemId = orderItemId,
            ProductId = productId,
            Quantity = quantity,
            UnitPrice = unitPrice
        });
    }
    
    // Change order status
    public void ChangeStatus(string newStatus, string reason = null)
    {
        if (Status == newStatus)
            return;
            
        // Validate status transition
        ValidateStatusTransition(Status, newStatus);
        
        ApplyChange(new OrderStatusChangedEvent
        {
            OrderId = Id,
            OldStatus = Status,
            NewStatus = newStatus,
            Reason = reason
        });
    }
    
    // Apply events to rebuild state
    protected override void Apply(object @event)
    {
        switch (@event)
        {
            case OrderCreatedEvent e:
                Id = e.OrderId;
                CustomerId = e.CustomerId;
                Status = "Created";
                
                foreach (var item in e.Items)
                {
                    Items.Add(new OrderItem
                    {
                        Id = Guid.NewGuid(),
                        ProductId = item.ProductId,
                        Quantity = item.Quantity,
                        UnitPrice = item.UnitPrice
                    });
                }
                break;
                
            case OrderItemAddedEvent e:
                Items.Add(new OrderItem
                {
                    Id = e.OrderItemId,
                    ProductId = e.ProductId,
                    Quantity = e.Quantity,
                    UnitPrice = e.UnitPrice
                });
                break;
                
            case OrderStatusChangedEvent e:
                Status = e.NewStatus;
                break;
        }
    }
    
    private void ValidateStatusTransition(string oldStatus, string newStatus)
    {
        // Implement status transition rules
        var validTransitions = new Dictionary<string, string[]>
        {
            ["Created"] = new[] { "Processing", "Cancelled" },
            ["Processing"] = new[] { "Shipped", "Cancelled" },
            ["Shipped"] = new[] { "Delivered" },
            ["Delivered"] = new[] { "Returned", "Completed" },
            ["Cancelled"] = new string[] { }
        };
        
        if (!validTransitions.ContainsKey(oldStatus) || 
            !validTransitions[oldStatus].Contains(newStatus))
        {
            throw new InvalidOperationException(
                $"Cannot transition from {oldStatus} to {newStatus}");
        }
    }
}

// Event store
public class EventStore : IEventStore
{
    private readonly EventStoreDbContext _dbContext;
    private readonly IEventBus _eventBus;
    
    public EventStore(EventStoreDbContext dbContext, IEventBus eventBus)
    {
        _dbContext = dbContext;
        _eventBus = eventBus;
    }
    
    public async Task SaveEventsAsync(Guid aggregateId, IEnumerable<object> events, int expectedVersion)
    {
        var eventDescriptors = await _dbContext.Events
            .Where(e => e.AggregateId == aggregateId)
            .OrderBy(e => e.Version)
            .ToListAsync();
            
        // Check if expected version matches
        if (eventDescriptors.Any() && eventDescriptors.Last().Version != expectedVersion)
        {
            throw new ConcurrencyException(aggregateId);
        }
        
        var version = expectedVersion;
        
        // Save new events
        foreach (var @event in events)
        {
            version++;
            
            var eventType = @event.GetType().Name;
            var eventData = JsonSerializer.Serialize(@event);
            
            var eventDescriptor = new EventDescriptor
            {
                AggregateId = aggregateId,
                Version = version,
                Timestamp = DateTime.UtcNow,
                EventType = eventType,
                EventData = eventData
            };
            
            await _dbContext.Events.AddAsync(eventDescriptor);
            
            // Publish event to update read models
            await _eventBus.PublishAsync(@event);
        }
        
        await _dbContext.SaveChangesAsync();
    }
    
    public async Task<List<object>> GetEventsAsync(Guid aggregateId)
    {
        var eventDescriptors = await _dbContext.Events
            .Where(e => e.AggregateId == aggregateId)
            .OrderBy(e => e.Version)
            .ToListAsync();
            
        var events = new List<object>();
        
        foreach (var descriptor in eventDescriptors)
        {
            var eventType = Type.GetType($"MyApp.Domain.Events.{descriptor.EventType}");
            var @event = JsonSerializer.Deserialize(descriptor.EventData, eventType);
            events.Add(@event);
        }
        
        return events;
    }
}
```

### Saga / Process Manager

Coordinating transactions across services:

- **Distributed Transactions**: Coordinate operations across services
- **Compensating Actions**: Roll back partial transactions
- **Event Choreography**: Services react to each other's events
- **Process Orchestration**: Central coordinator manages process flow

```csharp
// Order processing saga - orchestration style
public class OrderProcessingSaga : 
    ISaga<OrderProcessingSagaState>,
    IAmStartedByMessages<StartOrderCommand>,
    IHandleMessages<PaymentProcessedEvent>,
    IHandleMessages<PaymentFailedEvent>,
    IHandleMessages<InventoryAllocatedEvent>,
    IHandleMessages<InventoryAllocationFailedEvent>,
    IHandleMessages<ShippingArrangedEvent>,
    IHandleMessages<ShippingArrangementFailedEvent>
{
    private readonly IOrderService _orderService;
    private readonly IPaymentService _paymentService;
    private readonly IInventoryService _inventoryService;
    private readonly IShippingService _shippingService;
    private readonly ILogger<OrderProcessingSaga> _logger;
    
    public OrderProcessingSaga(
        IOrderService orderService,
        IPaymentService paymentService,
        IInventoryService inventoryService,
        IShippingService shippingService,
        ILogger<OrderProcessingSaga> logger)
    {
        _orderService = orderService;
        _paymentService = paymentService;
        _inventoryService = inventoryService;
        _shippingService = shippingService;
        _logger = logger;
    }
    
    // Saga data
    public OrderProcessingSagaState SagaState { get; set; }
    
    // Start the saga
    public async Task Handle(StartOrderCommand message, IMessageHandlerContext context)
    {
        _logger.LogInformation("Starting order processing saga for Order {OrderId}", message.OrderId);
        
        SagaState.OrderId = message.OrderId;
        SagaState.CustomerId = message.CustomerId;
        SagaState.OrderItems = message.OrderItems;
        SagaState.TotalAmount = message.TotalAmount;
        
        // Step 1: Process payment
        await context.Send(new ProcessPaymentCommand
        {
            OrderId = SagaState.OrderId,
            CustomerId = SagaState.CustomerId,
            Amount = SagaState.TotalAmount
        });
    }
    
    // Handle payment processed
    public async Task Handle(PaymentProcessedEvent message, IMessageHandlerContext context)
    {
        _logger.LogInformation("Payment processed for Order {OrderId}", SagaState.OrderId);
        
        SagaState.PaymentId = message.PaymentId;
        SagaState.PaymentStatus = "Processed";
        
        // Step 2: Allocate inventory
        await context.Send(new AllocateInventoryCommand
        {
            OrderId = SagaState.OrderId,
            OrderItems = SagaState.OrderItems
        });
    }
    
    // Handle payment failed
    public async Task Handle(PaymentFailedEvent message, IMessageHandlerContext context)
    {
        _logger.LogWarning("Payment failed for Order {OrderId}: {Reason}", 
            SagaState.OrderId, message.Reason);
        
        SagaState.PaymentStatus = "Failed";
        SagaState.FailureReason = message.Reason;
        
        // Update order status to payment failed
        await _orderService.UpdateOrderStatusAsync(
            SagaState.OrderId, 
            "PaymentFailed", 
            message.Reason);
            
        // Mark saga as complete
        MarkAsComplete();
    }
    
    // Handle inventory allocated
    public async Task Handle(InventoryAllocatedEvent message, IMessageHandlerContext context)
    {
        _logger.LogInformation("Inventory allocated for Order {OrderId}", SagaState.OrderId);
        
        SagaState.InventoryStatus = "Allocated";
        
        // Step 3: Arrange shipping
        await context.Send(new ArrangeShippingCommand
        {
            OrderId = SagaState.OrderId,
            CustomerId = SagaState.CustomerId,
            ShippingAddress = await _orderService.GetShippingAddressAsync(SagaState.OrderId)
        });
    }
    
    // Handle inventory allocation failed
    public async Task Handle(InventoryAllocationFailedEvent message, IMessageHandlerContext context)
    {
        _logger.LogWarning("Inventory allocation failed for Order {OrderId}: {Reason}", 
            SagaState.OrderId, message.Reason);
        
        SagaState.InventoryStatus = "Failed";
        SagaState.FailureReason = message.Reason;
        
        // Compensating transaction: Refund payment
        if (SagaState.PaymentStatus == "Processed")
        {
            await context.Send(new RefundPaymentCommand
            {
                PaymentId = SagaState.PaymentId,
                OrderId = SagaState.OrderId,
                Reason = "Inventory allocation failed"
            });
        }
        
        // Update order status
        await _orderService.UpdateOrderStatusAsync(
            SagaState.OrderId, 
            "InventoryFailed", 
            message.Reason);
            
        // Mark saga as complete
        MarkAsComplete();
    }
    
    // Handle shipping arranged
    public async Task Handle(ShippingArrangedEvent message, IMessageHandlerContext context)
    {
        _logger.LogInformation("Shipping arranged for Order {OrderId}, Tracking: {TrackingNumber}", 
            SagaState.OrderId, message.TrackingNumber);
        
        SagaState.ShippingStatus = "Arranged";
        SagaState.TrackingNumber = message.TrackingNumber;
        
        // Update order with success
        await _orderService.UpdateOrderStatusAsync(
            SagaState.OrderId, 
            "Processing", 
            $"Shipping arranged, tracking: {message.TrackingNumber}");
            
        // Notify customer
        await context.Send(new SendOrderConfirmationCommand
        {
            OrderId = SagaState.OrderId,
            CustomerId = SagaState.CustomerId,
            TrackingNumber = message.TrackingNumber
        });
        
        // Mark saga as complete
        MarkAsComplete();
    }
    
    // Handle shipping arrangement failed
    public async Task Handle(ShippingArrangementFailedEvent message, IMessageHandlerContext context)
    {
        _logger.LogWarning("Shipping arrangement failed for Order {OrderId}: {Reason}", 
            SagaState.OrderId, message.Reason);
        
        SagaState.ShippingStatus = "Failed";
        SagaState.FailureReason = message.Reason;
        
        // Compensating transactions
        
        // 1. Release inventory
        if (SagaState.InventoryStatus == "Allocated")
        {
            await context.Send(new ReleaseInventoryCommand
            {
                OrderId = SagaState.OrderId,
                OrderItems = SagaState.OrderItems,
                Reason = "Shipping arrangement failed"
            });
        }
        
        // 2. Refund payment
        if (SagaState.PaymentStatus == "Processed")
        {
            await context.Send(new RefundPaymentCommand
            {
                PaymentId = SagaState.PaymentId,
                OrderId = SagaState.OrderId,
                Reason = "Shipping arrangement failed"
            });
        }
        
        // Update order status
        await _orderService.UpdateOrderStatusAsync(
            SagaState.OrderId, 
            "ShippingFailed", 
            message.Reason);
            
        // Mark saga as complete
        MarkAsComplete();
    }
}

// Saga state
public class OrderProcessingSagaState : ContainSagaData
{
    public Guid OrderId { get; set; }
    public string CustomerId { get; set; }
    public List<OrderItemDto> OrderItems { get; set; }
    public decimal TotalAmount { get; set; }
    
    public string PaymentId { get; set; }
    public string PaymentStatus { get; set; }
    
    public string InventoryStatus { get; set; }
    
    public string ShippingStatus { get; set; }
    public string TrackingNumber { get; set; }
    
    public string FailureReason { get; set; }
}
```

### Service Mesh

Infrastructure layer for service communication:

- **Traffic Management**: Routing, load balancing
- **Observability**: Metrics, logging, tracing
- **Security**: Authentication, encryption
- **Resilience**: Circuit breaking, retries, timeouts
- **Sidecar Pattern**: Communication handled by proxy

```yaml
# Istio configuration example for routing
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - match:
    - headers:
        x-api-version:
          exact: "v2"
    route:
    - destination:
        host: order-service
        subset: v2
  - route:
    - destination:
        host: order-service
        subset: v1

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      connectionPool:
        http:
          http1MaxPendingRequests: 100
          maxRequestsPerConnection: 10
      outlierDetection:
        consecutive5xxErrors: 5
        interval: 30s
        baseEjectionTime: 60s
```

## Deployment and Hosting

### Containerization

Packaging services in containers:

- **Docker Containers**: Lightweight, portable runtime
- **Image Consistency**: Same environment everywhere
- **Resource Isolation**: Contained resource usage
- **Fast Startup**: Quick deployment and scaling

```dockerfile
# Dockerfile for a .NET microservice
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["OrderService/OrderService.csproj", "OrderService/"]
RUN dotnet restore "OrderService/OrderService.csproj"
COPY . .
WORKDIR "/src/OrderService"
RUN dotnet build "OrderService.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "OrderService.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "OrderService.dll"]
```

### Orchestration with Kubernetes

Managing containerized services:

- **Service Orchestration**: Automated deployment and scaling
- **Service Discovery**: Find and connect to services
- **Load Balancing**: Distribute traffic across instances
- **Health Monitoring**: Automatic recovery from failures
- **Rolling Updates**: Zero-downtime deployments

```yaml
# Kubernetes deployment for a microservice
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: myregistry.io/order-service:1.0.0
        ports:
        - containerPort: 80
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__OrderDatabase
          valueFrom:
            secretKeyRef:
              name: order-service-secrets
              key: db-connection-string
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health/live
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### CI/CD Pipeline

Automated build and deployment:

- **Continuous Integration**: Automated testing on code changes
- **Continuous Deployment**: Automated deployment to production
- **Pipeline as Code**: Define pipeline in version control
- **Infrastructure as Code**: Deploy infrastructure changes
- **Multi-Environment Promotion**: Promote through test environments

```yaml
# Azure DevOps pipeline for a microservice
trigger:
  branches:
    include:
    - main
  paths:
    include:
    - src/OrderService/**

variables:
  serviceDirectory: 'src/OrderService'
  dockerRegistry: 'myregistry.azurecr.io'
  imageName: 'order-service'
  tag: '$(Build.BuildId)'
  k8sNamespace: 'microservices'

stages:
- stage: Build
  jobs:
  - job: BuildAndTest
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: DotNetCoreCLI@2
      displayName: 'Restore packages'
      inputs:
        command: 'restore'
        projects: '$(serviceDirectory)/**/*.csproj'
        
    - task: DotNetCoreCLI@2
      displayName: 'Build solution'
      inputs:
        command: 'build'
        projects: '$(serviceDirectory)/**/*.csproj'
        arguments: '--configuration Release'
        
    - task: DotNetCoreCLI@2
      displayName: 'Run tests'
      inputs:
        command: 'test'
        projects: '$(serviceDirectory).Tests/**/*.csproj'
        arguments: '--configuration Release'
        
    - task: Docker@2
      displayName: 'Build and push Docker image'
      inputs:
        containerRegistry: 'MyDockerRegistry'
        repository: '$(imageName)'
        command: 'buildAndPush'
        Dockerfile: '$(serviceDirectory)/Dockerfile'
        buildContext: '$(Build.SourcesDirectory)'
        tags: |
          $(tag)
          latest

- stage: DeployDev
  dependsOn: Build
  jobs:
  - deployment: DeployToDev
    environment: 'Development'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: 'Deploy to Kubernetes'
            inputs:
              action: 'deploy'
              kubernetesServiceConnection: 'DevK8sCluster'
              namespace: '$(k8sNamespace)'
              manifests: '$(serviceDirectory)/k8s/dev/*.yml'
              containers: '$(dockerRegistry)/$(imageName):$(tag)'

- stage: DeployProd
  dependsOn: DeployDev
  jobs:
  - deployment: DeployToProd
    environment: 'Production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: 'Deploy to Kubernetes'
            inputs:
              action: 'deploy'
              kubernetesServiceConnection: 'ProdK8sCluster'
              namespace: '$(k8sNamespace)'
              manifests: '$(serviceDirectory)/k8s/prod/*.yml'
              containers: '$(dockerRegistry)/$(imageName):$(tag)'
```

## Observability and Monitoring

### Distributed Tracing

Tracking requests across services:

- **Request Correlation**: Track requests across services
- **Performance Analysis**: Identify bottlenecks
- **Error Tracking**: Find sources of failures
- **Dependency Mapping**: Visualize service dependencies

```csharp
// Configure OpenTelemetry in ASP.NET Core
public void ConfigureServices(IServiceCollection services)
{
    services.AddOpenTelemetryTracing(builder =>
    {
        builder
            .SetResourceBuilder(ResourceBuilder.CreateDefault()
                .AddService("OrderService"))
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddEntityFrameworkCoreInstrumentation()
            .AddSource("OrderService")
            .AddJaegerExporter(options =>
            {
                options.AgentHost = Configuration["Jaeger:AgentHost"];
                options.AgentPort = int.Parse(Configuration["Jaeger:AgentPort"]);
            });
    });
}

// Custom tracing in service methods
public class OrderService : IOrderService
{
    private readonly ApplicationDbContext _dbContext;
    private readonly ActivitySource _activitySource;
    
    public OrderService(
        ApplicationDbContext dbContext,
        ITracer tracer)
    {
        _dbContext = dbContext;
        _activitySource = new ActivitySource("OrderService");
    }
    
    public async Task<Order> CreateOrderAsync(OrderCreateDto dto)
    {
        // Create activity for this operation
        using var activity = _activitySource.StartActivity(
            "CreateOrder",
            ActivityKind.Internal);
            
        // Add custom attributes
        activity?.SetTag("customerId", dto.CustomerId);
        activity?.SetTag("itemCount", dto.Items.Count);
        
        try
        {
            // Creating order business logic
            var order = new Order
            {
                CustomerId = dto.CustomerId,
                Items = dto.Items.Select(i => new OrderItem {...}).ToList(),
                Status = OrderStatus.Created,
                CreatedAt = DateTime.UtcNow
            };
            
            using (activity?.StartActivity("SaveOrderToDatabase"))
            {
                await _dbContext.Orders.AddAsync(order);
                await _dbContext.SaveChangesAsync();
            }
            
            using (activity?.StartActivity("PublishOrderCreatedEvent"))
            {
                // Publish event
                await _eventBus.PublishAsync(new OrderCreatedEvent(...));
            }
            
            activity?.SetStatus(ActivityStatusCode.Ok);
            return order;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        }
    }
}
```

### Centralized Logging

Aggregating logs from all services:

- **Log Aggregation**: Collect logs from all services
- **Structured Logging**: Machine-parseable log entries
- **Context Enrichment**: Add request IDs, correlation IDs
- **Log Searching**: Find relevant log entries across services

```csharp
// Configure Serilog in ASP.NET Core
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .UseSerilog((context, config) =>
        {
            config
                .ReadFrom.Configuration(context.Configuration)
                .Enrich.FromLogContext()
                .Enrich.WithProperty("Application", "OrderService")
                .Enrich.WithProperty("Environment", context.HostingEnvironment.EnvironmentName)
                .WriteTo.Console(outputTemplate: 
                    "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
                .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(
                    new Uri(context.Configuration["Elasticsearch:Uri"]))
                {
                    AutoRegisterTemplate = true,
                    IndexFormat = $"order-service-{context.HostingEnvironment.EnvironmentName}-{DateTime.UtcNow:yyyy-MM}"
                });
        })
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });

// Logging in controllers/services with context
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    private readonly ILogger<OrdersController> _logger;
    
    public OrdersController(
        IOrderService orderService,
        ILogger<OrdersController> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }
    
    [HttpPost]
    public async Task<ActionResult<OrderDto>> CreateOrder(OrderCreateDto dto)
    {
        var correlationId = Request.Headers["X-Correlation-Id"].FirstOrDefault() ?? Guid.NewGuid().ToString();
        
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["CorrelationId"] = correlationId,
            ["CustomerId"] = dto.CustomerId,
            ["ItemCount"] = dto.Items.Count
        }))
        {
            _logger.LogInformation("Creating new order for customer {CustomerId}", dto.CustomerId);
            
            try
            {
                var order = await _orderService.CreateOrderAsync(dto);
                
                _logger.LogInformation("Order {OrderId} created successfully", order.Id);
                
                return CreatedAtAction(
                    nameof(GetOrder), 
                    new { id = order.Id }, 
                    new OrderDto
                    {
                        Id = order.Id,
                        CustomerId = order.CustomerId,
                        Status = order.Status.ToString(),
                        TotalAmount = order.CalculateTotal(),
                        CreatedAt = order.CreatedAt
                    });
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error creating order for customer {CustomerId}", dto.CustomerId);
                throw;
            }
        }
    }
}
```

### Health Monitoring

Monitoring service health:

- **Health Checks**: Service-defined health status
- **Readiness Probes**: Service ready to handle requests
- **Liveness Probes**: Service is running correctly
- **Dependency Checks**: Check downstream services
- **Alerting**: Notify when services are unhealthy

```csharp
// Configure health checks in ASP.NET Core
public void ConfigureServices(IServiceCollection services)
{
    services.AddHealthChecks()
        // Basic readiness check
        .AddCheck("self", () => HealthCheckResult.Healthy())
        // Database check
        .AddDbContextCheck<ApplicationDbContext>("database")
        // Message broker check
        .AddRabbitMQ(
            rabbitConnectionString: Configuration["RabbitMQ:ConnectionString"],
            name: "rabbitmq")
        // External API check
        .AddUrlGroup(
            new Uri(Configuration["Services:Inventory"] + "/health"),
            name: "inventory-service-api",
            timeout: TimeSpan.FromSeconds(3))
        // Custom check for business logic
        .AddCheck("business-logic", () => 
        {
            // Check business rules or resources
            return HealthCheckResult.Healthy();
        });
        
    // Configure health check response format
    services.AddHealthChecksUI(setup =>
    {
        setup.SetEvaluationTimeInSeconds(60);
        setup.MaximumHistoryEntries = 10;
    })
    .AddInMemoryStorage();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // ... other middleware

    // Expose health endpoints
    app.UseHealthChecks("/health", new HealthCheckOptions
    {
        Predicate = _ => true,
        ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
    });
    
    app.UseHealthChecks("/health/ready", new HealthCheckOptions
    {
        Predicate = check => check.Tags.Contains("ready"),
        ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
    });
    
    app.UseHealthChecks("/health/live", new HealthCheckOptions
    {
        Predicate = check => check.Tags.Contains("live"),
        ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
    });
    
    app.UseHealthChecksUI(options =>
    {
        options.UIPath = "/health-ui";
    });
}
```

### Metrics Collection

Gathering performance metrics:

- **Business Metrics**: Domain-specific metrics
- **Technical Metrics**: CPU, memory, network usage
- **Application Metrics**: Request rates, error rates, latency
- **Custom Metrics**: Service-defined measurements
- **Dashboards**: Visualize metrics

```csharp
// Configure Prometheus metrics in ASP.NET Core
public void ConfigureServices(IServiceCollection services)
{
    // Add Prometheus metrics
    services.AddMetrics();
    
    services.AddOpenTelemetryMetrics(builder =>
    {
        builder
            .AddMeter("OrderService")
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddPrometheusExporter();
    });
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // ... other middleware
    
    // Expose Prometheus metrics endpoint
    app.UseOpenTelemetryPrometheusScrapingEndpoint();
}

// Custom metrics in services
public class OrderService : IOrderService
{
    private readonly ApplicationDbContext _dbContext;
    private readonly Counter<long> _orderCreatedCounter;
    private readonly Histogram<double> _orderProcessingDuration;
    
    public OrderService(
        ApplicationDbContext dbContext,
        IMeterFactory meterFactory)
    {
        _dbContext = dbContext;
        
        var meter = meterFactory.Create("OrderService");
        
        _orderCreatedCounter = meter.CreateCounter<long>(
            "orders_created_total",
            description: "Total number of orders created");
            
        _orderProcessingDuration = meter.CreateHistogram<double>(
            "order_processing_duration_seconds",
            description: "Time taken to process orders");
    }
    
    public async Task<Order> CreateOrderAsync(OrderCreateDto dto)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            // Order creation logic
            var order = new Order
            {
                CustomerId = dto.CustomerId,
                Items = dto.Items.Select(i => new OrderItem {...}).ToList(),
                Status = OrderStatus.Created,
                CreatedAt = DateTime.UtcNow
            };
            
            await _dbContext.Orders.AddAsync(order);
            await _dbContext.SaveChangesAsync();
            
            // Record metrics
            _orderCreatedCounter.Add(1, KeyValuePair.Create<string, object>("customer_type", GetCustomerType(dto.CustomerId)));
            
            return order;
        }
        finally
        {
            stopwatch.Stop();
            _orderProcessingDuration.Record(stopwatch.Elapsed.TotalSeconds);
        }
    }
    
    private string GetCustomerType(string customerId)
    {
        // Logic to determine customer type
        return "regular";
    }
}
```

## Best Practices

### Service Design Practices

Good practices for designing microservices:

- **Single Responsibility**: Focus on one business capability
- **Independently Deployable**: Services can be deployed without affecting others
- **Encapsulated State**: Services own their data and expose only APIs
- **Domain-Driven Design**: Align services with business domains
- **API-First Design**: Define stable APIs before implementation
- **Size Appropriately**: Not too large, not too small
- **Interface Autonomy**: Minimize runtime dependencies

### Resilience Practices

Building fault-tolerant systems:

- **Circuit Breakers**: Prevent cascading failures
- **Timeouts**: Don't wait indefinitely for responses
- **Retries with Backoff**: Retry with increasing delays
- **Fallbacks**: Provide alternatives when dependencies fail
- **Bulkheads**: Isolate resources to prevent total failure
- **Safe Defaults**: Provide reasonable defaults when data is unavailable

### Evolution Practices

Managing change in microservices:

- **API Versioning**: Support multiple API versions
- **Consumer-Driven Contracts**: Design APIs based on consumer needs
- **Incremental Migration**: Move to microservices gradually
- **Strangler Pattern**: Replace systems incrementally
- **Feature Toggles**: Enable/disable features at runtime
- **Parallel Change**: Support old and new behavior simultaneously

## Common Pitfalls

### Distributed Monolith

Avoiding tightly-coupled microservices:

- **Symptom**: Services must be deployed together
- **Symptom**: Changes in one service ripple to others
- **Symptom**: Synchronous communication between many services
- **Solution**: Enforce service autonomy
- **Solution**: Use asynchronous communication
- **Solution**: Identify and fix shared state

### Inappropriate Service Boundaries

Designing the right-sized services:

- **Symptom**: Services changing together frequently
- **Symptom**: Excessive inter-service communication
- **Symptom**: Inconsistent domain terminology
- **Solution**: Use Domain-Driven Design to identify boundaries
- **Solution**: Align services with business capabilities
- **Solution**: Refactor service boundaries as you learn

### Ignoring Network Effects

Dealing with distributed system challenges:

- **Symptom**: Timeouts due to slow network calls
- **Symptom**: Cascading failures
- **Symptom**: Inconsistent data across services
- **Solution**: Design for failures and latency
- **Solution**: Implement resilience patterns
- **Solution**: Use asynchronous communication when possible

### Insufficient Monitoring

Understanding the system:

- **Symptom**: Difficulty diagnosing issues
- **Symptom**: Unexpected failures in production
- **Symptom**: Cannot track requests across services
- **Solution**: Implement distributed tracing
- **Solution**: Centralize logging
- **Solution**: Collect and alert on metrics

## Further Reading

- [Building Microservices](https://www.oreilly.com/library/view/building-microservices/9781491950340/) by Sam Newman
- [Microservices Patterns](https://www.manning.com/books/microservices-patterns) by Chris Richardson
- [Domain-Driven Design: Tackling Complexity in the Heart of Software](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215) by Eric Evans
- [.NET Microservices: Architecture for Containerized .NET Applications](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/)
- [Production-Ready Microservices](https://www.oreilly.com/library/view/production-ready-microservices/9781491965962/) by Susan Fowler

## Related Topics

- [Domain-Driven Design for Microservices](domain-driven-design.md)
- [Event-Driven Architecture](../event-driven/introduction.md)
- [API Gateway Pattern](../patterns/api-gateway.md)
- [Service Mesh Implementation](../../containers/service-mesh/introduction.md)
- [Containerization with Docker](../../containers/docker/dotnet-containers.md)