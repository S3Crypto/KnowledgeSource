---
title: "Relational Databases for .NET Applications"
date_created: 2023-04-02
date_updated: 2023-04-02
authors: ["Repository Maintainers"]
tags: ["database", "relational", "sql", "entity-framework", "dapper"]
difficulty: "intermediate"
---

# Relational Databases for .NET Applications

## Overview

Relational databases remain the cornerstone of data persistence for most enterprise applications. For .NET developers, the Microsoft ecosystem offers excellent integration with various relational database systems, particularly SQL Server, but also supports other popular options like PostgreSQL, MySQL, and Oracle. This document explores relational database technologies, access patterns, optimization techniques, and best practices for .NET applications.

## Core Concepts

### Relational Database Fundamentals

Relational databases organize data into tables with rows and columns, establishing relationships between entities through foreign keys. Key concepts include:

- **Tables**: Collections of related data organized in rows and columns
- **Primary Keys**: Unique identifiers for each row in a table
- **Foreign Keys**: References to primary keys in other tables, establishing relationships
- **Normalization**: Organizing data to reduce redundancy and improve data integrity
- **Transactions**: Groups of operations that execute as a single unit, maintaining ACID properties
- **Indexing**: Data structures that improve query performance

### Common Relational Databases for .NET Applications

#### Microsoft SQL Server

SQL Server is the most common choice for .NET applications due to its tight integration with the Microsoft stack:

- **Advantages**: Native .NET support, excellent tooling, integration with Visual Studio and Azure
- **Editions**: Express (free), Standard, Enterprise, Azure SQL Database (cloud)
- **Use cases**: Enterprise applications, data warehousing, BI solutions

#### PostgreSQL

An open-source RDBMS known for standards compliance and advanced features:

- **Advantages**: Free, open-source, excellent for complex data and geographical data
- **Use cases**: Applications requiring advanced data types, geospatial applications

#### MySQL/MariaDB

Popular open-source options with wide community support:

- **Advantages**: Free, widely supported, lightweight
- **Use cases**: Web applications, content management systems

#### SQLite

A file-based, embedded database engine:

- **Advantages**: Zero configuration, self-contained, portable
- **Use cases**: Mobile applications, desktop applications, testing

## .NET Data Access Technologies

### Entity Framework Core

EF Core is Microsoft's recommended ORM (Object-Relational Mapper) for .NET applications:

```csharp
// Define a model
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public List<Order> Orders { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public DateTime OrderDate { get; set; }
    public decimal TotalAmount { get; set; }
    public int CustomerId { get; set; }
    public Customer Customer { get; set; }
}

// Define a DbContext
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    { }
    
    public DbSet<Customer> Customers { get; set; }
    public DbSet<Order> Orders { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Configure relationships, indexes, constraints
        modelBuilder.Entity<Customer>()
            .HasIndex(c => c.Email)
            .IsUnique();
            
        modelBuilder.Entity<Order>()
            .HasOne(o => o.Customer)
            .WithMany(c => c.Orders)
            .HasForeignKey(o => o.CustomerId);
    }
}

// Register in DI (ASP.NET Core)
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));

// Query example
public async Task<List<Customer>> GetCustomersWithRecentOrdersAsync()
{
    return await _context.Customers
        .Include(c => c.Orders.Where(o => o.OrderDate >= DateTime.Now.AddDays(-30)))
        .ToListAsync();
}
```

Key EF Core features:
- **LINQ Integration**: Querying using LINQ expressions
- **Change Tracking**: Automatic tracking of entity changes
- **Migrations**: Versioned database schema changes
- **Lazy/Eager Loading**: Control over relationship loading
- **Multiple Database Providers**: Support for various database engines

### Dapper

Dapper is a lightweight ORM that focuses on performance:

```csharp
// Basic query with Dapper
public async Task<IEnumerable<Customer>> GetCustomersAsync()
{
    using var connection = new SqlConnection(_connectionString);
    return await connection.QueryAsync<Customer>("SELECT Id, Name, Email FROM Customers");
}

// Parameterized query
public async Task<Customer> GetCustomerByIdAsync(int id)
{
    using var connection = new SqlConnection(_connectionString);
    return await connection.QuerySingleOrDefaultAsync<Customer>(
        "SELECT Id, Name, Email FROM Customers WHERE Id = @Id",
        new { Id = id });
}

// Multiple result sets
public async Task<CustomerOrdersDto> GetCustomerWithOrdersAsync(int customerId)
{
    using var connection = new SqlConnection(_connectionString);
    
    using var multi = await connection.QueryMultipleAsync(
        @"SELECT Id, Name, Email FROM Customers WHERE Id = @Id;
          SELECT Id, OrderDate, TotalAmount FROM Orders WHERE CustomerId = @Id",
        new { Id = customerId });
    
    var customer = await multi.ReadSingleOrDefaultAsync<Customer>();
    var orders = await multi.ReadAsync<Order>();
    
    return new CustomerOrdersDto
    {
        Customer = customer,
        Orders = orders.ToList()
    };
}
```

