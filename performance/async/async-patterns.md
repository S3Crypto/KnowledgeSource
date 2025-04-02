---
title: "Asynchronous Programming Patterns in C#"
date_created: 2023-04-02
date_updated: 2023-04-02
authors: ["Repository Maintainers"]
tags: ["async", "threading", "performance", "task", "concurrency"]
difficulty: "intermediate"
---

# Asynchronous Programming Patterns in C#

## Overview

Asynchronous programming is a cornerstone of modern C# development, enabling responsive applications and efficient resource utilization. This document covers the evolution of asynchronous patterns in C#, best practices, common pitfalls, and advanced techniques. Understanding these patterns is essential for building high-performance, scalable applications.

## Core Concepts

### Synchronous vs. Asynchronous Execution

In synchronous execution, operations occur sequentially, with each operation blocking until completed:

```csharp
// Synchronous operations
public string DownloadAndProcessData(string url)
{
    string data = DownloadData(url);      // Blocks until download completes
    string processed = ProcessData(data);  // Blocks until processing completes
    return processed;
}
```

In asynchronous execution, operations can be initiated and then suspended without blocking, allowing other work to be done:

```csharp
// Asynchronous operations
public async Task<string> DownloadAndProcessDataAsync(string url)
{
    string data = await DownloadDataAsync(url);      // Yields control during download
    string processed = await ProcessDataAsync(data);  // Yields control during processing
    return processed;
}
```

### Concurrency vs. Parallelism

- **Concurrency**: Managing multiple operations in progress (potentially interleaved)
- **Parallelism**: Executing multiple operations simultaneously (truly at the same time)

```csharp
// Concurrency with async/await
public async Task<List<string>> ProcessUrlsConcurrentlyAsync(List<string> urls)
{
    var tasks = urls.Select(url => ProcessUrlAsync(url));
    return await Task.WhenAll(tasks);
}

// Parallelism with Parallel.ForEach
public List<string> ProcessUrlsInParallel(List<string> urls)
{
    var results = new ConcurrentBag<string>();
    Parallel.ForEach(urls, url => 
    {
        string result = ProcessUrl(url);
        results.Add(result);
    });
    return results.ToList();
}
```

### Tasks and the Task Parallel Library (TPL)

A `Task` represents an asynchronous operation that may or may not return a value:

- `Task`: Represents a void-returning operation
- `Task<T>`: Represents an operation that returns a value of type T
- Tasks can be awaited, combined, and manipulated

```csharp
// Creating tasks
Task task1 = Task.Run(() => Console.WriteLine("Hello from Task"));
Task<int> task2 = Task.Run(() => ComputeValue());
Task task3 = Task.Delay(1000); // Timer-based task

// Waiting for tasks
await task1;
int result = await task2;
await task3;
```

## Evolution of Asynchronous Patterns in C#

### APM (Asynchronous Programming Model)

The original pattern using Begin/End method pairs:

```csharp
// APM pattern
public void DownloadFileWithAPM(string url, string destination)
{
    WebClient client = new WebClient();
    client.DownloadFileCompleted += (sender, e) => Console.WriteLine("Download completed");
    
    // Begin asynchronous operation
    IAsyncResult asyncResult = client.BeginDownloadFile(
        new Uri(url), 
        destination, 
        null, 
        null);
    
    // Do other work here...
    
    // Wait for completion if needed
    asyncResult.AsyncWaitHandle.WaitOne();
    
    // End asynchronous operation
    client.EndDownloadFile(asyncResult);
}
```

### EAP (Event-based Asynchronous Pattern)

Uses events to signal completion:

```csharp
// EAP pattern
public void DownloadFileWithEAP(string url, string destination)
{
    WebClient client = new WebClient();
    
    // Setup event handlers
    client.DownloadProgressChanged += (sender, e) => 
        Console.WriteLine($"Downloaded {e.ProgressPercentage}%");
        
    client.DownloadFileCompleted += (sender, e) => 
    {
        if (e.Error != null)
            Console.WriteLine($"Error: {e.Error.Message}");
        else
            Console.WriteLine("Download completed successfully");
    };
    
    // Start asynchronous operation
    client.DownloadFileAsync(new Uri(url), destination);
    
    // Do other work here...
}
```

