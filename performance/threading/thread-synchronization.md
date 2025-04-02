---
title: "Thread Synchronization in C#"
date_created: 2023-05-10
date_updated: 2023-05-10
authors: ["Repository Maintainers"]
tags: ["threading", "synchronization", "lock", "mutex", "concurrent", "performance"]
difficulty: "intermediate"
---

# Thread Synchronization in C#

## Overview

Thread synchronization is a critical aspect of multithreaded programming that ensures proper coordination between concurrently executing threads. As applications become more complex and take advantage of parallel processing, understanding synchronization mechanisms becomes essential for writing reliable, efficient, and correct code.

This document explores C# synchronization primitives, patterns, and best practices for effectively coordinating thread access to shared resources, preventing race conditions, and maintaining data integrity in concurrent environments.

## Core Concepts

### The Need for Synchronization

In multithreaded applications, multiple threads may attempt to access and modify shared resources simultaneously, which can lead to:

- **Race Conditions**: When the behavior of a program depends on the relative timing of events, leading to unpredictable results
- **Data Corruption**: When concurrent modifications result in invalid or corrupted data
- **Inconsistent States**: When operations that should be atomic are interrupted, leaving data in an invalid intermediate state

For example, consider this classic race condition:

```csharp
// Shared resource
int _counter = 0;

// Thread 1
void IncrementCounter()
{
    for (int i = 0; i < 100000; i++)
    {
        _counter++; // Non-atomic operation
    }
}

// Thread 2
void IncrementCounter()
{
    for (int i = 0; i < 100000; i++)
    {
        _counter++; // Non-atomic operation
    }
}
```

This example has a race condition because the `_counter++` operation is not atomic—it involves reading the current value, incrementing it, and writing it back. When two threads execute this operation concurrently, they may both read the same value, increment it independently, and write back the same incremented value, effectively losing one of the increments.

### Types of Synchronization

C# provides several types of synchronization mechanisms:

1. **Exclusive Locking**: Ensures only one thread executes a critical section at a time
2. **Non-exclusive Locking**: Allows multiple reader threads but restricts writer threads
3. **Signaling**: Enables threads to wait for and signal events
4. **Non-blocking Synchronization**: Uses atomic operations to coordinate without blocking threads
5. **Concurrent Collections**: Thread-safe collection types for concurrent access

### Synchronization vs. Coordination

It's important to distinguish between synchronization and coordination:

- **Synchronization** focuses on controlling access to shared resources to prevent conflicts
- **Coordination** involves scheduling threads to work together in a specific order or pattern

Both are essential aspects of multithreaded programming, and many primitives in C# support both purposes.

## Exclusive Locking Mechanisms

### The `lock` Statement

The `lock` statement provides a simple way to ensure exclusive access to a critical section:

```csharp
private readonly object _lockObject = new object();
private int _counter = 0;

public void IncrementCounter()
{
    lock (_lockObject)
    {
        _counter++;
    }
}
```

Key points about `lock`:

- It's a syntactic shortcut for `Monitor.Enter` and `Monitor.Exit` with exception handling
- The lock object should be private and not used for any other purpose
- It's reentrant, meaning the same thread can acquire the lock multiple times
- Only use thread-safe operations inside a lock to avoid releasing it prematurely

### Monitor Class

The `Monitor` class provides more control than the `lock` statement:

```csharp
private readonly object _lockObject = new object();
private int _counter = 0;

public void IncrementCounter()
{
    bool lockTaken = false;
    try
    {
        Monitor.Enter(_lockObject, ref lockTaken);
        _counter++;
    }
    finally
    {
        if (lockTaken) Monitor.Exit(_lockObject);
    }
}
```

Additional `Monitor` capabilities:

```csharp
public bool TryIncrementWithTimeout()
{
    // Try to acquire the lock with a timeout
    if (Monitor.TryEnter(_lockObject, 1000)) // 1 second timeout
    {
        try
        {
            _counter++;
            return true;
        }
        finally
        {
            Monitor.Exit(_lockObject);
        }
    }
    return false;
}

public void WaitForCondition()
{
    lock (_lockObject)
    {
        while (!_conditionMet)
        {
            // Release lock and wait to be pulsed
            Monitor.Wait(_lockObject);
        }
        
        // Process when condition is met
        ProcessData();
    }
}

public void SignalConditionMet()
{
    lock (_lockObject)
    {
        _conditionMet = true;
        
        // Signal one waiting thread
        Monitor.Pulse(_lockObject);
        
        // Or signal all waiting threads
        // Monitor.PulseAll(_lockObject);
    }
}
```

### Mutex Class

A `Mutex` is similar to a `lock` but can work across processes:

```csharp
private Mutex _mutex = new Mutex(false, "Global\\MyApplicationMutex");

public void DoExclusiveWork()
{
    try
    {
        // Wait until the mutex is available
        _mutex.WaitOne();
        
        // Exclusive section
        PerformCriticalOperation();
    }
    finally
    {
        // Always release the mutex
        _mutex.ReleaseMutex();
    }
}

// Detect if another instance is running
public bool IsAnotherInstanceRunning()
{
    try
    {
        // Try to create named mutex
        bool createdNew;
        using (var mutex = new Mutex(true, "Global\\MyApplicationMutex", out createdNew))
        {
            return !createdNew;
        }
    }
    catch (UnauthorizedAccessException)
    {
        // Another instance is running with elevated permissions
        return true;
    }
}
```

### SpinLock Struct

`SpinLock` provides a lightweight alternative to `lock` when contention is expected to be brief:

```csharp
private SpinLock _spinLock = new SpinLock(enableThreadOwnerTracking: false);
private int _counter = 0;

public void IncrementCounter()
{
    bool lockTaken = false;
    try
    {
        _spinLock.Enter(ref lockTaken);
        _counter++;
    }
    finally
    {
        if (lockTaken) _spinLock.Exit(false);
    }
}
```

When to use `SpinLock`:
- For very short critical sections
- When lock contention is rare
- On multi-core systems with high-performance requirements
- When lock overhead is a concern

Caution: Excessive spinning can waste CPU cycles and may perform worse than traditional locks in high-contention scenarios.

## Reader-Writer Synchronization

### ReaderWriterLockSlim

`ReaderWriterLockSlim` allows concurrent read access but exclusive write access:

```csharp
private readonly ReaderWriterLockSlim _rwLock = new ReaderWriterLockSlim();
private Dictionary<int, string> _cache = new Dictionary<int, string>();

public string Read(int key)
{
    _rwLock.EnterReadLock();
    try
    {
        return _cache.TryGetValue(key, out string value) ? value : null;
    }
    finally
    {
        _rwLock.ExitReadLock();
    }
}

public void Write(int key, string value)
{
    _rwLock.EnterWriteLock();
    try
    {
        _cache[key] = value;
    }
    finally
    {
        _rwLock.ExitWriteLock();
    }
}

public void UpdateIfExists(int key, string newValue)
{
    // First try with upgradeable read lock
    _rwLock.EnterUpgradeableReadLock();
    try
    {
        if (_cache.ContainsKey(key))
        {
            // Upgrade to write lock
            _rwLock.EnterWriteLock();
            try
            {
                _cache[key] = newValue;
            }
            finally
            {
                _rwLock.ExitWriteLock();
            }
        }
    }
    finally
    {
        _rwLock.ExitUpgradeableReadLock();
    }
}
```

Key points:
- More efficient when reads are more frequent than writes
- Upgradeable read locks prevent common read-check-write patterns from deadlocking
- Always release locks in the opposite order they were acquired
- Use try/finally blocks to ensure locks are released even when exceptions occur

### Asynchronous Reader-Writer Lock

A custom implementation for async/await scenarios:

```csharp
public class AsyncReaderWriterLock
{
    private readonly SemaphoreSlim _readerSemaphore = new SemaphoreSlim(1, 1);
    private readonly SemaphoreSlim _writerSemaphore = new SemaphoreSlim(1, 1);
    private int _readerCount = 0;
    
    public async Task<IDisposable> ReaderLockAsync()
    {
        await _readerSemaphore.WaitAsync();
        try
        {
            if (_readerCount == 0)
                await _writerSemaphore.WaitAsync();
                
            _readerCount++;
        }
        finally
        {
            _readerSemaphore.Release();
        }
        
        return new ReaderReleaser(this);
    }
    
    public async Task<IDisposable> WriterLockAsync()
    {
        await _writerSemaphore.WaitAsync();
        return new WriterReleaser(this);
    }
    
    private void ReleaseReaderLock()
    {
        _readerSemaphore.Wait();
        try
        {
            _readerCount--;
            if (_readerCount == 0)
                _writerSemaphore.Release();
        }
        finally
        {
            _readerSemaphore.Release();
        }
    }
    
    private void ReleaseWriterLock()
    {
        _writerSemaphore.Release();
    }
    
    private class ReaderReleaser : IDisposable
    {
        private readonly AsyncReaderWriterLock _parent;
        
        public ReaderReleaser(AsyncReaderWriterLock parent)
        {
            _parent = parent;
        }
        
        public void Dispose()
        {
            _parent.ReleaseReaderLock();
        }
    }
    
    private class WriterReleaser : IDisposable
    {
        private readonly AsyncReaderWriterLock _parent;
        
        public WriterReleaser(AsyncReaderWriterLock parent)
        {
            _parent = parent;
        }
        
        public void Dispose()
        {
            _parent.ReleaseWriterLock();
        }
    }
}

// Usage
private AsyncReaderWriterLock _asyncLock = new AsyncReaderWriterLock();
private Dictionary<string, object> _data = new Dictionary<string, object>();

public async Task<object> ReadAsync(string key)
{
    using (await _asyncLock.ReaderLockAsync())
    {
        return _data.TryGetValue(key, out var value) ? value : null;
    }
}

public async Task WriteAsync(string key, object value)
{
    using (await _asyncLock.WriterLockAsync())
    {
        _data[key] = value;
    }
}
```

