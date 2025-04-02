---
title: "Azure Cloud Architecture for .NET Applications"
date_created: 2023-04-02
date_updated: 2023-04-02
authors: ["Repository Maintainers"]
tags: ["azure", "cloud", "architecture", "dotnet", "infrastructure"]
difficulty: "advanced"
---

# Azure Cloud Architecture for .NET Applications

## Overview

Microsoft Azure provides a comprehensive cloud platform that integrates seamlessly with .NET technologies. This document explores cloud architecture patterns, services, and best practices for building scalable, resilient, and cost-effective .NET applications on Azure. Understanding these architectural principles is critical for designing modern cloud-native solutions.

## Core Azure Services for .NET Applications

### Compute Services

Azure offers multiple compute options for hosting .NET applications:

#### Azure App Service

App Service is a fully managed platform for building, deploying, and scaling web apps:

```csharp
// Program.cs in a typical ASP.NET Core application deployed to App Service
var builder = WebApplication.CreateBuilder(args);

// App Service provides configuration through environment variables
// automatically mapped to configuration
builder.Configuration.AddEnvironmentVariables();

// Add App Insights for monitoring (commonly used with App Service)
builder.Services.AddApplicationInsightsTelemetry();

// Add services to the container
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Add health checks
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"));

var app = builder.Build();

// Configure the HTTP request pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();
app.MapHealthChecks("/health");

app.Run();
```

**Key features:**
- Multiple deployment slots for staging and production
- Auto-scaling based on metrics
- Built-in load balancing
- Integration with Azure DevOps
- Support for multiple .NET versions

#### Azure Functions

Serverless compute for event-driven applications:

```csharp
// Example Azure Function triggered by HTTP request
public static class OrderProcessingFunction
{
    [FunctionName("ProcessOrder")]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] HttpRequest req,
        [Queue("orders")] IAsyncCollector<Order> orderQueue,
        ILogger log)
    {
        log.LogInformation("Processing new order request");

        string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
        var order = JsonConvert.DeserializeObject<Order>(requestBody);
        
        if (order == null || string.IsNullOrEmpty(order.CustomerId))
        {
            return new BadRequestObjectResult("Please provide a valid order");
        }

        // Add order to queue for processing
        await orderQueue.AddAsync(order);
        
        log.LogInformation($"Order for customer {order.CustomerId} queued for processing");
        
        return new OkObjectResult($"Order received. ID: {order.Id}");
    }
    
    [FunctionName("ProcessOrderQueue")]
    public static async Task ProcessOrderQueue(
        [QueueTrigger("orders")] Order order,
        [CosmosDB("OrdersDb", "Orders", ConnectionStringSetting = "CosmosDbConnection")] IAsyncCollector<Order> ordersDocument,
        [SendGrid(ApiKey = "SendGridApiKey")] IAsyncCollector<SendGridMessage> messageCollector,
        ILogger log)
    {
        log.LogInformation($"Processing order: {order.Id}");
        
        // Store order in Cosmos DB
        await ordersDocument.AddAsync(order);
        
        // Send confirmation email
        var message = new SendGridMessage();
        message.AddTo(order.CustomerEmail);
        message.SetFrom("orders@example.com");
        message.SetSubject("Order Confirmation");
        message.AddContent("text/plain", $"Thank you for your order. Your order ID is {order.Id}");
        
        await messageCollector.AddAsync(message);
        
        log.LogInformation($"Order {order.Id} processed successfully");
    }
}
```

**Key features:**
- Pay-per-execution pricing model
- Automatic scaling
- Stateless design
- Multiple triggers and bindings
- Integration with other Azure services

#### Azure Kubernetes Service (AKS)

Managed Kubernetes for container orchestration:

```yaml
# Example Kubernetes manifest for a .NET application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myregistry.azurecr.io/myapp:1.0.0
        ports:
        - containerPort: 80
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "200m"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

**Key features:**
- Managed Kubernetes control plane
- Integration with Azure Container Registry
- Virtual node integration with Azure Container Instances
- RBAC and Azure Active Directory integration
- Advanced networking options

#### Choosing the Right Compute Service

| Feature | App Service | Functions | AKS |
|---------|------------|-----------|-----|
| Complexity | Low | Low | High |
| Scalability | Good | Excellent | Excellent |
| Control | Limited | Limited | Full |
| DevOps | Simple | Simple | Complex |
| Cost | Predictable | Pay-per-use | Infrastructure costs |
| Use Cases | Web apps, APIs | Event processing, scheduled tasks | Complex microservices |

### Data Storage Services

Azure provides various data storage options for .NET applications:

#### Azure SQL Database

Managed relational database service:

```csharp
// Configuring Entity Framework Core with Azure SQL Database
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"),
                sqlOptions =>
                {
                    // Configure resiliency
                    sqlOptions.EnableRetryOnFailure(
                        maxRetryCount: 5,
                        maxRetryDelay: TimeSpan.FromSeconds(30),
                        errorNumbersToAdd: null);
                    
                    // Improve performance
                    sqlOptions.CommandTimeout(30);
                    sqlOptions.MigrationsHistoryTable("__EFMigrationsHistory", "dbo");
                }));
    }
}

// Connection string in appsettings.json
// {
//   "ConnectionStrings": {
//     "DefaultConnection": "Server=tcp:myserver.database.windows.net,1433;Database=mydb;Authentication=Active Directory Default; Encrypt=True;TrustServerCertificate=False;"
//   }
// }
```

**Key features:**
- Managed service with high availability
- Automatic backups and point-in-time restore
- Automatic scaling options
- Advanced security features
- Compatible with Entity Framework Core

#### Azure Cosmos DB

Globally distributed, multi-model database service:

```csharp
// Configuring Cosmos DB in ASP.NET Core
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Add Cosmos DB client as singleton
        services.AddSingleton<CosmosClient>(sp =>
        {
            var configuration = sp.GetRequiredService<IConfiguration>();
            var cosmosDbSettings = configuration.GetSection("CosmosDb").Get<CosmosDbSettings>();
            
            return new CosmosClient(
                cosmosDbSettings.EndpointUri,
                cosmosDbSettings.PrimaryKey,
                new CosmosClientOptions
                {
                    ApplicationName = "MyApp",
                    ConnectionMode = ConnectionMode.Direct,
                    SerializerOptions = new CosmosSerializationOptions
                    {
                        PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase
                    }
                });
        });
        
        // Add repository that uses Cosmos DB
        services.AddScoped<IProductRepository, CosmosProductRepository>();
    }
}

// Repository implementation for Cosmos DB
public class CosmosProductRepository : IProductRepository
{
    private readonly CosmosClient _cosmosClient;
    private readonly Container _container;
    
    public CosmosProductRepository(CosmosClient cosmosClient, IConfiguration configuration)
    {
        _cosmosClient = cosmosClient;
        
        var databaseName = configuration["CosmosDb:DatabaseName"];
        var containerName = configuration["CosmosDb:ContainerName"];
        
        _container = _cosmosClient.GetContainer(databaseName, containerName);
    }
    
    public async Task<Product> GetByIdAsync(string id, string partitionKey)
    {
        try
        {
            var response = await _container.ReadItemAsync<Product>(id, new PartitionKey(partitionKey));
            return response.Resource;
        }
        catch (CosmosException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
        {
            return null;
        }
    }
    
    public async Task<IEnumerable<Product>> GetAllAsync(string queryString = "SELECT * FROM c")
    {
        var query = _container.GetItemQueryIterator<Product>(new QueryDefinition(queryString));
        var results = new List<Product>();
        
        while (query.HasMoreResults)
        {
            var response = await query.ReadNextAsync();
            results.AddRange(response.ToList());
        }
        
        return results;
    }
    
    public async Task<Product> AddAsync(Product product)
    {
        var response = await _container.CreateItemAsync(product, new PartitionKey(product.Category));
        return response.Resource;
    }
    
    public async Task UpdateAsync(Product product)
    {
        await _container.ReplaceItemAsync(
            product,
            product.Id,
            new PartitionKey(product.Category));
    }
    
    public async Task DeleteAsync(string id, string partitionKey)
    {
        await _container.DeleteItemAsync<Product>(id, new PartitionKey(partitionKey));
    }
}
```

**Key features:**
- Multi-model (document, key-value, graph, column-family)
- Global distribution
- Multi-master replication
- Automatic indexing
- Low-latency reads and writes

#### Azure Storage

Storage for blobs, files, queues, and tables:

```csharp
// Working with Azure Blob Storage
public class BlobStorageService : IBlobStorageService
{
    private readonly BlobServiceClient _blobServiceClient;
    
    public BlobStorageService(IConfiguration configuration)
    {
        string connectionString = configuration.GetConnectionString("AzureStorage");
        _blobServiceClient = new BlobServiceClient(connectionString);
    }
    
    public async Task<string> UploadFileAsync(string containerName, string fileName, Stream content)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        await containerClient.CreateIfNotExistsAsync();
        
        var blobClient = containerClient.GetBlobClient(fileName);
        await blobClient.UploadAsync(content, overwrite: true);
        
        return blobClient.Uri.ToString();
    }
    
    public async Task<Stream> DownloadFileAsync(string containerName, string fileName)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        var blobClient = containerClient.GetBlobClient(fileName);
        
        var response = await blobClient.DownloadAsync();
        return response.Value.Content;
    }
    
    public async Task DeleteFileAsync(string containerName, string fileName)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        var blobClient = containerClient.GetBlobClient(fileName);
        
        await blobClient.DeleteIfExistsAsync();
    }
}

// Working with Azure Queue Storage
public class QueueService : IQueueService
{
    private readonly QueueClient _queueClient;
    
    public QueueService(IConfiguration configuration)
    {
        string connectionString = configuration.GetConnectionString("AzureStorage");
        string queueName = configuration["QueueName"];
        
        var queueServiceClient = new QueueServiceClient(connectionString);
        _queueClient = queueServiceClient.GetQueueClient(queueName);
        
        _queueClient.CreateIfNotExists();
    }
    
    public async Task SendMessageAsync<T>(T message)
    {
        string messageJson = JsonSerializer.Serialize(message);
        string base64Message = Convert.ToBase64String(Encoding.UTF8.GetBytes(messageJson));
        
        await _queueClient.SendMessageAsync(base64Message);
    }
    
    public async Task<T> ReceiveMessageAsync<T>()
    {
        var response = await _queueClient.ReceiveMessageAsync();
        
        if (response.Value != null)
        {
            string base64Message = response.Value.MessageText;
            string messageJson = Encoding.UTF8.GetString(Convert.FromBase64String(base64Message));
            
            T message = JsonSerializer.Deserialize<T>(messageJson);
            
            await _queueClient.DeleteMessageAsync(response.Value.MessageId, response.Value.PopReceipt);
            
            return message;
        }
        
        return default;
    }
}
```

**Key features:**
- Durable, scalable storage
- Multiple storage types
- Tiered storage options
- Geo-redundancy
- Content Delivery Network integration

### Integration Services

#### Azure Service Bus

Enterprise messaging service:

```csharp
// Publishing messages to Azure Service Bus
public class OrderService
{
    private readonly ServiceBusClient _client;
    private readonly ServiceBusSender _sender;
    