Key Dapper features:
- **High Performance**: Minimal overhead compared to EF Core
- **SQL Control**: Direct usage of SQL queries
- **Multi-Mapping**: Support for complex result mapping
- **Bulk Operations**: Efficient handling of batch operations

### ADO.NET

The lowest level of database access, providing direct control:

```csharp
public async Task<List<Customer>> GetCustomersAsync()
{
    var customers = new List<Customer>();
    
    using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();
    
    using var command = new SqlCommand("SELECT Id, Name, Email FROM Customers", connection);
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
```

Key ADO.NET features:
- **Maximum Control**: Direct access to database features
- **Performance**: Minimal abstraction overhead
- **Batching**: Manual control over batching
- **Advanced Features**: Access to database-specific capabilities

## Database Design Patterns

### Repository Pattern

Abstracts the data access layer, promoting cleaner architecture:

```csharp
// Repository interface
public interface ICustomerRepository
{
    Task<Customer> GetByIdAsync(int id);
    Task<IEnumerable<Customer>> GetAllAsync();
    Task AddAsync(Customer customer);
    Task UpdateAsync(Customer customer);
    Task DeleteAsync(int id);
}

// EF Core implementation
public class CustomerRepository : ICustomerRepository
{
    private readonly ApplicationDbContext _context;
    
    public CustomerRepository(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public async Task<Customer> GetByIdAsync(int id)
    {
        return await _context.Customers
            .Include(c => c.Orders)
            .SingleOrDefaultAsync(c => c.Id == id);
    }
    
    public async Task<IEnumerable<Customer>> GetAllAsync()
    {
        return await _context.Customers.ToListAsync();
    }
    
    public async Task AddAsync(Customer customer)
    {
        await _context.Customers.AddAsync(customer);
        await _context.SaveChangesAsync();
    }
    
    public async Task UpdateAsync(Customer customer)
    {
        _context.Customers.Update(customer);
        await _context.SaveChangesAsync();
    }
    
    public async Task DeleteAsync(int id)
    {
        var customer = await _context.Customers.FindAsync(id);
        if (customer != null)
        {
            _context.Customers.Remove(customer);
            await _context.SaveChangesAsync();
        }
    }
}
```

### Unit of Work Pattern

Coordinates operations across multiple repositories:

```csharp
public interface IUnitOfWork : IDisposable
{
    ICustomerRepository Customers { get; }
    IOrderRepository Orders { get; }
    Task<int> SaveChangesAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private ICustomerRepository _customerRepository;
    private IOrderRepository _orderRepository;
    
    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public ICustomerRepository Customers => 
        _customerRepository ??= new CustomerRepository(_context);
        
    public IOrderRepository Orders => 
        _orderRepository ??= new OrderRepository(_context);
    
    public async Task<int> SaveChangesAsync()
    {
        return await _context.SaveChangesAsync();
    }
    
    public void Dispose()
    {
        _context.Dispose();
    }
}

// Usage
public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;
    
    public OrderService(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }
    
    public async Task CreateOrderAsync(Order order)
    {
        // Verify customer exists
        var customer = await _unitOfWork.Customers.GetByIdAsync(order.CustomerId);
        if (customer == null)
            throw new ArgumentException("Customer not found");
        
        // Add order
        await _unitOfWork.Orders.AddAsync(order);
        
        // Update customer statistics
        customer.LastOrderDate = order.OrderDate;
        customer.TotalOrders++;
        await _unitOfWork.Customers.UpdateAsync(customer);
        
        // Save all changes in a single transaction
        await _unitOfWork.SaveChangesAsync();
    }
}
```

### Query Object Pattern

Encapsulates query logic in specialized objects:

```csharp
public class CustomerQuery
{
    public string NameFilter { get; set; }
    public DateTime? OrderedSince { get; set; }
    public decimal? MinTotalSpent { get; set; }
    public int? Page { get; set; }
    public int? PageSize { get; set; }
}

public class CustomerQueryHandler
{
    private readonly ApplicationDbContext _context;
    
    public CustomerQueryHandler(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public async Task<(IEnumerable<Customer> Customers, int TotalCount)> ExecuteAsync(CustomerQuery query)
    {
        IQueryable<Customer> customersQuery = _context.Customers
            .Include(c => c.Orders);
        
        // Apply filters
        if (!string.IsNullOrWhiteSpace(query.NameFilter))
        {
            customersQuery = customersQuery.Where(c => 
                c.Name.Contains(query.NameFilter));
        }
        
        if (query.OrderedSince.HasValue)
        {
            customersQuery = customersQuery.Where(c => 
                c.Orders.Any(o => o.OrderDate >= query.OrderedSince));
        }
        
        if (query.MinTotalSpent.HasValue)
        {
            customersQuery = customersQuery.Where(c => 
                c.Orders.Sum(o => o.TotalAmount) >= query.MinTotalSpent);
        }
        
        // Get total count before pagination
        int totalCount = await customersQuery.CountAsync();
        
        // Apply pagination
        if (query.Page.HasValue && query.PageSize.HasValue)
        {
            int skip = (query.Page.Value - 1) * query.PageSize.Value;
            customersQuery = customersQuery
                .Skip(skip)
                .Take(query.PageSize.Value);
        }
        
        return (await customersQuery.ToListAsync(), totalCount);
    }
}
```

## Performance Optimization

### Query Optimization

Strategies for optimizing database queries:

1. **Use Indexes Effectively**:
   ```csharp
   // In EF Core OnModelCreating method
   modelBuilder.Entity<Customer>()
       .HasIndex(c => c.Email);  // Single-column index
       
   modelBuilder.Entity<Order>()
       .HasIndex(o => new { o.CustomerId, o.OrderDate });  // Composite index
   ```

2. **Select Only Required Columns**:
   ```csharp
   // Select specific columns with Dapper
   var customers = await connection.QueryAsync<CustomerDto>(
       "SELECT Id, Name FROM Customers");
       
   // Select specific columns with EF Core
   var customers = await _context.Customers
       .Select(c => new CustomerDto { Id = c.Id, Name = c.Name })
       .ToListAsync();
   ```

3. **Use Pagination**:
   ```csharp
   // EF Core pagination
   var pageNumber = 2;
   var pageSize = 20;
   var customers = await _context.Customers
       .Skip((pageNumber - 1) * pageSize)
       .Take(pageSize)
       .ToListAsync();
   ```

4. **Optimize Relationship Loading**:
   ```csharp
   // Explicit loading in EF Core
   var customer = await _context.Customers.FindAsync(id);
   
   // Load only specific related data
   await _context.Entry(customer)
       .Collection(c => c.Orders)
       .Query()
       .Where(o => o.OrderDate >= DateTime.Now.AddDays(-30))
       .LoadAsync();
   ```

5. **Use SQL Server-specific features** (when appropriate):
   ```csharp
   // Use table hints
   var customers = await _context.Customers
       .FromSqlRaw("SELECT * FROM Customers WITH (NOLOCK)")
       .ToListAsync();
   ```

### Connection Management

Best practices for database connections:

1. **Connection Pooling**:
   ```csharp
   // Connection string with pooling settings
   "Server=myServerAddress;Database=myDataBase;User Id=myUsername;Password=myPassword;Max Pool Size=100;Min Pool Size=5;"
   ```

2. **Connection Resilience**:
   ```csharp
   // Enable connection resiliency in EF Core
   services.AddDbContext<ApplicationDbContext>(options =>
       options.UseSqlServer(connectionString, 
           sqlOptions => sqlOptions.EnableRetryOnFailure(
               maxRetryCount: 5,
               maxRetryDelay: TimeSpan.FromSeconds(30),
               errorNumbersToAdd: null)));
   ```

3. **Async/Await Best Practices**:
   ```csharp
   // Proper async usage
   public async Task<List<Customer>> GetCustomersAsync()
   {
       using var connection = new SqlConnection(_connectionString);
       await connection.OpenAsync();
       
       return (await connection.QueryAsync<Customer>(
           "SELECT * FROM Customers")).ToList();
   }
   ```

### Caching Strategies

Implementing caching to reduce database load:

1. **In-Memory Caching**:
   ```csharp
   // In-memory caching in ASP.NET Core
   public async Task<Customer> GetCustomerByIdAsync(int id)
   {
       string cacheKey = $"customer-{id}";
       
       if (!_memoryCache.TryGetValue(cacheKey, out Customer customer))
       {
           customer = await _dbContext.Customers.FindAsync(id);
           
           var cacheOptions = new MemoryCacheEntryOptions()
               .SetAbsoluteExpiration(TimeSpan.FromMinutes(10));
               
           _memoryCache.Set(cacheKey, customer, cacheOptions);
       }
       
       return customer;
   }
   ```