## Signaling Mechanisms

### ManualResetEvent and AutoResetEvent

Event wait handles for thread signaling:

```csharp
// ManualResetEvent - stays signaled until reset
private readonly ManualResetEvent _dataReady = new ManualResetEvent(false);
private readonly AutoResetEvent _workItemReady = new AutoResetEvent(false);
private readonly Queue<WorkItem> _workItems = new Queue<WorkItem>();
private object _lockObject = new object();

public void ProducerThread()
{
    while (true)
    {
        var workItem = GenerateWorkItem();
        
        lock (_lockObject)
        {
            _workItems.Enqueue(workItem);
        }
        
        // Signal that work is available
        _workItemReady.Set(); // Auto-resets after a single thread is released
    }
}

public void ConsumerThread()
{
    while (true)
    {
        // Wait for work to arrive
        _workItemReady.WaitOne();
        
        WorkItem item;
        lock (_lockObject)
        {
            item = _workItems.Dequeue();
        }
        
        ProcessWorkItem(item);
    }
}

public void InitializeData()
{
    LoadData();
    
    // Signal that data is ready for all waiting threads
    _dataReady.Set(); // Stays signaled
}

public void WorkerThread()
{
    // Wait for initialization to complete
    _dataReady.WaitOne();
    
    // Begin processing with initialized data
    StartProcessing();
}

public void Reset()
{
    // Reset the event to non-signaled state
    _dataReady.Reset();
}
```

### ManualResetEventSlim and CountdownEvent

More efficient versions for short wait times:

```csharp
private ManualResetEventSlim _dataReady = new ManualResetEventSlim(false);
private CountdownEvent _taskCompletion = new CountdownEvent(10); // Waits for 10 signals

public void PerformParallelWork()
{
    // Launch tasks
    for (int i = 0; i < 10; i++)
    {
        int taskId = i;
        Task.Run(() => 
        {
            // Wait for the signal to start
            _dataReady.Wait();
            
            // Perform work
            ProcessTask(taskId);
            
            // Signal that this task is complete
            _taskCompletion.Signal();
        });
    }
    
    // Prepare for work
    PrepareData();
    
    // Signal all tasks to start
    _dataReady.Set();
    
    // Wait for all tasks to complete
    _taskCompletion.Wait();
    
    Console.WriteLine("All tasks completed");
}
```

### SemaphoreSlim

Controls access to a resource or pool of resources:

```csharp
private readonly SemaphoreSlim _throttler = new SemaphoreSlim(initialCount: 5, maxCount: 5);
private readonly HttpClient _httpClient = new HttpClient();

public async Task<IEnumerable<string>> FetchUrlsAsync(IEnumerable<string> urls)
{
    var results = new ConcurrentBag<string>();
    var tasks = new List<Task>();
    
    foreach (var url in urls)
    {
        tasks.Add(FetchUrlAsync(url, results));
    }
    
    await Task.WhenAll(tasks);
    return results;
}

private async Task FetchUrlAsync(string url, ConcurrentBag<string> results)
{
    // Wait to enter the semaphore (throttle to max 5 concurrent requests)
    await _throttler.WaitAsync();
    
    try
    {
        // Perform HTTP request
        var response = await _httpClient.GetStringAsync(url);
        results.Add(response);
    }
    finally
    {
        // Always release the semaphore
        _throttler.Release();
    }
}
```

### Barrier

Coordinates multiple threads that need to work in phases:

```csharp
private readonly Barrier _barrier;
private readonly double[][] _matrix;
private readonly int _threadCount;

public ParallelProcessor(int size, int threadCount)
{
    _matrix = new double[size][];
    for (int i = 0; i < size; i++)
    {
        _matrix[i] = new double[size];
    }
    
    _threadCount = threadCount;
    _barrier = new Barrier(threadCount, barrier => 
    {
        // This post-phase action runs after each phase completes
        Console.WriteLine($"Phase {barrier.CurrentPhaseNumber} completed");
    });
}

public void Process()
{
    // Divide work among threads
    int rowsPerThread = _matrix.Length / _threadCount;
    
    var threads = new Thread[_threadCount];
    for (int i = 0; i < _threadCount; i++)
    {
        int threadId = i;
        int startRow = threadId * rowsPerThread;
        int endRow = (threadId == _threadCount - 1) ? _matrix.Length : startRow + rowsPerThread;
        
        threads[i] = new Thread(() => ProcessRange(startRow, endRow));
        threads[i].Start();
    }
    
    foreach (var thread in threads)
    {
        thread.Join();
    }
}

private void ProcessRange(int startRow, int endRow)
{
    // Phase 1: Initialize data
    for (int row = startRow; row < endRow; row++)
    {
        for (int col = 0; col < _matrix[row].Length; col++)
        {
            _matrix[row][col] = GenerateInitialValue(row, col);
        }
    }
    
    // Wait for all threads to complete phase 1
    _barrier.SignalAndWait();
    
    // Phase 2: Process data (requires all data to be initialized)
    for (int row = startRow; row < endRow; row++)
    {
        for (int col = 0; col < _matrix[row].Length; col++)
        {
            _matrix[row][col] = ProcessValue(_matrix, row, col);
        }
    }
    
    // Wait for all threads to complete phase 2
    _barrier.SignalAndWait();
    
    // Phase 3: Finalize data (requires all processing to be complete)
    for (int row = startRow; row < endRow; row++)
    {
        for (int col = 0; col < _matrix[row].Length; col++)
        {
            _matrix[row][col] = FinalizeValue(_matrix, row, col);
        }
    }
    
    // Wait for all threads to complete phase 3
    _barrier.SignalAndWait();
}
```

## Non-blocking Synchronization

### Interlocked Operations

Atomic operations on numeric values:

```csharp
private int _counter = 0;
private long _totalValue = 0;

public void IncrementCounter()
{
    // Atomic increment
    Interlocked.Increment(ref _counter);
}

public void AddToTotal(long value)
{
    // Atomic addition
    Interlocked.Add(ref _totalValue, value);
}

public int GetAndResetCounter()
{
    // Atomic exchange
    return Interlocked.Exchange(ref _counter, 0);
}

public bool UpdateIfMatches(ref int location, int expectedValue, int newValue)
{
    // Atomic compare-and-swap
    return Interlocked.CompareExchange(ref location, newValue, expectedValue) == expectedValue;
}
```

### Memory Barriers

Ensuring visibility of memory operations across threads:

```csharp
private bool _stopRequested = false;
private int _value = 0;

public void WorkerThread()
{
    int cachedValue = 0;
    
    while (!_stopRequested)
    {
        // Do some work
        cachedValue = ComputeValue();
        
        // Ensure the write is visible to all threads
        Interlocked.Exchange(ref _value, cachedValue);
    }
}

public void RequestStop()
{
    // Set the stop flag
    _stopRequested = true;
    
    // Ensure the write is visible to all threads
    Thread.MemoryBarrier();
}

public int GetValue()
{
    // Read with memory barrier to ensure we see the latest value
    return Interlocked.CompareExchange(ref _value, 0, 0);
}
```

### Volatile Fields

Prevent compiler and CPU optimizations that might reorder memory operations:

```csharp
private volatile bool _stopRequested = false;
private volatile int _currentState = 0;

public void WorkerThread()
{
    while (!_stopRequested)
    {
        // Process according to current state
        switch (_currentState)
        {
            case 0:
                ProcessStateZero();
                break;
            case 1:
                ProcessStateOne();
                break;
            case 2:
                ProcessStateTwo();
                break;
        }
    }
}

public void ChangeState(int newState)
{
    _currentState = newState;
}

public void RequestStop()
{
    _stopRequested = true;
}
```

## Concurrent Collections

### ConcurrentDictionary

Thread-safe dictionary with atomic operations:

```csharp
private readonly ConcurrentDictionary<string, User> _userCache = 
    new ConcurrentDictionary<string, User>();

public User GetUser(string userId)
{
    // Thread-safe get or add
    return _userCache.GetOrAdd(userId, id => 
    {
        // This factory function is called only if the key doesn't exist
        return FetchUserFromDatabase(id);
    });
}

public void UpdateUserProperty(string userId, string propertyName, string value)
{
    // Thread-safe update
    _userCache.AddOrUpdate(
        userId,
        // If key doesn't exist, create new user
        id => CreateUserWithProperty(id, propertyName, value),
        // If key exists, update the existing user
        (id, existingUser) => 
        {
            existingUser.UpdateProperty(propertyName, value);
            return existingUser;
        }
    );
}

public bool TryRemoveInactiveUser(string userId, out User removedUser)
{
    return _userCache.TryRemove(userId, out removedUser);
}

public void UpdateUserIfExists(string userId, Action<User> updateAction)
{
    // Update if key exists
    _userCache.TryUpdate(
        userId,
        existingUser => 
        {
            // Create new instance with updates
            var updatedUser = existingUser.Clone();
            updateAction(updatedUser);
            return updatedUser;
        },
        // Compare to current value
        existingUser => existingUser
    );
}
```

### ConcurrentQueue, ConcurrentStack, and ConcurrentBag

Thread-safe collections for different usage patterns:

```csharp
// FIFO (First In, First Out) processing
private readonly ConcurrentQueue<WorkItem> _workQueue = new ConcurrentQueue<WorkItem>();

public void EnqueueWork(WorkItem item)
{
    _workQueue.Enqueue(item);
}

public bool TryProcessNextItem()
{
    if (_workQueue.TryDequeue(out var workItem))
    {
        Process(workItem);
        return true;
    }
    return false;
}

// LIFO (Last In, First Out) processing
private readonly ConcurrentStack<StackFrame> _callStack = new ConcurrentStack<StackFrame>();

public void PushFrame(StackFrame frame)
{
    _callStack.Push(frame);
}

public bool TryProcessTopFrame()
{
    if (_callStack.TryPop(out var frame))
    {
        Process(frame);
        return true;
    }
    return false;
}

// Unordered collection (optimized for scenarios where any thread can produce and consume items)
private readonly ConcurrentBag<LogEntry> _logEntries = new ConcurrentBag<LogEntry>();

public void AddLogEntry(LogEntry entry)
{
    _logEntries.Add(entry);
}

public bool TryProcessAnyEntry()
{
    if (_logEntries.TryTake(out var entry))
    {
        Process(entry);
        return true;
    }
    return false;
}
```

### BlockingCollection

Combines a concurrent collection with blocking and cancellation support:

```csharp
private readonly BlockingCollection<WorkItem> _workItems = 
    new BlockingCollection<WorkItem>(new ConcurrentQueue<WorkItem>(), boundedCapacity: 100);

// Producer thread
public void ProduceWork()
{
    try
    {
        while (true)
        {
            var item = GenerateWorkItem();
            
            // Will block if the collection is at capacity
            _workItems.Add(item);
        }
    }
    catch (OperationCanceledException)
    {
        // Handle cancellation
    }
    finally
    {
        _workItems.CompleteAdding();
    }
}

// Consumer thread
public void ConsumeWork(CancellationToken cancellationToken)
{
    try
    {
        // GetConsumingEnumerable blocks when empty and exits when completed
        foreach (var item in _workItems.GetConsumingEnumerable(cancellationToken))
        {
            ProcessWorkItem(item);
        }
    }
    catch (OperationCanceledException)
    {
        // Handle cancellation
    }
}

// Timed operations
public bool TryProduceWithTimeout(WorkItem item)
{
    return _workItems.TryAdd(item, TimeSpan.FromSeconds(5));
}

public bool TryConsumeWithTimeout(out WorkItem item)
{
    return _workItems.TryTake(out item, TimeSpan.FromSeconds(5));
}
```

## Synchronization Patterns

### Lazy Initialization

Thread-safe initialization of a resource:

```csharp
// Lazy initialization with default thread safety (ExecutionAndPublication)
private readonly Lazy<DatabaseConnection> _dbConnection = 
    new Lazy<DatabaseConnection>(() => new DatabaseConnection("connection-string"));

public DatabaseConnection GetDatabaseConnection()
{
    return _dbConnection.Value; // Thread-safe initialization happens here
}

// Lazy initialization with custom thread safety mode
private readonly Lazy<ExpensiveResource> _resource = 
    new Lazy<ExpensiveResource>(
        valueFactory: () => new ExpensiveResource(),
        mode: LazyThreadSafetyMode.PublicationOnly // Faster but may run factory multiple times
    );

// Double-checked locking pattern (when Lazy<T> is not an option)
private volatile ExpensiveResource _resourceInstance;
private readonly object _resourceLock = new object();

public ExpensiveResource GetResource()
{
    var instance = _resourceInstance;
    
    if (instance == null)
    {
        lock (_resourceLock)
        {
            instance = _resourceInstance;
            
            if (instance == null)
            {
                instance = new ExpensiveResource();
                _resourceInstance = instance;
            }
        }
    }
    
    return instance;
}
```

### Producer-Consumer Pattern

Coordinating producer and consumer threads:

```csharp
public class ProducerConsumer<T>
{
    private readonly BlockingCollection<T> _queue;
    private readonly List<Task> _consumers = new List<Task>();
    private readonly CancellationTokenSource _cts = new CancellationTokenSource();
    
    public ProducerConsumer(int boundedCapacity = -1, int consumerCount = 1)
    {
        _queue = boundedCapacity > 0
            ? new BlockingCollection<T>(boundedCapacity)
            : new BlockingCollection<T>();
            
        // Start consumers
        for (int i = 0; i < consumerCount; i++)
        {
            _consumers.Add(Task.Factory.StartNew(Consume, _cts.Token, 
                TaskCreationOptions.LongRunning, TaskScheduler.Default));
        }
    }
    
    public void Produce(T item)
    {
        _queue.Add(item);
    }
    
    private void Consume()
    {
        try
        {
            foreach (var item in _queue.GetConsumingEnumerable(_cts.Token))
            {
                ProcessItem(item);
            }
        }
        catch (OperationCanceledException)
        {
            // Expected when cancellation is requested
        }
    }
    
    private void ProcessItem(T item)
    {
        // Process the item
    }
    
    public void Complete()
    {
        _queue.CompleteAdding();
        Task.WaitAll(_consumers.ToArray());
    }
    
    public void Cancel()
    {
        _cts.Cancel();
        _queue.CompleteAdding();
    }
    
    public void Dispose()
    {
        Cancel();
        _queue.Dispose();
        _cts.Dispose();
    }
}

// Usage
var pc = new ProducerConsumer<WorkItem>(boundedCapacity: 100, consumerCount: 4);

// Producer code
for (int i = 0; i < 1000; i++)
{
    pc.Produce(new WorkItem { Id = i });
}

// Signal completion
pc.Complete();
```

### Reader-Writer Lock Pattern

Implementing a custom reader-writer lock for specialized scenarios:

```csharp
public class CustomReaderWriterLock
{
    private int _readerCount = 0;
    private readonly object _writerLock = new object();
    private readonly SemaphoreSlim _readLock = new SemaphoreSlim(1, 1);
    
    public void EnterReadLock()
    {
        _readLock.Wait();
        try
        {
            _readerCount++;
            if (_readerCount == 1)
            {
                // First reader acquires the writer lock
                Monitor.Enter(_writerLock);
            }
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

### Memory Usage Considerations

Different synchronization primitives have different memory footprints:

```csharp
// Memory-efficient for large numbers of objects
public class MemoryEfficientStateMachine
{
    // Using int with Interlocked operations instead of lock objects
    private int _state = 0; // 0=Idle, 1=Running, 2=Completed
    
    public bool TryStartProcessing()
    {
        // Atomically change from Idle to Running
        return Interlocked.CompareExchange(ref _state, 1, 0) == 0;
    }
    
    public bool TryCompleteProcessing()
    {
        // Atomically change from Running to Completed
        return Interlocked.CompareExchange(ref _state, 2, 1) == 1;
    }
    
    public bool IsCompleted => Interlocked.CompareExchange(ref _state, 0, 0) == 2;
}

// SpinLock has lower overhead than Monitor when contention is low
public class LightweightSynchronization
{
    private SpinLock _spinLock = new SpinLock(enableThreadOwnerTracking: false);
    
    public void ProcessWithLowOverhead()
    {
        bool lockTaken = false;
        try
        {
            _spinLock.Enter(ref lockTaken);
            // Very quick operation
            QuickProcess();
        }
        finally
        {
            if (lockTaken) _spinLock.Exit(false);
        }
    }
}
```

## Common Pitfalls and Solutions

### Deadlocks

Recognizing and preventing deadlocks:

```csharp
// DEADLOCK EXAMPLE - Circular wait condition
public class DeadlockExample
{
    private readonly object _lockA = new object();
    private readonly object _lockB = new object();
    
    public void Process1()
    {
        lock (_lockA)
        {
            // Simulate work
            Thread.Sleep(100);
            
            lock (_lockB)
            {
                // This might deadlock if Process2 holds _lockB and waits for _lockA
                ProcessData();
            }
        }
    }
    
    public void Process2()
    {
        lock (_lockB)
        {
            // Simulate work
            Thread.Sleep(100);
            
            lock (_lockA)
            {
                // This might deadlock if Process1 holds _lockA and waits for _lockB
                ProcessData();
            }
        }
    }
}

// SOLUTION 1: Consistent lock ordering
public class DeadlockSolution1
{
    private readonly object _lockA = new object();
    private readonly object _lockB = new object();
    
