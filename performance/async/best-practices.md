---
title: "Asynchronous Programming Best Practices in C#"
date_created: 2023-04-10
date_updated: 2023-04-10
authors: ["Repository Maintainers"]
tags: ["async", "await", "performance", "best practices", "task"]
difficulty: "intermediate"
---

# Asynchronous Programming Best Practices in C#

## Overview

Asynchronous programming in C# enables applications to remain responsive while performing potentially long-running operations. When implemented correctly, async code improves scalability and resource utilization. However, improper implementation can lead to hard-to-diagnose bugs, performance issues, and even application deadlocks. This document outlines best practices for writing efficient, reliable, and maintainable asynchronous code in C#.

## Naming Conventions

### Use Async Suffix

Always append the "Async" suffix to method names that return `Task` or `Task<T>`:

```csharp
// GOOD
public async Task<string> GetUserDataAsync()
{
    return await _repository.GetUserByIdAsync(1);
}

// BAD - missing Async suffix
public async Task<string> GetUserData()
{
    return await _repository.GetUserByIdAsync(1);
}
```

### Provide Synchronous Alternatives When Appropriate

If an operation can be performed both synchronously and asynchronously, provide both methods:

```csharp
// Synchronous version
public string GetData()
{
    return File.ReadAllText("data.txt");
}

// Asynchronous version
public async Task<string> GetDataAsync()
{
    return await File.ReadAllTextAsync("data.txt");
}
```

## Task Return Types

### Return Task, Not void

Avoid `async void` methods except for event handlers:

```csharp
// GOOD - returns Task
public async Task ProcessDataAsync()
{
    await Task.Delay(1000);
    // Process data
}

// BAD - async void can't be awaited or have exceptions properly caught
public async void ProcessData()
{
    await Task.Delay(1000);
    // Process data
}

// ACCEPTABLE - event handler
private async void Button_Click(object sender, EventArgs e)
{
    try
    {
        await ProcessDataAsync();
    }
    catch (Exception ex)
    {
        // Handle exception
        LogError(ex);
    }
}
```

### Use ValueTask for Performance When Appropriate

Use `ValueTask<T>` instead of `Task<T>` when a method is likely to complete synchronously often:

```csharp
private Dictionary<string, User> _cache = new Dictionary<string, User>();

// GOOD - uses ValueTask to avoid allocations for cache hits
public ValueTask<User> GetUserAsync(string userId)
{
    if (_cache.TryGetValue(userId, out User cachedUser))
    {
        // Synchronous path - no Task allocation
        return new ValueTask<User>(cachedUser);
    }
    
    // Asynchronous path - falls back to Task
    return new ValueTask<User>(FetchUserFromDatabaseAsync(userId));
}
```

### Return Completed Tasks When Possible

For methods that complete synchronously but still need to return a Task, use `Task.FromResult` instead of using `async`/`await`:

```csharp
// GOOD - no state machine overhead
public Task<int> GetCountAsync()
{
    if (_cache.TryGetValue("count", out int count))
    {
        return Task.FromResult(count);
    }
    
    return CalculateCountAsync();
}

// BAD - unnecessary async/await overhead
public async Task<int> GetCountAsync()
{
    if (_cache.TryGetValue("count", out int count))
    {
        // Unnecessary state machine created for a synchronous result
        return count;
    }
    
    return await CalculateCountAsync();
}
```

## Task Composition

### Use Task.WhenAll for Parallel Execution

When executing multiple independent async operations, use `Task.WhenAll` to run them in parallel:

```csharp
// GOOD - runs all tasks in parallel
public async Task<DashboardViewModel> GetDashboardDataAsync(string userId)
{
    Task<UserProfile> profileTask = _userService.GetProfileAsync(userId);
    Task<List<Order>> ordersTask = _orderService.GetRecentOrdersAsync(userId);
    Task<List<Notification>> notificationsTask = _notificationService.GetNotificationsAsync(userId);
    
    // Wait for all tasks to complete
    await Task.WhenAll(profileTask, ordersTask, notificationsTask);
    
    return new DashboardViewModel
    {
        Profile = await profileTask,
        RecentOrders = await ordersTask,
        Notifications = await notificationsTask
    };
}

// BAD - sequential execution
public async Task<DashboardViewModel> GetDashboardDataSequentialAsync(string userId)
{
    var profile = await _userService.GetProfileAsync(userId);
    var orders = await _orderService.GetRecentOrdersAsync(userId);
    var notifications = await _notificationService.GetNotificationsAsync(userId);
    
    return new DashboardViewModel
    {
        Profile = profile,
        RecentOrders = orders,
        Notifications = notifications
    };
}
```

