---
title: "Advanced Exception Handling in C#"
date_created: 2023-04-02
date_updated: 2023-04-02
authors: ["Repository Maintainers"]
tags: ["c#", "exceptions", "error-handling", "try-catch", "advanced"]
difficulty: "intermediate"
---

# Advanced Exception Handling in C#

## Overview

Exception handling is a crucial aspect of writing robust, maintainable C# applications. While most developers are familiar with basic `try-catch-finally` blocks, advanced exception handling involves careful exception design, proper propagation, performance considerations, and architectural patterns. This document covers these advanced topics to help you build more resilient applications with cleaner, more maintainable code.

## Exception Handling Fundamentals

### The Exception Class Hierarchy

Understanding the exception hierarchy helps you decide which exceptions to catch and handle:

```
System.Exception
├── System.SystemException
│   ├── System.ArgumentException
│   │   └── System.ArgumentNullException
│   │   └── System.ArgumentOutOfRangeException
│   ├── System.ArithmeticException
│   │   └── System.DivideByZeroException
│   │   └── System.OverflowException
│   ├── System.NullReferenceException
│   ├── System.InvalidOperationException
│   ├── System.IndexOutOfRangeException
│   ├── System.IO.IOException
│   └── System.OutOfMemoryException
└── System.ApplicationException (discouraged as a base class)
```

### Basic Exception Handling

The fundamental syntax for exception handling in C#:

```csharp
try
{
    // Code that might throw an exception
    int result = Divide(10, 0);
}
catch (DivideByZeroException ex)
{
    // Handle specific exception type
    Console.WriteLine($"Division by zero error: {ex.Message}");
}
catch (ArithmeticException ex)
{
    // Handle base class exceptions (catches all arithmetic exceptions except those already caught)
    Console.WriteLine($"Arithmetic error: {ex.Message}");
}
catch (Exception ex)
{
    // Handle all other exceptions
    Console.WriteLine($"Unexpected error: {ex.Message}");
}
finally
{
    // This code always executes, whether an exception occurred or not
    Console.WriteLine("Finally block executed");
}
```

### Exception Properties and Methods

Key members of the `Exception` class:

```csharp
public class Exception
{
    // Properties
    public virtual string Message { get; } // Error message
    public virtual IDictionary Data { get; } // Additional user-defined information
    public Exception InnerException { get; } // The exception that caused this exception
    public virtual string StackTrace { get; } // Stack trace as a string
    public MethodBase TargetSite { get; } // Method that threw the exception
    public string Source { get; set; } // Name of the application or object that caused the error
    public string HelpLink { get; set; } // Link to help file associated with this exception
    
    // Methods
    public virtual Exception GetBaseException(); // Returns the root exception in a chain
    public Type GetType(); // Returns the runtime type of the current instance
    public virtual string ToString(); // Returns a string representation of the exception
}
```

Example of accessing exception information:

```csharp
try
{
    // Code that might throw
    throw new FileNotFoundException("Config file not found", "config.json");
}
catch (Exception ex)
{
    Console.WriteLine($"Message: {ex.Message}");
    Console.WriteLine($"Stack trace: {ex.StackTrace}");
    Console.WriteLine($"Source: {ex.Source}");
    Console.WriteLine($"Target site: {ex.TargetSite.Name}");
    
    if (ex is FileNotFoundException fileEx)
    {
        Console.WriteLine($"File name: {fileEx.FileName}");
    }
    
    // Get additional data
    if (ex.Data.Count > 0)
    {
        Console.WriteLine("Additional data:");
        foreach (DictionaryEntry entry in ex.Data)
        {
            Console.WriteLine($"  {entry.Key}: {entry.Value}");
        }
    }
}
```

## Advanced Exception Handling Techniques

### Exception Filters

C# 6.0 introduced exception filters that allow you to specify conditions for catch blocks:

```csharp
try
{
    // Code that might throw
    ProcessFile("data.txt");
}
catch (IOException ex) when (ex.HResult == -2147024893) // 0x8007000B (file not found)
{
    Console.WriteLine("File not found, creating new file");
    CreateDefaultFile("data.txt");
}
catch (IOException ex) when (IsRecoverable(ex))
{
    Console.WriteLine("Recoverable I/O error");
    // Recovery logic
}
catch (IOException ex)
{
    Console.WriteLine("Unrecoverable I/O error");
    throw;
}

bool IsRecoverable(IOException ex)
{
    // Custom logic to determine if the exception is recoverable
    return ex.HResult != -2147024784; // 0x80070070 (disk full)
}
```

Benefits of exception filters:
- They don't unwind the stack until a match is found, improving performance
- They allow more specific error handling without nesting catch blocks
- The stack trace is preserved, showing the original throw point

### Using Inner Exceptions

Use inner exceptions to preserve the original error when throwing a higher-level exception:

```csharp
public void SaveData(string data, string filePath)
{
    try
    {
        // Try to save data to file
        File.WriteAllText(filePath, data);
    }
    catch (Exception ex)
    {
        // Wrap the original exception with more context
        throw new DataPersistenceException(
            $"Failed to save data to {filePath}", 
            ex);
    }
}
```

Handling and inspecting inner exceptions:

```csharp
try
{
    // Code that calls SaveData
    dataService.SaveData("important stuff", "C:/data.txt");
}
catch (Exception ex)
{
    // Log the complete exception chain
    LogExceptionChain(ex);
}

void LogExceptionChain(Exception ex)
{
    int level = 0;
    
    while (ex != null)
    {
        Console.WriteLine($"Level {level++}: {ex.GetType().Name} - {ex.Message}");
        ex = ex.InnerException;
    }
}
```

### Adding Custom Data to Exceptions

The `Exception.Data` property allows you to add custom information to exceptions:

```csharp
public Customer GetCustomer(int customerId)
{
    try
    {
        // Database access code that might fail
        return _repository.GetCustomerById(customerId);
    }
    catch (Exception ex)
    {
        // Add helpful context information
        ex.Data["CustomerId"] = customerId;
        ex.Data["RequestTime"] = DateTime.UtcNow;
        ex.Data["UserId"] = _currentUser.Id;
        
        throw; // Rethrow with added data
    }
}
```

The data is accessible when handling the exception:

```csharp
try
{
    var customer = GetCustomer(42);
}
catch (Exception ex)
{
    Console.WriteLine($"Error: {ex.Message}");
    
    if (ex.Data.Contains("CustomerId"))
    {
        Console.WriteLine($"Customer ID: {ex.Data["CustomerId"]}");
    }
    
    // Log all custom data
    foreach (DictionaryEntry entry in ex.Data)
    {
        Console.WriteLine($"  {entry.Key}: {entry.Value}");
    }
}
```

### Exception Handling with Async/Await

Exception handling with asynchronous code follows the same patterns as synchronous code:

```csharp
public async Task ProcessDataAsync(string filePath)
{
    try
    {
        string data = await File.ReadAllTextAsync(filePath);
        await ProcessTextAsync(data);
    }
    catch (FileNotFoundException ex)
    {
        Console.WriteLine($"File not found: {ex.FileName}");
    }
    catch (Exception ex)
    {
        // Handle other exceptions
        Console.WriteLine($"Error processing file: {ex.Message}");
    }
}
```

Key considerations for async exception handling:
- Exceptions in async methods are captured and placed on the returning Task
- When awaiting, the exception is unwrapped and thrown in the context of the await
- `Task.WhenAll` will only throw the first exception - check the `Exception.InnerExceptions` property to get all exceptions

Example handling multiple exceptions with `Task.WhenAll`:

```csharp
public async Task ProcessMultipleFilesAsync(string[] filePaths)
{
    var tasks = filePaths.Select(path => ProcessFileAsync(path)).ToArray();
    
    try
    {
        await Task.WhenAll(tasks);
    }
    catch (Exception ex)
    {
        // This only catches the first exception
        Console.WriteLine($"First error: {ex.Message}");
        
        // Check all tasks for exceptions
        foreach (var task in tasks)
        {
            if (task.IsFaulted && task.Exception != null)
            {
                // Each task.Exception is an AggregateException
                foreach (var innerEx in task.Exception.InnerExceptions)
                {
                    Console.WriteLine($"Task error: {innerEx.Message}");
                }
            }
        }
    }
}
```

### Using ExceptionDispatchInfo

The `ExceptionDispatchInfo` class allows you to capture an exception to rethrow it later while preserving the original stack trace:

```csharp
public void ProcessWithDelay(Action action)
{
    ExceptionDispatchInfo capturedEx = null;
    
    try
    {
        action();
    }
    catch (Exception ex)
    {
        // Capture the exception
        capturedEx = ExceptionDispatchInfo.Capture(ex);
    }
    
    if (capturedEx != null)
    {
        // Do some work before rethrowing
        Console.WriteLine("Processing exception...");
        Thread.Sleep(1000); // Simulate work
        
        // Rethrow with original stack trace preserved
        capturedEx.Throw();
    }
}
```

This is especially useful in asynchronous scenarios:

```csharp
public async Task<T> RetryAsync<T>(Func<Task<T>> operation, int maxRetries)
{
    ExceptionDispatchInfo lastException = null;
    
    for (int i = 0; i < maxRetries; i++)
    {
        try
        {
            return await operation();
        }
        catch (Exception ex) when (ShouldRetry(ex))
        {
            lastException = ExceptionDispatchInfo.Capture(ex);
            await Task.Delay(GetDelay(i));
        }
    }
    
    // If we get here, all retries failed
    lastException?.Throw();
    
    // This line is never reached but needed for compilation
    return default;
}

private bool ShouldRetry(Exception ex)
{
    // Logic to determine if the exception is transient
    return ex is TimeoutException || 
           ex is HttpRequestException ||
           (ex is SqlException sqlEx && IsTransient(sqlEx.Number));
}

private TimeSpan GetDelay(int retryAttempt)
{
    // Exponential backoff
    return TimeSpan.FromSeconds(Math.Pow(2, retryAttempt));
}
```

## Creating and Using Custom Exceptions

### When to Create Custom Exceptions

Create custom exceptions when:

1. You need to throw domain-specific errors that aren't covered by existing exception types
2. You want to add custom properties to provide more context
3. You want to categorize exceptions for specific handling
4. You want to provide more meaningful error messages

### Custom Exception Design Guidelines

Follow these guidelines when creating custom exceptions:

1. Always derive from `Exception` or more specific existing exceptions, not `ApplicationException`
2. Name should end with "Exception"
3. Make serializable with the `[Serializable]` attribute for cross-AppDomain scenarios
4. Provide constructors that match the base Exception class
5. Include any custom properties that provide additional context
6. Override the `Message` property only if you need dynamic message generation

### Custom Exception Example