    public void Process1()
    {
        lock (_lockA) // Always acquire locks in same order
        {
            Thread.Sleep(100);
            
            lock (_lockB)
            {
                ProcessData();
            }
        }
    }
    
    public void Process2()
    {
        lock (_lockA) // Same order as Process1
        {
            Thread.Sleep(100);
            
            lock (_lockB)
            {
                ProcessData();
            }
        }
    }
}

// SOLUTION 2: Try-based lock acquisition with timeout
public class DeadlockSolution2
{
    private readonly object _lockA = new object();
    private readonly object _lockB = new object();
    
    public bool Process1()
    {
        if (Monitor.TryEnter(_lockA, 1000)) // 1 second timeout
        {
            try
            {
                Thread.Sleep(100);
                
                if (Monitor.TryEnter(_lockB, 1000))
                {
                    try
                    {
                        ProcessData();
                        return true;
                    }
                    finally
                    {
                        Monitor.Exit(_lockB);
                    }
                }
                return false; // Failed to acquire second lock
            }
            finally
            {
                Monitor.Exit(_lockA);
            }
        }
        return false; // Failed to acquire first lock
    }
}
```

### Race Conditions

Identifying and fixing race conditions:

```csharp
// RACE CONDITION EXAMPLE - Initialization check
public class RaceConditionExample
{
    private ComplexObject _instance;
    
    public ComplexObject GetInstance()
    {
        if (_instance == null) // Race condition here
        {
            _instance = new ComplexObject(); // Multiple threads might execute this
        }
        return _instance;
    }
}

// SOLUTION 1: Lock-based approach
public class RaceConditionSolution1
{
    private ComplexObject _instance;
    private readonly object _lock = new object();
    
    public ComplexObject GetInstance()
    {
        if (_instance == null) // First check (not thread-safe but optimizes performance)
        {
            lock (_lock)
            {
                if (_instance == null) // Second check (thread-safe)
                {
                    _instance = new ComplexObject();
                }
            }
        }
        return _instance;
    }
}

// SOLUTION 2: Interlocked-based approach for simple state
public class RaceConditionSolution2
{
    private int _initialized = 0;
    private ComplexObject _instance;
    
    public ComplexObject GetInstance()
    {
        if (_initialized == 0) // First check (not thread-safe)
        {
            if (Interlocked.CompareExchange(ref _initialized, 1, 0) == 0)
            {
                // Only one thread will enter here
                _instance = new ComplexObject();
            }
            else
            {
                // Wait for initialization to complete
                SpinWait.SpinUntil(() => _instance != null);
            }
        }
        return _instance;
    }
}

// SOLUTION 3: Lazy<T> approach (recommended)
public class RaceConditionSolution3
{
    private readonly Lazy<ComplexObject> _instance = 
        new Lazy<ComplexObject>(() => new ComplexObject(), LazyThreadSafetyMode.ExecutionAndPublication);
    
    public ComplexObject GetInstance()
    {
        return _instance.Value;
    }
}
```

### Priority Inversion

Understanding and mitigating priority inversion:

```csharp
// PRIORITY INVERSION SCENARIO
public class PriorityInversionExample
{
    private readonly object _sharedResource = new object();
    
    public void LowPriorityTask()
    {
        lock (_sharedResource)
        {
            // Low priority task holds the lock for a long time
            SlowOperation();
        }
    }
    
    public void HighPriorityTask()
    {
        // High priority task needs the lock but is blocked by low priority task
        lock (_sharedResource)
        {
            CriticalOperation();
        }
    }
}

// MITIGATION 1: Minimize lock duration
public class PriorityInversionMitigation1
{
    private readonly object _sharedResource = new object();
    
    public void LowPriorityTask()
    {
        // Prepare data outside the lock
        var data = PrepareData();
        
        lock (_sharedResource)
        {
            // Minimize time spent holding the lock
            QuickUpdate(data);
        }
        
        // Complete processing outside the lock
        CompleteProcessing(data);
    }
}

// MITIGATION 2: Use priority boosting (platform-specific)
// Windows uses priority boosting automatically for critical system locks
// but not for application locks

// MITIGATION 3: Use lock timeout and retry
public class PriorityInversionMitigation3
{
    private readonly object _sharedResource = new object();
    
    public bool HighPriorityTask()
    {
        for (int attempt = 0; attempt < 5; attempt++)
        {
            if (Monitor.TryEnter(_sharedResource, 100)) // 100ms timeout
            {
                try
                {
                    CriticalOperation();
                    return true;
                }
                finally
                {
                    Monitor.Exit(_sharedResource);
                }
            }
            
            // If can't acquire lock, do other work or yield
            Thread.Yield();
        }
        
        return false; // Failed after multiple attempts
    }
}
```

### Lock Convoy

Understanding and mitigating lock convoys:

```csharp
// LOCK CONVOY SCENARIO
public class LockConvoyExample
{
    private readonly object _globalLock = new object();
    private readonly Dictionary<int, UserData> _userDataStore = new Dictionary<int, UserData>();
    
    public UserData GetUserData(int userId)
    {
        lock (_globalLock)
        {
            // Frequently accessed by many threads
            if (_userDataStore.TryGetValue(userId, out var data))
            {
                return data;
            }
            
            // Potentially slow operation inside lock
            data = FetchUserFromDatabase(userId);
            _userDataStore[userId] = data;
            return data;
        }
    }
}

// SOLUTION 1: Fine-grained locks
public class LockConvoySolution1
{
    private readonly ReaderWriterLockSlim _storeLock = new ReaderWriterLockSlim();
    private readonly Dictionary<int, UserData> _userDataStore = new Dictionary<int, UserData>();
    
    public UserData GetUserData(int userId)
    {
        // Use read lock for lookups (multiple threads can read)
        _storeLock.EnterReadLock();
        try
        {
            if (_userDataStore.TryGetValue(userId, out var data))
            {
                return data;
            }
        }
        finally
        {
            _storeLock.ExitReadLock();
        }
        
        // Fetch outside any lock
        var newData = FetchUserFromDatabase(userId);
        
        // Use write lock only when updating (exclusive)
        _storeLock.EnterWriteLock();
        try
        {
            // Double-check in case another thread added it
            if (_userDataStore.TryGetValue(userId, out var existingData))
            {
                return existingData;
            }
            
            _userDataStore[userId] = newData;
            return newData;
        }
        finally
        {
            _storeLock.ExitWriteLock();
        }
    }
}

// SOLUTION 2: Partitioned data with multiple locks
public class LockConvoySolution2
{
    private readonly Dictionary<int, UserData>[] _partitions;
    private readonly object[] _locks;
    private readonly int _partitionCount;
    
    public LockConvoySolution2(int partitionCount = 32)
    {
        _partitionCount = partitionCount;
        _partitions = new Dictionary<int, UserData>[partitionCount];
        _locks = new object[partitionCount];
        
        for (int i = 0; i < partitionCount; i++)
        {
            _partitions[i] = new Dictionary<int, UserData>();
            _locks[i] = new object();
        }
    }
    
    private int GetPartitionIndex(int userId)
    {
        return (userId.GetHashCode() & 0x7fffffff) % _partitionCount;
    }
    
    public UserData GetUserData(int userId)
    {
        int partitionIndex = GetPartitionIndex(userId);
        
        lock (_locks[partitionIndex])
        {
            var partition = _partitions[partitionIndex];
            
            if (partition.TryGetValue(userId, out var data))
            {
                return data;
            }
            
            data = FetchUserFromDatabase(userId);
            partition[userId] = data;
            return data;
        }
    }
}

// SOLUTION 3: Use ConcurrentDictionary
public class LockConvoySolution3
{
    private readonly ConcurrentDictionary<int, UserData> _userDataStore = 
        new ConcurrentDictionary<int, UserData>();
    
    public UserData GetUserData(int userId)
    {
        return _userDataStore.GetOrAdd(userId, id => FetchUserFromDatabase(id));
    }
}
```

### Thread Starvation

Preventing thread starvation:

```csharp
// THREAD STARVATION SCENARIO
public class ThreadStarvationExample
{
    private readonly object _lock = new object();
    
    public void PotentiallyLongOperation()
    {
        lock (_lock)
        {
            // If this operation takes too long, other threads can't acquire the lock
            VeryLongOperation();
        }
    }
}

// SOLUTION 1: Break down long operations
public class ThreadStarvationSolution1
{
    private readonly object _lock = new object();
    private List<int> _dataList = new List<int>();
    
    public void ProcessLargeDataSet(IEnumerable<int> data)
    {
        // Process data in chunks, releasing lock between chunks
        foreach (var chunk in data.Chunk(100))
        {
            ProcessChunk(chunk);
        }
    }
    
    private void ProcessChunk(IEnumerable<int> chunk)
    {
        // Process outside lock
        var processedChunk = chunk.Select(x => ProcessItem(x)).ToList();
        
        // Acquire lock only for updating shared state
        lock (_lock)
        {
            _dataList.AddRange(processedChunk);
        }
    }
}