### Use Task.WhenAny for Timeouts or Cancellation

When you need to implement timeouts or respond to the first completed task:

```csharp
public async Task<string> GetWithTimeoutAsync(string url, TimeSpan timeout)
{
    using var client = new HttpClient();
    
    Task<string> downloadTask = client.GetStringAsync(url);
    Task timeoutTask = Task.Delay(timeout);
    
    Task completedTask = await Task.WhenAny(downloadTask, timeoutTask);
    
    if (completedTask == timeoutTask)
    {
        throw new TimeoutException($"The operation timed out after {timeout.TotalSeconds} seconds");
    }
    
    return await downloadTask; // Unwrap the result
}
```

### Combine Tasks for Complex Workflows

Use LINQ and task composition for more complex parallel operations:

```csharp
public async Task<List<ProcessResult>> ProcessItemsAsync(List<Item> items)
{
    // Create and start all tasks
    var tasks = items.Select(item => ProcessItemAsync(item)).ToList();
    
    // Process in batches of 10 concurrent operations
    var results = new List<ProcessResult>();
    
    while (tasks.Any())
    {
        // Take at most 10 tasks
        var batch = tasks.Take(10).ToList();
        tasks = tasks.Skip(10).ToList();
        
        // Wait for the current batch to complete
        await Task.WhenAll(batch);
        
        // Add results
        results.AddRange(batch.Select(t => t.Result));
    }
    
    return results;
}
```

## Error Handling

### Always Handle Exceptions in Async Methods

Unhandled exceptions in async methods can crash your application:

```csharp
// GOOD - properly handle exceptions
public async Task<User> GetUserAsync(string userId)
{
    try
    {
        return await _repository.GetUserByIdAsync(userId);
    }
    catch (NotFoundException)
    {
        // Handle specific exceptions
        return null;
    }
    catch (Exception ex) when (IsTransientException(ex))
    {
        // Log and retry for transient exceptions
        _logger.LogWarning(ex, "Transient error retrieving user");
        await Task.Delay(500);
        return await _repository.GetUserByIdAsync(userId);
    }
    catch (Exception ex)
    {
        // Log unexpected exceptions
        _logger.LogError(ex, "Error retrieving user");
        throw; // Rethrow to propagate
    }
}
```

### Use Exception Filters for Logging Without Unwinding the Stack

C# 6.0 introduced exception filters which allow logging without affecting the stack trace:

```csharp
public async Task ProcessTransactionAsync(Transaction transaction)
{
    try
    {
        await _transactionService.ProcessAsync(transaction);
    }
    catch (Exception ex) when (LogException(ex))
    {
        // This code never executes because LogException returns false
        // But the exception gets logged without unwinding the stack
    }
}

private bool LogException(Exception ex)
{
    _logger.LogError(ex, "Error during transaction processing");
    return false; // Continue the search for a handler
}
```

### Handle AggregateException When Using Task.WhenAll

`Task.WhenAll` throws only the first exception, but others may have occurred:

```csharp
public async Task ProcessMultipleRequestsAsync(List<string> urls)
{
    var tasks = urls.Select(url => ProcessUrlAsync(url)).ToList();
    
    try
    {
        await Task.WhenAll(tasks);
    }
    catch
    {
        // Find all exceptions that occurred
        var exceptions = tasks
            .Where(t => t.IsFaulted)
            .Select(t => t.Exception.InnerException)
            .ToList();
            
        foreach (var ex in exceptions)
        {
            _logger.LogError(ex, "Error processing URL");
        }
        
        throw; // Rethrow if necessary
    }
}
```

## Context and ConfigureAwait