2. **Distributed Caching**:
   ```csharp
   // Redis caching in ASP.NET Core
   public async Task<List<Product>> GetFeaturedProductsAsync()
   {
       string cacheKey = "featured-products";
       
       var productsBytes = await _distributedCache.GetAsync(cacheKey);
       if (productsBytes != null)
       {
           return JsonSerializer.Deserialize<List<Product>>(productsBytes);
       }
       
       var products = await _dbContext.Products
           .Where(p => p.IsFeatured)
           .ToListAsync();
           
       var serializedProducts = JsonSerializer.SerializeToUtf8Bytes(products);
       
       await _distributedCache.SetAsync(cacheKey, serializedProducts, 
           new DistributedCacheEntryOptions
           {
               AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
           });
           
       return products;
   }
   ```

3. **Second-Level Caching in EF Core**:
   ```csharp
   // Using EF Core second-level cache extension
   // (Requires a third-party package like EFCoreSecondLevelCacheInterceptor)
   services.AddDbContext<ApplicationDbContext>(options => 
       options.UseSqlServer(connectionString)
              .AddInterceptors(new SecondLevelCacheInterceptor()));
   
   // Query with caching
   var products = await _dbContext.Products
       .Where(p => p.CategoryId == categoryId)
       .Cacheable(CacheExpirationMode.Sliding, TimeSpan.FromMinutes(10))
       .ToListAsync();
   ```

## Entity Framework Core Best Practices

### Migration Strategies

Managing database schema changes:

1. **Creating Migrations**:
   ```bash
   # Create a new migration
   dotnet ef migrations add AddCustomerLoyaltyPoints
   
   # Apply migrations to the database
   dotnet ef database update
   ```

2. **Scripting Migrations**:
   ```bash
   # Generate SQL script for applying migrations
   dotnet ef migrations script 0 AddCustomerLoyaltyPoints
   ```

3. **Migration in Production**:
   ```csharp
   // Applying migrations programmatically in ASP.NET Core
   public class Program
   {
       public static void Main(string[] args)
       {
           var host = CreateHostBuilder(args).Build();
           
           using (var scope = host.Services.CreateScope())
           {
               var services = scope.ServiceProvider;
               var dbContext = services.GetRequiredService<ApplicationDbContext>();
               dbContext.Database.Migrate();
           }
           
           host.Run();
       }
   }
   ```

### Performance Tuning

Optimizing EF Core performance:

1. **Compiled Queries**:
   ```csharp
   // Define compiled query
   private static readonly Func<ApplicationDbContext, int, Task<Customer>> GetCustomerById =
       EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
           context.Customers
               .Include(c => c.Orders)
               .FirstOrDefault(c => c.Id == id));
   
   // Use compiled query
   public async Task<Customer> GetCustomerByIdAsync(int id)
   {
       return await GetCustomerById(_dbContext, id);
   }
   ```

2. **Bulk Operations**:
   ```csharp
   // Using EFCore.BulkExtensions package
   public async Task BulkInsertCustomersAsync(List<Customer> customers)
   {
       await _dbContext.BulkInsertAsync(customers);
   }
   ```

3. **No-Tracking Queries**:
   ```csharp
   // Read-only query with no tracking
   var products = await _dbContext.Products
       .AsNoTracking()
       .Where(p => p.Category == "Electronics")
       .ToListAsync();
   ```

4. **Selective Loading**:
   ```csharp
   // Split queries for large result sets with many related entities
   var customers = await _dbContext.Customers
       .Include(c => c.Orders)
       .AsSplitQuery()
       .ToListAsync();
   ```

## Security Considerations

### SQL Injection Prevention

Protecting against SQL injection:

1. **Parameterized Queries**:
   ```csharp
   // Dapper - safe parameterized query
   var customer = await connection.QuerySingleOrDefaultAsync<Customer>(
       "SELECT * FROM Customers WHERE Email = @Email",
       new { Email = email });
   
   // EF Core - automatic parameterization
   var customer = await _dbContext.Customers
       .FirstOrDefaultAsync(c => c.Email == email);
   
   // ADO.NET - parameterized query
   using var command = new SqlCommand(
       "SELECT * FROM Customers WHERE Email = @Email", connection);
   command.Parameters.AddWithValue("@Email", email);
   ```