### TAP (Task-based Asynchronous Pattern)

The modern approach using async/await:

```csharp
// TAP pattern
public async Task DownloadFileWithTAPAsync(string url, string destination)
{
    using (HttpClient client = new HttpClient())
    {
        byte[] fileBytes = await client.GetByteArrayAsync(url);
        await File.WriteAllBytesAsync(destination, fileBytes);
        Console.WriteLine("Download completed successfully");
    }
    
    // Do other work here...
}
```

## Async/Await Pattern

### Basic Async/Await Usage

The async/await pattern simplifies asynchronous code:

```csharp
public async Task<string> GetWebContentAsync(string url)
{
    using (HttpClient client = new HttpClient())
    {
        // await suspends execution without blocking the thread
        string content = await client.GetStringAsync(url);
        return content;
    }
}

// Calling async methods
public async Task ProcessUrlAsync()
{
    string content = await GetWebContentAsync("https://example.com");
    Console.WriteLine($"Content length: {content.Length}");
}
```

### Async Method Signatures

```csharp
// Common async method signatures
public async Task DoWorkAsync() { /* No return value */ }
public async Task<int> CalculateAsync() { /* Returns int */ }
public async ValueTask<string> GetValueAsync() { /* More efficient for cached results */ }

// Avoid these antipatterns
public async void ButtonClick() { /* Avoid async void except for event handlers */ }
public Task<int> BadMethodAsync() { /* Missing await inside method body */ }
```

### Async Method Naming

- Suffix async methods with "Async"
- Keep sync and async versions if needed
- Be consistent with naming

```csharp
// Good naming practice
public string GetData() { /* Synchronous implementation */ }
public async Task<string> GetDataAsync() { /* Asynchronous implementation */ }
```

## Task Composition Patterns

### Sequential Composition

Execute async operations in sequence:

```csharp
public async Task<OrderResult> ProcessOrderAsync(Order order)
{
    // Sequential operations where each depends on the previous
    ValidationResult validation = await ValidateOrderAsync(order);
    if (!validation.IsValid)
        return new OrderResult { Success = false, Error = validation.Error };
    
    PaymentResult payment = await ProcessPaymentAsync(order);
    if (!payment.Success)
        return new OrderResult { Success = false, Error = "Payment failed" };
    
    ShippingResult shipping = await ArrangeShippingAsync(order, payment);
    
    return new OrderResult 
    { 
        Success = true, 
        OrderId = order.Id,
        TrackingNumber = shipping.TrackingNumber
    };
}
```

### Parallel Composition

Execute multiple async operations in parallel:

```csharp
public async Task<DashboardData> LoadDashboardDataAsync(string userId)
{
    // Start all operations without awaiting them immediately
    Task<UserProfile> profileTask = GetUserProfileAsync(userId);
    Task<List<Order>> recentOrdersTask = GetRecentOrdersAsync(userId);
    Task<List<Notification>> notificationsTask = GetNotificationsAsync(userId);
    Task<List<Product>> recommendationsTask = GetProductRecommendationsAsync(userId);
    
    // Now await all tasks to complete
    await Task.WhenAll(profileTask, recentOrdersTask, notificationsTask, recommendationsTask);
    
    // All tasks are now complete, and their results are available
    return new DashboardData
    {
        Profile = await profileTask,
        RecentOrders = await recentOrdersTask,
        Notifications = await notificationsTask,
        Recommendations = await recommendationsTask
    };
}
```

### Interleaved Operations

Start operations in parallel but process them in a specific order:

```csharp
public async Task<ProcessingResult> ProcessDataWithDependenciesAsync()
{
    // Start all operations
    Task<ConfigData> configTask = LoadConfigurationAsync();
    Task<RawData> dataTask = FetchDataAsync();
    
    // Wait for config first since we need it to process the data
    ConfigData config = await configTask;
    
    // Now wait for data
    RawData data = await dataTask;
    
    // Process with the configuration
    return await ProcessWithConfigAsync(data, config);
}
```

### Task Coordination

Patterns for coordinating multiple tasks:

```csharp
// Wait for all tasks to complete
public async Task ProcessAllAsync(List<int> items)
{
    var tasks = items.Select(item => ProcessItemAsync(item));
    await Task.WhenAll(tasks);
    Console.WriteLine("All items processed");
}

// Wait for any task to complete
public async Task<int> GetFirstResponseAsync(List<string> urls)
{
    var tasks = urls.Select(url => FetchDataAsync(url));
    Task<int> firstCompleted = await Task.WhenAny(tasks);
    return await firstCompleted;
}

// Process tasks as they complete
public async Task ProcessInCompletionOrderAsync(List<WorkItem> workItems)
{
    // Create a list of tasks
    var tasks = workItems.Select(item => ProcessWorkItemAsync(item)).ToList();
    
    // Process results as they arrive
    while (tasks.Count > 0)
    {
        Task<WorkResult> completed = await Task.WhenAny(tasks);
        tasks.Remove(completed);
        
        WorkResult result = await completed;
        Console.WriteLine($"Processed item: {result.ItemId}");
    }
}
```

### WhenAll with Handling Exceptions

Properly handling exceptions with `Task.WhenAll`:

```csharp
public async Task ProcessAllWithErrorHandlingAsync(List<string> urls)
{
    var tasks = urls.Select(url => ProcessUrlWithRetryAsync(url)).ToList();
    
    try
    {
        await Task.WhenAll(tasks);
        Console.WriteLine("All URLs processed successfully");
    }
    catch (Exception)
    {
        // WhenAll throws the first exception, but we want to know all failures
        var exceptions = tasks
            .Where(t => t.IsFaulted)
            .Select(t => t.Exception.InnerException)
            .ToList();
            
        foreach (var ex in exceptions)
        {
            Console.WriteLine($"Error processing URL: {ex.Message}");
        }
    }
}
```

## Threading Patterns

### Thread Pool Usage

The .NET ThreadPool manages threads for optimal resource usage:

```csharp
// ThreadPool manages threads automatically
public void QueueWorkItem()
{
    ThreadPool.QueueUserWorkItem(state => 
    {
        Console.WriteLine($"Work executing on thread {Thread.CurrentThread.ManagedThreadId}");
        // Do work...
    });
}

// Task.Run queues work to the ThreadPool
public async Task DoWorkOnThreadPoolAsync()
{
    await Task.Run(() => 
    {
        Console.WriteLine($"Task running on thread {Thread.CurrentThread.ManagedThreadId}");
        // Do work...
    });
}
```

### Background Workers

Long-running operations that don't block the main thread:

```csharp
// Background worker pattern
public class DataProcessor
{
    private readonly CancellationTokenSource _cts = new CancellationTokenSource();
    private Task _processingTask;
    
    public void StartProcessing()
    {
        _processingTask = Task.Run(async () => 
        {
            try
            {
                while (!_cts.Token.IsCancellationRequested)
                {
                    // Process next batch of data
                    await ProcessNextBatchAsync(_cts.Token);
                    
                    // Avoid tight CPU loops
                    await Task.Delay(100, _cts.Token);
                }
            }
            catch (OperationCanceledException)
            {
                // Expected when cancellation is requested
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Background processing error: {ex.Message}");
            }
        }, _cts.Token);
    }
    
    public async Task StopProcessingAsync()
    {
        if (_processingTask != null)
        {
            _cts.Cancel();
            await _processingTask;
            _cts.Dispose();
        }
    }
    
    private async Task ProcessNextBatchAsync(CancellationToken token)
    {
        // Implement batch processing logic
    }
}
```