// SOLUTION 2: Use fair locks (not directly supported in .NET)
// Implement a custom fair lock
public class FairLock
{
    private readonly Queue<TaskCompletionSource<bool>> _waiters = 
        new Queue<TaskCompletionSource<bool>>();
    private bool _locked = false;
    private readonly object _lock = new object();
    
    public async Task<IDisposable> LockAsync()
    {
        TaskCompletionSource<bool> waiter = null;
        
        lock (_lock)
        {
            if (!_locked)
            {
                _locked = true;
                return new LockReleaser(this);
            }
            
            waiter = new TaskCompletionSource<bool>();
            _waiters.Enqueue(waiter);
        }
        
        await waiter.Task;
        return new LockReleaser(this);
    }
    
    private void Release()
    {
        TaskCompletionSource<bool> nextWaiter = null;
        
        lock (_lock)
        {
            if (_waiters.Count > 0)
            {
                nextWaiter = _waiters.Dequeue();
            }
            else
            {
                _locked = false;
            }
        }
        
        nextWaiter?.SetResult(true);
    }
    
    private class LockReleaser : IDisposable
    {
        private readonly FairLock _lock;
        
        public LockReleaser(FairLock fairLock)
        {
            _lock = fairLock;
        }
        
        public void Dispose()
        {
            _lock.Release();
        }
    }
}

// SOLUTION 3: Use timeouts and backoff
public class ThreadStarvationSolution3
{
    private readonly ReaderWriterLockSlim _rwLock = new ReaderWriterLockSlim();
    
    public bool TryProcess(int maxWaitTime)
    {
        // Try to acquire lock with timeout
        if (_rwLock.TryEnterWriteLock(maxWaitTime))
        {
            try
            {
                ProcessData();
                return true;
            }
            finally
            {
                _rwLock.ExitWriteLock();
            }
        }
        
        // If we couldn't get the lock, perform low-priority work instead
        PerformAlternativeWork();
        return false;
    }
}
```

## Real-World Examples

### Thread-Safe Cache with Expiration

```csharp
public class ThreadSafeCache<TKey, TValue>
{
    private class CacheItem
    {
        public TValue Value { get; }
        public DateTime ExpiresAt { get; }
        
        public CacheItem(TValue value, TimeSpan expiration)
        {
            Value = value;
            ExpiresAt = DateTime.UtcNow.Add(expiration);
        }
        
        public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
    }
    
    private readonly ConcurrentDictionary<TKey, CacheItem> _cache = 
        new ConcurrentDictionary<TKey, CacheItem>();
    private readonly Timer _cleanupTimer;
    private readonly TimeSpan _defaultExpiration;
    
    public ThreadSafeCache(TimeSpan? defaultExpiration = null, TimeSpan? cleanupInterval = null)
    {
        _defaultExpiration = defaultExpiration ?? TimeSpan.FromMinutes(10);
        var interval = cleanupInterval ?? TimeSpan.FromMinutes(1);
        
        _cleanupTimer = new Timer(CleanupExpiredItems, null, interval, interval);
    }
    
    public TValue GetOrAdd(TKey key, Func<TKey, TValue> valueFactory, TimeSpan? expiration = null)
    {
        // Check if item exists and is not expired
        if (_cache.TryGetValue(key, out var existingItem) && !existingItem.IsExpired)
        {
            return existingItem.Value;
        }
        
        // Create new item
        var newItem = new CacheItem(
            valueFactory(key),
            expiration ?? _defaultExpiration
        );
        
        // Add or update cache
        _cache[key] = newItem;
        return newItem.Value;
    }
    
    public bool TryGetValue(TKey key, out TValue value)
    {
        if (_cache.TryGetValue(key, out var item) && !item.IsExpired)
        {
            value = item.Value;
            return true;
        }
        
        value = default;
        return false;
    }
    
    public void Remove(TKey key)
    {
        _cache.TryRemove(key, out _);
    }
    
    public void Clear()
    {
        _cache.Clear();
    }
    
    private void CleanupExpiredItems(object state)
    {
        // Find all expired keys
        var expiredKeys = _cache
            .Where(kvp => kvp.Value.IsExpired)
            .Select(kvp => kvp.Key)
            .ToList();
        
        // Remove expired items
        foreach (var key in expiredKeys)
        {
            _cache.TryRemove(key, out _);
        }
    }
    
    public void Dispose()
    {
        _cleanupTimer?.Dispose();
    }
}

// Usage example
public class CacheExample
{
    private readonly ThreadSafeCache<string, User> _userCache = 
        new ThreadSafeCache<string, User>(TimeSpan.FromMinutes(30));
    private readonly IUserRepository _userRepository;
    
    public CacheExample(IUserRepository userRepository)
    {
        _userRepository = userRepository;
    }
    
    public User GetUser(string userId)
    {
        return _userCache.GetOrAdd(userId, id => _userRepository.GetById(id));
    }
    
    public void UpdateUser(User user)
    {
        _userRepository.Update(user);
        
        // Invalidate cache
        _userCache.Remove(user.Id);
    }
}
```

### Thread-Safe State Machine

```csharp
public class ThreadSafeStateMachine<TState, TTrigger> where TState : struct where TTrigger : struct
{
    private readonly ReaderWriterLockSlim _stateLock = new ReaderWriterLockSlim();
    private TState _currentState;
    
    private readonly Dictionary<(TState State, TTrigger Trigger), TState> _transitions =
        new Dictionary<(TState, TTrigger), TState>();
    
    private readonly Dictionary<(TState From, TState To), Action> _entryActions =
        new Dictionary<(TState, TState), Action>();
    
    private readonly Dictionary<(TState From, TState To), Action> _exitActions =
        new Dictionary<(TState, TState), Action>();
    
    public ThreadSafeStateMachine(TState initialState)
    {
        _currentState = initialState;
    }
    
    public TState CurrentState
    {
        get
        {
            _stateLock.EnterReadLock();
            try
            {
                return _currentState;
            }
            finally
            {
                _stateLock.ExitReadLock();
            }
        }
    }
    
    public void ConfigureTransition(TState fromState, TTrigger trigger, TState toState)
    {
        _stateLock.EnterWriteLock();
        try
        {
            _transitions[(fromState, trigger)] = toState;
        }
        finally
        {
            _stateLock.ExitWriteLock();
        }
    }
    
    public void ConfigureEntryAction(TState fromState, TState toState, Action action)
    {
        _stateLock.EnterWriteLock();
        try
        {
            _entryActions[(fromState, toState)] = action;
        }
        finally
        {
            _stateLock.ExitWriteLock();
        }
    }
    
    public void ConfigureExitAction(TState fromState, TState toState, Action action)
    {
        _stateLock.EnterWriteLock();
        try
        {
            _exitActions[(fromState, toState)] = action;
        }
        finally
        {
            _stateLock.ExitWriteLock();
        }
    }
    
    public bool TryFire(TTrigger trigger)
    {
        _stateLock.EnterUpgradeableReadLock();
        try
        {
            var currentState = _currentState;
            
            if (!_transitions.TryGetValue((currentState, trigger), out var newState))
            {
                return false;
            }
            
            _stateLock.EnterWriteLock();
            try
            {
                // Execute exit action if defined
                if (_exitActions.TryGetValue((currentState, newState), out var exitAction))
                {
                    exitAction();
                }
                
                // Change state
                _currentState = newState;
                
                // Execute entry action if defined
                if (_entryActions.TryGetValue((currentState, newState), out var entryAction))
                {
                    entryAction();
                }
                
                return true;
            }
            finally
            {
                _stateLock.ExitWriteLock();
            }
        }
        finally
        {
            _stateLock.ExitUpgradeableReadLock();
        }
    }
}

// Usage example
public enum OrderState { Created, Processing, Shipped, Delivered, Cancelled }
public enum OrderTrigger { Process, Ship, Deliver, Cancel }

public class OrderProcessor
{
    private readonly ThreadSafeStateMachine<OrderState, OrderTrigger> _stateMachine;
    
    public OrderProcessor()
    {
        _stateMachine = new ThreadSafeStateMachine<OrderState, OrderTrigger>(OrderState.Created);
        
        // Configure transitions
        _stateMachine.ConfigureTransition(OrderState.Created, OrderTrigger.Process, OrderState.Processing);
        _stateMachine.ConfigureTransition(OrderState.Processing, OrderTrigger.Ship, OrderState.Shipped);
        _stateMachine.ConfigureTransition(OrderState.Shipped, OrderTrigger.Deliver, OrderState.Delivered);
        _stateMachine.ConfigureTransition(OrderState.Created, OrderTrigger.Cancel, OrderState.Cancelled);
        _stateMachine.ConfigureTransition(OrderState.Processing, OrderTrigger.Cancel, OrderState.Cancelled);
        
        // Configure actions
        _stateMachine.ConfigureEntryAction(OrderState.Created, OrderState.Processing, 
            () => Console.WriteLine("Order processing started"));
            
        _stateMachine.ConfigureEntryAction(OrderState.Processing, OrderState.Shipped,
            () => Console.WriteLine("Order has been shipped"));
            
        _stateMachine.ConfigureEntryAction(OrderState.Shipped, OrderState.Delivered,
            () => Console.WriteLine("Order has been delivered"));
            
        _stateMachine.ConfigureEntryAction(OrderState.Created, OrderState.Cancelled,
            () => Console.WriteLine("Order cancelled before processing"));
            
        _stateMachine.ConfigureEntryAction(OrderState.Processing, OrderState.Cancelled,
            () => Console.WriteLine("Order cancelled during processing"));
    }
    