    public OrderService(IConfiguration configuration)
    {
        string connectionString = configuration.GetConnectionString("ServiceBus");
        _client = new ServiceBusClient(connectionString);
        _sender = _client.CreateSender("orders");
    }
    
    public async Task SubmitOrderAsync(Order order)
    {
        // Create a message containing the order
        var message = new ServiceBusMessage(JsonSerializer.SerializeToUtf8Bytes(order))
        {
            ContentType = "application/json",
            Subject = $"Order-{order.Id}",
            MessageId = Guid.NewGuid().ToString(),
            CorrelationId = order.ReferenceNumber
        };
        
        // Add custom properties
        message.ApplicationProperties.Add("OrderType", order.Type);
        message.ApplicationProperties.Add("Priority", order.Priority);
        
        // Send the message
        await _sender.SendMessageAsync(message);
    }
    
    public async ValueTask DisposeAsync()
    {
        await _sender.DisposeAsync();
        await _client.DisposeAsync();
    }
}

// Consuming messages from Azure Service Bus
public class OrderProcessor : BackgroundService
{
    private readonly ILogger<OrderProcessor> _logger;
    private readonly ServiceBusClient _client;
    private readonly ServiceBusProcessor _processor;
    private readonly IOrderRepository _orderRepository;
    
    public OrderProcessor(
        IConfiguration configuration,
        IOrderRepository orderRepository,
        ILogger<OrderProcessor> logger)
    {
        _logger = logger;
        _orderRepository = orderRepository;
        
        string connectionString = configuration.GetConnectionString("ServiceBus");
        
        _client = new ServiceBusClient(connectionString);
        
        // Create a processor that handles messages
        _processor = _client.CreateProcessor("orders", new ServiceBusProcessorOptions
        {
            MaxConcurrentCalls = 2,
            AutoCompleteMessages = false
        });
        
        // Configure the message handler
        _processor.ProcessMessageAsync += ProcessMessageAsync;
        _processor.ProcessErrorAsync += ProcessErrorAsync;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Start processing messages
        await _processor.StartProcessingAsync(stoppingToken);
    }
    
    public override async Task StopAsync(CancellationToken stoppingToken)
    {
        // Stop processing messages
        await _processor.StopProcessingAsync(stoppingToken);
        await _processor.DisposeAsync();
        await _client.DisposeAsync();
    }
    
    private async Task ProcessMessageAsync(ProcessMessageEventArgs args)
    {
        var body = args.Message.Body.ToArray();
        var json = Encoding.UTF8.GetString(body);
        
        try
        {
            var order = JsonSerializer.Deserialize<Order>(json);
            
            _logger.LogInformation($"Processing order {order.Id}");
            
            // Process the order
            order.Status = "Processing";
            await _orderRepository.UpdateAsync(order);
            
            // Complete the message
            await args.CompleteMessageAsync(args.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing message");
            
            // Abandon the message so it can be reprocessed
            await args.AbandonMessageAsync(args.Message);
        }
    }
    
    private Task ProcessErrorAsync(ProcessErrorEventArgs args)
    {
        _logger.LogError(args.Exception, "Message handler encountered an exception");
        return Task.CompletedTask;
    }
}
```

**Key features:**
- Queues and topics (publish-subscribe)
- Message scheduling
- Transaction support
- Dead-letter queues
- Duplicate detection

#### Azure Event Grid

Event routing service:

```csharp
// Publishing events to Azure Event Grid
public class ProductService
{
    private readonly EventGridPublisherClient _eventGridClient;
    
    public ProductService(IConfiguration configuration)
    {
        string topicEndpoint = configuration["EventGrid:TopicEndpoint"];
        string accessKey = configuration["EventGrid:AccessKey"];
        
        _eventGridClient = new EventGridPublisherClient(
            new Uri(topicEndpoint),
            new AzureKeyCredential(accessKey));
    }
    
    public async Task<Product> CreateProductAsync(Product product)
    {
        // Business logic to create product
        // ...
        
        // Publish event to Event Grid
        var eventGridEvent = new EventGridEvent(
            subject: $"products/{product.Id}",
            eventType: "Product.Created",
            dataVersion: "1.0",
            data: new 
            {
                Id = product.Id,
                Name = product.Name,
                Category = product.Category,
                Price = product.Price
            });
            
        await _eventGridClient.SendEventAsync(eventGridEvent);
        
        return product;
    }
}

// Handling Event Grid events with an Azure Function
public static class EventGridFunctions
{
    [FunctionName("HandleProductEvents")]
    public static async Task Run(
        [EventGridTrigger] EventGridEvent eventGridEvent,
        [CosmosDB(
            databaseName: "CatalogDb",
            containerName: "ProductEvents",
            Connection = "CosmosDBConnection")] IAsyncCollector<ProductEvent> productEvents,
        ILogger log)
    {
        log.LogInformation($"Processing event: {eventGridEvent.EventType}");
        
        if (eventGridEvent.EventType == "Product.Created" || 
            eventGridEvent.EventType == "Product.Updated" ||
            eventGridEvent.EventType == "Product.Deleted")
        {
            var data = JsonSerializer.Deserialize<ProductData>(eventGridEvent.Data.ToString());
            
            var productEvent = new ProductEvent
            {
                Id = Guid.NewGuid().ToString(),
                ProductId = data.Id,
                EventType = eventGridEvent.EventType,
                Timestamp = DateTime.UtcNow,
                Data = data
            };
            
            await productEvents.AddAsync(productEvent);
            
            log.LogInformation($"Event for product {data.Id} recorded successfully");
        }
    }
    
    public class ProductData
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public string Category { get; set; }
        public decimal Price { get; set; }
    }
    
    public class ProductEvent
    {
        public string Id { get; set; }
        public string ProductId { get; set; }
        public string EventType { get; set; }
        public DateTime Timestamp { get; set; }
        public ProductData Data { get; set; }
    }
}
```

**Key features:**
- Pub/sub architecture
- Event filtering
- Built-in event sources
- Built-in webhooks
- Integration with Azure Functions

#### Azure Logic Apps

Workflow and integration service:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "logicAppName": {
      "type": "string"
    },
    "serviceBusConnectionString": {
      "type": "securestring"
    },
    "storageConnectionString": {
      "type": "securestring"
    },
    "sqlConnectionString": {
      "type": "securestring"
    }
  },
  "resources": [
    {
      "type": "Microsoft.Logic/workflows",
      "apiVersion": "2019-05-01",
      "name": "[parameters('logicAppName')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "definition": {
          "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
          "contentVersion": "1.0.0.0",
          "parameters": {},
          "triggers": {
            "When_a_message_is_received_in_a_queue": {
              "type": "ApiConnection",
              "inputs": {
                "host": {
                  "connection": {
                    "name": "@parameters('$connections')['servicebus']['connectionId']"
                  }
                },
                "method": "get",
                "path": "/@{encodeURIComponent('order-queue')}/messages/head",
                "queries": {
                  "queueType": "Main"
                }
              },
              "recurrence": {
                "frequency": "Minute",
                "interval": 1
              }
            }
          },
          "actions": {
            "Parse_JSON_Order": {
              "type": "ParseJson",
              "inputs": {
                "content": "@base64ToString(triggerBody().ContentData)",
                "schema": {
                  "properties": {
                    "customerId": { "type": "string" },
                    "id": { "type": "string" },
                    "items": {
                      "items": {
                        "properties": {
                          "productId": { "type": "string" },
                          "quantity": { "type": "integer" },
                          "unitPrice": { "type": "number" }
                        },
                        "type": "object"
                      },
                      "type": "array"
                    },
                    "orderDate": { "type": "string" },
                    "totalAmount": { "type": "number" }
                  },
                  "type": "object"
                }
              },
              "runAfter": {}
            },
            "Insert_order_into_SQL": {
              "type": "ApiConnection",
              "inputs": {
                "body": {
                  "CustomerId": "@body('Parse_JSON_Order').customerId",
                  "Id": "@body('Parse_JSON_Order').id",
                  "OrderDate": "@body('Parse_JSON_Order').orderDate",
                  "Status": "Processing",
                  "TotalAmount": "@body('Parse_JSON_Order').totalAmount"
                },
                "host": {
                  "connection": {
                    "name": "@parameters('$connections')['sql']['connectionId']"
                  }
                },
                "method": "post",
                "path": "/datasets/default/tables/@{encodeURIComponent('[dbo].[Orders]')}/items"
              },
              "runAfter": {
                "Parse_JSON_Order": [ "Succeeded" ]
              }
            },
            "Create_order_document": {
              "type": "ApiConnection",
              "inputs": {
                "body": "@body('Parse_JSON_Order')",
                "host": {
                  "connection": {
                    "name": "@parameters('$connections')['azureblob']['connectionId']"
                  }
                },
                "method": "post",
                "path": "/datasets/default/files",
                "queries": {
                  "folderPath": "/orders",
                  "name": "@concat(body('Parse_JSON_Order').id, '.json')",
                  "queryParametersSingleEncoded": true
                }
              },
              "runAfter": {
                "Insert_order_into_SQL": [ "Succeeded" ]
              }
            },
            "Send_email_notification": {
              "type": "ApiConnection",
              "inputs": {
                "body": {
                  "Body": "<p>A new order has been received.</p><p>Order ID: @{body('Parse_JSON_Order').id}<br>Customer ID: @{body('Parse_JSON_Order').customerId}<br>Total Amount: $@{body('Parse_JSON_Order').totalAmount}</p>",
                  "Subject": "New Order Received: @{body('Parse_JSON_Order').id}",
                  "To": "orders@example.com"
                },
                "host": {
                  "connection": {
                    "name": "@parameters('$connections')['office365']['connectionId']"
                  }
                },
                "method": "post",
                "path": "/v2/Mail"
              },
              "runAfter": {
                "Create_order_document": [ "Succeeded" ]
              }
            }
          }
        },
        "parameters": {
          "$connections": {
            "value": {
              "servicebus": {
                "id": "[concat('/subscriptions/', subscription().subscriptionId, '/providers/Microsoft.Web/locations/', resourceGroup().location, '/managedApis/servicebus')]",
                "connectionId": "[resourceId('Microsoft.Web/connections', 'servicebus')]",
                "connectionName": "servicebus"
              },
              "sql": {
                "id": "[concat('/subscriptions/', subscription().subscriptionId, '/providers/Microsoft.Web/locations/', resourceGroup().location, '/managedApis/sql')]",
                "connectionId": "[resourceId('Microsoft.Web/connections', 'sql')]",
                "connectionName": "sql"
              },
              "azureblob": {
                "id": "[concat('/subscriptions/', subscription().subscriptionId, '/providers/Microsoft.Web/locations/', resourceGroup().location, '/managedApis/azureblob')]",
                "connectionId": "[resourceId('Microsoft.Web/connections', 'azureblob')]",
                "connectionName": "azureblob"
              },
              "office365": {
                "id": "[concat('/subscriptions/', subscription().subscriptionId, '/providers/Microsoft.Web/locations/', resourceGroup().location, '/managedApis/office365')]",
                "connectionId": "[resourceId('Microsoft.Web/connections', 'office365')]",
                "connectionName": "office365"
              }
            }
          }
        }
      }
    }
  ]
}
```

