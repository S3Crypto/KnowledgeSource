---
title: "Task Parallel Library (TPL) in C#"
date_created: 2023-05-14
date_updated: 2023-05-14
authors: ["Repository Maintainers"]
tags: ["tpl", "parallel", "task", "threading", "concurrency", "performance"]
difficulty: "intermediate"
---

# Task Parallel Library (TPL) in C#

## Overview

The Task Parallel Library (TPL) is a set of APIs introduced in .NET Framework 4.0 that provides abstractions for writing concurrent and parallel code in C#. TPL dramatically simplifies the process of adding parallelism and concurrency to applications, enabling developers to efficiently leverage multicore processors while handling the complexities of threading and synchronization.

This document covers the core concepts of TPL, its primary components, common patterns, and best practices for effectively using parallelism to improve application performance.

## Core Concepts

### Tasks vs. Threads

Understanding the distinction between tasks and threads is fundamental:

- **Thread**: An execution context that can run a program. Threads are relatively heavyweight OS resources.
- **Task**: A higher-level abstraction representing an asynchronous operation. Tasks are more lightweight and flexible.

```csharp
// Thread example
Thread thread = new Thread(() => Console.WriteLine("Hello from a thread"));
thread.Start();

// Task example
Task task = Task.Run(() => Console.WriteLine("Hello from a task"));
```

Key differences:

| Task | Thread |
|------|--------|
| Lightweight, more efficient | Heavier resource usage |
| Supports cancellation, continuations | More basic functionality |
| Can use thread pool | One-to-one with OS thread |
| Return values with `Task<T>` | Cannot directly return values |
| Built-in exception propagation | Manual exception handling |

### Task States and Lifecycle

Tasks go through various states during their lifecycle:

1. **Created**: Task has been created but not yet scheduled
2. **WaitingForActivation**: Task is waiting to be scheduled
3. **WaitingToRun**: Task has been scheduled but not yet running
4. **Running**: Task is executing
5. **WaitingForChildrenToComplete**: Task has completed but is waiting for child tasks
6. **RanToCompletion**: Task completed successfully
7. **Canceled**: Task was canceled before completion
8. **Faulted**: Task threw an unhandled exception

You can check a task's status through properties:

```csharp
Task task = Task.Run(() => LongRunningOperation());

// Check task status
Console.WriteLine($"Is task completed? {task.IsCompleted}");
Console.WriteLine($"Is task faulted? {task.IsFaulted}");
Console.WriteLine($"Is task canceled? {task.IsCanceled}");
```

## Task Creation and Execution

### Creating and Starting Tasks

There are multiple ways to create and start tasks:

```csharp
// Method 1: Task.Run (most common for simple operations)
Task task1 = Task.Run(() => Console.WriteLine("Hello from Task.Run"));

// Method 2: TaskFactory.StartNew (more options but more complex)
Task task2 = Task.Factory.StartNew(() => Console.WriteLine("Hello from TaskFactory"), 
                                   TaskCreationOptions.LongRunning);

// Method 3: Task constructor + Start (two-step approach)
Task task3 = new Task(() => Console.WriteLine("Hello from constructor"));
task3.Start();

// Method 4: TaskCompletionSource (for manual completion)
var tcs = new TaskCompletionSource<int>();
Task<int> task4 = tcs.Task;
// Later, complete the task:
tcs.SetResult(42);
```

### Tasks with Return Values

Use `Task<T>` for tasks that return values:

```csharp
// Create a task that returns an int
Task<int> task = Task.Run(() =>
{
    Console.WriteLine("Computing result...");
    Thread.Sleep(1000); // Simulate work
    return 42;
});

// Get the result (blocks until complete)
int result = task.Result;
Console.WriteLine($"Result: {result}");

// Better approach: await the task
int result = await task;
Console.WriteLine($"Result: {result}");
```

### Configuring Task Creation

Configure task behavior with TaskCreationOptions:

```csharp
// Long-running task (may use a dedicated thread instead of thread pool)
Task longTask = Task.Factory.StartNew(() => LongOperation(), 
                                     TaskCreationOptions.LongRunning);

// Attached child tasks (parent waits for children to complete)
Task parentTask = Task.Factory.StartNew(() =>
{
    Console.WriteLine("Parent task starting");
    
    Task.Factory.StartNew(() =>
    {
        Thread.Sleep(1000);
        Console.WriteLine("Child task completed");
    }, TaskCreationOptions.AttachedToParent);
    
    Console.WriteLine("Parent task returning");
}, TaskCreationOptions.DenyChildAttach);
```

## Task Coordination and Composition

### Waiting for Tasks

Different ways to wait for task completion:

```csharp
Task task1 = Task.Run(() => LongOperation(1));
Task task2 = Task.Run(() => LongOperation(2));

// Wait for a single task
task1.Wait(); // Blocks until task1 is complete

// Wait for multiple tasks
Task.WaitAll(task1, task2); // Blocks until all tasks complete
Task.WaitAny(task1, task2); // Blocks until any task completes

// Wait with timeout
bool completed = task1.Wait(TimeSpan.FromSeconds(5)); // Returns false if timeout expires

// Asynchronous waiting (preferred)
await task1; // Waits without blocking the thread
await Task.WhenAll(task1, task2); // Waits for all tasks
await Task.WhenAny(task1, task2); // Waits for any task
```

### Task Continuations

Continuations allow you to chain tasks:

```csharp
// Basic continuation
Task<int> task = Task.Run(() => ComputeValue())
    .ContinueWith(previousTask => 
    {
        int result = previousTask.Result;
        return result * 2;
    });

// Conditional continuations
Task<int> task = Task.Run(() => ComputeValue())
    .ContinueWith(t => ProcessSuccess(t.Result), 
                  TaskContinuationOptions.OnlyOnRanToCompletion)
    .ContinueWith(t => ProcessFailure(t.Exception), 
                  TaskContinuationOptions.OnlyOnFaulted);

// Multiple continuations
Task initialTask = Task.Run(() => Initialize());

Task processTask = initialTask.ContinueWith(t => Process());
Task logTask = initialTask.ContinueWith(t => Log());

// Continuation with different scheduler
Task uiTask = computeTask.ContinueWith(t => UpdateUI(),
                                      CancellationToken.None,
                                      TaskContinuationOptions.None,
                                      TaskScheduler.FromCurrentSynchronizationContext());
```

### Task Composition Patterns

Combine tasks in different ways:

```csharp
// Sequential composition
async Task<int> SequentialCompositionAsync()
{
    int value1 = await Step1Async();
    int value2 = await Step2Async(value1);
    return await Step3Async(value2);
}

// Parallel composition
async Task<int> ParallelCompositionAsync()
{
    Task<int> task1 = Step1Async();
    Task<int> task2 = Step2Async();
    
    await Task.WhenAll(task1, task2);
    
    return task1.Result + task2.Result;
}

// Interleaved composition
async Task<(int, string)> InterleavedCompositionAsync()
{
    // Start both tasks but don't await immediately
    Task<int> numberTask = GetNumberAsync();
    Task<string> stringTask = GetStringAsync();
    
    // Interleave other work
    PrepareForResults();
    
    // Now await the results
    int number = await numberTask;
    string text = await stringTask;
    
    return (number, text);
}
```

## Parallel Class

### Parallel.For and Parallel.ForEach

For loop parallelization:

```csharp
// Parallel.For
Parallel.For(0, 1000, i =>
{
    ProcessItem(i);
});

// Parallel.For with options
ParallelOptions options = new ParallelOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount,
    CancellationToken = cancellationToken
};

Parallel.For(0, 1000, options, i =>
{
    ProcessItem(i);
});

// Parallel.ForEach
List<string> items = GetItems();
Parallel.ForEach(items, item =>
{
    ProcessItem(item);
});

// Parallel.ForEach with local state
Parallel.ForEach(
    items,                 // Source collection
    () => new List<int>(), // Initialize local state for each task
    (item, state, index, localState) =>
    {
        // Process the item and update local state
        localState.Add(ProcessItem(item));
        return localState;
    },
    localState =>
    {
        // Merge local state into shared state
        lock (_lockObject)
        {
            _results.AddRange(localState);
        }
    });
```

### Parallel.Invoke

Execute multiple operations in parallel:

```csharp
Parallel.Invoke(
    () => ProcessPart1(),
    () => ProcessPart2(),
    () => ProcessPart3()
);

// With options
Parallel.Invoke(
    new ParallelOptions { MaxDegreeOfParallelism = 2 },
    () => ProcessPart1(),
    () => ProcessPart2(),
    () => ProcessPart3()
);
```

### Controlling Parallelism

Set the degree of parallelism:

```csharp
ParallelOptions options = new ParallelOptions
{
    // Limit to half of processor cores
    MaxDegreeOfParallelism = Environment.ProcessorCount / 2,
    
    // Support for cancellation
    CancellationToken = cancellationToken
};

Parallel.ForEach(items, options, item =>
{
    ProcessItem(item);
    
    // Check cancellation periodically for long operations
    options.CancellationToken.ThrowIfCancellationRequested();
});
```

## Parallel LINQ (PLINQ)

### Basic PLINQ Operations

Use PLINQ to parallelize LINQ queries:

```csharp
// Convert LINQ query to parallel query with AsParallel()
var result = from num in numbers.AsParallel()
             where IsEven(num)
             select num * num;
             
// Fluent syntax
var result = numbers.AsParallel()
                   .Where(n => IsEven(n))
                   .Select(n => n * n);
                   
// Force query execution with a terminal operation
var resultList = result.ToList();
```

### Configuring PLINQ Execution

Control parallel execution with options:

```csharp
// Set degree of parallelism
var result = numbers.AsParallel()
                   .WithDegreeOfParallelism(4)
                   .Where(n => IsEven(n))
                   .Select(n => n * n);
                   
// Preserve ordering of source elements
var result = numbers.AsParallel()
                   .AsOrdered()
                   .Where(n => IsEven(n))
                   .Select(n => n * n);
                   
// Cancel execution
var cts = new CancellationTokenSource();
var result = numbers.AsParallel()
                   .WithCancellation(cts.Token)
                   .Where(n => IsEven(n))
                   .Select(n => n * n);
                   
// Later, cancel the operation
cts.Cancel();
```

### ForAll for Parallel Processing of Results

Use ForAll for parallel processing of results without intermediate collection:

```csharp
// Process results in parallel without creating a collection
numbers.AsParallel()
       .Where(n => IsEven(n))
       .Select(n => n * n)
       .ForAll(n => Console.WriteLine($"Square: {n}"));
```

## Concurrent Collections

### ConcurrentDictionary