```csharp
[Serializable]
public class OrderProcessingException : Exception
{
    public int OrderId { get; }
    public string OrderStatus { get; }
    
    // Default constructor
    public OrderProcessingException() : base() { }
    
    // Constructor with message
    public OrderProcessingException(string message) : base(message) { }
    
    // Constructor with message and inner exception
    public OrderProcessingException(string message, Exception innerException) 
        : base(message, innerException) { }
    
    // Custom constructor with order information
    public OrderProcessingException(string message, int orderId, string orderStatus) 
        : base(message)
    {
        OrderId = orderId;
        OrderStatus = orderStatus;
    }
    
    // Constructor with order information and inner exception
    public OrderProcessingException(string message, int orderId, string orderStatus, Exception innerException) 
        : base(message, innerException)
    {
        OrderId = orderId;
        OrderStatus = orderStatus;
    }
    
    // Required for serialization
    protected OrderProcessingException(System.Runtime.Serialization.SerializationInfo info,
        System.Runtime.Serialization.StreamingContext context) : base(info, context)
    {
        OrderId = info.GetInt32(nameof(OrderId));
        OrderStatus = info.GetString(nameof(OrderStatus));
    }
    
    // Override GetObjectData for serialization
    public override void GetObjectData(System.Runtime.Serialization.SerializationInfo info,
        System.Runtime.Serialization.StreamingContext context)
    {
        base.GetObjectData(info, context);
        info.AddValue(nameof(OrderId), OrderId);
        info.AddValue(nameof(OrderStatus), OrderStatus);
    }
}
```

Usage example:

```csharp
public void ProcessOrder(Order order)
{
    if (order == null)
    {
        throw new ArgumentNullException(nameof(order));
    }
    
    try
    {
        // Process order
        if (!_paymentService.ProcessPayment(order.PaymentDetails))
        {
            throw new OrderProcessingException(
                "Payment processing failed", 
                order.Id, 
                order.Status);
        }
        
        // Update inventory
        try
        {
            _inventoryService.UpdateInventory(order.Items);
        }
        catch (Exception ex)
        {
            throw new OrderProcessingException(
                "Inventory update failed", 
                order.Id, 
                order.Status, 
                ex);
        }
    }
    catch (OrderProcessingException)
    {
        // Just re-throw domain exceptions
        throw;
    }
    catch (Exception ex)
    {
        // Wrap unexpected exceptions
        throw new OrderProcessingException(
            "Unexpected error processing order", 
            order.Id, 
            order.Status, 
            ex);
    }
}
```

### Creating Exception Hierarchies

For complex domains, create exception hierarchies:

```csharp
// Base exception for all payment-related issues
[Serializable]
public class PaymentException : Exception
{
    public string TransactionId { get; }
    
    public PaymentException() : base() { }
    public PaymentException(string message) : base(message) { }
    public PaymentException(string message, Exception innerException) 
        : base(message, innerException) { }
    
    public PaymentException(string message, string transactionId) 
        : base(message)
    {
        TransactionId = transactionId;
    }
    
    public PaymentException(string message, string transactionId, Exception innerException) 
        : base(message, innerException)
    {
        TransactionId = transactionId;
    }
    
    protected PaymentException(System.Runtime.Serialization.SerializationInfo info,
        System.Runtime.Serialization.StreamingContext context) : base(info, context)
    {
        TransactionId = info.GetString(nameof(TransactionId));
    }
    
    public override void GetObjectData(System.Runtime.Serialization.SerializationInfo info,
        System.Runtime.Serialization.StreamingContext context)
    {
        base.GetObjectData(info, context);
        info.AddValue(nameof(TransactionId), TransactionId);
    }
}

// Specific payment exceptions
[Serializable]
public class PaymentDeclinedException : PaymentException
{
    public string DeclineReason { get; }
    
    public PaymentDeclinedException(string message, string transactionId, string declineReason) 
        : base(message, transactionId)
    {
        DeclineReason = declineReason;
    }
    
    // Other constructors and serialization code
}

[Serializable]
public class PaymentGatewayException : PaymentException
{
    public string GatewayErrorCode { get; }
    
    public PaymentGatewayException(string message, string transactionId, string gatewayErrorCode) 
        : base(message, transactionId)
    {
        GatewayErrorCode = gatewayErrorCode;
    }
    
    // Other constructors and serialization code
}
```

## Exception Handling Strategies

### Catch and Rethrow

The different ways to rethrow exceptions have important implications:

```csharp
// 1. Rethrowing the same exception preserving the original stack trace
try
{
    // Some code that throws
}
catch (Exception ex)
{
    // Log or process the exception
    Logger.LogError(ex);
    
    throw; // Preserves the original stack trace
}

// 2. Rethrowing by creating a new exception (loses original stack trace)
try
{
    // Some code that throws
}
catch (Exception ex)
{
    // This loses the original stack trace
    throw ex; // AVOID THIS!
}

// 3. Wrapping in a more specific exception
try
{
    // Database access code
}
catch (SqlException ex)
{
    // Wrap in a more specific exception
    throw new DataAccessException("Database error", ex);
}
```

### Exception Translation

Translate low-level exceptions to domain-specific ones:

```csharp
public Customer GetCustomerById(int id)
{
    try
    {
        return _repository.GetById(id);
    }
    catch (SqlException ex) when (ex.Number == 4060)
    {
        throw new DatabaseConnectionException("Cannot connect to database", ex);
    }
    catch (SqlException ex) when (ex.Number == 208)
    {
        throw new DatabaseSchemaException("Invalid table or view", ex);
    }
    catch (DbException ex)
    {
        throw new DataAccessException($"Error retrieving customer {id}", ex);
    }
}
```

### Fail Fast vs. Resilience

Choose between failing fast or building resilience based on the context:

```csharp
// Fail fast approach - good for programming errors
public void ProcessInput(string input)
{
    // Validate preconditions
    if (string.IsNullOrEmpty(input))
    {
        throw new ArgumentException("Input cannot be null or empty", nameof(input));
    }
    
    // Process the valid input
    // ...
}

// Resilient approach - good for external dependencies
public async Task<string> GetApiDataAsync(string url)
{
    const int MaxRetries = 3;
    int retryCount = 0;
    TimeSpan delay = TimeSpan.FromSeconds(1);
    
    while (true)
    {
        try
        {
            return await _httpClient.GetStringAsync(url);
        }
        catch (HttpRequestException ex) when (IsTransient(ex) && retryCount < MaxRetries)
        {
            retryCount++;
            _logger.LogWarning(ex, "Transient error, retrying {RetryCount}/{MaxRetries}...", 
                retryCount, MaxRetries);
                
            // Exponential backoff
            await Task.Delay(delay);
            delay = TimeSpan.FromSeconds(Math.Pow(2, retryCount));
        }
    }
}

private bool IsTransient(HttpRequestException ex)
{
    // Logic to determine if the exception is transient
    return ex.StatusCode == HttpStatusCode.ServiceUnavailable ||
           ex.StatusCode == HttpStatusCode.GatewayTimeout ||
           ex.StatusCode == HttpStatusCode.RequestTimeout;
}
```