### Use ConfigureAwait(false) in Library Code

In library code, use `ConfigureAwait(false)` to avoid forcing continuation on the original context:

```csharp
// GOOD - library method that doesn't need context
public async Task<int> LibraryMethodAsync()
{
    // Use ConfigureAwait(false) for all awaits in libraries
    var data = await _dataService.GetDataAsync().ConfigureAwait(false);
    var result = await ProcessDataAsync(data).ConfigureAwait(false);
    return result;
}

// BAD - capturing context unnecessarily
public async Task<int> LibraryMethodWithContextAsync()
{
    // This will capture and resume on the original SynchronizationContext
    var data = await _dataService.GetDataAsync();
    var result = await ProcessDataAsync(data);
    return result;
}
```

### Don't Use ConfigureAwait(false) in Application UI Code

In UI applications, you typically want to return to the UI thread:

```csharp
// GOOD - UI method that needs to update UI
private async void Button_Click(object sender, EventArgs e)
{
    Button.Enabled = false;
    
    try
    {
        // Without ConfigureAwait(false) to ensure continuation on UI thread
        var result = await _dataService.GetDataAsync();
        ResultTextBox.Text = result; // This needs UI thread
    }
    finally
    {
        Button.Enabled = true;
    }
}
```

### Be Consistent with ConfigureAwait

If you use `ConfigureAwait(false)`, use it consistently for all awaits in the same method:

```csharp
// GOOD - consistent use of ConfigureAwait(false)
public async Task<DataResult> GetAndProcessDataAsync()
{
    var rawData = await FetchDataAsync().ConfigureAwait(false);
    var processedData = await ProcessDataAsync(rawData).ConfigureAwait(false);
    return new DataResult { Data = processedData };
}

// BAD - inconsistent use can lead to subtle bugs
public async Task<DataResult> InconsistentContextAsync()
{
    var rawData = await FetchDataAsync().ConfigureAwait(false);
    var processedData = await ProcessDataAsync(rawData); // Inconsistent!
    return new DataResult { Data = processedData };
}
```

## Cancellation Support

### Always Support Cancellation

Support cancellation in all asynchronous operations that might take significant time:

```csharp
public async Task<List<Customer>> GetCustomersAsync(
    string searchTerm = null, 
    CancellationToken cancellationToken = default)
{
    // Check cancellation early
    cancellationToken.ThrowIfCancellationRequested();
    
    // Use cancellation token with all async calls
    var customers = await _dbContext.Customers
        .Where(c => searchTerm == null || c.Name.Contains(searchTerm))
        .ToListAsync(cancellationToken);
    
    // Can check cancellation between processing steps
    cancellationToken.ThrowIfCancellationRequested();
    
    return await EnrichCustomerDataAsync(customers, cancellationToken);
}
```

### Propagate Cancellation Tokens

Pass cancellation tokens through the call stack:

```csharp
public async Task<OrderResult> PlaceOrderAsync(
    Order order, 
    CancellationToken cancellationToken = default)
{
    // Validate order (pass cancellation token)
    await ValidateOrderAsync(order, cancellationToken);
    
    // Process payment (pass cancellation token)
    await _paymentService.ProcessPaymentAsync(order.Payment, cancellationToken);
    
    // Create order (pass cancellation token)
    var orderId = await _orderRepository.CreateOrderAsync(order, cancellationToken);
    
    // Notify user (pass cancellation token)
    await _notificationService.SendOrderConfirmationAsync(orderId, cancellationToken);
    
    return new OrderResult { OrderId = orderId, Success = true };
}
```

### Implement Timeouts Using CancellationTokens

Combine timeout and cancellation using `CancellationTokenSource`:

```csharp
public async Task<string> GetWithTimeoutAsync(
    string url, 
    TimeSpan timeout,
    CancellationToken userCancellationToken = default)
{
    // Create linked token source that combines user token and timeout
    using var timeoutCts = new CancellationTokenSource(timeout);
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        userCancellationToken, timeoutCts.Token);
    
    try
    {
        using var client = new HttpClient();
        return await client.GetStringAsync(url, linkedCts.Token);
    }
    catch (OperationCanceledException)
    {
        if (timeoutCts.IsCancellationRequested)
        {
            throw new TimeoutException($"The operation timed out after {timeout.TotalSeconds} seconds");
        }
        
        // Propagate regular cancellation
        throw;
    }
}
```