Thread-safe dictionary operations:

```csharp
ConcurrentDictionary<string, int> counts = new ConcurrentDictionary<string, int>();

// Add or update atomically
counts.AddOrUpdate(
    key: "counter",
    addValue: 1,                          // Initial value if key doesn't exist
    updateValueFactory: (key, oldValue) => oldValue + 1  // Update function if key exists
);

// Get or add atomically
int value = counts.GetOrAdd(
    key: "counter",
    valueFactory: key => ExpensiveComputation(key)  // Called only if key doesn't exist
);

// Try operations
if (counts.TryGetValue("counter", out int currentValue))
{
    Console.WriteLine($"Current value: {currentValue}");
}

// Thread-safe update with retry logic
bool updated = false;
while (!updated)
{
    if (counts.TryGetValue("counter", out int currentValue))
    {
        updated = counts.TryUpdate("counter", currentValue + 1, currentValue);
    }
}
```

### ConcurrentQueue, ConcurrentStack, and ConcurrentBag

Thread-safe collections for different needs:

```csharp
// ConcurrentQueue - FIFO (First In, First Out)
ConcurrentQueue<string> queue = new ConcurrentQueue<string>();
queue.Enqueue("Item 1");
queue.Enqueue("Item 2");

if (queue.TryDequeue(out string item))
{
    Console.WriteLine($"Dequeued: {item}");
}

// ConcurrentStack - LIFO (Last In, First Out)
ConcurrentStack<string> stack = new ConcurrentStack<string>();
stack.Push("Item 1");
stack.Push("Item 2");

if (stack.TryPop(out string topItem))
{
    Console.WriteLine($"Popped: {topItem}");
}

// ConcurrentBag - unordered collection
ConcurrentBag<string> bag = new ConcurrentBag<string>();
bag.Add("Item 1");
bag.Add("Item 2");

if (bag.TryTake(out string anyItem))
{
    Console.WriteLine($"Taken: {anyItem}");
}
```

### BlockingCollection for Producer-Consumer Scenarios

Use BlockingCollection for producer-consumer patterns:

```csharp
// Create a bounded blocking collection
BlockingCollection<WorkItem> workItems = new BlockingCollection<WorkItem>(boundedCapacity: 100);

// Producer task
Task producerTask = Task.Run(() =>
{
    for (int i = 0; i < 500; i++)
    {
        var item = new WorkItem { Id = i };
        workItems.Add(item);
        Thread.Sleep(10); // Simulate time to produce each item
    }
    
    // Signal that no more items will be added
    workItems.CompleteAdding();
});

// Consumer tasks
Task[] consumerTasks = new Task[4];
for (int i = 0; i < consumerTasks.Length; i++)
{
    consumerTasks[i] = Task.Run(() =>
    {
        // GetConsumingEnumerable blocks until items are available,
        // and exits when CompleteAdding is called and all items consumed
        foreach (var item in workItems.GetConsumingEnumerable())
        {
            ProcessItem(item);
        }
    });
}

// Wait for all work to complete
producerTask.Wait();
Task.WaitAll(consumerTasks);
```

## Task Scheduling and Synchronization

### TaskScheduler

Control how tasks are scheduled:

```csharp
// Get the default scheduler
TaskScheduler defaultScheduler = TaskScheduler.Default;

// Create a scheduler for the current context (e.g., UI thread)
TaskScheduler uiScheduler = TaskScheduler.FromCurrentSynchronizationContext();

// Schedule a task on the UI thread
Task.Factory.StartNew(() => UpdateUI(), 
                     CancellationToken.None, 
                     TaskCreationOptions.None, 
                     uiScheduler);
                     
// Create a limited concurrency scheduler
LimitedConcurrencyLevelTaskScheduler limitedScheduler = 
    new LimitedConcurrencyLevelTaskScheduler(4);

// Schedule a task on a custom scheduler
Task.Factory.StartNew(() => ProcessItem(), 
                     CancellationToken.None, 
                     TaskCreationOptions.None, 
                     limitedScheduler);
```

Implementation of a custom limited concurrency scheduler:

```csharp
public class LimitedConcurrencyLevelTaskScheduler : TaskScheduler
{
    // Limits concurrency level
    private readonly int _maxDegreeOfParallelism;
    
    // Tasks waiting to be executed
    private readonly LinkedList<Task> _tasks = new LinkedList<Task>();
    
    // Synchronization object
    private readonly object _lockObject = new object();
    
    // Track running tasks
    private int _runningTasks = 0;
    
    public LimitedConcurrencyLevelTaskScheduler(int maxDegreeOfParallelism)
    {
        _maxDegreeOfParallelism = maxDegreeOfParallelism;
    }
    
    protected override void QueueTask(Task task)
    {
        lock (_lockObject)
        {
            _tasks.AddLast(task);
            if (_runningTasks < _maxDegreeOfParallelism)
            {
                _runningTasks++;
                NotifyThreadPoolOfPendingWork();
            }
        }
    }
    
    private void NotifyThreadPoolOfPendingWork()
    {
        ThreadPool.QueueUserWorkItem(_ =>
        {
            Task task = null;
            
            while (true)
            {
                lock (_lockObject)
                {
                    if (_tasks.Count == 0)
                    {
                        _runningTasks--;
                        break;
                    }
                    
                    task = _tasks.First.Value;
                    _tasks.RemoveFirst();
                }
                
                TryExecuteTask(task);
            }
        });
    }
    
    protected override bool TryExecuteTaskInline(Task task, bool taskWasPreviouslyQueued)
    {
        if (taskWasPreviouslyQueued)
            return false;
            
        return TryExecuteTask(task);
    }
    
    protected override IEnumerable<Task> GetScheduledTasks()
    {
        lock (_lockObject)
        {
            return _tasks.ToArray();
        }
    }
    
    public override int MaximumConcurrencyLevel => _maxDegreeOfParallelism;
}
```