### Centralized Exception Handling

Implement centralized exception handling to ensure consistent error responses:

```csharp
// ASP.NET Core middleware for centralized exception handling
public class GlobalExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionHandlingMiddleware> _logger;
    
    public GlobalExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<GlobalExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            await HandleExceptionAsync(context, ex);
        }
    }
    
    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        var response = new ApiErrorResponse();
        
        switch (exception)
        {
            case ValidationException validationEx:
                context.Response.StatusCode = StatusCodes.Status400BadRequest;
                response.Status = "Validation Failed";
                response.Errors = validationEx.Errors;
                break;
                
            case NotFoundException notFoundEx:
                context.Response.StatusCode = StatusCodes.Status404NotFound;
                response.Status = "Not Found";
                response.Message = notFoundEx.Message;
                break;
                
            case UnauthorizedAccessException:
                context.Response.StatusCode = StatusCodes.Status401Unauthorized;
                response.Status = "Unauthorized";
                response.Message = "You are not authorized to access this resource.";
                break;
                
            case PaymentException paymentEx:
                context.Response.StatusCode = StatusCodes.Status400BadRequest;
                response.Status = "Payment Failed";
                response.Message = paymentEx.Message;
                response.TransactionId = paymentEx.TransactionId;
                break;
                
            default:
                context.Response.StatusCode = StatusCodes.Status500InternalServerError;
                response.Status = "Error";
                response.Message = "An unexpected error occurred.";
                response.TraceId = context.TraceIdentifier;
                break;
        }
        
        await context.Response.WriteAsync(JsonSerializer.Serialize(response));
    }
}

// Register the middleware in ASP.NET Core
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // Add the exception handler middleware early in the pipeline
    app.UseMiddleware<GlobalExceptionHandlingMiddleware>();
    
    // Other middleware
    app.UseRouting();
    app.UseAuthorization();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

### Circuit Breaker Pattern

Use the Circuit Breaker pattern to handle failures in external dependencies:

```csharp
// Circuit breaker pattern implementation
public class CircuitBreaker
{
    private readonly Func<Task> _operation;
    private readonly int _maxFailures;
    private readonly TimeSpan _resetTimeout;
    private readonly ILogger _logger;
    
    private int _failureCount;
    private DateTime _lastFailureTime;
    private CircuitState _state;
    
    public enum CircuitState
    {
        Closed,      // Normal operation
        Open,        // Not allowing operations
        HalfOpen     // Testing if circuit can close again
    }
    
    public CircuitBreaker(
        Func<Task> operation,
        int maxFailures,
        TimeSpan resetTimeout,
        ILogger logger)
    {
        _operation = operation;
        _maxFailures = maxFailures;
        _resetTimeout = resetTimeout;
        _logger = logger;
        _state = CircuitState.Closed;
    }
    
    public async Task ExecuteAsync()
    {
        if (_state == CircuitState.Open)
        {
            // Check if we should try half-open state
            if (DateTime.UtcNow - _lastFailureTime > _resetTimeout)
            {
                _state = CircuitState.HalfOpen;
                _logger.LogInformation("Circuit moving to half-open state");
            }
            else
            {
                _logger.LogWarning("Circuit is open, rejecting request");
                throw new CircuitBreakerOpenException("Circuit is open");
            }
        }
        
        try
        {
            await _operation();
            
            if (_state == CircuitState.HalfOpen)
            {
                // Success in half-open state, reset the circuit
                _failureCount = 0;
                _state = CircuitState.Closed;
                _logger.LogInformation("Circuit closed after successful operation");
            }
        }
        catch (Exception ex)
        {
            _lastFailureTime = DateTime.UtcNow;
            
            if (_state == CircuitState.HalfOpen || _state == CircuitState.Closed)
            {
                _failureCount++;
                
                if (_failureCount >= _maxFailures)
                {
                    _state = CircuitState.Open;
                    _logger.LogWarning(ex, "Circuit opened after {FailureCount} failures", _failureCount);
                }
                else
                {
                    _logger.LogWarning(ex, "Operation failed, failure count: {FailureCount}/{MaxFailures}", 
                        _failureCount, _maxFailures);
                }
            }
            
            throw;
        }
    }
}

// Example usage
public class ExternalServiceClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ExternalServiceClient> _logger;
    private readonly CircuitBreaker _circuitBreaker;
    
    public ExternalServiceClient(
        HttpClient httpClient,
        ILogger<ExternalServiceClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
        
        // Create circuit breaker for the API call
        _circuitBreaker = new CircuitBreaker(
            operation: () => _httpClient.GetAsync("https://api.example.com/data"),
            maxFailures: 3,
            resetTimeout: TimeSpan.FromMinutes(1),
            logger: logger);
    }
    
    public async Task<string> GetDataAsync()
    {
        try
        {
            await _circuitBreaker.ExecuteAsync();
            
            var response = await _httpClient.GetAsync("https://api.example.com/data");
            response.EnsureSuccessStatusCode();
            
            return await response.Content.ReadAsStringAsync();
        }
        catch (CircuitBreakerOpenException)
        {
            return "Service temporarily unavailable. Please try again later.";
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting data from external service");
            throw;
        }
    }
}
```

## Exception Handling Anti-Patterns

### Empty Catch Blocks

Avoid empty catch blocks as they hide errors:

```csharp
// ANTI-PATTERN: Empty catch block
try
{
    SomeOperation();
}
catch (Exception)
{
    // Swallowing the exception, causing silent failures
}