**Key features:**
- Low-code/no-code development
- Built-in connectors for Azure and third-party services
- Visual workflow designer
- Enterprise integration capabilities
- Serverless execution

### Monitoring and Diagnostics

#### Azure Application Insights

Application performance monitoring:

```csharp
// Configure Application Insights in ASP.NET Core
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
            .ConfigureLogging((context, logging) =>
            {
                logging.ClearProviders();
                logging.AddConsole();
                
                // Add Application Insights logging
                if (!context.HostingEnvironment.IsDevelopment())
                {
                    logging.AddApplicationInsights(
                        context.Configuration["ApplicationInsights:InstrumentationKey"]);
                }
            });
}

// Add Application Insights in Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    // Add Application Insights telemetry
    services.AddApplicationInsightsTelemetry();
    
    // Add custom telemetry initializer
    services.AddSingleton<ITelemetryInitializer, TelemetryEnrichment>();
    
    // Add other services
}

// Custom telemetry initializer
public class TelemetryEnrichment : ITelemetryInitializer
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public TelemetryEnrichment(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }
    
    public void Initialize(ITelemetry telemetry)
    {
        var httpContext = _httpContextAccessor.HttpContext;
        if (httpContext != null)
        {
            // Add custom dimension for tenant
            if (httpContext.Request.Headers.TryGetValue("X-Tenant-Id", out var tenantId))
            {
                telemetry.Context.GlobalProperties["TenantId"] = tenantId;
            }
            
            // Add user ID if authenticated
            if (httpContext.User?.Identity?.IsAuthenticated == true)
            {
                telemetry.Context.User.AuthenticatedUserId = httpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            }
        }
        
        telemetry.Context.GlobalProperties["Environment"] = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT");
        telemetry.Context.GlobalProperties["ApplicationVersion"] = Assembly.GetExecutingAssembly().GetName().Version.ToString();
    }
}

// Manual tracking of telemetry
public class OrderService
{
    private readonly TelemetryClient _telemetryClient;
    
    public OrderService(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }
    
    public async Task<Order> CreateOrderAsync(OrderDto orderDto)
    {
        // Track dependency call
        using (var operation = _telemetryClient.StartOperation<DependencyTelemetry>("CreateOrder"))
        {
            operation.Telemetry.Type = "Business Operation";
            operation.Telemetry.Data = "Create new order";
            
            try
            {
                // Start timing
                var stopwatch = Stopwatch.StartNew();
                
                // Create order
                var order = new Order
                {
                    Id = Guid.NewGuid().ToString(),
                    CustomerId = orderDto.CustomerId,
                    Items = orderDto.Items,
                    TotalAmount = orderDto.Items.Sum(i => i.Quantity * i.UnitPrice),
                    Status = "New",
                    CreatedAt = DateTime.UtcNow
                };
                
                await _orderRepository.AddAsync(order);
                
                // Track custom event
                _telemetryClient.TrackEvent("OrderCreated", new Dictionary<string, string>
                {
                    ["OrderId"] = order.Id,
                    ["CustomerId"] = order.CustomerId,
                    ["ItemCount"] = order.Items.Count.ToString(),
                    ["TotalAmount"] = order.TotalAmount.ToString("F2")
                });
                
                // Track custom metric
                _telemetryClient.TrackMetric("OrderValue", order.TotalAmount);
                
                stopwatch.Stop();
                
                // Track timing
                _telemetryClient.TrackMetric("OrderCreationTime", stopwatch.ElapsedMilliseconds);
                
                operation.Telemetry.Success = true;
                return order;
            }
            catch (Exception ex)
            {
                // Track exception
                _telemetryClient.TrackException(ex, new Dictionary<string, string>
                {
                    ["CustomerId"] = orderDto.CustomerId,
                    ["Operation"] = "CreateOrder"
                });
                
                operation.Telemetry.Success = false;
                throw;
            }
        }
    }
}
```

**Key features:**
- Request monitoring
- Dependency tracking
- Exception tracking
- Custom events and metrics
- Distributed tracing
- Live metrics and diagnostics

## Cloud Architecture Patterns

### Scalability Patterns

#### Horizontal Scaling

Scaling out by adding more instances:

```csharp
// Implementing a scalable service with partitioning
public class UserService
{
    private readonly CosmosClient _cosmosClient;
    private readonly ILogger<UserService> _logger;
    
    // Number of partitions
    private const int PartitionCount = 16;
    
    public UserService(CosmosClient cosmosClient, ILogger<UserService> logger)
    {
        _cosmosClient = cosmosClient;
        _logger = logger;
    }
    
    public async Task<User> GetUserAsync(string userId)
    {
        // Determine partition
        string partitionKey = GetPartitionKey(userId);
        
        _logger.LogInformation("Getting user {UserId} from partition {Partition}", userId, partitionKey);
        
        // Get database and container
        var container = _cosmosClient.GetContainer("UsersDb", "Users");
        
        try
        {
            // Get user from the correct partition
            var response = await container.ReadItemAsync<User>(userId, new PartitionKey(partitionKey));
            return response.Resource;
        }
        catch (CosmosException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
        {
            _logger.LogWarning("User {UserId} not found", userId);
            return null;
        }
    }
    
    public async Task CreateUserAsync(User user)
    {
        // Assign partition key
        user.PartitionKey = GetPartitionKey(user.Id);
        
        _logger.LogInformation("Creating user {UserId} in partition {Partition}", user.Id, user.PartitionKey);
        
        var container = _cosmosClient.GetContainer("UsersDb", "Users");
        await container.CreateItemAsync(user, new PartitionKey(user.PartitionKey));
    }
    
    // Determine partition based on user ID
    private string GetPartitionKey(string userId)
    {
        // Simple hash-based partitioning
        int hashCode = userId.GetHashCode();
        int partitionIndex = Math.Abs(hashCode % PartitionCount);
        return partitionIndex.ToString();
    }
}
```

**Key implementation strategies:**
- Stateless services
- Distributed caching
- Data partitioning
- Queue-based load leveling
- Auto-scaling based on metrics

#### Queue-Based Load Leveling

Using queues to handle varying workloads:

```csharp
// API endpoint that offloads work to a queue
[ApiController]
[Route("api/[controller]")]
public class ReportsController : ControllerBase
{
    private readonly QueueClient _queueClient;
    private readonly ILogger<ReportsController> _logger;
    
    public ReportsController(IConfiguration configuration, ILogger<ReportsController> logger)
    {
        _logger = logger;
        
        string connectionString = configuration.GetConnectionString("StorageAccount");
        _queueClient = new QueueClient(connectionString, "report-generation");
        
        _queueClient.CreateIfNotExists();
    }
    
    [HttpPost]
    public async Task<IActionResult> GenerateReport(ReportRequest request)
    {
        var reportJob = new ReportJob
        {
            Id = Guid.NewGuid().ToString(),
            ReportType = request.ReportType,
            Parameters = request.Parameters,
            UserId = User.FindFirstValue(ClaimTypes.NameIdentifier),
            RequestedAt = DateTime.UtcNow
        };
        
        _logger.LogInformation("Queueing report generation request {ReportId}", reportJob.Id);
        
        // Serialize and encode message
        string messageJson = JsonSerializer.Serialize(reportJob);
        
        // Send to queue
        await _queueClient.SendMessageAsync(
            Convert.ToBase64String(Encoding.UTF8.GetBytes(messageJson)),
            timeToLive: TimeSpan.FromHours(2),
            visibilityTimeout: TimeSpan.FromMinutes(5));
            
        return Accepted(new 
        { 
            id = reportJob.Id,
            message = "Report generation has been queued",
            statusUrl = $"/api/reports/{reportJob.Id}/status"
        });
    }
    
    [HttpGet("{id}/status")]
    public async Task<IActionResult> GetReportStatus(string id)
    {
        // Check status in database
        var status = await _reportStatusRepository.GetStatusAsync(id);
        
        if (status == null)
            return NotFound();
            
        return Ok(status);
    }
}

// Background service that processes queued jobs
public class ReportGenerationService : BackgroundService
{
    private readonly QueueClient _queueClient;
    private readonly IReportGenerator _reportGenerator;
    private readonly IReportStatusRepository _statusRepository;
    private readonly ILogger<ReportGenerationService> _logger;
    
    public ReportGenerationService(
        IConfiguration configuration,
        IReportGenerator reportGenerator,
        IReportStatusRepository statusRepository,
        ILogger<ReportGenerationService> logger)
    {
        _reportGenerator = reportGenerator;
        _statusRepository = statusRepository;
        _logger = logger;
        
        string connectionString = configuration.GetConnectionString("StorageAccount");
        _queueClient = new QueueClient(connectionString, "report-generation");
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Report Generation Service is starting");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // Get messages from queue
                var messages = await _queueClient.ReceiveMessagesAsync(
                    maxMessages: 5,
                    visibilityTimeout: TimeSpan.FromMinutes(10),
                    cancellationToken: stoppingToken);
                    
                foreach (var message in messages.Value)
                {
                    await ProcessMessageAsync(message, stoppingToken);
                }
                
                // If no messages, wait before polling again
                if (messages.Value.Count() == 0)
                {
                    await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
                }
            }
            catch (Exception ex) when (!stoppingToken.IsCancellationRequested)
            {
                _logger.LogError(ex, "Error processing report generation queue");
                
                // Delay before retrying
                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            }
        }
    }
    
    private async Task ProcessMessageAsync(QueueMessage message, CancellationToken stoppingToken)
    {
        try
        {
            // Decode and deserialize
            string json = Encoding.UTF8.GetString(Convert.FromBase64String(message.MessageText));
            var reportJob = JsonSerializer.Deserialize<ReportJob>(json);
            
            _logger.LogInformation("Processing report job {ReportId}", reportJob.Id);
            
            // Update status
            await _statusRepository.UpdateStatusAsync(reportJob.Id, "Processing");
            
            // Generate report
            var reportResult = await _reportGenerator.GenerateReportAsync(
                reportJob.ReportType, 
                reportJob.Parameters,
                stoppingToken);
                
            // Update status
            await _statusRepository.CompleteAsync(reportJob.Id, reportResult.ReportUrl);
            
            // Delete message from queue
            await _queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt, stoppingToken);
            
            _logger.LogInformation("Report job {ReportId} completed successfully", reportJob.Id);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing report job from message {MessageId}", message.MessageId);
            
            // Don't delete the message - it will become visible again after the visibility timeout
        }
    }
}
```

**Benefits:**
- Smooths out traffic spikes
- Decouples services
- Enables independent scaling
- Provides a buffer for backend processing
- Improves resilience

### Resilience Patterns

#### Circuit Breaker

Preventing cascading failures:

```csharp
// Circuit breaker implementation with Polly
public class ExternalServiceClient
{
    private readonly HttpClient _httpClient;
    private readonly IAsyncPolicy<HttpResponseMessage> _circuitBreakerPolicy;
    private readonly ILogger<ExternalServiceClient> _logger;
    
    public ExternalServiceClient(HttpClient httpClient, ILogger<ExternalServiceClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
        
        // Define circuit breaker policy
        _circuitBreakerPolicy = Policy
            .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
            .OrTransientHttpError()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromMinutes(1),
                onBreak: (result, breakDuration) =>
                {
                    _logger.LogWarning(
                        "Circuit breaker opened for {BreakDuration}. Last status code: {StatusCode}",
                        breakDuration,
                        result.Result?.StatusCode);
                },
                onReset: () =>
                {
                    _logger.LogInformation("Circuit breaker reset");
                },
                onHalfOpen: () =>
                {
                    _logger.LogInformation("Circuit breaker half-open");
                });
    }
    
    public async Task<T> GetDataAsync<T>(string endpoint)
    {
        try
        {
            // Execute with circuit breaker policy
            var response = await _circuitBreakerPolicy.ExecuteAsync(() => 
                _httpClient.GetAsync(endpoint));
                
            response.EnsureSuccessStatusCode();
            
            string json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<T>(json);
        }
        catch (BrokenCircuitException)
        {
            _logger.LogError("Circuit is open. Returning default or cached value");
            return GetFallbackData<T>();
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Error calling external service");
            throw;
        }
    }
    
    private T GetFallbackData<T>()
    {
        // Return cached or default data
        return default;
    }
}
```

**Implementation considerations:**
- Failure detection criteria
- Circuit states (closed, open, half-open)
- Timeout settings
- Retry configurations
- Fallback mechanisms

#### Bulkhead

Isolating failures:

```csharp
// Bulkhead pattern implementation with Polly
public class MultiTenantApiClient
{
    private readonly HttpClient _httpClient;
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _tenantSemaphores;
    private readonly ILogger<MultiTenantApiClient> _logger;
    
    // Maximum concurrent requests per tenant
    private const int MaxConcurrentRequestsPerTenant = 10;
    
    public MultiTenantApiClient(HttpClient httpClient, ILogger<MultiTenantApiClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
        _tenantSemaphores = new ConcurrentDictionary<string, SemaphoreSlim>();
    }
    
    public async Task<ApiResponse> SendRequestAsync(string tenantId, ApiRequest request)
    {
        // Get or create semaphore for this tenant
        var semaphore = _tenantSemaphores.GetOrAdd(
            tenantId,
            _ => new SemaphoreSlim(MaxConcurrentRequestsPerTenant));
            
        // Try to enter the semaphore with timeout
        if (!await semaphore.WaitAsync(TimeSpan.FromSeconds(5)))
        {
            _logger.LogWarning(
                "Bulkhead rejection for tenant {TenantId}. Too many concurrent requests.",
                tenantId);
                
            throw new BulkheadRejectedException($"Too many concurrent requests for tenant {tenantId}");
        }
        
        try
        {
            _logger.LogInformation(
                "Sending request for tenant {TenantId}. Current concurrent requests: {Count}",
                tenantId,
                MaxConcurrentRequestsPerTenant - semaphore.CurrentCount + 1);
                
            // Add tenant ID to request headers
            _httpClient.DefaultRequestHeaders.Remove("X-Tenant-ID");
            _httpClient.DefaultRequestHeaders.Add("X-Tenant-ID", tenantId);
            
            // Send the request
            var response = await _httpClient.SendAsync(
                new HttpRequestMessage(HttpMethod.Post, "/api/data")
                {
                    Content = new StringContent(
                        JsonSerializer.Serialize(request),
                        Encoding.UTF8,
                        "application/json")
                });
                
            response.EnsureSuccessStatusCode();
            
            string json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<ApiResponse>(json);
        }
        finally
        {
            // Release the semaphore
            semaphore.Release();
        }
    }
}

public class BulkheadRejectedException : Exception
{
    public BulkheadRejectedException(string message) : base(message) { }
}
```

**Key implementation strategies:**
- Resource isolation per tenant/service
- Thread pool isolation
- Connection limits
- Timeout handling
- Queue-based throttling

### Cost Optimization Patterns

#### Consumption-Based Scaling

Scaling resources based on demand:

```csharp
// Function app with consumption plan scaling
public class ProcessOrderFunction
{
    [FunctionName("ProcessOrder")]
    public static async Task Run(
        [ServiceBusTrigger("orders", Connection = "ServiceBusConnection")] string message,
        [CosmosDB(
            databaseName: "OrdersDb",
            containerName: "Orders",
            Connection = "CosmosDbConnection")] IAsyncCollector<Order> orderCollector,
        ILogger log)
    {
        try
        {
            log.LogInformation($"Processing order: {message}");
            
            var order = JsonSerializer.Deserialize<Order>(message);
            
            // Process order
            order.Status = "Processing";
            order.ProcessedAt = DateTime.UtcNow;
            
            // Save to Cosmos DB
            await orderCollector.AddAsync(order);
            
            log.LogInformation($"Order {order.Id} processed successfully");
        }
        catch (Exception ex)
        {
            log.LogError(ex, $"Error processing order: {message}");
            throw; // Service Bus will retry
        }
    }
}
```

**Key considerations:**
- Pay-per-use pricing models
- Automated scaling rules
- Serverless architectures
- Resource deprovisioning
- Cold start management

#### Storage Tiering

Using appropriate storage tiers for different data:

```csharp
// Tiered storage manager for document archiving
public class DocumentStorageManager
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly ILogger<DocumentStorageManager> _logger;
    
    public DocumentStorageManager(
        IConfiguration configuration,
        ILogger<DocumentStorageManager> logger)
    {
        _logger = logger;
        
        string connectionString = configuration.GetConnectionString("StorageAccount");
        _blobServiceClient = new BlobServiceClient(connectionString);
    }
    
    public async Task<string> StoreDocumentAsync(DocumentType type, Stream content, string filename)
    {
        // Choose container based on document type
        string containerName = GetContainerName(type);
        
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        await containerClient.CreateIfNotExistsAsync();
        
        // Choose storage tier based on document type
        var blobClient = containerClient.GetBlobClient(filename);
        
        // Upload with appropriate access tier
        await blobClient.UploadAsync(
            content,
            new BlobUploadOptions
            {
                AccessTier = GetAccessTier(type),
                HttpHeaders = new BlobHttpHeaders
                {
                    ContentType = GetContentType(filename)
                }
            });
            
        _logger.LogInformation(
            "Document uploaded to {Container} with {AccessTier} access tier: {Filename}",
            containerName,
            GetAccessTier(type),
            filename);
            
        return blobClient.Uri.ToString();
    }
    
    public async Task ArchiveDocumentAsync(DocumentType type, string filename)
    {
        string containerName = GetContainerName(type);
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        var blobClient = containerClient.GetBlobClient(filename);
        
        // Set to Archive tier
        await blobClient.SetAccessTierAsync(AccessTier.Archive);
        
        _logger.LogInformation(
            "Document archived: {Container}/{Filename}",
            containerName,
            filename);
    }
    
    private string GetContainerName(DocumentType type)
    {
        return type switch
        {
            DocumentType.Invoice => "invoices",
            DocumentType.Contract => "contracts",
            DocumentType.Report => "reports",
            _ => "documents"
        };
    }
    
    private AccessTier GetAccessTier(DocumentType type)
    {
        return type switch
        {
            DocumentType.Invoice => AccessTier.Hot, // Frequently accessed
            DocumentType.Contract => AccessTier.Cool, // Infrequently accessed
            DocumentType.Report => AccessTier.Cool, // Infrequently accessed
            _ => AccessTier.Hot
        };
    }
    
    private string GetContentType(string filename)
    {
        string extension = Path.GetExtension(filename).ToLowerInvariant();
        
        return extension switch
        {
            ".pdf" => "application/pdf",
            ".docx" => "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
            ".xlsx" => "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
            ".json" => "application/json",
            ".xml" => "application/xml",
            _ => "application/octet-stream"
        };
    }
}

public enum DocumentType
{
    Invoice,
    Contract,
    Report,
    Other
}
```

**Implementation strategies:**
- Hot, cool, and archive storage tiers
- Data lifecycle management
- Automated tiering policies
- Retention policies
- Cost-based data access patterns

### Security Patterns

#### Identity and Access Management

Centralized authentication and authorization:

```csharp
// Configure Azure AD authentication in ASP.NET Core
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddMicrosoftIdentityWebApi(Configuration.GetSection("AzureAd"));
            
        services.AddAuthorization(options =>
        {
            // Add policies for roles
            options.AddPolicy("RequireAdministratorRole", policy =>
                policy.RequireRole("Administrator"));
                
            options.AddPolicy("ProductManagement", policy =>
                policy.RequireClaim("permission", "products.write"));
                
            // Add policies for app roles
            options.AddPolicy("ServiceAccess", policy =>
                policy.RequireAssertion(context =>
                    context.User.IsInRole("Service") || 
                    context.User.IsInRole("Administrator")));
        });
        
        // Add user accessor for logging/telemetry
        services.AddHttpContextAccessor();
        services.AddSingleton<ICurrentUserService, CurrentUserService>();
        
        services.AddControllers();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // ...
        
        app.UseAuthentication();
        app.UseAuthorization();
        
        // ...
    }
}

// Current user service
public interface ICurrentUserService
{
    string UserId { get; }
    string Username { get; }
    bool IsAuthenticated { get; }
    bool IsInRole(string role);
    string GetTenant();
}

public class CurrentUserService : ICurrentUserService
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public CurrentUserService(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }
    
    public string UserId => _httpContextAccessor.HttpContext?.User?.FindFirstValue(ClaimTypes.NameIdentifier);
    
    public string Username => _httpContextAccessor.HttpContext?.User?.FindFirstValue(ClaimTypes.Name);
    
    public bool IsAuthenticated => _httpContextAccessor.HttpContext?.User?.Identity?.IsAuthenticated ?? false;
    
    public bool IsInRole(string role) => _httpContextAccessor.HttpContext?.User?.IsInRole(role) ?? false;
    
    public string GetTenant() => _httpContextAccessor.HttpContext?.User?.FindFirstValue("tenant");
}

// Using authorization in a controller
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly ICurrentUserService _currentUserService;
    
    public ProductsController(
        IProductService productService,
        ICurrentUserService currentUserService)
    {
        _productService = productService;
        _currentUserService = currentUserService;
    }
    
    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        var products = await _productService.GetProductsForTenantAsync(
            _currentUserService.GetTenant());
            
        return Ok(products);
    }
    
    [HttpPost]
    [Authorize(Policy = "ProductManagement")]
    public async Task<ActionResult<Product>> CreateProduct(ProductCreateDto dto)
    {
        var product = await _productService.CreateProductAsync(
            dto,
            _currentUserService.GetTenant(),
            _currentUserService.UserId);
            
        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }
    
    [HttpGet("{id}")]
    public async Task<ActionResult<Product>> GetProduct(string id)
    {
        var product = await _productService.GetProductByIdAsync(id);
        
        if (product == null)
            return NotFound();
            
        // Check tenant access
        if (product.TenantId != _currentUserService.GetTenant())
            return Forbid();
            
        return Ok(product);
    }
    
    [HttpDelete("{id}")]
    [Authorize(Roles = "Administrator")]
    public async Task<IActionResult> DeleteProduct(string id)
    {
        var success = await _productService.DeleteProductAsync(
            id,
            _currentUserService.GetTenant());
            
        if (!success)
            return NotFound();
            
        return NoContent();
    }
}
```