## Progress Reporting

### Use IProgress<T> for Progress Updates

Implement progress reporting with `IProgress<T>`:

```csharp
public async Task<ProcessingResult> ProcessFilesAsync(
    List<string> filePaths,
    IProgress<int> progress = null,
    CancellationToken cancellationToken = default)
{
    int totalFiles = filePaths.Count;
    var results = new List<FileProcessingResult>();
    
    for (int i = 0; i < totalFiles; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();
        
        // Process each file
        var result = await ProcessFileAsync(filePaths[i], cancellationToken);
        results.Add(result);
        
        // Report progress if a progress reporter is provided
        progress?.Report((i + 1) * 100 / totalFiles);
    }
    
    return new ProcessingResult 
    { 
        FileResults = results, 
        TotalProcessed = results.Count 
    };
}
```

Usage in a UI application:

```csharp
private async void ProcessButton_Click(object sender, EventArgs e)
{
    ProcessButton.Enabled = false;
    ProgressBar.Value = 0;
    
    try
    {
        // Create a progress reporter that updates the UI
        var progress = new Progress<int>(percent => {
            ProgressBar.Value = percent;
            StatusLabel.Text = $"Processing: {percent}%";
        });
        
        // Process files with progress reporting
        var result = await _fileProcessor.ProcessFilesAsync(
            _selectedFiles, 
            progress, 
            _cancellationToken);
            
        StatusLabel.Text = $"Processed {result.TotalProcessed} files successfully";
    }
    catch (OperationCanceledException)
    {
        StatusLabel.Text = "Operation was canceled";
    }
    catch (Exception ex)
    {
        StatusLabel.Text = $"Error: {ex.Message}";
    }
    finally
    {
        ProcessButton.Enabled = true;
    }
}
```

## Resource Management

### Properly Dispose of Async Resources

Use `IAsyncDisposable` and `await using` for async resources:

```csharp
// GOOD - proper async disposal
public async Task ProcessDataAsync()
{
    await using var resource = new AsyncResource();
    await resource.InitializeAsync();
    await resource.ProcessAsync();
    // IAsyncDisposable.DisposeAsync is called automatically
}

// Resource implementing IAsyncDisposable
public class AsyncResource : IAsyncDisposable
{
    private DbConnection _connection;
    private bool _disposed;
    
    public async Task InitializeAsync()
    {
        _connection = new SqlConnection("connection-string");
        await _connection.OpenAsync();
    }
    
    public async Task ProcessAsync()
    {
        if (_disposed) throw new ObjectDisposedException(nameof(AsyncResource));
        // Use the connection
    }
    
    public async ValueTask DisposeAsync()
    {
        if (!_disposed)
        {
            if (_connection != null)
            {
                await _connection.DisposeAsync();
            }
            
            _disposed = true;
        }
    }
}
```

### Reuse HttpClient Instances

Avoid creating and disposing HttpClient instances frequently:

```csharp
// BAD - creating new HttpClient for each request
public async Task<string> GetDataBadPracticeAsync(string url)
{
    using var client = new HttpClient(); // Poor practice!
    return await client.GetStringAsync(url);
}

// GOOD - reuse HttpClient instance
public class ApiClient
{
    private static readonly HttpClient _httpClient = new HttpClient();
    
    public async Task<string> GetDataAsync(string url)
    {
        return await _httpClient.GetStringAsync(url);
    }
}

// BETTER - use IHttpClientFactory in ASP.NET Core
public class ApiClientWithFactory
{
    private readonly IHttpClientFactory _clientFactory;
    
    public ApiClientWithFactory(IHttpClientFactory clientFactory)
    {
        _clientFactory = clientFactory;
    }
    
    public async Task<string> GetDataAsync(string url)
    {
        var client = _clientFactory.CreateClient("ApiClient");
        return await client.GetStringAsync(url);
    }
}
```

## Performance Considerations

### Avoid Unnecessary Async/Await