// BETTER: Log the exception at minimum
try
{
    SomeOperation();
}
catch (Exception ex)
{
    _logger.LogError(ex, "Error in SomeOperation");
    // Consider re-throwing if it's not truly recoverable
}
```

### Catching Too General Exceptions

Don't catch exceptions that you can't meaningfully handle:

```csharp
// ANTI-PATTERN: Catching Exception for specific handling
try
{
    ProcessFile(path);
}
catch (Exception ex) // Too general
{
    // Only handling file not found, but capturing ALL possible exceptions
    Console.WriteLine("File not found");
}

// BETTER: Catch specific exceptions
try
{
    ProcessFile(path);
}
catch (FileNotFoundException ex)
{
    Console.WriteLine($"File not found: {ex.FileName}");
}
catch (UnauthorizedAccessException ex)
{
    Console.WriteLine("Access denied. Check file permissions.");
}
```

### Throw Ex

Avoid `throw ex` which resets the stack trace:

```csharp
// ANTI-PATTERN: Losing stack trace
try
{
    SomeOperation();
}
catch (Exception ex)
{
    _logger.LogError(ex, "Error");
    throw ex; // Resets stack trace to this location
}

// BETTER: Preserve stack trace
try
{
    SomeOperation();
}
catch (Exception ex)
{
    _logger.LogError(ex, "Error");
    throw; // Preserves original stack trace
}
```

### Exception For Control Flow

Don't use exceptions for normal control flow:

```csharp
// ANTI-PATTERN: Using exceptions for control flow
public bool IsUserAuthorized(string userId)
{
    try
    {
        var user = _repository.GetUserById(userId);
        return user.IsActive && user.HasPermission("SomeAction");
    }
    catch (UserNotFoundException)
    {
        return false;
    }
}

// BETTER: Use normal control flow
public bool IsUserAuthorized(string userId)
{
    var user = _repository.TryGetUserById(userId, out var user);
    if (user == null)
    {
        return false;
    }
    
    return user.IsActive && user.HasPermission("SomeAction");
}
```

### Ignoring IDisposable

Always ensure proper resource disposal:

```csharp
// ANTI-PATTERN: Not disposing resources
public void WriteToFile(string content, string path)
{
    var writer = new StreamWriter(path); // Not disposed if exception occurs
    writer.WriteLine(content);
    writer.Close(); // Never reached if exception occurs
}

// BETTER: Use using statement
public void WriteToFile(string content, string path)
{
    using (var writer = new StreamWriter(path))
    {
        writer.WriteLine(content);
        // Automatically disposed even if exception occurs
    }
}

// BEST (C# 8.0+): Using declaration
public void WriteToFile(string content, string path)
{
    using var writer = new StreamWriter(path);
    writer.WriteLine(content);
    // Automatically disposed at end of scope
}
```

## Performance Considerations

### Exception Performance Impact

Exceptions have significant performance costs and should not be used for normal control flow:

```csharp
// SLOW: Using exceptions for control flow
public int ParseInteger(string input)
{
    try
    {
        return int.Parse(input);
    }
    catch (FormatException)
    {
        return 0;
    }
}

// FASTER: Using normal control flow
public int ParseInteger(string input)
{
    if (int.TryParse(input, out int result))
    {
        return result;
    }
    return 0;
}
```

### Exception Throwing vs Validation

Prefer validation to prevent exceptions:

```csharp
// INEFFICIENT: Relying on exceptions
public void ProcessOrder(Order order)
{
    try
    {
        // This might throw if order is null, or has null/invalid properties
        _orderProcessor.Process(order);
    }
    catch (ArgumentNullException ex)
    {
        // Handle null order
    }
    catch (OrderValidationException ex)
    {
        // Handle invalid order
    }
}

// EFFICIENT: Validate before proceeding
public void ProcessOrder(Order order)
{
    // Validate first
    if (order == null)
    {
        HandleNullOrder();
        return;
    }
    
    var validationResult = _validator.Validate(order);
    if (!validationResult.IsValid)
    {
        HandleInvalidOrder(validationResult.Errors);
        return;
    }
    
    // Process only valid orders
    _orderProcessor.Process(order);
}
```

### PreserveStackTrace Attribute

Use `[MethodImpl(MethodImplOptions.NoInlining)]` to improve debugging by preventing method inlining:

```csharp
[MethodImpl(MethodImplOptions.NoInlining)]
public void MethodThatThrows()
{
    throw new InvalidOperationException("Error in method");
}
```

### Try-Catch Scope

Keep try-catch blocks as small as possible:

```csharp
// SLOW: Large try block
public void ProcessFile(string path)
{
    try
    {
        // All of this is in the try block
        ValidateFilePath(path);
        var fileContent = File.ReadAllText(path);
        var processedData = ProcessData(fileContent);
        SaveResults(processedData);
    }
    catch (Exception ex)
    {
        // Handle exception
    }
}

// FASTER: Smaller, targeted try blocks
public void ProcessFile(string path)
{
    ValidateFilePath(path); // Not in try block - let any exceptions propagate
    
    string fileContent;
    try
    {
        // Only file I/O in try block
        fileContent = File.ReadAllText(path);
    }
    catch (IOException ex)
    {
        // Handle file I/O issues
        _logger.LogError(ex, "Error reading file: {Path}", path);
        throw;
    }
    
    var processedData = ProcessData(fileContent); // Not in try block
    
    try
    {
        // Only saving in try block
        SaveResults(processedData);
    }
    catch (DbException ex)
    {
        // Handle database issues
        _logger.LogError(ex, "Error saving results");
        throw;
    }
}
```

## Best Practices

### Layer-Specific Exception Handling

Handle exceptions differently at different application layers:

```csharp
// Data access layer - translate data exceptions
public class CustomerRepository : ICustomerRepository
{
    public Customer GetById(int id)
    {
        try
        {
            // Data access code
            return _context.Customers.Find(id);
        }
        catch (DbException ex)
        {
            throw new DataAccessException($"Error retrieving customer {id}", ex);
        }
    }
}