**Key implementation features:**
- Centralized identity provider
- Role-based access control
- Claims-based authorization
- OAuth 2.0 and OpenID Connect
- Conditional access policies

#### Managed Identities

Using Azure-managed identities for secure service authentication:

```csharp
// Using managed identity for Azure Storage access
public class SecureStorageService
{
    private readonly BlobServiceClient _blobServiceClient;
    
    public SecureStorageService(IConfiguration configuration)
    {
        // Get Storage account name
        string accountName = configuration["Storage:AccountName"];
        
        // Use DefaultAzureCredential which supports managed identities
        _blobServiceClient = new BlobServiceClient(
            new Uri($"https://{accountName}.blob.core.windows.net"),
            new DefaultAzureCredential());
    }
    
    public async Task<string> UploadFileAsync(string containerName, string fileName, Stream content)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        await containerClient.CreateIfNotExistsAsync();
        
        var blobClient = containerClient.GetBlobClient(fileName);
        await blobClient.UploadAsync(content, overwrite: true);
        
        return blobClient.Uri.ToString();
    }
}

// Using managed identity for Azure SQL Database
public class SqlDatabaseService
{
    private readonly string _connectionString;
    
    public SqlDatabaseService(IConfiguration configuration)
    {
        // Get SQL Server and database name
        string server = configuration["Sql:Server"];
        string database = configuration["Sql:Database"];
        
        // Create connection string with managed identity
        _connectionString = $"Server={server}; Database={database}; Authentication=Active Directory Default;";
    }
    
    public async Task<IEnumerable<Customer>> GetCustomersAsync()
    {
        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();
        
        var customers = new List<Customer>();
        
        using var command = new SqlCommand("SELECT * FROM Customers", connection);
        using var reader = await command.ExecuteReaderAsync();
        
        while (await reader.ReadAsync())
        {
            customers.Add(new Customer
            {
                Id = reader.GetInt32(0),
                Name = reader.GetString(1),
                Email = reader.GetString(2)
            });
        }
        
        return customers;
    }
}

// Using managed identity for Key Vault access
public class SecretManager
{
    private readonly SecretClient _secretClient;
    
    public SecretManager(IConfiguration configuration)
    {
        string keyVaultUrl = configuration["KeyVault:Url"];
        
        // Use DefaultAzureCredential which supports managed identities
        _secretClient = new SecretClient(
            new Uri(keyVaultUrl),
            new DefaultAzureCredential());
    }
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        KeyVaultSecret secret = await _secretClient.GetSecretAsync(secretName);
        return secret.Value;
    }
}
```

**Benefits:**
- No stored credentials in code or configuration
- Automatic credential rotation
- Simplified key management
- Supports multiple Azure services
- Role-based access control integration

## Cloud-Native Application Patterns

### Valet Key Pattern

Delegating restricted access to resources:

```csharp
// Valet key pattern for secure file uploads
public class SecureFileUploadService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly ILogger<SecureFileUploadService> _logger;
    
    public SecureFileUploadService(IConfiguration configuration, ILogger<SecureFileUploadService> logger)
    {
        _logger = logger;
        
        string connectionString = configuration.GetConnectionString("StorageAccount");
        _blobServiceClient = new BlobServiceClient(connectionString);
    }
    
    // Generate a Shared Access Signature for direct upload
    public async Task<FileUploadSasToken> GetUploadSasTokenAsync(string userId, string filename, string contentType)
    {
        // Validate input
        if (string.IsNullOrEmpty(userId) || string.IsNullOrEmpty(filename))
        {
            throw new ArgumentException("User ID and filename are required");
        }
        
        // Generate unique blob name
        string blobName = $"{userId}/{Guid.NewGuid()}-{Path.GetFileName(filename)}";
        
        // Create container if it doesn't exist
        var containerClient = _blobServiceClient.GetBlobContainerClient("user-uploads");
        await containerClient.CreateIfNotExistsAsync();
        
        // Get blob client
        var blobClient = containerClient.GetBlobClient(blobName);
        
        // Define SAS token properties
        var sasBuilder = new BlobSasBuilder
        {
            BlobContainerName = containerClient.Name,
            BlobName = blobName,
            Resource = "b", // "b" for blob
            StartsOn = DateTimeOffset.UtcNow,
            ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(15) // Short expiration time
        };
        
        // Set permissions (only write and create)
        sasBuilder.SetPermissions(BlobSasPermissions.Write | BlobSasPermissions.Create);
        
        // Generate SAS token
        var sasToken = sasBuilder.ToSasQueryParameters(
            new StorageSharedKeyCredential(
                _blobServiceClient.AccountName,
                GetStorageAccountKey())).ToString();
                
        // Return upload information
        var uploadInfo = new FileUploadSasToken
        {
            SasUri = blobClient.Uri + "?" + sasToken,
            BlobUri = blobClient.Uri.ToString(),
            Filename = blobName,
            ExpiresOn = sasBuilder.ExpiresOn,
            ContentType = contentType
        };
        
        _logger.LogInformation(
            "Generated upload SAS token for user {UserId}. Blob: {BlobName}. Expires: {ExpiryTime}",
            userId,
            blobName,
            sasBuilder.ExpiresOn);
            
        return uploadInfo;
    }
    
    // Get storage account key from Key Vault (in production)
    private string GetStorageAccountKey()
    {
        // In a real application, this should be retrieved from Key Vault
        // using a managed identity
        return Environment.GetEnvironmentVariable("StorageAccountKey");
    }
}

public class FileUploadSasToken
{
    public string SasUri { get; set; }
    public string BlobUri { get; set; }
    public string Filename { get; set; }
    public DateTimeOffset ExpiresOn { get; set; }
    public string ContentType { get; set; }
}

// API endpoint to get SAS token
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class FileUploadsController : ControllerBase
{
    private readonly SecureFileUploadService _uploadService;
    private readonly ICurrentUserService _currentUserService;
    
    public FileUploadsController(
        SecureFileUploadService uploadService,
        ICurrentUserService currentUserService)
    {
        _uploadService = uploadService;
        _currentUserService = currentUserService;
    }
    
    [HttpPost("get-upload-token")]
    public async Task<ActionResult<FileUploadSasToken>> GetUploadToken(UploadRequest request)
    {
        // Validate request
        if (string.IsNullOrEmpty(request.Filename))
        {
            return BadRequest("Filename is required");
        }
        
        // Validate file type
        string extension = Path.GetExtension(request.Filename).ToLowerInvariant();
        if (!AllowedExtensions.Contains(extension))
        {
            return BadRequest($"File type not allowed: {extension}");
        }
        
        // Get content type
        string contentType = GetContentType(extension);
        
        // Generate SAS token
        var token = await _uploadService.GetUploadSasTokenAsync(
            _currentUserService.UserId,
            request.Filename,
            contentType);
            
        return Ok(token);
    }
    
    private static readonly HashSet<string> AllowedExtensions = new HashSet<string>
    {
        ".jpg", ".jpeg", ".png", ".gif", ".pdf", ".docx", ".xlsx", ".pptx"
    };
    
    private string GetContentType(string extension)
    {
        return extension switch
        {
            ".jpg" or ".jpeg" => "image/jpeg",
            ".png" => "image/png",
            ".gif" => "image/gif",
            ".pdf" => "application/pdf",
            ".docx" => "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
            ".xlsx" => "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
            ".pptx" => "application/vnd.openxmlformats-officedocument.presentationml.presentation",
            _ => "application/octet-stream"
        };
    }
}
```

**Key implementation features:**
- Time-limited access
- Scoped permissions
- Secure token generation
- Policy enforcement
- Audit logging

### Gatekeeper Pattern

Protecting sensitive operations with a dedicated service:

```csharp
// Gatekeeper service for secure payment processing
public class PaymentGatekeeperService
{
    private readonly HttpClient _paymentProcessorClient;
    private readonly ITokenValidationService _tokenValidator;
    private readonly ILogger<PaymentGatekeeperService> _logger;
    
    public PaymentGatekeeperService(
        IHttpClientFactory httpClientFactory,
        ITokenValidationService tokenValidator,
        ILogger<PaymentGatekeeperService> logger)
    {
        _paymentProcessorClient = httpClientFactory.CreateClient("PaymentProcessor");
        _tokenValidator = tokenValidator;
        _logger = logger;
    }
    
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request, string authToken)
    {
        // Validate the authentication token
        var validationResult = await _tokenValidator.ValidateTokenAsync(authToken);
        if (!validationResult.IsValid)
        {
            _logger.LogWarning(
                "Invalid authentication token from {IpAddress}. Error: {Error}",
                GetClientIp(),
                validationResult.Error);
                
            throw new UnauthorizedAccessException("Invalid or expired token");
        }
        
        // Validate the payment request
        ValidatePaymentRequest(request);
        
        // Sanitize sensitive data
        var sanitizedRequest = SanitizeRequest(request);
        
        // Add security headers
        _paymentProcessorClient.DefaultRequestHeaders.Add("X-Transaction-ID", Guid.NewGuid().ToString());
        _paymentProcessorClient.DefaultRequestHeaders.Add("X-Client-ID", validationResult.ClientId);
        
        try
        {
            // Forward the request to the payment processor
            var response = await _paymentProcessorClient.PostAsJsonAsync(
                "api/payments",
                sanitizedRequest);
                
            response.EnsureSuccessStatusCode();
            
            var result = await response.Content.ReadFromJsonAsync<PaymentResult>();
            
            _logger.LogInformation(
                "Payment processed successfully. Transaction ID: {TransactionId}, Amount: {Amount}, Status: {Status}",
                result.TransactionId,
                request.Amount,
                result.Status);
                
            return result;
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(
                ex,
                "Error processing payment. Amount: {Amount}, Merchant: {Merchant}",
                request.Amount,
                request.MerchantId);
                
            throw new PaymentProcessingException("Payment processing failed", ex);
        }
    }
    
    private void ValidatePaymentRequest(PaymentRequest request)
    {
        // Validate amount
        if (request.Amount <= 0)
        {
            throw new ValidationException("Invalid payment amount");
        }
        
        // Validate credit card data
        if (string.IsNullOrEmpty(request.CreditCardNumber) ||
            request.CreditCardNumber.Length < 13 ||
            request.CreditCardNumber.Length > 19)
        {
            throw new ValidationException("Invalid credit card number");
        }
        
        // Additional validations...
    }
    
    private PaymentRequest SanitizeRequest(PaymentRequest request)
    {
        // Create a new request with masked credit card info
        return new PaymentRequest
        {
            MerchantId = request.MerchantId,
            Amount = request.Amount,
            Currency = request.Currency,
            CreditCardNumber = MaskCreditCardNumber(request.CreditCardNumber),
            ExpiryMonth = request.ExpiryMonth,
            ExpiryYear = request.ExpiryYear,
            // Don't include CVV in forwarded request for security
            CardholderName = request.CardholderName,
            OrderReference = request.OrderReference
        };
    }
    
    private string MaskCreditCardNumber(string creditCardNumber)
    {
        // Keep only the last 4 digits
        if (creditCardNumber.Length > 4)
        {
            return new string('*', creditCardNumber.Length - 4) + creditCardNumber.Substring(creditCardNumber.Length - 4);
        }
        
        return creditCardNumber;
    }
    
    private string GetClientIp()
    {
        // In a real application, get from HttpContext
        return "0.0.0.0";
    }
}

public class PaymentProcessingException : Exception
{
    public PaymentProcessingException(string message, Exception innerException) 
        : base(message, innerException)
    {
    }
}

// API controller
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class PaymentsController : ControllerBase
{
    private readonly PaymentGatekeeperService _gatekeeperService;
    
    public PaymentsController(PaymentGatekeeperService gatekeeperService)
    {
        _gatekeeperService = gatekeeperService;
    }
    
    [HttpPost]
    public async Task<ActionResult<PaymentResult>> ProcessPayment(PaymentRequest request)
    {
        // Get auth token from Authorization header
        string authToken = Request.Headers["Authorization"].ToString().Replace("Bearer ", "");
        
        try
        {
            var result = await _gatekeeperService.ProcessPaymentAsync(request, authToken);
            return Ok(result);
        }
        catch (UnauthorizedAccessException)
        {
            return Unauthorized();
        }
        catch (ValidationException ex)
        {
            return BadRequest(ex.Message);
        }
        catch (PaymentProcessingException)
        {
            return StatusCode(500, "Payment processing failed");
        }
    }
}
```