2. **Avoiding Raw SQL** (when possible):
   ```csharp
   // BAD - SQL injection vulnerability
   var name = "Robert'; DROP TABLE Customers; --";
   var query = $"SELECT * FROM Customers WHERE Name = '{name}'";
   
   // GOOD - EF Core's parameterization
   var customers = await _dbContext.Customers
       .Where(c => c.Name == name)
       .ToListAsync();
   ```

### Data Encryption

Protecting sensitive data:

1. **Column Encryption**:
   ```sql
   -- SQL Server column encryption
   CREATE TABLE Customers (
       Id INT PRIMARY KEY,
       Name NVARCHAR(100),
       CreditCardNumber VARBINARY(128) ENCRYPTED WITH (
           COLUMN_ENCRYPTION_KEY = [MyCEK],
           ENCRYPTION_TYPE = DETERMINISTIC,
           ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
       )
   )
   ```

2. **Application-Level Encryption**:
   ```csharp
   // Encrypt credit card number before storing
   public async Task SaveCreditCardAsync(string cardNumber, int customerId)
   {
       // Encrypt sensitive data
       byte[] encryptedCardNumber = _encryptionService.Encrypt(cardNumber);
       
       var paymentInfo = new PaymentInfo
       {
           CustomerId = customerId,
           EncryptedCardNumber = encryptedCardNumber
       };
       
       await _dbContext.PaymentInfos.AddAsync(paymentInfo);
       await _dbContext.SaveChangesAsync();
   }
   ```

3. **Transport Encryption**:
   ```
   // Connection string with encryption enabled
   "Server=myServer;Database=myDb;User Id=myUser;Password=myPassword;Encrypt=True;TrustServerCertificate=False;"
   ```

### Authorization and Access Control

Implementing data access control:

1. **Row-Level Security**:
   ```sql
   -- SQL Server row-level security
   CREATE SCHEMA Security;
   GO
   
   CREATE FUNCTION Security.fn_securitypredicate(@TenantId AS INT)
       RETURNS TABLE
   WITH SCHEMABINDING
   AS
       RETURN SELECT 1 AS fn_securitypredicate_result
       WHERE DATABASE_PRINCIPAL_ID() = DATABASE_PRINCIPAL_ID('dbo') -- DBAs can see everything
       OR @TenantId = CONVERT(INT, SESSION_CONTEXT(N'TenantId'));
   GO
   
   CREATE SECURITY POLICY Security.TenantFilter
   ADD FILTER PREDICATE Security.fn_securitypredicate(TenantId)
   ON dbo.Customers;
   ```

2. **Application-Level Filtering**:
   ```csharp
   // Tenant filtering in repository
   public class MultiTenantRepository<T> where T : class, ITenantEntity
   {
       private readonly ApplicationDbContext _context;
       private readonly ITenantProvider _tenantProvider;
       
       public MultiTenantRepository(
           ApplicationDbContext context,
           ITenantProvider tenantProvider)
       {
           _context = context;
           _tenantProvider = tenantProvider;
       }
       
       public IQueryable<T> GetAll()
       {
           return _context.Set<T>()
               .Where(e => e.TenantId == _tenantProvider.GetTenantId());
       }
   }
   ```

## Best Practices

### General Database Best Practices

1. **Use Database Transactions**:
   ```csharp
   public async Task TransferFundsAsync(int fromAccountId, int toAccountId, decimal amount)
   {
       using var transaction = await _dbContext.Database.BeginTransactionAsync();
       try
       {
           var fromAccount = await _dbContext.Accounts.FindAsync(fromAccountId);
           var toAccount = await _dbContext.Accounts.FindAsync(toAccountId);
           
           if (fromAccount.Balance < amount)
               throw new InvalidOperationException("Insufficient funds");
               
           fromAccount.Balance -= amount;
           toAccount.Balance += amount;
           
           await _dbContext.SaveChangesAsync();
           
           // Record the transaction
           _dbContext.TransactionHistory.Add(new TransactionRecord
           {
               FromAccountId = fromAccountId,
               ToAccountId = toAccountId,
               Amount = amount,
               Date = DateTime.UtcNow
           });
           
           await _dbContext.SaveChangesAsync();
           await transaction.CommitAsync();
       }
       catch (Exception)
       {
           await transaction.RollbackAsync();
           throw;
       }
   }
   ```