// Business logic layer - add domain context
public class CustomerService : ICustomerService
{
    public Customer GetActiveCustomer(int id)
    {
        try
        {
            var customer = _repository.GetById(id);
            
            if (customer == null)
            {
                throw new CustomerNotFoundException(id);
            }
            
            if (!customer.IsActive)
            {
                throw new InactiveCustomerException(id);
            }
            
            return customer;
        }
        catch (DataAccessException ex)
        {
            _logger.LogError(ex, "Data access error for customer {CustomerId}", id);
            throw;
        }
    }
}

// API layer - transform to appropriate responses
[ApiController]
[Route("api/[controller]")]
public class CustomersController : ControllerBase
{
    [HttpGet("{id}")]
    public ActionResult<CustomerDto> GetCustomer(int id)
    {
        try
        {
            var customer = _customerService.GetActiveCustomer(id);
            return Ok(_mapper.Map<CustomerDto>(customer));
        }
        catch (CustomerNotFoundException)
        {
            return NotFound();
        }
        catch (InactiveCustomerException)
        {
            return BadRequest(new { error = "Customer account is inactive" });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting customer {CustomerId}", id);
            return StatusCode(500, new { error = "An unexpected error occurred" });
        }
    }
}
```

### Validate Arguments Early

Validate arguments at the beginning of methods:

```csharp
public void ProcessOrder(Order order, Customer customer, PaymentMethod paymentMethod)
{
    // Validate all arguments up front
    if (order == null)
        throw new ArgumentNullException(nameof(order));
        
    if (customer == null)
        throw new ArgumentNullException(nameof(customer));
        
    if (paymentMethod == null)
        throw new ArgumentNullException(nameof(paymentMethod));
        
    if (order.Items.Count == 0)
        throw new ArgumentException("Order must contain at least one item", nameof(order));
        
    if (!customer.IsActive)
        throw new ArgumentException("Customer account is not active", nameof(customer));
        
    if (!paymentMethod.IsValid)
        throw new ArgumentException("Payment method is not valid", nameof(paymentMethod));
    
    // Now proceed with valid arguments
    // ...
}
```

### Consistent Exception Naming

Follow consistent naming conventions for exceptions:

```csharp
// All exception names should end with "Exception"
public class OrderNotFoundException : Exception { }
public class PaymentDeclinedException : Exception { }
public class ShippingFailedException : Exception { }

// Group related exceptions in namespaces
namespace MyApp.Exceptions.Orders { }
namespace MyApp.Exceptions.Payments { }
namespace MyApp.Exceptions.Shipping { }
```

### Logging Strategy

Implement a consistent logging strategy:

```csharp
// Log at the appropriate level
try
{
    ProcessOrder(order);
}
catch (ValidationException ex)
{
    // Expected exceptions - Warning level
    _logger.LogWarning(ex, "Order validation failed: {OrderId}", order.Id);
    return BadRequest(ex.Errors);
}
catch (Exception ex)
{
    // Unexpected exceptions - Error level
    _logger.LogError(ex, "Error processing order: {OrderId}", order.Id);
    return StatusCode(500, "An unexpected error occurred");
}

// Include context information
try
{
    // Code that might throw
}
catch (Exception ex)
{
    // Add contextual information to log
    _logger.LogError(ex, 
        "Error processing order {OrderId} for customer {CustomerId} with payment {PaymentId}", 
        order.Id, order.CustomerId, order.PaymentId);
    throw;
}