**Key implementation features:**
- Centralized validation
- Request sanitization
- Enhanced logging and monitoring
- Access control enforcement
- Rate limiting and throttling

## Implementation Approaches

### Infrastructure as Code (IaC)

Using Azure Resource Manager (ARM) templates:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "appServicePlanName": {
      "type": "string",
      "metadata": {
        "description": "Name of the App Service Plan"
      }
    },
    "appServicePlanSku": {
      "type": "string",
      "defaultValue": "S1",
      "allowedValues": [
        "F1",
        "D1",
        "B1",
        "B2",
        "B3",
        "S1",
        "S2",
        "S3",
        "P1",
        "P2",
        "P3"
      ],
      "metadata": {
        "description": "App Service Plan SKU"
      }
    },
    "webAppName": {
      "type": "string",
      "metadata": {
        "description": "Name of the Web App"
      }
    },
    "sqlServerName": {
      "type": "string",
      "metadata": {
        "description": "Name of the SQL Server"
      }
    },
    "sqlDatabaseName": {
      "type": "string",
      "metadata": {
        "description": "Name of the SQL Database"
      }
    },
    "sqlAdministratorLogin": {
      "type": "string",
      "metadata": {
        "description": "SQL Server administrator login"
      }
    },
    "sqlAdministratorPassword": {
      "type": "securestring",
      "metadata": {
        "description": "SQL Server administrator password"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]",
      "metadata": {
        "description": "Location for all resources"
      }
    }
  },
  "variables": {
    "appInsightsName": "[concat(parameters('webAppName'), '-insights')]",
    "storageAccountName": "[concat('storage', uniqueString(resourceGroup().id))]"
  },
  "resources": [
    {
      "type": "Microsoft.Web/serverfarms",
      "apiVersion": "2021-02-01",
      "name": "[parameters('appServicePlanName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "[parameters('appServicePlanSku')]"
      },
      "kind": "app",
      "properties": {
        "reserved": false
      }
    },
    {
      "type": "Microsoft.Web/sites",
      "apiVersion": "2021-02-01",
      "name": "[parameters('webAppName')]",
      "location": "[parameters('location')]",
      "kind": "app",
      "dependsOn": [
        "[resourceId('Microsoft.Web/serverfarms', parameters('appServicePlanName'))]",
        "[resourceId('Microsoft.Insights/components', variables('appInsightsName'))]"
      ],
      "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', parameters('appServicePlanName'))]",
        "siteConfig": {
          "netFrameworkVersion": "v6.0",
          "alwaysOn": true,
          "ftpsState": "Disabled",
          "http20Enabled": true,
          "appSettings": [
            {
              "name": "APPINSIGHTS_INSTRUMENTATIONKEY",
              "value": "[reference(resourceId('Microsoft.Insights/components', variables('appInsightsName')), '2020-02-02').InstrumentationKey]"
            },
            {
              "name": "APPLICATIONINSIGHTS_CONNECTION_STRING",
              "value": "[reference(resourceId('Microsoft.Insights/components', variables('appInsightsName')), '2020-02-02').ConnectionString]"
            },
            {
              "name": "ApplicationInsightsAgent_EXTENSION_VERSION",
              "value": "~2"
            },
            {
              "name": "ASPNETCORE_ENVIRONMENT",
              "value": "Production"
            },
            {
              "name": "ConnectionStrings__DefaultConnection",
              "value": "[concat('Server=tcp:', parameters('sqlServerName'), '.database.windows.net,1433;Initial Catalog=', parameters('sqlDatabaseName'), ';Persist Security Info=False;User ID=', parameters('sqlAdministratorLogin'), ';Password=', parameters('sqlAdministratorPassword'), ';MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;')]"
            },
            {
              "name": "StorageConnectionString",
              "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';EndpointSuffix=', environment().suffixes.storage, ';AccountKey=', listKeys(resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName')), '2021-06-01').keys[0].value)]"
            }
          ]
        }
      }
    },
    {
      "type": "Microsoft.Insights/components",
      "apiVersion": "2020-02-02",
      "name": "[variables('appInsightsName')]",
      "location": "[parameters('location')]",
      "kind": "web",
      "properties": {
        "Application_Type": "web",
        "Request_Source": "rest"
      }
    },
    {
      "type": "Microsoft.Sql/servers",
      "apiVersion": "2021-05-01-preview",
      "name": "[parameters('sqlServerName')]",
      "location": "[parameters('location')]",
      "properties": {
        "administratorLogin": "[parameters('sqlAdministratorLogin')]",
        "administratorLoginPassword": "[parameters('sqlAdministratorPassword')]",
        "version": "12.0"
      },
      "resources": [
        {
          "type": "databases",
          "apiVersion": "2021-05-01-preview",
          "name": "[parameters('sqlDatabaseName')]",
          "location": "[parameters('location')]",
          "sku": {
            "name": "Standard",
            "tier": "Standard"
          },
          "dependsOn": [
            "[resourceId('Microsoft.Sql/servers', parameters('sqlServerName'))]"
          ],
          "properties": {
            "collation": "SQL_Latin1_General_CP1_CI_AS",
            "maxSizeBytes": 1073741824
          }
        },
        {
          "type": "firewallRules",
          "apiVersion": "2021-05-01-preview",
          "name": "AllowAllAzureIPs",
          "dependsOn": [
            "[resourceId('Microsoft.Sql/servers', parameters('sqlServerName'))]"
          ],
          "properties": {
            "startIpAddress": "0.0.0.0",
            "endIpAddress": "0.0.0.0"
          }
        }
      ]
    },
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2021-06-01",
      "name": "[variables('storageAccountName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Standard_LRS"
      },
      "kind": "StorageV2",
      "properties": {
        "supportsHttpsTrafficOnly": true,
        "minimumTlsVersion": "TLS1_2",
        "allowBlobPublicAccess": false
      }
    }
  ],
  "outputs": {
    "webAppUrl": {
      "type": "string",
      "value": "[concat('https://', reference(resourceId('Microsoft.Web/sites', parameters('webAppName'))).defaultHostName)]"
    },
    "sqlServerFqdn": {
      "type": "string",
      "value": "[reference(resourceId('Microsoft.Sql/servers', parameters('sqlServerName'))).fullyQualifiedDomainName]"
    },
    "storageAccountName": {
      "type": "string",
      "value": "[variables('storageAccountName')]"
    }
  }
}
```

**Alternative approaches:**
- Azure Bicep
- Terraform (see separate document on IaC and Terraform)
- PowerShell scripts
- Azure CLI scripts

### DevOps Integration

Azure DevOps CI/CD pipeline for .NET applications:

```yaml
# azure-pipelines.yml
name: $(Date:yyyyMMdd)$(Rev:.r)

trigger:
  branches:
    include:
    - main
    - feature/*
  paths:
    include:
    - src/**
    - tests/**
    - azure-pipelines.yml

variables:
  # Base
  solution: '**/*.sln'
  buildPlatform: 'Any CPU'
  buildConfiguration: 'Release'
  
  # Project settings
  projectName: 'MyApp'
  
  # Azure resources
  azureSubscription: 'MyAzureSubscription'
  appServiceName: 'myapp-$(environment)'
  resourceGroupName: 'myapp-$(environment)-rg'
  
  # Docker settings
  dockerRegistry: 'myregistry.azurecr.io'
  dockerImageName: 'myapp'
  dockerImageTag: '$(Build.BuildNumber)'

stages:
- stage: Build
  variables:
    environment: 'dev'
  jobs:
  - job: BuildAndTest
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: UseDotNet@2
      displayName: 'Install .NET SDK'
      inputs:
        packageType: 'sdk'
        version: '6.0.x'
        
    - task: DotNetCoreCLI@2
      displayName: 'Restore NuGet packages'
      inputs:
        command: 'restore'
        projects: '$(solution)'
        feedsToUse: 'select'
        
    - task: DotNetCoreCLI@2
      displayName: 'Build solution'
      inputs:
        command: 'build'
        projects: '$(solution)'
        arguments: '--configuration $(buildConfiguration) --no-restore'
        
    - task: DotNetCoreCLI@2
      displayName: 'Run unit tests'
      inputs:
        command: 'test'
        projects: '**/tests/**/*[Uu]nit[Tt]ests/*.csproj'
        arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"'
        
    - task: DotNetCoreCLI@2
      displayName: 'Run integration tests'
      inputs:
        command: 'test'
        projects: '**/tests/**/*[Ii]ntegration[Tt]ests/*.csproj'
        arguments: '--configuration $(buildConfiguration) --no-build'
        
    - task: DotNetCoreCLI@2
      displayName: 'Publish web app'
      inputs:
        command: 'publish'
        publishWebProjects: true
        arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/app'
        zipAfterPublish: true
        
    - task: Docker@2
      displayName: 'Build Docker image'
      inputs:
        containerRegistry: 'MyDockerRegistry'
        repository: '$(dockerRegistry)/$(dockerImageName)'
        command: 'build'
        Dockerfile: 'src/$(projectName)/Dockerfile'
        buildContext: '$(Build.SourcesDirectory)'
        tags: |
          $(dockerImageTag)
          latest
        
    - task: Docker@2
      displayName: 'Push Docker image'
      inputs:
        containerRegistry: 'MyDockerRegistry'
        repository: '$(dockerRegistry)/$(dockerImageName)'
        command: 'push'
        tags: |
          $(dockerImageTag)
          latest
        
    - publish: $(Build.ArtifactStagingDirectory)/app
      artifact: WebApp
      
    - publish: infrastructure
      artifact: Infrastructure