Only use async/await when you need to await asynchronous operations:

```csharp
// BAD - unnecessary async overhead
public async Task<int> BadComputeAsync(int value)
{
    // This just performs CPU-bound work, no need for async
    return await Task.FromResult(value * value);
}

// GOOD - synchronous method for CPU-bound work
public int Compute(int value)
{
    return value * value;
}

// GOOD - wrapping CPU-bound work for UI responsiveness
public Task<int> ComputeOnThreadPoolAsync(int value)
{
    return Task.Run(() => Compute(value));
}
```

### Avoid Blocking on Async Code

Never block on async code using `.Result` or `.Wait()`:

```csharp
// BAD - can cause deadlock
public string GetDataBlocking()
{
    // This can deadlock if called from UI or ASP.NET thread
    return GetDataAsync().Result;
}

// GOOD - async all the way
public async Task<string> GetDataAsync()
{
    return await _service.FetchDataAsync();
}

// GOOD - if you must block (e.g., in Main), use GetAwaiter().GetResult()
public string GetDataThreadSafe()
{
    return Task.Run(() => GetDataAsync()).GetAwaiter().GetResult();
}
```

### Optimize Task Creation

Minimize task allocations for better performance:

```csharp
// BAD - creates a new task unnecessarily
public Task<int> BadFactorialAsync(int n)
{
    if (n <= 1)
    {
        // Unnecessary task creation
        return Task.FromResult(1);
    }
    
    return FactorialImplAsync(n);
}

// GOOD - reuses cached tasks for common values
public Task<int> GoodFactorialAsync(int n)
{
    if (n <= 1)
    {
        // Reuse a cached task
        return Task.FromResult(1);
    }
    
    return FactorialImplAsync(n);
}

// BETTER - uses ValueTask to avoid allocations
public ValueTask<int> OptimizedFactorialAsync(int n)
{
    if (n <= 1)
    {
        // No allocation at all
        return new ValueTask<int>(1);
    }
    
    return new ValueTask<int>(FactorialImplAsync(n));
}
```

### Use Concurrent Collections for Thread Safety

When multiple async operations access shared data:

```csharp
// GOOD - uses thread-safe collection
public class ConcurrentCache
{
    private readonly ConcurrentDictionary<string, Task<object>> _cache = 
        new ConcurrentDictionary<string, Task<object>>();
    
    public Task<object> GetOrAddAsync(string key, Func<string, Task<object>> valueFactory)
    {
        // Thread-safe, concurrent operations can share this cache
        return _cache.GetOrAdd(key, k => valueFactory(k));
    }
}
```

## Testing Async Code

### Use Async Test Methods

Write async test methods for testing async code:

```csharp
[Fact]
public async Task GetUser_WhenUserExists_ReturnsUser()
{
    // Arrange
    var repository = new UserRepository();
    var userId = "user123";
    
    // Act
    var user = await repository.GetUserByIdAsync(userId);
    
    // Assert
    Assert.NotNull(user);
    Assert.Equal(userId, user.Id);
}
```

### Test Cancellation Scenarios

Test that your code correctly handles cancellation:

```csharp
[Fact]
public async Task GetData_WhenCancelled_ThrowsOperationCanceledException()
{
    // Arrange
    var service = new DataService();
    var cts = new CancellationTokenSource();
    
    // Act & Assert
    var dataTask = service.GetDataAsync(cts.Token);
    cts.Cancel();
    
    await Assert.ThrowsAsync<OperationCanceledException>(() => dataTask);
}
```

### Use Task Completion Sources for Testing Async Code

Use TaskCompletionSource to control async behavior in tests:

```csharp
public class TestAsyncHelper
{
    public static Task<T> CreateCompletedTask<T>(T result)
    {
        return Task.FromResult(result);
    }
    
    public static Task CreateFaultedTask(Exception exception)
    {
        var tcs = new TaskCompletionSource<object>();
        tcs.SetException(exception);
        return tcs.Task;
    }
    
    public static Task CreateCanceledTask()
    {
        var tcs = new TaskCompletionSource<object>();
        tcs.SetCanceled();
        return tcs.Task;
    }
    
    public static async Task<T> WithTimeout<T>(Task<T> task, TimeSpan timeout)
    {
        if (await Task.WhenAny(task, Task.Delay(timeout)) != task)
        {
            throw new TimeoutException($"Task did not complete within {timeout}");
        }
        
        return await task;
    }
}
```