// Use structured logging
public void ProcessOrder(Order order)
{
    try
    {
        // Using scope to add context to all log entries inside this method
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["OrderId"] = order.Id,
            ["CustomerId"] = order.CustomerId,
            ["OrderAmount"] = order.TotalAmount
        }))
        {
            _logger.LogInformation("Starting order processing");
            
            // Process order...
            
            _logger.LogInformation("Order processing completed");
        }
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Order processing failed");
        throw;
    }
}
```

### Exception Documentation

Document exceptions in code:

```csharp
/// <summary>
/// Processes a customer order
/// </summary>
/// <param name="order">The order to process</param>
/// <exception cref="ArgumentNullException">Thrown when order is null</exception>
/// <exception cref="OrderValidationException">Thrown when order fails validation</exception>
/// <exception cref="PaymentDeclinedException">Thrown when payment is declined</exception>
/// <exception cref="InventoryUnavailableException">Thrown when items are out of stock</exception>
public void ProcessOrder(Order order)
{
    // Implementation
}
```

## Real-World Examples

### Web API Exception Handler

A complete exception handling middleware for ASP.NET Core:

```csharp
public class GlobalExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionHandlingMiddleware> _logger;
    private readonly IHostEnvironment _environment;
    
    public GlobalExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<GlobalExceptionHandlingMiddleware> logger,
        IHostEnvironment environment)
    {
        _next = next;
        _logger = logger;
        _environment = environment;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }
    
    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        // Get detailed information for logging
        var requestPath = context.Request.Path;
        var requestMethod = context.Request.Method;
        var traceId = context.TraceIdentifier;
        
        // Map exception to appropriate HTTP status code and response
        HttpStatusCode statusCode;
        string errorType;
        string message;
        object details = null;
        
        switch (exception)
        {
            case ValidationException validationEx:
                statusCode = HttpStatusCode.BadRequest;
                errorType = "Validation_Error";
                message = "One or more validation errors occurred.";
                details = validationEx.Errors;
                
                _logger.LogWarning(exception, 
                    "Validation error on {RequestMethod} {RequestPath} (TraceId: {TraceId})",
                    requestMethod, requestPath, traceId);
                break;
                
            case NotFoundException notFoundEx:
                statusCode = HttpStatusCode.NotFound;
                errorType = "Resource_Not_Found";
                message = notFoundEx.Message;
                
                _logger.LogInformation(exception,
                    "Resource not found on {RequestMethod} {RequestPath} (TraceId: {TraceId})",
                    requestMethod, requestPath, traceId);
                break;
                
            case UnauthorizedAccessException:
                statusCode = HttpStatusCode.Unauthorized;
                errorType = "Unauthorized";
                message = "You are not authorized to access this resource.";
                
                _logger.LogWarning(exception,
                    "Unauthorized access attempt on {RequestMethod} {RequestPath} (TraceId: {TraceId})",
                    requestMethod, requestPath, traceId);
                break;
                
            case ForbiddenException:
                statusCode = HttpStatusCode.Forbidden;
                errorType = "Forbidden";
                message = "You do not have permission to access this resource.";
                
                _logger.LogWarning(exception,
                    "Forbidden access attempt on {RequestMethod} {RequestPath} (TraceId: {TraceId})",
                    requestMethod, requestPath, traceId);
                break;
                
            default:
                statusCode = HttpStatusCode.InternalServerError;
                errorType = "Server_Error";
                message = "An unexpected error occurred.";
                
                _logger.LogError(exception,
                    "Unhandled exception on {RequestMethod} {RequestPath} (TraceId: {TraceId})",
                    requestMethod, requestPath, traceId);
                break;
        }
        
        // Create error response
        var response = new ErrorResponse
        {
            TraceId = traceId,
            Type = errorType,
            Title = message,
            Status = (int)statusCode
        };
        
        // Add details for non-production environments or certain error types
        if (details != null)
        {
            response.Details = details;
        }
        else if (_environment.IsDevelopment() || statusCode != HttpStatusCode.InternalServerError)
        {
            response.Details = exception.Message;
        }
        
        // Add stack trace in development environment for debugging
        if (_environment.IsDevelopment())
        {
            response.StackTrace = exception.StackTrace;
        }
        
        // Set response details
        context.Response.StatusCode = (int)statusCode;
        context.Response.ContentType = "application/json";
        
        // Write the response
        await context.Response.WriteAsync(JsonSerializer.Serialize(response, new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
            WriteIndented = true
        }));
    }
}

public class ErrorResponse
{
    public string TraceId { get; set; }
    public string Type { get; set; }
    public string Title { get; set; }
    public int Status { get; set; }
    public object Details { get; set; }
    public string StackTrace { get; set; }
}
```

### Retry Pattern with Exponential Backoff

Implementing a robust retry mechanism:

```csharp
public class RetryWithBackoff<T>
{
    private readonly Func<Task<T>> _operation;
    private readonly int _maxRetries;
    private readonly Func<Exception, int, bool> _shouldRetry;
    private readonly ILogger _logger;
    
    public RetryWithBackoff(
        Func<Task<T>> operation,
        int maxRetries,
        Func<Exception, int, bool> shouldRetry,
        ILogger logger)
    {
        _operation = operation;
        _maxRetries = maxRetries;
        _shouldRetry = shouldRetry;
        _logger = logger;
    }
    
    public async Task<T> ExecuteAsync()
    {
        int attempt = 0;
        Exception lastException = null;
        
        while (attempt <= _maxRetries)
        {
            try
            {
                if (attempt > 0)
                {
                    // Calculate delay using exponential backoff with jitter
                    var delay = CalculateDelay(attempt);
                    _logger.LogInformation("Retrying operation (attempt {Attempt} of {MaxRetries}) after {Delay}ms",
                        attempt, _maxRetries, delay.TotalMilliseconds);
                        
                    await Task.Delay(delay);
                }
                
                attempt++;
                return await _operation();
            }
            catch (Exception ex)
            {
                lastException = ex;
                
                if (attempt <= _maxRetries && _shouldRetry(ex, attempt))
                {
                    _logger.LogWarning(ex, "Operation failed, will retry (attempt {Attempt} of {MaxRetries})",
                        attempt, _maxRetries);
                }
                else
                {
                    _logger.LogError(ex, "Operation failed after {Attempt} attempts", attempt);
                    throw;
                }
            }
        }
        
        // This should never be reached due to the throw in the catch block,
        // but the compiler doesn't know that
        throw lastException;
    }
    
    private TimeSpan CalculateDelay(int attempt)
    {
        // Exponential backoff: 2^attempt * 100ms base
        double baseDelay = 100 * Math.Pow(2, attempt - 1);
        
        // Add jitter: +/- 20% of the delay
        var random = new Random();
        double jitter = (random.NextDouble() - 0.5) * 0.4 * baseDelay;
        
        // Apply delay with jitter, ensuring it doesn't exceed reasonable bounds
        int delayMs = (int)Math.Min(baseDelay + jitter, 30000); // Cap at 30 seconds
        
        return TimeSpan.FromMilliseconds(delayMs);
    }
}