### Producer-Consumer Pattern

Processing items from a queue with multiple producers and consumers:

```csharp
public class ProducerConsumerQueue<T>
{
    private readonly BlockingCollection<T> _queue = new BlockingCollection<T>();
    private readonly List<Task> _consumers = new List<Task>();
    private readonly CancellationTokenSource _cts = new CancellationTokenSource();
    
    public void Start(int consumerCount, Func<T, Task> processItem)
    {
        for (int i = 0; i < consumerCount; i++)
        {
            _consumers.Add(Task.Run(async () => 
            {
                try
                {
                    foreach (var item in _queue.GetConsumingEnumerable(_cts.Token))
                    {
                        await processItem(item);
                    }
                }
                catch (OperationCanceledException)
                {
                    // Expected when cancellation is requested
                }
            }));
        }
    }
    
    public void Produce(T item)
    {
        _queue.Add(item);
    }
    
    public async Task CompleteAsync()
    {
        _queue.CompleteAdding();
        await Task.WhenAll(_consumers);
    }
    
    public async Task StopAsync()
    {
        _cts.Cancel();
        _queue.CompleteAdding();
        await Task.WhenAll(_consumers);
    }
}

// Usage
var queue = new ProducerConsumerQueue<string>();
queue.Start(5, async (url) => 
{
    var data = await DownloadDataAsync(url);
    await ProcessDataAsync(data);
});

foreach (var url in urls)
{
    queue.Produce(url);
}

await queue.CompleteAsync();
```

## Advanced Patterns

### Throttling and Rate Limiting

Limiting the concurrency of operations:

```csharp
public class RateLimiter
{
    private readonly SemaphoreSlim _throttler;
    
    public RateLimiter(int maxConcurrency)
    {
        _throttler = new SemaphoreSlim(maxConcurrency);
    }
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> function)
    {
        await _throttler.WaitAsync();
        try
        {
            return await function();
        }
        finally
        {
            _throttler.Release();
        }
    }
}

// Usage
public async Task ProcessUrlsWithThrottlingAsync(List<string> urls)
{
    var limiter = new RateLimiter(10); // Max 10 concurrent operations
    
    var tasks = urls.Select(async url => 
    {
        return await limiter.ExecuteAsync(async () => 
        {
            var result = await DownloadDataAsync(url);
            return result;
        });
    });
    
    var results = await Task.WhenAll(tasks);
}
```

### Batching Operations

Processing items in batches for efficiency:

```csharp
public class Batcher<T>
{
    private readonly int _batchSize;
    private readonly Func<IEnumerable<T>, Task> _processBatch;
    
    public Batcher(int batchSize, Func<IEnumerable<T>, Task> processBatch)
    {
        _batchSize = batchSize;
        _processBatch = processBatch;
    }
    
    public async Task BatchProcessAsync(IEnumerable<T> items)
    {
        var batches = items
            .Select((item, index) => new { item, index })
            .GroupBy(x => x.index / _batchSize)
            .Select(g => g.Select(x => x.item).ToList());
            
        foreach (var batch in batches)
        {
            await _processBatch(batch);
        }
    }
}

// Usage
var batcher = new Batcher<Order>(100, async orders => 
{
    await _repository.BulkInsertAsync(orders);
});

await batcher.BatchProcessAsync(allOrders);
```

### Asynchronous Initialization Pattern

Safe initialization of asynchronous resources:

```csharp
public class AsyncInitializer<T>
{
    private readonly Func<Task<T>> _factory;
    private readonly SemaphoreSlim _lock = new SemaphoreSlim(1);
    private Task<T> _initTask;
    
    public AsyncInitializer(Func<Task<T>> factory)
    {
        _factory = factory;
    }
    
    public async Task<T> GetValueAsync()
    {
        if (_initTask == null)
        {
            await _lock.WaitAsync();
            try
            {
                if (_initTask == null)
                {
                    _initTask = _factory();
                }
            }
            finally
            {
                _lock.Release();
            }
        }
        
        return await _initTask;
    }
}

// Usage
public class DatabaseService
{
    private readonly AsyncInitializer<DbConnection> _connectionInitializer;
    
    public DatabaseService(string connectionString)
    {
        _connectionInitializer = new AsyncInitializer<DbConnection>(async () => 
        {
            var connection = new SqlConnection(connectionString);
            await connection.OpenAsync();
            return connection;
        });
    }
    
    public async Task<int> ExecuteQueryAsync(string sql)
    {
        var connection = await _connectionInitializer.GetValueAsync();
        using var command = connection.CreateCommand();
        command.CommandText = sql;
        return await command.ExecuteNonQueryAsync();
    }
}
```

### Timeouts and Cancellation

Implementing timeouts and supporting cancellation:

```csharp
public async Task<T> WithTimeout<T>(Task<T> task, TimeSpan timeout)
{
    using var cts = new CancellationTokenSource();
    var timeoutTask = Task.Delay(timeout, cts.Token);
    var completedTask = await Task.WhenAny(task, timeoutTask);
    
    if (completedTask == timeoutTask)
    {
        throw new TimeoutException($"Operation timed out after {timeout.TotalSeconds} seconds");
    }
    
    cts.Cancel(); // Cancel the timeout task
    return await task; // Unwrap the result
}

// Usage with cancellation
public async Task<string> GetDataWithTimeoutAsync(string url, TimeSpan timeout, CancellationToken cancellationToken)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
    cts.CancelAfter(timeout);
    
    try
    {
        using var client = new HttpClient();
        return await client.GetStringAsync(url, cts.Token);
    }
    catch (OperationCanceledException)
    {
        if (cancellationToken.IsCancellationRequested)
            throw; // Original cancellation
            
        throw new TimeoutException($"Request to {url} timed out after {timeout.TotalSeconds} seconds");
    }
}
```

## Best Practices

### Async All the Way

Avoid mixing synchronous and asynchronous code:

```csharp
// GOOD: Async all the way
public async Task<string> GoodAsyncMethod()
{
    using var client = new HttpClient();
    var data = await client.GetStringAsync("https://example.com");
    return await ProcessDataAsync(data);
}

// BAD: Blocking on async code
public string BadBlockingMethod()
{
    using var client = new HttpClient();
    // Blocking call can lead to deadlocks!
    var data = client.GetStringAsync("https://example.com").Result;
    return ProcessDataAsync(data).Result;
}
```

### ConfigureAwait Usage

Use `ConfigureAwait(false)` for library code:

```csharp
// Library method: Use ConfigureAwait(false) to avoid context capturing
public async Task<string> LibraryMethodAsync()
{
    using var client = new HttpClient();
    var data = await client.GetStringAsync("https://example.com").ConfigureAwait(false);
    return await ProcessDataAsync(data).ConfigureAwait(false);
}

// UI application method: Maintain context for UI updates
public async Task UpdateUIAsync()
{
    var result = await GetDataAsync();
    // This needs to run on the UI thread
    ResultTextBox.Text = result;
}
```

### Task Creation and Handling

Proper task creation and error handling:

```csharp
// Create tasks correctly
public Task<int> CalculateValueAsync()
{
    // GOOD: Return the task directly when possible
    return Task.Run(() => ComputeValue());
    
    // BAD: Unnecessary async/await here
    // return await Task.Run(() => ComputeValue());
}

// Handle task exceptions properly
public async Task<string> ProcessWithErrorHandlingAsync()
{
    try
    {
        return await SomeOperationThatMightFailAsync();
    }
    catch (HttpRequestException ex)
    {
        // Handle specific exception
        _logger.LogError(ex, "HTTP request failed");
        return "Default value due to request failure";
    }
    catch (Exception ex)
    {
        // Handle other exceptions
        _logger.LogError(ex, "Unexpected error");
        throw; // Rethrow if you can't handle it
    }
}
```