    public OrderState CurrentState => _stateMachine.CurrentState;
    
    public bool TryProcess() => _stateMachine.TryFire(OrderTrigger.Process);
    public bool TryShip() => _stateMachine.TryFire(OrderTrigger.Ship);
    public bool TryDeliver() => _stateMachine.TryFire(OrderTrigger.Deliver);
    public bool TryCancel() => _stateMachine.TryFire(OrderTrigger.Cancel);
}
```

### Asynchronous Task Processor with Rate Limiting

```csharp
public class AsyncTaskProcessor<T>
{
    private readonly Func<T, CancellationToken, Task> _processor;
    private readonly SemaphoreSlim _throttler;
    private readonly BlockingCollection<T> _queue = new BlockingCollection<T>();
    private readonly CancellationTokenSource _cts = new CancellationTokenSource();
    private readonly Task _processingTask;
    private readonly int _maxRetries;
    private readonly ILogger _logger;
    
    public AsyncTaskProcessor(
        Func<T, CancellationToken, Task> processor,
        int maxConcurrency = 4,
        int maxRetries = 3,
        ILogger logger = null)
    {
        _processor = processor;
        _throttler = new SemaphoreSlim(maxConcurrency, maxConcurrency);
        _maxRetries = maxRetries;
        _logger = logger;
        
        // Start processing loop
        _processingTask = Task.Run(ProcessItemsAsync);
    }
    
    public void Enqueue(T item)
    {
        _queue.Add(item);
    }
    
    public void CompleteAdding()
    {
        _queue.CompleteAdding();
    }
    
    public async Task WaitForCompletionAsync()
    {
        await _processingTask;
    }
    
    public void Cancel()
    {
        _cts.Cancel();
    }
    
    private async Task ProcessItemsAsync()
    {
        var tasks = new List<Task>();
        
        try
        {
            foreach (var item in _queue.GetConsumingEnumerable(_cts.Token))
            {
                // Wait for a slot to be available
                await _throttler.WaitAsync(_cts.Token);
                
                // Process item in separate task
                tasks.Add(Task.Run(async () =>
                {
                    try
                    {
                        await ProcessWithRetriesAsync(item, _cts.Token);
                    }
                    finally
                    {
                        _throttler.Release();
                    }
                }));
            }
            
            // Wait for all tasks to complete
            await Task.WhenAll(tasks);
        }
        catch (OperationCanceledException)
        {
            // Expected when cancellation is requested
        }
    }
    
    private async Task ProcessWithRetriesAsync(T item, CancellationToken cancellationToken)
    {
        int attempt = 0;
        bool success = false;
        
        while (!success && attempt < _maxRetries && !cancellationToken.IsCancellationRequested)
        {
            try
            {
                attempt++;
                
                if (attempt > 1)
                {
                    // Exponential backoff
                    int delayMs = (int)Math.Pow(2, attempt - 1) * 100;
                    await Task.Delay(delayMs, cancellationToken);
                }
                
                await _processor(item, cancellationToken);
                success = true;
            }
            catch (Exception ex) when (!(ex is OperationCanceledException))
            {
                _logger?.LogError(ex, "Error processing item on attempt {Attempt}/{MaxRetries}", 
                    attempt, _maxRetries);
                
                if (attempt >= _maxRetries)
                {
                    _logger?.LogError("Max retries reached for item, giving up");
                }
            }
        }
    }
    
    public void Dispose()
    {
        _cts.Cancel();
        _queue.Dispose();
        _throttler.Dispose();
        _cts.Dispose();
    }
}

// Usage example
public class TaskProcessorExample
{
    private readonly AsyncTaskProcessor<EmailMessage> _emailProcessor;
    
    public TaskProcessorExample(IEmailSender emailSender, ILogger<TaskProcessorExample> logger)
    {
        _emailProcessor = new AsyncTaskProcessor<EmailMessage>(
            // Processing function
            async (email, token) => await emailSender.SendEmailAsync(email, token),
            
            // Configuration
            maxConcurrency: 10,
            maxRetries: 5,
            logger: logger
        );
    }
    
    public void QueueEmail(EmailMessage email)
    {
        _emailProcessor.Enqueue(email);
    }
    
    public void Shutdown()
    {
        _emailProcessor.CompleteAdding();
        _emailProcessor.WaitForCompletionAsync().Wait(TimeSpan.FromSeconds(30));
    }
}
```

## Further Reading

- [Threading in C# (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/standard/threading/)
- [Synchronization Primitives (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/standard/threading/overview-of-synchronization-primitives)
- [Thread Synchronization (C# In Depth)](https://csharpindepth.com/Articles/Threads)
- [Concurrent Collections (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/standard/collections/thread-safe/)
- [Threading in C# (Joseph Albahari)](http://www.albahari.com/threading/)
- [Concurrency in C# Cookbook (O'Reilly)](https://www.oreilly.com/library/view/concurrency-in-c/9781491906675/) by Stephen Cleary
- [Concurrent Programming on Windows (Addison-Wesley)](https://www.pearson.com/us/higher-education/program/Duffy-Concurrent-Programming-on-Windows/PGM270267.html) by Joe Duffy

## Related Topics

- [Async/Await Deep Dive](../async/async-await-deep-dive.md)
- [Task Parallel Library](../async/task-parallel-library.md)
- [Parallel Programming](parallel-programming.md)
- [Thread Pool](thread-pool.md)
- [Memory Optimization](../optimization/memory-optimization.md)
- [CPU-Bound Optimization](../optimization/cpu-bound-optimization.md)

            _readLock.Release();
        }
    }
    
    public void ExitReadLock()
    {
        _readLock.Wait();
        try
        {
            _readerCount--;
            if (_readerCount == 0)
            {
                // Last reader releases the writer lock
                Monitor.Exit(_writerLock);
            }
        }
        finally
        {
            _readLock.Release();
        }
    }
    
    public void EnterWriteLock()
    {
        Monitor.Enter(_writerLock);
    }
    
    public void ExitWriteLock()
    {
        Monitor.Exit(_writerLock);
    }
    
    // Provides convenient interface with IDisposable pattern
    public IDisposable AcquireReaderLock()
    {
        EnterReadLock();
        return new LockReleaser(this, isWriter: false);
    }
    
    public IDisposable AcquireWriterLock()
    {
        EnterWriteLock();
        return new LockReleaser(this, isWriter: true);
    }
    
    private class LockReleaser : IDisposable
    {
        private readonly CustomReaderWriterLock _lock;
        private readonly bool _isWriter;
        
        public LockReleaser(CustomReaderWriterLock rwLock, bool isWriter)
        {
            _lock = rwLock;
            _isWriter = isWriter;
        }
        
        public void Dispose()
        {
            if (_isWriter)
                _lock.ExitWriteLock();
            else
                _lock.ExitReadLock();
        }
    }
}

// Usage
private readonly CustomReaderWriterLock _rwLock = new CustomReaderWriterLock();
private readonly Dictionary<int, string> _data = new Dictionary<int, string>();

public string ReadValue(int key)
{
    using (_rwLock.AcquireReaderLock())
    {
        return _data.TryGetValue(key, out var value) ? value : null;
    }
}

public void WriteValue(int key, string value)
{
    using (_rwLock.AcquireWriterLock())
    {
        _data[key] = value;
    }
}
```

### Thread-Safe Singleton Pattern

Implementing a thread-safe singleton:

```csharp
public sealed class Singleton
{
    // Lazy<T> implementation (recommended)
    private static readonly Lazy<Singleton> _lazyInstance = 
        new Lazy<Singleton>(() => new Singleton());
        
    public static Singleton Instance => _lazyInstance.Value;
    
    private Singleton()
    {
        // Private constructor
    }
    
    // Alternative: static initialization (also thread-safe)
    private static readonly Singleton _staticInstance = new Singleton();
    public static Singleton StaticInstance => _staticInstance;
    
    // Alternative: double-checked locking pattern
    private static volatile Singleton _volatileInstance;
    private static readonly object _lockObject = new object();
    
    public static Singleton VolatileInstance
    {
        get
        {
            if (_volatileInstance == null)
            {
                lock (_lockObject)
                {
                    if (_volatileInstance == null)
                    {
                        _volatileInstance = new Singleton();
                    }
                }
            }
            return _volatileInstance;
        }
    }
}
```

## Best Practices

### Minimize Lock Scope

Keep critical sections as small as possible:

```csharp
// BAD - lock held during expensive computation
public void ProcessDataBad(List<int> data)
{
    lock (_lockObject)
    {
        var result = ExpensiveComputation(data); // Holds lock during computation
        _results.Add(result);
    }
}

// GOOD - lock held only during data access
public void ProcessDataGood(List<int> data)
{
    // Do computation outside the lock
    var result = ExpensiveComputation(data);
    
    // Lock only when updating shared data
    lock (_lockObject)
    {
        _results.Add(result);
    }
}
```