### SynchronizationContext

Understand synchronization context for UI and ASP.NET applications:

```csharp
// Capture the current synchronization context (e.g., UI thread)
SynchronizationContext uiContext = SynchronizationContext.Current;

// Execute work on a background thread
Task.Run(() =>
{
    // Perform work on a background thread
    var result = PerformCalculation();
    
    // Post the result back to the UI thread
    uiContext.Post(_ =>
    {
        // This code runs on the UI thread
        ResultTextBox.Text = result.ToString();
    }, null);
});
```

### SemaphoreSlim for Throttling

Limit the degree of parallelism:

```csharp
public async Task ProcessFilesAsync(List<string> filePaths)
{
    // Limit concurrent file operations to 5
    using var semaphore = new SemaphoreSlim(5);
    var tasks = new List<Task>();
    
    foreach (var filePath in filePaths)
    {
        // Wait to enter the semaphore
        await semaphore.WaitAsync();
        
        // Start a new task but don't await it yet
        tasks.Add(Task.Run(async () =>
        {
            try
            {
                await ProcessFileAsync(filePath);
            }
            finally
            {
                // Release the semaphore when done
                semaphore.Release();
            }
        }));
    }
    
    // Wait for all tasks to complete
    await Task.WhenAll(tasks);
}
```

## Advanced Patterns

### Pipeline Pattern

Implement a parallel processing pipeline:

```csharp
public class ParallelPipeline<TInput, TOutput>
{
    private readonly BlockingCollection<TInput> _inputQueue = new BlockingCollection<TInput>();
    private readonly BlockingCollection<TOutput> _outputQueue = new BlockingCollection<TOutput>();
    private readonly CancellationTokenSource _cts = new CancellationTokenSource();
    private readonly Task _processingTask;
    
    public ParallelPipeline(Func<TInput, TOutput> processor, int workerCount)
    {
        _processingTask = Task.Run(() =>
        {
            var workers = new Task[workerCount];
            
            for (int i = 0; i < workerCount; i++)
            {
                workers[i] = Task.Run(() =>
                {
                    try
                    {
                        foreach (var item in _inputQueue.GetConsumingEnumerable(_cts.Token))
                        {
                            var result = processor(item);
                            _outputQueue.Add(result, _cts.Token);
                        }
                    }
                    catch (OperationCanceledException)
                    {
                        // Ignore cancellation
                    }
                }, _cts.Token);
            }
            
            try
            {
                Task.WaitAll(workers);
            }
            catch (AggregateException)
            {
                // Handle worker exceptions
            }
            finally
            {
                _outputQueue.CompleteAdding();
            }
        });
    }
    
    public void AddInput(TInput input)
    {
        _inputQueue.Add(input);
    }
    
    public void CompleteAdding()
    {
        _inputQueue.CompleteAdding();
    }
    
    public IEnumerable<TOutput> GetOutputEnumerable()
    {
        return _outputQueue.GetConsumingEnumerable();
    }
    
    public void Cancel()
    {
        _cts.Cancel();
    }
    
    public Task Completion => _processingTask;
}

// Usage example
var pipeline = new ParallelPipeline<string, int>(
    processor: s => ProcessString(s),
    workerCount: Environment.ProcessorCount
);

// Add inputs
foreach (var input in inputData)
{
    pipeline.AddInput(input);
}

pipeline.CompleteAdding();

// Process outputs as they become available
foreach (var output in pipeline.GetOutputEnumerable())
{
    Console.WriteLine($"Output: {output}");
}

// Wait for completion
await pipeline.Completion;
```

### Map-Reduce Pattern

Implement a parallel map-reduce pattern:

```csharp
public class MapReduce<TInput, TIntermediate, TOutput>
{
    private readonly Func<TInput, IEnumerable<TIntermediate>> _mapper;
    private readonly Func<IEnumerable<TIntermediate>, TOutput> _reducer;
    private readonly int _parallelism;
    
    public MapReduce(
        Func<TInput, IEnumerable<TIntermediate>> mapper,
        Func<IEnumerable<TIntermediate>, TOutput> reducer,
        int parallelism = -1)
    {
        _mapper = mapper;
        _reducer = reducer;
        _parallelism = parallelism > 0 ? parallelism : Environment.ProcessorCount;
    }
    
    public TOutput Process(IEnumerable<TInput> inputs)
    {
        // Apply the map function in parallel
        var intermediateResults = inputs
            .AsParallel()
            .WithDegreeOfParallelism(_parallelism)
            .SelectMany(input => _mapper(input))
            .ToList();
        
        // Apply the reduce function
        return _reducer(intermediateResults);
    }
    
    public async Task<TOutput> ProcessAsync(IEnumerable<TInput> inputs)
    {
        // Split inputs into batches
        var batches = inputs
            .Select((input, index) => new { input, index })
            .GroupBy(x => x.index % _parallelism)
            .Select(g => g.Select(x => x.input).ToList())
            .ToList();
        
        // Process batches in parallel
        var tasks = batches.Select(async batch =>
        {
            return await Task.Run(() => 
                batch.SelectMany(input => _mapper(input)).ToList());
        }).ToList();
        
        // Wait for all mapping to complete
        var intermediateResults = await Task.WhenAll(tasks);
        
        // Apply the reduce function
        return _reducer(intermediateResults.SelectMany(x => x));
    }
}

// Usage example - word count
var wordCount = new MapReduce<string, KeyValuePair<string, int>, Dictionary<string, int>>(
    // Map: Split text into words and count each occurrence
    mapper: text => text.Split()
                        .Where(word => !string.IsNullOrWhiteSpace(word))
                        .Select(word => word.ToLower())
                        .GroupBy(word => word)
                        .Select(g => new KeyValuePair<string, int>(g.Key, g.Count())),
    
    // Reduce: Combine word counts
    reducer: pairs => pairs.GroupBy(pair => pair.Key)
                          .ToDictionary(g => g.Key, g => g.Sum(pair => pair.Value))
);

// Process a collection of text documents
var result = wordCount.Process(documents);

// Print results
foreach (var pair in result.OrderByDescending(p => p.Value))
{
    Console.WriteLine($"{pair.Key}: {pair.Value}");
}
```

### Thread-Local Storage with Tasks

Use thread-local state with parallel operations:

```csharp
// Example using ThreadLocal<T>
public void ProcessItemsWithThreadLocal(List<int> items)
{
    int totalSum = 0;
    var localSum = new ThreadLocal<int>(() => 0);
    
    Parallel.ForEach(items, item =>
    {
        localSum.Value += item;
    });
    
    // Combine results from all threads
    Parallel.ForEach(localSum.Values, threadSum =>
    {
        Interlocked.Add(ref totalSum, threadSum);
    });
    
    Console.WriteLine($"Total sum: {totalSum}");
    
    // Dispose the ThreadLocal instance when done
    localSum.Dispose();
}

// Example using Thread Static
[ThreadStatic]
private static int _threadStaticCounter;

public void ProcessItemsWithThreadStatic(List<int> items)
{
    // Reset the counter for all threads
    _threadStaticCounter = 0;
    
    Parallel.ForEach(items, item =>
    {
        _threadStaticCounter++;
        ProcessItem(item);
    });
    
    // Note: We can't easily combine the results from all threads
    // without additional synchronization
}
```

## Performance Considerations

### CPU-Bound vs. I/O-Bound Operations

Choose the right parallelism approach based on the type of work:

```csharp
// CPU-bound operations - use Task.Run or Parallel
public async Task<int> ProcessCpuBoundOperationAsync(List<int> data)
{
    // Compute-intensive work runs on a background thread
    return await Task.Run(() =>
    {
        // Use Parallel.ForEach for CPU-bound operations
        int totalSum = 0;
        
        Parallel.ForEach(
            data,
            () => 0,  // Thread-local initial state
            (item, state, index, localSum) => localSum + item,  // Body
            localSum => Interlocked.Add(ref totalSum, localSum)  // Final action
        );
        
        return totalSum;
    });
}

// I/O-bound operations - use native async APIs
public async Task<List<string>> ProcessIoBoundOperationAsync(List<string> urls)
{
    // For I/O-bound operations, use Task.WhenAll with native async APIs
    var tasks = urls.Select(url => DownloadContentAsync(url));
    
    // Task.WhenAll is more appropriate than Parallel.ForEach for I/O operations
    var results = await Task.WhenAll(tasks);
    
    return results.ToList();
}
```

### Partitioning Strategies

Choose the right partitioning strategy:

```csharp
// Range partitioning
public void ProcessUsingRangePartitioner(int[] data)
{
    var rangePartitioner = Partitioner.Create(0, data.Length);
    
    Parallel.ForEach(rangePartitioner, range =>
    {
        for (int i = range.Item1; i < range.Item2; i++)
        {
            ProcessItem(data[i]);
        }
    });
}

// Chunk partitioning
public void ProcessUsingChunkPartitioner<T>(IList<T> data)
{
    var chunkPartitioner = Partitioner.Create(data, loadBalance: true);
    
    Parallel.ForEach(chunkPartitioner, chunk =>
    {
        foreach (var item in chunk)
        {
            ProcessItem(item);
        }
    });
}

// Custom partitioning
public void ProcessWithCustomPartitioning<T>(IEnumerable<T> data, int degreeOfParallelism)
{
    var batches = data
        .Select((item, index) => new { Item = item, Index = index })
        .GroupBy(x => x.Index % degreeOfParallelism)
        .Select(g => g.Select(x => x.Item).ToList())
        .ToList();
    
    Parallel.ForEach(batches, batch =>
    {
        foreach (var item in batch)
        {
            ProcessItem(item);
        }
    });
}
```

### Load Balancing

Ensure balanced workloads:

```csharp
// Static partitioning (may lead to unbalanced workloads)
public void StaticPartitioningExample(List<WorkItem> items, int workerCount)
{
    // Divide items evenly among workers
    var itemsPerWorker = (items.Count + workerCount - 1) / workerCount;
    var tasks = new Task[workerCount];
    
    for (int i = 0; i < workerCount; i++)
    {
        int workerIndex = i; // Capture for closure
        tasks[i] = Task.Run(() =>
        {
            int start = workerIndex * itemsPerWorker;
            int end = Math.Min(start + itemsPerWorker, items.Count);
            
            for (int j = start; j < end; j++)
            {
                ProcessItem(items[j]);
            }
        });
    }
    
    Task.WaitAll(tasks);
}

// Dynamic partitioning (better load balancing)
public void DynamicPartitioningExample(List<WorkItem> items)
{
    var partitioner = Partitioner.Create(items, loadBalance: true);
    
    Parallel.ForEach(partitioner, item =>
    {
        ProcessItem(item);
    });
}

// Work stealing queue
public void WorkStealingExample(List<WorkItem> items)
{
    var workStealingQueue = new ConcurrentQueue<WorkItem>(items);
    var processingTasks = new Task[Environment.ProcessorCount];
    
    for (int i = 0; i < processingTasks.Length; i++)
    {
        processingTasks[i] = Task.Run(() =>
        {
            while (workStealingQueue.TryDequeue(out var item))
            {
                ProcessItem(item);
            }
        });
    }
    
    Task.WaitAll(processingTasks);
}
```

### Thread Pool Considerations

Understand and manage the thread pool:

```csharp
// Get thread pool information
ThreadPool.GetAvailableThreads(out int workerThreads, out int completionPortThreads);
ThreadPool.GetMaxThreads(out int maxWorkerThreads, out int maxCompletionPortThreads);

Console.WriteLine($"Available worker threads: {workerThreads}/{maxWorkerThreads}");
Console.WriteLine($"Available completion port threads: {completionPortThreads}/{maxCompletionPortThreads}");

// Configure thread pool
ThreadPool.SetMinThreads(workerThreads: 16, completionPortThreads: 16);
ThreadPool.SetMaxThreads(workerThreads: 64, completionPortThreads: 64);

// Use QueueUserWorkItem for simple thread pool tasks
ThreadPool.QueueUserWorkItem(state =>
{
    Console.WriteLine("Working on thread pool thread");
});

// Better: Use Task.Run instead
Task.Run(() =>
{
    Console.WriteLine("Working on thread pool thread with Task");
});
```

## Common Pitfalls and Solutions

### Race Conditions

Avoid and fix race conditions:

```csharp
// PROBLEMATIC - Race condition
int counter = 0;
Parallel.For(0, 100000, i =>
{
    counter++; // Race condition!
});

// SOLUTION 1: Use Interlocked
int counter = 0;
Parallel.For(0, 100000, i =>
{
    Interlocked.Increment(ref counter);
});

// SOLUTION 2: Use lock
int counter = 0;
object lockObj = new object();
Parallel.For(0, 100000, i =>
{
    lock (lockObj)
    {
        counter++;
    }
});

// SOLUTION 3: Use thread-local counters
int counter = 0;
Parallel.For(0, 100000,
    () => 0, // Initialize local counter
    (i, loop, localCounter) =>
    {
        return localCounter + 1; // Update local counter
    },
    localCounter => Interlocked.Add(ref counter, localCounter) // Merge local counters
);
```

### Deadlocks

Avoid and detect deadlocks:

```csharp
// PROBLEMATIC - Potential deadlock
object lock1 = new object();
object lock2 = new object();

Task task1 = Task.Run(() =>
{
    lock (lock1)
    {
        Thread.Sleep(100); // Increase chance of deadlock
        lock (lock2)
        {
            // Process with both locks
        }
    }
});

Task task2 = Task.Run(() =>
{
    lock (lock2)
    {
        Thread.Sleep(100); // Increase chance of deadlock
        lock (lock1)
        {
            // Process with both locks
        }
    }
});

// SOLUTION - Always acquire locks in the same order
Task betterTask1 = Task.Run(() =>
{
    lock (lock1)
    {
        lock (lock2)
        {
            // Process with both locks
        }
    }
});

Task betterTask2 = Task.Run(() =>
{
    lock (lock1)
    {
        lock (lock2)
        {
            // Process with both locks
        }
    }
});

// SOLUTION - Use a timeout to detect deadlocks
bool lockAcquired = false;
try
{
    Monitor.TryEnter(lockObj, TimeSpan.FromSeconds(5), ref lockAcquired);
    if (lockAcquired)
    {
        // Process with lock
    }
    else
    {
        // Probable deadlock detected
        Console.WriteLine("Failed to acquire lock - potential deadlock");
    }
}
finally
{
    if (lockAcquired)
    {
        Monitor.Exit(lockObj);
    }
}
```

### Excessive Task Creation

Avoid creating too many tasks:

```csharp
// PROBLEMATIC - Too many small tasks
var result = Enumerable.Range(0, 10000)
    .Select(i => Task.Run(() => ProcessItem(i))) // Creates 10,000 tasks!
    .ToList();
    
Task.WaitAll(result.ToArray());

// SOLUTION - Batch work into larger tasks
var result = Enumerable.Range(0, 10000)
    .Chunk(1000) // Group into batches of 1000
    .Select(chunk => Task.Run(() =>
    {
        foreach (var i in chunk)
        {
            ProcessItem(i);
        }
    }))
    .ToList();
    
Task.WaitAll(result.ToArray());

// BETTER SOLUTION - Use Parallel.ForEach
Parallel.ForEach(Enumerable.Range(0, 10000), i =>
{
    ProcessItem(i);
});
```