- stage: DeployToDev
  displayName: 'Deploy to Dev'
  dependsOn: Build
  variables:
    environment: 'dev'
  jobs:
  - deployment: DeployWebApp
    displayName: 'Deploy Web App'
    environment: 'Dev'
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: WebApp
            
          - download: current
            artifact: Infrastructure
            
          - task: AzureResourceManagerTemplateDeployment@3
            displayName: 'Deploy ARM template'
            inputs:
              deploymentScope: 'Resource Group'
              azureResourceManagerConnection: '$(azureSubscription)'
              subscriptionId: '$(subscriptionId)'
              action: 'Create Or Update Resource Group'
              resourceGroupName: '$(resourceGroupName)'
              location: 'East US'
              templateLocation: 'Linked artifact'
              csmFile: '$(Pipeline.Workspace)/Infrastructure/arm-templates/main.json'
              csmParametersFile: '$(Pipeline.Workspace)/Infrastructure/arm-templates/parameters.$(environment).json'
              deploymentMode: 'Incremental'
              deploymentOutputs: 'armOutputs'
              
          - task: AzureRmWebAppDeployment@4
            displayName: 'Deploy to App Service'
            inputs:
              ConnectionType: 'AzureRM'
              azureSubscription: '$(azureSubscription)'
              appType: 'webApp'
              WebAppName: '$(appServiceName)'
              packageForLinux: '$(Pipeline.Workspace)/WebApp/*.zip'
              enableCustomDeployment: true
              DeploymentType: 'zipDeploy'
              TakeAppOfflineFlag: true
              
          - task: AzureAppServiceSettings@1
            displayName: 'Update App Settings'
            inputs:
              azureSubscription: '$(azureSubscription)'
              appName: '$(appServiceName)'
              resourceGroupName: '$(resourceGroupName)'
              appSettings: |
                [
                  {
                    "name": "ASPNETCORE_ENVIRONMENT",
                    "value": "$(environment)",
                    "slotSetting": false
                  },
                  {
                    "name": "ApplicationInsights:InstrumentationKey",
                    "value": "$(appInsightsKey)",
                    "slotSetting": false
                  }
                ]

- stage: DeployToStaging
  displayName: 'Deploy to Staging'
  dependsOn: DeployToDev
  condition: succeeded()
  variables:
    environment: 'staging'
  jobs:
  - deployment: DeployWebApp
    displayName: 'Deploy Web App'
    environment: 'Staging'
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: WebApp
            
          # Similar deployment steps as Dev but for Staging environment
          
- stage: DeployToProduction
  displayName: 'Deploy to Production'
  dependsOn: DeployToStaging
  condition: succeeded()
  variables:
    environment: 'prod'
  jobs:
  - deployment: DeployWebApp
    displayName: 'Deploy Web App'
    environment: 'Production'
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: WebApp
            
          # Production deployment steps with additional approval gates
```

**Key DevOps practices:**
- Continuous integration and delivery
- Automated testing
- Infrastructure as code
- Environment promotion
- Secrets management
- Approval gates

## Best Practices

### Designing for Cloud

- **Design for scale**: Build horizontally scalable services
- **Design for failure**: Assume components will fail, handle gracefully
- **Managed services first**: Use managed services when available
- **Right-sizing**: Choose appropriate service tiers and instances
- **Cost optimization**: Design with costs in mind, optimize resources
- **Security by design**: Implement security at all layers
- **Observable systems**: Comprehensive logging, monitoring, and alerting
- **DevOps integration**: Automate deployment and operations
- **Stateless where possible**: Design services to be stateless for easy scaling
- **Configuration externalization**: Store configuration outside the application

### Resource Naming Conventions

Consistent naming makes resource management easier:

```csharp
// Example resource naming convention class
public static class AzureNamingConvention
{
    // General format: {resource-type}-{environment}-{workload}-{region}-{instance}
    
    public static string GenerateResourceName(
        ResourceType resourceType,
        Environment environment,
        string workload,
        Region region,
        int? instance = null)
    {
        string resourceCode = GetResourceTypeCode(resourceType);
        string environmentCode = GetEnvironmentCode(environment);
        string regionCode = GetRegionCode(region);
        string instanceSuffix = instance.HasValue ? instance.Value.ToString("D2") : "";
        
        return $"{resourceCode}-{environmentCode}-{workload}-{regionCode}{instanceSuffix}".ToLower();
    }
    
    private static string GetResourceTypeCode(ResourceType resourceType)
    {
        return resourceType switch
        {
            ResourceType.AppServicePlan => "plan",
            ResourceType.WebApp => "app",
            ResourceType.FunctionApp => "func",
            ResourceType.SqlDatabase => "sql",
            ResourceType.StorageAccount => "st",
            ResourceType.KeyVault => "kv",
            ResourceType.ApplicationInsights => "ai",
            ResourceType.ServiceBus => "sb",
            ResourceType.CosmosDb => "cosmos",
            ResourceType.ContainerRegistry => "acr",
            ResourceType.AksCluster => "aks",
            ResourceType.VirtualNetwork => "vnet",
            ResourceType.LoadBalancer => "lb",
            ResourceType.ApiManagement => "apim",
            _ => "rsc"
        };
    }
    
    private static string GetEnvironmentCode(Environment environment)
    {
        return environment switch
        {
            Environment.Development => "dev",
            Environment.Testing => "test",
            Environment.Staging => "stage",
            Environment.Production => "prod",
            _ => "env"
        };
    }
    
    private static string GetRegionCode(Region region)
    {
        return region switch
        {
            Region.EastUS => "eus",
            Region.WestUS => "wus",
            Region.NorthEurope => "neu",
            Region.WestEurope => "weu",
            Region.SoutheastAsia => "sea",
            Region.AustraliaEast => "aue",
            _ => "reg"
        };
    }
}

public enum ResourceType
{
    AppServicePlan,
    WebApp,
    FunctionApp,
    SqlDatabase,
    StorageAccount,
    KeyVault,
    ApplicationInsights,
    ServiceBus,
    CosmosDb,
    ContainerRegistry,
    AksCluster,
    VirtualNetwork,
    LoadBalancer,
    ApiManagement
}

public enum Environment
{
    Development,
    Testing,
    Staging,
    Production
}

public enum Region
{
    EastUS,
    WestUS,
    NorthEurope,
    WestEurope,
    SoutheastAsia,
    AustraliaEast
}
```

**Recommended patterns:**
- Abbreviate resource types to stay within length limits
- Use lowercase letters, numbers, and hyphens
- Include environment to distinguish between production/non-production
- Include region to identify resource location
- Include instance number for multiple related resources

### Tagging Strategy

Organize and manage resources with tags:

```csharp
// Example tagging helper class
public static class AzureResourceTags
{
    public static IDictionary<string, string> GetBaseResourceTags(
        string application,
        string component,
        string environment,
        string owner,
        string costCenter)
    {
        return new Dictionary<string, string>
        {
            { "Application", application },
            { "Component", component },
            { "Environment", environment },
            { "Owner", owner },
            { "CostCenter", costCenter },
            { "CreatedDate", DateTime.UtcNow.ToString("yyyy-MM-dd") },
            { "CreatedBy", Environment.UserName }
        };
    }
    
    public static IDictionary<string, string> GetDevOpsResourceTags(
        string application,
        string component,
        string environment,
        string owner,
        string costCenter,
        string buildId,
        string releaseId,
        string repositoryUrl)
    {
        var tags = GetBaseResourceTags(application, component, environment, owner, costCenter);
        
        tags.Add("BuildId", buildId);
        tags.Add("ReleaseId", releaseId);
        tags.Add("RepositoryUrl", repositoryUrl);
        
        return tags;
    }
}

// ARM template tag example
```json
"tags": {
    "Application": "ECommerce",
    "Component": "WebAPI",
    "Environment": "Production",
    "Owner": "retail-team@example.com",
    "CostCenter": "CC-12345",
    "BuildId": "[parameters('buildId')]",
    "ReleaseId": "[parameters('releaseId')]"
}
```

**Key tagging categories:**
- Organization/business unit ownership
- Environment (dev, test, prod)
- Cost center/billing information
- Application name and components
- Security classification
- Automation tags (maintenance windows, backup policies)
- Deployment source (build ID, commit hash)

### Security Best Practices

Implement defense in depth:

- **Use managed identities**: Avoid stored credentials
- **Implement RBAC**: Use role-based access control
- **Encrypt data**: At rest and in transit
- **Network security**: Use private endpoints, NSGs, and WAF
- **Key management**: Use Azure Key Vault for secrets
- **Security monitoring**: Enable Microsoft Defender for Cloud
- **Least privilege**: Grant minimum required permissions
- **Continuous security updates**: Keep services patched
- **DDoS protection**: Enable DDoS Protection Standard
- **Penetration testing**: Regular security assessments

```csharp
// Example of using Azure Key Vault with managed identities
public class SecureConfigurationService
{
    private readonly SecretClient _secretClient;
    
    public SecureConfigurationService(IConfiguration configuration)
    {
        // KeyVault URL from configuration
        string keyVaultUrl = configuration["KeyVault:Url"];
        
        // DefaultAzureCredential supports managed identities
        // and fallback to other authentication methods
        var credential = new DefaultAzureCredential();
        
        _secretClient = new SecretClient(
            new Uri(keyVaultUrl),
            credential);
    }
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        KeyVaultSecret secret = await _secretClient.GetSecretAsync(secretName);
        return secret.Value;
    }
    
    public async Task<Dictionary<string, string>> GetAllSecretsAsync()
    {
        var secrets = new Dictionary<string, string>();
        
        AsyncPageable<SecretProperties> secretProperties = _secretClient.GetPropertiesOfSecretsAsync();
        
        await foreach (SecretProperties secretProperty in secretProperties)
        {
            if (!secretProperty.Enabled.GetValueOrDefault())
                continue;
                
            KeyVaultSecret secret = await _secretClient.GetSecretAsync(secretProperty.Name);
            secrets.Add(secretProperty.Name, secret.Value);
        }
        
        return secrets;
    }
}
```

### Cost Management

Design with cost efficiency:

- **Right-size resources**: Choose appropriate SKUs
- **Auto-scaling**: Scale in/out based on demand
- **Reserved instances**: Pre-pay for predictable workloads
- **Dev/Test pricing**: Use for non-production environments
- **Consumption models**: Use serverless for variable workloads
- **Resource shutdown**: Automate shutdown of non-critical resources
- **Cost allocation**: Use tags for cost tracking
- **Budget alerts**: Set up alerts for budget overruns
- **Tiered storage**: Move less frequently accessed data to lower-cost tiers
- **Cost optimization reviews**: Regular reviews of resources and usage