2. **Defensive Database Access**:
   ```csharp
   public async Task<Customer> GetCustomerByIdAsync(int id)
   {
       try
       {
           var customer = await _dbContext.Customers.FindAsync(id);
           
           if (customer == null)
               throw new NotFoundException($"Customer with ID {id} not found");
               
           return customer;
       }
       catch (SqlException ex)
       {
           _logger.LogError(ex, "Database error while retrieving customer {CustomerId}", id);
           throw new DatabaseAccessException("Failed to retrieve customer data", ex);
       }
   }
   ```

3. **Database Versioning**:
   ```csharp
   // Setting up database schema version tracking
   public static class DatabaseVersioning
   {
       public static void Initialize(IApplicationBuilder app)
       {
           using var serviceScope = app.ApplicationServices.CreateScope();
           var context = serviceScope.ServiceProvider.GetService<ApplicationDbContext>();
           
           // Ensure SchemaVersions table exists
           if (!context.SchemaVersions.Any())
           {
               context.SchemaVersions.Add(new SchemaVersion
               {
                   Version = "1.0.0",
                   AppliedOn = DateTime.UtcNow,
                   Description = "Initial schema"
               });
               
               context.SaveChanges();
           }
       }
   }
   ```

### Microservice Database Patterns

1. **Database Per Service**:
   ```csharp
   // OrderService database context
   public class OrderDbContext : DbContext
   {
       public OrderDbContext(DbContextOptions<OrderDbContext> options)
           : base(options)
       { }
       
       public DbSet<Order> Orders { get; set; }
       public DbSet<OrderItem> OrderItems { get; set; }
       
       // No Customer table - that belongs to CustomerService
   }
   
   // CustomerService database context
   public class CustomerDbContext : DbContext
   {
       public CustomerDbContext(DbContextOptions<CustomerDbContext> options)
           : base(options)
       { }
       
       public DbSet<Customer> Customers { get; set; }
       public DbSet<CustomerAddress> CustomerAddresses { get; set; }
   }
   ```

2. **Saga Pattern for Distributed Transactions**:
   ```csharp
   // Simplified saga implementation
   public class OrderCreationSaga : ISaga
   {
       private readonly IOrderService _orderService;
       private readonly IPaymentService _paymentService;
       private readonly IInventoryService _inventoryService;
       private readonly INotificationService _notificationService;
       
       // Steps in the saga
       public async Task<SagaResult> ExecuteAsync(OrderCreationSagaData data)
       {
           try
           {
               // Step 1: Create order (local transaction)
               var orderId = await _orderService.CreateOrderAsync(data.Order);
               data.OrderId = orderId;
               
               // Step 2: Process payment
               var paymentResult = await _paymentService.ProcessPaymentAsync(
                   data.PaymentInfo, data.OrderId);
                   
               if (!paymentResult.Success)
               {
                   // Compensating transaction
                   await _orderService.CancelOrderAsync(data.OrderId);
                   return SagaResult.Failed("Payment failed");
               }
               
               // Step 3: Reserve inventory
               var inventoryResult = await _inventoryService.ReserveInventoryAsync(
                   data.OrderId, data.Items);
                   
               if (!inventoryResult.Success)
               {
                   // Compensating transactions
                   await _paymentService.RefundPaymentAsync(data.OrderId);
                   await _orderService.CancelOrderAsync(data.OrderId);
                   return SagaResult.Failed("Inventory reservation failed");
               }
               
               // Step 4: Send notification
               await _notificationService.SendOrderConfirmationAsync(data.OrderId);
               
               return SagaResult.Succeeded();
           }
           catch (Exception ex)
           {
               // Handle unexpected failures with compensating transactions
               if (data.OrderId > 0)
               {
                   await _paymentService.RefundPaymentAsync(data.OrderId);
                   await _inventoryService.ReleaseInventoryAsync(data.OrderId);
                   await _orderService.CancelOrderAsync(data.OrderId);
               }
               
               return SagaResult.Failed(ex.Message);
           }
       }
   }
   ```

## Common Pitfalls

### Common EF Core Pitfalls

1. **N+1 Query Problem**:
   ```csharp
   // PROBLEMATIC: Generates N+1 queries
   var customers = await _dbContext.Customers.ToListAsync();
   foreach (var customer in customers)
   {
       // This generates an additional query for each customer
       var orders = await _dbContext.Orders
           .Where(o => o.CustomerId == customer.Id)
           .ToListAsync();
   }
   
   // SOLUTION: Use Include for eager loading
   var customersWithOrders = await _dbContext.Customers
       .Include(c => c.Orders)
       .ToListAsync();
   ```

2. **Tracking Large Result Sets**:
   ```csharp
   // PROBLEMATIC: Tracking thousands of entities
   var products = await _dbContext.Products.ToListAsync();
   
   // SOLUTION: Use AsNoTracking for read-only queries
   var products = await _dbContext.Products
       .AsNoTracking()
       .ToListAsync();
   ```