### Proper Exception Handling

Handle exceptions in parallel code:

```csharp
// PROBLEMATIC - Exceptions lost
Parallel.For(0, 100, i =>
{
    if (i == 50) throw new Exception("Error at item 50");
    ProcessItem(i);
});

// SOLUTION - Capture exceptions
try
{
    Parallel.For(0, 100, i =>
    {
        if (i == 50) throw new Exception($"Error at item {i}");
        ProcessItem(i);
    });
}
catch (AggregateException ae)
{
    foreach (var ex in ae.InnerExceptions)
    {
        Console.WriteLine($"Parallel.For error: {ex.Message}");
    }
}

// BETTER SOLUTION - Handle exceptions per task
var results = new ConcurrentBag<(int Index, Exception Error)>();

Parallel.For(0, 100, i =>
{
    try
    {
        if (i == 50) throw new Exception($"Error at item {i}");
        ProcessItem(i);
    }
    catch (Exception ex)
    {
        results.Add((i, ex));
    }
});

foreach (var (index, error) in results)
{
    Console.WriteLine($"Item {index} failed: {error.Message}");
}
```

## Real-World Examples

### Parallel Image Processing

Process images in parallel:

```csharp
public class ParallelImageProcessor
{
    public void ProcessImages(string[] imagePaths, string outputDirectory)
    {
        Directory.CreateDirectory(outputDirectory);
        
        ParallelOptions options = new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount
        };
        
        Parallel.ForEach(imagePaths, options, imagePath =>
        {
            try
            {
                string outputPath = Path.Combine(
                    outputDirectory, 
                    Path.GetFileNameWithoutExtension(imagePath) + "_processed.jpg"
                );
                
                using (var image = Image.FromFile(imagePath))
                {
                    // Apply image processing
                    using (var processedImage = ApplyEffects(image))
                    {
                        processedImage.Save(outputPath, ImageFormat.Jpeg);
                    }
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error processing {imagePath}: {ex.Message}");
            }
        });
    }
    
    private Image ApplyEffects(Image sourceImage)
    {
        // Apply various image effects
        // (Resize, crop, adjust brightness, etc.)
        return processedImage;
    }
}

// Usage
var processor = new ParallelImageProcessor();
processor.ProcessImages(
    imagePaths: Directory.GetFiles(@"C:\Images", "*.jpg"),
    outputDirectory: @"C:\Images\Processed"
);
```

### Parallel Data Processing

Process large datasets in parallel:

```csharp
public class ParallelDataProcessor
{
    public async Task<ProcessingResult> ProcessLargeDatasetAsync(
        string dataFilePath,
        IProgress<int> progress = null,
        CancellationToken cancellationToken = default)
    {
        // Read the data
        string[] lines = await File.ReadAllLinesAsync(dataFilePath, cancellationToken);
        
        // Process in parallel with progress reporting
        var result = new ProcessingResult();
        var lockObj = new object();
        int processedLines = 0;
        int totalLines = lines.Length;
        
        try
        {
            await Task.Run(() =>
            {
                Parallel.ForEach(
                    Partitioner.Create(0, lines.Length),
                    new ParallelOptions
                    {
                        CancellationToken = cancellationToken,
                        MaxDegreeOfParallelism = Environment.ProcessorCount
                    },
                    () => new LocalResult(), // Initialize thread-local state
                    (range, state, index, localResult) =>
                    {
                        for (int i = range.Item1; i < range.Item2; i++)
                        {
                            // Process each line
                            var lineResult = ProcessLine(lines[i]);
                            
                            // Update local result
                            localResult.ValidRecords += lineResult.IsValid ? 1 : 0;
                            localResult.InvalidRecords += lineResult.IsValid ? 0 : 1;
                            localResult.TotalValue += lineResult.Value;
                            
                            if (!string.IsNullOrEmpty(lineResult.Error))
                            {
                                localResult.Errors.Add($"Line {i+1}: {lineResult.Error}");
                            }
                            
                            // Report progress
                            int processed = Interlocked.Increment(ref processedLines);
                            if (processed % 100 == 0 || processed == totalLines)
                            {
                                progress?.Report((int)((double)processed / totalLines * 100));
                            }
                            
                            cancellationToken.ThrowIfCancellationRequested();
                        }
                        
                        return localResult;
                    },
                    localResult =>
                    {
                        // Merge thread-local results into the shared result
                        lock (lockObj)
                        {
                            result.ValidRecords += localResult.ValidRecords;
                            result.InvalidRecords += localResult.InvalidRecords;
                            result.TotalValue += localResult.TotalValue;
                            result.Errors.AddRange(localResult.Errors);
                        }
                    }
                );
            }, cancellationToken);
            
            return result;
        }
        catch (OperationCanceledException)
        {
            result.WasCancelled = true;
            return result;
        }
    }
    
    private LineResult ProcessLine(string line)
    {
        // Process a single line of data
        var result = new LineResult();
        try
        {
            // Parse and validate the line
            // Set result properties
        }
        catch (Exception ex)
        {
            result.IsValid = false;
            result.Error = ex.Message;
        }
        return result;
    }
    
    // Local result for thread-local state
    private class LocalResult
    {
        public int ValidRecords { get; set; }
        public int InvalidRecords { get; set; }
        public decimal TotalValue { get; set; }
        public List<string> Errors { get; set; } = new List<string>();
    }
    
    // Result for a single line
    private class LineResult
    {
        public bool IsValid { get; set; }
        public decimal Value { get; set; }
        public string Error { get; set; }
    }
}

// Overall processing result
public class ProcessingResult
{
    public int ValidRecords { get; set; }
    public int InvalidRecords { get; set; }
    public decimal TotalValue { get; set; }
    public List<string> Errors { get; set; } = new List<string>();
    public bool WasCancelled { get; set; }
}
```