```csharp
// Automatic resource shutdown for development environments
public class ResourceShutdownService : BackgroundService
{
    private readonly ILogger<ResourceShutdownService> _logger;
    private readonly AzureManagementClient _azureClient;
    private readonly IConfiguration _configuration;
    
    public ResourceShutdownService(
        ILogger<ResourceShutdownService> logger,
        AzureManagementClient azureClient,
        IConfiguration configuration)
    {
        _logger = logger;
        _azureClient = azureClient;
        _configuration = configuration;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Only run in development environments
        string environment = _configuration["Environment"];
        if (environment != "Development" && environment != "Test")
        {
            _logger.LogInformation("Resource shutdown service only runs in Development or Test environments");
            return;
        }
        
        // Run once at startup, then every day
        await ShutdownResourcesAsync();
        
        using PeriodicTimer timer = new PeriodicTimer(TimeSpan.FromHours(24));
        
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            await ShutdownResourcesAsync();
        }
    }
    
    private async Task ShutdownResourcesAsync()
    {
        _logger.LogInformation("Checking for resources to shut down");
        
        // Get current time in the target timezone
        var targetTimeZone = TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time");
        var currentTime = TimeZoneInfo.ConvertTimeFromUtc(DateTime.UtcNow, targetTimeZone);
        
        // Check if it's after working hours (after 7 PM)
        if (currentTime.Hour < 19)
        {
            _logger.LogInformation("Not after hours yet, skipping shutdown");
            return;
        }
        
        try
        {
            // Get resources to shut down
            var resourceGroupName = _configuration["ResourceGroup:Name"];
            
            // Shut down VM resources
            _logger.LogInformation("Shutting down VM resources in {ResourceGroup}", resourceGroupName);
            await _azureClient.ShutdownVirtualMachinesAsync(resourceGroupName);
            
            // Scale down App Service Plans
            _logger.LogInformation("Scaling down App Service Plans in {ResourceGroup}", resourceGroupName);
            await _azureClient.ScaleDownAppServicePlansAsync(resourceGroupName);
            
            _logger.LogInformation("Resource shutdown completed successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error shutting down resources");
        }
    }
}
```

## Common Pitfalls

### Excessive Service Coupling

**Problem**: Tightly coupled services create cascading failures and deployment challenges.

**Symptoms**:
- Changes to one service require changes to multiple services
- Service failures cascade to other services
- Deployment requires coordinated releases

**Solutions**:
- Design services with clear boundaries
- Use asynchronous communication patterns
- Implement resilience patterns (circuit breakers, retries)
- Version APIs and implement backward compatibility
- Use service discovery and load balancing

### Improper Data Partitioning

**Problem**: Poor data partitioning leads to performance bottlenecks and scaling issues.

**Symptoms**:
- Hotspots in storage
- Throttling and request rate limits
- Increased latency
- Inability to scale horizontally

**Solutions**:
- Choose appropriate partition keys
- Distribute workloads evenly
- Use sharding for large datasets
- Consider fan-out query patterns
- Implement caching strategies

### Ignoring Resource Limits

**Problem**: Not accounting for Azure service limits leads to application failures.

**Symptoms**:
- Throttling errors
- Failed operations
- Inconsistent performance
- Increased latency

**Solutions**:
- Document and monitor service limits
- Implement retry policies with exponential backoff
- Design for scale partitioning
- Use queue-based load leveling
- Consider premium tiers for higher limits

### Security Misconfiguration

**Problem**: Improper security configuration exposes resources to unnecessary risk.

**Symptoms**:
- Publicly accessible resources
- Overly permissive access control
- Security alerts and scan findings
- Compliance violations

**Solutions**:
- Use security baselines and benchmarks
- Implement security as code
- Regular security assessments
- Enable Microsoft Defender for Cloud
- Follow the principle of least privilege

### Poor Monitoring and Alerting

**Problem**: Insufficient monitoring leads to undetected issues and delayed response.

**Symptoms**:
- Delayed issue detection
- Inability to diagnose root causes
- User-reported outages
- Recurring issues

**Solutions**:
- Implement comprehensive monitoring
- Set up proactive alerts
- Use Application Insights for application monitoring
- Implement distributed tracing
- Create operational dashboards

## Cloud Architecture Decision Framework

When designing Azure solutions, consider these decision factors:

### Service Selection Criteria

When choosing between similar Azure services:

1. **Operational Complexity**: Managed services reduce operational overhead
2. **Scaling Requirements**: Consider both vertical and horizontal scaling needs
3. **Feature Requirements**: Match service capabilities to functional requirements
4. **Integration Needs**: Consider integration with existing systems
5. **Cost Structure**: Compare pricing models (consumption vs. provisioned)
6. **Compliance Requirements**: Ensure services meet regulatory requirements
7. **Performance Characteristics**: Consider latency, throughput, and SLAs
8. **Team Skills**: Consider your team's familiarity with the service

### Compute Service Decision Matrix

| Consideration | App Service | Azure Functions | Azure Container Apps | AKS |
|---------------|-------------|-----------------|---------------------|-----|
| Complexity    | Low         | Low             | Medium              | High |
| Control       | Limited     | Limited         | Moderate            | Full |
| Scaling       | Automatic   | Automatic       | Automatic           | Manual/Auto |
| Density       | Medium      | High            | High                | High |
| Cold Start    | No          | Yes (Consumption) | Possible           | No |
| OS Support    | Windows/Linux | Windows/Linux  | Linux               | Windows/Linux |
| Pricing Model | Instance    | Consumption/Premium | Consumption      | Cluster + VMs |
| Best For      | Web apps, APIs | Event processing | Containerized apps | Complex microservices |

### Data Store Decision Matrix

| Consideration | Azure SQL | Cosmos DB | Table Storage | Blob Storage |
|---------------|-----------|-----------|---------------|--------------|
| Data Structure| Relational | Document/Graph/Key-Value | Key-Value | Unstructured |
| Schema        | Fixed     | Flexible   | Flexible      | None        |
| Query Capabilities | Rich SQL | SQL/MongoDB/Gremlin/Cassandra | Basic | Minimal |
| Transactions  | Full ACID | Multi-document | Entity Group | None |
| Scaling       | Vertical/Horizontal | Horizontal | Horizontal | Horizontal |
| Global Distribution | Manual | Automatic | Manual | Manual |
| Cost          | Moderate to High | Moderate to High | Low | Low |
| Best For      | Relational data | Global, schema-less | Simple structured data | Files, media |

## Azure Cloud Reference Architecture

### Web Application Pattern

A reference architecture for a scalable web application:

```
                        ┌─────────────────┐
                        │  Azure Front Door│
                        │   (Global CDN)   │
                        └─────────┬───────┘
                                  │
                                  ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  Application    │      │  App Service    │      │  Azure SQL      │
│  Gateway WAF    │─────▶│  (Web App)      │─────▶│  Database       │
└─────────────────┘      └─────────┬───────┘      └─────────────────┘
                                  │
                                  │
                 ┌────────────────┼────────────────┐
                 │                │                │
                 ▼                ▼                ▼
        ┌─────────────────┐┌─────────────────┐┌─────────────────┐
        │  Azure Storage  ││  Azure Redis    ││  Azure Service  │
        │  (Blob Storage) ││  Cache          ││  Bus            │
        └─────────────────┘└─────────────────┘└─────────────────┘
                                                      │
                                                      │
                                                      ▼
                                            ┌─────────────────┐
                                            │  Azure Functions│
                                            │  (Background    │
                                            │   Processing)   │
                                            └─────────┬───────┘
                                                      │
                                                      │
                                                      ▼
                                            ┌─────────────────┐
                                            │  Azure Cosmos DB│
                                            │  (Analytics)    │
                                            └─────────────────┘
```

**Key components:**
- **Azure Front Door**: Global CDN and load balancer
- **Application Gateway**: WAF and regional routing
- **App Service**: Hosting web application
- **Azure SQL Database**: Primary data store
- **Azure Redis Cache**: Session state and caching
- **Azure Storage**: Static content and file storage
- **Azure Service Bus**: Message queue for reliable processing
- **Azure Functions**: Background processing
- **Azure Cosmos DB**: Analytics and reporting data

### Microservices Architecture Pattern

A reference architecture for microservices on Azure:

```
┌────────────────────────────────────────────────────────────────────┐
│                    Azure Kubernetes Service                         │
│                                                                     │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌──────────┐│
│   │  API Gateway│   │ Service A   │   │ Service B   │   │Service C ││
│   │             │   │             │   │             │   │          ││
│   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └────┬─────┘│
│          │                 │                 │                │     │
└──────────┼─────────────────┼─────────────────┼────────────────┼─────┘
           │                 │                 │                │     
           ▼                 │                 │                │     
┌─────────────────┐          │                 │                │     
│  Azure API      │          │                 │                │     
│  Management     │          │                 │                │     
└─────────┬───────┘          │                 │                │     
          │                  │                 │                │     
          ▼                  │                 │                │     
┌─────────────────┐          │                 │                │     
│  Application    │          │                 │                │     
│  Gateway        │          │                 │                │     
└─────────────────┘          │                 │                │     
                             │                 │                │     
                             ▼                 ▼                ▼     
                    ┌─────────────────┐┌─────────────────┐┌──────────┐
                    │  SQL Database A ││  Cosmos DB B    ││ Storage C│
                    │                 ││                 ││          │
                    └─────────────────┘└─────────────────┘└──────────┘
                             │                 │              
                             │                 │              
                             ▼                 ▼              
                    ┌─────────────────────────────────────┐   
                    │       Event Grid / Service Bus      │   
                    │                                     │   
                    └──────────────────┬──────────────────┘   
                                       │                      
                                       ▼                      
                              ┌─────────────────┐             
                              │  Azure Functions│             
                              │  (Event         │             
                              │   Processing)   │             
                              └─────────────────┘             
```

**Key components:**
- **AKS**: Container orchestration for microservices
- **API Management**: API gateway and management
- **Application Gateway**: WAF and routing
- **Azure SQL Database & Cosmos DB**: Service-specific databases
- **Event Grid/Service Bus**: Event-driven communication
- **Azure Functions**: Event processing and integration
- **Azure Monitor/App Insights**: Monitoring and observability

## Further Reading

- [Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/)
- [Cloud Design Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/)
- [Azure Well-Architected Framework](https://docs.microsoft.com/en-us/azure/architecture/framework/)
- [.NET Microservices: Architecture for Containerized .NET Applications](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/)
- [Architecting Cloud-Native .NET Applications for Azure](https://docs.microsoft.com/en-us/dotnet/architecture/cloud-native/)

## Related Topics

- [Infrastructure as Code with Terraform](../iac/terraform/introduction.md)
- [Containerization with Docker](../../containers/docker/dotnet-containers.md)
- [Kubernetes for .NET Applications](../../containers/kubernetes/kubernetes-basics.md)
- [CI/CD for Cloud Applications](../../devops/cicd-pipelines.md)
- [Microservices Architecture](../../architecture/microservices/introduction.md)