## Common Anti-patterns

### Hidden Async Void

Watch out for indirect async void methods:

```csharp
// BAD - hidden async void
public void StartProcess()
{
    // This is an async void method in disguise!
    Task.Run(async () => 
    {
        await ProcessDataAsync();
    });
}

// GOOD - properly handled async operation
public Task StartProcessAsync()
{
    return ProcessDataAsync();
}

// ALTERNATIVE - if you need fire-and-forget with error handling
public void StartBackgroundProcess()
{
    Task.Run(async () => 
    {
        try
        {
            await ProcessDataAsync();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Background process failed");
        }
    });
}
```

### Async Over-Optimization

Don't complicate code with premature optimizations:

```csharp
// BAD - over-engineered for a negligible gain
public ValueTask<User> OverOptimizedGetUserAsync(string userId)
{
    if (_localCache.TryGetValue(userId, out User user))
    {
        return new ValueTask<User>(user);
    }
    
    if (_distributedCache.TryGetValue(userId, out user))
    {
        _localCache[userId] = user;
        return new ValueTask<User>(user);
    }
    
    if (_secondaryCache.TryGetValue(userId, out Task<User> pendingTask) && 
        !pendingTask.IsCompleted)
    {
        return new ValueTask<User>(pendingTask);
    }
    
    var task = GetUserFromDatabaseAsync(userId);
    _secondaryCache[userId] = task;
    return new ValueTask<User>(task);
}

// GOOD - clear, maintainable, still performant
public async Task<User> GetUserAsync(string userId)
{
    // Check cache first
    if (_cache.TryGetValue(userId, out User user))
    {
        return user;
    }
    
    // Get from database and cache
    user = await _repository.GetUserByIdAsync(userId);
    if (user != null)
    {
        _cache[userId] = user;
    }
    
    return user;
}
```

### Excessive Parallelism

Avoid excessive parallelism without throttling:

```csharp
// BAD - uncontrolled parallelism
public async Task ProcessAllItemsAsync(List<Item> items)
{
    // This will start ALL tasks at once, potentially overwhelming resources
    var tasks = items.Select(item => ProcessItemAsync(item));
    await Task.WhenAll(tasks);
}

// GOOD - controlled parallelism
public async Task ProcessAllItemsThrottledAsync(List<Item> items)
{
    // Process in batches of 10 at a time
    var semaphore = new SemaphoreSlim(10);
    var tasks = new List<Task>();
    
    foreach (var item in items)
    {
        await semaphore.WaitAsync();
        
        tasks.Add(Task.Run(async () =>
        {
            try
            {
                await ProcessItemAsync(item);
            }
            finally
            {
                semaphore.Release();
            }
        }));
    }
    
    await Task.WhenAll(tasks);
}
```

## Real-World Examples

### Retry Pattern with Exponential Backoff

```csharp
public async Task<T> RetryWithBackoffAsync<T>(
    Func<Task<T>> operation,
    int maxRetries = 3,
    CancellationToken cancellationToken = default)
{
    int retryCount = 0;
    Exception lastException = null;
    
    while (retryCount < maxRetries)
    {
        try
        {
            if (retryCount > 0)
            {
                // Calculate delay with exponential backoff and jitter
                var delay = TimeSpan.FromSeconds(Math.Pow(2, retryCount) + Random.Shared.NextDouble());
                await Task.Delay(delay, cancellationToken);
            }
            
            return await operation();
        }
        catch (Exception ex) when (IsTransientException(ex) && retryCount < maxRetries - 1)
        {
            lastException = ex;
            retryCount++;
            _logger.LogWarning(ex, $"Operation failed, retrying ({retryCount}/{maxRetries})...");
        }
    }
    
    throw new AggregateException($"Operation failed after {maxRetries} attempts", lastException);
}

private bool IsTransientException(Exception ex)
{
    return ex is HttpRequestException ||
           ex is TimeoutException ||
           ex is SqlException sqlEx && (sqlEx.Number == 1205 || sqlEx.Number == -2);
}
```