### Cancellation Support

Support cancellation in long-running operations:

```csharp
public async Task<List<SearchResult>> SearchAsync(
    string query, 
    CancellationToken cancellationToken = default)
{
    // Check cancellation early
    cancellationToken.ThrowIfCancellationRequested();
    
    var results = new List<SearchResult>();
    
    // Search multiple sources
    foreach (var source in _searchSources)
    {
        // Check cancellation in loop
        cancellationToken.ThrowIfCancellationRequested();
        
        // Pass the token to other async methods
        var sourceResults = await source.SearchAsync(query, cancellationToken);
        results.AddRange(sourceResults);
    }
    
    return results;
}
```

### Progress Reporting

Communicating progress from long-running operations:

```csharp
public async Task<ProcessingResult> ProcessFilesAsync(
    List<string> filePaths, 
    IProgress<int> progress = null,
    CancellationToken cancellationToken = default)
{
    int totalFiles = filePaths.Count;
    var results = new List<FileResult>();
    
    for (int i = 0; i < totalFiles; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();
        
        var result = await ProcessFileAsync(filePaths[i], cancellationToken);
        results.Add(result);
        
        // Report progress if a progress reporter was provided
        progress?.Report((i + 1) * 100 / totalFiles);
    }
    
    return new ProcessingResult { FileResults = results };
}

// Usage from UI
private async void ProcessButton_Click(object sender, EventArgs e)
{
    ProcessButton.Enabled = false;
    ProgressBar.Value = 0;
    
    var progress = new Progress<int>(value => 
    {
        ProgressBar.Value = value;
    });
    
    try
    {
        var result = await _processor.ProcessFilesAsync(
            _selectedFiles, 
            progress,
            _cancellationTokenSource.Token);
            
        MessageBox.Show($"Processed {result.FileResults.Count} files");
    }
    catch (OperationCanceledException)
    {
        MessageBox.Show("Operation canceled");
    }
    catch (Exception ex)
    {
        MessageBox.Show($"Error: {ex.Message}");
    }
    finally
    {
        ProcessButton.Enabled = true;
    }
}
```

## Common Pitfalls

### Deadlocks

Blocking on asynchronous code can cause deadlocks:

```csharp
// This can deadlock in a UI or ASP.NET application
public void DeadlockExample()
{
    // Blocking call to an async method without ConfigureAwait(false)
    string result = GetDataAsync().Result;
}

public async Task<string> GetDataAsync()
{
    await Task.Delay(1000); // Captures the current synchronization context
    return "Data";
}

// FIX: Use ConfigureAwait(false) or make the calling method async too
public async Task<string> GetDataAsync()
{
    await Task.Delay(1000).ConfigureAwait(false);
    return "Data";
}
```

### Task Creation Anti-patterns

```csharp
// ANTI-PATTERN: Task wrapping
public Task<int> WrapResultInTask()
{
    var result = ComputeValue();
    // Unnecessary task creation
    return Task.FromResult(result);
}

// ANTI-PATTERN: async void
public async void AsyncVoidMethod()
{
    await Task.Delay(1000);
    // Exception here is unobservable and can crash the application
    throw new Exception("Unhandled exception");
}

// ANTI-PATTERN: async without await
public async Task<int> NoAwaitMethod()
{
    // Compiler warning: async method lacks await operators
    return ComputeValue();
}
```

### Resource Management

```csharp
// ANTI-PATTERN: Not disposing HttpClient properly
public async Task<string> BadHttpClientUsage()
{
    // Creating a new HttpClient for each request is inefficient
    var client = new HttpClient();
    return await client.GetStringAsync("https://example.com");
}

// GOOD PATTERN: Reusing HttpClient
public class ApiClient
{
    private static readonly HttpClient _httpClient = new HttpClient();
    
    public async Task<string> GetDataAsync(string url)
    {
        return await _httpClient.GetStringAsync(url);
    }
}
```

### Exception Handling