3. **Loading Too Much Data**:
   ```csharp
   // PROBLEMATIC: Loading unnecessary data
   var customer = await _dbContext.Customers
       .Include(c => c.Orders)
       .Include(c => c.PaymentMethods)
       .Include(c => c.ShippingAddresses)
       .FirstOrDefaultAsync(c => c.Id == id);
   
   // SOLUTION: Be selective about what you load
   var customer = await _dbContext.Customers
       .Include(c => c.Orders.Where(o => o.Status == OrderStatus.Active))
       .Select(c => new CustomerViewModel
       {
           Id = c.Id,
           Name = c.Name,
           Email = c.Email,
           ActiveOrders = c.Orders.Select(o => new OrderViewModel
           {
               Id = o.Id,
               Date = o.OrderDate,
               Total = o.TotalAmount
           }).ToList()
       })
       .FirstOrDefaultAsync(c => c.Id == id);
   ```

### Database Anti-Patterns

1. **String Concatenation for Queries**:
   ```csharp
   // ANTI-PATTERN: SQL Injection risk
   string nameFilter = "O'Brien";
   string sql = $"SELECT * FROM Customers WHERE LastName = '{nameFilter}'";
   
   // CORRECT APPROACH: Parameterized query
   string sql = "SELECT * FROM Customers WHERE LastName = @LastName";
   var customers = await connection.QueryAsync(sql, new { LastName = nameFilter });
   ```

2. **Inappropriate Use of Transactions**:
   ```csharp
   // ANTI-PATTERN: Transaction for read-only operations
   using var transaction = await _dbContext.Database.BeginTransactionAsync();
   var customers = await _dbContext.Customers.ToListAsync();
   await transaction.CommitAsync();
   
   // CORRECT APPROACH: No transaction needed for reads
   var customers = await _dbContext.Customers.ToListAsync();
   ```

3. **Ignoring Connection Management**:
   ```csharp
   // ANTI-PATTERN: Creating many connections
   public async Task<Customer> GetCustomerByIdAsync(int id)
   {
       // New connection each time this method is called
       using var connection = new SqlConnection(_connectionString);
       await connection.OpenAsync();
       return await connection.QuerySingleOrDefaultAsync<Customer>(
           "SELECT * FROM Customers WHERE Id = @Id", new { Id = id });
   }
   
   // CORRECT APPROACH: Connection pooling with proper scope
   public class CustomerRepository
   {
       private readonly IDbConnectionFactory _connectionFactory;
       
       public CustomerRepository(IDbConnectionFactory connectionFactory)
       {
           _connectionFactory = connectionFactory;
       }
       
       public async Task<Customer> GetCustomerByIdAsync(int id)
       {
           using var connection = await _connectionFactory.CreateConnectionAsync();
           return await connection.QuerySingleOrDefaultAsync<Customer>(
               "SELECT * FROM Customers WHERE Id = @Id", new { Id = id });
       }
   }
   ```

## Code Examples

### Complex Query with EF Core

Example of a complex query with multiple includes and filtering:

```csharp
public async Task<List<OrderDetailDto>> GetActiveOrdersWithDetailsAsync(
    DateTime since,
    int? customerId = null)
{
    var query = _context.Orders
        .Where(o => o.Status == OrderStatus.Active && o.OrderDate >= since)
        .Include(o => o.Customer)
        .Include(o => o.Items)
            .ThenInclude(i => i.Product)
                .ThenInclude(p => p.Category)
        .Include(o => o.ShippingAddress)
        .AsQueryable();
        
    if (customerId.HasValue)
    {
        query = query.Where(o => o.CustomerId == customerId.Value);
    }
    
    return await query
        .Select(o => new OrderDetailDto
        {
            OrderId = o.Id,
            OrderDate = o.OrderDate,
            Status = o.Status.ToString(),
            Customer = new CustomerDto
            {
                Id = o.Customer.Id,
                Name = o.Customer.Name,
                Email = o.Customer.Email
            },
            Items = o.Items.Select(i => new OrderItemDto
            {
                ProductId = i.Product.Id,
                ProductName = i.Product.Name,
                CategoryName = i.Product.Category.Name,
                Quantity = i.Quantity,
                UnitPrice = i.UnitPrice
            }).ToList(),
            TotalAmount = o.TotalAmount,
            ShippingAddress = new AddressDto
            {
                Street = o.ShippingAddress.Street,
                City = o.ShippingAddress.City,
                State = o.ShippingAddress.State,
                PostalCode = o.ShippingAddress.PostalCode
            }
        })
        .OrderByDescending(o => o.OrderDate)
        .ToListAsync();
}
```