### Throttled Parallel Processing

```csharp
public class ParallelProcessor<T, TResult>
{
    private readonly Func<T, Task<TResult>> _processor;
    private readonly int _maxConcurrency;
    
    public ParallelProcessor(Func<T, Task<TResult>> processor, int maxConcurrency = 10)
    {
        _processor = processor;
        _maxConcurrency = maxConcurrency;
    }
    
    public async Task<IReadOnlyList<TResult>> ProcessAsync(
        IEnumerable<T> items, 
        IProgress<int> progress = null,
        CancellationToken cancellationToken = default)
    {
        var results = new ConcurrentBag<TResult>();
        var itemList = items.ToList();
        var processed = 0;
        var total = itemList.Count;
        
        await Parallel.ForEachAsync(
            itemList,
            new ParallelOptions
            {
                MaxDegreeOfParallelism = _maxConcurrency,
                CancellationToken = cancellationToken
            },
            async (item, token) =>
            {
                var result = await _processor(item);
                results.Add(result);
                
                var count = Interlocked.Increment(ref processed);
                progress?.Report(count * 100 / total);
            });
        
        return results.ToList();
    }
}
```

### ASP.NET Core Controller With Best Practices

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    private readonly ILogger<UsersController> _logger;
    
    public UsersController(
        IUserService userService,
        ILogger<UsersController> logger)
    {
        _userService = userService;
        _logger = logger;
    }
    
    [HttpGet]
    public async Task<ActionResult<List<UserDto>>> GetUsers(
        [FromQuery] string searchTerm = null,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var users = await _userService.GetUsersAsync(
                searchTerm, page, pageSize, cancellationToken);
                
            return Ok(users);
        }
        catch (OperationCanceledException) when (cancellationToken.IsCancellationRequested)
        {
            // Don't log cancellations as errors
            return StatusCode(499); // Client Closed Request
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error retrieving users");
            return StatusCode(500, "An error occurred while processing your request");
        }
    }
    
    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> GetUser(
        string id,
        CancellationToken cancellationToken)
    {
        try
        {
            var user = await _userService.GetUserByIdAsync(id, cancellationToken);
            
            if (user == null)
                return NotFound();
                
            return Ok(user);
        }
        catch (OperationCanceledException) when (cancellationToken.IsCancellationRequested)
        {
            return StatusCode(499);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error retrieving user {UserId}", id);
            return StatusCode(500, "An error occurred while processing your request");
        }
    }
    
    [HttpPost]
    public async Task<ActionResult<UserDto>> CreateUser(
        [FromBody] CreateUserRequest request,
        CancellationToken cancellationToken)
    {
        try
        {
            var user = await _userService.CreateUserAsync(request, cancellationToken);
            return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
        }
        catch (ValidationException ex)
        {
            return BadRequest(ex.Message);
        }
        catch (OperationCanceledException) when (cancellationToken.IsCancellationRequested)
        {
            return StatusCode(499);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating user");
            return StatusCode(500, "An error occurred while processing your request");
        }
    }
}
```

## Further Reading

- [Asynchronous Programming with async and await (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/)
- [Task-based Asynchronous Pattern (TAP) (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap)
- [Asynchronous Best Practices (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/best-practices-for-asynchronous-programming)
- [Debugging Async Methods (Microsoft Docs)](https://docs.microsoft.com/en-us/visualstudio/debugger/navigating-through-code-with-the-debugger)
- [Concurrency in C# Cookbook](https://www.oreilly.com/library/view/concurrency-in-c/9781492054498/) by Stephen Cleary

## Related Topics

- [Async/Await Deep Dive](async-await-deep-dive.md)
- [Task Parallel Library](task-parallel-library.md)
- [Thread Synchronization](../threading/thread-synchronization.md)
- [Memory Optimization](../optimization/memory-optimization.md)
- [CPU-Bound Optimization](../optimization/cpu-bound-optimization.md)
- [Network Optimization](../optimization/network-optimization.md)