```csharp
// ANTI-PATTERN: Swallowing exceptions
public async Task<string> SwallowedException()
{
    try
    {
        return await FetchDataAsync();
    }
    catch
    {
        // Bad: Exception information is lost
        return "Default value";
    }
}

// GOOD PATTERN: Proper exception handling
public async Task<string> ProperExceptionHandling()
{
    try
    {
        return await FetchDataAsync();
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to fetch data");
        return "Default value";
    }
}
```

## Real-World Examples

### Web API with Async Database Access

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _repository;
    
    public ProductsController(IProductRepository repository)
    {
        _repository = repository;
    }
    
    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetProducts(
        [FromQuery] string category = null,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 25,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var products = await _repository.GetProductsAsync(
                category, page, pageSize, cancellationToken);
                
            var totalCount = await _repository.GetProductCountAsync(category, cancellationToken);
            
            Response.Headers.Add("X-Total-Count", totalCount.ToString());
            
            return Ok(products.Select(p => new ProductDto
            {
                Id = p.Id,
                Name = p.Name,
                Price = p.Price,
                Category = p.Category
            }));
        }
        catch (OperationCanceledException)
        {
            // The request was canceled
            return StatusCode(499, "Request canceled");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error retrieving products");
            return StatusCode(500, "An error occurred while processing your request");
        }
    }
    
    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDto>> GetProduct(int id, CancellationToken cancellationToken)
    {
        var product = await _repository.GetProductByIdAsync(id, cancellationToken);
        
        if (product == null)
            return NotFound();
            
        return new ProductDto
        {
            Id = product.Id,
            Name = product.Name,
            Price = product.Price,
            Category = product.Category
        };
    }
}
```

### Background Service with Queue Processing

```csharp
public class QueueProcessingService : BackgroundService
{
    private readonly IQueueClient _queueClient;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<QueueProcessingService> _logger;
    private readonly SemaphoreSlim _throttler;
    
    public QueueProcessingService(
        IQueueClient queueClient,
        IServiceProvider serviceProvider,
        ILogger<QueueProcessingService> logger,
        IConfiguration configuration)
    {
        _queueClient = queueClient;
        _serviceProvider = serviceProvider;
        _logger = logger;
        
        int maxConcurrentProcessing = configuration.GetValue("Queue:MaxConcurrentMessages", 10);
        _throttler = new SemaphoreSlim(maxConcurrentProcessing);
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Queue processing service starting");
        
        // Register message handler
        _queueClient.RegisterMessageHandler(
            async (message, token) => await ProcessMessageAsync(message, token),
            new MessageHandlerOptions(ExceptionReceivedHandler)
            {
                MaxConcurrentCalls = 100,
                AutoComplete = false
            });
            
        // Keep the service running until cancellation is requested
        await Task.Delay(Timeout.Infinite, stoppingToken);
    }
    
    private async Task ProcessMessageAsync(Message message, CancellationToken token)
    {
        await _throttler.WaitAsync(token);
        
        try
        {
            _logger.LogInformation("Processing message {MessageId}", message.MessageId);
            
            string messageBody = Encoding.UTF8.GetString(message.Body);
            var orderData = JsonSerializer.Deserialize<OrderProcessingData>(messageBody);
            
            using var scope = _serviceProvider.CreateScope();
            var processor = scope.ServiceProvider.GetRequiredService<IOrderProcessor>();
            
            await processor.ProcessOrderAsync(orderData, token);
            
            // Complete the message
            await _queueClient.CompleteAsync(message.SystemProperties.LockToken);
            
            _logger.LogInformation("Completed message {MessageId}", message.MessageId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing message {MessageId}", message.MessageId);
            
            // Abandon the message to make it available for processing again
            await _queueClient.AbandonAsync(message.SystemProperties.LockToken);
        }
        finally
        {
            _throttler.Release();
        }
    }
    
    private Task ExceptionReceivedHandler(ExceptionReceivedEventArgs exceptionReceivedEventArgs)
    {
        _logger.LogError(
            exceptionReceivedEventArgs.Exception,
            "Message handler encountered an exception. Action: {Action}",
            exceptionReceivedEventArgs.ExceptionReceivedContext.Action);
            
        return Task.CompletedTask;
    }
}
```

### Parallel Data Processing Pipeline

```csharp
public class DataProcessingPipeline<TInput, TOutput>
{
    private readonly Func<TInput, Task<TOutput>> _processor;
    private readonly int _maxDegreeOfParallelism;
    private readonly int _batchSize;
    