### Performance-Optimized Repository

Example of a repository implementation with performance optimizations:

```csharp
public class OptimizedProductRepository : IProductRepository
{
    private readonly ApplicationDbContext _context;
    private readonly IMemoryCache _cache;
    
    // Compiled queries for performance
    private static readonly Func<ApplicationDbContext, int, Task<Product>> GetProductByIdQuery =
        EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
            context.Products
                .Include(p => p.Category)
                .FirstOrDefault(p => p.Id == id));
                
    private static readonly Func<ApplicationDbContext, int, int, Task<List<Product>>> GetPagedProductsQuery =
        EF.CompileAsyncQuery((ApplicationDbContext context, int skip, int take) =>
            context.Products
                .Include(p => p.Category)
                .OrderBy(p => p.Name)
                .Skip(skip)
                .Take(take)
                .ToList());
    
    public OptimizedProductRepository(ApplicationDbContext context, IMemoryCache cache)
    {
        _context = context;
        _cache = cache;
    }
    
    public async Task<Product> GetByIdAsync(int id)
    {
        string cacheKey = $"product-{id}";
        
        if (!_cache.TryGetValue(cacheKey, out Product product))
        {
            product = await GetProductByIdQuery(_context, id);
            
            if (product != null)
            {
                _cache.Set(cacheKey, product, TimeSpan.FromMinutes(10));
            }
        }
        
        return product;
    }
    
    public async Task<List<Product>> GetPagedProductsAsync(int page, int pageSize)
    {
        int skip = (page - 1) * pageSize;
        string cacheKey = $"products-page-{page}-size-{pageSize}";
        
        if (!_cache.TryGetValue(cacheKey, out List<Product> products))
        {
            products = await GetPagedProductsQuery(_context, skip, pageSize);
            _cache.Set(cacheKey, products, TimeSpan.FromMinutes(5));
        }
        
        return products;
    }
    
    public async Task<List<Product>> SearchProductsAsync(string term)
    {
        if (string.IsNullOrWhiteSpace(term))
            return new List<Product>();
            
        string cacheKey = $"products-search-{term}";
        
        if (!_cache.TryGetValue(cacheKey, out List<Product> products))
        {
            // Can't use compiled query here due to dynamic nature of search
            products = await _context.Products
                .Where(p => p.Name.Contains(term) || 
                           p.Description.Contains(term) ||
                           p.Category.Name.Contains(term))
                .Take(50)
                .AsNoTracking()  // No need to track for read-only results
                .ToListAsync();
                
            _cache.Set(cacheKey, products, TimeSpan.FromMinutes(5));
        }
        
        return products;
    }
    
    public async Task AddAsync(Product product)
    {
        await _context.Products.AddAsync(product);
        await _context.SaveChangesAsync();
        
        // Invalidate potentially affected cache entries
        _cache.Remove($"product-{product.Id}");
        // Could use a more sophisticated cache invalidation strategy
    }
    
    public async Task UpdateAsync(Product product)
    {
        _context.Entry(product).State = EntityState.Modified;
        await _context.SaveChangesAsync();
        
        // Update cache
        _cache.Set($"product-{product.Id}", product, TimeSpan.FromMinutes(10));
    }
    
    public async Task DeleteAsync(int id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product != null)
        {
            _context.Products.Remove(product);
            await _context.SaveChangesAsync();
            
            // Invalidate cache
            _cache.Remove($"product-{id}");
        }
    }
}
```

## Further Reading

- [Entity Framework Core Documentation](https://docs.microsoft.com/en-us/ef/core/)
- [Dapper GitHub Repository](https://github.com/StackExchange/Dapper)
- [SQL Server Documentation](https://docs.microsoft.com/en-us/sql/sql-server/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Pro Entity Framework Core 2 for ASP.NET Core MVC](https://www.apress.com/gp/book/9781484234082) by Adam Freeman
- [Designing Data-Intensive Applications](https://dataintensive.net/) by Martin Kleppmann

## Related Topics

- [Object-Relational Mapping (ORM)](../orm/entity-framework-core.md)
- [Document Databases for .NET](../document/document-databases-dotnet.md)
- [Database Migration Strategies](../patterns/database-migrations.md)
- [Microservices Data Access](../patterns/microservices-data-access.md)
- [Database Performance Tuning](../../performance/optimization/database-performance.md)