// Example usage
public class ApiClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ApiClient> _logger;
    
    public ApiClient(HttpClient httpClient, ILogger<ApiClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }
    
    public async Task<ApiResponse> GetDataAsync(string endpoint)
    {
        var retry = new RetryWithBackoff<ApiResponse>(
            operation: () => CallApiAsync(endpoint),
            maxRetries: 3,
            shouldRetry: (ex, attempt) => IsTransientException(ex),
            logger: _logger);
            
        return await retry.ExecuteAsync();
    }
    
    private async Task<ApiResponse> CallApiAsync(string endpoint)
    {
        var response = await _httpClient.GetAsync(endpoint);
        response.EnsureSuccessStatusCode();
        
        var content = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<ApiResponse>(content);
    }
    
    private bool IsTransientException(Exception ex)
    {
        // Determine if the exception is transient/retryable
        return ex is HttpRequestException ||
               ex is TimeoutException ||
               (ex is ApiException apiEx && (
                   apiEx.StatusCode == HttpStatusCode.ServiceUnavailable ||
                   apiEx.StatusCode == HttpStatusCode.GatewayTimeout ||
                   apiEx.StatusCode == HttpStatusCode.RequestTimeout ||
                   apiEx.StatusCode == HttpStatusCode.TooManyRequests));
    }
}
```

### Complex Transaction Handling

Managing database transactions with proper exception handling:

```csharp
public class OrderService : IOrderService
{
    private readonly ApplicationDbContext _dbContext;
    private readonly IPaymentService _paymentService;
    private readonly IInventoryService _inventoryService;
    private readonly ILogger<OrderService> _logger;
    
    public OrderService(
        ApplicationDbContext dbContext,
        IPaymentService paymentService,
        IInventoryService inventoryService,
        ILogger<OrderService> logger)
    {
        _dbContext = dbContext;
        _paymentService = paymentService;
        _inventoryService = inventoryService;
        _logger = logger;
    }
    
    public async Task<OrderResult> PlaceOrderAsync(OrderRequest request)
    {
        // Validate request
        if (request == null)
            throw new ArgumentNullException(nameof(request));
            
        if (request.Items == null || !request.Items.Any())
            throw new ValidationException("Order must contain at least one item");
            
        // Start database transaction
        using var transaction = await _dbContext.Database.BeginTransactionAsync();
        
        try
        {
            // Create order record
            var order = new Order
            {
                CustomerId = request.CustomerId,
                OrderDate = DateTime.UtcNow,
                Status = OrderStatus.Pending,
                ShippingAddress = request.ShippingAddress,
                Items = request.Items.Select(i => new OrderItem
                {
                    ProductId = i.ProductId,
                    Quantity = i.Quantity,
                    UnitPrice = i.UnitPrice
                }).ToList()
            };
            
            _dbContext.Orders.Add(order);
            await _dbContext.SaveChangesAsync();
            
            // Check inventory
            try
            {
                await _inventoryService.ReserveInventoryAsync(order.Items);
            }
            catch (InventoryUnavailableException ex)
            {
                _logger.LogWarning(ex, "Inventory unavailable for order");
                throw new OrderProcessingException(
                    "Cannot place order due to inventory shortage", 
                    order.Id,
                    "Pending",
                    ex);
            }
            
            // Process payment
            try
            {
                var paymentResult = await _paymentService.ProcessPaymentAsync(
                    request.PaymentInfo, 
                    order.GetTotalAmount());
                    
                if (!paymentResult.Success)
                {
                    throw new PaymentDeclinedException(
                        "Payment was declined", 
                        paymentResult.TransactionId, 
                        paymentResult.DeclineReason);
                }
                
                // Update order with payment information
                order.PaymentTransactionId = paymentResult.TransactionId;
                order.Status = OrderStatus.Paid;
                await _dbContext.SaveChangesAsync();
            }
            catch (PaymentDeclinedException ex)
            {
                // Release inventory
                await _inventoryService.ReleaseInventoryAsync(order.Items);
                
                _logger.LogWarning(ex, 
                    "Payment declined for order {OrderId}: {Reason}", 
                    order.Id, ex.DeclineReason);
                
                throw new OrderProcessingException(
                    $"Payment declined: {ex.DeclineReason}", 
                    order.Id,
                    "Pending",
                    ex);
            }
            catch (Exception ex)
            {
                // Release inventory
                await _inventoryService.ReleaseInventoryAsync(order.Items);
                
                _logger.LogError(ex, "Payment error for order {OrderId}", order.Id);
                
                throw new OrderProcessingException(
                    "Error processing payment", 
                    order.Id,
                    "Pending",
                    ex);
            }
            
            // Commit transaction
            await transaction.CommitAsync();
            
            // Return success result
            return new OrderResult
            {
                OrderId = order.Id,
                Status = order.Status.ToString(),
                TotalAmount = order.GetTotalAmount(),
                TransactionId = order.PaymentTransactionId
            };
        }
        catch (OrderProcessingException)
        {
            // Already handled and containing appropriate context
            await transaction.RollbackAsync();
            throw;
        }
        catch (Exception ex)
        {
            // Unexpected exception
            _logger.LogError(ex, "Unexpected error processing order for customer {CustomerId}", 
                request.CustomerId);
            
            await transaction.RollbackAsync();
            
            throw new OrderProcessingException(
                "An unexpected error occurred while processing your order", 
                0, // No order ID in case of early failure
                "Failed", 
                ex);
        }
    }
}
```

## Further Reading

- [Exception Handling Best Practices in C#](https://docs.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions)
- [Exceptions and Performance](https://docs.microsoft.com/en-us/dotnet/framework/performance/performance-tips)
- [Error Handling in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/error-handling)
- [Custom Exceptions in .NET](https://docs.microsoft.com/en-us/dotnet/standard/exceptions/how-to-create-user-defined-exceptions)
- [Task Exception Handling in Async Programming](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/exception-handling-task-parallel-library)

## Related Topics

- [C# Asynchronous Programming](../advanced/async-await.md)
- [Logging and Monitoring](../../devops/monitoring/logging.md)
- [Design Patterns - Circuit Breaker](../../architecture/patterns/circuit-breaker.md)
- [Clean Code Principles](../../architecture/principles/clean-code.md)
- [API Error Handling](../../api-development/rest/error-handling.md)