    public DataProcessingPipeline(
        Func<TInput, Task<TOutput>> processor,
        int maxDegreeOfParallelism = 10,
        int batchSize = 100)
    {
        _processor = processor;
        _maxDegreeOfParallelism = maxDegreeOfParallelism;
        _batchSize = batchSize;
    }
    
    public async Task<IReadOnlyCollection<TOutput>> ProcessAsync(
        IEnumerable<TInput> inputs,
        IProgress<int> progress = null,
        CancellationToken cancellationToken = default)
    {
        var results = new ConcurrentBag<TOutput>();
        var items = inputs.ToList();
        var totalItems = items.Count;
        var processedItems = 0;
        
        // Process in batches
        foreach (var batch in items.Chunk(_batchSize))
        {
            // Create a semaphore to limit concurrency
            using var semaphore = new SemaphoreSlim(_maxDegreeOfParallelism);
            var batchTasks = new List<Task>();
            
            foreach (var item in batch)
            {
                cancellationToken.ThrowIfCancellationRequested();
                
                // Wait for a slot to be available
                await semaphore.WaitAsync(cancellationToken);
                
                batchTasks.Add(Task.Run(async () =>
                {
                    try
                    {
                        var result = await _processor(item);
                        results.Add(result);
                        
                        var processed = Interlocked.Increment(ref processedItems);
                        progress?.Report(processed * 100 / totalItems);
                    }
                    finally
                    {
                        semaphore.Release();
                    }
                }, cancellationToken));
            }
            
            // Wait for all tasks in the batch to complete
            await Task.WhenAll(batchTasks);
        }
        
        return results.ToArray();
    }
}

// Usage example
public class ImageProcessor
{
    public async Task ProcessImagesAsync(
        string[] filePaths, 
        string outputDirectory,
        IProgress<int> progress = null,
        CancellationToken cancellationToken = default)
    {
        var pipeline = new DataProcessingPipeline<string, string>(
            filePath => ResizeAndSaveImageAsync(filePath, outputDirectory),
            maxDegreeOfParallelism: Environment.ProcessorCount,
            batchSize: 20);
            
        await pipeline.ProcessAsync(filePaths, progress, cancellationToken);
    }
    
    private async Task<string> ResizeAndSaveImageAsync(string filePath, string outputDirectory)
    {
        // Image processing logic
        // ...
        
        string outputPath = Path.Combine(outputDirectory, Path.GetFileName(filePath));
        return outputPath;
    }
}
```

## Further Reading

- [Asynchronous Programming with async and await (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/)
- [Task-based Asynchronous Pattern (TAP) (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap)
- [Async/Await - Best Practices in Asynchronous Programming](https://msdn.microsoft.com/en-us/magazine/jj991977.aspx)
- [Concurrency in C# Cookbook](https://www.oreilly.com/library/view/concurrency-in-c/9781492054498/) by Stephen Cleary
- [Pro Asynchronous Programming with .NET](https://www.apress.com/gp/book/9781430259206) by Richard Blewett and Andrew Clymer

## Related Topics

- [Thread Synchronization](../threading/synchronization.md)
- [Parallel Programming](../threading/parallel-programming.md)
- [Performance Optimization Techniques](../optimization/cpu-bound-optimization.md)
- [Async in ASP.NET Core](../../api-development/rest/async-api-design.md)
- [Scalable Application Architecture](../../architecture/principles/scalability.md)