### Web Scraper with TPL

Implement a parallel web scraper:

```csharp
public class ParallelWebScraper
{
    private readonly HttpClient _httpClient;
    private readonly int _maxConcurrency;
    private readonly TimeSpan _requestDelay;
    
    public ParallelWebScraper(int maxConcurrency = 10, TimeSpan? requestDelay = null)
    {
        _httpClient = new HttpClient();
        _httpClient.DefaultRequestHeaders.UserAgent.ParseAdd(
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
        );
        _maxConcurrency = maxConcurrency;
        _requestDelay = requestDelay ?? TimeSpan.FromMilliseconds(500);
    }
    
    public async Task<ScrapingResult> ScrapeUrlsAsync(
        IEnumerable<string> urls,
        CancellationToken cancellationToken = default)
    {
        var result = new ScrapingResult();
        var semaphore = new SemaphoreSlim(_maxConcurrency);
        var tasks = new List<Task>();
        
        foreach (var url in urls)
        {
            // Throttle concurrency with semaphore
            await semaphore.WaitAsync(cancellationToken);
            
            tasks.Add(Task.Run(async () =>
            {
                try
                {
                    var pageResult = await ScrapeUrlAsync(url, cancellationToken);
                    
                    lock (result)
                    {
                        result.SuccessCount++;
                        result.Pages.Add(pageResult);
                        
                        // Extract links for further processing if needed
                        result.ExtractedUrls.AddRange(pageResult.Links);
                    }
                }
                catch (Exception ex)
                {
                    lock (result)
                    {
                        result.FailureCount++;
                        result.Errors.Add(new UrlError
                        {
                            Url = url,
                            Error = ex.Message
                        });
                    }
                }
                finally
                {
                    // Ensure rate limiting between requests
                    await Task.Delay(_requestDelay, CancellationToken.None);
                    semaphore.Release();
                }
            }, cancellationToken));
        }
        
        await Task.WhenAll(tasks);
        
        return result;
    }
    
    private async Task<PageResult> ScrapeUrlAsync(string url, CancellationToken cancellationToken)
    {
        var result = new PageResult { Url = url };
        
        var response = await _httpClient.GetAsync(url, cancellationToken);
        response.EnsureSuccessStatusCode();
        
        var content = await response.Content.ReadAsStringAsync(cancellationToken);
        result.Content = content;
        
        // Extract information (simplified, use a proper HTML parser in production)
        result.Title = ExtractTitle(content);
        result.Links = ExtractLinks(content, url);
        
        return result;
    }
    
    private string ExtractTitle(string content)
    {
        // Extract page title (simplified)
        var titleMatch = Regex.Match(content, @"<title>(.*?)</title>");
        return titleMatch.Success ? titleMatch.Groups[1].Value : string.Empty;
    }
    
    private List<string> ExtractLinks(string content, string baseUrl)
    {
        // Extract links (simplified)
        var links = new List<string>();
        var matches = Regex.Matches(content, @"href=""(.*?)""");
        
        foreach (Match match in matches)
        {
            if (match.Success && match.Groups.Count > 1)
            {
                var href = match.Groups[1].Value;
                if (Uri.TryCreate(new Uri(baseUrl), href, out Uri absoluteUri))
                {
                    links.Add(absoluteUri.ToString());
                }
            }
        }
        
        return links;
    }
}

public class ScrapingResult
{
    public int SuccessCount { get; set; }
    public int FailureCount { get; set; }
    public List<PageResult> Pages { get; set; } = new List<PageResult>();
    public List<UrlError> Errors { get; set; } = new List<UrlError>();
    public HashSet<string> ExtractedUrls { get; set; } = new HashSet<string>();
}

public class PageResult
{
    public string Url { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    public List<string> Links { get; set; } = new List<string>();
}

public class UrlError
{
    public string Url { get; set; }
    public string Error { get; set; }
}
```

## Further Reading

- [Task Parallel Library (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/task-parallel-library-tpl)
- [Parallel Programming in .NET (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/)
- [Task-based Asynchronous Pattern (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap)
- [Patterns of Parallel Programming (White Paper)](https://www.microsoft.com/en-us/download/details.aspx?id=19222)
- [Concurrent Collections (Microsoft Docs)](https://docs.microsoft.com/en-us/dotnet/standard/collections/thread-safe/)
- [Parallel Programming with .NET (Stephen Toub)](https://devblogs.microsoft.com/pfxteam/)
- [Concurrency in C# Cookbook (Stephen Cleary)](https://www.oreilly.com/library/view/concurrency-in-c/9781491906675/)

## Related Topics

- [Async/Await Deep Dive](async-await-deep-dive.md)
- [Async Best Practices](best-practices.md)
- [Thread Synchronization](../threading/thread-synchronization.md)
- [Parallel Programming](../threading/parallel-programming.md)
- [Thread Pool](../threading/thread-pool.md)
- [CPU-Bound Optimization](../optimization/cpu-bound-optimization.md)