### Avoid Nested Locks

Be cautious with nested locks to prevent deadlocks:

```csharp
// BAD - potential deadlock
public void TransferFundsBad(Account from, Account to, decimal amount)
{
    lock (from)
    {
        lock (to) // Deadlock if another thread locks in opposite order
        {
            from.Withdraw(amount);
            to.Deposit(amount);
        }
    }
}

// GOOD - consistent lock ordering
public void TransferFundsGood(Account from, Account to, decimal amount)
{
    // Lock accounts in a consistent order based on ID to prevent deadlocks
    if (from.Id < to.Id)
    {
        lock (from)
        {
            lock (to)
            {
                PerformTransfer(from, to, amount);
            }
        }
    }
    else
    {
        lock (to)
        {
            lock (from)
            {
                PerformTransfer(from, to, amount);
            }
        }
    }
}
```

### Use Lock Timeouts

Detect and handle deadlocks with timeouts:

```csharp
public bool TryProcessWithTimeout(WorkItem item)
{
    bool lockTaken = false;
    
    try
    {
        // Try to acquire lock with timeout
        Monitor.TryEnter(_lockObject, TimeSpan.FromSeconds(5), ref lockTaken);
        
        if (lockTaken)
        {
            ProcessItem(item);
            return true;
        }
        else
        {
            // Failed to acquire lock within timeout
            LogPotentialDeadlock("Processing", item.Id);
            return false;
        }
    }
    finally
    {
        if (lockTaken)
        {
            Monitor.Exit(_lockObject);
        }
    }
}
```

### Prefer Higher-Level Synchronization

Use concurrent collections and coordination primitives when possible:

```csharp
// BAD - reinventing thread-safe collection
public class ManualThreadSafeList<T>
{
    private readonly List<T> _items = new List<T>();
    private readonly object _lock = new object();
    
    public void Add(T item)
    {
        lock (_lock)
        {
            _items.Add(item);
        }
    }
    
    public bool Remove(T item)
    {
        lock (_lock)
        {
            return _items.Remove(item);
        }
    }
    
    public List<T> GetSnapshot()
    {
        lock (_lock)
        {
            return new List<T>(_items);
        }
    }
}

// GOOD - using built-in concurrent collection
public class BetterThreadSafeList<T>
{
    private readonly ConcurrentBag<T> _items = new ConcurrentBag<T>();
    
    public void Add(T item)
    {
        _items.Add(item);
    }
    
    public bool TryRemove(out T item)
    {
        return _items.TryTake(out item);
    }
    
    public List<T> GetSnapshot()
    {
        return _items.ToList();
    }
}
```

### Choose the Right Synchronization Primitive

Select the appropriate primitive for the scenario:

```csharp
// For simple mutual exclusion
private readonly object _lockObject = new object();

// For reader-writer scenarios
private readonly ReaderWriterLockSlim _rwLock = new ReaderWriterLockSlim();

// For limited concurrent operations
private readonly SemaphoreSlim _throttler = new SemaphoreSlim(10);

// For signaling between threads
private readonly ManualResetEventSlim _dataReady = new ManualResetEventSlim(false);

// For atomic operations
private int _counter = 0; // Use with Interlocked methods

// For thread-safe collections
private readonly ConcurrentDictionary<string, User> _userCache = new ConcurrentDictionary<string, User>();

// For producer-consumer scenarios
private readonly BlockingCollection<WorkItem> _workItems = new BlockingCollection<WorkItem>();

// For lazy initialization
private readonly Lazy<ExpensiveResource> _resource = new Lazy<ExpensiveResource>();
```

### Prefer Async Synchronization for I/O Operations

Use async-compatible primitives for I/O-bound operations:

```csharp
// Synchronous primitives (avoid for I/O operations)
private readonly object _lockObject = new object();
private readonly ReaderWriterLockSlim _rwLock = new ReaderWriterLockSlim();

// Asynchronous primitives (better for I/O operations)
private readonly SemaphoreSlim _asyncLock = new SemaphoreSlim(1, 1);
private readonly SemaphoreSlim _asyncThrottler = new SemaphoreSlim(10, 10);

public async Task ProcessItemAsync(WorkItem item)
{
    // Acquire async lock
    await _asyncLock.WaitAsync();
    try
    {
        // Critical section with async operations
        await SaveItemAsync(item);
    }
    finally
    {
        _asyncLock.Release();
    }
}

public async Task ProcessMultipleItemsAsync(IEnumerable<WorkItem> items)
{
    var tasks = new List<Task>();
    
    foreach (var item in items)
    {
        // Control concurrency
        await _asyncThrottler.WaitAsync();
        
        tasks.Add(Task.Run(async () =>
        {
            try
            {
                await ProcessSingleItemAsync(item);
            }
            finally
            {
                _asyncThrottler.Release();
            }
        }));
    }
    
    await Task.WhenAll(tasks);
}
```

## Performance Considerations

### Lock Contention and Granularity

Fine-tune lock granularity for better performance:

```csharp
// BAD - coarse-grained locking (high contention)
public class CoarseGrainedCache<TKey, TValue>
{
    private readonly Dictionary<TKey, TValue> _cache = new Dictionary<TKey, TValue>();
    private readonly object _lock = new object();
    
    public TValue GetOrAdd(TKey key, Func<TKey, TValue> valueFactory)
    {
        lock (_lock) // Single lock for all operations
        {
            if (!_cache.TryGetValue(key, out var value))
            {
                value = valueFactory(key);
                _cache[key] = value;
            }
            return value;
        }
    }
}

// GOOD - fine-grained locking (reduced contention)
public class FineGrainedCache<TKey, TValue>
{
    private readonly ConcurrentDictionary<TKey, TValue> _cache = new ConcurrentDictionary<TKey, TValue>();
    private readonly ConcurrentDictionary<TKey, SemaphoreSlim> _locks = 
        new ConcurrentDictionary<TKey, SemaphoreSlim>();
    
    public async Task<TValue> GetOrAddAsync(TKey key, Func<TKey, Task<TValue>> valueFactory)
    {
        // Fast path - check if already in cache
        if (_cache.TryGetValue(key, out var cachedValue))
        {
            return cachedValue;
        }
        
        // Get or create a lock for this specific key
        var keyLock = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));
        
        await keyLock.WaitAsync();
        try
        {
            // Double-check after acquiring lock
            if (_cache.TryGetValue(key, out cachedValue))
            {
                return cachedValue;
            }
            
            // Generate and cache the value
            var newValue = await valueFactory(key);
            _cache[key] = newValue;
            return newValue;
        }
        finally
        {
            keyLock.Release();
            
            // Clean up the lock if not needed
            if (_cache.ContainsKey(key) && _locks.TryRemove(key, out var removedLock))
            {
                removedLock.Dispose();
            }
        }
    }
}
```

### Choosing Between Lock Types

Different locks have different performance characteristics:

```csharp
public class LockPerformanceComparison
{
    // Lock statement - general purpose, reentrant, good performance
    private readonly object _standardLock = new object();
    
    // SpinLock - high performance for very short operations, not reentrant
    private SpinLock _spinLock = new SpinLock(enableThreadOwnerTracking: false);
    
    // Mutex - cross-process capable but slower
    private readonly Mutex _mutex = new Mutex();
    
    // SemaphoreSlim - efficient for async code
    private readonly SemaphoreSlim _semaphore = new SemaphoreSlim(1, 1);
    
    public void StandardLockMethod()
    {
        lock (_standardLock)
        {
            // Short operations (sub-millisecond)
            DoWork();
        }
    }
    
    public void SpinLockMethod()
    {
        bool lockTaken = false;
        try
        {
            _spinLock.Enter(ref lockTaken);
            // Very short operations (microseconds)
            DoVeryQuickWork();
        }
        finally
        {
            if (lockTaken) _spinLock.Exit(false);
        }
    }
    
    public void MutexMethod()
    {
        _mutex.WaitOne();
        try
        {
            // Operations that need cross-process coordination
            DoWork();
        }
        finally
        {
            _mutex.ReleaseMutex();
        }
    }
    
    public async Task SemaphoreSlimMethodAsync()
    {
        await _semaphore.WaitAsync();
        try
        {
            // Async operations
            await DoWorkAsync();
        }
        finally
        ---
title: "Thread Synchronization in C#"
date_created: 2023-05-10
date_updated: 2023-05-10
authors: ["Repository Maintainers"]
tags: ["threading", "synchronization", "lock", "mutex", "concurrent", "performance"]
difficulty: "intermediate"
---

# Thread Synchronization in C#

## Overview

Thread synchronization is a critical aspect of multithreaded programming that ensures proper coordination between concurrently executing threads. As applications become more complex and take advantage of parallel processing, understanding synchronization mechanisms becomes essential for writing reliable, efficient, and correct code.

This document explores C# synchronization primitives, patterns, and best practices for effectively coordinating thread access to shared resources, preventing race conditions, and maintaining data integrity in concurrent environments.

## Core Concepts

### The Need for Synchronization

In multithreaded applications, multiple threads may attempt to access and modify shared resources simultaneously, which can lead to:

- **Race Conditions**: When the behavior of a program depends on the relative timing of events, leading to unpredictable results
- **Data Corruption**: When concurrent modifications result in invalid or corrupted data
- **Inconsistent States**: When operations that should be atomic are interrupted, leaving data in an invalid intermediate state

For example, consider this classic race condition:

```csharp
// Shared resource
int _counter = 0;

// Thread 1
void IncrementCounter()
{
    for (int i = 0; i < 100000; i++)
    {
        _counter++; // Non-atomic operation
    }
}

// Thread 2
void IncrementCounter()
{
    for (int i = 0; i < 100000; i++)
    {
        _counter++; // Non-atomic operation
    }
}
```

This example has a race condition because the `_counter++` operation is not atomic—it involves reading the current value, incrementing it, and writing it back. When two threads execute this operation concurrently, they may both read the same value, increment it independently, and write back the same incremented value, effectively losing one of the increments.

### Types of Synchronization

C# provides several types of synchronization mechanisms:

1. **Exclusive Locking**: Ensures only one thread executes a critical section at a time
2. **Non-exclusive Locking**: Allows multiple reader threads but restricts writer threads
3. **Signaling**: Enables threads to wait for and signal events
4. **Non-blocking Synchronization**: Uses atomic operations to coordinate without blocking threads
5. **Concurrent Collections**: Thread-safe collection types for concurrent access

### Synchronization vs. Coordination

It's important to distinguish between synchronization and coordination:

- **Synchronization** focuses on controlling access to shared resources to prevent conflicts
- **Coordination** involves scheduling threads to work together in a specific order or pattern

Both are essential aspects of multithreaded programming, and many primitives in C# support both purposes.

## Exclusive Locking Mechanisms

### The `lock` Statement

The `lock` statement provides a simple way to ensure exclusive access to a critical section:

```csharp
private readonly object _lockObject = new object();
private int _counter = 0;

public void IncrementCounter()
{
    lock (_lockObject)
    {
        _counter++;
    }
}
```

Key points about `lock`:

- It's a syntactic shortcut for `Monitor.Enter` and `Monitor.Exit` with exception handling
- The lock object should be private and not used for any other purpose
- It's reentrant, meaning the same thread can acquire the lock multiple times
- Only use thread-safe operations inside a lock to avoid releasing it prematurely

### Monitor Class

The `Monitor` class provides more control than the `lock` statement:

```csharp
private readonly object _lockObject = new object();
private int _counter = 0;

public void IncrementCounter()
{
    bool lockTaken = false;
    try
    {
        Monitor.Enter(_lockObject, ref lockTaken);
        _counter++;
    }
    finally
    {
        if (lockTaken) Monitor.Exit(_lockObject);
    }
}
```

Additional `Monitor` capabilities:

```csharp
public bool TryIncrementWithTimeout()
{
    // Try to acquire the lock with a timeout
    if (Monitor.TryEnter(_lockObject, 1000)) // 1 second timeout
    {
        try
        {
            _counter++;
            return true;
        }
        finally
        {
            Monitor.Exit(_lockObject);
        }
    }
    return false;
}

public void WaitForCondition()
{
    lock (_lockObject)
    {
        while (!_conditionMet)
        {
            // Release lock and wait to be pulsed
            Monitor.Wait(_lockObject);
        }
        
        // Process when condition is met
        ProcessData();
    }
}

public void SignalConditionMet()
{
    lock (_lockObject)
    {
        _conditionMet = true;
        
        // Signal one waiting thread
        Monitor.Pulse(_lockObject);
        
        // Or signal all waiting threads
        // Monitor.PulseAll(_lockObject);
    }
}
```

### Mutex Class

A `Mutex` is similar to a `lock` but can work across processes:

```csharp
private Mutex _mutex = new Mutex(false, "Global\\MyApplicationMutex");

public void DoExclusiveWork()
{
    try
    {
        // Wait until the mutex is available
        _mutex.WaitOne();
        
        // Exclusive section
        PerformCriticalOperation();
    }
    finally
    {
        // Always release the mutex
        _mutex.ReleaseMutex();
    }
}

// Detect if another instance is running
public bool IsAnotherInstanceRunning()
{
    try
    {
        // Try to create named mutex
        bool createdNew;
        using (var mutex = new Mutex(true, "Global\\MyApplicationMutex", out createdNew))
        {
            return !createdNew;
        }
    }
    catch (UnauthorizedAccessException)
    {
        // Another instance is running with elevated permissions
        return true;
    }
}
```

### SpinLock Struct

`SpinLock` provides a lightweight alternative to `lock` when contention is expected to be brief:

```csharp
private SpinLock _spinLock = new SpinLock(enableThreadOwnerTracking: false);
private int _counter = 0;

public void IncrementCounter()
{
    bool lockTaken = false;
    try
    {
        _spinLock.Enter(ref lockTaken);
        _counter++;
    }
    finally
    {
        if (lockTaken) _spinLock.Exit(false);
    }
}
```

When to use `SpinLock`:
- For very short critical sections
- When lock contention is rare
- On multi-core systems with high-performance requirements
- When lock overhead is a concern

Caution: Excessive spinning can waste CPU cycles and may perform worse than traditional locks in high-contention scenarios.

## Reader-Writer Synchronization

### ReaderWriterLockSlim

`ReaderWriterLockSlim` allows concurrent read access but exclusive write access:

```csharp
private readonly ReaderWriterLockSlim _rwLock = new ReaderWriterLockSlim();
private Dictionary<int, string> _cache = new Dictionary<int, string>();

public string Read(int key)
{
    _rwLock.EnterReadLock();
    try
    {
        return _cache.TryGetValue(key, out string value) ? value : null;
    }
    finally
    {
        _rwLock.ExitReadLock();
    }
}

public void Write(int key, string value)
{
    _rwLock.EnterWriteLock();
    try
    {
        _cache[key] = value;
    }
    finally
    {
        _rwLock.ExitWriteLock();
    }
}

public void UpdateIfExists(int key, string newValue)
{
    // First try with upgradeable read lock
    _rwLock.EnterUpgradeableReadLock();
    try
    {
        if (_cache.ContainsKey(key))
        {
            // Upgrade to write lock
            _rwLock.EnterWriteLock();
            try
            {
                _cache[key] = newValue;
            }
            finally
            {
                _rwLock.ExitWriteLock();
            }
        }
    }
    finally
    {
        _rwLock.ExitUpgradeableReadLock();
    }
}
```

Key points:
- More efficient when reads are more frequent than writes
- Upgradeable read locks prevent common read-check-write patterns from deadlocking
- Always release locks in the opposite order they were acquired
- Use try/finally blocks to ensure locks are released even when exceptions occur

### Asynchronous Reader-Writer Lock

A custom implementation for async/await scenarios:

```csharp
public class AsyncReaderWriterLock
{
    private readonly SemaphoreSlim _readerSemaphore = new SemaphoreSlim(1, 1);
    private readonly SemaphoreSlim _writerSemaphore = new SemaphoreSlim(1, 1);
    private int _readerCount = 0;
    
    public async Task<IDisposable> ReaderLockAsync()
    {
        await _readerSemaphore.WaitAsync();
        try
        {
            if (_readerCount == 0)
                await _writerSemaphore.WaitAsync();
                
            _readerCount++;
        }
        finally
        {
            _readerSemaphore.Release();
        }
        
        return new ReaderReleaser(this);
    }
    
    public async Task<IDisposable> WriterLockAsync()
    {
        await _writerSemaphore.WaitAsync();
        return new WriterReleaser(this);
    }
    
    private void ReleaseReaderLock()
    {
        _readerSemaphore.Wait();
        try
        {
            _readerCount--;
            if (_readerCount == 0)
                _writerSemaphore.Release();
        }
        finally
        {
            _readerSemaphore.Release();
        }
    }
    
    private void ReleaseWriterLock()
    {
        _writerSemaphore.Release();
    }
    
    private class ReaderReleaser : IDisposable
    {
        private readonly AsyncReaderWriterLock _parent;
        
        public ReaderReleaser(AsyncReaderWriterLock parent)
        {
            _parent = parent;
        }
        
        public void Dispose()
        {
            _parent.ReleaseReaderLock();
        }
    }
    
    private class WriterReleaser : IDisposable
    {
        private readonly AsyncReaderWriterLock _parent;
        
        public WriterReleaser(AsyncReaderWriterLock parent)
        {
            _parent = parent;
        }
        
        public void Dispose()
        {
            _parent.ReleaseWriterLock();
        }
    }
}

// Usage
private AsyncReaderWriterLock _asyncLock = new AsyncReaderWriterLock();
private Dictionary<string, object> _data = new Dictionary<string, object>();

public async Task<object> ReadAsync(string key)
{
    using (await _asyncLock.ReaderLockAsync())
    {
        return _data.TryGetValue(key, out var value) ? value : null;
    }
}

public async Task WriteAsync(string key, object value)
{
    using (await _asyncLock.WriterLockAsync())
    {
        _data[key] = value;
    }
}
```

## Signaling Mechanisms

### ManualResetEvent and AutoResetEvent

Event wait handles for thread signaling:

```csharp
// ManualResetEvent - stays signaled until reset
private